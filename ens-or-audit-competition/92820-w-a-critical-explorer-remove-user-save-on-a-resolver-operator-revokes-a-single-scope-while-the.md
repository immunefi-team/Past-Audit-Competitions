# 92820 w a critical explorer remove user save on a resolver operator revokes a single scope while the row aggregates root and name scopes so root level roles survive a remove this user fr&#x20;

**Submitted on Sep 12th 2026 at 15:27:18 UTC by @luT for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92820
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

Impact category chosen as the closest match: the Explorer builds a transaction whose arguments do not match the action it presents ("Modifying transaction arguments or parameters"). Severity is claimed as Medium, not Critical: no attacker controls the construction, the precondition is an operator the owner had previously trusted, and loss requires that operator to act after the failed revocation.

On the Explorer's Resolver page, the roles table groups every role an account holds on a `PermissionedResolver` into one row, regardless of the resource the role was granted on (root resource = all names, or one specific name). When the resolver owner edits that row with **Save** or **Remove user**, the sidebar submits the merged role set under a **single** scope: the first non-root name in the row, or root only if there is no name at all.

For an operator that holds a role at the root (granted with the sheet's default "Root (all nodes)" scope) and another role on a name, the sidebar shows "Global roles (all names)", the dialog says "remove this user from all roles", the wallet signs one transaction, the transaction succeeds, and the root-level role is still there. The "removed" operator keeps write access to the address and text records of **every** name served by that resolver.

## Vulnerability Details

### Affected code

* `apps/portal/src/lib/roles/resolverRoles.ts:179-221` — `groupRolesByAccount` merges `decodedRoles` across all resources per account and keeps only a `Set` of resolved names; the resource each role was granted on is discarded.
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:166-169` — a single scope for the whole edit: `roleName = resolvedNames.find((n) => n !== '(root)') ?? ''`.
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:299-323` — `handleSaveChanges` / `handleRemoveUser` submit the merged diff / the full `decodedRoles` under that one `roleName`.
* `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61` — `name === ''` maps to ensjs scope `root` (`revokeRootRoles(bitmap, account)`), anything else to scope `name` (`authorizeNameRoles(dnsEncode(name), bitmap, account, false)`).
* UI wording that contradicts the encoded call: `ResolverRolesSidebar.tsx:393-395` ("Global roles (all names)") and `:446-447` ("remove this user from all roles").
* The mixed state is produced by the app itself: `ResolverAddUserSheet.tsx` offers "Root (all nodes)" as the default placeholder scope and any node of the resolver as an alternative (`grantResolverRoles.ts:45-61`: root → `grantRootRoles`, node → `authorizeNameRoles(..., true)`).

On-chain (Sepolia `ensPermissionedResolverImpl` `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`, Sourcify exact match), the two scopes are distinct storage resources: `authorizeNameRoles(name, bitmap, account, false)` calls `_revokeRoles(PermissionedResolverLib.resource(node, 0), ...)` (`PermissionedResolver.sol:273-293`), while root grants live under `ROOT_RESOURCE = 0` and apply to every node as a fallback (`EnhancedAccessControl.sol:19, 53-54`, `hasRoles` via `_effectiveRoles`). A name-scoped revoke therefore cannot touch a root-scoped grant.

### Encoded vs. shown

Operator `X` holds `ROLE_SET_ADDR` at root and `ROLE_SET_TEXT` on `alice.eth`.

| What the owner sees                                                        | What the Explorer sends                                                                               |
| -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Row for `X` labelled "Global roles (all names)", roles: Set Addr, Set Text | `authorizeNameRoles(dnsEncode("alice.eth"), ROLE_SET_ADDR \| ROLE_SET_TEXT, X, false)`                |
| Dialog: "remove this user from all roles"                                  | only `resource(namehash("alice.eth"), 0)` is cleared; `ROLE_SET_ADDR` at `ROOT_RESOURCE` is untouched |

Same defect for **Save** (unticking a role that is held at root). Two further variants follow from the same code path (derived from the code, not executed here): roles on two names → only the first name's resource is revoked; a role on a name that is not in the resolver's node list → `resolvedNames` is empty, `roleName = ''`, and a root-scoped revoke is sent for roles that live on the name (no-op).

### Preconditions

* The operator was granted at least one role at the root scope and at least one role on a name of the same resolver (both through the Explorer's own "Add user" sheet; root is the sheet's default scope).
* The owner uses "Remove user" or "Save" on that operator's row.
* The operator acts after the "removal". No other privilege or user interaction is required.

### Known issues check

No listed known issue covers this. QA-01 explicitly states that "a demonstrated security consequence (e.g., a role combination that grants unintended authority) would be a new finding". WEB-716, WEB-1249 and WEB-1305 concern the registry page, subname creation and V1 transfers, not resolver role revocation.

## Impact Details

An owner-initiated revocation of a resolver operator is confirmed by the UI and by a successful transaction, but the operator's global write access remains. The ex-operator can call `setAddr` / `setText` / `setContenthash` for any name resolved by that resolver, including redirecting the ETH address of names to an address of their choosing. Funds sent to those names then go to the attacker. No on-chain check is bypassed; the Explorer encodes a narrower call than the action it presents, and the owner has no signal that the revocation was partial.

## Recommendation

Keep the per-resource role sets in `AccountRoleGroup` (or key the sidebar rows by `(account, resource)`), and emit one call per resource on Save / Remove: `revokeRootRoles(bitmap, account)` for the root-held bits and `authorizeNameRoles(name, bitmap, account, false)` for each name. Until then, refuse Save / Remove when a group spans more than one resource and tell the owner which grants remain.

## References

* Repository: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready>, commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`
* Deployed `PermissionedResolver` implementation on Sepolia: `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e` (Sourcify exact match) — `authorizeNameRoles` at `PermissionedResolver.sol:273-293`, `ROOT_RESOURCE` fallback at `EnhancedAccessControl.sol:19, 53-54`
* Attached archive `F2_poc.zip`: `run.sh`, `calldata.json`, `run.log`, `pocCalldata.poc.test.ts`, `ResolverRolesSidebar.scope.test.tsx`

## Link to Proof of Concept

<https://gist.github.com/louisvrd/b9c86a7614881178ab0d3c58d285267c>

## Proof of Concept

### A. Executed: on-chain replay of the Explorer's own calldata (Sepolia fork)

The three Explorer transactions (two grants, then "Remove user") are encoded by the Explorer's own builders (`prepareGrantResolverRolesTransaction`, `prepareRevokeResolverRolesTransaction`) with the exact parameters the sidebar passes for the aggregated row (captured by the regression test in part C). They are then sent to a real `PermissionedResolver` deployed through the Sepolia `VerifiableFactory` on an Anvil fork of Sepolia.

Files: `poc/run.sh`, `poc/calldata.json`, `poc/pocCalldata.poc.test.ts`.

```
# terminal 1
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --port 8545
# terminal 2, from the repo root
cd apps/portal && pnpm vitest run src/features/resolver/helpers/pocCalldata.poc.test.ts   # writes poc/calldata.json
cd ../../poc && ./run.sh
```

Output:

```
== 1. Deploy a PermissionedResolver owned by OWNER (same deployProxy/initialize the Explorer uses)
   tx 0x80d7a4eccbdadc660495a51713cbb3748a46b706d93e3c51270867750056e840 status=0x1
   resolver proxy: 0x1ca65Dec7d994AE0d42602328b2821322e46BDF8
   verifyContract(proxy) -> 0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
== 2. Explorer 'Add user' -> Root (all nodes), ROLE_SET_ADDR for OPERATOR
   calldata: 0x072d5d77000000000000000000000000000000000000000000000000000000000000000100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8
   tx 0x07937e1029a07d0049a84db48877bc92da5ac5f05406d8494acfaa3f29e7ab5c status=0x1
== 3. Explorer 'Add user' -> node alice.eth, ROLE_SET_TEXT for OPERATOR
   calldata: 0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000001000000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c80000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000b05616c6963650365746800000000000000000000000000000000000000000000
   tx 0xe89cfebd7241dff1d1d15b8b7e8e638c0f5cec2fcd3b9e406b6cdb87b271818a status=0x1
== 4. State before removal
   hasRootRoles(ROLE_SET_ADDR, OPERATOR)          = true
   hasRoles(alice.eth, ROLE_SET_TEXT, OPERATOR)   = true
== 5. Explorer 'Remove user' -> confirm 'remove this user from all roles' (single tx the sidebar builds)
   calldata: 0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000001100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c80000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000b05616c6963650365746800000000000000000000000000000000000000000000
   decoded : 0x05616c6963650365746800 17 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 false 
   tx 0x55c1156c8758a498ec291cf8f815b70bb673a669bbcffaeb664b3f88bce9a9fb status=0x1
== 6. State after removal
   hasRoles(alice.eth, ROLE_SET_TEXT, OPERATOR)   = false   (expected false)
   hasRootRoles(ROLE_SET_ADDR, OPERATOR)          = true   (expected false, actual: still granted)
== 7. Removed OPERATOR still rewrites the ETH address of ANY name on this resolver
   addr(bob.eth, 60) before = 0x
   tx 0x8fac2a0e62abd007cb3751d0a97818b0f4dece42f42473cb6ab049e9e0b4fbb9 status=0x1
   addr(bob.eth, 60) after  = 0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc   (ATTACKER = 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC)
== 8. What the app should have sent: revokeRootRoles for the root-held bits
   tx 0xb2e4401903d69f1d887a25405a3eaf020efab4ca861b5cc78c4735b9317bca8d status=0x1
   hasRootRoles(ROLE_SET_ADDR, OPERATOR)          = false
   OPERATOR setAddr(bob.eth) now reverts with:
   execution reverted: custom error 0x4b27a133: e3f4672e728b79206ba500b850eb553d617e3156863fda5c15fb6e97ff4156e9000000000000000000000000000000000000000000000000000000000000000100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8
```

Step 5 is the single transaction the sidebar builds; step 6 shows the root grant surviving it; step 7 shows the "removed" operator redirecting `bob.eth`'s ETH address to the attacker; step 8 shows the call the Explorer should have sent, after which the same `setAddr` reverts with `EACUnauthorizedAccountRoles`.

### B. Reproduction through the Explorer UI (Sepolia)

1. As the resolver owner, open `/resolver/<your PermissionedResolver>/roles` and click **Add user**. Enter address `X`, leave the scope on **Root (all nodes)**, tick **Set Addr**, sign.
2. Click **Add user** again. Enter `X`, pick a node of the resolver (e.g. `alice.eth`), tick **Set Text**, sign.
3. Open the row for `X`. The sidebar shows "Global roles (all names)" with Set Addr and Set Text.
4. Click **Remove user**, confirm "remove this user from all roles", sign the single transaction (decode it: it is `authorizeNameRoles("alice.eth", 0x11, X, false)`).
5. Read `hasRootRoles(1, X)` on the resolver: expected `false`, actual `true`.
6. From `X`, send `setAddr(namehash("<any name on this resolver>"), 60, <attacker>)`: succeeds.

### C. Regression test (fails on the current code)

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.scope.test.tsx` renders the sidebar with a root + name role group, clicks Remove user → Remove → start, and asserts that a root-scoped revoke (`name === ''`) containing `ROLE_SET_ADDR` is issued.

```
cd apps/portal && pnpm vitest run src/features/resolver/components/ResolverRolesSidebar.scope.test.tsx
AssertionError: revoke calls were: [{"resolverAddress":"0x1234...","name":"alice.eth","account":"0x7099...","roles":["ROLE_SET_ADDR","ROLE_SET_TEXT"], ...}]: expected undefined to be defined
 Tests  1 failed (1)
```
