# 89328 w a high registration checkout charges more than displayed

**Submitted on Aug 18th 2026 at 16:17:42 UTC by @Griffinf for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89328
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

**Severity claimed: Critical.**

This maps verbatim to the in-scope Critical impact "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient", enumerated example "showing the wrong recipient or **amount** on a transaction the user then approves". The checkout modal displays one total; the registration machine signs and the chain charges a materially larger one; the user approves the flow end to end with no further disclosure.

**Asset:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager> (register-v2 checkout) plus `packages/transaction-manager` (registration machine, imported by it). Verified at the `audit-comp-ready` branch head `1c9b47f18fcddd2e864dfe385c4171061c9811ae`; the program page lists audited revision `63772fd872af472ced58b009499355f3430c2a86`; per the repository's own history it is the fork point (commit `cda79ac`), and the only commits between it and the verified head are a hono dependency bump and its merge, touching none of the cited paths.

### Relationship to published Known Issues

* **QA-03** ("Minor price mismatch in registration and extension modals", filed against the Explorer app, display math) carves this report out verbatim: "Display-level only; **a mismatch between the displayed total and the amount actually charged on-chain would be a new finding.**" That invited case is what I demonstrate, in the Manager app, with a different root cause: two independent unreconciled budget quotes plus a rent-only fallback display.
* **WEB-1188 / WEB-1187** fence a missing warning that sETH is needed for gas, ending in a wallet-level failure. Here the user holds the funds and is silently charged more than displayed; nothing fails.
* **EXP-GAP-006** and the **R3** series cover telemetry and liveness in other packages; no shared root cause.
* Trust boundary: the Rhinestone SDK is an unmodified dependency and I claim nothing against it. Every defect cited is first-party code: the app signs an uncapped remotely quoted value and renders a rent-only fallback with the hedge removed.

The Manager's USDC registration path (the Hybrid Custody Account route) charges the wallet a funding budget, not the rent: `budget = registrationPrice + commitLegCost + registerLegCost`, the leg costs being fees the intent rail charges for the commit and register fills. The checkout screen and the signing machine obtain this budget from **two separate quote round-trips to the remote orchestrator, with no reconciliation between them**, and when the checkout-side quote is slow or fails, the screen degrades to displaying the rent alone as an exact, unhedged "Total".

The developers documented the live magnitude in the permit preflight (`packages/transaction-manager/src/machines/registration/registration.hca.actors.ts` lines 621 to 626): the wallet is charged the "funding shortfall for the whole HCA BUDGET (price + both Rhinestone leg costs), which is materially larger"; "on Sepolia an 8.00 USDC name has run to a \~20.20 USDC budget", roughly 6.10 USDC per leg. At those costs, the name in my proof of concept displays "Total $16.52 USD" while the machine signs and the chain charges about **$28.72, 1.74 times the displayed figure**. The app's own offline fallback estimator agrees the screen is wrong: it prices the legs at 5.00 USDC each (my run logged `source: fallback, total: 26515112`), so the codebase's own constants put the true total at $26.52 minimum while the screen says $16.52 flat.

My executed run shows nothing caps the divergence: with the sign-time quote returning $20.00 per leg, the session that displayed "Total $16.52 USD" signed a permit for 56,515,112 USDC units and the chain charged **$56.52, 3.4 times the displayed total**, through a registration that completed with a normal success screen. The $20-per-leg figure is an injected upper-bound illustration (the local mock orchestrator settles for free, so a nonzero fee had to be supplied); it proves the signed value is whatever the second quote says, bound to nothing the user saw.

## Vulnerability Details

Paths relative to the repository root at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

### Defect 1: displayed figure and signed figure are two independent quotes; the intended invariant binding them is enforced by nothing

The checkout display comes from `apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts` (`staleTime` 60 s, `retry: 1`), rendered through `computeRegistrationFunding` and `PaymentTotalRow`. The comment at `TokenPickerContent.tsx` lines 148 to 154 states the intended invariant: the budget query "Mirrors what `registrationUi.machine.ts` hands the machine for `START_REGISTRATION`, so the figure on this screen is the one the permit is sized from."

It is not. `START_REGISTRATION` carries only `price: event.totalPrice`, the rent (`apps/manager/src/features/register-v2/state/registrationUi.machine.ts` lines 544 to 559); no budget value crosses from screen to machine. The machine's `computingHcaBudget` state (`registration.machine.ts` lines 604 to 653) notes "A caller-supplied `hcaBudget` override wins", and because the app never supplies one, it invokes `estimateHcaBudget` **again**: two fresh `prepareTransaction` round-trips to the remote orchestrator.

`signingFundingPermit` (lines 704 to 733) then signs the EIP-2612 permit for:

```ts
value: bigintMax(
  context.hcaBudget! - (context.hcaUsdcBalance ?? 0n),
  0n,
),
```

There is zero comparison against any displayed figure. The only check is the balance preflight at `registration.hca.actors.ts` line 627, which verifies the wallet merely holds the amount. The preflight's own error string concedes the divergence ("The funding amount covers the registration price plus the execution costs of both the commit and register legs, so it is larger than the price shown at checkout."), and that sentence is shown only to users too poor to be overcharged.

### Defect 2: the rent-only fallback removes even the hedged figure

`TokenPickerContent.tsx` lines 434 to 438:

```ts
const requiredAmount = funding?.walletDebit ?? pricingData
const displayTotal = funding?.total ?? pricingData
```

The comment at lines 165 to 167 states the behavior: the funding quote is "Absent until the quote lands, and permanently absent if it fails", in which case "the screen falls back to showing the rent alone rather than blocking on a flaky quote".

With `funding` null, `PaymentTotalRow` receives `isEstimate` false and renders "Total $16.52 USD" with no "up to" hedge and no Network cost row (`PaymentTotalRow.tsx` lines 17 to 44 print "up to" only when `isEstimate` is true; the component's doc comment says the headline figure is "what actually leaves the wallet"). The click path proceeds without a quote: the availability mutation re-fetches the budget with `.catch(() => null)` (`TokenPickerContent.tsx` lines 229 to 233, comment: "A quote failure is not a funding failure. Fall through and let the machine (and its own pre-permit balance check) surface the problem."). The machine re-quotes; if its quote succeeds, the permit is signed for the full budget shortfall. Displayed: exact rent, asserted as the total. Signed and charged: rent plus both legs.

**The money is really gone.** The permit funds the HCA up to the budget and the leg costs are paid to the intent rail as execution fees. The budget "carries no buffer any more", being "the sum of two real quotes plus the price" (`registration.hca.actors.ts` lines 476 to 478), so the delta between displayed and charged is consumed as fees, not parked recoverably.

## Impact

{% stepper %}
{% step %}

### No attacker (Demonstrated mechanism, Source-confirmed live magnitude)

On the live orchestrator the leg costs can be materially nonzero; the developers document an $8.00 registration reaching a \~$20.20 budget. Any checkout-quote latency or transient failure (the query retries once, then is "permanently absent") puts the flat rent-only Total on screen; a user who clicks through is charged the full budget. At documented costs: $28.72 charged against $16.52 displayed.
{% endstep %}

{% step %}

### Selective disruption

An adversary capable of selectively disrupting the checkout quote request can force the rent-only display while allowing the later sign-time quote to succeed. No wallet compromise, no first-party code touched.
{% endstep %}

{% step %}

### No cap at all (Source-confirmed structural absence; not demonstrated)

Even with the healthy hedged "up to" figure shown, the machine signs whatever the second quote returns, bounded only by the wallet's USDC balance. I did not execute this variant; the executed $56.52 run proves the mechanism, and no code path exists that would stop it.
{% endstep %}
{% endstepper %}

The wallet's EIP-2612 prompt is not a defense: the signature contains the raw six-decimal value (56515112 in the demonstrated run); wallet presentation varies and does not provide an application-level re-confirmation of the checkout total. In my run the stock test fixture auto-signs with no prompt surface at all. The impact wording presumes exactly this shape: the app shows the wrong amount "on a transaction the user then approves".

## Link to Proof of Concept

<https://gist.github.com/aussinfosec/ea6e9b95178275a3fb9f3f67d16e0585>

## Proof of Concept

Executed end to end on 2026-08-18 against the repo's own local e2e stack: the pinned Anvil Sepolia fork plus the pinned mock orchestrator image from `e2e/infra/docker-compose.yml`, the Manager dev build with the repo's own `.env.ci` values, and the stock Playwright fixture (headless EIP-6963 wallet, first Anvil dev account `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`; its HCA was deployed once through the real factory and its USDC balance zeroed pre-run so the flow is a fresh user's). Everything local; no production contact.

A self-contained reproduction bundle (both Playwright specs and the full run log) is available at <https://gist.github.com/aussinfosec/ea6e9b95178275a3fb9f3f67d16e0585> (secret gist).

### Steps from a clean checkout

{% stepper %}
{% step %}

### Clone and install

```sh
git clone https://github.com/immunefi-team/audit-comp-ens && cd audit-comp-ens && git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
```

Then run:

```sh
pnpm install
```

{% endstep %}

{% step %}

### Bring up the e2e stack

Bring up the e2e stack (`e2e/infra/docker-compose.yml`; see `e2e/README.md`).

Reproduction notes: the mock orchestrator does not execute `setupOps`, so deploy the HCA for the test account once via the `StandaloneHCAFactory` first (the CI snapshot image with it pre-deployed is not publicly pullable); a foreign process bound to `127.0.0.1:8545` will shadow Docker's port publish, so use `http://[::1]:8545` or a clean host.
{% endstep %}

{% step %}

### Start the Manager dev server

Start the Manager dev server on `:3000` against the local stack.
{% endstep %}

{% step %}

### Run the baseline and PoC specs

Copy the attached `zz-tx1-baseline.spec.ts` and `zz-tx1-poc.spec.ts` into `e2e/projects/manager/tests/` and run them with the project's Playwright setup (baseline first).

Artifacts are written to `./tx1-poc-evidence` (or `POC_EVIDENCE_DIR`).
{% endstep %}
{% endstepper %}

The baseline spec drives the stock flow and passes green: the healthy checkout shows "Network cost $0.00" and "Total up to $16.52" (`baseline-checkout.png`), because the mock orchestrator quotes zero fees.

The PoC spec then intercepts **only at the app-to-orchestrator HTTP boundary**; no first-party code is modified:

* **Checkout phase:** `POST /orchestrator/intents/route` is aborted (the network-flake equivalent the code is written to degrade through). Observed: the budget query retried once, degraded (console: `HCA budget: {source: fallback, total: 26515112n ...}`) and the query actor threw; `funding` became null; the modal rendered **"Total $16.52 USD"**, exact rent, no "up to", no Network cost row, sidebar total card the same (`poc-1-checkout-rent-only.png`). Four quote calls were blocked in total, including the click-path re-fetch, all logged.
* **Sign phase:** quotes pass through with `intentCost.tokensSpent` set to $20.00 per leg (the mock settles free; a live orchestrator returns real nonzero costs per the developer comment quoted above). Observed: machine console `HCA budget: {source: quote, total: 56515112n, commitCost: 20000000n, registerCost: 20000000n, registrationPrice: 16515112n}`; the funding permit was signed for the full 56,515,112 with no reconciliation against anything displayed; the commit batch filled.

On-chain outcome (fork, block 11515495, tx `0xb9aad6a9aeb80e3f630101d6eba6cd484184d1f827e5b6a8087b091cc543ae8c`): `USDC.Transfer(wallet -> HCA, 56515112)`; wallet USDC balance 983,484,888 to 926,969,776 (delta exactly -56,515,112, matching the permit calldata value `0x35e5a28`); "Registration Complete" banner (`poc-2-success.png`). The user experiences a normal happy path end to end.

Attached evidence: `poc-log.txt` (timestamped blocked quotes, both budget console lines, permit and transfer calldata, before and after balance reads), the four screenshots, and `baseline-orchestrator.jsonl` (quote request and response bodies, including the zero-fee mock shape that necessitated the injection). Evidence tiers: the display-sign-charge divergence is **Demonstrated**; the live-orchestrator magnitude (roughly $6.10 per leg, $28.72 for this name) is **Source-confirmed** by the developers' own comment and the app's own fallback constants; the up-to-wallet-balance variant is **Source-confirmed structural absence, not demonstrated**.
