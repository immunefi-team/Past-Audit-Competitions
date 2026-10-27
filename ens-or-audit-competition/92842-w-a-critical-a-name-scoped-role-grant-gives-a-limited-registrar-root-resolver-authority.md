# 92842 w a critical a name scoped role grant gives a limited registrar root resolver authority

**Submitted on Sep 12th 2026 at 18:57:01 UTC by @PlasticGinnie98802 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92842
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Summary

Explorer accepts bracketed 64-character hashes as the display form for ENS labels whose plaintext is unknown. The same bracketed text can also exist on-chain as a literal label. Those two cases look identical as strings, but they identify different resources.

The roles flow loses that distinction. When a user opens the literal name below and grants an account the name-level **Set Resolver / Manager** role, Explorer passes the displayed label through `labelToCanonicalId()`:

```
[0000000000000000000000000000000000000000000000000000000000000000].parent.eth
```

Viem treats the brackets as encoded-label notation and returns labelhash zero. ENSjs treats zero as the registry-root sentinel. Explorer therefore prepares `grantRootRoles`, even though the user opened a child name and selected a name-scoped role.

I reproduced the full consequence locally. An attacker began with only `ROLE_REGISTRAR`, which allowed new labels to be created but did not allow the attacker to change the resolver of an existing sibling. After the victim approved Explorer's ordinary Add-user transaction for the literal bracket child, the attacker received root `ROLE_SET_RESOLVER` and replaced the sibling's resolver. A correctly scoped grant for the child could not do that.

I selected Critical because Explorer changes the scope of a transaction prepared for an already-connected wallet: a child-name role grant becomes a registry-root role grant. That is the program's listed transaction-parameter impact. The attacker's existing registrar role and the victim's single approval are disclosed below; if those feasibility limits are weighted more heavily, High is the conservative fallback.

## Affected revision

* Asset: Explorer app Files (`apps/portal`)
* Competition import: `cda79acaad59711b943fc68207ebb3f1d0ff8596`
* Competition branch: `audit-comp-ready@1c9b47f18fcddd2e864dfe385c4171061c9811ae`
* Official upstream: `63772fd872af472ced58b009499355f3430c2a86`
* Imported tree: `8e0d79dac1ab4b4fdb80d6afed810087ae9f00ba`

Affected files:

* `apps/portal/src/features/roles/helpers/grantRoles.ts:58-94`
* `apps/portal/src/features/roles/components/RolesAddUserSheet.tsx:41-118,128-264`
* `apps/portal/src/features/roles/components/NameRolesOverviewTable.tsx:84-130`
* `apps/portal/src/utils/token/isNormalized.ts:3-27,39-56`
* `apps/portal/node_modules/@ensdomains/ensjs/src/utils/v2/registry/labelToCanonicalId.ts:1-7`
* `apps/portal/node_modules/@ensdomains/ensjs/src/actions/wallet/v2/grantRoles.ts:26-66`

The ENS V2 registry contracts behave as designed. They are used below only to show the consequence of the transaction Explorer prepares. The affected target is the Explorer application.

## Vulnerability details

### 1. Explorer accepts encoded-label notation as a valid route name

`isNormalized()` accepts either a normal ENS label or the bracketed encoded-label form:

```ts
const ENCODED_LABELHASH_RE = /^\[[0-9a-f]{64}\]$/

export const isNormalized = (name: string) =>
  name
    .split('.')
    .every((label) => isEncodedLabelhash(label) || isNormalizedLabel(label))
```

This is needed when only a labelhash is known. It also means the direct roles route accepts the exact same character sequence when those brackets and digits are the real DNS label bytes.

The production owner-discovery path resolves the literal bracket child correctly. It identifies the victim as owner, reports ENSv2, and returns the containing `parent` registry. The child is a real non-root resource.

### 2. The write helper recomputes authority from the display string

The roles page already has the resolved name and containing registry. The grant helper does not preserve the literal resource identity. It extracts the first displayed label and derives a new resource value from that string:

```ts
const { label } = makeLabelNodeAndParent(name)
const resource = labelToCanonicalId(label)

const writeParams = grantRolesWriteParameters(walletClient, {
  registryAddress,
  account,
  resource,
  roles: [...roles],
})
```

For the literal label used in this report, the real on-chain label ID is:

```
0xc96c72c22703f1c7d0c4a9196336f77e1d824f1a6ff133c2b782a0c51a2b9ff8
```

`labelToCanonicalId()` instead sends the bracketed string to viem's `labelhash()`. Viem interprets it as pre-encoded notation and returns:

```
0x0000000000000000000000000000000000000000000000000000000000000000
```

The two values refer to different scopes. Explorer does not compare them before preparing the transaction.

### 3. Zero selects the root-only transaction builder

ENSjs uses resource zero as a control value:

```ts
const isRootResource = resource === 0n

if (isRootResource) {
  return {
    functionName: 'grantRootRoles',
    args: [roleBitmap, account],
  }
}
```

The user selected a role for one child name. The wallet request instead calls `grantRootRoles` for the containing registry.

The transaction modal does not expose the change in scope. In the reproduced flow, it showed `Grant roles` and `Grant roles for 0x…`. It did not mention root, global, the registry, or all resources.

### 4. The resulting role applies to sibling names

The V2 registry combines root roles with resource-scoped roles when checking authorization. Root `ROLE_SET_RESOLVER` is therefore effective for every resource in that registry.

Before the faulty grant, the attacker had only `ROLE_REGISTRAR` and a resolver change against the existing `valuable` sibling reverted. After the victim executed Explorer's prepared calldata, the same attacker successfully replaced that sibling's resolver. The victim had only intended to delegate resolver management for the displayed bracket child.

## Preconditions

1. The victim administers an ENS V2 child registry and can grant resolver roles.
2. The attacker already has the limited `ROLE_REGISTRAR` role in that registry. This permits new labels but not resolver changes for existing siblings.
3. The attacker creates the literal bracket label and assigns that child to the victim. Explorer's own create-subname input rejects this spelling, so the attacker uses the permissioned registry contract or another compatible registrar client.
4. The victim opens the literal child's direct Explorer roles route, enters the attacker as a delegate, selects **Set Resolver / Manager**, and approves the one transaction presented by Explorer.

The attacker does not need the victim's key, wallet access, a malicious RPC, a browser defect, or control of an existing sibling. The victim chooses a child-scoped resolver grant; the undisclosed root scope is introduced by Explorer.

## Impact

The attacker expands a limited label-creation role into registry-wide resolver authority. The reproduced transaction grants root `ROLE_SET_RESOLVER`, after which the attacker can replace the resolver of existing siblings that the original role could not modify.

The reproduced consequence is replacement of an unrelated existing sibling's resolver. That new authority can redirect records served for names in the affected registry. The PoC does not claim NFT theft, wallet compromise, or a completed downstream transfer of funds.

## Why the affected Explorer code is necessary

The contracts do not confuse the resource. They execute the root-role call exactly as encoded. The attacker's starting `ROLE_REGISTRAR` also does not permit the sibling resolver change.

Explorer creates the new authority. It resolves and displays one real non-root child, then recomputes the write resource from the display label and encodes the root-role entrypoint. A correctly scoped transaction does not produce the sibling effect.

## Recommended fix

Do not derive a write-authoritative resource ID from a display-form label.

The roles route already discovers the containing registry on-chain. Extend that typed result so the actual resource ID associated with the resolved child is preserved through the form, modal, and transaction builder. The code should also remember whether a bracketed label came from unknown-label notation or from literal DNS bytes; a plain string cannot represent that distinction safely.

As a separate guard, refuse to produce resource zero from a non-root route. `grantRootRoles` should be available only from an explicit registry-root workflow. The confirmation should identify the registry and whether the role applies to one name or every resource.

Regression tests should cover the literal zero bracket label, nonzero encoded-label notation, ordinary labels, and the explicit root workflow. A name-scoped page must never emit `grantRootRoles`.

## Duplicate assessment

This is not submission `#92416`, the raw/canonical transfer report. That issue case-folds a route name, selects another ERC-1155 token ID, and calls `safeTransferFrom`. This report preserves the selected name but converts its literal resource into the root sentinel and calls `grantRootRoles`.

It does not share the stale-indexer migration root in submission `#92509`. That report restores a former manager during migration; this one does not use migration or indexer state.

It is also different from rejected submission `#92545` and the earlier matched report `#89387`. That resolver-user removal flow failed to revoke a root role the account already held. Here Explorer creates new root authority for an account that did not have `ROLE_SET_RESOLVER` before the victim's transaction.

`EXP-INPUT-005` covers deceptive-character display behavior. This report demonstrates a state-changing resource-selection error and a cross-sibling authorization expansion. The producer conversion, transaction function, capability gained, and fix are different.

The first security-relevant divergence here is the conversion of a real non-root literal label into resource zero while Explorer builds a new grant transaction.

## Verification

The natural entry, rendered UI, exact calldata, registry execution, and cross-sibling consequence were reproduced in separate TEST and independent REPLAY runs. The correctly scoped negative control could not modify the sibling resolver.

## Proof of Concept

## Proof of Concept

This is a step-by-step explanatory PoC for the audit-competition form. It uses no public RPC, production account, production transaction, or third-party service.

I used the competition branch at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, Node.js 24.2.0, pnpm 10.27.0, a fresh local Anvil chain, and the ENS V2 contracts from the competition's PR #409 test material.

The proof has three boundaries:

* the natural-entry check calls the production `resolveEnsOwner()` path over local JSON-RPC;
* the UI check renders the production `NameRolesOverviewTable`, `RolesAddUserSheet`, and `TransactionModal`, then captures the calldata produced by the real gas-intent path;
* the impact check executes that calldata against the local V2 registry and compares it with a correctly scoped grant.

Wallet connection, address lookup, query results, transaction-store state, and gas estimation are controlled test fixtures. Resource resolution, the rendered role flow, calldata construction, registry authorization, and the sibling resolver change use the audited application or exact contract implementation.

### Step 1: Create the two different resource identities

1. Deploy a local `root → eth → parent` V2 registry hierarchy.
2. Give the attacker root `ROLE_REGISTRAR` only.
3. Create an existing sibling called `valuable.parent.eth` for the victim.
4. Have the attacker register this literal child to the victim:

   ```
   [0000000000000000000000000000000000000000000000000000000000000000].parent.eth
   ```
5. Confirm that the DNS packet contains the literal bracket characters. Its real label ID is nonzero:

   ```
   0xc96c72c22703f1c7d0c4a9196336f77e1d824f1a6ff133c2b782a0c51a2b9ff8
   ```
6. Pass the same displayed label to viem's `labelhash()`. The encoded-label interpretation is zero.

The production owner-discovery result was:

```json
{
  "attackerIsLimitedRegistrar": true,
  "protocolVersion": "ENSv2",
  "ownerMatches": true,
  "registryMatches": true,
  "fullHierarchy": true,
  "literalId": "0xc96c72c22703f1c7d0c4a9196336f77e1d824f1a6ff133c2b782a0c51a2b9ff8",
  "interpretedLabelHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "literalResourceIsNonRoot": true,
  "portalInterpretationIsRoot": true
}
```

### Step 2: Follow Explorer's normal Add-user flow

1. Connect the victim wallet.
2. Open the direct roles page for the literal bracket child.
3. Enter the attacker account in **Add user**.
4. Select **Set Resolver / Manager**. The Admin option remains disabled, which makes this look like a limited name-level grant.
5. Click **Save**, then continue to the real transaction modal.
6. Confirm that the overview says `Grant roles` and identifies the grantee, but does not disclose root or all-resource scope.
7. Click **Start** and capture the wallet request before signing it.

Explorer produced this calldata:

```
0x072d5d770000000000000000000000000000000000000000000000000000000001000000000000000000000000000000c5e8d55b2c2656689ea7c52b73191491cb5e5b8e
```

Decoding it gives `grantRootRoles`, not a grant for the displayed child:

```json
{
  "managerCheckboxEnabled": true,
  "adminCheckboxDisabled": true,
  "pageContainsParentRegistryRoles": true,
  "decodedFunction": "grantRootRoles",
  "rootScopeDisclosureAbsent": true
}
```

The selector is `0x072d5d77`, which decodes to `grantRootRoles(uint256,address)`. The account argument is `0xC5e8d55b2c2656689ea7C52B73191491Cb5e5b8e`.

### Step 3: Execute the prepared transaction and test the consequence

1. Before the victim grant, call `setResolver()` for `valuable.parent.eth` from the attacker. It reverts because `ROLE_REGISTRAR` does not include resolver authority.
2. Execute the exact `grantRootRoles` calldata from the victim account.
3. Check the attacker's roles. Root `ROLE_SET_RESOLVER` is now present.
4. Call `setResolver()` for the existing `valuable` sibling from the attacker. It succeeds and stores the attacker-selected resolver.
5. Reset to the same initial state for the negative control.
6. Call `grantRoles(literalChildResource, ROLE_SET_RESOLVER, attacker)` instead of the root grant.
7. Repeat the sibling resolver change. It reverts.

The impact observation was:

```json
{
  "attackerStartingAuthority": "ROLE_REGISTRAR",
  "attackerCanModifySiblingBefore": false,
  "attackerRootSetResolverBefore": false,
  "portalFunction": "grantRootRoles",
  "portalGrantSuccess": true,
  "attackerRootSetResolverAfter": true,
  "rootRolesApplyAcrossResources": true,
  "siblingResolverReplaced": true,
  "scopedGrantSiblingReverted": true,
  "victimGrantTransactions": 1
}
```

I ran each of the three boundaries as a TEST and an independent REPLAY. The decisive observations matched in both runs.
