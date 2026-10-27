# 92483 w a critical migrate to v2 hand your name s resolver to a third party the app never validated

**Submitted on Sep 10th 2026 at 07:55:18 UTC by @MalikX31 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92483
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

Hi team,

Spent the afternoon in the migration service and hit this while tracing what actually gets granted on-chain during a v1→v2 upgrade. Writing it up now. It's not a records bug, it's who ends up holding a role on the migrated name.

## Brief/Intro

When a user migrates an unwrapped `.eth` name, the Manager builds an atomic batch that includes `grantRoles(resource, ROLE_SET_RESOLVER, managerAddress)`, and `managerAddress` is the v1 ENS Registry controller read straight from the hosted subgraph (`domain.owner.id`), set precisely when it differs from the connected signer. That address is never validated on-chain and never shown to the user before signing. The sibling value in the same object, `tokenHolder`, is re-validated on-chain via `ownerOf`/`getData`; the flow authenticates the token and never authorizes the assignee.

Two measured legs, chained by identical calldata, cover the whole path. Leg 1: I ran the app's own `classifyName` and `buildRoleGrantCall` with the victim as signer and a distinct address as the subgraph controller, and the emitted calldata grants `ROLE_SET_RESOLVER` to that third-party address (measured, real repo functions). Leg 2: I sent that same `grantRoles` on a Sepolia fork, against a real registered v2 name whose owner holds the admin bit, and it lands, the third party goes from no role to holding `ROLE_SET_RESOLVER`, tx status 1, with a control that reverts (measured, fork). The joint between them is trivial because it's the same calldata: Leg 1 proves the Manager constructs the grant with an unvalidated subgraph assignee, Leg 2 proves that grant takes effect on the deployed registry. The row this satisfies, "assigns ... the resolver to an attacker," is met by the grant itself; I did not need to then repoint the resolver.

## Vulnerability Details

**The two halves that disagree.**

Source, the `managerAddress` derivation (in `@ens-apps/migration`, re-exported at `apps/manager/.../classifyNames.ts` and invoked in-scope at `computeMigrationPreflight.ts:201`), `service/classifyNames.ts:162-167`:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

`domain.owner.id` comes from the v1 subgraph (`v1SubgraphClient.ts` GET\_NAMES\_QUERY selects `owner{id}` and `registrant{id}` side by side). The same function forces the connected wallet to equal the registrant (`:150 if (registrant?.id.toLowerCase() !== addr) return null`), so `managerAddress` is by construction an address that is not the signer.

Sink (`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9,16,19-24`):

```ts
const ROLE_SET_RESOLVER = 1n << 24n
...
const resource = labelToCanonicalId(name.label)
return { to: V2_CONTRACTS.ETHRegistry, data: encodeFunctionData({
  abi: ETH_REGISTRY_V2_ABI, functionName: 'grantRoles',
  args: [resource, ROLE_SET_RESOLVER, name.managerAddress] }), value: 0n }
```

The only gate is the null-throw at `:12-14`. Injection into the batch is unconditional on truthiness (`buildAtomicMigrationBatches.ts:613` calls `buildRoleGrantCall(classified)`).

**Where this sits in scope.** The harmful action is native in-scope `apps/manager` code: `buildRoleGrantCall` (`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts`, 27 lines, not a re-export) builds the grant, and `buildAtomicMigrationBatches.ts:613` injects it. The `managerAddress` it grants to is produced by `classifyName`, which the Manager re-exports at `apps/manager/src/features/migration/service/classifyNames.ts` and invokes at `computeMigrationPreflight.ts:201` (`classifyNames([...domains], eoa)`); the derivation body lives in the first-party `@ens-apps/migration` workspace package. So the finding doesn't depend on treating that package as separately in-scope: the in-scope Manager app is what invokes the classification and emits `grantRoles` to its unvalidated output.

And the fix is in-scope too, not only the defect. "The real fix is in `packages/migration` (validate `managerAddress` at the derivation), so this is out of scope" does not hold: the missing check belongs in the in-scope app that emits the grant, and that is where I recommend it. `apps/manager` can read `ENS.owner(namehash(name))` on-chain before building the grant, exactly as its own preflight already re-validates the token owner (`ownerOf`/`getData`) and as the Explorer already does in `V1NameManagerRecord.tsx:35-56` on the same chain object. The missing validation and its remedy both live in the in-scope Manager app.

This is an unvalidated-assignee authorization: an authority grant whose grantee is sourced off-chain and never corroborated, while the token it concerns IS corroborated. I first read `managerAddress` as the victim's own controller being restored (the feature is literally "manager restoration"), then realized the branch fires exactly when the controller is NOT the signer, which in ENS v1 is the default state after any BaseRegistrar `transferFrom` (the NFT moves, `ENSRegistry.owner(node)` does not; the new owner must separately call `reclaim()`), and is also settable in advance by the outgoing controller via `ENSRegistry.setOwner(node, anyAddress)`.

**The read was available and skipped (source).** `preflightChecks.ts:57-71` validates the NFT owner on-chain (`ownerOf`/`getData`) and compares it to `migrationOwner`, but nothing re-derives or checks `managerAddress`. The v1 ENS Registry isn't even in the migration app's address book (`contracts/addresses.ts:11-20` = `{ BaseRegistrar, NameWrapper }`), and `reclaim` has zero hits in `apps/manager`. The portal already does the missing call: `apps/portal/src/features/ownership/components/V1NameManagerRecord.tsx:35-56` reads `owner(bytes32)` off `ensLegacyRegistry` on the same chain object.

**The assignee is never disclosed (source).** `MigrationStepDescriptor` (`buildStepDescriptors.ts:13-31`) has no address field. The only manager-related copy is "Restore the existing managers for your names." (`WalletConfirmationStepsDialog.tsx:87`), which reads as restoring the victim's own managers, and it's attached to a different transaction (the `setApprovalForAll` at `migrationApprovals.ts:116-122`), not the grant. Grepping `managerAddress` across `apps/` and `packages/`, the only `.tsx` hit in the migration feature is a test fixture setting it to `null`; the set is collapsed to a boolean (`.some(n => n.managerAddress !== null)`) before it can reach any UI.

**Duplicate check.** QA-01 in the known-issues list covers role-combination errors on transfer and explicitly carves out that "a role combination that grants unintended authority would be a new finding." This isn't even a role-combination computation (the bitmap is the constant `1n<<24n`); nothing errors; the flow is migration, not transfer; and the defect is the third argument (the grantee), not the roleBitmap. It's adjacent to SEC-TXM-002 (cached address not cross-checked on-chain) but inverts it: the codebase enforces on-chain cross-checking for the cached account address everywhere except the one cached address that becomes a permission grantee. I found no fix branch or well-known-issues entry touching `buildRoleGrantCalls` or the `managerAddress` derivation across the in-scope tag; absence of a note isn't proof nobody reported it.

## Impact Details

An address chosen by a third party, never validated on-chain and never displayed, receives `ROLE_SET_RESOLVER` on the victim's migrated v2 name. This repo documents that role as "Can change the resolver addresses" (`apps/portal/src/lib/roles/permissions.ts:30-34`). The holder repoints resolution to an attacker-controlled resolver, so every wallet resolving the name to an ETH address gets the attacker's, which is the "resolving an ENS name to an attacker-controlled address" impact realized inside a migration.

What makes this the migration's fault and not a v1 carry-over: migration is the exact transition where the attacker's residual v1 controllership would have stopped mattering, since the name leaves the v1 node and resolution moves to v2 where the attacker has no standing. The code re-mints that expiring power as a durable v2 role and, in the same batch, destroys the remedy. In v1 the registrant evicts a controller unilaterally with one cheap `reclaim()`. Post-migration `reclaim()` is meaningless and the Manager offers no substitute (`ETH_REGISTRY_V2_ABI` is grant-only; zero `revokeRoles` in `apps/manager`).

Two ways in, distinct attacker and victim, no privileged role, no leaked key, no phishing:

* Path A: the outgoing controller of an unwrapped `.eth` 2LD optionally calls `ENSRegistry.setOwner(node, ATTACKER2)` (invisible to NFT marketplaces, which render `ownerOf`), then sells the ERC-721 to the victim. The victim migrates; every on-chain check passes; the subgraph returns `owner.id = ATTACKER2`, `registrant = victim`; `ATTACKER2` gets `ROLE_SET_RESOLVER`.
* Path B is the one I'd lead with. The victim does the correct ENS-documented cleanup, `reclaim()`, making themselves the controller, then migrates inside the staleness window (`useV1Names.ts:22 staleTime: 5 * 60 * 1000` plus indexer lag plus the unbounded user-paced gap between plan construction and signing). Nothing re-derives the assignee: the only pre-signature freshness check (`migrationService.ts:367-391`) compares sorted approval-ID sets, and approval IDs carry no assignee address. The evicted party, holding zero on-chain v1 authority at the block the migration lands, is granted the role. That's authority creation, not carry-over.

I'd put this at High. I don't think it's Critical, and I'd rather say so than have you discount the rest: the victim keeps v2 ownership, the granted bit is bare (no `*_ADMIN`, so the grantee can't re-grant), and an on-chain revoke does exist in the portal, just not in the migrating app. The resolver-control-to-a-third-party plus the destroyed remedy is what carries the severity.

## Recommended Fix

Before emitting the grant, read `ENSRegistry.owner(namehash(name))` on-chain and only keep `managerAddress` if it matches the subgraph value at signing time. The app already has this exact call in the portal (`V1NameManagerRecord.tsx:35-56`); the migration path just needs to run it, the same way `preflightChecks` already re-validates the token owner. Systemically, the flow authenticates the token and never authorizes the assignee: any subgraph-sourced value that becomes an on-chain grantee should get the same on-chain confirmation `tokenHolder` gets. And surface the grantee address in the confirmation UI so "restore managers" isn't silently granting a role to an address the user never sees. Don't fix it by dropping the manager grant entirely, that reintroduces the functional gap manager-restoration was added to close.

## References

* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9,16,19-24` (in scope): `grantRoles(resource, ROLE_SET_RESOLVER, managerAddress)` sink
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:613` (in scope): `buildRoleGrantCall(classified)` injected into the atomic batch
* `apps/manager/src/features/migration/service/computeMigrationPreflight.ts:201` (in scope): the Manager invokes `classifyNames([...domains], eoa)`
* `apps/manager/src/features/migration/service/classifyNames.ts` (in scope): re-exports `classifyName` from `@ens-apps/migration`
* `@ens-apps/migration` `service/classifyNames.ts:150,162-167`: managerAddress derived from subgraph `owner.id`, only when it differs from the signer (derivation body, re-exported and invoked in scope above)
* on-chain preflight validates the token owner only (`ownerOf`/`getData`), never the manager/controller assignee
* `apps/manager/src/features/migration/contracts/addresses.ts:11-20`: v1 Registry absent from the app's address book
* `apps/portal/src/features/ownership/components/V1NameManagerRecord.tsx:35-56`: the on-chain `owner(bytes32)` read the migration path omits
* `e2e/scripts/seed-managed-name.ts:50-70`: repo's own statement that the name owner holds `ROLE_SET_RESOLVER` admin at registration (so the grant lands)
* `apps/manager/src/features/migration/service/migrationService.ts:367-391`: the only pre-signature freshness check; keys on approval-ID sets, no assignee address

## Link to Proof of Concept

<https://gist.github.com/SatisLoeb/1d45c4a6ba042ee6c9beb998dc0123fd>

## Proof of Concept

## Proof of Concept

**What the PoC does.** Runs the repo's real `classifyName` (the `@ens-apps/migration` body that the in-scope Manager re-exports at `apps/manager/.../classifyNames.ts` and invokes at `computeMigrationPreflight.ts:201`; body byte-identical to source, only the import header swapped for a shim) and the real in-scope `buildRoleGrantCall` (`apps/manager/.../buildRoleGrantCalls.ts`, same), with real `viem` and real `@ensdomains/ensjs` (the catalog-pinned `e96662c` build) supplying `isAddress` / `labelToCanonicalId` / `encodeFunctionData` / `eacGrantRolesSnippet`. Input is a synthetic unwrapped `.eth` V1Domain matching the subgraph selection set: `registrant = VICTIM` (the signer), `owner.id = ATTACKER2` (the subgraph controller), no resolver. A `diff` of the copied bodies against the in-scope files is included and shows only the import header differs.

**Measured output.**

```
signer (connected wallet == registrant): 0x1111...1111
subgraph domain.owner.id (v1 controller): 0x2222...2222

classifyName().type            = classified
classifyName().tokenType       = unwrapped
classifyName().managerAddress  = 0x2222222222222222222222222222222222222222   [!= signer]

buildRoleGrantCall() ->
  to   = <V2 ETHRegistry>
  decoded fn   = grantRoles
  arg[1] roleBitmap = 16777216   (1<<24 = ROLE_SET_RESOLVER)
  arg[2] account    = 0x2222222222222222222222222222222222222222   [the third party]
```

So with the victim's own wallet as the signer, the Manager builds a `grantRoles` whose grantee is the subgraph controller, an address that is not the signer and was never validated on-chain.

**The grant lands on-chain (measured, Sepolia fork).** I forked Sepolia and ran it against a real deployed v2 name. I picked an existing registered name whose owner holds the admin role (found by scanning `EACRolesChanged` logs on the v2 ETHRegistry `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`: resource `42585474...372224`, owner `0x15CeC1B04A1e7aea7BF561c0d3929Ce3660a6bC9`), impersonated that owner (which is exactly who signs the grant in the real flow, since the migrated name owner is the victim-signer), and sent the grant:

```
hasRoles(resource, SET_RESOLVER_ADMIN, owner)          = true      (owner holds the admin bit)
hasRoles(resource, SET_RESOLVER, ATTACKER2)  [before]  = false
grantRoles(resource, SET_RESOLVER, ATTACKER2) from owner -> status 1 (SUCCESS)   tx 0x432ac0da...06a1
hasRoles(resource, SET_RESOLVER, ATTACKER2)  [after]   = true      (third party now holds resolver-setting authority)
CONTROL grantRoles(resource, ROLE_RENEW, ATTACKER2)    -> REVERT (EACCannotGrantRoles)
```

So the grant of `ROLE_SET_RESOLVER` to an arbitrary third party succeeds; the resolver-hijack branch is measured, not inferred. The control (the same owner cannot grant `ROLE_RENEW`, which it has no admin over) reverts with `EACCannotGrantRoles`, which confirms the EAC is actually enforcing and the success is not a fork artifact. This matches the repo's own e2e note (`seed-managed-name.ts:50-70`: the name owner receives admin over `ROLE_SET_RESOLVER` and `ROLE_SET_SUBREGISTRY` but not `ROLE_RENEW`).

**Chain acceptance verification.**

* Ethical variant used: synthetic addresses, no live victim, no deployed tx.
* What this proves: two measured facts. Tier 1 shows the Manager frontend, driven by the victim's own wallet as signer, constructs an authenticated `grantRoles` that hands `ROLE_SET_RESOLVER` on the victim's migrated name to a third-party address taken from the subgraph and never validated on-chain. Tier 2 shows that same grant lands on the deployed v2 ETHRegistry (Sepolia fork, real registered name, owner-signed): the third party goes from holding no role to holding `ROLE_SET_RESOLVER`, tx status 1.
* What this does NOT prove: I did not run the full migration batch end-to-end in one transaction (register/migrate then grant); I measured the grant step in isolation against an already-registered v2 name whose owner carries the same role set a migrated owner gets (per `seed-managed-name.ts:50-70`), and I impersonated that owner rather than producing its signature. I also did not then call `setResolver` and resolve the name to a live attacker address; that is the standard downstream use of `ROLE_SET_RESOLVER`, which I left unexercised.

**Testing conduct.** Source review, a local run of the app's own functions with throwaway addresses (Tier 1), and a local anvil fork of Sepolia (Tier 2). Every write happened on the local fork; the only calls to the public Sepolia RPC were read-only (`eth_getLogs` to find a real name, `hasRoles`, `codesize`). Nothing was written to live Sepolia and no artifact was left on any real network.

Happy to attach the full runnable script and the diff proof, or to run the fork branch if you want the on-chain landing measured before you rate it.
