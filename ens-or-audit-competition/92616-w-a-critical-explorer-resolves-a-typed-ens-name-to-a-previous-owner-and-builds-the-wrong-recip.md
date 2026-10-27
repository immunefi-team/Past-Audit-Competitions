# 92616 w a critical explorer resolves a typed ens name to a previous owner and builds the wrong recipient transaction

**Submitted on Sep 10th 2026 at 23:27:17 UTC by @v4rvl for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92616
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

Every "ENS name or address" field in the Explorer resolves what the user typed through one shared helper. When the universal resolver returns no address for the name, that helper does not report the name as unresolvable. It reads the name's owner instead and hands that address to the flow as if it were the resolution.

For an unwrapped ENSv1 `.eth` name the value it reads is not the party who holds the name. ENSv1 splits a name into an ERC-721 registrant, who owns the asset, and a registry owner, who controls its records. An ERC-721 sale moves only the registrant, so after every secondary-market sale in which the buyer has not called `reclaim()`, the registry-owner slot still belongs to the seller. The helper keeps that slot and discards the registrant.

Measured on a local Sepolia fork with a name registered to a seller and then sold to a buyer by ERC-721 transfer: the Explorer's own resolution query answers `0xBAd0000000000000000000000000000000000Bad`, the seller, for a name whose registrant is `0xb0B0000000000000000000000000000000000B0B`, the buyer. The transfer form then renders the seller as the recipient with no warning, the roles Add-user sheet prints `Resolved: 0xBAd0…0Bad`, and the app's own grant-roles builder encodes the seller as the grantee. The seller is not passive here: the same run measures that the seller still holds live ENSv1 write authority over the sold name, and uses it to clear the resolver, which is what makes the universal resolver return nothing in the first place.

Evidence spine, measured with a matched control:

|                                                  | attack arm                                                                        | control arm                                  |
| ------------------------------------------------ | --------------------------------------------------------------------------------- | -------------------------------------------- |
| ERC-721 registrant (holder)                      | `0xb0B0…0B0B` buyer                                                               | `0xb0B0…0B0B` buyer                          |
| ENSv1 registry owner                             | `0xBAd0…0Bad` seller                                                              | `0xb0B0…0B0B` buyer                          |
| ENSv1 resolver                                   | `0x0000…0000`                                                                     | `0x0000…0000`                                |
| `getEnsAddress` via the app's universal resolver | `null`                                                                            | `null`                                       |
| ensjs `getOwner` v1                              | `{"registrant":"0xb0B0…0B0B","owner":"0xBAd0…0Bad","ownershipLevel":"registrar"}` | registrant and owner both `0xb0B0…0B0B`      |
| app `getResolvedAddress`                         | `0xBAd0…0Bad` seller                                                              | `0xb0B0…0B0B` buyer                          |
| transfer form recipient preview                  | `0xBAd0000000000000000000000000000000000Bad`                                      | `0xb0B0000000000000000000000000000000000B0B` |
| refusal messages rendered                        | 0                                                                                 | not applicable                               |
| grant-roles calldata grantee                     | `0xBAd0…0Bad`                                                                     | not run                                      |

`16777216` in the calldata is `1 << 24`, the `ROLE_SET_RESOLVER` bit; the Explorer labels that role "Can change the resolver addresses" (`apps/portal/src/lib/roles/permissions.ts:31-33`), and the Manager declares the same constant at `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9`.

Boundary of the proof: the run measures the resolution, what the two containers render, and the calldata the app's own builder produces from that resolution. It does not broadcast the transfer or the grant, because that needs an ENSv2 name the connected account already owns, and no such name exists on this fork.

## Who loses what

Three parties, one action.

* **The Explorer user** is the party who types the name. They intend to hand a name or a role to whoever holds `foo.eth`. The app tells them the name resolved and shows them a truncated hex, and the transaction it prepares targets someone else. In the transfer flow, the real transfer helper would place the field's address in the `safeTransferFrom` recipient argument (`apps/portal/src/features/transfer/helpers/transferToken.ts:67-71`). In the roles flow, the real grant builder encodes that address as the grantee. This proof stops at the wallet request and does not claim that either transaction was broadcast.
* **The intended recipient**, the ERC-721 registrant, receives nothing. Measured: registrant `0xb0B0000000000000000000000000000000000B0B` appears in zero rows of the rendered transfer form after the user types the name that belongs to them.
* **The previous owner** is selected as the recipient even though they are not the current ERC-721 registrant. The seller's retained ENSv1 registry slot explains why the fallback returns that address, but it does not establish entitlement to receive a different user's ENSv2 name or its resolver role.

The previous owner can also cause the condition rather than wait for it. Measured in the same run: with the ERC-721 already transferred to the buyer, `ENS.setResolver(node, 0x0)` sent by the seller returned `receipt.status=success`, and the app's answer for that name was the seller both before and after. Once the buyer calls `reclaim()` the same write returns `receipt.status=reverted`.

## Root cause

`apps/portal/src/features/roles/helpers/addUser.handlers.ts:19-39` is the shared resolver behind every "ENS name or address" field in the app:

```ts
  try {
    const resolved = await getEnsAddress(client, {
      name: nameOrAddress,
      universalResolverAddress,
    })

    let resolvedAddress = resolved

    // Fallback for names that do not set an address record:
    // use current ENS owner address so the role can still be granted.
    if (!resolvedAddress) {
      const ownerResult = await getEnsOwner({ name: nameOrAddress })
      if (ownerResult.isOk()) {
        resolvedAddress = ownerResult.value?.owner ?? null
      }
    }

    return resolvedAddress
  } catch {
    return null
  }
```

The comment above the branch is the design intent and it has to be answered directly. It permits substituting "the current ENS owner" when a name sets no address record. For an unwrapped ENSv1 `.eth` name, the field this code reads is not the current owner of the name. `apps/portal/src/utils/ens/resolveEnsOwner.ts:95-102` is where the value comes from:

```ts
  const v1Owner = await getOwnerV1(client, { name })
  if (v1Owner?.owner && v1Owner.owner !== zeroAddress) {
    return {
      owner: v1Owner.owner,
      registryAddress: v1EthRegistry,
      protocolVersion: 'ENSv1',
    }
  }
```

`getOwnerV1` returns both fields for this name shape and labels them at the return site in `@ensdomains/ensjs/src/actions/public/v1/getOwner.ts:216-225`:

```ts
    // if there is a registrar owner, then it's not a subdomain but we have also passed the namewrapper clause
    // this means that it's an unwrapped second-level name
    // the registrant is the owner of the NFT
    // the owner is the controller of the records
    if (registrarOwner) {
      return {
        registrant: registrarOwner,
        owner: registryOwner,
        ownershipLevel: 'registrar',
      } as GetOwnerReturnType<contract>
```

`resolveEnsOwner` keeps `owner`, the controller of the records, and drops `registrant`, the owner of the NFT. So the "current ENS owner" the comment promises is, for exactly this name shape, the party who controls the records rather than the party who owns the name. The two are the same address only until the name changes hands.

The substitution also erases a state the app declares and no caller can then reach. `apps/portal/src/features/address/hooks/useAddressResolution.ts:19-20` declares:

```ts
  /** Valid input, but no address could be resolved for it. */
  | 'unresolved'
```

and `apps/portal/src/features/address/components/AddressNameInput.tsx:70-74` renders that state as a refusal:

```tsx
        .with('unresolved', () => (
          <p className="text-sm mt-1.5 text-danger">
            Could not resolve an address for “{value.trim()}”
          </p>
        ))
```

Because the fallback always produces an address for a registered ENSv1 name, that message never appears for one. What the field prints instead is `AddressNameInput.tsx:62-68`, `Resolved: ${truncateAddress(address, 6, 4)}`.

Five call sites consume the result, and each of them gates only on the address being non-null:

* `apps/portal/src/features/transfer/components/SendNameForm.tsx:87-89` `const hasValidRecipient = !!recipient && !isSelf && !isZeroAddress`
* `apps/portal/src/features/roles/components/RolesAddUserSheet.tsx:99-100` `const canSave = !!address && !isResolving && selectedRoles.size > 0 && !isSuccess`
* `apps/portal/src/features/registry/components/v2/RegistryAddUserSheet.tsx:100-101`, the same expression
* `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:183-184`, the same expression
* `apps/portal/src/routes/$name/create-subname.tsx:109-110`, which feeds `ownerAddress` straight into the new subname's owner at `:164`

`SendNameForm.tsx:88` is the sibling that shows the app already knows a resolution can be unusable: it rejects the zero address on the very same value. There is no comparable check that the address the fallback invented is the one the name actually points at.

## Steps to reproduce

1. Start an Anvil Sepolia fork on `127.0.0.1:8546` with `--auto-impersonate`. Nothing in these steps touches a public network.
2. Give yourself an ENSv1 BaseRegistrar controller: read `BaseRegistrar.owner()` (`0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85`), fund it, and call `addController` for a clean address.
3. Register a fresh `.eth` label to the seller `0xBAd0000000000000000000000000000000000Bad` with `register(labelhash, seller, 31536000)`.
4. Simulate a secondary-market sale: as the seller, call `BaseRegistrar.transferFrom(seller, buyer, labelhash)` with buyer `0xb0B0000000000000000000000000000000000B0B`. Do not call `reclaim()`. The ERC-721 registrant is now the buyer; `ENS.owner(namehash)` is still the seller.
5. Optionally, as the seller, call `ENS.setResolver(namehash, 0x0)`. This succeeds, which is the measurement that the seller still holds write authority over a name they no longer own.
6. Open the Explorer's transfer page for any ENSv2 name you own, or the roles Add-user sheet, and type the sold name into the recipient field.
7. Read the address the app resolves it to. It is the seller.
8. Control: as the buyer, call `BaseRegistrar.reclaim(labelhash, buyer)` and repeat step 6. The app now resolves the name to the buyer.

The proof of concept below performs steps 2 to 8 itself, with a per-run label suffix, so it is repeatable and does not depend on state left behind by another run. Every name, sale and record it needs is minted on the local fork by the test.

**What is real and what is replaced.** The resolution under test is production code: the test imports `getResolvedAddress` from `@/features/address/queries/getResolvedAddress` unmodified, and it reads the chain through a viem client bound to the local fork, so the registrant and registry-owner values come from real contract state rather than from the harness. Three modules are replaced, none of which decides an asserted value:

* `@/lib/wagmi/helpers` returns the fork client from `safeGetClient`. It selects which chain is read, not what that chain answers.
* `wagmi` is partially mocked over `importOriginal` to report a connected account. A wallet connection is an external dependency, and the connected address is the victim rather than either party in the measurement.
* `@tanstack/react-router` is partially mocked over `importOriginal` so `useNavigate` resolves, because `useTransferName` calls it at hook level and this harness runs no router. It touches no address.

The reported registrant, registry owner, resolved address and `grantRoles` calldata are all produced downstream of those three, by unmodified application code.

## Proof of Concept

## Proof of concept

`apps/portal/src/__poc_w1305__/w1305_recipient_resolution.poc.test.tsx`

```tsx
/**
 * LANE-W1305
 *
 * Question: when the Explorer's shared "ENS name or address" recipient field is
 * given an ENSv1 `.eth` name, what address does the app hand to the flow that
 * follows?
 *
 * Everything that decides the answer is real: the production query function
 * (`getResolvedAddress`), the production resolver (`resolveAddressOrName`), the
 * production hook (`useAddressResolution`), the production field
 * (`AddressNameInput`), the production form (`SendNameForm`), the production
 * sheet (`RolesAddUserSheet`) and the production calldata builder
 * (`prepareGrantRolesTransaction`). Chain state is created, read and mutated on
 * a local Sepolia fork on 127.0.0.1; every other host is refused by the guard
 * installed below. Only the wallet connection and the router are replaced.
 *
 * The fixtures are provisioned by this file with a per-run label suffix, so the
 * run is repeatable and consumes nothing.
 */
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { getOwner as getOwnerV1 } from '@ensdomains/ensjs/public/v1'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { ok } from 'neverthrow'
import type { ReactNode } from 'react'
import {
  createPublicClient,
  createWalletClient,
  decodeFunctionData,
  getAddress,
  http,
  keccak256,
  namehash,
  parseAbi,
  toHex,
  zeroAddress,
} from 'viem'
import { getEnsAddress } from 'viem/actions'
import { sepolia } from 'viem/chains'
import { beforeAll, describe, expect, it, vi } from 'vitest'
import { createConfig, WagmiProvider } from 'wagmi'

const FORK = 'http://127.0.0.1:8546'

// Local-only guard. Anything the app tries to fetch off 127.0.0.1 is refused
// and counted, so the run cannot silently touch a public endpoint.
const blockedHosts: string[] = []
const realFetch = globalThis.fetch
globalThis.fetch = ((input: RequestInfo | URL, init?: RequestInit) => {
  const url = typeof input === 'string' ? input : input.toString()
  if (!url.startsWith('http://127.0.0.1:8546')) {
    blockedHosts.push(new URL(url).host)
    return Promise.reject(new Error(`blocked non-local fetch: ${url}`))
  }
  return realFetch(input as RequestInfo, init)
}) as typeof fetch

const forkChain = extendChainWithEns(sepolia)
const forkClient = createPublicClient({
  chain: forkChain,
  transport: http(FORK),
})

/** ENSv1 registry. `owner(node)` is the "manager"/controller slot. */
const V1_REGISTRY = '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e' as const
/** ENSv1 BaseRegistrar. `ownerOf(labelhash)` is the ERC-721 registrant. */
const V1_BASE_REGISTRAR = '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85' as const
/** An ENSv1 resolver already deployed on Sepolia. */
const V1_RESOLVER = '0x8FADE66B79cC9f707aB26799354482EB93a5B7dD' as const
/** The V2 root ETHRegistry, used only as the `registryAddress` prop. */
const V2_ETH_REGISTRY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2' as const

/** Sold the ERC-721. Kept the ENSv1 registry-owner slot. */
const SELLER = getAddress('0xbad0000000000000000000000000000000000bad')
/** Bought the ERC-721. Holds the name. */
const BUYER = getAddress('0xb0b0000000000000000000000000000000000b0b')
/** The Explorer user who types a name into a recipient field. */
const USER = getAddress('0x1111111111111111111111111111111111111111')
/** A registrar controller this file grants itself, to mint the fixtures. */
const MINTER = getAddress('0xc0ffee0000000000000000000000000000000001')

const SUFFIX = String(Date.now()).slice(-8)
/** Sold, v1 resolver still set and its addr record naming the holder. */
const SOLD_WITH_RECORD = `w1305r${SUFFIX}.eth`
/** Sold, no resolver. */
const SOLD_NO_RECORD = `w1305n${SUFFIX}.eth`
/** Sold, no resolver, and the holder reclaimed the registry-owner slot. */
const RECLAIMED = `w1305c${SUFFIX}.eth`

const registryAbi = parseAbi([
  'function owner(bytes32 node) view returns (address)',
  'function resolver(bytes32 node) view returns (address)',
  'function setResolver(bytes32 node, address resolver)',
])
const registrarAbi = parseAbi([
  'function owner() view returns (address)',
  'function ownerOf(uint256 id) view returns (address)',
  'function controllers(address) view returns (bool)',
  'function addController(address controller)',
  'function register(uint256 id, address owner, uint256 duration) returns (uint256)',
  'function transferFrom(address from, address to, uint256 id)',
  'function reclaim(uint256 id, address owner)',
])
const resolverAbi = parseAbi([
  'function addr(bytes32 node) view returns (address)',
  'function setAddr(bytes32 node, address a)',
])

// `apps/portal/src/main.tsx:64-68` installs this at app startup. Without it a
// mounted component throws inside react-query's `hashKey` and renders nothing.
;(BigInt.prototype as unknown as { toJSON: () => string }).toJSON = function (
  this: bigint,
) {
  return this.toString()
}

vi.mock('@/lib/wagmi/helpers', () => ({
  safeGetClient: () => ok(forkClient),
  safeGetConnectorClient: () => {
    throw new Error('not used')
  },
}))

// The wallet connection is an external dependency; it decides nothing here.
vi.mock('wagmi', async (importOriginal) => {
  const actual = (await importOriginal()) as Record<string, unknown>
  return {
    ...actual,
    useConnection: () => ({
      address: USER,
      isConnected: true,
      chain: forkChain,
      chainId: forkChain.id,
    }),
    useAccount: () => ({ address: USER, chainId: forkChain.id }),
    useWalletClient: () => ({
      data: { account: { address: USER }, chain: forkChain },
    }),
    usePublicClient: () => forkClient,
  }
})

// Routing is an external dependency; `useTransferName` calls `useNavigate` at
// hook level, and this harness runs no router.
vi.mock('@tanstack/react-router', async (importOriginal) => {
  const actual = (await importOriginal()) as Record<string, unknown>
  return { ...actual, useNavigate: () => () => undefined }
})

const { getResolvedAddress } = await import(
  '@/features/address/queries/getResolvedAddress'
)
const { resolveAddressOrName } = await import(
  '@/features/roles/helpers/addUser.handlers'
)
const { prepareGrantRolesTransaction } = await import(
  '@/features/roles/helpers/grantRoles'
)
const { universalResolverAddress } = await import(
  '@/lib/constants/universalResolver'
)
const { SendNameForm } = await import(
  '@/features/transfer/components/SendNameForm'
)
const { RolesAddUserSheet } = await import(
  '@/features/roles/components/RolesAddUserSheet'
)
const { TransactionManagerProvider } = await import(
  '@ens-apps/transaction-manager'
)

const localConfig = createConfig({
  chains: [forkChain],
  transports: { [forkChain.id]: http(FORK) },
})

const Providers = ({ children }: { children: ReactNode }) => (
  <WagmiProvider config={localConfig} reconnectOnMount={false}>
    <QueryClientProvider
      client={
        new QueryClient({
          defaultOptions: { queries: { retry: false, gcTime: 0 } },
        })
      }
    >
      <TransactionManagerProvider publicClient={forkClient as never}>
        {children}
      </TransactionManagerProvider>
    </QueryClientProvider>
  </WagmiProvider>
)

const walletFor = (account: `0x${string}`) =>
  createWalletClient({ account, chain: forkChain, transport: http(FORK) })

const send = async (
  account: `0x${string}`,
  params: Parameters<ReturnType<typeof walletFor>['writeContract']>[0],
) => {
  const hash = await walletFor(account).writeContract(params)
  return forkClient.waitForTransactionReceipt({ hash })
}

const labelhashOf = (name: string) =>
  BigInt(keccak256(toHex(name.split('.')[0])))

/** The app's own answer for a typed recipient string. */
const appAnswerFor = async (name: string) => {
  const result = await getResolvedAddress({ nameOrAddress: name })
  return result.isOk() ? String(result.value) : `ERR ${String(result.error)}`
}

const chainState = async (name: string) => {
  const node = namehash(name)
  const [registryOwner, resolver, registrant] = await Promise.all([
    forkClient.readContract({
      address: V1_REGISTRY,
      abi: registryAbi,
      functionName: 'owner',
      args: [node],
    }),
    forkClient.readContract({
      address: V1_REGISTRY,
      abi: registryAbi,
      functionName: 'resolver',
      args: [node],
    }),
    forkClient.readContract({
      address: V1_BASE_REGISTRAR,
      abi: registrarAbi,
      functionName: 'ownerOf',
      args: [labelhashOf(name)],
    }),
  ])
  return { registryOwner, resolver, registrant }
}

beforeAll(async () => {
  expect(await forkClient.getChainId()).toBe(11155111)

  const registrarOwner = await forkClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: registrarAbi,
    functionName: 'owner',
  })
  for (const a of [registrarOwner, MINTER, SELLER, BUYER, USER]) {
    await forkClient.request({
      method: 'anvil_setBalance' as never,
      params: [a, '0x56BC75E2D63100000'] as never,
    })
  }
  const isMinter = await forkClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: registrarAbi,
    functionName: 'controllers',
    args: [MINTER],
  })
  if (!isMinter) {
    await send(registrarOwner, {
      address: V1_BASE_REGISTRAR,
      abi: registrarAbi,
      functionName: 'addController',
      args: [MINTER],
      chain: forkChain,
      account: registrarOwner,
    })
  }

  // Every fixture is registered to the SELLER, then sold to the BUYER by an
  // ERC-721 transfer, which is what a secondary-market sale is.
  for (const name of [SOLD_WITH_RECORD, SOLD_NO_RECORD, RECLAIMED]) {
    await send(MINTER, {
      address: V1_BASE_REGISTRAR,
      abi: registrarAbi,
      functionName: 'register',
      args: [labelhashOf(name), SELLER, 31_536_000n],
      chain: forkChain,
      account: MINTER,
    })
    if (name === SOLD_WITH_RECORD) {
      await send(SELLER, {
        address: V1_REGISTRY,
        abi: registryAbi,
        functionName: 'setResolver',
        args: [namehash(name), V1_RESOLVER],
        chain: forkChain,
        account: SELLER,
      })
      await send(SELLER, {
        address: V1_RESOLVER,
        abi: resolverAbi,
        functionName: 'setAddr',
        args: [namehash(name), BUYER],
        chain: forkChain,
        account: SELLER,
      })
    }
    await send(SELLER, {
      address: V1_BASE_REGISTRAR,
      abi: registrarAbi,
      functionName: 'transferFrom',
      args: [SELLER, BUYER, labelhashOf(name)],
      chain: forkChain,
      account: SELLER,
    })
  }
  // The one changed condition for the control arm.
  await send(BUYER, {
    address: V1_BASE_REGISTRAR,
    abi: registrarAbi,
    functionName: 'reclaim',
    args: [labelhashOf(RECLAIMED), BUYER],
    chain: forkChain,
    account: BUYER,
  })

  for (const name of [SOLD_WITH_RECORD, SOLD_NO_RECORD, RECLAIMED]) {
    const s = await chainState(name)
    console.log(
      `[FIXTURE] ${name} registrant=${s.registrant} v1RegistryOwner=${s.registryOwner} resolver=${s.resolver}`,
    )
  }
}, 300_000)

describe('Explorer recipient resolution for an ENSv1 name', () => {
  it('1. a sold name whose own v1 resolver record names the holder is answered as the seller', async () => {
    const state = await chainState(SOLD_WITH_RECORD)
    const recordedAddr = await forkClient.readContract({
      address: state.resolver,
      abi: resolverAbi,
      functionName: 'addr',
      args: [namehash(SOLD_WITH_RECORD)],
    })
    const urAnswer = await getEnsAddress(forkClient, {
      name: SOLD_WITH_RECORD,
      universalResolverAddress,
    })
    const app = await appAnswerFor(SOLD_WITH_RECORD)
    console.log(`[1 SOLD] name=${SOLD_WITH_RECORD}`)
    console.log(
      `[1 SOLD] registrant(holder)=${state.registrant} v1RegistryOwner(seller)=${state.registryOwner}`,
    )
    console.log(`[1 SOLD] its resolver ${state.resolver} addr = ${recordedAddr}`)
    console.log(`[1 SOLD] universalResolver addr(60) = ${String(urAnswer)}`)
    console.log(`[1 SOLD] app getResolvedAddress     = ${app}`)
    expect(state.registrant).toBe(BUYER)
    expect(state.registryOwner).toBe(SELLER)
    expect(recordedAddr).toBe(BUYER)
    expect(app.toLowerCase()).toBe(SELLER.toLowerCase())
  }, 120_000)

  it('2. the seller retains ENSv1 write authority over the name they sold', async () => {
    const receipt = await send(SELLER, {
      address: V1_REGISTRY,
      abi: registryAbi,
      functionName: 'setResolver',
      args: [namehash(SOLD_WITH_RECORD), zeroAddress],
      chain: forkChain,
      account: SELLER,
    })
    const state = await chainState(SOLD_WITH_RECORD)
    const app = await appAnswerFor(SOLD_WITH_RECORD)
    console.log(
      `[2 SELL] seller ENS.setResolver(${SOLD_WITH_RECORD}, 0x0) receipt.status=${receipt.status}`,
    )
    console.log(
      `[2 SELL] resolver now=${state.resolver} registrant still=${state.registrant}`,
    )
    console.log(`[2 SELL] app getResolvedAddress = ${app}`)
    expect(receipt.status).toBe('success')
    expect(state.resolver).toBe(zeroAddress)
    expect(state.registrant).toBe(BUYER)
    expect(app.toLowerCase()).toBe(SELLER.toLowerCase())
  }, 120_000)

  it('3. a sold name that never had a resolver is answered as the seller', async () => {
    const state = await chainState(SOLD_NO_RECORD)
    const ensjsOwnership = await getOwnerV1(forkClient as never, {
      name: SOLD_NO_RECORD,
    })
    const app = await appAnswerFor(SOLD_NO_RECORD)
    const viaHelper = await resolveAddressOrName({
      client: forkClient as never,
      nameOrAddress: SOLD_NO_RECORD,
    })
    console.log(
      `[3 SOLD] ${SOLD_NO_RECORD} registrant=${state.registrant} v1RegistryOwner=${state.registryOwner} resolver=${state.resolver}`,
    )
    console.log(`[3 SOLD] ensjs getOwner v1     = ${JSON.stringify(ensjsOwnership)}`)
    console.log(`[3 SOLD] app getResolvedAddress = ${app}`)
    console.log(`[3 SOLD] resolveAddressOrName   = ${String(viaHelper)}`)
    expect(state.registrant).toBe(BUYER)
    expect(ensjsOwnership?.registrant).toBe(BUYER)
    expect(ensjsOwnership?.owner).toBe(SELLER)
    expect(app.toLowerCase()).toBe(SELLER.toLowerCase())
    expect(String(viaHelper).toLowerCase()).toBe(SELLER.toLowerCase())
  }, 120_000)

  it('4. control: same shape, same missing resolver, holder reclaimed the registry slot -> answered as the holder', async () => {
    const state = await chainState(RECLAIMED)
    const app = await appAnswerFor(RECLAIMED)
    console.log(
      `[4 CTRL] ${RECLAIMED} registrant=${state.registrant} v1RegistryOwner=${state.registryOwner} resolver=${state.resolver}`,
    )
    console.log(`[4 CTRL] app getResolvedAddress = ${app}`)
    expect(state.resolver).toBe(zeroAddress)
    expect(state.registryOwner).toBe(BUYER)
    expect(app.toLowerCase()).toBe(BUYER.toLowerCase())
  }, 120_000)

  it('5. container: the transfer form accepts the sold name and renders the seller as the recipient', async () => {
    const user = userEvent.setup()
    render(
      <Providers>
        <SendNameForm
          name="w1305-subject.eth"
          owner={USER}
          registryAddress={V2_ETH_REGISTRY}
        />
      </Providers>,
    )
    const field = await screen.findByPlaceholderText('ENS name or address')
    await user.type(field, SOLD_NO_RECORD)

    await waitFor(() => expect(screen.getByText(SELLER)).toBeTruthy(), {
      timeout: 60_000,
    })
    const refusals = [
      ...screen.queryAllByText('The recipient already owns this name.'),
      ...screen.queryAllByText('Can’t transfer to the zero address.'),
      ...screen.queryAllByText(/Could not resolve an address/),
    ]
    console.log(
      `[5 FORM] typed "${SOLD_NO_RECORD}" -> recipient preview shows ${SELLER}; holder rows = ${screen.queryAllByText(BUYER).length}; refusal messages = ${refusals.length}`,
    )
    expect(screen.queryAllByText(BUYER).length).toBe(0)
    expect(refusals.length).toBe(0)
  }, 180_000)

  it('5b. positive control: the same form, the same keystrokes, the reclaimed name -> the holder', async () => {
    const user = userEvent.setup()
    render(
      <Providers>
        <SendNameForm
          name="w1305-subject.eth"
          owner={USER}
          registryAddress={V2_ETH_REGISTRY}
        />
      </Providers>,
    )
    const field = await screen.findByPlaceholderText('ENS name or address')
    await user.type(field, RECLAIMED)
    await waitFor(() => expect(screen.getByText(BUYER)).toBeTruthy(), {
      timeout: 60_000,
    })
    console.log(
      `[5b CTRL] typed "${RECLAIMED}" -> recipient preview shows ${BUYER}; seller rows = ${screen.queryAllByText(SELLER).length}`,
    )
    expect(screen.queryAllByText(SELLER).length).toBe(0)
  }, 180_000)

  it('6. container: the Add-user sheet reports the name as Resolved and never says it could not be resolved', async () => {
    const user = userEvent.setup()
    render(
      <Providers>
        <RolesAddUserSheet
          name="w1305-subject.eth"
          onOpenChange={() => undefined}
          open
          registryAddress={V2_ETH_REGISTRY}
        />
      </Providers>,
    )
    const field = await screen.findByPlaceholderText('User name or address')
    await user.type(field, SOLD_NO_RECORD)

    await waitFor(() => expect(screen.getByText(/^Resolved: /)).toBeTruthy(), {
      timeout: 60_000,
    })
    const line = screen.getByText(/^Resolved: /).textContent
    const unresolved = screen.queryAllByText(/Could not resolve an address/)
    console.log(`[6 SHEET] status line = "${line}"`)
    console.log(`[6 SHEET] "Could not resolve" lines = ${unresolved.length}`)
    expect(line).toContain('0xBAd0')
    expect(unresolved.length).toBe(0)
  }, 180_000)

  it('7. the app grant-roles builder encodes the seller as the grantee', async () => {
    const resolved = await appAnswerFor(SOLD_NO_RECORD)
    const intent = prepareGrantRolesTransaction({
      name: 'w1305-subject.eth',
      account: resolved as `0x${string}`,
      roles: ['ROLE_SET_RESOLVER'] as never,
      walletClient: { account: { address: USER }, chain: forkChain } as never,
      chainId: forkChain.id,
      registryAddress: V2_ETH_REGISTRY,
    })
    const request = (intent as { request: { to: string; data: `0x${string}` } })
      .request
    const decoded = decodeFunctionData({
      abi: parseAbi([
        'function grantRoles(uint256 resource, uint256 roleBitmap, address account) returns (bool)',
      ]),
      data: request.data,
    })
    console.log(
      `[7 CALL] to=${request.to} ${decoded.functionName}(resource=${decoded.args?.[0]}, roleBitmap=${decoded.args?.[1]}, account=${decoded.args?.[2]})`,
    )
    expect(String(decoded.args?.[2]).toLowerCase()).toBe(SELLER.toLowerCase())
  }, 120_000)

  it('8. same-name control: the holder reclaims the registry slot and the same call answers with the holder', async () => {
    const before = await appAnswerFor(SOLD_NO_RECORD)
    await send(BUYER, {
      address: V1_BASE_REGISTRAR,
      abi: registrarAbi,
      functionName: 'reclaim',
      args: [labelhashOf(SOLD_NO_RECORD), BUYER],
      chain: forkChain,
      account: BUYER,
    })
    const state = await chainState(SOLD_NO_RECORD)
    const after = await appAnswerFor(SOLD_NO_RECORD)

    const sellerWrite = await send(SELLER, {
      address: V1_REGISTRY,
      abi: registryAbi,
      functionName: 'setResolver',
      args: [namehash(SOLD_NO_RECORD), V1_RESOLVER],
      chain: forkChain,
      account: SELLER,
      gas: 200_000n,
    })

    console.log(`[8 SAME] ${SOLD_NO_RECORD} before reclaim app answer = ${before}`)
    console.log(
      `[8 SAME] after reclaim v1RegistryOwner=${state.registryOwner} app answer = ${after}`,
    )
    console.log(
      `[8 SAME] seller ENS.setResolver on the same name receipt.status = ${sellerWrite.status}`,
    )
    console.log(`[8 SAME] non-local fetches blocked during run = ${blockedHosts.length} (${[...new Set(blockedHosts)].join(', ') || 'none'})`)
    expect(before.toLowerCase()).toBe(SELLER.toLowerCase())
    expect(after.toLowerCase()).toBe(BUYER.toLowerCase())
    expect(sellerWrite.status).toBe('reverted')
  }, 180_000)
})
```

Command:

```
pnpm --filter portal exec vitest run \
  src/__poc_w1305__/w1305_recipient_resolution.poc.test.tsx \
  --no-file-parallelism --reporter=verbose
```

Captured output:

```
[FIXTURE] w1305r80621294.eth registrant=0xb0B0000000000000000000000000000000000B0B v1RegistryOwner=0xBAd0000000000000000000000000000000000Bad resolver=0x8FADE66B79cC9f707aB26799354482EB93a5B7dD
[FIXTURE] w1305n80621294.eth registrant=0xb0B0000000000000000000000000000000000B0B v1RegistryOwner=0xBAd0000000000000000000000000000000000Bad resolver=0x0000000000000000000000000000000000000000
[FIXTURE] w1305c80621294.eth registrant=0xb0B0000000000000000000000000000000000B0B v1RegistryOwner=0xb0B0000000000000000000000000000000000B0B resolver=0x0000000000000000000000000000000000000000
[1 SOLD] name=w1305r80621294.eth
[1 SOLD] registrant(holder)=0xb0B0000000000000000000000000000000000B0B v1RegistryOwner(seller)=0xBAd0000000000000000000000000000000000Bad
[1 SOLD] its resolver 0x8FADE66B79cC9f707aB26799354482EB93a5B7dD addr = 0xb0B0000000000000000000000000000000000B0B
[1 SOLD] universalResolver addr(60) = null
[1 SOLD] app getResolvedAddress     = 0xBAd0000000000000000000000000000000000Bad
[2 SELL] seller ENS.setResolver(w1305r80621294.eth, 0x0) receipt.status=success
[2 SELL] resolver now=0x0000000000000000000000000000000000000000 registrant still=0xb0B0000000000000000000000000000000000B0B
[2 SELL] app getResolvedAddress = 0xBAd0000000000000000000000000000000000Bad
[3 SOLD] w1305n80621294.eth registrant=0xb0B0000000000000000000000000000000000B0B v1RegistryOwner=0xBAd0000000000000000000000000000000000Bad resolver=0x0000000000000000000000000000000000000000
[3 SOLD] ensjs getOwner v1     = {"registrant":"0xb0B0000000000000000000000000000000000B0B","owner":"0xBAd0000000000000000000000000000000000Bad","ownershipLevel":"registrar"}
[3 SOLD] app getResolvedAddress = 0xBAd0000000000000000000000000000000000Bad
[3 SOLD] resolveAddressOrName   = 0xBAd0000000000000000000000000000000000Bad
[4 CTRL] w1305c80621294.eth registrant=0xb0B0000000000000000000000000000000000B0B v1RegistryOwner=0xb0B0000000000000000000000000000000000B0B resolver=0x0000000000000000000000000000000000000000
[4 CTRL] app getResolvedAddress = 0xb0B0000000000000000000000000000000000B0B
[5 FORM] typed "w1305n80621294.eth" -> recipient preview shows 0xBAd0000000000000000000000000000000000Bad; holder rows = 0; refusal messages = 0
[5b CTRL] typed "w1305c80621294.eth" -> recipient preview shows 0xb0B0000000000000000000000000000000000B0B; seller rows = 0
[6 SHEET] status line = "Resolved: 0xBAd0…0Bad"
[6 SHEET] "Could not resolve" lines = 0
[7 CALL] to=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 grantRoles(resource=6889652877488229080686558571182453047713146547384829744857871981275017904128, roleBitmap=16777216, account=0xBAd0000000000000000000000000000000000Bad)
[8 SAME] w1305n80621294.eth before reclaim app answer = 0xBAd0000000000000000000000000000000000Bad
[8 SAME] after reclaim v1RegistryOwner=0xb0B0000000000000000000000000000000000B0B app answer = 0xb0B0000000000000000000000000000000000B0B
[8 SAME] seller ENS.setResolver on the same name receipt.status = reverted
[8 SAME] non-local fetches blocked during run = 12 (pulse.walletconnect.org, api.web3modal.org, sepolia.optimism.io, sepolia-rollup.arbitrum.io)

 Test Files  1 passed (1)
      Tests  9 passed (9)
   Duration  31.11s
```

Twelve outbound requests were attempted by third-party wallet and L2 libraries during the run and all twelve were refused by the local-only guard, so every value above was read from `127.0.0.1:8546`.

## Control

Same action, same function, same client, one condition changed.

**The changed condition** is who holds the ENSv1 registry-owner slot. In the attack arm the seller keeps it after selling the ERC-721. In the control arm the buyer calls `BaseRegistrar.reclaim(labelhash, buyer)`, which is the step an ENSv1 buyer is supposed to take. Everything else is identical: same registrant, same absent resolver, same typed string, same `getResolvedAddress` call.

|                                 | attack                       | control                      |
| ------------------------------- | ---------------------------- | ---------------------------- |
| condition                       | buyer did not `reclaim()`    | buyer called `reclaim()`     |
| `ENS.owner(node)`               | `0xBAd0…0Bad`                | `0xb0B0…0B0B`                |
| app answer for the typed name   | `0xBAd0…0Bad`                | `0xb0B0…0B0B`                |
| transfer form recipient preview | `0xBAd0…0Bad`, 0 holder rows | `0xb0B0…0B0B`, 0 seller rows |

Test 8 runs the same control on the very same name rather than a sibling: before `reclaim()` the app answered `0xBAd0…0Bad`, after `reclaim()` the same call answered `0xb0B0…0B0B`. The same test then sends the seller's `ENS.setResolver` on that name again and reads `receipt.status = reverted`, which is the measurement that the authority moved rather than the app changing its mind.

The control arm also supplies the non-zero row that proves the container harness works: test 5b types a name into the same mounted form with the same keystrokes and the form renders the buyer.

## Impact

The impact row is entailed predicate by predicate.

* *display or pre-fill an incorrect address*: the roles Add-user sheet printed `Resolved: 0xBAd0…0Bad` for a name whose holder is `0xb0B0000000000000000000000000000000000B0B`, and the transfer form printed the full seller address in the recipient preview with zero refusal messages and zero rows naming the holder.
* *resolving an ENS name to an attacker-controlled address*: the address is the seller's own account. Test 2 measures that the seller still has live ENSv1 write authority over the sold name and uses it to remove the resolver, which is the state that routes the user's input to them. The condition is therefore chosen, not merely inherited.
* *so that a user submits a transaction to an unintended recipient*: the app's own calldata builder, given the address the app resolved, produced `grantRoles(resource=6889652877488229080686558571182453047713146547384829744857871981275017904128, roleBitmap=16777216, account=0xBAd0…0Bad)` addressed to the ENSv2 registry `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`. `16777216` is `ROLE_SET_RESOLVER`, which the Explorer describes as "Can change the resolver addresses". The transfer path builds `safeTransferFrom(owner, recipient, tokenId, 1, '0x')` on the leaf registry from the same recipient value (`apps/portal/src/features/transfer/helpers/transferToken.ts:67-71`). This run did not broadcast either transaction.

Four flow families consume the same value: transferring an ENSv2 name, granting roles on a name or registry, and choosing the owner of a new subname. In the first the user parts with the name itself; in the others they hand a permission or a new subname to a stranger. Nothing later in any of those flows re-checks the recipient against the name the user typed.

The affected state is an ordinary ENSv1 ownership state, not a corrupted or contrived one. An ERC-721 transfer changes the registrant and does not automatically change the ENS registry-owner slot, so a name that changed hands and whose buyer has not called `reclaim()` carries a registry-owner slot pointing at the seller. Reaching the fallback additionally requires the name to have no address record. This report does not measure how many mainnet names are currently in that state and does not need to.

A previous owner who wants the condition can also create it rather than wait for it: measured above, `ENS.setResolver(node, 0x0)` sent by the seller after the sale returned `receipt.status=success`.

## Suggested fix

The invariant to restore: the address a name resolves to in a recipient field is the address ENS resolution returns for it, and when ENS returns none the field reports none.

1. Delete the owner fallback at `apps/portal/src/features/roles/helpers/addUser.handlers.ts:27-34`, and let `useAddressResolution` reach its declared `unresolved` state so `AddressNameInput` renders "Could not resolve an address for ...". A name with no address record has no address, and that is the answer the five call sites are already written to handle.
2. If a fallback is wanted for convenience, it must not use ENSv1's registry-owner slot for an unwrapped `.eth` name. `getOwnerV1` returns `registrant` alongside `owner` for exactly this shape; `resolveEnsOwner` should carry `registrant` through and prefer it when `ownershipLevel === 'registrar'`. It should also surface the divergence rather than resolve it silently: when `registrant` and `owner` differ, the field should say so and require the user to confirm the address, because either party may be the one they meant.
3. Whichever of the two is chosen, show the recipient in full before the user signs. The roles, registry and resolver Add-user sheets currently show a 6-and-4 truncation (`AddressNameInput.tsx:66`) and the create-subname owner field shows the same, which is not enough for a user to notice that the address is not the one they expected.
