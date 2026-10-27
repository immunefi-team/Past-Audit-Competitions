# 92806 w a critical resolver role removal silently leaves role set data authority on chain

**Submitted on Sep 12th 2026 at 12:33:35 UTC by @pks271 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92806
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Bug Description

The Portal's resolver role UI builds "Remove user" transactions from a hand-written role table that does not include `ROLE_SET_DATA`. The write codec used for the actual transaction does support `ROLE_SET_DATA`, so the application can grant or observe that authority on-chain, but its removal path cannot represent the same bit once it decodes a role bitmap for display.

The local Portal table contains the visible resolver roles and their admin variants, but it omits `ROLE_SET_DATA`:

```typescript
// apps/portal/src/lib/roles/resolverRoles.ts
export const resolverRoles = {
  ROLE_SET_ADDR: 1n << 0n,
  ROLE_SET_TEXT: 1n << 4n,
  ROLE_SET_CONTENTHASH: 1n << 8n,
  ROLE_SET_PUBKEY: 1n << 12n,
  ROLE_SET_ABI: 1n << 16n,
  ROLE_SET_INTERFACE: 1n << 20n,
  ROLE_SET_NAME: 1n << 24n,
  ROLE_SET_ALIAS: 1n << 28n,
  ROLE_CLEAR: 1n << 32n,
  ROLE_UPGRADE: 1n << 124n,
} as const
```

`decodeResolverRoleBitmap` iterates only that local table. A bitmap containing `ROLE_SET_ADDR | ROLE_SET_DATA` is decoded as only `ROLE_SET_ADDR`.

```typescript
// apps/portal/src/lib/roles/resolverRoles.ts
for (const [roleName, roleValue] of Object.entries(resolverRoles)) {
  if ((bitmapValue & roleValue) !== 0n) {
    roles.push(roleName as ResolverRoleKey)
  }
}
```

The Portal then passes this decoded list directly into the Remove user transaction:

```typescript
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx
setPendingAction({
  type: 'remove',
  name: roleName,
  account: selectedAccount,
  roles: decodedRoles,
})
```

The transaction helper forwards that list into `revokeResolverRolesWriteParameters`:

```typescript
// apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts
const writeParams = revokeResolverRolesWriteParameters(walletClient, {
  resolverAddress,
  targetAccount: account,
  scope: name === '' ? 'root' : 'name',
  name,
  roles,
})
```

The ensjs codec used by that helper can encode `ROLE_SET_DATA`; the loss happens earlier in the Portal's display/decode model:

```typescript
// apps/portal/node_modules/@ensdomains/ensjs/src/utils/v2/roles/resolverRoles.ts
export const RESOLVER_ROLE_SET_DATA = 1n << 36n

const resolverRoleMap: Record<ResolverRole, bigint> = {
  ROLE_SET_DATA: RESOLVER_ROLE_SET_DATA,
}
```

The confirmation dialog tells the resolver administrator they are removing the user from all roles:

```typescript
// apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx
Are you sure you want to remove this user from all roles?
```

That statement is false for accounts that hold `ROLE_SET_DATA` together with any visible resolver role. The signed transaction revokes only the visible roles and leaves `ROLE_SET_DATA` intact.

## Impact

A resolver administrator can remove a user through the official Portal UI and receive a successful transaction, while the removed account retains `ROLE_SET_DATA` on the resolver. The retained authority lets that account continue calling `setData(bytes32 node, string key, bytes value)` for the scoped resolver resource.

This is a concrete authorization failure, not a display-only mismatch. The application constructs an incomplete revoke transaction from a lossy model, the on-chain role remains set after the transaction, and the UI communicates that the removal covered all roles.

## Recommendation

Use one canonical resolver-role model for display, editing, and transaction construction. When removing a user, derive the revoke bitmap from the exact on-chain or indexed role bitmap, not from a lossy decoded display list. If an unknown role bit is present, fail closed and show that the Portal cannot safely remove the user until the role model is updated.

## Proof of Concept

## Proof of Concept

File:

```
portal/src/lib/roles/resolver-role-set-data-onchain.test.ts
```

The test starts from the application's real role decoder, uses the ensjs write helper that the Portal uses for the Remove user transaction, deploys a real PermissionedResolver proxy on a local Sepolia fork, grants `ROLE_SET_ADDR | ROLE_SET_DATA`, submits the app-built Remove transaction, and reads the chain afterward.

Expected result: the visible `ROLE_SET_ADDR` authority is removed, while `ROLE_SET_DATA` remains assigned to the same account.

Start a local Sepolia fork:

```bash
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --port 8555
```

Run the proof from the repository root:

```bash
cd apps/portal
pnpm --filter portal exec vitest run src/lib/roles/resolver-role-set-data-onchain.test.ts
```

code:

```typescript
/**
 * Full-chain PoC: SET_DATA survives the Portal's "Remove user".
 *
 * App-code path, no doubles: the Portal's own resolverRoles table lacks
 *   ROLE_SET_DATA (bit 36). decodeResolverRoleBitmap(SET_ADDR|SET_DATA)
 *   returns ['ROLE_SET_ADDR'] only, and the ensjs encoder used by the Remove
 *   flow encodes exactly that lossy list.
 * On-chain path, local Sepolia fork with bytecode-verified contracts:
 *   deploy a real PermissionedResolver proxy via the deployed
 *   VerifiableFactory, grant SET_ADDR|SET_DATA on-chain, submit the app-built
 *   Remove transaction, read the chain: SET_ADDR revoked, SET_DATA RETAINED
 *   after the UI-reported-complete removal.
 *
 * Contract provenance (verified 2026-09-12, artifact-template vs live with
 * masked immutables, byte-identical): PermissionedResolverImpl
 * 0x9eae5c2730a7dd16bdd1dee6421a1b91e3b0365e, VerifiableFactory
 * 0x10dc6333cdfe1fcef624c6e0a8221b91804cd7ef.
 * Run with a fork up:
 * anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --port 8555
 * Fails hard when the fork is not reachable.
 */
import { beforeAll, describe, expect, it } from 'vitest'
import {
  grantResolverRolesWriteParameters,
  revokeResolverRolesWriteParameters,
} from '@ensdomains/ensjs/wallet/v2'
import {
  createPublicClient,
  createWalletClient,
  encodeFunctionData,
  http,
  parseAbi,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import {
  decodeResolverRoleBitmap,
  resolverPermissions,
  resolverRoles,
} from './resolverRoles'

const FORK = 'http://127.0.0.1:8555'
const FACTORY = '0x10dc6333cdfe1fcef624c6e0a8221b91804cd7ef'
const RESOLVER_IMPL = '0x9eae5c2730a7dd16bdd1dee6421a1b91e3b0365e'
const INIT_BITMAP = BigInt(
  '0x1111111111111111111111111111111111111111111111111111111111111111',
)
const SET_ADDR = 1n << 0n
const SET_DATA = 1n << 36n
const VICTIM = '0x2222222222222222222222222222222222222222'
const PROXY_DEPLOYED_TOPIC =
  '0xbc7cd75a20ee27fd9adebab32041f755214dbc6bffa90cc0225b39da2e5c2d3b'

// A deterministic Anvil account is re-funded so the test is fork-state-proof.
const DEPLOYER_KEY =
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b7b7853b7'

const resolverAbi = parseAbi([
  'function initialize(address owner, uint256 bitmap, bytes[] setters)',
  'function grantRootRoles(uint256 roleBitmap, address account) returns (bool)',
  'function hasRootRoles(uint256 roleBitmap, address account) returns (bool)',
  'function revokeRootRoles(uint256 roleBitmap, address account) returns (bool)',
])
const factoryAbi = parseAbi([
  'function deployProxy(address implementation, uint256 salt, bytes data) returns (address)',
])

let forkUp = false
let publicClient: ReturnType<typeof createPublicClient>
let walletClient: ReturnType<typeof createWalletClient>
const deployer = privateKeyToAccount(DEPLOYER_KEY)

const send = (to: `0x${string}`, data: `0x${string}`) =>
  walletClient.sendTransaction({ account: deployer, chain: sepolia, to, data })

beforeAll(async () => {
  try {
    publicClient = createPublicClient({ chain: sepolia, transport: http(FORK) })
    await publicClient.getBlockNumber()
    forkUp = true
  } catch (e) {
    throw new Error(
      `[SET_DATA_POC] fork not reachable at ${FORK}; start: anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --port 8555 (${String(e)})`,
    )
  }
  walletClient = createWalletClient({
    account: deployer,
    chain: sepolia,
    transport: http(FORK),
  })
  await publicClient.transport.request({
    method: 'anvil_setBalance',
    params: [deployer.address, '0x21E19E0C9BAB2400000'],
  })
}, 60_000)

describe('full chain: SET_DATA survives Remove user', () => {
  it('portal decode is lossy, ensjs encodes the loss', () => {
    expect(Object.keys(resolverRoles).some((k) => k.includes('SET_DATA'))).toBe(
      false,
    )
    expect(resolverPermissions).toHaveLength(10)

    const onChainBitmap = SET_DATA | SET_ADDR
    expect(decodeResolverRoleBitmap(onChainBitmap)).toEqual(['ROLE_SET_ADDR'])

    // ensjs can encode SET_DATA. The codec supports what the table omits.
    const grant = grantResolverRolesWriteParameters(walletClient as never, {
      resolverAddress: RESOLVER_IMPL,
      targetAccount: VICTIM,
      scope: 'root',
      roles: ['ROLE_SET_DATA'],
    })
    expect(grant.functionName).toBe('grantRootRoles')
    expect(grant.args[0]).toBe(SET_DATA)
  })

  it(
    'on-chain grant, app-built Remove, SET_DATA retained',
    { timeout: 120_000 },
    async () => {
      // Deploy a real PermissionedResolver proxy through the live factory.
      const initData = encodeFunctionData({
        abi: resolverAbi,
        functionName: 'initialize',
        args: [deployer.address, INIT_BITMAP, []],
      })
      const deployHash = await send(
        FACTORY,
        encodeFunctionData({
          abi: factoryAbi,
          functionName: 'deployProxy',
          args: [RESOLVER_IMPL, BigInt(Date.now()), initData],
        }),
      )
      const deployReceipt = await publicClient.waitForTransactionReceipt({
        hash: deployHash,
      })
      const proxyLog = deployReceipt.logs.find(
        (l) =>
          l.topics[0] === PROXY_DEPLOYED_TOPIC &&
          l.topics[1] ===
            `0x000000000000000000000000${RESOLVER_IMPL.slice(2).toLowerCase()}`,
      )
      expect(proxyLog).toBeTruthy()
      const resolver = proxyLog!.address

      // On-chain premise: the victim holds SET_ADDR | SET_DATA at root.
      const grantHash = await send(
        resolver,
        encodeFunctionData({
          abi: resolverAbi,
          functionName: 'grantRootRoles',
          args: [SET_ADDR | SET_DATA, VICTIM],
        }),
      )
      await publicClient.waitForTransactionReceipt({ hash: grantHash })

      const hasBoth = await publicClient.readContract({
        address: resolver,
        abi: resolverAbi,
        functionName: 'hasRootRoles',
        args: [SET_ADDR | SET_DATA, VICTIM],
      })
      expect(hasBoth).toBe(true)

      // The app's Remove path verbatim: portal decode -> ensjs encode.
      const removeUserRoles = decodeResolverRoleBitmap(SET_ADDR | SET_DATA)
      const revoke = revokeResolverRolesWriteParameters(walletClient as never, {
        resolverAddress: resolver,
        targetAccount: VICTIM,
        scope: 'root',
        roles: removeUserRoles as never,
      })
      const revokeBitmap = revoke.args[0] as bigint
      expect(revokeBitmap & SET_ADDR).not.toBe(0n)
      expect(revokeBitmap & SET_DATA).toBe(0n)

      // Submit the app-built transaction unmodified.
      const revokeHash = await send(
        revoke.address,
        encodeFunctionData({
          abi: revoke.abi,
          functionName: revoke.functionName,
          // biome-ignore lint/suspicious/noExplicitAny: ensjs args boundary
          args: revoke.args as any,
        }),
      )
      await publicClient.waitForTransactionReceipt({ hash: revokeHash })

      // On-chain outcome: visible role revoked, SET_DATA retained.
      const hasSetAddr = await publicClient.readContract({
        address: resolver,
        abi: resolverAbi,
        functionName: 'hasRootRoles',
        args: [SET_ADDR, VICTIM],
      })
      expect(hasSetAddr).toBe(false)

      const hasSetData = await publicClient.readContract({
        address: resolver,
        abi: resolverAbi,
        functionName: 'hasRootRoles',
        args: [SET_DATA, VICTIM],
      })
      expect(hasSetData).toBe(true)
    },
  )
})
```

Result:

```
 RUN  v4.1.5 /home/dev/sec/immunefi/comps/ens-comp/repo/apps/portal

 ✓ src/lib/roles/resolver-role-set-data-onchain.test.ts (2 tests) 2596ms
   ✓ full chain: SET_DATA survives Remove user (2)
     ✓ portal decode is lossy, ensjs encodes the loss 1ms
     ✓ on-chain grant, app-built Remove, SET_DATA retained  2579ms

 Test Files  1 passed (1)
      Tests  2 passed (2)
```
