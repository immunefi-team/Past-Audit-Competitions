# 91698 w a critical crafted renew url renews an attacker s look alike name while the page displays the victim s own name and on chain expiry

**Submitted on Sep 3rd 2026 at 18:27:31 UTC by @SaiyanVegeta for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91698
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Bug Description

**Brief.** `/renew/$name` and `/renew-v1/$name` run **two different canonicalizers** over the same route param and never reconcile them. Ownership, protocol and **expiry** are resolved with ENSIP-15 `normalize()` — the victim's node — while the label placed in the `renew()` calldata comes from `parseName`, which only does `trim().toLowerCase()`. A link to the genuine app therefore renders the victim's own name and their real expiry date, and signs a renewal of an attacker-controlled node.

**Details (verified at HEAD).**

**Both renewal routes are affected. The v2 route — the current protocol, and the one demonstrated end-to-end in PoC 3 — is the weaker of the two** (see *Aggravators*); `/renew-v1` is affected by the same defect.

*Guard / display side — ENSIP-15:*

* `apps/manager/src/routes/renew/$name.tsx:26` and `apps/manager/src/routes/renew-v1/$name.tsx:21,36` call `profileExpiryQuery(name, …)` / `profileOwnerQuery(name)`.
* `features/profile/service/profileOwner.ts:25` → `normalizeEthName(params.name)`; `features/profile/service/profileExpiry.ts:77` → `normalizeEth2LdName(name)`.
* Both live in `features/profile/service/profileName.ts:18,62` and call viem's `normalize()`. So the protocol check, the existence/ownership check and **`currentExpiry`** all resolve `namehash(normalize(name))`.

*Signed side — `trim().toLowerCase()`:*

* `features/renew/utils/renewableName.ts:25` → `parseRenewableName` → `parseName` (`features/register-v2/utils/name-parser.ts:46`).
* Its entire canonicalization is `name.trim().toLowerCase()`, guarded by a `/\s/` test, a `..` test and an ASCII-punctuation blacklist `/[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/`. **There is no ENSIP-15 anywhere on this path.**` parseName\` is a syntax check being used as a canonicalizer.

*The loader hands the UI one field from each side:*

```ts
// routes/renew/$name.tsx:60-63  (routes/renew-v1/$name.tsx:50-53 is identical in this respect)
return {
  label: parsedName.value.label,     // SIGNED side  — the attacker's node
  currentExpiry: expiryData.expiry,  // GUARD side   — the victim's on-chain expiry
}
```

`label` flows `RenewalPage` → `RenewalUiProvider` → `ConfirmPurchase` → `renewalUi.machine` → `submitRenewActor` → `encodeRenewData` (`packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1031`), which places it verbatim into `renew(label, duration, paymentToken, referrer)`.

**Why the invisible character survives.** JavaScript's `\s` class covers `-` but **not `U+200B`** (zero-width space); `U+00AD` (soft hyphen) and `U+FE0F` are likewise excluded, and `toLowerCase()` does not remove any of them. All three pass `parseName` untouched while `normalize()` strips them.

**Aggravators — the renewability gate does not catch it on either route, and on v2 it embeds the very same divergence.**

* *v2 (`/renew/$name`, the current protocol — weaker):* there is **no on-chain check on the signed label at all**. `canRenewV2Name` (`features/renew/utils/renewableName.ts:40-47`) calls `isRenewableV2EthName(`${parsed.value.label}.eth`, expiryDate)`, and that function (`features/grace/utils/gracePeriod.ts:41-55`) does exactly two things: a **regex shape test** `/^[^.]+\.eth$/` on the raw label — which a zero-width-space label passes, since it contains no dot — and a grace-window check on the `expiryDate` the loader took from the **victim's** node. The gate therefore validates *the attacker's label against the victim's expiry*: the divergence is baked into the guard itself. A raw label that does not exist on-chain at all still reaches the signature step.
* *v1 (`/renew-v1/$name`):* `features/renew/data/queries/v1Renewable.query.ts:19,28` does run the real on-chain `isRenewable`, but on `parseRenewableName(name).label` — i.e. on the **attacker's** node, which is renewable precisely because the attacker owns it. The one genuine safety gate is aimed at the wrong name and passes.
* **The guard that would catch this already exists in the codebase and is simply not wired into these routes.** `features/renew/utils/thirdPartyRenewalWarning.ts` implements exactly the needed check — its own test is titled *"warns when the connected user is not the owner"* — and `ThirdPartyRenewalDialog` consumes it. But the only call site is `features/profile/components/view/ProfileRenewAction.tsx`, i.e. the profile page. Neither `routes/renew/$name.tsx` nor `routes/renew-v1/$name.tsx` compares the name's owner against the connected wallet, so a victim arriving from a link never sees the "this is not your name" notice the project already wrote. (Verified: the v2 loader gates only on expiry, protocol, grace window and `canRenewV2Name` — no ownership check at all.)

***

## Impact

The victim approves a transaction whose every inspectable field describes their own name — the rendered name, their **real** on-chain expiry, and the correct price for that tier — while the calldata renews a different node.

1. **Fee loss (direct, bounded).** The victim pays a full renewal for a name they do not own, scaled by the duration they select. The ERC-20 approval is `price + 10%`, not unlimited, so this is *not* a wallet drain.
2. **Asset loss (indirect, larger).** The victim's own name is **not** renewed, and the success screen tells them it was: the "Expires …" line is computed from `currentExpiry` (their real expiry) plus the chosen duration, so they receive a false confirmation and have no reason to re-check. The name reaches expiry, passes through grace, and becomes registrable — with the attacker best placed to take it, knowing the exact date.
3. **Attacker gain.** The attacker's registration is extended at the victim's expense.

No wallet compromise, no XSS, no cross-origin content: the victim is on the genuine origin the whole time.

***

## Risk Breakdown

**This is not the dashboard bulk-renew path — stated first, because the two are easy to conflate and they are mirror images rather than one defect.** `toBulkRenewName` (`features/dashboard/components/NamesTable.tsx`) **already applies ENSIP-15 on the signing side**: it signs `domain.normalizedName`, and any divergence there lives in the *display* string set beside it. The renew routes do the opposite — they sign the **un-normalized** `parseName` label and normalize on the *lookup* side. There is nothing to transplant from one to the other. The decisive test: **one patch does not fix both.** Correcting `toBulkRenewName` + `dashboard/utils.ts` leaves `routes/renew*/$name.tsx` returning `parseName`'s label exactly as before; correcting `parseName` / `renewableName.ts` leaves the dashboard's display string exactly as before. The input source differs too — two indexer columns versus a **URL route param** — as does the trigger: bulk renewal requires the attacker to have first moved the twin token into the victim's own dashboard, whereas this needs only a link. And this path carries a harm the dashboard one does not: the page presents the victim's **real on-chain expiry**, read from the other node, as the expiry of the name being renewed.

* **Attacker-controlled trigger.** Two steps, no privileged position. (a) Register the non-normalized look-alike label directly against the registrar — permissionless, and confirmed on the real deployed contract (`isAvailable(<twin>) == true`, PoC 3). Picking a victim label of ≥5 code points keeps the twin in the same price tier, so the displayed price is the honest one. (b) Send a link to the real origin, e.g. `https://app.ens.dev/renew/alice%E2%80%8B.eth`, framed as an expiry reminder. Neither route defines a `params.parse`, so the raw param reaches the loader unmodified.
* **No dependency on indexer data.** The whole attack is driven by a **URL parameter**. Nothing has to appear in, or be served by, the indexer.
* **Not EXP-INPUT-005.** That issue is scoped to display confusion "rather than incorrect resolution". Here the app *resolves two different nodes at once* — it reads owner/protocol/expiry from `namehash(normalize(name))` and signs `labelhash(parseName(name))` — and funds move. The fix locus is the renew loaders / `parseName`, not any display component.
* **Not EXP-INPUT-009 / the retry family.** No notification payload and no retry state are involved.
* **Distinct from the profile-page resolution issue.** That defect lives in `features/profile/service/profileRecords.ts` — the records lookup keyed on the raw name while its sibling `profileOwner.ts` normalizes — and its fix is to make those two lookups agree. That patch touches neither renew route and leaves `parseName` unchanged.
* **Distinct fix locus here:** apply ENSIP-15 (or assert equality against it) in `features/renew/utils/renewableName.ts` / `features/register-v2/utils/name-parser.ts`, or in the two route loaders. See the mirror-image analysis at the top of this section.
* **Severity High (honest ceiling).** Per-event loss is one renewal fee; the larger harm — the victim's name lapsing — is delayed rather than immediate, and the victim must click the attacker's link and complete the renewal. No connected-wallet tampering and no unlimited approval, so I do not claim Critical.

***

## Recommendation

Make the renew routes derive the signed label from the **same** canonicalization used for the ownership/expiry lookups:

1. In `features/renew/utils/renewableName.ts`, apply ENSIP-15 (`normalizeEth2LdName`) and return that label — or, minimally, assert `` `${parsed.label}.eth` === normalizeEth2LdName(name)?.name `` and reject on mismatch.
2. Better, fix the root: `parseName` is a syntax validator, not a canonicalizer. Every call site that feeds a transaction should normalize; consider renaming it so the distinction is not re-lost.
3. Point the v1 `isRenewable` check (`v1Renewable.query.ts:28`) at the canonical label, and add an existence/renewability check to the v2 route.
4. Wire the existing `thirdPartyRenewalWarning` / `ThirdPartyRenewalDialog` (today only used by `ProfileRenewAction.tsx`) into both renew routes, so a user arriving at `/renew/<name>` for a name they do not own is warned. The primitive already exists; only the call site is missing.

***

## References

* `apps/manager/src/routes/renew/$name.tsx:20,26,52,60-63` and `apps/manager/src/routes/renew-v1/$name.tsx:15,21,36,50-53`
* `apps/manager/src/features/grace/utils/gracePeriod.ts:41-55` (`isRenewableV2EthName` — regex shape test + grace check on the victim's expiry)
* `apps/manager/src/features/renew/utils/renewableName.ts:25`
* `apps/manager/src/features/register-v2/utils/name-parser.ts:46` (`trim().toLowerCase()`, no ENSIP-15)
* `apps/manager/src/features/profile/service/profileName.ts:18,62` (viem `normalize()`)
* `apps/manager/src/features/profile/service/profileOwner.ts:25`, `profileExpiry.ts:77`
* `apps/manager/src/features/renew/data/queries/v1Renewable.query.ts:19,28`
* `packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1031` (`encodeRenewData`)

***

## Proof of Concept

## Proof of Concept

### PoC 1 — the two canonicalizers diverge (vitest, the app's real functions, 5/5 pass)

Drop into `apps/manager/src/audit/` and run `npx vitest run src/audit/renew-label-divergence.test.ts` from `apps/manager`.

```ts
import { ethRegistrarRenewSnippet } from '@ensdomains/ensjs-abi/v2'
import { decodeFunctionData, encodeFunctionData, labelhash, namehash } from 'viem'
import { describe, expect, it } from 'vitest'
import { normalizeEth2LdName } from '@/features/profile/service/profileName'
import { parseRenewableName } from '@/features/renew/utils/renewableName'

const VICTIM_NAME = 'alice.eth'
const VICTIM_LABEL = 'alice'

// exactly what encodeRenewData() does (registration.actors.ts:1020-1031)
const buildRenewCalldata = (label: string) =>
  encodeFunctionData({
    abi: ethRegistrarRenewSnippet,
    functionName: 'renew',
    args: [label.replace('.eth', ''), 31557600n,
      '0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238',
      '0x0000000000000000000000000000000000000000000000000000000000000000'],
  })

const CRAFTED = [
  ['ZWSP U+200B', 'alice​.eth'],
  ['soft hyphen U+00AD', 'alice­.eth'],
  ['fullwidth uppercase', 'ＡＬＩＣＥ.eth'],
  ['variation selector U+FE0F', 'alice️.eth'],
] as const

describe('renew route: guard normalizes, signature does not', () => {
  it.each(CRAFTED)('%s — guard resolves the victim, calldata signs another node', (_k, crafted) => {
    const parsed = parseRenewableName(crafted)
    expect(parsed.isOk()).toBe(true)                       // the route accepts it
    const signedLabel = parsed._unsafeUnwrap().label

    const guard = normalizeEth2LdName(crafted)
    expect(guard!.name).toBe(VICTIM_NAME)                  // owner/expiry = the victim's node
    expect(guard!.label).toBe(VICTIM_LABEL)

    expect(labelhash(signedLabel)).not.toBe(labelhash(VICTIM_LABEL))
    expect(namehash(`${signedLabel}.eth`)).not.toBe(namehash(VICTIM_NAME))

    const decoded = decodeFunctionData({
      abi: ethRegistrarRenewSnippet, data: buildRenewCalldata(signedLabel),
    })
    expect(decoded.args?.[0]).toBe(signedLabel)            // calldata carries the twin
    expect(decoded.args?.[0]).not.toBe(VICTIM_LABEL)
  })
})
```

Result: `Test Files 1 passed · Tests 5 passed`.

### PoC 2 — the divergence, standalone against the repo's pinned viem

```
JS \s matches U+200B?  false  | U+00AD? false  | U+200C? false

victim owns: "alice.eth"  labelhash= 0x9c0257114eb9399a

kind                   | parseName label (SIGNED)     | normalize (GUARD/LOOKUP) | same node?
ZWSP  U+200B           | "alice​" 0x2707eeb0          | alice.eth 0x9c025711     | DIFFERENT
soft hyphen U+00AD     | "alice­" 0x98438388          | alice.eth 0x9c025711     | DIFFERENT
ZWNJ  U+200C           | label="alice‌"               | GUARD REJECTED (null)
fullwidth uppercase    | "ａｌｉｃｅ" 0xb01db196        | alice.eth 0x9c025711     | DIFFERENT
variation sel U+FE0F   | "alice️" 0xb9c6c557          | alice.eth 0x9c025711     | DIFFERENT

=> crafted names that pass parseName, resolve to the victim under ENSIP-15,
   yet sign a different node: 4 / 5
```

(The U+200C row is included deliberately: `normalize()` rejects it outright, so the route errors instead of mis-signing — the guard is not uniformly broken, only unreconciled with the signing path.)

### PoC 3 — the exact harm, end-to-end on the real deployed contracts (Sepolia fork)

`anvil --silent --port 8545 --chain-id 11155111 --fork-url <any Sepolia RPC>`, then `node onchain__s1-e2e-fork.mjs` (attached). The script embeds byte-exact copies of the app's `parseName` and `normalizeEth2LdName`, registers the victim's name and the attacker's twin against the **real** v2 registrar, and sends the `renew()` the app would encode for the crafted URL. All writes are served by the local fork; nothing is sent to Sepolia.

```
crafted URL param      : "alice459459​.eth"   (/renew/alice459459%E2%80%8B.eth)
parseName -> SIGNED    : "alice459459​"      labelhash 0x224934026a
normalize -> GUARD     : "alice459459.eth"   labelhash 0x15bd36a73a
  the two differ       : true
  guard == victim name : true

registrar isAvailable(twin with U+200B): true
registered "alice459459.eth"  -> VICTIM
registered "alice459459​.eth" -> ATTACKER (twin)

what the PAGE shows (currentExpiry, read via the GUARD/normalized node): 1790878719
calldata decodes to    : renew(label="alice459459​", ...)
  displayed name was   : "alice459459.eth"

=== S1 END-TO-END RESULT (real renew tx on real forked contracts) ===
{
  "victim_own_name_expiry_delta": "0",
  "attacker_twin_expiry_delta":   "2419200",   // +28 days
  "victim_usdc_spent":            "613701"
}

ASSERTIONS:
  registrar accepted the non-normalized twin      : true
  guard/expiry resolved the VICTIM's node         : true
  signed label != guarded label                   : true
  ATTACKER twin extended by the victim's renewal  : true
  VICTIM's own name NOT renewed                   : true
  VICTIM paid for it                              : true

>>> S1 HARM REPRODUCED END-TO-END <<<
```

Every link is demonstrated rather than inferred: the registrar accepts the non-normalized label, the guard resolves the victim's node (so the page shows the victim's own expiry), the calldata carries the twin, and on-chain the attacker's registration gains the paid duration while the victim's gains nothing and the victim's balance falls.
