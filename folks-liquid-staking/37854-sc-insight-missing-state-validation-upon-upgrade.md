# #37854 \[SC-Insight] Missing state validation upon Upgrade

**Submitted on Dec 17th 2024 at 13:51:59 UTC by @k13n for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37854
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The contract is updatable meaning that the code of the contract can be changed. Right now an old version of this contract is deployed on Algorand and according to the developers this old contract will be updated to contain the code subject to this audit competition. This new code does not validate the global state of the contract upon update. Therefore, certain invariants cannot be guaranteed after an update. This can lead to unexpected behavior and loss of user funds.

## Vulnerability Details

Function `initialise` must be called after upgrading the contract and before any other meaningful operation (minting, burning, etc.) can be executed. This function assumes that certain global state exists and is valid without further checks. In case this global state was manipulated prior to the upgrade, certain invariants may not hold.

Function `update_fee` ensures the invariant that the fee is between 0 and 100%. This invariant is asserted in function `check_fee` which checks that the fee is below 100% (since Algorand only supports unsigned integers, the fee must be >= 0%). This `check_fee()` (and other similar sanity checks) are missing in the `initialise` function. Therefore, it is possible that the invariant doesn't hold.

## Impact Details

Assume that through the above loophole the fee is set to 10000% by a rogue admin. As a result, the admin can collect more fees than he should possibly get and these additional fees are taken from the users' stake, resulting in the theft of user funds.

## Recommendation

Check that the given invariants (`check_fee`, `check_premium`, ...) are maintained after updating in `initialise`.

## References

* <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py#L283>
* <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py#L72>

## Proof of Concept

## Proof of Concept

### Initial State

* The old code (code that is not subject to this audit) is deployed on Algorand

### (Rogue) Admin prepares Attack

* A (rouge) admin manipulates the global state and for example sets the fee to 10'000%

### Admin deploys update

* An admin updates the contract and deploys the code subject to this audit competition

### Attacker withdraws user funds

* The attacker makes a protocol donation to ensure that rewards are > 0
* The attacker calls `claim_fee`, which in turns calls `send_unclaimed_fees` and `update_total_rewards_and_unclaimed_fees`.
* Here the code computes the new rewards (which are > 0 because of the donation) and multiplies that with the fee percentage that the attacker set to > 100%. As a result, the global state `total_unclaimed_fees` is updated to a large value, larger than the actual rewards that have been collected
* Function `claim_fee` transfers the fees to the admin account which results in the theft of user funds
