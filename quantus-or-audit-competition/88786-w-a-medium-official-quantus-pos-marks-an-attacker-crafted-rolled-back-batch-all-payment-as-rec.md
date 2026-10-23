# 88786 w a medium official quantus pos marks an attacker crafted rolled back batch all payment as received in violation of the node s documented integration requirement

**Submitted on Aug 15th 2026 at 03:38:31 UTC by @archcypher for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88786
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**

## Description

## Brief/Intro

The official Quantus mobile wallet's point-of-sale flow promotes a `txWatch_transfer` mempool notification directly to a completed-payment state. An attacker can place a transfer matching the POS invoice inside `Utility::batch_all`, followed by a child transfer that must fail. `txWatch` decodes and announces the merchant transfer from ready-pool call data **before execution**; when the extrinsic executes, `batch_all` atomically rolls that transfer back. The merchant's POS nevertheless enters its paid state, titles the screen **Payment Received**, and enables **Done** / **New Charge**. The merchant receives no QTU.

**This report does not claim that `txWatch` should provide finality.** Its zero-confirmation behavior is documented, intentional, and correct. The vulnerability is that the in-scope official POS client consumes that explicitly-untrusted signal as payment confirmation — which the project's own node documentation prohibits in those words, while describing this exact `batch_all` attack as the reason.

## Vulnerability Details

### 1. `txWatch` announces requested transfers before execution (intended, documented)

`node/src/txwatch.rs:74-118` subscribes to `pool.import_notification_stream()`, takes the extrinsic from `pool.ready_transaction()`, decodes it, calls `extract_all_transfers(&uxt.function, 0)`, and broadcasts the result. No execution occurs and no dispatch event is consulted. `extract_all_transfers` (`:207-231`) deliberately descends into `Utility::batch`, `batch_all`, and `force_batch`, so a transfer requested inside an atomic batch produces a notification.

### 2. A later `batch_all` child deterministically rolls the payment back

`pallets/utility/src/lib.rs` implements `batch_all` atomically: if any child returns an error, the entire batch reverts, including earlier successful children. A second balance transfer larger than the attacker's free balance is sufficient, and remains valid transaction-pool call data (ordinary validation does not simulate dispatch-level balances).

### 3. The official app promotes the notification straight to paid — with no revocation path

In `mobile-app/lib/v2/screens/pos/pos_qr_screen.dart`, the `onTransfer` callback (`:75-104`) compares only the requested numeric amount against the invoice. It then adds a pending record and calls `startPolling(pendingTx)` — which returns `void`, starts timer-driven background polling, and returns immediately. Critically, `startPolling` accepts an optional `{void Function(TransactionEvent result)? onFound}` confirmation callback (`services/pending_transaction_polling_service.dart:23`); **the POS supplies no `onFound` argument**, so nothing gates the paid state on the poller's result. The callback then executes `setState(() { _paidTransfer = tx; ... })`.

* `_isPaid` is defined as `_paidTransfer != null` (`:48`).
* `_paidTransfer` is assigned at exactly one place (`:100`) and is **never reset to null anywhere in the file**. There is no callback that revokes the paid state if the expected transfer event never materialises.
* `_isPaid` drives the app bar title to `posQrTitlePaymentReceived` (`:173`) and swaps the bottom actions to the paid-state **Done** / **New Charge** buttons (`:188`).

The background poller therefore does not protect the merchant: it runs *after* the UI has already declared success, and its timeout only removes a separate pending-history record.

{% hint style="info" %}
**Scope note, disclosed for accuracy:** at `:184` the paid pane is invoked without returning its widget (`if (_isPaid) _buildPaidContent(...)` followed by `return _buildQrContent(...)`), so the *main pane* still renders the QR. The merchant therefore sees the **title change to "Payment Received"** and the **Done / New Charge buttons appear**, but not a full paid screen. Those two indicators are the operative signal for fulfilment, and they are sufficient for the attack; the missing `return` is additional evidence that the paid-state integration is incomplete, not a separate finding.
{% endhint %}

## Impact Details

An attacker obtains goods or services from a merchant using the official POS flow while paying only the failed extrinsic's transaction fee. The attack is deterministic, repeatable, requires no privilege, no node or RPC compromise, no transaction-replacement or chain-reorg race, no fork, no invalid signature, and no social engineering — the attacker performs exactly the intended customer action: scan the merchant QR and submit a signed transaction.

**Stated plainly, so the severity mapping is not overstated:** no QTU leaves or enters the merchant's account — the PoC asserts the merchant's balance is exactly unchanged. The loss to the merchant is the goods or services released against a false confirmation, not an on-chain debit. I am therefore *not* claiming "direct theft of user funds" in the on-chain sense, and I leave the final impact mapping to the program. What is demonstrated is a deterministic, attacker-controlled false-confirmation state in an in-scope official application, causing real economic loss to the merchant who relies on it.

The defect is a violation of the project's own documented security requirement for this client, producing a deterministic false-confirmation state that a merchant is expected to act on. I defer entirely to the program's classification of severity and impact.

### Why this is not covered by the zero-confirmation design decision

The node documentation (`docs/rpc_additions/transaction_pool_listener.md`) is explicit, and it is the project's own integration contract for this exact client:

> **Never release goods, mark an invoice paid, or credit a balance based on a `txWatch_transfer` notification alone.** Treat it as "customer initiated a payment". Before fulfilling, verify the transfer actually executed…

The same document anticipates this precise attack:

> **This is attacker-reachable.** Anyone can deliberately submit a transaction that triggers a convincing notification (any address, any amount) but never moves funds, paying at most a transaction fee — e.g. a `batch_all` containing one real-looking transfer and one transfer that must fail.

and separately notes that batch semantics are not modelled, so "a batch that looks like several payments can execute none of them."

The documentation's allowance that accepting zero-conf may be "a deliberate risk decision … only do so for amounts you can afford to lose" appears immediately after the attacker-reachable bullet, so — stated fairly — it contemplates *deliberate attacks*, not merely probabilistic settlement risk, and it does permit an integrator to knowingly accept that risk. My claim is narrower than "the app is forbidden from accepting zero-conf":

1. The documentation frames zero-conf acceptance as **a deliberate, bounded risk decision** — "only do so for amounts you can afford to lose". The official POS applies it **unconditionally**: no amount threshold, no setting, no warning, and no distinct pending/unconfirmed state on the POS screen itself.
2. The POS screen does not gate confirmation behind that pending state. It adds a `PendingTransactionEvent` to the history provider, but the *screen* immediately presents the paid title and the fulfilment actions, and supplies no `onFound` callback that could revoke them.
3. Because there is no revocation path at all (`_paidTransfer` is never reset), the merchant is never informed that the payment failed — so even an integrator who consciously accepted zero-conf risk is not given the information needed to bound it.

## Recommended Fix

Adopt a three-state POS flow — **detected/pending**, **confirmed**, **failed**:

* A `txWatch_transfer` notification may populate only the detected/pending state (e.g. "payment detected…"), and must not enable fulfilment actions.
* Promote to **Payment Received** / enable **Done** only after the extrinsic is included and finalized, returns `ExtrinsicSuccess`, and emits the expected native `Balances.Transfer` event for that merchant and amount.
* Actively revoke the pending state when the extrinsic is dropped, replaced, dispatch-fails, or is finalized without the expected event. (`_paidTransfer` currently has no reset path at all.)
* Do not attempt to fix this by rejecting `batch_all` alone — the node documentation notes that ordinary calls can also be dropped, replaced, invalidated, or fail at execution.
* Add a regression test asserting that a pool notification alone never sets `_isPaid`, and that a rolled-back/failed extrinsic never exposes the paid actions.
* Separately, return the widget at `pos_qr_screen.dart:184`.

## References

* `mobile-app/lib/v2/screens/pos/pos_qr_screen.dart:48, 75-104, 173, 184, 188` — paid state set from the pool notification, never revoked, drives title and actions.
* `node/src/txwatch.rs:74-118, 207-231` — pre-execution announce; descends into `batch`/`batch_all`/`force_batch`.
* `pallets/utility/src/lib.rs` — `batch_all` atomicity.
* `docs/rpc_additions/transaction_pool_listener.md` — the prohibition and the `batch_all` attacker example.
* Commit `5507d6c` — the security review that introduced the "never mark an invoice paid" integration requirement and documented this attack. The scoped app snapshot (`d5fc4d6`) post-dates it and still violates it.
* Duplicate check: `Quantus-Network/chain#446` (CLOSED) introduced the commerce transaction-pool feature. Disclosed in full for fairness, it says of the pool signal: *"It's not on chain yet, and it's not final, but probably good enough for small day to day, or at least for showing a pending state in an app."* So it does contemplate zero-conf acceptance for small amounts. Two points on why that does not cover this report:

  * It is a pre-implementation design proposal that **predates** the security review in `5507d6c`, which subsequently added the explicit "never mark an invoice paid" requirement and the `batch_all` attacker example.
  * The shipped POS implements **no amount threshold** — it applies the same immediate paid state to any invoice value, so the "small day to day" qualifier is not realised in code. Neither issue discloses the official app marking an atomically rolled-back payment as received.

  `Quantus-Network/quantus-apps#424` concerns the 30-second watch timeout, not false confirmation.

## Link to Proof of Concept

<https://gist.github.com/archcypher/ba845759f33d88e9614bef8490296547>

## Proof of Concept

The PoC runs against the **real** `quantus-node` and `quantus-runtime` crates — not mocks — and asserts both halves of the exploit inside a single externalities execution:

1. The exact extraction function used by `txWatch` (`extract_transfers_to` → `extract_all_transfers`) identifies the merchant payment from the malicious call data.
2. The real runtime rejects the batch and leaves the merchant's balance unchanged.

**Scope of the automated test, stated precisely.** The test calls the extraction function directly; it does not instantiate a transaction pool, deliver a live `txWatch_transfer` subscription notification, or drive the Flutter UI. Those two links are established by source inspection rather than by this test: `node/src/txwatch.rs:74-118` takes the extrinsic from `pool.ready_transaction()`, passes it to that same `extract_all_transfers`, and broadcasts the result via `fanout.send(...)` with no execution or event check; and `pos_qr_screen.dart:75-104, 48, 173, 188` sets `_paidTransfer` from that notification and renders the paid title and actions. The test proves the two ends that can be proven automatically — the malicious call data is extracted as a merchant payment, and the runtime rolls it back.

## Run

```bash
# default layout: <workspace>/poc/ alongside <workspace>/repos/chain/
./poc/run_pos_false_payment_poc.sh

# or point it at any checkout of audit-comp-quantus-chain:
QUANTUS_CHAIN_DIR=/path/to/audit-comp-quantus-chain ./poc/run_pos_false_payment_poc.sh
```

The script applies `poc/F2-pos-false-payment.patch` (which adds one `#[test]` to `node/src/txwatch.rs`), runs it, and reverts only that hunk on exit (including on failure).

**Requirements:** Rust `1.93.0` (per the repo's `rust-toolchain`), `protoc` on PATH, and `git`. `cargo` is taken from PATH; override with `CARGO_BIN` if needed. libclang (for `librocksdb-sys`/bindgen) is normally found automatically on Linux — install `clang`/`libclang-dev` if not — and on macOS the script defaults to the Command Line Tools path only when that directory actually exists; override either case with `LIBCLANG_PATH`. First cold build \~9 minutes; subsequent runs seconds.

## The test

```rust
let malicious = batch_all(vec![
    native_transfer(&merchant, payment),
    // Necessarily fails (2x the customer's balance). Because batch_all is atomic,
    // it rolls the merchant transfer back as well.
    native_transfer(&sink, customer_balance * 2),
]);

// 1. The extraction function txWatch uses identifies this payment from call data,
//    before any execution takes place.
assert_eq!(extract_transfers_to(&malicious, &merchant), vec![(payment, None)]);

// 2. The real runtime rejects the batch and the merchant gains nothing.
let merchant_before = Balances::free_balance(&merchant);
assert!(malicious.dispatch(RuntimeOrigin::signed(customer)).is_err());
assert_eq!(Balances::free_balance(&merchant), merchant_before);
```

## Expected output

```
running 1 test
test txwatch::tests::poc_batch_all_extracted_payment_that_runtime_rolls_back ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 38 filtered out; finished in 0.01s

PASS: the txWatch extraction function identified the merchant payment from the malicious call data.
PASS: the real Quantus runtime rejected the later child and rolled the merchant balance back.
```

Note: the two `PASS:` lines are printed by the wrapper script after `cargo test` exits 0 — they are not emitted by the test itself. The substantive evidence is the `... ok` result plus the two assertions in the test body.

## End-to-end attack sequence

{% stepper %}
{% step %}

## Merchant creates a POS invoice

Merchant enters an amount in Quantus POS; the app displays its QR and calls `txWatch_watchAddress(merchant)`.
{% endstep %}

{% step %}

## Attacker reads invoice details

Attacker reads merchant address + amount from the QR.
{% endstep %}

{% step %}

## Attacker submits a malicious atomic batch

Attacker signs and submits:

```
Utility.batch_all([
  transfer_keep_alive(merchant, invoice_amount),
  transfer_keep_alive(sink, > attacker_balance)
])
```

{% endstep %}

{% step %}

## Node accepts the transaction into the ready pool

The node accepts it into the ready pool (ordinary validation does not simulate dispatch-level balances).
{% endstep %}

{% step %}

## `txWatch` announces the requested merchant payment

`txWatch` extracts the first child and pushes a `txWatch_transfer` notification with the exact merchant address and amount.
{% endstep %}

{% step %}

## POS marks the payment as received

`pos_qr_screen.dart:75-104` matches the amount and sets `_paidTransfer` ⇒ `_isPaid == true` ⇒ title becomes **Payment Received** (`:173`) and **Done / New Charge** are enabled (`:188`).
{% endstep %}

{% step %}

## Batch execution rolls back the payment

On dispatch the second child fails; `batch_all` rolls the first child back. The merchant's balance is unchanged (asserted above).
{% endstep %}

{% step %}

## POS never revokes the paid state

Nothing in the app ever revokes `_paidTransfer` — it has no reset path (`:48`, `:100`).
{% endstep %}

{% step %}

## Merchant fulfils the sale

Merchant fulfils the sale. Attacker repeats at the cost of a failed-extrinsic fee.

The attack requires no transaction replacement, congestion, fork, invalid signature, malicious RPC endpoint, or compromised indexer.
{% endstep %}
{% endstepper %}
