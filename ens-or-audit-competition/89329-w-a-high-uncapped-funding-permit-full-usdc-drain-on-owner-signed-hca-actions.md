# 89329 w a high uncapped funding permit full usdc drain on owner signed hca actions

## #89329 \[W\&A-High] Uncapped funding permit → full USDC drain on owner-signed HCA actions

**Submitted on Aug 18th 2026 at 16:19:58 UTC by @bdr4k for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89329
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>
* **Impacts:**
  * Direct theft of user funds
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

### Description

When an action runs through the user's Hidden Contract Account (HCA), the client tops the HCA up with USDC pulled from the owner's wallet via an EIP-2612 `permit` + `transferFrom`, sized inside `planHcaIntentFunding`:

```ts
// hca-intent-funding.ts
const quotedFeeUsdc = await quoteIntentUsdc({ … })      // :234
const shortfall = quotedFeeUsdc - balance               // :255
const permit = await signFundingPermitActor({
  wallet: ownerAddress, hca, value: shortfall, …        // :257-262  (spender = HCA)
})
// calls: permit(wallet, HCA, shortfall) + transferFrom(wallet, HCA, shortfall) + …  (:271-304)
```

`quotedFeeUsdc` is taken **verbatim from the Rhinestone orchestrator's quote**, with no upper bound:

```ts
// hca-intent-funding.ts:137
return readUsdcSpend(prepared.intentRoute?.intentCost, chain.id)

// registration.hca.actors.ts:122-141 — returns the orchestrator's own number, no ceiling
return BigInt(entry.locked ?? '0') + BigInt(entry.unlocked ?? '0')
```

The **only** bound before the wallet is asked to sign is a *floor* check, whose effect is to cap the permit at the wallet's entire balance:

```ts
// registration.hca.actors.ts:627 — this is the drain ceiling, not a protection
if (walletBalance < input.value) throw new Error('Insufficient USDC …')
```

So `permit.value == min(quotedFeeUsdc − balance, walletBalance)`. If the orchestrator quote ≥ the wallet balance, the client asks the user to sign a permit authorizing their **entire USDC balance** to be moved into the HCA and spent as the intent's fill fee.

**Why `set-primary-name` is the fully-exploitable variant:** the registration path is protected on-chain by a session-validator refund cap (`MAX_REFUND_AMOUNT = 100 USDC/leg`, `manifest.ts:205`), so an inflated permit only over-funds the *user-owned* HCA and the excess is recoverable. But `setPrimaryNameWithHca` runs **owner-signed** and, per its own comment, "applies no action policy at all — which is also what makes the funding permit legal here, unlike inside the session-signed commit leg" (`setPrimaryName.ts:181-198`). There is **no on-chain refund cap** on this path, so the inflated fee is actually collected by the relayer — a non-recoverable loss.

### Attack scenario / preconditions

{% stepper %}
{% step %}

### User performs an owner-signed HCA action

User performs a cheap owner-signed HCA action (e.g. *Set primary name*, real cost ≈ 1 USDC).
{% endstep %}

{% step %}

### Orchestrator returns an inflated quote

The Rhinestone orchestrator's `prepareTransaction` response returns `intentCost.tokensSpent[usdc] ≈ the user's full wallet USDC balance`.
{% endstep %}

{% step %}

### Client sizes and requests the permit

`planHcaIntentFunding` sizes the permit to that value (`spender` = the user's own HCA — reads as benign), the floor check passes, and the user is prompted to sign.
{% endstep %}

{% step %}

### Full balance is transferred and spent

On approval, `transferFrom` moves the full balance into the HCA and the intent settles the inflated fee to the relayer. **Funds lost.**
{% endstep %}
{% endstepper %}

**Preconditions** step 2 requires the orchestrator — a trusted third-party service — to be **malicious, compromised, MITM'd, or buggy**; a plain external attacker cannot trigger it. Partial mitigation: the EIP-2612 `value` *is* shown in the wallet prompt (though `spender = your own HCA` reads as safe, and blind-signing is common). Absent an adversarial orchestrator, an attacker-independent **Sepolia gas-price spike** still inflates the un-clamped primary quote and causes a *bounded, non-recoverable* overcharge on this no-refund-cap path.

### Impact

Loss of user USDC. Worst case (adversarial orchestrator): the wallet's entire USDC balance. Realistic-attacker case (gas manipulation / quote bug): a bounded, non-recoverable overpayment on the owner-signed path.

### Proof of Concept

Two PoCs (both included, both pass):

{% stepper %}
{% step %}

### End-to-end, on-chain

`poc/funding-permit-cap/e2e/` (viem + anvil). Deploys a real EIP-2612 USDC on a live local EVM, sizes the permit with the audited client's own logic from an orchestrator quote, has the victim sign the **real** EIP-2612 `permit` (viem `signTypedData`), and executes the exact `permit` + `transferFrom` calldata the client bundles.

Result:

```
── HONEST orchestrator (quote 0.9 USDC) ──
   permit value signed: 0.9 USDC   victim: 5000 -> 4999.1 USDC   relayer collected: 0.9 USDC
── MALICIOUS orchestrator (quote == full balance) ──
   permit value signed: 5000 USDC  victim: 5000 -> 0 USDC        relayer collected: 5000 USDC
```

Run:

```bash
cd poc/funding-permit-cap/e2e && npm install && forge build && node e2e.mjs
```

{% endstep %}

{% step %}

### Dependency-free logic

`poc/funding-permit-cap/poc.mjs`. Isolates the sizing defect (`readUsdcSpend` + `planHcaIntentFunding` + the floor guard) with no chain: `permit.value` tracks the orchestrator number with no ceiling other than the wallet balance; a quote *above* the balance reverts on the floor guard.

Run:

```bash
node poc/funding-permit-cap/poc.mjs
```

{% endstep %}
{% endstepper %}

### Recommended remediation

Apply a client-side sanity ceiling to `shortfall` before requesting the signature — an absolute cap and/or a small multiple of the independently-known expected cost (a local gas-model upper bound already exists in `budget.ts`). Reject or prominently warn when the orchestrator quote is disproportionate to the action. The registration path is guarded on-chain; this owner-signed path needs the client-side backstop because nothing on-chain bounds it.

### Link to Proof of Concept

<https://gist.github.com/BDR-Pro/5170a0dfa6366a67ae9fd444d8879916>

### Proof of Concept

## PoC — uncapped funding permit → USDC drain

Faithful, dependency-free reproduction of the client's permit-sizing logic, showing the EIP-2612 permit `value` is taken from the Rhinestone orchestrator's quote with **no ceiling** — the only guard caps it at the wallet's entire balance. On the owner-signed `set-primary-name` path there is no on-chain refund cap, so an inflated fee is collected (non-recoverable), not just parked in the user-owned HCA.

### Run

```bash
node poc.mjs
```

Expected:

```
victim wallet USDC: 5000.000000 USDC   HCA USDC: 0.000000 USDC

(a) honest quote (0.90 USDC):
    permit authorizes: 0.900000 USDC  -> expected, fine

(b) malicious quote (== full wallet balance):
    permit authorizes: 5000.000000 USDC  spender=HCA (user-owned)
    transferFrom moves 5000.000000 USDC WALLET -> HCA, then paid to relayer as the fill fee

(c) quote just above balance -> reverts (walletBalance < value guard)
    => the only bound is a floor at the wallet balance; ...

RESULT: permit.value tracks the orchestrator quote with NO ceiling other than the
        wallet balance (drain ceiling = 100% of USDC). ...
```

### Scope of this PoC

This isolates the **client-side defect** (the missing ceiling). A live end-to-end drain additionally requires the Rhinestone orchestrator to return an inflated `intentCost` (compromise / MITM / bug) — see the *Preconditions* section of `submission.md`, ] The reproduced logic maps line-for-line to:

* `readUsdcSpend` — `registration.hca.actors.ts:122-141` (returns `locked+unlocked`, no cap)
* sizing — `hca-intent-funding.ts:234-269` (`shortfall = quotedFeeUsdc - balance`)
* the only guard — `registration.hca.actors.ts:627` (`walletBalance < value` floor)
* owner-signed, no refund cap — `setPrimaryName.ts:181-198`
