# 93093 w a critical a pending role grant can be relabelled with a trusted recipient while retaining attacker calldata

## #93093 \[W\&A-Critical] A pending role grant can be relabelled with a trusted recipient while retaining attacker calldata

**Submitted on Sep 14th 2026 at 08:28:05 UTC by @kl4r10n for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93093
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

### Summary

The Portal role-grant flow uses the constant transaction ID `tx-grant-roles` for every execution. The application treats that ID as both:

1. The identity of the live transaction actor and immutable wallet request; and
2. The identity of a newly rendered UI descriptor constructed from the current form state.

Those two objects do not have the same lifetime.

After the victim opens a wallet request granting a role to address A, closing Portal's transaction modal intentionally leaves the submitting actor alive because the wallet request may still be pending. Closing the parent sheet simultaneously erases its recipient and role state. The sheet can then be reopened and populated with address B.

When the confirmation modal is reopened, its descriptor is reconstructed from B but retains the same fixed ID. `getActiveTransaction()` joins that new descriptor to the old actor by ID alone. The transaction-details component consequently renders B's description while reading status and request data from A's actor. Accepting the already-open wallet prompt executes the old calldata for A.

For the security-relevant role `ROLE_SET_RESOLVER`, this creates the following result:

```
Portal transaction detail: Grant roles for 0x3333…3333   (trusted, truncated)
Live wallet request:        grantRoles(resource, ROLE_SET_RESOLVER, ATTACKER)
On-chain result:            ATTACKER may change the name's resolver
```

The exploit does not require a second wallet request. The second **Save** only rebuilds the descriptor and reopens the modal. Because `tx-grant-roles` is already `submitting`, `TransactionStateContent` does not call `onStart`: the overview shows **Next** rather than **Start**, and the state view shows disabled **Waiting...** rather than **Open wallet**. The victim returns to the still-open first wallet prompt after using Portal to inspect the relabelled in-progress transaction.

Production `transactionName` uses `truncateAddress(address, 6, 4)` from the live resolved form field, not a frozen copy of the actor's grantee. The UI therefore shows a truncated trusted address even when the pending request is still the attacker.

### Root cause

#### 1. Every role-grant execution uses one fixed identity

`apps/portal/src/features/roles/components/RolesAddUserSheet.tsx:32` declares a process-wide semantic ID:

```ts
const GRANT_ROLES_TX_ID = 'tx-grant-roles'
```

The same ID is passed into every new role grant at lines 110–118:

```ts
grantRoles({
  name,
  account: pendingGrant.account,
  roles: pendingGrant.roles,
  id: GRANT_ROLES_TX_ID,
  registryAddress,
})
```

The ID identifies an operation type, not a unique execution or a specific `(name, account, roles, calldata)` tuple.

#### 2. Closing the transaction modal deliberately preserves the in-flight actor

`apps/portal/src/features/transaction-manager/components/TransactionModal.tsx:54-65` closes only the UI. It clears the transaction manager only after an error:

```tsx
onOpenChange={(open) => {
  if (!open) {
    handleClose()
    // Only clear when transaction has failed - don't clear pending (tx may still be in wallet)
    // or success (user might want to reopen and see the result)
    if (txState?.error) {
      clearTransaction()
    }
  }
}}
```

`apps/portal/src/features/transaction-manager/hooks/useTransactionModal.ts:8-16` confirms that closing the modal and clearing the transaction are independent operations:

```ts
export const closeTransactionModal = () => transactionModalAtom.set(false)

const clearTransaction = () => {
  transactionManager.clear()
}
```

Therefore, returning focus to Portal and closing the application modal while the extension prompt remains open leaves the original actor in `submitting` state with the original request.

#### 3. Closing the parent sheet erases the immutable context needed to describe that actor

`RolesAddUserSheet.tsx:66-71` stores the form recipient, roles, and prepared grant in component state. Lines 79–88 erase all of it whenever the sheet closes:

```ts
useEffect(() => {
  if (open) return
  setUserInput('')
  setSelectedRoles(new Set())
  setPendingGrant(null)
  reset()
}, [open, reset])
```

This reset does not clear or cancel the transaction manager actor. The UI context and the actor context can now diverge.

#### 4. The reopened flow describes the transaction with live form state

On a later render, `RolesAddUserSheet.tsx:239-258` constructs a fresh descriptor under the same fixed ID. Its displayed recipient comes from the current resolved `address`, whereas its prepared request would come from the current `pendingGrant`:

```tsx
<TransactionModal
  transactions={[
    {
      id: GRANT_ROLES_TX_ID,
      title: 'Grant roles',
      transactionName: address
        ? `Grant roles for ${truncateAddress(address, 6, 4)}`
        : 'Grant roles',
      intent: {
        prepare: pendingGrant
          ? ({ walletClient, chainId }) =>
              prepareGrantRolesTransaction({
                name,
                account: pendingGrant.account,
                roles: pendingGrant.roles,
                walletClient,
                chainId,
                registryAddress,
              })
          : undefined,
      },
      onStart: handleStartTransaction,
      onDone: handleDone,
    },
  ]}
/>
```

Nothing binds this descriptor to the request already owned by the live actor.

#### 5. The transaction manager stores and exposes the actor by the caller-supplied ID

`packages/transaction-manager/src/providers/transactionManager.ts:176-251` creates and starts a transaction actor. After subscribing to it, lines 338–340 store it under the provided ID:

```ts
// Add to active transactions
this.transactions.set(txId, actor)
this.notifyListeners()
```

The manager does not attach a request fingerprint to the UI descriptor and does not require the ID to be a unique execution identifier.

`apps/portal/src/features/transaction-manager/hooks/useActiveTransactionState.ts:44-67` derives the active transaction solely from the map entry and exposes only the map key as `txId`:

```ts
const entries = Array.from(transactions.entries())
const lastEntry = entries[entries.length - 1]
const [txId, actor] = lastEntry ?? []

// ...

return {
  txId,
  machineState: snapshot.value as TransactionMachineState,
  hash: snapshot.context.hash,
  error: snapshot.context.error,
}
```

#### 6. Portal joins the new descriptor to the old actor by ID only

`apps/portal/src/features/transaction-manager/utils/getActiveTransaction.ts:16-18` performs the unsafe join:

```ts
if (txState) {
  const tx = transactions.find((t) => t.id === txState.txId)
  if (tx) return tx
}
```

No recipient, calldata, chain, contract, or request hash is compared.

`TransactionsOverviewContent.tsx:43-53` selects the fresh descriptor with this helper, then derives its status from the old actor map using the same ID. In the detail view, `TransactionStateContent.tsx:46-47` repeats the join:

```ts
const activeTransaction = getActiveTransaction(transactions, txState)
const activeTxStatus = getStatus(activeTransaction.id, activeTransactionsMap)
```

For each transaction, lines 75–77 fetch the actor snapshot from the map:

```ts
const activeTxSnapshot = activeTransactionsMap
  .get(transaction.id)
  ?.getSnapshot()
```

But lines 112–114 render the transaction description from the new descriptor:

```tsx
<h3 className="text-base font-medium whitespace-normal leading-snug break-all min-w-0">
  {transaction.transactionName}
</h3>
```

This is the concrete descriptor/actor split: presentation comes from the new form, while execution comes from the old actor.

#### 7. The old request contains real attacker-role calldata

`apps/portal/src/features/roles/helpers/grantRoles.ts:58-94` builds the production request and encodes the selected account and roles directly into `grantRoles` calldata:

```ts
const writeParams = grantRolesWriteParameters(
  walletClient as Parameters<typeof grantRolesWriteParameters>[0],
  {
    registryAddress,
    account,
    resource,
    roles: [...roles],
  },
)

const data = encodeFunctionData({
  abi: writeParams.abi,
  functionName: writeParams.functionName,
  args: writeParams.args,
} as Parameters<typeof encodeFunctionData>[0])
```

The Portal's own permission copy in `apps/portal/src/lib/roles/permissions.ts:31-33` states the consequence of the role used in the PoC:

```ts
{
  key: 'ROLE_SET_RESOLVER',
  title: 'Set Resolver',
  description: 'Can change the resolver addresses',
}
```

### Preconditions

1. The victim is authorized to grant roles for a Portal-managed name.
2. A harmful role-grant request addressed to the attacker reaches the wallet prompt. This can result from a wrong paste, deceptive instructions, a stale input, or another in-scope address-confusion issue. The fixed-ID vulnerability does not depend on a particular source for that first address.
3. The wallet request remains pending while the victim returns focus to the Portal tab. Browser wallets commonly present the request in a separate extension window, so returning to the dapp does not itself reject it.
4. The victim closes the Portal transaction modal and role sheet, then reopens the role flow and enters the intended trusted address.
5. The victim inspects the in-progress transaction details and then accepts the still-open wallet request.

This is a multi-step social/UX sequence. It does not require control of the victim's browser, wallet, private key, or application session.

### Step-by-step attack

1. Alice manages `valuable.eth` in Portal.
2. A role-grant flow is prepared for Mallory's address with `ROLE_SET_RESOLVER`.
3. Alice clicks **Open wallet**. The production transaction actor enters `submitting`, and the extension displays the `grantRoles(..., Mallory)` request.
4. Alice returns focus to Portal without accepting or rejecting the extension request.
5. Alice closes Portal's transaction modal. The code intentionally leaves the submitting actor alive.
6. Alice closes the **Add user** sheet. Portal erases `userInput`, `selectedRoles`, and `pendingGrant`, but the actor and wallet request remain.
7. Alice reopens **Add user**, enters Bob's trusted address, selects the same role, and clicks **Save**. This only `setPendingGrant(Bob)` and `openModal()`. It does not call `grantRoles` / `onStart`.
8. The newly mounted transaction descriptor is `Grant roles for 0x3333…3333` (Bob, 6/4 truncated) and has ID `tx-grant-roles`. Because the actor is already `submitting`, the modal shows **Waiting...**, not a second **Open wallet**.
9. The existing actor also has ID `tx-grant-roles`, so Portal labels its in-progress status/detail with Bob's fresh descriptor. Opening the transaction detail displays truncated Bob while the wallet request is still Mallory.
10. Reassured by Portal, Alice returns to the original wallet prompt and signs it.
11. The actual calldata grants `ROLE_SET_RESOLVER` to Mallory.
12. Mallory calls the permitted resolver-changing function, points `valuable.eth` at an attacker-controlled resolver, and sets its address records to Mallory. Users resolving the name can subsequently send funds to Mallory.

The wallet itself may decode and display the raw grantee address. A user who independently recognizes and compares both hexadecimal addresses can catch the attack. A user who relies on Portal's role recipient—the security UI specifically intended to describe the transaction—cannot.

### Optional amplifier: Portal can source the first attacker address from an ENS name

The descriptor collision is independently exploitable with any pending harmful grant. A separate Portal behavior can make the first attacker address arise wholly inside the normal UI.

`apps/portal/src/features/roles/helpers/addUser.handlers.ts:19-36` tries forward resolution and, when it is empty, falls back to ENS ownership:

```ts
const resolved = await getEnsAddress(client, {
  name: nameOrAddress,
  universalResolverAddress,
})

let resolvedAddress = resolved

if (!resolvedAddress) {
  const ownerResult = await getEnsOwner({ name: nameOrAddress })
  if (ownerResult.isOk()) {
    resolvedAddress = ownerResult.value?.owner ?? null
  }
}

return resolvedAddress
```

For an unwrapped V1 name, `apps/portal/src/utils/ens/resolveEnsOwner.ts:95-101` returns `v1Owner.owner` and does not return or compare the registrar's `registrant`:

```ts
const v1Owner = await getOwnerV1(client, { name })
if (v1Owner?.owner && v1Owner.owner !== zeroAddress) {
  return {
    owner: v1Owner.owner,
    registryAddress: v1EthRegistry,
    protocolVersion: 'ENSv1',
  }
}
```

For an unwrapped V1 name, that value can be the registry controller rather than the registrant. Thus a name with no ETH address record and an attacker-controlled V1 controller can seed Mallory's address into the initial role grant. This is the ENS-05 address-confusion primitive; it is an amplifier, not a prerequisite or the root cause claimed here.

Even if that behavior is fixed, a wrong paste or deceptive initial grantee still reaches the descriptor/actor collision. Conversely, fixing the collision prevents a pending request from being falsely relabelled regardless of how the initial address was obtained.

### Why the impact is Critical

This is not merely stale text or a missing transaction-history entry:

1. Portal presents an incorrect security-sensitive transaction detail: the grantee of an administrative role.
2. The victim then submits a transaction whose actual grantee differs from the displayed grantee.
3. The attacker receives a live on-chain permission, not a cosmetic badge or local state.
4. `ROLE_SET_RESOLVER` lets the attacker redirect the victim's ENS resolution to attacker-controlled records.
5. That sequence matches the listed Critical of displaying or pre-filling an incorrect address so the user submits a transaction to an unintended recipient.

This report does **not** claim that Portal modifies the already-connected wallet's transaction arguments. The pending `eth_sendTransaction` is the original grant to the attacker. The integrity failure is the dapp relabelling that request as a grant to a trusted address.

Exploitability is lower than a zero-click calldata swap because the wallet request must remain pending and the victim must close, reopen, and inspect the Portal flow before signing the old prompt. A wallet that decodes `grantRoles` can show the attacker address; a user who compares hex can catch the mismatch.

## Relationship to known issue R3-07

R3-07 states that a reused transaction ID can skip archiving, history, and telemetry after a prior terminal run because `completedTelemetry` is tracked by transaction ID and never cleared for a retry.

This report shares the design smell of reusable IDs but is not the same root cause or consequence:

| Property                 | R3-07                                                                     | ENS-08                                                                                                                  |
| ------------------------ | ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| State involved           | Terminal completion bookkeeping                                           | Live, non-terminal wallet authorization                                                                                 |
| Trigger                  | A failed/completed run is retried with the same ID                        | A wallet request remains in `submitting` while the form/descriptor is reset and rebuilt                                 |
| Vulnerable logic         | `completedTelemetry.has(txId)` suppresses later archive/history/telemetry | Modal abandonment preserves the actor; UI and actor lifetimes diverge; descriptor and actor are joined only by fixed ID |
| User-visible consequence | Missing or stale transaction history/telemetry                            | Portal displays a trusted grantee while the wallet request grants an attacker                                           |
| On-chain consequence     | None described                                                            | Attacker receives `ROLE_SET_RESOLVER` and may hijack resolution                                                         |
| Fix for known issue      | Scope/reset completion bookkeeping per run                                | Does not bind a live descriptor to its actor/request and does not prevent this attack                                   |
| Fix for this issue       | Unique execution ID plus immutable request-bound UI metadata              | Does not by itself repair R3-07's terminal bookkeeping                                                                  |

Specifically, R3-07's vulnerable guard is `transactionManager.ts:300-302`:

```ts
// Terminal — run the completion side effects exactly once.
if (this.completedTelemetry.has(txId)) return
this.completedTelemetry.add(txId)
```

ENS-08 occurs before the actor is terminal, so that guard is never reached. Deleting or resetting `completedTelemetry`, the direct R3-07 repair, leaves every step of this exploit intact.

The Known Issues rules expressly permit a new consequence of a listed root cause when it materially changes severity. Even if triage treats fixed ID reuse as a common contributing cause, the consequence changes from Medium bookkeeping loss to a privileged on-chain grant to an unintended account and resolver hijack. This is the precise material-severity exception.

### Other non-duplicates

* **R2-03 / broadly scoped session keys:** No session key or HCA session is involved. The victim's connected EOA signs a role grant whose recipient is misrepresented by Portal.
* **EXP-INPUT-005 / ENS input validation:** No homoglyph, normalization, or malformed ENS label is required.
* **QA transaction-display issues:** This report proves that the actual ABI-decoded `account` differs from the address Portal renders, rather than only showing incomplete metadata for the same request.
* **Other Portal flows with fixed IDs** (`tx-reg-register` and similar): the same ID-reuse pattern exists elsewhere. This report claims only the role-grant collision with a pending `grantRoles` actor. Those other flows are out of scope here.

### Recommended remediation

#### 1. Separate operation type from execution identity

Keep `grant-roles` as telemetry metadata if needed, but generate a unique immutable ID for every transaction execution. `TransactionManager.startTransaction()` should reject any duplicate ID whose actor is still active instead of silently treating it as the same logical request.

#### 2. Render transaction details from immutable actor metadata

At transaction start, store a frozen summary containing at least:

* Name/resource;
* Recipient account;
* Role bitmap and human-readable roles;
* Chain ID;
* `from`, `to`, and value; and
* A hash of calldata.

The in-progress and confirmation UI must render this actor-owned snapshot, not fields derived from the current form.

#### 3. Bind every descriptor to the exact request

Compute a fingerprint such as:

```
keccak256(chainId || from || to || value || keccak256(data))
```

Require the descriptor fingerprint to equal the actor request fingerprint before showing an active status or permitting the flow to continue. An ID match alone must never be sufficient.

#### 4. Handle abandoned wallet prompts explicitly

Because a dapp cannot reliably close an extension prompt, closing Portal's modal should either:

* Keep the immutable original recipient and roles visible until the request resolves/rejects; or
* Mark the actor as abandoned and block the same flow from reopening until the old wallet request has settled, with a clear warning that an old request remains open.

Resetting the form while silently retaining the actor is unsafe.

#### 5. Freeze the role-grant descriptor

At minimum, `transactionName` must use the captured `pendingGrant.account`, not the live resolved `address`. This is defense in depth; it does not replace request fingerprinting because `pendingGrant` itself is component state that can be reset.

#### 6. Expand the final confirmation

Show the full grantee address, name/resource, exact roles, registry contract, and chain in both Portal's confirmation and in-progress views. Highlight any mismatch with a still-active request and require the user to reject the stale wallet prompt before proceeding.

### Proof of Concept

### Proof of concept

#### What the PoC proves

The test uses:

* The production `prepareGrantRolesTransaction()` builder;
* The production `transactionManager` and transaction machine;
* The production modal close function;
* The production `getActiveTransaction()` selection logic; and
* ABI decoding of the exact request passed to `eth_sendTransaction`.

The only controlled boundary is a minimal EIP-1193 wallet transport that leaves `eth_sendTransaction` unresolved, reproducing an extension prompt that is still waiting for user approval. No application function is mocked, no calldata is handwritten, and no contract result is fabricated.

The PoC does not mount `RolesAddUserSheet`. It models the reopen descriptor and the production `getActiveTransaction` join. In the live UI, the second Save does not start a second actor: `onStart` runs only when status is `undefined` (Start) or `error` (Try again). While status is submitting, the button is disabled **Waiting...**.

The PoC's `transactionName` uses the full trusted address. Production uses `truncateAddress(address, 6, 4)`, so the live label is `Grant roles for 0x3333…3333`. The ID-only join is the same.

#### PoC file

Add the following as:

`apps/portal/src/features/transaction-manager/F08MutableDescriptorCollision.poc.test.ts`

```ts
import {
  transactionManager,
  type EOASigner,
} from '@ens-apps/transaction-manager'
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest'
import {
  type Address,
  decodeFunctionData,
  type Hash,
  parseAbi,
  type PublicClient,
  type WalletClient,
} from 'viem'
import { sepolia } from 'viem/chains'
import { prepareGrantRolesTransaction } from '@/features/roles/helpers/grantRoles'
import type { ActiveTransactionState } from './hooks/useActiveTransactionState'
import { closeTransactionModal } from './hooks/useTransactionModal'
import type { Transaction } from './types'
import { getActiveTransaction } from './utils/getActiveTransaction'

const ALICE = '0x1111111111111111111111111111111111111111' as Address
const ATTACKER = '0x2222222222222222222222222222222222222222' as Address
const TRUSTED = '0x3333333333333333333333333333333333333333' as Address
const REGISTRY = '0x4444444444444444444444444444444444444444' as Address
const FIXED_ID = 'tx-grant-roles'
const grantRolesAbi = parseAbi([
  'function grantRoles(uint256 resource, uint256 roleBitmap, address account)',
])

describe('F-08 PoC: a fixed transaction id binds new UI text to old calldata', () => {
  beforeEach(() => transactionManager.clear())
  afterEach(() => transactionManager.clear())

  it('shows the trusted recipient while the live actor still grants the attacker', async () => {
    // Keep the wallet request pending, exactly as when the extension prompt is
    // open and the user returns to the Portal tab.
    const walletPrompt = new Promise<Hash>(() => {})
    const request = vi.fn(
      ({ method }: { method: string }) =>
        (method === 'eth_chainId'
          ? Promise.resolve('0xaa36a7')
          : method === 'eth_sendTransaction'
            ? walletPrompt
            : Promise.reject(new Error(`unexpected RPC method: ${method}`))) as Promise<unknown>,
    )

    const walletClient = {
      account: { address: ALICE },
      chain: sepolia,
      request,
    } as unknown as WalletClient
    const signer: EOASigner = {
      type: 'eoa',
      walletClient,
    }

    // Drive the real Portal builder, including its ENS label resource and role
    // bitmap encoding. No handwritten calldata is used by the PoC.
    const attackerIntent = prepareGrantRolesTransaction({
      name: 'valuable.eth',
      account: ATTACKER,
      roles: ['ROLE_SET_RESOLVER'],
      walletClient,
      chainId: 11155111,
      registryAddress: REGISTRY,
    })
    const attackerGrant = attackerIntent.request

    transactionManager.startTransaction(attackerIntent, signer, {
      id: FIXED_ID,
      publicClient: {} as PublicClient,
      chainId: 11155111,
    })

    await vi.waitFor(() =>
      expect(
        request.mock.calls.some(([args]) => args.method === 'eth_sendTransaction'),
      ).toBe(true),
    )

    closeTransactionModal()
    expect(transactionManager.getTransaction(FIXED_ID)).toBeDefined()

    // Closing the transaction modal/sheet does not clear an in-flight actor.
    // The reopened form constructs a fresh descriptor under the same fixed ID.
    const trustedDescriptor: Transaction = {
      id: FIXED_ID,
      title: 'Grant roles',
      transactionName: `Grant roles for ${TRUSTED}`,
      onStart: vi.fn(),
      onDone: vi.fn(),
    }
    const oldActor = transactionManager.getTransaction(FIXED_ID)
    expect(oldActor).toBeDefined()

    const uiState: ActiveTransactionState = {
      txId: FIXED_ID,
      machineState: 'submitting',
      hash: undefined,
      error: undefined,
    }
    const displayed = getActiveTransaction([trustedDescriptor], uiState)

    // What Portal renders after the form is reopened.
    expect(displayed.transactionName).toContain(TRUSTED)

    // What the still-pending production actor will submit if the user accepts
    // the already-open wallet request.
    expect(oldActor?.getSnapshot().context.request).toEqual(attackerGrant)
    const decoded = decodeFunctionData({
      abi: grantRolesAbi,
      data: attackerGrant.type === 'eoa' ? attackerGrant.data! : '0x',
    })
    expect(decoded.functionName).toBe('grantRoles')
    expect(decoded.args?.[2]).toBe(ATTACKER)
    console.log('[F-08] fixed-id descriptor/actor split', {
      transactionId: FIXED_ID,
      displayedTransaction: displayed.transactionName,
      decodedCalldataRecipient: decoded.args?.[2],
      actorState: oldActor?.getSnapshot().value,
    })
    const sendCall = request.mock.calls.find(
      ([args]) => args.method === 'eth_sendTransaction',
    )?.[0] as { params?: readonly [{ from?: string; to?: string; data?: string }] }
    expect(sendCall.params?.[0]).toMatchObject({
      from: ALICE,
      to: REGISTRY,
      data: attackerGrant.data,
    })
    expect(sendCall.params?.[0]?.data).toContain(ATTACKER.slice(2))
  })
})
```

#### Run command

From the repository root:

```bash
pnpm --filter=portal exec vitest run src/features/transaction-manager/F08MutableDescriptorCollision.poc.test.ts --reporter=verbose
```

#### Execution result

```
RUN  v4.1.5 apps/portal

📊 [TRANSACTION MANAGER] Transaction tx-grant-roles state: submitting

[F-08] fixed-id descriptor/actor split {
  transactionId: 'tx-grant-roles',
  displayedTransaction: 'Grant roles for 0x3333333333333333333333333333333333333333',
  decodedCalldataRecipient: '0x2222222222222222222222222222222222222222',
  actorState: 'submitting'
}

✓ F08MutableDescriptorCollision.poc.test.ts > F-08 PoC: a fixed transaction id binds new UI text to old calldata > shows the trusted recipient while the live actor still grants the attacker 58ms

Test Files  1 passed (1)
Tests       1 passed (1)
Duration    1.18s
```

The decisive result is that the same active transaction is presented as granting roles to the trusted address, while ABI decoding the request already sent to the wallet yields `0x2222…2222` as the actual `account` argument. Production truncates that trusted address in `transactionName`; the PoC uses the full address so the mismatch is unambiguous in the assertion.
