# 93080 w a critical improper resolver role grouping causes incomplete revocation and attacker controlled ens resolution

## #93080 \[W\&A-Critical] Improper resolver-role grouping causes incomplete revocation and attacker-controlled ENS resolution

**Submitted on Sep 14th 2026 at 07:26:16 UTC by @zealynx for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93080
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

### Brief/Intro

The ENS Portal groups all resolver-role assignments for one account into a single actionable row even when they belong to different Enhanced Access Control (EAC) resources. When the resolver owner selects **Remove user**, Portal submits one revocation against only the first non-root name while presenting the action as removal from all roles. Assignments on every unselected resource remain effective, so a malicious or compromised delegate whose surviving assignment includes `ROLE_SET_ADDR` can change the unaffected name's ETH address to an attacker-controlled address after the removal transaction succeeds.

### Vulnerability Details

The resolver overview query retains each assignment's `account`, `resource`, and `roleBitmap`. However, [`groupRolesByAccount`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220) keys its map only by the lower-cased account:

```ts
const account = role.account.toLowerCase()
const resource = role.resource.toLowerCase()
const decoded = decodeResolverRoleBitmap(role.roleBitmap)
const resolvedName = resourceToName?.get(resource) ?? null

const existing = grouped.get(account)
```

Assignments from root and different names are appended to the same group, and their decoded role names are combined. [`ResolverRolesTable`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L90-L137) renders that account group as one row with one set of actions.

[`ResolverRolesSidebar`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L172) then reduces the group's resources to one write target:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The remove handler passes this one `roleName` together with the group's combined `decodedRoles`. [`revokeResolverRoles`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L45-L67) encodes exactly one call: `revokeRootRoles(bitmap, account)` when the name is empty, or `authorizeNameRoles(name, bitmap, account, false)` otherwise. Input ordering determines which non-root resource is selected; no call is prepared for the other resources.

The table lists the row's known scopes, but a root-containing row is summarized in the sidebar as `Global roles (all names)`. The confirmation asks Alice whether she wants to “remove this user from all roles,” without explaining that the encoded transaction covers only one resource. The mined call is correctly treated as successful because the submitted call succeeds. Resolver state is subsequently polled and refetched, so a surviving assignment can reappear after indexer synchronization, but the flow does not require every requested `(account, resource, role)` bit to be zero before treating the transaction as complete.

The resolver contracts correctly enforce the scope Portal supplies. At the pinned supporting revision, [`authorizeNameRoles`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/resolver/PermissionedResolver.sol#L266-L293) updates only the resource computed for its supplied name. [`_effectiveRoles`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/access-control/EnhancedAccessControl.sol#L452-L455) combines root and resource-local roles, and [`setAddr`](https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/resolver/PermissionedResolver.sol#L660-L682) permits an address write while `ROLE_SET_ADDR` remains effective. The resource identity is lost in Portal before the typed contract call is selected.

The demonstrated exploit proceeds as follows:

1. Alice grants Mallory `ROLE_SET_TEXT` on `alpha.eth` and `ROLE_SET_ADDR` on `beta.eth` through the same permissioned resolver. Mallory has no root role.
2. Portal displays one row for Mallory containing both names and both permissions.
3. Alice selects **Remove user** and confirms removal from all roles.
4. Assuming `alpha.eth` is the first indexed name, Portal submits `authorizeNameRoles(alpha.eth, ROLE_SET_TEXT | ROLE_SET_ADDR, mallory, false)`.
5. The call clears the selected `alpha.eth` resource only. Mallory's `ROLE_SET_ADDR` on `beta.eth` remains effective.
6. Before Alice observes the surviving assignment and sends the missing revocation, Mallory changes `beta.eth`'s ETH address to an address she controls.
7. A user who resolves `beta.eth` receives Mallory's address and can send funds there.

The same defect has a broader variant: if Mallory holds root `ROLE_SET_ADDR` plus any name-scoped role, Portal selects the name and leaves the root assignment intact, preserving address-changing authority over every name served by that resolver.

#### Recommended remediation

Portal must preserve the exact EAC resource and its scope type from the resolver overview response through display and transaction construction. Each displayed assignment should map to one explicit typed target such as `root`, `name`, or record `part`; transaction builders must not infer the target from the first entry in `resolvedNames`.

The clearest fix is to key actionable rows by `(account, resource)`. If the account-level row is retained, **Remove user** must expand into one correctly typed revocation per underlying resource and execute the operations atomically where supported. An unmapped resource must block the mutation with an explicit error instead of falling back to another scope. After synchronization, Portal should read back every requested `(account, resource, role)` and present the removal as complete only when all targeted bits are zero.

Regression tests should cover two names in both indexer orderings, root plus name, root-only and name-only controls, duplicate role bits, unmapped and part-scoped resources, partial batch failure, and both Remove and Save operations.

### Impact Details

This reaches the competition's listed Critical impact: **“Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient,”** specifically the example of resolving an ENS name to an attacker-controlled address.

The Foundry PoC demonstrates the complete downstream effect: Mallory's `ROLE_SET_ADDR` survives the Portal-style revocation, Mallory changes `beta.eth`'s resolver record to her address, `resolver.addr(betaNode)` returns that address, and ETH sent to the resolved address is received by Mallory.

This is a broken deauthorization, not a new privilege grant. Mallory must already hold `ROLE_SET_ADDR` on one of at least two resources associated with the same resolver, and Alice must use Portal's account-level removal action. The incomplete removal is deterministic, but harmful exploitation requires Mallory to act before Alice notices the surviving assignment and submits a correctly scoped revocation. The surviving row can become visible after indexer synchronization and Alice can recover by revoking the remaining resource, so the issue does not permanently remove her control. It does not require compromise of Alice's wallet, the indexer, or the resolver contracts.

### References

* [Portal target at the frozen competition revision](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal)
* [Resolver overview query retains `resource`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L60-L118)
* [Account-only grouping](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220)
* [Single-name selection and mutation path](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L322)
* [Scope summary and removal confirmation](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L391-L447)
* [Pinned supporting resolver revision](https://github.com/ensdomains/contracts-v2/tree/8d1c89350729f87b3968cf41ef3d683951444a44)
* [ENS Audit Competition information and impacts](https://immunefi.com/audit-competition/audit-competition-ens/information/)

### Link to Proof of Concept

<https://gist.github.com/vendrell46/c741eb3bac36fe4235596b65804e8393>

### Proof of Concept

## Proof of Concept

The secret Gist contains these three files:

* `README.md` — setup and execution instructions;
* `resolverRoleRemovalScope.poc.test.ts` — Portal grouping and calldata test;
* `RoleRemovalScope.t.sol` — resolver authorization, address-change, fund-flow, and correct-revocation controls.

**Secret Gist:** <https://gist.github.com/vendrell46/c741eb3bac36fe4235596b65804e8393>

The following steps reproduce the issue without interacting with a live ENS deployment.

I ran the two parts independently: the Portal test invokes the production grouping and calldata-building functions, while the Foundry test executes the equivalent transaction against the pinned resolver contracts. Together they establish the Portal-generated calldata and its contract-level consequence; they do not claim a live browser, indexer, and deployed-resolver end-to-end run.

1. Clone the frozen Portal target and install its locked dependencies:

   ```bash
   git clone https://github.com/immunefi-team/audit-comp-ens.git app
   git -C app checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
   pnpm --dir app install --frozen-lockfile
   ```
2. Clone the secret Gist and place the Portal test in the target tree:

   ```bash
   git clone https://gist.github.com/vendrell46/c741eb3bac36fe4235596b65804e8393 ens-role-resource-poc
   cp ens-role-resource-poc/resolverRoleRemovalScope.poc.test.ts \
     app/apps/portal/src/lib/roles/resolverRoleRemovalScope.poc.test.ts
   ```
3. Run the Portal PoC:

   ```bash
   pnpm --dir app --filter portal exec vitest run \
     src/lib/roles/resolverRoleRemovalScope.poc.test.ts
   ```

   The observed result is `1` passing test file containing `5` passing tests. The multi-resource cases produce one account group and one name-scoped `authorizeNameRoles` call, while the root-only and name-only controls select their correct typed operations.
4. Clone the resolver contracts at the revision used to validate the consequence, then copy the Foundry test:

   ```bash
   git clone --recurse-submodules https://github.com/ensdomains/contracts-v2.git contracts-v2
   git -C contracts-v2 checkout 8d1c89350729f87b3968cf41ef3d683951444a44
   git -C contracts-v2 submodule update --init --recursive
   cp ens-role-resource-poc/RoleRemovalScope.t.sol \
     contracts-v2/contracts/test/RoleRemovalScope.t.sol
   ```
5. Run the contract PoC from the contracts workspace:

   ```bash
   cd contracts-v2/contracts
   forge clean
   forge test --match-path test/RoleRemovalScope.t.sol -vv
   ```

   The observed result is `2 passed; 0 failed; 0 skipped`. The two-name test shows that the unselected `ROLE_SET_ADDR` assignment survives, Mallory changes the resolved address, and funds sent to that resolution reach Mallory. The root-plus-name test shows that the same Portal-style name revocation leaves root address-changing authority effective. The correct-revocation control then clears the actual surviving scope and confirms Mallory's next address write reverts.

The tests operate only on local in-memory state and disposable local contracts. They require no network credentials, private keys, or cleanup beyond deleting the two cloned directories.
