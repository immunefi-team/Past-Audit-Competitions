# #46320 \[SC-Low] Executor fee will be stuck in the contract when rejectInvalidRedemption is called

**Submitted on May 28th 2025 at 11:26:56 UTC by @Oxgritty for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46320
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/RedemptionRequests.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

* When agent vault owner will call `rejectInvalidRedemption`, to reject the redemption request, request will be deleted without burning executor fee.

## Vulnerability Details

* When a redeemer creates a redemption request, he has the option to pay an executor fee with it. This is either paid to the executor, if he calls `confirmRedemptionPayment` with the payment proof or it is burned if agent submits the payment proof.
* The problem lies in `rejectInvalidRedemption` function, which is called by the agent for a redemption request where the redeemer provided a invalid address, here the redemption request is deleted without burning the executor fee.

## Impact Details

* Executor fee will be stuck in the contract, instead of just burning it by sending it to a dead address.

## References

* This function lacks the executor fee burn mechanism: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/RedemptionRequests.sol#L231-#L261>

## Proof of Concept

## Proof of Concept

1. Redeemer calls `redeem` to create a redeem request where he provides invalid underlying redeemer address and also provides executor fees.
2. Agent vault owner calls `rejectInvalidRedemption` with the proof of invalid address to delete the redemption request.
3. Redemption request will be deleted, but executor fee will remain in the contract.

```solidity
// rejectInvalidRedemption function is missing this code:-
            Redemptions.payOrBurnExecutorFee(request);

```
