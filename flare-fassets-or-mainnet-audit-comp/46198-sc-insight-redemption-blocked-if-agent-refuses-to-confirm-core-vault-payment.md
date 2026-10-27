# #46198 \[SC-Insight] Redemption Blocked if Agent Refuses to Confirm Core Vault Payment

**Submitted on May 26th 2025 at 10:21:32 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46198
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/CoreVaultFacet.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Description

The redemption flow relies on the agent to confirm inbound payments to the Core Vault via the `confirmReturnFromCoreVault()` function. Until this confirmation is made, redemption tickets are not issued, and collateral remains reserved.

This creates a critical dependency on agent cooperation for users to redeem their F-assets. If an agent is offline, unresponsive, or deliberately avoids confirming, users are indefinitely blocked from redeeming—even after fully paying into the Core Vault.

This is an attack vector where an agent can block redemptions without any penalty or loss. If the underlying asset becomes more valuable or scarce, the agent has an incentive to delay or avoid releasing it.

Since the confirmation is gated strictly behind an `onlyAgentVaultOwner` modifier, no other party can finalize the process.

```solidity
// In CoreVaultFacet.sol
function confirmReturnFromCoreVault(
    IPayment.Proof calldata _payment,
    address _agentVault
)
    external
    nonReentrant
   @>>  onlyAgentVaultOwner(_agentVault)
{
    Agent.State storage agent = Agent.get(_agentVault);
    CoreVault.confirmReturnFromCoreVault(_payment, agent);
}
```

## Impact Details

If the agent refuses to confirm a Core Vault payment, user redemptions are blocked indefinitely. This creates a single point of failure.

## Recommendations

It's better to Leverage existing payment proof structures to permit anyone to confirm valid payments after a timeout.

## Proof of Concept

## Proof of Concept

Consider this scenario where by:

1. Malicious Agent make request for `requestReturnFromCoreVault()`
2. Agent's request was granted by the CoreVault.
3. Agent does not call `confirmReturnFromCoreVault()`.
4. Redemption ticket is not issued; user cannot redeem.
5. Funds remain in Underlying Address but no redemption tickets are generated.
6. User X wants to redeem, `10,000`UBA amounts, but cannot due to insufficient redemption tickets

The agent faces no repercussions and may even benefit from delay
