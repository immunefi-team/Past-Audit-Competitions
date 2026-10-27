# #37607 \[SC-Low] bricking redeem function

## #37607 \[SC-Low] Bricking Redeem Function

**Submitted on Dec 10th 2024 at 12:11:56 UTC by @Blockian for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37607
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/protocol-manager-contract/src/main.sw>
* **Impacts:**
  * Protocol insolvency
  * Permanent freezing of funds

### Description

## Fluid Bug Report

### Bricking Redeem Functionality

#### Overview

A critical issue in the `redeem_collateral` function causes it to always revert, effectively preventing the redemption of USDF. This renders the `redeem_collateral` functionality inoperable under certain conditions.

### Root Cause Analysis

The problem originates from the `get_all_assets_info` function. Specifically, when it attempts to populate the current borrower for a specific asset, it calls `trove_manager.get_current_icr` for the candidate `current_borrower`. If this call is made for a non-existent identity (e.g., `null_identity_address`), the function reverts.

This behavior occurs because `trove_manager.get_current_icr` tries to read from an uninitialized storage address, resulting in a revert in Sway. The chain of function calls leading to this issue is as follows:

* **`trove_manager.get_current_icr`** → **`internal_get_current_icr`** → **`internal_get_entire_debt_and_coll`** → **`internal_get_pending_asset_reward`**.

The problematic code snippet:

```rs
fn internal_get_pending_asset_reward(address: Identity) -> u64 {
    let snapshot_asset = storage.reward_snapshots.get(address).read().asset; // Reverts here
    // ...
    if (reward_per_unit_staked == 0
        || storage.troves.get(address).read().status != Status::Active) // Also a potential revert
    {
        // ...
    }
}
```

#### Why Does This Occur?

The `get_all_assets_info` function calls `trove_manager.get_current_icr` with the candidate `current_borrower` has an ICR lower than the MCR. In this case, the function fetches the previous borrower using `sorted_troves.get_prev`. If the previous borrower is `null`, it results in a revert.

Relevant code snippet:

```rs
fn get_all_assets_info() -> AssetInfo {
    // ...
    while (current_borrower != null_identity_address() && current_cr < MCR) {
        current_borrower = sorted_troves.get_prev(current_borrower, asset); // Returns null
        current_cr = trove_manager.get_current_icr(current_borrower, price); // Reverts here
    }
    // ...
}
```

### How Can This Occur?

This issue arises when all borrowers in a specific trove have an ICR below the MCR. This scenario can occur due to:

* Sudden token price drops (e.g., rug pulls, de-pegging, or hacks).
* Other events leading to a significant decrease in collateral value.

While the likelihood of this issue may seem low, the protocol's support for a wide variety of tokens—along with future additions—amplifies the probability of encountering such a situation over time.

### Impact

The inability to call `redeem_collateral` disrupts the protocol's core functionality. A potential workaround could involve liquidating the entire trove. However, the protocol enforces a constraint (`require_more_than_one_trove_in_system`) that prevents liquidating all troves. As a result, the affected trove cannot be liquidated, leaving the `redeem_collateral` function effectively bricked.

### Proposed Solution

Modify the behavior of `trove_manager.get_current_icr` to return `u64::max` when called for a non-existent trove. This change ensures the function handles edge cases gracefully, avoiding reverts in scenarios where borrowers or troves are invalid.

### Proof of Concept

### Proof of Concept

Run the following commands after applying the git diff:

`forc test test_works_for_existing_trove --logs` -> runs the control test

`forc test test_reverts_for_non_existing_trove --logs` -> run the test that fails

1. Apply the git patch below:

```diff
diff --git a/contracts/trove-manager-contract/src/main.sw b/contracts/trove-manager-contract/src/main.sw
index 7e02245..dd93eaa 100644
--- a/contracts/trove-manager-contract/src/main.sw
+++ b/contracts/trove-manager-contract/src/main.sw
@@ -183,7 +183,7 @@ impl TroveManager for Contract {
     }
     #[storage(read, write)]
     fn set_trove_status(id: Identity, status: Status) {
-        require_caller_is_borrow_operations_contract();
+        // require_caller_is_borrow_operations_contract();
         match storage.troves.get(id).try_read() {
             Some(trove) => {
                 let mut new_trove = trove;
@@ -204,7 +204,7 @@ impl TroveManager for Contract {
     }
     #[storage(read, write)]
     fn update_stake_and_total_stakes(id: Identity) -> u64 {
-        require_caller_is_borrow_operations_contract();
+        // require_caller_is_borrow_operations_contract();
         internal_update_stake_and_total_stakes(id)
     }
     #[storage(read, write)]
@@ -280,7 +280,7 @@ impl TroveManager for Contract {
     }
     #[storage(read, write)]
     fn update_trove_reward_snapshots(id: Identity) {
-        require_caller_is_borrow_operations_contract();
+        // require_caller_is_borrow_operations_contract();
         internal_update_trove_reward_snapshots(id);
     }
     #[storage(read)]
@@ -979,3 +979,24 @@ fn internal_update_system_snapshots_exclude_coll_remainder(coll_remainder: u64)
         .total_collateral_snapshot
         .write(active_pool_coll - coll_remainder + liquidated_coll);
 }
+
+#[test]
+fn test_reverts_for_non_existing_trove() {
+    let price = 3_000_000_000_000;
+
+    let caller = abi(TroveManager, CONTRACT_ID);
+
+    let icr = caller.get_current_icr(null_identity_address(), price);
+}
+
+#[test]
+fn test_works_for_existing_trove() {
+    let price = 3_000_000_000_000;
+
+    let caller = abi(TroveManager, CONTRACT_ID);
+    caller.set_trove_status(null_identity_address(), Status::Active);
+    caller.update_trove_reward_snapshots(null_identity_address());
+    let _ = caller.update_stake_and_total_stakes(null_identity_address());
+
+    let icr = caller.get_current_icr(null_identity_address(), price);
+}
\ No newline at end of file
```
