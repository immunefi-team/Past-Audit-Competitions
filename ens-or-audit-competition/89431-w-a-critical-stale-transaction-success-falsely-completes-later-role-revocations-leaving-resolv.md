# 89431 w a critical stale transaction success falsely completes later role revocations leaving resolver authority active

**Submitted on Aug 19th 2026 at 09:51:09 UTC by @Avicoh for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89431
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>
* **Impacts:**
  * Direct theft of user funds

## Description

## Brief / Introduction

A completed transaction actor remains in the transaction manager's active global map, while every Explorer role revocation reuses the same application-wide ID, `tx-revoke-roles`. After one role revocation succeeds, closing its modal with X or navigating away leaves that successful actor active. A later "Remove user" flow for a different ENS name/account inherits the earlier actor's success state and transaction hash, immediately renders the new removal as **Done**, and invokes only the new descriptor's cleanup callback. Its `onStart` callback—and therefore `revokeRoles(...)`—never runs. A compromised delegate can consequently retain `ROLE_SET_RESOLVER` after the owner is falsely told removal completed, redirect the victim name to an attacker-controlled resolver/address, and steal funds subsequently sent by name.

Affected frozen revision: `cda79acaad59711b943fc68207ebb3f1d0ff8596`.

Primary asset: `packages/transaction-manager`. The exploitable consumer is the in-scope Explorer Portal in `apps/portal`.

## Vulnerability details

### 1. Successful actors remain in the active transaction map

`packages/transaction-manager/src/providers/transactionManager.ts:290-340` archives terminal actors and emits telemetry, but never removes them from `this.transactions`. The active entry is removed only by an explicit cancel/clear path (`:348-377`, `:477-487`).

### 2. Ordinary successful-modal exits do not clear the actor

`apps/portal/src/features/transaction-manager/components/TransactionModal.tsx:54-65` clears only when `txState.error` exists; the source comment explicitly preserves pending and successful actors. `useTransactionModal.ts:27-42` closes only the modal atom on route resolution and does not call `transactionManager.clear()`.

Therefore, a user can legitimately leave a terminal actor active by closing a successful modal with X or navigating elsewhere instead of clicking Done.

### 3. Unrelated role removals reuse one global ID

`apps/portal/src/features/roles/utils/buildRoleTransactionDescriptors.ts:12-13,69-94` assigns `tx-revoke-roles` to every role revocation, without binding the ID to the ENS name/resource, account, role bitmap, registry, chain, or operation instance.

### 4. The modal combines the old actor with the new descriptor

`useActiveTransactionState.ts:41-67` can return a terminal actor. `TransactionsOverviewContent.tsx:43-53,59-68,112-139,190-208` retrieves status/hash by descriptor ID and directly renders Done for a successful actor. `TransactionStateContent.tsx:46-52,70-124,170-187` renders the *new* descriptor's name text but the *old* actor's success/hash.

The resulting screen describes the new target while linking to the earlier transaction.

### 5. The new revocation is never constructed or submitted

`buildRoleTransactions.ts:61-76,78-109` binds the new descriptor's `onStart` to `revokeRoles(...)`; its final `onDone` is cleanup. `RolesSidebar.tsx:104-140,159-178` opens the new removal and clears transactions from `handleDone`.

Because the retained actor makes the new descriptor terminal before `onStart`, clicking Done invokes cleanup and erases the stale state without ever calling `revokeRoles`. There is no new calldata, wallet request, transaction hash, or on-chain revocation.

The supplied integrated test uses the real `transactionManager`, `TransactionManagerProvider`, `TransactionModal`, and production role transaction builder. It proves the exact sequence `pending -> success -> retained actor -> new victim.eth descriptor -> Done`, confirms that the new target appears beside the old hash, and asserts that the new `revokeRoles` callback is never called.

## Impact details

The selected in-scope impact is **Direct theft of user funds**.

Concrete exploitation sequence:

{% stepper %}
{% step %}

## A compromised or deceptive delegate holds `ROLE_SET_RESOLVER` on `victim.eth`.

{% endstep %}

{% step %}

## The owner previously completed an unrelated role revocation and exited its success modal with X or navigation rather than Done.

The actor remains active under `tx-revoke-roles`.
{% endstep %}

{% step %}

## The owner opens `victim.eth`, selects the delegate, confirms Remove user, and immediately sees the operation as Done with the earlier hash.

No wallet prompt appears.
{% endstep %}

{% step %}

## Clicking Done clears the UI state while the delegate remains authorized on-chain.

{% endstep %}

{% step %}

## The delegate calls `setResolver` and points `victim.eth` to an attacker-controlled resolver/address.

{% endstep %}

{% step %}

## Funds sent to the victim by resolving `victim.eth` are delivered to the attacker.

{% endstep %}
{% endstepper %}

Focused exact-contract tests pass 6/6 and confirm that an authorized role holder can call `setResolver` and that the owner could have revoked the role. The vulnerable omission is entirely in the in-scope transaction-manager/Portal path.

The direct-funds consequence depends on an already-authorized compromised delegate and on the owner relying on the false success screen. Those are explicit feasibility conditions, not hidden assumptions. If the program applies a feasibility downgrade, High is a reasonable fallback; the primitive itself deterministically suppresses the intended security revocation.

## Why this is not published known issue R3-07

R3-07 concerns the separate `completedTelemetry` set being keyed forever by a reused ID. In that issue, after a failed attempt, a **new transaction is submitted and succeeds**, but its terminal event is not archived/reported and it appears pending. Its stated consequence is history/telemetry integrity and it is rated Medium.

This report does not rely on `completedTelemetry`. A previous terminal actor remains in the active `transactions` map and is consumed by a later descriptor **before the later `onStart` executes**. The later transaction never exists and no calldata is submitted. The consequence is suppressed access revocation and retained resolver-redirection authority.

The program terms expressly keep new consequences of a listed root cause in scope when they materially change severity. Preventing a security revocation and preserving authority capable of redirecting funds is materially different from missing history/telemetry for a transaction that actually executed.

## Distinction from the resolver multi-resource removal finding

The previously submitted multi-resource removal finding constructs and submits a real transaction, but that transaction revokes only one name-scoped resource because Portal merges root-level and name-level role entries into one account row.

This finding does not depend on role aggregation or mixed resources. A single ordinary role on a single name is sufficient. A stale terminal transaction actor causes the later removal to inherit an earlier success state, so the later transaction is never constructed, approved, or submitted at all.

The fixes are independent. Correctly separating role resources does not evict terminal actors or make transaction IDs operation-instance-specific. Evicting stale actors or using unique transaction IDs does not fix multi-resource revocation construction.

No local-storage tampering, malformed ENS name, indexer behavior, wallet quirk, contract defect, or mixed root/name resource is required for this PoC.

## Suggested remediation

1. Evict terminal actors from the active map after archival; retain displayable results in a separate history store.
2. Clear terminal actors before opening every new transaction flow.
3. Bind IDs to chain, operation, target contract, name/resource, account, arguments, and a unique operation nonce.
4. Bind UI descriptors to their exact actor instance/request hash; never infer execution solely from a matching global string ID.
5. Add a regression covering Name A success followed by Name B removal, and require Name B to begin Not Started and invoke its own calldata builder.

## Verification summary

* Integrated Portal PoC: 2/2 passed.
* Transaction-manager retention PoC: 1/1 passed; complete package invocation: 119/119 passed.
* Portal typecheck: passed.
* Transaction-manager typecheck: passed.
* Supporting exact-contract tests: 6/6 passed.
* Frozen target tree remained unchanged; PoC files exist only in an isolated copy.
* PoC bundle SHA-256: `35d12f7b6ec3a187c4784eb9c1dcca26adb02dfd0039648f9cb4f0b180881040`.

## References

* Frozen target: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>
* Portal consumer: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>

## Proof of Concept

### Environment

Use the competition repository at frozen commit:

```
cda79acaad59711b943fc68207ebb3f1d0ff8596
```

The PoC does not modify production source. Add the following two test files to an isolated copy of the target.

### Test 1: integrated real-manager / real-modal collision

Create:

```
apps/portal/src/features/transaction-manager/components/TransactionStateContent.stale-id.poc.test.tsx
```

```tsx
import {
  TransactionManagerProvider,
  type TransactionMachineActor,
  transactionManager,
} from '@ens-apps/transaction-manager'
import { act, fireEvent, render, screen, waitFor } from '@testing-library/react'
import type {
  Address,
  Hash,
  PublicClient,
  TransactionReceipt,
  WalletClient,
} from 'viem'
import { afterEach, describe, expect, it, vi } from 'vitest'
import { Dialog, DialogContent } from '@/components/ui/dialog'
import { buildRoleTransactions } from '@/features/roles/utils/buildRoleTransactions'
import {
  closeTransactionModal,
  openTransactionModal,
} from '../hooks/useTransactionModal'
import { TransactionModal } from './TransactionModal'
import { TransactionStateContent } from './TransactionStateContent'

vi.mock('wagmi', () => ({ useChainId: () => 1 }))
vi.mock('@/lib/wagmi', () => ({ wagmiConfig: { chains: [] } }))
vi.mock('@/utils/blockExplorer/getBlockExplorerTxUrl', () => ({
  getBlockExplorerTxUrl: (_chains: unknown, _chainId: number, hash: Hash) =>
    `https://explorer.invalid/tx/${hash}`,
}))
vi.mock('./EstimatedGasCost', () => ({
  EstimatedGasCost: () => <span>0.001 ETH</span>,
}))

const ATTACKER = '0x2222222222222222222222222222222222222222' as Address
const REGISTRY = '0x3333333333333333333333333333333333333333' as Address
const OLD_HASH = `0x${'ab'.repeat(32)}` as Hash

afterEach(() => {
  closeTransactionModal()
  transactionManager.clear()
  vi.clearAllMocks()
})

describe('stale global transaction id PoC', () => {
  it('shows a new role removal as done and skips its revoke callback', () => {
    const grantRoles = vi.fn()
    const revokeRoles = vi.fn()
    const handleDone = vi.fn()

    const [newRemoval] = buildRoleTransactions(
      null,
      { account: ATTACKER, roles: ['ROLE_SET_RESOLVER'] },
      'victim.eth',
      { grantRoles, revokeRoles, handleDone },
      REGISTRY,
    )

    expect(newRemoval.id).toBe('tx-revoke-roles')

    const staleSuccessfulActor = {
      getSnapshot: () => ({
        value: 'success',
        context: { hash: OLD_HASH, error: undefined },
      }),
    } as unknown as TransactionMachineActor

    const activeTransactionsMap = new Map<string, TransactionMachineActor>([
      ['tx-revoke-roles', staleSuccessfulActor],
    ])

    render(
      <Dialog open>
        <DialogContent>
          <TransactionStateContent
            transactions={[newRemoval]}
            activeTransactionId="tx-revoke-roles"
            txState={{
              txId: 'tx-revoke-roles',
              machineState: 'success',
              hash: OLD_HASH,
              error: undefined,
            }}
            activeTransactionsMap={activeTransactionsMap}
            setTransactionModalContentState={vi.fn()}
          />
        </DialogContent>
      </Dialog>,
    )

    expect(screen.getByText(/victim\.eth/)).toBeInTheDocument()
    expect(screen.getByRole('link')).toHaveAttribute(
      'href',
      `https://explorer.invalid/tx/${OLD_HASH}`,
    )

    fireEvent.click(screen.getByRole('button', { name: 'Done' }))

    expect(handleDone).toHaveBeenCalledOnce()
    expect(revokeRoles).not.toHaveBeenCalled()
    expect(grantRoles).not.toHaveBeenCalled()
  })

  it('reproduces the collision through the real manager, provider, and modal', async () => {
    const revokeRoles = vi.fn()
    const handleDone = vi.fn()
    const [newRemoval] = buildRoleTransactions(
      null,
      { account: ATTACKER, roles: ['ROLE_SET_RESOLVER'] },
      'victim.eth',
      { grantRoles: vi.fn(), revokeRoles, handleDone },
      REGISTRY,
    )

    const publicClient = {
      waitForTransactionReceipt: vi.fn().mockResolvedValue({
        status: 'success',
        transactionHash: OLD_HASH,
      } as TransactionReceipt),
    } as unknown as PublicClient

    render(
      <TransactionManagerProvider publicClient={publicClient}>
        <TransactionModal transactions={[newRemoval]} />
      </TransactionManagerProvider>,
    )

    await act(async () => {
      transactionManager.startTransaction(
        {
          type: 'eoa',
          from: ATTACKER,
          to: REGISTRY,
          chainId: 1,
          value: 0n,
          data: '0x',
        },
        {
          type: 'eoa',
          walletClient: {
            account: { address: ATTACKER },
            chain: null,
            request: vi.fn().mockResolvedValue(OLD_HASH),
          } as unknown as WalletClient,
        },
        {
          id: 'tx-revoke-roles',
          publicClient,
        },
      )
    })

    await waitFor(() => {
      expect(
        transactionManager
          .getTransaction('tx-revoke-roles')
          ?.getSnapshot().value,
      ).toBe('success')
    })

    act(() => openTransactionModal())

    expect(await screen.findByRole('button', { name: 'Done' })).toBeVisible()
    expect(revokeRoles).not.toHaveBeenCalled()

    fireEvent.click(screen.getByText('Remove user'))
    expect(await screen.findByText(/victim\.eth/)).toBeVisible()
    expect(screen.getByRole('link')).toHaveAttribute(
      'href',
      `https://explorer.invalid/tx/${OLD_HASH}`,
    )

    fireEvent.click(screen.getByRole('button', { name: 'Done' }))
    expect(handleDone).toHaveBeenCalledOnce()
    expect(revokeRoles).not.toHaveBeenCalled()
  })
})
```

Run:

```bash
cd apps/portal
pnpm exec vitest run src/features/transaction-manager/components/TransactionStateContent.stale-id.poc.test.tsx --reporter=verbose
```

Observed result:

```
Transaction tx-revoke-roles state: pending
Transaction tx-revoke-roles state: success
1 file passed
2 tests passed
```

The integrated test demonstrates that the first actor reaches real transaction-manager success, stays available, and makes the later `victim.eth` descriptor render Done before its new `revokeRoles` callback runs. Clicking Done executes cleanup, while `revokeRoles` remains at zero calls.

### Test 2: independent transaction-manager retention proof

Create:

```
packages/transaction-manager/src/providers/transactionManager.stale-id.poc.test.ts
```

```ts
import type { Address, PublicClient, TransactionReceipt, WalletClient } from 'viem'
import { afterEach, describe, expect, it, vi } from 'vitest'
import type { EOASigner } from '../types/signer.types'
import type { EOATransactionRequest } from '../types/transaction.types'
import { transactionManager } from './transactionManager'

const sendTransaction = vi.fn()
vi.mock('viem/actions', async (importOriginal) => {
  const actual = await importOriginal<typeof import('viem/actions')>()
  return {
    ...actual,
    sendTransaction: (...args: unknown[]) => sendTransaction(...args),
  }
})

const ACCOUNT = '0x1111111111111111111111111111111111111111' as Address
const TO = '0x2222222222222222222222222222222222222222' as Address
const HASH = `0x${'cd'.repeat(32)}` as const

afterEach(() => {
  transactionManager.clear()
  vi.clearAllMocks()
})

describe('terminal transaction lifecycle PoC', () => {
  it('retains a successful fixed-id actor in the active global map', async () => {
    sendTransaction.mockResolvedValue(HASH)

    const request: EOATransactionRequest = {
      type: 'eoa',
      from: ACCOUNT,
      to: TO,
      chainId: 1,
      value: 0n,
      data: '0x',
    }
    const signer: EOASigner = {
      type: 'eoa',
      walletClient: {
        account: { address: ACCOUNT },
        chain: null,
      } as unknown as WalletClient,
    }
    const publicClient = {
      waitForTransactionReceipt: vi.fn().mockResolvedValue({
        status: 'success',
        transactionHash: HASH,
      } as TransactionReceipt),
    } as unknown as PublicClient

    const id = transactionManager.startTransaction(request, signer, {
      id: 'tx-revoke-roles',
      publicClient,
    })

    await vi.waitFor(() => {
      expect(transactionManager.getTransaction(id)?.getSnapshot().value).toBe(
        'success',
      )
    })

    const retained = transactionManager.getTransaction('tx-revoke-roles')
    expect(retained).toBeDefined()
    expect(retained?.getSnapshot().context.hash).toBe(HASH)
    expect(transactionManager.getTransactions().has(id)).toBe(true)
  })
})
```

Run:

```bash
cd packages/transaction-manager
pnpm exec vitest run src/providers/transactionManager.stale-id.poc.test.ts --reporter=verbose
```

Observed result: 1/1 passed. Running the complete transaction-manager invocation also passed 119/119.

### Typechecks

```bash
cd apps/portal && pnpm typecheck
cd packages/transaction-manager && pnpm typecheck
```

Both commands exit successfully.

### Supporting contract reachability

From the exact contracts-v2 checkout:

```bash
cd contracts
forge test --match-path 'test/unit/registry/PermissionedRegistry.t.sol' \
  --match-test 'test_setResolver|test_revokeRoles_asOwnerHavingAdmin' -vv
```

Observed result: 6/6 passed. This confirms that a holder of the retained resolver role can call `setResolver`, and that a genuine owner revocation would remove roles. Contracts are supporting impact evidence, not the claimed vulnerable asset.

### Manual reproduction

{% stepper %}
{% step %}

## Complete any role revocation for Name A.

{% endstep %}

{% step %}

## Close the successful modal with X, or navigate to another route, instead of clicking Done.

Do not reload the application.
{% endstep %}

{% step %}

## Open Name B (`victim.eth`) and select a different delegate holding `ROLE_SET_RESOLVER`.

{% endstep %}

{% step %}

## Click Remove user and confirm.

{% endstep %}

{% step %}

## Observe that the modal immediately shows Remove user / Done with Name A's old explorer hash.

No new wallet prompt appears.
{% endstep %}

{% step %}

## Open the operation detail.

Name B/new user is displayed beside the old transaction link.
{% endstep %}

{% step %}

## Click Done.

{% endstep %}

{% step %}

## Query `hasRoles` for Name B.

The supposedly removed delegate still holds `ROLE_SET_RESOLVER`, and no Name B revocation transaction exists.
{% endstep %}
{% endstepper %}

### Integrity

PoC bundle SHA-256:

```
35d12f7b6ec3a187c4784eb9c1dcca26adb02dfd0039648f9cb4f0b180881040
```
