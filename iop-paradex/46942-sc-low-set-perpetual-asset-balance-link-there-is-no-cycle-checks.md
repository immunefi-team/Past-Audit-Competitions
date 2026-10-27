# #46942 \[SC-Low] set perpetual asset balance link there is no cycle checks

**Submitted on Jun 6th 2025 at 14:37:44 UTC by @gln for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46942
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

By calling set\_perpetual\_asset\_balance\_link function admin could create cycles in Paraclear\_perpetual\_asset\_balance list. As a result, any function which loads account state from storage will revert.

## Vulnerability Details

Let's look at how contract handles account loading from storage, code from paraclear.cairo:

```
   fn _load_account_no_asset_data(
            self: @ContractState, account: ContractAddress,
        ) -> AccountState {
            // load fees and referral
            let fee_rates = self.account._get_account_fee_rate(account);
            let referral = self.account.get_account_referral(account);

            // load perpetual markets (both futures and options)
            let (perp_names, perp_balances) = self.perpetual_future._get_account_markets(account);

            // load token markets
            let (mut token_names, mut token_balances) = self.token._get_account_markets(account);
             ...
 }
 fn _load_account_v2(self: @ContractState, account: ContractAddress) -> AccountState {
            let mut account_state = self._load_account_no_asset_data(account);
            if account_state.perpetual_names.is_empty() && account_state.token_names.is_empty() {
                return account_state;
            }
            account_state
                .asset_data = self
                ._load_asset_data(account_state.perpetual_names, account_state.token_names);
            account_state
        }


```

Now look at the function \_get\_account\_markets from future.cairo:

```
       fn _get_account_markets(
            self: @ComponentState<TContractState>, account: ContractAddress,
        ) -> (Array<felt252>, Array<@PerpetualAssetBalance>) {
            let mut markets = ArrayTrait::new();
            let mut balances = ArrayTrait::new();
            let start_market = self.Paraclear_perpetual_asset_balance_tail.read(account);
            if start_market.is_zero() {
                return (markets, balances);
            }

            let mut balance = self.Paraclear_perpetual_asset_balance.read((account, start_market));
            markets.append(start_market);
            balances.append(@balance);

1)          while balance.prev.is_non_zero() {
                let market = balance.prev;
                balance = self.Paraclear_perpetual_asset_balance.read((account, market));
                markets.append(market);
                balances.append(@balance);
            }

            (markets, balances)
        }

```

1. If we manage to create cycle here, this function will always revert due to out of gas error

The only function which is capable of createing such loop is set\_perpetual\_asset\_balance\_link() :

```
      fn set_perpetual_asset_balance_link(
            ref self: ComponentState<TContractState>,
            account: ContractAddress,
            market: felt252,
            prev_market: felt252,
            next_market: felt252,
        ) {
            self.assert_only_role(roles::ADMIN_ROLE);
            let prev_perpetual_asset = self.Paraclear_perpetual_asset.read(prev_market);
            assert!(prev_perpetual_asset.market != 0, "Can't set asset link to an unset asset");
            let current_perpetual_asset = self.Paraclear_perpetual_asset.read(market);
            assert!(current_perpetual_asset.market != 0, "Can't set asset link of an unset asset");
            self.Paraclear_perpetual_asset_balance.entry((account, market)).prev.write(prev_market);
            self.Paraclear_perpetual_asset_balance.entry((account, market)).next.write(next_market);
        }


```

There is no cycle check, such that prev\_market could equal to market.

Malicious or compromised admin could create cycle by calling this function.

As a result any contract functions which load account data from storage will fail with out of gas error.

## Impact Details

After creating cycle in Paraclear\_perpetual\_asset\_balance list, Paradex contract becomes unusable.

## Proof of Concept

## Proof of Concept

How to reproduce:

1. Add test to paraclear/tests/test\_paraclear.cairo.

See gist link - <https://gist.github.com/gln7/871b2e0d4cdc04a69718923ca332c7a9>

2. Run the test:

```
$ snforge test test_set_perpetual_asset_balance_link_issue

Running 1 test(s) from tests/
Creating a cycle...
[FAIL] paradex_paraclear::paraclear::tests::test_paraclear::test_set_perpetual_asset_balance_link_issue

Failure data:
    Got an exception while executing a hint: Hint Error: Error at pc=0:71856:
Could not reach the end of the program. RunResources has no remaining steps.

```
