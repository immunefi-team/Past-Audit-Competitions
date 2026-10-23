# 58133 sc low toke rewards permanently locked in strategy adapter

**Submitted on Oct 30th 2025 at 21:21:06 UTC by @nem0thefinder for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58133
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Rewarder Toke rewards are locked in strategy adapter

## Description

> Note! The same happen in `TokeAutoETH`

## Summary

TOKE reward tokens claimed from the Tokemak rewarder are permanently locked in the `TokeAutoUSDStrategy` contract with no mechanism to retrieve them, resulting in continuous value loss to the protocol and users.

## Description

The `TokeAutoUSDStrategy` contract stakes `autoUSD` shares received from allocating to `autoUsdVault` in a Tokemak `rewarder` that distributes `TOKE` tokens as rewards. During deallocation, the strategy claims these TOKE rewards:

```solidity
TokeAutoUSDStrategy.sol
function _deallocate(uint256 amount) internal override returns (uint256) {
    // ...
 @>   rewarder.withdraw(address(this), sharesNeeded, true);  // claim=true
    // TOKE tokens are transferred to the strategy here
    
  @>  autoUSD.redeem(sharesNeeded, address(this), address(this));
    // Only autoUSD → USDC conversion happens

    // approve only usdc to vault to pull out from the adpater
    TokenUtils.safeApprove(address(usdc), msg.sender, amount);
    return amount;
    // TOKE tokens remain in the strategy contract
}
```

The issue here that `deallocate` function don't handle `Toke` tokens received from the rewarder leaving them locked in the contract. making the whole rewarder staking process useless.

## Impact

* **Financial Loss:** TOKE rewards accumulate indefinitely in the strategy contract
* **No Recovery Mechanism:** Tokens are permanently inaccessible to protocol, users, and governance

## Mitigation

1. **Implement reward Handling:** override `MYT::claimRewards` and implement your custom logic for handling toke rewards OR
2. **Token Rescue Function:** Add rescue function that allow trusted roles to withdraw those tokens.

## Proof of Concept

## Proof of Concept

> Note! These instructions will be applied in `TokeAutoUSDCStrategy.t.sol`

### 1.Add the following interface

```solidity
interface IBaseRewarder {
    /**
     * @notice The amount of tokens staked for the specified account
     * @param account The address of the account to get the balance of
     */
    function balanceOf(address account) external view returns (uint256);

    /**
 * @notice The total amount of tokens staked
     */
    function totalSupply() external view returns (uint256);

    /**
     * @notice Calculates the rewards per token for the current block.
     * @dev The total amount of rewards available in the system is fixed, and it needs to be distributed among the users
     * based on their token balances and staking duration.
     * Rewards per token represent the amount of rewards that each token is entitled to receive at the current block.
     * The calculation takes into account the reward rate, the time duration since the last update,
     * and the total supply of tokens in the staking pool.
     * @return The updated rewards per token value for the current block.
     */
    function rewardPerToken() external view returns (uint256);

    /**
     * @notice Get the current reward rate per block.
     * @return The current reward rate per block.
     */
    function rewardRate() external view returns (uint256);

    /**
 * @notice Calculate the earned rewards for an account.
     * @param account Address of the account.
     * @return The earned rewards for the given account.
     */
    function earned(address account) external view returns (uint256);

    /**
 * @notice Claims and transfers all rewards for the specified account
     */
    function getReward() external;

    /**
 * @notice Token distributed as rewards
     * @return reward token address
     */
    function rewardToken() external view returns (address);

    /**
 * @notice Get the last block where rewards are applicable.
     * @return The last block number where rewards are applicable.
     */
    function lastBlockRewardApplicable() external view returns (uint256);

}
```

```solidity
interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address a) external view returns (uint256);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
}

```

### 2. Fork the current block

```solidity
    function getForkBlockNumber() internal pure override returns (uint256) {
       // return 22_089_302;
        return 0;
    }

```

### 3.Paste the following test

```solidity
    function test_toke_rewards_permanently_locked() public {
        // ========== STEP 1: ALLOCATE $100K TO STRATEGY ==========
        uint256 amountToAllocate = 100_000e6; // $100,000 USDC
        deal(testConfig.vaultAsset, strategy, amountToAllocate);

        vm.startPrank(vault);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));

        uint256 strategyRealAssets = IMYTStrategy(strategy).realAssets();
        uint256 strategyRewarderSharesBalance = IERC20(REWARDER).balanceOf(strategy);
        uint256 strategyAutoUSDShares = IERC20(TOKE_AUTO_USD_VAULT).balanceOf(strategy);

        console.log("=== ALLOCATION COMPLETE ===");
        console.log("Strategy real assets: %e USDC", strategyRealAssets);
        console.log("Strategy rewarder shares staked: %e", strategyRewarderSharesBalance);
        console.log("Strategy autoUSD shares (should be 0, staked in rewarder): %e", strategyAutoUSDShares);
        console.log("");

        // ========== STEP 2: SIMULATE 10 DAYS OF STAKING ==========
        console.log("=== FAST FORWARD 1 DAY (72,000 blocks) ===");
        uint256 blocksIn1Day = 72000;
        vm.roll(block.number + blocksIn1Day);

        address rewardToken = IBaseRewarder(REWARDER).rewardToken();
        uint256 rewarderEarned = IBaseRewarder(REWARDER).earned(strategy);
        uint256 rewardRate = IBaseRewarder(REWARDER).rewardRate();
        uint256 preClaimRewardTokenBalance = IERC20(rewardToken).balanceOf(strategy);

        console.log("TOKE reward token address: %s", rewardToken);
        console.log("TOKE earned by strategy: %e (~27 TOKE)", rewarderEarned);
        console.log("Reward rate per block: %e TOKE", rewardRate);
        console.log("Strategy TOKE balance BEFORE deallocation: %e (should be 0)", preClaimRewardTokenBalance);
        console.log("");

        // ========== STEP 3: CALCULATE PROJECTED ANNUAL REWARDS ==========
        console.log("=== PROJECTED ANNUAL REWARDS ===");

        uint256 AvgexpectedYearlyRewards =rewarderEarned * 365 ; // i know this is not exact but close enough for estimation

        console.log("Expected TOKE rewards for 1 year: %e (~10,000 TOKE)", AvgexpectedYearlyRewards);
        console.log("");

        // ========== STEP 4: DEALLOCATE (TRIGGERS REWARD CLAIM) ==========
        console.log("=== DEALLOCATING $100K ===");
        uint256 amountToDeallocate = IMYTStrategy(strategy).previewAdjustedWithdraw(amountToAllocate);
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);

        // Adjust for slippage (bypass the require statement)
        deal(testConfig.vaultAsset, strategy, amountToAllocate - amountToDeallocate);

        uint256 vaultUSDCBalanceBefore = IERC20(USDC).balanceOf(vault);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        // Mocking vault automatic transfer of USDC from strategy to vault
        IERC20(USDC).transferFrom(strategy, vault,amountToDeallocate);
        uint256 vaultUSDCBalanceAfter = IERC20(USDC).balanceOf(vault);

        uint256 strategyUSDCBalance = IERC20(USDC).balanceOf(strategy);
        uint256 strategyTokeBalance = IERC20(rewardToken).balanceOf(strategy);

        console.log("USDC returned to vault: %e", vaultUSDCBalanceAfter - vaultUSDCBalanceBefore);
        console.log("Strategy USDC balance (leftover): %e", strategyUSDCBalance); // amount used to bypass the require
        console.log("Strategy TOKE balance AFTER deallocation: %e (~27 TOKE)", strategyTokeBalance);
        console.log("");

        // ========== STEP 5: VERIFY REWARDS ARE LOCKED ==========
        console.log("=== VERIFICATION ===");
        assertGt(strategyTokeBalance, 0, "TOKE rewards should have been claimed");
        assertEq(strategyTokeBalance, rewarderEarned, "All earned TOKE should be in strategy");

        console.log("[CRITICAL] TOKE rewards are STUCK in strategy contract!");
        console.log("[CRITICAL] No rescue function exists to recover these tokens!");
        console.log("[CRITICAL] claimRewards() implementation is empty and does nothing!");
        console.log("");

        // Try to call claimRewards - it does nothing
        uint256 tokeBalanceBefore = IERC20(rewardToken).balanceOf(strategy);
        IMYTStrategy(strategy).claimRewards();
        uint256 tokeBalanceAfter = IERC20(rewardToken).balanceOf(strategy);

        assertEq(tokeBalanceBefore, tokeBalanceAfter, "claimRewards() does nothing - tokens still stuck");
        console.log("claimRewards() called: TOKE balance unchanged (%e)", tokeBalanceAfter);

        vm.stopPrank();

        // ========== SUMMARY ==========
        console.log("");
        console.log("=== IMPACT SUMMARY ===");
        console.log("Locked in 1 day: ~27 TOKE");
        console.log("Projected annual loss per $100k: ~10,000 TOKE ");
    }
```

### 4. Run it via `forge test --mc TokeAutoUSDStrategyTest --mt test_toke_rewards_permanently_locked -vvv`

### Logs

```
Logs:
  === ALLOCATION COMPLETE ===
  Strategy real assets: 9.9994472186e10 USDC
  Strategy rewarder shares staked: 9.5579303179302758119086e22
  Strategy autoUSD shares (should be 0, staked in rewarder): 0e0
  
  === FAST FORWARD 1 DAY (72,000 blocks) ===
  TOKE reward token address: 0x2e9d63788249371f1DFC918a52f8d799F4a38C94
  TOKE earned by strategy: 2.6917086659885339924e19 (~27 TOKE)
  Reward rate per block: 1.49429180530373971e17 TOKE
  Strategy TOKE balance BEFORE deallocation: 0e0 (should be 0)
  
  === PROJECTED ANNUAL REWARDS ===
  Expected TOKE rewards for 1 year: 9.82473663085814907226e21 (~10,000 TOKE)
  
  === DEALLOCATING $100K ===
  USDC returned to vault: 9.999e10
  Strategy USDC balance (leftover): 4.472434e6
  Strategy TOKE balance AFTER deallocation: 2.6917086659885339924e19 (~27 TOKE)
  
  === VERIFICATION ===
  [CRITICAL] TOKE rewards are STUCK in strategy contract!
  [CRITICAL] No rescue function exists to recover these tokens!
  [CRITICAL] claimRewards() implementation is empty and does nothing!
  
  claimRewards() called: TOKE balance unchanged (2.6917086659885339924e19)
  
  === IMPACT SUMMARY ===
  Locked in 1 day: ~27 TOKE
  Projected annual loss per $100k: ~10,000 TOKE 

```
