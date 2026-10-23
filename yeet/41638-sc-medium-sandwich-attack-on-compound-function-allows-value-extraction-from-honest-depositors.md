# #41638 \[SC-Medium] Sandwich Attack on \`compound()\` Function Allows Value Extraction from Honest Depositors

**Submitted on Mar 17th 2025 at 07:53:19 UTC by @DoD4uFN for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41638
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The `compound()` function in `MoneyBrinter.sol` allows the strategy manager to swap reward tokens and deposit them into `BeradromeFarmPlugin`, increasing the share price. However, this function is vulnerable to a sandwich attack, where an attacker can front-run the compound transaction by minting shares before it executes and then immediately selling after the share price increases, allowing them to extract a portion of the compounded rewards at the expense of honest depositors.

## Vulnerability Details

The vulnerability arises due to the predictable and public nature of the `compound()` function’s execution. The attacker can monitor the mempool for `compound()` transactions and execute the following steps:

1. **Front-run the compound transaction:** The attacker mints new shares by depositing right before the `compound()` function executes.
2. **Let the compound transaction execute:** The function swaps reward tokens and deposits them into the farm, increasing the value of each share.
3. **Sell the inflated shares:** The attacker then sells their shares immediately after, capturing a portion of the newly compounded rewards.

This results in an unfair redistribution of yield, where honest depositors receive less benefit from the compounding process while the attacker profits.

### Relevant Code Snippet

```solidity
function compound(
    address[] calldata swapInputTokens,
    IZapper.SingleTokenSwap[] calldata swapToToken0,
    IZapper.SingleTokenSwap[] calldata swapToToken1,
    IZapper.KodiakVaultStakingParams calldata stakingParams,
    IZapper.VaultDepositParams calldata vaultStakingParams
) public override onlyStrategyManager nonReentrant returns (uint256) {
    uint256 initialSupply = totalSupply();
    uint256 shareValueBefore = previewRedeem(initialSupply);
    
    _approveTokens(swapInputTokens, swapToToken0, swapToToken1);
    
    (uint256 islandTokensMinted, uint256 vaultSharesMinted) =
        zapper.zapInWithMultipleTokens(swapParams, stakingParams, vaultStakingParams);
    
    _depositIntoFarm(islandTokensMinted);
    
    uint256 shareValueAfter = previewRedeem(initialSupply);
    require(shareValueAfter >= shareValueBefore, "MoneyBrinter: Bad Compound");
    
    return islandTokensMinted;
}
```

## Impact Details

This exploit enables an attacker to extract value from the yield compounding process, leading to loss of rewards for honest depositors.

## References

[MoneyBrinter:compound](https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol#L199-L227)

## Proof of Concept

## Proof of Concept

1. The attacker monitors the mempool for an impending `compound()` transaction.
2. They submit a transaction to deposit assets and mint shares, ensuring it gets confirmed right before `compound()` executes.
3. Once `compound()` executes, increasing the share price, the attacker sells their shares immediately after, capturing the newly compounded rewards.
