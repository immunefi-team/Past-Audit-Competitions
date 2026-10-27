# 92155 w a critical renewal flow checks and displays one ens name but signs renew calldata for a different one so the user pays to extend a name they were never shown

**Submitted on Sep 7th 2026 at 15:22:42 UTC by @JollyCat68056 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92155
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

Both Manager renewal routes decide one thing and do another. Every eligibility gate — protocol version, grace period, renewability, and the expiry date printed on the page — is evaluated against the ENSIP-15-normalized name. The label that becomes the first ABI argument of `renew()` is derived separately, by a parser whose only transform is `toLowerCase()`. Nothing reconciles the two. When they differ, the app checks one registration and transacts against another, and the expiry date it shows the user is a true fact about a name the transaction never touches. The result is a wallet signature whose arguments do not match the operation the UI described.

I am aware of EXP-INPUT-005 and address it directly below rather than relying on this being a different entry point, which I know the rules disallow as a distinguisher. In short: that entry is about validators admitting input that normalization would *reject*, and its impact is explicitly bounded to "display confusion... rather than incorrect resolution". Here nothing is rejected, the divergence is between two normalizations inside one flow, and the outcome is calldata that does not match the screen. I also invoke the preamble carve-out for "new consequences of a listed root cause that materially change its severity".

## Vulnerability Details

Two routes are affected: `/renew/$name` (v2) and `/renew-v1/$name` (v1). Both do the same thing.

In `apps/manager/src/routes/renew/$name.tsx` the loader takes the route param and sends it down two paths that normalize differently. Line 20:

```ts
const parsedName = parseRenewableName(name)
```

which goes through `parseName` in `features/register-v2/utils/name-parser.ts`. The only transform there is line 46:

```ts
const normalized = name.trim().toLowerCase()
```

plus a character blacklist at line 11:

```ts
const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/
```

No ENSIP-15, and no invisible characters in that blacklist, so U+00AD and U+FE0F both survive.

Line 27 of the same loader takes the other path:

```ts
const expiryData = await queryClient.ensureQueryData(profileExpiryQuery(name))
```

`getExpiry` in `features/profile/service/profileExpiry.ts:77` runs `normalizeEth2LdName(name)`, which is viem's ENSIP-15 `normalize` (`features/profile/service/profileName.ts:18`). So the expiry that gets displayed belongs to the normalized name, i.e. the victim's real name.

The loader then returns both, at line 61:

```ts
return {
  label: parsedName.value.label,   // raw
  currentExpiry: expiryData.expiry, // normalized name's real expiry
}
```

Every gate in between is evaluated against `expiryData`, which is why the page looks completely normal: the protocol check (`expiryData?.protocol !== 'v2'`), `isPastGracePeriod` at line 44, and `canRenewV2Name` at line 52 all describe the victim's genuine name. `canRenewV2Name` re-parses the raw string but only tests it against `/^[^.]+\.eth$/` (`features/grace/utils/gracePeriod.ts:48`), which a soft hyphen passes, then checks the date, which again is the real name's.

From there the raw label goes straight to the wallet. I traced it hop by hop:

1. `routes/renew/$name.tsx:61` returns `label` from the loader
2. `RenewalPage` passes it into `RenewalUiProvider`
3. `features/renew/workflow/pricing/components/ConfirmPurchase.tsx:14` reads it from context, and line 64 sends `{ type: 'renewal.start', label, ... }`
4. `features/renew/state/renewalUi.machine.ts:148-151` stores it as `submissionData.label`
5. same file line 349 hands it to the actor as `{ label: submission.label, ... }`
6. `packages/transaction-manager/src/machines/registration/registration.actors.ts:1072` calls `encodeRenewData(input.label, ...)`
7. lines 1020-1031:

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

8. lines 1081-1085 wrap that in `createTransactionRequest({ calls: [{ to: renewerAddress, data: renewData }] })`, a plain wallet transaction the user signs directly.

Nothing on that path normalizes. `grep -rniE "from 'viem/ens'|ens_normalize|normalizeEth" apps/manager/src/features/renew/` returns nothing, and the same grep over `registration.actors.ts` returns nothing.

The price is fetched with the raw label too — `ConfirmPurchase.tsx:22` calls `getRenewPriceQueryOptions(label, ...)`, which ends up at `ETHRegistrar.getRenewPrice(label, ...)`. Both labels are the same length so the number looks right, and nothing on screen contradicts the rest of the page.

The last thing the user sees before signing is the transaction description, built at `registration.actors.ts:1090`:

```ts
description: `Renew ${input.label}.eth`,
```

Same raw label. With the soft-hyphen variant that renders as `Renew vitalik.eth`.

`routes/renew-v1/$name.tsx` has the identical split: `parseRenewableName(name)` at line 15, `profileOwnerQuery(name)` at line 21 and `profileExpiryQuery(name, 'v1')` at line 36 (both normalize), and `label: parsedName.value.label` at line 51.

Worth noting how isolated normalization is in this app. The whole Manager has three ENSIP-15 call sites — `profileName.ts:18`, `profileName.ts:48`, and `setPrimaryName.ts:49`. Every other name-consuming path, including every renewal write, is unnormalized. That is why two branches of one route can disagree.

## Impact Details

Selected impact (Critical): "Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters."

The wallet is asked to sign `renew(label, ...)` where `label` is not the label whose expiry, protocol and renewability the app just checked and displayed. The argument does not match the operation the interface described.

To be precise about where the money goes: the rent is paid to the ETHRegistrar, not to the attacker. I am not claiming direct theft. What the victim loses is the fee plus the renewal they believed they had bought; what the attacker gains is an extended registration funded by someone else. The victim's own name is untouched and keeps moving toward expiry, which is the part that costs them most if it lapses.

The centrepiece is that the app states a false fact and invites the user to rely on it. `currentExpiry` (`renew/$name.tsx:62`) is the victim's genuine expiry, read from the normalized name, and it is rendered as the baseline the new duration is added to — for a transaction that never touches that name. This is checkable: pick two names whose real expiries are a year apart, and the page will show the first name's date while the calldata carries the second name's label. Nothing else on screen contradicts it, because the price is fetched with the raw label too (`ConfirmPurchase.tsx:22`) and both labels are the same length, and because the wallet-facing description at `registration.actors.ts:1090` is built from that same raw label.

There is no ownership check anywhere on these routes, which is why the deception survives scrutiny: a user who stops to confirm that the name and expiry really are theirs gets a genuine confirmation, because both come from the normalized lookup.

On duplicates: the closest known issue is EXP-INPUT-005 (Medium), which says name validators accept homoglyphs, bidi and control characters, and that "the practical impact is homograph-style display confusion in our UI rather than incorrect resolution". I want to be straightforward that `parseName` is a permissive validator of the kind that entry describes. The reason this is still a separate issue is that the defect being reported is the mismatch, not the permissiveness. A validator that simply rejected those characters would not describe what is wrong here: one flow applies two different normalizations to the same input, ENSIP-15 for what it displays and gates on, `toLowerCase()` for what it signs. Fixing `parseName` fixes it, and so does normalizing once at the route boundary. The consequence is also outside what EXP-INPUT-005 covers: the user's money pays to renew a registration they were never shown, which is a wrong transaction rather than a confusing label. The program's rules keep "new consequences of a listed root cause that materially change its severity" in scope, and Medium to Critical is material.

Not R2-02 (no injection, no CSP dependency), and not WEB-1301 or QA-05. Also not WEB-626 ("renewing one of your own names that is in grace period shows a modal implying you are extending a name owned by someone else"): that is a grace-period display defect where the name being renewed is still the correct one and only the wording misleads. Here the name being renewed is a different registration from the one displayed, and the divergence is in the calldata rather than the copy.

Worth raising before you do: this app already ships a guard for "you are renewing a name you do not own" — `shouldShowThirdPartyRenewalWarning` (`features/renew/utils/thirdPartyRenewalWarning.ts`) feeding `ThirdPartyRenewalDialog` at `features/profile/components/view/ProfileRenewAction.tsx:66`. It never fires here, for two compounding reasons. It is gated on `isOwner`, which is computed from the normalized owner, so for a twin the victim reads as the owner. And the renewal routes themselves never query ownership at all, so the guard is structurally unreachable from `/renew/$name` and `/renew-v1/$name`. The protection ENS built for exactly this situation is bypassed by the same normalization split.

Why the trigger is not EXP-INPUT-005's class. I ran viem 2.52.2, the version this repo pins, against the three examples that entry names and against the inputs that drive this report:

| Input                                                              | `normalize()`                 | Class                |
| ------------------------------------------------------------------ | ----------------------------- | -------------------- |
| `vitalik<U+00AD>.eth` (soft hyphen) — this report                  | returns `vitalik.eth`         | silently TRANSFORMED |
| `vitalik<U+FE0F>.eth`                                              | returns `vitalik.eth`         | silently TRANSFORMED |
| `my<U+200E>cool.eth` — the entry's *bidirectional-control* example | throws `disallowed character` | REJECTED             |
| `вitalik.eth` (Cyrillic) — its *homoglyph* example                 | throws `illegal mixture`      | REJECTED             |
| `vitalik.eth.` — its *trailing dot* example                        | throws `empty label`          | REJECTED             |

EXP-INPUT-005 concerns inputs a validator admits that normalization would refuse. Nothing here is refused. Normalization accepts these inputs and quietly returns a different name, and the defect is that one flow then uses both the input and the result as if they were the same name.

That the same split exists independently on `/renew-v1/$name` shows this is structural rather than one bad input check, and it sits in the transaction-construction path rather than at the input boundary: `grep -rniE "from 'viem/ens'|ens_normalize|normalizeEth"` over `apps/manager/src/features/renew/` and over `packages/transaction-manager/src/machines/registration/` returns nothing — the only "normalize" hits in that second path are number parsing and payment-token address lowercasing.

## Why this is not the same root cause as my profile-page report

I have filed a separate report about the Manager profile page (`/$name`) rendering a genuine owner beside an attacker-controlled `addr(60)`. These are different defects and should be assessed separately.

That report is about *reads*: two lookups on one page disagree about which node they describe, and the sink is rendered HTML. Its trigger is plain ASCII case, and it is fixed by making the profile services agree on one name.

This report is about a *write*: the eligibility gates and the transaction arguments are derived from different strings, and the sink is signed calldata sent to the ETHRegistrar. Its trigger requires a character normalization discards, and it is fixed in `parseName` / at the renewal route boundary. Fixing either one leaves the other fully intact — a profile page that agrees with itself still builds `renew()` from the raw label, and a renewal route that normalizes still leaves `profileRecords.ts:123` resolving the raw name. Different files, different sinks, different triggers, different fixes.

Fix: normalize once at the route boundary. In the `parse` for `/renew/$name`, `/renew-v1/$name` and `/register/$name`, run viem `normalize(name)` and redirect to the canonical name when it differs, or reject it. Then derive the renewal label from the same normalized value that produced the expiry on screen. Making `parseName` ENSIP-15-aware instead of `toLowerCase()`-based would also close it. A regression test asserting that the label in `renew()` calldata equals the label whose expiry was displayed would catch any recurrence.

## References

Audited revision `63772fd872af472ced58b009499355f3430c2a86`.

* `apps/manager/src/routes/renew/$name.tsx` — :20 raw parse, :27 normalized expiry, :44 and :52 gates, :61 return
* `apps/manager/src/routes/renew-v1/$name.tsx` — :15, :21, :36, :51 (same split)
* `apps/manager/src/features/register-v2/utils/name-parser.ts` — :11 blacklist, :46 `trim().toLowerCase()`
* `apps/manager/src/features/profile/service/profileExpiry.ts:77` — `normalizeEth2LdName`
* `apps/manager/src/features/profile/service/profileName.ts:18` — viem `normalize`
* `apps/manager/src/features/grace/utils/gracePeriod.ts:41-55` — `isRenewableV2EthName`
* `apps/manager/src/features/renew/utils/renewableName.ts:40-47` — `canRenewV2Name`
* `apps/manager/src/features/renew/workflow/pricing/components/ConfirmPurchase.tsx:14,22,64`
* `apps/manager/src/features/renew/state/renewalUi.machine.ts:148-151,349`
* `packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1031,1072,1081-1085,1090`
* `apps/manager/src/routes/register/$name.tsx:24` and `apps/manager/src/utils/domain.ts:27-33` — how the twin is registered
* ENSIP-15: <https://docs.ens.domains/ensip/15>

## Link to Proof of Concept

<https://gist.github.com/fulldev1123/dea5ce900b8479c877994458d9421505>

## Proof of Concept

## Step-by-step proof of concept

Everything below runs offline. No network, no chain, no keys.

### Setup

Check out `63772fd872af472ced58b009499355f3430c2a86` and install (Node 22, pnpm 10):

```bash
pnpm install --frozen-lockfile
```

### Show that the two code paths disagree

Save this as `apps/manager/src/features/renew/poc-renew-mismatch.test.ts`. It imports the real `normalizeEth2LdName`, which drives the expiry the page displays, and the real `parseName`, which drives the `renew()` calldata, and asserts they return different labels for the same input.

```ts
import { describe, expect, it } from 'vitest'
import { normalizeEth2LdName } from '@/features/profile/service/profileName'
import { parseName } from '@/features/register-v2/utils/name-parser'

const INVISIBLE_CASES: Array<[string, string]> = [
  ['vitalik­.eth', 'U+00AD SOFT HYPHEN (renders as nothing)'],
  ['vitalik️.eth', 'U+FE0F VARIATION SELECTOR-16 (renders as nothing)'],
]

describe('renew flow pays for a different name than it displays', () => {
  it.each(INVISIBLE_CASES)('%s — %s', (input) => {
    const displayed = normalizeEth2LdName(input)   // drives the expiry shown
    const parsed = parseName(input)                // drives renew() calldata

    const displayedLabel = displayed!.label
    const renewedLabel = parsed._unsafeUnwrap().label

    console.log(
      `input=${JSON.stringify(input)}  displayed=${JSON.stringify(displayedLabel)}  renewed=${JSON.stringify(renewedLabel)}`,
    )

    expect(renewedLabel).not.toBe(displayedLabel)
    expect(displayedLabel).toBe('vitalik')  // the victim's real name
  })

  it('control: ASCII case does not diverge, because parseName lowercases', () => {
    expect(parseName('Vitalik.eth')._unsafeUnwrap().label).toBe('vitalik')
    expect(normalizeEth2LdName('Vitalik.eth')!.label).toBe('vitalik')
  })
})
```

Run it:

```bash
pnpm --filter manager exec vitest run src/features/renew/poc-renew-mismatch.test.ts
```

Output:

```
  input = "vitalik­.eth"
  DISPLAYED (normalized, drives the expiry shown) = "vitalik"
  RENEWED   (parseName, drives renew() calldata)  = "vitalik­"
  => user is shown "vitalik.eth" but pays to renew "vitalik­.eth"

  input = "vitalik️.eth"
  DISPLAYED = "vitalik"   RENEWED = "vitalik️"

 Test Files  1 passed (1)
      Tests  3 passed (3)
```

The third case is a control: plain ASCII case does not diverge here, because `parseName` lowercases. This attack needs a character that ENSIP-15 drops, which is also what makes it invisible.

### The attack, end to end

1. Attacker registers the twin label, `vitalik` followed by U+00AD. This can be done through the app itself: `/register/$name` runs the same `parseName` (`routes/register/$name.tsx:24`) and checks availability with `normalizeDomainNameFromUrl` (`utils/domain.ts:27-33`), neither of which applies ENSIP-15, so the label is taken as typed and reads as available. A direct call to the registrar works too. Cost is one registration fee.
2. Attacker sends the victim `https://app.ens.domains/vitalik%C2%AD.eth`. That profile page normalizes for the owner lookup, so it shows the victim's own address as owner and renders the normal "Renew Name" button with no third-party-renewal warning. The button carries the raw param through to `/renew/$name`. (Sending `/renew/vitalik%C2%AD.eth` directly also works.)
3. Victim lands on the renewal page. The heading reads `vitalik.eth`, because U+00AD is zero-width. The expiry shown is their own real expiry, read from the normalized name. The price is fetched for the raw label but both labels are the same length, so the figure looks correct.
4. Victim picks a duration and confirms. The wallet is asked to sign `renew("vitalik­", duration, paymentToken, referrer)` against `ETHRegistrar`, described in the UI as `Renew vitalik.eth`.
5. The transaction succeeds, because renewal is permissionless: the registrar lets anyone renew any registered label. The attacker's registration is extended with the victim's money. The victim's name is untouched and still heading toward expiry.

### Same defect on the v1 route

`routes/renew-v1/$name.tsx` has the same split: raw parse at line 15, normalized owner and expiry at lines 21 and 36, raw label returned at line 51. The test above covers both, since both routes call the same two functions.

### Optional: watch it on a local fork

Bring up the e2e stack (`pnpm e2e:infra:up`), register `vitalik` to one account and `vitalik` + U+00AD to another, run `pnpm dev:manager`, and open `/renew/vitalik%C2%AD.eth`. The page shows the first account's expiry while the outgoing calldata carries the second account's label.
