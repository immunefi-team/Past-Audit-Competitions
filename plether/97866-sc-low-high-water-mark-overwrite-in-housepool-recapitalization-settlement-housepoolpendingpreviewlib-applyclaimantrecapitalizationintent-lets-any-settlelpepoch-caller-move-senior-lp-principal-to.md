# \#97866 [SC-Low] High-water-mark overwrite in HousePool recapitalization settlement (`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`) lets any `settleLpEpoch` caller move Senior LP principal to Junior LPs and to new unprivileged entrants.

**Submitted on Nov 3rd 2026 at 17:03:02 UTC by @thorthumbz00 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97866
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
After the owner recapitalizes the pool, a single permissionless `settleLpEpoch` call can discard the stored Senior high-water mark and replace it with the small recapitalization amount. The Senior tranche then loses the restoration it is owed, and that value is paid to Junior LPs and to anyone who deposits into the Junior tranche after the cut. I reproduced a Senior LP collecting 5,289.62 USDG where the untouched twin pays 100,007.31 USDG, a 94.7% loss, and a fresh unprivileged entrant paying 20,000 USDG and collecting 66,723.79 USDG of Senior's recovery. The caller needs no role.

## Vulnerability Details
The failure is in `HousePoolPendingPreviewLib.sol:118-122` (reached from HousePool.sol). On the branch where both principals are zero but Senior shares are still outstanding, `applyClaimantRecapitalizationIntent` sets

```
state.waterfall.seniorPrincipal += remaining;
state.waterfall.seniorHighWaterMark =
    recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
```

It writes `max(recapTarget, amount)` into `seniorHighWaterMark`, throwing away the stored mark that records what Senior paid in. That branch is reachable on the normal settlement path because `HousePoolReconcilePlanLib.sol:68-71` subtracts the pending recapitalization bucket from `distributable` before any principal is restored, so at an unrecovered dip the reconcile sees `seniorPrincipal == 0 && juniorPrincipal == 0` while shares still exist. The owner's recapitalization (`CfdEngine.recapitalizePool`, SECURITY.md:157) is a documented trusted action; the bug fires when a stranger then calls `router.settleLpEpoch` at the dip (HousePool.sol:1598, 1769-1771).

The NatSpec at `HousePoolPendingPreviewLib.sol:104-106` calls this an "empty bootstrap," but Senior shares are outstanding here, so it is not an empty pool. The write contradicts the documented invariant "High-water-mark protection: Senior impairment must be restored before junior extracts surplus" (SECURITY.md:394) and the detailed intent that the high-water term "prevents an impaired principal balance from falsely reopening capacity while its restoration entitlement remains" and that recapitalization "cannot mint new senior LP shares, and excess value continues through the canonical claimant/unassigned routing" (SECURITY.md:1197-1198, 1210-1214).

Because the cut lowers the mark, the entry gate `HousePoolTrancheGateLib.sol:36` (`projectedSeniorPrincipal < projectedSeniorHighWaterMark` returns false) reopens, and `TrancheVault.sol:1408-1409` prices the new entrant at the collapsed Junior NAV. So the beneficiary is not limited to an incumbent Junior LP: any attacker can deposit at dust NAV and take Senior's recovery (test `test_H49d_entrantOnReleaseParameters`).

**Who can trigger it, and what it needs.** Trigger: any EOA. The incumbent leg is any caller of `router.settleLpEpoch` (keepers are permissionless, rules-and-trust-model.md:39-40); the entrant leg is any caller that deposits into the Junior tranche. Precondition: a documented owner recapitalization at a dip where claimed equity is zero while Senior shares remain, which `HousePoolReconcilePlanLib.sol:68-71` makes a normal reconcile outcome. Capital: the entrant leg needs only its deposit (20,000 USDG shown, profitable); the incumbent leg needs no capital. No user interaction. Repeatable each recapitalization cycle. A pause stops new settlement but does not restore the overwritten mark.

**Recommended fix.** On the zero-principal branch in `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`, keep `seniorHighWaterMark` at `max(storedMark, recapTarget)` instead of overwriting it with the recapitalization amount, so restoration is bounded below by what Senior paid in.

## Impact Details
Direct theft of Senior LP principal (at-rest USDG held by HousePool), moved to Junior LPs and to new entrants, permanent once exit completes.
- Incumbent leg: Senior LP collects 5,289.62 USDG against 100,007.31 USDG in the twin (94.7% of principal lost); Junior LP rises from 62,239.82 to 184,535.19 USDG.
- Entrant leg: a stranger pays 20,000 USDG and collects 66,723.79 USDG; Senior holds 80,001.39 USDG against 100,068.49 USDG owed.
Blast radius scales to the whole Senior tranche (cap 40M USDG) whenever a recapitalization lands at a zero-equity dip.

### Before / after state delta (incumbent leg, USDG, 6 decimals)
| Account / state | Before (twin, no pass) | After (stranger pass) | Delta |
|---|---|---|---|
| Senior high-water mark | 100,001.00 | 5,289.67 | -94,711.33 |
| Senior LP collected | 100,007.31 | 5,289.62 | -94,717.69 |
| Junior LP collected | 62,239.82 | 184,535.19 | +122,295.37 |

Entrant leg: entrant -20,000.00 paid, +66,723.79 collected (net +46,723.79); Senior held 80,001.39 vs 100,068.49 owed (-20,067.10).

## References
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:104-122` (overwrite) at a82d4fef.
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:68-71` (bucket reserved ahead of principal).
- `packages/perps/src/libraries/HousePoolTrancheGateLib.sol:36` (entry gate reopens); `packages/perps/src/TrancheVault.sol:1408-1409` (dust-NAV entrant pricing).
- `packages/perps/src/HousePool.sol:1598,1769-1771`.
- SECURITY.md:157 (owner recapitalization is a documented action), :394, :1197-1198, :1210-1214 (the invariant this defeats). In scope under rules-and-trust-model.md:33-35 (a documented trusted action that triggers a bug harming users).
- Prior audits: Cecuro MED-12 / LOW-22 describe other recapitalization behaviour, not this high-water-mark rewrite at the zero-equity branch; the branch predates that audit. Not in known-issues.


## Link to Proof of Concept

https://gist.github.com/tdev00/f4fbf55629f7579a176d0a75de87eaa6

## Proof of Concept

## Proof of Concept

**Attack, step by step (as the tests execute it, release parameters from DEPLOYMENT.md).**
1. Senior and Junior LPs deposit into HousePool through the tranche vaults (Senior about 100,000 USDG, Junior about 62,000 USDG in the test). The pool opens with a Senior high-water mark of about 100,001 USDG.
2. A book of trader positions is opened on both sides and the mark moves against the pool (price 2e8 in the test), so claimed equity reaches zero while Senior shares are still outstanding: an in-bounds loss, Senior impaired and owed restoration.
3. The owner performs the documented recapitalization (`CfdEngine.recapitalizePool`, about 5,290 USDG in the test, sized to maximum liability plus a margin) and clears degraded mode (`clearDegradedMode`, owner-only). Both are the trusted owner actions SECURITY.md describes; they are not the bug.
4. The Junior LP requests a redemption and the losing Long is liquidated at the dip (in the test the liquidation call carries the router's authority, as any keeper's call through the router does in production). A stranger with no role, `address(0xABC)`, then calls `router.settleLpEpoch()` at this dip. The reconcile reserves the recapitalization bucket ahead of principal (HousePoolReconcilePlanLib.sol:68-71), sees both principals at zero, and takes the "empty bootstrap" branch in HousePoolPendingPreviewLib.sol:118-122, which overwrites the stored Senior high-water mark (about 100,001 USDG) with the recapitalization amount (about 5,290 USDG).
5. The market recovers and the LPs redeem. Senior's restoration entitlement is gone: the Senior LP collects 5,289.62 USDG where the untouched twin run collects 100,007.31 USDG; the Junior LP collects 184,535.19 instead of 62,239.82.
6. Entrant leg (second test): with the mark cut, the Junior entry gate (HousePoolTrancheGateLib.sol:36) reopens and TrancheVault prices entry at the collapsed Junior NAV. An unprivileged `address(0xE1)` deposits 20,000 USDG and collects 66,723.79 USDG on redemption, while Senior holds 80,001.39 against 100,068.49 owed.

**Runnable tests.** Two Foundry tests that run through the `BasePerpTest` harness. Only Pyth and USDG are mocked (the harness `MockPyth` and the mock settlement token), which the rules permit. Full test files (`TriageLpRecap.t.sol`, `TriageLpEntry.t.sol`) and the captured forge logs are in the secret Gist linked below.

Steps:
1. Clone `github.com/immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (submodules included) and install solc 0.8.35 (the repository's pinned compiler).
2. Create `poc/` at the repository root and copy `TriageLpRecap.t.sol` and `TriageLpEntry.t.sol` into it. The root `foundry.toml` remaps `@plether/perps/<File>.sol` to `packages/perps/src/<File>.sol`; the files import the harness as `../packages/perps/test/perps/BasePerpTest.sol`.
3. Run, from the repository root, with the test directory pointed at `poc/` (the repository's own test and script trees are skipped so only the attached files compile as tests):
   ```
   export FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]'
   forge test --match-path poc/TriageLpRecap.t.sol --match-test test_Release_realRouterPassMovesSeniorToJunior -vv
   forge test --match-path poc/TriageLpEntry.t.sol --match-test test_H49d_entrantOnReleaseParameters -vv
   ```
   The first compile takes a few minutes (via_ir). Both tests print `[PASS]` with the figures quoted below.

Headline test (incumbent leg), `contract O14ReleaseParamsTest`, release parameters from DEPLOYMENT.md (maintenance margin 0.1%, initial margin 0.2%, Senior cap 40M USDG and 80%). A stranger `address(0xABC)` calls `router.settleLpEpoch` at the dip:

```solidity
function test_Release_realRouterPassMovesSeniorToJunior() public {
    uint256 snap = vm.snapshotState();
    (Out memory twin,, uint256 hwmBefore,,) = _run(false);   // no pass at the dip
    assertTrue(vm.revertToState(snap));
    (Out memory run, uint256 recap,, uint256 preview, uint256 stored) = _run(true); // stranger pass at the dip
    assertEq(preview, stored, "preview and live agree on the wrong value");
    assertLt(run.senior, twin.senior, "senior LP collects less than the twin");
    assertGt(run.junior, twin.junior, "junior LP collects what senior lost");
}
// inside _run(true): assertLt(pool.seniorHighWaterMark(), hwmBefore / 5, "mark overwritten");
```

Entrant leg ("any attacker"), `contract A14EntrantRelease`, same release parameters. The entry gate is shut before the cut and open after it; an unprivileged `address(0xE1)` deposits and collects:

```solidity
// peer (mark uncut): assertEq(juniorVault.maxRequestDeposit(entrant), 0, "entry shut at this mark with the mark uncut");
// after the cut:      assertGt(juniorVault.maxRequestDeposit(entrant), 0, "entry open: the cut removed the impairment gate");
assertLt(seniorHeld, seniorOwed, "senior LP loses to the entrant's path");
assertGt(entrantUsd, paid, "an unprivileged entrant collects more than it paid");
```

Captured output (`TriageLpRecap.log`, `TriageLpEntry.log` in the Gist):

```
[PASS] test_Release_realRouterPassMovesSeniorToJunior() (gas: 25691402)
  twin no dip pass:  seniorWallet 100007306059  juniorWallet 62239823370
  real Router pass at dip: seniorWallet 5289620255  juniorWallet 184535190070
  recap 5289673186   H after real Router pass 5289673186   hwmBefore 100001000675

[PASS] test_H49d_entrantOnReleaseParameters() (gas: 25166927)
  entrant paid: 20000000000   entrant collected: 66723788659
  senior held: 80001391730    senior owed (peer): 100068493150
  junior held: 310336588      junior peer: 0
```

