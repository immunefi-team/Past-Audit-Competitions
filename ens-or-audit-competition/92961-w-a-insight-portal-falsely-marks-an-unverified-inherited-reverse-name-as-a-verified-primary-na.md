# 92961 w a insight portal falsely marks an unverified inherited reverse name as a verified primary name

**Submitted on Sep 13th 2026 at 14:06:10 UTC by @deltasec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92961
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The portal's reverse-resolution inspector can display an attacker-chosen ENS name as the verified primary name for the attacker's address, even when the name does not forward-resolve to that address.

An address owner may set their own `default.reverse` record to any ENS name string, including a high-trust name they do not own. The portal correctly asks ENSJS for this inherited default reverse name with `allowMismatch: true`, but it then discards the returned forward-match result. For L2 rows that inherit this default reverse name, the UI hardcodes `Forward match: True` and marks the row as `Primary name`.

If exploited in production, a user inspecting an address can be shown a false verified identity claim: an attacker-controlled address appears to have a verified primary name such as `vitalik.eth`. This can mislead users who rely on the portal's verification badges before sending funds or interacting with an address.

## Vulnerability Details

Reverse resolution is only trustworthy when the reverse name also forward-resolves back to the queried address. The codebase models this explicitly with a `forwardMatch` flag.

For a row's own reverse record, the portal preserves ENSJS' `match` result:

```ts
forwardMatch: nameResult?.match ?? false,
```

Affected code:

* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:97-104`

The inherited `default.reverse` path is different. It requests an unverified answer by passing `allowMismatch: true`, but then keeps only the name and drops the returned `match` value:

```ts
const defaultResult = await getName(l1Client, {
  address,
  coinType: DEFAULT_EVM_COIN_TYPE,
  allowMismatch: true,
})
const defaultName = defaultResult?.name ?? null
```

Affected code:

* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-241`

The table renderer then treats any inherited default name as verified. When the L2 row has no direct reverse name and can inherit `default.reverse`, the `forwardMatch` column ignores `row.original.forwardMatch` and hardcodes the positive badges:

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

Affected code:

* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:91-111`

The shared display-state helper repeats the same flawed assumption. It treats inheritance itself as sufficient for primary-name status:

```ts
const isPrimaryName = forwardMatch || isInheritingDefault
```

Affected code:

* `apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:76-83`

### Attack path

1. The attacker controls address `0xATTACKER`.
2. The attacker sets the address's `default.reverse` record to a trusted name they do not control, for example `vitalik.eth`.
3. The attacker cannot make `vitalik.eth` forward-resolve to `0xATTACKER`, so a correct forward-match check would return false.
4. A user opens the portal reverse-resolution view for `0xATTACKER`.
5. For L2 rows without their own reverse record, the portal inherits the attacker's `default.reverse` name.
6. The portal displays the inherited name with `Forward match: True` and `Primary name`, even though the forward match was not performed or preserved on that path.

## Impact Details

An attacker can cause the portal to falsely present an attacker-controlled address as having a verified primary ENS name belonging to someone else.

The practical impact is identity substitution in the portal's address verification surface. Users commonly rely on primary-name and forward-match status to decide whether an address corresponds to an expected person, organization, or counterparty. By displaying `Forward match: True` and `Primary name` for an unverified inherited name, the portal can lead a user to trust the attacker's address as if it were associated with the displayed ENS name.

This creates a realistic path to misdirected funds or unintended transactions: a victim who checks an address in the portal before paying or interacting with it can be shown that the address is the verified primary address for a trusted ENS name, when in fact the attacker merely set their own reverse record to that name.

The attacker does not need to compromise the trusted ENS name, modify its forward records, or interact with the victim. The false verification is produced by the portal's handling of the attacker's own `default.reverse` record.

## References

* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:97-104`
* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-241`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:91-111`
* `apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:76-83`

## Proof of Concept

## Proof of Concept

The following inline Vitest reproducers exercise the real exported data-layer and UI helper code with controls.

### How to run

1. In a checkout of the project, go to the portal package:

```bash
cd apps/portal
```

2. Save the first code block below as `default-reverse-data.poc.test.ts`.
3. Save the second code block below as `default-reverse-ui.poc.test.tsx`.
4. Run both reproducers with Vitest:

```bash
pnpm vitest run default-reverse-data.poc.test.ts default-reverse-ui.poc.test.tsx
```

Expected result: all tests pass. The passing assertions show that an inherited `default.reverse` name whose forward match is false is still rendered as `Forward match: True` and `Primary name`.

Data-layer reproducer:

```ts
/**
 * Data-layer half of the same defect: `getReverseResolution` asks `getName` for
 * an unverified answer and then drops the verification flag on the floor.
 *
 * Drives the REAL exported `getReverseResolutionQueryOptions().queryFn`, with
 * only `@ensdomains/ensjs/public` and the wagmi client stubbed. `getName` is
 * made to answer the way the chain answers for an address whose
 * `default.reverse` record points at a name that does NOT forward-resolve back
 * to it: `{ name: 'vitalik.eth', match: false }`.
 */
import { okAsync } from 'neverthrow'
import { describe, expect, it, vi } from 'vitest'

const H = vi.hoisted(() => ({
  getNameCalls: [] as Array<Record<string, unknown>>,
  defaultAnswer: { name: 'vitalik.eth', match: false, normalized: true } as
    | Record<string, unknown>
    | null,
}))

vi.mock('@ensdomains/ensjs/public', () => ({
  // every per-network read returns "no record of its own"
  getName: vi.fn(async (_client: unknown, args: Record<string, unknown>) => {
    H.getNameCalls.push(args)
    // 0x80000000 is DEFAULT_EVM_COIN_TYPE — the `default.reverse` read
    return args.coinType === 0x80000000 ? H.defaultAnswer : null
  }),
  getReverseRecordFromRegistry: vi.fn(async () => null),
  getAddressRecord: vi.fn(async () => null),
}))

vi.mock('@/lib/wagmi/helpers', () => ({
  safeGetClient: () => okAsync({} as never),
}))

vi.mock('@/lib/wagmiL2', () => ({ l2WagmiConfig: {} }))

const { getReverseResolutionQueryOptions } = await import(
  '@/features/reverse-resolution/hooks/useReverseResolution'
)

const ATTACKER = '0x00000000000000000000000000000000DeaDBeef' as const

/** Two networks: mainnet (coin 60) and one L2, matching the real config shape. */
const networks = [
  { coinType: 60, label: 'Ethereum', icon: '', reverseRegistrarChainId: undefined },
  {
    coinType: 0x80000000 | 8453,
    label: 'Base',
    icon: '',
    reverseRegistrarChainId: 8453,
  },
] as never

describe('S12 data layer — the default.reverse match flag is discarded', () => {
  it('asks for an unverified answer and keeps only the name', async () => {
    H.getNameCalls = []
    const options = getReverseResolutionQueryOptions({ address: ATTACKER, networks })
    const result = await (options.queryFn as (c: unknown) => Promise<unknown>)({
      queryKey: options.queryKey,
    } as never)

    const rows = (result as { value?: unknown }).value ?? result
    console.log('getName calls ->', H.getNameCalls)
    console.log('rows          ->', JSON.stringify(rows, null, 2))

    // 1. the default.reverse read explicitly opts out of verification
    const defaultCall = H.getNameCalls.find((c) => c.coinType === 0x80000000)
    expect(defaultCall).toBeDefined()
    expect(defaultCall?.allowMismatch).toBe(true)

    // 2. the chain said match:false; every row carries the name anyway
    expect(H.defaultAnswer?.match).toBe(false)
    for (const row of rows as Array<Record<string, unknown>>) {
      expect(row.defaultName).toBe('vitalik.eth')
      // 3. and there is nowhere for the flag to live: the row type has
      //    `forwardMatch` for the row's OWN record only
      expect(row.forwardMatch).toBe(false)
      expect(Object.keys(row)).not.toContain('defaultForwardMatch')
      expect(Object.keys(row)).not.toContain('defaultMatch')
    }
  })

  it('CONTROL: when the chain reports no default.reverse record, nothing is inherited', async () => {
    H.getNameCalls = []
    H.defaultAnswer = null
    const options = getReverseResolutionQueryOptions({ address: ATTACKER, networks })
    const result = await (options.queryFn as (c: unknown) => Promise<unknown>)({
      queryKey: options.queryKey,
    } as never)
    const rows = (result as { value?: unknown }).value ?? result
    console.log('CONTROL rows ->', JSON.stringify(rows))
    for (const row of rows as Array<Record<string, unknown>>) {
      expect(row.defaultName).toBeNull()
    }
    H.defaultAnswer = { name: 'vitalik.eth', match: false, normalized: true }
  })
})

/**
 * The same discard, one layer up. `computeDisplayNameState` is the shared
 * helper the detail sidebar uses, and it defines
 *
 *     isPrimaryName = forwardMatch || isInheritingDefault
 *
 * so inheritance alone asserts primacy. The sidebar then renders a green
 * success banner "This is the primary name on {label}"
 * (ReverseResolutionSidebar.tsx:645) and a "Primary name: True" badge (`:689`).
 * Its negative branch reads "The set address does not resolve back to this name
 * on {label}" (`:654`) — which is precisely the check that was skipped.
 */
describe('S12 — computeDisplayNameState asserts primacy from inheritance alone', () => {
  it('CONTROL: an own record with forwardMatch false is not primary', async () => {
    const { computeDisplayNameState } = await import(
      '@/utils/reverseResolution/computeDisplayNameState'
    )
    const state = computeDisplayNameState({
      name: 'attacker.eth',
      defaultName: null,
      forwardMatch: false,
      reverseRegistrarChainId: 10 as never,
    })
    console.log('CONTROL ->', state)
    expect(state.isPrimaryName).toBe(false)
  })

  it('an inherited default with forwardMatch false IS reported primary', async () => {
    const { computeDisplayNameState } = await import(
      '@/utils/reverseResolution/computeDisplayNameState'
    )
    const state = computeDisplayNameState({
      name: null,
      defaultName: 'vitalik.eth',
      forwardMatch: false,
      reverseRegistrarChainId: 10 as never,
    })
    console.log('INHERITED ->', state)

    expect(state.displayName).toBe('vitalik.eth')
    expect(state.isInheritingDefault).toBe(true)
    // drives the green "This is the primary name on {label}" banner
    expect(state.isPrimaryName).toBe(true)
    // and the user is not offered the chance to set a real one
    expect(state.canSetAsPrimary).toBe(false)
  })
})
```

UI-renderer reproducer:

```tsx
/**
 * The inherited default reverse path calls `getName` with `allowMismatch: true`
 * and keeps only the returned name:
 *
 *     const defaultResult = await getName(l1Client, {
 *       address, coinType: DEFAULT_EVM_COIN_TYPE, allowMismatch: true,
 *     })
 *     const defaultName = defaultResult?.name ?? null      // .match dropped
 *
 * Every direct reverse row threads the real `match` flag into `forwardMatch`.
 * This inherited path throws the flag away, and the table then hardcodes
 * "Forward match: True" and "Primary name" for inherited default names.
 *
 * A reverse record is self-sovereign: an address owner can point it at any
 * name string they like. The forward-match check is the entire defence against
 * that, and on this path the UI asserts a check that was never performed.
 *
 * These tests drive the REAL exported `columns` cell renderers. Nothing is
 * reimplemented.
 */
import { render } from '@testing-library/react'
import { describe, expect, it } from 'vitest'
import { columns } from '@/features/reverse-resolution/components/ReverseResolutionTable/columns'
import { DEFAULT_EVM_COIN_TYPE, MAINNET_COIN_TYPE } from '@/lib/coinType'
import type { ReverseResolutionResult } from '@/features/reverse-resolution/hooks/useReverseResolution'

/** An L2 coin type — anything that is neither `default.reverse` nor coin 60. */
const BASE_COIN_TYPE = 0x80000000 | 8453

const baseRow = (over: Partial<ReverseResolutionResult>): ReverseResolutionResult =>
  ({
    label: 'Base',
    icon: undefined,
    coinType: BASE_COIN_TYPE,
    name: null,
    reverseResolverAddress: null,
    resolverAddress: null,
    normalized: true,
    forwardMatch: false,
    defaultName: null,
    ...over,
  }) as ReverseResolutionResult

const colByKey = (key: string) => {
  const col = columns.find((c) => (c as { accessorKey?: string }).accessorKey === key)
  if (!col?.cell || typeof col.cell !== 'function') throw new Error(`no cell for ${key}`)
  return col.cell
}

const renderCell = (key: string, original: ReverseResolutionResult) => {
  const cell = colByKey(key)
  // minimal CellContext: these renderers only read `row.original`
  return render(<>{cell({ row: { original } } as never)}</>)
}

describe('S12 — default.reverse inheritance asserts an unperformed check', () => {
  it('CONTROL: a row with its own name and match=false is reported False', () => {
    const { container } = renderCell(
      'forwardMatch',
      baseRow({ name: 'attacker.eth', forwardMatch: false }),
    )
    console.log('CONTROL (own record, match=false) ->', container.textContent)
    expect(container.textContent).toContain('False')
    expect(container.textContent).not.toContain('Primary name')
  })

  it('CONTROL: a row with its own name and match=true is reported True + Primary name', () => {
    const { container } = renderCell(
      'forwardMatch',
      baseRow({ name: 'legit.eth', forwardMatch: true }),
    )
    console.log('CONTROL (own record, match=true) ->', container.textContent)
    expect(container.textContent).toContain('True')
    expect(container.textContent).toContain('Primary name')
  })

  it('an inherited default.reverse name is asserted True + Primary name with forwardMatch FALSE', () => {
    const row = baseRow({
      name: null,                    // this L2 has no reverse record of its own
      defaultName: 'vitalik.eth',    // attacker's self-set default.reverse
      forwardMatch: false,           // ...and it does not forward-resolve back
    })

    const nameCell = renderCell('name', row)
    const matchCell = renderCell('forwardMatch', row)

    console.log('Name column         ->', nameCell.container.textContent)
    console.log('Forward match column->', matchCell.container.textContent)

    // the name the attacker chose is rendered for this network
    expect(nameCell.container.textContent).toContain('vitalik.eth')

    // ...and the verification badge says True even though the row says false
    expect(row.forwardMatch).toBe(false)
    expect(matchCell.container.textContent).toContain('True')
    expect(matchCell.container.textContent).not.toContain('False')
    expect(matchCell.container.textContent).toContain('Primary name')
  })

  it('the badge is unconditional: it ignores forwardMatch entirely on this path', () => {
    for (const forwardMatch of [true, false]) {
      const { container } = renderCell(
        'forwardMatch',
        baseRow({ name: null, defaultName: 'vitalik.eth', forwardMatch }),
      )
      console.log(`forwardMatch=${forwardMatch} ->`, container.textContent)
      expect(container.textContent).toContain('True')
    }
  })

  it('scope: the fabricated badge is L2-only — mainnet and the Default row are unaffected', () => {
    for (const coinType of [MAINNET_COIN_TYPE, DEFAULT_EVM_COIN_TYPE]) {
      const { container } = renderCell(
        'forwardMatch',
        baseRow({ coinType, name: null, defaultName: 'vitalik.eth', forwardMatch: false }),
      )
      console.log(`coinType=${coinType} ->`, container.textContent)
      expect(container.textContent).toContain('False')
    }
  })
})
```
