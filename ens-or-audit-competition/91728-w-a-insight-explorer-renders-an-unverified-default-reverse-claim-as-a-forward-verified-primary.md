# 91728 w a insight explorer renders an unverified default reverse claim as a forward verified primary name on every l2 row letting any address be displayed as any ens name

**Submitted on Sep 3rd 2026 at 21:25:04 UTC by @risinghorse for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91728
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

ENSIP-19 requires a reverse-resolution claim to be **forward-verified** — the claimed name must resolve back to the address — before it may be presented as that address's primary name. The Explorer fetches the address's `default.reverse` name with verification explicitly disabled (`allowMismatch: true`) and keeps only the name string, discarding the `match` flag the very same call returns. Every L2 row then renders that unverified string with a green "Forward match: **True**" badge and a "**Primary name**" badge, and the row's detail sheet shows a green banner reading "This is the primary name on Base" beside a bidirectional address ↔ name pairing. `DefaultReverseRegistrar.setName(string)` is permissionless and accepts any UTF-8 string, so **one transaction from any address**, owning nothing, makes the ENS Explorer affirmatively certify that the attacker's address *is* a well-known name. A counterparty checking "is this address really `victim.eth`?" on the official explorer is told yes, and pays the attacker.

***

**Severity:** High **Asset:** `apps/portal` (ENS Explorer app) — in scope per the program's Assets in Scope **Target:** ENS Manager & Explorer — Audit Competition **Audited revision:** `63772fd872af472ced58b009499355f3430c2a86`

**Selected impact:** *Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient* — the *"display an incorrect … name"* limb, and the sub-bullet *"resolving or rendering an ENS name to an attacker-controlled address."*

> **Testing & rules compliance.** No testing was performed against mainnet or any public testnet. The defective logic is in pure, exported client-side functions and a table cell; the proof of concept runs entirely offline, and the repository's own existing unit test already asserts the defective behaviour. No public-testnet or mainnet contract was registered, called, or written to.

***

## Vulnerability Details

### 1. The default-name fetch disables verification and then throws away the result

`apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-243`

```ts
const defaultResult = await getName(l1Client, {
  address,
  coinType: DEFAULT_EVM_COIN_TYPE,
  allowMismatch: true,          // <-- forward verification disabled
})
const defaultName = defaultResult?.name ?? null   // <-- `.match` discarded

return ok(resolvedResults.map((r) => ({ ...r, defaultName })))
```

`getName(..., { allowMismatch: true })` returns a result carrying a `match` boolean. Line 241 takes `.name` and drops `.match`, so the verification status is destroyed at the point it is read — even though the information is right there.

**The same file uses that flag correctly elsewhere.** `getL1ReverseRecord` makes an identical call and propagates the flag:

`apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:74-78, 96-104`

```ts
let nameResult = await getName(client, {
  address,
  coinType,
  allowMismatch: true,
})
// ...
return {
  ...network,
  name: nameResult?.name ?? null,
  reverseResolverAddress: nameResult?.reverseResolverAddress ?? null,
  resolverAddress: nameResult?.resolverAddress ?? null,
  normalized: nameResult?.normalized ?? true,
  forwardMatch: nameResult?.match ?? false,      // <-- correct: carries the flag
  defaultName: null,
}
```

And `getL2ReverseRecord` performs a genuine ENSIP-19 forward check for a name actually set on an L2 registrar:

`apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:156-173`

```ts
let forwardMatch = true
try {
  // Per ENSIP-19, an L2 reverse record's forward verification reads the
  // chain-specific address record on the name's resolver. ...
  const addrRecord = await getAddressRecord(l1Client, {
    name,
    coin: network.coinType,
  })
  forwardMatch =
    !!addrRecord?.value &&
    isAddress(addrRecord.value, { strict: false }) &&
    isAddressEqual(addrRecord.value, address)
} catch {
  forwardMatch = false
}
```

So two of the three read paths verify correctly. Only the inherited-default path does not — and that is the path that populates **every L2 row**.

### 2. L2 rows carry `forwardMatch: false`, and the table cell ignores it

An address with no reverse record of its own on a given L2 yields `createEmptyResult`:

`apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:49-59`

```ts
function createEmptyResult(network: Network): ReverseResolutionResult {
  return {
    ...network,
    name: null,
    reverseResolverAddress: null,
    resolverAddress: null,
    normalized: true,
    forwardMatch: false,        // <-- correctly false
    defaultName: null,
  }
}
```

`defaultName` is then stapled onto every row (`:243`). The `forwardMatch` column cell short-circuits before it ever reads that `false`:

`apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:91-111`

```tsx
cell: ({ row }) => {
  const name = row.original.name
  const forwardMatch = row.original.forwardMatch
  const defaultName = row.original.defaultName

  if (!name && defaultName && canInheritDefault(row.original.coinType)) {
    return (
      <div className="flex flex-row items-center gap-2">
        <Badge variant="outline" className="text-xs">
          <CheckCircle2 className="w-4 h-4" />
          <span>True</span>
        </Badge>
        <Badge variant="outline" className="text-xs">
          <SquareUser className="w-4 h-4" />
          <span>Primary name</span>
        </Badge>
        <Badge variant="outline" className="text-xs">
          Default
        </Badge>
      </div>
    )
  }
  // ...
```

`forwardMatch` is destructured on line 93 and then never used on this branch. The column is headed "Forward match" and reports **True** for a check that was never performed.

### 3. The sidebar reaches the same conclusion through a different path

`apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:76-83`

```ts
// Display name: use name if set, otherwise use defaultName on L2 only
const displayName = name || (defaultName && !isL1 ? defaultName : undefined)

// Inheriting default: L2 with no name but has defaultName
const isInheritingDefault = !name && !!defaultName && !isL1

// Primary name: either matches forward resolution OR inherits default
const isPrimaryName = forwardMatch || isInheritingDefault      // <-- forwardMatch OR-ed away
```

`isL1ReverseRegistrarChainId` is true only for chain ids `1` and `60` (`apps/portal/src/lib/reverseRegistrarChainId.ts:23-28`), so every L2 row takes the inheriting branch.

The sidebar then renders a success banner and an affirmative "Primary name" row:

`apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx:645-652`

```tsx
{isPrimaryName && displayName && (
  <div className="flex items-center gap-3 bg-success-fill text-success-text p-4 rounded-md">
    <CheckCircle2 className="w-6 h-6 shrink-0" />
    <span className="font-medium">
      This is the primary name on {label}
    </span>
  </div>
)}
```

The app already has the correct warning for exactly this situation — it is simply unreachable here, because `isPrimaryName` is true (`ReverseResolutionSidebar.tsx:654-660`):

```tsx
{!isPrimaryName && displayName && (
  <div className="flex items-center gap-3 bg-danger-fill text-danger-text p-4 rounded-md">
    <XCircle className="w-6 h-6 shrink-0" />
    <span className="text-sm">
      The set address does not resolve back to this name on {label}
    </span>
  </div>
)}
```

and `:687-699` renders **Primary name: True** with the address and name shown side by side.

### 4. The inconsistency is visible inside the feature itself

`canInheritDefault` excludes the Default row and the Mainnet row (`columns.tsx:12-13`):

```ts
const canInheritDefault = (coinType: number) =>
  coinType !== DEFAULT_EVM_COIN_TYPE && coinType !== MAINNET_COIN_TYPE
```

So the **same unverified `default.reverse` string** renders as **False** on the Default and Mainnet rows (which correctly use `nameResult.match` from `getL1ReverseRecord:102`) and as **True + Primary name** on the five L2 rows, in one table, at the same time.

### 5. The behaviour is codified in the repository's own test

`apps/portal/src/utils/reverseResolution/computeDisplayNameState.test.ts:91-105`

```ts
it('should inherit defaultName when no name is set', () => {
  const result = computeDisplayNameState({
    name: null,
    defaultName: 'vitalik.eth',
    forwardMatch: false,          // <-- explicitly unverified
    reverseRegistrarChainId: 10,
  })

  expect(result).toEqual({
    displayName: 'vitalik.eth',
    isInheritingDefault: true,
    isPrimaryName: true,          // <-- asserted as a primary name anyway
    canSetAsPrimary: false,
  })
})
```

The same pairing appears in the function's JSDoc (`computeDisplayNameState.ts:58-66`). Inheriting `default.reverse` is correct ENSIP-19 fallback behaviour; what is not correct is asserting a forward match that was never computed.

### 6. Writing an arbitrary `default.reverse` name is permissionless

The project states this itself (`apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/hooks/useReverseResolutionMutations.ts:79-83`):

```ts
// Reverse resolution always goes through the ENSv1 `ReverseRegistrar` on
// L1 and the ENSv1 `L2ReverseRegistrar` on L2 — both accept `setName(string)`
// for any UTF-8 string (V1 name, V2 name, subname, DNS-imported, even
// a non-existent name), so we don't gate the input on existence or
// protocol version at all.
```

and documents the registrar (`apps/portal/src/features/reverse-resolution/config.ts:23-35`):

```ts
/**
 * Standalone ENSv1 `DefaultReverseRegistrar` on Sepolia (ENSIP-19
 * `default.reverse`, coin type `0x80000000`). `setName(string)` sets the
 * caller's cross-chain primary name — NOT the ENSv2 permissioned-resolver path.
 * ...
 */
export const DEFAULT_REVERSE_REGISTRAR_ADDRESS =
  '0x4f382928805ba0e23b30cfb75fc9e848e82dfd47' as Address
```

`setName(string)` writes the **caller's own** reverse record. No ownership of the claimed name is required or checked — which is precisely why ENSIP-19 mandates the forward check that this path skips.

### Recommended fix

The fix is small and local: **stop discarding the flag that is already returned.**

1. Carry the match through at `useReverseResolution.ts:241`, e.g. `const defaultName = defaultResult?.match ? (defaultResult.name ?? null) : null`, or add a `defaultForwardMatch` field to `ReverseResolutionResult` and populate it from `defaultResult?.match ?? false`.
2. In `columns.tsx:96`, gate the True/"Primary name" badges on that flag rather than on the mere presence of `defaultName`; render the inherited name with the "Default" badge and a **False** forward match when it is unverified.
3. In `computeDisplayNameState.ts:83`, require verification for the inherited case: `const isPrimaryName = forwardMatch || (isInheritingDefault && defaultForwardMatch)`.
4. Update `computeDisplayNameState.test.ts:91-105`, which currently locks in the defective assertion.

Note that the L2 inheritance itself is legitimate ENSIP-19 behaviour and should be kept — only the unverified **claim of a forward match** needs removing.

***

## Impact Details

### What an exploit costs the victim

The Explorer's reverse-resolution view is the surface a user consults to answer "does this address really belong to this name?" This bug makes the answer an unconditional yes, for any address, against any name.

* **Loss is the full amount transferred.** A counterparty who verifies an address on the official ENS Explorer, sees `victim.eth` with a green "Forward match: True" and "This is the primary name on Base", and then pays that address, loses the full payment irreversibly. There is no cap.
* **Cost to the attacker is one transaction.** No name ownership, no registration, no prior relationship with the victim, and no compromise of anything. `setName("victim.eth")` from the attacker's own EOA is sufficient.
* **The attacker can claim any identity.** Because `setName` accepts any UTF-8 string, including names the attacker does not own and names that do not exist, the impersonation target set is unbounded.
* **Five rows reinforce it.** Optimism, Arbitrum, Base, Linea and Scroll all render the same forged assertion simultaneously, which reads as corroboration rather than as a single suspicious row.
* **The app's own contradicting evidence is suppressed.** The red "The set address does not resolve back to this name" banner — the exact warning this situation calls for — is unreachable because `isPrimaryName` is true.
* **It defeats the diligent user specifically.** Someone who does not check is phished by ordinary means. This bug targets the user who *does* check, on the tool built for checking.

### Mapping to the program's in-scope impacts

Claimed:

* ✅ **Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds … to an unintended recipient** — the app displays an incorrect **name** for an address, with an affirmative verification badge, and that is the basis on which a user decides where to send funds.
* ✅ **resolving or rendering an ENS name to an attacker-controlled address** — the app renders the binding `0xATTACKER ↔ victim.eth` as verified.

Explicitly **not** claimed:

* ❌ *Direct theft of user funds* as a standalone impact — the application does not move funds; the user does, on the strength of the false attribution.
* ❌ *Injection of malicious HTML or XSS through metadata* — nothing executes; the name renders as escaped text.
* ❌ *Hijacking or bricking an ENS name* — the real `victim.eth`, its ownership, resolver and records are untouched. The attacker never interacts with it.
* ❌ *Taking state-modifying authenticated actions on behalf of other users* — the attacker writes only their own reverse record.

### Why High and not Critical

The impact bullet this matches is a Critical one, and I want to be transparent about why I am nonetheless filing High: the deception is delivered on the Explorer's reverse-resolution inspection view rather than on a primary send-funds surface, so it requires the victim to be looking up the attacker's address there. Against that, the exploit has **no preconditions at all** beyond one permissionless transaction, and the view in question exists specifically to answer the identity question it answers wrongly. I have set High and am flagging the Critical argument rather than asserting it; the project is better placed to weigh how the view is used in practice.

### Prerequisites

One transaction from the attacker's own address. No name ownership, no victim wallet connection, no signature from the victim, and no interaction beyond the victim viewing the page.

### Duplicate analysis

No Known Issues entry covers reverse-resolution forward verification, `useReverseResolution`, `computeDisplayNameState`, or the reverse-resolution table.

* Not **EXP-INPUT-005** (validators accepting homoglyphs/bidi): no deceptive characters are involved — the attacker sets the exact, correctly-normalized target name.
* Not **QA-06 / WEB-1257** (a deleted or non-resolving subname still shown as primary, and primary-name setting after a parent transfer): those concern a name the user legitimately set that later stops resolving, and errors that follow a transfer. This report concerns a name the attacker **never had any relationship with**, asserted as verified against an address that was never checked.
* Not **R2-04** (OG-image worker SSRF) or any transaction-manager entry.

***

## References

In-scope code (audited revision `63772fd872af472ced58b009499355f3430c2a86`):

* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-243` — `allowMismatch: true`, `.match` discarded
* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:74-78, 96-104` — `getL1ReverseRecord`, the correct handling of the same flag
* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:156-173` — `getL2ReverseRecord`, a genuine ENSIP-19 forward check
* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:49-59` — `createEmptyResult`, `forwardMatch: false`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:12-13` — `canInheritDefault`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:91-111` — the forward-match cell short-circuit
* `apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:76-83` — `isPrimaryName = forwardMatch || isInheritingDefault`
* `apps/portal/src/utils/reverseResolution/computeDisplayNameState.test.ts:91-105` — the existing test asserting the defect
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx:324-330, 645-652, 654-660, 687-699` — the success banner, the suppressed warning, and the Primary name row
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/hooks/useReverseResolutionMutations.ts:79-83` — the project's own note that `setName` accepts any string
* `apps/portal/src/features/reverse-resolution/config.ts:23-35` — `DefaultReverseRegistrar` address and semantics
* `apps/portal/src/lib/reverseRegistrarChainId.ts:23-28` — `isL1ReverseRegistrarChainId`
* `apps/portal/src/lib/coinType.ts` — `DEFAULT_EVM_COIN_TYPE = 0x80000000`, `MAINNET_COIN_TYPE = 60`

Specifications:

* ENSIP-19, Reverse Resolution (the bidirectional/forward-verification requirement and the `default.reverse` fallback) — <https://docs.ens.domains/ensip/19>
* ENSIP-11, EVM Coin Types (`0x80000000 | chainId`) — <https://docs.ens.domains/ensip/11>
* ENS documentation — <https://docs.ens.domains/>

## Proof of Concept

## Proof of Concept

The defective logic is a pure function plus a table cell, so it is provable offline with no chain and no network.

### Part 1 — The repository's existing test already proves it

`apps/portal/src/utils/reverseResolution/computeDisplayNameState.test.ts:91-105` asserts, today, on a clean checkout:

```ts
computeDisplayNameState({
  name: null,
  defaultName: 'vitalik.eth',
  forwardMatch: false,      // unverified
  reverseRegistrarChainId: 10,
})
// => { displayName: 'vitalik.eth', isInheritingDefault: true,
//      isPrimaryName: true, canSetAsPrimary: false }
```

An unverified claim becomes `isPrimaryName: true`. Running `pnpm test:portal` on an unmodified checkout demonstrates the core of the vulnerability without adding a line of code.

### Part 2 — Reproduce both surfaces offline

I re-implemented `computeDisplayNameState` (`computeDisplayNameState.ts:68-95`), `canInheritDefault` and the forward-match cell branch (`columns.tsx:91-111`) verbatim, and ran them on the state an attacker produces — `name: null` (no L2 record), `defaultName: 'victim.eth'` (the attacker's `setName`), `forwardMatch: false` (from `createEmptyResult`). Observed output:

```
-- Base Sepolia row (coinType 0x80000000|84532) --
  table cell        : True + Primary name + Default
  sidebar state     : {
    displayName: 'victim.eth',
    isInheritingDefault: true,
    isPrimaryName: true,          <-- green banner, "Primary name: True"
    canSetAsPrimary: false
  }
-- Default row (coinType 0x80000000), same unverified data --
  table cell        : False
-- Mainnet row (coinType 60), same unverified data --
  table cell        : False
```

The last two lines are the finding in miniature: **identical unverified data** renders as **False** on the Default and Mainnet rows and as **True + Primary name** on the L2 rows, in the same table, at the same moment.

*(`node_modules` is not installed in this checkout, so `vitest` could not be executed directly; the functions were re-implemented verbatim from the cited line ranges.)*

### Part 3 — Optional local-chain confirmation

If an end-to-end check is wanted, it can be run against the repository's **local** Anvil fork only (`pnpm e2e:infra:up`, chain on `127.0.0.1:8545`), which forks the state containing the `DefaultReverseRegistrar`:

1. From an attacker EOA that owns no ENS name, send one transaction: `DefaultReverseRegistrar.setName("victim.eth")` at `0x4f382928805ba0e23b30cfb75fc9e848e82dfd47` (`config.ts:34-35`). No ownership check is performed; the attacker's own `default.reverse` record now reads `victim.eth`, and `victim.eth`'s resolver is never touched.
2. Open the Explorer at `/addr/<attacker address>/reverse-resolution`.
3. `getReverseResolution` returns `name: null` for all five L2 rows and `defaultName: "victim.eth"` for every row, because `allowMismatch: true` suppressed the bidirectional check.

**No public testnet or mainnet is involved.**

### Observed result

| Surface                                          | What it shows                                                                            |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| Optimism / Arbitrum / Base / Linea / Scroll rows | `victim.eth` with a green **"Forward match: True"** badge and a **"Primary name"** badge |
| Row detail sheet                                 | Green banner: **"This is the primary name on Base"**                                     |
| Sheet info rows                                  | **Primary name: True**, with the attacker's address and `victim.eth` shown side by side  |
| Default row / Mainnet row                        | **False** — from the same unverified data                                                |

The application affirmatively certifies a bidirectional binding it never checked, and suppresses the warning it has for precisely this case.
