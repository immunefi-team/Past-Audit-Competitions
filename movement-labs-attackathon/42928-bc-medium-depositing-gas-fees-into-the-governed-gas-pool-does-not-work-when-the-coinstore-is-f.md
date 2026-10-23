# #42928 \[BC-Medium] Depositing gas fees into the governed gas pool does not work when the CoinStore is frozen

**Submitted on Mar 29th 2025 at 14:18:31 UTC by @HollaDieWaldfee for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42928
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

When depositing gas fees into the governed gas pool, it is checked that the `CoinStore` is not frozen. As a result, gas fees cannot be deposited when the `CoinStore` is frozen. However, paying gas fees should still be possible even when the `CoinStore` is frozen.

## Vulnerability Details

`deposit_gas_fee_v2()` deposits the gas fees from the gas payer into the governed gas pool by first withdrawing the gas fees from the `CoinStore`. Inside of this function, it is ensured that the `CoinStore` is not frozen which means that the gas fees cannot be paid when the `CoinStore` is frozen. However, it should be possible to pay for gas fees as `collect_into_aggregatable_coin()`, which is called inside of `collect_fee()`, does not check whether the `CoinStore` is frozen or not. This can be verified by taking a look at reference (1). The same also applies for `burn_fee()`. It's possible to burn transaction fees even when the `CoinStore` is frozen.

It is clear that gas payments should be allowed regardless of the frozen status. The freezing mechanism is meant to restrict arbitrary transfers, not gas payments. Therefore, `withdraw_from()` should be implemented like `collect_into_aggregatable_coin()`.

## Impact Details

It is impossible to pay for gas fees when the `CoinStore` is frozen.

## References

(1): <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/framework/aptos-framework/sources/coin.move#L616-L645>

(2): <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/framework/aptos-framework/sources/governed\\_gas\\_pool.move#L152-L158>

(3): <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/framework/aptos-framework/sources/governed\\_gas\\_pool.move#L115-L117>

(4): <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/framework/aptos-framework/sources/coin.move#L1168-L1208>

(5): <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/framework/aptos-framework/sources/coin.move#L1179-L1182>

## Proof of Concept

## Proof of Concept

1. `deposit_gas_fee_v2()` is called to deposit gas fees into the governed gas pool. (reference (2)).
2. This calls `deposit_from()` to deposit the gas fees from the gas payer to the governed gas pool. (reference (3)).
3. To do so, the gas fees are withdrawn from the gas payer by calling `withdraw_from()` (reference (4)).
4. The call to withdraw the gas fees reverts when the `CoinStore` is frozen (reference (5)).
