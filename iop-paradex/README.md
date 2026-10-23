# Paradex | IOP

## Reports by Severity

<details>

<summary>Critical</summary>

* \#47198 \[SC-Critical] The operator can perform unauthorized fund transfers.
* \#47370 \[SC-Critical] \`account\_transfer\_partial\` should not be enabled when \`transfer\_registry\_address\` is not configured.
* \#46843 \[SC-Critical] Bypass of Restrictions When Paraclear\_transfer\_registry Is Unregistered

</details>

<details>

<summary>High</summary>

* \#46892 \[SC-High] small deposits could prevent users from withdrawing their funds
* \#46888 \[SC-High] account\_transfer\_partial: lack of input validation when working with signed integers

</details>

<details>

<summary>Medium</summary>

* \#46997 \[SC-Medium] The vault performs an unsafe conversion on the getAccountValue result.
* \#47309 \[SC-Medium] Type mishandling allows for users to withdraw FAST from vault instead of STANDARD
* \#47314 \[SC-Medium] account\_transfer\_partial(...) function doesn't check sender's health after transferring balances
* \#47310 \[SC-Medium] Integer to Felt conversion completely ruins the Vaults accounting
* \#46856 \[SC-Medium] The calculation of shares obtained through token trades will be incorrect, causing users to pay excessive yield fees.

</details>

<details>

<summary>Low</summary>

* \#47316 \[SC-Low] account\_transfer\_partial(...) function doesn't check that receiver has a registered account in the system
* \#47317 \[SC-Low] Transfer function only allows collateral transfers from free balance but can be bypassed
* \#47330 \[SC-Low] The fee calculation in \`settle\_market\` is unreasonable.
* \#47351 \[SC-Low] Funds get stuck in the bridge if attempted to be deposited into a restricted address
* \#46942 \[SC-Low] set\_perpetual\_asset\_balance\_link - there is no cycle checks
* \#46639 \[SC-Low] The \`\_settlement\_fee\_payments\` function contains a calculation error that leads to abnormal user balances.
* \#46839 \[SC-Low] \`max\_withdraw\` and \`max\_withdraw\` do not fully consider global restrictions.

</details>

<details>

<summary>Insight</summary>

* \#46867 \[SC-Insight] The \`is\_liquidation\` field in \`transfer\_internal\` is not properly differentiated.
* \#47257 \[SC-Insight] Lack of position quantity limit for a single account.
* \#46960 \[SC-Insight] trade order sizes are not validated properly
* \#46989 \[SC-Insight] Invalid trade side check
* \#46910 \[SC-Insight] Token Balance Event Data Inconsistency in Position Transfers
* \#47291 \[SC-Insight] Serveal bugs in function set\_prices\_and\_funding\_snapshot
* \#47295 \[SC-Insight] Configurator Can Manipulate Critical Parameters to Force Mass Liquidations and Drain Protocol Funds
* \#47313 \[SC-Insight] Transfer(...) function doesn't account for current USDC price
* \#47318 \[SC-Insight] If the counterparty happens to be their own referrer, the protocol does not take the referral fee into account during the risk check.
* \#47377 \[SC-Insight] No Restriction on Self Transfer
* \#47380 \[SC-Insight] Incorrect token\_assets\_value in AccountLiquidated Event
* \#46570 \[SC-Insight] account list DoS issue
* \#46747 \[SC-Insight] Self-Referral Vulnerability in Account Referral System
* \#46675 \[SC-Insight] Insufficient Time Validation in function settle\_trade\_v2
* \#46676 \[SC-Insight] Unrestricted Minimum Lockup Period
* \#47299 \[SC-Insight] The \`is\_risky\` check is improper.
* \#46611 \[SC-Insight] Missing staleness checks in oracle queries

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#46997 \[SC-Medium] The vault performs an unsafe conversion on the getAccountValue result.
* \#46867 \[SC-Insight] The \`is\_liquidation\` field in \`transfer\_internal\` is not properly differentiated.
* \#47257 \[SC-Insight] Lack of position quantity limit for a single account.
* \#46892 \[SC-High] small deposits could prevent users from withdrawing their funds
* \#47198 \[SC-Critical] The operator can perform unauthorized fund transfers.
* \#46960 \[SC-Insight] trade order sizes are not validated properly
* \#46989 \[SC-Insight] Invalid trade side check
* \#46910 \[SC-Insight] Token Balance Event Data Inconsistency in Position Transfers
* \#47291 \[SC-Insight] Serveal bugs in function set\_prices\_and\_funding\_snapshot
* \#47295 \[SC-Insight] Configurator Can Manipulate Critical Parameters to Force Mass Liquidations and Drain Protocol Funds
* \#47309 \[SC-Medium] Type mishandling allows for users to withdraw FAST from vault instead of STANDARD
* \#47313 \[SC-Insight] Transfer(...) function doesn't account for current USDC price
* \#47314 \[SC-Medium] account\_transfer\_partial(...) function doesn't check sender's health after transferring balances
* \#47316 \[SC-Low] account\_transfer\_partial(...) function doesn't check that receiver has a registered account in the system
* \#47317 \[SC-Low] Transfer function only allows collateral transfers from free balance but can be bypassed
* \#47318 \[SC-Insight] If the counterparty happens to be their own referrer, the protocol does not take the referral fee into account during the risk check.
* \#47330 \[SC-Low] The fee calculation in \`settle\_market\` is unreasonable.
* \#47351 \[SC-Low] Funds get stuck in the bridge if attempted to be deposited into a restricted address
* \#47370 \[SC-Critical] \`account\_transfer\_partial\` should not be enabled when \`transfer\_registry\_address\` is not configured.
* \#47377 \[SC-Insight] No Restriction on Self Transfer
* \#47380 \[SC-Insight] Incorrect token\_assets\_value in AccountLiquidated Event
* \#46570 \[SC-Insight] account list DoS issue
* \#46888 \[SC-High] account\_transfer\_partial: lack of input validation when working with signed integers
* \#46747 \[SC-Insight] Self-Referral Vulnerability in Account Referral System
* \#47310 \[SC-Medium] Integer to Felt conversion completely ruins the Vaults accounting
* \#46675 \[SC-Insight] Insufficient Time Validation in function settle\_trade\_v2
* \#46676 \[SC-Insight] Unrestricted Minimum Lockup Period
* \#46942 \[SC-Low] set\_perpetual\_asset\_balance\_link - there is no cycle checks
* \#46639 \[SC-Low] The \`\_settlement\_fee\_payments\` function contains a calculation error that leads to abnormal user balances.
* \#47299 \[SC-Insight] The \`is\_risky\` check is improper.
* \#46839 \[SC-Low] \`max\_withdraw\` and \`max\_withdraw\` do not fully consider global restrictions.
* \#46611 \[SC-Insight] Missing staleness checks in oracle queries
* \#46843 \[SC-Critical] Bypass of Restrictions When Paraclear\_transfer\_registry Is Unregistered
* \#46856 \[SC-Medium] The calculation of shares obtained through token trades will be incorrect, causing users to pay excessive yield fees.

</details>
