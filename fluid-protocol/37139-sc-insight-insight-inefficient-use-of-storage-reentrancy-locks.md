# #37139 \[SC-Insight] insight inefficient use of storage reentrancy locks

## #37139 \[SC-Insight] INSIGHT: Inefficient Use of Storage Reentrancy Locks

**Submitted on Nov 26th 2024 at 18:18:50 UTC by @Blockian for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37139
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/sorted-troves-contract/src/main.sw>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Gas Optimization Insight

### Description

## Fluid Insight Report

### Inefficient Use of Storage Reentrancy Locks

#### Overview

Many contracts within the Fluid protocol are using storage-based reentrancy locks. This practice appears to be inherited from the Liquity codebase, which is written for the Ethereum codebase. However, in the Fuel ecosystem, there is a more efficient alternative for managing reentrancy using Sway Libs, as detailed in the [official documentation](https://docs.fuel.network/docs/sway-libs/reentrancy/).

Additionally, in the Fuel ecosystem, token transfers to an address do not invoke a function on the recipient, even if the recipient is a contract. This behavior contrasts with Ethereum's token transfer mechanics, where a transfer can trigger a contract call. As a result, reentrancy locks are often unnecessary in Fuel-based contracts.

**Example:** The `stake` function in the `fpt-staking-contract` employs a `lock_stake` mechanism. However, since this function never transfers execution to the user, reentrancy is not a concern.

#### Impact

The unnecessary use of reentrancy locks increases gas consumption for users without providing any meaningful benefit, making the protocol less efficient.

#### Proposed Solution

To address this inefficiency:

1. **Remove reentrancy locks entirely** where they are unnecessary.
2. **Adopt the Sway Libs `reentrancy_guard`**, a more gas efficient alternative for preventing reentrancy.

### Proof of Concept

### Proof of Concept

Below is an example demonstrating how to implement the `reentrancy_guard` from Sway Libs effectively:

```rust
contract;
 
use sway_libs::reentrancy::reentrancy_guard;
use sway_libs::reentrancy::is_reentrant;
 
abi MyContract {
    fn my_non_reentrant_function();
}
 
impl MyContract for Contract {
    fn my_non_reentrant_function() {
        reentrancy_guard();
 
        // code here
    }
}

fn check_if_reentrant() {
    assert(!is_reentrant());
}
```

#### Benefits

Switching to the Sway Libs `reentrancy_guard` or eliminating unnecessary locks entirely will:

* Reduce gas costs for users.
* Improve the protocol's overall efficiency.
* Align with best practices in the Fuel ecosystem.

This change is a straightforward yet impactful optimization that enhances both developer and user experience.
