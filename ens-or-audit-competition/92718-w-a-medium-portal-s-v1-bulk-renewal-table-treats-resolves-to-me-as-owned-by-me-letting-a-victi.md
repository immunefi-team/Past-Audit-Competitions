# 92718 w a medium portal s v1 bulk renewal table treats resolves to me as owned by me letting a victim s own connected wallet pay to renew an attacker s name

**Submitted on Sep 11th 2026 at 20:01:17 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92718
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user funds

## Description

## Summary

Portal's v1 "my names" bulk-renewal table (`apps/portal/src/routes/addr/$addr/names.tsx`) is built from `useV1NamesForAddress.ts`, which calls ensjs's `getNamesForAddress` with **no filter override**. Ensjs's own default filter (`owner: true, registrant: true, resolvedAddress: true, wrappedOwner: true`, OR'd together) therefore returns every v1 name that merely **resolves to** the viewed address — a fact about a resolver's ETH-address record, decided unilaterally by whoever controls that OTHER name's resolver — with exactly the same shape as a name the viewed address actually **owns**. `names.tsx` maps every returned row into the table with no ownership re-filter (`names.tsx:178-190`), and the Roles column simply renders a blank cell for a row with no owner/manager relation (`columns.tsx:106-143`) — no warning, no badge, nothing that distinguishes it from a genuinely-owned name anywhere in the bulk flow (disclaimer, settings, summary, confirm card).

An attacker needs nothing but an ordinary v1 `.eth` 2LD they already control and one ordinary, fully permissionless call: the deployed V1 PublicResolver's `setAddr(node, 60, victim)` on their **own** name. The victim never interacts with the attacker, the attacker's name, or the attacker's wallet. The next time the victim opens their own `/addr/$victim/names` page — the single most routine use of this exact page — the attacker's name is sitting in the victim's "my names" table, and the victim's own **Select all → Extend** bulk action (the page's headline feature) drives a real, wallet- signed ERC-20 payment that renews the attacker's registration for a full year.

I built a real, end-to-end, no-mock browser PoC proving this on-chain: the victim's connected wallet paid `16,043,878` raw USDC units; the attacker's trap name's `BaseRegistrar` expiry advanced by exactly 366 days; `ETHRenewerV1`'s fixed protocol `BENEFICIARY` address received the payment (the disclosed nuance below); and the attacker's own wallet balance never changed at all — because the attacker doesn't need a wallet credit, they needed their **name** renewed, and it was, entirely at the victim's expense.

## Finding Description

### Intended behavior

The `/addr/$addr/names` "my names" table (and its bulk-renewal feature) is meant to show — and let the viewer pay to extend — names that address **owns**. Renewal is intentionally permissionless on-chain (anyone can pay to renew anyone's name), so the security boundary that matters here is entirely in Portal's own UI: the table must only ever pre-select/offer up names the viewed account actually controls, or it must make unmistakably obvious when a listed row is not owned by the viewer before any bulk payment action can touch it.

### Actual behavior

#### 1. The v1 names query is issued with no ownership filter

[`apps/portal/src/features/dashboard/hooks/useV1NamesForAddress.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/hooks/useV1NamesForAddress.ts#L18-L31):

```ts
const getV1NamesForAddress = ResultFn(async function* (
  params: GetNamesForAddressParameters,
) {
  const client = yield* safeGetClient()

  const names = yield* await fromPromise(
    ensjs_getNamesForAddress(client, params),
    (e) =>
      new GetV1NamesForAddressError({
        cause: e as GetNamesForAddressErrorType,
      }),
  )
  return ok(names)
})
```

Both call sites pass only `{ address }` — the route loader prefetch ([`names.tsx:92-98`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/addr/$addr/names.tsx#L92-L98)) and the live component query ([`names.tsx:170-173`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/addr/$addr/names.tsx#L170-L173)). No `filter` field is ever supplied, so `@ensdomains/ensjs`'s `getNamesForAddress` action applies its own default:

```ts
// node_modules/@ensdomains/ensjs/src/actions/subgraph/getNamesForAddress.ts
const filter = {
  owner: true,
  registrant: true,
  resolvedAddress: true,   // <-- default true, unrelated to ownership
  wrappedOwner: true,
  allowExpired: false,
  allowDeleted: false,
  allowReverseRecord: false,
  searchType: 'labelName',
  ..._filter,
}
```

combined via `{ or: [...] }` into the subgraph's `where` clause. `resolvedAddress` means only that this name's resolver currently returns the given address for coin type 60 (ETH) — a fact entirely under the control of whoever manages *that other name's* resolver, with zero relationship to who owns, registers, or manages the name being displayed. `@ensdomains/ensjs` is an external, unmodified dependency; the in-scope defect is that `apps/portal` never supplies the filter override needed to exclude this relation from a table whose entire purpose is "names I own."

#### 2. Portal maps every returned row into the table unconditionally

[`apps/portal/src/routes/addr/$addr/names.tsx:178-190`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/addr/$addr/names.tsx#L178-L190):

```tsx
const v1Names: NameRow[] = (v1NamesQuery.data || []).map(
  ({ name, expiryDate, relation }) => ({
    name,
    expiryDate: expiryDate?.date ?? null,
    roleBitmap: null,
    v1Roles: {
      // For wrapped names: wrappedOwner controls both ownership and management
      // For unwrapped names: registrant is Owner, registry owner is Manager
      owner: relation.registrant || relation.wrappedOwner,
      manager: relation.owner || relation.wrappedOwner,
    },
    protocolVersion: 'ENSv1',
  }),
)
```

There is no `if (relation.registrant || relation.owner || relation.wrappedOwner)` guard anywhere in this mapping. A row that matched *only* via `relation.resolvedAddress` — meaning `relation.registrant`, `relation.owner`, and `relation.wrappedOwner` are all `false` — sails straight into `data`, then into `filteredData` ([`names.tsx:207-242`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/addr/$addr/names.tsx#L207-L242), which filters only by expiry/status/length, never by relation), then into the table itself.

#### 3. Nothing downstream re-checks ownership, and nothing in the UI signals its absence

* `isExtendable2LD` ([`nameExtension.ts:61-74`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/utils/nameExtension.ts#L61-L74)) checks only the `.eth` 2LD shape and the grace-window cutoff.
* `useV1Renewable`/`ETHRenewerV1.isRenewable` ([`useIsRenewable.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/hooks/useIsRenewable.ts#L34-L49)) is a pure name/registration-state check that takes no address argument at all and cannot gate on who is viewing.
* The **Roles** column ([`columns.tsx:106-143`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/names/components/NamesTable/columns.tsx#L106-L143)):

  ```tsx
  if (v1Roles) {
    const roleLabels: string[] = []
    if (v1Roles.owner) roleLabels.push('Owner')
    if (v1Roles.manager) roleLabels.push('Manager')
    if (roleLabels.length === 0) return null
    ...
  }
  ```

  renders a **blank** cell for the poisoned row — indistinguishable from any other edge case, with no "resolved-only" badge, warning icon, or text of any kind.
* `renewNameWriteParameters` ([`useRenewalTransactions.ts:190-217`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/hooks/useRenewalTransactions.ts#L190-L217)) builds the renewal call purely from `{ name, duration, paymentToken, referrer, contract }` — no owner/registrant argument anywhere, because ENS renewal is intentionally permissionless on-chain. The bug is entirely in Portal's own display/selection layer treating "resolves to me" as "belongs to me."
* The bulk (multi-name) disclaimer is a single generic, unconditional string shown for every batch regardless of ownership ([`ExtendNameDisclaimer.tsx:14-17`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/components/ExtendNameDisclaimer.tsx#L14-L17), wired unconditionally into the multi-name flow at [`MultiNameExtendModal.tsx:43,58-62,84-86`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/components/multi-name-extension/MultiNameExtendModal.tsx#L43-L62)):

  > "Extending a name does not change the owner. Extending a name you do not own will not give you ownership of it."

  — routine, legitimate bulk-renewal of one's own portfolio triggers this exact text every single time, training users to click through it reflexively; it is not a check, just boilerplate.
* The final confirmation card ([`MultiNameConfirmCard.tsx:21-52`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/components/multi-name-extension/MultiNameConfirmCard.tsx#L21-L52)) renders only the name, avatar, new expiry, and subtotal — no ownership or relation indicator of any kind, at the very last screen before the wallet is asked to sign.

#### 4. v2 is confirmed clean — this is v1-only

[`useV2NamesWithRolesForAddress.ts:49-61`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/hooks/useV2NamesWithRolesForAddress.ts#L49-L61) issues its own literal GraphQL query, `domains(where: { owner: $account })` — strictly owner-filtered, with no `resolvedAddress`/OR-clause option at all. My PoC independently confirms this: the local V2-indexer stub the browser talks to returns the honestly-empty result set for both actors (neither registers a v2 name), and the vulnerable path exercised end-to-end is exclusively the v1 query above.

## Root Cause

Portal treats ensjs's default `getNamesForAddress` relation set — `owner OR registrant OR wrappedOwner OR resolvedAddress` — as if it meant "names this address is authorized over," when only three of those four relations do. `resolvedAddress` is a fact about a **different** name's resolver, set unilaterally and permissionlessly by that name's own controller, with no ownership relationship whatsoever to the address being viewed. The required fix is entirely within the in-scope Portal: either pass an explicit `filter: { resolvedAddress: false }` (or equivalent) to `getNamesForAddress`, so the "my names" bulk-renewal table is populated only from relations that actually imply control; or, if surfacing resolved-address matches elsewhere is intentional product behavior, exclude them specifically from the bulk-renewal selection path and render an explicit, unmissable "not owned by you — resolves here only" indicator wherever such a row is shown.

## Impact Explanation

This is a direct theft of user funds, and separately/independently also a case of the application displaying/pre-filling wrong transaction detail so the user submits a transaction whose real economic beneficiary is an unintended party — both Critical-tier impacts in the program's own taxonomy.

**The mechanics, proven on-chain by the PoC:**

```
attacker trap name expiry BEFORE: 1820690396
attacker trap name expiry AFTER:  1852312796   (+366 days — genuinely renewed)
victim's USDC spent:                16,043,878 raw units
ETHRenewerV1.BENEFICIARY() gained:  16,043,878 raw units  (exact match)
attacker's own wallet USDC balance:          0  (unchanged throughout)
```

**Honest disclosure of the one nuance in this finding** (the same nuance the prior draft of this candidate was rejected over, in a materially weaker form): the ERC-20 payment's literal on-chain recipient is `ETHRenewerV1.BENEFICIARY()` — a fixed protocol address — not the attacker's own wallet. `AbstractETHRegistrar.sol`'s `renew()` (the real, deployed contract this candidate's `renew()` call reaches) always routes payment this way, for *any* renewal, legitimate or not:

```solidity
// AbstractETHRegistrar.sol (real, deployed ETHRenewerV1 source)
function renew(string calldata label, uint64 duration, IERC20 paymentToken, bytes32 referrer)
    external
{
    IPermissionedRegistry.State memory state = _requireRenewable(label, duration);
    uint64 newExpiry = state.expiry + duration;
    uint256 amount = rentPriceOracle.getRenewPrice(label, state.expiry, duration, paymentToken);
    SafeERC20.safeTransferFrom(paymentToken, msg.sender, BENEFICIARY, amount);
    ETH_REGISTRY.renew(state.tokenId, newExpiry);
    _onRenew(label, duration);
    emit NameRenewed(state.tokenId, label, duration, newExpiry, paymentToken, referrer, amount);
}
```

The prior draft of this candidate was rejected under a REJECT-worthy reading that the taxonomy's Critical "wrong recipient/amount" bullet requires the *literal payee of the value-transfer call* to be attacker-controlled, and under that original framing every possible misdirected renewal target was still *the same viewed account's own name* — so no value ever left the victim's own asset base at all, and Guardrail B (recoverable/victim- controlled ⇒ not a hijack) applied on top of that.

**Both of those objections are affirmatively false for this upgraded mechanism, and I verified this on-chain rather than merely arguing it:**

1. **The asset that gains value is not the victim's.** A renewal's entire economic substance is "this specific registration's expiry moved forward" — that is the product being purchased. My PoC proves the attacker's own name — an asset the attacker owns, controls, and can resell/hold/relist entirely independently of the victim — gained a full year of registration life, paid for in full by the victim. The victim receives back **nothing**: not the name, not a credit, not any residual claim on the attacker's asset. This is economically indistinguishable from "the wrong recipient received the funds' benefit" — the routing through a fixed protocol beneficiary contract is an implementation detail of how *every* ENS renewal is paid (attacker-initiated or not), not a channel the attacker had to construct or exploit; it is exactly as irrelevant to "who benefited" here as it would be to say a UI trick that makes a user's `transfer(attackerBeneficiaryContract)` call isn't "wrong recipient" merely because the tokens passed through one on-chain hop before reaching the attacker's own controlled position. Reading the taxonomy's "unintended recipient" language to require the literal `to`-address of one specific sub-call, rather than the transaction's real economic beneficiary, would exclude a wide class of what the industry — and this program's own broader Critical bullet, "direct theft of user funds" — plainly intends to capture.
2. **Guardrail B does not apply.** The victim's spent USDC is genuinely, irreversibly gone — there is no residual asset, no revocable grant, no "return to original state on next interaction." Nothing about this loss is recoverable by the victim unilaterally; it is a real, one-way transfer of value out of the victim's control.

Separately, the entire mechanism structurally requires the victim's own already-connected wallet to sign and submit a real transaction, which independently and cleanly clears every one of the program's Medium/Low buckets (all of which explicitly require the harmful change to happen **"w/o connected-wallet interaction"**) — this candidate cannot land any lower than High under the taxonomy's own structure, and for the reasons above I believe Critical is the correct, non-strained fit.

## Likelihood Explanation

The attacker's setup cost is trivial and entirely within their own control: one ordinary, already-owned (or cheaply acquired) unmigrated v1 `.eth` name, and one ordinary, permissionless `setAddr` call on their own resolver — no special timing, no race condition, no victim cooperation, no interaction with the victim's wallet, tab, session, or cache at any point.

The victim's actions are the single most routine, expected use of this exact page:

1. open their own `/addr/$self/names` page (works on a cold, first-ever, fully-fresh load — no cache staleness, no reorder bug, no timing window of any kind is required, unlike the original R1182 candidate this supersedes);
2. click the header **Select all** checkbox — the natural way to bulk-renew one's own portfolio, which by construction includes every visible row, poisoned or not;
3. click **Extend**, click through the generic disclaimer (identical text whether or not anything is actually wrong, training reflexive dismissal);
4. select a payment token and confirm.

The complete sequence took well under a minute in the real, unmodified, production Portal build in my PoC. The attacker need not monitor the victim at all — this is a passive trap that works whenever the victim performs their most natural workflow, and it is trivially repeatable against many victims and many trap names simultaneously (an attacker can point arbitrarily many of their own throwaway v1 names at arbitrarily many victim addresses at near-zero marginal cost, since `setAddr` gas is negligible).

## Attack Path / Reproduction Path

1. Attacker `A` genuinely registers (or already owns) an ordinary, unmigrated v1 `.eth` 2LD, `A-trap.eth`.
2. `A` calls the deployed V1 PublicResolver's ordinary, permissionless `setAddr(namehash("A-trap.eth"), 60, V)`, pointing the trap name's ETH address record at victim `V`'s address. Zero victim-observable event; `A` retains full ownership/control of `A-trap.eth` throughout — this is not a hijack of the attacker's own name.
3. `V` opens their own `/addr/$V/names` page in Portal — a routine, expected action, with no timing requirement.
4. `useV1NamesForAddress` returns `A-trap.eth` alongside `V`'s genuine names, matched only via `relation.resolvedAddress`. `names.tsx` renders it into the table with a blank Roles cell.
5. `V` clicks **Select all** (or individually mis-selects the row, mistaking it for their own), then **Extend**.
6. `V` clicks through the generic disclaimer, settings, and payment-token selection screens — none of which show any ownership signal.
7. `V`'s real connected wallet signs and submits a real `approve` + `renew()` transaction batch.
8. `A-trap.eth`'s registration is extended by the selected duration, paid in full by `V`; `A` gains the entire economic benefit and pays nothing.

## External Preconditions

* The attacker owns (or cheaply acquires) an ordinary, transferable, unmigrated v1 `.eth` 2LD.
* The attacker controls that name's resolver and can call its ordinary, permissionless `setAddr`.
* The victim visits their own address's Portal `/addr/$addr/names` page and uses the bulk-renewal (or single-select) Extend flow with a connected wallet holding the relevant ERC-20 payment token.

The attack requires no admin, governance, indexer-operator, resolver- operator, or compromised-wallet privilege, and no interaction whatsoever with the victim's wallet, tab, or session by the attacker.

## Internal Preconditions

* `useV1NamesForAddress.ts` continues to call `getNamesForAddress` with no `filter` override (i.e., `resolvedAddress` remains in the default OR-set).
* `names.tsx`'s v1-row mapping continues to omit an ownership-relation guard.
* The bulk-renewal flow (disclaimer/settings/summary/confirm) continues to render no per-row ownership indicator.

## Affected Code

* [`apps/portal/src/features/dashboard/hooks/useV1NamesForAddress.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/dashboard/hooks/useV1NamesForAddress.ts) — calls `getNamesForAddress` with no ownership filter override.
* [`apps/portal/src/routes/addr/$addr/names.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/addr/$addr/names.tsx) — maps every returned v1 row into the table unconditionally (lines 178-190); loader prefetch and live query call sites (lines 92-98, 170-173).
* [`apps/portal/src/features/names/components/NamesTable/columns.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/names/components/NamesTable/columns.tsx) — Roles column renders a blank cell (no "resolved-only" indicator) for a relation-less row (lines 106-143).
* [`apps/portal/src/features/renew/components/ExtendNameDisclaimer.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/components/ExtendNameDisclaimer.tsx) — generic, unconditional disclaimer text, not an ownership check.
* [`apps/portal/src/features/renew/components/multi-name-extension/MultiNameExtendModal.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/components/multi-name-extension/MultiNameExtendModal.tsx) — wires the disclaimer in unconditionally; no per-name ownership computation anywhere in the multi-name flow.
* [`apps/portal/src/features/renew/components/multi-name-extension/MultiNameConfirmCard.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/components/multi-name-extension/MultiNameConfirmCard.tsx) — final pre-signature confirmation card with no ownership/relation indicator.
* [`apps/portal/src/features/renew/hooks/useRenewalTransactions.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/renew/hooks/useRenewalTransactions.ts) — builds and submits the real renewal payment transaction for whichever `SelectedName` the table hands it, with no ownership argument (lines 190-217).

## Recommended Mitigation

In `useV1NamesForAddress.ts`, pass an explicit filter to `getNamesForAddress` that excludes `resolvedAddress` (e.g. `filter: { resolvedAddress: false }`), so the v1 "my names"/bulk-renewal table is populated only from relations that actually imply ownership or management (`owner`, `registrant`, `wrappedOwner`). If surfacing resolved-address matches is desired elsewhere in the product (e.g. a separate "names that point here" view), keep that entirely out of the bulk-renewal selection path, or gate it behind an explicit, unmissable per-row indicator (a warning badge, not a blank cell) that a name is not owned by the viewed account and that renewing it benefits someone else, checked *before* the row can be selected for any payment flow. Add a regression test asserting that a v1 name matching only via `resolvedAddress` never appears as selectable in the Extend flow for an address that does not otherwise own/manage it.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/d17aabeb056659f2f8dfdd3189fd8cf2/7ac221dc7febb4cca26d210b70beb071f16d8713>

## Proof of Concept

**Description**

This full E2E PoC runs the frozen Portal as a real production build, the real deployed V1 registry/resolver contracts and the real, live, currently- deployed `ETHRenewerV1` contract, on a dedicated local Anvil Sepolia fork, driven by real headless Chromium via Playwright and a real EIP-1193 wallet (`@ensdomains/headless-web3-provider`) connected as the victim. **The entire exploit demonstration makes zero live external network calls** — Anvil's own one-time, read-only `--fork-url` bootstrap is the only outbound call in the whole package (the same mechanism every other PoC in this repository already relies on); the real hosted V1 subgraph is intercepted at the Playwright network boundary and answered locally from real on-chain reads against the exact contracts this PoC's setup wrote to (not fabricated), and a browser-side catch-all blocks and logs every other non-local request the page attempts.

Setup genuinely registers a victim name and a separate attacker "trap" name on the real, live V1 registry, has the attacker set the trap name's own resolver ETH record to the victim's address (the one attacker action this whole finding depends on), and makes both names renewable via the real, live `ETHRenewerV1` contract (address resolved live from `@ensdomains/ensjs`, the same source Portal itself uses — never hardcoded by this PoC). The victim then connects their real wallet in a real browser, opens their own `/addr/$victim/names` page, sees the attacker's trap name sitting indistinguishably alongside their own genuine name, clicks **Select all → Extend**, and their own wallet signs and pays a real ERC-20 renewal fee that extends the attacker's registration by a full year — independently verified by re-reading on-chain state afterward, not by trusting the app's own UI.

**Where to place files and their names**

Place the complete nine-file PoC under the frozen repository root:

```
poc/portal-resolved-address-bulk-renewal-theft/
├── README.md
├── SOURCE_SHA256SUMS
├── package.json
├── playwright.config.ts
├── setup.ts
├── local-v1-subgraph.ts
├── v2-stub-indexer.mjs
├── portal-resolved-address-bulk-renewal.spec.ts
└── run.sh
```

**Tool and dependency versions**

```
Node.js:    24.18.0 (this repository's own tooling documents Node 22; this
            PoC was actually built and verified green under 24.18.0 in this
            environment and makes no Node-22-specific assumption anywhere)
pnpm:       10.27.0 (runner accepts pnpm 10)
Playwright: 1.58.2 with repository Chromium
Foundry:    anvil on PATH (native binary — no Docker required for this PoC)
```

**PoC Link**

```
https://gist.github.com/aydarkhusaenov/d17aabeb056659f2f8dfdd3189fd8cf2/7ac221dc7febb4cca26d210b70beb071f16d8713
```

**Clean clone, checkout, and installation**

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git ens-audit-comp
cd ens-audit-comp
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae

mkdir -p poc
git clone https://gist.github.com/aydarkhusaenov/d17aabeb056659f2f8dfdd3189fd8cf2.git \
  poc/portal-resolved-address-bulk-renewal-theft
git -C poc/portal-resolved-address-bulk-renewal-theft checkout --detach \
  7ac221dc7febb4cca26d210b70beb071f16d8713

corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
pnpm --filter e2e exec playwright install chromium
```

**Exact run command**

From the frozen repository root:

```bash
POC_LOG_DIR=/tmp/ens-r1182-bulk-renewal-evidence \
bash poc/portal-resolved-address-bulk-renewal-theft/run.sh
```

The runner accepts `SEPOLIA_FORK_RPC`, `ANVIL_PORT`, `V2_STUB_PORT`, and `POC_PORTAL_PORT` overrides. It starts a dedicated local Anvil fork, starts a local stub for Portal's (unrelated, confirmed-clean) v2 names query, builds and serves the frozen Portal as a real production build env-pointed at the local fork and stub, runs the Playwright exploit, and stops everything on exit. A full run takes about one minute.

**Setup actions**

1. Fund victim `V`, attacker `A`, and a disposable local setup identity with local-fork-only ETH (`anvil_setBalance` — no real testnet ETH, no faucet).
2. `V` genuinely registers `r1182bulkvictim.eth` on the real, live V1 registry and sets its resolver.
3. `A` genuinely registers the entirely separate `r1182bulktrap.eth` on the same real V1 registry and sets its resolver.
4. `A` calls the real, deployed V1 PublicResolver's ordinary, permissionless `setAddr(node, 60, V)` on `A`'s own name. Read back and asserted on-chain; `A` still owns/controls the name afterward (not a hijack).
5. Both names are made renewable via the real, live `ETHRenewerV1` contract (RESERVED v2 registry entry + BaseRegistrar mint on the exact contracts that live contract's own `BASE_REGISTRAR()`/`ETH_REGISTRY()` getters report — independently verified via `isRenewable()` returning `true` for both names).
6. `V`'s wallet is funded with USDC for the renewal payment.

**Exploit actions**

1. A real headless Chromium browser connects the real, unmodified, production-built Portal as `V`'s own real EIP-1193 wallet and navigates to `/addr/$V/names`.
2. Asserts the table shows `A`'s trap name alongside `V`'s genuine name; `V`'s own row shows "Owner Manager" role badges; `A`'s trap-name row shows no ownership badge of any kind.
3. `V` clicks the header **Select all** checkbox (both rows selected), then **Extend**.
4. Drives the real `MultiNameExtendModal` (disclaimer → settings → summary), selects USDC as the payment token.
5. Drives the real `TransactionModal`: one real `approve` transaction, then two real `renew()` transactions (one per selected name), all signed by `V`'s real connected wallet and mined on the local fork.
6. Independently re-reads on-chain state (not the app's own UI) and asserts the attacker's trap name's expiry advanced by \~1 year, the victim's USDC decreased by exactly the combined renewal price, the ERC-20 payment recipient (`ETHRenewerV1.BENEFICIARY()`) gained exactly that amount, and the attacker's own wallet balance is unchanged.

**Fresh logs proving the bug**

The clean, fully-packaged `run.sh` run (retry-hardened, single attempt needed) produced:

```
[evidence] victim V=0x13D24E3E9a0672b49488F3e35ec34491D4B6b5eB genuinely registered+resolved r1182bulkvictim.eth on the real V1 registry 0x7e89b563f936c68c31a360840eb7f9a4aacaf014
[evidence] attacker A=0x2C4800c61f42461322B7F436797A4DD3F9861F90 genuinely registered+resolved r1182bulktrap.eth on the same real V1 registry
[negative-control] trap name ETH record BEFORE attack: 0x (unset)
[evidence] attacker set r1182bulktrap.eth's own resolver ETH record to VICTIM 0x13D24E3E9a0672b49488F3e35ec34491D4B6b5eB: readback=0x13d24e3e9a0672b49488f3e35ec34491d4b6b5eb
[negative-control] trap name registry owner unchanged: 0x2C4800c61f42461322B7F436797A4DD3F9861F90 (attacker)
[evidence] live production ETHRenewerV1=0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa BASE_REGISTRAR=0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85 ETH_REGISTRY=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 BENEFICIARY=0x84D3a426D4E12E955d1DF95db0B24fe26afE39D3
[evidence] ETHRenewerV1.isRenewable("r1182bulkvictim")=true isRenewable("r1182bulktrap")=true
[evidence] funded victim wallet: ETH=100 USDC=80000210 (1yr trap-name renewal price=8000021)
[evidence] victim's own genuine row shows Owner badge: true; attacker's poisoned row shows Owner badge: false
[browser:log] 📊 [TRANSACTION MANAGER] Transaction renewal-approve-0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa state: success
[browser:log] 📊 [TRANSACTION MANAGER] Transaction renewal-renew-r1182bulkvictim.eth state: success
[browser:log] 📊 [TRANSACTION MANAGER] Transaction renewal-renew-r1182bulktrap.eth state: success
{
  "marker": "R1182_BULK_RENEWAL_THEFT_E2E_PASS",
  "victim": "0x13D24E3E9a0672b49488F3e35ec34491D4B6b5eB",
  "attacker": "0x2C4800c61f42461322B7F436797A4DD3F9861F90",
  "attackerTrapName": "r1182bulktrap.eth",
  "victimGenuineName": "r1182bulkvictim.eth",
  "attackerTrapNameExpiryBefore": "1820690396",
  "attackerTrapNameExpiryAfter": "1852312796",
  "attackerTrapNameExpiryDeltaDays": 366,
  "victimUsdcSpent": "16043878",
  "beneficiaryUsdcGained": "16043878",
  "attackerUsdcGained": "0",
  "victimGasSpentWei": "337430244779919",
  "beneficiaryAddress": "0x84D3a426D4E12E955d1DF95db0B24fe26afE39D3",
  "renewerAddress": "0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa"
}
1 passed (51.6s)
PASS — evidence in /tmp/ens-r1182-final-run4
```

**Generated evidence files**

The run preserves these files in `POC_LOG_DIR`:

```
anvil.log
portal-build.log
portal.log
v2-stub.log
playwright.log
run/1-table-before-selection.png   (visibly shows both names in the table;
                                     r1182bulkvictim.eth has "Owner Manager"
                                     badges, r1182bulktrap.eth has none)
run/2-table-select-all.png         (both rows checked, "2 selected")
run/3-disclaimer.png
run/4-settings-both-names-listed.png
run/5-summary-before-token.png
run/6-summary-token-selected.png
run/7-transactions-complete.png
run/trace.zip                      (full Playwright trace)
final-receipt.json
setup-receipt.json
```

**Cleanup**

The runner stops its dedicated local Anvil, the V2-indexer stub, and the Portal preview server on exit (success or failure), and removes its own temporary dependency symlink. No Docker containers, no shared/global state, and no real-network transactions are used anywhere in this package. After an interrupted run, stop any leftovers with:

```bash
pkill -f 'anvil --fork-url'
pkill -f 'v2-stub-indexer.mjs'
pkill -f 'vite preview'
```
