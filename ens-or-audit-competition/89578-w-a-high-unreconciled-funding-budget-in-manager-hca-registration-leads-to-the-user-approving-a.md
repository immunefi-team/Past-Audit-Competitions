# 89578 w a high unreconciled funding budget in manager hca registration leads to the user approving a permit for 2 5 the total the checkout displays

**Submitted on Aug 19th 2026 at 23:42:19 UTC by @Ispolin11 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89578
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

On the standalone-HCA registration route — the one the Manager app ships with — the wallet signs a single EIP-2612 permit sized from the funding budget (`registrationPrice + commitLegCost + registerLegCost`, less whatever the HCA already holds), and the application reports `registrationPrice` alone. Two consequences follow, and they are not equally conditional. The success receipt captions the registration price **"Total Paid"** in every case, so the application never states the amount that actually left the wallet. And while the budget quote is still outstanding — a state that needs no failure of any kind — the checkout screen states that same price as an *exact* total, the "up to" hedge that would mark it provisional dropped, with the register button already live, so a user who acts there approves a permit for **2.5×** the figure the screen just gave them — the ratio when the HCA is empty, a state [`registrationFunding.ts#L24-L31`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.ts#L24-L31) calls *"the common case"*. Nothing anywhere in the application compares the number it displayed against the number it signs for: the two come from separate quotes, and the one that sizes the permit is accepted with no ceiling and no plausibility check.

## Vulnerability Details

Every line reference below is against commit `1c9b47f` (the tip of `audit-comp-ready`), and every link is addressed by that commit rather than by the branch, so the line numbers cannot drift. The app is configured for Sepolia only ([`wagmi.ts#L80`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/lib/wagmi.ts#L80), `chains: [sepoliaWithEns]`), so the USDC figures throughout are that deployment's. That is the deployment as it stands; the defect is in the application's logic — one state standing for two, two quotes that are never reconciled, a value accepted without a bound — and none of that is chain-specific, so a mainnet deployment would move the figures below and not the structure. Each table is the output of running the shipped code with the surrounding module injected — no network, no orchestrator; the PoC section carries the files and the commands.

### 1. Two numbers, and only one of them reaches the user

The permit is sized from the budget, in [`registration.machine.ts#L723`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.machine.ts#L723):

```ts
value: bigintMax(
  context.hcaBudget! - (context.hcaUsdcBalance ?? 0n),
  0n,
),
```

and the budget itself is the sum of three parts, in [`budget.ts#L330`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/budget.ts#L330):

```ts
const total = commitCost + registerCost + registrationPrice
```

The number the application carries into the registration is `registrationPrice` alone. It enters at [`TokenPickerContent.tsx#L203-L221`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx#L203-L221), where `startRegistration` sends `totalPrice: pricingQuery.data.rawPrice` together with `basePriceNumber` and `premiumPriceNumber` — three fields, all from the pricing query, none from `budgetQuery`, which is in scope a few lines above in the same component ([`:163`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx#L163)). From there it travels through [`registrationUi.machine.ts#L546-L560`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/state/registrationUi.machine.ts#L546-L560) as `price: event.totalPrice`, and that event carries **no budget field at all**. The checkout screen does show the budget when it has one — section 3 sets out exactly when it does not.

The two figures are not ones I picked. The project's own unit-test fixture fixes them:

```ts
// apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.test.ts:6-10
/** The live failure this gate was built for: 20.000000 held, 20.196054 needed. */
const PRODUCTION_FAILURE = {
  total: 20_196_054n,
  registrationPrice: 8_000_021n,
}
```

To be exact about what that records: it is a **funding shortfall** — a wallet holding `20.000000` against a budget of `20.196054` — not a display failure. I cite it only for what it does establish, which is the scale of the two numbers on a real registration: the project itself puts the rent at `8.000021` and the budget at `20.196054`, a ratio of **2.5245**. The PoCs accompanying this report use `12_200_000` for the two legs, the figure from [`registration.hca.actors.ts#L621-L625`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L621-L625) — *"on Sepolia an 8.00 USDC name has run to a \~20.20 USDC budget"* — which gives `20.200021` and the same 2.52.

### 2. Root cause: one `null` carries two different meanings

`computeRegistrationFunding` returns `null` for two states that require opposite handling, and [`registrationFunding.ts#L51-L57`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.ts#L51-L57) documents both in a single sentence:

> Returns `null` when no budget has been quoted yet (**or the quote failed**), in which case callers should show the price alone rather than block: a flaky orchestrator is not evidence the user cannot pay.

The first meaning — "this route has no budget" — is the pure-EOA renewal flow, which reuses the same components ([`renew/…/TokenPickerContent.tsx#L9`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/workflow/pricing/components/TokenPickerContent.tsx#L9)) and where the rent genuinely is the whole cost. The second — "this route has a budget and we could not obtain it" — is the HCA route. The consumer cannot tell them apart, so it applies the first behaviour to both.

Four separate things fall back on that single value in [`TokenPickerContent.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx#L434-L438):

```ts
const requiredAmount = funding?.walletDebit ?? pricingData   // :434 — the checkout gate
const displayTotal   = funding?.total      ?? pricingData    // :438 — the headline figure
…
{(funding || isQuotingFunding) && <NetworkCostRow … />}      // :491 — the network cost line
…
<PaymentTotalRow isEstimate={!!funding} total={displayTotal} />  // :575 — the "up to" hedge
```

The headline figure and the affordability gate both drop to the rent the moment `funding` is `null`, and the hedge goes with them, since it is `isEstimate={!!funding}`. The network cost row survives one of the two states: gated on `(funding || isQuotingFunding)`, it renders with an em dash while the quote is in flight and disappears entirely once the quote has settled into error.

### 3. What the screen states in each of the three states

Rendering the shipped `TokenPickerContentBase` at one rent under all three budget states, with a wallet holding `$10.00` — above the rent, below the budget, which is precisely the wallet the gate exists to stop:

| Budget state    | Total shown | `up to` hedge | Network cost row | Register button (wallet `$10.00`) |
| --------------- | ----------- | ------------- | ---------------- | --------------------------------- |
| quote present   | `$20.20`    | yes           | yes              | **disabled**                      |
| quote in flight | `$8.00`     | no            | yes (value `—`)  | **enabled**                       |
| quote failed    | `$8.00`     | no            | **no**           | **enabled**                       |

The first row is correct, and I want to be explicit about that: with a quote in hand this screen does its job. The defect is what the two unquoted states do.

The button in that first row is disabled *because* the wallet holds `$10.00` — the gate is working. A wallet holding the whole budget clears all three states, and that is the case the impact runs through: the user who can pay is exactly the user who proceeds. Rendered at a `$50.00` balance, the in-flight state gives `total=$8.00`, `hedged=false`, `canProceed=true`.

The **in-flight** row matters most, because it requires no failure of any kind. It is the state of every first visit to this screen while the quote is outstanding, and that quote is deliberately expensive — [`hcaBudget.query.ts#L36-L41`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts#L36-L41) notes it costs "TWO `prepareTransaction` round trips to the Rhinestone orchestrator".

It is also not a one-shot race on page load. The budget query is keyed on the duration and on the primary-name opt-in ([`hcaBudget.query.ts#L136-L145`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts#L136-L145)), the second with the code's own note — *"Keyed, not just passed: toggling the primary-name switch changes the quote, so it has to refetch rather than serve the other variant."* That switch is rendered on this same screen ([`TokenPickerContent.tsx#L338-L341`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx#L338-L341)). A different key is a different cache entry, and this query configures no `placeholderData`, so the first time a combination is selected there is nothing to show and the screen re-enters the row above: the total back at the rent, the hedge gone, the button live. Returning to a combination already fetched does not reopen it: that key has data, so `funding` is not `null` and both disclosures are back. The window belongs to keys with nothing cached — the first visit, and the first selection of each new combination. The PoC confirms both keys differ.

The obvious objection is that the quote lands a moment later and the screen corrects itself. That is right, and it bounds this row to users who act inside the window rather than to all of them. It does not close it: the window opens on every first visit and again on each new combination of those inputs, the button is live throughout it, and nothing on the screen marks the figure as provisional — the one element that would say so, the `up to` hedge, is precisely what is missing. I have not measured how long the window stays open in production; that depends on the orchestrator's response time and is not observable from the audit environment.

The **failed** row is reachable because `isQuotingFunding` is `budgetQuery.isLoading` ([`:358`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx#L358)) and the query carries `retry: 1`. On the installed React Query `5.90.12`, a settled `error` state gives `isLoading: false` with `data: undefined`, so `funding` and `isQuotingFunding` are false together and the last disclosure goes with them.

That row is narrower than the one above it, and the reason is worth stating plainly. The screen's query and the registration machine call the same `estimateHcaBudgetActor`, and that actor throws outright when the budget could not be quoted ([`registration.hca.actors.ts#L481-L487`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L481-L487)). A persistent failure therefore fails both: the machine errors and no permit is signed at all. This row produces the divergence only when the failure is transient — the screen's two attempts fail and the machine's later, independent attempt succeeds. The in-flight row needs no failure anywhere, which is why it, and not this one, carries the claim.

### 4. The receipt reports the rent in every case

[`RegistrationDetails.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/registering/components/RegistrationDetails.tsx#L51-L55) computes its figure from `confirmedData`, which holds `basePriceNumber`, `premiumPriceNumber` and `totalPrice` — all rent, no budget:

```ts
const totalPrice =
  basePriceWithoutDiscount + details.premiumPriceNumber - discountAmount   // :51-55
```

and captions it at [`:122-126`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/registering/components/RegistrationDetails.tsx#L122-L126):

```tsx
<Trans>Total Paid</Trans>
```

The component renders during registration ([`RegisteringStep.tsx#L221`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/registering/RegisteringStep.tsx#L221)) and again on success ([`SuccessStep.tsx#L10`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/result/SuccessStep.tsx#L10)), and it is route-agnostic — the same component serves the EOA registration, where the caption is correct.

State this precisely, because the obvious phrasing is wrong in one direction: **the receipt states the amount that left the wallet in no case at all.** Running the shipped machine at three HCA balances. The debit and the permit count come from the run; the receipt column is the rent — the figure this section has just shown the row computes — set beside them for comparison:

| HCA balance | Wallet debited | Receipt states | Permits signed |
| ----------- | -------------- | -------------- | -------------- |
| `$0.00`     | `$20.20`       | `$8.00`        | 1              |
| `$5.00`     | `$15.20`       | `$8.00`        | 1              |
| `$20.20`    | `$0.00`        | `$8.00`        | 0              |

It understates against an empty HCA and overstates against a fully funded one. The figure printed is the rent every time, because the rent is the only number `confirmedData` carries. This half needs no failure, no race and no particular timing.

### 5. No reconciliation exists, and the two figures come from different quotes

Feeding the shipped machine a budget ten times the rent produces a permit for `$80.00` against a displayed `$8.00`, and the machine reports **no error** — nothing anywhere compares the two. The displayed price is not even an input to the estimator; its captured input keys are `name, duration, publicClient, chainId, signer, sessionEnable, primaryName`.

The two figures are also independently derived. The UI's quote is never handed to the machine: `context.hcaBudget` is `undefined` at the moment `START_REGISTRATION` lands, and the machine re-quotes for itself at [`registration.machine.ts#L607-L636`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.machine.ts#L607-L636).

Two comments in [`hcaBudget.query.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts) speak to this and do not agree with one another. At `:29-33`:

> This runs the SAME estimator the registration machine runs in `computingHcaBudget`, **so the number shown at checkout is the number the machine will size the permit from.**

Eight lines below, at `:36-41`:

> It only has to be fresh enough to catch a gas regime change between opening the confirm screen and pressing register; **the machine re-quotes for real either way.**

A third sits in the component that renders the figure, at [`TokenPickerContent.tsx#L148-L154`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx#L148-L154):

> Mirrors what `registrationUi.machine.ts` hands the machine for `START_REGISTRATION`, **so the figure on this screen is the one the permit is sized from.** Quoting without it under-funds and the permit preflight then rejects a wallet this screen just told the user was sufficient.

One of the three describes the shipped behaviour — the machine re-quotes. The other two state an equality that nothing enforces. The closing clause of the third is the team's own description of the failure mode: a wallet that *"this screen just told the user was sufficient"* being rejected further down. I am not claiming the team promised one thing and built another — I am pointing at the gap the re-quote comment leaves open and the other two assume closed: the numbers are separately derived, and no code anywhere reconciles them. A larger re-quote is therefore signed without objection — a budget of `$24.00` against a screen that had said `up to $20.20`, machine error `none`. I did not observe a gas move producing that divergence in the wild; what the PoC establishes is that the code accepts it if one occurs.

### 6. Every bound sits on the branch that never funds anything

`budget.ts` carries an explicit sanity ceiling, and its comment says exactly what it is for:

```ts
// budget.ts:133-137
/**
 * Sanity ceiling on the per-leg gas price used by the FALLBACK model, to stop
 * a transient base-fee spike from ballooning the estimate. …
 */
const FALLBACK_MAX_GAS_PRICE_WEI = 5_000_000_000n
```

That guards the number the application computes for itself. The other branch — `quotedCommit.value` / `quotedRegister.value`, returned by the orchestrator and taken verbatim at [`budget.ts#L322-L323`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/budget.ts#L322-L323) — reaches `total` with no ceiling, no plausibility check, and no comparison against the price the user was shown. Driving the shipped `estimateHcaBudget` down both branches. The two are provoked by different amounts, named in the table, because what separates them is not the size of the ratio but whether it can grow at all:

| Branch                                                                                | Benign input | Hostile input | Ratio     |
| ------------------------------------------------------------------------------------- | ------------ | ------------- | --------- |
| the model the app computes — gas price `1 gwei` → `500 gwei` (**500×**)               | `$10.70`     | `$21.50`      | **×2.01** |
| the value the app is given — quote `6_100_000` → `5_000_000_000` per leg (**\~820×**) | `$20.20`     | `$10008.00`   | **×495**  |

The ceiling is 5 gwei, and it absorbs everything above itself: run the computed branch at `5 gwei` and again at `500 gwei` and it returns the identical `$21.50`.

And the unbounded value reaches the permit unchanged: carried into the shipped machine it is signed for in full — `$10008.00` against a displayed `$8.00`, machine error `none`.

I am not suggesting the orchestrator returns implausible values, and this is not a claim about Rhinestone. The claim is only about this code: whatever the quote returns is what the permit is signed for.

The sharp form of the asymmetry is not that one branch is clamped and the other is not. It is that **the clamped branch never funds anything**. [`registration.hca.actors.ts#L481-L487`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L481-L487) rejects it outright:

```ts
if (breakdown.source !== 'quote') {
  throw new Error(
    `HCA budget could not be quoted (source: ${breakdown.source}). ` +
      `Refusing to size the funding permit from the fallback model. ` +
      …
  )
}
```

So there are **two** guards against the budget the team distrusts — a gas-price clamp and a hard refusal — and **none at all** on the value they trust, which is the only one that ever sizes a permit.

### 7. What the team decided, and what those decisions cover

The `null` behaviour is not accidental, and there are two decisions on record. The first is in the project's own test:

> [`registrationFunding.test.ts#L166`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.test.ts#L166) — *"A flaky orchestrator must fall back to showing the price, not to a block."*

The second is in the component that renders the hedge, [`PaymentTotalRow.tsx#L4-L10`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/PaymentTotalRow.tsx#L4-L10):

> With a funding budget quoted that is `rent + networkFee`, and the fee half is an estimate, so the figure is an upper bound: hence "up to". **Without a budget the total is exact rent and the hedge would be a lie, so it is dropped.**

I am not reporting that the screen should block, and I am not disputing either decision on the route it was written for. On the renewal and pure-EOA routes there is no budget, the total genuinely *is* exact rent, and a hedge there would indeed be a lie.

What neither decision covers is the HCA route, where "without a budget" does not mean there is no budget — it means the budget exists and could not be obtained. There the rent is not the total, so the hedge is not a lie; it is the only qualification on a figure that needs one. The first decision is about **not blocking**, and says nothing about also removing the hedge, hiding the network cost row, and moving the affordability gate from the budget to the rent — all of which the same `null` does. The second is about a route where the premise "the total is exact rent" holds, applied through shared components to one where it does not.

It is worth noting where the testing stops. `computeRegistrationFunding` has **13 unit tests**, including one asserting the two-state overload explicitly (`returns null when no budget has been quoted`). Not one of them asserts what a consumer renders, or gates on, when that `null` arrives.

### 8. This is the route the app ships with

The finding applies to the standalone-HCA path and not to the pure-EOA one. That is not a narrowing — the EOA path is a debug switch. [`apps/manager/.env.example`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/.env.example) ships:

```
# When `true`, force the transaction manager to use plain EOA signing only
# (no Rhinestone smart account) … Set `true` only for
# local EOA-only debugging on forks without bundler/relayer infrastructure.
VITE_FF_USE_EOA=false
```

At that value, and with the variable unset, `isFeatureEnabled('USE_EOA')` evaluates `false`. The budget query whose absent result drives this entire finding is `enabled` for the `rhinestone` signer and disabled for the EOA one.

One qualification I should make myself: `.env.example` is an example file, not the production deployment's configuration, which is not in the repository. What supports the reading beyond it is that the whole budget, permit, `NetworkCostRow` and preflight apparatus exists only for the HCA path — none of that is built for a debugging switch.

### 9. What a fix has to touch

Noting this because the shape of the fix bears on how the defect reads: it is not a null check. Give the funding summary a third state instead of overloading one `null` — say `RegistrationFunding | 'no-budget-on-this-route' | 'unquoted'` — so `TokenPickerContentBase` can keep today's behaviour for the renewal route while telling the user, on the HCA route, that the network half is unknown. `RegistrationDetails` needs the budget threaded through `confirmedData`, or the row re-captioned to something the rent actually answers. Neither closes the third gap: as long as the permit is sized from a value that is never compared against the figure shown, the two can part again for any reason.

## Impact Details

### What the user authorises versus what the application states

The permit is an EIP-2612 approval over the user's USDC, honoured by a `transferFrom(wallet → HCA)` executed inside the session-signed commit batch. At the project's own figures the rent is `$8.000021` and the budget `$20.196054` — a factor of **2.5245**, of which about 60% is the network half.

At the moment of approval the two figures diverge in the unquoted states of section 3 — chiefly the in-flight one, which needs no failure anywhere, and the failed one only when the failure is transient, for the reason given there. In those states the screen states the rent as an exact total, with the hedge that [`PaymentTotalRow.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/PaymentTotalRow.tsx) exists to add explicitly dropped, and the permit that follows is signed for the budget. In absolute terms that is `$12.196033` on an `$8.000021` name — the difference between the two figures, and a number the project's own test asserts directly as the network half, at [`registrationFunding.test.ts#L20-L22`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.test.ts#L20-L22). In the quoted state the screen is right and there is no divergence at signature — that state is not part of this claim.

Where they diverge in every case is afterwards: the receipt captions the rent "Total Paid" regardless, so the amount that left the wallet is stated nowhere in the application.

Two further bounds are absent rather than wrong. Because the two figures are separately quoted and never compared, the authorised amount is not bounded by the displayed one even when the screen is correct — a re-quote of `$24.00` behind a displayed `up to $20.20` is signed with no error. And because the orchestrator's value is accepted without a ceiling, nothing in the application limits how far the two can part: driven with an implausible quote, the permit is signed for `$10008.00` against a displayed `$8.00`.

### Where the money goes, stated plainly

The surplus is **not stolen and not sent to a third party**. It lands in the user's own Hybrid Custody Account, and a later registration draws it down: the permit tops the HCA up rather than re-funding it, so a subsequent budget is reduced by whatever is already there.

Two things bound that mitigation:

1. **The application never shows the user that balance.** `hcaBalance` is read only inside the budget query and the registration machine, and I found no place in the UI that renders it.
2. **The application offers no way to take it back.** I found no withdraw, sweep, or transfer-out path for the HCA's USDC in `apps/manager`, `packages/smart-account`, or `packages/transaction-manager`. A user who registers one name and stops is left with the difference sitting in a smart account whose balance the app does not display. It is their own account and it is not cryptographically lost — but within the audited application it is neither visible nor recoverable.

### There is no attacker, and the selected impact does not require one

Nothing here is attacker-influenced: no third party chooses the name, the timing, or the amount. The wallet's own signing prompt carries the permit `value`, so the true figure is disclosed at the moment of signature — by the wallet, which is the only place in the flow where it appears at all. The application states it nowhere, before the signature or after it.

I state these plainly because they are the facts that argue against a higher rating, and the assessment should be made on the complete picture. They do not remove the impact. The in-scope row selected for this report is:

> **Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient**, such as: … *showing the wrong recipient or **amount** on a transaction the user then approves*

That row describes what an application does to its own user. It names `amount` explicitly, in the program's own example, and a wrong amount cannot be a wrong recipient — so the word carries independent meaning. The sequence it describes is the one demonstrated here: the checkout states an exact total, the user clicks Register, and the next thing that happens is a signature request for 2.5× that figure.

### My own assessment of severity

I assess this as **High**, not Critical, and I would rather say so than let it be inferred. The impacts table offers no row that describes a wrong payment figure below the Critical tier, which is why the Critical row is the one selected; but the Critical rows in this program otherwise describe loss or takeover, and here the recipient is correct, nothing is stolen, and the wallet prompt discloses the true value before the user signs. What carries the finding above an informational note is the class of the defect rather than a loss: an externally supplied value sizes a signed token authorisation with no bound and no comparison against the figure presented to the user, in an application that demonstrably knows such bounds are necessary and applied two of them to the branch that never funds anything.

### Not the same as the two adjacent known issues

**WEB-1187 / WEB-1188** — *"The checkout flow does not warn that sETH is needed for gas; the user can proceed until the wallet itself rejects."* Those describe a missing warning about the native gas token on the EOA checkout, ending in a transaction the wallet refuses. This is a USDC permit on the standalone-HCA route, where a specific figure **is** stated, the transaction **succeeds**, and the amount authorised is 2.5× what was shown. Different currency, different route, opposite symptom.

**QA-03** — *"In the registration and extension modals, the per-year discounted price and the displayed total do not always agree (display-level only)."* That entry then carves out this report by name: *"A mismatch between the displayed total and the amount actually charged on-chain would be a new finding."* To be precise about the boundary: the arithmetic inside `RegistrationDetails` is internally consistent — the discount and the total agree with each other. The defect is that the whole rent-based figure has no network addend at all.

One further note in the interest of completeness: [`registration.hca.actors.ts#L617-L625`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L617-L625) records that the team is aware the pricing screen gates on the price rather than the budget — *"This bound is NOT the one the pricing UI enforces. That screen gates on the registration PRICE"*. That comment is about the affordability gate. It does not describe the total row, and it does not describe the receipt; and the gate is offered in this report as evidence that the fallback is not cosmetic rather than as the impact, since a wallet short of the budget is caught downstream by the preflight in the same file.

## References

**The two figures and the path between them**

* [`packages/smart-account/src/providers/rhinestone/budget.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/budget.ts) — `total = commitCost + registerCost + registrationPrice` (`:330`); the fallback-only gas-price ceiling (`:133-137`); the quoted values taken verbatim (`:322-323`)
* [`packages/transaction-manager/src/machines/registration/registration.machine.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.machine.ts) — the machine re-quotes (`:607-636`); the permit is sized from the budget (`:723`)
* [`packages/transaction-manager/src/machines/registration/registration.hca.actors.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts) — the fallback budget is refused outright (`:481-487`); the measured `~20.20` budget against an `8.00` name (`:621-625`); the team's note on the pricing gate, and the preflight that catches a short wallet (`:617-625`)
* [`apps/manager/src/features/register-v2/state/registrationUi.machine.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/state/registrationUi.machine.ts) — `confirmedData` (`:513-519`); `START_REGISTRATION` sent without a budget (`:546-560`)

**Where the rent is presented**

* [`apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx) — `startRegistration`, which forwards the rent and not the budget (`:203-221`), with `budgetQuery` in scope at (`:163`); the four fallbacks (`:434`, `:438`, `:491`, `:575`); `isQuotingFunding` (`:358`); the third statement that the displayed figure is the one the permit is sized from (`:148-154`); the primary-name switch that re-keys the budget query (`:338-341`)
* [`apps/manager/src/features/register-v2/workflow/pricing/components/PaymentTotalRow.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/PaymentTotalRow.tsx) — the `up to` hedge, and the recorded reason for dropping it (`:4-10`)
* [`apps/manager/src/features/register-v2/workflow/pricing/components/NetworkCostRow.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/NetworkCostRow.tsx) — the network cost line
* [`apps/manager/src/features/register-v2/workflow/registering/components/RegistrationDetails.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/registering/components/RegistrationDetails.tsx) — `confirmedData` (`:17-19`), the total (`:51-55`), the `Total Paid` caption (`:122-126`)

**The overloaded state, and the project's own tests**

* [`apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.ts) — both meanings of `null` in one sentence (`:51-57`)
* [`apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.test.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.test.ts) — the `PRODUCTION_FAILURE` fixture (`:6-10`); the rent, network half and total as absolute figures (`:20-22`); the recorded intent for `null` (`:166`)
* [`apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts) — the asserted equality (`:29-33`) and the acknowledgement that the machine re-quotes (`:36-41`); the query key, including the primary-name opt-in (`:136-145`); `retry: 1` and the `enabled` gate (`:149-158`)

**Route and reuse**

* [`apps/manager/.env.example`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/.env.example) — `VITE_FF_USE_EOA=false`, described as a local debugging switch
* [`apps/manager/src/features/renew/workflow/pricing/components/TokenPickerContent.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/workflow/pricing/components/TokenPickerContent.tsx) — the EOA-only renewal flow that reuses the same components, and for which the rent-as-exact-total behaviour is correct

## Link to Proof of Concept

<https://gist.github.com/Ispolin11/04e0ce4132f90011a7f81eb5c05883a5>

## Proof of Concept

## Proof of Concept

Seven test files, 28 tests, against the code at `audit-comp-ready` HEAD `1c9b47f`. Five of them put **shipped code** under test; the other two support it — a transcription of one three-line expression, and a measurement of the query library's own semantics. Both are named in *What is injected, and what is not* at the end, and in their own sections. They run offline in a few seconds: **no network, no Rhinestone orchestrator, no wallet, no fork, no RPC**. Nothing in the target repository is modified — the files are copied in, run, and deleted.

Each file prints the figures the report quotes, so every number in the Description can be traced to a line of output below.

There is no walkthrough through the running UI, and the reason is worth stating rather than leaving to be asked. A live registration on this route needs the Rhinestone orchestrator (an API key, `VITE_RHINESTONE_API_KEY` in `apps/manager/.env.example`), a funded Sepolia wallet, and the local stack that `e2e/infra/scripts/start-local-env.sh` brings up — anvil, the Alto bundler and a mock orchestrator. None of that is needed to establish the divergence this report describes: both figures are produced by the shipped code, and the tests below read them straight out of it. A live run would add a screenshot of the wallet prompt, not a fact the report rests on.

***

### Step 1 — check out the audited commit

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout 1c9b47f          # the audited commit, tip of audit-comp-ready
pnpm install
```

By commit rather than by branch on purpose. This document carries line references of its own — `budget.ts:137`, `registration.hca.actors.ts:481-487` and the rest — and those are read against whatever tree you check out, as is the source you open beside them. Pinning the checkout is what keeps them aligned: a branch that moved would shift them silently, giving wrong lines rather than an error. The Description's links are protected separately — each carries the commit SHA in the URL, so they hold whatever happens to the branch.

One note on the install, so a failure there is not mistaken for a defect in this PoC. `pnpm-workspace.yaml:10-11` pins `@ensdomains/ensjs` and `@ensdomains/ensjs-abi` to `pkg.pr.new` build artifacts. Both were reachable when this was written; artifacts on that host are ephemeral by nature, and if they are pruned the install fails for anyone who tries it. That pinning is the repository's own — nothing here depends on it beyond needing the workspace to install at all. `pnpm-lock.yaml` is committed, so every other version is fixed: that is where the `@tanstack/react-query` 5.90.12 in the table below comes from.

### Step 2 — place the four Manager-app files and run them

Copy from the attached Gist into `apps/manager/`:

```bash
cp finding8-x1-inflight-gate.test.tsx \
   finding8-x4-hca-is-default.test.ts \
   finding8-x5-receipt-total-paid.test.ts \
   finding8-x6-isloading-semantics.test.ts \
   apps/manager/

pnpm --filter manager exec vitest run finding8-x1-inflight-gate.test.tsx
pnpm --filter manager exec vitest run finding8-x4-hca-is-default.test.ts
pnpm --filter manager exec vitest run finding8-x5-receipt-total-paid.test.ts
pnpm --filter manager exec vitest run finding8-x6-isloading-semantics.test.ts
```

One command per file, because vitest interleaves the output when several run at once. Expected: **15 tests across 4 files, all passing**. Each command prints exactly the block below it.

`finding8-x1-inflight-gate.test.tsx` — 5 tests:

```
  [in flight] total=$8.00 hedged=false networkRow=true canProceed=true
  [gate] wallet holds $10.00 (rent $8.00 < x < budget $20.20)
    quoted    : total=$20.20  canProceed=false
    in flight : total=$8.00   canProceed=true
    failed    : total=$8.00   canProceed=true
  [failed] total=$8.00 hedged=false networkRow=false
  [query key] toggling the primary-name switch changes it : true
              changing the duration changes it           : true
  [quoted] total=$20.20 hedged=true networkRow=true
```

`finding8-x4-hca-is-default.test.ts` — 4 tests:

```
  [.env.example] VITE_FF_USE_EOA=false
  [flag] VITE_FF_USE_EOA="false" -> isFeatureEnabled('USE_EOA') = false
  [flag] VITE_FF_USE_EOA unset -> isFeatureEnabled('USE_EOA') = false
  [budget query enabled] rhinestone=true eoa=false disconnected=false
```

`finding8-x5-receipt-total-paid.test.ts` — 3 tests:

```
  [Total Paid] shown $8.00 | wallet debited $20.20 | ratio x2.52
  [composition] rent + premium = $11.00; no term corresponds to the $12.20 network half
  [baseRate = 0n] the row reads $0.00 for a name whose rent is $8.00
```

`finding8-x4` prints a tail of `DOMException [AbortError]` traces and a `[Reown Config] Failed to fetch remote project configuration` line **after** its summary. That is the environment tearing down, not a failure: the file imports `getHcaBudgetQueryOptions`, which pulls in `lib/wagmi`, which tries to fetch a WalletConnect configuration that is unreachable with no network. `Tests 4 passed (4)` prints before any of it.

`finding8-x6-isloading-semantics.test.ts` — 3 tests:

```
  [quote failed] status=error fetchStatus=idle isPending=false isFetching=false isLoading=false data=undefined
  [query disabled] status=pending fetchStatus=idle isLoading=false data=undefined
  [quote in flight] status=pending isLoading=true data=undefined
```

These four files add to the app's own suite rather than replacing anything, and nothing in it breaks. `pnpm --filter manager exec vitest run` reports **174 files, 1533 passed, 1 skipped** on the untouched checkout and **178 files, 1548 passed, 1 skipped** with the four in place — exactly the four files and the fifteen tests above.

### Step 3 — place the three transaction-manager files and run them

Copy from the Gist into `packages/transaction-manager/`:

```bash
cp finding8-displayed-vs-signed.test.ts \
   finding8-x2-permit-unbounded.test.ts \
   finding8-x3-unclamped-quote.test.ts \
   packages/transaction-manager/

cd packages/transaction-manager
pnpm exec vitest run finding8-displayed-vs-signed.test.ts
pnpm exec vitest run finding8-x2-permit-unbounded.test.ts
pnpm exec vitest run finding8-x3-unclamped-quote.test.ts
```

These three filter one thing out of their own console output, and it is worth saying which. The registration machine logs every state transition itself, with `console.log` (`registration.machine.ts:448` and `:581`); left in, those lines outnumber the ones that carry the finding by roughly ten to one. Each file drops exactly that one prefix and passes everything else through untouched. The block that does it sits at the top of each file and can be deleted to see the machine's output in full.

Expected: **13 tests across 3 files, all passing**. Each command prints exactly the block below it.

`finding8-displayed-vs-signed.test.ts` — 5 tests:

```
  displayed to user : $8.00
  permit signed for : $20.20
  ratio             : x2.52
  displayed $8.00 -> permit $80.00 (x10), machine error: none
  estimator input keys: name, duration, publicClient, chainId, signer, sessionEnable, primaryName
  HCA holds $5.00 -> permit $15.20 vs displayed $8.00
  HCA fully funded -> permit calls: 0
```

`finding8-x2-permit-unbounded.test.ts` — 4 tests:

```
  [D1] user saw "up to $20.20" -> permit $24.00 (+$3.80 above the hedged figure), machine error: none
  [D1b] hcaBudget carried by the UI event: undefined -> machine re-quoted its own: $20.20
  [D2] wallet debit at three HCA balances, against the rent:
    HCA holds $ 0.00 | wallet debited $20.20 | rent $8.00 | permits: 1
    HCA holds $ 5.00 | wallet debited $15.20 | rent $8.00 | permits: 1
    HCA holds $20.20 | wallet debited $ 0.00 | rent $8.00 | permits: 0
  [D3] context.tokenPrice (feeds "Total Paid") : $8.00
       permit value (what leaves the wallet) : $20.20
```

`finding8-x3-unclamped-quote.test.ts` — 4 tests:

```
  [fallback]   5 gwei -> $21.50   500 gwei -> $21.50   (source: fallback)
  [quoted]     sane  -> $20.20   absurd -> $10008.00   (source: quote)
  [asymmetry]  benign -> hostile input; the two provocations differ and are named:
      gas price, x500  : $10.70 -> $21.50  x2.01 (clamped on gas price; branch also refused at hca.actors:481-487, not exercised here)
      quote, x820 per leg : $20.20 -> $10008.00  x495 (unbounded, and this is the one that funds)
  [end to end] shown to user $8.00 -> permit signed for $10008.00 (x1251), machine error: none
```

### Step 4 — remove the copies

```bash
cd ../..
rm packages/transaction-manager/finding8-*.test.ts apps/manager/finding8-*.test.*
git status --porcelain          # clean
```

***

## What each file establishes

### `finding8-x3-unclamped-quote.test.ts` — the value that sizes the permit has no bound

Runs the shipped `estimateHcaBudget` down both of its branches, then carries the result into the shipped `registrationMachine`. The two branches are provoked by different amounts — stated here rather than glossed, because what separates them is not the size of the ratio but whether a ratio can grow at all.

1. **Fallback branch — bounded against the gas price.** The ceiling (`budget.ts:137`) is 5 gwei, so a 100× move from 5 to 500 gwei changes nothing at all: both return the identical `$21.50`. Dropping the baseline below the ceiling to make the clamp visible, a 500× move from 1 to 500 gwei takes the total from `$10.70` to `$21.50` — **×2.01** — and no larger *gas-price* move can take it further, because everything above 5 gwei clamps to that same value. To be exact about the limit of that guard: `fallbackLegFee6dp` (`budget.ts:179-190`) clamps `gasPrice` alone, and the `prices.eth / prices.usdc` factor beside it is not bounded, so this branch is not bounded against everything. It does not need to be: the closing paragraph of this section sets out why a fallback-sourced budget never funds anything — a fact about the source, not something this file runs.
2. **Quoted branch — proportional, with no ceiling.** The orchestrator's value is taken verbatim (`budget.ts:322-323`). An \~820× quote — `6_100_000` against `5_000_000_000` per leg — takes the total from `$20.20` to `$10008.00`, **×495**, and a larger quote would take it further in the same proportion. No ceiling, no plausibility check.
3. **End to end.** That unbounded budget, fed to the shipped machine, is signed for in full: `$10008.00` against a displayed `$8.00`, and `context.error` is `undefined`.

The second guard is not exercised by this file and should not be read as if it were. `x3` calls `estimateHcaBudget` from `packages/smart-account`, which computes the budget; the refusal lives in `estimateHcaBudgetActor` (`registration.hca.actors.ts:481-487`) in `packages/transaction-manager`, which throws whenever `source !== 'quote'`. That is a fact about the source, established by reading it. Taken together with what this file does measure, it puts both guards on the branch that never funds anything and none on the one that does.

### `finding8-x1-inflight-gate.test.tsx` — what the screen states, and what the gate does

Renders the shipped `TokenPickerContentBase` with one rent under all three budget states.

1. **The three states.** Quoted: `$20.20`, hedge present, network line present. In flight: `$8.00`, **no hedge**, network line present with an em dash. Failed: `$8.00`, **no hedge, no network line**.
2. **The gate moves.** With a wallet holding `$10.00` — above the rent, below the budget, the exact wallet the gate exists to stop — the quoted state correctly **blocks** and both unquoted states **let the user through**.
3. **The in-flight state needs no failure.** At a `$50.00` balance it renders `total=$8.00`, `hedged=false`, `canProceed=true`.
4. **The query is re-keyed by controls on the same screen.** `getHcaBudgetQueryOptions` returns a different `queryKey` when the primary-name switch is toggled and when the duration changes. That is all this test measures. That a newly-keyed query then has nothing to show, returning the screen to state 2, follows from the options carrying no `placeholderData` and from the third case in `x6` — not from an assertion here.

### `finding8-displayed-vs-signed.test.ts` — displayed against signed, from one run

Runs the shipped `registrationMachine` once with the HCA actors injected. Of the two figures, the permit value is read off the run; the displayed one is supplied to it as `START_REGISTRATION.price`, because that is the field the UI machine sends (`registrationUi.machine.ts:546-560`). So this file establishes what the machine does with that number and what it signs for — not that the UI sends the rent. That last part is a fact about the source, and the line to read is `TokenPickerContent.tsx:203-221`, where `startRegistration` forwards `totalPrice: pricingQuery.data.rawPrice` — with `budgetQuery` in scope a few lines above at `:163`, and not forwarded.

1. The application carries `$8.00` as the price while the funding permit is signed for `$20.20` — **×2.52**.
2. **No reconciliation exists.** A budget ten times the rent produces a permit for `$80.00` against a displayed `$8.00`, and the machine reports **no error**.
3. **The displayed price is not an input to the estimator** — its captured input keys are `name, duration, publicClient, chainId, signer, sessionEnable, primaryName`.
4. **Two refutation attempts, both recorded.** A part-funded HCA holding `$5.00` reduces the permit to `$15.20`, still above the displayed `$8.00`. The gap closes in exactly one case: an HCA already covering the whole budget, where no permit is signed at all.

### `finding8-x2-permit-unbounded.test.ts` — the three defences, tested

1. **D1 — the `up to` hedge does not bound the permit.** After a screen that said `up to $20.20`, a re-quote of `$24.00` is signed with no error raised.
2. **D1b — why that is reachable.** `context.hcaBudget` is `undefined` at the moment `START_REGISTRATION` lands; the machine re-quotes independently.
3. **D2 — the wallet debit at three HCA balances,** printed against the rent: `$20.20`, `$15.20` and `$0.00` debited, with `$8.00` alongside each. The rent column is the constant the run was given, carried for comparison — it is not read from a rendered receipt. That the receipt states that figure comes from `x5` and from the shape of `confirmedData`, not from this table.
4. **D3 — inside the application, the true figure exists only as the permit input.** `context.tokenPrice`, which feeds the receipt, stays at the rent.

### `finding8-x5-receipt-total-paid.test.ts` — the receipt row's arithmetic

Runs the expression at `RegistrationDetails.tsx:51-55` through the shipped `calculateDiscount`.

**It does not render the component, and that should be said plainly.** Mounting `RegistrationDetails` needs `RegisterV2Context` with a live machine actor, the `useBaseRate` query and the router. Instead the file imports the shipped `calculateDiscount` and transcribes the three-line expression around it — quoted verbatim in the Description — so the arithmetic under test is the component's while the render is not exercised. What carries the claim is not this transcription but the source it is taken from: `confirmedData` holds `basePriceNumber`, `premiumPriceNumber` and `totalPrice`, and no budget field at all.

1. The figure that expression produces — the one the component captions **"Total Paid"** — equals the rent: `$8.00` against `$20.20` debited, **×2.52**.
2. **The premium reaches the row; the network half has no term at all.** With a `$3` premium the row reads `$11.00` — rent plus premium, and nothing that could carry the `$12.20`.
3. Incidental, recorded because it is the same row: before the base rate loads the row reads `$0.00`.

### `finding8-x6-isloading-semantics.test.ts` — the library semantics the failed row rests on

Measured on the installed `@tanstack/react-query` 5.90.12, with the same `retry: 1` the budget query configures.

1. **After a settled failure:** `status: error`, `isLoading: false`, `data: undefined` — so `funding` and `isQuotingFunding` are false together and both disclosures vanish.
2. **A disabled query reports the same** `isLoading: false` — the legitimate pure-EOA state, which is why the fallback exists.
3. **While the quote is outstanding:** `isLoading: true`, `data: undefined` — the in-flight row, where the network line renders with an em dash and the hedge is already gone.

### `finding8-x4-hca-is-default.test.ts` — the affected route is the shipped one

1. `.env.example` ships `VITE_FF_USE_EOA=false`, and describes `true` as being for *"local EOA-only debugging on forks without bundler/relayer infrastructure"*.
2. At that value, and with the variable unset, `isFeatureEnabled('USE_EOA')` evaluates `false`.
3. The budget query whose absent result drives the whole finding is `enabled` for the `rhinestone` signer and disabled for the EOA one.

***

## Mapping to the report

| Claim in the Description                                | File                  | Line of output                                    |
| ------------------------------------------------------- | --------------------- | ------------------------------------------------- |
| Permit `$20.20` against a displayed `$8.00`, ×2.52      | `displayed-vs-signed` | `permit signed for : $20.20`                      |
| Nothing compares the two figures                        | `displayed-vs-signed` | `permit $80.00 (x10), machine error: none`        |
| The displayed price never reaches the estimator         | `displayed-vs-signed` | `estimator input keys: …`                         |
| Three budget states, section 3 table                    | `x1`                  | `[quoted] / [in flight] / [failed]`               |
| The gate falls from budget to rent                      | `x1`                  | `[gate] …`                                        |
| The budget query is re-keyed by controls on that screen | `x1`                  | `[query key] …`                                   |
| The `up to` hedge does not bound the permit             | `x2`                  | `[D1] …`                                          |
| The two figures come from separate quotes               | `x2`                  | `[D1b] hcaBudget … undefined`                     |
| The wallet debit at three HCA balances                  | `x2`                  | `[D2] …`                                          |
| The figure the receipt row computes                     | `x5`                  | `[Total Paid] …`                                  |
| The receipt has no network term at all                  | `x5`                  | `[composition] …`                                 |
| The computed branch is clamped, the quoted one is not   | `x3`                  | `[asymmetry] …`                                   |
| A fallback-sourced budget is refused before it can fund | —                     | source only: `registration.hca.actors.ts:481-487` |
| The unbounded value reaches the permit                  | `x3`                  | `[end to end] …`                                  |
| The failed row's library semantics                      | `x6`                  | `[quote failed] …`                                |
| The affected route is the default                       | `x4`                  | `[budget query enabled] …`                        |

***

## Inputs, and where they come from

| Value                  | Source                                                                                                                                                                                                                                                                                                          |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| rent `8_000_021` (6dp) | Read from `ETHRegistrar.getRegisterPrice` for a 15-character `.eth` label, duration `31536000`, paid in USDC, on an anvil fork of Sepolia at block 11517150. **No fork is needed to check it:** the same integer is the `registrationPrice` of the project's own fixture at `registrationFunding.test.ts:6-10`. |
| legs `12_200_000`      | the project's own measurement, `registration.hca.actors.ts:621-625` — *"on Sepolia an 8.00 USDC name has run to a \~20.20 USDC budget"*                                                                                                                                                                         |
| budget `20_196_054`    | the same fixture, labelled *"The live failure this gate was built for"* — `20.196054` needed against a rent of `8.000021`. Used by the Description; the tests use `rent + 12_200_000 = 20_200_021`, which gives the same 2.52.                                                                                  |
| React Query `5.90.12`  | the version pnpm resolves from `^5.90.2` in `apps/manager/package.json`                                                                                                                                                                                                                                         |

## What is injected, and what is not

Five of the seven files put shipped code under test — `registrationMachine`, `estimateHcaBudget`, `TokenPickerContentBase`, `getHcaBudgetQueryOptions` and `isFeatureEnabled` — and substitute only the environment around it, exactly as the project's own `registration.machine.test.ts` does:

* `estimateHcaBudget`, `readHcaUsdcBalance`, `signFundingPermit` and `submitFundingAndCommit` are injected actors, so the machine runs without an orchestrator and the permit value can be read off the call rather than signed by a wallet;
* in `x3` the estimator is **not** stubbed — the real `estimateHcaBudget` runs, and only its per-leg quoter and the `readContract` for the price are supplied;
* `x1` passes props directly, which is how `TokenPickerContentBase` is designed to be driven.

The remaining two do less than that, and it is better said here than discovered:

* **`x5` never mounts a component.** It imports the shipped `calculateDiscount` and transcribes the three-line expression around it from `RegistrationDetails.tsx:51-55`. The arithmetic is the component's; the render is not exercised. What the claim rests on is the shape of `confirmedData` in the source, not this transcription.
* **`x6` tests no application code at all.** It drives a `QueryObserver` from `@tanstack/react-query` directly, configured with the same `retry: 1` the budget query uses. It establishes the library semantics the failed row depends on, and nothing about this application.

Two things the PoC deliberately does **not** claim:

1. The implausible quote in `x3` is supplied by the test. The claim is that this code accepts whatever the quoter returns and bounds it nowhere — not that the orchestrator returns such values.
2. The `$24.00` re-quote in `x2` is likewise supplied. No gas move producing that divergence was observed in the wild; what is shown is that the code accepts one if it occurs.
