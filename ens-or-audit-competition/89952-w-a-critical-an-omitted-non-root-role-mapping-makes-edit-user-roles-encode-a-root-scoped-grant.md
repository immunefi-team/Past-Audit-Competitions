# 89952 w a critical an omitted non root role mapping makes edit user roles encode a root scoped grant

**Submitted on Aug 22nd 2026 at 03:02:35 UTC by @hmiya for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89952
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Selected asset and impact

* Asset: `apps/portal`
* Listed impact: **Critical — Malicious interactions with an already-connected wallet, including modifying transaction arguments or parameters or submitting malicious transactions**

The live scope was re-observed read-only on 22 August 2026. Its exact row uses an open-ended `such as` list containing modified transaction arguments or parameters, substituted contract addresses, and submitted malicious transactions; it does not restrict the impact to XSS, compromised dependencies, session hijacking, or external calldata injection. The live information page also identifies anything that lets a transaction be built, signed, or attributed with the wrong chain, sender, target address, or arguments as a priority concern. The observation artifact is SHA-256 `04fc5af18ce65be23d193f748b63920419b90985f86479a97a2a341b000c9d71`.

The relevant Websites and Apps exclusion concerns un-prompted in-app actions outside normal workflows. This report instead uses the Portal's prompted, first-party **Edit user roles**, **Save**, and **Open wallet** flow; A remains an independent signer and no hidden click or forged prompt is assumed.

## Summary

The resolver roles page fetches raw role rows together with a limited `nodes` array. It resolves a role's resource hash to a human-readable name only through that array. The reviewed route uses `graphql-request` and TanStack Query, not an Apollo normalized cache; a refetch replaces the returned node array rather than merging old node objects back into it.

The successful signer class is deliberately narrow: A must hold the selected role's **root-scoped admin counterpart**. The PoC separately proves that a name-only role administrator cannot execute the widened root grant. The defect does not bypass A's authority; it causes the Portal to spend that authority on a broader resource than the ordinary row edit preserves or discloses.

No existing deployed victim is claimed, and the exact default-window role-without-name response has not been observed on one resolver. The current backend contract and a same-resolver paired read establish the narrower reachability facts: `nodes` is a limited projection, while `roles` and `namedResources` are separately returned contract-scoped projections. Changing only `nodes.first` to zero removed every returned node without changing the non-root role or its authoritative resource-name mapping. This proves projection independence, but not the complete omitted/default conjunction.

If X has a legitimate role scoped to one older name, but that name is absent from the current node window, X's raw role remains in the table with no resolved name. When administrator A uses the ordinary **Edit user roles** action on that row to add a permission requested for the same delegated name, the Sidebar derives an empty `roleName`, shows neither a name-scoped nor a global scope warning, and presents only `Update roles for X` before the **Open wallet** action.

The transaction helper treats that empty name as the resolver root. The Portal therefore encodes:

```solidity
grantRootRoles(selectedRole, X)
```

instead of the name-scoped call represented by the narrow permission workflow:

```solidity
authorizeNameRoles(name, selectedRole, X, true)
```

Internally, the helper records `Grant resolver roles for (root)` in `TransactionOptions.description`, so the system has already classified the empty name as root. That description is not part of the Portal's display `Transaction` type and none of the overview, info, or pre-wallet state views renders it. The Edit flow's visible `transactionName` remains only `Update roles for X`. This is not imported from the separate **Add user** flow, which does use a different `(root)` transaction name.

The grant is root-scoped rather than name-limited. The PoC proves that this creates a capability X did not previously have and that X can consume it to replace a value previously committed by distinct legitimate scoped writer V on another name. Revoking X's root role blocks later writes but does not restore the already-committed value.

## Source binding and root cause

The listed asset is `apps/portal` at audited fork root `63772fd872af472ced58b009499355f3430c2a86`. The competition source reviewed here is `1c9b47f18fcddd2e864dfe385c4171061c9811ae`; the full `apps/portal` tree and every cited file are byte-identical between the competition fork baseline and that head. The baseline commit is `cda79acaad59711b943fc68207ebb3f1d0ff8596`; both boundaries have the exact `apps/portal` tree `33d5046a405f21a383e5515693511f5f8b17529b`, and the scoped diff is empty. The source-boundary observation is SHA-256 `86516031471976150020df65e2d7eecc9ffec1d4667b06c743297107694657ce`. The public competition repository does not contain the upstream object `63772fd...` itself. The relation therefore relies on the platform's audited revision declaration together with the fork-root provenance, exact ancestry, byte-identical Portal tree, and empty scoped diff; it does not claim a direct checkout of that missing object.

`useResolverOverview` requests `nodes` without a `first` value and requests `roles` independently. Read-only introspection of the staging schema describes `nodes` as "Domains using this resolver (limited)" with optional `first: Int`, while `roles` and `namedResources` have no nodes-window argument. The same schema describes `namedResources` as the resource-name associations announced by the resolver. The route requests `roles` and `nodes`, but does not request `namedResources`; it passes `resolver.roles` and `resolver.nodes` directly to `ResolverRolesTable`. There is no role-resource backfill, `fetchMore`, or load-more step, and the grouping path ignores `role.name`.

This absence was checked across the production call chain, not inferred only from the PoC: `ResolverRolesTable` is the sole production caller of `ResolverRolesSidebar`; it constructs the row with `groupRolesByAccount(roles, buildResourceToNameMap(nodes))`; the Sidebar never reads `row.original.roles` or a raw `resource`, performs no reverse lookup or secondary query, and forwards only the derived `roleName` into Save. The source-only closure artifact is SHA-256 `79e70ca4034d6cdfc76ec4e76a426c9f9eebd724346329c1e88fcb217609e224`.

The resource-to-name map contains only the exact root constant and hashes computed from the returned nodes. An unmapped role remains in its account row, but contributes no `resolvedNames` entry. `ResolverRolesSidebar` then uses:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

Both Save and Remove forward that value. For grants, the helper selects `scope: 'root'` exactly when `name === ''`; every nonempty name selects `scope: 'name'`. The resulting write parameters are encoded directly into the EOA custom intent. A known non-root resource is therefore widened only because its presentation mapping is absent.

The final confirmation path was checked separately from the grant helper. The Edit flow sets `transactionName` to `Update roles for <account>` and passes that descriptor to `TransactionModal`. The Portal `Transaction` display type has no `description` field; `TransactionModal` accepts only those descriptors; and all three child views render only title, transaction name, cost, steps, status, or error. The state view renders `transactionName` and only then exposes **Open wallet**. A complete source search under the Portal transaction-manager feature finds no `description` consumer. This closure is recorded in `edit-flow-confirmation-surface.v1.json` and avoids relying on the PoC's modal shim for a negative disclosure claim.

## Ordinary-admin workflow and prerequisites

Actors:

* A: connected resolver administrator holding the selected role's root-scoped admin counterpart;
* X: existing legitimate name-scoped delegate and attacker;
* V: distinct legitimate scoped writer on another name using the resolver.

This authority precondition is exact. A separate verified-resolver control grants another principal only the selected role's name-scoped admin counterpart; that principal's `grantRootRoles` call reverts and leaves root state unchanged. The initialized root administrator makes the same call successfully. The report does not claim that a name-only administrator can land the widened grant.

The required pre-state is a raw non-root role whose mapped name is outside the accompanying limited `nodes` projection. The schema is consistent with that state: `roles` is contract-scoped and has no nodes-window argument, while `nodes` is explicitly limited. On resolver `0x0e7c...1e36`, changing only `nodes` from its default projection to `first: 0` removed the returned name but left both the non-root `SET_TEXT` role and the matching `namedResources` entry canonically identical. A top-level `roles(contract, protocol: "v2")` query returned the same role payload. This proves that those projections are not filtered by the selected `nodes` cardinality. It does not by itself exclude a special omitted/default-only node-prioritization or backfill rule. No such rule is exposed by the schema or observed in the sampled responses, and no complete omitted/default-window conjunction was captured.

Role persistence and ordinary state construction are separately bounded. An exact contracts-v2 control registers `older-name.eth` on R, grants X a non-root role on R, repoints the registry entry to R2, and proves R still stores X's unchanged name-scoped role. A current default query for resolver `0x43ee...787d` reports 24,887 nodes but returns only 100. The exploit's primary precondition is simply a real non-root role whose name is absent from that bounded response; attacker-driven displacement is **not** required by the claim. The optional attachment control only shows that more than 100 ordinary foreign-resolver attachments are contract-reachable. It is supporting plausibility, not a claim that X controls production indexer ordering or has already created the conjunction.

The bounded workflow is:

1. X initially has only `ROLE_SET_ADDR` for `older-name.eth`; X has no root role and cannot modify the text value V committed on `other.eth`.
2. X reads the same public resolver-detail query used by the Portal and proceeds only if `older-name.eth` is absent while X's raw role remains.
3. X asks A to add the ordinary **Set Text** permission for X's existing delegation. This is the normal role-management workflow, not phishing or a forged prompt; A independently chooses the permission and whether to sign.
4. A opens the standard resolver Roles page, clicks **Edit user roles** on X's existing non-root role row, selects **Set Text — Can set text records**, and presses **Save**.
5. The Portal displays no scope sentence. Its pre-wallet state view says only `Update roles for X`, while the non-rendered helper metadata records `Grant resolver roles for (root)`. It prepares an EOA request for `grantRootRoles(ROLE_SET_TEXT, X)`.
6. After A signs and submits the Portal-generated request, X uses the new root role to replace the value V previously committed on `other.eth`.

Under the backend-reachability and root-admin preconditions above, the listed connected-wallet impact is satisfied because A performs a scope-preserving edit of X's existing non-root role row pursuant to a narrow same-name permission request, but the Portal discards the row's raw resource, omits scope, and constructs a root grant with a different entrypoint and resource scope. This is a target-generated substitution: X is the recipient and later consumer of the new authority, while no separate process needs to inject or mutate calldata.

## Read-only reachability observations

Current introspection at `https://staging-graphql.ens.dev/graphql` describes `ResolverDetail.nodes` as limited and parameterized by `first`, but exposes `ResolverDetail.roles` and `ResolverDetail.namedResources` without a nodes argument. `EACRoleAssignment.contractAddress` is the contract that emitted the role event, and `NamedResourceEntry` is the resource/name association emitted by the resolver.

For resolver `0x0e7cd4d93c25352d80b3aa2cdf027e1196bf1e36`, two read-only requests held the resolver, role selection, named-resource selection, protocol, and indexed state constant. The only changed dimension was default `nodes` versus `nodes(first: 0)`. The latter returned zero nodes, but both arms returned the same non-root `SET_TEXT` role and the same resource mapping to `poc-scope-1.eth`. Their canonical role hashes were both `b6580c3f41d879906921ed0cf3c3380bf4fd8e3893f0d3515b17bccab5878779`, and their canonical named-resource hashes were both `b4080b4d240b8e1949d05cf59aad9869428325e8de80c0783f70dab9e39c0504`. The contract-filtered top-level roles result had the same role hash.

This paired response proves projection independence; `first: 0` is not the Portal's omitted/default argument and is not presented as the complete exploit witness. Separately, resolver `0x43ee...787d` currently reports 24,887 nodes while the default projection returns 100, proving that the default projection omits at least 24,787 current names.

At `https://staging-graphql.ens.dev/`, read-only observations on 22 August 2026 returned the default 100 nodes for resolvers reporting 1,277 to 24,887 nodes. For resolver `0x43ee991f5c681f55216ac7cb8cf3fb3883cd787d`, every returned node in a later 100-node response had a `createdAt` value greater than the newest returned node in an earlier response. Three repeated reads of unchanged resolver `0xae66c62acae72098bdac57d8e8aed53ef000b2ba` returned the identical ordered array. This is empirical recency-consistent behavior, not a server ordering contract.

The schema observation was made at `2026-08-22T02:37:00+09:00`. A separate same-operation control at `2026-08-22T08:46:35+09:00` queried identical `nodes` projections with and without the Portal's `roles` selection for resolvers `0x43ee...787d` and `0x0e7c...1e36`; both pairs returned identical node arrays. The latter resolver's one current non-root role remained mapped, so the default arm is not the missing complete live witness. The `first: 0` control above separately isolates the backend's role/node projection independence without pretending that arm is the Portal request.

A bounded same-day census paginated 24,847 v2 resolver bindings (2,366 unique resolver addresses) and 27,795 role rows. It found 4,099 roles on enumerated v2 resolvers and exactly one non-root resolver role; that resolver had one node, and the node was returned and mapped. Thus no current exact witness was hidden among the enumerated rows. Mutable-endpoint drift prevents treating the census as a permanent completeness certificate, and absence at that time is not a backend invariant or negative proof.

## Known limitations

* No current resolver was observed with the complete default-response conjunction: a returned raw non-root role whose authoritative name is absent from that same resolver's default node-derived map. No existing victim or victim transaction is claimed.
* The `nodes(first: 0)` arm is an isolation control, not the Portal's omitted default argument. It proves that role and named-resource projections are not filtered by the selected nodes cardinality; it does not exclude a special default-only node-prioritization or backfill rule and is not presented as the exploit witness.
* The same-day census is a time-bounded observation of a mutable staging endpoint. Its non-finding is disclosed but is not used as a permanent impossibility claim.
* The endpoint showed stable, recency-consistent 100-item windows, but no immutable ordering contract is claimed. Attacker-controlled displacement is not required by the primary claim; the optional attachment tests establish only contract-level state construction and cost.

These limitations bound prevalence, not the deterministic Portal behavior once the source-accepted precondition is present. Obtaining a synthetic live joint witness would require public-network state changes; this report performs none.

## Displacement-setup feasibility and cost

An exact `contracts-v2` control assigns one principal registry authority, deploys the required registry/helper components, grants only the helper's registrar role, and creates 101 names pointing to a foreign resolver. A separate runtime control makes fixture bootstrap distinct from X, lets only X operate the registry-side relay, rejects an outsider, and shows that X needs no role on or call to the foreign resolver.

The measured 101-attachment action used 9,094,967 gas. The conservative full-fixture bound, including deployment and authority setup, is 13,006,203 gas. This bounds only X's contract-level ability to create more attachments than the observed 100-row response size; it does **not** prove production indexer inclusion, ordering, eviction, a universal threshold, or the exploit's Severity.

For reproducibility only, the read-only Sepolia reference was block 11,537,701 (`0x1eee4071e9a2532939f6ea289d962df13f200149c5c21283bf5608373522fcac`), observed at `2026-08-22T03:14:24+09:00`, with a 60,000,000 gas limit and suggested gas price 1,041,007,153 wei. Thus the conservative full fixture is approximately 21.7% of that testnet block limit and 0.01354 Sepolia ETH at that point-in-time gas price. No fiat or mainnet value is claimed.

## Impact

### Why this is the listed connected-wallet Critical impact

The qualifying event is A's Portal-built transaction, not the downstream text record by itself. A selects a permission while editing X's existing non-root resource row. The Portal then substitutes both the resource scope and the entrypoint: name-scoped `authorizeNameRoles(name, ...)` becomes resolver-wide `grantRootRoles(...)`. Before **Open wallet**, the final Portal surface still says only `Update roles for X`; it does not expose the internal `(root)` description or identify the all-name scope. This is a first-party modification of the transaction arguments and parameters presented to an already-connected wallet, with attacker X as recipient and consumer. It does not depend on XSS, a compromised dependency, or an external process rewriting calldata.

A must already possess the selected role's root-scoped admin counterpart. That is an authority prerequisite, not informed consent to the substituted grant: the mapped control uses the same administrator action and remains name-scoped, while the unresolved non-root row alone becomes root-scoped.

The exact-contract control starts with V, distinct from A and X, as a legitimate scoped writer who commits `url=victim` on `other.eth`. It proves X lacks that write capability before A's root grant. After the grant, X commits `url=attacker`. A can revoke X's root role, but revocation only blocks later X writes; the attacker value remains until a separate authorized repair writes a replacement.

The defect is that the Portal causes A to grant materially broader authority than the edit context represents. The report does not claim permanent resolver takeover, theft, signature bypass, name ownership by V, or a currently observed victim transaction. It claims the listed connected-wallet selector/scope substitution. V's overwrite proves that the new authority is consumable and creates committed third-party state; it is not a claim that an isolated text record modification is inherently Critical.

The helper widening is role-mask independent. A second exact-resolver control repeats the causal contrast with `ROLE_SET_ADDR`: V commits the original Ethereum address, X cannot change it before the root grant, the root-scoped mask lets X replace it, and revocation blocks later X writes without restoring V's address. `ROLE_SET_TEXT` is the simplest visible proof, not the consequence ceiling.

## Known-issue boundary

Against the 59-row public-known snapshot last updated 14 August 2026 and observed `2026-08-18T12:38:13.196Z` (snapshot SHA-256 `743dd596f4f8d645f379e95988f4f24d4dc9e2d19d9beb4a7d5a9e52e4342c9f`), no row states the chain `independently returned roles plus limited nodes → unresolved non-root resource → empty-name root sentinel → root grant`. QA-01 discusses role combinations and resulting errors but does not state this unintended root-scope grant or the demonstrated broader authority. Private duplicates and private fixes remain unknown; no claim of novelty, duplicate absence, reward, or favorable sharing economics is made.

## Recommended fix

Preserve and branch on the raw resource, and fail closed for unresolved non-root resources. Pagination is defense in depth, not the security boundary.

1. Preserve each raw role resource through grouping and editing. Treat a role as root-scoped only when its raw resource is exactly `ROOT_RESOURCE`.
2. If a non-root resource cannot be resolved to a name, display the raw resource and disable editing until the name is resolved. If the contract exposes a resource-preserving update path, that path may be used instead.
3. Fetch or paginate every node mapping required for each returned role rather than joining an unpaginated role set to a limited node projection.
4. Show the exact resource scope and decoded transaction in final confirmation, with an explicit warning for root grants.

Regression tests should hold account and selected role constant across three cases: mapped non-root remains name-scoped; unmapped non-root emits no grant and fails closed; exact `ROOT_RESOURCE` may emit a root grant only with explicit root context.

## Link to Proof of Concept

<https://gist.github.com/gok11/4da432ec69a622856d1ab4bcce645de8>

## Proof of Concept

## Proof of concept

Complete multi-file Gist (no Zip):

```
https://gist.github.com/gok11/4da432ec69a622856d1ab4bcce645de8
```

The locally validated 20-file transfer manifest is SHA-256 `52159d7d0c6e9b8eef0b0067fdd70c0b4eeda85a23ffea193fce0a014da96cfc`. The Gist contains a manifest-equivalent README, the seven-test Portal reproduction, exact resolver consequence, optional attachment/cost control, runtime authority control, read-only staging evidence, exact source/display closures, current scope wording, and all build inputs.

Expected primary results:

```
Portal:   Test Files 1 passed (1); Tests 7 passed (7)
Resolver: [PASS] 3 tests; 0 failed
Repoint:  [PASS] testNameScopedResolverRoleSurvivesRegistryResolverRepoint()
Cost:     [PASS] testOnePrincipalFullSetupAndAttachmentGas()
Authority:[PASS] testRuntimeAttackerNeedsNoAuthorityOnForeignResolver()
```

The Portal controls hold X, the selected role, and user action constant across three ordinary-table cases: mapped non-root produces `authorizeNameRoles`; unmapped non-root produces an unlabeled `grantRootRoles`; exact root produces `grantRootRoles` with explicit `Global roles (all names)` context. They assert that the non-root raw resource equals the resource for `older-name.eth` and is not `ROOT_RESOURCE`. For the unmapped case they also assert that neither `Roles scoped to ...` nor `Global roles (all names)` is rendered and that the transaction name is only `Update roles for X`. A production TanStack Query refetch control proves the old mapped node is not cache-merged into the displaced 100-node response. Its `staleTime: 1 hour` and `retry: false` settings match `apps/portal/src/utils/queryClient.ts` exactly (SHA-256 `7d687b8d07f17c5f44e9ec69b91d08d55f328832911443d64f0546b7f7482976`); the production client defines no structural-sharing override, merge policy, or field policy. The captured transaction-manager request binds the target resolver, selector, exact `ROLE_SET_TEXT` mask, X, and A's EOA signer. No public network mutation or wallet signature is required.

The exact resolver suite also holds the requested root-grant call constant across two administrator scopes: a name-only administrator reverts, whereas the root-scoped administrator required by this report succeeds. This makes the successful signer class an explicit prerequisite rather than an inferred victim population.

The verified Sepolia standard-input source used by the Foundry proof is byte-identical to the referenced contracts-v2 source at `48b3e2d39513b9dd32ef1850877a29009bc807b9`: `PermissionedResolver.sol` is SHA-256 `d2bec253d9cf9cf982ad8b27ea690050118fb27e6bd373cc74c15459d7982167` and `EnhancedAccessControl.sol` is SHA-256 `167073195fe95b2f544c73ec73fc7a512131f12e339aec9feaacc02d04a980d6` in both sources. ENS smart contracts are not claimed as the vulnerable in-scope asset; this identity only binds the downstream authority consequence to the verified deployment used by the PoC. The vulnerable source and proposed fix remain in `apps/portal`.
