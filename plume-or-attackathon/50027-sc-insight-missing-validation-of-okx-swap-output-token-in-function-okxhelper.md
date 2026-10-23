# 50027 sc insight missing validation of okx swap output token in function okxhelper&#x20;

**Submitted on Jul 21st 2025 at 08:21:14 UTC by @Paludo0x for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50027
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Vulnerability Details

The `_okxHelper` function never verifies that the swap’s toToken matches the declared `supportedAsset`.

The wrapper’s `_okxHelper` function decodes only the amount returned by an OKX Router swap call, without ever checking that the swap’s `toToken` matches the expected `supportedAsset`.

If the wrapper contract already holds `supportedAssetAmount` of `supportedAsset` (for any reason), a malicious operator can submit calldata swapping a cheap token, decode the returned amount as `supportedAssetAmount`, and then the vault will pull the real `supportedAsset` out of the wrapper—minting shares at near-zero cost. In the worst case, this steals full-value shares for almost no input.

## Impact Details

* Failure-only scenario: If the wrapper does not hold pre-existing `supportedAssetAmount`, the subsequent `vault.enter` (or `safeTransferFrom`) will revert, making the entire call fail.
* Exploit scenario: If the wrapper holds `supportedAssetAmount` of the supported token, the operator calls `_okxHelper` with malicious calldata swapping a low-value token (e.g. 1 USDC) but returning `supportedAssetAmount` as the decoded result, the vault pulls the existing `supportedAssetAmount` and mints shares accordingly—netting full-value shares for a trivial cost.
* Severity: Low (requires a trusted operator to submit malformed calldata). Note: the 1inch path enforces this check:

```
if (desc.dstToken != supportedAsset || desc.dstReceiver != address(this)) revert();
```

## Proof of Concept

{% stepper %}
{% step %}

### Overview of how the function behaves

Inside `_okxHelper()`, the contract:

* Reads the OKX call selector via assembly.
* Approves the input token (or WETH) to the OKX approver.
* Executes `(bool success, bytes memory result) = okxRouter.call(okxCallData);`
* Decodes only the returned `uint256 supportedAssetAmount = abi.decode(result, (uint256));`
* Approves the vault to pull `supportedAssetAmount` of the assumed `supportedAsset`.

At no point does the code extract or verify the destination token argument embedded in `okxCallData`.
{% endstep %}

{% step %}

### Code excerpt (relevant function)

```
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
    bytes4 selector;
    assembly {
        selector := calldataload(okxCallData.offset)
    }

    if (
        selector == SMART_SWAP_BY_ORDER_ID_SELECTOR || selector == SMART_SWAP_TO_SELECTOR
            || selector == UNISWAP_V3_SWAP_TO_SELECTOR || selector == UNISWAP_V3_SWAP_TO_WITH_PERMIT_SELECTOR
            || selector == UNXSWAP_BY_ORDER_ID_SELECTOR || selector == UNXSWAP_TO_SELECTOR
    ) {
        bool useNative = _checkAndMintNativeAmount(nativeValueToWrap);
        if (useNative) {
            if (fromToken != address(canonicalWrapToken) || fromTokenAmount != nativeValueToWrap) {
                revert DexAggregatorWrapper__OkxSwapFailed();
            }
            // Use standard approve (as requested)
            canonicalWrapToken.approve(okxApprover, nativeValueToWrap);
        } else {
            // Cast fromToken address to ERC20 to use the library
            ERC20 depositAsset = ERC20(fromToken);

            // Use safeTransferFrom
            depositAsset.safeTransferFrom(msg.sender, address(this), fromTokenAmount);

            // Use standard approve (as requested) for the OKX approver
            depositAsset.safeApprove(okxApprover, fromTokenAmount);
        }

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
        // Cast teller address to TellerWithMultiAssetSupport to call vault()
        address vaultAddress = address(TellerWithMultiAssetSupport(payable(teller)).vault());
        if (vaultAddress == address(0)) {
            revert("DexAggregatorWrapper: Invalid vault address for approval");
        }
        // Use standard approve (as requested)
        supportedAsset.safeApprove(vaultAddress, supportedAssetAmount);

        // Return value needs to be here since it's declared in the function signature
        return supportedAssetAmount;
    } else {
        revert DexAggregatorWrapper__UnsupportedOkxFunction();
    }
    // Note: If the selector doesn't match, the function will revert above, so no explicit return needed here.
}
```

{% endstep %}

{% step %}

### Why this is exploitable

* The function never inspects `okxCallData` to verify the destination token or receiver.
* A malicious calldata can be crafted so the OKX router call returns an arbitrary uint256 (e.g., a large `supportedAssetAmount`) while actually swapping to a different token/value.
* If the wrapper already holds the real `supportedAssetAmount`, the vault will pull the real supported asset (because the wrapper has it) while the operator only provided a cheap input token—resulting in minting shares against real assets with negligible cost.
  {% endstep %}

{% step %}

### Comparison with 1inch handling

The `_oneInchHelper()` function includes an explicit check:

```
if (desc.dstToken != supportedAsset || desc.dstReceiver != address(this)) {
    revert DexAggregatorWrapper__InvalidSwapDescription();
}
```

This check prevents the described exploit for 1inch paths because the destination token and receiver must match expectations. `_okxHelper()` lacks an equivalent verification.
{% endstep %}

{% step %}

### Outcome

* If the wrapper does not hold the declared `supportedAssetAmount`, the call will likely revert (failure-only scenario).
* If the wrapper does hold that balance, an attacker or malicious/trusted operator can craft calldata to mint full-value shares for negligible cost (exploit scenario).
  {% endstep %}
  {% endstepper %}

## Suggested Mitigation (not exhaustive)

* Verify the destination token and receiver encoded in `okxCallData` match `supportedAsset` and `address(this)` before relying on the decoded return value, similar to the 1inch handler.
* Parse and validate the calldata fields relevant to destination token/receiver for supported OKX selectors prior to executing the call.
* Where direct parsing is infeasible for a selector, consider rejecting that selector (or route) or requiring an on-chain verified description object (like 1inch's `desc`) to be provided and validated.

## References

* Target source: <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>

{% hint style="info" %}
This report maintains the original author's details and findings. No links or code were modified from the original report.
{% endhint %}
