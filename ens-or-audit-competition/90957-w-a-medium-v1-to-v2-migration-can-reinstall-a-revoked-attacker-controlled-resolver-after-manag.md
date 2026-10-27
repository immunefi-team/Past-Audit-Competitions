# 90957 w a medium v1 to v2 migration can reinstall a revoked attacker controlled resolver after manager receives corrected v1 state

**Submitted on Aug 28th 2026 at 12:43:21 UTC by @JAO for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90957
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

### Brief/Intro

The Manager can keep and submit a V1-to-V2 migration plan containing an old attacker-controlled resolver even after its V1 name data updates to the safe replacement resolver. The gas-estimate/plan query is keyed only by the selected name IDs, so a resolver or owner revision for the same name does not rebuild the transaction-authoritative plan. Neither the Upgrade action nor the execution service compares that plan with the live V1 Registry. The production batch builder therefore places the revoked resolver into `Migration.Data.resolver`; a successful migration assigns the V2 name to that resolver, which can resolve the name to the attacker's payment address.

### Vulnerability Details

The audited application revision is:

```
cda79acaad59711b943fc68207ebb3f1d0ff8596
```

The primary demonstrated case uses the following fixed values in all three PoCs:

| Item                                   | Value                                        |
| -------------------------------------- | -------------------------------------------- |
| Name                                   | `victim.eth`                                 |
| Owner/intended payment address         | `0x00000000000000000000000000000000000000a1` |
| Revoked attacker-controlled resolver A | `0x00000000000000000000000000000000000000e1` |
| Current safe V1 resolver B             | `0x231b0Ee14048e9dCcD1d247744d114a4EB5E8E63` |
| Attacker payment address               | `0x00000000000000000000000000000000000000b1` |

#### 1. Manager obtains transaction-authoritative resolver data from the V1 query

`apps/manager/src/features/migration/service/v1SubgraphClient.ts:22-47` obtains `resolver.address`, `owner.id`, and `registrant.id` for each name. Those fields are not merely display data: they are passed into migration classification and plan construction.

#### 2. A same-name state revision does not invalidate the prepared plan

`apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts:62-88` derives the query key from the selected domain IDs only:

```typescript
const domainIds = domains.map((domain) => domain.id).sort().join(',')

queryKey: [
  'migration-gas-estimate',
  ownerAddress?.toLowerCase() ?? '',
  hcaAddress?.toLowerCase() ?? '',
  domainIds,
  selectionRevisionRef.current.revision,
]
```

`selectionRevisionRef` increments only when `domainIds` changes. A change from resolver A to resolver B for `victim.eth` leaves the key unchanged. The runnable hook PoC first prepares a plan from A, rerenders the production hook with corrected V1 domain data containing B, and proves that `buildMigrationPlan` is not called again and `result.current.plan` still contains A.

This is an application cache-key defect. The report does not claim that a defect in the subgraph/indexer is in scope. A stale initial response can come from an already loaded page or normal eventual consistency, but the decisive demonstrated failure is that Manager retains the old plan after its input has been revised for the same name.

#### 3. Production plan construction converts A into migration calldata

`apps/manager/src/features/migration/service/buildMigrationPlan.ts:317-410` classifies the `domains` supplied by the hook and builds the atomic batches from those values.

For a custom resolver, `packages/migration/src/service/classifyNames.ts:65-88` selects the `keep-v1` strategy. `apps/manager/src/features/migration/service/encodeMigration.ts:24-50` then returns `name.v1ResolverAddress`.

`apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:481-491` assigns that value to the migration struct:

```typescript
const expectedResolver = resolverFor(classified, defaultResolver, resolver)
const migrationData = createMigrationData({
  label: classified.label,
  owner: wallet,
  subregistry: zeroAddress,
  resolver: expectedResolver,
})
```

The production-service PoC decodes the actual `MigrationHelper.migrate` call and proves:

```
Migration.Data.label    = victim
Migration.Data.owner    = 0x00000000000000000000000000000000000000a1
Migration.Data.resolver = 0x00000000000000000000000000000000000000e1  (A)
```

#### 4. The Upgrade action submits the retained plan without a V1 resolver check

`apps/manager/src/features/migration/pages/MigrationPage.tsx:259-282` sends `gasEstimate.plan` directly to `migration.start`.

`apps/manager/src/features/migration/service/migrationService.ts:1161-1193` reconciles retries and checks the approval plan before opening the wallet prompts, but does not call V1 `ENSRegistry.resolver(node)` or compare it with `Migration.Data.resolver`. The execution-time batch rebuild refreshes V2 route/readiness state, not the V1 resolver input.

The confirmation text in `WalletConfirmationStepsDialog.tsx:91-108` is generic (`Upgrade # name` / `Migrate the selected names and restore their records`) and does not disclose the resolver address that will be installed.

#### 5. The accepted migration installs A even though live V1 contains B

The contract-side PoC is a consequence harness, not an allegation that the out-of-scope ENS contracts are vulnerable. It uses the application-produced struct at ENS v2 contracts revision:

```
48b3e2d39513b9dd32ef1850877a29009bc807b9
```

Before migration it establishes and asserts:

```
V1 Registry resolver(victim.eth) = B
B.addr(victim.eth)               = 0x00000000000000000000000000000000000000a1
```

It then submits the exact stale application value `Migration.Data.resolver = A`. `MigrationHelper.migrate` validates the current BaseRegistrar owner and approval, while `UnlockedMigrationController._inject` registers `md.resolver` directly. The transaction succeeds and the PoC asserts:

```
V2 resolver(victim.eth) = A
A.addr(victim.eth)      = 0x00000000000000000000000000000000000000b1
```

This proves a live-state/executed-state mismatch and a concrete attacker-controlled consequence. It is not a display-only discrepancy and does not assume that merely using an old value is harmful.

### Attack sequence

{% stepper %}
{% step %}

## Mallory previously controlled custom resolver A for `victim.eth`

For example, as an ordinary per-name delegate or through a resolver service that Alice has now determined is compromised.
{% endstep %}

{% step %}

## Alice opens the normal Manager migration page

Manager receives A and prepares the V1-to-V2 plan.
{% endstep %}

{% step %}

## Alice repairs the V1 resolver

Alice revokes the obsolete setup and changes the live V1 Registry resolver to safe resolver B. Mallory has no current registry or registrar authority after this repair.
{% endstep %}

{% step %}

## The selected V1 domain data updates

The V1 domain data for the already selected `victim.eth` updates from A to B. Because the selected domain ID is unchanged, the gas-estimate query retains the plan containing A.
{% endstep %}

{% step %}

## Alice continues the Upgrade workflow

Alice continues the intended V1-to-V2 Upgrade workflow. The confirmation does not identify A and no execution-time V1 resolver validation occurs.
{% endstep %}

{% step %}

## The migration reinstalls A

The owner-authorized migration succeeds and installs A as the V2 resolver, despite live V1 pointing to B at execution time.
{% endstep %}

{% step %}

## A resolves `victim.eth` to Mallory's address

Payments or other resolution-dependent interactions are redirected until Alice detects and repairs the V2 name.
{% endstep %}
{% endstepper %}

Mallory needs no governance privilege, leaked key, browser compromise, or access to the victim's device. The only user action after revocation is the program's ordinary V1-to-V2 migration flow.

### Impact Details

The PoC produces the exact Critical in-scope outcome: a Manager V1-to-V2 migration assigns the name's resolver to an attacker. Ownership remains with Alice, but name resolution is controlled by Mallory's resolver immediately after a transaction that Manager reports as a successful migration.

This is security-sensitive because an ENS resolver determines the addresses and resources returned for the name. The installed resolver in the PoC returns Mallory's payment address rather than Alice's intended address. The unsafe state persists on V2 until the owner notices and submits a corrective transaction.

The report selects only the explicit migration-hijack impact. It does not separately claim direct theft of funds, a contract vulnerability, or a subgraph/indexer vulnerability.

### References

* Secret Gist containing the README and all three executable PoCs: <https://gist.github.com/jaooli-cyber/2bd343a770f9e7280dd2df93fb47c974>
* Competition information and Known Issues: <https://immunefi.com/audit-competition/audit-competition-ens/information/>
* In-scope Manager asset and exact Critical migration impact: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Frozen Manager source: <https://github.com/immunefi-team/audit-comp-ens/tree/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager>
* Plan cache key: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L104>
* Upgrade uses the retained plan: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/pages/MigrationPage.tsx#L259-L282>
* Resolver encoding: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/encodeMigration.ts#L24-L50>
* Migration data construction: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L481-L491>
* No V1 resolver validation before submission: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L1161-L1193>
* Contract consequence source: <https://github.com/ensdomains/contracts-v2/blob/48b3e2d39513b9dd32ef1850877a29009bc807b9/src/migration/MigrationHelper.sol#L94-L113>
* Direct use of `md.resolver`: <https://github.com/ensdomains/contracts-v2/blob/48b3e2d39513b9dd32ef1850877a29009bc807b9/src/migration/UnlockedMigrationController.sol#L153-L165>

## Link to Proof of Concept

<https://gist.github.com/jaooli-cyber/2bd343a770f9e7280dd2df93fb47c974>

## Proof of Concept

### What the three files prove

The files form one chain and intentionally use the same name and addresses:

{% stepper %}
{% step %}

## `stale-v1-plan-cache.poc.test.tsx`

The production React hook retains plan A after its same-name V1 input changes to B.
{% endstep %}

{% step %}

## `stale-v1-state-migration.poc.test.ts`

The production classifier and atomic batch builder encode A as `Migration.Data.resolver`.
{% endstep %}

{% step %}

## `stale-migration-consequence.poc.t.sol`

With live V1 resolver B, the app-produced resolver A is accepted into V2 and returns Mallory's payment address.
{% endstep %}
{% endstepper %}

SHA-256:

```
8211447a88fb3705c90e8f6f473f8a5e5009aed65bf197c51f502b89db5969a5  stale-v1-plan-cache.poc.test.tsx
e54c08c4c86f6e1075c82206dc4819a720208c26cbfde32c7a4bfa9478074e63  stale-v1-state-migration.poc.test.ts
987dc131105735ba92f3af751c6bb196a9ec7181d1925329c8ef2a49974ae15c  stale-migration-consequence.poc.t.sol
```

{% stepper %}
{% step %}

## Run the Manager hook and calldata PoCs

Check out the frozen competition source:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
pnpm install --frozen-lockfile
```

Download the following files from the Secret Gist linked in References and copy them to:

```
apps/manager/src/features/migration/hooks/stale-v1-plan-cache.poc.test.tsx
apps/manager/src/features/migration/service/stale-v1-state-migration.poc.test.ts
```

Run:

```bash
pnpm --dir apps/manager exec vitest run \
  src/features/migration/hooks/stale-v1-plan-cache.poc.test.tsx \
  src/features/migration/service/stale-v1-state-migration.poc.test.ts \
  --reporter=verbose
```

Expected result:

```
PASS  stale-v1-plan-cache.poc.test.tsx
  keeps the attacker resolver plan after the V1 domain data updates to the safe resolver

PASS  stale-v1-state-migration.poc.test.ts
  assigns a stale custom resolver directly to the migrated V2 name

Test Files  2 passed
Tests       4 passed
```

The service file also contains two secondary regression cases for the same missing V1-state validation (revoked manager restoration and public-resolver record replay). They are not required for the selected primary impact.
{% endstep %}

{% step %}

## Run the contract consequence PoC

Check out the tested ENS v2 contract revision with dependencies:

```bash
git clone --recurse-submodules https://github.com/ensdomains/contracts-v2.git
cd contracts-v2
git checkout 48b3e2d39513b9dd32ef1850877a29009bc807b9
```

Download the following Solidity file from the Secret Gist linked in References and copy it to:

```
test/unit/migration/StaleMigrationConsequence.poc.t.sol
```

Run only the primary consequence test:

```bash
forge test \
  --match-path test/unit/migration/StaleMigrationConsequence.poc.t.sol \
  --match-test test_StaleResolverOverridesSafeLiveV1ResolverDuringMigration \
  -vv
```

Expected result:

```
[PASS] test_StaleResolverOverridesSafeLiveV1ResolverDuringMigration()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The assertions inside that test prove, in order:

1. Live V1 `resolver(victim.eth)` is B.
2. B resolves the intended payment address `0x...a1`.
3. The migration is submitted with the production-app value A.
4. The migration succeeds.
5. V2 `resolver(victim.eth)` is A, not B.
6. A resolves the attacker payment address `0x...b1`.
   {% endstep %}
   {% endstepper %}

### Validation performed by the researcher

```
Manager targeted PoCs:       4 passed, 0 failed
Manager TypeScript check:    passed
Biome check:                 passed
Full Manager regression:     exit status 0
Contract consequence PoCs:   2 passed, 0 failed
Full contracts regression:   exit status 0
```

The full-suite browser-environment output contains baseline Reown/happy-dom aborted remote-fetch messages during teardown; all commands exit with status 0.
