# 92474 w a critical improper resource binding in portal remove user leaves root resolver authority untouched

**Submitted on Sep 10th 2026 at 04:51:22 UTC by @Mackerel22862 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92474
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

### Brief/Intro

The Portal aggregates resolver grants solely by account even when those grants belong to different authorization resources, such as the resolver root and a specific ENS name. Its “Remove user” flow then selects one non-root name and prepares a single name-scoped revocation while telling the administrator that the account will be removed from all roles. If the account also holds root `ROLE_SET_ADDR`, a successfully authorized and mined name-scoped transaction does not target that root grant, which remains usable to modify address records.

### Vulnerability Details

`groupRolesByAccount` groups every grant belonging to the same account into one displayed row. It combines the role bitmaps and resolved names without retaining separate removal actions for each underlying authorization resource.

When that row is selected, `ResolverRolesSidebar` derives only one removal scope:

```typescript
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The sidebar then passes that single name together with the union of every displayed role to the revocation helper. The confirmation dialog nevertheless states that the user will be removed “from all roles.”

`revokeResolverRoles` selects root scope only when `name === ''`. A non-empty name therefore produces one call to:

```
authorizeNameRoles(dnsEncodedName, combinedRoleBitmap, account, false)
```

For example, suppose delegate `D` holds:

```
Root resource:       ROLE_SET_ADDR
victim.eth resource: ROLE_SET_TEXT
```

The Portal displays these grants as one account row. Selecting “Remove user” causes the prepared calldata to decode as:

```
function:   authorizeNameRoles
name:       DNS("victim.eth")
roleBitmap: 17 (ROLE_SET_ADDR | ROLE_SET_TEXT)
account:    D
authorized: false
```

This removes those bits only from the `victim.eth` resource. It does not revoke `D`’s separate `ROLE_SET_ADDR` grant from `ROOT_RESOURCE`.

Successful contract execution requires the connected administrator, `A`, to possess sufficient effective admin authority to revoke every bit in the combined bitmap on the selected name resource. Separately, UI reachability requires `A` to hold root `ROLE_SET_ADDR`, which makes the route’s `canManageRoles` query true, and to be different from `D`, because the sidebar hides editing for the connected account’s own row. The vulnerable UI path also requires the resolver overview to include `victim.eth` in its indexed node list, allowing `buildResourceToNameMap` to map the name resource back to `victim.eth`.

The referenced `PermissionedResolver` stores root and name permissions under distinct resources. Its address-record authorization path also accepts applicable root authority. Consequently, `D`’s root `ROLE_SET_ADDR` permission can remain effective even though the Portal represented and confirmed the operation as removing the account from all roles.

The same problem applies when one account has grants for multiple names: the UI creates one row but submits only one name-scoped revocation.

This is an incomplete withdrawal of previously authorized privileges, not acquisition of privileges by an arbitrary external account. The security failure occurs when an administrator intentionally attempts to revoke that existing authority and the Portal prepares a transaction covering only one of the resources represented by the UI.

The scenario does not require control of a governance or strategist address. `D` is an ordinary resolver-role delegate whose previously granted authority the resolver administrator is intentionally attempting to withdraw.

#### Known-issue distinction

The disclosed QA-01 issue concerns name transfers with certain role combinations breaking later profile or primary-name operations. WEB-1305 concerns transferring V2 roles to a V1 name and remaining pending indefinitely. This report does not require a name transfer or a V1 name: its root cause is account-only grouping in the resolver roles page followed by a single-resource revocation for a row representing multiple resources. The scope disclosure also states that a demonstrated security consequence of a role combination can go beyond the listed functional breakage.

### Impact Details

The directly demonstrated impact is incorrect transaction parameterization for an already-connected administrator wallet. The administrator confirms removal from all displayed roles, but the application prepares a transaction targeting only one name resource. If that transaction is authorized and mined, other represented resource grants remain untouched.

If the retained grant is root `ROLE_SET_ADDR`, the removed delegate may continue changing ENS address records. An untrusted or compromised former delegate could point affected names to an attacker-controlled address after the administrator believes its access has been revoked. Applications and users resolving those names could then display or send assets to the attacker-controlled address.

There is no fixed TVL that can be calculated for this issue. Potential funds exposure depends on payments made to affected ENS names while a malicious address record is active.

The PoC confirms the grouping error, component arguments, prepared calldata, and the referenced contract semantics that preserve the separate root grant. It does not claim that a transaction was mined on a live deployment, that an address was changed on mainnet, or that funds were actually transferred. Critical is proposed under the listed connected-wallet transaction-parameter impact because the prepared revocation covers only one resource despite confirmation covering all displayed roles. Subsequent ENS-payment redirection is a conditional consequence; no loss was demonstrated.

### References

* [`groupRolesByAccount` combines grants by account](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220)
* [The real table supplies the indexed resource-to-name map to the grouping function](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L90-L105)
* [The production route supplies roles/nodes and gates management on root `ROLE_SET_ADDR`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/resolver/%24address/roles/index.tsx#L30-L91)
* [The sidebar selects one non-root name](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L170)
* [The sidebar suppresses editing for the connected account’s own row](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L351-L359)
* [The removal action snapshots that one name with the combined roles](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L321)
* [The UI labels a root-containing row as global](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L391-L396)
* [The confirmation says all roles, then starts one removal mutation](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L501)
* [The revocation helper selects only root or one name scope](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L45-L74)
* [Contextual `PermissionedResolver` authorization implementation](https://github.com/ensdomains/contracts-v2/blob/48b3e2d39513b9dd32ef1850877a29009bc807b9/contracts/src/resolver/PermissionedResolver.sol)
* [`EnhancedAccessControl` keeps resource assignments separate and applies root fallback](https://github.com/ensdomains/contracts-v2/blob/48b3e2d39513b9dd32ef1850877a29009bc807b9/contracts/src/access-control/EnhancedAccessControl.sol#L11-L28)
* [Competition scope and public known-issue disclosure](https://immunefi.com/audit-competition/audit-competition-ens/scope/)

## Proof of Concept

### Proof of Concept

No RPC or live-system interaction is required for this source-level reproduction.

Step 1. Use the audited Portal revision at mirror commit `cda79acaad59711b943fc68207ebb3f1d0ff8596`, which records its import from the program’s audited upstream revision `63772fd872af472ced58b009499355f3430c2a86`.

Step 2. Consider an administrator `A` and a resolver delegate account `D`. For the UI flow, `A` and `D` must be different accounts, and `A` must hold root `ROLE_SET_ADDR` so that the route’s `canManageRoles` query succeeds. For the prepared call to execute successfully, `A` must additionally have sufficient effective admin authority to revoke both roles in the combined bitmap on `victim.eth`. Give `D` two grants stored under different resources:

```
ROOT_RESOURCE:
    account = D
    role    = ROLE_SET_ADDR

resource(DNS("victim.eth")):
    account = D
    role    = ROLE_SET_TEXT
```

Step 3. Ensure the resolver overview’s node list includes `{ name: "victim.eth" }`. This is required for the production table to build the resource-name lookup used by the UI.

Step 4. Build the same role entries and lookup used by `ResolverRolesTable`, then pass them through the production grouping function:

```typescript
const delegate = '0x1111111111111111111111111111111111111111'
const rootResource =
  '0x0000000000000000000000000000000000000000000000000000000000000000'
const roleEntries = [
  {
    account: delegate,
    resource: rootResource,
    roleBitmap: '0x1', // ROLE_SET_ADDR
  },
  {
    account: delegate,
    resource: computeNameResource('victim.eth'),
    roleBitmap: '0x10', // ROLE_SET_TEXT
  },
]
const grouped = groupRolesByAccount(
  roleEntries,
  buildResourceToNameMap([{ name: 'victim.eth' }]),
)
```

Step 5. Observe that the function returns one actionable account group representing two different resources. The resulting group contains both `"(root)"` and `"victim.eth"` and the union of both roles. A correct implementation may use separate rows or prepare separate resource-bound transactions, but it must preserve both revocation scopes.

Step 6. In a local component reproduction, wrap the component in a `QueryClientProvider` and render `ResolverRolesSidebar` with a table-row wrapper equivalent to `{ id: delegate, original: grouped[0] }`, `open={true}`, a `setOpen` callback, `canManageRoles={true}`, the connected wallet mocked as `A`, and `resolverAddress` set to the fixture resolver. These are the values the production `/resolver/<resolver-address>/roles/` route and `ResolverRolesTable` supply after the prerequisites above are satisfied. Because a non-root name exists, the sidebar chooses:

```
roleName = "victim.eth"
```

Step 7. Observe that the UI identifies the row as containing global roles.

Step 8. Select “Remove user,” observe that the confirmation dialog says the account will be removed from all roles, select “Remove,” and proceed to transaction preparation.

Step 9. Observe that the removal mutation receives only:

```
name:    "victim.eth"
account: D
roles:   [ROLE_SET_ADDR, ROLE_SET_TEXT]
```

No separate root-resource removal is prepared.

Step 10. Decode the calldata prepared by `revokeResolverRoles`. It is:

```
authorizeNameRoles(
    DNS("victim.eth"),
    17,
    D,
    false
)
```

Step 11. Because the selected name is non-empty, the helper uses the name-scoped function. The calldata does not invoke the root-role revocation path and therefore does not clear `D`’s `ROLE_SET_ADDR` grant from `ROOT_RESOURCE`.

Step 12. If `A` authorizes and mines this prepared call, the referenced resolver code revokes the requested bitmap only from the computed `victim.eth` resource. Its `ROOT_RESOURCE` storage entry for `D` remains unchanged. Because effective authorization ORs root roles with resource-specific roles, `D`’s root `ROLE_SET_ADDR` continues to satisfy the address-setting authorization check.

Step 13. The same source logic applies to grants for two indexed names: the account-only grouping still produces one row, `find` selects one name, and the helper still prepares one name-scoped call. This two-name variant is source-inferred and was not separately executed in the local tests.

## Evidence boundaries

* Confirmed locally: account-only grouping, one-scope component arguments, and encoded name-scoped calldata. The component test mocks the wallet, modal, revoke helper, and indexer synchronization; it verifies the application boundary arguments, not successful mining.
* Confirmed from the pinned contextual resolver source: root and name grants use distinct resources, and address-setting authorization can rely on root authority.
* Not demonstrated: a mined revocation transaction, post-transaction live-chain state, a subsequent malicious address change, or an unintended payment.
* Attacker precondition: the account previously held legitimate resolver authority that the administrator is attempting to revoke.
