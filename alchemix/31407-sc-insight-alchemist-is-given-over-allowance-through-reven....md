# 31407 - \[SC - Insight] Alchemist is given over Allowance through Reven...

Submitted on May 18th 2024 at 12:35:45 UTC by @gladiator111 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31407

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

If the deposits.length=0 in `RevenueHandler.sol::claim` then over Allowance is given to the alchemist.

## Vulnerability Details

In the function `RevenueHandler.sol::claim`

```solidity
    // @audit unnecessary approval to Alchemist if deposits length is 0
    /// @inheritdoc IRevenueHandler
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

If the deposits.length==0, then unnecessary/undesirable allowance is given to the alchemist.

```solidity
 if (alchemists[alchemist] != address(0)) {
            require(token == IAlchemistV2(alchemist).debtToken(), "Invalid alchemist/alchemic-token pair");

            (, address[] memory deposits) = IAlchemistV2(alchemist).accounts(recipient);
@>          IERC20(token).approve(alchemist, amount);

            // Only burn if there are deposits
@>          // If deposits.length=0 then the allowance will not be used up
            amountBurned = deposits.length > 0 ? IAlchemistV2(alchemist).burn(amount, recipient) : 0;
        }
```

## Impact Details

Unnecessary/Over allowance will be given to the Alchemist.

## Suggestion/Recommendation

Update the function as follows

```diff
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
-            IERC20(token).approve(alchemist, amount);

            // Only burn if there are deposits
-            amountBurned = deposits.length > 0 ? IAlchemistV2(alchemist).burn(amount, recipient) : 0;

+           if(deposits.length>0){
+               IERC20(token).approve(alchemist, amount);
+               amountBurned = IAlchemistV2(alchemist).burn(amount, recipient);
+           } else {
+               amountBurned = 0;
+           }
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

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L210-L213>

## Proof of Concept

Paste the following code in `RevenueHandler.t.sol` and run using

```bash
forge test --match-test testGivenOverAllowance -vvvv --fork-url $FORK_URL
```

```solidity
    function testGivenOverAllowance() public {
        uint256 tokenId = _setupClaimableRevenue(1e18);
        uint256 claimable = revenueHandler.claimable(tokenId, alusd);
        uint256 givenAllowance = IERC20(alusd).allowance(address(revenueHandler), address(alusdAlchemist));
        assert(givenAllowance==0);
        revenueHandler.claim(tokenId, alusd, address(alusdAlchemist), claimable, address(this));
        givenAllowance = IERC20(alusd).allowance(address(revenueHandler), address(alusdAlchemist));
        assert(givenAllowance!=0);
        console.log(givenAllowance);     
    }
```
