# #43135 \[BC-High] \`epilogue\_gas\_payer\` Silently Drops Excess Storage Fee Refunds Under Governed Gas Pool

**Submitted on Apr 2nd 2025 at 15:36:07 UTC by @jovi for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43135
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**
  * Direct loss of funds
  * Modification of transaction fees outside of design parameters

## Description

## Summary

When `governed_gas_pool_enabled()` is active, `epilogue_gas_payer` fails to refund excess storage fees (`storage_fee_refunded - amount_to_burn`), resulting in **silent, unavoidable loss of user funds**.

## Vulnerability Details

### Location

* **File**: `aptos_framework::transaction_validation::sources::transaction_validation.move`
* **Function**: `epilogue_gas_payer`
* **Relevant Code**:

  ```move
  } else if (amount_to_burn < storage_fee_refunded) {
      let mint_amount = storage_fee_refunded - amount_to_burn;
      if (!features::governed_gas_pool_enabled()) {
          transaction_fee::mint_and_refund(gas_payer, mint_amount);
      }
      // No refund when governed_gas_pool_enabled() is true
  };
  ```

### Description

The `epilogue_gas_payer` function finalizes transaction fee accounting. When `storage_fee_refunded` exceeds `amount_to_burn`, the surplus (`mint_amount`) should be returned to the `gas_payer`. This works when `governed_gas_pool_enabled()` is false, but **when true, the surplus is discarded** due to the missing `else` branch. Users lose entitled funds without notification.

## Impact

* **Significant Fund Loss**: Users lose all excess refunds in a widespread fashion, whenever the condition triggers, potentially affecting thousands of transactions .
* **Economic Instability**: Cumulative losses disrupt the fee model, deterring users/dApps that would be expecting refunds.
* **Exploitable Harm**: Attackers may maximize losses via storage-heavy transactions targeting refunds.

## Recommended Mitigation

```move
} else if (amount_to_burn < storage_fee_refunded) {
    let mint_amount = storage_fee_refunded - amount_to_burn;
    if (!features::governed_gas_pool_enabled()) {
        transaction_fee::mint_and_refund(gas_payer, mint_amount);
    } else {
        transaction_fee::mint_and_refund(gas_payer, mint_amount); // Direct refund
        // OR: coin::deposit(governed_gas_pool_address, mint_amount); // Pool redirect
    }
};
```

## Proof of Concept

1. **Setup**: Deploy with `governed_gas_pool_enabled()` set to `true`.
2. **Transaction**: Execute storage cleanup:
   * `storage_fee_refunded = 100 APT`
   * `amount_to_burn = 60 APT`
   * **Expected Refund**: 40 APT (not issued).
3. **Observation**: Toggling the flag to `false` restores refunds.
