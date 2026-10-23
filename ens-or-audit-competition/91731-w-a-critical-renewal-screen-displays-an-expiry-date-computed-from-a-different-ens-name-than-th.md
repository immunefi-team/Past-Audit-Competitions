# 91731 w a critical renewal screen displays an expiry date computed from a different ens name than the one the transaction renews

**Submitted on Sep 3rd 2026 at 21:31:15 UTC by @risinghorse for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91731
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager route `/renew/$name` canonicalises the same route parameter two different ways inside one loader: `parseRenewableName(name)` applies only `trim().toLowerCase()` and produces the `label` that ends up as the `renew()` calldata argument, while `profileExpiryQuery(name)` applies ENSIP-15 normalization and produces the `currentExpiry` that drives the eligibility gates and every date shown on the confirmation screen. For any input where the two canonicalisations differ, the renewal screen therefore renders "Renewing for … expiring on `<date>`" where the date is computed from the on-chain registration of a **different name** than the one the transaction will renew. In production this means a user can approve and pay for a renewal whose displayed expiry outcome is factually wrong for the name being renewed, and can conclude that a name was renewed when it was not.

***

**Severity:** Low **Asset:** `apps/manager` (ENS Manager app) — in scope per the program's Assets in Scope **Target:** ENS Manager & Explorer — Audit Competition **Audited revision:** `63772fd872af472ced58b009499355f3430c2a86`

**Selected impact:** *Causing the application to display or pre-fill an incorrect address, name, or transaction detail* — the *"incorrect transaction detail"* limb only. See **Impact Details** for what is deliberately **not** claimed.

> **Testing & rules compliance.** No testing was performed against mainnet or any public testnet. This is a client-side data-flow defect; it is demonstrated below as a static trace over the in-scope source at the audited revision, with an optional reproduction on a **local chain only**. No public-testnet or mainnet contract was registered, called, or written to.

> **Duplicate risk, stated up front.** The permissive validator this finding depends on (`parseName`) is the validator described by Known Issue **EXP-INPUT-005**. I am filing this narrowly on the one consequence that entry does not describe — an expiry date rendered for a *different name* than the one being renewed — and I set the severity to Low accordingly. If triage judges that consequence to fall inside EXP-INPUT-005's scope, this report is a duplicate and I do not contest that.

***

## Vulnerability Details

### The split

`apps/manager/src/routes/renew/$name.tsx:19-28`

```ts
loader: async ({ params: { name }, context: { queryClient } }) => {
  const parsedName = parseRenewableName(name)

  if (parsedName.isErr()) {
    throw parsedName.error
  }

  const expiryData = await queryClient.ensureQueryData(
    profileExpiryQuery(name),
  )
```

Both calls receive the same raw `name`. They canonicalise it differently:

* `parseRenewableName` → `parseName`, whose only canonicalisation is `apps/manager/src/features/register-v2/utils/name-parser.ts:46`:

  ```ts
  const normalized = name.trim().toLowerCase()
  ```

  Its only character filter is an ASCII-punctuation blacklist (`name-parser.ts:11`) with no `u` flag and no non-ASCII ranges, so every non-ASCII codepoint passes:

  ```ts
  const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/
  ```
* `profileExpiryQuery` → `getExpiry` → `normalizeEth2LdName` → `normalizeEthName`, which calls viem's `normalize` (`apps/manager/src/features/profile/service/profileName.ts:1, 18`). Every downstream on-chain read uses `ethName.name` (`profileExpiry.ts:88, 94, 106`).

The loader then returns one field from each (`routes/renew/$name.tsx:60-63`):

```ts
return {
  label: parsedName.value.label,     // raw canonicalisation
  currentExpiry: expiryData.expiry,  // ENSIP-15 canonicalisation
}
```

There is no `beforeLoad` and no `params.parse` on this route or its parent (`routes/renew/$name.tsx:18, 65-67`; `routeTree.gen.ts:83-87`), and no gate anywhere in the loader compares the two canonicalisations against each other.

### `label` becomes the calldata argument

Traced hop by hop, with no normalization at any step:

| # | File                                                             | Line      | What happens                                                |
| - | ---------------------------------------------------------------- | --------- | ----------------------------------------------------------- |
| 1 | `routes/renew/$name.tsx`                                         | 61, 72    | `label` returned, passed to `<RenewalPage label={label} …>` |
| 2 | `features/renew/workflow/RenewalPage.tsx`                        | 22-26     | passed to `<RenewalUiProvider label={label} …>`             |
| 3 | `features/renew/state/renewalUi.context.tsx`                     | 53-55     | `value={{ uiActor, label, currentExpiry, protocol }}`       |
| 4 | `features/renew/workflow/pricing/components/ConfirmPurchase.tsx` | 14, 63-72 | `uiActor.send({ type: 'renewal.start', label, … })`         |
| 5 | `features/renew/state/renewalUi.machine.ts`                      | 151, 349  | `label: event.label` → `label: submission.label`            |
| 6 | `packages/transaction-manager/.../registration.actors.ts`        | 1072-1076 | `encodeRenewData(input.label, …)`                           |

`packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1031`

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

`.replace('.eth', '')` is a no-op here because `parseName` already split on `.`.

### `currentExpiry` becomes every date on screen

`apps/manager/src/features/renew/workflow/pricing/components/PricingSummaryCard.tsx:9-22, 37-43`

```tsx
export const PricingSummaryCard = () => {
  const { uiActor, currentExpiry } = useRenewalUiContext()
  const duration = useSelector(uiActor, (state) => state.context.duration)
  const currentExpirationDate = useMemo(
    () => new Date(Number(currentExpiry) * 1000),
    [currentExpiry],
  )

  const newExpirationDate = useMemo(() => {
    return getDurationExpiryDateForDisplay(
      Number(duration),
      currentExpirationDate,
    )
  }, [currentExpirationDate, duration])
  // ...
        <span>
          <Trans>expiring on</Trans>
        </span>
        <span className="font-normal text-[#024A70]">
          {format(newExpirationDate, 'MMMM d, yyyy')}
        </span>
```

`DurationSelector.tsx:23-31` uses the same `currentExpiry` as the reference date for every "renew to" preset, and `RenewalDetails.tsx:15, 23-32, 69` reuses it on the in-flight screen.

**So the rendered expiry outcome describes the ENSIP-15-normalized name, while the transaction renews the raw label.** That is the defect.

### The gates are also evaluated against the other name

Of the six loader gates, four are driven purely by the normalized name's on-chain data — the v1 redirect (`:30-36`), `protocol !== 'v2'` (`:40-42`), `isPastGracePeriod` (`:44-50`) and the expiry-present check (`:56-58`). One is a raw-label syntax check (`:22-24`). `canRenewV2Name` (`:52`) is a hybrid: `features/renew/utils/renewableName.ts:44-46` re-parses the **raw** name and hands `${parsed.value.label}.eth` to `isRenewableV2EthName`, which only shape-tests it (`features/grace/utils/gracePeriod.ts:48`):

```ts
.with({ name: P.when((value) => !/^[^.]+\.eth$/.test(value)) }, () => false)
```

No `u` flag, no charset restriction. No gate compares the raw label to the normalized label.

### Recommended fix

Canonicalise once. In `routes/renew/$name.tsx`, derive the label from the same ENSIP-15-normalized string used for the expiry read — or reject the request when `normalize(name) !== parseName(name).label`, so the flow cannot proceed on two different names. The one-line version is to build `label` from the normalized name rather than from `parsedName.value.label` at `:61`.

***

## Impact Details

### What actually goes wrong

A user who opens `/renew/<L>.eth`, where `normalize(L) ≠ toLowerCase(L)` and the attacker has already registered the raw label `L`, is shown a renewal screen on which:

* the **name** is `L` — correct for the transaction (see below);
* the **price** is `L`'s price — correct, and correctly charged;
* the **expiry dates** ("Renewing for … expiring on `<date>`", and every duration preset) are computed from `normalize(L)`'s registration — **wrong for the name being renewed**.

If they confirm, they pay a correct amount to renew `L`, and their own name `normalize(L)` is not renewed despite the screen having described its expiry throughout. The secondary consequence is that a user may believe their name was renewed and allow it to lapse.

### Claimed impact

* ✅ **Causing the application to display … an incorrect … transaction detail** — the expiry date rendered on the confirmation screen belongs to a different name than the one the transaction renews.

### Explicitly NOT claimed

I ran an adversarial review of this finding before filing, and it refuted several claims I would otherwise have made. Stating them as refuted:

* ❌ **"The user is shown their own name while signing for another."** False. Every name string the flow renders is the **raw** label — `PricingDomainHeader` renders `${label}.eth` untruncated (`PricingStep.tsx:24, 27`), the confirm modal renders `truncateName(label, 'eth', 10)` plus `title={domainName}` (`ConfirmPurchase.tsx:198, 200`), the in-flight screen renders `${submissionData.label}.eth` (`RenewalDetails.tsx:50`), and the transaction description is `Renew ${input.label}.eth` (`registration.actors.ts:1090`). The displayed name and the signed name **agree**. Any name-level deception rests on a human not noticing a homoglyph, which is EXP-INPUT-005, not this report.
* ❌ **"There is no guard on the raw label."** False. The pricing query prices the **raw** label (`renewPricing.query.ts:36-42`, `ConfirmPurchase.tsx:22`, `PaymentCard.tsx:46`), and `getRenewPrice` → `_requireRenewable` reverts `NameNotRenewable(label)` unless that raw label is registered or in grace (`AbstractETHRegistrar.sol:102-113, 131-143`). The query error propagates, `pricingQuery.data` is undefined, and the Renew button is disabled (`ConfirmPurchase.tsx:44-50`; `ConfirmPurchaseBase` `disabled=` at `:235`). **The flow fails closed unless the attacker pre-registered the exact raw label.**
* ❌ **Direct theft of user funds.** False. `renew()` sends the rent to `BENEFICIARY` — "Address that receives payments" (`AbstractETHRegistrar.sol:26-27, 90`), i.e. the ENS treasury, not the attacker. The attacker receives nothing; their only gain is extended registration on a label that is by construction not ENSIP-15-normalized and therefore unresolvable by conformant clients.
* ❌ **Price or amount mismatch.** False. `StandardRentPriceOracle.getBasePrice` prices by Unicode codepoint count (`:363-372`), so a same-length homoglyph quotes an identical price, and the quoted price is exactly what is charged. Unlike QA-03, this flow has no price discrepancy.
* ❌ **Modifying transaction arguments against a connected wallet.** Not earned. The argument built is faithful to the URL the user opened; nothing alters a transaction the user legitimately initiated.
* ❌ **Hijacking or bricking an ENS name.** Not applicable — that bullet enumerates registration, v1-to-v2 migration and transfer; renewal is not listed, and the victim's own name is untouched.

### Severity

**Low.** The direct loss is one renewal fee on a name the user did not intend to renew; the attacker gains nothing of value; and the flow fails closed unless the attacker has pre-registered the exact raw label. What remains is a genuine data-integrity defect — a transaction detail displayed from the wrong name — with a second-order risk that the user believes a renewal occurred and lets their real name lapse. I considered High and Critical and reject both: no funds reach an attacker, and no listed Critical bullet is satisfied.

### Prerequisites

The attacker must have registered the exact non-normalized label on-chain, and the victim must open an attacker-supplied `/renew/…` URL. If ENSIP-15 **disallows** (rather than maps) the chosen codepoint, `normalize()` throws, `normalizeEthName` returns null (`profileName.ts:19-21`), `getExpiry` short-circuits to `{ expiry: null }` (`profileExpiry.ts:79-85`), and the loader throws "This name is not available for renewal." at `:53` — the flow fails closed.

### Duplicate analysis

**EXP-INPUT-005 is the real risk and I am not going to argue around it.** `parseName` is one of the validators that entry describes, down to the leading/trailing-dot behaviour (`name-parser.ts:58`, `split('.').filter(Boolean)`). The Known Issues document states that a different exploitation path or entry point is *not* sufficient to make a report new. The only applicable carve-out is "new consequences of a listed root cause that materially change its severity," and the consequence I am claiming — an expiry date rendered from a different name than the one being renewed — is a data-integrity defect in the loader rather than the "homograph-style display confusion" EXP-INPUT-005 scopes itself to. That is a narrow distinction and triage may reasonably reject it.

Checked and rejected as duplicates: **QA-05** (renewal blocked after registration — a false *negative* gate; opposite direction), **WEB-1255** (wrong grace window for the right name; also appears remediated at this revision — `gracePeriod.ts:12-13` now takes an explicit `protocol`), **WEB-626** (modal copy for grace-period names, correct calldata), **QA-03** (Explorer price formatting; different app, and this flow has no price mismatch), and **R3-01…R3-08** (all downstream of an already-built request; none touch calldata construction).

***

## References

In-scope code (audited revision `63772fd872af472ced58b009499355f3430c2a86`):

* `apps/manager/src/routes/renew/$name.tsx:19-28, 38-63, 69-74` — the dual canonicalisation, the gates, the loader return
* `apps/manager/src/features/register-v2/utils/name-parser.ts:11, 44-58` — `parseName`; `trim().toLowerCase()` and the ASCII-only blacklist
* `apps/manager/src/features/renew/utils/renewableName.ts:25-36, 40-47` — `parseRenewableName`, `canRenewV2Name`
* `apps/manager/src/features/grace/utils/gracePeriod.ts:41-55` — `isRenewableV2EthName` shape test
* `apps/manager/src/features/profile/service/profileExpiry.ts:73-88, 94, 106, 127-131` — normalized on-chain expiry read
* `apps/manager/src/features/profile/service/profileName.ts:1, 14-22, 62-73` — `normalizeEthName` / `normalizeEth2LdName`
* `apps/manager/src/features/renew/workflow/RenewalPage.tsx:22-26`, `features/renew/state/renewalUi.context.tsx:53-55`, `renewalUi.machine.ts:151, 349` — label propagation
* `apps/manager/src/features/renew/workflow/pricing/components/ConfirmPurchase.tsx:14, 22, 44-50, 63-72` — price query and the `renewal.start` event
* `apps/manager/src/features/renew/workflow/pricing/components/PricingSummaryCard.tsx:9-22, 37-43` — the displayed expiry
* `apps/manager/src/features/renew/workflow/pricing/components/DurationSelector.tsx:23-31` — duration presets from `currentExpiry`
* `apps/manager/src/features/renew/data/queries/renewPricing.query.ts:36-42` — price fetched for the raw label
* `packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1031, 1072-1076, 1090` — `encodeRenewData` and the renewal description
* `apps/manager/src/routeTree.gen.ts:83-87` — route registration, no param parser

Contracts (reference; out of scope themselves):

* `contracts-v2/contracts/src/registrar/AbstractETHRegistrar.sol:26-27, 84-94, 102-113, 131-143` — `BENEFICIARY`, `renew`, `getRenewPrice`, `_requireRenewable`
* `contracts-v2/contracts/src/registrar/StandardRentPriceOracle.sol:363-372` — pricing by Unicode codepoint count

Specifications:

* ENSIP-15, ENS Name Normalization — <https://docs.ens.domains/ensip/15>
* viem `normalize` — <https://viem.sh/docs/ens/utilities/normalize>

## Proof of Concept

## Proof of Concept

### Part 1 — Static proof of the split (no chain required)

For any input `N` where `normalize(N) !== N.trim().toLowerCase()`:

| Value                                  | Derived from                     | Source                                              |
| -------------------------------------- | -------------------------------- | --------------------------------------------------- |
| `label` → `renew()` argument           | `N.trim().toLowerCase()`         | `$name.tsx:61` → `registration.actors.ts:1029`      |
| `currentExpiry` → every displayed date | `normalize(N)`'s on-chain expiry | `$name.tsx:62` → `PricingSummaryCard.tsx:12-22, 41` |

The two describe different ENS nodes, and no gate in the loader compares them.

### Part 2 — Confirming the input class exists (offline)

The attack requires a codepoint that `String.prototype.toLowerCase()` leaves unchanged but ENSIP-15 **maps**. Verified with node (built-ins only, no libraries):

```
codepoint                        toLowerCase changes?   NFKC changes?
U+FF56 fullwidth small v         false                  true   -> 'v'
U+2170 roman numeral small one   false                  true   -> 'i'
ASCII 'V' (control)              true                   false
emoji + U+FE0F                   false                  false
```

So `vⅰctim.eth` (U+2170, which renders as an ordinary `i`) passes `parseName` verbatim while normalizing to `victim.eth`.

**Caveat I want to be explicit about:** `NFKC` is a *proxy* for the UTS-46 mapping ENSIP-15 inherits, not ENSIP-15 itself, and `node_modules` is not installed at this revision so I could not read `@adraffy/ens-normalize` to confirm the mapping table. If ENSIP-15 *disallows* rather than maps the chosen codepoint, `normalize()` throws and the flow fails closed as described under Prerequisites. A reviewer with dependencies installed can settle this in one line:

```js
import { normalize } from 'viem/ens'
normalize('vⅰctim.eth')   // expect 'victim.eth' -> attack viable
                               // throws              -> fails closed
```

I did **not** verify the U+FE0F variation-selector case and make no claim about it.

### Part 3 — Optional local-chain reproduction

Against the repository's **local** Anvil fork only (`pnpm e2e:infra:up`, chain on `127.0.0.1:8545`; point the Manager at it with `VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8545` per `e2e/infra/docker-compose.yml:8` and `apps/manager/src/lib/wagmi.ts:25`):

1. Register the ASCII label `victim` from one account, and the label `vⅰctim` (U+2170) from a second account. Both are legal registrations — the registrar validates byte length only.
2. Open `http://localhost:3000/renew/vⅰctim.eth`.
3. Observe on the pricing screen: the heading renders the U+2170 label (visually `vⅰctim.eth`), the price is the U+2170 label's price, and **"expiring on `<date>`" is computed from `victim.eth`'s registration**, not from the label being renewed.
4. Confirm and observe that `victim.eth`'s expiry is unchanged while the U+2170 label's expiry is extended.

**No public testnet or mainnet is involved.**
