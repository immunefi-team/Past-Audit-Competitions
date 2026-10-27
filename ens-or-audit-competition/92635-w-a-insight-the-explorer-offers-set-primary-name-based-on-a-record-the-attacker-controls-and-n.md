# 92635 w a insight the explorer offers set primary name based on a record the attacker controls and never checks name ownership so one click on a link replaces the victim s primary name with a&#x20;

**Submitted on Sep 11th 2026 at 05:31:49 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92635
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Summary

The Explorer's `/$name/address` page offers a **"Set primary name"** button whenever the name's own `addr(coinType)` record happens to equal the connected wallet. It never checks that the connected wallet owns the name.

That record belongs to the name, not to the viewer — so an attacker who owns any name can point its address record at a chosen victim and thereby cause the button to appear, on their own name's page, for that victim. One click writes the attacker's name into the victim's ENSIP-19 reverse record, destroying whatever primary name the victim had.

No homoglyph, no Unicode and no unnormalized name is involved. The attacker's name is an ordinary, fully-normalized `.eth` name.

## The gate

`apps/portal/src/features/forward-resolution/components/AddressResolution/AddressResolutionSidebar.tsx:453-458`

```ts
  // Offered only when the connected wallet *is* this address (setName is
  // msg.sender-scoped).
  const canSetPrimaryName =
    !!connectedAddress &&
    !!data?.address &&
    isAddress(data.address, { strict: false }) &&
    isAddressEqual(connectedAddress, data.address)
```

The comment states the reasoning exactly: they verified that you can only write **your own** reverse record, because `setName` is `msg.sender`-scoped. That is true and irrelevant. The question they did not ask is whether you should be allowed to point your own reverse record at **a name you do not own**, on the strength of a record that name's owner controls.

## `data.address` is chosen by the name's owner, not the viewer

`apps/portal/src/routes/$name/address.tsx:104-108`

```ts
  const addressByCoinType = new Map(
    (profileQuery.data?.records?.coins ?? []).map(
      (c) => [c.coinType, c.value] as const,
    ),
  )
```

Those are the `addr(coinType)` records read from the name's resolver. Whoever owns the name writes them. So the value the gate compares `connectedAddress` against is entirely under the attacker's control, and setting it is an ordinary record write on their own name.

## The button appears precisely because the pair is broken

`AddressResolutionSidebar.tsx:351`

```ts
  const offerSetPrimary = status === 'mismatch' && canSetPrimaryName
```

`status` comes from the ENSIP-19 reverse check. In this scenario it is guaranteed to be `mismatch`, because the victim's reverse record does not name the attacker's name yet — which is the whole point of the attack. The page therefore renders a red warning that the address "does not resolve back to this name", and puts the remediation button directly under it. The UI presents adopting the attacker's name as *fixing a problem*.

## The calldata

`AddressResolutionSidebar.tsx:531-553` — both L1 branches:

```ts
    if (data?.coinType === MAINNET_COIN_TYPE) {
      const registrarAddress = getRegistrarAddress(60, 'sepolia')
      if (!registrarAddress) return
      setReverseResolution({
        name,
        request: {
          address: registrarAddress,
          abi: reverseRegistrarSetNameSnippet,
          functionName: 'setName',
          args: [name],
        },
      })
      return
    }
    setReverseResolution({
      name,
      request: {
        address: DEFAULT_REVERSE_REGISTRAR_ADDRESS,
        abi: defaultReverseRegistrarSetNameSnippet,
        functionName: 'setName',
        args: [name],
      },
    })
```

`name` is the raw `/$name/address` route param — the attacker's name, straight from the link the victim followed.

## The check exists, is already loaded, and is thrown away

`AddressResolutionSidebar.tsx:431`

```ts
  const { data: owner } = useQuery(getEnsOwnerQueryOptions({ name }))
```

`owner` is referenced exactly once in the entire file — `:619`:

```ts
    protocolVersion: owner?.protocolVersion,
```

The component resolves the name's owner and uses only its protocol version. The owning address is never compared to `connectedAddress`. The data needed to block this is sitting in the component, one `isAddressEqual` away.

## Attack sequence

1. Attacker registers an ordinary name, e.g. `freebie.eth`, and sets its `addr(60)` record to the **victim's** address. This is a normal write on a name they own.
2. Attacker sends the victim a link to `/freebie.eth/address` — a plain Explorer URL on ENS's own domain.
3. The victim opens it with their wallet connected. `canSetPrimaryName` passes (the record names them), `status` is `mismatch` (their reverse record does not name `freebie.eth`), so the red banner and the **Set primary name** button render.
4. The victim clicks once. `setName("freebie.eth")` executes against their own reverse registrar.
5. The victim's primary name is now `freebie.eth`, a name the attacker owns outright.

## Impact

**Unconditional, on the click:** the victim's existing primary name is overwritten and their on-chain identity becomes a name the attacker owns and fully controls — its resolver, its records, its avatar — and which the attacker can transfer or let expire at will. Every ENS-aware wallet, explorer and dapp that reads the victim's primary name renders the attacker's chosen string, and it forward-verifies while the attacker keeps the address record pointed at the victim.

**Escalation, requiring one further attacker step:** the attacker later repoints `freebie.eth`'s `addr(60)` at their own address. Anyone who has seen the victim identified as `freebie.eth` and resolves that name — to pay them, to whitelist them, to address them — now resolves it to the attacker. That is the row's own example, "resolving or rendering an ENS name to an attacker-controlled address", reached from a name the victim was induced to adopt as their identity.

I want to be precise about the boundary: step 5 is unconditional once the victim clicks, and needs nothing beyond the link. The funds-misdirection consequence needs the attacker to repoint the record afterwards, which costs them one transaction and nothing else.

**Preconditions:** the victim has a connected wallet and follows a link. There is no requirement that they own any particular name, hold any role, or have any prior relationship with the attacker.

## Severity

**Critical.** The transaction the victim approves is presented as repairing their own configuration and instead adopts an attacker-controlled identity; the downstream effect is the row's stated example of rendering a name to an attacker-controlled address so funds reach an unintended recipient. It is also a malicious interaction with an already-connected wallet: the parameter written on chain is a name chosen by the attacker and supplied through a URL.

## Why this is not an already-known or already-reported issue

**Not a normalization issue.** This report does not rely on a homoglyph, a soft hyphen, mixed case, or any unnormalized input. The attacker's name is an ordinary normalized `.eth` name, and the attack works identically whatever it is called. The published known issue about validators accepting homoglyphs and control characters is not engaged.

**Not in the published known issues.** There is no entry covering the primary-name write path being unauthorized. The nearest entries are the opposite failure — a user being *unable* to set a primary name after a transfer or a role change (QA-01, WEB-1257) — and a display-metadata note on the address view (WEB-1301). Part 2's own preamble draws the line this report sits on: a demonstrated security consequence "rather than merely erroring" is a new finding.

**Distinct from the raw-vs-normalized primary-name reports.** Those concern the Manager's set-primary flow writing its forward legs against one form of a name and its reverse leg against another. This is the Explorer, a different app and a different code path, and the defect is an **authorization gap** — a missing ownership check — not a mismatch between two spellings of the same name. Removing the normalization question entirely leaves this bug fully intact.

## Recommended fix

Gate the button on ownership, using the query the component already runs:

```ts
const connectedOwnsName =
  !!connectedAddress && !!owner?.owner && isAddressEqual(connectedAddress, owner.owner)

const canSetPrimaryName =
  connectedOwnsName &&
  !!data?.address &&
  isAddress(data.address, { strict: false }) &&
  isAddressEqual(connectedAddress, data.address)
```

The existing address comparison should stay — it is correct for what it was written for — but it is a check that the *name* points at you, and it must not be mistaken for a check that *you* control the name. The sibling reverse-resolution table already restricts its sidebar to the connected account (`ReverseResolutionTable.tsx:64`, `isAddressEqual(account, address)`); this surface has no equivalent restriction.

Failing that, the confirmation copy must state plainly that the victim is adopting a name owned by someone else, and name that owner.

## Proof of Concept

## What this proves

The test executes the gate expression from `AddressResolutionSidebar.tsx:453-458` verbatim, against viem's real `isAddress` / `isAddressEqual`, and encodes the `setName` calldata with the same ABI snippet the component imports (`@ensdomains/ensjs-abi/reverseRegistrar`). Five assertions:

1. **the gate passes on a name the wallet does not own** — with the attacker's `addr(60)` pointed at the victim, `canSetPrimaryName(victim, row)` is `true` on the attacker's own name's page;
2. **the button is offered** — `offerSetPrimary('mismatch', victim, row)` is `true`, and the three non-mismatch states are `false`, confirming the button appears precisely in the state the attack creates;
3. **the calldata** decodes back to `setName("freebie.eth")` — the attacker's name — and the victim's real primary name appears nowhere in the transaction that overwrites it;
4. **the missing check** — `owner !== connectedAddress` would have blocked it, while the gate that does exist passes;
5. **control** — the gate correctly refuses on a non-matching, absent, or malformed record, so it is not trivially always-true.

## Environment

Frozen revision under test: 1c9b47f. Nothing is deployed and no live service is contacted.

```
pnpm install
```

Save the file below as:

```
apps/portal/src/features/forward-resolution/poc-set-primary-no-ownership-check.test.ts
```

Run it:

```
cd apps/portal
pnpm vitest run src/features/forward-resolution/poc-set-primary-no-ownership-check.test.ts
```

## Result

```
RUN  v4.1.5 .../apps/portal

  [gate]    canSetPrimaryName(victim, addr=victim) === true on the ATTACKER’s name
  [offer]   status=mismatch + gate -> red warning + "Set primary name" button
  [calldata] setName("freebie.eth") — overwrites the victim’s "alice.eth"
  [missing] owner=attacker !== connected=victim — one comparison away
  [control] gate refuses on a non-matching / absent / malformed record

 Test Files  1 passed (1)
      Tests  5 passed (5)
```

## Reproducing in the UI

1. From any wallet, register an ordinary `.eth` name (`freebie.eth`) and set its ETH address record to the victim’s address. Both are normal operations on a name you own.
2. Send the victim `https://<explorer>/freebie.eth/address`.
3. With their wallet connected, the victim sees the red banner “The set address does not resolve back to this name on Mainnet” and a **Set primary name** button beneath it.
4. One click. The wallet prompts for `setName` on the reverse registrar; the modal describes it as `Set freebie.eth as the primary name`.
5. After confirmation the victim’s primary name is `freebie.eth`. Their previous primary name is gone.

## Verifying the call sites

```
grep -n "canSetPrimaryName" -A 6 apps/portal/src/features/forward-resolution/components/AddressResolution/AddressResolutionSidebar.tsx
grep -n "offerSetPrimary" apps/portal/src/features/forward-resolution/components/AddressResolution/AddressResolutionSidebar.tsx
grep -n "args: [name]" apps/portal/src/features/forward-resolution/components/AddressResolution/AddressResolutionSidebar.tsx
grep -n "getEnsOwnerQueryOptions" apps/portal/src/features/forward-resolution/components/AddressResolution/AddressResolutionSidebar.tsx
grep -n "owner" apps/portal/src/features/forward-resolution/components/AddressResolution/AddressResolutionSidebar.tsx | grep -v connectedAddress
grep -n "addressByCoinType" -A 4 apps/portal/src/routes/$name/address.tsx
```

The second-to-last command returns exactly two lines — the fetch at `:431` and `protocolVersion: owner?.protocolVersion` at `:619` — which is the whole of the ownership data’s use.

## Full test file

/\*\*

* PoC — the Explorer's "Set primary name" button is gated on a record the
* ATTACKER controls, never on owning the name. One click on an attacker's link
* replaces the victim's ENSIP-19 primary name with a name the attacker owns.
*
* THE GATE — `features/forward-resolution/components/AddressResolution/AddressResolutionSidebar.tsx:453-458`:
*
* // Offered only when the connected wallet *is* this address (setName is
* // msg.sender-scoped).
* const canSetPrimaryName =
* ```
  !!connectedAddress &&
  ```
* ```
  !!data?.address &&
  ```
* ```
  isAddress(data.address, { strict: false }) &&
  ```
* ```
  isAddressEqual(connectedAddress, data.address)
  ```
*
* The comment shows the reasoning: they confirmed you can only write YOUR OWN
* reverse record. They never asked whether you should be allowed to point it at
* a name you do not own.
*
* `data.address` is NOT the viewer's property. It is the name's own
* `addr(coinType)` record, read from that name's resolver —
* `routes/$name/address.tsx:104-108`:
*
* const addressByCoinType = new Map(
* ```
  (profileQuery.data?.records?.coins ?? []).map(
  ```
* ```
    (c) => [c.coinType, c.value] as const,
  ```
* ```
  ),
  ```
* )
*
* — so whoever owns the name chooses the value the gate compares against.
*
* THE BUTTON — `AddressResolutionSidebar.tsx:351`:
*
* const offerSetPrimary = status === 'mismatch' && canSetPrimaryName
*
* `status === 'mismatch'` is guaranteed in the attack: the victim's reverse
* record does not yet name the attacker's name, which is exactly what the red
* "does not resolve back to this name" banner reports. The remediation button
* is rendered directly beneath it.
*
* THE CALLDATA — `AddressResolutionSidebar.tsx:531-553`, both L1 branches:
*
* args: \[name],
*
* `name` is the raw `/$name/address` route param — the attacker's name.
*
* THE DISCARDED CHECK. The component already fetches the answer and throws it
* away. `AddressResolutionSidebar.tsx:431`:
*
* const { data: owner } = useQuery(getEnsOwnerQueryOptions({ name }))
*
* and `owner` is referenced exactly once in the whole file, at `:619`:
*
* protocolVersion: owner?.protocolVersion,
*
* The owning address is never compared to `connectedAddress`.
*
* NOTE ON SCOPE: this needs no homoglyph, no Unicode and no unnormalized name.
* The attacker's name is an ordinary, fully-normalized `.eth` name. The
* published known issue about validators accepting homoglyphs is not involved.
*
* This test executes the REAL gate expression (viem `isAddress`/`isAddressEqual`)
* and encodes the REAL `setName` calldata with the same ABI snippet the
* component imports. \*/ import { reverseRegistrarSetNameSnippet } from '@ensdomains/ensjs-abi/reverseRegistrar' import { describe, expect, it } from 'vitest' import { type Address, decodeFunctionData, encodeFunctionData, getAddress, isAddress, isAddressEqual, } from 'viem'

// happy-dom swallows console.log under vitest; write to stdout directly. const log = (m: string) => process.stdout.write(m + String.fromCharCode(10))

const VICTIM = getAddress('0x5fa3a46a6b7f4e374e9671b9dd946d80958c9e47') const ATTACKER = getAddress('0x1c7d4b196cb0c7b01d743fbc6116a902379c7238')

/\*\* An ordinary, fully-normalized name the ATTACKER owns. */ const ATTACKER\_NAME = 'freebie.eth' /*\* The name the victim actually owns and currently has as their primary. \*/ const VICTIM\_REAL\_PRIMARY = 'alice.eth'

/\*\* `AddressResolutionSidebar.tsx:453-458`, reproduced verbatim. \*/ const canSetPrimaryName = ( connectedAddress: Address | undefined, data: { address: string | null } | null, ) => !!connectedAddress && !!data?.address && isAddress(data.address, { strict: false }) && isAddressEqual(connectedAddress, data.address as Address)

/\*\* `AddressResolutionSidebar.tsx:351`, reproduced verbatim. \*/ const offerSetPrimary = ( status: string, connectedAddress: Address | undefined, data: { address: string | null } | null, ) => status === 'mismatch' && canSetPrimaryName(connectedAddress, data)

/\*\* `AddressResolutionSidebar.tsx:534-541` — the Mainnet-row branch. \*/ const buildSetNameCalldata = (name: string) => encodeFunctionData({ abi: reverseRegistrarSetNameSnippet, functionName: 'setName', args: \[name], })

describe('Set primary name is gated on an attacker-controlled record', () => { it('the gate passes on a name the connected wallet does NOT own', () => { // The attacker owns freebie.eth and points its addr(60) at the VICTIM. // This is the only setup step, and it is a normal record write on their // own name. const row = { address: VICTIM as string }

```
// The victim opens /freebie.eth/address with their wallet connected.
expect(canSetPrimaryName(VICTIM, row)).toBe(true)

// The victim owns nothing here — ownership is never consulted.
log('  [gate]    canSetPrimaryName(victim, addr=victim) === true on the ATTACKER’s name')
```

})

it('the button is offered precisely because the pair does not match yet', () => { const row = { address: VICTIM as string }

```
// The victim's reverse record still says alice.eth, so the UniversalResolver
// reverse of their address does not return freebie.eth -> 'mismatch'.
expect(offerSetPrimary('mismatch', VICTIM, row)).toBe(true)

// The states that would NOT offer it, for contrast.
expect(offerSetPrimary('verified', VICTIM, row)).toBe(false)
expect(offerSetPrimary('pending', VICTIM, row)).toBe(false)
expect(offerSetPrimary('unverifiable', VICTIM, row)).toBe(false)

log('  [offer]   status=mismatch + gate -> red warning + "Set primary name" button')
```

})

it('one click signs setName(attackerName) on the victim’s own reverse record', () => { const data = buildSetNameCalldata(ATTACKER\_NAME) const decoded = decodeFunctionData({ abi: reverseRegistrarSetNameSnippet, data, })

```
expect(decoded.functionName).toBe('setName')
expect((decoded.args as readonly unknown[])[0]).toBe(ATTACKER_NAME)

// The victim's real primary name appears nowhere in the transaction that
// overwrites it.
expect(data).not.toContain(
  Buffer.from(VICTIM_REAL_PRIMARY).toString('hex'),
)

log(`  [calldata] setName("${ATTACKER_NAME}") — overwrites the victim’s "${VICTIM_REAL_PRIMARY}"`)
```

})

it('an ownership check would have blocked it — and the data is already loaded', () => { // What the component fetches at :431 but never compares. const ownerOfAttackerName = ATTACKER

```
// The check that does not exist:
const wouldBeBlocked = !isAddressEqual(VICTIM, ownerOfAttackerName)
expect(wouldBeBlocked).toBe(true)

// ...while the check that DOES exist passes.
expect(canSetPrimaryName(VICTIM, { address: VICTIM as string })).toBe(true)

log('  [missing] owner=attacker !== connected=victim — one comparison away')
```

})

it('control: the gate correctly refuses when the record names someone else', () => { // Sanity check that the gate is not simply always-true: if the attacker // points the record anywhere other than the victim, the button is withheld. expect(canSetPrimaryName(VICTIM, { address: ATTACKER as string })).toBe(false) expect(canSetPrimaryName(VICTIM, { address: null })).toBe(false) expect(canSetPrimaryName(undefined, { address: VICTIM as string })).toBe(false) expect(canSetPrimaryName(VICTIM, { address: 'not-an-address' })).toBe(false)

```
log('  [control] gate refuses on a non-matching / absent / malformed record')
```

}) })
