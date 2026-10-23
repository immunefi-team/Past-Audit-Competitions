# #45685 \[SC-Insight] Incorrect comments in finishRedemptionWithoutPayment

**Submitted on May 19th 2025 at 06:08:05 UTC by @ni8mare for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45685
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/RedemptionDefaultsFacet.sol>
* **Impacts:**

## Description

## Brief/Intro

The comment in finishRedemptionWithoutPayment is misleading and will confuse the understanding of the code.

## Vulnerability Details

```
/**
     * If the agent hasn't performed the payment, the agent can close the redemption request to free underlying funds.
     * It can be done immediately after the redeemer or agent calls redemptionPaymentDefault,
     * or this method can trigger the default payment without proof, but only after enough time has passed so that
     * attestation proof of non-payment is not available any more.
     * NOTE: may only be called by the owner of the agent vault in the redemption request.
     * @param _proof proof that the attestation query window can not not contain
     *      the payment/non-payment proof anymore
     * @param _redemptionRequestId id of an existing, but already defaulted, redemption request
     */
    function finishRedemptionWithoutPayment(
        IConfirmedBlockHeightExists.Proof calldata _proof,
        uint256 _redemptionRequestId
    )
        external
        nonReentrant
    {
        RedemptionFailures.finishRedemptionWithoutPayment(_proof, _redemptionRequestId.toUint64());
    }
```

`finishRedemptionWithoutPayment` is supposed to be called when both the agent and the redeemer become inactive for 24 hours. The comment here mentioned `It can be done immediately after the redeemer or agent calls redemptionPaymentDefault,` which is wrong, because this function is used when `redemptionPaymentDefault` is not called.

Also, when someone calls `redemptionPaymentDefault`, it will set the redemption status to `DEFAULTED`, and if someone tries to call `finishRedemptionWithoutPayment` after this, then the function will work as a no-op, as it requires the redemption status to be `ACTIVE`

## Impact Details

This should be considered a DOCUMENTATION MISTAKE and should be an insight.

## References

<https://github.com/flare-labs-ltd/fassets/blob/acb82a27b15c56ce9dfbb6dbbd76008da6753c26/contracts/assetManager/facets/RedemptionDefaultsFacet.sol#L64>

## Proof of Concept

## Proof of Concept

Please remove this comment - `It can be done immediately after the redeemer or agent calls redemptionPaymentDefault,` as this comment is misleading.
