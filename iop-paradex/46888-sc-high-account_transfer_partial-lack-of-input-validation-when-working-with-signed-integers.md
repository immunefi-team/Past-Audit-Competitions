# #46888 \[SC-High] account\_transfer\_partial: lack of input validation when working with signed integers

**Submitted on Jun 5th 2025 at 22:12:47 UTC by @gln for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46888
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The function account\_transfer\_partial() fails to validate signed integer parameter which represents an amount to transfer.

As a result, malicious user is able to steal funds from the contract.

## Vulnerability Details

Let's look at the code from paraclear.cairo:

```
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
1)                self
                    .token
                    .transfer_internal(
                        account, receiver, self.getSettlementTokenAsset(), amount_collateral, 1,
                    );
            }
         ...
     }
```

1. We asuume that amount\_collateral is not zero, so transfer\_internal() will be called.

Also note, that there is no any validations of amount\_collateral parameter (it is felt252).

Now let's look at transfer\_internal function from token.cairo:

```
      fn transfer_internal(
            ref self: ComponentState<TContractState>,
            sender: ContractAddress,
            recipient: ContractAddress,
            token_address: ContractAddress,
            amount: felt252,
            is_liquidation: felt252,
        ) {
            let sender_balance = self.Paraclear_token_asset_balance.read((sender, token_address));
            let recipient_balance = self
                .Paraclear_token_asset_balance
                .read((recipient, token_address));

            let sender_balance_amount = sender_balance.amount;

            // Ensure sender has enough balance
            let sender_balance_amount_i128: i128 = sender_balance_amount.try_into().unwrap();
1)            let amount_i128: i128 = amount.try_into().unwrap();
2)            assert!(
                sender_balance_amount_i128 >= amount_i128,
                "Transfer: Sender balance is insufficient",
            );

		    let sender_change_amount = -amount_i128;
            let recipient_change_amount = amount_i128;

3)          let updated_sender_balance_amount = self
                .upsert_asset_balance(sender, token_address, sender_change_amount);
            self
                .emit(
                    TokenAssetBalanceUpdate {
                        account: sender,
                        token_address: token_address,
                        prev_amount: sender_balance_amount,
                        updated_amount: updated_sender_balance_amount.into(),
                        is_liquidation: is_liquidation,
                    },
                );

            let updated_recipient_balance_amount = self
                .upsert_asset_balance(recipient, token_address, recipient_change_amount);
		...
	}
```

1. Note that if 'amount' is signed integer (i128)
2. If 'amount' is negative, this check could be easily bypassed
3. Sender balance is updated, so if 'amount' is very small negative value, sender balance will be increased !

As a result, sender will be able to increase balance to arbitrary value and later drain all contract's funds.

## Impact Details

Attacker will be able to drain all funds from the contract.

## Proof of Concept

## Proof of Concept

How to reproduce:

In this scenario we assume the Admin deposited some amount of funds to Paradex contract.

Now attack works like this:

1. Alice deposits very small amount of funds to contract
2. Alice calls account\_transfer\_partial() with crafted parameters
3. Now Alice is able withdraw all contract's funds

PoC reproduction:

1. apply patch (see gist link - <https://gist.github.com/gln7/19536922bbd3a87d56557f82f446b9d7> )
2. copy mock\_erc20.cairo to paraclear/src directory
3. run poc

```
$ cd src/paraclear
$ snforge test test_transfer_poc
...
Collected 1 test(s) from paradex_paraclear package
Running 1 test(s) from tests/
XXXXKE Alice after deposit 100000 balance 199999000
XXXXKE total balance of contract 100000001000 <! - initial USDC balance of contract
XXXXKE Alice balance before attack 199999000
XXXXKE Alice balance after attack 100200000000, contract balance 0 <!-- USDC balance after attack

```

As you can see, Alice was able to drain all USDC from contract.
