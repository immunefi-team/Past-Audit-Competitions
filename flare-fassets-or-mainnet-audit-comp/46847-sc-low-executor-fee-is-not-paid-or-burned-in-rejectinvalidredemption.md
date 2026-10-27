# #46847 \[SC-Low] executor fee is not paid or burned in \`rejectInvalidRedemption\`

**Submitted on Jun 5th 2025 at 08:47:16 UTC by @pseudoArtist for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46847
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/RedemptionRequests.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

When calling `rejectInvalidRedemption` the function doens't handle the executor fee correctly and doesn't call `payOrBurnExecutorFee` which will cause the fee to be stuck in the contract.

## Vulnerability Details

The function `payOrBurnExecutorFee` is called in function `rejectRedemptionRequest` which transfers the executor fee to the executor if the caller is the executor and burns the fee if it is the agent.

```solidity
    function payOrBurnExecutorFee(
        Redemption.Request storage _request
    )
        internal
    {
        uint256 executorFeeNatWei = _request.executorFeeNatGWei * Conversion.GWEI;
        if (executorFeeNatWei > 0) {
            _request.executorFeeNatGWei = 0;
            if (msg.sender == _request.executor) {
                Transfers.transferNAT(_request.executor, executorFeeNatWei);
            } else {
                Agents.burnDirectNAT(executorFeeNatWei);
            }
        }
    }
```

However the same call is not done in the function `rejectInvalidRedemption` , and later the requestID is deleted.

## Impact Details

This causes 2 problems

1. Executor fee is not burnt if the caller is agent and executors are not incentivised if they are meant to call the function.
2. The executor fee will will remain stuck in the contract as their is no function to withdraw the native tokens recieved as fee.

## References

## Proof of Concept

## Proof of Concept

Step 1: `RedemptionRequestsFacet.redeem()` is called with some executor fee.

Step 2: The function calls `RedemptionRequests.redeem()` and creates a new redemption request.

Step 3: If the address is invalid or not normalised agent calls `rejectInvalidRedemption()` and deletes redemption request without properly calling `payOrBurnExecutorFee`.

Step 4: The fee will be stuck in the contract with no way to get it.
