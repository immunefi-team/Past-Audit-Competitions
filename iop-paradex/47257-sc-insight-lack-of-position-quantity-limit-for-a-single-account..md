# #47257 \[SC-Insight] Lack of position quantity limit for a single account.

**Submitted on Jun 11th 2025 at 15:10:01 UTC by @shaflow1 for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47257
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

The protocol does not currently impose a limit on the number of positions an individual account can hold. As the type of futures and options has already exceeded 100 and is expected to grow further in the app, it is necessary to impose a limit on the number of positions per account to prevent potential DoS issues caused by excessive loop iterations.

## Vulnerability Details

```rust
        fn _create_asset_balance(
            ref self: ComponentState<TContractState>,
            account: ContractAddress,
            market: felt252,
            amount: felt252,
            cost: felt252,
            current_funding: felt252,
        ) -> PerpetualAssetBalance {
            // Get current tail of synthetic asset balances
            let tail_market = self.Paraclear_perpetual_asset_balance_tail.read(account);
            // Create balance for new market
            let new_balance = PerpetualAssetBalance {
                market: market,
                amount: amount,
                cost: cost,
                cached_funding: current_funding,
                prev: tail_market,
                next: 0 // UNSET in Cairo 1
            };

            // Write balance to storage
            self.Paraclear_perpetual_asset_balance.write((account, market), new_balance);

            // Set new tail
            self.Paraclear_perpetual_asset_balance_tail.write(account, market);

            if tail_market.is_non_zero() {
                // Write updated tail balance to storage
                self
                    .Paraclear_perpetual_asset_balance
                    .entry((account, tail_market))
                    .next
                    .write(market);
            }

            new_balance
        }
```

A position linked list is maintained for each account, where each type of position held by the account occupies one node in the list.

When executing a trade, we need to `load_account`, iterate through the linked list to construct the `account_state`, and also loop through to fetch the corresponding market mark prices and asset prices.

There is no restriction on the maximum number of positions a single account can hold. This makes it easy for certain system functions to encounter DoS issues due to excessive iteration.

## Impact Details

Certain system functions may suffer from DoS due to excessive iteration over account positions.

The most likely scenario is that a malicious operator, together with sub-operators, opens an excessive number of positions to prevent users from withdrawing assets from the vault, since deposits into the vault do require traversing positions is more less withdrawals.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/vaults/src/vault/vault.cairo#L697> <https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/vaults/src/vault/vault.cairo#L1048>

## Proof of Concept

## Proof of Concept

Functions that require iterating over positions about vault function include:

* `deposit`: calls `_convert_to_shares` once and `_is_vault_healthy` once.
* `withdraw`: calls `_convert_to_shares` once, `_convert_to_assets` once, and for each operator and sub-operator, calls `account_transfer_partial`. Each `account_transfer_partial` in turn calls `_load_account_v2`, `excess_balance`, and `_transfer_positions_internal`, each of which iterates over positions.

1. There are 100 types of positions in the market (futures and option assets).
2. In Vault 1, a malicious operator intends to prevent users from withdrawing funds.
3. The malicious operator opens 100 positions—most of which can be minimal—and the sub-operator does the same.
4. In this case, the number of iterations during a withdraw operation could exceed: **100 \* (1 + 1 + 4 \* (1 + 1 + 1)) = 1400 iterations.**
