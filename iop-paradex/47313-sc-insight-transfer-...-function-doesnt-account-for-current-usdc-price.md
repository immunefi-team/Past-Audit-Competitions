# #47313 \[SC-Insight] Transfer(...) function doesn't account for current USDC price

**Submitted on Jun 12th 2025 at 14:46:43 UTC by @Kalogerone for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47313
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The Paraclear contract implements the `transfer(...)` function that allows users to transfer collateral balance to other accounts. However, it doesn't account for the current USDC/collateral price at the moment of the transfer.

## Vulnerability Details

This is the implementation of the function:

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

As we can see, the only assertion after the transfer is that the `free balance` is greater than 0. However, it is possible that for a timestamp that USDC price is $1.1, that `free balance` is greater than 0 but when USDC comes back to $1, then without anything else changing the `free balance` would fall under 0.

This check is done correctly in the `withdraw(...)` function:

```
        fn withdraw(
            ref self: ContractState, token_address: ContractAddress, amount: felt252,
        ) -> felt252 {
            self.token.assert_withdrawals_allowed();

            let caller = get_caller_address();
            let account_state = self._load_account_v2(caller);
            let free_balance: i128 = account_state.free_balance().try_into().unwrap();
            let amount_128: i128 = amount.try_into().unwrap();
            let socialized_loss_factor: i128 = self.getSocializedLossFactor().try_into().unwrap();
            let amount_settlement_full: i128 = mul_128(
                amount_128, account_state.asset_data.settlement_token_price.into(),
            )
                .into();
@>          assert!(
                free_balance >= amount_settlement_full,
                "Withdraw: Insufficient free balance {free_balance}, amount: {amount_settlement_full}",
            );
			...
```

Here it takes the amount of tokens and applies the USDC price at the current time to compare with the `full balance`. This ensures that any price deviation of USDC won't affect the health of the account in the future.

The same should be done with the `transfer(...)` function. The value of the collateral transferred is what matters to a healthy position check.

## Impact Details

Users who want to withdraw more collateral than they are allowed to can wait for the collateral price to rise, transfer the collateral to another account while keeping their initial account "healthy" and withdraw from the second account. Also unsuspected users will see their accounts' free balance become negative as collateral prices go back to normal.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/main/paraclear/src/paraclear/paraclear.cairo#L1315>

<https://github.com/tradeparadex/audit-competition-may-2025/blob/main/paraclear/src/paraclear/paraclear.cairo#L1280>

## Proof of Concept

## Proof of Concept

1. Bob wants to withdraw certain amount of USDC but withdraw function doesn't let him as he doesn't have enough free balance
2. Bob creates another address and `transfers(...)` his required USDC there at a moment where USDC has increased in price
3. Bob withdraws the USDC from his other address
