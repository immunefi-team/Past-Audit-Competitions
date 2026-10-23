# 90755 w a medium cached v1 manager authority is restored as a persistent v2 resolver role during migration

**Submitted on Aug 26th 2026 at 22:44:42 UTC by @Catherin88387 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90755
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The ENS Manager V1-to-V2 migration can restore resolver authority to a revoked former V1 manager. The app treats indexed V1 ownership data as fresh for five minutes and uses the cached owner as the recipient of a V2 `ROLE_SET_RESOLVER` grant without checking the current V1 Registry owner. If exploited, the former manager can replace or brick the migrated name’s resolver even after the migration’s temporary HCA approval has been removed.

## Vulnerability Details

The assessed version is ENS Manager commit `cda79acaad59711b943fc68207ebb3f1d0ff8596`.

In the example below, Alice is the legitimate registrant/current owner and Mallory is a former manager whom Alice has revoked.

The migration list is fetched from the V1 indexer and considered fresh for five minutes:

```typescript
resultQueryOptions({
  queryKey: qk('migration', 'v1_names', {
    address: address?.toLowerCase(),
  }),
  queryFn:
    enabled && address ? () => getV1NamesForAddress(address) : skipToken,
  staleTime: 5 * 60 * 1000,
})
```

`classifyName()` treats a V1 Registry owner that differs from the Base Registrar token holder as the legacy manager:

```typescript
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

That address becomes the recipient of a V2 resolver-role grant:

```typescript
return {
  to: V2_CONTRACTS.ETHRegistry,
  data: encodeFunctionData({
    abi: ETH_REGISTRY_V2_ABI,
    functionName: 'grantRoles',
    args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
  }),
  value: 0n,
}
```

The application performs a live ownership preflight, but for unwrapped names it reads `BaseRegistrar.ownerOf(labelhash)`. That confirms Alice still holds the registration NFT; it does not check the current V1 `ENSRegistry.owner(node)` and therefore cannot detect that Mallory has been revoked:

```typescript
name.tokenType === 'unwrapped'
  ? {
      address: BASE_REGISTRAR,
      abi: BASE_REGISTRAR_ABI,
      functionName: 'ownerOf',
      args: [BigInt(name.domain.labelhash)],
    }
```

The exploitable sequence is:

{% stepper %}
{% step %}

## Alice holds the Base Registrar token

Alice holds the Base Registrar token for `alice.eth`, while Mallory is initially the differing V1 Registry owner/manager.
{% endstep %}

{% step %}

## Alice opens the migration flow

The app retrieves `registrant=Alice` and `owner=Mallory`, then considers this response fresh for five minutes.
{% endstep %}

{% step %}

## Alice revokes Mallory

Alice revokes Mallory by returning the V1 Registry node to herself. Alice remains the Base Registrar token holder.
{% endstep %}

{% step %}

## Alice continues the migration

Alice continues the normal migration before the cached query is refreshed.
{% endstep %}

{% step %}

## The live preflight confirms token ownership

The live preflight confirms Alice still owns the registration token, but never checks the current V1 Registry owner.
{% endstep %}

{% step %}

## The cached manager address is used

The cached address is retained as `managerAddress`, and the HCA batch migrates the name before granting Mallory V2 `ROLE_SET_RESOLVER`.
{% endstep %}

{% step %}

## Post-migration verification succeeds

Post-migration verification checks that Mallory received the planned role and therefore treats the unintended grant as success.
{% endstep %}

{% step %}

## Cleanup removes temporary approval only

Cleanup removes only the HCA’s temporary operator approval. Mallory’s directly stored role remains.
{% endstep %}

{% step %}

## Mallory replaces the resolver

Mallory calls `setResolver()` and replaces the migrated name’s resolver.
{% endstep %}
{% endstepper %}

For the synthetic `alice.eth` fixture, production application code generates:

```
target   = 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
resource = 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb800000000
role     = 0x1000000 (ROLE_SET_RESOLVER)
account  = 0x00000000000000000000000000000000000000B1
```

The attached app PoC imports the production classifier and migration planner. It demonstrates that the valid-current-manager and stale-revoked-manager cases generate byte-identical grants to Mallory, while a fresh snapshot after revocation generates no manager grant.

The separate Foundry PoC submits the exact generated grant to the production `PermissionedRegistry` implementation in a local deployment. After Alice removes the HCA approval, the HCA can no longer grant roles, but Mallory’s direct role persists and allows Mallory to replace the resolver. The no-grant and explicit-revocation controls both reject Mallory.

Observed results were:

```
Vitest: 1/1 passed
TypeScript and Biome checks: passed
Foundry: 3/3 passed
forge fmt: passed
```

No wallet, RPC endpoint, public deployment or chain transaction was used. The migration route is feature-flagged, and current production enablement is not asserted.

## Impact Details

This matches the two selected Critical impacts in the [ENS competition scope](https://immunefi.com/audit-competition/audit-competition-ens/scope/):

* causing the application to construct or pre-fill an incorrect transaction detail that the user then approves; and
* hijacking or bricking an ENS name during a Manager V1-to-V2 migration, including assigning its resolver to an attacker.

The unintended grant gives Mallory persistent permission to replace the migrated name’s resolver. Mallory can point the name at a malicious resolver that returns attacker-controlled address records or content, or at a non-functional resolver that prevents the name from resolving correctly. This could redirect payments or application traffic that relies on the ENS name, although the PoC conservatively stops at demonstrating the unauthorized resolver change and does not claim or quantify downstream fund theft.

The authority remains after the HCA’s temporary approval is removed. Alice would need to discover and explicitly revoke Mallory’s role to restore the intended authorization state.

The attack requires Mallory to have previously been the V1 manager, Alice to load the migration list before revoking Mallory, and Alice to proceed with the ordinary migration within the five-minute cache window without a refetch. It does not require Mallory to compromise Alice’s wallet, take over the HCA, exploit the V2 contracts, corrupt the indexer or obtain a hidden signature.

## References

* [ENS Audit Competition scope](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [Assessed ENS Manager snapshot](https://github.com/immunefi-team/audit-comp-ens/tree/cda79acaad59711b943fc68207ebb3f1d0ff8596)
* [`useV1Names.ts` — five-minute freshness period](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L23)
* [`classifyNames.ts` — cached owner becomes `managerAddress`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L148-L186)
* [`buildRoleGrantCalls.ts` — manager address becomes the grant recipient](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26)
* [`preflightChecks.ts` — checks token ownership, not V1 Registry ownership](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/preflightChecks.ts#L47-L101)
* [`buildAtomicMigrationBatches.ts` — migration followed by manager-role grant](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L625)
* [Secret Gist containing the report and portable PoCs](https://gist.github.com/jasonheron/810dbf2cd3d55df1eea56bbef900e1cf)

## Link to Proof of Concept

<https://gist.github.com/jasonheron/810dbf2cd3d55df1eea56bbef900e1cf>

## Proof of Concept

## Safety and target

The secret Gist supplied in Immunefi's PoC Link field contains two portable, local-only harnesses:

* an app test that imports the frozen ENS Manager classifier and migration planner; and
* a contract consequence test that submits the app-generated role grant to the production `PermissionedRegistry` implementation in a local Foundry deployment.

The app is pinned to competition commit `cda79acaad59711b943fc68207ebb3f1d0ff8596`. The contract sink is pinned to `ensdomains/contracts-v2` commit `8d1c89350729f87b3968cf41ef3d683951444a44`.

Both tests use synthetic state and addresses. They do not use a wallet, key, RPC endpoint, mainnet fork, testnet, production service, or broadcast transaction. The contract half proves only the effect of the scoped Manager app's wrong-recipient call; it does not allege a contract vulnerability.

In the test labels, **Alice** means the victim/current name owner and **Mallory** means the revoked former manager. These are conventional placeholder names, not real users.

## Run

Download every file from the secret Gist into one directory, then run:

```sh
chmod +x run.sh
./run.sh all
```

The modes can also be run separately:

```sh
./run.sh app
./run.sh contract
```

Prerequisites are Git, Bash, an LTS Node.js runtime with Corepack, and Foundry with Solidity `0.8.34` available for offline compilation. The runner creates disposable, exact-revision checkouts under the system temporary directory. The first run downloads the two public repositories and their pinned dependencies.

## App proof: stale authority becomes the grant argument

The Vitest harness imports the production `classifyName` and `buildMigrationPlan` functions from the assessed Manager revision. It does not reproduce the vulnerable classifier or transaction builder.

It runs three cases against the same synthetic `alice.eth` name:

{% stepper %}
{% step %}

## Current-manager baseline

The cached V1 registry owner and current V1 owner are both Mallory. A manager-restoration grant to Mallory is expected.
{% endstep %}

{% step %}

## Stale-revoked positive case

The cached owner is Mallory, but the current owner is Alice because Mallory was revoked after the list loaded. The planner still creates the same grant to Mallory.
{% endstep %}

{% step %}

## Fresh negative control

The cached and current owner are both Alice. No manager grant or manager-role verification expectation is created.
{% endstep %}
{% endstepper %}

For the stale-revoked case, the harness proves:

* the production planner performs zero preview-time chain reads that could bind the recipient to the current V1 registry owner;
* `classifiedManager` and the post-state verification account remain Mallory;
* the outer HCA `executeByOwner` call contains the exact production `grantRoles` call; and
* the stale positive case is byte-identical to the valid current-manager baseline, while the fresh control omits the grant.

The decoded stale grant is:

```
target   = 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
resource = 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb800000000
role     = 0x1000000
account  = 0x00000000000000000000000000000000000000B1
calldata = 0x7c3005869c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb800000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000000000b1
```

## Contract proof: the unintended role persists and changes the resolver

The Foundry harness imports the production V2 registry implementation at the pinned commit. It starts with Alice as the current owner and Mallory unauthorized, then:

{% stepper %}
{% step %}

## Approve the synthetic HCA

Alice temporarily approves the synthetic HCA, matching the app's migration flow.
{% endstep %}

{% step %}

## Submit the grant

The HCA submits the exact app-generated grant to Mallory.
{% endstep %}

{% step %}

## Remove HCA approval

Alice removes the HCA approval.
{% endstep %}

{% step %}

## Verify the role persists

The harness proves the HCA can no longer grant roles, while Mallory's directly stored `ROLE_SET_RESOLVER` remains.
{% endstep %}

{% step %}

## Change the resolver

Mallory calls `setResolver` and changes the resolver from `0x...00D1` to `0x...00E1`.
{% endstep %}
{% endstepper %}

Two controls prove causality:

* without the stale grant, Mallory's resolver call reverts and the original resolver remains; and
* explicitly revoking Mallory's role before use also makes the resolver call revert.

## Observed result

The packaged tests were run locally with these results:

```
App:
Test Files  1 passed (1)
Tests       1 passed (1)
isolated TypeScript check: exit 0
Biome focused check: exit 0

Contract:
3 tests passed; 0 failed; 0 skipped
forge fmt --check: exit 0
```

The positive and negative results are preserved in `app-vitest.txt` and `contract-forge.txt`.

## What the PoC establishes

The app—not the V2 registry—is the vulnerable asset. A cached V1 manager address is promoted into a victim-approved V2 role grant without checking the current V1 registry owner. The official contract sink then shows that the resulting direct resolver role survives removal of the HCA's temporary approval and lets the revoked former manager change the migrated name's resolver.

The migration route is feature-flagged, and this PoC does not claim that the flag is currently enabled in production. It establishes the concrete authority consequence in the frozen Manager flow expressly placed in scope by the ENS audit competition.
