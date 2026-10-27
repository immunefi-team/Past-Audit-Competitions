# 92593 w a critical explorer s extend flow charges the user to renew a stranger s ens name instead of the one shown

**Submitted on Sep 10th 2026 at 20:15:42 UTC by @v4rvl for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92593
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

`.eth` labels are not normalized on-chain. The registrar hashes the label string it is handed, so `Nike.eth` and `nike.eth` are two independent registrations with two different owners. Normalizing a label before hashing therefore does not canonicalise a name; it selects a different registration.

The Explorer's Extend flow displays the name as the indexer returned it and normalizes it for every read and write it performs. `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:197` builds the `renew()` calldata from `getLabel(params.name)`; `apps/portal/src/features/renew/hooks/useIsRenewable.ts:40` asks the renewer's eligibility gate about `getLabel(name)`; `apps/portal/src/features/register/hooks/useRegistrationPrice.ts:54` quotes the price for `getLabel(name)`. The strings on screen come from `params.name` unchanged (`:221`, `:291-292`, `:459-460`).

The mixed-case value can reach this flow through the production v1 names route. The route calls `getV1NamesForAddressQueryOptions`, which delegates to ensjs `getNamesForAddress`; ensjs's `makeNameObject` assigns `name: decryptName(domain.name)`, and `decryptName` only expands encoded labelhash placeholders. It does not apply ENSIP-15 normalization. `names.tsx:178-181` then copies that returned `name` into the row, and `getSelectedNames` copies it into `SelectedName` without a normalization or equality check. This is the production bridge for the v1 arm; the local fork is used only to demonstrate the deployed contract consequence.

Measured end to end on a local Anvil Sepolia fork for a user who owns `Nike.eth` and selects it:

| Quantity                       | Value the flow uses                   | Value for the name shown                                   |
| ------------------------------ | ------------------------------------- | ---------------------------------------------------------- |
| eligibility gate `isRenewable` | `'nike'` -> `true`                    | `'Nike'` -> `false`                                        |
| price quoted and approved      | `getRenewPrice('nike')` = `160000009` | `getRenewPrice('Nike')` reverts `NameNotRenewable("Nike")` |
| `renew()` label in calldata    | `"nike"`                              | never referenced                                           |

The submitted transaction succeeds. The user's USDC balance falls by `160000009`; `nameExpires` of `nike.eth`, owned by `0x7F1266aa4E48567f42E02E9dA040eeec33EDd5Cb`, rises by exactly `31536000` seconds; `nameExpires` of the user's own `Nike.eth` changes by `0`; and a `160000009` allowance survives on the renewer.

The team already knows the two forms differ. The Manager's bulk-renew builder carries the comment "the display name may be unnormalized (e.g. mixed case) and would hash to the wrong label" at `apps/manager/src/features/dashboard/components/NamesTable.tsx:84-85`, and the repo's own fixture at `apps/manager/src/features/dashboard/v2NameRoles.test.ts:43` is `makeDomain({ name: 'Alaska.eth', normalizedName: 'alaska.eth' })`. The Explorer's renew flow has no equivalent separation: it holds one string, prints it raw and hashes it normalized, so the two can never be compared.

## Who loses what

| Actor                                          | Opening state                                  | After the flow                                | Entitled change                            | Evidence                     |
| ---------------------------------------------- | ---------------------------------------------- | --------------------------------------------- | ------------------------------------------ | ---------------------------- |
| User (owns `Nike.eth`), `0xbeEF...f1F1`        | 1000000000 USDC; `Nike.eth` expires 1820602402 | 839999991 USDC; `Nike.eth` expires 1820602402 | pays for and receives a year on `Nike.eth` | fork transcript below        |
| Third party (owns `nike.eth`), `0x7F1266aa...` | `nike.eth` expires 2262550824                  | `nike.eth` expires 2294086824                 | no change; paid nothing                    | fork transcript below        |
| Renewer contract `0x4ad56feb...`               | allowance 0                                    | allowance 160000009                           | 0 after a completed renewal                | `allowance()` after the flow |

The user is told the name renewed. `handleDone` at `useRenewalTransactions.ts:365-373` collects `renewedNames` from `flow.selectedName.name` / `flow.renewals[].selectedName.name`, which is the un-normalized name, and at `:386` invalidates the expiry queries for that name, so the UI refetches and reports on the registration the transaction never touched.

## Root cause

The intent builder replaces the name with its normalized twin. `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:190-203`:

```ts
function buildRenewIntent(params: RenewParams): CustomTransactionIntent {
  // ensjs splits the label without normalizing, so pass a normalized 2LD name.
  const writeParams = renewNameWriteParameters(
    params.publicClient as unknown as Parameters<
      typeof renewNameWriteParameters
    >[0],
    {
      name: `${getLabel(params.name)}.eth`,
      duration: BigInt(params.duration),
      paymentToken: params.tokenAddress,
      referrer: REFERER_ADDRESS,
      contract: params.isV2 ? 'ensEthRegistrar' : 'ensEthRenewerV1',
    },
  )
```

That comment is about ensjs, not about the name, and its remedy is what produces the divergence. `renewNameWriteParameters` does split without normalizing - `const [label] = name.split('.');` at line 24 of `@ensdomains/ensjs/dist/actions/wallet/renewName.js` - but an `.eth` label is not case- or Unicode-folded on-chain, so normalizing the input before handing it over changes which registration the call names. Nothing at the site reconciles the substituted label with the name printed 24 lines later at `:221`, and no caller compares them.

`getLabel` is real ENSIP-15 normalization, `apps/portal/src/utils/token/getLabel.ts:10-19`:

```ts
export function getLabel(name: string): string {
  const normalized = ens_normalize(name)
  const labels = ens_split(normalized)
  const first = labels[0]

  if (!first || first.error) {
    throw first?.error
  }

  return String.fromCodePoint(...first.input)
}
```

Every string the user reads is built from `params.name` / `selectedName.name` instead, un-normalized. `useRenewalTransactions.ts:220-223`:

```ts
  transactionManager.startTransaction(buildRenewIntent(params), signer, {
    id: RENEWAL_TX_IDS.renew(params.name),
    publicClient: params.publicClient,
    description: `Renew ${params.name}`,
  })
```

`useRenewalTransactions.ts:457-460` (single-name flow):

```ts
      const renewTx: Transaction = {
        id: RENEWAL_TX_IDS.renew(single.selectedName.name),
        title: `Extend ${single.selectedName.name}`,
        transactionName: `Extend ${single.selectedName.name}`,
```

`useRenewalTransactions.ts:290-292` (batch flow):

```ts
          id: RENEWAL_TX_IDS.renew(step.name),
          title: `Extend ${step.name}`,
          transactionName: `Extend ${step.name}`,
```

The same substitution is applied to both chain reads the flow makes, which is why no gate catches it.

The eligibility gate, `apps/portal/src/features/renew/hooks/useIsRenewable.ts:38-46`:

```ts
  const client = yield* safeGetClient()
  const label = yield* fromSync(
    () => getLabel(name),
    (cause) =>
      new IsRenewableError({ cause: cause as UnsupportedNameTypeError }),
  )
  const renewable = yield* fromPromise(
    ensjs_isRenewable(client, { renewerAddress, label }),
```

The price, `apps/portal/src/features/register/hooks/useRegistrationPrice.ts:46-58`, shared by the register and renew price paths:

```ts
export const resolvePriceInputs = ResultFn(async function* ({
  name,
  duration,
  token,
}: BasePriceParameters) {
  const client = yield* safeGetClient()
  const paymentToken = token ?? SUPPORTED_TOKENS.USDC
  const label = yield* fromSync(
    () => getLabel(name),
    (cause) =>
      new GetRegistrationPriceError({
        cause: cause as UnsupportedNameTypeError,
      }),
```

One `price` object drives both what the confirmation screen renders and the amount that becomes the approval, `apps/portal/src/features/renew/components/ExtendNameModal.tsx:110-123`:

```tsx
          .with('confirm', () =>
            price ? (
              <ExtendNameConfirmation
                selectedName={selectedName}
                durationSeconds={durationSeconds}
                price={price}
                onBack={() => setStep('settings')}
                onConfirm={(token) =>
                  onExtend({
                    duration: durationSeconds,
                    tokenAddress: token.address,
                    tokenPrice: token.price.total,
                    tokenAllowance: token.allowance,
                  })
```

and the approval asks for twice it, `useRenewalTransactions.ts:157`: `amount: params.tokenPrice * 2n`.

### How an un-normalized name reaches this flow

1. `apps/portal/src/routes/addr/$addr/names.tsx:172-173` fetches the address's names through `getV1NamesForAddressQueryOptions` and `getV2NamesWithRolesForAddressQueryOptions`.
2. The v1 query is ensjs's subgraph `getNamesForAddress`, whose `makeNameObject` sets `name: decrypted` where `decrypted = decryptName(domain.name)`. `decryptName` only expands `[labelhash]` placeholders against a local label cache; it applies no ENSIP-15 normalization. The registered label string arrives verbatim.
3. `names.tsx:177-204` maps each result straight into a `NameRow` with `name` unchanged.
4. `getSelectedNames` copies it again without touching it, `apps/portal/src/features/renew/utils/nameExtension.ts:11-23`:

```ts
export const getSelectedNames = (
  rowSelection: RowSelectionState,
  filteredData: NameRow[],
): readonly SelectedName[] =>
  Object.keys(rowSelection)
    .map((idx) => filteredData[Number(idx)])
    .filter((row): row is NameRow => Boolean(row))
    .filter((row): row is NameRow & { name: string } => row.name !== null)
    .map((row) => ({
      name: row.name,
      isV2: row.protocolVersion === 'ENSv2',
      expiryDate: row.expiryDate,
    }))
```

5. The client-side gate is a shape regex over that same raw string, `nameExtension.ts:61-66`:

```ts
export const isExtendable2LD = ({
  name,
  isV2,
  expiryDate,
}: SelectedName): boolean => {
  if (!/^[^.]+\.eth$/.test(name)) return false
```

6. The only additional gate for a v1 name is `isRenewable`, which as shown above is asked about the substituted label. For v2 names there is no chain gate at all: `names.tsx:115` states "v2 is fully covered by `isExtendable2LD` and passes through untouched", implemented at `:132` as `candidates.filter((name) => name.isV2 || isRenewable(name.name))`.

The reachable arm is the v1 one, and it is the arm measured throughout this report. The two lists have different upstreams:

* **v1 rows are unfiltered.** `getNamesForAddress` queries the ENS subgraph, and `makeNameObject` passes `domain.name` through `decryptName` only, so a registered label that is not ENSIP-15-normalized arrives verbatim.
* **v2 rows are filtered upstream.** `getV2NamesWithRolesForAddressQueryOptions` sends `domains(where: { owner: $account })` to the ens.dev indexer (`apps/portal/src/features/dashboard/hooks/useV2NamesWithRolesForAddress.ts:55`) and sets neither `isNormalized` nor `includeUnnormalized`. That indexer's `DomainFilter.isNormalized` documents the default as "When omitted, only normalized names are returned (non-normalized names like `Nick.eth` are hidden by default)". Measured against that endpoint, the owner `0xe9fc00df05c7f149c10351fe1d30c6965b9a4cf8` holds `AgeisHookTreasury.eth` with `normalizedName` `ageishooktreasury.eth` and `isNormalized` `false`: an owner-scoped query without the flag returns 0 rows, and the same query with `includeUnnormalized: true` returns 1. So a v2 row whose displayed name is not normalized does not reach this flow today.

The PoC's second test drives four more label shapes with `isV2: true` to show the builder substitutes regardless of which registrar is selected. The production claim in this report rests on the v1 path.

## Steps to reproduce

1. Own a `.eth` 2LD whose first label is not ENSIP-15-normalized. Measured example: `Nike.eth`, registered on the fork to `0xBeeF00000000000000000000000000000000f1F1`. Its normalized twin `nike.eth` is a different registration owned by `0x7F1266aa4E48567f42E02E9dA040eeec33EDd5Cb`.
2. Open the Explorer at `/addr/0xBeeF00000000000000000000000000000000f1F1/names`.
3. The row for `Nike.eth` is offered as extendable: `isExtendable2LD` tests `/^[^.]+\.eth$/` against the raw name and passes, and the v1 renewability gate resolves `true` because it queries `isRenewable('nike')`.
4. Select the row and click Extend. The modal header and the transaction row both read `Extend Nike.eth`, and the price shown is `getRenewPrice('nike')` = `160.000009 USDC`.
5. Approve. Two transactions are submitted: `approve(renewer, 320000018)` and `renew("nike", 31536000, USDC, 0x0)`.
6. Read back `nameExpires` for both labelhashes and the user's USDC balance.

## Proof of Concept

## Proof of concept

### Part 1 - the application layer

Runnable with the repo's own vitest from `apps/portal`. It renders the real `useRenewalTransactions` hook, starts a real single-name Extend flow, and reads back both the strings the modal renders and the decoded `renew` calldata the modal's own `intent.prepare` produces. The mocks replace wagmi's connection accessors and the transaction-modal hook; the label under test is produced by the real `getLabel` and the real `renewNameWriteParameters`, and nothing mocked touches it.

Save as `apps/portal/src/norm_renew_label_divergence.test.tsx`:

```tsx
/**
 * The Explorer's Extend/Renew flow labels the transaction with the name it was
 * given and builds the calldata from the ENS-normalized twin of that name.
 * Drives the real `useRenewalTransactions` hook; the only mocks are wagmi's
 * connection accessors and the transaction-modal hook, neither of which touches
 * the label.
 */
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { renderHook, act } from '@testing-library/react'
import type { ReactNode } from 'react'
import { type Address, decodeFunctionData, labelhash } from 'viem'
import { beforeAll, describe, expect, it, vi } from 'vitest'

const CONNECTED = '0x00000000000000000000000000000000000000A1' as Address

// Holder assigned in beforeAll. Importing `@/lib/wagmi` inside the mock factory
// would re-enter the module that is being mocked and deadlock the import graph.
const holder = vi.hoisted(() => ({ publicClient: null as unknown }))

vi.mock('wagmi', async (importOriginal) => {
  const actual = await importOriginal<typeof import('wagmi')>()
  return {
    ...actual,
    useConfig: () => ({}),
    useConnection: () => ({ address: CONNECTED, isConnected: true }),
    usePublicClient: () => holder.publicClient,
  }
})

vi.mock('@/features/transaction-manager/hooks/useTransactionModal', () => ({
  useTransactionModal: () => ({
    isOpen: false,
    openModal: vi.fn(),
    closeModal: vi.fn(),
    clearTransaction: vi.fn(),
  }),
}))

import { ethRegistrarRenewSnippet } from '@ensdomains/ensjs-abi/v2/ethRegistrar'
import { SUPPORTED_TOKENS } from '@/lib/constants/tokens'
import { sepoliaWithEns } from '@/lib/wagmi'
import {
  type SelectedName,
  useRenewalTransactions,
} from '@/features/renew/hooks/useRenewalTransactions'
import { isExtendable2LD } from '@/features/renew/utils/nameExtension'

const wrapper = ({ children }: { children: ReactNode }) => {
  const client = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  })
  return <QueryClientProvider client={client}>{children}</QueryClientProvider>
}

const walletClient = {
  account: { address: CONNECTED },
  chain: sepoliaWithEns,
} as never

const ONE_YEAR = 31536000

beforeAll(() => {
  holder.publicClient = { chain: sepoliaWithEns }
})

/**
 * Runs the real single-name Extend flow for `name` and returns, side by side,
 * the strings the modal renders and the label the renew calldata carries.
 */
const runExtendFlow = async (selected: SelectedName) => {
  const { result } = renderHook(() => useRenewalTransactions(), { wrapper })

  act(() => {
    result.current.startFlow(selected, {
      duration: ONE_YEAR,
      tokenAddress: SUPPORTED_TOKENS.USDC,
      tokenPrice: 5_000_000n,
      // Allowance already covers the price, so the flow is a single renew step
      // and its `intent.prepare` is defined (see useRenewalTransactions.ts:466).
      tokenAllowance: 5_000_000n,
    })
  })

  const tx = result.current.transactions[0]!
  const intent = await tx.intent!.prepare!({
    walletClient,
    chainId: sepoliaWithEns.id,
  })
  const request = (intent as { request: { to: Address; data: `0x${string}` } })
    .request

  const decoded = decodeFunctionData({
    abi: ethRegistrarRenewSnippet,
    data: request.data,
  })

  return {
    displayedTitle: tx.title,
    displayedTransactionName: tx.transactionName,
    txId: tx.id,
    renewer: request.to,
    calldataLabel: decoded.args[0] as string,
    calldataDuration: decoded.args[1] as bigint,
  }
}

describe('Explorer Extend flow renews the normalized twin', () => {
  it('ATTACK: an un-normalized .eth name is displayed, its normalized twin is renewed', async () => {
    const displayed = 'Nike.eth' // mixed case: a legal, registered v1 .eth label
    const selected: SelectedName = {
      name: displayed,
      isV2: false,
      expiryDate: new Date(Date.now() + 30 * 24 * 3600 * 1000),
    }

    // The Extend button's own gate accepts it, unchanged.
    expect(isExtendable2LD(selected)).toBe(true)

    const out = await runExtendFlow(selected)

    console.log('[ATTACK] name shown in the names table :', displayed)
    console.log('[ATTACK] modal title                   :', out.displayedTitle)
    console.log('[ATTACK] modal transactionName         :', out.displayedTransactionName)
    console.log('[ATTACK] transaction id                :', out.txId)
    console.log('[ATTACK] renew() label in calldata     :', JSON.stringify(out.calldataLabel))
    console.log('[ATTACK] labelhash(displayed label)    :', labelhash('Nike'))
    console.log('[ATTACK] labelhash(calldata label)     :', labelhash(out.calldataLabel))
    console.log('[ATTACK] renewer contract              :', out.renewer)

    expect(out.displayedTitle).toBe('Extend Nike.eth')
    expect(out.displayedTransactionName).toBe('Extend Nike.eth')
    expect(out.calldataLabel).toBe('nike')
    expect(out.calldataLabel).not.toBe('Nike')
    expect(labelhash(out.calldataLabel)).not.toBe(labelhash('Nike'))
  })

  it('ATTACK: the same divergence for four more un-normalized label shapes', async () => {
    const cases: Array<[string, string, string]> = [
      ['U+00AD SOFT HYPHEN', 'ni­ke.eth', 'nike'],
      ['U+200B ZERO WIDTH SPACE', 'ni​ke.eth', 'nike'],
      ['U+FF2E FULLWIDTH LATIN N', 'Ｎike.eth', 'nike'],
      ['UPPERCASE', 'NIKE.eth', 'nike'],
    ]

    for (const [tag, displayed, expectedLabel] of cases) {
      const out = await runExtendFlow({
        name: displayed,
        isV2: true,
        expiryDate: new Date(Date.now() + 30 * 24 * 3600 * 1000),
      })
      console.log(
        `[ATTACK] ${tag.padEnd(24)} displayed ${JSON.stringify(displayed)} -> renew(${JSON.stringify(out.calldataLabel)})`,
      )
      expect(out.displayedTransactionName).toBe(`Extend ${displayed}`)
      expect(out.calldataLabel).toBe(expectedLabel)
      expect(out.calldataLabel).not.toBe(displayed.split('.')[0])
    }
  })

  it('CONTROL: identical flow, an already-normalized name -> shown and renewed agree', async () => {
    const displayed = 'nike.eth'
    const out = await runExtendFlow({
      name: displayed,
      isV2: false,
      expiryDate: new Date(Date.now() + 30 * 24 * 3600 * 1000),
    })

    console.log('[CONTROL] modal transactionName        :', out.displayedTransactionName)
    console.log('[CONTROL] renew() label in calldata    :', JSON.stringify(out.calldataLabel))
    console.log('[CONTROL] labelhash(displayed label)   :', labelhash('nike'))
    console.log('[CONTROL] labelhash(calldata label)    :', labelhash(out.calldataLabel))

    expect(out.displayedTransactionName).toBe('Extend nike.eth')
    expect(out.calldataLabel).toBe('nike')
    expect(labelhash(out.calldataLabel)).toBe(labelhash('nike'))
  })
})
```

Run:

```bash
pnpm --filter portal exec vitest run src/norm_renew_label_divergence.test.tsx --reporter=verbose --silent=false --testTimeout=15000
```

Captured output:

```
stdout | src/norm_renew_label_divergence.test.tsx > Explorer Extend flow renews the normalized twin > ATTACK: an un-normalized .eth name is displayed, its normalized twin is renewed
[ATTACK] name shown in the names table : Nike.eth
[ATTACK] modal title                   : Extend Nike.eth
[ATTACK] modal transactionName         : Extend Nike.eth
[ATTACK] transaction id                : renewal-renew-Nike.eth
[ATTACK] renew() label in calldata     : "nike"
[ATTACK] labelhash(displayed label)    : 0x6c21d8fef3525fe22e6b86a7719c6f56f3eddc10afd18248b4abb9c575eac438
[ATTACK] labelhash(calldata label)     : 0x80bccd5038fff69315e5b733c1354a3bc76ee64cfe3b0a25fbbbaf135dae5a6c
[ATTACK] renewer contract              : 0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa

stdout | src/norm_renew_label_divergence.test.tsx > Explorer Extend flow renews the normalized twin > ATTACK: the same divergence for four more un-normalized label shapes
[ATTACK] U+00AD SOFT HYPHEN       displayed "ni­ke.eth" -> renew("nike")

stdout | src/norm_renew_label_divergence.test.tsx > Explorer Extend flow renews the normalized twin > ATTACK: the same divergence for four more un-normalized label shapes
[ATTACK] U+200B ZERO WIDTH SPACE  displayed "ni​ke.eth" -> renew("nike")

stdout | src/norm_renew_label_divergence.test.tsx > Explorer Extend flow renews the normalized twin > ATTACK: the same divergence for four more un-normalized label shapes
[ATTACK] U+FF2E FULLWIDTH LATIN N displayed "Ｎike.eth" -> renew("nike")

stdout | src/norm_renew_label_divergence.test.tsx > Explorer Extend flow renews the normalized twin > ATTACK: the same divergence for four more un-normalized label shapes
[ATTACK] UPPERCASE                displayed "NIKE.eth" -> renew("nike")

stdout | src/norm_renew_label_divergence.test.tsx > Explorer Extend flow renews the normalized twin > CONTROL: identical flow, an already-normalized name -> shown and renewed agree
[CONTROL] modal transactionName        : Extend nike.eth
[CONTROL] renew() label in calldata    : "nike"
[CONTROL] labelhash(displayed label)   : 0x80bccd5038fff69315e5b733c1354a3bc76ee64cfe3b0a25fbbbaf135dae5a6c
[CONTROL] labelhash(calldata label)    : 0x80bccd5038fff69315e5b733c1354a3bc76ee64cfe3b0a25fbbbaf135dae5a6c

 ✓ src/norm_renew_label_divergence.test.tsx > Explorer Extend flow renews the normalized twin > ATTACK: an un-normalized .eth name is displayed, its normalized twin is renewed 478ms
 ✓ src/norm_renew_label_divergence.test.tsx > Explorer Extend flow renews the normalized twin > ATTACK: the same divergence for four more un-normalized label shapes 14ms
 ✓ src/norm_renew_label_divergence.test.tsx > Explorer Extend flow renews the normalized twin > CONTROL: identical flow, an already-normalized name -> shown and renewed agree 4ms

 Test Files  1 passed (1)
      Tests  3 passed (3)
   Start at  21:32:09
   Duration  22.84s (transform 2.01s, setup 1.23s, import 18.13s, tests 501ms, environment 1.55s)
```

The four escaped code points in the second block are literal `U+00AD`, `U+200B` and `U+FF2E` characters in the terminal; they are written escaped here so they survive copy-paste.

### Part 2 - the chain layer

Submits the exact calldata the modal built, against a local Anvil Sepolia fork (`http://127.0.0.1:8546`, chain id 11155111), and measures every balance and expiry it moves. No mock is present: the renewer, the registrar and the USDC contract are the ones the app is configured against on Sepolia.

Save as `onchain_renew_divergence.sh`:

```bash
#!/usr/bin/env bash
# Executes, against a local Anvil Sepolia fork, the exact calldata the Explorer's
# Extend modal builds for a user who selected "Nike.eth", and measures who pays
# and whose registration gains duration.
set -u
RPC=http://127.0.0.1:8546
REG=0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85          # v1 BaseRegistrar
RENEWER=0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa      # ensEthRenewerV1 (Sepolia)
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
VICTIM=0xBeeF00000000000000000000000000000000f1F1       # owns Nike.eth
ZERO=0x0000000000000000000000000000000000000000000000000000000000000000
YEAR=31536000
L_LOWER=$(cast to-uint256 "$(cast keccak 'nike')")
L_MIXED=$(cast to-uint256 "$(cast keccak 'Nike')")

usdc()    { cast call "$USDC" 'balanceOf(address)(uint256)' "$1" --rpc-url $RPC; }
expires() { cast call "$REG"  'nameExpires(uint256)(uint256)' "$1" --rpc-url $RPC; }
owner()   { cast call "$REG"  'ownerOf(uint256)(address)' "$1" --rpc-url $RPC; }

# Give the victim gas and 1000 USDC (mapping slot 0, located by write-and-read).
cast rpc anvil_setBalance "$VICTIM" 0xDE0B6B3A7640000 --rpc-url $RPC >/dev/null
cast rpc anvil_setStorageAt "$USDC" "$(cast index address "$VICTIM" 0)" \
  0x000000000000000000000000000000000000000000000000000000003b9aca00 --rpc-url $RPC >/dev/null

echo "== two registrations, two owners =="
echo "labelhash nike        : $(cast keccak 'nike')"
echo "labelhash Nike        : $(cast keccak 'Nike')"
echo "ownerOf(nike)         : $(owner "$L_LOWER")"
echo "ownerOf(Nike)         : $(owner "$L_MIXED")"

echo
echo "== what the Explorer asks the chain about, for a user who selected Nike.eth =="
echo "isRenewable('nike')   : $(cast call "$RENEWER" 'isRenewable(string)(bool)' 'nike' --rpc-url $RPC)"
echo "isRenewable('Nike')   : $(cast call "$RENEWER" 'isRenewable(string)(bool)' 'Nike' --rpc-url $RPC)"
echo -n "getRenewPrice('nike') : "; cast call "$RENEWER" 'getRenewPrice(string,uint64,address)(uint256)' 'nike' $YEAR "$USDC" --rpc-url $RPC
echo -n "getRenewPrice('Nike') : "; cast call "$RENEWER" 'getRenewPrice(string,uint64,address)(uint256)' 'Nike' $YEAR "$USDC" --rpc-url $RPC 2>&1 | sed 's/, data:.*//'

echo
echo "== BEFORE =="
echo "victim USDC           : $(usdc "$VICTIM")"
echo "nameExpires(nike)     : $(expires "$L_LOWER")"
echo "nameExpires(Nike)     : $(expires "$L_MIXED")"
B_USDC=$(usdc "$VICTIM" | awk '{print $1}')
B_LOW=$(expires "$L_LOWER" | awk '{print $1}')
B_MIX=$(expires "$L_MIXED" | awk '{print $1}')

echo
echo "== the two transactions the modal submits =="
echo -n "approve(renewer, 320000018)  status: "
cast send "$USDC" 'approve(address,uint256)' "$RENEWER" 320000018 \
  --from "$VICTIM" --unlocked --rpc-url $RPC 2>&1 | awk '/^status/{print $2, $3}'
echo -n "renew('nike',31536000,USDC,0x0) status: "
cast send "$RENEWER" 'renew(string,uint64,address,bytes32)' 'nike' $YEAR "$USDC" $ZERO \
  --from "$VICTIM" --unlocked --rpc-url $RPC 2>&1 | awk '/^status/{print $2, $3}'

echo
echo "== AFTER =="
A_USDC=$(usdc "$VICTIM" | awk '{print $1}')
A_LOW=$(expires "$L_LOWER" | awk '{print $1}')
A_MIX=$(expires "$L_MIXED" | awk '{print $1}')
echo "victim USDC           : $A_USDC   (delta $((A_USDC - B_USDC)))"
echo "nameExpires(nike)     : $A_LOW  (delta $((A_LOW - B_LOW))  <- third party 0x7F12..)"
echo "nameExpires(Nike)     : $A_MIX  (delta $((A_MIX - B_MIX))  <- the victim's own name)"
echo "residual allowance    : $(cast call "$USDC" 'allowance(address,address)(uint256)' "$VICTIM" "$RENEWER" --rpc-url $RPC)"

echo
echo "== CONTROL: identical call, label the user actually selected =="
echo -n "renew('Nike',31536000,USDC,0x0) : "
cast call "$RENEWER" 'renew(string,uint64,address,bytes32)' 'Nike' $YEAR "$USDC" $ZERO \
  --from "$VICTIM" --rpc-url $RPC 2>&1 | sed 's/, data:.*//'
echo -n "selector 0x1caefaa0 is         : "; cast 4byte 0x1caefaa0
```

Captured output:

```
== two registrations, two owners ==
labelhash nike        : 0x80bccd5038fff69315e5b733c1354a3bc76ee64cfe3b0a25fbbbaf135dae5a6c
labelhash Nike        : 0x6c21d8fef3525fe22e6b86a7719c6f56f3eddc10afd18248b4abb9c575eac438
ownerOf(nike)         : 0x7F1266aa4E48567f42E02E9dA040eeec33EDd5Cb
ownerOf(Nike)         : 0xbeEF00000000000000000000000000000000f1F1

== what the Explorer asks the chain about, for a user who selected Nike.eth ==
isRenewable('nike')   : true
isRenewable('Nike')   : false
getRenewPrice('nike') : 160000009 [1.6e8]
getRenewPrice('Nike') : Error: server returned an error response: error code 3: execution reverted: custom error 0x1caefaa0:  Nike

== BEFORE ==
victim USDC           : 1000000000 [1e9]
nameExpires(nike)     : 2262550824 [2.262e9]
nameExpires(Nike)     : 1820602402 [1.82e9]

== the two transactions the modal submits ==
approve(renewer, 320000018)  status: 1 (success)
renew('nike',31536000,USDC,0x0) status: 1 (success)

== AFTER ==
victim USDC           : 839999991   (delta -160000009)
nameExpires(nike)     : 2294086824  (delta 31536000  <- third party 0x7F12..)
nameExpires(Nike)     : 1820602402  (delta 0  <- the victim's own name)
residual allowance    : 160000009 [1.6e8]

== CONTROL: identical call, label the user actually selected ==
renew('Nike',31536000,USDC,0x0) : Error: server returned an error response: error code 3: execution reverted: custom error 0x1caefaa0:  Nike
selector 0x1caefaa0 is         : NameNotRenewable(string)
```

`nameExpires(nike)` reads `2262550824` here rather than the registration's original `2231014824` because an earlier run of the same script already renewed it once. The delta is what the claim rests on, and it is `31536000` in both runs.

## Control

Two controls, each changing one load-bearing condition.

**Application-layer control.** Same hook, same flow, same builder; the only variable is whether the selected name's first label is already ENSIP-15-normalized.

* ATTACK, `Nike.eth`: modal renders `Extend Nike.eth`; calldata carries `renew("nike", ...)`. `labelhash("Nike") = 0x6c21d8fe...` against `labelhash("nike") = 0x80bccd50...`.
* CONTROL, `nike.eth`: modal renders `Extend nike.eth`; calldata carries `renew("nike", ...)`. `labelhash` is `0x80bccd50...` on both sides.

This attributes the divergence to the `getLabel` substitution at `:197` and rules out the encoder, the renewer selection and the modal.

**Chain-layer control.** Same call, same signer, same duration, same token, same referrer; the only variable is which label string the call carries.

* ATTACK, `renew("nike", ...)`: `status 1 (success)`, payer `-160000009` USDC, third party's registration `+31536000` seconds.
* CONTROL, `renew("Nike", ...)`: reverts `NameNotRenewable("Nike")`. No transfer, no expiry change.

The control arm executed a static call and reverted at the renewer's eligibility check, so no approval was consumed and no state changed. It shows that the same action addressed to the name the user selected is refused by the renewer, and that the flow reaches success only because the label was substituted first.

## Impact

The row's two predicates are both measured.

**The application pre-fills an incorrect transaction detail.** The confirmation screen names `Nike.eth` and quotes `160.000009 USDC`. The calldata names `"nike"`. The quote itself belongs to `nike.eth`: `getRenewPrice('Nike')` reverts `NameNotRenewable("Nike")`, so no price for the displayed name exists, and the number on screen can only be the other registration's.

**The user then submits a transaction against an unintended recipient and loses funds.** Measured across the two submitted transactions: the payer's USDC falls by `160000009`; `nike.eth`, owned by `0x7F1266aa4E48567f42E02E9dA040eeec33EDd5Cb`, gains exactly `31536000` seconds; `Nike.eth`, owned by the payer, gains `0`. A `160000009` allowance remains granted to the renewer after the flow, and `useRenewalTransactions.ts` contains no revoke step.

The loss is not recoverable by the user. `renew` takes no owner argument and pulls rent from `_msgSender()`, documented at `packages/transaction-manager/src/machines/registration/registration.actors.ts:1006-1009`, and there is no refund path. The user's own name keeps the expiry it had and, once past grace, is re-registrable by anyone, while the Explorer reports it as renewed because `handleDone` derives `renewedNames` from the un-normalized string at `:368-373`.

**Recipient and attacker model.** No malicious interaction with the payer is needed. The executed proof uses an independently owned normalized twin, not an attacker-controlled fixture. Stated precisely, because the three parties receive three different things: the payer sends `160000009` USDC, which the renewer contract collects as rent; the `nike.eth` registration, which the payer does not own, gains `31536000` seconds of registration term; and the third-party owner of that registration pays nothing and takes no action. The valuable entitlement lands on a registration the payer has no claim to, which is already the listed wrong-transaction consequence. If a malicious party owns the normalized twin, the same production path directs future Extend payments to that party, but attacker ownership is not required for the measured impact and is not asserted as part of the fork proof.

**Affected set.** Every unmigrated v1 `.eth` 2LD whose first label is not ENSIP-15-normalized, since those rows reach the table verbatim from the ENS subgraph. The PoC measured 5 of 5 un-normalized label shapes diverging in the builder (mixed case, all caps, U+00AD, U+200B, U+FF2E) and 1 of 1 normalized name agreeing. The same builder would divergently renew a v2 row too; today the v2 list is filtered upstream, so fixing only the indexer filter would leave `:197` still substituting.

**Strongest counterargument, and the evidence against it.** The strongest rejection is that normalizing before hashing is deliberate, since the comment at `:191` says so. The evidence that defeats it is the on-chain reads: `isRenewable('nike') = true` while `isRenewable('Nike') = false`, and `getRenewPrice('Nike')` reverting `NameNotRenewable("Nike")`. The renewer distinguishes the two strings, so the substitution is not a canonicalisation of one name but a selection of a second one, and the sibling path in the same repository already implements the correct separation at `apps/manager/src/features/dashboard/components/NamesTable.tsx:84-88`, keeping `displayName` and `name` as separate fields.

## Suggested fix

The invariant to restore: the registration a flow reads, prices and writes must be the one identified by the exact label string that flow displays. Because `.eth` labels are not normalized on-chain, that string is the label the indexer returned, and normalizing it before hashing selects a different registration.

1. In `buildRenewIntent` (`useRenewalTransactions.ts:190-203`), pass the selected name through unchanged rather than through `getLabel`. `renewNameWriteParameters` splitting without normalizing is then the correct behaviour, not a hazard.
2. Apply the same to both reads, so the gate, the quote and the write all address one registration: `getIsRenewable` (`useIsRenewable.ts:38-46`) and `resolvePriceInputs` (`useRegistrationPrice.ts:46-58`).
3. Where normalization is genuinely wanted - registration of a new name, where the user is choosing the string - keep it, but render the normalized form to the user before they sign, so the string shown and the string hashed are the same string.
4. Carry the two strings separately through the flow, as `apps/manager/src/features/dashboard/components/NamesTable.tsx:83-88` already does with `displayName` and `name`, and refuse to build any intent whose label differs from the first label of the name in `transactionName`. That turns this class of divergence into a build-time failure instead of a silent substitution.
