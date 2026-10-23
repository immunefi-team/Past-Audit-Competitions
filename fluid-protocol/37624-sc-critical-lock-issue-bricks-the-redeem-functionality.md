# #37624 \[SC-Critical] lock issue bricks the redeem functionality

## #37624 \[SC-Critical] Lock Issue Bricks The Redeem Functionality

**Submitted on Dec 11th 2024 at 01:20:08 UTC by @Blockian for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37624
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/trove-manager-contract/src/main.sw>
* **Impacts:**
  * Permanent freezing of funds
  * Protocol insolvency

### Description

## Fluid Bug Report

### Lock Issue Bricks The Redeem Functionality

#### Overview

There is a critical issue in the `redeem_collateral`, where the `lock_internal_redeem_collateral_from_trove` flag is permanently set to `true`. This effectively disables the `redeem_collateral` functionality, preventing users from redeeming USDF and severely impacting protocol operations.

### Root Cause

The bug resides in the `redeem_collateral_from_trove` logic within the `trove_manager`. When a redemption operation is cancelled, the system fails to release the lock on `internal_redeem_collateral_from_trove`. This oversight prevents future invocations of the function, leaving the relevant troves perpetually locked.

#### Relevant Code

The issue arises in the following snippet of the `trove_manager` implementation:

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
        // ... not relevant parts
        if (new_debt < MIN_NET_DEBT) {
            // Issue: Lock is not released on cancellation
            single_redemption_values.cancelled_partial = true;
            return single_redemption_values;
        }
```

The missing lock release upon redemption cancellation results in a permanent lock on the function.

### Impact

The perpetual lock prevents any subsequent calls to `internal_redeem_collateral_from_trove`, rendering the `redeem_collateral` function entirely inoperable. This effectively disables a core feature of the protocol, creating a severe usability issue for end-users.

### Exploit Vector

An attacker can systematically exploit this vulnerability to disable the redemption functionality for all troves using the following steps:

1. Create a trove with an Initial Collateral Ratio (ICR) of 135% and a debt of $500.
2. Attempt to redeem a small amount (e.g., $100), triggering a redemption cancellation due to the resulting debt falling below the minimum net debt threshold.
3. Close the trove and withdraw the collateral.

This sequence can be repeated across all troves in a single transaction, effectively locking out the `redeem_collateral` functionality for the entire system. The attack carries no risk to the attacker, as it does not expose them to liquidation or full redemption vulnerabilities since the position is closed in the same transaction.

### Proposed Solutions

There are two possible solutions:

1. **Release the Lock on Cancellation**\
   Ensure that the `lock_internal_redeem_collateral_from_trove` is reset to `false` when a redemption operation is cancelled.
2. **Eliminate the Lock Mechanism**\
   Evaluate the necessity of the lock mechanism in the current system design. If it is deemed redundant, remove it entirely. Additionally, The Fuel framework offers a better alternative to address reentrancy concerns without relying on storage based locks.

## Immunefi Bath Robe Post

<https://x.com/immunefi/status/1866040553491616112>

Can I now get the bath robe?

### Proof of Concept

### Proof of Concept

Run `forc test` after applying the following steps:

1. Add the following method to the `trove_manager_interface`:

```rs
    #[storage(read)]
    fn get_lock_internal_redeem_collateral_from_trove() -> bool;
```

2. Apply the git patch below:

```diff
diff --git a/contracts/trove-manager-contract/src/main.sw b/contracts/trove-manager-contract/src/main.sw
index 7e02245..cc408b0 100644
--- a/contracts/trove-manager-contract/src/main.sw
+++ b/contracts/trove-manager-contract/src/main.sw
@@ -140,6 +140,10 @@ impl TroveManager for Contract {
     fn get_trove_rewards_snapshot(id: Identity) -> RewardSnapshot {
         return storage.reward_snapshots.get(id).read();
     }
+    #[storage(read)]
+    fn get_lock_internal_redeem_collateral_from_trove() -> bool {
+        return storage.lock_internal_redeem_collateral_from_trove.read();
+    }
     #[storage(read, write)]
     fn redeem_collateral_from_trove(
         borrower: Identity,
@@ -149,7 +153,7 @@ impl TroveManager for Contract {
         upper_partial_hint: Identity,
         lower_partial_hint: Identity,
     ) -> SingleRedemptionValues {
-        require_caller_is_protocol_manager_contract();
+        // require_caller_is_protocol_manager_contract();
         internal_redeem_collateral_from_trove(
             borrower,
             max_usdf_amount,
@@ -183,7 +187,7 @@ impl TroveManager for Contract {
     }
     #[storage(read, write)]
     fn set_trove_status(id: Identity, status: Status) {
-        require_caller_is_borrow_operations_contract();
+        // require_caller_is_borrow_operations_contract();
         match storage.troves.get(id).try_read() {
             Some(trove) => {
                 let mut new_trove = trove;
@@ -199,17 +203,17 @@ impl TroveManager for Contract {
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
@@ -224,7 +228,7 @@ impl TroveManager for Contract {
     }
     #[storage(read, write)]
     fn add_trove_owner_to_array(id: Identity) -> u64 {
-        require_caller_is_borrow_operations_contract();
+        // require_caller_is_borrow_operations_contract();
         storage.trove_owners.push(id);
         let indx = storage.trove_owners.len() - 1;
         let mut trove = storage.troves.get(id).read();
@@ -280,7 +284,7 @@ impl TroveManager for Contract {
     }
     #[storage(read, write)]
     fn update_trove_reward_snapshots(id: Identity) {
-        require_caller_is_borrow_operations_contract();
+        // require_caller_is_borrow_operations_contract();
         internal_update_trove_reward_snapshots(id);
     }
     #[storage(read)]
@@ -979,3 +983,25 @@ fn internal_update_system_snapshots_exclude_coll_remainder(coll_remainder: u64)
         .total_collateral_snapshot
         .write(active_pool_coll - coll_remainder + liquidated_coll);
 }
+
+#[test]
+fn test_bricking_redeem_function() {
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
+    assert(caller.get_lock_internal_redeem_collateral_from_trove() == false); // lock is free before redemption cancellation
+
+    let _ = caller.redeem_collateral_from_trove(sender, 10, price, 0, sender, sender); // this will cause a redeeption cancellation
+
+    assert(caller.get_lock_internal_redeem_collateral_from_trove()); // lock is locked after redemption cancellation
+}
\ No newline at end of file
```
