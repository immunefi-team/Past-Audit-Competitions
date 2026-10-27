# #47053 \[SC-Low] \`transferToCoreVault()\` allows agents to have unbacked synthetic assets by extracting underlying value without burning

**Submitted on Jun 8th 2025 at 15:39:58 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47053
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/CoreVaultFacet.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Description

The `transferToCoreVault` mechanism in the `FAssets` protocol is intended to move value from an agent to the core vault in exchange for redeeming f-assets. However, due to improper enforcement of payment delivery and lack of corrective logic upon redemption default, this function can be abused by an agent to extract capital from the system with little or no fee.

## Details

When agent initiates the `transferToCoreVault` of huge underlying balance the agent can use the the generated payment reference to transfer the value to another underlying address without any issue and use it to generate yield for a while.

After sometime the `transferToCoreVault`(redemption) is marked as defaulted by the system bit that is after the `lastUnderlyingBlock` has elapsed, now during this process the `redemptionTickets` are created back but the agent collateral is not touched since this is a `transferToCoreVault` .

So now the system has been brought back to a state of `freeBalanceNegative` but the main thing is we can't challenge/liquidate the malicious agent since the agent is made the withdrawal through the `redemptionRequestId`.

due to this effect:

```solidity
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
```

Now after sometime and making profit from the free capital the agent calls `selFMint(lots=0)` to increase underlying balance, before now confirming the redemption using the payment reference of the first withdrawal, safely avoiding full-liquidation.

This breaks the core invariant of the protocol — that every minted f-asset must be fully backed by an equivalent amount of underlying value and also the agent can keep repeating this process anytime.

## Impact Details

The impact is that agent are allowed to withdraw value from the underlying chain through the pretense of transferring to `CoreVault`, agents can use this attack path to game and get free capital from the system.

This attack is possible because we only allow confirmation of the redemption payment by agent instantaneously, hence creating this opportunity of free riding of the underlying value.

From docs:

```
// If the agent doesn't confirm payment in enough time (several hours, setting confirmationByOthersAfterSeconds),
 //     anybody can do it and get rewarded from agent's vault.
```

## Recommendation

I think it's better to allow for the confirmation of redemption by anyone immediately payments are detected on flare-network, but only pay the actor from the agent's vault after necessary time has gone and the agent refuses to confirm the payment.

## Proof of Concept

## Proof of Concept

consider this Attack path:

1. Agent initiates a large `transferToCoreVault()` redemption, generating a payment reference(`_redemptionRequestId`) for the redemption process
2. Agent references this `_redemptionRequestId` but make the actual payment to a personal underlying address and never actually pays the core vault.
3. Agent uses this value for arbitrage and some yield generating system
4. After a while the redemption is defaulted and agent redemption tickets are recreated, now the agent has a lot of time to hold on to the funds, since their collateral are not touched.
5. Agent delays confirmation intentionally.
6. Before the confirmation window expires, and agent has gotten some profit, agent now `selfMint` with `lots=0` to increase the underlying balance.
7. Finally the agent confirms the redemption.

The malicious agent has now been able to withdraw a lot of value beyond the limit from the underlying for a long while safely avoiding the challenge procedure, also this real value (used off-chain/on-chain or for arbitrage) while still appearing to back 100% of their issued f-assets.
