# 51034 sc low sales information is lost when enabling token

**Submitted on Jul 30th 2025 at 15:12:56 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51034
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `ArcTokenPurchase` lost all its historical (`amountSold` and `totalAmountForSale`) information when a previously stored token is re-enabled.

### Vulnerability Details

The `ArcTokenPurchase` allows a previously enabled/disabled token to be enabled again, but the problem is that it overrides the information of the previous record:

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenPurchase.sol#L172-L173>

```solidity
    function enableToken(
        address _tokenContract,
        uint256 _numberOfTokens,
        uint256 _tokenPrice
    ) external onlyTokenAdmin(_tokenContract) {
        ...
        ps.tokenInfo[_tokenContract] =
            TokenInfo({ isEnabled: true, tokenPrice: _tokenPrice, totalAmountForSale: _numberOfTokens, amountSold: 0 });


      ...
    }
```

This causes the sales information to be broken, as the token previously could have sold an X amount and had a Y amount left for sale.

### Example 1 - Token is disabled and re-enabled later

* Token A is enabled first time with 100 tokens for sale, `amountSold` is 0.
* Token A has 50 units sold. Now `totalAmountForSale` is 50 and `amountSold` is 50.
* Token A is disabled, the record information remains.
* After some time, Token A is enabled again, with 100 tokens. But now:
  1. `totalAmountForSale` is 100, even though it should be 150
  2. `amountSold` is 0, even though 50 tokens were sold previously.

### Example 2

* Token A is enabled first time with 100 tokens for sale, `amountSold` is 0.
* Token A has 50 units sold. Now `totalAmountForSale` is 50 and `amountSold` is 50.
* `enableToken` is called again with the same information. The `amountSold` is set to zero again, and `totalAmountForSale` is 100, even though it should be 150.

### Impact Details

Re-enabling a previously added token resets its sales data, causing `amountSold` and `totalAmountForSale` to misrepresent the actual tokens sold and available.

{% hint style="warning" %}
Recommendation (two alternatives):

* Prevent re-enabling a token that is already enabled.
* When re-enabling a disabled token, merge the new data with existing sales info to preserve accurate `amountSold` and `totalAmountForSale` values.
  {% endhint %}

## Proof of Concept

<details>

<summary>PoC — examples demonstrating overwritten sales data</summary>

**Context**

The PoC below represents the same as added in the report. Two cases show how the sales data will be overwritten, hence incorrectly stored.

**Example 1 - Token is disabled and re-enabled later**

* Token A is enabled first time with 100 tokens for sale, `amountSold` is 0.
* Token A has 50 units sold. Now `totalAmountForSale` is 50 and `amountSold` is 50.
* Token A is disabled, the record information remains.
* After some time, Token A is enabled again, with 100 tokens. But now:
  1. `totalAmountForSale` is 100, even though it should be 150
  2. `amountSold` is 0, even though 50 tokens were sold previously.

**Example 2 - Token is enabled again while already active**

* Token A is enabled first time with 100 tokens for sale, `amountSold` is 0.
* Token A has 50 units sold. Now `totalAmountForSale` is 50 and `amountSold` is 50.
* `enableToken` is called again with the same information. The `amountSold` is set to zero again, and `totalAmountForSale` is 100, even though it should be 150.

</details>
