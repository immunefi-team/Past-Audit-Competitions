# 69245 sc insight no view function to compute current claimable amounts

**Submitted on Mar 13th 2026 at 19:17:53 UTC by @ZenHunter for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69245
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**

## Description

### Brief/Intro

`Staking` exposes `getUserStake` to read raw `UserStake` storage fields, but provides no external view function that computes how much a user can currently withdraw. The internal accrual logic lives entirely in `_getAccrued`, an `internal pure` function that is inaccessible to any external caller. Every off-chain consumer — frontend, monitoring bot, aggregator, or composing smart contract — must independently replicate the accrual formula to compute claimable amounts.

### Vulnerability Details

The accrual formula is implemented once, in `_getAccrued`:

```solidity
// src/Staking.sol#L332-L334
function _getAccrued(uint256 amount, uint256 duration, uint256 elapsed) internal pure returns (uint256) {
    return Math.mulDiv(amount, Math.min(elapsed, duration), duration);
}
```

It is called exclusively inside `_withdraw`, which also applies the pre-conditions that gate a valid withdrawal:

```solidity
// src/Staking.sol#L304-L330
function _withdraw(uint8 stakeIndex) internal {
    if (stakeIndex >= userStakes[msg.sender].length) revert StakeNotFound();

    UserStake storage userStake = userStakes[msg.sender][stakeIndex];
    if (block.timestamp <= userStake.unlockTime) {
        revert RewardsNotAvailableYet(uint64(block.timestamp), userStake.unlockTime);
    }
    if (userStake.claimedAmount + userStake.claimedReward >= userStake.amount + userStake.reward) {
        revert AlreadyWithdrawn(stakeIndex);
    }

    uint256 accruedAmount =
        _getAccrued(userStake.amount, userStake.unlockDuration, block.timestamp - userStake.unlockTime);
    uint256 accruedReward =
        _getAccrued(userStake.reward, userStake.unlockDuration, block.timestamp - userStake.unlockTime);

    uint256 amountToClaim = accruedAmount - userStake.claimedAmount;
    uint256 rewardToClaim = accruedReward - userStake.claimedReward;
    ...
}
```

`IStakingV1` exposes only raw storage reads:

```solidity
// src/interfaces/IStakingV1.sol#L111-L114
function getUserStakes(address user) external view returns (UserStake[] memory);
function getUserStake(address user, uint8 stakeIndex) external view returns (UserStake memory);
```

To derive the currently claimable amounts from these, an external caller must independently reconstruct the full computation — including the pre-conditions and the `Math.mulDiv` + `Math.min` formula:

```solidity
UserStake memory s = IStakingV1(staking).getUserStake(user, stakeIndex);

// Caller must replicate every step:
if (block.timestamp <= s.unlockTime) { /* not yet claimable */ }
if (s.claimedAmount + s.claimedReward >= s.amount + s.reward) { /* already fully withdrawn */ }

uint256 elapsed      = block.timestamp - s.unlockTime;
uint256 clampedElapsed = elapsed < s.unlockDuration ? elapsed : s.unlockDuration;
uint256 accruedAmount  = (s.amount * clampedElapsed) / s.unlockDuration; // simplified; real uses mulDiv
uint256 accruedReward  = (s.reward * clampedElapsed) / s.unlockDuration;
uint256 claimableAmount = accruedAmount - s.claimedAmount;
uint256 claimableReward = accruedReward - s.claimedReward;
```

This is non-trivial to replicate correctly — `Math.mulDiv` uses full 512-bit intermediate multiplication to avoid overflow — and there is no interface-level contract that these externally-replicated formulas will remain valid across contract upgrades.

### Impact Details

**Impact category:** Architectural Decentralization and Composability

### References

* Accrual logic: `src/Staking.sol#L332–L334`
* Only call site: `src/Staking.sol#L315–L321`
* Interface view functions: `src/interfaces/IStakingV1.sol#L111–L114`

## Recommendation

Add `getClaimable` and `getClaimableAll` to `IStakingV1` and implement them in `Staking`. Storage fields are cached into locals to avoid repeated warm SLOADs. `getClaimableAll` delegates to `getClaimable` to avoid duplicating the per-stake logic:

```solidity
// IStakingV1.sol
function getClaimable(address user, uint8 stakeIndex)
    external view returns (uint256 claimableAmount, uint256 claimableReward);

function getClaimableAll(address user)
    external view returns (uint256 totalClaimableAmount, uint256 totalClaimableReward);
```

```solidity
// Staking.sol
function getClaimable(address user, uint8 stakeIndex)
    external
    view
    returns (uint256 claimableAmount, uint256 claimableReward)
{
    if (stakeIndex >= userStakes[user].length) revert StakeNotFound();
    UserStake storage userStake = userStakes[user][stakeIndex];

    // Cache storage fields into locals — avoids repeated warm SLOADs.
    uint256 amount         = userStake.amount;
    uint256 reward         = userStake.reward;
    uint256 claimedAmount  = userStake.claimedAmount;
    uint256 claimedReward  = userStake.claimedReward;
    uint64  unlockTime     = userStake.unlockTime;
    uint64  unlockDuration = userStake.unlockDuration;

    if (block.timestamp <= unlockTime) return (0, 0);
    if (claimedAmount + claimedReward >= amount + reward) return (0, 0);

    uint256 elapsed       = block.timestamp - unlockTime;
    uint256 accruedAmount = _getAccrued(amount, unlockDuration, elapsed);
    uint256 accruedReward = _getAccrued(reward, unlockDuration, elapsed);

    claimableAmount = accruedAmount - claimedAmount;
    claimableReward = accruedReward - claimedReward;
}

function getClaimableAll(address user)
    external
    view
    returns (uint256 totalClaimableAmount, uint256 totalClaimableReward)
{
    uint256 count = userStakes[user].length;
    for (uint8 i = 0; i < count; ++i) {
        (uint256 ca, uint256 cr) = this.getClaimable(user, i);
        totalClaimableAmount += ca;
        totalClaimableReward += cr;
    }
}
```

This surfaces the authoritative computation through the interface with no additional logic risk, and ensures any future formula change is automatically reflected to all callers.

## Proof of Concept

**File:** `test/NoClaimableView.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {Test, console} from "forge-std/Test.sol";
import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";

contract Token is ERC20Permit {
    constructor() ERC20Permit("T") ERC20("T", "T") {}
    function mint(address to, uint256 amount) external { _mint(to, amount); }
}

contract StakingWithClaimable is Staking {
    constructor(address _admin, address _manager, address _pauser, address _token)
        Staking(_admin, _manager, _pauser, _token) {}

    function getClaimable(address user, uint8 stakeIndex)
        external view returns (uint256 claimableAmount, uint256 claimableReward)
    {
        if (stakeIndex >= userStakes[user].length) revert StakeNotFound();
        UserStake storage userStake = userStakes[user][stakeIndex];

        uint256 amount         = userStake.amount;
        uint256 reward         = userStake.reward;
        uint256 claimedAmount  = userStake.claimedAmount;
        uint256 claimedReward  = userStake.claimedReward;
        uint64  unlockTime     = userStake.unlockTime;
        uint64  unlockDuration = userStake.unlockDuration;

        if (block.timestamp <= unlockTime) return (0, 0);
        if (claimedAmount + claimedReward >= amount + reward) return (0, 0);

        uint256 elapsed       = block.timestamp - unlockTime;
        uint256 accruedAmount = _getAccrued(amount, unlockDuration, elapsed);
        uint256 accruedReward = _getAccrued(reward, unlockDuration, elapsed);

        claimableAmount = accruedAmount - claimedAmount;
        claimableReward = accruedReward - claimedReward;
    }

    function getClaimableAll(address user)
        external view returns (uint256 totalClaimableAmount, uint256 totalClaimableReward)
    {
        uint256 count = userStakes[user].length;
        for (uint8 i = 0; i < count; ++i) {
            (uint256 ca, uint256 cr) = this.getClaimable(user, i);
            totalClaimableAmount += ca;
            totalClaimableReward += cr;
        }
    }
}

contract NoClaimableViewTest is Test {
    uint64  constant STAKING_DURATION = uint64(365 days);
    uint64  constant UNLOCK_DURATION  = uint64(30 days);
    uint32  constant APR_BPS          = 1_000;
    uint256 constant STAKE_AMOUNT     = 1_000e18;
    uint256 constant PREFUND          = 1_000_000e18;

    address admin = address(0x10); address manager = address(0x11);
    address pauser = address(0x12); address alice = address(0x13);

    Token                token;
    StakingWithClaimable stakingFixed;
    IStakingV1.StakeParams stakeParams;

    function setUp() public {
        vm.warp(10 * 365 days);
        token        = new Token();
        stakingFixed = new StakingWithClaimable(admin, manager, pauser, address(token));

        stakeParams = IStakingV1.StakeParams({
            maxStakingDurationSeconds: STAKING_DURATION,
            maxUnlockDurationSeconds:  UNLOCK_DURATION,
            minAprBps: APR_BPS, referrer: address(0)
        });

        vm.prank(manager);
        stakingFixed.addStakingPeriod(10_000_000e18, STAKING_DURATION, UNLOCK_DURATION, APR_BPS, true);

        token.mint(address(stakingFixed), PREFUND);
        token.mint(alice, STAKE_AMOUNT * 2);
        vm.startPrank(alice);
        token.approve(address(stakingFixed), type(uint256).max);
        stakingFixed.stake(0, STAKE_AMOUNT, stakeParams);
        vm.stopPrank();

        // Advance past unlockTime and halfway through the unlock period.
        vm.warp(block.timestamp + STAKING_DURATION + UNLOCK_DURATION / 2);
    }

    // getClaimable() returns values that exactly match what withdraw() transfers.
    function test_GetClaimableMatchesWithdraw() public {
        (uint256 claimableAmount, uint256 claimableReward) = stakingFixed.getClaimable(alice, 0);

        console.log("getClaimable() claimableAmount:", claimableAmount);
        console.log("getClaimable() claimableReward:", claimableReward);

        uint256 balanceBefore = token.balanceOf(alice);
        vm.prank(alice);
        stakingFixed.withdraw(0);
        uint256 received = token.balanceOf(alice) - balanceBefore;

        console.log("withdraw() transferred:", received);
        assertEq(received, claimableAmount + claimableReward);
    }

    // getClaimableAll() equals the sum of per-stake getClaimable() calls.
    function test_GetClaimableAllMatchesSumOfGetClaimable() public {
        token.mint(alice, STAKE_AMOUNT);
        vm.prank(alice);
        stakingFixed.stake(0, STAKE_AMOUNT, stakeParams);

        (uint256 ca0, uint256 cr0) = stakingFixed.getClaimable(alice, 0);
        (uint256 ca1, uint256 cr1) = stakingFixed.getClaimable(alice, 1);
        (uint256 totalAmount, uint256 totalReward) = stakingFixed.getClaimableAll(alice);

        console.log("getClaimable(0) amount:", ca0, "reward:", cr0);
        console.log("getClaimable(1) amount:", ca1, "reward:", cr1);
        console.log("getClaimableAll  amount:", totalAmount, "reward:", totalReward);

        assertEq(totalAmount, ca0 + ca1);
        assertEq(totalReward, cr0 + cr1);
    }
}
```

**Command:**

```bash
forge test --match-path "test/NoClaimableView.t.sol" -vv
```

**Output:**

```
Ran 2 tests for test/NoClaimableView.t.sol:NoClaimableViewTest
[PASS] test_GetClaimableAllMatchesSumOfGetClaimable() (gas: 231944)
Logs:
  getClaimable(0) amount: 500000000000000000000 reward: 50000000000000000000
  getClaimable(1) amount: 0 reward: 0
  getClaimableAll  amount: 500000000000000000000 reward: 50000000000000000000

[PASS] test_GetClaimableMatchesWithdraw() (gas: 136231)
Logs:
  getClaimable() claimableAmount: 500000000000000000000
  getClaimable() claimableReward: 50000000000000000000
  withdraw() transferred: 550000000000000000000

Suite result: ok. 2 passed; 0 failed; 0 skipped
```
