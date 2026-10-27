# 92097 w a medium manager migration resurrects a removed attacker controlled resolver and re grants a relinquished manager role from a stale v1 snapshot

**Submitted on Sep 7th 2026 at 06:25:59 UTC by @Jornason for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92097
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

ENS Manager's V1-to-V2 migration planner snapshots an indexed `V1Domain` (the query result is cached as fresh for five minutes), builds a migration plan from that snapshot, and then retains both the snapshot's manager address and custom resolver address through execution without ever re-reading live V1 Registry state. A former V1 manager — a prior owner or delegated manager who still controls the V1 Registry node when the victim opens the migration page — plants an attacker-controlled address as the name's custom resolver while still authorized, then simply waits. The victim performs the ordinary migration approvals, which are public on-chain transactions. Observing that public signal, the attacker runs a single `setRecord(node, victim, safeResolver, ttl)` call — an ordinary, still-authorized V1 Registry operation — that hands V1 ownership back to the victim and removes the attacker resolver in one atomic transaction. Live V1 state is now clean: owner = victim, resolver = zero, attacker unauthorized. Yet Manager's retained stale plan still executes two attacker-favorable terminals in the victim's own migration batch: (head 1) the unlocked migration controller registers the victim's V2 name with the attacker-controlled resolver, because the plan's `Migration.Data.resolver` still carries the snapshot address; and (head 2) Manager emits `grantRoles(resource, ROLE_SET_RESOLVER, formerManager)`, re-granting the remotely relinquished manager a persistent V2 resolver role that survives the app's own cleanup, after which the attacker replaces the migrated name's resolver at will. Executed proof on a local Anvil fork of Sepolia pinned at block `11534085`, running calldata produced by the frozen production `buildMigrationPlan()`: a three-case matrix shows the byte-exact stale production batch assigns the attacker resolver and grants the role (pre-submit live state: V1 owner = victim, V1 resolver = zero, attacker role = false), a byte-exact fresh production plan built from the post-handoff state does neither, and the stale batch with only the inner manager grant removed still resurrects the attacker resolver while the role stays false — proving the two heads are independent sinks of one shared root. The production consequence is that any `.eth` name migrating through Manager while a former manager watches the mempool can settle in V2 pointing at an attacker-controlled resolver, and with a persistent attacker role on the name, at the exact moment the application tells the victim their names and records are being migrated and restored automatically.

## Vulnerability Details

At audited public head `1c9b47f18fcddd2e864dfe385c4171061c9811ae` (tree `fd2fa034e987dd81c4a170b3e8370e6edf2a3376`, branch `audit-comp-ready`), the required invariant is: a custom resolver preserved by migration must be the live resolver at submission time (or its exact address must be explicitly reviewed and approved by the current registrant), and Manager may preserve V1 resolver authority only for a principal that is still the live V1 manager when the permanent V2 grant is submitted. Manager violates both halves of that invariant from one shared root: the indexed V1-domain snapshot is never reconciled against live V1 Registry state at execution.

#### Attack setup — the attacker is a third party, not the victim

The victim owns an unwrapped, unexpired `.eth` second-level name (BaseRegistrar NFT holder) and performs only the ordinary Manager migration flow. The attacker is a former manager of the name's V1 Registry node — a prior owner who was reclaimed, or any delegated manager — who still owns the V1 node when the victim opens the migration page. While still authorized, the attacker calls V1 `setResolver(node, attackerResolver)` once, planting an attacker-controlled resolver address. Nothing about this preparation is anomalous: setting a custom resolver is ordinary manager behavior, and the V1 Registry state is public. No stolen credential, no victim-local state edit, no malicious contract approval trick, and no attacker-supplied calldata to the victim is required at any point. Manager's own migration list then ingests this state: `useV1Names` caches the indexed V1-domain query with a five-minute `staleTime` (`apps/manager/src/features/migration/hooks/useV1Names.ts:22`), and the subgraph query materializes `resolver { address }` and `owner { id }` into the `V1Domain` (`apps/manager/src/features/migration/service/v1SubgraphClient.ts:22`) with no execution-time RPC binding.

#### Root cause: the snapshot's resolver and manager are conserved through execution, live state is never re-read

The migration plan does not conserve the authorization/state tuple `(node, live V1 owner, live V1 resolver, snapshot revision)`. Stage 1 — classification copies the snapshot into the plan: `classifyName` sets `v1ResolverAddress = domain.resolver?.address ?? null` (`packages/migration/src/service/classifyNames.ts:137`), selects the `keep-v1` strategy for every custom resolver that is not a known public resolver (`packages/migration/src/service/classifyNames.ts:84-86`), and copies `domain.owner.id` into `managerAddress` whenever the V1 node owner differs from the registrant (`packages/migration/src/service/classifyNames.ts:162-167`). Stage 2 — plan construction bakes both values into calldata: `resolverFor` returns the retained custom resolver for `keep-v1` names (`apps/manager/src/features/migration/service/encodeMigration.ts:24`, keep-v1 branch at :34-40); `buildAtomicMigrationBatches` places that address into `Migration.Data` via `createMigrationData({ ..., resolver: expectedResolver })` (`apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:481-492`) and, when `classified.managerAddress` is set, appends a `manager-role-grant` inner execution plus a matching post-state expectation (`apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:607-624`); `buildRoleGrantCall` ABI-encodes `grantRoles(resource, ROLE_SET_RESOLVER = 1 << 24, managerAddress)` (`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:11-27`). Stage 3 — execution reuses the built plan: the page starts migration with the plan already held by the gas estimate (`apps/manager/src/features/migration/pages/MigrationPage.tsx:275-282`, `plan: gasEstimate.plan`), the UI state machine captures that plan and sets first execution's `reconcileBeforeSubmit` to `false` (`apps/manager/src/features/migration/state/migrationUi.machine.ts:150-163`), and the execution-time freshness check `assertMigrationApprovalPlanCurrent` compares only the sorted set of missing approval keys against the preview — it never compares the retained resolver or manager principal against `V1Registry.resolver(node)` / `V1Registry.owner(node)` (`apps/manager/src/features/migration/service/migrationService.ts:367-391`). Subsequent batches are rebuilt from the retained classifications (`migrationService.ts:886-901`, with `remaining` initialized from `params.plan.classified` at :1052), and the flow submits the approvals and then executes the retained authorization plan (:1169-1182). Finally, post-state verification accepts the stale values as the definition of success: the `name-resolver` expectation passes when the live V2 resolver equals the planned (stale) resolver (`apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:263-275`), and the `manager-role` expectation does the same for the stale grantee — so the app itself reports the poisoned migration as fully verified.

#### The attack sequence and the ordering window

1. The attacker (still the live V1 node owner) plants `resolver = attackerResolver`. The victim's BaseRegistrar registration is unaffected.
2. The victim opens Manager's migration page; Manager snapshots the attacker as manager and the attacker resolver as a custom resolver to preserve, and builds the plan.
3. The victim starts migration and mines the normal approval transactions — including the public `eth-registry:hca` operator approval — an ordinary, publicly observable on-chain signal that a migration batch is imminent.
4. Watching for that signal (a trivially automatable mempool/chain watcher over the victim's approval transaction), the attacker mines one V1 `setRecord(node, victim, safeResolver, ttl)` call. Under the canonical `ENSRegistry`, `setRecord` authorizes through `setOwner` while the attacker is still the owner, changes the owner, and then writes resolver and TTL internally without a second authorization check — intended atomic Registry behavior. Live V1 state now has neither the attacker as owner nor the attacker resolver.
5. Manager submits the retained batch; V2 receives the stale resolver in `Migration.Data.resolver` and the stale manager in `grantRoles`.

The attacker must win the ordering window between the victim's approval transaction and the victim's outer migration batch confirmation. That window is user-paced — it spans at least one further wallet confirmation, plus any gas-funding wait — so it is typically minutes, not milliseconds, and the attacker's single `setRecord` is an ordinary authorized-until-handoff call that can be fired automatically the moment the approval is observed. This report does not claim the window is unlimited: if the victim's batch confirms first, the stale plan executes against a live state where the attacker is still manager and the resolver is still planted, which is the benign product-intended outcome; the attacker's handoff must land in between.

#### Head 1 — resolver sink: `Migration.Data.resolver`

`UnlockedMigrationController` registration consumes `md.resolver` directly: the executed fork terminal shows `V2 ETHRegistry.getResolver(label)` settling to the attacker-controlled address solely because Manager submits the stale plan. The locked `CANNOT_SET_RESOLVER` branch instead reads live V1 state and is not part of this finding; the victim's unwrapped name takes the unlocked direct route.

#### Head 2 — role sink: `grantRoles(resource, ROLE_SET_RESOLVER, formerManager)`

The same retained plan carries the snapshot's `managerAddress` into a persistent V2 role grant. After the app's own cleanup revokes only the temporary HCA approval, the former manager — genuinely unauthorized in V1 since before the batch — holds `ROLE_SET_RESOLVER` on the migrated name's V2 resource and successfully calls V2 `setResolver`, replacing the name's resolver with an arbitrary attacker address (executed: resolver changes `0xaD3006320EFFd21CC7A758CAbC884f322D574d18` to `0x43EE991F5C681F55216Ac7Cb8CF3FB3883Cd787D`, receipt success).

#### The two heads are independent — the `STALE_NO_MANAGER_GRANT` control

Both heads share the carrier (stale `V1Domain` snapshot, public approval signal, handoff window) but not the sink. The fixed-fork matrix executes three cases from the identical attainable pre-submit state (V1 owner = victim, V1 resolver = zero, former manager V2 role = false): `STALE_PLAN` (byte-exact production stale outer call) → V2 resolver = attacker-controlled address, former manager role = `true`; `FRESH_PLAN` (byte-exact production plan rebuilt from the post-handoff state) → V2 resolver = fresh resolver `0xaD30...4d18`, role = `false`; `STALE_NO_MANAGER_GRANT` (the stale outer call with only the inner manager grant filtered out) → V2 resolver = attacker-controlled address, role = `false`. The resolver resurrection survives removal of the role grant, and the role grant is absent from the fresh plan — so revalidating only `managerAddress` does not fix head 1, and revalidating only the resolver does not fix head 2. Both are consequences of the shared unreconciled snapshot, presented here as one report with two heads.

#### The migration UI never displays the resolver address

At no point does the migration flow show the victim which resolver (or which manager) will be carried into V2. The selection screen says only "Your names, text records, and addresses will migrate automatically" (`apps/manager/src/features/migration/components/SelectNamesStepSelectionOptions.tsx:122`); the atomic-batch wallet step says only "Migrate the selected names and restore their records." (`apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:107`); and the approval step for the manager grant says only "Approve manager restoration" / "Restore the existing managers for your names." (`WalletConfirmationStepsDialog.tsx:81-89`) — after the attacker's handoff, the cached grantee is no longer an "existing manager" in live state at all. A full read of the migration feature's components and pages confirms no resolver or manager address is rendered anywhere in the flow (the only address literal in the feature's components is a test fixture). The victim therefore cannot review the decisive value the app is about to bind on-chain, and the UI copy affirmatively describes an automatic restore-the-existing-state operation while the app in fact restores a resolver the former manager has already removed.

#### Runtime proof summary (fixed Sepolia fork, production calldata)

The PoC (full steps in the Proof of Concept section) calls the frozen production `buildMigrationPlan()` directly with a representable stale `V1Domain` snapshot (manager = former manager, resolver = attacker address) and a fresh post-handoff snapshot (manager = victim, resolver = null) — no migration calldata is hand-authored — and executes the resulting batches on a local Anvil fork of Sepolia pinned at block `11534085` (hash `0x31a09581824c5daafbe560b49a5c8954ea360a00120a2a8e769f89483b38505e`), reached through an unauthenticated read-only archive RPC. The stale plan carries `Migration.Data.resolver = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8` (outer calldata hash `0xc8c5880f561a704f574739a1cca8714f37a06bd517f5b8acc214d374d5bdb69b`, inner phases `[migrate, manager-role-grant]`) and `grantRoles(resource = 15074162868894410784243258703647169117691534494726043500355491667019361681408, roleBitmap = 16777216, account = 0x7099...79C8)`; the fresh plan carries resolver `0xaD3006320EFFd21CC7A758CAbC884f322D574d18` and no manager grant (outer calldata hash `0x9e575cf8f5d160080bf0e5c30eb1b14432f32e1cb0b6d62f05cde116bfddc181`). All six decisive assertions hold: (1) pre-submit stale V1 owner is the victim; (2) pre-submit stale V1 resolver no longer equals the attacker address; (3) the stale submission restores that address as the V2 resolver; (4) a fresh plan from the same post-handoff state excludes it; (5) removing only the manager grant does not remove the resolver terminal; (6) the no-grant control leaves the former manager's V2 role false. For head 2, the stale batch plus the app's own cleanup leaves the former manager with `ROLE_SET_RESOLVER`, and the former manager's subsequent V2 `setResolver` succeeds; with only the grant removed, the same call reverts. The production plan artifact regenerates byte-identical from a clean process (SHA-256 `ed61311e1ca8587e83889b11f23e803e414182a76150f0ac994a0219db95a22c`), and the fork artifacts reproduce identically in calldata, pre-states, terminals, transaction labels/status/gas, and assertions (only local Anvil transaction hashes differ across processes). A second clean-process rerun of all three stages at packaging time on 2026-09-07 reproduced all of the above.

#### Boundary with the migration-Retry profile-replay family (#90020/#89766)

Reports #90020 and #89766 concern Manager's migration Retry path replaying stale `MigrationPlan.profiles` values (profile record content re-applied on a retry after the first flow). This finding is different in sink, flow, and minimum fix: here the first normal migration execution assigns the V2 resolver at registration time via `Migration.Data.resolver` and creates a persistent V2 role via `grantRoles`, both from a snapshot whose V1 owner/resolver identity is never re-read; the minimum fix is to reconcile the live V1 Registry owner and resolver immediately before each batch (or to bind and display the exact carried addresses), not to refresh profile record values on retry. The shared theme is stale plan state, but no fix limited to the retry/profile path addresses either terminal demonstrated here.

#### Evidence boundaries (full honesty list — nothing implied beyond it)

1. The harness calls production `buildMigrationPlan()` directly with a representable `V1Domain` snapshot; it does not automate the browser, the subgraph response, the wallet UI, or real mempool timing. The race is demonstrated deterministically in the fixed-fork sequence and has not been measured against a production browser and public mempool.
2. The attacker resolver address carries an EIP-7702 delegation designator (`0xef0100` bytecode prefix at the pin block): it is an attacker-CONTROLLED resolver address, not an empty-code EOA. The security assertion is control of the resolver address; any contract resolver the attacker controls (or an EIP-7702-delegated account they control) serves identically.
3. All writes were local-Anvil only against the pinned Sepolia fork at block `11534085`; no public Sepolia transaction was broadcast. The fork establishes the prerequisite state with real contract calls and Anvil account impersonation used purely as a signing harness (no storage writes, no mocked migration contracts); the fixture name is not claimed to be a real victim.
4. The victim remains the V2 owner and can repair the resolver (and revoke the role through Portal) after discovery; the demonstrated impact is the attacker-controlled assignment and role existing from settlement until discovery, not permanent irrecoverability.
5. The attack requires the former manager to win the approval→batch ordering window described above.

#### Recommended remediation

1. Immediately before building or submitting each migration batch, read the live V1 Registry `owner(node)` and `resolver(node)` for every selected name. If either differs from the snapshot, abort for explicit re-review; drop the manager grant when the current V1 owner equals the registrant, and drop (or re-confirm) a `keep-v1` resolver that is no longer live.
2. Display and bind the exact resolver address and the exact manager grantee address in the confirmation UI, and make that binding expire on any V1 owner/resolver event for the selected nodes.
3. Extend the execution-time reconciliation (`assertMigrationApprovalPlanCurrent`) beyond approval keys to the authorization principals and resolver identities that justify `manager-role-grant` and `keep-v1` plan elements.
4. Regression coverage: a former manager atomically changing V1 owner and resolver after the approval but before the outer batch must cause the app to refuse the old plan or rebuild from live state; assert the stale address receives no V2 role and the stale resolver is not registered.

## Impact Details

Selected impact row (Websites & Apps, Critical): **"Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow"**

This is the literal resolver limb of that row: the executed terminal proves that the victim's ordinary V1→V2 migration assigns the name's V2 resolver to an attacker-controlled address — `V2 ETHRegistry.getResolver(label)` settles to `0x70997970C51812dc3A010C7d01b50e0d17dc79C8` solely because Manager submits the stale plan, while the fresh plan from the same live state assigns the product-intended fresh resolver. Every resolution of the migrated name from settlement until discovery is answered by a resolver the attacker controls: the attacker can point the name's address record at an attacker address, serve attacker content, or break resolution, under the victim's own name. Head 2 compounds the assignment with persistence: Manager creates `ROLE_SET_RESOLVER` for the attacker on the name's V2 resource (capability false → true, commissioned by the app's own transaction), the app's cleanup removes only its temporary HCA approval, and the attacker then replaces the resolver at any later time with one ordinary call — demonstrated — so even a victim who notices and fixes the initial resolver must also discover and revoke a hidden role the app never displayed.

The attacker is a third party — a former manager acting while still authorized in V1 — and the victim performs only the ordinary, prompted migration flow (open migration page → approve → confirm batch). The planting requires no victim interaction at all, and the trigger the attacker relies on is the victim's own public approval transaction. The decision surface gives the victim no opportunity to catch the substitution: the resolver address and the grantee address appear nowhere in the migration UI, the copy promises that names, records, and addresses "migrate automatically" and that the batch will "migrate the selected names and restore their records", and the approval step speaks of restoring "existing managers" when the cached grantee has already relinquished that status on-chain. Post-state verification then reports success because the app's own expectations were built from the same stale plan.

Honest limits, stated plainly: the victim remains the V2 owner throughout — this is not NFT theft and not permanent bricking — and can repair the resolver pointer and revoke the role after discovery; the demonstrated impact is the attacker-controlled assignment and persistent role from settlement until discovery, plus the demonstrated unauthorized resolver replacement before detection. Repairability does not prevent the assignment, and it is not disqualifying in this program's own adjudication practice: escalated report #91997 in this same competition passed triage despite the affected state being owner-editable after the fact. The attack requires the attacker to win the approval→batch ordering window (typically minutes, automatable against the public approval signal, but not guaranteed per attempt), and it requires a name whose V1 node a former manager still controls at preview time — a normal state for names that were ever co-managed, traded, or operated by a prior owner. The planted resolver address in the executed proof is an EIP-7702-delegated attacker account; the claim is an attacker-controlled resolver address, which any attacker contract or delegated account satisfies. Within those limits, the demonstrated behavior is precisely the selected Critical row: a v1-to-v2 migration that assigns the name's resolver to an attacker.

## References

* In-scope asset: `apps/manager` (Manager app, Websites & Apps): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* Five-minute V1-names cache (`staleTime: 5 * 60 * 1000`): <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts>
* Subgraph query materializing `resolver { address }` / `owner { id }` without an execution-time RPC binding: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/v1SubgraphClient.ts>
* Snapshot → `v1ResolverAddress` / `keep-v1` / `managerAddress` classification: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts>
* `resolverFor` returning the retained custom resolver: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/encodeMigration.ts>
* `Migration.Data.resolver` placement and the `manager-role-grant` inner execution + expectations: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts>
* `grantRoles(resource, ROLE_SET_RESOLVER, managerAddress)` encoding: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts>
* Execution started with the already-built gas-estimate plan: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/pages/MigrationPage.tsx>
* Plan captured with `reconcileBeforeSubmit: false`: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/state/migrationUi.machine.ts>
* Approval-keys-only freshness check and batch rebuild from retained classifications: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts>
* Post-state verification accepting equality with the planned stale resolver as success: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts>
* Wallet confirmation copy ("Approve manager restoration" / "Migrate the selected names and restore their records."): <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx>
* Selection copy ("Your names, text records, and addresses will migrate automatically"): <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/SelectNamesStepSelectionOptions.tsx>
* Official competition scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Canonical V1 `ENSRegistry` (the `setRecord` atomic owner+resolver handoff used by the attacker): <https://etherscan.io/address/0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e#code>

Public-state refresh on 2026-09-07: `git ls-remote https://github.com/immunefi-team/audit-comp-ens` shows `audit-comp-ready` still at `1c9b47f18fcddd2e864dfe385c4171061c9811ae` (the exact revision executed by the PoC) and a single pull ref — PR #1 (`d35fbdf827f37d7b22ad7cd347342ad425250163`, a Hono dependency bump touching only package manifests and the lockfile), unrelated to the migration planning, classification, batch-building, role-grant, reconciliation, verification, or UI sources cited above. The competition's public Known Issues list (dated 2026-08-14) contains no entry covering migration snapshot freshness, resolver restoration, or manager-role grants; its three migration-adjacent entries concern subnames missing from the migration list (WEB-390), subname records not displaying after migration (WEB-806), and a profile-edit 500 for unmigratable names (WEB-1303) — none touches this path. Private-report overlap remains unknowable and is not claimed either way.

## Link to Proof of Concept

<https://gist.github.com/Jornason/39b5b437f6352fc1f4ba2b849ea0f65c>

## Proof of Concept

## Proof of Concept

### What this PoC proves

The linked secret Gist bundle (<https://gist.github.com/Jornason/39b5b437f6352fc1f4ba2b849ea0f65c>) preserves an executed end-to-end reproduction (run tag q7m4v9) against the frozen public competition head `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. It calls the frozen production `buildMigrationPlan()` directly with a representable stale `V1Domain` snapshot (manager = former manager `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`, resolver = the same attacker-controlled address) and a fresh post-handoff snapshot (manager = victim, resolver = null); captures both production migration batches without hand-authoring any calldata; and executes three cases on a local Anvil fork of Sepolia pinned at block `11534085` (hash `0x31a09581824c5daafbe560b49a5c8954ea360a00120a2a8e769f89483b38505e`, chain ID 11155111) from the identical attainable pre-submit state (V1 owner = victim `0x0454e152198A6f7e488DC582957aD0CD94FF6478`, V1 resolver = zero address, former manager V2 role = false, HCA approval present). It proves the byte-exact stale production batch registers the victim's V2 name with the attacker-controlled resolver and re-grants the former manager `ROLE_SET_RESOLVER`; the byte-exact fresh production batch does neither; and the stale batch with only the inner manager grant removed still resurrects the attacker resolver while the role stays false. A separate head-2 run on the same carrier proves the former manager, after the app's own cleanup, successfully replaces the migrated name's V2 resolver (receipt success; with the grant removed, the same call reverts).

All writes are local to a disposable Anvil fork. No public Sepolia transaction is broadcast, no production user, site, or deployed contract is contacted for writes or modified, the only remote input is an unauthenticated read-only archive RPC, and every account is a public Anvil deterministic test account.

### Prerequisites

* A clean checkout of the official repository at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae` with locked dependencies installed (`corepack enable && pnpm install --frozen-lockfile`; the completed runs used pnpm with Node.js v22 on macOS arm64).
* Foundry (`anvil`; the completed runs used 1.5.1-stable), `shasum`.
* A Sepolia archive RPC URL (the completed runs used the unauthenticated read-only `https://sepolia.gateway.tenderly.co`) and one free loopback port.

### Step-by-step reproduction

1. Download the secret Gist files into one directory and verify the manifest in the Gist root:

```bash
shasum -a 256 -c SHA256SUMS.txt
```

2. Prepare the frozen checkout (no tracked file is modified; the bundle's tests live outside the repository and are selected through the bundle's own vitest config):

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git ens-audit-q7m4v9
git -C ens-audit-q7m4v9 checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae
(cd ens-audit-q7m4v9 && corepack enable && pnpm install --frozen-lockfile)
```

3. Regenerate the production plan artifact from a clean process (pure computation, no RPC):

```bash
export FROZEN_DIR="$PWD/ens-audit-q7m4v9" BUNDLE_DIR="$PWD"
VITEST="$FROZEN_DIR/apps/manager/node_modules/.bin/vitest"
H2W6S9_PLAN_OUTPUT="$PWD/plan.regen.json" \
  "$VITEST" run --config "$BUNDLE_DIR/vitest.config.q7m4v9.ts" custom-resolver-plan-q7m4v9
shasum -a 256 plan.regen.json
# expected: ed61311e1ca8587e83889b11f23e803e414182a76150f0ac994a0219db95a22c (byte-identical to the included artifact)
```

4. Execute the three-case fixed fork (the test resets a plain local Anvil onto the pinned fork itself):

```bash
anvil --port 8563 --silent &
H2W6S9_LOCAL_RPC="http://127.0.0.1:8563" \
H2W6S9_ARCHIVE_RPC="https://sepolia.gateway.tenderly.co" \
H2W6S9_PLAN_INPUT="$BUNDLE_DIR/production-plan-custom-resolver-q7m4v9.json" \
H2W6S9_FORK_OUTPUT="$PWD/fork.regen.json" \
  "$VITEST" run --config "$BUNDLE_DIR/vitest.config.q7m4v9.ts" custom-resolver-fixed-fork-q7m4v9
```

5. Execute the head-2 remote-manager carrier and its grant-removed control:

```bash
Q7M4V9_LOCAL_RPC="http://127.0.0.1:8563" \
Q7M4V9_ARCHIVE_RPC="https://sepolia.gateway.tenderly.co" \
Q7M4V9_PLAN_INPUT="$BUNDLE_DIR/production-plan-remote-manager-q7m4v9.json" \
Q7M4V9_REMOTE_OUTPUT="$PWD/remote.regen.json" \
  "$VITEST" run --config "$BUNDLE_DIR/vitest.config.q7m4v9.ts" remote-manager-role-regrant-q7m4v9
```

Expected output: each stage reports `1 passed`; stage 3 prints `H2W6S9_PRODUCTION_PLAN_PASS` with `staleResolver` = `0x70997970C51812dc3A010C7d01b50e0d17dc79C8` and `freshResolver` = `0xaD3006320EFFd21CC7A758CAbC884f322D574d18`; stage 4 prints `H2W6S9_THREE_CASE_FIXED_FORK_PASS` with `noGrantResolver` = the attacker address and `noGrantRole` = `false`; stage 5 prints `Q7M4V9_REMOTE_MANAGER_CARRIER_PASS`. The regenerated fork artifacts match the included references in calldata, pre-states, terminal states, transaction labels/status/gas, and decisive assertions; local Anvil transaction hashes differ per process and are not a reproducibility requirement. A full clean-process rerun of all three stages was repeated at packaging time on 2026-09-07 and reproduced all of the above (plan artifact byte-identical).

### Executed three-case matrix (head 1 + independence control)

All three cases begin from the same contract state: the victim holds the BaseRegistrar NFT of the fixture name (`zesty-gecko.eth`, node `0x000805a2b5d7e2c469758cabc03b5e9b2d59f062cbbc6c6479e6496868d53ace`); the former manager is seeded as V1 node owner via a real `reclaim` and plants the attacker resolver via a real `setResolver`; the production HCA is deployed and the production approval calls are mined; then the attacker runs the real one-call handoff `setRecord(node, victim, zeroAddress, ttl)`. Immediately before submission: V1 owner = victim, V1 resolver = zero, former manager V2 role = false.

| Case                     | Submitted batch                                                           | V2 owner | V2 resolver                       | Former manager `ROLE_SET_RESOLVER` | Result                               |
| ------------------------ | ------------------------------------------------------------------------- | -------- | --------------------------------- | ---------------------------------- | ------------------------------------ |
| `STALE_PLAN`             | Byte-exact production stale outer call (hash `0xc8c5880f…bdb69b`)         | victim   | attacker-controlled `0x7099…79C8` | `true`                             | Positive terminal reproduced         |
| `FRESH_PLAN`             | Byte-exact production post-handoff plan (hash `0x9e575cf8…ddc181`)        | victim   | fresh `0xaD30…4d18`               | `false`                            | Negative control excludes attacker   |
| `STALE_NO_MANAGER_GRANT` | Stale call with only the manager grant removed (hash `0xa5ee34f7…e5cb5c`) | victim   | attacker-controlled `0x7099…79C8` | `false`                            | Resolver sink survives independently |

All six decisive assertions in the fork artifact are `true`: pre-submit stale V1 owner is the victim; pre-submit stale V1 resolver no longer equals the attacker address; the stale submission restores that address as the V2 resolver; a fresh plan from the same post-handoff state excludes it; removing only the manager grant does not remove the resolver terminal; the no-grant control leaves the former manager's V2 role false.

### Executed head-2 terminal (role re-grant → attacker resolver replacement)

The stale plan's inner execution index 3 decodes as `grantRoles(resource = 15074162868894410784243258703647169117691534494726043500355491667019361681408, roleBitmap = 16777216, account = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8)` (outer calldata hash `0x9b764ce317990355a5ede79c525d98b7f751e8081ba612f72951a4060f672a21`). After the remote `setOwner` handoff, the stale batch, and the app's own cleanup (temporary HCA approval revoked, verified `false`), the former manager's V2 `setResolver` succeeds and the name's resolver changes from the fresh production resolver `0xaD3006320EFFd21CC7A758CAbC884f322D574d18` to the attacker's `0x43EE991F5C681F55216Ac7Cb8CF3FB3883Cd787D`. In the grant-removed control — same carrier, only the manager-grant inner execution filtered out — migration and cleanup still succeed, the role reads `false`, and the same `setResolver` reverts with the resolver unchanged. This isolates the unauthorized terminal to Manager's stale grant: the role is created by the in-scope app transaction, not retained from V1.

### Honest simplifications (nothing implied beyond this list)

1. The harness calls production `buildMigrationPlan()` directly with a representable stale/fresh `V1Domain` snapshot; it does not automate the browser, subgraph response, wallet UI, or real mempool timing.
2. The attacker resolver address carries an EIP-7702 delegation designator — it is an attacker-CONTROLLED resolver address, not an empty-code EOA.
3. All writes are local-Anvil only against the pinned Sepolia fork at block `11534085`; account impersonation is used purely as a signing harness (no storage edits, no mocked contracts); the fixture name is not claimed to be a real victim.
4. The victim remains the V2 owner and can repair the resolver and revoke the role after discovery.
5. The attack requires winning the approval→batch ordering window; success is not guaranteed per attempt.

### Bundle contents and integrity

| File                                                                       | Content                                                                                                                                    |
| -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `custom-resolver-plan-q7m4v9.poc.test.ts`                                  | Production-plan capture against frozen `buildMigrationPlan()` (SHA-256 `04d801988370b9fdb0942ee9df329ff9b0c34eb9df6929a5d46ea95e70fe4880`) |
| `custom-resolver-fixed-fork-q7m4v9.poc.test.ts`                            | Three-case fixed-fork execution + six decisive assertions (SHA-256 `a8e6ad7f0f10483e48f86af1f578adca90ac8e58c659836aa6f7804c92721c15`)     |
| `remote-manager-role-regrant-q7m4v9.poc.test.ts`                           | Head-2 carrier + grant-removed control (SHA-256 `084cc9190403e2c00b66b81f65c0fb8fb83437e075677284ed21ad5b1f0e3fb2`)                        |
| `production-plan-custom-resolver-q7m4v9.json`                              | Stale + fresh production plans, decoded inner calls (SHA-256 `ed61311e…db95a22c`)                                                          |
| `fork-results-custom-resolver-q7m4v9.json`                                 | Three-case terminals + assertions (SHA-256 `08f57981…f99b0cc`)                                                                             |
| `production-plan-remote-manager-q7m4v9.json`                               | Head-2 plan artifact (SHA-256 `6527500e…401ece`)                                                                                           |
| `fork-results-remote-manager-q7m4v9.json`                                  | Head-2 attack + control terminals (SHA-256 `e7c50aea…5b32d`)                                                                               |
| `fixed-fork-verdict-q7m4v9.md`, `remote-manager-carrier-verdict-q7m4v9.md` | Verdict notes (calldata gate, independence proof, decision surface, boundaries)                                                            |
| `vitest.config.q7m4v9.ts`                                                  | Generic runner config (`FROZEN_DIR` / `BUNDLE_DIR`)                                                                                        |
| `README-q7m4v9.md`                                                         | Full reproduction guide and environment notes                                                                                              |
| `SHA256SUMS.txt`                                                           | Integrity manifest covering every bundle file (verifies in the Gist root)                                                                  |

### Safety limits

* The production applications, deployed ENS contracts, and public Sepolia are never contacted for writes; everything executes on disposable local Anvil instances, with the only remote input being read-only archive RPC calls.
* All accounts are public Anvil deterministic test accounts; no real wallet key, user credential, or production data is required or included.
* The frozen competition checkout is never modified: the bundle's tests live outside the repository and are selected through the bundle's own vitest config; no tracked file is changed.
* No phishing, no social engineering, no third-party service interaction beyond public read-only RPC; the demonstration uses only local processes.
