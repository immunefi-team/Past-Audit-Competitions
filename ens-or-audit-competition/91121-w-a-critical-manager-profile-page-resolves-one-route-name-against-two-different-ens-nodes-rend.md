# 91121 w a critical manager profile page resolves one route name against two different ens nodes rendering an attacker controlled receiving address beside the victim s genuine ownership data

**Submitted on Aug 29th 2026 at 21:41:20 UTC by @archcypher for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91121
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager's `/$name` page issues two on-chain lookups for the **same** route parameter under **two different canonicalisation rules** and renders both results on one page: the owner and expiry come from the ENSIP-15-**normalised** name, while the address records come from the **raw** route parameter.

An attacker registers a decoy 2LD whose raw label normalises to a victim's real label (`POCTESTENS` → `poctestens`), sets `addr(60)` on it to an address they control, and the profile page then presents the **victim's genuine ownership block beside the attacker's receiving address** — inside a copy button that puts the attacker's full, untruncated address on the user's clipboard.

In production, a user who copies the "Main receiving address" from what looks like a legitimate, correctly-owned ENS profile — the normal use of that control — is holding the attacker's address, and any payment they then make on the strength of it goes to the attacker.

## Vulnerability Details

One route parameter, two canonicalisations.

**Normalised — the identity panel**

```ts
// apps/manager/src/features/profile/service/profileOwner.ts:25
const ethName = normalizeEthName(params.name)      // profileName.ts:18 -> viem normalize() (ENSIP-15)
// …:28-51 is the non-.eth / v1 branch, elided…
// :55
const v2Owner = yield* fromPromise(ensjsv2_getOwner(client, { name: ethName.name }), …)
```

`profileExpiry.ts:77` and `profileRegistration.ts:66` normalise the same way, via `normalizeEth2LdName` (`profileName.ts:62`), which is a thin wrapper over the same `normalizeEthName`.

**Raw — the address records**

```ts
// apps/manager/src/features/profile/service/profileRecords.ts:122
const records = yield* fromPromise(
  getRecords(client, { name, texts, coins, contentHash: true, abi: true, ignoreInvalidCoinTypes: true }), …)
//                     ^^^^ the raw route parameter, never normalised
```

`ProfileView.tsx:94` and `:100` call `profileRecordsQuery(name)` and `profileOwnerQuery(name)` with the same `name` and render both.

`ProfileAddressesSection.tsx:90-94` puts the coinType-60 record inside a `CopyableButton` whose `value` is the **full** address. The visible text is truncated at `:42`, but the copy value is not.

`ProfileDetails.tsx:121` renders the Owner row as `ownerReverseName || truncateAddress(owner)` — i.e. the victim's own primary ENS name when they have one.

`labelhash("POCTESTENS") = 0xc1e3293b…` and `labelhash("poctestens") = 0xf779601d…` are different names with different owners; the ENSv2 registrar does not enforce ENSIP-15 canonicalisation, and accepted both raw labels tested here, so the decoy is registrable.

### Nothing in the Manager stops it

1. `routes/$name/route.tsx:6` — the only route-parameter validation is `name.length < 3 || !name.includes('.')`.
2. The loader's `parseName` (`features/register-v2/utils/name-parser.ts:46`) only `.trim().toLowerCase()`s, and its `INVALID_LABEL_CHARS` regex contains no letters and not U+00AD, so both decoys parse with `tld === 'eth'` and the `unsupported-tld` branch never fires.
3. That leaves one `NameFallbackCard` path, the `!ownerData` branch (`routes/$name/index.tsx:77`, rendered at `:171`; the other producer, `'unsupported-tld'` at `:72`, is the one bullet 2 just ruled out) — and the owner lookup is the *normalised* one, so it always resolves and the page renders.

The team applies the opposite rule elsewhere, which is why this reads as an oversight rather than a design choice:

```ts
features/dashboard/components/NamesTable.tsx:84-86
```

It deliberately picks `domain.normalizedName` over the display name, commented:

> "Use the canonical normalized name for the on-chain label — the display name may be unnormalized (e.g. mixed case) and would hash to the wrong label."

This is not a defect in `@ensdomains/ensjs`: `getOwner` and `getRecords` each hash exactly the name they are given. The defect is that `apps/manager` hands one route parameter to two lookups under two different canonicalisations.

### Two decoy variants

`POCTESTENS.eth` is plain ASCII — no homoglyph, no bidi, no control character — and is reached by any link, bookmark, QR or capitalised URL. ENS names are routinely written capitalised in prose, banners and headlines.

`poctest{U+00AD}ens.eth` renders as nothing and additionally survives the Manager's own header search (`searchSuggestions.utils.ts:54` only does `.trim().toLowerCase()`), giving an entry point inside the app.

The finding does not depend on the second — the uppercase variant carries the whole result.

## Impact Details

**Selected impact — 6285, Critical:**

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
>
> * resolving or rendering an ENS name to an attacker-controlled address
> * showing the wrong recipient or amount on a transaction the user then approves

The first of those two bullets is the one this report demonstrates.

The page renders, on one screen:

```
[ POCTESTENS.eth ]
Owner  poctestens.eth      Expires  AUGUST 28, 2027
Addresses
  Main receiving address
  POCTESTENS.eth   0x7099...79C8     [copy]
```

`POCTESTENS.eth` was registered for **three years**; the victim's `poctestens.eth` for **one**. The page shows the victim's date, because expiry is read through the normalised name (`profileExpiry.ts:77`).

The PoC asserts that equality directly — the displayed expiry *equals* the genuine name's and the decoy's own differs — rather than asserting any particular calendar date, which moves with the fork head.

`Owner poctestens.eth` is the **victim's** real primary name, resolved from the victim's address, and the `Expires` row is the victim's expiry instant. The decoy was deliberately registered for a different term, so `name-split-poc.out` can show the page displaying the victim's date rather than the decoy's own.

The whole ownership block is read through the normalised name and is genuinely theirs, while `0x7099…79C8` is the **attacker's** address.

To be precise about which half is which:

* The badge
* The page `<title>` (`routes/$name/index.tsx:137`)
* The avatar (`ProfileView.tsx:139`)
* The name printed inside the address card (`ProfileAddressesSection.tsx:116`)

All render the **raw** attacker string. That is the same split, seen from the other side, and it is what makes the page incoherent rather than merely wrong.

Whichever name you consider the page to belong to, one half is false:

* Read as the attacker's name, the app attributes the victim's legitimate ownership to it.
* Read as the victim's name, the app renders an attacker-controlled receiving address for it.

### Clipboard impact

The harm is not merely that a wrong string is displayed.

Clicking the Copy control — the normal use of that control, which exists for exactly this purpose — places the attacker's **full, untruncated** address on the user's clipboard.

This is asserted deterministically. The following is the complete, verbatim stdout of `f1-ui-capture.mjs`, supplied as `F1-clipboard-assertion.out` in the linked gist:

```
PASS  /poctestens.eth  clipboard=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266  expected=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
PASS  /POCTESTENS.eth  clipboard=0x70997970C51812dc3A010C7d01b50e0d17dc79C8  expected=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
PASS  /poctest%C2%ADens.eth  clipboard=0x70997970C51812dc3A010C7d01b50e0d17dc79C8  expected=0x70997970C51812dc3A010C7d01b50e0d17dc79C8

All clipboard assertions passed: the decoy pages copy the ATTACKER address,
the genuine page copies the victim address.
```

`0xf39F…2266` is the victim; `0x7099…79C8` is the attacker.

### Loss

Any funds a user sends to the "Main receiving address" of what the application presents as a legitimate, correctly-owned ENS profile.

Attacker cost is a single registration and gas. The attacker is unprivileged — no leaked credentials and no privileged role — and the user's own step is the normal workflow of the page: viewing an ENS profile and copying its receiving address.

The uppercase variant needs no invisible or homoglyph character and no attacker-authored URL. ENS names are routinely written capitalised, so any link, bookmark or QR code that preserves that capitalisation — including one written in good faith by a third party — lands the user on the decoy.

The Manager's own header search lowercases its input at `searchSuggestions.utils.ts:54`, so the in-app search entry point belongs to the soft-hyphen variant, not the uppercase variant.

{% hint style="info" %}
The PoC proves that the attacker's address is rendered and copied through the application's own control; it does not execute a downstream payment. The claim is a **payment-redirection path when a user relies on the page's receiving-address control**, not an executed theft — and deliberately not a claim under the separate "Direct theft of user funds" impact.
{% endhint %}

### Relationship to the published Known Issues

The nearest entry is **EXP-INPUT-005**:

> "Our name validators accept inputs that ENS normalization later rejects or transforms, including homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot. The practical impact is homograph-style display confusion in our UI rather than incorrect resolution."

Even if triage considers acceptance of non-normalised input part of the same broad family, that entry **expressly limits its practical impact to display confusion rather than incorrect resolution**, and the known-issues document **expressly keeps in scope**:

> "new consequences of a listed root cause that materially change its severity"

This report demonstrates that higher consequence: the application renders an attacker-controlled receiving address alongside the victim's genuine ownership data, and hands that address to the user's clipboard.

### Scope of this finding

This defect is confined to `apps/manager`'s profile route — a read-and-render path. It is not the Explorer's transfer flow, and a fix there would leave this one live.

* **Different impact.** The path demonstrated here signs nothing: the application renders an address and the user copies it. The harm is a rendered and copied receiving address — impact **6285**.
* **Different asset, and therefore a different remedy.** `apps/manager` and `apps/portal` are separately enumerated in-scope assets and separate workspace packages that do not import one another. They share generic utility packages, but none performs ENS name normalisation — `normalizeEthName` exists only under `apps/manager`, `getLabel.ts` only under `apps/portal`, each pulling its normaliser straight from `viem` / `@adraffy/ens-normalize`. There is no shared module a single fix could land in; the remedy here is in the Manager's profile services, or a route-level canonicalisation in `apps/manager`.
* **The harm flows through the raw side.** It is the **raw** route parameter that supplies the dangerous value — the `addr(60)` record that is rendered and copied — while the identity block (Owner, Expires) is the half read through the **normalised** name. The page displays the raw name as its subject either way, so what is *shown* is not the distinguishing feature; which lookup the *harm* flows through is.

Also checked and distinct:

* **WEB-1301:** Address view shows owner/expiry from the wallet's primary name — different route, different data source, no attacker-controlled address.
* **QA-06:** Deleted subname still shown as primary.
* Possible prior art: `ensdomains/ens-app#1346`, filed 2021-11-15 and **closed**, is a normalisation exception on the address page of the **legacy ENS manager app** — a single-lookup failure, materially different from a dual-canonicalisation split that renders an attacker-controlled address beside genuine ownership data.

## Recommended fix

Canonicalise the route parameter once, before any lookup:

* Redirect `/$name` to `normalize(name)` when they differ, or
* Pass the normalised name to `profileRecords` as `profileOwner`, `profileExpiry`, and `profileRegistration` already do.

A name that fails `normalize()` should render the existing fallback rather than a spliced page.

## References

Line numbers are from `immunefi-team/audit-comp-ens` at `cda79acaad59711b943fc68207ebb3f1d0ff8596` ("fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86").

`apps/manager/src` is byte-identical at branch `audit-comp-ready` HEAD `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, so every citation resolves at either commit.

* `apps/manager/src/features/profile/service/profileOwner.ts` — L25, L55, L56
* `apps/manager/src/features/profile/service/profileName.ts` — L14-40 (`normalizeEthName`, `normalize()` at L18); L62 (`normalizeEth2LdName`)
* `apps/manager/src/features/profile/service/profileRecords.ts` — L122
* `apps/manager/src/features/profile/service/profileExpiry.ts` — L77
* `apps/manager/src/features/profile/service/profileRegistration.ts` — L66
* `apps/manager/src/features/profile/components/view/ProfileView.tsx` — L94, L100, L139 (avatar from the raw name)
* `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx` — L90-94 (copy value), L42 (truncated display), L116 (raw name inside the card)
* `apps/manager/src/features/profile/components/view/ProfileDetails.tsx` — L121, L124 (the conditional `Registered` row)
* `apps/manager/src/routes/$name/route.tsx` — L6
* `apps/manager/src/routes/$name/index.tsx` — L72 (`'unsupported-tld'` producer), L77 (`!ownerData` guard), L137 (page title from the raw name), L171 (fallback render)
* `apps/manager/src/features/register-v2/utils/name-parser.ts` — L46
* `apps/manager/src/features/navigation/Header/search/searchSuggestions.utils.ts` — L54
* `apps/manager/src/features/dashboard/components/NamesTable.tsx` — L84-86 (the comment showing the team's own rule)

Also live in the deployed beta: `https://app.ens.dev/assets/index-yq2Vvs65.js` (sha256 `126defada7273daca3faeacd65d93927f6a82f5ee948f83d2e4edf3bb5f3f815`, fetched 2026-08-28) still contains the compiled `profileRecords` service `$ae=fa(async function*(e){ … JA(t,{name:e, texts:…, coins:…, contentHash:!0, abi:!0, ignoreInvalidCoinTypes:!0}) … })` — the generator's raw `e` parameter reaches `getRecords` with no normalisation in between.

ENSIP-15 (ENS Name Normalization): <https://docs.ens.domains/ensip/15>

***

## Link to Proof of Concept

<https://gist.github.com/archcypher/5012cc1ba0d80eea28ecf7cf7ff46986>

## Proof of Concept

Everything below runs against a **local anvil fork of Sepolia**. No transaction was broadcast to any live network, and no live host was probed, tested or attacked.

Being exact about what *did* leave the machine, since some of it is visible in the frames: the fork sources its state from a public Sepolia endpoint over read-only RPC, which is what forking means. The browser leg additionally makes the app's own ordinary outbound requests, because it runs the Manager unmodified — the ENS metadata avatar endpoint (`ProfileView.tsx:139`, which has no record of a fork-only name, which is why the frames show the pattern-avatar fallback rather than an image), Google Fonts, and the bundled WalletConnect/Intercom widgets.

All read-only; none of it exercises a host under test.

{% stepper %}
{% step %}

## Register the genuine name

The victim owns `poctestens.eth` with `addr(60)` set to their own address.
{% endstep %}

{% step %}

## Register the attacker decoy

The attacker registers `POCTESTENS.eth` through the same ENSv2 `ETHRegistrar` — a distinct on-chain name, `labelhash 0xc1e3293b…` versus the genuine `0xf779601d…` — and sets its `addr(60)` to an address they control.

Optionally, they also copy the victim's avatar, description and links so the page looks right.
{% endstep %}

{% step %}

## Open the decoy route

A user who intends to pay the owner of `poctestens.eth` opens `https://<manager>/POCTESTENS.eth` — from a link, a bookmark, a QR code, or a name written capitalised anywhere.
{% endstep %}

{% step %}

## Route parsing succeeds

The route parse passes:

```
name.length >= 3 && name.includes('.')
```

The owner lookup normalises to `poctestens.eth` and returns the genuine owner, so the "name not found" fallback never fires.
{% endstep %}

{% step %}

## View the mixed profile data

The page renders with the badge `POCTESTENS.eth`, **Owner `poctestens.eth`** — the victim's real primary name — and the victim's real Expires date, not the decoy's, which is two years later.
{% endstep %}

{% step %}

## View the attacker receiving address

Under **Addresses → Main receiving address**, the page shows the attacker's `0x7099…79C8`.
{% endstep %}

{% step %}

## Copy the attacker address

The user clicks **Copy** — the normal use of that control — and the attacker's full, untruncated address is placed on their clipboard.
{% endstep %}
{% endstepper %}

### Running it

```bash
anvil --fork-url <sepolia-rpc> --chain-id 11155111

# the script imports the app's own ensjs/viem, so it must run from inside that
# workspace package for pnpm to resolve them:
cp name-split-poc.mjs <repo>/apps/manager/
cd <repo>/apps/manager && node name-split-poc.mjs
# POC_RPC overrides the RPC URL.
```

`name-split-poc.mjs` registers the genuine label and both decoy labels through the real Sepolia `ETHRegistrar` deployment as forked — proving a non-normalised label is registrable — then exercises the app's two service-layer lookups.

`normalizeEthName` in the script is a copy of `profileName.ts:14-40` with the TypeScript annotations stripped and the logic unchanged, and the lookups are the same ensjs `getOwner` v2 and `getRecords` the services call.

Its complete, verbatim output — `name-split-poc.out` in the gist:

```
victim   : 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
attacker : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

=== (A) PRECONDITION: register the real name and the decoys ===
  poctestens.eth        registered by victim   for 1 year  (resolver 0x620f4f7684433ab07eb35e7ddeed74a8e50a83c6)
  "POCTESTENS".eth  registered by ATTACKER for 3 years — labelhash 0xc1e3293b7f… (real: 0xf779601d10…)
  "poctest­ens".eth  registered by ATTACKER for 3 years — labelhash 0xbe0acc3ddc… (real: 0xf779601d10…)

=== (B) IMPACT: the Manager profile page, per route param ===

  route param "poctestens.eth"
    normalize()               -> "poctestens.eth"
    "Owner"            (norm) -> 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266  <- VICTIM
    "Expires"          (norm) -> 2027-08-29
    "Main receiving address"  -> 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266  <- VICTIM

  route param "POCTESTENS.eth"
    normalize()               -> "poctestens.eth"
    "Owner"            (norm) -> 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266  <- VICTIM
    "Expires"          (norm) -> 2027-08-29   <- the VICTIM's; this name's own expiry is 2029-08-28
    "Main receiving address"  -> 0x70997970C51812dc3A010C7d01b50e0d17dc79C8  <- ATTACKER
    *** SPLIT: genuine owner shown beside attacker address ***

  route param "poctest­ens.eth"
    normalize()               -> "poctestens.eth"
    "Owner"            (norm) -> 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266  <- VICTIM
    "Expires"          (norm) -> 2027-08-29   <- the VICTIM's; this name's own expiry is 2029-08-28
    "Main receiving address"  -> 0x70997970C51812dc3A010C7d01b50e0d17dc79C8  <- ATTACKER
    *** SPLIT: genuine owner shown beside attacker address ***

=== assertions ===
  PASS  genuine "poctestens.eth" shows NO split
  PASS  genuine "poctestens.eth" displays its own expiry
  PASS  decoy "POCTESTENS.eth" shows the split
  PASS  decoy "POCTESTENS.eth" displays the VICTIM's expiry (equal), and its own differs
  PASS  decoy "poctest­ens.eth" shows the split
  PASS  decoy "poctest­ens.eth" displays the VICTIM's expiry (equal), and its own differs

CONFIRMED: both decoys render the victim as Owner AND the victim's expiry
beside the ATTACKER's receiving address; the genuine name renders the victim
on both halves.

The calendar dates above move with the fork head — expiry is block.timestamp +
duration. The asserted invariant is the EQUALITY above: the decoy page shows the
genuine name's expiry, and the decoy's own, distinct expiry never reaches it.
```

`register-raw-label.sh` corroborates the on-chain half of the precondition with `cast` alone — no node, no ensjs — on its own fresh fork: that the registrar **accepts both case-distinct raw labels and assigns them to different owners**.

It does not itself run ENSIP-15; the normalisation collision is executed by the canonical Node PoC above.

It reads ownership through the registry's own `getTokenId(uint256 labelhash)` — the ERC-1155 id is *versioned*; `uint256(labelhash)` is not it — asserts both owners, and exits non-zero on failure.

Its recorded run is `register-raw-label.out`:

```
== ASSERTIONS
  PASS  poctestens is owned by the victim
  PASS  POCTESTENS is owned by the ATTACKER
  PASS  the two labels have DIFFERENT owners
  PASS  the two labels are DIFFERENT on-chain names
```

### The rendered page and clipboard

`f1-ui-capture.mjs` (Playwright) drives the Manager against the fork, captures each page, then clicks the **Main receiving address** control and reads `navigator.clipboard.readText()` back.

It exits non-zero if any assertion fails.

All six captures and all three assertions come from a **single seeding and a single run**. `F1-capture-manifest.txt` in the gist records the sha256 of every frame that invocation wrote, so a reviewer can confirm the attached PNGs are the recorded artifact set rather than a later or mixed one.

A checksum manifest binds the attachments to that set; it is not by itself independent proof of provenance.

Setup is in `REPRO-F1.txt` — no Docker and none of the e2e stack's services, though the capture script is run from `<repo>/e2e` to borrow that workspace's Playwright install. The captures are uploaded via the **Attachments** field.

`F1-ui-03-crop-uppercase-decoy.png` is the frame to look at — `/POCTESTENS.eth`:

```
[ POCTESTENS.eth ]
Owner  poctestens.eth      Expires  AUGUST 28, 2027
Addresses
  Main receiving address
  POCTESTENS.eth   0x7099...79C8     [copy]
```

`F1-ui-00-baseline-genuine.png` is the genuine page for comparison, and `F1-ui-02-softhyphen-decoy.png` is the invisible-codepoint variant, whose badge is visually identical to the baseline.

{% hint style="info" %}
The **Registered** row is absent from the frames — a harness artifact, not part of the finding.

`ProfileDetails.tsx:124` renders that row only when a registration date resolves, so it is omitted rather than shown empty. The date comes from `getRegistrationDate`, which issues `eth_getLogs` with `fromBlock: 0n`; anvil forwards pre-fork log queries to the upstream endpoint, but the public Sepolia provider rejects a whole-chain range with `{"code":-32701,"message":"exceed maximum block range: 50000"}`, so no registration date is returned.

It is an upstream RPC range limit — not the stub indexer, and not anvil. The two rows this report relies on, Owner and Expires, are point reads and resolve.
{% endhint %}

The frame reads `AUGUST 28, 2027` where `name-split-poc.out` prints `2027-08-29`: the same instant (epoch `1819518458` = `2027-08-29T05:47:38Z`), with the PoC printing the UTC day and the Manager rendering it in the capture machine's local timezone (`America/Los_Angeles`).

The calendar dates are not the claim. Expiry is `block.timestamp + duration`, so every date above moves with the fork head and will differ on your re-run.

The invariant the PoC asserts is the equality: the decoy page displays the genuine name's expiry, and the decoy's own distinct expiry — here epoch `1882590579`, two years later — never reaches the page.

Clipboard result — the complete, verbatim contents of `F1-clipboard-assertion.out`:

```
PASS  /poctestens.eth  clipboard=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266  expected=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
PASS  /POCTESTENS.eth  clipboard=0x70997970C51812dc3A010C7d01b50e0d17dc79C8  expected=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
PASS  /poctest%C2%ADens.eth  clipboard=0x70997970C51812dc3A010C7d01b50e0d17dc79C8  expected=0x70997970C51812dc3A010C7d01b50e0d17dc79C8

All clipboard assertions passed: the decoy pages copy the ATTACKER address,
the genuine page copies the victim address.
```

`0xf39F…2266` is the victim (anvil account 0); `0x7099…79C8` is the attacker (anvil account 1).

### Files supplied in the linked secret gist

| File                                              | What it is                                                                |
| ------------------------------------------------- | ------------------------------------------------------------------------- |
| `name-split-poc.mjs` / `.out`                     | **canonical PoC** and its output                                          |
| `register-raw-label.sh` / `.out`                  | standalone precondition proof via `cast` alone, and its recorded run      |
| `f1-ui-capture.mjs`, `F1-clipboard-assertion.out` | the capture + clipboard assertion, and its output                         |
| `F1-capture-manifest.txt`                         | sha256 of each frame that run wrote, so the attachments can be tied to it |
| `F1-ui-evidence.txt`                              | what each browser capture shows                                           |
| `ui-stub-indexer.mjs`, `REPRO-F1.txt`             | 20-line GraphQL stub and the full repro guide                             |

### Files uploaded as attachments

`F1-ui-00-baseline-genuine.png` · `F1-ui-01-uppercase-decoy.png` · `F1-ui-02-softhyphen-decoy.png` · **`F1-ui-03-crop-uppercase-decoy.png`** · `F1-ui-04-crop-softhyphen-decoy.png` · `F1-ui-05-crop-baseline-genuine.png`
