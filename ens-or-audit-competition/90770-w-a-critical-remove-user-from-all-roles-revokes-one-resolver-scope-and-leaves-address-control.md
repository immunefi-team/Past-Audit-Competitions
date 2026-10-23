# 90770 w a critical remove user from all roles revokes one resolver scope and leaves address control authority active

**Submitted on Aug 27th 2026 at 02:32:44 UTC by @hmiya for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90770
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Impact

Select `ENS-IMPACT-11` (Critical):

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient — including resolving an ENS name to an attacker-controlled address.

Portal received one distinct non-Root resource row, which its accepted resource-to-name mapping resolves to the returned node's `SET_TEXT` scope, and one Root role row for the same operator from its accepted indexer endpoint. It grouped both rows into one account entry, promised **Remove user from all roles**, but composed only one name-scoped revoke intent, which the administrator wallet submitted. Root `SET_ADDR` remained active.

The removed operator then used Portal's real **Edit records** route to replace an ENS address. The administrator subsequently revoked every remaining role, but that correction did not revert the written record. A distinct owner later entered the poisoned ENS name in Portal's real Transfer form; Portal resolved and displayed the operator address and prepared `safeTransferFrom` with that address as `to`. The distinct-owner wallet submitted it, and the fork mined the name transfer to the unintended recipient.

This is not a claim that displaying a resolved address is itself wrong. The defect is Portal's false offboarding operation, which leaves the former operator able to make the later resolution attacker-controlled. The form shows the full address and an irreversible-transfer warning, but no signal that the record was written after Portal reported the delegate removed.

The administrator and harmed owner are independent actors. No phishing page, attacker-authored transfer calldata, hidden address, or victim collusion is required.

## Root cause

1. `resolverRoles.ts:179-221` groups raw `(account, resource, bitmap)` rows by account and unions their role bits.
2. `ResolverRolesSidebar.tsx:165-169` chooses one `roleName` for that grouped account.
3. `ResolverRolesSidebar.tsx:441-448` promises removal from **all roles**.
4. `revokeResolverRoles.ts:28-110` encodes one Root or one name-resource authorization change; it never expands the group back into every resource.
5. `EnhancedAccessControl` stores and revokes roles per resource.

For the observed name+Root input, Portal selected the name scope and encoded one name revoke. The transaction succeeded, the name bitmap became zero, and the Root bitmap remained. One successful transaction cannot clear both resource assignments.

With only one resource, the selected and assigned scopes match and removal clears the bitmap. With a resource-complete fix, the former operator cannot open the record editor. Those controls isolate the grouping-to-single-revoke composition as the cause.

## Decisive producer and end-to-end proof

`staging-indexer-resolver-overview-activation.json` is a frozen response to the accepted-source production `getResolverOverview` document at Portal's accepted default endpoint. For one account on a PermissionedResolver, the same response contains:

* one distinct non-Root resource row that Portal's accepted mapping resolves to the returned node and whose bitmap decodes to `SET_TEXT`; and
* one Root-resource `SET_ADDR | UPGRADE` row.

`staging-indexer-producer-evidence.md` records the read-only capture time, endpoint, query method, exact accepted-source query location and variable, response hash, block numbers, and transaction hashes. The fixture identifiers look synthetic; this proves producer reachability and the exact Portal input shape, not organic incidence or historical loss.

The literal-response arm in `resolverPoisonedTransferUiForkJoin.poc.test.tsx` carries that exact tuple through one pinned-fork snapshot:

{% stepper %}
{% step %}

## Grouped scopes and removal promise

The real roles route and Sidebar render the grouped scopes and remove-all promise.
{% endstep %}

{% step %}

## Incomplete production revoke

The production revoke request is submitted and mined.
{% endstep %}

{% step %}

## Surviving Root assignment

Production polling refreshes to the surviving Root assignment without a partial-removal warning or corrective transaction.
{% endstep %}

{% step %}

## Record update by stale non-owner

The stale non-owner opens Portal's real Edit-records route, passes its production role gate solely via retained Root `SET_ADDR`, edits the ETH address, and submits the production save request through the operator wallet.
{% endstep %}

{% step %}

## Remaining roles revoked

The owner revokes all remaining Root bits, while direct and universal resolution still return the operator address.
{% endstep %}

{% step %}

## Wrong-recipient transfer

A distinct owner enters that ENS name in the real Transfer form, sees the resolved operator address and warnings, submits the exact prepared `safeTransferFrom`, and loses ownership to the operator. Receipt, registry owner, and token balance are read back from the fork.
{% endstep %}
{% endstepper %}

No direct resolver write supplies the poison. The full-chain test uses real Portal record and transfer routes; separate controls submit production `saveRecords` and `transferToken` requests through the public transaction manager/EOA machines and verify their receipts and post-state.

## Minimum-privilege and recovery controls

A separate two-name/no-Root arm proves the same defect without any Root role. Its schema-faithful overview rows are built from measured same-fork role and node state, so it is minimum-privilege and ordinary-use corroboration rather than the sole producer-emission proof. It independently reaches incomplete revoke, Portal record poison, full role correction, persistent resolution, and the distinct-owner Transfer consequence.

Other controls prove:

* one assigned resource is fully removed;
* both row orders still omit one resource;
* the administrator's later complete role correction prevents future edits but cannot undo the already-written address or committed transfer;
* the surviving Root `UPGRADE` bit can authorize a UUPS upgrade on a bounded snapshot, although upgrade is not needed for the selected Impact11 chain; and
* a resource-complete removal is the direct fix-applied counterfactual.

## Reproduction

Run the exact six-file Vitest command in `README.md` against a disposable Sepolia fork pinned to block `11569705`, hash `0x39d322bf79b3c91d343f64c507ccd23055801125ec6519c606d1a24fafab3aee`.

Expected result:

```
Test Files  6 passed (6)
Tests       12 passed (12)
```

`test-run-transcript.md` binds the clean source head, runtime versions, source and test hashes, command, passing output, and pinned-fork identity. No public chain write or live application/indexer action occurs during the test.

## Why closed report #89844 does not recur

`#89844` was closed because its Vitest mocked resolver output, its Foundry test did not run Portal UI, its pagination premise was not shown through the real query/UI, and externally authorized state was merely displayed.

This report has no pagination premise or external state creator. It starts from a frozen accepted-endpoint response. Portal composes the incomplete revoke intent; the administrator wallet submits it and the fork mines it. Portal then admits the surviving operator through its real record editor; the operator wallet submits the production save request and the fork mines it. Portal later resolves the poisoned name in Transfer, and the distinct owner wallet submits the wrong-recipient transaction. Receipts and post-state come from one pinned-fork chain, not mocked resolver output.

`ENS-KNOWN-34`/QA-01 records a UI error case without a demonstrated security consequence and expressly says a role combination that grants unintended authority would be a new finding. Here Portal reports offboarding complete, leaves usable authority, and that authority is consumed into a committed third-party ownership transfer.

## Fix

Build revoke intents from the original per-resource rows, submit one revoke for every nonzero assignment, and report success only after all observed assignments are zero. If any assignment survives, show partial completion and the exact remaining scopes. Test both two-name and name+Root combinations in both row orders.

## Explicit non-claims

The synthetic staging identifiers do not establish prevalence, an organic victim configuration, historical loss, originality, duplicate status, or reward entitlement. The report claims one demonstrated, realistically reachable wrong-recipient transfer through the accepted Portal source and no severity multiplier from repetition or the optional upgrade consequence.

## Link to Proof of Concept

<https://gist.github.com/gok11/3f3a5f89383813fdf59842f7cc85763a>

## Proof of Concept

### Proof of Concept / Steps to Reproduce

The complete 29-file PoC package is supplied through the dedicated secret Gist field.

{% stepper %}
{% step %}

## Prepare the exact accepted source

```sh
git clone https://github.com/immunefi-team/audit-comp-ens.git /tmp/audit-comp-ens
cd /tmp/audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack pnpm install --frozen-lockfile

git clone https://gist.github.com/3f3a5f89383813fdf59842f7cc85763a.git /tmp/ens-resolver-multiscope-poc
```

Use Node.js 22.22.1 or newer (`JSON.rawJSON` must be available), pnpm/Corepack, and Foundry 1.7.1.
{% endstep %}

{% step %}

## Start a disposable pinned Sepolia fork

In a separate terminal:

```sh
export SEPOLIA_ARCHIVE_RPC="https://sepolia.gateway.tenderly.co"
anvil   --fork-url "$SEPOLIA_ARCHIVE_RPC"   --fork-block-number 11569705   --port 8547   --chain-id 11155111   --silent
```

The expected pinned block hash is:

`0x39d322bf79b3c91d343f64c507ccd23055801125ec6519c606d1a24fafab3aee`
{% endstep %}

{% step %}

## Run the six-file suite

```sh
cd /tmp/audit-comp-ens
export PACKAGE_DIR=/tmp/ens-resolver-multiscope-poc
export ENS_SOURCE_ROOT="$PWD"
export ENS_PORTAL_NODE_MODULES="$PWD/apps/portal/node_modules"

apps/portal/node_modules/.bin/vitest run   "$PACKAGE_DIR/resolverMultiscopeMinimal.poc.test.ts"   "$PACKAGE_DIR/resolverTransactionManagerCompletion.poc.test.tsx"   "$PACKAGE_DIR/resolverDownstreamTransactionManagerCompletion.poc.test.tsx"   "$PACKAGE_DIR/resolverPoisonedTransferUiForkJoin.poc.test.tsx"   "$PACKAGE_DIR/resolverRoleQuerySidebarForkJoin.poc.test.tsx"   "$PACKAGE_DIR/resolverAddUserSheetGrantPath.poc.test.tsx"   --config "$PACKAGE_DIR/resolverRoleQuerySidebarForkJoin.poc.config.ts"   --reporter=verbose
```

Expected result:

```
Test Files  6 passed (6)
Tests       12 passed (12)
```

{% endstep %}

{% step %}

## Observe the decisive chain

The literal-response full-chain arm proves, in one pinned fork snapshot:

1. Portal groups one name-resource row and one Root row for the same operator and presents **Remove user from all roles**.
2. The administrator submits Portal's production revoke request; only the selected name scope is cleared and Root `SET_ADDR` survives.
3. The stale non-owner passes Portal's real Edit-records authorization gate through the omitted Root role and submits the production record update.
4. The owner then revokes every remaining role, but the poisoned address record continues to resolve.
5. A distinct owner enters that ENS name in Portal's real Transfer form; Portal resolves and pre-fills the stale operator address.
6. The distinct-owner wallet submits the exact Portal-prepared `safeTransferFrom`; the receipt and post-state show ownership transferred to the unintended recipient.

The one-resource control clears fully, and the resource-complete fix arm prevents the record edit and downstream wrong-recipient transfer. No public-chain write is performed.
{% endstep %}
{% endstepper %}
