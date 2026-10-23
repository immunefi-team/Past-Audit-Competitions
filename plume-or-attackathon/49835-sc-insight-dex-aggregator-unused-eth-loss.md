# 49835 sc insight dex aggregator unused eth loss

**Submitted on Jul 19th 2025 at 21:56:53 UTC by @Blobism for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49835
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

If a user calls either of the Dex Aggregator deposit methods with a value of ETH larger than the amount they request to wrap, the remaining ETH gets left in the contract, resulting in the funds being locked/stolen.

### Vulnerability Details

Calling `depositOneInch` or `depositOkxUniversal` with the `nativeValueToWrap` parameter will result in only that amount of sent ETH getting wrapped to WETH, and any remaining ETH gets left in the contract.

The wrapping of the WETH is shown below. Notice an amount can be wrapped that is smaller than `msg.value`:

```solidity
function _checkAndMintNativeAmount(uint256 nativeAmount) internal returns (bool useNative) {
    if (nativeAmount > msg.value) {
        revert DexAggregatorWrapper__InsufficientEthForSwap();
    }
    if (nativeAmount > 0) {
        // Direct WETH call, no SafeTransferLib needed here
        canonicalWrapToken.deposit{ value: nativeAmount }();
        useNative = true;
    }
    // Implicitly returns false if nativeAmount is 0
}
```

After the ETH is wrapped and sent, the remaining ETH is NOT refunded in `depositOneInch` and `depositOkxUniversal`, so the remaining ETH gets left in the contract. See the 1inch version:

```solidity
function depositOneInch(
    // ...
    uint256 nativeValueToWrap,
    PredicateMessage calldata predicateMessage
)
    external
    payable
    nonReentrant
    returns (uint256 shares)
{
    // ...

    // Use safeTransfer to send shares to msg.sender
    ERC20(vaultAddress).safeTransfer(msg.sender, shares);

    // <----------- BUG: no excess ETH refund

    _calcSharesAndEmitEvent(
        supportedAsset,
        CrossChainTellerBase(address(teller)),
        address(desc.srcToken),
        desc.amount,
        supportedAssetAmount
    );
}
```

Compare this to `depositAndBridgeOneInch` and `depositAndBridgeOkxUniversal`, where remaining ETH is refunded at the end of the methods. See the 1inch version:

```solidity
function depositAndBridgeOneInch(
    // ...
)
    external
    payable
    nonReentrant
{
    // ...

    // Refund any excess ETH
    _refundExcessEth(payable(msg.sender));

    _calcSharesAndEmitEvent(supportedAsset, teller, address(desc.srcToken), desc.amount, supportedAssetAmount);
}
```

{% hint style="warning" %}
Important note: the ETH refund method contains a vulnerability which allows this unused ETH to be stolen. This is a distinct bug — the current issue is critical regardless of whether the excess ETH is permanently locked or can be stolen by a separate exploit. While a user might attempt to recover funds via the broken refund mechanics, an attacker can race to use that approach first and steal funds from the contract. Aside from that exploit, the funds may remain permanently locked.
{% endhint %}

## Impact Details

Because `nativeValueToWrap` is a parameter of the deposit methods, it is reasonable for a user to pass a larger amount of ETH than what they are wrapping. It is a critical vulnerability that the remaining ETH is not returned to the user.

The "deposit and bridge" methods handle this correctly in principle (disregarding the distinct refund vulnerability) because they refund remaining ETH at the end of the methods.

## References

See `src/helper/DexAggregatorWrapperWithPredicateProxy.sol` in the target repository: <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>

## Proof of Concept

The following path leads to the vulnerability:

{% stepper %}
{% step %}
User calls `depositOneInch` or `depositOkxUniversal` with a nonzero `nativeValueToWrap` which is smaller than `msg.value` (native ETH sent).
{% endstep %}

{% step %}
Some of the native ETH is not wrapped and remains in the contract.
{% endstep %}

{% step %}
The remaining ETH is now locked in the contract, or can potentially be stolen by an attacker using a separate exploit to drain the unused ETH.
{% endstep %}
{% endstepper %}
