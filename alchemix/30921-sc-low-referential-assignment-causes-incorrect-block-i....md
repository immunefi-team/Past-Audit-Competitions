# 30921 - \[SC - Low] Referential assignment causes incorrect block i...

Submitted on May 8th 2024 at 07:08:16 UTC by @NinetyNineCrits for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30921

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Due to referential assignment of struct variables, the block interpolation is done incorrectly. Since the blocknumbers of that struct are currently not used this does not have a direct impact, beyond storing incorrect values (which may cause issues on future use)

## Vulnerability Details

The `VotingEscrow._checkpoint` function does this assignment:

```solidity
Point memory initialLastPoint = lastPoint;
```

This behaves like references in other languages such as Java, where if you manipulate a property on either variables, you would change the data on the underlying instance and both variables would then use the new value. A minimal foundry test to showcase this:

```solidity
struct TestStruct {
    uint256 a;
    uint256 b;
}

function testReference() public {
    TestStruct memory testStruct = TestStruct(1, 2);
    TestStruct memory testStructReference = testStruct;

    testStruct.a = 3;
    console2.log("testStructReference.a", testStructReference.a); //@note prints 3
}
```

The same thing happens in `_checkpoint` where `lastPoint.ts` is changed and `initialLastPoint.ts` then uses that value:

```solidity
lastPoint.ts = _time;
lastPoint.blk = initialLastPoint.blk + (blockSlope * (_time - initialLastPoint.ts)) / MULTIPLIER;
```

The term `_time - initialLastPoint.ts` will now always be 0 and `lastPoint.blk` always be calculated incorrectly (at least if `_checkpoint` has not been called for more than a week).

A POC that logs the resulting values:

```js
// tested on fork number 19771835
function testMinhCheckpointing() public {
    uint256 week = 1 weeks;
    uint256 blocksPerWeek = 7*24*60*5; //@note assuming 1 block per 12 secs

    veALCX.checkpoint();

    vm.warp(block.timestamp + 3 * week);
    vm.roll(block.number + 3 * blocksPerWeek);

    veALCX.checkpoint();

    console2.log("point.blk for epoch 2:", veALCX.getPointHistory(2).blk);
    console2.log("point.blk for epoch 3:", veALCX.getPointHistory(3).blk);
    console2.log("point.blk for epoch 4:", veALCX.getPointHistory(4).blk);
    console2.log("point.blk for epoch 5:", veALCX.getPointHistory(5).blk);
}
```

This will log the following:

```
  point.blk for epoch 2: 19771835
  point.blk for epoch 3: 19771835
  point.blk for epoch 4: 19771835
  point.blk for epoch 5: 19923035
```

Note how the `blk` value only changes on the last iteration.

## Impact Details

As mentioned in the intro for now the impact is limited to incorrect calculations and storage of the result. The `blk` field is currently not in use, but is referenced in the unused `_findBlockEpoch` method, which would be affected on future use.

## Recommendation

Direct assignment doesnt copy, thats why Velodrome creates a new struct object explicitly:

```solidity
GlobalPoint memory initialLastPoint = GlobalPoint({
    bias: lastPoint.bias,
    slope: lastPoint.slope,
    ts: lastPoint.ts,
    blk: lastPoint.blk,
    permanentLockBalance: lastPoint.permanentLockBalance
});
```

## References

None

## Proof of Concept

Same test as in the description:

```js
// tested on fork number 19771835
function testMinhCheckpointing() public {
    uint256 week = 1 weeks;
    uint256 blocksPerWeek = 7*24*60*5; //@note assuming 1 block per 12 secs

    veALCX.checkpoint();

    vm.warp(block.timestamp + 3 * week);
    vm.roll(block.number + 3 * blocksPerWeek);

    veALCX.checkpoint();

    console2.log("point.blk for epoch 2:", veALCX.getPointHistory(2).blk);
    console2.log("point.blk for epoch 3:", veALCX.getPointHistory(3).blk);
    console2.log("point.blk for epoch 4:", veALCX.getPointHistory(4).blk);
    console2.log("point.blk for epoch 5:", veALCX.getPointHistory(5).blk);
}
```
