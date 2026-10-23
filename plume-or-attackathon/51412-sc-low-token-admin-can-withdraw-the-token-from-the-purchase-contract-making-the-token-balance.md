# 51412 sc low token admin can withdraw the token from the purchase contract making the token balance to be less than the totalamountforsale

**Submitted on Aug 2nd 2025 at 15:06:27 UTC by @TeamJosh for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #51412
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

When listing a token for sale the token owner needs to set the `totalAmountForSale` and send at least that amount into the contract. Failure to do so means that the token cannot be listed. However, once this token has been listed the token owner can withdraw tokens without reducing the `totalAmountForSale`. This leads to inconsistent states that will cause a revert in the `buy` function when buyers attempt to purchase tokens that are no longer present in the contract.

### Vulnerability Details

Token admins can list tokens by calling the `enableToken` function where they set the token price and `totalAmountForSale`. The `totalAmountForSale` is validated to ensure that the owner has sent at least that amount of token.

See the `_numberOfTokens` param in the function below.

```sol
    function enableToken(
        address _tokenContract,
        uint256 _numberOfTokens,
        uint256 _tokenPrice
    ) external onlyTokenAdmin(_tokenContract) { 

...
        if (
            ArcToken(_tokenContract).balanceOf(address(this)) < _numberOfTokens
        ) {
            revert ContractMissingRequiredTokens();
        }

        ps.tokenInfo[_tokenContract] =
            TokenInfo({ isEnabled: true, tokenPrice: _tokenPrice, totalAmountForSale: _numberOfTokens, amountSold: 0 });

        ps.enabledTokens.add(_tokenContract);

        emit TokenSaleEnabled(_tokenContract, _numberOfTokens, _tokenPrice); //@audit should be restricted.
    }
```

However, the owner can always withdraw tokens without updating the `totalAmountForSale`. This will cause a situation where the contract appears to have more tokens for sale than it actually has.

```sol
    function withdrawUnsoldArcTokens( //@audit insight Token admin can withdraw while sale is active
        address _tokenContract,
        address to,
        uint256 amount
    ) external onlyTokenAdmin(_tokenContract) {
        if (to == address(0)) {
            revert CannotWithdrawToZeroAddress();
        }
        if (amount == 0) {
            revert AmountMustBePositive();
        }

        ArcToken token = ArcToken(_tokenContract);
        uint256 contractBalance = token.balanceOf(address(this));
        if (contractBalance < amount) {
            revert InsufficientUnsoldTokens();
        }

        bool success = token.transfer(to, amount);
        if (!success) {
            revert ArcTokenWithdrawalFailed();
        }
    }
```

## Impact Details

1. Inconsistent state of the amount of token that is really available for sale.
2. The sale will appear to offer more tokens than are actually available, causing buyer transactions to revert.

## Proof of Concept

{% stepper %}
{% step %}

### Step

Enable Token by calling `enableToken`.
{% endstep %}

{% step %}

### Step

Withdraw tokens by calling `withdrawUnsoldArcTokens`.
{% endstep %}
{% endstepper %}

## References

<details>

<summary>Relevant links</summary>

* Vulnerable target contract: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* Submission / bounty page: <https://immunefi.com/audit-competition/plume-network-attackathon>

</details>
