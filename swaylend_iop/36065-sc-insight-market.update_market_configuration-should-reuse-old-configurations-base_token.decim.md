# #36065 \[SC-Insight] \`Market.update\_market\_configuration\` should reuse old configuration's \`base\_toke

**Submitted on Oct 17th 2024 at 16:05:55 UTC by @jasonxiale for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #36065
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/main/contracts/market/src/main.sw>
* **Impacts:**

## Description

## Brief/Intro

In [Market.update\_market\_configuration](https://github.com/Swaylend/swaylend-monorepo/blob/569fb4b2ccee8a4e089167c65cde8184b04c61c8/contracts/market/src/main.sw#L973-L990), while updating the \`MarketConfiguration\`, orignal \`configuration.base\_token\` can't be changed in [main.sw#L980](https://github.com/Swaylend/swaylend-monorepo/blob/569fb4b2ccee8a4e089167c65cde8184b04c61c8/contracts/market/src/main.sw#L980), \`MarketConfiguration.base\_token\_decimals\` shouldn't be changed too.

## Vulnerability Details

\`\`\`Rust 971 // # 11. Changing market configuration 972 #\[storage(write)] 973 fn update\_market\_configuration(configuration: MarketConfiguration) { 974 // Only owner can update the market configuration 975 only\_owner(); 976 977 let mut configuration = configuration; 978 979 // Cannot change base token and tracking index scale 980 configuration.base\_token = storage.market\_configuration.read().base\_token; <<<--- \`MarketConfiguration.base\_token\_decimals\` should be reused here too 981 configuration.base\_tracking\_index\_scale = storage.market\_configuration.read().base\_tracking\_index\_scale; 982 983 // Update the market configuration 984 storage.market\_configuration.write(configuration); 985 986 // Emit market configuration event\
987 log(MarketConfigurationEvent { 988 market\_config: configuration, 989 }); 990 } \`\`\`

## Impact Details

To avoid the mistake

## References

Add any relevant links to documentation or code

## Link to Proof of Concept

<https://gist.github.com/crazy4linux/382b80b244a219346a827755a3ea594e>

## Proof of Concept

## Proof of Concept
