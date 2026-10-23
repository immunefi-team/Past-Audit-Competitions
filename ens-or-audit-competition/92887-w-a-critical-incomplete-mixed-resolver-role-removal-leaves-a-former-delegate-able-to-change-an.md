# 92887 w a critical incomplete mixed resolver role removal leaves a former delegate able to change an ens address record

**Submitted on Sep 13th 2026 at 03:42:43 UTC by @BugNetter for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92887
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Changing details of other users (including modifying browser local storage) without already-connected wallet interaction and with significant user interaction, such as:
* Iframing leading to modifying the backend/browser state (must demonstrate impact with PoC)

## Description

## Brief description

Portal combines a delegate's root resolver role and `n.eth` role in one `Global roles (all names)` account row and offers to remove the delegate from all roles. In the mixed-role case, the multi-step removal flow prepares and accepts only `authorizeNameRoles(n.eth, 0x11, T, false)`, clearing the name-local role while leaving T's root `ROLE_SET_ADDR` authority effective. After the administrator completes removal, the former delegate can change `n.eth`'s address record. The local proof maps this to Low impact 53 because the later change occurs without an already-connected wallet interaction and after significant user interaction.

## Vulnerability details

### Location

The affected application code is `apps/portal` at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

The relevant source locations are:

* `apps/portal/src/lib/roles/resolverRoles.ts`, lines 179-220, where role records are keyed by account and decoded roles are flattened across resources.
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx`, lines 165-193, 295-323, 389-396, 441-447, and 486-501, where the mixed row is presented as global/all-role removal, a non-root name is selected, and the action is started.
* `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts`, lines 29-74, where a nonempty name selects the name-scoped ENSjs revoke path.

### Root cause

`groupRolesByAccount` loses resource provenance when it merges records for the same account. The sidebar then treats a row that includes `(root)` as a global role assignment but chooses the first non-root name for the pending action. The helper interprets that nonempty name as an instruction to revoke only on that name resource.

### Trigger and resulting state

The local fixture gives T a root `ROLE_SET_ADDR` bitmap of `0x01` and an `n.eth` `ROLE_SET_TEXT` bitmap of `0x10`. An honest, authorized administrator reviews the single grouped row and completes Portal's multi-step removal path. The component test reaches `transactionManager.startTransaction` after Remove, Remove, Start, and Open wallet. The terminal manager and wallet boundary are mocked, so this is not evidence of a signature, broadcast, or wallet-specific display.

The prepared request is `authorizeNameRoles(n.eth, 0x11, T, false)`. The two supplied offline PyEVM receipts record that this request is accepted, changes the name bitmap from `0x10` to `0x00`, and leaves root bitmap `0x01`. Root fallback keeps `ROLE_SET_ADDR` effective for `n.eth`; T then successfully calls `setAddr` and changes the stored address to `0x1212121212121212121212121212121212121212` in the local lab.

The name-local `ROLE_SET_TEXT` bit is removed by the demonstrated call. The supplied CR-003 evidence demonstrates the later `setAddr` change, not a later `setText` change. The complete-revoke control is decisive: `revokeRootRoles(0x01, T)` is accepted and the same later `setAddr` attempt reverts.

T was legitimately delegated these roles before the administrator attempted removal. After the name-only revoke, T remains contract-authorized only because the represented root role was not revoked; the administrator and T are distinct accounts. The defect leaves authority that the completed removal flow represented as revoked.

## Impact

**Impact (exact program wording):** `Changing details of other users (including modifying browser local storage) without already-connected wallet interaction and with significant user interaction` (impact 53, Low).

The later record change is made after the administrator's removal process, without an already-connected wallet interaction at the time of that later change. It is not a Medium finding because the relevant honest-administrator action is not at-most-one-click: it requires the Portal removal sequence and wallet confirmation. The resolver and removal subject are the intended target of the action.

This report claims only the demonstrated local address-record integrity change under impact 53.

## Severity rationale

**Assessed severity: Low.** Impact 53 is classified Low, and the proof satisfies its significant-interaction condition through the administrator's multi-step removal and wallet confirmation.

## Recommendation

Keep resource provenance from the grouped row through request construction. For a mixed root and name-scoped row, emit one revoke for each represented resource, including `revokeRootRoles` for root authority, or prevent bulk removal and require a resource-scoped review that matches the exact requests.

Review text, selected resources, role bitmaps, and prepared calls should describe the same set. Add mixed root-plus-name coverage alongside root-only and name-only controls, and assert that completed removal blocks the former delegate's record write.

## References

* [Account grouping and flattened role decoding](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220).
* [The sidebar selects the first non-root name and flattens the grouped roles into one permission set](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L193).
* [The removal action stores that selected name and the flattened roles in one pending request](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323).
* [The removal mutation forwards the pending name, account, and roles to `revokeResolverRoles`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L251-L276).
* [A row containing the root resource is displayed as `Global roles (all names)`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L389-L396).
* [The confirmation dialog represents the action as removing the user from all roles](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L447).
* [Transaction start executes the pending removal action](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L486-L501).
* [A nonempty name selects the name-scoped revoke path instead of the root-scoped path](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74).
* Local evidence: `CR-003_account_resource_grouping.test.tsx`, SHA-256 `c6f6a3e122d15cb996d5de70c668a0db79f87480e78d3eb6b1589deb5ace7590`.
* Local execution receipt: `COMPONENT_TEST_RECEIPT.txt`, SHA-256 `da66a568a866cce8ff6ca6112ca9ed793f260fe0c765c0e05a5d2d7412d83d2c`.
* Local evidence: `LOCAL_EVM_RECEIPT.json` and `LOCAL_EVM_RECEIPT_RERUN.json`, the two preserved offline consequence and control receipts.

## Link to Proof of Concept

<https://gist.github.com/Gonzo15918/4b5c71c7e23bf5d0f05d51b3f41a872f>

## Proof of Concept

## Proof of concept

The complete reviewer bundle is available in the [CR-003 proof-of-concept Gist](https://gist.github.com/Gonzo15918/4b5c71c7e23bf5d0f05d51b3f41a872f). Its `README.md` explains the evidence, file roles, integrity check, and reproduction procedure.

The bundle contains the TypeScript/Vitest component test and two PyEVM receipt ledgers. The component test covers the actual sidebar and transaction-modal route through the mocked transaction-manager boundary, with the divergent mixed case plus aligned root-only and name-only controls. The ledgers record the accepted name-only call, retained root authority, later address-record write, and complete-revoke control.

Use a bash terminal and a clean frozen Portal checkout at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae` that already has its dependencies installed. From a cloned or downloaded copy of the Gist, copy `CR-003_account_resource_grouping.test.tsx` into that checkout as `apps/portal/poc-tests/CR-003_account_resource_grouping.test.tsx`, then run:

```bash
cd apps/portal
./node_modules/.bin/vitest run poc-tests/CR-003_account_resource_grouping.test.tsx --reporter=verbose
```

Expected result:

```
Test Files  1 passed (1)
Tests       3 passed (3)
```

Do not use an RPC endpoint, a real wallet, a signature, or a broadcast. `LOCAL_EVM_RECEIPT.json` and `LOCAL_EVM_RECEIPT_RERUN.json` are preserved results from two offline in-memory PyEVM executions, not a claim that this package freshly reruns them. `lead007_permissioned_resolver_evm.py` documents the pinned local-lab procedure and requires the exact clean application and contracts checkouts plus preinstalled local Python packages.

## Evidence and limits

`COMPONENT_TEST_RECEIPT.txt` records three passing cases in local jsdom. The two PyEVM receipts are `PASS`, used no network during EVM execution, and normalize identically after removing only `executed_at`. Their source identity binds the application commit, ENSjs pin `e96662c`, and contracts commit `a399be8e8bcff572a56613c66aca7d4528437665`.

The finding would be falsified by a source-faithful mixed-role execution that submits a root or multi-resource revoke, rejects the demonstrated name-scoped request, clears or disables root authority, or permits the later address write after the complete root-revoke control.
