# 89930 w a critical manager profile displays a trusted name s owner but an attacker controlled receiving address owner node vs records node normalization split&#x20;

**Submitted on Aug 21st 2026 at 22:47:29 UTC by @shakreiner for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89930
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The ENS Manager app renders a name's profile by resolving the **owner/expiry from the normalized name** but the **records — including the "Main receiving address" — from the raw, un-normalized URL name**. Because `namehash` does not normalize and ENS keys labels on-chain by `keccak256(labelBytes)` (no on-chain normalization), `Vitalik.eth` and `vitalik.eth` are distinct, separately-ownable nodes. An attacker who registers the literal label `Vitalik` and sets an ETH address record on it can share the link `/Vitalik.eth`: in production the page shows the **real `vitalik.eth` owner, expiry, and avatar** (every trust cue valid) while the copyable "Main receiving address" is the **attacker's** wallet — causing a victim to send funds directly to the attacker.

## Vulnerability Details

The `$name` profile loader passes the **raw** URL name to both the owner query and the records query, and never normalizes or redirects to a canonical URL:

```ts
// apps/manager/src/routes/$name/index.tsx:50-56
loader: async ({ params: { name }, context: { queryClient } }) => {
  const [profileRecords, ownerData] = await Promise.all([
    queryClient.ensureQueryData(profileRecordsQuery(name)), // records ← RAW name
    queryClient.fetchQuery(profileOwnerQuery(name)),        // owner   ← RAW name
  ])
```

There is no `beforeLoad`, no `normalize`, and no redirect-to-canonical. `parseName` only lowercases for TLD classification, so `Vitalik.eth` is accepted as a normal `.eth` name and, because the normalized-node owner is non-null, the full `ProfileView` renders (no fallback card).

**Owner path — normalizes the name** (points at the legitimate node):

```ts
// apps/manager/src/features/profile/service/profileOwner.ts:25,56
const ethName = normalizeEthName(params.name)        // -> "vitalik.eth"
...
ensjsv2_getOwner(client, { name: ethName.name })     // resolves namehash("vitalik.eth")
// normalizeEthName -> profileName.ts:18  normalize(name)  (viem/ens, UTS-46/ENSIP-15)
```

**Records path — does NOT normalize** (points at a different, attacker-controllable node). It hands the raw name to `getRecords`, which `namehash`es it as-is — per viem's own docs, *"This function does not normalize the name. You must normalize the name before passing it."* The ETH address (coinType 60) is always probed via `alwaysProbeAddressRecords`, regardless of the indexer:

```ts
// apps/manager/src/features/profile/service/profileRecords.ts:116-131
const coins = unique([...alwaysProbeAddressRecords.map(Number), ...]) // always includes coinType 60
...
getRecords(client, { name, texts, coins, contentHash: true, abi: true }) // RAW name -> namehash(RAW)
```

Both queries receive the same raw `name` prop in the view, and the resulting address is rendered as a full, copyable value next to the (authentic) name and avatar:

```ts
// apps/manager/src/features/profile/components/view/ProfileView.tsx:93,99
profileRecordsQuery(name)   // raw-node records  → attacker addr
profileOwnerQuery(name)     // normalized-node owner → REAL owner (trust cue)

// apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:94
<CopyableButton value={address.value}>  // FULL address copied to clipboard (display truncated at :42/:119)
```

Because `normalize()` folds case and compatibility characters while `namehash` does not, the two paths resolve **different nodes for the same URL**:

| URL string                   | `normalize()` | Owner node (queried)      | Records node (queried)    | Same node? |
| ---------------------------- | ------------- | ------------------------- | ------------------------- | ---------- |
| `Vitalik.eth`                | `vitalik.eth` | `namehash("vitalik.eth")` | `namehash("Vitalik.eth")` | **No**     |
| `VITALIK.eth`                | `vitalik.eth` | `namehash("vitalik.eth")` | `namehash("VITALIK.eth")` | **No**     |
| `ﬁxie.eth` (U+FB01 ligature) | `fixie.eth`   | `namehash("fixie.eth")`   | `namehash("ﬁxie.eth")`    | **No**     |

An attacker can register the literal label (e.g. `Vitalik`) because ENS on-chain registration enforces only `valid()` (label length ≥ 3) and availability — **not** normalization — so the non-normalized label is available and, once registered, gives the attacker ownership of the records node while the owner node still belongs to the real name. Setting the attacker's address as the coinType‑60 record on that node is then reflected verbatim in the profile's "Main receiving address" card.

## Impact Details

This is a direct **loss of user funds**. The victim is shown a profile page on which **every trust signal is genuine** — correct owner address, correct expiry, correct name header and avatar for the well-known name — while the single field that governs a payment (the "Main receiving address") is attacker-controlled. A user who copies that address (the clipboard receives the **full** attacker address, not the truncated display value) and sends ETH/tokens loses the entire amount sent, with funds going irrecoverably to the attacker.

* **Assets at risk:** any funds a victim sends after reading/copying the receiving address off a spoofed-but-authentic-looking profile — unbounded per victim.
* **Attacker prerequisites:** register one non-normalized label and set one address record (one-time, low cost); then distribute a normal-looking link. No privileged position, no victim wallet connection required, no interaction with the victim's account.
* **Why the trust cues make it severe:** careful users specifically verify owner/expiry/name before paying; here all of those validate against the real name, so the usual defenses do not catch the substitution.

**In-scope impact this maps to (verbatim from the program):** *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient — e.g. resolving or rendering an ENS name to an attacker-controlled address; showing the wrong recipient or amount on a transaction the user then approves."*

## References

* Loader (raw name to both queries): `apps/manager/src/routes/$name/index.tsx#L50-L56`
* Owner path (normalizes): `apps/manager/src/features/profile/service/profileOwner.ts#L25` and `#L56`; `apps/manager/src/features/profile/service/profileName.ts#L18`
* Records path (does not normalize): `apps/manager/src/features/profile/service/profileRecords.ts#L116-L131`
* View wiring: `apps/manager/src/features/profile/components/view/ProfileView.tsx#L93` and `#L99`
* Copyable receiving-address sink: `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx#L94`
* viem `namehash` (no normalization) and `normalize` (ENSIP-15/UTS-46): <https://viem.sh/docs/ens/utilities/namehash> / <https://viem.sh/docs/ens/utilities/normalize>
* ENSIP-15 (ENS Name Normalization Standard): <https://docs.ens.domains/ensip/15>

## Proof of Concept

## Proof of Concept

The exploit has two verifiable parts: (A) a **runnable test** that proves the core defect — the owner query and the records query resolve **different ENS nodes for the same URL** — using the repository's own normalization function; and (B) the **end-to-end reproduction** showing how that split is weaponized into a wrong-address display.

### Part A — Runnable proof of the owner-node vs records-node split

Add this test file to the in-scope `apps/manager` package and run it. It imports the app's real `normalizeEthName` (the exact function the owner path uses) and viem's `namehash` (which `getRecords` uses internally, and which does not normalize), and asserts the two paths target different nodes.

**File:** `apps/manager/src/features/profile/service/name-normalization-split.poc.test.ts`

```ts
import { namehash } from 'viem'
import { describe, expect, it } from 'vitest'
import { normalizeEthName } from './profileName'

// Node the OWNER path queries: getOwner uses normalizeEthName(name).name
function ownerNode(urlName: string): `0x${string}` {
  const ethName = normalizeEthName(urlName)
  if (!ethName) throw new Error('normalize rejected the name')
  return namehash(ethName.name)
}

// Node the RECORDS path queries: getProfileRecords passes the RAW url name
// straight to ensjs getRecords, which namehashes it WITHOUT normalizing.
function recordsNode(urlName: string): `0x${string}` {
  return namehash(urlName)
}

describe('PoC: owner-node vs records-node normalization split', () => {
  const cases: Array<{ url: string; normalizesTo: string; label: string }> = [
    { url: 'Vitalik.eth', normalizesTo: 'vitalik.eth', label: 'uppercase' },
    { url: 'VITALIK.eth', normalizesTo: 'vitalik.eth', label: 'all caps' },
    // U+FB01 LATIN SMALL LIGATURE FI -> UTS-46 maps to "fi"
    { url: 'ﬁxie.eth', normalizesTo: 'fixie.eth', label: 'fi-ligature' },
  ]

  it.each(cases)(
    'URL "$url" ($label): owner resolves the trusted node, records resolve a DIFFERENT (attacker) node',
    ({ url, normalizesTo }) => {
      // Owner path collapses the URL to the trusted, normalized name.
      expect(normalizeEthName(url)?.name).toBe(normalizesTo)
      expect(ownerNode(url)).toBe(namehash(normalizesTo))

      // Records path uses the raw string -> a completely different node the
      // attacker can register/control (ENS keys labels by keccak256(labelBytes),
      // no on-chain normalization).
      expect(recordsNode(url)).not.toBe(ownerNode(url))
    },
  )

  it('the raw and normalized nodes are genuinely distinct 32-byte hashes', () => {
    const owner = ownerNode('Vitalik.eth')     // == namehash("vitalik.eth")
    const records = recordsNode('Vitalik.eth') // == namehash("Vitalik.eth")
    expect(owner).toMatch(/^0x[0-9a-f]{64}$/)
    expect(records).toMatch(/^0x[0-9a-f]{64}$/)
    expect(owner).not.toBe(records)
    expect(owner).toBe(namehash('vitalik.eth')) // owner node == the REAL vitalik.eth node
  })
})
```

**Run:**

```bash
# From a fresh clone: `git checkout audit-comp-ready && pnpm install` first, then:
cd apps/manager
pnpm exec vitest run src/features/profile/service/name-normalization-split.poc.test.ts
```

**Observed output:**

```
 Test Files  1 passed (1)
      Tests  4 passed (4)
```

This proves that for `Vitalik.eth` / `VITALIK.eth` / `ﬁxie.eth`, the owner-path node (`namehash(normalizeEthName(url).name)`) equals the genuine `namehash("vitalik.eth")`/`namehash("fixie.eth")`, while the records-path node (`namehash(rawUrl)`) is a different node. Since `getProfileRecords` (`profileRecords.ts:123`) passes the raw `name` to `getRecords` — unlike `getOwner`, which calls `normalizeEthName` first — the profile page reads its addresses from that different node.

### Part B — End-to-end reproduction (wrong-address display)

Environment: the app runs against Sepolia (`sepoliaWithEns`); reproduce on Sepolia.

**Attacker setup (one-time, on-chain):**

1. Choose a trusted target, e.g. `vitalik.eth`. Pick a non-normalized string that UTS-46-normalizes to it — `Vitalik.eth` (capital V), or the `ﬁ`-ligature variant of another target.
2. Register the **literal** label `Vitalik` via the ETHRegistrar/controller. It is available (its `keccak256("Vitalik")` labelhash differs from `keccak256("vitalik")`) and passes `valid()` (length ≥ 3). The attacker now owns node `namehash("Vitalik.eth")`.
3. Set a resolver on that node and set its ETH address record (coinType 60) to the **attacker's** address. Optionally copy `vitalik.eth`'s `avatar`/`header`/text records to visually mirror it.

**Victim reproduction:** 4. Open the Manager app at `/(app)/Vitalik.eth` (the link an attacker would distribute). 5. Observe the rendered profile: - **Header / Owner / Expiry / Avatar** — resolved from the normalized node → the **real `vitalik.eth`** data (page validates as authentic). - **"Main receiving address" card** — resolved from the raw node `namehash("Vitalik.eth")` → the **attacker's** address. 6. Click the "Main receiving address" card. The clipboard receives the **full attacker address** (`ProfileAddressesSection.tsx:94`, `value={address.value}`), even though the on-screen value is truncated. 7. Send funds to the copied address → funds go to the attacker while every trust cue on the page belonged to the legitimate name.

**Expected result:** a single profile page that displays the authentic owner/expiry/avatar of the trusted name alongside an attacker-controlled receiving address — the exact "resolving/rendering an ENS name to an attacker-controlled address; showing the wrong recipient" condition, leading to loss of any funds the victim sends.
