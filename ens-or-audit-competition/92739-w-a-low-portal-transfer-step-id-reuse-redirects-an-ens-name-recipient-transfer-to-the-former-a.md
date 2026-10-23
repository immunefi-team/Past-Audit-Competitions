# 92739 w a low portal transfer step id reuse redirects an ens name recipient transfer to the former address

**Submitted on Sep 11th 2026 at 22:10:16 UTC by @GaelicThunder for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92739
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

### Summary

Portal keys every transfer runner and descriptor as `transfer-${name}-${step}`. The ID contains neither an attempt identity nor the recipient. Closing Portal's transaction modal retains pending and successful actors.

This permits one completed step from an A attempt to be interpreted through a later B attempt's live callbacks:

1. Ordinary owner O opens N's name-specific **Ownership → Transfer** flow, enters A, leaves **Set the ETH address to the recipient** enabled and starts the two-step plan.
2. O authorizes the real deployed-resolver call `setAddr(N,60,A)` and closes Portal's modal while the actor is pending. The write succeeds behind the closed modal, so transfer(N→A) never starts.
3. O corrects the still-mounted form to B and clicks **Transfer name** once.
4. The B plan reuses the record-step ID. Portal matches A's retained successful actor to B's current descriptor and invokes that descriptor's `onDone`, creating a fresh `safeTransferFrom(O,B,tokenId(N),1,0x)` wallet request without creating `setAddr(N,60,B)`.
5. O authorizes the truthful B transfer. Deployed-contract reads show owner(N)=B and N balances O/A/B=0/0/1, but the real resolver returns A for `addr(N,60)`.
6. O owns another ENS v2 name X. From X's own **Ownership → Transfer** route, O enters B's exact ENS name N in the prompted **ENS name or address** field. Portal resolves its stale N record and visibly previews the complete A address.
7. O proceeds through Portal's ordinary transaction modal. The captured request targets the real registry and decodes as `safeTransferFrom(O,A,tokenId(X),1,0x)`. A has no code and zero token roles.
8. O authorizes it. The registry ends owner(X)=A with X balances O/A/B=0/1/0.

### Root cause

References use official audited revision `63772fd872af472ced58b009499355f3430c2a86`, imported locally as `cda79acaad59711b943fc68207ebb3f1d0ff8596`; the source lock binds executed target HEAD `dc8a100d58d0d00f91327d6e759e1ef435ee7a3f`, its Git tree and dependency lock. It also pins the tracked official competition archive at HEAD `1c9b47f` by SHA-256/revision comment and proves the cited production sources/runtime manifests and competition dependency lock match it. The runner owns both services and hashes the source served by Vite before and after each arm.

* `apps/portal/src/features/transfer/components/SendNameForm.tsx:68-108` retains an editable recipient and starts the plan for its current resolved value.
* `apps/portal/src/features/transfer/utils/buildTransferPlan.ts:22-42` orders `set-eth-addr` before `transfer-token`.
* `apps/portal/src/features/transfer/hooks/useTransferName.ts:109-120,148-181` replaces the saved recipient and callbacks but gives both attempts the same `transfer-${name}-${step}` IDs.
* `apps/portal/src/features/transaction-manager/components/TransactionModal.tsx:47-64` retains pending/success actors when closed.
* `packages/transaction-manager/src/providers/transactionManager.ts:225-251,338-340` stores them under the caller-provided ID.
* `apps/portal/src/features/transaction-manager/hooks/useActiveTransactionState.ts:41-67` and `useAutoAdvanceTransaction.ts:17-28` match A's successful actor to B's current descriptor and invoke B's next-step callback.
* `apps/portal/src/features/address/hooks/useAddressResolution.ts:47-82`, `features/address/queries/getResolvedAddress.ts:18-31` and `features/roles/helpers/addUser.handlers.ts:19-36` resolve typed N through the configured Universal Resolver.
* `SendNameForm.tsx:113-118,248-280` shows the warning and full resolved address; `features/transfer/helpers/transferToken.ts:63-91` encodes that address in X's `safeTransferFrom` request.

The root defect is cross-attempt identity aliasing: one attempt's terminal actor is reinterpreted through another attempt's current callback closures.

### Defect-only counterfactual

The attached patch adds `attemptId: crypto.randomUUID()` when a plan is prepared and includes it in both per-step ID sites. That is the only production-code change. Setup creates N and X once, then both arms start from the same checked- restored Anvil snapshot. The fixed B plan exposes `setAddr(N,60,B)`; O authorizes that restored intended request before transfer(N→B). Under the later identical X flow, Portal previews B, calldata targets B and the registry ends owner(X)=B with balances 0/0/1.

### Honest limitations

* O is the affected ordinary owner, with no protocol/admin privilege beyond owning N and X. A is a passive, code-less, zero-role former intended recipient; no remote injection or direct invocation by A is claimed.
* Before the shared snapshot, the repository's official fixture clears inherited EIP-7702 code only from standard synthetic O/B fixture addresses. The runner records raw-fork/post-fixture code, proves non-fixture A/payer code is unchanged, and performs no registry, resolver or role-storage override.
* O must close the modal in the pending-to-success window, then correct to B with the record option selected.
* O later enters N specifically as the identity for intended B. ENS ownership and forward resolution can legitimately differ; the claimed error is Portal omitting its selected B write and then consuming that stale A value.
* Portal shows A's complete address and warns that the transfer is irreversible and the recipient should be checked. The uncropped evidence preserves both. This reduces likelihood but does not block the approved transaction.
* B can repair N prospectively, but cannot recover X without A's cooperation.
* R3-07 shares fixed IDs but documents Medium terminal telemetry/history loss on registration/renewal retry. This PoC uses no failed retry or terminal fast-return; it omits an on-chain write and executes a separate asset transfer. The exact policy preserves “new consequences of a listed root cause that materially change its severity,” but broad same-root duplicate closure remains possible.

### Recommendation

Create a fresh immutable attempt ID for every prepared transfer and include it in every runner and descriptor ID. Never match an actor from one attempt to another attempt's callbacks. Also serialize same-name attempts while an earlier actor is in flight and bind completion rows to current transaction arguments.

## Link to Proof of Concept

<https://gist.github.com/GaelicThunder/e21ce741ce4611d3979fb5d8b1a2b0cc>

## Proof of Concept

The available in the secret Gist Playwright specification is complete and runnable at:

`e2e/projects/portal/tests/zz-poc-transfer-record-auto-advance-alias.spec.ts`

The evidence wrapper starts one setup Portal process, then executes two matched-snapshot impact arms: locked vulnerable production sources and the defect-only attempt-ID patch. It owns fresh Portal processes and a pinned local fork of public Sepolia block 11682410. It creates N and X once, restores the same snapshot for the fixed arm, and uses the project EIP-1193 test wallet plus deployed ENS v2 registry/resolver. Every load-bearing N/X arm request is captured, decoded, authorized and forwarded.

Formal reproduction from the bounty repository root requires no prestarted service:

```bash
python3 .agents/skills/bounty-hunting/scripts/capture_local_run.py \
  findings/hunts/ens-apps/candidates/portal-transfer-record-auto-advance-alias/CANDIDATE.json \
  --git-root . --workdir . --force -- \
  python3 findings/hunts/ens-apps/candidates/portal-transfer-record-auto-advance-alias/poc/run_local_capture.py
```

The revision-3 parser requires the vulnerable artifact to establish:

```
N: owner=B, addr(60)=A
X typed recipient=N
X preview=A; decoded safeTransferFrom recipient=A
X owner=A; balances O/A/B=0/1/0
```

After checked restoration of the same snapshot, it requires the fixed artifact to establish:

```
N: owner=B, addr(60)=B
X typed recipient=N
X preview=B; decoded safeTransferFrom recipient=B
X owner=B; balances O/A/B=0/0/1
```

The structured parser accepts revision 3 only when both arm pre-state tuples equal the setup baseline, all seven X assertions pass, and the sorted executed attack graph and vulnerable/fixed predicates match the candidate. See the attached formal logs, result manifest, screenshots and defect-only patch for the captured values.
