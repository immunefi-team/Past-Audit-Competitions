# 88938 bc high missing zk tree leaf record in the wormhole miner volume fee credit path leads to permanent freezing of funds

**Submitted on Aug 15th 2026 at 19:46:16 UTC by @v4rvl for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88938
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

`process_exit_bundle` settles the wormhole volume fee by minting the miner's share to the block author with `Unbalanced::increase_balance` (`pallets/wormhole/src/lib.rs:955-959`) and then emitting `MinerVolumeFeePaid` (`lib.rs:961-964`). It never calls `record_transfer`.

The block author is a **keyless** wormhole address: `extract_author_from_digest` unconditionally hashes the miner-supplied pre-runtime preimage into `Poseidon(Poseidon(salt || secret))` (`primitives/wormhole/src/lib.rs:143-148`), and an account can only produce a valid signature when it equals the Poseidon image of an ML-DSA public key (`primitives/dilithium-crypto/src/traits.rs:160-184, 191-201`). No ML-DSA keypair maps to that AccountId32, so the account can never sign. Its **only** spend instrument is a ZK-tree leaf redeemed through a wormhole exit.

No leaf is created for this credit by any path:

1. The wormhole pallet does not record one at the credit site. The exit credits eleven lines earlier do (`lib.rs:851-859`); the miner-fee arm does not.
2. `Unbalanced::increase_balance` is event-free by deliberate design at this pallet (`lib.rs:822-828`), so the runtime's event-scanning `WormholeProofRecorderExtension`, which matches only `Balances::Transfer`, `Balances::Minted`, `Balances::TransferOnHold` and `Balances::ReserveRepatriated` (`runtime/src/transaction_extensions.rs:87-88, 339-380`), has nothing to match.
3. The extension's scan cannot run for this call anyway: `verify_private_batch` is unsigned (`pallets/wormhole/src/lib.rs:1002-1019`), and the extension does not implement `bare_post_dispatch`, whose upstream default is a no-op (`sp-runtime-45.0.0/src/traits/transaction_extension/mod.rs:438-445`).
4. The pallet's only other leaf-creating hook runs at block 1 and only for genesis balances (`lib.rs:468-472`), so it can never reach a later credit.

The result is that every miner volume-fee share is credited to an account that can never spend it, forever. What this report claims as its own loss is the part of that credit a recorded leaf would actually have made spendable, `floor(share / 10^10) * 10^10` (`pallets/zk-tree/src/tree.rs:146-148`); the sub-quantum remainder is the sibling report's class, and the boundary is drawn explicitly under Impact. The sibling pallet that pays the **same** account from the **same** helper does record the leaf (`pallets/mining-rewards/src/lib.rs:230-237`), which is what establishes that the omission here is a bug rather than a design choice.

## Root Cause

**In-scope trigger.** The root cause is reached on-chain, permissionlessly, by a non-privileged party: `verify_private_batch` is an unsigned dispatchable admitted through the pallet's own `validate_unsigned` (`pallets/wormhole/src/lib.rs:583-585`, `:1002-1019`), so any network participant can submit a private-batch exit proof with no account, no fee payer, no role and no origin check. The block author whose fee share is stranded is the ordinary honest miner of that block, resolved from the QPoW pre-runtime digest that every mined block already carries (`primitives/wormhole/src/lib.rs:127-148`). This report does not rely on any privileged actor, admin key, governance action, operator, sequencer, compromised signer, off-chain infrastructure component or mock component: none of them appears anywhere on the path, and the defect fires identically when every such party behaves honestly. The only prerequisite is that a block settles a nonzero wormhole exit, which is the pallet's normal operation.

**The credit site (the omission).**

```rust
// pallets/wormhole/src/lib.rs:944-965
if !miner_fee.is_zero() {
    let digest = frame_system::Pallet::<T>::digest();
    if let Some(author) = qp_wormhole::extract_author_from_digest::<
        <T as frame_system::Config>::AccountId,
        _,
    >(digest.logs.iter())
    {
        match <T::Currency as Unbalanced<_>>::increase_balance(
            &author,
            miner_fee,
            frame_support::traits::tokens::Precision::Exact,
        ) {
            Ok(_) => {
                Self::deposit_event(Event::MinerVolumeFeePaid {
                    miner: author,
                    amount: miner_fee,
                });
            },
```

There is no `Self::record_transfer(...)` in that `Ok` arm. Contrast the exit credit in the same function, which uses the same event-free primitive and then records its leaf explicitly:

```rust
// pallets/wormhole/src/lib.rs:851-859
// Record transfer proof for the minted tokens
let from_account: <T as Config>::WormholeAccountId = mint_account.clone().into();
let to_account: <T as Config>::WormholeAccountId = exit_account.clone().into();
Self::record_transfer(
    T::AssetId::default(),
    &from_account,
    &to_account,
    *exit_balance,
);
```

**Fact 1: the recipient is keyless and the miner cannot opt out.** `extract_author_from_digest` takes the 32-byte pre-runtime payload and returns `derive_wormhole_address(preimage)`, never the raw bytes:

```rust
// primitives/wormhole/src/lib.rs:143-148
if data.len() != 32 {
    return None;
}
let preimage: [u8; 32] = data.as_slice().try_into().ok()?;
let address_bytes = derive_wormhole_address(preimage).ok()?;
return AccountId::decode(&mut &address_bytes[..]).ok();
```

`derive_wormhole_address` is `rehash_to_bytes(&inner_digest)` (`primitives/wormhole/src/lib.rs:109-111`), i.e. `address = Poseidon(Poseidon(salt || secret))`. Because the runtime hashes whatever the miner supplies, a miner cannot substitute a signable account. The only accounts that can sign are Poseidon images of ML-DSA public keys:

```rust
// primitives/dilithium-crypto/src/traits.rs:194-200
fn into_account(self) -> AccountId32 {
    // Use injective encoding for account ID derivation (collision-resistant for security)
    match self {
        Self::Dilithium87(who) => hash_bytes(who.as_ref()).into(),
        Self::Dilithium65(who) => hash_bytes(who.as_ref()).into(),
    }
}
```

and `Verify` rejects any signature whose derived account differs from the claimed signer (`primitives/dilithium-crypto/src/traits.rs:162-165, 173-176`). Producing a signature for the author address therefore requires a Poseidon preimage of a fixed 32-byte target, which is not available.

**Fact 2: the only spend instrument is a leaf.** `record_transfer` bumps the recipient's `TransferCount` and inserts the `(recipient, transfer_count, asset_id, amount)` leaf into the 4-ary Poseidon tree (`pallets/wormhole/src/lib.rs:1294-1310`). The exit path then admits an exit only for accounts and amounts that a proof binds to a tree leaf (`lib.rs:777-807`). A credit with no leaf has no representation the exit circuit can consume.

**Fact 3: no fallback recorder covers this credit.** The event-based recorder matches exactly four `Balances` events:

```rust
// runtime/src/transaction_extensions.rs:339, 350, 360, 372 (match arms)
RuntimeEvent::Balances(pallet_balances::Event::Transfer { .. })
RuntimeEvent::Balances(pallet_balances::Event::Minted { who, amount })
RuntimeEvent::Balances(pallet_balances::Event::TransferOnHold { .. })
RuntimeEvent::Balances(pallet_balances::Event::ReserveRepatriated { .. })
```

`Unbalanced::increase_balance` emits none of them, which is exactly why the pallet documents this credit primitive as event-free and pins it with `exit_credits_emit_no_scannable_transfer_events` (`pallets/wormhole/src/lib.rs:822-828`; `pallets/wormhole/src/tests.rs:792-835`). PoC assertion (5) re-proves it empirically. Independently, the scan is unreachable for this extrinsic: `verify_private_batch` is admitted through `validate_unsigned` (`pallets/wormhole/src/lib.rs:1002-1019`), and a bare extrinsic reaches `bare_post_dispatch`, which `WormholeProofRecorderExtension` does not implement and whose upstream default returns `Ok(())` without scanning (`sp-runtime-45.0.0/src/traits/transaction_extension/mod.rs:438-445`).

**The comparator that makes this an omission, not a design.** `pallet-mining-rewards` resolves the miner through the very same helper (`pallets/mining-rewards/src/lib.rs:205-208`) and, on every successful mint, records the leaf:

```rust
// pallets/mining-rewards/src/lib.rs:230-237
match T::Currency::mint_into(miner, reward) {
    Ok(_) => {
        T::ProofRecorder::record_transfer_proof(
            None, // Native token
            mint_account,
            miner.clone(),
            reward,
        );
```

Same payee, same account class, same runtime, opposite treatment.

## Attack Path

There is no attacker and no privileged actor. The loss is spontaneous and fires on any block that settles a wormhole exit.

{% stepper %}
{% step %}

## Submit a private-batch exit proof

A user submits a private-batch exit proof. `verify_private_batch` is unsigned and permissionless (`pallets/wormhole/src/lib.rs:583-585, 1002-1019`).
{% endstep %}

{% step %}

## Credit exits and record their leaves

`process_exit_bundle` credits the exits and records a leaf for each (`lib.rs:818-860`).
{% endstep %}

{% step %}

## Compute and split the volume fee

It computes `total_fee = volume_fee_for_exit(minted_exit, 4)` (`lib.rs:871-872, 1082-1092`), splits it `burn = Permill(50%) * fee`, `miner_fee = fee - burn` (`lib.rs:888-890`).
{% endstep %}

{% step %}

## Credit the block author without a leaf

It resolves the block author from the pre-runtime digest and credits `miner_fee` with `increase_balance` (`lib.rs:945-959`), then emits `MinerVolumeFeePaid` (`lib.rs:961-964`). The runtime now reports the miner as paid.
{% endstep %}

{% step %}

## Omit the leaf

No leaf exists for that credit, under either the raw or the Goldilocks-canonical keying (`lib.rs:1266-1271, 1294-1301`). The author's `TransferCount` is still 0.
{% endstep %}

{% step %}

## Funds are permanently unreachable

The author cannot sign (Fact 1) and has no leaf to prove (Fact 2), so the credited balance is unreachable forever.
{% endstep %}
{% endstepper %}

## Impact

Permanent freezing of funds: the block author's wormhole volume-fee share is credited to an account with no spend instrument, on every block that settles an exit, indefinitely.

### Corrected impact figures: what a recorded leaf would actually have made spendable

A ZK-tree leaf does not commit a planck amount. It commits `floor(amount / 10^10)` quanta:

```rust
// pallets/zk-tree/src/tree.rs:146-148
let amount_u128: u128 = leaf.amount.into();
let amount_quantized =
	u32::try_from(amount_u128 / AMOUNT_SCALE_DOWN_FACTOR).unwrap_or(u32::MAX);
```

with `AMOUNT_SCALE_DOWN_FACTOR = 10_000_000_000` (`pallets/zk-tree/src/tree.rs:34`). The value the missing leaf would actually have made spendable is therefore `floor(miner_share / 10^10) * 10^10`, not the whole credit. At the fee floor the miner share is 5,000,000,000 planck, half a quantum, so a leaf recorded there would have committed **zero** and the fix asked for here would have recovered nothing. The miner share first reaches one whole quantum when the bundle settles a two-quantum fee, that is at an exit volume of 25 UNIT per block or more (`ceil(exit_quanta * 4 / 9996) >= 2` from `exit_quanta >= 2500`, `pallets/wormhole/src/lib.rs:1082-1092`).

| Exit volume per block                       |   Miner share credited | Leaf-representable, recoverable by this fix | Loss attributable to THIS finding                                 |
| ------------------------------------------- | ---------------------: | ------------------------------------------: | ----------------------------------------------------------------- |
| floor (any nonzero exit) / 1 UNIT / 10 UNIT |   5,000,000,000 planck |                                           0 | **none** - see the sibling-overlap note below                     |
| 100 UNIT                                    |  25,000,000,000 planck |                       20,000,000,000 planck | 20,000,000,000 planck per settling block = 52,560 UNIT per year   |
| 1,000 UNIT                                  | 205,000,000,000 planck |                      200,000,000,000 planck | 200,000,000,000 planck per settling block = 525,600 UNIT per year |

Every credited figure in that table is measured on the production harnesses below and appears verbatim in `production_baseline_transcript.txt` and `persistent_baseline_transcript.txt`; the leaf-representable column is `floor(share / 10^10) * 10^10` at `pallets/zk-tree/src/tree.rs:146-148`; the annual column is 2,628,000 blocks per year, asserted in the harness from `TARGET_BLOCK_TIME_MS = 12_000` (`runtime/src/lib.rs:87`). The residue left over in every row is a constant 5,000,000,000 planck, the half-quantum that the 50 percent split at `pallets/wormhole/src/lib.rs:888-890` always leaves on a whole-quantum fee.

## Funds At Risk

* `Asset identity:` native Quantus planck. The loss asset is fixed at `pallets/wormhole/src/lib.rs:955` (`T::Currency` = `Balances`, wired at `runtime/src/configs/mod.rs`), settled in the native `Balance` type whose unit is `UNIT = 1_000_000_000_000` planck (`runtime/src/lib.rs:97`).
* `Unit->USD:` deliberately **not** converted. Quantus is pre-launch at this pin and has no listed market price; asserting a USD figure would be fabrication. The figures below are given in native UNIT and as a percentage of the fixed emission cap, which is the honest denominator available in-tree.
* `Market-size scenario:` the loss is a fixed fraction of wormhole exit volume. Scenarios below span a single minimal exit per block up to 1,000 UNIT of exit volume per block.
* `Absolute figure vs floor:` two figures are reported per row and they must not be conflated. The **credit-level** figure is the whole miner share the runtime settles: 5,000,000,000 planck per block at the arithmetic floor = 13,140 UNIT per year, 0.0626 percent of the 21,000,000 UNIT cap per year, rising to 65,700 UNIT per year at 100 UNIT of exit volume per block (0.313 percent of the cap per year). The **attributable** figure is the part of that credit a recorded leaf would have made spendable, `floor(share / 10^10) * 10^10`: it is **0 at the floor and at 1 and 10 UNIT of exit volume**, 20,000,000,000 planck per block (52,560 UNIT per year) at 100 UNIT, and 200,000,000,000 planck per block (525,600 UNIT per year) at 1,000 UNIT. This report claims the attributable figure only. No program-stated USD floor is claimed to be cleared, because no price exists at the pin.
* `Backing artifact:` the production harnesses and transcripts inlined under Proof Of Concept, at `production_constants_tests.rs`, `production_persistent_tests.rs`, `production_baseline_transcript.txt` and `persistent_baseline_transcript.txt`. The earlier standalone quantifier `quantify_miner_volume_fee.py` and its output `quantification_output.txt` are retained on disk and agree with the runtime measurements on every credit-level row; the runtime measurement supersedes them as the authority.

Arithmetic, at production constants only (`SCALE_DOWN_FACTOR = 10_000_000_000` at `pallets/wormhole/src/lib.rs:214`; `VolumeFeeRateBps = 4` at `runtime/src/configs/mod.rs:734`; `VolumeFeesBurnRate = Permill::from_percent(50)` at `:736`; `TARGET_BLOCK_TIME_MS = 12_000` at `runtime/src/lib.rs:87`, so 7,200 blocks per day and 2,628,000 per year):

```
total_fee = ceil( (exit_planck / 1e10) * 4 / 9996 ) * 1e10        [lib.rs:1082-1092]
miner_fee = total_fee - floor(total_fee / 2)                      [lib.rs:888-890]
leaf_representable = floor(miner_fee / 1e10) * 1e10               [zk-tree/tree.rs:146-148]

exit volume/block   miner fee credited/block   % of one block's author income   credited/year
        0.01 UNIT          5,000,000,000                    0.7110%              13,140 UNIT
        1.00 UNIT          5,000,000,000                    0.7110%              13,140 UNIT
       10.00 UNIT          5,000,000,000                    0.7110%              13,140 UNIT
      100.00 UNIT         25,000,000,000                    3.5549%              65,700 UNIT
    1,000.00 UNIT        205,000,000,000                   29.1505%             538,740 UNIT
   10,000.00 UNIT      2,005,000,000,000                        n/a           5,269,140 UNIT
```

Every row of that curve is printed by the production harnesses (`PRODUCTION CURVE` / `PERSISTENT CURVE` lines in the inlined transcripts), computed from constants the harness reads off the live runtime rather than from hard-coded values. It is the **credit-level** curve. The attributable curve is the same rows put through `floor(x / 1e10) * 1e10`, which is 0 for the first three rows and 20,000,000,000 / 200,000,000,000 planck per block for the 100 and 1,000 UNIT rows, as tabulated under Impact.

The "% of one block's author income" column uses 703,246,474,598 planck, the production block-author income measured on a real runtime block in the companion mining-rewards report; it is omitted for the 10,000 UNIT row because the share there exceeds a single block's author income and the ratio would not be meaningful. Because any nonzero exit pays at least one whole quantum of fee (`div_ceil` at `lib.rs:1090`), the floor row is exact, not an estimate: a block settling even one minimal exit credits 5,000,000,000 planck with no spend instrument, of which 0 is leaf-representable.

## Severity Justification

**Why this is not the High `Permanent freezing of unclaimed yield`** This is the one contested question on this finding and it is adjudicated here in full rather than asserted.

1. **The leaf is this chain's universal money representation, not a yield ticket.** The block-1 sweep records a leaf for **every** account holding a nonzero balance, that is for genesis principal, not for yield (`pallets/wormhole/src/lib.rs:478-488`). The identical `record_transfer` helper leafs a user's own exit principal eleven lines above the defect (`pallets/wormhole/src/lib.rs:854-859`). The runtime's event scanner leafs ordinary `Balances` credits of every kind (`runtime/src/transaction_extensions.rs:86-101`). A leaf is what a Quantus balance *is* on the spend side, for principal and for income alike, so the absence of one is not a yield-accrual condition.
2. **A wormhole address's free balance is a dead sink by design, so the leaf is the ONLY instrument.** The codebase states the spend model itself: a keyless account is one that "no signed call this extension weighs can move funds out of ... (`force_transfer` from it is Root-only)" (`runtime/src/transaction_extensions.rs:290-292`). Freezing the leaf therefore freezes the value outright rather than deferring a claim.
3. **The payee is not chooseable, so the miner cannot route the value to a signable account.** `extract_author_from_digest` forces `rehash_to_bytes(preimage)` (`primitives/wormhole/src/lib.rs:146-148`, `quantus-poseidon/src/lib.rs:193-198`), while a signable `AccountId` is `hash_bytes(ML-DSA pubkey)` over a disjoint encoding (`primitives/dilithium-crypto/src/traits.rs:194-199`). Making the two coincide is a cross-domain collision on a 32-byte target, on the order of `2^128` work. There is no opt-out and therefore no "leave it unclaimed" state of the miner's choosing.
4. **"Unclaimed" fails on its own terms.** The push into `System::Account(author).data.free` is completed at `pallets/wormhole/src/lib.rs:955-959` and announced as PAID by `MinerVolumeFeePaid` at `:961-964`. There is no accrual store, no pending-balance map and no claim step on this path; the only `claim()` dispatchable in the tree is `pallets/vesting/src/lib.rs:331`, in a different pallet, for a different mechanism. Nothing is awaiting a claim, so the value is not unclaimed.
5. **The canonical treatment of the IDENTICAL payee is mint plus leaf.** `pallet-mining-rewards` credits the same keyless block author, resolved by the same helper, and records the leaf on success (`pallets/mining-rewards/src/lib.rs:230-237`); its undeliverable fallback redirects to the treasury and **also** records a leaf (`:250-257`). The project's own design rule is that a credit to this payee class always carries a leaf. Omitting it is a defect against that rule, not a yield-accrual policy.

## Originality And Duplicate Defense

Post-audit-pin distinction: the audit pin is `3b243b870a5442a6e5f443056f67bbba87d472f5` and the code above is the pin state. The defect is live at the pin.

Nearest neighbours, each distinguished:

* `EQ-QNT-WORMHOLE-F-03` "Miner Rewards Silently Burned Due to Incorrect Preimage Handling" (Eiger, medium, `fixed_verified`): the wormhole credited the **raw preimage bytes** as an account instead of Poseidon-hashing them. That finding is about **which address** receives the fee. The fix introduced the shared `extract_author_from_digest` helper, which now correctly yields the keyless wormhole address. This report is about **what is recorded for that address**: the fix delivered the correct payee but did not add the leaf that makes the payee's credit spendable. Different defect, different line, and this one only became visible after that fix landed.
* `EQ-QNT-WORMHOLE-F-05` "Transfer Variants Missing from Wormhole Proof Recorder" (Eiger, `severity: low`, `fixed_verified`, dedup class `wormhole-transfer-callsite-omission`, recorded against `runtime/src/transaction_extensions.rs:129-167`): **the nearest structural neighbour, and it is not a duplicate.** F-05 is the same shape - a credit reaching an account with no proof record - but the severity turns entirely on **who the payee is**, and the two payees are opposites. In F-05 the payee is the `dest` argument of a user's own `Balances::transfer_all` or `Assets::transfer_approved`: the sender chooses it, it is ordinarily an ML-DSA-signable account, and a signable holder who has the value in `free` can still move it with an ordinary signed call. What F-05 loses is therefore **exit capacity through the wormhole**, not the value itself, which is exactly the `SEVERITY.md:48` shape ("Contract fails to deliver promised returns, but doesn't lose value") and is why Eiger rated it Low. Here the payee is **forced keyless**: `extract_author_from_digest` rehashes whatever preimage the digest carries (`primitives/wormhole/src/lib.rs:146-148`) and no ML-DSA public key hashes to that target (`primitives/dilithium-crypto/src/traits.rs:194-199`), so there is no signed call to fall back to and the **value itself** is lost. Same recorder-omission shape, opposite payee class, and therefore a different rubric row. F-05's own dedup note anticipates precisely this: "a newly omitted call site requiring a different fix is a potential novel variant". This is that variant, at a different file (`pallets/wormhole/src/lib.rs:955`, not `runtime/src/transaction_extensions.rs`) and with a different fix (add `record_transfer` at the credit site, not add match arms to the extractor).
* `PUBLIC-PR639-TREASURY-KEYLESS-SENTINEL-UNSPENDABLE-PAYOUTS` (`fixed_verified`): a genesis default sent **treasury** payouts to the keyless minting sentinel. Different pallet, different payee, different mechanism (a misconfigured destination, not a missing leaf). It is cited below as a designed-as-intended anchor, not as a duplicate.
* `PUBLIC-ISSUE417-MINING-MINT-INTO-FAILURE-SILENT-PROOF-EVENT` (`team_aware`): mining-rewards records a proof and emits a delivery event when `mint_into` **fails**. That is the inverse case (a leaf without a mint) in a different pallet. Here the mint succeeds and the leaf is absent.
* `PUBLIC-PR648-ZERO-AMOUNT-CREDITS-CREATE-PURE-STATE-GROWTH-LEAVES`, `PUBLIC-PR631-*`, `PUBLIC-PR654-*`: all concern leaves that **are** recorded (zero-value, non-canonical keying, insert pricing). None concerns a credit for which no leaf is recorded at all.
* No row in the 149-row corpus names `MinerVolumeFeePaid`, the volume-fee split at `lib.rs:888-890`, or the miner-fee credit at `lib.rs:955`.

### The merged change that touched this exact line, and what it left in place

The project has already looked at this credit and changed it. Pull request #619, "Emit MinerVolumeFeePaid event for wormhole exit volume fees", was merged on 2026-07-22, ahead of the pin `3b243b870a5442a6e5f443056f67bbba87d472f5` (2026-08-10), so its result **is** the code this report analyses. It is the strongest thing a reviewer could raise against this report's originality, so it is set out here in full rather than summarised.

**It states the same mechanical fact.** Its opening sentence reads: "On `Wormhole.verify_aggregated_proof`, the block author's share of the exit volume fee is minted via `increase_balance` **without** `record_transfer`." That is this report's call site and this report's missing call, named by the authors themselves. They also observed that "no wormhole-pallet event carries this amount", and contrasted the credit with the two paths that do announce themselves, "exit payouts (`NativeTransferred`) and mining rewards (`MinerRewarded` + `NativeTransferred`)".

**The harm it names is an off-chain accounting harm, and the remedy it ships matches that harm.** The stated consequence is that "indexed miner balances drift below RPC state on every wormhole exit (observed gap: 330980981066 planck on one deposit-only miner account)". The stated cause is that a bare `Balances.Minted` is one "which is ambiguous (exit payouts and mining rewards mint through the same event) and therefore intentionally ignored by our subsquid indexer". The stated cure is an off-chain companion change: "Companion subsquid PR consumes this event and credits `miner.free` directly." The problem being solved, start to finish, is that an off-chain indexer under-reports a balance the node already reports correctly.

**The consequence this report documents is not named anywhere in that discussion.** The body, both inline review comments and all three reviews were read in full. The words "unspendable", "spend", "frozen", "freeze", "strand", "keyless" and "leaf" appear zero times across all of them. The one inline exchange concerns a build dependency in `pallets/wormhole/Cargo.toml` and whether a pallet may take it and stay no-std ("This seems wrong, pallets are no-std, why do we need this new dependency?"); it does not reach the credit's recoverability. No participant asks whether the payee can move the value that was just credited to it, and no participant states an answer.

**What landed is an event, not a leaf.** The change touches five files and adds twelve lines to `pallets/wormhole/src/lib.rs`; not one of them is a `record_transfer` call. The new variant is described as "Emitted immediately after the `increase_balance` mint", and appended last in the enum "so existing variant indices (`NativeTransferred` = 0, `AssetTransferred` = 1, `ProofVerified` = 2) are unchanged for indexers decoding by metadata", which is an indexer-compatibility consideration rather than a state-recording one. The pin carries exactly that shape: `increase_balance` at `pallets/wormhole/src/lib.rs:955-959`, `MinerVolumeFeePaid` at `:961-964`, no call into the recorder at `:1280`, while the exit arm earlier in the same function still records its leaf at `:854`. Every figure and every assertion in this report was produced against that post-change code, and the fix-mutant control adds the recorder call the change did not.

**The two concerns are therefore stacked, not the same one.** Before the change the credit was invisible to the indexer and unmovable by the payee. After it the credit is visible to the indexer and still unmovable by the payee, because for a keyless block author it is the ZK-tree leaf, not the event, that makes a balance movable, for the reasons set out under Root Cause and under "Why the miner cannot simply exit with a proof". Stated plainly and with no suggestion of carelessness: the change closed the observability defect it set out to close and did so cleanly, and the by-product for the payee is that a miner whose indexer reads the new event and credits `miner.free` is now shown a balance that cannot be moved. The displayed number and the movable number were driven further apart rather than reconciled. This report's one-line fix at the credit site is disjoint from what that change shipped, and applying it disturbs neither the event nor the indexer that now consumes it.

**Open issue #563 is a different concern and does not reach this one.** "Silent shielded-supply inflation possible" asks for a per-asset turnstile counter, running deposits minus withdrawals, so that no withdrawal can exceed what was deposited and a forged inclusion proof cannot mint value into the shielded set; this report is the opposite direction, a genuine credit that already exists in `System::Account` and can never be withdrawn at all, which no turnstile of that design would either detect or repair.

## Proof of Concept
