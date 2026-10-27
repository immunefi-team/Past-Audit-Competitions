# #35999 \[SC-Insight] Incorrect event name

**Submitted on Oct 15th 2024 at 13:58:05 UTC by @jasonxiale for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35999
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

While \`Market.add\_collateral\_asset\` is called and configuration.asset\_id has already been in the system, an event named \`Error::UnknownAsset\` is emitted in [main.sw#L148-L155](https://github.com/Swaylend/swaylend-monorepo/blob/569fb4b2ccee8a4e089167c65cde8184b04c61c8/contracts/market/src/main.sw#L148-L155), however this is incorrect. An event named \`Error::AssetAlreadyExist\` should be more proper

Especially this should be different from \`Error::UnknownAsset\` in [main.sw#L224](https://github.com/Swaylend/swaylend-monorepo/blob/569fb4b2ccee8a4e089167c65cde8184b04c61c8/contracts/market/src/main.sw#L224), because \`Error::UnknownAsset\` in Market.update\_collateral\_asset is used as the asset doesn't exist.

## Vulnerability Details

\`\`\`Rust 171 fn add\_collateral\_asset(configuration: CollateralConfiguration) { 172 // Only owner can add new collateral asset 173 only\_owner(); 174 175 // Check if asset already exists 176 require( 177 storage 178 .collateral\_configurations 179 .get(configuration.asset\_id) 180 .try\_read() 181 .is\_none(), 182 Error::UnknownAsset, 183 ); 184 185 storage 186 .collateral\_configurations 187 .insert(configuration.asset\_id, configuration); 188 storage 189 .collateral\_configurations\_keys 190 .push(configuration.asset\_id); 191 192 log(CollateralAssetAdded { 193 asset\_id: configuration.asset\_id, 194 configuration, 195 }); 196 } \`\`\`

## Impact Details

incorrect event name

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

The following code will emit the event \`\`\`Rust 171 fn add\_collateral\_asset(configuration: CollateralConfiguration) { 172 // Only owner can add new collateral asset 173 only\_owner(); 174 175 // Check if asset already exists 176 require( 177 storage 178 .collateral\_configurations 179 .get(configuration.asset\_id) 180 .try\_read() 181 .is\_none(), 182 Error::UnknownAsset, 183 ); 184 185 storage 186 .collateral\_configurations 187 .insert(configuration.asset\_id, configuration); 188 storage 189 .collateral\_configurations\_keys 190 .push(configuration.asset\_id); 191 192 log(CollateralAssetAdded { 193 asset\_id: configuration.asset\_id, 194 configuration, 195 }); 196 } \`\`\`
