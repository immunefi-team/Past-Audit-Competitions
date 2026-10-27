# #45550 \[SC-Medium] \[H-01] \`illegalPaymentChallenge\` is vulnerable to frontrunning by external challengers stealing the reward

**Submitted on May 16th 2025 at 16:29:31 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45550
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/ChallengesFacet.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Description

The `illegalPaymentChallenge` function in the Flare system allows any whitelisted user to submit proof of an agent's illegal payment in order to trigger liquidation and receive a reward.

However, the current implementation does not bind the provided proof to the sender's address (`msg.sender`). Since the proof is passed entirely via calldata, it is fully visible in the public mempool.

This allows malicious actors, including bots or block proposers, to observe a pending transaction containing a valid proof, copy the calldata, and submit their own transaction with the same proof.

Due to the implementation of `illegalPaymentChallenge`, the protocol only checks that:

1. The proof is valid and properly attested.
2. The payment originates from the agent’s address.
3. The proof has not yet been confirmed.

There is no verification step that the proof was submitted by the originator or discoverer. This means that the reward will be assigned to whichever transaction is executed first, regardless of who originally discovered the proof.

## Impact Details

This vulnerability exposes users to loss of challenge rewards despite performing honest work in discovering violations, as the rewards can be stolen by third-party actors monitoring the mempool.

Furthermore, this can reduce the incentive to act as a challenger, since any party with high MEV infrastructure (like bots or validators) will have an unfair advantage in claiming rewards without performing any honest work.

## References

```solidity
    function illegalPaymentChallenge(
        IBalanceDecreasingTransaction.Proof calldata _payment,
        address _agentVault
    )
        internal
    {
        AssetManagerState.State storage state = AssetManagerState.get();
        Agent.State storage agent = Agent.get(_agentVault);
        // if the agent is already being fully liquidated, no need for more challenges
        // this also prevents double challenges
        require(agent.status != Agent.Status.FULL_LIQUIDATION, "chlg: already liquidating");
        // verify transaction
        TransactionAttestation.verifyBalanceDecreasingTransaction(_payment);
        // check the payment originates from agent's address
        require(_payment.data.responseBody.sourceAddressHash == agent.underlyingAddressHash,
            "chlg: not agent's address");
        // check that proof of this tx wasn't used before - otherwise we could
        // trigger liquidation for already proved redemption payments
        require(!state.paymentConfirmations.transactionConfirmed(_payment), "chlg: transaction confirmed");
        // check that payment reference is invalid (paymentReference == 0 is always invalid payment)
        bytes32 paymentReference = _payment.data.responseBody.standardPaymentReference;
        if (paymentReference != 0) {
            if (PaymentReference.isValid(paymentReference, PaymentReference.REDEMPTION)) {
                uint256 redemptionId = PaymentReference.decodeId(paymentReference);
                Redemption.Request storage redemption = state.redemptionRequests[redemptionId];
                // Redemption must be for the correct agent and
                // only statuses ACTIVE and DEFAULTED mean that redemption is still missing a payment proof.
                // We do not check for timestamp of the payment, because on UTXO chains legal payments can be
                // delayed by arbitrary time due to high fees and cannot be canceled, which could lead to
                // unnecessary full liquidations.
                bool redemptionActive = redemption.agentVault == _agentVault
                    && (redemption.status == Redemption.Status.ACTIVE ||
                        redemption.status == Redemption.Status.DEFAULTED);
                require(!redemptionActive, "matching redemption active");
            }
            if (PaymentReference.isValid(paymentReference, PaymentReference.ANNOUNCED_WITHDRAWAL)) {
                uint256 announcementId = PaymentReference.decodeId(paymentReference);
                // valid announced withdrawal cannot have announcementId == 0 and must match the agent's announced id
                // but PaymentReference.isValid already checks that id in the reference != 0, so no extra check needed
                require(announcementId != agent.announcedUnderlyingWithdrawalId, "matching ongoing announced pmt");
            }
        }
        // start liquidation and reward challengers
        _liquidateAndRewardChallenger(agent, msg.sender, agent.mintedAMG);
        // emit events
        emit IAssetManagerEvents.IllegalPaymentConfirmed(_agentVault, _payment.data.requestBody.transactionId);
    }
```

```solidity
    function illegalPaymentChallenge(
        IBalanceDecreasingTransaction.Proof calldata _transaction,
        address _agentVault
    )
        external
        onlyWhitelistedSender
        nonReentrant
    {
        @>> Challenges.illegalPaymentChallenge(_transaction, _agentVault);
    }

```

## Recommendations

To mitigate this issue, implement one of the following: use a `Commit-reveal scheme` that is Require challengers to first submit a hash commitment of the proof, and only allow the challenger to reveal it after a delay.

## Proof of Concept

## Proof of Concept

An attacker can execute the following steps:

1. Monitor the public mempool for any transaction calling `illegalPaymentChallenge`.
2. Extract the `_payment` proof data from the calldata.
3. Submit the exact same `illegalPaymentChallenge` call, using the extracted `_payment` and `_agentVault`.

Since the contract does not bind the proof to the original discoverer, the attacker will receive the full reward if their transaction is mined first.
