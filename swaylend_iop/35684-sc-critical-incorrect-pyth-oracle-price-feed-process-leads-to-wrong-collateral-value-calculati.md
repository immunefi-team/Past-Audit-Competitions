# #35684 \[SC-Critical] Incorrect Pyth Oracle Price Feed Process Leads to Wrong Collateral Value Calcul

**Submitted on Oct 3rd 2024 at 09:29:18 UTC by @ret2happy for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35684
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Protocol insolvency

## Description

## For Triaggers

We enjoy sharing professional and detailed reports with runnable proof of concepts, but this bug can be undeniably confirmed by just reading a specific code snippet from the codebase.

Investing hours developing what may be interpreted as a "proof of concept" for a bug of this nature makes no sense. We expect the good judgment of the reviewers to agree with us after reading the report, but if not, let us know and we will code what you may consider a valid proof of concept for this report.

## Brief/Intro

\`market\` contract simply uses the \`price\` and \`exp\` field from the \`get\_price\_internal\` function, without taking the \`confidence\` into the calculation, leading to the loan and collateral value mis-calculated.

## Vulnerability Details

In the \`get\_price\_internal\` function \[1], which is used by the calculation of value of the collateral, it checks whether the confidence is in a reasonable range. However, such confidence is only used in the value check, rather in the value calculation. Due to the violate market value, certain asset‘s value could reach the minimum threshold as \`price - confidence\`. However, all usage of the \`get\_price\_internal\` doesn't takes \`confidence\` of the \`price into the calculation. This is violated with the practical usage mentioned in the Pyth document and will cause the collateral values undercollateralized (since the price of the collateral is over-estimated, i.e., the actual price can be lower).

For example, in \`is\_borrow\_collateralized\` function \[2]: \`\`\` // Checks that the dollar value of the user's collateral multiplied by borrow\_collateral\_factor is greater than the (planned) loan amount. #\[storage(read)] fn is\_borrow\_collateralized(account: Identity) -> bool { let principal = storage.user\_basic.get(account).try\_read().unwrap\_or(UserBasic::default()).principal; // decimals: base\_asset\_decimal if principal >= I256::zero() { return true };

```
let present &#x3D; present_value(principal); // decimals: base_token_decimals
let mut borrow_limit: u256 &#x3D; 0;

let mut index &#x3D; 0;
let len &#x3D; storage.collateral_configurations_keys.len();

while index &lt; len {
    let collateral_configuration &#x3D; storage.collateral_configurations.get(storage.collateral_configurations_keys.get(index).unwrap().read()).read();

    let balance: u256 &#x3D; storage.user_collateral.get((account, collateral_configuration.asset_id)).try_read().unwrap_or(0).into(); // decimals: collateral_configuration.decimals
    let price &#x3D; get_price_internal(collateral_configuration.price_feed_id); // decimals: price.exponent decimals
    let price_scale &#x3D; u256::from(10_u64).pow(price.exponent);
    let price &#x3D; u256::from(price.price); // decimals: price.exponent
    let collateral_scale &#x3D; u256::from(10_u64).pow(collateral_configuration.decimals);
    let base_scale &#x3D; u256::from(10_u64).pow(storage.market_configuration.read().base_token_decimals);

    let amount &#x3D; balance * price / price_scale; // decimals:  collateral_configuration.decimals  // [AUDIT]: the price is not take confidence into calculation. The actual collateral price should be &#x60;price * exp - confidence&#x60; to make a conservative price calculation.
    borrow_limit +&#x3D; amount * collateral_configuration.borrow_collateral_factor * base_scale / FACTOR_SCALE_18 / collateral_scale; // decimals: base_token_decimals
    index +&#x3D; 1;
}
```

\`\`\`

The \`amount\` is directly calculated by \`amount = balance \* price / price\_scale;\`. According to the Pyth document \[3], we have the following notice:

\> To expand upon the first option, we recommend using the confidence interval to protect your users from these unusual market conditions. The simplest way to do so is to use Pyth's confidence interval to compute a range in which the true price probably lies. This principle is common sense. Imagine that you are lending money to a friend, and your friend pledges a bitcoin as collateral. Also imagine that Pyth says the bitcoin price is $50000 +- $1000. (Note that $1000 is an unusually large confidence interval for bitcoin; the confidence interval is typically $50 dollars). You therefore calculate that the true price is between $49000 and $51000. When originating the loan, you would value the bitcoin at $49000. The lower price is conservative in this instance because it limits the amount of borrowing that is possible while the price is uncertain. On the other hand, if you were to issue a loan of bitcoin, you would value the borrowed bitcoin at $51000. The higher price is conservative, as it protects you from allowing someone to borrow in excess during times of increased volatility.

This means if the price for BTC is \`$50000\` and the confidence interval is \`$1000\`, we should use \`$50000 +- $1000\` for the collateral calculation. Although the \`get\_price\_internal\` function already checks that the \`confidence\` should be within 1% width of the \`price\`, that is still too large for collateral lending. One would have a differences of 2% (1% + 1%) when he supply BTC in the lower confidence price (-1%) and borrow another token in higher confidence price (+1%). 2% is enough to break the health factor of many user and the platform.

## Impact Details

Miscalculation of the collateral token value would make the protocol insolvent.

## References

\[1] <https://github.com/Swaylend/swaylend-monorepo/blob/fba606f6fbf2fd3a6e99758d55c2059ce3f3064c/contracts/market/src/main.sw#L985-L1019> \[2] <https://github.com/Swaylend/swaylend-monorepo/blob/fba606f6fbf2fd3a6e99758d55c2059ce3f3064c/contracts/market/src/main.sw#L1294> \[3] <https://docs.pyth.network/price-feeds/best-practices>

## Proof of Concept

## Proof of Concept

As aforementioned, this bug can be undeniably confirmed by just reading a specific code snippet from the codebase. All \`get\_price\_internal\` function usage in \`swaylend-monorepo/contracts/market/src/main.sw\` doesn't take \`confidence\` into the calculation. 1% width check for the confidence is too wide for the lending protocol.
