# #45961 \[SC-Insight] \`selfMint()\` Can Lead to Permanent Loss of Agents' Funds During Emergency Pause

**Submitted on May 22nd 2025 at 23:35:41 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45961
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/MintingFacet.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Description

The `selfMint()` function in the Flare Assets protocol enables agents to mint assets for themselves after submitting an underlying payment. However, the function is guarded by the `notEmergencyPaused` modifier, which disables minting when an emergency pause is active.

This becomes problematic because the proof of payment used in `selfMint()` is only valid for 24 hours. If the system is paused during this time, users are blocked from calling `selfMint()`, and once the proof expires, the user has no way to recover or utilize the sent funds, resulting in a total loss of funds.

```solidity
function selfMint(
        IPayment.Proof calldata _payment,
        address _agentVault,
        uint256 _lots
    )
        external
        onlyAttached
       @>>  notEmergencyPaused    {
        Minting.selfMint(_payment, _agentVault, _lots.toUint64());
    
}
```

unlike other functionalities that involves submitting valid proof the `notEmergencyPaused` is safely removed to prevent this possibility

```solidity
   function executeMinting(
        IPayment.Proof calldata _payment, 
        uint256 _collateralReservationId
    )
        external
        nonReentrant
    {
        Minting.executeMinting(_payment, _collateralReservationId.toUint64());
    }
```

## Impact Details

Consider a situation when user sends the required underlying payment and the system is now paused before they call `selfMint()`, the function becomes completely inaccessible due to the `notEmergencyPaused` modifier. Since payment proofs expire after 24 hours, any emergency pause exceeding this window results in a total, irrecoverable loss of user funds, even though the payment was valid.

Furthermore what makes this funds irrecoverable is actually due to the fact that if the agent withdraws this from the underlying address they can get challenged and enter `fullLiquidation`

```solidity
function paymentsMakeFreeBalanceNegative(
        IBalanceDecreasingTransaction.Proof[] calldata _payments,
        address _agentVault
    )
        internal
    {
        
        for (uint256 i = 0; i < _payments.length; i++) {
            IBalanceDecreasingTransaction.Proof calldata pmi = _payments[i];
            TransactionAttestation.verifyBalanceDecreasingTransaction(pmi);
            // check there are no duplicate transactions
            for (uint256 j = 0; j < i; j++) {
                require(_payments[j].data.requestBody.transactionId != pmi.data.requestBody.transactionId,
                    "mult chlg: repeated transaction");
            }
            require(pmi.data.responseBody.sourceAddressHash == agent.underlyingAddressHash,
                "mult chlg: not agent's address");
            if (state.paymentConfirmations.transactionConfirmed(pmi)) {
                continue;   // ignore payments that have already been confirmed
            }
            bytes32 paymentReference = pmi.data.responseBody.standardPaymentReference;
            if (PaymentReference.isValid(paymentReference, PaymentReference.REDEMPTION)) {
                // for redemption, we don't count the value that should be paid to free balance deduction
                uint256 redemptionId = PaymentReference.decodeId(pmi.data.responseBody.standardPaymentReference);
                Redemption.Request storage request = state.redemptionRequests[redemptionId];
                total += pmi.data.responseBody.spentAmount - SafeCast.toInt256(request.underlyingValueUBA);
            } else {
                // for other payment types (announced withdrawal), everything is paid from free balance
                total += pmi.data.responseBody.spentAmount;
            }
        }

      
        // check that total spent free balance is more than actual free underlying balance
     @>>   int256 balanceAfterPayments = agent.underlyingBalanceUBA - total;
        uint256 requiredBalance = UnderlyingBalance.requiredUnderlyingUBA(agent);
        require(balanceAfterPayments < requiredBalance.toInt256(), "mult chlg: enough balance");
        // start liquidation and reward challengers
        _liquidateAndRewardChallenger(agent, msg.sender, agent.mintedAMG);
        // emit events
        emit IAssetManagerEvents.UnderlyingBalanceTooLow(_agentVault, balanceAfterPayments, requiredBalance);
    }
```

## Recommendations

To prevent against this we can promptly remove the modifier on `selfMint()`:

```solidity
function selfMint(
        IPayment.Proof calldata _payment,
        address _agentVault,
        uint256 _lots
    )
        external
        onlyAttached
         {
        Minting.selfMint(_payment, _agentVault, _lots.toUint64());
    
}
```

## Proof of Concept

## Proof of Concept

Let's consider this Scenario where the agents' funds can be locked Scenario:

1. A user makes an underlying payment of `10,000` tokens intending to mint.
2. They plan to immediately call `selfMint()` but find the system is emergency paused.
3. The pause lasts let's say for `36` hours.
4. Once the pause is lifted, the user cannot make a call for `selfMint()` due to an expired proof.

The funds are now permanently locked in the agents' EOA account
