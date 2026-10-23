# #43132 \[BC-Medium] upgrade\_burn\_percentage Resets Block Proposer, Blocking Fee Distribution

**Submitted on Apr 2nd 2025 at 15:14:26 UTC by @jovi for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43132
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**
  * Modification of transaction fees outside of design parameters
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

**Summary**\
A governance transaction that calls `upgrade_burn_percentage` as the very first transaction in a new block causes `process_collected_fees` to reset the block proposer to `None`. As a result, the intended proposer for that block is effectively “penalized” by having no valid proposer set for fee distribution.

***

## Vulnerability Details

### Location

* **`aptos_framework::transaction_fee::sources::transaction_fee`**
  * `upgrade_burn_percentage` function

### Description

When a new block begins, the logical flow is supposed to be:

1. The runtime calls `block_prologue` (or `block_prologue_ext`) to:
   * Distribute fees collected from the previous block (`process_collected_fees`).
   * Register a new proposer for the current block (`register_proposer_for_fee_collection`).
2. Transactions in the new block are then executed.

However, if a governance transaction that calls `upgrade_burn_percentage` runs right after`block_prologue`, it will invoke `process_collected_fees` too early—at a time when the aggregator coin for the new block is still empty (because no transactions have yet happened in this block). The `process_collected_fees` function sees a zero-amount aggregator coin and sets the stored proposer to `None`. This effectively clears the proposer for the new block and results in any subsequent transaction fees in that block being burned or mishandled rather than assigned to the correct proposer.

```move
public fun upgrade_burn_percentage(
    aptos_framework: &signer,
    new_burn_percentage: u8
) acquires AptosCoinCapabilities, CollectedFeesPerBlock {
    ...
    // This call zeroes out the proposer if `amount` is empty.
    process_collected_fees();

    ...
}
```

At the process\_collected\_fees function, notice how it resets the proposer in case there are no collected fees:

```move
public(friend) fun process_collected_fees() acquires AptosCoinCapabilities, CollectedFeesPerBlock {
        ...
        // If there are no collected fees, only unset the proposer. See the rationale for
        // setting proposer to option::none() below.
        if (coin::is_aggregatable_coin_zero(&collected_fees.amount)) {
            if (option::is_some(&collected_fees.proposer)) {
                let _ = option::extract(&mut collected_fees.proposer);
            };
            return
        };
...
```

#### Impact

* The block’s proposer, who should normally collect transaction fees, will lose out on their fees in this edge case.
* All fees from that block could instead be burned, or misassigned, if no valid proposer is found.
* This penalizes proposers and undermines intended incentive mechanisms.
* Since those upgrades are triggered by governance actions, it could be used maliciously to purposefully target specific proposers by setting the proposal acceptance transaction with an elevated gas price.

#### Exploitability

The vulnerability lies on executing upgrade\_burn\_percentage as the first transaction in a block. An attacker could:

* Submit a governance proposal to modify the burn percentage.
* Coordinate its acceptance (e.g., via voting influence or timing manipulation) to align with a target proposer’s block.
* Set an elevated gas price for the acceptance transaction to increase the fees lost by the proposer.

## Proof of Concept

1. **Deploy the modules** `block` and `transaction_fee` with the described logic.
2. **Start a new block** where the first transaction is a governance proposal that calls `upgrade_burn_percentage(..., new_burn_percentage)`.
3. Inside `upgrade_burn_percentage`, `process_collected_fees` detects an empty aggregator coin for the current block and sets `proposer` to `None`.
4. **Subsequent transactions** in that same block incur gas fees, but when `process_collected_fees` is eventually called again (e.g., at the next block prologue), no valid proposer is recognized for the block in which those fees were generated.
5. **Observe** that the rightful proposer never receives fees; the aggregator coin is burned instead.
