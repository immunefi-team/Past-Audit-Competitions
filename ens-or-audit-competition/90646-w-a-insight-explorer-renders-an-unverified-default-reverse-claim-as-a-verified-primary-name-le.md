# 90646 w a insight explorer renders an unverified default reverse claim as a verified primary name letting any address assert any ens name e g vitalik eth with a green forward match true&#x20;

**Submitted on Aug 26th 2026 at 06:44:58 UTC by @promael for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90646
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

Reverse records in ENS are unauthenticated by design: `DefaultReverseRegistrar.setName(string)` lets **any** address claim **any** name, and the only thing that makes a reverse claim trustworthy is *forward verification* — resolving the claimed name back to the address. That is precisely why `UniversalResolver.reverse()` reverts `ReverseAddressMismatch` when the round-trip fails.

The Explorer asks for that verification result and then throws it away. `useReverseResolution.ts` calls ensjs `getName` with `allowMismatch: true` — the switch that converts the on-chain mismatch revert into a returned `{ name, match: false }` — and keeps only `.name`. Downstream, the reverse-resolution table hardcodes a green **"✅ True / 👤 Primary name"** badge for the inherited-default row without ever reading `forwardMatch`, and the detail sidebar computes `isPrimaryName = forwardMatch || isInheritingDefault`, so inheritance bypasses verification entirely and the red "does not resolve back to this name" warning is suppressed.

Result: for one testnet transaction and with no victim interaction, an attacker makes the Explorer's address view assert that `0xATTACKER` **is** `vitalik.eth`, verified, complete with `vitalik.eth`'s avatar — on the exact screen a user visits to answer "is this address really that name?"

## Vulnerability Details

### 1. The verification flag is fetched and discarded

`apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-241`

```ts
const defaultResult = await getName(l1Client, {
  address,
  coinType: DEFAULT_EVM_COIN_TYPE,
  allowMismatch: true,      // returns the name even when forward-verify FAILS
})
const defaultName = defaultResult?.name ?? null   // <-- `.match` DISCARDED
```

`allowMismatch: true` is the ensjs option that catches the `ReverseAddressMismatch` revert and returns `{ name: <claimed>, match: false }` instead of `null`.

### 2. The table hardcodes the verified badge

`apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:96-113`

```tsx
if (!name && defaultName && canInheritDefault(row.original.coinType)) {
  return (<> <CheckCircle2/> <span>True</span>
           <SquareUser/> <span>Primary name</span> <Badge>Default</Badge> </>)
}
```

`row.original.forwardMatch` is never consulted on this path.

### 3. The sidebar repeats it and suppresses the warning

`apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:80`

```ts
const isPrimaryName = forwardMatch || isInheritingDefault   // inheritance bypasses verification
```

Consumed at `ReverseResolutionSidebar.tsx:325` (renders `Primary name: ✅ True`), `:687-712` (renders the pairing `0xATTACKER ⟷ vitalik.eth` with `vitalik.eth`'s avatar), and `:654-662` (the mismatch warning is gated off).

### The codebase contradicts itself — this is a defect, not a policy

Three places in the same repository treat the identical record differently:

| Location                                                      | Handling of `match`                                                   | What the user sees for `0xATTACKER`                                         |
| ------------------------------------------------------------- | --------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `useReverseResolution.ts:103` (sibling path)                  | `forwardMatch: nameResult?.match ?? false` — **kept**                 | the `Default` row correctly shows **❌ False**                               |
| `useReverseResolution.ts:241` (this bug)                      | `.match` **discarded**                                                | all five L2 rows show **✅ True / Primary name**                             |
| `apps/portal/src/features/profile/hooks/usePrimaryName.ts:43` | `if (defaultResult?.match) return ok(defaultResult.name)` — **gated** | transfer-recipient preview / wallet menu correctly show **no primary name** |

So a single table shows ❌ False on the `Default` row and ✅ True on the rows that *inherit that very record*, and the same address simultaneously has "no primary name" elsewhere in the app. The verification flag is requested deliberately at line 238 and then dropped at line 241.

### Attacker-controlled input

`DefaultReverseRegistrar.setName(string)` — the app's own configured address (`apps/portal/src/features/reverse-resolution/config.ts:35`, Sepolia `0x4f382928805ba0e23b30cfb75fc9e848e82dfd47`). Reverse registrars perform **no ownership check** on the string; forward verification is the sole binding. Cost: one transaction from the attacker's own address.

### Steps to reproduce

{% stepper %}
{% step %}

## Set an arbitrary reverse name

From `0xATTACKER`, call `DefaultReverseRegistrar.setName("vitalik.eth")`.
{% endstep %}

{% step %}

## Trigger a failed forward verification

`vitalik.eth`'s `addr(0x80000000)` ≠ `0xATTACKER`, so `UniversalResolver.reverse()` reverts `ReverseAddressMismatch`.
{% endstep %}

{% step %}

## Open the reverse-resolution view

Victim opens `/addr/0xATTACKER/reverse-resolution` — no wallet connection, no interaction required.
{% endstep %}

{% step %}

## Observe the discarded verification result

ensjs returns `{ name: "vitalik.eth", match: false }`; line 241 keeps only the name.
{% endstep %}

{% step %}

## Observe false verified results

All five L2 rows (Optimism, Arbitrum, Base, Linea, Scroll) render **Name = `vitalik.eth` + `Default`**, **Forward match = ✅ True**, **👤 Primary name**.
{% endstep %}

{% step %}

## Open a row

Opening any row shows `Primary name: ✅ True` and the pairing `0xATTA…CKER ⟷ vitalik.eth` with `vitalik.eth`'s avatar, and **no** mismatch warning.
{% endstep %}
{% endstepper %}

### Recommended fix

Use the flag that is already being fetched:

```ts
// useReverseResolution.ts:241
const defaultName = defaultResult?.match ? defaultResult.name : null
```

And remove the unconditional badge at `columns.tsx:96-113` and the `|| isInheritingDefault` term at `computeDisplayNameState.ts:80`. Inheriting the *name* for display is fine; asserting **Forward match: True / Primary name** for an unverified claim is not.

## Impact Details

This is the Explorer's name-verification surface returning a false positive for a forged claim. Concretely:

* **Impersonation with an affirmative verification badge.** Any address can be made to display as any ENS name — including high-value names — with a green check, the "Primary name" label, and the impersonated name's avatar. A user who visits the Explorer specifically to check whether an address belongs to a name is told, affirmatively, that it does.
* **The contradicting signal is hidden.** The UI does not merely omit the warning; it *suppresses* the "The set address does not resolve back to this name" notice that exists for exactly this case, and replaces ❌ with ✅.
* **Zero-interaction, zero-cost, persistent.** No victim interaction, no wallet connection, one transaction, and the state persists until the attacker clears it.

Mapped to the program's impact list: *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient — rendering an ENS name to an attacker-controlled address."*

**Honest severity note.** I am filing this as **High** rather than Critical. The defect is confined to the Explorer's reverse-resolution views: the app's transaction-adjacent paths (`usePrimaryName`, used by the transfer-recipient preview and wallet menu) apply the check correctly, so this does not by itself pre-fill a transaction field. The harm runs through a user trusting the Explorer's verification and then acting on it. I would rather state that limit than overclaim.

## References

* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-241` — `allowMismatch: true`, `.match` discarded
* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:96-105` — sibling path that correctly keeps `match`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:96-113` — hardcoded verified badge
* `apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:80` — `forwardMatch || isInheritingDefault`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionSidebar.tsx:325, 654-662, 687-712` — verified label, suppressed warning, rendered pairing
* `apps/portal/src/features/profile/hooks/usePrimaryName.ts:43` — the correct gate, elsewhere in the same app
* `apps/portal/src/features/reverse-resolution/config.ts:35` — configured `DefaultReverseRegistrar`
* ENSIP-19 (reverse resolution): forward verification is required for `default.reverse`, which is why `UniversalResolver.reverse()` reverts on mismatch

## Dedup

Not in the disclosed known-issues set (R2-01, R2-02, R2-04, R2-05, EXP-INPUT-003/005/008/009, EXP-GAP-006, SEC-MGR-*, R3-*). In particular this is **not** EXP-INPUT-005 (homoglyph/bidi display confusion): the displayed and resolved strings are byte-identical here; the defect is a missing forward-verification check, not a confusable rendering.

**Corroboration from ENS's own production app.** The publicly deployed ENS manager, `ensdomains/ens-app-v3`, treats this exact flag as the security boundary — `src/hooks/ensjs/public/usePrimaryName.ts:58`:

```ts
if (!res || !res.name || (!res.match && !params.allowMismatch)) return null
```

`allowMismatch` is an explicit opt-in there, and when it is not set the unverified name is not surfaced at all (the same file even comments, at :127, that a non-normalized name "should be treated as a mismatch"). So across the ENS ecosystem there are three independent implementations of this check: `ens-app-v3` gates it, this repo's own `usePrimaryName.ts:43` gates it, and only `useReverseResolution.ts:241` opts into mismatch and then renders the result as verified.

**Triage risk I want to disclose up front:** `computeDisplayNameState.test.ts:92-105` asserts this exact output as expected, so the behaviour may be argued as intended "default inheritance". Three points against that reading:

1. ENSIP-19 requires forward verification for `default.reverse` — that is why the contract reverts.
2. The code deliberately requests `match` at line 238 and drops it at line 241.
3. The same repository applies the check correctly in `usePrimaryName.ts:43`, and the same table renders ❌ False for the underlying `Default` row.

Inheriting the name is defensible; labelling it **verified** is not.

## Link to Proof of Concept

<https://gist.github.com/Kyenghwan-Hwang/77065a097fa29926c15c6c82f88cd681>

## Proof of Concept

Two test files, 5/5 passing, no modification to any application source.

```bash
cd apps/portal && npx vitest run src/features/reverse-resolution/POC-*
```

**`POC-ensjs-getName-allowMismatch.test.ts`** — no ensjs mocking. Drives the real `getName` against a viem transport that returns a genuine ABI-encoded `ReverseAddressMismatch` revert, and asserts:

* with `allowMismatch: true` → `{ name: 'vitalik.eth', match: false }` (the name survives the failed verification)
* without it → `null`

This establishes that the flag discarded at line 241 is the only thing distinguishing a verified from a forged claim.

**`POC-unverified-default-primary.test.tsx`** — three steps against real application code:

* **STEP 1** — the real `getReverseResolution` propagates `defaultName = 'vitalik.eth'` to all five L2 rows while `forwardMatch` is `false`.
* **STEP 2** — renders the real `columns` cells: the DOM contains `vitalik.eth`, `True` and `Primary name`, and contains **no** `False`.
* **STEP 3** — the real `computeDisplayNameState` returns `isPrimaryName: true, canSetAsPrimary: false` for the unverified claim.
