# #35768 \[SC-Insight] \`Market.set\_pyth\_contract\_id\` should emit an event

**Submitted on Oct 7th 2024 at 09:43:53 UTC by @jasonxiale for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35768
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/market/src/main.sw>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

While changing the system configuration, an event should be emitted. \`Market.set\_pyth\_contract\_id\` is used to change \`Pyth contract\`. Because Pyth is an important factor in the system, so an event should be emitted.

## Vulnerability Details

As shown in the following [code](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L942-L947), there is not event emitted \`\`\`Rust 941 // # 10. Pyth Oracle management 942 #\[storage(write)] 943 fn set\_pyth\_contract\_id(contract\_id: ContractId) { 944 // Only owner can set the Pyth contract ID 945 only\_owner(); 946 storage.pyth\_contract\_id.write(contract\_id); <<<--- event should be emitted 947 } \`\`\`

## Impact Details

Information about the Pyth is changed will be missing.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

As shown in the following [code](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/market/src/main.sw#L942-L947), there is not event emitted \`\`\`Rust 941 // # 10. Pyth Oracle management 942 #\[storage(write)] 943 fn set\_pyth\_contract\_id(contract\_id: ContractId) { 944 // Only owner can set the Pyth contract ID 945 only\_owner(); 946 storage.pyth\_contract\_id.write(contract\_id); <<<--- event should be emitted 947 } \`\`\`
