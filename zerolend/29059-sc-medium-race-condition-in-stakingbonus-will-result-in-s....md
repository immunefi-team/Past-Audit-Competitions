# 29059 - \[SC - Medium] Race condition in StakingBonus will result in s...

Submitted on Mar 6th 2024 at 01:38:00 UTC by @Trust for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29059

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/zerolend/governance>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

Holders of VestedZeroNFT tokens can lock them in exchange for boosted rewards through the StakingBonus contract.

## Vulnerability Details

When transferring tokens into StakingBonus, it's `onERC721Received()` function determines the bonus amount. If there's insufficient funds for the bonus, it sets bonus to zero.

```
function calculateBonus(
    uint256 amount
) public view override returns (uint256) {
    uint256 bonus = (amount * bonusBps) / 100;
    // if we don't have enough funds to pay out bonuses, then return 0
    if (zero.balanceOf(address(this)) < bonus) return 0;
    return (amount * bonusBps) / 100;
}
```

The lock is set for four years:

```
// stake for 4 years for the user
locker.createLockFor(
    pending + bonus, // uint256 _value,
    86400 * 365 * 4, // uint256 _lockDuration,
    to, // address _to,
    stake // bool _stakeNFT
);
```

Suppose there's 100 Zero remaining as bonus in the contract, the bonus % is 50%, and two NFT holders of 200 Zero pending wish to lock for the bonus. They will both see there's enough bonus and transfer their NFT. In fact, there is a race condition - only the first one which is executed will receive the bonus. Because of the architecture of blockchains, there's literally no way to know if the user's call will be frontrun, therefore it is not the fault of the user.

The `data` passed to `onERC721Received()` should include a `minBonus` amount, to prevent slippage (i.e. lack of bonus) being suffered by the user.

## Impact Details

A user will lock their tokens for 4 years unnecessarily, so they suffer from an unwanted freeze for lack of rewards.

## Proof of Concept

We have modified the test in StakingBonus.test.ts to show the issue:

```
it("giving bonus fails when another user snatches the bonus", async function () {
  // fund 10 Zero
  await zero.transfer(stakingBonus.target, e18 * 100n);
  // give a 50% bonus. For 20 Zero vest, it will be 10 Zero, the remaining in StakingBonus
  await stakingBonus.setBonusBps(50);
  // User 1's execution comes before User 2. They empty the stakingBonus Zero
  expect(
      await vest
          .connect(ant)
          ["safeTransferFrom(address,address,uint256)"](
          ant.address,
          stakingBonus.target,
          1
      )
  );
  // This one will not give any bonus, although user expected it when calling the function. It is now locker for no reason.
  expect(
      await vest
          .connect(ant2)
          ["safeTransferFrom(address,address,uint256)"](
          ant2.address,
          stakingBonus.target,
          2
      )
  );
  expect(await locker.balanceOfNFT(2)).eq(20);
});
```
