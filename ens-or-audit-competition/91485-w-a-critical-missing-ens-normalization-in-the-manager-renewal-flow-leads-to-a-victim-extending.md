# 91485 w a critical missing ens normalization in the manager renewal flow leads to a victim extending the attacker s name with their own funds

**Submitted on Sep 1st 2026 at 22:18:19 UTC by @velikoros for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91485
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

### Brief/Intro

The Manager app's renewal flow normalizes the name it *reads* (owner check, expiry, display — ENSIP-15) but not the name it *writes*: the label is carried from the URL into `renew()` calldata with only `trim().toLowerCase()`. Because ENSIP-15 normalization silently strips invisible characters (zero-width space, soft hyphen, word joiner) rather than rejecting them, an attacker can register an invisible twin of any victim's name (e.g. `alicepoc` + U+200B — same price, distinct on-chain name).

When the victim opens `/<twin>.eth`, the app renders the victim's own profile — their owner badge, their real expiry, no third-party warning (the ownership check resolves to the victim's address) — and shows their expiry "extended" after renewal, while the transaction it builds renews the attacker's name.

Exploited in production, every deceived user pays real rent that extends the attacker's registration; the user's own name receives nothing and later expires. Validated end-to-end on a fork of live Sepolia: the victim's wallet submitted the exact calldata the app builds and the attacker's name was extended by exactly one year at the victim's expense.

### Vulnerability Details

**The write path never normalizes.** The renewal path parses names with `apps/manager/src/features/register-v2/utils/name-parser.ts` — the entire normalization is one line (line 46), and the only character gate is an ASCII punctuation blocklist (line 11) that zero-width/invisible characters pass:

```ts
const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/   // :11
...
const normalized = name.trim().toLowerCase()                     // :46 — that is all
```

The raw label then flows unchanged into on-chain calldata:

```ts
// apps/manager/src/routes/renew/$name.tsx:20,27,61 — loader returns the RAW label
const parsedName = parseRenewableName(name)            // keeps raw label
const expiryData = await queryClient.ensureQueryData(profileExpiryQuery(name)) // normalized read
return { label: parsedName.value.label, currentExpiry: expiryData.expiry }

// apps/manager/src/features/renew/state/renewalUi.machine.ts:349
input: ({ context }) => ({ label: submission.label, ... })

// packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1031
function encodeRenewData(label: string, duration: bigint, paymentToken: Address): Hash {
  const cleanLabel = label.replace('.eth', '')         // no ENSIP-15 anywhere
  return encodeFunctionData({ abi: ethRegistrarRenewSnippet, functionName: 'renew',
    args: [cleanLabel, duration, paymentToken, REFERER_ADDRESS] })
}
```

**The read path normalizes — and diverges silently.** Every read (`features/profile/service/profileName.ts:18` → `profileOwner.ts:25`, `profileExpiry.ts:77`) runs the same input through `normalize()` (viem/ens → `@adraffy/ens-normalize`, pinned 1.11.1). Executed directly against the pinned version, normalization does not reject the invisible characters — it silently strips them:

| Input label (UTF-8 hex)                   | `ens_normalize(label + '.eth')` |
| ----------------------------------------- | ------------------------------- |
| `616c696365706f63` (alicepoc)             | `alicepoc.eth` (unchanged)      |
| `616c696365706f63e2808b` (+U+200B suffix) | `alicepoc.eth` — **stripped**   |
| `616c696365e2808b706f63` (+U+200B mid)    | `alicepoc.eth` — **stripped**   |
| `616c696365706f63c2ad` (+U+00AD)          | `alicepoc.eth` — **stripped**   |
| `616c696365706f63e281a0` (+U+2060)        | `alicepoc.eth` — **stripped**   |

So for the twin URL the ownership check resolves to the victim's address — `ProfileRenewAction.tsx:66` shows the third-party-renewal warning only when `isOwner === false` (`thirdPartyRenewalWarning.ts:3`), so for the twin it is **suppressed** — while the twin is a genuinely distinct on-chain name:

```
keccak("alicepoc")        = 0xc172cbae6c49977ebf107bde5be6a78d4b7add1a707fda7022e306eb5a0ba9bd
keccak("alicepoc"+U+200B) = 0xfa3b434bc69bd065231a5c9ce2ce0f2c3ae5c7a289fe95a582bf4c0070a811e9
→ distinct namehash, distinct tokenId, independently registrable
```

**The twin is registrable at the normal price.** The registrar treats labels as opaque bytes — re-fired on live Sepolia against the app's own contract constants (`ENS_SEPOLIA_CONTRACTS.ETHRegistrar` = `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`, USDC = `0x768F42455A2D082E23ceeF7d51e5787C82d67a39`):

```
getRegisterPrice("alicepoc", 31536000, USDC)         -> 8000021, 0
getRegisterPrice("alicepoc"+U+200B, 31536000, USDC)  -> 8000021, 0   (all 5 variants identical)
```

The app's availability check skips ENS normalization as well (`availability.query.ts:9` → `utils/domain.ts:27-33` — lowercase + `.eth` suffix only), so the twin reports "available" inside the app.

**The display completes the deception.** `PricingDomainHeader.tsx:9,29` renders the raw `${label}.eth` (pixel-identical to the victim's name — U+200B occupies no space), and `PricingSummaryCard.tsx:12-22` shows the expiry derived from the *normalized* name's real on-chain expiry, extended by the chosen duration. Every element on both screens describes the victim's name; only the calldata encodes the attacker's.

### Impact Details

Selected impact: **"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient"** — this finding is that row executed end-to-end: the app displays name A (the victim's) and submits a transaction whose actual target is name B (the attacker's). It also falls under the program's stated top concern, "anything that lets a transaction be built … with the wrong … **arguments**" (the `renew()` label argument is wrong relative to everything the app displayed).

Demonstrated loss (fork of live Sepolia, two wallets — see PoC): the victim's wallet paid **8.000021 USDC** of rent and the **attacker's** name was extended by exactly one year (+31,536,000 s on-chain); the victim's own name was untouched.

Breakdown of production impact:

* **Direct funds loss per victim, repeatable.** The rent leaves the victim and benefits the attacker (their registration is extended for free). Each deceived visitor pays a further renewal; the attacker's cost is one twin registration at the same price the app quotes for any label (no premium).
* **The victim's own name later expires.** The user believed they renewed it; the renewal protected nothing of theirs.
* **The safety control is defeated by the same defect.** The third-party-renewal warning is suppressed because the ownership check runs on the normalized name — the app affirmatively tells the victim "this is your name".
* **Works against any name.** The attacker only registers the twin of a target label; the victim is anyone who opens that URL (link, search, paste) while connected.
* **No privileges, no special state, normal user actions only** — open profile → Renew → approve.

Dedup note. EXP-INPUT-005 covers validators accepting inputs that ENS normalization "rejects or transforms … causing homograph-style **display confusion** in the UI **rather than incorrect resolution**." This finding is not display confusion: it is wrong-name transaction construction with a funds path and a suppressed safety control. Per the program's own known-issues preamble, "new consequences of a listed root cause that materially change its severity" are in scope as new findings — this moves the class from display confusion (Medium) to the Critical impact row quoted above.

### References

* Write path: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager/src/features/register-v2/utils/name-parser.ts>
* Renew route loader (raw label out): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager/src/routes/renew/$name.tsx>
* Renewal machine (label pass-through): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager/src/features/renew/state/renewalUi.machine.ts>
* Calldata construction: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager/src/machines/registration/registration.actors.ts>
* Read-path normalization: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager/src/features/profile/service/profileName.ts>
* Suppressed warning: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx>
* ENSIP-15 (normalization spec — silent-strip "ignored" characters): <https://docs.ens.domains/ensip/15>
* @adraffy/ens-normalize (pinned implementation): <https://github.com/adraffy/ens-normalize.js>

## Link to Proof of Concept

<https://gist.github.com/yoptascript/ca17e6d3effbcc6ffb962aa495e991f5>

## Proof of Concept

Step-by-step below; the runnable script (`poc-F1.sh`) and the full captured run log (`F1-poc-fork-run.txt`) are in the secret Gist:\
<https://gist.github.com/yoptascript/ca17e6d3effbcc6ffb962aa495e991f5>

Prereqs: foundry (anvil + cast), openssl, jq. The contracts interacted with are the app's actual Sepolia deployments; all funds are fork-only.

### Setup

{% stepper %}
{% step %}

## Start a fork of live Sepolia and run the script

```bash
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --port 8546
bash poc-F1.sh        # env F overrides the fork URL
```

Two fresh code-less EOAs are used: attacker `0x0f6B…2C58`, victim `0xE48C…9EDc`.
{% endstep %}

{% step %}

## Fund both wallets

The script funds both wallets (fork-only gas + USDC seeded from a live-funded account).

Observed:

```
USDC attacker: 10000000  victim: 50000000
```

{% endstep %}

{% step %}

## Attacker registers the invisible twin

The attacker registers `alicepoc`+U+200B via the registrar's normal flow (`makeCommitment` → `commit` → 60 s → `register`) — the same flow the app drives.

Observed: both txs `status:0x1` (register tx `0x514a…`).
{% endstep %}

{% step %}

## Victim registers `alicepoc`

The victim registers `alicepoc` the same way.

Observed: `status:0x1` (tx `0x9d5f…`).
{% endstep %}

{% step %}

## Verify the baseline before the deceived renewal

On-chain `registry.getExpiry`:

```
expiry(twin 'alicepoc'+ZWSP)  = 1819833261   <- attacker's name
expiry(plain 'alicepoc')      = 1819833323   <- victim's name
USDC victim: 41999979  attacker: 1999979
```

{% endstep %}

{% step %}

## Victim submits the renewal with the exact calldata the Manager app builds

Captured from `encodeRenewData`/`submitRenewActor`; note the string argument — length 9 — contains the zero-width-space bytes `e2808b`; duration `31536000` = 1 year; referrer = zeroHash, exactly as the app passes:

```
0x89d779c30000000000000000000000000000000000000000000000000000000000000080
00000000000000000000000000000000000000000000000000000000001e13380
000000000000000000000000768f42455a2d082e23ceef7d51e5787c82d67a39
0000000000000000000000000000000000000000000000000000000000000000
00000000000000000000000000000000000000000000000000000000000000
0b616c696365706f63e2808b
```

Observed: `tx 0xd6f6… status:0x1`.
{% endstep %}

{% step %}

## Verify the result

The renewal the app built from the twin URL extended the **attacker's** name with the **victim's** USDC:

```
expiry(twin 'alicepoc'+ZWSP)  = 1851369261  (before 1819833261)  EXTENDED by 31536000s
expiry(plain 'alicepoc')      = 1819833323  (before 1819833323)  UNCHANGED
USDC victim:   33999958  (before 41999979 — paid 8000021 units = 8.000021 USDC)
USDC attacker: 1999979   (before 1999979 — paid NOTHING)
```

{% endstep %}

{% step %}

## Review what the victim saw on screen

From the code path, pinned commit `1c9b47f`, the profile at `/renew/alicepoc%E2%80%8B.eth` renders `alicepoc.eth` — the victim's own owner badge (ownership check resolved via the normalized name), the victim's real expiry, no third-party warning — and `PricingSummaryCard` displays the victim's expiry extended by one year, while the transaction encoded `alicepoc`+U+200B.
{% endstep %}
{% endstepper %}

**Expected:** the app refuses names that are not already ENS-normalized (or encodes the same normalized name it displayed).

**Actual:** the raw URL label is encoded into `renew()` calldata; the screen validates, displays and ownership-checks a different name than the one renewed.

Gist (secret) with the runnable script and the full captured run:\
<https://gist.github.com/yoptascript/ca17e6d3effbcc6ffb962aa495e991f5>
