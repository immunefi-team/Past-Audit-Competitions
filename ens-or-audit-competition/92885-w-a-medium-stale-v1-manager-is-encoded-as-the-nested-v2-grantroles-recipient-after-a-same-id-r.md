# 92885 w a medium stale v1 manager is encoded as the nested v2 grantroles recipient after a same id refresh

**Submitted on Sep 13th 2026 at 03:14:19 UTC by @BugNetter for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92885
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief

The in-scope Manager app can retain a migration plan built for former Manager A after the authoritative V1 result for the same domain ID and registrant changes to current Manager B. An ordinary Upgrade carries the stale plan through the production migration and transaction path to a local, non-broadcast Viem boundary. The intended HCA remains the outer target, while former Manager A is encoded as the nested V2 `grantRoles` recipient and transaction argument; the fresh control encodes B. This maps to the competition's Critical C11 incorrect-transaction-detail category and is limited to the ENS audit competition.

## Vulnerability Details

### Location

`apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts:81-109` and `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:11-26` at frozen app commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

### Root cause

The migration query and plan identity includes owner, HCA, selected domain IDs, and an ID-based revision, but omits the mutable manager identity. A plan created when the V1 result names A can therefore remain ready after a same-ID refresh names B. `buildRoleGrantCalls` then encodes the planned manager address in the production `grantRoles(resource, 1 << 24, managerAddress)` call.

### Trigger and exact tuple

The preserved test obtains an A-built plan from the mounted production V1 query and page path, then refreshes the same domain ID and registrant to B. It starts the ordinary Upgrade action without an intervening mandatory rebuild. The captured stale request contains this tuple:

* Outer target, intended HCA: `0x1000000000000000000000000000000000000002`
* Nested target, V2 Registry: `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`
* Resource: `70564938991660933374592024341600875602376452319261984317470407481572850663424`
* Role bitmap: `16777216` (`1 << 24`)
* Incorrect nested recipient and `grantRoles` argument, former Manager A: `0x6813Eb9362372EEF6200f3b1dbC3f819671cBA69`
* Fresh control recipient, current Manager B: `0x1efF47bc3a10a45D4B230B5d10E37751FE6AA718`

Stale outer and inner hashes: `0x503936a4742675d1498fef677a678b4f3cddd25fd3bc44bb3ccce370a9768d09` and `0x4861b8b87882a1264cf6177c11b557f99a2ed9dd48d5b3fd27f0ae62067ecd5a`. Fresh-B outer and inner hashes: `0xa09a74ce88272afc011f6b99abb945902c0129d8f297a52207f92e893bba3a8b` and `0x9ec39ed60fcd97a2b15ae9856aaaffc444f76f26f53e48c09d5f5b036fd2cab9`.

### Transport and consequence evidence

The test reaches the real migration service, transaction manager, transaction machine, EOA transport, and Viem wallet action. Its terminal adapter is deliberately local and non-broadcast. It rejects a missing local acknowledgement with `LOCAL_APPROVAL_REQUIRED` and a one-byte outer-data mutation with `LOCAL_BYTES_MISMATCH`, both before capture. With acknowledgement, it captures the exact stale or fresh bytes.

The separate local PyEVM lab verifies both hashes and that each inner calldata sequence occurs verbatim within its outer request, then sends the unchanged nested bytes to initialized proxy fixtures using pinned official `UserRegistryImpl` bytecode. Stale bytes grant A the resolver role and block B; A then changes `alice`'s resolver. Fresh bytes produce the inverse result for B.

A supplemental Foundry proof executes the exact captured outer bytes without decoding or re-encoding them. It uses the pinned `StandaloneSingleOwnerHCA` proxy/implementation, `MigrationHelper`, `UnlockedMigrationController`, ENS V1 registrar/name wrapper, and `PermissionedRegistry` semantics at the captured destinations. The stale request completes migration, grants A the resolver-management role, lets A change `alice`'s resolver, and blocks B. The fresh request produces the inverse for B. An unauthorized caller is rejected without registry mutation, a one-byte change fails the capture hash gate, and the complete four-test suite passes twice.

## Impact Details

The exact frozen C11 wording is: “Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves”.

The outer transaction is correctly addressed to the user's HCA, but its calldata contains former Manager A as the unintended recipient of the nested V2 `grantRoles` call. When the registrant selects Upgrade, the Manager carries this stale argument to the wallet-submission boundary. This matches C11's incorrect-transaction-detail impact and the program's explicit concern about transactions constructed with incorrect target addresses or arguments.

The recorded complete local outer execution establishes resolver authority through the pinned HCA and migration stack. It does not establish direct theft, smart-account takeover, C12 name hijacking or bricking, a live deployment result, a user count, or any monetary loss.

Exploitation requires the registrant to select Upgrade while the stale plan remains ready; former Manager A cannot initiate or sign the transaction. An explicit refresh, reset, or rebuild produces the correct Manager B transaction.

## Recommendation

Include the current manager identity in the migration-plan and query identity. Immediately before request construction, compare the planned grant recipient with current V1 data and rebuild or reject if they differ. As defense in depth, show the nested recipient and role bitmap in the review step.

## References

* [useV1Names.ts#L12-L29](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L29) keys the V1-name query only by the active owner or wallet address and returns that address-keyed query.
* [useMigrationGasEstimate.ts#L62-L109](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L109) derives identity from domain IDs and keys the plan query by owner, HCA, those IDs, and the ID-based selection revision, but not manager identity.
* [MigrationPage.tsx#L259-L283](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/pages/MigrationPage.tsx#L259-L283) requires the estimate to be ready and sends its retained `gasEstimate.plan` into `migration.start` from `handleBeginUpgrade`.
* [buildRoleGrantCalls.ts#L11-L26](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L11-L26) encodes `managerAddress` as the `grantRoles` recipient.
* [buildAtomicMigrationBatches.ts#L608-L624](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624) carries the role grant and matching manager-role expectation into the atomic plan.
* [migrationUi.machine.ts#L150-L163](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/state/migrationUi.machine.ts#L150-L163) directly captures the supplied plan; this assignment performs no current-manager equality check.
* [migrationService.ts#L886-L937](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L886-L937) builds the next atomic batch from the remaining classified names and profiles retained in the plan.
* [migrationService.ts#L1044-L1105](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L1044-L1105) submits each generated outer call through `submitCall`.
* [migrationService.ts#L210-L250](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L210-L250) copies the outer call's `to`, `data`, and `value` into an EOA request and passes it to the transaction manager as a custom intent.
* [transactionManager.ts#L176-L251](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/providers/transactionManager.ts#L176-L251) recognizes the custom intent, supplies it and the signer to the production transaction actor, and starts that actor.
* [transaction.machine.ts#L305-L316](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/transaction.machine.ts#L305-L316) extracts the custom intent's embedded request directly into transaction-machine context.
* [transaction.machine.ts#L107-L170](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/transaction.machine.ts#L107-L170) passes that request and the EOA signer directly to `submitEOATransaction`.
* [eoa-transport.actor.ts#L22-L80](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/actors/eoa-transport.actor.ts#L22-L80) copies `to`, `value`, and `data` into Viem's wallet action after the signer check.

## Link to Proof of Concept

<https://gist.github.com/Gonzo15918/ca024cfb0593ed839a7b1381805e5f3f>

## Proof of Concept

## Proof of Concept

PoC bundle: [secret GitHub Gist](https://gist.github.com/Gonzo15918/ca024cfb0593ed839a7b1381805e5f3f)

The PoC bundle preserves the already executed TypeScript/Vitest transport test and four exact evidence artifacts. The host test SHA-256 is `630969281956a1e595a27defe88d640cfca4413f5567e983fa3dd49e19da1ee5`. Its recorded result is one passing test in 4.58 seconds. It is not a standalone one-file runner: it must be placed in the existing dependency-installed Manager checkout at the frozen app commit. No dependency installation is required.

In a bash terminal, set `APP_CHECKOUT` to that frozen checkout and `GIST_DIR` to the directory containing the cloned or downloaded Gist files. Then run:

```bash
cd "$GIST_DIR"
sha256sum -c source_hashes.sha256

cd "$APP_CHECKOUT/apps/manager"
mkdir -p .workflow-tests
cp "$GIST_DIR/enhanced_poc.test.tsx" .workflow-tests/CR-001_exact_transport_bridge.test.tsx
TZ=UTC CR001_CAPTURE_PATH="$PWD/.workflow-tests/CR-001_transport_captures.generated.json" node_modules/.bin/vitest run .workflow-tests/CR-001_exact_transport_bridge.test.tsx --reporter=dot
```

Expected host output is one passing test. It exercises the source-authentic same-ID A-to-B reachability, missing-acknowledgement rejection, one-byte-tamper rejection, distinct stale and fresh raw hashes, and the fresh-B inverse control.

The PyEVM lab must run only in a separately prepared local environment that has the pinned contracts checkout at `a399be8e8bcff572a56613c66aca7d4528437665` and the already installed local Python dependencies. It is not part of the Manager checkout and does not use a live RPC:

```bash
cd "$GIST_DIR"
"$PYEVM_PYTHON" CR-001_exact_nested_bytes_evm.py --app-repo "$APP_CHECKOUT" --contracts-repo "$CONTRACTS_CHECKOUT" --captures CR-001_transport_captures.json --json-out CR-001_exact_nested_bytes_evm_receipt.generated.json
```
