# 92128 w a critical inconsistent ens name normalization in the manager profile page renders a trusted name s owner beside an attacker controlled receiving address leading to theft of user funds

**Submitted on Sep 7th 2026 at 11:58:41 UTC by @JollyCat68056 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92128
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager profile route (`/$name`, and `/p/$name` which forwards to it) resolves the **owner, primary name, registration and expiry** from the **ENSIP-15-normalized** name, but resolves the **address records** from the **raw, un-normalized** URL string. Because `normalize("Vitalik.eth") === "vitalik.eth"` while `namehash("Vitalik.eth") !== namehash("vitalik.eth")`, these are two different ENS nodes, and the page renders a composite of both. An attacker who registers the un-normalized twin of a well-known name and points its `addr(60)` at themselves produces a profile page on which every trust signal — the owner address, the owner's forward-verified primary name, and the real registration/expiry dates — belongs to the genuine name, while the value presented under the heading "Main receiving address" is the attacker's. In production this leads to direct loss of user funds: a victim who opens the shared link and copies the displayed receiving address sends funds to the attacker.

## Vulnerability Details

The route accepts the name from the URL without normalizing it, then fans that same raw string into two data services that normalize differently.

**1. The route guard performs no normalization** — `apps/manager/src/routes/$name/route.tsx:5-13`

```ts
parse: ({ name }) => {
  if (name.length < 3 || !name.includes('.')) {
    throw new Error('Invalid ENS name')
  }
  return { name }   // raw string kept as-is: no case fold, no ENSIP-15, no character check
}
```

`apps/manager/src/routes/p/$name/index.tsx:9` forwards the parameter to `/$name` unchanged, and `/p/<name>` is the app's own profile-sharing format.

**2. The loader passes the same raw string to both services** — `apps/manager/src/routes/$name/index.tsx:51-56`

```ts
const [profileRecords, ownerData] = await Promise.all([
  queryClient.ensureQueryData(profileRecordsQuery(name)),   // RAW name
  queryClient.fetchQuery(profileOwnerQuery(name)),          // normalized internally
])
```

The loader's only two `redirect()` calls (`:82`, `:111`) both target `/register/$name`. There is **no redirect to a canonical or normalized name**, so the raw string survives for the lifetime of the page — it is even used for the document title (`head:` at `:131`, `title:` at `:137`).

**3. Owner, expiry and registration are ENSIP-15 normalized** — `apps/manager/src/features/profile/service/profileOwner.ts:24-25`

```ts
export const getOwner = ResultFn(async function* (params: { name: string }) {
  const ethName = normalizeEthName(params.name)      // 'Vitalik.eth' -> 'vitalik.eth'
  ...
  const v2Owner = yield* fromPromise(ensjsv2_getOwner(client, { name: ethName.name }), ...)
```

`normalizeEthName` is viem's ENSIP-15 `normalize` — `apps/manager/src/features/profile/service/profileName.ts:1` (`import { normalize } from 'viem/ens'`), definition at `:14`, call at `:18`. `profileExpiry.ts` and `profileRegistration.ts` normalize the same way. `ProfileView.tsx:117` then resolves `profileReverseNameQuery(owner)`, which uses viem `getEnsName` (UniversalResolver, forward-verified), so the Owner row displays the **genuine owner's verified primary name**.

**4. Text and address records are read from the RAW name** — `apps/manager/src/features/profile/service/profileRecords.ts:82,123`

```ts
export const getProfileRecords = ResultFn(async function* (name: string) {   // raw route param
  ...
  const records = yield* fromPromise(
    getRecords(client, { name, texts, coins, contentHash: true, abi: true, ignoreInvalidCoinTypes: true }),
```

ensjs encodes the name exactly as given. In the pinned build (`@ensdomains/ensjs@5.0.0-alpha.1`, the `pkg.pr.new` PR-230 build this repo uses), `dist/actions/public/getRecords.js` passes `name` straight into `getTextParameters` / `getAddressParameters` / `getContentHashParameters`; the only read action in that package that imports `normalize` is `getName.js` (reverse resolution). `alwaysProbeAddressRecords = ['60']` guarantees `addr(60)` is always queried, on the **raw** node.

**5. Both results are rendered on the same page** — `apps/manager/src/features/profile/components/view/ProfileView.tsx:94,100,117`

* **Owner row** — `ProfileDetails.tsx:115-121`: `value={ownerReverseName || truncateAddress(owner)}` with `copyValue={owner}` — the genuine owner's primary name and address.
* **"Main receiving address"** — `ProfileAddressesSection.tsx:171` renders that heading, fed by `getMainReceivingAddress(records)` (`ProfileView.helpers.ts:239`), which selects `coinType === ETH_COIN_TYPE` (60) at `:234` with **no verification that the record belongs to the same node as the owner**, and renders it inside a `CopyableButton value={address.value}`.

**Root-cause arithmetic** (verified against viem 2.52.2, the version this repo pins):

| Input                                                    | `normalize(input)` | Same ENS node? |
| -------------------------------------------------------- | ------------------ | -------------- |
| `Vitalik.eth`                                            | `vitalik.eth`      | **NO**         |
| `VITALIK.eth`                                            | `vitalik.eth`      | **NO**         |
| `vitalik<U+00AD>.eth` (soft hyphen — renders as nothing) | `vitalik.eth`      | **NO**         |
| `vitalik<U+FE0F>.eth` (VS16 — renders as nothing)        | `vitalik.eth`      | **NO**         |
| `nick.eth` *(control)*                                   | `nick.eth`         | yes            |

"Same ENS node?" is `namehash(input) === namehash(normalize(input))`. Every **NO** row is an exploitable divergence: the node whose **owner** is displayed and the node whose **receiving address** is displayed are different names.

**Scope note.** The Explorer app (`apps/portal`) is **not** affected, and the contrast confirms this is an application defect rather than user error: the Explorer's profile path contains no `normalize` call at all — `getEnsOwner` (`features/profile/hooks/useEnsOwner.ts` -> `utils/ens/resolveEnsOwner.ts`) and `getProfile` both resolve the **same raw name**, so it coherently displays the attacker's own name and never composes two nodes. Only the Manager mixes the two normalizations.

## Impact Details

**Selected impact (Critical):** "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient — resolving or rendering an ENS name to an attacker-controlled address."

This finding is that impact exactly. The Manager renders a real, trusted ENS name to an attacker-controlled address.

**Loss.** Unbounded, and limited only by what victims send. The value presented under "Main receiving address" is the address the app tells a user to pay, offered with a one-click Copy button. Any funds a victim sends after reading that page go to the attacker. The natural targets are names with a public donation or payment reputation, where the page is shared precisely so strangers can pay it.

**Why the usual defence fails.** The victim's instinct is to check that the profile really is the name they intend to pay. Here that check *passes*: the Owner row shows the genuine owner's forward-verified primary name, and Registered/Expires show the genuine dates, because all of those are read from the normalized node. The one field that is attacker-controlled is the one the victim acts on. The deception is corroborated by the application's own on-chain reads, so careful users are given a false confirmation rather than a warning.

**Preconditions.** One ENS name registration plus gas. No privileged access, no leaked keys, no flash loan, no capital at risk, and no unusual in-app action by the victim — the victim only opens a profile link, which is the app's own sharing mechanism (`/p/<name>`). Using the invisible-character variant (U+00AD or U+FE0F), the rendered name is visually identical to the genuine one, so there is no visual cue at all.

**Not a duplicate of EXP-INPUT-005 (Medium).** That known issue states: "Our name *validators* accept inputs that ENS normalization later rejects or transforms, including homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot. The practical impact is homograph-style display confusion in our UI **rather than incorrect resolution**." This report differs on every axis:

|             | EXP-INPUT-005                                              | This report                                                                                                                                       |
| ----------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Root cause  | a permissive **validator**                                 | **inconsistent normalization between two data services** on one page — no validator is on the path; the `/$name` guard is a length/dot check only |
| Trigger     | homoglyphs, bidi, control characters, leading/trailing dot | **plain ASCII case** (`Vitalik`) — not a deceptive character under any reading of that entry                                                      |
| Consequence | "display confusion … rather than incorrect resolution"     | **incorrect resolution** — precisely the consequence that entry disclaims                                                                         |
| Severity    | Medium                                                     | **Critical**                                                                                                                                      |

A validator that merely rejected deceptive characters would not describe this defect: the bug is that one page applies **two different normalizations** to the same input. Additionally, the program's rules keep "new consequences of a listed root cause that materially change its severity" in scope, and Medium -> Critical is material. It is also not R2-02 (no injection, no CSP dependency), not WEB-1301 (address-view metadata), and not QA-06.

**Recommended fix.** Normalize once at the boundary: in `routes/$name/route.tsx`'s `parse`, run viem `normalize(name)` and redirect to the canonical name when it differs (or reject the non-normalized name). Every profile service must then key off that single normalized value, so the owner and the records can never describe different ENS nodes. A regression test should assert that `getRecords` and `getOwner` are always called with the same string.

## References

Audited revision: `63772fd872af472ced58b009499355f3430c2a86`

* `apps/manager/src/routes/$name/route.tsx:5-13` — route guard, no normalization
* `apps/manager/src/routes/p/$name/index.tsx:9` — forwards the raw parameter
* `apps/manager/src/routes/$name/index.tsx:51-56` — both services receive the raw name; `:82`,`:111` the only redirects; `:131`,`:137` raw name in the title
* `apps/manager/src/features/profile/service/profileOwner.ts:24-25` — owner lookup normalizes
* `apps/manager/src/features/profile/service/profileName.ts:1,14,18` — viem ENSIP-15 `normalize`
* `apps/manager/src/features/profile/service/profileRecords.ts:82,123` — records lookup uses the raw name
* `apps/manager/src/features/profile/components/view/ProfileView.tsx:94,100,117` — both queries plus reverse-name resolution
* `apps/manager/src/features/profile/components/view/ProfileDetails.tsx:115-121` — Owner row
* `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:171` — "Main receiving address" heading
* `apps/manager/src/features/profile/components/view/ProfileView.helpers.ts:234,239` — selects `coinType 60` with no owner check
* `@ensdomains/ensjs@5.0.0-alpha.1` `dist/actions/public/getRecords.js` — encodes the name as given; only `getName.js` normalizes
* ENSIP-15 (ENS Name Normalization): <https://docs.ens.domains/ensip/15>

## Link to Proof of Concept

<https://gist.github.com/fulldev1123/422756b71332bc0ca9e429f38d4e15c2>

## Proof of Concept

## Step-by-step proof of concept

### Part A — Demonstrate the root cause (offline, no network, \~1 minute)

1. Check out the audited revision `63772fd872af472ced58b009499355f3430c2a86` and run `pnpm install --frozen-lockfile` (Node 22, pnpm 10).
2. Create `apps/manager/poc-normalize.mjs` with:

```js
import { normalize } from 'viem/ens'
import { namehash } from 'viem'
for (const raw of ['Vitalik.eth', 'VITALIK.eth', 'vitalik­.eth', 'vitalik️.eth', 'nick.eth']) {
  const norm = normalize(raw)
  console.log(JSON.stringify(raw), '-> normalize =', JSON.stringify(norm),
              '| same node?', namehash(raw) === namehash(norm))
}
```

3. Run `cd apps/manager && node poc-normalize.mjs`. Observed output:

```
"Vitalik.eth"     -> normalize = "vitalik.eth"  | same node? false
"VITALIK.eth"     -> normalize = "vitalik.eth"  | same node? false
"vitalik­.eth" -> normalize = "vitalik.eth"  | same node? false
"vitalik️.eth" -> normalize = "vitalik.eth"  | same node? false
"nick.eth"        -> normalize = "nick.eth"     | same node? true
```

This establishes the premise: for these inputs the normalized name and the raw name are **different ENS nodes**.

### Part B — Demonstrate that the application uses both (offline, no network, \~1 minute)

4. Create `apps/manager/src/features/profile/service/poc-normalization-mismatch.test.ts`. The test calls the application's **real** `getProfileRecords()` and `getOwner()`, mocking only the wagmi client and the ensjs transport seams so that no network request is made, and records which name string each one passes downstream:

```ts
import { ok } from 'neverthrow'
import { describe, expect, it, vi } from 'vitest'

const mocks = vi.hoisted(() => ({ client: {}, getRecords: vi.fn(), getOwnerV2: vi.fn(), getOwnerV1: vi.fn() }))
vi.mock('@/lib/wagmi/helpers', () => ({ safeGetClient: () => ok(mocks.client) }))
vi.mock('@ensdomains/ensjs/public', async (o) => ({ ...(await o()), getRecords: mocks.getRecords }))
vi.mock('@ensdomains/ensjs/public/v2', async (o) => ({ ...(await o()), getOwner: mocks.getOwnerV2 }))
vi.mock('@ensdomains/ensjs/public/v1', async (o) => ({ ...(await o()), getOwner: mocks.getOwnerV1 }))
vi.mock('@ensdomains/ensjs/utils', async (o) => ({ ...(await o()), getCoderFromCoin: () => ({ name: 'eth', coinType: 60 }) }))
vi.mock('@ens-apps/indexer/urql', () => ({ default: { query: () => ({ toPromise: async () => ({ data: { domain: null } }) }) } }))
vi.mock('@ens-apps/indexer', () => ({ DomainDocument: {} }))

import { getProfileRecords } from './profileRecords'
import { getOwner } from './profileOwner'

const ATTACKER = '0x000000000000000000000000000000000000dEaD'
const REAL_OWNER = '0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045'

describe('profile page composes two different ENS nodes', () => {
  it('records use the RAW name while owner uses the NORMALIZED name', async () => {
    mocks.getRecords.mockResolvedValue({
      texts: [], coins: [{ coinType: 60, value: ATTACKER }],
      resolverAddress: '0x0000000000000000000000000000000000000abc', contentHash: null, abi: null,
    })
    mocks.getOwnerV2.mockResolvedValue(REAL_OWNER)

    await getProfileRecords('Vitalik.eth')
    await getOwner({ name: 'Vitalik.eth' })

    expect(mocks.getRecords).toHaveBeenCalledWith(mocks.client, expect.objectContaining({ name: 'Vitalik.eth' }))
    expect(mocks.getOwnerV2).toHaveBeenCalledWith(mocks.client, expect.objectContaining({ name: 'vitalik.eth' }))
  })
})
```

5. Run:

```
pnpm --filter manager exec vitest run src/features/profile/service/poc-normalization-mismatch.test.ts
```

Observed result:

```
 ✓ src/features/profile/service/poc-normalization-mismatch.test.ts (1 test)
 Test Files  1 passed (1)
      Tests  1 passed (1)
```

Both assertions pass, proving that for the single input `"Vitalik.eth"` the application queries records for `"Vitalik.eth"` and the owner for `"vitalik.eth"` — two different nodes, whose results are then rendered on one page.

### Part C — End-to-end attack, step by step

Preconditions: the attacker owns the un-normalized twin label; the victim opens one link. No privileged access is needed.

6. **Attacker registers the twin.** Call the `ETHRegistrarController` commit/register directly for the label `Vitalik` (or, for the invisible variant, `vitalik` + U+00AD). `valid()` enforces only `strlen >= 3` and does not apply ENSIP-15, and `labelhash("Vitalik") != labelhash("vitalik")`, so it is a distinct, unregistered name. (For the invisible variant no direct contract call is even required — `/register/$name` uses `parseName` (`features/register-v2/utils/name-parser.ts:46`), whose only transform is `trim().toLowerCase()` plus a character blacklist containing no invisible characters, so U+00AD/U+FE0F survive and are registered verbatim.)
7. **Attacker sets the payout address.** Set a resolver on the twin and `setAddr(namehash("Vitalik.eth"), 60, ATTACKER_ADDRESS)`. Optionally copy the genuine name's `avatar` and `description` text records into the same resolver so the page is visually identical.
8. **Attacker shares the profile link** `https://app.ens.domains/Vitalik.eth` — or `/p/Vitalik.eth`, the app's own share format, which forwards unchanged — for example as "donate to vitalik.eth here".
9. **Victim opens the link.** The page renders:
   * name badge: `Vitalik.eth` (with the U+00AD variant, indistinguishable from `vitalik.eth`);
   * **Owner**: `vitalik.eth` — the genuine owner's forward-verified primary name, with the genuine owner address behind the copy icon;
   * **Registered / Expires**: the genuine name's dates;
   * **Main receiving address**: `ATTACKER_ADDRESS`, with a one-click Copy button.
10. **Victim copies the "Main receiving address" and sends funds.** The funds go to the attacker. The victim's verification of the profile succeeded, because every field they could check was read from the genuine node.

### Optional — full UI reproduction on a local fork

11. Start the repository's e2e Anvil Sepolia-fork stack (`pnpm e2e:infra:up`), register `vitalik` to one account and the twin label to another, set the twin's `addr(60)`, run `pnpm dev:manager`, and open `http://localhost:3000/Vitalik.eth`. The Owner row shows the genuine owner while "Main receiving address" shows the attacker address.
