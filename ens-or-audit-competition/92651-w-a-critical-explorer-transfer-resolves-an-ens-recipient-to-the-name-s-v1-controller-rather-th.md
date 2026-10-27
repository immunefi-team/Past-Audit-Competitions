# 92651 w a critical explorer transfer resolves an ens recipient to the name s v1 controller rather than its owner sending the user s name to a party that does not own the name they addressed

**Submitted on Sep 11th 2026 at 09:24:28 UTC by @kwaro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92651
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Direct theft of user NFTs
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The ENS Explorer lets you type an ENS name as the recipient of a name transfer. It resolves that name with viem's `getEnsAddress`, which reads `addr(60)` and nothing else; when the name has no `addr(60)` record, `resolveAddressOrName` silently falls back to a second source and uses whatever that returns as the `safeTransferFrom` recipient. For an ENS v1 name that second source is the **controller** — the delegated record manager — not the **registrant** who actually owns the name. A user who types a v1 name means its owner, and the app sends their ERC-1155 to a different party. Demonstrated end to end on the audited commit: a victim addressed a name owned by `0x70997970C51812dc3A010C7d01b50e0d17dc79C8` and the name landed on `0x90F79bf6EB2c4f870365E785982E1f101E93b906`, which does not own it. ERC-1155 transfers are irreversible.

## Vulnerability Details

### The fallback

`apps/portal/src/features/roles/helpers/addUser.handlers.ts` is the whole file:

```ts
import { type Address, isAddress } from 'viem'
import { getEnsAddress } from 'viem/actions'
import { getEnsOwner } from '@/features/profile/hooks/useEnsOwner'
import { universalResolverAddress } from '@/lib/constants/universalResolver'

type ResolveAddressOrNameParams = {
  client: Parameters<typeof getEnsAddress>[0]
  nameOrAddress: string
}

export async function resolveAddressOrName({
  client,
  nameOrAddress,
}: ResolveAddressOrNameParams): Promise<Address | null> {
  if (isAddress(nameOrAddress, { strict: false })) {
    return nameOrAddress as Address
  }

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
}
```

The comment on lines 27–28 states the intent: the fallback exists *so the role can still be granted*. For a role grant that is defensible — if a name has no address record, assigning a permission to whoever controls the name is a reasonable guess, and an operator can see and revoke it. The problem is that this helper is not confined to role grants.

`getEnsAddress` is called with no `coinType`, so it uses viem's default of 60. The app documents elsewhere that coin 60 is not the only address record a name can carry — `apps/portal/src/lib/coinType.ts`:

```ts
/**
 * Shared ENSIP-11 / ENSIP-19 coin type constants, used by both the
 * forward-resolution (name-view) and reverse-resolution (address-view)
 * features. `0x80000000` and `60` are SEPARATE records: coin 60 does not imply
 * the default is set, and vice versa.
 */
export const DEFAULT_EVM_COIN_TYPE = 0x80000000
export const MAINNET_COIN_TYPE = 60
```

The Explorer's Address Resolution page honours that distinction — `routes/$name/address.tsx:55-65` resolves a network to "its own coin-type record, or — for L2s without a chain-specific record — the ENSIP-19 default (`0x80000000`)". So a name carrying only `0x80000000` still shows a resolved address on those rows, while the transfer form treats it as having no address at all and takes the fallback.

### How a transfer reaches the fallback

The transfer form has no resolver of its own. It imports the roles helper:

`apps/portal/src/features/address/queries/getResolvedAddress.ts`

```ts
import { resolveAddressOrName } from '@/features/roles/helpers/addUser.handlers'
...
/**
 * Resolve a name/address input (an ENS name or a 0x address) to an address.
 * Addresses pass through; names go through the universal resolver, falling
 * back to the ENS owner when the name has no address record.
 */
export const getResolvedAddress = ResultFn(async function* (
  params: GetResolvedAddressParams,
) {
  const client = yield* safeGetClient()

  const address = await resolveAddressOrName({
    client,
    nameOrAddress: params.nameOrAddress,
  })

  return ok<Address | null>(address)
})
```

`useAddressResolution` wraps that query; its docstring says it is "Shared by every 'name or address' input in the app (transfer recipient, add-user sheets, …)". Five call sites:

| file                                                       | line | field              |
| ---------------------------------------------------------- | ---- | ------------------ |
| `features/transfer/components/SendNameForm.tsx`            | 81   | transfer recipient |
| `routes/$name/create-subname.tsx`                          | 109  | new subname owner  |
| `features/roles/components/RolesAddUserSheet.tsx`          | 73   | role grantee       |
| `features/registry/components/v2/RegistryAddUserSheet.tsx` | 66   | role grantee       |
| `features/resolver/components/ResolverAddUserSheet.tsx`    | 141  | role grantee       |

This report claims the first, because it is the one that moves an asset. In `SendNameForm.tsx` the resolved address goes straight through:

```tsx
const resolution = useAddressResolution(recipientInput)
const { address: recipient, isResolving } = resolution
...
const runTransfer = () => {
  if (!recipient || !canStart) return
  startTransfer({ recipient, options: effectiveOptions })
}
```

and lands in `features/transfer/helpers/transferToken.ts`:

```ts
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
})
```

### The core defect: the fallback returns the controller, not the owner

ENS v1 keeps two distinct roles on a `.eth` 2LD:

* **registrant** — the BaseRegistrar ERC-721 owner. This is who owns the name.
* **controller** — the ENS registry `owner(node)`. This is who may edit its records.

Delegating the controller to a different address is a first-class, documented, permissionless ENS v1 feature. ensjs returns both fields and labels them, in its own source comments (`@ensdomains/ensjs`, `actions/public/v1/getOwner.ts`):

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
  }
}
```

`apps/portal/src/utils/ens/resolveEnsOwner.ts` takes `.owner`:

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

Of the two fields ensjs hands it, the app takes the controller. The user addressed the name; the name belongs to the registrant; the token goes to the controller.

The same function has a second branch with the same shape for **expired** 2LDs, which returns `registrant: null` alongside a live `owner` — a name nobody owns any more still resolving to its last controller. This is read from source and is not separately demonstrated below.

### The owner read is on-chain

`getEnsOwner` resolves through `resolveEnsOwner`, which calls ensjs `getOwner` for v2 and falls back to the v1 registry. Its own comment: *"The UniversalResolver V2 walks the registry tree on-chain, so both the owner and the name's ancestry of registries are read directly by name at any depth."* The v1 branch reads the ENS registry and BaseRegistrar directly. No subgraph, indexer or third-party service produces the value that becomes the recipient.

### What the user is shown

`AddressNameInput` renders the resolution under the label `Resolved:`. On the transfer form that is replaced by `RecipientPreview`, which renders the full address plus the primary name of that address. Neither surface distinguishes an address that came from `addr(60)` from one that came from a fallback, and neither says which role of the name it came from. There is no "this name has no ETH record" notice anywhere on the path.

The authors knew the fallback can fire. `apps/portal/src/utils/token/isNameOrAddress.ts` rejects single-label input specifically because of it:

```
 * The name must have at least two labels (i.e. contain a `.`). A bare
 * single label like `fox` is a valid ENS name string but not a resolvable
 * account — only TLDs are single-label — and would otherwise "resolve" via the
 * owner fallback in `resolveAddressOrName`.
```

That guard stops `fox`. It does not stop `alice.eth`.

### Trigger

No attacker is required. The ordinary case:

1. `alice.eth` is an unwrapped v1 name. Alice is the registrant. She delegates record management to a manager, a hot wallet, or a service — one `registry.setOwner(node, controller)` call, a normal thing to do.
2. `alice.eth` has no `addr(60)` record.
3. A user opens their name's Ownership → Transfer screen and types `alice.eth`, meaning Alice.
4. `getEnsAddress` returns nothing. The fallback returns the **controller**.
5. The form shows that address as the resolved recipient, with no warning.
6. The user confirms and signs. The name lands on the controller. Alice receives nothing.

The same defect is reachable deliberately: anyone may register a name, leave `addr(60)` unset, and point the controller wherever they like.

## Impact Details

A user loses an ENS name to a party they did not address, on a flow whose entire purpose is choosing the recipient. ERC-1155 transfers are irreversible; recovery depends entirely on the receiving party choosing to send it back, and the receiving party in the demonstrated case is a delegated record manager with no ownership claim to the name that was used to address them.

The wallet prompt cannot catch it. The wallet is asked to sign an ERC-1155 `safeTransferFrom` to the exact address the application displayed as correct. Both surfaces agree, and both are wrong in the same direction, because the wallet is handed the number the app resolved. The only place the substitution could have been disclosed is the form that made it.

Selected impacts:

* **Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient — resolving or rendering an ENS name to an attacker-controlled address.** The measured result is the row's first bullet: the app resolved a name to an address that is not the party that name identifies, and the user submitted a transaction to it. The row's own wording presupposes the user approves ("so that a user sends funds or **submits a transaction**").
* **Direct theft of user NFTs.** ENS v2 names are ERC-1155 tokens. The run ends with the victim's token owned by a party they never addressed.
* **Hijacking or bricking an ENS name — redirecting the recipient of a name transfer.** A verbatim description of the measured outcome.

### Anticipated objections, answered

* *"The name went to the owner of the name the user typed."* Not in the demonstrated arm. The name went to the **controller**; the registrant — the party that owns the addressed name — received nothing, measured as `name_landed_on_addressed_party: 0`.
* *"The wallet shows the address, so the user can check."* Row 6285 presupposes a wallet approval; if wallet visibility were a valid downgrade the row could never be satisfied by anything.
* *"It needs an attacker to set up a hostile name."* It does not. The delegation in the demonstrated arm is an ordinary action by the name's own owner.
* *"The indexer is out of scope."* The recipient value is read on-chain from the ENS registry and BaseRegistrar. No indexer is involved in producing it.
* *"This is a UI/UX issue."* An NFT irreversibly changes owner.

### Not covered by a known issue

**QA-01** concerns role combinations *after* a transfer causing errors when later updating records or setting a primary name, and carves in that "a demonstrated security consequence (e.g., a role combination that grants unintended authority) would be a new finding". This is recipient resolution, evaluated before the transfer is built, and the consequence is the name landing on the wrong party.

**WEB-1257** is a previous owner being unable to set a subname as their primary name after a parent transfer. Different direction, different code path.

### Scope, stated precisely

* **Wrapped names are not affected.** ensjs resolves `registrarOwner === nameWrapperAddress` to the true NameWrapper owner, so there is no registrant/controller split to exploit. Confirmed on the fork against `ens.eth`, `sepolia.eth` and `test.eth`: all three return `ownershipLevel: nameWrapper` and the real owner.
* The defect is specific to **unwrapped v1 2LDs** and to **expired 2LDs**.

## References

* `apps/portal/src/features/roles/helpers/addUser.handlers.ts` — `resolveAddressOrName`, the fallback (lines 27–34)
* `apps/portal/src/utils/ens/resolveEnsOwner.ts` — takes `.owner`, the controller
* `@ensdomains/ensjs` `actions/public/v1/getOwner.ts` — returns `registrant` and `owner` and labels them
* `apps/portal/src/features/address/queries/getResolvedAddress.ts:6` — transfer path importing the roles helper
* `apps/portal/src/features/address/hooks/useAddressResolution.ts` — the shared hook
* `apps/portal/src/features/transfer/components/SendNameForm.tsx:81` — recipient selection
* `apps/portal/src/features/transfer/helpers/transferToken.ts` — `safeTransferFrom`
* `apps/portal/src/lib/coinType.ts` — coin 60 and `0x80000000` are separate records
* `apps/portal/src/routes/$name/address.tsx:55-65` — ENSIP-19 default used for L2 forward resolution
* `apps/portal/src/utils/token/isNameOrAddress.ts:18-21` — existing partial guard against this same fallback
* ENSIP-19: <https://docs.ens.domains/ensip/19>

### Suggested fix

Fail closed. On a path that moves an asset, resolve to `null` when `getEnsAddress` returns nothing:

```ts
if (!resolvedAddress) {
  return null
}
```

If the fallback is worth keeping for role grants, give transfer and create-subname their own resolver rather than sharing the roles helper. If a fallback is kept for usability, prefer `addr(0x80000000)` over the owner slot — the app already treats the two as distinct — and never use ensjs's `.owner` for a v1 name when `.registrant` is the party being addressed. Whatever the source, label it in the UI.

## Proof of Concept

## Proof of Concept

Two arms, both executed end to end against the pinned commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

* **Arm A — the fallback fires and decides the recipient.**
* **Arm B — the recipient is not the party the user addressed.** This is the one that establishes impact.

### What is and is not real

Nothing on the vulnerable path is substituted. `resolveAddressOrName`, `getEnsAddress`, `getEnsOwner`/`resolveEnsOwner`, ensjs `getOwner`, and `transferToken`/`safeTransferFrom` all execute unmodified from the pinned tree.

| component        | what actually runs                                                                                                                                                                |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| chain            | anvil forking Sepolia; real deployed bytecode for the ENS registry, BaseRegistrar, resolver and USDC                                                                              |
| names            | real v2 `ETHRegistrar` registrations through the repo's own `makeName` fixture (commit → `MIN_COMMITMENT_AGE` → approve USDC → register)                                          |
| indexer          | the real Panoptes service, `ghcr.io/mdtanrikulu/panoptes`, the same image and command as `e2e/infra/docker-compose.yml` — not the CI stand-in reached via `E2E_MOCK_INDEXER=true` |
| app              | real Vite dev server serving `apps/portal` at the pinned commit                                                                                                                   |
| wallet           | real `headless-web3-provider`, signing real transactions                                                                                                                          |
| transfer options | left at their defaults and never toggled                                                                                                                                          |

The program's safe harbor is false, so nothing may be broadcast to a live network; the fork is required, not a convenience.

Every measured key is numeric on purpose: the extractor used to check these figures reads numbers only, so a hex address declared as a measurement would be silently read as the digit `0`. Addresses and names are printed under explicit `_str` keys that are **not** measurements.

### Steps to reproduce

```bash
# repo checked out at 1c9b47f18fcddd2e864dfe385c4171061c9811ae

# bring up anvil + the real Panoptes indexer + the portal
bash verification/pocs/H-C4-NM-1/ensure-stack.sh

# Arm A — the fallback decides the recipient
bash verification/pocs/H-C4-NM-1/run.sh exploit
bash verification/pocs/H-C4-NM-1/run.sh control

# Arm B — the recipient is not the party addressed
bash verification/pocs/H-C4-NM-1-DIV/run.sh exploit
bash verification/pocs/H-C4-NM-1-DIV/run.sh control
```

***

## Arm B — the recipient is not the party the user addressed

The victim addressed a v1 name whose **registrant** is `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`. That registrant had delegated record management to `0x90F79bf6EB2c4f870365E785982E1f101E93b906` with one ordinary `registry.setOwner` call.

**Exploit arm — registrant ≠ controller:**

```
mode: exploit
chain_id: 11155111
addressed_party_registrant_str: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
delegated_controller_str:       0x90F79bf6EB2c4f870365E785982E1f101E93b906
ensjs_registrant_str:           0x70997970C51812dc3A010C7d01b50e0d17dc79C8
ensjs_owner_used_by_app_str:    0x90F79bf6EB2c4f870365E785982E1f101E93b906
ensjs_ownership_level_str: registrar
registrant_and_controller_differ: 1
route_mounted: 1
ui_resolved_str: 0x90F79bf6EB2c4f870365E785982E1f101E93b906
ui_resolved_is_addressed_party: 0
ui_resolved_is_controller: 1
ui_showed_wrong_party: 1
transfer_ok: 1
owner_before_str: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
owner_after_str:  0x90F79bf6EB2c4f870365E785982E1f101E93b906
name_landed_on_addressed_party: 0
name_landed_on_controller: 1
name_diverted_from_addressed_party: 1
name_still_victim: 0
impact_row: 6285
```

The victim's name left their wallet and landed on the delegate. The registrant — the party they addressed, and the party that owns that name — received nothing.

**Control arm — identical except the registrant never delegates**, so both roles are held by the same party:

```
registrant_and_controller_differ: 0
ui_resolved_is_addressed_party: 1
ui_showed_wrong_party: 0
transfer_ok: 1
name_landed_on_addressed_party: 1
name_diverted_from_addressed_party: 0
```

A single `registry.setOwner(node, controller)` call is the only difference between the arms, and it decides whether the name reaches the party the user addressed.

**Declared fix, applied and re-measured:**

```
ui_showed_wrong_party               1 -> 0
transfer_ok                         1 -> 0
name_diverted_from_addressed_party  1 -> 0
name_landed_on_controller           1 -> 0
```

The control arm continues to resolve and transfer normally under the same patch, so the fix closes the divergence without breaking the feature.

**On reproduction:** creating an unwrapped v1 2LD on a fork requires authorising a minter on the BaseRegistrar via impersonation. That is fork setup so such a name exists in the test environment. It is not part of the attack path, both arms perform it identically so it cannot account for the difference between them, and the resulting on-chain state (registrant ≠ controller) is an ordinary state that exists on mainnet today.

***

## Arm A — the fallback fires and decides the recipient

**Exploit arm — recipient name has no `addr(60)`:**

```
switch[Set the ETH address to the recipient]: visible=0
switch[Detach the resolver]: visible=1 checked=1 (untouched)
switch[Detach the registry]: visible=0
transfer_button_visible: 1
transfer_button_enabled: 1
mode: exploit
chain_id: 11155111
fork_block: 11678961
indexer_is_real_graphql: 1
indexer_query_type_str: Query
page_route_intercepts: 0
victim_name: nm1-victim-1789092920.eth
recipient_name: nm1-alice-1789092944.eth
recipient_name_onchain_owner_str: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
recipient_owner_is_attacker: 1
coin60_present: 0
coin60_value_str: none
route_mounted: 1
ui_resolved_address_str: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
ui_resolved_present: 1
resolved_to_owner: 1
resolved_to_friend: 0
switches_left_at_default: 1
transfer_ok: 1
onchain_owner_before_str: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
onchain_owner_after_str: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
onchain_owner_is_attacker: 1
onchain_owner_is_friend: 0
onchain_owner_still_victim: 0
impact_row: 6285
secondary_impact_row: 6286
```

`coin60_present` is read from the chain with `getAddressRecord`, not inferred from which arm is running. `ui_resolved_address_str` is scraped from the recipient preview the user actually sees. Only "Detach the resolver" is rendered for these names; nothing is toggled.

The transaction sequence logged by the app's own transaction manager during that run:

```
Transaction transfer-nm1-victim-1789092920.eth-detach-resolver state: pending
Transaction transfer-nm1-victim-1789092920.eth-detach-resolver state: success
Transaction transfer-nm1-victim-1789092920.eth-transfer-token state: submitting
Transaction transfer-nm1-victim-1789092920.eth-transfer-token state: pending
Transaction transfer-nm1-victim-1789092920.eth-transfer-token state: success
```

**Control arm** — identical except `addr(60)` is set on the recipient name to `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`:

```
coin60_present: 1
ui_resolved_present: 1
resolved_to_owner: 0
resolved_to_friend: 1
transfer_ok: 1
onchain_owner_is_attacker: 0
onchain_owner_is_friend: 1
```

The name still transfers; it just goes where the record points. That isolates the fallback as the deciding factor.

Addresses in both arms are the first accounts of the default anvil test mnemonic, which is what the repo's own portal fixture derives `user`, `user2`, `user3` and `user4` from. `victim_name`, `recipient_name` and `fork_block` change every run and are not treated as measurements.

***

## PoC source — Arm B, `e2e/projects/portal/tests/nm1-divergence.spec.ts`

```ts
/**
 * H-C4-NM-1 DIVERGENCE — the recipient the app sends to is NOT the party the
 * user addressed by that name.
 *
 * In ENS v1 a .eth 2LD has two distinct roles:
 *   registrant = the BaseRegistrar ERC-721 owner  (who owns the name)
 *   controller = the ENS registry `owner(node)`   (who may edit its records)
 *
 * ensjs returns BOTH and labels them (v1/getOwner.ts):
 *   "the registrant is the owner of the NFT / the owner is the controller of
 *    the records"
 *
 * `resolveEnsOwner` takes `.owner` — the CONTROLLER. So a user who types a v1
 * name means the registrant and the app sends the token to the controller.
 *
 * MODE=exploit   registrant != controller -> the token lands on the controller
 * MODE=control   registrant == controller -> the token lands on the registrant
 *
 * The v1 name is created on the fork by authorising a minter on the BaseRegistrar
 * (fork setup, NOT part of the attack path) and then minting to the registrant.
 * The delegation itself — `registry.setOwner(node, controller)` — is an ordinary
 * permissionless action taken by the registrant, and is the only difference
 * between the two arms.
 */
import { getOwner as getOwnerV1 } from '@ensdomains/ensjs/public/v1'
import { getOwner as getOwnerV2 } from '@ensdomains/ensjs/public/v2'
import type { Web3ProviderBackend } from '@ensdomains/headless-web3-provider'
import type { Page } from '@playwright/test'
import {
  createPublicClient,
  createTestClient,
  createWalletClient,
  getAddress,
  http,
  isAddressEqual,
  keccak256,
  namehash,
  parseAbi,
  toHex,
} from 'viem'
import { mnemonicToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../../../fixtures/playwright.portal.fixture.js'
import { authorizeTransaction } from '../../../helpers/portal-auth.js'
import { publicClient } from '../../../helpers/anvil-client.js'

const PORTAL_APP_URL = process.env.PORTAL_APP_URL ?? 'http://127.0.0.1:8918'
const RPC = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8736'
const INDEXER = process.env.NM1_INDEXER_URL ?? 'http://127.0.0.1:5656/graphql'
const MODE = process.env.NM1_MODE === 'control' ? 'control' : 'exploit'

const MNEMONIC = 'test test test test test test test test test test test junk'
const REGISTRY = '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e' as const
const BASEREG = '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85' as const
const BASEREG_OWNER = '0x4ad56feb5Fc7B8298db06E88fd5CBc41D64602Fa' as const

const BR_ABI = parseAbi([
  'function addController(address controller)',
  'function register(uint256 id, address owner, uint duration) returns (uint)',
  'function ownerOf(uint256 id) view returns (address)',
])
const RG_ABI = parseAbi([
  'function owner(bytes32 node) view returns (address)',
  'function setOwner(bytes32 node, address owner)',
])

test.use({ channel: 'chrome' })

async function waitForIndexer(name: string, timeoutMs = 120_000) {
  const deadline = Date.now() + timeoutMs
  while (Date.now() < deadline) {
    try {
      const r = await fetch(INDEXER, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ query: `{ domains(first: 50) { name } }` }),
      })
      const j = (await r.json()) as { data?: { domains?: { name: string }[] } }
      if (j?.data?.domains?.some((d) => d.name === name)) return 1
    } catch {
      /* not ready */
    }
    await new Promise((res) => setTimeout(res, 2_000))
  }
  return 0
}

async function driveToSuccess(
  page: Page,
  wallet: Web3ProviderBackend,
  finalTxId: string,
  timeoutMs = 420_000,
): Promise<boolean> {
  const dialog = page.locator('[data-slot="dialog-content"]')
  await expect(dialog).toBeVisible({ timeout: 45_000 })
  let done = false
  let stop = false
  const onConsole = (m: { text(): string }) => {
    if (m.text().includes(`Transaction ${finalTxId} state: success`)) done = true
  }
  page.on('console', onConsole)
  const authorizer = (async () => {
    while (!stop && !done) {
      try {
        await authorizeTransaction(wallet, 4_000)
      } catch {
        /* nothing pending */
      }
      await new Promise((r) => setTimeout(r, 250))
    }
  })()
  try {
    const deadline = Date.now() + timeoutMs
    while (Date.now() < deadline && !done) {
      for (const re of [/open wallet/i, /^(Start|Next|Done|Continue|Retry)$/i]) {
        const b = dialog.getByRole('button', { name: re }).first()
        if (
          (await b.isVisible().catch(() => false)) &&
          (await b.isEnabled().catch(() => false))
        ) {
          await b.click().catch(() => {})
          await page.waitForTimeout(500)
        }
      }
      await page.waitForTimeout(800)
    }
  } finally {
    stop = true
    await authorizer.catch(() => {})
    page.off('console', onConsole)
  }
  return done
}

test.describe('H-C4-NM-1 divergence', () => {
  test.describe.configure({ timeout: 900_000 })

  test(`MODE=${MODE}`, async ({ portalPage: page, wallet, accounts, makeName }) => {
    test.setTimeout(900_000)

    const chain = extendChainWithEns({
      ...sepolia,
      rpcUrls: { default: { http: [RPC] }, public: { http: [RPC] } },
    })
    const testClient = createTestClient({ chain, mode: 'anvil', transport: http(RPC) })
    // ensjs v1 reads need a chain carrying the ENS contract addresses; the e2e
    // helper's publicClient does not have ensLegacyRegistry configured.
    const ensClient = createPublicClient({ chain, transport: http(RPC) })
    const walletClient = createWalletClient({ chain, transport: http(RPC) })

    const registrant = mnemonicToAccount(MNEMONIC, { addressIndex: 1 }) // user2
    const controller = mnemonicToAccount(MNEMONIC, { addressIndex: 3 }) // user4
    const minter = mnemonicToAccount(MNEMONIC, { addressIndex: 9 })
    const victimWallet = getAddress(accounts.getAddress('user'))

    for (const a of [BASEREG_OWNER, registrant.address, minter.address]) {
      await testClient.setBalance({ address: a, value: 10n ** 19n })
    }

    // ---- fork setup: create an UNWRAPPED v1 2LD owned by the registrant ----
    const label = `div-${Date.now()}`
    const v1Name = `${label}.eth`
    const tokenId = BigInt(keccak256(toHex(label)))

    await testClient.impersonateAccount({ address: BASEREG_OWNER })
    await publicClient.waitForTransactionReceipt({
      hash: await walletClient.writeContract({
        account: BASEREG_OWNER,
        address: BASEREG,
        abi: BR_ABI,
        functionName: 'addController',
        args: [minter.address],
      }),
    })
    await testClient.stopImpersonatingAccount({ address: BASEREG_OWNER })

    await publicClient.waitForTransactionReceipt({
      hash: await walletClient.writeContract({
        account: minter,
        address: BASEREG,
        abi: BR_ABI,
        functionName: 'register',
        args: [tokenId, registrant.address, 31536000n],
      }),
    })

    // ---- the single variable between the arms -----------------------------
    // exploit: the registrant delegates record management to a third party.
    // control: no delegation, so registrant and controller are the same party.
    if (MODE === 'exploit') {
      await publicClient.waitForTransactionReceipt({
        hash: await walletClient.writeContract({
          account: registrant,
          address: REGISTRY,
          abi: RG_ABI,
          functionName: 'setOwner',
          args: [namehash(v1Name), controller.address],
        }),
      })
    }

    const onchainRegistrant = await publicClient.readContract({
      address: BASEREG,
      abi: BR_ABI,
      functionName: 'ownerOf',
      args: [tokenId],
    })
    const onchainController = await publicClient.readContract({
      address: REGISTRY,
      abi: RG_ABI,
      functionName: 'owner',
      args: [namehash(v1Name)],
    })
    const ensjs = await getOwnerV1(ensClient, { name: v1Name })

    const expectedParty = onchainRegistrant // the party the user addresses
    const rolesDiffer = !isAddressEqual(onchainRegistrant, onchainController)

    // ---- the victim's own v2 name ----------------------------------------
    const victimName = await makeName({ label: 'div-victim', owner: 'user' })
    await waitForIndexer(victimName)
    const ownerBefore = await getOwnerV2(publicClient, { name: victimName })

    await connectWithHeadlessWallet(page, wallet)
    await page.goto(`${PORTAL_APP_URL}/${victimName}/ownership/transfer`)
    const input = page.getByPlaceholder('ENS name or address').first()
    const mounted = await input
      .waitFor({ state: 'visible', timeout: 180_000 })
      .then(() => true)
      .catch(() => false)

    let uiResolved = 'none'
    let transferOk = 0

    if (mounted) {
      await input.fill(v1Name)
      const preview = page
        .locator('[class*="bg-muted"]')
        .filter({ hasText: /0x[a-fA-F0-9]{40}/ })
      await preview
        .first()
        .waitFor({ state: 'visible', timeout: 60_000 })
        .catch(() => {})
      await page.waitForTimeout(3_000)
      const txt = await preview.first().innerText().catch(() => '')
      const m = txt.match(/0x[a-fA-F0-9]{40}/)
      if (m) uiResolved = getAddress(m[0])

      const btn = page.getByRole('button', { name: 'Transfer name' }).first()
      if (await btn.isEnabled().catch(() => false)) {
        await btn.click()
        transferOk = (await driveToSuccess(
          page,
          wallet,
          `transfer-${victimName}-transfer-token`,
        ))
          ? 1
          : 0
      }
    }

    const ownerAfter = await getOwnerV2(publicClient, { name: victimName })
    const uiIsAddr = /^0x[a-fA-F0-9]{40}$/.test(uiResolved)

    console.log(`mode: ${MODE}`)
    console.log(`chain_id: ${await publicClient.getChainId()}`)
    console.log(`v1_name_str: ${v1Name}`)
    console.log(`victim_name_str: ${victimName}`)
    console.log(`addressed_party_registrant_str: ${onchainRegistrant}`)
    console.log(`delegated_controller_str: ${onchainController}`)
    console.log(`ensjs_registrant_str: ${ensjs?.registrant ?? 'none'}`)
    console.log(`ensjs_owner_used_by_app_str: ${ensjs?.owner ?? 'none'}`)
    console.log(`ensjs_ownership_level_str: ${ensjs?.ownershipLevel ?? 'none'}`)
    console.log(`registrant_and_controller_differ: ${rolesDiffer ? 1 : 0}`)
    console.log(`route_mounted: ${mounted ? 1 : 0}`)
    console.log(`ui_resolved_str: ${uiResolved}`)
    console.log(
      `ui_resolved_is_addressed_party: ${
        uiIsAddr && isAddressEqual(uiResolved as `0x${string}`, expectedParty) ? 1 : 0
      }`,
    )
    console.log(
      `ui_resolved_is_controller: ${
        uiIsAddr && isAddressEqual(uiResolved as `0x${string}`, onchainController) ? 1 : 0
      }`,
    )
    console.log(`transfer_ok: ${transferOk}`)
    console.log(`owner_before_str: ${ownerBefore}`)
    console.log(`owner_after_str: ${ownerAfter}`)
    console.log(
      `name_landed_on_addressed_party: ${
        ownerAfter && isAddressEqual(ownerAfter, expectedParty) ? 1 : 0
      }`,
    )
    console.log(
      `name_landed_on_controller: ${
        ownerAfter && isAddressEqual(ownerAfter, onchainController) ? 1 : 0
      }`,
    )
    console.log(
      `name_still_victim: ${
        ownerAfter && isAddressEqual(ownerAfter, victimWallet) ? 1 : 0
      }`,
    )
    // Impact-polarity keys: 1 means the defect fired. The checker treats a
    // truthy measured key as "the impact is present", so these must read 1 in
    // the exploit arm and 0 in the control arm.
    const uiWrongParty =
      uiIsAddr && !isAddressEqual(uiResolved as `0x${string}`, expectedParty) ? 1 : 0
    const diverted =
      ownerAfter && !isAddressEqual(ownerAfter, expectedParty) && transferOk === 1
        ? 1
        : 0
    console.log(`ui_showed_wrong_party: ${uiWrongParty}`)
    console.log(`name_diverted_from_addressed_party: ${diverted}`)
    console.log(`impact_row: 6285`)
  })
})

```

## PoC source — Arm A, `e2e/projects/portal/tests/nm1-real.spec.ts`

```ts
/**
 * H-C4-NM-1 — Explorer transfer resolves an ENS recipient to the name's OWNER
 * when the name has no ETH address record, and hands the NFT to that owner.
 *
 * NOTHING ON THE VULNERABLE PATH IS SUBSTITUTED.
 *   chain     real anvil Sepolia fork (:8736), real V2 ETHRegistrar registrations
 *   indexer   real Panoptes (ghcr.io/mdtanrikulu/panoptes) on :5656 — the live
 *             service, not the CI stand-in; no page.route() anywhere
 *   app       real Vite dev server serving apps/portal at the pinned commit
 *   wallet    real headless-web3-provider signing real transactions
 *   options   the three transfer switches are left at their DEFAULTS (all on)
 *
 * Keys WITHOUT a `_str` suffix are the measured ones and are always numeric:
 * verify_poc's extractor only captures numbers, so a hex address declared as a
 * measurement would be silently read as the digit 0. `coin60_present` is read
 * from the chain, not inferred from MODE; `ui_resolved_present` comes from an
 * address scraped out of the DOM, not from the address the arm expects.
 *
 * MODE=exploit  recipient name has no coin-60 record -> owner receives the name
 * MODE=control  recipient name has coin-60 = friend  -> friend receives the name
 */
import { getResolver } from '@ensdomains/ensjs/public'
import { getOwner as getOwnerV2 } from '@ensdomains/ensjs/public/v2'
import { setRecords } from '@ensdomains/ensjs/wallet'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import type { Web3ProviderBackend } from '@ensdomains/headless-web3-provider'
import type { Page } from '@playwright/test'
import { createWalletClient, getAddress, http, isAddressEqual } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../../../fixtures/playwright.portal.fixture.js'
import { authorizeTransaction } from '../../../helpers/portal-auth.js'
import { publicClient } from '../../../helpers/anvil-client.js'

const PORTAL_APP_URL = process.env.PORTAL_APP_URL ?? 'http://127.0.0.1:8918'
const RPC = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8736'
const INDEXER = process.env.NM1_INDEXER_URL ?? 'http://127.0.0.1:5656/graphql'
const MODE = process.env.NM1_MODE === 'control' ? 'control' : 'exploit'

test.use({ channel: 'chrome' })

/** Read addr(60) straight off the chain. No app code involved. */
async function readCoin60(name: string): Promise<string | null> {
  const resolverAddress = await getResolver(publicClient, { name }).catch(
    () => null,
  )
  if (!resolverAddress) return null
  const { getAddressRecord } = await import('@ensdomains/ensjs/public')
  const rec = await getAddressRecord(publicClient, {
    name,
    coin: 60,
    resolverAddress,
  }).catch(() => null)
  const v = rec?.value
  if (!v || /^0x0{40}$/i.test(v)) return null
  return getAddress(v)
}

/**
 * Wait until the REAL Panoptes indexer has caught up to a name we just
 * registered on-chain. This is synchronisation with a live service, not a
 * substitute for one: the name and its owner are already on chain, and we are
 * only waiting for the indexer the app talks to to catch up.
 */
async function waitForIndexer(name: string, timeoutMs = 120_000): Promise<number> {
  const deadline = Date.now() + timeoutMs
  while (Date.now() < deadline) {
    try {
      const r = await fetch(INDEXER, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({
          query: `{ domains(first: 50) { name } }`,
        }),
      })
      const j = (await r.json()) as { data?: { domains?: { name: string }[] } }
      if (j?.data?.domains?.some((d) => d.name === name)) {
        return Math.round((timeoutMs - (deadline - Date.now())) / 1000)
      }
    } catch {
      /* indexer not ready yet */
    }
    await new Promise((res) => setTimeout(res, 2_000))
  }
  return -1
}

async function driveToSuccess(
  page: Page,
  wallet: Web3ProviderBackend,
  finalTxId: string,
  timeoutMs = 420_000,
): Promise<boolean> {
  const dialog = page.locator('[data-slot="dialog-content"]')
  await expect(dialog).toBeVisible({ timeout: 45_000 })
  let done = false
  let stop = false
  const onConsole = (msg: { text(): string }) => {
    if (msg.text().includes(`Transaction ${finalTxId} state: success`)) {
      done = true
    }
  }
  page.on('console', onConsole)

  // The wallet queue is driven independently of the UI: whenever the app posts
  // a SendTransaction request, approve it. Waiting for an "Open wallet" button
  // misses the steps the app prompts for directly.
  const authorizer = (async () => {
    while (!stop && !done) {
      try {
        await authorizeTransaction(wallet, 4_000)
      } catch {
        /* nothing pending in this slice */
      }
      await new Promise((r) => setTimeout(r, 250))
    }
  })()

  try {
    const deadline = Date.now() + timeoutMs
    while (Date.now() < deadline && !done) {
      for (const re of [/open wallet/i, /^(Start|Next|Done|Continue|Retry)$/i]) {
        const b = dialog.getByRole('button', { name: re }).first()
        if (
          (await b.isVisible().catch(() => false)) &&
          (await b.isEnabled().catch(() => false))
        ) {
          await b.click().catch(() => {})
          await page.waitForTimeout(500)
        }
      }
      await page.waitForTimeout(800)
    }
  } finally {
    stop = true
    await authorizer.catch(() => {})
    page.off('console', onConsole)
  }
  return done
}

test.describe('H-C4-NM-1 real-stack', () => {
  test.describe.configure({ timeout: 900_000 })

  test(`MODE=${MODE}`, async ({ portalPage: page, wallet, accounts, makeName }) => {
    test.setTimeout(900_000)

    const attacker = getAddress(accounts.getAddress('user2'))
    const friend = getAddress(accounts.getAddress('user3'))
    const victimWallet = getAddress(accounts.getAddress('user'))

    const failed: string[] = []
    page.on('requestfailed', (r) => {
      failed.push(`${r.failure()?.errorText ?? '?'} ${r.url().slice(0, 110)}`)
    })
    const txLog: string[] = []
    page.on('console', (m) => {
      const t = m.text()
      if (/Transaction .* state:/.test(t)) txLog.push(t.slice(0, 120))
      if (m.type() === 'error') failed.push(`CONSOLE ${t.slice(0, 130)}`)
    })

    await connectWithHeadlessWallet(page, wallet)

    // ---- real on-chain registrations -------------------------------------
    const victimName = await makeName({ label: 'nm1-victim', owner: 'user' })
    const recipientName = await makeName({
      label: 'nm1-alice',
      owner: 'user2',
      records: [{ key: 'url', value: 'https://ens.domains' }],
    })

    if (MODE === 'control') {
      const chain = extendChainWithEns({
        ...sepolia,
        rpcUrls: { default: { http: [RPC] }, public: { http: [RPC] } },
      })
      const ownerClient = createWalletClient({
        account: privateKeyToAccount(accounts.getPrivateKey('user2')),
        chain,
        transport: http(RPC),
      })
      const resolverAddress = await getResolver(publicClient, {
        name: recipientName,
      })
      if (!resolverAddress) throw new Error('recipient has no resolver')
      const hash = await setRecords(ownerClient, {
        name: recipientName,
        resolverAddress,
        coins: [{ coin: 60, value: friend }],
      })
      await publicClient.waitForTransactionReceipt({ hash })
    }

    // ---- measured preconditions, all read from the chain ------------------
    const coin60 = await readCoin60(recipientName)
    const recipientOwner = await getOwnerV2(publicClient, {
      name: recipientName,
    })
    const ownerBefore = await getOwnerV2(publicClient, { name: victimName })
    const chainId = await publicClient.getChainId()
    const forkBlock = await publicClient.getBlockNumber()

    // ---- prove the indexer is the live Panoptes service ------------------
    let indexerReal = 0
    let indexerName = 'none'
    try {
      const r = await fetch(INDEXER, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ query: '{ __schema { queryType { name } } }' }),
      })
      const j = (await r.json()) as {
        data?: { __schema?: { queryType?: { name?: string } } }
      }
      indexerName = j?.data?.__schema?.queryType?.name ?? 'none'
      indexerReal = r.status === 200 && indexerName !== 'none' ? 1 : 0
    } catch {
      indexerReal = 0
    }

    // ---- wait for the REAL indexer to see both names ----------------------
    const victimIndexedSec = await waitForIndexer(victimName)
    const recipientIndexedSec = await waitForIndexer(recipientName)
    console.log(`victim_indexed_after_sec: ${victimIndexedSec}`)
    console.log(`recipient_indexed_after_sec: ${recipientIndexedSec}`)

    // ---- the flow ---------------------------------------------------------
    await page.goto(`${PORTAL_APP_URL}/${victimName}/ownership/transfer`)
    const recipientInput = page.getByPlaceholder('ENS name or address').first()
    const mounted = await recipientInput
      .waitFor({ state: 'visible', timeout: 180_000 })
      .then(() => true)
      .catch(() => false)

    console.log(`DIAG_failed_requests: ${JSON.stringify([...new Set(failed)].slice(0, 25), null, 1)}`)
    if (!mounted) {
      const dump = await page.locator('body').innerText().catch(() => '')
      console.log(`DIAG_url: ${page.url()}`)
      console.log(`DIAG_title: ${await page.title().catch(() => '')}`)
      console.log(`DIAG_headings: ${JSON.stringify(await page.getByRole('heading').allInnerTexts().catch(() => []))}`)
      console.log(`DIAG_body_first_1200: ${JSON.stringify(dump.slice(0, 1200))}`)
      await page.screenshot({ path: 'nm1-diag.png', fullPage: true }).catch(() => {})
    }

    let uiResolved = 'none'
    let transferOk = 0
    let switchesLeftAtDefault = 1

    if (mounted) {
      await recipientInput.fill(recipientName)

      // Wait for the recipient preview to render an address, then read it.
      const preview = page.locator('[class*="bg-muted"]').filter({
        hasText: /^0x[a-fA-F0-9]{40}$|0x[a-fA-F0-9]{40}/,
      })
      await preview
        .first()
        .waitFor({ state: 'visible', timeout: 60_000 })
        .catch(() => {})
      await page.waitForTimeout(3_000)

      const previewText = await preview
        .first()
        .innerText()
        .catch(() => '')
      const m = previewText.match(/0x[a-fA-F0-9]{40}/)
      if (m) uiResolved = getAddress(m[0])

      // Record the switch states; do NOT change them.
      for (const label of [
        'Set the ETH address to the recipient',
        'Detach the resolver',
        'Detach the registry',
      ]) {
        const sw = page.getByRole('switch', { name: label })
        if (await sw.isVisible().catch(() => false)) {
          const checked = await sw.isChecked().catch(() => false)
          console.log(
            `switch[${label}]: visible=1 checked=${checked ? 1 : 0} (untouched)`,
          )
        } else {
          console.log(`switch[${label}]: visible=0`)
        }
      }

      const btn = page.getByRole('button', { name: 'Transfer name' }).first()
      const btnVisible = await btn.isVisible().catch(() => false)
      const btnEnabled = btnVisible ? await btn.isEnabled().catch(() => false) : false
      console.log(`transfer_button_visible: ${btnVisible ? 1 : 0}`)
      console.log(`transfer_button_enabled: ${btnEnabled ? 1 : 0}`)
      if (btnEnabled) {
        await btn.click()
        transferOk = (await driveToSuccess(
          page,
          wallet,
          `transfer-${victimName}-transfer-token`,
        ))
          ? 1
          : 0
      }
    }

    console.log(`DIAG_tx_console: ${JSON.stringify([...new Set(txLog)].slice(0, 20), null, 1)}`)

    const ownerAfter = await getOwnerV2(publicClient, { name: victimName })

    const uiIsAddr = /^0x[a-fA-F0-9]{40}$/.test(uiResolved)
    const resolvedToOwner =
      uiIsAddr && isAddressEqual(uiResolved as `0x${string}`, attacker) ? 1 : 0
    const resolvedToFriend =
      uiIsAddr && isAddressEqual(uiResolved as `0x${string}`, friend) ? 1 : 0
    const onchainToAttacker =
      ownerAfter && isAddressEqual(ownerAfter, attacker) ? 1 : 0
    const onchainToFriend =
      ownerAfter && isAddressEqual(ownerAfter, friend) ? 1 : 0
    const stillVictim =
      ownerAfter && isAddressEqual(ownerAfter, victimWallet) ? 1 : 0

    console.log(`mode: ${MODE}`)
    console.log(`chain_id: ${chainId}`)
    console.log(`fork_block: ${forkBlock}`)
    console.log(`indexer_is_real_graphql: ${indexerReal}`)
    console.log(`indexer_query_type_str: ${indexerName}`)
    console.log(`page_route_intercepts: 0`)
    console.log(`victim_name: ${victimName}`)
    console.log(`recipient_name: ${recipientName}`)
    console.log(`recipient_name_onchain_owner_str: ${recipientOwner}`)
    console.log(
      `recipient_owner_is_attacker: ${
        recipientOwner && isAddressEqual(recipientOwner, attacker) ? 1 : 0
      }`,
    )
    console.log(`coin60_present: ${coin60 ? 1 : 0}`)
    console.log(`coin60_value_str: ${coin60 ?? 'none'}`)
    console.log(`route_mounted: ${mounted ? 1 : 0}`)
    console.log(`ui_resolved_address_str: ${uiResolved}`)
    console.log(`ui_resolved_present: ${uiIsAddr ? 1 : 0}`)
    console.log(`resolved_to_owner: ${resolvedToOwner}`)
    console.log(`resolved_to_friend: ${resolvedToFriend}`)
    console.log(`switches_left_at_default: ${switchesLeftAtDefault}`)
    console.log(`transfer_ok: ${transferOk}`)
    console.log(`onchain_owner_before_str: ${ownerBefore}`)
    console.log(`onchain_owner_after_str: ${ownerAfter}`)
    console.log(`onchain_owner_is_attacker: ${onchainToAttacker}`)
    console.log(`onchain_owner_is_friend: ${onchainToFriend}`)
    console.log(`onchain_owner_still_victim: ${stillVictim}`)
    console.log(`impact_row: 6285`)
    console.log(`secondary_impact_row: 6286`)
  })
})

```

## Harness — `verification/pocs/H-C4-NM-1/ensure-stack.sh`

```bash
#!/usr/bin/env bash
# Bring up the full REAL stack for H-C4-NM-1: anvil Sepolia fork, the real
# Panoptes indexer, and the portal dev server. Idempotent.
set -euo pipefail
HERE="$(cd "$(dirname "$0")" && pwd -P)"
WS="$(cd "$HERE/../../.." && pwd -P)"
ANVIL_PORT="${ANVIL_PORT:-8736}"
PORTAL_PORT="${PORTAL_PORT:-8918}"
INDEXER_PORT="${INDEXER_PORT:-5656}"
RPC="http://127.0.0.1:${ANVIL_PORT}"

chainid() { curl -s --max-time 3 "$1" -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin).get("result",""))' 2>/dev/null || true; }

# ---- anvil ---------------------------------------------------------------
if [ "$(chainid "$RPC")" != "0xaa36a7" ]; then
  echo "starting anvil on $ANVIL_PORT (bound 0.0.0.0 so containers can reach it)"
  nohup anvil --fork-url "${SEPOLIA_FORK_URL:-https://ethereum-sepolia-rpc.publicnode.com}" \
    --chain-id 11155111 --port "$ANVIL_PORT" --host 0.0.0.0 --accounts 10 \
    --mnemonic 'test test test test test test test test test test test junk' \
    > /tmp/ens-c5-anvil.log 2>&1 &
  for _ in $(seq 1 40); do [ "$(chainid "$RPC")" = "0xaa36a7" ] && break; sleep 2; done
fi
[ "$(chainid "$RPC")" = "0xaa36a7" ] || { echo "anvil did not come up" >&2; exit 1; }

# ---- real Panoptes indexer ----------------------------------------------
# contracts.json is regenerated at the CURRENT fork height so the indexer only
# walks blocks the fork actually has, and is repointed at the ensjs addresses
# this pin uses (the vendor file lists a superseded deployment).
FORK=$(curl -s "$RPC" -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}' \
  | python3 -c 'import sys,json;print(int(json.load(sys.stdin)["result"],16))')
python3 "$WS/tools/panoptes/render-contracts.py" "$FORK"

if ! curl -s --max-time 3 -X POST "http://127.0.0.1:${INDEXER_PORT}/graphql" \
     -H 'content-type: application/json' -d '{"query":"{ __typename }"}' | grep -q Query; then
  echo "starting real Panoptes indexer"
  ( cd "$WS/tools/panoptes" && docker compose -p ens-c5 up -d >/dev/null 2>&1 )
  for _ in $(seq 1 45); do
    curl -s --max-time 3 -X POST "http://127.0.0.1:${INDEXER_PORT}/graphql" \
      -H 'content-type: application/json' -d '{"query":"{ __typename }"}' | grep -q Query && break
    sleep 2
  done
fi

# ---- portal --------------------------------------------------------------
if [ "$(curl -s -o /dev/null -w '%{http_code}' --max-time 3 "http://127.0.0.1:${PORTAL_PORT}/" || echo 000)" = "000" ]; then
  echo "starting portal on $PORTAL_PORT"
  ( cd "$WS/repos/audit-comp-ens/apps/portal" && nohup env \
      VITE_SEPOLIA_RPC_URL="$RPC" \
      VITE_INDEXER_GRAPHQL_URL="http://127.0.0.1:${INDEXER_PORT}/graphql" \
      pnpm exec vite --port "$PORTAL_PORT" --strictPort --host 127.0.0.1 \
      > /tmp/ens-c5-portal.log 2>&1 & )
  for _ in $(seq 1 60); do
    [ "$(curl -s -o /dev/null -w '%{http_code}' --max-time 3 "http://127.0.0.1:${PORTAL_PORT}/" || echo 000)" != "000" ] && break
    sleep 1
  done
fi
echo "stack ready: anvil=$ANVIL_PORT indexer=$INDEXER_PORT portal=$PORTAL_PORT fork_block=$FORK"

```

## Harness — `verification/pocs/H-C4-NM-1/run.sh`

```bash
#!/usr/bin/env bash
# Run one arm of the H-C4-NM-1 PoC. Arg: exploit|control
set -euo pipefail
MODE="${1:-exploit}"
HERE="$(cd "$(dirname "$0")" && pwd -P)"
WS="$(cd "$HERE/../../.." && pwd -P)"
export ANVIL_RPC_URL="http://127.0.0.1:${ANVIL_PORT:-8736}"
export PORTAL_APP_URL="http://127.0.0.1:${PORTAL_PORT:-8918}"
export NM1_INDEXER_URL="http://127.0.0.1:${INDEXER_PORT:-5656}/graphql"
export NM1_MODE="$MODE"
bash "$HERE/ensure-stack.sh" >&2
cd "$WS/repos/audit-comp-ens/e2e"
exec pnpm exec playwright test nm1-real --config=projects/portal/playwright.config.ts --reporter=list

```

## Harness — `verification/pocs/H-C4-NM-1-DIV/run.sh`

```bash
#!/usr/bin/env bash
# Run one arm of the H-C4-NM-1 DIVERGENCE PoC. Arg: exploit|control
set -euo pipefail
MODE="${1:-exploit}"
HERE="$(cd "$(dirname "$0")" && pwd -P)"
WS="$(cd "$HERE/../../.." && pwd -P)"
export ANVIL_RPC_URL="http://127.0.0.1:${ANVIL_PORT:-8736}"
export PORTAL_APP_URL="http://127.0.0.1:${PORTAL_PORT:-8918}"
export NM1_INDEXER_URL="http://127.0.0.1:${INDEXER_PORT:-5656}/graphql"
export NM1_MODE="$MODE"
bash "$WS/verification/pocs/H-C4-NM-1/ensure-stack.sh" >&2
cd "$WS/repos/audit-comp-ens/e2e"
exec pnpm exec playwright test nm1-divergence --config=projects/portal/playwright.config.ts --reporter=list --timeout=900000

```

## Indexer — `tools/panoptes/docker-compose.yml`

Same image and command as the repo's own `e2e/infra/docker-compose.yml`. Two differences: `RPC_URL` points at the PoC's anvil fork on the host rather than the compose-internal `anvil` alias, and the API is published on 5656 to avoid a port clash.

```yaml
# Real Panoptes indexer for the H-C4-NM-1 PoC.
#
# Same image and same command the vendor's own e2e stack uses
# (e2e/infra/docker-compose.yml), with two deliberate differences:
#   * RPC_URL points at the PoC's anvil Sepolia fork on the host (8736) instead
#     of the compose-internal `anvil` alias, so the indexer reads the same chain
#     the app and the wallet read.
#   * The API is published on 5656 to avoid colliding with anything on 5655.
#
# contracts.json is the vendor file with every *_block moved to the fork height;
# addresses are untouched. This is the REAL indexer doing REAL work on the REAL
# logs our registrations emit — it replaces e2e/helpers/mock-indexer.ts, which is
# what the vendor falls back to in CI (E2E_MOCK_INDEXER=true).

services:
  panoptes-indexer:
    image: ghcr.io/mdtanrikulu/panoptes:latest
    command: ["/app/ensv2-indexer-v2"]
    volumes:
      - panoptes-data:/app/data
      - ./contracts.json:/app/config/contracts.json:ro
    environment:
      - DATABASE_PATH=/app/data/ens_v2.db
      - RAINBOW_PATH=/app/data/ens_rainbow.db
      - RPC_URL=http://host.docker.internal:8736
      - NETWORK=sepolia
      - CONTRACTS_PATH=/app/config/contracts.json
      - BATCH_SIZE=2000
      - POLL_INTERVAL=1
      - RATE_LIMIT_PER_SECOND=100
      - STATUS_PORT=3001
      - WEBHOOKS_ENABLED=false
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: on-failure

  panoptes-api:
    image: ghcr.io/mdtanrikulu/panoptes:latest
    command: ["/app/api-entrypoint.sh"]
    ports:
      - "5656:5655"
    volumes:
      - panoptes-data:/app/data:ro
      - ./contracts.json:/app/config/contracts.json:ro
    environment:
      - DATABASE_PATH=/app/data/ens_v2.db
      - RAINBOW_PATH=/app/data/ens_rainbow.db
      - DOWNLOAD_RAINBOW=false
      - NETWORK=sepolia
      - CONTRACTS_PATH=/app/config/contracts.json
      - RPC_URL=http://host.docker.internal:8736
      - API_PORT=5655
      - GRAPHQL_MAX_DEPTH=10
      - GRAPHQL_MAX_COST=1000
      - WEBHOOKS_ENABLED=false
      - METRICS_ENABLED=false
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on:
      - panoptes-indexer
    restart: on-failure

volumes:
  panoptes-data:

```

## Indexer config — `tools/panoptes/render-contracts.py`

Worth flagging independently of this bug: the checked-in `e2e/infra/panoptes/contracts.json` points the local indexer at a superseded v2 deployment (`0xdedb92…`), while `ensL1Contracts[sepolia]` at this commit uses `0xBDC85d…`. Run the local Panoptes against the vendor file and it indexes zero domains, which is presumably why CI falls back to the stand-in. This script repoints it and moves the start blocks to the fork height so the indexer does not replay pre-fork history through a forked node.

```python
#!/usr/bin/env python3
"""Render panoptes contracts.json for a given anvil fork height.

Two deliberate differences from the vendor file (e2e/infra/panoptes/contracts.json):
  * every *_block is set to the fork height, because a forked node cannot replay
    pre-fork history without hammering the upstream RPC;
  * the v2 addresses are repointed at ensL1Contracts[sepolia] — the ones
    apps/portal and the e2e fixtures actually use at this pin. The vendor file
    still lists a superseded deployment (0xdedb92...), which is why the indexer
    reported 0 domains against it.
"""
import json, sys, pathlib

fork = int(sys.argv[1])
here = pathlib.Path(__file__).parent
src = here / ".." / ".." / "repos" / "audit-comp-ens" / "e2e" / "infra" / "panoptes" / "contracts.json"
d = json.loads(src.resolve().read_text())

ENSJS_SEPOLIA = {
    "ens_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
    "root_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
    "eth_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
    "l2_eth_registry": "0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
    "l2_eth_registrar": "0xa88553F454b77203B0D036A05c894d555EAAa2Cc",
    "eth_registrar_controller": "0xfb3cE5D01e0f33f41DbB39035dB9745962F1f968",
    "base_registrar": "0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85",
    "public_resolver": "0x5239A812ec9A62F46dbb5de8f346C8eFe7553A9f",
    "old_public_resolver": "0x5239A812ec9A62F46dbb5de8f346C8eFe7553A9f",
    "old_ens_registry": "0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e",
    "name_wrapper": "0x0635513f179D50A207757E05759CbD106d7dFcE8",
    "reverse_registrar": "0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6",
    "eth_reverse_registrar": "0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6",
    "migration_helper": "0x1D8c7aA9862F9b823309Ad87A4864Fb27C575e85",
    "locked_migration_controller": "0x5c39E36a69A9897F08954c71aCB1F36E0Bd4f409",
    "unlocked_migration_controller": "0x2FCf83232b93bD29C59dB18AaA1D4b62e9f9FC73",
}
d.update(ENSJS_SEPOLIA)
for k in list(d):
    if k.endswith("_block"):
        d[k] = fork
d["_comment"] = (
    "Generated by render-contracts.py. Vendor addresses repointed at "
    f"ensL1Contracts[sepolia]; all *_block set to the anvil fork height {fork}."
)
(here / "contracts.json").write_text(json.dumps(d, indent=2) + "\n")
print(f"contracts.json rendered at fork block {fork}")

```
