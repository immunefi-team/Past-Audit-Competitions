# Folks Smart Contract Library | Aud Comp

## Reports by Severity

<details>

<summary>Low</summary>

* \#49075 \[SC-Low] \`SetLib.remove\_item()\` is not safe on empty Dynamic arrays
* \#49559 \[SC-Low] The remove functionality in \`UInt64SetLib::remove\_item\` underflows on empty array
* \#48983 \[SC-Low] Potential Underflow in remove\_item() on Empty Array
* \#48990 \[SC-Low] Integer underflow in remove\_item leads to AVM trap and DoS via empty array call
* \#48894 \[SC-Low] Underflow in index calculation fails remove\_item
* \#49003 \[SC-Low] Array Underflow Vulnerability in UInt64SetLib leads to contract failure
* \#49687 \[SC-Low] An underflow in \`remove\_item\` function in \`Uint64SetLib\` Contract.
* \#48885 \[SC-Low] No items length check in remove\_item leads to a revert with an underflow
* \#48998 \[SC-Low] Critical UInt64 underflow in set removal causes permanent denial of service
* \#49938 \[SC-Low] Underflow Revert in \`remove\_item\` When Removing from an Empty Array
* \#49390 \[SC-Low] \`UInt64SetLib#remove\_item\` would revert if the item is empty
* \#49527 \[SC-Low] Invariant breakage , Edge case Integer UInt64SetLib.py::remove\_item leads to int underflow
* \#49690 \[SC-Low] Integer Underflow in UInt64SetLib.py

</details>

<details>

<summary>Insight</summary>

* \#49409 \[SC-Insight] Incorrect comment in UInt64SetLib
* \#49250 \[SC-Insight] \`AccessControl\`: unnecessary box usage in \`\_grant\_role\`
* \#48747 \[SC-Insight] Consider emitting BucketConsumed for infinite buckets in RateLimiter
* \#49970 \[SC-Insight] Malicious upgradable admin can permanently brick contract upgradeability
* \#49553 \[SC-Insight] program\_sha256\`, Reducing Auditability and Monitoring Efficiency
* \#48717 \[SC-Insight] RateLimiter current capacity can be permanently held at zero
* \#48804 \[SC-Insight] TBD
* \#49061 \[SC-Insight] The Function \`get\_current\_capacity()\` returns wrong value when bucket is infinite
* \#49051 \[SC-Insight] Improving and expanding documentation to reduce risks
* \#49437 \[SC-Insight] \`RateLimiter\`: incorrect infinite -> finite bucket transition
* \#49413 \[SC-Insight] discrepancy between document and codebase
* \#48718 \[SC-Insight] Contract Upgrade Completion DoS/Takeover Risk

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#49075 \[SC-Low] \`SetLib.remove\_item()\` is not safe on empty Dynamic arrays
* \#49559 \[SC-Low] The remove functionality in \`UInt64SetLib::remove\_item\` underflows on empty array
* \#49409 \[SC-Insight] Incorrect comment in UInt64SetLib
* \#48983 \[SC-Low] Potential Underflow in remove\_item() on Empty Array
* \#48990 \[SC-Low] Integer underflow in remove\_item leads to AVM trap and DoS via empty array call
* \#49250 \[SC-Insight] \`AccessControl\`: unnecessary box usage in \`\_grant\_role\`
* \#48894 \[SC-Low] Underflow in index calculation fails remove\_item
* \#49003 \[SC-Low] Array Underflow Vulnerability in UInt64SetLib leads to contract failure
* \#48747 \[SC-Insight] Consider emitting BucketConsumed for infinite buckets in RateLimiter
* \#49970 \[SC-Insight] Malicious upgradable admin can permanently brick contract upgradeability
* \#49553 \[SC-Insight] program\_sha256\`, Reducing Auditability and Monitoring Efficiency
* \#48717 \[SC-Insight] RateLimiter current capacity can be permanently held at zero
* \#49687 \[SC-Low] An underflow in \`remove\_item\` function in \`Uint64SetLib\` Contract.
* \#48885 \[SC-Low] No items length check in remove\_item leads to a revert with an underflow
* \#48998 \[SC-Low] Critical UInt64 underflow in set removal causes permanent denial of service
* \#48804 \[SC-Insight] TBD
* \#49061 \[SC-Insight] The Function \`get\_current\_capacity()\` returns wrong value when bucket is infinite
* \#49938 \[SC-Low] Underflow Revert in \`remove\_item\` When Removing from an Empty Array
* \#49051 \[SC-Insight] Improving and expanding documentation to reduce risks
* \#49437 \[SC-Insight] \`RateLimiter\`: incorrect infinite -> finite bucket transition
* \#49413 \[SC-Insight] discrepancy between document and codebase
* \#49390 \[SC-Low] \`UInt64SetLib#remove\_item\` would revert if the item is empty
* \#49527 \[SC-Low] Invariant breakage , Edge case Integer UInt64SetLib.py::remove\_item leads to int underflow
* \#48718 \[SC-Insight] Contract Upgrade Completion DoS/Takeover Risk
* \#49690 \[SC-Low] Integer Underflow in UInt64SetLib.py

</details>
