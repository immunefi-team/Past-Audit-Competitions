# 89352 w a medium v1 to v2 migration silently replays prior owner resolver records pointing the victim s migrated name at the attacker

**Submitted on Aug 18th 2026 at 18:56:52 UTC by @Griffinf for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89352
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

**Severity claimed: Critical.**

Two in-scope Critical impacts apply, quoted verbatim:

* "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient", first example bullet "resolving or rendering an ENS name to an attacker-controlled address". My proof of concept demonstrates exactly this end state: after a migration through the app's own pipeline, the victim's authoritative V2 resolver returns the attacker's address for `addr(60)`.
* "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer)", first example bullet "a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker". The migration writes attacker-authored resolver records into the victim's new resolver under the victim's own signature.

The program's stated priorities also name this shape: "anything that lets a transaction be built, signed, or attributed with the wrong chain, sender, target address, or arguments". The batch the victim signs carries attacker-authored arguments that no screen in the flow ever showed. Graceful fallback: if the panel discounts the Critical mapping because the hostile values originate with the name's previous owner, the finding still maps to the High impact "Injecting/modifying the static content on the target application without JavaScript (persistent)", second example bullet "Replacing existing text with arbitrary text"; it clears High either way.

**Asset:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager> (migration feature) plus `packages/migration` (imported by it). Verified at the `audit-comp-ready` branch head `1c9b47f18fcddd2e864dfe385c4171061c9811ae`; the program page lists audited revision `63772fd872af472ced58b009499355f3430c2a86`; per the repository's own history it is the fork point (commit `cda79ac`), and the only commits between it and the verified head are a hono dependency bump and its merge, touching none of the cited paths.

## Relationship to published Known Issues

* **WEB-806** ("Subname records not appearing after being migrated") is the nearest-sounding entry and the opposite in data-flow direction: a post-migration read and display gap. This report is a write-side provenance defect plus a verification step that structurally cannot reject the written values. No shared root cause.
* **WEB-390** ("Subnames missing from the migration list") is an enumeration bug in what the list shows, unrelated to what the batch writes.
* **QA-01** and its section preamble carve security consequences back in, verbatim: "However, a demonstrated *security* consequence of one of them (for example, a role combination that grants authority it should not, rather than merely erroring) goes beyond the known breakage and is a new finding." QA-01 concerns role combinations erroring after transfers; this report concerns record content provenance during migration.
* **WEB-1257** ("Cannot set primary name after parent transfer") is a transfer-time detach gap harming the previous owner's own action; unrelated.
* **R3-01 through R3-08** read `packages/transaction-manager` and `packages/smart-account`. The record replay and its verifier live in `apps/manager/src/features/migration/service` and `packages/migration`, outside that deep-read scope.
* No entry concerns record provenance, replay review, or verification circularity. The preamble governs: "Anything **not** listed here is in scope under the normal competition rules".

This report is related to, but has a distinct root cause and fix from, my report "V1 to V2 migration silently grants ROLE\_SET\_RESOLVER on the victim's name to the stale V1 registry controller (the seller of a secondhand name); one attacker transaction then swaps the resolver, and the grant survives resale". That report's defect is the manager classification and grant path (`classifyNames` trusting the stale V1 controller, `buildRoleGrantCalls` minting the grant); this one's is the record replay path (`fetchV1Profiles` sourcing prior-owner content, replayed with no review, certified by a circular `verifyAtomicMigrationBatch`). Fixing either leaves the other fully exploitable. The two share one proof-of-concept file because a single secondhand-name scenario exercises both; the file's second test case, the reclaimed-buyer variant, exercises only this report's defect and is described in the Vulnerability Details and Proof of Concept sections below.

## Description

When a user migrates a name from V1 to V2, the Manager app reads the name's V1 record inventory and live values and replays them into the brand-new, HCA-owned `PermissionedResolver` that becomes the name's authoritative V2 resolver. The core defect is one of write-set provenance and consent: the application chooses attacker-authored historical data as the requested write set without displaying it or obtaining content-level consent, and the replay happens inside one opaque signed batch. The advertised post-state verification cannot compensate, because `verifyAtomicMigrationBatch` compares the chain against the very objects fetched from the hostile source: verification confirms the writes landed as requested; nothing evaluates whether the requested content is trustworthy or shows it to the user. Whoever authored the V1 records, in the realistic case the previous owner of a secondhand name, gets their content certified into the victim's new resolver under the victim's own signature.

## Vulnerability Details

The taint path, all at the audited revision:

{% stepper %}
{% step %}

## Key inventory

`computeMigrationPreflight.ts` obtains the name's V1 profile keys from the v1 subgraph: every text key and coinType ever emitted for the `(resolver, node)` pair, including by prior owners.
{% endstep %}

{% step %}

## Value fetch

`apps/manager/src/features/migration/service/buildMigrationPlan.ts` (lines 68 to 87, `fetchProfilesForNames`) calls `fetchV1Profiles.ts`, which reads the values live from the name's V1 resolver via multicall. No key allowlist, no content check, no display; the only normalization is `cleanResolverTextRecords` (twitter and github key handling).
{% endstep %}

{% step %}

## Replay

`buildAtomicMigrationBatches.ts` (inside `buildNameExecution`; replay encoding at lines 627 to 687) encodes those same `Profile` objects verbatim as `setText`, `setAddr`, `setContenthash`, and `setABI` calldata against the victim's new resolver, and simultaneously as the post-state `verificationExpectations`.
{% endstep %}

{% step %}

## Verification cannot reject the content

Secondary to the write-set choice above, `verifyAtomicMigrationBatch.ts` compares the V2 resolver's post-state to the values fetched in step 2. The expectation set and the written values are the same tainted objects, so verification confirms the writes landed as requested; nothing evaluates whether the requested content is trustworthy or shows it to the user. The verifier can detect a failed write, never hostile content.

The release notes (`apps/manager/docs/HCA_MIGRATION_RELEASE_NOTES.md`, lines 58 to 59) present this verification as the safety property:

> "A name is reported complete only after its v2 owner, resolver, manager roles, resolver roles, and replayed records have been verified on-chain."

That promise is true only against the tainted expectation set.
{% endstep %}

{% step %}

## Zero decision-time disclosure

The plan, including the profiles, is built silently inside `useMigrationGasEstimate.ts` while the user sits on the select screen. The only plan surface any migration component renders is the `MigrationStepDescriptor` union (`buildStepDescriptors.ts`, lines 13 to 31), which carries only `type`, `approvalId`, `count`, `name`, `tokenId`, `index`, and `total`; it structurally cannot represent a record key, value, or address.

I enumerated every component under `apps/manager/src/features/migration/components` and `.../pages`; there is no review screen. The batch step's descriptive copy, verbatim from `WalletConfirmationStepsDialog.tsx` (line 107): "Migrate the selected names and restore their records." Its heading is a name count. The approval step (lines 84 and 87) reads "Approve manager restoration" / "Restore the existing managers for your names." Categories, never content.
{% endstep %}
{% endstepper %}

A gate that does not save it: `classifyNames.resolverStrategyFor` only enters the replay path when the V1 resolver is an allowlisted public resolver (`packages/migration/src/contracts/knownResolvers.ts`), so a hostile custom resolver contract is excluded. But the allowlisted PublicResolver's storage is exactly where a prior owner's records live. A secondary-market sale transfers the BaseRegistrar ERC-721; that transfer touches neither the V1 registry owner nor the resolver's record storage. PublicResolver records are keyed by node and persist across the transfer, and clearing them or bumping the record version takes registry-owner authority, which an un-reclaimed buyer lacks. A buyer who never called `reclaim()` cannot even sanitize the records before migrating; a buyer who did reclaim still inherits every record they do not individually know to clear.

The defect does not depend on any residual authority of the previous owner. My proof of concept includes a reclaimed-buyer variant as a second test case: the buyer calls `reclaim()` on the BaseRegistrar before migrating and becomes the V1 registry owner, so the former owner holds zero current authority anywhere, the migration classifies the name with no manager to restore (`managerAddress` null), and the signed batch contains no role grant of any kind, asserted three ways in-run: no manager-role-grant phase, no manager-role verification expectation, and the `grantRoles` selector absent from every inner call and from the signed outer calldata. The former owner's authored records are still silently re-attested into the buyer's fresh V2 resolver and the verification confirms them.

This finding therefore stands independent of my report "V1 to V2 migration silently grants ROLE\_SET\_RESOLVER on the victim's name to the stale V1 registry controller (the seller of a secondhand name); one attacker transaction then swaps the resolver, and the grant survives resale": fixing the grant leaves this replay fully intact.

Two adjunct observations sharpening the same defect:

* **Stale reuse:** `plan.profiles` is fetched once at preview (gas-estimate) time and reused verbatim at execution (`buildNextAtomicBatch` passes `profiles: plan.profiles`); no re-fetch, no re-confirmation between preview and signature.
* **Undiscoverable records:** the profile UI builds its address rows exclusively from `coinTypeToNameMap` of `@ensdomains/address-encoder` (`apps/manager/src/features/profile/data/records/address.tsx`), and its text rows from a fixed key list. A record under a coinType absent from that map, such as the ENSIP-11 EVM coinType `2147568179` my proof of concept plants, cannot be rendered by the app anywhere. I verified the coinType is absent from the map in both address-encoder versions the repository pins. The migration was the one moment the app could have surfaced it, and instead it certified it.

Intended functionality, conceded up front: the release notes advertise the mechanism ("restores legacy manager access, and replays" records, line 8), and record continuity is a legitimate goal. The defect is that the application chooses prior-owner-authored data as the requested write set with zero decision-time disclosure, the flow structurally cannot show the values it signs, and no content-level consent; the on-chain verification the flow advertises as its safety check then only confirms those requested writes landed, so no step in the pipeline can fail on content. Revocable legacy residue becomes victim-attested, app-certified authoritative state.

## Impact

* **Demonstrated:** after the victim's migration, the authoritative V2 resolver returns the attacker's address for `addr(60)`. Any party that resolves the name and pays the address it returns pays the attacker: the Critical impact "resolving or rendering an ENS name to an attacker-controlled address", realized on-chain through the app's own pipeline.
* **Demonstrated:** the replayed `text(url)` record returns an attacker-controlled URL from the victim's new resolver.
* **Inferred** (labeled as such): the planted hidden-coinType record (`addr(2147568179)`, an ENSIP-11 EVM coinType) would direct any wallet resolving that coinType to the attacker. I did not exercise a third-party wallet against it, so I claim it only as the discoverability argument; `addr(60)` carries the demonstrated payment impact on its own.
* Honest precondition, stated once: the same values already resolved attacker-ward in V1. The deltas the Critical claim rests on:
  * The app builds and has the victim sign a transaction whose arguments are attacker-authored with zero display, the program's own prioritized transaction-construction concern.
  * The replay copies revocable legacy residue into a fresh victim-owned resolver and certifies it, with the success screen saying everything was verified on-chain.
  * No targeting is required, since any seller of a name whose V1 resolver is the allowlisted PublicResolver is pre-positioned, and hidden-coinType or unknown-key records have no rendering surface in the app, verified against both pinned address-encoder versions.

## Link to Proof of Concept

<https://gist.github.com/aussinfosec/2e9b6ef68c81d5c7d6a4ed635421fc22>

## Proof of Concept

Attached: `migration-hostile-replay.poc.test.ts` (the runnable test, two test cases) and `migration-poc-evidence.txt` (one evidence block per passing test case). The first test case's scenario proves this finding and the cross-referenced manager-restoration finding; the second, the reclaimed-buyer variant, proves this finding alone, with no role grant anywhere in the batch. The assertions specific to this report are called out below.

A self-contained reproduction bundle, the runnable test and the evidence file from my passing runs, is available at <https://gist.github.com/aussinfosec/2e9b6ef68c81d5c7d6a4ed635421fc22> (secret gist).

Steps from a clean checkout:

{% stepper %}
{% step %}

## Clone the repository and check out the branch

```bash
git clone https://github.com/immunefi-team/audit-comp-ens && cd audit-comp-ens && git checkout audit-comp-ready
```

Branch head at reporting time: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`; the program-named revision `63772fd872af472ced58b009499355f3430c2a86` is, per the repository's own history, the fork point (commit `cda79ac`), and the only commits between it and this head are a hono dependency bump and its merge, touching none of the cited paths.
{% endstep %}

{% step %}

## Install dependencies

```bash
pnpm install
```

{% endstep %}

{% step %}

## Start a fresh Sepolia fork

In a second terminal, run:

```bash
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --chain-id 11155111 --port 8545
```

The test defaults to `http://127.0.0.1:8545`; override with `POC_RPC_URL` if needed.
{% endstep %}

{% step %}

## Copy the test file

Copy the attached `migration-hostile-replay.poc.test.ts` into:

```
apps/manager/src/features/migration/service/
```

{% endstep %}

{% step %}

## Run the test

```bash
cd apps/manager && ./node_modules/.bin/vitest run src/features/migration/service/migration-hostile-replay.poc.test.ts --no-coverage
```

{% endstep %}
{% endstepper %}

Expected result: `2 passed`; each test case prints its evidence block, and `migration-poc-evidence.txt` in `apps/manager` ends up containing the main scenario's block followed by the reclaimed-buyer variant's block.

What the run establishes for this report:

* **Premigration reserved-entry probe:** before any scaffolding, the test reads the live deployment on the fork: existing V1 labels (`test`, `vitalik`, `ens`, `sepolia`, `name1`, `wallet`) all carry V2 ETHRegistry entries pointing at the deployed v1-mirror resolver `0xae66c62AcAE72098BdAc57d8E8AED53EF000b2Ba`, while an unregistered control label carries none. The test's premigration scaffold mirrors this production state; its entry's resolver field uses the V1 PublicResolver rather than the v1-mirror value, which is immaterial because the migration batch overwrites the resolver. The deployed MigrationHelper and ETHRegistry accepted the batch either way, and the evidence records the live value and the post-migration resolver; the scaffold value is pinned in the test source.
* The attacker registers a V1 name, points it at the allowlisted PublicResolver (`0x8FADE66B79cC9f707aB26799354482EB93a5B7dD`, an entry in `knownResolvers.ts`), and plants `addr(60)`, `addr(2147568179)`, and `text(url)` records. The victim buys the name via a plain ERC-721 `transferFrom` on the BaseRegistrar, the standard marketplace shape.
* The victim migrates through the app's real `buildMigrationPlan`, approval builders, and `atomicBatches[0].outerCall` (`executeByOwner`), submitted with the identical `{from, to, data, value}` fields the app's `buildEOARequest` would send. The only substitution in the pipeline is the wallet transport; every byte of migration calldata is produced by the app's own builders. Subgraph inputs are constructed objects cross-checked on-chain inside the test before use; the V1 registration scaffold produces state byte-identical to a normal registration and the exploit path never uses its privilege.
* **Silent plan:** the serialized step descriptors, the only plan surface the UI renders, contain no record value and no attacker address; asserted in-run.
* **Verification blesses the requested writes:** the app's own `verifyAtomicMigrationBatch` returns confirmed, evidence line `verifier: confirmed, all 10 expectations satisfied`, despite every replayed value being hostile; it verifies against the same fetched objects, so it confirms the writes landed as requested and evaluates nothing about the content.
* **End state:** from the evidence file, the victim's fresh `PermissionedResolver` at `0xe9AFa3e8B2722156445a0cB991877A974e782589` returns:
  * `addr(60) = 0xdbbf701b1f346e786c6c7bbe45e0914c9ac72707` (the attacker)
  * `addr(2147568179)` = the attacker
  * `text(url) = https://attacker-payments.example`
* **Reclaimed-buyer variant (second test case):** identical scaffold, except the buyer calls `reclaim()` on the BaseRegistrar before migrating, and the constructed subgraph state mirrors the reclaim, registry owner = buyer, cross-checked on-chain in-run. Expected assertions, all made in-run:
  * The app's own classification returns `managerAddress` null.
  * Its approval planner (`planMigrationApprovals`) schedules only the token approval and no eth-registry operator approval.
  * The built batch contains no manager-role-grant phase, no manager-role verification expectation, and the `grantRoles` selector (`0x7c300586`) appears nowhere in the inner calls or in the signed outer calldata.
  * After execution the former owner holds no role on the name, `hasRoles` returns false.
  * The verifier still returns confirmed, evidence line `verifier: confirmed, all 9 expectations satisfied`.
  * The buyer's fresh `PermissionedResolver` at `0x52e9CA8882d3B82d8d1c417b7FbCf42003769d23` still returns:
    * `addr(60) = 0x7df2fc9796fe34c69816e56eb628c3d047a44a52` (the former owner)
    * The hidden-coinType address = the former owner
    * `text(url) = https://attacker-payments.example`

The evidence file's second block records these values.

Evidence gaps, disclosed rather than papered over: I did not run the flow in a browser, the end-to-end snapshot image is access-gated, so the nothing-is-shown claim is source-confirmed at the type level plus a full component enumeration plus the in-run step-descriptor assertion, not screenshot-demonstrated. The hidden-coinType payment impact is Inferred as labeled; the v1 subgraph was not run live, with constructed inputs cross-checked on-chain instead. The on-chain impact itself is demonstrated through the app's own pipeline.
