# 26479 - \[SC - Insight] ExchangeV cannot be reinitialized after an upgrade

Submitted on Dec 3rd 2023 at 21:46:50 UTC by @Paludo0x for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26479

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

In the event of an upgrade of the ExchangeV3 contract via a proxy, it is not possible to re-initialize the new implementation. This is because the initialize function uses the `onlyWhenUninitialized` modifier.

```
    modifier onlyWhenUninitialized()
    {
        require(
            address(state.loopring) == address(0) && state.merkleRoot == bytes32(0),
            "INITIALIZED"
        );
        _;
    }

```

Before initialization the modifier checks that `address(state.loopring) == address(0) && state.merkleRoot == bytes32(0);` otherwise, the contract has already been initialized. That means that by calling `initialization` after an upgrade of **ExchangeV3** with a contract that uses the same storage slots, the call will fail because these two storage slots values are not empty.

The `initialize` function updates the owner of the implementation, the **Loopring** contract address, the **Merkle trees roots** and the **DOMAIN\_SEPARATOR**.

1. Regarding the owner, there are already dedicated functions in the claimable contract that allows address to be updated.
2. As for Loopring, I cannot find dedicated functions for updating the address. Therefore it is not changeable in case an updated is needed after ExchangeV3 update. This issue should be easily overcome by implementing dedicated functions as per owner updating.
3. Regarding the Merkle roots, I am not able to say whether a reinitialization is ever required after an upgrade, I can't figure out a scenario that the merkle roots need to be reinizialaized.
4. As for the DOMAIN\_SEPARATOR, this depends on the version of the implementation and there are no functions that allow modifying it. DOMAIN\_SEPARATOR hash should be updated once the **Exchange** contract is updated.

## Impact

I believe this is an issuewith a medium impact because the upgradability of ExchangeV3 through a proxy was recently implemented, but this is limited by the bug just described.

## Recommendation

Suggestion is that modifier `onlyWhenUninitialized` checks the **INITIALIZED** state by means of a variable `initialized`. This variable could be reset by a function callable by proxy owner only before calling the initialize function and of course in the same transaction to avoid initialization front running. Alternative is to implement a permissioned functions to upgrade **DOMAIN\_SEPARATOR** hash value (and **Loopring** contract address)

## Proof of concept

The purpose of the Proof of Concept (POC) is to demonstrate whether the initialize function works or not if it is called after an upgrade of implementation contract.

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.6.2 <0.9.0;
pragma abicoder v2;

import {Test, console2} from "forge-std/Test.sol";
import {OwnedUpgradabilityProxy} from "../contracts/thirdparty/proxies/OwnedUpgradabilityProxy.sol";  
import {ExchangeV3} from "../contracts/core/impl/ExchangeV3.sol";  

contract TestPlaygorund is Test {

    address constant contractsDeployer = 0xacD3A62F3eED1BfE4fF0eC8240d645c1F5477F82;
    OwnedUpgradabilityProxy constant excV3ProxyInterfDeployed = OwnedUpgradabilityProxy(0x9C07A72177c5A05410cA338823e790876E79D73B);
    ExchangeV3 constant excV3ProxyImplDeployed = ExchangeV3(0x9C07A72177c5A05410cA338823e790876E79D73B);
    
    ExchangeV3 newExchangeImplementation = new ExchangeV3();

    function test_initialize_upgraded_implementation() public {

        
        address loopring =0x9385aCd9d78dFE854c543294770d0C94c2B07EDC;
        address owner = contractsDeployer;
        bytes32 genesisMerkleRoot = 0x03e1788bf14436c39a3841ae888ffb3e6ec8405bc2773afa28b6d4dfc309cf19;
        bytes32 genesisMerkleAssetRoot = 0x071c8b14d71d432750479f5fe6e08abe1ec04712835a83cdf84d0483b9382ae8;
        
        console2.log("Implementation address before upgrade:", excV3ProxyInterfDeployed.implementation() );

        address proxyOwner = excV3ProxyInterfDeployed.proxyOwner();
        vm.startPrank(proxyOwner);
        excV3ProxyInterfDeployed.upgradeTo(address(newExchangeImplementation));
        console2.log("Implementation address after upgrade:", excV3ProxyInterfDeployed.implementation() );

        vm.expectRevert("INITIALIZED");
        excV3ProxyImplDeployed.initialize(loopring, owner, genesisMerkleRoot, genesisMerkleAssetRoot);
        vm.stopPrank();
        
    }

}

```
