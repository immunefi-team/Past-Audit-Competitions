# 90094 w a critical renewing a name from the portal confirm screen builds the transaction for a different on chain name than the one it displays the payment renews normalize name not the nam&#x20;

**Submitted on Aug 22nd 2026 at 23:29:07 UTC by @GaelicThunder for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90094
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The executed proof, in one sentence: **the victim paid, and a stranger's registration got longer** — while every screen the victim saw named the victim's own name.

The portal carries **two name-resolution layers that disagree** for any name whose raw string differs from its ENSIP-15 normalized form (uppercase, fullwidth characters, invisible characters — all of which pass every gate on the path):

1. **Display + ownership layer — RAW string.** The profile/owner reads go through ensjs v2 `getOwner`/`findOwner`, which sends `packetToBytes(name)` of the literal string. The renew confirm screen renders that same raw string (`ExtendNameConfirmation.tsx:33`, `MultiNameConfirmCard.tsx:35`). This is what the user sees and confirms.
2. **Action layer — normalized.** The renewal derives its on-chain label via `getLabel(name)` = first label of `ens_normalize(name)` (`useRenewalTransactions.ts:191-197`), because ensjs' `renewNameWriteParameters` splits the label without normalizing — a hazard the app's own comment at line 191 acknowledges.

So for such a name the app shows the user their name and charges them for an action on a **different on-chain node**. This is precisely the class the program's impact list calls out — Impact **6285** (Critical):

> "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: … **showing the wrong recipient or amount on a transaction the user then approves**."

Here the wrong detail is the *name the transaction acts on*: the confirm screen shows the user's raw name while the signed calldata targets the normalized twin. The only shape check between the names list and the confirm screen is `isExtendable2LD`'s regex `^[^.]+\.eth$` (`nameExtension.ts:66`); there is **no normalization re-validation anywhere** in `useCanExtend → ExtendNameConfirmation → buildRenewIntent`, or in the transfer plan.

## Executed proof — the decisive E2E

`poc/raw-name-renew.e2e.poc.spec.ts` drives the **shipped portal UI** (names list → Extend → Confirm → wallet signature → mined transaction) against an anvil fork of the competition's Sepolia deployment. Two matched names are minted through the production registration path (`ETHRegistrarController` commit/reveal — the same flow the app uses):

* **attacker** (`user3`) owns the *normalized* form: `pdiv1787437457033.eth` (`0x3C44…93BC`)
* **victim** (`user`, the connected wallet) owns the *raw* form of the same label: `PDIV1787437457033.eth` (`0xf39F…2266`)

Raw log excerpts (`poc/output-e2e-decisive.txt`, run 2026-08-23 CEST):

```
[BUG] before: attacker expiry=1789856713, victim expiry=1789856774,
      victim USDC=19999386299
[BUG] names list shows the name verbatim: true (row: Name Copy PDIV1787437457033.eth …)
[BUG] confirm-screen h2 headings: ["Confirm extension","PDIV1787437457033.eth"]
[BUG] renewal transactions reported success: ["renewal-approve-0xa88553…",
      "renewal-renew-PDIV1787437457033.eth"]
[BUG] after: attacker expiry=1821414313 (delta=31557600s ≈ 365.25d),
      victim expiry=1789856774 (delta=0), victim USDC=19991380798
[BUG] victim's renew calldata (0x7a2ec02a4cee380b17b645e90ddb5b85c787138090cb2ae483eba49299395621):
      functionName=renew, duration=31557600,
      token=0x768F…7a39        [tx mined at block 11545939]
[BUG] SIGNED LABEL = pdiv1787437457033 ≠ DISPLAYED NAME = PDIV1787437457033.eth
```

What each assertion establishes:

| # | Assertion (in-code `expect`)                                                         | Established fact                                                                                              |
| - | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| 1 | confirm-screen h2 headings contain the **raw** name                                  | The paying user confirms `PDIV1787437457033.eth` — their actual name                                          |
| 2 | attacker expiry delta ∈ \[year − 3d, year + 3d]                                      | The victim's payment extended **the attacker's node** by +31,557,600 s (\~365 d)                              |
| 3 | victim expiry after === victim expiry before                                         | The victim's own node was **not renewed at all** (+0 s) — silently                                            |
| 4 | victim USDC strictly decreased                                                       | The victim paid (−8,005,501 units) for the wrong target                                                       |
| 5 | exactly **one successful normalized-label** `renew(...)` tx is found from the victim | Confirms the payment acting on the other account's normalized node                                            |
| 6 | decoded calldata `args[0] === NORM_LABEL`                                            | The signed renewal targets the normalized label even though the confirmation named the victim-owned raw label |

**Control arm (same test run, 2/2):** the identical UI flow with an already-normalized name (`pctrl1787437492969.eth`, owned by the same victim) renews the victim's **own** node (+31,557,600 s) and the signed label equals the displayed one. The two arms vary only the defect — raw vs normalized label — never the user's behaviour. That is what makes the bug arm a defect rather than a design choice.

## Root cause

One user-visible operation runs under two different canonicalization regimes (all sites re-read at branch HEAD `1c9b47f` on 2026-08-22; the official branch was verified byte-identical on 2026-08-23):

| Role                                       | Site                                                                                                                                                                                                                                              |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Shape gate (regex only — no normalization) | `apps/portal/src/features/renew/utils/nameExtension.ts:66`                                                                                                                                                                                        |
| Gate consumer                              | `apps/portal/src/features/renew/hooks/useCanExtend.ts:67`                                                                                                                                                                                         |
| Confirm display (raw string)               | `apps/portal/src/features/renew/components/ExtendNameConfirmation.tsx:33`; `apps/portal/src/features/renew/components/multi-name-extension/MultiNameConfirmCard.tsx:35`                                                                           |
| Action derivation (normalized)             | `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:191` (hazard comment) and `:197` (`name: \`${getLabel(params.name)}.eth\`\`)                                                                                                      |
| Same divergence in transfer                | `apps/portal/src/features/transfer/hooks/useTransferName.ts:112-113,132` (`getEnsTokenId({ label: getLabel(name) })`); `:177` — the transaction-step labels render the raw `name`, so display shows raw while the action targets `getLabel(name)` |
| Ownership layer resolves the literal bytes | `@ensdomains/ensjs` v2 `getOwner` (`packetToBytes`)                                                                                                                                                                                               |
| Hazard acknowledged in-code                | `useRenewalTransactions.ts:191` comment; ensjs `renewNameWriteParameters` "splits the label without normalizing"                                                                                                                                  |
| Manager variant (supporting pattern only)  | `apps/manager/src/features/register-v2/utils/name-parser.ts:46` — parse lowercases instead of ENSIP-15-normalizing                                                                                                                                |

The manager renew path is display/action-consistent but still diverges from the owned raw name; it is offered as a supporting pattern, not an independent claim.

## Why this violates documented requirements

Normalization before hashing is not a style preference — it is the documented contract for every integration, and this app violates it *within a single operation* by applying two regimes to the same input:

* **docs.ens.domains/resolution/names** — three normative MUSTs, which map one-to-one onto the two layers:

  > "**You MUST normalize a name before you attempt to create a namehash!**" "**You MUST normalize a name before you attempt to create a labelhash!**" "**You MUST normalize a name before you DNS-encode it!** … incorrect namehashes/labelhashes may be derived."

  The ownership/display layer DNS-encodes the RAW string (MUST #3 violated) while the action layer labelhashes the normalized form (MUST #2 satisfied). The docs classify exactly this mixed handling as deriving **incorrect** hashes/nodes — i.e., incorrect resolution, not cosmetic confusion.
* **ENSIP-1 § Name Syntax** — "two names with different case but identical spelling will produce the same namehash". Raw and normalized are *the same name by protocol definition*; an app may not treat them as two different targets.
* **Official ENS blog ("How ENS Normalization Works")** — "The capitalized versions are acceptable inputs. **They are not separate normalized ENS names that different people can register and use independently.**"; "The safe order is: User input → normalization → namehash → ENS contracts."
* **The sponsor's own STYLEGUIDE.md** (L2575): "Always normalize ENS names and import types from ENSjs for proper error handling"; (L2621): "**Always normalize** - Use `normalize()` from `viem/ens` for user input". The codebase mandates at input boundaries what the renew/transfer path does only for the action half.

## Impact

**Renewal (executed above):** the victim pays for an extension that lands on a node they do not own, and their own registration silently does not extend. Two consequences, both live in the executed run:

* *Any holder of the normalized twin captures the payment value.* In the E2E that holder is a second account (`user3`) that registered the normalized form first; no malice is required — an innocent registrant of the normalized name receives the same windfall. Where the holder pre-positions the twin, the effect is a repeatable capture: the victim's every renewal funds a stranger's registration.
* *Or, when nobody owns the normalized form,* the transaction reverts after confirmation — gas burned and intent defeated either way (unit-level corroboration: Suite A's fullwidth/ZWSP variants below).

Either way the user paid for an action on a name that is not the one they confirmed, and their real name lapses silently toward grace period — the failure mode is invisible until it is expensive.

Supporting form (transfer): `useTransferName.ts:112-113,132` derives the token id via `getEnsTokenId({ label: getLabel(name) })` — the wrong-tokenId variant of the same defect. It mostly reverts (self-harm shape), so it is presented as corroboration of the divergence, not as an independent impact claim.

## Precondition — the production path mints such names today

The precondition is not hypothetical, and it needs neither migration nor legacy population. **Executed on-chain** (`poc/rawlabel-registration.probe.ts`, transcript below; script registered a name through the production `ETHRegistrarController` commit/reveal flow with a mixed-case label):

```
[probe] raw label      = FOOprobe-1787437536
[probe] normalized     = fooprobe-1787437536
[probe] nodes differ   = true
[probe] getOwner("FOOprobe-1787437536.eth") -> 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[probe] getOwner("fooprobe-1787437536.eth") -> 0x0000000000000000000000000000000000000000
[probe] PASS: production registration minted the raw node only
```

The v2 registrar accepts the raw label, the ownership layer resolves the raw node to its owner, and the normalized node is a **separate, unowned** node — free for any third party to register, which is exactly what the decisive E2E's attacker account does minutes before capturing the victim's renewal. The E2E itself re-proves the same point end to end: both arms' names were minted through the production commit/reveal path, surfaced verbatim in the portal names list, and were priced and renewed there.

Governance precedent corroborates the population: EP4.3 refunded ".eth names invalidated by ENSIP-15" — DAO policy treats non-normalized variants as invalid forms of a name, not independent names, consistent with this report treating raw-vs- normalized as one name handled under two regimes.

> **Footnote (legacy/migration population).** Independently of everything above, v1→v2 migration tooling passes non-normalized subgraph `labelName`s through unchanged into `migrate()` calldata (Suite B, 4/4, `poc/output-rawlabel-migration.txt`), so legacy pre-2019 v1 registrations with non-normalized labels are a second route by which such names exist on v2. This route is supplementary; the production-path evidence above is the load-bearing one.

## Not a duplicate — boundary against EXP-INPUT-005

Known issue EXP-INPUT-005 states, verbatim:

> "Our name validators accept inputs that ENS normalization later rejects or transforms, including homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot. The practical impact is homograph-style display confusion in our UI **rather than incorrect resolution**."

We do **not** argue a different mechanism — the permissive validator EXP-INPUT-005 describes is the same root cause visible here (the regex-only gate at `nameExtension.ts:66` admitting non-normalized names). What is new is the consequence, and it is covered by the program's own written allowance for *"new consequences of a listed root cause that materially change its severity"*:

* **Disclosed consequence:** homograph-style display confusion in the UI — explicitly *"rather than incorrect resolution"*.
* **New consequence demonstrated here:** because the action layer normalizes while the ownership/display layers do not, the *same* permissive admission reaches the money path: a confirmed-and-paid renewal executes against a **different on-chain node** (`SIGNED LABEL ≠ DISPLAYED NAME`, executed above). Resolution is incorrect in exactly the sense the known issue disclaims.

That moves the severity from Medium display confusion to misdirected payment plus silent non-renewal — a material change, which is the program's stated criterion for treating such reports as new findings rather than duplicates.

Other known issues are unrelated: **QA-03** is price-displayed-vs-charged; **R3-\*** are tx-machine issues.

## Link to Proof of Concept

<https://gist.github.com/GaelicThunder/ff030804540fdf820bd28be9ef73113f>

## Proof of Concept

## Proof of concept

Three tiers, all executed 2026-08-23. Run instructions and the full environment disclosure: `poc/RUN.md`.

**Tier 1 — decisive E2E (2/2, `poc/output-e2e-decisive.txt`)** — described in its own section above; drives the shipped portal UI end to end, asserts the on-chain deltas and decodes the signed calldata.

**Tier 2 — unit-level divergence (6/6, `poc/output-dot-divergence.txt`)** — `poc/poc-dot-divergence.test.ts` (copied into `apps/portal/src/__poc-dot-divergence.test.ts` on detached public HEAD), driving only real exported functions:

| Test                     | Assertion                                                                                                                           |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| CONTROL — normal name    | `getLabel('vitalik.eth')` → `vitalik`; renewal label `vitalik` (self-consistent case works)                                         |
| Gate admits raw names    | `isExtendable2LD` → `true` for `FOO.eth`, `ｆｏｏ.eth`, `foo\u200bbar.eth`                                                             |
| EXPLOIT (case)           | confirm shows `FOO.eth`; renewal label = `foo`; ownership bytes `\x03FOO\x03eth\x00`; `namehash('FOO.eth') ≠ namehash('foo.eth')`   |
| EXPLOIT (fullwidth)      | `ｆｏｏ.eth` renews `foo`; different node                                                                                              |
| EXPLOIT (invisible char) | ZWSP stripped — `foo\u200bbar.eth` renews `foobar`; different node                                                                  |
| Hazard source            | ensjs `renewNameWriteParameters('FOO.eth')` sends label `FOO` verbatim — the app's pre-normalization is what creates the divergence |

Tier 2 covers the Unicode variants the E2E does not exercise, and pins the mechanism (`getLabel` vs raw bytes) that Tier 1 observes from the outside.

*(Tier 3 — precondition population via v1→v2 migration — demoted to the footnote in the Precondition section; 4/4 in `poc/output-rawlabel-migration.txt`.)*

No mock replaces a security boundary or shipped function. Tier 1 operates the Portal as a user does with the real local Panoptes and deployed Sepolia contracts; the complete two-file test-harness parameterisation is disclosed and shipped in `poc/f1-e2e-harness.patch`.

## Honest bounds

* Executed today (2026-08-23): the decisive E2E (2/2, including the control arm), both unit suites (6/6 + 4/4), and the production-path registration probe. Raw outputs are committed alongside this report in `poc/`.
* The E2E runs on the competition's own local rig (anvil fork of the Sepolia deployment, headless wallet authorizing signatures exactly as a user's wallet would). No mainnet or public-testnet funds were moved. Every environment/harness change the E2E requires is disclosed item-by-item with justification in `poc/RUN.md`; shipped application code is untouched relative to `1c9b47f`.
* The report does not require the normalized-twin holder to be malicious: capture of the payment value happens identically with an innocent holder; malice only makes it repeatable.
* Severity is claimed as **High**, not Critical: no address is assigned and no victim-controlled asset is transferred anywhere; the demonstrated loss is a misdirected payment plus the silent non-renewal of the victim's real name.

## Recommendation

Validate `ens_normalize(name) === name` per label once at every money-flow entry (renew selection / `useCanExtend`, transfer plan), rejecting or canonicalizing non-normalized names before they reach a confirm screen — or derive the action label from the same literal string the ownership layer resolved, so display and transaction can never disagree. The sponsor's own STYLEGUIDE rule ("always normalize … for user input") already prescribes the fix.
