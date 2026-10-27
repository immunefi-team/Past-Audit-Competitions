# 92572 w a critical ensv2 registration applies no ensip 15 normalization at any step so a crafted url prices displays and registers a label the user cannot see and that ens does not consider par&#x20;

**Submitted on Sep 10th 2026 at 17:52:26 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92572
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief

The ENSv2 registration flow in the Manager never applies ENSIP-15 normalization. Availability, price, the on-screen name and the register calldata are all derived from the raw route parameter by way of String.prototype.toLowerCase().

ENSIP-15 and toLowerCase() are not the same function. Where they disagree the Manager registers a label that is not the name it displayed, and the name it displayed stays unregistered.

U+00AD SOFT HYPHEN is the clearest case. ENSIP-15 deletes it, toLowerCase() keeps it, and it renders zero-width in every browser. A link to /register/vi\[U+00AD]talik.eth therefore shows vitalik.eth, quotes a price for vitalik.eth, and asks the connected wallet to sign register("vi\[U+00AD]talik", ...). The victim's stablecoins buy a label they were never shown.

## Vulnerability details

Four steps, none of which normalize.

Step 1. Availability. apps/manager/src/features/register-v2/data/queries/availability.query.ts:9

```
const normalizedName = normalizeDomainNameFromUrl(routeName)
```

normalizeDomainNameFromUrl is apps/manager/src/utils/domain.ts:27-33. Despite the name it only strips a trailing .eth, truncates the label to 255 bytes and re-appends .eth. There is no ENSIP-15 in it.

Step 2. Label parsing. apps/manager/src/features/register-v2/utils/name-parser.ts:46

```
const normalized = name.trim().toLowerCase()
```

That is JS case folding. INVALID\_LABEL\_CHARS on line 11 is a fixed list of ASCII punctuation and does not contain U+00AD. The length gate at routes/register/$name.tsx:40 counts code points, which the soft hyphen passes.

Step 3. Price. apps/manager/src/features/register-v2/data/queries/pricing.query.ts:68

```
args: [label, BigInt(Math.ceil(durationInSeconds)), HCA_CONTRACTS.usdc]
```

Step 4. Calldata. packages/smart-account/src/providers/rhinestone/registration-calls.ts:286

```
functionName: 'register',
args: [params.label, params.wallet, params.secret, ZERO_ADDRESS, params.resolver, params.duration, c.usdc, REFERER]
```

The same string is what the page renders: routes/register/$name.tsx:71 passes label straight into RegistrationV2UiProvider.

Grepping for ens\_normalize and viem's normalize across apps/manager/src/features/register-v2 and apps/manager/src/features/weave-registration returns nothing. @adraffy/ens-normalize is not a dependency of apps/manager at all. Only apps/portal declares it, and apps/portal does normalize: getLabel.ts:11 and isNormalized.ts:18 both call ens\_normalize. The two in-scope apps do not agree on what an ENS name is.

## Attack path

1. The attacker publishes a link to /register/vi\[U+00AD]talik.eth. The character is invisible, so the link text and the page both read vitalik.eth.
2. The route loader runs the availability query on the raw parameter. vi\[U+00AD]talik is genuinely unregistered, so the answer is available and the flow proceeds instead of redirecting to the profile page.
3. Pricing quotes on the same label.
4. The page renders that label. The soft hyphen has zero width, so the victim reads vitalik.eth.
5. The victim connects, approves USDC and signs. The registrar receives register("vi\[U+00AD]talik", ...).
6. The victim owns labelhash("vi\[U+00AD]talik"). Every ENSIP-15 client, including ENS's own Explorer, resolves the name the victim believes they bought to labelhash("vitalik"), a different name that is still unregistered.

At no point was the victim shown the string that was actually signed.

## Impact details

I have selected the connected-wallet row. The arguments in the transaction the application asks the wallet to sign are not the arguments the application displayed: the label in the calldata is a different ENS name from the label on screen, and the divergence is invisible by construction rather than merely easy to miss.

I am not claiming theft of an existing name from its owner. The loss is the registration fee plus the failure to obtain the name that was displayed and paid for, and the displayed name remains available for anyone else to take.

## Why this is not the known deceptive-characters issue

The published known issues include an entry for name validators accepting homoglyphs, bidirectional controls and other deceptive input, rated Medium, whose stated impact is display confusion rather than incorrect resolution.

This is a different defect. No confusable is involved and no validator is being asked for a judgement call. The divergence is a deletion rule that ENSIP-15 specifies and toLowerCase() does not implement, and the ENSIP-15 function is absent from the flow entirely rather than applied too leniently. The consequence is that funds buy a different on-chain name, which the known entry expressly says it is not describing.

It is also separable from the renewal seam reported earlier in this competition, which was an asymmetry between two normalizers applied to the same name. Here there is no second normalizer to be asymmetric with: a different flow, different files, and a root cause that is an omission rather than an inconsistency.

## Recommended fix

Normalize once, at the route boundary, and let every downstream step use that one value. viem is already a dependency of apps/manager:

```
import { normalize } from 'viem/ens'
```

In the /register/$name loader compute normalize(name) and use it for the availability query, the price query, the rendered label and the calldata. Where normalize(input) !== input, either redirect to the canonical URL or refuse the registration; do not silently register the raw form. Applying the same change inside parseName also covers /renew/$name and /renew-v1/$name, which share it.

## References

ENSIP-15 (ENS name normalization): <https://docs.ens.domains/ensip/15>

## Proof of Concept

Tested against audit-comp-ready at 1c9b47f18fcddd2e864dfe385c4171061c9811ae. That branch commit only bumps Hono over the upstream audited revision 63772fd872af472ced58b009499355f3430c2a86; git diff confirms every registration source cited in the report is unchanged between the two.

The test uses the application's own parseName and normalizeDomainNameFromUrl plus viem, which apps/manager already depends on. It needs no chain, no wallet and no local stack beyond the repository's own dependencies. It opens with the ENSIP-15 result as a control so the comparison cannot be dismissed as an artefact of the test's own normalization.

Save as apps/manager/src/features/register-v2/poc-registration-soft-hyphen.test.ts and run from the repository root:

```
pnpm install --frozen-lockfile
pnpm --filter manager exec vitest run src/features/register-v2/poc-registration-soft-hyphen.test.ts --reporter=verbose
```

Observed on the audited revision: 6 passed.

```
control: ENSIP-15 deletes the soft hyphen, so ENS calls this name vitalik.eth
the availability normalizer preserves it, so availability is answered for a different name
parseName preserves it, so the registered label is not the ENSIP-15 label
the two labels are different ENS names on chain
control: the character is invisible, so the two strings are visually identical
control: an ordinary ASCII name is unaffected, so this is not a blanket defect

Test Files  1 passed (1)
Tests       6 passed (6)
```

Full test source:

```ts
import { labelhash } from 'viem'
import { normalize } from 'viem/ens'
import { describe, expect, it } from 'vitest'
import { normalizeDomainNameFromUrl } from '@/utils/domain'
import { parseName } from './utils/name-parser'

// The ENSv2 registration flow never applies ENSIP-15.
//
//   availability.query.ts:9      normalizeDomainNameFromUrl(routeName)
//   name-parser.ts:46            name.trim().toLowerCase()
//   pricing.query.ts:68          args: [label, duration, usdc]
//   registration-calls.ts:286    register(label, ...)
//
// U+00AD SOFT HYPHEN is deleted by ENSIP-15, preserved by toLowerCase(), and
// renders zero-width, so the availability answer, the price, the on-screen
// label and the calldata all carry a character the user cannot see.
//
// Written as an escape so the character survives copy and paste.
const SH = '\u00AD'
const CRAFTED = `vi${SH}talik.eth`

describe('ENSv2 registration: soft hyphen survives to calldata', () => {
  it('control: ENSIP-15 deletes the soft hyphen, so ENS calls this name vitalik.eth', () => {
    expect(normalize(CRAFTED)).toBe('vitalik.eth')
  })

  it('the availability normalizer preserves it, so availability is answered for a different name', () => {
    const checked = normalizeDomainNameFromUrl(CRAFTED)
    expect(checked).toBe(CRAFTED)
    expect(checked).not.toBe(normalize(CRAFTED))
  })

  it('parseName preserves it, so the registered label is not the ENSIP-15 label', () => {
    const parsed = parseName(CRAFTED)
    expect(parsed.isOk()).toBe(true)
    if (!parsed.isOk()) throw new Error('parseName rejected the crafted name')

    expect(parsed.value.label).toBe(`vi${SH}talik`)
    expect(parsed.value.label).not.toBe('vitalik')
  })

  it('the two labels are different ENS names on chain', () => {
    expect(labelhash(`vi${SH}talik`)).not.toBe(labelhash('vitalik'))
  })

  it('control: the character is invisible, so the two strings are visually identical', () => {
    expect(CRAFTED.replace(SH, '')).toBe('vitalik.eth')
    expect([...CRAFTED].length).toBe([...'vitalik.eth'].length + 1)
  })

  it('control: an ordinary ASCII name is unaffected, so this is not a blanket defect', () => {
    const parsed = parseName('vitalik.eth')
    expect(parsed.isOk()).toBe(true)
    if (!parsed.isOk()) throw new Error('unexpected')
    expect(parsed.value.label).toBe('vitalik')
    expect(labelhash(parsed.value.label)).toBe(
      labelhash(normalize('vitalik.eth').split('.')[0]),
    )
  })
})
```
