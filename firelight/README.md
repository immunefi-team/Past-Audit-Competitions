# Firelight

## Reports by Severity

<details>

<summary>Low</summary>

* \#59091 \[SC-Low] \[LOW]: `FirelightVault.sol` Implementation Contract Does Not Disable Initializers
* \#59226 \[SC-Low] Logic Flaw in periodAtTimestamp Function Breaks Historical Queries, Returning Current Period Instead
* \#59023 \[SC-Low] Unprotected Implementation Contract Initializer Allows Unauthorized Admin Role Assignment Leading to Potential Governance Manipulation
* \#59100 \[SC-Low] periodAtTimestamp() Returns Current Period Instead of Queried Historical Period
* \#59334 \[SC-Low] `periodAtTimestamp` function uses current timestamp instead of input parameter, causing incorrect period calculation for historical or future queries
* \#59937 \[SC-Low] periodAtTimestamp() uses current time instead of input timestamp, returning wrong period numbers
* \#59422 \[SC-Low] periodAtTimestamp ignores the supplied timestamp
* \#59467 \[SC-Low] periodAtTimestamp() Ignores Input Parameter
* \#59715 \[SC-Low] `periodAtTimestamp` will return different period for the same timestamp input
* \#59967 \[SC-Low] Broken Historical Period Calculation
* \#59879 \[SC-Low] Logic bug in `periodAtTimestamp`
* \#59605 \[SC-Low] Logic error in periodAtTimestamp returns incorrect periods
* \#59280 \[SC-Low] `periodAtTimestamp(uint48 timestamp)` ignores `timestamp` and return incorrect values when it is not `Time.timestamp()`
* \#58993 \[SC-Low] Incorrect timestamp calculation in `periodAtTimestamp()` leads to broken historical period lookups
* \#59635 \[SC-Low] Timestamp-agnostic periodAtTimestamp misreports historical periods, breaking time-locked logic
* \#59820 \[SC-Low] `periodAtTimestamp()` Returns Current Period Instead of Historical Period
* \#59928 \[SC-Low] Incorrect Period Calculation periodAtTimestamp() function
* \#59054 \[SC-Low] periodAtTimestamp() returns incorrect period number
* \#59559 \[SC-Low] Period calculation does not use provided `timestamp` in `periodAtTimestamp`
* \#59691 \[SC-Low] Missing \_disableInitializers() Allows Direct Implementation Initialization Leading to Vault Takeover
* \#59371 \[SC-Low] Avoid leaving a vault contract uninitialized.
* \#59728 \[SC-Low] Underflow issue leading to a `periodAtTimestamp` DOS
* \#59296 \[SC-Low] periodAtTimestamp(uint48 timestamp) ignores its parameter and always returns the current period
* \#59236 \[SC-Low] Implementation Contract Lacks Initializer Protection
* \#59335 \[SC-Low] `periodAtTimestamp` Function Returns Current Period Instead of Queried Period, Leading to Temporary Freezing of Funds
* \#59355 \[SC-Low] periodAtTimestamp ignores the input timestamp and returns the wrong period for non-current timestamps
* \#59852 \[SC-Low] Incorrect period calculation inside `periodAtTimestamp()`, resulting in returning period now instead of period at given timestamp
* \#59445 \[SC-Low] `periodAtTimestamp` does not work as expected
* \#59007 \[SC-Low] periodAtTimestamp() Returns Current Period Instead of Historical
* \#58992 \[SC-Low] The `FirelightVault` contract doesn't call `_disableInitializers` in its construcotor
* \#59369 \[SC-Low] The function `periodAtTimestamp` uses the current timestamp instead of provided timestamp causing incorrect period calculation
* \#59031 \[SC-Low] `periodAtTimestamp` returns incorrect period numbers for non-current timestamps
* \#59168 \[SC-Low] Incorrect time semantics in `periodAtTimestamp` cause off-chain miscalculations and data inconsistency
* \#59533 \[SC-Low] FirelightVault.sol#periodAtTimestamp() will return an incorrect period number due to flawed logic
* \#59298 \[SC-Low] function `periodAtTimestamp()` ignores the input timestamp, returning the current period instead
* \#59179 \[SC-Low] periodAtTimestamp() bug: returns current period for all timestamps
* \#59385 \[SC-Low] Timestamp Ignored, Current Block Time Used
* \#59235 \[SC-Low] Firelight Vault deviation from security best practice of locking down implementation logic
* \#59740 \[SC-Low] periodAtTimestamp provides period of current timestamp even for different timestamps
* \#59115 \[SC-Low] `periodAtTimestamp` function is incorrectly implemented and always returns period at current timestamp.

</details>

<details>

<summary>Insight</summary>

* \#59124 \[SC-Insight] Inefficient Loop Direction in `periodConfigurationAtTimestamp` Causes Unnecessary Gas Consumption
* \#59034 \[SC-Insight] `isLogAssets` parameter of the `_logTrace` function will always be set to true and can be removed.
* \#59027 \[SC-Insight] withdrawalsOf() view function does not account for already withdrawn funds
* \#59931 \[SC-Insight] Useless check
* \#59330 \[SC-Insight] RESCUER\_ROLE Not Assigned During Initialization
* \#59288 \[SC-Insight] Repeated Array Access in `rescueWithdrawFromBlocklisted` Loop Causes Unnecessary Gas Consumption

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#59124 \[SC-Insight] Inefficient Loop Direction in `periodConfigurationAtTimestamp` Causes Unnecessary Gas Consumption
* \#59091 \[SC-Low] \[LOW]: `FirelightVault.sol` Implementation Contract Does Not Disable Initializers
* \#59226 \[SC-Low] Logic Flaw in periodAtTimestamp Function Breaks Historical Queries, Returning Current Period Instead
* \#59023 \[SC-Low] Unprotected Implementation Contract Initializer Allows Unauthorized Admin Role Assignment Leading to Potential Governance Manipulation
* \#59100 \[SC-Low] periodAtTimestamp() Returns Current Period Instead of Queried Historical Period
* \#59334 \[SC-Low] `periodAtTimestamp` function uses current timestamp instead of input parameter, causing incorrect period calculation for historical or future queries
* \#59937 \[SC-Low] periodAtTimestamp() uses current time instead of input timestamp, returning wrong period numbers
* \#59034 \[SC-Insight] `isLogAssets` parameter of the `_logTrace` function will always be set to true and can be removed.
* \#59422 \[SC-Low] periodAtTimestamp ignores the supplied timestamp
* \#59467 \[SC-Low] periodAtTimestamp() Ignores Input Parameter
* \#59715 \[SC-Low] `periodAtTimestamp` will return different period for the same timestamp input
* \#59967 \[SC-Low] Broken Historical Period Calculation
* \#59879 \[SC-Low] Logic bug in `periodAtTimestamp`
* \#59605 \[SC-Low] Logic error in periodAtTimestamp returns incorrect periods
* \#59280 \[SC-Low] `periodAtTimestamp(uint48 timestamp)` ignores `timestamp` and return incorrect values when it is not `Time.timestamp()`
* \#58993 \[SC-Low] Incorrect timestamp calculation in `periodAtTimestamp()` leads to broken historical period lookups
* \#59635 \[SC-Low] Timestamp-agnostic periodAtTimestamp misreports historical periods, breaking time-locked logic
* \#59027 \[SC-Insight] withdrawalsOf() view function does not account for already withdrawn funds
* \#59820 \[SC-Low] `periodAtTimestamp()` Returns Current Period Instead of Historical Period
* \#59928 \[SC-Low] Incorrect Period Calculation periodAtTimestamp() function
* \#59054 \[SC-Low] periodAtTimestamp() returns incorrect period number
* \#59931 \[SC-Insight] Useless check
* \#59559 \[SC-Low] Period calculation does not use provided `timestamp` in `periodAtTimestamp`
* \#59691 \[SC-Low] Missing \_disableInitializers() Allows Direct Implementation Initialization Leading to Vault Takeover
* \#59371 \[SC-Low] Avoid leaving a vault contract uninitialized.
* \#59728 \[SC-Low] Underflow issue leading to a `periodAtTimestamp` DOS
* \#59296 \[SC-Low] periodAtTimestamp(uint48 timestamp) ignores its parameter and always returns the current period
* \#59236 \[SC-Low] Implementation Contract Lacks Initializer Protection
* \#59335 \[SC-Low] `periodAtTimestamp` Function Returns Current Period Instead of Queried Period, Leading to Temporary Freezing of Funds
* \#59355 \[SC-Low] periodAtTimestamp ignores the input timestamp and returns the wrong period for non-current timestamps
* \#59852 \[SC-Low] Incorrect period calculation inside `periodAtTimestamp()`, resulting in returning period now instead of period at given timestamp
* \#59445 \[SC-Low] `periodAtTimestamp` does not work as expected
* \#59007 \[SC-Low] periodAtTimestamp() Returns Current Period Instead of Historical
* \#58992 \[SC-Low] The `FirelightVault` contract doesn't call `_disableInitializers` in its construcotor
* \#59369 \[SC-Low] The function `periodAtTimestamp` uses the current timestamp instead of provided timestamp causing incorrect period calculation
* \#59031 \[SC-Low] `periodAtTimestamp` returns incorrect period numbers for non-current timestamps
* \#59168 \[SC-Low] Incorrect time semantics in `periodAtTimestamp` cause off-chain miscalculations and data inconsistency
* \#59533 \[SC-Low] FirelightVault.sol#periodAtTimestamp() will return an incorrect period number due to flawed logic
* \#59298 \[SC-Low] function `periodAtTimestamp()` ignores the input timestamp, returning the current period instead
* \#59179 \[SC-Low] periodAtTimestamp() bug: returns current period for all timestamps
* \#59385 \[SC-Low] Timestamp Ignored, Current Block Time Used
* \#59235 \[SC-Low] Firelight Vault deviation from security best practice of locking down implementation logic
* \#59330 \[SC-Insight] RESCUER\_ROLE Not Assigned During Initialization
* \#59288 \[SC-Insight] Repeated Array Access in `rescueWithdrawFromBlocklisted` Loop Causes Unnecessary Gas Consumption
* \#59740 \[SC-Low] periodAtTimestamp provides period of current timestamp even for different timestamps
* \#59115 \[SC-Low] `periodAtTimestamp` function is incorrectly implemented and always returns period at current timestamp.

</details>
