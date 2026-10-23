# #46839 \[SC-Low] \`max\_withdraw\` and \`max\_withdraw\` do not fully consider global restrictions.

**Submitted on Jun 5th 2025 at 07:49:32 UTC by @shaflow1 for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46839
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/vaults>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The Vault protocol includes `max_withdraw` and `max_deposit` functions, which can return the maximum amount currently available for deposit or withdrawal, similar to the implementation in ERC4626. However, these functions do not fully account for global restrictions, which may result in incorrect return values under certain conditions.

## Vulnerability Details

```rust
        fn max_deposit(self: @ContractState, receiver: ContractAddress) -> u256 {
            let status = self.status.read();
            if (status == VaultStatus::Closed) {
                return 0;
            }

            if (!self._is_vault_healthy()) {
                return 0;
            }

            let tvl_limit = self.tvl_limit.read();
            if (tvl_limit == 0) {
                return Bounded::<u256>::MAX;
            }

            let current_assets = self._total_assets();
            if (current_assets >= tvl_limit.into()) {
                return 0;
            }

            tvl_limit.into() - current_assets
        }
```

For the `max_deposit` function, it does not consider the `owner`'s `shares` status. If the caller is not the `owner` and the `owner`'s `shares` equal zero, the caller is not permitted to deposit. Therefore, `max_deposit` should return `0` in this case.

```rust
        fn max_withdraw(self: @ContractState, receiver: ContractAddress) -> u256 {
            if (!self._is_vault_healthy()) {
                return 0;
            }
            let shares = self.erc20.balance_of(receiver);
            let assets = self._convert_to_assets(shares);

            let status = self.status.read();
            if (status != VaultStatus::Closed) {
                let average_deposit_time = self.asset_weighted_time.read(receiver);
                let current_time = get_block_timestamp();
                let holding_time = current_time.into() - average_deposit_time;
                let min_lockup = self.lockup_period_seconds().try_into().unwrap();
                if (holding_time < min_lockup) {
                    return 0;
                }
            }
            assets
        }
```

For the `max_withdraw` function, it does not check whether the `registry` has paused withdrawals. If withdrawals are paused, the function should return `0` since no withdrawals are allowed in this case. Moreover, it fails to account for the existence of pending orders of operator/sub operator, which could cause the transfer to revert.

## Impact Details

When called by users or integrated with other projects, it may return misleading values and does not comply with EIP-4626.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/vaults/src/vault/vault.cairo#L834> <https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/vaults/src/vault/vault.cairo#L811> <https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/vaults/src/vault/vault.cairo#L584> <https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/vaults/src/vault/vault.cairo#L487>

## Proof of Concept

## Proof of Concept

For example:

1. The registry has enabled all\_vault\_withdrawals\_paused.
2. When a user calls max\_withdraw, the function fails to consider this restriction and does not return 0 as required. Users may be misled during interactions.
