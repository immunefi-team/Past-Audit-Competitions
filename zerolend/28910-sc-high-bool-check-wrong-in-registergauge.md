# 28910 - \[SC - High] Bool check wrong in registerGauge

Submitted on Mar 1st 2024 at 17:07:59 UTC by @offside0011 for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #28910

Report type: Smart Contract

Report severity: High

Target: <https://github.com/zerolend/governance>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

registerGauge function has a boolean value check written incorrectly, causing the pool to never be registered.

## Vulnerability Details

in the function registerGauge, the if bool check is wrong,

```
 mapping(address => bool) public isPool; // pool => bool


if (!isPool[_asset]) {
    _pools.push(_asset);
    isPool[_asset] = true;
}
```

```
// register the gauge in the factory
  const gauges = await factory.gauges(lending.erc20.target);
  await poolVoter.registerGauge(lending.erc20.target, gauges.splitterGauge);
```

## Impact Details

lead to pools will never be success registered

## References

<https://github.com/zerolend/governance/blob/main/contracts/voter/PoolVoter.sol#L136>

## Proof of concept

```
function testEXP() public {

    address owner = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266;
    deal(address(stake), address(owner), 1 ether);
    vm.startPrank(owner);
    poolVoter.registerGauge(address(1), address(11111));
    console.log(poolVoter.length());

    poolVoter.registerGauge(address(3), address(22222));
    console.log(poolVoter.length());

}
```
