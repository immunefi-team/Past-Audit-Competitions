# \#98620 [SC-Medium] Incorrect fee accounting in the close planner nets the treasury's execution fee into the VPI rebate, so the protocol permanently loses all or part of the fee on skew-reducing closes

**Submitted on Nov 9th 2026 at 03:38:48 UTC by @godwinudo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98620
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Every trade on Plether Perps pays a 4 bps execution fee to the protocol treasury, on opens and on closes. Trades that reduce the book's skew also earn a VPI rebate from the pool. When a close earns a rebate at least as large as the fee, `CfdEnginePlanLib._planCloseActionSettlement` nets the two amounts together. The pool pays the trader `rebate − fee`, the treasury is credited zero, and the fee stays in the LP pool. The trader is still charged the fee; it just never reaches the treasury. With the release parameters, this happens on pretty much every close on the heavier side of the book once the skew is around 4% of pool depth or more. The open path handles the same netting correctly and pays the treasury the full fee from pool cash.

## Vulnerability Details

Plether settles every open and close as price PnL plus a set of separate action items: a signed VPI (virtual price impact, a charge when you push skew up and a rebate when you bring it down), a flat execution fee, and, where they apply, unpaid carry and the frozen-market close spread. 

The protocol is very clear about who owns the fee. The whitepaper says in its risk-admission section that *"the protocol execution fee belongs to treasury rather than LP assets"* (WHITEPAPER.md:411-413). Its fee table lists the execution fee as *"Protocol service revenue on traded notional — Treasury, only when cash credited"* 
Also, the public docs at docs.plether.com ("Trading costs, fees, carry and VPI") go further: *"Opening and closing are separate executions, so a round trip normally pays the fee twice"*, and *"Only a fee collected from trader collateral or funded from available pool cash is credited to treasury margin."* So the fee belongs to the treasury, it is charged on both legs, and when it can't be collected straight from the trader, free pool cash is supposed to fund it.

The open path does exactly that. In `OpenAccountingLib`, VPI and the fee are folded into one signed trade cost:

```solidity
// packages/perps/src/libraries/OpenAccountingLib.sol:134-135
state.executionFeeAssets = (state.notionalAssets * inputs.executionFeeBps) / 10_000;
state.tradeCostAssets = state.vpiAssets + int256(state.executionFeeAssets);
```

If a skew-reducing open earns a rebate bigger than the fee, `tradeCostAssets` goes negative and the pool pays the trader the net (`poolRebatePayoutAssets = -tradeCostAssets`, CfdEnginePlanLib.sol:333). The planner still carries the full fee forward (`delta.executionFeeAssets = openState.executionFeeAssets;`, CfdEnginePlanLib.sol:399), and the settlement sidecar tops the treasury up with whatever the trader didn't pay directly:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:498  (executeOpen)
_settleProtocolFeeTopUp(host, delta.executionFeeAssets, protocolFeeCreditedAssets);
```

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:905-926
function _settleProtocolFeeTopUp(ICfdEngineSettlementHost host, uint256 amountAssets, uint256 clearinghouseCreditedAssets) private {
    ...
    uint256 poolFundedAssets = amountAssets - clearinghouseCreditedAssets;
    ...
    uint256 freeCashAssets = pool.getSettlementFreeAssets();
    uint256 topUpAssets = poolFundedAssets < freeCashAssets ? poolFundedAssets : freeCashAssets;
    ...
    pool.payOut(host.clearinghouse(), topUpAssets);
    IMarginClearinghouse(host.clearinghouse()).settleAssets(host.protocolTreasury(), int256(topUpAssets));
}
```

So on an open, as long as the pool has the free cash, it pays out `|VPI|` in total: `|VPI| − fee` goes to the trader and `fee` goes to the treasury. That is the correct split, and it's the policy the team confirmed when they acknowledged Cecuro LOW-26 ("retaining partial protocol-fee funding … a top-up capped by post-open settlement-free pool cash").

The close path is where it breaks. `CloseAccountingLib.buildCloseState` computes the close's signed VPI, clamped so a position can never get back more than it paid over its lifetime, and the full fee on the closed notional:

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:108-121
state.vpiDeltaAssets = CfdMath.calculateVPI(inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor);
...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
uint256 notionalAssets = (inputs.sizeDelta * inputs.oraclePrice) / CfdMath.ASSETS_TO_TOKEN_SCALE;
state.executionFeeAssets = (notionalAssets * inputs.executionFeeBps) / 10_000;
```

Then `_planCloseActionSettlement` folds everything into one signed "action amount" and decides how much of the fee the treasury actually gets:

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:1036-1074
function _planCloseActionSettlement(...) private pure {
    int256 actionNetAssets = cs.vpiDeltaAssets
        + int256(cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets);
    uint256 vpiClawbackWithheldAssets;
    if (actionNetAssets > 0) {
        delta.actionChargeAssessedAssets = uint256(actionNetAssets);
        ... // withhold from gain / collect from account
    } else if (actionNetAssets < 0) {
        delta.actionRebateAssets = uint256(-actionNetAssets);
    }

    delta.netReleasedMarginAssets = delta.unlockMarginAssets - delta.actionChargeFromReleasedMarginAssets;
    uint256 nonClawbackWithheldAssets = delta.actionChargeWithheldAssets - vpiClawbackWithheldAssets;
    uint256 nonVpiReserveCollectedAssets = delta.actionChargeCollectedAssets - delta.vpiRebateReserveConsumedAssets;
    uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;
    delta.executionFeeAssets =
        cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;
    ...
}
```

Walk through a normal live-market close that reduces skew. There's no frozen spread outside oracle-frozen windows, and carry has normally already been settled from margin, so the action amount is basically `VPI + fee`. The rebate is bigger than the fee, so `actionNetAssets < 0` and we land in the `else if` branch. Nothing is withheld and nothing is collected, so `actionChargeWithheldAssets` and `actionChargeCollectedAssets` are both zero, `executionFeeEligibleAssets` is zero, and `delta.executionFeeAssets` is capped down to **zero**. The pool is told to pay a rebate of `|VPI| − fee`. That number already has the fee taken out, so the trader really did pay it. Back in `_planIsolatedCloseSettlement`, the treasury top-up is sized off that zeroed number:

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:984-987
uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets;
freePoolCashAssets =
    _freePoolCashForProtocolFee(snap, delta, poolCashAfterCollectionsAssets, claimsAfterPriceNettingAssets);
delta.protocolFeeTopUpAssets = feeTopUpAssets < freePoolCashAssets ? feeTopUpAssets : freePoolCashAssets;
```

`feeTopUpAssets` is `0 − 0 = 0`. The sidecar faithfully executes the plan (`_settleProtocolFeeTopUp(host, protocolFeeCreditedAssets + delta.protocolFeeTopUpAssets, protocolFeeCreditedAssets)` at CfdEngineSettlementSidecar.sol:626), and the treasury gets nothing, no matter how much free cash the pool has. The pool pays out `|VPI| − fee` instead of `|VPI|`, so the fee simply stays in LP assets. When the rebate is positive but smaller than the fee, the same cap gives the treasury only `fee − rebate`. In general, on a live close the treasury loses `min(fee, rebate)` every time the close earns a rebate.

The real issue is that the cap mixes up two different things. "The trader couldn't pay" is the legitimate full-close waiver, where nobody has the money. "The trader paid through a smaller rebate" means the money is sitting in the pool. The close planner treats the second case like the first, and the open path doesn't.

Nobody has to do anything malicious for this to happen; normal trading triggers it. With the release parameters (`vpiFactor = 0.01e18`, `executionFeeBps = 4`, DEPLOYMENT.md), the VPI potential is `C(S) = 0.01·S² / (2D)`, where `S` is the dollar skew and `D` is pool depth. Closing `n` dollars of notional on the heavy side earns a rebate of `0.01·n·(2S − n)/(2D)` and costs a fee of `0.0004·n`. The rebate is at least the fee whenever `S − n/2 ≥ 4% of D`, i.e. whenever the skew at the middle of the close is at least 4% of pool depth. The open skew cap is 40% of depth, so once the book leans even modestly one way, every close on the heavy side wipes out its whole treasury fee. The lifetime clamp rarely helps. It only caps the rebate at the VPI the position paid when it opened, and to save the fee that entry VPI would have to be smaller than the fee. That only happens if the position was opened while the mid-trade skew was under about 4% of depth.

## Impact Details

The protocol treasury permanently loses execution-fee revenue it has earned and that the trader has actually paid. The money is moved to the LPs. The treasury is a distinct, named fee recipient in the program's trust model ("Protocol treasury … receive fees"). That makes this a systematic diversion of the protocol's earned yield to another party, and it's why I've classified it as Theft of unclaimed yield: the fee is owed to the treasury at settlement, it is never credited, and it is irreversibly handed to LP NAV.

## Proof of Concept

## Proof of Concept

Create a file at `test/audit/CloseFeeRebateNettingE2E.t.sol` (repo root), paste in the code below, and run with:

```bash
forge test --match-path test/audit/CloseFeeRebateNettingE2E.t.sol -vvv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {DeployPerpsArbitrumSepolia} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {MockPyth} from "../mocks/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {Test} from "forge-std/Test.sol";

/// @dev Mirrors the Arbitrum precompile's block-number view inside the local test EVM (external dependency).
contract E2EArbSys {

    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }

}

/// @notice End-to-end PoC: a voluntary close that earns a VPI rebate larger than the 4 bps execution fee credits the
///         protocol treasury nothing, while the trader is still charged the fee and the pool keeps it.
/// @dev Stack = the checked-in release deploy script (`DeployPerpsArbitrumSepolia.run()`), release parameters
///      (DEPLOYMENT.md "Initial release parameters"). Only external code is replaced: Pyth (faithful MockPyth etched at
///      the pinned release address) and the ArbSys precompile. Owner actions are the documented bootstrap
///      (1 USDG seed per tranche + `activateTrading`, DEPLOYMENT.md). Every other step is a public-interface call.
contract CloseFeeRebateNettingE2ETest is Test {

    // ----- deployment / environment (same values as test/scripts/MockUSDGPerpsLifecycle.t.sol) -----
    address internal constant RELEASE_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    address internal constant ARBSYS = address(100);
    uint256 internal constant ARBITRUM_SEPOLIA_CHAIN_ID = 421_614;
    uint256 internal constant DEPLOYER_KEY = 0xA11CE;
    uint256 internal constant MONDAY_MORNING = 1_709_532_000; // live FX session
    uint256 internal constant SEED_ASSETS = 1e6; // DEPLOYMENT.md: Senior / Junior seed 1e6 each

    // ----- actors (fresh, unprivileged) -----
    address internal constant LP = address(0x1F);
    address internal constant TRADER = address(0x7ADE);
    address internal constant KEEPER = address(0xB07);

    // ----- scenario -----
    uint256 internal constant LP_ASSETS = 1_000_000e6;
    uint256 internal constant TRADER_ASSETS = 10_000e6;
    uint256 internal constant BASKET_PRICE = 1.2e8; // != 1.00
    uint256 internal constant POSITION_SIZE = 150_000e18; // 1,500 lots -> 180,000 USDG notional at 1.20
    uint256 internal constant OPEN_MARGIN = 5000e6;
    uint256 internal constant COMMIT_TO_EXECUTE_SECONDS = 7;
    uint256 internal constant COMMIT_TO_EXECUTE_BLOCKS = 2;
    uint256 internal constant HOLD_SECONDS = 60;

    // ----- release economics (DEPLOYMENT.md) -----
    uint256 internal constant EXECUTION_FEE_BPS = 4;
    uint256 internal constant VPI_DENOMINATOR = 200; // C(S) = vpiFactor * S^2 / (2 D), vpiFactor = 0.01
    uint256 internal constant BASE_CARRY_BPS = 500;
    uint256 internal constant BPS = 10_000;
    uint256 internal constant YEAR = 365 days;
    uint256 internal constant SIZE_TO_NOTIONAL = 1e20; // size(18dp) * price(8dp) / 1e20 = USDG(6dp)
    uint256 internal constant VPI_ROUNDING_ATOMS = 2;

    // ----- trader-pinned bounds -----
    uint256 internal constant MAX_BOUNTY = 100e6;
    uint256 internal constant MAX_NOTIONAL = 200_000e6;
    uint256 internal constant MAX_DEBIT = 6000e6;
    uint256 internal constant MAX_ACTION_CHARGE = 500e6;
    uint256 internal constant MAX_LEVERAGE_BPS = 1_000_000;
    uint256 internal constant OPEN_TARGET_PRICE = 1.15e8;
    uint256 internal constant CLOSE_TARGET_PRICE = 1.3e8;
    uint8 internal constant LIVE_MODE = 1;

    DeployPerpsArbitrumSepolia.DeployedContracts internal d;

    function setUp() public {
        vm.chainId(ARBITRUM_SEPOLIA_CHAIN_ID);
        vm.warp(MONDAY_MORNING);
        vm.etch(ARBSYS, address(new E2EArbSys()).code);
        vm.etch(RELEASE_PYTH, address(new MockPyth()).code);
        vm.setEnv("TEST_PRIVATE_KEY", vm.toString(DEPLOYER_KEY));
        d = DeployPerpsArbitrumSepolia(deployCode("DeployPerpsArbitrumSepolia.s.sol:DeployPerpsArbitrumSepolia")).run();

        // Documented owner bootstrap: seed both tranches and activate trading.
        address owner = vm.addr(DEPLOYER_KEY);
        d.settlementToken.mint(owner, 2 * SEED_ASSETS);
        vm.startPrank(owner);
        d.settlementToken.approve(address(d.housePool), 2 * SEED_ASSETS);
        d.housePool.initializeSeedPosition(false, SEED_ASSETS, owner);
        d.housePool.initializeSeedPosition(true, SEED_ASSETS, owner);
        d.housePool.activateTrading();
        vm.stopPrank();

        _setBasket(BASKET_PRICE);
        vm.prank(KEEPER);
        d.router.updateMarkPrice(_updateData());
    }

    function test_E2E_RebateCloseCreditsTreasuryZeroFee() public {
        address treasury = d.engine.protocolTreasury();
        uint256 treasury0 = d.clearinghouse.balanceAssets(treasury);

        // 1. LP supplies junior liquidity through the async vault; a keeper settles the LP epoch; LP claims shares.
        d.settlementToken.mint(LP, LP_ASSETS);
        vm.startPrank(LP);
        d.settlementToken.approve(address(d.juniorVault), LP_ASSETS);
        uint256 requestId = d.juniorVault.requestDeposit(LP_ASSETS, LP, LP);
        vm.stopPrank();
        vm.warp(d.housePool.lpEpochStart(requestId));
        _setBasket(BASKET_PRICE);
        vm.prank(KEEPER);
        d.router.settleLpEpoch(_updateData());
        vm.prank(LP);
        d.juniorVault.claimDeposit(requestId, LP_ASSETS, LP, LP);

        // 2. Fresh trader deposits margin.
        d.settlementToken.mint(TRADER, TRADER_ASSETS);
        vm.startPrank(TRADER);
        d.settlementToken.approve(address(d.clearinghouse), TRADER_ASSETS);
        d.clearinghouse.depositMargin(TRADER_ASSETS);
        vm.stopPrank();

        uint256 traderStart = d.clearinghouse.balanceAssets(TRADER);
        uint256 poolStart = d.settlementToken.balanceOf(address(d.housePool));
        uint256 keeperStart = d.clearinghouse.balanceAssets(KEEPER);
        uint256 depthAtOpen = d.housePool.totalAssets();

        // 3. Trader opens a LONG at zero skew through the Router; a keeper executes it with a fresh Pyth update.
        _trade(false, bytes32("open"), OPEN_MARGIN, OPEN_TARGET_PRICE);
        uint256 openPrice = d.engine.lastMarkPrice();
        uint256 openNotional = POSITION_SIZE * openPrice / SIZE_TO_NOTIONAL;
        uint256 expectedOpenFee = openNotional * EXECUTION_FEE_BPS / BPS;
        uint256 vpiPaidAtOpen = openNotional * openNotional / (VPI_DENOMINATOR * depthAtOpen);
        uint256 treasuryAfterOpen = d.clearinghouse.balanceAssets(treasury);
        assertEq(treasuryAfterOpen - treasury0, expectedOpenFee, "open: treasury credited the full 4 bps fee");

        // 4. Time passes at the same basket; the trader fully closes through the Router (close bounty is zero).
        vm.warp(vm.getBlockTimestamp() + HOLD_SECONDS);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 traderBeforeClose = d.clearinghouse.balanceAssets(TRADER);
        uint256 depthAtClose = d.housePool.totalAssets();
        uint256 freeCashBeforeClose = d.housePool.getSettlementFreeAssets();
        _trade(true, bytes32("close"), 0, CLOSE_TARGET_PRICE);
        (uint256 sizeAfter,,,,,,) = d.engine.positions(TRADER);
        assertEq(sizeAfter, 0, "position fully closed");

        // 5. Independent expectations for the close leg.
        uint256 closePrice = d.engine.lastMarkPrice();
        uint256 closeNotional = POSITION_SIZE * closePrice / SIZE_TO_NOTIONAL;
        uint256 expectedCloseFee = closeNotional * EXECUTION_FEE_BPS / BPS;
        uint256 curveRebate = closeNotional * closeNotional / (VPI_DENOMINATOR * depthAtClose);
        uint256 expectedRebate = curveRebate < vpiPaidAtOpen ? curveRebate : vpiPaidAtOpen; // lifetime clamp
        uint256 carryBound = closeNotional * BASE_CARRY_BPS * (HOLD_SECONDS + 2 * COMMIT_TO_EXECUTE_SECONDS)
            / (BPS * YEAR) + 1;
        assertGt(expectedRebate, expectedCloseFee, "precondition: rebate exceeds fee");
        assertGt(freeCashBeforeClose, expectedCloseFee, "pool had free cash to fund the fee when the close settled");

        // 6. Impact: the trader was charged the close fee (received rebate - fee) ...
        int256 traderCloseDelta = int256(d.clearinghouse.balanceAssets(TRADER)) - int256(traderBeforeClose);
        assertApproxEqAbs(
            traderCloseDelta,
            int256(expectedRebate) - int256(expectedCloseFee),
            carryBound + VPI_ROUNDING_ATOMS,
            "trader received rebate minus the close fee"
        );
        // ... but the treasury received nothing for the close, and lost exactly the close fee overall.
        uint256 treasuryFinal = d.clearinghouse.balanceAssets(treasury);
        assertEq(treasuryFinal - treasuryAfterOpen, 0, "close: treasury credited zero");
        uint256 expectedTreasuryRevenue = expectedOpenFee + expectedCloseFee;
        assertEq(expectedTreasuryRevenue - (treasuryFinal - treasury0), expectedCloseFee, "treasury lost the close fee");

        // 7. Conservation over real balances: what the trader paid went to keeper, treasury and the pool (LPs).
        uint256 traderPaid = traderStart - d.clearinghouse.balanceAssets(TRADER);
        uint256 keeperGain = d.clearinghouse.balanceAssets(KEEPER) - keeperStart;
        uint256 poolGain = d.settlementToken.balanceOf(address(d.housePool)) - poolStart;
        assertEq(traderPaid, keeperGain + (treasuryFinal - treasury0) + poolGain, "value conserved");
        assertGe(poolGain, expectedCloseFee, "the withheld close fee stayed in the LP pool");
    }

    function _trade(
        bool isClose,
        bytes32 clientOrderId,
        uint256 margin,
        uint256 targetPrice
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = margin;
        request.targetPrice = targetPrice;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + d.router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(d.router.maxExecutionWindowSeconds()),
            allowedExecutionModes: LIVE_MODE,
            expectedConfigHash: d.lifecycleBook.currentExecutionConfigHash(),
            maxExecutionBountyAssets: MAX_BOUNTY,
            maxExecutionNotionalAssets: MAX_NOTIONAL,
            maxGrossAccountDebitAssets: MAX_DEBIT,
            maxActionChargeAssets: MAX_ACTION_CHARGE,
            maxExplicitFeesAssets: MAX_ACTION_CHARGE,
            maxPostPositionSize: POSITION_SIZE,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: uint32(MAX_LEVERAGE_BPS)
        });
        vm.prank(TRADER);
        uint64 orderId = d.router.commitOrder(request);
        vm.warp(vm.getBlockTimestamp() + COMMIT_TO_EXECUTE_SECONDS);
        vm.roll(vm.getBlockNumber() + COMMIT_TO_EXECUTE_BLOCKS);
        _setBasket(BASKET_PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = d.router.executeOrder(orderId, _updateData());
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    /// @dev Publishes a fresh Pyth price for every release feed (external dependency), moving each by the same factor.
    function _setBasket(
        uint256 basketPrice
    ) internal {
        PletherOracle oracle = PletherOracle(d.pletherOracle);
        for (uint256 i; i < 6; ++i) {
            uint256 normalized = oracle.basePrices(i) * basketPrice / 1e8;
            uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalized : normalized;
            MockPyth(RELEASE_PYTH)
                .setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), int32(-8), vm.getBlockTimestamp());
        }
    }

    function _updateData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

}
```
