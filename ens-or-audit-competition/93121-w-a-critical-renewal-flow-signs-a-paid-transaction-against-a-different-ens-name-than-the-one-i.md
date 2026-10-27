# 93121 w a critical renewal flow signs a paid transaction against a different ens name than the one it resolves gates and displays

**Submitted on Sep 14th 2026 at 10:14:20 UTC by @tomX0 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93121
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief / Intro

The Manager app's renewal route derives two different ENS names from one URL parameter. The chain reads normalize the parameter under ENSIP-15. The chain write does not.

An attacker who registers a label that normalizes onto a victim's label, for instance the victim's own name with a trailing soft hyphen (U+00AD, which renders as nothing), can send that victim a link that reads as their own name. The app resolves the victim's real name, reads the victim's real expiry, decides the victim owns it, suppresses its own "you are renewing someone else's name" warning on that basis, and displays a renewal date computed from the victim's genuine on-chain expiry. The `renew()` calldata behind the button carries the attacker's label instead.

The victim approves. Their payment extends the attacker's registration by a year. Their own name is untouched. The app then reports that the renewal succeeded and shows an expiry date that is not true of any name they own.

This is measured rather than argued. On a fork, the victim's expiry moves by 0 seconds and the attacker's by exactly the 31,536,000 purchased, both read from the same registry the app itself reads.

## Vulnerability details

### Root cause: two names derived from one route parameter

`normalize` from `viem/ens` is imported in exactly two files in the whole Manager app, `profile/service/profileName.ts:1` and `profile/service/setPrimaryName.ts:28`. Every other name path goes through a homegrown parser.

The parser, at `apps/manager/src/features/register-v2/utils/name-parser.ts`:

```ts
// :11
const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/

// :46
const normalized = name.trim().toLowerCase()
```

`toLowerCase()` performs no UTS-46 mapping, no NFC composition, no label-extension check and no underscore-placement check. The filter beside it is an ASCII punctuation blocklist. The second validator, `validateENSName` at `shared/registration/nameUtils.ts:90-143`, repeats that same blocklist at line 132 and adds a three-code-point minimum at 124-130. Neither consults ENSIP-15. A label containing a NUL byte passes both.

In `apps/manager/src/routes/renew/$name.tsx`, line 20 calls `parseRenewableName(name)`, which calls `parseName` and yields the raw label. Lines 26 to 28 call `profileExpiryQuery(name)`, which reads expiry for the normalized name. Line 52 passes the raw label and the normalized expiry together into `canRenewV2Name(name, expiryDate)`. Lines 60 to 63 then return both out of the same loader:

```ts
return { label: parsedName.value.label, currentExpiry: expiryData.expiry }
```

`profileExpiry.ts:77` calls `normalizeEth2LdName(name)` and lines 104 to 110 read the registry with that normalized name, so `currentExpiry` belongs to the victim's real name. `renewableName.ts:25-36` returns `parseName`'s label untouched, so `label` belongs to the attacker's twin. `RenewalPage.tsx:22-26` hands both into the same UI context at `renewalUi.context.tsx:54`, and every screen and the state machine take whichever they need from there.

### The write path never normalizes

The first screen renders one name while pricing the other. `PricingSummaryCard.tsx:10` takes `currentExpiry` from that context, lines 17 to 22 add the selected duration via `getDurationExpiryDateForDisplay`, and lines 28, 39 and 42 render "Renewing for expiring on ". That sentence is true of the victim's name and of nothing else on the page, and it is on screen at the moment the victim decides.

The raw label is never reconciled with anything after that. `ConfirmPurchase.tsx:14` takes `label` from the renewal context, line 22 prices that raw label, and lines 63 to 72 send it into the machine as `renewal.start`. `renewalUi.machine.ts:148-159` stores it as `submissionData.label` and lines 346 to 355 pass it to `submitRenewal`. It arrives at `registration.actors.ts:1020-1031`:

```ts
function encodeRenewData(
  label: string,
  duration: bigint,
  paymentToken: Address,
): Hash {
  const cleanLabel = label.replace('.eth', '')
  return encodeFunctionData({
    abi: ethRegistrarRenewSnippet,
    functionName: 'renew',
    args: [cleanLabel, duration, paymentToken, REFERER_ADDRESS],
  })
}
```

There is no `normalize()` call anywhere on that path.

### The consent gate is evaluated against the wrong name

The Manager ships an interstitial for exactly the situation of paying to extend somebody else's registration. From `ThirdPartyRenewalDialog.tsx:64-70`:

> Renewing this name will add time to the current owner's registration. You won't gain ownership of the name, but you will make someone's day.

It is gated on one predicate. This is `renew/utils/thirdPartyRenewalWarning.ts:1-3` in full:

```ts
export const shouldShowThirdPartyRenewalWarning = (
  isOwner: boolean | undefined,
) => isOwner === false
```

`ProfileRenewAction.tsx:66-78` renders the dialog when that returns true, and lines 80 to 88 render a plain link straight into the paid flow when it does not.

`isOwner` comes from the normalizing read. `ProfileView.tsx:99-101` calls `profileOwnerQuery(name)`, and `profileOwner.ts:25` normalizes before anything else:

```ts
const ethName = normalizeEthName(params.name)
```

Lines 55 to 58 read ownership for that normalized name. `ProfileView.tsx:119` compares the result against the connected wallet to produce `isOwner`, line 152 resolves it, line 230 hands it to `ProfileActions`.

So on the twin's profile page the ownership read returns the victim's ownership of their own real name, `isOwner` is true, the predicate returns false, and the one screen whose entire job is to say "you will not gain ownership of this name" never renders. The plain link rendered in its place carries the raw route parameter straight through, since `ProfileRenewAction.tsx:83` passes `params={{ name }}`, so the twin reaches `/renew/$name` untouched. The consent gate and the money point at different names.

One limit on this argument, stated before someone else states it for me. The dialog is a profile-page gate, not a renewal-route gate. A link sent straight to `/renew/<anything>` skips it for any name at all, and I do not count that against this bug. My claim is narrower: on the profile route, which is the ordinary way a user reaches renewal and the link I would send, the gate is not skipped. It is evaluated, and answered wrongly, because the question it asks is about a different name than the one the button will renew.

### The confirmation screen asserts a renewal that did not occur

`SuccessStep.tsx:5-12` renders `RenewalCompletionBanner` and then `RenewalDetails`. The banner strings at `RenewalCompletionBanner.tsx:13` and `:16` are "Renewal Complete!" and "The ENS domain has been successfully renewed."

`RenewalDetails.tsx:50` renders the name from the raw label:

```tsx
domainName={`${submissionData.label}.eth`}
```

Lines 23 to 32 compute the date from the victim's expiry:

```ts
expirationDate = getDurationExpiryDateForDisplay(
  Number(submissionData.duration),
  new Date(Number(currentExpiry) * 1000),
)
```

Lines 105 to 108 render that as the Expires date. Line 69 uses the same `currentExpiry` as the reference date for the Renewal Period label, so that line is wrong in the same way.

The screen therefore names the attacker's label and dates it from the victim's name. In the run below, the victim's real expiry does not move at all while the screen reports an expiry a year out.

### Attack scenario

1. The attacker registers the victim's label with a soft hyphen appended. U+00AD renders as nothing, so the name is visually identical. ENSIP-15 normalization maps it back to the victim's label. `labelhash` does not.
2. The attacker sends the owner of the real name a link. Expiry dates are public, so the link can arrive in exactly the window when a renewal reminder would be plausible.
3. The loader parses the raw parameter for the label it will transact with and separately normalizes it for the expiry and ownership reads. Those reads land on the victim's real name. Every eligibility gate passes and the consent dialog is suppressed.
4. The pricing screen shows a renewal date computed from the victim's genuine on-chain expiry. The price beside it and the calldata behind the button are for the attacker's label.
5. The victim approves. Their payment extends the attacker's name by a year. Their own name's expiry does not move.
6. The success screen confirms the renewal, shows the raw twin label, and shows an Expires date computed from the victim's real expiry plus the purchased duration.

### Relationship to the Known Issues entry on name validators

The Known Issues entry a triager will reach for is the one on lenient name validators, and the program's rule is strict: "Reports that describe the same root cause as an entry below will be treated as duplicates, even if the report describes a different exploitation path or entry point." So the argument has to be about root cause, not path. Two facts do that work.

First, that entry affirmatively disclaims this consequence. Its text, verbatim:

> Our name validators accept inputs that ENS normalization later rejects or transforms, including homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot. The practical impact is homograph-style display confusion in our UI rather than incorrect resolution.

Severity Medium, status "open, fix ready (Manager and Explorer)". What is demonstrated here is not display confusion. It is a paid, signed, on-chain write executed against a different node than the one the application resolved, gated and displayed, followed by a confirmation screen affirming a renewal that did not occur. The entry names the opposite outcome as the reason for its rating. The program's carve-out is explicit about this situation, since "new consequences of a listed root cause that materially change its severity" are in scope. Even on the least generous reading, where the root-cause sentence is shared, that is what this is.

Second, the consent gate the project shipped proves it already treats this distinction as consequential. `ThirdPartyRenewalDialog` exists for one purpose, to tell a user they will not gain ownership before they pay to extend someone else's registration. A display-confusion defect does not suppress that dialog. This one does, structurally, because the gate keys on `isOwner` and `isOwner` comes from the normalizing read while the money follows the raw label. The project built the guard. The divergence removes it silently.

Supporting both points: the divergence does not require a deceptive character at all. 17 of the 51 accepted-and-divergent inputs measured in Part 1 of the PoC are neither homoglyphs, nor bidirectional-control or other control characters, nor a leading or trailing dot. Six are plain ASCII (`fo_o`, `ab--cd`, `xn--a`, `12--34`, `a_b_c`, `ab--`) and eleven are fullwidth, circled, mathematical-bold, ligature or long-s characters that look nothing like the ASCII letters normalization maps them onto. A validator fix scoped to that entry's own enumeration does not close those.

One honest qualification so the argument is read correctly. In sibling raw-parameter cases elsewhere in the app the divergence fires on a bare capital letter, since with this repository's pinned viem `normalize('Victim.eth')` equals `victim.eth` while `namehash('Victim.eth')` does not equal `namehash('victim.eth')`. That case does not apply here, because `parseName` lowercases at `name-parser.ts:46`.

Two tracker entries sit closer than the rest and neither covers this. One concerns the same dialog firing when it should not, where renewing your own name in grace period shows a modal implying you are extending someone else's name. This is its mirror image: the modal staying silent when the name really does belong to someone else, and silent because of which name was read rather than because of grace. The other concerns a renewal extending the base-registrar expiry but not the NameWrapper expiry on a wrapped V1 name, which is one name with two expiry records out of step. Here there are two different names, and the v2 registrar behaves exactly as specified for the label it was handed. I also checked the renewal-blocked-after-registration QA item, which is unrelated.

## Impact details

### What the victim loses

The transaction detail the user approves, meaning which name is being renewed, is wrong on screen before the signature at `PricingSummaryCard.tsx:42` and wrong again after it at `RenewalDetails.tsx:50` and `105-108`. That is the impact entry selected above, and both of its sub-bullets are satisfied.

The demonstrated loss is the renewal fee, 8.000021 USDC in the run below, plus a name the user believes is renewed and is not.

On the phrase "unintended recipient" in that entry. A reviewer may observe that the funds reach the registrar at the correct price, which is entirely intended, and conclude the clause is not met. The recipient of a `renew()` is not the contract collecting the rent. The registrar is a payment rail on every renewal, correct or not. The party that receives what was purchased is whoever owns the label in the calldata, and here that is the attacker rather than the user who approved. The value transferred is a year of registration, and it lands on a name the victim neither chose nor knows about.

The realistic attacker motive is the second half of the loss rather than the first, and this part is reasoned rather than demonstrated. A victim who sees "Renewal Complete!" and an expiry a year out has no reason to renew again. A name that is not renewed eventually enters grace and then release, at which point anyone, including the attacker, can register the real name. I did not run a fork forward through expiry, grace and re-registration, and I do not claim loss of the name as a demonstrated impact. I raise it because it explains why an attacker would pay for a twin that is worth nothing on its own.

### Factors bearing on severity

I am submitting this at the severity the program assigns to the matched entry. Three factors bound the attack, and I would rather surface them than have a triager find them. If triage weighs them and lands on High, that is a reasonable outcome and I would accept it.

The attacker pays per victim, in advance. The twin must be registered before the link goes out, at full registration price for that specific name. There is no broadcast version of this attack. One twin buys one target.

Delivery is targeted and manual. The victim has to open a link the attacker chose for them. It reads as a routine link to their own name, but it is still one link to one person. Nothing fires on a user who reaches the Manager under their own steam.

The attacker's direct gain is close to zero. The registration the victim's money extends sits under a label that is not in ENSIP-15 normalized form, so no conformant client resolves to that node given the name as it renders. It cannot be sold or used as an ENS name. The attacker converts the victim's money into a year of an asset nobody can resolve, minus what they paid for the twin.

One factor cuts the other way. Timing is free, because expiry dates are public, so the link arrives exactly when a renewal reminder would be expected and the victim is least likely to question it.

### Impacts I am not claiming

Direct theft of user funds: no. `ETHRegistrar.renew` pulls the rent from `_msgSender()`, as the transaction-manager comment at `registration.actors.ts:1006-1010` says, and the balance delta below matches the quoted price exactly. The attacker receives an in-kind benefit, not the victim's money.

Direct theft of user NFTs: no. No token changes hands.

Hijacking or bricking an ENS name during a Manager flow: no. That entry enumerates registration, v1 to v2 migration and transfer. Renewal is not among them, and the victim's name is neither reassigned nor rendered permanently inaccessible by this transaction. It is simply not renewed.

Taking state-modifying authenticated actions on behalf of other users without any interaction by that user: no. The victim interacts, by approving.

Malicious interactions with an already-connected wallet: a triager may read it that way and I will not argue the point, since its first listed example, "Modifying transaction arguments or parameters", describes the mechanism exactly. I rest the claim on the display entry because that entry describes what the user sees before approving, which is the part I measured. I am not asking for the two to be counted twice.

### Demonstrated versus reasoned

The consent-gate suppression is executed at the predicate and code-traced at the input. `shouldShowThirdPartyRenewalWarning(true) === false` is executed. That `isOwner` is true on the twin's page follows from `profileOwner.ts:25` and `ProfileView.tsx:99-152`. I did not drive a browser to observe the missing dialog.

Both screens are reconstructed rather than screenshotted. The PoC calls the app's own `getDurationExpiryDateForDisplay` with the same two inputs `RenewalDetails.tsx:23-32` gives it and formats with the same `date-fns` call at line 108. The banner strings are quoted from `RenewalCompletionBanner.tsx:13` and `:16`. The pre-signature pricing line is the same computation, since `PricingSummaryCard.tsx:17-22` calls that same function with the same `currentExpiry`, so the date shown for it is the date that screen would show. I did not render either component in a browser.

The name-lapse escalation is reasoned from the display, not demonstrated, as stated above.

The on-chain half is fully executed, including a before-and-after expiry control on both names and a negative control.

## Proof of Concept

## Proof of concept

### Environment

Everything ran against a local anvil fork of Sepolia at chain id 11155111. No deployed ENS asset was contacted.

Node 22.18 or later is required, because the script relies on native TypeScript type stripping to import the app's own modules. My run was on Node v26.5.0. The script must sit in `apps/manager/` so that the repository's pinned `viem`, `@ensdomains/ensjs` and `date-fns`, and the app's own source, all resolve from there.

Contracts read and written, all on the fork:

* ETHRegistrar: `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`
* Payment token (USDC): `0x768F42455A2D082E23ceeF7d51e5787C82d67a39`
* Resolver used at registration: `0x640294a2b2d87e7f522db3e3e3e876764bce170d`
* Registry: resolved through `getChainContractAddress({ chain, contract: 'ensRegistry' })`, the same registry the app reads expiry from at `profileExpiry.ts:54-57`

Accounts: the attacker is anvil's first default account, the victim is index 2 of the standard anvil test mnemonic. Both are funded and both approve the registrar for the payment token in the harness. The script first clears any EIP-7702 delegation squatted on those addresses on the forked chain, because such a delegation reverts inside `onERC1155Received` and would otherwise break registration for reasons unrelated to this bug. Duration in every case is one year, 31,536,000 seconds.

On what is and is not stubbed: the script imports the real in-scope exports rather than transcribed copies, namely `parseName`, `validateENSName`, `parseRenewableName`, `canRenewV2Name`, `shouldShowThirdPartyRenewalWarning`, `normalizeEth2LdName`, `getDurationExpiryDateForDisplay`, and the Explorer's `isValidEnsName`. The two module hooks at the top supply only what a bundler would supply: the `@/*` alias from `tsconfig.json` and `vitest.config.ts`, extensionless specifiers, and a null stand-in for the `.tsx` icon module that `nameUtils.ts` imports but never calls, since Node cannot strip JSX. No logic under test is stubbed. I chose this over transcribing the functions so that a reviewer is checking the shipped code rather than my copy of it.

Steps:

1. `anvil --fork-url <sepolia-rpc> --chain-id 11155111`
2. Save the script below as `poc.mjs` in `apps/manager/`
3. `node poc.mjs`

### Source

```javascript
/**
 * PoC — Manager renewal: the app resolves, gates and displays one ENS node and
 *       signs a paid `renew()` against a different one.
 *
 * SETUP
 *   1. anvil --fork-url <sepolia-rpc> --chain-id 11155111
 *   2. Place this file in apps/manager/ (the repo's pinned viem, @ensdomains/ensjs,
 *      date-fns and the app's own source all resolve from there)
 *   3. node poc.mjs            (Node >= 22.18 — native TypeScript type stripping)
 *
 * PART 1  offline, run against the REAL in-scope apps/manager exports
 * PART 2  on chain, with a before/after expiry control on BOTH names
 * PART 3  appendix, no severity claimed
 */
import { readFileSync } from 'node:fs'
import { registerHooks } from 'node:module'
import { fileURLToPath, pathToFileURL } from 'node:url'

// ---------------------------------------------------------------------------
// Load the application's own modules — no transcription. Node strips the types;
// these two hooks only supply what a bundler would normally supply:
//   resolve: the `@/*` -> `src/*` alias from tsconfig.json / vitest.config.ts,
//            and extensionless / `.js`-spelled TypeScript specifiers
//   load:    a null stand-in for `.tsx` React components that a validator module
//            imports but never calls (nameUtils.ts -> StableCoinsIcons.tsx).
//            Node cannot strip JSX. No logic under test is stubbed.
// ---------------------------------------------------------------------------
const SRC = pathToFileURL(`${process.cwd()}/src/`).href
registerHooks({
  resolve(spec, ctx, next) {
    const cands = []
    const push = (s) =>
      cands.push(s, s.replace(/\.js$/, '.ts'), `${s}.ts`, `${s}.tsx`, `${s}/index.ts`)
    if (spec.startsWith('@/')) push(SRC + spec.slice(2))
    else push(spec)
    let lastErr
    for (const c of cands) {
      try { return next(c, ctx) } catch (e) { lastErr = e }
    }
    throw lastErr
  },
  load(url, ctx, next) {
    if (url.endsWith('.tsx')) {
      const src = readFileSync(fileURLToPath(url), 'utf8')
      const names = [...src.matchAll(/^export\s+(?:const|function|class)\s+([A-Za-z0-9_$]+)/gm)]
      const body = names.map(([, n]) => `export const ${n} = () => null;`).join('\n')
      return { format: 'module', shortCircuit: true, source: `${body}\nexport default null;\n` }
    }
    return next(url, ctx)
  },
})

// ===== the real, in-scope apps/manager exports =============================
const { parseName } =
  await import('./src/features/register-v2/utils/name-parser.ts')
const { validateENSName } =
  await import('./src/features/shared/registration/nameUtils.ts')
const { parseRenewableName, canRenewV2Name } =
  await import('./src/features/renew/utils/renewableName.ts')
const { shouldShowThirdPartyRenewalWarning } =
  await import('./src/features/renew/utils/thirdPartyRenewalWarning.ts')
const { normalizeEth2LdName } =
  await import('./src/features/profile/service/profileName.ts')
const { getDurationExpiryDateForDisplay } =
  await import('./src/features/register-v2/utils/time.ts')
// the Explorer's validator, for contrast (apps/portal, also in scope)
const { isValidEnsName } =
  await import('../portal/src/utils/token/isNormalized.ts')

const {
  createPublicClient, createWalletClient, http, encodeFunctionData, parseAbi, labelhash,
} = await import('viem')
const { privateKeyToAccount, mnemonicToAccount } = await import('viem/accounts')
const { sepolia } = await import('viem/chains')
const { normalize } = await import('viem/ens')
const { extendChainWithEns, getChainContractAddress } = await import('@ensdomains/ensjs/chain')
const { getOwner: getOwnerV2 } = await import('@ensdomains/ensjs/public/v2')
const { format } = await import('date-fns')

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
// the app's two validators, called exactly as the register/renew paths call them
const managerAccepts = (name) => parseName(name).isOk() && validateENSName(name) === null

// ===========================================================================
console.log('=== PART 1 — offline, against the REAL apps/manager exports ===\n')
// ===========================================================================

// --- 1.1 how wide the raw-vs-normalized divergence is ----------------------
// Each entry is tagged with the category a triager needs in order to decide
// whether it falls inside EXP-INPUT-005's enumeration ("homoglyphs,
// bidirectional-control and other control characters, and a single leading or
// trailing dot"). `ascii-structural` and `compat-mapping` fall outside it.
const corpus = new Map() // label -> category
const add = (label, cat) => { if (!corpus.has(label)) corpus.set(label, cat) }

// pure ASCII. ENSIP-15 underscore-placement and label-extension rules.
for (const s of ['fo_o', 'a_b_c', 'ab--cd', 'xn--a', '12--34', 'ab--']) add(s, 'ascii-structural')
// invisible / format / control characters
for (const n of [0x00ad, 0x200b, 0x200c, 0x200d, 0x2060, 0xfeff, 0xfe0f, 0xfe0e,
  0x034f, 0x180e, 0x061c, 0x200e, 0x200f, 0x202a, 0x202b, 0x202c, 0x202d, 0x202e,
  0x2066, 0x2067, 0x2068, 0x2069, 0x0000]) add(`ali${cp(n)}ce`, 'invisible/control')
// characters that LOOK like their ASCII target
for (const [from, to] of [['i', cp(0x0130)], ['l', cp(0x217c)], ['v', cp(0x2174)],
  ['o', cp(0x03bf)], ['e', cp(0x0435)], ['a', cp(0x0430)], ['i', cp(0x0131)]]) {
  if ('alice'.includes(from)) add('alice'.replace(from, to), 'homoglyph')
  add(to + 'alice', 'homoglyph')
}
// characters that DO NOT look like their ASCII target but normalization maps
// onto it anyway: fullwidth, circled, mathematical-bold, ligatures, long s.
for (const [from, to] of [['a', cp(0xff41)], ['c', cp(0xff43)], ['s', cp(0x017f)]]) {
  if ('alice'.includes(from)) add('alice'.replace(from, to), 'compat-mapping')
  add(to + 'alice', 'compat-mapping')
}
for (const n of [0x2460, 0x24b6, 0x1d400, 0x1d41a, 0x0132, 0xfb00]) {
  add(cp(n) + 'alice', 'compat-mapping')
}

const classA = [] // accepted, but ENSIP-15 rejects the label outright
const classB = [] // accepted, but ENSIP-15 maps it onto a DIFFERENT label
for (const [label, cat] of corpus) {
  const input = `${label}.eth`
  const parsed = parseName(input)
  if (parsed.isErr() || validateENSName(input) !== null) continue // Manager rejects it
  const writes = parsed.value.label
  let norm
  try { norm = normalize(input).split('.')[0] }
  catch { classA.push({ label, cat, writes, portalRejects: !isValidEnsName(input) }); continue }
  if (writes !== norm) classB.push({ label, cat, writes, norm, portalRejects: !isValidEnsName(input) })
}
const all = [...classA, ...classB]

console.log(`corpus: ${corpus.size}   accepted-and-divergent: ${all.length}\n`)
console.log(`CLASS A - buys a name no conformant client can resolve: ${classA.length}`)
for (const r of classA) console.log(`   ${esc(r.label).padEnd(30)} registers ${esc(r.writes)}`)
console.log(`\nCLASS B - transacts on a different node than displayed: ${classB.length}`)
for (const r of classB) {
  console.log(`   ${esc(r.label).padEnd(26)} registers ${esc(r.writes).padEnd(26)} displayed-as ${esc(r.norm)}`)
}
console.log(`\nOf these ${all.length}, the Explorer's validator would have rejected ` +
  `${all.filter((r) => r.portalRejects).length}.`)

const byCat = {}
for (const r of all) byCat[r.cat] = (byCat[r.cat] ?? 0) + 1
console.log('\nBy input category (which of these are inside EXP-INPUT-005\'s enumeration?):')
for (const [cat, n] of Object.entries(byCat)) console.log(`   ${cat.padEnd(20)} ${n}`)
const outside = all.filter((r) => r.cat === 'ascii-structural' || r.cat === 'compat-mapping')
console.log(`   -> ${outside.length} of ${all.length} are neither homoglyphs, nor bidirectional-control`)
console.log('      or other control characters, nor a leading/trailing dot.')

// --- 1.2 the renewal twin, through the real route-loader functions ---------
console.log('\n\n--- 1.2 renew/$name loader, real functions, on the twin ---\n')
const VLABEL = 'victim'
const TWIN = `victim${cp(0x00ad)}` // U+00AD SOFT HYPHEN: renders as nothing
const twinName = `${TWIN}.eth`
const farFuture = new Date(Date.now() + 200 * 86400_000)
console.log(`  route param                              ${esc(twinName)}`)
console.log(`  parseRenewableName(param).label      ->  ${esc(parseRenewableName(twinName)._unsafeUnwrap().label)}   <- RAW, goes to renew() calldata`)
console.log(`  normalizeEth2LdName(param).name      ->  ${esc(normalizeEth2LdName(twinName).name)}   <- NORMALIZED, what the expiry read + display use`)
console.log(`  canRenewV2Name(param, expiry)        ->  ${canRenewV2Name(twinName, farFuture)}   <- eligibility gate passes`)
console.log(`  labelhash(raw)    ${labelhash(TWIN)}`)
console.log(`  labelhash(normal) ${labelhash(VLABEL)}`)

// --- 1.3 the consent gate the project itself shipped ----------------------
console.log('\n--- 1.3 ThirdPartyRenewalDialog consent gate ---\n')
console.log(`  shouldShowThirdPartyRenewalWarning(false) -> ${shouldShowThirdPartyRenewalWarning(false)}  (dialog shown: "You won't gain ownership of the name")`)
console.log(`  shouldShowThirdPartyRenewalWarning(true)  -> ${shouldShowThirdPartyRenewalWarning(true)}  (no dialog, plain link straight into the paid flow)`)
console.log('  isOwner is derived from profileOwner.ts getOwner(), which normalizes first')
console.log('  (profileOwner.ts:25). On the twin\'s page that read returns the VICTIM\'s own')
console.log('  ownership, so isOwner === true and the interstitial never renders. (code-traced)')

// ===========================================================================
console.log('\n\n=== PART 2 — on chain ===\n')
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
// the same registry the app reads expiry from (profileExpiry.ts:54-57)
const REGISTRY = getChainContractAddress({ chain, contract: 'ensRegistry' })

const regAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function isAvailable(string label) view returns (bool)',
  'function getRenewPrice(string label, uint64 duration, address paymentToken) view returns (uint256)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256)',
  'function renew(string label, uint64 duration, address paymentToken, bytes32 referrer)',
])
const registryAbi = parseAbi(['function getExpiry(uint256 id) view returns (uint64)'])
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
const DUR = 31536000n
const register = async (acct, label) => {
  const secret = `0x${'33'.repeat(32)}`
  const cm = await pub.readContract({ address: REG, abi: regAbi, functionName: 'makeCommitment', args: [label, acct.address, secret, ZERO, RES, DUR, ZH] })
  await send(wc(acct), REG, encodeFunctionData({ abi: regAbi, functionName: 'commit', args: [cm] }))
  await rpc('evm_increaseTime', [90]); await rpc('evm_mine', [])
  await send(wc(acct), REG, encodeFunctionData({ abi: regAbi, functionName: 'register', args: [label, acct.address, secret, ZERO, RES, DUR, USDC, ZH] }))
}
// expiry exactly as the app reads it: registry.getExpiry(uint256(labelhash(label)))
const expiryOf = async (label) => pub.readContract({
  address: REGISTRY, abi: registryAbi, functionName: 'getExpiry', args: [BigInt(labelhash(label))],
})
const asDate = (secs) => (secs === 0n ? 'not registered' : new Date(Number(secs) * 1000).toISOString())

await prep(attacker); await prep(victim)
const SUF = String(Math.floor(Math.random() * 1e6))
const BASE = `renewdemo${SUF}`              // the victim's real name
const IMPOSTOR = `${BASE}${cp(0x00ad)}`     // the attacker's twin

console.log(`  victim name  : ${JSON.stringify(`${BASE}.eth`)}`)
console.log(`  attacker name: ${esc(`${IMPOSTOR}.eth`)}   (renders identically)`)
console.log(`  normalize(attacker name) -> ${JSON.stringify(normalize(`${IMPOSTOR}.eth`))}`)
console.log(`  labelhash differ: ${labelhash(BASE)} vs ${labelhash(IMPOSTOR)}\n`)

await register(victim, BASE)
await register(attacker, IMPOSTOR)

// --- what the app has in hand when the victim opens /renew/<twin>.eth ------
const loaderLabel = parseRenewableName(`${IMPOSTOR}.eth`)._unsafeUnwrap().label
const loaderExpiry = await expiryOf(normalizeEth2LdName(`${IMPOSTOR}.eth`).label)
console.log('  /renew/$name loader output (real functions + real chain reads):')
console.log(`    label        = ${esc(loaderLabel)}                 -> renew() calldata`)
console.log(`    currentExpiry= ${loaderExpiry} (${asDate(loaderExpiry)})  <- the VICTIM's name\n`)

// --- BEFORE ---------------------------------------------------------------
const vBefore = await expiryOf(BASE)
const aBefore = await expiryOf(IMPOSTOR)
const balBefore = await pub.readContract({ address: USDC, abi: tokAbi, functionName: 'balanceOf', args: [victim.address] })
console.log('  BEFORE')
console.log(`    victim   ${esc(`${BASE}.eth`)} expiry ${vBefore} (${asDate(vBefore)})`)
console.log(`    attacker ${esc(`${IMPOSTOR}.eth`)} expiry ${aBefore} (${asDate(aBefore)})`)

// --- what the app shows BEFORE the signature ------------------------------
// PricingSummaryCard.tsx:10  currentExpiry, straight from the renewal context
//                      :17-22 getDurationExpiryDateForDisplay(duration, that expiry)
//                      :28/:39/:42  "Renewing for <duration> expiring on <date>"
const shownPre = getDurationExpiryDateForDisplay(
  Number(DUR),
  new Date(Number(loaderExpiry) * 1000),
)
console.log('\n  PRICING SCREEN, BEFORE THE SIGNATURE (the app\'s own function, the app\'s own inputs):')
console.log(`    "Renewing for <1 year> expiring on ${format(shownPre, 'MMMM d, yyyy')}"  <- from the VICTIM's expiry`)
console.log(`    price + calldata on the same screen use label ${esc(loaderLabel)}  <- the ATTACKER's registration`)

// --- the victim signs -----------------------------------------------------
let outcome
let price = 0n
try {
  price = await pub.readContract({ address: REG, abi: regAbi, functionName: 'getRenewPrice', args: [loaderLabel, DUR, USDC] })
  await send(wc(victim), REG, encodeFunctionData({ abi: regAbi, functionName: 'renew', args: [loaderLabel, DUR, USDC, ZH] }))
  outcome = 'SUCCEEDED'
} catch (e) { outcome = `REVERTED (${String(e.message).split('\n')[0].slice(0, 60)})` }

const vAfter = await expiryOf(BASE)
const aAfter = await expiryOf(IMPOSTOR)
const balAfter = await pub.readContract({ address: USDC, abi: tokAbi, functionName: 'balanceOf', args: [victim.address] })
console.log(`\n  victim signs renew(${esc(loaderLabel)}, 1 year, USDC) -> ${outcome}`)
console.log('\n  AFTER')
console.log(`    victim   ${esc(`${BASE}.eth`)} expiry ${vAfter} (${asDate(vAfter)})   delta ${vAfter - vBefore}s`)
console.log(`    attacker ${esc(`${IMPOSTOR}.eth`)} expiry ${aAfter} (${asDate(aAfter)})   delta ${aAfter - aBefore}s`)
console.log(`    victim USDC spent: ${(balBefore - balAfter).toString()} (6 decimals), price quoted ${price}`)

// --- what the app tells the victim afterwards -----------------------------
// RenewalDetails.tsx:23-32 + :50 + :105-108, rendered by SuccessStep.tsx:8-9
const shown = getDurationExpiryDateForDisplay(Number(DUR), new Date(Number(loaderExpiry) * 1000))
console.log('\n  SUCCESS SCREEN (computed with the app\'s own getDurationExpiryDateForDisplay):')
console.log('    "Renewal Complete! The ENS domain has been successfully renewed."')
console.log(`    DomainCard domainName = ${esc(`${loaderLabel}.eth`)}   <- RAW twin label`)
console.log(`    Expires               = ${format(shown, 'MMMM d, yyyy')}   <- victim's expiry + 1y`)
console.log(`    victim's ACTUAL new expiry = ${asDate(vAfter)}`)

// --- negative control: same flow, twin not registered ---------------------
const NEVER = `neverreg${SUF}${cp(0x00ad)}`
let caseA
try {
  await send(wc(victim), REG, encodeFunctionData({ abi: regAbi, functionName: 'renew', args: [NEVER, DUR, USDC, ZH] }))
  caseA = 'SUCCEEDED'
} catch { caseA = 'REVERTED' }
console.log(`\n  NEGATIVE CONTROL — identical flow, twin never registered: renew() ${caseA}`)

console.log('\n=== RESULT ===')
if (balBefore - balAfter > 0n && aAfter > aBefore && vAfter === vBefore) {
  console.log("The victim paid. The ATTACKER's registration gained the time.")
  console.log("The victim's own name gained nothing and keeps expiring.")
  console.log('The app then displayed the raw twin label with a renewed expiry date.')
} else {
  console.log('Preconditions not met on this run — see the deltas above.')
}

// ===========================================================================
console.log('\n\n=== PART 3 — appendix (no severity claimed) ===\n')
// The same missing normalization on the REGISTRATION path. There is no attacker
// and no victim here: the user receives exactly the label shown to them. It is
// recorded only because it is the same one-line defect, cheaply demonstrated.
// ===========================================================================
for (const stem of ['fo_o', 'ab--cd', 'plain']) {
  const label = stem + SUF
  const accepted = managerAccepts(`${label}.eth`)
  let normVerdict
  try { normVerdict = `normalize -> ${JSON.stringify(normalize(`${label}.eth`))}` }
  catch { normVerdict = 'normalize THROWS' }
  const avail = await pub.readContract({ address: REG, abi: regAbi, functionName: 'isAvailable', args: [label] })
  let outcome2
  try { await register(attacker, label); outcome2 = 'REGISTERED' } catch { outcome2 = 'register reverted' }
  let lookup
  try { lookup = JSON.stringify(await getOwnerV2(pub, { name: normalize(`${label}.eth`) })) }
  catch { lookup = 'UNRESOLVABLE by any conformant client' }
  console.log(`  ${esc(label).padEnd(16)} manager-accepts=${accepted}  ${normVerdict.padEnd(34)} isAvailable=${avail}  ${outcome2}`)
  console.log(`  ${''.padEnd(16)} conformant lookup: ${lookup}`)
}
console.log('\n  ("plain" is the control: it registers AND resolves, so the harness is sound)')
```

### Output, Part 1: how wide the divergence is

```
corpus: 52   accepted-and-divergent: 51

CLASS A - buys a name no conformant client can resolve: 32
CLASS B - transacts on a different node than displayed: 19

Of these 51, the Explorer's validator would have rejected 51.

By input category:
   ascii-structural     6
   invisible/control    22
   homoglyph            12
   compat-mapping       11
   -> 17 of 51 are neither homoglyphs, nor bidirectional-control
      or other control characters, nor a leading/trailing dot.
```

The category tagging is mine, not the program's, so here is the rule I used. `ascii-structural` covers `fo_o`, `a_b_c`, `ab--cd`, `xn--a`, `12--34` and `ab--`, which are plain ASCII letters, digits, hyphens and underscores rejected by ENSIP-15 for underscore placement and label extension. `compat-mapping` covers fullwidth, circled, mathematical-bold, ligature and long-s characters, which are visually distinct from the ASCII letters normalization maps them onto, so they are not homoglyphs in any ordinary sense.

### Output, Part 2: on chain, with a before and after control on both names

This is the stdout of a real run. The label suffix is randomised per run and the timestamps depend on when the fork was taken, so a reviewer's own output will differ in the label and the dates and in nothing else. The deltas are the invariant part.

```
  victim name  : "renewdemo529736.eth"
  attacker name: "renewdemo529736\u{ad}.eth"   (renders identically)
  normalize(attacker name) -> "renewdemo529736.eth"
  labelhash differ: 0xecc7c95f133c3ed761cef1ad88673255a118894dda83aa001b956c647fa462a7 vs 0x6c4564632d3e83bb3d092e0848b43fe067a9d9b98ad49d1454f72a7c10d9f1a4

  /renew/$name loader output (real functions + real chain reads):
    label        = "renewdemo529736\u{ad}"                 -> renew() calldata
    currentExpiry= 1820771640 (2027-09-12T17:54:00.000Z)  <- the VICTIM's name

  BEFORE
    victim   "renewdemo529736.eth" expiry 1820771640 (2027-09-12T17:54:00.000Z)
    attacker "renewdemo529736\u{ad}.eth" expiry 1820771733 (2027-09-12T17:55:33.000Z)

  PRICING SCREEN, BEFORE THE SIGNATURE (the app's own function, the app's own inputs):
    "Renewing for <1 year> expiring on September 11, 2028"  <- from the VICTIM's expiry
    price + calldata on the same screen use label "renewdemo529736\u{ad}"  <- the ATTACKER's registration

  victim signs renew("renewdemo529736\u{ad}", 1 year, USDC) -> SUCCEEDED

  AFTER
    victim   "renewdemo529736.eth" expiry 1820771640 (2027-09-12T17:54:00.000Z)   delta 0s
    attacker "renewdemo529736\u{ad}.eth" expiry 1852307733 (2028-09-11T17:55:33.000Z)   delta 31536000s
    victim USDC spent: 8000021 (6 decimals), price quoted 8000021

  SUCCESS SCREEN (computed with the app's own getDurationExpiryDateForDisplay):
    "Renewal Complete! The ENS domain has been successfully renewed."
    DomainCard domainName = "renewdemo529736\u{ad}.eth"   <- RAW twin label
    Expires               = September 11, 2028   <- victim's expiry + 1y
    victim's ACTUAL new expiry = 2027-09-12T17:54:00.000Z

  NEGATIVE CONTROL - identical flow, twin never registered: renew() REVERTED
```

The `<1 year>` span is mine. That one span is rendered by `DurationLabel`, a React component, and the script computes values rather than rendering components. Every other string and date in that block is the app's own function output.

The two expiry reads are the control that matters. The victim's name moves by 0 seconds and the attacker's by exactly the purchased 31,536,000, read from the same registry the app itself reads. The claim is measured, not inferred from the calldata label.

The negative control bounds the harmless variant. If nobody holds the twin, `renew()` reverts and the victim loses only gas, though their own name is still not renewed and they have no way to know that from the UI.

One caveat before someone else raises it. In this run the attacker's twin happens to land on the same calendar day as the victim's name, because the harness registers both names ninety seconds apart. That is an artefact of the harness, not of the attack. The date on screen is computed from `currentExpiry`, which is always the victim's expiry, and it is asserted about the name the victim believes they renewed, which did not move at all.

### Output, Part 3: the registration path, appendix, no severity claimed

The same missing normalization is present on `/register/$name`. The loader derives its label from `parseName` at `routes/register/$name.tsx:24`, returned at line 57, and every downstream consumer uses that same raw value. Part 3 registers `fo_o<suffix>` and `ab--cd<suffix>` on the fork. Both are accepted by both Manager validators, both register, and neither resolves through a conformant client, while the `plain<suffix>` control registers and resolves normally.

I am not claiming an impact for this. There is no attacker here, since the user types a label and receives exactly the label they were shown, and it maps to no entry on the impacts list. It is in the script because it is the same one-line defect and costs nothing to demonstrate, and because the pure-ASCII cases are useful evidence about the shape of the root cause. It should not carry any part of this report's severity.

## Recommended fix

1. Normalize the route parameter in the `/renew/$name` loader before anything else, and fail closed on throw. Where `normalize(param)` differs from `param`, redirect to the canonical path so the URL, the displayed name, the consent gate and the calldata all agree on one node.
2. Derive the renewal label from the same normalized value the expiry read uses, `normalizeEth2LdName(name).label`, instead of `parseName`'s raw label, so `renew/$name.tsx:60-63` can no longer return a label and an expiry belonging to different names.
3. Replace `parseName`'s `trim().toLowerCase()` with ENSIP-15 `normalize()` and delete the ASCII blocklists at `name-parser.ts:11` and `nameUtils.ts:132`. Normalization subsumes them.
4. Add a write-path regression assertion: `labelhash(labelSentOnChain) === labelhash(normalize(userInput))`. The 52-case corpus in Part 1 works directly as a test-vector set.
5. Until item 1 lands, the consent gate is unreliable by construction. `shouldShowThirdPartyRenewalWarning` should key on ownership of the label actually being renewed, not on ownership of its normalized form.

`apps/portal/src/utils/token/isNormalized.ts:39-57`, the Explorer's `isValidEnsName`, is a working implementation of the check the Manager is missing, in the same monorepo. It rejected all 51 divergent inputs in Part 1.

## Adjacent prior art, disclosed

There is an open issue and an unmerged pull request in `@ensdomains/ensjs` about `getAvailable` hashing the raw label instead of the normalized one. I disclose it because a reviewer will find it, and because it corroborates rather than disqualifies.

`@ensdomains/ensjs` is an unmodified third-party dependency of this monorepo, and the program places unmodified third-party dependencies out of scope, whereas the root cause reported here is in `apps/manager`. That fix also touches only the availability read, and the label reaching `renew()` never passes through `getAvailable`, so merging it would not change the renewal path at all. That issue is specifically about emoji and variation selectors, and the ASCII and compat-mapping cases above are unrelated to it. The pinned dependency in this repository predates the fix in any case.

Together with a merged ensjs change enforcing normalization in `getName`, and with the production ENS app normalizing the name at the edge in its Cloudflare middleware, these show the organization already treats missing ENSIP-15 normalization as a defect to fix rather than as intended behaviour.

The target repository has issues disabled and no relevant pull requests or commits. The upstream monorepo is private. I found no public disclosure of this.

## References

Affected source, all under `apps/manager/src` unless noted:

* `routes/renew/$name.tsx:20,26-28,52,60-63`, the loader where the split happens
* `features/register-v2/utils/name-parser.ts:11,46`, the homegrown parser
* `features/shared/registration/nameUtils.ts:90-143`, blocklist at `:132`, minimum length at `:124-130`
* `features/renew/utils/renewableName.ts:25-36`, returns the raw label untouched
* `features/renew/utils/thirdPartyRenewalWarning.ts:1-3`, the suppressed consent gate
* `features/renew/.../ThirdPartyRenewalDialog.tsx:64-70` and `ProfileRenewAction.tsx:66-88`, the dialog and its call site
* `features/profile/service/profileOwner.ts:25,55-58`, the normalizing ownership read
* `features/profile/service/profileExpiry.ts:54-57,77,104-110`, the normalizing expiry read
* `PricingSummaryCard.tsx:10,17-22,28,39,42`, the pre-signature display
* `RenewalDetails.tsx:15,23-32,50,69,105-108` and `RenewalCompletionBanner.tsx:13,16`, the confirmation screen
* `packages/transaction-manager/.../registration.actors.ts:1006-1010,1020-1031`, the calldata encoder and the payer comment
* `apps/portal/src/utils/token/isNormalized.ts:39-57`, the working validator in the same monorepo

External:

* ENSIP-15, the normalization standard: <https://docs.ens.domains/ensip/15/>
* ENS documentation on name processing, which states the violated rule directly: "You MUST normalize a name before you DNS-encode it! If you don't, then when you pass those DNS-encoded bytes into a contract method, incorrect namehashes/labelhashes may be derived." <https://docs.ens.domains/resolution/names/>
* ENS blog on normalization, equally direct: if a name cannot be normalized, an integration should reject it rather than hash or resolve the raw input. <https://ens.domains/blog/post/how-ens-normalization-works>
