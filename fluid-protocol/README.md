# Fluid Protocol

## Reports by Severity

[Critical](<README.md#critical>) | [Medium](<README.md#medium>) | [Low](<README.md#low>) | [Insight](<README.md#insight>)
<details>

<summary>Critical</summary>

* [37671 - [SC - Critical] CRITICAL-02 / The contract could be permanently locked due to not reseting the boolen lock](./37671-sc-critical-critical-02-the-contract-could-be-permanently-locked-due-to-not-reseting-the-boole.md)
* [37323 - [SC - Critical] Permanent dead Lock in internal\_redeem\_collateral\_from\_trove](./37323-sc-critical-permanent-dead-lock-in-internal_redeem_collateral_from_trove.md)
* [37452 - [SC - Critical] \`trove-manager-contract.redeem\_collateral\_from\_trove\` can be locked forever](./37452-sc-critical-trove-manager-contract.redeem_collateral_from_trove-can-be-locked-forever.md)
* [37624 - [SC - Critical] Lock Issue Bricks The Redeem Functionality](./37624-sc-critical-lock-issue-bricks-the-redeem-functionality.md)

</details>

<details>

<summary>Medium</summary>

* [37276 - [SC - Medium] Redstone's price feed is used incorrectly.](./37276-sc-medium-redstones-price-feed-is-used-incorrectly..md)

</details>

<details>

<summary>Low</summary>

* [37668 - [SC - Low] Incorrect Scale Factor value leads to early scale change](./37668-sc-low-incorrect-scale-factor-value-leads-to-early-scale-change.md)
* [37650 - [SC - Low] Redeem Functionality Partially Failing](./37650-sc-low-redeem-functionality-partially-failing.md)
* [37354 - [SC - Low] Single below MCR trove temporarily blocks redemptions](./37354-sc-low-single-below-mcr-trove-temporarily-blocks-redemptions.md)
* [37283 - [SC - Low] Improper Trove Validation Check Allows Low-Cost Griefing Attack to Block Protocol Redemptions](./37283-sc-low-improper-trove-validation-check-allows-low-cost-griefing-attack-to-block-protocol-redem.md)
* [37192 - [SC - Low] Trove that under MCR might be redeemed.](./37192-sc-low-trove-that-under-mcr-might-be-redeemed..md)
* [37409 - [SC - Low] Can not redeem when all \`current\_cr\` less than \`MCR\`.](./37409-sc-low-can-not-redeem-when-all-current_cr-less-than-mcr-..md)
* [37607 - [SC - Low] Bricking Redeem Function](./37607-sc-low-bricking-redeem-function.md)

</details>

<details>

<summary>Insight</summary>

* [36922 - [SC - Insight] the function claim\_collateral in borrowOperation have read only attribute while the invoked claim\_collateral function have write attribute, this lead to compiler-time error](./36922-sc-insight-the-function-claim_collateral-in-borrowoperation-have-read-only-attribute-while-the.md)
* [37139 - [SC - Insight] INSIGHT: Inefficient Use of Storage Reentrancy Locks](./37139-sc-insight-insight-inefficient-use-of-storage-reentrancy-locks.md)
* [37056 - [SC - Insight] \`require\_at\_least\_min\_net\_debt\` did not emit correct error message](./37056-sc-insight-require_at_least_min_net_debt-did-not-emit-correct-error-message.md)
* [37202 - [SC - Insight] some checks can be removed since its not required(best practice report, not an issue)](./37202-sc-insight-some-checks-can-be-removed-since-its-not-required-best-practice-report-not-an-issue.md)
* [37343 - [SC - Insight] Inaccurate Check Leading to Debt Miscalculation](./37343-sc-insight-inaccurate-check-leading-to-debt-miscalculation.md)
* [37382 - [SC - Insight] Inconsistent Collateral Ratio Checks in Stability Pool Withdrawals Lead to Fund-Locking DoS](./37382-sc-insight-inconsistent-collateral-ratio-checks-in-stability-pool-withdrawals-lead-to-fund-loc.md)
* [37425 - [SC - Insight] redeem\_collateral does not redeem collateral from riskiest trove but wrongly redeem lowest healthy troves with lowest collateral Ratio](./37425-sc-insight-redeem-collateral-does-not-redeem-collateral-from-riskiest-trove-but-wrongly-redeem.md)
* [37595 - [SC - Insight] \`require\_caller\_is\_bo\_or\_tm\_or\_sp\_or\_pm\` did not emit correct message](./37595-sc-insight-require_caller_is_bo_or_tm_or_sp_or_pm-did-not-emit-correct-message.md)

</details>

## Reports by Type

[Smart Contract](<README.md#smart-contract>)
<details>

<summary>Smart Contract</summary>

* [37668 - [SC - Low] Incorrect Scale Factor value leads to early scale change](./37668-sc-low-incorrect-scale-factor-value-leads-to-early-scale-change.md)
* [37650 - [SC - Low] Redeem Functionality Partially Failing](./37650-sc-low-redeem-functionality-partially-failing.md)
* [37276 - [SC - Medium] Redstone's price feed is used incorrectly.](./37276-sc-medium-redstones-price-feed-is-used-incorrectly..md)
* [36922 - [SC - Insight] the function claim\_collateral in borrowOperation have read only attribute while the invoked claim\_collateral function have write attribute, this lead to compiler-time error](./36922-sc-insight-the-function-claim_collateral-in-borrowoperation-have-read-only-attribute-while-the.md)
* [37139 - [SC - Insight] INSIGHT: Inefficient Use of Storage Reentrancy Locks](./37139-sc-insight-insight-inefficient-use-of-storage-reentrancy-locks.md)
* [37354 - [SC - Low] Single below MCR trove temporarily blocks redemptions](./37354-sc-low-single-below-mcr-trove-temporarily-blocks-redemptions.md)
* [37671 - [SC - Critical] CRITICAL-02 / The contract could be permanently locked due to not reseting the boolen lock](./37671-sc-critical-critical-02-the-contract-could-be-permanently-locked-due-to-not-reseting-the-boole.md)
* [37056 - [SC - Insight] \`require\_at\_least\_min\_net\_debt\` did not emit correct error message](./37056-sc-insight-require_at_least_min_net_debt-did-not-emit-correct-error-message.md)
* [37202 - [SC - Insight] some checks can be removed since its not required(best practice report, not an issue)](./37202-sc-insight-some-checks-can-be-removed-since-its-not-required-best-practice-report-not-an-issue.md)
* [37283 - [SC - Low] Improper Trove Validation Check Allows Low-Cost Griefing Attack to Block Protocol Redemptions](./37283-sc-low-improper-trove-validation-check-allows-low-cost-griefing-attack-to-block-protocol-redem.md)
* [37323 - [SC - Critical] Permanent dead Lock in internal\_redeem\_collateral\_from\_trove](./37323-sc-critical-permanent-dead-lock-in-internal_redeem_collateral_from_trove.md)
* [37343 - [SC - Insight] Inaccurate Check Leading to Debt Miscalculation](./37343-sc-insight-inaccurate-check-leading-to-debt-miscalculation.md)
* [37382 - [SC - Insight] Inconsistent Collateral Ratio Checks in Stability Pool Withdrawals Lead to Fund-Locking DoS](./37382-sc-insight-inconsistent-collateral-ratio-checks-in-stability-pool-withdrawals-lead-to-fund-loc.md)
* [37192 - [SC - Low] Trove that under MCR might be redeemed.](./37192-sc-low-trove-that-under-mcr-might-be-redeemed..md)
* [37425 - [SC - Insight] redeem\_collateral does not redeem collateral from riskiest trove but wrongly redeem lowest healthy troves with lowest collateral Ratio](./37425-sc-insight-redeem-collateral-does-not-redeem-collateral-from-riskiest-trove-but-wrongly-redeem.md)
* [37409 - [SC - Low] Can not redeem when all \`current\_cr\` less than \`MCR\`.](./37409-sc-low-can-not-redeem-when-all-current_cr-less-than-mcr-..md)
* [37452 - [SC - Critical] \`trove-manager-contract.redeem\_collateral\_from\_trove\` can be locked forever](./37452-sc-critical-trove-manager-contract.redeem_collateral_from_trove-can-be-locked-forever.md)
* [37595 - [SC - Insight] \`require\_caller\_is\_bo\_or\_tm\_or\_sp\_or\_pm\` did not emit correct message](./37595-sc-insight-require_caller_is_bo_or_tm_or_sp_or_pm-did-not-emit-correct-message.md)
* [37607 - [SC - Low] Bricking Redeem Function](./37607-sc-low-bricking-redeem-function.md)
* [37624 - [SC - Critical] Lock Issue Bricks The Redeem Functionality](./37624-sc-critical-lock-issue-bricks-the-redeem-functionality.md)

</details>
