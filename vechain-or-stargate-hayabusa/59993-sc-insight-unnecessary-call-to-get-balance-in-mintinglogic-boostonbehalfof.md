# 59993 sc insight unnecessary call to get balance in mintinglogic boostonbehalfof&#x20;

**Submitted on Nov 17th 2025 at 12:46:06 UTC by @JJSOnChain for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59993
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/libraries/MintingLogic.sol>

## Description

### Brief / Intro

The `boostOnBehalfOf()` function redundantly fetches a user's `VTHO` balance twice, increasing code verbosity and causing a small, avoidable gas overhead.

### Vulnerability Details

In `boostOnBehalfOf()`, the code retrieves the balance into a local variable, then calls `balanceOf` again when checking the required boost amount:

```solidity
uint256 balance = $.vthoToken.balanceOf(_sender); 
// check that the boost amount is enough
if ($.vthoToken.balanceOf(_sender) < requiredBoostAmount) { // @audit-info - could simply use `balance`
    revert Errors.InsufficientBalance(
        address($.vthoToken),
        _sender,
        requiredBoostAmount,
        balance
    );
}
```

Since the balance was already fetched into `balance`, the second call to `$.vthoToken.balanceOf(_sender)` is redundant. The condition can (and should) use the already-read `balance` variable to avoid the extra external call.

### Impact Details

This results in an unnecessary external call to the token contract, increasing transaction gas slightly and adding verbosity to the code. While not a functional vulnerability, it is an efficiency and code-quality improvement to consolidate to a single balance read.

## Proof of Concept

<details>

<summary>Code excerpt (from packages/contracts/contracts/StargateNFT/libraries/MintingLogic.sol around Line 116)</summary>

```solidity
function boostOnBehalfOf(
    DataTypes.StargateNFTStorage storage $,
    address _sender,
    uint256 _tokenId
) external {
    // check that the token is not already boosted
    if ($.maturityPeriodEndBlock[_tokenId] <= Clock._clock()) {
        revert Errors.MaturityPeriodEnded(_tokenId);
    }

    uint256 requiredBoostAmount = _boostAmount($, _tokenId);

    uint256 balance = $.vthoToken.balanceOf(_sender);   // <--- first read
    // check that the boost amount is enough
    if ($.vthoToken.balanceOf(_sender) < requiredBoostAmount) {  // <--- redundant second read
        revert Errors.InsufficientBalance(
            address($.vthoToken),
            _sender,
            requiredBoostAmount,
            balance
        );
    }
    ...
}
```

Suggested minimal fix: use the previously read `balance` in the `if` condition:

```solidity
if (balance < requiredBoostAmount) {
    revert Errors.InsufficientBalance(
        address($.vthoToken),
        _sender,
        requiredBoostAmount,
        balance
    );
}
```

</details>
