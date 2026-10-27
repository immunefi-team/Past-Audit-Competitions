# 31416 - \[SC - Insight] Impossible to set boostMultiplier to MIN\_BOOST

Submitted on May 18th 2024 at 20:29:44 UTC by @RNemes for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31416

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `setBoostMultiplier` function in the contract contains a logic flaw where the check for `_boostMultiplier` being greater than `MIN_BOOST` inadvertently prevents setting the `boostMultiplier` to `MIN_BOOST`, which is defined as zero. This prevents users from setting the `boostMultiplier` to its minimum allowed value, potentially leading to unexpected behavior or inability to achieve certain intended configurations.

## Vulnerability Details

The vulnerability lies in the `setBoostMultiplier` function's requirement check:

```solidity
require(_boostMultiplier <= MAX_BOOST && _boostMultiplier > MIN_BOOST, "Boost multiplier is out of bounds");
```

Given the definition:

```solidity
uint256 internal constant MIN_BOOST = 0;
```

The condition `_boostMultiplier > MIN_BOOST` translates to `_boostMultiplier > 0`, which means `_boostMultiplier` must be greater than zero. Therefore, it's impossible to set `_boostMultiplier` to zero, even though zero is defined as the minimum boost allowed (`MIN_BOOST`). This restriction can prevent the proper functioning of the contract if setting the `boostMultiplier` to zero is a required use case.

## Impact Details

The impact of this vulnerability is primarily operational rather than financial. If the `boostMultiplier` needs to be set to zero for certain operations or configurations, the current implementation will prevent this, potentially leading to incorrect contract behavior or inability to revert to a default state. This could disrupt contract functionality and the ability of the admin to control the `boostMultiplier` as intended. In scenarios where setting the multiplier to zero is necessary for security or protocol reasons, this bug could pose a more significant risk.

## Proof of Concept

Add the following failing test to `src/test/Voting.t.sol`

```solidity
  function testSetBoostMultiplierToMinValue() public {
        hevm.prank(address(timelockExecutor));
        voter.setAdmin(devmsig);

        hevm.startPrank(devmsig);

        voter.acceptAdmin();

        voter.setBoostMultiplier(0);
    }
```

```bash
Failing tests:
Encountered 1 failing test in src/test/Voting.t.sol:VotingTest
[FAIL. Reason: revert: Boost multiplier is out of bounds] testSetBoostMultiplierToMinValue() (gas: 26707)
```
