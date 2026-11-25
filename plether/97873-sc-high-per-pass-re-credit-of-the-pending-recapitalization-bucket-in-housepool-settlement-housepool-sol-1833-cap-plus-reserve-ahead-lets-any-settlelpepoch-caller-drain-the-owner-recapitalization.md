# \#97873 [SC-High] Per-pass re-credit of the pending recapitalization bucket in HousePool settlement (`HousePool.sol:1833` cap plus reserve-ahead) lets any `settleLpEpoch` caller drain the owner recapitalization and Senior LP principal into Junior LPs

**Submitted on Nov 3rd 2026 at 17:27:44 UTC by @thorthumbz00 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97873
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
After the owner recapitalizes the pool at a dip, each permissionless `settleLpEpoch` pass caps the settleable recapitalization bucket by the whole distributable, applies it, and decreases the bucket, then charges last pass's credited value back to Senior as a loss and re-credits it. Repeating the pass walks the bucket down to zero and moves the recapitalization plus Senior's entitlement into Junior holders. I reproduced the owner's 56,727.53 USDG of unassigned recapitalization falling to 0 and Senior falling from 100,007.31 to 19,077.45 USDG while Junior gained 137,671.35 USDG, all driven by a Junior LP's redeem request and a stranger's settlement calls, with no market move between passes.

**This report is distinct from the high-water-mark overwrite reported separately (report #97866).** That finding is a single pass that rewrites the mark. This finding is a repeated-pass accounting error at a different fix site: the per-pass cap and the reserve-ahead subtraction, which together re-credit and burn the recapitalization bucket pass by pass. The two have separate root-cause lines and separate fixes.

## Vulnerability Details
Each pass, `HousePool.sol:1565-1566` builds the settleable claimant buckets through `_settleablePendingClaimantBuckets`, which at `HousePool.sol:1833` calls `HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable)`, capping the recapitalization bucket by the whole distributable rather than by what remains unsettled. `HousePool.sol:1769-1772` then applies the capped buckets to the waterfall (`applyPendingClaimantBucketsPreview`) and calls `_decreasePendingClaimantBuckets`. Because `HousePoolReconcilePlanLib.sol:68-71` subtracts the whole pending bucket from `distributable` before principal is restored, the value credited on the previous pass is treated as a loss to Senior on the next pass and re-credited from the bucket, so the bucket falls each pass and the restored value lands in Junior instead of restoring Senior. The log shows the bucket stepping 44,177.56 -> 31,627.60 -> 19,077.64 -> 6,527.68 -> 0 over five passes.

This defeats the same documented invariant as the overwrite, "Senior impairment must be restored before junior extracts surplus" (SECURITY.md:394), and the rule that recapitalization value "continues through the canonical claimant/unassigned routing" and "cannot mint new senior LP shares" (SECURITY.md:1210-1214). The owner recapitalization is a documented trusted action (SECURITY.md:157), in scope because a stranger's permissionless passes then harm Senior LPs (rules-and-trust-model.md:33-35).

**Who can trigger it, and what it needs.** Trigger: any EOA. A Junior holder submits a whole-balance redeem request (no minimum), and a stranger calls `router.settleLpEpoch` repeatedly (keepers are permissionless, rules-and-trust-model.md:39-40). Precondition: a documented owner recapitalization at a dip. Capital: none beyond gas; the Junior LP driving the passes gains. No market move is needed between passes. Repeatable until the bucket is zero. A pause halts settlement but does not restore the burned value.

**Recommended fix.** Cap the settleable recapitalization bucket by the amount still unsettled in the bucket, not by the whole distributable (`HousePool.sol:1833`), and reconcile principal restoration before reserving the bucket out of `distributable` (`HousePoolReconcilePlanLib.sol:68-71`), so a pass cannot re-credit value it already applied.

## Impact Details
Direct theft of Senior LP principal and of the owner's recapitalization (at-rest USDG in HousePool), moved to Junior LPs, permanent once exit completes. On release parameters: owner unassigned recapitalization 56,727.53 USDG burned to 0; Senior LP 100,007.31 -> 19,077.45 USDG (a loss of 80,929.86); Junior LP +137,671.35 USDG. The Junior LP that forces the passes profits. Blast radius scales to the recapitalization size and the Senior tranche (cap 40M USDG).

### Before / after state delta (USDG, 6 decimals)
| Account / state | Before (no pass) | After (repeated passes) | Delta |
|---|---|---|---|
| Owner unassigned recapitalization | 56,727.53 | 0 | -56,727.53 |
| Senior LP collected | 100,007.31 | 19,077.45 | -80,929.86 |
| Junior LP collected | 45,770.09 | 183,441.44 | +137,671.35 |
| Pending recapitalization bucket | 56,727.53 | 0 | drained over 5 passes |

## References
- `packages/perps/src/HousePool.sol:1565-1566,1769-1772,1833` (per-pass cap, apply, decrease) at a82d4fef.
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:68-71` (bucket reserved ahead of principal).
- SECURITY.md:157, :394, :1210-1214. In scope under rules-and-trust-model.md:33-35.
- Prior audits: Cecuro MED-12 is a revenue-continuation fix, a different mechanism; not this repeated-pass re-credit. Not in known-issues.


## Link to Proof of Concept

https://gist.github.com/tdev00/b3efefa76a9b241bd4e06228b4708c0f

## Proof of Concept

## Proof of Concept

**Attack, step by step (as the test executes it, release parameters from DEPLOYMENT.md).**
1. A Senior LP and a Junior LP deposit into HousePool through the tranche vaults (about 100,000 USDG each in the test).
2. A book of trader positions is opened on both sides; one trader's winning close at a lower mark drives the pool into degraded mode (the test asserts `engine.degradedMode()` is latched by that real close).
3. The owner performs the documented recapitalization (`CfdEngine.recapitalizePool`, 56,727.53 USDG in the test, sized to maximum liability plus a margin) and clears degraded mode (`clearDegradedMode`, owner-only). Both are the trusted owner actions SECURITY.md describes; they are not the bug. The recapitalization sits in the pending bucket as the owner's unassigned value.
4. The mark moves against the pool (price 2e8) and the losing Long is liquidated at the dip (in the test the liquidation call carries the router's authority, as any keeper's call through the router does in production).
5. Each pass at the dip, with no market move between passes: the Junior LP requests a whole-balance redemption, a stranger with no role (`address(0xABC)`) calls `router.settleLpEpoch()`, and the Junior LP claims its redeem refund. The reconcile caps the settleable recapitalization bucket by the whole distributable (HousePool.sol:1833), applies it, decreases the bucket, and reserves the remaining bucket ahead of principal (HousePoolReconcilePlanLib.sol:68-71), so last pass's credited value reads as a new Senior loss and is re-credited from the bucket. The bucket steps 44,177.56, 31,627.60, 19,077.64, 6,527.68, 0 over five passes.
6. The market returns and every LP exits. The owner's unassigned recapitalization is 0 instead of 56,727.53; the Senior LP collects 19,077.45 USDG where the twin run with no passes collects 100,007.31; the Junior LP collects 183,441.44 instead of 45,770.09.

**Runnable test.** A Foundry test that runs through the `BasePerpTest` harness. Only Pyth and USDG are mocked (harness `MockPyth` and the mock settlement token), which the rules permit. The full test file (`TriageLpRecap.t.sol`) and the captured forge log are in the secret Gist linked below.

Steps:
1. Clone `github.com/immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (submodules included) and install solc 0.8.35 (the repository's pinned compiler).
2. Create `poc/` at the repository root and copy `TriageLpRecap.t.sol` into it. The root `foundry.toml` remaps `@plether/perps/<File>.sol` to `packages/perps/src/<File>.sol`; the file imports the harness as `../packages/perps/test/perps/BasePerpTest.sol`.
3. Run, from the repository root, with the test directory pointed at `poc/` (the repository's own test and script trees are skipped so only the attached file compiles as a test):
   ```
   export FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]'
   forge test --match-path poc/TriageLpRecap.t.sol --match-test test_Release_realRouterPassesBurnRecap -vv
   ```
   The first compile takes a few minutes (via_ir). The test prints `[PASS]` with the figures quoted below.

Test, `contract O14ReleaseParamsTest`. A stranger `address(0xABC)` calls `router.settleLpEpoch` once per pass at the dip, each driven by a Junior LP whole-balance redeem request:

```solidity
function test_Release_realRouterPassesBurnRecap() public {
    uint256 snap = vm.snapshotState();
    (Out memory twin,) = _runBurn(0);     // no pass at the dip
    assertTrue(vm.revertToState(snap));
    (Out memory run, uint256 recap) = _runBurn(12);  // up to 12 stranger passes at the dip
    assertLt(run.unassigned, twin.unassigned, "owner's unassigned below the twin");
    assertLt(run.senior, twin.senior, "senior LP below the twin");
    assertGt(run.junior, twin.junior, "recap value reached junior");
}
// inside _runBurn: each pass -> juniorVault.requestRedeem(...); vm.prank(stranger); router.settleLpEpoch{value:...}(data);
```

Captured output (`TriageLpRecap.log` in the Gist):

```
[PASS] test_Release_realRouterPassesBurnRecap() (gas: 31410208)
  no pass at the dip:  seniorWallet 100007305935  juniorWallet 45770092953  unassigned 56727526733
  recap 56727526733
  pass pending 44177564060
  pass pending 31627601387
  pass pending 19077638714
  pass pending 6527676041
  pass pending 0
  real Router passes: seniorWallet 19077447811  juniorWallet 183441438190  unassigned 0  pendingRecap 0
```

