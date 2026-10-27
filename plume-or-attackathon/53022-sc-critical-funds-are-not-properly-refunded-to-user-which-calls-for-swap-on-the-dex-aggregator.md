# 53022 sc critical funds are not properly refunded to user which calls for swap on the dex aggregator

* **Submitted on:** Aug 14th 2025 at 17:13:54 UTC by @valkvalue for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #53022
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

Funds are not properly refunded to a user who initiates a swap via the dex aggregator.

### Vulnerability Details

The contract attempts to refund leftover ETH/native token balance, but it does not account for refunds that originate from the swap operation itself when those refunds are returned as ERC20 tokens. The code only handles native ETH refunds and has no logic to capture and re-send ERC20 tokens accidentally left in the aggregator contract by the routed swap.

Relevant excerpt from the 0x (Ork) dex router's swap implementation:

```solidity
    function swap(
        IAggregationExecutor executor,
        SwapDescription calldata desc,
        bytes calldata data
    )
        external
        payable
        whenNotPaused()
        returns (
            uint256 returnAmount,
            uint256 spentAmount
        )
    {
        if (desc.minReturnAmount == 0) revert ZeroMinReturn();

        IERC20 srcToken = desc.srcToken;
        IERC20 dstToken = desc.dstToken;

        bool srcETH = srcToken.isETH();
        if (desc.flags & _REQUIRES_EXTRA_ETH != 0) {
            if (msg.value <= (srcETH ? desc.amount : 0)) revert RouterErrors.InvalidMsgValue();
        } else {
            if (msg.value != (srcETH ? desc.amount : 0)) revert RouterErrors.InvalidMsgValue();
        }

        if (!srcETH) {
            srcToken.safeTransferFromUniversal(msg.sender, desc.srcReceiver, desc.amount, desc.flags & _USE_PERMIT2 != 0);
        }

        returnAmount = _execute(executor, msg.sender, desc.amount, data);
        spentAmount = desc.amount;

        if (desc.flags & _PARTIAL_FILL != 0) {
            uint256 unspentAmount = srcToken.uniBalanceOf(address(this));
            if (unspentAmount > 1) {
                // we leave 1 wei on the router for gas optimisations reasons
                unchecked { unspentAmount--; }
                spentAmount -= unspentAmount;
                srcToken.uniTransfer(payable(msg.sender), unspentAmount);
            }
            if (returnAmount * desc.amount < desc.minReturnAmount * spentAmount) revert RouterErrors.ReturnAmountIsNotEnough(returnAmount, desc.minReturnAmount * spentAmount / desc.amount);
        } else {
            if (returnAmount < desc.minReturnAmount) revert RouterErrors.ReturnAmountIsNotEnough(returnAmount, desc.minReturnAmount);
        }

        address payable dstReceiver = (desc.dstReceiver == address(0)) ? payable(msg.sender) : desc.dstReceiver;
        dstToken.uniTransfer(dstReceiver, returnAmount);
    }
```

Note: when the order is partially filled, the router returns unspent srcToken to msg.sender (the dex aggregator contract in this flow). The aggregator contract's only refund logic is:

```solidity
    function _refundExcessEth(address payable _recipient) internal {
        uint256 balance = address(this).balance;
        if (balance > 0) {
            (bool success,) = _recipient.call{ value: balance }("");
            if (!success) {
                revert DexAggregatorWrapper__EthRefundFailed();
            }
        }
        // If balance is 0, do nothing.
    }
```

This only handles native ETH. It does not handle ERC20 srcToken refunds that the router may have transferred to the aggregator contract, so those ERC20 tokens can remain stuck inside the aggregator indefinitely.

### Impact Details

Permanent funds stuck in the aggregator contract — users' leftover ERC20 tokens (from partial fills or other swap-side refunds) are not returned and thus become unrecoverable through the intended flow.

### References

* <https://vscode.blockscan.com/42161/0x111111125421ca6dc452d289314280a0f8842a65>
* <https://vscode.blockscan.com/ethereum/0x2E1Dee213BA8d7af0934C49a23187BabEACa8764>

{% hint style="danger" %}
Severity: Critical — permanent freezing of user funds is possible.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Reproduce flow

* User calls one of the wrapper functions in DexAggregatorWrapperWithPredicateProxy:
  * depositAndBridgeOkxUniversal
  * depositOkxUniversal
  * depositOneInch
  * depositAndBridgeOneInch
    {% endstep %}

{% step %}

### Partial fill or other swap-side refund occurs

* The dex router executes a swap that results in an unspent amount of the src token being transferred back to msg.sender (the aggregator contract).
* Those refunded tokens are ERC20 tokens, not native ETH.
  {% endstep %}

{% step %}

### Aggregator refund logic

* The aggregator attempts to refund leftover ETH/native token via \_refundExcessEth, but there is no logic to detect or transfer ERC20 tokens that were returned from the router to the aggregator.
  {% endstep %}

{% step %}

### Result

* ERC20 tokens remain stuck in the aggregator contract and are not returned to the user, leading to permanently frozen funds.
  {% endstep %}
  {% endstepper %}

## Suggested Remediation (high level)

* Detect and forward any ERC20 token balances that belong to the user (or were part of the swap flow) back to the user when appropriate, similar to how ETH refunds are handled.
* After a swap call, check balances of relevant ERC20 tokens (srcToken and possibly intermediary tokens) and transfer any unexpected balances to the intended recipient.
* Ensure the router's behavior (especially when using flags like PARTIAL\_FILL) is accounted for: the aggregator should not assume only ETH/native refunds are possible.
* Consider explicit handling for all tokens involved in the swap (srcToken, dstToken, and any tokens the executor may refund) to avoid leaving tokens in the aggregator.

(Do not add any behavior not described here without further design review — the above are high-level mitigations derived from the observed issue.)
