# Boost \_ Folks Finance 34076 - \[Smart Contract - Low] Wrong way of deriving message keys using destin

Submitted on Mon Aug 05 2024 08:50:39 GMT-0400 (Atlantic Standard Time) by @IronsideSec for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34076

Report type: Smart Contract

Report severity: Low

Target: <https://sepolia.etherscan.io/address/0x7cdB014Bc73C74Da5b3830eDE6a4494ec52C3738>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Look at the following examples from official wormhole sdk and curvance (crooschain protocol), they use source chain's cctp domain id to derive the message keys, but folks are using the destination chain's cctp domain id.

<https://github.com/wormhole-foundation/wormhole-solidity-sdk/blob/b9e129e65d34827d92fceeed8c87d3ecdfc801d0/src/CCTPBase.sol#L125>

```solidity
            MessageKey(
 CCTPMessageLib.CCTP_KEY_TYPE,
 >>>           abi.encodePacked(getCCTPDomain(wormhole.chainId()), nonce)
 );
```

<https://github.com/curvance/Curvance-CantinaCompetition/blob/76669e36572d9c6803a0e706214a9149d5247586/contracts/architecture/FeeTokenBridgingHub.sol#L151>

```solidity
   IWormholeRelayer.MessageKey[]
            memory messageKeys = new IWormholeRelayer.MessageKey[](1);
 messageKeys[0] = IWormholeRelayer.MessageKey(
            2, // CCTP_KEY_TYPE
 >>>        abi.encodePacked(centralRegistry.cctpDomain(block.chainid), nonce)
 );
```

## Vulnerability Details

`WormholeCCTPAdapter::sendMessage` uses destination chain id's cctp domain id to create the message keys, its actually should use the source chain id's cctp domain. Look at line 270 below using `destinationDomain` to create a message key, the `destinationDomain` is derived from getChainAdapter(message, destinationChainId) where `destinationChainId` is used, but it should have used block.chain id i.e src chain to get the cctp doamin id.

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/bridge/WormholeCCTPAdapter.sol#L121>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/bridge/WormholeCCTPAdapter.sol#L264>

```solidity
WormholeCCTPAdapter

    function sendMessage(Messages.MessageToSend calldata message) external payable onlyBridgeRouter {
        // get chain adapter if available
  >>>   (uint16 wormholeChainId, bytes32 adapterAddress, uint32 cctpDestinationDomain) = getChainAdapter(
            message.destinationChainId
        );
...SNAP....
        bytes memory payloadWithMetadata;
        MessageKey[] memory messageKeys;
...SNAP....
            // burn Circle Token and retrieve info needed to pair with Wormhole message
            uint64 nonce;
            (messageKeys, nonce) = _transferCircleToken(
                extraArgs.amount,
  >>>           cctpDestinationDomain,
                extraArgs.recipient,
                adapterAddress
            );
...SNAP....
    }


250:     function _transferCircleToken(
251:         uint256 amount,
252:         uint32 destinationDomain,
253:         bytes32 receipientAddress,
254:         bytes32 destinationCaller
255:     ) internal returns (MessageKey[] memory, uint64) {
...SNAP....
268:         // return info so can pair Circle Token transfer with Wormhole message
269:         MessageKey[] memory messageKeys = new MessageKey[](1);
272:   >>>   messageKeys[0] = MessageKey(CCTPMessageLib.CCTP_KEY_TYPE, abi.encodePacked(destinationDomain, nonce));
273:         return (messageKeys, nonce);
274:     }


```

## Impact Details

Wrong way of deriving message keys might revert on some messages. Not a loss of funds, but fits the Low severity.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/bridge/WormholeCCTPAdapter.sol#L264>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/bridge/WormholeCCTPAdapter.sol#L121>

## Proof of concept

## Proof of Concept

POC is just to show the flow of call, the revert happening is not due to the wrong cctp domain. Also, the wrong cctip domain doesn't cause loss of funds, its just wrong implemtation in the contract

for POC to work,

1. on `https://github.com/Folks-Finance/folks-finance-xchain-contracts` directory, do `forge i foundry-rs/forge-std --no-commit`,
2. then add `ds-test/=node_modules/ds-test/` to `remappings.txt`,
3. then create a file `Foundry.t.sol` on test/ dirctory.
4. Then run the poc with `forge t --mt testIssue -f https://ethereum-sepolia.rpc.subquery.network/public --fork-block-number 6438899 -vvvv`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

import "../contracts/bridge/BridgeRouterSpoke.sol";
import "../contracts/bridge/WormholeDataAdapter.sol";
import "../contracts/bridge/WormholeCCTPAdapter.sol";
import "../contracts/bridge/libraries/Messages.sol";


contract PoC is Test {
     address from = 0xD5fba05dE4b2d303D03052e8aFbF31a767Bd908e;
    bytes32 accountId = 0xd32cc9b5264dc39d42622492da52b2b8100e6444367e20c9693ce28fe71286be;
    bytes32 loanId = 0xb7a13b4af5fa7d53dd926b64fd7e4f9545840f7775499b90eca6af6f7caba0da;

    address wormholeRelayer = address(0x7B1bD7a6b4E61c2a123AC6BC2cbfC614437D0470);
    BridgeRouterSpoke constant bridgeRouterSpoke = BridgeRouterSpoke(0xBeF7aB7C5e6CeFF384cde460dd20C862047CDFa5);
    WormholeCCTPAdapter constant wormholeCCTPAdapter = WormholeCCTPAdapter(0x7cdB014Bc73C74Da5b3830eDE6a4494ec52C3738);

    function setUp() public {}

    function testIssue() public {
        
        // POC is just to  show the flow of call, the revert happening is not due to the wrong cctp domain.
        // also, the wrong cctip domain doesn't cause loss of funds, its just wrong implemtation in the contract

        Messages.MessageToSend memory send = Messages.MessageToSend(
            Messages.MessageParams(3,0,0, 1946690, 0),
            Messages.convertEVMAddressToGenericAddress(from),
            1,
            0x0000000000000000000000008a81dbf6d6b6a8693181de7ad6ff7f4c47a5b8bd,
            Messages.encodeMessagePayload(
                Messages.MessagePayload({
                    action: Messages.Action(1),
                    accountId: accountId,
                    userAddress: Messages.convertEVMAddressToGenericAddress(from),
                    data: ""
                })
            ),
            15,
            Messages.extraArgsToBytes(Messages.ExtraArgsV1(
                Messages.convertEVMAddressToGenericAddress(0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238), 
                Messages.convertEVMAddressToGenericAddress(from),
                1
            ))
        );

        deal(0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238, address(wormholeCCTPAdapter), 1e18);
        deal(address(this), 1 ether);

        vm.prank(address(bridgeRouterSpoke));
        wormholeCCTPAdapter.sendMessage{value : 0.01 ether}(send);
    }
         
}


```
