# 93090 w a critical explorer send name resolves a recipient ens name with no address record to the name s registry owner instead of failing

**Submitted on Sep 14th 2026 at 08:12:06 UTC by @abarbatei for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93090
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer shares one name-or-address resolution hook between the role "add user" sheets and the name-transfer recipient field. The underlying helper, written for role grants, falls back to the name's registry owner when the universal resolver returns no address. In the transfer flow that means typing `bob.eth` when `bob.eth` has no ETH address record silently pre-fills the address of whoever currently holds `bob.eth`, which may be a custodial or escrow contract, a subregistry, a smart account, or a wallet Bob does not use for receiving. The preview shows that address without saying resolution failed, the form accepts it, and `safeTransferFrom` sends the name there. The same fallback sets the owner of a newly created subname.

## Vulnerability Details

Intended behavior: when a user enters an ENS name as a recipient, the destination must be that name's address record. A name without an address record is unresolvable; every wallet treats it that way.

Root cause: a role-grant convenience reused as a transfer resolver.

1. [`addUser.handlers.ts::resolveAddressOrName`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/roles/helpers/addUser.handlers.ts#L11-L40):

```ts
    const resolved = await getEnsAddress(client, { name: nameOrAddress, universalResolverAddress })
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

The comment scopes the fallback to role grants. `getEnsOwner` resolves the v2 registry owner and falls back to the v1 owner.

2. [`getResolvedAddress.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/address/queries/getResolvedAddress.ts#L16-L29) wraps that helper, and [`useAddressResolution.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/address/hooks/useAddressResolution.ts#L45-L90) is documented as "shared by every name-or-address input in the app (transfer recipient, add-user sheets, …)". A fallback owner comes back as `status: 'resolved'`.
3. [`SendNameForm.tsx` lines 80-110](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/transfer/components/SendNameForm.tsx#L80-L110) only rejects self-transfer and the zero address:

```tsx
  const { address: recipient, isResolving } = resolution
  const isSelf = !!recipient && isAddressEqual(recipient, owner)
  const isZeroAddress = !!recipient && isAddressEqual(recipient, zeroAddress)
  const hasValidRecipient = !!recipient && !isSelf && !isZeroAddress
```

`RecipientResolvedContent` (lines 228-262) renders the address and its primary name with no indication that it came from ownership rather than resolution.

4. [`transferToken.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/transfer/helpers/transferToken.ts#L60-L72) encodes `safeTransferFrom(sender, recipient, tokenId, 1, '0x')` on the leaf registry.
5. [`create-subname.tsx#L109`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/routes/$name/create-subname.tsx#L109) uses the same hook for the new subname's owner, so a typed name without an address record makes the *holder* of that name the owner of the new subname.

Note on scope of the fallback: viem's `getEnsAddress` returns `null` only for universal-resolver revert classes (no resolver, resolver not a contract, wildcard unsupported); a CCIP gateway error is thrown and caught, returning `null` without the owner fallback. So the substitution needs a genuinely missing address record or a name with no resolver. Both are common: a fresh registration before records are set, or a name whose previous owner used this very transfer flow's "detach the resolver" option.

Sequence:

1. Victim opens Send name for a name they own and types `bob.eth`.
2. `bob.eth` has no addr(60) record. `getEnsAddress` returns null; the helper returns the registry owner of `bob.eth`.
3. The form shows that address as the resolved recipient. The victim, trusting the app's resolution, confirms.
4. The wallet signs `safeTransferFrom(victim, ownerOfBob, tokenId, 1)`.
5. If the holder of `bob.eth` is an ERC-1155-capable contract (escrow, marketplace vault, smart account) or a wallet Bob does not control for receiving, the name is gone.

## Impact Details

* The app pre-fills a recipient that is not the typed name's address record and presents it as a normal resolution. This is the program's "display or pre-fill an incorrect address ... so that a user submits a transaction to an unintended recipient" concern.
* No attacker can force another person's name to lose its address record, so the exposure is opportunistic: names that never had a record or whose resolver was detached. That bounds the severity to Low.
* `safeTransferFrom` reverts for contracts without a receiver hook, so only receiver-capable holders can strand the name; EOAs and smart accounts always accept.
* The subname-owner variant grants ownership of a new subname to the holder of a typed name rather than its address.

## Mitigation

* Add a `fallbackToOwner` option to `resolveAddressOrName`, default `false`, and enable it only in the three add-user sheets.
* In `useAddressResolution` consumers that move assets (transfer recipient, subname owner), treat a null address as `unresolved` and say "this name has no ETH address record".
* If the owner fallback is ever shown to a user, label it explicitly as the name's owner.

## References

* [`addUser.handlers.ts::resolveAddressOrName`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/roles/helpers/addUser.handlers.ts#L11-L40): owner fallback.
* [`useAddressResolution.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/address/hooks/useAddressResolution.ts#L45-L90): shared hook.
* [`SendNameForm.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/transfer/components/SendNameForm.tsx#L80-L110): acceptance and preview.
* [`transferToken.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/features/transfer/helpers/transferToken.ts#L60-L72): `safeTransferFrom` encoding.
* [`create-subname.tsx#L109`](https://github.com/immunefi-team/audit-comp-ens/blob/63772fd872af472ced58b009499355f3430c2a86/apps/portal/src/routes/$name/create-subname.tsx#L109): same hook for the subname owner.

## Proof of Concept

## Proof of Concept

A Vitest test in the Explorer app calls the real `resolveAddressOrName` with `getEnsAddress` stubbed to return `null` (no address record) and `getEnsOwner` stubbed to return a distinct owner address, asserts the owner is returned as the "resolved" recipient, then passes that recipient to the real `transferToken`, capturing the intent handed to the transaction manager and decoding `safeTransferFrom(victim, owner, tokenId, 1)`.

Audited revision `63772fd872af472ced58b009499355f3430c2a86`; the cited files are unchanged in the audit-comp-ready branch. Node 22, pnpm 10, the repo's existing install and the Explorer `vitest.config.ts`.

Place the file at:

```
apps/portal/test/poc/M07/ProofTransferRecipientOwnerFallback.test.ts
```

Run from the repository root:

```bash
pnpm install --frozen-lockfile
pnpm --filter portal exec vitest run --config vitest.config.ts test/poc/M07/ProofTransferRecipientOwnerFallback.test.ts
```

Expected: 1 test file passed, 1 test passed, exit code 0.

### Proof source

```ts
import { ok } from 'neverthrow'
import { describe, expect, it, vi } from 'vitest'
import { type Address, decodeFunctionData, erc1155Abi, type Hex } from 'viem'
import { sepolia } from 'viem/chains'

const VICTIM = '0x1111111111111111111111111111111111111111' as Address
const BOB_OWNER = '0x2222222222222222222222222222222222222222' as Address
const REGISTRY = '0x4444444444444444444444444444444444444444' as Address

const startTransaction = vi.hoisted(() => vi.fn(() => 'transfer-tx'))
vi.mock('viem/actions', () => ({
  // bob.eth has no addr(60) record: the universal resolver returns null.
  getEnsAddress: vi.fn(async () => null),
}))
vi.mock('@/features/profile/hooks/useEnsOwner', () => ({
  // The registry owner of bob.eth is some other account (here: a contract/escrow).
  getEnsOwner: vi.fn(async () => ok({ owner: BOB_OWNER, registryAddress: REGISTRY, protocolVersion: 'ENSv2' })),
}))
vi.mock('@ens-apps/transaction-manager', () => ({
  transactionManager: { startTransaction },
  waitForTransaction: vi.fn(async () => ({ hash: `0x${'11'.repeat(32)}` })),
}))

import { resolveAddressOrName } from '@/features/roles/helpers/addUser.handlers'
import { transferToken } from '@/features/transfer/helpers/transferToken'

describe('M07 Send-name recipient falls back to the registry owner', () => {
  it('encodes safeTransferFrom to the owner of bob.eth instead of failing resolution', async () => {
    const recipient = await resolveAddressOrName({ client: {} as never, nameOrAddress: 'bob.eth' })
    console.log('1. resolveAddressOrName("bob.eth") with no addr record ->', recipient)
    expect(recipient).toBe(BOB_OWNER)

    // SendNameForm only rejects self and zero address, so this passes hasValidRecipient
    // and reaches transferToken with the fallback address.
    await transferToken({
      name: 'victimname.eth',
      registryAddress: REGISTRY,
      tokenId: 42n,
      recipient: recipient as Address,
      walletClient: { account: { address: VICTIM }, chain: sepolia } as never,
      publicClient: {} as never,
      signer: { type: 'eoa', walletClient: {} } as never,
      chainId: sepolia.id,
      id: 'transfer-victimname',
    })
    const [intent] = startTransaction.mock.calls[0] as unknown as [{ request: { to: Address; data: Hex } }]
    const decoded = decodeFunctionData({ abi: erc1155Abi, data: intent.request.data })
    console.log('2. wallet request:', decoded.functionName, decoded.args)
    expect(intent.request.to).toBe(REGISTRY)
    expect(decoded.functionName).toBe('safeTransferFrom')
    expect(decoded.args?.[0]).toBe(VICTIM)
    expect(decoded.args?.[1]).toBe(BOB_OWNER)
    console.log('3. name goes to the OWNER of bob.eth, not to an address bob.eth resolves to')
  })
})
```
