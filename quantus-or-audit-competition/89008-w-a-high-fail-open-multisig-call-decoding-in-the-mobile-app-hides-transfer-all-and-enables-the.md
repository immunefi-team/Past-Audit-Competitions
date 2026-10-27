# 89008 w a high fail open multisig call decoding in the mobile app hides transfer all and enables theft of the full multisig balance

**Submitted on Aug 16th 2026 at 13:48:33 UTC by @sl1de for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89008
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Direct theft of user funds
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions

## Description

### Brief/Intro

A malicious signer who does not have enough approvals to spend from an ordinary multisig can create a proposal that transfers the multisig's full transferable balance to an attacker address. The proposal is a valid `Utility.batch`, but its second child makes the mobile app's recursive display decoder throw. The app suppresses that error, shows only the generic labels `Batch` and `Utility`, omits the amount, recipient, and nested calls, and leaves `Yes, Approve` enabled. If an honest software-wallet signer approves through the normal Android flow and completes LocalAuth, the app submits the exact hidden proposal bytes. The attacker can then execute the proposal. The transfer runs first, the second child interrupts the non-atomic batch, and the earlier transfer is not reverted. I reproduced this end to end on Android with a local evidence entrypoint that mounts the production multisig widgets, SDK, software signer, LocalAuth, and submission services against a private chain. A 50 QUAN multisig was reduced to zero and the attacker-controlled beneficiary gained exactly 50 QUAN.

### Vulnerability Details

#### Attack scenario

The verified scenario uses a normal 2-of-2 ordinary multisig:

* Alice is a legitimate but malicious signer. Alice has one approval and cannot execute alone.
* Bob is the honest signer. Bob's account is stored as a software wallet in the mobile app.
* Charlie is an address controlled by Alice.
* The multisig holds 50 QUAN.

Alice proposes this call:

```
Utility.batch([
  Balances.transfer_all(Charlie, keep_alive=false),
  Multisig.approve(
    multisig_address=0x0000000000000000000000000000000000000000000000000000000000000000,
    proposal_id=0,
    call=0xfa00
  )
])
```

The exact 80-byte call from the verified final Android run was:

```
0x09000802040097bc5f2db1efa23fb71f6737fcb26e41e448aff07447011369df81ce4355546500130200000000000000000000000000000000000000000000000000000000000000000000000008fa00
```

The beneficiary AccountId changes between clean runs, so the 32 bytes after the `transfer_all` prefix are expected to change. The structure and total length remain the same.

The outer SCALE encoding is valid. The runtime can decode the top-level batch, both batch children, and the `Multisig.approve` fields. The `call` field inside `Multisig.approve` is an opaque `Vec<u8>`, so `0xfa00` is legal at this layer. The mobile display decoder goes further and tries to decode that opaque value as another `RuntimeCall`. Variant index `0xfa` is invalid, so display decoding throws even though the stored proposal itself is valid.

#### Root cause and full call path

The failure crosses the SDK's proposal model, chain reader, recursive decoder, and the mobile approval path:

{% stepper %}
{% step %}

## The SDK reads the authoritative proposal bytes

The SDK replaces indexer data with the authoritative proposal bytes from `Multisig.Proposals` storage in `_withOnChainCallData`: `quantus_sdk/lib/src/services/multisig_service.dart:293-321`.
{% endstep %}

{% step %}

## The proposal model suppresses decoding failures

`MultisigProposal.decodedCall` calls `_decodeCall`. `_decodeCall` catches every decoder exception and returns `null`: `quantus_sdk/lib/src/models/multisig_proposal.dart:142-149,268-274`.
{% endstep %}

{% step %}

## The SDK creates an empty display summary

`_withOnChainCallData` treats that `null` result as an empty summary and writes `recipient: ''` and `amount: 0` into the display model: `quantus_sdk/lib/src/services/multisig_service.dart:315-321`.
{% endstep %}

{% step %}

## The approval sheet fetches the same authoritative bytes

The approval sheet independently fetches the same authoritative bytes using `fetchProposalCallBytes`: `mobile-app/lib/v2/screens/multisig/multisig_approve_confirm_sheet.dart:38-53` and `quantus_sdk/lib/src/services/multisig_service.dart:395-412`.
{% endstep %}

{% step %}

## The recursive decoder eagerly decodes every batch child

`CallDecoder._utility` routes the batch to `_batch`, which eagerly describes every child: `quantus_sdk/lib/src/chain/call_decoder.dart:538-545,587-595`.
{% endstep %}

{% step %}

## The opaque inner call causes the display decode to throw

Describing the second child reaches `_multisig(Approve)`, which calls `decodeBytes(call)` on `0xfa00`: `quantus_sdk/lib/src/chain/call_decoder.dart:315-353`.
{% endstep %}

{% step %}

## The approval sheet falls back to the already-empty proposal model

`_decodedProposalCall` catches the resulting exception and falls back to the proposal model. That model has already converted the same failure to `null`, zero, and an empty recipient: `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:183-209`.
{% endstep %}

{% step %}

## The UI renders generic labels without a warning

The sheet therefore renders the generic top-level labels without the transfer amount, beneficiary, child calls, or a decode warning: `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:345-400`.
{% endstep %}

{% step %}

## Confirm remains enabled

The Confirm blocker checks `_submitting`, `_loadingCallBytes`, and `_callBytesFailed`. A display decode exception sets none of these. It only logs the exception, so `Yes, Approve` remains enabled: `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:361-408`.
{% endstep %}

{% step %}

## Bob approves the unchanged hidden bytes

Bob taps the enabled button and completes LocalAuth. `_confirm` passes the unchanged `_callBytes` into the approval submission: `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:249-276`.
{% endstep %}

{% step %}

## The mobile app submits those exact bytes

`TransactionSubmissionService.approveProposal` and `MultisigService.submitApproveExtrinsic` build and submit `Multisig.approve` with those exact bytes: `mobile-app/lib/services/transaction_submission_service.dart:200-230` and `quantus_sdk/lib/src/services/multisig_service.dart:439-448`.
{% endstep %}

{% step %}

## Alice executes after threshold approval

Alice executes after Bob's approval reaches the threshold. `Utility.batch` commits the first successful child before the second child interrupts the batch.
{% endstep %}
{% endstepper %}

The critical fail-open behavior is visible in these two short excerpts. First, the SDK converts an undecodable authoritative call into empty transfer fields:

```dart
final withBytes = p.copyWith(callRaw: call);
final summary = withBytes.decodedCall?.summary;
return withBytes.copyWith(
  recipient: summary?.recipient ?? '',
  amount: summary?.amount ?? BigInt.zero,
);
```

Then the approval sheet logs a decoder exception but does not add it to the blocking state:

```dart
try {
  return CallDecoder.decodeBytes(bytes);
} catch (e) {
  quantusPrint('${widget.logPrefix} could not decode stored proposal call: $e');
}
return widget.proposal.decodedCall;

final blocked = _submitting || _loadingCallBytes || _callBytesFailed;
```

The what-you-see-is-not-what-you-sign gap is therefore direct:

```
Alice stores 80 malicious bytes on chain
             |
             v
Mobile app reads those exact bytes from chain
             |
             v
Recursive display decoding throws and the UI falls back
             |
             v
Bob sees Batch / Utility, with no amount or beneficiary
             |
             v
Bob completes LocalAuth and the app submits those same 80 bytes
             |
             v
Alice executes: transfer succeeds, poison interrupts, transfer remains
```

#### What the victim sees

In the verified Android run, the production proposal and confirmation components showed:

* Proposal list: `-0 QUAN`, `1/2`, and `PROPOSED`.
* Proposal detail: no transfer recipient or amount, one approval from Alice, and an enabled `Approve` button for Bob.
* Final confirmation: `Are you sure?`, `You are about to approve a transfer of`, `Batch`, `Utility`, a network fee, and an enabled `Yes, Approve` button.
* No beneficiary address, no 50 QUAN amount, no `transfer_all`, no nested call list, and no warning that decoding failed.

The vulnerable list row, proposal detail, `MultisigActionConfirmSheet`, LocalAuth call, software signer, and submission services are production components. The local evidence entrypoint supplies throwaway accounts, a run identifier, and loopback endpoints. The visible `Local evidence run c01-final-20260816-07` label is harness provenance and is not presented as production UI.

The local GraphQL fixture supplies only neutral discovery metadata needed to make the proposal available in the app. It is read-only, rejects mutations, does not sign, and does not connect to RPC. It reports the top-level `Utility.batch` identity with empty call bytes, zero amount, and no recipient. The production SDK independently queries `Multisig.Proposals` over the local chain RPC and replaces the fixture call with the authoritative 80 bytes before both display and approval. The omission is not caused by the fixture's amount or recipient fields. Once authoritative decoding returns `null`, `_withOnChainCallData` overwrites any prior values with an empty recipient and zero amount. The same RPC bytes cause the display failure and are included in Bob's submitted approval.

#### Verified chain result

The verified final run used Android API 36, runtime spec 136, transaction version 3, and run ID `c01-final-20260816-07`.

| Item                         | Verified value                                                                                    |
| ---------------------------- | ------------------------------------------------------------------------------------------------- |
| Multisig                     | `qznR3ccVoLuGz2McYD2mr3n1mzyTjSSesp65jicUEzc9KzWFE`                                               |
| Type                         | Ordinary 2-of-2, nonce 77                                                                         |
| Beneficiary                  | `qzntBpmqHZF1jxC8KJKpuxcYuHST892jyXBqRctpAxd1WQ9BL`                                               |
| Payload SHA-256              | `0xc2ce891665a2481a4d1b36bc475836fb36844c10a5424c6ded18f3484af57180`                              |
| Starting multisig balance    | `50000000000000` base units, 50 QUAN                                                              |
| Starting beneficiary balance | `100000000000000000` base units                                                                   |
| Bob approval extrinsic       | `0x80c0be91cf01ab260a26b426e48ef2805a67ebc6ee3c6a4c6f1d02de94ab4553`                              |
| Approval inclusion           | Block 35, extrinsic 1, block `0xdde12c018f7c61a49e9fb4cb247bdd453aca065e2f8275188024a8e8ad3c5dd3` |
| Alice execution extrinsic    | `0x76ffe4aa68a983cfce173e10cc726ab5eb6aa107f204bb3ec1e3853334f44676`                              |
| Execution inclusion          | Block 39, extrinsic 1, block `0x333b71a6f14b8672168a38898a69ac11940c6058606e79010f08f5fbfe82d718` |
| Ending multisig balance      | `0`                                                                                               |
| Ending beneficiary balance   | `100050000000000000` base units                                                                   |
| Beneficiary gain             | `50000000000000` base units, exactly 50 QUAN                                                      |

The attestation decoded Bob's included extrinsic as `Multisig.approve`, verified Bob as the signer, and verified byte-for-byte equality with the stored 80-byte proposal. Its events included `Multisig.SignerApproved`, `Multisig.ProposalReadyToExecute`, and phase-scoped `System.ExtrinsicSuccess`.

Alice's execution emitted these relevant events in order:

```
Balances.Transfer(amount=50000000000000, to=Charlie)
Utility.ItemCompleted
Utility.BatchInterrupted(index=1)
Multisig.ProposalExecuted
System.ExtrinsicSuccess
```

This event order proves that the transfer completed before the poison child failed and that the outer execution still succeeded. The final verifier also confirmed the exact balance delta, proposal removal, unchanged payload, local runtime version, and live chain storage.

#### Preconditions and boundaries

* The target is a funded ordinary multisig.
* The attacker is a legitimate signer but controls fewer approvals than the threshold.
* The victim uses a locally held software-wallet account and completes the normal approval interaction.
* No private key theft, physical device access, malicious RPC, compromised indexer, compromised backend, or attacker-controlled quorum is needed.
* The attacker and victim need enough funds for normal proposal, approval, and execution fees.
* I do not claim this payload bypasses high-security multisig filtering.
* I do not claim a Keystone hardware-wallet display bypass. The verified path is an ordinary multisig approved by a software wallet.

#### Controls that work but do not stop this exploit

* Failure to fetch proposal bytes sets `_callBytesFailed` and disables Confirm. This exploit supplies valid authoritative bytes, so the fetch succeeds.
* Approval byte equality is enforced. This confirms that Bob approved the hidden bytes; it does not confirm that the UI rendered them.
* Cancelling Android LocalAuth prevents submission. In the final run, cancelling produced no approval and left the sheet open with `Authentication required`. Completing LocalAuth then submitted the hidden approval normally.
* High-security multisig filtering rejects this batch. The demonstrated target is the ordinary multisig path.

### Impact Details

This maps directly to both selected Critical impacts.

First, it enables direct theft of user funds. The verified payload used `transfer_all` and reduced the funded multisig from 50 QUAN to zero. The technique is not limited to 50 QUAN. It targets the account's full transferable balance at execution time.

Second, it causes a malicious interaction with an already configured wallet. The victim's wallet approves exact attacker-chosen transaction arguments, including the attacker's beneficiary and `keep_alive=false`, while the confirmation omits those arguments. The attacker starts below threshold and only gains the ability to execute because the honest signer approves through this misleading confirmation.

This does require Bob to perform the normal approval interaction, but that is the interaction the vulnerable confirmation is meant to protect. The program's Critical impact explicitly covers malicious wallet interactions such as modifying transaction parameters and submitting malicious transactions. Here, the app conceals the transaction's only material economic effect before requesting Bob's authentication.

### Recommended Remediation

{% stepper %}
{% step %}

## Treat decode failures as hard approval errors

Treat any failure to fully decode authoritative proposal bytes as a hard approval error.
{% endstep %}

{% step %}

## Block confirmation on unsafe or decode-failed state

Set an explicit unsafe or decode-failed state and include it in the Confirm blocking condition.
{% endstep %}

{% step %}

## Do not replace decoder failures with empty display data

Never replace a decoder failure with zero amount, empty recipient, or a generic top-level label on a signing screen.
{% endstep %}

{% step %}

## Return typed decoding results

Return a typed success or failure result from `CallDecoder` so callers cannot silently turn an exception into display data.
{% endstep %}

{% step %}

## Add regression coverage

Add a regression test using the valid outer call and invalid opaque inner value from this report. Assert that list, detail, software approval, and hardware approval paths all fail closed.
{% endstep %}

{% step %}

## Show secondary evidence only as a supplement

Consider showing the complete call hash and raw bytes as secondary evidence, but do not use raw bytes as a substitute for blocking an undecodable transaction.
{% endstep %}
{% endstepper %}

### References

* [Approval sheet catches decoder errors and falls back](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart#L183-L209)
* [Confirm remains enabled after a decode error](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart#L345-L408)
* [Approval sheet loads and submits authoritative call bytes](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_approve_confirm_sheet.dart#L38-L53)
* [Software-wallet approval submission](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/services/transaction_submission_service.dart#L200-L230)
* [SDK converts a decoder exception to null](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/models/multisig_proposal.dart#L142-L149)
* [SDK clears recipient and amount after the failure](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/services/multisig_service.dart#L293-L321)
* [Recursive decoding of the opaque approve call](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L315-L353)
* [Batch decoder walks every child](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L538-L595)
* [Pinned spec-136 chain source used for the local reproduction](https://github.com/Quantus-Network/chain/tree/b4316f8ed1727a5a55f67f643fd297a8708922b7)

## Proof of Concept

### Safety and environment

This PoC is local and non-destructive. It uses throwaway Alice, Bob, and Charlie accounts, a private spec-136 node, a read-only local GraphQL availability fixture, and an Android API 36 emulator. UID-specific IPv4 and IPv6 firewall rules restrict the app process to the two host loopback services exposed through `adb reverse`. No public Quantus RPC, indexer, account, key, or funds are used.

Attachment `19-c01-poc-source.zip` contains the controller, local fixture, Android entrypoint, build scripts, recorder, verifier, tests, and runtime harness. Generated SDKs, emulator images, build caches, and node databases are excluded. The clean runner deliberately stops before victim approval. Alice's execution is a separate command and is refused unless Bob's included approval has been independently decoded and matched to the exact stored payload.

### Reproduction steps

{% stepper %}
{% step %}

## Use the pinned sources

Use the pinned application source at commit `d5fc4d6572ebc2244e95009a33f5da5048935867` and the pinned chain source at commit `b4316f8ed1727a5a55f67f643fd297a8708922b7`.
{% endstep %}

{% step %}

## Prepare a fresh recorded run

From the supplied engagement directory, set a throwaway emulator-only PIN in the environment. Do not put it in a file or command log. Run the local preflight and prepare a fresh recorded run:

```sh
export C01_TEST_PIN='<temporary emulator PIN>'
C01_REQUIRE_CLEAN_WORKTREES=0 \
  ./poc/c01_realistic_e2e/scripts/preflight.sh
./poc/c01_realistic_e2e/scripts/run_clean_poc.sh prepare c01-review-0001 --record
```

The clean source worktree remains pinned and unchanged. `C01_REQUIRE_CLEAN_WORKTREES=0` is needed only because the isolated application worktree contains the supplied PoC entrypoint and test overlay on top of that exact commit.

`prepare` creates a fresh private chain, creates Alice and Bob's ordinary 2-of-2 multisig, funds it with 50 QUAN, proves control of Charlie, stores the malicious proposal with only Alice's approval, starts the read-only fixture, cold-boots the emulator, builds the Android x86\_64 APK, installs it, and launches the production multisig flow. It then returns at the manual approval gate while the local services and recording remain active. It cannot execute the proposal.
{% endstep %}

{% step %}

## Verify the proposal list

In the Android app, open `Alice and Bob`. Confirm that the proposal row says `-0 QUAN`, `1/2`, and `PROPOSED` even though the stored first child is `transfer_all(Charlie, false)`.
{% endstep %}

{% step %}

## Verify the proposal detail

Open the proposal detail. Confirm that Bob is marked `YOU`, Alice is the only approved signer, the transfer recipient and amount are absent, and `Approve` is enabled.
{% endstep %}

{% step %}

## Verify the hidden confirmation

Tap `Approve`. Wait for the authoritative chain read and fee estimate. The final sheet shows only `Batch`, `Utility`, and the network fee. Confirm that there is no beneficiary, no 50 QUAN amount, no nested call list, and no decoder warning. `Yes, Approve` is enabled.
{% endstep %}

{% step %}

## Cancel Android authentication as a negative control

Tap `Yes, Approve`, then cancel the real Android authentication prompt. The sheet remains open with `Authentication required`, no approval extrinsic is logged, and the proposal remains at one approval. This confirms that LocalAuth itself is active.
{% endstep %}

{% step %}

## Approve the proposal

Tap `Yes, Approve` again and complete Android authentication with the throwaway emulator credential. Wait for the app to submit Bob's approval.
{% endstep %}

{% step %}

## Execute with the separately gated command

On the host, run the separately gated execution command:

```sh
./poc/c01_realistic_e2e/scripts/run_clean_poc.sh execute \
  ./poc/c01_realistic_e2e/runs/c01-review-0001 \
  --confirm-bob-approved
```

Before sending Alice's execution, this command collects the app log, locates the single app-reported approval hash, finds that extrinsic in a local block, recomputes its hash, decodes its signer and call, verifies Bob signed `Multisig.approve` for the expected multisig and proposal, checks exact equality with the 80 stored bytes, and requires phase-scoped `System.ExtrinsicSuccess`.
{% endstep %}

{% step %}

## Verify the execution result

The command then lets Alice execute and runs the independent verifier. The expected result is:

```
verified true
payload_bytes 80
drained_amount 50000000000000
batch_interrupted_index 1
runtime 136 / tx3
live_storage_matches true
beneficiary_control_verified true
approval_evidence_reverified true
execution_evidence_reverified true
```

{% endstep %}

{% step %}

## Inspect the generated verification artifacts

Inspect `controller-state.json` and `verification-report.txt` in the run directory. The verifier requires all of the following before it passes:

* Bob's included approval contains the exact stored 80-byte call.
* Alice, not Bob, signs the later execution extrinsic.
* The transfer event occurs before `BatchInterrupted(index=1)`.
* `ProposalExecuted` and `System.ExtrinsicSuccess` occur in the same execution phase.
* The multisig balance changes from 50 QUAN to zero.
* Charlie's balance increases by exactly 50 QUAN.
* The proposal is removed after execution.
* The raw app log still contains the decoder failure before its one approval submission line.
  {% endstep %}
  {% endstepper %}

### Verified final-run evidence

The completed evidence run is `c01-final-20260816-07`. The primary submission video is a continuous 1920x1080 H.264 walkthrough of the Android emulator beside the controller terminal. It runs for 187.85 seconds, contains 3,757 decoded frames, and has no cuts. Plain local captions identify the important UI and chain transitions.

The public walkthrough is derived from the continuous raw host capture. Its only visual redaction is a solid mask over the disposable emulator credential-entry controls from 120.5 through 130.0 seconds. The genuine Android system prompt, the production app, the controller terminal, and the continuous timeline remain visible. The mask prevents transient PIN-character and key-press echoes from appearing in the submission. The derivation metadata records the exact rectangle, time interval, source hash, output hash, duration equality, and `synthetic_or_generated_media: false`.

The unredacted host capture is retained as a private source and should not be uploaded because it briefly contains the destroyed throwaway emulator PIN. The separate raw Android device recording is safe supporting evidence, but Android blanks the secure credential surface and that recording ends before attacker execution. The public walkthrough is therefore the strongest and safest end-to-end attachment.

Android protects the device-credential surface from `adb screenrecord` and screenshots. This is why raw full-run screenshot 005 and that portion of the device video are black. The public walkthrough shows the first genuine `Authentication required` prompt before any credential entry. It also shows the second prompt heading and `Authenticate to approve` text while only the credential-entry controls are masked. The throwaway credential was destroyed after the run and is not stored in the configuration or evidence JSON.

In the full run, screenshots 001 through 007 are raw 1080x1920 PNGs. Each adjacent JSON file records its capture time, local chain block number and hash, image dimensions, image SHA-256, and successful PNG signature and IHDR CRC checks. Two additional public-safe 1920x1080 PNGs were extracted from the continuous sanitized walkthrough. They show the real Android prompt and the independent final verification.

| Attachment filename                | SHA-256                                                            |
| ---------------------------------- | ------------------------------------------------------------------ |
| `01-c01-sanitized-walkthrough.mp4` | `97f828990e97c2233c9625b7c9361c2d5d83b775c96430ac2bdfccc3b6163e2d` |
| `02-proposal-list.png`             | `133ea41392a427b4578af9baacae0f6f4098e8fb422aec02fe407ae4fd0346e2` |
| `03-proposal-detail.png`           | `5d484c8cb2fec1e2dbd19208e26c7ae77cd58aa46362eba7047bfce26cc8bf7b` |
| `04-approve-enabled.png`           | `e2f3d789ba1891ab8e3cd37b1da43a28e5bed6dccfcd12424a7874a37a9ba375` |
| `05-hidden-confirmation.png`       | `4655d4ebd0a14c9d778a0a411fa1ef6c6a577dc904f865efb67cf27edfa2713f` |
| `06-real-android-auth-prompt.png`  | `4793e28cd1d04d8cc4aeae95193daa844d5dc4b92887c55fe030ccc36f739083` |
| `07-final-verification.png`        | `d1e32a306c29ff738592eabd87ccf2984410bde688ff117c5f6da6008b332beb` |
| `08-verification-report.txt`       | `2fa897e177b2d58ddbd417cd85ded608c935a6e9e7114901ea22dff2006b1fa4` |
| `09-independent-hashes.json`       | `56c9ac50b81a3fbfd0fc7e028dbba506987edea2a058df5a6a558fda210b296d` |
| `10-controller-state.json`         | `2f4985db2d1c90e8817e93afe8ed54f8b12f34cb91084a261c4b5cd57d7aef69` |
| `11-walkthrough-metadata.json`     | `9f8b0b46081fcc595d85eb20f3fd0e9d41229b110fbc637b79bc9294490e1702` |
| `12-app-logcat.txt`                | `ce91a9356106ca628c943a81fcd712c50e2b8d342b7d0393f14f2733ba309026` |
| `13-approval-attestation.txt`      | `93c6becb85c1b3bda283ce12f70e8eee1ce0a7c50c29cb280d9dcd8062976571` |
| `14-execution-transcript.txt`      | `aff37fee9906a7428831dbf368ed008be61ddc2eeb9e2b326a4499337315b008` |
| `15-controller-verification.txt`   | `cc2d7bec38a1ecaff907ded5c9bb25bb19792595251d70508f439fe87771863d` |
| `16-network-isolation.json`        | `52a31eeb921bda13fb3091bb1213a1e71036a8189e4b3dad6344444ccc12f1f1` |
| `17-raw-media-metadata.json`       | `711860e4171bff5766354ada35299ff72bed4c00516762f4b83798aaad5e76d0` |
| `18-full-run-SHA256SUMS.txt`       | `f97517b60719e62a8230c5f3d67ebe06193b858c807621a1d64871febb4c7ac1` |
| `19-c01-poc-source.zip`            | `cbfb298db7367b7f8609d9fe190a970aebb79dd1ee41f451f05b52cb10a13341` |

After the node and fixture were stopped, the privacy-safe walkthrough and two public screenshots were added, and the ephemeral local-node networking identity was excluded, `sha256sum -c SHA256SUMS` passed for all 106 stable files listed in the manifest. Transient run-control files such as PID, lock, and capture-state files are intentionally excluded from the submission manifest. The independent verification report returned `VERDICT: PASS`. It re-read the final state and confirmed the exact 80-byte payload, Bob's approval, Alice's execution, transfer-before-interruption event order, exact 50 QUAN balance delta, proposal removal, runtime 136 and transaction version 3, and beneficiary control.

The network evidence is intentionally stated more narrowly. Android's system app-link verifier logged a failed verification for `www.quantus.com`; this exact `IntentFilterIntentOp` failure is classified separately in `failed_os_app_link_verification_hosts`. Outside that failed OS record, the app evidence contains no forbidden Quantus, Supabase, Firebase, Google API, TelemetryDeck, or Senoti service hostname. UID-specific IPv4 and IPv6 firewall rules allowed the app only to reach ports 9944 and 4350 over loopback, the node had zero peers, and the post-run app socket snapshots contained only local endpoints. The final network JSON records `forbidden_contact_count: 0` under this explicit definition and `passed: true`; it is not presented as a packet-capture claim about unrelated Android system services.

The raw app log binds the UI failure to Bob's included approval. The final decoder failure is at line 1679, followed by the single approval submission at line 1728:

```
[MultisigApprove] could not decode stored proposal call: Exception: RuntimeCall: Invalid variant index: "250"
[Approve] submitted: 0x80c0be91cf01ab260a26b426e48ef2805a67ebc6ee3c6a4c6f1d02de94ab4553
```

The controller then recomputed that extrinsic's Blake2b-256 hash, found it at block 35 index 1, decoded Bob as the signer, decoded `Multisig.approve`, and matched its call field to the stored payload byte for byte. Alice's execution hash was independently recomputed as `0x76ffe4aa68a983cfce173e10cc726ab5eb6aa107f204bb3ec1e3853334f44676` and matched block 39 index 1.
