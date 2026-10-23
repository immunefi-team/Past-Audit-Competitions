# 92121 w a critical manager renewal signs a different ens registration from the one the page describes the route parameter is ensip 15 normalized for the expiry it displays and only lower cased f&#x20;

**Submitted on Sep 7th 2026 at 11:10:34 UTC by @drMurlly for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92121
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Manager's renewal route derives **which ENS registration the renewal acts on** twice, from the same URL parameter, and reconciles the two nowhere.

The evidence the page shows the user — expiry, protocol, grace state, and (on the profile view that links here) the owner — is read from the **ENSIP-15 normalized** name. The label encoded into the signed `renew()` call is the **raw parameter, lower-cased only**. For any canonically equivalent spelling those are two different on-chain registrations.

The consequence is that a user opening an ordinary renewal link sees a label that renders pixel-identically to their own name, and their own real expiry, is shown no warning, approves, pays, and extends **someone else's** registration. Their own name is not renewed and lapses on schedule.

Demonstrated in two halves, both executed and both attached. **The application half** runs the route's own two derivations — `parseRenewableName` -> `name-parser`, and `normalizeEth2LdName` — against the application's modules imported unmodified, and shows one route parameter producing two different labels and two different labelhashes (`E8-01_renewRouteDivergence.poc.test.ts`, 10 of 10 green, no network and no chain). **The chain half** takes those same two labelhashes onto a local fork: the victim paid 8.005501 USDC, their own name gained **0 seconds**, and the attacker's name gained a full year. The two halves meet on one pair of numbers.

## Vulnerability Details

### The two derivations

Both are inside one route loader, `apps/manager/src/routes/renew/$name.tsx:19-63`, and both read the same raw `name` path parameter.

**A — the DECISION / EVIDENCE side (`:26-28`).** `profileExpiryQuery(name)` to `profileExpiry.ts:77` `normalizeEth2LdName` to `profileName.ts:18` `normalize(name)` — viem's ENSIP-15 normalizer, which composes to NFC. Inside this loader it supplies the protocol gate (`:40`), the grace gate (`:44`), and the expiry the pricing screen renders as "expiring on ". The renew loader itself computes no `isOwner`; the owner side of derivation A sits on the profile route that links here, where `ProfileView.tsx:119` derives `isOwner` from `profileOwner.ts:25`, which normalizes the same name the same way (`normalizeEthName`).

**B — the TRANSACTION side (`:20`).** `parseRenewableName(name)` to `renewableName.ts:26` to `name-parser.ts:46`, whose entire normalization is `name.trim().toLowerCase()`. That label is returned at `:61`, carried by `renewalUi.context.tsx:54`, sent as `renewal.start { label }` by the renewal `ConfirmPurchase.tsx:63-72`, and encoded at `packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1029` as `renew(label, duration, paymentToken, referrer)` after only a `label.replace('.eth', '')` (`:1025`) — which strips a suffix and leaves every divergent byte of the label intact.

The loader returns one value from each: `{ label: <B>, currentExpiry: <A> }` (`:61-62`). One screen therefore describes two different registrations.

### Why the two disagree

An ENS labelhash is keccak over the label's raw UTF-8 bytes. `normalize()` composes `e` + U+0301 COMBINING ACUTE into U+00E9; `toLowerCase()` does not. The two spellings are **canonically equivalent** — by Unicode's own definition the same characters — so any renderer that applies canonical equivalence paints them identically, while the two byte strings hash to different nodes. Measured here on the application itself: the two spellings are two different HTML responses that render to a byte-identical page (`poc/app-half/app-half-control.log`). No other renderer, and no wallet, was measured for this report.

Executed against the repository's own viem (`E8-01_labelhash_derivation.mjs`, output in `E8-01_labelhash_derivation.log`):

```
/renew/jose%CC%81            raw path segment  "jose" + U+0301
  A (screen)   normalize()   "jose" + U+00E9
               labelhash     0xd1b363f99f5db1114896026b88aaf8dcdcc2941cf0685259a11d89b1493fd590
  B (calldata) toLowerCase() "jose" + U+0301   (unchanged)
               labelhash     0x042f930693b6e300fe04af9bc6bfe87689ce2aabcec26f031f3664b4c717f2fe
```

The same script, run on the label the on-chain demonstration actually used, reproduces that run's two labelhashes independently of the chain:

```
  A (screen)   0xb67e9ced05bc8af2d3880ca2671205ac3338c1d05aa33beb8c3ebc74472b596e
  B (calldata) 0x99ba41c83d4f316c239152268ce1c3935582775a403aa38e73ba21b7cea53e6b
```

— the same pair the receipts in `E8-01_run.log` carry.

The two spellings are canonically equivalent, and on the application's own renderer they produce a byte-identical page, as measured above. This is a property of the *pair*, not of any one character: the screen and the calldata disagree because the two code paths apply different transformations, not because the input is exotic.

The divergence is also not confined to combining marks. Run against the repository's own viem (2.52.2) and attached as `poc/E8-01_fullwidth_reach.mjs` with its output in `poc/E8-01_fullwidth_reach.log`, `normalize()` of the FULLWIDTH spelling of `vitalik` (U+FF56 U+FF49 U+FF54 U+FF41 U+FF4C U+FF49 U+FF4B) returns the ASCII `vitalik`, while `.trim().toLowerCase()` returns the fullwidth string unchanged, and the two labelhashes differ (`0xaf2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a67c7103cc` against `0xbf176fcdb903e3ee207e4d1e0791a57f77145cddd183faaa9dc6081aba0b432b`) — so an ordinary ASCII name has a twin that derivation A folds onto it and derivation B signs as a separate registration. Fullwidth forms are *compatibility* equivalent, not canonically equivalent, so that variant is not claimed to render identically, none was measured, and the application's own gates were not exercised on that input. It is stated as reach — the divergence is a property of the two transformations, not of one character class — and not as a second demonstrated exploit. The same run carries its non-vacuity control: given the already-ASCII `vitalik`, both derivations return one string and one labelhash.

### The guardrail is disabled by the same divergence

The Manager already has an interstitial for renewing a name you do not own — `ThirdPartyRenewalDialog.tsx:62-70`, *"That's generous of you! Renewing this name will add time to the current owner's registration"*. It is gated by `thirdPartyRenewalWarning.ts:1-3` (`isOwner === false`), and `isOwner` traces through `ProfileView.tsx:119` to `profileOwner.ts:25` — **derivation A**. A connected victim is the owner *of that name*, so `isOwner` is true, the dialog never renders, and `ProfileRenewAction.tsx:80-87` links straight to the renewal route with the raw bytes unchanged.

That interstitial has exactly **one** render site in the application: `ProfileRenewAction.tsx:66-77`, on the **profile** view (a grep for `ThirdPartyRenewalDialog` over `apps/manager/src` returns that one non-test use). So the divergence suppresses it on the profile-then-Renew path, and a victim who opens the renewal link directly is never offered it at all — the renewal route renders no owner and carries no third-party check. Both routes end at an unwarned signature.

That dialog is also the application's own statement of the on-chain premise: `renew()` succeeds against a name you do not own, and the registrar pulls the rent from `_msgSender()` — stated in the code's own comment at `registration.actors.ts:1006-1012`.

The deployed registrar carries the same premise, and it is not a comment. The verified source of `ETHRegistrar` at `0xa88553F454b77203B0D036A05c894d555EAAa2Cc` on Sepolia — the address the PoC transacts against on its local fork — holds `renew` in `AbstractETHRegistrar.sol`. Its signature and the first five statements of its body, verbatim (the remaining two are the `_onRenew` hook and the `NameRenewed` event):

```
function renew(string calldata label, uint64 duration, IERC20 paymentToken, bytes32 referrer)
    external
{
    IPermissionedRegistry.State memory state = _requireRenewable(label, duration); // reverts if not
    uint64 newExpiry = state.expiry + duration; // reverts if overflow
    uint256 amount = rentPriceOracle.getRenewPrice(label, state.expiry, duration, paymentToken); // reverts if invalid
    SafeERC20.safeTransferFrom(paymentToken, msg.sender, BENEFICIARY, amount); // reverts if payment failed
    ETH_REGISTRY.renew(state.tokenId, newExpiry);
```

There is no caller check anywhere on that path: `_requireRenewable` tests only renewability and a minimum duration, the payer is `msg.sender`, and the registration acted on is selected by `LibLabel.id(label)` — *"Compute `labelhash(label)`"*, `uint256(keccak256(bytes(label)))`. The label's raw bytes pick the registration, and anyone may pay to extend any of them.

### Nothing upstream canonicalizes

`routes/$name/route.tsx:4-13` validates only `length < 3` and `includes('.')`. The router is created with no `parseParams`/`stringifyParams`. The header search reaches the same route through `trim().toLowerCase()` only. Across `apps/manager`'s non-test source there are **three** ENSIP-15 normalize sites (`profileName.ts:18`, `profileName.ts:48`, `setPrimaryName.ts:49`) against **nine** `parseName`/`parseRenewableName` sites, and all **three** route loaders that hand a label onward into calldata are on the unnormalized side — `routes/register/$name.tsx:24,57`, `routes/renew/$name.tsx:20,61` and `routes/renew-v1/$name.tsx:15,51`, each returning `parsedName.value.label` straight out of `.trim().toLowerCase()`. The codebase already knows the rule: `setPrimaryName.ts:46-49` normalizes before building calldata, commenting *"a non-canonical name would fail the bidirectional check at resolution time"* — and it is the one write path that does.

### Why this is not the published known issue

The nearest entry in the programme's published Known Issues is `EXP-INPUT-005`:

> *"ENS name validators accept deceptive characters. Severity: Medium · Status: open, fix ready (Manager and Explorer). Our name validators accept inputs that ENS normalization later rejects or transforms, including homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot. The practical impact is homograph-style display confusion in our UI rather than incorrect resolution."*

**This report is offered THROUGH that entry, not around it.** It does not dispute that the entry names a related root cause, and it takes no position on whether the divergence character falls inside or outside the entry's list — the list is prefaced by *"including"*, so it is illustrative, and `ens-normalize` transforms this vector just as it transforms the ones named. The claim is about the CONSEQUENCE, which is what the programme's own rules make eligible.

**What the entry's promised fix would, and would not, close.** The entry names two halves — validators accept inputs ENS normalization *rejects* **or** *transforms*. This vector is on the second half: `normalize('jose' + U+0301)` does not throw, it returns `josé` (`U+00E9`), run against the repository's own viem in `poc/E8-01_labelhash_derivation.log`. So a validator hardened only to *reject* what ENSIP-15 rejects does not close this — the decomposed label passes it and reaches `renew()` byte for byte. If the ready fix instead has the validator *return* the normalized label, it does close this route, because `renew/$name.tsx:61` signs `parsedName.value.label`, the validator's own return value. This report does not depend on which of the two the ready fix turns out to be.

What the code shows at this commit: **all three** ENSIP-15 normalization sites in the app are on the read/claim side (`profileName.ts:18`, `profileName.ts:48`, `setPrimaryName.ts:49`, all via `normalize` from `viem/ens`; the third builds the primary-name claim string, not renew calldata), while the write-side modules that build calldata — `name-parser.ts`, `nameUtils.ts`, `utils/domain.ts`, `nameChainContractService.ts` — carry **no ENSIP-15 call at all**. The write side's only transformation is `.trim().toLowerCase()`, which folds ASCII case and nothing else.

So the defect is an **asymmetry between two derivations of the same name** — a relation between two code paths — while the entry describes a validator's acceptance predicate and bounds its impact to *"homograph-style display confusion in our UI rather than incorrect resolution"*.

**The display half is conceded, and it is not what is claimed here.** The page does render the attacker's spelling: the loader's B label reaches `PricingStep.tsx:13,24,27` and is printed as the page `<h1>` by `PricingDomainHeader.tsx:9,29`, and the captured live response `poc/app-half/body-renew-nfd.html` carries that 15-code-point label in both the `<h1>` text and its `title` attribute. That is homograph-style display confusion, it sits squarely inside `EXP-INPUT-005`, and no part of this report's severity rests on it.

**What the entry does not carry is the transaction half.** The same divergence makes the page's TRANSACTION DETAIL wrong about a registration the calldata never touches: the expiry the pricing screen states is read under derivation A, from the victim's own registration, while the label the wallet signs is derivation B. The entry's own bound — *"rather than incorrect resolution"* — does not hold either, and not because a resolver returns a wrong record: the application resolves one URL label to the **wrong on-chain registration**, reading expiry, protocol and grace state from one registration while `renew()` names another. Resolving a name to the wrong registration is incorrect resolution, which is the thing the entry's sentence puts *outside* its own impact. What is demonstrated, by execution, is a **completed and paid-for state change against a registration the interface never described**: 8.005501 USDC debited, zero seconds added to the payer's own registration, one year added to a stranger's. A Medium bounded to *"homograph-style display confusion in our UI rather than incorrect resolution"* does not price that.

The door is the one the Known Issues preamble opens itself — *"Anything not listed here is in scope under the normal competition rules, including: … new consequences of a listed root cause that materially change its severity"* — and the consequence is a completed, paid-for transaction against the wrong registration, which is a different kind of harm from a misleading label, and a different severity.

Two neighbouring doors are **not** claimed. The preamble's *"issues we fixed incorrectly or incompletely (a bypass of a shipped fix is a new finding)"*, and the competition rules' *"Fix Bypasses: Bypassing a deployed fix is considered a new, valid bug"*, both require a **shipped** fix, and nothing is shipped at this commit. Eligibility is not argued from non-deployment.

## Impact Details

A user who opens an ordinary renewal link pays for, and extends, a registration belonging to someone else, while the registration they believe they renewed is untouched and lapses.

**Which limb of the impact rows this lands on, stated precisely.** The row claimed is the Critical one that opens *"Causing the application to display or pre-fill an incorrect address, name, or **transaction detail**, so that a user sends funds or submits a transaction to an unintended recipient…"*, and the limb is **transaction detail**.

The *displayed name* and the *signed label* are rendered from the same string (`PricingDomainHeader.tsx:9,29` receives it from `renew/$name.tsx:61`): the page prints the attacker's spelling, as conceded above, but it is the same spelling the calldata carries. So this report does **not** rest on a mismatch between the name on screen and the name in the calldata. What is incorrect is the transaction detail the page pre-fills and then states back to the user. `PricingSummaryCard.tsx:10` takes `currentExpiry` from the loader — derivation A, the victim's own registration, traced at `renew/$name.tsx:62` -> `profileExpiry.ts:77` — and `:28,39,42` render "Renewing for expiring on " computed from it. `RenewalDetails.tsx:15,23-32,105-108` computes the "Expires" date the same way from the same `currentExpiry`, on the renewing screen (`RenewingStep.tsx:62`) and again on the success screen (`SuccessStep.tsx:9`). Neither figure describes the registration the calldata renews. The user approves a transaction whose stated effect is not the effect it has, and is then shown a success screen restating that same wrong expiry — and the third-party-renewal interstitial that exists to catch exactly this is suppressed by the same divergence.

A second row is engaged, and it is offered as a **secondary characterisation, not as the basis of the claim**:

* *"Malicious interactions with an already-connected wallet, such as: **Modifying transaction arguments or parameters**…"*

It is secondary because on this route the application faithfully encodes the label the URL supplied — no argument is rewritten between the page and the wallet — so a triager may reasonably decline it. Declining it leaves the row above untouched.

**Measured on a local fork of Sepolia (no public chain was written to):**

| quantity                                           | measured                                                                                                                   |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| victim USDC before / after                         | 991.994499 / 983.988998                                                                                                    |
| victim paid                                        | **8.005501 USDC**                                                                                                          |
| victim's own name extended by                      | **0 seconds**                                                                                                              |
| attacker's canonically-equivalent name extended by | **31,557,600 seconds (1 year)**                                                                                            |
| control: victim renewing their own label           | +31,557,600 seconds                                                                                                        |
| renewal receipt                                    | ERC-20 `Transfer` of **8005501** (6-decimal USDC), tx `0xbad9385f96b6481983f6b5f765d23ae6a691257ef53cb5b3ca59b5fe7c101f9f` |
| labelhash on screen / in calldata                  | `0xb67e9ced05bc8af2…` / `0x99ba41c83d4f316c…`                                                                              |

Every figure above is greppable out of the attached `E8-01_run.log`, whose stdout matches `E8-01_fork_result.json` field for field (40 of 40).

The control is what makes the result meaningful: the same harness, the same victim and the same registrar DO extend the victim's own registration when the calldata carries the normalized label. The zero above is the defect, not a broken harness.

Two losses follow, and the second is the larger one:

1. **The rent is paid to a third party's benefit.** The victim is debited the full renewal price and receives nothing; the attacker's registration is extended for free.
2. **The victim's name is not renewed.** The user has done the thing that renews a name, has been charged for it, and has been shown a success screen asserting a new expiry that never happened on chain. Their registration keeps its old expiry and lapses on schedule, after which anyone — including the attacker who set this up — may register it.

Every step is permissionless. The attacker needs no privileged role, no access to the victim's wallet and no interaction with the victim beyond getting a link in front of them. The victim performs only the application's advertised workflow: open the page, click Renew, choose a duration, approve. The wallet is not a defence, because the calldata argument is a string that renders with the same glyphs as the victim's own name.

## References

**Where the defect is, and where it is not.** The defect and its fix are wholly inside `apps/manager`: the loader derives the name twice at `routes/renew/$name.tsx:20` and `:61-62`, and normalizing the label the loader returns closes it there. The `packages/transaction-manager` entries below are cited as the SINK — they faithfully encode whatever label they are handed and no defect is claimed in them — so this report is filed against the Manager app asset. Nothing on the path is outside the programme's declared assets in any case: `packages/transaction-manager` is itself one of the five.

* `apps/manager/src/routes/renew/$name.tsx:19-63` — the loader holding both derivations
* `apps/manager/src/routes/register/$name.tsx:24,57` and `apps/manager/src/routes/renew-v1/$name.tsx:15,51` — the two other loaders that return the unnormalized label
* `apps/manager/src/features/register-v2/utils/name-parser.ts:46` — `trim().toLowerCase()`, the transaction side
* `apps/manager/src/features/profile/service/profileName.ts:18` — `normalize()`, the evidence side
* `apps/manager/src/features/profile/service/profileExpiry.ts:77` and `profileOwner.ts:25` — the two consumers of the normalized name
* `apps/manager/src/features/renew/utils/renewableName.ts:26` — `parseRenewableName`
* `apps/manager/src/features/renew/workflow/pricing/components/ConfirmPurchase.tsx:63-72` — `renewal.start { label }`
* `apps/manager/src/features/renew/workflow/pricing/PricingStep.tsx:13,24,27` and `apps/manager/src/features/register-v2/workflow/pricing/components/PricingDomainHeader.tsx:9,29` — the loader's B label printed as the page `<h1>`
* `apps/manager/src/features/renew/workflow/pricing/components/PricingSummaryCard.tsx:10,28,39,42` and `apps/manager/src/features/renew/workflow/renewing/components/RenewalDetails.tsx:15,23-32,105-108` — the stated new expiry, computed from derivation A
* `packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1029` — `renew(label, ...)` encoding
* `packages/transaction-manager/src/machines/registration/registration.actors.ts:1006-1012` — the code's own note that the registrar pulls rent from `_msgSender()`
* `apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts:1-3` and `components/ThirdPartyRenewalDialog.tsx:62-70` — the suppressed guardrail
* `apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx:66-77` — its only render site, on the profile view
* `AbstractETHRegistrar.sol` in the verified source of `ETHRegistrar` at `0xa88553F454b77203B0D036A05c894d555EAAa2Cc` (Sepolia) — `renew()` with no caller check, rent from `msg.sender`, registration selected by `LibLabel.id(label)`
* `apps/manager/src/features/profile/service/setPrimaryName.ts:46-49` — the one write path that does normalize, and its comment stating the rule
* `apps/manager/src/routes/$name/route.tsx:4-13` — the route parameter validation that does not canonicalize
* ENSIP-15 (ENS Name Normalization) and Unicode Annex 15 (Normalization Forms) for canonical equivalence

## Link to Proof of Concept

<https://gist.github.com/drMurlly/bec4a1da6a5d581abcb04b6594401398>

## Proof of Concept

## Proof of Concept

### What is demonstrated

That the renewal a user approves extends a different registration from the one the page described, that the user is charged for it, and that their own registration is untouched.

### Harness — LOCAL FORK ONLY

A local `anvil` fork of Sepolia, forked at block **11610837** (chain id 11155111; `poc/E8-01_fork_nodeinfo.log`). The harness recorded the head as 11610978 when the run began — the local stack's own setup transactions had already been mined on top of the fork base — and the renewal receipt landed at 11610988. Every block above 11610837 exists only on this local node. **No public testnet or mainnet was written to**, per the programme's PoC rules. Contracts are the ones the application itself resolves through `@ensdomains/ensjs` for Sepolia:

```
ETHRegistrar   0xa88553F454b77203B0D036A05c894d555EAAa2Cc
ETHRegistry    0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
USDC           0x768F42455A2D082E23ceeF7d51e5787C82d67a39
```

### The two labels

One base name in its two canonically equivalent spellings. They render identically. The demo pins the label so a re-run reproduces these hashes exactly (`E8_LABEL`, default `e8poc-jose`).

```
A  the name the page describes (NFC, what normalize() yields)
   14 code points, ending ... 65 6a 6f 73 e9
   labelhash   0xb67e9ced05bc8af2d3880ca2671205ac3338c1d05aa33beb8c3ebc74472b596e

B  the label the application signs (NFD, what parseName() yields)
   15 code points, ending ... 6a 6f 73 65 301
   labelhash   0x99ba41c83d4f316c239152268ce1c3935582775a403aa38e73ba21b7cea53e6b
```

The two labelhashes differ, so the registrar holds them as two independent registrations. Every number below is in the attached `poc/E8-01_run.log` and `poc/E8-01_fork_result.json`.

### Steps

1. **Attacker registers the twin, through the application's own registration flow.** Every gate on that path passes on the B spelling, and this was **executed against the application's own modules, imported unmodified** — `poc/E8-01_renewRouteDivergence.poc.test.ts`, raw output in `poc/E8-01_divergence_run.log` (node v22.14.0, vitest 4.1.5, 10 of 10 green, no network, no chain, no mocks): `parseName` returns ok, the code-point length check passes, `validateENSName` returns null, `normalizeDomainNameFromUrl` leaves the bytes unchanged. That ensjs `getAvailable` forwards the raw label to the registrar is a source read, corroborated on the fork by the availability probe below. Recorded on the fork: `available_before_registration` is `{nfc: true, nfd: true}` and, after both registrations, `available_after_registration` is `{nfc: false, nfd: false}`. **The registrar does not normalize:** the two spellings take two slots, at two different labelhashes, and they carry two independent expiries (1819846176 and 1819846297) throughout the run.
2. **Attacker sends the victim a link to the renewal route carrying the B spelling.** The header search reaches the same route the same way.
3. **Victim opens it.** The page's own data is read from A: the expiry rendered as "expiring on ", the protocol gate and the grace gate all come from `renew/$name.tsx:26-28,40,44` -> `profileExpiry.ts:77` -> `profileName.ts:18` `normalize()`, while the loader carries B's label forward unchanged — it returns `{ label: <B>, currentExpiry: <A> }` at `:61-62`.

   **This step is a code read, and it is labelled as one.** The two attached renders were captured with **no wallet connected** — both show `CONNECT` and `CONNECT TO RENEW` — so they say nothing about `isOwner` or about the interstitial. What the source says: the third-party-renewal interstitial has exactly one render site in the application, `ProfileRenewAction.tsx:66-77` on the **profile** view, gated by `thirdPartyRenewalWarning.ts:1-3` (`isOwner === false`), with `isOwner` traced through `ProfileView.tsx:119` to `profileOwner.ts:25` — derivation A again. So a connected victim who owns A, arriving on the B spelling, is resolved to owner = himself, `isOwner` is true, the dialog is suppressed, and `ProfileRenewAction.tsx:80-87` links straight on with the raw bytes. The renewal route reached directly renders no owner at all (`command grep -i owner` over `routes/renew/$name.tsx` and `features/renew/workflow/RenewalPage.tsx` returns 0), so it presents no third-party check either. That the owner is looked up under A **is** measured, in the same vitest run as step 1 (`normalizeEthName` of the B route parameter returns the A name). **A wallet-connected walkthrough is owed and is not claimed here.**
4. **Victim clicks Renew, chooses a duration and approves.** The submitted call is `renew(B, duration, USDC, referrer)`.

   **Measured, not read.** The same vitest run puts the single route parameter through the application's own `parseRenewableName` -> `name-parser` and through `normalizeEth2LdName`, and prints both results and both labelhashes. Quoted from that run, with the two label strings transcribed into code-point notation:

   ```
   [defect]  B calldata label: "e8poc-josejos" + U+0065 U+0301   0x73 0x65 0x301
   [defect]  A evidence label: "e8poc-josejos" + U+00E9          0x6f 0x73 0xe9
   [defect]  labelhash A (screen)  : 0xb67e9ced05bc8af2d3880ca2671205ac3338c1d05aa33beb8c3ebc74472b596e
   [defect]  labelhash B (calldata): 0x99ba41c83d4f316c239152268ce1c3935582775a403aa38e73ba21b7cea53e6b
   ```

   The two label strings are given as code points, not as literal text, because a web form cannot be relied on to preserve a combining mark: pasted literally, `e` + U+0301 composes to U+00E9 in transit and the printed string stops matching the byte annotation beside it. The exact bytes as the run printed them are in the attached `poc/E8-01_divergence_run.log`, which is the authoritative copy of this block.

   Those are the same two labelhashes the fork run registers and renews, so the application half and the chain half meet on one pair of numbers. The control in the same file is not vacuous: given the already-canonical spelling, both derivations return the same label and the same labelhash. `registration.actors.ts:1020-1029` then encodes that label into `renew()` after only a `label.replace('.eth','')` — a source read, and the fork run shows the resulting call succeeding.

### Result, measured

The victim wallet is funded with 1000.0 USDC (`E8-01_fork_demo.py:fund()`, `anvil_setStorageAt` of `1_000_000_000` raw units) and registers its own name first, which is why its balance immediately before the renewal is 991.994499. The renewal is what the following measures.

```
victim USDC before the renewal          991.994499
victim USDC after  the renewal          983.988998
victim paid                               8.005501 USDC

victim's own registration (A) expiry    1819846176  ->  1819846176     +0 s
attacker's registration   (B) expiry    1819846297  ->  1851403897     +31,557,600 s

CONTROL - same harness, victim renews A  +31,557,600 s
```

The USDC `Transfer` event on the renewal transaction (`0xbad9385f96b6481983f6b5f765d23ae6a691257ef53cb5b3ca59b5fe7c101f9f`, status `0x1`) carries `8005501` (6 decimals) from the victim's address `0x7f0fdc406b4731f6783bacb33b41a4b242f537f6` to the registrar's beneficiary `0x84d3a426d4e12e955d1df95db0b24fe26afe39d3`.

### The application half, and its control

The fork run above is the chain half. The eleven `poc/app-half/` files are the application half, and together they are its non-vacuity control — measured in `poc/app-half/app-half-control.log`:

```
render-nfc-victims-own.png     sha256 6fc86908b487ab66…
render-nfd-attackers-twin.png  sha256 6fc86908b487ab66…      <- BYTE-IDENTICAL

body-renew-nfc.html            sha256 f14de1cd04321ee0…  27218 bytes
body-renew-nfd.html            sha256 1c23a65193c42a33…  27225 bytes  <- DIFFERENT responses
```

The two HTML bodies are what make the identical screenshots evidence rather than a duplicated capture: they are two different server responses, and across 605 characters of visible text they differ in **exactly one place** — `U+00E9` in one, `U+0065 U+0301` in the other. Two different byte sequences went in; one pixel-identical page came out. That is the deception this finding turns on, and if the two pages had rendered differently there would be no finding.

**What these two captures do show.** Two byte-different requests, one pixel-identical page, and — inside one server-rendered payload — **one expiry carried against two different labels**. Both bodies serialise `currentExpiry:1851403776n`, the same value to the digit, while the `label` field beside it differs: in `body-renew-nfc.html` it is `"e8poc-josejos"` + U+00E9, 14 code points; in `body-renew-nfd.html` it is `"e8poc-josejos"` + U+0065 U+0301, 15 code points. That is `renew/$name.tsx:61-62` returning `{ label: <B>, currentExpiry: <A> }` — the divergence itself, on the wire, off the running application rather than in a test harness. The 15-code-point label is also what the page prints as its `<h1>`, in the element's text and its `title` attribute, so the response renders the spelling the attacker chose. The code points above were extracted from the two attached bodies, so the extraction is checkable against the attachments.

**What they do not show.** They were taken with **no wallet connected** — both show `CONNECT` in the header and `CONNECT TO RENEW` as the primary action, and neither body contains an owner field at all. So they are **not** evidence about `isOwner` or about the interstitial: with no wallet connected `isOwner` is `undefined`, and `shouldShowThirdPartyRenewalWarning(undefined)` is `false` by the project's own test (`thirdPartyRenewalWarning.test.ts:14`), so the dialog would be absent either way. The suppression argument is the source read given in step 3. The two filenames say which SPELLING was requested, not who was connected.

### The control, and why it matters

The final line is a non-vacuity control. The same harness, victim, registrar and payment token DO extend the victim's own registration when the calldata carries the A label. So the `+0 s` in step 4 is the divergence being demonstrated, not a harness that cannot renew anything.

### Reproducing

`poc/E8-01_fork_demo.py` (attached) carries the exact calls; `poc/E8-01_run.log` is its raw stdout and `poc/E8-01_fork_result.json` the same result as a file. Every number quoted above appears verbatim in both. The only text above that is not verbatim from a log is the pair of label strings in step 4, transcribed there into code-point notation for the reason given at that block; their raw bytes are in `poc/E8-01_divergence_run.log`. The demo refuses any RPC that is not loopback and does not answer `anvil_nodeInfo`, so it cannot touch a public chain.
