# #44084 \[SC-Insight] Incorrect Nat spec in \`calcIBTsToTokenizeForCurvePool\` and \`calcIBTsToTokenizeForCurvePoolCustomProp\`

**Submitted on Apr 16th 2025 at 18:55:10 UTC by @MrMorningstar for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44084
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/libraries/CurvePoolUtil.sol>
* **Impacts:**

## Description

## Brief/Intro

The natspec of `calcIBTsToTokenizeForCurvePool` and `calcIBTsToTokenizeForCurvePoolCustomProp` says the functions return the amount of IBT that will be deposited in curve pools:

```
   /**
     * @notice Return the amount of IBT to deposit in the curve pool, given the total amount of IBT available for deposit
     * @param _amount The total amount of IBT available for deposit
     * @param _curvePool The address of the pool to deposit the amounts
     * @param _pt The address of the PT
     * @return ibts The amount of IBT which will be deposited in the curve pool
     */
    function calcIBTsToTokenizeForCurvePool(
        uint256 _amount,
        address _curvePool,
        address _pt
    ) external view returns (uint256 ibts) {
        (uint256 ibtBalance, uint256 ptBalance) = _getCurvePoolBalances(_curvePool);
        uint256 ibtBalanceInPT = IPrincipalToken(_pt).previewDepositIBT(ibtBalance);
        // Liquidity added in a ratio that (closely) matches the existing pool's ratio
        ibts = _amount.mulDiv(ptBalance, ibtBalanceInPT + ptBalance);
    }

    /**
     * @notice Return the amount of IBT to deposit in the curve pool given the proportion in which we want to deposit, given the total amount of IBT available for deposit
     * @param _amount The total amount of IBT available for deposit
     * @param _prop The proportion in which we want to make the deposit: _prop = nIBT / (nIBT + nPT)
     * @param _pt The address of the PT
     * @return ibts The amount of IBT which will be deposited in the curve pool
     */
    function calcIBTsToTokenizeForCurvePoolCustomProp(
        uint256 _amount,
        uint256 _prop,
        address _pt
    ) external view returns (uint256 ibts) {
        uint256 rate = IPrincipalToken(_pt).previewDepositIBT(_amount).mulDiv(CURVE_UNIT, _amount);
        ibts = _amount.mulDiv(CURVE_UNIT, CURVE_UNIT + _prop.mulDiv(rate, CURVE_UNIT));
    }
```

However those functions calculate ibt amount that will be deposited in PT not in curve which is based on proportion of ibt and pt token balance on curve in `calcIBTsToTokenizeForCurvePool` or on custom proportion in `calcIBTsToTokenizeForCurvePoolCustomProp` which further more we can prove that in the following code snippet where those functions are used:

```
    .
    .
    .
} else if (
            command == Commands.CURVE_SPLIT_IBT_LIQUIDITY ||
            command == Commands.CURVE_NG_SPLIT_IBT_LIQUIDITY ||
            command == Commands.CURVE_SPLIT_IBT_LIQUIDITY_SNG
        ) {
            (
                address pool,
                uint256 ibts,
                address recipient,
                address ytRecipient,
                uint256 minPTShares
            ) = abi.decode(_inputs, (address, uint256, address, address, uint256));
            recipient = _resolveAddress(recipient);
            ytRecipient = _resolveAddress(ytRecipient);
            address ibt = ICurvePool(pool).coins(0);
            address pt = ICurvePool(pool).coins(1);
            ibts = _resolveTokenValue(ibt, ibts);
            uint256 ibtToDepositInPT = CurvePoolUtil.calcIBTsToTokenizeForCurvePool(ibts, pool, pt);
            if (ibtToDepositInPT != 0) {
                bool isRegisteredPT = IRegistry(registry).isRegisteredPT(pt);
                if (isRegisteredPT) {
                    _ensureApproved(ibt, pt, ibtToDepositInPT);
                } else {
                    IERC20(ibt).forceApprove(pt, ibtToDepositInPT);
                }
                IPrincipalToken(pt).depositIBT(
                    ibtToDepositInPT,
                    recipient,
                    ytRecipient,
                    minPTShares
                );
                if (!isRegisteredPT) {
                    IERC20(ibt).forceApprove(pt, 0);
                }
            }
            if (recipient != address(this) && (ibts - ibtToDepositInPT) != 0) {
                IERC20(ibt).safeTransfer(recipient, ibts - ibtToDepositInPT);
            }
        } else if (
            command == Commands.CURVE_SPLIT_IBT_LIQUIDITY_CUSTOM_PROP ||
            command == Commands.CURVE_SPLIT_IBT_LIQUIDITY_CUSTOM_PROP_NG ||
            command == Commands.CURVE_SPLIT_IBT_LIQUIDITY_CUSTOM_PROP_SNG
        ) {
            (
                address pool,
                uint256 ibts,
                uint256 prop,
                address recipient,
                address ytRecipient,
                uint256 minPTShares
            ) = abi.decode(_inputs, (address, uint256, uint256, address, address, uint256));
            recipient = _resolveAddress(recipient);
            ytRecipient = _resolveAddress(ytRecipient);
            address ibt = ICurvePool(pool).coins(0);
            address pt = ICurvePool(pool).coins(1);
            ibts = _resolveTokenValue(ibt, ibts);
            uint256 ibtToDepositInPT = CurvePoolUtil.calcIBTsToTokenizeForCurvePoolCustomProp(
                ibts,
                prop,
                pt
            );
            if (ibtToDepositInPT != 0) {
                bool isRegisteredPT = IRegistry(registry).isRegisteredPT(pt);
                if (isRegisteredPT) {
                    _ensureApproved(ibt, pt, ibtToDepositInPT);
                } else {
                    IERC20(ibt).forceApprove(pt, ibtToDepositInPT);
                }
                IPrincipalToken(pt).depositIBT(
                    ibtToDepositInPT,
                    recipient,
                    ytRecipient,
                    minPTShares
                );
                if (!isRegisteredPT) {
                    IERC20(ibt).forceApprove(pt, 0);
                }
            }
            if (recipient != address(this) && (ibts - ibtToDepositInPT) != 0) {
                IERC20(ibt).safeTransfer(recipient, ibts - ibtToDepositInPT);
            }
        } else if (
            .
            .
            .
```

*We can see in the code above that returned amounts is deposited in PrincipalToken actually (which later can be deposited on curve but not necessarily)*

Therefore I believe the Nat spec is misinformative and could lead to incorrect expectations.

For those reasons I believe it should be changed.

## Recommendation

Rewrite function description to represent function accurately

## Proof of Concept

## Proof of Concept

PoC is not needed as this Insight falls under document and code improvements.
