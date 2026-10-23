# 52794 sc low remainingforsale not updated after withdrawunsoldarctokens will cause following buy revert

**Submitted on Aug 13th 2025 at 08:26:11 UTC by @maggie for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52794
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

ArcTokenPurchase.sol#withdrawUnsoldArcTokens() allows the token admin to withdraw unsold ArcTokens after a sale (or if disabled). After a sale, if the token sale is not disabled and the admin withdraws part of the unsold ArcTokens, the remaining ArcTokens could still be sold. However, during the withdraw, the `remainingForSale` (as reported by `getMaxNumberOfTokens`) is not updated. The real tokens left on the contract become smaller than `getMaxNumberOfTokens()`, which can cause subsequent buy transactions to revert due to `ContractBalanceInsufficient`.

## Vulnerability Details

Current function:

```solidity
function withdrawUnsoldArcTokens(
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

The function compares `amount` to `token.balanceOf(address(this))`, but the unsold amount (the logical remaining amount for sale) is:

```solidity
getMaxNumberOfTokens(_tokenContract)

function getMaxNumberOfTokens(
    address _tokenContract
) external view returns (uint256) {
    TokenInfo storage info = _getPurchaseStorage().tokenInfo[_tokenContract];
    return info.totalAmountForSale - info.amountSold;
}
```

After withdrawing tokens, `info.totalAmountForSale` (or equivalent stored remaining-for-sale counter) should be updated to reflect the withdrawal. If it is not updated, `getMaxNumberOfTokens(_tokenContract)` will report a higher remaining-for-sale amount than actually available. A buyer relying on `getMaxNumberOfTokens` could attempt to buy that amount and the buy would revert due to `ContractBalanceInsufficient`.

Recommended change (as provided by reporter):

```solidity
function withdrawUnsoldArcTokens(
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
    TokenInfo storage info = _getPurchaseStorage().tokenInfo[_tokenContract];
    uint256 contractBalance = info.totalAmountForSale - info.amountSold;
    if (contractBalance < amount) {
        revert InsufficientUnsoldTokens();
    }

    bool success = token.transfer(to, amount);
    if (!success) {
        revert ArcTokenWithdrawalFailed();
    }
    info.totalAmountForSale = info.totalAmountForSale - amount;
}
```

## Impact Details

After withdrawing unsold ArcTokens, the remaining-for-sale amount returned by `getMaxNumberOfTokens(_tokenContract)` will be greater than the real tokens available in the contract. This mismatch can cause buy operations to revert with `ContractBalanceInsufficient`.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol?utm\\_source=immunefi#L439-L461>

## Proof of Concept

{% stepper %}
{% step %}

1. Enable tokenA for sale with amount = 500000000000000000000.
   {% endstep %}

{% step %}
2\. Withdraw half of unsold ArcTokens, withdraw amount = 250000000000000000000.
{% endstep %}

{% step %}
3\. Query remaining for sale amount.
{% endstep %}
{% endstepper %}

Add the following test to /test/ArcTokenPurchase.t.sol:

```solidity
function test_WithdrawUnsoldArcTokens_poc() public {
    console.log("before withdraw: remainingForSale         = %s",purchase.getMaxNumberOfTokens(address(token)));
    vm.prank(owner);
    uint256 withdran_amount = purchase.getMaxNumberOfTokens(address(token))/2;
    purchase.withdrawUnsoldArcTokens(address(token), bob, withdran_amount);

    assertEq(token.balanceOf(bob), withdran_amount);
    console.log("after withdraw: remainingForSale          = %s",purchase.getMaxNumberOfTokens(address(token)));
    console.log("after withdraw: real tokens forsale       = %s",token.balanceOf(address(purchase)));
}
```

Run:

forge test --via-ir --match-path test/ArcTokenPurchase.t.sol --match-contract ArcTokenPurchaseTest --match-test test\_WithdrawUnsoldArcTokens\_poc -vv

Test output provided by reporter:

```
[PASS] test_WithdrawUnsoldArcTokens_poc() (gas: 149154)
Logs:
  before withdraw: remainingForSale         = 500000000000000000000
  after withdraw: remainingForSale          = 500000000000000000000
  after withdraw: real tokens forsale       = 250000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.58ms (349.10µs CPU time)
```

This demonstrates `getMaxNumberOfTokens` still reporting 500...000 while actual token balance on the purchase contract is 250...000 after withdrawal.
