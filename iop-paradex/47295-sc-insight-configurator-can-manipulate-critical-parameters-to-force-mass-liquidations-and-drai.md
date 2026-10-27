# #47295 \[SC-Insight] Configurator Can Manipulate Critical Parameters to Force Mass Liquidations and Drain Protocol Funds

**Submitted on Jun 12th 2025 at 11:33:20 UTC by @Catchme for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47295
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The Paraclear protocol grants the `CONFIGURATOR_ROLE` unrestricted power to modify critical financial parameters including margin requirements, liquidation fees, and trading fees without any upper bounds validation. A malicious configurator can exploit this to set extreme parameter values (e.g., 1000% margin requirements, 100% liquidation fees) that instantly make all user positions unhealthy, then liquidate them through the insurance fund to extract maximum penalties and trading fees, effectively draining the entire protocol.

## Vulnerability Details

### 1. **Unrestricted Parameter Control**

The configurator role has unlimited power to modify critical parameters with no upper bounds:

**Liquidation Fee - No Upper Limit:**

```cairo
// paraclear/src/paraclear/paraclear.cairo:427-430
fn setLiquidationFee(ref self: ContractState, fee: felt252) {
    self._assert_only_configurator();
    self.Paraclear_liquidation_fee.write(fee);  // No validation whatsoever!
}
```

**Margin Parameters - No Upper Limit:**

```cairo
// paraclear/src/perpetual/future.cairo:154-167
fn update_perpetual_asset(ref self: ComponentState<TContractState>, new_asset: PerpetualAsset) {
    self.assert_only_role(roles::CONFIGURATOR_ROLE);
    assert!(new_asset.market != 0, "Synthetic: Can't update asset with unset market");
    // No validation on margin_params.imf_base, imf_factor, mmf_factor
    self.Paraclear_perpetual_asset.write(new_asset.market, new_asset);
}
```

**Trading Fees - No Upper Limit:**

```cairo
// paraclear/src/account/account.cairo:290-295
fn set_global_fee_rate(...) {
    let net_fee = maker_fee + taker_fee;
    assert!(
        net_fee.try_into().unwrap() >= 0_i128, 
        "Negative total fee rate is not allowed"  // Only checks non-negative!
    );
}
```

### 2. **Margin Calculation Impact**

The margin requirement calculation directly uses these unvalidated parameters:

```cairo
// paraclear/src/perpetual/future.cairo:792-804
fn future_margin_requirement(
    self: @PerpetualAssetBalance,
    imf_base: i128,           // Directly used without bounds checking
    mmf_factor: i128,
    margin_check_type: felt252,
    mark_price: i128,
) -> i128 {
    let current_value_abs = abs_128(self.get_value(mark_price));
    let margin_fraction = self.future_margin_fraction(imf_base, mmf_factor, margin_check_type);
    mul_128(current_value_abs, margin_fraction)  // Can result in extreme values
}
```

### 3. **Health Check Vulnerability**

Account health is determined by comparing account value to margin requirements:

```cairo
// paraclear/src/account/account.cairo:612-616
fn excess_balance(self: @AccountState, margin_check_type: felt252) -> i128 {
    self.account_value() - self.margin_requirement(margin_check_type)
}
```

When `imf_base` is set to extreme values (e.g., 10x normal), all accounts become immediately liquidatable.

### 4. **Liquidation Penalty Extraction**

The liquidation penalty is calculated as:

```cairo
// paraclear/src/paraclear/paraclear.cairo:1386-1387
let liquidation_fee = self.getLiquidateFee();
let liq_penalty_full = mul_128(margin_requirement, liquidation_fee.try_into().unwrap());
```

With extreme parameters:

* `margin_requirement` = position\_value × 1000% = 10× position value
* `liquidation_fee` = 100%
* `liq_penalty_full` = 10× position value

### 5. **Contradicts Official Documentation**

The [official Paradex documentation](https://docs.paradex.trade/risk/liquidations) states:

* "The Liquidation Fee is set to 70%"
* "Partial Liquidation of unhealthy accounts and attempts to minimise impact on the user's assets"

However, the code allows unlimited fee manipulation, violating these safety guarantees.

## Impact Details

1. **Parameter Manipulation:**
   * Set `imf_base` to 1000% (normal: 1-5%)
   * Set liquidation fee to 100% (documented: 70%)
   * Set trading fees to 50% (normal: 0.1%)
2. **Mass Liquidation Trigger:**
   * All user positions instantly become unhealthy (excess\_balance < 0)
   * Insurance fund can liquidate any account
3. **Profit Extraction:**

   ```
   For a user with $10,000 position:
   - New margin requirement: $10,000 × 1000% = $100,000
   - Liquidation penalty: $100,000 × 100% = $100,000
   - Additional trading fees during settlement: $10,000 × 50% = $5,000
   ```

## References

* [Paradex Liquidation Documentation](https://docs.paradex.trade/risk/liquidations)

## Proof of Concept

## Proof of Concept

## PoC

paraclear/src/paraclear/tests/test\_paraclear\_liquidations.cairo

```rust
use crate::tests::test_utils::{
    ADMIN, CONFIGURATOR, EXECUTOR, INSURANCE_FUND, NO_ROLE_ADDRESS, STATE_PUBLISHER,
    setup_paraclear, setup_paraclear_with_oracle,
};

#[test]
fn test_poc_configurator_manipulates_critical_parameters() {
    let (_, paraclear_dispatcher) = setup_paraclear();
    
    // Check initial liquidation fee (should be 0 in test setup)
    let initial_fee = paraclear_dispatcher.getLiquidateFee();
    assert(initial_fee == 0, 'fee should be 0');
    
    start_cheat_caller_address(paraclear_dispatcher.contract_address, CONFIGURATOR());
    
    // 1. Set extreme liquidation fee (1000% instead of documented 70%)
    // Documentation claims "The Liquidation Fee is set to 70%" but code has NO validation
    let extreme_liquidation_fee = 1000000000; // 1000% in 8-decimal format
    paraclear_dispatcher.setLiquidationFee(extreme_liquidation_fee);
    
    // 2. Test even more extreme values to show lack of bounds checking
    let absurd_fee = 50000000000; // 50000% - completely unreasonable
    paraclear_dispatcher.setLiquidationFee(absurd_fee);
    
    stop_cheat_caller_address(paraclear_dispatcher.contract_address);
    
    // 3. Verify extreme parameters were stored without any validation
    let final_fee = paraclear_dispatcher.getLiquidateFee();
    assert(final_fee == absurd_fee, 'fee set without bounds checking');
    
    // Impact: With 50000% liquidation fee, any liquidation would extract 500x more
    // than the normal fee, enabling complete protocol fund drainage
    // This completely violates the documented guarantee of 70% max fee
}

```
