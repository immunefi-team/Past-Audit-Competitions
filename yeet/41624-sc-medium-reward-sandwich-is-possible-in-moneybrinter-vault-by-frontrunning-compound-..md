# #41624 \[SC-Medium] Reward sandwich is possible in \`MoneyBrinter\` vault by frontrunning \`compound\`.

**Submitted on Mar 17th 2025 at 05:25:51 UTC by @OxAnmol for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41624
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

A malicious user can front-run the `compound` function with large deposits and withdraw immediately in the following block, securing risk-free profits and stealing rewards from other legitimate users.

## Vulnerability Details

`MoneyBrinter` allows users to deposit Kodiak's island token from the YEET-WBERA pool. The MoneyBrinter then deposits this token into Beradrome for rewards. Any user can call the `harvestKodiakRewards` and `harvestBeradromeRewards` functions to collect corresponding rewards from beradrome to vault.

The `strategyManager` can periodically call the `compound` function to compound rewards (swap reward tokens, provide liquidity to Kodiak, and deposit island tokens to Beradrome). This process increases the value of the ERC4626 shares for users.

```solidity
/**
     * @notice Compounds rewards by swapping harvested tokens, staking in Kodiak vault and depositing into Beradrome farm
     * @param swapInputTokens Array of input token addresses for swaps
     * @param swapToToken0 Array of swap params to swap input tokens to token0
     * @param swapToToken1 Array of swap params to swap input tokens to token1
     * @param stakingParams Parameters for staking in Kodiak vault
     * @param vaultStakingParams Parameters for depositing into vault
     * @return uint256 Amount of island tokens minted
     * @dev This function is non-reentrant and can only be called by the strategy manager
     * @dev It approves tokens, performs swaps, stakes in Kodiak vault, and deposits into farm
     * @dev Emits a VaultCompounded event
     */
    function compound(
        address[] calldata swapInputTokens,
        IZapper.SingleTokenSwap[] calldata swapToToken0,
        IZapper.SingleTokenSwap[] calldata swapToToken1,
        IZapper.KodiakVaultStakingParams calldata stakingParams,
        IZapper.VaultDepositParams calldata vaultStakingParams
    ) public override onlyStrategyManager nonReentrant returns (uint256) {
        // By adding the staking params receiver as the vault address, we ensure that zapper returns the island Tokens to the vault
        require(
            stakingParams.receiver == address(this),
            "Invalid staking receiver"
        );
        require(
            swapInputTokens.length == swapToToken0.length + swapToToken1.length,
            "Invalid swap data"
        );
        uint256 initialSupply = totalSupply();
        uint256 shareValueBefore = previewRedeem(initialSupply);
        _approveTokens(swapInputTokens, swapToToken0, swapToToken1);
        IZapper.MultiSwapParams memory swapParams = IZapper.MultiSwapParams({
            inputTokens: swapInputTokens,
            swapToToken0: swapToToken0,
            swapToToken1: swapToToken1
        });
        (uint256 islandTokensMinted, uint256 vaultSharesMinted) = zapper
            .zapInWithMultipleTokens(
                swapParams,
                stakingParams,
                vaultStakingParams
            );
        require(
            vaultSharesMinted == 0,
            "MoneyBrinter: vault shares minted while compounding"
        );
        require(
            islandTokensMinted >= stakingParams.amountSharesMin,
            "MoneyBrinter: not enough island tokens minted"
        );
        // deposit into farm
        emit VaultCompounded(_msgSender(), islandTokensMinted);
        _depositIntoFarm(islandTokensMinted);
        uint256 shareValueAfter = previewRedeem(initialSupply);
        require(
            shareValueAfter >= shareValueBefore,
            "MoneyBrinter: Bad Compound"
        );
        return islandTokensMinted;
    }

```

A malicious actor can monitor the `compound` function and front-run it with a large deposit into the vault. This makes them eligible for the compounded rewards. In the next block, they can simply withdraw their island tokens and receive more tokens without any risk.

Because the user deposits a large amount, they will capture most of the rewards from legitimate users who have been depositing in the vault for much longer periods.

## Impact Details

Honest users who have been staking in the vault for extended periods will receive significantly reduced rewards, while malicious users can obtain risk-free profits without contributing meaningfully to the vault. This undermines the incentive structure and fairness of the protocol.

i believe based on impact in scope this is `theft of unclaimed yield` so it is a high severity.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>

## Proof of Concept

## Flow

* Alice and Bob are honest users who each stake 50 island tokens and receive 50 share tokens each.
* Over time, the vault earns rewards in the form of xKDK and oBERO, worth approximately $1,000.
* Someone calls the harvest function to collect these rewards.
* The `strategyManager` prepares to call `compound` to swap and redeposit island tokens to Beradrome for more rewards.
* Jack, a malicious user, observes this transaction and front-runs it by calling `deposit` with 1,000 island tokens, receiving 1,000 shares in return.
* When `compound` is executed, the total assets increase by 100 tokens, bringing the total to 50 (Alice) + 50 (Bob) + 1,000 (Jack) + 100 (newly compounded) = 1,200 tokens.
* In the next block, when Jack redeems his shares, he receives: 1,000 \* 1,200 / 1,100 = 1,090 island tokens.
* Even with a 4% withdrawal fee (as mentioned by the sponsor), Jack would pay 43.6 island tokens as a fee (4% of 1,090).
* Jack's risk-free profit is therefore 1,090 - 1,000 - 43.6 = 46.4 island tokens.
