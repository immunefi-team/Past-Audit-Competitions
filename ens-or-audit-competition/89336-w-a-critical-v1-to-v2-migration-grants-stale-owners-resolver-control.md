# 89336 w a critical v1 to v2 migration grants stale owners resolver control

**Submitted on Aug 18th 2026 at 17:23:54 UTC by @Griffinf for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89336
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

**Severity claimed: Critical.**

This maps verbatim to the in-scope Critical impact "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer)", first example: "a registration or migration that assigns the name, its ownership, registry, **or resolver to an attacker**". The migration calls `grantRoles(canonicalId, ROLE_SET_RESOLVER, attacker)` on the V2 ETHRegistry inside the victim's migration batch, and I demonstrate the attacker then taking the resolver with a single plain transaction. The follow-on matches the Critical impact "resolving or rendering an ENS name to an attacker-controlled address".

**Asset:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager> (migration feature) plus `packages/migration` (imported by it). Verified at the `audit-comp-ready` branch head `1c9b47f18fcddd2e864dfe385c4171061c9811ae`; the program page lists audited revision `63772fd872af472ced58b009499355f3430c2a86`; per the repository's own history it is the fork point (commit `cda79ac`), and the only commits between it and the verified head are a hono dependency bump and its merge, touching none of the cited paths.

## Relationship to published Known Issues

* **QA-01 / WEB-1257 / WEB-1305** cover role combinations that cause **errors** after transfers. This finding is the opposite: a role grant that succeeds and hands out authority. The Known Issues document carves this direction out as new, verbatim: "a demonstrated *security* consequence of one of them (for example, a role combination that grants authority it should not, rather than merely erroring) goes beyond the known breakage and is a new finding."
* **WEB-806** is a post-migration display gap (migrated subname records not visible on the profile). This finding is a write-side authority grant executed during migration. No shared root cause.
* **R3-01 through R3-08** cover `packages/transaction-manager` and `packages/smart-account`. The migration service lives in `apps/manager/src/features/migration` and `packages/migration`, outside that audit's stated scope.
* No entry mentions manager restoration, role grants at migration, or trusting the stale V1 controller.

This report is related to, but has a distinct root cause and fix from, my report "V1 to V2 migration silently replays prior-owner resolver records into the victim's new authoritative resolver under the victim's signature; the advertised on-chain verification is circular and cannot reject hostile content". That report is about record content provenance in `fetchV1Profiles` / `buildProfileReplayCalls` and the circular `verifyAtomicMigrationBatch`; this one is about the trust decision in `classifyNames` and the grant in `buildRoleGrantCalls`. Different code sites, different patches; fixing either leaves the other fully exploitable. The two share one proof-of-concept file because a single scenario exercises both.

## Description

For unwrapped V1 names, the migration classifier treats the V1 registry owner (the "controller") as an intentional manager whenever it differs from the registrant, and re-creates that party's authority in the V2 role system as a durable `ROLE_SET_RESOLVER` grant, without the migration flow ever showing the user the address it is empowering.

The V1 registry-owner field cannot distinguish an appointed manager from marketplace residue. A secondary-market sale of an unwrapped .eth name transfers the BaseRegistrar ERC-721; the token transfer does not itself update the V1 registry owner, which remains the **seller** unless the sale or a subsequent transaction separately calls `reclaim()`. For an un-reclaimed secondhand name, "the existing manager" is the seller.

Manager restoration is a documented mechanism: the release notes (`apps/manager/docs/HCA_MIGRATION_RELEASE_NOTES.md`) advertise that the migration "restores legacy manager access, and replays records", and preserving a deliberate hot-wallet manager is legitimate. The defect is the combination of:

1. **Provenance:** the restoration source field (V1 registry owner) cannot tell consent from residue, and an un-reclaimed marketplace purchase leaves an unwrapped name in exactly this registrant-differs-from-controller state.
2. **Zero disclosure at the decision moment:** the flow never displays the grantee address. The copy for the grant's approval step is "Approve manager restoration" / "Restore the existing managers for your names." (`WalletConfirmationStepsDialog.tsx` lines 84 to 88) and "Approve manager restoration in your wallet" (`GameStep.tsx` line 137); the batch that executes the grant reads "Migrate the selected names and restore their records." (`WalletConfirmationStepsDialog.tsx` line 107); the flow's remaining copy is headings and name counts. Categories, never content. No opt-out, no review step, and the plan type the UI renders structurally cannot carry the address (see below).
3. **Escalation, not preservation:** in V1 the stale controller's power is revocable residue that dies with one `reclaim()`, and migration is precisely the moment it would otherwise be extinguished. The app instead re-mints it as a durable V2 role grant that, as demonstrated, survives resale of the V2 name.
4. **Certified as verified:** the app's post-state verifier confirms the grant as part of "success", and the release notes market that verification as the safety property ("A name is reported complete only after its v2 owner, resolver, manager roles, resolver roles, and replayed records have been verified on-chain").

## Vulnerability Details

Paths relative to the repository root at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

**The grant is created authority, not preserved authority.** For an unwrapped V1 name, migration is the moment the stale controller relationship would naturally end: the batch hands the name to the MigrationHelper and the name's authoritative state moves to V2, where the seller starts with nothing. Had the batch stopped at `MigrationHelper.migrate(...)`, the stale V1 controller would hold no power over the V2 name. The `ROLE_SET_RESOLVER` grant is a separate call the application itself appends to the same batch: the app is not carrying an existing authority forward, it is manufacturing a new, durable V2 authority for an address whose V1 power was revocable residue about to be extinguished. Every detail below should be read against that baseline.

**Classification.** `packages/migration/src/service/classifyNames.ts` lines 162 to 167 (re-exported through `apps/manager/src/features/migration/service/classifyNames.ts`):

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

Unconditional: any unwrapped name whose registry owner differs from the registrant gets that owner as `managerAddress`.

**The grant.** `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts` lines 9 to 27:

```ts
const ROLE_SET_RESOLVER = 1n << 24n
...
return {
  to: V2_CONTRACTS.ETHRegistry,
  data: encodeFunctionData({
    abi: ETH_REGISTRY_V2_ABI,
    functionName: 'grantRoles',
    args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
  }),
  ...
}
```

`buildAtomicMigrationBatches.ts` lines 608 to 625 places this call inside the same silent atomic batch as the migration and appends a `manager-role` verification expectation, so `verifyAtomicMigrationBatch.ts` confirms the grant as part of reporting the migration complete.

**Nondisclosure is structural.** The only plan surface any migration component renders is `stepDescriptors`. The `MigrationStepDescriptor` union (`apps/manager/src/features/migration/service/buildStepDescriptors.ts` lines 13 to 31) carries only `type`, `approvalId`, `count`, `name`, `tokenId`, `index`, and `total`; it cannot represent an address. I enumerated every component under `apps/manager/src/features/migration/components/` and `.../pages/`; none reads `plan.classified[].managerAddress`. The proof of concept asserts at runtime that the serialized `stepDescriptors` contain no attacker address.

One narrowing, stated precisely: the grantee address is never shown anywhere **in the migration flow**, the consent moment. The Explorer app's registry roles table (`apps/portal/src/features/registry/components/v2/NameRegistryRolesOverviewTable.tsx`) can display the grant after the fact, on an advanced page of a different application, to a user who navigates there unprompted after being told everything was verified. The impact wording does not require permanent concealment; the assignment itself is the impact.

**The consent conversion.** The grant is executed by the victim's Hybrid Custody Account using a temporary ETHRegistry operator approval signed under the generic prompt quoted above. Category-level consent ("restore managers") becomes a specific authority transfer to an address the app never displayed.

## Impact

The attacker's playbook is to be a seller:

{% stepper %}
{% step %}

### Attacker sells an unwrapped V1 .eth name

Attacker sells an unwrapped V1 .eth name through a marketplace or transfer path that transfers the BaseRegistrar ERC-721 without separately calling `reclaim()`. The ERC-721 moves; the registry controller (attacker) does not. No targeting needed; each such sale leaves the attacker as registry controller.
{% endstep %}

{% step %}

### Buyer migrates the name

The buyer, prompted by the app's own upgrade banners, migrates (the migration flow and its banners ship behind a feature flag, `isFeatureEnabled(POSTHOG_FEATURE_FLAGS.MIGRATION)`; this report is evaluated at the scoped code and makes no claim about the flag's current production state). The batch they sign grants `ROLE_SET_RESOLVER` on their new V2 name to the attacker. The success screen reports the migration verified.
{% endstep %}

{% step %}

### Attacker takes over resolution

At any later time, with no victim interaction, the attacker sends one plain transaction, `ETHRegistry.setResolver(canonicalId, attackerResolver)`. The registry then hands resolution of the name to a resolver the attacker controls, so address, text, and contenthash lookups return attacker-chosen values; a payment sent to the address the name resolves to goes to the attacker.
{% endstep %}

{% step %}

### Grant persists after resale

The grant survives resale of the V2 name (demonstrated), so downstream owners who never touched V1 inherit an attacker authority no flow ever disclosed to them.
{% endstep %}
{% endstepper %}

Honest flag: pre-migration, the stale controller could already manipulate V1 resolution of the name. The delta is that migration was the moment that exposure would have died, and the app instead converts it into victim-signed, app-verified, durable, resale-surviving V2 authority. That is "a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker".

## Link to Proof of Concept

<https://gist.github.com/aussinfosec/7525cfc89fe1b677d977b085649562be>

## Proof of Concept

One test file proves this finding end to end together with the record-replay finding (shared scenario, distinct assertions). It drives the app's real pipeline: `buildMigrationPlan`, `buildMigrationApprovalCall`, the atomic batch `outerCall` (`StandaloneHCA.executeByOwner`), and the app's own `verifyAtomicMigrationBatch`, against the real deployed Sepolia contracts on a fork.

A self-contained reproduction bundle (the runnable test and the evidence file from my passing run) is available at <https://gist.github.com/aussinfosec/7525cfc89fe1b677d977b085649562be> (secret gist).

Steps from a clean checkout:

{% stepper %}
{% step %}

### Clone the repository and check out the audited head

```bash
git clone https://github.com/immunefi-team/audit-comp-ens && cd audit-comp-ens && git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
```

The `audit-comp-ready` head; the program page's audited revision is `63772fd872af472ced58b009499355f3430c2a86`.
{% endstep %}

{% step %}

### Install dependencies

```bash
pnpm install
```

{% endstep %}

{% step %}

### Start a fresh Sepolia fork

```bash
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --chain-id 11155111 --port 8545
```

Any Sepolia RPC works; on another port, set `POC_RPC_URL=http://127.0.0.1:<port>`.
{% endstep %}

{% step %}

### Copy the proof-of-concept test

Copy the attached `migration-hostile-replay.poc.test.ts` into `apps/manager/src/features/migration/service/`.
{% endstep %}

{% step %}

### Run the test

```bash
cd apps/manager && ./node_modules/.bin/vitest run src/features/migration/service/migration-hostile-replay.poc.test.ts --no-coverage
```

{% endstep %}
{% endstepper %}

Expected result: `1 passed`, plus an evidence file `migration-poc-evidence.txt` in `apps/manager/`. The attached evidence file is from my run (test name `migpoc-1787072975060.eth`). Assertions relevant to this finding:

* Pre-seeding probe: on the live deployment, long-standing V1 labels (`test`, `vitalik`, `ens`, `sepolia`, `name1`, `wallet`) all carry V2 ETHRegistry (`0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`) entries pointing at the deployed v1-mirror resolver `0xae66c62AcAE72098BdAc57d8E8AED53EF000b2Ba`, while an unregistered control label carries none. The test's reserved-entry scaffold mirrors this production state for the fresh test name (the scaffold's resolver field value differs, which is immaterial because the migration batch overwrites the resolver; the live value and the post-migration resolver are in the evidence, and the scaffold value is pinned in the test source).
* Scenario fidelity, checked on-chain inside the test: ERC-721 registrant is the victim, registry controller is the attacker, resolver is the allowlisted PublicResolver. This is the state a marketplace sale without `reclaim()` produces.
* The plan schedules manager restoration for the attacker (`plan.classified[0].managerAddress == attacker`), and the serialized `stepDescriptors` (the only plan surface the UI renders) contain neither the attacker address nor any record value.
* After the victim's batch executes: `ETHRegistry.hasRoles(canonicalId, ROLE_SET_RESOLVER, attacker) == true`, and the app's verifier returns confirmed with all ten expectations satisfied.
* The attacker, with no approvals and no victim involvement, deploys a minimal resolver contract that answers `addr(node)` with the attacker's address, then from a plain externally owned account calls `ETHRegistry.setResolver(canonicalId, <attacker resolver>)`. `getResolver` returns the attacker's contract, and the test asserts that an `addr(node)` lookup through the name's current resolver now returns the attacker's address: resolution takeover demonstrated end to end, not merely a pointer swap.
* The victim resells the V2 name (ERC-1155 `safeTransferFrom` of the live versioned token id to a second buyer). After the transfer the new owner holds the name and the attacker **still** holds `ROLE_SET_RESOLVER`.

Substitutions, stated in the test header: the wallet transport is viem's `sendTransaction` of the identical `{from, to, data, value}` fields that `migrationService.buildEOARequest` (lines 210 to 227 of `migrationService.ts`) submits, so the calldata is byte-identical; the V1 registration scaffold produces state byte-identical to a normal registration and the exploit path never uses its privilege; the subgraph input objects are cross-checked on-chain inside the test before use.

## Recommended Fix

Any of the following breaks the attack; the first is the most direct:

1. Display the manager address in the migration review step and require an explicit opt-in per name before including a `grantRoles` call in the batch.
2. Refuse restoration when the V1 controller predates the current registrant's acquisition of the name (BaseRegistrar transfer history distinguishes residue from appointment), or at minimum when the controller was never re-set after the last token transfer.
3. Default manager restoration to off for names whose registrant and controller differ, surfacing it as an advanced choice.

Also surface existing role grants prominently in the Manager app after migration so a grant that does slip through is discoverable where the owner looks.
