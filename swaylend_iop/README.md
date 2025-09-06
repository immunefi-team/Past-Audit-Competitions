# Swaylend | IOP

## Reports by Severity

[Critical](<README.md#critical>) | [High](<README.md#high>) | [Medium](<README.md#medium>) | [Low](<README.md#low>) | [Insight](<README.md#insight>)
<details>

<summary>Critical</summary>

* [35767 - [SC - Critical] constanct value is used to check \`price.confidence\`](./35767-sc-critical-constanct-value-is-used-to-check-price.confidence.md)
* [35758 - [SC - Critical] Loss of yield to the protocol due to incorrect interest rate applied](./35758-sc-critical-loss-of-yield-to-the-protocol-due-to-incorrect-interest-rate-applied.md)
* [35684 - [SC - Critical] Incorrect Pyth Oracle Price Feed Process Leads to Wrong Collateral Value Calculation](./35684-sc-critical-incorrect-pyth-oracle-price-feed-process-leads-to-wrong-collateral-value-calculati.md)

</details>

<details>

<summary>High</summary>

* [35793 - [SC - High] \`src-20.burn\` should use "==" instead of ">="](./35793-sc-high-src-20.burn-should-use-instead-of-greater-than.md)
* [35876 - [SC - High] Users will lose funds on calls to critical functions if the prices are not updated](./35876-sc-high-users-will-lose-funds-on-calls-to-critical-functions-if-the-prices-are-not-updated.md)
* [35750 - [SC - High] User loss due to Pyth oracle update fee being smaller than the msg amount sent](./35750-sc-high-user-loss-due-to-pyth-oracle-update-fee-being-smaller-than-the-msg-amount-sent.md)
* [36117 - [SC - High] Permanent freezing of tokens when user sends extra tokens as update fee](./36117-sc-high-permanent-freezing-of-tokens-when-user-sends-extra-tokens-as-update-fee.md)
* [35831 - [SC - High] By bypassing base\_borrow\_min limitation borrows can create inabsorbable loans](./35831-sc-high-by-bypassing-base_borrow_min-limitation-borrows-can-create-inabsorbable-loans.md)

</details>

<details>

<summary>Medium</summary>

* [35815 - [SC - Medium] \`Market.present\_value\_borrow\` should be roundUp](./35815-sc-medium-market.present_value_borrow-should-be-roundup.md)
* [36137 - [SC - Medium] \`absorb\_internal\` might be DOSed](./36137-sc-medium-absorb_internal-might-be-dosed.md)
* [36034 - [SC - Medium] truncation in the \`present\_value\_borrow()\` can lead to loss of accrued borrow interests.](./36034-sc-medium-truncation-in-the-present_value_borrow-can-lead-to-loss-of-accrued-borrow-interests..m)
* [35853 - [SC - Medium] permissonless constructor always for front-running owner initialization.](./35853-sc-medium-permissonless-constructor-always-for-front-running-owner-initialization..m)

</details>

<details>

<summary>Low</summary>

* [35761 - [SC - Low] Unhandled smaller base decimals than 6 or bigger than the collateral's decimals](./35761-sc-low-unhandled-smaller-base-decimals-than-6-or-bigger-than-the-collaterals-decimals.md)
* [35760 - [SC - Low] \`market::available\_to\_borrow()\` compares the collateral in USD against the borrow in base units](./35760-sc-low-market-available_to_borrow-compares-the-collateral-in-usd-against-the-borrow-in-base-un.md)
* [35724 - [SC - Low] Users can withdraw collateral even when the admin pauses the contract.](./35724-sc-low-users-can-withdraw-collateral-even-when-the-admin-pauses-the-contract..md)
* [36158 - [SC - Low] \`Market.collateral\_value\_to\_sell\` will always revert if collateral\_configuration.decimals < storage.market\_configuration.base\_token\_decimals](./36158-sc-low-market.collateral_value_to_sell-will-always-revert-if-collateral_configuration.md)
* [35908 - [SC - Low] If the collateral token''s decimal is <= the base token decimal in a market, \`collateral\_value\_to\_sell()\` will always revert & \`available\_to\_borrow()\` will return a wrong amount tha...](./35908-sc-low-if-the-collateral-token-s-decimal-is-less-than-the-base-token-decimal-in-a-market-colla.md)
* [35732 - [SC - Low] Withdrawals can not be paused which could lead to protocol insolvency in case of issues](./35732-sc-low-withdrawals-can-not-be-paused-which-could-lead-to-protocol-insolvency-in-case-of-issues.md)

</details>

<details>

<summary>Insight</summary>

* [35708 - [SC - Insight] Adding too many collaterals will halt the protocol operation](./35708-sc-insight-adding-too-many-collaterals-will-halt-the-protocol-operation.md)
* [35999 - [SC - Insight] Incorrect event name](./35999-sc-insight-incorrect-event-name.md)
* [35794 - [SC - Insight] \`Market.absorb\` can be called when \`Market.supply\_collateral\` is paused](./35794-sc-insight-market.absorb-can-be-called-when-market.supply_collateral-is-paused.md)
* [36065 - [SC - Insight] \`Market.update\_market\_configuration\` should reuse old configuration's \`base\_token.decimals\`](./36065-sc-insight-market.update_market_configuration-should-reuse-old-configurations-base_token.decim.md)
* [36108 - [SC - Insight] \`recipient\` with a NULL address will lead to permanent loss of minted coins](./36108-sc-insight-recipient-with-a-null-address-will-lead-to-permanent-loss-of-minted-coins.md)
* [36138 - [SC - Insight] \`Market.update\_collateral\_asset\` should reuse old configuration's \`asset\_id\`](./36138-sc-insight-market.update_collateral_asset-should-reuse-old-configurations-asset_id.md)
* [35768 - [SC - Insight] \`Market.set\_pyth\_contract\_id\` should emit an event](./35768-sc-insight-market.set_pyth_contract_id-should-emit-an-event.md)

</details>

## Reports by Type

[Smart Contract](<README.md#smart-contract>)
<details>

<summary>Smart Contract</summary>

* [35708 - [SC - Insight] Adding too many collaterals will halt the protocol operation](./35708-sc-insight-adding-too-many-collaterals-will-halt-the-protocol-operation.md)
* [35761 - [SC - Low] Unhandled smaller base decimals than 6 or bigger than the collateral's decimals](./35761-sc-low-unhandled-smaller-base-decimals-than-6-or-bigger-than-the-collaterals-decimals.md)
* [35793 - [SC - High] \`src-20.burn\` should use "==" instead of ">="](./35793-sc-high-src-20.burn-should-use-instead-of-greater-than.md)
* [35876 - [SC - High] Users will lose funds on calls to critical functions if the prices are not updated](./35876-sc-high-users-will-lose-funds-on-calls-to-critical-functions-if-the-prices-are-not-updated.md)
* [35767 - [SC - Critical] constanct value is used to check \`price.confidence\`](./35767-sc-critical-constanct-value-is-used-to-check-price.confidence.md)
* [35750 - [SC - High] User loss due to Pyth oracle update fee being smaller than the msg amount sent](./35750-sc-high-user-loss-due-to-pyth-oracle-update-fee-being-smaller-than-the-msg-amount-sent.md)
* [35999 - [SC - Insight] Incorrect event name](./35999-sc-insight-incorrect-event-name.md)
* [35794 - [SC - Insight] \`Market.absorb\` can be called when \`Market.supply\_collateral\` is paused](./35794-sc-insight-market.absorb-can-be-called-when-market.supply_collateral-is-paused.md)
* [35758 - [SC - Critical] Loss of yield to the protocol due to incorrect interest rate applied](./35758-sc-critical-loss-of-yield-to-the-protocol-due-to-incorrect-interest-rate-applied.md)
* [35760 - [SC - Low] \`market::available\_to\_borrow()\` compares the collateral in USD against the borrow in base units](./35760-sc-low-market-available_to_borrow-compares-the-collateral-in-usd-against-the-borrow-in-base-un.md)
* [35815 - [SC - Medium] \`Market.present\_value\_borrow\` should be roundUp](./35815-sc-medium-market.present_value_borrow-should-be-roundup.md)
* [35724 - [SC - Low] Users can withdraw collateral even when the admin pauses the contract.](./35724-sc-low-users-can-withdraw-collateral-even-when-the-admin-pauses-the-contract..md)
* [36065 - [SC - Insight] \`Market.update\_market\_configuration\` should reuse old configuration's \`base\_token.decimals\`](./36065-sc-insight-market.update_market_configuration-should-reuse-old-configurations-base_token.decim.md)
* [36108 - [SC - Insight] \`recipient\` with a NULL address will lead to permanent loss of minted coins](./36108-sc-insight-recipient-with-a-null-address-will-lead-to-permanent-loss-of-minted-coins.md)
* [36117 - [SC - High] Permanent freezing of tokens when user sends extra tokens as update fee](./36117-sc-high-permanent-freezing-of-tokens-when-user-sends-extra-tokens-as-update-fee.md)
* [36137 - [SC - Medium] \`absorb\_internal\` might be DOSed](./36137-sc-medium-absorb_internal-might-be-dosed.md)
* [36138 - [SC - Insight] \`Market.update\_collateral\_asset\` should reuse old configuration's \`asset\_id\`](./36138-sc-insight-market.update_collateral_asset-should-reuse-old-configurations-asset_id.md)
* [36158 - [SC - Low] \`Market.collateral\_value\_to\_sell\` will always revert if collateral\_configuration.decimals < storage.market\_configuration.base\_token\_decimals](./36158-sc-low-market.collateral_value_to_sell-will-always-revert-if-collateral_configuration.md)
* [35831 - [SC - High] By bypassing base\_borrow\_min limitation borrows can create inabsorbable loans](./35831-sc-high-by-bypassing-base_borrow_min-limitation-borrows-can-create-inabsorbable-loans.md)
* [35684 - [SC - Critical] Incorrect Pyth Oracle Price Feed Process Leads to Wrong Collateral Value Calculation](./35684-sc-critical-incorrect-pyth-oracle-price-feed-process-leads-to-wrong-collateral-value-calculati.md)
* [35768 - [SC - Insight] \`Market.set\_pyth\_contract\_id\` should emit an event](./35768-sc-insight-market.set_pyth_contract_id-should-emit-an-event.md)
* [35908 - [SC - Low] If the collateral token''s decimal is <= the base token decimal in a market, \`collateral\_value\_to\_sell()\` will always revert & \`available\_to\_borrow()\` will return a wrong amount tha...](./35908-sc-low-if-the-collateral-token-s-decimal-is-less-than-the-base-token-decimal-in-a-market-colla.md)
* [35732 - [SC - Low] Withdrawals can not be paused which could lead to protocol insolvency in case of issues](./35732-sc-low-withdrawals-can-not-be-paused-which-could-lead-to-protocol-insolvency-in-case-of-issues.md)
* [36034 - [SC - Medium] truncation in the \`present\_value\_borrow()\` can lead to loss of accrued borrow interests.](./36034-sc-medium-truncation-in-the-present_value_borrow-can-lead-to-loss-of-accrued-borrow-interests..m)
* [35853 - [SC - Medium] permissonless constructor always for front-running owner initialization.](./35853-sc-medium-permissonless-constructor-always-for-front-running-owner-initialization..m)

</details>
