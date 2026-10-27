# 92585 w a medium explorer certifies any contract as the official audited ens permissioned resolver from a single storage word the contract writes itself

**Submitted on Sep 10th 2026 at 19:05:04 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92585
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Explorer has exactly one signal that tells a user whether a resolver can be trusted, and it is a sentence:

```tsx
// apps/portal/src/features/resolver/components/ResolverTypeValue.tsx:12-19
  <p className="text-p text-foreground">
    This is an instance of the official{' '}
    <ExternalLink href={docsHref} className="underline">{name}</ExternalLink>
    . It is audited and is considered secure.
  </p>
```

Rendered when, and only when, `useIsPermissionedResolver` says so (`:47-53`). The same sentence appears on the name's Resolver tab (`apps/portal/src/routes/$name/resolver.tsx:176-182`). Anything else is labelled "Custom Resolver".

The decision behind that sentence is one `eth_getStorageAt`:

```tsx
// apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts:46-66
  const implementationSlotValue = yield* fromPromise(
    getStorageAt(client, {
      address: params.resolverAddress,
      slot: EIP1967_IMPLEMENTATION_SLOT,
    }),
    …
  )
  const implementationAddress = decodeImplementationAddress(implementationSlotValue)
  if (!implementationAddress) return ok(false)
  return ok(
    knownPermissionedResolverImplementations.includes(
      implementationAddress.toLowerCase(),
    ),
  )
```

The contract's runtime code is never fetched. `EXTCODEHASH` is never compared. `supportsInterface` is never called — the interface id exists in the codebase (`apps/portal/src/lib/constants/resolverInterfaceIds.ts:15`) but only feeds a cosmetic "Interfaces" row. Nothing establishes that the address is a proxy, that it delegates anywhere, or that it behaves like the implementation it names.

`eth_getStorageAt` returns whatever the contract put in its own storage. A contract does not have to be a proxy to have a value in the EIP-1967 slot; one `sstore` in a constructor is enough. So the predicate answers *"is this contract willing to claim it is the official implementation"*, and the app renders that claim as ENS's own assurance that the contract is audited and secure.

***

## Impact

Precondition: none on the victim's side beyond opening a resolver page — which is the check a careful user performs before trusting a resolver, and the reason the badge exists.

1. The attacker deploys a contract whose constructor contains a single `sstore` of the official `PermissionedResolver` implementation address into `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc`. The contract is not a proxy and delegates to nothing; its `addr(bytes32,uint256)` returns the attacker's payout address for every node.
2. The attacker gets a name the victim will look at pointed at that contract. The cheapest route needs nothing from the victim: run an ordinary subname service on a 2LD the attacker owns and mint subnames whose resolver is the contract. The variant that costs the victim directly: persuade them to paste the address into the Explorer's own **Change resolver** form, where the same badge is what they will use to check it.
3. The victim opens `/<name>/resolver`, or the standalone `/resolver/<address>` page, and reads: *This is an instance of the official ENS Permissioned Resolver. It is audited and is considered secure.* — with a link to the real `PermissionedResolver.sol` in the ENS repository.
4. Every address that resolver returns is the attacker's, and the app has told the victim, in ENS's own words, that the contract answering is audited.

The badge is the whole of the app's resolver-safety story. There is no second check, no warning, and no way for a user to distinguish a genuine factory-deployed proxy from a contract that merely says it is one: the only difference is in code the app never reads.

***

## The flag is not display-only — it switches the record-edit authorization model

The demotion this finding must survive is the one applied to #89441: *"the badge is confined to a read-only table that no transaction path consumes."* That is not true here. The same boolean decides who the app believes may edit a name's records:

```tsx
// apps/portal/src/features/records/hooks/useCanEditRecords.ts:61-66
    ...getIsPermissionedResolverQueryOptions({
      resolverAddress: (resolverAddress ?? '0x0000…0000') as Address,
    }),
// :107-115
  const canEdit = (() => {
    if (!enabled || !connectedAddress || isLoading || !resolverAddress) return false

    if (isPermissioned) return hasResolverRole

    // Public / non-permissioned resolvers: token owner can edit.
    return isOwner
```

When the forged flag is `true`, authorization stops being "are you the token owner" and becomes "does the attacker's contract say you hold a role" — `hasResolverRole`, queried against the attacker's contract, which returns whatever the attacker wants. `canEdit` gates `apps/portal/src/routes/$name/records.tsx:35, 68` and `apps/portal/src/routes/$name/edit-records.tsx:75, 143`.

So a single forged storage word both wears the trust badge and flips the app's record-edit gate from on-chain ownership to an attacker-defined predicate.

## The same defect is duplicated server-side

`apps/portal/src/worker/ens.ts:105` is, by its own docblock at `:100`, a *"Worker-side mirror of `useIsPermissionedResolver`"* — the same storage read and the same membership test, in the worker that renders the OG social card. The forged claim therefore also propagates outside the app, into link previews.

## The correct check already exists in the same module

`apps/portal/src/features/resolver/utils/permissionedResolver.ts:95-105` implements `filterPermissionedResolverAddresses(logs, expectedImplementation)`, which establishes provenance from the `VerifiableFactory` `ProxyDeployed` event's `implementation` field. The codebase already owns the sound primitive; the badge simply does not use it.

***

## Root cause

| File:line                                                                    | Code                                                                    |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts:13-14` | `EIP1967_IMPLEMENTATION_SLOT`                                           |
| `apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts:46-66` | one `getStorageAt`, decode, membership test — the entire decision       |
| `apps/portal/src/features/resolver/components/ResolverTypeValue.tsx:33-53`   | the hook's boolean gates the audited notice                             |
| `apps/portal/src/routes/$name/resolver.tsx:176-182`                          | the same sentence on the name's Resolver tab                            |
| `apps/portal/src/lib/constants/resolverInterfaceIds.ts:15`                   | the `supportsInterface` id exists and is not used for the type decision |

What a correct check would need, and what is absent:

* the contract's code (`eth_getCode` / `EXTCODEHASH` against the known EIP-1167 or ERC-1967 proxy runtime),
* an `implementation()` call, or any call at all,
* `supportsInterface`,
* provenance — the `VerifiableFactory` `ProxyDeployed` event that actually attests a factory deployment.

***

## Preempting the likely closure arguments

**"The user has to point their name at the attacker's resolver."** For the subname variant they do not: an attacker running a subname service sets the resolver themselves, and the victim only opens the page. For the change-resolver variant the badge is not incidental to the attack — it is the check the victim performs *instead of* reading the contract, and it is the one the app offers them.

**"The badge is confined to a read-only surface."** This is the exact rationale ENS used to demote #89441, and it does not transfer: `useCanEditRecords.ts:113` consumes the same flag to choose the authorization model for record editing. See the section above.

**"It is only a label."** It is the only resolver-trust signal in the product, it is phrased as an ENS assurance rather than a heuristic, and it links to the audited source as if that were the code answering. A user who reads *audited and considered secure* on ENS's own Explorer has been given the app's strongest possible statement about a contract that is not what it claims.

**"The resolver contracts are out of scope."** No contract behaviour is being reported. The finding is the Explorer's classification logic — `apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts` — and the copy it gates.

**"A user could check the code themselves."** The Explorer's own Contract row links to the block explorer, but the badge is rendered above it and states the conclusion. That is what a trust badge is for; if users were expected to verify independently, the sentence would not be there.

**Recoverability.** The display corrects itself only if the resolver is changed. Any funds sent to an address the certified resolver returned are gone.

***

## Not a duplicate

* **#89441** (`getName(..., { allowMismatch: true })` badging an unverified reverse record) is the closest by theme. Its fix threads the `match` verdict through `useReverseResolution.ts:236-241` and makes `columns.tsx:96-107` read `row.original.forwardMatch`; it does not change a single line of `useIsPermissionedResolver.ts`, and this report's fix does not change a line of the reverse-resolution table. Different hook, different data source, different primitive: #89441 discards a verification result the library returned, this one infers contract identity from state the contract controls.
* **#91306** and **#92542** are role-scope defects inside the roles sidebars.
* **#91851** (`AddressResolutionSidebar` offering "Set primary name" with no authority check) is a missing authorization check on an action, not a contract-identity claim.
* **#92544** (`RegistrationSuccessBanner` trusting URL search params) takes its false claim from the URL; this one takes it from chain state the attacker controls.
* No known issue touches resolver type detection. EXP-INPUT-005 is validator leniency on typed names; WEB-716 is the registry page's deploy-subregistry affordance.

***

## Recommendation

1. Verify the code, not the claim. Compare `EXTCODEHASH` of the resolver address against the known ERC-1967 / EIP-1167 proxy runtime hashes, and only then read the implementation slot.
2. Prefer provenance. The `VerifiableFactory` emits `ProxyDeployed`; a resolver the app is willing to call official should be one that event attests, which is also how `permissionedResolver.ts` already discovers deployments.
3. Call something. `supportsInterface(0x91413117)` is already a constant in the repo; a contract that cannot answer it is not the implementation.
4. Soften the copy until the check is sound. "Reports the official implementation in its EIP-1967 slot" is a true statement; "audited and considered secure" is not one the app can make from a storage read.
5. Apply the same scepticism to `/registry/$address`, which renders a hardcoded `PermissionedRegistry` type badge for whatever address is in the URL (`routes/registry/$address/index.tsx:94-96`) without verifying the contract at all — the factory address is imported at `:33-36` but is not used to attest the address being displayed.

## Proof of Concept

## Proof of Concept

Executed against a local Anvil fork of Sepolia. No live network was touched. The verdict comes from the app's own function.

### Environment

```bash
anvil --fork-url <sepolia> --chain-id 11155111 --port 8545
```

```
ensPermissionedResolverImpl  0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
```

### The app's own predicate, before and after one storage write

`apps/portal/.poc/fake-resolver.test.ts` Run it with:

```bash
cd apps/portal && ./node_modules/.bin/vitest run .poc/fake-resolver.test.ts --reporter=verbose
```

```tsx
import { createPublicClient, http } from 'viem'
import { it } from 'vitest'
import { getIsPermissionedResolver } from '@/features/resolver/hooks/useIsPermissionedResolver'
import { sepoliaWithEns } from '@/lib/wagmi'

const EIP1967_IMPL_SLOT = '0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc'
const REAL_IMPL = '0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e'
const FAKE = '0x00000000000000000000000000000000000fa4e0'

it('any contract that writes the impl address into its own EIP-1967 slot is certified', async () => {
  const client = createPublicClient({ chain: sepoliaWithEns, transport: http('<http://127.0.0.1:8545>') })

  // A contract with runtime code of the attacker's choosing. The body is
  // irrelevant to the app's decision — only the storage word is read.
  await client.request({ method: 'anvil_setCode', params: [FAKE, '0x60006000f3'] })
  const before = await getIsPermissionedResolver({ resolverAddress: FAKE })
  console.log('  app verdict    :', before.value)

  // Exactly what `assembly { sstore(0x360894a1…, 0x9EAe5C27…) }` in a constructor leaves.
  const word = `0x${'00'.repeat(12)}${REAL_IMPL.slice(2).toLowerCase()}`
  await client.request({ method: 'anvil_setStorageAt', params: [FAKE, EIP1967_IMPL_SLOT, word] })

  const after = await getIsPermissionedResolver({ resolverAddress: FAKE })
  console.log('  contract code  :', await client.getCode({ address: FAKE }))
  console.log('  app verdict    :', after.value)
})
```

Actual output:

```
before the sstore:
  storage slot   : 0x0000000000000000000000000000000000000000000000000000000000000000
  app verdict    : false

after one sstore of the official implementation address:
  storage slot   : 0x0000000000000000000000009eae5c2730a7dd16bdd1dee6421a1b91e3b0365e
  contract code  : 0x60006000f3 (NOT a proxy, NOT the real implementation)
  app verdict    : true   <- rendered as the official, audited ENS Permissioned Resolver

 ✓ .poc/fake-resolver.test.ts  793ms
```

Five bytes of runtime code — `PUSH1 0x00 PUSH1 0x00 RETURN` — and one storage word. The app's own `getIsPermissionedResolver` returns `true`, which is the exact condition `ResolverTypeValue.tsx:47` uses to print *"It is audited and is considered secure."*

`anvil_setStorageAt` is used here only to place the state without deploying; on a real chain a constructor `sstore` leaves the identical state, and the app reads nothing else.

***
