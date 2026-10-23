# #47370 \[SC-Critical] \`account\_transfer\_partial\` should not be enabled when \`transfer\_registry\_address\` is not configured.

**Submitted on Jun 13th 2025 at 05:12:31 UTC by @shaflow1 for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47370
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When `transfer_registry_address` is not yet configured, `_detect_account_transfer_restriction` will skip the check. However, `_detect_account_transfer_restriction` is the only mechanism for enforcing the `account_transfer_partial` permission. If the check is skipped, it will result in unrestricted fund transfers. Therefore, `_detect_account_transfer_restriction` should revert when `transfer_registry_address` is not configured, instead of skipping the check.

## Vulnerability Details

```rust
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
```

`account_transfer_partial` is used for fund withdrawals and vault closure fund transfers. The only permission control in this function is the `_detect_account_transfer_restriction` function.

```rust
        fn _detect_account_transfer_restriction(
            self: @ComponentState<TContractState>,
            sender: ContractAddress,
            recipient: ContractAddress,
            account_share: u256,
        ) {
            // Read transfer registry address
            let transfer_registry_address = self.Paraclear_transfer_registry.read();

            // If transfer registry is set, check restrictions
            if transfer_registry_address.is_non_zero() {
                let registry_dispatcher = IRegistryDispatcher {
                    contract_address: transfer_registry_address,
                };
                let is_account_transfer_restricted: u8 = registry_dispatcher
                    .detect_account_transfer_restriction(sender, recipient, account_share);

                assert!(
                    is_account_transfer_restricted == 0,
                    "Transfer: Account transfer is not allowed",
                );
            }
        }
```

However, when `transfer_registry_address` is not configured, `_detect_account_transfer_restriction` skips the check, which causes the sole permission control for `account_transfer_partial` to fail. As a result, anyone can transfer others’ positions and funds.

Therefore, when `transfer_registry_address` is not configured, the function should revert instead of skipping the check.

## Impact Details

When `transfer_registry_address` is not configured, anyone can transfer the full amount of another user's funds.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/paraclear/src/token/token.cairo#L322>

## Proof of Concept

## Proof of Concept

1. The contract has not yet enabled the vault and `transfer_registry_address` is not configured.
2. Anyone can use the `account_transfer_partial` function to transfer another user's funds to their own account, thereby stealing funds.
