# 49854 sc critical dex aggregator partial fill token loss

**Submitted on Jul 20th 2025 at 00:39:49 UTC by @Blobism for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #49854
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:** Permanent freezing of funds

## Description

If one of the deposit methods is invoked with partial fill enabled in the DEX swap, some of the user's sent tokens can be left in the contract, and the contract does not attempt to refund them.

## Brief/Intro

DEX aggregators such as 1inch AggregationRouterV6 allow for partial fill, where only some of the provided ERC20 tokens are used in the swap. If a user invokes one of the deposit or "deposit and bridge" methods with partial fill enabled, the contract will not attempt to return any leftover tokens to the user, effectively locking them in the contract.

## Vulnerability Details

DEX aggregators (e.g., 1inch AggregationRouterV6) support partial fills. The vulnerable flow transfers the full user-provided ERC20 amount into the contract, approves the aggregator, then calls the aggregator swap. If the swap partially fills, leftover tokens remain in the contract and are not refunded.

Example excerpt showing the problematic flow:

```solidity
if (useNative) {
    // ...
} else {
    ERC20 depositAsset = desc.srcToken; // Assumes desc.srcToken is ERC20 type
    uint256 depositAmount = desc.amount;

    // Use safeTransferFrom
    depositAsset.safeTransferFrom(msg.sender, address(this), depositAmount);
    // ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    // all user funds moved to contract
    // (the issue lies in the fact that leftovers are not returned)

    // Approve agregator to take tokens from this contract
    depositAsset.safeApprove(address(aggregator), depositAmount);
}

// Perform swap
(supportedAssetAmount,) = aggregator.swap(executor, desc, data);
```

Recommended fix: either refund any remaining ERC20 tokens after the swap, or enforce no partial fill (disallow partial fills via aggregator parameters).

## Impact Details

Users who make use of partial fill with any available DEX are vulnerable to permanently losing their tokens to the contract.

## References

* 1inch quick start (high level): <https://portal.1inch.dev/documentation/apis/swap/classic-swap/quick-start>
* 1inch aggregation introduction: <https://portal.1inch.dev/documentation/contracts/aggregation-protocol/aggregation-introduction>
* 1inch AggregationRouter contract: <https://etherscan.io/address/0x111111125421cA6dc452d289314280a0f8842A65#code>

Note: OKX router may have similar options, but the reporter did not investigate. More details on OKX can be provided if needed.

## Proof of Concept

{% stepper %}
{% step %}

### Trigger deposit with partial fill enabled

User calls `depositOneInch` or `depositAndBridgeOneInch` with partial fill flag set in `AggregationRouterV6.SwapDescription`.
{% endstep %}

{% step %}

### Partial fill occurs

The aggregator performs a partial swap and does not consume the full provided token amount.
{% endstep %}

{% step %}

### Leftover tokens locked

Remaining tokens are left in the contract because the contract does not refund them, effectively locking the funds.
{% endstep %}
{% endstepper %}
