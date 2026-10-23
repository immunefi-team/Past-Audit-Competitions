# 92838 w a medium toctou in manager v1 to v2 migration reinstalls a replaced attacker controlled resolver and hijacks an ens name

**Submitted on Sep 12th 2026 at 18:28:16 UTC by @NuttyHare14789 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92838
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

ENS Manager retains V1 name data for five minutes and can reuse an obsolete resolver address when building a v1-to-v2 migration. If the current owner replaces a hostile resolver after Manager's first read but before approving the migration, Manager can still install the replaced resolver in V2. The migrated name then resolves to the former owner despite the victim having repaired the live V1 resolver before migration.

## Vulnerability Details

`useV1Names.ts` configures the V1 name query with a five-minute `staleTime`. The migration classification retains the resolver observed by that query as `v1ResolverAddress`.

For the `keep-v1` strategy, `resolverFor()` in `encodeMigration.ts` later copies that stored address into the migration plan. Neither migration execution nor its post-state verification compares the planned resolver with the current V1 Registry immediately before requesting the wallet transaction.

This creates a time-of-check/time-of-use window:

1. A legitimate V1 owner installs a resolver they control and then transfers the registration to another user.
2. The recipient reclaims V1 registry ownership, removing the former owner's authority. A later `setResolver` call from the former owner is rejected by the deployed V1 Registry.
3. The recipient opens Manager's normal, target-specific `Upgrade Names` flow. Manager reads the hostile resolver and retains that snapshot.
4. The recipient uses the official legacy ENS Manager to replace the hostile resolver with the known safe public resolver.
5. Without waiting for Manager's five-minute cache to expire, the recipient completes the already-open upgrade flow and approves its wallet requests.
6. Manager migrates the name using the resolver from its pre-recovery snapshot.

The PoC's V1 Graph adapter rereads the forked V1 Registry for every request. After recovery, it observes the safe resolver. This proves that the external adapter is not manufacturing or retaining the stale value. The harmful value is retained by the audited Manager application's client cache.

The post-state check is also self-confirming because it compares the migrated resolver with the same stale migration plan. It therefore accepts the hostile resolver as the expected result.

No project-privileged account is required at migration time. The attacker is only a former ordinary owner of the transferred name. The PoC confirms that this former owner can no longer change the resolver directly after transfer and reclaim.

The audited migration UI is pre-release and protected by its existing client feature flag. The repository's release notes state that the flag should remain disabled until the contract and QA rollout gates are complete.

The PoC uses the repository's own E2E flag override only to enter this shipped migration UI. It does not replace or bypass permission checks, deployed contracts, resolver reads, wallet requests, the migration encoder, or migration execution.

This rollout status is disclosed and is not assumed to represent current public availability. The Audit Competition's Critical impact definition nevertheless expressly includes a v1-to-v2 Manager migration that assigns a name, ownership, registry, or resolver to an attacker.

This issue is distinct from a stale-manager-role migration. The load-bearing field here is `v1ResolverAddress`, the sink is resolver selection during migration, and the demonstrated final state is V2 name resolution to the former owner. The defect-only remediation and regression test are resolver-specific.

The recommended fix is to read the live V1 resolver immediately before submission. If it differs from the planned resolver, Manager should reject or rebuild the migration plan and request fresh user confirmation. Post-state verification should also bind to that fresh value.

## Impact Details

The PoC demonstrates the selected in-scope Critical impact:

> Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker.

Immediately before migration, the live V1 resolver is the known safe public resolver. The ordinary Manager migration nevertheless completes with the former owner's resolver installed in V2, and `addr(bytes32)` returns the former owner's address.

Wallets, applications, and users relying on the migrated ENS name may therefore resolve it to the attacker and send assets to the wrong recipient. Transfers already made to that address are irreversible.

The hostile resolver can also return incorrect records or disrupt resolution. Although the current owner may later repair the resolver, that does not recover assets transferred while the hostile resolution was active.

No fixed monetary-loss amount is asserted.

## References

* In-scope Manager asset: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* V1 query cache: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/migration/hooks/useV1Names.ts>
* Migration encoder: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/migration/service/encodeMigration.ts>
* Migration execution: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/migration/service/migrationService.ts>
* Migration rollout gate: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/docs/HCA\\_MIGRATION\\_RELEASE\\_NOTES.md>
* Competition scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>

## Link to Proof of Concept

<https://gist.github.com/framarche83/058ba7ec49f0c0a9758f19f8cdb119ad>

## Proof of Concept

## Proof of Concept

### Demonstrated impact

This PoC demonstrates that the ENS Manager v1-to-v2 migration flow can reinstall an obsolete attacker-controlled V1 resolver after the current owner has already replaced it with a safe resolver.

The resulting V2 name uses the former owner's resolver, and `addr(bytes32)` returns the former owner's address.

### Environment

* Target repository: `https://github.com/immunefi-team/audit-comp-ens`
* Official branch: `audit-comp-ready`
* Official base commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`
* Audited upstream revision: `63772fd872af472ced58b009499355f3430c2a86`
* Legacy ENS Manager: `https://github.com/ensdomains/ens-app-v3`
* Legacy Manager commit: `89f8681d524e9f40511dadd016ee95a6abff346d`
* Node.js: `22.23.2`
* pnpm: `10.27.0`
* Playwright: `1.58.2`
* Anvil: `1.8.1`
* Chain: local Sepolia fork, chain ID `11155111`
* Fork block: `11689609`

All transactions execute against local Anvil forks. No exploit transaction is sent to a public network.

The complete runnable harness, supporting files, structured result, and run manifest are provided as complete files in the secret Gist linked to this report.

### Feature-flag disclosure

The audited migration UI is pre-release and remains behind its existing client feature flag, as documented in `apps/manager/docs/HCA_MIGRATION_RELEASE_NOTES.md`.

The PoC uses the repository's own E2E feature-flag override only to enter this shipped migration UI. It does not replace or bypass ownership checks, deployed contracts, resolver reads, wallet requests, the migration encoder, or migration execution.

The Audit Competition's Critical impact definition expressly includes a v1-to-v2 Manager migration that assigns the name, ownership, registry, or resolver to an attacker.

### Setup

1. Clone and check out the audited repository:

```bash
git clone --branch audit-comp-ready https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack enable
pnpm install --frozen-lockfile
```

2. Clone the official legacy Manager into the adjacent directory expected by the harness:

```bash
git clone https://github.com/ensdomains/ens-app-v3.git ../ens-app-v3-review
git -C ../ens-app-v3-review checkout 89f8681d524e9f40511dadd016ee95a6abff346d
pnpm --dir ../ens-app-v3-review install --frozen-lockfile
```

3. Copy every complete PoC file from the secret Gist to the destination specified in its `README.md`.
4. Apply the supplied wallet-fixture instrumentation patch at the audited repository root:

```bash
git apply 0001-instrument-manager-wallet-requests.patch
```

This patch exposes wallet requests to the E2E harness without copying the repository's test mnemonic.

5. Ensure that Google Chrome and Anvil are available.
6. Run the complete vulnerable, timing-control, and patched-control capture.

PowerShell:

```powershell
$env:ANVIL_BIN = (Get-Command anvil).Source
$env:BOUNTY_TEST_RESULT_PATH = (Join-Path $PWD "result.json")
node e2e/projects/manager/poc/run-stale-resolver-capture.mjs
```

POSIX:

```bash
ANVIL_BIN="$(command -v anvil)" \
BOUNTY_TEST_RESULT_PATH="$PWD/result.json" \
node e2e/projects/manager/poc/run-stale-resolver-capture.mjs
```

For each arm, the orchestrator starts a new local Sepolia fork at the same pinned block, the audited Manager, the official legacy Manager, and a fresh controlled-wallet browser journey.

Every arm therefore starts from the same chain snapshot and uses the same name and actors.

### Vulnerable execution

1. The harness uses the existing V1 registration `pampa.eth`, owned at the fork block by an ordinary EOA with no ENS project privileges.
2. While still the legitimate owner, that EOA deploys and installs a static resolver that returns the owner's address.
3. The EOA transfers the registration to the controlled victim.
4. The victim calls the deployed V1 `reclaim` function, becoming both registrant and V1 registry manager.
5. The harness verifies that a subsequent `setResolver` attempt from the former owner is rejected by the deployed V1 Registry. The former owner therefore has no authority over the name at this point.
6. The victim opens the audited Manager dashboard and enters its target-specific `Upgrade Names` flow. Manager reads and caches the hostile V1 resolver.
7. After this initial Manager read, the victim opens the official legacy ENS Manager and uses its normal profile flow to replace the hostile resolver.
8. The harness reads the V1 Registry directly and verifies that the live resolver is now the known safe resolver:

```
0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5
```

9. The test also verifies that the official legacy Manager displays its target-controlled `Open new app` link and that the destination is exactly:

```
https://app.ens.dev
```

No attacker URL, phishing prompt, or social engineering is used.

10. Without waiting for Manager's five-minute cache to expire, the victim returns to the already-open migration flow and approves the wallet requests produced by Manager.
11. The migration succeeds, but the final V2 resolver is the obsolete attacker-controlled resolver cached before recovery:

```
0xC13fd5fEDfcA81B990C71d68C4Fbd397F49395a1
```

12. Calling `addr(bytes32)` on that resolver for `pampa.eth` returns the former owner's address:

```
0x4f26c09d3900C39ae8506bDb3359Cc3e4a527806
```

The final V2 name therefore resolves to the former owner even though the victim repaired the live V1 resolver before approving the migration.

### Timing control

The orchestrator stops every service and starts a fresh identical Sepolia fork at the same block.

It repeats the same `pampa.eth` scenario with the same former owner and victim. The only changed variable is timing: the victim replaces the hostile resolver before Manager's first V1 read.

Manager consequently observes the safe resolver and does not install the hostile resolver in V2.

This establishes that the harmful result depends on Manager retaining the pre-recovery resolver snapshot.

### Defect-only patched control

The orchestrator starts another fresh identical fork at the same block and applies only the included production resolver-freshness patch.

It then repeats the vulnerable ordering using the same `pampa.eth`, former owner, victim, deployed contracts, browser path, and wallet journey.

The patched Manager compares the planned resolver with the live V1 Registry immediately before submission. It detects the change, blocks the migration before requesting a wallet transaction, and leaves V2 unchanged.

The patch is reversed after the control completes.

### Expected output

```
vulnerable: recovery after dashboard load still migrates the hostile resolver
1 passed

counterfactual: recovery before the first dashboard read blocks the hostile resolver
1 passed

patched: live resolver pre-submit check blocks the same stale migration sequence
1 passed

[capture] all three same-fixture arms passed
```

The generated `result.json` records:

* zero-privilege reachability through public entrypoints;
* end-to-end observed impact;
* the repaired live V1 resolver;
* V2 resolution to the controlled former-owner address;
* successful timing and defect-only counterfactuals;
* `only_defect_varied: yes`.

### Test-boundary clarification

The PoC's V1 Graph adapter reads the current forked V1 Registry on every request. After recovery, it observes the safe resolver. It does not fabricate or retain the stale value.

Unrelated V2 indexer responses are made deterministic only to populate the local dashboard.

Registration ownership, registry authority, resolver changes, migration transactions, wallet requests, and final resolver reads use deployed ENS contracts on the pinned Sepolia fork.

The stale value responsible for the result is retained by the audited Manager application's five-minute query cache.
