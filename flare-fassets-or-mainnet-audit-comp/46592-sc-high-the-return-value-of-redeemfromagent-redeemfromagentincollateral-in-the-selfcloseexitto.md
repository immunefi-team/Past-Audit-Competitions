# #46592 \[SC-High] The return value of redeemFromAgent/redeemFromAgentInCollateral in the selfCloseExitTo is not checked

**Submitted on Jun 2nd 2025 at 05:53:32 UTC by @ox9527 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46592
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

In CollateralPool.sol::selfCloseExitTo(), the user spends an amount of requiredFAssets to redeem collateral from the agent. However, the actual value redeemed may vary, and the return value is not validated. This could potentially lead to a loss of funds for the user.

## Vulnerability Details

From the code CollateralPool.sol->selfCloseExitTo->\_selfCloseExitTo()->assetManager.redeemFromAgent()

```solidity
        // redeem f-assets if necessary
        if (requiredFAssets > 0) {
            if (requiredFAssets < assetManager.lotSize() || _redeemToCollateral) {
                assetManager.redeemFromAgentInCollateral(
                    agentVault, _recipient, requiredFAssets);
            } else {
                // automatically pass `msg.value` to `redeemFromAgent` for the executor fee
                assetManager.redeemFromAgent{ value: msg.value }( //@audit the return value is not checked.
                    agentVault, _recipient, requiredFAssets, _redeemerUnderlyingAddress, _executor);  <@
            }
        }
```

And then RedemptionRequests.sol->Redemptions.closeTickets()

```solidity
        AssetManagerState.State storage state = AssetManagerState.get();
        // redemption tickets
        uint256 maxRedeemedTickets = Globals.getSettings().maxRedeemedTickets;
        uint64 lotSize = Globals.getSettings().lotSizeAMG;
        for (uint256 i = 0; i < maxRedeemedTickets && _closedAMG < _amountAMG; i++) {
            // each loop, firstTicketId will change since we delete the first ticket
            uint64 ticketId = state.redemptionQueue.agents[_agent.vaultAddress()].firstTicketId;
            if (ticketId == 0) {
                break;  // no more tickets for this agent
            }
            RedemptionQueue.Ticket storage ticket = state.redemptionQueue.getTicket(ticketId);
            uint64 maxTicketRedeemAMG = ticket.valueAMG + _agent.dustAMG;
            maxTicketRedeemAMG -= maxTicketRedeemAMG % lotSize; // round down to whole lots
            uint64 ticketRedeemAMG = SafeMath64.min64(_amountAMG - _closedAMG, maxTicketRedeemAMG);  <@ 
            // only remove from tickets and add to total, do everything else after the loop
            removeFromTicket(ticketId, ticketRedeemAMG);
            _closedAMG += ticketRedeemAMG;
        }
```

From the code above, we can see that the loop stops early in two cases: when `maxRedeemedTickets` is reached, or when there are no more tickets available for this agent.

Both of these cases can result in the actual redeemed amount being smaller than expected.

However, the final `closedUBA` obtained is not actually verified, which may result in the actual `closedUBA` being smaller than expected. As a result, the user may end up overpaying in `requiredFAssets`.

## Impact Details

User paid more FAssets then expected

## References

```solidity
        if (requiredFAssets > 0) {
            if (requiredFAssets < assetManager.lotSize() || _redeemToCollateral) {
                assetManager.redeemFromAgentInCollateral(
                    agentVault, _recipient, requiredFAssets);
            } else {
                // automatically pass `msg.value` to `redeemFromAgent` for the executor fee
                assetManager.redeemFromAgent{ value: msg.value }( //@audit the return value is not checked.
                    agentVault, _recipient, requiredFAssets, _redeemerUnderlyingAddress, _executor);
            }
        }
```

## Proof of Concept

## Proof of Concept

First add console.log to file:RedemptionRequests.sol::redeemFromAgent()

```solidity
        // burn the closed assets
        console.log("closedUBA:",closedUBA);
        Redemptions.burnFAssets(msg.sender, closedUBA);
    }
```

Track the actually closedUBA amount.

add test to Redemption.ts:

```ts
    it.only("mint and redeem from agent and redeem from agent in collateral branch test", async () => {
        const agentVault = await createAgent(agentOwner1, underlyingAgent1);
        await depositAndMakeAgentAvailable(agentVault, agentOwner1, toWei(3e10));
        // minter
        chain.mint(underlyingMinter1, toBNExp(10000000, 18));
        await updateUnderlyingBlock();
        const lots = 1;
        const agentInfo = await assetManager.getAgentInfo(agentVault.address);
        const crFee = await assetManager.collateralReservationFee(lots);
        const resAg = await assetManager.reserveCollateral(agentVault.address, lots, agentInfo.feeBIPS, ZERO_ADDRESS, [], { from: minterAddress1, value: crFee });
        const crt = requiredEventArgs(resAg, 'CollateralReserved');
        const paymentAmount = crt.valueUBA.add(crt.feeUBA);
        const txHash = await wallet.addTransaction(underlyingMinter1, crt.paymentAddress, paymentAmount, crt.paymentReference);
        const proof = await attestationProvider.provePayment(txHash, underlyingMinter1, crt.paymentAddress);
        const res = await assetManager.executeMinting(proof, crt.collateralReservationId, { from: minterAddress1 });
        const minted = requiredEventArgs(res, 'MintingExecuted');
        // redeemer "buys" f-assets
        await fAsset.transfer(redeemerAddress1, minted.mintedAmountUBA, { from: minterAddress1 });
        // redemption request
        collateralPool = await CollateralPool.at(await assetManager.getCollateralPool(agentVault.address));
        await impersonateContract(collateralPool.address, toBN(512526332000000000), accounts[0]);

        //Only collateral pool can redeem from agent
        const rs = await assetManager.redeemFromAgent(agentVault.address, redeemerAddress1, toBN(40000000000000000000), underlyingRedeemer1, ZERO_ADDRESS, { from: collateralPool.address });

    });
```

We use the amount is 40 ETH, and the output closedUBA is not this value:

```shell
npx hardhat test test/unit/fasset/library/Redemption.ts
...
closedUBA: 31200000000000000000
...
```
