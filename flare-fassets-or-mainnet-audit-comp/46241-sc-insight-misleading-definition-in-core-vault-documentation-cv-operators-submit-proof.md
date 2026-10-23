# #46241 \[SC-Insight] Misleading definition in Core-Vault documentation (“CV operators submit proof”)

**Submitted on May 27th 2025 at 07:10:26 UTC by @Paludo0x for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46241
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/CoreVaultFacet.sol>
* **Impacts:**

## Description

The code restricts confirmReturnFromCoreVault to onlyAgentVaultOwner, yet the docs say Core-Vault operators “CV operators execute the transfer and submit proof of payment to the asset manager.” This is reported here: <https://dev.flare.network/fassets/core-vault#request-for-return-agents-only>

Operators only generate the FDC proof; the on-chain submission must be performed by the agent (vault owner). Recommend clarifying the docs accordingly.

## Proof of Concept

## Proof of Concept

The contract function can only be executed by the agent-vault owner (enforced by onlyAgentVaultOwner).

```
function confirmReturnFromCoreVault(IPayment.Proof calldata _payment, address _agentVault)
    external
    onlyAgentVaultOwner(_agentVault)
{ … }
```
