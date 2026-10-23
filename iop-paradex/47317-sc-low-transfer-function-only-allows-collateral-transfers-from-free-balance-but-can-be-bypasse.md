# #47317 \[SC-Low] Transfer function only allows collateral transfers from free balance but can be bypassed

**Submitted on Jun 12th 2025 at 14:52:23 UTC by @Kalogerone for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47317
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The Paraclear contract implements the `transfer(...)` function that allows users to transfer collateral balance to other accounts. This function requires the sender to have a positive `free balance` after the transfer is completed. However, there is also the `account_transfer_partial(...)` which can also just transfer collateral balance to another account, but only requires the sender to have a positive `excess balance`, which is a lower margin and closer to liquidation than the `free balance`.

## Vulnerability Details

This is the implementation of the functions:

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
@>          assert!(excess_balance >= 0_i128, "AccountTransfer: account must be healthy");

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

```
        fn transfer(
            ref self: ContractState,
            recipient: ContractAddress,
            token_address: ContractAddress,
            amount: felt252,
        ) -> felt252 {
            let sender = get_caller_address();
            self.account._add_new_account_if_not_exists(recipient);
            self.token._transfer(sender, recipient, token_address, amount);

            let account_state = self._load_account_v2(sender);
            let free_balance: i128 = account_state.free_balance().try_into().unwrap();

@>          assert!(free_balance >= 0, "Transfer: Sender is unhealthy after transfer");
            return 1;
        }
```

Users can just bypass the `transfer(...)` function restriction by using the `account_transfer_partial(...)` function and just setting a value for the `amount_collateral` input parameter.

## Impact Details

The `transfer(...)` function has a pointless check that can be easily bypassed by using the `account_transfer_partial(...)` function and allow users to send collateral balance even with a negative `free balance`

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/main/paraclear/src/paraclear/paraclear.cairo#L1477>

<https://github.com/tradeparadex/audit-competition-may-2025/blob/main/paraclear/src/paraclear/paraclear.cairo#L1315>

## Proof of Concept

## Proof of Concept

Follow this scenario:

1. Bob wants to transfer some collateral balance to another account.
2. Bob calls the `transfer(...)` function but it reverts because he doesn't have positive `free balance`.
3. Bob calls the `account_transfer_partial(...)` function and succeeds since he doesn't need positive `free balance` to do it, just positive `excess balance`.
