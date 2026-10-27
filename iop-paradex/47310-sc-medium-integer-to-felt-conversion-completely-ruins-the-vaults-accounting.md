# #47310 \[SC-Medium] Integer to Felt conversion completely ruins the Vaults accounting

**Submitted on Jun 12th 2025 at 14:44:58 UTC by @Kalogerone for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47310
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The Paraclear contract implements the `getAccountValue(...)` function which returns an account's total value. This value can be negative when an account's losses exceed the collateral. However, the `getAccountValue(...)` function converts the integer amount to felt, which is always positive.

## Vulnerability Details

This is the `getAccountValue(...)` implementation:

```
        fn getAccountValue(self: @ContractState, account: ContractAddress) -> felt252 {
            Self::get_account_value(self, account)
        }

        fn get_account_value(self: @ContractState, account: ContractAddress) -> felt252 {
            let account_state = self._load_account_v2(account);

            account_state.account_value().into()
        }
```

We can notice that it returns a `felt252` type and it calls `account_value()` which returns `i128` type:

```
        fn account_value(self: @AccountState) -> i128 {
            let mut account_value: i128 = self.get_asset_value();
            account_value += self.total_unrealized_pnl();
            account_value
        }
```

If a negative value is returned from `account_value()`, the way that cairo handles this conversion, the `getAccountValue(...)` function will return the `felt252` max - the negative number.

This function is used by the `Vaults` and this type mishandling has serious implications:

1. Users can deposit into liquidated or unhealthy vaults:

During the deposit flow there is this check

```
            if (total_supply > 0) {
                // if vault is liquidated (but was deposited to before), no more deposits are
                // allowed only withdrawals and donations are allowed
@>              assert(!self._is_liquidated(), Errors::VAULT_LIQUIDATED);
            }
```

```
        /// @notice Checks if the vault has been liquidated
        /// @dev A vault is considered liquidated if its account value on Paraclear is <= 0
        /// @return True if the vault has been liquidated, false otherwise
        fn _is_liquidated(self: @ContractState) -> bool {
            let paraclear = self.paraclear();
            let paraclear_dispatcher = IParaclearDispatcher { contract_address: paraclear };
            let assets_holder = self._assets_holder();

@>          let vault_value = paraclear_dispatcher.getAccountValue(assets_holder);
            vault_value.try_into().unwrap() <= 0_i128
        }
```

This function doesn't work correctly, since if the account's value is negative the `vault_value` here will always be positive.

2. Miscalculation of `_total_assets`, a function that is constantly used throughout the vault and an inflated value can cause a huge deflation of shares and users with tiny amount of shares to be allowed to withdraw huge amount of assets.

```
        fn _total_assets(self: @ContractState) -> u256 {
            let paraclear = self.paraclear();
            let paraclear_dispatcher = IParaclearDispatcher { contract_address: paraclear };

            // If the vault is closed all assets are moved from the operator to the vault itself
            let assets_holder = self._assets_holder();
            // Get main vault value
@>          let mut vault_value: u256 = paraclear_dispatcher.getAccountValue(assets_holder).into();

	        ...
	        
            let sub_operators: Array<ContractAddress> = registry_dispatcher
                .get_sub_operators(self.operator());

            // Only enter loop if there are sub-operators
            if sub_operators.len() > 0 {
                let mut i = 0;
                loop {
                    if i >= sub_operators.len() {
                        break;
                    }
                    let sub_operator = sub_operators.at(i);
@>                  let sub_operator_value: u256 = paraclear_dispatcher
                        .getAccountValue(*sub_operator)
                        .into();
                    vault_value += sub_operator_value;
                    i += 1;
                };
            }

            self._convert_value_to_usdc(vault_value)
        }
```

As we can see, the call to `getAccountValue(...)` is used multiple times in this function, once for the assets holder and multiple times depending on the amount of sub-operators. If any of those accounts has a negative account value, the `getAccountValue(...)` will return a huge positive value and will cause the `_total_assets` value to be huge.

## Impact Details

The `_total_assets` function is used is important accounting tracking functions like `_convert_to_shares(...)` and `_convert_to_assets(...)`. A huge total assets value will cause a huge deflation of shares, where 1 share will equal to a huge amount of assets. This will mess up the accounting of the vault when converting shares to assets and assets to shares during deposits and withdrawals.

## References

<https://book.cairo-lang.org/ch02-02-data-types.html#felt-type>

<https://github.com/tradeparadex/audit-competition-may-2025/blob/main/vaults/src/vault/vault.cairo>

<https://github.com/tradeparadex/audit-competition-may-2025/blob/main/paraclear/src/paraclear/paraclear.cairo#L820>

## Proof of Concept

## Proof of Concept

Using the following test we can see in the console that:

```
Original i128: -42
Converted felt252: 3618502788666131213697322783095070105623107215331596699973092056135872020439
```

```
#[test]
fn i128_to_felt252() {
    // Set an i128 variable to a negative value
    let negative_i128: i128 = -42_i128;

    // Verify it's negative
    assert!(negative_i128 < 0, "i128 should be negative");

    // Convert to felt252
    let as_felt252: felt252 = negative_i128.into();

    println!("Original i128: {}", negative_i128);
    println!("Converted felt252: {}", as_felt252);
}
```
