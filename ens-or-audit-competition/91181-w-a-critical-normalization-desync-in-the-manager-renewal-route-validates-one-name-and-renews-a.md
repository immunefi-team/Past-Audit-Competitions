# 91181 w a critical normalization desync in the manager renewal route validates one name and renews another letting a link make a victim pay to extend an attacker s name

**Submitted on Aug 30th 2026 at 12:26:07 UTC by @webrainsec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91181
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions
    * Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated)
    * Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Manager's renewal route derives two values from the same URL parameter using two different normalizers: eligibility and the expiry date shown to the user come from the ENSIP-15 normalized name, while the label placed in the `renew` calldata comes from `String.prototype.toLowerCase`. ENSIP-15 deletes a set of zero-width characters that `toLowerCase` keeps, so an attacker can craft a parameter that resolves to the victim's real name at the validation gate but renews a different label, with no visible difference anywhere on screen. In production this lets an attacker register a pixel-identical lookalike of any `.eth` name, send the owner a renewal link, and have the application charge that owner a full renewal fee to extend the attacker's name while their own name is left unrenewed. The app then reports "Renewal Complete!" with an expiry date computed from the victim's real name, so the deception holds on both sides of the signature.

## Vulnerability Details

`apps/manager/src/routes/renew/$name.tsx` takes `name` from the URL and derives two values from it inside one loader.

The label, at `:20`, with no ENSIP-15 normalization:

```ts
const parsedName = parseRenewableName(name)
```

Eligibility and the expiry, at `:26`, with ENSIP-15 normalization:

```ts
const expiryData = await queryClient.ensureQueryData(profileExpiryQuery(name))
```

Both are handed to the page together, at `:60-63`:

```ts
return {
  label: parsedName.value.label,    // raw
  currentExpiry: expiryData.expiry, // normalized
}
```

The gate normalizes. `profileExpiry.ts:77` calls `normalizeEth2LdName`, which reaches `profileName.ts:18`:

```ts
normalized = normalize(name)   // viem/ens, ENSIP-15
```

The label does not. `parseRenewableName` calls `parseName`, and `register-v2/utils/name-parser.ts:46` is:

```ts
const normalized = name.trim().toLowerCase()
```

`toLowerCase` is ASCII case folding. It applies no ENSIP-15 mapping, does not compose combining sequences, and does not remove ignored characters. `INVALID_LABEL_CHARS` at `name-parser.ts:11` covers only ASCII punctuation, so the raw label passes validation unchanged.

Every check in the loader runs on the normalized name: protocol at `:30` and `:40`, grace period at `:44`, `canRenewV2Name` at `:52`, and the expiry returned at `:62`. `routes/renew-v1/$name.tsx:15-53` has the identical shape.

The raw label then reaches calldata unmodified:

```
routes/renew/$name.tsx:72                  <RenewalPage label={label} currentExpiry={currentExpiry} />
pricing/components/ConfirmPurchase.tsx:64  uiActor.send({ type: 'renewal.start', label })
renew/state/renewalUi.machine.ts:148       submissionData: { label: event.label, ... }
.../registration.actors.ts:1072            encodeRenewData(input.label, ...)
.../registration.actors.ts:1025            const cleanLabel = label.replace('.eth', '')
.../registration.actors.ts:1029            args: [cleanLabel, duration, paymentToken, REFERER_ADDRESS]
```

### The divergence

Take a victim who owns `alice.eth`, plain ASCII. The attacker's URL parameter is the same string with a zero-width space after `ali`:

```
param          "ali<U+200B>ce.eth"  cp: 0061 006c 0069 200b 0063 0065 002e 0065 0074 0068
victim's name  "alice.eth"          cp: 0061 006c 0069 0063 0065 002e 0065 0074 0068

GATE  normalize(param)  -> "alice.eth"      the victim's real name
SINK  parseName(param)  -> "ali<U+200B>ce"  a label the victim does not own

gate labelhash 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
sink labelhash 0xa42ad15f2e51ecf4f2a50035abe21ddf90001dd52e1f2915c5128d9b2cfda580
```

`normalize` deletes the character, `toLowerCase` keeps it, and it has no glyph, so the two strings are indistinguishable in the URL bar, in the page, and in a wallet's decoded calldata.

Five characters were confirmed to survive both the register and the renew path, that is `normalize` deletes them while `parseName` and `validateENSName` both accept them:

```
U+00AD  SOFT HYPHEN            U+FE00  VARIATION SELECTOR-1
U+200B  ZERO WIDTH SPACE       U+FE0F  VARIATION SELECTOR-16
U+2060  WORD JOINER
```

Because these carry no character-set restriction, every `.eth` name is targetable, including plain ASCII ones. Unicode normalization forms produce the same desync where the name contains a composable character, for example precomposed `café` against `cafe` plus U+0301.

### What the application displays

The label string rendered on the page is the raw label, so the name on screen is the name in the calldata. The values that come from the other name are the expiry dates, and they appear on both sides of the signature.

Before signing, `pricing/components/DurationSelector.tsx:23-26` reads `currentExpiry` and computes the renewal periods the user chooses between:

```ts
const { uiActor, label, currentExpiry, protocol } = useRenewalUiContext()
const currentExpiryDate = useMemo(
  () => new Date(Number(currentExpiry) * 1000),
  [currentExpiry],
)
```

After the transaction confirms, `RenewalPage.tsx:48` renders `RenewSuccessStep`, which renders `RenewalCompletionBanner` ("Renewal Complete!" at `:13`, "The ENS domain has been successfully renewed." at `:16`) and `RenewalDetails`, whose "Expires" value at `:105-109` is computed at `:23-32` as:

```ts
const expirationDate = useMemo(() => {
  ...
  return getDurationExpiryDateForDisplay(
    Number(submissionData.duration),
    new Date(Number(currentExpiry) * 1000),   // the victim's expiry
  )
}, [currentExpiry, submissionData?.duration])
```

The date shown after payment is the victim's real expiry plus the purchased duration. It is not the attacker's new expiry, and it is not the victim's, which did not move.

### Registering the lookalike

The attacker acquires the lookalike through the Manager's own registration flow, because the availability check does not normalize either. `routes/register/$name.tsx:44` calls `getRegistrationV2AvailabilityQueryOptions`, which passes the name through `normalizeDomainNameFromUrl` (`utils/domain.ts:27-33`, strips and re-appends `.eth` and truncates only). `checkRealNameAvailability` then runs `validateENSName` (`shared/registration/nameUtils.ts:90-142`, no normalization) and hands the raw leaf label to ensjs `getAvailable`, which calls `isAvailable(label)` verbatim. The lookalike reports available and registers.

## Impact Details

The victim pays a full renewal fee, plus gas, to extend a name owned by the attacker. Their own name is not renewed. On the fork run below the victim paid 8000021 USDC units (8.000021 USDC) for a one-year renewal of a 10-character name; ENS renewal pricing scales with label length and duration, so a victim renewing a shorter or multi-year name pays proportionally more, and the attacker receives the full value of every such renewal as a free extension of their own registration.

The application corroborates the wrong name at both decision points. Before signing, the duration selector prices the renewal periods from the victim's genuine expiry date, so the "renew until" dates the victim chooses between are their own. After signing, the success screen reports completion and shows an expiry equal to the victim's real expiry plus the duration they paid for, a date that is false for both names. Nothing in the flow surfaces the mismatch, and because the payload has no glyph, no amount of care in reading the name or the URL distinguishes the two strings.

The victim's own name remains unrenewed and continues toward expiry while the application has told them the renewal succeeded.

Renewal is the operation where this mis-derivation does damage, because it is the one ENS write that requires no authority over the target name, so the wrong label succeeds instead of reverting. Record writes and `setResolver` revert against a node the victim does not control, and `setPrimaryName.ts:48-49` normalizes before building calldata.

This matches two selected in-scope impacts. The label placed in `renew` calldata is not the value the route validated, and the expiry date pre-filled into the duration selector and the success screen belongs to a different name than the one being paid for.

## References

* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/routes/renew/%24name.tsx#L20> (label, unnormalized)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/routes/renew/%24name.tsx#L26> (gate, normalized)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/routes/renew/%24name.tsx#L60-L63> (both returned together)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/routes/renew-v1/%24name.tsx#L15-L53> (same shape)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/register-v2/utils/name-parser.ts#L46> (toLowerCase)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/service/profileExpiry.ts#L77>
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/service/profileName.ts#L18> (ENSIP-15 normalize)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/renew/workflow/pricing/components/DurationSelector.tsx#L23-L26> (expiry shown pre-signature)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/renew/workflow/renewing/components/RenewalDetails.tsx#L23-L32> (expiry shown post-signature)
* <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/transaction-manager/src/machines/registration/registration.actors.ts#L1025-L1029> (renew calldata)
* ENSIP-15, ENS Name Normalization: <https://docs.ens.domains/ensip/15>

## Link to Proof of Concept

<https://gist.github.com/webrainsec/4ca6012ffb5e093fd9e3ca26134f7f27>

## Proof of Concept

Three layers, each runnable against the ENS monorepo `immunefi-team/audit-comp-ens` at branch `audit-comp-ready`. No transaction is broadcast to any public network; the on-chain step runs on a local anvil fork of Sepolia.

### Setup

```sh
git clone https://github.com/immunefi-team/audit-comp-ens
cd audit-comp-ens && git checkout audit-comp-ready && pnpm install --frozen-lockfile
cd ..
```

Download the three PoC files from the linked secret Gist into this directory, so they sit alongside the `audit-comp-ens/` checkout, then make the shell script executable:

```sh
chmod +x poc_renewal_desync_onchain.sh
```

All commands below are run from this directory unless stated otherwise.

{% stepper %}
{% step %}

## The two derivations disagree, using the app's own code

`poc_renewal_desync.mjs` transpiles the shipped `parseName` from `apps/manager/src/features/register-v2/utils/name-parser.ts` and runs it beside `normalize` resolved from the repository's own `node_modules`. No network access.

```sh
node poc_renewal_desync.mjs audit-comp-ens
```

It runs five payloads. Observed for the first:

```
URL parameter      : "ali​ce.eth"
  codepoints       : 0061 006c 0069 200b 0063 0065 002e 0065 0074 0068

GATE  routes/renew/$name.tsx:26 -> profileExpiry.ts:77 -> profileName.ts:18
  normalize(param) : "alice.eth"  [ 0061 006c 0069 0063 0065 002e 0065 0074 0068 ]
  -> eligibility, protocol, grace window and expiry all read from THIS name

SINK  routes/renew/$name.tsx:61 -> name-parser.ts:46 (toLowerCase)
  parseName label  : "ali​ce"  [ 0061 006c 0069 200b 0063 0065 ]
  -> registration.actors.ts:1072 encodeRenewData(label) -> renew(label,...)

RESULT
  gate label       : "alice"  labelhash 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
  renewed label    : "ali​ce" labelhash 0xa42ad15f2e51ecf4f2a50035abe21ddf90001dd52e1f2915c5128d9b2cfda580
  SAME NAME?       : NO  <-- the route validated one name and renews another
```

{% endstep %}

{% step %}

## The shipped route loader returns the mismatched pair

Copy `poc_renewal_desync.test.ts` to `audit-comp-ens/apps/manager/src/routes/renew/desync.poc.test.ts`, then run from `audit-comp-ens/apps/manager`:

```sh
pnpm exec vitest run src/routes/renew/desync.poc.test.ts --reporter=verbose
```

The test imports `Route` from `./$name` and invokes `Route.options.loader` with the attacker's URL parameter and a `queryClient` whose `ensureQueryData` returns the victim's real v2 expiry, which is what the gate reads. Nothing is reimplemented; the assertions run against the shipped loader. Observed:

```
  route param   : 0061 006c 0069 200b 0063 0065 002e 0065 0074 0068
  normalize()   : 0061 006c 0069 0063 0065 002e 0065 0074 0068   <- name the GATE resolved
  loader label  : 0061 006c 0069 200b 0063 0065                  <- label that is RENEWED
  currentExpiry : 1793272764                                     <- victim's expiry, handed to the page

 ✓ 'zero-width space, plain ASCII victim'
 ✓ 'soft hyphen, plain ASCII victim'
 ✓ 'NFD, composable character'

 Test Files  1 passed (1)
      Tests  3 passed (3)
```

The loader returns the victim's expiry together with a label that is not the victim's name.
{% endstep %}

{% step %}

## End to end against the deployed registrar, on a local fork

Start a local anvil fork of Sepolia. All writes are local, the fork only reads chain state, and nothing is broadcast to the public network.

```sh
anvil --fork-url <sepolia-rpc> --port 8547   # leave running in a second terminal
./poc_renewal_desync_onchain.sh
```

The script:

1. Prepares two EOAs as the connected wallets and two ERC-1155 receiver addresses as the name owners, matching the app where the owner is the user's HCA and the connected EOA pays (`useBulkRenewSubmit.ts`).
2. Registers `ali<U+200B>cewrsec` to the attacker through the real `ETHRegistrar` commit and reveal, which is the label `parseName` produces.
3. Registers the plain ASCII `alicewrsec` to the victim, which is the name `normalize` produces and the name the route validates.
4. Sends, from the victim's connected EOA, the exact calldata the app builds: `renew(<raw lookalike label>, 31536000, USDC, referrer)`.
5. Reads both expiries and the victim's USDC balance before and after.

Both labels are 5 or more code points, so they price in the same tier and price is not a variable. Observed, on a fork at Sepolia block 11598471:

```
victim's name   alicewrsec (plain ASCII, 10 code points)
                bytes 616c6963657772736563        labelhash 0x36cbe3cf...
attacker's name ali<U+200B>cewrsec                labelhash 0xe6f96b02...

attacker lookalike label expiry : 1819624801   ->  1851160801   (+1 year)
victim   real name       expiry : 1819624891   ->  1819624891   (unchanged)
victim   USDC balance           : 99991999979  ->  99983999958  (paid 8000021)

PASS
The victim paid 8000021 USDC units to renew a name they do not own.
The extension landed on the attacker's identical-looking alicewrsec.eth.
The victim's own alicewrsec.eth was not renewed.
```

`PAYLOAD=nfd ./poc_renewal_desync_onchain.sh` runs the same proof with the Unicode normalization form payload instead of the zero-width one.
{% endstep %}
{% endstepper %}

## The attack as a user experiences it

{% stepper %}
{% step %}

## Alice owns `alice.eth`

Alice owns `alice.eth` and it is inside its renewal window.
{% endstep %}

{% step %}

## The attacker registers the lookalike

The attacker registers `ali<U+200B>ce` through the Manager at `/register/`.
{% endstep %}

{% step %}

## Alice opens the renewal link

Alice opens `/renew/ali<U+200B>ce.eth`. The URL and the page render identically to her own name.
{% endstep %}

{% step %}

## The loader validates Alice's name

The loader normalizes the parameter to `alice.eth`, confirms it is v2, in-window and renewable, and reads its expiry.
{% endstep %}

{% step %}

## Alice selects a renewal period

The duration selector offers renewal periods measured from Alice's genuine expiry date. She picks one and confirms.
{% endstep %}

{% step %}

## The wallet signs renewal calldata for the raw label

The wallet is asked to sign `renew(<raw label>, duration, USDC, referrer)`. It does not revert, because renewal requires no authority over the name being renewed.
{% endstep %}

{% step %}

## Alice is charged

Alice is charged. The attacker's name is extended by the full duration. Alice's `alice.eth` is unchanged.
{% endstep %}

{% step %}

## The app displays a false successful renewal

The app shows "Renewal Complete!" and an "Expires" date equal to Alice's real expiry plus the duration she paid for.
{% endstep %}
{% endstepper %}
