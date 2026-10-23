# #41831 \[SC-Critical] Miscalculation of excess rewards via external token transfers leads to contract insolvency and incomplete withdrawals

**Submitted on Mar 18th 2025 at 18:24:43 UTC by @vladi319 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41831
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Protocol insolvency
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The StakeV2 contract miscalculates excess rewards by using the difference between the contract’s token balance and the total staked tokens. External transfers to the contract can inflate this balance, causing erroneous reward distributions that deplete the tokens reserved for user withdrawals, potentially locking user funds and rendering the contract insolvent.

## Vulnerability Details

The vulnerability lies in the function `accumulatedDeptRewardsYeet()`, which computes excess rewards using the formula:

```solidity
return stakingToken.balanceOf(address(this)) - totalSupply;
```

This calculation assumes that all tokens held by the contract are a direct result of user stakes. However, if tokens are directly transferred to the contract—outside the standard staking process—the balance becomes artificially inflated. In the provided PoC, an external transfer of 100 ether worth of tokens is made to the contract, leading it to misinterpret this as an excess reward. When the `executeRewardDistributionYeet` function is subsequently called, it processes these “excess” tokens through the reward distribution mechanism. As a result, tokens that are not part of the users’ staked balances are used up, leaving the contract with insufficient tokens to cover legitimate withdrawal requests.

## Impact Details

Exploitation of this vulnerability can lead to significant financial losses:

* Insolvency: The contract becomes insolvent, as the reward distribution mechanism depletes tokens required for full user withdrawals.
* User Funds Locked: Users may are unable to withdraw the full amount of their staked tokens, effectively locking their funds within the contract.

## References

Link to the code:\
<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol?utm\\_source=immunefi#L148-L150>

## Proof of Concept

## Proof of Concept

1. Setup: Staking and Accounting
   * Owner stakes: 100 tokens
   * User stakes: 200 tokens
   * Total Supply: 100 + 200 = 300 tokens
   * Contract’s Token Balance: 300 tokens (all coming from proper stake calls)
2. Accumulation of “Excess Rewards”
   * Under normal operation, the contract is expected to receive rewards via methods such as `depositReward()`. In addition, due to swap inefficiencies or operational residues, extra tokens may be added indirectly.
   * Suppose that, as a result of these inefficiencies, the contract ends up with an additional 100 tokens.
   * New Token Balance: 300 (from stakes) + 100 (from inefficiencies/rewards) = 400 tokens
   * The function `accumulatedDeptRewardsYeet()` then computes the “excess” as:

     ```
     excessRewards = stakingToken.balanceOf(address(this)) - totalSupply 
                   = 400 - 300 = 100 tokens.
     ```

     *(Note: Depending on timing, this difference may be larger if unstaking has already reduced `totalSupply`.)*
3. Reward Distribution and Balance Reduction
   * A manager calls `executeRewardDistributionYeet()`. In this function:
     * The excess rewards amount is determined by calling `accumulatedDeptRewardsYeet()`.
     * The code then executes:

       ```solidity
       stakingToken.approve(address(zapper), accRevToken0);
       ```

       This approves the external zapper contract to spend the calculated excess tokens.
     * Next, one of the following external calls is made:

       ```solidity
       zapper.zapInToken0(swap, stakingParams, vaultParams);
       // or
       zapper.zapInToken1(swap, stakingParams, vaultParams);
       ```

       Crucial Point: The StakeV2 contract does not itself subtract these tokens from its balance. Instead, the approved tokens are transferred out when the external zapper function executes. In other words, the reduction of the StakeV2 contract’s token balance happens indirectly within the zapper’s `zapInToken0` or `zapInToken1` functions.
4. Unstaking and the Insolvency Scenario
   * Suppose after this reward distribution the following occurs:
     * The recorded `totalSupply` in the StakeV2 bookkeeping is adjusted only by user actions (for example, via `startUnstake`), but the actual token balance in the contract has been reduced by the external zapper call.
   * Example Scenario with Numbers:
     * Before unstaking, assume that due to an unstake action the `totalSupply` has been reduced to 200 tokens (because a user started unstaking 100 tokens), but the token balance was still 400 tokens.
     * When `executeRewardDistributionYeet()` is called at that point, it computes:

       ```
       excessRewards = 400 - 200 = 200 tokens.
       ```

       These 200 tokens are then approved and passed to the zapper, which transfers them out.
     * After reward distribution: The actual token balance in the contract is now 400 - 200 = 200 tokens.
     * Now, if a user (with a remaining recorded stake of, say, 200 tokens) tries to fully withdraw via a call to `rageQuit()`, the contract may not have enough tokens available due to the external zapper call having removed a significant portion of the tokens.
     * The result is that although the user’s balance (in the contract’s bookkeeping) indicates 200 tokens, the contract’s token balance is insufficient to honor the full withdrawal—effectively locking part of the user’s funds.

***

### Conclusion

* Balance Reduction Location:\
  The StakeV2 contract itself does not contain a direct line of code that subtracts tokens from its balance during reward distribution. Instead, the tokens are moved out when the external zapper contract is called via:

  ```solidity
  zapper.zapInToken0(swap, stakingParams, vaultParams);
  ```

  or

  ```solidity
  zapper.zapInToken1(swap, stakingParams, vaultParams);
  ```

  These external functions are responsible for transferring the approved excess tokens away from the StakeV2 contract.
* Validity of the Issue:\
  Because the reward distribution mechanism relies on the difference between the contract’s token balance and `totalSupply`, and since an external call removes tokens (reducing the actual balance) without corresponding adjustments in `totalSupply`, there is a risk that users will be unable to withdraw the full amount they are entitled to. This misalignment can lead to a situation where the contract appears solvent on paper but lacks sufficient tokens to satisfy withdrawal requests, thereby locking user funds.

```solidity
// SPDX-License-Identifier: AGPL-3.0-or-later
pragma solidity ^0.8.19;

// Import Foundry's Test framework (forge-std) and necessary interfaces/mocks.
// Note: The following import paths assume your project structure.
import "forge-std/Test.sol";
import "../src/StakeV2.sol";
import "../src/interfaces/IWETH.sol";
import "../src/interfaces/IZapper.sol";
import "./mocks/TokenMock.sol";
import "./mocks/SimpleZapperMock.sol";
import "./mocks/KodiakVaultV1.sol";

contract StakeV2_HandleExcessDebt_Test is Test {
    TokenMock token;
    IWETH wbera;
    SimpleZapperMock mockZapper;
    KodiakVaultV1 kodiakVault;
    StakeV2 stakeV2;
    address owner;
    address manager;
    address user;

    function setUp() public {
        owner = address(this);
        manager = address(this);
        // Deploy a mock token (e.g., 18 decimals)
        token = new TokenMock("Test Token", "TTK", 18);
        // For wbera, deploy a mock token as well (casting it to IWETH)
        wbera = IWETH(address(new TokenMock("WETH", "WETH", 18)));
        // Deploy a mock KodiakVaultV1 with token addresses from token and wbera
        kodiakVault = new KodiakVaultV1(address(token), address(wbera));
        // Deploy the zapper mock with token0 and token1 from kodiakVault
        mockZapper = new SimpleZapperMock(kodiakVault.token0(), kodiakVault.token1());
        // Deploy StakeV2 contract with the mock zapper
        stakeV2 = new StakeV2(IERC20(address(token)), IZapper(address(mockZapper)), owner, manager, wbera);

        // Create a test user
        user = vm.addr(1);
    }

    function test_break_contract_differentNumbers() public {
        // --- Setup: Staking and Accounting ---
        // Owner stakes: 150 tokens
        token.mint(owner, 300 ether);
        token.approve(address(stakeV2), 150 ether);
        stakeV2.stake(150 ether);

        // User stakes: 250 tokens
        token.mint(user, 300 ether);
        vm.startPrank(user);
        token.approve(address(stakeV2), 250 ether);
        stakeV2.stake(250 ether);
        vm.stopPrank();
        // At this point:
        //   Total Supply = 150 + 250 = 400 tokens.
        //   Contract token balance = 400 tokens (all from proper staking).

        // --- Accumulation of "Excess Rewards" ---
        // Simulate swap inefficiencies or operational residues by transferring extra tokens
        token.transfer(address(stakeV2), 150 ether);
        // New Contract Balance = 400 + 150 = 550 tokens.
        // The function accumulatedDeptRewardsYeet() will now return:
        //    excessRewards = 550 - 400 = 150 tokens.
        
        // Set the zapper mock's return values (these values simulate the external call behavior)
        mockZapper.setReturnValues(2, 2);

        // Deposit native reward to simulate a depositReward call
        stakeV2.depositReward{value: 2 ether}();

        // --- Start Unstake Process ---
        // Owner initiates unstake for 150 tokens.
        // This reduces the recorded totalSupply: 400 - 150 = 250 tokens.
        stakeV2.startUnstake(150 ether);

        // --- Reward Distribution and Balance Reduction ---
        // At this point:
        //   Contract token balance remains 550 tokens,
        //   Recorded totalSupply is 250 tokens,
        //   Hence, excessRewards = 550 - 250 = 300 tokens.
        stakeV2.executeRewardDistributionYeet(
            IZapper.SingleTokenSwap(
                token.balanceOf(address(stakeV2)) - stakeV2.totalSupply(), // 300 tokens in this example
                0,
                0,
                address(0),
                ""
            ),
            IZapper.KodiakVaultStakingParams(address(kodiakVault), 0, 0, 0, 0, 0, address(0)),
            IZapper.VaultDepositParams(address(0), address(0), 0)
        );
        // Note: The approved 300 tokens are transferred out via the external zapper call,
        //       reducing the StakeV2 contract's token balance indirectly.

        // --- Unstaking and the Insolvency Scenario ---
        // Owner completes unstake via rageQuit (this withdraws the tokens corresponding to the vesting).
        stakeV2.rageQuit(0);

        // --- Logging Balances ---
        console.log("Contract token balance: ", token.balanceOf(address(stakeV2)) / 1e18);
        console.log("Owner recorded stake: ", stakeV2.balanceOf(owner) / 1e18);
        console.log("User recorded stake: ", stakeV2.balanceOf(user) / 1e18);

        // Expected Outcome:
        //   After the external zapper call, the StakeV2 contract's token balance is reduced (e.g., 550 - 300 = 250 tokens).
        //   Yet, the bookkeeping still reflects a higher total staked amount.
        //   Therefore, if a user with a recorded stake (e.g., 250 tokens) attempts a full withdrawal,
        //   the contract will lack sufficient tokens, effectively locking some user funds.
    }
}
```

***

### Explanation Recap

1. Setup:
   * Owner stakes 150 tokens and user stakes 250 tokens, so the recorded total supply is 400 tokens.
   * The contract’s token balance is initially 400 tokens.
2. Accumulation of Excess Rewards:
   * Extra 150 tokens are added (to simulate inefficiencies), making the balance 550 tokens.
   * The excess rewards calculated become 550 – 400 = 150 tokens (though later, after unstake, the excess becomes 550 – 250 = 300 tokens).
3. Reward Distribution:
   * When `executeRewardDistributionYeet()` is called, 300 tokens are approved for the external zapper call.
   * The external zapper (via `zapInToken0` or `zapInToken1`) transfers out these tokens, reducing the contract’s balance indirectly.
4. Unstaking and Insolvency:
   * With the bookkeeping still showing higher stakes but the actual token balance reduced by the external call, a user trying to withdraw their full recorded amount will find insufficient tokens available, effectively locking their funds.
