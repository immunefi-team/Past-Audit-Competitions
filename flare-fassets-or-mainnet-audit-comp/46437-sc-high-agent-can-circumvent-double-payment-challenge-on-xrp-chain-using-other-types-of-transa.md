# #46437 \[SC-High] Agent can circumvent double payment challenge on XRP chain using other types of transaction

**Submitted on May 30th 2025 at 12:48:34 UTC by @nnez for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46437
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Challenges.sol>
* **Impacts:**
  * Protocol insolvency
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Vulnerability Details

In previous submission, specifically: #41764, I proposed an attack scenario where a malicious agent can circumvent the challenge system and perform multiple payments on the same redemption.

The attack hinged on two facts:

1. A rejected redemption cannot be confirmed via `confirmRedemptionPayment` which ultimately leads to undeletable redemption request.
2. A proof of balanceDecreasing transaction older than configured `VERIFICATION_CLEANUP_DAYS` cannot be used to challenge malicious payment/action.

The attack works because for all of challenge method:

1. `illegalPaymentChallenge` cannot challenge transaction with matching redemption\
   See: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Challenges.sol#L52-L55>
2. `doublePaymentChallenge` requires two transactions, however, if one transaction is too old then it will revert when trying to call `verifyBalanceDecreasingTransaction`\
   See: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Challenges.sol#L82-L83>

```solidity
    function verifyBalanceDecreasingTransaction(
        IBalanceDecreasingTransaction.Proof calldata _proof
    )
        internal view
    {
        AssetManagerSettings.Data storage _settings = Globals.getSettings();
        IFdcVerification fdcVerification = IFdcVerification(_settings.fdcVerification);
        require(_proof.data.sourceId == _settings.chainId, "invalid chain");
        require(fdcVerification.verifyBalanceDecreasingTransaction(_proof), "transaction not proved");
        require(_confirmationCannotBeCleanedUp(_proof.data.responseBody.blockTimestamp),
            "verified transaction too old");
    }
    function _confirmationCannotBeCleanedUp(uint256 timestamp) private view returns (bool) {
        return timestamp >= block.timestamp - PaymentConfirmations.VERIFICATION_CLEANUP_DAYS * 1 days;
    }
```

3. `paymentsMakeFreeBalanceNegative` requires a summation of all transactions to make underlying balance goes under, however, it doesn't count the amount being used in redemption.\
   See: <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Challenges.sol#L133>

In conclusion, if we can somehow sustain a redemption request (no one is able to confirm nor default), and wait for enough time for the previous transaction to become too old, then we can perform the attack and repeat and avoid being challenged.

The fix is introduced in a current version by allows for a redemption confirmation even though it has been rejected given that there is a payment on that redemptionId. The fix relies on the fact that there is an incentive for others to confirm a redemption after some hours has passed and agent fails to do so.

However, the fix only works on utxo-based chain because there is only one type of transaction that can be considered a payment.\
In contrast to that, XRP has many types of transaction that could be used to transfer the underlying asset out of agent's address, e.g. OfferCreate, EscrowCreate.

See: <https://livenet.xrpl.org/transactions/63DCCAF6FA76F88B216CAB1E04E15A6268E7F4C5808316602D2401ADB4883BEA/detailed>

In the above transaction, we can observe that an EscrowCreate transaction deducted `Balance` in `AccountRoot` of the sender and create an escrow for the destination account.

Unfortunately, in order to allow only a transaction of **Payment** type to be used as a mean of payment in the fAsset system, the payment summary will reject other types of payment as invalid.

See: <https://github.com/flare-foundation/multi-chain-client/blob/main/src/base-objects/transactions/XrpTransaction.ts#L436>

That means, even after the time has passed, others can't confirm redemption payment for agent due to the lack of proof.

Although, it is still possible to request for a proof of balance decreasing transaction for other types of transaction, the attack still works since attacker can sustain the redemption request and can utilize the "too old transaction" trick.

Consider the following attack scenario:

1. An agent makes a request for redemption against their own ticket for 100 XRP, suppose that the result redemptionId is 5.
2. An agent perform a transfer of 100 XRP using `EscrowCreate` transaction with payment reference to id=5 instead of `Payment`.
3. No one can confirm this payment because it is not of type `Payment` and no one can invoke a default path since only redeemer and agent can do it.
4. An agent waits for 14 days, performs a payment of 100 XRP with same payment reference to id=5, this should be considered a double payment.
5. Suppose someone detects this wrongdoing, and tries to challenge double payment on this agent with proof of decreasing balance of both transactions.
6. However, due to `_confirmationCannotBeCleanedUp`, the first transaction will cause a revert because it is too old.
7. This can't also be challenged using `illegalPaymentChallenge` and `paymentsMakeFreeBalanceNegative` because both transactions have a corresponding redemptionId=5
8. An agent can keep repeating from step 4. again an again
9. Once an agent is done, they wait another 14 days for all transactions to become too old, then call `finishRedemptionWithoutPayment`.

Side note: they (agent) can also make a redemption request to invalid underlying address so that in Step 9. they can just call `rejectInvalidRedemption` to end the redeeming request while retaining the underlying balance.

## Impact

This vulnerability allows a malicious agent to bypass double-payment protections and transfer underlying assets out of their address using non-payment transaction types (e.g., EscrowCreate). These transfers cannot be confirmed or challenged once they become too old, enabling the agent to reuse underlying assets and artificially increase their tracked underlying balance.

While earlier system versions limited the impact due to collateral requirements (2x), the introduction of the Core Vault amplifies the risk. The agent can now exploit this artificial balance to mint fAssets and force all redeemable value to go into core vault:

Consider the continuing scenario from above:

1. Agent re-use the underlying assets to top-up their tracked underlying balance.
2. Agent invokes `mintFromFreeUnderlying` (given that they have some free collateral lots) to mint fAssets from artifical underlying balance
3. Agent uses newly minted fAssets to redeem underlying assets from other agents in the system
4. Agent then top-up their underlying balance with the real just-redeemed underlying assets
5. Agent invokes `transferToCoreVault` to redeem against core vault and transfers underlying assets to core vault
6. Agent repeats until almost all underlying assets are transferred to core vault

This results in an issue: a majority of circulating fAssets become redeemable only through the Core Vault, which restricts access to whitelisted addresses. Regular users are effectively blocked from redeeming their fAssets for underlying tokens, undermining the system’s redemption guarantees.

**Initial state**

| Agent         | fXRP | XRP (underlying) | tracked balance |
| ------------- | ---- | ---------------- | --------------- |
| A (malicious) | 100  | 100              | 100             |
| B (honest)    | 100  | 100              | 100             |
| Core Vault    | -    | 0                | -               |

**Agent A artificially increase balance and invokes `mintFromFreeUnderlying`**

| Agent         | fXRP | XRP (underlying) | tracked balance |
| ------------- | ---- | ---------------- | --------------- |
| A (malicious) | 200  | 100              | 100+100=200     |
| B (honest)    | 100  | 100              | 100             |
| Core Vault    | -    | 0                | -               |

**Agent A redeems underlying asset from Agent B and top-up their tracked balance with just-redeemed underlying asset**

| Agent         | fXRP | XRP (underlying) | tracked balance |
| ------------- | ---- | ---------------- | --------------- |
| A (malicious) | 200  | 100+100=200      | 200+100=300     |
| B (honest)    | 0    | 0                | 0               |
| Core Vault    | -    | 0                | -               |

**Agent A invokes `transferToCoreVault` to redeem against core vault for 170 fXRP**

| Agent         | fXRP | XRP (underlying) | tracked balance |
| ------------- | ---- | ---------------- | --------------- |
| A (malicious) | 200  | 200-170=30       | 300-170=130     |
| B (honest)    | 0    | 0                | 0               |
| Core Vault    | -    | 170              | -               |

**Final state**

* Total circulating fXRP: 200
* Publicly redeemable via agents: only 30 XRP
* Locked in Core Vault (whitelist-only access): 170 XRP

This effectively blocks ordinary users from redeeming most fXRP in circulation, enabling a temporary denial-of-redemption and undermining fAsset liquidity.

## Rationale for severity

This issue has the potential to disrupt the redemption process and undermine protocol solvency to redeem fAsset back to underlying asset. While it does not directly lead to immediate insolvency, it enables a malicious agent to lock a majority of the underlying assets in the Core Vault, where only whitelisted addresses can redeem, effectively denying normal users access to redemption.

However, this denial is only temporary, other agents may choose to call a return of underlying assets from the Core Vault to meet the redemption demand. Therefore, the primary impact aligns with griefing and denial-of-service against fAsset holders.

Given the potential to temporarily block core protocol functionality, cause partial value inaccessibility, and degrade user experience without immediate insolvency, this issue might be best categorized as Medium severity.

That said, the impact could also escalate toward High severity under certain conditions, particularly during periods of high redemption demand. In such cases, liquidity constraints could lead to forced selling of fAssets on secondary markets (e.g., DEXes), resulting in price slippage and user losses, which would amplify the systemic risk. The true severity depends on how risk-tolerant the protocol is to temporary illiquidity.

* Medium - griefing attack
* High - protocol insolvency but only limited to malicious agent

I'll leave the final judgment of severity to the protocol team.

## Proof of Concept

## Proof-of-Concept

### I

If we take a look at a multi-chain-client test on XRP transaction, we can observe that a payment summary for transaction of other types would yield `PaymentSummaryStatus.NotNativePayment`.\
See: <https://github.com/flare-foundation/multi-chain-client/blob/main/test/XRP/transaction.test.ts#L101-L104>

### II

The following test demonstrates that in the current version with the fix, the attack is still exploitable given that a redemption request can be sustained.\
**Steps**

1. Put the following test in `test/integration/fasset-simulation/AttackScenarios.ts`

```typescript
    it("nnez - bypass 41764 in xrp chain, agent circumvent challenge on illegal payment", async () => {
        // Prelim setup
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(10000));
        const redeemer = await Redeemer.create(context, redeemerAddress1, underlyingRedeemer1);
        const challenger = await Challenger.create(context, challengerAddress1);

        // Make agent available and deposit some collateral
        const fullAgentCollateral = toWei(3e8);
        await agent.depositCollateralsAndMakeAvailable(fullAgentCollateral, fullAgentCollateral);
        // update block, passing agent creation block
        await context.updateUnderlyingBlock();

        // Perform minting
        const lots = 3;
        const crt = await minter.reserveCollateral(agent.vaultAddress, lots);
        const txHash = await minter.performMintingPayment(crt);
        const minted = await minter.executeMinting(crt, txHash);
        
        // Note that there is no longer need for non-zero handshake

        // Make a redemption request to agent's owned address
        await context.fAsset.transfer(redeemer.address, minted.mintedAmountUBA, { from: minter.address });
        const [redemptionRequests,,,] = await redeemer.requestRedemption(lots);
        const request = redemptionRequests[0];

        // Note that there is no need to reject the redemption  

        // Perform the first payment, 
        // Because there is no other type of transaction in the test suite, let's assume that this is transaction of type EscrowCreate  
        // So, it cannot be proved via paymentSummary  
        const paymentAmount = request.valueUBA.sub(request.feeUBA);
        const tx1Hash = await agent.performPayment(request.paymentAddress, paymentAmount, request.paymentReference);

        await deterministicTimeIncrease((await context.assetManager.getSettings()).confirmationByOthersAfterSeconds);
        
        // Since we are assumeing that the proof cannot be produced, no one can confirm  
        // await agent.confirmActiveRedemptionPayment(request, tx1Hash);

        // Agent waits for 14 days
        await deterministicTimeIncrease(15 * DAYS + 10);
        mockChain.skipTime(14 * DAYS + 10);
        mockChain.mine(100*14);

        // perform double payment to the same payment reference
        const tx2Hash = await agent.performPayment(request.paymentAddress, paymentAmount, request.paymentReference);

        // cannot challenge the old transaction
        await expectRevert(challenger.doublePaymentChallenge(agent, tx1Hash, tx2Hash), 'verified transaction too old');
        await expectRevert(challenger.illegalPaymentChallenge(agent, tx1Hash), 'verified transaction too old');
        // cannot challenge the new transaction too  
        await expectRevert(challenger.illegalPaymentChallenge(agent, tx2Hash), 'matching redemption active');
    });
```

2. Run `yarn hardhat test "test/integration/fasset-simulation/AttackScenarios.ts" --grep "nnez - bypass 41764 in xrp chain, agent circumvent challenge on illegal payment"`
3. Observe that the test passes, an indication that the attack still works on the current version given that an attacker can sustain a redemption request.
