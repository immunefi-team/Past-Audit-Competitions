# 51352 sc critical user will lose the unspent amount when executing partial swaps via 1inch

Submitted on Aug 1st 2025 at 22:29:36 UTC by @holydevoti0n for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #51352
* Report Type: Smart Contract
* Report severity: Critical
* Target: <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* Impacts:
  * Permanent freezing of funds

## Description

### Brief/Intro

In `DexAggregatorWrapperWithPredicateProxy.depositOneInch`, unspent funds from a partial swap are sent back to the contract but never deposited into `BoringVault` or returned to the user.

### Vulnerability Details

1. 1inch `AggregationRouterV6` (or older versions) allow users to execute partial swaps. The partial swap is enabled by setting the `flags` parameter from the `SwapDescription` to 1. See the live contract on Arbitrum:

   <https://arbiscan.io/address/0x111111125421ca6dc452d289314280a0f8842a65#code#F1#L4790>

   ```solidity
    struct SwapDescription {
           ERC20 srcToken;
           ERC20 dstToken;
           address payable srcReceiver;
           address payable dstReceiver;
           uint256 amount;
           uint256 minReturnAmount;
   @>        uint256 flags;
       }

    function swap(
           IAggregationExecutor executor,
   @>        SwapDescription calldata desc,
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
          ...
                // @audit-issue returning the spent amount to `msg.sender`
   @>        if (desc.flags & _PARTIAL_FILL != 0) {
               uint256 unspentAmount = srcToken.uniBalanceOf(address(this));
               if (unspentAmount > 1) {
                   // we leave 1 wei on the router for gas optimisations reasons
                   unchecked { unspentAmount--; }
                   spentAmount -= unspentAmount;
   @>                srcToken.uniTransfer(payable(msg.sender), unspentAmount);
               }
           ...
       }
   ```

   This means that when the desired swap amount is reached, any unspent source tokens are returned to `msg.sender`, which, in this case, is the `DexAggregatorWrapperWithPredicateProxy` contract itself since it initiates the swap call.

   <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/helper/DexAggregatorWrapperWithPredicateProxy.sol#L271>

   ```solidity
       function _oneInchHelper(
           ERC20 supportedAsset,
           address teller,
           address executor,
           AggregationRouterV6.SwapDescription calldata desc,
           bytes calldata data,
           uint256 nativeValueToWrap
       )
           internal
           returns (uint256 supportedAssetAmount)
       {
         ....
           // Perform swap
   @>        (supportedAssetAmount,) = aggregator.swap(executor, desc, data);
          ...

           return supportedAssetAmount;
       }
   ```

   The problem here is that the unspent amount is not accounted for. For instance:

   * User sends 100 USDC to be swapped for 1 COIN.
   * Only 80 USDC is used for the swap, 1inch returns 20 USDC to the proxy.
   * The proxy contract does not return the 20 USDC to the user.

### Impact Details

* When depositing via 1inch with partial swaps, unspent tokens are sent back to the proxy contract but never returned to the user, resulting in permanent loss of funds for those users.

### Recommendation

In the `_oneInchHelper` function, return the unspent tokens to the user.

## Proof of Concept

Context:

* User wants to execute a partial swap and then deposit into the `BoringVault` using the `DexAggregatorWrapperWithPredicateProxy.depositOneInch` function.
* User wants to swap 100 USDC for 1 COIN.

{% stepper %}
{% step %}

### Step

Construct a `SwapDescription` enabling partial fill:

```solidity
SwapDescription memory swapDesc = SwapDescription({
    srcToken: IERC20(_fromAsset),
    dstToken: IERC20(_toAsset),
    srcReceiver: payable(executer),
    dstReceiver: payable(msg.sender),
    amount: _fromAssetAmount, // 100 USDC
    minReturnAmount: 1e18,
    flags: 1 // enable partial fill (_PARTIAL_FILL)
});
```

{% endstep %}

{% step %}

### Step

Call the aggregator via `depositOneInch` which invokes the proxy's `_oneInchHelper`, leading to:

* 1Inch executes the swap but only needs 80 USDC to fulfill the requested dst amount (1 COIN).
* The aggregator returns the remaining 20 USDC to `msg.sender` (the proxy contract).
  {% endstep %}

{% step %}

### Step

Result:

* The proxy receives 20 USDC back from 1inch.
* The proxy does not forward those 20 USDC to the user or deposit them to `BoringVault`.
* Those funds remain stuck in the contract, resulting in permanent loss that accumulates with each partial swap.
  {% endstep %}
  {% endstepper %}

## Result

User funds are permanently lost with no way to recover them. This loss accumulates with every partial swap executed.
