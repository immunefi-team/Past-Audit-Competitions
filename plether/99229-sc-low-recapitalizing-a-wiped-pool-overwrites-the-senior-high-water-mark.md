# \#99229 [SC-Low] Recapitalizing a wiped pool overwrites the senior high-water mark.

**Submitted on Nov 12th 2026 at 13:55:36 UTC by @onsec for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99229
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Permanent freezing of funds

## Description

## Brief/Intro

The bootstrap branch of `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` **assigns**
the senior high-water mark instead of raising it, and that write is persisted to storage rather than
being preview-only. The mark is the record of what senior liquidity providers are still owed after a
loss, which the canonical accounting model states a loss must never reduce. On mainnet a well-intentioned
owner recapitalization of a pool whose tranches have been wiped would silently destroy every unit of
senior protection above the recapitalized amount — in the measured case 990,000 of 1,000,000 USDG. After
it, senior principal equals the mark so the tranche reads as fully restored, the future revenue that
should have repaired senior accrues to junior instead, freed senior admission capacity reopens, and
deposits are admitted onto a still-impaired tranche.


## Vulnerability Details

The bootstrap branch of `applyClaimantRecapitalizationIntent` **assigns** the senior high-water mark
rather than raising it:

```solidity
if (state.seniorSupply > 0) {
    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;                                         // :119  additive
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;  // :120-121  ASSIGNED
        remaining = 0;
    } else {
        uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
            ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal : 0;      // :124-125
        …                                                                                      // restores up to the mark
```

Every sibling senior-principal increase **adds** — `HousePool.sol:696`, `:753`, `:1088`, and
`HousePoolWaterfallAccountingLib.sol:89`. The only legitimate reduction is the proportional withdrawal
scaling at `HousePoolWaterfallAccountingLib.sol:127`. The **else-branch of this same function**
(`:124-130`) computes `gap = seniorHighWaterMark - seniorPrincipal` and fills it, which is the mark's
documented purpose.

`ACCOUNTING_SPEC.md:337` states the invariant directly — *"Charge a shortfall to Junior first, then
Senior. **Losses do not reduce the Senior high-water mark.**"* — and `:338` routes surplus *"first to
restore Senior principal to its high-water mark, then to Junior."*

**The mutation is persisted, not preview-only.** Despite the library's name:

- `HousePool._applyPendingClaimantBucketsLive:1769` calls `applyPendingClaimantBucketsPreview(state, …)`
- `:1774` calls `_setWaterfallState(state.waterfall)`
- `:1883` writes `seniorHighWaterMark = state.seniorHighWaterMark`

**A second instance with a weaker gate.** `HousePool.assignUnassignedAssets:691-697` performs the same
assignment:

```solidity
if (toSenior) {
    if (seniorPrincipal == 0) {
        seniorPrincipal = amount;
        seniorHighWaterMark = amount;      // :693  ASSIGNED
    } else {
        seniorPrincipal += amount;
        seniorHighWaterMark += amount;     // :696  additive
    }
}
```

Its bootstrap condition is `seniorPrincipal == 0` **alone**, rather than both principals being zero as
in the preview library. The two bootstrap rules disagree, and the supplied fix addresses only the
library.

## Impact Details

With a wiped pool carrying a recorded senior entitlement, a recapitalization reduces that entitlement to
the recapitalized amount. Three consequences follow from `ACCOUNTING_SPEC.md:338`:

- senior principal now equals the mark, so the tranche reads as fully restored and future surplus no
  longer owes senior anything — the revenue that should have restored senior accrues to **junior**;
- protected exposure collapses, freeing senior admission capacity that should not exist;
- `HousePoolTrancheGateLib.trancheDepositsAllowed` no longer reports senior as impaired, so deposits
  reopen at the reset mark on a still-impaired tranche.

**Scope.** The triggering action — owner recapitalization — is benign and intended to help. It is **in
scope** under `out-of-scope.md:35`: *"In scope: a legitimate, documented action by a trusted role that
triggers a bug harming users or the protocol."*

**Severity reasoning.** High. The destroyed senior entitlement is a permanent loss of a recorded claim
for senior holders, closest to `program-severity.md:11` (*"permanent freezing of unclaimed yield"*), and
the amount is **measured in protocol storage**: 990,000 of 1,000,000 USDG erased by one benign owner
call (PoC 3 below).

On likelihood, the **second instance is the more reachable one and governs**. The preview library's
bootstrap needs `seniorPrincipal == 0 && juniorPrincipal == 0` — a full wipe. `HousePool.sol:692` needs
`seniorPrincipal == 0` **alone**, which `ACCOUNTING_SPEC.md:337` describes as the *expected* outcome of a
loss: senior principal charged to zero with the mark deliberately left standing. That is an ordinary
impaired state, not an exotic one — and the supplied fix patches only the library, leaving it live.


## References
libraries/HousePoolPendingPreviewLib.sol:118-122`; **second instance** at `HousePool.sol:691-697`


## Proof of Concept

## Proof of Concept

Three tests across two files — **3/3 PASS**. Two prove the defective operator at the library level
with a control; the third proves the loss **in protocol storage** through a real owner call.

```
test/perps/spec/lp/SeniorHwmResetC25.t.sol
[PASS] test_C25_recapitalizingAWipedPoolDestroysRecordedSeniorEntitlement() (gas: 16658)
Logs:
  senior high-water mark before recapitalization: 1000000000000
  senior high-water mark after  recapitalization: 10000000000
  senior principal after recapitalization: 10000000000
  senior protection destroyed: 990000000000
[PASS] test_C25_nonBootstrapBranchPreservesTheMark() (gas: 7919)

test/perps/spec/lp/SeniorHwmStorageC25.t.sol
[PASS] test_C25_assigningUnassignedAssetsErasesSeniorEntitlementInStorage() (gas: 602708)
Logs:
  seniorHighWaterMark in storage BEFORE: 1000000000000
  seniorHighWaterMark in storage AFTER : 10000000000
  seniorPrincipal in storage AFTER     : 10000000000
  senior protection destroyed in storage: 990000000000
```

**PoC 1 — the defect.** A wiped pool (both principals zero, senior shares outstanding) carries a
mark of 1,000,000 USDG and receives a 10,000 USDG recapitalization. The mark is overwritten to
10,000 USDG — **990,000 USDG of recorded senior protection destroyed** — and senior principal then
equals the mark, so the tranche reads fully restored despite the unrecovered loss.

**PoC 2 — the control.** With `seniorPrincipal == 1` the non-bootstrap branch leaves the mark
standing at 1,000,000 USDG, which is the behaviour the bootstrap branch should match. This rules
out the counter-reading that the library simply never retains the mark.

**PoC 3 — the loss, in protocol storage.** PoCs 1 and 2 are `unit` tests: the target is
`internal pure` over a memory struct, so they run through a test-only shim and prove the wrong
operator rather than a recorded loss. PoC 3 closes that gap against the **second instance**,
`HousePool.assignUnassignedAssets:691-697`. `HousePool.seniorHighWaterMark` is a public storage
variable (`:153`), so the before and after figures are read straight out of storage, either side of
a real `onlyOwner` call. The oracle-live, fresh-mark and senior-capacity guards on that path
(`:669-671`, `:684-688`) do **not** block it.

**What PoC 3 does not show.** The impaired precondition — `seniorPrincipal == 0` with the mark
standing and unassigned value available — is installed with `stdstore`, not reached by trading. So
PoC 3 demonstrates **persistence and the amount lost**, not organic reachability. `vm.store` is
already used in five existing test files in this suite, so the technique is consistent with the
project's own practice, but it is not the same as driving a real loss and is not presented as such.
The reachability argument rests on `ACCOUNTING_SPEC.md:337` describing senior-impaired-to-zero as
the expected result of a loss.

### Exact PoC code

Save verbatim as `packages/perps/test/perps/spec/lp/SeniorHwmResetC25.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PROVENANCE: verifies an EXTERNALLY SUPPLIED finding (pashov-findings.md #12), read at the user's
// explicit direction. NOT independently derived. See the independence boundary in
// new-leads-findings-run3.md.
//
// CLAIM UNDER TEST: HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent:118-122 ASSIGNS the
// senior high-water mark on its bootstrap branch:
//     state.waterfall.seniorHighWaterMark =
//         recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
// Every sibling senior-principal increase adds instead (HousePool.sol:696, :753, :1088,
// HousePoolWaterfallAccountingLib.sol:89), and the else-branch of this same function (:124-130) reads
// the mark as an entitlement to restore. ACCOUNTING_SPEC.md:337 states "Losses do not reduce the Senior
// high-water mark"; :338 routes surplus "first to restore Senior principal to its high-water mark".
//
// The mutation is persisted, not preview-only: HousePool._applyPendingClaimantBucketsLive:1769 calls
// applyPendingClaimantBucketsPreview and :1774 calls _setWaterfallState, which writes
// seniorHighWaterMark at :1883.
//
// HARM ASSERTED: with a wiped pool (both principals zero) carrying a large recorded senior entitlement,
// a modest recapitalization REDUCES that entitlement to the recapitalized amount. The difference is
// senior protection destroyed; by :338 the surplus that should have restored it now routes to junior.
//
// No in-scope contract is modified by this file. The wrapper below only exposes an internal library.

import {Test} from "forge-std/Test.sol";
import {HousePoolPendingPreviewLib} from "@plether/perps/libraries/HousePoolPendingPreviewLib.sol";
import {HousePoolWaterfallAccountingLib} from "@plether/perps/libraries/HousePoolWaterfallAccountingLib.sol";

/// @dev Test-only shim: the library function is `internal pure`, so it needs an external surface.
contract PendingPreviewShim {
    function applyRecap(
        HousePoolPendingPreviewLib.PendingAccountingState memory state,
        uint256 amount,
        uint256 recapitalizationTargetAssets
    ) external pure returns (HousePoolPendingPreviewLib.PendingAccountingState memory) {
        HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent(state, amount, recapitalizationTargetAssets);
        return state;
    }
}

contract SeniorHwmResetC25Test is Test {

    PendingPreviewShim internal shim;

    uint256 internal constant RECORDED_SENIOR_ENTITLEMENT = 1_000_000e6; // mark standing after a wipe
    uint256 internal constant RECAPITALIZATION = 10_000e6;

    function setUp() public {
        shim = new PendingPreviewShim();
    }

    function test_C25_recapitalizingAWipedPoolDestroysRecordedSeniorEntitlement() public {
        // A wiped pool: both principals at zero, senior shares still outstanding, and the high-water
        // mark still recording what senior is owed -- which ACCOUNTING_SPEC.md:337 says a loss must not
        // reduce.
        HousePoolPendingPreviewLib.PendingAccountingState memory state;
        state.waterfall = HousePoolWaterfallAccountingLib.WaterfallState({
            seniorPrincipal: 0,
            juniorPrincipal: 0,
            seniorHighWaterMark: RECORDED_SENIOR_ENTITLEMENT
        });
        state.seniorSupply = 1_000_000e18;

        HousePoolPendingPreviewLib.PendingAccountingState memory after_ =
            shim.applyRecap(state, RECAPITALIZATION, RECAPITALIZATION);

        emit log_named_uint("senior high-water mark before recapitalization", RECORDED_SENIOR_ENTITLEMENT);
        emit log_named_uint("senior high-water mark after  recapitalization", after_.waterfall.seniorHighWaterMark);
        emit log_named_uint("senior principal after recapitalization", after_.waterfall.seniorPrincipal);

        // Principal rises by the recapitalized amount, as intended.
        assertEq(after_.waterfall.seniorPrincipal, RECAPITALIZATION, "principal credited");

        // THE HARM: the mark is overwritten downward rather than left standing.
        assertEq(
            after_.waterfall.seniorHighWaterMark,
            RECAPITALIZATION,
            "mark was ASSIGNED to the recapitalized amount, not left at the recorded entitlement"
        );
        assertLt(
            after_.waterfall.seniorHighWaterMark,
            RECORDED_SENIOR_ENTITLEMENT,
            "recorded senior entitlement was reduced by a recapitalization"
        );
        emit log_named_uint(
            "senior protection destroyed", RECORDED_SENIOR_ENTITLEMENT - after_.waterfall.seniorHighWaterMark
        );

        // And the consequence :338 depends on: senior principal now equals the mark, so the tranche
        // reads as fully restored and future surplus no longer owes senior anything.
        assertEq(
            after_.waterfall.seniorPrincipal,
            after_.waterfall.seniorHighWaterMark,
            "senior now reads as fully restored despite the unrecovered loss"
        );
    }

    /// @notice Control: with a nonzero principal the sibling branch RAISES the mark, which is the
    ///         behaviour the bootstrap branch should match.
    function test_C25_nonBootstrapBranchPreservesTheMark() public {
        HousePoolPendingPreviewLib.PendingAccountingState memory state;
        state.waterfall = HousePoolWaterfallAccountingLib.WaterfallState({
            seniorPrincipal: 1,
            juniorPrincipal: 0,
            seniorHighWaterMark: RECORDED_SENIOR_ENTITLEMENT
        });
        state.seniorSupply = 1_000_000e18;

        HousePoolPendingPreviewLib.PendingAccountingState memory after_ =
            shim.applyRecap(state, RECAPITALIZATION, RECAPITALIZATION);

        assertEq(
            after_.waterfall.seniorHighWaterMark,
            RECORDED_SENIOR_ENTITLEMENT,
            "the non-bootstrap branch leaves the recorded entitlement standing"
        );
    }
}
```

Save verbatim as `packages/perps/test/perps/spec/lp/SeniorHwmStorageC25.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PROVENANCE: verifies an EXTERNALLY SUPPLIED finding (pashov-findings.md #12), read at the user's
// explicit direction. NOT independently derived. See the independence boundary in
// new-leads-findings-run3.md.
//
// WHY THIS FILE EXISTS. SeniorHwmResetC25.t.sol proves the defect at the library level, through a shim
// over an `internal pure` function on a hand-built memory struct. That demonstrates the wrong operator
// but not that the protocol's own storage records the loss. This file closes that gap against the SECOND
// instance of the same defect, HousePool.assignUnassignedAssets:691-697:
//
//     if (seniorPrincipal == 0) {
//         seniorPrincipal = amount;
//         seniorHighWaterMark = amount;      // :693  ASSIGNED, not raised
//     } else {
//         seniorPrincipal += amount;
//         seniorHighWaterMark += amount;     // :696  additive
//     }
//
// That instance is gated on `seniorPrincipal == 0` ALONE -- strictly weaker than the preview library's
// `seniorPrincipal == 0 && juniorPrincipal == 0` -- so it fires in the ordinary senior-impaired state,
// which ACCOUNTING_SPEC.md:337 describes as expected: a loss zeroes senior principal while deliberately
// leaving the high-water mark standing. The supplied fix patches only the library, leaving this reachable.
//
// WHAT IS AND IS NOT DEMONSTRATED. The impaired precondition is installed with stdstore rather than
// reached by trading, so this proves PERSISTENCE and the VALUE LOST, not organic reachability.
// `HousePool.seniorHighWaterMark` is a public storage variable (:153), so the before/after figures below
// are read straight out of protocol storage.
//
// No in-scope contract is modified by this file.

import {BasePerpTest} from "../../BasePerpTest.sol";
import {StdStorage, stdStorage} from "forge-std/StdStorage.sol";

contract SeniorHwmStorageC25Test is BasePerpTest {

    using stdStorage for StdStorage;

    uint256 internal constant RECORDED_SENIOR_ENTITLEMENT = 1_000_000e6;
    uint256 internal constant UNASSIGNED = 10_000e6;

    function setUp() public override {
        super.setUp();
        _bootstrapSeededLifecycle();
    }

    function test_C25_assigningUnassignedAssetsErasesSeniorEntitlementInStorage() public {
        // Install the documented senior-impaired state: principal charged to zero by a loss, the mark
        // left standing as the record of what senior is still owed (ACCOUNTING_SPEC.md:337), and some
        // unassigned value available to assign.
        stdstore.target(address(pool)).sig("seniorPrincipal()").checked_write(uint256(0));
        stdstore.target(address(pool)).sig("seniorHighWaterMark()").checked_write(RECORDED_SENIOR_ENTITLEMENT);
        stdstore.target(address(pool)).sig("unassignedAssets()").checked_write(UNASSIGNED);

        assertEq(pool.seniorPrincipal(), 0, "precondition: senior principal impaired to zero");
        assertEq(
            pool.seniorHighWaterMark(),
            RECORDED_SENIOR_ENTITLEMENT,
            "precondition: the mark still records what senior is owed"
        );
        assertEq(pool.unassignedAssets(), UNASSIGNED, "precondition: unassigned value available");

        uint256 markBefore = pool.seniorHighWaterMark();

        // The owner performs a documented, benign action: assign unassigned value to senior.
        vm.prank(pool.owner());
        pool.assignUnassignedAssets(true, address(this));

        uint256 markAfter = pool.seniorHighWaterMark();

        emit log_named_uint("seniorHighWaterMark in storage BEFORE", markBefore);
        emit log_named_uint("seniorHighWaterMark in storage AFTER ", markAfter);
        emit log_named_uint("seniorPrincipal in storage AFTER     ", pool.seniorPrincipal());

        // THE HARM, read out of protocol storage rather than from a memory struct.
        assertLt(markAfter, markBefore, "the recorded senior entitlement was reduced by a benign owner action");
        assertEq(markAfter, UNASSIGNED, "the mark was ASSIGNED to the assigned amount, not raised");
        emit log_named_uint("senior protection destroyed in storage", markBefore - markAfter);

        // And the consequence ACCOUNTING_SPEC.md:338 depends on: senior now reads as fully restored, so
        // future surplus owes it nothing.
        assertEq(
            pool.seniorPrincipal(),
            pool.seniorHighWaterMark(),
            "senior reads as fully restored despite the unrecovered loss"
        );
    }
}
```

### Replication procedure

1. **Check out the audited commit.**

   ```bash
   git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
   ```

2. **Install dependencies.**

   ```bash
   forge --version        # expect a toolchain supporting solc 0.8.35
   git submodule update --init --recursive
   ```

3. **Create both test files**, with the sources above:

   | File | What it proves |
   |---|---|
   | `packages/perps/test/perps/spec/lp/SeniorHwmResetC25.t.sol` | the defective operator, plus the control |
   | `packages/perps/test/perps/spec/lp/SeniorHwmStorageC25.t.sol` | the 990,000 USDG loss in protocol storage |

   No file under `packages/perps/src` is created, edited or deleted. The build settings are already
   in `packages/perps/foundry.toml`: `solc = "0.8.35"`, `optimizer = true`, and the
   `@plether/perps/ -> src/` remapping.

4. **Run them.**

   ```bash
   forge test --root packages/perps \
     --match-path "test/perps/spec/lp/SeniorHwm*C25.t.sol" -vv
   ```

5. **Expected output** — all three pass against unpatched code:

   ```
   [PASS] test_C25_recapitalizingAWipedPoolDestroysRecordedSeniorEntitlement()
     senior protection destroyed: 990000000000
   [PASS] test_C25_nonBootstrapBranchPreservesTheMark()
   [PASS] test_C25_assigningUnassignedAssetsErasesSeniorEntitlementInStorage()
     seniorHighWaterMark in storage BEFORE: 1000000000000
     seniorHighWaterMark in storage AFTER : 10000000000
     senior protection destroyed in storage: 990000000000
   ```

6. **Confirm scope was untouched.**

   ```bash
   git diff --ignore-cr-at-eol --stat -- packages/perps/src   # must print nothing
   ```

**Harness notes.** PoCs 1-2 need no protocol bootstrap and run in milliseconds; the shim exists
only because the target is `internal pure`. PoC 3 uses `BasePerpTest` with
`_bootstrapSeededLifecycle()` so the oracle-live and fresh-mark guards pass, and `stdstore` to
install the impaired precondition. No existing test file is modified.
## Recommendation

```diff
-        state.waterfall.seniorHighWaterMark =
-            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
+        uint256 bootstrapMark = recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
+        if (state.waterfall.seniorHighWaterMark < bootstrapMark) {
+            state.waterfall.seniorHighWaterMark = bootstrapMark;
+        }
```

