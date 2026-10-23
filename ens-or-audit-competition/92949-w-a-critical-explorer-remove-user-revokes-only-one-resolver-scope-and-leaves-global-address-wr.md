# 92949 w a critical explorer remove user revokes only one resolver scope and leaves global address write authority active

**Submitted on Sep 13th 2026 at 12:43:48 UTC by @jh42 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92949
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Intro

ENS Explorer incorrectly revokes resolver permissions when the same delegate holds roles at both the resolver root and an individual ENS name. Explorer merges those separate permissions into one row and tells the owner that **Remove user** will remove the delegate from **all roles**. However the resulting transaction succeeds, but it revokes only the name-scoped permissions and leaves the root permission active.

## Root cause

1. `groupRolesByAccount` groups resolver records only by account and appends decoded roles and resolved resource names into a single `AccountRoleGroup`.
2. `ResolverRolesTable` exposes that group as one editable row.
3. `ResolverRolesSidebar` selects `resolvedNames.find((name) => name !== '(root)')` as the transaction scope.
4. **Remove user** passes the flattened role list and that one name to `revokeResolverRoles`.
5. A nonempty name makes the ENSJS builder choose `scope: 'name'` with no root revocation is prepared or submitted.

## Code references

* [`groupRolesByAccount` merges every resource held by one account into one group](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220)
* [`ResolverRolesTable` renders that account group as one editable row](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L102-L115)
* [`ResolverRolesSidebar` selects the first non-root name from the merged row](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L169)
* [`handleRemoveUser` passes that one name and the flattened role list](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L321)
* [`prepareRevokeResolverRolesTransaction` interprets any nonempty name as name-scoped](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L45-L61)

## User actions

1. Alice intentionally grants Bob root `Set Address` and name-scoped `Set Text` permissions through Explorer.
2. Explorer displays both resources and permissions in one Bob row. Opening it labels the combined state **Global roles (all names)**.
3. Alice clicks **Remove user**, accepts the confirmation that Bob will be removed from **all roles**, and signs Explorer's transaction.
4. Explorer reports success, but the transaction clears only Bob's name-scoped bitmap. His root `Set Address` permission remains active.
5. Bob uses that surviving permission to set `alice-poc.eth`'s Ethereum record to `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`.
6. Subsequent forward resolution of `alice-poc.eth` returns the attacker-controlled address.

## Impact

An owner can believe a delegate was fully removed while the delegate retains global record-writing authority. Until the owner notices and sends an additional root revocation, the former delegate can redirect the ETH address of the victim name or other names using the same resolver. Wallets and applications that perform ordinary forward resolution can then receive the attacker address.

**Detailed screenshots should be present in the report.**

## Remediation

Preserve resource scope when grouping and editing resolver roles. A removal operation should build one revocation per represented resource, for example:

```
revoke root SET_ADDR from Bob
revoke alice-poc.eth SET_TEXT from Bob
```

Gist link: <https://gist.github.com/ashu26jha/385016291c4d9440c4c192f7ff70c826>

## Link to Proof of Concept

<https://gist.github.com/ashu26jha/385016291c4d9440c4c192f7ff70c826>

## Proof of Concept

## Proof of Concept

Run commands:

```bash
pnpm --dir apps/portal exec vitest run src/features/resolver/helpers/resolverRolesRemoval.poc.test.ts
```

Paste the following code:

```typescript
import {
  permissionedResolverAuthorizeNameRolesSnippet,
  permissionedResolverRevokeRootRolesSnippet,
} from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import {
  type Address,
  createWalletClient,
  decodeFunctionData,
  http,
  toFunctionSelector,
  zeroHash,
} from 'viem'
import { sepolia } from 'viem/chains'
import { describe, expect, it } from 'vitest'
import type {
  ResolverNode,
  ResolverRole,
} from '@/features/resolver/hooks/useResolverOverview'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  type ResolverRoleKey,
} from '@/lib/roles/resolverRoles'
import {
  prepareResolverRolesIntent,
  type ResolverRolesAction,
} from './prepareResolverRolesIntent'

const NAME = 'alice-poc.eth'
const RESOLVER = '0x1111111111111111111111111111111111111111'
const ALICE = '0x3333333333333333333333333333333333333333'
const BOB = '0x2222222222222222222222222222222222222222'
const SET_ADDR =
  '0x0000000000000000000000000000000000000000000000000000000000000001'
const SET_TEXT =
  '0x0000000000000000000000000000000000000000000000000000000000000010'

describe('remove-user PoC', () => {
  it('merges root and name roles, then prepares only a name-scoped revocation', () => {
    const nameResource = computeNameResource(NAME)
    const roles: ResolverRole[] = [
      {
        account: BOB,
        resource: zeroHash,
        roleBitmap: SET_ADDR,
        blockNumber: 1,
        transactionHash: `0x${'01'.padStart(64, '0')}`,
        timestamp: 1,
        name: null,
      },
      {
        account: BOB,
        resource: nameResource,
        roleBitmap: SET_TEXT,
        blockNumber: 2,
        transactionHash: `0x${'02'.padStart(64, '0')}`,
        timestamp: 2,
        name: null,
      },
    ]
    const nodes: ResolverNode[] = [
      {
        id: NAME,
        name: NAME,
        owner: { id: ALICE.toLowerCase() },
        resolver: { id: RESOLVER, address: RESOLVER },
      },
    ]

    const [groupedRow] = groupRolesByAccount(
      roles,
      buildResourceToNameMap(nodes),
    )
    expect(groupedRow.resolvedNames).toEqual(['(root)', NAME])
    expect(groupedRow.decodedRoles).toEqual(['ROLE_SET_ADDR', 'ROLE_SET_TEXT'])

    // This is ResolverRolesSidebar's actual scope-selection rule.
    const selectedName =
      groupedRow.resolvedNames.find((name) => name !== '(root)') ?? ''
    const action: ResolverRolesAction = {
      type: 'remove',
      name: selectedName,
      account: groupedRow.account as Address,
      roles: groupedRow.decodedRoles as ResolverRoleKey[],
    }
    const walletClient = createWalletClient({
      account: ALICE,
      chain: sepolia,
      transport: http(),
    })
    const intent = prepareResolverRolesIntent(action, RESOLVER, {
      walletClient,
      chainId: sepolia.id,
    })
    if (intent?.request.type !== 'eoa' || !intent.request.data) {
      throw new Error('Expected one EOA removal transaction')
    }

    const decoded = decodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      data: intent.request.data,
    })
    expect(decoded).toEqual({
      functionName: 'authorizeNameRoles',
      args: ['0x09616c6963652d706f630365746800', 17n, BOB, false],
    })
    expect(intent.request.data.slice(0, 10)).not.toBe(
      toFunctionSelector(permissionedResolverRevokeRootRolesSnippet[0]),
    )

    // The generated transaction targets only the name resource. The root
    // SET_ADDR assignment represented by the same UI row is left untouched.
    expect(selectedName).toBe(NAME)
    expect(roles.find((role) => role.resource === zeroHash)?.roleBitmap).toBe(
      SET_ADDR,
    )
  })
})
```
