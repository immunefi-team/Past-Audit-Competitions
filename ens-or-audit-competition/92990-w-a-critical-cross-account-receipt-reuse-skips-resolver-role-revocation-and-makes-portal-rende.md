# 92990 w a critical cross account receipt reuse skips resolver role revocation and makes portal render an attacker controlled ens address

**Submitted on Sep 13th 2026 at 17:44:13 UTC by @xenomxbr for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92990
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Portal registry-role editor reuses a fixed revoke transaction ID across different accounts and accepts status solely by that ID. After the victim successfully revokes account `A` and ordinarily dismisses the success modal, `A`'s successful transaction actor remains live. When the victim next removes account `B`, Portal consumes `A`'s success as completion of `B`'s revoke, renders the `B` flow as Done, and invokes no revoke mutation for `B`. The retained `ROLE_SET_RESOLVER` authority can then be used by `B` to redirect the victim-owned ENS name to an attacker-controlled address; the PoC proves that the deployed Universal Resolver returns that address and that Portal's real records-query and profile-card paths consume and render it.

## Vulnerability Details

### Broken security invariant

Transaction completion evidence must be bound to the exact logical security operation whose postcondition it is used to prove. For registry-role revocation, the minimum identity includes the registry, edited account, role delta, connected wallet, chain, flow generation, and transaction generation.

Portal instead reduces all registry-role revoke flows to one operation-only ID:

```ts
const REVOKE_TX_ID = 'tx-edit-registry-roles-revoke'
```

`RegistryEditUserSheet.buildModalTransactions()` builds the current account's descriptor with that fixed ID. The account and role delta enter the transaction intent, but not the identifier used to retrieve completion status.

On ordinary modal dismissal, `TransactionModal` closes the presentation and clears only an errored transaction:

```ts
if (!open) {
  handleClose()
  if (txState?.error) {
    clearTransaction()
  }
}
```

The shared transaction manager stores actors as `Map<transactionId, actor>`. Portal's status helper then performs only:

```ts
const actor = activeTransactionsMap.get(transactionId)
```

No account, registry, role, request, wallet, chain, or generation comparison follows. Therefore, a valid success for `A` becomes accepted completion evidence for the later current descriptor belonging to `B`.

### Reproduced sequence

1. Victim `V` administers one UserRegistry. Separate delegates `A` and `B` initially hold root `ROLE_SET_RESOLVER`.
2. `V` selects `A` through the real `RegistryRolesTable` row action and confirms Remove user.
3. The real revoke flow starts once for `A` and reaches clean success.
4. `V` dismisses the success modal with its ordinary X button and closes `A`'s edit sheet.
5. The successful `A` actor remains in the app-wide transaction-manager map under `tx-edit-registry-roles-revoke`.
6. `V` selects `B` through the same real table and confirms Remove user.
7. The current descriptor is for `B`, but status lookup returns the successful actor created for `A`.
8. Portal immediately renders the final Done action for `B`; no Start action is required.
9. Clicking Done closes and clears the current `B` flow.
10. The instrumented real mutation hook records exactly one revoke for `A` and zero revokes for `B`.
11. `B` retains root `ROLE_SET_RESOLVER` after Portal has represented the explicit removal as complete.
12. On an exact loopback fork, `B` deploys/configures a PermissionedResolver and the exact UserRegistry accepts `B.setResolver`.
13. Direct resolution and the deployed Universal Resolver return `B`'s attacker-controlled address while `V` remains owner.
14. Portal's actual `getRecords` path returns `B` as coin type 60.
15. The actual `NameProfileCard` renders `B` through the real `CoinRecord` and `EntityActionCopy` components.

### Causation control

A neighboring negative control explicitly clears the completed `A` actor before opening `B`. Portal then renders Start, requires the normal wallet flow, and records a fresh revoke for `B`. The actor's survival and cross-subject ID collision are the causal difference.

### Attacker model and prerequisites

* `V` can administer and revoke roles in the registry.
* `A` and `B` are different legitimate delegates before the sequence.
* `B` controls only its own legitimate delegate key; no victim key, device, wallet, RPC, or provider compromise is required.
* In one Portal application lifetime, `V` revokes `A`, ordinarily dismisses the success modal, closes `A`, and then removes `B`.
* `B` exercises the authority that Portal failed to revoke.

The security boundary is the attempted revocation: `B` is legitimate before it and must no longer retain the role after Portal reports the removal complete.

### Affected in-scope source

* `apps/portal/src/features/registry/components/v2/RegistryEditUserSheet.tsx`
* `apps/portal/src/features/transaction-manager/components/TransactionModal.tsx`
* `apps/portal/src/features/transaction-manager/utils/getStatus.ts`
* `apps/portal/src/features/transaction-manager/components/TransactionsOverviewContent.tsx`
* `packages/transaction-manager/src/providers/transactionManager.ts`

The ENS contracts are used only to demonstrate the consequence and are not alleged to be vulnerable.

## Impact Details

### Executed security result

The PoC proves the following chain:

```
valid success for A
→ accepted as completion of B's revoke
→ no B revoke submission
→ B retains ROLE_SET_RESOLVER
→ exact UserRegistry accepts B's resolver assignment
→ Universal Resolver returns B's address
→ Portal getRecords consumes B's address
→ real NameProfileCard renders B's address
```

The result is not merely a stale badge or incorrect internal value. A security-sensitive authorization removal is skipped, the retained authorization is successfully exercised, canonical ENS resolution changes, and the in-scope application renders the attacker-controlled address.

### Proposed severity and selected impact

Proposed severity: **Critical**.

Selected impact:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient

The competition expressly lists “resolving or rendering an ENS name to an attacker-controlled address” as an example. The exact PoC reproduces that example at the application layer.

No separate downstream funds transfer or victim transaction approval is claimed. If the introductory “so that” clause is interpreted as a mandatory additional condition even for the listed resolving/rendering example, final severity may be reduced. This is a severity-interpretation boundary; it does not affect the proven cross-subject completion, skipped revocation, retained resolver authority, exact resolution, or Portal rendering.

### Relationship to public known issue R3-07

R3-07 describes a failed same-subject attempt followed by a successful same-ID retry, where the `completedTelemetry` guard skips archive/history/telemetry side effects and leaves a pending record.

This finding has a different producer and consumer: a clean successful actor for account `A` remains in the live `transactions` map and is consumed by Portal's `getStatus` for a later account `B`. Its postcondition is also different: `B` is falsely shown Done, the `B` revoke hook is never invoked, security authority survives, and that authority is exercised to produce attacker-controlled ENS resolution rendered by Portal.

The natural repairs are orthogonal. Repairing R3-07's terminal telemetry/archive guard does not bind Portal's live actor to `B`; binding or clearing the live Portal actor does not repair R3-07's skipped archival side effects. If both are nevertheless abstracted to reused transaction IDs, the competition rules expressly keep eligible “new consequences of a listed root cause that materially change its severity.” This report proves such a consequence beyond R3-07's listed Medium bookkeeping result.

### Remediation

Bind every live actor and status lookup to an immutable logical-operation identity. For this flow, the identity must include at least `(registry, account, role delta, wallet, chain, flow generation, transaction generation)`.

Defensible implementations include:

* generate a unique actor ID for every logical operation and generation;
* store subject/request metadata with the actor and reject mismatched status evidence;
* scope actors to the owning flow generation; and
* clear successful actors when their owning security flow is dismissed.

Do not repair only the visual Done state. The invariant must guarantee that a descriptor for one logical security operation can never consume receipt/status evidence from another.

## References

* ENS competition scope and impacts: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* ENS competition information and known issues: <https://immunefi.com/audit-competition/audit-competition-ens/information/>
* ENS competition audited revision and environment: <https://immunefi.com/audit-competition/audit-competition-ens/resources/>
* Registry roles library used by the consequence proof: <https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/registry/libraries/RegistryRolesLib.sol>
* PermissionedRegistry semantics used by the consequence proof: <https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/registry/PermissionedRegistry.sol>

The linked secret Gist contains the canonical report, two runnable PoCs, isolated Vitest configuration, execution provenance, README, and SHA-256 manifest.

## Link to Proof of Concept

<https://gist.github.com/NasserddineB/2b2dd3ef3de5a59bb8f35bc0a6784189>

## Proof of Concept

## Proof of Concept

The linked secret Gist contains two complementary runnable PoCs:

1. A mounted Portal lifecycle test proving that a clean successful revoke actor for account `A` is accepted as completion of account `B`'s later revoke, so no `B` revoke is submitted.
2. An exact local-EVM consequence test proving that the retained `ROLE_SET_RESOLVER` lets `B` redirect the victim-owned name, that the deployed Universal Resolver returns `B`, and that Portal's real records-query and profile-card paths consume and render `B`.

These proofs are joined by the same chain, registry, account identities, role, fixed operation ID, and zero-`B`-revoke state. They are deliberately described as composable proofs rather than one browser-wallet-EVM process.

## Safety

The mounted lifecycle test is network-free. The exact consequence PoC refuses a non-loopback `LOCAL_RPC_URL`. All state-changing calls are sent only to loopback Anvil; the Sepolia endpoint supplies pinned historical state and bytecode and receives no transaction.

## Tested environment

```
Official source revision:  63772fd872af472ced58b009499355f3430c2a86
Audit-repo frozen import:  cda79acaad59711b943fc68207ebb3f1d0ff8596
Node:                     22.23.2
pnpm:                     10.27.0
Vitest:                   4.1.5
Foundry/Anvil:            1.7.1
Sepolia fork block:       11692735
Fork block hash:          0x84be4c0ff9f395b2a9d90263731933a7f44825334dbabc62153140101b848e83
```

A read-only Sepolia endpoint with historical state at block `11692735` is required for PoC 2.

## Step 1 — Prepare the workspace

Clone the competition repository and check out its frozen import. The import commit message records that it was created from source revision `63772fd872af472ced58b009499355f3430c2a86`, which is the revision listed on the competition Resources page:

```
git clone --branch audit-comp-ready https://github.com/immunefi-team/audit-comp-ens.git ens
git -C ens checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
```

Download the linked secret-Gist files. Gists store files at the top level, so arrange the three `sc-por-02-*` reproduction files under `reproductions/` as follows:

```
workspace/
├── ens/
└── SC-POR-02-submission/
    ├── SC-POR-02-report.md
    ├── execution-provenance.md
    ├── README.md
    ├── SC-POR-02-MANIFEST.txt
    └── reproductions/
        ├── sc-por-02-mounted-parent-lifecycle.test.tsx
        ├── sc-por-02-exact-registry.test.ts
        └── sc-por-02-vitest.config.mjs
```

Install the official frozen dependencies:

```
cd workspace/ens
corepack pnpm@10.27.0 install --frozen-lockfile
```

The supplied PoCs do not patch or modify application source.

## Step 2 — Reproduce the cross-account false completion

From `workspace/ens/apps/portal`, run:

```
corepack pnpm@10.27.0 run test sc-por-02-mounted-parent-lifecycle.test.tsx --root ../../.. --dir ../../../SC-POR-02-submission/reproductions --config SC-POR-02-submission/reproductions/sc-por-02-vitest.config.mjs --reporter=verbose
```

The test performs the following supported Portal actions and assertions:

1. It renders the actual `RegistryRolesTable` parent, `DataTable` row actions, `RegistryEditUserSheet`, transaction provider, transaction modal, status consumer, and actor state machine.
2. It supplies two distinct registry-role rows: account `A` and account `B`, each initially holding root `ROLE_SET_RESOLVER`.
3. Through the real row action, it selects `A` and confirms Remove user.
4. The real revoke helper creates the actor and drives the `A` transaction through submitting, pending, and clean success using a controlled local receipt seam.
5. It asserts that the mutation hook received exactly one revoke submission for `A`.
6. It dismisses the successful modal with the ordinary X button.
7. It closes `A`'s actual parent sheet, then selects `B` through the real table row action.
8. It confirms Remove user for `B`, creating a new descriptor with the same fixed `tx-edit-registry-roles-revoke` ID.
9. It asserts that Portal renders the final Done action for `B` by consuming the still-successful actor created for `A`.
10. It clicks Done and asserts that the current `B` sheet closes.
11. It asserts that the revoke hook still contains exactly one `A` call and zero `B` calls.
12. As a negative control, it repeats the sequence but explicitly clears the successful `A` actor before opening `B`.
13. In the control, Portal renders Start and submits a distinct revoke for `B`.

Expected result:

```
Test Files  1 passed (1)
Tests       2 passed (2)
SC_POR_02_PARENT_A_TO_B_FALSE_COMPLETION_REPRODUCED
SC_POR_02_CLEARED_ACTOR_SAFE_CONTROL
```

The negative control isolates the retained cross-subject actor as the causal difference.

## Step 3 — Start the pinned local Sepolia fork

In Terminal A, start Anvil bound to loopback:

```
anvil --host 127.0.0.1 --fork-url <READ_ONLY_SEPOLIA_RPC> --fork-block-number 11692735 --chain-id 11155111 --port 8545
```

Confirm that Anvil prints:

```
Block number: 11692735
Block hash:   0x84be4c0ff9f395b2a9d90263731933a7f44825334dbabc62153140101b848e83
Chain ID:     11155111
Listening on 127.0.0.1:8545
```

## Step 4 — Set the exact-proof environment

In Terminal B, set the same read endpoint and the loopback write endpoint.

POSIX:

```
export SEPOLIA_FORK_URL=<READ_ONLY_SEPOLIA_RPC>
export LOCAL_RPC_URL=http://127.0.0.1:8545
```

PowerShell:

```powershell
$env:SEPOLIA_FORK_URL = "<READ_ONLY_SEPOLIA_RPC>"
$env:LOCAL_RPC_URL = "http://127.0.0.1:8545"
```

## Step 5 — Reproduce the exact resolver and Portal consequence

From `workspace/ens/apps/portal`, run:

```
corepack pnpm@10.27.0 run test sc-por-02-exact-registry.test.ts --root ../../.. --dir ../../../SC-POR-02-submission/reproductions --config SC-POR-02-submission/reproductions/sc-por-02-vitest.config.mjs --reporter=verbose
```

The test performs these exact state transitions and assertions:

1. It rejects any `LOCAL_RPC_URL` whose hostname is not `127.0.0.1` or `localhost`.
2. It resets Anvil from the read endpoint at block `11692735` and asserts the expected block hash.
3. It verifies configured deployed contract code before using it.
4. It deploys deterministic UserRegistry `0xe6E4e8Ebe3fFc269faF85722e1d23F246544f9A9` through the deployed VerifiableFactory and UserRegistry implementation.
5. It constructs the deterministic parent/child name fixture.
6. It uses the frozen Portal helpers to mine a real grant and revoke for `A`, then asserts `A`'s role is zero.
7. It grants root `ROLE_SET_RESOLVER` to `B` and applies the lifecycle proof's exact postcondition: no `B` revoke transaction occurs.
8. It asserts `roles(0, B) == 16777216`.
9. Acting only with `B`'s deterministic fixture key, it deploys and configures a PermissionedResolver.
10. It calls the exact UserRegistry `setResolver`; the transaction succeeds.
11. It asserts direct resolver `addr(node)` returns `B`.
12. It asserts the deployed Universal Resolver selects the attacker resolver and returns `B`.
13. It asserts that victim `V` remains the ERC-1155 owner.
14. It routes Portal's actual `getRecords` helper to the loopback client and asserts that coin type 60 is `B`.
15. It mounts the actual `NameProfileCard` under real React Query and router providers.
16. `NameProfileCard` executes `getRecordsQueryOptions`; the real `CoinRecord` and `EntityActionCopy` render `B` in the DOM.

Expected result:

```
Test Files  1 passed (1)
Tests       1 passed (1)
SECOND_CRITICAL_PORTAL_EXACT_REGISTRY_REDIRECT_REPRODUCED
SC_POR_02_PORTAL_RECORDS_QUERY_CONSUMES_REDIRECT
SC_POR_02_PORTAL_PROFILE_RENDERS_REDIRECT
```

The structured marker also prints:

```
retainedRootRoles            = 0x1000000
skippedCurrentRevokeTxs      = []
finalResolver                = attacker-controlled PermissionedResolver
directResolvedAddress        = B
universallyResolvedAddress   = B
portalResolvedAddress        = B
victim remains owner         = true
```

Local Anvil transaction hashes may vary and are not success criteria.

## Step 6 — Verify package integrity

From `workspace/SC-POR-02-submission`:

POSIX:

```
sha256sum -c SC-POR-02-MANIFEST.txt
```

PowerShell:

```powershell
Get-Content .\SC-POR-02-MANIFEST.txt | ForEach-Object {
  $expected, $path = $_ -split '\s+', 2
  $actual = (Get-FileHash -Algorithm SHA256 -LiteralPath $path).Hash.ToLower()
  if ($actual -ne $expected) { throw "SHA-256 mismatch: $path" }
  "OK  $path"
}
```

Expected: all six non-manifest package members print `OK`.

## Evidence boundaries

* The mounted test controls query results, wallet/client receipt delivery, mutation recording, and presentation-only components. It retains the real parent selection, edit sheet, transaction provider/modal/status consumer, revoke helper, and actor state machine.
* The exact test begins at the mounted proof's measured zero-`B`-revoke postcondition; it does not claim both tests are one process.
* The exact test uses a deterministic local name fixture and deployed contract bytecode from the pinned fork; public-chain contracts are not alleged to be vulnerable.
* The application consequence uses the real Portal `getRecords` and `NameProfileCard` paths. Only `NameAvatar` and hover wrappers are controlled for presentation plumbing.
* No downstream user payment or transaction approval is reproduced or claimed.
* Nonfatal happy-dom, IndexedDB, or dialog accessibility warnings do not affect the exit code or asserted security postconditions.
