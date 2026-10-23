# 91924 w a high manager can display a lower registration total and execute a higher hca spend without reconfirmation

**Submitted on Sep 5th 2026 at 15:33:33 UTC by @Pig46940 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91924
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager's standalone Hybrid Custody Account (HCA) registration checkout enables **Register name** while its Rhinestone execution-cost quote is still pending. During that state, the application presents the registrar price alone as an unqualified **Total**. If the user clicks the enabled button, the click handler waits for the pending HCA quote as an internal preflight and then advances directly into registration; it does not return the user to checkout or ask the user to confirm the larger amount.

The registration machine does not treat the amount previously displayed by Manager as a confirmed spending ceiling. Instead, the HCA path independently computes the live HCA funding budget at runtime and uses that budget to size an EIP-2612 permit and a `transferFrom` from the user's wallet to the HCA.

In the reproduced production path, Manager displayed **Total $8.00 USD**, the wallet-bound permit and `transferFrom` were for **13.350591 USDC**, and fixed-fork execution successfully completed the registration while consuming **11.687095 USDC**.

This maps to the published Critical impact:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as ... showing the wrong recipient or amount on a transaction the user then approves.

Selected asset: **Manager App Files**.

## Vulnerability Details

The affected sequence is:

1. A user selects USDC while registering a name through a standalone HCA.
2. The registration price is available, but the two-leg Rhinestone funding quote is still pending.
3. Because `funding` is absent, Manager falls back to the registrar price and renders it as `Total $8.00 USD`. The **Register name** button remains enabled.
4. The user clicks **Register name** once, making the displayed amount the last application-level total the user confirms.
5. The click handler waits for the HCA quote as an internal preflight. In the reproduced path, the quote resolves to `13.350591 USDC`, consisting of an `8.000021 USDC` registration price plus quoted commit/reveal execution costs.
6. The handler advances to registration without rendering the newly resolved HCA total for another user confirmation.
7. The registration flow is started without binding the displayed registrar-only amount as a maximum confirmed HCA budget. The downstream registration machine therefore enters its HCA-budget computation path and independently determines the funding amount used for execution.
8. In the deterministic PoC, that runtime budget is `13.350591 USDC`. The machine asks the connected wallet to sign an EIP-2612 permit for that amount.
9. The submitted HCA setup batch contains both the permit and `transferFrom(wallet, HCA, 13.350591 USDC)`.
10. Fixed-fork execution successfully registers the name and consumes `11.687095 USDC`, exceeding the underlying `8.000021 USDC` registrar price that Manager rendered as `Total $8.00 USD`.

The root cause is a confirmation-boundary mismatch.

While the HCA quote is unresolved, the checkout falls back to `pricingData` for both the displayed total and the preliminary balance requirement. `canNext` does not require the HCA budget query to have completed, so the user can authorize registration while only the registrar price is visible.

The click path subsequently waits for the HCA quote, but that asynchronous result is treated as an internal execution preflight rather than a new confirmation boundary. The downstream registration machine is then free to compute and use the larger runtime HCA budget without requiring the user to confirm it in Manager.

The production code shows each part:

* The HCA quote is asynchronous and `funding` remains absent until it resolves: [TokenPickerContent.tsx lines 155-173](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx#L155-L173).
* The click handler waits for the HCA quote but treats it as an internal preflight rather than a new confirmation: [TokenPickerContent.tsx lines 223-243](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx#L223-L243).
* While `funding` is absent, both `requiredAmount` and `displayTotal` fall back to the registrar price. `canNext` does not require the HCA quote to have completed: [TokenPickerContent.tsx lines 429-462](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx#L429-L462).
* The fallback value is passed to the application's total row, while the enabled button is controlled by `canNext`: [TokenPickerContent.tsx lines 575-585](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx#L575-L585).
* The registration machine later computes and stores the runtime HCA budget when no budget override has been bound into the operation: [registration.machine.ts lines 604-636](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.machine.ts#L604-L636).
* The runtime budget determines the wallet funding shortfall and therefore the permit amount: [registration.machine.ts lines 704-733](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.machine.ts#L704-L733).
* The final HCA setup batch encodes the EIP-2612 permit and `transferFrom` using the same permit value: [registration.hca.actors.ts lines 815-844](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L815-L844).

Two independent live quote probes through the fixed-revision production estimator returned:

| Run | Registration price |  Commit quote |  Reveal quote |    Total quote |
| --- | -----------------: | ------------: | ------------: | -------------: |
| 1   |      8.000021 USDC | 2.675285 USDC | 2.675285 USDC | 13.350591 USDC |
| 2   |      8.000021 USDC | 2.597186 USDC | 2.597186 USDC | 13.194393 USDC |

The PoC uses the first captured quote. Its exact non-secret refund envelope is replayed through the deployed Sepolia HCA validator, IntentExecutor, gas-refund paymaster, registrar, and registry on an Anvil fork at block `11635612` (`0x671f17c620106232a6bba75c99f284a46532938cd747f150a7bd59f69df25880`). A deterministic 2 gwei local gas price makes the result reproducible:

* displayed Manager total: `8.00 USD`;
* underlying registrar price represented by that display: `8.000021 USDC`;
* wallet permit and `transferFrom`: `13.350591 USDC`;
* successful registration's total HCA spend: `11.687095 USDC`;
* spend above the underlying registrar price: `3.687074 USDC`;
* execution fees transferred to the relayer: `3.687074 USDC`;
* remaining quoted headroom in the user-owned HCA: `1.663496 USDC`;
* registered owner and resolver: both correct.

The quote is a funding ceiling, so the entire `13.350591 USDC` need not be consumed. This report therefore does not claim that the full permit amount is lost.

The relevant result is stronger and narrower: Manager rendered the `8.000021 USDC` registrar price as **Total $8.00 USD**, but the successful HCA execution consumed `11.687095 USDC`. Thus `3.687074 USDC` above the underlying registrar price was actually consumed by the operation rather than merely remaining as recoverable headroom in the user-owned HCA.

This report does not rely on a display-only price discrepancy. The PoC demonstrates that after Manager displays `$8.00` as the total, the wallet-bound permit and `transferFrom` use `13.350591 USDC`, and fixed-fork execution consumes `11.687095 USDC`. The demonstrated terminal state therefore includes both a wallet-bound amount divergence and an authoritative spend above the amount represented by the Manager total.

Suggested fix:

1. Include HCA quote loading/success in the checkout readiness gate and do not present the registrar price alone as the final **Total** for the standalone-HCA route.
2. If the HCA quote arrives or changes after the user's click, render the resulting total and require a new explicit confirmation before any wallet signature or funding request.
3. Bind the user-confirmed HCA budget into the registration operation and reject any later runtime budget that exceeds that confirmed amount unless the user reconfirms it.

## Impact Details

The user's last application-level amount confirmation occurs after Manager displays `Total $8.00 USD`. After that confirmation, the application obtains a larger HCA execution budget without returning to the checkout or presenting the new amount for another application-level confirmation.

In the reproduced path:

* Manager displayed `Total $8.00 USD`, representing an underlying registrar price of `8.000021 USDC`;
* the wallet-bound permit and `transferFrom` used `13.350591 USDC`;
* the successful registration consumed `11.687095 USDC`;
* the operation therefore consumed `3.687074 USDC` more than the underlying registrar price represented by the displayed total.

This is a concrete displayed-amount versus authorized-and-spent-amount divergence, not merely a difference between two estimates.

The condition is reachable through ordinary use when the HCA quote is still pending at the time the enabled **Register name** button is clicked and the subsequently resolved HCA budget exceeds the amount previously displayed by Manager, as reproduced by the PoC. No failed quote, malformed input, privileged account, or researcher-originated public-chain write is required.

The wallet request contains the higher EIP-2612 permit value and still requires the user's signature. How a particular wallet renders that structured permit value is wallet-dependent. A wallet signature therefore remains a material triage consideration, but it does not provide an application-level reconfirmation of the higher registration total: Manager presented one value as **Total**, accepted the user's registration confirmation, and only afterward constructed a larger wallet funding request and executed a higher spend.

## References

* [ENS Audit Competition scope and published impacts](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [Fixed audited revision](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae)
* [Manager HCA checkout logic](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/TokenPickerContent.tsx)
* [Registration transaction machine](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.machine.ts)
* [HCA permit and transfer construction](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts)

## Link to Proof of Concept

<https://gist.github.com/HiroyukiNaito/2c1cf42a27f2234a30bda7317090644b>

## Proof of Concept

## Proof of Concept

The complete PoC is available in the following public Gist:

**PoC Gist:** `https://gist.github.com/HiroyukiNaito/2c1cf42a27f2234a30bda7317090644b`

The Gist contains three files:

* `manager-hca-wrong-amount.poc.test.tsx` — mounts the real Manager checkout, registration XState machine, transaction manager, EIP-2612 permit builder, and final HCA request builder. Only external wallet, orchestrator, and chain boundaries use deterministic test adapters.
* `replay-live-quote-on-sepolia-fork.ts` — replays the captured live refund envelope through the deployed contracts at a fixed Sepolia block using loopback Anvil.
* `run-poc.sh` — runs both proofs and removes the temporary test files from the audited checkout afterward.

### Prerequisites

* Linux or WSL
* Node.js `22.16.0`
* The audited repository with its locked dependencies installed using pnpm
* Foundry (`anvil` and `cast`)
* A read-only Sepolia archive RPC endpoint capable of serving block `11635612`

### Run the PoC

Clone the Gist:

```bash
git clone https://gist.github.com/HiroyukiNaito/2c1cf42a27f2234a30bda7317090644b.git manager-hca-wrong-amount-poc
cd manager-hca-wrong-amount-poc
```

Prepare the audited repository:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git ../audit-comp-ens
git -C ../audit-comp-ens checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae

pnpm -C ../audit-comp-ens install --frozen-lockfile
```

Run the complete PoC:

```bash
SEPOLIA_ARCHIVE_RPC_URL="https://your-sepolia-archive-rpc.example" \
  bash ./run-poc.sh "$(realpath ../audit-comp-ens)"
```

### Expected result

```
Test Files  1 passed (1)
Tests       7 passed (7)

DISPLAYED TOTAL IS LOWER THAN WALLET FUNDING AND AUTHORITATIVE SPEND

PASS: UI showed $8.00, the wallet request funded 13.350591 USDC, and the
fixed-fork registration consumed 11.687095 USDC (3.687074 above the display).
```

The UI test demonstrates that the Manager displays `$8.00` before producing a wallet funding request for `13.350591 USDC`. The fixed-block fork replay then proves that the successful operation consumes `11.687095 USDC`.

The Sepolia RPC endpoint is used only to retrieve historical state for the local Anvil fork. All state-changing transactions execute exclusively on loopback Anvil. No public-chain transaction is submitted.
