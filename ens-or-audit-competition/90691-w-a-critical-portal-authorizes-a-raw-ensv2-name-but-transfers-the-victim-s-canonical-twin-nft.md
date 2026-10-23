# 90691 w a critical portal authorizes a raw ensv2 name but transfers the victim s canonical twin nft

**Submitted on Aug 26th 2026 at 12:50:46 UTC by @Jornason for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90691
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs

## Description

### Brief/Intro

ENS Portal uses two different ENS resources in one ownership-transfer flow. It authorizes the connected wallet as owner of the exact raw name in the route, but later normalizes that name before checking the transfer role and obtaining the ERC-1155 token ID. The confirmation continues to display the raw route name. An attacker can register a byte-distinct raw label such as `Foo.eth`, transfer that raw NFT to a victim who already owns canonical `foo.eth`, and send the victim the official Portal link for returning `Foo.eth`. The victim sees `Foo.eth` on the page and in the transaction confirmation, enters the attacker's address, and approves one ordinary Transfer transaction. Portal instead submits `safeTransferFrom` for `tokenId(foo)`. The transaction succeeds. The victim keeps the displayed raw `Foo.eth` token but loses the canonical `foo.eth` NFT and its transfer role to the attacker. This is an application-created wrong-asset transaction with terminal NFT theft, not a display-only homograph issue.

### Vulnerability Details

The transfer pipeline changes identity between its authorization and execution stages. The required invariant is simple: owner authorization, permission authorization, confirmation, and calldata must all refer to the same exact ENS resource. Portal violates that invariant before the wallet request is created.

#### One-screen identity split

| Transfer stage                    | Resource Portal uses     |
| --------------------------------- | ------------------------ |
| Route and owner authorization     | raw `Foo.eth`            |
| Page and confirmation text        | raw `Foo.eth`            |
| Transfer-role authorization       | canonical `foo.eth`      |
| Prepared ERC-1155 token ID        | canonical `tokenId(foo)` |
| Successful on-chain transfer      | canonical `foo.eth` NFT  |
| Asset left in the victim's wallet | raw `Foo.eth` NFT        |

#### Demonstrated attack path

{% stepper %}
{% step %}

## The victim owns canonical `foo.eth`

The victim normally owns canonical `foo.eth` and its ordinary transfer role.
{% endstep %}

{% step %}

## The attacker registers raw `Foo.eth`

The attacker registers byte-distinct raw `Foo.eth`. The deployed ENSv2 registrar assigns it a token ID different from canonical `foo.eth`.
{% endstep %}

{% step %}

## The attacker delivers the raw NFT

The attacker unilaterally transfers only the raw `Foo.eth` NFT to the victim and sends the direct Portal route `https://<portal>/Foo.eth/ownership/transfer`, asking the victim to return the mistakenly delivered name.
{% endstep %}

{% step %}

## Portal verifies raw ownership

Portal verifies that the connected victim owns raw `Foo.eth` and displays its transfer form.
{% endstep %}

{% step %}

## The victim enters the attacker's address

The victim enters the attacker's address and opens the transaction confirmation.
{% endstep %}

{% step %}

## The confirmation displays the raw name

The confirmation says `Transfer name - Foo.eth`; the victim approves the one wallet transaction, intending to return that exact raw asset.
{% endstep %}

{% step %}

## Portal transfers the canonical NFT

Portal submits and mines `safeTransferFrom(victim, attacker, tokenId(foo), 1, 0x)` instead.
{% endstep %}
{% endstepper %}

There is no failed transaction, Retry, race, stale indexer assumption, malicious resolver, privileged initial configuration, or post-failure victim action. The attacker's preparation occurs before the victim interaction; the victim-side exploit is a single documented Transfer flow with one successful wallet approval.

#### Root cause

1. `apps/portal/src/routes/$name.tsx:25-44` does not reject a non-normalized 2LD before nested routes load. An invalid `isValidEnsName(name)` result merely skips the TLD redirect and owner lookup still proceeds.
2. `apps/portal/src/routes/$name/ownership/transfer.tsx:22-27` passes the exact raw route parameter to `getEnsOwnerQueryOptions({ name })`.
3. `apps/portal/src/routes/$name/ownership/transfer.tsx:103-110` authorizes the connected wallet against the owner returned for that raw name.
4. `apps/portal/src/utils/ens/resolveEnsOwner.ts:44-50` forwards the exact name to ENSJS v2 `getOwner`. The pinned ENSJS implementation calls `packetToBytes(name)` directly; it does not normalize the string first.
5. `apps/portal/src/features/transfer/hooks/useCanTransferName.ts:55-73` calls `getLabel(name)`. `apps/portal/src/utils/token/getLabel.ts:10-19` first runs `ens_normalize(name)`, so `Foo.eth` becomes canonical label `foo` before the transfer-role check.
6. `apps/portal/src/features/transfer/hooks/useTransferName.ts:109-115` repeats `getLabel(name)` before `getEnsTokenId`, binding the plan to canonical `tokenId(foo)`.
7. `apps/portal/src/features/transfer/hooks/useTransferName.ts:174-180` still constructs the confirmation text from the original raw `name`.
8. `apps/portal/src/features/transfer/helpers/transferToken.ts:63-83` encodes the prepared canonical token ID into ERC-1155 `safeTransferFrom` and sends it to the registry.

The resulting dataflow is:

```
routeName = raw Foo.eth
ownerSubject = packetToBytes(raw Foo.eth)
displaySubject = raw Foo.eth

label = getLabel(routeName) = canonical foo
roleSubject = labelhash(canonical foo)
tokenSubject = getTokenId(labelhash(canonical foo))

walletCall = safeTransferFrom(victim, attacker, tokenId(canonical foo), 1, 0x)
```

#### Runtime proof

The end-to-end PoC ran the real Portal against deployed ENSv2 bytecode on a pinned local Sepolia fork. The target-provided mock-wallet test connector only supplied the connected Anvil EOA; no transfer logic was patched or replaced.

The concrete witness was:

```
raw name displayed:       R7m3k96vzuzzvrht.eth
canonical name:           r7m3k96vzuzzvrht.eth
raw token ID:             0x9d63aa50a1194ea52e984125954c1fb2c7be5933ed1a2a4b72f1a3dc00000000
canonical token ID:       0xc11caef758f1c3c0073df2e40fea56daa2c5e670fda7c92d5915ea8a00000000
confirmation:             Transfer name - R7m3k96vzuzzvrht.eth
submitted token ID:       canonical token ID
local transaction:        0xadfb755cfef2fe8370cd00ea56624f9b8d9bdaabae6479127d34de7512d6d588
receipt:                  success
```

The terminal state was:

| State                            | Before victim approval | After successful receipt |
| -------------------------------- | ---------------------: | -----------------------: |
| Victim raw-token balance         |                      1 |                        1 |
| Victim canonical-token balance   |                      1 |                        0 |
| Attacker canonical-token balance |                      0 |                        1 |
| Victim canonical transfer role   |                   true |                    false |
| Attacker canonical transfer role |                  false |                     true |

The decoded wallet request has selector `0xf242432a` (`safeTransferFrom`), recipient `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`, amount `1`, and the canonical token ID. That ID differs byte-for-byte from the raw token ID shown throughout the UI.

#### Controls

Two differential controls use the same Portal, wallet connector, fork, registry, victim, and attacker:

1. A normal canonical route displays, prepares, and successfully transfers the same canonical token ID.
2. A raw-only token with no canonical sibling passes raw owner resolution but fails the normalized role check. Portal displays `Transfer not available`, exposes no recipient form, and sends no transaction.

These controls prove that the exploit requires the specific raw-owner/canonical-token identity split and is not a generic test-harness or ERC-1155 artifact.

#### Recommended remediation

{% stepper %}
{% step %}

## Reject non-normalized route names

Reject any non-normalized route name before any nested `$name` route or query loads, while preserving explicitly supported encoded-labelhash representations.
{% endstep %}

{% step %}

## Bind transfers to one immutable identity object

Independently bind a transfer to one immutable identity object containing the exact name bytes, registry, resource/label hash, and versioned token ID.
{% endstep %}

{% step %}

## Authorize, display, and encode the same resource

Perform owner and role authorization against that exact resource, display the same resource, and encode only its already-bound token ID.
{% endstep %}

{% step %}

## Validate before opening the wallet

Before opening the wallet, assert that the displayed name resolves to the exact token ID in the transaction plan.
{% endstep %}

{% step %}

## Add a regression test

Add a regression with simultaneously registered `Foo.eth` and `foo.eth`; the route must either reject raw `Foo.eth` or transfer the raw token it displays, never canonical `foo.eth`.
{% endstep %}
{% endstepper %}

### Impact Details

Recommended selected impact: **Direct theft of user NFTs (Critical)**.

Portal's own transaction transfers a valuable ENSv2 NFT that the victim did not select. The attacker becomes the canonical name's owner and receives its transfer capability; the victim has neither the NFT nor the role needed to recover it. Recovery therefore requires the attacker's cooperation. A normal `setAddr`, role edit, reload, or repeat Transfer cannot restore the stolen canonical NFT.

The behavior also matches the program's Critical example for an application displaying an incorrect name or transaction detail that causes a user to submit an unintended transaction. The UI and confirmation identify raw `Foo.eth`, while calldata and final state concern canonical `foo.eth`. The primary impact selection remains direct NFT theft because the PoC proves the asset movement itself.

No downstream fund theft or incorrect resolution is needed for this impact claim.

#### Why this is not public known issue EXP-INPUT-005

The public disclosure for `EXP-INPUT-005` says name validators accept deceptive characters that normalization later rejects or transforms, and explicitly limits the practical impact to homograph-style UI confusion "rather than incorrect resolution." This finding differs in ingress, root cause, sink, and terminal impact:

| Dimension             | Public EXP-INPUT-005                          | This finding                                                                                                    |
| --------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Ingress               | a name-input validator accepts deceptive text | an already registered raw on-chain resource is opened through a public nested route; no input validator is used |
| Root                  | validation and normalization disagree         | owner authorization and transaction asset are bound to different registered resources                           |
| Decisive code         | validators in Manager/Explorer                | owner route, role gate, token-ID preparation, and ERC-1155 transfer sink                                        |
| Sink                  | UI display confusion                          | successful `safeTransferFrom` for the victim's different NFT                                                    |
| Terminal impact       | publicly described as display-only            | canonical NFT and transfer role move to the attacker                                                            |
| Sufficient narrow fix | align/reject deceptive validator inputs       | preserve one exact resource identity through authorization, display, and calldata                               |

The exploit does not rely on the victim failing to distinguish uppercase from lowercase. Even a victim who reads the exact raw spelling correctly is shown that raw spelling while the application encodes a different token ID.

A broad parent-route rule rejecting every non-normalized name would mitigate both issues, but that does not make a validator-only display issue the same as this cross-resource transfer. I cannot inspect the private `EXP-INPUT-005` ticket or proposed patch, so this comparison is limited to the program's public description.

#### Triage-facing clarifications

* **Not a manufactured victim sequence:** registration and delivery are attacker-controlled setup. The victim performs one ordinary return-transfer flow and approves one successful transaction.
* **Not self-repairable:** the terminal owner and transfer role belong to the attacker. The raw decoy remaining with the victim does not confer control over canonical `foo.eth`.
* **Not display confusion alone:** exact calldata decoding, a successful receipt, ERC-1155 balance deltas, owner reads, and role reads all prove movement of the wrong NFT.
* **Not caused by modified application code:** the five transaction-decisive Portal files executed by the PoC hash identically to frozen competition source. PoC files only seed local state, drive the UI, capture RPC, and verify post-state.
* **Not dependent on dashboard inclusion:** `/$name/ownership/transfer` is an in-scope public route. The parent and nested route accept the raw on-chain name and render the real transfer form.
* **Not a contract-only harm:** the registrar merely supplies two valid distinct resources. The in-scope Portal affirmatively chooses the canonical token ID and creates the wrong-asset wallet request.

### References

Scope, source, and public-fix check:

* In-scope asset: `apps/portal` (Explorer app / Websites & Apps).
* Frozen competition source: `cda79acaad59711b943fc68207ebb3f1d0ff8596`.
* Public `audit-comp-ready` head checked on 2026-08-26: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.
* The public delta after the frozen revision contains only Hono/package-manifest and lockfile changes; no public owner-binding, route-normalization, or transfer-token fix exists.
* The decisive Portal files in the executed checkout and frozen competition source have identical SHA-256 hashes.

Links:

* Program scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* In-scope Portal: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Parent name route: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/routes/%24name.tsx>
* Transfer route: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/routes/%24name/ownership/transfer.tsx>
* Owner resolution: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/utils/ens/resolveEnsOwner.ts>
* Transfer-role check: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/hooks/useCanTransferName.ts>
* Transfer preparation: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/hooks/useTransferName.ts>
* ERC-1155 transfer helper: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/helpers/transferToken.ts>

## Link to Proof of Concept

<https://gist.github.com/Jornason/abd695924da1408b30385c6877f76343>

## Proof of Concept

### What this PoC proves

The linked Gist bundle drives the real Portal against a local Anvil fork pinned to Sepolia block `11570503` (hash `0xba286ed6ee35619278654460d15d29d0b2fa90d9ed2f01f136bf083bb898839e`). It registers simultaneous raw/canonical twins using deployed ENSv2 bytecode, delivers the attacker's raw token to the victim, opens the real raw transfer route, captures the exact Portal wallet request, mines it locally, and independently verifies the receipt and terminal state.

All writes are local. No Sepolia transaction is broadcast, no production user or site is contacted, and no real user key is used.

### Reproduction steps

{% stepper %}
{% step %}

## Clone and install dependencies

Clone the in-scope repository, check out `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, and install dependencies.
{% endstep %}

{% step %}

## Install required tooling

Install Python Playwright plus Chromium and ensure Foundry is available.
{% endstep %}

{% step %}

## Run the one-command runner

Download the linked Gist, restore the `scripts/` and `evidence/` layout per its README, and run the one-command runner with the checkout and an archive RPC:

```bash
chmod +x run-poc-r7m3k9.zsh
ENS_SOURCE_ROOT=/absolute/path/to/audit-comp-ens \
ENS_POC_PYTHON=python3 \
ENS_POC_ARCHIVE_RPC=https://your-sepolia-archive-rpc.example \
./run-poc-r7m3k9.zsh
```

{% endstep %}

{% step %}

## Observe the terminal oracles

```
SEED_RESULT=PASS_PRECONDITIONS_R7M3K9
PASS_REAL_PORTAL_SINGLE_TRANSFER_R7M3K9
PASS_L5_RAW_UI_CANONICAL_NFT_THEFT_R7M3K9
CONTROL_SEED_RESULT=PASS_R7M3K9
PASS_NEGATIVE_AND_NORMAL_CONTROLS_R7M3K9
PASS_CHAIN_CONTROLS_R7M3K9
PASS_REVIEWER_PIPELINE_R7M3K9
```

{% endstep %}
{% endstepper %}

### Captured exploit steps and evidence

{% stepper %}
{% step %}

## Register raw and canonical twins

The seed script registers canonical `r7m3k96vzuzzvrht.eth` to the victim and byte-distinct raw `R7m3k96vzuzzvrht.eth` to the attacker. It proves their token IDs differ.
{% endstep %}

{% step %}

## Deliver the raw NFT

The attacker locally transfers only the raw NFT to the victim. Pre-state is raw `1`, canonical `1` for the victim; canonical `0` for the attacker. See `seed-state-r7m3k9.json`.
{% endstep %}

{% step %}

## Open the raw transfer route

The browser opens `/R7m3k96vzuzzvrht.eth/ownership/transfer`. Portal displays the raw name and enables the transfer form. See `01-transfer-page-displays-raw-name-r7m3k9.png`.
{% endstep %}

{% step %}

## Open the transaction confirmation

The victim enters the attacker recipient and opens the real transaction modal. It says `Transfer name - R7m3k96vzuzzvrht.eth`. See `02-confirmation-displays-raw-name-r7m3k9.png`.
{% endstep %}

{% step %}

## Submit the Portal request

The victim opens the mock wallet and the single Portal request mines successfully. The modal still reports success for the raw name. See `03-success-still-displays-raw-name-r7m3k9.png` and `browser-evidence-r7m3k9.json`.
{% endstep %}

{% step %}

## Decode the mined request

The verifier decodes the mined request as:

```
function: safeTransferFrom
from:     0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
to:       0x70997970C51812dc3A010C7d01b50e0d17dc79C8
tokenId:  0xc11caef758f1c3c0073df2e40fea56daa2c5e670fda7c92d5915ea8a00000000
amount:   1
data:     0x
```

{% endstep %}

{% step %}

## Verify terminal state

`chain-verification-r7m3k9.json` proves the receipt succeeded, raw stayed with the victim, canonical moved to the attacker, and the canonical transfer role moved with it.
{% endstep %}

{% step %}

## Verify controls

`04-raw-only-control-blocked-r7m3k9.png` proves a raw-only name cannot reach the form without the normalized sibling. `05-normal-control-success-r7m3k9.png` and the control JSON prove a normal route transfers exactly what it displays.
{% endstep %}
{% endstepper %}

### Exact machine verdict

```json
{
  "verdict": "PASS_L5_RAW_UI_CANONICAL_NFT_THEFT_R7M3K9",
  "uiShownName": "R7m3k96vzuzzvrht.eth",
  "roleCheckLabel": "r7m3k96vzuzzvrht",
  "preparedTokenId": "87347059066722313936298309983891943904162053342580060744654204081119908855808",
  "displayedRawTokenId": "71189210557626260531288390020493476451691489673828082926364042426814463737856",
  "receipt": { "status": "success" },
  "balancesAfter": {
    "victimRaw": "1",
    "victimCanonical": "0",
    "attackerRaw": "0",
    "attackerCanonical": "1"
  },
  "capabilitiesAfter": {
    "victimRawTransferRole": true,
    "victimCanonicalTransferRole": false,
    "attackerCanonicalTransferRole": true
  }
}
```

### Attached code and integrity

The complete runner, four TypeScript seed/verifier programs, two Playwright drivers, captured JSON, and screenshots are included in the linked Gist (a flattened copy of `poc_bundle/`): <https://gist.github.com/Jornason/abd695924da1408b30385c6877f76343> — `SHA256SUMS.r7m3k9.txt` there binds every submitted file and verifies in the Gist root as-is; restore `scripts/` and `evidence/` layout before re-running. The runner also verifies SHA-256 for the five transaction-decisive Portal files before it starts.
