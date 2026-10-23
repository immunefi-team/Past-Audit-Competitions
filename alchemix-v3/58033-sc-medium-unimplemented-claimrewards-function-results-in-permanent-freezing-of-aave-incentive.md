# 58033 sc medium unimplemented claimrewards function results in permanent freezing of aave incentive rewards

**Submitted on Oct 30th 2025 at 06:28:43 UTC by @theboiledcorn for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58033
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The `AaveV3ARBUSDCStrategy` contract fails to implement the required `_claimRewards()` function inherited from its `MYTStrategy` base contract. When the strategy deploys USDC to Aave V3, it accrues Aave incentive rewards (e.g., ARB tokens). Because the contract lacks the code to claim these rewards, and Aave's protocol only allows the aToken holder (the strategy itself) to claim, all earned yield becomes **permanently frozen and unrecoverable**, leading to a direct loss of funds.

## Vulnerability Details

The core of the vulnerability lies in an incomplete implementation of the `MYTStrategy` interface.

1. **Base Contract's Expectation:** The `MYTStrategy` base contract (which `AaveV3ARBUSDCStrategy` inherits from) provides a public `claimRewards()` function. This function's sole purpose is to call an internal virtual function, `_claimRewards()`, which is empty by default and **intended to be overridden by child strategies**.

   ```solidity
   // In MYTStrategy.sol (Base Contract)
   function claimRewards() public virtual returns (uint256) {
       require(!killSwitch, "emergency");
       _claimRewards(); // Calls the internal function
   }

   /// @dev override this function to claim all available rewards...
   function _claimRewards() internal virtual returns (uint256) {} // Empty by default
   ```
2. **Missing Implementation:** The `AaveV3ARBUSDCStrategy` contract correctly overrides other necessary functions like `_allocate` and `_deallocate`, but it **completely omits an override for `_claimRewards()`**. Therefore, when the public `claimRewards()` function is called on this strategy, it executes the empty base implementation, doing nothing.
3. **Aave's Reward Mechanism:** When the strategy allocates funds, it supplies USDC to the Aave pool and receives `aUSDC` tokens. The `onBehalfOf` parameter is set to `address(this)`, making the strategy contract the direct owner of the aTokens.

   ```solidity
   // In AaveV3ARBUSDCStrategy.sol
   function _allocate(uint256 amount) internal override returns (uint256) {
       // ...
       // Strategy (address(this)) becomes the owner of the aTokens
       pool.supply(address(usdc), amount, address(this), 0);
       return amount;
   }
   ```

   As the `aUSDC` holder, the strategy contract begins to accrue Aave incentive rewards. According to Aave's protocol design, **only the aToken holder (the strategy) can call the `IncentivesController` contract to claim its own rewards**.
4. **Permanent Freeze:** Since the strategy contract is the only entity that *can* claim the rewards, but it lacks the *code* to do so, there is no possible way to retrieve the accrued incentives. They will remain permanently locked in Aave's `IncentivesController` contract.

## Impact Details

This vulnerability leads to the **Permanent Freezing of Unclaimed Yield**.

* **Severity:** **High**. This is a direct, irreversible loss of funds. The primary yield (USDC interest) is accessible, but the secondary yield (Aave incentive rewards) is lost forever.
* **Classification:** This matches the severity classification: "Permanent freezing of unclaimed yield: A yield is any asset distributed as a reward for participation in a system. Whenever... the yield \[cannot] be able to move from the contract... this would mean the yield is permanently frozen." In this case, the yield cannot be moved from Aave's `IncentivesController`.
* **Financial Loss:** The total financial loss is 100% of all Aave incentive rewards ever earned by this strategy. The magnitude of this loss depends on three factors:

  1. The total amount of USDC deposited into the strategy.
  2. The incentive APR% offered by Aave for the USDC pool on Arbitrum.
  3. The duration of time the strategy is deployed.

  **Example Scenario:** If **$1,000,000 USDC** is deployed in this strategy for **one year** at an average incentive APR of **2%**, approximately **$20,000** worth of ARB (or other reward tokens) would be permanently and irretrievably lost.

## Link to Proof of Concept

<https://gist.github.com/theboiledcorn/ab8d6dd1fee8f6ebba4436e5a6ce19ef>

## Proof of Concept

## 3. Proof of Concept (PoC)

This Proof of Concept demonstrates that the reward-claiming mechanism is completely missing and that no other party can claim on the strategy's behalf. This proves that any future rewards emitted by Aave to `aUSDC` holders **will be permanently frozen**.

**Test Setup:**

* A test is run on an Arbitrum mainnet fork.
* The `AaveV3ARBUSDCStrategy` contract is deployed, funded with USDC, and its `allocate()` function is called.
* The test uses the function `test_aave_rewards_cannot_be_claimed` from the `AaveV3ARBUSDCStrategyTest` contract.

**Step-by-Step Explanation:**

1. **Allocate Funds & Verify aToken Ownership:**
   * **Action:** The test calls `allocate(1000e6)` on the strategy contract.
   * **Result:** The strategy successfully supplies `1,000` USDC to the Aave V3 Pool. The test confirms that the `AaveV3ARBUSDCStrategy` contract itself is now the owner of the `aUSDC` tokens.
   * **Code:** `assertGt(aTokenBalance, 0, "Strategy should hold aTokens");`
2. **Confirm Missing `_claimRewards` Implementation:**
   * **Action:** The public `claimRewards()` function is called on the `AaveV3ARBUSDCStrategy` contract.
   * **Result:** The function executes without error but returns `0`. This demonstrates that the contract is using the empty base implementation of `_claimRewards()`, which takes no action.
   * **Code:** `uint256 returned = MockAaveV3ARBUSDCStrategy(strategy).claimRewards();`
   * **Code:** `assertEq(returned, 0, "Empty implementation returns 0");`
3. **Confirm No External Party Can Claim:**
   * **Action:** The test simulates a call from the `strategyOwner`. The owner *directly* calls the `claimRewards` function on Aave's `IncentivesController` (`0x9...473e`), attempting to claim any potential rewards (indicated by `address(0)` as the reward token) to their own address.
   * **Result:** The call succeeds but returns `0` rewards. This proves that, per Aave's protocol rules, only the aToken holder (the strategy contract) can claim its own rewards. No external party, not even the owner, can intervene.
   * **Code:** `uint256 ownerClaimed = IAaveIncentivesController(AAVE_INCENTIVES_CONTROLLER).claimRewards(assets, type(uint256).max, strategyOwner, address(0));`
   * **Code:** `assertEq(ownerClaimed, 0, "Owner cannot claim strategy's rewards");`

**Conclusion:**

The test (`test_aave_rewards_cannot_be_claimed`) definitively proves two things:

1. The strategy contract has **no code to claim rewards** (Step 2).
2. No other entity **can claim rewards on its behalf** (Step 3).

Therefore, when Aave enables an incentive program for the Arbitrum USDC pool, 100% of the rewards accrued by this strategy will be **permanently and irretrievably frozen**.
