# #35732 \[SC-Low] Withdrawals can not be paused which could lead to protocol insolvency in case of iss

**Submitted on Oct 5th 2024 at 15:48:17 UTC by @SimaoAmaro for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35732
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The \`PauseConfiguration\` has the \`withdraw\_paused\` flag, allowing the protocol to pause withdrawals. However, the market's \`withdraw\_collateral()\` \[function}(<https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L305>) does not check if it is paused, leading to users always being able to withdraw funds and the protocol's insolvency depending on the reason the withdrawal is paused.

## Vulnerability Details

The \`withdraw\_collateral()\` never checks that withdrawals are paused as mentioned above. However, [withdraw\_base()](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L456) or [CompoundV3::withdrawInternal()](https://github.com/compound-finance/comet/blob/main/contracts/Comet.sol#L1087) (Swaylend is a compound v3 fork) do check if withdrawals are paused.

## Impact Details

Protocol funds are at risk due to the inability of the protocol to pause withdrawals when necessary.

## References

<https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L305> <https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L456> <https://github.com/compound-finance/comet/blob/main/contracts/Comet.sol#L1087>

## Proof of Concept

## Proof of Concept

Modify the test file \`main\_test\_uni\_no\_debug\_mode.rs\` as following. The test still passes, when it should not as withdrawals are paused. \`\`\`diff diff --git a/contracts/market/tests/local\_tests/main\_test\_uni\_no\_debug\_mode.rs b/contracts/market/tests/local\_tests/main\_test\_uni\_no\_debug\_mode.rs index 6976dc8..9bcdfa1 100644 --- a/contracts/market/tests/local\_tests/main\_test\_uni\_no\_debug\_mode.rs +++ b/contracts/market/tests/local\_tests/main\_test\_uni\_no\_debug\_mode.rs @@ -5,7 +5,7 @@ use fuels::programs::calls::{CallHandler, CallParameters}; use fuels::programs::responses::CallResponse; use fuels::types::transaction::TxPolicies; use fuels::types::transaction\_builders::VariableOutputPolicy; -use market::PriceDataUpdate; +use market::{PriceDataUpdate, PauseConfiguration}; use market\_sdk::{convert\_i256\_to\_u64, is\_i256\_negative, parse\_units};

// Multiplies all values by this number @@ -18,6 +18,7 @@ async fn main\_test\_no\_debug() { let scale\_9 = 10u64.pow(9) as f64; let TestData { wallets,

* ```
     admin,
     alice,
     alice_account,
     bob,
  ```

@@ -567,6 +568,20 @@ async fn main\_test\_no\_debug() { // 🤙 Call: withdraw\_collateral // 💰 Amount: 270 UNI

* let pause\_config = PauseConfiguration {
* ```
     supply_paused: true,
  ```
* ```
     withdraw_paused: true,
  ```
* ```
     absorb_paused: true,
  ```
* ```
     buy_paused: true,
  ```
* };
*
* let admin\_pause\_collat\_res = market
* ```
     .with_account(&amp;admin)
  ```
* ```
     .await
  ```
* ```
     .unwrap()
  ```
* ```
     .pause(&amp;pause_config)
  ```
* ```
     .await;
  ```
* let amount = market .get\_user\_collateral(chad\_account, uni.asset\_id) .await \`\`\`
