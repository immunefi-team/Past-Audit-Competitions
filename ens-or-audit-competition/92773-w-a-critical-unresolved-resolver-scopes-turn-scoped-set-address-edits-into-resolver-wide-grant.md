# 92773 w a critical unresolved resolver scopes turn scoped set address edits into resolver wide grants

**Submitted on Sep 12th 2026 at 06:26:20 UTC by @xylz for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92773
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

When an administrator adds Set Address to an existing scoped resolver delegate, the Explorer can ask the connected wallet to grant that permission at the resolver root. The account and selected permission remain correct, but the application replaces the unresolved resource scope with root without displaying the change. The local proof starts with a delegate holding only a name-scoped Text permission and zero root authority. Before the affected edit, the delegate's attempt to change an unrelated name's address reverts; after the Explorer-generated transaction, the same write succeeds.

The delegate can trigger the ordinary name-scoped case by pointing 101 names it controls at the shared resolver. The Explorer's unpaginated node relation then omits the original name. Separately, the query omits the metadata needed to identify fine-grained record resources, producing the same fallback even when the name is present. The affected row remains editable, and the production pre-start transaction modal does not disclose root scope.

The direct impact is an application-generated wallet request with a wider authorization scope than the existing delegation. Local resolution and transfer tests demonstrate downstream loss when another user relies on the changed address and approves a transfer. A separate conditional Upgrade consequence demonstrates persistent resolver-administration takeover. For a supported name whose resolver pointer was already locked, the owner and a later token recipient cannot recover through the tested normal repair paths, and paid renewal can preserve the compromised registration.

## Vulnerability Details

The tests use application revision `cda79acaad59711b943fc68207ebb3f1d0ff8596`, which pins upstream ENS Apps revision `63772fd872af472ced58b009499355f3430c2a86`, and ENS v2 contracts revision `48b3e2d39513b9dd32ef1850877a29009bc807b9`.

### 1. Resolver scope is lost before the sidebar renders

`useResolverOverview.ts:68-108` requests `nodes` and `roles` without pagination and does not request the indexer's `namedResources` relation:

```graphql
resolver(id: $id) {
  nodeCount
  nodes { id name owner { id } resolver { id address } }
  roles { account resource roleBitmap ... }
}
```

`resolverRoles.ts:128-160` (`buildResourceToNameMap`) creates mappings only for the root and `resource(namehash(name), bytes32(0))`. This can map an ordinary whole-name resource. It cannot map the contract's nonzero record-part resources for a Text key or Address coin type.

`resolverRoles.ts:179-220` (`groupRolesByAccount`) keeps the role bitmap when lookup fails but omits the missing scope from `resolvedNames`:

```ts
const resolvedName = resourceToName?.get(resource) ?? null
...
existing.decodedRoles.push(...decoded)
if (resolvedName) existing.resolvedNames.add(resolvedName)
```

The resulting account row still contains editable permissions, but `resolvedNames` can be empty. The original `roles` array retains each resource and its bitmap; the sidebar's mutation logic uses the aggregate permissions and display-derived name instead of that retained pairing.

### 2. An empty display scope is converted to root calldata

`ResolverRolesSidebar.tsx:165-193` derives the only scope used for mutations as follows:

```ts
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The sidebar renders a scope label only when `resolvedNames.length > 0` (`ResolverRolesSidebar.tsx:389-397`). There is no “unknown scope” warning and the controls remain enabled.

Save and Remove both copy `roleName` into the pending action (`ResolverRolesSidebar.tsx:295-321`). `grantResolverRoles.ts:29-74` (`prepareGrantResolverRolesTransaction`) and `revokeResolverRoles.ts:29-74` (`prepareRevokeResolverRolesTransaction`) both interpret an empty name as `scope: 'root'`.

That changes the transaction itself:

```
Expected for a name-scoped row:
authorizeNameRoles(dnsEncodedName, ROLE_SET_ADDR, delegate, true)

Prepared by the Explorer when that scope is unresolved:
grantRootRoles(ROLE_SET_ADDR, delegate)
```

The administrator selects a permission in a row whose existing role is narrow. The affected UI offers no root-scope selection and shows no scope at all. The sidebar supplies the modal title `Save resolver role changes` and transaction label `Update roles for <account>` (`ResolverRolesSidebar.tsx:339-349`).

A focused component run uses the production `TransactionModal`, rather than the earlier intent-capture substitute. While its estimate path prepares and decodes `grantRootRoles(ROLE_SET_ADDR, delegate)`, the overview renders only `Save resolver role changes`, status, and estimated cost. It does not render the account or any resource scope. Opening the modal's optional information view adds `Update roles for <account>` but still shows no root, global, all-names, or other scope text. The grant helper separately records an internal transaction-manager description containing "(root)"; that description is not rendered in either pre-start modal view. This test covers the application boundary, not an extension-wallet recording. A wallet may separately decode the method; the proof does not assume that wallets hide raw calldata.

### 3. Attacker-controlled 100-node cutoff

The public staging GraphQL schema exposes `resolver.nodes(first: Int)`. A read-only query against the competition endpoint found a resolver with `nodeCount = 159165`; the exact query used by the Explorer returned 100 nodes, `first: 101` returned 101, and the relation was ordered newest first. The first name outside the Explorer window was `ggojvwwui.eth`.

The local proof starts with an ordinary Portal-built name grant:

```
authorizeNameRoles(profile.eth, ROLE_SET_TEXT, delegate, true)
```

The delegate has no root roles. Using an attacker-controlled parent and the Portal's supported UserRegistry deployment and attachment helpers, the delegate creates 101 children that all point at the shared PermissionedResolver. A batching contract registers the 101 nodes in one transaction using 9,141,803 gas in the fresh local run. This needs ownership of the attacker's own names or subregistry, but no administrative role over the target resolver.

The public query establishes the indexer's newest-100 response. After mining the 101-node setup, the local proof models that verified response as the newest 100 names while retaining the original whole-name role. Its name is absent from the modeled `nodes` response, so the real grouping function yields `resolvedNames = []`. The administrator adds Set Address to the displayed delegate. The intended helper call decodes as `authorizeNameRoles`; the Explorer's empty-scope call decodes and executes as `grantRootRoles`. The local EVM test does not run an indexer; the separate read-only query and component test establish the production response shape.

The delegate then changes the address of unrelated `beta.eth`. The deployed `UniversalResolverV2` returns the delegate's address, and the Explorer's production `transferToken` helper sends the local ERC-1155 token for `valuable.eth` to that resolved address. The recipient ends with one token.

This branch matters because the missing-scope state is not limited to unusual fine-grained roles and does not require control of the resolver administrator. The attacker controls the data-volume trigger. The administrator still has to perform and sign a normal role edit.

### 4. Fine-grained resource metadata is structurally omitted

The second trigger does not depend on relation length. A delegate receives `ROLE_SET_TEXT` for one text key on `dave.eth`, with zero root roles. Even when `dave.eth` is present in the node list, `buildResourceToNameMap` cannot reconstruct the text-key resource, so the Explorer produces the same blank-scope row.

The administrator checks Set Address / Manager in that row. The production helper generates `grantRootRoles(ROLE_SET_ADDR, delegate)`, and the local wallet mines that exact target and calldata. The delegate gains root Set Address authority and changes `beta.eth`, which was outside the original text-key grant. Standard resolution returns the delegate.

A recorded read-only census of the competition indexer enumerated 173,490 resolver rows across 6,264 resolver addresses. It found 11,902 active role rows. Two hundred and four active rows matched the indexer's `namedResources` relation. Across all active rows, 187 could not be mapped by the audited Explorer logic; 179 of those unmapped rows had matching `namedResources` metadata that the query does not request. Sixty-one account groups reached the blank-scope/root fallback. As census-only corroboration, saved read-only contract simulations show 60 of those groups with a then-indexed administrator able to grant root Set Address and 48 able to grant root Upgrade. These are `eth_call` simulations against the recorded state, not public-chain writes or evidence that any administrator would make the edit.

The recorded component run selected both a text-key row and an ETH-address row from the endpoint, rendered the empty scope, and decoded the Explorer's root-scoped Save and Remove intents. These saved measurements document the competition's Sepolia deployment at collection time. They are not presented as mainnet prevalence or evidence that any public account was exploited.

The seven-case component suite exercises the production role table, sidebar, and intent builders with mocked wallet and transaction-modal dependencies. The separate modal test renders the production `TransactionModal`; it mocks wallet-client and gas-estimation plumbing so it can inspect the prepared request before submission. The EVM tests independently execute the resulting helper-generated transactions on a local chain. They compose resolution with the production transfer helper; they do not exercise a complete browser-and-extension-wallet transfer flow. The fine-grained proof uses a minimal registry tree around the production `UniversalResolverV2`, while the pagination proof deploys the production registry contracts.

### 5. The same scope substitution can grant resolver code control

Upgrade is one of the Manager permissions rendered by the role editor. Its description says `Can upgrade the resolver contract`, so the administrator knowingly selects a powerful permission. The missing fact is its scope: the affected row shows neither a name nor `Global roles (all names)`, and the production transaction modal does not add root, global, or all-names text. The prepared request is:

```
grantRootRoles(ROLE_UPGRADE, delegate)
```

This distinction is functional. `PermissionedResolver._authorizeUpgrade` accepts Upgrade only through the root role. In the local non-root control, the same selected bit is encoded through `authorizeNameRoles`; the delegate retains zero root roles and its upgrade attempt reverts. On the affected row, the empty-scope fallback creates usable root Upgrade authority.

The clean EVM composition begins with one Text-key role on `profile.eth`, zero root roles, and the exact name present in the node map. The part resource remains unresolved because the Explorer does not fetch and map the record-resource metadata. After the Explorer-built root grant:

* A predeployed contract recipient uses one transaction in the next block to move the original owner's complete root/admin bitmap to itself, restore the official resolver implementation, and set the address of unrelated `treasury.eth`. The original owner's root bitmap is zero, the recipient holds the fixture's complete bitmap, and the owner's revoke transaction reverts.
* In an independent ordinary-EOA branch, a predeployed compatible implementation is installed in the first post-grant attacker transaction. It returns the EOA for every address lookup. The original owner's attempt to restore the official implementation reverts through the installed implementation's upgrade guard.

These branches use expected UUPS and EAC behavior to measure the consequence of the Explorer's wrong-scope grant; they do not claim a separate contract defect. The first branch proves persistent resolver-administration takeover even after official code is restored. The second proves that a smart-contract recipient is not required. Neither transfers ENS registry ownership. A name owner who retains separate registry Set Resolver authority can detach the compromised resolver.

### 6. A supported locked-name configuration removes the owner's resolver escape route

The previous paragraph states the ordinary recovery path. A separate real-contract composition proves that it is unavailable for a supported migrated name whose owner has already locked its resolver pointer.

The setup uses a real `PermissionedResolver`, a wrapped ENSv1 `.eth` name, and the owner-authorized `CANNOT_UNWRAP | CANNOT_SET_RESOLVER` fuse state. Manager's locked migration deliberately preserves that resolver. The resulting ENSv2 token remains owned and transferable, but the owner receives neither registry `ROLE_SET_RESOLVER` nor its admin role. That owner-controlled lock was established while the resolver was still trusted; it does not authorize a narrow resolver delegate to obtain root authority later.

After the wrong-scope root Upgrade grant, two focused branches execute the consequence against that retained resolver:

* One branch installs compatible locked resolver code. Direct, extended, and Universal Resolution return the attacker. The owner cannot restore the implementation, repoint the registry, regrant itself the missing registry role, or repair the record. Transfer succeeds, but the recipient inherits the same inability to repoint.
* The second branch temporarily installs a storage-compatible implementation, moves the resolver's complete root/admin bitmap to the attacker, poisons the record, and restores the official implementation. Factory verification again identifies the official implementation, while the former owner has zero root roles and cannot revoke, repair, upgrade, or repoint.

Both fresh tests start from a 3,650-day registration and report 320,716,801 seconds remaining after migration. A third test uses the real `ETHRegistrar.renew` entry point one second before expiry. An unrelated funded payer, with no name ownership or token renewal role, pays to extend the same registration by 365 days. Ownership, resolver pointer, missing owner authority, and attacker-controlled Universal Resolution remain unchanged.

This is persistent loss of owner-controlled resolution for the maintained registration, not a claim that every affected name is permanently inaccessible. The fuse burn is voluntary and material. Continued persistence requires funded renewal and registrar continuity; an unrenewed expiry followed by re-registration resets the pointer and roles, the expiry value has a `uint64` ceiling, and the parent `.eth` controller can replace the entire subregistry. Those system-level routes do not give the affected current or later token owner a normal repair transaction.

## Impact Details

### Primary impact: connected wallet signs a transaction with a widened scope

The bug changes the authorization scope in the wallet request. The administrator is editing a scoped delegate, but the Explorer replaces that resource with root:

```
intended scope: profile.eth
signed scope:    resolver root, covering every name on the resolver
```

The local proof executes the privilege boundary before and after the generated transaction: the delegate begins with zero root roles and an unrelated Set Address attempt reverts; after the Explorer-built transaction, it has root Set Address authority and the same class of write succeeds. This maps to the competition's Critical category: “Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters.”

The existing narrow role does not authorize a root role. The application gives no indication that adding Set Address to this row will replace the retained name resource with resolver root and cover every name on the resolver.

### Concrete consequence: unintended resolution and transfer

Once the delegate receives root Set Address, it can redirect any name using that resolver. The lead EVM test changes an unrelated name, uses the deployed ENS v2 `UniversalResolverV2` path, and feeds the resolved address to the Explorer's production transfer helper. The resulting `safeTransferFrom` sends an ERC-1155 name token to the delegate.

The token transfer shows the consequence once another user relies on the changed resolution and approves the transfer. The primary Critical claim is the application-generated wallet request with an unintended root scope.

### Strongest conditional consequence: persistent resolver takeover

If the administrator deliberately selects Upgrade on the same affected row, the wrong scope changes a non-root, unusable Upgrade grant into root code authority. The clean proof demonstrates a first-post-grant transaction that either takes the resolver's complete administration while restoring official code, or installs locked code from an ordinary EOA and redirects every address lookup. In the administration-seizure branch, the former owner can no longer revoke the delegate. This consequence does not need a second victim transaction or payment approval, although the recipient still has to exercise the granted authority.

The UI describes Upgrade as permission to upgrade the resolver contract. The Critical distinction in this branch is therefore not an unselected role bit. It is the undisclosed replacement of the row's non-root resource with resolver root, turning a scoped edit into executable control over the shared resolver and every name that still uses it.

For the supported migrated-and-fused configuration described above, separate real-contract evidence shows that the affected owner cannot revoke the recipient, repair the record, restore the implementation, repoint the registry, regrant itself pointer authority, or recover that authority by transferring the token. An unrelated payer can renew the same compromised registration without restoring those permissions. This is conditional, persistent loss of the individual owner's resolver control. Ordinary names with independent registry Set Resolver authority can detach, and unrenewed expiry/re-registration or parent-registry intervention remain distinct recovery routes.

### Feasibility and limitations

The following conditions are included so the attack model is clear:

* The beneficiary already has a legitimate narrow resolver role. It has no root role and cannot alter unrelated addresses before the Explorer transaction.
* For the grant path, the resolver administrator must open that row, deliberately add a permission, and sign the request. The application, rather than the attacker, supplies the wrong root scope.
* The connected administrator must have permission to grant the selected role at root; otherwise the transaction fails. The affected blank-scope row belongs to another account whose group contains no mapped scope.
* In the pagination path, the attacker must control enough names or a subregistry to create 101 newer nodes on the shared resolver and wait for indexing. The local proof batches the writes; the public scan establishes the 100-node behavior, not a production attack rate or cost estimate.
* A careful user or wallet may decode `grantRootRoles` and reject it. The tested pre-start application views do not disclose the root scope. This does not establish what a particular extension wallet may decode.
* Before the delegate acts, an administrator with the relevant root admin role can revoke the mistaken grant. After the demonstrated administration seizure, the original owner's revoke fails; after the locked-code branch, its normal in-place restoration fails. Name owners who retain separate registry Set Resolver authority can move their names. The fused composition is a supported but voluntary configuration in which the current and later token owner lack that route; renewal funding, registrar continuity, expiry/re-registration, and parent-registry intervention bound the persistence claim. None of those actions reverses a completed token transfer.
* The three application EVM suites ran on fresh local Anvil chains configured with the Sepolia chain ID. The two component suites inspect application behavior without sending transactions. The conditional recovery tests were local Foundry executions. Public endpoint use was read-only, and no public-chain account was modified.

These conditions limit when the bug can be triggered. The tested transaction still turns an edit on a narrow role into a resolver-wide grant.

## Distinction from nearby reports

Report #92645, confirmed as a duplicate of #90931, concerns a different defect in the same role editor: TanStack's positional row identity can retain one account's checkbox state after another account replaces it. Its fix is stable semantic row identity and editor-state reset.

This report does not require state from another row. The raw role and resource remain attached to the correct account, but the display-derived scope passed into mutation is empty and that empty value is interpreted as root. Fixing #92645 does not fix either the missing `namedResources` data, the 100-node cutoff, or the unsafe empty-scope fallback described here.

Report #92448 and its #89249 duplicate family concern a captured V1-to-V2 migration plan in the Manager app. This report's root cause is in the Explorer's V2 resolver-role query and transaction construction; it does not depend on the captured-plan defect. A completed supported migration supplies only the locked-name configuration used for the conditional recovery test.

The supplied known-issues snapshot, last updated 14 August 2026, contains no entry for resolver role resource mapping, `namedResources`, node pagination, or an unresolved scope being treated as root. This comparison does not establish uniqueness against other competition submissions. Private competition duplicates cannot be searched.

### Supporting regression from the same unsafe fallback

The same unresolved row also makes **Remove user** prepare `revokeRootRoles`, although the permission is stored under a name or record resource. The transaction succeeds while leaving the scoped permission usable. Both the fine-grained and pagination tests reproduce this result. It supports the root-cause analysis and the recommendation to reject unresolved scopes, but it is not the primary Critical impact or a separate severity claim in this report.

## Recommended Fix

1. Treat an unresolved resource as an error and disable Save and Remove. Never use an empty display value as a root sentinel.
2. Use the exact resource identity already retained on every raw role. Fetch `namedResources` or equivalent record-part metadata to identify and display that scope instead of reconstructing only `part = 0` resources.
3. Paginate resolver nodes and role relations, or resolve each role resource directly. A count above the first page must not silently remove authorization scope.
4. Carry the existing resource/bitmap pairing through each mutation. Do not use permissions aggregated across resources as one mutation target.
5. Before opening the wallet, show the recipient, exact resource scope, and every role being granted or revoked. Re-read the relevant role and resource before encoding.

## References

Audited Explorer source (`cda79acaad59711b943fc68207ebb3f1d0ff8596`): <https://github.com/immunefi-team/audit-comp-ens/tree/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal>

ENS v2 contracts source (`48b3e2d39513b9dd32ef1850877a29009bc807b9`): <https://github.com/ensdomains/ens-contracts-v2/tree/48b3e2d39513b9dd32ef1850877a29009bc807b9>

ENS Audit Competition scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>

* Complete test files, raw logs, and read-only evidence are prepared in the accompanying local evidence bundle; the Proof of Concept field records its publication status.

## Link to Proof of Concept

<https://gist.github.com/Xylzlol/0291acd3098902e07c4952751fcdfea9>

## Proof of Concept

## Proof of Concept

The complete test files, read-only query scripts, recorded output, and checksums are in the secret Gist supplied in the dedicated PoC link field. The Gist supports this reproduction; the steps and expected results are included here so the link is not being used as a substitute for the PoC.

The three application EVM suites run state changes against fresh local Anvil chains configured with the Sepolia chain ID. The two component suites inspect application behavior without sending transactions. The conditional recovery tests are separate local Foundry executions. The only public-network operations are read-only GraphQL and contract calls against the competition's Sepolia environment.

### Step-by-step reproduction

This summary composes separate component, read-only endpoint, and local-chain proofs. The local suite models the endpoint's verified newest-100 response shape; it does not run a live indexer, and the public endpoint sample is not the local `profile.eth` fixture.

1. Check out the exact Apps and ENS v2 contract revisions listed below, add the five supplied TypeScript test files to the Portal, add the supplied PoC-only Solidity sources to the ENS v2 checkout, and build the contract artifacts.
2. Run the five Portal suites. Three execute state changes on fresh local Anvil instances; two component suites inspect the production UI behavior without sending transactions. Together they exercise the resolver-role table, sidebar, transaction modal, role grouping, transaction builders, registry contracts, resolver, and Universal Resolver.
3. In the lead EVM path, give the delegate only a whole-name Text role on `profile.eth`. Confirm its root bitmap is zero and that its attempt to change unrelated `beta.eth` rejects.
4. Give the delegate control of its own UserRegistry and register 101 newer child names that point to the shared resolver. Separately, the read-only endpoint probe confirms that the Explorer's unpaginated relation returns the newest 100 nodes and that requesting 101 exposes the next omitted node.
5. Model that verified newest-100 response from the local fixture state and pass it through the production grouping code. The delegate's role remains in the row, but its scope is absent, the row stays editable, and neither the sidebar nor the production pre-start modal says root, global, or all names.
6. As the resolver administrator, check only Set Address / Manager for that displayed delegate and press Save.
7. Compare the correctly scoped request with the request built from the Explorer state. The expected request is `authorizeNameRoles(profile.eth, ROLE_SET_ADDR, delegate, true)`; the Explorer prepares `grantRootRoles(ROLE_SET_ADDR, delegate)`.
8. Mine the exact target and calldata built by the production helper. Confirm the delegate now has root Set Address authority.
9. Repeat the same unrelated `beta.eth` write that rejected before the Explorer transaction. Confirm it succeeds and `UniversalResolverV2` returns the delegate's address.
10. Pass that resolved address to the Explorer's production `transferToken` helper and confirm the local ERC-1155 token for `valuable.eth` is transferred to the delegate.
11. Run the fine-grained-resource suite to reproduce the same root fallback without pagination, and run the Upgrade and locked-name suites to reproduce the conditional administration and recovery consequences.
12. Compare the fresh controls: a mapped name resource prepares `authorizeNameRoles`, the delegate remains unable to modify unrelated names before the affected grant, and a non-root Upgrade grant cannot reach the root-only upgrade path.

### Revisions and prerequisites

Use:

```
ENS Apps: cda79acaad59711b943fc68207ebb3f1d0ff8596
Pinned upstream Apps revision: 63772fd872af472ced58b009499355f3430c2a86
ENS v2 contracts: 48b3e2d39513b9dd32ef1850877a29009bc807b9
```

Install the Apps dependencies. Copy `PortalResolverNodeFlood.poc.t.sol` to `contracts/test/unit/registry/`, and copy `PortalUniversalResolutionHarness.poc.sol` and `PortalLockedUpgrade.poc.sol` to `contracts/test/unit/resolver/` in the ENS v2 checkout, then build the contract artifacts. Provide the tests with the paths to Anvil and `contracts/out`. These PoC-only sources supply `ResolverNodeFlooder`, the minimal Universal Resolver harness, and the storage-compatible Upgrade consequence targets; they are included in the local evidence bundle.

On Linux or macOS, from `apps/portal`:

```bash
RESOLVER_AUDIT_ANVIL_BIN=/path/to/anvil \
RESOLVER_AUDIT_ARTIFACTS=/path/to/ens-contracts-v2/contracts/out \
../../node_modules/.bin/vitest run \
  src/features/resolver/components/ResolverRolesSidebar.audit.graphql.test.tsx \
  src/features/resolver/components/ResolverRolesSidebar.audit.real-modal.test.tsx \
  src/features/resolver/components/ResolverRolesSidebar.audit.part-resource.evm.test.ts \
  src/features/resolver/components/ResolverRolesSidebar.audit.pagination.scope.evm.test.ts \
  src/features/resolver/components/ResolverRolesSidebar.audit.scope-upgrade-seizure.evm.test.ts \
  --reporter verbose --silent=false
```

### Test 1: recorded endpoint data and production component

File:

```
apps/portal/src/features/resolver/components/ResolverRolesSidebar.audit.graphql.test.tsx
```

The recorded run of this suite drove the actual `ResolverRolesSidebar` and production intent builders with responses collected from the competition indexer. It did not send a public-chain transaction. The public data may change between runs.

1. Query the current resolver overview for an active fine-grained Text row whose `namedResources` entry identifies its name and key.
2. Render that response through the Explorer's current query shape, which omits `namedResources`.
3. Open the role row. Confirm the scope label is absent although the permission checkbox is present.
4. Check Set Address / Manager and save.
5. Decode the captured intent. Confirm it is `grantRootRoles(ROLE_SET_ADDR, delegate)`, not a scoped authorization call.
6. Repeat with an active fine-grained ETH-address delegate and click Remove user. Confirm the prepared call is `revokeRootRoles` and does not target the indexed part resource.
7. Query the live resolver's default node relation and its first 101 nodes. Confirm that the default response omits the 101st name. Construct an ordinary name-scoped role for that omitted name and use it with the live node response in the component test. This isolates the application behavior without claiming that the sampled public name currently has such a role.
8. Render the resulting row, add Set Address, and confirm the intent again decodes as `grantRootRoles`.

Recorded result:

```
Test Files  1 passed (1)
Tests       7 passed (7)

PORTAL_LIVE_FINE_GRAINED_ADDR_REMOVE_TO_ROOT_VERIFIED
PORTAL_PAGINATION_ROOT_ESCALATION_VERIFIED
```

Representative recorded Text result:

```json
{
  "liveResolver": "0x1a0578825afdf388f5107117f81a57375cf7060f",
  "liveDelegate": "0x1d494e7fdb3a6b4161400b7143ea97a68314c040",
  "name": "openai.rewall.rewall-test-1.eth",
  "recordKey": "rewall.allow",
  "portalOverviewQueryIncludesNamedResources": false,
  "portalResolvedNames": [],
  "sidebarScopeLabelPresent": false,
  "explicitOwnerEdit": "checked Set Address / Manager",
  "preparedFunction": "grantRootRoles",
  "preparedScope": "root",
  "newlyGrantedRole": "ROLE_SET_ADDR"
}
```

Representative recorded node-window result:

```json
{
  "liveResolver": "0x43ee991f5c681f55216ac7cb8cf3fb3883cd787d",
  "liveNodeCount": 159165,
  "portalNodeCount": 100,
  "firstOmittedName": "ggojvwwui.eth",
  "returnedOrder": "newest createdAt first",
  "prerequisitePreparedByPortal": "authorizeNameRoles",
  "portalResolvedNames": [],
  "scopeLabelPresent": false,
  "explicitEdit": "Set Address / Manager for existing scoped delegate",
  "preparedFunction": "grantRootRoles",
  "preparedScope": "root"
}
```

### Test 1b: production pre-start transaction modal

File:

```
apps/portal/src/features/resolver/components/ResolverRolesSidebar.audit.real-modal.test.tsx
```

This focused test renders the actual `ResolverRolesTable`, `ResolverRolesSidebar`, and production `TransactionModal`. It does not replace the modal with the intent-capture button used by Test 1.

1. Supply a role row whose retained resource is absent from the client node-derived map.
2. Open that account through the real table and sidebar and confirm neither a scoped-name label nor `Global roles (all names)` is present.
3. Check Set Address / Manager and press Save.
4. Observe the actual transaction overview. It shows `Save resolver role changes`, status, and estimated cost. It does not show the target account or a resource scope.
5. Capture the request passed through the modal's production estimate path and decode it as `grantRootRoles(ROLE_SET_ADDR, delegate)`.
6. Open the modal's optional information view. It adds `Update roles for <account>` but still displays no root, global, all-names, or other scope text.
7. Repeat with Upgrade / Manager. Confirm the sidebar describes it as `Can upgrade the resolver contract`, while the row, overview, and information view still do not say root, global, or all names.
8. Decode the request as `grantRootRoles(ROLE_UPGRADE, delegate)`.

Fresh result:

```
Test Files  1 passed (1)
Tests       2 passed (2)
RESOLVER_REAL_MODAL_SCOPE_OMISSION_VERIFIED
RESOLVER_REAL_MODAL_UPGRADE_SCOPE_OMISSION_VERIFIED
```

The test exercises the application review boundary. It does not record an extension wallet and makes no claim about how a particular wallet decodes calldata.

### Test 2 (lead): attacker-controlled node pagination turns a whole-name edit into a root grant

File:

```
apps/portal/src/features/resolver/components/ResolverRolesSidebar.audit.pagination.scope.evm.test.ts
```

1. Deploy the reference resolver, registry stack, `UserRegistry`, and `UniversalResolverV2` on a clean local chain.
2. Use the production grant helper to give the delegate only a whole-name Text role on `profile.eth`. Decode and confirm the starting call is `authorizeNameRoles` and the delegate has zero root roles.
3. Before the affected edit, attempt to set the address of unrelated `beta.eth` from the delegate wallet. Confirm the call reverts.
4. Register `attacker.eth` to the delegate. Use the Portal's production helpers to deploy a UserRegistry proxy controlled by that delegate and attach it under `attacker.eth`.
5. Give a batching contract registration authority only in the attacker's registry. In one transaction, register 101 child names pointing to the target shared resolver. Confirm 101 `LabelRegistered` and 101 `ResolverUpdated` events and gas below 30 million.
6. Model the indexer's separately verified newest-100 relation. Confirm `profile.eth` is absent while its ordinary name-scoped role remains in the role list.
7. Pass the role through the real grouping function. Confirm the row remains editable and its `resolvedNames` list is empty.
8. Generate the intended Set Address transaction for `profile.eth`. Confirm it is `authorizeNameRoles`.
9. Generate the transaction from the Explorer's derived empty scope. Confirm it is `grantRootRoles`, mine it, and confirm the delegate gains root Set Address.
10. Repeat the previously rejected `beta.eth` address write from the same delegate. Confirm it now succeeds.
11. Resolve through the deployed `UniversalResolverV2` and transfer the local ERC-1155 token for `valuable.eth` to the resolved delegate using the Explorer's production transfer helper.

The EVM test does not run an indexer. The public read-only probe establishes the current 100-node response and newest-first ordering; this test models that verified response after producing the corresponding 101-node chain state.

Recorded result:

```
Test Files  1 passed (1)
Tests       1 passed (1)
PORTAL_PAGINATION_FULL_EVM_CHAIN_VERIFIED
```

Key output:

```json
{
  "startingGrant": {
    "functionName": "authorizeNameRoles",
    "name": "profile.eth",
    "role": "ROLE_SET_TEXT",
    "delegateRootRolesBeforeEdit": "0",
    "unrelatedSetAddressBeforePortalEdit": "reverted"
  },
  "attackerCutoff": {
    "newNodes": 101,
    "transactionCount": 1,
    "gasUsed": "9141803",
    "portalWindowAfterRefetch": 100
  },
  "portalEdit": {
    "scopeShown": false,
    "explicitPermission": "Set Address / Manager",
    "preparedFunction": "grantRootRoles",
    "status": "success"
  },
  "concreteLoss": {
    "poisonedName": "beta.eth",
    "portalTransferFunction": "safeTransferFrom",
    "transferredName": "valuable.eth",
    "attackerTokenBalanceAfter": "1"
  }
}
```

### Test 3: omitted record-resource metadata produces the same unsafe fallback

File:

```
apps/portal/src/features/resolver/components/ResolverRolesSidebar.audit.part-resource.evm.test.ts
```

This test is independent corroboration for the empty-scope-to-root root cause. It is not the lead scope-intent argument.

1. Deploy the reference `PermissionedResolver` through `VerifiableFactory` using the Portal's production initializer.
2. Give the delegate only `ROLE_SET_TEXT` for one text key on `dave.eth` with `authorizeTextRoles`. Verify `roles(root, delegate) == 0`.
3. Before the affected edit, attempt to set the address of unrelated `beta.eth` from the delegate wallet. Confirm the call reverts.
4. Pass that role and a complete node list through `buildResourceToNameMap` and `groupRolesByAccount`. Confirm `resolvedNames` is empty because the client maps only `part = 0`.
5. Reproduce the component's Set Address edit with the production `prepareGrantResolverRolesTransaction` helper and the derived empty name.
6. Decode and mine the exact generated request. Confirm it is `grantRootRoles(ROLE_SET_ADDR, delegate)`.
7. Repeat the previously rejected unrelated address write. Confirm it now succeeds and `UniversalResolverV2` returns the delegate.
8. Pass the resolved address into the Explorer's production `transferToken` helper for `victim.eth`. Confirm the generated `safeTransferFrom` sends the local ERC-1155 token to the delegate.

Because a Text-key resource is not a valid Set Address scope, the safe behavior for this branch is to display the retained resource and reject an incompatible edit. The defect is that the application silently substitutes resolver root.

Recorded result:

```
Test Files  1 passed (1)
Tests       1 passed (1)
```

Security-relevant output:

```json
{
  "initialDelegatePrivilege": {
    "role": "ROLE_SET_TEXT",
    "name": "dave.eth",
    "recordKey": "dvod.endpoint",
    "rootRolesBeforePortalEdit": "0",
    "unrelatedSetAddressBeforePortalEdit": "reverted"
  },
  "portalRow": {
    "resolvedNames": [],
    "sidebarScopeLabelPresent": false,
    "explicitOwnerEdit": "checked Set Address / Manager"
  },
  "portalBuiltRootGrant": {
    "functionName": "grantRootRoles",
    "roleBitmap": "1",
    "status": "success",
    "rootSetAddrAfter": true
  },
  "unrelatedNameHijack": {
    "name": "beta.eth",
    "immediatelyFollowedRootGrant": true
  },
  "portalSendNameTransfer": {
    "typedRecipient": "beta.eth",
    "victimName": "victim.eth",
    "functionName": "safeTransferFrom",
    "attackerTokenBalanceAfter": "1"
  }
}
```

In the raw log, `universalResolverUsed` is the resolver address returned by `UniversalResolverV2`, not the address of the Universal Resolver contract. The test calls the Universal Resolver contract and records its `resolverUsed` return value. The pre-grant Address controls assert that the simulation rejects; they do not pin a particular EAC error selector.

### Test 4: root Upgrade becomes resolver administration and code control

File:

```
apps/portal/src/features/resolver/components/ResolverRolesSidebar.audit.scope-upgrade-seizure.evm.test.ts
```

1. Deploy two clean reference resolvers through `VerifiableFactory`.
2. On the control resolver, use the production helper with `profile.eth` and Upgrade. Confirm it encodes `authorizeNameRoles`, leaves the recipient's root bitmap at zero, and cannot reach the resolver's root-only UUPS upgrade path.
3. On the affected resolver, give a contract delegate only one Text-key role on `profile.eth`. Confirm the exact name is present, the Explorer grouping still returns `resolvedNames = []`, and the delegate has zero root roles.
4. Use the production helper with the derived empty name and Upgrade. Confirm it encodes and mines `grantRootRoles(ROLE_UPGRADE, delegate)`.
5. In the first post-grant state-changing transaction, call the predeployed recipient. It installs a storage-compatible implementation, moves the owner's complete resolver root/admin bitmap to itself, restores the official implementation, and changes the address of unrelated `treasury.eth`.
6. Confirm the factory again reports the official implementation, the former owner's root bitmap is zero, the delegate holds the fixture's complete bitmap, `treasury.eth` returns the chosen address, and the former owner's revoke transaction reverts.
7. Repeat on a third clean resolver with an ordinary EOA delegate. Begin with only the same Text-key role and zero root roles. Predeploy the compatible locked implementation before the erroneous grant.
8. Mine the Explorer-built root Upgrade grant, then use one EOA transaction in the next block to install the locked implementation. Confirm an unrelated address lookup returns the EOA and the resolver owner's attempt to restore the official implementation reverts.

Recorded result:

```
Test Files  1 passed (1)
Tests       1 passed (1)
ENS_SCOPE_CLEAN_ATOMIC_ADMIN_SEIZURE_VERIFIED
ENS_SCOPE_EOA_LOCKED_RESOLVER_VERIFIED
```

Key output:

```json
{
  "cleanBaseline": {
    "initialRole": "one text key on profile.eth",
    "initialRootRoles": "0",
    "priorRootSetAddressGrant": false
  },
  "correctScopeControl": {
    "preparedFunction": "authorizeNameRoles",
    "rootRolesAfterGrant": "0",
    "upgradeAttackStatus": "reverted"
  },
  "affectedPortalIntent": {
    "resolvedNames": [],
    "preparedFunction": "grantRootRoles",
    "grantedRole": "ROLE_UPGRADE"
  },
  "immediateConsequence": {
    "attackerTransactionCount": 1,
    "officialImplementationRestored": true,
    "ownerRootRolesAfter": "0",
    "poisonedName": "treasury.eth",
    "ownerRevokeStatus": "reverted"
  },
  "ordinaryEOABranch": {
    "postGrantAttackerTransactions": 1,
    "everyAddressLookupRecipient": "delegate EOA",
    "ownerRestoreStatus": "reverted"
  }
}
```

The Upgrade description is visible, and the administrator deliberately selects it. The bug demonstrated here is the undisclosed substitution of resolver root for the unresolved non-root resource. The consequence targets are audit-only contracts; their behavior shows what the root Upgrade permission authorizes. This is not presented as a contract vulnerability or as transfer of registry ownership.

### Test 5: locked-name recovery and renewal boundary

Files:

```
LockedMigrationController.fused-recovery.patch
LockedFusedAttackerRenewal.audit.t.sol
PortalLockedUpgrade.poc.sol
```

Apply the focused patch to the pinned ENS v2 contracts checkout, place `LockedFusedAttackerRenewal.audit.t.sol` in `contracts/test/unit/migration/`, and place `PortalLockedUpgrade.poc.sol` in `contracts/test/unit/resolver/`. From `contracts/`, run:

```bash
forge test --match-path test/unit/migration/LockedMigrationController.t.sol --match-test test_audit_fusedNameCannotEscapeLockedResolverUpgrade --out audit-out-locked-recovery --cache-path audit-cache-locked-recovery -vv
forge test --match-path test/unit/migration/LockedMigrationController.t.sol --match-test test_audit_fusedNameCannotEscapeStealthResolverSeizure --out audit-out-locked-recovery --cache-path audit-cache-locked-recovery -vv
forge test --match-path test/unit/migration/LockedFusedAttackerRenewal.audit.t.sol --match-test test_attackerCanPayToRenewFusedHijackWithoutRepairingOwnerAuthority --out audit-out-locked-recovery --cache-path audit-cache-locked-recovery -vv
```

Expected markers:

```
PORTAL_FUSED_NAME_LOCKED_RESOLVER_TAKEOVER_VERIFIED
PORTAL_FUSED_STEALTH_SEIZURE_VERIFIED
ENS_FUSED_ATTACKER_RENEWAL_VERIFIED
```

The first two tests start with a real `PermissionedResolver`, a wrapped ENSv1 `.eth` name, and the owner-authorized `CANNOT_UNWRAP | CANNOT_SET_RESOLVER` fuse state. Manager's supported locked migration retains that resolver and gives the ENSv2 owner neither `ROLE_SET_RESOLVER` nor its admin role. After the modeled root Upgrade grant, the tests cover a locked-code takeover and a separate administration-seizure branch that restores the official implementation. They verify failed owner repair, failed pointer replacement, failed owner role regrant, failed revocation, and failed repair by a later token recipient.

Both fresh branches begin with a 3,650-day registration and print `320716801` seconds remaining after migration. The renewal test moves to one second before expiry and uses the real public `ETHRegistrar.renew` entry point from an unrelated funded payer. It extends the same registration by 365 days while preserving ownership, the resolver pointer, missing owner authority, and attacker-controlled Universal Resolution.

These are separate composition tests. Test 4 proves that the Explorer-generated request grants usable root Upgrade. Test 5 begins from that exact authority and isolates its consequences in a supported locked-name state. It is not represented as one atomic browser-to-migration run. The fuse burn is voluntary; persistence requires funded renewals and registrar continuity; unrenewed expiry followed by re-registration resets the state; the expiry field has a `uint64` ceiling; and the parent `.eth` controller can replace the subregistry. Raw Foundry output may include a nonfatal unresolved-symbol diagnostic from vendored source alongside the passing result and zero exit code.

### Supporting regression: Remove user targets root instead of the retained scoped role

The two EVM suites contain a separate Remove branch. For a permission stored under a name or record resource, the blank-scope row prepares and successfully mines `revokeRootRoles`. The actual scoped permission survives and remains usable. This supports the unsafe-fallback root cause and proposed fix; it is not the lead Critical consequence.

### Controls

The tests include the following controls:

* A complete mapping of an ordinary name resource produces the expected `authorizeNameRoles` transaction.
* Before the Explorer-built grant, the delegate has zero root authority and its unrelated Set Address call reverts.
* After the exact Explorer-built request is mined, the same class of write succeeds.
* The non-root Upgrade control leaves root authority at zero and cannot execute the root-only upgrade path.
* The administration-seizure branch changes `treasury.eth`, which is unrelated to the original `profile.eth` Text-key resource.
* The ordinary-EOA branch predeploys its target before the root grant, so the only post-grant attacker transaction is the upgrade itself.
* The role hidden by the node window remains an ordinary name-scoped role; the attacker-created nodes do not alter it.
* `revokeRootRoles` succeeds but cannot remove a permission stored under a name or part resource; this is supporting regression evidence.
* The fused tests enumerate immediate owner repair, role regrant, transfer, expiry, renewal, and parent-registry boundaries instead of treating persistence as absolute.
* The address change, Universal Resolution, and token transfer happen only after the unintended grant.

### Supporting read-only census

`probe-resolver-part-scope.py` and `probe-resolver-node-window.py` are the scripts used for the recorded public-endpoint measurements. Their saved JSON outputs include the endpoint URL, counts, exact example resources, and the query-window result; the node-window output also includes its completion token. The saved reachability census records successful `eth_call` simulations for root Set Address in 60 current candidate groups and root Upgrade in 48. These measurements establish reachable states at collection time; they do not establish exploitation, mainnet prevalence, or administrator intent. The local packet verifier checks those outputs, reruns all five application suites, and invokes the locked-name verifier before printing:

```
ENS_UNRESOLVED_SCOPE_CRITICAL_PACKET_VERIFIED
```

The separate bundle-integrity verifier checks the exact 25-file set, all 24 SHA-256 entries, saved completion tokens, and source-to-packet test copies before printing:

```
ENS_RESOLVER_SCOPE_LOCAL_PACKET_VERIFIED
```

The secret Gist linked in the dedicated PoC field must contain the exact verified 25-file bundle from `ENS_RESOLVER_SCOPE_CRITICAL_GIST_READY`; `SHA256SUMS.txt` binds every supporting file reviewed with this report.
