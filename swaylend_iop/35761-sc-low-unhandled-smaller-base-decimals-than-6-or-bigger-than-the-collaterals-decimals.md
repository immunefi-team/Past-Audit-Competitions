# #35761 \[SC-Low] Unhandled smaller base decimals than 6 or bigger than the collateral's decimals

**Submitted on Oct 6th 2024 at 23:28:34 UTC by @SimaoAmaro for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35761
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The \`market\` converts between decimals when going from collateral to base or vice-versa (and price).

There are instances when depending on the decimal values of the collateral, base and price, it could lead to the \`market\` reverting to underflow or division by 0.

## Vulnerability Details

\`market::collateral\_value\_to\_sell()\` calculates the \`scale\` to go from collateral decimals to base by doing \`\`\`rust let scale = u256::from(10\_u64).pow( collateral\_configuration .decimals - storage .market\_configuration .read() .base\_token\_decimals, ); \`\`\` If the base token decimals are bigger than the collateral's decimals, this reverts.

In \`market::available\_to\_borrow()\`, the scale is calculated as \`\`\`rust let scale = u256::from(10\_u64).pow( collateral\_configuration .decimals + price\_exponent - storage .market\_configuration .read() .base\_token\_decimals, ); \`\`\` which also underflows whenever the base token decimals is bigger than the collateral's + the price exponent.

Lastly, \`market::update\_base\_principal()\` gets the \`accrual\_descale\_factor\` by doing \`u256::from(10\_u64).pow(market\_configuration.base\_token\_decimals) / BASE\_ACCRUAL\_SCALE;\`, which is 0 whenever the base token decimals is smaller than 6. This will make it revert and the market becomes unusable (funds can not be supplied so no funds are stuck but it needs to be redeployed).

## Impact Details

View functions (\`market::collateral\_value\_to\_sell()\` and \`market::available\_to\_borrow()\` will not work or the contract has to be redeployed due to this issue.

## References

<https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw?utm\\_source=immunefi#L716> <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw?utm\\_source=immunefi#L568> <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw?utm\\_source=immunefi#L1489-L1499>

## Proof of Concept

## Proof of Concept

Change the decimals of usdc to 5 in tokens.json and run \`cargo test --release main\_test\_no\_debug -- --nocapture\`. The rest fails on step 0 when supplying base due to division by 0.
