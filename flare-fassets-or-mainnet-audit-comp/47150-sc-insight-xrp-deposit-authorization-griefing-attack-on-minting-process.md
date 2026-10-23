# #47150 \[SC-Insight] XRP Deposit Authorization Griefing Attack on Minting Process

**Submitted on Jun 9th 2025 at 08:37:00 UTC by @Bluedragon for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47150
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/MintingFacet.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary:

Agents can enable Deposit Authorization on their XRP accounts to block all incoming payments from minters, causing minting payment defaults and allowing agents to collect reservation fees without providing any service.

## Vulnerability Details:

The attack exploits the XRP Ledger's Deposit Authorization feature combined with the fAssets minting process:

1. **Normal Minting Flow**: Minters reserve collateral and pay fees, then must send underlying payments to agent's XRP address to complete minting.
2. **Deposit Authorization Blocking**: When agents enable Deposit Authorization on their XRP accounts, all direct payments are rejected with `tecNO_PERMISSION` error.
3. **Payment Default Exploitation**: As the minters' underlying payments fail due to Deposit Authorization, after the `lastUnderlyingPaymentTimestamp` expires, agents can call `mintingPaymentDefault()` to collect the reservation fees without providing any backing or service.

## Impact

* **Systematic Minter Exploitation**: Minters lose reservation fees for failed mintings they cannot complete
* **Agent Fee Farming**: Agents can continuously collect fees without backing any assets
* **System Reliability**: Undermines trust in the minting process for XRP fAssets
* **Economic Attack**: Low-cost, high-reward attack vector for malicious agents

## References:

* Deposit Authorization docs (<https://xrpl.org/docs/concepts/accounts/depositauth>)

## Recommended Mitigation:

Implement a payment status check in the execute minting process to check that if the payment is failed with error code 2 which is `RECEIVER_FAILURE`, if thats the case allow the minter to get back the reservation fee.

```solidity
// Add validation in execute minting 
if(
    _payment.data.responseBody.status == 2 // RECEIVER_FAILURE
) {
  uint256 totalFee = crt.reservationFeeNatWei + unclaimedExecutorFee;
  // Refund the reservation fee to the minter  
  traTransfers.transferNAT(crt.executor, unclaimedExecutorFee);
}
```

## Proof of Concept

## Proof of Concept (Scenario step by step):

1. Malicious agent creates agent vault and enters available agents list
2. Agent enables Deposit Authorization on their XRP underlying address
3. Minter calls `reserveCollateral()` and pays reservation fee
4. System emits `CollateralReserved` event with agent's XRP address for payment
5. Minter attempts to send underlying payment to agent's XRP address
6. XRP Ledger rejects payment with `tecNO_PERMISSION` due to Deposit Authorization
7. Minter cannot complete `executeMinting()` within time window
8. Agent calls minting payment default and collects reservation fee
9. Agent repeats process with multiple minters
