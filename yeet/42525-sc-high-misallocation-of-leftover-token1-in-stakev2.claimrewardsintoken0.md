# #42525 \[SC-High] Misallocation of leftover token1 in StakeV2.claimRewardsInToken0

**Submitted on Mar 24th 2025 at 14:03:34 UTC by @nnez for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42525
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Description

**Overview:**\
StakeV2 distributes rewards in the form of vault tokens. These vault tokens are backed by Kodiak IslandToken which is a token representing liquidity position on an AMM pool. In order to allow users to receive their rewards in a specific target token (token0), users can call StakeV2’s reward-claim function (`claimRewardsInToken0`). This function internally calls the Zapper’s `zapOutToToken0` function to convert the Vault tokens into token0.

See: <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L345-L359>

For the ease of understanding, let's say that token0 is Yeet and token1 is WBERA\
Inside Zapper, the process is as follows:

1. **Vault Token Redemption & Liquidity Removal:**\
   Zapper redeems the vault tokens and removes liquidity from Kodiak. This step yields two underlying tokens:\
   See: <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L255>
   * **token0 (Yeet)**
   * **token1 (WBERA)**
2. **Swap Operation:**\
   Zapper then attempts to swap token1 (WBERA) into token0 (Yeet) using a specified swap input amount, as defined by swapData.inputAmount. For example, the following code snippet shows the key steps:\
   See: <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol#L259-L262>

   ```solidity
   token1Debt -= swapData.inputAmount;
   token0Debt += _verifyTokenAndSwap(swapData, address(token1), address(token0), address(this));
   _sendERC20Token(token0, receiver, token0Debt);
   _sendERC20Token(token1, _msgSender(), token1Debt); // @audit: _msgSender() is StakeV2
   ```

   Here, any token1 remaining after the swap (i.e. token1Debt that isn’t covered by swapData.inputAmount) is returned to the caller, which in this case is StakeV2.
3. **Reward Delivery:**\
   The converted token0 (Yeet) is sent to the user as their reward. The expectation is that all value from token1 is fully converted into token0 so that the user receives the entire reward in Yeet.

### The Bug

The swap parameters (specifically swapData.inputAmount) are determined based on the state at a given moment before the transaction is sent. However, while the transaction is pending, market conditions or state variables (such as liquidity or token price) may change. Consequently, the actual token1 amount withdrawn from liquidity (token1Debt) may be higher than anticipated. The code then subtracts the pre-determined swapData.inputAmount from token1Debt:

```solidity
token1Debt -= swapData.inputAmount;
```

If token1Debt is greater than swapData.inputAmount due to these state changes, a residue of token1 (WBERA) remains unconverted. Instead of crediting the reward-claiming user with the full value, this leftover token1 is sent to StakeV2 (since \_msgSender() in Zapper is StakeV2), thereby misallocating part of the user’s reward.

### Example scenario

* A user is entitled to claim rewards via vault token redemption.
* The redemption and liquidity removal (handled inside Zapper) yield:
  * **190 Yeet (token0)**
  * **20 WBERA (token1)**
* Based on the state at transaction construction, the user sets swapData.inputAmount to swap **15 WBERA**, expecting to convert it fully to, say, **10 Yeet**.
* However, due to a state change (e.g., a price shift or increased liquidity withdrawal), the actual token1Debt is **20 WBERA**.
* The swap then converts 15 WBERA, and the code subtracts:

  ```solidity
  token1Debt = 20 - 15 = 5 WBERA;
  ```
* This leftover **5 WBERA** is returned to StakeV2 rather than to the user.
* As a result, the user receives **200 Yeet** (190 + 10), but the value represented by the unconverted 5 WBERA is misallocated into the overall rewards pool.

### Additional Note

This bug pattern is also present in other claim functions that zap out to tokens other than the vault token. Including: `claimRewardsInNative`, `claimRewardsInToken0`, `claimRewardsInToken1`, `claimRewardsInToken`

## Impact

The bug could cause a loss of portion of unclaimed yield for users. Instead of receiving the full value of their redemption, the claiming user misses out on the portion of the reward represented by the leftover token1 as it is sent to StakeV2 instead of the claimer.

It's noteworthy that the maximum loss of rewards for users are allowed slippage because the swapped amount of token1 to token0 must also be greater than specified `mintOutput` for transaction to be successfully executed.

## Recommended Mitigations

Add another option paramter to send unused tokens to receiver rather than arbitrary set it to msg.sender

## Proof of Concept

## Proof-of-Concept

Assumptions:

* token0 = Yeet
* token1 = WBERA

### Steps

**Step 1: Reward Accumulation and Claim Initiation**

* User A has accumulated rewards in Vault tokens.
* User A invokes `claimRewardsInToken0` on StakeV2 to convert their rewards into Yeet (token0).

**Step 2: Execution within Zapper**

* Zapper redeems the vault tokens and removes liquidity from Kodiak, yielding:
  * 190 Yeet (token0)
  * 20 WBERA (token1)
* Based on the state at the time of transaction construction, User A specifies swapData.inputAmount as 15 WBERA, expecting this to fully cover the necessary swap.

**Step 3: State Change and Incomplete Swap**

* While User A’s transaction is pending, market conditions change, and the actual token1Debt from liquidity removal is 20 WBERA.
* During the swap operation, Zapper converts 15 WBERA into 10 Yeet.
* The code then computes:

  ```solidity
  token1Debt = 20 - 15 = 5 WBERA;
  ```
* These 5 WBERA remain unswapped.

**Step 4: Misallocation of Leftover Token1**

* The Zapper function sends the swapped token0 (totaling 200 Yeet) to User A.
* The unswapped 5 WBERA is returned to StakeV2 (since \_msgSender() in Zapper is StakeV2) instead of being credited to User A.
* Consequently, these 5 WBERA become part of the general rewards pool, unclaimable by user A
