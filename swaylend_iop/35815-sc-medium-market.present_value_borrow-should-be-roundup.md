# #35815 \[SC-Medium] \`Market.present\_value\_borrow\` should be roundUp

**Submitted on Oct 9th 2024 at 16:11:41 UTC by @jasonxiale for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35815
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

In order to guarantee the contract does not become insolvent, incoming assets should be rounded up, while outgoing assets should be rounded down.

So while a borrower pays debt, his payment should be rounded up. \`Market.present\_value\_borrow\` is used to calculate the amount of base token to pay for debt in \`Market.withdraw\_base\` and other functions, but the function has a rounding error.

## Vulnerability Details

As [Market.present\_value\_borrow](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L1124-L1126) shows \`\`\`Rust 1124 pub fn present\_value\_borrow(base\_borrow\_index: u256, principal: u256) -> u256 { 1125 principal \* base\_borrow\_index / BASE\_INDEX\_SCALE\_15 <<<--- here should be roundUp 1126 } \`\`\`

## Impact Details

the contract might be insolvent

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

The following Rust code is used to demonstrate that \`principal \* base\_borrow\_index / BASE\_INDEX\_SCALE\_15\` is rounded down

\`\`\`bash rustc calc.rs ; .calc 1 1.000000004419746 \`\`\`

\`\`\`Rust pub const BASE\_INDEX\_SCALE\_15: u64 = 1\_000\_000\_000\_000\_000; // 1e15

fn main() { let principal: u64 = 12345678; let base\_borrow\_index: u64 = 81000007; println!("{}", principal \* base\_borrow\_index / BASE\_INDEX\_SCALE\_15); println!("{}", principal as f64 \* base\_borrow\_index as f64 / BASE\_INDEX\_SCALE\_15 as f64); } \`\`\`
