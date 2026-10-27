# 92622 w a critical an unmappable name scoped resolver role becomes a resolver wide root grant signed by the victim

**Submitted on Sep 11th 2026 at 02:00:19 UTC by @Ailenia for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92622
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Portal keeps an old resolver role editable after its name is repointed, but reconstructs the role's scope only from the old resolver's current node list. When that lookup fails, `ResolverRolesSidebar` converts the missing name to `''`, and the production grant builder defines `''` as root scope. A victim who voluntarily adds one permission to the still-visible delegate row is shown no name or root warning but signs `grantRootRoles` on the old resolver. This is accepted impact 6288 because attacker-controlled ordinary ENS state modifies the authorization resource and entry point of a transaction requested from an already-connected victim wallet. The new root authority lets the attacker rewrite the address record of `victim.eth`, although the attacker previously had authority only for `child.victim.eth`.

## Vulnerability Details

Assume `victim.eth` uses PermissionedResolver `R1`. The victim delegates `child.victim.eth` to the attacker and gives the attacker a limited name-scoped resolver role on `R1`, such as `ROLE_SET_TEXT`. The child owner has the ordinary registry `ROLE_SET_RESOLVER`, so the attacker can repoint only `child.victim.eth` from `R1` to `R2`. Registry resolver pointers and resolver EAC roles are independent: this removes the child from `R1.nodes`, but it neither revokes nor moves the child's role stored on `R1`.

My PoC sends this sequence through a complete local Anvil -> official contracts -> digest-pinned Panoptes -> GraphQL pipeline. It waits for Panoptes to ingest both names and the attacker's real child-scoped role, broadcasts the attacker's resolver change, waits for the new block to be indexed, and queries the fields consumed by `useResolverOverview`:

```
PANOPTES BEFORE nodeCount=2 nodes=["child.victim.eth","victim.eth"] roleBitmap=0x10
PANOPTES AFTER  nodeCount=1 nodes=["victim.eth"] sameRole=true
```

The role remains byte-for-byte unchanged, but `buildResourceToNameMap()` only maps root and names still present in `nodes`. `groupRolesByAccount()` preserves the role and decoded bitmap even when the map lookup fails, producing one row with `roles.length === 1`, `decodedRoles === ['ROLE_SET_TEXT']`, and `resolvedNames === []`.

The row remains editable. Its Name cell is blank, and the sidebar suppresses the scope paragraph because `resolvedNames.length` is zero. The victim sees the delegate's account and permissions, enables the Address manager checkbox, and clicks Save in the ordinary resolver-role workflow. The transaction modal says `Save resolver role changes` and `Update roles for <attacker>`; it presents neither a name nor root scope before requesting wallet approval.

The scope substitution occurs here:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

That empty string is frozen in `pendingAction.name` and passed unchanged to the grant helper. The helper explicitly interprets it as resolver root:

```ts
name === ''
  ? { resolverAddress, targetAccount: account, scope: 'root', roles }
  : { resolverAddress, targetAccount: account, scope: 'name', name, roles }
```

Adding only `ROLE_SET_ADDR` produces this exact request shape:

```
from:    victim
to:      R1
chainId: 11155111
value:   0
call:    grantRootRoles(1, attacker)
```

The correctly associated control keeps `child.victim.eth` in `R1.nodes` and supplies its authoritative full resource. With the same role, account, resolver, permission, chain, and sender, the production builder instead emits `authorizeNameRoles(dnsEncode('child.victim.eth'), 1, attacker, true)`. As a conservative control, the PoC replaces only Panoptes's resource field with that exact on-chain resource and reruns Portal grouping. It remains unmappable after the child leaves `R1.nodes`, and this corrected-resource path is used for the executed grant, proving indexer serialization is not the carrier. The missing lookup alone changes name authority into root authority.

## Impact Details

The demonstrated impact is project-authored Critical impact 6288: "Malicious interactions with an already-connected wallet" including modifying transaction arguments or parameters. The application shows a blank scope while preparing a resolver-wide authorization transaction from the victim. This is an authorization-resource substitution in the signed transaction itself, not an otherwise correct transaction that merely leaves authority behind. The only authorization represented by the row is the attacker's existing child resource, but the missing resource-to-name lookup changes both the expected resource and entry point from `authorizeNameRoles(child.victim.eth, ...)` to `grantRootRoles(...)`. The missing lookup alone therefore widens the signed permission from one child to `ROOT_RESOURCE` for every name on `R1`.

The official PermissionedRegistry and PermissionedResolver PoC proves that the attacker owns the delegated child, can repoint it through `ROLE_SET_RESOLVER`, and leaves the old resolver role unchanged. Its negative control proves that a child-scoped `ROLE_SET_ADDR` lets the attacker edit the child but rejects the identical write to the parent. It then executes the exact `grantRootRoles(1, attacker)` call from the victim, verifies `hasRootRoles(ROLE_SET_ADDR, attacker)`, and proves the attacker can set the parent `victim.eth` address to an attacker-controlled target. Any other name using `R1` is within the newly granted resolver-wide scope.

The attacker unilaterally creates and observes the prerequisite by repointing the child they own and waiting for the public state change to be indexed. The victim then performs the normal prompted role-management action: add one permission, click Save, and approve the request. The beneficiary had previously been trusted only on the child, but the application creates new resolver-wide authority that was never displayed or granted. Once the request executes, the attacker can rewrite the parent and every other name served by `R1`.

## Scope Position

The programme lists `apps/portal` as the in-scope Explorer application and publishes impact 6288 for malicious connected-wallet interactions that modify transaction arguments or parameters. The filed root cause is entirely in Portal's scope reconstruction and transaction construction. PermissionedResolver and Panoptes are not alleged to be vulnerable; they are local execution and reachability context.

## Existing Safeguards and Counterarguments

The final wallet request is the vulnerable artifact, not an earlier quote or simulation: the PoC decodes and executes the exact request submitted by production Portal transport. Portal labels it as an update to the displayed delegate but replaces the missing child resource with the root entry point. Contract authorization cannot stop the call because the victim legitimately administers the resolver, and neither the contract nor the wallet can reconstruct the child resource that Portal discarded. The negative control proves the attacker lacks parent authority until this exact request executes. The PoC explicitly reruns the exploit with the exact on-chain child resource, so correcting Panoptes's current resource serialization would not fix Portal's fail-open root fallback.

## Duplicate Check

None of the 59 Known Issues describes an unresolved role resource becoming the empty-string root sentinel; QA-01 expressly says a demonstrated role combination granting unintended authority is a new finding. Programme feedback characterized report `#89253` as account-level aggregation across multiple names causing an incomplete one-name revoke. This finding needs only one role and one resource, creates new root authority through `grantRootRoles`, and does not use the removal path. Splitting rows by resource fixes `#89253` but not this lookup failure; rejecting unresolved scope fixes this finding but not cross-resource aggregation.

## Remediation

Treat an unresolved role resource as an error, never as the existing empty-string root sentinel. Preserve the raw `resource` identity through the row and pending action, and require an explicit tagged scope such as `{ scope: 'root' }` or `{ scope: 'name', name }`; do not infer root from a missing name. If the indexer supplies a trustworthy name for the role, verify that `computeNameResource(name) === role.resource` before building a name-scoped transaction. Otherwise disable editing and ask the user to refresh or resolve the resource. The confirmation screen should always state `Root (all names)` or the exact name and should decode/assert that the final calldata matches that displayed scope.

## References

* Scope reconstruction: `../ens/repos/audit-comp-ens/apps/portal/src/lib/roles/resolverRoles.ts:129-220`
* Blank but editable row: `../ens/repos/audit-comp-ens/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:65-70` and `:102-116`
* Missing-name fallback, hidden scope, pending action, and submission: `../ens/repos/audit-comp-ens/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-169`, `:295-304`, `:338-349`, `:389-430`, and `:467-501`
* Empty string to root transaction: `../ens/repos/audit-comp-ens/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:18-74`
* Resolver overview query: `../ens/repos/audit-comp-ens/apps/portal/src/features/resolver/hooks/useResolverOverview.ts:14-22` and `:68-110`
* Ordinary subname-owner roles and resolver change builders: `../ens/repos/audit-comp-ens/apps/portal/src/features/registry/utils/create-subname.helpers.ts:15-27` and `:58-83`; `../ens/repos/audit-comp-ens/apps/portal/src/features/resolver/helpers/changeResolver.ts:42-65`

## Link to Proof of Concept

<https://gist.github.com/niafreu/6c9f2ee1f4f38b0379d621b9e1444952>

## Proof of Concept

## Proof of Concept

PoC at <https://gist.github.com/niafreu/6c9f2ee1f4f38b0379d621b9e1444952>

The attached four-file bundle uses the production Portal UI, transaction builder, and EOA transport; the digest-pinned Panoptes indexer and API; and the official ENS v2 `PermissionedRegistry` and `PermissionedResolver`. During execution, all chain and indexer traffic targets loopback. Setup fetches the two pinned repositories, locked dependencies, and pinned container image; the PoC never connects to an ENS deployment, public RPC, or third-party application.

Prerequisites are Git, Node.js, pnpm 10.27.0, Foundry (`forge` and `anvil`), and Docker.

Set up the pinned code:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
git -C audit-comp-ens checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
(
  cd audit-comp-ens
  corepack pnpm install --frozen-lockfile
)

git clone --recurse-submodules https://github.com/ensdomains/contracts-v2.git
git -C contracts-v2 checkout 48b3e2d39513b9dd32ef1850877a29009bc807b9
git -C contracts-v2 submodule update --init --recursive

git clone https://gist.github.com/niafreu/6c9f2ee1f4f38b0379d621b9e1444952.git poc
```

From the directory containing `audit-comp-ens`, `contracts-v2`, and `poc`, run:

```bash
AUDIT_REPO_DIR="$PWD/audit-comp-ens" \
CONTRACTS_V2_DIR="$PWD/contracts-v2" \
audit-comp-ens/apps/portal/node_modules/.bin/vitest run \
  --disableConsoleIntercept --reporter=verbose --config poc/vitest.config.ts
```

Expected: Vitest reports `2 passed (2)`.

* `unmappable-root-grant.ui.poc.test.tsx` renders the real sidebar with an unmappable role. It asserts the absence of name/root scope, performs the ordinary Address-manager edit, and decodes the resulting request as `grantRootRoles(1, attacker)`.
* `unmappable-root-grant.poc.test.ts` deploys the official contracts to local Anvil, launches the pinned Panoptes indexer and API, and waits for real ingestion before and after the attacker-owned child is repointed. It proves both the actual role and the exact on-chain resource are unmappable against the post-repoint GraphQL node list, then submits Portal's resulting request through production EOA transport. The negative control proves the attacker cannot edit the parent before the grant; the final assertion proves the attacker rewrites the parent afterward.

### Evidence Coverage

| Evidence                                                                                                                               | Treatment                                                                                                                                                                                                     |
| -------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Child ownership, resolver repoint, surviving child role, rejected parent write, Portal-built root grant, and successful parent rewrite | Directly executed against official contracts on local Anvil                                                                                                                                                   |
| Blank scope, permission edit, modal labels, and final `grantRootRoles(1, attacker)` calldata                                           | Directly executed through the production Portal component and transaction builder                                                                                                                             |
| Panoptes `nodes`/role response before and after the resolver change                                                                    | Directly indexed from genuine local contract events and queried through the digest-pinned GraphQL API; the exploit is separately rerun with the exact on-chain resource to exclude serialization as its cause |
| ENS deployment state                                                                                                                   | Not assumed or modified; the PoC is local-only as required by the competition rules                                                                                                                           |

The contract-backed test automatically pulls the digest-pinned Panoptes image if absent, starts the indexer and API, and removes both containers on completion. Its output includes:

```
PANOPTES BEFORE nodeCount=2 nodes=["child.victim.eth","victim.eth"] ...
PANOPTES AFTER nodeCount=1 nodes=["victim.eth"] sameRole=true
"submittedFunction": "grantRootRoles"
"rootRoleCreated": true
"canonicalResourceStillUnmappable": true
"parentAddressAfterExploit": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8"
```

Thank you for reading my submission!
