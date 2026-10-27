# #38003 \[BC-Medium] A malicious coordinator calling \`Emily::update\_deposits\` can make the entire Sign

**Submitted on Dec 21st 2024 at 12:44:37 UTC by @f4lc0n for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38003
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/emily>
* **Impacts:**
  * API crash preventing correct processing of deposits

## Description

## Brief/Intro

The Signers network uses a multi-signature mechanism to execute sBTC deposits. For example, in a 3-5 multi-signature, as long as 3 coordinators are honest, the Signers network can operate normally.

However, in the current implementation, as long as a malicious coordinator calls the `Emily::update_deposits` api to update the status of deposits from *Pending* to *Confirmed*, all Signers will no longer be able to process these deposits.

## Vulnerability Details

The `emily/handler/src/api/handlers/deposit.rs::update_deposits` code is as follows.

```rust
#[instrument(skip(context))]
pub async fn update_deposits(
    context: EmilyContext,
    body: UpdateDepositsRequestBody,
) -> impl warp::reply::Reply {
    debug!("In update deposits");
    // Internal handler so `?` can be used correctly while still returning a reply.
    async fn handler(
        context: EmilyContext,
        body: UpdateDepositsRequestBody,
    ) -> Result<impl warp::reply::Reply, Error> {
        // Get the api state and error if the api state is claimed by a reorg.
        //
        // Note: This may not be necessary due to the implied order of events
        // that the API can receive from stacks nodes, but it's being added here
        // in order to enforce added stability to the API during a reorg.
        let api_state = accessors::get_api_state(&context).await?;
        api_state.error_if_reorganizing()?;
        // Validate request.
        let validated_request: ValidatedUpdateDepositsRequest = body.try_into()?;

        // Infer the new chainstates that would come from these deposit updates and then
        // attempt to update the chainstates.
        let inferred_chainstates = validated_request.inferred_chainstates()?;
        for chainstate in inferred_chainstates {
            // TODO(TBD): Determine what happens if this occurs in multiple lambda
            // instances at once.
            crate::api::handlers::chainstate::add_chainstate_entry_or_reorg(&context, &chainstate)
                .await?;
        }

        // Create aggregator.
        let mut updated_deposits: Vec<(usize, Deposit)> =
            Vec::with_capacity(validated_request.deposits.len());

        // Loop through all updates and execute.
        for (index, update) in validated_request.deposits {
            let updated_deposit =
                accessors::pull_and_update_deposit_with_retry(&context, update, 15).await?;
            updated_deposits.push((index, updated_deposit.try_into()?));
        }

        updated_deposits.sort_by_key(|(index, _)| *index);
        let deposits = updated_deposits
            .into_iter()
            .map(|(_, deposit)| deposit)
            .collect();
        let response = UpdateDepositsResponse { deposits };
        Ok(with_status(json(&response), StatusCode::CREATED))
    }
    // Handle and respond.
    handler(context, body)
        .await
        .map_or_else(Reply::into_response, Reply::into_response)
}
```

It does not check the identity of the caller. According to the sBTC architecture, all coordinators have the authority to call the API. Therefore, any coordinator can maliciously call the API to destroy the status of deposits in Emily.

A malicious coordinator can update all deposits to *Confirmed*, and then all other coordinators will not process them anymore.

## Impact Details

This will cause the Signers network to no longer be able to process user deposits.

## Fix

Emily should keep all the opinions of the coordinator on updating the deposits status and execute them when the opinions meet the threshold. Just like the Signer, it should follow the principle of multi-signature.

## References

None

## Proof of Concept

1. Patch `docker/sbtc/Dockerfile`, fix database bug

   ```diff
   RUN git clone https://github.com/stacks-network/sbtc.git
   WORKDIR /code/sbtc

   -ARG GIT_BRANCH=main
   +ARG GIT_BRANCH=immunefi_attackaton_0.9
   RUN git checkout $GIT_BRANCH

   # Run an install that we'll cache the result of and then build the code
   ```
2. Patch `signer/Cargo.toml`, add *poc4* bin

   ```diff
   [[bin]]
   name = "demo-cli"
   path = "src/bin/demo_cli.rs"
   +
   +[[bin]]
   +name = "poc5"
   +path = "src/bin/poc5.rs"
   ```
3. Add [this code](https://gist.github.com/al-f4lc0n/8c34298ddc97dc185f339217a1bd3947) to `signer/src/bin/poc5.rs`
4. Build docker

   ```sh
   make devenv-up
   make devenv-down
   docker compose -f docker/docker-compose.yml --profile default --profile bitcoin-mempool --profile sbtc-signer build
   make devenv-up
   ```
5. Wait for a while, run the demo

   ```sh
   ./signers.sh demo
   ```
6. Wait for a while, run the PoC. This PoC simulates user behavior to create 10 deposits, and then simulates a malicious coordinator calling `Emily::update_deposits` to update the status of these deposits to *Confirmed*

   ```sh
   cargo run -p signer --bin poc5
   ```
7. You will find that these deposits are not executed.
