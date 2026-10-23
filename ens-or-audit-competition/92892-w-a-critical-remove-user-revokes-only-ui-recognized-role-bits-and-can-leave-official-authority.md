# 92892 w a critical remove user revokes only ui recognized role bits and can leave official authority active

**Submitted on Sep 13th 2026 at 04:22:06 UTC by @BugNetter for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92892
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**

## Description

## Brief/Intro

The Portal converts authoritative resolver and registry role bitmaps into closed UI taxonomies, then builds **Remove user** revocations from only the represented roles. When an account holds a displayed role together with an official role that the UI does not represent, the removal transaction succeeds but the omitted role remains active. An administrator can therefore approve an apparently successful removal while the former delegate retains an official capability. This report is submitted as a Security Best Practices Insight and does not claim a Low-or-higher vulnerability.

The [unlisted supporting-evidence Gist](https://gist.github.com/Gonzo15918/22f3d2b5b33759ec1a7ea1806c6f3061) contains two source-bound Portal tests, preserved local-EVM receipts, a side-by-side root-cause trace, source provenance, and a SHA-256 ledger.

## Insight Details

### Shared root cause

Both removal paths reconstruct the revocation bitmap from a lossy UI projection rather than the account's authoritative raw bitmap. The resolver and registry demonstrations below use different contracts and roles, but they cross the same application boundary and have the same durable fix. They are two manifestations of one Insight, not two unrelated recommendations.

### Resolver manifestation: omitted `SET_DATA`

The Portal's [resolver role table](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L24-L45) omits the official `SET_DATA` role, and its [decoder iterates only that table](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L110-L127). The [removal mutation accepts the decoded role list](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L251-L276), and [`handleRemoveUser` stores only that list](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323). The [revoke helper encodes the supplied roles](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74), not the raw bitmap.

For an initial bitmap of `SET_ADDR | SET_DATA` (`0x1000000001`), the Portal displays `SET_ADDR` and prepares a revoke of only `0x1`. The official contracts define [`SET_DATA = 1 << 36`](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/resolver/libraries/PermissionedResolverLib.sol#L51-L54) and [authorize `setData` with that role](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/resolver/PermissionedResolver.sol#L424-L438). The local official-bytecode replay leaves `0x1000000000` active, after which the former delegate changes generic resolver data from `before-removal` to `after-removal`.

The state-matched control revokes the complete raw bitmap, leaves roles `0x0`, and rejects the identical later write with `EACUnauthorizedAccountRoles(uint256,uint256,address)` selector `0x4b27a133`. The official access-control implementation exposes [root grant, root revoke, and raw role lookup](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/access-control/EnhancedAccessControl.sol#L132-L171), while [`_revokeRoles` clears only the supplied bits](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/access-control/EnhancedAccessControl.sol#L284-L316).

### Registry manifestation: omitted `CAN_NAME`

The registry [bitmap decoder iterates only the installed ENSJS `registryRoles` table](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/decodeRoleBitmap.ts#L14-L25), and [`useRegistryRoles` converts the raw bitmap into that decoded list](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/registry/hooks/useRegistryRoles.ts#L69-L86). `CAN_NAME` is not represented. The registry sheet [derives `removableRoles` from the displayed roles](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/registry/components/v2/RegistryEditUserSheet.tsx#L161-L182), [passes that list into the revoke step](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/registry/components/v2/RegistryEditUserSheet.tsx#L200-L255), and the [helper encodes only those roles](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/registry/helpers/revokeRegistryRoles.ts#L49-L84).

For an initial bitmap of `SET_RESOLVER | CAN_NAME` (`0x1000000000000000000000001000000`), the Portal prepares `revokeRootRoles(0x1000000, delegate)`, leaving `0x1000000000000000000000000000000`. The official registry defines [`CAN_NAME = 1 << 120`](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/registry/libraries/RegistryRolesLib.sol#L55-L58) and implements [`isContractNamer` as a root-role check](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/registry/PermissionedRegistry.sol#L267-L270). The official adapter [routes `setName` through `requireNamer`](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/reverse-registrar/DefaultReverseRegistrarAdapter.sol#L48-L54), and [`AccountNamerLib` consults `isContractNamer` or reverts `UnauthorizedNamer`](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/reverse-registrar/libraries/AccountNamerLib.sol#L14-L45).

The local replay reports `isContractNamer(delegate) == true` after the partial removal, and the former delegate changes the registry-managed reverse label from `victim-managed.eth` to `former-delegate.eth`. The state-matched complete-revoke control leaves roles `0x0`, reports `isContractNamer(delegate) == false`, and rejects the same operation with `UnauthorizedNamer(address)` selector `0x0d1b7e4e`.

## Impact Details

This Insight is valuable wherever the Portal's role taxonomy is incomplete now or becomes incomplete after an official role is added. A successful **Remove user** action should mean that the selected account no longer has authority, but the current design can leave an unrepresented capability active without warning the administrator.

The demonstrated consequence is retained post-removal authority, not a pre-removal authorization bypass. The resolver mutation is generic `IDataResolver.data` storage and is not claimed to be a Portal-visible profile change. The registry storage boundary is mocked, although its authorization executes in pinned official adapter and UserRegistry bytecode. No production prevalence, affected-user count, funds loss, or Low-or-higher impact is claimed.

## Recommendation

Use an invariant-preserving removal design:

1. Read and revoke the authoritative raw bitmap for the selected account and resource; or
2. Refuse removal and display a precise warning whenever the raw bitmap contains unknown bits.

Add regression coverage for both mixed bitmaps. Merely adding `SET_DATA` and `CAN_NAME` to today's tables is incomplete because a later official role could recreate the same defect.

## References

* [Unlisted supporting-evidence Gist](https://gist.github.com/Gonzo15918/22f3d2b5b33759ec1a7ea1806c6f3061)
* [Resolver role table](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L24-L45) and [resolver bitmap decoder](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L110-L127)
* [Resolver removal handler](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L251-L276), [`handleRemoveUser`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L323), and [resolver revoke helper](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74)
* [Registry bitmap decoder](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/decodeRoleBitmap.ts#L14-L25), [`useRegistryRoles`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/registry/hooks/useRegistryRoles.ts#L69-L86), [registry removal selection](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/registry/components/v2/RegistryEditUserSheet.tsx#L161-L182), [registry revoke step](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/registry/components/v2/RegistryEditUserSheet.tsx#L200-L255), and [registry revoke helper](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/registry/helpers/revokeRegistryRoles.ts#L49-L84)
* Official ENS contract references: [`SET_DATA`](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/resolver/libraries/PermissionedResolverLib.sol#L51-L54), [`setData`](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/resolver/PermissionedResolver.sol#L424-L438), [root role operations](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/access-control/EnhancedAccessControl.sol#L132-L171), [`_revokeRoles`](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/access-control/EnhancedAccessControl.sol#L284-L316), [`CAN_NAME`](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/registry/libraries/RegistryRolesLib.sol#L55-L58), [`isContractNamer`](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/registry/PermissionedRegistry.sol#L267-L270), [reverse registrar adapter](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/reverse-registrar/DefaultReverseRegistrarAdapter.sol#L48-L54), and [`AccountNamerLib`](https://github.com/ensdomains/contracts-v2/blob/a399be8e8bcff572a56613c66aca7d4528437665/contracts/src/reverse-registrar/libraries/AccountNamerLib.sol#L14-L45)

## Link to Proof of Concept

<https://gist.github.com/Gonzo15918/22f3d2b5b33759ec1a7ea1806c6f3061>

## Proof of Concept

## Proof of Concept

The complete supporting bundle is available in this [unlisted Gist](https://gist.github.com/Gonzo15918/22f3d2b5b33759ec1a7ea1806c6f3061). It includes two source-bound Portal tests, preserved local-EVM receipts, a shared-root trace, official-source provenance, and a SHA-256 ledger.

## Conditions under which this Insight is valuable

1. A legitimate delegate holds a mixed bitmap containing one UI-represented role and one official role omitted from the UI taxonomy.
2. The Portal decodes and displays only the represented role.
3. An administrator selects **Remove user** and approves the displayed removal transaction.
4. The Portal constructs the revoke bitmap from the decoded list rather than the authoritative raw bitmap.
5. The transaction succeeds, but the omitted role remains active and can authorize a later operation by the former delegate.

This condition is security-relevant even without current production prevalence: a removal interface should either revoke the account's complete authority or explicitly warn that it cannot do so safely. Any present or future official role missing from the UI taxonomy can violate that invariant.

## Step-by-step demonstration

### A. Resolver manifestation

1. Give the delegate the root bitmap `SET_ADDR | SET_DATA` = `0x1000000001`.
2. Decode it through the Portal taxonomy. The displayed list contains `SET_ADDR` but omits official `SET_DATA`.
3. Select **Remove user**. The Portal component test decodes the prepared call and observes a revoke bitmap of only `0x1`.
4. Execute the equivalent partial revoke in the local official-bytecode replay. The call succeeds and leaves `0x1000000000` (`SET_DATA`) active.
5. Call `setData` as the former delegate. It succeeds and changes the generic data value from `before-removal` to `after-removal`.
6. Repeat from matched initial state, but revoke the complete raw bitmap `0x1000000001`. Roles become `0x0`; the same later `setData` call is rejected with selector `0x4b27a133`, and the prior value is preserved.

### B. Registry manifestation

1. Give the delegate the root bitmap `SET_RESOLVER | CAN_NAME` = `0x1000000000000000000000001000000`.
2. Decode it through the Portal/installed-ENSJS taxonomy. The displayed list contains `SET_RESOLVER` but omits official `CAN_NAME`.
3. Select **Remove user**. The Portal component test decodes the prepared call and observes `revokeRootRoles(0x1000000, delegate)`.
4. Execute the equivalent partial revoke in the local replay. The call succeeds, leaves `0x1000000000000000000000000000000`, and `isContractNamer(delegate)` remains true.
5. Call the official adapter's `setName` path as the former delegate. Authorization succeeds and the mocked storage boundary records a change from `victim-managed.eth` to `former-delegate.eth`.
6. Repeat from matched initial state, but revoke the complete raw bitmap. Roles become `0x0`, `isContractNamer(delegate)` becomes false, and the same later call is rejected with selector `0x0d1b7e4e`; the prior label is preserved.

The registry mock covers storage only. Authorization executes in the pinned official DefaultReverseRegistrarAdapter and UserRegistry bytecode.

## Reproduce the Portal transaction-construction results

Use a clean checkout of `https://github.com/immunefi-team/audit-comp-ens` at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, with its pinned dependencies already installed. Clone or download the Gist, then run:

```bash
export ENS_APP_REPO=/absolute/path/to/audit-comp-ens
export INSIGHT_GIST=/absolute/path/to/downloaded-gist
mkdir -p "$ENS_APP_REPO/apps/portal/.workflow-tests"
cp "$INSIGHT_GIST/INSIGHT-002_resolver_set_data.test.tsx" "$ENS_APP_REPO/apps/portal/.workflow-tests/"
cp "$INSIGHT_GIST/INSIGHT-002_registry_can_name.test.tsx" "$ENS_APP_REPO/apps/portal/.workflow-tests/"
corepack pnpm@10.27.0 --dir "$ENS_APP_REPO/apps/portal" exec vitest run \
  .workflow-tests/INSIGHT-002_resolver_set_data.test.tsx \
  .workflow-tests/INSIGHT-002_registry_can_name.test.tsx \
  --reporter=verbose
```

Expected result:

```
Test Files  2 passed (2)
Tests       2 passed (2)
```

The Gist's `COMPONENT_TEST_RECEIPT.txt` preserves the original passing run. `RESOLVER_EVM_RECEIPT.json`, `REGISTRY_EVM_DECISION_RECEIPT.json`, and `EVM_COMPARISON_RECEIPT.txt` preserve the local-EVM consequence and complete-revoke controls. The tests and replays make no RPC request and perform no real wallet connection, signature, broadcast, public-network deployment, or production-state mutation.
