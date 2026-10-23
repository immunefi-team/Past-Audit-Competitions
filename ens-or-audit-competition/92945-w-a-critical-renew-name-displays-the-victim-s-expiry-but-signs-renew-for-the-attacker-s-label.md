# 92945 w a critical renew name displays the victim s expiry but signs renew for the attacker s label the victim pays to extend someone else s name

**Submitted on Sep 13th 2026 at 11:36:45 UTC by @spryon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92945
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager's `/renew/$name` loader canonicalises the same URL parameter two different ways in the same function: the expiry it **displays** and the eligibility it **gates** on are read from an ENSIP-15–normalised name, while the `label` it hands to the signed `ETHRegistrar.renew()` transaction is produced by `trim().toLowerCase()` only. For any name where those two canonicalisers disagree, a victim opening an attacker-supplied link sees their own name and their own real expiry date, and then signs a transaction that renews a **different registrar entry owned by the attacker**. Exploited in production, the victim pays the full renewal fee to extend the attacker's registration, their own name is silently not renewed and continues toward expiry, and the app's one safeguard against renewing someone else's name never fires on this route.

## Vulnerability Details

### The split

Both branches originate from the same `params.name` in the same loader:

| Concern                              | Path                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Canonicalisation                | Node reached       |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- | ------------------ |
| Displayed expiry + renewability gate | [`routes/renew/$name.tsx:27`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/$name.tsx#L27) → [`profileExpiry.ts:77`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileExpiry.ts#L77) → [`profileName.ts:18`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileName.ts#L18)      | viem `normalize` (**ENSIP-15**) | the **victim's**   |
| `label` handed to the transaction    | [`routes/renew/$name.tsx:20,61`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/$name.tsx#L20-L61) → [`renewableName.ts:25`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/utils/renewableName.ts#L25) → [`name-parser.ts:46`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils/name-parser.ts#L46) | `name.trim().toLowerCase()`     | the **attacker's** |

### Every hop

**1.** The loader takes `params.name` verbatim — [`routes/renew/$name.tsx:19`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/$name.tsx#L19).

**2.** [`:20`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/$name.tsx#L20) calls `parseRenewableName(name)` ([`renewableName.ts:25`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/utils/renewableName.ts#L25)), which delegates to `parseName`. The only canonicalisation it performs is [`name-parser.ts:46`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils/name-parser.ts#L46):

```ts
const normalized = name.trim().toLowerCase()
```

Its rejection set, [`name-parser.ts:11`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils/name-parser.ts#L11), is ``/[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/`` — which contains none of the characters that produce the divergence, and `toLowerCase()` does not remove them either.

**3.** [`:27`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/$name.tsx#L27) calls `profileExpiryQuery(name)` on the **raw** param, which internally normalises ([`profileExpiry.ts:77`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileExpiry.ts#L77) → [`:104-107`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileExpiry.ts#L104-L107)) and reads the chain for the **canonical** name. This is the value shown to the user.

**4.** [`:52`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/$name.tsx#L52) `canRenewV2Name(name, expiryDate)` is itself mixed — the label comes from the weak parser, the `expiryDate` from the normalised read.

**5.** [`:61`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/$name.tsx#L61) returns `label: parsedName.value.label`, and [`:70-72`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/$name.tsx#L70-L72) renders `<RenewalPage currentExpiry={currentExpiry} label={label} />` — the victim's real expiry displayed directly beside the attacker's label.

**6.** That label reaches the signed calldata unchanged at [`registration.actors.ts:1020-1031`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.actors.ts#L1020-L1031), invoked at [`:1072`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.actors.ts#L1072) with `input.label`:

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

### This is a canonicalisation disagreement, not a permissive charset

The known-issue index carries **EXP-INPUT-005** — "name validators accept homoglyphs / bidi / control chars, leading/trailing dot". This finding is not that, and the distinction is testable rather than rhetorical.

ENSIP-15 applies **NFKC compatibility mappings**. `trim().toLowerCase()` implements none of them. So the two canonicalisers diverge on ordinary, visible, non-homoglyph characters:

| Input                            | ENSIP-15 `normalize` | `trim().toLowerCase()` | Unicode `Cf`? |
| -------------------------------- | -------------------- | ---------------------- | ------------- |
| `victim­.eth` (soft hyphen)      | `victim.eth`         | unchanged              | **yes**       |
| `ﬁnance.eth` (U+FB01 ligature)   | `finance.eth`        | unchanged              | no            |
| `①.eth` (U+2460 circled digit)   | `1.eth`              | unchanged              | no            |
| `ｖitalik.eth` (U+FF56 fullwidth) | `vitalik.eth`        | unchanged              | no            |

The PoC reproduces the full read/write split for **all four**, and asserts the General\_Category of each: three of the four contain **no format character at all**. A validator hardened against invisible and bidi characters — the EXP-INPUT-005 remedy — would leave this finding fully exploitable, because the defect is that one loader uses two different canonical forms of one string.

### The app's own safeguard never fires

The Manager ships an interstitial for exactly this situation — `ThirdPartyRenewalDialog`, gated by [`thirdPartyRenewalWarning.ts:1-3`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts#L1-L3):

```ts
export const shouldShowThirdPartyRenewalWarning = (isOwner: boolean | undefined) =>
  isOwner === false
```

It is rendered from exactly one place — the profile page, [`ProfileRenewAction.tsx:68`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx#L68). A repo-wide grep for `owner|isOwner|profileOwner` inside [`routes/renew/$name.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/$name.tsx) returns nothing: the route performs **no ownership check of any kind**. A direct link to `/renew/<lookalike>.eth` therefore bypasses the one control built to catch this, and the victim is never told they are renewing a name they do not own.

### Delivery vectors already present in the app

* **In-app notifications.** [`name-expiry.tsx:18,25,36`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/notifications/notifications/name-expiry.tsx#L18-L36) gates on `isRenewableName(payload.name)` (lowercase only) and links to `/renew/$name` using a name supplied by the notifications **backend**. A hostile or poisoned payload routes a victim straight into the sink from inside the product.
* **Permalink forwarding.** [`routes/p/$name/index.tsx:5-10`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/p/$name/index.tsx#L5-L10) forwards the parameter verbatim.
* Ordinary links, since the URL renders identically to the victim's own name.

### Attacker prerequisites

1. **Register the lookalike label.** The registrars gate on length, not ENSIP-15 — stated in the codebase itself at [`isNameOrAddress.ts:9-10`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/token/isNameOrAddress.ts#L9-L10): *"the ENSv2 contracts impose no character constraints on labels (only a 1–255 byte length)"*. The Manager's own registration flow will mint it: there are **zero** `normalize` / `ens_normalize` calls anywhere under [`apps/manager/src/features/register-v2/utils/`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils). Cost is one ordinary registration, which the victim then reimburses.
2. **Send the victim the link.** It renders as their own name.
3. **Nothing else.** No wallet gifting, no prior relationship, no ownership check to satisfy. The victim need only confirm a renewal they already intended to perform.

### Secondary instance: `renew-v1/$name`

[`routes/renew-v1/$name.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew-v1/$name.tsx) has the same shape: `profileOwnerQuery` and `profileExpiryQuery` normalise, while the label and the `isRenewable` gate use the raw string, reaching `renew(label, …)` on `ETHRenewerV1`. That route's `isRenewable` check happens to be a raw-label on-chain read, so an *unregistered* variant throws — an incidental brake that disappears the moment the attacker's variant is itself v1-renewable. Reported here as secondary, not as a separate issue.

## Impact Details

### Selected impact: display of incorrect transaction details leading to unintended fund transfers

**Direct loss, the renewal fee.** The victim pays rent that extends the **attacker's** registration. Under ENS's standard `.eth` pricing this is roughly $5/year for names of five or more characters, \~$160/year for four characters and \~$640/year for three, multiplied by whatever duration the victim selects. The deception is unusually complete because the price, the "current expiry → new expiry" delta and the name shown are all *correct for the victim's real name* — every figure on the confirmation screen reconciles. Only the label buried in the calldata differs.

**Consequential loss — the name itself.** The victim believes the renewal succeeded. It did not. Their real name continues toward expiry and, after the grace period, can be released and re-registered by anyone, including the attacker. The loss ceiling here is the full market value of the name, which for a desirable name greatly exceeds the renewal fee. This is the "bricking / permanent inaccessibility" outcome reached by omission rather than by a direct hostile write.

**Repeatability.** Nothing is consumed by a successful attack. One registered lookalike can be sent to many victims, and each renewal extends the attacker's own registration further.

### What I am not claiming

* No compromise of keys, sessions, or the smart account.
* The PoC exercises the app's real resolution and label-derivation code with an in-memory chain; it does not submit an on-chain transaction.
* The `renew-v1` instance is reported as a secondary occurrence, not as an independent finding.

### Recommended fix

1. **Canonicalise once.** Normalise with ENSIP-15 in `params.parse` / `beforeLoad` and `redirect()` to the canonical URL, then derive the label from that single string. This also removes the class of divergence rather than this one instance.
2. **Make `parseName` agree with the read path** — [`name-parser.ts:46`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils/name-parser.ts#L46) should use `ens_normalize`, or reject any input where `normalize(name) !== name`.
3. **Assert the invariant at the boundary**: the renew machine should refuse to submit when the label being renewed differs from the name whose expiry was displayed.

Fix (1) is the highest-leverage: the same two-canonicaliser pattern exists on other name-taking routes in both apps.

## References

All repository links are permalinks pinned to commit [`1c9b47f`](https://github.com/immunefi-team/audit-comp-ens/commit/1c9b47f18fcddd2e864dfe385c4171061c9811ae).

**The split**

* [`apps/manager/src/routes/renew/$name.tsx#L19-L72`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/$name.tsx#L19-L72) — the loader containing both canonicalisations
* [`apps/manager/src/features/register-v2/utils/name-parser.ts#L46`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils/name-parser.ts#L46) — `trim().toLowerCase()`, the write-path canonicaliser
* [`apps/manager/src/features/register-v2/utils/name-parser.ts#L11`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils/name-parser.ts#L11) — the rejection set, which does not cover the divergent characters
* [`apps/manager/src/features/renew/utils/renewableName.ts#L25`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/utils/renewableName.ts#L25) — `parseRenewableName`
* [`apps/manager/src/features/profile/service/profileExpiry.ts#L77`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileExpiry.ts#L77) and [`#L104-L107`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileExpiry.ts#L104-L107) — the normalised read path
* [`apps/manager/src/features/profile/service/profileName.ts#L18`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileName.ts#L18) — ENSIP-15 `normalize`

**The sink**

* [`packages/transaction-manager/src/machines/registration/registration.actors.ts#L1020-L1031`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.actors.ts#L1020-L1031) — `encodeRenewData`
* [`packages/transaction-manager/src/machines/registration/registration.actors.ts#L1072`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.actors.ts#L1072) — call site passing `input.label`

**The safeguard that never fires**

* [`apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts#L1-L3`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts#L1-L3)
* [`apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx#L68`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx#L68) — its only render site

**Attacker prerequisite**

* [`apps/portal/src/utils/token/isNameOrAddress.ts#L9-L10`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/token/isNameOrAddress.ts#L9-L10) — "no character constraints on labels (only a 1–255 byte length)"
* [`apps/manager/src/features/register-v2/utils/`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/utils) — no `normalize` call anywhere in the register flow's utils

**Delivery**

* [`apps/manager/src/features/notifications/notifications/name-expiry.tsx#L18-L36`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/notifications/notifications/name-expiry.tsx#L18-L36) — backend-supplied name linked into `/renew/$name`

**Secondary instance**

* [`apps/manager/src/routes/renew-v1/$name.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew-v1/$name.tsx)

**Specifications**

* ENSIP-15 (ENS Name Normalization) — <https://docs.ens.domains/ensip/15>
* UAX #15 (Unicode Normalization Forms, NFKC) — <https://unicode.org/reports/tr15/>
* Unicode General\_Category `Cf` (format characters) — <https://www.unicode.org/reports/tr44/#General\\_Category\\_Values>

## Proof of Concept

The code under test: `parseRenewableName`, `parseName`, `getExpiry`, `normalizeEth2LdName` is the **real application code**, imported from the repo.

(Importing the Manager's module graph pulls in the wallet-connect config, which attempts one unrelated outbound fetch and fails harmlessly offline. It is stderr noise; no assertion depends on it.)

### Run

```bash
cp renew-label-split.test.ts apps/manager/src/__poc_renew_label_split.test.ts
cd apps/manager
npx vitest run src/__poc_renew_label_split.test.ts --reporter=verbose
rm src/__poc_renew_label_split.test.ts
```

### What each assertion establishes

| Test                   | Establishes                                                                                                                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| step 0                 | The two canonicalisers disagree; the URL renders indistinguishably (`/renew/victim%C2%AD.eth`)                                                                                                   |
| step 1                 | The loader's gate and displayed expiry are read from the **victim's** node — `getExpiry` is called with `victim.eth`, returning the victim's real 2030 expiry, not the attacker entry's 2026 one |
| step 2                 | `parseRenewableName` yields the **attacker's** label; `labelhash` differs, so it is a different registrar entry                                                                                  |
| step 3                 | The encoded `renew()` calldata differs from the honest calldata and literally contains the attacker label's UTF-8 bytes (`c2ad`)                                                                 |
| control                | A plain name keeps read and write on the same node — proving a divergence, not a blanket mismatch                                                                                                |
| divergent cases ×4     | The full read/write split reproduces for the soft hyphen **and** for the ligature, circled digit and fullwidth letter                                                                            |
| format-character check | Three of the four inputs contain **no** Unicode `Cf` character — so this is not the EXP-INPUT-005 charset issue                                                                                  |

### Observed output

```
 ✓ BUCKET B: /renew/$name display path and tx path canonicalise differently > step 0 — the two canonicalisers disagree, and the URL is invisible 308ms
 ✓ BUCKET B: /renew/$name display path and tx path canonicalise differently > step 1 — the loader gate + displayed expiry come from the VICTIM node 4ms
 ✓ BUCKET B: /renew/$name display path and tx path canonicalise differently > step 2 — the label handed to the renew tx is the ATTACKER node 3ms
 ✓ BUCKET B: /renew/$name display path and tx path canonicalise differently > step 3 — the signed calldata renews the attacker label 4ms
 ✓ BUCKET B: /renew/$name display path and tx path canonicalise differently > control — a plain name keeps read and write on the same node 1ms
 ✓ the split is a canonicalisation disagreement, not a charset problem > read/write split reproduces with U+00AD soft hyphen (format char) 1ms
 ✓ the split is a canonicalisation disagreement, not a charset problem > read/write split reproduces with U+FB01 ligature (ﬁ) 1ms
 ✓ the split is a canonicalisation disagreement, not a charset problem > read/write split reproduces with U+2460 circled digit (①) 1ms
 ✓ the split is a canonicalisation disagreement, not a charset problem > read/write split reproduces with U+FF56 fullwidth letter (ｖ) 1ms
 ✓ the split is a canonicalisation disagreement, not a charset problem > three of the four carry no format character at all 2ms

 Test Files  1 passed (1)
      Tests  10 passed (10)
   Duration  3.61s
```

### Complete PoC source

```ts
/**
 * HERMETIC PoC — /renew/$name write-path canonicalisation split (BUCKET B)
 *
 * Repo: audit-comp-ens @ apps/manager
 * Run:  cp this file to apps/manager/src/__poc_renew_label_split.test.ts
 *       pnpm --filter manager exec vitest run src/__poc_renew_label_split.test.ts
 *       then DELETE the copy.
 *
 * Every chain read is a vi.mock'd in-memory table, so no assertion depends on
 * the network. (Importing the app's module graph pulls in the wallet-connect
 * config, which attempts one unrelated outbound fetch and fails harmlessly
 * offline; it is stderr noise, not part of the tested path.)
 *
 * Claim
 * -----
 * apps/manager/src/routes/renew/$name.tsx reads the expiry/protocol that it
 * DISPLAYS and GATES on through profileExpiryQuery(name)  ->  getExpiry()
 * ->  normalizeEth2LdName()  ->  viem `normalize` (ENSIP-15).
 * It hands the RENEW TRANSACTION a label produced by
 * parseRenewableName(name) -> parseName() -> `name.trim().toLowerCase()` only.
 * Those two canonicalisations disagree on invisible characters, so the page
 * shows the VICTIM's name + the VICTIM's real expiry while the signed
 * `ETHRegistrar.renew(label, ...)` carries the ATTACKER's label.
 */
import { okAsync } from 'neverthrow'
import { encodeFunctionData, labelhash } from 'viem'
import { normalize } from 'viem/ens'
import { beforeEach, describe, expect, it, vi } from 'vitest'

// ---------------------------------------------------------------------------
// In-memory "chain". victim.eth is the real name. The U+00AD variant is a
// separate on-chain label the attacker registered for ~1 year of rent.
// ---------------------------------------------------------------------------
const VICTIM = 'victim.eth'
const LOOKALIKE = 'victim­.eth' // soft hyphen: renders identically
const VICTIM_EXPIRY = 1_893_456_000n // 2030-01-01
const ATTACKER_EXPIRY = 1_767_225_600n // 2026-01-01

const CHAIN_EXPIRY: Record<string, bigint> = {
  [VICTIM]: VICTIM_EXPIRY,
  [LOOKALIKE]: ATTACKER_EXPIRY,
}
const CHAIN_OWNER: Record<string, `0x${string}`> = {
  [VICTIM]: '0x1111111111111111111111111111111111111111',
  [LOOKALIKE]: '0x2222222222222222222222222222222222222222',
}

/**
 * The divergence is NOT confined to invisible characters.
 *
 * The known-issue index carries EXP-INPUT-005, "name validators accept
 * homoglyphs/bidi/control chars". If this finding rested only on U+00AD it
 * would sit inside that wording. It does not: ENSIP-15 also applies NFKC
 * compatibility mappings, and `trim().toLowerCase()` implements none of them.
 * The cases below are ordinary visible characters — a typographic ligature, a
 * circled digit, a fullwidth letter — none of which is a homoglyph, a bidi
 * control, or a format character. `formatChar` records that distinction and is
 * asserted against Unicode General_Category `Cf` in the test.
 */
const DIVERGENT_CASES = [
  { desc: 'U+00AD soft hyphen (format char)', raw: 'victim­.eth',
    canonical: VICTIM, canonicalLabel: 'victim', formatChar: true },
  { desc: 'U+FB01 ligature (ﬁ)', raw: 'ﬁnance.eth',
    canonical: 'finance.eth', canonicalLabel: 'finance', formatChar: false },
  { desc: 'U+2460 circled digit (①)', raw: '①.eth',
    canonical: '1.eth', canonicalLabel: '1', formatChar: false },
  { desc: 'U+FF56 fullwidth letter (ｖ)', raw: 'ｖitalik.eth',
    canonical: 'vitalik.eth', canonicalLabel: 'vitalik', formatChar: false },
] as const

// Seed the in-memory chain: each canonical name is the victim's, each raw
// variant is a separate registrar entry the attacker owns.
for (const c of DIVERGENT_CASES) {
  CHAIN_EXPIRY[c.canonical] ??= VICTIM_EXPIRY
  CHAIN_EXPIRY[c.raw] ??= ATTACKER_EXPIRY
  CHAIN_OWNER[c.canonical] ??= '0x1111111111111111111111111111111111111111'
  CHAIN_OWNER[c.raw] ??= '0x2222222222222222222222222222222222222222'
}

const expiryCalls: string[] = []
const ownerCalls: string[] = []

vi.mock('@/lib/wagmi/helpers', () => ({
  safeGetClient: () => okAsync({} as never),
}))
vi.mock('@ensdomains/ensjs/public/v2', () => ({
  getExpiry: async (_c: unknown, { name }: { name: string }) => {
    expiryCalls.push(name)
    return CHAIN_EXPIRY[name] ?? 0n
  },
  getOwner: async (_c: unknown, { name }: { name: string }) => {
    ownerCalls.push(name)
    return CHAIN_OWNER[name] ?? '0x0000000000000000000000000000000000000000'
  },
}))
vi.mock('@ensdomains/ensjs/public/v1', () => ({
  getExpiry: async () => null,
  getOwner: async () => null,
}))

import { getExpiry } from '@/features/profile/service/profileExpiry'
// Real repo code under test
import { parseRenewableName } from '@/features/renew/utils/renewableName'

describe('BUCKET B: /renew/$name display path and tx path canonicalise differently', () => {
  beforeEach(() => {
    expiryCalls.length = 0
    ownerCalls.length = 0
  })

  it('step 0 — the two canonicalisers disagree, and the URL is invisible', () => {
    // What the READ path does (profileName.ts:18 `normalize`)
    expect(normalize(LOOKALIKE)).toBe(VICTIM)
    // What the WRITE path does (name-parser.ts:46 `.trim().toLowerCase()`)
    expect(LOOKALIKE.trim().toLowerCase()).toBe(LOOKALIKE)
    expect(normalize(LOOKALIKE)).not.toBe(LOOKALIKE.trim().toLowerCase())
    // the link the victim clicks
    expect(encodeURI(`/renew/${LOOKALIKE}`)).toBe('/renew/victim%C2%AD.eth')
  })

  it('step 1 — the loader gate + displayed expiry come from the VICTIM node', async () => {
    // routes/renew/$name.tsx:27  ->  profileExpiryQuery(name)
    const res = await getExpiry(LOOKALIKE)
    expect(res.isOk()).toBe(true)
    const data = res._unsafeUnwrap()

    // ensjs was asked about victim.eth, NOT about the URL the victim opened
    expect(expiryCalls).toEqual([VICTIM])
    expect(ownerCalls).toEqual([VICTIM])

    // routes/renew/$name.tsx:40  protocol gate passes
    expect(data.protocol).toBe('v2')
    // routes/renew/$name.tsx:62  currentExpiry handed to <RenewalPage/>
    expect(data.expiry).toBe(VICTIM_EXPIRY)
    // …i.e. the pricing page shows the VICTIM's real 2030 expiry,
    // not the attacker name's 2026 expiry.
    expect(data.expiry).not.toBe(ATTACKER_EXPIRY)
  })

  it('step 2 — the label handed to the renew tx is the ATTACKER node', () => {
    // routes/renew/$name.tsx:20 + :61  ->  label returned to RenewalPage
    const parsed = parseRenewableName(LOOKALIKE)
    expect(parsed.isOk()).toBe(true)
    const label = parsed._unsafeUnwrap().label

    expect(label).toBe('victim­')
    expect(label).not.toBe('victim')
    // different on-chain label => different registrar entry
    expect(labelhash(label)).not.toBe(labelhash('victim'))
  })

  it('step 3 — the signed calldata renews the attacker label', () => {
    // Mirrors packages/transaction-manager/src/machines/registration/
    //   registration.actors.ts:1020-1031  encodeRenewData()
    const renewAbi = [
      {
        type: 'function',
        name: 'renew',
        stateMutability: 'nonpayable',
        inputs: [
          { name: 'label', type: 'string' },
          { name: 'duration', type: 'uint64' },
          { name: 'paymentToken', type: 'address' },
          { name: 'referrer', type: 'address' },
        ],
        outputs: [],
      },
    ] as const

    const label = parseRenewableName(LOOKALIKE)._unsafeUnwrap().label
    const cleanLabel = label.replace('.eth', '')

    const attackerCalldata = encodeFunctionData({
      abi: renewAbi,
      functionName: 'renew',
      args: [
        cleanLabel,
        31_536_000n,
        '0x3333333333333333333333333333333333333333',
        '0x0000000000000000000000000000000000000000',
      ],
    })
    const honestCalldata = encodeFunctionData({
      abi: renewAbi,
      functionName: 'renew',
      args: [
        'victim',
        31_536_000n,
        '0x3333333333333333333333333333333333333333',
        '0x0000000000000000000000000000000000000000',
      ],
    })

    // THE BUG: user saw "victim.eth — expires 2030 — extend by 1 year",
    // signs calldata that extends a DIFFERENT registrar entry.
    expect(attackerCalldata).not.toBe(honestCalldata)
    // the attacker label is literally in the calldata (utf-8 c2 ad)
    expect(attackerCalldata.includes('c2ad')).toBe(true)
  })

  it('control — a plain name keeps read and write on the same node', async () => {
    const res = await getExpiry(VICTIM)
    expect(expiryCalls).toEqual([VICTIM])
    expect(res._unsafeUnwrap().expiry).toBe(VICTIM_EXPIRY)
    expect(parseRenewableName(VICTIM)._unsafeUnwrap().label).toBe('victim')
  })
})

describe('the split is a canonicalisation disagreement, not a charset problem', () => {
  beforeEach(() => {
    expiryCalls.length = 0
    ownerCalls.length = 0
  })

  it.each(DIVERGENT_CASES.map((c) => [c.desc, c] as const))(
    'read/write split reproduces with %s',
    async (_desc, c) => {
      // READ path (profileExpiry.ts -> profileName.ts:18 `normalize`)
      expect(normalize(c.raw)).toBe(c.canonical)
      // WRITE path (name-parser.ts:46) does NOT reach the canonical form.
      expect(c.raw.trim().toLowerCase()).not.toBe(c.canonical)

      // Distinct registrar entries.
      expect(labelhash(c.raw.replace('.eth', ''))).not.toBe(
        labelhash(c.canonicalLabel),
      )

      // The loader reads the VICTIM's node …
      const res = await getExpiry(c.raw)
      expect(res.isOk()).toBe(true)
      expect(expiryCalls).toEqual([c.canonical])
      expect(res._unsafeUnwrap().expiry).toBe(VICTIM_EXPIRY)

      // … and hands the ATTACKER's label to the transaction.
      const label = parseRenewableName(c.raw)._unsafeUnwrap().label
      expect(label).not.toBe(c.canonicalLabel)
    },
  )

  it('three of the four carry no format character at all', () => {
    // Unicode General_Category Cf is the class EXP-INPUT-005 describes
    // ("control chars"); bidi controls and the soft hyphen live there.
    // The NFKC cases are ordinary visible letters and digits, so tightening a
    // charset against invisibles would not close this finding.
    for (const c of DIVERGENT_CASES) {
      expect(/\p{Cf}/u.test(c.raw)).toBe(c.formatChar)
    }
    expect(DIVERGENT_CASES.filter((c) => !c.formatChar)).toHaveLength(3)
  })
})
```
