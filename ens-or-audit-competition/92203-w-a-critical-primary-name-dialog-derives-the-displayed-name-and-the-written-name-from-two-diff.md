# 92203 w a critical primary name dialog derives the displayed name and the written name from two different expressions so the confirmed operation writes a reverse record for a different ens node&#x20;

**Submitted on Sep 8th 2026 at 07:45:55 UTC by @Nyko for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92203
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

`ChoosePrimaryNameDialog` in `apps/manager` builds the name it displays with `resolveDomainLabel(domain)`, which returns the raw indexer label, and writes the primary name through `setPrimaryName`, which passes that same string through `normalize()` first. For any name whose on-chain label is not already canonical, the displayed subject and the signed subject are two different ENS nodes. The application reports "Primary name set successfully" in both outcomes: either the claim cannot resolve at all and the user is told it was set, or, where another party holds the canonical form, the user's address now reverse-resolves to a name that party owns and whose address record they can repoint at any address they choose.

This is the same structural defect the team assessed at Critical in report #91062: a value that must be single is derived from two different expressions, one of which reaches the screen and the other the transaction. As with #91062, the defect and every line of the fix are in application code; the resolvers, registrars and indexer behave correctly throughout.

## Vulnerability Details

All line numbers were asserted programmatically against the frozen revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, with the clone verified unmodified.

**1. The display path deliberately prefers the raw label.**

File: `apps/manager/src/features/dashboard/utils.ts` lines 42 to 46.

```ts
export const resolveDomainLabel = (domain: {
  id: string
  name?: string | null
  normalizedName?: string | null
}): string => domain.name ?? domain.normalizedName ?? domain.id
```

The field `normalizedName` is present on every row and is not preferred.

**2. The dialog makes that raw label the selection, and the selection is what both writes receive.**

File: `apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx`

```
line 81   const label = resolveDomainLabel(domain)
line 82   const isSelected = selectedName === label      so selectedName IS the raw displayed label
line 229  forward leg   saveRecords({ name: selectedName, ... })
line 278  reverse leg   setPrimaryName({ name: selectedName, ... })
```

**3. Only the reverse leg normalizes, so the two legs address different nodes.**

File: `apps/manager/src/features/profile/service/setPrimaryName.ts` lines 46 to 49.

```ts
// Normalized claim string: a non-canonical name would fail the bidirectional
// check at resolution time and read as "no primary name".
const withEthSuffix = (name: string) =>
  normalize(name.endsWith('.eth') ? name : `${name}.eth`)
```

That comment states the consequence exactly, and it is the consequence the dialog delivers, because nothing upstream normalized the selection. For an indexed label "Alaska.eth" the two legs resolve as follows.

```
leg                              code                                          node addressed
forward, address record written  saveRecords({ name: selectedName })  line 229  namehash("Alaska.eth")
reverse, primary name claimed    setPrimaryName -> normalize(...)     line 49   namehash("alaska.eth")
```

**4. The pre-write gate is computed on the displayed name, not the signed one.**

Lines 296 to 309. The helper `shouldUpdateEthAddress` decides whether the forward leg runs, using `hasMatchingEthAddress(selectedNameRecords, ownerAddress)`. Those records belong to the selected raw name. Nothing in the component inspects the normalized string that is actually claimed, so the guard cannot observe the divergence.

**5. Success is reported unconditionally.**

Lines 345 to 346. The mutation's `onSuccess` calls `toast.success` with the message "Primary name set successfully". There is no read-back of the reverse record, so a claim that cannot resolve is indistinguishable in the interface from one that can.

**The same hazard is documented and handled correctly one component away.**

File `NamesTable.tsx` lines 83 to 86 uses the same helper and splits the two uses on purpose.

```ts
const displayName = resolveDomainLabel(domain)
// Use the canonical normalized name for the on-chain label — the display name
// may be unnormalized (e.g. mixed case) and would hash to the wrong label.
const name = domain.normalizedName ?? displayName
```

The dialog omits that split. The non-canonical label is therefore not a hypothetical state: the repository asserts it here, and the Explorer ships a "Not Normalized" badge for it at `apps/portal/src/routes/$name/token.tsx` line 352.

## Impact Details

Both outcomes follow from the same divergence.

**Direction A. The confirmed operation cannot succeed, and the application reports that it did.** This requires no attacker and no preconditions beyond a name whose indexed label is not canonical. The reverse record claims "alaska.eth". Reverse resolution forward-verifies that claim. The user owns "Alaska.eth", a different node, so the bidirectional check fails and the address reads as having no primary name. That is precisely the outcome the comment at `setPrimaryName.ts` lines 46 to 47 warns about. The user has signed and paid, their identity is not set, and the interface told them it was. The dashboard confirms the failure silently, because `profileReverseName.ts` resolves through `getEnsName`, which forward-verifies and simply returns nothing.

**Direction B. The user's on-chain identity is bound to a name another party owns.** The canonical form is a distinct ENS node. Where it is held by an attacker, the victim's reverse record now names the attacker's property. The attacker controls that name's address record, its text records and its avatar. Repointing the address record makes the victim's address render, to any client performing reverse resolution, as a name resolving to an attacker-controlled address. The victim selected and confirmed "Alaska.eth", a name they own, on every screen of the flow.

**On severity.** I am filing this at High and making the case for Critical rather than asserting it. Direction B is the first example of the Critical impact as written, "resolving or rendering an ENS name to an attacker-controlled address", reached through a transaction whose displayed subject and signed subject are different nodes, which is the same reading the team applied when setting #91062 to Critical. Against that, and stated plainly rather than left for review: report #91062 conferred authority over every name a resolver serves, whereas this binds one victim's identity at a time; Direction B requires the attacker to already hold the canonical variant; and the act that monetises it, repointing the address record, breaks the bidirectional check that displays the binding, so the attacker's window is the interval in which a counterparty records the name. Direction A carries no such condition and fires for any non-canonical name. What is not conditional in either case is the mechanism: one confirmed operation addresses two different ENS nodes, and neither is necessarily the one the user was shown. I would not dispute a considered call either way.

**Why this is not covered by the published known issues.**

* EXP-INPUT-005 concerns name validators accepting deceptive characters, and its stated impact is explicitly "homograph-style display confusion in our UI rather than incorrect resolution". This finding needs no deceptive characters, plain ASCII mixed case is sufficient, and it is incorrect resolution: the signed name is a different node from the displayed one. It is the case that entry disclaims. No validator sits on this path either, because the string arrives from the indexer through `resolveDomainLabel`, so the remediation implied by EXP-INPUT-005 does not reach it.
* QA-01, WEB-1249, WEB-1257, WEB-1305 and WEB-716 are role, subname and transfer-flow issues that error or leave state detached. Here nothing errors: both transactions succeed and are reported as success.
* WEB-1255 is a wrongly-defaulted scalar whose value is presentational. Here the wrongly-chosen string determines the ENS node written on-chain.

## References

* `apps/manager/src/features/dashboard/utils.ts` lines 42 to 46, the display helper that prefers the raw label
* `apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx` lines 81, 82, 229, 278, 296 to 309, 345 to 346
* `apps/manager/src/features/profile/service/setPrimaryName.ts` lines 46 to 49, the normalizing leg
* `apps/manager/src/features/dashboard/components/NamesTable.tsx` lines 83 to 86, the same hazard documented and handled correctly with the same helper
* `apps/manager/src/features/profile/service/profileReverseName.ts`, the forward-verifying read-back that makes Direction A silent
* `apps/portal/src/routes/$name/token.tsx` line 352, the shipped "Not Normalized" badge
* `apps/portal/src/features/profile/hooks/usePrimaryName.ts` lines 43 and 93, the correctly verified counter-example
* Frozen revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

## Proof of Concept

## Step-by-step reproduction

**Direction A, no attacker and no preconditions.**

1. Hold a name whose indexed label is not canonical, for example mixed case. The Manager dashboard lists it, the Explorer renders such names with a "Not Normalized" badge at `apps/portal/src/routes/$name/token.tsx` line 352, and `NamesTable.tsx` lines 84 to 85 state that this situation exists.
2. Open the Manager dashboard and choose "Set primary name". Every screen of the dialog shows the raw label "Alaska.eth", because line 81 renders `resolveDomainLabel(domain)` and line 82 makes that same string the selection.
3. Confirm and sign. The forward leg writes the address record for "Alaska.eth" at line 229. The reverse leg passes the same string through `withEthSuffix` at `setPrimaryName.ts` lines 48 to 49 and claims "alaska.eth".
4. Observed: the toast reads "Primary name set successfully", line 346.
5. Reload the dashboard. No primary name is shown. The reverse record claims a node the user does not own, so the bidirectional check fails and `profileReverseName.ts` returns nothing.

**Direction B, the canonical form is held by another party.** Identical steps, with "alaska.eth" registered to an attacker. After step 4 the victim's address reverse-resolves to the attacker's name. The attacker repoints that name's address record at an address they control, and any client performing reverse resolution on the victim's address now renders a name resolving to the attacker.

## Runnable reproduction

Save as `poc.mjs` and run `node poc.mjs` with Node 22. No install, no network, no chain. It exits 0 on reproduction. The functions marked VERBATIM are copied unmodified from the frozen revision.

```js
// VERBATIM from apps/manager/src/features/dashboard/utils.ts lines 42-46
const resolveDomainLabel = (domain) => domain.name ?? domain.normalizedName ?? domain.id

// VERBATIM from apps/manager/src/features/profile/service/setPrimaryName.ts lines 48-49
// normalize is viem/ens (ENSIP-15). For the ASCII uppercase input demonstrated
// here, ENSIP-15 maps to lowercase, which toLowerCase() reproduces exactly.
const normalize = (s) => s.toLowerCase()
const withEthSuffix = (name) => normalize(name.endsWith('.eth') ? name : `${name}.eth`)

// An indexer row for a name whose on-chain label is not canonical.
// NamesTable.tsx lines 84-85 state this case is real; the Explorer ships a
// "Not Normalized" badge for it.
const domain = { id: '0xabc', name: 'Alaska.eth', normalizedName: 'alaska.eth' }

// ChoosePrimaryNameDialog.tsx lines 81-82: selectedName IS the raw displayed label
const displayedLabel = resolveDomainLabel(domain)
const selectedName = displayedLabel

const forwardLegTarget = selectedName                 // line 229 saveRecords({ name: selectedName })
const reverseLegTarget = withEthSuffix(selectedName)  // line 278 -> setPrimaryName -> withEthSuffix

const ok = (c) => (c ? 'PASS' : 'FAIL')
console.log('displayed in the dialog      :', JSON.stringify(displayedLabel))
console.log('toast shown to the user      : "Primary name set successfully" (line 346)')
console.log('leg 1 FORWARD, addr record on:', JSON.stringify(forwardLegTarget))
console.log('leg 2 REVERSE, name claimed  :', JSON.stringify(reverseLegTarget))
console.log(ok(reverseLegTarget !== displayedLabel),  'the signed name differs from the displayed name')
console.log(ok(forwardLegTarget !== reverseLegTarget), 'direction A: the legs address different ENS nodes')
console.log(ok(reverseLegTarget === 'alaska.eth'),     'direction B: the claim names the canonical form')
process.exit(reverseLegTarget !== displayedLabel && forwardLegTarget !== reverseLegTarget ? 0 : 1)
```

Observed output:

```
displayed in the dialog      : "Alaska.eth"
toast shown to the user      : "Primary name set successfully" (line 346)
leg 1 FORWARD, addr record on: "Alaska.eth"
leg 2 REVERSE, name claimed  : "alaska.eth"
PASS the signed name differs from the displayed name
PASS direction A: the legs address different ENS nodes
PASS direction B: the claim names the canonical form
```

## Verification against the shipped source

The stub above is only for portability. The same result was produced with the real shipped code: the repository's own `resolveDomainLabel`, transpiled directly from `apps/manager/src/features/dashboard/utils.ts` rather than retyped, and the exact `normalize` the repository pins, which is viem 2.52.2 per the catalog entry and the lockfile. The real `resolveDomainLabel` returns "Alaska.eth" and the real `normalize("Alaska.eth")` returns "alaska.eth". A control case confirms that a name which is already canonical does not diverge, so the check produces no false positive. Every line cited in this report was asserted against the frozen commit with the clone verified unmodified.

## Recommended fix

Use `domain.normalizedName` for the write and keep `resolveDomainLabel` for display, which is the split that `NamesTable.tsx` lines 83 to 86 already makes, and derive both legs from one normalized value so they cannot address different nodes. Read the reverse record back before reporting success, or surface the name actually claimed in the confirmation step.
