# 92794 w a critical manager s bulk renew dialog substitutes a normalized lookalike for the victim s real registered label letting a victim s own connected wallet pay to renew an attacker s name

**Submitted on Sep 12th 2026 at 10:56:42 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92794
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

`apps/manager/src/features/dashboard/components/NamesTable.tsx`'s `toBulkRenewName` builds the on-chain `renew()` calldata's `label` argument from `domain.normalizedName` — the ENSIP-15-canonical, lowercased form of a name — while every render path in the same dialog (`resolveDomainLabel`, `NamesBreakdown.tsx`) shows only `domain.name`, the domain's real, raw, case-preserving registered label. These are two already-distinct, separately-populated fields the (out-of-scope) indexer exposes on every `Domain` record (`packages/indexer/documents/fragments/Domain.graphql`); Manager's own choice of which field to trust for which purpose is squarely first-party and in-scope.

A victim who owns a V2 registration whose raw registered label is not ENSIP-15-normalized (e.g. mixed-case) — the standing, ordinary end-state of migrating a V1-era mixed-case name through Manager's own in-scope migration wizard, which carries a V1 name's raw label through with zero normalization anywhere in the path — opens their own Manager Dashboard, selects their own real name (shown correctly, mixed-case, throughout), and clicks **Renew**. An attacker who has separately, and entirely passively, registered the lowercase counterpart of that exact label gets **their own** registration renewed instead: the mined `renew()` transaction's ABI-encoded `label` argument is the attacker's lowercase label, not the victim's real raw label, at every step of the dialog — selection, summary, confirm, and the "Renewal Complete" success receipt.

I built a real, end-to-end, no-mock PoC proving this on-chain: the victim's connected wallet paid `8,021,939` raw USDC units to renew a name that was never theirs; the attacker's registration's real on-chain expiry advanced by exactly `31,622,400` seconds (366 days — the selected 1-year duration); the victim's own real registration's expiry did not move at all; and the attacker's own wallet balance never changed — they didn't need a credit, they needed their **registration** renewed, and it was, entirely at the victim's expense.

## Finding Description

### Intended behavior

Manager's dashboard bulk-renew dialog is meant to renew the exact registration it displays to the user. A domain's raw, as-registered bytes (`domain.name`) are the only identity that can be correct to submit on-chain: ENS's V2 `ETHRegistrar.renew(label, ...)` computes `labelhash = keccak256(bytes(label))` from whatever string is passed, with no on-chain normalization anywhere — non-normalized (e.g. mixed-case) `.eth` registrations are a real, standing, independently-owned condition, which is exactly why the indexer exposes both a raw `name` field and a separately-computed `normalizedName` field on every `Domain` in the first place.

### Actual behavior

#### 1. `toBulkRenewName` substitutes the normalized field for the calldata label

`apps/manager/src/features/dashboard/components/NamesTable.tsx:59-93`:

```ts
type SelectableDomain = {
  readonly id: string
  readonly name?: string | null
  readonly normalizedName?: string | null
  readonly expiryDate?: number | null
}

const selectionKey = (domain: SelectableDomain): string =>
  resolveDomainLabel(domain).toLowerCase()

const isSelectableDomain = (domain: SelectableDomain): boolean =>
  isRenewableV2Domain(
    domain.normalizedName ?? domain.name ?? '',
    domain.expiryDate,
  )

const toBulkRenewName = (domain: SelectableDomain): BulkRenewName | null => {
  if (!isSelectableDomain(domain)) return null
  const displayName = resolveDomainLabel(domain)
  // Use the canonical normalized name for the on-chain label — the display name
  // may be unnormalized (e.g. mixed case) and would hash to the wrong label.
  const name = domain.normalizedName ?? displayName
  return {
    displayName,
    label: name.replace(/\.eth$/i, ''),
    name,
    currentExpiry: BigInt(domain.expiryDate as number),
  }
}
```

`resolveDomainLabel` (`apps/manager/src/features/dashboard/utils.ts:42-46`):

```ts
export const resolveDomainLabel = (domain: {
  id: string
  name?: string | null
  normalizedName?: string | null
}): string => domain.name ?? domain.normalizedName ?? domain.id
```

`displayName` (line 83) prefers `domain.name` — the raw, as-registered bytes. `name` (line 86), the value that becomes the on-chain `label`, prefers `domain.normalizedName` — a separately-computed ENSIP-15-canonical string. The comment on line 84-85 shows the author *knew* the display name can be unnormalized — but the fix is backwards: if a domain's true on-chain registration used non-normalized bytes, the only correct `renew()` argument is those exact bytes (the same identity the on-chain contract itself hashes with no cross-check, as `ETHRegistrar.renew` never re-derives or validates `label` against anything). Substituting the normalized form computes the labelhash of a **different, independently-existing registration**.

Critically, no parser runs anywhere in this code path — no `normalize()`, no `parseName()`. `domain.name` and `domain.normalizedName` are two already-distinct fields the indexer populates once, verbatim, on every `Domain`; Manager's bug is purely picking the wrong one for the calldata.

#### 2. The display path never shows the substituted identity

`apps/manager/src/features/bulk-renew/components/NamesBreakdown.tsx:37,40`:

```tsx
<NameAvatar name={row.label} preview={row.preview} />
...
{row.displayName}
```

`row.displayName` (the real, mixed-case string) is the only thing ever rendered — on the summary row, the confirm step, the in-flight renewing step (`RenewingStep.tsx` reuses the same rows), and the success receipt (`SuccessStep.tsx` reuses the same rows). `row.label` — the substituted string that actually reaches the chain — is never shown anywhere in this dialog.

#### 3. The substituted label drives the price quote too — internally consistent, still wrong identity

`apps/manager/src/features/bulk-renew/hooks/useBulkRenew.ts:99-100,155`:

```ts
const presetQueries = useQueries({
  queries: PRESETS.flatMap((preset) =>
    names.map((n) => ({
      ...getRenewPriceQueryOptions(
        n.label,
        ...
const activeQueries = useQueries({
  queries: names.map((n) => {
    ...
    return {
      ...getRenewPriceQueryOptions(n.label, Number(duration), selectedToken),
```

Both the preset-summary and active-selection price reads use `n.label` — the same substituted value that reaches the chain. The displayed price is exactly what the mined transaction actually charges, which makes the dialog look entirely trustworthy to the victim: a correct-looking quote for a name-length that matches what they see, computed for the wrong identity throughout.

#### 4. The submission path signs and broadcasts the substituted label unmodified

`apps/manager/src/features/bulk-renew/hooks/useBulkRenewSubmit.ts:76-83`:

```ts
const renewOne = (ctx: Context, item: RenewItem): ResultAsync<void, Error> =>
  submitRenewActor({
    label: item.label,
    duration: item.duration,
    selectedToken: ctx.token,
    signer: ctx.signer,
    publicClient,
  })
```

`packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1031`:

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

`item.label` traces straight back to `toBulkRenewName`'s substituted `label` field with zero transformation in between. The victim's real wallet (`submitRenewActor`, lines 1039-1099) signs and broadcasts this exact calldata.

#### 5. Precondition reachability: Manager's own migration wizard produces the exact victim state

Manager's own `/register/$name` flow always lowercases via `parseName` before anything reaches the chain, so an ordinary registration through Manager cannot by itself produce a mixed-case V2 registration. But Manager's own, first-party, in-scope **migration** flow does, with zero external tool required:

`packages/migration/src/service/classifyNames.ts:132`:

```ts
const label = domain.labelName
```

`apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:479,486-487`:

```ts
const name = classified.domain.name
...
const migrationData = createMigrationData({
  label: classified.label,
  owner: wallet,
  subregistry: zeroAddress,
  resolver: expectedResolver,
})
```

`domain.labelName` is the V1 subgraph's raw label field — the literal label text as it was registered on ENS V1, case preserved. There is no `.toLowerCase()`/`normalize()` call anywhere between `domain.labelName` and the final ABI-encoded migration calldata. ENS V1's own registrar never enforced ENSIP-15 either, so mixed-case `.eth` registrations from the pre-normalization era are a real, historically-documented condition. A completely ordinary victim who registered a mixed-case name years ago on V1 and later uses Manager's own, first-party, in-scope **Migrate to V2** wizard ends up owning a V2 registration whose raw on-chain label is still mixed-case — no external tool, no crafted input, and no action outside Manager's own normal, intended workflow is needed anywhere in this precondition.

## Root Cause

`toBulkRenewName` (`apps/manager/src/features/dashboard/components/NamesTable.tsx:86`) picks `domain.normalizedName` — a separately-computed, ENSIP-15-canonical string — as the value that reaches the on-chain `renew()` call's `label` argument, while every render path in the same dialog shows `domain.name`, the domain's real, raw, as-registered bytes. Because ENS's V2 registrar computes `labelhash` purely from whatever bytes it is given, with no normalization or cross-check against the domain's real identity, this substitution silently redirects the transaction to a completely different, independently-existing registration whenever the two fields differ (any case difference is sufficient). The fix is entirely within this one function: use the domain's real, raw label (`resolveDomainLabel(domain)`, the same value already used for `displayName`) for the calldata too, never `domain.normalizedName`.

## Impact Explanation

This is the taxonomy's Critical "displays/pre-fills wrong ... transaction detail so a user sends funds ... to an unintended recipient" bucket, and separately/independently a direct theft of user funds — both Critical-tier in the program's own taxonomy.

**The mechanics, proven on-chain by the PoC** (independent, from-scratch re-run; full JSON in `poc/manager-bulkrenew-normalization-substitution/evidence/verify.log`):

```
victim's real registration expiry BEFORE: 1820765589
victim's real registration expiry AFTER:  1820765589   (completely unchanged)
attacker's registration expiry BEFORE:    1820765673
attacker's registration expiry AFTER:     1852388073   (+31,622,400s = 366 days — genuinely renewed)
victim's USDC balance BEFORE:             999,991,994,499 (raw units)
victim's USDC balance AFTER:              999,983,972,560 (raw units)
victim's USDC spent:                      8,021,939 raw units (~$8.02)
mined renew() tx: 0xd3b09d91b3a8c064dbb8e2b17065249c2354ac03cf7b32cd32abf5add9853187
  from (victim's own wallet):  0xb2a567d8befe6fa4f04729679854c9790cef8f33
  decoded calldata label:      "pocvictimbb4e42ce"   (the ATTACKER's real registered label)
  victim's real registered label was: "PocVictimbb4e42ce"  (NEVER submitted on-chain)
```

* **The asset that gains value is not the victim's.** A renewal's entire economic substance is "this specific registration's expiry moved forward." The PoC proves the attacker's own registration — an asset the attacker owns, controls, and can hold/resell entirely independently of the victim — gained a full year of registration life, paid for in full by the victim. The victim receives back nothing: not the name, not a credit, not any residual claim on the attacker's asset.
* **The victim's real name is left exposed.** The victim's own `PocVictimbb4e42ce.eth` registration's expiry never moved — the exact renewal they were trying to perform silently did not happen. If the victim's real name was near the end of its own renewal window, it now lapses for the price the victim just paid specifically to prevent that outcome.
* **No fail-closed backstop exists anywhere in this chain.** `isRenewableV2Domain`/`isRenewableV2EthName` (the only gate on this path) is a pure format/expiry check on whatever string it is handed — it never cross-checks the final calldata label against the victim's real registered bytes, and the on-chain contract itself performs no normalization or cross-check either.
* **Guardrail-style recoverability does not apply.** The victim's spent USDC is genuinely, irreversibly gone; retrying the exact same bulk-renew (or single-renew) action via Manager repeats the identical substitution every time, since there is no code path anywhere in this family that ever sends the victim's raw, non-normalized label on-chain — this bug does not self-heal on a follow-up attempt.
* The entire mechanism structurally requires the victim's own already-connected wallet to sign and submit a real transaction, which independently clears this program's Medium/Low buckets (which require the harmful change to happen without connected-wallet interaction).

## Likelihood Explanation

The attacker's setup cost is a single, ordinary, cheap V2 registration of the lowercase counterpart of any mixed-case label they can observe — no special privilege, no interaction with the victim's wallet, tab, session, or cache at any point, and no per-victim timing dependency:

1. The attacker scans the public indexer for any V1 `.eth` domain whose `labelName` is mixed-case and not yet migrated (or any already-migrated V2 `Domain` where `name != normalizedName`), owned by anyone.
2. The attacker registers the lowercase counterpart on V2 today, through Manager's own public `/register/$name` flow — an ordinary registration at ordinary price, since V2 is a fresh registry and virtually every such counterpart is currently unregistered.
3. The attacker does nothing else and waits, indefinitely. Whenever that specific victim (a) migrates their old V1 name to V2 through Manager's own wizard, and (b) later opens Manager's dashboard and bulk-renews (or singly renews) it, the substitution fires automatically and deterministically on the very first attempt.

The victim's own actions are the single most routine, expected use of Manager's dashboard: open `/dashboard`, select their own real name (shown correctly), click **Renew**, confirm. No phishing link, no crafted URL, and no attacker-supplied input of any kind is involved in the trigger — the input is the victim's own ownership data, reached by using their own dashboard normally. This is a completely passive, scalable attack: one attacker can pre-register the lowercase counterpart of every mixed-case name they find on the public indexer, at near-zero marginal cost per target, and simply wait for any of those victims to migrate and later renew.

## Attack Path / Reproduction Path

1. Victim `V` owns a real V2 `.eth` registration whose raw registered label is mixed-case (e.g. `"PocVictimXYZ"`) — the ordinary end-state of migrating an old V1 name through Manager's own migration wizard.
2. Attacker `A`, with no special access, separately registers the exact lowercase counterpart (`"pocvictimxyz.eth"`) through Manager's own public `/register/$name` flow.
3. `V` opens their own Manager Dashboard, sees their real name `"PocVictimXYZ.eth"` listed with its correct expiry, selects it, and opens Bulk Renew (or renews it alone — a batch of one reaches the identical code path).
4. The dialog shows `"PocVictimXYZ.eth"` throughout: summary row, confirm step, in-flight renewing step, success receipt. `V` approves USDC and confirms.
5. `useBulkRenewSubmit.submit` → `renewOne` → `submitRenewActor({ label: "pocvictimxyz", ... })` → `encodeRenewData` ABI-encodes `renew("pocvictimxyz", duration, paymentToken, referrer)`. The mined transaction extends `A`'s `"pocvictimxyz.eth"` registration's expiry. `V`'s real `"PocVictimXYZ.eth"` registration is completely untouched.
6. `V` sees "Renewal Complete" and walks away believing their name was renewed. It was not.

## External Preconditions

* The victim owns (via any ordinary path — most concretely, Manager's own in-scope V1-to-V2 migration wizard) a real V2 `.eth` registration whose raw registered label is not ENSIP-15-normalized.
* The attacker separately, and passively, registers the exact ENSIP-15-normalized counterpart of that label, through Manager's own ordinary public registration flow.
* The victim uses Manager's own dashboard bulk-renew (or single-renew) flow with a connected wallet holding the relevant ERC-20 payment token.

The attack requires no admin, governance, indexer-operator, or compromised-wallet privilege, and no interaction whatsoever with the victim's wallet, tab, or session by the attacker.

## Internal Preconditions

* `toBulkRenewName` (`NamesTable.tsx`) continues to prefer `domain.normalizedName` over `resolveDomainLabel(domain)` for the on-chain `label`.
* No consumer in the bulk-renew chain (`useBulkRenew.ts`, `useBulkRenewSubmit.ts`, `registration.actors.ts`) cross-checks the final calldata label against the domain's real raw registered bytes before submission.

## Affected Code

* [`apps/manager/src/features/dashboard/components/NamesTable.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/dashboard/components/NamesTable.tsx#L59-L93) — `toBulkRenewName` substitutes `domain.normalizedName` for the calldata label (lines 81-93); `isSelectableDomain`/`selectionKey` share the same normalized-preferring inputs (lines 70-78).
* [`apps/manager/src/features/dashboard/utils.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/dashboard/utils.ts#L42-L46) — `resolveDomainLabel`, the correct (raw) value that is never used for the calldata.
* [`apps/manager/src/features/bulk-renew/components/NamesBreakdown.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/bulk-renew/components/NamesBreakdown.tsx#L37-L40) — renders only `row.displayName`; `row.label` (the substituted, actually-submitted value) is never shown.
* [`apps/manager/src/features/bulk-renew/hooks/useBulkRenew.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/bulk-renew/hooks/useBulkRenew.ts#L96-L164) — prices, and builds `renewItems`, from the substituted `n.label` (lines 99-100, 155, 182).
* [`apps/manager/src/features/bulk-renew/hooks/useBulkRenewSubmit.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/bulk-renew/hooks/useBulkRenewSubmit.ts#L76-L83) — `renewOne` submits `item.label` unmodified.
* [`packages/transaction-manager/src/machines/registration/registration.actors.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.actors.ts#L1020-L1099) — `encodeRenewData`/`submitRenewActor` ABI-encode and sign whatever `label` they are given, with no cross-check.
* [`packages/migration/src/service/classifyNames.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L132) — `label = domain.labelName`, carrying a V1 name's raw case through unmodified (precondition reachability).
* [`apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L479-L487) — the migration calldata that ends the victim up owning the mixed-case V2 registration.

## Recommended Mitigation

In `toBulkRenewName` (`NamesTable.tsx`), use the domain's real, raw registered label (`resolveDomainLabel(domain)`, the same value already used for `displayName`) for the on-chain `label` too — never `domain.normalizedName`. If a defensive check against genuinely malformed/absent `domain.name` values is desired, compare the two fields and refuse to render the row as selectable (with a clear message) rather than silently substituting a different identity. Add a regression test asserting that for a `Domain` fixture where `name !== normalizedName`, `toBulkRenewName(...).label` equals the label derived from `name`, never from `normalizedName`, and that the resulting `renew()` calldata's `label` argument matches what is displayed at every step of the dialog.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/e4afce873c884366f6f4587c84dca6b0/04f37599eb7abdbc90e64a9e991c715720444727>

## Proof of Concept

**Description**

This full PoC drives the real, unmodified `NamesTable` → `BulkRenewDialog` → `NamesBreakdown` / `useBulkRenew` → `useBulkRenewSubmit` → `registration.actors.ts` chain through the project's own real Vitest + React Testing Library toolchain (the same tool this repository's own `NameRow.test.tsx` / `BulkRenewDialog.test.tsx` / `useBulkRenewSubmit.test.ts` already use), against real deployed V2 ENS contracts on a dedicated local Sepolia fork. **The entire exploit demonstration makes zero live external network calls** — Anvil's own one-time, read-only `--fork-url` bootstrap is the only outbound call in the whole package, the same mechanism every other PoC in this repository already relies on.

Setup genuinely registers a victim's mixed-case V2 `.eth` name and a separate attacker's lowercase counterpart, using only the app's own real, exported `registration.actors.ts` functions (the identical pipeline Manager's real `/register/$name` UI drives). The victim's real dashboard data is then rendered through the real, unmodified component chain; the victim selects their own real (correctly displayed) name, opens Bulk Renew, and confirms. The victim's own wallet (a real viem `WalletClient` bound to the local fork, using the standard Anvil `anvil_impersonateAccount` cheatcode — no private key anywhere in this process) then signs and broadcasts a real transaction, mined on the local fork. A separate, independent process then re-reads on-chain state from scratch (real `NameRenewed` event logs, a fresh decode of the mined transaction's calldata) and confirms: the attacker's registration's expiry advanced by exactly the selected duration; the victim's own registration's expiry never moved; the mined `renew()` call's `label` argument is the attacker's real registered label, never the victim's own.

**Where to place files and their names**

Place the complete PoC under the frozen repository root:

```
poc/manager-bulkrenew-normalization-substitution/
├── README.md
├── SOURCE_SHA256SUMS
├── setup.ts
├── bulkrenew-substitution.test.tsx
├── verify.ts
├── run.sh
└── evidence/
```

**Tool and dependency versions**

```
Node.js: 24.18.0
pnpm:    10.27.0
Vitest:  4.1.5 (this repository's own real test toolchain)
Foundry: anvil on PATH (native binary — no Docker required for this PoC)
```

**PoC Link**

```
https://gist.github.com/aydarkhusaenov/e4afce873c884366f6f4587c84dca6b0/04f37599eb7abdbc90e64a9e991c715720444727
```

**Clean clone, checkout, and installation**

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git ens-audit-comp
cd ens-audit-comp
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae

mkdir -p poc
git clone https://gist.github.com/aydarkhusaenov/e4afce873c884366f6f4587c84dca6b0.git \
  poc/manager-bulkrenew-normalization-substitution
git -C poc/manager-bulkrenew-normalization-substitution checkout --detach \
  04f37599eb7abdbc90e64a9e991c715720444727

corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
```

**Exact run command**

From the frozen repository root:

```bash
POC_LOG_DIR=/tmp/ens-bulkrenew-substitution-evidence \
bash poc/manager-bulkrenew-normalization-substitution/run.sh
```

The runner accepts `SEPOLIA_FORK_RPC` and `ANVIL_PORT` overrides. It starts a dedicated local Anvil fork, runs the three phases below, and stops everything on exit. A full run takes well under a minute.

**Setup actions**

1. Fund a victim and an attacker address (freshly generated, `anvil_impersonateAccount`'d for real signature-free local submission — no real testnet ETH, no faucet, no private key ever held by this process) with local-fork-only ETH (`anvil_setBalance`) and MockUSDC (the token's own real, permissionless `mint()`).
2. The victim genuinely registers a real V2 `.eth` name whose raw label is mixed-case (`PocVictim<random>`), end to end (resolver deploy → commit → wait `MIN_COMMITMENT_AGE` → approve → register), using only the app's own real `registration.actors.ts` functions.
3. The attacker genuinely registers the exact lowercase counterpart (`pocvictim<random>.eth`) through the identical real pipeline.
4. Both registrations' real on-chain expiries are read and recorded (via the same `@ensdomains/ensjs` v2 `getExpiry` helper Manager's own `profileExpiry.ts` uses).

**Exploit actions**

1. The real, unmodified `NamesTable` component is rendered (via the project's own real Vitest + RTL setup) with the victim's owned-domains data fixture describing exactly this state (`name: "PocVictim<random>.eth"`, `normalizedName: "pocvictim<random>.eth"`).
2. Asserts the dashboard row shows the victim's real, mixed-case name.
3. Selects that row and opens **Bulk Renew** — the real `toBulkRenewName` executes.
4. Asserts the summary step shows only the victim's real name.
5. Clicks **Next**, waits for the real on-chain price read to resolve, asserts the confirm step still shows only the victim's real name.
6. Clicks **Confirm** — the real `useBulkRenewSubmit().submit()` calls the real `submitApprovalActor` / `submitRenewActor` / `encodeRenewData`, signing and broadcasting real transactions from the victim's own wallet to the local Anvil fork.
7. Waits for the real "Renewal Complete" success receipt — still showing only the victim's real name.
8. A separate, independent process re-reads on-chain state from scratch: real `NameRenewed` event logs, a fresh decode of the mined transaction's calldata, fresh expiry/balance reads.

**Fresh logs proving the bug**

The clean, independently re-run `run.sh` (fresh Anvil fork, fresh setup, fresh UI stage, fresh verification process — Step 9 of this program's own methodology; single attempt needed) produced, in its final independent-verification phase:

```
[verify] found 1 NameRenewed event(s) on 0xa88553F454b77203B0D036A05c894d555EAAa2Cc
[verify] renewed labels (from real on-chain NameRenewed events): [ 'pocvictimbb4e42ce' ]
[verify] decoded renew() calldata from mined transactions: [
  {
    txHash: '0xd3b09d91b3a8c064dbb8e2b17065249c2354ac03cf7b32cd32abf5add9853187',
    from: '0xb2a567d8befe6fa4f04729679854c9790cef8f33',
    label: 'pocvictimbb4e42ce'
  }
]
{
  "victim": {
    "address": "0xb2A567d8bEFE6fA4F04729679854C9790CEF8f33",
    "label": "PocVictimbb4e42ce",
    "expiryBefore": "1820765589",
    "expiryAfter": "1820765589",
    "usdcBalanceBefore": "999991994499",
    "usdcBalanceAfter": "999983972560",
    "usdcSpent": "8021939"
  },
  "attacker": {
    "address": "0xE8741a3C897836B8fCBb709410d083f792f9aD27",
    "label": "pocvictimbb4e42ce",
    "expiryBefore": "1820765673",
    "expiryAfter": "1852388073",
    "expiryExtendedBySeconds": "31622400"
  },
  "renewTransactions": [
    {
      "txHash": "0xd3b09d91b3a8c064dbb8e2b17065249c2354ac03cf7b32cd32abf5add9853187",
      "from": "0xb2a567d8befe6fa4f04729679854c9790cef8f33",
      "label": "pocvictimbb4e42ce"
    }
  ],
  "checks": {
    "victimExpiryUnchanged": true,
    "attackerExpiryExtended": true,
    "victimSignedRenewForAttackerLabel": true,
    "victimNeverSignedRenewForOwnLabel": true,
    "victimPaidRealMoney": true
  }
}
ENS_BULKRENEW_SUBSTITUTION_PASS

PASS — full evidence in <POC_LOG_DIR>
```

and, from the UI stage immediately before it:

```
POC_UI_STAGE_PASS
 ✓ src/features/dashboard/components/.poc-bulkrenew-substitution-generated.test.tsx > bulk-renew normalization-substitution PoC (real code, real Anvil fork) > misdirects the victim's real renewal payment to the attacker's registration
 Test Files  1 passed (1)
      Tests  1 passed (1)
```

Note the victim's own real registered label was `"PocVictimbb4e42ce"` (mixed-case) throughout every screen of the dialog; the transaction their own wallet actually signed encodes `"pocvictimbb4e42ce"` (the attacker's real, separately-registered, all-lowercase name) — confirmed by an independent decode of the mined transaction's calldata in a fresh process, not by trusting the application's own UI or the test process's own claims.

**Generated evidence files**

The run preserves these files in `POC_LOG_DIR` (and are bundled, from an independent re-run, in this package's own `evidence/`):

```
anvil.log      (the local fork's own log)
setup.log      (Phase 1 — on-chain registration setup)
ui-stage.log   (Phase 2 — the real Vitest/RTL UI-stage run, full output)
verify.log     (Phase 3 — the independent on-chain verification, full JSON summary)
fixture.json   (addresses/labels/expiries recorded by setup.ts)
```

**Cleanup**

`run.sh` stops its dedicated local Anvil, removes the generated `apps/manager/src/features/dashboard/components/.poc-bulkrenew-substitution-generated.test.tsx`, and removes its own temporary `node_modules` symlink, on both success and failure. No Docker containers, no shared/global state, and no real-network transactions are used anywhere in this package. After an interrupted run, stop any leftovers with:

```bash
pkill -f 'anvil --fork-url'
```
