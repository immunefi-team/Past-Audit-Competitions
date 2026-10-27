# #35767 \[SC-Critical] constanct value is used to check \`price.confidence\`

**Submitted on Oct 7th 2024 at 09:07:06 UTC by @jasonxiale for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35767
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

In \[Market.get\_price\_internal] (<https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L1017-L1050>), while validating the Pyth's price, the function checks the \`price.confidence\` in [main.sw#L1045-L1048](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L1045-L1048).

The issue is that \`get\_price\_internal\` will be used to get price for **different \`price\_fee\_id\`**, but the returned price will be checked against the same **constant value**

## Vulnerability Details

As shown in [main.sw#L1048-L1051](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L1045-L1048), [Market.get\_price\_internal](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L1017-L1050) uses constant values to check \`price.confidence\`

\`\`\`Rust 1017 #\[storage(read)] 1018 fn get\_price\_internal(price\_feed\_id: PriceFeedId) -> Price { 1019 let contract\_id = storage.pyth\_contract\_id.read(); 1020 require( 1021 contract\_id != ContractId::zero(), 1022 Error::OracleContractIdNotSet, 1023 ); 1024 ... 1045 require( 1046 u256::from(price.confidence) <= (u256::from(price.price) \* ORACLE\_MAX\_CONF\_WIDTH / ORACLE\_CONF\_BASIS\_POINTS), 1047 Error::OraclePriceValidationError, 1048 ); <<<--- constant value is checked here 1049 1050 price 1051 } \`\`\`

## Impact Details

\`Market.get\_price\_internal\` is important, because it is used while borrow/liquidate assets:

1. if the constant value is too wide for some price-feeds, the user/protocol might execute the tx at a bad price
2. if the constant value is too restrict for some price-feeds, the tx might always revert

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

According to the technical walkthrough at [185s](https://www.youtube.com/watch?v=cNsOq38W6Jg\&t=185s), the derivate BTC might be used,

And from [Pyth's network](https://www.pyth.network/price-feeds), we will take \`BTC/USD\` and \`TBTC/USD\` pair as an example

At the moment: >BTC/USD: price $63620, confidence: +- $16

\>CBBTC/USD: price $63610, confidence: +- $509

To prove the point, we'll use the following code to simulate the check in [main.sw#L1045-L1048](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L1045-L1048)

Please put the following code into a new file named \`check.rs\` and run: \`\`\`bash root\@4d98affa1474:/in/temp# rustc check.rs ; ./check BTC check: true CBBTC check: false \`\`\`

As we can see, BTC/USD can pass the check, and CBBTC/USD can't pass the check

\`\`\`Rust pub const ORACLE\_MAX\_CONF\_WIDTH: u64 = 20; pub const ORACLE\_CONF\_BASIS\_POINTS: u64 = 10\_000; fn main() { let btc\_price : u64 = 63620; let btc\_confidence: u64 = 16;

```
let cbbtc_price     : u64 &#x3D; 63610;
let cbbtc_confidence: u64 &#x3D; 509;

println!(&quot;BTC check: {}&quot;, btc_confidence &lt;&#x3D; btc_price * ORACLE_MAX_CONF_WIDTH / ORACLE_CONF_BASIS_POINTS);
println!(&quot;CBBTC check: {}&quot;, cbbtc_confidence &lt;&#x3D; cbbtc_price * ORACLE_MAX_CONF_WIDTH / ORACLE_CONF_BASIS_POINTS);
```

} \`\`\`
