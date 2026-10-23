# 30999 - \[SC - Critical] An edge-case mints times more FLUX than it should

Submitted on May 10th 2024 at 12:45:39 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30999

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Protocol insolvency
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Background for Immunefi's triage team

In the 3rd interaction with ChainSecurity (referred to as `Version 3`) the `poke(...)` and `pokeTokens(...)` functions were added to Alchemix's codebase.

This report is about the `pokeTokens(...)` function.

Before diving right into it, we want to add context for Immunefi's triage team; below is ChainSecurity's overview of the new `pokeTokens(..)` function:

```
pokeTokens: introduced in Version 3, allows the admin to call the poke function on all veALCX tokens in the system. Note that we assume that this function is called by the keeper at the end of each epoch (otherwise users would be unable to vote).
```

> A link to the full PDF is on the Boost page.

Finally, here's Alchemix's overview of this function:

```
/**
    * @notice Update the voting status of multiple veALCXs to maintain the same voting status
    * @param _tokenIds Array of token IDs to poke
    * @dev Resets tokens that have expired
    */
function pokeTokens(uint256[] memory _tokenIds) external;
```

> Github Link: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/interfaces/IVoter.sol#L117-L122>

## Vulnerability Details

The `pokeTokens(...)` function is called by the **admin** to update the voting status of multiple veALCXs and reset expired tokens.

```
function pokeTokens(uint256[] memory _tokenIds) external {

    require(msg.sender == admin, "not admin");

    for (uint256 i = 0; i < _tokenIds.length; i++) {

        uint256 _tokenId = _tokenIds[I];

        // If the token has expired, reset it
        if (block.timestamp > IVotingEscrow(veALCX).lockEnd(_tokenId)) {
            reset(_tokenId);
        }

        poke(_tokenId);

    }

}
```

> Github Link: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol?#L215-L225>

The `reset(...)` function besides resetting the token, accrues FLUX rewards.

On top of maintaining the same voting status, the `poke(...)` function also accrues FLUX rewards.

As part of the system design, the amount of FLUX rewards that can be accrued in every epoch for a position that enabled "max lock" never decays.

> *Max lock is 1 year, if a user does not interact with the protocol for over a year his "max lock" position expires*

There's an edge case when using `pokeTokens(...)` to update the voting status of an expired position with max lock enabled. FLUX is accrued twice for the position, first inside `reset(...)` and then inside `poke(...)`.

## Impact Details

The impacts of minting more FLUX tokens than what the system was designed to are well known to the protocol.

FLUX is used to boost a veToken holder's voting power and exit a ve-position early, therefore edge cases that can break the protocol invariants and mint a bunch of additional FLUX will destabilize Alchemix's ecosystem.

## Proof of Concept

We modified the foundry test "**testVotingPowerDecay**" inside `alchemix-v2-dao/src/test/Voting.t.sol` to include this edge case.

Our test (named "**testVotingPowerDecayDoubleAccrue**") asserts that the user receives the correct amount of FLUX tokens.

When running this test in the current version of the codebase the user will receive 2x more FLUX tokens, therefore the test will "fail" until Alchemix fixes this edge case.

```
    function testVotingPowerDecayDoubleAccrue() public {
        // Kick off epoch cycle
        hevm.warp(newEpoch());
        voter.distribute();

        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, true);

        uint256[] memory tokens = new uint256[](2);
        tokens[0] = tokenId1;

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        // Vote and record used weights
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        // Move to the next epoch
        hevm.warp(newEpoch());
        voter.distribute();

        // Move to when token1 expire
        hevm.warp(block.timestamp + MAXTIME);

        // Amount of claimable flux for tokenId1 before poking
        uint256 claimableFLUX = IVotingEscrow(veALCX).claimableFlux(tokenId1);
        uint256 unclaimedFlux = flux.unclaimedFlux(tokenId1);

        // Mock poking idle tokens to sync voting
        hevm.prank(voter.admin());
        voter.pokeTokens(tokens);

        assertEq(flux.unclaimedFlux(tokenId1), unclaimedFlux + claimableFLUX, "claimed too much flux");

        console2.log("unclaimedFlux before poking", unclaimedFlux);
        console2.log("claimableFLUX before poking", claimableFLUX);
        console2.log("unclaimedFlux after poking", flux.unclaimedFlux(tokenId1));

    }
```
