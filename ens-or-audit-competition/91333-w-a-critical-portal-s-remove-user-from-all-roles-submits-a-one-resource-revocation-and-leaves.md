# 91333 w a critical portal s remove user from all roles submits a one resource revocation and leaves the removed delegate s root resolver authority active

**Submitted on Aug 31st 2026 at 17:44:30 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91333
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Summary

PermissionedResolver roles are stored per resource. Root resource `0` applies globally; a name resource applies to one ENS name. It is valid for the same account to hold different role bitmaps in both resources.

The Portal's resolver overview receives those assignments as separate rows, each with its own `resource` and `roleBitmap`. The Portal then groups the rows solely by account. During grouping it keeps the original rows but also creates one union of every decoded role and one list of every resolved scope.

The table shows one account row. In the reproduced state that row accurately shows:

```
scopes:       (root), poc-scope-1.eth
permissions:  Set Address, Set Text, Upgrade
```

The mutation path does not use the retained per-resource rows. The sidebar reduces the complete scope list to one scalar:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

Because a non-root name exists, `roleName` becomes `poc-scope-1.eth`. The remove action combines that one name with the union of all three permissions. The ENSjs builder therefore emits one name-scoped call:

```
authorizeNameRoles(
  dnsEncode("poc-scope-1.eth"),
  ROLE_SET_ADDR | ROLE_SET_TEXT | ROLE_UPGRADE,
  delegate,
  false
)
```

It never emits `revokeRootRoles(...)` and never iterates the account group's original resource rows. Removing extra role bits from the name resource cannot remove the same bits from the independent root resource. The transaction finishes successfully, so the owner receives a successful transaction for an action that did not perform the stated account-wide revocation.

The defect is deterministic for any account that has both a root assignment and at least one named assignment. It is independent of role-row ordering: `find(n !== '(root)')` necessarily selects a name whenever one exists.

## Finding Description

### Intended role and resource model

The Portal defines independent resolver permission bits, including:

```ts
ROLE_SET_ADDR = 1n << 0n
ROLE_SET_TEXT = 1n << 4n
ROLE_UPGRADE  = 1n << 124n
```

It also defines the independent resources:

```
root resource = 0x0000000000000000000000000000000000000000000000000000000000000000

name resource = keccak256(
  abi.encodePacked(namehash(name), bytes32(0))
)
```

For `poc-scope-1.eth`:

```
node:
0x2591e47f569d62d0ffb76384cc61bf1c4b21825d274b3fd69ee5c388ce1f6997

name resource:
0xb43b6ead2bb5e640a7ba93d783df400395a1dd5a4057632db3e42707553ce062
```

The scoped Portal itself documents that PermissionedResolver writes are authorized by either root or name-scoped roles:

```ts
// Permissioned resolvers: authorized by resolver `ROLE_SET_*` (root or
// name-scoped).
```

This is also exercised by the scoped smart-account registration path, which initializes a resolver with root roles and then performs ordinary standalone record setters because those root roles authorize the writes.

Therefore these assignments are not interchangeable:

```
roles(root, delegate)          = global authority
roles(nameResource, delegate)  = authority stored only for that name resource
```

An account-wide removal must address every resource on which that account has roles. A single name-scoped subtraction cannot clear root storage.

### Reachable correct-resource state

The PoC uses this permissionlessly indexed public Sepolia witness:

```
fork block:       11523593
block hash:       0xb353fd495c9607694797d1a195820332924a531081b3a4657c3a9135e375c419
resolver:         0x0e7cd4d93c25352d80b3aa2cdf027e1196bf1e36
owner:            0x5549c1f12cbaee908bf2d9a1ba79d9701af46523
delegate:         0x00000000000000000000000000000000deadbeef
name:             poc-scope-1.eth
```

At the frozen block, direct contract reads return:

```
roles(root, delegate):
0x10000000000000000000000000000001
= ROLE_SET_ADDR | ROLE_UPGRADE

roles(nameResource, delegate):
0x10
= ROLE_SET_TEXT

hasRoles(nameResource, ROLE_SET_ADDR, delegate): true
```

The two grants were normal owner-originated resolver transactions:

```
root grant:
0x6629d212aee10fbddc08ad279d76330356aedbba9b6c54163f8df0136a0d6c9d

name grant:
0x6dcd31ab0d16ad066ae6fbd634798d1e8524217cd8935380d329d544c1479d69
```

The official endpoint `https://staging-graphql.ens.dev/` returns exactly two delegate rows with the full 256-bit resource identifiers above, the exact bitmaps, the exact grant hashes, and a node entry mapping the name, owner, and resolver. The PoC rejects any truncated, missing, extra, or mismatched row.

This matters because it separates the scoped bug from the out-of-scope data service: the Portal receives correct data and constructs the wrong transaction from it.

The witness is used for reproducibility and is not characterized as an organic victim. The PoC sends no public transaction. It unlocks the ordinary owner and delegate addresses only on the disposable fork.

### 1. The overview preserves every resource row

`apps/portal/src/features/resolver/hooks/useResolverOverview.ts:14-22` defines each role assignment with separate fields:

```ts
export type ResolverRole = {
  readonly account: string
  readonly resource: string
  readonly roleBitmap: string
  readonly blockNumber: number
  readonly transactionHash: string | null
  readonly timestamp: number | null
  readonly name: string | null
}
```

The `getResolverOverview` query at lines 60-111 requests the complete `nodes` and `roles` collections, including `resource` and `roleBitmap`. The roles route passes both complete arrays to `ResolverRolesTable` at `apps/portal/src/routes/resolver/$address/roles/index.tsx:61-90`.

There is no missing-resource condition at this stage.

### 2. The table merges independent resources solely by account

`apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:102-105` does this:

```ts
const data = useMemo(
  () => groupRolesByAccount(roles, buildResourceToNameMap(nodes)),
  [roles, nodes],
)
```

`apps/portal/src/lib/roles/resolverRoles.ts:129-160` maps resource zero to `(root)` and computes each complete name resource. The mapping is correct.

The loss of scope begins in `groupRolesByAccount()` at lines 179-220:

```ts
for (const role of roles) {
  const account = role.account.toLowerCase()
  const resource = role.resource.toLowerCase()
  const decoded = decodeResolverRoleBitmap(role.roleBitmap)
  const resolvedName = resourceToName?.get(resource) ?? null

  const existing = grouped.get(account)

  if (existing) {
    existing.roles.push(role)
    existing.decodedRoles.push(...decoded)
    if (resolvedName) existing.resolvedNames.add(resolvedName)
  } else {
    // one entry keyed by account
  }
}
```

The original `roles` rows remain present, but `decodedRoles` becomes a cross-resource union. The table renders one row using that union. In the PoC, the real DOM row contains all of:

```
0x0000…beef
(root)
poc-scope-1.eth
Set Address
Set Text
Upgrade
```

### 3. The sidebar promises global/account-wide removal

`ResolverRolesSidebar.tsx:165-169` receives that grouped row:

```ts
const selectedAccount = row?.original.account as Address
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The sidebar uses the presence of `(root)` to tell the owner the scope is global, at lines 391-396:

```tsx
{resolvedNames.includes('(root)')
  ? 'Global roles (all names)'
  : `Roles scoped to ${...}`}
```

The owner presses the button labelled `Remove user`. The confirmation at lines 441-448 then says:

```
Are you sure you want to remove this user from all roles? This action cannot be undone.
```

These statements agree with the grouped row the owner is looking at: the UI represents one account with global and named permissions. Nothing tells the owner that only one underlying resource will be passed to the wallet.

### 4. The remove action discards the resource rows

`handleRemoveUser()` at lines 307-323 constructs the pending action:

```ts
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

This is the decisive mismatch:

```
row.original.roles          = both original resource assignments
pendingAction.name          = one selected non-root name
pendingAction.roles         = union of bits from both resources
```

No resource identifier is retained in the action. The mutation at lines 251-277 and the transaction-modal start path at lines 486-493 both pass the same one `name` and one union `roles` array. Neither path iterates `row.original.roles`.

The gas-estimate/preflight path is not safer. `prepareResolverRolesIntent.ts` lines 19-24 and 38-46 also represent a remove action with exactly one scalar name and call the same transaction builder.

### 5. One non-empty name forces one name-scoped wallet request

`apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61` selects exactly one branch:

```ts
const writeParams = revokeResolverRolesWriteParameters(
  walletClient,
  name === ''
    ? {
        resolverAddress,
        targetAccount: account,
        scope: 'root',
        roles,
      }
    : {
        resolverAddress,
        targetAccount: account,
        scope: 'name',
        name,
        roles,
      },
)
```

Because `poc-scope-1.eth` is non-empty, the root branch is unreachable for the remove action. Lines 63-74 encode one calldata blob and one EOA intent. The actual submission path at lines 99-117 starts that one transaction.

The PoC pauses the real `eth_sendTransaction` request from the unmodified Portal before it reaches Anvil. The exact decoded request is:

```
wallet chain: 11155111 (asserted separately)
from:         0x5549c1f12cbaee908bf2d9a1ba79d9701af46523
to:           0x0e7cd4d93c25352d80b3aa2cdf027e1196bf1e36
value:        0

function: authorizeNameRoles
name:     0x0b706f632d73636f70652d310365746800
bitmap:   0x10000000000000000000000000000011
account:  0x00000000000000000000000000000000deadbeef
grant:    false
```

Complete calldata:

```
0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000001000000000000000000000000000001100000000000000000000000000000000000000000000000000000000deadbeef000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000110b706f632d73636f70652d310365746800000000000000000000000000000000
```

The name argument is the only represented resource. The wallet is never given a root revoke.

### 6. The call succeeds while root authority remains

The union bitmap does not compensate for the wrong scope. It removes matching bits only from the computed `poc-scope-1.eth` resource:

```
before:
  root = ROLE_SET_ADDR | ROLE_UPGRADE
  name = ROLE_SET_TEXT

Portal request:
  name &= ~(ROLE_SET_ADDR | ROLE_SET_TEXT | ROLE_UPGRADE)

after:
  root = ROLE_SET_ADDR | ROLE_UPGRADE
  name = 0
```

The transaction succeeds because clearing absent Set Address/Upgrade bits from the name resource is harmless. A successful receipt therefore does not tell the owner that the root assignment was untouched.

The PoC reads both resources immediately after the Portal transaction and asserts the exact `[rootBitmap, 0]` state. It separately reads `hasRoles(nameResource, ROLE_SET_ADDR, delegate)` and receives `true`.

### 7. The removed account uses the surviving authority

Without restoring or granting any authority, the delegate submits:

```
setAddr(
  0x2591e47f569d62d0ffb76384cc61bf1c4b21825d274b3fd69ee5c388ce1f6997,
  60,
  0x1111111111111111111111111111111111111111
)
```

The deployed resolver bytecode on the fixed fork accepts the transaction. Direct `addr(node, 60)` and a fresh ENSjs `getRecords()` resolution both return the delegate-selected address.

The delegate is not exploiting a contract authorization error: it is using the root role that the Portal's incomplete revocation left active. That makes the source of the impact unambiguously the scoped Portal transaction composition.

### 8. Corrected control proves the missing resource is decisive

The PoC restores the original empty address and the original name role. The owner then performs the two resource-specific revocations separately:

```
revokeRootRoles(
  ROLE_SET_ADDR | ROLE_UPGRADE,
  delegate
)

authorizeNameRoles(
  dnsEncode("poc-scope-1.eth"),
  ROLE_SET_TEXT,
  delegate,
  false
)
```

Both transactions succeed. Final direct role storage is `[0, 0]`. The same delegate `setAddr` mutation to `0x2222222222222222222222222222222222222222` now reverts, and the original record remains unchanged.

This control uses the same owner, delegate, resolver, name, fork, and role state. The only material difference is addressing both resources rather than collapsing them into one name-scoped call.

## Root Cause

The Portal uses incompatible data models for display and mutation:

{% stepper %}
{% step %}

## The GraphQL model

The GraphQL model is one row per `(account, resource, roleBitmap)`.
{% endstep %}

{% step %}

## Role grouping

`groupRolesByAccount()` merges those rows into one account object and unions their role names.
{% endstep %}

{% step %}

## UI representation

The UI correctly exposes the merged object as one user with all scopes and permissions.
{% endstep %}

{% step %}

## Scope reduction

The mutation reduces the merged scope set to one scalar non-root name.
{% endstep %}

{% step %}

## Incorrect encoding

The union bitmap is encoded against only that one name resource.
{% endstep %}

{% step %}

## Unused resource rows

The retained `row.original.roles` resource rows are never used to prepare transactions.
{% endstep %}
{% endstepper %}

In compact form:

```
[(root, SET_ADDR|UPGRADE), (name, SET_TEXT)]
                   |
                   | group by account
                   v
{ scopes: [root,name], roles: SET_ADDR|UPGRADE|SET_TEXT }
                   |
                   | choose one non-root name
                   v
authorizeNameRoles(name, SET_ADDR|UPGRADE|SET_TEXT, false)
                   |
                   v
root untouched; name cleared
```

The account-wide UI action and the one-resource transaction cannot represent the same operation.

## Impact Explanation

This is the listed Critical wallet impact because the application supplies a connected wallet with materially different authorization parameters from the operation the user selected and confirmed:

| User-visible operation   | Actual wallet operation                        |
| ------------------------ | ---------------------------------------------- |
| Remove this user         | Revoke a bitmap on one name resource           |
| Global roles (all names) | No root-resource operation                     |
| Remove from all roles    | No iteration over the account's role resources |

The discrepancy changes authorization state, not presentation alone. The transaction target, chain, and sender are valid, so an ordinary wallet has no basis to infer the omitted resource. The user would need to manually decode the calldata and reconstruct every role resource to detect the error.

The post-transaction authority is security-sensitive:

* root `ROLE_SET_ADDR` can update address records across names using the resolver;
* the removed delegate successfully performs such an update in the PoC; and
* direct contract reads plus fresh ENSjs resolution confirm the changed externally consumed record.

## Likelihood Explanation

The trigger is a normal supported state: one delegate holds roles on root and one named resource of the same PermissionedResolver. The Portal's own Add user flow supports selecting root or named scope independently, and role grants are stored per resource.

The owner follows the safest normal action available in the interface:

{% stepper %}
{% step %}

## Connect the wallet

Connect the wallet that can manage resolver roles.
{% endstep %}

{% step %}

## Open the account row

Open the account row showing all scopes and permissions.
{% endstep %}

{% step %}

## Remove the user

Select **Remove user**.
{% endstep %}

{% step %}

## Confirm all-role removal

Read and accept the explicit all-role confirmation.
{% endstep %}

{% step %}

## Approve the transaction

Approve the one transaction requested by the Portal.
{% endstep %}
{% endstepper %}

The owner does not choose a malicious recipient, unsafe call order, custom calldata, unusual parameter, or nonstandard composition. There is no safe alternative ordering presented inside that all-role action. The incomplete scope is generated by the Portal after the user's selection.

The delegate needs only the already-granted role that the owner is trying to revoke. It requires no administrator key, governance action, compromised oracle, browser extension, contract bug, indexer error, malformed response, or unusual chain configuration.

## Attack Path / Reproduction Path

{% stepper %}
{% step %}

## Grant root and name permissions

A resolver owner grants account `D` at least one root permission and at least one permission scoped to name `N` on the same PermissionedResolver.
{% endstep %}

{% step %}

## Delegate becomes untrusted

`D` later becomes untrusted or should no longer have any resolver access.
{% endstep %}

{% step %}

## Open the Roles page

The owner connects to Explorer and opens the resolver's Roles page.
{% endstep %}

{% step %}

## Receive correct resource rows

Explorer receives correct separate root and name role rows.
{% endstep %}

{% step %}

## Display combined account state

Explorer groups those rows into one account row for `D` and displays their combined scopes and permissions.
{% endstep %}

{% step %}

## Open the grouped account row

The owner opens that row. Explorer describes it as `Global roles (all names)`.
{% endstep %}

{% step %}

## Confirm removal

The owner clicks **Remove user** and confirms removal `from all roles`.
{% endstep %}

{% step %}

## Build the incorrect transaction

Explorer selects one non-root name, unions every role bit in the account group, and constructs one `authorizeNameRoles(name, union, D, false)` call.
{% endstep %}

{% step %}

## Submit the transaction

The connected wallet approves and submits that application-generated request.
{% endstep %}

{% step %}

## Name resource is cleared

The name resource is cleared, but the root resource remains unchanged.
{% endstep %}

{% step %}

## Root authority remains active

`D` continues to satisfy the resolver's root-role fallback for name writes.
{% endstep %}

{% step %}

## Modify an address record

`D` changes an address record through a normal `setAddr` transaction.
{% endstep %}
{% endstepper %}

## External Preconditions

* The victim controls an ordinary wallet that can manage roles on a PermissionedResolver.
* One delegate has assignments on both the resolver root and at least one name resource.
* The victim uses the Portal's ordinary account-row **Remove user** action and approves the transaction it prepares.

## Internal Preconditions

* The resolver overview returns the delegate's correct separate resource rows.
* `groupRolesByAccount()` continues grouping by account and concatenating decoded roles across resources.
* The sidebar continues selecting one non-root `roleName`.
* The removal builder continues selecting exactly one of root or name scope from that scalar.
* Root roles continue authorizing the corresponding name write, as the scoped code and deployed resolver state demonstrate.

Every internal condition is asserted by the PoC against the frozen competition revision and fixed deployed Sepolia state.

## Affected Code

Frozen revision: [`1c9b47f18fcddd2e864dfe385c4171061c9811ae`](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae)

* [`useResolverOverview.ts#L14-L22`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L14-L22) models the correct per-resource role rows.
* [`useResolverOverview.ts#L60-L111`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L60-L111) requests complete role and node data.
* [`resolverRoles.ts#L129-L160`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L129-L160) correctly maps root and complete name resources.
* [`resolverRoles.ts#L168-L220`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L168-L220) groups all resource rows by account and concatenates their role names.
* [`ResolverRolesTable.tsx#L102-L105`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L102-L105) makes that account group the editable row.
* [`ResolverRolesSidebar.tsx#L165-L192`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L192) reduces all scopes to one non-root name while retaining the role union.
* [`ResolverRolesSidebar.tsx#L307-L323`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323) builds the remove action from that one name and the cross-resource roles.
* [`ResolverRolesSidebar.tsx#L391-L396`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L391-L396) labels any root-containing group `Global roles (all names)`.
* [`ResolverRolesSidebar.tsx#L441-L493`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L493) promises all-role removal and submits the one-name pending action.
* [`prepareResolverRolesIntent.ts#L19-L46`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts#L19-L46) uses the same one-name model for the prepared wallet intent.
* [`revokeResolverRoles.ts#L29-L74`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74) selects either one root scope or one name scope and encodes one request.
* [`revokeResolverRoles.ts#L84-L117`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L84-L117) submits that same single request.
* [`useCanEditRecords.ts#L35-L41`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useCanEditRecords.ts#L35-L41) states that root or name-scoped roles authorize resolver record writes.
* [`useCanEditRecords.ts#L68-L84`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/hooks/useCanEditRecords.ts#L68-L84) checks that effective authorization on the name resource.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/9eba662fe7b2456cc29a29e755839b14>

## Proof of Concept

I provide the complete runnable source and full reproduction guide in this Gist:

[**ENS Portal account-grouped resolver-revocation E2E PoC**](https://gist.github.com/aydarkhusaenov/9eba662fe7b2456cc29a29e755839b14)

Place the package at:

```
poc/portal-account-grouped-resolver-revocation/
```

### What the PoC uses

* the exact unmodified competition commit;
* the Portal's real React UI and transaction modal;
* the Portal's production GraphQL client and official configured endpoint;
* the official response containing correct full role resources;
* real Chromium driven by Playwright;
* the production ENSjs calldata builder used by the Portal;
* the actual Portal-generated `eth_sendTransaction` request;
* a digest-pinned Anvil fork at an exact Sepolia block/hash;
* the deployed Universal Resolver and PermissionedResolver bytecode/state;
* real mined revocation and delegate-mutation transactions on the fork; and
* direct contract reads plus fresh ENSjs resolution.

It does not mock contracts, role rows, resolver behavior, GraphQL responses, the transaction builder, or the impact. No state-changing request is sent to a public network.

Fork-only account unlocking is the actor harness: the public witness owner and delegate are ordinary EOAs whose signing keys are not required or claimed. Their addresses are unlocked only inside the disposable local node. The Portal's wallet request is captured first and forwarded unchanged only after explicit test authorization.

### Package layout

```
poc/portal-account-grouped-resolver-revocation/
├── .gitignore
├── README.md
├── SOURCE_SHA256SUMS
├── TARGET_SHA256SUMS
├── constants.mts
├── docker-compose.poc.yml
├── package.json
├── playwright.config.ts
├── portal-account-grouped-revocation.spec.ts
├── run.sh
├── setup.mts
└── verify-final.mts
```

`SOURCE_SHA256SUMS` binds every PoC source file. `TARGET_SHA256SUMS` binds each decisive production file, including the official GraphQL URL resolver.

### Complete source and clean-clone reproduction

From an empty directory, clone the frozen target and the complete PoC directly into the path expected by `run.sh`:

```bash
mkdir ens30-reproduction
cd ens30-reproduction
git clone https://github.com/immunefi-team/audit-comp-ens.git audit-comp-ens
cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
git clone https://gist.github.com/aydarkhusaenov/9eba662fe7b2456cc29a29e755839b14.git \
  poc/portal-account-grouped-resolver-revocation

corepack enable
pnpm install --frozen-lockfile
pnpm --dir e2e exec playwright install chromium

cd poc/portal-account-grouped-resolver-revocation
shasum -a 256 -c SOURCE_SHA256SUMS
cd ../..
shasum -a 256 -c \
  poc/portal-account-grouped-resolver-revocation/TARGET_SHA256SUMS
```

Every manifest entry must print `OK` before execution. The canonical `SOURCE_SHA256SUMS` file has SHA-256 `636a706b2803ebfb2059454b9c5ae400fa448480cbd5911e51e863923b422bdb`.

### Requirements

* Bash;
* Git;
* Docker with Docker Compose v2;
* Node.js 22;
* repository dependencies installed from the frozen lockfile;
* Playwright Chromium; and
* read access to an archive-capable Sepolia RPC and `https://staging-graphql.ens.dev/`.

If dependencies are not already installed, from the repository root:

```bash
corepack enable
pnpm install --frozen-lockfile
pnpm --dir e2e exec playwright install chromium
```

### Run command

From the root of the submitted repository:

{% tabs %}
{% tab title="Standard" %}

```bash
chmod +x poc/portal-account-grouped-resolver-revocation/run.sh
POC_LOG_DIR=/tmp/ens30-evidence \
  poc/portal-account-grouped-resolver-revocation/run.sh
```

{% endtab %}

{% tab title="mise" %}

```bash
PATH="$(mise where node@22)/bin:$PATH" \
POC_LOG_DIR=/tmp/ens30-evidence \
  poc/portal-account-grouped-resolver-revocation/run.sh
```

{% endtab %}

{% tab title="Alternative Sepolia endpoint" %}

```bash
SEPOLIA_FORK_URL='https://your-sepolia-archive-rpc.example' \
POC_LOG_DIR=/tmp/ens30-evidence \
  poc/portal-account-grouped-resolver-revocation/run.sh
```

{% endtab %}
{% endtabs %}

### Automated preflight

Before starting the browser, `setup.mts` fails unless it verifies all of:

{% stepper %}
{% step %}

## Repository and source verification

Verify the exact repository commit and exact PoC and target source hashes.
{% endstep %}

{% step %}

## Fork verification

Verify the exact archive block number and block hash, loopback-only fixed Anvil fork, resolver and Universal Resolver bytecode, and ordinary EOA actor code state.
{% endstep %}

{% step %}

## Resource verification

Verify the computed node and complete name resource, Universal Resolver result, and exact direct root, name, and owner role state.
{% endstep %}

{% step %}

## Authorization verification

Verify effective root `ROLE_SET_ADDR` authorization.
{% endstep %}

{% step %}

## GraphQL verification

Verify the exact official GraphQL resolver, node, owner, and full-resource rows.
{% endstep %}

{% step %}

## Transaction verification

Verify exact root/name grant transaction hashes and bitmaps, and owner-originated public grant transactions targeting the resolver.
{% endstep %}
{% endstepper %}

The preflight has bounded retries for a transient official-endpoint outage but never substitutes a local or captured response.

### Automated browser and impact sequence

The Playwright test performs and asserts this complete sequence:

{% stepper %}
{% step %}

## Start the Portal

Start the frozen unmodified Portal against the fixed fork and official GraphQL endpoint.
{% endstep %}

{% step %}

## Connect the fixture owner

Connect as the fork-only unlocked fixture owner through the browser wallet.
{% endstep %}

{% step %}

## Open the Roles page

Open the real resolver Roles page.
{% endstep %}

{% step %}

## Assert the grouped row

Assert exactly one delegate row exists, and that the row displays `(root)`, `poc-scope-1.eth`, Set Address, Set Text, and Upgrade.
{% endstep %}

{% step %}

## Save row evidence

Save a screenshot of that correctly sourced grouped row.
{% endstep %}

{% step %}

## Open role editing

Open **Edit user roles** and assert the sidebar says `Global roles (all names)`.
{% endstep %}

{% step %}

## Confirm removal wording

Select **Remove user**, assert the confirmation says `remove this user from all roles`, and save a screenshot.
{% endstep %}

{% step %}

## Start the wallet flow

Start the real transaction flow and press **Open wallet**.
{% endstep %}

{% step %}

## Capture the wallet request

Pause the actual wallet request without modifying it.
{% endstep %}

{% step %}

## Verify transaction cardinality

Assert the request's owner, resolver target, value, and single-transaction cardinality.
{% endstep %}

{% step %}

## Decode the request

Decode the request as the exact one-name `authorizeNameRoles` call above.
{% endstep %}

{% step %}

## Submit unchanged request

Authorize and forward the unchanged request to local Anvil.
{% endstep %}

{% step %}

## Verify Portal completion

Assert the Portal transaction modal reaches Done and exposes the same mined hash.
{% endstep %}

{% step %}

## Verify incomplete revocation

Assert post-state `[rootBitmap, 0]` and that the delegate still has effective name `ROLE_SET_ADDR`.
{% endstep %}

{% step %}

## Mutate as the delegate

Send the delegate's normal `setAddr` call and require a successful receipt.
{% endstep %}

{% step %}

## Verify changed address

Assert direct resolver state and fresh ENSjs resolution both return `0x1111111111111111111111111111111111111111`.
{% endstep %}

{% step %}

## Restore the original state

Restore the original address and name role.
{% endstep %}

{% step %}

## Revoke both resources

Revoke root and name roles separately as the owner.
{% endstep %}

{% step %}

## Verify final role state

Assert final role state `[0, 0]`.
{% endstep %}

{% step %}

## Verify mutation is blocked

Repeat the delegate mutation and require it to revert.
{% endstep %}

{% step %}

## Verify original record remains

Assert the final record remains the original value.
{% endstep %}
{% endstepper %}

### Independent final verification

`verify-final.mts` does not trust the browser assertions alone. It reloads the five decisive transactions and their receipts from the local node and checks:

* all receipts succeeded where success is expected;
* Portal removal was mined before the delegate continuation;
* owner/delegate sender identities and resolver targets;
* exact Portal name, bitmap, account, and `grant=false` calldata;
* exact delegate node, coin type, and replacement address calldata;
* exact separate root and name control calldata;
* official correct-resource GraphQL rows observed by the browser;
* final authoritative roles `[0, 0]`; and
* final restored record state.

### Captured successful run

The final fresh run on 28 August 2026 produced:

```
official GraphQL status: 200
official delegate rows:  2 (full root and name resources)

Portal revoke:
  hash:   0xe007884b7089ac07ed60e981709aed1389b8930af66ab7f3a8f1d802d69cc4c5
  block:  11523594
  status: success
  roles before: [21267647932558653966460912964485513217, 16]
  roles after:  [21267647932558653966460912964485513217, 0]
  effective name SET_ADDR after removal: true

surviving delegate mutation:
  hash:   0x5a9744ae25bfe1551f4389887bf097d9c202134ec2ee27d3563d6efd4f1e5394
  block:  11523595
  status: success
  direct address:   0x1111111111111111111111111111111111111111
  resolved address: 0x1111111111111111111111111111111111111111

corrected resource-specific control:
  root revoke: 0x5e396f4535399c3dc2c85103e854a0ee44fbaf61139d0117ee27029adb542033
  name revoke: 0xea0d3e388a9b644b299147008b0dde876f59969a19bf73c9dc0831550d46f91e
  blocked mutation:
    0xc00c9d8d85287201f07eca296f2839e9077bbc3558a772d9e5e018583d3ad954
  blocked status: reverted
  final roles:    [0, 0]
  final address:  0x
```

The test succeeds only with all four markers:

```
ENS30_CORRECT_RESOURCE_SETUP_PASS
ENS30_CORRECT_RESOURCE_BROWSER_E2E_PASS
ENS30_FINAL_VERIFICATION_PASS
ENS30_ACCOUNT_GROUPED_RESOLVER_REVOCATION_E2E_PASS
```

### Evidence files

The selected `POC_LOG_DIR` contains:

```
setup.json
browser.json
final.json
setup.log
portal.log
playwright.log
verify-final.log
compose-up-anvil.log
01-correct-resource-grouped-row.png
02-all-roles-confirmation.png
03-single-name-revoke-success.png
playwright-results/
```

The JSON receipts include the complete official GraphQL exchange, exact wallet request, decoded parameters, mined hashes and block numbers, before/after role state, delegate continuation, fresh resolved address, corrected control, and harness boundaries.

### Cleanup

The runner removes only its explicitly named disposable Docker Compose project and stops the local Portal on success, failure, interruption, or shell exit. It does not reset tracked source, remove unrelated files, prune shared Docker state, or send a state-changing request to a public network.
