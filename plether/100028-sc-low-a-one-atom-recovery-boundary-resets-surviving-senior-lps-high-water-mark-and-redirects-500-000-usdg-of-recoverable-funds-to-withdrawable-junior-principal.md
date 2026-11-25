# \#100028 [SC-Low] A one-atom recovery boundary resets surviving Senior LPs' high-water mark and redirects 500,000 USDG of recoverable funds to withdrawable Junior principal

**Submitted on Nov 16th 2026 at 05:12:35 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100028
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief / Introduction

I found a recovery-ownership failure in `HousePool` that can turn value owed to surviving Senior LP shares into redeemable Junior LP principal following a complete pool impairment. The decisive condition is not a new trade, new loss, change in total assets, or size of the later recovery. It is whether **one USDG base unit (0.000001 USDG) of backed Senior principal exists at the exact moment an authorized recapitalization is reconciled**.

After a complete wipeout, the protocol preserves the historical `seniorHighWaterMark` precisely because a zero NAV does not burn the outstanding Senior shares or eliminate their recorded recovery priority. However, the first owner-funded recapitalization takes a bootstrap branch whenever both recorded principals are zero. That branch **overwrites** the existing HWM with the recapitalization amount, even when Senior shares and a much larger historical recovery target survive.

Once the HWM has been replaced, Senior principal equals its new target and the entire subsequent recovery is credited to Junior. Those Junior funds are not just reflected in a view: they can be funded through the standard redemption epoch and transferred to a Junior LP wallet.

I reproduced this on the audited Plether code with four passing Foundry tests. In the matched-cash differential, both branches had **600,000.000001 USDG** in the pool before redemption, the same **100,000 USDG** owner recapitalization, and the same **500,000.000001 USDG** of aggregate revenue. Moving just **0.000001 USDG of that revenue** from immediately after to immediately before the recapitalization changed the final allocation by **500,000.000001 USDG**:

| Final state (USDG) | Recapitalization first | One base unit of revenue first |
|---|---:|---:|
| Physical pool cash, before redemption | **600,000.000001** | **600,000.000001** |
| Historical Senior HWM before either branch | 1,000,666.666666 | 1,000,666.666666 |
| Senior HWM after recap and recovery | **100,000** | **1,000,666.666666** |
| Senior principal after recovery | **100,000** | **600,000.000001** |
| Junior principal after recovery | **500,000.000001** | **0** |

I then exercised the real Junior vault redemption path in the recapitalization-first branch. My Junior LP received **1,997.329587 USDG** in its wallet. That is the actual withdrawal attributable to the Junior shares held in this PoC; the 500,000.000001 USDG figure is the collective allocation shift across the tranches, not a claim that one wallet withdrew the full amount.

**The security issue is the change in ownership of existing recovery assets.** Token balances reconcile, but a balance-conservation invariant does not detect the funds being assigned to the wrong tranche. The historical Senior recovery obligation is effectively replaced by a smaller number when the owner's recovery funding happens to follow an exact-zero state.

## Vulnerability Details

### 1. A total loss does not extinguish Senior ownership or its recovery record

The ordinary waterfall explicitly treats the HWM as a recovery entitlement independent of current principal. `HousePoolWaterfallAccountingLib.absorbLoss()` leaves the mark unchanged after a loss, including a complete write-down, so later revenue can restore Senior first. `scaleSeniorOnWithdraw()` also preserves the HWM when zero Senior principal is actually redeemed. `distributeRevenue()` first fills the gap between Senior principal and HWM; only the remainder becomes Junior principal.

In the test, I funded a Senior seed position with 1,000,000 USDG and a Junior seed position with 250,000 USDG, and gave a separate Junior LP 1,000 USDG of shares before the loss. I then constructed a full impairment and reconciled the real pool. Both principals became zero; the Senior HWM remained **1,000,666.666666 USDG** (including legitimate fixture coupon accrual). The existing Senior share supply was not destroyed by the wipeout.

This behavior is consistent with `SECURITY.md`'s High-water-mark protection invariant: **“Senior impairment must be restored before junior extracts surplus.”**

### 2. A one-time recapitalization replaces the protected HWM at exactly zero principal

The affected code is `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent()` (`HousePoolPendingPreviewLib.sol:103-136`):

```solidity
if (state.seniorSupply > 0) {
    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
        remaining = 0;
    } else {
        uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
            ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
            : 0;
        // The existing Senior HWM is preserved in this branch.
    }
}
```

The condition examines **current principal** but not whether existing Senior owners retain a historical HWM. As long as both principals are zero, the bootstrap assigns the current recap amount as the recovery target. It neither compares against nor preserves the stored HWM.

I invoked the actual owner-only `CfdEngine.recapitalizePool(100_000e6)` once and reconciled once. There was **no outstanding trader claim**, **no partially settled recapitalization**, and **no repeated consumption of a pending balance**. The first recap was fully applied, with `pendingRecapitalizationAssets() == 0` afterward.

The pool went from a surviving historical HWM of **1,000,666.666666 USDG** to a new HWM of **100,000 USDG**. That is a **900,666.666666 USDG decrease in the recorded Senior recovery target** without any Senior share redemption or burn attributable to the recap.

### 3. One actual base unit of backing prevents the reset

I also ran a differential test that inserts **one USDG base unit of genuine test-token backing** through `HousePool.recordClaimantInflow(..., Revenue, CashArrived)` before the same recapitalization. This is not a direct storage overwrite.

That single unit establishes `seniorPrincipal = 1` while the original HWM remains intact. The recapitalization now selects the non-bootstrap path, which credits the owner-funded amount toward the existing Senior recovery gap. The historical HWM is preserved. The next 500,000 USDG restores Senior rather than Junior.

Under the zero-principal sequence, the same 500,000 USDG is assigned to Junior because the bootstrap has already reset the HWM to the amount of the recapitalization.

The difference is deterministic and exact: it is visible in the underlying account values, not inferred from a formula or a static-code hypothetical.

### 4. Matched-cash evidence isolates the ownership switch

In `test_H2_SameCashOneAtomRevenueOrderChangesTrancheOwnership()`, I branch from the same Foundry snapshot immediately after the wipeout:

**Branch A — recapitalization first:**

1. The owner recapitalizes with **100,000 USDG**.
2. The first reconciliation reduces the HWM to **100,000 USDG**.
3. **0.000001 USDG** revenue arrives and is accounted.
4. A further **500,000 USDG** revenue arrives.
5. **500,000.000001 USDG** is assigned to Junior, while Senior remains at **100,000 USDG**.

**Branch B — one unit before recapitalization:**

1. **0.000001 USDG** revenue arrives and becomes backed Senior principal.
2. The owner recapitalizes with the same **100,000 USDG**.
3. The old Senior HWM stays at **1,000,666.666666 USDG**.
4. The same **500,000 USDG** later revenue arrives.
5. Senior principal reaches **600,000.000001 USDG** and Junior receives **zero**.

Both branches end with identical pre-redemption physical cash of **600,000.000001 USDG**. The only deliberately varied input is the order of the one-base-unit revenue receipt relative to the same recapitalization. The recorded principal difference is **500,000.000001 USDG**; it is not caused by different total funding.

### 5. Junior owners can redeem the reassigned value as actual USDG

I carried the recapitalization-first branch through the real `TrancheVault`/`HousePool` settlement lifecycle: `requestRedeem()`, settlement maturity, public `settleLpEpoch(0,0)` and `claimRedeem()`. I asserted the change in the Junior LP's ERC20 wallet balance against the assets returned by `claimRedeem()`.

The actual output is:

```text
H2_JUNIOR_WALLET_RECEIPT_6DP 1997329587
```

The Junior LP received **1,997.329587 USDG**. This is a completed token movement in the deterministic PoC, not an unrealized share-price or accounting-only observation. It demonstrates that the shifted Junior allocation is withdrawable through an ordinary public LP exit.

## Impact Details — Critical: Direct theft of user funds

The impact I am reporting is **misallocation and withdrawal of recovery assets that would otherwise restore existing Senior LP ownership**. It is not theft of unclaimed yield, a rounding-only discrepancy, temporary unavailability, or a purely informational invariant violation.

When the recovery HWM remains preserved, the same 500,000.000001 USDG of revenue is entirely attributable to Senior restoration. When the zero-principal recapitalization branch overwrites the mark, that value becomes Junior principal despite unchanged Senior share ownership and no Senior share redemption reducing the historical entitlement. Junior LPs can then take the value out as actual settlement tokens.

The PoC establishes both economic dimensions: **500,000.000001 USDG moved from Senior allocation to Junior allocation**, and **1,997.329587 USDG withdrawn by one Junior account**. The broader collective exposure scales with the size of recovery while the original HWM exceeds the recapitalization amount.

I am selecting Immunefi's **Critical — “Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield”** because the claimed failure affects principal recovery rights and permits the competing tranche to redeem the funds. The requested category concerns the demonstrated type of impact; the final severity assessment belongs to the program's reviewers.

## Why this is not the confirmed H1 finding

My previously submitted report #99895 was confirmed High as a duplicate of #97844, *“Repeated capped recapitalization erases Senior's recovery watermark.”* That issue relies on repeated reconciliation of a **partially settled** recapitalization: the pending residual is selected again against value already recognized for Senior, causing a second bootstrap and HWM ratchet.

This finding uses a different and independent transition. It requires **one** recapitalization and **one** settlement, no residual pending balance, no outstanding trader claim, and no repeated pending-bucket replay. The HWM changes solely because both principals were exactly zero when that first recapitalization was applied. Fixing the H1 double-selection logic would not, on its own, prevent this one-time HWM replacement.

The matched one-atom test isolates that root cause: putting one backed base unit into Senior just before the first recap selects the other branch and prevents the ownership switch.

## Why the existing bootstrap test does not resolve the security issue

The codebase includes `test_PostWipeoutRequiresExplicitRecapForHwmReset()` and an independent reference model that expect the fresh HWM on a terminally wiped Senior tranche. I have not treated that expectation as evidence that the branch is untested or accidental. **The concern is that a previously encoded rule still allows an owner-funded recovery to extinguish the recovery priority of surviving Senior shares at an exact-zero boundary**, in tension with the separate Senior-priority invariant.

The existing reset check establishes the implementation's bootstrap behavior. It does not establish that a 500,000 USDG Senior/Junior transfer under identical total funding is an authorized transfer of rights among existing owners, or examine a one-base-unit backed-Senior differential and funded Junior redemption. My four tests supply that economic and wallet-level comparison. I request a security review of the ownership result rather than a restatement that the code follows the existing bootstrap test.

## Preconditions and reproduction scope

The Junior LP already owns shares acquired while the tranche had value. A complete pool impairment occurs, the authorized owner funds a smaller recapitalization, and the pool later receives recoverable value. The **owner-only** recapitalization is an external precondition, not an attacker privilege. The Junior LP's actual withdrawal uses the standard public redemption route.

For repeatability, my Foundry PoC constructs the full impairment using the repository's mock USDG token and supplies recovery value using mock-token minting paired with the real `recordClaimantInflow(..., Revenue, CashArrived)` accounting entrypoint under a test-only Engine impersonation. It does not purport to independently show that an unprivileged trader can force the full impairment, owner recapitalization, or subsequent revenue through ordinary market activity. These fixture operations set up the financial conditions in which the audited recovery and withdrawal logic executes.

## Recommended Remediation

A terminal zero NAV should not, by itself, be sufficient to discard a valid historical Senior HWM while Senior share owners continue to exist. On an ordinary, non-dilutive claimant recapitalization, restore Senior toward the recorded recovery target rather than replacing the target solely because both current principals equal zero.

If an explicit reset of surviving Senior recovery rights is an intended governance action, make that transition separate and explicit, with a clear authorization and accounting treatment rather than binding it implicitly to the exact-zero principal condition. Add a regression comparing zero principal and one genuinely backed USDG base unit with the same recap amount, identical cumulative recovery and actual Junior withdrawal checks.



## Proof of Concept

## Proof of Concept

## Audited environment

- **Commit:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- **Framework:** Foundry; Solidity 0.8.35
- **Execution:** Four tests passed using Plether's release risk configuration
- **Production source:** Unmodified (`git diff --exit-code HEAD -- packages/perps/src` returned 0)
- **Test contracts:** `HousePool`, `CfdEngine`, `TrancheVault`, real claimant-inflow accounting; repository mock USDG and fixture funding for controlled loss/recovery

The complete Solidity PoC files are included as Appendices A and B and separately as below:
```text
packages/perps/test/perps/h2_audit/H2PostWipeOneShotDiagnostic.t.sol
packages/perps/test/perps/h2_audit/H2FinalMatchedCashTimingPoC.t.sol
```

## Reproduction commands

```bash
cd /workspaces/audit-comp-plether
export PATH="$HOME/.foundry/bin:$PATH"
test "$(git rev-parse HEAD)" = "a82d4fefb02768e5b3bddcc97dee371eee24fea8"
cd packages/perps
forge test -j 1 --match-contract H2FinalMatchedCashTimingPoC -vv
cd ../..
git diff --exit-code HEAD -- packages/perps/src
```

## Actual Foundry result

```text
[PASS] test_H2_NoRecapRecoveryStillRestoresOldHwm()
[PASS] test_H2_OneAtomBackingChangesRecoveryRouting()
[PASS] test_H2_SameCashOneAtomRevenueOrderChangesTrancheOwnership()
[PASS] test_H2_SingleRecapResetAndJuniorCash()
Suite result: ok. 4 passed; 0 failed; 0 skipped
FORGE EXIT STATUS: 0
PRODUCTION DIFF STATUS: 0
```

## Decisive output (USDG, six decimals)

```text
H2_ORIGINAL_HWM_6DP 1000666666666
H2_MATCHED_CASH_A_6DP 600000000001
H2_MATCHED_CASH_B_6DP 600000000001
H2_MATCHED_SENIOR_A_6DP 100000000000
H2_MATCHED_SENIOR_B_6DP 600000000001
H2_MATCHED_JUNIOR_A_6DP 500000000001
H2_MATCHED_JUNIOR_B_6DP 0
H2_MATCHED_HWM_A_6DP 100000000000
H2_MATCHED_HWM_B_6DP 1000666666666
H2_MATCHED_JUNIOR_WALLET_A_6DP 1997329587
```

## Source references

- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:103-136` — zero-principal recapitalization HWM overwrite
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:112-175` — Senior-first restoration and loss-preserved HWM
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:68-105` — normal reconcile/accounting routing
- `packages/perps/src/CfdEngine.sol:627-653` — owner-funded recapitalization that mints no LP shares
- `packages/perps/src/HousePool.sol:1549-1598` — reconciliation and claimant intent application
- `packages/perps/src/HousePool.sol:794-835` — public settlement path
- `packages/perps/SECURITY.md:386-400` — High-water-mark protection invariant
- `packages/perps/ACCOUNTING_SPEC.md:329-340` — Senior/Junior recovery ownership
- `packages/perps/test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol:24-55` — existing reset expectation
- `packages/perps/test/perps/invariant/properties/PerpWaterfallReferenceInvariant.t.sol:154-169` — independent reference model

---

# Appendix A — Complete one-shot H2 PoC

File: `packages/perps/test/perps/h2_audit/H2PostWipeOneShotDiagnostic.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

contract H2PostWipeOneShotDiagnostic is BasePerpTest {
    address internal constant JUNIOR_LP = address(0xB002);
    uint256 internal constant RECAP = 100_000e6;
    uint256 internal constant RECOVERY = 500_000e6;
    uint256 internal constant ONE_ATOM = 1;

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1_000_000e6;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 250_000e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
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

    function setUp() public override {
        super.setUp();
        uint256 originalTimestamp = block.timestamp;
        ICfdEngineAdminHost.EngineRiskConfig memory config = _engineRiskConfig();
        config.riskParams = _riskParams();
        config.executionFeeBps = 4;
        config.frozenCloseSpreadBps = 50;
        config.settlementBufferBps = 25;
        engineAdmin.proposeRiskConfig(config);
        vm.warp(engineAdmin.riskConfigActivationTime());
        engineAdmin.finalizeRiskConfig();
        vm.warp(originalTimestamp);
        assertEq(_engineRiskConfig().riskParams.maxSkewRatio, 0.4e18);
    }

    function _wipeAndPreserveHwm() internal returns (uint256 priorHwm) {
        // Real unlocked Junior shares exist before the loss.
        _fundJunior(JUNIOR_LP, 1_000e6);
        assertGt(juniorVault.balanceOf(JUNIOR_LP), 0);
        priorHwm = pool.seniorHighWaterMark();
        assertGe(priorHwm, 1_000_000e6);

        // TEST-ONLY: manufacture a full economic loss.
        settlementToken.burn(address(pool), pool.totalAssets());
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.seniorHighWaterMark(), priorHwm);
        console2.log("H2_ORIGINAL_HWM_6DP", priorHwm);
    }

    function _oneShotOwnerRecap() internal {
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        assertEq(pool.pendingRecapitalizationAssets(), RECAP);
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.pendingRecapitalizationAssets(), 0);
    }

    function _testRevenue(uint256 amount) internal {
        // TEST-ONLY: emulate a claimant-owned revenue receipt with real
        // HousePool accounting, not a complete natural trading sequence.
        settlementToken.mint(address(pool), amount);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            amount,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function _redeemJuniorCash() internal returns (uint256 received) {
        uint256 unlockAt = juniorVault.lastDepositTime(JUNIOR_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlockAt) vm.warp(unlockAt);
        uint256 shares = juniorVault.maxRequestRedeem(JUNIOR_LP);
        assertGt(shares, 0, "no unlocked junior shares");
        vm.prank(JUNIOR_LP);
        uint256 requestId = juniorVault.requestRedeem(shares, JUNIOR_LP, JUNIOR_LP);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp < maturity) vm.warp(maturity);
        vm.roll(block.number + 1);
        pool.settleLpEpoch(0, 0);
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, JUNIOR_LP);
        assertGt(claimable, 0, "junior redemption was not funded");
        uint256 beforeBalance = settlementToken.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        received = juniorVault.claimRedeem(requestId, claimable, JUNIOR_LP, JUNIOR_LP);
        assertEq(settlementToken.balanceOf(JUNIOR_LP) - beforeBalance, received);
        console2.log("H2_JUNIOR_WALLET_RECEIPT_6DP", received);
    }

    function test_H2_SingleRecapResetAndJuniorCash() public {
        uint256 oldHwm = _wipeAndPreserveHwm();
        _oneShotOwnerRecap();
        assertEq(pool.seniorPrincipal(), RECAP);
        assertEq(pool.seniorHighWaterMark(), RECAP);
        assertLt(pool.seniorHighWaterMark(), oldHwm);
        console2.log("H2_AFTER_ONE_RECAP_HWM_6DP", pool.seniorHighWaterMark());

        _testRevenue(RECOVERY);
        assertEq(pool.seniorPrincipal(), RECAP);
        assertEq(pool.juniorPrincipal(), RECOVERY);
        console2.log("H2_FINAL_SENIOR_6DP", pool.seniorPrincipal());
        console2.log("H2_FINAL_JUNIOR_6DP", pool.juniorPrincipal());
        uint256 payout = _redeemJuniorCash();
        assertGt(payout, 0);
    }

    function test_H2_NoRecapRecoveryStillRestoresOldHwm() public {
        uint256 oldHwm = _wipeAndPreserveHwm();
        _testRevenue(RECOVERY);
        console2.log("H2_NO_RECAP_HWM_6DP", pool.seniorHighWaterMark());
        console2.log("H2_NO_RECAP_SENIOR_6DP", pool.seniorPrincipal());
        console2.log("H2_NO_RECAP_JUNIOR_6DP", pool.juniorPrincipal());
        assertEq(pool.seniorHighWaterMark(), oldHwm);
        assertEq(pool.seniorPrincipal(), RECOVERY);
        assertEq(pool.juniorPrincipal(), 0);
    }

    function test_H2_OneAtomBackingChangesRecoveryRouting() public {
        uint256 oldHwm = _wipeAndPreserveHwm();
        uint256 startState = vm.snapshotState();

        // Exact wipe: the documented recapitalization bootstrap.
        _oneShotOwnerRecap();
        _testRevenue(RECOVERY);
        uint256 zeroCaseSenior = pool.seniorPrincipal();
        uint256 zeroCaseJunior = pool.juniorPrincipal();
        uint256 zeroCaseHwm = pool.seniorHighWaterMark();
        assertEq(zeroCaseSenior, RECAP);
        assertEq(zeroCaseJunior, RECOVERY);
        assertEq(zeroCaseHwm, RECAP);

        assertTrue(vm.revertToState(startState));

        // Alternative state: one additional USDG base unit is actually funded
        // and accounted as claimant revenue, rather than directly editing storage.
        _testRevenue(ONE_ATOM);
        assertEq(pool.seniorPrincipal(), ONE_ATOM);
        assertEq(pool.seniorHighWaterMark(), oldHwm);
        _oneShotOwnerRecap();
        assertEq(pool.seniorHighWaterMark(), oldHwm);
        _testRevenue(RECOVERY);
        uint256 oneAtomSenior = pool.seniorPrincipal();
        uint256 oneAtomJunior = pool.juniorPrincipal();
        console2.log("H2_ZERO_SENIOR_6DP", zeroCaseSenior);
        console2.log("H2_ZERO_JUNIOR_6DP", zeroCaseJunior);
        console2.log("H2_ZERO_HWM_6DP", zeroCaseHwm);
        console2.log("H2_ONE_ATOM_SENIOR_6DP", oneAtomSenior);
        console2.log("H2_ONE_ATOM_JUNIOR_6DP", oneAtomJunior);
        console2.log("H2_ONE_ATOM_HWM_6DP", pool.seniorHighWaterMark());
        assertEq(oneAtomSenior, RECAP + RECOVERY + ONE_ATOM);
        assertEq(oneAtomJunior, 0);
        assertGt(zeroCaseJunior, oneAtomJunior);
        assertGt(pool.seniorHighWaterMark(), zeroCaseHwm);
    }
}
```

# Appendix B — Complete matched-cash differential PoC

File: `packages/perps/test/perps/h2_audit/H2FinalMatchedCashTimingPoC.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {H2PostWipeOneShotDiagnostic} from "./H2PostWipeOneShotDiagnostic.t.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Controls total tokens, owner recap, later revenue and test snapshot.
///      Only the timing of the ONE_ATOM revenue relative to recap differs.
///      Uses test-only impairment and revenue from the inherited fixture.
contract H2FinalMatchedCashTimingPoC is H2PostWipeOneShotDiagnostic {
    function test_H2_SameCashOneAtomRevenueOrderChangesTrancheOwnership() public {
        uint256 oldHwm = _wipeAndPreserveHwm();
        uint256 state = vm.snapshotState();

        // Case A: the zero-principal bootstrap runs first.
        _oneShotOwnerRecap();
        _testRevenue(ONE_ATOM);
        _testRevenue(RECOVERY);
        uint256 cashA = pool.totalAssets();
        uint256 seniorA = pool.seniorPrincipal();
        uint256 juniorA = pool.juniorPrincipal();
        uint256 hwmA = pool.seniorHighWaterMark();
        assertEq(hwmA, RECAP, "expected a new HWM after terminal wipe");
        assertEq(seniorA, RECAP, "expected only new Senior basis");
        assertGt(juniorA, 0, "no Junior allocation");
        uint256 cashPaidA = _redeemJuniorCash();
        assertGt(cashPaidA, 0, "actual Junior transfer not proven");

        assertTrue(vm.revertToState(state), "snapshot restoration failed");

        // Case B: the identical ONE_ATOM revenue arrives before the recap.
        // Its actual backing keeps Senior principal nonzero during the recap.
        _testRevenue(ONE_ATOM);
        _oneShotOwnerRecap();
        _testRevenue(RECOVERY);
        uint256 cashB = pool.totalAssets();
        uint256 seniorB = pool.seniorPrincipal();
        uint256 juniorB = pool.juniorPrincipal();
        uint256 hwmB = pool.seniorHighWaterMark();

        console2.log("H2_MATCHED_CASH_A_6DP", cashA);
        console2.log("H2_MATCHED_CASH_B_6DP", cashB);
        console2.log("H2_MATCHED_SENIOR_A_6DP", seniorA);
        console2.log("H2_MATCHED_SENIOR_B_6DP", seniorB);
        console2.log("H2_MATCHED_JUNIOR_A_6DP", juniorA);
        console2.log("H2_MATCHED_JUNIOR_B_6DP", juniorB);
        console2.log("H2_MATCHED_HWM_A_6DP", hwmA);
        console2.log("H2_MATCHED_HWM_B_6DP", hwmB);
        console2.log("H2_MATCHED_JUNIOR_WALLET_A_6DP", cashPaidA);

        assertEq(cashA, cashB, "the end-state cash was not matched");
        assertEq(cashA, RECAP + ONE_ATOM + RECOVERY);
        assertEq(hwmB, oldHwm, "historic HWM should survive");
        assertEq(juniorB, 0, "all recovery should be allocated to Senior");
        assertGt(seniorB, seniorA, "Senior allocation unchanged");
        assertGt(juniorA, juniorB, "Junior allocation unchanged");
    }
}
```

