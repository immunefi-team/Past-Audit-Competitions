# 74978 bc medium a bug in the respective layer 0 1 2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

**Submitted on Apr 26th 2026 at 13:19:23 UTC by @coinsspor for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74978
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
  * Modification of transaction fees outside of design parameters

## Description

## Brief/Intro

While reviewing Base Azul's consensus engine for the audit competition, I traced through the code path that reconciles L1-derived payload attributes against an unsafe L2 block before that block can be consolidated as the safe head. In `crates/consensus/engine/src/attributes.rs` at line 214, the function `AttributesMatch::check_eip1559()` decodes the Jovian `extra_data` from the block header and immediately throws away one of the three fields it returns — the `min_base_fee`. From that point forward, no comparison touches that field, so an unsafe block can carry any `min_base_fee` value (zero, `u64::MAX`, anything in between) and still pass the consolidation check, while the next block's base fee floor will follow whatever value that header encodes.

If this is exploited on a live deployment, the chain's minimum base fee diverges silently from the value derived by L1 SystemConfig. The next-block fee calculation reads the parent header's `extra_data` directly and clamps to the embedded `min_base_fee`, so the floor that EIP-1559 enforces — and that the EVM `BASEFEE` opcode exposes to every contract — becomes whatever the producer of the unsafe block decided to write into the header rather than what L1 actually said.

## Vulnerability Details

The bug sits in a single line. Here is the relevant block from `crates/consensus/engine/src/attributes.rs` (around line 213-219):

```rust
let extra_data_decoded = if config.is_jovian_active(block.header.timestamp) {
    JovianExtraData::decode(&block.header.extra_data).map(|(be, bd, _)| (be, bd))
} else if config.is_holocene_active(block.header.timestamp) {
    HoloceneExtraData::decode(&block.header.extra_data)
} else {
    return AttributesMismatch::MissingBlockEIP1559.into();
};
```

The pattern `|(be, bd, _)|` is the entire bug. To understand why, I went and looked at what `JovianExtraData::decode` actually returns. From `crates/common/consensus/src/extra/jovian.rs`:

```rust
/// Returns (`elasticity`, `denominator`, `min_base_fee`).
pub fn decode(extra_data: &[u8]) -> Result<(u32, u32, u64), EIP1559ParamError> { ... }
```

So the third tuple element being discarded is specifically `min_base_fee`. After this `map`, the rest of `check_eip1559` only compares `(be, bd)` against `(ae, ad)` derived from the attributes — elasticity and denominator. The attribute side has the value available too: `attributes.attributes().min_base_fee` is an `Option<u64>` field defined in `crates/common/rpc-types-engine/src/attributes.rs`. The payload builder enforces it being `Some(_)` post-Jovian and uses it when encoding the block's own `extra_data`. So both sides have the value — but the comparison closure throws one of them away before any equality check runs.

I read the rest of the function carefully to make sure I wasn't missing a comparison further down. There isn't one. The function falls through to `Self::Match` if elasticity and denominator agree.

I also wanted to confirm this was Base-specific code rather than something inherited from upstream. The Jovian branch with `JovianExtraData` and the `|(be, bd, _)|` pattern is in `base-consensus` only; `op-node` is in Go and doesn't have this exact construct. The competition scope explicitly lists "Offchain consensus/execution logic" and "edge cases in EIP-1559 fee calculation" as primary concern areas, both of which match what I found.

The next thing I traced was where this check is actually called from. In `crates/consensus/engine/src/task_queue/tasks/consolidate/task.rs` at lines 47-49:

```rust
fn is_consistent_with_block(&self, cfg: &RollupConfig, block: &Block<Transaction>) -> bool {
    match self {
        Self::Attributes(attributes) => {
            crate::AttributesMatch::check(cfg, attributes, block).is_match()
        }
        ...
    }
}
```

So the consolidation task's "is this unsafe block consistent with what we derived from L1?" decision is exactly the boolean returned by `AttributesMatch::check(...).is_match()`. I could not find any later comparison of `attributes.min_base_fee` against the block header's Jovian `min_base_fee` in the consolidation path. The immediate check used by `AttributesMatch` discards the field, and nothing downstream re-checks it before the unsafe block becomes the parent of the next block.

The downstream usage of `min_base_fee` is in `crates/execution/chainspec/src/basefee.rs`. The function `compute_jovian_base_fee` at lines 51-71 reads the parent's `extra_data` directly:

```rust
let (elasticity, denominator, min_base_fee) = JovianExtraData::decode(parent.extra_data())?;
// ... compute next_base_fee from gas usage and EIP-1559 dynamics ...
if next_base_fee < min_base_fee {
    return Ok(min_base_fee);
}
```

The `min_base_fee` from the parent header's `extra_data` becomes the floor for the next block's base fee. That value isn't re-derived from L1 at this point; it's trusted as already-validated. But the only thing that was supposed to validate it against the L1-derived value was the `AttributesMatch` check that just threw the field away.

To verify all of this empirically rather than just reading the code, I wrote five unit tests against `v0.8.0-rc.24` and ran them. They are in the PoC section. Briefly:

* The validator returns `Match` for the same block with `min_base_fee` ranging from `0` to `u64::MAX` against attributes saying anything else, including the `None vs MAX` case (which is itself a spec violation since Jovian requires `Some(_)` post-activation).
* A control test where elasticity and denominator differ does correctly produce `Mismatch(EIP1559Parameters(...))`, which rules out a generic test-environment failure and isolates the bug to the discarded field.
* The next block's base fee, computed from a parent whose `extra_data` carries a poisoned `min_base_fee = u64::MAX/2` versus an honest `5_000_000` wei, diverges by a factor of roughly 9.2 billion.

## Impact Details

The value of the next block's fee floor becomes bound to what the producer of the unsafe block writes into its header rather than what L1 SystemConfig actually said it should be, because the only reconciliation check between those two values silently drops the field.

The concrete consequences I see:

**Fee floor drift on the live chain.** Every block after the consolidated unsafe one computes its base fee against a `min_base_fee` that may diverge arbitrarily from the L1-derived value. POC-6 in the next section measures this directly — identical gas inputs, only the parent header's `min_base_fee` changes, and the result diverges by 9,223,372,036×. The fee floor at that point is whatever was encoded into the unsafe header.

**Multi-client divergence and fork risk.** The competition scope flags this category explicitly: "Multi-client discrepancies – execution differences between CL and EL that cause chain forks." A future fixed version of `base-consensus`, or any other client implementation that compares `min_base_fee`, will reject the same block that the current `base-consensus` accepts. Once that happens, the two clients' views of the canonical chain diverge.

**EVM-observable fee distortion.** The `BASEFEE` opcode reads the current block's base fee per gas. Smart contracts and wallets routinely use this for gas estimation, fee oracles, MEV calculations, and conditional logic. A poisoned floor produces a `BASEFEE` value that does not match the L1-derived design parameters, and every contract that reads `BASEFEE` sees the distorted value. This maps to "Modification of transaction fees outside of design parameters" — the design says the floor comes from L1 SystemConfig, the actual behavior is that the floor comes from the unsafe block's `extra_data`.

**Transaction inclusion impact for users.** If the floor drifts upward, transactions that priced themselves against the L1-derived floor get excluded until either the chain re-converges through normal EIP-1559 dynamics (which can take many blocks) or someone intervenes manually. If the floor drifts downward, the chain loses its spam protection until the same re-convergence happens.

I am submitting this under the impact "A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk" because the immediate effect is on `BASEFEE` semantics observed by EVM contracts and on chain fee parameters. No funds are directly at risk in the bridge or contracts. Consensus-layer fee invariants are broken, and the competition's in-scope impacts include this category.

I'm asserting **Medium severity** rather than reaching for High. The trigger sits within the block-producer's authority over header `extra_data` encoding; the impact is on fee market parameters rather than direct fund loss; and while I traced the consolidation call site by inspection (the 3-line direct call at `task.rs:47-49` into the bypassed function), I am not running an end-to-end integration test that drives the full consolidation pipeline. If a reviewer concludes that the multi-client fork risk or the prolonged transaction-exclusion impact warrants High, I would not object — but Medium is what I'm comfortable asserting from the evidence I have in the PoC.

The bug is post-Jovian only. Pre-Jovian (Holocene-only) blocks pass through a different decode path and are unaffected. Jovian is active on Base Sepolia, which is the competition environment. Mainnet is configured for Jovian but is out of scope for this competition; my impact assessment treats mainnet exposure as a planned-deployment risk only.

## References

Code locations in `https://github.com/base/base/tree/v0.8.0-rc.24`:

* `crates/consensus/engine/src/attributes.rs` line 214 — the discarding `map(|(be, bd, _)| ...)`
* `crates/common/consensus/src/extra/jovian.rs` line 26 — `JovianExtraData::decode` signature returning `(u32, u32, u64)`
* `crates/common/rpc-types-engine/src/attributes.rs` — `min_base_fee: Option<u64>` field on `PayloadAttributes`
* `crates/consensus/engine/src/task_queue/tasks/consolidate/task.rs` lines 47-49 — `is_consistent_with_block` calling `AttributesMatch::check(...).is_match()`
* `crates/execution/chainspec/src/basefee.rs` lines 51-71 — `compute_jovian_base_fee` reading `min_base_fee` from parent `extra_data` and clamping

Specifications and documentation:

* OP Stack Jovian execution engine spec: <https://specs.optimism.io/protocol/jovian/exec-engine.html>
* Jovian minimum base fee in block header: <https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/jovian/exec-engine.md#minimum-base-fee-in-block-header>
* Base mainnet documented minimum base fee (5,000,000 wei = 0.005 gwei): <https://docs.base.org/base-chain/network-information/network-fees>

## Proof of Concept

{% stepper %}
{% step %}

## Get the codebase

```bash
git clone https://github.com/base/base.git
cd base
git checkout v0.8.0-rc.24
```

I worked against `v0.8.0-rc.24` directly since that is the current scope target after the Discord update. The relevant crates are `base-consensus-engine` (v0.8.0) and `base-execution-chainspec` (v0.8.0). No special toolchain setup — stable Rust per the project's `rust-toolchain.toml` is enough.
{% endstep %}

{% step %}

## Add the tests

I added five tests in two files. Both files already have a `mod tests` block with helpers I reused (`eip1559_test_setup`, `get_chainspec`, `JOVIAN_TIMESTAMP`, `HoloceneExtraData`, `JovianExtraData`, `BaseFeeParams`, `FixedBytes`), so I only needed to append the test functions inside those existing modules — no extra `use` statements.

Append these four tests inside the `mod tests` block at the end of `crates/consensus/engine/src/attributes.rs`:

```rust
// ==========================================================================
// SECURITY POC: Jovian minBaseFee Validation Bypass
// ==========================================================================
// Bug location: attributes.rs:214
//   JovianExtraData::decode(&block.header.extra_data).map(|(be, bd, _)| (be, bd))
//                                                                  ^^^
//                                                          min_base_fee discarded
// ==========================================================================

/// POC-1: Core bug - validator returns Match despite min_base_fee mismatch.
#[test]
fn poc_1_jovian_min_base_fee_ignored() {
    let (mut cfg, mut attributes, mut block) = eip1559_test_setup();
    cfg.hardforks.jovian_time = Some(0);

    let base_fee_params = BaseFeeParams {
        max_change_denominator: 8,
        elasticity_multiplier: 8,
    };

    let eip1559_extra_params_h = HoloceneExtraData::encode(
        Default::default(),
        base_fee_params,
    )
    .unwrap();
    let eip1559_params: FixedBytes<8> =
        eip1559_extra_params_h.clone().split_off(1).as_ref().try_into().unwrap();

    attributes.attributes.eip_1559_params = Some(eip1559_params);
    attributes.attributes.min_base_fee = Some(100);

    let block_min_base_fee: u64 = 1_000_000;
    block.header.extra_data = JovianExtraData::encode(
        eip1559_params,
        base_fee_params,
        block_min_base_fee,
    )
    .unwrap();

    let check = AttributesMatch::check(&cfg, &attributes, &block);

    println!("\n[POC-1] Direct AttributesMatch::check() bypass");
    println!("  Derived attributes min_base_fee: {:?}", attributes.attributes.min_base_fee);
    println!("  Block header min_base_fee:       {}", block_min_base_fee);
    println!("  Ratio:                           {}x divergence", block_min_base_fee / 100);
    println!("  Result:                          {:?}", check);
    println!("  >>> Validator returned Match. min_base_fee never compared.");

    assert_eq!(check, AttributesMatch::Match);
}

/// POC-2: Extreme min_base_fee values still accepted - bug is unbounded.
#[test]
fn poc_2_jovian_extreme_min_base_fee_values_accepted() {
    let (mut cfg, mut attributes, mut block) = eip1559_test_setup();
    cfg.hardforks.jovian_time = Some(0);

    let base_fee_params = BaseFeeParams {
        max_change_denominator: 8,
        elasticity_multiplier: 8,
    };
    let eip1559_extra_params_h = HoloceneExtraData::encode(
        Default::default(),
        base_fee_params,
    )
    .unwrap();
    let eip1559_params: FixedBytes<8> =
        eip1559_extra_params_h.clone().split_off(1).as_ref().try_into().unwrap();

    attributes.attributes.eip_1559_params = Some(eip1559_params);

    let test_cases = vec![
        (Some(0u64),       u64::MAX,    "min=0 vs MAX"),
        (Some(1u64),       u64::MAX,    "min=1 vs MAX"),
        (Some(5_000_000),  100,         "Mainnet floor (5M wei) vs 100 wei"),
        (Some(u64::MAX),   0,           "MAX vs 0"),
        (None,             u64::MAX,    "None vs MAX"),
    ];

    println!("\n[POC-2] Extreme value bypass test");
    for (attr_mbf, block_mbf, label) in test_cases {
        attributes.attributes.min_base_fee = attr_mbf;
        block.header.extra_data = JovianExtraData::encode(
            eip1559_params,
            base_fee_params,
            block_mbf,
        )
        .unwrap();

        let check = AttributesMatch::check(&cfg, &attributes, &block);
        println!("  [{}] attr={:?}, block={}: {:?}",
            label, attr_mbf, block_mbf, check);

        assert_eq!(check, AttributesMatch::Match,
            "POC-2 FAILED at case '{}'", label);
    }
    println!("  >>> All extreme mismatches accepted as Match.");
}

/// POC-3: Control test - prove elasticity/denominator ARE checked.
/// Isolates the bug to min_base_fee specifically and rules out a generic failure.
#[test]
fn poc_3_jovian_other_params_correctly_checked() {
    let (mut cfg, mut attributes, mut block) = eip1559_test_setup();
    cfg.hardforks.jovian_time = Some(0);

    let attr_params = BaseFeeParams {
        max_change_denominator: 8,
        elasticity_multiplier: 8,
    };
    let block_params = BaseFeeParams {
        max_change_denominator: 99,
        elasticity_multiplier: 2,
    };

    let attr_extra_data = HoloceneExtraData::encode(
        Default::default(),
        attr_params,
    )
    .unwrap();
    let attr_eip_params: FixedBytes<8> =
        attr_extra_data.clone().split_off(1).as_ref().try_into().unwrap();

    attributes.attributes.eip_1559_params = Some(attr_eip_params);
    attributes.attributes.min_base_fee = Some(100);

    let block_extra_data_h = HoloceneExtraData::encode(
        Default::default(),
        block_params,
    )
    .unwrap();
    let block_eip_params: FixedBytes<8> =
        block_extra_data_h.clone().split_off(1).as_ref().try_into().unwrap();

    block.header.extra_data = JovianExtraData::encode(
        block_eip_params,
        block_params,
        100,
    )
    .unwrap();

    let check = AttributesMatch::check(&cfg, &attributes, &block);

    println!("\n[POC-3] Control test - elasticity/denominator mismatch IS detected");
    println!("  Attribute params: {:?}", attr_params);
    println!("  Block params:     {:?}", block_params);
    println!("  Result:           {:?}", check);
    println!("  >>> Confirms validation logic works for OTHER fields - bug is specific to min_base_fee");

    assert!(check.is_mismatch(),
        "POC-3 FAILED: elasticity/denominator mismatch should be detected");
}

/// POC-4: Pre-Jovian sanity check - bug only affects post-Jovian blocks.
#[test]
fn poc_4_pre_jovian_unaffected() {
    let (cfg, mut attributes, mut block) = eip1559_test_setup();

    let base_fee_params = BaseFeeParams {
        max_change_denominator: 8,
        elasticity_multiplier: 8,
    };
    let eip1559_extra_params_h = HoloceneExtraData::encode(
        Default::default(),
        base_fee_params,
    )
    .unwrap();
    let eip1559_params: FixedBytes<8> =
        eip1559_extra_params_h.clone().split_off(1).as_ref().try_into().unwrap();

    attributes.attributes.eip_1559_params = Some(eip1559_params);
    attributes.attributes.min_base_fee = None;
    block.header.extra_data = eip1559_extra_params_h;

    let check = AttributesMatch::check(&cfg, &attributes, &block);

    println!("\n[POC-4] Pre-Jovian control - validation works correctly");
    println!("  Result: {:?}", check);
    println!("  >>> Confirms scope: bug is post-Jovian only");

    assert_eq!(check, AttributesMatch::Match);
}
```

Then append this test inside the `mod tests` block at the end of `crates/execution/chainspec/src/basefee.rs`:

```rust
/// POC-6: Demonstrates that a poisoned parent.extra_data min_base_fee
/// directly drives the next block's base fee floor. This proves the
/// real impact of the AttributesMatch validation bypass: an unsafe block
/// consolidated with attacker-chosen min_base_fee will dictate the next
/// block's fee floor, regardless of what L1 SystemConfig derived.
///
/// Bug chain:
///   1. AttributesMatch::check() ignores min_base_fee mismatch (POC-1/2 prove this)
///   2. ConsolidateInput::is_consistent_with_block() calls AttributesMatch::check
///      and returns its is_match() boolean (task.rs:47-49) - inherits the bypass
///   3. Block consolidates as safe head with poisoned min_base_fee in header
///   4. compute_jovian_base_fee() reads parent's extra_data min_base_fee
///   5. Next base fee gets clamped to attacker-chosen value (this test)
#[test]
fn poc_6_poisoned_parent_min_base_fee_drives_next_base_fee() {
    let chain_spec = get_chainspec();
    let mut parent = chain_spec.genesis_header().clone();
    let timestamp = JOVIAN_TIMESTAMP;

    const GAS_LIMIT: u64 = 10_000_000_000;
    const BLOB_GAS_USED: u64 = 100_000_000;
    const GAS_USED: u64 = 1_000_000_000;

    // Realistic L1-derived value (Base mainnet floor: 0.005 gwei = 5_000_000 wei).
    const HONEST_MIN_BASE_FEE: u64 = 5_000_000;
    // Attacker-chosen value encoded in unsafe block header.
    const POISONED_MIN_BASE_FEE: u64 = u64::MAX / 2;

    parent.blob_gas_used = Some(BLOB_GAS_USED);
    parent.gas_used = GAS_USED;
    parent.gas_limit = GAS_LIMIT;

    // === Scenario A: Honest parent with L1-derived min_base_fee ===
    parent.extra_data = JovianExtraData::encode(
        [0; 8].into(),
        BaseFeeParams::base_sepolia(),
        HONEST_MIN_BASE_FEE,
    )
    .unwrap();

    let honest_next_fee = compute_jovian_base_fee(
        chain_spec.clone(),
        &parent,
        timestamp,
    )
    .unwrap();

    // === Scenario B: Poisoned parent (same gas, only min_base_fee differs) ===
    parent.extra_data = JovianExtraData::encode(
        [0; 8].into(),
        BaseFeeParams::base_sepolia(),
        POISONED_MIN_BASE_FEE,
    )
    .unwrap();

    let poisoned_next_fee = compute_jovian_base_fee(
        chain_spec,
        &parent,
        timestamp,
    )
    .unwrap();

    println!("\n[POC-6] Poisoned parent min_base_fee drives next block fee");
    println!("  ----- Inputs -----");
    println!("  parent.gas_used:        {}", GAS_USED);
    println!("  parent.gas_limit:       {}", GAS_LIMIT);
    println!("  parent.blob_gas_used:   {}", BLOB_GAS_USED);
    println!("  Honest min_base_fee:    {} wei (L1 SystemConfig-derived)", HONEST_MIN_BASE_FEE);
    println!("  Poisoned min_base_fee:  {} wei (attacker-chosen u64::MAX/2)", POISONED_MIN_BASE_FEE);
    println!("  ----- Outputs -----");
    println!("  Honest next base fee:   {} wei", honest_next_fee);
    println!("  Poisoned next base fee: {} wei", poisoned_next_fee);
    println!("  Divergence factor:      {}x", poisoned_next_fee / honest_next_fee.max(1));
    println!("  ----- Conclusion -----");
    println!("  >>> Identical gas inputs, only min_base_fee differs in parent extra_data.");
    println!("  >>> Next block base fee diverges by {}x.", poisoned_next_fee / honest_next_fee.max(1));
    println!("  >>> When AttributesMatch silently accepts a min_base_fee mismatch (POC-1/2),");
    println!("  >>> this is the downstream consequence: chain fee floor follows the unsafe");
    println!("  >>> block header's value rather than the L1-derived SystemConfig value.");

    assert_eq!(
        poisoned_next_fee, POISONED_MIN_BASE_FEE,
        "Poisoned scenario: next base fee should be clamped to attacker-chosen floor"
    );
    assert!(
        honest_next_fee < POISONED_MIN_BASE_FEE / 1_000_000,
        "Honest scenario should produce a normal market base fee"
    );
    assert!(
        poisoned_next_fee > honest_next_fee.saturating_mul(1_000_000_000),
        "Divergence between honest and poisoned outcomes must be massive (>1B x)"
    );
}
```

### Step 3 — Run the tests

```bash
# The 4 attributes-layer tests
cargo test -p base-consensus-engine poc_ -- --nocapture --test-threads=1

# The downstream impact test
cargo test -p base-execution-chainspec poc_6 -- --nocapture --test-threads=1
```

### Step 4 — Actual output

The output below is what I got running the commands above on my local checkout of `v0.8.0-rc.24`.

`base-consensus-engine`:

```
running 4 tests
test attributes::tests::poc_1_jovian_min_base_fee_ignored ...
[POC-1] Direct AttributesMatch::check() bypass
  Derived attributes min_base_fee: Some(100)
  Block header min_base_fee:       1000000
  Ratio:                           10000x divergence
  Result:                          Match
  >>> Validator returned Match. min_base_fee never compared.
ok
test attributes::tests::poc_2_jovian_extreme_min_base_fee_values_accepted ...
[POC-2] Extreme value bypass test
  [min=0 vs MAX] attr=Some(0), block=18446744073709551615: Match
  [min=1 vs MAX] attr=Some(1), block=18446744073709551615: Match
  [Mainnet floor (5M wei) vs 100 wei] attr=Some(5000000), block=100: Match
  [MAX vs 0] attr=Some(18446744073709551615), block=0: Match
  [None vs MAX] attr=None, block=18446744073709551615: Match
  >>> All extreme mismatches accepted as Match.
ok
test attributes::tests::poc_3_jovian_other_params_correctly_checked ...
[POC-3] Control test - elasticity/denominator mismatch IS detected
  Attribute params: BaseFeeParams { max_change_denominator: 8, elasticity_multiplier: 8 }
  Block params:     BaseFeeParams { max_change_denominator: 99, elasticity_multiplier: 2 }
  Result:           Mismatch(EIP1559Parameters(BaseFeeParams { max_change_denominator: 8, elasticity_multiplier: 8 }, BaseFeeParams { max_change_denominator: 99, elasticity_multiplier: 2 }))
  >>> Confirms validation logic works for OTHER fields - bug is specific to min_base_fee
ok
test attributes::tests::poc_4_pre_jovian_unaffected ...
[POC-4] Pre-Jovian control - validation works correctly
  Result: Match
  >>> Confirms scope: bug is post-Jovian only
ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 112 filtered out
```

`base-execution-chainspec`:

```
running 1 test
test basefee::tests::poc_6_poisoned_parent_min_base_fee_drives_next_base_fee ...
[POC-6] Poisoned parent min_base_fee drives next block fee
  ----- Inputs -----
  parent.gas_used:        1000000000
  parent.gas_limit:       10000000000
  parent.blob_gas_used:   100000000
  Honest min_base_fee:    5000000 wei (L1 SystemConfig-derived)
  Poisoned min_base_fee:  9223372036854775807 wei (attacker-chosen u64::MAX/2)
  ----- Outputs -----
  Honest next base fee:   1000000000 wei
  Poisoned next base fee: 9223372036854775807 wei
  Divergence factor:      9223372036x
  ----- Conclusion -----
  >>> Identical gas inputs, only min_base_fee differs in parent extra_data.
  >>> Next block base fee diverges by 9223372036x.
  >>> When AttributesMatch silently accepts a min_base_fee mismatch (POC-1/2),
  >>> this is the downstream consequence: chain fee floor follows the unsafe
  >>> block header's value rather than the L1-derived SystemConfig value.
ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 18 filtered out
```

### Step 5 — What each test proves

**POC-1** is the direct demonstration. Derived attributes carry `min_base_fee = 100`; the block header encodes `min_base_fee = 1_000_000` — a 10,000× divergence. With identical elasticity and denominator, `AttributesMatch::check()` returns `Match`. The function call at `attributes.rs:214` decodes the block's Jovian `extra_data` and discards the third tuple element via `|(be, bd, _)|`. No subsequent comparison reads it.

**POC-2** rules out any "the values happen to coincide" interpretation. Five test pairs span the full `u64` range: `0 vs MAX`, `1 vs MAX`, `MAX vs 0`, `None vs MAX`, and a realistic mainnet pair (`5_000_000` wei, the documented Base mainnet minimum) versus `100` wei. Every pair returns `Match`. The `None vs MAX` case is particularly notable because Jovian explicitly requires `min_base_fee` to be `Some(_)` post-activation, but the consolidation check accepts even that mismatch.

**POC-3** is the control. Same code path, same function call, but I made the `eip_1559_params` differ between attributes and block. The validator correctly returns `Mismatch(EIP1559Parameters(...))` with the exact `BaseFeeParams` values from both sides printed. This proves three things at once: the `check_eip1559` function is reached and executed, the Jovian decode path is reached (otherwise we'd hit `MissingBlockEIP1559`), and the comparison logic works correctly for the first two tuple elements. Combined with POC-1/2, this isolates the bug to exactly the `_` in `|(be, bd, _)|`. Not a test-environment artifact, not an edge case.

**POC-4** confirms the scope boundary. Pre-Jovian (Holocene-only) blocks pass through the alternate decode path and are unaffected.

**POC-6** proves the downstream fee-market impact. Same parent header in two scenarios with identical gas inputs (`gas_used`, `gas_limit`, `blob_gas_used` all equal), only the `min_base_fee` byte field in `extra_data` changes. `compute_jovian_base_fee()` at `basefee.rs:51-71` decodes the parent's `extra_data` directly and clamps `next_base_fee` to `min_base_fee` if the natural EIP-1559 result is lower. The honest scenario produces a normal market base fee of 1 gwei (above the 5,000,000 wei floor, so no clamping happens — the expected non-buggy behavior). The poisoned scenario clamps to `u64::MAX / 2 ≈ 9.2 × 10^18` wei. The divergence factor between the two is 9,223,372,036×.

Putting it all together, the chain a reviewer should walk:

1. POC-1, POC-2 prove `AttributesMatch::check()` returns `Match` despite mismatch.
2. POC-3 isolates the bug to the `min_base_fee` field, ruling out generic test-environment failure.
3. `ConsolidateInput::is_consistent_with_block()` at `task.rs:47-49` calls `AttributesMatch::check(...).is_match()` directly — three lines of inspection show it inherits the bypass.
4. POC-6 proves that once the mismatched parent is consolidated, the next block's base fee follows the parent's `extra_data` value, not the L1-derived attribute.

I did not write a full integration test that drives the entire consolidation pipeline end-to-end. The validation-layer bypass shown here is the actual bug, and the consolidation logic at `task.rs:47-49` is a 3-line direct call into the bypassed function. If the reviewer wants an integration test that exercises the full consolidation task, I'm happy to provide one as a follow-up.

### Step 6 — Suggested fix

The fix is roughly 10 lines. The idea is to keep both the Holocene 2-tuple and the Jovian 3-tuple decode results uniformly typed so the rest of the function can compare them cleanly, and add a `min_base_fee` equality check post-Jovian.

In `crates/consensus/engine/src/attributes.rs`, replace the discarding `map` with full destructuring:

```rust
// Decode block header extra_data, keeping min_base_fee where applicable.
let extra_data_decoded: Result<(u32, u32, Option<u64>), _> =
    if config.is_jovian_active(block.header.timestamp) {
        JovianExtraData::decode(&block.header.extra_data)
            .map(|(be, bd, mbf)| (be, bd, Some(mbf)))
    } else if config.is_holocene_active(block.header.timestamp) {
        HoloceneExtraData::decode(&block.header.extra_data)
            .map(|(be, bd)| (be, bd, None))
    } else {
        return AttributesMismatch::MissingBlockEIP1559.into();
    };

let (be, bd, block_mbf) = match extra_data_decoded {
    Ok(t) => t,
    Err(EIP1559ParamError::NoEIP1559Params) => {
        return AttributesMismatch::MissingBlockEIP1559.into();
    }
    Err(EIP1559ParamError::InvalidVersion(_)) => {
        return AttributesMismatch::InvalidExtraDataVersion.into();
    }
    Err(e) => {
        return AttributesMismatch::UnknownExtraDataDecodingError(e).into();
    }
};

if ae != be.into() || ad != bd.into() {
    return AttributesMismatch::EIP1559Parameters(
        BaseFeeParams { max_change_denominator: ad, elasticity_multiplier: ae },
        BaseFeeParams { max_change_denominator: bd.into(), elasticity_multiplier: be.into() },
    )
    .into();
}

// Post-Jovian: also reconcile min_base_fee.
if config.is_jovian_active(block.header.timestamp) {
    let attr_mbf = attributes.attributes().min_base_fee;
    if attr_mbf != block_mbf {
        return AttributesMismatch::MinBaseFee(attr_mbf, block_mbf).into();
    }
}

Self::Match
```

A new variant on the `AttributesMismatch` enum is needed:

```rust
pub enum AttributesMismatch {
    // ... existing variants ...
    /// The Jovian min_base_fee from the block header does not match the value
    /// from the derived payload attributes.
    MinBaseFee(Option<u64>, Option<u64>),
}
```

After this change, the four `attributes.rs` PoCs above flip behavior:

* POC-1, POC-2 should return `Mismatch(MinBaseFee(...))` instead of `Match`.
* POC-3, POC-4 should keep returning the same result they already do.

POC-6 in `basefee.rs` is independent of this fix because it tests downstream behavior; it would only become unreachable in production once the consolidation check actually rejects the mismatched block.
{% endstep %}
{% endstepper %}
