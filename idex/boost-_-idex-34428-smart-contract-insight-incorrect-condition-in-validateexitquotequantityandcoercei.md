# Boost \_ IDEX 34428 - \[Smart Contract - Insight] Incorrect Condition in validateExitQuoteQuantityAndC

Submitted on Mon Aug 12 2024 10:15:39 GMT-0400 (Atlantic Standard Time) by @holydevoti0n for [Boost | IDEX](https://immunefi.com/bounty/boost-idex/)

Report ID: #34428

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/idexio/idex-contracts-ikon/blob/main/contracts/libraries/Withdrawing.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The \`\`validateExitQuoteQuantityAndCoerceIfNeeded\` function is intended to handle small negative values resulting from rounding errors by coercing them to zero, allowing wallet positions to be closed out correctly. However, there is a discrepancy between the implemented logic and the comments provided in the code.

<https://github.com/idexio/idex-contracts-ikon/blob/a4bfee2cb80daec8ba22ee926a13884807d0a94a/contracts/libraries/Withdrawing.sol#L290>

```solidity
Math.abs(walletQuoteQuantityToWithdraw) <= Constants.MINIMUM_QUOTE_QUANTITY_VALIDATION_THRESHOLD
```

According to the comments here: <https://github.com/idexio/idex-contracts-ikon/blob/a4bfee2cb80daec8ba22ee926a13884807d0a94a/contracts/libraries/Constants.sol#L67-L68>

"Positions SMALLER than this threshold will skip quote quantity validation for Position Below Minimum liquidations and skip non-negative total quote validation Wallet Exits"

## Vulnerability Details

According to the comments, positions smaller than the `Constants.MINIMUM_QUOTE_QUANTITY_VALIDATION_THRESHOLD` should skip quote quantity validation. However, the current implementation uses a `<=` comparison, which may incorrectly coerce values that are exactly equal to the threshold, rather than values strictly less than the threshold.

## Impact Details

This discrepancy could lead to unexpected behavior where positions with a quote quantity exactly equal to the `MINIMUM_QUOTE_QUANTITY_VALIDATION_THRESHOLD` are coerced to zero, potentially skipping necessary validations. This could result in incorrect handling of wallet exits or position liquidations, particularly in edge cases.

## Recommendations

To align the logic with the intended behavior as described in the comments, the condition should be modified from `<=` to `<`:

```diff
- Math.abs(walletQuoteQuantityToWithdraw) <= Constants.MINIMUM_QUOTE_QUANTITY_VALIDATION_THRESHOLD
+ Math.abs(walletQuoteQuantityToWithdraw) < Constants.MINIMUM_QUOTE_QUANTITY_VALIDATION_THRESHOLD

```

## Proof of concept

## Proof of Concept

On the file `exits.ts` add the following test inside the `validateExitQuoteQuantityAndCoerceIfNeeded` closure:

```
expect(
          (
            await withdrawExitValidationsMock.validateExitQuoteQuantityAndCoerceIfNeeded(
              false,
              -10000, // @audit value is == `MINIMUM_QUOTE_QUANTITY_VALIDATION_THRESHOLD`
            )
          ).toString(),
        ).to.equal('0');
```

And run: `npx hardhat test test/exits.ts`

Output: test succeeds but it should revert as -10000 is not smaller than `MINIMUM_QUOTE_QUANTITY_VALIDATION_THRESHOLD`
