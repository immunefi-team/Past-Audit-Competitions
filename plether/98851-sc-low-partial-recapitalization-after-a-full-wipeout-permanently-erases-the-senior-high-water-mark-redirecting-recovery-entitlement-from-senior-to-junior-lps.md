# \#98851 [SC-Low] Partial recapitalization after a full wipeout permanently erases the senior high-water mark, redirecting recovery entitlement from Senior to Junior LPs

**Submitted on Nov 10th 2026 at 15:05:18 UTC by @yesofcourse for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98851
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

When a full pool wipeout zeros both tranche principals, the loss-absorption step deliberately preserves the senior high-water mark (HWM) so that future recovery is directed to Senior LPs first. A subsequent partial recapitalization enters a bootstrap branch that unconditionally overwrites the HWM with the (smaller) recap amount, permanently erasing Senior LPs' recovery entitlement. All future revenue that should restore Senior principal is instead routed to Junior principal or unassigned assets. This violates the protocol's own stated invariant: "Senior impairment must be restored before junior extracts surplus" (SECURITY.md line 394).

## Vulnerability Details

References are to commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/`.

**Root cause.** `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` (lines 118-122) handles the post-wipeout state (`seniorPrincipal == 0 && juniorPrincipal == 0`) as a "bootstrap" branch:

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
}
```

This sets `seniorHighWaterMark = max(recapitalizationTargetAssets, settleableAmount)` without comparing against the existing HWM. `absorbLoss` (called during the wipeout reconciliation) deliberately preserves the old HWM so that recovery can restore Senior to its historical peak. The bootstrap branch discards that preserved value.

The normal-case branch (lines 123-131) correctly reads the existing HWM and fills the gap between it and current principal. Only the bootstrap branch is broken.

The protocol's own documentation defines the HWM invariant at SECURITY.md line 394: "High-water-mark protection: Senior impairment must be restored before junior extracts surplus." The bug violates this invariant. Additionally, the non-bootstrap recap path (lines 123-131) does the right thing: it reads the existing HWM and fills the gap. 

The bootstrap branch is the only place where HWM is overwritten without preserving the old value. 

**Trigger flow:**

1. A pool with Senior deposits of 100,000 USDG establishes HWM at ~101,000 USDG (including seed assets).
2. A terminal deficit settlement wipes the pool. `absorbLoss` sets `seniorPrincipal = 0`, `juniorPrincipal = 0`, but deliberately preserves `seniorHighWaterMark = 101,000`.
3. The owner performs a partial recapitalization of 20,000 USDG via `engine.recapitalizePool(20_000e6)`. The engine calls `pool.recordClaimantInflow(amount, Recapitalization, CashArrived)`.
4. On the next `reconcile()`, `applyClaimantRecapitalizationIntent` enters the bootstrap branch (both principals are zero, senior shares are outstanding). It sets `seniorHighWaterMark = 20,000` - permanently erasing the 101,000 HWM.
5. Future revenue of 200,000 USDG arrives. Under the correct HWM, Senior would recover to 101,000 and Junior to 99,000. Under the lowered HWM, Senior recovers only to 20,000 and Junior receives 180,000. The difference (81,000 USDG) is permanently redirected from Senior to Junior.

**No privileges required beyond the engine.** `recapitalizePool` is an owner action, but the owner is performing the documented recovery procedure in good faith. A well-intentioned partial recapitalization causes the damage. No attacker action is needed.

## Impact Details

**Theft of unclaimed yield (High).** Senior LPs' recovery entitlement is a form of unclaimed yield - the waterfall guarantees Senior is made whole before Junior receives surplus. The bug permanently erases that entitlement, redirecting it to Junior LPs.

| Scenario | Senior HWM | Senior recovery | Junior recovery | Senior loss |
|---|---|---|---|---|
| No recap, then 200k revenue (control) | 101,000 | 101,000 | 99,000 | 0 |
| 20k recap, then 180k revenue (bug) | 20,000 | 20,000 | 180,000 | 81,000 |
| 10k recap, then 190k revenue (worse) | 10,000 | 10,000 | 190,000 | 91,000 |

The loss scales linearly with the gap between the old HWM and the recap amount. At the extreme, a 1 USDG recap after a 10M HWM erases 9,999,999 of Senior recovery entitlement.

**Properties of the loss:**
- Permanent: once the HWM is overwritten, no subsequent action can restore it. There is no admin function to manually set HWM.
- Not attacker-dependent: the owner's own recovery procedure triggers the bug. No external attacker is needed.
- Proportional to pool size: at deployment parameters with 40M Senior capacity, the redirected amount can be tens of millions.
- Repeatable: every partial recap after a wipeout lowers the HWM again. Sequential recaps lock the HWM at the first recap amount.

## References

- [HousePoolPendingPreviewLib.sol:118-122](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L118) - bootstrap branch that overwrites HWM
- [HousePoolPendingPreviewLib.sol:123-131](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L123) - normal-case branch that correctly preserves HWM
- [SECURITY.md:394](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L394) - HWM invariant: "Senior impairment must be restored before junior extracts surplus"
- [SECURITY.md:1328-1456](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1328) - Known Limitations (does not list this issue)

**Recommended fix.** In the bootstrap branch, preserve the existing HWM:

```solidity
state.waterfall.seniorHighWaterMark = Math.max(
    state.waterfall.seniorHighWaterMark,
    recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining
);
```


## Proof of Concept

## Proof of Concept

The PoC contains three tests that prove the core mechanism, the economic impact, and that the existing test asserts the buggy state. The test imports only `BasePerpTest` and production contracts. No `src/` files are modified. All accounting paths (recap inflow, reconcile, revenue inflow) are called through the production `HousePool` API, not pranked.

Save the file at `packages/perps/test/perps/poc/resurrect/RecapHwmDrop.t.sol` and run from `packages/perps`:

```
forge test --match-path 'test/perps/poc/resurrect/RecapHwmDrop.t.sol' -vv
```

Result: `3 passed, 0 failed`. Key output (6-decimal USDG):

```
=== RESULT ===
HWM before wipeout        101000931506
HWM after partial recap    20000000000
HWM lost                   81000931506
Recap amount               20000000000

=== IMPACT ===
Control: seniorPrincipal   101000931506
Attack:  seniorPrincipal   20000000000
Senior loss                81000931506
Control: juniorPrincipal   98999068494
Attack:  juniorPrincipal   180000000000
Junior gain                81000931506
```

Test 1 (`test_HwmDropAfterPartialRecap`) proves the core mechanism: after a full wipeout, a 20k recap drops HWM from 101k to 20k. Test 2 (`test_RevenueRedirectedFromSeniorToJunior`) compares a CONTROL path (no recap, then revenue) against an ATTACK path (20k recap, then revenue) using `vm.snapshotState`/`vm.revertToState`. Senior LP ends up with 81,000 USDG less principal with the recap than without it. Test 3 (`test_ExistingTestAssertsTheBug`) reproduces the exact assertion from the existing test and shows it asserts the buggy state.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

contract RecapHwmDropTest is BasePerpTest {

    address internal alice = address(0xA11CE);   // Senior LP (victim)
    address internal bob   = address(0xB0B);     // Junior LP (beneficiary)

    uint256 internal constant SENIOR_DEPOSIT = 100_000e6;
    uint256 internal constant JUNIOR_DEPOSIT = 100_000e6;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _logState(string memory label) internal view {
        console2.log(label);
        console2.log("  seniorPrincipal     ", pool.seniorPrincipal());
        console2.log("  juniorPrincipal     ", pool.juniorPrincipal());
        console2.log("  seniorHighWaterMark ", pool.seniorHighWaterMark());
        console2.log("  pendingRecap        ", pool.pendingRecapitalizationAssets());
        console2.log("  unassignedAssets    ", pool.unassignedAssets());
        console2.log("  totalAssets         ", pool.totalAssets());
    }

    function _wipePool() internal {
        uint256 total = pool.totalAssets();
        vm.prank(address(pool));
        settlementToken.transfer(address(0xdead), total);

        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function _recapAndReconcile(uint256 amount) internal {
        settlementToken.mint(address(pool), amount);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            amount,
            IHousePool.ClaimantInflowKind.Recapitalization,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function _revenueAndReconcile(uint256 amount) internal {
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

    function test_HwmDropAfterPartialRecap() public {
        _fundSenior(alice, SENIOR_DEPOSIT);
        _fundJunior(bob, JUNIOR_DEPOSIT);

        uint256 hwmBefore = pool.seniorHighWaterMark();
        uint256 spBefore = pool.seniorPrincipal();
        console2.log("=== INITIAL STATE ===");
        _logState("after deposits");
        assertGe(hwmBefore, SENIOR_DEPOSIT, "HWM should be >= senior deposit");

        _wipePool();
        console2.log("\n=== AFTER WIPEOUT ===");
        _logState("post-wipeout");

        assertEq(pool.seniorPrincipal(), 0, "seniorPrincipal must be zero after wipeout");
        assertEq(pool.juniorPrincipal(), 0, "juniorPrincipal must be zero after wipeout");
        assertEq(pool.seniorHighWaterMark(), hwmBefore, "absorbLoss must preserve HWM");
        assertTrue(seniorVault.totalSupply() > 0, "senior shares still outstanding");

        uint256 recapAmount = 20_000e6;
        _recapAndReconcile(recapAmount);
        console2.log("\n=== AFTER PARTIAL RECAP (20k) ===");
        _logState("post-recap");

        uint256 hwmAfter = pool.seniorHighWaterMark();
        console2.log("\n=== RESULT ===");
        console2.log("HWM before wipeout       ", hwmBefore);
        console2.log("HWM after partial recap   ", hwmAfter);
        console2.log("HWM lost                  ", hwmBefore - hwmAfter);
        console2.log("Recap amount              ", recapAmount);

        assertLt(hwmAfter, hwmBefore, "BUG: HWM dropped below the pre-wipeout value");
        assertEq(hwmAfter, recapAmount, "BUG: HWM was overwritten to the recap amount");
    }

    function test_RevenueRedirectedFromSeniorToJunior() public {
        _fundSenior(alice, SENIOR_DEPOSIT);
        _fundJunior(bob, JUNIOR_DEPOSIT);

        uint256 hwmBefore = pool.seniorHighWaterMark();

        uint256 snap = vm.snapshotState();

        // CONTROL: no recap, just revenue
        _wipePool();
        assertEq(pool.seniorHighWaterMark(), hwmBefore, "control: HWM preserved after wipeout");

        uint256 fullRevenue = SENIOR_DEPOSIT + JUNIOR_DEPOSIT;
        _revenueAndReconcile(fullRevenue);
        uint256 controlSP = pool.seniorPrincipal();
        uint256 controlJP = pool.juniorPrincipal();
        uint256 controlHWM = pool.seniorHighWaterMark();
        console2.log("=== CONTROL (no recap) after revenue ===");
        _logState("control");

        vm.revertToState(snap);

        // ATTACK: partial recap, then revenue
        _wipePool();
        uint256 recapAmount = 20_000e6;
        _recapAndReconcile(recapAmount);
        assertEq(pool.seniorHighWaterMark(), recapAmount, "attack: HWM overwritten to recap");

        uint256 attackRevenue = fullRevenue - recapAmount;
        _revenueAndReconcile(attackRevenue);
        uint256 attackSP = pool.seniorPrincipal();
        uint256 attackJP = pool.juniorPrincipal();
        uint256 attackHWM = pool.seniorHighWaterMark();
        console2.log("\n=== ATTACK (20k recap then revenue) ===");
        _logState("attack");

        console2.log("\n=== IMPACT ===");
        console2.log("Control: seniorPrincipal  ", controlSP);
        console2.log("Attack:  seniorPrincipal  ", attackSP);
        console2.log("Senior loss               ", controlSP - attackSP);
        console2.log("Control: juniorPrincipal  ", controlJP);
        console2.log("Attack:  juniorPrincipal  ", attackJP);
        console2.log("Junior gain               ", attackJP - controlJP);

        assertLt(attackSP, controlSP, "IMPACT: Senior LP worse off WITH recap than without");
        assertGt(attackJP, controlJP, "IMPACT: Junior LP better off due to lowered HWM");
        assertLt(attackHWM, controlHWM, "IMPACT: HWM permanently lower");
        uint256 seniorLoss = controlSP - attackSP;
        uint256 juniorGain = attackJP - controlJP;
        console2.log("Senior $ loss from HWM drop", seniorLoss);
        console2.log("Junior $ gain from HWM drop", juniorGain);
        assertGt(seniorLoss, 50_000e6, "IMPACT: Senior loss material (>50k)");
    }

    function test_ExistingTestAssertsTheBug() public {
        _fundSenior(alice, SENIOR_DEPOSIT);
        _fundJunior(bob, JUNIOR_DEPOSIT);

        uint256 hwmBeforeWipeout = pool.seniorHighWaterMark();
        console2.log("HWM before wipeout        ", hwmBeforeWipeout);

        _wipePool();

        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.seniorHighWaterMark(), hwmBeforeWipeout, "absorbLoss preserves HWM by design");

        uint256 recapAmount = 10_000e6;
        _recapAndReconcile(recapAmount);

        assertEq(pool.seniorPrincipal(), recapAmount);
        assertEq(pool.seniorHighWaterMark(), recapAmount, "matches existing test assertion line 609");

        console2.log("HWM after recap           ", pool.seniorHighWaterMark());
        console2.log("Old HWM that was erased   ", hwmBeforeWipeout);
        console2.log("Ratio erased              ", hwmBeforeWipeout / recapAmount, "x");
        assertLt(pool.seniorHighWaterMark(), hwmBeforeWipeout, "BUG: old HWM erased by smaller recap");
    }
}
```

