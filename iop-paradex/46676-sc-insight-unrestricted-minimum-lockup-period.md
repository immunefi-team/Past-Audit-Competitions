# #46676 \[SC-Insight] Unrestricted Minimum Lockup Period

**Submitted on Jun 3rd 2025 at 08:55:14 UTC by @Catchme for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46676
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/vaults>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The documentation specifies that the lockup period should be between 1 and 4 days. However, the contract only restricts the maximum lockup period during initialization and does not enforce a minimum lockup period. Additionally, when updating the lockup period, there is no enforcement of the minimum lockup period, allowing the owner to bypass the lockup constraints.

## Vulnerability Details

According to the official documentation, the lockup period should be within the range of 1 to 4 days. Nevertheless, the contract's `deploy_vault` function only limits the maximum lockup period during initialization and fails to impose a minimum lockup period. Furthermore, the `set_lockup_period_seconds` function, which is used to update the lockup period, does not include any restrictions on the minimum lockup period. This oversight enables the owner to circumvent the intended lockup period constraints.

```
 fn deploy_vault(
            ref self: ContractState,
            vault_owner: ContractAddress,
            vault_operator: ContractAddress,
            paraclear: ContractAddress,
            underlying: ContractAddress,
            profit_share_percentage: u128,
            tvl_limit: u128,
            lockup_period_seconds: u128,
        ) -> ContractAddress {
            self._assert_only_owner();
            assert(
                profit_share_percentage <= self.max_profit_share_percentage(),
                Errors::INVALID_PROFIT_SHARE,
            );
            assert(
                lockup_period_seconds <= self.max_lockup_period_seconds(),
                Errors::INVALID_LOCKUP_PERIOD,
            );
         }

        fn set_lockup_period_seconds(ref self: ContractState, new_lockup_period_seconds: u128) {
            let caller = get_caller_address();
            assert(caller == self.owner(), Errors::INVALID_CALLER); 
            assert(
                new_lockup_period_seconds <= self.lockup_period_seconds(),
                Errors::LOCKUP_PERIOD_INCREASE,
            );
            self._set_lockup_period_seconds(new_lockup_period_seconds);

            self
                .emit(
                    VaultLockupPeriodUpdated {
                        caller: caller, lockup_period_seconds: new_lockup_period_seconds,
                    },
                );
        }
```

## Impact Details

The actual lockup period may not align with the expectations outlined in the official documentation.

## Reference Links

* [Paradex Documentation - Vault Parameters](https://docs.paradex.trade/documentation/vaults/key-features)

## Proof of Concept

## Proof of Concept
