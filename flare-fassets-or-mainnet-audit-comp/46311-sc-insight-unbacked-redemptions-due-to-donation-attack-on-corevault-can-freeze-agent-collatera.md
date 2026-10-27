# #46311 \[SC-Insight] Unbacked Redemptions Due to Donation- Attack on CoreVault Can Freeze Agent Collateral

**Submitted on May 28th 2025 at 09:08:15 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46311
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/userInterfaces/ICoreVaultManager.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The Flare FAssets system allows redemptions based on the `CoreVault.availableFunds` without verifying that those funds correspond to actual minted FAssets. This flawed accounting enables an attacker to lock agent collateral by donating to the vault without minting, creating an huge imbalance between minted supply and redeemable funds.

## Vulnerability Details

The FAssets protocol supports direct payments into the `CoreVault` via a `confirmPayment()` function that increases `availableFunds`:

```solidity
if (!confirmedPayments[_proof.data.requestBody.transactionId]) {
    uint128 receivedAmount = uint128(uint256(_proof.data.responseBody.receivedAmount));
    confirmedPayments[_proof.data.requestBody.transactionId] = true;
    availableFunds += receivedAmount;
    emit PaymentConfirmed(...);
}
```

However, this increase is not tied to any actual minting of FAssets. As a result, an attacker can donate funds (e.g., XRP) to the `CoreVault`, thereby increasing the apparent availability for redemption without creating any obligation (i.e., minted tokens).

Subsequently, the system allows the creation of redemption tickets using this phantom liquidity, treating donated funds as if they back minted FAssets. This will trigger the freezing of agent collateral, even though no real user has minted or holds FAssets to redeem.

The collateral tied to these redemptions is effectively held hostage until the ticket resolves, but since the redemption was never backed by minted assets, it remains in an unresolved state.

```solidity
Total minted FAssets < Cumulative redemption receipts Amount
```

This issue is rooted in the following logic from `requestReturnFromCoreVault()`:

```solidity
function requestReturnFromCoreVault(
    address _agentVault,
    uint64 _lots
) external notEmergencyPaused nonReentrant onlyAgentVaultOwner(_agentVault) {
    Agent.State storage agent = Agent.get(_agentVault);
    CoreVault.requestReturnFromCoreVault(agent, _lots);
}
```

This call enables agents to request returns limited by `availableFunds`, which includes attacker donations and not just genuine FAsset-backed deposits.

## Impact Details

This attack although a loss to the attacker can causes an indirect freezing of agents collateral since no real mint occurred. It also causes long-term capital inefficiency by tying collateral to ghost redemptions. agent Loses opportunity cost, as funds are locked backing nothing.

## Recommendation

The current recommendation against this donation attack is to prevent direct injection of funds into the core vault or better ensure that the cumulative amount of redemption tickets generated is limited by the total supply of F-asset in circulation, to prevent any capital inefficiencies in the system.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/CoreVault.sol#L125-L157>

## Proof of Concept

## Proof of Concept

consider this attack path:

1. Attacker donates assets to `CoreVault`, confirmed via `confirmPayment()` — increasing `availableFunds`.
2. No minting function is triggered — so no `FAssets` are actually issued.
3. System incorrectly allows redemption tickets to be created using the direct `availableFunds`.
4. These redemptions freeze agent collateral — despite there being nothing to redeem.

Collateral is indefinitely tied up in an unresolvable state, harming agent liquidity and usability.

For numerical analysis: To back the redemption, the protocol freezes the agent’s collateral,

Agent has not minted anything, yet (No F-asset in circulation ) Collateral backing = 1,000 USD Collateral frozen: 1,000 USD (backing a fake redemption)
