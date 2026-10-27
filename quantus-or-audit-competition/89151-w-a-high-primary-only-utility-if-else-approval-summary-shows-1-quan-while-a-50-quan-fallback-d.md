# 89151 w a high primary only utility if else approval summary shows 1 quan while a 50 quan fallback drains the multisig&#x20;

**Submitted on Aug 17th 2026 at 14:25:34 UTC by @sl1de for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89151
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

## Brief/Intro

The mobile multisig approval screen can present a validly encoded and fully decodable `Utility.if_else` proposal with a prominent headline of `1 QUAN` and a valid beneficiary even though the exact proposal contains a fallback that immediately transfers `50 QUAN` to that same beneficiary. When the victim has enabled the normal Hide Balances setting, both nested branch amounts are replaced with `-----`, while the unmasked `1 QUAN` headline remains and `Yes, Approve` stays enabled.

The attacker makes the primary 1 QUAN call fail deterministically by requesting a one-block reversible-transfer delay when the runtime minimum is two blocks. `Utility.if_else` then executes the 50 QUAN fallback. I verified this in an ordinary 2-of-2 multisig using the real Android approval flow and LocalAuth: the honest signer approved the exact 100-byte proposal shown by the app, after which the malicious signer executed it and drained the multisig from 50 QUAN to zero.

## Vulnerability Details

### Attack scenario

The verified scenario uses an ordinary 2-of-2 multisig:

* Alice is an existing but malicious signer. Alice has one approval and cannot execute alone.
* Bob is the honest second signer. Bob's software-wallet account is stored in the mobile app.
* The multisig holds 50 QUAN.
* The beneficiary is controlled by Alice.
* Bob has enabled Hide Balances as a normal local privacy preference.

Alice proposes the following validly encoded and fully decodable call:

```
Utility.if_else(
  main = ReversibleTransfers.schedule_transfer_with_delay(
    beneficiary,
    1 QUAN,
    BlockNumber(1)
  ),
  fallback = Balances.transfer_allow_death(
    beneficiary,
    50 QUAN
  )
)
```

Both branches use the same beneficiary. The exact 100-byte SCALE payload used in the Android and current-runtime verification was:

```
0x09060b040097bc5f2db1efa23fb71f6737fcb26e41e448aff07447011369df81ce435554650010a5d4e80000000000000000000000000100000002000097bc5f2db1efa23fb71f6737fcb26e41e448aff07447011369df81ce435554650b00203d88792d
```

Payload SHA-256:

```
0x425c2afad5639ea8329380333ee7a512a26782d01488385e6326e4af1d5d32fe
```

The payload is not malformed. The mobile SDK and current runtime metadata decode both branches completely and re-encode them to the same 100 bytes.

### Root cause

The decoder correctly creates separate transfer summaries for the two nested calls:

* The primary summary contains 1 QUAN and the beneficiary.
* The fallback summary contains 50 QUAN and the same beneficiary.

However, the `Utility.if_else` handler assigns the outer call only the primary call's summary:

```dart
case utility.IfElse(:final main, :final fallback):
  return DecodedCall(
    pallet: 'Utility',
    call: 'if_else',
    fields: [
      NestedCallField('Primary call', describe(main)),
      NestedCallField('Fallback call', describe(fallback)),
    ],
    summary: describe(main).summary,
  );
```

The approval sheet builds its prominent headline from this outer summary. It therefore describes the proposal as a 1 QUAN transfer to the beneficiary even though either branch can be the economic result.

The nested call tree is rendered separately. Every native-token `AmountField` is passed through the global transaction amount display provider:

```dart
case AmountField(:final label, :final token, :final assetId, :final note):
  final value = assetId == null
      ? ref.watch(txAmountDisplayProvider)(token, isSend: true).primaryAmount
      : '$token (asset #$assetId, raw units)';
```

When Hide Balances is enabled, that provider replaces both token amounts with `-----`. This hides the 50 QUAN fallback amount but does not hide or qualify the primary-only 1 QUAN headline.

The final confirmation therefore shows:

```
1 QUAN
beneficiary address

Utility · if_else

Primary call
ReversibleTransfers · schedule_transfer_with_delay
Destination: beneficiary address
Amount: -----
Delay: 1 blocks

Fallback call
Balances · transfer_allow_death
Destination: beneficiary address
Amount: -----

Yes, Approve
```

The fallback call name and beneficiary remain visible. The security problem is that its exact amount is hidden while the screen affirmatively summarizes the transaction as 1 QUAN. Even after reviewing the nested fallback section, Bob cannot verify its amount from this screen.

This is also inconsistent with the application's other authorization screens. The normal send review and multisig proposal-creation review preserve the exact transaction amount even when balance privacy is enabled. The privacy preference is therefore being applied to a security-critical approval argument in this decoded-call tree, rather than merely concealing passive balance information.

The confirmation blocker only considers submission state, call-byte loading, and call-byte fetch failure:

```dart
final blocked = _submitting || _loadingCallBytes || _callBytesFailed;
```

A conditional call whose headline ignores one possible outcome, or whose authorization amounts are hidden, does not enter a blocked state. After Bob completes LocalAuth, `_confirm` passes the unchanged authoritative proposal bytes to the normal approval submission path.

This does not depend on a malicious indexer or RPC. The approval sheet obtains the proposal bytes from `Multisig.Proposals` chain storage. The chain then requires Bob's submitted inner call bytes to match the stored proposal byte for byte.

### Why the fallback executes deterministically

The primary call requests `BlockNumber(1)`. The current runtime defines the minimum reversible-transfer delay as two blocks:

```rust
pub const MinDelayPeriodBlocks: BlockNumber = 2;
```

`schedule_transfer_with_delay` validates the delay before scheduling a transfer. A one-block delay therefore returns `DelayTooShort` before the 1 QUAN transfer changes economic state.

`Utility.if_else` executes its fallback whenever the primary call returns an error. The fallback is a valid `Balances.transfer_allow_death` of 50 QUAN, so it succeeds and the outer `if_else` call returns success.

The resulting chain behavior is:

```
Primary 1 QUAN call
        |
        v
DelayTooShort
        |
        v
Fallback transfer executes
        |
        v
Balances.Transfer of 50 QUAN
        |
        v
IfElseFallbackCalled
        |
        v
ProposalExecuted and System.ExtrinsicSuccess
```

### What the victim sees

On the same installed APK and the same stored proposal, the visible-state control showed:

* Headline: `1 QUAN`
* Primary amount: `-1 QUAN`
* Fallback amount: `-50 QUAN`
* The same beneficiary in the headline and both branches

With Hide Balances enabled, the production confirmation instead showed:

* `You are about to approve a transfer of`
* An unmasked `1 QUAN` headline
* The same valid beneficiary
* `Utility · if_else`
* `ReversibleTransfers · schedule_transfer_with_delay`, delay `1 blocks`, and amount `-----`
* `Balances · transfer_allow_death` and amount `-----`
* An enabled `Yes, Approve` button

The fallback call name and beneficiary are visible, but its exact 50 QUAN amount is not. There is no warning that the primary is invalid or that the fallback will transfer fifty times the headline amount.

The Android API 36 reproduction mounted the audited production multisig proposal components, `CallDecoder`, nested call renderer, privacy provider, software-wallet signer, LocalAuth, transaction submission service, and SDK against a private spec-136 chain.

The local GraphQL fixture supplied only neutral proposal-discovery metadata. It was read-only, could not sign, and did not connect to RPC. The production SDK independently replaced its empty call bytes with the authoritative 100-byte proposal from local chain storage before display and approval.

### Verified chain result

| Item                      | Verified value                                                                                     |
| ------------------------- | -------------------------------------------------------------------------------------------------- |
| Run ID                    | `if-else-api36-20260817-02`                                                                        |
| Android environment       | API 36 evidence build mounting the production approval and signing components                      |
| Runtime                   | Spec 136, transaction version 3                                                                    |
| Multisig                  | Ordinary 2-of-2, Alice initially at 1 of 2 approvals                                               |
| Multisig address          | `qznR3ccVoLuGz2McYD2mr3n1mzyTjSSesp65jicUEzc9KzWFE`                                                |
| Beneficiary               | `qzntBpmqHZF1jxC8KJKpuxcYuHST892jyXBqRctpAxd1WQ9BL`                                                |
| Payload                   | Fully decoded, exactly 100 bytes                                                                   |
| Payload SHA-256           | `0x425c2afad5639ea8329380333ee7a512a26782d01488385e6326e4af1d5d32fe`                               |
| Starting multisig balance | `50000000000000` base units, 50 QUAN                                                               |
| Bob approval              | `0xe647a0a05a9ea0770bb79baeaa3e2731e8a8eae44b42689778bed68ac85f7a4e`                               |
| Approval inclusion        | Block 208, extrinsic 1, block `0x5242fd11340475a975c1366c31446730cd5d4b0b31b7cc36567284ca6d3cf5e4` |
| Alice execution           | `0xef731b0ae0449699dbbab9f138e0c968ba327dc8daa857e843ff8d342222df66`                               |
| Execution inclusion       | Block 318, extrinsic 1, block `0x2f1f3205e8fa1ad9e9c6f8d576aff2aeab29e1bc06edbcfdf057320f3eab6397` |
| Primary result            | `ReversibleTransfers.DelayTooShort`                                                                |
| Utility result            | `Utility.IfElseFallbackCalled`                                                                     |
| Transfer                  | Exactly `50000000000000` base units to the beneficiary                                             |
| Ending multisig balance   | `0`                                                                                                |
| Beneficiary gain          | `50000000000000` base units, exactly 50 QUAN                                                       |
| Final proposal state      | Executed and removed                                                                               |

Bob completed the genuine Android device-credential prompt. Independent decoding located the app-reported approval in block 208, recomputed its hash, identified Bob as the signer, decoded `Multisig.approve`, and matched its inner call to the stored 100-byte payload. Alice's execution then emitted the exact 50 QUAN `Balances.Transfer`, `Utility.IfElseFallbackCalled`, `Multisig.ProposalExecuted`, and phase-scoped `System.ExtrinsicSuccess`.

### Current runtime confirmation

I separately reproduced the same exact payload against the current audit-chain source at commit `3b243b870a5442a6e5f443056f67bbba87d472f5`, runtime spec 143 and transaction version 3.

The current-runtime run confirmed:

* The 100-byte call fully decodes as `Utility.if_else`.
* The ordinary multisig begins below threshold with only Alice's approval.
* Bob's exact approval is `0xe6dbfaeb13293002a386856d3881a0bf2ecbee8ea4d465181684d9528a32673f`.
* Alice's execution is `0xf7ef6cc1e8e678daaa23c599a44c487293f715990e45dcb00640167bafa1fc51`.
* The primary call fails with `ReversibleTransfers.DelayTooShort`.
* `IfElseFallbackCalled` is emitted.
* The fallback transfers exactly 50 QUAN.
* The multisig balance changes from 50 QUAN to zero.
* The beneficiary gains exactly 50 QUAN.
* The proposal is removed and the outer execution succeeds.

The same payload was tested against a high-security multisig and was rejected. The demonstrated vulnerable path is the ordinary multisig approval flow.

### Preconditions and boundaries

* The target is a funded ordinary multisig.
* The attacker is a legitimate signer but controls fewer approvals than the threshold.
* The victim uses a locally held software-wallet account and completes the normal approval interaction.
* The victim has already enabled the user-facing Hide Balances feature. The attacker cannot enable it remotely.
* The fallback call name, beneficiary, and amount placeholder remain visible.
* No private key theft, physical device access, malicious RPC, compromised indexer, compromised backend, altered call bytes, or attacker-controlled quorum is required.
* The attacker and victim need enough funds for normal proposal, approval, and execution fees.
* I do not claim this payload bypasses high-security multisig filtering or a Keystone hardware-wallet display.

### Controls that work but do not stop this exploit

* Failure to fetch proposal bytes sets `_callBytesFailed` and disables approval. This payload supplies valid authoritative bytes, so the fetch succeeds.
* Full recursive decoding succeeds. There is no decoder error for a fail-closed patch to catch.
* Approval byte equality is enforced. This proves Bob approved the misleadingly displayed bytes, but it does not make the semantic presentation accurate.
* With Hide Balances disabled, the same screen reveals both branch amounts. The vulnerable state is the normal privacy-enabled configuration.
* Android LocalAuth prevents submission when authentication fails. Completing the normal prompt still submits the exact misleadingly presented call.
* High-security multisig policy rejects this call. The verified target is an ordinary multisig.

## Impact Details

This maps directly to both selected Critical impacts.

First, it enables direct theft of user funds. In the verified execution, the multisig held 50 QUAN and the fallback transferred all 50 QUAN to the attacker-controlled beneficiary. The multisig ended with a zero balance. The technique is not limited to the demonstrated amount. The fallback can request any chosen amount available to the ordinary multisig.

Second, it causes a malicious interaction with an already-connected wallet. Bob's software wallet approves exact attacker-chosen transaction arguments containing a 50 QUAN fallback while the confirmation headline affirmatively states 1 QUAN. The transaction is not presented as zero, blank, or undecodable. It shows a plausible nonzero amount and a valid beneficiary, and both branches use that same beneficiary.

The relevant preconditions are:

* The target is a funded ordinary multisig.
* The attacker is already a legitimate signer but remains below the approval threshold.
* The victim has enabled Hide Balances.
* The victim opens the normal proposal approval flow and completes LocalAuth using a software-wallet account.
* The parties have enough funds for normal proposal, approval, and execution fees.

The attacker cannot remotely enable Hide Balances, and this is disclosed as a victim-side precondition. The fallback call name and beneficiary remain visible. However, the exact 50 QUAN value that the signer needs to authorize is replaced with a placeholder, while the screen's unmasked economic headline states 1 QUAN. A signer checking the displayed amount and destination can therefore approve what appears to be a 1 QUAN transaction while authorizing a deterministic 50 QUAN transfer.

No stolen key, physical device access, malicious RPC, compromised indexer, compromised backend, altered proposal bytes, or attacker-controlled quorum is required.

## Recommended Remediation

{% stepper %}
{% step %}

## Treat `Utility.if_else` as a multi-outcome call

Never present one branch's summary as the definitive transaction amount.
{% endstep %}

{% step %}

## Always display exact authorization amounts

Always display exact authorization amounts on signing and multisig approval screens, regardless of the balance privacy preference.
{% endstep %}

{% step %}

## Disable unsafe approvals

Disable approval if any economically relevant branch amount is hidden or cannot be summarized safely.
{% endstep %}

{% step %}

## Show outcomes and conditions prominently

Prominently show the maximum possible transfer and the conditions that select each branch.
{% endstep %}

{% step %}

## Add regression tests

Add regression tests for `if_else`, `batch`, `batch_all`, and `force_batch` under every balance-privacy state.
{% endstep %}
{% endstepper %}

## References

* [Primary and fallback transfers receive separate summaries](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L140-L170)
* [Utility.if\_else incorrectly uses only the primary summary](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L572-L580)
* [Nested AmountField values use the privacy-aware display provider](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/components/decoded_call_view.dart#L66-L75)
* [Hide Balances replaces transaction amounts with the hidden placeholder](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/providers/currency_display_provider.dart#L283-L380)
* [The approval sheet uses the primary-only headline and leaves approval enabled](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart#L345-L407)
* [LocalAuth completion submits the unchanged stored call bytes](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart#L249-L275)
* [The approval flow reads authoritative bytes from chain storage](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_approve_confirm_sheet.dart#L38-L53)
* [The SDK fetches and submits the exact stored proposal bytes](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/services/multisig_service.dart#L395-L448)
* [Regular send review renders exact authorization values](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/send/regular_send_strategy.dart#L72-L98)
* [Multisig proposal creation also renders the exact amount](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/send/multisig_propose_strategy.dart#L123-L160)
* [Utility.if\_else executes the fallback after a primary error](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/utility/src/lib.rs#L568-L628)
* [The reversible transfer validates delay before moving funds](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/lib.rs#L467-L485)
* [A delay below the runtime minimum returns DelayTooShort](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/lib.rs#L681-L690)
* [The current runtime minimum delay is two blocks](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L524-L540)

## Proof of Concept

### Safety and environment

I reproduced this entirely in a private local environment using throwaway Alice, Bob, and beneficiary accounts. No public Quantus RPC, indexer, wallet, key, account, or funds were used.

The Android proof used the application source at commit `d5fc4d6572ebc2244e95009a33f5da5048935867`, an Android API 36 emulator, the production multisig approval components, production SDK decoder, software-wallet signer, genuine Android LocalAuth, and a private spec-136 chain. The APK was an isolated evidence entrypoint that mounted those production components, not an unmodified store release. The entrypoint supplied only throwaway account state and loopback endpoints. The application process was restricted to loopback RPC and fixture ports.

The completed Android and chain evidence run ID was `if-else-api36-20260817-02`. Its runtime reported spec 136 and transaction version 3.

I then executed the exact same 100-byte call independently against the current in-scope chain source at commit `3b243b870a5442a6e5f443056f67bbba87d472f5`, runtime spec 143 and transaction version 3. This second run confirms that the payload and runtime behavior remain valid on the current chain.

### Actors and preconditions

The realistic setup is an ordinary 2-of-2 multisig:

* Alice is an existing authorized signer, but is malicious.
* Alice begins with one approval and cannot execute alone.
* Bob is the honest co-signer and holds his account as a software wallet in the mobile app.
* The beneficiary is controlled by Alice.
* The beneficiary address is identical in both branches of the call.
* The multisig holds 50 QUAN.
* Bob has enabled the normal Hide Balances setting in the app.
* Bob completes the ordinary approval interaction and Android authentication.

I do not claim that Alice can remotely enable Hide Balances. The exploit applies when Bob has already enabled this normal user-facing privacy setting.

### Malicious proposal

Alice creates the following validly encoded and fully decodable proposal:

```
Utility.if_else(
  main = ReversibleTransfers.schedule_transfer_with_delay(
    beneficiary,
    1 QUAN,
    BlockNumber(1)
  ),
  fallback = Balances.transfer_allow_death(
    beneficiary,
    50 QUAN
  )
)
```

Both branches use this same beneficiary:

```
qzntBpmqHZF1jxC8KJKpuxcYuHST892jyXBqRctpAxd1WQ9BL
```

The exact 100-byte SCALE call is:

```
0x09060b040097bc5f2db1efa23fb71f6737fcb26e41e448aff07447011369df81ce435554650010a5d4e80000000000000000000000000100000002000097bc5f2db1efa23fb71f6737fcb26e41e448aff07447011369df81ce435554650b00203d88792d
```

Payload SHA-256:

```
0x425c2afad5639ea8329380333ee7a512a26782d01488385e6326e4af1d5d32fe
```

The call is not malformed. It fully decodes as `Utility.if_else`, consumes all 100 bytes, and re-encodes to the identical payload.

The main branch requests a one-block reversible-transfer delay. The runtime minimum is two blocks, so the main branch deterministically returns `ReversibleTransfers.DelayTooShort`. `Utility.if_else` then dispatches the fallback, which immediately transfers 50 QUAN to the same beneficiary.

### Reproduction steps

{% stepper %}
{% step %}

## Create the multisig

Create an ordinary 2-of-2 multisig containing Alice and Bob.
{% endstep %}

{% step %}

## Fund the multisig

Fund the multisig with exactly `50000000000000` base units, equal to 50 QUAN.
{% endstep %}

{% step %}

## Submit the proposal

Have Alice submit the 100-byte proposal above. Confirm that the proposal is active with only Alice's approval. At this point Alice is below threshold and cannot execute it.
{% endstep %}

{% step %}

## Review the visible-state control

As a control, open the proposal with Hide Balances disabled. The mobile confirmation correctly renders:

```
You are about to approve a transfer of
1 QUAN

Utility · if_else

Primary call
ReversibleTransfers · schedule_transfer_with_delay
Amount: -1 QUAN
Delay: 1 blocks

Fallback call
Balances · transfer_allow_death
Amount: -50 QUAN
```

This confirms that both branches decode successfully and that the mobile app knows the fallback amount is 50 QUAN.
{% endstep %}

{% step %}

## Enable Hide Balances

Enable the normal Hide Balances preference, then open the same stored proposal again as Bob.
{% endstep %}

{% step %}

## Observe the misleading confirmation

Observe that the prominent confirmation still says:

```
You are about to approve a transfer of
1 QUAN
```

The same beneficiary is shown, but the nested amounts now render as:

```
Primary amount: -----
Fallback amount: -----
```

The screen contains no visible `50 QUAN` value and no warning that the fallback can transfer fifty times the headline amount. The fallback call name and beneficiary remain visible.
{% endstep %}

{% step %}

## Confirm approval remains enabled

Scroll to the bottom of the sheet. `Yes, Approve` remains enabled even though the fallback amount is hidden.
{% endstep %}

{% step %}

## Approve the proposal

Tap `Yes, Approve`. The genuine Android system prompt displays `Authentication required` and `Authenticate to approve`.
{% endstep %}

{% step %}

## Complete Android authentication

Complete Android authentication using Bob's throwaway emulator credential. The production approval submission path submits Bob's approval.
{% endstep %}

{% step %}

## Verify the approval extrinsic

Independently locate the application-reported approval extrinsic on the local chain, recompute its hash, decode its signer and call, and compare its inner call field with the proposal stored in `Multisig.Proposals`.

The Android approval was:

```
Transaction:
0xe647a0a05a9ea0770bb79baeaa3e2731e8a8eae44b42689778bed68ac85f7a4e

Inclusion:
Block 208, extrinsic 1
Block hash:
0x5242fd11340475a975c1366c31446730cd5d4b0b31b7cc36567284ca6d3cf5e4

Signer:
qzkYEQv8tQsmniZYdame3Cku18RL5g9bGK9Pdydq5TMPdpE3y
```

The attester verified:

```
Outer call: Multisig.approve
Inner payload length: 100 bytes
Inner payload SHA-256:
0x425c2afad5639ea8329380333ee7a512a26782d01488385e6326e4af1d5d32fe
Exact stored payload: true
System.ExtrinsicSuccess: true
Approvals after inclusion: 2/2
```

This proves that Bob's real software wallet approved the exact attacker-created bytes shown as a 1 QUAN transfer.
{% endstep %}

{% step %}

## Execute the approved proposal

After Bob reaches the threshold, let Alice separately execute the approved proposal.

The Android-linked execution was:

```
Transaction:
0xef731b0ae0449699dbbab9f138e0c968ba327dc8daa857e843ff8d342222df66

Inclusion:
Block 318, extrinsic 1
Block hash:
0x2f1f3205e8fa1ad9e9c6f8d576aff2aeab29e1bc06edbcfdf057320f3eab6397
```

{% endstep %}

{% step %}

## Verify events and balances

Verify the execution events and final balances. The main branch fails with `DelayTooShort`, then the fallback transfers exactly `50000000000000` base units. The relevant successful outcome is:

```
Balances.Transfer(amount=50000000000000, to=beneficiary)
Utility.IfElseFallbackCalled
Multisig.ProposalExecuted
System.ExtrinsicSuccess
```

Final state:

```
Multisig before:     50000000000000
Multisig after:      0

Beneficiary before:  100000000000000000
Beneficiary after:   100050000000000000
Beneficiary gain:    50000000000000

Proposal removed:    true
```

The multisig therefore loses 50 QUAN even though Bob's confirmation affirmatively described the approval as a 1 QUAN transfer.
{% endstep %}
{% endstepper %}

### Current-runtime confirmation

I repeated the chain portion with the same 100-byte payload on current chain commit `3b243b870a5442a6e5f443056f67bbba87d472f5`, runtime spec 143 and transaction version 3.

The current-runtime result was:

```
Alice proposal:
0x45bae9b1a099b7eebaafd75c427a2d0752cff54efe1b14da206b3255948fb89e

Bob exact-byte approval:
0xe6dbfaeb13293002a386856d3881a0bf2ecbee8ea4d465181684d9528a32673f

Alice execution:
0xf7ef6cc1e8e678daaa23c599a44c487293f715990e45dcb00640167bafa1fc51

Runtime: 143 / transaction version 3
Main result: ReversibleTransfers.DelayTooShort
Utility result: IfElseFallbackCalled
Transferred: 50000000000000 base units
Multisig balance: 50000000000000 to 0
Beneficiary gain: 50000000000000
Proposal removed: true
```

To execute the audited runtime without relying on a separately modified production node, this local check placed the audited spec-143 runtime Wasm into a private development genesis and ran it through the cached host node's Wasm executor. The live `:code` hash matched the audited Wasm, RPC reported spec 143 and transaction version 3, and the node had zero peers. Only stale or optional legacy development-genesis fields were removed. No production runtime source was changed.

### Negative controls

* With Hide Balances disabled, the same mobile confirmation reveals both `-1 QUAN` and `-50 QUAN`.
* The payload fully decodes and re-encodes exactly. No decoder exception or fail-open error path is involved.
* Alice remains unable to execute before Bob supplies the second approval.
* A high-security multisig rejects the exact same payload with `Multisig.CallNotAllowedForHighSecurityMultisig`, and the rejected proposal is not stored. The demonstrated target is an ordinary multisig.
* Both branches use the same beneficiary. The exploit does not depend on hiding or substituting the recipient.
* The app reads the authoritative stored bytes and submits those same bytes. No malicious RPC, indexer, fixture mutation, or backend compromise is required.

### Verified final-run evidence

The six attached PNG files provide the following evidence:

1. The same proposal with both nested amounts replaced by `-----` while the headline remains 1 QUAN.
2. The visible control where the app reveals the real 1 QUAN and 50 QUAN branch amounts.
3. The enabled `Yes, Approve` button while the 50 QUAN amount is hidden.
4. The independently decoded Bob approval containing the exact 100-byte payload.
5. The final verifier showing the 50 QUAN drain.
6. The genuine Android authentication prompt before credential entry.

PoC video: [Android multisig approval and verified 50 QUAN drain](https://drive.google.com/file/d/1F2EX5f7Nrm05gTJttL-LUUX6QBLZComC/view?usp=sharing)

The walkthrough shows the visible-amount control, the misleading privacy-enabled confirmation, the enabled approval action, genuine Android authentication, Bob's exact-byte approval, Alice's execution, and the final balance change.

Walkthrough SHA-256:

```
b4dc30054adc798afff9198bbbc2eea8fdaa0467b514f57acbcf5015f42d8a54
```

| Attachment                                                                                      | SHA-256                                                            |
| ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| [PoC video](https://drive.google.com/file/d/1F2EX5f7Nrm05gTJttL-LUUX6QBLZComC/view?usp=sharing) | `b4dc30054adc798afff9198bbbc2eea8fdaa0467b514f57acbcf5015f42d8a54` |
| `03-hidden-one-vs-fifty.png`                                                                    | `604bbecff8d6a7e300cbee0dd8c3bb3a6a01b6ebb8371898357e44876c2a04a4` |
| `02-visible-confirmation-control.png`                                                           | `f6bd09273e4f58b52a5ac273ad165f59929a9c85500381522e6cbf908d324482` |
| `04-hidden-approve-enabled.png`                                                                 | `9cf0684c5c5310a69eb1c734791bd333d44a9e973c5e126d465a0e4dbb53b5c5` |
| `06-approval-attestation.png`                                                                   | `d940178fc8b9377f6ac03b6d8b40beb6d025908f732a81397734c049fc842718` |
| `07-final-drain-verification.png`                                                               | `2b4cf35cd8c85c92f629e2f1eb37388a11727f4a30f092c86dd85795eba86bba` |
| `05-real-android-auth-prompt-before-input.png`                                                  | `de41eb29e793e27101d022f6740a022797e41fe8cffb62d93b58a8db03553d85` |
