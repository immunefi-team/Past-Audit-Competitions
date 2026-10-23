# 92042 w a critical ensv2 migration grants role set resolver on migrated names to a subgraph sourced manager address it never verifies on chain or shows the user resolver hijack&#x20;

**Submitted on Sep 6th 2026 at 15:45:56 UTC by @YFarmX for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92042
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

### Summary

When a user migrates an **unwrapped** `.eth` 2LD from ENSv1 to ENSv2, the Manager app takes a "manager" address from the V1 subgraph's `owner.id` field (the ENSv1 registry controller) and, if it differs from the registrant, without telling the user plans a "manager restoration": the user signs `ETHRegistry.setApprovalForAll(HCA, true)`, and inside the opaque `executeByOwner(bytes)` migration batch the HCA calls `ETHRegistry.grantRoles(labelToCanonicalId(label), ROLE_SET_RESOLVER, managerAddress)`.

The problem is not that a grant happens — it is that **the beneficiary is taken from an off-chain index, is never checked against on-chain state, and is never shown to the user**, while every *other* migration input is certified on-chain. The app verifies the *registrant* on-chain (`BaseRegistrar.ownerOf`, in preflight and again on retry) but performs **no** on-chain check on the address it is about to grant resolver-control to. With `ROLE_SET_RESOLVER`, that address can call `ETHRegistry.setResolver(id, …)` and redirect every `addr()`/`contenthash()` answer for the name (and, by wildcard, its subnames) — so payments to the name and the site behind it become attacker-controlled — until the victim discovers and revokes a grant the app never disclosed.

### Root cause (all in-scope unless noted)

1. **Beneficiary comes only from the subgraph** — derivation `packages/migration/src/service/classifyNames.ts:162-167` (out of scope; pure mapping): `managerAddress = owner.id` when `owner.id != registrant.id`, else `null`. `owner.id` is the `owner { id }` field of `GET_NAMES_QUERY` (`apps/manager/src/features/migration/service/v1SubgraphClient.ts:22-49`, field at L36), fetched from `v1-graphql.ens.dev` and cached \~5 min (`hooks/useV1Names.ts:22`). No chain read is involved.
2. **The grant is built with a hardcoded role** — `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9-27`: `ROLE_SET_RESOLVER = 1n << 24n` (L9); `grantRoles(labelToCanonicalId(label), ROLE_SET_RESOLVER, managerAddress)` on `V2_CONTRACTS.ETHRegistry`.
3. **It executes inside the opaque atomic batch** — `service/buildAtomicMigrationBatches.ts:608-625` appends the grant unconditionally when `managerAddress` is set (plus a `manager-role` verification expectation); `:774-780` orders it immediately after `migrate`, wrapped by `buildHcaOwnerExecutionCall` into a single `executeByOwner((address,uint256,bytes)[])` the wallet cannot inspect. The `setApprovalForAll(HCA, true)` that enables it is planned in `service/migrationApprovals.ts:116-123,320-324,343-352` and `service/computeMigrationPreflight.ts:205-207`; executed via `service/migrationService.ts:333-334,1189-1204`.
4. **"Verification" only confirms the grant landed** — `service/verifyAtomicMigrationBatch.ts:337-354`: the `manager-role` case reads `hasRoles(resource, roleBitmap, account)` and nothing else. There is **no** `ENSRegistry.owner(namehash(name))` read anywhere in the migration service (grep of `apps/manager/.../migration` and `packages/migration/src`: none). The asymmetry is the bug: the registrant is on-chain-verified (`packages/migration/src/service/preflightChecks.ts:57-71`; retry recheck `migrationService.ts:717-724`), the grant beneficiary is not. In other words the app already refuses to trust this same subgraph for *ownership* — it re-reads `BaseRegistrar.ownerOf` on-chain before every migration and again before every retry — yet it performs a **persistent on-chain privilege grant** to a third party straight from the unverified off-chain snapshot. The defect is an inconsistent trust model around a privileged write, not the manager-restoration goal itself.
5. **The beneficiary is never disclosed** — the only user-facing copy is *"Approve manager restoration — Restore the existing managers for your names"* (`components/WalletConfirmationStepsDialog.tsx:80-89`; progress copy `components/GameStep.tsx:135-138`, `migrationService.ts:333-334`). `managerAddress` is rendered by **no** component (grep of `apps/manager/src`: hits only in `service/*` and tests, never in `components/`, `hooks/`, `state/`, `pages/`). The wallet shows only `executeByOwner(bytes)`.

On-chain effect (verified against the upstream ENS **v2 contracts**, which are out of the app's audit scope but define what the grant does): `PermissionedRegistry.setResolver` is gated by `ROLE_SET_RESOLVER` **alone** (`contracts-v2` `PermissionedRegistry.sol:150-154` → `_checkRoles` L611-621); an `isApprovedForAll` operator inherits the owner's roles (`_getRoles` L576-590); `RegistryRolesLib.ROLE_SET_RESOLVER == 1 << 24` (matches the app's hardcoded `1n<<24n`). The migration controller deliberately clears the V1 record to the graveyard (`UnlockedMigrationController.onERC721Received` → `reclaim(tokenId,this)` + `setRecord(node, GRAVEYARD, 0, 0)`; corroborated in-repo by `MIGRATION_CASE_STUDY.md:70-76`), and `BaseRegistrar.transferFrom` does not update `ENSRegistry.owner` — which is exactly why a marketplace buyer's `owner.id` is still the seller.

### Why this is a vulnerability, not a feature

The contracts intentionally give the migrated name a **clean slate** (record → graveyard). The app then re-creates a persistent third-party authority over it from an unverified, undisclosed off-chain field. Two scenarios:

**A — Common case (marketplace name, no `reclaim()`).** Buyer owns `alice.eth` (ERC-721) but never called `reclaim()`, so the V1 registry `owner` is still the seller (standard ENS behaviour; the v4 Manager app surfaces this nowhere). On migration the app grants the **seller** `ROLE_SET_RESOLVER` on the V2 name. Honest caveat: pre-migration the seller already held broader V1 control over the node, and post-migration the owner can `revokeRoles` — so the marginal new exposure is "a narrowed authority is silently carried across the clean-slate boundary, and the user is never told, so they don't know to revoke." This is the case that caps the severity at Medium.

**B — Escalation (the strong case): beneficiary with no on-chain claim.** Because the beneficiary is never reconciled with `ENSRegistry.owner(node)` and the subgraph is cached/asynchronous, the grant can go to an address that holds **no** authority on-chain at migration time: e.g. a user who `reclaim()`s away a rogue/abandoned manager and migrates within the \~5-min index/cache window, or any stale/poisoned `v1-graphql.ens.dev` response. Here the app manufactures brand-new resolver-control for a party with no legitimate claim → clean resolver hijack of the migrated name, undisclosed. This is an ordinary journey, not a contrivance: the migration list is served from a 5-minute react-query cache that survives SPA navigation (`hooks/useV1Names.ts:22`), so a buyer who notices the missing "Manager" pill on the dashboard (`dashboard/v1NameRoles.ts:22-29`), reclaims in the V1 app, and returns to "Upgrade Names" within five minutes has the just-removed manager silently re-granted resolver control — **the app reverses the user's own remediation.**

### Attack scenario (case B, end-to-end)

1. `alice.eth` (unwrapped) has a stale/rogue V1 manager `M` (≠ registrant). The user `reclaim()`s to remove `M`, or `M` was already illegitimate; on-chain `owner` no longer favours `M`, but the subgraph still reports `owner.id = M` (cache/lag).
2. User opens Migration and completes it. The plan derives `managerAddress = M`, adds *"Approve manager restoration"* (no address shown), and inside `executeByOwner` runs `grantRoles(labelToCanonicalId('alice'), 1<<24, M)`.
3. The migration controller has cleared the V1 record to the graveyard; on V2, `M` now holds `ROLE_SET_RESOLVER` on `alice.eth`.
4. `M` calls `ETHRegistry.setResolver(id, evilResolver)` (only `ROLE_SET_RESOLVER` is checked). Every ENSv2 lookup of `alice.eth` and `*.alice.eth` returns attacker values → payments to the name go to `M`, `alice.eth.limo` serves `M`'s content. The victim can only revoke if they discover a grant the app never showed them.

### Impact

The app places resolver control of the migrated name in the hands of an address it neither verifies on-chain nor discloses. Consequences: **misdirection of arbitrary future payments and web content** for the name (and wildcarded subnames), persistent until the victim discovers and revokes the role. No attacker interaction is needed at migration time; the precondition (subgraph `owner ≠ registrant`) is routine for secondary-market names and for names whose manager was rotated/abandoned.

### Severity justification

Filed **Medium.** For the common marketplace case (A) the beneficiary already had broader V1 authority and the owner can revoke, so the marginal harm is "silent carry-over of authority across a clean-slate migration + no disclosure." The escalation case (B) is a genuine hijack granting authority to a party with no on-chain claim, but depends on an index/cache-timing or subgraph-integrity precondition. Either way — unlike a self-inflicted loss — the outcome is a persistent permission held by *someone other than the user* over the user's asset, in a transaction whose beneficiary the app neither renders (`executeByOwner(bytes)` is all the wallet sees) nor verifies, and it outlives the flow's own cleanup, which revokes the HCA operator approval but never the manager role (`migrationApprovals.ts:74-81`; `migrationService.ts:449,1207`). Net: Medium. (The selected impact is a "name hijack during migration" bucket, so the program may weigh it higher; that is upside, not claimed here.)

### Scope

`packages/migration` is a `private: true` workspace package whose only production consumer of `managerAddress` is `apps/manager`; the derivation it holds is a pure data mapping (`owner.id ≠ registrant`). Every security-relevant decision is in-scope `apps/manager` code: turning that field into a hardcoded on-chain role grant (`buildRoleGrantCalls.ts`), planning the operator approval (`migrationApprovals.ts`, `computeMigrationPreflight.ts`), executing it inside the opaque `executeByOwner` batch (`buildAtomicMigrationBatches.ts`, `migrationService.ts`), verifying only that it landed (`verifyAtomicMigrationBatch.ts`), and rendering address-free copy (`WalletConfirmationStepsDialog.tsx`, `GameStep.tsx`). The in-scope app pins and invokes the derivation in its own suite (`apps/manager/.../service/classifyNames.test.ts`; `computeMigrationPreflight.ts:201`). The fix (read `ENSRegistry.owner(node)` on-chain, display the address, make it opt-in) lives entirely in `apps/manager`. Even a bit-perfect `classifyNames` would leave the in-scope defect intact — `buildAtomicMigrationBatches.ts:608` turns any non-null `managerAddress` into a `grantRoles` with no reconciliation and no display — and the fix is exactly the on-chain read the sibling portal already performs for display (`apps/portal/src/features/ownership/components/V1NameManagerRecord.tsx:35-56`). Submitted as: **in-scope code executes an unverified, undisclosed on-chain privilege grant.** (Note: the `/migration` route is behind the PostHog `MIGRATION` flag — rollout gating, not a disable; the grant itself is unconditional in the normal path — no opt-in, no `owner == wallet` guard, no per-name toggle.)

### Novelty vs known issues

The program's known-issues list (not shipped in-repo, so cross-checked from the program page) — WEB-1249 (subregistry replacement / orphaned subnames), WEB-1255 (`isV2` grace-window default), QA-01 (role combinations that *error*), SEC-MGR-003/008/010/011, SEC-TXM-002, R2-01..R2-06 — none describes a *successful* `grantRoles` of a resolver role to a subgraph-sourced, unverified, undisclosed beneficiary. In-repo docs (`MIGRATION_CASE_STUDY.md`) never mention manager restoration. In particular this is **not** the general opacity of the `executeByOwner` batch: this finding is a specific third-party privilege grant with an off-chain-derived, never-reconciled beneficiary — a distinct root cause (trust-model inconsistency) and a distinct fix (on-chain read + address display) that no batch-disclosure improvement would address.

### Recommended fix

1. **Verify on-chain before granting:** read `ENSRegistry.owner(namehash(name))` (V1 legacy registry) at plan *and* execution time and only grant if it matches; add the beneficiary as an *input* invariant in `verificationExpectations`, not just a post-state read. The sibling portal already does this on-chain read for display (`apps/portal/src/features/ownership/components/V1NameManagerRecord.tsx:35-56`) — reuse it.
2. **Disclose + opt-in:** render the beneficiary address (and ENS name) per name; make "restore manager" an explicit per-name checkbox defaulting **off** whenever the manager is not the connected wallet/HCA; name the address in the wallet-confirmation dialog.
3. Prefer the V3 model (sync manager to owner) for unwrapped names; at minimum never grant when `registrant ≠ owner` without explicit confirmation.
4. Revoke the `eth-registry:hca` operator approval in a `finally` so it cannot outlive the flow.

## Link to Proof of Concept

<https://gist.github.com/yfxtest/7ca19e23541c64f804cd445f8b849095>

## Proof of Concept

`MGR-1-manager-role-grant.mjs` — Part A is offline and deterministic: it feeds a realistic V1-subgraph `Domain` row for a marketplace name (registrant = buyer `0x1111…`, registry owner = seller `0x2222…`) through the **vendored** derivation (`classifyNames.ts:162-167`) and call builder (`buildRoleGrantCalls.ts:9-27`), then decodes the resulting inner call of the HCA `executeByOwner` batch.

Run:

```bash
npm i viem
node MGR-1-manager-role-grant.mjs
```

Expected (Part A):

```
derived managerAddress:          0x2222222222222222222222222222222222222222
inner call appended to the HCA executeByOwner batch (buildAtomicMigrationBatches.ts:608-614):
  to:    0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2  (ETHRegistry)
  data:  0x7c300586…2222222222222222222222222222222222222222
  decoded: grantRoles(anyId=0x9c0257114eb9…00000000, roleBitmap=0x1000000 (ROLE_SET_RESOLVER), account=0x2222…2222)
```

It shows the app builds `grantRoles(id, ROLE_SET_RESOLVER, seller)` from a subgraph field, that the only user-facing copy is "Approve manager restoration" (address shown nowhere), and that the post-state check (`verifyAtomicMigrationBatch.ts:337-354`) confirms only that the grant landed — never that the beneficiary is (still) the on-chain manager.

**Part B (optional, on-chain confirmation)** — set `SEPOLIA_RPC_URL`, `LABEL`, `MANAGER` and re-run to read `ETHRegistry.hasRoles(id, ROLE_SET_RESOLVER, MANAGER) == true` and `ENSRegistry(V1).owner(namehash) == Graveyard`. Sepolia runbook:

1. Register `mgr1test.eth` on ENSv1 Sepolia (unwrapped) from wallet **A**.
2. `cast send 0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85 "transferFrom(address,address,uint256)" A B $(cast keccak mgr1test)` — **no** `reclaim`. Check `ENSRegistry.owner(namehash("mgr1test.eth")) == A`, `BaseRegistrar.ownerOf == B`.
3. Connect **B** → Migration → migrate `mgr1test.eth`; observe the address-free "Approve manager restoration" step; complete.
4. `SEPOLIA_RPC_URL=… LABEL=mgr1test MANAGER=A node MGR-1-manager-role-grant.mjs` → `hasRoles(id, 1<<24, A) = true`, `ENSRegistry.owner(node) = Graveyard`.
5. From **A**: `cast send <ETHRegistry> "setResolver(uint256,address)" <id> <evilResolver>` succeeds; `UniversalResolver.resolve("mgr1test.eth", addr)` now returns `evilResolver`'s answer.

PoC Gist: <https://gist.github.com/yfxtest/7ca19e23541c64f804cd445f8b849095>
