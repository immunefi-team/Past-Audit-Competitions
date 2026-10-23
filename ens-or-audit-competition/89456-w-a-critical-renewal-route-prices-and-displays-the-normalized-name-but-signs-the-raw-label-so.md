# 89456 w a critical renewal route prices and displays the normalized name but signs the raw label so a crafted url renews an attacker s lookalike name with the victim s funds

**Submitted on Aug 19th 2026 at 13:00:45 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89456
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The renewal route resolves the name in the URL twice, through two pipelines that normalize differently.

Everything the user is shown, the name, the current expiry, the new expiry and the price, is derived from the ENSIP-15 normalized name. The label that is encoded into the `renew()` call is taken from a separate parser that never applies ENSIP-15.

For any label containing a code point that ENSIP-15 maps away, such as a zero width space, those two pipelines return different names for the same URL. The page renders the victim's real name with the victim's real expiry, and the transaction the victim signs renews a different label.

The wallet confirmation does not help. The transaction description is built from the same raw label, so it also displays as the victim's name.

## Where the two pipelines diverge

Route loader, `apps/manager/src/routes/renew/$name.tsx`:

* line 20, `parseRenewableName(name)` produces the label that is later signed
* lines 26 to 28, `profileExpiryQuery(name)` produces the expiry that is displayed
* line 51, `canRenewV2Name(name, expiryDate)` gates the route
* lines 58 to 61, the loader returns `parsedName.value.label` alongside `expiryData.expiry`

Path A, the signed label. `parseRenewableName` in `apps/manager/src/features/renew/utils/renewableName.ts` delegates to `parseName` in `apps/manager/src/features/register-v2/utils/name-parser.ts`. At line 46 that parser does `name.trim().toLowerCase()`, at line 49 it rejects whitespace, and at line 11 it rejects a fixed set of punctuation. There is no ENSIP-15 or UTS-46 step anywhere in this path. A zero width space is not whitespace under `/\s/` and is not in the punctuation set, so it survives into the label.

Path B, the displayed data. `profileExpiry.ts` line 77 calls `normalizeEth2LdName(name)`, and line 106 fetches the expiry for `ethName.name`. `profileName.ts` line 1 imports `normalize` from `viem/ens`, which is ENSIP-15, and line 19 applies it. `profileOwner.ts` resolves ownership the same way.

The raw label then reaches the chain unchanged. `packages/transaction-manager/src/machines/registration/registration.actors.ts` lines 1020 to 1031 encode `renew(cleanLabel, duration, paymentToken, REFERER_ADDRESS)` where `cleanLabel` is the loader's label with only `.eth` stripped. Line 1089 builds the human readable description from the same value. Pricing uses it too, in `renewPricing.query.ts` line 39, and the header renders it verbatim in `PricingDomainHeader.tsx` line 9.

## Why this is not the known issue about deceptive characters

The published known issues list an entry for name validators accepting homoglyphs, bidirectional control characters and other deceptive input, rated Medium, with the stated impact being display confusion in the interface rather than incorrect resolution.

This report is a different consequence of that class, and the difference changes the severity.

The known entry describes text that looks misleading. What happens here is that the application resolves ownership, expiry and price for one name, presents all of it as fact, and then submits a transaction against a different name. The user is not confused by how something looks. The user pays to extend a name they do not own, while the name they intended to renew continues to expire. Nothing in the interface or in the wallet prompt exposes the substitution at any point.

The competition rules state that a new consequence of a listed root cause that materially changes its severity is a new finding. Display confusion and loss of funds with a silent failure to renew are not the same severity.

## Scope correction

Punycode does not reach this state. `normalize()` throws on an `xn--` label, so `normalizeEth2LdName` returns null, the expiry query fails and the route errors out before rendering. Only code points that ENSIP-15 maps away, rather than rejects, produce the divergence. This is stated so the finding is not read as broader than it is.

## Impact

An attacker converts a single crafted link into a renewal payment made by the victim against a name the attacker controls. The victim's own name is not renewed and can lapse into grace and then expire. Every value the victim uses to make the decision, the name, the ownership, the current expiry, the new expiry and the price, is correct for their own name, so there is no point in the flow at which the substitution is visible.

## Recommended fix

Derive the transaction label from the same normalized name that produced the displayed data.

In `apps/manager/src/routes/renew/$name.tsx`, replace the loader's use of `parsedName.value.label` with the label taken from `normalizeEth2LdName(name)`. Alternatively, reject the route when the parsed label differs from the ENSIP-15 normalized label and redirect to the canonical URL.

The same rule should apply anywhere a label is taken from a route parameter and later encoded into a transaction, so that the value that is priced and displayed is always the value that is signed.

## Proof of Concept

Two tests, both run against the competition tree with no chain access required.

Part 1 shows the two name pipelines disagree on the same input. Part 2 runs the shipped route loader itself and shows it returning the attacker's label together with the victim's expiry from a single URL.

### Part 1: the two pipelines disagree

Save as `apps/manager/src/features/renew/poc-renew-label-split.test.ts` and run:

```sh
pnpm --filter manager exec vitest run src/features/renew/poc-renew-label-split.test.ts
```

Result: 7 tests passed.

```ts
import { describe, expect, it } from 'vitest'
import { canRenewV2Name, parseRenewableName } from '@/features/renew/utils/renewableName'
import { normalizeEth2LdName } from '@/features/profile/service/profileName'

// U+200B ZERO WIDTH SPACE, built by code point so it cannot be lost in copy and paste.
// It renders as nothing in a URL bar, in the page header and in a wallet prompt.
const ZWSP = String.fromCharCode(0x200B)
const VICTIM = 'vitalik.eth'
const CRAFTED = `vitalik${ZWSP}.eth`

describe('renewal renders one name and signs another', () => {
  it('the crafted URL is visually identical to the victim name', () => {
    expect(CRAFTED).not.toBe(VICTIM)
    expect(CRAFTED.replace(ZWSP, '')).toBe(VICTIM)
  })

  it('PATH A: the label that becomes the renew() argument keeps the invisible character', () => {
    const parsed = parseRenewableName(CRAFTED)
    expect(parsed.isOk()).toBe(true)

    const label = parsed._unsafeUnwrap().label
    expect(label).toBe(`vitalik${ZWSP}`)
    expect(label).not.toBe('vitalik')
    expect(label.length).toBe(8) // 7 visible glyphs plus 1 invisible
  })

  it('PATH B: the name used to look up the expiry drops the invisible character', () => {
    const normalised = normalizeEth2LdName(CRAFTED)
    expect(normalised).not.toBeNull()
    expect(normalised?.name).toBe(VICTIM)
  })

  it('THE SPLIT: one URL resolves to two different names in the same request', () => {
    const signedLabel = parseRenewableName(CRAFTED)._unsafeUnwrap().label
    const displayedName = normalizeEth2LdName(CRAFTED)?.name

    expect(`${signedLabel}.eth`).not.toBe(displayedName)
    expect(displayedName).toBe(VICTIM)         // what the page shows and prices
    expect(`${signedLabel}.eth`).toBe(CRAFTED) // what the wallet actually signs
  })

  it('the eligibility gate passes using the victim name expiry', () => {
    const victimExpiry = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
    expect(canRenewV2Name(CRAFTED, victimExpiry)).toBe(true)
  })

  it('control: a plain name keeps both paths in agreement', () => {
    const signedLabel = parseRenewableName(VICTIM)._unsafeUnwrap().label
    const displayedName = normalizeEth2LdName(VICTIM)?.name

    expect(`${signedLabel}.eth`).toBe(displayedName)
    expect(signedLabel).toBe('vitalik')
  })

  it('control: punycode does NOT reach this state, it is rejected earlier', () => {
    expect(normalizeEth2LdName('xn--ls8h.eth')).toBeNull()
  })
})
```

### Part 2: the shipped loader wires one into the transaction and the other into the display

Save as `apps/manager/src/features/renew/poc-renew-loader-split.test.ts` and run:

```sh
pnpm --filter manager exec vitest run src/features/renew/poc-renew-loader-split.test.ts
```

Result: 2 tests passed.

This calls the exported `Route.options.loader` from `apps/manager/src/routes/renew/$name.tsx` directly. The only thing stubbed is the query client, which returns the expiry the chain would return for the normalised name. Everything else is production code.

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { normalizeEth2LdName } from '@/features/profile/service/profileName'

const ZWSP = String.fromCharCode(0x200B)
const VICTIM = 'vitalik.eth'
const CRAFTED = `vitalik${ZWSP}.eth`

// Expiry the chain returns for the REAL vitalik.eth: 60 days out, comfortably
// renewable and not in grace.
const VICTIM_EXPIRY_SECONDS = BigInt(
  Math.floor(Date.now() / 1000) + 60 * 24 * 60 * 60,
)

vi.mock('@/features/renew/workflow/RenewalPage', () => ({ RenewalPage: () => null }))
vi.mock('@/features/renew/workflow/components/RenewalRouteError', () => ({
  RenewalRouteError: () => null,
}))

const seenQueries: unknown[] = []

const queryClient = {
  ensureQueryData: async (options: { queryKey: unknown }) => {
    seenQueries.push(options.queryKey)
    // What the chain returns for the normalised name, i.e. the victim's name.
    return { protocol: 'v2', expiry: VICTIM_EXPIRY_SECONDS }
  },
}

describe('PoC part 2: the shipped loader returns the raw label with the victim expiry', () => {
  beforeEach(() => {
    seenQueries.length = 0
  })

  it('one crafted URL produces the attacker label and the victim expiry together', async () => {
    const { Route } = await import('@/routes/renew/$name')

    const result = await Route.options.loader({
      params: { name: CRAFTED },
      context: { queryClient },
    } as never)

    // This is the value handed on to the renew() call.
    expect(result.label).toBe(`vitalik${ZWSP}`)
    expect(result.label).not.toBe('vitalik')

    // This is the expiry rendered to the user, resolved through the
    // normalising path, so it belongs to the victim's real name.
    expect(result.currentExpiry).toBe(VICTIM_EXPIRY_SECONDS)

    // The expiry lookup was performed, and the normalising helper on that path
    // maps the crafted URL onto the victim's name.
    expect(seenQueries.length).toBe(1)
    expect(normalizeEth2LdName(CRAFTED)?.name).toBe(VICTIM)
  })

  it('control: a clean URL returns a label that matches the name it displays', async () => {
    const { Route } = await import('@/routes/renew/$name')

    const result = await Route.options.loader({
      params: { name: VICTIM },
      context: { queryClient },
    } as never)

    expect(result.label).toBe('vitalik')
    expect(`${result.label}.eth`).toBe(normalizeEth2LdName(VICTIM)?.name)
  })
})
```

## Steps to reproduce in the application

{% stepper %}
{% step %}

### Register the crafted label

The attacker registers the raw label `vitalik` followed by U+200B by calling the registrar directly. This does not go through the Manager app, so the app's parser is not involved. The label is 8 code points, which passes the length rules.
{% endstep %}

{% step %}

### Send the crafted renewal link

The attacker sends the owner of `vitalik.eth` a link to the renewal route for `vitalik%E2%80%8B.eth`, for example in a message warning that the name is expiring.
{% endstep %}

{% step %}

### Load the page

The page loads. The header reads `vitalik.eth`. The current expiry shown is the genuine on chain expiry of `vitalik.eth`. The new expiry and the price are calculated for that same 7 character tier.
{% endstep %}

{% step %}

### Confirm the renewal

The victim connects a wallet and confirms. The wallet prompt describes the action as renewing `vitalik.eth`.
{% endstep %}

{% step %}

### Observe the renewed name

`ETHRegistrar.renew` executes with the attacker's label. The attacker's name is extended using the victim's funds. The expiry of `vitalik.eth` is unchanged, and the victim believes the renewal succeeded.
{% endstep %}
{% endstepper %}

If the attacker skips step 1, the same link is still damaging. The renewal reverts after the victim has paid gas and granted a payment token allowance, and the victim's name is left unrenewed while the interface reported success conditions.
