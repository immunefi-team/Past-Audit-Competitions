# #45897 \[SC-Low] Executor Fee Lost in \`rejectInvalidRedemption()\` Due to Missing Handling Logic

**Submitted on May 22nd 2025 at 06:47:34 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45897
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/RedemptionRequestsFacet.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Description

The `rejectInvalidRedemption()` function is used to cancel redemption requests when the redeemer's underlying address is found to be invalid or not normalized. However, this code path fails to handle the executor fee if it was originally attached to the request, resulting in silent permanent freezing of Executor Fee in the contract.

In typical redemption workflows, the executor fee is either paid to the executor (if they submit the correct proof) or burned. This ensures all fees are properly routed, preventing against the freezing of funds in the contracts. In contrast, when a redemption is rejected due to invalid address validation, the executor fee is neither paid nor burned, it is silently lost when the associated `Redemption.Request` is deleted.

The following snippet shows the affected function:

```solidity
function rejectInvalidRedemption(
    IAddressValidity.Proof calldata _proof,
    uint64 _redemptionRequestId
) internal {
    Redemption.Request storage request = Redemptions.getRedemptionRequest(_redemptionRequestId);
    ...
    Agents.endRedeemingAssets(agent, request.valueAMG, request.poolSelfClose);
    emit IAssetManagerEvents.RedemptionRejected(request.agentVault, request.redeemer,
        _redemptionRequestId, Conversion.convertAmgToUBA(request.valueAMG));
  @>>   Redemptions.deleteRedemptionRequest(_redemptionRequestId);
}
```

Notice that no call to Redemptions.payOrBurnExecutorFee(request) is made before deletion.

In contrast, the correct handling path (e.g., in successful redemptions) explicitly calls:

```solidity
Redemptions.payOrBurnExecutorFee(request);
```

The payOrBurnExecutorFee logic is as follows:

```solidity
function payOrBurnExecutorFee(
    Redemption.Request storage _request
) internal {
    uint256 executorFeeNatWei = _request.executorFeeNatGWei * Conversion.GWEI;
    if (executorFeeNatWei > 0) {
        _request.executorFeeNatGWei = 0;
        if (msg.sender == _request.executor) {
            Transfers.transferNAT(_request.executor, executorFeeNatWei);
        } else {
            Transfers.transferToBurnAddress(executorFeeNatWei);
        }
    }
}
```

## Impact

Silent Fund Loss since the Executor fee is permanently removed from circulation without any payout or burn and cannot be recovered.

## Recommendations

I will recommend for the `payOrBurnExecutorFee()` to be called before permanently deleting the request or better still it the Agents should be given the executor fee as compensation for gas usages.

## Proof of Concept

## Proof of Concept

1. Create Redemption Request, A user initiates a redemption, including an `executorFeeNatGWei` value.
2. Call `rejectInvalidRedemption()`Agent calls the function to reject the redemption due to invalid address.
3. Observe Fee Handling a. `payOrBurnExecutorFee()` was never called. b. `executorFeeNatGWei` is silently lost when c. `Redemptions.deleteRedemptionRequest()` is called.
