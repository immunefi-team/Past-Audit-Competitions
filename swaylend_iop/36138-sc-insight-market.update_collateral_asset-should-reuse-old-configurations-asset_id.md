# #36138 \[SC-Insight] \`Market.update\_collateral\_asset\` should reuse old configuration's \`asset\_id\`

**Submitted on Oct 21st 2024 at 15:32:58 UTC by @jasonxiale for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #36138
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In [Market.update\_market\_configuration](https://github.com/Swaylend/swaylend-monorepo/blob/41b1329983c0b58db6f81e7ecd05a82be03038bd/contracts/market/src/main.sw#L1299-L1317), while updating market configuration, the old \`base\_token\` is reused in [main.sw#L1307](https://github.com/Swaylend/swaylend-monorepo/blob/41b1329983c0b58db6f81e7ecd05a82be03038bd/contracts/market/src/main.sw#L1307)

The same rule should apply to [Market.update\_collateral\_asset](https://github.com/Swaylend/swaylend-monorepo/blob/41b1329983c0b58db6f81e7ecd05a82be03038bd/contracts/market/src/main.sw#L262-L285), while updating CollateralConfiguration, \`CollateralConfiguration.asset\_id\` should be reused.

## Vulnerability Details

As the following code shows, while updating CollateralConfiguration, the function doesn't reuse old \`CollateralConfiguration.asset\_id\` \`\`\`Rust 262 #\[storage(write)] 263 fn update\_collateral\_asset(asset\_id: AssetId, configuration: CollateralConfiguration) { 264 // Only owner can update collateral asset 265 only\_owner(); 266 267 // Check if asset exists 268 require( 269 storage 270 .collateral\_configurations 271 .get(asset\_id) 272 .try\_read() 273 .is\_some(), 274 Error::UnknownAsset, 275 ); 276 277 storage 278 .collateral\_configurations 279 .insert(asset\_id, configuration); 280 281 log(CollateralAssetUpdated { 282 asset\_id, 283 configuration, 284 }); 285 } \`\`\`

## Impact Details

avoid mistake

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

As the following code shows, while updating CollateralConfiguration, the function doesn't reuse old \`CollateralConfiguration.asset\_id\` \`\`\`Rust 262 #\[storage(write)] 263 fn update\_collateral\_asset(asset\_id: AssetId, configuration: CollateralConfiguration) { 264 // Only owner can update collateral asset 265 only\_owner(); 266 267 // Check if asset exists 268 require( 269 storage 270 .collateral\_configurations 271 .get(asset\_id) 272 .try\_read() 273 .is\_some(), 274 Error::UnknownAsset, 275 ); 276 277 storage 278 .collateral\_configurations 279 .insert(asset\_id, configuration); 280 281 log(CollateralAssetUpdated { 282 asset\_id, 283 configuration, 284 }); 285 } \`\`\`
