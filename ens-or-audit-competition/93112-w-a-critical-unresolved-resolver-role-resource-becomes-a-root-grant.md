# 93112 w a critical unresolved resolver role resource becomes a root grant

**Submitted on Sep 14th 2026 at 09:49:57 UTC by @unineko for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93112
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Summary

The Portal receives an existing resolver role with an authoritative nonzero `resource`. For text-key, address-coin-type, and other record-part roles, its nodes-derived lookup cannot identify that resource. The role nevertheless remains editable. On Save, the Sidebar turns its missing display name into `name = ''`, the same sentinel used for an explicitly selected resolver root. The production helper passes `scope: 'root'` to the locked ENSjs encoder, which creates `grantRootRoles` calldata.

In the victim scenario, a legitimate root administrator adds `ROLE_SET_ADDR` to a previously limited delegate's unresolved row and approves the wallet request. The Chromium PoC executes that UI and wallet flow against a deployed PermissionedResolver proxy on a local Sepolia fork, with two genuinely registered `.eth` names sharing the proxy. The selected delegate and resolver addresses remain correct; the undisclosed change is the **root function and resource scope**. The mined application-generated call grants resolver-root `ROLE_SET_ADDR`, allowing the delegate to modify the sibling name. This is not a contract access-control bypass or an interactionless exploit.

## Root cause and reachable path

The overview query fetches `nodes` and `roles { account resource roleBitmap name }` but not `namedResources` (`apps/portal/src/features/resolver/hooks/useResolverOverview.ts:68-110`). `buildResourceToNameMap()` covers only literal root and each node's broad part-zero name resource (`apps/portal/src/lib/roles/resolverRoles.ts:129-159`). A legitimate text-key resource is `keccak256(namehash(name), keccak256(key))`, not `keccak256(namehash(name), 0)`; `authorizeTextRoles` and `authorizeAddrRoles` are normal contract APIs for such fine-grained assignments.

`groupRolesByAccount()` retains an unmatched role and bitmap but leaves `resolvedNames` empty (`resolverRoles.ts:179-220`). The table's Edit action checks manager access, not map success (`apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:102-117`). For that row, the Sidebar executes:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
// Save snapshots name: roleName into pendingAction.
```

The scope sentence is hidden when `resolvedNames` is empty, yet Save is enabled (`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-170,295-305,389-430`). Both modal preflight and submission use this same pending action (`:195-239,467-502`). The grant helper interprets `name === ''` as `scope: 'root'` (`apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:29-74`), and ENSjs encodes `grantRootRoles(1, delegate)`. Neither the helper nor ENSjs receives the discarded original resource.

This is not an intentional root selection: Add User visibly labels its empty root default **Root (all nodes)**. The existing-row editor neither asks for root scope nor says that an unknown nonzero resource will become root. A text-key role also cannot safely be converted to a broad name-scoped address role. An unsupported resource should be read-only or require an explicit new scope selection.

## Preconditions and impact

* A legitimate name-level or root administrator has already created a fine-grained role for delegate `D`. The current Portal UI does not create these record-part assignments; `D` cannot manufacture this prerequisite with a mere manager role.
* Victim `O` is connected on Sepolia, holds root regular `ROLE_SET_ADDR` for the Portal gate and the matching root admin bit for the grant, edits `D` (not self), deliberately selects **Set Address**, clicks Save, and approves the wallet request. Merely passing the Portal screen gate is insufficient on-chain.
* The new root role has fewer than 15 assignees. For the demonstrated cross-name consequence, another valuable name uses the same resolver instance and `D` acts before revocation. An ABI-aware wallet may reveal the root selector, allowing `O` to reject it.

The contract correctly executes the request the Portal supplies: its root role bit applies to all resources **within the selected resolver instance**, including a sibling name. The delegate could not change that sibling's address before the grant, but can after it. The PoC shows the sibling address changing from `0x4000…0004` to `0x5000…0005`; it does not claim a subsequent payment, fund theft, a permanent takeover, or an effect on names using other resolvers. The administrator can later revoke the root role, leaving an unwanted authority window until then.

The official connected-wallet impact category is Critical, but actual triage may discount the required fine-grained prerequisite, privileged victim action, wallet approval, capacity, and same-resolver sibling. We conservatively score the scenario **High impact × Low likelihood = Medium** internally.

## Scope and known issues

The vulnerable component is the in-scope Portal: it converts a received resource into the wrong wallet-request scope. ENSjs correctly encodes the Portal's explicit root instruction, and the resolver correctly enforces and applies it. The contract—semantic fixture in the original suite, deployed proxy in the Chromium E2E—is an effect oracle, not an alleged out-of-scope defect. The delegate starts as a limited customer account, not a trusted deployer, governance operator, or compromised administrator; the privileged victim action and signature are disclosed above.

QA-01 addresses later functional failures from role/transfer combinations and expressly distinguishes unintended authority. This bug needs no transfer and constructs an undisclosed root authorization. R2-06 (SmartSession cleanup) and R3-07 (transaction history IDs) have different root causes and post-states. No matching public Known Issue was found, although confidential duplicates cannot be excluded.

## Remediation

Preserve the authoritative `{account, resource, bitmap}` through query, table, editor, confirmation, and transaction. Model supported root, broad-name, text-key, and address-coin-type scopes explicitly. For an unmapped nonzero resource, fail closed before opening the wallet; never reuse `''` for both explicit root and failed lookup. If account-level display grouping remains, mutation must enumerate every exact resource and the confirmation must disclose each resulting call. Cover unknown-vs-zero, multi-resource removal, matching admin bits, and the 15-assignee boundary in regressions.

## Link to Proof of Concept

<https://gist.github.com/unineko5555/58a6db732f9f87c86730d7ada1b70c28>

## Proof of Concept

[`PoC`](https://gist.github.com/unineko5555/58a6db732f9f87c86730d7ada1b70c28)

**Primary PoC:** the Chromium E2E in the [Finding 18 Gist](https://gist.github.com/unineko5555/58a6db732f9f87c86730d7ada1b70c28), verified revision `6a17f813e61eefd535a8de5ceb30d642bf416d6f` (15 flat files). It drives the audited Portal through an injected test wallet to a mined transaction against a deployed PermissionedResolver proxy and proves the sibling-name write. The fixture-backed `run.sh` is supplementary; running it alone does **not** establish the deployed-contract browser claim. The PoC imports the ENS apps checkout at `1c9b47f18fcddd2e864dfe385c4171061c9811ae` and cannot run alone. Install Node 24, pnpm 10.27.0, Docker, Playwright Chromium, `solc`, Anvil, `curl`, and Bash as needed. From a clean checkout root, restore Gist's flat Solidity and E2E file names:

```sh
mkdir -p poc
git clone https://gist.github.com/unineko5555/58a6db732f9f87c86730d7ada1b70c28.git poc/finding18
git -C poc/finding18 checkout 6a17f813e61eefd535a8de5ceb30d642bf416d6f
mkdir -p poc/finding18/contracts poc/finding18/e2e
mv poc/finding18/PermissionedResolverFixture.sol poc/finding18/contracts/PermissionedResolverFixture.sol
mv poc/finding18/e2e-finding18-ui.spec.ts poc/finding18/e2e/finding18-ui.spec.ts
mv poc/finding18/e2e-playwright.config.ts poc/finding18/e2e/playwright.config.ts
mv poc/finding18/e2e-package.json poc/finding18/e2e/package.json
pnpm install --frozen-lockfile
```

### Primary reproduction: Chromium E2E on a deployed resolver

Run exclusively on a disposable fork; the test snapshots and reverts the fork, including any concurrent writer's changes. From the checkout root, start Anvil, then leave the unmodified Portal development server running in that terminal:

```sh
docker compose -f e2e/infra/docker-compose.yml up -d anvil
VITE_SEPOLIA_RPC_URL=/rpc \
VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:5655/graphql \
VITE_RHINESTONE_ENDPOINT_URL=/orchestrator \
VITE_RHINESTONE_USE_DEV_CONTRACTS=true \
pnpm --filter portal dev --mode development --host 127.0.0.1 --port 3001
```

From a second terminal in the **same checkout**:

```sh
cd e2e
PORTAL_APP_URL=http://127.0.0.1:3001 \
ANVIL_RPC_URL=http://127.0.0.1:8545 \
pnpm exec playwright test --config=../poc/finding18/e2e/playwright.config.ts
```

Expected: `1 passed`, `OVERGRANT`, and `PASS both consolidated Finding 18 consequences` (the same test also emits `UNDERREVOKE` for the secondary branch). The primary assertion is the mined Portal wallet call `grantRootRoles(1,D)` on the deployed proxy: `D` has only a target-name text-part role before it, cannot write the sibling address before it, can write the sibling after it, and loses that access after root revoke. The selected delegate and resolver are correct; the wrong **root scope** is the security consequence. Stop the Portal and Anvil after the test.

Only indexer GraphQL is substituted, with role bitmaps rebuilt from deployed-proxy reads; node names and event-shaped metadata are harness-supplied. The wallet is an injected test wallet. This is not a hosted Portal, independent Panoptes, external-wallet, or public-Sepolia demonstration.

### Supplementary isolated and joined tests

After the primary E2E, the fixture-backed suite can check helper invariants and additional branches. From the checkout root:

```sh
bash poc/finding18/run.sh
```

The isolated runner builds the semantic fixture, typechecks, starts its own loopback Anvil, and performs no public-chain write. The clone's original `run.sh` lacks executable mode, so invoke it through Bash. Dependency installation may require network access; `run.sh` restricts test traffic to localhost.

Expected isolated-run summary, reproduced on the audited checkout:

```
Test Files  3 passed (3)
Tests       12 passed (12)
[PROPERTY-PASS] seed=0x18c01f00d cases=72 unmappedRecordRoot=72 mappedBroadName=72 addrBoundaryRoot=5
[POC-PASS] FINDING-18 over-grant and under-revoke verified
```

The tests import the production map, grouping, permission diff, grant/revoke intent builders, and locked ENSjs encoder. The primary branch's exact evidence is:

```
authoritative resource = 0xdcf60a9ea553686c8fa25af4fcb2a180001b7938e7c072e0143147e962c631b9
resolvedNames         = []
Sidebar name          = ""
selector              = 0x072d5d77
decoded wallet call   = grantRootRoles(1, 0x70997970C51812dc3A010C7d01b50e0d17dc79C8)
root authorization    = false → true
sibling authorization = false → true
sibling address       = 0x4000000000000000000000000000000000000004
                     → 0x5000000000000000000000000000000000000005
```

The joined local test feeds an indexer-shaped role into the real Table, clicks Edit / Set Address / Save, advances the real TransactionModal, and uses the real transaction manager and an EIP-1193 `eth_sendTransaction` request. Anvil mines the root grant; the delegate then writes the sibling address. A separate Sidebar test isolates checkbox/Save-to-intent behavior; effect tests decode the helper-produced calldata and execute it on Anvil.

The 12 tests also check mapped broad-name and literal-root controls; no-change behavior; missing admin bit; the 15-assignee cap; mixed grant/revoke; 72 deterministic unmapped text-part cases, 72 mapped controls, and five address coin-type boundaries. The named-revert helper requires an actual caught simulation error, and a self-check rejects a simulation known to succeed. These corrections and both UI regressions were included before the verified 12/12 runs.

### Browser E2E assertion details

The primary E2E above covers **both** consolidated branches. The final verified run on Node 24.11.1 and the local Sepolia fork reported `1 passed`, `OVERGRANT`, `UNDERREVOKE`, and `PASS both consolidated Finding 18 consequences`. The audited `makeName` fixture registers two real `.eth` names through the deployed registrar, creates a PermissionedResolver proxy through VerifiableFactory, and attaches the second name to that same proxy through the deployed registry. The test checks both names' resolver address, the proxy's EIP-1967 implementation slot, the on-chain bitmaps, the wallet-submitted function selectors and arguments, the final record values, and snapshot cleanup.

For the primary branch, `authorizeTextRoles` gives delegate `D` only the target name's text-part role (`16`), with root role `0`. A sibling `setAddr` call reverts. Editing that nameless row in Chromium and selecting only **Set Address** produces a wallet-authorized, mined `grantRootRoles(1, D)` on the deployed proxy. The root bitmap becomes `1`, and `D` can then write the sibling's ETH address. After root revoke, the same write reverts while the original text-part role remains.

For the secondary branch, `D` receives name-wide `ROLE_SET_ADDR` on each of the two registered names. The grouped Portal row displays both; **Remove user** submits only `authorizeNameRoles(firstName, 1, D, false)`. The first-name bitmap falls to `0`, but the second stays `1` and a sibling `setAddr` transaction still mines. The browser test preserves newest-assignment-first role ordering; either name could be left behind depending on which the UI selects first.

### PoC boundaries and isolation

The effect-test file and Sidebar-only test file mock unused transaction-manager entrypoints. The joined test does **not**: the real Table, Sidebar, modal, manager, and wallet-request path execute continuously. Its indexer-shaped input, wallet connection hooks, and local wallet transport are harness-controlled; it does not fetch a hosted indexer or use an external wallet. It adapts the configured Sepolia chain ID to isolated Anvil.

`contracts/PermissionedResolverFixture.sol` is a purpose-built **semantic fixture**, not deployed resolver bytecode. It preserves the relevant ABI, resource calculation, root fallback, admin check, role cap, name-role mutation, and address authorization; fixture-only owner methods establish the prerequisite state. Those relevant paths were cross-checked against `EnhancedAccessControl.sol` and `PermissionedResolver.sol`. `run.sh` executes this fixture on isolated Anvil; the optional `run-fork.sh` executes the same fixture on a Sepolia fork, not the deployed PermissionedResolver. The joined Vitest path is a continuous **local UI-to-mined-effect integration**, not a deployed-contract E2E.

The separate Chromium E2E **does** call the deployed registrar, registry, and PermissionedResolver proxy. Its trust boundary is narrower than a production-indexer test: an injected headless wallet authorizes the browser request, and only the indexer GraphQL transport is substituted. Resolver-overview rows are built from current on-chain `roles(...)` bitmaps for the tracked assignments; node names and event-shaped metadata are supplied by the harness. It does not prove that an independent Panoptes instance emits the same rows, nor does it use the hosted Portal or an external wallet. The audited `e2e/*` fixtures are imported, not modified.

`run.sh` builds and typechecks locally, sanitizes the environment, forces offline package-manager mode, starts Anvil on `127.0.0.1`, and cleans compiler artifacts. `network-guard.ts` rejects non-loopback fetch, HTTP(S), TCP, TLS, and DNS targets. `run-fork.sh` and the Chromium E2E require loopback RPC and Sepolia-fork provenance and restore a pre-test snapshot on ordinary exit. No public-chain exploit transaction was sent.

## Related under-revocation branch (consolidated Finding 2)

The same account-level display projection also collapses two **mapped** name resources to the first name during “remove all”. The Portal emits one `authorizeNameRoles(firstName, bitmap, D, false)` call, leaving `D`'s second-name role active although the UI promises removal “from all roles”. Both the isolated fixture suite and the deployed-proxy Chromium E2E execute that exact revoke and prove the second-name role and sibling write survive. This requires two mapped assignments and does **not** create the primary branch's fine-grained prerequisite; it is included only as a related resource-identity loss and not submitted for a second award. The fixed names below are from the isolated fixture test; the E2E registers fresh `.eth` names.

```
displayed resources = [limited.eth, sibling.eth]
selected scalar name = limited.eth
decoded wallet call = authorizeNameRoles(limited.eth, 1, D, false)
limited.eth role = true → false
sibling.eth role = true → true
sibling address = 0x4000000000000000000000000000000000000004
                → 0x5000000000000000000000000000000000000005
```

## PoC package files and integrity

* `finding18.poc.test.ts`: both branches, controls, and deterministic invariant.
* `finding18.sidebar.test.tsx`: isolated real Sidebar-to-modal intent regression.
* `finding18.joined.test.tsx`: continuous local Table-to-mined-effect regression.
* `contracts/PermissionedResolverFixture.sol`: shared semantic effect oracle.
* `run.sh`, `network-guard.ts`, `vitest.config.ts`, `tsconfig.json`: isolated runner and configuration.
* `run-fork.sh`, `fork-rpc.mjs`, `fork-rpc.d.mts`: Sepolia-fork runner, provenance check, and TypeScript declaration.
* `e2e/finding18-ui.spec.ts`, `e2e/playwright.config.ts`, `e2e/package.json`: browser-to-deployed-resolver regression and Playwright configuration. In the flat Gist these three files have an `e2e-` prefix; the restore commands above move them into `e2e/`.

SHA-256 of all non-README files in the verified 15-file Gist revision:

```
ef4f0a23d2c0db8b3e7fd4b5ae6c16a7a3523033edffbff504963e0ff13bb816  run.sh
f742634b0f242803bd09823bce0c94b95be76d512b4e235d0122d452d8b0b8ec  finding18.poc.test.ts
5d1984d3b551d776b0dfb44a94115deff0f55b7a33a5a193fd3feec84ceb4842  finding18.sidebar.test.tsx
0bf5b4f5a85b4aaa9daefd26522b47f263140caad2d8008bdbeca21c5e9aaca6  finding18.joined.test.tsx
f8319cb8478b73d67542cb064f860fc9d275900b431bfaa0f433272eb1b0e3b9  contracts/PermissionedResolverFixture.sol
32262010ad774b5f9d1dd048538ef7e279f39cb80c284be8650a03e3229b4aba  network-guard.ts
af8b7fe8e9bf17247ed7b5683d961dedef3028ff9e09fc3f98c5e3dd9e587e13  vitest.config.ts
041997fe5e18ca167f8cb3b766ca692566fd31f2d403f284add47d02f3ffd566  tsconfig.json
e28c20560d19d1f5677e9dbc2b5cc7e3e07cc19e0d8afdd1a4887957dbfd0082  run-fork.sh
cf8364ce83b69bace0763dfe04a2db2309c1f135ac4580fea16b461973af1e5b  fork-rpc.mjs
b8f69bf4f5ee1524f2866bd2c7ab1b440c823d11d3f4852ed31cb844b5181b46  fork-rpc.d.mts
80029b270d980c9d111e34336a06c36eb0ad20793f0325d8d2d49acb7a1e4403  e2e/finding18-ui.spec.ts
2de17742178879c3090565c384c471e9c4a9dbe3b63ddf539205df927ca0f1a6  e2e/playwright.config.ts
3ca9d4afd21425087cf31893b8f9f63c81b0b8408db5e343ca76e5f8aa26ab9a  e2e/package.json
```
