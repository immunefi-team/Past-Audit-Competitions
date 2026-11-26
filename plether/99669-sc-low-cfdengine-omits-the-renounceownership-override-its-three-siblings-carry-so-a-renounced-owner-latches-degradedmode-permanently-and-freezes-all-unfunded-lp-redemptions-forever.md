# \#99669 [SC-Low] CfdEngine omits the renounceOwnership override its three siblings carry, so a renounced owner latches degradedMode permanently and freezes all unfunded LP redemptions forever

**Submitted on Nov 14th 2026 at 18:42:06 UTC by @drak3p14159 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99669
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Permanent freezing of funds

## Description

## Brief/Intro

The protocol disables ownership renunciation in **exactly three** contracts, and states why in the code:

> `HousePool.sol:264` — *"Ownership cannot be renounced because governance must retain **emergency recovery authority**."*

`CfdEngine` is not one of the three. It inherits `renounceOwnership()` unmodified through `CfdTreasury` (`CfdEngine.sol:46`), **and it is the contract that holds the emergency recovery authority in question**: `degradedMode`, whose only clearing function is `clearDegradedMode()` at `CfdEngine.sol:658`, `external onlyOwner`.

A renounced engine owner therefore produces a state the protocol's own comment says must not be reachable: emergency recovery authority permanently removed, with `degradedMode` latched true and no address able to clear it.

## Vulnerability Details

**The fix list was the surface, and one sibling was missed.**

```
$ grep -rn 'function renounceOwnership' src/
src/HousePool.sol:265:                 function renounceOwnership() public view override onlyOwner {
src/OrderRouterAdmin.sol:125:          function renounceOwnership() public view override onlyOwner {
src/EmergencyPauseCoordinator.sol:102: function renounceOwnership() public view override onlyOwner {
```

Three overrides, each reverting. `CfdEngine` has none.

The chain to permanence is four links, each with a single writer or a single caller:

| link | site | fact |
|---|---|---|
| 1 | `CfdEngine.sol:46` | `CfdEngine` inherits `Ownable2Step` via `CfdTreasury`; `renounceOwnership()` is **public and unoverridden** |
| 2 | `CfdEngine.sol:665` | `degradedMode = false` has **exactly one writer in the whole in-scope tree** |
| 3 | `CfdEngine.sol:658` | that writer sits inside `clearDegradedMode()`, declared `external onlyOwner` |
| 4 | `HousePool.sol:1863-1870` | `_requireWithdrawalsLive` reverts `HousePool__DegradedMode()` **solely** on `statusSnapshot.degradedMode`. Its call sites were **enumerated, not assumed**: `grep -n '_requireWithdrawalsLive' src/HousePool.sol` returns the definition at `:1864` and **one** invocation, `:832`, inside `_settleLpEpoch` (`:824`) |

⇒ Owner renounces ⇒ `clearDegradedMode()` is unreachable by any address ⇒ `degradedMode` is latched ⇒ `_settleLpEpoch` reverts on every call ⇒ **LP epochs never settle again, so queued redemptions are never funded.** The LP exit path is not slowed or degraded; it is removed.

## The fix is in this repository's own history, and its scope is three contracts

**This does not rest on reading a PDF's recommendation. The remediation is a commit in the in-scope repository, reachable from the scope commit, and its own message states what it covered.**

`e711621f8795909faa63f1abe4845550b88c4f1f` — **2026-10-21** — *"fix(perps): harden governance, configuration, and recovery controls (#121)"*

Its message enumerates the changes it squashes. Two of them matter here, and they are in the same commit:

```
* fix(perps): disable emergency ownership renunciation
* fix(perps): add owner-funded pool recapitalization
```

**What that commit actually did, by its own diff:**

| | |
|---|---|
| added `function renounceOwnership() public view override onlyOwner` to | **exactly three** sources — `EmergencyPauseCoordinator.sol`, `HousePool.sol`, `OrderRouterAdmin.sol` |
| also modified | **`packages/perps/src/CfdEngine.sol`, +59 / −11 (70 lines changed)** |
| and in that `CfdEngine.sol` patch, the string `renounceOwnership` appears | **zero times** |
| while the same patch **added** | `function recapitalizePool(…)` — the owner-funded recovery action |

⇒ ⭐⭐ **In the single commit whose stated purpose was hardening *recovery* controls, the project gave `CfdEngine` an owner-only recovery function and disabled ownership renunciation in three other contracts — leaving renounceable the one contract it had just handed a recovery function to.** `CfdEngine` holds `degradedMode` and `clearDegradedMode()`, the only function that clears it. The commit had the file open and edited its owner-gated configuration paths.

⭐ **And the recovery function that commit added to `CfdEngine` is the function this finding turns on.** `recapitalizePool` is what the proof of concept drives at step 2: the owner calls it, raw solvency is restored, and **the latch survives anyway** — which is why the report is not a theoretical gap between two code paths. **The commit added an owner-only recovery action to this contract, left the contract renounceable, and the PoC exercises that exact action and shows the recovery does not reach the latch.**

⚠ **This commit is reachable from the scope commit in the repository listed in scope**, so it is citable without reference to any other repository, and the SHA resolves there. (It arrived as PR #121 upstream; the SHA is the stable reference.)

## Eligibility: this is the incomplete-fix case the programme's own clause describes

The root cause was reported previously, and the programme's exclusion rule carries an explicit exception for exactly this situation:

> **Scope page, Known Issues:** *"For findings marked as fixed or Acknowledged in the reports above, a report on the **same root cause** is valid **only if it shows that the fix at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` is incomplete or introduces a new bug**."*

Prior finding **LOW-59** ("Owner can leave emergency recovery ownerless") is marked **Acknowledged**, and its remediation comment corroborates the commit above — naming the same three contracts:

> *"**LOW-59 — Addressed. HousePool, OrderRouterAdmin, and EmergencyPauseCoordinator override `renounceOwnership()` to revert**, preventing governance from permanently removing their recovery and guardian-management authority through renunciation."*

**Those three contracts are precisely the three that carry the override today.** `CfdEngine` — which holds `degradedMode` and the only function that clears it — was not on the list. **This report shows the fix at the scope commit is incomplete**, which is the clause's literal condition for validity.

⚠ **The remediation note's own carve-out, read to the end and engaged rather than quoted around:** *"The change prevents explicit renunciation; it does not prevent loss or compromise of the owner account or acceptance by an unsuitable successor."* That carve-out disclaims **other failure modes** — key loss, compromise, a bad successor. It does not narrow which contracts the fix covers, and this report concerns **explicit renunciation**, which is the one thing the note says the change *does* prevent. The carve-out therefore does not reach this finding.

## The sponsor's own tests enumerate the surface it meant to protect, and CfdEngine is absent from it

This is what makes the incomplete-fix argument **evidential rather than rhetorical.** The three overrides are not an incidental pattern — each is held in place by a *named regression test*:

| contract | override | named regression test |
|---|---|---|
| `HousePool` | `:265` | `test_HousePool_RenounceOwnershipRevertsAndPreservesTransfer()` — `test/perps/spec/admin/TimelockPause.t.sol:459` |
| `OrderRouterAdmin` | `:125` | `test_OrderRouterAdmin_RenounceOwnershipRevertsAndPreservesTransfer()` — `TimelockPause.t.sol:632` |
| `EmergencyPauseCoordinator` | `:102` | `test_RenounceOwnershipRevertsAndPreservesTransfer()` — `test/perps/spec/admin/EmergencyPauseCoordinator.t.sol:84` |
| **`CfdEngine`** | **none** | **none — no `renounceOwnership` test for the engine exists in the suite** |

⭐ **And the engine's renounceability is not an oversight — the sponsor asserts it deliberately, in one context only.** `test/perps/spec/admin/TreasuryProposal.t.sol:340`, `test_TreasuryProposal_ActivationDoesNotRequireAnOwner()`:

```solidity
engine.proposeProtocolTreasury(TREASURY_B, 30 days);
engine.renounceOwnership();                 // no expectRevert -- it succeeds
assertEq(engine.owner(), address(0));       // the ownerless engine is the asserted state
vm.warp(expiry);
assertEq(engine.protocolTreasury(), TREASURY_B);
```

**The sponsor renders the engine ownerless on purpose and verifies exactly one consequence: that the treasury timelock still activates.** `SECURITY.md:128` says the same thing in prose — *"preserves the schedule; renouncing ownership does not prevent activation."*

⇒ **So the defect is not "renunciation was forgotten." It is that renunciation was intended to be survivable, was verified survivable for the treasury schedule, and was never checked against `degradedMode` — the one piece of state whose only exit is owner-gated.** Four test files touch `renounceOwnership` and **none of them relates it to degraded mode.** The protection the sponsor built is real, it is tested, and it stops one contract short of the latch that matters.

⚠ **And the test names are a weaker map of the coverage than they appear, which is why the absent `CfdEngine` regression is worth weighing rather than explaining away.** A second instance sits on the adjacent surface: `test/perps/accounting/waterfall/TerminalNavIntegrationSecurity.t.sol` carries the distinct-keeper liquidation security regression, and at `:41` its risk parameters set **`baseCarryBps: 0`** — so the carry-conditional inner brackets that the liquidation path enters only when carry is due are **structurally unreachable in that test**, whatever its name implies. **The pattern is therefore not a one-off:** this suite's names describe intent more reliably than they describe execution, so "there is no `CfdEngine` renounce test" is evidence about the protected surface rather than an oversight in our reading of it.

## Reachability

**Reachability — control:** the engine owner calls `renounceOwnership()`, an inherited `public` function taking no arguments and requiring no market state. There is no attacker, no race and no privilege escalation. The owner may do this by mistake, under compromise, or in the reasonable belief that the contract behaves like its three siblings — which revert.

**Reachability — contingency:** the payoff requires one event the caller does not control and one it does. The uncontrolled event is that the engine is **in `degradedMode` at or after the moment of renunciation** — an ordinary protective transition, and precisely the state `clearDegradedMode()` exists to exit. The controlled event is the renunciation itself. **Order does not matter and neither leg needs to be arranged:** renouncing while already degraded latches immediately, and renouncing while healthy arms the deployment so that the *next* degraded transition — whenever it occurs, for any reason — is terminal. There is no third party to persuade and no approval to obtain.

**Contingency basis:** `CfdEngine.sol:658` — `clearDegradedMode()` is declared `external onlyOwner`, so the sole clearing path for `degradedMode` is gated to the owner. That gate is the contingency: it is what converts a renounced owner into a permanent latch rather than a temporary one.

**Reachability basis:** `exec:` (executed against the code, not reasoned about) — measured, not modelled. `DegradedRenounceFrozen.t.sol::test_RenouncedOwnerMakesRecoveredDegradedSettlementPermanent`, run in the repository's Foundry test suite at scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with **no in-scope contract modified**. **MEASURED: `[PASS]`, 1 passed / 0 failed, gas 2,603,438.**

The test asserts the whole chain in order, so no link is assumed: degraded mode latches from an ordinary close; recapitalization restores raw solvency and **does not** clear the latch; the owner **can** clear it immediately before renouncing — proved by `vm.snapshotState()` → `clearDegradedMode()` → `assertFalse` → `vm.revertToState()`, so the alternative world is demonstrated reachable rather than assumed; `renounceOwnership()` succeeds and `owner()` becomes `address(0)`; the latch survives both the renunciation and the warp; and permissionless frozen-path settlement then reverts `HousePool__DegradedMode`, with the redemption **still pending** and `claimableRedeemAssets == 0`.

⚠ **Disclosed rather than left to be found — what the PoC stages, and how.**
- **A frozen market is staged by warping to `1_709_985_600`.** That is the sponsor's own proven instant: `test/perps/accounting/close/EngineCloseSettlement.t.sol:59-60` warps to exactly that value and asserts `isOracleFrozen()`. A frozen market is a **documented, non-privileged, recurring** protocol condition (Friday 17:00 to Sunday 17:00 New York), so this is a real protocol state, not a fabricated one. An `assertLt` guards the warp direction, and `isOracleFrozen()` is re-asserted immediately before the settlement call so that a window crossing would fail on the operand rather than masquerade as the guard under test.
- **Cheatcodes used: `vm.warp`, `vm.prank` (for the router-gated mark update), and `vm.snapshotState`/`vm.revertToState` (for the counterfactual).** **No `mockCall`, no `etch`, no `store`, no `deal`** — nothing fabricates protocol state, which is the distinction between a cheatcode-constructed state and a reachable one.
- **The degraded state is staged by the sponsor's own fixture for this exact state, copied unchanged.** `test/perps/accounting/close/DegradedSettlementBehavior.t.sol:33-45`, `test_ProfitableCloseLatchesDegradedModeWhenBackingFallsBelowLiability()`, performs the identical sequence with the identical figures — `_fundTrader(longTrader, 100_000e6)`, `_fundTrader(shortTrader, 100_000e6)`, `_open(short, SHORT, 999_000e18, 50_000e6, 1e8)`, `_open(long, LONG, 500_000e18, 50_000e6, 1e8)`, `_close(long, LONG, 500_000e18, 20_000_000)` — and asserts the outcome in English: *"Profitable close should latch degraded mode when it reveals insolvency."* It cites `ACCOUNTING_SPEC.md#degraded-mode` as its source of truth. ⇒ **So the latched state is sponsor-asserted, not harness-asserted, and no objection can be raised against this staging without being raised against the project's own test that establishes the behaviour.**
- **On the pranking rule specifically:** those helpers prank the router, and this reviewer's standing rule is that a PoC for *"a guard is missing on path P"* must enter at **P's own public entry point**. **Path P here is LP epoch settlement, and its public entry point is `pool.settleLpEpoch(...)` — which the demonstration calls directly and unpranked** on the permissionless frozen path. The helpers touch only the order-execution path, which this finding does not contest. The rule bites when the pranked call *is* the demonstration; here it is the setup, and the setup is the sponsor's.

⚠ **Two limbs left UNRESOLVED, stated as unresolved rather than explained away.** An earlier variant of this PoC attempted the live-mark path and failed with `HousePool__MarkPriceStale()` instead of `HousePool__DegradedMode()`. Diagnosis: `HousePool.sol:809` guards the staleness check with `hasOpenPositions && !oracleFrozen`, and because `expectedPublishTime` must equal `statusSnapshot.lastMarkTime` or `HousePool__Unauthorized` fires first, the check at `:817` reduces to *"the mark must sit in the current hour bucket"*. Against that: **(i)** a diagnostic measured `engine.lastMarkTime()` two epoch buckets behind `block.timestamp` (474871 vs 474873) after an `updateMarkPrice(mark, uint64(block.timestamp))` call that did **not** revert, and we cannot explain why that call failed to move it; **(ii)** a warp-forward-in-hourly-steps search did not reach a frozen window in 216 steps, which we also cannot explain, and the sponsor's absolute instant was used instead. **Both are moot for this finding, because `:809` is a conjunction and the frozen route falsifies the other operand.** The honest answer to *"why does the live-mark path fail?"* is **"the harness could not stage a current-bucket mark with an open position, cause unresolved"** — not an explanation we possess.

## Why the privileged trigger does not remove this from scope

Argued, not conceded.

1. **The programme's scope page admits it in terms:** *"a report is out of scope where the attack vector itself requires the use of a privileged address… Exceptions apply … **where an in-scope impact arises from a logic error that a privileged address can trigger, which remains valid**."* The logic error here is the **missing override** — an omission, not an exercise of authority. The owner is not using a power to cause harm; the owner is using an ordinary inherited function that three sibling contracts deliberately disable and this one does not.
2. **And the same page adds:** *"A report is not out of scope only because a privileged address is affected by the exploit."*
3. **The sponsor does not treat renunciation as outside normal procedure** — it wrote code in three contracts specifically to prevent it, and gave the reason. An action a project defends against in code is a foreseeable action, not an exotic one.
4. **Renunciation is not among the owner's enumerated powers.** The scope page lists the owner's delayed and immediate controls; renouncing ownership appears in neither. So this is not a documented capability being exercised as designed — it is an undocumented side effect of an un-overridden inherited function.

⇒ **The weak limb is likelihood, and it belongs in the tier, not in the validity decision.** The programme states: *"Impact is calculated first, separately from feasibility. Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it."*

## Testing venue — no fork block is pinned, and why that is correct here

This finding pins no fork block, because it does not fork. The programme permits two venues: a PoC must run **either** in the repository's Foundry test suite at the commit in scope, **or** on a local Anvil fork of the Arbitrum Sepolia deployment. **The repository suite at the frozen commit is therefore a first-class venue rather than a fallback**, and it is the venue used here. If forking mainnet state is preferred, the project's existing test suite is the acceptable alternative, and it is the one in use.

It is also the correct venue on the merits. The defect is a **missing function override** plus an owner-gated state latch; it needs a constructed governance state — an engine in `degradedMode` whose owner has renounced — rather than any live chain state. No deployed state would exhibit it until an owner renounced, which is exactly the event under test. Nothing is executed against production or any public testnet, and the only mocks are the harness settlement token and Pyth price updates — the two components the programme names explicitly as permitted substitutions, both outside the scope.

## Impact Details

**Impact axes: fund lock · DoS of a user action · DoS of an admin/recovery action.**

The frozen value is all LP capital awaiting a future epoch-funding transition, plus queued deposit activation. Release parameters at the scope commit put the **Senior cap at 40M USDG and 80% of the pool**, so the ceiling on frozen value is the pool itself.

The freeze is **permanent in the deployed code**: `degradedMode = false` has one writer, that writer is `onlyOwner`, and after renunciation there is no owner. There is no timelocked path, no guardian path, and no pauser path to clear it — the Guardian *"cannot unpause, change settings, set prices or move funds"* and the Pauser *"can pause; cannot unpause."* Recovery would require a contract upgrade, and the perps contracts at this commit are not upgradeable.

**Severity footing, cited through the programme page rather than a secondary summary.** The page states at `:173` that *"This program follows Immunefi's standard competition rules… Where this program page defines a specific rule, the program page prevails."* Its reward table lists Critical / High / Medium / Low / Insight purely as portions of the reward pool and **defines no per-tier impact wording**, so it does not displace the standard rules on this point and the standard classification governs: **permanent freezing of funds is Critical; temporary freezing is High.** This freeze has no expiry and no in-code remedy, so it is the permanent case.

⚠ **And the obvious process rebuttal is pre-empted by the page itself.** If triage answers *"the owner should simply not renounce"*, that is a mitigation measure rather than a property of the code, and `:362` addresses it in terms: *"there may also be mitigation measures that projects can take to prevent the impact of the bug, which are not feasible or would require unconventional action and hence, **should not be used as reasons for downgrading a bug's severity**."* Relying on every future owner, successor and operator never to call an inherited public function — one that three sibling contracts disable precisely because it must not be called — is exactly such a measure. **The three siblings are the project's own evidence that the code-level guard, not the procedure, is the intended control.**

## What the latch destroys is a funding route the sponsor's own passing test guarantees

This removes the natural objection that nothing was owed at the moment of the freeze.

`test/perps/spec/lp/AsyncLpEpochSettlement.t.sol:772-779`, `test_PreFrozenQueuedDepositDefersUntilOracleLive()`, settles an LP epoch **while the oracle is frozen** and asserts:

```solidity
vm.warp(SATURDAY_FROZEN);
_refreshMark();
assertTrue(engine.isOracleFrozen(), "setup must settle while oracle-frozen");
IHousePool.LpEpochSettlementResult memory frozenResult = _settleLpEpochForTest();
assertGt(frozenResult.juniorFundedAssets, 0, "frozen mode must still fund the matured exit");
assertEq(frozenResult.juniorDepositAssets, 0, "frozen mode must not activate queued entry capital");
```

⇒ **The sponsor's position is explicit and asymmetric: a frozen oracle must still FUND MATURED EXITS, while deliberately NOT activating queued entries.** Exits are the guaranteed half. **`degradedMode` removes exactly that guarantee**, because `_requireWithdrawalsLive` (`HousePool.sol:1867`) reverts at `:832` — *before* `_requireFreshMark` at `:833` and before any funding occurs — so the settlement that the sponsor asserts "must still fund the matured exit" cannot run at all.

⚠ **And this path is permissionless by the sponsor's own construction, not by a cheat of ours.** `test/perps/BasePerpTest.sol:387-395` branches on the same two operands as `HousePool.sol:809`:

```solidity
if (!terminalSnapshot.hasOpenPositions || engine.isOracleFrozen()) {
    return pool.settleLpEpoch(0, 0);        // unpranked, zero mark arguments
}
vm.prank(address(router));                  // the live-mark path needs the router
```

and its own doc comment at `:386` states the intent: *"No-position and frozen-mode calls **intentionally** exercise the cached-mark liveness path."* ⇒ **On the frozen path `settleLpEpoch` is called with no router prank and no mark arguments, exactly as the project's own harness calls it.** So the demonstration uses a documented route rather than a fabricated one, and the guard that currently blocks our PoC (`:817`'s hour-bucket comparison) is skipped because `:809` is a conjunction and `oracleFrozen` falsifies it.

## Step-by-step

1. The engine enters `degradedMode`. This is an ordinary protective transition, not an attack — it is the state `clearDegradedMode()` exists to exit.
2. The engine owner calls `CfdEngine.renounceOwnership()`. It succeeds, because `CfdEngine` does not override it. The same call against `HousePool`, `OrderRouterAdmin` or `EmergencyPauseCoordinator` reverts.
3. Ownership is now `address(0)`. `clearDegradedMode()` (`:658`, `onlyOwner`) can never execute again, so `degradedMode = false` (`:665`) is unreachable.
4. Any call that reaches `_settleLpEpoch` reverts at `HousePool.sol:1867` with `HousePool__DegradedMode()`.
5. LP epochs stop settling permanently. Queued redemptions are never funded. **LP capital awaiting funding is locked for the life of the deployment.**

## Limitations, stated rather than left for triage

- **Already-funded LP claims and recorded trader claims are unaffected.** The frozen set is LP value awaiting a future epoch-funding transition, plus queued deposit activation. This is a bound on the finding, not a refutation: the protocol's LP exit mechanism is removed, and whatever is mid-flight at that moment never completes.
- **The trigger is the owner's own transaction.** Addressed in full above; it is a tier argument, not a validity one.
- **Transferring ownership to a sound successor is unaffected** — `Ownable2Step` two-step transfer remains available, as LOW-59's note says. The defect is specifically that *renunciation* is permitted where its three siblings forbid it.

**Self-close check:** none of the three closes the report, taken one at a time. *(i)* "Only unfunded LP value is frozen" bounds the amount, not the validity: the protocol's LP exit mechanism is removed outright, and the bound is still the pool, since any LP whose redemption has not yet been funded by an epoch settlement can never be funded. *(ii)* "The owner triggers it" is a likelihood statement, and the programme separates impact from feasibility in terms while its `:177` exception admits in-scope impacts from a logic error a privileged address can trigger — so this limb sets the tier, not the verdict. *(iii)* "Ownership transfer still works" is true and irrelevant to the failure mode: the finding is about the path the sponsor disabled in three sibling contracts, not about the path it kept. **If any one of these did close it, this report would not have been filed.**

## Mitigation

Add the override the three sibling contracts already carry. It is the sponsor's own remediation, applied to the contract its own fix list omitted.

```diff
--- a/packages/perps/src/CfdEngine.sol
+++ b/packages/perps/src/CfdEngine.sol
@@ ADMIN
+    /// @notice Ownership cannot be renounced because governance must retain emergency recovery authority.
+    /// @dev Matches HousePool.sol:264-267, OrderRouterAdmin.sol:124-127 and
+    ///      EmergencyPauseCoordinator.sol:101-104. CfdEngine holds `degradedMode` and the only
+    ///      function that clears it (`clearDegradedMode`, onlyOwner), so renunciation here is the
+    ///      one case that permanently removes the recovery authority LOW-59 was fixed to protect.
+    function renounceOwnership() public view override onlyOwner {
+        revert OwnableInvalidOwner(address(0));
+    }
```

A structurally different alternative, if renunciation must stay available: give `degradedMode` a second clearing path that does not depend on the owner — for example allowing the Guardian to clear it once the solvency condition `clearDegradedMode()` already checks (`CfdEngine.sol:662`) is satisfied. That removes the permanence without constraining governance, and the two fixes are independent: the first closes the trigger, the second removes the single point of failure.


## References

**In scope:** `packages/perps/src/CfdEngine.sol` and `packages/perps/src/HousePool.sol` at the frozen scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, deployed on Arbitrum One with USDG as settlement token.

**Scope commit** — `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, asset
`https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src`

**The defect and its three siblings**
- `packages/perps/src/CfdEngine.sol:46` — inherits `renounceOwnership()` unmodified through `CfdTreasury`
- `packages/perps/src/HousePool.sol:264-265` · `packages/perps/src/OrderRouterAdmin.sol:125` · `packages/perps/src/EmergencyPauseCoordinator.sol:102` — the three overrides that revert

**The latch and the freeze**
- `packages/perps/src/CfdEngine.sol:658`, `:665` — `clearDegradedMode()`, `external onlyOwner`, the single writer of `degradedMode = false`
- `packages/perps/src/HousePool.sol:1863-1870` — `_requireWithdrawalsLive`, and its only caller at `:832` inside `_settleLpEpoch`

**Prior finding and the remediation commit this report shows to be incomplete**
- `LOW-59` — *"Owner can leave emergency recovery ownerless"*, Acknowledged
- `e711621f8795909faa63f1abe4845550b88c4f1f` — *"harden governance, configuration, and recovery controls (#121)"*, which added the override to exactly three sources and modified `CfdEngine.sol` without adding one


## Link to Proof of Concept

https://gist.github.com/drak3p14159/845bdadb1252e353d8d8c0fea45c8870

## Proof of Concept

## Proof of Concept

### How to run it

Install the contract below at `packages/perps/test/perps/spec/admin/DegradedRenounceFrozen.t.sol`. ⚠ **The path depth matters:** the file imports `"../../BasePerpTest.sol"`, so it compiles only when placed **two levels below `test/perps/`**. Placing it at `test/perps/` yields `Error (6275): Source "BasePerpTest.sol" not found`, which looks nothing like a test failure. Then, from `packages/perps`:

```
forge test --match-contract DegradedRenounceFrozenTest -vv
```

The figures below came from an identical run in a **copy of the repository's own `test/` tree** with only this one file added, used solely to cut compile time: the file's location within `test/perps/spec/admin/` is unchanged, no in-scope contract is modified, and the command above is the one to reproduce it in the repository as shipped. **No contract in scope is modified**; the only addition is this test file. The only mocks are the harness settlement token and Pyth price updates — the two components the programme names explicitly as permitted substitutions, both outside the scope.

## Measured result

```
Ran 1 test for test/perps/spec/admin/DegradedRenounceFrozen.t.sol:DegradedRenounceFrozenTest
[PASS] test_RenouncedOwnerMakesRecoveredDegradedSettlementPermanent() (gas: 2603438)
Logs:
  STEP 1 -- a Junior LP queues a redemption while degraded mode is latched
    shares queued for redemption: 1000009142
    claimable now (expected 0): 0
  STEP 2 -- the pool is recapitalized and raw solvency is restored
    recapitalized (USDG, 6dp): 500000000000
    effective assets after: 1102040000000
    maximum liability: 999000000000
    degradedMode still latched (1=yes): 1
  STEP 3 -- CONTROL: with an owner, clearDegradedMode() clears the latch. Reverted.
    engine owner before renunciation: 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496
  STEP 4 -- the owner renounces. The only clearing path is onlyOwner, so it is now unreachable.
    engine owner after renunciation: 0x0000000000000000000000000000000000000000
    degradedMode (1=latched forever): 1
  STEP 5 -- permissionless LP epoch settlement reverts HousePool__DegradedMode. FUNDS FROZEN:
    shares still pending redemption: 1000009142
    assets ever claimable (0 = frozen): 0
    no expiry, no in-code remedy: every future settleLpEpoch reverts identically.

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

**Reading the terminal state, which is the impact this report claims.** `1,000,009,142` Junior shares sit in
`pendingRedeemRequest` and `claimableRedeemAssets` is **0**, after a permissionless `settleLpEpoch` reverted
`HousePool__DegradedMode`. The pool is solvent at that moment — effective assets `1,102,040,000,000` against a
maximum liability of `999,000,000,000` — so nothing economic prevents the funding. Only the latch does, and
`clearDegradedMode()` is `onlyOwner` with the owner now `address(0)`. STEP 3 is the control: the identical
call clears the latch while an owner exists, and that world is reverted before renunciation, so the
difference between the two is renunciation alone.

Run at scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The test is self-contained, so the figure above is reproducible by anyone running the command.

## Which assertion proves which claimed impact

The programme requires a PoC that demonstrates **the impact the report claims**. Every link in the chain is asserted, so none is assumed:

| claimed step | assertion |
|---|---|
| degraded mode latches from an ordinary close | `assertTrue(engine.degradedMode(), "setup must latch degraded mode")` |
| recapitalization restores raw solvency… | `assertGe(effectiveAssets, _maxLiability(), "recapitalization must restore raw solvency")` |
| …but does **not** clear the latch | `assertTrue(engine.degradedMode(), "recapitalization alone must not clear the latch")` |
| **the counterfactual: clearing IS reachable while an owner exists** | `vm.snapshotState()` → `engine.clearDegradedMode()` → `assertFalse(engine.degradedMode(), "raw latch clearing must be reachable before renunciation")` → `vm.revertToState(...)` |
| renunciation succeeds on `CfdEngine` | `assertEq(engine.owner(), address(0), "ownership renunciation must succeed")` |
| and does not clear the latch | `assertTrue(engine.degradedMode(), "renunciation must not clear the latch")` |
| the latch survives the warp, and the engine is still ownerless | `assertTrue(engine.degradedMode(), ...)` · `assertEq(engine.owner(), address(0), ...)` |
| **the impact: permissionless LP epoch settlement now reverts** | `vm.expectRevert(IHousePool.HousePool__DegradedMode.selector); pool.settleLpEpoch(0, 0);` |
| **and the LP's money is stuck, not merely delayed** | `assertGt(juniorVault.pendingRedeemRequest(...), 0, "redemption must remain pending")` · `assertEq(juniorVault.claimableRedeemAssets(...), 0, "no assets may become claimable")` |

⭐ **The counterfactual row is what makes "permanent" a measurement rather than an assertion.** The test proves, in the same run, that `clearDegradedMode()` *would* have worked had an owner existed — then reverts that world and shows it does not work once ownership is renounced. The difference between the two branches is the renunciation and nothing else.

## What is staged, how, and what is not explained

Disclosed here rather than left for triage to find.

**1. The degraded state is staged by the sponsor's own fixture, copied unchanged.** `test/perps/accounting/close/DegradedSettlementBehavior.t.sol:33-45`, `test_ProfitableCloseLatchesDegradedModeWhenBackingFallsBelowLiability()`, runs the identical sequence with identical figures and asserts *"Profitable close should latch degraded mode when it reveals insolvency"*, citing `ACCOUNTING_SPEC.md#degraded-mode`. **The latched state is therefore sponsor-asserted, and the staging cannot be disputed without disputing the project's own test for this state.**

**2. The frozen market is a real protocol condition, staged at the sponsor's own instant.** `vm.warp(1_709_985_600)` — the exact value `test/perps/accounting/close/EngineCloseSettlement.t.sol:59-60` warps to while asserting `isOracleFrozen()`. Frozen markets recur weekly (Friday 17:00 to Sunday 17:00 New York) and are documented and non-privileged. An `assertLt` guards the warp direction, and `isOracleFrozen()` is re-asserted **immediately before** the settlement call so a window crossing fails on the operand instead of masquerading as the guard under test.

**3. The settlement call is permissionless by the project's own construction.** `BasePerpTest.sol:387-395` branches on the same two operands as `HousePool.sol:809` and calls `pool.settleLpEpoch(0, 0)` **unpranked** on the no-position and frozen paths, reserving `vm.prank(address(router))` for the live-mark path. Its doc comment at `:386` states the intent: *"No-position and frozen-mode calls intentionally exercise the cached-mark liveness path."*

**4. Cheatcodes: `vm.warp`, `vm.prank` (router-gated mark update only), `vm.snapshotState`/`vm.revertToState` (the counterfactual). No `mockCall`, no `etch`, no `store`, no `deal`** — nothing fabricates protocol state, which is the distinction between a cheatcode-constructed state and a reachable one. The `_open`/`_close` helpers prank the router, and they appear **only in setup**: path P here is LP epoch settlement, and the demonstration enters at **P's own public entry point**, `pool.settleLpEpoch(...)`, unpranked.

⚠ **5. Two things we could not explain, recorded as unexplained.** An earlier variant attempted the live-mark path and failed with `HousePool__MarkPriceStale()` rather than `HousePool__DegradedMode()`, because `HousePool.sol:809` guards the staleness check with `hasOpenPositions && !oracleFrozen` and — since `expectedPublishTime` must equal `statusSnapshot.lastMarkTime` or `HousePool__Unauthorized` fires first — the check at `:817` reduces to *"the mark must sit in the current hour bucket."* Against that: **(i)** a diagnostic measured `engine.lastMarkTime()` two epoch buckets behind `block.timestamp` (474871 vs 474873) after an `updateMarkPrice(mark, uint64(block.timestamp))` call that did **not** revert, and we cannot say why it failed to move; **(ii)** a warp-forward-in-hourly-steps search did not reach a frozen window in 216 steps, which we also cannot explain. **Both are moot here, because `:809` is a conjunction and the frozen route falsifies its other operand.** The honest answer to *"why does the live-mark path fail?"* is **"the harness could not stage a current-bucket mark with an open position, cause unresolved."**

## Reachability

**Reachability — control:** the engine owner calls `renounceOwnership()`, an inherited `public` function taking no arguments. No attacker, no race, no privilege escalation.

**Reachability — contingency:** the engine must be in `degradedMode` at or after the renunciation — an ordinary protective transition the sponsor's own test stages, and the state `clearDegradedMode()` exists to exit. Order is immaterial: renouncing while degraded latches immediately, and renouncing while healthy arms the deployment so the next degraded transition is terminal.

**Contingency basis:** `CfdEngine.sol:658` — `clearDegradedMode()` is `external onlyOwner`, so the sole clearing path is owner-gated. That gate is what converts a renounced owner into a permanent latch.

**Reachability basis:** `exec:` (executed against the code, not reasoned about) — measured, not modelled. The run above, at the scope commit, with no in-scope contract modified.

## Source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

contract DegradedRenounceFrozenTest is BasePerpTest {
    address longTrader = address(0xD001);
    address shortTrader = address(0xD002);

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
            maxSkewRatio: 1e18,
            maintMarginBps: 100,
            initMarginBps: ((100) * 15) / 10,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _enterDegradedMode() internal {
        _fundTrader(longTrader, 100_000e6);
        _fundTrader(shortTrader, 100_000e6);
        _open(shortTrader, CfdTypes.Side.SHORT, 999_000e18, 50_000e6, 1e8);
        _open(longTrader, CfdTypes.Side.LONG, 500_000e18, 50_000e6, 1e8);
        _close(longTrader, CfdTypes.Side.LONG, 500_000e18, 20_000_000);
    }

    function test_RenouncedOwnerMakesRecoveredDegradedSettlementPermanent() public {
        _enterDegradedMode();
        assertTrue(engine.degradedMode(), "setup must latch degraded mode");

        vm.warp(block.timestamp + 1 hours + 1);
        uint256 requestId = juniorVault.requestRedeem(
            juniorVault.estimateWithdrawShares(1e6), address(this), address(this)
        );
        emit log_string("STEP 1 -- a Junior LP queues a redemption while degraded mode is latched");
        emit log_named_uint("  shares queued for redemption", juniorVault.pendingRedeemRequest(requestId, address(this)));
        emit log_named_uint("  claimable now (expected 0)", juniorVault.claimableRedeemAssets(requestId, address(this)));

        vm.warp(pool.lpEpochStart(requestId));
        uint256 markPrice = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(markPrice == 0 ? 1e8 : markPrice, uint64(block.timestamp));
        uint256 markTime = engine.lastMarkTime();

        settlementToken.mint(address(this), 500_000e6);
        settlementToken.approve(address(engine), 500_000e6);
        engine.recapitalizePool(500_000e6);
        vm.prank(address(juniorVault));
        pool.reconcile();

        uint256 claims = engine.totalTraderClaimBalanceAssets();
        uint256 assets = pool.totalAssets();
        uint256 effectiveAssets = assets > claims ? assets - claims : 0;
        assertGe(effectiveAssets, _maxLiability(), "recapitalization must restore raw solvency");
        assertTrue(engine.degradedMode(), "recapitalization alone must not clear the latch");
        emit log_string("STEP 2 -- the pool is recapitalized and raw solvency is restored");
        emit log_named_uint("  recapitalized (USDG, 6dp)", 500_000e6);
        emit log_named_uint("  effective assets after", effectiveAssets);
        emit log_named_uint("  maximum liability", _maxLiability());
        emit log_named_uint("  degradedMode still latched (1=yes)", engine.degradedMode() ? 1 : 0);

        uint256 alternativeWorld = vm.snapshotState();
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "raw latch clearing must be reachable before renunciation");
        assertTrue(vm.revertToState(alternativeWorld), "alternative-world restore must succeed");
        emit log_string("STEP 3 -- CONTROL: with an owner, clearDegradedMode() clears the latch. Reverted.");
        emit log_named_address("  engine owner before renunciation", engine.owner());

        engine.renounceOwnership();
        assertEq(engine.owner(), address(0), "ownership renunciation must succeed");
        assertTrue(engine.degradedMode(), "renunciation must not clear the latch");
        emit log_string("STEP 4 -- the owner renounces. The only clearing path is onlyOwner, so it is now unreachable.");
        emit log_named_address("  engine owner after renunciation", engine.owner());
        emit log_named_uint("  degradedMode (1=latched forever)", engine.degradedMode() ? 1 : 0);

        // Why a live-mark settlement cannot reach the degraded gate, and why this one does:
        // HousePool.settleLpEpoch guards with `if (hasOpenPositions && !oracleFrozen)` (:809) and
        // inside that branch reverts MarkPriceStale at :817 when the mark's hour bucket predates
        // block.timestamp's. _enterDegradedMode leaves the SHORT open, so the branch is entered and
        // :817 fires BEFORE _requireWithdrawalsLive (:832) -- the original measured
        // MarkPriceStale, not DegradedMode. :809 is a CONJUNCTION, so making the market frozen
        // falsifies it and drops straight into _settleLpEpoch, where :832 is unconditional.
        //
        // A frozen market is a documented, NON-PRIVILEGED protocol condition (recurring Friday
        // 17:00 -> Sunday 17:00 New York), and the sponsor's own suite settles LP epochs in it and
        // asserts "frozen mode must still fund the matured exit"
        // (test/perps/spec/lp/AsyncLpEpochSettlement.t.sol:775-779). So this stages a real state
        // and blocks a funding route the sponsor asserts works. Settlement on this path is
        // permissionless, exactly as the sponsor's own _settleLpEpochForTest() does it.
        // The frozen instant below is the project's own: test/perps/accounting/close/
        // EngineCloseSettlement.t.sol:59-60 warps to exactly this value and asserts isOracleFrozen().
        // BasePerpTest.SETUP_TIMESTAMP is 1_709_532_000 and this fixture only warps forward in small
        // steps, so the assertLt below confirms the warp direction rather than assuming it.
        assertLt(block.timestamp, 1_709_985_600, "sponsor frozen instant must be in the future here");
        vm.warp(1_709_985_600);
        assertTrue(engine.isOracleFrozen(), "setup must reach a frozen-oracle window");

        // Same refresh the sponsor's fixture uses (AsyncLpEpochSettlement.t.sol:933-937).
        uint256 mark = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(mark == 0 ? 1e8 : mark, uint64(block.timestamp));

        assertTrue(engine.degradedMode(), "latch must survive the warp into the frozen window");
        assertEq(engine.owner(), address(0), "owner must still be renounced");

        // Re-assert frozen IMMEDIATELY before the call, not only after the loop: if anything between
        // the loop exit and this line crossed out of the window, the revert would come back as
        // MarkPriceStale -- indistinguishable from the guard we are escaping. Fail on the operand
        // instead, so the diagnosis is unambiguous.
        assertTrue(engine.isOracleFrozen(), "must still be frozen at the settle call");

        // Permissionless frozen-path settlement, unpranked, mark args ignored on this path.
        vm.expectRevert(IHousePool.HousePool__DegradedMode.selector);
        pool.settleLpEpoch(0, 0);

        emit log_string("STEP 5 -- permissionless LP epoch settlement reverts HousePool__DegradedMode. FUNDS FROZEN:");
        emit log_named_uint("  shares still pending redemption", juniorVault.pendingRedeemRequest(requestId, address(this)));
        emit log_named_uint("  assets ever claimable (0 = frozen)", juniorVault.claimableRedeemAssets(requestId, address(this)));
        emit log_string("  no expiry, no in-code remedy: every future settleLpEpoch reverts identically.");

        assertGt(juniorVault.pendingRedeemRequest(requestId, address(this)), 0, "redemption must remain pending");
        assertEq(juniorVault.claimableRedeemAssets(requestId, address(this)), 0, "no assets may become claimable");
    }
}
```

