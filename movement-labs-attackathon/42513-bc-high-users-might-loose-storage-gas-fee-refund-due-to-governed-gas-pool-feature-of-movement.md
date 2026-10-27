# 42513 \[BC-High] users might loose storage gas fee refund due to governed gas pool feature of movement logic bug&#x20;

## #42513 \[BC-High] Users might loose Storage Gas Fee Refund Due to Governed Gas Pool Feature of Movement logic bug

**Submitted on Mar 24th 2025 at 13:09:05 UTC by @perseverance for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42513
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**
  * Direct loss of funds

### Description

### Background Information

#### Brief/Intro

A bug exists in Movement Network's transaction fee handling mechanism where the gas fee refund behavior diverges when the `governed_gas_pool_enabled` feature flag is active. This inconsistency can lead to incorrect gas fee refunds and economic impacts on users.

**Overview of Storage Fees and Refunds**

Movemenet Network is based on Aptos and have Storage fees feature. Storage fees in Aptos are charged for persistently storing data on the blockchain, such as smart contract states or account data. These fees are measured in fixed APT (In Movement, Move is used), ensuring stability despite fluctuations in gas unit prices due to network load. A key feature of Aptos is that storage fees are fully refundable when the allocated storage slot is deleted, with the network configured to refund the entirety of the fee paid over the slot's lifetime ([Gas and Storage Fees | Aptos Docs](https://aptos.dev/en/network/blockchain/gas-txn-fee)).

The refund process involves minting new tokens and issuing them to the transaction payer, as outlined in Aptos Improvement Proposal (AIP-32) ([AIPs/aips/aip-32.md at main · aptos-foundation/AIPs](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-32.md)). This mechanism ensures that users are incentivized for removing unused storage, promoting efficient resource management.

### The Vulnerability

#### Vulnerability Details

The bug exists in the transaction validation module, specifically in the gas fee refund logic:

This `epilogue_gas_payer` function is exectuted at the end of every succesfull transaction to charge gas fee. **So this logic affects all transactions.**

<https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/framework/aptos-framework/sources/transaction\\_validation.move#L328-L333>

```move

/// Epilogue function with explicit gas payer specified, is run after a transaction is successfully executed.
    /// Called by the Adapter
    fun epilogue_gas_payer(
        account: signer,
        gas_payer: address,
        storage_fee_refunded: u64,
        txn_gas_price: u64,
        txn_max_gas_units: u64,
        gas_units_remaining: u64
    ) {

    if (amount_to_burn > storage_fee_refunded) {
            let burn_amount = amount_to_burn - storage_fee_refunded;
            if (features::governed_gas_pool_enabled()) {
                governed_gas_pool::deposit_gas_fee_v2(gas_payer, burn_amount);
            } else {
                transaction_fee::burn_fee(gas_payer, burn_amount);
            }
        } else if (amount_to_burn < storage_fee_refunded) { // @audit-issue 
            let mint_amount = storage_fee_refunded - amount_to_burn;
            if (!features::governed_gas_pool_enabled()) { // @audit-issue when governed_gas_pool_enabled() is true, no refund 
                transaction_fee::mint_and_refund(gas_payer, mint_amount);
            }
        };
}
```

The issue is that when `governed_gas_pool_enabled()` is true:

1. The refund logic is completely skipped
2. Users don't receive their entitled gas fee refunds
3. The difference between `storage_fee_refunded` and `amount_to_burn` is effectively lost

The governed\_gas\_pool() is a feature added by Movement team as proposed in MIP-52 Reference: <https://github.com/movementlabsxyz/MIP/pull/52/files#diff-2bc4e605841f320290eb506f2937e0fe7652e5fea468eb24d716ed5295423cbd>

Pull request added this feature: <https://github.com/movementlabsxyz/aptos-core/pull/114/files#diff-e2cd0cdc3954f53567cf594ea8649c44005a616790a8a585a4ae597734b309cf>

**So this feature is only related to Movement code change to Aptos-core and in scope of this Attackathon.**

In analysing the MIP-52, I understand that the proposal is that instead of burning the Gas fee, the Movement team proposed to transfer the gas fee to a Governed Gas Pool address. So this proposal should not affect the gas refund mechanism.

So the current bug can cause different behavior of the system when governed\_gas\_pool\_enabled() is turn on and off.

When governed\_gas\_pool\_enabled is turn off, the behavior is like Aptos, so users are refunded.

**When governed\_gas\_pool\_enabled is turn on, the fee surplus is not refunded. This will cause users to loose this money. In total gradually with time, with million of transactions, the sum can be big amount.**

Since the refund mechanism is to incentivise users to release state slots by deleting state items. So it is important to have consitent behavior.

<https://aptos.dev/en/network/blockchain/base-gas>

```
On the other hand, the storage refund incentivizes releasing state slots by deleting state items. The state slot fee is fully refunded upon slot deallocation, while the excess state byte fee is non-refundable. This will soon change by differentiating between permanent bytes (those in the global state) and relative ephemeral bytes (those that traverse the ledger history).

Some cost optimization strategies concerning the storage fee:

Minimize state item creation.
Minimize event emissions.
Avoid large state items, events, and transactions.
Clean up state items that are no longer in use.
If two fields are consistently updated together, group them into the same resource or resource group.
If a struct is large and only a few fields are updated frequently, move those fields to a separate resource or resource group.

```

### Severity Assessment

Bug Severity: Critical

Impact category: Direct loss of funds

This is assessed as Critical severity because:

**Impact:**

* Direct economic loss to users
* Affects all transactions when governed gas pool feature is enabled
* No way for users to recover lost refunds
* Can significantly impact high-frequency users and large transaction users

**Likelihood:**

* High - The issue occurs for every transaction with gas refunds when the feature governed\_gas\_pool\_enabled is enabled
* No special privileges required

### Mitigation

Probably the project should consider pay the refund from the governed gas pool coin to have consitent behavior for end users.

The fix should ensure consistent refund behavior regardless of the feature flag:

```move
    else if (amount_to_burn < storage_fee_refunded) {
    let mint_amount = storage_fee_refunded - amount_to_burn;
    // Always process refunds, but handle them differently based on the feature flag
    if (features::governed_gas_pool_enabled()) { // @note to fix the bug 
        // Handle refund through governed gas pool mechanism
        governed_gas_pool::process_refund(gas_payer, mint_amount);
    } else {
        // Use existing refund mechanism
        transaction_fee::mint_and_refund(gas_payer, mint_amount);
    }
};
```

### Proof of Concept

## POC

To create an e2e test for this is quite complicated. But I think to demonstrate this bug, it is enough to have a simple Unit test with input parameters to demonstrate this bug.

Input paramters:

```rust
let txn_gas_price = 100 // Octas 
let txn_max_gas_units = 1000; // Octas 
let gas_units_remaining = 980;
let storage_fee_refunded = 5000;
let governed_gas_pool_enabled = true 

    epilogue(account, storage_fee_refunded , txn_gas_price ,txn_max_gas_units ,  gas_units_remaining) ;   // epilogue is a public function will call epilogue_gas_payer() 
    
```

When execute the unit test

```rust
    gas_used = txn_max_gas_units - gas_units_remaining = 20 
    transaction_fee = txn_gas_price * gas_used = 20 * 1000 = 2000
    amount_to_burn = 2000 

    if (amount_to_burn < storage_fee_refunded) { // Evaluated as true 
            let mint_amount = storage_fee_refunded - amount_to_burn; // = 3000 
            if (!features::governed_gas_pool_enabled()) {
                transaction_fee::mint_and_refund(gas_payer, mint_amount);
            }
        };

```

So in this unit test, when governed\_gas\_pool\_enabled is false, user will be refunded 3000 Octas. But when governed\_gas\_pool\_enabled is true, nothing is refunded.

This vulnerability represents a significant deviation from expected behavior and can lead to economic losses for users when the governed gas pool feature is enabled.
