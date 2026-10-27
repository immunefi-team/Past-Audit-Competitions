# #42333 \[SC-Medium] compound MoneyBrinter.sol can be sandwiched to extract value from other depositors

**Submitted on Mar 23rd 2025 at 00:29:04 UTC by @cryptostaker for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42333
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The assets inside MoneyBrinter needs to be manually increased through calling compound, which opens the contract to a sandwich attack.

## Vulnerability Details

The asset in MoneyBrinter is the token received from Beradrome Finance when staking KDK LP tokens.

Anyone can call `harvest()` to harvest rewards from the asset, which is sent to the vault contract.

The strategy manager then has to call `compound()` to convert all the reward tokens back to KDK LP tokens, and deposit the LP tokens into Beradrome vault which gets more assets.

Before calling `compound()`, a user can deposit some KDK LP tokens into the contract, wait for the compound, and withdraw the shares to gain extra KDK tokens. Although there is a 4% withdrawal fee (for now its set at 0%), as long as the compounding amount is greater than 4%, malicious actors can take advantage of this sandwich attack to extract value at the expense of other stakers.

## Impact Details

Not easy to mitigate this issue, one way is to have a small timelock when depositing inside the vault to prevent any sort of sandwich attack.

## References

Harvest can be called by anyone, but compound only can be called by strategyManager.

```
    function harvestBeradromeRewards() public override nonReentrant {
        beradromeFarmRewardsGauge.getReward(address(this)); // claims Beradrome rewards
        // allocate xKDK to token rewards module.
        if (allocateXKDKToKodiakRewards) {
            uint256 xKdkBalance = IXKdkToken(xKdk).balanceOf(address(this));
            IXKdkToken(xKdk).approveUsage(kodiakRewards, xKdkBalance); // approve xKDK to be used by Token Rewards Module
            IXKdkTokenUsage(xKdk).allocate(
                kodiakRewards, xKdkBalance, "" /* calldata unused in Token Rewards Module */
            );
        }
        emit BeradromeRewardsHarvested(_msgSender());
    }

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
```

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/MoneyBrinter.sol#L199-L209>

## Proof of Concept

## Proof of Concept

Let's say the current asset to share ratio is 1:1, and there is 1000 asset inside the vault with 1000 shares.

These 1000 asset has about 100 assets worth as rewards, and the strategyManager is ready to compound.

Right before compounding, a malicious actor deposit another 1000 asset inside to vault, bringing the vault up to 2000 shares.

`Compound()` is called, which raises the asset to 2100, while shares still remains at 2000.

The malicious actor then withdraws the 1000 shares he has and gets 1050 assets, netting him 50 asset tokens, in this case, KDK LP tokens.

If there are no withdrawal fees, the malicious actor gets away with 50 KDK LP tokens.
