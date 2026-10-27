# 31430 - \[SC - Insight] QA

Submitted on May 19th 2024 at 01:07:31 UTC by @imsrybr0 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31430

Report type: Smart Contract

Report severity: Insight

Target: <https://immunefi.com>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

### `VotingEscrow` cannot be deployed

**Issue Description:**

The contract size (27031) currently exceeds the maximum allowed code size.

### `RevenueHandler@claim` might leave unused approvals

**Issue Description:**

This can happen if :

* There are no deposits.
* Debt is lower than the approved amount.

[RevenueHandler.sol](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol#L186-L225)

```solidity
    function claim(
        uint256 tokenId,
        address token,
        address alchemist,
        uint256 amount,
        address recipient
    ) external override {
        // ...
        if (alchemists[alchemist] != address(0)) {
            require(token == IAlchemistV2(alchemist).debtToken(), "Invalid alchemist/alchemic-token pair");

            (, address[] memory deposits) = IAlchemistV2(alchemist).accounts(recipient);
            IERC20(token).approve(alchemist, amount); // <==== Audit

            // Only burn if there are deposits
            amountBurned = deposits.length > 0 ? IAlchemistV2(alchemist).burn(amount, recipient) : 0;
        }
        // ...
    }
```

### `Bribe@getRewardForOwner` writes the same checkpoint in a loop

**Issue Description**

When multiple tokens are claimed, the same check point (same balance, same timestamp) will be written multiple times (up to the number of claimed tokens).

[Bribe.sol](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L283-L300)

```solidity
    function getRewardForOwner(uint256 tokenId, address[] memory tokens) external lock {
        // ...
        for (uint256 i = 0; i < length; i++) {
            // ...
            _writeCheckpoint(tokenId, balanceOf[tokenId]); // <==== Audit
			// ...
        }
    }
    
    function _writeCheckpoint(uint256 tokenId, uint256 balance) internal {
        uint256 _timestamp = block.timestamp;
        uint256 _nCheckPoints = numCheckpoints[tokenId];
        if (_nCheckPoints > 0 && checkpoints[tokenId][_nCheckPoints - 1].timestamp == _timestamp) { // <==== Audit
            checkpoints[tokenId][_nCheckPoints - 1].balanceOf = balance;
        } else {
            checkpoints[tokenId][_nCheckPoints] = Checkpoint(_timestamp, balance);
            numCheckpoints[tokenId] = _nCheckPoints + 1;
        }
    }
```

### `Voter@claimBribes` does not check if the bribe is valid

**Issue Description**

`_bribes[i]` is a user input and can be any contract that implements `getRewardForOwner`.

[Voter.sol](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L332-L338)

```solidity
    function claimBribes(address[] memory _bribes, address[][] memory _tokens, uint256 _tokenId) external {
        require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId));

        for (uint256 i = 0; i < _bribes.length; i++) {
            IBribe(_bribes[i]).getRewardForOwner(_tokenId, _tokens[i]); // <=== Audit
        }
    }
```

### `staleThreshold` is 60 days in `RewardDistributor@amountToCompound`

**Issue Description**

It seems that the stale threshold was updated for testing purposes.

[RewardsDistributor.sol](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol#L116-L133)

```solidity
    function amountToCompound(uint256 _alcxAmount) public view returns (uint256, uint256[] memory) {
        // Increased for testing since tests go into future
        uint256 staleThreshold = 60 days; // <==== Audit

        (uint80 roundId, int256 alcxEthPrice, , uint256 priceTimestamp, uint80 answeredInRound) = priceFeed
            .latestRoundData();

        require(answeredInRound >= roundId, "Stale price");
        require(block.timestamp - priceTimestamp < staleThreshold, "Price is stale");
        require(alcxEthPrice > 0, "Chainlink answer reporting 0");

        uint256[] memory normalizedWeights = IManagedPool(address(balancerPool)).getNormalizedWeights();

        uint256 amount = (((_alcxAmount * uint256(alcxEthPrice)) / 1 ether) * normalizedWeights[0]) /
            normalizedWeights[1];

        return (amount, normalizedWeights);
    }
```

### Rewards will be lost for a token if it's merged before claiming them

**Issue Description**

`VotingEscrow@merge` doesn't claim rewards from `RewardsDistributor` for the merged token before burning it. This makes it impossible to claim the rewards if it wasn't done before merging since the approval check will fail.

[VotingEscrow.sol](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L618-L651)

```solidity
    function merge(uint256 _from, uint256 _to) external {
        require(!voted[_from], "voting in progress for token");
        require(_from != _to, "must be different tokens");
        require(_isApprovedOrOwner(msg.sender, _from), "not approved or owner");
        require(_isApprovedOrOwner(msg.sender, _to), "not approved or owner");

        LockedBalance memory _locked0 = locked[_from];
        LockedBalance memory _locked1 = locked[_to];

        // Cannot merge if cooldown is active or lock is expired
        require(_locked0.cooldown == 0, "Cannot merge when cooldown period in progress");
        require(_locked1.cooldown == 0, "Cannot merge when cooldown period in progress");
        require(_locked0.end > block.timestamp, "Cannot merge when lock expired");
        require(_locked1.end > block.timestamp, "Cannot merge when lock expired");

        uint256 value0 = uint256(_locked0.amount);

        // If max lock is enabled retain the max lock
        _locked1.maxLockEnabled = _locked0.maxLockEnabled ? _locked0.maxLockEnabled : _locked1.maxLockEnabled;

        IFluxToken(FLUX).mergeFlux(_from, _to);

        // If max lock is enabled end is the max lock time, otherwise it is the greater of the two end times
        uint256 end = _locked1.maxLockEnabled
            ? ((block.timestamp + MAXTIME) / WEEK) * WEEK
            : _locked0.end >= _locked1.end
            ? _locked0.end
            : _locked1.end;

        locked[_from] = LockedBalance(0, 0, false, 0);
        _checkpoint(_from, _locked0, LockedBalance(0, 0, false, 0));
        _burn(_from, value0);
        _depositFor(_to, value0, end, _locked1.maxLockEnabled, _locked1, DepositType.MERGE_TYPE);
    }
```

[RewardsDistributor.sol](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol#L151-L210)

```solidity
    function claim(uint256 _tokenId, bool _compound) external payable nonReentrant returns (uint256) {
        if (!_compound) {
            require(msg.value == 0, "Value must be 0 if not compounding");
        }

        bool approvedOrOwner = IVotingEscrow(votingEscrow).isApprovedOrOwner(msg.sender, _tokenId);
        bool isVotingEscrow = msg.sender == votingEscrow;

        require(approvedOrOwner || isVotingEscrow, "not approved"); // <==== Audit
		// ...
    }
```

### `Voter@pokeTokens` will fail when the poked token lock is expired

**Issue Description**

It will fail at the reset step if the token already voted in the current epoch, either normally or by the owner front running the `pokeTokens` transaction with a reset.

It will fail at the poke (poke -> vote) step because the voting power of the token will be 0.

[Voter.sol](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L215-L225)

```solidity
    function pokeTokens(uint256[] memory _tokenIds) external {
        // ...
        for (uint256 i = 0; i < _tokenIds.length; i++) {
            uint256 _tokenId = _tokenIds[i];
            // If the token has expired, reset it
            if (block.timestamp > IVotingEscrow(veALCX).lockEnd(_tokenId)) {
                reset(_tokenId); // <==== Audit
            }
            poke(_tokenId); // <==== Audit
        }
    }

	function poke(uint256 _tokenId) public {
        // Previous boost will be taken into account with weights being pulled from the votes mapping
        uint256 _boost = 0;
        // ...
        _vote(_tokenId, _poolVote, _weights, _boost); // <==== Audit
    }

    function reset(uint256 _tokenId) public onlyNewEpoch(_tokenId) { // <==== Audit
    }

	function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
        // ...

        uint256 totalPower = (IVotingEscrow(veALCX).balanceOfToken(_tokenId) + _boost); // <==== Audit

        for (uint256 i = 0; i < _poolCnt; i++) {
            // ...

            uint256 _poolWeight = (_weights[i] * totalPower) / _totalVoteWeight; // <==== Audit
            require(votes[_tokenId][_pool] == 0, "already voted for pool");
            require(_poolWeight != 0, "cannot vote with zero weight"); // <==== Audit
			// ...
        }
    }
```

### Boost is not carried over in `Voter` when poking

**Issue Description**

The following comment in the `Voter@poke` function suggests that boosts is carried over when poking :

> // Previous boost will be taken into account with weights being pulled from the votes mapping

However this is not the case.

For example, a user votes on a single pool with a weight of X using a token which has a voting power of Y and uses a boost of Z :

* `_totalVoteWeight` will be equal to X
* `totalPower` will be equal to Y + Z
* `_poolWeight` will be equal to X \* (Y + Z) / X = Y + Z

Now the user pokes using that same token :

* `_totalVoteWeight` will be equal to Y + Z
* `totalPower` will be equal to Y + 0 = Y
* `_poolWeight` will be equal to (Y + Z) \* (Y) / (Y + Z) = Y

The weights stay proportional to the initial vote weights but the boost is lost.

[Voter.sol](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L195-L212)

```solidity
    function poke(uint256 _tokenId) public {
        // Previous boost will be taken into account with weights being pulled from the votes mapping
        uint256 _boost = 0;

        // ...

        address[] memory _poolVote = poolVote[_tokenId];
        uint256 _poolCnt = _poolVote.length;
        uint256[] memory _weights = new uint256[](_poolCnt);

        for (uint256 i = 0; i < _poolCnt; i++) {
            _weights[i] = votes[_tokenId][_poolVote[i]];
        }

        _vote(_tokenId, _poolVote, _weights, _boost); // <==== Audit
    }

    function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
        _reset(_tokenId);

        uint256 _poolCnt = _poolVote.length;
        uint256 _totalVoteWeight = 0;
        uint256 _totalWeight = 0;

        for (uint256 i = 0; i < _poolCnt; i++) {
            _totalVoteWeight += _weights[i]; // <==== Audit
        }

        IFluxToken(FLUX).accrueFlux(_tokenId);
        uint256 totalPower = (IVotingEscrow(veALCX).balanceOfToken(_tokenId) + _boost); // <==== Audit

        for (uint256 i = 0; i < _poolCnt; i++) {
            address _pool = _poolVote[i];
            address _gauge = gauges[_pool];

            require(isAlive[_gauge], "cannot vote for dead gauge");

            uint256 _poolWeight = (_weights[i] * totalPower) / _totalVoteWeight; // <==== Audit
            // ...
            votes[_tokenId][_pool] += _poolWeight;
            // ...
        }

        // ...
    }
```

### There is no link between the `Minter` and `Voter` epochs

**Issue Description**

There is no link between the `Minter` and `Voter` epochs even if they share the same duration.

This means that voters can vote as soon as possible even before a distribution.

In this case, total votes will be reset to 0 in bribes discarding previous voters and leading to the rewards being split based on future voters but still be distributed to all of them. Not all of them will able to claim though (fewer voters are accounted for mean more rewards per voter).

[Voter.sol](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L341-L380)

```solidity
    modifier onlyNewEpoch(uint256 _tokenId) {
        // Ensure new epoch since last vote
        require((block.timestamp / DURATION) * DURATION > lastVoted[_tokenId], "TOKEN_ALREADY_VOTED_THIS_EPOCH"); // <=== Audit
        _;
    }

	function vote(
        uint256 _tokenId,
        address[] calldata _poolVote,
        uint256[] calldata _weights,
        uint256 _boost
    ) external onlyNewEpoch(_tokenId) { // <=== Audit
	    // ...
    }
    
    function distribute() external {
        IMinter(minter).updatePeriod(); // <=== Audit
    }

    /*
        Internal functions
    */

    function _distribute(address _gauge) internal {
        // ...

        IBribe(bribes[_gauge]).resetVoting(); // <=== Audit

        // ...
    }
```

[Bribe.sol](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L332-L335)

```solidity
    function resetVoting() external {
        require(msg.sender == voter);
        totalVoting = 0;
    }
```

### Proposals can be created with 0 voting power

**Issue Description**

* veALCX supply starts at 0
* veALCX supply is dynamic
* The proposer threshold (amount needed to be able to create a proposal) is checked at the proposal creation time based on the veALCX balance of the creator and the veALCX total supply at that time :

```solidity
    function propose(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description,
        uint256 chainId
    ) public virtual override returns (uint256) {
        require(
            getVotes(_msgSender(), block.timestamp - 1) >= proposalThreshold(),
            "Governor: veALCX power below proposal threshold"
        );

        // ...
	    proposal.voteStart.setDeadline(block.timestamp.toUint64() + votingDelay.toUint64());
		// ...
	}

    function proposalThreshold() public view override(L2Governor) returns (uint256) {
        return (token.getPastTotalSupply(block.timestamp) * proposalNumerator) / PROPOSAL_DENOMINATOR;
    }
```

* The proposal quorum threshold is checked against the total supply at the start of the voting period :

```solidity
    function proposalSnapshot(uint256 proposalId) public view virtual override returns (uint256) {
        return _proposals[proposalId].voteStart.getDeadline();
    }
    
	function _quorumReached(uint256 proposalId) internal view virtual override returns (bool) {
        ProposalVote storage proposalvote = _proposalVotes[proposalId];

        return quorum(proposalSnapshot(proposalId)) <= proposalvote.forVotes + proposalvote.abstainVotes;
    }
	
    function quorum(uint256 blockTimestamp) public view virtual override returns (uint256) {
        return (token.getPastTotalSupply(blockTimestamp) * quorumNumerator()) / quorumDenominator();
    }
```

* A voter's power is calculated based on the start of the voting period :

```solidity
    function _castVote(
        uint256 proposalId,
        address account,
        uint8 support,
        string memory reason,
        bytes memory params
    ) internal virtual returns (uint256) {
        // ...
        uint256 weight = _getVotes(account, proposal.voteStart.getDeadline(), params);
        _countVote(proposalId, account, support, weight, params);
		// ...
    }

	function _getVotes(
        address account,
        uint256 blockTimestamp,
        bytes memory /*params*/
    ) internal view virtual override returns (uint256) {
        return token.getPastVotes(account, blockTimestamp);
    }
```

This would allow to create a proposal with 0 voting power as the proposal threshold is initially 0 when veALCX total voting power is still 0 (or decays to 0). This may also lead to easily pass a proposal depending on the veALCX voting power created up to the start of the voting period.

### Reentrancy guard modifier is named differently across different contracts

**Issue Description**

| Contract           | Name         | Custom Implementation             |
| ------------------ | ------------ | --------------------------------- |
| VotingEscrow       | nonreentrant | Yes                               |
| Voter              | lock         | Yes                               |
| RewardsDistributor | nonReentrant | No (OpenZeppelin ReentrancyGuard) |
| Bribe              | lock         | Yes                               |
| BaseGauge          | lock         | Yes                               |

## Proof of Concept

QA report
