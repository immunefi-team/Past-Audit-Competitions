# #41875 \[SC-High] Permanent Lock of User Funds in StakeV2 Due to Incorrect token Debt Handling

**Submitted on Mar 19th 2025 at 03:17:58 UTC by @Bluedragon for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41875
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary:

When users claim rewards in native tokens using the `claimRewardsInNative` function in the `StakeV2` contract, any remaining token debt (e.g., `token0Debt` or `token1Debt`) from the Zapper contract is sent back to the `StakeV2` contract instead of the user. This occurs because the `msg.sender` in the context of Zapper contract is `StakeV2`, not the actual user. As a result, users lose access to their funds, and the tokens are permanently locked in the `StakeV2` contract.

## Vulnerability Details:

The issue arises in the `claimRewardsInNative` function of the `StakeV2` contract. When a user claims rewards, the `zapOutNative` function in the Zapper contract is called. If there is any remaining token debt (e.g., `token0Debt` or `token1Debt`), the Zapper contract uses the `_clearUserDebt` function to send these tokens back to the `msg.sender`. However, since the `msg.sender` in this context is the `StakeV2` contract, the tokens are sent to `StakeV2` instead of the user. This results in the loss of user funds, as the tokens are permanently locked in the `StakeV2` contract.

**List Of Affected Functions**:

* `StakeV2::claimRewardsInNative`
* `StakeV2::claimRewardsInToken0`
* `StakeV2::claimRewardsInToken1`
* `StakeV2::claimRewardsInToken`

## Impact:

* **Loss of User Funds**: Users lose access to their tokens (e.g., `token0` or `token1`) when claiming rewards in other tokens with custom swap data. These tokens are permanently locked in the `StakeV2` contract.
* **Permanent Lock of Tokens**: Tokens sent to the `StakeV2` contract cannot be recovered, as there is no mechanism to withdraw them.

## Reference Code Snippet:

* \[StakeV2::claimRewardsInNative] (<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L327>)
* \[Zapper::zapOut] (<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L352>)
* \[Zapper::\_swapBera] (<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L622>)

## Proof of Concept

## Proof of Concept:

1. **Scenario**:
   * User `bluedragon` has 600e18 rewards in the `StakeV2` contract.
   * `bluedragon` wants to withdraw these rewards in native BERA and uses the `claimRewardsInNative` function.
   * As custom swap data can be used for claiming rewards, `bluedragon` provides custom swap data to convert 300e18 of the rewards to BERA and the remaining 300e18 to `token0` (e.g., USDbr). \[Assuming 1 BERA = 6 USDbr.]
   * The `claimRewardsInNative` function calls the `zapOutNative` function in the Zapper contract.
   * The Zapper contract swaps 300e18 of the rewards to BERA and 300e18 to `token0` (USDbr).
   * After the swap, the Zapper contract has 50e18 WBERA (converted to BERA) and 300e18 USDbr.
   * The Zapper contract sends 50e18 BERA to `bluedragon`.
   * The remaining 300e18 USDbr is sent to the `StakeV2` contract via the `_clearUserDebt` function, as the `msg.sender` is `StakeV2`.
2. **Issue**:
   * The 300e18 USDbr send to the `StakeV2` contract is permanently locked, and `bluedragon` loses access to these funds.
