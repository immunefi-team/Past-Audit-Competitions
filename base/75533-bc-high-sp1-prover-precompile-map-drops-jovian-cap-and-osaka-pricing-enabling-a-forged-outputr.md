# 75533 bc high sp1 prover precompile map drops jovian cap and osaka pricing enabling a forged outputroot and 10 bridge drain wherever the zk fast path is active

**Submitted on Apr 29th 2026 at 17:01:39 UTC by @Oxodus for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75533
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Unintended permanent chain split requiring hard fork (network partition requiring hard fork)
  * Direct loss to Base or users ≥ 10% of funds held within Bridge

## Description

## Brief/Intro

The SP1 ZK prover's per-spec precompile registry in `base/crates/succinct/utils/client/src/precompiles/mod.rs` calls `precompiles.extend(get_precompiles())` after cloning the spec-correct `BasePrecompiles` set. Because revm's `Precompiles::extend` overwrites entries by address, this silently replaces the Jovian-capped `bn254-pair` (input cap 81 984 B) with vanilla `bn254::pair::ISTANBUL` (uncapped) and the Osaka-priced `secp256r1::P256VERIFY_OSAKA` (\~6 900 gas) with vanilla `secp256r1::P256VERIFY` (3 450 gas). A second bug mismaps `OpSpecId::JOVIAN` to `BasePrecompiles::isthmus()` instead of `jovian()`. The live execution client and the FPVM (TEE) prover both apply the correct caps and pricing, so the SP1 prover proves a state-transition that no honest Base node ever derives. The divergent `outputRoot` finalizes through `AggregateVerifier`, propagates to `AnchorStateRegistry`, and authorizes withdrawals via `OptimismPortal2` against state the attacker chose.

## Vulnerability Details

The bug is the interaction of two lines in `base/crates/succinct/utils/client/src/precompiles/mod.rs`. First, the per-spec base set at lines 90–98 mismaps `JOVIAN` to the Isthmus precompile family:

```rust
let base = match spec {
    ...
    OpSpecId::ISTHMUS | OpSpecId::JOVIAN => BasePrecompiles::isthmus().clone(),
    OpSpecId::BASE_V1 => BasePrecompiles::base_v1().clone(),
};
```

Second, immediately after, line 100 unconditionally overwrites the spec-correct entries with Istanbul-era / pre-Osaka constants:

```rust
let mut precompiles = base;
precompiles.extend(get_precompiles());   // overwrites by address
```

`get_precompiles()` (lines 65–75) returns:

```rust
vec![
    bn254::add::ISTANBUL,
    bn254::mul::ISTANBUL,
    bn254::pair::ISTANBUL,        // uncapped
    secp256k1::ECRECOVER,
    secp256r1::P256VERIFY,         // 3 450 gas, pre-Osaka
    kzg_point_evaluation::POINT_EVALUATION,
]
```

Revm's `Precompiles::extend` is a `HashMap::insert` per entry, overwriting by address. The same overwrite primitive is used inside `BasePrecompiles::base_v1()` itself to upgrade pricing (`extend([modexp::OSAKA, secp256r1::P256VERIFY_OSAKA])`), so its semantics are not in doubt; the SP1 path immediately tramples that upgrade.

Net effect, per address: at `0x08` the live EL and FPVM run `bn254_pair::JOVIAN` with the 81 984-byte cap, while the SP1 prover runs vanilla `bn254::pair::ISTANBUL` with no cap. At `0x100` the live EL and FPVM run `P256VERIFY_OSAKA` at \~6 900 gas, while the SP1 prover runs vanilla `P256VERIFY` at 3 450 gas. The same divergence applies on `JOVIAN`: the base set is wrongly Isthmus (`bn254-pair` cap 112 687 + Isthmus BLS variants instead of Jovian variants), then `extend` overwrites bn254-pair to entirely uncapped.

The SP1 range program ([base/crates/succinct/programs/range/ethereum/src/main.rs](broken://pages/37c04a27ab6b8d15a04dce9b8bd59c9612a52ffc#L34)) and aggregation program use `ZkvmOpEvmFactory` ([base/crates/succinct/utils/client/src/witness/executor.rs](broken://pages/37b4b6f3418ed5f30d3dcfbe8a117fc8c54a6d68#L148)), which wires `OpZkvmPrecompiles` directly into the proof EVM. `OpZkvmPrecompiles::run` ([mod.rs:152-225](broken://pages/6f416de54eb33a53a8e77cd9ce85df8c4fa302f6#L152-L225)) executes whatever precompile is in the map (the `precompile.execute(input_bytes, inputs.gas_limit)` call) — there is no separate cap or pricing guard.

The trigger is a single L2 transaction. An attacker deploys a contract whose function `staticcall`s `0x08` with `len > 81_984` bytes (multiple of 192, syntactically-valid pairing input):

* Live EL: `bn254_pair::JOVIAN` returns `Bn254PairLength` → `STATICCALL` returns 0
* SP1 prover: `bn254::pair::ISTANBUL` runs the pairing → `STATICCALL` returns 32-byte success

The contract `SSTORE`s the result, the live and proven post-state roots diverge, and the SP1 prover honestly proves its own divergent execution. On-chain `AggregateVerifier._verifyZkProof` accepts the proof against `ZK_RANGE_HASH` (= the program's vkey). The game finalizes against the prover's `outputRoot`; `closeGame` → `AnchorStateRegistry.setAnchorState` adopts the forged root; `OptimismPortal2.proveWithdrawalTransaction` accepts withdrawal Merkle proofs the attacker pre-planted in the divergent block.

The author's intent contradicts the implementation: the FPVM (TEE) provider in [base/crates/proof/fpvm-precompiles/src/precompiles/provider.rs](broken://pages/cbb9aa0142516bbdf3c0d5ea05ec84d0c96de529#L45-L67) correctly distinguishes `JOVIAN => BasePrecompiles::jovian()` and uses accelerated wrappers (e.g., `fpvm_bn128_pair_jovian`) that re-enforce the cap. The SP1 path was clearly meant to mirror that pattern.

## Impact Details

* Direct bridge loss ≥ 10%: On any network where multiproof + ZK finalization is active, a single ZK proof under `verifyProposalProof` finalizes a forged `outputRoot`. The forged anchor authorizes any `proveWithdrawalTransaction` call whose Merkle proof matches the divergent state. The attacker plants `L2ToL1MessagePasser`-style storage in their controlled L2 contract during the divergent block and withdraws bridge funds without bound. This trivially exceeds 10% of bridge TVL since there is no rate or amount cap on a successful withdrawal proof.
* Permanent chain split requiring hard fork: Once a forged anchor is adopted on L1, honest Base nodes cannot derive the canonical chain past that anchor without ignoring on-chain finalization. Recovery requires either deploying a fixed prover binary, rotating `ZK_RANGE_HASH` to the new vkey, and manually rolling back in-flight forged anchors and any withdrawals they authorized — or a coordinated hardfork.
* The issue has two independent triggers; (i) `bn254-pair` cap divergence on `JOVIAN` / `BASE_V1`: any L2 transaction that `staticcall`s `0x08` with input > 81 984 bytes diverges between live EL and SP1 prover. (ii) P256 gas-cost divergence on `BASE_V1`: 3 450 vs 6 900 gas, so every P256-using contract becomes a potential trigger — any branch on `gasleft()` after `STATICCALL(0x100, …)` flips between live EL and prover.

## References

* `base/crates/succinct/utils/client/src/precompiles/mod.rs`
  * `get_precompiles()` returning vanilla Istanbul / pre-Osaka constants — lines 65–75
  * `JOVIAN` mismapped to `BasePrecompiles::isthmus()` — line 96
  * `precompiles.extend(get_precompiles())` overwrite — line 100
  * `OpZkvmPrecompiles::run` direct execution (no cap guard) — lines 152–225 (the `precompile.execute(…)` call sits inside this body with no surrounding cap/pricing logic)
* `base/crates/common/evm/src/precompiles.rs` — live EL `BasePrecompiles::jovian()` / `base_v1()` (correct caps and Osaka pricing) — lines 91–131 and `bn254_pair::JOVIAN` definition lines 197–211
* `base/crates/proof/fpvm-precompiles/src/precompiles/provider.rs` — FPVM correct mirror with per-spec accelerated wrappers — lines 45–67
* `base/crates/proof/fpvm-precompiles/src/precompiles/bn128_pair.rs` — `fpvm_bn128_pair_jovian` re-enforces `BN256_MAX_PAIRING_SIZE_JOVIAN = 81_984` — lines 17, 73–86
* `base/crates/succinct/utils/client/src/witness/executor.rs` — `ZkvmOpEvmFactory::new()` wired into `BaseExecutor` — line 148
* `base/crates/succinct/programs/range/ethereum/src/main.rs` — range program entrypoint — line 34
* revm `Precompiles::extend` overwrite-by-address semantics (used as the standard upgrade primitive inside `BasePrecompiles::base_v1()` itself).

## Proof of Concept

Append the following block to the existing `mod tests { … }` in `base/crates/succinct/utils/client/src/precompiles/mod.rs`:

```rust
    // ===== SP1 vs live-EL precompile-map divergence =====
    //
    // These tests demonstrate that `OpZkvmPrecompiles` (used by the SP1 range and
    // aggregation programs via `ZkvmOpEvmFactory`) exposes precompiles at addresses
    // 0x08 (bn254-pair) and 0x100 (P256VERIFY) that diverge in *behavior* and
    // *gas pricing* from the live execution-client `BasePrecompiles` for the same
    // `OpSpecId`. Any L2 transaction that triggers either divergence proves a
    // post-state in the SP1 prover that no honest Base node ever derives.

    /// JOVIAN: live EL enforces `JOVIAN_MAX_INPUT_SIZE = 81_984` on bn254-pair;
    /// the SP1 prover does not (vanilla `bn254::pair::ISTANBUL` is uncapped).
    #[test]
    fn sp1_vs_live_bn254_pair_jovian_cap_diverges() {
        let addr_pair = revm::precompile::u64_to_address(8);

        // 90_000 bytes > 81_984 cap, multiple of 192 (the per-pair stride),
        // so it is a syntactically-valid pairing input; only the cap rejects it.
        // 90_000 / 192 = 468.75 -> round down to 468 pairs = 89_856 bytes.
        let input = vec![0u8; 89_856];

        // --- Live EL on JOVIAN ---
        let live = BasePrecompiles::jovian();
        let live_pair = live.get(&addr_pair).expect("live JOVIAN must define 0x08");
        let live_res = live_pair.execute(&input, u64::MAX);
        assert!(
            matches!(live_res, Err(PrecompileError::Bn254PairLength)),
            "live EL on JOVIAN must reject oversized bn254-pair input via the cap, got: {:?}",
            live_res
        );

        // --- SP1 ZK prover on JOVIAN ---
        let zk = OpZkvmPrecompiles::new_with_spec(OpSpecId::JOVIAN);
        let zk_pair = zk
            .inner
            .precompiles
            .get(&addr_pair)
            .expect("SP1 JOVIAN must define 0x08");
        let zk_res = zk_pair.execute(&input, u64::MAX);
        assert!(
            !matches!(zk_res, Err(PrecompileError::Bn254PairLength)),
            "SP1 prover on JOVIAN must NOT enforce the cap (this is the bug); got: {:?}",
            zk_res
        );
    }

    /// BASE_V1: live EL applies Osaka pricing for P256VERIFY (~6 900 gas);
    /// the SP1 prover applies the pre-Osaka constant (3 450 gas).
    #[test]
    fn sp1_vs_live_p256verify_osaka_pricing_diverges() {
        let addr_p256 = revm::precompile::u64_to_address(0x100);

        // P256VERIFY rejects malformed input length but still charges the base cost,
        // which is what we are comparing.
        let input = vec![0u8; 160];

        let live = BasePrecompiles::base_v1();
        let live_p256 = live.get(&addr_p256).expect("live BASE_V1 must define 0x100");
        let live_gas = live_p256
            .execute(&input, u64::MAX)
            .expect("live P256VERIFY must charge gas, even on invalid sig")
            .gas_used;

        let zk = OpZkvmPrecompiles::new_with_spec(OpSpecId::BASE_V1);
        let zk_p256 = zk
            .inner
            .precompiles
            .get(&addr_p256)
            .expect("SP1 BASE_V1 must define 0x100");
        let zk_gas = zk_p256
            .execute(&input, u64::MAX)
            .expect("SP1 P256VERIFY must charge gas, even on invalid sig")
            .gas_used;

        assert!(
            live_gas > zk_gas,
            "live EL must charge MORE for P256VERIFY than the SP1 prover under BASE_V1 \
             (Osaka pricing vs pre-Osaka). live={}, zk={}",
            live_gas,
            zk_gas,
        );
        // Concrete sanity check: Osaka is 6_900, pre-Osaka is 3_450.
        assert_eq!(zk_gas, 3_450, "SP1 prover should still be on pre-Osaka pricing");
        assert_eq!(live_gas, 6_900, "live EL on BASE_V1 should be on Osaka pricing");
    }

    /// JOVIAN base set is mismapped to `BasePrecompiles::isthmus()`. Even ignoring
    /// the `extend(get_precompiles())` overwrite, the underlying base set picked
    /// for JOVIAN in the SP1 prover is NOT the one the live EL uses for JOVIAN.
    /// We pin this by comparing the bn254-pair `id`-pointer at construction time:
    /// the live EL uses `bn254_pair::JOVIAN`; the SP1 base set (before extend)
    /// would use `bn254_pair::GRANITE` (Isthmus inherits Granite at 0x08).
    /// After extend it is overwritten to vanilla ISTANBUL — so the SP1 map's
    /// 0x08 function pointer must NOT equal the live EL's JOVIAN function
    /// pointer.
    #[test]
    fn sp1_jovian_to_isthmus_mismap_visible() {
        let addr_pair = revm::precompile::u64_to_address(8);

        let live_jovian = BasePrecompiles::jovian()
            .get(&addr_pair)
            .expect("live JOVIAN 0x08");
        let live_isthmus = BasePrecompiles::isthmus()
            .get(&addr_pair)
            .expect("live ISTHMUS 0x08");

        // Cross-check the codebase invariant we rely on: live JOVIAN's bn254-pair
        // function differs from live ISTHMUS's bn254-pair function (the Jovian cap
        // is 81_984, the Granite/Isthmus cap is 112_687).
        let probe = vec![0u8; 100_032]; // 100_032 = 521 * 192, between Jovian and Granite caps.
        let live_jovian_res = live_jovian.execute(&probe, u64::MAX);
        let live_isthmus_res = live_isthmus.execute(&probe, u64::MAX);
        assert!(
            matches!(live_jovian_res, Err(PrecompileError::Bn254PairLength)),
            "JOVIAN must reject 100_032-byte input"
        );
        assert!(
            !matches!(live_isthmus_res, Err(PrecompileError::Bn254PairLength)),
            "ISTHMUS must accept 100_032-byte input (cap 112_687)"
        );

        // The SP1 prover under JOVIAN must NOT match live JOVIAN behavior
        // (it would either match ISTHMUS — pre-overwrite — or be uncapped — post-overwrite).
        let zk = OpZkvmPrecompiles::new_with_spec(OpSpecId::JOVIAN);
        let zk_pair = zk
            .inner
            .precompiles
            .get(&addr_pair)
            .expect("SP1 JOVIAN 0x08");
        let zk_res = zk_pair.execute(&probe, u64::MAX);
        assert!(
            !matches!(zk_res, Err(PrecompileError::Bn254PairLength)),
            "SP1 JOVIAN must NOT enforce the Jovian cap on a 100_032-byte input \
             (either due to JOVIAN→isthmus mismap or the extend overwrite)"
        );
    }
```

Run with:

```bash
cargo test -p base-succinct-client-utils --lib -- sp1_vs_live sp1_jovian --nocapture
```

Output:

```
running 3 tests
test precompiles::tests::sp1_vs_live_p256verify_osaka_pricing_diverges ... ok
test precompiles::tests::sp1_vs_live_bn254_pair_jovian_cap_diverges ... ok
test precompiles::tests::sp1_jovian_to_isthmus_mismap_visible ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 15 filtered out; finished in 0.01s
```
