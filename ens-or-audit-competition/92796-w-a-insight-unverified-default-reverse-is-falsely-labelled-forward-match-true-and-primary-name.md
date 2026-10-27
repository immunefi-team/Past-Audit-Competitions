# 92796 w a insight unverified default reverse is falsely labelled forward match true and primary name on inherited l2 rows

**Submitted on Sep 12th 2026 at 11:18:07 UTC by @kl4r10n for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92796
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Summary

The Explorer's reverse-resolution page accepts an ENS name returned from an address's `default.reverse` record even when ENSJS reports that the name does **not** forward-resolve to that address. The application then propagates only the unverified string to empty L2 rows and presents those rows as:

* `Forward match: True`
* `Primary name`
* a green `This is the primary name on <network>` confirmation banner

An address controls its own reverse record and can set that record to a name it does not own, such as `vitalik.eth`. Reverse-record control is therefore not proof of a name/address relationship; forward verification is the security boundary. ENSJS returns `match: false` for this case, but the Explorer discards that result for the inherited default name.

Consequently, an attacker can make the Explorer authoritatively claim that the attacker's address and another user's ENS name form a verified bidirectional relationship on every supported L2 where the attacker has no chain-specific reverse record.

## Impact and severity rationale

The reverse-resolution Explorer is intended to tell users whether an address-to-name reverse record is authenticated by a matching name-to-address forward record. This vulnerability changes an explicitly failed verification result into affirmative, application-authored trust indicators.

The attacker-controlled name is stored on-chain and remains displayed until the record is changed, so the false attestation is persistent. For each empty L2 row, the application converts an actual failed verification state into the attacker's chosen name plus `True` and `Primary name`. Opening the row adds a green confirmation banner.

High severity is proposed because the affected UI is the Explorer's security mechanism for distinguishing an arbitrary reverse string from a verified primary name. One `setName()` transaction on the attacker's own address persistently inverts that mechanism across every inherited L2 row: a known `match: false` becomes an application-authored `Forward match: True`, `Primary name`, and success banner. This is not a cosmetic rendering error in neutral metadata; it is a false identity attestation in the UI specifically intended to authenticate the address/name binding.

## ENS security invariant

ENSIP-19 defines primary-name resolution as a two-phase reverse-then-forward procedure. A reverse result is a primary name if and only if that name resolves to the original address. If the resolved address differs, no primary name exists and the application must display the address.

The official ENS integration guidance is equally explicit: after retrieving a reverse name, applications must forward-resolve it, and when the addresses do not match they must display the original address rather than the name because this prevents spoofing.

References:

* [ENSIP-19: Multichain Primary Names](https://docs.ens.domains/ensip/19/)
* [ENS documentation: Primary Names](https://docs.ens.domains/web/reverse/)
* [DefaultReverseRegistrar deployment source](https://github.com/ensdomains/ens-contracts/blob/staging/deployments/sepolia/DefaultReverseRegistrar.json)

The registrar's `setName(string)` sets the supplied string for `msg.sender`; it does not and should not establish name ownership. Safety comes from the subsequent forward-resolution check.

## Root cause

### 1. The default reverse lookup deliberately retains mismatches

[`useReverseResolution.ts#L236-L243`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts#L236-L243) performs the aggregate default lookup with `allowMismatch: true`:

```typescript
const defaultResult = await getName(l1Client, {
  address,
  coinType: DEFAULT_EVM_COIN_TYPE,
  allowMismatch: true,
})
const defaultName = defaultResult?.name ?? null

return ok(resolvedResults.map((r) => ({ ...r, defaultName })))
```

For an attacker address whose reverse record says `vitalik.eth`, ENSJS returns the equivalent of:

```typescript
{
  name: 'vitalik.eth',
  match: false,
  normalized: true,
}
```

The code copies `name` but discards `match`. Therefore, every row receives an unverified `defaultName` with no associated verification state.

This differs from the per-record conversion at [`useReverseResolution.ts#L97-L104`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts#L97-L104), which correctly preserves `nameResult.match` as `forwardMatch`.

### 2. The table hardcodes successful verification for inherited strings

When an L2 row has no chain-specific name but has any non-empty `defaultName`, [`columns.tsx#L96-L110`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx#L96-L110) renders success without consulting a match result:

```tsx
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
```

This creates a particularly clear contradiction: the Default row can show `vitalik.eth / Forward match: False`, while empty L2 rows derived from that same failed record show `vitalik.eth / Forward match: True / Primary name`.

### 3. The sidebar independently upgrades inheritance into primary-name status

[`computeDisplayNameState.ts#L76-L88`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts#L76-L88) defines inheritance itself as proof that the name is primary:

```typescript
const displayName = name || (defaultName && !isL1 ? defaultName : undefined)
const isInheritingDefault = !name && !!defaultName && !isL1
const isPrimaryName = forwardMatch || isInheritingDefault
```

The sidebar consumes this value and renders the green confirmation “This is the primary name on {label}” at [`ReverseResolutionSidebar.tsx#L644-L651`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx#L644-L651).

The banner condition is exactly `isPrimaryName && displayName`. The PoC proves both values are truthy for the inherited mismatched record, so the full sidebar deterministically takes the green success branch even though the PoC renders only the smaller table cell.

## Why this is not “working as designed”

Default-name inheritance is an intended ENSIP-19 feature. Treating an **unverified** default reverse string as a verified primary name is not.

The repository already contains a unit test at [`computeDisplayNameState.test.ts#L91-L104`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/utils/reverseResolution/computeDisplayNameState.test.ts#L91-L104) that codifies the vulnerable behavior:

```typescript
const result = computeDisplayNameState({
  name: null,
  defaultName: 'vitalik.eth',
  forwardMatch: false,
  reverseRegistrarChainId: 10,
})

expect(result).toEqual({
  displayName: 'vitalik.eth',
  isInheritingDefault: true,
  isPrimaryName: true,
  canSetAsPrimary: false,
})
```

That test documents current implementation behavior, but it cannot redefine the ENSIP-19 security invariant. ENSIP-19 permits an L2 to inherit the default reverse result only as part of primary-name resolution, which still requires the resulting name to forward-resolve to the queried address. Here, ENSJS has already performed that check and returned `match: false`.

The Default/L2 contradiction also rules out an intentional interpretation: the Default row preserves the same lookup as `Forward match: False`, while the derived L2 rows relabel it `True / Primary name`. A single verification result cannot be both false at its source and true merely because it was inherited. The intended fix is therefore not to remove inheritance, but to make inheritance carry and respect the default record's verification state.

## Attack scenario

1. The attacker controls address `A` but does not own or control `vitalik.eth`.
2. The attacker calls `setName("vitalik.eth")` on the Default Reverse Registrar from address `A`. The registrar permits an address to choose its own reverse string; this is expected behavior.
3. `vitalik.eth` does not have a default EVM forward address equal to `A`.
4. ENSJS looks up `A` with `coinType: 0x80000000` and `allowMismatch: true`, returning `name: "vitalik.eth"` and `match: false`.
5. A viewer opens `/addr/A/reverse-resolution`.
6. The Default row retains the failed match, but the aggregate query separately copies only `"vitalik.eth"` into `defaultName` for every row.
7. Each L2 without a chain-specific reverse name displays `vitalik.eth`, `Forward match: True`, and `Primary name`.
8. Opening an affected row shows a green banner affirming that `vitalik.eth` is the primary name for attacker address `A` on that network.

No ownership of `vitalik.eth`, compromise of another account, malformed Unicode, race condition, or stale previously-valid record is required.

## Why this is not a disclosed known issue

### Not `EXP-INPUT-005` — homoglyphs, bidi, and control characters

`EXP-INPUT-005` concerns confusing or transformed input caused by Unicode homoglyphs, bidirectional controls, other control characters, or normalization edge cases. Its stated impact is visual confusion rather than incorrect resolution.

This finding uses the normalized, ordinary ASCII name `vitalik.eth`; the mocked result explicitly has `normalized: true`. No lookalike characters or parsing ambiguity are involved. ENSJS reaches the correct security conclusion, `match: false`, because the name does not resolve to the attacker. The vulnerability occurs afterward when the application discards that security result and manufactures affirmative verification labels. Fixing input normalization does not fix this issue.

### Not `QA-06` — deleted, non-resolving subname remains displayed as primary

`QA-06` is the closest disclosed issue, but it describes a materially different lifecycle and attack surface: a user legitimately sets a subname as primary while it resolves, after which that subname is deleted or stops resolving; the stale primary remains visible and opening its profile errors.

This finding differs in every condition relevant to reproduction and remediation:

| Dimension                          | QA-06                                                     | This finding                                                              |
| ---------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------- |
| Initial validity                   | Name was resolving when set                               | Name never needs to resolve to attacker                                   |
| Trigger                            | Later deletion or resolution failure                      | Initial `default.reverse` value with `match: false`                       |
| Attacker authority over shown name | Previously legitimate relationship                        | No ownership or control of shown name                                     |
| Character requirements             | Deleted/non-resolving subname                             | Any normalized existing name, e.g. `vitalik.eth`                          |
| Vulnerable path                    | Stale primary/profile lifecycle                           | Default-record aggregation and L2 inheritance UI                          |
| Visible failure                    | Stale name and profile error                              | Explicit false verification badges and green banner                       |
| Root cause                         | Stale previously-valid primary is not removed/revalidated | `defaultResult.match` is discarded; inheritance is treated as proof       |
| Required fix                       | Repair stale-name invalidation/profile handling           | Preserve default verification state and gate all inherited trust UI on it |

Most importantly, this PoC does not delete anything and does not create a formerly valid primary name. The reverse record is unverified from its first lookup. A QA-06 fix limited to invalidating deleted subnames would leave this issue fully exploitable.

### Not `WEB-1301` — address view shows misleading owner and expiry metadata

`WEB-1301` concerns owner, registration, and expiry fields being sourced from a wallet's primary name in the general address view. This report affects the dedicated reverse-resolution route and falsifies the security result itself: `Forward match: False` becomes `True / Primary name`. Correcting owner or expiry metadata would not preserve `defaultResult.match` or change the hardcoded inherited-row badges.

Therefore, the finding is not merely a new example of a disclosed behavior. It has an independent entry point, prerequisite, root cause, affected UI, PoC, and required patch.

## Recommended remediation

Do not propagate a default reverse string as an authenticated identity unless the default lookup passed forward verification.

The safest minimal change is to suppress mismatched defaults:

```typescript
const defaultResult = await getName(l1Client, {
  address,
  coinType: DEFAULT_EVM_COIN_TYPE,
  allowMismatch: true,
})

const defaultName = defaultResult?.match ? defaultResult.name : null
```

Alternatively, if the UI intentionally needs to display mismatched reverse records for diagnostics, propagate the verification result explicitly:

```typescript
const defaultName = defaultResult?.name ?? null
const defaultForwardMatch = defaultResult?.match ?? false

return ok(
  resolvedResults.map((row) => ({
    ...row,
    defaultName,
    defaultForwardMatch,
  })),
)
```

Then require `defaultForwardMatch` everywhere that inheritance is described as primary:

```typescript
const isInheritingVerifiedDefault =
  !name && !!defaultName && defaultForwardMatch && !isL1

const isPrimaryName = forwardMatch || isInheritingVerifiedDefault
```

The table must derive its `True` and `Primary name` badges from the same verified state rather than hardcoding success whenever `defaultName` is non-empty. A mismatched reverse string may be shown as diagnostic data only if it is clearly labelled `Forward match: False` and never receives primary-name styling or a success banner.

## Proof of Concept

## Proof of concept

The following test uses the real production query projection, the real table column renderer, and the real sidebar state helper. Only external chain reads are mocked:

* `getName()` returns the exact adversarial but valid library result: an ASCII normalized name with `match: false`.
* the L2 registrar lookup returns no chain-specific record, creating the empty row to which `default.reverse` is supposed to fall back.
* all vulnerable application logic remains unmodified.

Create this file as:

`apps/portal/src/features/reverse-resolution/hooks/F01UnverifiedDefaultReverse.poc.test.tsx`

```tsx
import { render, screen } from '@testing-library/react'
import { ok } from 'neverthrow'
import type { ReactElement } from 'react'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { computeDisplayNameState } from '@/utils/reverseResolution/computeDisplayNameState'
import { columns } from '../components/ReverseResolutionTable/columns'
import type { ReverseResolutionNetwork } from '../config'
import {
  getReverseResolutionQueryOptions,
  type ReverseResolutionResult,
} from './useReverseResolution'

const ATTACKER = '0x1111111111111111111111111111111111111111' as const
const SPOOFED_NAME = 'vitalik.eth'
const DEFAULT_EVM_COIN_TYPE = 0x80000000
const OP_SEPOLIA_CHAIN_ID = 11155420

const getNameMock = vi.fn()

vi.mock('@ensdomains/ensjs/public', () => ({
  getName: (...args: unknown[]) => getNameMock(...args),
  getAddressRecord: vi.fn(),
  getReverseRecordFromRegistry: vi.fn(),
}))

vi.mock('@/lib/wagmi/helpers', () => ({
  safeGetClient: vi.fn(() => ok({})),
}))

// Returning no registrar makes the L2 query produce the same empty row seen
// when an address has no chain-specific reverse record. The default.reverse
// fallback is still resolved through the real production query code.
vi.mock('@ens-apps/l2-primary/v1', () => ({
  getRegistrarAddress: vi.fn(() => undefined),
  getChainIdForReverseRegistrarChainId: vi.fn(),
  l2ReverseRegistrarNameForAddrSnippet: [],
}))

const networks: ReverseResolutionNetwork[] = [
  {
    coinType: DEFAULT_EVM_COIN_TYPE,
    label: 'Default',
    icon: '/icons/Link.svg',
  },
  {
    coinType: DEFAULT_EVM_COIN_TYPE + OP_SEPOLIA_CHAIN_ID,
    reverseRegistrarChainId: 10,
    label: 'Optimism',
    icon: '/icons/Optimism.svg',
  },
]

describe('F-01: unverified default.reverse is presented as an L2 primary name', () => {
  beforeEach(() => {
    getNameMock.mockReset()
    getNameMock.mockResolvedValue({
      name: SPOOFED_NAME,
      match: false,
      normalized: true,
      reverseResolverAddress: null,
      resolverAddress: null,
    })
  })

  it('turns match:false into Forward match True and Primary name', async () => {
    const options = getReverseResolutionQueryOptions({
      address: ATTACKER,
      networks,
    })
    const queryFn = options.queryFn as (
      context: unknown,
    ) => Promise<ReverseResolutionResult[]>
    const rows = await queryFn({})

    const defaultRow = rows.find((row) => row.label === 'Default')
    const optimismRow = rows.find((row) => row.label === 'Optimism')
    if (!defaultRow || !optimismRow) {
      throw new Error('PoC setup failed to produce both expected rows')
    }

    // ENSJS correctly reports that vitalik.eth does not resolve to the
    // attacker. The Default row preserves that result.
    expect(getNameMock).toHaveBeenCalledWith(
      expect.anything(),
      expect.objectContaining({
        address: ATTACKER,
        coinType: DEFAULT_EVM_COIN_TYPE,
        allowMismatch: true,
      }),
    )
    expect(defaultRow).toMatchObject({
      name: SPOOFED_NAME,
      forwardMatch: false,
    })

    // The empty L2 row receives only the name string. The default record's
    // match:false result is not propagated.
    expect(optimismRow).toMatchObject({
      name: null,
      defaultName: SPOOFED_NAME,
      forwardMatch: false,
    })

    // The real sidebar helper treats the inherited, unverified name as a
    // primary name despite forwardMatch remaining false.
    const displayState = computeDisplayNameState({
      name: optimismRow.name,
      defaultName: optimismRow.defaultName,
      forwardMatch: optimismRow.forwardMatch,
      reverseRegistrarChainId: optimismRow.reverseRegistrarChainId ?? 10,
    })
    expect(displayState).toEqual({
      displayName: SPOOFED_NAME,
      isInheritingDefault: true,
      isPrimaryName: true,
      canSetAsPrimary: false,
    })

    // Render the production Forward match table cell. It manufactures both
    // affirmative badges even though the row's forwardMatch value is false.
    const forwardMatchColumn = columns.find(
      (column) =>
        'accessorKey' in column && column.accessorKey === 'forwardMatch',
    )
    if (!forwardMatchColumn || typeof forwardMatchColumn.cell !== 'function') {
      throw new Error('PoC setup failed to locate the Forward match cell')
    }
    const cell = forwardMatchColumn.cell as (context: unknown) => ReactElement
    render(cell({ row: { original: optimismRow } }))

    expect(screen.getByText('True')).toBeTruthy()
    expect(screen.getByText('Primary name')).toBeTruthy()
    expect(screen.getByText('Default')).toBeTruthy()
  })
})
```

### Run the PoC

From `apps/portal`:

```bash
./node_modules/.bin/vitest run \
  src/features/reverse-resolution/hooks/F01UnverifiedDefaultReverse.poc.test.tsx \
  --reporter=verbose
```

### Execution result

```
RUN  v4.1.5

✓ src/features/reverse-resolution/hooks/F01UnverifiedDefaultReverse.poc.test.tsx
  > F-01: unverified default.reverse is presented as an L2 primary name
  > turns match:false into Forward match True and Primary name 12ms

Test Files  1 passed (1)
     Tests  1 passed (1)
  Duration  916ms (transform 61ms, setup 63ms, import 578ms,
                   tests 14ms, environment 160ms)
```

The passing test establishes all of the following in one execution:

1. ENSJS reports `match: false` for the attacker/name pair.
2. The Default row correctly retains `forwardMatch: false`.
3. The inherited Optimism row contains `forwardMatch: false` and the spoofed `defaultName`.
4. The sidebar production helper nevertheless returns `isPrimaryName: true`.
5. The production table cell nevertheless renders `True`, `Primary name`, and `Default`.
6. The same state satisfies the sidebar's `isPrimaryName && displayName` success condition, which renders the cited green primary-name banner.
