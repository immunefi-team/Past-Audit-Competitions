# 93137 w a high charged vs displayed divergence on the hca registration checkout on a failed budget quote the screen shows the rent 8 but the funding permit charges the full budget 20 20&#x20;

**Submitted on Sep 14th 2026 at 10:58:19 UTC by @FestiveSRT for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93137
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**

## Description

## Brief/Intro

On the standalone-HCA registration checkout (apps/manager), when the funding-budget quote fails (or is still in flight) at render, the screen falls back to displaying, and gating the proceed button on, the registrar rent alone (~~$8.00 on the repo's own Sepolia example). Meanwhile the registration machine independently re-quotes and sizes the single EIP-2612 funding permit on the FULL budget (~~$20.20 = registration price + both Rhinestone leg costs). In production, a user who funds the full budget is charged the whole \~$20.20 on-chain having been shown only \~$8.00 — a δ ≈ $12.20 charged-vs-displayed divergence. The extra amount is legitimate network cost (relayer/gas for the commit and register legs), not value diverted to any third party; the defect is that the checkout screen, the figure the user judges sufficiency from, does not disclose it. This is a demonstrated instance of the programme's QA-03 inversion: "a mismatch between the displayed total and the amount actually charged on-chain would be a new finding."

## Vulnerability Details

On the standalone-HCA route the wallet signs ONE EIP-2612 permit for the whole funding budget (registration price + commit-leg cost + register-leg cost), and the commit batch transfers it into the HCA. The checkout screen is supposed to display that budget as the total. But the budget quote is best-effort: when it fails, the screen deliberately falls back to the registrar rent alone and lets the click proceed, while the machine's own re-quote still sizes the permit on the full budget. The result is displayed(rent) ≠ charged(full budget). The two halves live in different packages: the display half is in apps/manager, the on-chain charge half in packages/transaction-manager.

Every step verified in the pinned tree:

1. The UI budget quote is best-effort and can end in error. getHcaBudgetQueryOptions runs the estimator with retry: 1 and, per its own comment, callers must "treat 'no budget' as 'show the price alone and let the machine surface any failure', never as a hard block" (apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts:154-158). On failure budgetQuery.data is undefined.
2. Failed/absent budget → funding summary is null. computeRegistrationFunding({ budget: budgetQuery.data, … }) (apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx:168-173) returns null when the budget is absent: `if (!budget) return null` (apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.ts:72; docstring :51-57: "Returns `null` when no budget has been quoted yet (or the quote failed), in which case callers should show the price alone rather than block"). The container comment states the fallback outright (TokenPickerContent.tsx:165-167): "permanently absent if it fails — in which case the screen falls back to showing the rent alone rather than blocking on a flaky quote."
3. The displayed total becomes the rent. In TokenPickerContentBase, `const displayTotal = funding?.total ?? pricingData` (TokenPickerContent.tsx:438); with funding absent (null from computeRegistrationFunding, mapped to undefined by the container at :353), displayTotal = pricingData = the registrar price row (the rent). It is painted by `<PaymentTotalRow isEstimate={!!funding} total={displayTotal} />` (:575); because funding is absent, isEstimate is false, so the rent is shown as a firm total with no "up to" hedge. The NetworkCostRow is not rendered at all (:491, `{(funding || isQuotingFunding) && …}`), so the dropped \~$12 network cost is invisible on the screen.
4. The proceed button is gated on the rent, not the charge. `const requiredAmount = funding?.walletDebit ?? pricingData` (:434) → the rent; hasSufficientBalanceForSelectedCoin (:449-455) compares the wallet against requiredAmount; canNext (:457-462) enables the Register button on the rent alone.
5. The click path treats a quote failure as "not a funding failure" and falls through. In availabilityMutation.mutationFn the budget is re-fetched with `.catch(() => null)` and the comment "A quote failure is not a funding failure. Fall through…" (:229-233); with budget === null, walletDebitRaw = null (:238-242) and the InsufficientFundingError guard is skipped (:244-253). startRegistration dispatches.
6. The machine re-quotes the budget INDEPENDENTLY and fails loudly rather than under-fund. State computingHcaBudget invokes estimateHcaBudget (packages/transaction-manager/src/machines/registration/registration.machine.ts:607-637) — the same estimator the UI uses (hcaBudget.query.ts:28-33: "This runs the SAME estimator the registration machine runs … so the number shown at checkout is the number the machine will size the permit from") but a fresh call, not the UI's cached error; on the estimator's own failure it goes to error, not to a guessed amount (:638-648). On success hcaBudget = breakdown.total (the full budget).
7. The permit is sized on the full budget. signingFundingPermit sets `value: bigintMax(context.hcaBudget! - (context.hcaUsdcBalance ?? 0n), 0n)` (registration.machine.ts:721-725) = the full budget for an empty HCA.
8. The charge is that permit value. signFundingPermitActor signs an EIP-2612 permit whose message.value = input.value (registration.hca.actors.ts:702-716), and the commit batch's transferFrom(wallet, HCA, value) pulls exactly that.

The code's authors documented the two-figure gap themselves (registration.hca.actors.ts:620-626): "That screen gates on the registration PRICE; `value` is the funding shortfall for the whole HCA BUDGET … on Sepolia an 8.00 USDC name has run to a \~20.20 USDC budget." They closed the revert-legibility half of it (an unclassifiable revert became a descriptive error); they did not reconcile the displayed total with the charge.

Net: the screen showed the rent (~~$8.00); the on-chain charge is the full budget (~~$20.20). The divergence is exclusive to the quote-unavailable path: when the quote succeeds, displayTotal = funding.total = the full budget = the charge, so displayed and charged agree. The divergence does not even require the click-time re-fetch (step 5) to also fail: the Total was already painted at render time from budgetQuery.data (step 3), so the condition is simply that the budget is unavailable at render (the quote FAILED or is still loading), the machine re-quote then succeeded, and the wallet funds the full budget. The in-flight isLoading window is therefore a second path to the same divergence.

A smaller sub-instance of the same class arises without any quote failure: the primary-name toggle's default is resolved twice from two eligibility queries. At render they are in flight (toggle OFF, budget quoted WITHOUT the primary-name call); at click resolveSetAsPrimary() (TokenPickerContent.tsx:179-197) re-resolves TRUE and dispatches postRegistrationSetup.primaryName.enabled (:217-219), widening the reveal-leg gas limit the permit is priced on, without re-quoting the budget for the resolved toggle. Measured δ ≈ 40,000 gas ≈ $0.08 at the repo's live-verified rate (priced on the register-leg marginal instead, the same 40,000 gas is ≈ $0.17: packages/smart-account/src/providers/rhinestone/budget.ts:113-114 — a 450k→510k limit moves the orchestrator quote 3,277,666→\~3,539,296 6dp USDC, ≈4.36e-6/gas × 40,000). Either figure is immaterial to severity; it is included only as corroboration that the divergence is structural.

## Impact Details

Selected impact: "Security Best practices" (Insight tier). This finding — a charged-vs-displayed non-disclosure with no attacker, no fund diversion, and no third-party recipient — matches that category, and is in scope under the programme's QA-03 inversion clause, which names "a mismatch between the displayed total and the amount actually charged on-chain" as a new finding with no magnitude floor (the general clause also covers "new consequences of a listed root cause that materially change its severity"; display-level → charged). It may be graded Low under that clause; filed conservatively as Insight and left for the triager to elevate.

Loss magnitude: whenever the budget quote fails at render (or is still loading) but succeeds in the machine's independent re-quote, a wallet that funds the full budget is charged \~$20.20 (the repo's own Sepolia example) having been shown only \~$8.00, a δ ≈ $12.20 understatement of the amount actually pulled on-chain. The extra is legitimate network cost (relayer/gas for the commit and register legs), not value diverted anywhere; the defect is non-disclosure on the screen the user judges sufficiency from.

Why at most Low, not Critical (the honest counters, verified to hold):

* Pre-permit wallet-balance check. Before prompting for a signature, signFundingPermitActor reads the wallet's real USDC balance and throws if walletBalance < input.value (the full budget) (registration.hca.actors.ts:609-637). The machine never silently pulls more than the wallet holds; a wallet holding between rent and budget clears the rent-gated button and is then rejected here: a confusing failed registration, but no charge.
* The true charge is shown at signing. The permit signTypedData message carries value = input.value = the full budget (registration.hca.actors.ts:689-716). The transaction the user actually approves in their wallet shows the true \~$20.20. The mismatch is between the app's checkout screen and the wallet's signature prompt, not a hidden/silent overcharge. We explicitly do NOT claim the Critical "showing the wrong recipient or amount on a transaction the user then approves" reading (the bullet whose heading requires that "a user sends funds or submits a transaction to an unintended recipient"): the recipient is the user's own HCA (not an unintended recipient) and the approved permit shows the true value, so that bullet's conditions are not met.

Honest downside (why a triager may land this at Insight or no-reward): the funds go to the user's OWN HCA, and unspent surplus is retained and credited on the next run (registration.machine.ts:668-671, 713-718), so the "extra" is not even fully consumed; there is no attacker; and the window needs the render-time quote to fail (or still be loading) and the machine re-quote to succeed — a flaky-orchestrator precondition the code's comments repeatedly acknowledge ("permanently absent if it fails", "a flaky orchestrator"), plausible but unmeasured. With the wallet re-displaying the true value at signing, a triager could reasonably grade this Insight or no-reward; we do not argue against that call. High/Medium on a fund-loss or attacker theory is explicitly not claimed. No data exfiltration, no attacker, no diverted value.

Distinctness from the excluded QA-03: QA-03 (tagged Functional, Explorer) is a display-level-only price mismatch between figures shown in the registration/extension modals; nothing charged differs. This finding is the programme's explicit inversion: the displayed total (~~$8) differs from the amount actually charged on-chain (~~$20.20), on a different app/component (the apps/manager HCA registration checkout), demonstrated with an executing driver. The distinctness rests on this being a different mechanism, not on QA-03's tag or its inversion carve-out transferring automatically. The programme states transaction-manager and smart-account "should not be assumed exhausted."

## References

* Displayed half (apps/manager): TokenPickerContent.tsx:165-197 (fallback comment + computeRegistrationFunding call + resolveSetAsPrimary), :217-219 (postRegistrationSetup dispatch), :229-253 (click-path re-fetch `.catch(() => null)` + skipped InsufficientFundingError guard), :345-354 (container funding mapping), :434 (requiredAmount rent fallback), :438 (displayTotal rent fallback), :449-462 (balance check + canNext), :491 (NetworkCostRow gated off), :575 (PaymentTotalRow); all under apps/manager/src/features/register-v2/workflow/pricing/components/.
* Funding derivation: apps/manager/src/features/register-v2/workflow/pricing/lib/registrationFunding.ts:51-72 (null-on-absent-budget). Budget query: apps/manager/src/features/register-v2/data/queries/hcaBudget.query.ts:28-33 (same estimator), :154-158 (best-effort, retry:1).
* Charged half (packages/transaction-manager): registration.machine.ts:607-648 (independent re-quote, fail-loud on error), :668-671 & :713-718 (surplus retained / top-up-only), :721-725 (permit value = full budget). registration.hca.actors.ts:609-637 (pre-permit balance gate), :620-626 (authors' documented two-figure gap), :689-716 (permit message.value = input.value shown at signing), :702-716 (EIP-2612 permit signed).
* Primary-name sub-instance rate basis: packages/smart-account/src/providers/rhinestone/budget.ts:113-114 (register-leg gas-limit widening; live-orchestrator-verified quote figures).
* Programme rules relied on (Resources/Information tab): the QA-03 inversion clause — "a mismatch between the displayed total and the amount actually charged on-chain would be a new finding"; the general clause "new consequences of a listed root cause that materially change its severity"; the "should not be assumed exhausted" note for transaction-manager and smart-account.
* Provenance: the audited tree is byte-identical to the public competition repo immunefi-team/audit-comp-ens at commit cda79ac (root tree 8e0d79dac1ab4b4fdb80d6afed810087ae9f00ba); that commit's message forks ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86, the revision named on the programme's Resources tab (63772fd8). In-scope source CODE is unchanged through the public repo's head 1c9b47f (only hono dependency-version bumps in manifests). Every file:line citation above is to that pinned tree.

## Recommended remediation

On the HCA route, do not show the rent as the total until a budget has been quoted for THIS route; gate on budget-quoted, not on budgetQuery.isError alone (an isError-only gate leaves the initial isLoading window open: the button is enabled on the rent while the quote is in flight, and the click-path re-fetch then succeeds and the machine charges the full budget):

* Thread budgetRequired (true only on the HCA route) and budgetSettled = !isLoading && !isError && funding != null into TokenPickerContentBase. When budgetRequired && !budgetSettled, do not let displayTotal/requiredAmount fall through to pricingData, and force canNext false (:457, e.g. `&& (!budgetRequired || budgetSettled)`). Renew and pure-EOA routes leave budgetRequired=false, so their behaviour is unchanged (a route-agnostic funding-presence gate would regress them). On persistent quote failure the checkout fails closed (retry-recoverable); do not show an "upper-bound budget" (none was quoted).
* In mutationFn keep the hard stop, but trigger it whenever the HCA-route budget is unavailable (fetch failed OR still returns null), not only on the `.catch(() => null)` path.
* (Primary-name sub-instance) Block the click until the eligibility lookups (profileReverseNameQuery + ownedNamesCountQuery) settle, so the budget query re-keys on the resolved primary-name and the displayed total matches the charge. (The "re-quote before dispatch" alternative suffices only if the re-quoted figure is re-rendered and re-confirmed before proceed is re-enabled.)

## Link to Proof of Concept

<https://gist.github.com/AFestive/9f4c2e98c71fec905098232b44bedc1d>

## Proof of Concept

## Proof of Concept

Offline and permissionless-only: no fork, no network. The harness drives the real audited code: it imports and runs the real computeRegistrationFunding (the function whose null-on-failed-budget return IS the fallback), renders the real TokenPickerContentBase component and reads the Total it paints and the enabled/disabled state of its Register button out of the DOM, maps the funding prop verbatim from the real container (TokenPickerContent.tsx:345-354), and computes the charge from the real permit formula (registration.machine.ts:721-725) and pre-permit gate (registration.hca.actors.ts:609-637).

Honest scope: the displayed half ($8 rent) is executed against the real component and read from the DOM. The charged half (\~$20.20) is the code's own documented figure (registration.hca.actors.ts:620-626 comment "an 8.00 USDC name has run to a \~20.20 USDC budget"; permit.value = full budget), cited rather than independently machine-executed. For an audit competition the step-by-step below is sufficient on its own.

Steps:

1. Run the harness: `bash run.sh` (runs `vitest run` from apps/manager under a scratch config and writes verdict.json).
2. Control — quote SUCCEEDS: real computeRegistrationFunding → non-null; screen Total = $20.20; Network-cost row shown; charge = $20.20 → diverges: false (displayed = charged).
3. Vuln — quote FAILS, wallet funds the budget: real computeRegistrationFunding → null; screen Total = $8.00 (rendered "Total$8.00USD", no "up to" hedge, no Network-cost row); Register button ENABLED on the rent; charge = $20.20; machine does NOT throw → diverges: true, δ = $12.20 (the charged divergence).
4. Boundary — quote FAILS, wallet holds between rent and budget ($10): screen Total = $8.00, button enabled on the rent, machine pre-permit gate would throw → failed registration, no charge.
5. Discriminating guard (verify behaviour, not shape): the test asserts the divergence is present when the quote fails and absent when it succeeds. The code-fix negative control — the same real component with the `?? pricingData` rent-fallbacks removed at :434/:438 (the fix) — fails the guard (displayed $0.00, Register button disabled), proving the probe distinguishes the vulnerable code from a fixed one (a true fail-before/pass-after against a code change, not a test-input mutation).
6. The primary-name sub-instance (δ ≈ $0.08) is recorded in the evidence file below.

Evidence manifest (offline; belongs in a secret Gist, not as attachments):

* run.sh — runner (vitest run from apps/manager under a scratch config)
* the vitest scratch config and the main divergence test
* verdict.json — the control / vuln / boundary rows above
* the negative-control component (rent-fallbacks removed), its test, its vitest config, and its verdict.json
* evidence-divergence.json — the primary-name sub-instance measurement (δ ≈ $0.08)
