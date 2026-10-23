# #47108 \[SC-High] selfCloseExitTo() can cause users to receive partial payments without validation, leading to permanent asset loss

**Submitted on Jun 9th 2025 at 01:55:29 UTC by @rilwan99 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47108
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Agents.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The selfCloseExitTo() function in CollateralPool.sol allows users to exit the pool and redeem FAssets for agent vault collateral, but fails to validate that the agent vault has sufficient collateral balance before proceeding. When the agent vault has insufficient funds, users receive only partial payments while their CPT tokens are fully burned, resulting in permanent and unrecoverable asset loss with no transaction revert or compensation mechanism.

## Vulnerability Details

The vulnerability exists in the \_selfCloseExitTo() function when users choose to redeem FAssets for agent vault collateral (\_redeemToCollateral = true). The function calls assetManager.redeemFromAgentInCollateral() without validating the agent vault's collateral balance:

```solidity
// In CollateralPool.sol _selfCloseExitTo()
if (requiredFAssets < assetManager.lotSize() || _redeemToCollateral) {
    // @audit BUG No check on agent vault balance -> Can lead to user receiving a loss
    assetManager.redeemFromAgentInCollateral(
        agentVault, _recipient, requiredFAssets);
}
```

This leads to a call chain: `_selfCloseExitTo()` → `RedemptionRequests.redeemFromAgentInCollateral()` → `Agents.payoutFromVault()`, where the critical flaw occurs:

```solidity
// In Agents.sol payoutFromVault()
function payoutFromVault(
    Agent.State storage _agent,
    address _receiver,
    uint256 _amountWei
) internal returns (uint256 _amountPaid) {
    CollateralTypeInt.Data storage collateral = getVaultCollateral(_agent);
    // @audit This allows partial payments without reverting
    _amountPaid = Math.min(_amountWei, collateral.token.balanceOf(address(vault)));
    vault.payout(collateral.token, _receiver, _amountPaid);
}
```

The Math.min() logic ensures that if the vault has insufficient collateral, only the available amount is paid out, with no revert or compensation mechanism. The user's CPT tokens are still burned via token.burn(msg.sender, \_tokenShare, false) regardless of whether they received their full entitlement.

**Contrast with underlying asset redemption**: The code properly handles insufficient capacity for underlying asset redemptions by scaling down the entire operation proportionally and emitting an IncompleteSelfCloseExit event. However, no such protection exists for vault collateral redemptions.

```solidity
// Proper handling for underlying assets
uint256 maxAgentRedemption = assetManager.maxRedemptionFromAgent(agentVault);
if (maxAgentRedemption < requiredFAssets) {
    requiredFAssets = maxAgentRedemption;
    natShare = _getNatRequiredToNotSpoilCR(assetData, requiredFAssets);
    _tokenShare = assetData.poolTokenSupply.mulDiv(natShare, assetData.poolNatBalance);
    emit IncompleteSelfCloseExit(_tokenShare, requiredFAssets);
}
```

## Impact Details

Severity: **High - Direct permanent asset loss for users**

Financial Impact:

This vulnerability results in permanent loss of funds for CPT holders. The maximum loss per transaction is capped at the difference between the value of FAssets being redeemed and the agent vault's available collateral balance.

Permanent Nature of Loss:

Once CPT tokens are burned, they cannot be recovered, leaving users with no means of compensation or protection. The burned tokens represent the user's proportional ownership in the collateral pool, and their destruction is irreversible regardless of whether the user received their full entitlement. Protocol Responsibility:

**This is NOT a user error**. The responsibility lies with the protocol to ensure that when users' CPT tokens are burned, they receive equivalent value in return. The protocol should either:

* Validate sufficient agent vault balance before proceeding, or
* Scale down the CPT token burn proportionally to match the available payout

## References

<https://github.com/flare-labs-ltd/fassets/blob/main/contracts/assetManager/library/Agents.sol>

## Proof of Concept

## Proof of Concept

Assume the following: Collateral Pool State:

* Total collateral: 500 NAT
* Total CPT supply: 100 tokens
* Alice holds 50 CPT tokens (50% of pool)
* Pool has accumulated 50 FAssets in fees
* Alice has earned 10 FAssets in fees

Market Conditions:

* NAT (FLR) price: $2
* FAsset (XRP) price: $3
* USDC price: $1

Agent State:

* Agent is backing 300 FAssets ($900 total value)
* Agent vault contains: 400 USDC ($400)
* Pool collateral: 500 NAT ($1,000)

Collateral Ratios (Both Below Required):

* Vault CR: $400 USDC / $900 FAssets = 0.44 (Below required 1.5)
* Pool CR: $1,000 NAT / $900 FAssets = 1.11 (Below required 2.0)

Step 1: Alice initiates selfCloseExitTo

* Alice calls selfCloseExitTo() to exit with her 50 CPT tokens (50% of entire pool)
* She sets \_redeemToCollateral = true (wants USDC from agent vault)

Step 2: Protocol calculates FAsset redemption requirement

* Alice's exit would remove 250 NAT ($500) from pool
* New pool collateral after Alice's exit: 250 NAT ($500)
* To maintain the same pool CR of 1.11, new FAsset backing should be: $500 / 1.11 = $450
* Required FAsset redemption: $900 - $450 = $450 worth of FAssets
* offset by FAsset fees earned by Alice: $450 - $30 = $420
* Converting to USDC needed: $420 worth of FAssets = 420 USDC required from agent vault

Step 3: Alice receives less than required

* Alice requires: $420 from agent vault
* Agent vault contains: Only 400 USDC
* **Shortfall: $420 - $400 = $20 USD deficit**
