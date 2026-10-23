# 92746 w a critical ens explorer s remove user from all roles leaves active address control permission on another name

**Submitted on Sep 11th 2026 at 23:56:32 UTC by @thudlow for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92746
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief / Intro

ENS Explorer combines a delegate's permissions across multiple names into one row with one **Remove user** action. When the administrator confirms that the delegate will be removed “from all roles,” Portal constructs only one name-scoped revoke transaction.

If the delegate has `ROLE_SET_ADDR` on two names sharing the same `PermissionedResolver`, the successful transaction clears one name and leaves the other permission active. The delegate can then use that retained permission to replace the other name's Ethereum address. This violates the security postcondition presented to the administrator: completing the row-level removal does not remove the permissions represented by that row.

This is an `apps/portal` transaction-construction defect. The resolver contract correctly executes the one-name calldata it receives and is used only as a local consequence oracle.

## Preconditions

* Two names controlled by the same administrator use one permissioned resolver.
* The same delegate has name-scoped `ROLE_SET_ADDR` on both names. Explorer's ordinary **Add user** flow can create this state by selecting each name in turn.
* Both grants are present in the resolver overview and both resource nodes resolve into names in `resolvedNames` when removal begins.
* The remover is the resolver's initial/root EAC administrator, or another account with the effective admin roles required to revoke the grants (for the headline case, `ROLE_SET_ADDR_ADMIN`), and has a connected wallet.
* The remover passes Portal's `canManageRoles` gate so editing is exposed, and is distinct from the delegate because the table disables self-editing.

The delegate needs no root role, owner key, wallet access, key compromise, ownership transfer, or control over which indexed role row appears first.

## Vulnerability Details

The frozen target is the ENS audit-competition Explorer source under `apps/portal`. Line references below use competition import `cda79acaad59711b943fc68207ebb3f1d0ff8596`, which identifies audited upstream revision `63772fd872af472ced58b009499355f3430c2a86`. The affected Portal files are unchanged at current competition branch head `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

### 1. The ordinary UI permits multiple name-scoped grants to one account

`apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:183-205` accepts any resolved account, selected name, and non-empty role set. Its name selector lists every name associated with the resolver at lines 248-289, and the prepared grant uses exactly that selection at lines 316-339. There is no guard against reopening the sheet and granting the same account a role on another name.

### 2. Explorer collapses distinct resources by account

`apps/portal/src/lib/roles/resolverRoles.ts:179-220` keys `groupRolesByAccount` only by the lower-cased account. When another assignment for that account arrives, lines 201-204 append its raw resource, append its decoded roles to one flat array, and add its name to one set.

`apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:102-115` passes that per-account output to one editable table row. The table does visibly list both names, but there is only one edit/remove action for the merged row.

### 3. The Sidebar chooses one resource but retains the row-wide role union

In `ResolverRolesSidebar.tsx:165-170`, the component reads the merged `decodedRoles` and `resolvedNames`, then selects one write scope:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The **Remove user** confirmation at lines 441-463 says:

```
Are you sure you want to remove this user from all roles?
This action cannot be undone.
```

But `handleRemoveUser` at lines 307-323 builds one action containing only `name: roleName` plus the flattened `roles: decodedRoles`. The transaction title at lines 338-349 is only `Remove user <account>` and does not enumerate the selected name or omitted resources.

The modal's production preparation and execution paths at lines 467-501 both consume that same single-name action.

### 4. The helper encodes one name-scoped revoke

`apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts:32-47` passes the selected name and unioned roles to one `prepareRevokeResolverRolesTransaction` call.

`apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-74` chooses either root or one name scope and produces one EOA intent. In the pinned ensjs dependency, `src/actions/wallet/v2/revokeResolverRoles.ts:121-132` encodes:

```solidity
authorizeNameRoles(selectedName, OR(allRowRoles), delegate, false)
```

There is no loop or second descriptor for any other raw resource stored in the merged row.

### 5. The successful call changes only the selected name resource

The exact resolver implementation from pinned contracts-v2 commit `8d1c893` uses `PermissionedResolver.sol:273-293` to derive one resource from the supplied DNS name and revoke only that resource. `EnhancedAccessControl.sol:290-315` computes `currentRoles & ~roleBitmap` for that resource, so union bits absent from the selected resource are harmless no-ops and the call can return success instead of revealing that another name was omitted.

The address setter at `PermissionedResolver.sol:665-681` accepts a caller with the surviving name's `ROLE_SET_ADDR`. A local exact-contract replay proves that the delegate can change and read back the other name's Ethereum address after the Portal-equivalent revoke.

## Impact Details

This is an incomplete authorization revocation. An administrator uses Explorer's security control to terminate a delegate's displayed access, signs the transaction prepared by Explorer, and receives a successful result while the delegate retains usable authority over another name. The retained `ROLE_SET_ADDR` permission allows the delegate to redirect that name's resolution after the removal operation.

Custom Web & App impact `6288` applies because Explorer presents an all-resource authorization change to an already-connected wallet while constructing transaction parameters scoped to only one of those resources.

The administrator retains authority to revoke the remaining role once the incomplete removal is discovered. No ownership or root-role takeover is claimed.

## Suggested Fix

Preserve resource boundaries all the way from the indexed assignments to the signed transaction.

Either:

1. group/edit rows by `(account, resource)` and make **Remove user** explicitly remove only that displayed resource; or
2. keep a per-account row but build one revoke for every raw resource, using that resource's own role bitmap and exact name/root scope.

For an all-resource action, the confirmation and transaction review should list every name/root scope. Do not derive a single scope from `resolvedNames[0]` or use one union bitmap as a substitute for resource assignments. After receipt and refresh, verify that the account has zero roles on every resource covered by the action; otherwise keep the flow open and warn that removal is incomplete.

Regression tests should include:

* the same account with `ROLE_SET_ADDR` on two names in both input orders;
* different role sets on two names;
* root plus name assignments;
* root-only and single-name controls;
* a postcondition that every intended resource is actually cleared.

## Scope and Knownness

The vulnerable asset is the in-scope **Explorer app Files** target. Smart contracts and the indexer are not claimed as defective.

No same-root public item was found in the competition's known-issue catalogue or focused ENS issue/PR searches. Nearby public items are distinct:

* `contracts-v2#418` concerns delegated roles surviving a name ownership transfer in contract semantics;
* `contracts-v2#272` concerns a proposed revoke/transfer ordering race;
* `ensjs#313` corrects a registry role-resource read ID;
* `ensjs#378` adapts ensjs to a future resolver API.

None describes account-only Portal grouping followed by a one-name revoke from an all-role removal flow.

## Link to Proof of Concept

<https://gist.github.com/majesticio/1997b43feae107a27b74fda4db5742e3>

## Proof of Concept

## Proof of Concept

The retained PoCs below used synthetic local state, the frozen Explorer source, and separately pinned contracts-v2 source. The later product-path attempt used read-only public Sepolia state hydration for local Anvil, but every transaction remained local and synthetic. No hosted ENS target, real name, wallet session, credential, public-testnet transaction, private or user-specific third-party data, callback, fund, or denial of service was used.

The PoC has two parts:

1. a rendered React test that clicks the production Sidebar removal flow and decodes its real prepared transaction; and
2. an exact-contract test that proves the omitted name's role and address-write authority remain after the successful one-name revoke.

### Preconditions represented by the PoC

* `owner` is the resolver's initial/root EAC administrator and has the admin roles needed to revoke delegate roles;
* `victim.eth` and `other.eth` use that resolver;
* `delegate` has name-scoped `ROLE_SET_ADDR` on both names;
* the resolver overview supplies both assignments, both resource nodes resolve into `resolvedNames`, and `owner` passes `canManageRoles` so editing is exposed;
* `owner` and `delegate` are distinct because the table disables self-editing.

The delegate has zero root roles.

### 1. Prepare the frozen Explorer checkout

Use competition import commit:

```
cda79acaad59711b943fc68207ebb3f1d0ff8596
```

It identifies audited upstream revision `63772fd872af472ced58b009499355f3430c2a86`. The affected Portal files are unchanged at current competition branch head `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

The successful run used Node.js `v26.7.0`, pnpm `10.27.0`, and Vitest `4.1.5`. From the repository root, dependencies were installed with:

```sh
pnpm install --filter 'portal...' --filter '@ens-apps/e2e...' \
  --frozen-lockfile --ignore-scripts --store-dir .pnpm-store \
  --network-concurrency=8
```

Place attachment `E0003-portal-rendered-sidebar.test.tsx` at:

```
apps/portal/src/features/resolver/components/ResolverRolesSidebar.rendered.probe.test.tsx
```

### 2. Run the rendered production-Sidebar PoC

From `apps/portal`:

```sh
./node_modules/.bin/vitest run \
  src/features/resolver/components/ResolverRolesSidebar.rendered.probe.test.tsx \
  --config vitest.config.ts --no-cache --no-file-parallelism \
  --maxWorkers 2 --reporter=verbose
```

Optional focused format/lint check from the repository root:

```sh
./node_modules/.bin/biome check \
  apps/portal/src/features/resolver/components/ResolverRolesSidebar.rendered.probe.test.tsx
```

Observed result:

```
Test Files  1 passed (1)
     Tests  4 passed (4)
  Duration  2.41s
```

The tests exercise the following production behavior:

1. Production `groupRolesByAccount` creates one row for one delegate across two name resources.
2. The real Sidebar renders both names and the relevant role checkbox state.
3. The test clicks the real **Remove user** button.
4. The real Dialog promises removal “from all roles.”
5. The test clicks the real **Remove** confirmation.
6. The Sidebar builds its real pending action and transaction descriptor.
7. The descriptor's unmodified `intent.prepare` callback runs production `prepareResolverRolesIntent`, the production revoke helper, pinned ensjs, and Viem encoding.

The order-independent headline cases grant `ROLE_SET_ADDR` on both names:

* with `victim.eth` first, exactly one `authorizeNameRoles(DNS("victim.eth"), 1, delegate, false)` is prepared;
* with `other.eth` first, exactly one `authorizeNameRoles(DNS("other.eth"), 1, delegate, false)` is prepared.

No call is prepared for the other name in either order. Ordering chooses which single name gets revoked, but another name necessarily retains address authority.

A mixed-role control (`TEXT` on `victim.eth`, `ADDR` on `other.eth`) proves that the Sidebar also unions roles across resources and prepares one `authorizeNameRoles(victim.eth, 17, delegate, false)` call. A root-only control prepares the correct `revokeRootRoles` call.

The only stderr lines are unrelated Radix accessibility warnings about the production Sheet's missing Dialog description.

### 3. Run the exact-contract consequence PoC

Check out `ensdomains/contracts-v2` at:

```
8d1c89350729f87b3968cf41ef3d683951444a44
```

Initialize its submodules; the successful run used ENS contracts submodule `3b1cc225ccdf64581d5fdc81db574f51ba5c8c09`, Forge `1.7.1` commit `4072e48705af9d93e3c0f6e29e93b5e9a40caed8`, and solc `0.8.27`.

Set `CONTRACTS_V2_ROOT` to the absolute path of that checkout and `POC_DIR` to a writable temporary directory. Place attachment `E0004-TwoNameRevocation.t.sol` in `POC_DIR` as `TwoNameRevocationReview.t.sol`. With Forge `1.7.1` available on `PATH`, run:

```sh
CONTRACTS_V2_ROOT=/absolute/path/to/contracts-v2
POC_DIR=/tmp/ens-role-revocation-poc
mkdir -p "$POC_DIR"
cp /absolute/path/to/attachments/E0004-TwoNameRevocation.t.sol \
  "$POC_DIR/TwoNameRevocationReview.t.sol"
env \
  FOUNDRY_TEST="$POC_DIR" \
  FOUNDRY_OUT="$POC_DIR/out" \
  FOUNDRY_CACHE_PATH="$POC_DIR/cache" \
  forge test \
  --root "$CONTRACTS_V2_ROOT/contracts" --offline \
  --match-contract TwoNameRevocationReview --threads 2 -vvv
```

The successful offline run had solc `0.8.27` already installed. If that compiler is not cached locally, omit `--offline` for the first run so Foundry can install the pinned compiler, then repeat offline if desired.

Observed result:

```
Ran 2 tests for TwoNameRevocationReview
[PASS] test_sameRoleOnTwoNamesLeavesOtherAddressAuthority() (gas: 173938)
[PASS] test_twoNameRemoveClearsOnlySelectedName() (gas: 174305)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

The exact resolver assertions establish:

1. both ordinary name resources hold `ROLE_SET_ADDR` for the delegate;
2. the Portal-equivalent first-name revoke returns success;
3. the delegate's role on the selected name is cleared while the other name's role is unchanged;
4. the delegate changes and reads back the other name's Ethereum address;
5. after the owner correctly revokes the surviving name, the identical delegate write reverts with `EACUnauthorizedAccountRoles`;
6. the same result holds for the mixed `TEXT|ADDR` union-bitmap input prepared by the rendered test.

### Expected result

Because the confirmation says the user will be removed from all roles, the prepared action should cover every raw resource represented by the merged row. After the transaction completes, the delegate should have no role on either name and both address writes should revert.

### Actual result

Explorer prepares one revoke for one selected name, and source shows that the production modal submits that prepared action. Separately, the contract harness executes equivalent calldata: the resolver returns success, but the other name's role remains and the delegate can still change that name's address record.

### Composition and limitations

The rendered PoC executes the production Sidebar, Sheet/Dialog/Button/Checkbox primitives, grouping, permission mapping, TanStack mutation state, intent preparation, ensjs builder, and Viem encoder. It supplies synthetic GraphQL rows and mocks Wagmi wallet/public-client hooks, fixed chain context, cosmetics, and the transaction-modal shell. The replacement shell only captures and invokes the unmodified production descriptor's `prepare` callback; it does not call `onStart`, submit to a wallet, or await a receipt.

The Solidity PoC executes the exact resolver implementation from pinned contracts-v2 commit `8d1c893`, including its EAC implementation, name-resource derivation, role storage, role guard, address setter, and readback. It uses synthetic owner/delegate identities and names.

The two parts are complementary local tests bound by the same names, account, role bitmap, and one-name revoke semantics. The rendered production path reaches final calldata preparation, and the exact resolver replay executes matching calldata and verifies the retained authority.

The Portal preparation and exact-contract execution are separate local tests; no full product-path reproduction is claimed.
