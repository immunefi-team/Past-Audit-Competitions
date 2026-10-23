# #37470 \[BC-Medium] SBTC Signers do not page through pending deposit requests making it trivially eas

**Submitted on Dec 5th 2024 at 14:31:01 UTC by @throwing5tone7 for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #37470
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Temporarily Freezing Network Transactions

## Description

## Brief/Intro

The Emily API to fetch deposit requests is designed to use paging of results, such that if called without paging info, it will only return a limited number of the most recently submitted pending deposit requests when asked (around 1684 in my testing). This means that an attacker can prevent deposit requests from being noticed by the sBTC Signers by submitting bogus deposit requests that fill up that single page of results, such that the legitimate deposit request isn't in the most recent page. To exploit this, an attacker would continuously spam the Emily API with bogus requests, and based on the exact timings of spam vs legitimate requests, most of the legitimate requests will be blocked but some may win the race and get through into the first page and be noticed. The higher the rate of spamming, the higher the proportion of legitimate requests be blocked. If the request isn't noticed by the signers, then the legitimate user doesn't get their BTC converted into sBTC, instead just locking their BTC into an address the signers control, hence the deposit transaction flow is halted.

## Vulnerability Details

The vulnerability resides in the `EmilyClient` implementation of `EmilyInteract::getDeposits` - specifically <https://github.com/stacks-network/sbtc/blob/53cc756c0ddecff7518534a69bef59fadb5ab1d4/signer/src/emily\\_client.rs#L220>

The code is as shown:

```
let resp = deposit_api::get_deposits(&self.config, Status::Pending, None, None)
    .await
    .map_err(EmilyClientError::GetDeposits)
    .map_err(Error::EmilyApi)?;

resp.deposits
    .iter()
    .map(|deposit| {
        Ok(CreateDepositRequest {
            outpoint: OutPoint {
                txid: Txid::from_str(&deposit.bitcoin_txid)
                    .map_err(Error::DecodeHexTxid)?,
                vout: deposit.bitcoin_tx_output_index,
            },
            reclaim_script: ScriptBuf::from_hex(&deposit.reclaim_script)
                .map_err(Error::DecodeHexScript)?,
            deposit_script: ScriptBuf::from_hex(&deposit.deposit_script)
                .map_err(Error::DecodeHexScript)?,
        })
    })
    .collect()
```

Critically, you can see that the last two arguments passed to `deposit_api::get_deposits` are `None, None` - these are the parameters specifying the `next_token` parameter - the start of the next page to fetch from the API and the `page_size` parameter - the size of the page of results to fetch. If these are not specified, then the Emily API returns whatever DynamoDB returns as a page - looking at the docs, it seems like it will return around 1MB worth of results, which in my testing, appears to be around 1684 deposit requests. The correct pattern for using a paging API like the Emily deposits API, would be to call it in a loop, fetching a fixed size (e.g. 1000 results) in a page. To do this, the code would specify `None, 1000` for the paging arguments in the initial call, and then would iterate requesting each further page of results using `resp.next_token, 1000` as the paging arguments until `resp.next_token` becomes `None`.

Because the Signers do not page the Emily API correctly, they only receive the first (arbitrarily-sized) page of results which means any time there are more than a single page worth of pending deposit requests (on an invocation of `BlockObserver::load_latest_deposit_requests` each time a BTC block is observed), the Signers do not notice any deposit requests that are beyond the first page of results. Hence, an attacker can prevent them noticing legitimate deposit requests by spamming bogus deposit requests that fill up the initial page of results.

## Impact Details

By spamming the Emily API with bogus deposit requests, an attacker can prevent the Signers from noticing legitimate deposit requests. If the Signers do not notice a deposit request, then they will not sweep the deposit into their single UTXO of pegged funds, and will not mint the corresponding sBTC. Hence the depositors' BTC funds have been locked without them receiving sBTC. When the spamming is performed continuously, this shuts down the deposit transaction flow for most or all depositors, until the spamming stops (or is blocked via operational means). After the spamming stops, the legitimate deposit requests can be resubmitted, which will mean they begin appearing the initial page of results again and will be processed by the Signers.

Since the deposits can eventually be recovered, and since there will be operational means to manage the spamming, this is a temporary freezing of network transactions.

Whether an individual transaction is blocked or not is down to luck relating to the timings of:

* When they submitted their deposit request
* When the BTC block is completed, which leads to
* When the Signers request pending deposits from Emily API

Given that the Signers request deposits from Emily API every BTC block, and they should start processing within 100ms of noticing that block, then any given deposit request submitted close to the BTC block completion they are part of, could get into Emily's data store before enough of the spam transactions fill up the page after it (by being sent after it). Depending on how fast the spamming is (which I expect to be limited by the throughput of Emily API), an attacker will not deterministically achieve a blanket block of all transactions, but will probabilistically block the majority under reasonable assumptions.

For example, a spam rate of 100 req/s should be trivial to achieve, in which case, only legitimate requests from the last 17s (assuming around 1684 results in the first page) will be likely to be in the page at the time the Signers request deposit requests. The Signers only request from Emily API when they notice a new BTC block, so assuming BTC blocks happen once every 10 minutes, and assuming a uniform arrival time for legitimate deposit requests, a 100 req/s spam rate would be expected to block around (600-17)/600 \~ 97% of legitimate deposits. There is clearly a tradeoff between spam rate and blocking success, and so the feasibility of achieving a blocking of all transactions in practice would come down to operational factors (like processing rates, API throttling and the attacker's infrastructure). However, sending 100 req/s is achievable from a single machine, based on my testing.

## Link to Proof of Concept

<https://gist.github.com/throwin5tone7/57a5d8351698151804838e40d23874f1>

## Proof of Concept

## Proof of Concept

### High-level overview

In order to demonstrate an exploit with this bug, I need to spam the Emily API at a sufficient rate to cause the issue. The rate that is required is proportional to the BTC blockrate, since that is also the frequency that the Signers fetch requests from Emily API. In order to demonstrate the effect, I used the devenv docker compose infrastructure from the SBTC repo, but I turned down the BTC block production rate, after an initial burn-in period. I use a BTC block rate of 1 block every 10 minutes (approximating the real world), and then run a spamming script. While the spamming runs, we submit a legitimate deposit request using the `signers.sh` script and can see that the Signers do not notice it in the current or any future blocks.

### Steps

Setup the code:

* Modify the devenv bitcoin-miner script slightly, so that after some threshold, it starts producing blocks every 10 minutes - in my example I have this happen after from block 252 onwards, which you can recreate by applying `docker-compose.yml.patch` from my GIST to `docker/docker-compose.yml` in the repo
* Make a script / program to run the spamming, for example apply `demo_cli.rs.patch` from my GIST to `signer/src/bin/demo_cli.rs` in your repo to follow my steps
  * NOTE: this example rust script only achieves a request rate of around 70 req/s, but I have achieved better than 100 req/s by pre-generating the random tx IDs and then feeding them to the network testing tool `wrk` - this is a bit harder to communicate the steps for, but please ask if you want more details, as I can easily achieve more than 100 req/s with the alternative setup.

Run the PoC (assuming you've used the same patches as I supplied):

* Launch the devenv using `make devenv-up`
* Wait until BTC block 252 occurs
  * At this point, double check the Stacks explorer to see that the setup steps for sBTC have happened correctly (i.e. rotate key transaction has occurred along with the other bits mentioned in your readme)
* Begin the spamming script e.g. `cargo run -p signer --bin demo-cli spam`
* Some time during the first 5 minutes after the latest block, submit a legitimate deposit transaction, by running `./signers.sh demo` from your repo
  * You might want to capture the txid for reference
* Wait 30 seconds or so - at this point if you ask Emily API for pending deposit requests without paging parameters then you will not see the legitimate deposit txid in the results returned (and neither will the Signers)
* Now monitor the next few blocks - the Signers do not notice the transaction, nor sweep it into their pegged UTXO, nor mint sBTC for the recipient

Once the spamming stops, resubmitting the legitimate deposit transaction appears to allow the deposit to proceed.
