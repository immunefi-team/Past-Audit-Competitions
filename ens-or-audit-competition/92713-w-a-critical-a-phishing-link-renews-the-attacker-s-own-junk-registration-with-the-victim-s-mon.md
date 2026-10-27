# 92713 w a critical a phishing link renews the attacker s own junk registration with the victim s money while manager displays the victim s real name

**Submitted on Sep 11th 2026 at 19:20:37 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92713
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

Manager's renewal routes validate and display a name using full ENSIP-15 normalization (`normalize()`), but build the on-chain `renew()` calldata from a second, independent parser (`parseName`) that only lowercases the input and rejects a fixed 22-character ASCII denylist. It performs no Unicode/IDNA handling at all.

`normalize()` treats certain codepoints — Default-Ignorable format characters such as U+FE0F (VARIATION SELECTOR-16) — as `ignored` and maps them to the empty string. `parseName` retains them byte-for-byte. A label carrying such a codepoint therefore normalizes to the victim's real, legitimate name for every validation/display gate, while the raw route parameter — carrying the extra invisible byte — is what the app actually prices, ABI-encodes, and sends to the wallet for signature.

An attacker registers their own real V2 `.eth` registration whose label is the victim's target name plus one invisible U+FE0F codepoint — entirely through Manager's own `/register/$name` flow, no different from registering any other name. They then send the victim a link of the shape `https://app.ens.dev/renew/<target>️.eth`. The victim's browser renders the exact target name they expect, with the correct current expiry, and the wallet is asked to sign an ordinary-looking renewal. The signed `renew()` transaction's `label` argument is the attacker's byte sequence, not the victim's target: the payment renews the attacker's junk registration. The victim's actual name is completely untouched.

My E2E PoC drives the real, unmodified Manager UI for both the attacker's registration and the victim's renewal, against real deployed V2 ENS contracts on a local Sepolia fork, using real headless Chromium and a real injected EIP-1193 wallet for each actor. The mined `renew()` transaction is decoded straight from chain state and its `label` argument is asserted to be byte-identical to the attacker's raw route-parameter bytes, and different from the bystander's clean label the victim saw on screen. The bystander's real target-name expiry is asserted unchanged; the attacker's junk registration's expiry is asserted extended; the victim's real USDC balance is asserted to have decreased by the exact on-chain quoted price.

## Finding Description

### Intended behavior

The renewal routes (`/renew/$name` and its legacy counterpart `/renew-v1/$name`) must renew the same name that they validated ownership/expiry for and that they display to the user. Whatever raw bytes the route receives, the SAME canonicalized identity must flow through every gate — ownership check, expiry check, price quote, on-screen name — and into the final signed `renew()` transaction's `label` argument.

### Actual behavior

#### 1. Two independent name-parsing regimes exist in Manager, and the renewal routes mix them on the same raw input

`apps/manager/src/features/profile/service/profileName.ts:14-40` (`normalizeEthName`) wraps `viem/ens` `normalize()` (full ENSIP-15/UTS-46 normalization) in a `try/catch` that returns `null` on rejection:

```ts
export const normalizeEthName = (name: string): EthName | null => {
  let normalized: string
  try {
    normalized = normalize(name)
  } catch {
    return null
  }
  ...
}
```

`normalizeEth2LdName` (`profileName.ts:62-73`) is the 2LD-specific wrapper used by the expiry gate. This is the ONLY normalizing code path in Manager.

`apps/manager/src/features/register-v2/utils/name-parser.ts:11,33-82` (`parseName`) is the second, naive parser used everywhere a route builds display/pricing/calldata:

```ts
const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/
...
export const parseName = (name: string): Result<ParsedName, ...> => {
  const normalized = name.trim().toLowerCase()
  ...
  if ([...labels, label, tld].some((part) => INVALID_LABEL_CHARS.test(part))) {
    return ParseNameError.err('INVALID_CHARACTER')
  }
  return ok({ subLabels: labels, label, tld })
}
```

It performs no IDNA/Unicode handling whatsoever — every non-ASCII/invisible/combining codepoint passes straight through untouched, as long as it is not one of the 22 listed ASCII punctuation characters.

`apps/manager/src/routes/renew/$name.tsx:19-63` (the mainline v2 route) mixes both regimes on the SAME raw route param `name`:

```ts
loader: async ({ params: { name }, context: { queryClient } }) => {
  const parsedName = parseRenewableName(name)                 // Regime B (naive)
  if (parsedName.isErr()) throw parsedName.error

  const expiryData = await queryClient.ensureQueryData(
    profileExpiryQuery(name),                                 // Regime A (normalize())
  )
  if (expiryData?.protocol === 'v1') throw redirect({ ... })

  const expiryDate = profileExpiryDateFromSeconds(expiryData?.expiry)
  if (expiryData?.protocol !== 'v2') throw new Error(...)
  if (isPastGracePeriod(expiryDate, expiryData.protocol)) throw redirect({ ... })
  if (!canRenewV2Name(name, expiryDate)) throw new Error(...) // Regime B again, format-only
  if (!expiryData.expiry) throw new Error(...)

  return { label: parsedName.value.label, currentExpiry: expiryData.expiry }
},
```

`parseRenewableName` (`apps/manager/src/features/renew/utils/renewableName.ts:25-36`) is a thin wrapper around `parseName` that only additionally checks TLD/subname shape. `canRenewV2Name` (`renewableName.ts:40-47`) re-parses the SAME raw `name` with `parseName` again and calls `isRenewableV2EthName` (`apps/manager/src/features/grace/utils/gracePeriod.ts:41-55`), which only regex-tests the format `^[^.]+\.eth$` and compares the ALREADY-normalized expiry from the line above against `now` — **it performs no on-chain check of the raw label at all.**

The legacy `apps/manager/src/routes/renew-v1/$name.tsx:14-57` route has the analogous split: gates 1-2 (`profileOwnerQuery`, `profileExpiryQuery`) run Regime A; gate 3 (`getV1RenewableQueryOptions`, `apps/manager/src/features/renew/data/queries/v1Renewable.query.ts:18-34`) re-parses the same raw `name` with naive `parseName` and sends that raw label to the on-chain `ETHRenewerV1.isRenewable(label)` call.

#### 2. ENSIP-15 normalize() silently strips certain codepoints that parseName retains byte-for-byte

I executed the actual installed `viem/ens` `normalize()` and the actual shipped `parseName` against a label carrying U+FE0F (VARIATION SELECTOR-16) directly in this PoC (Phase 2, logged verbatim below): `normalize()` returns the clean label with the codepoint stripped to nothing; `parseName` returns the exact same bytes with the codepoint retained. The two therefore compute DIFFERENT `labelhash`es for the identical nominal route parameter.

U+FE0F was chosen over U+00AD (soft hyphen) specifically because U+00AD is a CSS soft-hyphen and can render as a visible `-` glyph at a line-wrap point under some viewport/length combinations, which would make the divergence occasionally perceptible; U+FE0F after a plain ASCII base character has zero rendering footprint in any viewport, confirmed against the actual installed `@adraffy/ens-normalize` build (the library `normalize()` wraps), whose own documented tokenizer output classifies a non-emoji-context U+FE0F as `ignored`.

#### 3. The naive label reaches the price quote, the on-screen name, and the final signed transaction unchanged

`apps/manager/src/features/renew/data/queries/renewPricing.query.ts:26-50` (`getRenewPrice`) takes the loader's naive `label` directly and calls `ensGetRenewPrice(publicClient, { renewerAddress: getRenewerAddress(protocol), label, duration, paymentToken })` — a live, client-side on-chain read keyed on the raw byte sequence.

`apps/manager/src/features/register-v2/workflow/pricing/components/PricingDomainHeader.tsx:7-33` renders that same label unfiltered:

```tsx
export const PricingDomainHeader = ({ label }: { label: string }) => {
  const name = `${label}.eth`
  ...
  return (... <h1 title={name}>{name}</h1> ...)
}
```

No sanitization, escape-visualization, or invisible-character flagging exists anywhere in this component or its callers.

`apps/manager/src/features/renew/workflow/RenewalPage.tsx:13-29`, `apps/manager/src/features/renew/state/renewalUi.context.tsx:16-56`, and `apps/manager/src/features/renew/workflow/pricing/components/ConfirmPurchase.tsx:13-79` all carry the SAME naive `label` verbatim — `ConfirmPurchase.tsx:63-72` sends it straight into the `renewal.start` event with zero re-validation. `apps/manager/src/features/renew/state/renewalUi.machine.ts:184-199` sets `context.renewerAddress = getRenewerAddress(input.protocol)` once at machine construction and never touches `label` again except to forward it:

```ts
// renewalUi.machine.ts:144-160 (startRenewal), :341-356 (submittingPlainRenewal)
submissionData: ({ event }) => event.type === 'renewal.start' ? { label: event.label, ... } : undefined,
...
input: ({ context }) => {
  const submission = requireSubmissionData(context.submissionData)
  return { label: submission.label, duration: submission.duration, ..., renewerAddress: context.renewerAddress }
},
```

The terminal sink, `packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1031` (`encodeRenewData`), ABI-encodes that exact string as the raw `renew()` argument:

```ts
function encodeRenewData(label: string, duration: bigint, paymentToken: Address): Hash {
  const cleanLabel = label.replace('.eth', '')
  return encodeFunctionData({
    abi: ethRegistrarRenewSnippet,
    functionName: 'renew',
    args: [cleanLabel, duration, paymentToken, REFERER_ADDRESS],
  })
}
```

`registration.actors.ts:1006-1015`'s own comment confirms this is a deliberate design point, not an oversight in only this one place: `ETHRegistrar.renew` "pulls the rent from `_msgSender()`" and "registrar uses HCA-aware sender resolution... always pulls payment from the EOA" — the label is a raw ABI `string`, labelhash = `keccak256(raw label bytes)`, never pre-hashed or re-checked against the normalized name anywhere downstream of the loader.

#### 4. The attacker's precondition — their own oddly-labeled registration — is achievable entirely through Manager's own register flow

`apps/manager/src/routes/register/$name.tsx:20-58` calls the same naive `parseName` on the raw route param. `apps/manager/src/features/register-v2/data/queries/availability.query.ts:6-20` calls `normalizeDomainNameFromUrl` (`apps/manager/src/utils/domain.ts:27-33`), which only lowercases the `.eth` suffix and UTF-8-truncates to 255 bytes — no Unicode normalization. `checkRealNameAvailability` (`apps/manager/src/features/shared/service/nameChainContractService.ts:28-60`) calls `validateENSName` (`apps/manager/src/features/shared/registration/nameUtils.ts:90-143`), whose denylist (`nameUtils.ts:132`) is byte-for-byte identical to `name-parser.ts`'s — ASCII-only, no Default-Ignorable coverage — then reads real on-chain availability with `getAvailable(publicClient, { name })` using the raw, unnormalized name. `generateCommitment`/`encodeRegisterCall` (`packages/transaction-manager/src/machines/registration/registration.actors.ts:133-174,223-256`) use the raw name verbatim for `makeCommitment`/`commit`/`register`. No validator anywhere in this chain rejects a Default-Ignorable codepoint. My PoC (Phase 3) proves this directly: the attacker completes the entire registration for `<target-label>️.eth` through Manager's real, unmodified `/register/$name` UI — real wallet connect, real "Pay with stablecoins", real USDC approve, real `commit`→wait→`register` sequence, real on-chain state after — with no code path substituted or bypassed.

## Root Cause

`apps/manager/src/routes/renew/$name.tsx` (and `apps/manager/src/routes/renew-v1/$name.tsx`) use two independently-implemented name parsers against the identical raw route parameter: full ENSIP-15 `normalize()` for the ownership/expiry gates and the on-screen name, and a naive `.trim().toLowerCase()` + fixed ASCII denylist (`parseName`) for the price quote, the third gate, and the final `renew()` calldata. `normalize()` maps Default-Ignorable codepoints (e.g. U+FE0F) to the empty string; `parseName` does not touch them at all. Whenever the raw parameter carries such a codepoint, the two parsers diverge to different `labelhash`es for what renders as the identical visible string, and nothing downstream of the loader ever re-checks the naive label against the normalized one.

## Impact Explanation

The renewal transaction's `label` argument is a `tx-detail` in the literal ABI sense — a named parameter of the calldata the connected wallet is asked to sign — and it is provably different from what Manager's own trusted validation/display pipeline computed and showed the user. This matches the selected impact directly: the application displays a correct name and pre-fills a transaction that in fact targets a different, attacker-chosen recipient identity, and the connected wallet signs and submits it.

The harm completes entirely inside the single `renew()` transaction Manager itself constructs and gets the wallet to sign — no separate, later call by another actor is required (no out-of-scope follow-on). It is a one-shot, irreversible loss: once mined, the ERC-20 payment is spent and the attacker's registration's expiry is extended; the victim has no revoke path and the victim's real target name is simply untouched, not recoverable-by-the-victim in any sense that offsets the loss. There is no cache, no session, and no prior victim-performed mutation the attacker must wait on — the mismatch is a static property of the parsing logic itself and fires identically on the very first load of a freshly crafted link.

The attacker's junk registration is a real, held ENS asset under the attacker's ownership; extending its expiry with the victim's payment is a direct, quantifiable transfer of value to that attacker-controlled asset — economically identical to the attacker receiving the payment and spending it to renew their own name themselves.

Realistic UI-preset durations are 1/3/6 years and ordinary `.eth` renewal pricing is low-double-digit USD/year, so a typical exploitation nets the attacker on the order of tens of dollars of victim-funded value per victim interaction; a victim who manually extends the duration can push this considerably higher.

## Likelihood Explanation

The attacker needs only: (a) their own funded EOA to register one ordinary-looking name through Manager's real, public `/register/$name` flow (no elevated privilege, no OOS Solidity assumption — demonstrated end-to-end in this PoC), and (b) public knowledge of one real, healthy `.eth` name to target (does not need to be owned by the victim or the attacker — renewal is permissionless and pays on behalf of whichever labelhash the transaction targets).

The victim's actions are the ordinary, expected renewal flow: open a link, see their own name and its correct current expiry rendered, pick a duration/token, and click "Renew Name" twice (token picker, then confirm) — the same two clicks any legitimate renewal requires. Nothing in the UI differs in any way a user could notice: the display, the current-expiry value, and the price quote page all read and render exactly like a legitimate renewal of the real name, right up to the wallet's raw calldata, which an ordinary user cannot inspect. `ThirdPartyRenewalDialog` (`apps/manager/src/features/renew/components/ThirdPartyRenewalDialog.tsx`), Manager's own shipped "you're renewing someone else's name" warning, is gated on `isOwner` computed for the PROFILE VIEW route the button is clicked from (`apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx:58-78`); it is never imported or rendered by `RenewalPage.tsx`/`RenewPricingStep.tsx`/`ConfirmPurchase.tsx`, so it structurally cannot fire on a direct-URL phishing link, which is the entire premise of this attack.

## Attack Path / Reproduction Path

1. Attacker picks a real, existing, healthy v2 `.eth` name `T` to target (public knowledge; does not need to be attacker- or victim-owned).
2. Attacker opens Manager's real `/register/<label of T>️.eth` (percent-encoded U+FE0F in the URL) and completes the ordinary registration flow: connect wallet, pay with USDC, wait for the real on-chain commit→register sequence. This yields a real, attacker-owned v2 registration whose label is byte-identical to `T`'s label plus one invisible U+FE0F.
3. Attacker sends the victim `https://app.ens.dev/renew/<label of T>️.eth`.
4. Victim (any connected EOA wallet) opens the link. The loader's expiry/protocol gate resolves against `normalize()`, which strips the invisible codepoint and returns `T`'s real, current expiry; the page renders `T`'s exact name and expiry.
5. Victim selects a duration/token and clicks "Renew Name" (token picker), then "Renew Name" again (confirm step). The wallet is asked to sign `renew(<label of T>️, duration, paymentToken, referrer)` addressed to `ETHRegistrar`.
6. Victim signs. ERC-20 rent is pulled from the victim's EOA. The mined transaction extends the ATTACKER's registration's expiry. `T`'s expiry and ownership are completely untouched.

## External Preconditions

* The attacker has a funded EOA capable of completing one ordinary Manager registration.
* The victim can use the in-scope Manager app with an ordinary connected wallet on the supported chain.
* The victim opens an attacker-supplied link (a normal phishing precondition — no special delivery mechanism required beyond sending a URL).

## Internal Preconditions

* The target name `T` is a real, currently registered, non-past-grace v2 `.eth` name at the time the victim opens the link (or a v1 name reachable via the legacy `/renew-v1/$name` route, gated by `ETHRenewerV1.isRenewable`).
* No validator in the register or renew pipelines rejects Default-Ignorable Unicode codepoints (confirmed for the entire chain: `parseName`, `validateENSName`, `normalizeDomainNameFromUrl`, `canRenewV2Name`/`isRenewableV2EthName`).

## Affected Code

* [`apps/manager/src/routes/renew/$name.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/$name.tsx) — mainline v2 loader mixes both regimes on the raw route param.
* [`apps/manager/src/routes/renew-v1/$name.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew-v1/$name.tsx) — legacy v1 loader, identical split.
* [`apps/manager/src/features/profile/service/profileName.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileName.ts) — the only ENSIP-15 `normalize()` path (Regime A).
* [`apps/manager/src/features/register-v2/utils/name-parser.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils/name-parser.ts) — the naive ASCII-only parser (Regime B), `parseName`.
* [`apps/manager/src/features/renew/utils/renewableName.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/utils/renewableName.ts) — `parseRenewableName`/`canRenewV2Name`, both naive.
* [`apps/manager/src/features/grace/utils/gracePeriod.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/grace/utils/gracePeriod.ts) — `isRenewableV2EthName`, format-only, no on-chain check.
* [`apps/manager/src/features/renew/data/queries/renewPricing.query.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/data/queries/renewPricing.query.ts) — price quote keyed on the naive label.
* [`apps/manager/src/features/renew/data/queries/v1Renewable.query.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/data/queries/v1Renewable.query.ts) — v1 gate 3, naive label sent on-chain.
* [`apps/manager/src/features/register-v2/workflow/pricing/components/PricingDomainHeader.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/pricing/components/PricingDomainHeader.tsx) — unsanitized display.
* [`apps/manager/src/features/renew/workflow/RenewalPage.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/workflow/RenewalPage.tsx) — protocol-agnostic pass-through.
* [`apps/manager/src/features/renew/state/renewalUi.context.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/state/renewalUi.context.tsx) — carries `label` unchanged.
* [`apps/manager/src/features/renew/state/renewalUi.machine.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/state/renewalUi.machine.ts) — `startRenewal`/`submittingPlainRenewal`, zero re-validation.
* [`apps/manager/src/features/renew/workflow/pricing/components/ConfirmPurchase.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/workflow/pricing/components/ConfirmPurchase.tsx) — dispatches `renewal.start` with the naive label.
* [`apps/manager/src/features/renew/utils/renewalProtocol.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/utils/renewalProtocol.ts) — `getRenewerAddress` (ETHRegistrar / ETHRenewerV1).
* [`packages/transaction-manager/src/machines/registration/registration.actors.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.actors.ts) — `encodeRenewData`/`submitRenewActor`, the terminal sink; `generateCommitment`/`encodeRegisterCall`, the attacker's setup sink.
* [`packages/transaction-manager/src/contracts/ens-sepolia.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/contracts/ens-sepolia.ts) — `ETHRegistry`/`ETHRegistrar`/`ETHRenewerV1` addresses.
* [`apps/manager/src/routes/register/$name.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/register/$name.tsx) — attacker's setup entry point.
* [`apps/manager/src/features/register-v2/data/queries/availability.query.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/data/queries/availability.query.ts) — availability check, naive normalization.
* [`apps/manager/src/utils/domain.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/utils/domain.ts) — `normalizeDomainNameFromUrl`, no Unicode handling.
* [`apps/manager/src/features/shared/service/nameChainContractService.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/shared/service/nameChainContractService.ts) — `checkRealNameAvailability`.
* [`apps/manager/src/features/shared/registration/nameUtils.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/shared/registration/nameUtils.ts) — `validateENSName`, the identical ASCII-only denylist.

## Recommended Mitigation

Re-derive the canonical, normalized name at every gate that currently trusts a raw or naively-parsed route parameter, and reject the request if the naive label does not exactly equal the normalized label's corresponding raw form. Concretely:

* In both `renew/$name.tsx` and `renew-v1/$name.tsx`, after `profileExpiryQuery`/`profileOwnerQuery` resolve the normalized name, compare it against `parseRenewableName(name)`'s result and throw if they differ (rather than trusting the naive parse for anything downstream of that point).
* Have `parseName` reject (not silently pass through) any codepoint ENSIP-15 classifies as Default-Ignorable/mapped-to-empty, or — simpler and more robust — replace every downstream use of the naive `label` (price quote, display, `renewal.start`, `encodeRenewData`) with the ENSIP-15-normalized label the loader already computed for gates 1-2, so exactly one canonical string ever reaches the signed transaction.
* Apply the same fix to the registration pipeline (`availability.query.ts`, `nameChainContractService.ts`, `generateCommitment`/`encodeRegisterCall`) so an attacker cannot register a Default-Ignorable-bearing label as a setup step for this or any similar mismatch in the first place.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/a405ecfd90c47b01b1b85cc8a4b14101/8864566d75a4049b5242c48a356c5281ccc983de>

## Proof of Concept

**PoC Link**

<https://gist.github.com/aydarkhusaenov/a405ecfd90c47b01b1b85cc8a4b14101/8864566d75a4049b5242c48a356c5281ccc983de>

**Description**

This is a real, no-mock, end-to-end reproduction against a live Anvil fork of Sepolia, the real unmodified Manager Vite dev server, and real headless Chromium sessions driving the real UI for both actors, using the project's own `@ensdomains/headless-web3-provider` dev dependency for each actor's injected EIP-1193 wallet — nothing about the application's own source is modified.

1. An unrelated bystander registers a real, healthy v2 `.eth` name directly on-chain (scene-setting only — not the exploited path, so no UI is needed for this step).
2. The ATTACKER registers their own junk name — the bystander's label plus one invisible U+FE0F codepoint — entirely through Manager's real, unmodified `/register/$name` UI: real wallet connect, real "Pay with stablecoins", real USDC approve, real on-chain deploy-resolver→commit→(real `MIN_COMMITMENT_AGE` wait)→register sequence.
3. The VICTIM (a separate real EOA, real injected wallet) opens a crafted link to `/renew/<bystander-label>%EF%B8%8F.eth` as a fresh navigation.
4. The DOM is inspected directly: the `<h1>` text is asserted to contain the invisible U+FE0F codepoint, and stripping it is asserted to equal the bystander's exact real name — the pixel-identical-display claim, proven, not assumed.
5. The victim drives the real renewal UI to completion. The `approve()` and `renew()` transactions the wallet is asked to sign are captured from the wallet's own queue, authorized, and then independently RE-READ from the MINED chain state (not the pre-submission request) and decoded. The `renew()` `label` argument is asserted to equal the attacker's raw byte sequence and to differ from the bystander's clean label.
6. On-chain state is read before/after: the bystander's real name expiry is asserted unchanged; the attacker's junk registration's expiry is asserted increased by exactly the selected duration; the victim's real USDC balance is asserted decreased by exactly the on-chain quoted price.
7. A dedicated refutation pass actively tries to break the "PASS" framing (could the victim have noticed anything visually different; is this genuinely incorrect resolution and not mere display confusion) before accepting it.

**Where to place files and their names**

```
poc/manager-renew-v1-normalization-mismatch/
├── README.md
├── SOURCE_SHA256SUMS
├── poc.ts
├── rpc-proxy.mjs
├── run.sh
└── evidence/
```

**Tool and dependency versions**

```
Node.js:    24.18.0 (runner accepts Node.js 22/24)
pnpm:       10.27.0
Playwright: 1.58.2 (repository Chromium)
Anvil:      1.8.1
```

**Clean clone, checkout, and installation**

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git ens-audit-comp
cd ens-audit-comp
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae

mkdir -p poc
git clone https://gist.github.com/aydarkhusaenov/a405ecfd90c47b01b1b85cc8a4b14101.git \
  poc/manager-renew-v1-normalization-mismatch
git -C poc/manager-renew-v1-normalization-mismatch checkout --detach \
  8864566d75a4049b5242c48a356c5281ccc983de

corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
pnpm --filter e2e exec playwright install chromium
```

**Exact run command**

Run from the frozen repository root:

```bash
POC_LOG_DIR=/tmp/ens-renew-normalization-evidence \
bash poc/manager-renew-v1-normalization-mismatch/run.sh
```

The runner verifies the exact expected commit, starts a disposable local Anvil fork of Sepolia, a transparent RPC-logging proxy, and the real Manager Vite dev server (`VITE_FF_USE_EOA=true` — a first-party feature flag that bypasses the Rhinestone smart-account flow this local environment has no bundler infra for; the renewal flow itself is unconditionally plain-EOA regardless of this flag), then runs `poc.ts` (executed from a generated, import-path-adjusted copy inside `e2e/` — see the comment above `GENERATED_POC` in `run.sh` for why), and tears everything down on exit.

**Fresh logs proving the bug**

A real, clean run of this exact PoC against the frozen commit, via the exact command above (verified by `run.sh`'s own PASS gate, which greps the log for the marker before printing "PASS — full evidence in ..."), produced (full untruncated log and `result.json` are in `evidence/`):

```
[poc] === Phase 1: bystander registers the real target name (direct on-chain, unrelated 3rd party) ===
[poc] CLEAN_LABEL      = "r0171bystander-1789152437"
[poc] MALICIOUS_LABEL  = "r0171bystander-1789152437️" (CLEAN_LABEL + U+FE0F)
[poc] on-chain confirmed: ETH_REGISTRY.getExpiry(labelhash("r0171bystander-1789152437")) = 1791571710

[poc] === Phase 2: source-level normalization-divergence proof ===
[poc] normalize("r0171bystander-1789152437️.eth") === "r0171bystander-1789152437.eth" (U+FE0F stripped)
[poc] parseName("r0171bystander-1789152437️.eth").label === "r0171bystander-1789152437️" (U+FE0F retained, byte-for-byte)
[poc] labelhash("r0171bystander-1789152437")                    = 0x5f2569f0de74cdf4e4815eafd343a0c4a3e6f7931aef9b50d6ff4fb056684fdf
[poc] labelhash("r0171bystander-1789152437️", w/ U+FE0F)      = 0x6e65eeef1de4ffa50339159dfe05a1b05893ef8ebc9c50829716494f5a92bb6e  <-- DIFFERENT
[poc] Phase 2 PASS: parseName retains the invisible codepoint that normalize() strips; the two pipelines target different labelhashes for the same nominal input.

[poc] === Phase 3: ATTACKER registers the malicious label via the REAL /register/$name UI flow ===
[poc] navigating attacker to http://127.0.0.1:13041/register/r0171bystander-1789152437%EF%B8%8F.eth
[poc] wallet injection confirmed present after navigation (attempt 1/5)
[poc] attacker wallet connected: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[poc] attacker registration UI reports "Registration Complete!"
[poc] on-chain confirmed: ETH_REGISTRY.getExpiry(labelhash("r0171bystander-1789152437️")) = 1883847010 (attacker's real, extendable V2 registration)

[poc] === Phase 4: fund victim wallet with real MockUSDC ===
[poc] on-chain quoted 1-year renewal price for the ATTACKER's label = 8000021 (raw USDC units, 6 decimals)

[poc] === Phase 5: VICTIM opens the crafted link and drives the REAL /renew/$name UI flow ===
[poc] wallet injection confirmed present after navigation (attempt 1/5)
[poc] victim wallet connected: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[poc] navigating victim to the crafted link: http://127.0.0.1:13041/renew/r0171bystander-1789152437%EF%B8%8F.eth
[poc] DOM h1.textContent = "r0171bystander-1789152437️.eth"
[poc] DOM h1 codepoints  = [72, 30, 31, 37, 31, 62, 79, 73, 74, 61, 6e, 64, 65, 72, 2d, 31, 37, 38, 39, 31, 35, 32, 34, 33, 37, fe0f, 2e, 65, 74, 68]
[poc] Visually (invisible codepoint stripped): "r0171bystander-1789152437.eth" — pixel-identical to the real bystander name the victim believes they are renewing
[poc] clicked "Renew Name" on the confirm step — renewal.start dispatched
[poc] authorizing wallet transaction [approve()] (pre-tx block 11683769)
[poc]   mined [approve()]: hash=0x6caa30a68941d34670fc6c2613bf5b2155502d74c34efeca0a3ad1ad13295f25 block=11683770 to=0x768f42455a2d082e23ceef7d51e5787c82d67a39 data=0x095ea7b3...
[poc] approve() spender=0xa88553F454b77203B0D036A05c894d555EAAa2Cc amount=8806051
[poc] authorizing wallet transaction [renew()] (pre-tx block 11683770)
[poc]   mined [renew()]: hash=0xc056e57b3a5462f3e28d23e2af9d3f0415f098a2f0c8b7bf6188ee55e0794483 block=11683771 to=0xa88553f454b77203b0d036a05c894d555eaaa2cc data=0x89d779c3...
[poc] renew() calldata decoded: label="r0171bystander-1789152437️" duration=31557600 paymentToken=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
[poc] *** renew() targets the ATTACKER'S labelhash, NOT the displayed/validated bystander target. ***
[poc] UI reports "Renewal Complete!"

[poc] === Phase 6: on-chain proof (re-read the MINED transaction, not the pre-submit request) ===
[poc] mined tx 0xc056e57b3a5462f3e28d23e2af9d3f0415f098a2f0c8b7bf6188ee55e0794483: renew(label="r0171bystander-1789152437️", ...) — bytes-exact match to the attacker's registration
[poc] "r0171bystander-1789152437.eth" (bystander's real name; victim believed they renewed this) expiry: 1791571710 -> 1791571710
[poc] "r0171bystander-1789152437️.eth" (attacker's junk registration) expiry: 1883847010 -> 1915404610
[poc] victim USDC balance: 11678077516 -> 11670072015 (spent 8005501)
[poc] CONFIRMED: the victim paid real USDC; the ATTACKER'S junk registration was renewed; the bystander's real name the victim believed they were paying for is completely unaffected.

[poc] === Phase 7: gate 1-2 (ownership/expiry, real name) vs gate 3/price (ETHRegistrar, malicious label) RPC-call divergence ===
[poc] gate 1-2 ETH_REGISTRY.getExpiry(labelhash(CLEAN_LABEL))     calls observed: 0
[poc] gate 1-2 ETH_REGISTRY.getExpiry(labelhash(MALICIOUS_LABEL)) calls observed: 0  (expected 0 — gate 1-2 never resolves the malicious id)
[poc] gate 1-2 UniversalResolver.findOwner(dns(CLEAN_LABEL))      calls observed (substring match): 1
[poc] gate 1-2 UniversalResolver.findOwner(dns(MALICIOUS_LABEL))  calls observed (substring match): 0  (expected 0)
[poc] gate 3 / price ETHRegistrar.getRenewPrice(MALICIOUS_LABEL)  calls observed: 0
[poc] gate 3 / price ETHRegistrar.getRenewPrice(CLEAN_LABEL)      calls observed: 0  (expected 0 — gate 3/price never re-checks the clean/displayed name)

[poc] === Phase 8: refutation checks (trying to break the PASS before accepting it) ===
[poc] (a) victim-could-have-noticed check: visually-stripped DOM text = "r0171bystander-1789152437.eth" vs real target "r0171bystander-1789152437.eth" -> IDENTICAL (bug holds: no visual signal existed)
[poc] (b) no-missed-revalidation check: gate 1-2's getExpiry was never observed called with the malicious label's id (count=0); no evidence of a second, hidden re-check against the malicious registration.
[poc] (c) display-confusion-vs-incorrect-resolution check: submitted renew() label ("r0171bystander-1789152437️") != displayed/validated target ("r0171bystander-1789152437") -> INCORRECT RESOLUTION (exceeds known EXP-INPUT-005 ceiling, which is display-confusion only)
[poc] Phase 8: refutation attempts did NOT break the finding. PASS stands.
{
  "marker": "ENS_R0171_RENEW_NORMALIZATION_LABEL_SPLIT_PASS",
  "variant": "v2 (mainline /renew/$name -> ETHRegistrar)",
  "cleanLabel": "r0171bystander-1789152437",
  "maliciousLabel": "r0171bystander-1789152437️",
  "attacker": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "victim": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
  "bystander": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
  "ethRegistrar": "0xa88553F454b77203B0D036A05c894d555EAAa2Cc",
  "ethRegistry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
  "approveTxHash": "0x6caa30a68941d34670fc6c2613bf5b2155502d74c34efeca0a3ad1ad13295f25",
  "renewTxHash": "0xc056e57b3a5462f3e28d23e2af9d3f0415f098a2f0c8b7bf6188ee55e0794483",
  "cleanExpiryBefore": "1791571710", "cleanExpiryAfter": "1791571710",
  "maliciousExpiryBefore": "1883847010", "maliciousExpiryAfter": "1915404610",
  "victimBalanceBefore": "11678077516", "victimBalanceAfter": "11670072015",
  "victimUsdcSpent": "8005501",
  "rpcEvidence": {
    "gateExpiryCleanCalls": 0, "gateExpiryMaliciousCalls": 0,
    "gateFindOwnerCleanCalls": 1, "gateFindOwnerMaliciousCalls": 0,
    "gate3PriceMaliciousCalls": 0, "gate3PriceCleanCalls": 0
  }
}

$ tail -3 stdout.log
PASS — full evidence in /tmp/ens-renew-normalization-evidence
```

`maliciousExpiryAfter - maliciousExpiryBefore = 31557600` seconds — exactly the 1-year duration the victim selected. `victimUsdcSpent = 8005501` raw USDC units (\~$8.01) matches the on-chain quoted price to the unit. The wire-level capture in Phase 7 independently confirms the mechanism from a third angle: the SSR loader's ownership resolution (`UniversalResolver.findOwner`) was observed called with the CLEAN name's DNS-encoded bytes and never with the malicious ones, while `getExpiry` was captured 0 times for either id via that specific decode path (the loader used the `findOwner` path for protocol resolution on this run, consistent with `profileExpiry.ts:87-88`'s `protocol ?? (yield* getOwner(...))?.protocol` fallback) — gate 1-2 never touched the attacker's labelhash at any point, while the mined `renew()` transaction (Phase 6) targeted it exclusively.

**Generated evidence files**

The run preserves these files in `POC_LOG_DIR` (`evidence/` in the submitted package):

```
01-attacker-register-pricing.png / 02-attacker-registration-complete.png
03-victim-pricing-page-pixel-identical.png / 04-victim-confirm-step-pixel-identical.png
05-victim-renewal-complete.png
anvil.log / manager.log / proxy.log / poc.log / stdout.log / rpc-calls.ndjson / result.json
```
