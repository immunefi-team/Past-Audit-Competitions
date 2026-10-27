# 92568 w a critical profile editing authorizes the normalized name but encodes the raw route param so a crafted url makes the victim sign a record write to an attacker s node on an attacker s res&#x20;

**Submitted on Sep 10th 2026 at 17:39:13 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92568
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Manager's profile feature resolves the `$name` route parameter through two different pipelines. Everything that decides *whether you are allowed to edit* runs the parameter through ENSIP-15 `normalize()`. Everything that builds the resolver transaction uses the raw parameter unchanged.

For any label containing a code point that ENSIP-15 maps away or folds — a fullwidth character, a soft hyphen, mixed case — those two pipelines resolve two different nodes from one URL. The page confirms the victim owns the name and renders the Edit Profile button, and the transaction the victim then signs writes records to a node the victim does not own, on a resolver contract the attacker chose.

The route itself imposes no constraint that would prevent this. `apps/manager/src/routes/$name/route.tsx:5-13` is the only gate:

```ts
parse: ({ name }) => {
  if (name.length < 3 || !name.includes('.')) {
    throw new Error('Invalid ENS name')
  }
  return { name }
}
```

## Where the two pipelines diverge

**Path A — authorization. Normalized.**

`apps/manager/src/features/profile/service/profileOwner.ts:25`:

```ts
export const getOwner = ResultFn(async function* (params: { name: string }) {
  const ethName = normalizeEthName(params.name)
```

`normalizeEthName` is viem's ENSIP-15 `normalize` (`apps/manager/src/features/profile/service/profileName.ts:14-21`). The same normalization is applied by `profileExpiry.ts:77` and `profileRegistration.ts:66`, so the expiry card and the registration redirect also resolve the victim's real name.

The Edit Profile button is gated on the result — `apps/manager/src/features/profile/components/view/ProfileEditAction.tsx:35`:

```ts
if (!isOwner || isInGrace || isUnmigratedEthName) return null
```

**Path B — the transaction. Raw.**

`apps/manager/src/features/profile/service/profileRecordTransactions.ts:399-402`:

```ts
const ensParams: Parameters<typeof setRecordsWriteParameters>[1] = {
  name,
  resolverAddress,
}
```

`name` here is the raw route parameter, threaded down from `EditProfileDialog.machine.ts:183-193`. There is no `normalize` call anywhere in that file. `setRecordsWriteParameters` hashes it directly, producing `setText(namehash(raw), …)` and `setAddr(namehash(raw), …)` inside a `multicall` sent to `resolverAddress` — the resolver of the raw name.

So `owner(normalize(raw))` decides whether the button appears, and `namehash(raw)` decides which node is written.

## The two nodes, computed

Run against the repository's pinned `viem@2.52.2`:

```
raw route param   : "Ｎａｍｅ.eth"
  normalize()     : "name.eth"            <- what AUTHORIZATION asks about
  namehash(raw)   : 0xaa6bc74ea2ef0886b4a21e0965d0aaa436f03511263c0e18fbd681d30f1a94c8   <- what CALLDATA writes
  namehash(norm)  : 0x95419af60496b899715eddd4a23fed7213b7c9606bb9f1bfbde17dfd6a148f02   <- the victim's real node
  SAME NODE?      : false

raw route param   : "na­me.eth"      (soft hyphen)
  normalize()     : "name.eth"
  namehash(raw)   : 0xa286c26c941fb965fcbbaabecd3107d0481ee9cd2a614355c628989c8a797968
  namehash(norm)  : 0x95419af60496b899715eddd4a23fed7213b7c9606bb9f1bfbde17dfd6a148f02
  SAME NODE?      : false

raw route param   : "MyName.eth"
  normalize()     : "myname.eth"
  namehash(raw)   : 0xb68b5ab74e596450be61730f703e57a96e4c4d3f9445cca58f0a9c0ee4f9f26e
  namehash(norm)  : 0x6cbc8d00d20a89e588f430e62b937a6402557bf0bc2127fb1378457331aa463d
  SAME NODE?      : false
```

## The attacker can mint the twin through the Manager's own registration flow

This is the part that makes the attack cheap and self-contained. `/register/$name` validates with `parseName`, which does only `name.trim().toLowerCase()` (`apps/manager/src/features/register-v2/utils/name-parser.ts:46`) plus a fixed punctuation blocklist at `:11`. There is no `normalize()` anywhere in `features/register-v2`.

`toLowerCase()` does not fold fullwidth characters to ASCII:

```
"Ｎａｍｅ.eth"  parseName fold -> "ｎａｍｅ.eth"   (registers a distinct node)
                normalize()    -> "name.eth"     (the victim's node)
```

So the attacker registers the twin through the ENS Manager UI itself, with no special tooling, and the two forms remain distinct on chain because `labelToCanonicalId` is `labelhash(label)` with the low 32 bits cleared and applies no normalization.

## Attack path

The attacker needs one ordinary registration, control of that name's resolver, and a link. No role, no privileged address, no access to the victim's device.

1. The victim owns `name.eth`. The attacker registers the fullwidth twin `Ｎａｍｅ.eth` through the Manager's own register flow.
2. The attacker points the twin's resolver at a contract they control, and mirrors the victim's real text records onto it so the profile page looks authentic.
3. The attacker sends the victim a link to `/Ｎａｍｅ.eth`. Rendered, that is visually the victim's own name.
4. `profileOwnerQuery` normalizes to `name.eth`, finds the **victim** as owner, sets `isOwner = true`, and the Edit Profile button renders. `profileExpiryQuery` and `profileRegistrationQuery` normalize too, so there is no fallback card and no redirect to `/register`.
5. `getRecords` resolves the **twin's** resolver, because `resolveNameData` DNS-encodes the raw name. The page shows the attacker's mirrored records.
6. `resolverWriteAccessQuery` dry-runs `setAddr` against the attacker's contract (`resolverWriteAccess.ts:46-54`). The attacker's contract returns success, so `needsResolverSetup` is false and the flow takes the in-place write path.
7. The victim edits a field — most naturally their ETH address, for example after rotating away from a compromised wallet — and saves.

The transaction the victim signs is:

```
type: 'eoa'
to:   <attacker-controlled resolver contract>
data: multicall([ setText(namehash("Ｎａｍｅ.eth"), …),
                  setAddr(namehash("Ｎａｍｅ.eth"), 60, …) ])
value: 0n
```

described in the UI as "Update profile records for Ｎａｍｅ.eth".

Both the `to` address and the node argument are attacker-chosen, on a page that authorized and displayed the victim's own name.

## Impact

This is the wallet-argument impact directly, including its "substituting contract addresses" clause: the application constructs a transaction whose target contract and whose node argument are both different from the name it authorized and displayed. The program overview names this class — "anything that lets a transaction be built, signed, or attributed with the wrong chain, sender, target address, or arguments."

The realized consequence is that the victim's record edit silently does not take effect. They see a success state, and `name.eth` continues resolving to whatever it resolved to before. In the wallet-rotation case that means funds sent to their name keep going to the address they believe they have just replaced, indefinitely, with no indication anything failed.

I want to be straight about the boundary of that claim, which is set out under Scope correction below.

## Why this is not an already-known issue

The published known issues include an entry for name validators accepting homoglyphs, bidirectional controls and other deceptive characters, rated Medium, whose stated impact is "display confusion in our UI rather than incorrect resolution."

That entry establishes only that such names can exist. This finding is not about a validator. It is the asymmetric handling of one name inside one flow — normalized on the authorization read, raw on the calldata — which produces an incorrectly targeted signed transaction. The known entry expressly disclaims incorrect resolution as its consequence.

It shares its direction with report #89456 (authorize normalized, encode raw), and I am volunteering that rather than leaving it to be found. The differences are that #89456 is the renewal route, its divergence is between two client-side parsers (`parseName` against `normalizeEth2LdName`), its asset is the renewal fee, and its calldata target is the fixed registrar. This is the profile record-write path, its divergence is between `normalizeEthName` and no normalizer at all, and its calldata carries both an attacker-chosen `to` address and an attacker-chosen node. Different files, different builder, different asset.

It is not the transfer seam of #89960 — different app, different asset, no `getLabel`/`ens_normalize` on this path — and it is not in the role/revoke family.

## Not intended behaviour

The authors normalize deliberately elsewhere and know the route parameter arrives unnormalized:

* `profileName.test.ts:5-11` asserts `normalizeEthName('Sub.FOO.ETH')` yields `'sub.foo.eth'`.
* `setPrimaryName.test.ts:245` is named "normalizes the claimed name before encoding it" and asserts input `'LeOn'` encodes `'leon.eth'` — the requirement is stated explicitly for one write leg.

`profileRecordTransactions.ts` has no test file, and no comment or document anywhere claims the raw name is intentional on this path.

## Scope correction

The signed transaction moves no asset. `value` is zero and the call is a resolver multicall, so nothing is transferred out of the victim's wallet in that transaction. I am not claiming theft.

What the victim loses is the effect of the edit they believed they made, on a page that told them they owned the name and were authorized to change it. Whether that reaches Critical or sits below it is the program's call, and I would rather state the limit than argue past it.

One further note in the same spirit: `changeResolver.ts:6` strips `.eth` with an unanchored `replace('.eth', '')`, but its only caller is gated on `parseInput(fullName).is2LD` and a 2LD label cannot contain a dot, so that line is not independently exploitable. I mention it only as corroboration that the raw name is threaded through this feature unexamined.

## Recommended fix

Normalize once at the boundary and use that value everywhere downstream.

In `routes/$name/route.tsx`, run `normalize()` inside `params.parse` and redirect to the canonical form when it differs, so a non-canonical URL can never reach the feature. Independently, make `saveRecords`, `buildRecordsUpdateCalls` and `buildSetResolverCall` accept an already-normalized name, or have them call `normalizeEthName` themselves and throw when it returns null — so a future caller cannot reintroduce the split.

More generally, any place a name is used both to authorize and to encode should derive both from one string.

## Proof of Concept

**Revision note.** The competition Resources tab names the audited revision as `63772fd872af472ced58b009499355f3430c2a86`. This repository's `audit-comp-ready` branch imports that revision at commit `cda79ac`, and its head is `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, which the scope page's asset links point at. `git diff cda79ac 1c9b47f` touches only `apps/manager/package.json`, `workers/api-worker/package.json`, `packages/dqa-server/package.json` and `pnpm-lock.yaml` — a Hono version bump. No source file in any in-scope asset differs, so every line number below is identical at either revision.

Tested against the competition repository at `audit-comp-ready` HEAD `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

The test imports three things from production source: `normalizeEthName` (the function the authorization read uses), `parseName` (the function the register flow validates with), and viem's `namehash`. Only the route gate is transcribed, and it is four lines, quoted with its line numbers.

It needs no chain, no wallet, no RPC and no local infrastructure beyond the repository's own dependencies. The whole test is below, so nothing outside this report is needed to reproduce it.

Save as:

```
apps/manager/src/features/profile/poc-profile-record-node-split.test.ts
```

Run from the repository root:

```
pnpm install --frozen-lockfile
pnpm --filter manager exec vitest run src/features/profile/poc-profile-record-node-split.test.ts --reporter=verbose
```

Observed result on the audited revision:

```
✓ the crafted URLs pass the route gate unchanged
✓ AUTHORIZATION normalizes the crafted name onto the victim name
✓ CALLDATA hashes the raw param, which is a different node
✓ the Manager's OWN register flow accepts the twin, so the attacker can mint it in-app
✓ control: a canonical name keeps authorization and calldata on one node

Test Files  1 passed (1)
Tests       5 passed (5)
```

Full exact test:

```ts
import { describe, expect, it } from 'vitest'
import { namehash } from 'viem/ens'
import { normalizeEthName } from '@/features/profile/service/profileName'
import { parseName } from '@/features/register-v2/utils/name-parser'

// The Manager authorizes profile editing on the ENSIP-15 normalized name
// (profileOwner.ts:25 -> normalizeEthName) but encodes the RAW route param into
// the resolver calldata (profileRecordTransactions.ts:399-402 -> ensParams.name).
//
// Both helpers below are production source.

// apps/manager/src/routes/$name/route.tsx:5-13 — the only route gate.
const routeGatePasses = (name: string) => name.length >= 3 && name.includes('.')

const VICTIM = 'name.eth'
const FULLWIDTH_TWIN = 'Ｎａｍｅ.eth' // Ｎａｍｅ.eth
const SOFT_HYPHEN_TWIN = 'na­me.eth'

describe('profile record write targets a different node than it authorizes', () => {
  it('the crafted URLs pass the route gate unchanged', () => {
    expect(routeGatePasses(FULLWIDTH_TWIN)).toBe(true)
    expect(routeGatePasses(SOFT_HYPHEN_TWIN)).toBe(true)
  })

  it('AUTHORIZATION normalizes the crafted name onto the victim name', () => {
    expect(normalizeEthName(FULLWIDTH_TWIN)?.name).toBe(VICTIM)
    expect(normalizeEthName(SOFT_HYPHEN_TWIN)?.name).toBe(VICTIM)
  })

  it('CALLDATA hashes the raw param, which is a different node', () => {
    for (const twin of [FULLWIDTH_TWIN, SOFT_HYPHEN_TWIN]) {
      expect(namehash(twin)).not.toBe(namehash(VICTIM))
    }
    expect(namehash(FULLWIDTH_TWIN)).toBe(
      '0xaa6bc74ea2ef0886b4a21e0965d0aaa436f03511263c0e18fbd681d30f1a94c8',
    )
    expect(namehash(VICTIM)).toBe(
      '0x95419af60496b899715eddd4a23fed7213b7c9606bb9f1bfbde17dfd6a148f02',
    )
  })

  it("the Manager's OWN register flow accepts the twin, so the attacker can mint it in-app", () => {
    const parsed = parseName(FULLWIDTH_TWIN)
    expect(parsed.isOk()).toBe(true)
    // trim().toLowerCase() does not fold fullwidth to ASCII, so the registered
    // label stays distinct from the victim's.
    expect(`${parsed._unsafeUnwrap().label}.eth`).not.toBe(VICTIM)
    expect(normalizeEthName(FULLWIDTH_TWIN)?.name).toBe(VICTIM)
  })

  it('control: a canonical name keeps authorization and calldata on one node', () => {
    expect(normalizeEthName(VICTIM)?.name).toBe(VICTIM)
    expect(namehash(VICTIM)).toBe(namehash(normalizeEthName(VICTIM)!.name))
  })
})
```

Steps to reproduce in the application

1. Victim owns `name.eth`.
2. Attacker opens the Manager's own register flow at `/register/Ｎａｍｅ.eth` and registers it. `parseName` folds only with `trim().toLowerCase()`, which does not convert fullwidth characters to ASCII, so this is a distinct registration.
3. Attacker sets that name's resolver to a contract they control, and writes text records mirroring the victim's real profile so the page looks authentic.
4. Attacker sends the victim a link to `/Ｎａｍｅ.eth`.
5. Victim opens it while connected. The page shows their name and profile, and the "Edit Profile" button is present, because `profileOwnerQuery` normalized the URL to `name.eth` and found the victim as owner.
6. Victim changes their ETH address record and saves.
7. Inspect the transaction before signing: `to` is the attacker's resolver contract, and every inner call carries `namehash("Ｎａｍｅ.eth")` = `0xaa6bc74e…94c8`, not the victim's `namehash("name.eth")` = `0x95419af6…8f02`.
8. After the transaction succeeds, read `addr(namehash("name.eth"))` on the victim's real resolver. It is unchanged. The victim's address record was never written.
