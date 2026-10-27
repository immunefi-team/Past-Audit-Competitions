# #47377 \[SC-Insight] No Restriction on Self Transfer

**Submitted on Jun 13th 2025 at 06:38:07 UTC by @Catchme for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47377
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `detect_transfer_restriction()` function in the Registry contract fail to restrict self-transfers. This oversight allows an account to transfer assets to itself, which is typically undesirable and should be restricted for the sake of security and data integrity. The lack of a self-transfer restriction may lead to unintended asset movements and can complicate the tracking and monitoring of user activity.

## Vulnerability Details

In the `detect_transfer_restriction()` function, there is no check to identify whether the `sender` and `recipient` are the same account. This allows an account to perform self-transfers, which is often undesirable or unnecessary, and can be exploited in various attack vectors, such as token manipulation or fund misappropriation.

Self-transfers typically should be restricted or at least closely monitored, as they may bypass certain business rules and cause confusion in transaction logs. This missing check introduces a gap in transaction validation and can lead to data inconsistencies, unexpected behavior, or regulatory concerns.

Vulnerable Code Location

* `detect_transfer_restriction()` function

```rust
// Missing self-transfer check
if sender_is_operator {
    return TransferRestriction::InvalidTransferPair.into();
}
if sender_is_sub_operator {
    return TransferRestriction::InvalidTransferPair.into();
}
if sender_is_auxiliary {
    return TransferRestriction::InvalidTransferPair.into();
}
```

## Impact Details

Unrestricted self-transfers could violate transaction monitoring or regulatory requirements.

## Proof of Concept

## Proof of Concept

```
use crate::tests::test_utils::{RECEIVER, deploy_registry};

#[test]
fn test_detect_transfer_restriction_same_address() {
    let registry = deploy_registry(OWNER(), FACTORY(), PARACLEAR());
    let registry_dispatcher = IRegistryDispatcher {
        contract_address: registry.contract_address,
    };

    let sender: ContractAddress = RECEIVER();

    let invalid_pair: u8 = 1;

    start_cheat_caller_address(registry.contract_address, sender);

    assert_ne!(
        registry_dispatcher.detect_transfer_restriction(sender, sender, 100),
        invalid_pair,
        "same address should be restricted",
    );

    stop_cheat_caller_address(registry.contract_address);
}
```
