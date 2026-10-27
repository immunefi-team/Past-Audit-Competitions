# 92648 w a medium the explorer certifies a resolver as official audited and considered secure from a storage word any contract can forge and the same forged bit replaces the record edit owners&#x20;

**Submitted on Sep 11th 2026 at 08:05:48 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92648
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Summary

The Explorer tells users that a resolver contract **"is audited and is considered secure"** on the strength of a single storage word that the contract itself writes. There is no bytecode check anywhere on that path — no `getCode`, no `getBytecode`, no EXTCODEHASH, and no proof the address is an ERC-1967 proxy or delegatecalls anything.

Any contract whose constructor runs `sstore(0x360894…bbc, OFFICIAL_IMPL)` earns the badge, at the cost of one SSTORE, while its code does whatever the attacker wants.

The same forged bit also flips the record-editing authorization model from "you own this name" to "this contract says you may".

## The verdict

`apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts:44-66`:

```ts
  const implementationSlotValue = yield* fromPromise(
    getStorageAt(client, {
      address: params.resolverAddress,
      slot: EIP1967_IMPLEMENTATION_SLOT,
    }),
    (error) =>
      new IsPermissionedResolverError({
        cause: error,
      }),
  )

  const implementationAddress = decodeImplementationAddress(
    implementationSlotValue,
  )
  if (!implementationAddress) return ok(false)

  return ok(
    knownPermissionedResolverImplementations.includes(
      implementationAddress.toLowerCase(),
    ),
  )
```

`eth_getStorageAt` is pure storage introspection. It reports what is in a slot; it says nothing about the code that reads it. And the decoder simply slices the low 20 bytes of whatever came back — `apps/portal/src/features/resolver/utils/permissionedResolver.ts:78-79`:

```ts
  const normalized = storageValue.slice(2).padStart(64, '0')
  const rawAddress = `0x${normalized.slice(24)}` as Address
```

Writing that slot confers no proxy behaviour. A contract is free to store the official implementation address and then implement every resolver method itself, however it likes.

## The claim it backs

`apps/portal/src/features/resolver/components/ResolverTypeValue.tsx:13-19`:

```tsx
  <p className="text-p text-foreground">
    This is an instance of the official{' '}
    <ExternalLink href={docsHref} className="underline">
      {name}
    </ExternalLink>
    . It is audited and is considered secure.
  </p>
```

rendered at `:47-53` whenever the forgeable verdict is true:

```tsx
  if (isPermissionedResolver)
    return (
      <AuditedNotice
        name="ENS Permissioned Resolver"
        docsHref="https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/resolver/PermissionedResolver.sol"
      />
    )
```

## The same component proves the correct check was available

Two branches later, `ResolverTypeValue.tsx:55`:

```tsx
  if (isAddressEqual(resolverAddress, officialPublicResolverAddress))
```

The **Public** Resolver is attested by a real identity comparison against a known address. The **Permissioned** Resolver is attested by a value the subject of the attestation controls. Same component, same sentence of user-facing copy, two completely different standards of proof.

The codebase also already knows how to establish this properly: `apps/portal/src/features/resolver/hooks/useUserPermissionedResolvers.ts` enumerates genuine resolvers from the VerifiableFactory's `ProxyDeployed` logs filtered on `implementation`, and `permissionedResolver.ts` documents that the factory's CREATE2 address is bound to the deployer. The badge path uses neither.

## The authorization flip

`apps/portal/src/features/records/hooks/useCanEditRecords.ts:106-117`:

```ts
  const hasResolverRole = roleQueries.some((query) => query.data === true)

  const canEdit = (() => {
    if (!enabled || !connectedAddress || isLoading || !resolverAddress) {
      return false
    }

    if (isPermissioned) return hasResolverRole

    // Public / non-permissioned resolvers: token owner can edit.
    return isOwner
  })()
```

`hasResolverRole` is read by calling `hasRoles(...)` **on the resolver address under test**. So when the forged bit is set, the app stops asking "does this wallet own the name" and starts asking the attacker's contract, which can answer `true` for everyone. The counterfactual is the point: without the forged word, `canEdit` for a non-owner is `false`.

## Impact

The badge is the application's only safety verdict on the contract that decides **where a name resolves**. A user choosing a resolver through the app's own "Use custom resolver" field (`ChangeResolverForm.tsx:244-249`, validated only by `isAddress`) can paste an address, be told by ENS's Explorer that it is the official, audited, secure Permissioned Resolver, and point their name at it. From that moment the attacker controls what that name resolves to — including its ETH address record, so anyone paying that name pays the attacker.

The attacker's costs are: deploy one contract, and share a link to `/resolver/0xEVIL` or to a name they already pointed at it. The route param is unvalidated (`routes/resolver/$address/index.tsx`), so the page renders for any address.

Secondarily, the forged bit makes `/$name/records` offer an "Edit records" affordance to visitors who own nothing, and the resulting wallet signature is sent to `to: resolverAddress` — attacker code the application has just vouched for.

I want to be precise about what I am and am not claiming. I am **not** claiming the storage write gives the attacker any on-chain privilege; it gives them nothing except the app's endorsement. The harm is entirely that ENS's own interface asserts a security property about a contract it never checked, and that a user acting on that assertion hands over control of their name's resolution.

## The same forgeable read also runs server-side

This is not only client-side rendering. `apps/portal/src/worker/ens.ts:104-133` carries a second copy, and its own doc comment names the relationship:

```
 * Determine whether a resolver is an ENS Permissioned Resolver, so the OG card
 * can render the "Permissioned Resolver" subtitle.
 *
 * Worker-side mirror of {@link useIsPermissionedResolver}: read the EIP-1967
 * implementation slot and compare against the known permissioned-resolver
 * implementation for the chain.
```

```ts
    const slotValue = await getStorageAt(client, {
      address: address as Address,
      slot: EIP1967_IMPLEMENTATION_SLOT,
    })

    const implementation = decodeImplementationAddress(slotValue)
    if (!implementation) return false

    return implementation.toLowerCase() === knownImpl
```

It imports the very same `decodeImplementationAddress` from `@/features/resolver/utils/permissionedResolver`, so both paths are one root cause with one fix. The consequence is that the attacker's forged attestation is also emitted by ENS's own Worker into the Open Graph card for the name — the artefact that is rendered by third parties (chat and social previews) and served from the edge cache, where the viewer never loads the app at all and has nothing to inspect.

## Severity

**Critical**, under "causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user … submits a transaction to an unintended recipient", whose own example is "resolving or rendering an ENS name to an attacker-controlled address" — which is precisely the end state when a user adopts the endorsed resolver. It is also "malicious interactions with an already-connected wallet": the edit-records signature is offered, and directed at attacker code, because of the forged bit.

If the panel prefers to require a completed fund transfer inside the app, the conservative floor is High as persistent content injection — the attacker persists a false security attestation about a contract of their choosing onto ENS's own pages.

## Why this is not an already-known or already-reported issue

**Not in the published known issues.** Searching the document for `audited`, `attest`, `1967`, `getStorageAt`, `storage slot`, `implementation`, `proxy` and `custom resolver` returns no matching entry. The nearest items are the OG-image SSRF (R2-04, a different worker function) and the address view's misleading owner/expiry metadata (WEB-1301, genuine data presented confusingly — here the data is fabricated by a third party).

**Not the resolver/role revocation family.** Nothing is revoked, restored, or left stale. This is resolver *identity attestation* plus the records authorization model — a read path that makes a claim, not a write path that under-delivers.

**Not a normalization issue.** No name is parsed anywhere in this finding; the attacker's contract address is an ordinary address and the victim's name is ordinary.

**Not `ensureOwnedPermRes`.** That helper pins both the expected salt and the implementation when scanning factory logs — it is the check this path is missing, not an instance of the same bug.

## Recommended fix

Prove identity, not storage. Either:

* resolve the resolver through the VerifiableFactory's `ProxyDeployed` logs filtered on `implementation` — the method `useUserPermissionedResolvers.ts` already uses — and treat anything not found there as a custom resolver; or
* verify the deployed bytecode matches the known proxy bytecode (EXTCODEHASH against a pinned hash) before reading the slot, so that storing the word is not sufficient.

Until identity is proven, the copy should not assert that the contract is audited or secure. A contract that merely *claims* to be a Permissioned Resolver should render as "Custom Resolver", which is the existing fallback.

Independently, `useCanEditRecords` should not delegate authorization to an unverified contract: the `isOwner` path should remain a required condition, not an alternative one.

## Proof of Concept

## What this proves

The test drives the REAL exported `decodeImplementationAddress` from `features/resolver/utils/permissionedResolver.ts`, reproduces the hook's own membership check verbatim, and then asserts against the repository's own source that no code check exists anywhere on the path. Five assertions:

1. **the forgery** — a contract that merely STORES the official implementation address in the EIP-1967 slot is declared official;
2. **the verdict is a pure function of that word** — flipping only the stored value flips the verdict; the address under audit never enters the computation;
3. **no bytecode check exists** — the hook contains `getStorageAt` and none of `getBytecode`, `getCode`, `extcodehash`, `EXTCODEHASH`;
4. **the asymmetry** — the same component attests the Public Resolver with `isAddressEqual(resolverAddress, officialPublicResolverAddress)` while attesting the Permissioned Resolver with the forgeable read, under the identical "It is audited and is considered secure." copy;
5. **the authorization flip** — `useCanEditRecords` contains `if (isPermissioned) return hasResolverRole`, replacing its `return isOwner` fallback.

## Environment

Frozen revision under test: 1c9b47f. Nothing is deployed and no live service is contacted. No contract is deployed: assertion 1 substitutes the value `eth_getStorageAt` would return, which is exactly the attacker’s input to the verdict.

```
pnpm install
```

Save the file below as:

```
apps/portal/src/features/resolver/poc-forged-audited-resolver-badge.test.ts
```

Run it:

```
cd apps/portal
pnpm vitest run src/features/resolver/poc-forged-audited-resolver-badge.test.ts
```

## Result

```
RUN  v4.1.5 .../apps/portal

  [forged]   arbitrary contract storing OFFICIAL_IMPL -> verdict TRUE
  [pure]     verdict flips on the stored word alone; bytecode is never read
  [nocheck]  getStorageAt only — no getCode/getBytecode/EXTCODEHASH
  [asymmetry] Public branch uses isAddressEqual; Permissioned branch uses storage
  [authflip] isPermissioned=true => canEdit is whatever the contract answers

 Test Files  1 passed (1)
      Tests  5 passed (5)
```

## The attacker contract, for completeness

The forged slot needs no proxy machinery at all — one constructor line:

```
contract EvilResolver {
    constructor(address officialImpl) {
        assembly {
            sstore(0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc, officialImpl)
        }
    }
    // ... every resolver method implemented however the attacker likes,
    // including hasRoles(...) returning true for all callers.
}
```

Nothing delegatecalls `officialImpl`. The slot is inert storage that only the Explorer reads.

## Reproducing in the UI

1. Deploy the contract above with `officialImpl` = the chain’s `ensPermissionedResolverImpl`.
2. Open `/resolver/<evil address>` on the Explorer — the route param is unvalidated, so no setup on a name is required. The Type row reads “This is an instance of the official ENS Permissioned Resolver. It is audited and is considered secure.”
3. For the authorization half, point a name at the contract (`/<name>/change-resolver` → “Use custom resolver” accepts any address passing `isAddress`) and open `/<name>/records` from an unrelated wallet: with `hasRoles` returning true, “Edit records” is offered and the save targets the contract.

## Verifying the call sites

```
grep -n "getStorageAt" -B 4 -A 18 apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts
grep -nE "getBytecode|getCode|extcodehash" apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts
grep -n "decodeImplementationAddress" -A 16 apps/portal/src/features/resolver/utils/permissionedResolver.ts
grep -n "audited and is considered secure" -B 8 apps/portal/src/features/resolver/components/ResolverTypeValue.tsx
grep -n "isPermissionedResolver|isAddressEqual" apps/portal/src/features/resolver/components/ResolverTypeValue.tsx
grep -n "if (isPermissioned) return hasResolverRole" -B 6 -A 4 apps/portal/src/features/records/hooks/useCanEditRecords.ts
```

The second command returns nothing — that absence is the finding.

## Full test file

/\*\*

* PoC — the Explorer tells users a resolver contract "is audited and is
* considered secure" on the strength of a single storage word that any
* contract can write, and the same forged word flips the record-editing
* authorization model from token-ownership to trusting the contract itself.
*
* THE VERDICT — `features/resolver/hooks/useIsPermissionedResolver.ts:44-66`:
*
* const implementationSlotValue = yield\* fromPromise(
* ```
  getStorageAt(client, {
  ```
* ```
    address: params.resolverAddress,
  ```
* ```
    slot: EIP1967_IMPLEMENTATION_SLOT,
  ```
* ```
  }), ... )
  ```
* const implementationAddress = decodeImplementationAddress(implementationSlotValue)
* if (!implementationAddress) return ok(false)
* return ok(
* ```
  knownPermissionedResolverImplementations.includes(
  ```
* ```
    implementationAddress.toLowerCase(),
  ```
* ```
  ),
  ```
* )
*
* `eth_getStorageAt` is pure storage introspection. There is no `getBytecode`,
* no `getCode`, no EXTCODEHASH, and no check that the address is an ERC-1967
* proxy or delegatecalls anything. A contract whose constructor runs
* `sstore(0x360894…bbc, OFFICIAL_IMPL)` and whose code is otherwise arbitrary
* satisfies it. Writing that slot costs one SSTORE and confers no proxy
* behaviour whatsoever.
*
* THE SAFETY CLAIM — `features/resolver/components/ResolverTypeValue.tsx:13-19,47-53`:
*
* . It is audited and is considered secure.
* ...
* if (isPermissionedResolver)
* ```
  return (
  ```
* ```
    <AuditedNotice
  ```
* ```
      name="ENS Permissioned Resolver"
  ```
* ```
      docsHref="https://github.com/ensdomains/contracts-v2/.../PermissionedResolver.sol"
  ```
* ```
    />
  ```
* ```
  )
  ```
*
* THE ASYMMETRY, two branches later in the same component (`:55`):
*
* if (isAddressEqual(resolverAddress, officialPublicResolverAddress))
*
* The Public Resolver is attested by a real identity comparison. The
* Permissioned Resolver is attested by a value the subject of the attestation
* controls.
*
* THE AUTHORIZATION FLIP — `features/records/hooks/useCanEditRecords.ts:106-117`:
*
* const hasResolverRole = roleQueries.some((query) => query.data === true)
* ...
* if (isPermissioned) return hasResolverRole
* // Public / non-permissioned resolvers: token owner can edit.
* return isOwner
*
* With the bit forged, `canEdit` stops meaning "you own this name" and starts
* meaning "this contract said yes" — and the contract is the attacker's.
*
* This test drives the REAL exported `decodeImplementationAddress` and
* reproduces the hook's own membership check verbatim, then asserts against the
* repository's own source that no code check exists anywhere on the path. \*/ import { readFileSync } from 'node:fs' import { join } from 'node:path' import { describe, expect, it } from 'vitest' import { type Address, getAddress, type Hex } from 'viem' import { decodeImplementationAddress } from './utils/permissionedResolver'

// happy-dom swallows console.log under vitest; write to stdout directly. const log = (m: string) => process.stdout.write(m + String.fromCharCode(10))

/\*\* Repo root, from apps/portal/src/features/resolver. \*/ const REPO = join(\_\_dirname, '..', '..', '..', '..', '..') const read = (rel: string) => readFileSync(join(REPO, rel), 'utf8')

/\*\* A stand-in for the chain's real `ensPermissionedResolverImpl`. */ const OFFICIAL\_IMPL = getAddress('0x1c7d4b196cb0c7b01d743fbc6116a902379c7238') /*\* The attacker's contract — arbitrary code, not a proxy, not audited. \*/ const EVIL\_RESOLVER = getAddress('0x5fa3a46a6b7f4e374e9671b9dd946d80958c9e47')

/\*\* `useIsPermissionedResolver.ts:16-23`, reproduced verbatim. \*/ const knownPermissionedResolverImplementations = \[OFFICIAL\_IMPL] .filter(Boolean) .map((a) => a.toLowerCase())

/\*\*

* `useIsPermissionedResolver.ts:44-66` with the RPC replaced by its return
* value — i.e. exactly the verdict the hook computes, given what the chain
* reports for that slot. `storageWord` is what the attacker's contract stores. \*/ const verdict = (storageWord: Hex | null): boolean => { const implementationAddress = decodeImplementationAddress(storageWord) if (!implementationAddress) return false return knownPermissionedResolverImplementations.includes( implementationAddress.toLowerCase(), ) }

/\*\* An EIP-1967 slot word holding `addr` in its low 20 bytes. \*/ const slotWordFor = (addr: Address): Hex => `0x${addr.slice(2).toLowerCase().padStart(64, '0')}` as Hex

describe('the "audited and considered secure" verdict rests on forgeable storage', () => { it('a contract that merely STORES the official address is declared official', () => { // The attacker's contract is not a proxy and does not delegatecall // anything. Its constructor wrote one word. const forged = slotWordFor(OFFICIAL\_IMPL)

```
expect(verdict(forged)).toBe(true)

log('  [forged]   arbitrary contract storing OFFICIAL_IMPL -> verdict TRUE')
```

})

it('the verdict is a pure function of that word — nothing about the code', () => { // Same attacker contract, same bytecode, only the stored word differs. expect(verdict(slotWordFor(OFFICIAL\_IMPL))).toBe(true) expect(verdict(slotWordFor(EVIL\_RESOLVER))).toBe(false) expect(verdict(null)).toBe(false) expect(verdict('0x' as Hex)).toBe(false)

```
// The address under audit never enters the computation at all.
log('  [pure]     verdict flips on the stored word alone; bytecode is never read')
```

})

it('no bytecode check exists anywhere on the verdict path', () => { const hook = read( 'apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts', ) const util = read( 'apps/portal/src/features/resolver/utils/permissionedResolver.ts', )

```
// The only chain read is the storage slot.
expect(hook).toContain('getStorageAt')
for (const codeCheck of [
  'getBytecode',
  'getCode',
  'extcodehash',
  'EXTCODEHASH',
]) {
  expect(hook).not.toContain(codeCheck)
}

// And the decoder simply slices the low 20 bytes of whatever it was given.
expect(util).toContain("const normalized = storageValue.slice(2).padStart(64, '0')")
expect(util).toContain("const rawAddress = `0x${normalized.slice(24)}` as Address")

log('  [nocheck]  getStorageAt only — no getCode/getBytecode/EXTCODEHASH')
```

})

it('the sibling attestation in the same component DOES verify identity', () => { const cmp = read( 'apps/portal/src/features/resolver/components/ResolverTypeValue.tsx', )

```
// The claim being made.
expect(cmp).toContain('It is audited and is considered secure.')

// Permissioned: gated on the forgeable verdict.
expect(cmp).toContain('if (isPermissionedResolver)')
// Public: gated on a real address identity comparison.
expect(cmp).toContain(
  'isAddressEqual(resolverAddress, officialPublicResolverAddress)',
)

log('  [asymmetry] Public branch uses isAddressEqual; Permissioned branch uses storage')
```

})

it('the same forged bit flips record-editing away from ownership', () => { const gate = read( 'apps/portal/src/features/records/hooks/useCanEditRecords.ts', )

```
// With the bit set, authorization is delegated to the contract itself...
expect(gate).toContain('if (isPermissioned) return hasResolverRole')
// ...instead of the token owner.
expect(gate).toContain('return isOwner')
// And `hasResolverRole` is read FROM the resolver address under test.
expect(gate).toContain('hasResolverRole')

log('  [authflip] isPermissioned=true => canEdit is whatever the contract answers')
```

}) })
