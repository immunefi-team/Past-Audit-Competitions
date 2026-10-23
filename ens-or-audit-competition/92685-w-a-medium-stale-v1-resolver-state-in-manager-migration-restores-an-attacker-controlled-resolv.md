# 92685 w a medium stale v1 resolver state in manager migration restores an attacker controlled resolver and hijacks the migrated ens name

**Submitted on Sep 11th 2026 at 15:15:45 UTC by @NuttyHare14789 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92685
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Summary

ENS Manager caches the V1 name inventory, including each name's resolver address, for five minutes and copies that address into the v1-to-v2 migration plan.

If the legitimate new owner replaces a hostile resolver after Manager's first read but before confirming migration, Manager does not re-read the canonical V1 registry. It submits the old resolver from the cached plan. The final V2 name therefore points to resolver code controlled by the former owner, although the live V1 resolver was safe when the victim confirmed **Upgrade Names**.

Manager reports the migration as successful because its post-state verification compares the final V2 resolver against the same stale value used to construct the transaction.

## Scope and severity

* **Asset:** `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager`
* **Category:** Websites & Apps
* **Severity:** Critical
* **Selected impact:** Hijacking or bricking an ENS name during a Manager flow (v1-to-v2 migration): the migration assigns the name's resolver to an attacker.
* **Official audited revision:** `63772fd872af472ced58b009499355f3430c2a86`
* **Competition mirror base used for the PoC:** `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

The relevant migration files are identical between the audited revision and the competition mirror base used for the PoC. The mirror base adds only the competition fork commit and an unrelated `hono` dependency update.

## Impact

An ENS resolver controls the addresses and other records returned for a name. In the executed PoC, the victim authorizes the normal Manager migration, but the final V2 registry points to the former owner's resolver. Calling `addr(bytes32)` on that resolver returns the former owner's address.

Applications resolving the migrated name can consequently receive an attacker-controlled address rather than the victim's intended address. This is the exact scoped Critical state: a Manager migration assigns the name's resolver to an attacker.

The PoC does not rely on a theoretical incorrect display and does not claim unproven direct theft. It demonstrates the on-chain name-hijack state and attacker-controlled resolution.

## Preconditions and attack path

1. The attacker is the legitimate ordinary owner of an existing V1 name before transferring or selling it to the victim.
2. While still authorized, the attacker sets a custom resolver they control.
3. The attacker transfers the registration to the victim.
4. The victim reclaims V1 registry manager authority. The former owner can no longer change the resolver directly.
5. The victim opens the new Manager dashboard. Manager reads the hostile resolver into its migration state.
6. The victim repairs the name through the official legacy ENS Manager using **More → Resolver → Edit → Latest resolver → Update**.
7. The victim returns to the already-open new Manager and confirms **Upgrade Names** before the five-minute query becomes stale.
8. Manager migrates the cached hostile resolver instead of the live safe resolver.

This is a normal post-transfer recovery and migration sequence. The attacker does not require access to the victim's device, network, wallet, credentials, or private key. The attacker's only actions are ordinary transactions performed while they legitimately own the name.

## Root cause

The defect is a missing freshness invariant across preview, transaction construction, and verification:

1. `apps/manager/src/features/migration/hooks/useV1Names.ts:12-29` sets `staleTime: 5 * 60 * 1000` for the V1 inventory.
2. `packages/migration/src/service/classifyNames.ts:50-88` stores `v1ResolverAddress` and selects `keep-v1` for an unknown custom resolver. Lines 137-183 copy `domain.resolver.address` into the classified migration name.
3. `apps/manager/src/features/migration/service/encodeMigration.ts:24-50` returns the stored `name.v1ResolverAddress` for `keep-v1`.
4. `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:453-491` places that address into `MigrationData.resolver` as `expectedResolver`.
5. `apps/manager/src/features/migration/service/migrationService.ts:1161-1180` refreshes mutable approval state before the first wallet prompt, but does not refresh or compare the live V1 resolver.
6. `buildAtomicMigrationBatches.ts:544-565` records the same stale value as the post-state expectation. `verifyAtomicMigrationBatch.ts:263-275` then checks only that the final V2 resolver equals that stale expectation.

Both the migration calldata and the success verifier therefore agree on the same obsolete attacker-controlled address. Verification accepts the exploit result instead of detecting it.

## Why this is not a disclosed known issue

* `QA-01` concerns errors in follow-up actions for certain post-transfer role combinations. Here migration succeeds and assigns an attacker-controlled resolver.
* `WEB-1305` concerns a role transfer to a V1 name hanging. This report concerns a completed v1-to-v2 migration with stale resolver calldata.
* `WEB-390` concerns subnames missing from the migration list. The PoC uses ordinary second-level `.eth` names that appear in the migration list.
* `SEC-TXM-002` concerns a cached smart-account address. This finding concerns a cached V1 resolver in Manager's migration plan, with a different root cause and a demonstrated final impact.

## Recommended remediation

Immediately before the first wallet prompt and before each migration batch, read the resolver for every selected source name from the canonical V1 ENS registry. Compare it with the resolver used to construct the preview and calldata. If any value changed, fail closed, invalidate the plan, refetch the V1 inventory, and require the user to review a rebuilt transaction.

Post-state verification must bind to the fresh pre-submit source value rather than the old cached plan. If supported by the migration contracts, the strongest fix is to include the expected source resolver in the atomic migration and revert on-chain when the V1 resolver differs, eliminating the residual check-to-sign race.

## Link to Proof of Concept

<https://gist.github.com/framarche83/a1b9c48694ffe63dbfcce820a37c6e37>

## Proof of Concept

## PoC resources

* **Network:** local Ethereum Sepolia fork, chain ID `11155111`
* **Pinned fork block:** `11681965`
* **Execution date:** 2026-09-11

The secret Gist contains the complete PoC patch series, final Playwright source, deterministic Docker override, demonstration fix, and executed-result record. The steps and expected results are also included below, so the Gist is not a replacement for this Proof of Concept.

## Prerequisites

* Node.js 22
* pnpm 10 through Corepack
* Docker Desktop or Docker Engine
* Git
* a reviewer-supplied Sepolia RPC endpoint

All exploit execution occurs on the local fork. Nothing is broadcast to public Sepolia.

## Step 1 — Prepare the scoped Manager repository

The commands below use PowerShell. Create an empty working directory and run the commands from that directory.

```powershell
$work = (Get-Location).Path
$managerRepo = Join-Path $work 'audit-comp-ens'
$pocRepo = Join-Path $work 'ens-stale-resolver-poc'
$legacyRepo = Join-Path $work 'ens-app-v3'

git clone --branch audit-comp-ready https://github.com/immunefi-team/audit-comp-ens.git $managerRepo
git clone https://gist.github.com/a1b9c48694ffe63dbfcce820a37c6e37.git $pocRepo
git -C $managerRepo checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
git -C $managerRepo am `
  "$pocRepo\0001-test-capture-stale-V1-resolver-migration-impact.patch" `
  "$pocRepo\0002-test-emit-submission-capture-artifacts.patch" `
  "$pocRepo\0003-test-bind-V1-graph-fields-to-live-fork-state.patch" `
  "$pocRepo\0004-test-reproduce-stale-resolver-through-legacy-manager.patch"

Set-Location $managerRepo
corepack enable
corepack pnpm install --frozen-lockfile
```

## Step 2 — Prepare the official legacy recovery UI

```powershell
git clone https://github.com/ensdomains/ens-app-v3.git $legacyRepo
git -C $legacyRepo checkout 89f8681d524e9f40511dadd016ee95a6abff346d
Set-Location $legacyRepo
corepack pnpm install --frozen-lockfile
```

The legacy application is used only for the victim's ordinary resolver recovery. The browser PoC injects the victim's local test wallet and points the UI to the local fork.

## Step 3 — Start a deterministic Sepolia fork and services

Use a fresh fork and fresh Panoptes volume. The tests mutate their three real names, so reset the environment before repeating the suite.

```powershell
Set-Location $managerRepo
if (-not $env:SEPOLIA_FORK_URL) {
  throw 'Set SEPOLIA_FORK_URL to a reviewer-controlled Sepolia RPC endpoint.'
}
$env:PANOPTES_CONTRACTS_FILE = './panoptes/contracts.stale-resolver-poc.json'

docker compose `
  -f e2e/infra/docker-compose.yml `
  -f "$pocRepo\docker-compose.stale-resolver-poc.override.yml" `
  down -v

docker compose `
  -f e2e/infra/docker-compose.yml `
  -f "$pocRepo\docker-compose.stale-resolver-poc.override.yml" `
  up -d
```

The supplied override pins Anvil to block `11681965`.

## Step 4 — Start both ENS applications

Manager, in the first PowerShell window:

```powershell
Set-Location $managerRepo
corepack pnpm --filter manager dev --mode ci --host 127.0.0.1
```

Official legacy Manager, in the second PowerShell window:

```powershell
Set-Location $legacyRepo
$env:NEXT_PUBLIC_CHAIN_NAME = 'sepolia'
$env:NODE_NO_WARNINGS = '1'
corepack pnpm exec next dev -p 3001
```

## Step 5 — Run the vulnerable case and timing control

In a third PowerShell window:

```powershell
Set-Location "$managerRepo\e2e"
$env:MANAGER_APP_URL = 'http://127.0.0.1:3000'
$env:LEGACY_MANAGER_APP_URL = 'http://127.0.0.1:3001'

corepack pnpm exec playwright test `
  --config=projects/manager/playwright.stale-resolver-poc.config.ts `
  --grep "vulnerable:|counterfactual:"
```

Expected runner result:

```
2 passed (approximately 1.2 minutes)
```

The vulnerable `pampa.eth` test performs and asserts every load-bearing edge:

1. The existing holder is an ordinary EOA with no code and is not a `BaseRegistrar` controller.
2. The existing V2 premigration resolver is nonzero; the PoC does not seed or register the V2 name.
3. The former owner deploys and sets a malicious resolver while authorized, then transfers the real V1 registration to the victim.
4. The victim reclaims registry manager authority.
5. A simulated post-transfer `setResolver` from the former owner reverts.
6. Manager reads the malicious resolver into its V1 migration state.
7. The victim uses the official legacy UI to set the latest safe resolver and authorizes the emitted wallet transaction.
8. The canonical V1 registry is asserted to contain the safe resolver.
9. The victim authorizes Manager's displayed **Upgrade Names** flow.
10. The final V2 resolver is asserted to equal the old malicious resolver.
11. `addr(bytes32)` on the final resolver is asserted to return the former owner's address.

Captured vulnerable result:

```
name                              = pampa.eth
ordinary former owner             = 0x4f26c09d3900C39ae8506bDb3359Cc3e4a527806
existing V2 premigration resolver = 0xae66c62AcAE72098BdAc57d8E8AED53EF000b2Ba
malicious resolver                = 0xc13fd5fedfca81b990c71d68c4fbd397f49395a1
live V1 resolver after recovery   = 0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5
final V2 resolver                 = 0xC13fd5fEDfcA81B990C71d68C4Fbd397F49395a1
final addr(bytes32) answer        = 0x4f26c09d3900C39ae8506bDb3359Cc3e4a527806
Playwright                        = passed in 40.4 seconds
```

The `agentmesh.eth` timing control changes only when the official recovery reaches Manager: recovery occurs before the first Manager read. The malicious resolver is not installed, and the final V2 resolver is the safe resolver `0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5`. This test passed in 28.6 seconds.

The PoC creates its own browser traces and screenshots under:

```
audit-comp-ens/.bounty-evidence/manager-stale-v1-resolver-resurrection/runtime/
```

The directory contains `vulnerable-trace.zip`, `counterfactual-trace.zip`, the final-state screenshots, and screenshots of the official legacy recovery UI. These are generated locally by the test and require no Immunefi file attachment.

## Step 6 — Run the same-order fixed-code control

Reset the Docker environment with the `down -v` and `up -d` commands from Step 3. Apply the demonstration fix:

```powershell
git -C $managerRepo am "$pocRepo\0005-fix-reject-stale-V1-resolver-migration-previews.patch"
```

Restart Manager, then run:

```powershell
Set-Location "$managerRepo\e2e"
$env:MANAGER_APP_URL = 'http://127.0.0.1:3000'
$env:LEGACY_MANAGER_APP_URL = 'http://127.0.0.1:3001'

corepack pnpm exec playwright test `
  --config=projects/manager/playwright.stale-resolver-poc.config.ts `
  --grep "patched:"
```

Expected runner result:

```
1 passed (approximately 1.4 minutes)
```

The `luoyingjie.eth` control preserves the vulnerable ordering: Manager reads the hostile resolver, the victim recovers through the official UI, and the victim confirms migration. The added live-registry check rejects the stale plan before any wallet transaction request. V2 remains at the premigration resolver `0xae66c62AcAE72098BdAc57d8E8AED53EF000b2Ba`.

This establishes causality: with vulnerable code the stale resolver is migrated; with only the claimed freshness check added, the identical attack ordering is blocked.

## Test-environment integrity

* `pampa.eth`, `agentmesh.eth`, and `luoyingjie.eth` are real, valid, already-premigrated Sepolia V1 names at the pinned fork block.
* No ENS owner, registrar controller, governance address, or V2 registrar is impersonated.
* No V2 `register` or V2 state-seeding call is used by the final PoC path.
* Fork impersonation is limited to the ordinary former owner and reproduces only transactions this actor can sign with their own key: deploy a resolver, set it while owning the name, and transfer the name.
* The victim uses a controlled local test key and authorizes the recovery and migration through the browser UIs.
* The V1 Graph adapter reads resolver, manager, registrant, and expiry from the canonical fork state on every request. It observes the safe resolver after recovery and does not create the stale value.
* The stale value originates in Manager's five-minute application cache.

## Evidence generated by the PoC

* `vulnerable-trace.zip` — complete vulnerable browser trace
* `vulnerable-final.png` — final migrated state
* `pampa-official-v1-recovery.png` — official recovery UI
* `counterfactual-trace.zip` — timing-control trace
* `counterfactual-final.png` — safe timing-control result
* `agentmesh-official-v1-recovery.png` — timing-control recovery UI
* `patched-code-trace.zip` — same-order patched-code trace
* `patched-code-blocked.png` — patched failure screen
* `luoyingjie-official-v1-recovery.png` — patched-case recovery UI

Additional verification completed: 83 focused unit tests passed across three files, Manager typecheck passed, and Biome passed for every changed file.

All testing was performed on a local fork. No public-chain name, third-party funds, or external system was modified.
