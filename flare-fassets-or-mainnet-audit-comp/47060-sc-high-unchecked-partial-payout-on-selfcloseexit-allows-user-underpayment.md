# #47060 \[SC-High] Unchecked Partial Payout on selfCloseExit Allows User Underpayment

**Submitted on Jun 8th 2025 at 16:48:13 UTC by @RNemes for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47060
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/RedemptionRequests.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The `selfCloseExit` function in the CollateralPool contract allows users to exit the pool and redeem their f-assets for collateral. However, due to unchecked return values in the downstream `redeemFromAgentInCollateral` call, users may receive less collateral than expected if the agent vault is underfunded. This results in direct user loss, as the user's f-assets are fully burned regardless of the actual payout.

## Vulnerability Details

When a user calls `selfCloseExit`, the function ultimately routes to `_selfCloseExitTo`, which, for small redemptions or when requested, calls `assetManager.redeemFromAgentInCollateral`. Inside this function, the protocol attempts to pay the redeemer in collateral (NAT) by calling `Agents.payoutFromVault`. Critically, the return value of this function—which indicates the actual amount transferred—is not checked. If the agent's vault does not have enough collateral, `payoutFromVault` will transfer as much as it can (potentially less than the required amount), but the function does not revert or compensate for the shortfall. The user's f-assets are still fully burned, regardless of the actual payout.

**Relevant code snippets:**

```solidity
// contracts/assetManager/implementation/CollateralPool.sol
function selfCloseExit(
    uint256 _tokenShare,
    bool _redeemToCollateral,
    string memory _redeemerUnderlyingAddress,
    address payable _executor
) external payable nonReentrant {
    _selfCloseExitTo(
        _tokenShare,
        _redeemToCollateral,
        payable(msg.sender),
        _redeemerUnderlyingAddress,
        _executor
    );
}

// ...

// contracts/assetManager/implementation/CollateralPool.sol
function _selfCloseExitTo(...) private {
    // ...
    if (requiredFAssets > 0) {
        if (requiredFAssets < assetManager.lotSize() || _redeemToCollateral) {
            assetManager.redeemFromAgentInCollateral(
                agentVault,
                _recipient,
                requiredFAssets
            );
        }
        // ...
    }
    // ...
}

// ...

// contracts/assetManager/library/RedemptionRequests.sol
function redeemFromAgentInCollateral(
    address _agentVault,
    address _redeemer,
    uint256 _amountUBA
) internal {
    // ...
    Agents.payoutFromVault(agent, _redeemer, paymentWei); // return value not checked
    // ...
    Redemptions.burnFAssets(msg.sender, closedUBA);
}
```

## Impact Details

* **Direct User Loss:** Users may lose f-assets and receive less collateral than expected, resulting in a direct financial loss.
* **Protocol Accounting Inconsistency:** The protocol assumes the f-assets are fully redeemed and burned, but the corresponding collateral is not fully released, leading to accounting mismatches.
* **Potential Exploitation:** Malicious agents could intentionally keep their vaults just below the required collateral, causing repeated underpayments and harming users.
* **No Automatic Fallback:** There is no fallback to pay the remainder from the pool or another source, unlike in some other redemption default scenarios.

## References

* contracts/assetManager/implementation/CollateralPool.sol
* contracts/assetManager/library/RedemptionRequests.sol

***

**Recommendation:**

* Always check the return value of `payoutFromVault` and revert or compensate the user if the full amount is not paid. Consider fallback mechanisms (e.g., paying the remainder from the pool) to ensure users are never underpaid for their f-assets.

## Proof of Concept

## Proof of Concept

**Scenario: Alice and Bob**

1. **Setup:**
   * Bob is an agent and operates an agent vault, but his vault is underfunded (e.g., it contains only 80 NAT, but should cover 100 NAT worth of f-assets).
   * Alice is a user who holds f-assets and wants to exit the collateral pool using `selfCloseExit`.
2. **Action:**
   * `selfCloseExit` is called on the CollateralPool, specifying parameters that will route to `redeemFromAgentInCollateral` (for example, a small redemption or `_redeemToCollateral = true`) for Alice to redeem her collateral.
3. **Result:**
   * The protocol burns Alice's f-assets equivalent to 100 NAT.
   * The protocol attempts to pay Alice 100 NAT from Bob's agent vault, but since the vault only has 80 NAT, Alice receives just 80 NAT.
   * The transaction does not revert, and Alice is underpaid by 20 NAT.
4. **Consequence:**
   * Alice loses f-assets worth 100 NAT but receives only 80 NAT in return.
   * Bob's vault is now empty, but the protocol considers the redemption complete.

**Summary:**

* Alice is left with a direct financial loss, and the protocol's accounting is inconsistent with the actual collateral movement.
