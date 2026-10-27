# 89460 w a critical every ens registration overwrites the previous one s transaction history

**Submitted on Aug 19th 2026 at 13:23:27 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89460
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>
* **Impacts:**
  * Temporarily disabling user to access target site, such as:
    * Locking up the victim from login
    * Cookie bombing, etc.

## Description

## Summary

The registration machine labels its four on-chain steps with module level constants. Every registration, of every name, by every user, reuses `tx-reg-deploy-resolver`, `tx-reg-commit`, `tx-reg-approve` and `tx-reg-register`. The history store is keyed on exactly that id, and archiving is a write by key rather than an append, so a user's history can hold at most one registration. Each new registration silently replaces the last.

The same id is also the idempotency key the api worker upserts on, where the schema states in its own comment that the id is unique per user. It is not.

## Root cause

### 1. The ids are module constants, not per run values

`packages/transaction-manager/src/machines/registration/registration.machine.ts:77`

```tsx
export const REGISTRATION_TX_IDS = {
  deployResolver: 'tx-reg-deploy-resolver',
  commit: 'tx-reg-commit',
  approve: 'tx-reg-approve',
  register: 'tx-reg-register',
} as const
```

Each is passed straight through as `options.id`, for example `registration.machine.ts:1401` on the register step, and `transactionManager.startTransaction` takes it verbatim at `providers/transactionManager.ts:225`.

### 2. The history store is keyed by that id, and archiving writes by key

`packages/transaction-manager/src/helpers/transaction-persistence.ts:111`

```tsx
const historyStore = db.createObjectStore(HISTORY_STORE, {
  keyPath: 'id',
})
```

`transaction-persistence.ts:300` and `:334`

```tsx
await this.db.put(HISTORY_STORE, data)             // IndexedDB, put by keyPath 'id'
memoryStorage.history.set(transaction.id, data)    // in-memory fallback
```

`providers/transactionManager.ts:312` archives the persisted record whose `id` is that fixed txId, on every terminal transaction. Nothing in the record or the key identifies which registration, or which name, it belongs to.

### 3. The backend key carries the same collision

`workers/api-worker/src/core/database/schema/transactions.ts:45` and `:67`

```tsx
/** Client-generated transaction id (idempotency key, unique per user) */
tx_id: text('tx_id').notNull(),
...
primaryKey({ columns: [table.user_id, table.tx_id] }),
```

`app/routes/transactions/index.ts` upserts `onConflictDoUpdate` on that pair. The uniqueness the schema documents is the invariant `REGISTRATION_TX_IDS` breaks.

## Impact

A user who registers a second name loses the first from their transaction history. The four records are replaced, not appended, so `getArchivedTransactions`, `getTransactionHistory` and `exportAllData` return only the most recent registration. There is no recovery path because the previous values are overwritten in place rather than pruned by an age or size policy, and no error surfaces at any layer.

This is the ordinary path, not an edge case. It needs no failure, no retry and no unusual input. Two successful registrations are enough, and both apps reach it: the portal clears the transaction manager at the start of every registration at `apps/portal/src/features/register/hooks/useRegistrationTransactions.ts:163`, and the manager does the same on wallet connect and disconnect at `apps/manager/src/lib/wallet/WalletLifecycle.tsx:41`.

## Recommendation

Mint a run identifier once per `START_REGISTRATION` and scope the step ids to it, keeping the step suffix stable so the transaction modal can still track each step by a predictable key:

```tsx
const runId = crypto.randomUUID()
const ids = {
  deployResolver: `tx-reg-${runId}-deploy-resolver`,
  commit: `tx-reg-${runId}-commit`,
  approve: `tx-reg-${runId}-approve`,
  register: `tx-reg-${runId}-register`,
}
```

If the fixed ids are wanted for in-flight UI keying, keep them for the active store and give the archived record its own unique id before it reaches `archiveTransaction` and `buildArchivedTransaction`, so the history key is distinct even when the in-flight key is not.

An invariant test asserting that two consecutive registrations produce eight distinct archived ids would keep this from being re-lost.

## Known issues

Not listed. R3-07 is a different defect with a different mechanism and a different fix. Its mechanism is the `completedTelemetry` set at `providers/transactionManager.ts:146`, which is never cleared per transaction and therefore suppresses a second archive under the same id within one page session. Both shipped apps clear that set before a new registration, the portal at `useRegistrationTransactions.ts:163` and the manager at `WalletLifecycle.tsx:41`, so both archives run and the second overwrites the first, which is the state the PoC reproduces. R3-07's fix, clearing the completion set, makes both archives run everywhere; it does not change the key they are written under, so it widens this defect rather than closing it. The fix here is unique ids, which touches neither the completion set nor the archive gate.

R3-05 is a key scan in the localStorage fallback whose `tx-` prefix also matches the `tx-history` key, so clearing active transactions destroys the archive. That is a prefix match in one fallback backend, triggered by a clear operation. This is a key collision in the primary IndexedDB store and in the in-memory fallback, triggered by a successful registration with nothing cleared, and the localStorage backend is the one backend where it does not apply because that path appends to an array.

R3-06 concerns a missing field on the archived record. This concerns the record's key. The two are independent: assigning the chain identifier leaves the four keys unchanged, and assigning unique keys leaves the chain identifier unchanged.

R3-08 is a stale error string attached to a record that succeeded, again a field on a record that is otherwise correctly stored. R3-01 through R3-04 are liveness defects in the telemetry trimmer, the actor await helpers, the status poller and the commitment retry target; none of them touch persistence keys, and the PoC's two registrations both complete without entering a retry or an error state.

## Proof of Concept

Local only. No fork, no localnet, no network and no chain. The transports and contract reads are stubbed. The registration machine, the transaction machine, the transaction manager singleton and the persistence layer are the real ones. The file is reproduced in full below.

{% stepper %}
{% step %}

## Install and run

```bash
pnpm install
pnpm --filter @ens-apps/transaction-manager test src/__audit__
```

{% endstep %}

{% step %}

## Create `packages/transaction-manager/src/__audit__/registration-history-overwrite.poc.test.ts`

```tsx
/**
 * PoC: every ENS registration reuses the same four transaction ids, and the
 * history store is keyed by that id, so each registration overwrites the last.
 *
 * Local only. No fork, no network, no chain. The transports and contract reads
 * are stubbed; the registration machine, the transaction machine, the
 * transaction manager singleton and the persistence layer are the real ones.
 *
 *   pnpm --filter @ens-apps/transaction-manager test src/__audit__
 */
import { ethRegistrarRegisterSnippet } from '@ensdomains/ensjs-abi/v2/ethRegistrar'
import type { Address, Hex, PublicClient } from 'viem'
import {
  decodeFunctionData,
  encodeAbiParameters,
  encodeEventTopics,
  parseAbi,
} from 'viem'
import { sepolia } from 'viem/chains'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { createActor, waitFor } from 'xstate'

const sendTransaction = vi.fn(async () => `0x${'ab'.repeat(32)}` as Hex)
const readContract = vi.fn()
const multicall = vi.fn()
vi.mock('viem/actions', async (io) => ({
  ...(await io<typeof import('viem/actions')>()),
  sendTransaction: (...a: unknown[]) => sendTransaction(...a),
  readContract: (...a: unknown[]) => readContract(...a),
  multicall: (...a: unknown[]) => multicall(...a),
}))
vi.mock('../contracts/paymentToken', () => ({
  assertPaymentTokenSupported: vi.fn().mockResolvedValue(undefined),
}))

import { ENS_SEPOLIA_CONTRACTS } from '../contracts/ens-sepolia'
import {
  clearTransactionHistory,
  getArchivedTransactions,
} from '../helpers/transaction-persistence'
import {
  REGISTRATION_TX_IDS,
  registrationMachine,
} from '../machines/registration/registration.machine'
import {
  type ArchivedTransaction,
  transactionManager,
} from '../providers/transactionManager'
import type { Signer } from '../types/signer.types'

const WALLET = '0x1111111111111111111111111111111111111111' as Address
const RESOLVER = '0x5555555555555555555555555555555555555555' as Address

/** A genuine VerifiableFactory ProxyDeployed log for the resolver deployment. */
const proxyLog = {
  address: ENS_SEPOLIA_CONTRACTS.VerifiableFactory,
  topics: encodeEventTopics({
    abi: parseAbi([
      'event ProxyDeployed(address indexed sender, address indexed proxyAddress, uint256 salt, address implementation)',
    ]),
    eventName: 'ProxyDeployed',
    args: { sender: WALLET, proxyAddress: RESOLVER },
  }),
  data: encodeAbiParameters(
    [{ type: 'uint256' }, { type: 'address' }],
    [1n, ENS_SEPOLIA_CONTRACTS.PermissionedResolverImpl],
  ),
}

const publicClient = {
  chain: sepolia,
  waitForTransactionReceipt: vi.fn(async ({ hash }: { hash: Hex }) => ({
    status: 'success',
    transactionHash: hash,
    logs: [proxyLog],
  })),
} as unknown as PublicClient

const signer = {
  type: 'eoa',
  walletClient: { account: { address: WALLET }, chain: sepolia },
} as unknown as Signer

/** Run one complete EOA registration through the real machine and manager. */
async function register(name: string) {
  const actor = createActor(registrationMachine, {
    input: { chainId: sepolia.id },
  })
  actor.start()
  actor.send({
    type: 'START_REGISTRATION',
    name,
    duration: 31_536_000n,
    token: 'USDC',
    price: 8_000_000n,
    signer,
    accountAddress: WALLET,
    publicClient,
  })
  await waitFor(actor, (s) => s.matches('success') || s.matches('error'), {
    timeout: 8000,
  })
  expect(actor.getSnapshot().value).toBe('success')
}

/** The label inside the surviving `register` record in the history store. */
async function registeredLabelInHistory() {
  const history = await getArchivedTransactions()
  const record = history.find((h) => h.id === REGISTRATION_TX_IDS.register)
  const request = (record?.context as { request?: { data: Hex } })?.request
  return decodeFunctionData({
    abi: ethRegistrarRegisterSnippet,
    data: request!.data,
  }).args[0]
}

beforeEach(async () => {
  transactionManager.clear()
  await clearTransactionHistory()
  readContract.mockReset()
  multicall.mockReset()
  readContract.mockImplementation(async (_c: unknown, args: any) => {
    if (args.functionName === 'makeCommitment') return `0x${'cc'.repeat(32)}`
    if (args.functionName === 'MIN_COMMITMENT_AGE') return 0n
    throw new Error(`unexpected read ${args.functionName}`)
  })
  multicall.mockResolvedValue([0n, [8_000_000n, 0n]])
})

describe('registration transaction ids are shared by every registration', () => {
  it('the four ids are module constants, not per-registration values', () => {
    expect(Object.values(REGISTRATION_TX_IDS)).toEqual([
      'tx-reg-deploy-resolver',
      'tx-reg-commit',
      'tx-reg-approve',
      'tx-reg-register',
    ])
  })

  it('the second registration destroys the first in the history store', async () => {
    const archived: ArchivedTransaction[] = []
    const off = transactionManager.onTransactionArchived((t) => archived.push(t))

    await register('alpha.eth')
    // The portal calls this at the start of every registration:
    // apps/portal/src/features/register/hooks/useRegistrationTransactions.ts:163
    transactionManager.clear()
    await register('beta.eth')
    off()

    // Eight transactions completed and all eight were archived.
    expect(archived).toHaveLength(8)
    // They carry only four distinct ids.
    expect(new Set(archived.map((a) => a.txId)).size).toBe(4)

    // The history store is keyed by that id, so four records exist.
    const history = await getArchivedTransactions()
    expect(history).toHaveLength(4)
    expect(history.map((h) => h.id).sort()).toEqual(
      Object.values(REGISTRATION_TX_IDS).sort(),
    )

    // The surviving register record is beta's. alpha.eth is gone.
    expect(await registeredLabelInHistory()).toBe('beta')
  })

  it('the backend primary key (user_id, tx_id) collapses them identically', async () => {
    const archived: ArchivedTransaction[] = []
    const off = transactionManager.onTransactionArchived((t) => archived.push(t))
    await register('alpha.eth')
    transactionManager.clear()
    await register('beta.eth')
    off()

    // workers/api-worker/src/core/database/schema/transactions.ts:67
    //   primaryKey({ columns: [user_id, tx_id] })  +  onConflictDoUpdate
    const rows = new Map<string, ArchivedTransaction>()
    for (const a of archived) rows.set(`user-uuid|${a.txId}`, a)

    expect(rows.size).toBe(4)
  })
})
```

{% endstep %}

{% step %}

## Output

```
 ✓ the four ids are module constants, not per-registration values 2ms
 ✓ the second registration destroys the first in the history store 10ms
 ✓ the backend primary key (user_id, tx_id) collapses them identically 5ms

 Test Files  1 passed (1)
      Tests  3 passed (3)
```

{% endstep %}
{% endstepper %}

The second test is the finding. It drives two complete EOA registrations, `alpha.eth` then `beta.eth`, recording every archived transaction through the public `onTransactionArchived` seam and then reading the persistence layer back. Both registrations reach the machine's `success` state, all eight transactions reach a terminal state and all eight are archived, yet they carry only four distinct ids, the history store holds four records, and the surviving `register` record decodes to the label `beta`. The assertion that proves the loss reads the calldata out of the stored record and decodes it with the real `ethRegistrarRegisterSnippet`.

The third test replays the eight archived records through the api worker's composite primary key, a `Map` keyed on `(user_id, tx_id)` exactly as `primaryKey({ columns: [user_id, tx_id] })` with `onConflictDoUpdate` behaves, and gets four rows from eight transactions.

The suite is additive. The package's existing 118 tests still pass alongside it, 121 total.
