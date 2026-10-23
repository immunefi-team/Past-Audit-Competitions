# 53071 sc insight okxhelper function incompatible with the uniswap v3 swap to with permit selector&#x20;

**Submitted on Aug 14th 2025 at 18:52:21 UTC by @frolic for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53071
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `_okxHelper` function in the `DexAggregatorWrapperWithPredicateProxy` contract is incompatible with the `UNISWAP_V3_SWAP_TO_WITH_PERMIT_SELECTOR` route. This selector requires a valid permit signature from the token holder to allow the router to spend tokens, but the contract only uses `approve` and does not generate or forward a permit. Since the contract itself is the token holder after receiving tokens from the user, it must generate the permit signature, which is not possible without the contract holding the private key. As a result, swaps using this selector will always fail.

### Vulnerability Details

The `UNISWAP_V3_SWAP_TO_WITH_PERMIT_SELECTOR` function is designed to let users approve token spending and execute a Uniswap V3 swap in a single transaction by providing a signed permit. In the `_okxHelper` function, when this selector is detected, the contract performs a standard `approve` on the token and never extracts or forwards the permit data to the router:

```solidity
if (useNative) {
    canonicalWrapToken.approve(okxApprover, nativeValueToWrap);
} else {
    ERC20 depositAsset = ERC20(fromToken);
    depositAsset.safeTransferFrom(msg.sender, address(this), fromTokenAmount);
    depositAsset.safeApprove(okxApprover, fromTokenAmount);
    // No permit logic here
}
(bool success, bytes memory result) = address(okxRouter).call(okxCallData);
```

However, after the user sends tokens to the contract, the contract becomes the token holder. The permit mechanism requires the token holder to sign a message with their private key, but smart contracts cannot produce such signatures. Therefore, the contract cannot generate a valid permit for the router, and the router will not be authorized to spend the tokens. This makes the `UNISWAP_V3_SWAP_TO_WITH_PERMIT_SELECTOR` route unusable through this wrapper.

### Impact Details

Any user attempting to use the `uniswapV3SwapToWithPermit` route via this contract will have their transaction revert or fail, as the router will not receive a valid permit and cannot spend the contract's tokens. This results in failed swaps, wasted gas, and a broken user experience. The inability to use permit-based swaps also removes the intended UX and gas-saving benefits of the permit mechanism for users of this wrapper.

## References

* DexAggregatorWrapperWithPredicateProxy.sol - \_okxHelper function\
  <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/helper/DexAggregatorWrapperWithPredicateProxy.sol#L303>
* UNISWAP\_V3\_SWAP\_TO\_WITH\_PERMIT\_SELECTOR interface and behavior\
  <https://vscode.blockscan.com/42161/0x01d8edb8ef96119d6bada3f50463dee6fe863b4c>. /Arbiscan (One)/DexRouter/contracts/8/DexRouter.sol

## Proof of Concept

{% stepper %}
{% step %}

### Reproduce the issue

User calls `depositOkxUniversal` with `okxCallData` set to use the `UNISWAP_V3_SWAP_TO_WITH_PERMIT_SELECTOR` route, including a permit signature.
{% endstep %}

{% step %}

### Contract receives tokens

The contract receives the tokens and attempts to approve the router, but does not generate or forward a permit.
{% endstep %}

{% step %}

### Router expects permit from the token holder

The router expects a permit from the token holder (now the contract), but the contract cannot produce a valid signature.
{% endstep %}

{% step %}

### Outcome

The router cannot spend the tokens, and the swap fails or reverts.
{% endstep %}
{% endstepper %}
