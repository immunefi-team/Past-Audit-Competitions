# #47198 \[SC-Critical] The operator can perform unauthorized fund transfers.

**Submitted on Jun 10th 2025 at 03:35:03 UTC by @shaflow1 for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47198
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/registry>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The root cause is that a single address can simultaneously act as the operator of one vault and the sub-operator of another. This allows the operator to bypass transfer restrictions in `detect_transfer_restriction` and move funds between different vaults, enabling the theft of user assets.

## Vulnerability Details

```rust
        fn register_sub_operator(
            ref self: ContractState,
            vault: ContractAddress,
            sub_operator: ContractAddress,
            nonce: felt252,
            expiry: u64,
            signature: Array<felt252>,
        ) {
            let caller: ContractAddress = get_caller_address();
            let vault_operator: ContractAddress = self.get_operator(vault);
            assert!(caller == vault_operator, "Caller is not vault operator");

            assert!(starknet::get_block_timestamp() < expiry, "Expired sub-operator signature");

            self.nonces.use_checked_nonce(sub_operator, nonce);

            // Build hash for calling `is_valid_signature`
            let message = SubOperatorRegistrationMessage {
                vault, sub_operator, nonce, expiry: expiry.try_into().unwrap(),
            };
            let hash = message.get_message_hash(sub_operator);

            let is_valid_signature_felt = ISRC6Dispatcher { contract_address: sub_operator }
                .is_valid_signature(hash, signature);

            // Check either 'VALID' or True for backwards compatibility for legacy accounts before
            // SNIP-6
            let is_valid_signature = is_valid_signature_felt == starknet::VALIDATED
                || is_valid_signature_felt == 1;
            assert!(is_valid_signature, "Invalid sub-operator signature");

            // Get current number of sub-operators for this operator
            let current_len: u8 = self.operator_sub_operators_len.read(vault_operator);

            // Check if sub_operator is already registered 
            let other_vault_operator = self.get_parent_operator(sub_operator);
            // Re-emit event if sub-operator is already registered
            // This is to ensure replayability of events if downstream services fail to save mapping
            if other_vault_operator == vault_operator {
                self
                    .emit(
                        SubOperatorRegistered {
                            vault: vault, operator: vault_operator, sub_operator: sub_operator,
                        },
                    );
                return;
            }
            assert!(other_vault_operator.is_zero(), "Sub-operator already registered to a vault");

            // Assert that we haven't reached the maximum number of sub-operators
            let max_sub_operators: u8 = self.max_sub_operators.read();
            assert!(current_len < max_sub_operators, "Max sub-operators reached");

            // Add sub-operator to the list
            self.operator_to_sub_operators.write((vault_operator, current_len), sub_operator);

            // Increment the count of sub-operators for this operator
            self.operator_sub_operators_len.write(vault_operator, current_len + 1);

            // Maintain reverse mapping
            self.sub_operator_to_operator.write(sub_operator, vault_operator); 
            ...
```

The `register_sub_operator` function allows a sub-operator to be registered for a vault without requiring the contract owner's approval—only the signatures of the vault’s operator and the sub-operator to be registered are needed.

Furthermore, the protocol fails to check whether the registering operator is already a sub-operator of another vault. This oversight enables a malicious operator to register themselves as a sub-operator for their own separate vault. As a result, when transferring funds, the transfer restriction check can be bypassed via the following logic:

```rust
if sender_is_sub_operator
    && recipient_is_operator
    && self.sub_operator_to_operator.read(sender) == recipient {
    return TransferRestriction::NoRestriction.into();
}

// operator -> sub-operator : allowed
if sender_is_operator
    && recipient_is_sub_operator
    && self.sub_operator_to_operator.read(recipient) == sender {
    return TransferRestriction::NoRestriction.into();
}
```

This results in the operator's assets being double-counted and reused across both vaults. And this allows unauthorized fund transfers across vaults under the control of a single malicious actor, enabling theft of user assets.

## Impact Details

User funds in the vault are not protected by `detect_transfer_restriction`. A malicious operator can freely steal user assets without restriction.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/registry/src/registry.cairo#L291>

## Proof of Concept

## Proof of Concept

1. There exists a `vault1` with `operator = addr1`, and it holds **10,000 USDC** in assets.
2. The malicious operator wants to steal the funds from `vault1`, so they create a new `vault2` with `operator = addr2`.
3. The attacker deposits into `vault2`, minting 100% of the shares, and immediately calls `set_tvl_limit(0)` to prevent any further deposits, ensuring they remain the sole user of `vault2`.
4. `addr2` (operator of `vault2`) then calls `register_sub_operator` to register `addr1` (operator of `vault1`) as a **sub\_operator** of `vault2`. This causes the positions and assets managed by `addr1` to be counted as part of `vault2`’s total assets.
5. addr1 can call transfer to move all positions and assets to addr2 (which qualifies as a sub-operator → operator transfer), effectively transferring all of vault1’s funds into vault2.

```rust
            if sender_is_sub_operator
                && recipient_is_operator
                && self.sub_operator_to_operator.read(sender) == recipient {
                return TransferRestriction::NoRestriction.into();
            }
```

Or the `vault2` owner now calls `request_withdrawal`. Since they own 100% of the shares, they are entitled to withdraw **all assets in `vault2`**, which now includes the full balance (10,000 USDC) from `vault1`.

6. As a result, **all funds in `vault1` are drained**, despite `vault1` users having no interaction with `vault2`.
