# #42602 \[SC-Medium] Some of the Compounded Reward Island token can be stolen by sandwiching the compound() function call

**Submitted on Mar 24th 2025 at 22:56:12 UTC by @kaysoft for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42602
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

MEV bots can be setup to monitor the mempool for the `compound()` transaction and sandwich it for profit.

The attacker first buys the MoneyBrinter.sol shares for cheaper just before the `compound()` execution then dump the shares immediately after the `compound()` execution because `compound()`execution increases the totalAsset of the MoneyBrinter.sol at that instant of execution. The attacker then gets more Island tokens than deposited since the asset received is determined by the fraction of the shares held by the attacker.

The more Island token the attacker have to execute this attack, the more of the reward tokens that will be stolen by the attacker.

## Vulnerability Details

The `compound()` function of MoneyBrinter.sol converts rewards to the underlying token(island token) before depositing it to the Beradome farm. This compounding increases the totalAsset of MoneyBrinter.sol instantly.

This creates an opportunity for malicious user to deposit to MoneyBrinter.sol just before the `compound()` is called by the StrategyManager.

When a malicious user calls deposit just before the compounding, the user buys the shares cheaper because the shares is calculated with the totalAsset as the denominator.

When the `compound()` is executed, the malicious user immediately withdraws by burning the earlier bought cheap shares. Now the totalAsset has already bumped up with extra reward earned over time by previous deposits.

The malicious user makes profit with sandwich attack in the same block.

```solidity
File: MoneyBrinter.sol
function compound(
        address[] calldata swapInputTokens,
        IZapper.SingleTokenSwap[] calldata swapToToken0,
        IZapper.SingleTokenSwap[] calldata swapToToken1,
        IZapper.KodiakVaultStakingParams calldata stakingParams,
        IZapper.VaultDepositParams calldata vaultStakingParams
    ) public override onlyStrategyManager nonReentrant returns (uint256) {
        // By adding the staking params receiver as the vault address, we ensure that zapper returns the island Tokens to the vault
        require(stakingParams.receiver == address(this), "Invalid staking receiver");
        require(swapInputTokens.length == swapToToken0.length + swapToToken1.length, "Invalid swap data");
        uint256 initialSupply = totalSupply();
        uint256 shareValueBefore = previewRedeem(initialSupply);
        _approveTokens(swapInputTokens, swapToToken0, swapToToken1);
        IZapper.MultiSwapParams memory swapParams = IZapper.MultiSwapParams({
            inputTokens: swapInputTokens,
            swapToToken0: swapToToken0,
            swapToToken1: swapToToken1
        });
        (uint256 islandTokensMinted, uint256 vaultSharesMinted) =
            zapper.zapInWithMultipleTokens(swapParams, stakingParams, vaultStakingParams);
        require(vaultSharesMinted == 0, "MoneyBrinter: vault shares minted while compounding");
        require(islandTokensMinted >= stakingParams.amountSharesMin, "MoneyBrinter: not enough island tokens minted");
        // deposit into farm
        emit VaultCompounded(_msgSender(), islandTokensMinted);
        _depositIntoFarm(islandTokensMinted);//@audit this instantly increases totalAsset creating MEV opportunity
        uint256 shareValueAfter = previewRedeem(initialSupply);
        require(shareValueAfter >= shareValueBefore, "MoneyBrinter: Bad Compound");
        return islandTokensMinted;
    }
```

Remember the totalAsset is equivalent to the shares minted from the BeraDome Farm which is increased by the `_depositIntoFarm(...)`. This `_depositIntoFarm(...)` is called both in the deposit() and compound() functions of moneyBrinter.sol

## Impact Details

* This creates a situation where MEV bots are setup to snipe profit from the `compound(...)` function call through sandwich attack(deposit and withdraw in the same block.
* Users whose deposits actually generate those rewards lose most of the rewards to MEV bots.

## Recommendation

Consider ensuring user deposits accumulate rewards over time in MoneyBrinter.sol so that a new user does not profit from already made rewards.

## Proof of Concept

## Proof of Concept

1. Alice deposits 100 Island token(underlying token) to MoneyBrinter.sol and 100 shares token is minted to Alice.
2. After 2 days, this deposited 100 Island token generates 50 island token worth of reward tokens.
3. The StrategyManager submits a transaction to execute the `compound()` function to the mempool. This transaction is supposed to convert the reward earned from Alice's deposit to 50 island token which will be deposited in to Beradome Farm.
4. Bob is already monitoring the mempool and frontruns the above profitable compound(...) transaction with a 1000 island token deposit and gets 1000 shares.
5. The `compound()` function is now executed after Bob's transaction adding this 50 island token reward.

Now totalAsset is 1000 + 100 + 50 island token from compounding = 1150 while\
totalShares still remain 1000 + 100 = 1100.

6. Bob immediately backruns the above `compound(...)` execution with a withdrawal of the 1000 shares he got earlier.
7. For 1000 shares Bob gets: 1000 \* 1150 / 1100 = 1045.
8. Bob makes a profit of 45 Island token by sandwiching the `compound()` transaction in the same block. Remember this 45 island token is part of the 50 island token generated by Alice's deposit over 2days.
9. Alice only gets 100 \* 1150 / 1100 = 5\
   Alice gets only 5 island tokens instead of the whole 50 that her deposit generated over 2days.

This is an MEV opportunity for bots to profit from.
