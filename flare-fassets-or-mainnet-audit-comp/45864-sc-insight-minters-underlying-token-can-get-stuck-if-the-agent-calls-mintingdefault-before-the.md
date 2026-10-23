# #45864 \[SC-Insight] Minter's underlying token can get stuck if the agent calls mintingDefault before the minter’s transaction is recorded on the underlying blockchain.

**Submitted on May 21st 2025 at 16:16:19 UTC by @ni8mare for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45864
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/CollateralReservationsFacet.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The minter’s underlying asset can become permanently stuck if the agent invokes `mintingPaymentDefault` before the minter’s transaction is confirmed on the underlying blockchain. This issue arises when the transaction remains in the mempool beyond the payment deadline, even though the network continues to advance in terms of block number and timestamp.

## Vulnerability Details

Once a minter calls `reserveCollateral`, they are required to:

1. Transfer the underlying amount (e.g., XRP) to the agent’s underlying address.
2. Call `executeMinting` with the corresponding payment proof.

This typically must be done **before** the deadline defined by `lastUnderlyingTimestamp` or `lastUnderlyingBlock`. However, the system does allow the minter to execute the minting even after the deadline, as long as the agent hasn’t triggered a default.

If the payment deadline expires and the agent has not received the payment, they are allowed to call `CollateralReservations:mintingPaymentDefault` with a non-payment proof. This action unlocks the agent’s locked collateral and deletes the collateral reservation associated with that minting process.

```
function mintingPaymentDefault(
        IReferencedPaymentNonexistence.Proof calldata _nonPayment,
        uint64 _crtId
    ) internal {
        ...SNIP...
        require(
            _nonPayment.data.responseBody.firstOverflowBlockNumber >
                crt.lastUnderlyingBlock &&
                _nonPayment.data.responseBody.firstOverflowBlockTimestamp >
                crt.lastUnderlyingTimestamp,
            "minting default too early"
        );
        
        ...SNIP...
    }
```

## Impact Details

A minter can lose funds if the underlying blockchain experiences congestion or if validators deprioritise the transaction due to low fees. This situation is particularly relevant on blockchains where transaction inclusion is fee-based and non-deterministic.

Although the **impact** is a **permanent loss of funds for the minter** (making the severity critical), the **likelihood** of such an event occurring on a stable chain like XRP is relatively low-medium. Therefore, based on both factors, the **overall severity is best rated as High( medium likelihood + critical impact = high)**

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/facets/CollateralReservationsFacet.sol#L127>

## Proof of Concept

## Proof of Concept

1. **Alice (the minter)** calls `reserveCollateral`. The payment deadline is set to 15 minutes or 500 blocks.
2. After 14 minutes, Alice decides to execute the minting. She transfers 100 XRP to the agent’s underlying address.
3. Due to network congestion or low gas fees, the Ripple network does not process Alice’s transaction for 2 minutes. The transaction remains in the mempool.
4. The payment deadline passes. The agent, having not received payment, calls `mintingPaymentDefault` to release the locked collateral. This succeeds and deletes the reservation state.
5. Two minutes later, Alice’s XRP transaction is finally processed. The 100 XRP is sent to the agent’s underlying address.
6. Alice attempts to call `executeMinting` with the payment proof, but the transaction reverts because the collateral reservation no longer exists.

At this point, Alice’s 100 XRP is permanently lost. The agent has no obligation under the protocol to return the funds, even though they received them.
