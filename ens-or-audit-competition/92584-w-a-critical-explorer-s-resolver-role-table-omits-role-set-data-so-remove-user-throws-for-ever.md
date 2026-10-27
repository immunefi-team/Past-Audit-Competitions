# 92584 w a critical explorer s resolver role table omits role set data so remove user throws for every account the registration flow provisions and otherwise leaves write access the roles table r&#x20;

**Submitted on Sep 10th 2026 at 19:03:06 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92584
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Explorer carries its own hand-written copy of the `PermissionedResolver` role table:

```tsx
// apps/portal/src/lib/roles/resolverRoles.ts:24-45
export const resolverRoles = {
  ROLE_SET_ADDR: 1n << 0n,
  ROLE_SET_ADDR_ADMIN: (1n << 0n) << 128n,
  … ROLE_SET_TEXT, ROLE_SET_CONTENTHASH, ROLE_SET_PUBKEY, ROLE_SET_ABI,
    ROLE_SET_INTERFACE, ROLE_SET_NAME, ROLE_SET_ALIAS, ROLE_CLEAR …
  ROLE_UPGRADE: 1n << 124n,
  ROLE_UPGRADE_ADMIN: (1n << 124n) << 128n,
} as const
```

Ten roles and their admin bits. The canonical table — in `@ensdomains/ensjs`, the same library the app uses to **encode** every resolver role write — has eleven:

```tsx
// @ensdomains/ensjs/src/utils/v2/roles/resolverRoles.ts:9-19, 32
export const RESOLVER_ROLE_CLEAR    = 1n << 32n
export const RESOLVER_ROLE_SET_DATA = 1n << 36n      // <- absent from the app
export const RESOLVER_ROLE_UPGRADE  = 1n << 124n
…
export const RESOLVER_ROLE_SET_DATA_ADMIN = RESOLVER_ROLE_SET_DATA << 128n
```

`ROLE_SET_DATA` is real. The deployed `PermissionedResolver` implementation exposes both `setData(bytes32,string,bytes)` and `authorizeDataRoles(bytes,string,address,bool)`.

Two consequences follow, and they are independent.

**One — the app's decoder is blind to it.** `decodeResolverRoleBitmap` iterates the app's ten-role table, so an account holding `ROLE_SET_DATA` decodes to the empty list. The roles table renders it as holding nothing, and `ResolverRolesSidebar.tsx:310` disables "Remove user" when `decodedRoles.length === 0`. The account cannot even be selected for removal.

**Two — the app's encoder is fed names the library does not know.** "Remove user" passes `decodedRoles` straight through:

```tsx
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:317-320
      type: 'remove',
      …
      roles: decodedRoles,
```

Those names reach ensjs's `encodeResolverRoleBitmap`, whose map contains **only the eleven non-admin roles**. Any `*_ADMIN` name resolves to `undefined`, and `bitmap |= undefined` throws before a single byte of calldata is built.

Every account the registration flow provisions holds every admin bit: registration deploys a dedicated resolver initialised with `ROLES.ALL` (`0x1111…1111`, every nibble set) for the registrant (`registration.actors.ts:71-73`, `smart-account/manifest.ts` `ROLES_ALL`). So on the accounts that matter most, "Remove user" is inoperable.

Strip the admin names and it does build a transaction — one that leaves `ROLE_SET_DATA` and `ROLE_SET_DATA_ADMIN` set, forever, with no path in the app to see or revoke them.

***

## Impact

Precondition: a `PermissionedResolver` on which some account holds roles the user wants to withdraw. That is the entire purpose of `/resolver/$address/roles`.

The registration flow is the source of the accounts that matter. It deploys a dedicated resolver per name and grants the registrant `ROLES.ALL` at the resolver root:

```tsx
// packages/transaction-manager/src/machines/registration/registration.actors.ts:71-73
const DEDICATED_RESOLVER_ROLE_BITMAP = BigInt(
  '0x1111111111111111111111111111111111111111111111111111111111111111',
)
```

Bit 36 (`SET_DATA`) and bit 164 (`SET_DATA_ADMIN`) are both inside that mask.

| What the user does                                                             | What the app shows                           | What actually happens                                                                                                               |
| ------------------------------------------------------------------------------ | -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Opens `/resolver/<addr>/roles` and looks at an account holding only `SET_DATA` | *no roles*                                   | the account can write record data on every name the resolver serves                                                                 |
| Presses **Remove user** on an account provisioned by registration              | the confirmation sheet, then a wallet prompt | nothing — `prepareRevokeResolverRolesTransaction` throws `TypeError: Cannot mix BigInt and other types` before any calldata exists  |
| Presses **Remove user** on an account holding only manager roles               | *removed*                                    | `SET_DATA` and `SET_DATA_ADMIN` remain set; `SET_DATA_ADMIN` lets the account re-grant `SET_DATA` to itself or anyone, indefinitely |

`SET_DATA_ADMIN` is what makes the residue permanent rather than merely leftover: the removed account administers its own remaining role. Nothing in the Explorer can see it, display it, or take it away.

The user's action here is the app's own answer to "this account should no longer have access to my resolver". They perform it, the app reports success, and the roles table then shows the account with nothing — which is precisely the state that convinces them not to look further.

***

## Root cause

### The app's table

| File:line                                            | Code                                                                       |
| ---------------------------------------------------- | -------------------------------------------------------------------------- |
| `apps/portal/src/lib/roles/resolverRoles.ts:24-45`   | `resolverRoles`, ten roles + ten admin bits — no `ROLE_SET_DATA`           |
| `apps/portal/src/lib/roles/resolverRoles.ts:57-108`  | `resolverPermissions`, the ten checkboxes the sheet can ever show or grant |
| `apps/portal/src/lib/roles/resolverRoles.ts:113-127` | `decodeResolverRoleBitmap` iterates that table only                        |
| `apps/portal/src/lib/roles/resolverRoles.ts:193-215` | `groupRolesByAccount` builds each row's `decodedRoles` from it             |

### The library's table, and the write path that uses it

| File:line                                                                       | Code                                                                                 |
| ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `@ensdomains/ensjs/src/utils/v2/roles/resolverRoles.ts:18, 32`                  | `RESOLVER_ROLE_SET_DATA = 1n << 36n`, `RESOLVER_ROLE_SET_DATA_ADMIN = … << 128n`     |
| `@ensdomains/ensjs/src/utils/v2/roles/resolverRoles.ts:48-60`                   | `resolverRoleMap` — the eleven non-admin roles, **no `*_ADMIN` keys at all**         |
| `@ensdomains/ensjs/src/utils/v2/roles/resolverRoles.ts:65-71`                   | `encodeResolverRoleBitmap`: \`bitmap                                                 |
| `@ensdomains/ensjs/src/actions/wallet/v2/revokeResolverRoles.ts:113-119`        | `scope: 'root'` → `revokeRootRoles(encodeResolverRoleBitmap(roles), targetAccount)`  |
| `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61`        | the app's builder, which calls it with `roles as ResolverRole[]` — an unchecked cast |
| `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:317-320` | `roles: decodedRoles` — the app's decoded names, admin bits included                 |
| `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:310`     | Remove disabled when `decodedRoles.length === 0`                                     |

The cast at `revokeResolverRoles.ts:52` and `:59` is the seam: `ResolverRoleKey` (the app's 20 names, including admin variants) is asserted to be `ResolverRole` (ensjs's 11 non-admin names). TypeScript is silenced and the mismatch surfaces at runtime. `grantResolverRoles.ts:23` declares `roles: ResolverRole[]` honestly, with no cast, which is why the grant path never hits this.

### The sibling table does not drift, because it does not re-declare anything

The registry roles decoder imports its constants:

```tsx
// apps/portal/src/lib/roles/decodeRoleBitmap.ts:1
import { type Role, registryRoles } from '@ensdomains/ensjs/utils/v2'
```

The resolver roles decoder re-declares them by hand (`resolverRoles.ts:24-45`). That is the whole defect: one table is definitionally in sync with the contract's role set and the other has drifted by one bit. It is a data-table bug in `lib/roles/`, not a logic bug in the sidebar.

### The role is real on chain

`ensPermissionedResolverImpl` on Sepolia, `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`, function selectors present in its runtime code:

```
FOUND  setData(bytes32,string,bytes)                    4eb9c45e
FOUND  authorizeDataRoles(bytes,string,address,bool)    1a76b72c
FOUND  authorizeNameRoles(bytes,uint256,address,bool)   bbd9abb5
FOUND  grantRootRoles(uint256,address)                  072d5d77
FOUND  revokeRootRoles(uint256,address)                 ce156e82
FOUND  roles(uint256,address)                           5adf4724
FOUND  hasRoles(uint256,uint256,address)                d3bf89b1
```

***

## Preempting the likely closure arguments

**"The owner can just replace the resolver."** They can, and that is the answer the project already gave to a different report about resolver roles surviving a transfer. It does not apply to what is claimed here. The claim is that the Explorer offers a control whose stated purpose is to remove an account's access, which either throws or under-delivers, and then reports a state — *this account holds no roles* — that is false. A user who sees "no roles" has been told they do not need to replace the resolver.

**"A throwing button is a functional bug."** On its own it would be. It is reported here because it is the same defect as the silent under-revoke: one hand-maintained table that has drifted from the contract, feeding both the display and the calldata. The two halves fail in opposite directions from the same line.

**"Who would hold `ROLE_SET_DATA`?"** Everyone the product provisions. `DEDICATED_RESOLVER_ROLE_BITMAP` is `ROLES.ALL`, and every name registered through either app grants it to the registrant at the resolver root. The Explorer's roles page is where a later owner goes to withdraw it.

**Recoverability.** Not through the Explorer, by construction: the role is invisible to its decoder, absent from its permission list, and excluded from every bitmap it can build. Recovery requires calling `revokeRootRoles` outside the app with a bitmap the app cannot produce — and even then `SET_DATA_ADMIN` lets the account re-grant itself until that specific bit is taken too.

***

## Not a duplicate

* **#91306** (*Resolver-roles sidebar signs a different scope than it displays*) is `ResolverRolesSidebar.tsx:169` — `roleName = resolvedNames.find(n => n !== '(root)') ?? ''`, a **resource** defect: which scope the write targets. This is a **role-set** defect: which bits the write carries, and which bits the table can decode. Different line, different value, different root cause. Fixing the scope sentinel does not add `ROLE_SET_DATA` to the app's table or stop the admin-name cast from throwing.
* **#92542** (*"Remove user" revokes transfer rights it never discloses*) is `RolesSidebar.tsx:125-141`, the **registry** roles surface, and its defect is revoking **more** than the sheet shows. This is the **resolver** surface revoking **less**, from a different table in a different file.
* **#89279** (*Seller retains resolver control after transferring a name*) is about roles not moving with the token, and was answered with `registry.setResolver`. This is about the app's own removal control being unable to express the roles that exist.
* **#92540** (`useRoleHistory` filtering by label hash) is the history panel's query.
* The row-identity defect reported separately (`ResolverRolesSidebar.tsx:179-184`, the checkbox map keyed on the table row index) is in the same component but is a different root cause with a disjoint fix: that one originates in `components/DataTable.tsx:38-47` supplying no `getRowId`, and is about *whose* permission set is applied. This one is about *which roles the app can express at all*. Neither failure mode disappears when the other is repaired.
* No known issue mentions resolver roles. QA-01 / WEB-1257 / WEB-1305 concern registry role combinations after a transfer.

***

## Recommendation

1. Delete `apps/portal/src/lib/roles/resolverRoles.ts`'s bitmap table and import the constants from `@ensdomains/ensjs/utils/v2` so the app and the encoder can never diverge again.
2. Add `ROLE_SET_DATA` to `resolverPermissions` so it is visible, grantable and revocable.
3. Remove the `roles as ResolverRole[]` casts at `revokeResolverRoles.ts:52` and `:59`; type the boundary so an admin name cannot reach `encodeResolverRoleBitmap`. (`grantResolverRoles.ts:23` already declares `roles: ResolverRole[]` with no cast — the revoke helper is the only one that silences the mismatch.)
4. Handle admin bits explicitly. Either extend ensjs's `resolverRoleMap` with the `_ADMIN` names, or have the app shift the manager bitmap by 128 itself — but revoke both halves, so a removed account cannot re-grant itself.
5. Fail loudly on an unknown role. `decodeResolverRoleBitmap` should report residual bits it does not recognise rather than dropping them, so an account with an unknown role never renders as having none.

***

## Proof of Concept

## Proof of Concept

Executed against a local Anvil fork of Sepolia. No live network was touched. Every value below comes from the app's own modules.

### Environment

```bash
anvil --fork-url <sepolia> --chain-id 11155111 --port 8545
cast chain-id --rpc-url <http://127.0.0.1:8545>          # 11155111
```

```
ETHRegistrar                 0xa88553F454b77203B0D036A05c894d555EAAa2Cc
ETHRegistry (v2)             0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
MockUSDC                     0x768F42455A2D082E23ceeF7d51e5787C82d67a39
ensPermissionedResolverImpl  0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
```

### The app's own decoder and its own revoke builder

`apps/portal/.poc/resolver-roles.test.ts` Run it with:

```bash
cd apps/portal && ./node_modules/.bin/vitest run .poc/resolver-roles.test.ts --reporter=verbose
```

```tsx
/**
 * PoC — apps/portal resolver roles.
 *
 * `ResolverRolesSidebar` builds "Remove user" from `decodedRoles`, which comes
 * from the app's OWN role table (`@/lib/roles/resolverRoles`). That table is a
 * strict subset of the PermissionedResolver role set ensjs defines, and the
 * role NAMES it produces are fed straight into ensjs's encoder by the app's own
 * `prepareRevokeResolverRolesTransaction`.
 */
import { createWalletClient, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { it } from 'vitest'
import { prepareRevokeResolverRolesTransaction } from '@/features/resolver/helpers/revokeResolverRoles'
import {
  decodeResolverRoleBitmap,
  resolverPermissions,
  resolverRoles,
} from '@/lib/roles/resolverRoles'
import { sepoliaWithEns } from '@/lib/wagmi'

// contracts-v2 `ROLES.ALL` — every nibble set. Exactly what the registration
// flow grants the registrant at the resolver root
// (registration.actors.ts:71-73 DEDICATED_RESOLVER_ROLE_BITMAP; smart-account
// manifest.ts ROLES_ALL).
const ROLES_ALL = 0x1111111111111111111111111111111111111111111111111111111111111111n
const SET_DATA = 1n << 36n
const SET_DATA_ADMIN = SET_DATA << 128n

const RESOLVER = '0x1111111111111111111111111111111111111111' as const

it('the Explorer can neither see nor revoke the roles the registration flow grants', () => {
  const account = privateKeyToAccount(`0x${'33'.repeat(32)}`)
  const walletClient = createWalletClient({
    account,
    chain: sepoliaWithEns,
    transport: http('<http://127.0.0.1:8545>'),
  })

  console.log('app resolverPermissions (all the sheet can show or grant):')
  console.log('  ', resolverPermissions.map((p) => p.key).join(', '))
  console.log('app resolverRoles keys:', Object.keys(resolverRoles).length)
  console.log('  ROLE_SET_DATA in table?      ', 'ROLE_SET_DATA' in resolverRoles)
  console.log('  ROLE_SET_DATA_ADMIN in table?', 'ROLE_SET_DATA_ADMIN' in resolverRoles)

  console.log('\n--- 1. an account holding only SET_DATA renders as holding nothing ---')
  console.log('decodeResolverRoleBitmap(SET_DATA | SET_DATA_ADMIN) =',
    JSON.stringify(decodeResolverRoleBitmap(SET_DATA | SET_DATA_ADMIN)))

  console.log('\n--- 2. what the app decodes from the bitmap registration actually grants ---')
  const decoded = decodeResolverRoleBitmap(ROLES_ALL)
  console.log('decodeResolverRoleBitmap(ROLES_ALL) ->', decoded.length, 'role names')

  console.log('\n--- 3. "Remove user" feeds exactly that list to the app\'s own builder ---')
  try {
    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: '',                      // scope: root — the sidebar\'s sentinel
      account: account.address,
      roles: decoded,
      walletClient,
      chainId: sepoliaWithEns.id,
    })
    console.log('built calldata:', JSON.stringify(intent).slice(0, 200))
  } catch (e) {
    console.log('prepareRevokeResolverRolesTransaction THREW:',
      `${(e as Error).name}: ${(e as Error).message}`)
  }

  console.log('\n--- 4. the same builder with only the non-admin roles (what a plain row yields) ---')
  const plain = decoded.filter((r) => !r.endsWith('_ADMIN'))
  const intent = prepareRevokeResolverRolesTransaction({
    resolverAddress: RESOLVER,
    name: '',
    account: account.address,
    roles: plain,
    walletClient,
    chainId: sepoliaWithEns.id,
  })
  const j = (o: unknown) => JSON.stringify(o, (_k, v) => (typeof v === 'bigint' ? v.toString() : v))
  console.log('intent shape:', j(intent).slice(0, 320))
  const anyIntent = intent as unknown as Record<string, unknown>
  const data = (anyIntent.data
    ?? (anyIntent.request as Record<string, unknown> | undefined)?.data) as string
  // revokeRootRoles(uint256 roleBitmap, address account)
  const bitmap = BigInt('0x' + data.slice(10, 74))
  console.log('revokeRootRoles bitmap =', '0x' + bitmap.toString(16))
  const remaining = ROLES_ALL & ~bitmap
  console.log('roles remaining after that revoke = 0x' + remaining.toString(16))
  console.log('  ROLE_SET_DATA still held?      ', (remaining & SET_DATA) !== 0n)
  console.log('  ROLE_SET_DATA_ADMIN still held?', (remaining & SET_DATA_ADMIN) !== 0n)
  console.log('  ...and the roles table decodes the remainder as:',
    JSON.stringify(decodeResolverRoleBitmap(remaining)))
})
```

Actual output:

```
app resolverPermissions (all the sheet can show or grant):
   ROLE_SET_ADDR, ROLE_SET_TEXT, ROLE_SET_CONTENTHASH, ROLE_SET_PUBKEY, ROLE_SET_ABI,
   ROLE_SET_INTERFACE, ROLE_SET_NAME, ROLE_SET_ALIAS, ROLE_CLEAR, ROLE_UPGRADE
app resolverRoles keys: 20
  ROLE_SET_DATA in table?       false
  ROLE_SET_DATA_ADMIN in table? false

--- 1. an account holding only SET_DATA renders as holding nothing ---
decodeResolverRoleBitmap(SET_DATA | SET_DATA_ADMIN) = []

--- 2. what the app decodes from the bitmap registration actually grants ---
decodeResolverRoleBitmap(ROLES_ALL) -> 20 role names

--- 3. "Remove user" feeds exactly that list to the app's own builder ---
prepareRevokeResolverRolesTransaction THREW: TypeError: Cannot mix BigInt and other types, use explicit conversions

--- 4. the same builder with only the non-admin roles (what a plain row yields) ---
intent shape: {"type":"custom","request":{"type":"eoa","from":"0x5CbDd86a2FA8Dc4bDdd8a8f69dBa48572EeC07FB",
  "to":"0x1111111111111111111111111111111111111111",
  "data":"0xce156e8200000000000000000000000000000000100000000000000000000001111111110000000000000000000000005cbdd86a2fa8dc4bddd8a8f69dba48572eec07fb",
  "value":"0","chainId":11155111 …
revokeRootRoles bitmap = 0x10000000000000000000000111111111
roles remaining after that revoke = 0x1111111111111111111111111111111101111111111111111111111000000000
  ROLE_SET_DATA still held?       true
  ROLE_SET_DATA_ADMIN still held? true
  ...and the roles table decodes the remainder as: ["ROLE_SET_ADDR_ADMIN","ROLE_SET_TEXT_ADMIN",
     "ROLE_SET_CONTENTHASH_ADMIN","ROLE_SET_PUBKEY_ADMIN","ROLE_SET_ABI_ADMIN","ROLE_SET_INTERFACE_ADMIN",
     "ROLE_SET_NAME_ADMIN","ROLE_SET_ALIAS_ADMIN","ROLE_CLEAR_ADMIN","ROLE_UPGRADE_ADMIN"]

 ✓ .poc/resolver-roles.test.ts  13ms
```

`0xce156e82` is `revokeRootRoles(uint256,address)`. The bitmap it carries, `0x10000000000000000000000111111111`, covers ten manager bits and `ROLE_UPGRADE`. Bit 36 and bit 164 are not in it.

### The on-chain side

```bash
$ cast code 0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e --rpc-url <http://127.0.0.1:8545> | \
    grep -c 4eb9c45e      # setData(bytes32,string,bytes)
1
$ … grep -c 1a76b72c      # authorizeDataRoles(bytes,string,address,bool)
1
```

***
