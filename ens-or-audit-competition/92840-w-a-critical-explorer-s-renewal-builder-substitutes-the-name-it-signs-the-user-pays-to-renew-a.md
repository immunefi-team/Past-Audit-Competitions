# 92840 w a critical explorer s renewal builder substitutes the name it signs the user pays to renew a different ens name than the one displayed

**Submitted on Sep 12th 2026 at 18:34:32 UTC by @nullbevel for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92840
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer's Extend flow re-normalizes the name for the renewal transaction but displays it without re-normalizing — two copies of one variable in one hook: the displayed title (`useRenewalTransactions.ts:459-460`) and the raw `name` reaching the builder (`:431`) on the path the user signs. When they differ, the user is shown one `.eth` name, quoted the price of a second, and pays to renew that second name. When the normalized twin is registered and the allowance covers the price, `renew` is permissionless and succeeds without checking that the payer owns it: the payment goes through, the other name is extended, the user's own is not, and the app reports success. I drove this end to end in a browser against a local chain: the app showed the user `ﬁle.eth`, which belongs to the connected account; the wallet signed `renew("file", …)`; a different address's `file.eth` moved forward a year; the victim's `ﬁle.eth` did not change; and 160.438366 USDC left the victim.

## Vulnerability Details

At commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, asset `apps/portal`. The PoC prints the calldata produced by the app's own functions, not a description of them.

### 1. One variable, normalized for the chain and not for the screen

`useRenewalTransactions.ts:454-478` builds the single-name renewal:

```ts
title: `Extend ${single.selectedName.name}`,   // :459  raw
prepare: ... buildRenewIntent({
  name: single.selectedName.name,              // :469  raw, in
```

and `buildRenewIntent` at `:190-206`:

```ts
// ensjs splits the label without normalizing, so pass a normalized 2LD name.
name: `${getLabel(params.name)}.eth`,          // :197  normalized
```

`getLabel` (`utils/token/getLabel.ts:10-19`) runs `ens_normalize` then `ens_split`. So `title` carries the raw string into the overview the user reads, and `args[0]` carries the normalized one — two copies of one variable transformed differently in one flow. `renewNameWriteParameters` (`renewName.js:24,34`) does `const [label] = name.split('.')`, so whatever label is handed in reaches `renew(...)`. ensjs genuinely does not normalize; normalization is applied to the calldata and not the display, and that is what splits them.

### 2. Every surface shows the raw string

* `ExtendNameSettings.tsx:34-35`, `ExtendNameConfirmation.tsx:32-33`: `<NameAvatar name={selectedName.name}/>`, `<h2>{selectedName.name}</h2>`.
* `ExtendNameConfirmation.tsx:35-39`: `baseDate={selectedName.expiryDate}` — the raw name's expiry, so "new expiry" is computed from the user's own name while the transaction extends the other one.
* `useRenewalTransactions.ts:459-460`: the title in the wallet-adjacent step list; `:221,:223` write the raw name into the transaction-history record.

`selectedName.name` is never normalized on any of these paths — it is the indexer's string verbatim.

### 3. The price is computed for the normalized name; the eligibility gate does not reconcile identities

Why the flow completes rather than errors, and why the money figure is not an independent witness:

* Price: `useNamePricing.ts:37-44` → `useRegistrationPrice.ts:53-54` does `getLabel(name)` — so the quote is the **normalized** name's price, not the displayed name's.
* Eligibility: `isExtendable2LD` filters on the **raw** name and its expiry (`nameExtension.ts:61-74` — `/^[^.]+\.eth$/` plus a grace window, no `getLabel`), and `useRenewableNames` drops non-renewable v1 names via on-chain `isRenewable` while **v2 rows pass through untouched** (`names.tsx:110-134,266-272`). The gate never compares the raw label against its normalized form, so it neither catches nor reconciles the divergence.

So only the price crosses to the normalized label, while the screen keeps the raw one. In the browser run the modal quoted `$160.11/yr` (`file`'s 4-char price); `ﬁle` (3 codepoints) costs 640/yr — `poc-015-setup-local-chain.mjs` prints `getRenewPrice("file")=160000009`, `getRenewPrice("ﬁle")=640000005`. Distinct names, distinct prices; the user sees their own name and the other's price.

### 4. The raw name reaches both the display and the calldata, and nothing reconciles them

`routes/addr/$addr/names.tsx:177-204` builds rows from the two name queries; `useV2NamesWithRolesForAddress.ts:49-62,86` selects `name` unchanged, and `getSelectedNames` (`nameExtension.ts:11-23`) passes it to `SelectedName.name`. The only name-shape filter is `isExtendable2LD` (`:61-74`, test `/^[^.]+\.eth$/` at `:66`); it checks shape and grace, not canonicity, so the raw string reaches both the display and, via `getLabel`, the calldata, with nothing reconciling them.

### 5. The app already has the check, and uses it on the other entry point

`utils/token/isNormalized.ts` exports `isNormalized` (`:24-27`) and `isValidEnsName` (`:38-57`). The profile route uses them: `routes/$name/index.tsx:442` computes `isValidEnsName(name)` and `:458-460` renders `<InvalidNameMessage/>`, so a name's own page is unreachable for a non-canonical name. That block is right at an *entry point*, where a name is typed; the renewal path is not one — the name is already owned and selected, and the correct response is to renew it under the name it actually is and show that name. It does neither: it refuses nothing and reconciles nothing, proceeding with the raw name on screen and the normalized one in the calldata.

### 6. Non-canonical rows exist in live data — including seven that normally render indistinguishably

`poc-015-canonicity-census.mjs` (output `poc-015-census-output.txt`) runs every domain the local indexer serves through the repo's own `@adraffy/ens-normalize@1.11.1` (its local query selects `name` only). Two rows are non-canonical — `TeStinG.eth` (→ `testing.eth`) and `ｃａｔ.eth` (fullwidth, → `cat.eth`) — proving non-canonical rows are served into the list the flow reads. I claim no live renewable twin for either: `TeStinG.eth` is a v2 row, and the end-to-end payment is shown on the created `ﬁle`/`file` v2 pair (Proof of Concept field, Confirmation 3). It arrives raw because it was not typed into the app — `name-parser.ts:46` lowercases typed input, so a mixed-case label reached the chain through some path that preserved it, and it comes to the app from the indexer.

`TeStinG.eth` was already present in the local indexed state; this PoC did not insert it. (The browser PoC injects a *different* row, `ﬁle.eth`, because the e2e fixtures register through a registrar the indexer's `contracts.json` does not list, Confirmation 3.) The app also chooses to receive such rows: the indexer exposes an `isNormalized` filter (`graphql.gen.ts:158`) the list query declines (`useV2NamesWithRolesForAddress.ts:52-58` passes only `owner`).

The staging endpoint shows the **shape** (re-run 2026-09-12): of 44 rows flagged `isNormalized:false`, 31 `ens_normalize` rejects; the remaining 13 normalize to a *different* string, and **seven of those normally render indistinguishably from their canonical form**:

```
raw='café.eth'
  raw codepoints        : U+0063 U+0061 U+0066 U+0065 U+0301 U+002E U+0065 U+0074 U+0068
  normalized codepoints : U+0063 U+0061 U+0066 U+00E9 U+002E U+0065 U+0074 U+0068
```

`c a f e ◌́` versus `c a f é`: canonically equivalent code-point sequences for the same graphemes (`name.normalize('NFC') === ens_normalize(name).normalize('NFC')` for all seven). The census establishes only this equivalence — not whether the twins are registered, or by whom. Separately, the registrar accepts a non-canonical label (I registered `ﬁle`, U+FB01, through the v2 `ETHRegistrar`: the contract stores whatever label it is given), and the indexer already carries `normalizedName` (`graphql.gen.ts:103`) that the Manager reads (`dashboard/utils.ts:45-46`) but the Explorer's renewal query does not select.

### 7. Not a single-name quirk — the bulk path signs the same substitution

`buildRenewIntent` is the only renew-intent builder and always normalizes; its bulk call site (`:298-299`) carries the raw name to the display (`:291-292`) exactly as the single path. The same divergence affects every selected bulk row, and the payment planner aggregates them under one approval (`renewerPayments.ts:37-50,:83-90`), assembled via a select-all checkbox (`NamesTable/columns.tsx:31-39`). I did not drive the bulk flow in a browser (see poc).

### 8. The failure is silent

`handleDone` (`:365-402`) invalidates the **raw** name's queries (`:386-399`), so the app refetches the user's own name, gets its unchanged expiry, and reports completion. Nothing compares the label signed against the name displayed; the only signal is that the expiry did not move.

## Why this is not EXP-INPUT-005

EXP-INPUT-005, in the program's Public Disclosure of Known Issues, is a **validator** defect: *"ENS name validators accept inputs that ENS normalization later rejects or transforms … causing homograph-style display confusion in the UI rather than incorrect resolution."* Its root cause is at the **input boundary** — a control that should reject a deceptive typed string does not — and its consequence is visual confusion.

**This report is a different root cause: identity substitution inside the transaction builder.** The name already exists and arrives as authoritative identity from the indexer; after the user selects it, `buildRenewIntent` replaces that identity with `getLabel(params.name)` (`:197`) while the title keeps the original (`:459`). The defect is an inconsistent transformation of one value inside a transaction constructor, not an input a validator failed to reject. The renewal path applies no ENSIP-15 or canonicity validator — only `isExtendable2LD`, a shape-and-grace filter (§4).

**Independence proof, verified in the audited code.** The audited snapshot contains a strict ENSIP-15 validator (`utils/token/isNormalized.ts`) and applies it on the individual-name page (`routes/$name/index.tsx:442,:458-460`), where it rejects `ﬁle.eth`. The by-address flow instead copies `domain.name` through unchanged (`useV2NamesWithRolesForAddress.ts:49-62,:86`) and the builder still signs `file`. So the validator can work perfectly and the bug still occurs — the PoC control shows exactly this (`isValidEnsName("ﬁle.eth") = false` beside `ON SCREEN: Extend ﬁle.eth` / `IN CALLDATA: renew("file")`).

**Fix-disjoint both ways.** Enforcing ENSIP-15 on every user input fixes EXP-INPUT-005 but leaves indexer rows reaching the builder unchanged; removing the substitution in `buildRenewIntent` (or aborting when raw and signed labels differ) fixes this but not EXP-INPUT-005's validators. Regardless of its unpublished patch, enforcing normalization at input boundaries does not preserve identity at this transaction boundary.

**Alternative, if read as one root cause.** The program's scope rules admit *"new consequences of a listed root cause that materially change its severity"* as new findings — and the consequence here (a signed transaction renewing a different registration, value leaving the balance) is a demonstrated escalation over EXP-INPUT-005's display confusion, not an automatic one.

## Impact Details

**The harm is material, and it is not a display defect — the distinction that decides scope.** Two effects land at once: 160.438366 USDC leaves the user's balance to extend someone else's registration, and their own name is *not* renewed while the app reports success — a user who renewed near expiry believes they have headroom they do not, which can end in a lapsed name. So the out-of-scope line *"UX and UI impacts that do not materially disrupt use of the platform"* does not reach this: a paid renewal of the wrong name, and an unrenewed name shown as renewed, is not a rendering blemish — it acted on a different ENS node than the one approved, with money moving. Nor is it "garbage in": the app lists, prices, builds calldata for and reports the name as renewed — treating it as a first-class renewable — so its own conduct refutes that signing a *different* name is acceptable by design.

**Selected impact — Critical: "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user … submits a transaction to an unintended recipient … showing the wrong recipient or amount on a transaction the user then approves"** (from the program's Impacts in Scope). The application displays one name on every screen, including the last before signing, and encodes a different one into the calldata. I map this to the selected impact because the displayed name and the signed transaction detail diverge, and the registrar applies the payment to another registration. (The EVM destination `to` remains the registrar in both cases.)

**The attacker action, and its bound.** `renew` reverts on an unregistered label, so registering the canonical twin turns a loud free failure into a silent paying success — one ordinary registration, no access to the victim (`poc-015-twin-absent-reverts.sh`, output `poc-015-twin-absent-output.txt`: as a negative control an unregistered label reverts `0x1caefaa0`, while the registered canonical twin `file`'s `eth_call` succeeds (simulation does not revert)). It does **not** buy target selection — the attacker chooses only among holders of a divergent name, and the money reaches the registrar, not them. The severity is the victim's demonstrated loss (payment for a name they do not own, their own left to lapse), not the attacker's gain.

## Severity

**Critical, under the "incorrect address, name, or transaction detail" impact above, with its magnitude stated plainly.** Support: fully demonstrated end to end, an executed ERC-20 payment in the local end-to-end PoC, app reporting success; no privileged position; non-canonical rows present in the indexed state used. The preconditions, which are not trivial, are stated below.

Limits, plainly:

* Magnitude is one renewal at the *other* name's price, keyed on the normalized label's codepoint length (`useRegistrationPrice.ts:53-59` → `useBaseRate.ts:48-63`), so the error runs either way; one tier step measured locally is 4×.
* A basket aggregates the user's selected divergent rows under one approval (`toggleAllPageRowsSelected` selects every filtered row); the per-session figure is bounded by how many divergent renewable names the victim holds, and by the usual transaction, gas and wallet-balance limits.
* **The app's own warning is suppressed for exactly this victim.** `ExtendNameDisclaimer.tsx:14-17` warns "Extending a name you do not own will not give you ownership of it"; `ExtendNameModal.tsx:57-72` resolves the **raw** name's owner and skips the warning when the connected wallet matches ("noise for owners"). The victim *is* the owner of the name on screen, so it is skipped while the transaction extends a name they do not own. (The bulk modal always shows it.)
* The user must hold a non-canonical name — a minority; on staging, 7 of 44 flagged rows are NFC-equivalent variants that normally render indistinguishably.
* The canonical twin must exist and be renewable, or the flow reverts with an error. Partly self-limiting.
* It cannot be aimed at a chosen victim unless they first acquire the lookalike.

## Proof of Concept

See the Proof of Concept field. Executed: the browser drive of the single-name flow (`drive-015-extend.mjs`), the fix-sensitive unit test rendering the real hook (`poc-015-real-hook-signs-different-label.test.tsx`, 3/3), the attacker-step unregistered-label negative control and registered-label witness (`poc-015-twin-absent-reverts.sh`), and the canonicity census (`poc-015-canonicity-census.mjs`).

**Reproduction note:** the local anvil chain expires the fixtures. With `pnpm e2e:infra:up` running, from `repo/e2e` (where `viem` resolves) re-seed, set the allowance, then run the control:

```
# copy the .mjs/.sh from the gist into repo/e2e, then run from there:
node poc-015-setup-local-chain.mjs
# USDC = 0x768F42455A2D082E23ceeF7d51e5787C82d67a39 ; ETHRegistrar = 0xa88553F454b77203B0D036A05c894d555EAAa2Cc
# anvil account 0 (public test key):
cast send 0x768F42455A2D082E23ceeF7d51e5787C82d67a39 "approve(address,uint256)" \
  0xa88553F454b77203B0D036A05c894d555EAAa2Cc 1000000000000 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --rpc-url http://127.0.0.1:8545
bash poc-015-twin-absent-reverts.sh
```

Without the `approve`, `renew("file")` reverts with an allowance error and the finding appears not to reproduce. The browser drive (`drive-015-extend.mjs`) additionally needs the Explorer served locally — see the Proof of Concept field.

## Recommendation

Normalizing here is the defect. The name the user owns and the screen shows is the raw on-chain label, so the renewal must sign **that** label — not a normalized rewrite of it that resolves to a different registration.

**Primary fix — sign the label the user owns.** In `buildRenewIntent`, pass the raw label rather than the normalized one: after confirming the name is a 2LD `.eth`, use `params.name.split('.')[0]` (the on-chain label) in place of `getLabel(params.name)`. Display and calldata then address the same node — the one the user holds — and the renewal renews it.

**Safe alternative — refuse divergent labels.** If the flow should not renew non-canonical names at all, reject them explicitly instead of silently substituting:

```ts
const label = params.name.split('.')[0]
if (!isNormalized(params.name) || getLabel(params.name) !== label) {
  throw new Error(`Refusing to renew ${params.name}: label is not normalized`)
}
```

`buildRenewIntent` is the only renew-intent builder, so either change covers the single path, the bulk path and `buildRenewTransaction`.

## References

Program: [scope](https://immunefi.com/audit-competition/audit-competition-ens/scope/) and [information](https://immunefi.com/audit-competition/audit-competition-ens/information/) (the source of the Impacts in Scope, the Public Disclosure of Known Issues, and the duplicate/severity rules cited above).

Repository at `1c9b47f`, asset `apps/portal`:

* `src/features/renew/hooks/useRenewalTransactions.ts:190-206` (`buildRenewIntent`, comment `:191`, substitution `:197`); `:454-478` (single-name tx, raw `title`/`transactionName` `:459-460`, signed path `:431`); `:221,:223` (raw name in the history record); `:290-292,:298-299,:310` (bulk path); `:365-402` (`handleDone`); `:157` (2× approve); `:482` (approve skipped when allowance covers price).
* `src/utils/token/getLabel.ts:10-19`; `src/utils/token/isNormalized.ts:24-27,:38-57`.
* `src/features/renew/utils/nameExtension.ts:11-23,:61-74`; `renewerPayments.ts:37-50,:83-90,:92-97`.
* `src/features/renew/hooks/useNamePricing.ts:37-44`, `useRegistrationPrice.ts:53-54` (price on the normalized label).
* `src/features/renew/components/ExtendNameSettings.tsx:34-35,:40`, `ExtendNameConfirmation.tsx:32-39`, `ExtendNameModal.tsx:57-72`, `ExtendNameDisclaimer.tsx:14-17`.
* `src/routes/addr/$addr/names.tsx:170-204,:266-272,:408,:419` (`useRenewableNames` at `:110-134`); `useV2NamesWithRolesForAddress.ts:49-62,:86`; `NamesTable/columns.tsx:31-39,:51,:61-63`.
* `src/routes/$name/index.tsx:442,:458-460`.
* `packages/indexer/graphql.gen.ts:103` (`normalizedName` on `Domain`), `:158` (`isNormalized` in `DomainFilter`, the filter the list query declines); `apps/manager/src/features/dashboard/utils.ts:45-46`.
* Dependency (to establish the encoder does not normalize; not reported as a bug in it): `@ensdomains/ensjs/wallet`, `dist/actions/wallet/renewName.js:24,:34`.

## Link to Proof of Concept

<https://gist.github.com/casaisdev/153a921adc1265ea03bd424df815f566>

## Proof of Concept

## Proof of Concept

Three pieces of evidence: a unit test that renders the app's real renewal hook (fix-sensitive), a scan of the indexer the app reads, and a full headless-browser click-through ending in a signed transaction and a changed on-chain expiry. Commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, asset `apps/portal`.

**No mainnet or public-testnet contract was written to or tested; all on-chain execution used the local Anvil fork from `pnpm e2e:infra:up` (`http://127.0.0.1:8545`).** In the browser drive the page context blocks outbound requests and locally fulfils the v1 subgraph query with an empty response. The one *intentional* call to ENS infrastructure is the census (Confirmation 2), a single read-only query to ENS's public staging GraphQL endpoint. Separately, the unit tests import the app's `@/lib/wagmi`, whose AppKit/WalletConnect initialisation may attempt unrelated outbound requests (e.g. to `pulse.walletconnect.org` / `api.web3modal.org`); these are incidental to the app booting; they may fail (e.g. `EAI_AGAIN`), be aborted during teardown, or complete — either way they touch no ENS system and the tests do not depend on them (the tests acknowledge them in a header comment).

### Preconditions

A user who holds a `.eth` name whose on-chain label is not its own normalized form, and a second party who holds the normalized form. These are uncommon preconditions — a divergent name must exist, and its canonical twin must be registered and renewable — but no privileged account is needed and the second party takes no action; they only have to own the canonical label.

### Step-by-step

1. Open `/addr/<address>/names`: rows come from the indexer unfiltered (`names.tsx:177-204`, `useV2NamesWithRolesForAddress.ts:49-62,:86`); the cell renders the stored string (`columns.tsx:61-63`).
2. Tick the row, press Extend. For a v2 row the only name-eligibility gate is `isExtendable2LD` (`nameExtension.ts:61-74,:66`) — shape and grace, not canonicity (v1 rows also face `useRenewableNames`, `names.tsx:266-272`).
3. Settings/confirmation show the raw name (`ExtendNameSettings.tsx:34-35`, `ExtendNameConfirmation.tsx:32-33,:38`); the price is the **normalized** label's (`useNamePricing.ts:37-44` → `useRegistrationPrice.ts:53-54`).
4. The transaction overview — last screen before the wallet — is titled from the raw name (`useRenewalTransactions.ts:459-460`).
5. On start, `buildRenewIntent` (`:190-206`) encodes `getLabel(params.name)` at `:197`; ensjs takes the label (`renewName.js:24,:34`); the wallet signs `renew(<normalized>, …)`.
6. `renew` mines and the other name is extended; `handleDone` (`:365-402`) invalidates the **raw** name's queries (`:386-399`) and reports success.

### Confirmation 1: the real hook, and a test that fails when the bug is fixed

`poc-015-real-hook-signs-different-label.test.tsx` — **3/3 passing**. Copy to `apps/portal/src/features/renew/hooks/` and run `npx vitest run --pool=threads poc-015-real-hook-signs-different-label.test.tsx` from `apps/portal` (`--pool=threads` makes the test's `process.stdout.write` evidence lines reach the terminal deterministically; Vitest's default fork pool can drop them depending on version/OS).

It renders the **real** `useRenewalTransactions`, mocks only the boundary (transaction-manager, wagmi, wallet client, modal, query client — the chain object is the repo's own `sepoliaWithEns`), calls the real `onStart`, decodes the calldata that reached `transactionManager.startTransaction`, and — as a control — runs the app's own ENSIP-15 validator on the same name:

```
VALIDATOR  : isValidEnsName("ﬁle.eth") = false
ON SCREEN  : Extend ﬁle.eth
IN CALLDATA: renew("file", ...)  ->  0xa88553F454b77203B0D036A05c894d555EAAa2Cc
Tests  3 passed (3)
```

The control is the point: the app's canonicity validator (`utils/token/isNormalized.ts`) *rejects* `ﬁle.eth`, yet the renewal flow shows it and signs `renew("file")` regardless — the validator works, and the bug occurs anyway, because the renewal path never calls it.

Three cases: a canonical-name control that signs the label it shows, the divergence above, and an assertion of a single `startTransaction` call (there is no alternative renewal intent builder in the app). It is **fix-sensitive**: either recommended fix turns it red — the safe-alternative throw shown here, or the primary fix (signing the raw label the user owns), which removes the divergence the test asserts:

|                                                    | with the bug | with the fix                                                    |
| -------------------------------------------------- | ------------ | --------------------------------------------------------------- |
| `poc-015-real-hook-signs-different-label.test.tsx` | 3/3 pass     | **fails**: `Refusing to renew ﬁle.eth: label is not normalized` |

A second file, `poc-015-renewal-signs-a-different-name.test.ts` (6/6), runs the app's own helpers over six non-canonical test inputs (one the `TeStinG.eth` row observed in the local indexer, the other five synthetic) and a five-name basket; its assertions demonstrate 3 of 5 basket rows signing a label different from the one displayed, all covered by the single `800000000` approval the user signs once. It is supporting, not decisive — it transcribes the private `buildRenewIntent` line rather than importing it; the real-hook test above carries the fix-sensitivity.

### Confirmation 2: the indexer serves non-canonical names, and the Explorer renders them

`poc-015-canonicity-census.mjs` runs every domain the local Panoptes container serves through the repo's own `@adraffy/ens-normalize@1.11.1` (output `poc-015-census-output.txt`):

```
=== (a) LOCAL indexer http://127.0.0.1:5655/graphql
total domains indexed: 1667
non-canonical rows   : 2
   raw='ｃａｔ.eth'  normalized='cat.eth'
   raw='TeStinG.eth'  normalized='testing.eth'
```

The census's local query selects `name` only; the container's schema exposes `normalizedName` (`packages/indexer/graphql.gen.ts:103`), a field the Explorer's list query never selects. Opening that owner's page renders the raw row (`drive-015-recon.mjs`, non-local requests aborted): `"Name Copy TeStinG.eth July 23, 2027 5 Roles"` — the app, its indexer, and a row already there, nothing created or patched. On staging (re-run 2026-09-12), 44 rows are flagged `isNormalized:false`; the census classifies **7 as NFC-equivalent variants** (NFD vs NFC) that normally render the same.

### Confirmation 3: the full click-through, ending in a signed transaction

Local setup (`poc-015-setup-local-chain.mjs`) registers two names using a local script adapted from `e2e/fixtures/makeName.ts`: `ﬁle` to the victim (`0xf39F…2266`), `file` to the attacker (`0x7099…79C8`). They are distinct on-chain — `getRenewPrice` returns `160000009` for 4-char `file`, `640000005` for 3-char `ﬁle`. That the registrar accepts `ﬁle` (U+FB01) is part of the finding: the chain stores whatever label it is given.

`drive-015-extend.mjs` then connects the victim's wallet to the running Explorer and walks the flow. Abbreviated below (addresses and modal dumps shortened, `…` marks elisions); the full transcript is `poc-015-browser-output.txt`:

```
=== local chain, before ===
  victim   0xf39F…2266  owns ﬁle.eth   expiry 1820761799 (2027-09-12)
  attacker 0x7099…79C8  owns file.eth  expiry 1820761737 (2027-09-12)
  victim USDC balance: 10000000000

=== driving the Explorer ===
  row 0 as rendered: "Name Copy ﬁle.eth September 12, 2027"
  Extend settings modal: … ﬁle.eth … $160.11/yr … New expiry: Sep 12, 2028 … Total: $160.44 …
  confirmation modal: Confirm extension | ﬁle.eth | 1 year | New expiry Sep 12, 2028 | Total cost $160.44
  Transaction overview | Extend ﬁle.eth | …
  click Open wallet -> authorize eth_sendTransaction

=== transaction submitted to the ETHRegistrar ===
  tx    0x3dd7fc23c77139f46e1b5f921dbd1a68e9e0ee2e3778a9a1a9b83148b5526f8f
  from  0xf39f…2266   to  0xa88553f4…  (ETHRegistrar)
  decoded renew(label="file", duration=31622400, paymentToken=0x768F…7a39)
  ON SCREEN: ﬁle.eth   IN CALLDATA: file.eth
  status success
  event NameRenewed(label="file", newExpiry=1852384137 (2028-09-12), amount=160438366)

=== local chain, after ===
  victim   ﬁle.eth  expiry 1820761799   changed=false
  attacker file.eth expiry 1852384137   changed=true
  victim USDC balance: 9839561634   paid 160438366 (6 decimals)
```

Every screen up to the overview says `ﬁle.eth`; the transaction says `file`; the quote is `file`'s 4-char rate, not `ﬁle`'s 640; the attacker's `file.eth` gained a year while the victim's `ﬁle.eth` did not move; 160.438366 USDC left the victim; the app reported completion. At the default font U+FB01 renders as `fi`, so in the captured font the row and modals appear indistinguishable from `file.eth` (attached screenshots 01–05).

### One honest caveat about this run

**One response is patched.** `e2e/infra/panoptes/contracts.json` pins an older registry than the fixture uses, so the container logs the two registrations as "unknown contracts" and never returns them; the driver intercepts the owned-names response and prepends one row (`{"name":"ﬁle.eth","expiryDate":1820761799}`, the chain's real expiry), logged as `[route] patched owned-names response`. Every other query and row is the container's own. This supplies the *row*, not the *defect*, and is not what shows such rows exist — Confirmation 2 is, on unpatched data. The transcript above and the five click-through screenshots `01`–`05` are from the same run (2026-09-12), so their dates agree; the separate `recon-0x205d26.png` is from the earlier recon run (2026-09-04).

### The v1 half, which the v2-indexer objection does not cover

`poc-015-v1-path-signs-canonical.test.ts` — **3/3**. The likeliest closure is that the v2 indexer filters non-canonical rows from list queries. That covers only one of the table's **two** sources: `names.tsx:170-175` runs two queries and `:203` concatenates them, and the v1 half goes through `@ensdomains/ensjs/subgraph`, whose fragment carries **no `normalizedName`**. That does not make filtering impossible (the app could `ens_normalize` or compare labels), but the current v1 query receives no canonical field and the code performs no raw/canonical check on either path. The test drives the real ensjs encoder through the same expression `buildRenewIntent` uses:

```
v1 path -> contract 0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa (ETHRenewerV1)
DISPLAYED: ﬁle.eth    SIGNED: renew("file")
```

**Bound:** this exercises the encoder, not a full v1 flow; it does not show a non-canonical row returned by the v1 subgraph today (I did not query it), only that the code compares no labels.

### What is not demonstrated

* **I did not register through the Manager UI.** I registered directly against the same v2 `ETHRegistrar`, with the repo's e2e fixture flow. The claim that the Manager's register flow also produces such labels is read from source (`name-parser.ts:46`, no `ens_normalize`), not driven in a browser.
* **I did not find a live victim pair.** `TeStinG.eth` is a real non-canonical row, but its twin `testing.eth` exists on the fork as a v1-renewable name, not as a v2 registration renewable through the path this flow selects, so I cannot point at a live pair where the transaction would land. The Confirmation 3 pair is one I created.
* **I did not drive the bulk flow in a browser.** For it I executed the app's own `planMultiRenewSteps` and `computeRenewerPayments` over a basket and read out the titles, signed labels, and single aggregate approval; the raw-name surfaces on that path (`MultiNameConfirmCard.tsx:35`) are read from source.

### Supporting evidence

The executable scripts and text outputs listed below are included in the secret Gist. The PNG screenshots are attached separately to this report.

* `poc-015-real-hook-signs-different-label.test.tsx` (+ `poc-015-realhook-output.txt`) — **the fix-sensitive unit evidence** (Confirmation 1).
* `poc-015-renewal-signs-a-different-name.test.ts` — supporting unit coverage, incl. the bulk basket.
* `poc-015-canonicity-census.mjs` (+ `poc-015-census-output.txt`) — the population census, local and staging.
* `poc-015-setup-local-chain.mjs` — registers the two names on the fork.
* `drive-015-recon.mjs`, `drive-015-extend.mjs` (+ `poc-015-browser-output.txt`, `poc-015-recon-output.txt`) — the browser runs.
* `poc-015-twin-absent-reverts.sh` (+ `poc-015-twin-absent-output.txt`) — the attacker-step control (`cast call`/`eth_call`): an unregistered label reverts, the registered canonical twin's `eth_call` succeeds (detailed in the description).
* `poc-015-v1-path-signs-canonical.test.ts` (+ `poc-015-v1-output.txt`) — the v1-half structural proof.
* Attached screenshots: the five click-through PNGs from the 2026-09-12 run whose transcript is `poc-015-browser-output.txt` — `01-names-table`, `02-extend-settings`, `03-confirm-extension`, `04-transaction-overview`, `05-after` (the confirmation and transaction-overview modals are captured separately, `03`/`04`) — plus `recon-0x205d26.png`, from the separate earlier recon run (2026-09-04).

The browser scripts (copy the `.mjs`/`.sh` from the gist into `repo/e2e` first, where their imports resolve) need the Explorer pointed at the local stack: `VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:5655/graphql` and `VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8545` in `apps/portal/.env`, with `pnpm e2e:infra:up` running and `pnpm --filter portal dev -- --port 3001` serving the app.
