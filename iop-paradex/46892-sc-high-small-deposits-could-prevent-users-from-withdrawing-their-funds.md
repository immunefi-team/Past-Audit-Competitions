# #46892 \[SC-High] small deposits could prevent users from withdrawing their funds

**Submitted on Jun 5th 2025 at 23:20:32 UTC by @gln for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46892
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

Small deposits could be used to introduce inconsistency between real balance and internal balance of Paradex contract.

## Vulnerability Details

Let's look at the code from paraclear.cairo:

```
     fn deposit(
            ref self: ContractState, token_address: ContractAddress, amount: felt252,
        ) -> felt252 {
            assert!(token_address.is_non_zero(), "Deposit: token_address is zero");
            let recipient = get_caller_address();
            self.account._add_new_account_if_not_exists(recipient);
            self.token._deposit(recipient, recipient, token_address, amount)
        }

```

Implementation of \_deposit function from token.cairo:

```
 fn _deposit(
            ref self: ComponentState<TContractState>,
            sender: ContractAddress,
            recipient: ContractAddress,
            token_address: ContractAddress,
            amount: felt252,
        ) -> felt252 {
            let token_dispatcher = ERC20ABIDispatcher { contract_address: token_address };
            let decimals = token_dispatcher.decimals();

            let contract_address = get_contract_address();
1)            let amount_scaled: u256 = amount.into() / self._scale_factor(decimals);

            let allowance_amount: u256 = token_dispatcher.allowance(sender, contract_address);
            assert!(
                allowance_amount >= amount_scaled,
                "Deposit: Insufficient allowance {allowance_amount}, amount_scaled: {amount_scaled}",
            );

            self._detect_transfer_restriction(sender, recipient, token_address, amount_scaled);

2)            let is_transfer_successful = token_dispatcher
                .transferFrom(sender, contract_address, amount_scaled);
            assert!(is_transfer_successful, "Deposit: Transfer failed");

3)            let updated_balance_amount = self
                .upsert_asset_balance(
                    account: recipient,
                    token_address: token_address,
                    balance_change: amount.try_into().unwrap(),
                );
            self.emit(Deposit { account: recipient, token_address, amount });
            updated_balance_amount.into()
        }

```

1. scale factor is 100, so if 'amount' is less than 100, amount\_scaled will be set to 0
2. transferFrom() does not transfer USDC, as amount\_scaled is zero
3. upsert\_asset\_balance() will change internal balance of account, note it uses original 'amount' value

The accounts' balances stored in Paraclear\_token\_asset\_balance Map.

If a malicious user will deposit a huge number of small deposits, difference between real USDC balance and balance stored in Paraclear\_token\_asset\_balance map will increase.

As a result, some users may not be able to withdraw their funds or their liquidation may fail.

## Impact Details

User may not be able to withdraw their funds from contract.

The attack can be executed against any user by using deposit\_on\_behalf\_of() function.

Attacker may increase internal balance of any user (stored in Paraclear\_token\_asset\_balance map) arbitrarily.

It will introduce inconsistency between real funds stored in USDC contract and internal balance stored in Paradex contract.

## Proof of Concept

## Proof of Concept

Attack works like this:

1. Alice deposits some amount of funds to contract
2. Bob executes attack and withdraws some small amount from contract
3. Now Alice will not be able to withdraw her funds

PoC reproduction:

1. apply patch (see gist link - <https://gist.github.com/gln7/f790708c867cfb13c93f3d9899a914a5> )
2. copy mock\_erc20.cairo to paraclear/src directory
3. run poc

```
$ cd src/paraclear
$ snforge test test_small_deposit_issue
...
Running 1 test(s) from tests/
XXXXKE Alice after deposit 100000 balance 199999000
XXXXKE Bob balance 450
XXXXKE Alice tries to withdraw, balance before 199999000, account value 100000
[FAIL] paradex_paraclear::paraclear::tests::test_paraclear::test_small_deposit_issue

Failure data:
    0x496e73756666696369656e742062616c616e6365 ('Insufficient balance')

```

As you can see, Alice tries to withdraw her funds and transaction reverts.
