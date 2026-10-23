# 31042 - \[SC - High] Claiming alchemic-token rewards can fail for so...

Submitted on May 11th 2024 at 11:50:44 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31042

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Temporary freezing of funds for 12 hours

## Description

## Description

When claiming an alchemic-token in `RevenueHandler`, the `claim(...)` function checks if a user has deposits in AlchemistV2 and attempts to pay his debt using the alchemic-token.

```javascript
function claim(
    uint256 tokenId,
    address token,
    address alchemist,
    uint256 amount,
    address recipient
) external override {

    // ----------------------- Code above is omitted for brevity

    // Get the deposits for the recipient
    (, address[] memory deposits) = IAlchemistV2(alchemist).accounts(recipient);
    IERC20(token).approve(alchemist, amount);

    // Only burn if there are deposits <-- wrong check, we should only burn if there is "debt"
    amountBurned = deposits.length > 0 ? IAlchemistV2(alchemist).burn(amount, recipient) : 0;

    // ----------------------- Code below is omitted for brevity

}
```

> Github Link: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L209-L213>

Users can have deposits in AlchemistV2 with no debt. Attempting to burn debt from an AlchemistV2 account that has 0 debt reverts the transaction, therefore **checking for deposits is wrong**, we should be checking for **debt** instead.

> (**int256 debt**, address\[] memory deposits) = IAlchemistV2(alchemist).accounts(recipient);

Users with deposits but no debt in AlchemistV2, when calling `RevenueHandler.claim(...)` with an alchemist address and an alchemic-token will have their *claim(...)* transaction reverted.

> Rewards are not permanently locked though. An advanced user can read the smart contract, detect the bug, and manually send a transaction to the revenueHandler smart contract with "*address(0)*" as the *alchemist* to bypass the bug and claim his reward.

## Impact Details

Claiming alchemic-token rewards can fail for some users

## Proof of Concept

Here's a test demonstrating the `revenueHandler.claim(...)` transaction reverting because the user has deposits but no debt.

```
    function testClaimAlchemicRevenue() external {
        revenueHandler.addAlchemicToken(address(alethAlchemist));

        uint256 revAmt = 1000e18;
        uint256 tokenId = _setupClaimableRevenue(revAmt);

        uint256 claimable = revenueHandler.claimable(tokenId, alusd);

        assertApproxEq(revAmt, claimable, revAmt / DELTA);

        deal(dai, address(this), 3 * 5000e18);
        IERC20(dai).approve(address(alusdAlchemist), 3 * 5000e18);
        alusdAlchemist.depositUnderlying(ydai, 3 * 5000e18, address(this), 0);

        revenueHandler.claim(tokenId, alusd, address(alusdAlchemist), claimable, address(this));

    }
```
