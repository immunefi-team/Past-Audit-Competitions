# #37425 \[SC-Insight] redeem collateral does not redeem collateral from riskiest trove but wrongly red

## #37425 \[SC-Insight] redeem\_collateral does not redeem collateral from riskiest trove but wrongly redeem lowest healthy troves with lowest collateral Ratio

**Submitted on Dec 4th 2024 at 14:14:53 UTC by @perseverance for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37425
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/protocol-manager-contract/src/main.sw>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

### Description

## Description

### Brief/Intro

The redeem\_collateral() in protocol\_manager can redeem collateral at any time to receive collateral.

<https://github.com/Hydrogen-Labs/fluid-protocol/blob/main/contracts/protocol-manager-contract/src/main.sw#L134-L139>

```rust
#[storage(read, write), payable]
    fn redeem_collateral(
        max_iterations: u64,
        partial_redemption_hint: u64,
        upper_partial_hint: Identity,
        lower_partial_hint: Identity,
    ) 

```

According to the [documentation of Fluid Protocol](https://docs.hydrogenlabs.xyz/fluid-protocol-community/protocol-design/redemption),

```
When USDF is redeemed, the collateral provided to the redeemer is allocated from the Trove(s) with the lowest collateral ratio (CR), even if it is above 135%.

```

```
When redeemed, the system uses the USDF to repay the riskiest Trove(s) based on the lowest collateral ratio, and transfers the respective amount of collateral from the affected positions to the redeemer.

On the contrary, redemptions do have a positive effect on the total collateralization of the protocol, improving robustness and stability.
```

So according to the design of the protocol, the redeem\_collateral should redeem from the riskiest trove means that the trove with lowest collateral ratio.

### The vulnerability

#### Vulnerability Details

The vulnerability in redeem\_collateral() in internal function get\_all\_assets\_info() from line 328-328.

Line 318: Find the last node in sorted\_troves for the asset. This is the riskiest trove for the current asset.

Line 326-328: Loop until it finds the current\_borrower that is not zero and have current\_cr >= MCR = 135%. Then push that current\_borrower to the current\_borrowers vector at Line 330.

**So it means that current\_borrowers vector contains that borrower that have collateral ratio is healthy means >= 135%**

**So notice that in the case, there are troves with Collateral Ratio < 135% that is unhealthy, the system still look for the healhthy troves to redeem.**

<https://github.com/Hydrogen-Labs/fluid-protocol/blob/main/contracts/protocol-manager-contract/src/main.sw#L313-L333>

```rust
    fn get_all_assets_info() -> AssetInfo {
        // Removed for simplicity
            while (i < length) {
         
Line 318:   let mut current_borrower = sorted_troves.get_last(asset);
            let mut current_cr = u64::max();
            if (current_borrower != null_identity_address()) {
                current_cr = trove_manager.get_current_icr(current_borrower, price);
            }
         
Line 326:    while (current_borrower != null_identity_address() && current_cr < MCR) {
Line 327:           current_borrower = sorted_troves.get_prev(current_borrower, asset);
Line 328:           current_cr = trove_manager.get_current_icr(current_borrower, price);
            }
         
Line 330:            current_borrowers.push(current_borrower);
            current_crs.push(current_cr);
            i += 1;
        }

       // Removed for simplicity 

   }
```

After that, at line 161, the contract find the borrower with the lowest CR in the current\_borrowers vector to redeem the collateral. At line 177, the contract call trove\_manager\_contract.redeem\_collateral\_from\_trove to redeem the collateral.

<https://github.com/Hydrogen-Labs/fluid-protocol/blob/main/contracts/protocol-manager-contract/src/main.sw#L159-L184>

```rust
        let mut assets_info = get_all_assets_info();
        let mut remaining_usdf = msg_amount();
Line 161   let (mut current_borrower, mut index) = find_min_borrower(assets_info.current_borrowers, assets_info.current_crs);
        let mut remaining_iterations = max_iterations;

        // Iterate through troves, redeeming collateral until conditions are met
        while (current_borrower != null_identity_address() && remaining_usdf > 0 && remaining_iterations > 0) {
            let contracts_cache = assets_info.asset_contracts.get(index).unwrap();
            let trove_manager_contract = abi(TroveManager, contracts_cache.trove_manager.bits());
            let price = assets_info.prices.get(index).unwrap();
            let mut totals = assets_info.redemption_totals.get(index).unwrap();
            remaining_iterations -= 1;
            let next_user_to_check = sorted_troves.get_prev(current_borrower, contracts_cache.asset_address);

            // Apply pending rewards to ensure up-to-date trove state
            trove_manager_contract.apply_pending_rewards(current_borrower);

            // Attempt to redeem collateral from the current trove
Line 177:       let single_redemption = trove_manager_contract.redeem_collateral_from_trove(
                current_borrower,
                remaining_usdf,
                price,
                partial_redemption_hint,
                upper_partial_hint,
                lower_partial_hint,
            );
    // Remove for simplicity
    }
```

In summary, the documentation states that when redeem the collateral, the protocol should redeem from the riskiest troves in the system. But in the code, it does not redeem from the riskiest. In the scenario that there are troves that are not healthy, the system still redeem from the healthy troves.

I give an example as the POC:

```rust
For asset_0 , price = 10

# Wallet_1 : 

collateral   = 2000 
debt = 10_000
CR = 2 

# Wallet_2 

collateral   = 900 
debt = 5_000
CR = 1.8 

# Wallet_3 

collateral   = 160 
debt = 1000
CR = 1.6


For Asset_1, price = 10 

# Wallet_2 

collateral   = 2000 
debt = 14_500
CR = 1.37

# Wallet_3 

collateral  = 500 
debt = 1000
CR = 5


Now if the price of asset_1 drop 10% and is 9 

For Wallet_2 

collateral value  = 2000*9 = 18_000
debt = 14_500
CR = 1.24

So now this trove is riskiest in the system. 
So if users want to redeem the collateral then it should take from this trove by liquidating the collateral of that trove. 
It should redeem from the trove with CR = 1.24

But the current code does redeem from wallet 3 of asset_0, then wallet_2 and then wallet_1 
```

**Note that this bug can affect the live contract of Fluid Protocol**.

So since the system does not take collateral from the riskiest trove, it cause other users to loose his collateral. So if he think that his position is not the lowest CR in the system, he is surprised to see that his collateral was redeemed and lost money.

## Impacts

## About the severity assessment

Bug Severity: **Critical**

Impact category:

**Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield**

### Link to Proof of Concept

<https://gist.github.com/Perseverancesuccess2021/a0a584cc51049b585872b1b87028b1a5#file-success\\_redemptions\\_many-rs>

### Proof of Concept

## Proof of concept

Steps to execute the attack:

### Step 1: Setup the precondition for this bug as described above

### Step 2: Call redeem\_collateral

## Test Code

Test code to demonstrate this bug: <https://gist.github.com/Perseverancesuccess2021/a0a584cc51049b585872b1b87028b1a5#file-success\\_redemptions\\_many-rs>

Replace the test code function proper\_multi\_collateral\_redemption\_from\_partially\_closed below with the Protocol proper\_multi\_collateral\_redemption\_from\_partially\_closed in fluid-protocol\contracts\protocol-manager-contract\tests\success\_redemptions\_many.rs

```rust
#[tokio::test]
async fn proper_multi_collateral_redemption_from_partially_closed() {
    let (contracts, _admin, mut wallets) = setup_protocol(5, true, false).await;

    let healthy_wallet1 = wallets.pop().unwrap();
    let healthy_wallet2 = wallets.pop().unwrap();
    let healthy_wallet3 = wallets.pop().unwrap();

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[0].mock_pyth_oracle,
        pyth_price_feed(10),
    )
    .await;

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[1].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[1].mock_pyth_oracle,
        pyth_price_feed(10),
    )
    .await;

    borrow_operations_utils::mint_token_and_open_trove(
        healthy_wallet1.clone(),
        &contracts.asset_contracts[0],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        2_000 * PRECISION,
        10_000 * PRECISION,
    )
    .await;
    
    
    println!("mint_token_and_open_trove: healthy_wallet1");
    

    borrow_operations_utils::mint_token_and_open_trove(
        healthy_wallet2.clone(),
        &contracts.asset_contracts[0],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        900 * PRECISION,
        5_000 * PRECISION,
    )
    .await;

    
    println!("mint_token_and_open_trove: healthy_wallet2");
    

    borrow_operations_utils::mint_token_and_open_trove(
        healthy_wallet3.clone(),
        &contracts.asset_contracts[0],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        160 * PRECISION,
        1_000 * PRECISION,
    )
    .await;

    
    println!("mint_token_and_open_trove: healthy_wallet3");
    

    borrow_operations_utils::mint_token_and_open_trove(
        healthy_wallet2.clone(),
        &contracts.asset_contracts[1],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        2_000 * PRECISION,
        14_500 * PRECISION,
    )
    .await;

    
    println!("mint_token_and_open_trove: healthy_wallet2 asset_contracts[1]");
    

    res_1 = borrow_operations_utils::mint_token_and_open_trove(
        healthy_wallet3.clone(),
        &contracts.asset_contracts[1],
        &contracts.borrow_operations,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.active_pool,
        &contracts.sorted_troves,
        5_00 * PRECISION,
        1_000 * PRECISION,
    )
    .await;

   
    println!("mint_token_and_open_trove: healthy_wallet3 asset_contracts[1]");


    println!("Simulate the scenario that price of asset_1 drop 10% ");  
   

    let feed = Bits256::zeroed(); 

    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[1].mock_pyth_oracle,
        pyth_price_feed(9),
    )
    .await;


    let price  = pyth_oracle_abi::price(       
        &contracts.asset_contracts[1].mock_pyth_oracle,
        &feed,
     ).await;

    println!("Price: {:?}", price); 

    let redemption_amount: u64 = 8_000 * PRECISION;

    let protocol_manager_health1 = ContractInstance::new(
        ProtocolManager::new(
            contracts.protocol_manager.contract.contract_id().clone(),
            healthy_wallet1.clone(),
        ),
        contracts.protocol_manager.implementation_id,
    );

    let pre_redemption_active_pool_debt = active_pool_abi::get_usdf_debt(
        &contracts.active_pool,
        contracts.asset_contracts[0].asset_id.into(),
    )
    .await
    .value;

    let res = protocol_manager_abi::redeem_collateral(
        &protocol_manager_health1,
        redemption_amount,
        20, // max iteration 
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
    
    println!("After redeem_collateral");
    println!("Logs: {:?}", logs);

    trove_manager_utils::assert_trove_coll(
        &contracts.asset_contracts[1].trove_manager,
        Identity::Address(healthy_wallet2.address().into()),
        2_000 * PRECISION,
    )
    .await;

    trove_manager_utils::assert_trove_debt(
        &contracts.asset_contracts[1].trove_manager,
        Identity::Address(healthy_wallet2.address().into()),
        14572500000000,
    )
    .await;
}


```

### Explanation:

In the POC, scenario is explained above.

The test log showed that the redemptionEvent for wallet\_1 and wallet\_2 and wallet\_3 for asset\_0. The collateral of wallet\_1 and wallet\_2 and wallet\_3 was redeemed.

```
Ok("RedemptionEvent { borrower: Address(5d99ee966b42cd8fc7bdd1364b389153a9e78b42b7d4a691470674e817888d4e), usdf_amount: 1005000000000, collateral_amount: 100500000000, collateral_price: 10000000000 }"), Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 31657500000000, sender: ContractId(85cec1ef66e096095e53e86f66dc5dde7f16f815b1d8c86785d59fbd2bc1be5f) }"), Ok("RedemptionEvent { borrower: Address(bdaad6a89e073e177895b3e5a9ccd15806749eda134a6438dae32fc5b6601f3f), usdf_amount: 5025000000000, collateral_amount: 502500000000, collateral_price: 10000000000 }"), Ok("RedemptionEvent { borrower: Address(95a7aa6cc32743f8706c40ef49a7423b47da763bb4bbc055b1f07254dc729036), usdf_amount: 1970000000000, collateral_amount: 197000000000, collateral_price: 10000000000 }")

```

**The trove of wallet\_2 with asset\_1 is unhealthy and is riskiest in the system, but stay untouched.**

Test log: <https://gist.github.com/Perseverancesuccess2021/a0a584cc51049b585872b1b87028b1a5#file-proper\\_multi\\_collateral\\_redemption\\_from\\_partially\\_closed\\_241204\\_1730-log>

```log
running 1 test
test success_redemptions_many::proper_multi_collateral_redemption_from_partially_closed ... Deploying core contracts...
Initializing core contracts...
Log events: LogResult { results: [Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 50000000000, sender: ContractId(649bcbbeb992e86e7b55467aa3dbc2bc9c8b0910f31538213b9988a28bbccbae) }"), Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 10050000000000, sender: ContractId(649bcbbeb992e86e7b55467aa3dbc2bc9c8b0910f31538213b9988a28bbccbae) }"), Ok("OpenTroveEvent { user: Address(95a7aa6cc32743f8706c40ef49a7423b47da763bb4bbc055b1f07254dc729036), asset_id: 50ca32ac0ff3f0736818008c5b5d31393b426d4032f949f90a1fb5dd6f47ca4d, collateral: 2000000000000, debt: 10050000000000 }")] }
mint_token_and_open_trove: healthy_wallet1
Log events: LogResult { results: [Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 10075000000000, sender: ContractId(649bcbbeb992e86e7b55467aa3dbc2bc9c8b0910f31538213b9988a28bbccbae) }"), Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 15075000000000, sender: ContractId(649bcbbeb992e86e7b55467aa3dbc2bc9c8b0910f31538213b9988a28bbccbae) }"), Ok("OpenTroveEvent { user: Address(bdaad6a89e073e177895b3e5a9ccd15806749eda134a6438dae32fc5b6601f3f), asset_id: 50ca32ac0ff3f0736818008c5b5d31393b426d4032f949f90a1fb5dd6f47ca4d, collateral: 900000000000, debt: 5025000000000 }")] }
mint_token_and_open_trove: healthy_wallet2
Log events: LogResult { results: [Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 15080000000000, sender: ContractId(649bcbbeb992e86e7b55467aa3dbc2bc9c8b0910f31538213b9988a28bbccbae) }"), Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 16080000000000, sender: ContractId(649bcbbeb992e86e7b55467aa3dbc2bc9c8b0910f31538213b9988a28bbccbae) }"), Ok("OpenTroveEvent { user: Address(5d99ee966b42cd8fc7bdd1364b389153a9e78b42b7d4a691470674e817888d4e), asset_id: 50ca32ac0ff3f0736818008c5b5d31393b426d4032f949f90a1fb5dd6f47ca4d, collateral: 160000000000, debt: 1005000000000 }")] }
mint_token_and_open_trove: healthy_wallet3
Log events: LogResult { results: [Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 16152500000000, sender: ContractId(649bcbbeb992e86e7b55467aa3dbc2bc9c8b0910f31538213b9988a28bbccbae) }"), Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 30652500000000, sender: ContractId(649bcbbeb992e86e7b55467aa3dbc2bc9c8b0910f31538213b9988a28bbccbae) }"), Ok("OpenTroveEvent { user: Address(bdaad6a89e073e177895b3e5a9ccd15806749eda134a6438dae32fc5b6601f3f), asset_id: 287af6dc9c5b5c6f7839d92fd3b1d3407f941ea5eb219d6a1e496c575039d4a5, collateral: 2000000000000, debt: 14572500000000 }")] }
mint_token_and_open_trove: healthy_wallet2 asset_contracts[1]
Log events: LogResult { results: [Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 30657500000000, sender: ContractId(649bcbbeb992e86e7b55467aa3dbc2bc9c8b0910f31538213b9988a28bbccbae) }"), Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 31657500000000, sender: ContractId(649bcbbeb992e86e7b55467aa3dbc2bc9c8b0910f31538213b9988a28bbccbae) }"), Ok("OpenTroveEvent { user: Address(5d99ee966b42cd8fc7bdd1364b389153a9e78b42b7d4a691470674e817888d4e), asset_id: 287af6dc9c5b5c6f7839d92fd3b1d3407f941ea5eb219d6a1e496c575039d4a5, collateral: 500000000000, debt: 1005000000000 }")] }
mint_token_and_open_trove: healthy_wallet3 asset_contracts[1]
Simulate the scenario that price of asset_1 drop 10% 
Price: CallResponse { value: Price { confidence: 0, exponent: 9, price: 9000000000, publish_time: 1724166967 }, receipts: [Call { id: 0000000000000000000000000000000000000000000000000000000000000000, to: ef1bfbda29454e1a3222c84446cf317effcfca3367feb9458ffd293083ed06ec, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 3028, param1: 10480, param2: 10493, pc: 11712, is: 11712 }, ReturnData { id: ef1bfbda29454e1a3222c84446cf317effcfca3367feb9458ffd293083ed06ec, ptr: 67107328, len: 28, digest: 4a6a5d6235b410d88d1d8f0c5df8aeb12fbf796e102e0984b148b5cf7955c206, pc: 14328, is: 11712, data: Some(00000000000000000000000900...) }, Return { id: 0000000000000000000000000000000000000000000000000000000000000000, val: 1, pc: 10388, is: 10368 }, ScriptResult { result: Success, gas_used: 3075 }], gas_used: 3075, log_decoder: LogDecoder { log_formatters: {LogId(ef1bfbda29454e1a3222c84446cf317effcfca3367feb9458ffd293083ed06ec, "10098701174489624218"): LogFormatter { type_id: TypeId { t: (4803088177761407886, 17565082866899838598) } }}, decoder_config: DecoderConfig { max_depth: 45, max_tokens: 10000 } }, tx_id: Some(a99e6cc042a2f7dac2c93d4ff8a13c76fd9ae7804d1281a5fc8d3ab52cc7130b) }
After redeem_collateral
Logs: LogResult { results: [Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 31657500000000, sender: ContractId(85cec1ef66e096095e53e86f66dc5dde7f16f815b1d8c86785d59fbd2bc1be5f) }"), Ok("RedemptionEvent { borrower: Address(5d99ee966b42cd8fc7bdd1364b389153a9e78b42b7d4a691470674e817888d4e), usdf_amount: 1005000000000, collateral_amount: 100500000000, collateral_price: 10000000000 }"), Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 31657500000000, sender: ContractId(85cec1ef66e096095e53e86f66dc5dde7f16f815b1d8c86785d59fbd2bc1be5f) }"), Ok("RedemptionEvent { borrower: Address(bdaad6a89e073e177895b3e5a9ccd15806749eda134a6438dae32fc5b6601f3f), usdf_amount: 5025000000000, collateral_amount: 502500000000, collateral_price: 10000000000 }"), Ok("RedemptionEvent { borrower: Address(95a7aa6cc32743f8706c40ef49a7423b47da763bb4bbc055b1f07254dc729036), usdf_amount: 1970000000000, collateral_amount: 197000000000, collateral_price: 10000000000 }"), Ok("TotalSupplyEvent { asset: 7b7155e342577a7fe2a506dc0c9ef7fddb9099720b17391830907d8cc23bbc26, supply: 23657500000000, sender: ContractId(2da3e2bca5712082abb59bc293d8eac5b612b82ed92ff5c96b521d8d76bf318d) }")] }
ok

```

### Full POC:

Download Patch files: <https://drive.google.com/file/d/1kYg8\\_INyCcEZPCxM4FhK8vU6ZlfZpM5c/view?usp=sharing>

In folder fluid-protocol , apply all patches file based on commit: 78ab7bdd243b414b424fca6e1eb144218f36a18a

```
git am *.patch 
```

```bash
 cargo test --release success_redemptions_many::proper_multi_collateral_redemption_from_partially_closed -- --exact --test-threads=1 --nocapture

```
