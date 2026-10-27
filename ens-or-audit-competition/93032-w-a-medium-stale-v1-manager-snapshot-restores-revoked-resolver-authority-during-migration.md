# 93032 w a medium stale v1 manager snapshot restores revoked resolver authority during migration

**Submitted on Sep 14th 2026 at 01:35:30 UTC by @undi for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93032
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The Manager app can restore V2 resolver authority to a former V1 manager after the registrant has revoked that address. When the migration preview is created, the app stores the V1 Registry owner as `managerAddress`. If the BaseRegistrar registrant subsequently calls `reclaim` and becomes the current V1 Registry owner before signing, the live migration checks do not reread the V1 Registry owner. The owner-signed HCA batch therefore still grants the former manager `ROLE_SET_RESOLVER` (`1 << 24`).

The revoked address can then call `setResolver` on the migrated V2 name. A resolver controlled by that address can return attacker-selected address records, creating the selected in-scope Critical impact: a v1-to-v2 Manager migration that assigns the name's resolver authority to an attacker-controlled address. This requires no stolen wallet, compromised RPC, privileged project role, contract bug, or public-network testing.

## Vulnerability Details

### 1. The preview stores two different V1 identities

For an unwrapped `.eth` 2LD, Alice can own the BaseRegistrar NFT while Mallory is the ENS V1 Registry owner/manager. The V1 query requests both `owner` and `registrant`. `classifyName` recognizes Alice as registrant while retaining the different Registry owner as `managerAddress`:

```ts
// packages/migration/src/service/classifyNames.ts:148-167
const registrant = domain.registrant
if (registrant?.id.toLowerCase() !== addr) return null

const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

`buildMigrationPlan` stores this classified object. When Alice clicks Upgrade, `MigrationPage` passes the existing `gasEstimate.plan` into the state machine rather than rebuilding it. The V1 names query has a five-minute stale window, and the gas-estimate query key includes selected domain IDs but not manager identity.

### 2. Alice can revoke Mallory without changing registrar ownership

After the preview is built, Alice calls V1 `BaseRegistrarImplementation.reclaim(labelhash('alice'), Alice)`. The function lets the token owner or approved account replace the ENS Registry owner while Alice remains the BaseRegistrar NFT owner. The current V1 Registry owner is now Alice and Mallory no longer has V1 authority, but the preview still contains `managerAddress = Mallory`.

### 3. Live preflight does not bind the stored manager to current V1 state

`runEligibilityChecks` invokes `checkOwnership` and checks V2 migration reservation state. For an unwrapped name, `checkOwnership` reads BaseRegistrar `ownerOf`; it never reads the current V1 ENS Registry `owner`. `assertMigrationApprovalPlanCurrent` compares approval keys and derives whether manager restoration is required only from the stored manager's null/non-null status. Resolver-readiness and route checks are refreshed, but the stale classified object is passed unchanged into batch construction.

The observed PoC executes the production `runEligibilityChecks` function and records its requested multicalls. It observes `ownerOf` and `getStatus`, and no `owner` read.

### 4. The owner-signed batch consumes the stale address

The decisive sink passes the stored address directly to the V2 registry:

```ts
// apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:18-24
return {
  to: V2_CONTRACTS.ETHRegistry,
  data: encodeFunctionData({
    abi: ETH_REGISTRY_V2_ABI,
    functionName: 'grantRoles',
    args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
  }),
}
```

`buildAtomicMigrationBatches` orders migration before `manager-role-grant`, wraps both calls in one `executeByOwner` HCA batch, and submits it through Alice's signer. The UI describes this only as “Restore the existing managers for your names” and does not identify the recipient address.

The observed PoC invokes the production batch builder, decodes the HCA payload and inner registry call, and obtains:

```
grantRoles(labelToCanonicalId('alice'), 1 << 24, Mallory)
```

### 5. The grant succeeds under the deployed registry authorization rules

The following source-level path shows that the encoded call does not merely revert:

1. Because the stored manager is non-null, the approval planner first asks Alice to approve her HCA as an operator of the V2 ETH registry.
2. Migration registers Alice as V2 owner with `REGISTRATION_ROLE_BITMAP`, including `ROLE_SET_RESOLVER` and `ROLE_SET_RESOLVER_ADMIN`.
3. `PermissionedRegistry._getRoles` combines Alice's roles with an approved operator's roles, so the HCA inherits Alice's role-admin authority.
4. `_getSettableRoles` converts that admin bit into authority to grant the corresponding regular bit. The HCA can therefore execute `grantRoles(..., ROLE_SET_RESOLVER, Mallory)` after migration.
5. `PermissionedRegistry.setResolver` checks `ROLE_SET_RESOLVER` directly and has no owner-only restriction. Mallory can call it after the batch.

The application's post-state verification reinforces the issue: it records a `manager-role` expectation for the same stale address and checks `hasRoles(resource, roleBitmap, account)`. Mallory receiving the unauthorized role is treated as successful completion.

## Impact Details

The direct demonstrated impact is restoration of resolver-setting authority to an address that the registrant revoked before signing. This crosses the authorization boundary preserved by `reclaim`: Alice remains registrar owner and explicitly replaces Mallory as V1 Registry owner, yet the normal Manager migration recreates Mallory's authority in V2.

After the batch, Mallory can set the name's V2 resolver to a contract under Mallory's control. That resolver can return attacker-selected address records for `alice.eth`, so applications and users resolving the name may send payments to an attacker-selected address. The selected program impact explicitly covers a v1-to-v2 migration that assigns the name, registry, or resolver to an attacker.

The PoC demonstrates the application path and exact harmful calldata. I did not broadcast a transaction, mutate a deployed registry, change a resolver, or demonstrate fund loss. Contract-source analysis is included only to establish that the application's encoded grant has a valid authorization path. The vulnerability and remediation are in the in-scope Manager application; the out-of-scope smart contracts are not alleged to be defective.

## References

* Audited competition snapshot: <https://github.com/immunefi-team/audit-comp-ens/tree/cda79acaad59711b943fc68207ebb3f1d0ff8596>
* Classification sink: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L148-L167>
* V1 cache window: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L23>
* Live eligibility checks: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/preflightChecks.ts#L47-L101>
* Stale role-grant sink: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L18-L24>
* Atomic batch ordering: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L737-L815>
* Self-confirming verification: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353>
* V1 `reclaim`: <https://github.com/ensdomains/ens-contracts/blob/staging/contracts/ethregistrar/BaseRegistrarImplementation.sol#L852-L859>
* V2 migration registration: <https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/migration/UnlockedMigrationController.sol#L141-L153>
* V2 registry role grant rules: <https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/registry/PermissionedRegistry.sol#L501-L547>
* V2 `setResolver` authorization: <https://github.com/ensdomains/contracts-v2/blob/8d1c89350729f87b3968cf41ef3d683951444a44/contracts/src/registry/PermissionedRegistry.sol#L129-L141>

## Suggested Remediation

Before presenting final confirmation and again immediately before building each owner-signed atomic batch, read the current V1 ENS Registry owner directly from chain for every unwrapped 2LD. Compare that exact address with the manager identity used in the reviewed plan. If it differs, invalidate the plan and return the user to a preview that displays the new effective manager and role grant. Batch construction and post-state verification must consume only this freshly bound identity.

Add regression cases for an unchanged manager, a manager replaced by the registrant, a manager replaced by another legitimate address, and a manager change during prerequisite approval transactions.

## Proof of Concept

## Proof of Concept

### Preconditions

* Target: `immunefi-team/audit-comp-ens` commit `cda79acaad59711b943fc68207ebb3f1d0ff8596`
* Route: an unwrapped `.eth` 2LD migrated through the Manager app
* Alice owns the BaseRegistrar NFT; Mallory is the V1 ENS Registry owner when the preview is built
* After preview, Alice calls `BaseRegistrarImplementation.reclaim` so Alice becomes the current V1 Registry owner and Mallory is revoked

No public testnet or mainnet was used. The runnable application test executed against the source snapshot in a disposable, network-disabled container.

### Step-by-step reproduction

1. Construct the queried V1 domain for `alice.eth` with `registrant.id = Alice` and `owner.id = Mallory`.
2. Call the production `classifyName(domain, Alice)`. It classifies the name for Alice and stores `managerAddress = Mallory`.
3. Model the intervening `reclaim`: the current V1 Registry owner is now Alice, which differs from the stored `managerAddress`.
4. Call the production `runEligibilityChecks` with that stale classified name. Mock only its multicall transport so every contract function requested by production preflight can be inspected.
5. Return Alice for BaseRegistrar `ownerOf` and an available V2 `getStatus`. The classified name remains eligible.
6. Inspect the requested functions. They contain `ownerOf` and `getStatus`, but no V1 Registry `owner` read.
7. Pass the same classified name to the production `buildAtomicMigrationBatches` function for the unlocked migration route.
8. Decode the produced HCA `executeByOwner` payload, locate the V2 ETH Registry execution, and decode it with the production registry ABI.
9. Observe `grantRoles(labelToCanonicalId('alice'), 1 << 24, Mallory)`.
10. Inspect `batch.verificationExpectations`. It contains a `manager-role` expectation for Mallory and the same role bitmap, so post-state verification treats the unauthorized grant as successful migration completion.

The essential test assertions are:

```ts
const staleDomain = makeDomain({
  ownerId: MALLORY,
  registrantId: ALICE,
  resolverAddress: null,
})
const classification = classifyName(staleDomain, ALICE)
assert(classification?.type === 'classified')
const staleClassified = classification.name
expect(staleClassified.managerAddress).toBe(MALLORY)

// Alice reclaimed the current V1 Registry node after this snapshot.
const currentV1RegistryOwner = ALICE
expect(currentV1RegistryOwner).not.toBe(staleClassified.managerAddress)

const eligibility = await runEligibilityChecks(
  publicClientWithMockedMulticallTransport,
  [staleClassified],
  ALICE,
)
expect(eligibility.eligible).toEqual([staleClassified])
expect(preflightFunctions).toContain('ownerOf')
expect(preflightFunctions).toContain('getStatus')
expect(preflightFunctions).not.toContain('owner')

const atomicPlan = await buildAtomicMigrationBatches({
  wallet: ALICE,
  classified: [staleClassified],
  directRoutes: unlockedRoute,
  // deterministic test inputs omitted for brevity
})
const grant = decodeRegistryExecution(atomicPlan.batches[0])
expect(grant.functionName).toBe('grantRoles')
expect(grant.args).toEqual([
  labelToCanonicalId('alice'),
  1n << 24n,
  MALLORY,
])
expect(atomicPlan.batches[0].verificationExpectations).toContainEqual(
  expect.objectContaining({
    type: 'manager-role',
    account: MALLORY,
    roleBitmap: 1n << 24n,
  }),
)
```

Run the full test from the repository root using the frozen project dependencies:

```sh
pnpm exec biome check apps/manager/src/features/migration/service/staleManagerRegrant.poc.test.ts
pnpm --filter manager exec vitest run \
  src/features/migration/service/staleManagerRegrant.poc.test.ts \
  --reporter=verbose
```

Observed output from the exact full test:

```
Checked 1 file in 14ms. No fixes applied.

✓ src/features/migration/service/staleManagerRegrant.poc.test.ts > PoC: stale V1 manager restoration > re-grants resolver authority after the registrant revoked the former manager 16 ms
Test Files  1 passed (1)
Tests       1 passed (1)
Duration    3.07s (transform 359ms, setup 0ms, import 2.63s, tests 17ms, environment 270ms)
```

The full test also passed the repository's targeted TypeScript checker with no diagnostics. Exact full-test SHA-256:

`1A4036413F3051A28B68D01794D6AEE051BBF34B9EA81DF66B0AC3420065A513`

### Why the resulting call succeeds

The source references in the main report show the complete authorization path. The migration first registers Alice with the resolver role and its admin bit. Alice has already approved her HCA as a registry operator. The registry combines the owner's roles with an approved operator's roles, allowing the HCA to grant Mallory the regular resolver bit after migration. `setResolver` then checks that regular bit directly.

### Validation boundary

The test executes production classification, eligibility logic, batch construction, calldata encoding, and verification expectations. Only the multicall transport is mocked so the requested live reads can be asserted. It does not broadcast a transaction or claim observed fund loss. The main report's deployed-source trace establishes the authorization path for the exact decoded grant and Mallory's subsequent `setResolver` call.
