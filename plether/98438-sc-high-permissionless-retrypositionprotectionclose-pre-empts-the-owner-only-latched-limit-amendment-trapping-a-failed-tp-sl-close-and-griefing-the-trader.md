# \#98438 [SC-High] Permissionless retryPositionProtectionClose pre-empts the owner-only latched-limit amendment, trapping a failed TP/SL close and griefing the trader

**Submitted on Nov 8th 2026 at 03:33:08 UTC by @StevenJ for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98438
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Description

### Summary

A triggered TP/SL close can fail a ceiling the trader set. The protection then becomes `Latched`. The documented way out is for the owner to amend the ceilings with `amendLatchedPositionProtectionCloseLimits` and then retry. The acknowledged Cecuro MED-9 response relies on the same remedy.

The owner's amendment and the permissionless `retryPositionProtectionClose` share one precondition: the protection is `Latched` and the account has zero pending orders. Execution is also permissionless. So any account can execute the failing child and retry it in the same transaction, once per second. The protection is then never observably `Latched`, and every owner amendment reverts with `OrderRouter__ProtectionNotLatched()`.

While this continues, the trader cannot amend, cancel, replace or submit an ordinary close. The position stays fully exposed in the adverse direction that triggered the stop.

The PoC uses a real ERC-4337 SimpleAccount and the real EntryPoint on an Arbitrum Sepolia fork, with production Plether contracts at release parameters. In it the owner is blocked for a full hour: 3,600 of 3,600 SDK-shaped amendment UserOps fail. The trader loses 100.040511 USDG more than in the matched honest branch.

The resulting impact is Griefing.

### Affected contracts

Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`:

- `PositionProtectionBook.sol`:
  - `amendLatchedPositionProtectionCloseLimits` (L315-L327)
  - `retryPositionProtectionClose` (L382-L405)
- `router/OrderCommitHandler.sol`: the pending-order counter (L43-L59)
- `router/OrderLiquidationBatchLogic.sol`: the protection lock on ordinary commits (L313)

The deployed Arbitrum Sepolia v2.0.0 release runs the same source. `git diff 1209ca8a..a82d4fef -- packages/perps/src` is empty.

### Root cause

The owner's only remedy for a latched protection requires a childless `Latched` state:

```solidity
// PositionProtectionBook.sol L315-L327
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}
```

The retry needs exactly the same state, but anyone may call it. It immediately moves the protection back to `Triggered` with a fresh child that carries the old, still-failing limits:

```solidity
// PositionProtectionBook.sol L382-L405
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);   // no owner check
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);
    IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
    if (router.pendingOrderCounts(protection.account) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    ...
    _recordCloseAttempt(protection, linkedOrderId);
}
```

The new child increments `pendingOrderCounts`:

```solidity
// router/OrderCommitHandler.sol L55
if (++pendingOrderCounts[account] > maxPendingOrders) {
```

The owner has no other way out while a protection is active:

- Ordinary commits are rejected with `OrderRouter__ProtectionActive` (`OrderLiquidationBatchLogic.sol` L313).
- `replacePositionProtection`, `cancelPositionProtection` and `cancelPositionProtectionAndCommitClose` only accept `PendingOpen` or `Armed`.

The attack sequence is:

```
trader sets a finite close ceiling (documented style)
        ↓
stop-loss triggers; close child fails its ceiling → protection Latched
        ↓
griefer tx (once per second, at commitTime + 1, the earliest second any caller can execute,
            since OrderRouterExecutionSidecar.sol L450 rejects publishTime <= commitTime):
    executeOrder(child)  → Failed → Latched
    retryPositionProtectionClose(pid) → new child, same ceiling → Triggered
        ↓
owner's amendLatchedPositionProtectionCloseLimits always lands while a child is live
        ↓
reverts OrderRouter__ProtectionNotLatched()
        ↓
position remains open and exposed; repeat
```

### The documented remedy this breaks

- `WORKING_WITH_AI_AGENTS.md` L284: "protection becomes `Latched`; the owner may amend its limits before retrying."
- `CANONICAL_ENTRYPOINTS.md` L145-L148: "The owner may amend the close ceilings of a `Latched` protection with `amendLatchedPositionProtectionCloseLimits` …"
- `DEPLOYMENT.md` L419-L421 tells honest keepers: "Investigate planner, constraint, or other persistent failures before retrying; do not continuously queue the same failing attempt." Honest infrastructure therefore leaves the owner's window open. Only an adversarial keeper closes it, and keepers are untrusted in this program.
- The Cecuro MED-9 response (Acknowledged/Addressed) says: "Only the owner may amend the retained ceilings while protection is Latched … Tight limits may prevent execution until conditions improve or the owner amends them." The second half of that remedy can be denied by any third party.

### Impact

The impact is Griefing (Medium). A trader whose protected exit fails its ceiling loses the ability to fix the ceiling for as long as an unprivileged account keeps rotating. During that time:

- The protected close is held open. The owner cannot amend, cancel, replace or submit an ordinary close.
- The full position keeps its exposure in the adverse direction that triggered the stop, so the extra loss scales with position size, the length of the hold and the size of the adverse move. If the move continues far enough, the position can reach liquidation. The PoC below measures only the one-hour drift case.
- Every failed amendment attempt costs the victim gas.

Measured in the PoC, with release parameters, a 10,000-lot LONG, $500 margin, a $25 gross-debit cap and a price drift from 1.005 to 1.015:

```
rotations / blocked owner amendments : 3,600 / 3,600 over 3,600 s
honest branch final wealth           : 49,941.580000 USDG
attacked branch final wealth         : 49,841.539489 USDG
extra loss caused by the block       : 100.040511 USDG
```

The attacker gains nothing. The trigger bounty is paid once and retries are unpaid. The attack costs about 1.3M gas per rotation for each victim, independent of the victim's position size. This is the "no profit motive, damage to users" griefing case.

### Attack prerequisites

- **The trader set a finite action-charge or gross-debit ceiling.** This is the documented style. The SDK README says "use `maxUint256` only when unlimited authority is intended … builders do not insert unlimited defaults" (`packages/perps-aa-client/README.md` L208-L216). All four limits are mandatory, and zero means zero allowance (`WORKING_WITH_AI_AGENTS.md` L253).
- **One close attempt failed that ceiling.** A single ordinary adverse move does this. In the PoC, the same $25 cap passes at 1.001 and fails at 1.005, where the actual gross debit is 53.520999.
- **The owner uses the standard amendment path.** The official SDK builder `buildAmendLatchedProtectionCloseLimitsAction` (`perps-aa-client/src/protection.ts` L103-L107) emits a lone `amendLatchedPositionProtectionCloseLimits` call. The PoC uses this call shape from a real SimpleAccount.

No owner, guardian, pauser or other trusted role is involved, and no configuration change is needed. The griefer only calls the public `executeOrder` and `retryPositionProtectionClose`.

The only escape is a custom smart-account batch, `[executeOrder(child, freshPythPayload), amendLatched…, retry…]`, which must land in the same second as the griefer. The SDK has single-call keeper builders (`keeperExecution.ts`) and the lone amendment action (`protection.ts` L103-L107), but no owner action that combines them. The trader must also obtain a fresh signed Pyth payload and assemble the batch themselves. The PoC demonstrates this escape. A trader who chose `uint256.max` limits is not affected.

### Observed testnet usage of unlimited limits

**Fact.** In the public v2.0.0 Arbitrum Sepolia `PositionProtectionBook` (`0x81e3b03cf29865b6c0f85925839567e2bc61073d`), between blocks 325,228,631 and 326,657,959:

- 14 protections were created.
- All 14 `PositionProtectionCloseLimitsUpdated` events set all four limits to `uint256.max`.

Anyone can reproduce this from public event logs.

This does not narrow the affected set, for the following reasons:

- **The contract and the official SDK do not default to unlimited limits.** The SDK README (`packages/perps-aa-client/README.md` L211-L214) says: "Zero is a zero allowance; use `maxUint256` only when unlimited authority is intended. Builders reject missing or invalid close limits; they do not insert unlimited defaults." `WORKING_WITH_AI_AGENTS.md` L253 requires all four limits to be supplied explicitly. So an unlimited setting is an application-level choice made on a testnet with mock collateral and a handful of protections. The documented intended use is a finite bound.
- **Finite limits are the protection MED-9 introduced.** The MED-9 response says "uint256.max explicitly opts into unlimited allowance". A trader who sets unlimited limits gives up that protection, and any keeper may then fill the stop-loss at any cost. This finding affects every trader who uses the MED-9 protection as designed.
- **Program rules.** "Impact is calculated first, separately from feasibility. Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it." For the keeper side, the program also says: "Researchers should consider any contract-permitted behavior, including deviations from the reference implementation."

### Why this is not a known issue: direct evidence

The program rules say: "For findings marked as fixed or Acknowledged … a report on the same root cause is valid only if it shows that the fix at commit a82d4fef is incomplete or introduces a new bug." Each item below is a public commit, test or file that anyone can check.

1. **"Owner amendment after relatch" is itself the MED-9 fix.**
   - The fix is commit `eadc5c58` (2026-10-23, PR #124). Its commit message reads: *"MED-9: retain trader-selected close financial ceilings across protection triggers and retries, with owner amendment after relatch."*
   - That same commit added `amendLatchedPositionProtectionCloseLimits` and its regression test.
   - This report shows that the owner-amendment part of the fix can be blocked indefinitely by any account. So this is the "fix is incomplete" case the rules accept, not a re-report of MED-9's root cause (missing trader limits).

2. **The project's own MED-9 regression test only covers the friendly ordering.**
   - The test is `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry` (`packages/perps/test/perps/spec/trader/PositionProtection.t.sol` L954-L1000), added in `eadc5c58`.
   - In it, ALICE amends first and then a stranger, CAROL, retries. The test asserts "amended retry must execute".
   - The assertion just above it (L950) states the intended design: *"failed protected close must relatch for a **trader-authorized** retry."*
   - The PoC test `ReorderedMed9Regression.t.sol` reuses that exact fixture and setup and changes one thing: CAROL's permissionless retry comes before ALICE's amendment. ALICE's amendment then reverts on every one of 20 rounds, and the position is never closed. The upstream ordering still passes in the same file.

3. **The project's own reference keeper never retries constraint failures.**
   - In `Plether-Fi/plether-app` `apps/backend/protection-worker/policy.mjs` L15-L22, `retryDecision` returns `'retry'` only when the previous attempt failed with `Expired` (`outcome.reason !== 2 || outcome.status !== 3` → `'operator-required'`).
   - `DEPLOYMENT.md` L419-L421 gives the same instruction in prose.
   - So the protocol's intended operation leaves the latched window open for the owner. Only an untrusted caller closes it.
   - The program states: "Keepers are not trusted … harm caused by any caller acting as a keeper is in scope". It also says: "Researchers should consider any contract-permitted behavior, including deviations from the reference implementation".

4. **The `SECURITY.md` known limitation predates the amendment function and describes a different failure.**
   - The bullet at `SECURITY.md` L1356-L1359 ("Liveness depends on keepers … operators must alert on prolonged `Latched` state without a live child") was written before the amendment existed:
     - the alert sentence was added in commit `5661cc55` on 2026-08-29;
     - the "Liveness depends on keepers" wording was added in `0dbcc324` on 2026-10-15.
   - `amendLatchedPositionProtectionCloseLimits` did not exist until `eadc5c58` on 2026-10-23, so the limitation cannot describe a race against it.
   - The limitation covers retries that are too few, and its mitigation is an alert on a prolonged childless `Latched` state.
   - The PoC asserts that this alert condition is observed **0 times** during the attack, because a griefer child is always live. The documented mitigation cannot detect this issue.

5. **It is a fixable defect, not a design constraint.** A 5-line change lets the owner amend while `Triggered` when the only pending order is the protection's own child. The live child keeps its limits, and the next retry uses the new ones.
   - With the change (`fix.patch` in the Gist), the same griefer ordering no longer blocks the owner: the griefer's own retry copies the amended limits and the close executes (`test_Fix_GrieferOrderingNoLongerBlocksOwner`).
   - Across the full `packages/perps` suite (2,672 tests), the patched source fails only these, compared with the unpatched baseline:
     - the one assertion in the MED-9 regression test that the owner cannot amend after a retry, which is the restriction being removed;
     - this report's attack test, which no longer reproduces under the fix.
   - The remaining 10 failures are identical before and after the patch: fork lanes without an RPC variable, and non-IR bytecode-size checks.

The ceilings themselves are enforced correctly. The defect is that the owner remedy and the permissionless retry share one precondition.

Note on scope: the vulnerable code is the in-scope `PositionProtectionBook`. The SDK (`perps-aa-client`, out of scope) is cited only as evidence of how a standard owner submits the amendment. The program also notes that the reference setup "requires … paid Hermes API access for signed Pyth updates". So the custom execute-amend-retry escape batch requires the trader to obtain signed oracle data, which the reference setup gets through paid API access, and to compose the batch themselves.

### Recommendation

Any one of the following fixes the issue:

- Allow the owner to amend while `Triggered`, with the new limits applied to the next attempt.
- Add an owner-only `amendLatchedAndRetry` that amends and requeues atomically. Optionally, give the owner an exclusive grace period after a failed attempt before `retryPositionProtectionClose` becomes permissionless.
- Have `retryPositionProtectionClose` refuse to requeue an attempt that failed with `ConstraintViolation`, unless the caller is the owner. This matches the keeper policy in `DEPLOYMENT.md` L419-L421.

### References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L327
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L43-L59
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L284
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L145-L148
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L419-L421
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1353-L1359
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa-client/README.md#L203-L216
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa-client/src/protection.ts#L103-L107
- Cecuro AI audit report, MED-9: "Keepers can retry protection closes without the trader's financial limits" (Acknowledged)
- MED-9 fix commit: https://github.com/immunefi-team/audit-comp-plether/commit/eadc5c587e2f2f1ec26c51132472ab876d6edc15
- Commit that added the SECURITY.md latched-liveness bullet: https://github.com/immunefi-team/audit-comp-plether/commit/5661cc555f0b316100fe64b681a31954b40416e5
- Project MED-9 regression test: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/trader/PositionProtection.t.sol#L950-L1000
- Reference keeper retry policy: https://github.com/Plether-Fi/plether-app/blob/04fc8da8a14059037fe4ae18a388acaeeeb0ad9b/apps/backend/protection-worker/policy.mjs#L15-L22
- v2.0.0 Arbitrum Sepolia PositionProtectionBook: https://sepolia.arbiscan.io/address/0x81e3b03cf29865b6c0f85925839567e2bc61073d#events

## Link to Proof of Concept

https://gist.github.com/Steven-J-22/e6cd7f254b2c1633c4b6a79a7d121515

## Proof of Concept

### Proof of Concept

The PoC is in the secret Gist

It has three parts, and `run.sh` runs all of them:

1. **`ReorderedMed9Regression.t.sol`:** the project's own MED-9 regression test, using the project's own fixture, with the permissionless retry ordered before the owner's amendment. It runs without a fork.
2. **`LatchedAmendGriefFork.t.sol`:** the impact PoC, described below. It uses a real ERC-4337 SimpleAccount and release parameters, and runs for one hour.
3. **`FixValidation.t.sol` + `fix.patch`** (optional, `RUN_FIX=1`): applies the minimal fix and shows the same attack ordering no longer blocks the owner. It also runs the project's PositionProtection suite. The source is restored afterwards.

### Part 1: reordered upstream regression (`test_ReorderedRetryFirst_OwnerAmendmentNeverLands`)

Setup is identical to the upstream regression test: a 1 USDG execution-notional ceiling, trigger, and the first attempt expires, so the protection is `Latched`.

1. CAROL calls `retryPositionProtectionClose`.
2. For 20 rounds:
   - ALICE's `amendLatchedPositionProtectionCloseLimits` reverts with `OrderRouter__ProtectionNotLatched`.
   - The protection is never observed `Latched`, and `pendingOrderCounts(ALICE) == 1` throughout.
   - CAROL executes the child, which fails with `ConstraintViolation`, and retries.
3. At the end the position is still fully open, and the protection is `Triggered`.

`test_UpstreamOrder_OwnerAmendsThenRetryCloses` in the same file shows that the upstream ordering closes the position.

### Part 2: impact on a real smart account (`test_SmartAccountBlockedOneHour`)

- **Environment.** It runs in Foundry at the in-scope commit, forking Arbitrum Sepolia.
- **Real AA layer.** The victim is a SimpleAccount v0.8 created through the real deployed factory `0x13E9ed32155810FDbd067D4522C492D6f68E5944`. Its implementation is `0x28426d752372D68d34340bd94390950DcE3C9ec3`. Every victim action is a UserOperation signed by the owner key and executed by the real EntryPoint `0x4337084D9E255Ff0702461CF8895CE9E3b5Ff108`. All three codehashes are asserted.
- **Production contracts.** The Plether perps stack is deployed from the in-scope source, including the production `OrderRouter` rather than the test adapter. `setUp` asserts the release parameters: VPI 0.01e18, max skew 0.4e18, maintenance 10 bps, initial 20 bps, FAD 300 bps, 4 bps execution fee, 5 pending orders, 60 s execution window and 15 s settlement window.
- **Mocks and cheatcodes.** Only Pyth (`MockPyth`) and the settlement token are mocked, which the rules allow. No `vm.store`, `vm.mockCall` or `etch` touches an in-scope contract. The only `etch` is the repository's own ArbSys shim (`ArbitrumForkSupport.install()`), as in `test/fork/PerpsAccountAbstractionFork.t.sol`.

The test `test_SmartAccountBlockedOneHour` reproduces the attack in these steps:

1. **Open.** One UserOp batch from the SimpleAccount approves and deposits 50,000 USDG and commits a 10,000-lot LONG with $500 margin. A keeper executes the open at 1.000.
2. **Protect.** A second UserOp calls `createPositionProtection`: TP 0.999, SL 1.001, close limits `{notional: max, grossAccountDebit: 25e6, actionCharge: max, explicitFees: max}`.
3. **Baseline.** From a snapshot, triggering at 1.001 closes successfully within the same $25 ceiling.
4. **Natural failure.** The price gaps to 1.005. A stranger triggers and executes the child, which fails with `ConstraintKind.GrossAccountDebit`, actual 53.520999 against a limit of 25.000000. The protection is now `Latched` and `pendingOrderCounts == 0`.
5. **Matched control.** From a snapshot, the owner sends the SDK-shaped UserOp `execute(book, 0, amendLatchedPositionProtectionCloseLimits(pid, max))`. It succeeds, and the retry and close follow. Final wealth is 49,941.580000.
6. **Attack.** The griefer calls `retryPositionProtectionClose(pid)`. Then, for 3,600 consecutive seconds, a griefer contract calls `executeOrder(child)` (status `Failed`) and `retryPositionProtectionClose(pid)` in one transaction, while the price drifts from 1.005 to 1.015.
7. **Owner blocked.** After every rotation the owner submits the same amendment UserOp. 3,600 of 3,600 fail. The EntryPoint's `UserOperationRevertReason` contains `OrderRouter__ProtectionNotLatched()` (`0x5b54e624`). After every rotation, `pendingOrderCounts(account) == 1` and the full 10,000-lot position is still open.
8. **Exit.** After 3,600 s the only exit, a custom atomic batch of execute, amend and retry, closes the position. Final wealth is 49,841.539489, an extra loss of **100.040511 USDG**.

The other tests:

- `test_SmartAccountOwnerCannotAmend` is the same attack with 30 rotations.
- `test_HonestCounterfactual` is the control alone.
- `test_AtomicEscape` is the escape alone.
- `test_OneRotationTrace` is a single-rotation minimal trace, intended for use with `-vvvv`.

### Requirements

- Foundry
- Git
- An Arbitrum Sepolia RPC. The default is the public `https://sepolia-rollup.arbitrum.io/rpc`.

### Run

```bash
bash run.sh            # parts 1 and 2
RUN_FIX=1 bash run.sh   # also part 3 (fix validation); the source is restored afterwards
```

`run.sh` clones `immunefi-team/audit-comp-plether` and checks out commit `a82d4fef` with its submodules. It then runs:

```bash
# part 1 (no fork)
cd packages/perps && FOUNDRY_TEST=test/poc FOUNDRY_VIA_IR=false \
  forge test --match-contract ReorderedMed9RegressionTest --match-test 'test_(UpstreamOrder|ReorderedRetryFirst)' -vv
# part 2 (fork)
FOUNDRY_TEST=test/poc FOUNDRY_VIA_IR=true FOUNDRY_MEMORY_LIMIT=4294967296 \
  forge test --root . --match-path test/poc/LatchedAmendGriefFork.t.sol --block-gas-limit 1000000000000 -vv
```

- Part 2 forks the latest block by default, because public RPCs keep little historical state. `FORK_BLOCK_NUMBER=<n>` pins a block.
- The memory and gas flags are needed only because the one-hour case runs 3,600 rotations and 3,600 UserOps inside one Foundry test transaction.
- No public-chain transaction is sent.

### Expected result

```
=== (1) Project's own MED-9 regression test, retry reordered before the owner's amendment ===
[PASS] test_ReorderedRetryFirst_OwnerAmendmentNeverLands()
[PASS] test_UpstreamOrder_OwnerAmendsThenRetryCloses()
Suite result: ok. 2 passed; 0 failed; 0 skipped
=== (2) Real ERC-4337 SimpleAccount on an Arbitrum Sepolia fork, release parameters, 1 hour ===
[PASS] test_AtomicEscape()
[PASS] test_HonestCounterfactual()
[PASS] test_OneRotationTrace()
[PASS] test_SmartAccountBlockedOneHour()
  PoC first failure actual/limit 53520999 25000000
  PoC elapsed seconds/rotations/blocked AA ops 3600 3600 3600
  PoC honest/attacked/extra loss assets 49941580000 49841539489 100040511
[PASS] test_SmartAccountOwnerCannotAmend()
  PoC rotations/blocked AA ops 30 30
Suite result: ok. 5 passed; 0 failed; 0 skipped
=== (3) Fix validation (patched source, restored afterwards) ===        [RUN_FIX=1 only]
[PASS] test_Fix_GrieferOrderingNoLongerBlocksOwner()
Suite result: ok. 1 passed; 0 failed; 0 skipped
PositionProtectionTest: 103 passed; 1 failed
[FAIL: next call did not revert as expected] test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry()
  (expected: this is the upstream assertion that the owner cannot amend after a retry, i.e. the restriction the fix removes)
```

The test asserts that:

```
same ceiling passes at 1.001, fails at 1.005      (genuine ceiling failure)
honest branch: owner amendment succeeds, close executes
attack branch: 3,600 / 3,600 owner amendment UserOps revert ProtectionNotLatched
attack branch: pendingOrderCounts == 1 and full position open after every rotation
honest final wealth > attacked final wealth
reordered upstream regression: owner amendment reverts 20/20, Latched-without-child observed 0 times
with fix.patch: the same griefer ordering ends with the position closed; the live child's limits are unchanged
```
