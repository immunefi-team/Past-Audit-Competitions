# 92373 w a medium same id migration refresh retains removed resolver authority in an executable plan

**Submitted on Sep 9th 2026 at 10:55:41 UTC by @kryoka for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92373
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief / Intro

Manager can keep an executable V1-to-V2 migration plan after the underlying V1 name data has changed. The plan is keyed by the connected owner, HCA, and domain IDs, but not by the resolver or registry manager captured inside the plan.

If a ready plan contains an attacker-controlled custom resolver and the owner then removes that resolver, a normal refresh for the same ENS node can show the current V1 data without replacing the plan. Clicking **Begin Upgrade** still uses the old plan and encodes the removed resolver into the migration request. The local replay confirms installation of the historical resolver address and immediate attacker-selected resolution through that same address. No later attacker transaction or former-manager resolver change is required.

This report does not depend on a stale or malicious subgraph response. The reproduced refresh returns the correct current object. The defect is that Manager does not bind its executable plan to that object's mutable authority fields.

## Vulnerability Details

At commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, [`useMigrationGasEstimate`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L109) selects complete `V1Domain` objects, including their resolver and owner, and closes over them while building the migration plan. However, the query key is only:

```
['migration-gas-estimate', owner, hca, sortedDomainIds, selectionRevision]
```

`selectionRevision` changes only when the joined domain-ID string changes. Replacing a resolver or manager for the same node therefore does not create a new query identity.

The focus-refresh fixture records the resulting state split:

1. the V1 query initially returns a name with the old resolver and Manager builds a ready plan;
2. the V1 query refetches the same node ID with the correct safe resolver and current owner;
3. current V1 data reflects the safe state, but the gas-plan query remains ready with its old resolver and manager classification.

Manager does not repair the mismatch when the user proceeds. [`MigrationPage`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/pages/MigrationPage.tsx#L259-L299) passes `gasEstimate.plan` directly into `migration.start`.

The first-submit rebuild refreshes some chain-dependent information, but [`buildNextAtomicBatch`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L886-L960) reuses `remaining`, which is the stale classified-name list from that plan. The later approval freshness check validates migration approvals; it does not re-read and reclassify the V1 resolver or manager ([`migrationService.ts:1161-1204`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L1161-L1204)).

For an unwrapped name, classification copies `domain.resolver.address` into `v1ResolverAddress` ([`classifyNames.ts:125-184`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L125-L184)). The `keep-v1` strategy then places that cached address into the migration data ([`encodeMigration.ts:24-50`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/encodeMigration.ts#L24-L50)).

The executor fixture carries the stale plan through the real Manager executor and captures an EOA-to-HCA request constructed for the victim at the mocked transaction-manager boundary. Decoding it finds the removed resolver in the nested `MigrationHelper.migrate(...)` tuple. The captured 1,028-byte outer request is byte-identical to the included local Forge replay. The replay places attacker-controlled resolver runtime at the historical address before the victim replaces it, executes the unchanged Manager request, and shows Universal Resolver selecting that reinstalled historical resolver and returning the attacker address. No later attacker transaction is performed.

## Impact Details

Critical is requested under the listed migration/name-hijacking impact. The submitted local evidence supports stale authority in the prepared request, reinstatement by the exact request in a local replay, and immediate attacker-selected resolution through the reinstalled resolver.

The attacker prerequisite is control of a custom V1 resolver previously used by the name. The bridge installs deterministic attacker-controlled resolver runtime at that historical address before the victim switches to a safe resolver. The separate former-manager grant is still decoded and asserted, but it is not required for the demonstrated resolution consequence.

The cache-identity and submission defects are in the listed Manager application. `packages/migration` is supporting classification code, not one of the five enumerated assets. ENS contract code supports the local replay only; no contract or indexer vulnerability is alleged.

## Preconditions and Attack Sequence

1. The victim's V1 name uses custom resolver R, and the attacker retains administrative control of R.
2. The victim opens the Manager migration flow and a ready plan is built while R is present in the V1 data.
3. The victim removes or replaces R through an authorized V1 client, or an already-pending resolver-change transaction confirms.
4. A normal focus refresh returns the correct current V1 object for the same ENS node ID.
5. Manager's current V1 data is safe, but its ready migration plan still contains R.
6. The scenario requires the victim to click **Begin Upgrade** and approve migration prompts.
7. Manager prepares a nested migration tuple containing R. The local replay installs that address; Universal Resolver immediately selects R and returns the attacker address without a later attacker transaction.

No phishing, stolen credential, local access, malicious wallet, browser defect, third-party compromise, public-chain probing, or high-volume traffic is required.

## Expected Result

Any change to a selected name's resolver, registry manager, ownership, wrapper/fuse state, or expiry must invalidate the ready plan. Immediately before submission, Manager must re-read and reclassify current V1 state and reject any plan whose authority inputs no longer match.

## Actual Result

A correct same-ID refresh leaves the old plan ready. Manager's execution path uses the stale classification and prepares migration calldata containing the removed resolver. The supplemental local replay records installation of that resolver and immediate attacker-selected resolution.

## Evidence Scope and Prerequisites

The Manager fixtures exercise real React Query scheduling, classification, plan builders, calldata encoders, and the migration executor. V1 responses, chain reads, HCA verification, transaction submission, receipts, and post-submit verification are mocked. They capture a constructed request, not wallet approval; no real wallet UI was opened or transaction broadcast to a public chain.

The Forge replay executes the byte-identical request against first-party ENSv2 source using configured-address runtime aliases and locally deployed HCA infrastructure. Those local substitutions do not establish deployed helper identity or compatibility.

The scenario requires the migration page to remain mounted across an authorized resolver change outside it, followed by return and approval before a fresh plan rebuild. This lifecycle has not been established as a normally reachable production workflow. Migration feature allocation, deployed helper/HCA compatibility, and the complete wallet flow were not verified. Both Manager fixtures bypass the production migration route's feature gate.

A completed plan rebuild from fresh authority data closes the demonstrated stale-plan condition. Remount, reload, and invalidation behavior were not covered by these fixtures.

## Published Known-Issue Separation

QA-01/WEB-1257 describe transfer-related failures; WEB-806/WEB-390 concern migration visibility; SEC-TXM-002 concerns cached HCA identity. The reported root is retention of executable resolver/manager authority after correct data refresh. No identical root was identified in the 2026-09-08 [published-issue comparison](https://immunefi.com/audit-competition/audit-competition-ens/information/).

## Recommended Remediation

Include a deterministic digest of all plan-affecting V1 fields in the query key or invalidate the plan whenever a selected domain object changes. At **Begin Upgrade**, fetch and reclassify the current V1 objects and compare them with the plan. Abort on any resolver, manager, ownership, fuse, wrapper, or expiry mismatch. The final confirmation should also identify every resolver and delegated address that the migration will install.

## Link to Proof of Concept

<https://gist.github.com/ferbiy/c3ee709e1f688cf95f152ef062d62ac5>

## Proof of Concept

## Proof of Concept

The accompanying evidence bundle contains the following files. Use confidential submission attachments or a secret Gist shared only through the submission. A secret Gist is unlisted, not access-controlled; anyone with its URL can read it. Do not publish an unpatched finding publicly.

```
README.md
ENS-61-REPORT.md
SHA256SUMS
focus-authority-refresh.test.tsx
stale-executor-runtime.test.tsx
ManagerOuterHCABridge.t.sol
outer-calldata-check.json
forge-output.txt
forge-receipt.json
source-provenance.txt
manager-receipt-summary.json
```

The published filenames are flat. `SHA256SUMS` covers the other ten files, including this report as `ENS-61-REPORT.md`. The receipt and provenance copies are explicitly sanitized; original record digests are retained. Source fixture contents and recorded test outcomes are unchanged.

References: [GitHub Gist visibility](https://docs.github.com/en/get-started/writing-on-github/editing-and-sharing-content-with-gists/creating-gists), [competition disclosure rules](https://immunefi.com/audit-competition/audit-competition-ens/scope/).

The two Vitest files contain the Manager fixtures. The contract files contain the exact-byte local replay and its recorded output and receipt from 2026-09-08. The Manager results come from an earlier run; those fixtures were not rerun with the Forge replay.

### Manager Fixture Reproduction

The recorded recipe targets the pinned listed-asset checkout:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git ens-manager
git -C ens-manager checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae
cd ens-manager
corepack pnpm install --frozen-lockfile

mkdir -p apps/manager/.ens78-poc/IMP-1
# Place the two supplied Manager fixture files in the directory above.

cd apps/manager
./node_modules/.bin/vitest run --configLoader runner \
  .ens78-poc/IMP-1/focus-authority-refresh.test.tsx \
  .ens78-poc/IMP-1/stale-executor-runtime.test.tsx
```

Expected result:

```
✓ focus-authority-refresh.test.tsx (1 test)
✓ stale-executor-runtime.test.tsx (1 test)
Test Files  2 passed (2)
Tests       2 passed (2)
```

The first fixture records the focus-refetch state split. It starts with the old resolver/manager, refetches a correct safe object with the same domain ID, and asserts that the ready plan still contains the removed authority.

The second test feeds the stale plan into the production migration executor. It decodes the request captured at the mocked transaction boundary and asserts:

* `from` is the victim;
* `to` is the victim's HCA;
* the nested target is the configured Migration Helper;
* the nested migration tuple contains the removed custom resolver;
* the same stale root also carries a resolver-role grant to the former manager;
* current V1 owner and resolver values are the safe replacements.

The former-manager assertion arises from the same missing-plan-identity root, not a separate report. It is not required for the bridge's immediate resolution consequence.

### Recorded Manager Execution

The included `manager-receipt-summary.json` summarizes the earlier isolated run. It identifies the original receipt by SHA-256 `977a0567c6163605cb79a20c9d48e72a76c7d39ecce26b545297dd3d763892f1`; it is a publication summary, not the original sealed record:

```
command: oracle:manager-vitest
exit code: 0
duration: 4014 ms
target pin: 1c9b47f18fcddd2e864dfe385c4171061c9811ae
target tree: 410ba8d6be340fc927271024ed55dd0c58ba9131ca54d655e8eb9cb12f6aff16
target before/after: 4d8fa18c8fc1c592bc7046e20f552690a295d107b3d2c9ae2a5a7cf5e841dab8 (identical)
network proof: ENETUNREACH
sealed request: 00059e918ea61a8389461e67fd173777f7c9e0086f2d94c55b2c327544c4c154
stdout: 6ec35bf6144f2e6ea1dc21bc34ea2728f3aee18cc723ce51ef825ab2ad02b464
```

### Supplemental exact-byte replay

`outer-calldata-check.json` records that the 1,028-byte HCA request used by `ManagerOuterHCABridge.t.sol` is byte-identical to the Manager oracle request:

```
status: PASS
byte_identical_to_manager_oracle_stdout: true
outer calldata SHA-256: 7ab9a32297792f541389b863f1a4aebf4ff33a56097b21da93d0bd006de7b318
```

The recorded command used Foundry's `--offline` option:

```bash
forge test --offline --match-contract ManagerOuterHCABridgeTest -vvv
```

`forge-output.txt` and `forge-receipt.json` record one bridge-specific test passing alongside 17 inherited contract tests. The bridge-specific test is `test_exactManagerOuterCalldataExecutesThroughCertifiedHCA()`. The bridge uses the first-party ENSv2 migration source identified in `source-provenance.txt`. The assertions establish the historical resolver address, restored former-manager permission, and immediate attacker-selected resolution through that same historical resolver without a later attacker transaction.

Execution provenance: the Manager receipt records network-namespace isolation. Forge's `--offline` option prevents missing compiler downloads; the Forge receipt does not establish network isolation. [Foundry compiler configuration](https://getfoundry.sh/reference/config/solidity-compiler/)
