# 90159 sc high redundant state update in rescuewithdrawfromblocklisted permanently freezes funds for unblocklisted users withdrawing again

**Submitted on Aug 23rd 2026 at 11:43:18 UTC by @shekdo for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90159
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

In `FirelightVault`, `rescueWithdrawFromBlocklisted` redundantly sets `isWithdrawClaimed` to true when moving pending withdrawals from a blocklisted user. Because withdrawal requests are always scheduled for the next period, this rescue operation flags a future period as claimed. If the user is subsequently unblocklisted and makes a new withdrawal targeting that same period, their shares are burned and the assets enter `pendingWithdrawAssets`, but both `claimWithdraw` and further administrative rescues will revert. This permanently freezes the user's new withdrawal with no protocol recovery path.

## Vulnerability Details

When the `RESCUER_ROLE` calls `rescueWithdrawFromBlocklisted` to move pending withdrawals, it zeroes out `withdrawSharesOf[periods[i]][from]` and sets `isWithdrawClaimed[periods[i]][from] = true`:

```solidity
// contracts/core/FirelightVault.sol:860-873
            withdrawSharesOf[periods[i]][to] += _withdrawOf;
            withdrawSharesOf[periods[i]][from] = 0;
            isWithdrawClaimed[periods[i]][from] = true;
```

The function places no constraints on `periods[i]` other than the user having a non-zero withdrawal balance for it:

```solidity
// contracts/core/FirelightVault.sol:861-867
            if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);
```

Because all withdrawal requests are automatically scheduled for `currentPeriod() + 1` (with no ability for the caller to choose the period), the rescued period is often a future period:

```solidity
// contracts/core/FirelightVault.sol:1006-1010
        uint256 period = currentPeriod() + 1;
```

If the user is unblocklisted and makes another withdrawal before the rescued period starts, the new request targets the exact same future period. The user's shares are burned immediately within `_requestWithdraw`:

```solidity
// contracts/core/FirelightVault.sol:1021-1026
        _update(owner, address(0), shares);
```

The withdrawn assets are added to `pendingWithdrawAssets`, meaning they remain counted in the vault's pending-withdrawal accounting forever:

```solidity
// contracts/core/FirelightVault.sol:1015-1020
        pendingWithdrawAssets += assets;
```

When the user attempts to claim the withdrawal with `claimWithdraw(period)`, the transaction reverts with `AlreadyClaimedPeriod` because the prior rescue set the flag. This occurs before the function reads the caller's share balance:

```solidity
// contracts/core/FirelightVault.sol:788-804
        if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

The `RESCUER_ROLE` cannot recover the funds either, as `rescueWithdrawFromBlocklisted` also reverts if `isWithdrawClaimed` is true:

```solidity
// contracts/core/FirelightVault.sol:860-867
            if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
```

The `isWithdrawClaimed` flag is a storage mapping that cannot be cleared by any function in the in-scope sources. An upgrade deleting the offending line prevents new cases but releases no already-trapped funds. The write itself is redundant, as the zeroing of `withdrawSharesOf` already ensures a double-claim reverts with `NoWithdrawalAmount`:

```solidity
// contracts/core/FirelightVault.sol:795-806
        if (assets == 0) revert NoWithdrawalAmount(period);
```

## Impact Details

`rescueWithdrawFromBlocklisted` stamps `isWithdrawClaimed` on a period that has not started, which is the exact period every new withdrawal request targets. A user who is cleared and withdraws again inside that period has their shares burned and their assets moved into `pendingWithdrawAssets`, where `claimWithdraw` and any further rescue both revert forever. The stamp is storage and nothing in the contracts ever clears it, so even shipping a fix does not release funds already trapped. The amount is bounded only by the user's own share balance, not by the rescued amount. While the bug requires administrative compliance actions to trigger, it bypasses the intended protocol constraint that unblocklisting a user restores their ability to withdraw normally, fulfilling the program's privileged-actor carve-out.

## References

<https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>

## Proof of Concept

## Proof of Concept

The test deploys the `FirelightVault` implementation behind an `ERC1967Proxy` using the live vault's 1-day period. It replays the scenario: a user withdraws to a future period, is blocklisted, their withdrawal is rescued, they are unblocklisted, and they redeem their remaining position into the same period. After the period elapses, the user's attempt to `claimWithdraw` reverts with `AlreadyClaimedPeriod`, and a second rescue attempt reverts with the same error, proving the funds are permanently stuck in `pendingWithdrawAssets`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

import {FirelightVault} from "../src/core/FirelightVault.sol";
import {MockERC20} from "../src/MockERC20.sol";

/// @notice rescueWithdrawFromBlocklisted marks a not-yet-started period claimed for the
///         rescued account. Because every withdrawal request is scheduled into currentPeriod()+1,
///         a user who is un-blocklisted and withdraws again inside the same period lands in that
///         exact poisoned period. Their shares are burned, the assets enter pendingWithdrawAssets,
///         and every path out is closed: claimWithdraw reverts AlreadyClaimedPeriod, and a further
///         rescue reverts with the same error.
contract POC_Test is Test {
    uint256 internal constant DECIMALS = 6;
    uint256 internal constant DEPOSIT_LIMIT = 1_000_000 * 10 ** DECIMALS;
    uint256 internal constant INITIAL_DEPOSIT = 100 * 10 ** DECIMALS;
    uint256 internal constant FIRST_WITHDRAW = 40 * 10 ** DECIMALS;

    /// @dev Matches the live Flare vault's configuration: currentPeriodConfiguration() on
    ///      0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3 returns duration 0x15180 == 86400.
    uint48 internal constant PERIOD = 1 days;

    FirelightVault internal vault;
    MockERC20 internal token;

    address internal userA = makeAddr("userA");
    address internal safeAddr = makeAddr("safeAddr");

    function setUp() public {
        token = new MockERC20("USDC", "USDC", uint8(DECIMALS));

        FirelightVault implementation = new FirelightVault();

        FirelightVault.InitParams memory params = FirelightVault.InitParams({
            defaultAdmin: address(this),
            limitUpdater: address(this),
            blocklister: address(this),
            pauser: address(this),
            periodConfigurationUpdater: address(this),
            rescuer: address(this),
            depositLimit: DEPOSIT_LIMIT,
            periodConfigurationDuration: PERIOD
        });

        bytes memory initData = abi.encodeCall(
            FirelightVault.initialize,
            (IERC20(address(token)), "Firelight Vault", "flVault", abi.encode(params))
        );

        vault = FirelightVault(address(new ERC1967Proxy(address(implementation), initData)));

        token.mint(userA, INITIAL_DEPOSIT);
        vm.prank(userA);
        token.approve(address(vault), type(uint256).max);
    }

    function test_exploit() public {
        // ---------------------------------------------------------------- setup
        vm.prank(userA);
        vault.deposit(INITIAL_DEPOSIT, userA);

        uint256 periodAtRescue = vault.currentPeriod();
        uint256 targetPeriod = periodAtRescue + 1;

        // A withdrawal request is always scheduled into currentPeriod() + 1. The caller does not
        // choose the period.
        vm.prank(userA);
        vault.withdraw(FIRST_WITHDRAW, userA, userA);
        assertGt(vault.withdrawSharesOf(targetPeriod, userA), 0, "first request scheduled");

        // ------------------------------------------------- documented compliance flow
        vault.addToBlocklist(userA);

        uint256[] memory periods = new uint256[](1);
        periods[0] = targetPeriod;
        vault.rescueWithdrawFromBlocklisted(userA, safeAddr, periods);

        // The rescue moved the balance AND stamped a period that has not started yet.
        assertEq(vault.withdrawSharesOf(targetPeriod, userA), 0, "rescued balance zeroed");
        assertGt(vault.withdrawSharesOf(targetPeriod, safeAddr), 0, "balance moved to safe address");
        assertTrue(vault.isWithdrawClaimed(targetPeriod, userA), "future period stamped as claimed");

        vault.removeFromBlocklist(userA);
        assertEq(vault.currentPeriod(), periodAtRescue, "still inside the same period");

        // ------------------------------------------------------------- the trap
        // The cleared user withdraws their ENTIRE remaining position. It is scheduled into the
        // same, already-stamped period - nothing lets them pick another one.
        uint256 remainingShares = vault.balanceOf(userA);
        assertGt(remainingShares, 0, "user still holds a position");

        vm.prank(userA);
        vault.redeem(remainingShares, userA, userA);

        assertEq(vault.balanceOf(userA), 0, "shares burned at request time");
        uint256 strandedAssets = vault.withdrawalsOf(targetPeriod, userA);
        assertGt(strandedAssets, 0, "vault records assets owed to the user");

        // ------------------------------------------------------- period elapses
        vm.warp(block.timestamp + 2 * uint256(PERIOD) + 1);
        assertGt(vault.currentPeriod(), targetPeriod, "target period is now claimable");

        // The user cannot claim assets they never received, because of a flag set by a rescue of a
        // DIFFERENT withdrawal.
        vm.prank(userA);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, targetPeriod));
        vault.claimWithdraw(targetPeriod);
        assertEq(token.balanceOf(userA), 0, "user recovers nothing");

        // The rescue itself worked as designed - the destination claims normally. The defect is
        // confined to the redundant stamp written on the rescued account.
        vm.prank(safeAddr);
        uint256 rescuedOut = vault.claimWithdraw(targetPeriod);
        assertGt(rescuedOut, 0, "rescue destination claims normally");
        assertEq(token.balanceOf(safeAddr), rescuedOut, "rescue destination is paid");

        // Everything still sitting in the vault's pending-withdrawal accounting is the user's
        // stranded position, and it can never leave.
        assertApproxEqAbs(
            vault.pendingWithdrawAssets(), strandedAssets, 2, "residual pending == stranded amount"
        );
        assertEq(vault.withdrawalsOf(targetPeriod, userA), strandedAssets, "still recorded as owed");

        // The last recovery path is closed too: re-blocklisting and rescuing again reverts on the
        // same flag.
        vault.addToBlocklist(userA);
        vm.expectRevert(abi.encodeWithSelector(FirelightVault.AlreadyClaimedPeriod.selector, targetPeriod));
        vault.rescueWithdrawFromBlocklisted(userA, safeAddr, periods);
    }
}
```
