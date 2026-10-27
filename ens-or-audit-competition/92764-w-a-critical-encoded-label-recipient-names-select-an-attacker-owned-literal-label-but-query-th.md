# 92764 w a critical encoded label recipient names select an attacker owned literal label but query the honest hash only node redirecting portal name transfers

**Submitted on Sep 12th 2026 at 04:15:30 UTC by @Ailenia for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92764
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

I found that Portal accepts `[H]` as a genuine hash-only ENS label in action-bearing inputs, but does not bind resolver discovery to the node queried from that resolver. In a compatible public hybrid subnamespace, an attacker can register the distinct literal `[H]` label and make Portal resolve the honest V1 representation to the attacker's address. A victim who approves the displayed transfer then sends their ENSv2 name NFT to the attacker.

## Vulnerability Details

The affected revision is `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

Affected asset: `../ens/repos/audit-comp-ens/apps/portal`.

Portal accepts `[64 lowercase hexadecimal characters]` as the indexer representation of a label known only by hash (`../ens/repos/audit-comp-ens/apps/portal/src/utils/token/isNormalized.ts:3,39-57`). `../ens/repos/audit-comp-ens/apps/portal/src/utils/token/isNameOrAddress.ts:23` consequently accepts a multi-label encoded name in account-bearing inputs.

The same file recognizes that a literal bracket label is indistinguishable from an encoded labelhash, but protects only registration inputs (`../ens/repos/audit-comp-ens/apps/portal/src/utils/token/isNormalized.ts:59`). A public registrar can accept the distinct 66-byte literal label and recipient resolution does not reject it.

Portal's dashboard obtains V1 names (`../ens/repos/audit-comp-ens/apps/portal/src/features/dashboard/hooks/useV1NamesForAddress.ts:18`) and preserves the V1 `name` while merging (`../ens/repos/audit-comp-ens/apps/portal/src/utils/names/mergeNamesData.ts:81`). It renders or copies that value with `EntityBadge` (`../ens/repos/audit-comp-ens/apps/portal/src/features/names/components/NamesTable/columns.tsx:22`; `../ens/repos/audit-comp-ens/apps/portal/src/components/EntityBadge.tsx:192,502`).

Without a V1 label preimage, the displayed and copied value is `[H].zone.eth`, identifying the honest hash-only resource. My PoC passes that exact production copy value to `SendNameForm`.

The vulnerable resolution flow is:

1. `../ens/repos/audit-comp-ens/apps/portal/src/features/address/hooks/useAddressResolution.ts:43` documents that the shared hook is used by transfer-recipient and other name-or-address fields. It validates the encoded input and calls `getResolvedAddress`.
2. `../ens/repos/audit-comp-ens/apps/portal/src/features/address/queries/getResolvedAddress.ts:26` passes the string unchanged to `resolveAddressOrName`.
3. `../ens/repos/audit-comp-ens/apps/portal/src/features/roles/helpers/addUser.handlers.ts:20` calls Viem's `getEnsAddress` with that string and the Universal Resolver, without checking that resolver discovery and resolver calldata refer to the same node.
4. The DNS packet used for resolver discovery contains the literal 66-byte `[H]` label, so target-era `UniversalResolverV2` traversal selects the resolver of `keccak256(parent, keccak256(bytes("[H]")))`. The record calldata uses `namehash("[H].zone.eth")`; Viem recognizes the encoded notation and substitutes `H`, producing `keccak256(parent, H)`. These nodes are distinct.
5. The owner of the literal `[H]` registration can select its resolver while that resolver is asked for the honest hash-only node. The attacker configures its resolver to return the attacker address for the decoded node.
6. `../ens/repos/audit-comp-ens/apps/portal/src/features/transfer/components/SendNameForm.tsx:81` consumes the returned address, supplies it to `startTransfer` at lines 106-109, and shows the full address at line 248. `../ens/repos/audit-comp-ens/apps/portal/src/features/transfer/helpers/transferToken.ts:67` then encodes it as the recipient of `safeTransferFrom`.

I report the in-scope failure in Portal: it admits ambiguous encoded-label notation into action-bearing inputs without identity binding. Resolver traversal, Viem encoding, V1 representation, and registrar state are execution context, not my filed root cause.

I demonstrate a locally configured, contract-permitted public hybrid `.zone.eth` subnamespace. I do not claim that it is canonical `.eth` or a currently deployed namespace. A separate bootstrap actor creates the parent infrastructure.

The ordinary `zone.eth` owner creates raw `H` before migration and later uses normal V2 token roles to configure its child registry, `ENSV1Resolver` mirror, and public `ETHRegistrar`. The attacker has no global-root, parent, token, registrar-admin, or namespace-owner authority before or after the exploit.

Before the attacker registers anything, `[H].zone.eth` resolves to the honest address. After the attacker registers only the distinct literal bracket label, the same input selects the attacker's resolver and returns the attacker address. My PoC also makes that resolver return the honest address for its own literal node. This proves that the attacker result depends on the discovery/calldata identity split rather than merely owning the literal resource.

This is not Known Issue `EXP-INPUT-005`, which covers normalization transformations and homograph-style UI confusion "rather than incorrect resolution." Here two distinct on-chain resources are interpreted inconsistently, Portal resolves the honest representation to an attacker-controlled address, and that address is placed into and used by a signed transaction.

## Impact Details

This seems to enter into the Impact: "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient," including "resolving or rendering an ENS name to an attacker-controlled address."

In a pre-existing compatible public hybrid subnamespace, an attacker can monitor a public hash-only representation and register its literal bracket-label counterpart. When a victim uses the Portal-provided copied name as a transfer recipient, Portal resolves and pre-fills the attacker address. After the victim approves the transaction, the victim's ENSv2 name is transferred to the attacker. My PoC demonstrates that NFT loss as the consequence of the wrong-recipient impact.

## Existing Safeguards and Counterarguments

* **"The full address and irreversible-transfer warning are visible."** This reduces likelihood but does not restore recipient provenance. The impact I chose covers a user approving after the application resolves a supplied ENS name to an attacker-controlled address.
* **"The namespace topology is operator-configured."** My PoC demonstrates a contract-permitted local topology, not a live or first-party deployment. Its configuration precedes the attack. The attacker needs no namespace role and uses only the public registrar plus its own resolver.
* **"The contracts and Viem create the mismatch."** They behave consistently with their respective literal and encoded inputs. The in-scope Portal accepts one ambiguous string in an action-bearing field without requiring those identities to match.

## Duplicate Check

EXP-INPUT-005 covers normalization and homograph-style display confusion rather than incorrect resolution. My report resolves encoded labelhash notation to an attacker-controlled address, executes the resulting transfer, and proves final NFT ownership by the attacker.

## Remediation

I recommend treating encoded labelhash notation as display-only unless the resolver stack can prove one identity for DNS traversal and record calldata. As an immediate application-layer fix, reject encoded labelhash components in transfer-recipient, role-grantee, owner, and other authority-bearing inputs, and require an explicit address. Add a regression covering a genuine hash-only V1 child and a separately registered literal `[H]` V2 child.

## References

* `../ens/repos/audit-comp-ens/apps/portal/src/utils/token/isNormalized.ts:3`
* `../ens/repos/audit-comp-ens/apps/portal/src/utils/token/isNameOrAddress.ts:23`
* `../ens/repos/audit-comp-ens/apps/portal/src/features/dashboard/hooks/useV1NamesForAddress.ts:18`
* `../ens/repos/audit-comp-ens/apps/portal/src/utils/names/mergeNamesData.ts:81`
* `../ens/repos/audit-comp-ens/apps/portal/src/features/names/components/NamesTable/columns.tsx:22`
* `../ens/repos/audit-comp-ens/apps/portal/src/components/EntityBadge.tsx:192`
* `../ens/repos/audit-comp-ens/apps/portal/src/features/address/hooks/useAddressResolution.ts:43`
* `../ens/repos/audit-comp-ens/apps/portal/src/features/address/queries/getResolvedAddress.ts:26`
* `../ens/repos/audit-comp-ens/apps/portal/src/features/roles/helpers/addUser.handlers.ts:20`
* `../ens/repos/audit-comp-ens/apps/portal/src/features/transfer/components/SendNameForm.tsx:81`
* `../ens/repos/audit-comp-ens/apps/portal/src/features/transfer/components/SendNameForm.tsx:248`
* `../ens/repos/audit-comp-ens/apps/portal/src/features/transfer/helpers/transferToken.ts:67`

## Link to Proof of Concept

<https://gist.github.com/niafreu/6a0880e49c34d8d7944ec31f27f77727>

## Proof of Concept

## Proof of Concept

My PoC is available in this [gist](https://gist.github.com/niafreu/6a0880e49c34d8d7944ec31f27f77727). It starts a fresh loopback Anvil process, compiles the test-only controls and pinned ENSv2 contracts, and imports the affected production Portal source from `../ens/repos/audit-comp-ens`.

Prerequisites are Foundry, target dependencies installed under `../ens/repos/audit-comp-ens/apps/portal/node_modules`, the target checkout at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, and the `ensdomains/contracts-v2` checkout at `48b3e2d39513b9dd32ef1850877a29009bc807b9` with Foundry dependencies installed. Set `TARGET_ROOT` and `CONTRACTS_ROOT` when those checkouts are not in the default supplied layout.

Download all five Gist files into one directory, then run from that directory:

```bash
TARGET_ROOT=/path/to/audit-comp-ens \
CONTRACTS_ROOT=/path/to/contracts-v2/contracts \
POC_PORT=19951 \
../ens/repos/audit-comp-ens/apps/portal/node_modules/.bin/vitest run \
  --config "$PWD/vitest.config.ts" --reporter=verbose
```

My test performs and asserts these steps:

1. Bootstrap creates parent infrastructure. The ordinary `zone.eth` owner creates the raw-`H` V1 child before migration. Its normal V2 token roles then set its supplied child registry and `ENSV1Resolver` mirror, and its own child registry authorizes an operator-deployed supplied public `ETHRegistrar` instance. This is a locally demonstrated, contract-permitted configuration; no live deployment is claimed.
2. It verifies that literal and decoded nodes differ; the attacker has zero global-root, parent-root, zone-token, registrar-admin, or namespace-owner authority before the attack; the literal label is unowned; and production `resolveAddressOrName` resolves `[H].zone.eth` to the honest address.
3. The attacker calls that operator-deployed public supplied `ETHRegistrar` instance's normal `commit` and `register` functions for the distinct literal 66-byte `[H]` string, assigns its own resolver, and configures that resolver to answer the decoded node with the attacker address. Zero commitment age and free price/payment controls accelerate timing and payment only; they grant no authority and do not change label identity or the registrar call path. The same authority-absence assertions run again after registration.
4. The test runs production `resolveAddressOrName` and `getResolvedAddress` and obtains the attacker address. It executes production `mergeNamesData` and `EntityBadge`; a minimal mock records the value supplied by EntityBadge's real Copy action, and that exact string is pasted into the real `SendNameForm`. The test confirms the full attacker address and irreversible-transfer warning are visible, clicks "Transfer name," and confirms the form supplies the attacker as recipient.
5. It calls production `transferToken`, decodes the request as ERC-1155 `safeTransferFrom(victim, attacker, tokenId, 1, 0x)`, broadcasts that exact captured request from the victim on local Anvil, and reads final `ownerOf(tokenId)` as the attacker.

I expect one passing test. The JSON evidence block shows the honest `beforeLiteralRegistration`, `attackerRootRoles: "0"`, the EntityBadge-derived `copiedRecipientName`, attacker-controlled `portalResolvedAddress`, `decodedFunction: "safeTransferFrom"`, attacker `decodedRecipient`, a successful `broadcastHash`, and attacker `finalOwner`.
