# #47316 \[SC-Low] account\_transfer\_partial(...) function doesn't check that receiver has a registered account in the system

**Submitted on Jun 12th 2025 at 14:50:36 UTC by @Kalogerone for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47316
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The Paraclear contract implements the `account_transfer_partial(...)` function that allows partial or full transfers of positions or collateral between accounts. However, there is no check that the `recipient` address has an account in the system.

## Vulnerability Details

This is the implementation of the function:

```
        /// Transfers a portion of an account's positions and collateral to another account
        /// @param account The account to transfer positions from
        /// @param receiver The account to transfer positions to
        /// @param account_share The percentage of positions to transfer (between 0 and 1 in fixed
        /// point format)
        /// @return felt252 1 if successful
        fn account_transfer_partial(
            ref self: ContractState,
            account: ContractAddress,
            receiver: ContractAddress,
            account_share: felt252,
            amount_collateral: felt252,
        ) -> felt252 {
            // Validate account share is between 0 and 1
            assert!(
                account_share.try_into().unwrap() > 0_i128
                    && account_share.try_into().unwrap() <= ONE,
                "AccountTransfer: account_share must be within [1,100000000]",
            );

            // detect transfer restriction
            self
                .token
                ._detect_account_transfer_restriction(account, receiver, account_share.into());

            // Load account state and verify account is healthy
            let account_state = self._load_account_v2(account);
            let excess_balance = account_state.excess_balance(MARGIN_CHECK_MAINTENANCE);
            assert!(excess_balance >= 0_i128, "AccountTransfer: account must be healthy");

            // Get account value and settlement token info
            let account_value = account_state.account_value();

            // Standard transfer mode, % of both collateral and positions
            if amount_collateral == 0 {
                // Transfer each perpetual position
                self._transfer_positions_internal(account_state, receiver, account_share, 0);
                // Transfer proportional collateral
                let token_transfer_share = mul_128(
                    account_value, account_share.try_into().unwrap(),
                );
                let token_transfer = div_128(
                    token_transfer_share, account_state.asset_data.settlement_token_price,
                );
                self
                    .token
                    .transfer_internal(
                        account, receiver, self.getSettlementTokenAsset(), token_transfer.into(), 1,
                    );
                // Fast transfer mode, collateral only
            } else {
                self
                    .token
                    .transfer_internal(
                        account, receiver, self.getSettlementTokenAsset(), amount_collateral, 1,
                    );
            }

            // Emit transfer event
            self
                .emit(
                    AccountComponent::Event::AccountTransfer(
                        AccountTransfer {
                            account: account, receiver: receiver, share: account_share,
                        },
                    ),
                );
            return 1;
        }
```

As we can see, there is no check that the `recipient` address has an account in the system using the `_add_new_account_if_not_exists(...)` function. This should follow the same design as the `transfer(...)` function that correctly checks for it:

```
        fn transfer(
            ref self: ContractState,
            recipient: ContractAddress,
            token_address: ContractAddress,
            amount: felt252,
        ) -> felt252 {
            let sender = get_caller_address();
@>          self.account._add_new_account_if_not_exists(recipient);
            self.token._transfer(sender, recipient, token_address, amount);

            let account_state = self._load_account_v2(sender);
            let free_balance: i128 = account_state.free_balance().try_into().unwrap();

            assert!(free_balance >= 0, "Transfer: Sender is unhealthy after transfer");
            return 1;
        }
```

## Impact Details

Having an account with open positions can be very dangerous to the system as the account can operate normally but it won't appear in many accounting tracking functions like `getSettlementAssetTotalBalance(...)` and some collateral will be missing from this. Also, it's possible that there won't be correct tracking of this account's positions to correctly flag it for liquidation.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/main/paraclear/src/paraclear/paraclear.cairo#L1477>

## Proof of Concept

## Proof of Concept

Follow this scenario:

1. Bob uses `account_transfer_partial(...)` to transfer some of his positions to his other address which isn't registered in the system.
2. His second address becomes unhealthy with those positions, but since it's not a registered account it doesn't get flagged for liquidation.
