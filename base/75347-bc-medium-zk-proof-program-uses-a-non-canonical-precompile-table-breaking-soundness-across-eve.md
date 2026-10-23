# 75347 bc medium zk proof program uses a non canonical precompile table breaking soundness across every base hardfork and enabling permanent chain split

**Submitted on Apr 28th 2026 at 17:16:24 UTC by @oxeix for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75347
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Unintended permanent chain split requiring hard fork (network partition requiring hard fork)

## Description

### Brief/Intro

The SP1/ZKVM proof program used by Base's ZK proving service runs EVM execution against `OpZkvmPrecompiles`, a precompile provider whose table diverges from the canonical Base precompile table (BasePrecompiles) on every hardfork from Fjord onward, and additionally activates pre-Fjord precompiles that do not exist on chain. Because this provider is wired into `ZkvmOpEvmFactory`, which is the executor backend of the production SP1 program, an attacker can craft a transaction whose execution outcome (success/revert, gas used, return data) differs between an honest Base node and the generated ZK proof. Once submitted on L1, the unsound ZK proof finalizes a state that no honest L2 node can reproduce - producing a permanent chain split, freezing canonical withdrawals, and enabling direct theft from the L1 bridge by any party who can submit a proof for the SP1 game type.

### Vulnerability Details

<https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/precompiles/mod.rs#L66-75>

```rust
fn get_precompiles() -> Vec<PrecompileWithAddress> {
    vec![
        bn254::add::ISTANBUL,
        bn254::mul::ISTANBUL,
        bn254::pair::ISTANBUL,        // pre-Granite, no input-size cap
        secp256k1::ECRECOVER,
        secp256r1::P256VERIFY,        // pre-Osaka pricing, RIP-7212 only
        kzg_point_evaluation::POINT_EVALUATION,
    ]
```

<https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/precompiles/mod.rs#L100>

```rust
// inside new_with_spec(spec):
let base = match spec { /* spec-correct table from BasePrecompiles */ };
let mut precompiles = base;
precompiles.extend(get_precompiles());        // <- unconditional override
```

The `extend` call is keyed by precompile address, so any precompile in `get_precompiles()` whose address collides with one in base overrides the spec-correct variant.

Concrete divergences:

1.

AZUL

Precompile: `0x100 P256VERIFY` Canonical: secp256r1::P256VERIFY\_OSAKA (EIP-7951, base gas 6 900) ZK (`OpZkEVMPrecompiles`) `secp256r1::P256VERIFY` (RIP-7212, base gas 3 450) Observable difference: At 5 000 gas: canonical OutOfGas, ZK Ok(gas\_used=3450)

2

GRANITE, HOLOCENE

Precompile: `0x08 bn254 pair` Canonical: bn254\_pair::GRANITE (rejects input > 112 687 B with Bn254PairLength) ZK: bn254::pair::ISTANBUL (no cap) Observable difference: At 112 704 B input: canonical Bn254PairLength, ZK Ok(0x…01)

3

JOVIAN, AZUL

Precompile: 0x08 bn254 pair (and bls12\_381 G1/G2/Pairing) Canonical: bn254\_pair::JOVIAN (rejects > 81 984 B) plus bls12\_381 Jovian caps ZK: Base built from Self::isthmus() (line 102 in mod.rs), then overridden with Istanbul pair Observable difference: At 82 176 B input: canonical Bn254PairLength, ZK Ok(0x…01)

4

BEDROCK, REGOLITH, CANYON, ECOTONE

Precompile: 0x100 P256VERIFY Canonical: precompile not present (RIP-7212 not yet active) ZK: precompile present Observable difference: A CALL to 0x100: canonical = empty-EOA call, ZK = P256 verifier

The ZK proof's contract with L1 is: "this output state is the result of applying canonical Base STF to the input state and the supplied transactions." The dispute game's resolution function on L1 trusts SP1 verifier output for state finalization. If the ZK program's EVM disagrees with canonical Base on any precompile observable, that contract is broken: a proposer can produce a valid SP1 proof for an output state that no honest Base node could ever reach, and the L1 game will accept it.

## Impact Details

This is a soundness break in the proof program resulting in an unintended permanent chain split requiring hard fork (Critical). Every Base fullnode runs canonical `BasePrecompiles`. The L1-finalized state from the ZK game runs `OpZkvmPrecompiles`. The first time a transaction touches any of the four divergent surfaces above (e.g., a pre-Osaka-priced P256 call cheap enough to land on L2 but too expensive on canonical Azul), the L1 truth and the L2 truth diverge. Recovery requires re-rolling the SP1 program and the L1 game's verifier key - a hard-fork-equivalent migration. Both branches' history must then be reconciled by social fork.

## References

<https://github.com/base/base/blob/v0.8.0-rc.28/crates/succinct/utils/client/src/precompiles/mod.rs>

## Proof of Concept

{% stepper %}
{% step %}

## PoC 1 — AZUL: P256VERIFY gas pricing rolled back from Osaka (EIP-7951) to pre-Osaka.

Canonical AZUL installs `secp256r1::P256VERIFY_OSAKA` at 0x100 with base gas 6900.\
At 5\_000 gas the canonical chain MUST revert OutOfGas (mirrors `provider.rs::test_get_azul_precompile_with_osaka_rules`).\
ZK AZUL extends `secp256r1::P256VERIFY` (regular, base gas 3450) at the same address, overriding the Osaka variant. The same call succeeds inside the proof.

```rust
#[test]
fn poc_azul_p256_gas_pricing_diverges() {
    let canonical = BasePrecompiles::new_with_spec(OpSpecId::AZUL);
    let zk = OpZkvmPrecompiles::new_with_spec(OpSpecId::AZUL);

    let p256_addr: Address = *secp256r1::P256VERIFY.address();
    let canonical_p = canonical
        .precompiles()
        .get(&p256_addr)
        .expect("canonical Azul must have P256 at 0x100");
    let zk_p =
        zk.inner.precompiles.get(&p256_addr).expect("ZK Azul must have P256 at 0x100");

    // Below Osaka base (6900), above non-Osaka base (3450).
    let gas = 5_000u64;
    let canonical_result = canonical_p.execute(&[], gas);
    let zk_result = zk_p.execute(&[], gas);

    assert!(
        matches!(canonical_result, Err(PrecompileError::OutOfGas)),
        "canonical Azul P256 at {gas} gas must be OutOfGas (Osaka pricing); got {canonical_result:?}"
    );

    let zk_out = zk_result.expect("ZK Azul P256 must succeed at 5000 gas — that IS the bug");
    assert_eq!(
        zk_out.gas_used,
        secp256r1::P256VERIFY_BASE_GAS_FEE,
        "ZK Azul charges pre-Osaka base gas ({}), not Osaka base gas ({})",
        secp256r1::P256VERIFY_BASE_GAS_FEE,
        secp256r1::P256VERIFY_BASE_GAS_FEE_OSAKA,
    );

    println!(
        "[PoC 1] AZUL P256VERIFY @ gas={gas}: canonical=OutOfGas, ZK=Ok(gas_used={}). \
         A tx that reverts on chain proves successfully in SP1.",
        zk_out.gas_used
    );
}
```

{% endstep %}

{% step %}

## PoC 2 — GRANITE: bn254 pairing input-size cap silently removed.

Canonical GRANITE installs `bn254_pair::GRANITE` which rejects any input larger than `GRANITE_MAX_INPUT_SIZE = 112_687` bytes with `Bn254PairLength`.\
ZK GRANITE extends `bn254::pair::ISTANBUL` at the same address, which has no cap. An attacker can craft a pairing call that the canonical chain rejects but the ZK proof attests succeeded.

```rust
#[test]
fn poc_granite_bn254_pair_input_cap_bypassed() {
    use base_common_evm::GRANITE_MAX_INPUT_SIZE;

    let canonical = BasePrecompiles::new_with_spec(OpSpecId::GRANITE);
    let zk = OpZkvmPrecompiles::new_with_spec(OpSpecId::GRANITE);

    let pair_addr = &bn254::pair::ADDRESS;
    let canonical_p = canonical.precompiles().get(pair_addr).unwrap();
    let zk_p = zk.inner.precompiles.get(pair_addr).unwrap();

    // Pick an input that is (a) > GRANITE_MAX_INPUT_SIZE and (b) a multiple of 192.
    // That isolates the Granite cap from the underlying Istanbul "% 192" check
    // (which would also return Bn254PairLength and confuse the diagnosis).
    const PAIR_ELEM: usize = 192;
    let len = ((GRANITE_MAX_INPUT_SIZE / PAIR_ELEM) + 1) * PAIR_ELEM;
    assert!(len > GRANITE_MAX_INPUT_SIZE && len % PAIR_ELEM == 0);
    let oversized = vec![0u8; len];

    let canonical_result = canonical_p.execute(&oversized, u64::MAX);
    let zk_result = zk_p.execute(&oversized, u64::MAX);

    assert!(
        matches!(canonical_result, Err(PrecompileError::Bn254PairLength)),
        "canonical Granite must reject input > {GRANITE_MAX_INPUT_SIZE} with Bn254PairLength; got {canonical_result:?}"
    );
    assert!(
        !matches!(zk_result, Err(PrecompileError::Bn254PairLength)),
        "ZK Granite must NOT enforce the Granite length cap; got {zk_result:?} (fix already shipped?)"
    );

    println!(
        "[PoC 2] GRANITE bn254_pair @ input_len={} (multiple of 192, > Granite cap): \
         canonical=Bn254PairLength, ZK={:?}. Cap is bypassed inside the proof.",
        oversized.len(),
        zk_result
    );
}
```

{% endstep %}

{% step %}

## PoC 3 — JOVIAN: spec base built from `isthmus()` instead of `jovian()`, so Jovian's tighter input-size caps are entirely missing from the proof.

We pick an input that is ABOVE the Jovian cap (81\_984) but BELOW the Granite cap (112\_687). Canonical Jovian rejects with `Bn254PairLength`; ZK Jovian uses an Isthmus-shaped base table that is then overridden with bn254-pair Istanbul, so it does not enforce any of Jovian's tighter caps.

```rust
#[test]
fn poc_jovian_uses_wrong_base_table() {
    use base_common_evm::JOVIAN_MAX_INPUT_SIZE;

    let canonical = BasePrecompiles::new_with_spec(OpSpecId::JOVIAN);
    let zk = OpZkvmPrecompiles::new_with_spec(OpSpecId::JOVIAN);

    let pair_addr = &bn254::pair::ADDRESS;
    let canonical_p = canonical.precompiles().get(pair_addr).unwrap();
    let zk_p = zk.inner.precompiles.get(pair_addr).unwrap();

    // Pick a length > Jovian cap, < Granite cap, and a multiple of 192.
    const PAIR_ELEM: usize = 192;
    let len = ((JOVIAN_MAX_INPUT_SIZE / PAIR_ELEM) + 1) * PAIR_ELEM;
    assert!(len > JOVIAN_MAX_INPUT_SIZE && len % PAIR_ELEM == 0);
    assert!(len < base_common_evm::GRANITE_MAX_INPUT_SIZE);
    let oversized = vec![0u8; len];

    let canonical_result = canonical_p.execute(&oversized, u64::MAX);
    let zk_result = zk_p.execute(&oversized, u64::MAX);

    assert!(
        matches!(canonical_result, Err(PrecompileError::Bn254PairLength)),
        "canonical Jovian must reject input > {JOVIAN_MAX_INPUT_SIZE} with Bn254PairLength; got {canonical_result:?}"
    );
    assert!(
        !matches!(zk_result, Err(PrecompileError::Bn254PairLength)),
        "ZK Jovian must NOT enforce the Jovian length cap; got {zk_result:?} (fix already shipped?)"
    );

    println!(
        "[PoC 3] JOVIAN bn254_pair @ input_len={} (Jovian-illegal, Granite-legal, %192=0): \
         canonical=Bn254PairLength, ZK={:?}. Wrong base table proven.",
        oversized.len(),
        zk_result
    );
}
```

{% endstep %}

{% step %}

## PoC 4 — Pre-Fjord: P256VERIFY (RIP-7212) installed by the ZK provider for hardforks in which it does not yet exist on the canonical chain.

On canonical BEDROCK/REGOLITH/CANYON/ECOTONE, address 0x100 is empty: a CALL to it succeeds returning empty data (it's just a regular EOA). Inside the SP1 program, however, the ZK precompile table answers with the P256 verifier and charges precompile gas, producing a different observable result.

```rust
#[test]
fn poc_pre_fjord_p256_should_not_exist() {
    let p256_addr: Address = *secp256r1::P256VERIFY.address();

    for spec in [
        OpSpecId::BEDROCK,
        OpSpecId::REGOLITH,
        OpSpecId::CANYON,
        OpSpecId::ECOTONE,
    ] {
        let canonical = BasePrecompiles::new_with_spec(spec);
        let zk = OpZkvmPrecompiles::new_with_spec(spec);

        let canonical_has = canonical.precompiles().get(&p256_addr).is_some();
        let zk_has = zk.inner.precompiles.get(&p256_addr).is_some();

        assert!(!canonical_has, "canonical {spec:?} must NOT have P256 at 0x100");
        assert!(zk_has, "ZK {spec:?} unexpectedly does NOT have P256 — that IS the bug");

        println!(
            "[PoC 4] {spec:?}: canonical_has_p256={canonical_has}, zk_has_p256={zk_has}. \
             RIP-7212 active too early in the proof."
        );
    }
}
```

{% endstep %}
{% endstepper %}

## Logs

```rust
running 4 tests
[PoC 1] AZUL P256VERIFY @ gas=5000: canonical=OutOfGas, ZK=Ok(gas_used=3450).
[PoC 2] GRANITE bn254_pair @ input_len=112704: canonical=Bn254PairLength,
        ZK=Ok(bytes: 0x…01)   ← proof attests pairing SUCCESS
[PoC 3] JOVIAN bn254_pair @ input_len=82176: canonical=Bn254PairLength,
        ZK=Ok(bytes: 0x…01)   ← proof attests pairing SUCCESS
[PoC 4] BEDROCK / REGOLITH / CANYON / ECOTONE: canonical_has_p256=false, zk_has_p256=true.

test result: ok. 4 passed; 0 failed
```
