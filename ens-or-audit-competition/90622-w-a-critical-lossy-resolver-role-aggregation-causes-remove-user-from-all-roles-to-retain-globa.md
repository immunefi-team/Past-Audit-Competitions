# 90622 w a critical lossy resolver role aggregation causes remove user from all roles to retain global permissions enabling ens record hijacking

**Submitted on Aug 26th 2026 at 02:19:54 UTC by @Subm4rin3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90622
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The ENS Portal incorrectly handles resolver permissions when the same account has roles on multiple resolver resources.

Portal groups all resolver-role assignments belonging to the same account into a single row, but does not preserve the relationship between each role and the resource on which that role was granted.

As a result, when the owner clicks **“Remove user”** and confirms that the user should be removed from **“all roles”**, Portal can generate a revocation transaction scoped to only one ENS name.

Any root/global resolver permission belonging to the supposedly removed account can therefore remain active.

I reproduced the following state using the normal Portal role-management flow:

* Bob has `ROLE_SET_ADDR` at the resolver root, which applies globally to names using that resolver.
* Bob has `ROLE_SET_TEXT` specifically on `scope.eth`.
* Alice uses Portal's **Remove user** action, whose confirmation states that Bob will be removed from **all roles**.
* Portal generates and submits a transaction that revokes the combined role bitmap only from `scope.eth`.
* Bob's root `ROLE_SET_ADDR` remains active.

After the removal transaction succeeds, Bob can still modify the address record of another name using the same resolver, such as `victim.eth`.

I additionally reproduced the downstream impact by changing:

```
victim.eth -> attacker-controlled address
```

using Bob's surviving root permission.

Portal subsequently resolves `victim.eth` to that attacker-controlled address in its normal Add User workflow, and Alice can submit the resulting transaction with the attacker-controlled address as the actual grantee.

The complete reproduced flow is:

```
Portal UI -> wallet signature -> production-generated transaction -> PermissionedResolver -> surviving global permission -> unauthorized setAddr -> poisoned ENS resolution -> Portal transaction targeting the attacker-controlled resolved address
```

No vulnerable calldata is manually constructed or injected into the transaction flow.

## Root Cause

The root cause is lossy aggregation of resolver-role assignments across different authorization resources.

For example, the real authorization state may be:

```
Bob
├── Root:      ROLE_SET_ADDR
└── scope.eth: ROLE_SET_TEXT
```

However, Portal groups these assignments into one account entry.

The relevant aggregation logic is located in:

```
apps/portal/src/lib/roles/resolverRoles.ts
```

The resolver-role sidebar then flattens the roles and resources associated with the account and selects a non-root name as the resource used for the removal action:

```
apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx
```

The production revoke helper interprets any non-empty name as a name-scoped revocation:

```
apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts
```

For the state above, Portal submits the equivalent of:

```solidity
authorizeNameRoles(
    dns("scope.eth"),
    ROLE_SET_ADDR | ROLE_SET_TEXT,
    Bob,
    false
);
```

This is incorrect because the two roles belong to different authorization resources.

The correct behavior requires revoking the root and name-scoped permissions independently:

```solidity
revokeRootRoles(
    ROLE_SET_ADDR,
    Bob
);
```

and:

```solidity
authorizeNameRoles(
    dns("scope.eth"),
    ROLE_SET_TEXT,
    Bob,
    false
);
```

Because root roles and name-specific roles are stored independently, revoking the combined role bitmap from `scope.eth` does not remove Bob's root permission.

## Expected Behavior

When Portal displays an account with both global and name-specific resolver permissions and the owner confirms **“Remove this user from all roles”**, all resolver-role assignments belonging to that account should be revoked from their corresponding resources.

After the transaction succeeds, Bob should have no effective resolver permissions.

## Actual Behavior

The transaction succeeds, but only the role stored on the selected name resource is removed.

The reproduced state transition is:

| State                | Root roles | `scope.eth` roles | Effective address permission on `victim.eth` |
| -------------------- | ---------: | ----------------: | -------------------------------------------: |
| Before removal       |     `0x01` |            `0x10` |                                       `0x01` |
| After Portal removal |     `0x01` |            `0x00` |                                       `0x01` |
| Expected             |     `0x00` |            `0x00` |                                       `0x00` |

Alice receives a successful transaction result and has explicitly confirmed removal from all roles, but Bob still retains global `ROLE_SET_ADDR`.

## Security Impact

The surviving global permission remains fully usable.

After Alice successfully completes Portal's all-role removal flow, Bob can still call the real `PermissionedResolver` and change the address record of `victim.eth`.

In the PoC, Bob changes the record to:

```
0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
```

The `UniversalResolver` subsequently resolves:

```
victim.eth
→ 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
```

The impact is therefore not limited to stale UI state or an incorrectly displayed permission.

The supposedly revoked account retains effective authorization and successfully modifies ENS resolution.

The PoC additionally demonstrates a security-sensitive downstream Portal consumer.

After Bob poisons the address record, Alice enters `victim.eth` in Portal's normal Add User workflow.

Portal resolves `victim.eth` to the attacker-controlled address and submits a role-grant transaction whose actual grantee is that address:

```solidity
grantRootRoles(
    ROLE_SET_TEXT,
    0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
);
```

Therefore, an account that the owner explicitly removed from all resolver roles can retain global resolver authority, redirect an ENS name to an attacker-controlled address, and cause a later Portal transaction to target that attacker-controlled resolved address.

## Attack Preconditions

The attacker must previously have been legitimately delegated resolver permissions by the victim.

In the demonstrated case:

```
Root:
Bob -> ROLE_SET_ADDR

scope.eth:
Bob -> ROLE_SET_TEXT
```

The vulnerability is triggered when the victim later uses the normal Portal **Remove user** workflow.

No compromised private key, malicious RPC, stale indexer state, migration, name transfer, browser-storage modification, or privileged protocol role is required.

## Known-Issue Independence

This issue does not depend on the known transfer-related role failures.

No name transfer occurs in the attack.

The independent root cause is:

```
lossy cross-resource resolver-role aggregation
+
single-resource revocation
```

The security consequence is also distinct from a functional error: the supposedly revoked account retains real authorization, successfully modifies an ENS address record, and causes a later Portal transaction to use an attacker-controlled resolved address.

The issue does not depend on V1-to-V2 migration, subregistry replacement, subgraph/indexer delay, HCA/session behavior, or keep-resolver transfer behavior.

## Link to Proof of Concept

<https://gist.github.com/5ubm4rin3/9a2cf853749e711e91f1e9fdf6bbc79f>

## Proof of Concept

A complete integrated Portal → wallet → local Anvil → real PermissionedResolver PoC is provided in the accompanying secret Gist.

### PoC files

* `run-integrated-e2e.sh`
* `h8-resolver-scope.e2e.spec.ts`
* `evidence.json`

### Run

From the competition repository root:

```bash
security-pocs/H8/run-integrated-e2e.sh
```

The runner:

{% stepper %}
{% step %}

### Start a local Anvil fork pinned to Sepolia block `11558366`

{% endstep %}

{% step %}

### Start the unmodified Portal locally

{% endstep %}

{% step %}

### Connect deterministic local Alice, Bob, and attacker wallets

{% endstep %}

{% step %}

### Run the Playwright Portal E2E

{% endstep %}

{% step %}

### Mine the Portal-generated resolver transactions on local Anvil

{% endstep %}

{% step %}

### Record the final transaction hashes and before/after authorization state

{% endstep %}
{% endstepper %}

Expected final output:

```
H8_PORTAL_REMOVAL_TX=0x8376aa8111e2157b81e6e164fe62d5ceb29eda20e28dc71ae93a8c19a44c724c
H8_ATTACK_TX=0xa5bb7f57394b3de49a98edc19c0f94f754e1562875afdfd823ca7cb349886965
H8_PORTAL_SINK_TX=0xc9b0a4aca9ecd60f756e45c96f0d72cd4ce0b5777a78a48caf918a06a79b051d
H8_FINAL_RESOLUTION=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
1 passed
```

## Step-by-Step Reproduction

{% stepper %}
{% step %}

## Create the precondition

Alice controls `victim.eth` and `scope.eth`, and both names use the same `PermissionedResolver`.

The PoC uses fork fixture setup only to establish this pre-existing ownership/resolver state.

It does **not** manually write resolver-role storage.

All security-relevant role grants, the vulnerable removal, and the final poisoned-name grant are produced and submitted through Portal's normal UI and transaction transport.

Alice then grants Bob through the normal Portal **Add user** workflow:

```
Root (all names):
ROLE_SET_ADDR = 0x01

scope.eth:
ROLE_SET_TEXT = 0x10
```

The effective state is:

```
Bob
├── Root:      ROLE_SET_ADDR
└── scope.eth: ROLE_SET_TEXT
```

{% endstep %}

{% step %}

## Observe the lossy role aggregation

Portal displays one Bob row containing both resources and both permissions:

```
(root)
scope.eth
Set Address
Set Text
```

Opening Bob's sidebar displays:

```
Global roles (all names)
Remove user
```

The resolver-overview data used by the local E2E is derived from live reads against the locally deployed resolver. It does not fabricate Bob's role state.
{% endstep %}

{% step %}

## Remove Bob through the production Portal UI

Alice clicks **Remove user**.

Portal displays the confirmation:

```
Are you sure you want to remove this user from all roles?
This action cannot be undone.
```

Alice confirms using the connected local wallet.

The production Portal transaction is submitted directly to local Anvil.

No vulnerable calldata is manually constructed or copied into another harness.
{% endstep %}

{% step %}

## Decode the mined removal transaction

The mined transaction is:

```
0x8376aa8111e2157b81e6e164fe62d5ceb29eda20e28dc71ae93a8c19a44c724c
```

Sender:

```
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```

Target PermissionedResolver:

```
0xBcd596BbF97C844A8AdF621798B80D89DA1069cC
```

Only after the transaction has mined, the PoC decodes it as:

```solidity
authorizeNameRoles(
    0x0573636f70650365746800, // dns("scope.eth")
    0x11,                   // ROLE_SET_ADDR | ROLE_SET_TEXT
    0x70997970C51812dc3A010C7d01b50e0d17dc79C8,
    false
);
```

This proves that the vulnerable scope is generated by Portal itself.
{% endstep %}

{% step %}

## Verify the incomplete revocation

Before Alice removes Bob:

```
Root roles:           0x01
scope.eth roles:      0x10
Effective victim.eth: 0x01
```

After the successful Portal removal:

```
Root roles:           0x01
scope.eth roles:      0x00
Effective victim.eth: 0x01
```

Expected secure state:

```
Root roles:           0x00
scope.eth roles:      0x00
Effective victim.eth: 0x00
```

Bob's name-specific role is removed, but the global `ROLE_SET_ADDR` survives.
{% endstep %}

{% step %}

## Exercise Bob's supposedly revoked authority

Bob then uses his own wallet to call the real resolver and changes the ETH address record for `victim.eth` to:

```
0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
```

Attack transaction:

```
0xa5bb7f57394b3de49a98edc19c0f94f754e1562875afdfd823ca7cb349886965
```

The transaction succeeds.
{% endstep %}

{% step %}

## Resolve the poisoned ENS name

The PoC queries the actual `UniversalResolver`.

Result:

```
victim.eth
→ 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
```

The selected resolver is the same real `PermissionedResolver`.
{% endstep %}

{% step %}

## Demonstrate a production Portal transaction sink

Alice opens Portal's normal **Add user** flow and enters:

```
victim.eth
```

Portal displays the resolved attacker-controlled address:

```
0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
```

Alice selects `ROLE_SET_TEXT` and confirms the normal Portal transaction.

The mined transaction is:

```
0xc9b0a4aca9ecd60f756e45c96f0d72cd4ce0b5777a78a48caf918a06a79b051d
```

Decoded after mining:

```solidity
grantRootRoles(
    0x10,
    0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
);
```

The attacker's final root-role bitmap is:

```
0x10
```

{% endstep %}
{% endstepper %}

## Result

The full demonstrated chain is:

```
Alice selects "Remove user from all roles"
→ Portal revokes only scope.eth
→ Bob's global ROLE_SET_ADDR survives
→ Bob changes victim.eth to attackerAddress
→ UniversalResolver returns attackerAddress
→ Portal resolves victim.eth to attackerAddress
→ Alice submits the production-generated transaction
→ attacker-controlled address receives the role
```

## Fixture / Harness Transparency

The E2E intentionally distinguishes test-environment setup from the vulnerable action.

The test uses direct fork storage setup only to establish the pre-existing registry ownership/resolver fixture for `victim.eth` and `scope.eth`, because a full local indexer/registration history is not available.

This setup does **not** create or modify the vulnerable resolver-role state.

The following security-relevant operations are all real transactions:

1. Bob's root `ROLE_SET_ADDR` grant through Portal.
2. Bob's `scope.eth` `ROLE_SET_TEXT` grant through Portal.
3. Alice's vulnerable **Remove user from all roles** transaction through Portal.
4. Bob's unauthorized post-removal `setAddr`.
5. Alice's poisoned-name role grant through Portal.

The locally intercepted resolver-overview GraphQL response is constructed from live reads against the actual resolver's current on-chain role state. It is presentation/indexer plumbing only and does not inject authorization.

The test also isolates two unrelated optional header RPC reads during initial page authorization on the cold fork. That interception is removed before any role transaction is opened, so transaction estimation and submission use Portal's normal browser RPC transport.
