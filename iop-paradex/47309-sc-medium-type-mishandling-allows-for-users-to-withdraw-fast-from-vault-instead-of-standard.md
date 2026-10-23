# #47309 \[SC-Medium] Type mishandling allows for users to withdraw FAST from vault instead of STANDARD

**Submitted on Jun 12th 2025 at 14:43:17 UTC by @Kalogerone for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47309
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The Paraclear contract implements the `get_account_free_balance(...)` function which returns an account's free balance (available balance to open new positions). This value can be negative, especially when an account approaches liquidation threshold. However, the `get_account_free_balance(...)` function converts the integer amount to felt, which is always positive.

## Vulnerability Details

This is the `get_account_free_balance(...)` implementation:

```
        fn get_account_free_balance(self: @ContractState, account: ContractAddress) -> felt252 {
            let account_state = self._load_account_v2(account);
            account_state.free_balance().into()
        }
```

We can notice that it returns a `felt252` type and it calls `free_balance()` which returns `i128` type:

```
        fn free_balance(self: @AccountState) -> i128 {
            self.account_value() - self.margin_requirement(MARGIN_CHECK_INITIAL)
        }
```

If a negative value is returned from `free_balance()`, the way that cairo handles this conversion, the `get_account_free_balance(...)` function will return the `felt252` max - the negative number.

This function is used by the `Vaults` and this type mishandling can mess up the withdrawals.

```
        fn request_withdrawal(ref self: ContractState, shares: u256) {
            let registry_dispatcher = self._get_registry_from_factory();
            let restricted = registry_dispatcher.get_all_vault_withdrawals_paused();
            assert(!restricted, Errors::WITHDRAWALS_RESTRICTED);
			...

            let withdrawal_mode = self.withdrawal_mode.read();
            if withdrawal_mode == WITHDRAWAL_MODE_FAST
                && sub_operators.len() > 0
                && status != VaultStatus::Closed {
                let assets_to_withdraw = self._convert_to_assets(shares_after_profit_share);
@>              let paraclear_free_balance = paraclear_dispatcher
                    .get_account_free_balance(assets_holder);

                let asset = self.asset();
                let erc20_disp = ERC20ABIDispatcher { contract_address: asset };
                let asset_decimals = erc20_disp.decimals();
                let paraclear_decimals: u8 = paraclear_dispatcher.decimals().try_into().unwrap();
                let decimals_diff = paraclear_decimals - asset_decimals;
                let multiplier = MATH::pow(10, decimals_diff.into());
                let assets_paraclear = assets_to_withdraw * multiplier;

@>              if assets_paraclear == 0 || assets_paraclear > paraclear_free_balance.into() {
                    // available assets are less than requested, let's switch to standard mode
                    applied_withdrawal_mode = WITHDRAWAL_MODE_STANDARD;
                } else {
                    applied_withdrawal_mode = WITHDRAWAL_MODE_FAST;
                    self
                        ._grant_transfer_authorization(
                            assets_holder, auxiliary_account, vault_fraction_to_withdraw_paraclear,
                        );
                    // We are passing vault_fraction_to_withdraw_paraclear to grant the transfer,
                    // but given we are passing assets_paraclear the function will transfer
                    // collateral only
                    let partial_transfer = paraclear_dispatcher
                        .account_transfer_partial(
                            assets_holder,
                            auxiliary_account,
                            vault_fraction_to_withdraw_paraclear,
                            assets_paraclear.try_into().unwrap(),
                        );
                    self._revoke_transfer_authorization(assets_holder, auxiliary_account);
                    assert(partial_transfer == 1, Errors::TRANSFER_FAILED);
                }
            }
			...
        }
```

As we can see, during withdrawal the vault checks the free balance of the assets holder. If the account has a negative free balance, the `get_account_free_balance(...)` function will return a huge positive value. This will result in this check to fail `if assets_paraclear == 0 || assets_paraclear > paraclear_free_balance.into()` (since `paraclear_free_balance` will be a huge number) and the withdrawal mode will be set to `WITHDRAWAL_MODE_FAST`, which is not intended. Withdrawing from an account with negative free balance can get the account closer to liquidation.

The transaction will succeed since the `account_transfer_partial(...)` checks that the `excess balance` is greater than 0.

## Impact Details

Users will be able to withdraw fast from a vault with no free balance and possibly get it closer to liquidation.

## References

<https://book.cairo-lang.org/ch02-02-data-types.html#felt-type>

<https://github.com/tradeparadex/audit-competition-may-2025/blob/main/vaults/src/vault/vault.cairo>

<https://github.com/tradeparadex/audit-competition-may-2025/blob/main/paraclear/src/paraclear/paraclear.cairo#L839>

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
