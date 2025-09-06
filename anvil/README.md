# Anvil

## Reports by Severity

[Critical](<README.md#critical>) | [Medium](<README.md#medium>) | [Low](<README.md#low>) | [Insight](<README.md#insight>)
<details>

<summary>Critical</summary>

* [36554 - [SC - Critical] Time Based Collateral Pool Users can release more than their due share of the pool, drawing from the due share of other users](./36554-sc-critical-time-based-collateral-pool-users-can-release-more-than-their-due-share-of-the-pool.md)

</details>

<details>

<summary>Medium</summary>

* [36475 - [SC - Medium] Token allowance signature can be front-run](./36475-sc-medium-token-allowance-signature-can-be-front-run.md)
* [36532 - [SC - Medium] Frontrun to invalidate collateralizable approval signature](./36532-sc-medium-frontrun-to-invalidate-collateralizable-approval-signature.md)
* [36552 - [SC - Medium] DoS for the user's calling \`stake\` and \`stakeReleasableTokensFrom\` function](./36552-sc-medium-dos-for-the-users-calling-stake-and-stakereleasabletokensfrom-function.md)
* [36567 - [SC - Medium] Anyone can cancel anyone's LOC](./36567-sc-insight-anyone-can-cancel-anyones-loc.md)
* [36268 - [SC - Medium] stake with signature can be front-run lead to user's stake failed](./36268-sc-medium-stake-with-signature-can-be-front-run-lead-to-users-stake-failed.md)
* [36303 - [SC - Medium] Attackers can cause griefing attack to cause stake transactions of TimeBasedCollateralPool of users to always revert by front-running the user transaction to make the provided si...](./36303-sc-medium-attackers-can-cause-griefing-attack-to-cause-stake-transactions-of-timebasedcolla.md)
* [36501 - [SC - Medium] Signature Front-Running Vulnerability in CollateralVault](./36501-sc-medium-signature-front-running-vulnerability-in-collateralvault.md)

</details>

<details>

<summary>Low</summary>

* [36309 - [SC - Low] TimeBasedCollateralPool: After \_resetPool gets called (internally) a depositor can break most functionalities of the smart contract](./36309-sc-low-timebasedcollateralpool-after-_resetpool-gets-called-internally-a-depositor-can-break-m.md)
* [36450 - [SC - Low] Contract TimeBasedCollateralPool will be unable to process new user transactions and user funds are temporary frozen if a user unstake transaction of TimeBasedCollateralPool execute...](./36450-sc-low-contract-timebasedcollateralpool-will-be-unable-to-process-new-user-transactions.md)

</details>

<details>

<summary>Insight</summary>

* [36340 - [SC - Insight] TimeBasedCollateralPool::\_resetAccountTokenStateIfApplicable does not adjust tokenEpochExitBalances after redeeming the account's unstake Units](./36340-sc-insight-timebasedcollateralpool-_resetaccounttokenstateifapplicable-does-not-adjust-tokenep.md)
* [36346 - [SC - Insight] Typehash Discrepancy in CollateralizableTokenAllowanceAdjustment](./36346-sc-insight-typehash-discrepancy-in-collateralizabletokenallowanceadjustment.md)
* [36306 - [SC - Insight] Incorrect nonce value emitted in \`TimeBasedCollateralPool::\_resetPool\` event](./36306-sc-insight-incorrect-nonce-value-emitted-in-timebasedcollateralpool-_resetpool-event.md)
* [36540 - [SC - Insight] Users Can Withdraw Funds at Incorrect Fee Rate](./36540-sc-insight-users-can-withdraw-funds-at-incorrect-fee-rate.md)
* [36092 - [SC - Insight] Collateralizable Contracts May Retain Status Unconditionally](./36092-sc-insight-collateralizable-contracts-may-retain-status-unconditionally.md)
* [36136 - [SC - Insight] Fee calculation error in withdraw function of collateralVault contract](./36136-sc-insight-fee-calculation-error-in-withdraw-function-of-collateralvault-contract.md)
* [36267 - [SC - Insight] Tokens can be stuck forever in UniswapLiquidator because function retrieveTokens always reverts for USDT and all tokens that have transfer function that do not return boolean](./36267-sc-insight-tokens-can-be-stuck-forever-in-uniswapliquidator-because-function-retrievetokens.md)

</details>

## Reports by Type

[Smart Contract](<README.md#smart-contract>)
<details>

<summary>Smart Contract</summary>

* [36309 - [SC - Low] TimeBasedCollateralPool: After \_resetPool gets called (internally) a depositor can break most functionalities of the smart contract](./36309-sc-low-timebasedcollateralpool-after-_resetpool-gets-called-internally-a-depositor-can-break-m.md)
* [36340 - [SC - Insight] TimeBasedCollateralPool::\_resetAccountTokenStateIfApplicable does not adjust tokenEpochExitBalances after redeeming the account's unstake Units](./36340-sc-insight-timebasedcollateralpool-_resetaccounttokenstateifapplicable-does-not-adjust-tokenep.md)
* [36346 - [SC - Insight] Typehash Discrepancy in CollateralizableTokenAllowanceAdjustment](./36346-sc-insight-typehash-discrepancy-in-collateralizabletokenallowanceadjustment.md)
* [36450 - [SC - Low] Contract TimeBasedCollateralPool will be unable to process new user transactions and user funds are temporary frozen if a user unstake transaction of TimeBasedCollateralPool execute...](./36450-sc-low-contract-timebasedcollateralpool-will-be-unable-to-process-new-user-transactions.md)
* [36475 - [SC - Medium] Token allowance signature can be front-run](./36475-sc-medium-token-allowance-signature-can-be-front-run.md)
* [36306 - [SC - Insight] Incorrect nonce value emitted in \`TimeBasedCollateralPool::\_resetPool\` event](./36306-sc-insight-incorrect-nonce-value-emitted-in-timebasedcollateralpool-_resetpool-event.md)
* [36532 - [SC - Medium] Frontrun to invalidate collateralizable approval signature](./36532-sc-medium-frontrun-to-invalidate-collateralizable-approval-signature.md)
* [36552 - [SC - Medium] DoS for the user's calling \`stake\` and \`stakeReleasableTokensFrom\` function](./36552-sc-medium-dos-for-the-users-calling-stake-and-stakereleasabletokensfrom-function.md)
* [36554 - [SC - Critical] Time Based Collateral Pool Users can release more than their due share of the pool, drawing from the due share of other users](./36554-sc-critical-time-based-collateral-pool-users-can-release-more-than-their-due-share-of-the-pool.md)
* [36567 - [SC - Medium] Anyone can cancel anyone's LOC](./36567-sc-insight-anyone-can-cancel-anyones-loc.md)
* [36540 - [SC - Insight] Users Can Withdraw Funds at Incorrect Fee Rate](./36540-sc-insight-users-can-withdraw-funds-at-incorrect-fee-rate.md)
* [36092 - [SC - Insight] Collateralizable Contracts May Retain Status Unconditionally](./36092-sc-insight-collateralizable-contracts-may-retain-status-unconditionally.md)
* [36136 - [SC - Insight] Fee calculation error in withdraw function of collateralVault contract](./36136-sc-insight-fee-calculation-error-in-withdraw-function-of-collateralvault-contract.md)
* [36267 - [SC - Insight] Tokens can be stuck forever in UniswapLiquidator because function retrieveTokens always reverts for USDT and all tokens that have transfer function that do not return boolean](./36267-sc-insight-tokens-can-be-stuck-forever-in-uniswapliquidator-because-function-retrievetokens.md)
* [36268 - [SC - Medium] stake with signature can be front-run lead to user's stake failed](./36268-sc-medium-stake-with-signature-can-be-front-run-lead-to-users-stake-failed.md)
* [36303 - [SC - Medium] Attackers can cause griefing attack to cause stake transactions of TimeBasedCollateralPool of users to always revert by front-running the user transaction to make the provided si...](./36303-sc-medium-attackers-can-cause-griefing-attack-to-cause-stake-transactions-of-timebasedcolla.md)
* [36501 - [SC - Medium] Signature Front-Running Vulnerability in CollateralVault](./36501-sc-medium-signature-front-running-vulnerability-in-collateralvault.md)

</details>
