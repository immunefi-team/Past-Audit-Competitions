# #46378 \[SC-High] Unconditional F-Asset burn during partial collateral redemptions enables direct theft of user funds

**Submitted on May 29th 2025 at 07:17:13 UTC by @DSbeX for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46378
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/RedemptionRequests.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `RedemptionRequests::redeemFromAgentInCollateral` function permanently burns users' F-assets before verifying successful collateral payout. When agent vaults are undercollateralized, users lose their F-assets while receiving only partial collateral (or nothing), enabling direct theft of user funds.

## Vulnerability Details

Agents remain operational during undercollateralization periods before liquidation triggers. The protocol fails to suspend redemptions or implement safeguards during this danger zone, allowing agents to process requests while incapable of full payment.

The vulnerability stems from two flawed interactions:

1. Partial Payments Without Reversion: The `payoutFromVault` function uses `Math.min()` to return partial collateral when agent vaults are undercollateralized, without reverting the transaction:

```javascript
_amountPaid = Math.min(_amountWei, collateral.token.balanceOf(address(vault)));
```

2. Unconditional F-Asset Burn: Immediately after payment, `burnFAssets` destroys the full F-asset amount regardless of actual collateral received:

```javascript
Redemptions.burnFAssets(msg.sender, closedUBA); // Burns entire amount
```

This creates a value imbalance where users permanently lose F-asset value exceeding the collateral received. The vulnerability is exacerbated by: Lack of validation between paid collateral and burned assets

No mechanism to refund/protect partially redeemed F-assets

Silent acceptance of undercollateralized payouts

## Impact Details

New Attack Vector: Malicious agents can intentionally hover just above liquidation thresholds to steal funds

Unintentional Harm: Legitimate agents facing temporary price drops harm users unintentionally

Systemic Risk: All users redeeming during collateral dips face uncompensated asset loss

## References

Vulnerable function: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/RedemptionRequests.sol#L97>

Partial payment logic: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Agents.sol#L252>

## Proof of Concept

## Proof of Concept

Scenario Setup:

```
Agent Collateral: 1 ETH ($2,000 value)

Agent Liability: 1,800 FUSD ($1,800 value)

ETH Price: Drops to $1,800 (agent becomes undercollateralized)

User redeems: 2,200 FUSD ($2,200 value)
```

Attack Flow:

1. Redemption Request:

   User calls `redeemFromAgentInCollateral(2,200 FUSD)`

   Protocol calculates required collateral:

```javascript
Required Collateral = (2,200 FUSD ÷ 1,800 FUSD) × 1 ETH = 1.222 ETH
```

paymentWei = 1.222 ETH (in wei equivalent)

2. Partial Payment Execution:

```javascript
// In payoutFromVault():
_amountPaid = min(1.222 ETH, 1 ETH) = 1 ETH
```

User receives only 1 ETH ($1,800 value at current prices)

3. Unconditional Asset Burn:

```javascript
Redemptions.burnFAssets(msg.sender, 2,200 FUSD) // $2,200 value destroyed
```

4. Final Position:

User receives: 1 ETH ($1,800)

User loses: 2,200 FUSD ($2,200)

Net Loss: $400 (18.2% of redeemed value)

Agent Gains: Liability reduced by $2,200 while paying only $1,800 worth of collateral
