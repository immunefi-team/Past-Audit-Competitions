# 31385 - \[SC - Low] RewardsDistributortokensPerWeek might be zero i...

Submitted on May 17th 2024 at 22:21:00 UTC by @jasonxiale for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31385

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

`RewardsDistributor.tokensPerWeek` is used to record the amount of alcx to distribute per week, if its value is zero, it means there is no alcx will be distributed. In current implementation, there will be an extreme case that if the `RewardsDistributor.checkpointToken` isn't called for more than 20 weeks, some weeks in the past will has empty `RewardsDistributor.tokensPerWeek`

## Vulnerability Details

In [RewardsDistributor.\_checkpointToken](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RewardsDistributor.sol#L226-L254), the function will update `lastTokenTime` to `block.timestamp` first, and then loop 20 WEEK in for-loop

```solidity
227     function _checkpointToken() internal {
...
232         uint256 t = lastTokenTime;
233         uint256 sinceLast = block.timestamp - t;
234         lastTokenTime = block.timestamp; <<<--- lastTokenTime is update to block.timestamp
235         uint256 thisWeek = (t / WEEK) * WEEK;
236         uint256 nextWeek = 0;
237 
238         for (uint256 i = 0; i < 20; i++) { <<<--- here 20 is used, it means that the function will loop 20 weeks at most
...
252         }
254     }
```

**And next time when `RewardsDistributor.checkpointToken` is called, the function record `RewardsDistributor.tokensPerWeek` from the timestamp the function is called instead of the `RewardsDistributor.tokensPerWeek` hasn't been recordeded.** So if the `RewardsDistributor.checkpointToken` hasn't been called in more than 20 weeks, the `RewardsDistributor.tokensPerWeek` will be like: RewardsDistributor.tokensPerWeek\[week\_00] -> value\_00 RewardsDistributor.tokensPerWeek\[week\_01] -> value\_01 RewardsDistributor.tokensPerWeek\[week\_02] -> value\_02 ... RewardsDistributor.tokensPerWeek\[week\_19] -> value\_19 RewardsDistributor.tokensPerWeek\[week\_20] -> 0 RewardsDistributor.tokensPerWeek\[week\_21] -> 0 RewardsDistributor.tokensPerWeek\[week\_22] -> 0 ... RewardsDistributor.tokensPerWeek\[week\_nn] -> value\_nn <<<<--- next RewardsDistributor.checkpointToken is called, the `RewardsDistributor.tokensPerWeek` will be updated from here RewardsDistributor.tokensPerWeek\[week\_nm] -> value\_nm

## Impact Details

Because `RewardsDistributor.tokensPerWeek` is used to calcuate the amount of alcx a user can claim, if its value is 0, it means there will be no alcx can be claim.

## References

Add any relevant links to documentation or code

## Proof of Concept

Put the following code in `src/test/Minter.t.sol`, and run

```bash
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/0TbY2mhyGA4gLPShfh-PwBlQ3PDNUdL1 --fork-block-number 17133822 --mc MinterTest --mt testNoEmissions -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/Minter.t.sol:MinterTest
[PASS] testNoEmissions() (gas: 5267453)
Logs:
  block.timestamp    0      : 1684972800
  rd.lastTokenTime()        : 1682553600
  block.timestamp    1      : 1700092801
  rd.lastTokenTime()        : 1684972800
  ===========================
  rd.tokensPerWeek          : 0
  rd.tokensPerWeek          : 0
  rd.tokensPerWeek          : 0
  rd.tokensPerWeek          : 0
  rd.tokensPerWeek          : 0
  rd.tokensPerWeek          : 0
  rd.tokensPerWeek          : 324063978567197184709
  rd.tokensPerWeek          : 324063978567197184709
  rd.tokensPerWeek          : 324063978567197184709
  rd.tokensPerWeek          : 324063978567197184709
  ===========================
  block.timestamp           : 1701302402
  rd.tokensPerWeek          : 4008540058167941329413
  rd.tokensPerWeek          : 0
  rd.tokensPerWeek          : 0
  rd.tokensPerWeek          : 0
  rd.tokensPerWeek          : 0
  rd.tokensPerWeek          : 0
  rd.tokensPerWeek          : 324063978567197184709
  rd.tokensPerWeek          : 324063978567197184709
  rd.tokensPerWeek          : 324063978567197184709
  rd.tokensPerWeek          : 324063978567197184709

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.80ms (1.44ms CPU time)

Ran 1 test suite in 1.37s (6.80ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

As we can see from the above output, after two `voter.distribute();` calls, there still some `rd.tokensPerWeek` contains 0

```solidity
    function testNoEmissions() external {
        // Mint emissions for the amount of epochs until tail emissions target
        uint WEEK = 1 weeks;
        RewardsDistributor rd = RewardsDistributor(payable(address(minter.rewardsDistributor())));
        // console2.log("block.timestamp           :", block.timestamp);
        uint startTime = IMinter(minter).activePeriod();

        hevm.warp(startTime + IMinter(minter).DURATION());
        console2.log("block.timestamp    0      :", block.timestamp);
        console2.log("rd.lastTokenTime()        :", rd.lastTokenTime());
        voter.distribute();

        hevm.warp(block.timestamp + 25 * WEEK + 1);
        console2.log("block.timestamp    1      :", block.timestamp);
        console2.log("rd.lastTokenTime()        :", rd.lastTokenTime());

        voter.distribute();
        uint index = block.timestamp / WEEK * WEEK;
        console2.log("===========================");
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 0 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 1 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 2 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 3 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 4 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 5 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 6 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 7 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 8 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 9 * WEEK));

        console2.log("===========================");
        hevm.warp(block.timestamp + 2 * WEEK + 1);
        voter.distribute();
        console2.log("block.timestamp           :", block.timestamp);
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 0 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 1 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 2 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 3 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 4 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 5 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 6 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 7 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 8 * WEEK));
        console2.log("rd.tokensPerWeek          :", rd.tokensPerWeek(index - 9 * WEEK));
    }
```
