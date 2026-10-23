# #46997 \[SC-Medium] The vault performs an unsafe conversion on the getAccountValue result.

**Submitted on Jun 7th 2025 at 15:37:52 UTC by @shaflow1 for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46997
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

In the \_total\_assets function, after obtaining the getAccountValue result, an unsafe type conversion is performed. When the system experiences significant asset price volatility causing one of the vault's operators to fall into bad debt (and before liquidation occur), the vault will convert a negative value to u256, resulting in \_total\_assets becoming abnormally large.

## Vulnerability Details

```rust
        fn getAccountValue(self: @ContractState, account: ContractAddress) -> felt252 {
            Self::get_account_value(self, account)
        }

        fn get_account_value(self: @ContractState, account: ContractAddress) -> felt252 {
            let account_state = self._load_account_v2(account);

            account_state.account_value().into()
        }


        fn _total_assets(self: @ContractState) -> u256 {
            ...
            let mut vault_value: u256 = paraclear_dispatcher.getAccountValue(assets_holder).into();

            // Skip sub-operators account values if vault is closed
            let status = self.status.read();
            if status == VaultStatus::Closed {
                return self._convert_value_to_usdc(vault_value);
            }
            ...
            if sub_operators.len() > 0 {
                let mut i = 0;
                loop {
                    if i >= sub_operators.len() {
                        break;
                    }
                    let sub_operator = sub_operators.at(i);
                    let sub_operator_value: u256 = paraclear_dispatcher
                        .getAccountValue(*sub_operator)
                        .into();
                    vault_value += sub_operator_value;
                    i += 1;
                };
            }
            ...
        }
```

In futures assets, there exist highly volatile meme assets whose prices may fluctuate drastically. Positions could become undercollateralized (bad debt) before liquidation occurs, resulting in negative account values.

At this point, if the getAccountValue function in \_total\_assets directly converts the felt252 type to u256, it will cause \_total\_assets to return an extremely large value.

## Impact Details

1. When users deposit in this abnormal state, since `_total_assets` returns an extremely large value, any amount of USDC deposited will result in 0 shares minted. The deposited tokens will be effectively lost and donated to the vault.
2. If the vault has `WITHDRAWAL_MODE_FAST` enabled or `profit_share_percentage` set, withdrawals will revert due to calculation failures.

The probability of occurrence is low, but it could potentially affect any user deposits, therefore the severity is assessed as Medium.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/vaults/src/vault/vault.cairo#L973>

## Proof of Concept

## Proof of Concept

1. A sub-operator in the vault opens a small long position in a meme asset.
2. A user then submit the transaction of depositing 10,000 USDC into the vault.
3. The meme price plummets, causing the sub-operator's account value to fall below zero (undercollateralized), but liquidation hasn't yet occurred.
4. During the user's deposit transaction execution, \_total\_assets becomes extremely large due to truncation, resulting in the user's shares being rounded down to 0.
5. The user loses their deposited funds.
