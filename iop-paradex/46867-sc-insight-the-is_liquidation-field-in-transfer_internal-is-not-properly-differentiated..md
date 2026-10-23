# #46867 \[SC-Insight] The \`is\_liquidation\` field in \`transfer\_internal\` is not properly differentiated.

**Submitted on Jun 5th 2025 at 14:46:02 UTC by @shaflow1 for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46867
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**

## Description

## Brief/Intro

In the `transfer_internal` function, there exists an `is_liquidation` field designed to distinguish between regular transfers and liquidation transfers. This field triggers events for off-chain monitoring. However, in the current system, all transfers incorrectly have `is_liquidation = 1`, failing to make this critical distinction. As a result, all emitted events are erroneously marked as liquidation transfers.

## Vulnerability Details

```rust
        fn transfer_internal(
            ref self: ComponentState<TContractState>,
            sender: ContractAddress,
            recipient: ContractAddress,
            token_address: ContractAddress,
            amount: felt252,
            is_liquidation: felt252,
        ) {
```

In the `transfer_internal` function, there exists an `is_liquidation` field designed to distinguish between regular transfers and liquidation transfers.

```rust
        fn _transfer(
            ref self: ComponentState<TContractState>,
            sender: ContractAddress,
            recipient: ContractAddress,
            token_address: ContractAddress,
            amount: felt252, // 8位
        ) {
            ...
            // Transfer the amount to recipient
            self.transfer_internal(sender, recipient, token_address, amount, 1);
        }

        fn account_transfer_partial(
            ref self: ContractState,
            account: ContractAddress,
            receiver: ContractAddress,
            account_share: felt252,
            amount_collateral: felt252,
        ) -> felt252 {
                ...
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
                    )
            }
        }
```

However, in non-liquidation transfers such as `account_transfer_partial` and regular transfer operations, the `is_liquidation` field is incorrectly set to 1 as well.

## Impact Details

Since all transfer events have `is_liquidation = 1`, off-chain systems cannot use this field to distinguish between liquidation transfers and regular transfers.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/paraclear/src/paraclear/paraclear.cairo#L1517> <https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/paraclear/src/token/token.cairo#L499>

## Proof of Concept

## Proof of Concept

1. When users transfer assets via the `transfer` function
2. The `TokenAssetBalanceUpdate` event incorrectly marks all transfers with `is_liquidation = 1`, causing off-chain monitoring systems to potentially misinterpret normal transfers as liquidation events
