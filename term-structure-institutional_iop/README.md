# Term Structure Institutional | IOP

## Reports by Severity

<details>

<summary>Critical</summary>

* \#46903 \[SC-Critical] malicious borrower can take theft of other borrower collateral
* \#47112 \[SC-Critical] addCollateral causes double economic loss through premature asset transfer and inflated settlement requirements
* \#47115 \[SC-Critical] Missing Settlement Status Validation in Loan Operations
* \#46819 \[SC-Critical] direct theft of users funds when expired loan get liquidated

</details>

<details>

<summary>High</summary>

* \#46893 \[SC-High] settlement functionality can be break forever and blocking settlement actions.
* \#47008 \[SC-High] any users with expired loan(not settled) can take theft of lenders collateral when the collateral price increase
* \#47118 \[SC-High] Incorrect Allowance Validation in addCollateralBeforeSettle

</details>

<details>

<summary>Medium</summary>

* \#47125 \[SC-Medium] Cross-Chain Signature Replay Attack in Settlement Contract
* \#46608 \[SC-Medium] Any call to the repay function can potentially be front-run by a malicious actor, lead to prevent users from repaying on time.
* \#47122 \[SC-Medium] Array Length Mismatch Enables Partial Settlement Processing

</details>

<details>

<summary>Low</summary>

* \#47009 \[SC-Low] Any position can be closed (by repaying the debt) even after the maturity date has passed

</details>

<details>

<summary>Insight</summary>

* \#47100 \[SC-Insight] some checks should be added even if the operator checks each input parameters
* \#47124 \[SC-Insight] Minimum Debt Value Updates Trigger Instant Liquidation Condition Changes

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#46893 \[SC-High] settlement functionality can be break forever and blocking settlement actions.
* \#46903 \[SC-Critical] malicious borrower can take theft of other borrower collateral
* \#47125 \[SC-Medium] Cross-Chain Signature Replay Attack in Settlement Contract
* \#46608 \[SC-Medium] Any call to the repay function can potentially be front-run by a malicious actor, lead to prevent users from repaying on time.
* \#47008 \[SC-High] any users with expired loan(not settled) can take theft of lenders collateral when the collateral price increase
* \#47122 \[SC-Medium] Array Length Mismatch Enables Partial Settlement Processing
* \#47118 \[SC-High] Incorrect Allowance Validation in addCollateralBeforeSettle
* \#47112 \[SC-Critical] addCollateral causes double economic loss through premature asset transfer and inflated settlement requirements
* \#47100 \[SC-Insight] some checks should be added even if the operator checks each input parameters
* \#47124 \[SC-Insight] Minimum Debt Value Updates Trigger Instant Liquidation Condition Changes
* \#47115 \[SC-Critical] Missing Settlement Status Validation in Loan Operations
* \#47009 \[SC-Low] Any position can be closed (by repaying the debt) even after the maturity date has passed
* \#46819 \[SC-Critical] direct theft of users funds when expired loan get liquidated

</details>
