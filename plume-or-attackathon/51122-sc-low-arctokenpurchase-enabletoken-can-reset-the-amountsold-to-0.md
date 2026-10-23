# 51122 sc low arctokenpurchase enabletoken can reset the amountsold to 0

**Submitted on Jul 31st 2025 at 10:42:08 UTC by @pks271 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51122
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>

## Description

ArcToken can be enabled to buy arc token by token contract admin through `ArcTokenPurchase#enableToken` function. But the related parameters can be reset again by calling `ArcTokenPurchase#enableToken` function including `amountSold`. This allows `amountSold` to be reset to 0 after some sales, which can lead to over-selling relative to the originally intended `totalAmountForSale`.

{% stepper %}
{% step %}

### Reproduction scenario (high level)

* The token contract admin calls `ArcTokenPurchase#enableToken` to set `totalAmountForSale = 1000`.
* Some tokens are purchased (e.g., `amountSold = 700`).
  {% endstep %}

{% step %}

### Admin updates configuration

* The token contract admin calls `ArcTokenPurchase#enableToken` again (for example to update `tokenPrice`).
* This second call resets `amountSold` to 0 while `remainingForSale` remains based on `totalAmountForSale`.
  {% endstep %}

{% step %}

### Over-sell occurs

* After the reset, more tokens can be sold up to `totalAmountForSale` again (e.g., another 1000).
* Total tokens sold becomes 1700 even though `totalAmountForSale` was originally 1000, causing over-sale and mismatch with underlying RWA.
  {% endstep %}
  {% endstepper %}

## Impact

{% hint style="warning" %}
Arc token can be sold out more than expected, which is unfair to earlier buyers and may result in insufficient RWA to deliver the promised assets.
{% endhint %}

## Recommendation

{% hint style="info" %}
Do not allow `amountSold` to be reset to 0 when calling `ArcTokenPurchase#enableToken`. Ensure `amountSold` is preserved (or updated correctly) when the admin re-enables or updates sale parameters so that total accounting cannot be reset and over-selling cannot occur.
{% endhint %}

## Proof of Concept

This test reproduces the issue by enabling a sale, selling 700 tokens, refilling the contract, re-calling `enableToken` (which resets `amountSold`), and then selling 1000 tokens again — resulting in 1700 tokens sold despite `totalAmountForSale` being 1000.

{% code title="ArcTokenPurchase.t.sol" %}

```solidity
 function test_EnableToken_Overflow() public {
        // Mint additional tokens to owner to ensure sufficient balance
        token.mint(owner, 2000e18);

        uint256 initialTokensForSale = 1000e18;
        uint256 tokenPrice = 100e6; // 100 USDC per token

        // Replenish contract balance to 1000e18
        uint256 currentBalance = token.balanceOf(address(purchase));
        uint256 needed = initialTokensForSale - currentBalance;
        token.transfer(address(purchase), needed);
        
        console.log("Initial contract balance:", token.balanceOf(address(purchase)) / 1e18, "tokens");
        
        vm.prank(owner);
        purchase.enableToken(address(token), initialTokensForSale, tokenPrice);
        
        // Verify initial state
        ArcTokenPurchase.TokenInfo memory info1 = purchase.getTokenInfo(address(token));
        assertEq(info1.totalAmountForSale, initialTokensForSale);
        assertEq(info1.amountSold, 0);
        
        // Alice purchases 700 tokens
        uint256 alicePurchaseAmount = 700 * tokenPrice; // 700 * 100 USDC
        purchaseToken.mint(alice, alicePurchaseAmount);
        vm.prank(alice);
        purchaseToken.approve(address(purchase), alicePurchaseAmount);
        vm.prank(alice);
        purchase.buy(address(token), alicePurchaseAmount, 700e18);
        
        // Verify state after purchase
        ArcTokenPurchase.TokenInfo memory info2 = purchase.getTokenInfo(address(token));
        assertEq(info2.amountSold, 700e18);
        assertEq(token.balanceOf(alice), 700e18);
        
        console.log("After Alice purchase - Contract balance:", token.balanceOf(address(purchase)) / 1e18, "tokens");
        
        // Replenish contract balance to 1000e18 (critical step!)
        uint256 remainingBalance = token.balanceOf(address(purchase));
        uint256 additionalNeeded = initialTokensForSale - remainingBalance;
        token.transfer(address(purchase), additionalNeeded);
        
        console.log("After refill - Contract balance:", token.balanceOf(address(purchase)) / 1e18, "tokens");
        
        // Admin resets tokenPrice to 200 USDC (innocent or malicious)
        vm.prank(owner);
        purchase.enableToken(address(token), initialTokensForSale, tokenPrice*2);
        
        // Verify reset state
        ArcTokenPurchase.TokenInfo memory info3 = purchase.getTokenInfo(address(token));
        assertEq(info3.amountSold, 0, "BUG: amountSold was reset to 0!");
        assertEq(info3.totalAmountForSale, initialTokensForSale);
        
        // Bob can purchase 1000 tokens again
        uint256 bobPurchaseAmount = 1000 * tokenPrice*2;
        purchaseToken.mint(bob, bobPurchaseAmount);
        vm.prank(bob);
        purchaseToken.approve(address(purchase), bobPurchaseAmount);
        vm.prank(bob);
        purchase.buy(address(token), bobPurchaseAmount, 1000e18);
        
        // Verify final state
        assertEq(token.balanceOf(alice), 700e18);
        assertEq(token.balanceOf(bob), 1000e18);
        
        // Total sold 1700 tokens, but initially only set 1000
        uint256 totalSold = token.balanceOf(alice) + token.balanceOf(bob);
        assertEq(totalSold, 1700e18);
    }
```

{% endcode %}
