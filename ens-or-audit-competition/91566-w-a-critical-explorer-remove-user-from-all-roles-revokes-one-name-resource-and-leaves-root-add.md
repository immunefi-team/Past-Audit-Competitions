# 91566 w a critical explorer remove user from all roles revokes one name resource and leaves root address authority live

**Submitted on Sep 2nd 2026 at 17:01:06 UTC by @go4ko for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91566
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Impact summary and selected category

**Selected in-scope impact — Critical:**

> **Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters; substituting contract addresses; submitting malicious transactions; signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated); inducing a blanket token approval (for example `setApprovalForAll` on the registry or NameWrapper) to an attacker-controlled address.**

Explorer asks the connected owner to approve a name-scoped revocation while explicitly representing the action as removing the delegate “from all roles.” The successful application-generated transaction therefore contains the wrong authorization resource: it clears the selected name resource but leaves the delegate's root `ROLE_SET_ADDR` authority live.

The E2E demonstrates the security consequence rather than treating incomplete removal as a display-only defect. After the successful removal and a fresh Explorer reload, the former delegate uses the surviving root role to change the name's canonical ETH address from the owner to an attacker address. A distinct payer resolves the same name and transfers exactly `0.01 ETH` to the attacker. That amount is a safe local-fork proof value, not a protocol-defined amount or loss ceiling; in practice, the redirected loss equals whatever value a payer sends through the compromised ENS resolution before the owner repairs the record.

The qualifying connected-wallet impact occurs at the owner's revocation transaction: Explorer asks the wallet to approve one name-scoped resource while representing the action as complete removal across the aggregate root-plus-name role set. The later resolver lookup and payment are downstream proof that the retained authority is security-relevant; this report does not allege that Universal Resolver returned a noncanonical record or treat ordinary resolution by itself as the vulnerability.

The authority is recoverable: after refresh, a vigilant owner can see the remaining `(root)` / `Set Address` row and submit a second, root-scoped revocation. This report therefore does not claim permanent name takeover, invisible authority after refresh, or compromise of the owner's wallet. Recovery stops future misuse but cannot reverse a payment already diverted during the failed-revocation window.

## Brief/Intro

ENS Explorer represents Permissioned Resolver roles as independent root and name resources in its indexer response, but `groupRolesByAccount` collapses every resource held by the same account into one table row. The resulting row contains a union of names and decoded permissions.

The edit sidebar then destroys the resource distinction in two ways:

1. It flattens every decoded permission into one list.
2. It selects the first non-root name as the single transaction scope whenever the aggregate contains both `(root)` and any name.

When the owner clicks `Remove user`, Explorer passes that one name together with the union of roles to `revokeResolverRoles`. A nonempty name selects ENSJS's name-scoped revocation path. Consequently, a row containing root `ROLE_SET_ADDR` and name `ROLE_SET_TEXT` produces:

```
authorizeNameRoles(name, ROLE_SET_ADDR | ROLE_SET_TEXT, delegate, false)
```

instead of a root revocation plus a name revocation. The name bitmap becomes zero, but the root bitmap is untouched. Because root roles apply across the resolver, effective `ROLE_SET_ADDR` for the name remains true.

The application confirmation is unambiguous: “remove this user from all roles.” The owner follows that workflow correctly, approves one application-generated transaction, and receives a successful transaction result. The defect is not a user choosing the wrong control or the resolver contract failing to execute valid calldata. Explorer constructs incomplete calldata for the action it promised.

### Security invariant and actors

* **O** — legitimate ENS name/resolver owner and connected-wallet signer.
* **D** — legitimate delegate before removal, but former, compromised, or adversarial delegate afterward.
* **R** — attacker-controlled recipient address selected by D.
* **P** — independent payer who resolves the ENS name after the failed revocation.

The required revocation invariant is:

```
If Explorer presents an aggregate account row covering resources S and confirms
“remove this user from all roles,” success must remove every displayed role from
every represented resource in S.

After success, no displayed address-setting role may remain effective for D.
```

For the proven root-plus-name case:

```
before:
  roles(ROOT_RESOURCE, D) = ROLE_SET_ADDR = 1
  roles(nameResource, D)  = ROLE_SET_TEXT = 16
  hasRoles(nameResource, ROLE_SET_ADDR, D) = true

promised action:
  remove D from all roles

required after-state:
  roles(ROOT_RESOURCE, D) = 0
  roles(nameResource, D)  = 0
  hasRoles(nameResource, ROLE_SET_ADDR, D) = false

actual after-state:
  roles(ROOT_RESOURCE, D) = 1
  roles(nameResource, D)  = 0
  hasRoles(nameResource, ROLE_SET_ADDR, D) = true
```

## Vulnerability Details

### 1. The resolver overview preserves separate role resources

The in-scope [`getResolverOverview` query](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L60-L119) asks the indexer for each role's `account`, `resource`, and `roleBitmap`:

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

At this boundary, the root and name roles are still independent entries. The E2E's controlled local-fork response supplies one root entry with bitmap `1` and one name-resource entry with bitmap `16`. Every supplied bitmap is regenerated from a live on-chain `roles(resource, account)` read; the test does not invent authorization state.

### 2. `groupRolesByAccount` collapses independent resources

[`buildResourceToNameMap`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L129-L160) maps the zero resource to `(root)` and name resources to human-readable names. However, [`groupRolesByAccount`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L162-L220) keys the group only by lowercase account:

```ts
const account = role.account.toLowerCase()
const existing = grouped.get(account)

if (existing) {
  existing.roles.push(role)
  existing.decodedRoles.push(...decoded)
  if (resolvedName) existing.resolvedNames.add(resolvedName)
}
```

The resource is read and resolved for display but is not part of the grouping key. The group therefore contains multiple raw role objects but only one flattened `decodedRoles` array and one `resolvedNames` set.

The role values used in the E2E are deliberately nonredundant. [`resolverRoles`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L1-L45) defines:

```
ROLE_SET_ADDR = 1 << 0 = 1
ROLE_SET_TEXT = 1 << 4 = 16
```

Thus the aggregate union `17` cannot be mistaken for the same permission duplicated across two resources.

### 3. The table exposes one editor for the aggregate

[`ResolverRolesTable`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L65-L87) renders all `resolvedNames` and permissions from `decodedRoles`. At [lines 102-116](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L102-L116), it feeds the grouped result directly into the table and opens one sidebar for that row:

```ts
const data = useMemo(
  () => groupRolesByAccount(roles, buildResourceToNameMap(nodes)),
  [roles, nodes],
)

buildEditActionColumn<AccountRoleGroup>((row) => {
  setEditingRow(row)
  setOpen(true)
})
```

The preserved browser run requires exactly one delegate row containing `(root)`, the test name, `Set Address`, and `Set Text`.

### 4. The sidebar combines the permissions but chooses one non-root scope

[`ResolverRolesSidebar`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L173) reads the aggregate values, then selects the first non-root name:

```ts
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

If the row contains root plus any named resource, `roleName` cannot represent root. The root resource is discarded even though its permission remains in `decodedRoles`.

The presentation strengthens the owner's reasonable interpretation. When any root entry is present, [the sidebar says `Global roles (all names)`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L389-L396).

### 5. “Remove user” binds aggregate roles to that one name

[`handleRemoveUser`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323) constructs one pending action:

```ts
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

[`prepareResolverRolesIntent`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts#L32-L46) forwards the same single name and union of roles to transaction preparation. The execution mutation does the same at [`ResolverRolesSidebar.tsx` lines 251-276](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L251-L276).

There is no loop over `row.original.roles`, no root-specific action, and no postcondition requiring every represented resource to be cleared.

### 6. A nonempty name deterministically produces the wrong on-chain function

[`prepareRevokeResolverRolesTransaction`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L28-L74) selects scope solely from whether the chosen string is empty:

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

Because `roleName` is the first non-root name, the root branch is unreachable for the mixed row. The preserved mined transaction decodes exactly as:

```
function: authorizeNameRoles
name:     DNS-encoded test name
bitmap:   17 (ROLE_SET_ADDR | ROLE_SET_TEXT)
account:  D
grant:    false
```

No `revokeRootRoles` call is submitted.

### 7. The confirmation and transaction metadata do not disclose partial scope

The confirmation at [`ResolverRolesSidebar.tsx` lines 441-448](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L448) states:

```
Are you sure you want to remove this user from all roles?
This action cannot be undone.
```

The transaction metadata at [lines 339-349](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L339-L349) says only `Remove resolver user` / `Remove user {account}`. It does not identify the selected name resource or warn that root authority will remain.

The owner's wallet signature is genuine, but it authorizes calldata that is inconsistent with the security operation Explorer described.

### 8. Transaction success is not revocation success

The removal mutation's success handler merely invokes [`pollForIndexerSync`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/query/pollForIndexerSync.ts#L36-L73). That helper waits and invalidates the query a fixed number of times; it does not check that every role represented by the removed row is absent.

Accordingly, the valid name-scoped transaction is mined successfully and the flow completes even though the security postcondition is false. The E2E performs a full page reload, requires a second live role-data request, and observes D still present with `(root)` and `Set Address`. The removed name and `Set Text` are absent, proving that the application performed a partial revoke rather than showing wholly stale data. No message warns that removal was partial.

### 9. The contract correctly enforces the authority Explorer failed to revoke

After the application-generated transaction:

```
raw root bitmap = 1
raw name bitmap = 0
effective ROLE_SET_ADDR for D on the name = true
```

D then submits a real `setAddr` transaction to the Permissioned Resolver. It succeeds, and the Universal Resolver returns R instead of O. This is expected contract behavior: the root role remains valid. The application defect is choosing the wrong revocation scope, not a broken contract access check.

The negative control submits the correct root revocation. The root bitmap becomes zero, effective authority becomes false, and D's next `setAddr` attempt reverts. This proves that a valid recovery exists and isolates the failure to Explorer's transaction construction.

### State transition demonstrated by the E2E

| Stage                                 | Root bitmap for D | Name bitmap for D | Effective `Set Address` | Canonical recipient | Security result                                                      |
| ------------------------------------- | ----------------: | ----------------: | ----------------------- | ------------------- | -------------------------------------------------------------------- |
| Legitimate baseline                   |                 0 |                 0 | No                      | O                   | The name resolves to its intended owner.                             |
| Independent grants                    | 1 (`Set Address`) |   16 (`Set Text`) | Yes                     | O                   | D is an authorized delegate.                                         |
| Explorer aggregate                    |                 1 |                16 | Yes                     | O                   | One row shows `(root)`, name, `Set Address`, and `Set Text`.         |
| Owner confirms removal from all roles |                 1 |                16 | Yes                     | O                   | Explorer prepares one name-scoped revoke with union bitmap 17.       |
| Application transaction succeeds      |             **1** |             **0** | **Yes**                 | O                   | Name scope clears; dangerous root authority survives.                |
| Fresh Explorer reload                 |                 1 |                 0 | Yes                     | O                   | D reappears as `(root)` / `Set Address`; no partial-removal warning. |
| Former delegate acts                  |                 1 |                 0 | Yes                     | **R**               | D changes the canonical ETH address to the attacker.                 |
| Independent payer resolves and pays   |                 1 |                 0 | Yes                     | R                   | P transfers 0.01 ETH; R gains exactly 0.01 ETH.                      |
| Correct root revoke and restore       |             **0** |                 0 | **No**                  | O                   | D's next address write reverts; future resolution is safe.           |

### Attack sequence

{% stepper %}
{% step %}

## Establish the legitimate owner and recipient

O owns a name using a Permissioned Resolver and initially sets the name's ETH address to O.

Canonical Universal Resolver output equals O, establishing the legitimate payment recipient before any delegate grants.
{% endstep %}

{% step %}

## Grant independent root and name roles

O legitimately grants D root `ROLE_SET_ADDR`, allowing D to manage address records across the resolver.

O separately grants D name-scoped `ROLE_SET_TEXT` for the tested name.
{% endstep %}

{% step %}

## Open the Explorer role-management flow

At a later time, D is no longer trusted, or D's key is suspected to be compromised.

O opens the production Explorer resolver roles page while connected with the authorized owner wallet.

Explorer groups D's two independent resource entries into one row showing `(root)`, the name, `Set Address`, and `Set Text`.

The sidebar labels the aggregate `Global roles (all names)`.
{% endstep %}

{% step %}

## Remove the user through Explorer

O clicks `Remove user`; the confirmation promises to remove D “from all roles.”

O confirms and approves exactly one application-generated resolver transaction.

Explorer submits `authorizeNameRoles(name, 17, D, false)`. The receipt succeeds, but no root revoke occurs.
{% endstep %}

{% step %}

## Observe the surviving root authority

The name bitmap becomes zero while the root bitmap remains `1`; effective `ROLE_SET_ADDR` for the name remains true.

A fresh page reload and second role-data request show D still holding `(root)` / `Set Address`, without a partial-removal warning.
{% endstep %}

{% step %}

## Redirect resolution and divert a payment

Before O notices and completes a second root-scoped removal, D calls `setAddr` and changes the ETH address from O to R.

Universal Resolver returns R for the same name.

Independent payer P resolves the name and transfers `0.01 ETH` to R; R's balance increases by exactly that amount.
{% endstep %}

{% step %}

## Recover with correct root revocation

O can recover by submitting the correct root revocation and restoring the legitimate record.

After the correct revocation, D's next `setAddr` attempt reverts.
{% endstep %}
{% endstepper %}

### Preconditions and practical likelihood

#### Required state

1. The same delegate holds at least one root role and at least one role on a named resource of the Permissioned Resolver.
2. The root role that survives has a security consequence; the proven case uses `ROLE_SET_ADDR`.
3. The resolver owner uses Explorer's aggregate `Remove user` action after the delegate should no longer be trusted.

This mixed-resource state is legitimate and supported by the underlying permission model. The PoC uses different roles at each scope to prove that the resources are independently meaningful, not duplicate indexer rows.

#### Exploitation window

The retained root role persists until it is separately revoked; it does not expire when the first removal reports success. Exploitation nevertheless depends on D acting before O notices the refreshed root-only row and submits a correct second revocation. Financial loss further depends on a payer resolving and paying the name while the malicious address is active.

No race is required against the first removal transaction: the PoC waits for the successful receipt and refreshes Explorer before D acts. A vigilant owner can recover, so likelihood and duration are lower than a hidden or permanent authorization. However, successful recovery only prevents future misuse; it cannot reverse a transfer already sent to R.

D does not need to observe Explorer, detect the inconsistent row, or front-run the first removal. The exploitable interval begins when Explorer's name-scoped revocation is mined and ends only when O notices the surviving root role and a separate root-scoped revocation is mined. D can discover that authority remains through public on-chain events or state, periodic role checks, or simply by continuing to use the permission. Private submission of O's first transaction does not close the path because its resulting role state is public after inclusion. The PoC intentionally makes the surviving role visible through a full refresh before D acts, then proves D can still change the record during that interval.

Without the defect, the advertised complete removal would revoke every displayed root and name resource when the first workflow finishes, leaving D unable to write afterward. With the defect, that first successful workflow leaves the root role active until O discovers the incomplete revocation and mines a separate corrective transaction.

#### Trust model

D is legitimately authorized before the removal. This is not treated as a vulnerability. The security boundary is the later owner-authorized revocation: after O uses the product's explicit all-role removal control, D should no longer retain the displayed root authority.

The attack model is a former delegate, an offboarded service, or a compromised delegate key. Revocation controls exist precisely for the point at which a previously trusted credential must stop being trusted. No ENS administrator, protocol operator, resolver owner other than O, wallet provider, or indexer operator must behave maliciously.

### Why existing controls do not stop the issue

| Existing control                     | What it establishes                                    | Why it is insufficient                                                                          |
| ------------------------------------ | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| Separate indexed role entries        | Root and name state can be represented accurately      | Explorer collapses them by account before constructing the action.                              |
| Aggregate row displays both scopes   | The owner can see `(root)` and the name before removal | One editor cannot preserve which permission belongs to which resource.                          |
| `Global roles (all names)` label     | Root authority is acknowledged in the sidebar          | `roleName` still selects the first non-root name for the transaction.                           |
| “Remove from all roles” confirmation | O explicitly authorizes complete removal               | The pending action contains only one resource and aggregate roles.                              |
| Wallet approval                      | O authorizes the application-generated calldata        | Metadata does not disclose the name-only scope or retained root role.                           |
| Successful receipt                   | The name-scoped revoke executed correctly              | Receipt success says nothing about root state or the promised aggregate postcondition.          |
| Indexer polling                      | Explorer requests updated role data                    | It never asserts that all resources represented by the removed row are empty.                   |
| Refreshed root-only row              | A vigilant owner can discover the incomplete removal   | Discovery requires attention and another transaction; authority remains usable in the meantime. |
| Correct contract root revoke         | Recovery is technically possible                       | Explorer did not submit that call during the promised all-role action.                          |

### Why this is not user error

O selects the only row representing D, clicks the explicit `Remove user` control, confirms removal “from all roles,” approves the application transaction, and receives a successful result. O does not select an incorrect name, manually edit calldata, ignore a failed receipt, or ask Explorer to retain root permissions.

The row's continued presence after refresh makes recovery possible, but it does not retroactively make the first transaction correct. A security-sensitive removal function must either perform its advertised complete operation, accurately disclose that it is removing only one resource, or fail and require explicit resource-by-resource decisions.

### Known-issue distinction

This finding was not matched to the root cause of any disclosed competition issue reviewed for this report:

| Disclosed or related issue                                                                                                                        | Distinction                                                                                                                                                                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| QA-01 — some transfer role combinations or names with subnames later break profile/primary-name operations                                        | QA-01 concerns post-transfer functional failures. ENS-005 performs no ownership transfer and demonstrates a newly unintended authority consequence from Explorer's removal transaction. The competition disclosure expressly leaves a demonstrated unintended-authority consequence open as a possible new finding. |
| [ensdomains/contracts-v2 issue #418](https://github.com/ensdomains/contracts-v2/issues/418) — delegated roles survive ERC-1155 ownership transfer | Issue #418 is a contract lifecycle issue: pre-existing roles survive a V2 ownership transfer. ENS-005 performs no transfer. The resolver can revoke correctly; Explorer chooses a name revoke while promising aggregate removal, leaving root state untouched.                                                      |
| WEB-1305 — transferring V2 name roles to a V1 name remains pending                                                                                | WEB-1305 concerns an unsupported transfer stuck in pending UI. ENS-005 concerns successful mined calldata with the wrong resource and usable post-success authority.                                                                                                                                                |
| R2-06 — smart-account session revocation clears browser storage while a time-boxed on-chain co-owner persists                                     | R2-06 concerns the disclosed session-key/co-owner design and its stated seven-day authority bound. ENS-005 does not use sessions, browser-stored keys, or smart-account co-owners; it sends a resolver-role revocation with the wrong root/name resource despite promising complete removal.                        |

The Permissioned Resolver behaves according to the remaining root role. The in-scope Explorer application is responsible for collapsing scopes and submitting incomplete revocation calldata.

## Impact Details

The exact Critical impact selected for this submission is:

> **Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters; substituting contract addresses; submitting malicious transactions; signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated); inducing a blanket token approval (for example `setApprovalForAll` on the registry or NameWrapper) to an attacker-controlled address.**

The demonstrated mapping is exact:

* O uses Explorer's connected-wallet role-management flow.
* The UI represents the action as complete removal across all displayed roles.
* Explorer substitutes a single name resource for the aggregate root-plus-name set.
* O approves one successful `authorizeNameRoles(name, 17, D, false)` transaction.
* A correct complete removal required root revocation as well.
* The wrong resource leaves effective address-setting authority true.
* D uses that authority to redirect canonical resolution from O to R.
* Distinct payer P sends `0.01 ETH` to R, which gains exactly `0.01 ETH`.

The classification follows the contest's enumerated connected-wallet wrong-parameter impact. Recoverability reduces practical persistence and likelihood but does not change what the vulnerable transaction does. There is no claim that the name NFT or owner wallet is taken, that D's authority is hidden after refresh, or that recovery is impossible.

The direct financial victim demonstrated by the PoC is P, whose `0.01 ETH` reaches R after resolving the affected name. O suffers unauthorized loss of record integrity, possible lost receivables, and reputational harm. The local-fork amount is a safe proof value rather than a claim of production loss or a maximum bound.

## Risk Breakdown

| Gate                                                  | Verdict | Evidence                                                                                        |
| ----------------------------------------------------- | ------- | ----------------------------------------------------------------------------------------------- |
| Does the owner use the intended UI correctly?         | **YES** | O selects D's aggregate row and confirms the explicit all-role removal.                         |
| Does Explorer generate the wrong authorization scope? | **YES** | The mined call is name-scoped `authorizeNameRoles`; no `revokeRootRoles` call exists.           |
| Does the transaction succeed?                         | **YES** | The owner-originated resolver receipt has successful status.                                    |
| Does dangerous authority survive?                     | **YES** | Root bitmap remains `1`, name bitmap becomes `0`, and effective `ROLE_SET_ADDR` remains true.   |
| Is the retained role usable after success?            | **YES** | D's post-removal `setAddr` transaction succeeds.                                                |
| Does canonical ENS resolution change?                 | **YES** | Universal Resolver changes from O before grants to R after D's write.                           |
| Is there a concrete third-party loss?                 | **YES** | P sends `0.01 ETH`; R's balance increases by exactly `0.01 ETH`.                                |
| Is authority permanent or irrecoverable?              | **NO**  | The root row is visible after refresh and a correct root revoke removes authority.              |
| Does ordinary record repair revoke D?                 | **NO**  | O must separately revoke the root role; changing the record alone does not clear authorization. |
| Does the path require a malicious ENS/project role?   | **NO**  | D is an ordinary former/compromised delegate; no project operator acts incorrectly.             |

Severity is Critical under the enumerated connected-wallet parameter impact. Likelihood is conditional on a mixed-resource delegate, an attempted all-role removal, D acting before corrective revocation, and a payer interacting while the record is redirected. These conditions and the recovery path should be assessed separately from the demonstrated impact.

## References

* [Resolver overview query and independent role fields](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L60-L119)
* [Resolver role definitions and resource-to-name mapping](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L1-L160)
* [Account-only grouping and role flattening](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L162-L220)
* [Aggregate resolver roles table](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L65-L137)
* [First-non-root scope selection](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L173)
* [Removal mutation](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L251-L287)
* [Aggregate removal pending action](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323)
* [Generic transaction metadata](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L338-L349)
* [Global-scope label and removal confirmation](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L389-L448)
* [Prepared resolver-role removal intent](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts#L32-L46)
* [Name-versus-root revocation selection](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L28-L74)
* [Post-transaction indexer polling](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/query/pollForIndexerSync.ts#L36-L73)
* [ENS audit competition scope and known issues](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [Related but distinct contracts-v2 issue #418](https://github.com/ensdomains/contracts-v2/issues/418)

## Link to Proof of Concept

<https://gist.github.com/0xgo4ko/5aa9e5a8ce7a421b5427b3ebc85748fa>

## Proof of Concept

### Secret Gist environment and integrity

The complete runnable PoC package is provided in the GitHub Secret Gist below. The Gist supports—but does not replace—the explanation, decisive assertions, reproduction steps, and preserved result included in this report. GitHub treats each Gist as a Git repository and provides a Download ZIP action, so reviewers can either clone the pinned revision or download the complete package.

| Gist reference | Value                                                              |
| -------------- | ------------------------------------------------------------------ |
| Web URL        | <https://gist.github.com/0xgo4ko/5aa9e5a8ce7a421b5427b3ebc85748fa> |
| Clone URL      | <https://gist.github.com/5aa9e5a8ce7a421b5427b3ebc85748fa.git>     |
| Pinned commit  | `cc17b1566b9a90d946d2ed31bc7dc6178d16e70c`                         |

Reviewers should read `RUN.md` first and follow it as the authoritative package-verification, environment-setup, reproduction, and cleanup guide. The essential commands are reproduced below so this report remains self-contained.

The pinned Gist contains these files at its repository root:

| File                                   | Purpose                                                                                              |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `RUN.md`                               | Locked-revision preparation, local-fork startup, reproduction, expected result, and cleanup.         |
| `EVIDENCE.md`                          | Evidence boundary, exact state transition, payment result, recovery, and preserved artifacts.        |
| `HYP-027-portal.e2e.spec.ts`           | Production-browser/local-fork E2E exploit and correct-revocation control.                            |
| `HYP-027.playwright.config.mjs`        | Sidecar Playwright configuration that starts the locked Portal source.                               |
| `ENS-005-mined-transaction-proof.json` | Preserved actors, calldata decode, transaction hashes, role state, resolution, payment, and control. |
| `ENS-005-playwright-report.json`       | Unmodified machine-generated successful Playwright JSON report.                                      |
| `ENS-005-success-trace.zip`            | Complete successful Playwright trace with embedded executed source.                                  |
| `package.json`                         | ESM metadata and convenience test command.                                                           |
| `SHA256SUMS`                           | Integrity manifest covering every non-manifest package file.                                         |

The Gist was created with **Secret** visibility, and its URL is disclosed only through this private competition submission. Secret Gists are unlisted rather than access-controlled private repositories; the package contains no production credentials or unrelated secrets.

**Role of the supplied trace.** `ENS-005-success-trace.zip` is not a runtime input to the exploit and does not need to be extracted before running the PoC. The E2E and configuration execute the reproduction independently and generate a fresh trace under `reproduction-output/`. The supplied trace is preserved evidence of the original successful execution. It should remain unchanged in the Gist because `SHA256SUMS` covers it and reviewers can inspect the successful browser flow before reproducing it.

Clone and pin the verified package with:

```sh
git clone "https://gist.github.com/5aa9e5a8ce7a421b5427b3ebc85748fa.git" ENS-005-PoC
cd ENS-005-PoC
git checkout --detach "cc17b1566b9a90d946d2ed31bc7dc6178d16e70c"
test "$(git rev-parse HEAD)" = "cc17b1566b9a90d946d2ed31bc7dc6178d16e70c"
sha256sum -c SHA256SUMS
unzip -tq ENS-005-success-trace.zip
```

On macOS, use `shasum -a 256 -c SHA256SUMS`. Integrity verification must report every listed file as `OK`, and ZIP validation must report no compressed-data errors.

### Evidence boundary

This is a production-browser/local-chain E2E with one controlled indexer boundary. A public ENS indexer cannot observe roles created dynamically inside the isolated Anvil fork. The test controls the Portal indexer GraphQL endpoint: for the security-relevant `getResolverOverview` query, it builds every returned root/name role entry from fresh on-chain `roles(resource, delegate)` reads; for unrelated indexer queries encountered during the isolated browser flow, it returns empty data. It requires an initial resolver-overview request and a second request after a full page reload.

The controlled boundary supplies transport data that the production UI requires; it does not decide whether D is authorized. The real Permissioned Resolver storage, application grouping, table, sidebar, confirmation, transaction modal, connected headless wallet, mined application calldata, role inheritance, `setAddr`, Universal Resolver lookup, payer transaction, and correct-revocation control are exercised.

Test-only RPC methods create and fund distinct local actors and the name fixture. They do not mutate production systems or grant authority during the exploit. The owner legitimately creates the two starting grants, and Anvil reverts the complete snapshot in `finally`.

### Decisive exploit assertions

The following abridged proof logic from `HYP-027-portal.e2e.spec.ts` places the decisive checks directly in the Immunefi report. It is explanatory pseudocode derived from non-contiguous parts of the test—not a verbatim source excerpt—and simplifies some local variables while preserving the asserted state transitions. Helper declarations, wallet-authorization plumbing, GraphQL response construction, and evidence serialization are omitted. Reproduction executes the complete Gist source.

```ts
// Establish a legitimate recipient before any delegate roles exist.
const initialOwnerAddressWrite = await sendResolverWrite(
  owner,
  resolverAddress,
  encodeFunctionData({
    abi: resolverAbi,
    functionName: 'setAddr',
    args: [node, ETH_COIN_TYPE, owner.address as Hex],
  }),
)
const initialResolvedAddress = await getEnsAddress(ensClient, {
  name,
  universalResolverAddress: CONTRACTS.ensUniversalResolver.address,
})
expect(initialResolvedAddress?.toLowerCase()).toBe(owner.address.toLowerCase())

// Use two independent resources and nonredundant roles.
await sendRoleWrite(
  owner,
  grantResolverRolesWriteParameters(ownerWallet, {
    resolverAddress,
    targetAccount: delegate.address,
    scope: 'root',
    roles: ['ROLE_SET_ADDR'],
  }),
)
await sendRoleWrite(
  owner,
  grantResolverRolesWriteParameters(ownerWallet, {
    resolverAddress,
    targetAccount: delegate.address,
    scope: 'name',
    name,
    roles: ['ROLE_SET_TEXT'],
  }),
)

expect(await readRawRoles(resolverAddress, 0n, delegate.address)).toBe(1n)
expect(await readRawRoles(resolverAddress, nameResource, delegate.address)).toBe(16n)
expect(
  await hasRoles(publicClient, {
    resolverAddress,
    resource: nameResource,
    roles: ['ROLE_SET_ADDR'],
    account: delegate.address,
  }),
).toBe(true)

// Production Explorer collapses the resources into one row.
const delegateRow = page
  .getByRole('row')
  .filter({ hasText: delegate.address.slice(-4) })
await expect(delegateRow).toHaveCount(1)
await expect(delegateRow).toContainText('(root)')
await expect(delegateRow).toContainText(name)
await expect(delegateRow).toContainText('Set Address')
await expect(delegateRow).toContainText('Set Text')

await delegateRow.getByRole('button', { name: 'Edit user roles' }).click()
await expect(sheet.getByText('Global roles (all names)')).toBeVisible()
await sheet.getByRole('button', { name: 'Remove user' }).click()
await expect(confirmation).toContainText('remove this user from all roles')

// O approves one application-generated resolver transaction.
const beforeRemovalBlock = await getLatestBlockNumber()
await confirmation.getByRole('button', { name: 'Remove' }).click()
const walletApprovalCount = await driveRemovalToSuccess(page, wallet)
const afterRemovalBlock = await getLatestBlockNumber()
expect(walletApprovalCount).toBe(1)

const ownerResolverTransactions = (
  await collectTransactions(beforeRemovalBlock + 1n, afterRemovalBlock)
).filter(
  (transaction) =>
    transaction.from.toLowerCase() === owner.address.toLowerCase() &&
    transaction.to?.toLowerCase() === resolverAddress.toLowerCase(),
)
expect(ownerResolverTransactions).toHaveLength(1)

const [applicationTransaction] = ownerResolverTransactions
const applicationReceipt = await publicClient.getTransactionReceipt({
  hash: applicationTransaction.hash,
})
expect(applicationReceipt.status).toBe('success')

const decoded = decodeFunctionData({
  abi: resolverAbi,
  data: applicationTransaction.input,
})
expect(decoded.functionName).toBe('authorizeNameRoles')
expect(decoded.args).toEqual([
  toHex(packetToBytes(name)),
  resolverRoles.ROLE_SET_ADDR | resolverRoles.ROLE_SET_TEXT,
  delegate.address,
  false,
])

// The name resource clears, but root address authority remains effective.
expect(await readRawRoles(resolverAddress, nameResource, delegate.address)).toBe(0n)
expect(await readRawRoles(resolverAddress, 0n, delegate.address)).toBe(1n)
expect(
  await hasRoles(publicClient, {
    resolverAddress,
    resource: nameResource,
    roles: ['ROLE_SET_ADDR'],
    account: delegate.address,
  }),
).toBe(true)

// Prove the surviving authority from a fresh Explorer view, not stale UI.
const indexerRequestsBeforeRefresh = getIndexerRequestCount()
await page.reload({ waitUntil: 'domcontentloaded' })
await expect
  .poll(getIndexerRequestCount, { timeout: 30_000 })
  .toBeGreaterThan(indexerRequestsBeforeRefresh)
const refreshedDelegateRow = page
  .getByRole('row')
  .filter({ hasText: delegate.address.slice(-4) })
await expect(refreshedDelegateRow).toHaveCount(1)
await expect(refreshedDelegateRow).toContainText('(root)')
await expect(refreshedDelegateRow).toContainText('Set Address')
await expect(refreshedDelegateRow).not.toContainText(name)
await expect(refreshedDelegateRow).not.toContainText('Set Text')
await expect(partialRemovalWarning).toHaveCount(0)

// D changes canonical resolution only after the successful flow and refresh.
const delegateWrite = await sendResolverWrite(
  delegate,
  resolverAddress,
  encodeFunctionData({
    abi: resolverAbi,
    functionName: 'setAddr',
    args: [node, ETH_COIN_TYPE, attacker.address as Hex],
  }),
)
expect(delegateWrite.receipt.status).toBe('success')

const canonicalResolvedAddress = await getEnsAddress(ensClient, {
  name,
  universalResolverAddress: CONTRACTS.ensUniversalResolver.address,
})
expect(canonicalResolvedAddress?.toLowerCase()).toBe(
  attacker.address.toLowerCase(),
)

// Independent payer P loses exactly the safe demonstration amount to R.
const attackerBalanceBefore = await publicClient.getBalance({
  address: attacker.address,
})
const paymentHash = await payerWallet.sendTransaction({
  account: payer,
  chain: payerWallet.chain,
  to: canonicalResolvedAddress,
  value: parseEther('0.01'),
})
expect(
  (await publicClient.waitForTransactionReceipt({ hash: paymentHash })).status,
).toBe('success')
const attackerBalanceAfter = await publicClient.getBalance({
  address: attacker.address,
})
expect(attackerBalanceAfter - attackerBalanceBefore).toBe(parseEther('0.01'))

// Correct root revocation is a recovery control and makes D's next write fail.
await sendRoleWrite(
  owner,
  revokeResolverRolesWriteParameters(ownerWallet, {
    resolverAddress,
    targetAccount: delegate.address,
    scope: 'root',
    roles: ['ROLE_SET_ADDR'],
  }),
)
expect(await readRawRoles(resolverAddress, 0n, delegate.address)).toBe(0n)
expect(
  await hasRoles(publicClient, {
    resolverAddress,
    resource: nameResource,
    roles: ['ROLE_SET_ADDR'],
    account: delegate.address,
  }),
).toBe(false)
await expect(
  sendResolverWrite(
    delegate,
    resolverAddress,
    encodeFunctionData({
      abi: resolverAbi,
      functionName: 'setAddr',
      args: [node, ETH_COIN_TYPE, attacker.address as Hex],
    }),
  ),
).rejects.toThrow()
```

### Exact snapshots

| Snapshot                                         | Commit/block                                                                                            |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| Official audited upstream revision               | `63772fd872af472ced58b009499355f3430c2a86`                                                              |
| Immunefi mirror import used by the preserved run | `cda79acaad59711b943fc68207ebb3f1d0ff8596`                                                              |
| Sepolia fork block                               | `11586545`                                                                                              |
| Pinned Anvil image                               | Foundry/Anvil `1.7.1`, digest `sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd` |

The test checks repository `HEAD` and tracked status before and after the browser flow. The preserved proof records a clean worktree at the locked mirror commit. The future Gist commit must be pinned separately so later Gist edits cannot silently change the package reviewed with this report.

### Required layout

```
parent/
├── audit-comp-ens/     # official mirror at cda79aca...
└── ENS-005-PoC/        # complete pinned Secret Gist package
```

Clone the Gist with the explicit destination `ENS-005-PoC`. If using GitHub's **Download ZIP**, extract the archive, place its top-level directory beside `audit-comp-ens`, and use that directory as `ENS-005-PoC` when following the commands below.

### Requirements

* Git
* Docker
* Node.js and Corepack
* pnpm version resolved from the locked repository
* Chromium installed through the locked Playwright dependency
* unzip
* an archival Sepolia RPC capable of serving block `11586545`
* Internet access for the repository, dependencies, container image, and fork RPC

### Verify the package

From `ENS-005-PoC/`:

```sh
sha256sum -c SHA256SUMS
unzip -tq ENS-005-success-trace.zip
```

On macOS:

```sh
shasum -a 256 -c SHA256SUMS
```

### Prepare the exact audited source

From `parent/`:

```sh
git clone https://github.com/immunefi-team/audit-comp-ens.git audit-comp-ens
git -C audit-comp-ens checkout --detach \
  cda79acaad59711b943fc68207ebb3f1d0ff8596

test "$(git -C audit-comp-ens rev-parse HEAD)" = \
  "cda79acaad59711b943fc68207ebb3f1d0ff8596"
test -z "$(git -C audit-comp-ens status --porcelain)"

corepack pnpm --dir audit-comp-ens install --frozen-lockfile
corepack pnpm --dir audit-comp-ens \
  --filter @ens-apps/e2e exec playwright install chromium
```

### Start the pinned local fork

Use an archival Sepolia endpoint and bind Anvil only to localhost:

```sh
export SEPOLIA_FORK_URL='https://your-archival-sepolia-rpc.example'

docker run --rm --name ens005-anvil -p 127.0.0.1:8545:8545 \
  --entrypoint anvil \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd \
  --fork-url "$SEPOLIA_FORK_URL" \
  --fork-block-number 11586545 \
  --chain-id 11155111 \
  --port 8545 \
  --host 0.0.0.0
```

A non-archival endpoint may fail because historical state is pruned. That is an RPC capability error rather than evidence about the PoC.

### Run the browser reproduction

From `ENS-005-PoC/` in a second terminal:

```sh
ln -sfn ../audit-comp-ens/e2e/node_modules node_modules
test ! -e reproduction-output

../audit-comp-ens/e2e/node_modules/.bin/playwright test \
  --config HYP-027.playwright.config.mjs
```

The fresh-output guard prevents old and new artifacts from being mixed. A successful run produces:

```
reproduction-output/ENS-005-playwright-report.json
reproduction-output/test-results/.../ENS-005-mined-transaction-proof.json
reproduction-output/test-results/.../trace.zip
```

Cleanup only resources created for this reproduction:

```sh
unlink node_modules
docker stop ens005-anvil
```

### Expected result

```
Running 1 test using 1 worker
✓ Explorer remove-all flow leaves root address authority usable
1 passed
```

The preserved machine report records:

```
expected:   1
unexpected: 0
flaky:      0
skipped:    0
duration:   14.9 seconds overall
```

### Preserved mined evidence

`ENS-005-mined-transaction-proof.json` records the following successful local-fork execution:

| Evidence                            | Preserved value                                                                     |
| ----------------------------------- | ----------------------------------------------------------------------------------- |
| Name                                | `ens005-e2e-mthunp81-1788217717.eth`                                                |
| Resolver                            | `0xEA26bD887CCab0d11EFEBbe73173416124397537`                                        |
| O / owner                           | `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`                                        |
| D / delegate                        | `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`                                        |
| P / payer                           | `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`                                        |
| R / attacker recipient              | `0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65`                                        |
| Legitimate baseline `setAddr`       | `0xcef91d6fe40ea1baaf7e50e5458f80880c0a06fa3e97d7acc015dc9eb5a2ce42`                |
| Explorer removal transaction        | `0xe8e4dfb682041d3b1eb040af5d4cc9169b45ee83bd85b747514e1af3364646e4`                |
| Removal function / bitmap           | `authorizeNameRoles` / `17` / `grant=false`                                         |
| State after application success     | root `1`, name `0`, effective `Set Address=true`                                    |
| Role-data requests around reload    | `1` before, `2` after                                                               |
| D's successful malicious `setAddr`  | `0x2b8f229329a679cf8f5f05ef16fb10e9422d7ea4bbbf0b18a91ee3444a4b323e`                |
| Canonical recipient after D's write | `0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65`                                        |
| P's payment                         | `0x80819dcc8c4e784ece822797267993df334a2aca0a2bbed1869a8ded45e5351e`                |
| Payment / R balance increase        | `10000000000000000 wei` / `10000000000000000 wei`                                   |
| Correct root revocation             | `0x6de84d334e4e4abc1773d4bc80c97c53071e02a1281c57a39a2c9bc4d79510ea`                |
| Final control                       | root `0`, effective role false, D's second write reverted, resolution restored to O |

These transaction hashes identify the deterministic local-fork execution and are not represented as Sepolia production transactions.

### What the browser PoC proves

1. The exact locked Portal source loads the resolver roles route.
2. Independent root and name roles for D are derived from live resolver storage.
3. Production grouping renders one aggregate row.
4. The production sidebar calls the row global and promises removal from all roles.
5. O approves exactly one resolver transaction generated by the application.
6. The mined calldata is bound to O, D, the resolver, the name, bitmap `17`, and `grant=false`.
7. The application submits no root revoke.
8. The name bitmap clears but the root bitmap and effective address role remain.
9. A fresh page reload and second on-chain-derived overview still show the root authority.
10. D successfully changes the canonical address only after the successful removal and refresh.
11. Universal Resolver changes from O to R.
12. Independent P sends exactly `0.01 ETH` to R.
13. A correct root revoke clears authority and makes D's next write revert.
14. The repository remains at the locked clean revision and the Anvil snapshot is reverted.

### Recovery and negative-control interpretation

The correct root revoke is not part of the exploit and does not erase the demonstrated payment. It establishes three important limits and controls:

1. The Permissioned Resolver can remove D correctly.
2. Explorer's wrong resource selection is the cause of retained authority.
3. O can prevent future exploitation after noticing the surviving role and submitting another transaction.

The report intentionally discloses this recovery. The finding is a recoverable but exploitable failed-revocation window, not a permanent or invisible takeover.

### Test-only setup is not the exploit

The following operations are fixture construction or isolation controls:

* Local Anvil account funding and code normalization.
* Creation of a fresh V2 name on the fork.
* O's initial legitimate address record.
* O's deliberate root/name grants to D.
* Controlled GraphQL transport for local-fork role entries.
* The final correct root revocation, owner repair, and snapshot revert.

The exploit begins when production Explorer collapses the independent resources and O uses the ordinary all-role removal flow. Production UI code constructs and submits the wrong-scope transaction. Real resolver storage, role inheritance, canonical resolution, D's malicious record write, and P's payment are not mocked.

### Evidence integrity

* `SHA256SUMS` covers every supplied non-manifest file, including the preserved trace.
* All package hashes were verified during final review.
* `ENS-005-success-trace.zip` passed compressed-data validation.
* The trace contains the production browser action ordering, including the all-role confirmation and post-success reload.
* `ENS-005-playwright-report.json` records one passed test and zero unexpected, flaky, or skipped tests.
* The proof JSON binds the UI assertions to exact mined transactions, state reads, Universal Resolver output, payer transfer, and recovery control.
* The E2E checks the locked repository commit and tracked cleanliness before and after execution.
