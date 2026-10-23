# #37343 \[SC-Insight] inaccurate check leading to debt miscalculation

## #37343 \[SC-Insight] Inaccurate Check Leading to Debt Miscalculation

**Submitted on Dec 2nd 2024 at 16:03:30 UTC by @Blockian for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37343
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/trove-manager-contract/src/main.sw>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Contract fails to deliver promised returns, but doesn't lose value

### Description

## Fluid Bug Report

### Inaccurate Check Leading to Debt Miscalculation

#### Overview

An edge case arises during reward application where a user's `pending_usdf` is not added to their debt due to an oversight. This miscalculation enables users to withdraw funds without accounting for the additional debt, potentially shifting the burden onto other protocol participants or causing minor fluctuations in the value of USDF due to inaccurate collateral backing.

#### Deep Dive

The issue stems from the `internal_has_pending_rewards` function, which verifies changes in the `l_asset` value but overlooks updates to the `l_usdf` value. Consequently, in scenarios where `l_usdf` changes while `l_asset` remains unchanged, the function erroneously returns false. This prevents the `internal_apply_pending_rewards` function from updating the pending rewards accurately.

How does `l_usdf` change without `l_asset`? The `internal_redistribute_debt_and_coll` function updates these values based on the latest liquidation data. Here's the critical implementation:

```rs
fn internal_redistribute_debt_and_coll(debt: u64, coll: u64) {
    // not really interesting
    let asset_numerator: U128 = U128::from(coll) * U128::from(DECIMAL_PRECISION) + U128::from(storage.last_asset_error_redistribution.read());
    let usdf_numerator: U128 = U128::from(debt) * U128::from(DECIMAL_PRECISION) + U128::from(storage.last_usdf_error_redistribution.read());
    let asset_reward_per_unit_staked = asset_numerator / U128::from(storage.total_stakes.read());
    let usdf_reward_per_unit_staked = usdf_numerator / U128::from(storage.total_stakes.read());

    // not really interesting

    storage
        .l_asset
        .write(storage.l_asset.read() + asset_reward_per_unit_staked.as_u64().unwrap());
    storage
        .l_usdf
        .write(storage.l_usdf.read() + usdf_reward_per_unit_staked.as_u64().unwrap());
    // not really interesting
}
```

In cases where `usdf_reward_per_unit_staked != 0` but `asset_reward_per_unit_staked == 0`, `l_usdf` is updated independently. This disparity can occur due to:

1. Significant price differences between the asset and USDF.
2. High `total_stakes` values combined with a higher USDF price compared to the asset.
3. Growth in `last_usdf_error_redistribution`.

During this interim period, pending rewards might need adjustment due to events such as liquidations, withdrawals, or redemptions.

#### Impact

The issue self-corrects once `internal_redistribute_debt_and_coll` is called again, updating `l_asset`. However, during the lag:

1. Users can perform actions (e.g., withdrawals) at a lower debt than intended.
2. Accumulated inaccuracies in the debt-to-collateral ratio can destabilize the USDF price if enough users act during this window.

#### Proposed Solution

Modify `internal_has_pending_rewards` to also evaluate changes in l\_usdf:

```rs
fn internal_has_pending_rewards(address: Identity) -> bool {
    if (storage.troves.get(address).read().status != Status::Active)
    {
        return false;
    }
    return (storage.reward_snapshots.get(address).read().asset < storage.l_asset.read() || storage.reward_snapshots.get(address).read().usdf_debt < storage.l_usdf.read());
}
```

### Proof of Concept

### Proof of Concept

Run `forc test` after applying the following steps:

1. Add the following methods to the `trove_manager_interface`:

```rs
    #[storage(read, write)]
    fn test_internal_redistribute_debt_and_coll(debt: u64, coll: u64);

    #[storage(read, write)]
    fn test_issue_setup(total_stake: u64);

    #[storage(read)]
    fn test_get_l_values() -> (u64, u64);
```

2. Apply the git patch below:

```diff
diff --git a/contracts/trove-manager-contract/src/main.sw b/contracts/trove-manager-contract/src/main.sw
index 7e02245..6b93a1e 100644
--- a/contracts/trove-manager-contract/src/main.sw
+++ b/contracts/trove-manager-contract/src/main.sw
@@ -291,6 +291,23 @@ impl TroveManager for Contract {
     fn get_pending_asset_rewards(id: Identity) -> u64 {
         internal_get_pending_asset_reward(id)
     }
+
+    #[storage(read, write)]
+    fn test_internal_redistribute_debt_and_coll(debt: u64, coll: u64) {
+        internal_redistribute_debt_and_coll(debt, coll);
+    }
+
+    #[storage(read, write)]
+    fn test_issue_setup(total_stake: u64) {
+        storage
+        .total_stakes
+        .write(total_stake);
+    }
+
+    #[storage(read)]
+    fn test_get_l_values() -> (u64, u64) {
+        (storage.l_asset.read(), storage.l_usdf.read())
+    }
 }
 #[storage(read, write)]
 fn internal_update_trove_reward_snapshots(id: Identity) {
@@ -676,7 +693,7 @@ fn internal_apply_liquidation(
 }
 #[storage(read, write)]
 fn internal_redistribute_debt_and_coll(debt: u64, coll: u64) {
-    let asset_contract_cache = storage.asset_contract.read();
+    // let asset_contract_cache = storage.asset_contract.read();
     if (debt == 0) {
         return;
     }
@@ -704,11 +721,11 @@ fn internal_redistribute_debt_and_coll(debt: u64, coll: u64) {
     storage
         .l_usdf
         .write(storage.l_usdf.read() + usdf_reward_per_unit_staked.as_u64().unwrap());
-    let active_pool = abi(ActivePool, storage.active_pool_contract.read().into());
-    let default_pool = abi(DefaultPool, storage.default_pool_contract.read().into());
-    active_pool.decrease_usdf_debt(debt, asset_contract_cache);
-    default_pool.increase_usdf_debt(debt, asset_contract_cache);
-    active_pool.send_asset_to_default_pool(coll, asset_contract_cache);
+    // let active_pool = abi(ActivePool, storage.active_pool_contract.read().into());
+    // let default_pool = abi(DefaultPool, storage.default_pool_contract.read().into());
+    // active_pool.decrease_usdf_debt(debt, asset_contract_cache);
+    // default_pool.increase_usdf_debt(debt, asset_contract_cache);
+    // active_pool.send_asset_to_default_pool(coll, asset_contract_cache);
 }
 #[storage(read, write)]
 fn internal_update_stake_and_total_stakes(address: Identity) -> u64 {
@@ -979,3 +996,35 @@ fn internal_update_system_snapshots_exclude_coll_remainder(coll_remainder: u64)
         .total_collateral_snapshot
         .write(active_pool_coll - coll_remainder + liquidated_coll);
 }
+
+#[test]
+fn test_issue_1() {
+    /*
+        for the following price, coll and debt, the ICR will be 100% which makes this theoretical position liquidatable
+    */
+    let price = 3_000_000_000_000;
+    let coll = 1_000_000_000;
+    let debt = 3_000_000_000_000;
+    let usdf_in_stab_pool = 0;
+
+    let needed_total_stake = 996_000_000_000_000_000;
+
+    let caller = abi(TroveManager, CONTRACT_ID);
+
+    caller.test_issue_setup(needed_total_stake);
+
+    let l_before = caller.test_get_l_values();
+    let l_asset_before = l_before.0;
+    let l_usdf_before = l_before.1;
+
+    let mut single_liquidation = LiquidationValues::default();
+    single_liquidation = get_offset_and_redistribution_vals(coll, debt, usdf_in_stab_pool, price);
+    caller.test_internal_redistribute_debt_and_coll(single_liquidation.debt_to_redistribute, single_liquidation.coll_to_redistribute);
+
+    let l_after = caller.test_get_l_values();
+    let l_asset_after = l_after.0;
+    let l_usdf_after = l_after.1;
+
+    assert(l_asset_after == l_asset_before); // no change in l_asset
+    assert(l_usdf_after > l_usdf_before); // there is change in l_usdf
+}
\ No newline at end of file
```
