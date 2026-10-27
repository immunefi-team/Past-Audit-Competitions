# 30667 - \[SC - Medium] Unlimited gauge numbers can DoS users distribut...

Submitted on May 3rd 2024 at 20:20:16 UTC by @Hoverfly9132 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30667

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Temporary freezing of funds for 12 hours

## Description

## Brief/Intro

`Voter#createGauge` allow users create unlimited gauges and distribute vote power to any numbers of alive gauges in `Voter#distribute`, once the gauges numbers reach a limitation, the users can't execute `Voter#distribute` any more.

## Vulnerability Details

Currently, there is no limit to `Voter#createGauge` function, as the doc [description](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/CONTRACTS.md#L27):

`- each veALCX tokenId may distribute their voting power across any number of gauges`, it means that any users can create any numbers of gauges and distribute vote power to them. Once the gauges be created, the pool will be recorded in gauges array, and in `Voter#distribute` function will traverse the array and then distribute vote power to them.

However, if the gauges is large enough, the gas cost may reach the ethereum/optimism gas limit, currently is 30\_000\_000, then such txs would be failed, which means users can't execute distribute vote power action any more, unless they remove the gauge, but there is no remove gauge function in this version, only have `killGauge` function to set the gauges to be not alive state.

## Impact Details

Users can't execute `Voter#distribute` any more when created gauges numbers is large enough.

## References

NA

## Proof of Concept

Because the needed gauges need cost much time in test case, so I will show distribute to different gauges would cost how much gas:

```solidity
function testExecutorCreateGaugeFrontRunDos() public {
    address attacker = address(0x2345);

    uint256 created_gauges = 1;
    hevm.startPrank(address(timelockExecutor));
    for (uint i = 0; i < created_gauges; i++) {
        voter.createGauge(address(uint160(i)), IVoter.GaugeType.Passthrough);
    }
    hevm.stopPrank();

    hevm.warp(block.timestamp + 3 weeks);
    uint256 gas_left = gasleft();
    voter.distribute();
    uint256 gas_after = gasleft();
    console.log("gas left: ", gas_left - gas_after);
}
```

When the `created_gauges` is equal to `1`, `Voter#distribute` gas cost is about \~1050000 wei:

```json
[PASS] testExecutorCreateGaugeFrontRunDos() (gas: 3607681)
Logs:
  gas left:  1054068
```

When the `created_gauges` is equal to `10`, `Voter#distribute` gas cost is about \~1154915 wei:

```json
[PASS] testExecutorCreateGaugeFrontRunDos() (gas: 27094489)
Logs:
  gas left:  1154915
```

When the `created_gauges` is equal to `20`, `Voter#distribute` gas cost is about \~1154915 wei:

```json
[PASS] testExecutorCreateGaugeFrontRunDos() (gas: 53190954)
Logs:
  gas left:  1266968
```

So distribute to more one gauge, gas cost is about: `(1266968 - 1154915) / 10 ~= 11205`, the max gauge in one tx is about `30000000 / 11205 ~= 2677`. Actually, the max gauge should less than 2677 because the distribute action will cost gas in other function. When set `created_gauges` to `2677`, the `Voter#distribute` function gas cost greater than `30_000_000` wei absolutely.
