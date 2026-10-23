# #35760 \[SC-Low] \`market::available\_to\_borrow()\` compares the collateral in USD against the borrow in

**Submitted on Oct 6th 2024 at 23:12:49 UTC by @SimaoAmaro for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35760
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

\`market::available\_to\_borrow()\` returns the available amount to borrow by comparing the collateral amount in USD with the already borrowed amount.

However, the borrowed amount is not multiplied by the base price to get the value in USD, similarly to the \`market::is\_borrow\_collateralized()\`, so it ends up comparing a collateral amount in USD with a borrow amount in base units.

This leads to an over/underestimation of the amount available to borrow if the base price is bigger/smaller than 1 USD, respectively.

## Vulnerability Details

\`market::available\_to\_borrow()\` gets the collateral amounts and multiplies each by the price, getting USD units. The borrowed amount is fetched from market::get\_user\_supply\_borrow\_internal()\`, which returns an amount in base units, not USD.

Thus, in the end it compares \`borrow\_limit\` in USD with \`borrow\` in base units.

## Impact Details

Whenever the base asset price is different than 1 USD, the available base to borrow will return more or less than it should.

## References

<https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw?utm\\_source=immunefi#L544> <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw?utm\\_source=immunefi#L574> <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw?utm\\_source=immunefi#L578-L583>

## Proof of Concept

## Proof of Concept

Add the following diffs and run \`cargo test main\_test\_no\_debug --release -- --nocapture\`. The base asset (USDC) was modified to be worth 2 USD. Alice supplies 40 UNI, being worth 200 USD and is able to borrow initially 100 USD (50 USDC, 2 USD each, the collateral factor is 0.5). Then, Alice withdraws 50 USDC, which is 100 USD. When available to borrow is called again, it returns that alice can borrow 50 USD: however, she has already borrowed 100 USD, the maximum, so it should return 0 instead. \`\`\`diff diff --git a/contracts/market/tests/tokens.json b/contracts/market/tests/tokens.json index 1b581b0..ba04e2f 100644 --- a/contracts/market/tests/tokens.json +++ b/contracts/market/tests/tokens.json @@ -5,7 +5,7 @@ "price\_feed\_decimals": 7, "name": "USD Coin", "symbol": "USDC",

* "default\_price": 1,
* "default\_price": 2, "decimals": 6, "mint\_amount": 10000 },

diff --git a/contracts/market/tests/local\_tests/main\_test\_uni\_no\_debug\_mode.rs b/contracts/market/tests/local\_tests/main\_test\_uni\_no\_debug\_mode.rs index 6976dc8..54ecf23 100644 --- a/contracts/market/tests/local\_tests/main\_test\_uni\_no\_debug\_mode.rs +++ b/contracts/market/tests/local\_tests/main\_test\_uni\_no\_debug\_mode.rs @@ -129,6 +129,10 @@ async fn main\_test\_no\_debug() { let log\_amount = format!("{} USDC", amount as f64 / scale\_6); print\_case\_title(2, "Alice", "withdraw\_base", log\_amount.as\_str());

* let available\_to\_borrow = market.available\_to\_borrow(&\[\&oracle.instance], alice\_account).await.unwrap();
* let log\_amount = format!("{} USDC", available\_to\_borrow as f64 / scale\_6);
* print\_case\_title(3, "Alice", "available to borrow initial", log\_amount.as\_str());
* // Alice calls withdraw\_base market .with\_account(\&alice) @@ -142,6 +146,10 @@ async fn main\_test\_no\_debug() { let balance = alice.get\_asset\_balance(\&usdc.asset\_id).await.unwrap(); assert!(balance == amount);
* let available\_to\_borrow = market.available\_to\_borrow(&\[\&oracle.instance], alice\_account).await.unwrap();
* let log\_amount = format!("{} USDC", available\_to\_borrow as f64 / scale\_6);
* print\_case\_title(4, "Alice", "available to borrow final", log\_amount.as\_str());
* market .print\_debug\_state(\&wallets, \&usdc, \&uni) \`\`.
