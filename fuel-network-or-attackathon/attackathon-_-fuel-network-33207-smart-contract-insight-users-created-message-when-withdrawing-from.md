# Attackathon \_ Fuel Network 33207 - \[Smart Contract - Insight] users created message when withdrawing

Submitted on Sun Jul 14 2024 14:24:12 GMT-0400 (Atlantic Standard Time) by @zeroK for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33207

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/fuel-bridge/tree/e3e673e31f9e72d757d68979bb6796a0b7f9c8bc/packages/fungible-token>

Impacts:

* Block stuffing
* Permanent freezing of funds
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

when users want to make withdraw call on L2(fuel) they should call the withdraw function which create a messageOut that correspond to the required data to call the `relayMessage` function on messagePortal, the sender of the messageOut should be set to the fuel bridge address so that the call to finalizedWithdraw which encoded as data can be passed, this is important step to bypass the `message.sender == assetIssuerID` check, however there is a function that updates the assetIssuerID to another 32 bytes address, this can cause lose of users funds because when users call withdraw function the sender of their message is equal to the bridge address which is the assetIssuerID and if the assetIssuer get Updated by the admin then old messages that set the bridge address as sender which is equal to the old assetIssuerID their message will never be able to executed as the `message.sender == assetIssuerID` will revert.

the reason we set this report to medium is that it depends on an action from the admin to update assetIssuerID innocently and cause user funds locked for ever.

## Vulnerability Details

this is how the withdraw message created:

```sway

    #[payable]
    #[storage(read, write)]
    fn withdraw(to: b256) {
        let amount: u64 = msg_amount();
        require(amount != 0, BridgeFungibleTokenError::NoCoinsSent);

        let asset_id = msg_asset_id();
        let sub_id = _asset_to_sub_id(asset_id);
        let token_id = _asset_to_token_id(asset_id);
        let l1_address = _asset_to_l1_address(asset_id);

        // Hexens Fuel1-4: Might benefit from a custom error message
        storage
            .tokens_minted
            .insert(
                asset_id,
                storage
                    .tokens_minted
                    .get(asset_id)
                    .read() - amount,
            );
        burn(sub_id, amount);

        // send a message to unlock this amount on the base layer gateway contract
        let sender = msg_sender().unwrap();
        send_message(
            BRIDGED_TOKEN_GATEWAY,
            encode_data(to, amount.as_u256().as_b256(), l1_address, token_id), // this will be the finalizedWithdraw 4 bytes + inputs
            0,
        );
        log(WithdrawalEvent {
            to: to,
            from: sender,
            amount: amount,
        });
    }

```

this call will set the message.sender to the l2 address that mint token or the assetIssuerID and when user try to execute the call to relayMessage on the message portal contract:

```solidity

    function relayMessage(
        Message calldata message,
        FuelBlockHeaderLite calldata rootBlockHeader,
        FuelBlockHeader calldata blockHeader,
        MerkleProof calldata blockInHistoryProof,
        MerkleProof calldata messageInBlockProof
    ) external payable virtual override whenNotPaused {
        if (withdrawalsPaused) {
            revert WithdrawalsPaused();
        }

        //verify root block header
        if (!_fuelChainState.finalized(rootBlockHeader.computeConsensusHeaderHash(), rootBlockHeader.height)) {
            revert UnfinalizedBlock();
        } // if not finalized then revert

        //verify block in history
        if (
            !verifyBinaryTree(
                rootBlockHeader.prevRoot,
                abi.encodePacked(blockHeader.computeConsensusHeaderHash()),
                blockInHistoryProof.proof,
                blockInHistoryProof.key,
                rootBlockHeader.height
            )
        ) revert InvalidBlockInHistoryProof();

        //verify message in block
        bytes32 messageId = CryptographyLib.hash(
            abi.encodePacked(message.sender, message.recipient, message.nonce, message.amount, message.data)
        );

        if (messageIsBlacklisted[messageId]) {
            revert MessageBlacklisted();
        }

        if (
            !verifyBinaryTree(
                blockHeader.outputMessagesRoot,
                abi.encodePacked(messageId),
                messageInBlockProof.proof,
                messageInBlockProof.key,
                blockHeader.outputMessagesCount
            )
        ) revert InvalidMessageInBlockProof();

        //execute message
        _executeMessage(messageId, message);
    }

    /// @notice Executes a message in the given header
    /// @param messageId The id of message to execute
    /// @param message The message to execute
    function _executeMessage(bytes32 messageId, Message calldata message) internal virtual override nonReentrant {
        if (_incomingMessageSuccessful[messageId]) revert AlreadyRelayed();

        //set message sender for receiving contract to reference
        _incomingMessageSender = message.sender;

        // v2: update accounting if the message carries an amount
        bool success;
        bytes memory result;
        if (message.amount > 0) {
            uint256 withdrawnAmount = message.amount * PRECISION;

            // Underflow check enabled since the amount is coded in `message`
            totalDeposited -= withdrawnAmount;

            (success, result) = address(uint160(uint256(message.recipient))).call{value: withdrawnAmount}(message.data); 
        } else {
            (success, result) = address(uint160(uint256(message.recipient))).call(message.data); // call finalizedWithdraw
        }

        if (!success) {
            // Look for revert reason and bubble it up if present
            if (result.length > 0) {
                // The easiest way to bubble the revert reason is using memory via assembly
                /// @solidity memory-safe-assembly
                assembly {
                    let returndata_size := mload(result)
                    revert(add(32, result), returndata_size)
                }
            }
            revert MessageRelayFailed();
        }

        //unset message sender reference
        _incomingMessageSender = NULL_MESSAGE_SENDER;

        //keep track of successfully relayed messages
        _incomingMessageSuccessful[messageId] = true;

        //emit event for successful message relay
        emit MessageRelayed(messageId, message.sender, message.recipient, message.amount);
    }

```

we can see the \_incomingMessageSender is set to message.sender which is same as assetIusserID, however if the admin make call to the function then any message created on L2 and ready to execute on L1 by calling relayMessage will revert because the assetIssuerID is no longer equal to the old one or message.sender to messages that created before updating the assetIssuerID.

```solidity
    /// @notice sets the entity on L2 that will mint the tokens
    function setAssetIssuerId(bytes32 id) external payable virtual onlyRole(DEFAULT_ADMIN_ROLE) {
        assetIssuerId = id;
    }


// the call to finalizeWithdraw will revert because of the update assetIssuerID after creating message according to the old one

    /// @notice Finalizes the withdrawal process from the Fuel side gateway contract
    /// @param to Account to send withdrawn tokens to
    /// @param tokenAddress Address of the token being withdrawn from Fuel
    /// @param l2BurntAmount Amount of tokens to withdraw
    /// @dev Made payable to reduce gas costs
    function finalizeWithdrawal(
        address to,
        address tokenAddress,
        uint256 l2BurntAmount,
        uint256 /*tokenId*/
    ) external payable virtual override whenNotPaused onlyFromPortal {
        if (l2BurntAmount == 0) {
            revert CannotWithdrawZero();
        }

        if (messageSender() != assetIssuerId) {
            revert InvalidSender();
        } 

        uint8 decimals = _getTokenDecimals(tokenAddress);
        uint256 amount = _adjustWithdrawalDecimals(decimals, l2BurntAmount); 

        //reduce deposit balance and transfer tokens (math will underflow if amount is larger than allowed)
        _deposits[tokenAddress] = _deposits[tokenAddress] - l2BurntAmount;
        IERC20MetadataUpgradeable(tokenAddress).safeTransfer(to, amount);

        //emit event for successful token withdraw
        emit Withdrawal(bytes32(uint256(uint160(to))), tokenAddress, amount);
    }


```

while the function exist its mean there is a point when the admin want to change the assetIssuerID and if this happen while the message that created depends on the old assetIssuerID will never be executed and users on L2 burn their tokens for nothing

## Impact Details

loss/stuck of users funds when the assetIssuerID get updated.

## References

its recommended to add functionality to the bridges contract to allow updating the messages that executed depends on previous L2 minter address/assetIssuerID when the assetIssuerID get updated when required, the functionality should have sanity checks that only allow updating message data and re execute it when :

* the assetIssuerID updated.
* the message not executed yet.

## Proof of concept

## Proof of Concept

on L2 POC:

```sway

contract;

use standards::src10::SRC10;

abi MyContract {
    fn call_withdraw();
 
}

impl MyContract for contract {

    fn foo(to_address: b256, bridge: ContractId, bridged_asset: AssetId) {
        let bridge_abi = abi(SRC10, bridge.value);
         bridge_abi {
            gas: 10000,
          coins: 100,
             asset_id: bridged_asset,
         }.withdraw(to_address);


        return true;
     }

}

#[test] 

  fn test_contract() {
     let caller = abi(MyContract, CONTRACT_ID);
     let to = 0xf8f8b6283d7fa5b672b530cbb84fcccb4ff8dc40f8176ef4544ddb1f1952ad07
     let contract_ids = ContractID::this();
     let asset_id = b256::zero();
    let result = caller.withdraw(to, contract_ids, asset_id);
    assert(result == true)
  }

// this will show killed because of an issue in forc 
```

the call to update the assetIssuerID on L1 POC:

```solidity
// SPDX-License-Identifier: BSD 3-Clause License

pragma solidity ^0.8.15;

import "forge-std/Test.sol";
import "./interfaces/IPortalMessage.sol";
import "./interfaces/IERC20.sol";

// run this test in this repo : https://github.com/SunWeb3Sec/defilabs  run forge test --contracts src/test/test.sol -vvvv

contract ContractTest is Test {
    event LogBytes(bytes32 data);

    address public alice;
    address public bob;

    bytes32 public assetIssuerID;

    FuelMessagePortalV3 fuelPortal =
        FuelMessagePortalV3(0x01855B78C1f8868DE70e84507ec735983bf262dA);

    address usdt = 0xdAC17F958D2ee523a2206206994597C13D831ec7;

    enum MessageType {
        DEPOSIT_TO_ADDRESS,
        DEPOSIT_TO_CONTRACT,
        DEPOSIT_WITH_DATA,
        METADATA
    }

    function setUp() public {
        vm.createSelectFork("sepolia");
        alice = vm.addr(1);
        bob = vm.addr(2);
        deal(address(alice), 1 ether);
    }

    function testExploitHere() public {
        //STEP 1 only to show the id before and after

        bytes32 id1 = bytes32(uint256(uint160(bob)));

        setAssetIssuerID(id1);

        uint256 l2MintedAmount = 100 * 10 ** 9;

        bytes memory depositMessage1 = abi.encodePacked(
            assetIssuerID, // this will be 0000
            uint256(MessageType.DEPOSIT_TO_ADDRESS),
            bytes32(uint256(uint160(usdt))),
            uint256(0), // token_id = 0 for all erc20 deposits
            bytes32(uint256(uint160(msg.sender))),
            alice,
            l2MintedAmount,
            uint256(18)
        ); // not important for this POC

        emit LogBytes(id1); // log 32 bytes

        //STEP 2: after user initialized transaction to withdraw on L2 with setting the the old L2 bridge address(assetIssuerID) the admin updated the assetIssuer which block user to withdraw their tokesn in L1

        bytes32 id2 = bytes32(uint256(uint160(alice)));

        setAssetIssuerID(id2);

        emit LogBytes(id2);
    }

    function setAssetIssuerID(bytes32 id) public {
        assetIssuerID = id;
    }
}

```
