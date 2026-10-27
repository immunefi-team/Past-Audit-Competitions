# 92452 w a critical explorer collapses per name resolver roles by account so remove user leaves a delegate able to redirect another ens name to an attacker controlled address

**Submitted on Sep 9th 2026 at 22:53:38 UTC by @Parthx108 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92452
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

**Affected in-scope asset:** [Explorer app files (`apps/portal`)](https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal)

**Submitted severity:** Critical

**PoC classification:** Websites and Applications (Web2), with an on-chain authorization consequence

**Selected in-scope impact:**

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
>
> * resolving or rendering an ENS name to an attacker-controlled address

A `PermissionedResolver` stores authorization independently for each `(resource, account)` pair. The same account can therefore hold `ROLE_SET_ADDR` for several names on one resolver without holding any root/global role.

Explorer receives those independent entries, but its editable roles table groups them only by account. The grouped row still retains the original entries, yet the removal path ignores their resource association: it flattens their decoded role labels, chooses only the first non-root name in the returned entry order, and builds one name-scoped revocation. The confirmation says the user will be removed “from all roles,” and the transaction succeeds, although every other name-scoped authorization for that account remains active.

A previously trusted delegate that later acts maliciously can then use a surviving `ROLE_SET_ADDR` grant to replace another name's legitimate Ethereum address record (coin type `60`) with an attacker-controlled address. The resolver contract correctly enforces the calldata it receives; the vulnerability is Explorer's construction of an incomplete removal transaction.

## Brief/Intro

In the scoped ENSv2 Explorer on Sepolia, an administrator can give a trusted delegate permission to edit address records for several ENS names. If the administrator later uses Explorer's normal **Remove user** action, Explorer shows all of those names together and promises to remove all roles, but it revokes only one name. The delegate retains `ROLE_SET_ADDR` on every unselected name despite the successful advertised removal and can replace a legitimate address record with the delegate's address.

## Vulnerability Details

The resolver overview preserves the security boundary in its source data: every role entry has a separate `account`, `resource`, and `roleBitmap`. The loss of scope occurs when `groupRolesByAccount` keys the editable UI row solely by the lowercased account. For each later resource belonging to the same account, it appends the original entry, concatenates the decoded role labels, and adds the resolved name to the same account-level row:

```typescript
const account = role.account.toLowerCase()
const existing = grouped.get(account)

if (existing) {
  existing.roles.push(role)
  existing.decodedRoles.push(...decoded)
  if (resolvedName) existing.resolvedNames.add(resolvedName)
} else {
  // one group keyed only by account
}
```

`ResolverRolesTable` renders that account-level group as one editable row and passes it to `ResolverRolesSidebar`. The sidebar reads the flattened role labels and all displayed names, but derives only one transaction scope:

```typescript
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

JavaScript `Set`/array insertion order means `roleName` is the first non-root name in the returned entry order; it is not a semantically selected resource. The sidebar visibly lists every merged name, then its **Remove user** handler stores only that one name together with the flattened roles:

```typescript
`Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`

setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

The confirmation text overstates the operation:

```
Are you sure you want to remove this user from all roles?
```

`prepareRevokeResolverRolesTransaction` treats every non-empty `name` as exactly one name-scoped revocation:

```typescript
name === ''
  ? { resolverAddress, targetAccount: account, scope: 'root', roles }
  : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
```

Consequently, a row representing names `A`, `B`, `C`, and `D` produces one call for whichever name appears first. That name is revoked and the other three permissions survive. Reversing the input order changes which name is revoked, but not the incomplete-revocation outcome. There is no loop over `row.original.roles`, no batch covering each exact resource/bitmap pair, and no post-transaction condition requiring every original resource to be cleared before the single submitted revocation is treated as done.

**Required preconditions:**

* One `PermissionedResolver` serves at least two names.
* The same delegate was intentionally granted name-scoped `ROLE_SET_ADDR` on at least two of those names.
* The previously trusted delegate retains its own key and later acts maliciously.
* A connected resolver administrator recognized by Explorer's root `ROLE_SET_ADDR` check uses the normal **Remove user** action.

The attacker needs no root role, resolver ownership, governance privilege, victim credential, race, indexer defect, or malicious resolver.

The initial name-scoped grants are legitimate setup state. At exploitation time, the administrator has accepted Explorer's explicit all-role confirmation and the resulting removal transaction has succeeded; the surviving name-scoped grant is therefore the vulnerability's result, not an assumed root, governance, ownership, or administrator privilege.

The shared-resolver topology is supported by normal Explorer flows rather than being a synthetic contract configuration: `ChangeResolverForm` lets a user attach a name to an existing permissioned resolver, and `ResolverAddUserSheet` lets an administrator select any name attached to that resolver and grant a delegate name-scoped roles. Repeating that supported grant for a second attached name creates the affected state.

**Exploit sequence:** the administrator accepts Explorer's explicit all-role confirmation; Explorer submits one valid revocation; the transaction succeeds; at least one other name-scoped role remains; the removed delegate calls `setAddr` for a surviving name; and the standard resolver read `addr(namehash(name), 60)` returns the attacker's address.

**Recommended correction:** editable authorization state must preserve `(account, resource, roleBitmap)`. Render resource-specific child rows or, for **Remove user**, enumerate `row.original.roles` and prepare the matching revocation for every exact resource. Submit the set atomically where possible and verify each original resource is cleared before reporting success. Do not use `resolvedNames.find(...)` as the authorization scope selector or apply a flattened role list to one arbitrary resource.

## Impact Details

The PoC proves this post-removal state without giving the attacker any root role:

```
attacker root role    = 0x0
selected name         = 0x1 before removal, 0x0 after removal
unselected name       = 0x1 before removal, 0x1 after removal
unselected name addr  = victim before attack, attacker after delegate write
```

Thus, after the administrator accepts Explorer's explicit all-role confirmation and the resulting transaction succeeds, the delegate still holds `ROLE_SET_ADDR` on a surviving name. After the delegate's write, a fresh standard resolver read `addr(namehash(survivingName), 60)` returns the attacker-controlled Ethereum destination. A wallet or application using that resolution result can display or pre-fill the attacker's address for a subsequent transfer. This exactly matches the program's Critical example, “resolving or rendering an ENS name to an attacker-controlled address.”

This is an Explorer transaction-construction failure that leaves residual on-chain authorization, not a stale visual row and not a resolver-contract authorization bypass. Explorer is an explicitly scoped asset. The underlying contracts are used only as the integration witness showing that the incomplete calldata has a real, persistent authorization consequence.

Exploitation is deterministic once one resolver serves multiple names, the same delegate has name-scoped permissions on more than one of them, and the resolver administrator uses Explorer's advertised all-role removal. It does not depend on a race, malicious resolver, leaked credential, global attacker privilege, browser defect, or indexer failure. One surviving name is sufficient for the proved attacker-controlled resolution consequence.

The impact boundary is destination substitution, not custody of existing wallet funds: the surviving role cannot sign from the administrator's wallet, transfer name ownership, obtain root authority, or move existing balances. The proof does not perform a real third-party payment. It proves that, after Explorer's explicit all-role confirmation and the successful transaction it constructs, standard coin-type-60 ENS resolution for the surviving name returns the attacker's address—the attacker-controlled-resolution example expressly enumerated by the selected Critical impact.

No listed public known issue describes this account-only collapse of independent `PermissionedResolver` resources followed by a one-resource revocation. In particular, QA-01/WEB-1257 concern post-transfer failures, WEB-1305 is a V2-to-V1 role-transfer hang, and R2-06 concerns browser-local Hybrid Custody Account session revocation.

## References

* [ENS audit competition scope and accepted impacts](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [ENS audit competition resources and environment](https://immunefi.com/audit-competition/audit-competition-ens/resources/)
* [ENS audit competition information and listed known issues](https://immunefi.com/audit-competition/audit-competition-ens/information/)
* [Pinned resolver overview schema preserves account, resource, and bitmap](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L14-L22)
* [Pinned account-only grouping implementation](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220)
* [Pinned editable grouped-table path](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L90-L138)
* [Pinned first-name selection and pending removal](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L146-L173)
* [Pinned Remove user action stores one name](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323)
* [Pinned all-role confirmation and one-action execution](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L493)
* [Pinned one-name revoke builder](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74)
* [Pinned root-role gate for resolver administration](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/resolver/%24address/roles/index.tsx#L41-L50)
* [Pinned Explorer flow for attaching a name to an existing permissioned resolver](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ChangeResolverForm.tsx#L267-L300)
* [Pinned Explorer flow for selecting a resolver name and granting roles](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L183-L211)
* [Pinned name selector displayed by the Add User flow](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L250-L290)
* [Immunefi step-by-step PoC requirements for audit competitions](https://immunefisupport.zendesk.com/hc/en-us/articles/33260632501777-Step-by-step-POCs-for-Audit-Competitions)
* [Immunefi Web2 PoC guidelines](https://immunefisupport.zendesk.com/hc/en-us/articles/18722746161553-Web2-PoC-guidelines)
* [Immunefi Web3 PoC guidelines, used here as supplemental private-fork safety guidance](https://immunefisupport.zendesk.com/hc/en-us/articles/18722863230353-Web3-PoC-guidelines)
* [Immunefi external PoC/Gist submission guidance](https://immunefisupport.zendesk.com/hc/en-us/articles/12225280961425-PoC-Submission-Update)
* [Official ENS application and network context](https://support.ens.domains/en/articles/8874786-official-ens-websites-apps-and-tools)

## Link to Proof of Concept

<https://gist.github.com/savyasachi-bugbounty/563c833e68a03dc023686647790f0734/8591a993ff7a625619033570be0014897abc59e2>

## Proof of Concept

## Proof of Concept

**Code Gist, pinned to an immutable revision:** [Critical-02 runnable PoC code](https://gist.github.com/savyasachi-bugbounty/563c833e68a03dc023686647790f0734/8591a993ff7a625619033570be0014897abc59e2)

**Separate setup Gist, pinned to an immutable revision:** [Critical-02 reproduction guide](https://gist.github.com/savyasachi-bugbounty/82b2228f0805d10083ba7ac7c195eeba/d394ad33fca47f4689773cf92e34a5f2d3586788)

**Applicable PoC standards:** This is a Websites and Applications (Web2) report because the vulnerable root is Explorer's client-side grouping and transaction construction. ENS requires a step-by-step PoC, while Immunefi accepts command-line or programmable runnable code for Web2 reports. The eight steps below provide the complete preconditions, exploit, and impact sequence and map directly to the runner's eight logged stages. No video is submitted, so the video-specific HTTP-request requirement does not apply; the flaw is client-side transaction construction, not an HTTP endpoint. Because the consequence is on-chain, the proof also follows the relevant Web3 safety and reproducibility practices by pinning the environment, printing the security-relevant state transitions, and confining every write to a private fork.

The code Gist contains exactly:

* `multi-name-resolver-removal.poc.ts`
* `run-primary.sh`
* `tsconfig.json`

The setup Gist contains only `Critical-02-PoC-setUp-README.md`. It provides the exact clone layout, dependency versions, frozen commit, private-fork command, safety boundary, and expected output.

**Setup:** Follow the separate setup Gist to clone the official competition wrapper and detach it at `cda79acaad59711b943fc68207ebb3f1d0ff8596`. That wrapper commit records the program-listed upstream provenance `63772fd872af472ced58b009499355f3430c2a86`. Clone the revision-pinned code Gist into `AuditFolder/Critical-02-PoC-Folder` beside the `ens-audited` checkout, install the frozen repository dependencies with Node.js `v22` and pnpm `10.27.0`, provide a historical Sepolia RPC URL, and run `bash AuditFolder/Critical-02-PoC-Folder/run-primary.sh`.

**Step-by-step attack and verification:**

1. On the private Anvil fork, deploy a proxy using the configured deployed `PermissionedResolver` implementation, register two victim-owned V2 names to that resolver, and set both coin-type-60 records to the victim's address. The registrar is impersonated only for local fixture registration; this is not attacker authority or an exploit step.
2. Using Explorer's frozen in-scope grant builder, grant the attacker exactly name-scoped `ROLE_SET_ADDR` on both names.
3. Confirm the administrator has the root `ROLE_SET_ADDR` required by Explorer's management gate, both attacker name resources equal exactly `0x1`, and the attacker root resource equals exactly `0x0`.
4. Construct accurate `(account, resource, roleBitmap)` entries from those on-chain reads and pass them through the frozen in-scope `groupRolesByAccount`. Confirm Explorer's data transformation creates one account row containing both names.
5. Read the frozen `ResolverRolesSidebar.tsx` source at runtime, confirm it contains both the exact `resolvedNames.find((n) => n !== '(root)')` selector and the advertised all-role confirmation, then apply that selector. Reverse the entries and confirm that the first selected name changes, proving the selector follows input order rather than resource semantics.
6. Pass the single selected name and grouped role labels to Explorer's frozen in-scope revoke builder. Decode the result and confirm it contains one `authorizeNameRoles(selectedName, ROLE_SET_ADDR, attacker, false)` call, then submit that builder-produced calldata as the resolver administrator. Print the exact calldata, transaction hash, receipt status, and block number.
7. After the successful removal transaction, read both exact name resources. Confirm the selected name changed from `0x1` to `0x0`, while the unselected name remains exactly `0x1`.
8. Confirm the surviving name still resolves to the victim, then call `setAddr` from the delegate that Explorer's flow was intended to remove for coin type `60`. Confirm the transaction succeeds and `addr(namehash(survivingName), 60)` changes from the victim's address to the attacker's address. The JSON field `"verdict"` must equal `"CONFIRMED"`, the runner must end with `Critical-02 primary PoC: PASS`, and the focused resolver-role tests must report `16 passed`.

All writes occur only on the private Anvil fork of Ethereum Sepolia at block `11556186`, chain ID `11155111`; no public-chain transaction is broadcast. The executable proof establishes the frozen in-scope grouping/builder defect and its on-chain consequence. The cited component source shows the intervening React wiring, while runtime source assertions bind the reproduced selector and confirmation text to the frozen revision. The PoC does not mount the complete React UI or perform a real payment, and neither unexecuted event is claimed.

**Run the PoC**

Enter the historical Sepolia RPC URL without writing it into shell history, then execute the one-command runner:

```bash
set -euo pipefail
printf 'Historical Sepolia RPC URL: ' >&2
IFS= read -r -s SEPOLIA_RPC_URL
printf '\n' >&2
export SEPOLIA_RPC_URL
bash AuditFolder/Critical-02-PoC-Folder/run-primary.sh
unset SEPOLIA_RPC_URL
```

The runner uses local port `18545`. If that port is occupied, set `POC_ANVIL_PORT` to a different unused port before running it.

**Expected result**

Successful execution exits with status `0`, ends with `Critical-02 primary PoC: PASS`, and prints evidence equivalent to:

```
administratorRootRolesBefore includes ROLE_SET_ADDR
rootRolesBefore       = 0x0
selectedNameBefore    = 0x1
survivorNameBefore    = 0x1
selectedNameAfter     = 0x0
survivorNameAfter     = 0x1
addressBeforeAttack   = victim
resolvedAddress       = attacker
selectorVerified      = true
advertisedConfirmationVerified = true
receiptStatus         = success
"verdict": "CONFIRMED"
```

The `exactPreparedRemoval` JSON object also prints the builder-produced calldata, transaction hash, successful receipt status, and local-fork block number. The focused resolver-role test suite must report `16 passed`. Reversing the two input entries changes which name is selected, but one of the two permissions still survives.
