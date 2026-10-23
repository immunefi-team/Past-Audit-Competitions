# 90647 w a critical v1 v2 migration silently grants a third party address permanent in practice role set resolver over the user s migrated name with the grantee never displayed and no way to re&#x20;

**Submitted on Aug 26th 2026 at 06:49:43 UTC by @promael for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90647
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

When migrating an unwrapped `.eth` 2LD, the Manager reads the v1 ENSRegistry `owner()` ("manager"/`reclaim` controller) from the subgraph and, **only when that address differs from the registrant**, appends `ETHRegistry.grantRoles(resource, ROLE_SET_RESOLVER, managerAddress)` to the atomic migration batch. By construction the grantee is therefore an address the migrating user does **not** control.

This is deliberate "manager restoration" code, and preserving a v1 relationship is a reasonable intent. The security problem is everything around it: the grantee address is **never rendered anywhere in the app**, the consent copy the user signs says only *"Approve manager restoration — Restore the existing managers for your names"* with no address and no statement of what power is transferred, and the app contains **zero** `revokeRoles` call sites and no roles UI, so there is no in-app way to see or undo the grant. The role lets the holder repoint the name's resolver at will, and therefore every address and text record the name resolves to.

The realistic path to a hostile manager is the well-known ENS v1 footgun: a seller calls `ENSRegistry.setOwner(node, attacker)` before transferring the BaseRegistrar NFT, and the marketplace transfer does not reset it. In v1 the buyer's cure is a single well-documented `reclaim()`. After migration the same adversary holds a first-class v2 registry role that the app never surfaces.

## Vulnerability Details

### 1. The grantee is, by construction, not the migrating user

`packages/migration/src/service/classifyNames.ts:150,162-167`

```ts
// (:150) the classifier already requires the connected user to be the registrant
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

`managerAddress` is non-null exactly when the v1 registry owner is somebody other than the user performing the migration.

### 2. It is encoded into a role grant

`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:10-26`

```ts
const ROLE_SET_RESOLVER = 1n << 24n
...
to: V2_CONTRACTS.ETHRegistry,
data: encodeFunctionData({
  abi: ETH_REGISTRY_V2_ABI,
  functionName: 'grantRoles',
  args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
})
```

### 3. It is appended unconditionally to the atomic batch

Alongside the migration that sets the v2 owner to the user's wallet — `buildAtomicMigrationBatches.ts:486-491` (`owner: wallet`) and `:608-624` (`if (classified.managerAddress)`), ordered at `:770-780`.

### 4. Success is *defined* as the third party holding the role

`apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:337-353` — the post-migration verifier requires `hasRoles(resource, ROLE_SET_RESOLVER, managerAddress)` to be true; the migration is not considered successful otherwise.

### 5. The user is never told, and cannot undo it in the app

* `grep -rn "managerAddress" apps/manager --include='*.tsx'` → no production hit (test fixture only). The address is never rendered.
* The signed step is labelled only *"Approve manager restoration / Restore the existing managers for your names"* (`components/WalletConfirmationStepsDialog.tsx:84-87`, `components/GameStep.tsx:137`).
* `grep -rn "revokeRoles" apps/manager/src` → **no results**. There is no roles UI and no revocation path.

## Steps to reproduce

{% stepper %}
{% step %}

## Set the v1 registry owner

Attacker owns `alice.eth` (unwrapped) and calls `ENSRegistry.setOwner(namehash('alice.eth'), ATTACKER)`.
{% endstep %}

{% step %}

## Transfer the BaseRegistrar NFT

Attacker sells/transfers the BaseRegistrar NFT to the victim. Subgraph now shows `registrant = VICTIM`, `owner = ATTACKER`.
{% endstep %}

{% step %}

## Open Migration

Victim opens ENS Manager → Migration. `alice.eth` is listed as eligible; nothing in the UI mentions `ATTACKER`.
{% endstep %}

{% step %}

## Approve manager restoration

Victim signs the extra "manager restoration" approval.
{% endstep %}

{% step %}

## Migrate the name

The atomic batch migrates the name to `VICTIM` **and** executes `grantRoles(resource(alice), ROLE_SET_RESOLVER, ATTACKER)`.
{% endstep %}

{% step %}

## Repoint the resolver

`ATTACKER` calls `setResolver` on the v2 name and repoints every address/text record. Funds sent to `alice.eth` go to the attacker.
{% endstep %}
{% endstepper %}

## Recommended fix

Display the manager address and the exact capability being granted in the confirmation step, make the grant opt-in per name rather than mandatory, and add a revoke action (the owner holds the admin bit, so revocation is possible — see below). At minimum, surface the outstanding grant in the name's settings after migration.

## Impact Details

An address the user does not control, and is never shown, gains the ability to repoint the resolver of the user's freshly-migrated name — and therefore to redirect every address the name resolves to. Funds sent to the name go to the attacker.

Mapped to the program's impact list, this is the migration half of *"Hijacking … an ENS name during a Manager flow (registration, v1→v2 migration, or transfer), such as: a registration or migration that assigns the name, its ownership, registry, or **resolver** to an attacker."*

### Honest severity note — why I am filing this as Medium and not Critical/High

I checked whether the grant is recoverable, and it is:

* `REGISTRATION_ROLE_BITMAP` (`contracts-v2 contracts/src/registrar/ETHRegistrar.sol:18-23`) includes `ROLE_SET_RESOLVER_ADMIN`, and `UnlockedMigrationController.sol:163` registers the migrated name to the user with exactly that bitmap.
* Holding the admin bit is what authorises revocation in the EAC model, so the victim **can** call `revokeRoles(resource, ROLE_SET_RESOLVER, attacker)` directly against the registry.

So the grant is **not** cryptographically permanent — it is permanent *within the application*, which offers neither visibility nor a revoke path. Additionally, the hostile-manager condition pre-exists in v1, where the same address already has resolver power; the delta this bug creates is that the v1 cure (`reclaim()`, widely documented) is replaced by a v2 cure (`revokeRoles` via a direct contract call) that the app never mentions and most users will never find.

That combination — an undisclosed grant of resolver control to a non-owner, made during a flow the user understands as "migrate my name", with no in-app remedy — is a real consent and recoverability defect, but I do not think it supports a Critical rating.

### Second, lesser issue in the same code path

The grant is executed by the HCA rather than by the name owner. If the v2 `ETHRegistry` rejects that caller, the entire atomic batch reverts, which would mean permanent migration failure for every name in this class. I could not determine which on-chain outcome occurs because the v2 contracts are not part of this repository; I am flagging both possibilities rather than asserting one.

## References

* `packages/migration/src/service/classifyNames.ts:150, 162-167` — grantee is by construction not the migrating user
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:10-26` — the `grantRoles` call
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:486-491, 608-624, 770-780` — owner set to wallet; grant appended and ordered
* `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:337-353` — success requires the third party to hold the role
* `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:84-87`, `components/GameStep.tsx:137` — the entire consent copy
* Corroborating repo tests: `service/buildRoleGrantCalls.test.ts:12-31`, `packages/migration/src/service/classifyNames.test.ts:105-117`
* Recoverability: `contracts-v2 contracts/src/registrar/ETHRegistrar.sol:18-23`, `contracts/src/migration/UnlockedMigrationController.sol:163`

## Dedup

Not in the disclosed known-issues set (SEC-MGR-003/008/010/011, EXP-INPUT-003/005/008/009, R2-01/02/04/05, EXP-GAP-006, R3-\*, WEB-1249, WEB-1257).

**Triage risk I want to disclose up front:** the code is explicitly labelled "manager restoration", so the grant itself is clearly intentional. My report is not that restoring a manager is wrong, but that:

* the grantee is by construction an address other than the migrating owner,
* it is never displayed and its capability is never stated in the consent step, and
* the app provides no way to inspect or revoke it afterwards.

## Link to Proof of Concept

<https://gist.github.com/Kyenghwan-Hwang/afc568df725864c855ad2cf65d98f216>

## Proof of Concept

`apps/manager/src/features/migration/service/POC_managerRoleGrant.test.ts` — the only file added; no application source modified.

```bash
cd apps/manager && npx vitest run src/features/migration/service/POC_managerRoleGrant.test.ts
# Test Files 1 passed (1) / Tests 2 passed (2)
```

Starting from a realistic v1-subgraph `V1Domain` (`registrant = VICTIM`, `owner = ATTACKER`), the test runs the **real** `classifyNames` → `buildAtomicMigrationBatches` pipeline, decodes the emitted calldata, and asserts:

1. the batch contains `grantRoles(resource, ROLE_SET_RESOLVER, ATTACKER)`;
2. the `manager-role` verification expectation targets `ATTACKER`;
3. the v2 name owner is `VICTIM` — i.e. the victim owns the name while the attacker holds resolver control.

A second test is the negative control: when the v1 manager equals the registrant (the normal case), zero role grants are emitted.
