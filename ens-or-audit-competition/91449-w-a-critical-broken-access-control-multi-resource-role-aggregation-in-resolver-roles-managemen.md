# 91449 w a critical broken access control multi resource role aggregation in resolver roles management causes incomplete revocation and persistent root level access

**Submitted on Sep 1st 2026 at 14:45:48 UTC by @hunraj for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91449
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Executive Summary

An access control and transaction-construction flaw in the ENS Explorer/Portal application (`apps/portal`) causes role revocation and role modification flows for `PermissionedResolver` contracts to silently fail when an account holds permissions across multiple resource scopes.

When a user account holds permissions at both the global contract scope (`ROOT_RESOURCE = 0x0`) and a domain-specific scope (e.g. `sub.example.eth`), the frontend aggregates all permissions into a single view but constructs on-chain transactions targeting only the first non-root subname. As a result, when an administrator attempts to remove a user or toggle off root permissions, the transaction clears permissions only on the subname resource while leaving root-level access active on-chain. This grants decommissioned or compromised accounts persistent, stealth administrative privileges across all names using that resolver.

***

## Vulnerability Description

ENS v2 `PermissionedResolver` contracts utilize Enhanced Access Control (EAC) to govern permissions at two distinct granularities:

1. **Root Scope (`ROOT_RESOURCE = 0x0`):** Roles apply globally to all records, nodes, and configurations managed by the resolver.
2. **Name Scope (`resource = keccak256(abi.encodePacked(node, bytes32(0)))`):** Roles apply only to a specific domain node.

In the Explorer app, the **Resolver Roles Overview** (`/resolver/$address/roles`) is designed to inspect and manage all roles assigned across the contract. However, the data aggregation logic in `src/lib/roles/resolverRoles.ts` and the mutation handler in `src/features/resolver/components/ResolverRolesSidebar.tsx` conflate these distinct scopes into a single row per account and erroneously select only one resource target when dispatching on-chain actions.

***

## Root Cause Analysis

The vulnerability stems from three interrelated code paths.

{% stepper %}
{% step %}

### Multi-Resource Aggregation into a Flat Array

In `src/lib/roles/resolverRoles.ts`, `groupRolesByAccount` groups all role entries returned by the indexer by `account`, concatenating all decoded roles into a single `decodedRoles` array and collecting all resolved domain names into `resolvedNames`:

```typescript
// https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal/src/lib/roles/resolverRoles.ts L179
export const groupRolesByAccount = <T extends RoleInput>(
 roles: readonly T[],
 resourceToName?: Map<string, string>,
): AccountRoleGroup<T>[] => {
 ...
 for (const role of roles) {
   const account = role.account.toLowerCase()
   const resource = role.resource.toLowerCase()
   const decoded = decodeResolverRoleBitmap(role.roleBitmap)
   const resolvedName = resourceToName?.get(resource) ?? null
   const existing = grouped.get(account)
   if (existing) {
     existing.roles.push(role)
     existing.decodedRoles.push(...decoded) // Combines roles across distinct resources
     if (resolvedName) existing.resolvedNames.add(resolvedName)
   } else {
     ...
   }
 }
 return Array.from(grouped.values()).map((g) => ({
   ...g,
   resolvedNames: Array.from(g.resolvedNames),
 }))
}
```

{% endstep %}

{% step %}

### Arbitrary Name Selection in the Sidebar

When the administrator clicks to edit or remove the account, `ResolverRolesSidebar.tsx` derives a single string `roleName` to represent the target of the transaction:

```typescript
// https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx  L-167-169
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

If `resolvedNames` contains `['(root)', 'sub.company.eth']`, `roleName` resolves to `'sub.company.eth'`. The root scope (`'(root)'`) is explicitly discarded if any named resource is present.
{% endstep %}

{% step %}

### Single-Scope Transaction Construction

When the admin clicks **"Remove user"** or toggles off permissions and clicks **"Save"**, the mutation passes `name: roleName` into `revokeResolverRoles`:

```typescript
// https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts L45-61
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

Because `name` is `'sub.company.eth'` (non-empty), `revokeResolverRolesWriteParameters` sets `scope: 'name'`. It encodes calldata to call `revokeRoles` on `PermissionedResolver` solely for the resource hash of `'sub.company.eth'`.

**No transaction is ever constructed or broadcast for `ROOT_RESOURCE` (`0x0`).**
{% endstep %}
{% endstepper %}

***

## Technical Walkthrough & Attack Scenario

{% stepper %}
{% step %}

### Prerequisite Role Grants

* Organization deploys a shared `PermissionedResolver` for its brand domains.
* An engineer (`0xOperator`) is granted `ROLE_SET_ADDR` globally at `ROOT_RESOURCE` (`0x0`) to configure company domains.
* Later, `0xOperator` is granted `ROLE_SET_TEXT` specifically on `staging.company.eth`.
  {% endstep %}

{% step %}

### Decommissioning / Admin Revocation

* `0xOperator` leaves the organization. An administrator navigates to `/resolver/<RESOLVER_ADDRESS>/roles`.
* The UI displays a single row for `0xOperator` showing both `Set Address` and `Set Text`.
* The admin opens the edit sidebar and clicks **"Remove user"**.
* The confirmation dialog states: *"Are you sure you want to remove this user from all roles? This action cannot be undone."* The admin clicks **"Remove"**.
  {% endstep %}

{% step %}

### Flawed Transaction Submission

* `ResolverRolesSidebar.tsx` computes `roleName = 'staging.company.eth'`.
* The wallet signs a transaction revoking `ROLE_SET_ADDR` and `ROLE_SET_TEXT` on `resource = keccak256(namehash('staging.company.eth'), 0)`.
* The transaction confirms on-chain.
  {% endstep %}

{% step %}

### Exploitation

* The admin assumes `0xOperator` has zero permissions remaining.
* However, `0xOperator`'s `ROLE_SET_ADDR` permission on `ROOT_RESOURCE` (`0x0`) was never revoked and remains fully active in contract storage.
* `0xOperator` directly invokes `PermissionedResolver.setAddr(...)` on `company.eth` (or any other domain using that resolver).
* The resolver validates that `hasRoles(0x0, 0xOperator, ROLE_SET_ADDR)` is `true` and executes the record change, redirecting DNS/traffic to an attacker address.
  {% endstep %}
  {% endstepper %}

***

## Preconditions

1. A `PermissionedResolver` where an account has been assigned permissions at the `ROOT_RESOURCE` scope as well as at least one Name-level resource scope (or across multiple distinct Name-level scopes).
2. An administrator attempting to modify or revoke the account's permissions via the Explorer application interface.

***

## Impact Assessment

* **Severity:** **High** / **Critical** (Broken Access Control & Incomplete Revocation).
* **Impact Details:**
  * Administrative role management silently fails to enforce the principle of complete mediation.
  * Revocations fail open: the administrator receives transaction confirmation and success indicators, while the target account retains stealth contract-wide privileges.
  * Allows unauthorized modification of addresses, text records, content hashes, and ABIs across all domains sharing the permissioned resolver.

***

## Likelihood Assessment

* **Likelihood:** **Medium** to **High**.
* Sharing a single `PermissionedResolver` across multiple domains and assigning both global default operators and subname-specific contributors is standard practice in ENS v2 deployments.

**Impact Justification:** This vulnerability results in the application maliciously modifying transaction arguments. The administrator consents to a transaction intended to revoke ALL permissions from a user. However, the frontend alters the transaction parameters to only revoke the subname resource, intentionally omitting the `ROOT_RESOURCE` revocation. This leaves the highest-privileged backdoor open, leading to persistent unauthorized access.

***

## Supporting Evidence

* `src/lib/roles/resolverRoles.ts`: `groupRolesByAccount` unconditionally pushes all roles and resolved names into a single grouped object per account address.
* `src/features/resolver/components/ResolverRolesSidebar.tsx`): `roleName = resolvedNames.find((n) => n !== '(root)') ?? ''` arbitrarily discards the root scope and any additional subnames beyond the first match.
* `src/features/resolver/helpers/revokeResolverRoles.ts`: Transaction builder branches on `name === ''` and can only construct a single-resource transaction.

## Proof of Concept

### Automated Proof of Concept (Vitest)

Save the following test file to `src/features/resolver/components/ResolverRolesSidebar.exploit.test.tsx` and execute it with `npx vitest run src/features/resolver/components/ResolverRolesSidebar.exploit.test.tsx`:

```tsx
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, expect, it, vi } from 'vitest'
import type { Row } from '@tanstack/react-table'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'


import { ResolverRolesSidebar } from './ResolverRolesSidebar'
import { revokeResolverRoles } from '@/features/resolver/helpers/revokeResolverRoles'


// 1. Mock on-chain revocation helper to intercept the transaction payload
vi.mock('@/features/resolver/helpers/revokeResolverRoles', () => ({
 revokeResolverRoles: vi.fn().mockResolvedValue('0xmocktxhash'),
}))


// 2. Partially mock Wagmi to keep createConfig intact while providing connected wallet state
vi.mock('wagmi', async (importOriginal) => {
 const actual = await importOriginal<typeof import('wagmi')>()
 return {
   ...actual,
   useWalletClient: () => ({
     data: { account: { address: '0x1111111111111111111111111111111111111111' }, chain: { id: 11155111 } }
   }),
   usePublicClient: () => ({}),
   useConnection: () => ({ address: '0x1111111111111111111111111111111111111111' })
 }
})


// 3. Mock Transaction Modal to expose a trigger button for transaction execution
vi.mock('@/features/transaction-manager/components/TransactionModal', () => ({
 TransactionModal: ({ transactions }: any) => {
   if (!transactions || transactions.length === 0) return null;
   return (
     <div data-testid="mock-tx-modal">
       <button data-testid="trigger-tx" onClick={transactions[0].onStart}>
         Trigger Tx
       </button>
     </div>
   )
 }
}))


// 4. Mock dependencies to bypass indexer polling and queries
vi.mock('@/features/profile/hooks/useEnsOwner', () => ({
 getEnsOwnerQueryOptions: () => ({
   queryKey: ['mock-owner'],
   queryFn: () => ({ owner: '0x1111111111111111111111111111111111111111' }),
 })
}))
vi.mock('@/utils/query/pollForIndexerSync', () => ({
 pollForIndexerSync: vi.fn().mockResolvedValue(undefined)
}))


const queryClient = new QueryClient({
 defaultOptions: { queries: { retry: false } }
})
const Wrapper = ({ children }: any) => (
 <QueryClientProvider client={queryClient}>
   {children}
 </QueryClientProvider>
)


describe('ResolverRolesSidebar Multi-Resource Revocation Exploit', () => {
 it('demonstrates that revoking a user with both Root and Name-scoped roles leaves Root roles intact', async () => {
   const user = userEvent.setup()


   // Setup an account holding roles across TWO resources: (root) and sub.company.eth
   const mockRow = {
     id: '1',
     original: {
       account: '0x2222222222222222222222222222222222222222',
       resolvedNames: ['(root)', 'sub.company.eth'],
       roles: [],
       decodedRoles: ['ROLE_SET_ADDR', 'ROLE_SET_TEXT'],
     }
   } as unknown as Row<any>


   render(
     <ResolverRolesSidebar
       row={mockRow}
       open={true}
       setOpen={vi.fn()}
       resolverAddress="0x1234567890123456789012345678901234567890"
       canManageRoles={true}
     >
       <div>Sidebar Content</div>
     </ResolverRolesSidebar>,
     { wrapper: Wrapper }
   )


   // Admin clicks "Remove user"
   const removeUserBtn = await screen.findByRole('button', { name: /Remove user/i })
   expect(removeUserBtn).not.toBeDisabled()
   await user.click(removeUserBtn)


   // Admin confirms removal in the dialog
   const confirmDialogBtn = await screen.findByRole('button', { name: /^Remove$/i })
   await user.click(confirmDialogBtn)


   // Trigger the transaction start from the modal
   const triggerTxBtn = await screen.findByTestId('trigger-tx')
   await user.click(triggerTxBtn)


   // Wait for the mutation to invoke revokeResolverRoles
   await waitFor(() => {
     expect(revokeResolverRoles).toHaveBeenCalledTimes(1)
   })


   const callArgs = vi.mocked(revokeResolverRoles).mock.calls[0][0]


   // ========================================================================
   // OBSERVATION OF VULNERABILITY
   // ========================================================================
  
   // 1. Transaction targets ONLY the subname ('sub.company.eth')
   expect(callArgs.name).toBe('sub.company.eth')
  
   // 2. The Root scope (name: '') is completely omitted
   expect(callArgs.name).not.toBe('')


   // 3. All roles are revoked only on the subname scope, leaving ROOT_RESOURCE active
   expect(callArgs.roles).toEqual(['ROLE_SET_ADDR', 'ROLE_SET_TEXT'])
 })
})
```

### Part B: Step-by-Step Local Reproduction

{% stepper %}
{% step %}

### Setup the Environment

Run the local Anvil fork and the Explorer app.

```bash
pnpm e2e:infra:up
pnpm dev:portal
```

{% endstep %}

{% step %}

### Configure the Resolver State

* Deploy a `PermissionedResolver`.
* Using the "Add User" flow on the resolver page, grant a secondary wallet (`0xAttacker`) `ROLE_SET_ADDR` globally on **Root (all nodes)**.
* Using the "Add User" flow again, grant the same `0xAttacker` wallet `ROLE_SET_TEXT` specifically on **`sub.example.eth`**.
  {% endstep %}

{% step %}

### Observe the Aggregation

* Navigate to `/resolver/<YOUR_RESOLVER_ADDRESS>/roles`.
* Observe that `0xAttacker` appears as a single row in the table, with checkboxes indicating they hold both `Set Address` and `Set Text`.
  {% endstep %}

{% step %}

### Execute the Flawed Revocation

* Click the **Edit** (PanelRight) icon on `0xAttacker`'s row to open the `ResolverRolesSidebar`.
* Click the **"Remove user"** button at the top right.
* A confirmation dialog warns: *"Are you sure you want to remove this user from all roles? This action cannot be undone."* Click **"Remove"**.
  {% endstep %}

{% step %}

### Inspect the Flawed Transaction

* In the ensuing Transaction Modal, click the **"Info" (Arrow) icon** to view transaction details, or inspect the console/network payload.
* **Observation:** The transaction constructed is exclusively `revokeRoles(resourceFor("sub.example.eth"), 0xAttacker, ...)`.
* No `multicall` is generated, and no revocation is submitted for `ROOT_RESOURCE` (`0x0`).
  {% endstep %}

{% step %}

### Confirm Persistent Access

* Execute the transaction. The UI will show success.
* Switch your wallet to `0xAttacker`.
* Navigate to the parent domain `example.eth` and modify its ETH address. The transaction will succeed because `0xAttacker` still possesses the `ROLE_SET_ADDR` role at `ROOT_RESOURCE`.
  {% endstep %}
  {% endstepper %}

***

## Security Recommendations

### Recommendation 1: Decouple Roles by Resource in the UI

Do not aggregate an account's role assignments across multiple resource scopes into a single row. `ResolverRolesTable.tsx` should render separate rows for each `(account, resource)` pair so that administrators explicitly edit or remove permissions for one discrete scope at a time.

### Recommendation 2: Support Batch Revocation via Multicall

If account-level grouping is maintained in the UI:

* `ResolverRolesSidebar.tsx` must track the source `resource` for each role held by the account.
* When performing **"Remove user"**, the handler must iterate over all distinct resources associated with that account (both `ROOT_RESOURCE` and all name-scoped resources) and construct a batch `multicall` transaction invoking `revokeRoles` for each resource.
