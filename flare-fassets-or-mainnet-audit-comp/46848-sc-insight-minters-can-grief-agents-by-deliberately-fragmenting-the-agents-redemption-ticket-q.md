# #46848 \[SC-Insight] Minters can grief agents by deliberately fragmenting the agent's redemption ticket queue with minimal size tickets, preventing or delaying large transfers to core vault

**Submitted on Jun 5th 2025 at 09:03:37 UTC by @niroh for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46848
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CoreVaultManager.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Minters can deliberately split their minting to large batches of small mints of a specific agent, causing fragmentation of the agent's redemption ticket queue, which will limit the amount of underlying the agent can transfer to the vault in one transaction.

## Vulnerability Details

When an agent wants to transfer some amount of underlying to the CoreVault, the process can be described as:

1. Agent starts the transfer process by calling transferToCoreVault.
2. Redemptions::closeTickets is called to close agent redemption tickets in the amount of the transfer
3. A redemption request is created for the agent, with the CoreVault as recipient, for the amount of the transfer.
4. The agent pays the redemption request on the underlying chain
5. The agent calls RedemptionConfirmations::confirmRedemptionPayment which updates the coreVault balance and releases agent collateral (by calling endRedeemingAssets).

The problem is that closeTickets limits the amount closed to up to the amount in the next maxRedeemedTickets, as can be seen here:

```solidity
//Redemptions.sol line 15
function closeTickets(
    Agent.State storage _agent,
    uint64 _amountAMG,
    bool _immediatelyReleaseMinted,
    bool _closeWholeLotsOnly
)
    internal
    returns (uint64 _closedAMG, uint256 _closedUBA)
{
    AssetManagerState.State storage state = AssetManagerState.get();
    // redemption tickets
    uint256 maxRedeemedTickets = Globals.getSettings().maxRedeemedTickets;
    uint64 lotSize = Globals.getSettings().lotSizeAMG;
    for (uint256 i = 0; i < maxRedeemedTickets && _closedAMG < _amountAMG; i++) {
```

This enables minters to grief agents by deliberately minting their desired mint amount in multiple, minimal size minting requests instead of one large mint. This will cause the agent's redemption ticket queue to be extremely fragmented, preventing or delaying them from transferring large amounts of collateral to the core vault. Please see a detailed attack scenario in the POC section.

## Recomendation

1. maxRedeemedTickets is applied in redemptions, to avoid excessive gas consumption/costs caused by too many tickets being closed in the same redemption. However, in the case of an agent transferring funds to the vault, it may make sense to remove this limit, especially given that all burned tickets are consolidated to one RedemptionRequest. This will enable the agent avoid ticket fragmentation issues (weather deliberate or organic).

## Impact Details

1. Causing long delays or preventing agents from transfering large amounts of underlying to the Core Vault (with associated opportunity cost of forgone minting profits)
2. Some incease in the cost of transfering funds to CoreVault through the increase in transaction costs (though as mentioned, these are not likely to be very high with XRP)

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Redemptions.sol#L28>

## Proof of Concept

## Proof of Concept

### attack scenario

1. Agent A is activated, and deposits collateral that enables minting $2M of XRP
2. When the agent is first activated, the attacker/s start creating batches of multiple mints with the minimal possible amount (1 lot) on agent A.
3. Currently for the XRP songbird deployment, 1 lot is set to 10XRP (\~20USD), and maxRedeemedTickets is set to 20. The attacker can mint a batch of 20 tickets with a total value of $400.
4. Since there is no limit to the number of CollateralReservations a user can create simultaneously, all reservations can be made in one block, and all underlying payments and proofs can be done together, making the time it takes to mint the batch equal to the time it takes to mint the amount in one request.
5. In terms of costs, all associated fees (minting fee, CR fee) are defined as rations from the mint amount, meaning the cost of minting $400 at once or with 20 requests is the same. The only overhead is transaction costs which are currently miniscule on both XRP and Flare/Songbird.
6. The attacker/s creates sporadic mints of the same type (batches of 20 or more mints of the minimal amount) throughout the lifetime of the agent, (for example, one batch per day) thus "spamming" the agent redemption tickets queue with large chains of minimal tickets that are present between the honest user mints (that are likely to be of larger amounts)
7. After 3 months, the agent has a total mint of $1.5M and wants to offboard $1M$ of the associated underlying to the core vault, to clear collateral for more mints.
8. Because of the large number of minimum value ($20) tickets on Agent A's queue, The agent will be forced to make a very large number of transfers (calling transferToCoreVault, paying on the underlying chain and calling confirmRedemptionPayment) to fully transfer the amount.
9. Assuming that honest user mints are equally spread throughout the three months, the agent will need to burn through the first 2 months tickes. Given the attacker minted a batch per day, the agent will need atleast 60 transfer transactions to fully transfer the desired amount.
10. Note that since only one active agent transfer can happen at a given time, that agent can not parralelize the transfers. Each transfer requires a proof from the FDC (up to a few minutes on XRP, probably more on other chains)

### attack cost

1. In terms of fees paid, the attacker pays the same fees for split multiple mints as they would for one large mint.
2. In terms of transaction costs: each minting batch of 20 requires 40 Flare/Songbird txs (reserve/execute) and 20 XRP transactions. An average songbird tx fee is 1 cent, and XRP transfers are even less, making the overall tx cost of each batch less than a dollar.
