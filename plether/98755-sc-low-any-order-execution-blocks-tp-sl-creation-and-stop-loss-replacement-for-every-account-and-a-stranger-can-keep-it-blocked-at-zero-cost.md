# \#98755 [SC-Low] Any order execution blocks TP/SL creation and stop-loss replacement for every account, and a stranger can keep it blocked at zero cost

**Submitted on Nov 10th 2026 at 02:22:06 UTC by @redbaron for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98755
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

**The split.** A delayed-order execution installs its historical Pyth basket as the Engine mark (`OrderRouterExecutionSidecar._updateEngineMark`). The Oracle's historical path never writes the Oracle's own accepted live basket. After any fill, `engine.lastMarkTime` is newer than the accepted basket. `ORACLE_SYNCHRONIZATION.md:150-151` describes this state ("historical execution may advance the Engine without advancing the accepted live basket").

**The consequence.** `PositionProtectionBook._freshProtectionObservation` reads `oracle.getLatestPrice(MarkRefresh)`. That view rebuilds the accepted basket and requires `publishTime >= engine.lastMarkTime` (`PletherOracle._snapshotFromBasket`), so in the split it reverts with `PletherOracle__PriceOutOfOrder`. Three calls then revert for **every** account, and none of them takes a Pyth payload:
- `createPositionProtection`;
- `replacePositionProtection` (tightening a stop-loss);
- `commitOpenOrderWithProtection`.

**The Book's own guard says this state is valid.** The next line of the Book, `_acceptedMarkInstalled()`, returns true for `acceptedTime < markTime` (`PositionProtectionBook.sol:893`). That branch can never be reached from `_freshProtectionObservation`, because the Oracle view at `:880` reverts first. Two one-sided guards in two contracts therefore only agree at exact equality of the accepted basket and the Engine mark, and every ordinary fill breaks that equality.

**The griefing.** The state clears only when someone pays the Pyth fee for a new live `updateMarkPrice(payload)`. An attacker re-opens it one second after each refresh by self-executing their own order with a newer tick. The Engine mark is installed **before** the execution item runs, so the order does not even need to fill. An unmeetable limit price ends it as a `Slippage` terminal, and the attacker, as executor, collects their own bounty back. **The attacker pays 0 USDG, never holds a position, and blocks the whole market.**

**Measured in the PoC** (repo fixture, production `OrderRouter`, execution sidecar, `PletherOracle` and `PositionProtectionBook`):
- a keeper refreshes every 10 s;
- a victim using the stock single-call `replacePositionProtection` succeeds in **6 of 60 seconds** under attack, against **60 of 60** in the control timeline;
- in the zero-cost variant the attacker's balance stays at 1000.000000 USDG, with locked margin 0 and position size 0.

## Vulnerability Details

Code at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.

### 1. The historical path never advances the accepted basket

`PletherOracle._resolveHistoricalOrderBasket` (`PletherOracle.sol:596-643`):
- parses the unique basket in `(commitTime, commitTime + window]`, capped at `block.timestamp`;
- syncs Pyth storage;
- returns the basket.

It never assigns `_acceptedBasket` or `_acceptedPrices`. Their only writer is the live path `_resolveCurrentBasket` (`:796-798`).

### 2. The Router installs the historical tick as the Engine mark before the item runs

```solidity
// packages/perps/src/OrderRouterExecutionSidecar.sol:1054-1061 (called from _prepareSingleOracle, :979)
if (result.publishTime > engine_.lastMarkTime()) {
    engine_.updateMarkPrice(neutralMark, result.publishTime);
}
```

`_prepareSingleOracle` runs before the item is dispatched in the `try host.executeOrderItemFromSidecar` frame (`:220`). The mark is therefore installed whether the order then fills, ends as a `Slippage` terminal (`:466-471`) or stays pending on `InsufficientGas` (`:473-476`).

### 3. Every reader of the accepted basket reverts in the split

```solidity
// packages/perps/src/PletherOracle.sol:891-895 (_snapshotFromBasket, enforcePublishOrder = true)
uint64 markTime = engine.lastMarkTime();
if (snapshot.publishTime < markTime) revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
```

### 4. The protection Book reads it before its own guard, which would accept the state

```solidity
// packages/perps/src/PositionProtectionBook.sol:880-895
IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh); // reverts
currentMarkPrice = current.markPrice;
currentPublishTime = current.publishTime;
if (!_acceptedMarkInstalled()) {
    revert PositionProtectionBook__OracleMarkAhead();
}
...
function _acceptedMarkInstalled() private view returns (bool) {
    (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = _activeOracle().getAcceptedBasketMark();
    ...
    return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
}
```

`_freshProtectionObservation` is the price source for:
- `createPositionProtection` (`:207/229`);
- `replacePositionProtection` (`:257/295`);
- `commitOpenOrderWithProtection` → `_stageAttached` (`:432/463`).

None of them takes a payload, and `src` has no multicall. The reference client builds create and replace as a single Book call (`packages/perps-aa-client/src/protection.ts:41-56`).

### 5. The fee-free keeper paths fail in the same state

The empty-payload branch of `_resolveCurrentBasket` (`:759`) replays the same accepted basket. So these revert too:
- `updateMarkPrice([])`;
- `executeLiquidation(acct, [])`;
- `executeLiquidationBatch(.., [])`;
- `settleLpEpoch([])`;
- `triggerPositionProtection(id, [])`.

`ORACLE_SYNCHRONIZATION.md:29-30` documents that an empty replay works "only if it still passes the selected policy", so we list this as a side effect, not as the claim.

### Relation to prior audits and known issues

- **Mechanism documented, consequence not.** `ORACLE_SYNCHRONIZATION.md:149-152` tells monitors that historical execution can leave the accepted basket behind the Engine. No document says that protection creation, replacement or protected opens then stop working for everyone:
  - `IPositionProtectionActions.sol:12` promises that creation "validates trigger geometry against the accepted fresh basket", with no refresh precondition;
  - `SECURITY.md:563` says existing-position protection creation "remains available" even in degraded mode;
  - the Book's own `_acceptedMarkInstalled` accepts the state.
- **Not a known issue.** Cecuro MED-17 (oracle lag blocks *triggering*) and MED-23 (mixed-time basket) are fixed and have different root causes. MED-28 ("arm from a stale engine mark") was fixed by adding the accepted-observation read in `_freshProtectionObservation`. That fix creates this liveness failure in the opposite direction, which the program accepts ("shows the fix ... introduces a new bug"). The repo tests cover only the opposite direction (`OracleMarkAhead`, `PositionProtection.t.sol:289, 2479, 3431`). No repo test expects `PriceOutOfOrder` from a protection call. The SECURITY.md Known Limitations (`:1328-1450`) do not mention it.

## Impact Details

**Medium, "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)".**

| Timeline (same snapshot, keeper refresh every 10 s, victim tries every second) | Victim `replacePositionProtection` OK / tries |
|---|---|
| Control, no attacker | 60 / 60 |
| Attacker re-arms after each refresh | 6 / 60; the other 54 revert `PriceOutOfOrder` |

- **Who is hit:** every trader in the market. Each re-arm blocks all accounts at once.
- **What is blocked:** creating a TP/SL, tightening or replacing a stop-loss, and opening with attached protection.
- **Damage example:** in `test_victimDamage`, a LONG 10,000 USDG position with SL 1.10 cannot tighten its stop to 1.045 after a 4% adverse tick. It cannot cancel and re-create either: cancel works, create reverts. If price then reaches 1.10, 550 USDG of adverse move was not cut.
- **Attacker cost:**
  - **0 USDG** with a non-filling execution (Slippage terminal, or a gas-starved execution that leaves one pending order to be reused for every refresh in its window);
  - 0.40 USDG per re-arm with minimum-size filling orders, because the bounty comes back to the attacker as executor.
  - Only L2 gas and the wei-scale Pyth fee remain.
- **Even without an attacker,** every ordinary fill by any trader opens the same window until the next paid live refresh.
- **Not claimed:** no funds are frozen or stolen. The victim can still exit: cancel plus a plain close commit works (`test_victimCanStillExit`). Already armed protections still trigger with a payload.
- **Caveats:**
  - The 54/60 figure assumes a 10 s refresh cadence. At any cadence, each re-arm blocks every account until the next refresh.
  - A smart-account user who batches their own `updateMarkPrice(payload)` with the create in one transaction escapes for themselves, at the cost of a Pyth update. EOAs and the stock client cannot.

### Preconditions

- A live market where someone occasionally posts a live mark refresh.
- Attacker: an ordinary account with about 30 USDG and a little ETH. No role is needed; executing orders is permissionless.

## Attack scenario

1. A live refresh lands at second S, with tick S-1. The accepted basket and the Engine mark are both at S-1.
2. In second S, the attacker commits a LONG open with `targetPrice` 1% above the mark (unreachable).
3. At S+1, the attacker calls `executeOrder(id, payload@S+1)`:
   - `_updateEngineMark` installs mark S+1;
   - the item ends `Slippage`, the escrow is released and the bounty is paid to the attacker as executor.
4. From S+1 until the next live refresh, every account's create, replace and protected open revert with `PriceOutOfOrder`.
5. The attacker repeats steps 2-3 after every refresh.

## Recommendation

In `_freshProtectionObservation`, when the accepted basket is older than the Engine mark (installed by a historical fill), use the Engine mark as the observation. This is consistent with `_acceptedMarkInstalled`:

```diff
+        (bool accepted,, uint64 acceptedTime) = oracle.getAcceptedBasketMark();
+        if (accepted && acceptedTime < markTime) {
+            // Engine mark was advanced by a historical order execution; use it as the observation.
+            return (cachedMarkPrice, cachedMarkPrice, markTime);
+        }
         IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
```

The full diff is in the gist (`fix.diff`). With it:
- the 7 tests that assert a protection revert fail, because create and replace now succeed (60/60 in the attack timeline);
- `test_sinks_emptyPayloadPaths` still passes;
- the repo's `PositionProtectionTest` suite passes (104/104), including the opposite-direction `OracleMarkAhead` tests.

The Book fix does not restore the fee-free empty-payload keeper paths (section 5). A broader oracle-side fix covers both:
- advance `_acceptedBasket` / `_acceptedPrices` in `_resolveHistoricalOrderBasket` when the parsed basket is newer;
- or install the Engine mark only after the execution item is admitted, so a non-filling execution cannot move protocol price state.

## Link to Proof of Concept

https://gist.github.com/the-red-ace/78707c5521a66046464e547a30cbfea7

## Proof of Concept

## Proof of Concept

**Gist:** https://gist.github.com/the-red-ace/78707c5521a66046464e547a30cbfea7. It contains four test files, `fix.diff` and a README.

**How the tests run:** in the repo's own Foundry suite at a82d4fe, on the repo's `OracleSynchronizationTestBase` fixture with six feeds.
- Production `OrderRouter`, execution sidecar, `PletherOracle` and `PositionProtectionBook` are used.
- `MockPyth` stands in for Pyth, which the program allows.
- No in-scope contract is modified.
- Cheats: `vm.prank`, `vm.warp`, `vm.roll`, `vm.deal` (ETH for Pyth fees) and snapshots for the control timeline. No `vm.store`, `vm.mockCall` or `vm.etch`.
- Existing victim positions are installed with the suite's own `_open` helper, as `PositionProtection.t.sol` does. Every attacker and bystander order goes through the real `commitOrder` → `executeOrder` flow.

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
cp ProtectionAfterFill.t.sol SustainedProtectionGrief.t.sol ProtectionAfterFillSinks.t.sol ZeroCostRearm.t.sol packages/perps/test/perps/
forge test --root packages/perps --match-contract "ProtectionAfterFillTest|SustainedProtectionGriefTest|ProtectionAfterFillSinksTest|ZeroCostRearmTest" -vv
```

| Test | What it shows |
|---|---|
| `test_ProtectionBlockedAfterOrdinaryFill` | One ordinary fill leaves the Engine at T2 and the accepted basket at T0. Create, replace, protected open and `updateMarkPrice([])` revert `PriceOutOfOrder`. A paid live refresh clears it, and the next fill re-opens it. |
| `test_reactiveAttack_vs_control` | Attacker commits only after seeing each refresh. Victim OK 60/60 in the control and 6/60 under attack. |
| `test_victimCanStillExit` | Cancel plus a plain close commit still work (no freeze claimed). |
| `test_griefingLoop` | Pre-committed variant: 6/6 cycles denied. |
| `test_sinks_emptyPayloadPaths` | `executeLiquidation`, batch liquidation, `settleLpEpoch`, `updateMarkPrice` and trigger with `[]` revert `PriceOutOfOrder`. Before the fill they reach their own guards. |
| `test_victimDamage` | Tighten 1.10 → 1.045 and cancel plus re-create both fail; 550 USDG of uncut move on 10k. |
| `test_A_slippageTerminal_rearmsForFree` | Non-filling execution: attacker 1000.000000 → 1000.000000 USDG, size 0; the victim's replace reverts. |
| `test_B_gasStarvedPending_reusableFreeRearm` | A 580k-gas execution leaves the order pending (`InsufficientGas`). One order re-arms 4/4 cycles at zero cost. |

Expected output (reproduced on a clean a82d4fe checkout with only the gist files):
```
[PASS] test_ProtectionBlockedAfterOrdinaryFill()
[PASS] test_reactiveAttack_vs_control()
  CONTROL tries / ok / PriceOutOfOrder 60 60 0
  ATTACK  tries / ok / PriceOutOfOrder 60 6 54
[PASS] test_victimCanStillExit()
[PASS] test_griefingLoop()
[PASS] test_sinks_emptyPayloadPaths()
[PASS] test_victimDamage()
[PASS] test_A_slippageTerminal_rearmsForFree()
[PASS] test_B_gasStarvedPending_reusableFreeRearm()
8 tests passed, 0 failed
```

