# Boost \_ Folks Finance 34148 - \[Smart Contract - Low] Full liquidations will fail for certain unhealt

Submitted on Tue Aug 06 2024 00:48:52 GMT-0400 (Atlantic Standard Time) by @JCN2023 for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34148

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xc1FBF54B25816B60ADF322d8A1eaCA37D9A50317>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Bug Description

Below I will explain an edge case in which a full liquidation (all debt to be repaid) will incorrectly calculate the amount to repay, resulting in an underflow error.

Below is an example state that I will use to explain the bug. For simple calculations, we will assume the violator has one debt position and one collateral position in the same asset and that the asset has a price of $1:

```
- asset_price = 1e18
- debt_balance = 95e6
- collateral_balance = 100e6
- liquidation_bonus = 800
```

Below is the section of code that calculates the amount to repay and amount of collateral to seize (see line 461 below). Next, the violator's debt balance will be reduced by the repay amount, which can be greater than the debt balance, causing an underflow error (see line 478 below).

[LoanManagerLogic::executeLiquidate](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LoanManagerLogic.sol#L461-L483)

```solidity
461:        DataTypes.LiquidationAmountParams memory liquidationAmounts = loansParams.calcLiquidationAmounts( // @audit: repay amount and collateral seized calculated
462:            userLoans,
463:            loanTypes,
464:            pools[loansParams.collateralPoolId],
465:            params.oracleManager,
466:            maxRepayBorrowValue,
467:            params.maxRepayAmount
468:        );
469:
470:        loansParams.updateLiquidationRewards(userLoans, loanTypes, userPoolRewards);
471:
472:        DataTypes.LiquidationBorrowTransfer memory liquidationBorrowTransfer;
473:        {
474:            // pool pre-checks and update interest indexes
475:            DataTypes.BorrowPoolParams memory borrowPoolParams = pools[params.borPoolId].preparePoolForRepay();
476:
477:            // transfer borrow from violator to liquidator
478:            liquidationBorrowTransfer = loansParams.updateLiquidationBorrows(
479:                borrowPoolParams,
480:                liquidationAmounts.repayBorrowAmount, // @audit: repay amount > actual debt balance
481:                userLoans
482:            );
483:        }
```

First we will examine the `calcLiquidationAmounts` function:

[LiquidationLogic::calcLiquidationAmounts](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LiquidationLogic.sol#L185-L221)

```solidity
185:    function calcLiquidationAmounts(
186:        DataTypes.LiquidationLoansParams memory loansParams,
187:        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans,
188:        mapping(uint16 => LoanManagerState.LoanType) storage loanTypes,
189:        IHubPool collPool,
190:        IOracleManager oracleManager,
191:        uint256 maxRepayBorrowValue,
192:        uint256 maxAmountToRepay
193:    ) external view returns (DataTypes.LiquidationAmountParams memory liquidationAmounts) {
...
205:        uint256 repayBorrowAmount;
206:        {
207:            uint256 maxRepayBorrowAmount = MathUtils.calcAssetAmount(
208:                maxRepayBorrowValue * MathUtils.ONE_10_DP,
209:                borrPriceFeed.price,
210:                borrPriceFeed.decimals
211:            );
212:            repayBorrowAmount = Math.min(maxAmountToRepay, Math.min(maxRepayBorrowAmount, violatorLoanBorrow.balance)); // @audit: assumption: total balance to be repaid
213:        }
214:        {
215:            uint256 seizeUnderlyingCollateralAmount = repayBorrowAmount.convToSeizedCollateralAmount( // @audit: collateral equivalent + liq bonus
216:                collPriceFeed.price,
217:                collPriceFeed.decimals,
218:                borrPriceFeed.price,
219:                borrPriceFeed.decimals,
220:                borrowLoanPool.liquidationBonus
221:            );
```

Since this is a full liquidation, the `repayBorrowAmount` will be set to the debt balance on line 212 above. On line 215 the `repayBorrowAmount` is converted into a dollar value and then the dollar value is converted to a collateral token amount, plus a liquidation bonus:

[MathUtils::convToSeizedCollateralAmount](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/libraries/MathUtils.sol#L455-L468)

```solidity
455:    function convToSeizedCollateralAmount(
456:        uint256 borrowAmount,
457:        uint256 collPrice,
458:        uint8 collDecimals,
459:        uint256 borrPrice,
460:        uint8 borrDecimals,
461:        uint256 liquidationBonus
462:    ) internal pure returns (uint256) {
463:        return
464:            Math.mulDiv(
465:                convertAssetAmount(borrowAmount, borrPrice, borrDecimals, collPrice, collDecimals), // @audit: repay amount converted to equivalent collateral amount
466:                (MathUtils.ONE_4_DP + liquidationBonus), // @audit: collateral amount increased by liquidation bonus
467:                MathUtils.ONE_4_DP
468:            );
```

[MathUtils::convertAssetAmount](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/libraries/MathUtils.sol#L55-L62)

```solidity
55:    function convertAssetAmount(
56:        uint256 amountFrom,
57:        uint256 priceFrom,
58:        uint8 decimalsFrom,
59:        uint256 priceTo,
60:        uint8 decimalsTo
61:    ) internal pure returns (uint256) {
62:        return calcAssetDollarValue(amountFrom, priceFrom, decimalsFrom).mulDiv(10 ** decimalsTo, priceTo); 
```

Considering our example state, the calculated collateral to seize will be as follows:

```
State:
- asset_price = 1e18
- debt_balance = 95e6
- collateral_balance = 100e6
- liquidation_bonus = 800

collateral_to_seize = debt_balance * (1e4 + liquidation_bonus) / 1e4
collateral_to_seize = 95e6 * (1e4 + liquidation_bonus) / 1e4 => 102600000 ~ 102.6e6
```

As we can see above, the calculated collateral to seize is greater than the available collateral balance for the violator. Therefore, the condition on line 226 in `calcLiquidationAmounts` will be true and the `repayBorrowAmount` will be re-calculated on line 228 in `LiquidationLogic.sol`, but this time it will be calculated based on the available collateral balance (total collateral balance is being seized):

[LiquidationLogic::calcLiquidationAmounts](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LiquidationLogic.sol#L222-L237)

```solidity
222:            uint256 collDepositInterestIndex = collPool.getUpdatedDepositInterestIndex();
223:            uint256 violatorUnderlingCollateralBalance = violatorLoanCollateral.balance.toUnderlingAmount(
224:                collDepositInterestIndex
225:            );
226:            if (seizeUnderlyingCollateralAmount > violatorUnderlingCollateralBalance) {
227:                seizeUnderlyingCollateralAmount = violatorUnderlingCollateralBalance; // @audit: available collateral balance
228:                repayBorrowAmount = seizeUnderlyingCollateralAmount.convToRepayBorrowAmount( // @audit: repay equivalent of available collateral + liq bonus
229:                    collPriceFeed.price,
230:                    collPriceFeed.decimals,
231:                    borrPriceFeed.price,
232:                    borrPriceFeed.decimals,
233:                    borrowLoanPool.liquidationBonus
234:                );
235:            }
236:
237:            liquidationAmounts.repayBorrowAmount = repayBorrowAmount; // @audit: debt to repay
```

[MathUtils::convToRepayBorrowAmount](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/libraries/MathUtils.sol#L502-L515)

```solidity
502:    function convToRepayBorrowAmount(
503:        uint256 collAmount,
504:        uint256 collPrice,
505:        uint8 collDecimals,
506:        uint256 borrPrice,
507:        uint8 borrDecimals,
508:        uint256 liquidationBonus
509:    ) internal pure returns (uint256) {
510:        return
511:            Math.mulDiv(
512:                convertAssetAmount(collAmount, collPrice, collDecimals, borrPrice, borrDecimals),
513:                (MathUtils.ONE_4_DP + liquidationBonus),
514:                MathUtils.ONE_4_DP
515:            );
```

As we can see above, the `repayBorrowAmount` is calculated the same way the collateral to seize was calculated, but this time the available collateral balance is being converted to an equivalent repay amount and then the liquidation bonus is added:

```
State:
- asset_price = 1e18
- debt_balance = 95e6
- collateral_balance = 100e6
- liquidation_bonus = 800

repay_borrow_amount = collateral_balance * (1e4 + liquidation_bonus) / 1e4
repay_borrow_amount = 100e6 * (1e4 + 800) / 1e4 => 108e6
```

As we can see above, the re-calculated `repayBorrowAmount` is now greater than the actual debt balance. After this point, the `repayBorrowAmount` is used to decrement the violator's debt balance, which will result in an underflow:

[LiquidationLogic::updateLiquidationBorrows](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LiquidationLogic.sol#L30-L50)

```solidity
30:    function updateLiquidationBorrows(
31:        DataTypes.LiquidationLoansParams memory loansParams,
32:        DataTypes.BorrowPoolParams memory borrowPoolParams,
33:        uint256 repayBorrowAmount,
34:        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans
35:    ) external returns (DataTypes.LiquidationBorrowTransfer memory liquidationBorrowTransfer) {
...
49:        (uint256 repaidBorrowAmount, uint256 repaidBorrowBalance, uint256 violatorStableRate) = UserLoanLogic
50:            .transferBorrowFromViolator(violatorLoan, loansParams.borrowPoolId, repayBorrowAmount); // @audit: violator debt decreased
```

[UserLoanLogic::transferBorrowFromViolator](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/UserLoanLogic.sol#L114-L128)

```solidity
114:    function transferBorrowFromViolator(
115:        LoanManagerState.UserLoan storage loan,
116:        uint8 poolId,
117:        uint256 repayBorrowAmount
118:    ) external returns (uint256 repaidBorrowAmount, uint256 repaidBorrowBalance, uint256 loanStableRate) {
119:        LoanManagerState.UserLoanBorrow storage loanBorrow = loan.borrows[poolId];
120:
121:        // violator loanBorrow has beed updated in prepareLiquidation
122:
123:        repaidBorrowBalance = repayBorrowAmount;
124:        repaidBorrowAmount = Math.min(repaidBorrowBalance, loanBorrow.amount);
125:        loanStableRate = loanBorrow.stableInterestRate;
126:
127:        loanBorrow.amount -= repaidBorrowAmount;
128:        loanBorrow.balance -= repaidBorrowBalance; // @audit: revert due to underflow -> repaidBorrBalance > loanBorrow.balance
```

The `repayBorrowAmount` should not be recalculated on line 228, since non-underwater positions (positions that will be liquidated for a profit) will have an overall collateral value greater than the debt value. Therefore, converting the collateral amount to an equivalent repay amount based on the collateral value will result in a repay value equivalent to the collateral value (collateral\_value == repay\_value). Additionally, the repay amount is then increased by the liquidation bonus. Therefore, as long as the debt position is not underwater (collateral\_value >= debt\_value), the recalculated repay amount will always result in a repay amount greater than the actual debt balance.

See `Recommended Mitigation` section below to observe an alternative way to readjust the `repayBorrowAmount` during these conditions.

## Impact

Full liquidations will be blocked for certain unhealthy positions.

*Necessary pre-conditions:*

* The debt position must have a high LTV (95% used in above example) so that the equivalent calculated collateral amount, when increased by the liquidation bonus, will exceed the available collateral balance. This means the collateral factor for the collateral asset would need to be 95% or the debt position remained unhealthy until the LTV reached 95%. Note that the LTV can be lower if the liquidation bonus is higher (800 used in above example) and vice versa.
* loan is not underwater, i.e. `collateral_value >= debt_value`

## Recommended Mitigation

On line 228 in `LiquidationLogic.sol`, the `repayBorrowAmount` should not be re-calculated the same way the collateral to be seized was calculated and instead should have simply been scaled down based on the ratio of available collateral and the calculated collateral to be seized. Going off our previous state, the calculation would look like so:

```
State:
- asset_price = 1e18
- debt_balance = 95e6
- collateral_balance = 100e6
- liquidation_bonus = 800

collateral_to_seize = 102600000 
repay_borrow_amount = debt_balance
adjusted_repay_amount = repay_borrow_amount * collateral_balance / collateral_to_seize
adjusted_repay_amount = 95e6 * 100e6 / 102600000 = 92592592 ~ 92.59e6
```

## Proof of concept

## Proof of Concept

For simplicity, I chose to slightly modify a liquidation unit test for folks finance test suite in order to showcase this issue. The diff below shows what lines I altered and the expected output shows that the test fails due to an underflow error, as described in my report.

To run POC:

* change the following files in the (`/test/hub/LoanManager.test.ts`)\[<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/test/hub/LoanManager.test.ts>] file
* run test with `npm run test ./test/hub/LoanManager.test.t`

```diff
diff --git a/./test/hub/LoanManager.test.ts b/./test/hub/LoanManagerNew.test.ts
index f7fbf2b..1171398 100644
--- a/./test/hub/LoanManager.test.ts
+++ b/./test/hub/LoanManagerNew.test.ts
@@ -194,7 +194,7 @@ describe("LoanManager (unit tests)", () => {

     const usdcCollateralFactor = BigInt(0.8e4);
     const usdcBorrowFactor = BigInt(1e4);
-    const usdcLiquidationBonus = BigInt(0.04e4);
+    const usdcLiquidationBonus = BigInt(0.05e4); // 5% liq bonus
     const usdcLiquidationFee = BigInt(0.1e4);

     const ethCollateralFactor = BigInt(0.7e4);
@@ -3695,7 +3695,7 @@ describe("LoanManager (unit tests)", () => {
       // Liquidator:
       // Collateral 10,000 USDC = $10,000
       // Borrow $0
-      const repayAmount = BigInt(100e6); // 100 USDC
+      const repayAmount = borrowBalance; // repay total debt balance
       const collateralFAmount = convToCollateralFAmount(
         repayAmount,
         ethNodeOutputData.price,
```

Expected output:

```js
  3) LoanManager (unit tests)
       Liquidate
         Should successfully liquidate variable borrow when seizing new borrow and collateral:
     Error: VM Exception while processing transaction: reverted with panic code 0x11 (Arithmetic operation overflowed outside of an unchecked block)
```
