# 89583 w a insight critical explorer renders an unverified default reverse name as a confirmed primary name the forward match column prints true on a branch that never reads forwardmatch

**Submitted on Aug 20th 2026 at 01:00:32 UTC by @jokerbuena for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89583
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief / Intro

The Explorer displays a reverse-resolution name inherited from `default.reverse` as a **confirmed primary name** - green banner, checkmark, `Primary name: True` - without ever performing the forward resolution that ENSIP-19 requires. Setting that record is permissionless: anyone can make the Explorer certify their own address as `victim.eth`.

The application already contains the correct warning for this exact failure. It is wired to the opposite branch and therefore never shown.

All line numbers are at `1c9b47f` on branch `audit-comp-ready`, the branch the asset table pins.

## Vulnerability Details

### 1. A column titled "Forward match" prints True without reading forwardMatch

`apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx`

```tsx
{
  accessorKey: 'forwardMatch',
  header: ({ column }) => (<SortButton ...>Forward match</SortButton>),   // :82-89
  cell: ({ row }) => {
    const name = row.original.name
    const forwardMatch = row.original.forwardMatch
    const defaultName = row.original.defaultName

    if (!name && defaultName && canInheritDefault(row.original.coinType)) {   // :96
      return (<div ...>
        <Badge ...><CheckCircle2 /><span>True</span></Badge>
        <Badge ...><SquareUser /><span>Primary name</span></Badge>
      ...)                                                                    // :97-107
    }
    ...
    return (<div ...>
      {forwardMatch ? (<><Badge><CheckCircle2 /><span>True</span></Badge> ...</>)
                    : (<Badge><XCircle /><span>False</span></Badge>)}         // :127-141
```

The branch at `:96` renders `True` and `Primary name` and **never reads `forwardMatch`**. The correct implementation - the ternary that does read it - sits twenty lines below, in the same column, unreachable from that branch.

### 2. The state function makes the two paths equivalent

`apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts`

```ts
const isInheritingDefault = !name && !!defaultName && !isL1
const isPrimaryName = forwardMatch || isInheritingDefault
```

The team's own docblock above it documents the vulnerable outcome as the **expected** one, with `forwardMatch: false` producing `isPrimaryName: true`, and `'vitalik.eth'` as the example value.

### 3. The same repository states why this is forbidden

`apps/manager/src/features/profile/service/profileReverseName.ts:13-28` - the Manager app, doing it correctly:

```
the forward-verification (bidirectional check) and the chain-specific -> default.reverse fallback
both happen on-chain in the reverse resolvers, so the client makes exactly one call and
must not implement its own fallback/default handling (ENSIP-19 Deprecating Mainnet as Default).
...
Returns null when the address has no verified primary name (including when the reverse claim
does not forward-resolve back to the address).
```

So the objection "this is by design" is answered by the repository itself: one app implements the client-side fallback that the other app documents as forbidden, and the one that does it also drops the verification.

### 4. The attacker's step is permissionless

Writing a `default.reverse` record for one's own address requires no permission from the victim and no interaction by the victim. The Explorer then presents that record, for any address the attacker chooses to claim, with the confirmation UI above.

## Impact Details

The impact selected is this program's own Critical line, verbatim:

> *Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: **resolving or rendering an ENS name to an attacker-controlled address***

That is precisely what happens: the Explorer **renders** an ENS name as the confirmed primary name of an address the attacker controls. A user reading the green banner, the checkmark and `Primary name: True` has been given the application's own assurance that the name and the address belong together. The verification that would have contradicted it was fetched and thrown away.

## Notes on duplication

Measured against the sponsor's published known-issues document, which is the baseline this competition sets:

```
forwardMatch          0 occurrences        Primary name        6 occurrences (all read, see below)
default.reverse       0                    reverse resolution  0
isInheritingDefault   0                    ENSIP-19            0
```

The six `Primary name` hits are a different mechanism each time, and I read all six rather than counting them: **QA-06** (a deleted, non-resolving subname still shown as primary), **WEB-1301** (address view showing owner/expiry taken from the wallet's primary name), **WEB-1257 / QA-01** (cannot set primary name after a parent transfer).

**QA-06 is the nearest neighbour and I address it directly**, because the rules say a report describing the same root cause is a duplicate even by a different path. The two entries are **disjoint by construction**: QA-06 requires `name !== null` (a subname was set as primary and later stopped resolving), whereas this report's branch is `isInheritingDefault = !name && !!defaultName && !isL1` - it only exists when `name` is null. On QA-06's input, `isPrimaryName` reduces to `forwardMatch` alone, which is the correct behaviour and is exercised by the negative control in the PoC. A fix for this report is a no-op on QA-06's input, and vice versa.

It is also not the same reader: `usePrimaryName.ts:43` checks the verdict (`if (defaultResult?.match)`), while in `useReverseResolution.ts` the string `.match` appears exactly once, at line 103, on the other branch.

### Public prior art, enumerated rather than sampled

I enumerated the upstream `ensdomains/bigname` repository in full through the issues endpoint - **498 items, 274 issues and 224 pull requests, all states** - rather than relying on keyword search, and read every thematic neighbour. Three came close enough to name here:

* **#204** *(closed 2026-07-21)* - "verified primary names check the normalized claim, not the on-chain string". That is a normalization divergence: bigname forward-resolves the **ENSIP-15 normalized** form while the Universal Resolver forward-resolves the literal string. It verifies differently. **This report is that the portal does not forward-resolve at all** on the `default.reverse` branch.
* **#106** *(closed 2026-07-01)* - inverted `owner`/`registrant` COALESCE precedence between two API surfaces of the indexer. An internal inconsistency between endpoints, not a display that asserts verification.
* **#372** *(open)* - a cleared controller shadowing the registrar fallback in `address_names.rs`.

All three live in **`bigname`, the Rust indexer**, which is not one of the five assets in scope. This report is in `apps/portal`, which is. I raise them because they cut the other way too: they show the team has examined primary-name verification and owner/registrant precedence **at the indexing layer**, and that the display path in the Explorer was not covered by that work.

## Proof of Concept

## Proof of Concept

Two PoC files, ten assertions, run at `1c9b47f` on branch `audit-comp-ready`. One is a pure unit test of the state function; the other **mounts the component and asserts the text a user actually sees**.

```
$ ./REJOUER.sh <repo>
depot   : .../audit-comp-ens
HEAD    : 1c9b47f (attendu 1c9b47f)  OK

POC_C1_isPrimaryName_bypass.test.ts          OK    Tests  6 passed (6)
POC_default_reverse_unverified.test.tsx      OK    Tests  4 passed (4)

PoC joues : 10    en echec : 0
```

### The mutation, which is what makes the PoC mean anything

A test that cannot fail proves nothing, so the vulnerable line was broken deliberately and the suite re-run:

```
mutation: computeDisplayNameState.ts
  -  const isPrimaryName = forwardMatch || isInheritingDefault
  +  const isPrimaryName = forwardMatch && isInheritingDefault

  -> 3 failed | 3 passed (6)        on the unit PoC
  -> 1 failed | 3 passed (4)        on the rendering PoC, the failure being
     screen.getByText('True')  -- the component stops printing True

restored -> 10 passed, 0 failed. Repository left clean: git status --porcelain -> 0 entries.
```

The six assertions that stay green under mutation are the negative controls: they cover the branch where `name` is set, on which `isPrimaryName` correctly reduces to `forwardMatch`. So the suite distinguishes the broken branch from the sound one rather than failing globally.

### Steps to reproduce

1. Write a `default.reverse` record pointing at an address you control, claiming a name you do not own. No permission from the name's owner is required, and the owner never interacts.
2. Open the Explorer's reverse-resolution view for that address.
3. The row shows, under a column headed **Forward match**: a check icon, `True`, and `Primary name`. The green banner reads *This is the primary name on* , and the info row reads `Primary name: True`.
4. No forward resolution was performed for that row. The verdict was fetched with `allowMismatch: true` and the `.match` field was never read.

### What is not claimed

No funds were moved and no victim was involved. The report claims the rendering, which is the impact line selected - not a completed theft. Whether a given user then sends funds to the displayed address is their action, and the program's Critical line is written around exactly that framing (*"so that a user sends funds... to an unintended recipient"*).
