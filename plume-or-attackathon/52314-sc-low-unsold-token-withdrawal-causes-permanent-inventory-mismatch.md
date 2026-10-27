# 52314 sc low unsold token withdrawal causes permanent inventory mismatch

**Submitted on Aug 9th 2025 at 19:21:31 UTC by @itsravin0x for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52314
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `ArcTokenPurchase` contract contains an accounting flaw where withdrawing unsold tokens via `withdrawUnsoldArcTokens()` does not update the internal `TokenInfo` state variables used to track remaining sale inventory. This creates a permanent mismatch between the reported number of tokens available (`getMaxNumberOfTokens`) and the actual token balance in the contract. As a result, buyers will be shown incorrect availability and any attempted purchases after such a withdrawal will fail, effectively halting the sale and misleading participants.

### Vulnerability Details

The contract tracks token sale data in the `TokenInfo` struct, where `totalAmountForSale` and `amountSold` are used to compute availability via:

```solidity
function getMaxNumberOfTokens(address _tokenContract) 
    external view returns (uint256) 
{
    TokenInfo storage info = _getPurchaseStorage().tokenInfo[_tokenContract];
    return info.totalAmountForSale - info.amountSold;
}
```

However, the `withdrawUnsoldArcTokens()` function transfers tokens out without adjusting `totalAmountForSale` or `amountSold`:

```solidity
function withdrawUnsoldArcTokens(address _tokenContract, address to, uint256 amount) 
    external onlyTokenAdmin(_tokenContract) 
{
    // ... validation checks ...
    ArcToken token = ArcToken(_tokenContract);
    uint256 contractBalance = token.balanceOf(address(this));
    if (contractBalance < amount) {
        revert InsufficientUnsoldTokens();
    }
    bool success = token.transfer(to, amount);
    if (!success) {
        revert ArcTokenWithdrawalFailed();
    }
    // Missing: update info.totalAmountForSale -= amount;
}
```

Because `getMaxNumberOfTokens()` reads from stale state, the reported availability remains unchanged after withdrawals, even when the contract’s balance is depleted.

Example (Observed in Test)

* Before withdrawal:
  * Reported remaining tokens: `500`
  * Actual balance: `500`
* After withdrawal of all 500:
  * Reported remaining tokens: `500` (stale)
  * Actual balance: `0`

Any subsequent purchase will revert due to:

```solidity
if (contractBalance < arcTokensBaseUnitsToBuy) {
    revert ContractBalanceInsufficient();
}
```

but UIs or off-chain systems relying on `getMaxNumberOfTokens` will still display the stale `500` tokens as available.

### Impact Details

* Denial of Service: All further purchases will fail after an unsold token withdrawal, halting sales completely.
* False Reporting / Market Manipulation Risk: Off-chain systems and frontends using `getMaxNumberOfTokens` will misrepresent sale inventory, misleading users into believing tokens remain for sale.
* Loss of Revenue: The project may lose potential buyers due to perceived availability followed by transaction failures.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol#L27-L32>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol#L359-L364>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol#L433-L461>

## Proof of Concept

<details>

<summary>Test demonstrating the inconsistent accounting (click to expand)</summary>

Paste this in `arc/test/ArcTkenPurchase.t.sol`

```solidity
function test_InconsistentTokenAccounting() public {
    uint256 reported_remaining_number_of_tokens_before = purchase
        .getMaxNumberOfTokens(address(token));
    console.log(
        "Reported remaining number of tokens BEFORE withdrawal: ",
        reported_remaining_number_of_tokens_before
    );
    uint256 actual_remaining_number_of_tokens_before = token.balanceOf(
        address(purchase)
    );

    console.log(
        "Actual remaining number of Tokens BEFORE withdrawal: ",
        actual_remaining_number_of_tokens_before
    );

    purchase.withdrawUnsoldArcTokens(
        address(token),
        address(this),
        actual_remaining_number_of_tokens_before
    );

    uint256 reported_remaining_number_of_tokens_after = purchase
        .getMaxNumberOfTokens(address(token));
    console.log(
        "Reported remaining number of tokens AFTER withdrawal: ",
        reported_remaining_number_of_tokens_after
    );
    uint256 actual_remaining_number_of_tokens_after = token.balanceOf(
        address(purchase)
    );

    console.log(
        "Actual remaining number of Tokens AFTER withdrawal: ",
        actual_remaining_number_of_tokens_after
    );
}
```

</details>
