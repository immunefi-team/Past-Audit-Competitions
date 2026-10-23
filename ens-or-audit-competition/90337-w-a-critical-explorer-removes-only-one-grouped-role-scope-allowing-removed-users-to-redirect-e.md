# 90337 w a critical explorer removes only one grouped role scope allowing removed users to redirect ens transfers

**Submitted on Aug 24th 2026 at 14:33:43 UTC by @abyy for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90337
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

### Brief/Intro

The Explorer combines a user’s resolver permissions across root and name-specific resources into one row, but its **Remove user from all roles** workflow revokes only one name-specific resource. A removed user can therefore retain global `ROLE_SET_ADDR`, change an ENS name to an attacker-controlled address through Explorer, and cause Explorer to display and use that address as the recipient of another user’s ENS name transfer.

### Vulnerability Details

PermissionedResolver roles are assigned per `(account, resource, bitmap)`. Root roles and name-scoped roles are separate on-chain assignments.

Explorer’s `groupRolesByAccount()` instead groups assignments solely by account. It merges all represented resources into `resolvedNames` and unions their decoded role bitmaps into `decodedRoles`:

```ts
const account = role.account.toLowerCase()
const decoded = decodeResolverRoleBitmap(role.roleBitmap)

const existing = grouped.get(account)

if (existing) {
  existing.roles.push(role)
  existing.decodedRoles.push(...decoded)
  if (resolvedName) existing.resolvedNames.add(resolvedName)
}
```

A delegate with these assignments therefore appears as one user:

```
(root):        ROLE_SET_ADDR = 1
recipient.eth: ROLE_SET_TEXT = 16
```

The grouped row represents both resources and both roles. If it contains a root assignment, Explorer labels it:

```
Global roles (all names)
```

The removal dialog further states:

```
Are you sure you want to remove this user from all roles?
```

However, `ResolverRolesSidebar` reduces the multi-resource row to one transaction scope:

```ts
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

Whenever the grouped row contains a name, `roleName` becomes the first non-root name. Root is selected only when no named resource exists.

`handleRemoveUser()` then creates a single removal action using that one name and the unioned roles:

```ts
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

For the example above, Explorer constructs the equivalent of:

```
authorizeNameRoles(
  recipient.eth,
  ROLE_SET_ADDR | ROLE_SET_TEXT,
  delegate,
  false
)
```

This revokes bitmap `17` only from `recipient.eth`. It does not revoke the delegate’s root resource, so root `ROLE_SET_ADDR` remains active.

The transaction succeeds and Explorer completes a workflow described as removing the user from all roles. Nevertheless, the removed user can still open the name’s **Address Resolution** page, edit its Mainnet address, and submit `setAddr` through Explorer.

After the record is changed to an attacker-controlled address, a separate user can enter that ENS name in Explorer’s **Transfer ownership** form. Explorer resolves and visibly displays the attacker address, then constructs the ERC-1155 `safeTransferFrom` transaction with the attacker address as `to`.

The PermissionedResolver contract behaves according to the submitted calldata. The vulnerability is in Explorer’s account-level aggregation and under-scoped removal transaction.

### Impact Details

This reaches the selected in-scope impact:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient.

The demonstrated exploit sequence is:

{% stepper %}
{% step %}

## A delegate has root `ROLE_SET_ADDR` and a separate name-scoped role

{% endstep %}

{% step %}

## The resolver owner uses Explorer’s **Remove user from all roles** workflow

{% endstep %}

{% step %}

## Explorer revokes only the name-scoped resource

{% endstep %}

{% step %}

## The removed delegate retains global address-writing authority

{% endstep %}

{% step %}

## The delegate changes an ENS name to an attacker-controlled address through Explorer

{% endstep %}

{% step %}

## An independent user enters that ENS name as the recipient of an ENS name transfer

{% endstep %}

{% step %}

## Explorer displays the attacker address and constructs the transfer with it as `to`

{% endstep %}

{% step %}

## The user approves the normal Explorer transaction

{% endstep %}

{% step %}

## The attacker becomes owner of the transferred ENS NFT

{% endstep %}
{% endstepper %}

The PoC demonstrates an ERC-1155 ENS name moving from an independent victim to the attacker. This therefore also reaches the selected **Direct theft of user NFTs** impact.

A retained root role is global to the resolver, so the removed delegate can poison the address record of any name using that resolver. No compromised wallet, malicious extension, local-device access, privileged project account, or third-party service compromise is required.

The required conditions are only that the delegate previously received root `ROLE_SET_ADDR`, also has at least one name-scoped assignment, and the resolver owner later uses Explorer’s ordinary grouped removal workflow.

## References

* [`groupRolesByAccount()` merges resources and role bitmaps](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220)
* [Sidebar selects only the first non-root name](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L170)
* [`handleRemoveUser()` creates one scoped action](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323)
* [Global-role label](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L389-L397)
* [“Remove user from all roles” confirmation](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L463)
* [Single-resource revocation helper](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74)

## Proof of Concept

### Tested version

```
Asset: apps/portal
Repository: https://github.com/immunefi-team/audit-comp-ens
Competition snapshot: cda79acaad59711b943fc68207ebb3f1d0ff8596
Audited revision: 63772fd872af472ced58b009499355f3430c2a86
Network: isolated local Sepolia fork, chain ID 11155111
```

### Actors and initial state

Use four local development accounts:

```
OWNER     = resolver owner and intended recipient
DELEGATE  = resolver user that OWNER removes
ATTACKER  = attacker-controlled transfer recipient
VICTIM    = independent owner of a valuable ENS name
```

Create these names:

```
recipientName = an ENS name owned by and initially resolving to OWNER
valuableName  = a separate ENS name owned by VICTIM
```

`recipientName` uses a PermissionedResolver. Give `DELEGATE` these two assignments on that resolver:

```
root resource:          ROLE_SET_ADDR = 1
recipientName resource: ROLE_SET_TEXT = 16
```

Confirm the initial state:

```
findOwner(recipientName) = OWNER
addr(recipientName)      = OWNER
roles(root, DELEGATE)    = 1
roles(recipientName, DELEGATE) = 16
findOwner(valuableName)  = VICTIM
```

The root and name assignments must belong to the same account. This causes Explorer to group them into one user row.

#### Concrete passing test case

```
OWNER:
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

DELEGATE:
0x70997970C51812dc3A010C7d01b50e0d17dc79C8

ATTACKER:
0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC

VICTIM:
0x90F79bf6EB2c4f870365E785982E1f101E93b906

PermissionedResolver:
0x830361B26CCD75680B7E28B9cE9dadAAA6C17776

recipientName:
recipient-1787575909.eth

valuableName:
valuable-1787575920.eth
```

{% stepper %}
{% step %}

## Remove the grouped user through Explorer

1. Connect Explorer as `OWNER`.
2. Open the PermissionedResolver’s **Roles** page.
3. Locate the row for `DELEGATE`.
4. Confirm that the row represents both `(root)` and `recipientName`.
5. Open **Edit user roles**.
6. Confirm that Explorer describes the row as `Global roles (all names)`.
7. Click **Remove user**.
8. Explorer displays this confirmation:

   ```
   Are you sure you want to remove this user from all roles?
   ```
9. Click **Remove** and approve the wallet transaction generated by Explorer.
10. Wait for Explorer to report that the transaction completed.

Explorer selects the first non-root name from the grouped row:

```ts
const decodedRoles = row.original.decodedRoles
const resolvedNames = row.original.resolvedNames
const roleName = resolvedNames.find((name) => name !== '(root)') ?? ''
```

It then submits one call equivalent to:

```
authorizeNameRoles(
  recipientName,
  ROLE_SET_ADDR | ROLE_SET_TEXT,
  DELEGATE,
  false
)
```

Decode the mined transaction and confirm:

```
function: authorizeNameRoles
scope: recipientName
bitmap: 17
account: DELEGATE
grant: false
```

Read both resources after confirmation:

```
roles(root, DELEGATE) = 1
roles(recipientName, DELEGATE) = 0
```

The name-scoped role was removed, but root `ROLE_SET_ADDR` remains active even though Explorer completed a workflow presented as removing the user from all roles.

Observed local transaction:

```
0x65b40f30f2f0fe771189a7c1d4a2d3dea689bd44c8ec99b6ca99bd6d9fa09cfa
```

{% endstep %}

{% step %}

## Change the recipient name as the removed user

1. Switch the connected wallet to `DELEGATE`.
2. Open `recipientName` in Explorer.
3. Select **Address Resolution**.
4. Open **Mainnet → More**.
5. Observe that Explorer still renders the address input and **Save** button for the removed user.
6. Replace the Mainnet address with `ATTACKER`.
7. Click **Save** and approve the wallet transaction generated by Explorer.
8. Wait for Explorer to report successful completion.

Decode the transaction and confirm:

```
sender: DELEGATE
function: setAddr
new address: ATTACKER
```

Read the record:

```
addr(recipientName) = ATTACKER
```

This step is performed through Explorer’s native Address Resolution workflow. The retained root role is what keeps the editor available and authorizes the transaction.

Observed local transaction:

```
0xdc116b2422a888d1edffd9543492a0ab6fa8c63f144156e71467c3014c7554a2
```

{% endstep %}

{% step %}

## Make Explorer display and use the attacker recipient

1. Switch the connected wallet to `VICTIM`.
2. Open the **Transfer ownership** page for `valuableName`.
3. Enter `recipientName` in the **ENS name or address** field.
4. Wait for Explorer to resolve the name.
5. Confirm that Explorer visibly displays `ATTACKER` as the recipient.
6. Click **Transfer name**.
7. Approve the normal Explorer transaction flow.
8. Wait for Explorer to report successful completion.

Decode the ERC-1155 transfer transaction:

```
function:
safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes data)

from: VICTIM
to: ATTACKER
```

Read the final owner and open Explorer’s **Ownership** page:

```
findOwner(valuableName) = ATTACKER
```

Explorer displays the attacker as the new owner. The independent victim’s ENS NFT has been transferred to the unintended recipient that Explorer resolved and displayed.

Observed local transaction and final state:

```
transfer transaction:
0xca15c3589daca62fb6a2412444fe115732179723309edcf0bbfcad97b9febeac

Explorer-displayed recipient:
0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC

decoded safeTransferFrom.to:
0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC

findOwner(valuable-1787575920.eth):
0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
```

{% endstep %}
{% endstepper %}

## Causality control

Restore the exact initial chain state. Change only the revocation scope by correctly revoking root `ROLE_SET_ADDR` from `DELEGATE`:

```
roles(root, DELEGATE) = 0
```

Repeat Steps 2 and 3.

Expected control result:

{% stepper %}
{% step %}

## Explorer’s Mainnet-resolution sheet does not render an address input or **Save** button for `DELEGATE`

{% endstep %}

{% step %}

## `DELEGATE` cannot change `recipientName` through Explorer

{% endstep %}

{% step %}

## Explorer’s transfer form resolves `recipientName` to `OWNER`

{% endstep %}

{% step %}

## `ATTACKER` is not displayed as the recipient

{% endstep %}

{% step %}

## The transfer-to-attacker consequence is blocked

{% endstep %}
{% endstepper %}

The only changed condition is correct revocation of the root role. This establishes that the final incorrect-recipient transfer is caused by Explorer’s under-scoped grouped-role removal.

Observed control state:

```
root-revocation transaction:
0xa22d0c4e349e61da03a438eee4ab29b3ea902af4b3b9055e4f2f059e99313375

roles(root, DELEGATE):
0

address editor visible for DELEGATE:
false

Explorer-displayed recipient:
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```

### Expected result summary

```
Vulnerable removal:
  root role after removal       = 1
  name role after removal       = 0
  address editor for DELEGATE   = visible
  addr(recipientName)           = ATTACKER
  Explorer transfer preview     = ATTACKER
  safeTransferFrom.to           = ATTACKER
  findOwner(valuableName)       = ATTACKER

Correct root revocation:
  root role after revocation    = 0
  address editor for DELEGATE   = unavailable
  Explorer transfer preview     = OWNER
  transfer-to-attacker          = blocked
```
