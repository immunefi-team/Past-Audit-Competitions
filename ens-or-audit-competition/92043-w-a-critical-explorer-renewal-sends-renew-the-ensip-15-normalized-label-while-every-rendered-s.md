# 92043 w a critical explorer renewal sends renew the ensip 15 normalized label while every rendered string is the raw on chain label the user pays full rent their own name gains zero seconds&#x20;

**Submitted on Sep 6th 2026 at 15:57:18 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92043
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs

## Description

## Summary

The Explorer's renewal flow reads one name two ways.

* Every rendered string — the dashboard row, the confirmation card, the avatar, the transaction title, the new-expiry preview — is `domain.name`, the **verbatim on-chain label** as the indexer returns it.
* The `label` argument in the `ETHRegistrar.renew(...)` calldata is `getLabel(name)`, which is `ens_split(ens_normalize(name))[0]` — the **ENSIP-15-normalized** label.

The v2 registrar takes `label` as a verbatim string and registers and renews labels byte-for-byte, so a non-canonical label and its normalized twin are **two different names with two different registrations**. For any owned name whose label is not already canonical, the flow charges the user for one name and extends the other.

The shipped comment on the defective line has the direction backwards:

```ts
// ensjs splits the label without normalizing, so pass a normalized 2LD name.
```

ensjs not normalizing is correct and is the *reason the raw label must be sent*: the raw label **is** the on-chain name. Normalizing before the call does not repair a name, it retargets the payment.

The price is quoted on the normalized label too (`useRegistrationPrice.ts:53-54` runs the same `getLabel`), so the amount displayed and the amount charged agree exactly. That is what makes the divergence invisible before signature: nothing on screen is arithmetically wrong. The only untrue thing on screen is *which registration the money buys*, and that string appears nowhere in the application.

This is not a hypothetical population. On Sepolia today there are **eight** live labels whose normalized twin is registered **to a different address**, with no relationship between the parties — and four of those eight differ from their twin only by Unicode composition, so the two names are **visually identical**.

***

## Impact

**There is no attacker in this mechanism.** The victim is any Explorer user who owns a v2 `.eth` name whose on-chain label is not ENSIP-15-canonical — a mixed-case label, or one differing only by Unicode composition — and who renews it from their own names page. The beneficiary is whoever holds the normalized twin, who takes no action and need not know the victim exists.

1. The victim opens `/addr/<their address>/names`, their own names list.
2. They select their own name and press Extend. The row, the confirmation card and the avatar all read the name as they registered it.
3. They approve the ERC-20 spend for the quoted amount, then sign `renew(...)`.

| Step the user sees                                     | What is actually executed                                                                       | File:line                                                                                                |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| A row and confirmation card reading `Crypto.eth`       | the string is `domain.name`, the raw indexer label                                              | `useV2NamesWithRolesForAddress.ts:56,86` → `nameExtension.ts:20-22` → `ExtendNameConfirmation.tsx:32-33` |
| `Est. Cost` / total `$8.00`, new expiry `Sep 18, 2027` | quoted for `crypto`, **not** for `Crypto`                                                       | `useRegistrationPrice.ts:53-54` (`getLabel(name)`)                                                       |
| Transaction step titled `Extend Crypto.eth`            | title built from the raw name                                                                   | `useRenewalTransactions.ts:291-292` (multi), `:458-460` (single)                                         |
| **Extend this name**                                   | `renew("crypto", 31536000, USDC, 0x00…)` — a **different registration**, owned by a third party | **`useRenewalTransactions.ts:197`**                                                                      |

The victim loses the full rent (8,000,021 raw USDC units for one year on the pinned deployment). Their own registration gains **zero seconds** and continues to expire on its original date. A stranger's registration gains a full year that the stranger did not pay for and cannot be made to return.

The victim's only remedy is to pay a second time, and **the Explorer offers no path that does it** — every renewal the app builds goes through the same `getLabel`. Recovering therefore costs 16,000,042 units to obtain the one year the first 8,000,021 was meant to buy.

*A note on the duration figure.* The transcripts below use a 365-day year (31,536,000s → 8,000,021 units). The app's own helper, `getDurationInSecondsFromYears(1)`, returns **31,557,600s** (a 365.25-day year), priced at 8,005,501 units. The defect is identical on either number and has been executed on both; the smaller figure is simply the one the primary transcript was captured with.

***

## Root cause

### The normalized branch: the money

| File:line                                                                    | Code                                                                                                                                                                                                                                                                                      |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:191-197`** | <p><code>// ensjs splits the label without normalizing, so pass a normalized 2LD name.</code><br><code>const writeParams = renewNameWriteParameters(publicClient, {</code><br><code>name: ``</code>${getLabel(params.name)}.eth` ``, ← <strong>the one line that must change</strong></p> |
| `apps/portal/src/utils/token/getLabel.ts:10-19`                              | `const normalized = ens_normalize(name)` then `ens_split(normalized)`, returning `String.fromCodePoint(...first.input)`                                                                                                                                                                   |
| `apps/portal/src/features/register/hooks/useRegistrationPrice.ts:53-54`      | `const label = yield* fromSync(() => getLabel(name), …)` — the **quote runs the same normalization**, which is why the displayed amount is internally consistent                                                                                                                          |

### The raw branch: everything the user sees

| File:line                                                                         | Code                                                                                                                      |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `apps/portal/src/features/dashboard/hooks/useV2NamesWithRolesForAddress.ts:52-56` | `domains(where: { owner: $account }) { name expiryDate … }` — the query selects `name`; it never selects `normalizedName` |
| `apps/portal/src/features/dashboard/hooks/useV2NamesWithRolesForAddress.ts:86`    | `name: domain.name`                                                                                                       |
| `apps/portal/src/features/renew/utils/nameExtension.ts:20-22`                     | `.map((row) => ({ name: row.name, isV2: …, expiryDate: … }))`                                                             |
| `apps/portal/src/features/renew/components/ExtendNameConfirmation.tsx:32-33`      | `<NameAvatar name={selectedName.name} …/>` and `<h2 …>{selectedName.name}</h2>`                                           |
| `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:291-292`          | `` title: `Extend ${step.name}` `` / `` transactionName: `Extend ${step.name}` ``                                         |
| `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:458-460`          | `id`, `title` and `transactionName` for the single-name flow                                                              |

### Why the two branches disagree

`@ensdomains/ensjs` does not normalize, and the repo says so in its own comment on the defective line. The v2 `ETHRegistrar` accepts `label` as a verbatim string with no character validation, so the label the registrar stores **is** the bytes it was given. Two consequences follow, and the code acts on only one of them:

1. A non-canonical label is a *real, distinct, owned name*. It is not a malformed spelling of the canonical one.
2. Therefore normalizing on the write side does not sanitize the request — it **redirects it to a different registration**.

The Explorer's own `normalizedName` handling makes the divergence structural rather than accidental. The indexer exposes a `normalizedName` field; **`apps/portal` never selects it and never reads it** — the string appears nowhere in `apps/portal/src` except as a local variable computed by `normalize()` in the unrelated reverse-resolution and history features. The Manager, by contrast, has the field in hand — `normalizedName` appears 44 times in `apps/manager/src`, `domain.normalizedName` specifically 6 times. The Explorer therefore has only the raw label available to display, and manufactures the normalized one at the moment of payment.

**The sibling that gets it right, in the same application.** `/$name` refuses to serve a non-canonical name at all: `routes/$name/index.tsx:442,458-459` computes `isValidEnsName(name)` and returns `<InvalidNameMessage title="Invalid name" />`, where `isValidEnsName` requires `ens_normalize(label) === label` (`utils/token/isNormalized.ts:18,39`). One route in this app treats a non-canonical label as a name it must not act on; the renewal flow treats it as a name to silently rewrite. Both cannot be the design — and note this also means the defect is **unreachable from `/$name`**, so it is not covered, even in part, by any route-param fix.

***

## Recommendation

1. **`apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:197` — send the raw label.** Replace `` `${getLabel(params.name)}.eth` `` with the verbatim `params.name`, and delete the comment above it, which has the direction backwards: ensjs not normalizing is precisely why the raw label must be passed through.
2. **Quote the price on the same label that is renewed** (`useRegistrationPrice.ts:53-54`), so the two can never disagree silently again.
3. **Select and use `normalizedName`.** Add it to the `domains` query in `useV2NamesWithRolesForAddress.ts:52-56` and assert `name === normalizedName` before building any write; fail closed when they differ. The field already exists on the indexer and the Manager already reads it.
4. **Make the mismatch impossible to express.** Have the renewal builder assert that the label in the calldata is byte-identical to the label rendered in the confirmation card, and refuse to build otherwise.
5. **Surface non-canonical names in the names list**, so a label that is not its own normalized form is visibly distinct — including the Unicode-composition cases, which are indistinguishable by eye.
6. Apply the same rule to any other Explorer flow that derives calldata via `getLabel` from an indexer-sourced row.

***

## Proof of Concept

## Proof of Concept

Executed against a local Anvil fork of Sepolia. No live network was written to.

### Environment

```
anvil/v1.6.0   chainId 11155111   fork block 11601093   RPC http://127.0.0.1:18545
guard: http://127.0.0.1:18545 ok: anvil/v1.6.0, chainId 11155111, fork block 11601093
```

The PoC carries an import-time guard that refuses to run unless the RPC host is loopback, `web3_clientVersion` matches `/anvil/i`, `eth_chainId` is `11155111`, and `anvil_nodeInfo.forkConfig.forkBlockNumber` is non-zero. Actors are derived from fixed seed strings with `getCode == 0x` asserted, because anvil's default dev accounts carry EIP-7702 delegation designators inherited from real Sepolia state.

```
ETHRegistry    0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
ETHRegistrar   0xa88553F454b77203B0D036A05c894d555EAAa2Cc
```

### Setup

None for Part B. The two registrations it uses already exist on Sepolia and predate the pinned block. Part A registers a synthetic pair to show the mechanism deterministically. Test USDC is minted via the deployment's public `mint` — a harness convenience only.

### The flow, driven through the app's own functions

The PoC imports and drives the shipped modules; nothing on the money path is hand-encoded:

`getSelectedNames`, `isExtendable2LD` (`features/renew/utils/nameExtension`) · `computeNamePricingDisplay` · `getRenewalPrice` (`features/register/hooks/useRenewalPrice`) · `getRenewerAddress` · **`getLabel`** (`utils/token/getLabel`) · `buildApproveIntent`, `toEoaCustomIntent` (`features/transaction-manager/helpers/intents`) · `createEOASigner` · `renewNameWriteParameters` (`@ensdomains/ensjs/wallet`) · `transactionManager`, `waitForTransaction` (`@ens-apps/transaction-manager`).

`buildRenewIntent` and `buildRenewalApproveIntent` are module-private, so their bodies are transcribed verbatim with line references while every helper they call is the imported shipped one. The only substitution is the RPC endpoint: `@/lib/wagmi` is mocked with `importActual` and every export passed through unchanged except `wagmiConfig.getClient()` (reached via `@/lib/wagmi/helpers`'s `safeGetClient`), which points at the local fork — the same redirection `vite.config.ts` performs for local E2E.

```
$ POC_RPC_URL=http://127.0.0.1:18545 pnpm --filter portal exec vitest run \
    .poc/explorer-renew-normalized-label.test.ts --reporter=verbose

 ✓ the Explorer renews the NORMALIZED label while displaying the raw one: the victim pays and a third party gains a year
 ✓ PART B — no attacker at all: a real Sepolia name pair, the live owner pays and a stranger gains the year
 Test Files  1 passed (1)      Tests  2 passed (2)
```

**Part A — synthetic pair, deterministic:**

```
victim USDC       999991999979 -> 999983999958   (delta -8000021)
"Qzlabelcrit1" (the name the app SHOWED, owned by the victim) expiry +0s
"qzlabelcrit1" (owned by 0x4956…369D)                          expiry +31536000s
renew tx 0xc95bde82f331d87391753a97d55f3711c15b1006429a81cc9e064a77384ce599
```

**Part B — real pre-existing Sepolia registrations, no attacker, no setup:**

```
"Crypto"  status=2 owner=0x24A2bC3c35f5E9C36f9102d14702C4DD43132096
"crypto"  status=2 owner=0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57
the Explorer shows : "Crypto.eth"  total $8.00  new expiry Sep 18, 2027
getLabel("Crypto.eth") -> "crypto"
renew tx 0x8de28829d85a21b8efbfc4f6dc0de77777d2e121159f08a4d41d9fd4a63248ba
payer 0x24A2bC3c…2096 USDC 1000006931495 -> 999998931474  (delta -8000021)
"Crypto" (payer's own name)          expiry +0s
"crypto" (stranger 0x0a7b367E…bB57)  expiry +31536000s
```

### The defect already misfires today, with no attacker at all

Read-only measurement, retry-wrapped, against the local fork of live Sepolia at 11601093 — `LabelRegistered` on the v2 `ETHRegistry`, blocks 11380000..11601093:

```
distinct labels seen                        : 158379
labels where ens_normalize(label) !== label : 9
```

Of those nine, **eight have a normalized twin registered to a different address right now**, both sides `status = 2` (registered):

| live label     | owner             | twin           | twin owner        |
| -------------- | ----------------- | -------------- | ----------------- |
| `Crypto`       | `0x24A2bC3c…2096` | `crypto`       | `0x0a7b367E…bB57` |
| `CRYPTO`       | `0x24A2bC3c…2096` | `crypto`       | `0x0a7b367E…bB57` |
| `Harmonia`     | `0x24A2bC3c…2096` | `harmonia`     | `0x801D2e48…c191` |
| `TESTens1111`  | `0x5d973676…D91e` | `testens1111`  | `0xFc5958B4…acf0` |
| `ébf-fd20d09e` | `0x13D7d726…D951` | `ébf-fd20d09e` | `0xA533308d…9f8B` |
| `ébf-5559b1f7` | `0x13D7d726…D951` | `ébf-5559b1f7` | `0x4e1ED9cF…D1dA` |
| `ébf-59bb0b6d` | `0x13D7d726…D951` | `ébf-59bb0b6d` | `0xeA187F4D…256c` |
| `ébf-f51f02a5` | `0x13D7d726…D951` | `ébf-f51f02a5` | `0x9aDaD7EC…FC3E` |

The ninth (`éhca-1a016f8610f`) has an unregistered twin (`status = 0`) and correctly does not qualify.

**The last four are the worst case, not a footnote.** Their labels differ from their twins only by Unicode composition — a decomposed `e` + U+0301 versus a precomposed U+00E9 — so the two names **render identically on screen**. The owner of `ébf-fd20d09e` cannot distinguish their own name from the one their rent will actually extend by looking at it, by copying it, or by comparing it character by character in the UI. Here the divergence is invisible to the eye *and* absent from every surface in the application.

### The rendered strings

Quoted from source rather than captured, and the point is what is *absent*: the string `crypto` — the label actually paid for — appears on **no surface in the application**. The confirmation card renders `{selectedName.name}` (`ExtendNameConfirmation.tsx:32-33`); the transaction rows render `` `Extend ${name}` `` from the raw name; and the transaction modal renders only `transaction.title`, a status badge and an `Est. Cost`/`Actual Cost` row (`TransactionsOverviewContent.tsx:59-183`) — no target contract, no calldata, no arguments. The wallet receives an un-decoded `renew(string,uint64,address,bytes32)` on an unverified v2 registrar.

***

## Preempting the likely closure arguments

**"The displayed value is the executed value and the user signed."** This is the criterion that closed #91900, so it is answered here on the merits rather than merely disclaimed. It is factually untrue on this path: the app displays `Crypto.eth` and executes `renew("crypto", …)`. The displayed *amount* is honest — and that is not a mitigation, it is the mechanism, because the quote is taken on the same normalized label, so the one screen that could have exposed the divergence is made internally consistent with it. As to consent: see the first screen. The victim identified the name by name, from a list this app populated, and the app is the sole source of the name→calldata mapping.

**"Recovery."** Conceded first, then defeated, with each route run rather than asserted. R1: there is no `refund` or un-renew on the registrar — the probe reverts. R2: `registry.roles(resource("crypto"), victim) == 0` — the victim holds nothing over the year they bought. R3: the only working remedy is to pay again on the raw label, which was **executed** — a second 8,000,021 units, after which the victim's own expiry finally moves +31,536,000s. That path exists nowhere in the Explorer: every renewal the app builds goes through `getLabel`. Net position after full recovery: 16,000,042 units spent for one year on one's own name, with half of it permanently with a third party.

**"This is normal protocol behaviour."** The registrar behaved correctly throughout: it renewed the label it was told to renew. ENS has never held that paying rent on name A extends name B. The defect is entirely in the application, which told it the wrong one. The app's own `/$name` route proves the project knows a non-canonical label is a distinct name that must not be silently rewritten.

**"The attacker sets bait and hopes."** There is no attacker to downgrade. This is not the weaker "any mixed-case row could trigger it" argument — **eight** live pairs exist on Sepolia today, held by unrelated parties, all predating the fork block, and Part B executes the loss against one of them. The beneficiary takes no action, sets no bait, and need not know the victim exists. The trigger is the victim renewing their own name from their own names page, which is the flow the route exists for.

**"The asset is a valueless testnet token."** What is lost is the ERC-20 the application requires for registration and renewal, spent through the code path under audit. Sepolia is where the code under audit runs; the same path on mainnet spends mainnet value. The program's own prioritized-vulnerability list names *"direct users to transfer funds to an address other than the intended one"* as its primary concern.

**"The victim's action was unusual."** Opening your own names list and renewing a name you own is the flow the page exists for. No debug route, no unusual setting, and no interaction with any third party.

**"Reading the row as written."** *Direct theft of user funds* is a **bare row** — no sub-bullets, no display clause, no consent clause, no attacker clause. This program demonstrably writes conditions into rows when it means them: the adjacent Critical rows carry sub-bullets, a purpose clause (*"so that a user sends funds…"*), a scope boundary (*"during a Manager flow"*), and an explicit carve-out (*"exploiting the 7-day session-key design … is not eligible"*). **The absence is meaningful.** Nor does any exclusion cover it: the one user-action exclusion in the terms is *"un-prompted, in-app user actions that are not part of the normal app workflows"*, and renewing your own name from your own names page is the normal workflow. Had the program intended to require an attacker, or to exempt signed transactions, those are conditions it knew how to write and wrote elsewhere.

***

## Not a duplicate

**#90190 — Manager bulk renewal displays the raw label but renews the normalized one.** This is the nearest prior art and it is named first deliberately. Same class and same polarity, **different application, different file, and an independently required fix**: #90190's fix is at `apps/manager/.../NamesTable.tsx:81-93` (`toBulkRenewName`); there is no `toBulkRenewName` in `apps/portal`, and applying #90190's fix in full leaves `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:197` untouched and this loss live. Two things here are also new relative to #90190: the no-attacker claim is **measured on live registrations** (eight extant pairs, four of them visually identical to their twin) rather than argued from a repo fixture, and the loss is executed against one of those real pairs. Per the program page, *"Duplicate submissions of bugs are valid."*

**#90161 — Explorer authorizes a transfer against the raw name but moves the normalized name's token.** Same app, and also a normalization split, so it is the most dangerous near-miss. It is **not** reachable here: #90161's fix is to gate or canonicalize the `/$name` route param, and `/$name` already **refuses** non-canonical names outright (`routes/$name/index.tsx:442,458-459` → `InvalidNameMessage`). The single-name Extend is therefore unreachable from that route, and the only live entry point is the indexer-sourced names table at `routes/addr/$addr/names.tsx`. #90161 is also about *which token moves*; this is about *which registration is paid for*, and no token moves at all.

**#90163 — Manager `/renew/$name` gates normalized / renews raw.** Different app, different file, and the **opposite polarity** (it sends the raw label where the normalized one was gated). Its fix is to send the normalized label; the fix here is to send the raw one. Applying #90163's fix to this line would make the defect worse.

**#89270 — Manager profile records raw / owner normalized.** Read-side only, different app, sink is a rendered address.

**#91837 — Manager primary name normalized into the reverse registrar.** Different app, different file, different contract; nothing is paid.

**EXP-INPUT-005 — validators accept homoglyphs, bidi and control characters.** Different root cause: that entry is leniency on **typed user input**. The name here is never typed — it arrives from the indexer as an on-chain label the victim already owns. That entry also self-scopes to *"homograph-style display confusion in the UI rather than incorrect resolution"*; this is neither confusion nor resolution, it is a payment landing on a different registration. Tightening a validator this flow never calls changes nothing.

**QA-03 — per-year price and displayed total do not always agree (display-level only).** Its carve-out is *"a mismatch between the displayed total and the amount actually charged on-chain would be a new finding"* — that is not this. Here the displayed total and the charged amount **agree**; what diverges is the name the payment buys.

**QA-10 — bulk renewal shows no avatars.** Cosmetic, same flow, unrelated cause.

**R3-01…R3-08, R3-02 in particular** — availability and data-integrity defects in `packages/transaction-manager`. This finding submits successfully and settles; nothing hangs, nothing is dropped from history.

**WEB-1187 / WEB-1188 (no gas warning), QA-05 (renew blocked after registration), WEB-1304 (wrapped V1 NameWrapper expiry), WEB-1255 (grace-period `isV2` flag)** — all renewal-adjacent, none touches label derivation.

***
