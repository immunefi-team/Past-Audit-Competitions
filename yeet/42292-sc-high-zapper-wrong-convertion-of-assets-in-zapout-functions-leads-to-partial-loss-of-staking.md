# 42292 sc high zapper wrong convertion of assets in zapout functions leads to partial loss of staking rewards

## #42292 \[SC-High] Zapper: Wrong Convertion of Assets in zapOut Functions Leads to Partial Loss of Staking Rewards

**Submitted on Mar 22nd 2025 at 14:12:16 UTC by @Ace30 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42292
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

### Description

### Brief/Introduction

In the Zapper contract, the function `zapOutToToken0` calls `_yeetOut` to convert vault share tokens into `token0` and `token1`.\
However, after receiving tokens, it does not swap **all amount** of the received `token1` (`token1Debt`) into `token0`. Instead, it only swaps `swapData.inputAmount` (a user-specified amount) of `token1`. Additionally, the remaining `token1Debt` is sent to `msg.sender` (which is `StakeV2`) and user does not get this part of his funds.

The same issue occurs in `zapOutToToken1`, `zapOutNative`, and `zapOut`, leading to a partial swap of assets and loss of funds for users.

### Vulnerability Details

Users can claim staking rewards in `token0` by calling `StakeV2.claimRewardsInToken0` as shown below:

````solidity
function claimRewardsInToken0(
    uint256 amountToWithdraw,
    IZapper.SingleTokenSwap calldata swapData,
    IZapper.KodiakVaultUnstakingParams calldata unstakeParams,
    IZapper.VaultRedeemParams calldata redeemParams
) external nonReentrant {
    --snip--
    uint256 receivedAmount = zapper.zapOutToToken0(msg.sender, swapData, unstakeParams, updatedRedeemParams);
}
This function internally calls `zapper.zapOutToToken0()`, which follows these steps:

1- Calls `_yeetOut()` to redeem vault shares and remove liquidity from Kodiak, receiving token0 and token1.

2-  However, instead of swapping all received token1 into token0, it only swaps `swapData.inputAmount` of token1.

3- The remaining `token1Debt` is sent to msg.sender (which is StakeV2 - not user), causing users to lose part of their funds.

```solidity
    function zapOutToToken0(
        address receiver,
        SingleTokenSwap calldata swapData,
        KodiakVaultUnstakingParams calldata unstakeParams,
        VaultRedeemParams calldata redeemParams
    ) public nonReentrant onlyWhitelistedKodiakVaults(unstakeParams.kodiakVault) returns (uint256 totalToken0Out) {
@1>     (IERC20 token0, IERC20 token1, uint256 token0Debt, uint256 token1Debt) = _yeetOut(redeemParams, unstakeParams);
        if (token0Debt == 0 && token1Debt == 0) {
            return (0);
        }
@2>     token1Debt -= swapData.inputAmount;
        token0Debt += _verifyTokenAndSwap(swapData, address(token1), address(token0), address(this));
        _sendERC20Token(token0, receiver, token0Debt);
@3>     _sendERC20Token(token1, _msgSender(), token1Debt);
        return (token0Debt);
    }
````

At step 2 (`@2>`) three possible scenarios arise:

1. If `swap.inputAmount > token1Debt` then the function will revert because of underflow
2. If `swap.inputAmount < token1Debt` then the remaining `token1Debt` is sent to `StakeV2`, resulting in a loss for the user.
3. If `swap.inputAmount = token1Debt` No issue occurs, but this scenario is very rare since amount of token 1 received (`token1Debt`) fluctuates due to changes in vault share price and Kodiak liquidity.

**Note:** Since any action on Kodiak Island affects the received amount of token0 and token1, in the current codebase, users must provide an additional 1-2% inputAmount to account for these fluctuations and prevent reverts.

**Note** Similar issue happens in other `StakeV2` claim functions too: `claimRewardsInToken1`, `claimRewardsInToken`, and `claimRewardsInNative`.

**Solution**\
A better solution is to fully convert the received token1 into token0. To determine the minAmountOut for the swap (to mitigate slippage), the protocol can obtain the user’s acceptable slippage percentage, retrieve the on-chain price from the swapper, and calculate the minAmountOut accordingly.

### Impact Details

All users trying to claim their staking rewards will experience either transaction revert or a partial loss of rewards

### References

Zapper.zapOutToToken0():\
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L249>

### Proof of Concept

## Proof of Concept (PoC)

**Step 1: User Initiates Staking Reward Claim**\
The user calls `StakeV2.claimRewardsInToken0()` to redeem **100 vault share tokens** and receive **token0** as a reward.

* Based on on-chain data, the user expects to receive:
  * **200 token0**
  * **100 token1**
* The user sets `swapData.inputAmount = 100`, expecting that all received `token1` will be converted to `token0`.

**Step 2: `zapOutToToken0()` Execution Begins**

`zapOutToToken0()` is called within `StakeV2.claimRewardsInToken0()`.\
Internally, it calls `_yeetOut()` to:

1. **Withdraw from the farm**
2. **Redeem vault shares from the Kodiak vault**
3. **Remove liquidity from Kodiak Island**

Due to on-chain fluctuations (other transactions affecting liquidity), the user **actually receives**:

* **210 token0**
* **110 token1**

**Step 3: Partial Token1 Swap Occurs**

The contract executes the following steps inside `zapOutToToken0()`:

1. Swaps `swapData.inputAmount = 100` of `token1` into `token0`.
2. Assume the swap gives back **200 token0**.
3. Now, the contract holds:
   * **210 (original) + 200 (swap) = 410 token0**
   * **110 (original) - 100 (swapped) = 10 token1**

**Step 4: Incorrect Fund Distribution**

After the swap, the contract distributes funds:

1. **User receives:**
   * **410 token0**
2. **StakeV2 (instead of the user) receives:**
   * **10 token1 (remaining token1Debt)**

**Expected Behavior:**

* The contract should have converted the full **110 token1** into `token0`, ensuring the user gets their **full expected amount**.

**Actual Issue:**

* Only **100 token1** was converted, leaving **10 token1 unconverted**.
* The unconverted **10 token1** is sent to `StakeV2` instead of the user, resulting in a **loss of funds**.
