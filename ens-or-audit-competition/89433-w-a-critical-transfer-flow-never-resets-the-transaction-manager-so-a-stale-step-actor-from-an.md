# 89433 w a critical transfer flow never resets the transaction manager so a stale step actor from an abandoned attempt is matched by id to the next attempt and the steps before it are skipped whi&#x20;

**Submitted on Aug 19th 2026 at 10:11:33 UTC by @c0mpos3r for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89433
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

`useTransferName` labels each step of a name transfer with a deterministic id built from the name and the step kind only. Starting a new attempt resets the hook's own guard but leaves the transaction manager untouched, and the modal picks its active step by matching the manager's last entry against the new attempt's array by that id.

An owner who abandons one attempt and starts another therefore lands on the last step of the plan. The configured steps before it are never invoked, and the overview renders them with the previous attempt's terminal status.

The transfer completes. The token moves, and the pre-transfer configuration the owner selected has not been applied.

Asset: `apps/portal`. Commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

## Vulnerability Details

### 1. Step ids carry no attempt identity

`apps/portal/src/features/transfer/hooks/useTransferName.ts`:

```ts
const id = `transfer-${name}-${step}`   // :159, and the same template at :175
```

`recipient` and the option toggles are captured by the runner closures only. Two attempts on the same name produce byte-identical ids.

### 2. A new attempt does not reset the manager

```ts
onSuccess: (params) => {                 // :116-120
  startedStepsRef.current = new Set()
  setSavedParams(params)
  openModal()
},
```

Only the hook's own idempotency guard is cleared. `transactionManager` is a module-level singleton (`packages/transaction-manager/src/providers/transactionManager.ts:338-339`, `:480-488`) whose Map is emptied only by `clear()`.

The single `clearTransaction()` call in this feature is at `:87`, inside `finishFlow`, which is reachable only as the last step's `onDone` (`:180`). It never runs at the head of a flow.

### 3. Neither closing the modal nor navigating away clears it

`features/transaction-manager/components/TransactionModal.tsx:56-65`:

```ts
if (!open) {
  handleClose()
  // Only clear when transaction has failed - don't clear pending (tx may still be in wallet)
  // or success (user might want to reopen and see the result)
  if (txState?.error) {
    clearTransaction()
  }
}
```

Dismissing while the last entry is pending or successful leaves every actor in place. The dialog has no dismissal guard, so it can be closed at any point in the flow. `useTransactionModal.ts:36-42` subscribes to route changes and calls `closeTransactionModal()` only.

### 4. The stale actor selects the active step of the new attempt

`hooks/useActiveTransactionState.ts:46-47` takes the last entry of the manager Map:

```ts
const entries = Array.from(transactions.entries())
const lastEntry = entries[entries.length - 1]
```

`utils/getActiveTransaction.ts:16-21` then matches that id against the new attempt's array:

```ts
if (txState) {
  const tx = transactions.find((t) => t.id === txState.txId)
  if (tx) return tx
}
return transactions[0]
```

The `transactions[0]` fallback is correct. It is never reached, because the ids from step 1 always match.

### 5. There is no path back to the skipped steps

`components/TransactionStateContent.tsx:46` computes the active step with that same call, and `:193` binds the primary button to it:

```tsx
onClick={activeTransaction.onStart}
```

so "Try again" invokes the runner for the matched step, not for the steps before it. `hooks/useAutoAdvanceTransaction.ts:23-28` finds the active index and calls `txs[activeIndex].onDone()`, which only moves forward.

`components/TransactionsOverviewContent.tsx` renders each row from `getStatus(transaction.id, activeTransactionsMap)`, so a stale successful actor makes the new attempt's row for that step display as finished.

### 6. A stale successful actor needs no click at all

`TransactionModal.tsx:24-25` derives the auto-advance id from the same stale state:

```ts
const autoAdvanceTxId =
  isOpen && txState?.machineState === 'success' ? txState.txId : null
```

and passes it to `useAutoAdvanceTransaction` at `:45`, whose effect (`:20-28`) resolves the id against the current array and calls `txs[activeIndex].onDone()`, the runner for the following step.

If the owner abandoned the previous attempt in the window after a step succeeded and before the next one started, that success is the manager's last entry. Opening the modal on the next attempt fires the following step immediately: the wallet prompts for the token transfer without the owner clicking anything, and the step before it never runs.

### 7. The sibling flows do reset, and one of them names this hazard

Four other flows clear the manager at flow head:

* `features/register/hooks/useRegistrationTransactions.ts:163` and `:182` call `transactionManager.clear()`
* `features/renew/hooks/useRenewalTransactions.ts:169`
* `routes/addr/$addr/names.tsx:358`
* `features/renew/components/ExtendNameButton.tsx:57-61`:

```ts
// Stale terminal-state transactions (success/error) block the modal;
// remove only that entry so a fresh extend flow can start without
// touching any other in-flight transactions in the manager.
if (activeTxState) {
  transactionManager.cancelTransaction(activeTxState.txId)
}
```

`features/transfer/components/SendNameForm.tsx:106-109` calls `startTransfer` with neither a reset nor a stale-entry check.

## Impact Details

The transfer succeeds while omitting steps the owner selected, and both parties are told those steps completed.

**Stale address record.** The name transfers to B while `addr(60)` still resolves to A, the recipient of the abandoned attempt. Anyone paying the name after the transfer sends funds to A rather than to the new owner. Neither party is prompted to check, because the modal displayed the address step as finished.

**Skipped detaches.** If the second attempt enables "Detach the resolver" or "Detach the registry", those steps are dropped and the token moves with the sender's resolver and subregistry still attached. The new owner receives a name whose resolution and subname authority still point at the previous owner's contracts, and the previous owner keeps whatever roles they hold on that resolver.

The sender cannot correct this afterwards. `buildTransferPlan.ts:1-5` states the reason the config steps are ordered first:

> Config steps run before the token moves, since the sender loses the roles to make them once the ERC-1155 token transfers.

The skip therefore breaks the exact invariant that ordering exists to protect.

**Relation to the known issues.** This is not the transfer-errors family. QA-01 and WEB-1305 describe transfers that fail or hang, and WEB-1257 describes being unable to set a primary name after a parent transfer. Here the transfer completes and reports success. Per the known-issues document, a demonstrated security consequence rather than the breakage itself is a new finding, and the consequence here is a name whose address record and attachments point at parties the new owner never chose.

## Severity

Selected impact: "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient".

The overview shows the address step as finished when it did not run in this attempt, and the resulting record sends third-party payments for the name to an address neither the sender nor the new owner intended.

Factors that reduce it:

* Reaching the state takes two attempts. The stale actors survive only if the modal is dismissed while its last entry is pending or successful; dismissing on an error does clear them (`TransactionModal.tsx:61-63`). Within the second attempt no further interaction is required when the stale entry is a success, since `useAutoAdvanceTransaction` fires on open.
* The skipped-detach instance additionally requires a toggle change between attempts. The stale-address instance requires only a recipient change, which is the usual reason to abandon an attempt.
* The new owner holds the roles to repair the records once they notice.
* `apps/portal` is Sepolia-only today (`lib/wagmi.ts:67`, `chains: [sepoliaWithEns]`), so this is a pre-launch defect rather than a live-funds issue.

The finding needs no contract semantics, no indexer behaviour and no deployment state. Every step is decided in client code.

## Suggested fix

Reset the manager at the head of the flow, as the register and renew flows already do. In `useTransferName.ts`, `prepareMutation.onSuccess` (`:116-120`) is the natural place:

```ts
onSuccess: (params) => {
  transactionManager.clear()          // or cancelTransaction for the stale ids only
  startedStepsRef.current = new Set()
  setSavedParams(params)
  openModal()
},
```

`ExtendNameButton.tsx:57-61` is the more conservative precedent: cancel only the stale entry so unrelated in-flight transactions elsewhere are not disturbed.

Making the ids attempt-specific would also break the accidental match, but it does not remove the residue, so a later flow could still read a foreign entry as its active step. Clearing at flow head addresses the cause.

## Notes

`utils/buildTransferPlan.test.ts` and the transaction-manager tests cover the plan and the machine in isolation. No test covers a second attempt against a non-empty manager, which is the state this report describes.

Searching `features/transfer` for `known|limitation|caveat|by design|intended|TODO|FIXME|trade-off|acceptable|assume` returns nothing that describes the residue as acceptable. `TransactionModal.tsx:59-60` explains why the manager is not cleared at close time, which is a separate decision; the sibling flows document the opposite for flow head.

## References

| File                                                                                  | Lines                      |
| ------------------------------------------------------------------------------------- | -------------------------- |
| `apps/portal/src/features/transfer/hooks/useTransferName.ts`                          | 87, 116-120, 159, 175, 180 |
| `apps/portal/src/features/transfer/utils/buildTransferPlan.ts`                        | 1-5, 28                    |
| `apps/portal/src/features/transfer/components/SendNameForm.tsx`                       | 106-109                    |
| `apps/portal/src/features/transaction-manager/components/TransactionModal.tsx`        | 24-25, 45, 56-65           |
| `apps/portal/src/features/transaction-manager/hooks/useTransactionModal.ts`           | 14-16, 36-42               |
| `apps/portal/src/features/transaction-manager/hooks/useActiveTransactionState.ts`     | 46-47                      |
| `apps/portal/src/features/transaction-manager/utils/getActiveTransaction.ts`          | 16-21                      |
| `apps/portal/src/features/transaction-manager/utils/getStatus.ts`                     | 20-31                      |
| `apps/portal/src/features/transaction-manager/components/TransactionStateContent.tsx` | 46, 193                    |
| `apps/portal/src/features/transaction-manager/hooks/useAutoAdvanceTransaction.ts`     | 20-28                      |
| `apps/portal/src/features/register/hooks/useRegistrationTransactions.ts`              | 163, 182                   |
| `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts`                      | 169                        |
| `apps/portal/src/features/renew/components/ExtendNameButton.tsx`                      | 57-61                      |
| `apps/portal/src/routes/addr/$addr/names.tsx`                                         | 358                        |
| `packages/transaction-manager/src/providers/transactionManager.ts`                    | 338-339, 480-488           |

## Proof of Concept

Deterministic and local-only: no network access, no wallet, no contract, and no transaction is broadcast. The test calls the app's own `buildTransferPlan`, `getActiveTransaction`, `getStatus` and `useAutoAdvanceTransaction` exactly as the modal calls them. The only inlined expression is the step-id template, copied verbatim from `useTransferName.ts:159`.

### Setup

{% stepper %}
{% step %}

## Clone and install dependencies

Clone the audit repo at `1c9b47f18fcddd2e864dfe385c4171061c9811ae` and run:

```sh
pnpm install --ignore-scripts
```

{% endstep %}

{% step %}

## Save the test file

Save the file below as `apps/portal/src/features/transfer/c18-poc.test.ts`.
{% endstep %}

{% step %}

## Run the test

From `apps/portal`, run:

```sh
pnpm vitest run src/features/transfer/c18-poc.test.ts --silent=false
```

{% endstep %}
{% endstepper %}

### Scenario

{% stepper %}
{% step %}

## Open the transfer flow

The owner opens `/foo.eth/ownership/transfer` for a v2 2LD they own. With "keep the resolver, point the ETH address at the recipient" the plan is `[set-eth-addr, transfer-token]` (`buildTransferPlan.ts:28`).
{% endstep %}

{% step %}

## Start the first attempt

Recipient A is entered. `set-eth-addr` is signed and succeeds, so `addr(60)` is now A. The flow auto-advances and opens the wallet for `safeTransferFrom` to A.
{% endstep %}

{% step %}

## Abandon the transfer

With the wallet prompt open the owner realises A is wrong and dismisses the modal. `txState` is the pending token step, not an error, so `TransactionModal.tsx:60-63` does not clear. The dialog has no dismissal guard.
{% endstep %}

{% step %}

## Reject the wallet prompt

The owner rejects the wallet prompt. The token actor settles to error, but the modal is already closed, so nothing clears it.
{% endstep %}

{% step %}

## Start a second attempt

The owner returns to the form, enters recipient B and clicks Transfer. A fresh transactions array is built with the same two ids and new runners bound to B.
{% endstep %}

{% step %}

## Open on the stale step

The modal reads the manager's last entry, matches it by id, and opens on the token step. The address row renders with the previous attempt's status.
{% endstep %}

{% step %}

## Retry the transfer

The owner clicks Try again. The token step runs against B. The address step never runs.
{% endstep %}
{% endstepper %}

### Result (actual test output)

```
=== C-18 PoC RESULT ===
attempt 1 recipient        : 0xAAAA000000000000000000000000000000000000 (abandoned)
attempt 2 recipient        : 0xBBBB000000000000000000000000000000000000
plan built for attempt 2   : set-eth-addr -> transfer-token
stale actor id in manager  : transfer-foo.eth-transfer-token
active step chosen by modal: transfer-foo.eth-transfer-token
set-eth-addr runner calls  : 0
transfer-token runner calls: 1
=> the token moves to B while addr(60) still points at A.

=== C-18 PoC: no-click variant ===
stale success actor        : transfer-foo.eth-set-eth-addr
user interaction required  : none (modal open alone triggers it)
set-eth-addr runner calls  : 0
transfer-token runner calls: 1

Test Files  1 passed (1)
     Tests  8 passed (8)
```

The eight tests cover: the documented plan order; identical ids across attempts; the stale match selecting the last step; the skipped runner never being invoked; the skipped row rendering as finished via `getStatus`; correct behaviour once the manager is cleared, which is what the sibling flows do; the same skip with both detach toggles on, where the two irreversible steps are the ones dropped; and the no-click variant, which drives the app's own `useAutoAdvanceTransaction` through `renderHook`.

### Test file

```ts
import type { TransactionMachineActor } from '@ens-apps/transaction-manager'
import { renderHook } from '@testing-library/react'
import { describe, expect, it, vi } from 'vitest'
import { useAutoAdvanceTransaction } from '@/features/transaction-manager/hooks/useAutoAdvanceTransaction'
import type { ActiveTransactionState } from '@/features/transaction-manager/hooks/useActiveTransactionState'
import type { Transaction } from '@/features/transaction-manager/types'
import { getActiveTransaction } from '@/features/transaction-manager/utils/getActiveTransaction'
import { getStatus } from '@/features/transaction-manager/utils/getStatus'
import {
  buildTransferPlan,
  STEP_LABELS,
  type TransferOptions,
} from './utils/buildTransferPlan'

const NAME = 'foo.eth'

/**
 * Rebuilds exactly what useTransferName.buildTransactions produces, including
 * the id template from useTransferName.ts:159 / :175:
 *   const id = `transfer-${name}-${step}`
 * `recipient` is captured by the runners only, never by the id.
 */
const buildAttempt = (options: TransferOptions, recipient: string) => {
  const steps = buildTransferPlan(options)
  const runners = steps.map((step) => vi.fn().mockName(`${step}->${recipient}`))
  const transactions: Transaction[] = steps.map((step, i) => ({
    id: `transfer-${NAME}-${step}`,
    title: STEP_LABELS[step],
    transactionName: `${STEP_LABELS[step]} - ${NAME}`,
    onStart: runners[i],
    onDone: i < runners.length - 1 ? runners[i + 1] : () => {},
  }))
  return { steps, runners, transactions }
}

/** Minimal stand-in for the actor the manager keeps; getStatus only calls getSnapshot(). */
const fakeActor = (value: string, error?: Error) =>
  ({
    getSnapshot: () => ({ value, context: { error } }),
  }) as unknown as TransactionMachineActor

// The owner's first attempt: send to A, keeping the resolver, so the plan is
// [set-eth-addr, transfer-token].
const OPTIONS: TransferOptions = {
  setEthAddress: true,
  detachResolver: false,
  detachRegistry: false,
}
const RECIPIENT_A = '0xAAAA000000000000000000000000000000000000'
const RECIPIENT_B = '0xBBBB000000000000000000000000000000000000'

describe('C-18 PoC - stale actors hijack the active step of a new transfer attempt', () => {
  it('step 1: the plan is the documented config-before-token order', () => {
    expect(buildTransferPlan(OPTIONS)).toEqual(['set-eth-addr', 'transfer-token'])
  })

  it('step 2: two different attempts produce IDENTICAL step ids', () => {
    const a = buildAttempt(OPTIONS, RECIPIENT_A)
    const b = buildAttempt(OPTIONS, RECIPIENT_B)

    // useTransferName.ts:159 - the id carries the name and step kind only.
    // No recipient, no attempt nonce, so the manager cannot tell the attempts apart.
    expect(a.transactions.map((t) => t.id)).toEqual(
      b.transactions.map((t) => t.id),
    )
    expect(a.transactions.map((t) => t.id)).toEqual([
      'transfer-foo.eth-set-eth-addr',
      'transfer-foo.eth-transfer-token',
    ])
  })

  it('step 3: a stale actor makes the NEW attempt open on the LAST step', () => {
    // Attempt 1 was abandoned with its transfer-token step in a terminal error
    // state. TransactionModal.tsx:60-63 only clears on error AT CLOSE TIME, and
    // useTransactionModal.ts:36-42 (route change) only closes the modal, so the
    // actor survives into attempt 2.
    const stale: ActiveTransactionState = {
      txId: 'transfer-foo.eth-transfer-token',
      machineState: 'error',
      hash: undefined,
      error: new Error('user rejected'),
    }

    const attempt2 = buildAttempt(OPTIONS, RECIPIENT_B)

    // This is the exact call TransactionStateContent.tsx:46 makes.
    const active = getActiveTransaction(attempt2.transactions, stale)

    expect(active.id).toBe('transfer-foo.eth-transfer-token')
    expect(active).toBe(attempt2.transactions[1]) // NOT transactions[0]
  })

  it('step 4: "Try again" runs the token step and NEVER the address step', () => {
    const stale: ActiveTransactionState = {
      txId: 'transfer-foo.eth-transfer-token',
      machineState: 'error',
      hash: undefined,
      error: new Error('user rejected'),
    }

    const attempt2 = buildAttempt(OPTIONS, RECIPIENT_B)
    const active = getActiveTransaction(attempt2.transactions, stale)

    // TransactionStateContent.tsx:193 - onClick={activeTransaction.onStart}
    active.onStart()

    const [setEthAddrRunner, transferTokenRunner] = attempt2.runners
    expect(transferTokenRunner).toHaveBeenCalledTimes(1)
    expect(setEthAddrRunner).not.toHaveBeenCalled() // <-- the configured step is skipped

    console.log('\n=== C-18 PoC RESULT ===')
    console.log('attempt 1 recipient        : ' + RECIPIENT_A + ' (abandoned)')
    console.log('attempt 2 recipient        : ' + RECIPIENT_B)
    console.log('plan built for attempt 2   : ' + attempt2.steps.join(' -> '))
    console.log('stale actor id in manager  : ' + stale.txId)
    console.log('active step chosen by modal: ' + active.id)
    console.log('set-eth-addr runner calls  : ' + setEthAddrRunner.mock.calls.length)
    console.log('transfer-token runner calls: ' + transferTokenRunner.mock.calls.length)
    console.log('=> the token moves to B while addr(60) still points at A.\n')
  })

  it('step 5: the skipped row is rendered as already finished, not as pending', () => {
    // Attempt 1's set-eth-addr actor settled successfully and is still in the map,
    // keyed by the SAME id the new attempt uses.
    const managerMap = new Map<string, TransactionMachineActor>([
      ['transfer-foo.eth-set-eth-addr', fakeActor('success')],
      [
        'transfer-foo.eth-transfer-token',
        fakeActor('error', new Error('user rejected')),
      ],
    ])

    const attempt2 = buildAttempt(OPTIONS, RECIPIENT_B)

    // TransactionsOverviewContent renders each row from getStatus(transaction.id, map).
    expect(getStatus(attempt2.transactions[0].id, managerMap)).toBe('success')
    expect(getStatus(attempt2.transactions[1].id, managerMap)).toBe('error')
    // So the address step the user configured shows as done, having never run.
  })

  it('step 6: clearing the manager at flow head restores correct behaviour', () => {
    // What the sibling flows do: useRegistrationTransactions.ts:163 and :182 call
    // transactionManager.clear(); ExtendNameButton.tsx:61 cancels the stale entry.
    // With no stale state, getActiveTransaction falls back to transactions[0].
    const attempt2 = buildAttempt(OPTIONS, RECIPIENT_B)
    const active = getActiveTransaction(attempt2.transactions, undefined)

    expect(active).toBe(attempt2.transactions[0])
    active.onStart()
    expect(attempt2.runners[0]).toHaveBeenCalledTimes(1)
    expect(attempt2.runners[1]).not.toHaveBeenCalled()
  })

  it('step 7: with detach toggles on, the skipped steps are the irreversible ones', () => {
    // Same abandonment, but attempt 2 turns the detaches on. The stale token
    // actor still selects the LAST step.
    const withDetaches: TransferOptions = {
      setEthAddress: true,
      detachResolver: true,
      detachRegistry: true,
    }
    const attempt2 = buildAttempt(withDetaches, RECIPIENT_B)

    // buildTransferPlan.ts:28 - set-eth-addr is dropped when the resolver is detached.
    expect(attempt2.steps).toEqual([
      'detach-resolver',
      'detach-registry',
      'transfer-token',
    ])

    const stale: ActiveTransactionState = {
      txId: 'transfer-foo.eth-transfer-token',
      machineState: 'error',
      hash: undefined,
      error: new Error('user rejected'),
    }
    const active = getActiveTransaction(attempt2.transactions, stale)
    active.onStart()

    expect(attempt2.runners[0]).not.toHaveBeenCalled() // detach-resolver skipped
    expect(attempt2.runners[1]).not.toHaveBeenCalled() // detach-registry skipped
    expect(attempt2.runners[2]).toHaveBeenCalledTimes(1) // token moves anyway
  })

  it('step 8: a stale SUCCESS actor auto-fires the next step with NO user click', () => {
    // Sharper variant. The owner dismissed attempt 1 in the window after
    // set-eth-addr succeeded and before the token step started, so that success
    // is the manager's only, and therefore last, entry.
    const staleSuccess: ActiveTransactionState = {
      txId: 'transfer-foo.eth-set-eth-addr',
      machineState: 'success',
      hash: '0xdead',
      error: undefined,
    }

    const attempt2 = buildAttempt(OPTIONS, RECIPIENT_B)

    // TransactionModal.tsx:24-25, verbatim condition.
    const isOpen = true
    const autoAdvanceTxId =
      isOpen && staleSuccess.machineState === 'success' ? staleSuccess.txId : null

    // TransactionModal.tsx:45 - the app's own hook, unmodified.
    renderHook(() =>
      useAutoAdvanceTransaction(autoAdvanceTxId, attempt2.transactions),
    )

    const [setEthAddrRunner, transferTokenRunner] = attempt2.runners
    expect(setEthAddrRunner).not.toHaveBeenCalled()
    expect(transferTokenRunner).toHaveBeenCalledTimes(1)

    console.log(
      '\n=== C-18 PoC: no-click variant ===\n' +
        'stale success actor        : ' + staleSuccess.txId + '\n' +
        'user interaction required  : none (modal open alone triggers it)\n' +
        'set-eth-addr runner calls  : ' + setEthAddrRunner.mock.calls.length + '\n' +
        'transfer-token runner calls: ' + transferTokenRunner.mock.calls.length + '\n',
    )
  })
})
```

### Testing conduct

All work was local: no deployed or public-testnet contract was contacted and no transaction was signed or broadcast.
