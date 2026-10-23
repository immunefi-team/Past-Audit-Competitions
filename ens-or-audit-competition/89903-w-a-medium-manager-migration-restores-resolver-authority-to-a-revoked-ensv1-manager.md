# 89903 w a medium manager migration restores resolver authority to a revoked ensv1 manager

## #89903 \[W\&A-Medium] Manager migration restores resolver authority to a revoked ENSv1 manager

**Submitted on Aug 21st 2026 at 18:21:53 UTC by @rtw0x1 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89903
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

### Description

### Brief/Intro

If a registrant prepares an ENSv1-to-ENSv2 migration while a distinct manager is current, then revokes or replaces that manager before execution, the Manager application can retain the old address in its ready migration plan. Continuing the normal migration sends `grantRoles(resource, ROLE_SET_RESOLVER, formerManager)` and restores persistent resolver-setting authority to the revoked address.

The root cause is entirely in the in-scope `apps/manager` application: its migration-plan query identity omits the name's manager address, and the execution path fails to revalidate that security-bearing value before building the transaction. The ENS contracts behave correctly and execute the application-supplied grant; they are used only as the downstream state oracle.

The attacker must control an address that was previously a legitimate ENSv1 manager for the exact name. The registrant must prepare the migration while that address is current, later revoke or replace it, keep the same selected-name workflow open, and approve the normal migration wallet prompts.

### Vulnerability Details

* Affected asset: `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager`
* Tested source revision: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`
* Vulnerable workflow: ENSv1-to-ENSv2 migration with manager restoration
* Restored privilege: `ROLE_SET_RESOLVER` (`1 << 24`)

The stale authority moves through the application as follows:

1. [`useMigrationGasEstimate`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L109) keys the ready migration plan by owner, HCA, stable selected domain IDs, and selection revision. It does not include `managerAddress` or an equivalent authority fingerprint.
2. If the same domain is refreshed after its ENSv1 manager changes, the selection revision and domain ID remain unchanged. The ready plan can therefore retain the former `classified[].managerAddress`.
3. [`buildRoleGrantCall`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L11-L24) encodes `grantRoles(resource, 1 << 24, name.managerAddress)`.
4. [`buildAtomicMigrationBatches`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624) creates the restoration call and verifier expectation from the retained `classified.managerAddress`.
5. [`executeMigration`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L1119-L1180) checks readiness and approvals but does not re-read the current manager or reject an authority change before submitting the plan.
6. [`verifyAtomicMigrationBatch`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353) then verifies `hasRoles` for the same stale address, so a successful wrong-recipient grant is accepted as the intended result.

The Manager confirmation text says **Approve manager restoration** and **Restore the existing managers for your names**, but it does not identify the recipient address or say that an address already revoked in the refreshed source state will be restored. A capable wallet may decode the calldata; the demonstrated issue is the stale application-supplied recipient, rather than universal concealment by wallets.

### Impact Details

The closest listed impact is:

> Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker.

The proof demonstrates the protected-state consequence rather than stopping at stale calldata:

* Before the Manager-produced grant, the former manager does not have `ROLE_SET_RESOLVER`, and an attempted resolver change reverts.
* Applying the exact application-produced tuple changes that role from false to true and persists it in the registry state.
* The former manager then successfully changes the name's resolver to an attacker-controlled resolver address.
* In the no-grant control, the same former-manager call still reverts and the original resolver remains unchanged.

This gives a revoked former manager durable control over which resolver serves the migrated name. I rate the finding High because the demonstrated result is restoration of revoked authority followed by a protected resolver replacement. Exploitation requires a former legitimate manager of the exact name, the still-open stale workflow, and normal registrant wallet approvals; those constraints materially reduce exploitability.

### Remediation

1. Include every security-bearing per-name input in the migration-plan identity, especially the current manager address and registrant/manager relationship.
2. Invalidate and rebuild a ready plan whenever refreshed authority state changes, even if the domain ID and selected-name set are unchanged.
3. Immediately before wallet submission, re-read the current manager for every name and reject the plan if it differs from the value used to build any role grant.
4. Build the execution batch only from the freshly validated source state; do not accept a previously classified manager recipient.
5. Show the exact name, recipient address, and role being restored in the Manager confirmation UI.
6. Make post-transaction verification compare the granted recipient with the freshly validated current manager, not only with the submitted stale plan.

### References

* [In-scope Manager source at the tested revision](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager)
* [Migration-plan query identity](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L109)
* [Role-grant call construction](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L11-L24)
* [Execution-time manager restoration](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624)
* [Missing execution-time authority revalidation](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L1119-L1180)
* [Verification of the stale recipient](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337-L353)
* [Manager restoration confirmation text](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L80-L89)
* [Exact downstream registry revision referenced by the Manager](https://github.com/ensdomains/contracts-v2/tree/892311a7268bf6051d5fe65740f1eebfaf8db431)

### Link to Proof of Concept

<https://gist.github.com/rtw0x1-bb/35f95ffadfd7198c57dee1a2bc539450>

### Proof of Concept

## Proof of Concept

### Prerequisites

* Linux with Git, Node.js 22, Corepack, and Foundry installed.
* The two PoC source files in the linked secret Gist:
  * `stale-manager-integrated-lifecycle-reducer.test.tsx`
  * `StaleManagerRegistryStateParity.t.sol`
* Secret Gist: `https://gist.github.com/rtw0x1-bb/35f95ffadfd7198c57dee1a2bc539450`
* No RPC endpoint, funded wallet, live ENS name, or production transaction is required. The competition is source-only.

The TypeScript test uses the real Manager plan hook, plan builder, execution batch builder, role-grant encoder, `executeMigration`, and post-transaction verifier. It mocks wallet transport, the receipt, and post-state reads so that no chain transaction is sent. The Solidity test applies the exact resulting `grantRoles` tuple to the official registry implementation as a correct downstream state oracle. The two tests meet at the deterministic `resource`, `1 << 24`, and former-manager recipient tuple.

### Steps to Reproduce

#### 1. Download the exact PoC files and create a clean application checkout

```bash
export ENS_POC_ROOT="$(mktemp -d /tmp/ens-manager-stale.XXXXXX)"
mkdir -p "$ENS_POC_ROOT/input"
curl -fsSLo "$ENS_POC_ROOT/input/stale-manager-integrated-lifecycle-reducer.test.tsx" \
  "https://gist.githubusercontent.com/rtw0x1-bb/35f95ffadfd7198c57dee1a2bc539450/raw/522d9c3c2d3f62f513148f14e5619de30fc47d15/stale-manager-integrated-lifecycle-reducer.test.tsx"
curl -fsSLo "$ENS_POC_ROOT/input/StaleManagerRegistryStateParity.t.sol" \
  "https://gist.githubusercontent.com/rtw0x1-bb/35f95ffadfd7198c57dee1a2bc539450/raw/97476c650571cf2b62ba4c08a2fb4317af7016c6/StaleManagerRegistryStateParity.t.sol"
cd "$ENS_POC_ROOT"
sha256sum --check <<'EOF'
5159391769ccb4b78e6a46cbf25a8f7318496dc898606bee07d629bfa33f3582  input/stale-manager-integrated-lifecycle-reducer.test.tsx
5dd56856a33e9c1fafcd98cc526fa11937c3eba784629bde94b707ef7a26ee71  input/StaleManagerRegistryStateParity.t.sol
EOF
git clone --filter=blob:none https://github.com/immunefi-team/audit-comp-ens.git "$ENS_POC_ROOT/audit-comp-ens"
git -C "$ENS_POC_ROOT/audit-comp-ens" checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae
cd "$ENS_POC_ROOT/audit-comp-ens"
corepack pnpm install --frozen-lockfile
```

Expected result: both PoC files report `OK`, the checkout reports exact HEAD `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, and dependencies install successfully.

#### 2. Run the Manager lifecycle proof

```bash
install -m 0644 "$ENS_POC_ROOT/input/stale-manager-integrated-lifecycle-reducer.test.tsx" "$ENS_POC_ROOT/audit-comp-ens/apps/manager/src/features/migration/service/stale-manager-integrated-lifecycle-reducer.test.tsx"
cd "$ENS_POC_ROOT/audit-comp-ens/apps/manager"
corepack pnpm exec vitest run src/features/migration/service/stale-manager-integrated-lifecycle-reducer.test.tsx --silent --reporter=verbose
```

Expected result: one test passes. The assertions prove that:

* the plan initially records `0x2222222222222222222222222222222222222222` as manager;
* refreshed same-domain state says the registrant is now the manager, but the same ready plan is retained;
* the real batch decodes to `grantRoles(resource, 1 << 24, 0x2222...2222)`;
* `executeMigration` submits the rebuilt outer call and the real verifier queries `hasRoles` for that revoked recipient;
* after a genuine selection revision, a fresh plan has `managerAddress === null` and emits no manager-restoration call.

Raw terminal capture: `step-02-manager-stale-plan-and-role-grant.png`.

#### 3. Run the persistent registry-state proof

```bash
git clone --filter=blob:none https://github.com/ensdomains/contracts-v2.git "$ENS_POC_ROOT/contracts-v2"
git -C "$ENS_POC_ROOT/contracts-v2" fetch origin refs/pull/388/head
git -C "$ENS_POC_ROOT/contracts-v2" checkout --detach 892311a7268bf6051d5fe65740f1eebfaf8db431
git -C "$ENS_POC_ROOT/contracts-v2" submodule update --init --recursive
install -m 0644 "$ENS_POC_ROOT/input/StaleManagerRegistryStateParity.t.sol" "$ENS_POC_ROOT/contracts-v2/contracts/test/unit/registry/StaleManagerRegistryStateParity.t.sol"
cd "$ENS_POC_ROOT/contracts-v2/contracts"
forge clean
forge test --match-path test/unit/registry/StaleManagerRegistryStateParity.t.sol -vv
```

Expected result: both tests pass:

```
[PASS] test_exactManagerGrantPersistsRoleAndEnablesResolverChange()
[PASS] test_withoutManagerGrantFormerManagerCannotChangeResolver()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

The positive test first proves that the former manager is unauthorized, then applies the exact Manager-produced `grantRoles` call and proves the role persists and `setResolver` succeeds. The control omits only the Manager grant and proves that `setResolver` reverts and the original resolver remains unchanged.

Raw terminal capture: `step-03-former-manager-resolver-control.png`.

#### 4. Clean up

```bash
cd /tmp
test -n "${ENS_POC_ROOT:-}"
case "$ENS_POC_ROOT" in /tmp/ens-manager-stale.*) rm -rf -- "$ENS_POC_ROOT" ;; *) exit 1 ;; esac
unset ENS_POC_ROOT
```

### Proof of Impact

The two PoC tests in the linked secret Gist produce a source-to-state proof at fixed revisions:

1. The in-scope Manager retains a former manager after current same-name authority data changes.
2. Its real execution path encodes and submits a `ROLE_SET_RESOLVER` grant to that former address and verifies the same wrong recipient.
3. Applying that exact tuple to the registry state changes the former manager from unauthorized to authorized.
4. The former manager can then replace the resolver; the no-grant control cannot.

The attached images are unmodified captures of the actual test commands and results. No live ENS, RPC, wallet, or chain state was touched.

## PoC Source Files

The complete runnable source files are in the linked secret Gist:

`https://gist.github.com/rtw0x1-bb/35f95ffadfd7198c57dee1a2bc539450`

1. `stale-manager-integrated-lifecycle-reducer.test.tsx` — complete application lifecycle test.
2. `StaleManagerRegistryStateParity.t.sol` — complete persistent-state and negative-control test.

## Screenshot Attachments

1. `step-02-manager-stale-plan-and-role-grant.png` — raw application test terminal output.
2. `step-03-former-manager-resolver-control.png` — raw registry-state test terminal output.
