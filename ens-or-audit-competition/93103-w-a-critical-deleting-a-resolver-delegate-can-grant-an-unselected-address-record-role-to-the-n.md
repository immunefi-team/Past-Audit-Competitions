# 93103 w a critical deleting a resolver delegate can grant an unselected address record role to the next delegate

**Submitted on Sep 14th 2026 at 09:06:35 UTC by @unineko for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93103
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

**Summary**:

After an administrator removes resolver delegate A through the normal Explorer workflow, delegate B can move into A's former table index. The B-labelled role editor then retains A's mutable permission state while comparing it with B's live roles.

In the verified sequence, A has `TEXT|ADDR` and B has only `TEXT`. The administrator opens B and selects only `CONTENTHASH`. The UI presents the stale `ADDR` checkbox as if it were already one of B's existing permissions, and the generic confirmation modal does not enumerate the role delta. The Portal therefore asks the connected wallet to grant B `ADDR|CONTENTHASH`, even though the only B-specific role selected by the administrator was `CONTENTHASH`.

The wallet request is mined, B's bitmap changes from `16` to `273`, and B can then use the newly acquired `ADDR` role to replace `victim.eth`'s address record. The same B call reverts before the unintended grant and after `ADDR` is revoked. The administrator's real root-role row remains present throughout the verified transition: `[A,B,admin]` becomes `[B,admin]`.

```
B's live roles before the edit:      TEXT                       = 16
Administrator-selected addition:     CONTENTHASH                = 256
Expected B post-state:               TEXT | CONTENTHASH         = 272
Actual wallet grant delta:           ADDR | CONTENTHASH         = 257
Actual B post-state:                 TEXT | ADDR | CONTENTHASH  = 273
```

**Description**:

`DataTable` accepts only columns and data and does not supply `getRowId`:

```typescript
const table = useReactTable({
  data,
  columns,
  getCoreRowModel: getCoreRowModel(),
  // no getRowId
})
```

TanStack's fallback row ID for a root row is its input-array index. Meanwhile, `ResolverRolesTable` stores the selected TanStack `Row` and renders one persistent sidebar around the table:

```typescript
const [editingRow, setEditingRow] = useState<Row<AccountRoleGroup> | null>(null)

<ResolverRolesSidebar row={editingRow} ...>
  {table}
</ResolverRolesSidebar>
```

Inside the sidebar, current identity and roles come from `row.original`, but the editable map is refreshed only when the positional ID changes:

```typescript
const selectedAccount = row?.original.account
const decodedRoles = row?.original.decodedRoles ?? []

const [prevRowId, setPrevRowId] = useState<string | null>(null)
if (row && row.id !== prevRowId) {
  setPrevRowId(row.id)
  setEditedPermissions(roleToPermissions(row.original.decodedRoles))
}

const { rolesToGrant, rolesToRevoke } = computeRoleChanges(
  decodedRoles,
  editedPermissions,
)
```

Suppose A is input row `"0"` with `TEXT|ADDR`, B is row `"1"` with `TEXT`, and the administrator's older root assignment is row `"2"`. After the owner removes A, an honest refreshed `[B,admin]` result gives B the same default ID `"0"`. The guard does not run because `prevRowId` is also `"0"`. The resulting form has mixed provenance:

```
selected account and name: B / victim.eth
authoritative comparison baseline: B = TEXT
editable permission map:            A = TEXT | ADDR
```

The stale `ADDR` box is visibly checked under B as though it were B's existing state. When the owner checks only `CONTENTHASH`, the desired map becomes `TEXT|ADDR|CONTENTHASH`. Comparing it with B's real `TEXT` baseline produces:

```
rolesToGrant  = ADDR | CONTENTHASH
rolesToRevoke = none
targetAccount = B
```

`handleSaveChanges` snapshots those values and the production helper encodes:

```
authorizeNameRoles(dns("victim.eth"), 257, B, true)
```

The resolver executes exactly what the Portal requested; no on-chain access control bypass is alleged.

**Why the row compaction is reachable**:

Resolver account grouping preserves the first occurrence of each account in the returned role array. Successful role removal also invokes the production polling path, which invalidates and refetches the active `resolver-overview` query without remounting the surrounding sidebar.

As a feasibility check, the exact Panoptes image resolved from the repository's e2e configuration during validation—digest `sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b`—was inspected offline. Its resolver-role handler deletes the assignment row when the new bitmap is zero, and its resolver API orders remaining assignments by descending timestamp. With the standard resolver initializer's older admin root assignment also present, a newer A followed by B naturally changes `[A,B,admin]` into `[B,admin]`, which changes B's Portal ID from `1` to `0`.

This evidence establishes an honest data transition only. Panoptes is not claimed as the vulnerable asset or root cause. Digest-pinned extraction details and the offline SQL trace are included in `poc/finding17/PANOPTES_REACHABILITY.md`; the default PoC does not require the image. The digest proves the behavior of the repository-referenced image at validation time; it is not an attestation of which digest a live deployment currently runs.

**Root cause**:

The editor treats TanStack's default input-array index as persistent account identity and uses it as the sole guard for rebasing account-bound draft state. A refetch can therefore change `row.original.account` from A to B without changing the key that controls synchronization.

This is distinct from account-wide aggregation bugs: one name and one resource per account are sufficient, and permission state crosses between two different accounts.

**Required state and normal user actions**:

* The ordinary owner/admin holds the matching on-chain admin bits needed to grant and revoke the tested resolver roles; the Portal's UI gate alone is not sufficient. Portal's standard deployment stores its all-roles initializer bitmap at `ROOT_RESOURCE`, so this authority is itself a persistent indexed role row.
* Two non-self customer delegates A and B have roles on the same known name; A appears before B and has at least one role B lacks.
* A is not the final row, so a remaining account can occupy A's old index.
* The owner removes A, stays on the same Roles route until the refreshed row set appears, then opens B for a legitimate role change.
* The owner clicks Save/Start and approves the connected-wallet request.

B is not an initially permissionless outsider: B begins as an ordinary limited delegate trusted only with `ROLE_SET_TEXT`. B is not a project administrator, governance actor, strategist, deployer, or system operator. Once the administrator completes the normal sequence above, a malicious or compromised B receives `ROLE_SET_ADDR` outside that entrusted boundary and can use it until an administrator notices and revokes it.

**Attack sequence**:

1. On `victim.eth`, A has `ROLE_SET_TEXT|ROLE_SET_ADDR` (bitmap `17`) and B has only `ROLE_SET_TEXT` (bitmap `16`). The administrator has Portal's standard `0x1111…1111` all-roles bitmap at `ROOT_RESOURCE`.
2. The owner opens A, then removes A through the normal Portal transaction flow.
3. The successful transaction clears A. The mutation invalidates the active role query; the refreshed result contains B and the administrator.
4. B changes from default row ID `"1"` to A's former ID `"0"`. The sidebar remains mounted with A's `TEXT|ADDR` map and `prevRowId = "0"`.
5. The owner opens B. The sheet identifies B, but falsely presents `ADDR` as a checked existing role even though B's live bitmap is still `16`.
6. The owner explicitly checks only `CONTENTHASH` and saves.
7. Portal's generic modal says `Update roles for B`. The wallet receives a grant for bitmap `257` (`ADDR|CONTENTHASH`) to B; the owner approves it.
8. B's bitmap becomes `273` (`TEXT|ADDR|CONTENTHASH`). B can now change `victim.eth`'s address record, whereas the identical call reverted before the grant.

**Impact**:

The in-scope Portal constructs a security-sensitive permission argument from another account's stale state. For B, the administrator changes only `CONTENTHASH`, but the connected wallet receives and submits a grant for `ADDR|CONTENTHASH`.

The stale `ADDR` checkbox does not communicate consent to grant that role: the B-labelled editor falsely presents it as part of B's current state, and the confirmation modal says only `Update roles for B` without enumerating the roles that will be granted. The application converts that false baseline into a new permission for B.

This is the selected Critical impact, **Malicious interactions with an already-connected wallet**, whose listed examples include modifying transaction arguments or parameters. The ENS competition also specifically prioritizes transactions built or signed with the wrong arguments. Here the signer and target contract are correct, but the permission bitmap is not: the intended B-specific addition is `CONTENTHASH` (`256`), while the Portal builds and submits `ADDR|CONTENTHASH` (`257`).

The result is not cosmetic. The extra bit is present in the transaction-manager request, the EIP-1193 wallet request, and the mined calldata. It changes B's effective authority and enables an `ADDR`-gated write that the same account could not perform before.

**Validity and scope assessment**:

* This is not a false positive: the extra role is decoded from the actual wallet request, mined, read back from the role bitmap, exercised, and removed in the authority controls.
* The intended business invariant is account-local rebasing. The sidebar's own comment says it synchronizes when the selected row changes; the generic roles sidebar resets from the selected Row object, and the sibling registry editor resets on open/current roles. A positional ID collision is not an intended cross-account merge.
* This is not a TanStack defect. Index fallback is documented library behavior; the in-scope Portal treats that incidental index as durable account identity.
* The smart contract and indexer exclusions do not apply to the root cause. They are used only as the normal transaction sink and a validated data boundary. The vulnerable state ownership and malformed argument construction are in `apps/portal`, an explicitly listed Explorer target.
* Every required victim action is prompted and part of the normal role-removal and role-edit workflows, so the exclusion for unprompted actions outside normal workflows does not describe this sequence.

**Known-issue comparison**:

The public Known Issue baseline was re-checked on 2026-09-10. No entry reviewed has the same positional-row identity → cross-account permission draft → new role grant root cause.

| Published issue | Different root cause and result                                                                                                                                                                                                                                                                    |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `QA-01`         | Covers transfer/role combinations causing later functional errors and expressly states that a demonstrated combination granting unintended authority would be a new finding. Finding 17 needs no transfer: its role-management transaction succeeds and creates new authority for another account. |
| `R3-07`         | Covers reused transaction IDs omitting history/telemetry. Finding 17 changes wallet calldata and the permission post-state even when transaction tracking succeeds.                                                                                                                                |
| `R2-06`         | Covers local browser cleanup without on-chain SmartSession revocation. Finding 17 uses neither browser storage nor SmartSession authority.                                                                                                                                                         |

**Recommendation**:

Use semantic identity for resolver-role rows and rebase drafts on the selected account rather than a positional ID.

1. Allow `DataTable` to receive `getRowId` and use the normalized account for `ResolverRolesTable`, for example `row.account.toLowerCase()`.
2. In the sidebar, track the prior Row object or normalized account and reset `editedPermissions` whenever the semantic account changes, even if a table index is reused.
3. Clear abandoned draft state when the sheet closes, and carefully reconcile authoritative role changes for the same account without overwriting an active user edit.
4. Before opening the wallet, display and verify the exact roles to grant and revoke, target account, and name.
5. Add a regression test for `[A(id "0"), B(id "1"), admin(id "2")] → [B(id "0"), admin(id "1")]`: opening B must show B's own roles and Save must remain disabled until a B-specific edit.

## Link to Proof of Concept

<https://gist.github.com/unineko5555/f69fd3517ed8e4a4a6fae8448c79cb7e>

## Proof of Concept

[`PoC`](https://gist.github.com/unineko5555/f69fd3517ed8e4a4a6fae8448c79cb7e).

**PoC Result**: PASS — the Chromium-to-deployed-resolver E2E passed `1/1` on the local Sepolia fork. Separately, the joined source-to-local-chain test passed `3/3` independent processes with identical calldata, post-state, and two authority negative controls.

Its 13 files comprise the README, standalone and optional semantic-fork runners, joined test and Vitest config, network guard, required hash-checked resolver-model creation bytecode, its Solidity source and optional build checker, the Chromium E2E test/config/package marker, and the optional Panoptes reachability appendix. No generated test output, trace, or Anvil state is included. The PoC imports the ENS apps checkout and cannot run as a standalone repository. GitHub Gist flattens directories, so restore them first.

Requirements for the standalone path: the complete contest checkout at locally tested revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, Node.js (`v22` in the competition Resources; PoC tested with `v24.11.1`), pnpm `10.27.0`, Bash, and Anvil (tested with Foundry `1.5.1`). The Chromium E2E additionally requires Docker Compose, Playwright Chromium, and a running Portal development server. From the checkout root, where `poc/finding17/` does not yet exist:

```bash
mkdir -p poc
git clone https://gist.github.com/unineko5555/f69fd3517ed8e4a4a6fae8448c79cb7e.git poc/finding17
mkdir -p poc/finding17/portal poc/finding17/support poc/finding17/fixtures poc/finding17/e2e
mv poc/finding17/Finding17.joined.poc.test.tsx poc/finding17/vitest.config.ts poc/finding17/portal/
mv poc/finding17/block-nonlocal-network.cjs poc/finding17/support/
mv poc/finding17/PermissionedResolverModel.creation.hex poc/finding17/PermissionedResolverModel.sol poc/finding17/verify-build.mjs poc/finding17/fixtures/
mv poc/finding17/finding17-ui.spec.ts poc/finding17/playwright.config.ts poc/finding17/package.json poc/finding17/e2e/
pnpm install --frozen-lockfile
bash ./poc/finding17/run.sh
```

The Gist clone does not retain executable mode. Dependency installation may use the network; the standalone runner itself starts Anvil on a random localhost port, clears inherited environment secrets, sets fail-closed proxy defaults, and loads a Node guard that rejects non-local HTTP, HTTPS, TCP, TLS, and `fetch` destinations. It needs no public RPC, API key, subgraph, explorer, wallet extension, Docker image, Solidity compiler, or external service. `bash ./run.sh` also works from `poc/finding17/` after restoration.

Expected result:

```
Test Files  1 passed (1)
Tests       1 passed (1)
[POC-PASS] stale row authority is granted to the replacement account and exercised
```

**Primary Chromium E2E**: From the repository root, start the disposable Sepolia fork and Portal development server:

```bash
docker compose -f e2e/infra/docker-compose.yml up -d anvil
VITE_SEPOLIA_RPC_URL=/rpc \
VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:5655/graphql \
VITE_RHINESTONE_ENDPOINT_URL=/orchestrator \
VITE_RHINESTONE_USE_DEV_CONTRACTS=true \
pnpm --filter portal dev --mode development --host 127.0.0.1 --port 3001
```

In another terminal:

```bash
cd e2e
PORTAL_APP_URL=http://127.0.0.1:3001 \
ANVIL_RPC_URL=http://127.0.0.1:8545 \
pnpm exec playwright test --config=../poc/finding17/e2e/playwright.config.ts
```

Expected: `1 passed` and `[finding17-e2e] PASS ... bitmap=257 ... roleRows=[A,B,admin]->[B,admin]`. The test rejects a non-fork or missing deployed ENS V2 contracts before writes, snapshots the disposable local fork, and reverts it on normal completion. Run it exclusively; an abrupt process kill may leave fixture writes behind, and `evm_revert` undoes concurrent writes.

The audited `makeName` fixture registers a fresh `.eth` name through the deployed ETHRegistrar and creates its PermissionedResolver proxy through the deployed VerifiableFactory. The test verifies UniversalResolver discovery and the proxy's EIP-1967 implementation slot. Chromium mounts the actual Portal roles route and uses an injected headless wallet. On the deployed resolver, it establishes `[A,B,admin]`, removes A through the Portal, observes the refreshed `[B,admin]` table, and selects only `CONTENTHASH` for B. The wallet-signed transaction mined at that proxy nevertheless grants bitmap `257` (`ADDR|CONTENTHASH`). B's live bitmap becomes `273`, and B successfully sets the new name's ETH address record; the identical B write reverts before the grant and after direct `ADDR` revocation.

Only the indexer GraphQL transport is substituted. Each resolver-overview response is rebuilt from the deployed proxy's current role bitmaps and assignment chronology, with zero-bit A omitted after removal. This exercises the mounted Portal route, query refetch, row compaction, wallet and contract execution, but does **not** claim a live Panoptes run or validate its discovery of the new proxy. The test keys and transactions stay on local Anvil; no transaction is sent to public Sepolia.

**Supplemental standalone and semantic-fork joined test**:

The test imports the unchanged production `ResolverRolesTable`, `DataTable`, sidebar/diff logic, transaction helpers, `TransactionModal`, transaction manager, and EOA transport. A local query reads A and B's name-scoped bitmaps and the administrator's persistent root bitmap; receipt-block order is used as a deterministic chronology proxy. The production mutation's invalidation callback refetches the query in the same mounted component and changes `[A,B,admin]` to `[B,admin]` after the actual A-removal transaction.

The wallet request decodes to:

```
authorizeNameRoles(dns("victim.eth"), 257, B, true)
calldata Keccak = 0xd785882ea2601e58e35a831a53e16fd0d99233c07aead58eb14713589ba3fcd9
```

The transaction-manager request, EIP-1193 request, and isolated-Anvil mined input are byte-for-byte identical, and the receipt succeeds. The test asserts:

```
B before = 16  (TEXT)
B after  = 273 (TEXT | ADDR | CONTENTHASH)
```

The identical B `setAddr(victim.eth, 0xdead...beef)` call reverts before the grant, succeeds afterward, and reverts again after revoking only `ADDR`.

The test additionally verifies that the administrator's root role is the standard `0x1111…1111` initializer bitmap, B's and A's assignments are ordered by their local receipt blocks, the actual A-removal transaction zeroes A's bitmap, the production mutation invalidates and refetches the still-mounted query, merely opening B sends no wallet request, and the owner selects only `CONTENTHASH` before Save. These checks distinguish the reachable `[A,B,admin] → [B,admin]` transition from an artificial row substitution.

The bundled resolver model is a narrow postcondition oracle, not the vulnerable asset or a byte-for-byte ENS deployment. It implements the relevant EAC semantics: independent resource bitmaps, `ROOT_RESOURCE` fallback, admin-bit-derived delegation, and role-gated setters. Its constructor uses the exact standard Portal initializer bitmap, and its hash-checked creation bytecode has Keccak `0xb01c04d61261b67f9398d5e081598a9dbee45a4945707af3f4a0cfd5269e1cdf`. No public RPC, fork, indexer, explorer, API key, wallet extension, external service, Docker image, or compiler is used by the normal runner. The remote indexer is intentionally not contacted: a local query constructs coherent rows from mined role bitmaps and follows the production invalidation/refetch lifecycle. The optional `PANOPTES_REACHABILITY.md` in the Gist documents the digest-pinned delete-on-zero and descending-order evidence supporting that boundary; it is not executed by `run.sh` and does not assert which image digest a live deployment currently runs. The normal runner consumes the included precompiled bytecode. An optional independent check with `solc 0.8.28` is `pnpm --filter=portal exec node ../../poc/finding17/fixtures/verify-build.mjs`, expected to print `[BUILD-PASS] fixture source reproduces committed creation bytecode`. Possible Dialog accessibility, `IndexedDB unavailable`, and React `act(...)` warnings in happy-dom do not change the asserted wallet request, mined authority, or negative controls.
