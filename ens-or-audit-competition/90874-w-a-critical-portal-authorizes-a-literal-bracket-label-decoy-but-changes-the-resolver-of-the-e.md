# 90874 w a critical portal authorizes a literal bracket label decoy but changes the resolver of the embedded hash ens name

**Submitted on Aug 27th 2026 at 18:47:56 UTC by @Jornason for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90874
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

### Brief/Intro

ENS Portal gives one route name two incompatible ENSv2 identities in its Change Resolver flow. Packet-based owner, registry, and current-resolver reads preserve a 66-byte label written as `[64 lowercase hex characters]`, so they operate on a separately registered literal label. The role gate and transaction builder later pass the same displayed label through Viem/ENSJS labelhash helpers, which recognize the bracket notation as an already-encoded labelhash and select the different name whose labelhash is embedded between the brackets.

An attacker can use the public registrar to pay for the literal bracket label and assign it directly to a victim who already owns the embedded-hash target. The victim opens the official Portal route for that delivered decoy, chooses a custom resolver for the displayed object, and confirms one normal Change Resolver transaction. Portal still says `Set resolver for [H].eth`, but its only wallet request calls `setResolver(canonicalId(H), attackerResolver)`, changing the victim's other name. The displayed literal entry `H2 = keccak256("[H]")` remains unchanged. The valuable name immediately resolves through the attacker-controlled resolver even though the attacker had no resolver role or resolver control on it before the Portal action.

This is an application-created wrong-object transaction, not a warning omission, failed cleanup, retained permission, stale cache, or display-only homograph.

### Vulnerability Details

The required invariant is that owner discovery, registry discovery, authorization, confirmation, and calldata construction all remain bound to the same ENS registry entry. A display string is not a typed identity when different helpers assign it different semantics. Portal violates that invariant.

#### Two independent registry entries

For the fixed proof fixture:

```
valuable label V = victimx4n8q2
H                = keccak256(bytes(V))
                 = 0x2815082cb9f036a459268ef51a4e65451f5fbd6c06e8e699c9b49fa32465cb0c

literal label E  = [2815082cb9f036a459268ef51a4e65451f5fbd6c06e8e699c9b49fa32465cb0c]
H2               = keccak256(bytes(E))
                 = 0x4fd6a46b0964315ea0f18d7efd4d7bad70420b3f26e4ca3cb1af8f11d543f092
```

|                              | A: displayed literal decoy   | B: victim's valuable name |
| ---------------------------- | ---------------------------- | ------------------------- |
| Name                         | `[H].eth`                    | `victimx4n8q2.eth`        |
| Registry label bytes         | complete 66-byte ASCII `[H]` | `victimx4n8q2`            |
| Labelhash                    | `H2 = keccak256("[H]")`      | `H`                       |
| Canonical ID                 | `0x4fd6...00000000`          | `0x2815...00000000`       |
| Portal read/display identity | selected                     | not displayed             |
| Portal role/write identity   | not selected                 | selected                  |

The two canonical IDs, token IDs, and registry entries are byte-distinct. The public registrar hashes the complete literal bytes and creates A under H2. There is no on-chain alias between A and B.

#### Demonstrated attack path

{% stepper %}
{% step %}

## Victim owns the valuable name

The victim normally owns valuable ENSv2 name B and therefore has the default `ROLE_SET_RESOLVER` on B.
{% endstep %}

{% step %}

## Attacker deploys a resolver

The attacker deploys a resolver that returns the attacker-controlled address for B.
{% endstep %}

{% step %}

## Attacker registers the literal decoy

Through the normal public `ETHRegistrar.commit()` / `register()` flow, the attacker pays for literal label E, chooses the victim as owner, and gives A a benign initial resolver. This creates A under H2 without modifying B.
{% endstep %}

{% step %}

## Attacker asks the victim to configure the decoy

The attacker asks the victim to configure the newly delivered A with the custom resolver intended for that decoy.
{% endstep %}

{% step %}

## Victim opens the Change Resolver route

The victim opens `/$name/resolver` for `[H].eth`. Portal reads A's owner and benign current resolver and exposes Change Resolver.
{% endstep %}

{% step %}

## Victim selects a custom resolver

The victim selects Use custom resolver, enters the supplied resolver, clicks Save changes, and sees `Set resolver for [H].eth` in the confirmation.
{% endstep %}

{% step %}

## Victim confirms the transaction

The victim confirms once. Portal submits `setResolver(canonicalId(H), attackerResolver)`, not `setResolver(canonicalId(H2), attackerResolver)`.
{% endstep %}

{% step %}

## Valuable name changes resolver

The transaction succeeds because the victim has the normal resolver role on B. B immediately resolves to the attacker; A remains on its benign resolver and still resolves to the victim.
{% endstep %}
{% endstepper %}

There is no failed transaction, Retry, race, indexer freshness requirement, privileged deployment configuration, prior attacker authority, or second victim action.

#### Root cause

1. `apps/portal/src/utils/token/isNormalized.ts:4-56` recognizes `[64hex]` as an encoded-label representation and lets it pass route name validation. Its registration-only guard at `:59-82` does not reject already-existing public registry state in Resolver routes.
2. `apps/portal/src/utils/ens/resolveEnsOwner.ts:20-24` retains only owner, registry address, and protocol version. It does not carry the registry-local label ID, token ID, canonical ID, or DNS bytes discovered during the read.
3. V2 owner discovery at `resolveEnsOwner.ts:44-50` gives the raw route name to ENSJS packet-based reads. Pinned Viem `packetToBytes.ts:30-52` preserves E as one literal 66-byte DNS label; on-chain traversal therefore selects A/H2.
4. `apps/portal/src/routes/$name/resolver.tsx:419-426` similarly reads the current resolver with the raw name, so the page displays A's benign resolver.
5. `apps/portal/src/routes/$name/change-resolver.tsx:42-50` loads that literal owner and registry, but `:48-59` then reduces the route to `name.split('.')[0]` and asks `hasRoles()` about the resulting display label.
6. Pinned Viem `encodedLabelToLabelhash.ts:7-13` recognizes E and returns embedded H. ENSJS `hasRoles.ts:145-156` therefore checks the victim's normal role on B, not its role on displayed A.
7. `apps/portal/src/features/resolver/components/ChangeResolverForm.tsx:323-367` keeps displaying `Set resolver for ${name}` using A.
8. `apps/portal/src/features/resolver/helpers/changeResolver.ts:63-77` calls `labelToCanonicalId(E)`, repeats the encoded-label interpretation, and encodes `setResolver(canonicalId(H), resolverAddress)`.
9. The EOA transaction manager preserves `to` and `data` into the wallet request (`packages/transaction-manager/src/actors/eoa-transport.actor.ts:50-70`).
10. `PermissionedRegistry.setResolver()` checks the role and changes the resolver of the entry selected by that same erroneous `anyId`, so the request succeeds against ordinary victim-owned B state.

The dataflow is:

```
route/display name A = literal [H].eth
  -> packetToBytes(A) preserves literal [H]
  -> owner/registry/current resolver reads select H2
  -> Portal discards H2 and keeps only the display string
  -> labelToCanonicalId([H]) decodes embedded H
  -> confirmation still names A
  -> setResolver(canonicalId(H), attackerResolver)
  -> B changes; A does not
```

#### Full-pipeline runtime proof

The PoC ran the unmodified Portal and real transaction manager against a fresh Anvil fork of the actual ENSv2 Sepolia contracts at block `11570503`. The fork block hash was independently checked as `0xba286ed6ee35619278654460d15d29d0b2fa90d9ed2f01f136bf083bb898839e`. All registration, deployment, and state-changing transactions occurred only on the local fork; the archive endpoint was read-only.

The runner verifies SHA-256 hashes of every decisive Portal, ENSJS, and Viem file before execution. The executed checkout was public head `1c9b47f18fcddd2e864dfe385c4171061c9811ae`; those files byte-match frozen competition revision `cda79acaad59711b943fc68207ebb3f1d0ff8596`.

The fixed witness was:

```
displayed name:       [2815082cb9f036a459268ef51a4e65451f5fbd6c06e8e699c9b49fa32465cb0c].eth
displayed resolver:   0x1bf32a79CbA9198eFE2bcaDA987E80c940Cb3FeF
confirmation:         Set resolver for [2815082c...465cb0c].eth
wallet requests:      1
wallet target:        ENSv2 Registry 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
decoded function:     setResolver
decoded anyId:        0x2815082cb9f036a459268ef51a4e65451f5fbd6c06e8e699c9b49fa300000000
decoded resolver:     0x28A15edC0DaAf81eb684C2B59EdC2680c1Aed971
local transaction:    0xd92e0566802621695183e656ea14ceb4a4a92f296bb4e05dcc04b57f2fb59925
receipt:              success (0x1)
Playwright pageErrors: 0
```

Captured Portal calldata:

```
0xbc7b6d622815082cb9f036a459268ef51a4e65451f5fbd6c06e8e699c9b49fa30000000000000000000000000000000028a15edc0daaf81eb684c2b59edc2680c1aed971
```

Correct literal-entry calldata:

```
0xbc7b6d624fd6a46b0964315ea0f18d7efd4d7bad70420b3f26e4ca3cb1af8f110000000000000000000000000000000028a15edc0daaf81eb684c2b59edc2680c1aed971
```

The captured request byte-equals the locally mined transaction input and differs from the correct literal request in the complete `anyId` word.

The state transition was:

| State                               | Before victim confirmation | After successful receipt |
| ----------------------------------- | -------------------------- | ------------------------ |
| Resolver of B                       | benign resolver            | attacker resolver        |
| Resolver of displayed A             | benign resolver            | benign resolver          |
| Address resolution of B             | victim                     | attacker                 |
| Address resolution of A             | victim                     | victim                   |
| Attacker resolver capability over B | false                      | true                     |

The full pipeline passed twice in independent directories:

```
run-20260827T164510Z-79235
run-20260827T170207Z-83492
```

#### Controls

The same runner executes four controls:

1. A normal Portal Change Resolver flow for `controlx4n8q2.eth` displays, encodes, and modifies that same canonical resource.
2. A direct call using `canonicalId(H2)` changes the literal entry, proving the contracts do not inherently alias A and B.
3. After revoking the victim's resolver role on H, the same erroneous B-targeting call reverts, proving the primary success consumes B's authority.
4. The parser artifact proves DNS bytes retain complete literal E and select H2, while the encoded-label parser returns H.

#### Test-boundary disclosure

The target-provided mock wallet supplies only the connected victim development EOA and captures the real `eth_sendTransaction`. The real Change Resolver route, form, intent builder, ENSJS/Viem encoding, transaction manager, wallet request, mined registry call, receipt, registry state, and Universal Resolver result all execute unchanged. External GraphQL/funding endpoints are locally stubbed because the direct Resolver path does not rely on them for owner, role, resolver, or transaction identity. Other external HTTPS requests are blocked. No vulnerable logic or decisive response is mocked.

#### Recommended remediation

1. Return the exact registry-local label ID or canonical resource from V2 owner and registry discovery and carry it through authorization and settlement.
2. Never rebuild a transaction target from a display label. Use the bound ID for `hasRoles()` and `setResolver()`.
3. Reject an existing literal bracket label when packet identity and encoded-label identity differ, or render it under an unambiguous typed representation.
4. Before confirmation and submission, assert that the packet-resolved entry, role resource, displayed entry, and calldata `anyId` are identical.
5. Show the bound resource ID and registry in the transaction review UI.
6. Add a regression with both `V.eth` and literal `[labelhash(V)].eth`; changing either resolver must never modify the other.

### Impact Details

Recommended primary impact: **Malicious interactions with an already-connected wallet, including modifying transaction arguments or submitting a malicious transaction - Critical**.

Also select the exact supporting Critical row: **Causing the application to display or pre-fill an incorrect address, name, or transaction detail so the user submits a transaction to an unintended target, including resolving an ENS name to an attacker-controlled address**.

Portal changes the decisive registry-resource argument before wallet submission. The victim selects and confirms A, while calldata changes B. This is not a wallet UI limitation or user typo: the application itself derives and submits an `anyId` that is byte-distinct from the object it read and displayed.

The transaction creates a capability the attacker did not have. Before the victim action, B used a benign resolver and resolved to the victim. After Portal's single successful request, B uses the attacker resolver and resolves to the attacker; displayed A is unchanged. That gives the attacker immediate control over B's resolution and can redirect users or downstream integrations until the victim detects and repairs the resolver.

The legitimate owner can later replace the resolver after discovery, so this report does not claim permanent irreversibility or direct NFT/fund theft. The security impact occurs at successful settlement: Portal has already commissioned the wrong-object state change and installed the attacker's resolver on B without any warning that the confirmed object and modified object differ.

#### Why this is not public Known Issue EXP-INPUT-005

The public `EXP-INPUT-005` disclosure describes name validators accepting deceptive characters or spellings that normalization later rejects or transforms, and states that the practical impact is homograph-style display confusion rather than incorrect resolution. This finding differs in ingress, parser split, sink, and terminal state:

| Dimension      | Public EXP-INPUT-005                     | This finding                                                            |
| -------------- | ---------------------------------------- | ----------------------------------------------------------------------- |
| Ingress        | validator accepts deceptive user text    | already-registered, victim-owned ENSv2 object opened in Change Resolver |
| Representation | homoglyph/control/normalization mismatch | exact literal `[64hex]` DNS label versus encoded-labelhash syntax       |
| Root           | validation differs from normalization    | packet discovery selects H2; role/write helpers select embedded H       |
| Sink           | publicly described display confusion     | successful `setResolver` on a different registry entry                  |
| Terminal state | explicitly not incorrect resolution      | B resolves through attacker resolver; displayed A remains benign        |
| Narrow fix     | align validators/normalization           | preserve the typed registry entry from discovery to calldata            |

Portal source contains a registration-only comment warning that a literal bracket label is ambiguous with encoded-label notation. That guard blocks Portal's own registration form but does not reject public-contract-created state in Resolver routes and does not describe or fix the cross-helper `setResolver` mismatch.

I cannot inspect private Known Issue tickets or private competition reports, so this comparison is limited to public material and does not claim that a private duplicate is impossible.

### References

Scope and revisions:

* In-scope asset: `apps/portal` (Explorer app / Websites & Apps).
* Frozen competition revision: `cda79acaad59711b943fc68207ebb3f1d0ff8596`.
* Executed public checkout: `1c9b47f18fcddd2e864dfe385c4171061c9811ae` with decisive source hashes verified against the frozen revision.
* Pinned fork: Sepolia block `11570503`, hash `0xba286ed6ee35619278654460d15d29d0b2fa90d9ed2f01f136bf083bb898839e`.
* Public state refreshed on 2026-08-28: remote `audit-comp-ready` remains `1c9b47f`; its only post-freeze delta is the Hono dependency update outside Portal/transaction-manager source; the official Known Issues document remains dated 2026-08-14 and contains no literal-bracket/encoded-labelhash/Change Resolver entry.

Links:

* Program scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Program resources: <https://immunefi.com/audit-competition/audit-competition-ens/resources/>
* In-scope Portal: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Name validation: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/utils/token/isNormalized.ts>
* Owner discovery: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/utils/ens/resolveEnsOwner.ts>
* Change Resolver route: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/routes/%24name/change-resolver.tsx>
* Change Resolver form: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ChangeResolverForm.tsx>
* Change Resolver transaction helper: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/helpers/changeResolver.ts>
* Transaction manager: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>

## Link to Proof of Concept

<https://gist.github.com/Jornason/459da23a397bf6d67eeeffc1a7c88aec>

## Proof of Concept

### What this PoC proves

The attached reviewer bundle runs the real Portal Change Resolver flow against the actual ENSv2 contracts on a pinned local Sepolia fork. It proves that Portal reads and displays literal A/H2, confirms A, sends exactly one wallet request for B/H, successfully mines that request, changes only B's resolver, and makes B resolve to the attacker. Four controls rule out inherent contract aliasing, a permissive registry, a generic parser failure, and normal-label behavior.

All writes are local. No public Sepolia transaction, production user, real Safe, real funds, or private user key is used.

### Environment and reproduction

Prerequisites: macOS or Linux; Foundry (`anvil`, `forge`, `cast`); Node 22 with pnpm 10; Python 3 with Playwright and Chromium; an `audit-comp-ens` checkout containing `apps/portal` with dependencies installed; and HTTPS access to a Sepolia archive RPC.

From `poc_bundle/`:

```bash
ENS_SOURCE_ROOT=/path/to/audit-comp-ens \
NODE_BIN_DIR=/path/to/node/bin \
python3 run_portal_anvil_x4n8q2.py
```

`ENS_POC_ARCHIVE_RPC` may override the default read-only endpoint. Foundry and pnpm paths can be overridden with `ANVIL_BIN`, `FORGE_BIN`, `CAST_BIN`, and `PNPM_BIN`. The runner uses dynamic localhost ports and shuts down Portal and Anvil on exit.

### Expected oracles

The runner exits non-zero on any failed assertion. A passing run prints:

```
PASS_L5_PORTAL_LITERAL_BRACKET_WRONG_RESOLVER_X4N8Q2
```

The generated `continuous-result-x4n8q2.json` proves:

```
fork expectedHash == observedHash
before.hashes.canonicalEmbedded != before.hashes.canonicalLiteral
before.attackerCapabilityOnValuable == false
primary.browser.walletSendCount == 1
primary.browser.transactionLabel == "Set resolver for [H].eth"
primary.decodedWalletRequest.function == "setResolver"
primary.decodedWalletRequest.anyId == canonicalId(H)
primary.browser.walletRequest.data == primary.expectedWrongObjectCalldata
primary.browser.walletRequest.data != primary.correctLiteralCalldata
primary.receipt.status == 0x1
primary.after.valuableResolver == attackerResolver
primary.after.literalResolver == benignResolver
primary.after.valuableResolution.address == attacker
primary.after.literalResolution.address == victim
primary.after.attackerCapabilityOnValuable == true
normal control, direct literal control, parser differential, authorization negative == PASS
```

### Retained witness

The packaged witness comes from independent successful run `run-20260827T170207Z-83492`:

1. `registration-evidence-x4n8q2.json` records successful real registrar commit/reveal transactions on the fork, including attacker-paid A with victim owner.
2. `01-resolver-page.png` shows Portal reading the literal object's benign resolver.
3. `02-change-form.png` shows the literal name in the official sidebar while the custom resolver is entered. This image is attached to the report.
4. `04-transaction-confirmation.png` shows the decision boundary still saying `Set resolver for [H].eth`. This image is attached.
5. The mock wallet captures the single request and mines byte-identical calldata; local transaction `0xd92e0566802621695183e656ea14ceb4a4a92f296bb4e05dcc04b57f2fb59925` has receipt `0x1`.
6. `05-transaction-complete.png` shows Portal success for the displayed literal object. This image is attached.
7. `continuous-result-x4n8q2.json` records the B-only resolver and resolution transition plus every control.

The local transaction hash is not a public Sepolia transaction and has no block explorer page.

### Attached code and integrity

The complete bundle — runner, Foundry configuration, fixed-address resolver fixture, retained machine evidence, all ten Portal/control screenshots, and a checksum manifest — is in the linked secret Gist: <https://gist.github.com/Jornason/459da23a397bf6d67eeeffc1a7c88aec> — the Gist is a flattened copy of `poc_bundle/` (restore `src/` and `evidence/` per the layout note in its README) and its `SHA256SUMS.txt` verifies in the Gist root as-is. The four report attachments are byte-identical to the corresponding `primary-*` screenshots in the bundle and are covered by a separate attachments checksum manifest.
