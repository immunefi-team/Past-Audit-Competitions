# 90843 w a critical portal turns an unresolved name scoped resolver role into root granting resolver wide control over unrelated names

**Submitted on Aug 27th 2026 at 14:33:31 UTC by @sl1de for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90843
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

Portal fails open when an indexed PermissionedResolver role has a nonzero resource that it cannot map back to a name. Authentic Panoptes returns the name-scoped role with the resource's low 32 bits cleared, while still returning the exact associated name in the same resolver response. Portal's exact lookup therefore produces no resolved name. The role table leaves the Name cell blank, the editor shows neither a named scope nor its normal `Global roles (all names)` warning, and the sidebar reduces the unresolved scope to the empty string. The transaction builder reserves that same empty string for ROOT.

As a result, a resolver administrator who opens an existing name-scoped collaborator and adds the ordinary `Set Address` permission signs `grantRootRoles(1, attacker)` instead of a name-scoped authorization. I reproduced the full path from the exact-name Roles page through Portal's `View` link, shared-resolver editor, transaction modal, connected wallet, mined transaction, and post-state. Before approval, the collaborator could not alter an unrelated victim name. After the Portal transaction, the collaborator set that name's ETH record to the attacker address, and Portal's own Records page rendered the attacker address.

The same sink also exposes `Upgrade`. A separate private-fork control proves that a genuinely name-scoped Upgrade role cannot upgrade the resolver, whereas Portal's unresolved row signs a ROOT Upgrade grant. The collaborator then replaced the shared PermissionedResolver implementation and made two names owned by different accounts resolve through ENSjs to the attacker.

## Vulnerability Details

### Actors and realistic setup

There are two principal actors:

* The victim administers a PermissionedResolver used by more than one ENSv2 name and has the legitimate root role required to manage resolver collaborators through Portal.
* The attacker is an ordinary collaborator who controls one name using that resolver. The attacker initially has only name-scoped `ROLE_SET_TEXT` on that one name and no root role.

The attacker can point their own ENSv2 name at the victim-administered shared resolver without obtaining resolver authority. The victim then grants the collaborator a low-privilege, name-scoped role for that name. This creates an ordinary existing collaborator row that the victim can later edit.

The expected invariant is:

```
indexed role scope == displayed role scope == confirmed role scope == signed role scope
```

The vulnerable flow is:

```
Attacker has SET_TEXT only on collaborator-name.eth
                         |
                         v
Panoptes returns the role with a lossy nonzero resource
and still returns collaborator-name.eth in resolver.nodes
                         |
                         v
Portal's exact resource lookup cannot recover the name
                         |
                         v
Name cell is blank; no named or Global scope warning appears
                         |
                         v
Victim adds Set Address to the existing collaborator row
                         |
                         v
Portal stores unresolved scope as name = ""
                         |
                         v
Builder interprets "" as ROOT_RESOURCE
                         |
                         v
Victim's connected wallet signs grantRootRoles(1, attacker)
                         |
                         v
Attacker changes an unrelated victim name's ETH record
                         |
                         v
Portal renders the attacker address on the victim's Records page
```

### Root cause 1: an authentic name-scoped role reaches Portal with a lossy resource

Portal requests a resolver's nodes and roles together. Each role includes `account`, `resource`, `roleBitmap`, and `name`:

```graphql
nodes {
  id
  name
}
roles {
  account
  resource
  roleBitmap
  name
}
```

For a name-scoped resolver role, Portal computes the expected full 256-bit resource:

```ts
export const computeNameResource = (name: string): string =>
  keccak256(
    encodePacked(['bytes32', 'bytes32'], [namehash(name), ZERO_BYTES32]),
  )
```

The final browser run observed these values for the same on-chain role:

```
Full on-chain name resource:
0x7b792a90b5b1287e5e7641cae209e052643c6be56042f94fefd86e4ee74d3c23

Panoptes role resource:
0x7b792a90b5b1287e5e7641cae209e052643c6be56042f94fefd86e4e00000000
```

The upper 224 bits are equal, but the indexed value has its lower 32 bits cleared. The same authentic Panoptes response still contained both the collaborator name and victim name in `resolver.nodes`. The role's `name` field was `null`.

This proof does not mock or alter the indexer response. The locally pinned Panoptes service produced the lossy value from real fork events.

### Root cause 2: Portal silently drops the unresolved scope

Portal builds an exact map from each node name to its full computed resource. ROOT alone is explicitly mapped to `(root)`:

```ts
map.set(ROOT_RESOURCE, '(root)')

for (const node of nodes) {
  if (node.name) {
    map.set(computeNameResource(node.name).toLowerCase(), node.name)
  }
}
```

`groupRolesByAccount` then performs a direct lookup with the indexed resource. A miss becomes `null`, and the role is retained without any name:

```ts
const resource = role.resource.toLowerCase()
const resolvedName = resourceToName?.get(resource) ?? null

if (resolvedName) names.add(resolvedName)
```

The final response therefore becomes:

```
decoded role:    ROLE_SET_TEXT
resolvedNames:   []
```

Portal had all information required to detect an inconsistent nonzero resource, but it neither rejects the row nor marks it unsafe.

### Root cause 3: the natural name context is lost at the editable route

The victim can begin at the collaborator's exact name page:

```
/<collaborator-name>/roles
```

That page labels the section `<collaborator-name> resolver roles`, displays the collaborator's `Set Text` role, and offers Portal's normal `View` link. The link opens the shared resolver editor:

```tsx
<Link
  params={{ address: resolverAddress }}
  to="/resolver/$address/roles"
>
  View
</Link>
```

The destination carries only the resolver address. The explicit name context is gone.

On that shared route, the roles table renders no value when `resolvedNames` is empty:

```tsx
const names = row.original.resolvedNames
if (names.length === 0) return null
```

The final full-browser path asserted all of the following:

```
exact-name heading visible:  <collaborator-name> resolver roles
existing role visible:       Set Text
Portal View link followed:   true
shared-resolver Name cell:   blank
```

### Root cause 4: unresolved and ROOT share the same sentinel

The editor knows two legitimate scope displays:

```tsx
{selectedAccount && resolvedNames.length > 0 && (
  <p>
    {resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${...}`}
  </p>
)}
```

For an unresolved role, neither message is rendered. The sidebar then converts that unresolved state to an empty string:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

Save snapshots the empty string into the security-sensitive pending action:

```ts
setPendingAction({
  type: 'save',
  name: roleName,
  account: selectedAccount,
  rolesToGrant,
  rolesToRevoke,
})
```

The grant builder explicitly defines the empty name as ROOT:

```ts
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
    }
```

This changes the ABI operation itself. The selected row originated from a nonzero name resource, but the signed call targets resource zero.

### Root cause 5: the confirmation omits the substituted scope

The victim deliberately checks the Manager box for `Set Address`. Portal's transaction flow then displays only:

```
Save resolver role changes
Update roles for 0x70997970c51812dc3a010c7d01b50e0d17dc79c8
```

It includes neither the collaborator name nor ROOT. The connected-wallet transaction from the final browser run decoded as:

```
grantRootRoles(
  roleBitmap = 1,
  account = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
)
```

Exact mined calldata:

```
0x072d5d77000000000000000000000000000000000000000000000000000000000000000100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8
```

The application correctly identifies the collaborator account. The substituted field is the authority resource and scope.

## Verified browser-to-chain result

The final clean run used the frozen competition import `cda79acaad59711b943fc68207ebb3f1d0ff8596`, which imports upstream commit `63772fd872af472ced58b009499355f3430c2a86`.

It used the production Portal route, production exact-name Roles page, production `View` link, production resolver table and sidebar, production transaction modal and transaction manager, a connected local wallet, authentic Panoptes, a real fork-deployed PermissionedResolver, and real ENSjs record reads and writes.

| Item                                               | Verified result                                                      |
| -------------------------------------------------- | -------------------------------------------------------------------- |
| Environment                                        | Private disposable Sepolia fork at block `11577942`                  |
| Node / pnpm                                        | `v22.23.2` / `10.27.0`                                               |
| Initial collaborator role                          | Name-scoped `ROLE_SET_TEXT`, bitmap `16`                             |
| Initial complete ROOT role bitmap                  | `0`                                                                  |
| Initial root `ROLE_SET_ADDR`                       | `false`                                                              |
| Victim ETH record before exploitation              | victim administrator address                                         |
| Collaborator can alter victim name before approval | `false`, call rejected                                               |
| Authentic Panoptes nodes include both names        | `true`                                                               |
| Portal resolved names                              | `[]`                                                                 |
| Table Name cell                                    | blank                                                                |
| Named-scope label                                  | absent                                                               |
| `Global roles (all names)` warning                 | absent                                                               |
| Modal mentions ROOT                                | `false`                                                              |
| Browser-signed function                            | `grantRootRoles`                                                     |
| Browser-signed arguments                           | `1`, attacker address                                                |
| Browser root-grant transaction                     | `0x3e5f561e27ba0f1dc8aaedba2a33c0456c9e66ac52869609c6e63ff522cd1be0` |
| Final collaborator root `ROLE_SET_ADDR`            | `true`                                                               |
| Collaborator's victim-record transaction           | `0x1bb8f452466c8932b7042277b43b1eabf674f77ba9f67aa7ca4a4efc15021771` |
| Victim ETH record after exploitation               | attacker address                                                     |
| Portal victim Records page                         | rendered attacker address                                            |
| Browser result                                     | `1 passed`                                                           |

The final browser-signed transaction input exactly matched the decoded root call. The role event was emitted for resource zero and the intended collaborator account.

### Maximum-blast-radius Upgrade consequence

Portal exposes every resolver permission on the same unresolved row, including:

```
Upgrade
Can upgrade the resolver contract
```

The text discloses that Upgrade is dangerous, but it does not disclose the critical scope substitution: the existing row is name-scoped, while the final transaction is root-scoped.

The production sidebar and builder prepared:

```
grantRootRoles(
  21267647932558653966460912964485513216,
  attacker
)
```

That bitmap is `ROLE_UPGRADE = 1 << 124`.

The private-fork amplifier used one shared resolver with two victim names owned by different accounts and one collaborator name. It proved:

{% stepper %}
{% step %}

## The collaborator initially had only name-scoped `ROLE_SET_TEXT` and no root Upgrade role.

{% endstep %}

{% step %}

## Granting the exact same Upgrade bit at the collaborator-name resource did not satisfy the resolver's root-only UUPS guard.

`upgradeToAndCall` reverted and the implementation slot did not change.
{% endstep %}

{% step %}

## Portal's empty-scope path mined `grantRootRoles(1 << 124, attacker)` through the production transaction manager.

{% endstep %}

{% step %}

## The collaborator's subsequent `upgradeToAndCall` succeeded.

{% endstep %}

{% step %}

## The EIP-1967 implementation slot changed while the proxy address and proxy bytecode remained unchanged.

{% endstep %}

{% step %}

## Two names owned by different accounts then resolved through ENSjs to the attacker.

{% endstep %}
{% endstepper %}

Final amplifier evidence:

| Item                                            | Verified result                                                      |
| ----------------------------------------------- | -------------------------------------------------------------------- |
| Root Upgrade grant                              | `0xa2d4fada2d412eff00f7a463004da7e8a8ec5e43b5137592596522347590b74d` |
| Resolver upgrade transaction                    | `0xf75053ebb5625799758e37c9292b7f94623bba304f6ef32deeb851254fe28a7b` |
| Name-scoped Upgrade negative control            | reverted                                                             |
| Implementation unchanged after negative control | `true`                                                               |
| Root Upgrade after Portal grant                 | `true`                                                               |
| Implementation before                           | `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`                         |
| Implementation after                            | `0xD98394c99151E9F7479447A52B6c4ECC521eb467`                         |
| Proxy bytecode unchanged                        | `true`                                                               |
| First victim ENSjs ETH result                   | attacker                                                             |
| Second victim ENSjs ETH result                  | attacker                                                             |

The locally compiled implementation is only a deterministic consequence demonstrator. The submitted vulnerability is Portal converting an unresolved name-scoped role into a root grant.

## Authorization controls work but do not stop the exploit

* Portal requires the connected victim to be a legitimate resolver role administrator before enabling edits.
* The PermissionedResolver enforces the administrator's authority on-chain.
* The collaborator's attempted victim-record write before the Portal approval is rejected.
* A genuinely name-scoped Upgrade role cannot pass the resolver's root-only upgrade guard.
* The connected wallet signs the exact bytes the chain executes.

These controls authenticate the signer and enforce the submitted resource. They do not ensure that the submitted resource matches the name-scoped row the victim selected.

## Negative controls

* Both exact names are present in the authentic Panoptes response when the lossy role is fetched.
* The full name resource is nonzero and differs from ROOT.
* The Panoptes resource differs only in the cleared low 32 bits.
* The attacker starts with no root role.
* The attacker cannot modify the unrelated victim name before the victim interaction.
* Portal's normal root-scope warning is absent.
* The transaction modal contains neither the name nor ROOT.
* The mined function is `grantRootRoles`, not `authorizeNameRoles`.
* The natural Set Address consequence changes the unrelated victim name's ETH record and Portal renders the changed value.
* The Upgrade negative control proves name-scoped Upgrade alone cannot replace the implementation.
* Two differently owned names resolve to the attacker only after the unintended root Upgrade grant and implementation replacement.
* A revoke-path control independently prepares `revokeRootRoles(16, attacker)` for the unresolved name-scoped row, confirming the same scope collision without being needed for the Critical consequence.
* No RPC or indexer response is mocked or forged.

## Difference from known issue QA-01

QA-01 covers certain role combinations that later cause functional errors. This report does not rely on a later error, a dead permission combination, or an unavailable operation.

Portal creates a new ROOT role that the collaborator did not have. The connected-wallet grant succeeds, the collaborator immediately exercises the new authority against an unrelated victim name, and Portal renders the attacker-controlled address. The Upgrade amplifier separately proves a successful implementation replacement and cross-name resolution change.

The competition boundary explicitly states that a role combination which grants unintended authority is a new finding. This is a direct, verified instance of that exception.

## Preconditions and honest boundaries

* The victim administers a PermissionedResolver shared by more than one name.
* The attacker controls one name using that resolver and already has a benign name-scoped role on that name.
* The victim follows Portal's role-management UI, adds `Set Address` or another permission to the existing collaborator, and approves the connected-wallet transaction.
* The attacker has no root role before that approval.
* The victim knows which account they are editing. The substituted value is the role scope, not the account.
* The proof does not claim that the attacker can force the victim to open the row or approve the transaction.
* A wallet that independently decodes and clearly presents `grantRootRoles` can expose the substitution, and the victim can reject it. Portal's own review omits the scope.
* The proof does not claim name or NFT ownership transfer.
* The record-hijack proof does not claim that a public-network payment was completed. A sender must use the attacker address rendered for the victim name for funds to be misdirected.
* The Upgrade amplifier uses a minimal locally compiled implementation only on the private fork. No public resolver is upgraded.
* No stolen key, wallet compromise, malicious RPC, forged indexer response, backend compromise, HCA/session takeover, or pre-existing root attacker authority is required.

## Impact Details

This maps directly to the Critical malicious connected-wallet interaction impact. Portal presents an edit to an existing collaborator row without any root indication, but changes the authorization-bearing scope from a nonzero name resource to ROOT. The victim's wallet signs and broadcasts `grantRootRoles`, giving the collaborator authority over names that were outside the selected role.

The ordinary `Set Address` consequence makes the severity concrete. The collaborator uses the newly granted root permission to set an unrelated victim name's ETH record to the attacker address, and Portal's own Records page renders that value for the victim name. No additional victim-administrator interaction is needed after the unintended root grant.

From the attacker's perspective, an existing benign name-scoped collaborator role supplies the row. One victim-approved Portal edit escalates that account from text access on one name to address control across the resolver.

From the victim administrator's perspective, Portal begins from the collaborator's exact name context, loses that context at its own `View` handoff, shows neither a named nor root scope, and confirms only the collaborator address. The resulting transaction grants resource-zero authority.

From another name owner's perspective, no interaction is required. If their name uses the same resolver, its records can be changed by the collaborator after the root escalation. The Upgrade variant demonstrates the maximum shared-resolver blast radius: complete implementation control and attacker resolution for multiple separately owned names.

## Recommended Remediation

{% stepper %}
{% step %}

## Represent scope as a discriminated type

Use a type such as `{ type: 'root' }` or `{ type: 'name', name, resource }`. Never use an empty name string as both lookup failure and ROOT.
{% endstep %}

{% step %}

## Fail closed for unresolved nonzero resources

When a nonzero role resource cannot be resolved to exactly one name, disable View/Edit/Save/Remove and show the raw inconsistent resource.
{% endstep %}

{% step %}

## Preserve and validate the source resource

Preserve the original resource on every role row. Before opening the wallet, recompute the name resource and require exact equality with the source resource.
{% endstep %}

{% step %}

## Preserve name context during navigation

Preserve the selected name context when navigating from a name page to a shared resolver page.
{% endstep %}

{% step %}

## Display exact scope throughout the flow

Display the exact scope in the table, sidebar, transaction modal, and wallet-facing metadata. Root changes should explicitly say `Global roles (all names)`.
{% endstep %}

{% step %}

## Restrict root-only permissions

Do not expose root-only permissions such as Upgrade on a row unless the source resource is exactly ROOT.
{% endstep %}

{% step %}

## Retain complete resource hashes in Panoptes

Fix Panoptes' resolver-role resource representation so full 256-bit hashes are retained rather than registry-style low-bit canonicalization.
{% endstep %}

{% step %}

## Add regression coverage

Add regression coverage for unknown and lossy nonzero resources. Grant and revoke paths must refuse to prepare a transaction.
{% endstep %}
{% endstepper %}

## References

* [Portal fetches resolver nodes and role resources](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L60-L118)
* [Portal computes full name resources and builds an exact lookup map](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L129-L160)
* [An unresolved nonzero resource is silently omitted from resolvedNames](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220)
* [The resolver table renders a blank Name cell for unresolved scope](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L65-L83)
* [The exact-name Roles page labels the name but View drops to the shared resolver route](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/NameResolverRolesOverviewTable.tsx#L73-L102)
* [The sidebar reduces unresolved scope to an empty string](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L193)
* [Save snapshots the empty name into the pending action](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L295-L305)
* [Scope text is omitted when names are unresolved](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L389-L397)
* [The confirmation identifies only the account](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L338-L349)
* [The pending action is forwarded to the grant builder](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts#L48-L61)
* [The grant builder interprets the empty name as ROOT](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L18-L74)
* [The prepared request enters the production transaction manager](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L99-L118)
* [The role-management route requires legitimate root management authority](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/resolver/$address/roles/index.tsx#L30-L50)
* [Portal's Records page fetches and renders live profile records](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/records.tsx#L15-L70)
* [Upgrade is exposed in the generic resolver permission catalogue](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L93-L108)

## Proof of Concept

### Safety and environment

This proof runs entirely against a private disposable Anvil fork using standard throwaway development accounts. It does not send transactions to public Sepolia, Ethereum mainnet, or any other public network. It does not use production names, funds, wallets, sessions, credentials, or services.

The required Sepolia endpoint is used only by local Anvil to read the state needed to initialize the fork at block `11577942`. Every name registration, resolver attachment, role grant, record change, implementation upgrade, and wallet approval is mined only inside the local fork.

The runner uses dedicated loopback ports:

```
Anvil:        23545
Panoptes API: 23655
Portal:       23701
```

It requires an explicit fork endpoint, uses digest-pinned Anvil and Panoptes images, refuses occupied ports or pre-existing resources for its Compose project, uses a local run lock, and removes only the containers, network, volume, and generated evidence files it owns. It prints `servicesOwnedByRunnerStopped=true` only after all three listeners and every project-labeled resource are gone.

### PoC evidence bundle

Download the complete runnable overlay and captured evidence here:

[Download ens-portal-resolver-root-scope-poc.zip](https://drive.google.com/file/d/1jLRZIv0C8mPlLeINlF_A2FT7tpzUwJKx/view?usp=sharing)

Save it with this exact filename:

`ens-portal-resolver-root-scope-poc.zip`

ZIP SHA-256:

```
b52264c072099952b1af148bb8ed8653e2ebe41e1b90e009490d8bb8d02b5011  ens-portal-resolver-root-scope-poc.zip
```

The archive contains the standalone runner, pinned Compose and Panoptes configurations, production unit/component/live tests, production Playwright test, minimal private-fork upgrade implementation, fresh structured evidence, complete clean log, and two production Portal screenshots. `POC-SHA256SUMS.txt` authenticates every included file except the manifest itself.

### Prerequisites

* Linux with Bash.
* Node.js v22. The final verification used `v22.23.2`.
* Corepack and pnpm v10. The final verification used `10.27.0`.
* Docker with Docker Compose v2 and permission to start containers.
* Foundry `forge` and `cast`.
* Chromium at `/usr/bin/chromium`, or another executable supplied through `CHROMIUM_PATH`.
* Git, curl, jq, `flock`, `grep`, `sed`, `setsid`, `ss`, `sha256sum`, and `unzip`.
* A researcher-controlled Sepolia RPC URL with historical read and fork support for block `11577942`, reachable from the Docker container.
* Local ports `23545`, `23655`, and `23701` available. Alternate valid ports can be supplied through `RESOLVER_ROOT_UPGRADE_ANVIL_PORT`, `RESOLVER_ROOT_UPGRADE_INDEXER_PORT`, and `RESOLVER_ROOT_UPGRADE_PORTAL_PORT`.
* Network access for the initial repository clone, frozen dependency installation, pinned image download, and read-only fork initialization.

{% stepper %}
{% step %}

## Obtain the frozen competition repository

```bash
mkdir ens-portal-resolver-root-reproduction
cd ens-portal-resolver-root-reproduction

git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-ens.git \
  audit-comp-ens

cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596

test "$(git rev-parse HEAD)" = \
  "cda79acaad59711b943fc68207ebb3f1d0ff8596"

git show -s --format='%H%n%B' HEAD
```

The commit message identifies imported upstream commit:

```
63772fd872af472ced58b009499355f3430c2a86
```

{% endstep %}

{% step %}

## Install frozen dependencies

```bash
node --version
corepack pnpm --version
corepack pnpm install --frozen-lockfile
```

Expected major versions:

```
Node.js: v22.x
pnpm:    10.x
```

{% endstep %}

{% step %}

## Overlay and verify the PoC

Place `ens-portal-resolver-root-scope-poc.zip` inside the cloned repository root, then verify the outer checksum before extraction:

```bash
printf '%s  %s\n' \
  'b52264c072099952b1af148bb8ed8653e2ebe41e1b90e009490d8bb8d02b5011' \
  'ens-portal-resolver-root-scope-poc.zip' | sha256sum -c -
```

Expected output:

```
ens-portal-resolver-root-scope-poc.zip: OK
```

Extract and verify the internal manifest:

```bash
unzip ens-portal-resolver-root-scope-poc.zip

chmod +x \
  poc/resolver-root-upgrade/run-clean.sh \
  poc/resolver-root-upgrade/test-runner-preflight.sh

sha256sum -c POC-SHA256SUMS.txt
```

Every path in `POC-SHA256SUMS.txt` must print `OK`.

The relevant layout is:

```
audit-comp-ens/
|-- POC-README.md
|-- POC-SHA256SUMS.txt
|-- apps/portal/poc/
|   |-- resolver-root-scope.vite.config.ts
|   |-- resolver-unresolved-root-intent-live.test.ts
|   |-- resolver-unresolved-root-sidebar.test.tsx
|   `-- resolver-unresolved-root-upgrade-live.test.ts
|-- e2e/poc/
|   `-- portal-resolver-root-scope.playwright.config.ts
|-- e2e/projects/portal/tests/
|   `-- portal-resolver-root-scope.poc.spec.ts
|-- poc/resolver-root-upgrade/
|   |-- anvil.compose.yml
|   |-- foundry.toml
|   |-- panoptes.json
|   |-- run-clean.sh
|   |-- test-runner-preflight.sh
|   `-- src/AttackerResolverImplementation.sol
`-- scans/
    |-- portal-resolver-root-grant-modal.png
    |-- portal-resolver-root-hijacked-record.png
    |-- portal-resolver-root-scope-browser.json
    |-- portal-resolver-root-upgrade.json
    |-- resolver-root-upgrade-api.log
    |-- resolver-root-upgrade-indexer.log
    |-- resolver-root-upgrade-joined.log
    |-- resolver-root-upgrade-panoptes-runtime.json
    `-- resolver-root-upgrade-vite.log
```

{% endstep %}

{% step %}

## Run the fail-closed preflight

```bash
poc/resolver-root-upgrade/test-runner-preflight.sh
```

Expected result:

```
runnerPreflightTests=PASS
```

The preflight checks shell syntax, executable bits, missing-endpoint rejection, port validation, local proxy configuration, pinned container digests, fixed chain and block, and loopback-only host bindings.
{% endstep %}

{% step %}

## Supply a private-fork endpoint

Supply a researcher-controlled endpoint without placing it in source files or shell history:

```bash
read -rsp 'Researcher-controlled Sepolia RPC URL: ' \
  RESOLVER_ROOT_UPGRADE_SEPOLIA_FORK_URL
export RESOLVER_ROOT_UPGRADE_SEPOLIA_FORK_URL
printf '\n'
```

The runner exits before starting Docker if the variable is missing.

If Chromium is not at `/usr/bin/chromium`, also set:

```bash
export CHROMIUM_PATH='/absolute/path/to/chromium'
```

{% endstep %}

{% step %}

## Execute the joined proof

From the frozen repository root:

```bash
poc/resolver-root-upgrade/run-clean.sh
```

The complete terminal record is written to:

```
scans/resolver-root-upgrade-joined.log
```

The runner performs this sequence:

1. Verifies Node v22, pnpm v10, the exact competition commit, required commands, run lock, ports, Chromium, and absence of pre-existing project resources.
2. Compiles the minimal Upgrade consequence implementation from source.
3. Starts digest-pinned Anvil at Sepolia block `11577942` and verifies chain ID `11155111` plus an Anvil client identity.
4. Starts digest-pinned Panoptes against the isolated fork.
5. Runs production Portal live and component tests for the natural Set Address consequence.
6. Starts the frozen Portal app with only its development RPC and indexer proxies pointed at the isolated services.
7. Runs the full connected-wallet browser path from the collaborator's exact-name Roles page through Portal's `View` link, resolver editor, transaction modal, and wallet approval.
8. Decodes the exact mined browser transaction, exercises the granted authority against an unrelated victim name, and verifies Portal renders the attacker address.
9. Runs the root Upgrade amplifier last, including the name-scoped negative control, implementation-slot transition, and two-victim ENSjs resolution assertions.
10. Removes and verifies all local resources in the exit trap.
    {% endstep %}

{% step %}

## Observe the authentic indexer mismatch

The final browser evidence contains the full authentic pre-grant Panoptes response. It must show both names in `nodes` and a name-scoped collaborator role with `name: null`.

The decisive resource values are:

```
Full on-chain collaborator resource:
0x7b792a90b5b1287e5e7641cae209e052643c6be56042f94fefd86e4ee74d3c23

Panoptes collaborator resource:
0x7b792a90b5b1287e5e7641cae209e052643c6be56042f94fefd86e4e00000000
```

The values are not ROOT and are not equal. The Panoptes value preserves the upper 224 bits and clears the lower 32 bits.
{% endstep %}

{% step %}

## Observe the pre-exploit authority boundary

The natural-path tests require:

```json
{
  "attackerInitialNameScopedRoleBitmap": "16",
  "attackerInitialRootRoleBitmap": "0",
  "attackerInitialRootSetAddr": false,
  "attackerCouldSetVictimAddressBeforeGrant": false,
  "victimEthAddressBefore": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266"
}
```

The collaborator starts with only name-scoped `ROLE_SET_TEXT`. A direct attempt to set the unrelated victim name's ETH address is rejected.
{% endstep %}

{% step %}

## Observe the production Portal and wallet path

The Playwright test uses authentic Panoptes and the frozen production Portal. It:

1. Opens `/<collaborator-name>/roles`.
2. Requires the heading `<collaborator-name> resolver roles` and existing `Set Text` role.
3. Clicks Portal's own `View` link to `/resolver/<shared-resolver>/roles`.
4. Requires the collaborator's Name cell to be blank.
5. Opens the production editor and requires both `Roles scoped to ...` and `Global roles (all names)` to be absent.
6. Checks `Set Address`, clicks the visible enabled Save control, starts the production transaction modal, and verifies the modal identifies only the account.
7. Opens and approves the connected local wallet.
8. Locates the emitted resource-zero role event, fetches its exact transaction, and ABI-decodes the input.

The modal screenshot is:

```
scans/portal-resolver-root-grant-modal.png
```

The mined call must be:

```
grantRootRoles(
  1,
  0x70997970C51812dc3A010C7d01b50e0d17dc79C8
)
```

Exact browser transaction:

```
0x3e5f561e27ba0f1dc8aaedba2a33c0456c9e66ac52869609c6e63ff522cd1be0
```

Exact input:

```
0x072d5d77000000000000000000000000000000000000000000000000000000000000000100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8
```

{% endstep %}

{% step %}

## Observe the unrelated-name record hijack and Portal sink

After the root grant, the collaborator submits a normal resolver write against the unrelated victim name. The final transaction is:

```
0x1bb8f452466c8932b7042277b43b1eabf674f77ba9f67aa7ca4a4efc15021771
```

The assertions are:

```json
{
  "attackerFinalRootSetAddr": true,
  "victimEthAddressAfter": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
  "portalVictimRecordRenderedAsAttacker": true
}
```

The production Records-page screenshot is:

```
scans/portal-resolver-root-hijacked-record.png
```

{% endstep %}

{% step %}

## Observe the maximum Upgrade consequence

The Upgrade leg first grants the same `ROLE_UPGRADE` bit at the actual collaborator-name resource. It requires:

```json
{
  "nameScopedUpgradeRoleWasActuallyGranted": true,
  "nameScopedUpgradeReverted": true,
  "implementationUnchangedAfterNameScopedNegativeControl": true,
  "attackerInitialRootUpgrade": false
}
```

It then sends the unresolved empty scope through the production builder and transaction manager. The final grant decodes as:

```
grantRootRoles(
  21267647932558653966460912964485513216,
  0x70997970C51812dc3A010C7d01b50e0d17dc79C8
)
```

The final transactions are:

```
Root Upgrade grant: 0xa2d4fada2d412eff00f7a463004da7e8a8ec5e43b5137592596522347590b74d
Implementation upgrade: 0xf75053ebb5625799758e37c9292b7f94623bba304f6ef32deeb851254fe28a7b
```

The EIP-1967 implementation must change from:

```
0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
```

to:

```
0xD98394c99151E9F7479447A52B6c4ECC521eb467
```

Both separately owned victim names must resolve through ENSjs to the attacker, and `bothUnrelatedNamesResolveToAttacker` must be `true`.
{% endstep %}

{% step %}

## Observe final pass and cleanup markers

The proof must finish with all of these markers:

```
Test Files  2 passed (2)
Tests  4 passed (4)
setAddressAndComponentResult=PASS
1 passed
browserResult=PASS
Test Files  1 passed (1)
Tests  1 passed (1)
componentAndForkResult=PASS
joinedResult=PASS
servicesOwnedByRunnerStopped=true
```

The cleanup marker is printed only after no listener remains on the three isolated ports and no container, network, or volume remains under the runner's Compose project label.
{% endstep %}
{% endstepper %}

### Proof boundaries

* The victim must be a legitimate resolver role administrator and approve the normal connected-wallet transaction.
* A wallet that independently decodes and clearly presents `grantRootRoles` can expose the substitution, and the victim can reject it. Portal's own review omits the scope.
* The collaborator starts with one benign name-scoped role but no root authority.
* The primary proof demonstrates a wrong-scope root grant, unauthorized record control over an unrelated name, and Portal's attacker-address rendering. It does not claim that a public-chain payment was completed.
* The Upgrade leg is a maximum-blast-radius amplifier. The permission is visibly named `Upgrade`; the defect is that Portal silently changes its scope from the collaborator name to ROOT.
* The proof does not claim name/NFT ownership transfer, stolen keys, wallet compromise, HCA/session takeover, malicious RPC behavior, forged indexer data, or public-chain exploitation.
* Contracts and Panoptes supply the authentic execution environment. The submitted application defect is Portal treating an unresolved nonzero resource as its ROOT sentinel and building a different authorization scope than the selected row.
