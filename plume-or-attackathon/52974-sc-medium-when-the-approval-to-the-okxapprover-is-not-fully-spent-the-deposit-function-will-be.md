# 52974 sc medium when the approval to the okxapprover is not fully spent the deposit function will be blocked

**Submitted on Aug 14th 2025 at 14:31:34 UTC by @TeamJosh for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52974
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Unbounded gas consumption
  * Smart Contract Unable to operate but not due to lack of funds

## Description

### Brief / Intro

The `DexAggregatorWrapperWithPredicateProxy.sol` dangerously uses `safeApprove` on the `okxApprover` without resetting the approval to zero. `safeApprove` reverts if the current allowance is not zero.

### Vulnerability Details

When the `okxApprover` uses fewer funds than what it was approved for, the deposit functions will be blocked for everyone.

Look at the `_okxHelper` function. This function is called by the `depositOkxUniversal` and `depositAndBridgeOkxUniversal` functions. First, it approves the `okxApprover` to spend the `depositAsset` using `safeApprove`, then it calls the `okxApprover` (via `okxRouter`) to swap the tokens. If the `okxApprover` fails to use the whole allowance, the transaction will go through, leaving the contract with some residual approval for that token. Subsequent transactions that call `safeApprove` to set a new non-zero allowance will revert because `safeApprove` requires address(0) -> new amount pattern (or first zero the allowance).

Relevant snippet:

```sol
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
...
                // Use safeTransferFrom
                depositAsset.safeTransferFrom(msg.sender, address(this), fromTokenAmount);

                // Use standard approve (as requested) for the OKX approver
 @->               depositAsset.safeApprove(okxApprover, fromTokenAmount);
            }

            // Execute the swap with the provided calldata
@->            (bool success, bytes memory result) = address(okxRouter).call(okxCallData);
            if (!success) {
                assembly {
                    revert(add(result, 32), mload(result))
                }
            }
...
    }
```

## Impact Details

The `okxApprover` will be permanently blocked from using the `DexAggregatorWrapperWithPredicateProxy.sol` (for that token) if residual allowances remain. This also affects integrations such as the 1inch aggregator that rely on similar approval flows.

## Proof of Concept

<details>

<summary>Proof of Concept (step-by-step)</summary>

StepBob, a malicious attacker, wants to block users from depositing USDC with Okx.StepBob calls depositAndBridgeOkxUniversal and supplies a fromTokenAmount that is greater than the actual amount the okxCallData will cause the okxApprover to swap.StepInside \_okxHelper, the contract executes:depositAsset.safeTransferFrom(msg.sender, address(this), fromTokenAmount);depositAsset.safeApprove(okxApprover, fromTokenAmount);The okxApprover performs the swap but uses less than the approved allowance, leaving a residual allowance.StepAfter this, another user (Alice) attempts to deposit. Her transaction calls safeApprove(okxApprover, amount) again; since the current allowance is non-zero, safeApprove reverts and the deposit is blocked.

</details>

## Remediation (suggested approach)

* When setting approvals for a third-party router/approver that may use a subset of the allowance, first set allowance to zero before setting the desired non-zero allowance, or adopt an approve-if-zero pattern or use increaseAllowance / decreaseAllowance where appropriate.
* Alternatively, use an approval pattern that sets an effectively infinite allowance once (and relies on trusted approver behavior), or use pull-based transfers when possible.
* Ensure callers sanitize `fromTokenAmount` vs calldata amounts, or validate the expected spend from `okxCallData` to avoid over-approving.
