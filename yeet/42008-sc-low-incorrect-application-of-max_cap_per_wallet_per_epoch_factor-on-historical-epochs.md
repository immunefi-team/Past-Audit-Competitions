# #42008 \[SC-Low] Incorrect Application of MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR on Historical Epochs

**Submitted on Mar 19th 2025 at 23:53:44 UTC by @MarsKittyHacker for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42008
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The Reward contract calculates user claimable rewards across multiple epochs. During this calculation, it applies the current MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR() setting to all past epochs. This behavior creates an unintended dependency on the current cap value rather than the historical one. It is valid when `claimable` is more than `maxClaimable`.

## Vulnerability Details

The formula used in the contract is:

```solidity
uint256 maxClaimable = (epochRewards[epoch] / rewardsSettings.MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR());
```

The larger MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR is, the smaller the maxClaimable value becomes.

The vulnerability arises from the fact that:

* Users accumulate rewards over many epochs without claiming.
* When they eventually claim, the contract calculates each epoch's cap based on the current value of MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR.
* If governance increases this factor before users claim, the users' historical claimable amounts are dramatically reduced.

For example, if a user has unclaimed rewards over 100 epochs when the factor was 10, and governance changes it to 100 right before claiming, each epoch’s cap is reduced tenfold (from epochRewards/10 to epochRewards/100), effectively slashing the user's rewards retroactively.

## Impact Details

* Retroactive change of the MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR enables governance to arbitrarily reduce user rewards for past epochs.
* Users who delay claiming are most affected, potentially losing significant portions of rewards.
* The economic fairness of the protocol can be compromised, undermining trust in the reward mechanism.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Reward.sol#L187>

## Proof of Concept

## Proof of Concept

1. Deploy the Reward and RewardSettings contracts.
2. Set MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR to a small number (e.g., 10).
3. Allow users to accumulate unclaimed rewards over several epochs.
4. Before users claim, increase the factor to a large number (e.g., 1000).
5. Have users call getClaimableAmount() — notice their claimable amounts are drastically lower than expected.
