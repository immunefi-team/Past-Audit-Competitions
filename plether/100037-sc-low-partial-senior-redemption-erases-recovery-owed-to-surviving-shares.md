# \#100037 [SC-Low] Partial Senior redemption erases recovery owed to surviving shares

**Submitted on Nov 16th 2026 at 05:50:15 UTC by @polymix123 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100037
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

# Partial Senior redemption erases recovery owed to surviving shares

**Severity:** High — Theft of unclaimed yield (Senior recovery value owed to surviving shares)

## Brief/Intro

A Senior redemption can receive all *current principal* while burning only part of the outstanding Senior shares. The withdrawal accounting treats this as a complete exit and clears the entire Senior high-water mark (HWM). The remaining Senior LPs keep their shares but lose priority over future trading revenue; that revenue instead funds a Junior redemption. The defect is a new bug introduced by the MED-11 repair: in the PoC's state the pre-repair share-fraction formula leaves the survivors `17,900,001.000018 USDG` of HWM, while the repaired asset-fraction formula leaves zero. The standalone launch-configuration PoC clears a `38,900,001.000018 USDG` HWM while the victim retains every share, then routes `15,959,999.610736 USDG` of later Pool revenue into an attacker-held Junior redemption.

## Vulnerability Details

### Full asset funding does not imply a full ownership exit

The [waterfall rules](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L340) preserve the Senior HWM through losses and assign later surplus to Senior until its HWM is restored. Pending redemption shares remain outstanding until their funded portion burns ([accounting rules](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L565-L574)). Accordingly, a partial ownership exit must not delete recovery priority belonging to surviving shares.

At LP settlement, `HousePool` freezes Senior principal and share supply, funds the queued Senior redemption, then passes **only funded assets** to the HWM scaler. It has already calculated `seniorPhase.fundedShares`, but does not use it for this update ([HousePool.sol, lines 843–865](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L843-L865)). The scaler subtracts the funded-asset fraction of the whole HWM ([HousePoolWaterfallAccountingLib.sol, lines 119–131](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L119-L131)):

```solidity
nextState.seniorHighWaterMark -= Math.mulDiv(
    state.seniorHighWaterMark, fundedAssets, state.seniorPrincipal, Math.Rounding.Floor
);
nextState.seniorPrincipal -= fundedAssets;
```

The redemption quote uses ERC-4626 virtual offsets: one virtual asset and 1,000 virtual shares ([TrancheVault.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L34-L37), [HousePoolRedemptionMathLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolRedemptionMathLib.sol#L30-L47)). At the live zero exit fee, for principal `P` and supply `S` measured in their respective token atoms, burning `x` shares funds:

```text
A(x) = floor(x * (P + 1) / (S + 1000))
```

Whenever `S >= 1001P + 1`, an `x < S` exists for which `A(x) = P`. For example, choose `x = ceil(P * (S + 1000) / (P + 1))`; up to `floor((S - 1000P) / (P + 1))` real shares can survive. Yet `fundedAssets = P` makes the scaler set `HWM' = HWM - floor(HWM * P / P) = 0`. `TrancheVault.fundRedeemEpoch` burns only the funded shares ([TrancheVault.sol, lines 1478–1502](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1478-L1502)); the surviving shares lose their recovery priority. The burning side need not be one holder: a matured redeem epoch is priced and funded as a single aggregate of every request in it ([HousePool.sol, lines 944–959](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L944-L959)), so holders who stay are wiped by the holders who exit.

This is a dust-principal *family*, not a one-atom rounding coincidence. With the PoC's HWM and pre-burn supply, residual principal of 10 USDG atoms can erase approximately `3.54m USDG` of survivor HWM, 100 atoms approximately `385k USDG`, and 1,000 atoms (`0.001 USDG`) approximately `38.9k USDG`. Material harm requires a severe prior impairment, but the accounting error applies to every state satisfying the same ownership boundary.

### Public lifecycle reproduction

The PoC deploys the scoped contracts without patching or mocking them. Mock USDG and Mock Pyth are the out-of-scope settlement-token and oracle dependencies. It applies the release configuration, including the six-feed topology, 8% Senior coupon, 40m-USDG/80% Senior caps, 40% maximum skew, VPI, execution and margin fees, and 1,000-USDG minimum open. Senior and Junior deposits and redemptions use the asynchronous vault lifecycle; trades use typed-V3 Router commitments and keeper execution.

1. The attacker and victim deposit into Senior; the attacker also deposits into Junior. The attacker queues a Senior redemption and a later-maturing Junior redemption. Before the attack, Senior has `38,900,001.000018 USDG` of HWM and the victim's shares represent approximately `17,899,999.999999 USDG` of it.
2. Public balanced opens and ordinary closes create a `39,917,953.136678 USDG` trader claim, then leave Pool cash at that claim plus **one USDG atom**. A cap-entry SHORT remains open, with `maxLiabilityAssets = 0` at the cap. Once that zero-liability state is established, the owner performs the documented `clearDegradedMode` recovery. The claimant publicly settles and leaves exactly one atom of recognized Pool cash.
3. A current six-feed payload lets the Router settle the matured Senior LP epoch while the position is open. Reconciliation leaves `seniorPrincipal = 1` atom. The attacker's **partial** Senior burn receives that atom. The victim keeps all `17,899,677,806,121,683` shares, and the post-burn supply remains `17,899,678,806,121,683` shares, but both Senior principal and the HWM become zero.
4. The surviving SHORT later closes adversely through the public Router, realizing `15,960,000 USDG` of Pool revenue. At the later no-position LP settlement, `15,959,999.610736 USDG` funds the attacker's Junior redemption, with `0.389264 USDG` retained as Junior principal. Senior receives zero.

The open cap-entry SHORT has zero maximum-profit liability at the critical settlement; the test asserts that property both before degraded-mode recovery and before the Senior burn. It uses `OrderRouter.settleLpEpoch` while that position is live. The subsequent direct `HousePool.settleLpEpoch` call occurs only after all positions are closed, when the function is permissionless by design.

### Why this is a new bug introduced by the MED-11 repair, not accepted rounding

The program accepts a report on a previously fixed or acknowledged root when it shows that the fix at the audited commit "is incomplete or introduces a new bug". This report shows the second.

MED-11 in the [Cecuro audit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf) (pp. 31–32) found that the old share-fraction HWM reduction could understate survivor protection by a virtual-offset-scale amount (1 USDC in its 2,000 USDC example), and asked that the "remaining senior entitlement" stay "aligned with remaining ownership". The [repair](https://github.com/immunefi-team/audit-comp-plether/commit/d757d0b175b5c2154babee946362ff90bd585f22) replaced the share fraction with the funded-asset fraction so that retained fees would not remove more HWM than the funded economic debit. Before it, [`HousePool`](https://github.com/immunefi-team/audit-comp-plether/blob/a27cf98a063233d65900ade65c7977c25848b78c/packages/perps/src/HousePool.sol#L849-L853) passed `seniorPhase.fundedShares` and `seniorPhase.pricingSupply`, and the [scaler](https://github.com/immunefi-team/audit-comp-plether/blob/a27cf98a063233d65900ade65c7977c25848b78c/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L112-L133) computed `HWM' = HWM - floor(HWM * fundedShares / preBurnSupply)`. Applied to the PoC's exact pre-burn state (`20,999,622,007,181,863` of `38,899,300,813,303,546` Senior shares burned), that pre-repair formula leaves `17,900,001.000018 USDG` of HWM for the surviving shares; this is one atom above the `17,900,001.000017 USDG` ownership floor the PoC asserts, because the old formula floors the amount removed rather than the amount kept. The repaired formula leaves zero. The total erasure did not exist before the repair and exists only because of it: the repair addresses the fee case but silently assumes that funding all principal means all protected ownership has exited, and the ERC-4626 offsets disprove that assumption.

The MED-11 and LOW-54 responses state that full principal funding clears the HWM, and the repair's library test [`test_PartialFundingRoundsProtectionUpAndFinalFundingClearsIt`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/SeniorWithdrawalMath.t.sol#L47-L58) asserts that endpoint. Both describe the asset formula in isolation; the test models no share supply, so neither addresses a full-principal funding that leaves real owners behind. Where the repair does address surviving owners, it points the other way. The current [accounting text](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L575-L583) says the formula "retains impaired protection proportionally" and that "Rounding favors the remaining protected claim". The scaler's [NatSpec](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L112-L115) and [`test_ZeroFundingPreservesWipedOutRecoveryRights`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/SeniorWithdrawalMath.t.sol#L40-L45) deliberately keep the HWM of a Senior tranche wiped to zero principal. Surviving shares are therefore meant to retain their recovery rights through a total impairment; with one more atom of principal, a single partial burn deletes them. The harmful state is the formula's unstated boundary case: full asset funding and a partial real-share burn coexist, so the formula no longer preserves the stated Senior-before-Junior recovery priority.

Other prior items do not cover this harm. The LOW-54 rounding allowance (p. 178) concerns less than one **extra** HWM atom per pass, in Senior's favor; here millions of HWM are deleted, in Junior's favor. SC-M6 in the [pre-audit report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf) (immutable `CAP_PRICE`) concerns protocol behaviour once the basket reaches the cap; the PoC marks the basket at the cap to keep a zero-liability SHORT open as the later revenue source, but the scaler is mark-independent and no cap behaviour erases the HWM. MED-24 and MED-29 (zero-payout refund epochs and tiny-share allocation inside a partially funded epoch) concern bounded queue work and per-controller allocation; neither involves the HWM update, and repairing either leaves this wipe intact.

### Proof of concept

The [secret gist](https://gist.github.com/fuzious/0051b12d3ca8f7de9f8e36488c971e8c) contains the standalone test, a one-file patch, and exact application instructions.

Place the standalone PoC at `packages/perps/test/foundry/ph049/PH049PublicLifecycle.t.sol` on commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, then run from `packages/perps`:

```bash
forge test --match-path test/foundry/ph049/PH049PublicLifecycle.t.sol \
  --match-test test_PublicClaimRouteLeavesOneAtomThenPartialBurnErasesVictimHwm -vv
```

The test asserts the victim's unchanged shares, the surviving ownership HWM floor, the erroneous zero HWM, the later realized revenue, and the Junior funding amount. It passed in clean checkouts of the pinned commit with only this test file added, on Forge `1.5.1-v1.5.1` and Forge `1.7.1` with Solc `0.8.35` (1/1 passed; gas `25,596,708`). The PoC's simulated six-feed prices demonstrate contract-permitted execution; their real-world publication frequency and a complete all-account profit calculation are outside this reproduction. If one entity funds every non-victim role, the test uses approximately `147m USDG` of capital, and the trace includes the honest owner-only degraded-mode recovery after zero liability.

## Impact Details

The selected impact is **High: Theft of unclaimed yield**. The Senior HWM is the tranche's recorded, not yet paid claim on future Pool revenue: the waterfall routes surplus to Senior until the mark is restored, and only then to Junior. Erasing the mark does not delay that payment; it reassigns it to another owner. This is recovery value taken from surviving Senior LPs, rather than a debit of already-booked Senior principal, which is why Critical is not claimed. In the executed trace, the `15,960,000 USDG` realized revenue is below the approximately `17,900,001.000017 USDG` HWM floor attributable to surviving Senior ownership, so all of it belongs to the survivors. Using the actual post-burn supply, the victim's surviving shares account for approximately `15,959,999.108363 USDG` of that revenue; permanent seed shares account for the small balance. Instead, Senior receives zero and the attacker-held Junior redemption is funded for `15,959,999.610736 USDG`. This is realized Pool cash transferred across the waterfall, not only a missed prospective coupon.

The party that triggers the wipe is the party that profits. By arithmetic on the PoC's logged supply, had the attacker simply kept its Senior shares, the same revenue would restore Senior pro rata: about `8,615,938 USDG` to the attacker's 53.98% of supply and about `7,344,062 USDG` to the victim's 46.02%. With the attack, the attacker collects `15,959,999.610736 USDG` through Junior. The attacker is about `7.34m USDG` better off than by holding, and the victim loses the full `15.96m USDG` it would receive under the pre-repair formula or the correction below. The triggering step needs no new capital: an already-impaired Senior position is redeemed for one atom.

For that reason the lower tiers do not fit. It is not "Contract fails to deliver promised returns, but doesn't lose value": the survivors' recorded entitlement is deleted and the corresponding cash is paid out of the Pool to a different owner. It is not griefing: the beneficiary and the trigger are the same party whenever the later revenue comes from a trader the attacker does not control, which is the ordinary source of Pool revenue. It is not the accepted LOW-54 dust: that allowance is below one atom per pass and favors Senior.

The owner recovery and extreme coherent price path are explicit prerequisites of the PoC, and the demonstration does not establish a net-positive strategy for a single actor financing every trader role. Under the program rules these are feasibility limitations: "Impact is calculated first, separately from feasibility. Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it." The owner step is "a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol", which the program keeps in scope. They affect exploit feasibility, not the observed contract-level misallocation of realized funds.

## References

- [Scoped source at the audited commit](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8)
- [HousePool Senior redemption and HWM update](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L865)
- [Senior HWM scaler](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L112-L131)
- [ERC-4626 redemption quote](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolRedemptionMathLib.sol#L20-L47)
- [TrancheVault share burn](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1478-L1502)
- [Senior recovery invariant](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L388-L395)
- [MED-11 repair commit](https://github.com/immunefi-team/audit-comp-plether/commit/d757d0b175b5c2154babee946362ff90bd585f22) and the [pre-repair share-fraction scaler](https://github.com/immunefi-team/audit-comp-plether/blob/a27cf98a063233d65900ade65c7977c25848b78c/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L112-L133)
- [Repair regression tests for Senior withdrawal math](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/SeniorWithdrawalMath.t.sol#L40-L58)
- [Cecuro audit, MED-11 and LOW-54](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)


## Link to Proof of Concept

https://gist.github.com/fuzious/0051b12d3ca8f7de9f8e36488c971e8c

## Proof of Concept

```diff
diff --git a/packages/perps/test/foundry/ph049/PH049PublicLifecycle.t.sol b/packages/perps/test/foundry/ph049/PH049PublicLifecycle.t.sol
new file mode 100644
index 00000000..69cff338
--- /dev/null
+++ b/packages/perps/test/foundry/ph049/PH049PublicLifecycle.t.sol
@@ -0,0 +1,433 @@
+// SPDX-License-Identifier: AGPL-3.0
+pragma solidity 0.8.35;
+
+import {MockSettlementToken} from "../../utils/MockSettlementToken.sol";
+import {CfdEngine} from "@plether/perps/CfdEngine.sol";
+import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
+import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
+import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
+import {CfdEngineProtocolLens} from "@plether/perps/CfdEngineProtocolLens.sol";
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
+/// @notice Public launch-configuration reachability and impact reproduction for PH-049.
+/// @dev All economic mutations use the production bounded Router, six coherent release feeds,
+///      public LP APIs, public claim settlement, or an honest clearDegradedMode after L=0.
+contract PH049PublicLifecycleTest is Test {
+    uint256 private constant START_TIME = 1_709_532_000;
+    uint256 private constant CAP_PRICE = 2e8;
+    uint256 private constant LOT = CfdTypes.SIZE_QUANTUM;
+
+    address private constant SENIOR_ATTACKER = address(0xA491);
+    address private constant SENIOR_VICTIM = address(0xB491);
+    address private constant JUNIOR_ATTACKER = address(0xC491);
+    address private constant DRAINER = address(0xD491);
+    address private constant WINNER = address(0xE491);
+    address private constant REPLENISHER = address(0xF491);
+    address private constant FINAL_LOSER = address(0x1491);
+    address private constant KEEPER = address(0x2491);
+
+    uint256 private constant SENIOR_ATTACKER_ASSETS = 21_000_000e6;
+    uint256 private constant SENIOR_VICTIM_ASSETS = 17_900_000e6;
+    uint256 private constant JUNIOR_ATTACKER_ASSETS = 41_000_000e6 + 47_739_783;
+
+    MockSettlementToken private token;
+    MockPyth private pyth;
+    MarginClearinghouse private clearinghouse;
+    CfdEngine private engine;
+    CfdEngineLens private lens;
+    CfdEngineProtocolLens private protocolLens;
+    HousePool private pool;
+    TrancheVault private senior;
+    TrancheVault private junior;
+    OrderRouter private router;
+    PletherOracle private oracle;
+    bytes32[] private feedIds;
+    uint256[] private basePrices;
+    uint256 private clientNonce;
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
+        protocolLens = new CfdEngineProtocolLens(address(engine));
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
+        _seedAndFundPool();
+        _warpToNextMondayAtTen();
+        router.updateMarkPrice(_livePayload(1, 1));
+
+        _fundTrader(DRAINER, 10_000_000e6);
+        _fundTrader(WINNER, 10_000_000e6);
+        _fundTrader(REPLENISHER, 40_000_000e6);
+        _fundTrader(FINAL_LOSER, 25_000_000e6);
+    }
+
+    function test_PublicClaimRouteLeavesOneAtomThenPartialBurnErasesVictimHwm() public {
+        assertEq(pool.seniorRateBps(), 800, "launch coupon must remain active");
+        uint256 attackerShares = senior.balanceOf(SENIOR_ATTACKER);
+        uint256 victimShares = senior.balanceOf(SENIOR_VICTIM);
+        uint256 supplyBefore = senior.totalSupply();
+        uint256 hwmBefore = pool.seniorHighWaterMark();
+        uint256 victimRecoveryBefore = hwmBefore * victimShares / supplyBefore;
+        emit log_named_uint("senior.hwmBefore", hwmBefore);
+        emit log_named_uint("senior.attackerShares", attackerShares);
+        emit log_named_uint("senior.victimShares", victimShares);
+        emit log_named_uint("senior.supplyBefore", supplyBefore);
+        emit log_named_uint("senior.victimRecoveryBefore", victimRecoveryBefore);
+        assertGt(victimRecoveryBefore, 17_800_000e6, "victim has material recovery priority");
+
+        // Queue the attacker's Senior exit now, then use the public five-minute cutoff to queue a Junior exit
+        // one epoch later. The latter provides real LP work for the eventual post-attack revenue reconciliation.
+        vm.prank(SENIOR_ATTACKER);
+        uint256 redeemId = senior.requestRedeem(attackerShares, SENIOR_ATTACKER, SENIOR_ATTACKER);
+        (uint256 imminentEpoch, uint256 imminentCutoff) = junior.getRequestEpochWindow();
+        assertEq(imminentEpoch, redeemId);
+        vm.warp(imminentCutoff + 1);
+        uint256 juniorExitShares = junior.balanceOf(JUNIOR_ATTACKER);
+        vm.prank(JUNIOR_ATTACKER);
+        uint256 juniorRedeemId = junior.requestRedeem(juniorExitShares, JUNIOR_ATTACKER, JUNIOR_ATTACKER);
+        assertGt(juniorRedeemId, redeemId);
+
+        // Balanced launch-valid entry book.
+        _trade(DRAINER, CfdTypes.Side.LONG, 200_000 * LOT, 2_000_000e6, false, 1, 1);
+        _trade(WINNER, CfdTypes.Side.SHORT, 200_000 * LOT, 2_000_000e6, false, 1, 1);
+        _trade(DRAINER, CfdTypes.Side.LONG, 200_000 * LOT, 2_000_000e6, false, 1, 1);
+        _trade(REPLENISHER, CfdTypes.Side.SHORT, 200_000 * LOT, 24_000_000e6, false, 1, 1);
+        _trade(DRAINER, CfdTypes.Side.LONG, 200_000 * LOT, 2_000_000e6, false, 1, 1);
+        _trade(WINNER, CfdTypes.Side.SHORT, 200_000 * LOT, 2_000_000e6, false, 1, 1);
+        _trade(DRAINER, CfdTypes.Side.LONG, 100_000 * LOT, 1_000_000e6, false, 1, 1);
+        _trade(REPLENISHER, CfdTypes.Side.SHORT, 97_702 * LOT, 14_000_000e6, false, 1, 1);
+
+        // At the capped endpoint a SHORT has exactly zero maximum-profit liability. Keep this ordinary public
+        // position open through the atom-small redemption, then realize its adverse move as public revenue later.
+        _trade(FINAL_LOSER, CfdTypes.Side.SHORT, 80_000 * LOT, 20_000_000e6, false, 3, 1);
+
+        // Drain immediate cash, create a deferred winning claim, then service it with a normal losing close.
+        _trade(DRAINER, CfdTypes.Side.LONG, 700_000 * LOT, 0, true, 1, 200);
+        assertEq(engine.traderClaimBalanceAssets(DRAINER), 0, "drainer payout must be immediate");
+        _trade(WINNER, CfdTypes.Side.SHORT, 400_000 * LOT, 0, true, 2, 1);
+        uint256 claim = engine.traderClaimBalanceAssets(WINNER);
+        assertGt(claim, 30_000_000e6, "winner must hold a material claim");
+        _trade(REPLENISHER, CfdTypes.Side.SHORT, 297_702 * LOT, 0, true, 1, 200);
+
+        assertEq(_positionSize(DRAINER), 0);
+        assertEq(_positionSize(WINNER), 0);
+        assertEq(_positionSize(REPLENISHER), 0);
+        assertEq(_positionSize(FINAL_LOSER), 80_000 * LOT, "only the zero-liability terminal position survives");
+        assertEq(engine.totalTraderClaimBalanceAssets(), claim);
+        uint256 cashBeforeClaim = token.balanceOf(address(pool));
+        emit log_named_uint("claim", claim);
+        emit log_named_uint("cashBeforeClaim", cashBeforeClaim);
+        emit log_named_int("cashMinusClaim", int256(cashBeforeClaim) - int256(claim));
+
+        // Honest recovery action: L=0, so one atom of free cash is solvent.
+        assertEq(protocolLens.getProtocolAccountingSnapshot().maxLiabilityAssets, 0);
+        if (engine.degradedMode()) engine.clearDegradedMode();
+
+        vm.prank(WINNER);
+        engine.settleTraderClaim(WINNER);
+        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
+        assertEq(token.balanceOf(address(pool)), 1, "public claim settlement leaves one cash atom");
+        assertEq(pool.accountedAssets(), 1, "accounted cash follows physical cash");
+        assertEq(pool.seniorHighWaterMark(), hwmBefore, "claim settlement does not erase HWM");
+
+        // A partial ownership exit receives the only real atom and is treated as a complete economic exit.
+        vm.warp(pool.lpEpochStart(redeemId));
+        router.settleLpEpoch(_livePayload(3, 1));
+        assertEq(senior.claimableRedeemAssets(redeemId, SENIOR_ATTACKER), 1);
+        assertEq(senior.totalSupply(), supplyBefore - attackerShares);
+        assertGt(senior.totalSupply(), 0, "victim and seed shares survive");
+        uint256 postBurnSupply = senior.totalSupply();
+        uint256 survivorHwmFloor = hwmBefore * postBurnSupply / supplyBefore;
+        assertEq(postBurnSupply, 17_899_678_806_121_683, "partial burn leaves material Senior supply");
+        assertEq(survivorHwmFloor, 17_900_001_000_017, "surviving ownership retains material HWM");
+        assertEq(senior.balanceOf(SENIOR_VICTIM), victimShares, "victim retains every share");
+        assertEq(pool.seniorPrincipal(), 0);
+        assertEq(pool.seniorHighWaterMark(), 0, "partial burn clears all recovery priority");
+
+        // The surviving position was a zero-NAV, zero-liability item at redemption. Its later adverse close is
+        // ordinary public trading revenue and requires no new entry into either terminally wiped tranche.
+        assertEq(protocolLens.getProtocolAccountingSnapshot().maxLiabilityAssets, 0);
+        _trade(FINAL_LOSER, CfdTypes.Side.SHORT, 80_000 * LOT, 0, true, 1, 200);
+        assertEq(_positionSize(FINAL_LOSER), 0);
+        uint256 publicRevenue = token.balanceOf(address(pool));
+        assertGt(publicRevenue, 10_000_000e6, "ordinary loss creates material revenue");
+
+        // The later-maturing public Junior exit advances reconciliation and exposes where that revenue accrued.
+        vm.warp(pool.lpEpochStart(juniorRedeemId));
+        router.updateMarkPrice(_livePayload(1, 200));
+        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
+        emit log_named_uint("postAttack.publicRevenue", publicRevenue);
+        emit log_named_uint("postAttack.juniorFundedAssets", result.juniorFundedAssets);
+        emit log_named_uint("postAttack.juniorPrincipal", pool.juniorPrincipal());
+        uint256 victimRevenueEntitlement = publicRevenue * victimShares / postBurnSupply;
+        emit log_named_uint("postAttack.victimRevenueEntitlement", victimRevenueEntitlement);
+        assertEq(pool.seniorPrincipal(), 0, "surviving Senior owners recover nothing");
+        assertEq(pool.seniorHighWaterMark(), 0);
+        assertGt(result.juniorFundedAssets, 10_000_000e6, "public revenue funds Junior, not Senior");
+        assertEq(pool.juniorPrincipal() + result.juniorFundedAssets, publicRevenue);
+        assertEq(victimRevenueEntitlement, 15_959_999_108_363, "post-burn victim entitlement is material");
+        assertGt(result.juniorFundedAssets, victimRevenueEntitlement, "cross-tranche attacker captures victim recovery");
+        assertGt(victimRecoveryBefore, publicRevenue, "victim's erased priority covers the public revenue");
+    }
+
+    function _positionSize(address account) private view returns (uint256 size) {
+        (size,,,,,,) = engine.positions(account);
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
+        uint256 currentSize = _positionSize(account);
+        OrderV3Types.OrderRequest memory request;
+        request.clientOrderId = keccak256(abi.encode("ph049-public", ++clientNonce));
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
+        vm.prank(account);
+        uint64 id = router.commitOrder(request);
+        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(id);
+        uint64 commit = pending.commitTime;
+        vm.roll(vm.getBlockNumber() + 1);
+        vm.warp(vm.getBlockTimestamp() + 1);
+        bytes[] memory executionPayload = _payload(numerator, denominator, commit + 1, commit);
+        vm.prank(KEEPER);
+        OrderV3Types.ExecutionResult memory executionResult = router.executeOrder(id, executionPayload);
+        if (uint8(executionResult.status) != uint8(OrderV3Types.LifecycleStatus.Executed)) {
+            emit log_named_uint("failed.order", id);
+            emit log_named_uint("failed.status", uint8(executionResult.status));
+            emit log_named_uint("failed.pendingReason", uint8(executionResult.pendingReason));
+            emit log_named_uint("failed.terminalReason", uint8(executionResult.terminalReason));
+        }
+        assertEq(uint8(executionResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order did not execute");
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
+    function _seedAndFundPool() private {
+        token.mint(address(this), 2e6);
+        token.approve(address(pool), 2e6);
+        pool.initializeSeedPosition(false, 1e6, address(this));
+        pool.initializeSeedPosition(true, 1e6, address(this));
+        pool.activateTrading();
+
+        uint256 juniorId = _requestDeposit(junior, JUNIOR_ATTACKER, JUNIOR_ATTACKER_ASSETS);
+        vm.warp(junior.depositEpochStart(juniorId));
+        pool.settleLpEpoch(0, 0);
+        vm.prank(JUNIOR_ATTACKER);
+        junior.claimDeposit(juniorId, JUNIOR_ATTACKER_ASSETS, JUNIOR_ATTACKER, JUNIOR_ATTACKER);
+
+        uint256 attackerId = _requestDeposit(senior, SENIOR_ATTACKER, SENIOR_ATTACKER_ASSETS);
+        uint256 victimId = _requestDeposit(senior, SENIOR_VICTIM, SENIOR_VICTIM_ASSETS);
+        uint256 maturity = senior.depositEpochStart(attackerId);
+        if (senior.depositEpochStart(victimId) > maturity) maturity = senior.depositEpochStart(victimId);
+        vm.warp(maturity);
+        pool.settleLpEpoch(0, 0);
+        vm.prank(SENIOR_ATTACKER);
+        senior.claimDeposit(attackerId, SENIOR_ATTACKER_ASSETS, SENIOR_ATTACKER, SENIOR_ATTACKER);
+        vm.prank(SENIOR_VICTIM);
+        senior.claimDeposit(victimId, SENIOR_VICTIM_ASSETS, SENIOR_VICTIM, SENIOR_VICTIM);
+    }
+
+    function _requestDeposit(TrancheVault vault, address account, uint256 amount) private returns (uint256 id) {
+        token.mint(account, amount);
+        vm.startPrank(account);
+        token.approve(address(vault), amount);
+        id = vault.requestDeposit(amount, account, account);
+        vm.stopPrank();
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
