# 91640 w a critical bulk renewal signs a different label than the name displayed so the fee renews an unintended name displayed vs signed mismatch&#x20;

**Submitted on Sep 3rd 2026 at 08:50:47 UTC by @SaiyanVegeta for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91640
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Bug Description

**Brief.** In the bulk-renew flow the row and the confirmation dialog display one name, but the `renew()` calldata carries a different label with a different labelhash. The user approves a renewal of the name they see and pays to extend a name they do not see; their own name is not renewed.

**Details (verified at HEAD).**

* `apps/manager/src/features/dashboard/utils.ts:46` — `resolveDomainLabel = domain.name ?? domain.normalizedName ?? domain.id`. The row text (`NameRow.tsx`) is this raw indexer `name`.
* `apps/manager/src/features/dashboard/components/NamesTable.tsx:83-90` — the bulk payload is built as `const name = domain.normalizedName ?? displayName; label = name.replace(/\.eth$/i,'')`. **The code comment on those lines states the display name "may be unnormalized … and would hash to the wrong label"** — the developers noted the exact divergence. `displayName` (raw) is shown; `label` (normalized) is signed.
* `apps/manager/src/features/bulk-renew/components/NamesBreakdown.tsx` shows `row.displayName`; `apps/manager/src/features/bulk-renew/hooks/useBulkRenew.ts` prices `n.label`; `apps/manager/src/features/bulk-renew/hooks/useBulkRenewSubmit.ts` renews `item.label`.
* Contrast: `apps/manager/src/routes/renew/$name.tsx:61` and `routes/register/$name.tsx:57` sign `parseName(raw).label`. So the same on-chain name is renewed under one label via the single-renew route and a **different** label via bulk renew — the app has no consistent canonical-label policy.

**Only transform-class labels diverge** (case is absorbed by `parseName`'s lowercase; non-normalizable names have `normalizedName: null` and fall back to raw in both flows): FE0F stripping, fullwidth/NFKC mapping, and ignorable-code-point removal (soft hyphen, ZWSP). The live indexer confirms this by its own contract — `Domain.name` is documented "Full domain name (e.g., 'vitalik.eth')" (the raw name) and `Domain.normalizedName` "ENSIP-15 normalized name (null if normalization fails)" are two separate fields — so for a transform-class label `name` (displayed) and `normalizedName` (signed) are non-null and differ (see PoC 3).

## Impact

The user approves a transaction whose displayed target (name X) differs from the signed target (label Y, different labelhash). Concretely: the fee extends a third party's registration, and the user's own listed name is silently not renewed and can lapse. This matches the enumerated impact "showing the wrong … transaction detail … on a transaction the user then approves"; QA-03's own text says a displayed-vs-on-chain divergence "would be a new finding."

## Risk Breakdown

* **Lead scenario (no self-inflicted, no validator dependency) — the gift path.** The attacker registers `victim.eth` and, separately, the fullwidth twin `ｖictim.eth` (both mintable on the v2 registrar — proven on-chain, see PoC 2), then ERC-1155-transfers the twin to the victim (the dashboard lists names by owner). The victim selects "all" and bulk-renews: the dialog shows `ｖictim.eth`, the calldata signs `renew('victim')` — a permissionless renewal of the attacker's `victim.eth` at the victim's expense. The victim's own `ｖictim.eth` is not renewed. This path needs no Manager validator flaw, so it is not EXP-INPUT-005, and the app (not the victim) performs the label swap in `toBulkRenewName`.
* **Not EXP-INPUT-005.** That issue is scoped to display confusion "rather than incorrect resolution"; here the calldata label differs from the displayed name (a signed-value divergence), and the fix locus is `toBulkRenewName`, not any validator. The report does not depend on how the twin came to exist.
* **No-attacker corollary (supporting evidence):** with no twin registered, every bulk renewal of a transform-class row reverts `NameNotRenewable` while the single-renew route succeeds — the two flows disagree even before an attacker appears, and the app never calls the registrar's `isAvailable`/`isRenewable` before submitting.
* **Precondition — confirmed, not assumed.** The two facts the attack rests on are both proven: (a) the indexer returns the raw `name` distinct from `normalizedName` (live schema, PoC 3); (b) a transform-class twin is registrable on the real v2 registrar (on-chain, PoC 2). This closes the "not reproducible on the live deployment" objection.
* **Severity Medium (honest ceiling).** Direct loss is bounded by one renewal fee per event and the attacker pre-positions a twin (cost parity \~1:1 unless the victim renews many years or the twin is 3-char premium). The secondary effect is the stronger harm: the victim's own name is never renewed via bulk and can lapse into release — asset loss, not just fee loss. No connected-wallet tampering and the victim must include the row in a bulk selection, so not High.

## Recommendation

Derive the bulk-renew label from the same on-chain source the single-renew route uses (`parseName(displayName).label`), or display `normalizedName` so shown and signed match. Refuse rows where `name !== normalizedName` with an explicit warning. Never substitute a normalized twin for signing.

## References

* `apps/manager/src/features/dashboard/utils.ts:46`, `apps/manager/src/features/dashboard/components/NamesTable.tsx:83-90` (comment admits the labelhash divergence)
* `apps/manager/src/features/bulk-renew/hooks/useBulkRenew.ts`, `useBulkRenewSubmit.ts`, `components/NamesBreakdown.tsx`
* Contrast: `apps/manager/src/routes/renew/$name.tsx:61`, `routes/register/$name.tsx:57`

## Proof of Concept

## Proof of Concept

### PoC 1 — displayed name vs signed label (8/8 pass)

`apps/manager/src/audit/tech-displayed-vs-signed-bulk-renew.test.tsx`. Renders the real `NamesTable` with mocked data hooks, toggles the real selection, captures the `BulkRenewName` handed to the dialog and compares with the single-renew label. Observed:

```
row "🏳️‍🌈.eth"  → bulk label "🏳‍🌈" (FE0F stripped), single-route "🏳️‍🌈", labelhash differs
row "ｖictim.eth" → bulk label "victim",  single-route "ｖictim"
row "a­bc.eth"    → bulk label "abc",     single-route "a­bc"
canonical rows   → consistent
```

```
cd apps/manager && NODE_OPTIONS="--max-old-space-size=8192" npx vitest run src/audit/tech-displayed-vs-signed-bulk-renew.test.tsx
```

### PoC 2 — the v2 registrar accepts the non-normalized twin (on-chain, Sepolia fork)

Same as Report A1 (`onchain__fork-label-validity.mjs`): against the real deployed v2 ETHRegistrar bytecode, uppercase / fullwidth / soft-hyphen / zero-width-space / FE0F labels all return `isAvailable = true` and `register()` succeeds. This proves both the attacker's twin and the victim's transform-class name are mintable on the in-scope deployment.

### PoC 3 — the live indexer returns the raw name distinct from the normalized one (read-only introspection)

The attack requires the indexer to expose the raw `name` alongside `normalizedName`, so the dashboard can display the raw name while the bulk-renew payload takes the normalized one. A read-only GraphQL introspection of the app's own indexer (`https://staging-graphql.ens.dev/`, the endpoint in `packages/indexer/urql/client.ts`) confirms the contract:

```
Domain.name           : String  — "Full domain name (e.g., 'vitalik.eth')"
Domain.labelName      : String  — "Label only (e.g., 'vitalik' for 'vitalik.eth')"
Domain.normalizedName : String  — "ENSIP-15 normalized name (null if normalization fails)"
```

`name` is the raw full name and `normalizedName` is a *separate*, ENSIP-15 field. For a transform-class label they are both non-null and differ, so `resolveDomainLabel` displays `name` (raw) while `toBulkRenewName` signs `normalizedName` — the divergence surfaces in production data, not just in the harness. (Read-only introspection only; no data mutated.)
