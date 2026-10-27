# 91014 w a critical scoped hca sessions are downgraded to owner mode bypassing refund authorization and transferring hca tokens

**Submitted on Aug 28th 2026 at 20:54:30 UTC by @mikio815 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91014
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>
* **Impacts:**
  * Unauthorized takeover of a user's smart account (Hybrid Custody Account) or authenticated session, letting an attacker transfer the victim's names or funds or act as them, such as:

    * adding, replacing, or escalating an account owner or co-owner
    * extending or widening a session key's authority beyond its intended scope or lifetime
    * stealing an authenticated session or bearer token

    (Exploiting the 7-day session-key design as described in the Known Issues is not eligible; a weakness that grants authority beyond its stated lifetime or permissions is in scope.)

## Description

## Summary

The audited transaction manager passes an active scoped HCA session to `@rhinestone/sdk` with `verifyExecutions: true`. Stock SDK `1.8.0` maps that input to an execution-emissary session mode (`0x0205`). The audited dependency patch instead maps a same-chain standalone HCA to plain ERC-1271 owner mode (`0x0201`).

That outer mode is an authorization decision, not only an encoding detail. The production IntentExecutor explicitly treats execution-emissary modes as session-key operations and requires `Paymaster.settleGasRefund_requireCallback()`. It treats plain ERC-1271 as an owner signature and calls `settleGasRefund()` directly because the owner is trusted. The audited patch therefore makes a scoped session signature enter the owner-only settlement branch and skips the callback designed to prevent session keys from draining account funds through refund parameters.

A holder of the valid limited session can submit an approval-only operation, select a refund rate up to the owner-signed ceiling, and choose the refund recipient. The fixed-session validator accepts the scoped signature, but the outer `0x0201` makes the production Paymaster transfer the HCA's payment token without the mandatory session callback. Fresh nonces repeat the debit until the session expires or is revoked, the HCA balance is exhausted, or another configured bound is reached.

## Selected impact

```
Critical | Extending or widening a session key's authority beyond its intended
scope or lifetime
```

The demonstrated authority expansion is from a refund-aware, fixed scoped session to the settlement path reserved for a trusted account owner. The PoC then demonstrates the resulting transfer of user-owned HCA payment tokens through the deployed production contracts.

## Affected in-scope path

```
Audited App revision: 63772fd872af472ced58b009499355f3430c2a86
Audit repository wrapper: cda79acaad59711b943fc68207ebb3f1d0ff8596
Modified dependency: @rhinestone/sdk@1.8.0
App component: packages/transaction-manager
Smart-account release: sepolia-deployment-2026-07-31
Contracts commit: a399be8e8bcff572a56613c66aca7d4528437665
Primary network: Ethereum Sepolia
```

The root cause is in the modified SDK dependency shipped by the audited App. The competition explicitly includes modifications to third-party dependencies, prioritizes transaction-manager and smart-account/session-key paths, and lists Rhinestone as an external dependency. The deployed IntentExecutor and Paymaster are the consumers that materialize the incorrect App-produced authorization mode into the token transfer.

## Attack path

{% stepper %}
{% step %}

## Enable a limited session

The HCA owner enables a normal short-lived fixed session with refund limits for a session signer. The signer does not have the owner key or general wallet authority.
{% endstep %}

{% step %}

## Pass the active session to the SDK

The App passes that active session to the SDK as `experimental_session` with `verifyExecutions: true`.
{% endstep %}

{% step %}

## Emit an owner mode

The audited patch detects a same-chain standalone HCA and emits outer mode `0x0201`, although the stock package emits session mode `0x0205` for the same security flag.
{% endstep %}

{% step %}

## Create an approval-only operation

Before expiry or revocation, the limited session signer creates an approval-only operation for the configured payment token and Paymaster. No ENS registration action is required by the operation policy.
{% endstep %}

{% step %}

## Sign within the configured limits

The session signer signs the operation and refund fields within the owner-configured token, rate, overhead, and per-operation amount ceilings.
{% endstep %}

{% step %}

## Validate the scoped session

`HCAOwnerAndSessionValidator` accepts the fixed-session signature and the constrained approval.
{% endstep %}

{% step %}

## Classify the signature as an owner signature

The IntentExecutor reads only outer mode `0x0201`, classifies the signature as an owner signature, and skips `settleGasRefund_requireCallback()`.
{% endstep %}

{% step %}

## Transfer the HCA payment token

The Paymaster calls `safeTransferFrom(HCA, chosenRecipient, computedAmount)`. A fresh nonce can repeat the same authority expansion while the session remains valid.
{% endstep %}
{% endstepper %}

## Root cause

The App correctly asks the SDK to verify session executions:

```ts
const sessionSigners = signer.session
  ? {
      type: 'experimental_session',
      session: signer.session.session,
      verifyExecutions: true,
    }
  : undefined
```

The audited patch then overrides the session classification for the same-chain standalone case:

```
anyVerifyExecutions == true
standaloneSameChain == true
-> SIG_MODE_ERC1271
-> outer 0x0201
```

The supplied differential oracle executes both the original and patched installed package:

```
audited_patch_verifyExecutions_true=1 (0x0201)
upstream_verifyExecutions_true=5 (0x0205)
audited_patch_verifyExecutions_false=1 (0x0201)
```

The production IntentExecutor's verified source defines the security boundary:

```
execution-emissary modes -> session key -> callback authorization required
plain ERC1271 modes      -> account owner -> direct settlement trusted
```

Because the actual signer remains the scoped session key, `0x0201` communicates the wrong principal to the refund consumer.

## End-to-end impact and controls

The combined test generates a fresh 586-byte signature with the actual patched SDK, deploys the exact SDK-derived HCA through the production factory on a fixed Sepolia fork, enables the fixed session, and passes the unmodified SDK payload through the production HCA validator, IntentExecutor, and Paymaster.

```
Fork block: 11,586,585
Fork block hash: 0xf155a1d5070a7bd1151fa44ae9904c3d3cd74b9eeb022fc59e22d4c014afd2d3
SDK outer mode: 0x0201
SDK/factory HCA: 0x9616a257c71D7D644a7b98433B4942de08721784
App payment token: 0x768F42455A2D082E23ceeF7d51e5787C82d67a39
Recipient delta: 11,595,920 token units
Result: 1 passed; 0 failed
```

The seven causal tests additionally prove:

* approval-only execution transfers HCA tokens;
* two fresh nonces produce two debits;
* changing only the unsigned recipient preserves the signed refund amount;
* session mode `0x0206` reverts with `Unauthorized()` when the callback is absent, while plain `0x0201` pays;
* the maximum allowed session rate produces over ten times the refund from the observed quote control; and
* direct token transfer, a rate above the session limit, and the same path after owner revocation all revert.

These controls isolate the impact to the owner-mode classification while preserving the validator's intended token, rate, operation, expiry, revocation, and nonce restrictions.

## Why the owner-signed ceiling does not authorize owner settlement

The refund ceiling is a bound, not a delegation of the owner's principal. The production IntentExecutor documents that session-key settlement must use a callback because a session key can otherwise manipulate the refund rate and drain account funds. The App also explicitly sets `verifyExecutions: true`.

The scoped policy allows the Paymaster approval needed for a legitimate refund, but it does not require any ENS action in the batch. Treating that limited approval as proof of an owner signature removes the exact per-execution protection selected by the consumer. The PoC therefore keeps every owner-configured ceiling intact and demonstrates an authority-classification bypass rather than an out-of-range value.

## Known-issue distinction

Known issue R2-03 concerns the older prompt-free key being an unscoped full co-owner for seven days. This report does not rely on that broad co-owner model or on extracting a key from browser storage. It uses the fixed refund-aware scoped HCA validator, which rejects arbitrary direct token transfer, and reaches owner settlement only because the audited SDK patch mislabels the scoped signature.

## Recommendation

{% stepper %}
{% step %}

## Preserve session classification

When `verifyExecutions=true`, never classify a same-chain scoped HCA session as plain `SIG_MODE_ERC1271`.
{% endstep %}

{% step %}

## Select a matching session mode

Select an execution-emissary-containing mode that matches the actual signature shape, and add a regression test that the downstream refund path calls `settleGasRefund_requireCallback()`.
{% endstep %}

{% step %}

## Pass signer provenance

Pass authenticated signer provenance to the refund consumer rather than deriving owner-versus-session authority only from an SDK-selected outer mode.
{% endstep %}

{% step %}

## Add defense in depth

Require at least one intended ENS action, bind the refund recipient, and enforce an aggregate session refund budget or use count.
{% endstep %}
{% endstepper %}

## Link to Proof of Concept

<https://gist.github.com/mikio815/23ebebc712bf8e4d05affb1062381b38>

## Proof of Concept

## Requirements

* Git
* Docker
* An archive-capable Ethereum Sepolia RPC URL
* Internet access to clone the pinned repositories, install the two pinned npm packages, and pull the pinned Node and Foundry images

## Run

Place all supplied Gist files in one directory, then run:

```bash
export SEPOLIA_RPC_URL='https://YOUR_ARCHIVE_SEPOLIA_RPC'
bash run_poc.sh
```

The runner uses fixed Sepolia block `11,586,585` and:

{% stepper %}
{% step %}

## Clone the audited App repository

Clone the audited App repository and check out wrapper commit `cda79acaad59711b943fc68207ebb3f1d0ff8596`, which identifies audited App revision `63772fd872af472ced58b009499355f3430c2a86`.
{% endstep %}

{% step %}

## Install and patch the SDK

Install exact `@rhinestone/sdk@1.8.0` and `viem@2.52.2`, save the stock mode-selection file, and apply the audited App's SDK patch.
{% endstep %}

{% step %}

## Run the mode oracle

Run the stock-versus-patched mode oracle with networking disabled.
{% endstep %}

{% step %}

## Verify the contracts

Clone official tag `sepolia-deployment-2026-07-31` recursively and verify the contracts and submodule commits.
{% endstep %}

{% step %}

## Generate a signed payload

Generate a new signed payload through the actual patched SDK with networking disabled.
{% endstep %}

{% step %}

## Execute against production contracts

Send that unchanged payload through the production HCA factory, validator, IntentExecutor, and Paymaster on the fixed fork.
{% endstep %}

{% step %}

## Run additional controls

Run seven additional mode, repeatability, recipient, limit, and revocation controls.
{% endstep %}
{% endstepper %}

The fixed-block hash verifier is limited to 1 CPU, 512 MiB memory/swap, and 128 PIDs. The Node containers are limited to 2 CPUs, 1 GiB memory/swap, and 256 PIDs. The Foundry test containers are limited to 8 CPUs, 8 GiB memory/swap, and 2,048 PIDs.

## Expected result

```
ENS HCA patched SDK mode oracle: PASS
audited_patch_verifyExecutions_true=1 (0x0201)
upstream_verifyExecutions_true=5 (0x0205)
audited_patch_verifyExecutions_false=1 (0x0201)

ENS HCA audited SDK payload generator: PASS
signature_mode=1 (0x0201)
signature_bytes=586

[PASS] testAuditedSdkGeneratedPayloadDebitsProductionHca()
sdk_payload_recipient_delta: 11595920
Suite result: ok. 1 passed; 0 failed; 0 skipped

[PASS] testApprovalOnlySessionTransfersHcaTokensToArbitraryRecipient()
[PASS] testDirectTransferAndOverLimitRefundRemainRejected()
[PASS] testFreshNoncesRepeatTheApprovalOnlyDebit()
[PASS] testGasRefundRecipientIsNotBoundToTheSignedIntent()
[PASS] testOwnerRevocationStopsTheSameApprovalOnlyPath()
[PASS] testSafeSessionModeRequiresCallbackWhilePlainOwnerModePays()
[PASS] testSessionMaximumRateInflatesRefundVersusObservedQuote()
Suite result: ok. 7 passed; 0 failed; 0 skipped
```

The primary positive oracle is not transaction success alone. It requires the actual patched SDK to emit `0x0201`, the SDK-derived and production-factory HCA addresses to match, the production validator to accept the 586-byte scoped-session signature, the HCA debit to equal the external recipient credit, and the nonce to be consumed.

The decisive A/B changes only the authenticated outer mode: plain owner mode `0x0201` transfers tokens, while execution-emissary session mode `0x0206` reaches the mandatory callback path and reverts with `Unauthorized()` when the callback is absent. The other controls require out-of-policy direct transfer, over-limit rate, and revoked-session use to fail.

The complete saved output is included as `poc-output.txt`.
