# #36158 \[SC-Low] \`Market.collateral\_value\_to\_sell\` will always revert if collateral\_configuration

**Submitted on Oct 22nd 2024 at 08:33:46 UTC by @jasonxiale for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #36158
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/market/src/main.sw>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In [Market.collateral\_value\_to\_sell](https://github.com/Swaylend/swaylend-monorepo/blob/41b1329983c0b58db6f81e7ecd05a82be03038bd/contracts/market/src/main.sw#L879-L906), while the function tries to calculate the scale in [main.sw#L894-L900](https://github.com/Swaylend/swaylend-monorepo/blob/41b1329983c0b58db6f81e7ecd05a82be03038bd/contracts/market/src/main.sw#L894-L900), \`collateral\_configuration.decimals - storage.market\_configuration.base\_token\_decimals\` is used, the issue is both \`collateral\_configuration.decimals\` and \`storage..market\_configuration.base\_token\_decimals\`'s type is u32, so if \`collateral\_configuration.decimals < storage..market\_configuration.base\_token\_decimals\`, the calculation will revert.

## Vulnerability Details

\`\`\`Rust 878 #\[storage(read)] 879 fn collateral\_value\_to\_sell(asset\_id: AssetId, collateral\_amount: u64) -> u64 { // decimals: base\_token\_decimals 880 let collateral\_configuration = storage.collateral\_configurations.get(asset\_id).read(); 881 let market\_configuration = storage.market\_configuration.read(); 882 ... 894 let scale = u256::from(10\_u64).pow( 895 collateral\_configuration 896 .decimals - storage <<<--- here might be reverted 897 .market\_configuration 898 .read() 899 .base\_token\_decimals, 900 ); ... 906 } \`\`\`

## Impact Details

\`Market.collateral\_value\_to\_sell\` will always revert if collateral\_configuration.decimals < storage..market\_configuration.base\_token\_decimals

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

Please put the following code in \`swaylend-monorepo/contracts/market/src/main.sw\` file, and run \`\`\`bash forc test -l ... Compiled contract "market" with 49 warnings. Finished debug \[unoptimized + fuel] target(s) \[168.928 KB] in 48.32s Running 2 tests, filtered 0 tests test test\_no\_revert\_overflow ... ok (200.787µs, 491 gas) Decoded log value: 10, log rb: 1970142151624111756 test test\_revert\_overflow ... ok (111.65µs, 61 gas)

test result: OK. 2 passed; 0 failed; finished in 312.437µs \`\`\`

as the output shows, test case \`test\_revert\_overflow\` will revert because \`collateral\_token\_decimals\` is smaller than \`base\_token\_decimals\`

\`\`\`Rust #\[test] fn test\_no\_revert\_overflow() { let base\_token\_decimals: u32 = 9; let collateral\_token\_decimals: u32 = 10; let scale = u256::from(10\_u64).pow(collateral\_token\_decimals - base\_token\_decimals); log(scale); }

\#\[test(should\_revert)] fn test\_revert\_overflow() { let base\_token\_decimals: u32 = 9; let collateral\_token\_decimals: u32 = 6; let scale = u256::from(10\_u64).pow(collateral\_token\_decimals - base\_token\_decimals); log(scale); } \`\`\`
