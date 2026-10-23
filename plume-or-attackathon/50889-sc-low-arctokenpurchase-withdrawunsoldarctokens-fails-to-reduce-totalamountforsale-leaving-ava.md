# 50889 sc low arctokenpurchase withdrawunsoldarctokens fails to reduce totalamountforsale leaving availability counters wrong

Submitted on Jul 29th 2025 at 11:56:37 UTC by @Paludo0x for Attackathon | Plume Network

* Report ID: #50889
* Report Type: Smart Contract
* Report severity: Low
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* Impacts:
  * Theft of gas

## Description

### Vulnerability Details

`ArcTokenPurchase::withdrawUnsoldArcTokens()` allows the token admin to pull unsold ArcTokens out of the contract without decreasing `totalAmountForSale`.

The helper `getMaxNumberOfTokens()` is intended to report how many ArcToken base-units are still available in an ongoing sale:

{% code title="getMaxNumberOfTokens" %}

```solidity
function getMaxNumberOfTokens(
    address _tokenContract
) external view returns (uint256) {
    TokenInfo storage info = _getPurchaseStorage().tokenInfo[_tokenContract];
    return info.totalAmountForSale - info.amountSold;
}
```

{% endcode %}

Because `withdrawUnsoldArcTokens()` does not decrement `totalAmountForSale` when transferring tokens out, front-ends that call `getMaxNumberOfTokens()` will display inflated availability. Honest buyers may then submit `buy()` transactions which revert with `ContractBalanceInsufficient`, wasting gas.

### Impact Details

{% hint style="warning" %}
No funds are lost, but front-ends display inflated availability; honest buyers can waste gas on failing transactions.
{% endhint %}

## Recommended Fix

Adjust accounting whenever unsold tokens are withdrawn. Example patch:

{% code title="Suggested fix" %}

```solidity
function withdrawUnsoldArcTokens(...)
    external onlyTokenAdmin(_tokenContract)
{
    TokenInfo storage info = ps.tokenInfo[_tokenContract];

    if (amount > (info.totalAmountForSale - info.amountSold)) {
        revert InsufficientUnsoldTokens();
    }

    info.totalAmountForSale -= amount;      // ← keep quota consistent
    token.safeTransfer(to, amount);
}
```

{% endcode %}

## Proof of Concept

Relevant snippet from the contract showing current behavior:

{% code title="withdrawUnsoldArcTokens (current)" %}

```
```

{% endcode %}

\`\`\`solidity /\*\* \* @dev Allows the token admin to withdraw unsold ArcTokens after a sale (or if disabled). \* @param \_tokenContract The ArcToken contract address. \* @param to The address to send the tokens to. \* @param amount The amount of ArcTokens to withdraw. \*/ function withdrawUnsoldArcTokens( address \_tokenContract, address to, uint256 amount ) external onlyTokenAdmin(\_tokenContract) { if (to == address(0)) { revert CannotWithdrawToZeroAddress(); } if (amount == 0) { revert AmountMustBePositive(); }

```
ArcToken token = ArcToken(_tokenContract);
uint256 contractBalance = token.balanceOf(address(this));
if (contractBalance < amount) {
    revert InsufficientUnsoldTokens();
}

bool success = token.transfer(to, amount);
if (!success) {
    revert ArcTokenWithdrawalFailed();
}
```

}

```
```
