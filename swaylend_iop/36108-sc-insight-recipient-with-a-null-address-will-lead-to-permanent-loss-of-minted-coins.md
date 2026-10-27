# #36108 \[SC-Insight] \`recipient\` with a NULL address will lead to permanent loss of minted coins

**Submitted on Oct 19th 2024 at 21:39:39 UTC by @savi0ur for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #36108
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/src-20/src/main.sw>
* **Impacts:**

## Description

## Bug Description

<https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/src-20/src/main.sw#L119-L147> \`\`\`sway #\[storage(read, write)] fn mint(recipient: Identity, sub\_id: Option\<SubId>, amount: u64) { require( sub\_id .is\_some() && sub\_id .unwrap() == DEFAULT\_SUB\_ID, "incorrect-sub-id", ); require( storage .owner .read() == State::Initialized(msg\_sender().unwrap()), AccessError::NotOwner, ); require( storage .total\_supply .read() + amount <= MAX\_SUPPLY, "max-supply-reached", );

```
let new_supply &#x3D; storage.total_supply.read() + amount;
storage.total_supply.write(new_supply);

mint_to(recipient, DEFAULT_SUB_ID, amount); //@audit-issue

TotalSupplyEvent::new(AssetId::default(), new_supply, msg_sender().unwrap())
	.log();
```

} \`\`\`

\`mint\` function is used to mint SRC20 token to the provided \`recipient\` identity. This \`recipient\` identity could be an \`Address\` or \`ContractId\`. However there is no validation performed on \`recipient\` to make sure its not a NULL address. Due to this, token minted to NULL address will get lost permanently as mentioned in the comment in \`std\` sway library of \`mint\_to\` function.

<https://github.com/FuelLabs/sway/blob/029c593b5f33a5c18ed2ddf91b1db12869139c91/sway-lib-std/src/asset.sw#L18>

\> If the \`to\` Identity is a contract, this will transfer coins to the contract even with no way to retrieve them (i.e: no withdrawal functionality on the receiving contract), possibly leading to the ***PERMANENT LOSS OF COINS*** if not used with care.

So it should have an check to avoid the possibility of minting tokens to NULL address.

## Impact

\`recipient\` with a NULL address will lead to permanent loss of minted coins.

## Recommendation

We recommend to have an check at the start of the function to make sure \`recipient\` is not NULL address, just like how \`ThunderNFT\` did,

<https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/pool/src/main.sw#L237C1-L242C7> \`\`\`sway fn \_transfer(from: Identity, to: Identity, asset: AssetId, amount: u64) { require( to != ZERO\_IDENTITY\_ADDRESS && to != ZERO\_IDENTITY\_CONTRACT, PoolErrors::IdentityMustBeNonZero ); \`\`\` <https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/libraries/src/constants.sw#L9-L14> \`\`\`sway pub const ZERO\_B256 = 0x0000000000000000000000000000000000000000000000000000000000000000; pub const ZERO\_ADDRESS = Address::from(ZERO\_B256); pub const ZERO\_ASSET\_ID = AssetId::from(ZERO\_B256); pub const ZERO\_CONTRACT\_ID = ContractId::from(ZERO\_B256); pub const ZERO\_IDENTITY\_ADDRESS = Identity::Address(ZERO\_ADDRESS); pub const ZERO\_IDENTITY\_CONTRACT = Identity::ContractId(ZERO\_CONTRACT\_ID); \`\`\`

## References

* <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/src-20/src/main.sw#L119-L147>

## Proof Of Concept

**Steps to Run:**

* Open terminal and run \`cd swaylend-monorepo\`
* Paste following rust code in \`contracts/market/tests/local\_tests/scenarios/owner.rs\`
* Run test using \`cargo test --package market --test integration\_tests -- local\_tests::scenarios::owner::owner\_mint\_to\_null\_test --exact --show-output\`

\`\`\`rust #\[tokio::test] async fn owner\_mint\_to\_null\_test() { let TestData { usdc\_contract, .. } = setup(None).await;

```
let null_address &#x3D; Address::zeroed();
let null_identity_address &#x3D; Identity::Address(null_address);
// let null_identity_contract_id &#x3D; Identity::ContractId(ContractId::zeroed());
let amount_to_mint &#x3D; 100_000_000u64;
let default_sub_id &#x3D; Bits256::zeroed();

let res &#x3D; usdc_contract
    .instance
    .methods()
    .mint(null_identity_address, Some(default_sub_id), amount_to_mint)
    .with_variable_output_policy(
        fuels::types::transaction_builders::VariableOutputPolicy::Exactly(1),
    )
    .with_tx_policies(fuels::types::transaction::TxPolicies::default().with_tip(1))
    .call()
    .await;
println!(&quot;Result: {:#?}&quot;, res);
assert!(res.is_ok());
```

} \`\`\`

**Console Output:**

\`\`\`shell running 1 test test local\_tests::scenarios::owner::owner\_mint\_to\_null\_test ... ok

successes:

\---- local\_tests::scenarios::owner::owner\_mint\_to\_null\_test stdout ---- Price for ETH = 3500 Price for USDC = 1 Price for BTC = 70000 Price for UNI = 5 Result: Ok( CallResponse { value: (), receipts: \[ Call { id: 0x0000000000000000000000000000000000000000000000000000000000000000, to: 0x3df03c285737d01c8f9101ea59fa929f6e8f3d8cc5abe9a5b2ebeb4f8787a61d, amount: 0, asset\_id: 0x0000000000000000000000000000000000000000000000000000000000000000, gas: 7590, param1: 10480, param2: 10492, pc: 11848, is: 11848, }, Mint { sub\_id: 0x0000000000000000000000000000000000000000000000000000000000000000, contract\_id: 0x3df03c285737d01c8f9101ea59fa929f6e8f3d8cc5abe9a5b2ebeb4f8787a61d, val: 100000000, pc: 14956, is: 11848, }, TransferOut { id: 0x3df03c285737d01c8f9101ea59fa929f6e8f3d8cc5abe9a5b2ebeb4f8787a61d, to: 0x0000000000000000000000000000000000000000000000000000000000000000, amount: 100000000, asset\_id: 0x27730a16b66a3565909ebe411c1c6501e605b709fd8c06970bf3fb1e138ec6b6, pc: 16736, is: 11848, }, ReturnData { id: 0x3df03c285737d01c8f9101ea59fa929f6e8f3d8cc5abe9a5b2ebeb4f8787a61d, ptr: 0, len: 0, digest: 0xe3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855, pc: 16744, is: 11848, data: Some(), }, Return { id: 0x0000000000000000000000000000000000000000000000000000000000000000, val: 1, pc: 10388, is: 10368, }, ScriptResult { result: Success, gas\_used: 7509, }, ], gas\_used: 7509, log\_decoder: LogDecoder { log\_formatters: { LogId( 0x3df03c285737d01c8f9101ea59fa929f6e8f3d8cc5abe9a5b2ebeb4f8787a61d, "10098701174489624218", ): LogFormatter { type\_id: TypeId { t: ( 8840596190301856573, 10685351592856360230, ), }, }, }, decoder\_config: DecoderConfig { max\_depth: 45, max\_tokens: 10000, }, }, tx\_id: Some( 0x6a9d716a80ade6b92b94b05adb0108e5584c8b4f75e2f1ea5d6ff0b1b206dd43, ), }, )

successes: local\_tests::scenarios::owner::owner\_mint\_to\_null\_test \`\`\`

As we can see in \`TransferOut\` block, its transferring minted assets to NULL address. Which shows, the code needs to handle null address check, else it will lead to permanent loss of minted assets.

## Proof of Concept

## Proof of Concept
