# 31309 - \[SC - Critical] slippage protection is inaccurate

Submitted on May 16th 2024 at 21:08:22 UTC by @jasonxiale for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31309

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

`RevenueHandler._melt` is used to swap normal token to alAsset(for example: WETH->alETH), but the slippage protection is inaccurate. So the function is subject to sandwich attack.

## Vulnerability Details

Function [RevenueHandler.\_melt](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L275-L295) is used to swap normal token to alAsset(for example: WETH -> alETH), and the function uses `IERC20(revenueToken).balanceOf(address(this));` as slippage protection.

The issue is that the price between normal token and alAsset isn't 1:1, based on [althe](https://www.coingecko.com/en/coins/alchemix-eth) and [weth](https://www.coingecko.com/en/coins/weth), the ratio between WETH and alETH is about 30:27. So it means that the protocol will use 3000$ WETH to swap for 2700$ alETH. Acutally, this is the same issue as <https://github.com/sherlock-audit/2024-04-alchemix-judging/issues/5>

```solidity
275     function _melt(address revenueToken) internal returns (uint256) {
276         RevenueTokenConfig storage tokenConfig = revenueTokenConfigs[revenueToken];
277         address poolAdapter = tokenConfig.poolAdapter;
278         uint256 revenueTokenBalance = IERC20(revenueToken).balanceOf(address(this));
279         if (revenueTokenBalance == 0) {
280             return 0;
281         }
282         IERC20(revenueToken).safeTransfer(poolAdapter, revenueTokenBalance);
283         /*  
284             minimumAmountOut == inputAmount
285             Here we are making the assumption that the price of the alAsset will always be at or below the price of the revenue token.
286             This is currently a safe assumption since this imbalance has always held true for alUSD and alETH since their inceptions.
287         */
288         return
289             IPoolAdapter(poolAdapter).melt(
290                 revenueToken,
291                 tokenConfig.debtToken,
292                 revenueTokenBalance,
293                 revenueTokenBalance <<<--- Here IERC20(revenueToken).balanceOf(address(this)) is used as slippage protection.
294             );
295     }
```

## Impact Details

`RevenueHandler._melt` is used to swap normal token to alAsset(for example: WETH->alETH), but the slippage protection is inaccurate. So the function is subject to sandwich attack.

## References

<https://github.com/sherlock-audit/2024-04-alchemix-judging/issues/5>

## Proof of Concept

Because there is no WETH/ALETH Curve pool onchain, I will use ETH/ALETH pool as example.

Add the following code in `src/test/RevenueHandler.t.sol` and run

```bash
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/0TbY2mhyGA4gLPShfh-PwBlQ3PDNUdL1 --fork-block-number 17133822 --mc RevenueHandlerTest --mt testSlippageIssue -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/RevenueHandler.t.sol:RevenueHandlerTest
[PASS] testSlippageIssue() (gas: 137715)
Logs:
  poolAdapter                     : 0xC4C319E2D4d66CcA4464C0c2B32c9Bd23ebe784e
  aleth                           : 0x0100546F2cD4C9D97f798fFC9755E47865FF7Ee6
  val                             : 1012512998195120273

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.42ms (231.43µs CPU time)

Ran 1 test suite in 1.34s (4.42ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

As we can see above, by using 1e18 weth, we can exchage 1012512998195120273 aleth. But if we set [minimumAmountOut == inputAmount](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L288-L294). Sandwich attack might happen

```solidity
    function testSlippageIssue() external {
        address aleth_eth = address(0xC4C319E2D4d66CcA4464C0c2B32c9Bd23ebe784e);
        revenueHandler.addRevenueToken(address(weth));
        revenueHandler.setDebtToken(address(weth), aleth);
        revenueHandler.setPoolAdapter(address(weth), aleth_eth);
        (, address poolAdapter, ) = revenueHandler.revenueTokenConfigs(address(weth));
        assertEq(poolAdapter, aleth_eth);
        console2.log("poolAdapter                     :", poolAdapter);
        console2.log("aleth                           :", address(aleth));
        int128 i = 0;
        int128 j = 1;
        uint val = ICurveStableSwap(poolAdapter).get_dy(i, j, 1e18);
        console2.log("val                             :", val);
    }
```
