# #46546 \[SC-Insight] Accounting Mismatches in AgentVault.sol Due to Non-Standard ERC20 Tokens

**Submitted on Jun 1st 2025 at 14:42:18 UTC by @EFCCWEB3 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46546
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/AgentVault.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Theft of unclaimed yield
  * Permanent freezing of unclaimed yield
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The [AgentVault.sol](https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/AgentVault.sol) contract, integral to the FAssets system for managing vault collateral, is vulnerable to accounting mismatches when handling non-standard ERC20 tokens, such as Fee-on-Transfer (FOT) tokens (e.g., PAXG), rebasing tokens (e.g., stETH), and tokens with variable balances or airdrops. These mismatches occur because the contract assumes the exact transfer amount is received, while FOT fees, rebasing, or external balance changes cause discrepancies between the recorded collateral (via `assetManager`) and the actual token balance. In production, this could result in permanent freezing of funds, inability to withdraw collateral, loss of collateral for users (especially the last to withdraw), and additional costs for the protocol to cover shortfalls, undermining the FAssets system’s guarantee of redeemability.

## Vulnerability Details

The FAssets system allows any governance-approved ERC20 token on the Flare blockchain as vault collateral, including stablecoins like USDC or USDT, and potentially non-standard tokens like PAXG (FOT), stETH (rebasing), or tokens with airdrops or variable balances, as noted in the FAssets documentation: *“Along with Flare's native token, FLR, any governance approved ERC-20 token on the Flare blockchain can be used as collateral.”* The `AgentVault.sol` contract manages this collateral using `SafeERC20.safeTransferFrom` and `SafeERC20.safeTransfer` in functions like `depositCollateral`, `withdrawCollateral`, payout, and destroy, assuming the specified `_amount` is exactly transferred. However, non-standard ERC20 tokens introduce discrepancies:

* **Fee-on-Transfer (FOT) Tokens:** Tokens like PAXG deduct a fee during transfers. In `depositCollateral`, the contract calls \_token.safeTransferFrom(msg.sender, address(this), \_amount) and updates the collateral via `assetManager.updateCollateral`, assuming `_amount` is received. However, the contract receives `_amount - fee`, leading to overstated collateral. For example:

```solidity
function depositCollateral(IERC20 _token, uint256 _amount) external override onlyOwner {
    _token.safeTransferFrom(msg.sender, address(this), _amount);// <---
    assetManager.updateCollateral(address(this), _token);
    _tokenUsed(_token, TOKEN_DEPOSIT);
}
```

If 100 PAXG is deposited with a 0.1% fee, the contract receives 99.9 PAXG but records 100 PAXG with `assetManager`.

* **Rebasing Tokens:** Tokens like stETH adjust balances automatically. Negative rebasing reduces the contract’s balance, while positive rebasing adds untracked tokens. The 1 wei corner case in stETH can also cause small transfer losses. In `withdrawCollateral`, the contract attempts to transfer `_amount`:

```solidity

function withdrawCollateral(IERC20 _token, uint256 _amount, address _recipient) external override onlyOwner nonReentrant {
    assetManager.beforeCollateralWithdrawal(_token, _amount);
    _token.safeTransfer(_recipient, _amount);
}
```

If negative rebasing reduces the balance below `_amount`, the transfer fails. Positive rebasing gains are unclaimable for collateral tokens due to restrictions in `transferExternalToken`.

* **Variable Balance/Airdrop Tokens:** External balance changes or airdrops can increase or decrease the contract’s balance. Excess tokens from airdrops for collateral tokens are locked, as `transferExternalToken` excludes collateral tokens:

```
solidity

function transferExternalToken(IERC20 _token, uint256 _amount) external override onlyOwner nonReentrant {
    require(!assetManager.isLockedVaultToken(address(this), _token), "only non-collateral tokens");
    _token.safeTransfer(ownerManagementAddress, _amount);
}
```

These mismatches cause `assetManager` to track incorrect collateral amounts, leading to potential failures in `withdrawCollateral`, payout, or destroy, and permanently locking excess tokens from airdrops or positive rebasing. The FAssets documentation’s allowance of any governance-approved ERC20 token increases the likelihood of such tokens being used, amplifying the vulnerability.

**What are Rebasing Tokens?**

Rebasing tokens, like stETH (Lრ�

* **System:** Lido Staked ETH), automatically adjust their balances in holders’ wallets based on certain conditions, such as market performance or staking rewards. For example:
* **Negative Rebasing:**\* The balance decreases if the underlying asset’s value drops (e.g., if ETH’s value decreases relative to stETH).
* **Positive Rebasing:** The balance increases if the underlying asset’s value grows (e.g., staking rewards increase stETH holdings).

How Does This Cause Issues in `AgentVault.sol`? The `AgentVault.sol` contract doesn’t account for these automatic balance changes. Let’s look at the `withdrawCollateral` function:

```solidity
function withdrawCollateral(IERC20 _token, uint256 _amount, address _recipient) external override onlyOwner nonReentrant {
    assetManager.beforeCollateralWithdrawal(_token, _amount);
    _token.safeTransfer(_recipient, _amount);
}
```

**What Happens?**

* When an agent deposits stETH as collateral via depositCollateral, the contract calls assetManager.updateCollateral to record `_amount` (e.g., 100 stETH). However, it doesn’t check the actual amount received.
* If stETH negatively rebases (e.g., the contract’s balance drops to 99 stETH due to market changes), the recorded collateral (100 stETH) is higher than the actual balance (99 stETH).
* When the agent tries to withdraw 100 stETH using withdrawCollateral, the safeTransfer call fails because there aren’t enough tokens (only 99 stETH available).
* If stETH positively rebases (e.g., the balance increases to 101 stETH due to staking rewards), the extra 1 stETH is not recorded by assetManager. These extra tokens are unclaimable because the transferExternalToken function prevents transferring collateral tokens:

> **stETH’s 1 Wei Corner Case:** As noted in the Lido documentation, stETH transfers can sometimes result in receiving 1 wei less than expected due to rounding issues. This causes small, cumulative losses in the contract’s balance compared to the recorded amount.

## Impact Details

The vulnerability aligns with the following in-scope impacts:

* **Smart Contract Unable to Operate Due to Lack of Token Funds:** Cumulative FOT fees or negative rebasing (e.g., stETH losing value) can reduce the contract’s balance below the recorded collateral, causing withdrawCollateral or payout to fail. In Alice’s scenario, the 2 PAXG shortfall prevents withdrawal, halting contract operations for that vault and undermining FAsset redemption guarantees.
* **Permanent Freezing of Funds:** Excess tokens from positive rebasing or airdrops (e.g., 10 PAXG from an airdrop) are unclaimable for collateral tokens, as transferExternalToken restricts collateral tokens, locking these funds indefinitely and violating the FAssets system’s collateral accessibility.
* **Permanent Freezing of Unclaimed Yield:** Collateral, akin to yield in the FAssets system as it backs minted assets, becomes unclaimable if the contract’s balance is insufficient. In Alice’s case, 2 PAXG of recorded collateral is permanently frozen due to the shortfall.
* **Theft of Unclaimed Yield:** The shortfall from FOT fees or rebasing is borne by the last user (e.g., Alice), who receives less collateral than recorded, resembling theft of their expected share. In the PoC, Alice loses 2 PAXG of her collateral.

## References

Lido: <https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case>

Scope: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/AgentVault.sol>

## Proof of Concept

## Proof of Concept

Alice, an agent in the FAssets system, creates a vault in `AgentVault.sol` and selects PAXG, a governance-approved ERC20 token with a 0.1% transfer fee, as her vault collateral to back minted FAssets. She deposits 1000 PAXG via `depositCollateral`. Due to the fee, the contract receives 999 PAXG (`1000 - 0.1% = 1 PAXG fee`), but `assetManager.updateCollateral` records 1000 PAXG. Later, Alice deposits another 1000 PAXG, so the contract receives 999 PAXG (total balance: 1998 PAXG), but assetManager records 2000 PAXG. When Alice attempts to withdraw her 2000 PAXG collateral via `withdrawCollateral` after redeeming FAssets, the contract tries to transfer 2000 PAXG but fails, as only 1998 PAXG is available, preventing access to 2 PAXG of her collateral. Additionally, if PAXG receives a 10-token airdrop, these tokens are locked in the vault, as transferExternalToken cannot be used for collateral tokens. If Alice’s vault is destroyed via destroy, only 1998 PAXG is transferred, permanently losing 2 PAXG, and the 10 airdrop tokens remain unclaimable.
