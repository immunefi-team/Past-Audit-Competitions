# #37650 \[SC-Low] redeem functionality partially failing

## #37650 \[SC-Low] Redeem Functionality Partially Failing

**Submitted on Dec 11th 2024 at 16:17:08 UTC by @Blockian for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37650
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/trove-manager-contract/src/main.sw>
* **Impacts:**
  * Protocol insolvency
  * USDF Pegging Mechanism Issue

### Description

## Fluid Bug Report

### Redeem Functionality Partially Failing

#### Overview

A design issue in the trove management system causes the `redeem_collateral` function to revert under certain conditions, even when sufficient troves exist in the system to allow redemptions.

#### Root Cause

The trove manager enforces a rule that at least one active trove must always remain for the managed asset. This constraint is implemented through the `require_more_than_one_trove_in_system` function.

However, this rule applies individually to every trove manager associated with each supported asset. Since some assets are less utilized than others, this creates an imbalance. For example, the Fluid system currently has 3 troves open for `wstETH` compared to 146 open for `ETH`.

If a user attempts to redeem USDF, the redemption will fail if even a single trove manager (for any asset) has only one active trove, regardless of the number of troves available across the entire system. This effectively blocks the redemption process despite there being sufficient collateral system-wide.

#### Impact

This design limitation could hinder the usability of the protocol, especially as new assets are introduced dynamically. The redeem functionality, being a critical part of the pegging mechanism for USDF, may fail under scenarios where less-utilized assets have only one trove active. This poses a risk to the stability of the USDF peg and limits the protocol’s efficiency.

#### Proposed Solution

Since this issue stems from a design issue rather than a straightforward coding bug, resolving it requires careful consideration. One possible approach might involve revisiting the requirement for every trove manager to maintain at least one active trove. However, giving up this constraint could have implications for other parts of the system, so is it a good choice?

So to be honest, I don't think I'm qualified to propose a solution to this issue.

### Proof of Concept

### Proof of Concept

Run the following command after applying the git diff:

`forc test test_close_trove_fails_for_a_single_trove --logs` -> runs the test

1. Apply the git patch below:

```diff
diff --git a/contracts/trove-manager-contract/src/main.sw b/contracts/trove-manager-contract/src/main.sw
index 7e02245..d488627 100644
--- a/contracts/trove-manager-contract/src/main.sw
+++ b/contracts/trove-manager-contract/src/main.sw
@@ -149,7 +149,7 @@ impl TroveManager for Contract {
         upper_partial_hint: Identity,
         lower_partial_hint: Identity,
     ) -> SingleRedemptionValues {
-        require_caller_is_protocol_manager_contract();
+        // require_caller_is_protocol_manager_contract();
         internal_redeem_collateral_from_trove(
             borrower,
             max_usdf_amount,
@@ -183,7 +183,7 @@ impl TroveManager for Contract {
     }
     #[storage(read, write)]
     fn set_trove_status(id: Identity, status: Status) {
-        require_caller_is_borrow_operations_contract();
+        // require_caller_is_borrow_operations_contract();
         match storage.troves.get(id).try_read() {
             Some(trove) => {
                 let mut new_trove = trove;
@@ -199,17 +199,17 @@ impl TroveManager for Contract {
     }
     #[storage(read, write)]
     fn increase_trove_coll(id: Identity, coll: u64) -> u64 {
-        require_caller_is_borrow_operations_contract();
+        // require_caller_is_borrow_operations_contract();
         internal_increase_trove_coll(id, coll)
     }
     #[storage(read, write)]
     fn update_stake_and_total_stakes(id: Identity) -> u64 {
-        require_caller_is_borrow_operations_contract();
+        // require_caller_is_borrow_operations_contract();
         internal_update_stake_and_total_stakes(id)
     }
     #[storage(read, write)]
     fn increase_trove_debt(id: Identity, debt: u64) -> u64 {
-        require_caller_is_borrow_operations_contract();
+        // require_caller_is_borrow_operations_contract();
         internal_increase_trove_debt(id, debt)
     }
     #[storage(read, write)]
@@ -224,7 +224,7 @@ impl TroveManager for Contract {
     }
     #[storage(read, write)]
     fn add_trove_owner_to_array(id: Identity) -> u64 {
-        require_caller_is_borrow_operations_contract();
+        // require_caller_is_borrow_operations_contract();
         storage.trove_owners.push(id);
         let indx = storage.trove_owners.len() - 1;
         let mut trove = storage.troves.get(id).read();
@@ -244,7 +244,7 @@ impl TroveManager for Contract {
     }
     #[storage(read, write)]
     fn close_trove(id: Identity) {
-        require_caller_is_borrow_operations_contract();
+        // require_caller_is_borrow_operations_contract();
         internal_close_trove(id, Status::ClosedByOwner);
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
@@ -979,3 +979,22 @@ fn internal_update_system_snapshots_exclude_coll_remainder(coll_remainder: u64)
         .total_collateral_snapshot
         .write(active_pool_coll - coll_remainder + liquidated_coll);
 }
+
+#[test]
+fn test_close_trove_fails_for_a_single_trove() {
+    let price = 3_000_000_000_000;
+    let debt = 500_000_000_000;
+    let coll = fm_multiply_ratio(debt * 2, DECIMAL_PRECISION, price); // not really relevant
+    let sender = null_identity_address();
+
+    let caller = abi(TroveManager, CONTRACT_ID);
+    caller.set_trove_status(sender, Status::Active);
+    let _ = caller.increase_trove_coll(sender, coll);
+    let _ = caller.increase_trove_debt(sender, 500_000_000_000); // 500 USDF
+    caller.update_trove_reward_snapshots(sender);
+    let _ = caller.update_stake_and_total_stakes(sender);
+    let _ = caller.add_trove_owner_to_array(sender);
+
+
+    caller.close_trove(sender); // reverts
+}
\ No newline at end of file
```
