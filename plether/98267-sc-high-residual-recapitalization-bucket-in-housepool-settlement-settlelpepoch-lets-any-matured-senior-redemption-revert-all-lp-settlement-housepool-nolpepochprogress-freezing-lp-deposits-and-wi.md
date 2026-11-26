# \#98267 [SC-High] Residual recapitalization bucket in HousePool settlement (`settleLpEpoch`) lets any matured Senior redemption revert all LP settlement (`HousePool__NoLpEpochProgress`), freezing LP deposits and withdrawals until the owner recapitalizes the full residual

**Submitted on Nov 6th 2026 at 15:03:57 UTC by @thorthumbz00 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98267
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro
After a recapitalization leaves a residual bucket and a Senior redemption has matured, every permissionless `settleLpEpoch` call reverts with `HousePool__NoLpEpochProgress`, and the revert rolls back the reconcile that would have drained the bucket. LP exit and LP entry are both wedged, with no permissionless way out. I reproduced the freeze holding at least 48 hours on the ordinary path and 144 hours on the governance rate-change route, triggered by a Senior redeem request as small as a 2 USDG holder's. The cash is present in the pool (the recapitalization sits in the residual bucket); an accounting latch holds it, so this is not a lack-of-liquidity case. This report is in the same HousePool recapitalization area as reports #97866 and #97873 but is a different mechanism, a wedge that blocks settlement rather than a value transfer, so it is not a duplicate of either.

## Vulnerability Details
The residual recapitalization bucket reserves the pool's free cash. `HousePool.sol:848` sets the Senior redemption budget to `_min(freeAssets, seniorPrincipal)`, and `freeAssets` is built with the pending claimant bucket reserved (`HousePool.sol:842-843`), so a Senior head with `seniorPrincipal > 0` receives a zero budget and stays as backlog. A zero-budget phase makes no progress (`HousePool.sol:988-990`, `if (phase.fundedAssets >= phase.budget) break;`). The outstanding bucket also blocks entries through the gate (`HousePool.sol:1108-1117`, which folds the pending claimant bucket into `projectedUnassignedAssets`), so no deposit can supply progress either. With no Senior, Junior, or deposit epoch processed, `_emitLpEpochSettled` reverts:

```
if (result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0 && depositProcessedEpochs == 0) {
    revert HousePool__NoLpEpochProgress();   // HousePool.sol:1183-1185
}
```

The revert discards the whole transaction, including the reconcile that would have drained the residual bucket. The latch clears only when the owner recapitalizes the full residual: a partial top-up still reverts, and only a recapitalization of the whole residual lets `settleLpEpoch` pass. Cancellation does not help, because a matured request can no longer be cancelled (`TrancheVault__RedeemCancellationUnavailable`). Time does not help (the wedge holds past 48 hours), and a Junior dust request does not help.

The owner recapitalization is a documented trusted action (SECURITY.md:157). The wedge then arises on the ordinary permissionless settlement path that any caller drives (`settleLpEpoch` takes no role), which is in scope because a trusted owner action leaves users unable to exit (rules-and-trust-model.md:33-35).

**Who can trigger it, and what it needs.** Trigger: any user. A Senior holder's matured redeem request plus a residual recapitalization bucket is enough, and a 2 USDG holder suffices. Precondition: a residual bucket after a recapitalization at a zero-equity state, reachable after a large mark round trip; the test reaches it with a whipsaw (a short that wins at a high mark and a long that then wins at a low mark) that drives the pool into degraded mode with trader claims above cash. Capital: negligible. No market move is needed once the state is set. The only exit is the owner recapitalizing the full residual, so the freeze persists until governance acts.

**Recommended fix.** Do not reserve the residual recapitalization bucket out of the Senior redemption budget while a matured Senior head exists, or let a settlement that drains the bucket count as progress, so `_emitLpEpochSettled` (`HousePool.sol:1183-1185`) does not revert and roll back the drain.

**Release configuration.** The three tests above run on the repository's `HousePoolTestBase` harness defaults. I re-ran the freeze at the release parameters of `packages/perps/DEPLOYMENT.md` (vpiFactor 0.01e18, maxSkewRatio 0.4e18, maintMarginBps 10, initMarginBps 20, baseCarryBps 500) in a second file, `F06Release.t.sol`, whose `_riskParams` override installs those values and whose first test asserts them. The release harness builds the deficit through skew-capped staged opens and real closes (deficit 19,911.85 USDG, owner recapitalization of 1.25x the deficit = 24,889.81 USDG), and every settlement attempt is the real `router.settleLpEpoch` from a roleless address. Result: at the Senior request's maturity, and again at maturity + 24 h and + 48 h, `router.settleLpEpoch` reverts `HousePool__NoLpEpochProgress` (log lines `at maturity router.settleLpEpoch reverted NoLpEpochProgress`, `... + 24h ...`, `... + 48h ...`, `hours frozen shown 48`); a junior deposit request is refused meanwhile; a half-residual owner top-up still reverts (`half residual settlement REVERTED`, claimable 0); the minimal owner recapitalization that clears it is the residual less twice the Senior principal plus 1 USDG (`minimal recap 9956925639`, `residual - 2P + 1 USDG settlement PASSED`, claimable 6,691.29 USDG). The queued-before-recap variant and a negative control (a recapitalization of 2.5x the deficit drains the bucket enough and settlement passes) run in the same file. All four release tests pass (`Suite result: ok. 4 passed; 0 failed`).

## Impact Details
Temporary freezing of all LP funds: Senior and Junior deposits and withdrawals in HousePool and both TrancheVault instances. Because `settleLpEpoch` is the shared LP settlement entry, when it reverts no Senior or Junior redemption settles and no new deposit is admitted (`canAcceptTrancheDeposits` and `canSettleDepositEntries` both read false). The matured Senior redeemer (its full balance in the main test, a 2 USDG holder in the queued-request variant) cannot be funded, and every other LP is frozen behind the same reverting call. The freeze holds at least 48 hours on the ordinary path and 144 hours on the governance rate-change route before any settlement lands, well past the 24-hour threshold. No funds are lost, and a Senior holder cannot cancel after maturity.

On the lack-of-liquidity exclusion: the pool holds the cash. The recapitalization is present as the residual bucket (39,988.86 USDG in the test, with Senior principal at 9,997.22 USDG); an accounting reservation, not missing liquidity, blocks settlement.

### Before / after state delta (test_H24b, USDG, 6 decimals)
| Account / state | During the wedge | After the owner routes the full residual |
|---|---|---|
| `getFreeAssets()` | 0 (the bucket reserves all free cash) | greater than 0 |
| Pending recapitalization bucket | 39,988.86 (residual) | drained |
| Senior principal, matured redeem | 9,997.22, redeem frozen (claimable 0 after 48 hours) | redeem funded (claimable greater than 0) |
| `settleLpEpoch` | reverts `HousePool__NoLpEpochProgress` (still at 48 hours) | succeeds |

## References
- `packages/perps/src/HousePool.sol:842-848` (budget reserves the bucket), `:988-990` (zero-budget phase breaks), `:1108-1117` (entries blocked), `:1183-1185` (the no-progress revert rolls back the reconcile) at a82d4fef.
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:72-82` (residual bucket).
- SECURITY.md:157 (owner recapitalization is a documented action). In scope under rules-and-trust-model.md:33-35.
- Prior audits: Cecuro MED-24 is a bounded-work, zero-payout refund-epoch delay, not a hard revert without a permissionless exit. Not a duplicate, and not in known-issues.


## Link to Proof of Concept

https://gist.github.com/tdev00/179049f9785c7bd29ce0654cbb6fb438

## Proof of Concept

## Proof of Concept

**Attack, step by step (as the tests execute it on the `HousePoolTestBase` harness).**
1. Two LPs seed HousePool through the tranche vaults: a Senior LP deposits about 100,000 USDG and a Junior LP deposits about 100,000 USDG, and 24 small Junior holders each deposit 2 USDG (the dust redeemers that drive a normal settlement pass).
2. A trader book is opened on both sides and a whipsaw drives the pool into degraded mode (`_wipeByWhipsaw`): a short position wins at a high mark and a long position then wins at a low mark, through real opens and closes, leaving trader claims above pool cash by the deficit. The test asserts claims exceed cash and logs `deficit 39988862067` (39,988.86 USDG) and `degraded true`. Mark updates inside the helper are applied with `vm.prank(address(router))` (the router's authority, as a keeper's mark update is in production), and `vm.warp` / `vm.roll` advance time.
3. The owner recapitalizes (`engine.recapitalizePool`, the documented action, sized to the deficit plus a 25% cushion) and clears degraded mode (`clearDegradedMode`, owner-only). These are the trusted owner actions, not the bug. One normal settlement pass and the long winner's `settleTraderClaim` then leave a residual recapitalization bucket of 39,988.86 USDG (`residual 39988862067`), and `getFreeAssets()` is 0: the bucket reserves all free cash. Senior principal stands at 9,997.22 USDG (`S 9997215516`).
4. The Senior LP requests redemption of its full balance, and the test advances to the request's epoch start. A call to `pool.settleLpEpoch(0, 0)` by an ordinary address, with no role and no prank, reverts with `HousePool__NoLpEpochProgress`. The Senior head has principal above zero but a zero redemption budget (the bucket reserves the free cash), Junior is skipped because Senior has backlog, and deposit entries are blocked, so no epoch is processed and the settlement reverts, rolling back the reconcile that would have drained the bucket.
5. Nothing permissionless clears it. A Junior dust redeem request is queued and the test warps 48 hours forward; `settleLpEpoch` still reverts, and the Senior redemption is still unfunded (claimable 0) while the Senior head still has value. The Senior holder cannot cancel: `cancelRedeemRequest` reverts `TrancheVault__RedeemCancellationUnavailable`, because the request has matured.
6. Only the owner routes it out. A partial top-up of half the residual still reverts; a recapitalization of the whole residual lets `settleLpEpoch` succeed and makes the Senior redemption claimable (claimable greater than 0).
7. Two variants confirm the reach. `test_H24c` queues a 2 USDG Senior holder's full-balance redeem request before the recapitalization, as LPs do during a crisis; after the recapitalization and a warp of 24 hours, `settleLpEpoch` reverts `HousePool__NoLpEpochProgress`, the holder's request is unfunded, and the whole recapitalization bucket is untouched (`pendingRecapitalizationAssets` still equals the full recap amount), so a 2 USDG holder's request alone wedges the first pass. `test_C3` shows the scope: with the Senior redemption wedged, `canAcceptTrancheDeposits(false)` and `canSettleDepositEntries()` both read false (no new entrant and no deposit can supply progress), while a trader still opens and closes a position normally (`traderSize` 0 after close, traders unaffected). The only route that lands is a governance rate-change: each `finalizePoolConfig` runs a reconcile outside the progress check and partially drains the bucket, and after 2 rounds of 72 hours each `settleLpEpoch` finally succeeds. The log records `rounds 2`, `hours frozen 144`, and the bucket stepping down (`pending after finalize 29991646550` then `19994431033`). The freeze lasted 144 hours.

**Runnable tests.** Three Foundry tests in TriageLpRecap.t.sol plus four release-configuration tests in F06Release.t.sol, all run through the `BasePerpTest` harness. Only Pyth and USDG are mocked (the harness `MockPyth` and the mock settlement token), which the rules permit. The full test files (TriageLpRecap.t.sol, F06Release.t.sol) and the captured forge logs (TriageLpRecap.log, forge-F06Release.log) are in the secret Gist linked below.

Steps:
1. Clone `github.com/immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (submodules included) and install solc 0.8.35 (the repository's pinned compiler).
2. Create `poc/` at the repository root and copy TriageLpRecap.t.sol into it. The root `foundry.toml` remaps `@plether/perps/<File>.sol` to `packages/perps/src/<File>.sol`; the file imports the harness as `../packages/perps/test/perps/BasePerpTest.sol`.
3. Run, from the repository root, with the test directory pointed at `poc/`:
   ```
   export FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]'
   forge test --match-path poc/TriageLpRecap.t.sol --match-test test_H24b_residualRecapBucketWedgesAllLpSettlement -vv
   forge test --match-path poc/TriageLpRecap.t.sol --match-test test_H24c_seniorRequestQueuedBeforeRecapWedgesFirstPass -vv
   forge test --match-path poc/TriageLpRecap.t.sol --match-test test_C3_wedgeRoutesAndOtherParties -vv
   ```
4. The release-configuration file runs the same way (copy F06Release.t.sol into `poc/`):
   ```
   forge test --match-path poc/F06Release.t.sol -vv
   ```
   Expected: `Suite result: ok. 4 passed; 0 failed` with `test_F06_A_residualFreezesRouterSettlementAtRelease` logging `at maturity + 48h router.settleLpEpoch reverted NoLpEpochProgress` and `hours frozen shown 48`.

Headline test (ordinary path), `contract O12WaterfallTest`. `settleLpEpoch(0, 0)` is called by an ordinary address with no role and no prank:

```solidity
function test_H24b_residualRecapBucketWedgesAllLpSettlement() public {
    uint256 deficit = _wipeByWhipsaw(40_000e6);
    uint256 cushion = deficit / 4;
    _recap(deficit + cushion);
    _pass();
    vm.prank(longWinner);
    engine.settleTraderClaim(longWinner);
    assertGt(pool.pendingRecapitalizationAssets(), 0, "residual bucket");
    assertEq(pool.getFreeAssets(), 0, "bucket reserves all free cash");

    _finishAsyncCooldown(seniorVault, seniorLp);
    uint256 shares = seniorVault.balanceOf(seniorLp);
    uint256 requestId = _requestAsyncRedeem(seniorVault, seniorLp, shares);
    vm.warp(pool.lpEpochStart(requestId));
    vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
    pool.settleLpEpoch(0, 0);

    // A Junior dust request does not help and time does not help.
    uint256 juniorRequest = _queueDust();
    vm.warp(pool.lpEpochStart(juniorRequest) + 48 hours);
    vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
    pool.settleLpEpoch(0, 0);
    uint256 seniorLpClaimable = seniorVault.claimableRedeemRequest(requestId, seniorLp);
    uint256 seniorLpValue = pool.seniorPrincipal();
    assertEq(seniorLpClaimable, 0, "Senior LP exit not funded after 48 hours");
    assertLt(seniorLpClaimable, shares, "Senior LP frozen");
    assertGt(seniorLpValue, 0, "the frozen head has value, so it is not refunded");

    // Cancellation is pre-maturity only.
    vm.prank(seniorLp);
    vm.expectRevert(bytes4(keccak256("TrancheVault__RedeemCancellationUnavailable()")));
    seniorVault.cancelRedeemRequest(requestId, seniorLp);

    // Route out: more owner cash, enough that distributable passes the residual bucket.
    uint256 residual = pool.pendingRecapitalizationAssets();
    uint256 tooLittle = residual / 2;
    _recap(tooLittle);
    vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
    pool.settleLpEpoch(0, 0);
    _recap(residual);
    pool.settleLpEpoch(0, 0);   // only now does it pass
    assertGt(seniorVault.claimableRedeemRequest(requestId, seniorLp), 0, "owner cash is the route out");
}
```

Queued-request variant ("a 2 USDG holder"), `contract O12WaterfallTest`. The request is queued before the recapitalization and the first pass still reverts 24 hours later:

```solidity
function test_H24c_seniorRequestQueuedBeforeRecapWedgesFirstPass() public {
    address smallSenior = carol;
    _fundSenior(smallSenior, 2e6);
    uint256 deficit = _wipeByWhipsaw(40_000e6);
    _finishAsyncCooldown(seniorVault, smallSenior);
    uint256 dustShares = seniorVault.balanceOf(smallSenior);
    uint256 requestId = _requestAsyncRedeem(seniorVault, smallSenior, dustShares);
    _recap(deficit + deficit / 4);
    vm.warp(pool.lpEpochStart(requestId) + 24 hours);
    vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
    pool.settleLpEpoch(0, 0);
    uint256 seniorLpClaimable = seniorVault.claimableRedeemRequest(requestId, smallSenior);
    assertLt(seniorLpClaimable, dustShares, "a 2 USDG Senior holder's request blocks all LP settlement");
    assertEq(pool.pendingRecapitalizationAssets(), deficit + deficit / 4, "nothing settled");
}
```

Scope and governance-route variant, `contract O12WaterfallTest`. Entries are refused, traders are unaffected, and the only route that lands takes 144 hours:

```solidity
    vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
    pool.settleLpEpoch(0, 0);

    assertFalse(pool.canAcceptTrancheDeposits(false), "new entrant refused while the bucket is outstanding");
    assertFalse(pool.canSettleDepositEntries(), "no deposit can supply progress");
    // ... a trader opens and closes; assertEq(traderSize, 0, "Trader unaffected");

    // Route: governance rate-change finalize runs a reconcile outside the progress check, 72 hours each.
    uint256 start = vm.getBlockTimestamp();
    bool landed;
    while (!landed && rounds < 12) {
        // propose a new seniorRateBps, warp to activation, finalize, then try to settle
        pool.finalizePoolConfig();
        try pool.settleLpEpoch(0, 0) returns (IHousePool.LpEpochSettlementResult memory) {
            landed = true;
        } catch {}
    }
    assertTrue(landed, "the governance route eventually lands");
    assertGt(vm.getBlockTimestamp() - start, 24 hours, "freeze lasted over 24 hours");
```

Captured output (TriageLpRecap.log in the Gist):

```
[PASS] test_H24b_residualRecapBucketWedgesAllLpSettlement() (gas: 7128749)
  deficit 39988862067
  degraded true
  residual 39988862067
  S 9997215516
  senior claimable after route out 57713763889814

[PASS] test_H24c_seniorRequestQueuedBeforeRecapWedgesFirstPass() (gas: 5685080)
  deficit 39988862667
  degraded true

[PASS] test_C3_wedgeRoutesAndOtherParties() (gas: 8956091)
  deficit 39988862067
  degraded true
  pending after finalize 29991646550
  pending after finalize 19994431033
  rounds 2
  hours frozen 144
  claimable shares 20205
  S 9997215516
  H 19994431032
```

