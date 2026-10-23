# 91962 w a insight hardcoded true in the explorer s forward match column certifies any address as the verified primary name holder of any ens name

**Submitted on Sep 5th 2026 at 20:24:42 UTC by @rev3rse for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91962
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

**Asset:** `apps/portal` (Explorer) · **Commit:** `1c9b47f18fcddd2e864dfe385c4171061c9811ae` **Severity requested:** **Critical** — *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: **resolving or rendering an ENS name to an attacker-controlled address**"* **Fallback if contested:** **High** — *"Injecting/modifying the static content on the target application without JavaScript (**persistent**) … replacing existing text with arbitrary text"*. **Not Medium**: the Medium row is the **reflected** form of that same impact, and this effect is persistent on-chain until the attacker clears it.

## Summary

For one transaction of gas, any address makes the Explorer state it is the forward-verified primary name holder of any registered ENS name, on five of the seven networks `/addr/$addr/reverse-resolution` draws. `routes/addr/$addr/reverse-resolution.tsx:28-38,126` declares this a public file route with no wallet or session gate: any visitor, any address, zero clicks.

ENSIP-19's `default.reverse` is self-assigned — anyone calls `setName("victim.eth")`, unchecked. **That is correct contract behaviour and the premise here, not the defect:** verifying the self-declaration is the consumer's job, and the Explorer has a column for it. `apps/portal` asserts that verification without computing it.

## Root cause

`features/reverse-resolution/components/ReverseResolutionTable/columns.tsx`:

```tsx
const forwardMatch = row.original.forwardMatch              // :93   bound
if (!name && defaultName && canInheritDefault(coinType)) {  // :96   every L2 row
  return <><Badge>True</Badge><Badge>Primary name</Badge>…  // :101,:105  JSX literals
```

`forwardMatch` is bound on `:93` and never referenced in that branch. This is not a wrong value rendered — it is **no value** rendered. `canInheritDefault` (`:12-13`) excludes only the Default and Mainnet rows, so this branch runs for every L2 row.

The honest value was fetched by the same hook and dropped: `useReverseResolution.ts:239-241` calls `getName(…, allowMismatch: true)`, keeps `.name`, discards `.match`. That same 260-line hook derives `forwardMatch` **correctly twice**, at `:103` and `:167-170`.

**Everything upstream of this file reported the truth.** The registrar recorded, accurately, that an address self-declared a name. The UniversalResolver reverted `ReverseAddressMismatch`, correctly, because that name does not resolve back to the address. `@ensdomains/ensjs`'s `getName` carried that answer into `apps/portal` faithfully, as `match: false`. Nothing outside this asset needs to behave differently — the string `True` is manufactured inside `apps/portal`, as a JSX literal, past the point where the correct answer already existed in a local variable one line above.

## Impact

The page exists to answer *"is this binding verified?"* It answers **yes** while holding, in the same request, the value that says no.

* **The application makes the false statement.** The attacker authors no copy and contacts nobody. `setForwardResolution.ts:4-5` documents the app's own definition of a "primary name": one that makes the name primary "by establishing the forward match." The badge asserts both halves of that sentence — `True` and `Primary name` — for a pair where the forward match is absent.
* **Three distinct principals.** The record is keyed on the attacker's address; the impersonated name's owner and the viewer are harmed, and neither has any input to it. The impersonated party takes no action and is never contacted; the viewer performs only the ordinary act of consulting the Explorer's address page.
* **No recovery.** The rendered verdict has exactly three inputs — the attacker's own `default.reverse` record, the attacker's own per-L2 record (absent), and a static config constant. No call anywhere in ENS lets a name's owner clear, dispute or annotate a reverse record held under someone else's address. The condition persists until the attacker chooses to clear it.
* **Not a corner case.** Any name in the same state as the PoC's victim name — registered, with a resolver, no per-chain forward record of its own set by anyone — is impersonable this way, from any attacker address, for one transaction of gas each. That is the state `default.reverse` exists to cover; it is not a degraded or unusual configuration.
* **Not a legibility complaint.** This is a factual claim rendered in a column whose sole stated purpose is to carry that claim, contradicted by the same application two rows above, in the same request.

I demonstrate no misdirected payment and claim none; no funds moved. I am not claiming any impact row that requires an already-connected wallet — this page requires none. The third-party harm is the **table**, which needs no click; the sidebar's corroborating banner is owner-gated (`ReverseResolutionTable.tsx:64`: `canModify = account ? isAddressEqual(account, address) : false`, the sheet's only trigger), so I do not count it as additional third-party exposure.

## Dedup note

Governing clause: *"Reports that describe the same root cause … will be treated as duplicates, even if the report describes a different exploitation path or entry point."* Root cause claimed here: `columns.tsx`'s `forwardMatch` cell hardcodes `True` on its inheritance branch, and `useReverseResolution.ts:241` discards the `match` flag that branch would need. Two issues share a root cause only if fixing one fixes the other:

* **EXP-INPUT-005** (ENS name validators accept homoglyphs/bidi/control characters). Hardening validators does not touch `columns.tsx`, which has no validator on its path — it would still render `✓ True` for a plain-ASCII name. Conversely, this report's two-line fix leaves every homoglyph case untouched. They cannot even share a code path: `getName` returns `null` for any unnormalized input, and `allowUnnormalized` occurs **0 times** anywhere under `apps/` (`git grep`, checkable directly), so `defaultName` is `null` and `columns.tsx:96` cannot fire for the inputs EXP-INPUT-005 describes. The name in this report is byte-exact normalized ASCII.
* **QA-06** (a subname still shown as primary after it stops resolving) requires `name !== null`. Both sites here (`columns.tsx:96`, and `computeDisplayNameState.ts:80`'s `isInheritingDefault = !name && !!defaultName && !isL1`) require the opposite. With `name !== null` the same code path correctly renders `False`. Disjoint precondition, disjoint branch — QA-06 cannot reach this bug and this bug cannot produce QA-06.
* **WEB-1257 / QA-01** describe role combinations that make a later write **error**. This is a call that **succeeds** and renders a false positive — different symptom, different storage location (EAC roles vs. a discarded local variable in this hook).

## Recommended fix

Carry the boolean the hook already has — `defaultResult?.match ?? false`, the pattern `:103` uses — and read it at `columns.tsx:96-112` instead of the literal. Two lines.

## Supporting materials

`poc.patch` — one self-contained file (`apps/portal/src/__poc_s27__/reverse-resolution-false-forward-match.poc.test.tsx`), modifies nothing tracked. Every claim above is either asserted by that suite or directly checkable against the cited lines.

## References

1. `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx` (12-13, 82-112) — the root cause: `canInheritDefault` at 12-13; header "Forward match" at 88; `forwardMatch` bound at 93; hardcoded badges at 96-112 that never reference it.
2. `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts` (103, 167-170, 236-241) — the internal-consistency argument and the fix: correct twice, discarded once.
3. `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionTable.tsx` (64) — the owner gate; why the table, not the sidebar, is the third-party harm.
4. `apps/portal/src/features/reverse-resolution/helpers/setForwardResolution.ts` (4-5) — the app's own definition of "primary name" as one whose forward match is established.
5. `apps/portal/src/routes/addr/$addr/reverse-resolution.tsx` (28-38, 126) — the unauthenticated any-visitor route.
6. `apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts` (80) — the dedup argument against QA-06: the disjoint precondition `!name`.

## Link to Proof of Concept

<https://gist.github.com/almog-shachar/95c7de86b2b0c765607f920806b6d0c7>

## Proof of Concept

One self-contained vitest file (`poc.patch` adds it, modifies no tracked file). It builds one real fixture on a local Sepolia fork — a fresh name with a resolver and an `addr` record, registered through the app's own registrar and `VerifiableFactory` — then feeds it through the real, shipped `apps/portal` code: `getReverseResolution`, `columns`, `<ReverseResolutionTable>`. No calldata is hand-built and no network response is mocked.

Three addresses are checked against that one name — **attack** (an unrelated address, the only one that costs gas: it calls `DefaultReverseRegistrar.setName(<name>)`), **positive** (the name's real owner, who legitimately completed the forward record) and **negative** (an address that wrote nothing). Positive and negative are controls only, showing the harness renders honestly given honest input.

Six tests, each checking one fact:

| # | Test         | What it checks                                                                                                                                                                                                      |
| - | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | `CONTROLS`   | The three addresses are distinct, and the raw `getName` call already returns the right answer for each (`positive.match=true`, `negative=null`, `attack.match=false`) before anything is rendered.                  |
| 2 | `CLAIM 1`    | The app's own `getReverseResolution` hook reports `forwardMatch=false` on every row for the attack address, while still carrying the impersonated name — the root cause: the honest value is fetched, then dropped. |
| 3 | `CLAIM 2`    | The real `<ReverseResolutionTable>` renders that data as `False` on Default/Mainnet and `True · Primary name` on all five L2 rows — the table below.                                                                |
| 4 | `CLAIM 2b`   | The positive and negative addresses rendered through the same component: the L2 columns are wrong for the attacker and empty for the negative control alike — no forward-verification signal in either direction.   |
| 5 | `CLAIM 5`    | Interaction cost: zero clicks already show five false badges, and an unconnected viewer sees zero "More" sidebar buttons — the `ReverseResolutionTable.tsx:64` owner gate cited in Impact below.                    |
| 6 | `DIAGNOSTIC` | A fetch guard asserts every RPC call in the run reached only the local fork or the app's own public L2 endpoints, never a live L1 endpoint — rules out an accidental live-chain read as the source of the result.   |

Save the gist below into `poc.patch` <https://gist.github.com/almog-shachar/95c7de86b2b0c765607f920806b6d0c7>

```bash
git checkout 1c9b47f1 && git apply poc.patch && pnpm install
# Fork at LATEST. Do not pin a block: this public endpoint prunes old state,
# and a pinned block fails once it ages out ("state at block #N is pruned").
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com \
      --port 8549 --silent &
cd apps/portal && VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8549 \
  ./node_modules/.bin/vitest run src/__poc_s27__/*.poc.test.tsx
```

**6 of 6 pass.** Attack cost: one `setName`, **46,894 gas**.

| Network                                     | Forward match             |
| ------------------------------------------- | ------------------------- |
| Default · Mainnet                           | **✗ False**               |
| Optimism · Base · Arbitrum · Linea · Scroll | **✓ True · Primary name** |

The data layer reports `forwardMatch === false` on **all seven** rows; the column renders `True` on five of them.
