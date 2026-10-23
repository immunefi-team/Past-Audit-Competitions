# 28992 - \[SC - High] Permanent freezing of additional reward tokens

Submitted on Mar 4th 2024 at 04:59:12 UTC by @MahdiKarimi for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #28992

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The `registerGuage` function never adds pool address to \_pool array and set isPool mapping, it would block `distributeEx` function which leads to permanent freezing of additional reward tokens.

## Vulnerability Details

In `registerGuage`there is an if statement as follows and if it's true it would add \_asset to the pool array and set isPool to true, since isPool have been set only here, this if statement always will be false and this piece of code never runs.

```
if (isPool[_asset]) {
            _pools.push(_asset);
            isPool[_asset] = true;
        }
```

this means registering a gauge won't add underlying pool to \_pool array and \_pool array has no value. `distributeEx` function uses \_pool array to distribute additional reward tokens, it iterates through \_pool array and transfers rewards to pool gauges, since \_pool array is empty, there is no way to transfer additional reward tokens and funds get blocked in PoolVoter contract .

## Impact Details

This leads to a situation which `distributeEx` is unable to distribute additional rewards, and permanent freezing of additional reward tokens.

## References

<https://github.com/zerolend/governance/blob/a30d8bb825306dfae1ec5a5a47658df57fd1189b/contracts/voter/PoolVoter.sol#L136-L139>

## Proof of Concept

```
  describe("freezing of additional reward tokens", () => {
    it("distributeEx is unable to distribute additional rewaard tokens", async function () {

    // so there is no differece 
    // some parts has been added to deployement as follow
    // a new lending pool has been deployed during deployment and its gauge has been registered at pool voter during deployment 
    // some zero tokens has been transferred to ant.address, so ant.address have enough zero token balance 

    // voting to both gauges equally 
    let vote1 = 1e8/2;
    let vote2 = 1e8/2; 
    await poolVoter.connect(ant).vote([reserve.target, reserve2.target],[vote1, vote2]);
   
    // we consider zero is additional reward token of the pool voter ( not main reward token ) 
    // transferring rewards to pool voter 
    await zero.connect(ant).transfer(poolVoter.target, 1e6);

    // retrieve pool voter contract balance 
    let balance = await zero.balanceOf(poolVoter);

    // distributeEx function wouldn't distribute any token, since _pool array is empty 
    await poolVoter["distributeEx(address)"](zero.target);

    // there is no change at balance of contract meaning no distribution happened 
    let balanceAfter = await zero.balanceOf(poolVoter);
    expect(balanceAfter).eq(balance);

    });
    
  });
```
