# 91641 w a critical explorer resolver role editor writes grants and revokes to a wrongly derived eac resource so revoking a delegate s permissions silently leaves them in place

**Submitted on Sep 3rd 2026 at 08:54:31 UTC by @risinghorse for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91641
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

Under ENSv2 a resolver permission is granted against a specific EAC **resource**, and `resource(0,0)` (all names), `resource(node,0)` (one name) and `resource(node, partHash(key))` (one record key of one name) are three distinct values. The Explorer's Roles page discards which resource each grant came from: `groupRolesByAccount` merges every role row for an account into one table row, and the sidebar then derives a **single** scope for that row with `resolvedNames.find((n) => n !== '(root)') ?? ''`, using it for every grant and revoke it builds. The scope the transaction targets is therefore frequently not the scope the grant lives at — and in the worst case not the scope the UI displays one line above the controls. In production, a name owner who opens the sidebar, unticks every permission for a delegate and clicks Save gets a successful transaction, a refetch and a closed sheet, while the delegate keeps the permission — typically `ROLE_SET_ADDR`, the role that controls where the name points.

***

**Severity:** High **Asset:** `apps/portal` (ENS Explorer app) — in scope per the program's Assets in Scope **Target:** ENS Manager & Explorer — Audit Competition **Audited revision:** `63772fd872af472ced58b009499355f3430c2a86`

**Selected impact:** *Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient* — the *"showing the wrong … detail on a transaction the user then approves"* limb. The terminal state also reaches the sub-bullet *"resolving or rendering an ENS name to an attacker-controlled address"*; I am claiming High rather than Critical because that outcome is conditional on a prior delegation (see **Impact Details**).

> **Testing & rules compliance.** No testing was performed against mainnet or any public testnet. The defect is in pure, exported client-side functions; the proof of concept runs entirely offline. No public-testnet or mainnet contract was registered, called, or written to.

***

## Vulnerability Details

### 1. The indexer supplies per-role name attribution; the app's own type drops it

`apps/portal/src/features/resolver/hooks/useResolverOverview.ts:14-22`

```ts
export type ResolverRole = {
  readonly account: string
  readonly resource: string
  readonly roleBitmap: string
  readonly blockNumber: number
  readonly transactionHash: string | null
  readonly timestamp: number | null
  readonly name: string | null      // <-- per-role name, supplied by the indexer
}
```

and it is requested in the GraphQL selection set (`useResolverOverview.ts:90-98`):

```graphql
roles {
  account
  resource
  roleBitmap
  blockNumber
  transactionHash
  timestamp
  name
}
```

But the grouping helper's input type omits it (`apps/portal/src/lib/roles/resolverRoles.ts:162-166`):

```ts
type RoleInput = {
  readonly account: string
  readonly resource: string
  readonly roleBitmap: string
}
```

The attribution needed to do this correctly is already on the wire and is discarded at the type boundary.

### 2. Grouping merges every resource into one row

`apps/portal/src/lib/roles/resolverRoles.ts:193-215`

```ts
for (const role of roles) {
  const account = role.account.toLowerCase()
  const resource = role.resource.toLowerCase()
  const decoded = decodeResolverRoleBitmap(role.roleBitmap)
  const resolvedName = resourceToName?.get(resource) ?? null

  const existing = grouped.get(account)

  if (existing) {
    existing.roles.push(role)
    existing.decodedRoles.push(...decoded)   // <-- flattened; resource discarded
    if (resolvedName) existing.resolvedNames.add(resolvedName)
  } else {
    // ...
  }
}
```

`decodedRoles` becomes a flat list with no record of which resource each role came from, and `resolvedNames` is an unordered `Set`.

### 3. The name→resource map can only ever resolve `part = 0` resources

`apps/portal/src/lib/roles/resolverRoles.ts:138-141`

```ts
export const computeNameResource = (name: string): string =>
  keccak256(
    encodePacked(['bytes32', 'bytes32'], [namehash(name), ZERO_BYTES32]),
  )
```

`apps/portal/src/lib/roles/resolverRoles.ts:147-160`

```ts
export const buildResourceToNameMap = (
  nodes: readonly { readonly name: string }[],
): Map<string, string> => {
  const map = new Map<string, string>()
  map.set(ROOT_RESOURCE, '(root)')

  for (const node of nodes) {
    if (node.name) {
      map.set(computeNameResource(node.name).toLowerCase(), node.name)
    }
  }

  return map
}
```

Only `ROOT_RESOURCE` and `resource(node, 0)` entries can ever exist, so a **part-scoped** resource is never resolvable and contributes nothing to `resolvedNames`.

### 4. One scope is derived for the whole merged row and drives both mutations

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:166-169`

```ts
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

That single string is what both handlers submit — `handleSaveChanges` (`:294-302`) and `handleRemoveUser` (`:306-321`):

```ts
setPendingAction({
  type: 'save',
  name: roleName,          // ResolverRolesSidebar.tsx:299
  account: selectedAccount,
  rolesToGrant: rolesToGrant,
  rolesToRevoke: rolesToRevoke,
})
```

### 5. The scope selector maps to a different on-chain resource

`apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61`

```ts
const writeParams = revokeResolverRolesWriteParameters(
  walletClient as Parameters<typeof revokeResolverRolesWriteParameters>[0],
  name === ''
    ? { resolverAddress, targetAccount: account, scope: 'root', roles: roles as ResolverRole[] }
    : { resolverAddress, targetAccount: account, scope: 'name', name, roles: roles as ResolverRole[] },
)
```

The convention is documented on the grant side (`grantResolverRoles.ts:20`):

```ts
/** Dotted name (e.g. "myname.eth") or empty string for ROOT_RESOURCE (all names). */
readonly name: string
```

So `roleName === ''` targets `ROOT_RESOURCE` and any non-empty `roleName` targets `resource(namehash(roleName), 0)`. Whichever concrete function `@ensdomains/ensjs` emits for each scope, the two address **different EAC resources** by construction, and neither addresses `resource(node, partHash(key))`.

On-chain the three resources are distinct (`contracts-v2/contracts/src/resolver/libraries/PermissionedResolverLib.sol:71-80`):

```solidity
function resource(bytes32 node, bytes32 part) internal pure returns (uint256 ret) {
    if (node != bytes32(0) || part != bytes32(0)) {
        assembly {
            mstore(0, node)
            mstore(32, part)
            ret := keccak256(0, 64)
        }
    }
}
```

(`resource(0,0)` short-circuits to `0`, i.e. `ROOT_RESOURCE`.)

and `EnhancedAccessControl` clears bits only in the resource it is given (`contracts-v2/contracts/src/access-control/EnhancedAccessControl.sol:146-156`):

```solidity
function revokeRoles(uint256 resource, uint256 roleBitmap, address account)
    public virtual canRevokeRoles(resource, roleBitmap) returns (bool)
{
    if (resource == ROOT_RESOURCE) {
        revert EACRootResourceNotAllowed();
    }
    return _revokeRoles(resource, roleBitmap, account, true);
}
```

*(Note: because `revokeRoles` reverts on `ROOT_RESOURCE`, ensjs must emit `revokeRootRoles` for `scope: 'root'`. I state this as an inference — `node_modules` is not installed at this revision, so I cannot quote ensjs. It does not affect the finding: either way the root path and the name path address different resources.)*

Part-scoped grants are created by the `authorize*Roles` family, e.g. `contracts-v2/contracts/src/resolver/PermissionedResolver.sol:369-388`:

```solidity
function authorizeAddrRoles(bytes calldata toName, uint256 coinType, address account, bool grant)
    external returns (bool updated)
{
    bytes32 node = NameCoder.namehash(toName, 0);
    uint256 roleBit = PermissionedResolverLib.ROLE_SET_ADDR;
    uint256 nodeResource = PermissionedResolverLib.resource(node, bytes32(0));
    uint256 partResource =
        PermissionedResolverLib.resource(node, PermissionedResolverLib.partHash(coinType));
    if (grant) {
        _checkCanGrantRoles(nodeResource, roleBit, msg.sender);
        // ...
        return _grantRoles(partResource, roleBit, account, true);   // <-- granted at partResource
```

and they are honoured at write time by `onlyPartRoles` (`PermissionedResolver.sol:178-187`), which accepts `resource(node, part)` or `resource(0, part)`. A part-scoped grant is therefore fully effective on-chain and entirely invisible to this UI.

### The three divergences

**A. Root + name mix — the UI contradicts its own calldata.** An account holds `ROLE_SET_ADDR` at `ROOT_RESOURCE` (all names) and `ROLE_SET_TEXT` on `foo.eth`. Both decode into one row; `resolvedNames = ['(root)', 'foo.eth']`. `roleName` is `'foo.eth'`, so the revoke is name-scoped and the **root grant survives** — the grant authorising `setAddr` on *every* name on that resolver. The caption for that same row reads (`ResolverRolesSidebar.tsx:389-396`):

```tsx
{selectedAccount && resolvedNames.length > 0 && (
  <p className="text-sm text-muted-foreground">
    {resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
  </p>
)}
```

The UI states the grant is global while building a name-scoped revoke.

**B. Two names — one is silently skipped.** Roles on `foo.eth` and `bar.eth` merge into one row; `roleName` is whichever entered the `Set` first. The revoke lands on that one and the other is untouched, with no indication a second scope existed.

**C. Part-scoped grants — the revoke goes to an unrelated resource.** A grant made with `authorizeAddrRoles` / `authorizeTextRoles` / `authorizeDataRoles` lives at `resource(node, partHash(key))`, which `buildResourceToNameMap` can never resolve. `resolvedNames` is empty, `roleName` is `''`, and the revoke is issued against `ROOT_RESOURCE` — a resource the account may hold nothing at. The part-scoped grant survives untouched.

### Recommended fix

1. **Stop discarding the resource.** Add `name: string | null` (and the raw `resource`) to `RoleInput` (`resolverRoles.ts:162`) and key `groupRolesByAccount` on `(account, resource)` rather than `account` alone — or keep one row per account but carry a per-resource breakdown of `decodedRoles`.
2. **Make the sidebar scope-aware.** Replace the single `roleName` (`ResolverRolesSidebar.tsx:169`) with a per-grant scope and issue one grant/revoke per resource.
3. **Never infer a scope by falling back to `''`.** An unresolvable resource currently degrades silently to `ROOT_RESOURCE`; it should be rendered with its raw resource hash and made non-editable, or block the mutation with an explicit message.
4. **Represent part-scoped grants** so per-record delegations are visible and revocable; today they are neither.

***

## Impact Details

### What an exploit costs the victim

The direct loss is the continued existence of a permission the owner believes they removed, plus the false assurance that they removed it.

* **The owner's only remedy silently fails.** Revocation through the Explorer is the mechanism a name owner has for cutting off a compromised or untrusted delegate. It reports success — transaction confirmed, data refetched, sheet closed — while leaving the permission in place. The owner stops looking.
* **The surviving grant is usually the most powerful one.** In divergence A the grant that survives is the **root-scoped** one: authority over every name on that resolver, not just the one named in the caption. The UI told the user that was exactly what they were removing.
* **Terminal state is address substitution.** The role most commonly delegated here is `ROLE_SET_ADDR`. A delegate who retains it can re-point the name's `addr` records at an address of their choosing at any time, at which point the name resolves to an attacker-controlled address and any funds sent to it are lost. That loss is unbounded and irreversible.
* **Blast radius scales with the resolver.** A `PermissionedResolver` may serve many names; a surviving root grant covers all of them.

### Mapping to the program's in-scope impacts

Claimed:

* ✅ **Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user … submits a transaction to an unintended recipient** — the scope the UI displays and the scope the approved transaction carries are different values.
* ✅ (terminal, conditional) **resolving or rendering an ENS name to an attacker-controlled address** — via the retained `ROLE_SET_ADDR`.

Explicitly **not** claimed:

* ❌ *Malicious interactions with an already-connected wallet / modifying transaction arguments or parameters* — the wrong argument originates in the application's own logic, not in attacker-supplied input; selecting this bullet would misdescribe the mechanism.
* ❌ *Hijacking or bricking an ENS name* — ownership and registry are untouched; only resolver record permissions are affected.
* ❌ *Taking state-modifying authenticated actions on behalf of other users* — the victim signs their own transaction; nothing executes as them.
* ❌ *Direct theft of user funds* — no funds move as a direct result of this bug alone.

### Why High and not Critical

The Critical outcome is real but conditional: the victim must first have delegated a resolver role to the attacker, which is a deliberate act the attacker cannot induce. The unconditional, always-present impact is the failed revocation together with the false success signal. I would rather state that than overstate the report; the project is better placed to judge whether the terminal state lifts the severity.

### Prerequisites

The victim must previously have granted the delegate a resolver role and must then attempt to revoke it. Divergences A and C additionally require the delegate to hold grants at more than one resource, or at a part-scoped resource — both of which the Explorer's own Add-user sheet produces, since its node selector **defaults to the root scope** (`ResolverAddUserSheet.tsx:39`, `const ROOT_NODE_VALUE = ''`, used as initial state at `:127`).

### Duplicate analysis

Not a duplicate of **WEB-716** ("No way to replace an existing subregistry"), which concerns the Registry page offering the deploy-subregistry action only when the slot is empty — different page, different gate, different root cause. The only shared trait is the general class "UI gate disagrees with on-chain permissions."

Also distinct from **QA-01 / WEB-1257** (role combinations breaking follow-up actions after a transfer), which describe flows that *error*. This report describes a transaction that **succeeds** while targeting the wrong resource. The Known Issues document explicitly carves this out: *"a demonstrated security consequence of one of them (for example, a role combination that grants authority it should not, rather than merely erroring) goes beyond the known breakage and is a new finding."* Here authority is not granted in error but **retained** in error, with a false success signal.

No Known Issues entry covers `groupRolesByAccount`, `ResolverRolesSidebar`, or resolver EAC resource derivation.

***

## References

In-scope code (audited revision `63772fd872af472ced58b009499355f3430c2a86`):

* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:166-169` — the single derived scope
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:294-302, 306-321` — both mutations use it
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:389-396` — the "Global roles (all names)" caption
* `apps/portal/src/lib/roles/resolverRoles.ts:138-141` — `computeNameResource`, mirrors `resource(node, 0)`
* `apps/portal/src/lib/roles/resolverRoles.ts:147-160` — `buildResourceToNameMap`
* `apps/portal/src/lib/roles/resolverRoles.ts:162-166` — `RoleInput`, drops `name`
* `apps/portal/src/lib/roles/resolverRoles.ts:179-215` — `groupRolesByAccount`
* `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61` — scope selection
* `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:20` — the documented convention
* `apps/portal/src/features/resolver/hooks/useResolverOverview.ts:14-22, 90-98` — the discarded `name` attribution
* `apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:103` — the grouping call site
* `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:39, 127` — root scope is the default grant

Contracts (reference; the contracts themselves are out of scope, but they define the resource model the app must match):

* `contracts-v2/contracts/src/resolver/libraries/PermissionedResolverLib.sol:71-80` — `resource(node, part)`
* `contracts-v2/contracts/src/resolver/libraries/PermissionedResolverLib.sol:85-90` — `partHash(uint256)`
* `contracts-v2/contracts/src/resolver/PermissionedResolver.sol:178-187` — `onlyPartRoles`
* `contracts-v2/contracts/src/resolver/PermissionedResolver.sol:369-388` — `authorizeAddrRoles` grants at `partResource`
* `contracts-v2/contracts/src/access-control/EnhancedAccessControl.sol:146-156` — `revokeRoles` is per-resource

Documentation:

* ENS documentation — <https://docs.ens.domains/>
* ENSv2 contracts overview (role-based access control, up to 32 roles) — `contracts-v2/contracts/README.md`

## Proof of Concept

## Proof of Concept

The defective logic is in exported **pure functions**, so it is provable offline with no chain and no network.

### Part 1 — Reproduce the wrong scope in the repository's own test runner

Save as `apps/portal/src/lib/roles/poc-h1.test.ts` and run `pnpm test:portal`:

```ts
import { describe, expect, it } from 'vitest'
import { encodePacked, keccak256 } from 'viem'
import { namehash } from 'viem/ens'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from './resolverRoles'

// Exactly the derivation used at ResolverRolesSidebar.tsx:169
const deriveScope = (resolvedNames: readonly string[]) =>
  resolvedNames.find((n) => n !== '(root)') ?? ''

const ROOT_RESOURCE = `0x${'0'.repeat(64)}`
const MALLORY = '0x1234567890abcdef1234567890abcdef12345678'
const NAME = 'alice.eth'

describe('H1 — resolver role revoke targets the wrong EAC resource', () => {
  it('A: a root-scoped grant is revoked name-scoped, so it survives', () => {
    const groups = groupRolesByAccount(
      [
        // granted at ROOT_RESOURCE -> authorises setAddr on EVERY name
        { account: MALLORY, resource: ROOT_RESOURCE,
          roleBitmap: `0x${resolverRoles.ROLE_SET_ADDR.toString(16)}` },
        // granted at resource(namehash('alice.eth'), 0)
        { account: MALLORY, resource: computeNameResource(NAME),
          roleBitmap: `0x${resolverRoles.ROLE_SET_TEXT.toString(16)}` },
      ],
      buildResourceToNameMap([{ name: NAME }]),
    )

    expect(groups).toHaveLength(1)                        // two resources -> ONE row
    const row = groups[0]
    expect(row.resolvedNames).toContain('(root)')         // caption: "Global roles (all names)"

    const scope = deriveScope(row.resolvedNames)
    expect(scope).toBe(NAME)                              // ...but the tx is name-scoped
    expect(scope).not.toBe('')                            // '' would be the ROOT_RESOURCE path
  })

  it('C: a part-scoped grant resolves to no name, so the revoke goes to ROOT_RESOURCE', () => {
    // resource(namehash(name), partHash(60)) — exactly what authorizeAddrRoles
    // grants at (PermissionedResolver.sol:376-377; partHash(uint256) at :85-90)
    const partHash60 = keccak256(encodePacked(['uint256'], [60n]))
    const partResource = keccak256(
      encodePacked(['bytes32', 'bytes32'], [namehash(NAME), partHash60]),
    )

    const groups = groupRolesByAccount(
      [{ account: MALLORY, resource: partResource,
         roleBitmap: `0x${resolverRoles.ROLE_SET_ADDR.toString(16)}` }],
      buildResourceToNameMap([{ name: NAME }]),
    )

    expect(groups[0].resolvedNames).toEqual([])           // part resources are unresolvable
    expect(deriveScope(groups[0].resolvedNames)).toBe('') // -> scope 'root', not the part resource
  })
})
```

Test A shows the caption and the calldata disagreeing; test C shows a part-scoped grant being revoked at an unrelated resource.

**Verification performed.** `node_modules` is not installed in this checkout, so `vitest` could not be executed directly. I re-implemented `buildResourceToNameMap`, `groupRolesByAccount` (`resolverRoles.ts:147-215`) and the `roleName` derivation (`ResolverRolesSidebar.tsx:169`) verbatim and ran them on the inputs above. Observed output:

```
A rows            : 1
A resolvedNames   : [ '(root)', 'alice.eth' ]
A decodedRoles    : [ 'ROLE_SET_ADDR', 'ROLE_SET_TEXT' ]
A derived scope   : "alice.eth"                 <-- name-scoped revoke
A caption shows   : Global roles (all names)    <-- while the UI claims global

C resolvedNames   : []
C derived scope   : ""                          <-- falls back to scope 'root'
```

Row A is the contradiction in a single frame: the caption asserts the grant is global, and the scope the transaction carries is `alice.eth`.

### Part 2 — Confirm the three resources are distinct (offline computation)

```ts
import { encodePacked, keccak256 } from 'viem'
import { namehash } from 'viem/ens'

const ZERO32 = `0x${'0'.repeat(64)}` as const
const node = namehash('alice.eth')

const rootResource = 0n                                              // resource(0,0) short-circuits
const nameResource = keccak256(encodePacked(['bytes32','bytes32'], [node, ZERO32]))
const partHash60   = keccak256(encodePacked(['uint256'], [60n]))
const partResource = keccak256(encodePacked(['bytes32','bytes32'], [node, partHash60]))

// All three differ -> a revoke aimed at one leaves the other two intact.
console.log(rootResource, nameResource, partResource)
```

This mirrors `PermissionedResolverLib.resource()` (`:71-80`) and `partHash(uint256)` (`:85-90`).

### Part 3 — Optional local-chain confirmation

If an end-to-end check is wanted it can be run against the repository's **local** Anvil fork only (`pnpm e2e:infra:up`, chain on `127.0.0.1:8545`): deploy a `PermissionedResolver`, grant `ROLE_SET_ADDR` at `ROOT_RESOURCE` and `ROLE_SET_TEXT` at `resource(node,0)` to a second account, submit the revoke calldata the app builds for that row, then read `hasRoles(ROOT_RESOURCE, ROLE_SET_ADDR, account)` and observe it is still `true`. **No public testnet or mainnet is involved.**
