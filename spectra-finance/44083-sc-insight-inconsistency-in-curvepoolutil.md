# #44083 \[SC-Insight] Inconsistency in \`CurvePoolUtil\`

**Submitted on Apr 16th 2025 at 18:50:25 UTC by @MrMorningstar for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44083
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/libraries/CurvePoolUtil.sol>
* **Impacts:**

## Description

## Brief/Intro

As we can see below the `previewRemoveLiquidity`, `previewRemoveLiquidityNG` and `previewRemoveLiquiditySNG` are very similiar and they have same purpose and that is to return expected amounts of IBT and PT withdrawn from curve pool:

```
    /**
     * @notice Returns the IBT and PT amounts received for burning a given amount of LP tokens
     * @notice Method to be used with legacy Curve Cryptoswap pools
     * @param _curvePool The address of the curve pool
     * @param _lpTokenAmount The amount of the lp token to burn
     * @return minAmounts The expected respective amounts of IBT and PT withdrawn from the curve pool
     */
    function previewRemoveLiquidity(
        address _curvePool,
        uint256 _lpTokenAmount
    ) external view returns (uint256[2] memory minAmounts) {
        address lpToken = ICurvePool(_curvePool).token();
        uint256 totalSupply = IERC20(lpToken).totalSupply();
        (uint256 ibtBalance, uint256 ptBalance) = _getCurvePoolBalances(_curvePool);
        // decrement following what Curve is doing
        if (_lpTokenAmount > APPROXIMATION_DECREMENT && totalSupply != 0) {
            _lpTokenAmount -= APPROXIMATION_DECREMENT;
            minAmounts = [
                (ibtBalance * _lpTokenAmount) / totalSupply,
                (ptBalance * _lpTokenAmount) / totalSupply
            ];
        } else {
            minAmounts = [uint256(0), uint256(0)];
        }
    }

    /**
     * @notice Returns the IBT and PT amounts received for burning a given amount of LP tokens
     * @notice Method to be used with Curve Cryptoswap NG pools
     * @param _curvePool The address of the curve pool
     * @param _lpTokenAmount The amount of the lp token to burn
     * @return minAmounts The expected respective amounts of IBT and PT withdrawn from the curve pool
     */
    function previewRemoveLiquidityNG(
        address _curvePool,
        uint256 _lpTokenAmount
    ) external view returns (uint256[2] memory minAmounts) {
        uint256 totalSupply = ICurveNGPool(_curvePool).totalSupply();
        (uint256 ibtBalance, uint256 ptBalance) = _getCurvePoolBalances(_curvePool);
        // reproduces Curve implementation
        if (_lpTokenAmount == totalSupply) {
            minAmounts = [ibtBalance, ptBalance];
        } else if (_lpTokenAmount > APPROXIMATION_DECREMENT && totalSupply != 0) {
            _lpTokenAmount -= APPROXIMATION_DECREMENT;
            minAmounts = [
                ibtBalance.mulDiv(_lpTokenAmount, totalSupply),
                ptBalance.mulDiv(_lpTokenAmount, totalSupply)
            ];
        } else {
            minAmounts = [uint256(0), uint256(0)];
        }
    }

    /**
     * @notice Returns the IBT and PT amounts received for burning a given amount of LP tokens
     * @notice Method to be used with StableSwap NG pools
     * @param _curvePool The address of the curve pool
     * @param _lpTokenAmount The amount of the lp token to burn
     * @return minAmounts The expected respective amounts of IBT and PT withdrawn from the curve pool
     */
    function previewRemoveLiquiditySNG(
        address _curvePool,
        uint256 _lpTokenAmount
    ) external view returns (uint256[] memory) {
        uint256 totalSupply = IERC20(_curvePool).totalSupply();
        (uint256 ibtBalance, uint256 ptBalance) = _getCurvePoolBalances(_curvePool);
        // decrement following what Curve is doing
        uint256[] memory minAmounts = new uint256[](2);
        if (_lpTokenAmount > APPROXIMATION_DECREMENT && totalSupply != 0) {
            _lpTokenAmount -= APPROXIMATION_DECREMENT;
            minAmounts[0] = (ibtBalance * _lpTokenAmount) / totalSupply;
            minAmounts[1] = (ptBalance * _lpTokenAmount) / totalSupply;
        } else {
            minAmounts[0] = 0;
            minAmounts[1] = 0;
        }
        return minAmounts;
    }
```

However only `previewRemoveLiquidityNG` use mulDiv function when calculating min amounts while other two don't.

It is consider best practice to use mulDiv as it is more secure and more precise when doing calculations as it improve accuracy because it reduce rounding errors by combining the operations more precisely, especially when working with large numbers.

## Recommendation

Use `mulDiv` when calculating `minAmounts` in `previewRemoveLiquiditySNG` and `previewRemoveLiquidity`

## Proof of Concept

## Proof of Concept

PoC not needed as this is Insight under Security best practices and Code Optimizations and Enhancements category.
