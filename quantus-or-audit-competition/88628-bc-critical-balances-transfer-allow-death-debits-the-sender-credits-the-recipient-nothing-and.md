# 88628 bc critical balances transfer allow death debits the sender credits the recipient nothing and reports success

**Submitted on Aug 14th 2026 at 08:55:03 UTC by @rev3rse for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88628
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

`fungible::Mutate::transfer` evaluates its precondition on one quantity and credits a different one. When a sender's whole balance is near the existential deposit, `transfer_allow_death` **debits the sender, writes nothing to the recipient, and returns `Ok`** — emitting a `Transfer` event for value that never moved.

At production parameters (`ExistentialDeposit = 1,000,000,000` planck) a transfer of 500,000,000 planck destroys the sender's full 1,000,000,000 while the recipient's balance stays at zero. It also leaves `total_issuance` permanently higher than the sum of all account balances, by exactly the transferred amount.

This is a regression introduced by Quantus commit `db94e2a` "Harden FRAME primitives against audit findings (#616)". Upstream Substrate rejects the identical call.

No attacker is required and no attacker benefits — this is accidental loss for ordinary users whose accounts sit near dust, plus a permanent accounting divergence.

## Vulnerability Details

**Asset:** `immunefi-team/audit-comp-quantus-chain` (Blockchain/DLT), frozen at `3b243b87`\
**File:** `frame/support/src/traits/tokens/fungible/regular.rs`\
**Entry point:** `Balances::transfer_allow_death` (`pallets/balances/src/lib.rs:671-679`)\
**Bug class:** Accounting desynchronisation — precondition checked on a different value than the one written\
**Severity claimed:** **C3 — "Direct loss of funds"** (Blockchain/DLT), the row this defect actually occupies. I state the measured magnitude bound in the same breath and leave the tier designation to triage. See "Severity" below for why no Low row fits.

### Root cause

`frame/support/src/traits/tokens/fungible/regular.rs:317-343`:

```rust
let dust   = Self::can_withdraw(source, amount).into_result(preservation != Expendable)?;
let actual = amount.saturating_add(dust);
Self::can_deposit(dest, actual, Extant).into_result()?;      // <-- gate on amount + dust
...
let debited = Self::decrease_balance(source, amount, BestEffort, preservation, Polite)?;
// Credit the amount actually removed from the source. When an expendable transfer reaps
// the source, `debited` can exceed `amount`; ignoring it desynchronizes issuance from
// account balances.
let _ = Self::increase_balance(dest, debited, BestEffort);    // <-- credit `debited`, result DISCARDED
Self::done_transfer(source, dest, debited);
Ok(debited)
```

Three facts combine:

1. **The gate and the credit use different quantities** — `amount + dust` versus `debited`.
2. **The comment's premise no longer holds in this tree.** `decrease_balance` (`:188-197`) returns `old_balance.saturating_sub(new_balance)` where `new_balance = old − amount`, i.e. exactly `amount`. Dust is disposed of separately via `write_balance`/`handle_dust` and never appears in the return value, so **`debited` can never exceed `amount`**.

   The guard was not written carelessly — it was correct for `pallets/assets`, the one implementor that overrode `decrease_balance` such that `debited > amount` was reachable. **That pallet was removed from this repository in `e362a87` ("Remove assets pallet", #637), and `pallets/assets` is absent from the tree at `3b243b87`.** The widened gate therefore has no remaining beneficiary in this codebase, which is what makes the fix below lossless rather than a trade-off.
3. **The credit silently no-ops.** `increase_balance(dest, debited, BestEffort)` (`:205-222`) returns `Ok(Default::default())` **writing nothing** when `new_balance < minimum_balance`, and the caller discards that result with `let _ =`.

When `amount < ED <= amount + dust`, the widened gate admits the call and the credit does nothing.

### The project's own commit and test both state the intended behaviour

This is not a disputed design call. The commit that introduced this code, `db94e2a` ("Harden FRAME primitives against audit findings", #616), describes its purpose as:

> *"Credit the full amount removed from the source in generic transfer helpers **so expendable reaping cannot desync issuance from balances**."*

The change was written specifically to prevent the invariant that it breaks: after this defect fires, `TotalIssuance` exceeds the sum of account balances by exactly the destroyed amount.

The project's own test suite states the expected outcome for this exact call shape. `pallets/balances/src/tests/dispatchable_tests.rs:31-45`, with `ExistentialDeposit = 10`, a dead destination and `amount = 9 < ED`:

```rust
assert_noop!(
    Balances::transfer_allow_death(Some(1).into(), 5, 9),
    TokenError::BelowMinimum,
);
```

**The suite says this call must fail with `BelowMinimum`.** It does not catch the bug only because its sender (account 1, balance 100) is well funded, so no reap occurs and `dust = 0`, leaving the gate un-widened. Make the sender reapable and the same call returns `Ok` while destroying the funds — which is precisely what my PoC does.

## Impact Details

An ordinary user whose account sits near the existential deposit — anyone sweeping a nearly-empty account, or any automated dust-consolidation service — loses their remaining balance while the chain reports the transfer succeeded and emits a `Transfer` event for value that never arrived. Off-chain indexers that credit on `Transfer` events will record a deposit that does not exist.

Each incident also permanently breaks `total_issuance == Σ(balances)` by the transferred amount. Nothing on-chain reconciles it, so the divergence accumulates for the life of the chain.

**Second, latent site:** `frame/support/src/traits/tokens/fungibles/regular.rs:373-396` is a verbatim clone of the same defect for the multi-asset trait. It is unreachable today — no `fungibles` implementor exists in-tree — but it arms the moment one is added. Please fix both together.

## References

* `frame/support/src/traits/tokens/fungible/regular.rs:317-343` — `transfer`, the defect
* `frame/support/src/traits/tokens/fungible/regular.rs:188-197` — `decrease_balance`, proving `debited <= amount`
* `frame/support/src/traits/tokens/fungible/regular.rs:205-222` — `increase_balance` returning `Ok(0)` below ED
* `frame/support/src/traits/tokens/fungibles/regular.rs:373-396` — the latent multi-asset clone
* `pallets/balances/src/lib.rs:671-679` — `transfer_allow_death`, passes `Expendable`
* `pallets/balances/src/lib.rs:745` — `transfer_all` with `keep_alive = false`, a second entry point
* `pallets/balances/src/impl_fungible.rs:191-211` — `impl fungible::Mutate` defines only `done_*`, so the defective default body is inherited
* `pallets/balances/src/impl_fungible.rs:204-210` — `done_transfer` emits `Event::Transfer` with the uncredited amount
* `runtime/src/lib.rs:102` — `EXISTENTIAL_DEPOSIT = MILLI_UNIT = 1e9`
* `runtime/src/lib.rs:97` — `UNIT = 1e12`
* Commit `db94e2a` "Harden FRAME primitives against audit findings (#616)", 2026-07-03 — the introducing change; `git log -S` on this hunk returns only this commit
* Upstream Substrate `fungible::Mutate::transfer` — gates on `amount` and credits `amount`
* `pallets/balances/src/tests/` — the existing 277-test suite passes with the defect live; it never constructs the `amount < ED <= amount + dust` case
* Prior audits checked for duplication: `quantus-substrate-audit-20260513.pdf` (Equilibrium — scope was treasury/multisig/scheduler/reversible-transfers at commit `0467a304`, with "other pallets and runtime modules not listed above" explicitly out of scope; `frame/support/` was never in scope for any engagement), `EigerWormholeAudit.pdf`, `Proof-of-Work and Poseidon Security Review 1.1.pdf`, `Neodyme-Audit.pdf`. None reports this.
* `chain/pallets/balances/src/tests/dispatchable_tests.rs:31-45` — the project's own test asserting `TokenError::BelowMinimum` for this call shape
* Commit `db94e2a` (#616) "Harden FRAME primitives against audit findings" — introduced the discarded-credit code; its message states the invariant it was meant to protect
* Commit `e362a87` (#637) "Remove assets pallet" — removed the only implementor for which the widened gate was necessary
* Immunefi program out-of-scope list, "Account reaping leading to replayed transactions" — addressed above; impact-scoped to replay, which this report does not claim

## Link to Proof of Concept

<https://gist.github.com/almog-shachar/1fa0520b5724fb3248b55a02a366786e>

## Proof of Concept

Single patch, applies to the frozen asset repo. Adds tests only — no production code is modified.

```bash
git apply f5-poc.patch

# Production runtime parameters (real ExistentialDeposit, real Runtime):
cargo test -p quantus-runtime --test mod poc_f5 -- --nocapture      # 4 passed

# Pallet level, plus the upstream differential:
cargo test -p pallet-balances --lib poc_f5 -- --nocapture           # 5 passed
```

### Measured at production parameters

`runtime/tests/transactions/poc_f5_transfer.rs` drives the **real `quantus_runtime::Runtime`**:

```
--- F5 at PRODUCTION runtime parameters ---
ExistentialDeposit     : 1000000000 planck
amount transferred     : 500000000 planck
transfer() returned    : Ok(500000000)          <-- the call SUCCEEDS
alice before / after   : 1000000000 / 0         <-- sender fully debited
bob   before / after   : 0 / 0                  <-- recipient receives NOTHING
issuance before / after: 1000000000 / 500000000
sum(bal) before / after: 1000000000 / 0
VALUE DESTROYED        : 1000000000
```

The test asserts the issuance relation exactly: `issuance_after − sum_after == amount`. Before the call the invariant `total_issuance == Σ(balances)` holds; afterwards it is broken by 500,000,000 planck, permanently.

### Controls — included so the finding is bounded, not overclaimed

```
--- F5 bound: funded sender ---
alice balance      : 10000000000        (10x ED)
transfer() returned: Err(Token(BelowMinimum))     <-- correctly REJECTED

--- F5 control (Preserve) ---
transfer() returned: Err(Token(NotExpendable))    <-- transfer_keep_alive unaffected
```

A funded sender is rejected, because without reaping there is no dust and the gate collapses back to `amount`. `Preservation::Preserve` — what `transfer_keep_alive` uses — is likewise rejected. **The defect is confined to the `Expendable` path with a near-dust sender.**

### Worst case, measured by exhaustive sweep

```
--- F5 worst case at production ED ---
ExistentialDeposit  : 1000000000 planck
worst single loss   : 1375000000 planck
as multiple of ED   : 1.38x
as QUAN (UNIT=1e12) : 0.001375
```

### The differential against upstream

`pallets/balances/src/tests/poc_f5_tests.rs` additionally shows the call is rejected upstream. I patched the single gate line to upstream's form and re-ran the identical test:

| gate                                                | `transfer()` result                                 |
| --------------------------------------------------- | --------------------------------------------------- |
| Quantus: `can_deposit(dest, amount + dust, Extant)` | **`Ok(5)`** — funds destroyed                       |
| Upstream: `can_deposit(dest, amount, Extant)`       | **`Err(Token(BelowMinimum))`** — correctly rejected |

That isolates the Quantus modification as the sole cause and shows the one-line revert is a complete fix. (`regular.rs` was restored to pristine afterwards; the patch does not modify it.)

### Mainnet fork — four real on-chain accounts losing their entire balances

`runtime/tests/transactions/poc_f5_mainnet_fork.rs` forks **real live state** into the **audited** runtime. It loads the verbatim `System::Account` storage entries of the four Heisenberg accounts that currently sit in the vulnerable band, plus the live `Balances::TotalIssuance`, captured read-only at a **pinned block**:

```
network : Heisenberg (a1-heisenberg.quantus.cat)
block   : #821811
hash    : 0xf8163ed7a2568ee4b492c1c56e3b20175d3288a443e97dcb608b8e887eb5cdda
```

The state is pinned so the PoC stays reproducible after the chain moves on. No transaction was ever sent to the live network — capture was `state_getKeysPaged` + `state_queryStorageAt` at that hash.

```
--- F5 on a fork of REAL Heisenberg state, block #821811 ---
ExistentialDeposit : 1000000000 planck
  live account 0x15c74d8b00f6cc6f… balance 1000000000
  live account 0x28207d983d956ba2… balance 1000000000
  live account 0x0b495a0592bb391c… balance 1000000001
  live account 0xf8fecf9eebd44180… balance 1000000001

victim (real, on-chain) : 0x15c74d8b00f6cc6f4085bfb4e7adf6557fbcc43d920c110810caa813588d2c18
balance before          : 1000000000 planck
amount sent             : 500000000 planck
transfer() returned     : Ok(500000000)
victim balance after    : 0
recipient balance after : 0
issuance delta          : 500000000
VALUE DESTROYED         : 1000000000 planck

--- all four real accounts, partial send of ED/4 ---
accounts destroyed      : 4 / 4
total value destroyed   : 4000000002 planck
```

The loaded balances match the live chain exactly, which is the control proving the fork is real rather than synthetic. **All four accounts lose their entire balance**, and the total — **4,000,000,002 planck** — reproduces the independent census figure above exactly.

**Why a fork of live&#x20;*****state*****&#x20;and not of the live&#x20;*****chain-spec*****.** The chain-spec's raw genesis embeds its own 775,279-byte runtime WASM (`:code`), so forking it would execute the *deployed* runtime rather than the audited tree this report concerns. Forking state into the audited runtime tests the in-scope code against production reality, which is the property that matters.

**One honest note on method.** My first fork attempt loaded `System::Account` but not `Balances::TotalIssuance`, leaving accounts funded against zero issuance. The transfer then returned `Err(Arithmetic(Underflow))` — which reads exactly like "the defect does not fire on mainnet". It was an inconsistent fork, not a refutation. State forks must be internally consistent or they manufacture false negatives.

## This is live on both public networks, and accounts are sitting in the vulnerable band today

Read-only RPC only — no transaction sent.

* **The deployed runtime is unfixed.** Live `specVersion` is **144** on both endpoints; `144` (`6f7094f0`, #661 — Dilithium pubkey caching) touches \*\*no `frame/` file at all. Public HEAD (`specVersion 145`) is byte-identical to the audit tree for this file. So the defect ships in production today.
* **Complete census of both networks** (`evidence/f5_exposure_census.py`, paged + batched read-only queries):

  |                                     |               Heisenberg |                    Planck |
  | ----------------------------------- | -----------------------: | ------------------------: |
  | accounts scanned                    |           151 (complete) |          2,806 (complete) |
  | in the vulnerable `[ED, 2×ED)` band |            **4** (2.65%) |            **28** (1.00%) |
  | balance at risk                     | **4,000,000,002 planck** | **41,748,476,000 planck** |
  |                                     |          = 0.004000 QUAN |           = 0.041748 QUAN |
  | total supply held in accounts       |           7,097,308 QUAN |            7,052,535 QUAN |

  **Combined currently at risk: 45,748,476,002 planck ≈ 0.0457 QUAN**, against \~14.15M QUAN of supply — roughly **0.0000003%**. I am reporting that figure rather than leaving it to be discovered: the monetary exposure is small and I am not going to imply otherwise.
* **At exactly ED the trap is total.** Heisenberg's four sit at precisely `1,000,000,000`, `1,000,000,000`, `1,000,000,001`, `1,000,000,001` planck — *at* the existential deposit, where **every** partial transfer `value ∈ (0, ED)` destroys the whole balance. There is no safe partial send from such an account. Planck's cluster sits at 1.807–1.808e9.
* **This is a snapshot, not lifetime exposure.** Any account that spends down toward ED enters the band, and accounts pass through it routinely. I have not measured that flow and am not estimating it.

**In scope:** `frame/support/src/traits/tokens/fungible/regular.rs` carries **two** commits — `75d32f8` (the vendoring commit) and **`db94e2a`** (the Quantus modification that introduced this). It is Quantus-modified code, not the pristine upstream that the OOS clause excludes.

**What I could NOT establish, and am not claiming:** whether the defect has already fired on either network. A `TotalIssuance` vs `Σ(free + reserved)` reconciliation returned **−249.77 QUAN** — the *opposite* direction from this bug's signature, cause unestablished. It is unusable as a detector and no claim in this report rests on it.

Adding gist link here once again: <https://gist.github.com/almog-shachar/1fa0520b5724fb3248b55a02a366786e>
