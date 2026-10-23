# 92542 w a medium explorer s roles editor revokes role can transfer admin in a remove user whose only warning describes a different consequence freezing the name for the rest of its registrat&#x20;

**Submitted on Sep 10th 2026 at 15:13:01 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92542
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

`/$name/roles` lists every account holding a role on the name and offers a **Remove user** button on each row. The button revokes the **raw decoded role list** for that account — every role the registry reports, not the roles the sheet showed the user.

For an ordinarily registered `.eth` 2LD the owner's decoded role list is

```
ROLE_CAN_TRANSFER_ADMIN, ROLE_SET_RESOLVER, ROLE_SET_RESOLVER_ADMIN,
ROLE_SET_SUBREGISTRY, ROLE_SET_SUBREGISTRY_ADMIN
```

The sheet the user is looking at while they click renders exactly four permissions — Renew, Set Subregistry, Set Resolver, Unregister (`lib/roles/permissions.ts:19-40`) — with its "Admin" checkboxes hard-disabled (`RolesSidebar.tsx:274`). `ROLE_CAN_TRANSFER` cannot appear in that list at all: the list's key type is `Exclude<Role, ${string}_ADMIN>` (`permissions.ts:11`) and ensjs defines *no* non-admin `ROLE_CAN_TRANSFER` — the role exists only as `ROLE_CAN_TRANSFER_ADMIN`.

The confirmation dialog names a different consequence:

> "You are trying to delete the owner of this name. **Deleting it would prohibit you from adding more users.** Are you sure?" — `RolesSidebar.tsx:345`

What happens on-chain is worse, and lasts for the rest of the paid registration term. Executed on a Sepolia fork:

| Call, after the app's own `revokeRoles()` lands                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Result                                                       |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `safeTransferFrom(...)`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | reverts `TransferDisallowed(uint256,address)` — `0xe58f6d5a` |
| `setResolver(labelhash, …)`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | reverts `EACUnauthorizedAccountRoles` — `0x4b27a133`         |
| `setSubregistry(labelhash, …)`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | reverts `EACUnauthorizedAccountRoles` — `0x4b27a133`         |
| `grantRoles(ROLE_CAN_TRANSFER_ADMIN, self)` or `grantRoles(ROLE_SET_RESOLVER_ADMIN, self)`, to undo it                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | reverts `EACCannotGrantRoles` — `0xd1a3b355`                 |
| The owner still holds the ERC-1155 token. For the remainder of the term they cannot move it, sell it, or change what the name points at, and no on-chain actor can put the roles back: revoking an `_ADMIN` role removes the authority needed to re-grant it, the issuing `ETHRegistrar` holds no role on the name resource, and even the account holding the registry's root role bitmap cannot grant on a name resource — all three measured below. The one exit is to stop renewing, let the name expire, and win it back in the release/premium window against everyone else bidding for it: a fresh fee plus the real risk of losing a valuable name to a sniper. |                                                              |

The app knows this role matters: `useCanTransferName`'s own header comment (`useCanTransferName.ts:8-25`) is entirely about `ROLE_CAN_TRANSFER_ADMIN` and `TransferDisallowed`. And the sibling sheet in the same app, `RegistryEditUserSheet`, implements the exact guard that is missing here — `getSoleAdminRoles`, `getRemovableRoles`, a destructive `Alert` naming the roles at stake, a second confirm dialog. `RolesSidebar` has none of them and does no filtering at all.

***

## Impact

Precondition: the connected wallet holds any `_ADMIN` role on the name, which every ordinary 2LD owner does (`NameRolesOverviewTable.tsx:105-107`). That is the same condition that makes "Remove user" appear, so it is not a limitation, it is the mechanism.

The flow is entirely ordinary: a user opens `/<name>.eth/roles`, clicks the edit icon on a row to see what that account can do, and uses the **Remove user** button in the sheet header. If the row is their own — and on a freshly registered name the owner's row is the *only* row — the name is frozen for the rest of its registration term.

The sheet lists four permissions and greys out the Admin column (`lib/roles/permissions.ts:19-40`; `RolesSidebar.tsx:274`), and the dialog says only "Deleting it would prohibit you from adding more users" (`:345`). What is signed is `revokeRoles(resource, 0x1110…0001100000, account)` built from the raw decoded list (`RolesSidebar.tsx:125-141`; `helpers/revokeRoles.ts:58-95`), whose bitmap includes `1 << 156` = `ROLE_CAN_TRANSFER_ADMIN`, under a transaction row reading "Remove user — Remove 0xeD59…1E15 from check90161.eth" (`buildRoleTransactionDescriptors.ts:83-95`).

Afterwards the Explorer reports the damage and then tells the user to do something that cannot be done. `/$name/ownership` stops rendering the Transfer button (`ownership/index.tsx:102`), and `/$name/ownership/transfer` renders:

> "This wallet can't transfer this name. … **Ask whoever issued the name to grant the transfer role (ROLE\_CAN\_TRANSFER\_ADMIN), or transfer from the wallet that holds it.**" — `routes/$name/ownership/transfer.tsx:172-193`

For a `.eth` 2LD there is no issuer with authority to do that and no other wallet holds the role: `grantRoles` reverts `EACCannotGrantRoles` for the owner, for the issuing registrar, and for the registry's root role holder — all three verified below.

A second, independent defect sits in the same handler: `handleRemoveUser` passes `originalRoles` through with **no** `getRemovableRoles(currentRoles, callerAdminRoles)` filtering (`registryRoleAccess.ts:43-47`), which the registry sheet applies before every removal. So when a delegated manager clicks Remove user on a row holding roles they cannot revoke, the whole transaction reverts instead of removing the subset they are permitted to remove — the same omission, failing the other way.

***

## Root cause

### The role reaches the removal payload

| File:line                                                                                | Code                                                                                                                                      |
| ---------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `features/roles/hooks/useNameRoleAccounts.ts:108`, `lib/roles/decodeRoleBitmap.ts:14-25` | `decodeRoleBitmap(data.newRoleBitmap)` iterates **all** of `registryRoles`, so `ROLE_CAN_TRANSFER_ADMIN` is in the list                   |
| `features/roles/components/RolesTable.tsx:48-50`                                         | `.map(([account, roleNames]) => ({ account, items: roleNames }))`                                                                         |
| `features/roles/components/RolesSidebar.tsx:88-91`                                       | `const originalRoles = useMemo(() => (row?.original.items ?? []) as Role[], [row])`                                                       |
| `features/roles/components/RolesSidebar.tsx:125-141`                                     | `handleRemoveUser` → `setPendingRemove({ account: selectedAccount, roles: originalRoles })` — no filter, no lockout check                 |
| `features/roles/utils/buildRoleTransactionDescriptors.ts:83-95`                          | one `revoke` descriptor titled `Remove user`, carrying those roles verbatim                                                               |
| `features/roles/helpers/revokeRoles.ts:58-95`                                            | `revokeRolesWriteParameters({ registryAddress, account, resource, roles })` → `encodeRoleBitmap` → `revokeRoles(uint256,uint256,address)` |

### The role does not reach the surface the user acts on

| File:line                                             | Code                                                                                                                                                                                      |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `lib/roles/permissions.ts:11`                         | `type PermissionKey = Exclude<Role,` ${string}\_ADMIN`>` — `ROLE_CAN_TRANSFER_ADMIN` has no non-admin variant in ensjs, so it can never be a `PermissionKey`                              |
| `lib/roles/permissions.ts:19-40`                      | `permissions` = `ROLE_RENEW`, `ROLE_SET_SUBREGISTRY`, `ROLE_SET_RESOLVER`, `ROLE_UNREGISTER` — four entries                                                                               |
| `features/roles/components/RolesSidebar.tsx:214`      | `{permissions.map((permission, index) => { … })}` — the sheet renders only those four                                                                                                     |
| `features/roles/components/RolesSidebar.tsx:274`      | the Admin `<Checkbox … disabled />`, unconditionally                                                                                                                                      |
| `features/roles/components/RolesAddUserSheet.tsx:213` | the Add-user sheet's Admin `<Checkbox … disabled />`, also unconditional                                                                                                                  |
| `features/roles/hooks/useEditedPermissions.ts:15-20`  | `setEditedPermissions(roleToPermissions(roles))` seeded from the full `items` — why **Save** leaves undisplayed roles alone and Remove user does not                                      |
| `features/roles/components/RolesSidebar.tsx:339-347`  | the confirm dialog: *"Deleting it would prohibit you from adding more users."* / *"remove this user from all roles"* — neither mentions transferability, the resolver, or the subregistry |

ensjs states the role's nature directly (`@ensdomains/ensjs/dist/utils/v2/roles/registryRoles.js`):

```js
/** Nybble 7 — authorizes ERC1155 token transfers. Admin-only (checked on token owner). */
const ROLE_CAN_TRANSFER_ADMIN = (1n << 28n) << 128n;
```

### The same app implements the missing guard next door

| File:line                                                           | Code                                                                                                                                                                          |
| ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `features/registry/components/v2/RegistryEditUserSheet.tsx:124-138` | `isSelfEdit`, `selfSoleAdminRoles = getSoleAdminRoles(rolesData, account)`, `willLockOutAdmin`                                                                                |
| `.../RegistryEditUserSheet.tsx:155-157`                             | *"we deliberately do NOT block when `willLockOutAdmin`. The destructive Alert above is the consent contract — the user is told exactly what will happen and gets to decide."* |
| `.../RegistryEditUserSheet.tsx:161-176`                             | `const removableRoles = getRemovableRoles(currentRoles, callerAdminRoles)` … `if (removeWillLockOutAdmin) { setIsRemoveConfirmOpen(true); return }`                           |
| `.../RegistryEditUserSheet.tsx:295-304`, `:324-334`                 | *"You're the only Admin for this role. Removing it will permanently lock out admin control."*                                                                                 |
| `features/registry/utils/registryRoleAccess.ts:43-68`               | `getRemovableRoles`, `getSoleAdminRoles` — pure, unit-tested helpers, **not imported by `RolesSidebar`**                                                                      |

`grep -n "getRemovableRoles\|getSoleAdminRoles" apps/portal/src/features/roles/` returns nothing: `RolesSidebar` imports neither helper.

***

## Preempting the likely closure arguments

**"The user clicked Remove user — they consented."** They consented to what the app told them: a sheet listing four permissions with the Admin column greyed out, and a dialog naming a *different, lesser* consequence — "would prohibit you from adding more users", not "this name cannot be transferred, and its resolver and subregistry cannot be changed, until it expires". The destroyed role is structurally excluded from that sheet (`permissions.ts:11`). The project set the standard itself, in the sibling sheet: *"The destructive Alert above is the consent contract — the user is told exactly what will happen and gets to decide"* (`RegistryEditUserSheet.tsx:155-157`). That contract is not honoured here.

**"This needs an un-prompted action outside the normal workflow" (`RULES.md:1006`).** It needs neither. Remove user is a shipped, prompted, first-class control in the header of every row's sheet, on the page whose stated job is managing who holds roles on a name, and the app puts its own confirmation dialog in front of it. Removing a role holder is the normal workflow; the abnormal part is what the app signs when the user takes it.

**"Only a name's own owner can do this, to themselves."** Correct, and this report leads with it rather than waiting to be caught on it (`RolesAddUserSheet.tsx:213`, `RolesSidebar.tsx:274`, `NameRolesOverviewTable.tsx:105-107`). The claim is not that an attacker steals a name; it is that the application composes and pre-fills a transaction that does not match what it displayed, and the user approves it — the `RULES.md:891` shape, which #91306 established in this program at Critical with no third-party attacker either.

**"Is it recoverable?"** Not within the term, and not by anyone: `grantRoles` reverts `EACCannotGrantRoles` for the owner, because revoking an `_ADMIN` role removes the authority to re-grant it; the issuing `ETHRegistrar` holds no role on the name; the registry's root role holder cannot grant on a name resource. Recovery exists only through expiry and re-registration. Compare a resolver mis-set, which `registry.setResolver` fixes in one transaction — a path this finding itself closes.

**"That's just the registry behaving correctly."** It is; the contract did exactly what it was told. The defect is entirely in the app, which composed a role bitmap containing a role it did not display on the surface where the user acted, under a label ("Remove user") and a warning that describe something else.

**"The table does show a 'Can Transfer' row."** It does — read-only, in the *table* (`rolesToEntries`, `roleTableColumns.tsx:29-43`). But the sheet the user is inside when they press the button lists four permissions and does not include it, and neither the button label nor either confirmation string mentions transferability. The gap is between what is shown at the moment of the destructive action and what that action does.

**"Then the whole roles editor is broken."** No — and the narrowness is evidence, not luck. The Save path is safe: `useEditedPermissions` seeds from the row's full `items` (`useEditedPermissions.ts:15-20`), so undisplayed roles survive a Save untouched. One control, `handleRemoveUser`, bypasses that and revokes the raw list.

***

## Not a duplicate

**QA-01** (`RULES.md:342`, `:766-770`) — "Certain role combinations when transferring a name, or a name with subnames, cause errors when later updating profile records or setting a primary name." This is not a transfer, and nothing errors: the transaction succeeds, and its success is the damage. Neither element of that entry is present, so the report is not inside it to begin with. Secondarily, `RULES.md:764` carves out demonstrated security consequences of known functional breakage — but that carve-out is not needed here. The fix is unrelated too: disclose and filter the revoke set in `RolesSidebar`.

**WEB-1305** (`:252`) — transferring V2 name roles to a V1 name hangs pending: a stuck transaction on a V1 target; here the transaction succeeds, on a V2 name. **WEB-1257** (`:261`) — after a parent transfer its registry is not detached, so a previous owner can't set a subname as their primary name: different trigger, different symptom, fully recoverable, and not the roles editor. **WEB-1249** (`:288`) — creating a subname can replace an existing child registry: different route, different call, and recoverable by re-pointing the subregistry — a call this finding removes for the rest of the term. **WEB-716** (`:246`, a missing deploy affordance), **WEB-1259 / QA-04** (fuses on wrapped V1 names), **EXP-INPUT-005**, **SEC-MGR-\*** and **R3-01…R3-08** — none touch role encoding, the roles sidebar, or `revokeRoles`.

**The raw-vs-normalized family** (#89270, #90163, #90190, #91837, #92043) — those are name-derivation divergences: one expression of a name is displayed and another is transacted. Here there is exactly one name, correctly derived on both sides; the divergence is between the *set of roles displayed* and the *set of roles revoked*.

**Report #91306** — cited above as precedent for the impact row, and *not* the same defect. That one is the *resolver* roles editor (`ResolverRolesSidebar`, `PermissionedResolver`, `computeResolverResource`), and its bug is the resource/scope argument, which widens authority to every name the resolver serves. This is the *registry name* roles sidebar (`RolesSidebar`, `PermissionedRegistry`), the resource is correct, and the bug is the role bitmap on the removal path plus the missing lockout disclosure. Different file, different contract, different argument, opposite direction, different fix.

**Reports #91838 / #91839** (Manager migration: a stale controller keeps `ROLE_SET_RESOLVER`; a token-scoped approve widened into `setApprovalForAll`) — both *grant* authority too broadly, in the Manager's v1→v2 migration batch. This *revokes* authority the owner cannot restore, in the Explorer.

***

## Recommendation

1. **Filter the removal set.** `RolesSidebar.handleRemoveUser` should use the helpers the project already wrote and tested — `getRemovableRoles(originalRoles, callerAdminRoles)` (`registryRoleAccess.ts:43-47`) — so the revoke contains only roles the caller may revoke and cannot revert wholesale.
2. **Port the lockout guard.** Apply `getSoleAdminRoles` / `willLockOutAdmin` and the destructive `Alert` + confirm dialog from `RegistryEditUserSheet.tsx:124-180, 294-304, 324-334`, so removing an `_ADMIN` role the account is the sole holder of is disclosed by name before it is signed.
3. **Do not revoke a role the surface did not display.** Intersect the removal set with the roles the sheet renders, or extend the sheet to render every held role — including admin-only ones like `ROLE_CAN_TRANSFER_ADMIN`, which `permissions.ts:11`'s `Exclude<Role, ${string}_ADMIN>` key type makes impossible to express. A `PermissionKey` union that admits admin-only roles, or a separate "irreversible permissions" section, fixes the structural cause.
4. **Fail closed on transferability.** Losing `ROLE_CAN_TRANSFER_ADMIN` cannot be undone for the life of the registration, so treat it like fuse-burning: a dedicated, explicitly-worded confirmation ("this name cannot be transferred again until it expires and is re-registered"), not a generic "remove from all roles".
5. **Correct the post-hoc advice.** `routes/$name/ownership/transfer.tsx:180-190` tells 2LD owners to "ask whoever issued the name" — for a 2LD the issuing registrar holds no role on the name and cannot re-grant. Distinguish that case and say plainly that the state lasts until the registration expires.

## Proof of Concept

## Proof of Concept

Executed against a local Anvil fork of Sepolia on `127.0.0.1:8651`. No live network was touched beyond forked reads.

### Environment

```bash
anvil --fork-url <sepolia-archive-rpc> --fork-block-number 11669614 \
      --chain-id 11155111 --port 8651 --silent &
cast chain-id --rpc-url http://127.0.0.1:8651     # 11155111
```

```
ETHRegistrar  0xa88553F454b77203B0D036A05c894d555EAAa2Cc
ETHRegistry   0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
MockUSDC      0x768F42455A2D082E23ceeF7d51e5787C82d67a39
```

### Setup

`roles-setup.sh` — registers one ordinary `.eth` 2LD to one ordinary EOA. Nothing unusual is configured.

```bash
#!/usr/bin/env bash
set -euo pipefail
R=http://127.0.0.1:8651
REGISTRAR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
REG=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
FK=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
Z=0x0000000000000000000000000000000000000000
ZH=0x0000000000000000000000000000000000000000000000000000000000000000

OWNER_KEY=0x6161616161616161616161616161616161616161616161616161616161616161
OWNER=$(cast wallet address --private-key $OWNER_KEY)
echo "OWNER=$OWNER"
# anvil_setCode clears the EIP-7702 delegation this vanity key carries on real
# Sepolia; then ETH and mock USDC are funded and approved to the registrar.
cast rpc anvil_setCode $OWNER 0x --rpc-url $R >/dev/null
cast rpc anvil_setBalance $OWNER 0x21e19e0c9bab2400000 --rpc-url $R >/dev/null
cast send $USDC "mint(address,uint256)" $OWNER 100000000000 --private-key $FK --rpc-url $R >/dev/null
cast send $USDC "approve(address,uint256)" $REGISTRAR 100000000000 --private-key $OWNER_KEY --rpc-url $R >/dev/null

L=${1:-brickme90161}
SEC=0x9999999999999999999999999999999999999999999999999999999999999999
C=$(cast call $REGISTRAR "makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)" \
      "$L" "$OWNER" "$SEC" $Z $Z 31536000 $ZH --rpc-url $R)
cast send $REGISTRAR "commit(bytes32)" "$C" --private-key $OWNER_KEY --rpc-url $R >/dev/null
cast rpc evm_increaseTime 120 --rpc-url $R >/dev/null
cast rpc evm_mine --rpc-url $R >/dev/null
cast send $REGISTRAR "register(string,address,bytes32,address,address,uint64,address,bytes32)" \
      "$L" "$OWNER" "$SEC" $Z $Z 31536000 $USDC $ZH --private-key $OWNER_KEY --rpc-url $R >/dev/null
LH=$(cast keccak "$L")
echo "registered $L.eth -> $OWNER"
echo "labelhash=$LH"
echo "tokenId=$(cast call $REG 'getTokenId(uint256)(uint256)' $LH --rpc-url $R | awk '{print $1}')"
echo "roles=$(cast call $REG 'roles(uint256,address)(uint256)' $LH $OWNER --rpc-url $R | awk '{print $1}')"
```

Actual output:

```
OWNER=0xeD5970f07eea5A9E7b359D3c5d3F82b34a8F1E15
registered check90161.eth -> 0xeD5970f07eea5A9E7b359D3c5d3F82b34a8F1E15
labelhash=0x4bed1609a2939fe6156ff404c78c307962c05da978ea768813fe356456d7b9dd
tokenId=34342358496277926101065531611843707563916929360209218694363242627803318845440
roles=97409655027181761882228017414928043058140282880
```

`97409655027181761882228017414928043058140282880` = `0x1110000000000000000000000000000001100000`: a plain registration grants `ROLE_SET_SUBREGISTRY(+ADMIN)`, `ROLE_SET_RESOLVER(+ADMIN)` and `ROLE_CAN_TRANSFER_ADMIN`, and nothing else.

### The flow, driven through the app's own functions

`apps/portal/.poc/90161-roles-remove-user-bricks-transfer.test.ts`. Every function on the destructive path is imported from the shipped source; the only substitution is the RPC endpoint (`@/lib/wagmi` re-exported with `wagmiConfig.getClient()` pointed at the fork, as `vite.config.ts` does for local dev). The owner is impersonated with `anvil_autoImpersonateAccount` because the transaction manager's EOA transport submits with `eth_sendTransaction`; in the real flow the owner signs the same calldata in their wallet.

Run with:

```
POC_NAME=check90161.eth POC_RPC_URL=http://127.0.0.1:8651 \
  pnpm --filter portal exec vitest run \
  .poc/90161-roles-remove-user-bricks-transfer.test.ts --reporter=verbose
```

```ts
/**
 * PoC — apps/portal `/$name/roles` "Remove user". Nothing here re-implements
 * the app: decodeRoleBitmap (what RolesTable rows carry as `items`),
 * roleToPermissions + permissions (what RolesSidebar renders),
 * buildRoleTransactionDescriptors (what "Remove user" queues), revokeRoles
 * (submitted through the real @ens-apps/transaction-manager),
 * getHasRolesQueryOptions (the gate useCanTransferName reads) and
 * prepareGrantRolesTransaction are all imported from the shipped source.
 */
import type { Role } from '@ensdomains/ensjs/utils/v2'
import { decodeFunctionData, erc1155Abi, parseAbi } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { expect, it, vi } from 'vitest'

const RPC = vi.hoisted(() => process.env.POC_RPC_URL ?? 'http://127.0.0.1:8615')

vi.mock('@/lib/wagmi', async (importActual) => {
  const actual = await importActual<typeof import('@/lib/wagmi')>()
  const { createClient, http } = await import('viem')
  const forkClient = createClient({ chain: actual.sepoliaWithEns, transport: http(RPC) })
  return {
    ...actual,
    wagmiConfig: { ...actual.wagmiConfig, getClient: () => forkClient },
  }
})

// ---- the app, unmocked ----------------------------------------------------
import { getEnsOwner } from '@/features/profile/hooks/useEnsOwner'
import { getEnsTokenId } from '@/features/profile/hooks/useTokenId'
import { getHasRolesQueryOptions } from '@/features/registry/hooks/useHasRoles'
import { createEOASigner } from '@/features/registry/utils/signer.helpers'
import { prepareGrantRolesTransaction } from '@/features/roles/helpers/grantRoles'
import { prepareRevokeRolesTransaction, revokeRoles } from '@/features/roles/helpers/revokeRoles'
import { buildRoleTransactionDescriptors } from '@/features/roles/utils/buildRoleTransactionDescriptors'
import { decodeRoleBitmap } from '@/lib/roles/decodeRoleBitmap'
import { permissions } from '@/lib/roles/permissions'
import { roleToPermissions } from '@/lib/roles/rolesToPermissions'
import { sepoliaWithEns } from '@/lib/wagmi'

const NAME = process.env.POC_NAME ?? 'handover90161.eth'
const LABEL = NAME.split('.')[0]
const OWNER_KEY = '0x6161616161616161616161616161616161616161616161616161616161616161' as const
const BUYER = '0x000000000000000000000000000000000000bEEF' as const

const registryAbi = parseAbi([
  'function roles(uint256 resource, address account) view returns (uint256)',
  'function getTokenId(uint256 labelHash) view returns (uint256)',
])

const hr = (t: string) => console.log(`\n======== ${t} ${'='.repeat(Math.max(0, 56 - t.length))}`)

const rpc = async (method: string, params: unknown[] = []) => {
  const res = await fetch(RPC, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  })
  const json = (await res.json()) as { result?: unknown; error?: unknown }
  if (json.error) throw new Error(`${method}: ${JSON.stringify(json.error)}`)
  return json.result as never
}

it(
  'the app\'s "Remove user" strips ROLE_CAN_TRANSFER_ADMIN, which it does not display, and the name cannot be transferred for the rest of the registration term',
  { timeout: 300_000 },
  async () => {
    const { createPublicClient, createWalletClient, http, labelhash } = await import('viem')

    hr('GUARD — local anvil fork only')
    const url = new URL(RPC)
    expect(['127.0.0.1', 'localhost', '[::1]']).toContain(url.hostname)
    const clientVersion = await rpc('web3_clientVersion')
    expect(String(clientVersion)).toMatch(/anvil/i)
    expect(Number(await rpc('eth_chainId'))).toBe(11155111)
    const nodeInfo = (await rpc('anvil_nodeInfo')) as { forkConfig?: { forkBlockNumber?: number } }
    console.log(`  ${RPC}: ${clientVersion}, chainId 11155111, fork block ${nodeInfo?.forkConfig?.forkBlockNumber}`)

    // The transaction manager's EOA transport submits with `eth_sendTransaction`,
    // so the sender must be unlocked. The owner signs willingly in the real flow.
    await rpc('anvil_autoImpersonateAccount', [true])

    const publicClient = createPublicClient({ chain: sepoliaWithEns, transport: http(RPC) })
    const owner = privateKeyToAccount(OWNER_KEY)
    const walletClient = createWalletClient({ account: owner, chain: sepoliaWithEns, transport: http(RPC) })
    const signer = createEOASigner(walletClient)

    hr('1. the app resolves the name (getEnsOwner)')
    const ownerRes = await getEnsOwner({ name: NAME })
    expect(ownerRes.isOk()).toBe(true)
    const ownerData = ownerRes._unsafeUnwrap()
    if (!ownerData) throw new Error('name not registered — run the setup script')
    const registryAddress = ownerData.registryAddress
    console.log('  name           :', NAME)
    console.log('  owner          :', ownerData.owner)
    console.log('  registry       :', registryAddress)
    expect(ownerData.owner.toLowerCase()).toBe(owner.address.toLowerCase())

    const resource = BigInt(labelhash(LABEL))
    const rawBitmap = await publicClient.readContract({
      address: registryAddress, abi: registryAbi, functionName: 'roles', args: [resource, owner.address],
    })
    console.log('  on-chain roles :', `0x${rawBitmap.toString(16)}`)

    hr('2. RolesTable row.original.items  (app: decodeRoleBitmap)')
    const items = decodeRoleBitmap(rawBitmap)
    console.log('  items =', items)
    expect(items).toContain('ROLE_CAN_TRANSFER_ADMIN')

    hr('3. RolesSidebar — what the user is shown before clicking')
    const perms = roleToPermissions(items)
    console.log('  roleToPermissions(items) keys =', [...perms.keys()])
    const renderedKeys = permissions.map((p) => p.key)
    console.log('  lib/roles/permissions.ts renders =', renderedKeys)
    const hidden = [...perms.keys()].filter((k) => !(renderedKeys as string[]).includes(k))
    console.log('  held but not rendered in the sidebar =', hidden)
    expect(hidden).toContain('ROLE_CAN_TRANSFER')

    hr('4. "Remove user" → buildRoleTransactionDescriptors(null, pendingRemove)')
    const descriptors = buildRoleTransactionDescriptors(null, { account: owner.address, roles: items as Role[] }, NAME)
    console.log('  descriptor.title           :', descriptors[0].title)
    console.log('  descriptor.transactionName :', descriptors[0].transactionName)
    console.log('  descriptor.roles           :', descriptors[0].roles)

    hr('5. the calldata (app: prepareRevokeRolesTransaction)')
    const intent = prepareRevokeRolesTransaction({
      name: NAME, account: owner.address, roles: descriptors[0].roles,
      walletClient, chainId: sepoliaWithEns.id, registryAddress,
    })
    const decoded = decodeFunctionData({
      abi: parseAbi(['function revokeRoles(uint256 resource, uint256 roleBitmap, address account)']),
      data: intent.request.data,
    })
    const bitmap = decoded.args[1] as bigint
    const CAN_TRANSFER_ADMIN = (1n << 28n) << 128n
    console.log('  to        :', intent.request.to)
    console.log('  fn        :', decoded.functionName)
    console.log('  resource  :', decoded.args[0])
    console.log('  roleBitmap:', `0x${bitmap.toString(16)}`)
    console.log('  bitmap contains ROLE_CAN_TRANSFER_ADMIN (1<<156)?', (bitmap & CAN_TRANSFER_ADMIN) === CAN_TRANSFER_ADMIN)
    expect(bitmap & CAN_TRANSFER_ADMIN).toBe(CAN_TRANSFER_ADMIN)

    hr('6. useCanTransferName gate BEFORE (app: getHasRolesQueryOptions)')
    const gate = async () => {
      const q = getHasRolesQueryOptions({
        registryAddress, label: LABEL, roles: ['ROLE_CAN_TRANSFER_ADMIN'] as Role[], account: owner.address,
      })
      return await (q.queryFn as (ctx: unknown) => Promise<unknown>)({ queryKey: q.queryKey } as never)
    }
    console.log('  canTransfer =', await gate())

    const tokenIdRes = await getEnsTokenId({ label: LABEL, registryAddress })
    const tokenId = tokenIdRes._unsafeUnwrap()
    console.log('  tokenId     =', tokenId.toString())
    const simTransfer = async () =>
      publicClient.simulateContract({
        account: owner.address, address: registryAddress, abi: erc1155Abi,
        functionName: 'safeTransferFrom', args: [owner.address, BUYER, tokenId, 1n, '0x'],
      })
    await simTransfer()
    console.log('  safeTransferFrom simulates OK — the name is sellable')

    hr('7. SUBMIT — the app helper revokeRoles(), real transaction manager')
    const revoked = await revokeRoles({
      name: NAME, account: owner.address, roles: descriptors[0].roles, walletClient, publicClient,
      signer, chainId: sepoliaWithEns.id, registryAddress, id: 'poc-90161-remove-user',
    })
    console.log('  tx hash =', revoked.hash)
    const after = await publicClient.readContract({
      address: registryAddress, abi: registryAbi, functionName: 'roles', args: [resource, owner.address],
    })
    console.log('  on-chain roles after =', `0x${after.toString(16)}`)
    expect(after).toBe(0n)

    hr('8. AFTER')
    console.log('  useCanTransferName gate =', await gate())
    expect(await gate()).toBe(false)

    const freshTokenId = (await getEnsTokenId({ label: LABEL, registryAddress }))._unsafeUnwrap()

    let transferErr = ''
    try {
      await publicClient.simulateContract({
        account: owner.address, address: registryAddress, abi: erc1155Abi,
        functionName: 'safeTransferFrom', args: [owner.address, BUYER, freshTokenId, 1n, '0x'],
      })
    } catch (e) {
      transferErr = (e as Error).message
    }
    console.log('  safeTransferFrom now:', transferErr.split('\n').filter(Boolean).slice(0, 4).join(' | '))
    expect(transferErr).not.toBe('')

    hr('9. the recovery the UI tells the user to ask for')
    const restore = prepareGrantRolesTransaction({
      name: NAME, account: owner.address, roles: ['ROLE_CAN_TRANSFER_ADMIN'] as Role[],
      walletClient, chainId: sepoliaWithEns.id, registryAddress,
    })
    let restoreErr = ''
    try {
      await publicClient.call({ account: owner.address, to: restore.request.to, data: restore.request.data })
    } catch (e) {
      restoreErr = (e as Error).message
    }
    console.log('  grantRoles(ROLE_CAN_TRANSFER_ADMIN) by the owner:', restoreErr.split('\n').filter(Boolean).slice(0, 4).join(' | '))
    expect(restoreErr).toMatch(/EACCannotGrantRoles|reverted/i)
  },
)
```

Actual output (noise lines from the transaction manager's IndexedDB warning, the WalletConnect core banner and the vitest banner removed; nothing else edited):

```
======== GUARD — local anvil fork only ===========================
  http://127.0.0.1:8651: anvil/v1.6.0, chainId 11155111, fork block 11669614

======== 1. the app resolves the name (getEnsOwner) ==============
  name           : check90161.eth
  owner          : 0xeD5970f07eea5A9E7b359D3c5d3F82b34a8F1E15
  registry       : 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
  on-chain roles : 0x1110000000000000000000000000000001100000

======== 2. RolesTable row.original.items  (app: decodeRoleBitmap)
  items = [
  'ROLE_CAN_TRANSFER_ADMIN',
  'ROLE_SET_RESOLVER',
  'ROLE_SET_RESOLVER_ADMIN',
  'ROLE_SET_SUBREGISTRY',
  'ROLE_SET_SUBREGISTRY_ADMIN'
]

======== 3. RolesSidebar — what the user is shown before clicking
  roleToPermissions(items) keys = [ 'ROLE_CAN_TRANSFER', 'ROLE_SET_RESOLVER', 'ROLE_SET_SUBREGISTRY' ]
  lib/roles/permissions.ts renders = [ 'ROLE_RENEW', 'ROLE_SET_SUBREGISTRY', 'ROLE_SET_RESOLVER', 'ROLE_UNREGISTER' ]
  held but not rendered in the sidebar = [ 'ROLE_CAN_TRANSFER' ]

======== 4. "Remove user" → buildRoleTransactionDescriptors(null, pendingRemove)
  descriptor.title           : Remove user
  descriptor.transactionName : Remove 0xeD59…1E15 from check90161.eth
  descriptor.roles           : [
[… 5 lines elided — the same five role names printed in step 2 …]
]

======== 5. the calldata (app: prepareRevokeRolesTransaction) ====
  to        : 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
  fn        : revokeRoles
  resource  : 34342358496277926101065531611843707563916929360209218694363242627803318845440n
  roleBitmap: 0x1110000000000000000000000000000001100000
  bitmap contains ROLE_CAN_TRANSFER_ADMIN (1<<156)? true

======== 6. useCanTransferName gate BEFORE (app: getHasRolesQueryOptions)
  canTransfer = true
  tokenId     = 34342358496277926101065531611843707563916929360209218694363242627803318845440
  safeTransferFrom simulates OK — the name is sellable

======== 7. SUBMIT — the app helper revokeRoles(), real transaction manager
📊 [TRANSACTION MANAGER] Transaction poc-90161-remove-user state: submitting
📊 [TRANSACTION MANAGER] Transaction poc-90161-remove-user state: pending
📊 [TRANSACTION MANAGER] Transaction poc-90161-remove-user state: success
  tx hash = 0x72c22b164730d0533a2b5bf192d17cc0fdfb487369fb6caa2b625333efa00ff2
  on-chain roles after = 0x0

======== 8. AFTER ================================================
  useCanTransferName gate = false
  safeTransferFrom now: The contract function "safeTransferFrom" reverted with the following signature: | 0xe58f6d5a | Unable to decode signature "0xe58f6d5a" as it was not found on the provided ABI. | Make sure you are using the correct ABI and that the error exists on it.

======== 9. the recovery the UI tells the user to ask for ========
  grantRoles(ROLE_CAN_TRANSFER_ADMIN) by the owner: Execution reverted with reason: custom error 0xd1a3b355: 4bed1609a2939fe6156ff404c78c307962c05da978ea768813fe3564000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000ed5970f07eea5a9e7b359d3c5d3f82b34a8f1e15. | Raw Call Arguments: |   from:  0xeD5970f07eea5A9E7b359D3c5d3F82b34a8F1E15 |   to:    0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2

 ✓ .poc/90161-roles-remove-user-bricks-transfer.test.ts (1 test) 587ms
 Test Files  1 passed (1)
      Tests  1 passed (1)
```

`0xe58f6d5a` is `TransferDisallowed(uint256,address)` and `0xd1a3b355` is `EACCannotGrantRoles(uint256,uint256,address)`:

```
$ cast sig "TransferDisallowed(uint256,address)"           -> 0xe58f6d5a
$ cast sig "EACCannotGrantRoles(uint256,uint256,address)"  -> 0xd1a3b355
$ cast sig "EACUnauthorizedAccountRoles(uint256,uint256,address)" -> 0x4b27a133
$ cast 4byte 0xe58f6d5a
TransferDisallowed(uint256,address)
```

`TransferDisallowed` is the exact error `useCanTransferName.ts:9-11` names as the thing the whole transfer gate exists to avoid.

## The rest of the freeze, and everyone who cannot undo it

The same "Remove user" also took `ROLE_SET_RESOLVER(_ADMIN)` and `ROLE_SET_SUBREGISTRY(_ADMIN)`. On the same fork state and the same name, this script asks three parties to repair the damage: the owner, the `ETHRegistrar` that issued the name, and the holder of the registry's root role bitmap at `ROOT_RESOURCE` (which reads `0`; the holder is `0x84d3a426d4e12e955d1df95db0b24fe26afe39d3`, bitmap printed below). The last two are impersonated with `anvil_impersonateAccount` — strictly generous to the defence, since it grants authority the attacker-free real world would still have to obtain.

```bash
# vars as in roles-setup.sh ($R $REG $REGISTRAR), plus:
ROOTADMIN=0x84d3a426d4e12e955d1df95db0b24fe26afe39d3
O=0xeD5970f07eea5A9E7b359D3c5d3F82b34a8F1E15
LH=0x4bed1609a2939fe6156ff404c78c307962c05da978ea768813fe356456d7b9dd   # check90161
SRA=$(python3 -c "print((1<<24)<<128)")   # ROLE_SET_RESOLVER_ADMIN
CTA=$(python3 -c "print((1<<28)<<128)")   # ROLE_CAN_TRANSFER_ADMIN
err() { grep -o "custom error 0x[0-9a-f]\{8\}" | head -1; }

echo "roles(name, owner) = $(cast call $REG 'roles(uint256,address)(uint256)' $LH $O --rpc-url $R | awk '{print $1}')"
echo; echo "-- the owner, on their own name --"
echo -n "setResolver                       -> "; cast call $REG "setResolver(uint256,address)" $LH $REGISTRAR --from $O --rpc-url $R 2>&1 | err
echo -n "setSubregistry                    -> "; cast call $REG "setSubregistry(uint256,address)" $LH $REGISTRAR --from $O --rpc-url $R 2>&1 | err
echo -n "grantRoles(ROLE_SET_RESOLVER_ADMIN) -> "; cast call $REG "grantRoles(uint256,uint256,address)" $LH $SRA $O --from $O --rpc-url $R 2>&1 | err
echo; echo "-- the .eth registrar that issued the name --"
echo "roles(ROOT=0, registrar)          = $(cast call $REG 'roles(uint256,address)(uint256)' 0 $REGISTRAR --rpc-url $R | awk '{print $1}')"
echo "roles(name,   registrar)          = $(cast call $REG 'roles(uint256,address)(uint256)' $LH $REGISTRAR --rpc-url $R | awk '{print $1}')"
cast rpc anvil_impersonateAccount $REGISTRAR --rpc-url $R >/dev/null
cast rpc anvil_setBalance $REGISTRAR 0x21e19e0c9bab2400000 --rpc-url $R >/dev/null
echo -n "registrar grantRoles(CAN_TRANSFER_ADMIN, owner) -> "; cast call $REG "grantRoles(uint256,uint256,address)" $LH $CTA $O --from $REGISTRAR --rpc-url $R 2>&1 | err
echo; echo "-- the registry's root super-admin --"
echo "roles(ROOT=0, 0x84d3…39d3)        = $(cast call $REG 'roles(uint256,address)' 0 $ROOTADMIN --rpc-url $R)"
echo "roles(name,   0x84d3…39d3)        = $(cast call $REG 'roles(uint256,address)(uint256)' $LH $ROOTADMIN --rpc-url $R | awk '{print $1}')"
cast rpc anvil_impersonateAccount $ROOTADMIN --rpc-url $R >/dev/null
cast rpc anvil_setBalance $ROOTADMIN 0x21e19e0c9bab2400000 --rpc-url $R >/dev/null
echo -n "root-admin grantRoles(CAN_TRANSFER_ADMIN, owner) -> "; cast call $REG "grantRoles(uint256,uint256,address)" $LH $CTA $O --from $ROOTADMIN --rpc-url $R 2>&1 | err
cast send $REG "grantRoles(uint256,uint256,address)" $LH $CTA $O --from $ROOTADMIN --unlocked --rpc-url $R >/dev/null 2>&1
echo "root-admin real send, roles(name, owner) after = $(cast call $REG 'roles(uint256,address)(uint256)' $LH $O --rpc-url $R | awk '{print $1}')"
```

Actual output:

```
roles(name, owner) = 0

-- the owner, on their own name --
setResolver                       -> custom error 0x4b27a133
setSubregistry                    -> custom error 0x4b27a133
grantRoles(ROLE_SET_RESOLVER_ADMIN) -> custom error 0xd1a3b355

-- the .eth registrar that issued the name --
roles(ROOT=0, registrar)          = 65537
roles(name,   registrar)          = 0
registrar grantRoles(CAN_TRANSFER_ADMIN, owner) -> custom error 0xd1a3b355

-- the registry's root super-admin --
roles(ROOT=0, 0x84d3…39d3)        = 0x0100000000000000000000100001011101000000000000000000001000000100
roles(name,   0x84d3…39d3)        = 0
root-admin grantRoles(CAN_TRANSFER_ADMIN, owner) -> custom error 0xd1a3b355
root-admin real send, roles(name, owner) after = 0
```

Three negative results; the second and third are the interesting ones:

* The **ETHRegistrar** that mints every `.eth` 2LD holds `roles(ROOT_RESOURCE, registrar) = 65537` — `ROLE_REGISTRAR | ROLE_RENEW`, no `_ADMIN` bit — and **zero** roles on the name resource. It can register and renew; it cannot re-grant.
* The registry's **root role holder** cannot either: its enormous root bitmap does not reach a per-name resource. The impersonated `grantRoles` reverts `0xd1a3b355`, and a real `cast send` from that account leaves `roles(name, owner)` at `0`. **Root roles do not cascade to per-name resources on this registry.** Not the owner, not the issuing registrar, and not the account holding the registry's root roles: on-chain `roles(resource, owner) == 0x0`, and it stays there for as long as the registration lives.

## What recovery costs, and what still works

The freeze is bounded by the registration term, and it is worth being exact about that rather than overclaiming. On a second name (`recoverb90161.eth`), registered the same way, this run revokes the exact bitmap "Remove user" builds, then warps the fork past the one-year term plus the grace and premium windows and re-registers:

```bash
# vars as in the scripts above; L=recoverb90161, LH=$(cast keccak "$L"), BEEF=0x…bEEF
BM=0x1110000000000000000000000000000001100000   # the bitmap "Remove user" builds (PoC step 5)
sim() { cast call $REG "safeTransferFrom(address,address,uint256,uint256,bytes)" $O $BEEF \
        "$(cast call $REG 'getTokenId(uint256)(uint256)' $LH --rpc-url $R | awk '{print $1}')" 1 0x \
        --from $O --rpc-url $R 2>&1 | head -1; }

cast send $REG "revokeRoles(uint256,uint256,address)" $LH $BM $O --private-key $K --rpc-url $R >/dev/null
echo "-- after the app's Remove-user bitmap is revoked --"
echo "roles(name, owner)                 = $(cast call $REG 'roles(uint256,address)(uint256)' $LH $O --rpc-url $R | awk '{print $1}')"
echo "safeTransferFrom simulation        = $(sim)"
echo; echo "-- warp 480 days (1-year term + grace + premium decay), then re-register --"
cast rpc evm_increaseTime 41472000 --rpc-url $R >/dev/null
cast rpc evm_mine --rpc-url $R >/dev/null
# … same commit + register as roles-setup.sh with a fresh secret, ending in
#   && echo "re-registration by the same owner: OK" (9 lines elided) …
echo "roles(name, owner)                 = $(cast call $REG 'roles(uint256,address)(uint256)' $LH $O --rpc-url $R | awk '{print $1}')"
echo "safeTransferFrom simulation        = $(sim)"
```

Actual output:

```
-- after the app's Remove-user bitmap is revoked --
roles(name, owner)                 = 0
safeTransferFrom simulation        = Error: server returned an error response: error code 3: execution reverted: custom error 0xe58f6d5a: e130939e2d933582c741e6146f991264d494fcf90049612523c978e000000001000000000000000000000000ed5970f07eea5a9e7b359d3c5d3f82b34a8f1e15

-- warp 480 days (1-year term + grace + premium decay), then re-register --
re-registration by the same owner: OK
roles(name, owner)                 = 97409655027181761882228017414928043058140282880
safeTransferFrom simulation        = 0x
```

The full role set — `ROLE_CAN_TRANSFER_ADMIN` included — comes back on re-registration. So the honest statement of impact is: the name is **frozen for the remainder of the paid registration term** (up to a full year from the click, longer if the owner had renewed ahead), during which it cannot be transferred, sold, or re-pointed, and the **only** exit is to stop renewing, let the registration lapse, and re-register the label once it is released. On a private fork that is uncontested; in production the expiry of every `.eth` name is public, release is watched by snipers, and the premium window is an auction the previous owner may lose. For a name with any value that is a **probable total loss**, plus a second registration fee and the abandoned remainder of the term — a bounded, quantifiable loss, not a claim of permanence.

Nor is the name inaccessible: the revoke removes the authority to *change* the resolver, not the resolver already stored. On a third name registered with a resolver (`stillresolves90161.eth`; the address is arbitrary):

```bash
# … same header and registration as the scripts above, but registering
# `stillresolves90161.eth` with RESOLVER=0x8FADE66B79cC9f707aB26799354482EB93a5B7dD
# passed as the resolver argument (22 lines elided) …
echo "before  getResolver(\"$L\") = $(cast call $REG 'getResolver(string)(address)' "$L" --rpc-url $R)"
cast send $REG "revokeRoles(uint256,uint256,address)" $LH $BM $O --private-key $K --rpc-url $R >/dev/null
echo "after   roles(name, owner)       = $(cast call $REG 'roles(uint256,address)(uint256)' $LH $O --rpc-url $R | awk '{print $1}')"
echo "after   getResolver(\"$L\") = $(cast call $REG 'getResolver(string)(address)' "$L" --rpc-url $R)"
echo -n "owner setResolver now           -> "
cast call $REG "setResolver(uint256,address)" $LH $Z --from $O --rpc-url $R 2>&1 | grep -o "custom error 0x[0-9a-f]\{8\}" | head -1
echo -n "owner setSubregistry now        -> "
cast call $REG "setSubregistry(uint256,address)" $LH $Z --from $O --rpc-url $R 2>&1 | grep -o "custom error 0x[0-9a-f]\{8\}" | head -1
```

Actual output:

```
before  getResolver("stillresolves90161") = 0x8FADE66B79cC9f707aB26799354482EB93a5B7dD
after   roles(name, owner)       = 0
after   getResolver("stillresolves90161") = 0x8FADE66B79cC9f707aB26799354482EB93a5B7dD
owner setResolver now           -> custom error 0x4b27a133
owner setSubregistry now        -> custom error 0x4b27a133
```

The name is therefore **untransferable and unmanageable**, not inaccessible: it keeps resolving to whatever it resolved to before, and the owner keeps the token — they simply cannot move it or change what it points at until the term ends.
