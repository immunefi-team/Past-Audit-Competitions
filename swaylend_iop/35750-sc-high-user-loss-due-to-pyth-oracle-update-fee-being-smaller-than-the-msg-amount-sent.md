# #35750 \[SC-High] User loss due to Pyth oracle update fee being smaller than the msg amount sent

**Submitted on Oct 6th 2024 at 13:02:39 UTC by @SimaoAmaro for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35750
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The Swaylend market allows users to send price updates directly when interacting with several functions.

The Pyth oracle requires paying a fee whenever paying prices, but it does not always charge the same fee and it may not even charge any fee.

However, the market contract does not deal with the 2 scenarios above and will lead to user loss of funds.

## Vulnerability Details

The \`market::update\_price\_feeds\_if\_necessary\_internal()\` function forwards the \`msg\_amount\` sent in the the \`price\_data\_update\` argument, without confirming if this fee is the right one. Then, it forwards the fee to the pyth oracle, which might be more than necessary \[1].

However, there are 2 situations that might lead to loss of funds for the user.

1. In case the user is frontrun by an update of the pyth price, the latest submitted publish time of the oracle will be more recent than the update data the user sent as argument to the market function. When this happens, the pyth oracle ignores the update data and does not enforce any payment, leading to loss of funds for the users that could have been saved. \[2]
2. In case the pyth oracle updates the fee and makes it smaller and frontruns the user. This means the user will send more fee than it should and it will never revert.

## Impact Details

Users will overpay for pyth oracle fees not getting these funds back, incurring loss of funds.

## References

\[1] <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw?utm\\_source=immunefi#L1067> \[2] <https://github.com/FuelLabs/Pyth-integration/blob/master/pyth-contract/src/main.sw#L273>

## Proof of Concept

## Proof of Concept

Apply the following diffs to confirm that it may overcharge fees. \`\`\`diff diff --git a/contracts/market/tests/local\_tests/main\_test\_uni\_no\_debug\_mode.rs b/contracts/market/tests/local\_tests/main\_test\_uni\_no\_debug\_mode.r s index 6976dc8..9b4ff8b 100644 --- a/contracts/market/tests/local\_tests/main\_test\_uni\_no\_debug\_mode.rs +++ b/contracts/market/tests/local\_tests/main\_test\_uni\_no\_debug\_mode.rs @@ -3,7 +3,7 @@ use chrono::Utc; use fuels::prelude::ViewOnlyAccount; use fuels::programs::calls::{CallHandler, CallParameters}; use fuels::programs::responses::CallResponse; -use fuels::types::transaction::TxPolicies; +use fuels::types::{transaction::TxPolicies, AssetId}; use fuels::types::transaction\_builders::VariableOutputPolicy; use market::PriceDataUpdate; use market\_sdk::{convert\_i256\_to\_u64, is\_i256\_negative, parse\_units}; @@ -575,6 +575,8 @@ async fn main\_test\_no\_debug() { let log\_amount = format!("{} UNI", amount as f64 / scale\_9); print\_case\_title(12, "Chad", "withdraw\_collateral", log\_amount.as\_str());

* let base\_balance\_before = chad.get\_asset\_balance(\&AssetId::zeroed()).await.unwrap();
* // Chad calls withdraw\_collateral market .with\_account(\&chad) @@ -588,6 +590,8 @@ async fn main\_test\_no\_debug() { ) .await .unwrap();
*
* assert!(chad.get\_asset\_balance(\&AssetId::zeroed()).await.unwrap() + 102 == base\_balance\_before);

diff --git a/libs/market\_sdk/src/market\_utils.rs b/libs/market\_sdk/src/market\_utils.rs index a9f8592..0433364 100644 --- a/libs/market\_sdk/src/market\_utils.rs +++ b/libs/market\_sdk/src/market\_utils.rs @@ -274,7 +274,7 @@ impl Market { price\_data\_update: \&PriceDataUpdate, ) -> anyhow::Result\<CallResponse<()>> { let tx\_policies = TxPolicies::default().with\_script\_gas\_limit(DEFAULT\_GAS\_LIMIT);

* ```
     let call_params &#x3D; CallParameters::default().with_amount(price_data_update.update_fee);
  ```
* ```
     let call_params &#x3D; CallParameters::default().with_amount(price_data_update.update_fee + 100);
  ```

\`\`\`
