# 52178 sc critical user will lose the unspent amount when executing partial swaps via okxrouter

**Submitted on Aug 8th 2025 at 14:13:41 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52178
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief/Intro

In `depositOkxUniversal` and `depositAndBridgeOkxUniversal` from `DexAggregatorWrapperWithPredicateProxy`, unspent funds from a partial swap in the OKX Router are left in the wrapper contract but never deposited into the vault or returned to the user.

### Vulnerability Details

The OKX Router allows partial swaps in functions like `smartSwapByOrderId` and `smartSwapTo` (both allowed to be called from `DexAggregatorWrapperWithPredicateProxy`).

Partial swaps occur when the sum of `batchesAmount` (totalBatchAmount) is less than the `BaseRequest.fromTokenAmount`, as enforced by a <= check in the internal `_smartSwapInternal` function. This design pulls only the specified `totalBatchAmount` from the payer, leaving any unspent input tokens in the payer's balance.

<https://github.com/okxlabs/DEX-Router-EVM-V1/blob/388a4a0f70a78a525824760e18354b8ea5b8324d/contracts/8/DexRouter.sol#L312-L325>

```solidity
// In _smartSwapInternal
uint256 totalBatchAmount;
for (uint256 i = 0; i < batchesAmount.length; ) {
    totalBatchAmount += batchesAmount[i];
    unchecked {
        ++i;
    }
}
require(
    totalBatchAmount <= _baseRequest.fromTokenAmount,
    "Route: number of batches should be <= fromTokenAmount"
);
```

The router then transfers only the `totalBatchAmount` via `_transferInternal` calls in `_exeForks`, using IApproveProxy.claimTokens to pull from the payer (the wrapper contract).

<https://github.com/okxlabs/DEX-Router-EVM-V1/blob/388a4a0f70a78a525824760e18354b8ea5b8324d/contracts/8/DexRouter.sol#L139>

```solidity
    function _transferInternal(
        address payer,
        address to,
        address token,
        uint256 amount
    ) private {
        if (payer == address(this)) {
            SafeERC20.safeTransfer(IERC20(token), to, amount);
        } else {
            IApproveProxy(_APPROVE_PROXY).claimTokens(token, payer, to, amount);
        }
    }
```

Problem is in the `DexAggregatorWrapperWithPredicateProxy`, the `_okxHelper` function (called by `depositOkxUniversal` and `depositAndBridgeOkxUniversal`) transfers the full `fromTokenAmount` from the user to the wrapper contract and approves the OKX approver for the full amount:

<https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/helper/DexAggregatorWrapperWithPredicateProxy.sol#L318-L325>

```solidity
function _okxHelper(
    ERC20 supportedAsset,
    address teller,
    address fromToken,
    uint256 fromTokenAmount,
    bytes calldata okxCallData,
    uint256 nativeValueToWrap
)
    internal
    returns (uint256 supportedAssetAmount)
{
    // ...
    ERC20 depositAsset = ERC20(fromToken);

    // Use safeTransferFrom
    depositAsset.safeTransferFrom(msg.sender, address(this), fromTokenAmount);

    // Use standard approve for the OKX approver
    depositAsset.safeApprove(okxApprover, fromTokenAmount);

    // Execute the swap with the provided calldata
    (bool success, bytes memory result) = address(okxRouter).call(okxCallData);
    if (!success) {
        assembly {
            revert(add(result, 32), mload(result))
        }
    }

    // Decode the return value
    supportedAssetAmount = abi.decode(result, (uint256));

    // Approve teller's vault to spend the supported asset
    // ...
    return supportedAssetAmount;
}
```

If the `okxCallData` specifies a partial swap, the OKX router pulls only the used amount from the wrapper, leaving the unspent input tokens in the wrapper contract. The wrapper does not account for or refund this remainder, it proceeds to deposit only the `supportedAssetAmount` (output from the swap) into the vault, ignoring the unspent input.

Example scenario (short):

* User sends 100 USDC as `fromTokenAmount`.
* okxCallData configures `batchesAmount` summing to 80 USDC.
* Wrapper transfers 100 USDC to itself.
* OKX router pulls 80 USDC, performs the swap.
* 20 USDC remain in the wrapper, not refunded or deposited.

### Impact Details

* When depositing via OKX with partial swaps, unspent input tokens are left in the `DexAggregatorWrapperWithPredicateProxy` contract but never returned to the user.
* This loss accumulates with every partial swap, potentially trapping significant value in the wrapper over time.
* The wrapper keeps approving the OKX router to spend more funds than it needs.

## Recommendation

{% hint style="info" %}
After executing the OKX swap in `_okxHelper`, check the wrapper contract's balance of the `fromToken` and refund any remainder to `msg.sender` using `SafeERC20.safeTransfer`. Ensure approvals are set minimally or reset (e.g., set to zero) if needed.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Context

User wants to execute a partial swap using the OKX Router.
{% endstep %}

{% step %}

### PoC Steps

* User sends 100 USDC as `fromTokenAmount`.
* okxCallData configures `batchesAmount` summing to 80 USDC.
* Wrapper transfers 100 USDC to itself.
* OKX router pulls 80 USDC, performs the swap.
* 20 USDC remain in the wrapper, not refunded or deposited.
  {% endstep %}

{% step %}

### Result

User loses the unspent input tokens as they are stuck in the `DexAggregatorWrapperWithPredicateProxy`.
{% endstep %}
{% endstepper %}
