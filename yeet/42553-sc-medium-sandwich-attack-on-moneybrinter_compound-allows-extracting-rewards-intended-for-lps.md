# #42553 \[SC-Medium] Sandwich attack on \`MoneyBrinter\_compound\` allows extracting rewards intended for LPs

**Submitted on Mar 24th 2025 at 16:06:04 UTC by @OldDingo56530 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42553
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Description

The `MoneyBrinter::compound` function is vulnerable to a sandwich attack that allows an attacker to extract rewards intended for legitimate liquidity providers (LPs).

An attacker can monitor the mempool for `compound` calls, front-run with a maximal deposit to capture a majority of the share supply, wait for `compound` to execute, then back-run with an immediate withdrawal to extract the share's inflated value. This attack is atomic within one block and remains profitable as long as the extracted value exceeds the withdrawal fee.

With a sufficient liquidity and infrequent `compound` calls, a significant amount of rewards will accumulate, making this attack even more lucrative for the attacker.

### Recommendation

One solution could be:

* Disallow deposits in the same block as the `compound` call to prevent front-running

This mitigation prevents the attacker from front-running the `compound` call with a deposit in the same block. By ensuring that deposits and the `compound` function cannot be executed in the same block, the attacker is unable to predict and front-run the `compound` call, rendering the sandwich attack infeasible.

## Proof of Concept

## Proof of Concept

1. Attacker monitors the mempool for upcoming calls to `MoneyBrinter::compound` by the manager.
2. Attacker front-runs the `compound` call with a maximal deposit to gain a majority share of the `MoneyBrinter` LP token supply.
3. The sandwiched `compound` call executes, depositing previously harvested rewards from `harvestKodiakRewards` and `harvestBeradromeRewards`, which increases the underlying share value.
4. Attacker back-runs the `compound` call with an immediate withdrawal of their inflated shares, extracting the harvested rewards (minus the 4% fees).
5. Attacker repeats this attack on all future worth `compound` calls.
