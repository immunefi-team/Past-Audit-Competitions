# 93045 w a critical explorer resolves a recipient ens name to the ens v1 registry controller so a retained controller name silently redirects name transfers subname ownership and role grants t&#x20;

**Submitted on Sep 14th 2026 at 03:44:16 UTC by @daivo for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93045
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

Every "name or address" input in the Explorer — transfer recipient, subname owner, and the three add-user/role sheets — resolves through one helper, `resolveAddressOrName`. When the typed name has no `addr` record, that helper does **not** fail. It silently substitutes the name's ENS **owner**, and the value it reads for "owner" is `getOwner()` from `@ensdomains/ensjs/public/v1`, taking the `.owner` field while ignoring the sibling `ownershipLevel` and `registrant` fields.

For an unwrapped `.eth` second-level name, ensjs's `.owner` is the ENS **v1 registry controller** (the "manager" slot), *not* the registrant who actually owns the NFT. ensjs itself labels this case `ownershipLevel: 'registrar'` precisely to tell callers that the authoritative owner is `registrant`, not `owner`. The Explorer ignores that signal.

Because the v1 controller slot is (a) settable to any address by the registrant, (b) **not reset when the registrant is transferred**, and (c) the very role that can change a name's resolver and records, an attacker who has ever held a v1 name can retain the controller slot, clear the name's `addr` record to force the fallback, and thereafter have the Explorer resolve that name **to themselves** for every ownership- and permission-granting flow in the app.

## Vulnerability Details

### The fallback

`apps/portal/src/features/roles/helpers/addUser.handlers.ts`:

```ts
export async function resolveAddressOrName({ client, nameOrAddress }) {
  if (isAddress(nameOrAddress, { strict: false })) return nameOrAddress as Address

  try {
    const resolved = await getEnsAddress(client, { name: nameOrAddress, universalResolverAddress })
    let resolvedAddress = resolved

    // Fallback for names that do not set an address record:
    // use current ENS owner address so the role can still be granted.
    if (!resolvedAddress) {
      const ownerResult = await getEnsOwner({ name: nameOrAddress })
      if (ownerResult.isOk()) resolvedAddress = ownerResult.value?.owner ?? null
    }

    return resolvedAddress
  } catch { return null }
}
```

### What `.owner` actually is

`apps/portal/src/utils/ens/resolveEnsOwner.ts:94-102` — the V1 branch:

```ts
const v1Owner = await getOwnerV1(client, { name })
if (v1Owner?.owner && v1Owner.owner !== zeroAddress) {
  return { owner: v1Owner.owner, registryAddress: v1EthRegistry, protocolVersion: 'ENSv1' }
}
```

`ownershipLevel` is never read; `registrant` is never read. But ensjs's own return value for an unwrapped `.eth` 2LD is explicit (`node_modules/@ensdomains/ensjs/src/actions/public/v1/getOwner.ts:218-224`):

```ts
// the registrant is the owner of the NFT
return {
  registrant: registrarOwner,   // ← the true owner
  owner: registryOwner,         // ← the registry CONTROLLER / manager slot
  ownershipLevel: 'registrar',
}
```

So for the single most common class of v1 name, the Explorer's "owner" is the controller. In ENS v1 the registrant and controller are independent slots: the registrant holds the BaseRegistrar ERC-721 and can set the controller to any address, and **`transferFrom` of the registrant does not reset the controller** — a long-documented ENS footgun. `ENSRegistry.setResolver(node, …)` is gated on `owner(node)`, i.e. the *controller*, so a retained controller keeps the power to change the name's resolver and records after ownership has moved on.

That gives the attacker control of both halves of the condition: they can force the `addr` record to be empty (making the fallback fire) and they are the value the fallback returns.

### Every consumer assigns ownership or authority

`resolveAddressOrName` → `getResolvedAddress` → `useAddressResolution`, consumed by six state-changing surfaces:

| Call site                                                     | What the resolved address becomes                                 |
| ------------------------------------------------------------- | ----------------------------------------------------------------- |
| `features/transfer/components/SendNameForm.tsx:81`            | `recipient` of `safeTransferFrom` — **the new owner of the name** |
| `routes/$name/create-subname.tsx:109`                         | `owner` of the newly created subname                              |
| `features/roles/components/RolesAddUserSheet.tsx:73`          | `account` granted registry roles                                  |
| `features/registry/components/v2/RegistryAddUserSheet.tsx:66` | `account` granted registry roles                                  |
| `features/resolver/components/ResolverAddUserSheet.tsx:141`   | `account` granted resolver roles                                  |

`SendNameForm.runTransfer` passes `recipient` straight into `transferToken`'s `safeTransferFrom(owner, recipient, tokenId, 1n, '0x')`. `CreateSubnameForm.handleStartTransaction` passes `ownerAddress` straight into `createSubname({ owner: ownerAddress, … })`. `RolesAddUserSheet.handleSubmit` passes `address` straight into `grantRoles({ account, roles, … })`.

None of them re-checks the provenance of the address.

### Why the UI does not save the user

`RecipientPreview` renders the resolved address and its primary name. This does not help:

1. The attacker's address is shown as *the resolution of the name the user typed*. The user typed `bait.eth` and the app is telling them, with a rendered avatar and address card, that `bait.eth` is that address. That is the deception, not a warning against it.
2. The attacker can set a primary name on their own address, so the preview can be made to render a plausible identity.
3. The impact text anticipates exactly this shape: *"resolving or rendering an ENS name to an attacker-controlled address"* is Critical **as** a rendering defect.

### Divergence from every other ENS client

Canonical ENSIP-1 resolution of a name with no `addr` record yields **no address**, and every conformant wallet refuses to send. This app is the only place where such a name silently acquires an address — and the address it acquires is a legacy permission slot that nobody treats as an identity.

## Impact Details

**A — redirected name transfer (permanent loss).** The victim transfers a v2 name to `bait.eth`. `safeTransferFrom` moves the ERC-1155 to the attacker. ENS name transfers are irreversible; the app's own banner says so.

**B — subname ownership assigned to the attacker.** The victim creates `sub.theirname.eth` with owner `bait.eth`; the attacker owns the subname and its resolution.

**C — role/permission grant to the attacker.** The victim grants registry or resolver roles to `bait.eth`. Depending on the roles selected this hands the attacker `ROLE_SET_ADDR`, `ROLE_SET_SUBREGISTRY`, `ROLE_RENEW`, or their `_ADMIN` variants on a name the victim still believes they solely control — i.e. the attacker can then repoint the name's address records.

**D — no interaction is needed from the attacker at exploit time.** The setup is done once, in advance, on a name the attacker previously held. Every later victim who types that name is redirected.

**Loss per event:** the full value of the transferred name (A), the subname and its namespace (B), or standing write authority over the victim's name (C).

## Relationship to known issues

Not covered by any published entry:

* **EXP-INPUT-005** is about *name validators accepting homoglyphs, bidi and control characters*, scoped to display confusion. This report is about a correctly-normalized, perfectly ordinary ASCII name resolving to the wrong address through a deliberate fallback. Different root cause, different code.
* **QA-01 / WEB-1257** concern role combinations breaking *follow-up actions* after a transfer, and the doc says a demonstrated security consequence of a role combination *"is a new finding."*
* The **Round 3** audit covered only `packages/transaction-manager` and `packages/smart-account`; this is `apps/portal`.
* **SEC-TXM-002** is about a cached *smart-account* address not matching the SDK-derived one — a different value, a different package, and explicitly "no concrete attack path"; this one has a concrete path.

## References

* `apps/portal/src/features/roles/helpers/addUser.handlers.ts:20-35` (the fallback)
* `apps/portal/src/utils/ens/resolveEnsOwner.ts:94-102` (takes `.owner`, drops `ownershipLevel`)
* `@ensdomains/ensjs/src/actions/public/v1/getOwner.ts:218-224` (`owner` = registry controller)
* `apps/portal/src/features/address/queries/getResolvedAddress.ts`
* `apps/portal/src/features/address/hooks/useAddressResolution.ts`
* `apps/portal/src/features/transfer/components/SendNameForm.tsx:81,108-111`
* `apps/portal/src/features/transfer/helpers/transferToken.ts:70-78`
* `apps/portal/src/routes/$name/create-subname.tsx:109,159-168`
* `apps/portal/src/features/roles/components/RolesAddUserSheet.tsx:73,101-107`
* ENSIP-1 (ENS): a name with no `addr` record resolves to no address

***

## Proof of Concept

## Proof of Concept

> Every part below is runnable against the audited revision (`63772fd872af472ced58b009499355f3430c2a86`) with no changes to app code. Parts 1–2 are read-only and need no keys. Part 3 needs one Sepolia name the researcher already controls. Part 4 is a static trace requiring nothing.

**Part 1 — the two notions of "owner" diverge, read-only on live Sepolia.**

ENS v1 stores a name's registrant (the BaseRegistrar ERC-721 holder, i.e. the real owner) and its controller (the ENS registry `owner(node)`, a manager slot) in two independent places. ensjs returns both and labels the case; the Explorer reads only `.owner`.

1. Pick any unwrapped `.eth` 2LD whose controller differs from its registrant. Part 3 creates one deterministically if none is to hand.
2. Read the name three ways — the ENSIP-1 address record, and both ensjs owner fields:

```ts
// pnpm --filter portal exec vite-node poc1.ts
import { createPublicClient, http } from 'viem'
import { sepolia } from 'viem/chains'
import { addEnsContracts } from '@ensdomains/ensjs'
import { getOwner as getOwnerV1 } from '@ensdomains/ensjs/public/v1'
import { getEnsAddress } from 'viem/actions'

const client = createPublicClient({
  chain: addEnsContracts(sepolia),
  transport: http(process.env.SEPOLIA_RPC_URL!),
})

for (const name of ['bait.eth' /* …add more */]) {
  const addr  = await getEnsAddress(client, { name })
  const owner = await getOwnerV1(client, { name })
  console.log({
    name,
    addrRecord:     addr,                    // ENSIP-1 answer
    ownershipLevel: owner?.ownershipLevel,   // 'registrar' for an unwrapped 2LD
    registrant:     owner?.registrant,       // the REAL owner
    dotOwner:       owner?.owner,            // the CONTROLLER — what the app uses
  })
}
```

3. Expected shape for an affected name — record the block height when you run it:

```
name            addrRecord  ownershipLevel  registrant   .owner (CONTROLLER)
bait.eth        null        registrar       0xVICTIM…    0xATTACKER…     <== DIVERGENT
```

`addrRecord: null` is what makes the fallback fire; `ownershipLevel: 'registrar'` is ensjs telling the caller that `registrant` — not `owner` — is authoritative. The Explorer ignores that field entirely.

**Part 2 — the app's own resolver returns the controller.**

`resolveAddressOrName` is the single helper behind the transfer-recipient field, the subname owner field and all three add-user sheets. Call it unmodified:

```ts
// pnpm --filter portal exec vite-node poc2.ts
import { resolveAddressOrName } from '@/features/roles/helpers/addUser.handlers'
// `client` as in Part 1

const resolved = await resolveAddressOrName({ client, nameOrAddress: 'bait.eth' })
console.log(resolved)          // → 0xATTACKER…  (the controller, not the registrant)
```

Compare against what every other ENS client answers for the same input:

```ts
console.log(await getEnsAddress(client, { name: 'bait.eth' }))  // → null
```

A conformant client says "this name has no address." The Explorer says "this name is 0xATTACKER…".

**Part 3 — end-to-end: the name transfer lands on the attacker.**

*Attacker setup, once, on a name they control:*

1. Register an unwrapped v1 `.eth` name, `bait.eth`.
2. Ensure the attacker holds the registry controller slot (already true for a name they registered): `ENSRegistry.setOwner(namehash('bait.eth'), ATTACKER)`
3. Clear the resolver so `addr` resolution yields nothing — permitted because `setResolver` is gated on the **controller**: `ENSRegistry.setResolver(namehash('bait.eth'), 0x0000…0000)`
4. Transfer only the registrant to the victim — a sale, airdrop, prize, or any handover. **`BaseRegistrar.transferFrom` does not reset the controller:** `BaseRegistrar.transferFrom(ATTACKER, VICTIM, labelhash('bait'))`

   Confirm the split state:

```bash
cast call $BASE_REGISTRAR "ownerOf(uint256)(address)" $(cast keccak "bait") \
  --rpc-url $SEPOLIA_RPC_URL          # → VICTIM   (registrant / real owner)
cast call $ENS_REGISTRY "owner(bytes32)(address)" $(cast namehash "bait.eth") \
  --rpc-url $SEPOLIA_RPC_URL          # → ATTACKER (controller — what the app reads)
```

*Victim, in the Explorer:*

5. Connect the victim wallet, open a v2 name they own, choose **Send name**.
6. Type `bait.eth` into Recipient. The field resolves and `RecipientPreview` renders a card showing the attacker's address — presented as the resolution of the name the victim typed.
7. Click **Transfer name** and sign.

*Result:* the submitted call is `safeTransferFrom(VICTIM, ATTACKER, tokenId, 1, 0x)` on the leaf registry. Verify the new holder:

```bash
cast call $LEAF_REGISTRY "balanceOf(address,uint256)(uint256)" $ATTACKER $TOKEN_ID \
  --rpc-url $SEPOLIA_RPC_URL          # → 1
```

ENS transfers are irreversible; the app's own banner says so.

The same three clicks reproduce the other two variants — `/{name}/create-subname` with `bait.eth` as owner assigns the subname to the attacker, and Roles → Add user with `bait.eth` grants the attacker registry or resolver roles.

**Part 4 — the defect is a five-line static trace.**

No execution needed; each step is grep-verifiable in the audited revision.

```
apps/portal/src/features/transfer/components/SendNameForm.tsx:81
    const resolution = useAddressResolution(recipientInput)
        ↓
apps/portal/src/features/address/hooks/useAddressResolution.ts:61
    getResolvedAddressQueryOptions({ nameOrAddress: debounced })
        ↓
apps/portal/src/features/address/queries/getResolvedAddress.ts:26
    resolveAddressOrName({ client, nameOrAddress })
        ↓
apps/portal/src/features/roles/helpers/addUser.handlers.ts:29-34
    if (!resolvedAddress) {                       ← no addr record
      const ownerResult = await getEnsOwner({ name: nameOrAddress })
      resolvedAddress = ownerResult.value?.owner ?? null
    }
        ↓
apps/portal/src/utils/ens/resolveEnsOwner.ts:94-102
    const v1Owner = await getOwnerV1(client, { name })
    return { owner: v1Owner.owner, … }            ← `.owner` = CONTROLLER
                                                  ← `ownershipLevel` never read
                                                  ← `registrant` never read
```

And the library's own definition of the field being used (`@ensdomains/ensjs/src/actions/public/v1/getOwner.ts:218-224`):

```ts
// the registrant is the owner of the NFT
return {
  registrant: registrarOwner,   // ← the true owner
  owner: registryOwner,         // ← the registry CONTROLLER / manager slot
  ownershipLevel: 'registrar',
}
```

## Suggested remediation

1. **Delete the fallback.** A name with no `addr` record has no address; that is what ENSIP-1 says and what every other client does. Return `null` and let the UI say "this name has no address record."
2. If a fallback is genuinely wanted for the role-granting flows, it must use the **authoritative** owner, honouring `ownershipLevel`:

```ts
const o = ownerResult.value
const authoritative =
  o?.ownershipLevel === 'registrar' ? (o.registrant ?? null) : (o?.owner ?? null)
```

3. Regardless of (1) or (2), the UI must distinguish *"resolved via the address record"* from *"substituted from ownership"*, and require explicit confirmation for the latter, rather than rendering both as the same resolution card.
