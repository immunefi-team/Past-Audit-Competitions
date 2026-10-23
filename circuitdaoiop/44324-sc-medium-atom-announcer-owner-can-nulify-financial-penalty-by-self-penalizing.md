# #44324 \[SC-Medium] atom announcer owner can nulify financial penalty by self penalizing&#x20;

## #44324 \[SC-Medium] Atom Announcer Owner Can Nulify Financial Penalty by Self-Penalizing

**Submitted on Apr 21st 2025 at 14:48:17 UTC by @perseverance for** [**IOP | CircuitDAO**](https://immunefi.com/audit-competition/iop-circuitdao)

* **Report ID:** #44324
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/CircuitDAO-IoP/tree/main/circuit\\_puzzles>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Protocol insolvency

### Description

## Short Summary

The `announcer_penalize.clsp` puzzle allows the keeper (means anyone), including the announcer coin's owner, to trigger a penalty if the conditions are met. While the penalty correctly reduces the `DEPOSIT` amount stored in the announcer coin's state, the keeper triggering the penalty (the "keeper") can claim the slashed amount. If the owner self-penalizes, they can direct this slashed amount back to themselves, effectively nullifying the intended financial disincentive of the penalty mechanism for the announcer.

So the attacker can potentially exploit this to behave badly. This can potentially affect protocol's health.

## Background Information

The Circuit DAO documentation emphasizes the importance of the penalization mechanism implemented in `announcer_penalize.clsp` to incentivize good behavior from data providers (Announcers): Ref: <https://docs.circuitdao.com/technical-manual/announcers>

> "Having a penalization mechanism in place protects the protocol as it sets a strong incentive for data providers to do their job well."

> "An Announcer is penalized by applying the Penalty Factor to the Announcer's deposit. This slashes XCH from the Announcer coin's amount. The slashed amount can either be claimed by the keeper performing the penalization or wholly or partly left to the farmer of the block."

## The Vulnerability / Vulnerability Details

The `announcer_penalize.clsp` puzzle, when called via the `atom_announcer.clsp` wrapper, can be triggered by **any party ("keeper") who provides a valid solution**, provided the penalization conditions within the puzzle are met (`current_timestamp > PENALIZABLE_AT`, `APPROVED = 1`, and at least one violation like `> current_timestamp TIMESTAMP_EXPIRES`).

<https://github.com/immunefi-team/CircuitDAO-IoP/blob/d2c3171f08864c29fdd436e25a39c95b371df860/circuit\\_puzzles/programs/announcer\\_penalize.clsp#L26-L38>

```chialisp
(assert
      ; must be past penalty timestamp to penalize
      (> current_timestamp PENALIZABLE_AT)
      (any
        ; penalize if expired
        (> current_timestamp TIMESTAMP_EXPIRES)
        ; penalize if deposit is less than announcer's min deposit
        (> MIN_DEPOSIT DEPOSIT)
        ; penalize if announcer's min deposit is less than Statues min deposit
        (> min_deposit MIN_DEPOSIT)
        ; penalize if announcer's delay is above expected delay
        (> DELAY announcer_max_delay)
      )
```

Document of CircuitDAO stated (Ref: <https://docs.circuitdao.com/technical-manual/advanced-topics/keepers>)

> Keepers are entities that execute operations on the protocol that anyone can perform.

The owner of "Atom Announcers" means that "data providers" can exploit this to perform self penalize himself. Since the "announcer\_penalize.clsp" does not check where the penalized XCH go to, the owner of "Atom Announcers" can take back the penalized XCH. By exploing this, he can nulify the penalty mechanism.

## Severity assessment

* **Bug Severity:** **Medium**

(Potentially High depending on reliance on this specific incentive)

* **Impact category:**
  * Weakening of core economic incentive mechanism.
  * Bypass of intended financial penalty for the announcer owner.
  * Undermines the effectiveness of the keeper ecosystem for this specific function.
* **Likelihood:** **High**
  * The owner has the full capability to execute this if penalization conditions are met.
  * The owner has a direct financial incentive to self-penalize (to avoid the fee going to an external keeper) if a penalty is inevitable.

**Rationale for Medium/High:** While this doesn't lead to direct theft of user funds, protocol insolvency, or minting unbacked tokens, it significantly degrades a core security mechanism designed to ensure data provider reliability. The effectiveness of the penalty as a deterrent is largely nullified *for the owner*. If the protocol heavily relies on this specific financial penalty to ensure announcer uptime and correctness, the severity could be judged as High. However, Medium reflects that core protocol funds are not directly at risk, and the coin state *is* updated correctly.

## Recommendation / Mitigation:

The most direct way to mitigate this is to modify the `announcer_penalize.clsp` puzzle to handle the slashed amount differently, rather than allowing it to be claimed by anyone. Options include:

1. **Send Fee to Treasury:** Modify the puzzle to explicitly create a `CREATE_COIN` condition sending the slashed amount to a predefined treasury address.
2. **Have some access control** : Just some user can perform this penalty action.

Option 1 may be is better and more aligned with the current design.

### Proof of Concept

## Proof of concept (Conceptual Steps):

Step 1 . An Announcer Owner identifies that their Atom Announcer coin meets the conditions for penalization (e.g., `current_timestamp > PENALIZABLE_AT` and `current_timestamp > TIMESTAMP_EXPIRES`).

Step 2: The owner of data provider means "atom\_announcer" craft a Spend Bundle and sign for it. Try to do it quickly when the conditions are met and try to front-run anyone else.

The Owner calculates the `penalized_deposit` using the formula in `announcer_penalize.clsp`.

The Owner crafts a solution for their `atom_announcer.clsp` coin, specifying `announcer_penalize.clsp` as the `operation`.

The Owner provides the other necessary arguments for `announcer_penalize` (penalty factors, timestamps, etc., matching current Statutes).

The Owner constructs the transaction spending the `atom_announcer` coin with this solution and signs it.

Result: Upon successful execution:

```
*   The child Announcer coin is created with `amount = penalized_deposit`.

*   The `PENALIZABLE_AT` time is updated.

*   The Owner receives back the penalized XCH. 

*   The net change in the Owner's XCH holdings (ignoring blockchain fees) is zero, despite the Announcer coin's deposit being slashed.
```
