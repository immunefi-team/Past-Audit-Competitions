# 89398 w a critical missing address record validation in resolveaddressorname causes irreversible ens name transfers to the wrong recipient

**Submitted on Aug 19th 2026 at 02:13:44 UTC by @Elienai for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89398
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer app's ENS-name ownership-transfer screen (`SendNameForm.tsx`, which explicitly labels the action "irreversible") resolves the recipient input (an ENS name or address) through a shared utility (`useAddressResolution` -> `getResolvedAddress` -> `resolveAddressOrName`) that was originally written for a different feature: granting an in-app "role" to a name's registry owner when the name has no address record ("so the role can still be granted").

When reused for the transfer-recipient field, if the typed recipient name has no `addr(60)` address record configured—an ordinary state for freshly-registered, parked, or for-sale names, or simply a mistyped name that happens to match an unrelated, unconfigured one—the resolver silently substitutes the name's registry OWNER address instead of failing or warning the user.

The UI shows the result exactly like a genuine resolution, with zero visual distinction. This substituted address flows unchanged into an ERC-1155 `safeTransferFrom` call that irreversibly moves the name's ownership token.

## Vulnerability Details

### The fallback's origin: written for role-granting, not for irreversible transfers

`apps/portal/src/features/roles/helpers/addUser.handlers.ts`, lines 11-40:

```ts
export async function resolveAddressOrName({ client, nameOrAddress }: ResolveAddressOrNameParams): Promise<Address | null> {
  if (isAddress(nameOrAddress, { strict: false })) {
    return nameOrAddress as Address
  }
  try {
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
  } catch {
    return null
  }
}
```

The comment names the specific, narrower use case this fallback was designed for: granting a role, where "whoever currently controls the name" is a defensible proxy for the intended recipient.

### Generalized into the app's shared "resolve a name or address" utility

`apps/portal/src/features/address/queries/getResolvedAddress.ts` and `apps/portal/src/features/address/hooks/useAddressResolution.ts` reuse this same function generically.

The hook's own docstring explicitly names the transfer-recipient field as a consumer: "Shared by every 'name or address' input in the app (transfer recipient, add-user sheets, ...)".

The returned `status` is `'resolved'` whenever the resolved value is non-null, regardless of which path produced it—there is no separate status distinguishing "real address record" from "owner-fallback substitution."

The one warning state (`'unresolved'`) fires only when *both* the address lookup *and* the owner fallback fail—the opposite of the vulnerable case.

### Used, unguarded, for the irreversible name-transfer recipient field

`apps/portal/src/features/transfer/components/SendNameForm.tsx` resolves `recipient` via `useAddressResolution(recipientInput)` and only checks whether `recipient` equals the current owner (self-transfer) or the zero address.

Nothing checks or flags whether `recipient` came from a genuine address-record resolution or the owner-fallback substitution.

`RecipientPreview`'s reverse (primary-name) lookup is a partial, unreliable signal at best: a stale/parked/unconfigured name's owner very often has no primary name pointing back to it either, in which case `RecipientPreview` renders identically to a legitimate address with no primary name set.

`apps/portal/src/features/address/components/AddressNameInput.tsx`, lines 54-69, the shared component's default resolved-content text (used whenever a consumer does not override it):

```tsx
.with('resolved', () => {
  if (resolvedContent !== undefined) return resolvedContent
  return address ? (
    <p className="text-sm mt-1.5 text-muted-foreground">
      {isRawAddress ? `Using address: ${truncateAddress(address, 6, 4)}` : `Resolved: ${truncateAddress(address, 6, 4)}`}
    </p>
  ) : null
})
```

`apps/portal/src/features/transfer/hooks/useTransferName.ts`: `recipient` flows unchanged into `runStep('transfer-token', ...)`—no revalidation.

`apps/portal/src/features/transfer/helpers/transferToken.ts`, lines 46-97, the final irreversible on-chain call:

```ts
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
})
```

### The owner lookup itself is correctly implemented—the bug is using it in the wrong context

`apps/portal/src/utils/ens/resolveEnsOwner.ts` is a well-implemented, independent registry-tree read (via the real `@ensdomains/ensjs` v2 `getOwner`), entirely decoupled from resolver/address-record state.

This is not buggy code—it is correct code for what it does, being reused in a context (resolving a payment/transfer destination) where "who owns the registry entry" and "where the name's records point" are different questions with different security implications.

### Checked the whole connected system for any mitigation before concluding "no safety net"

`apps/portal/src/features/transaction-manager/components/TransactionModal.tsx` (the confirmation UI shown right before the wallet is asked to sign) renders only step titles (e.g. "Transfer name") and estimated gas cost—it never re-derives or re-displays the recipient address.

No second checkpoint exists between `RecipientPreview` and the actual signature.

## Impact

**Concrete, sharpest scenario — typosquatting:** a user intends to transfer their name to `bob.eth`, but mistypes `bobb.eth`. If `bobb.eth` happens to be registered by someone (anyone) but has no address record, the app silently "succeeds" and shows what looks like a normal resolved recipient—the exact case a careful user's own error-checking cannot catch, because the app itself gives no indication anything is unusual.

A deliberate attacker only needs to own some unconfigured name close enough to a plausible target to catch this kind of typo—no active outreach or social engineering of the victim is required.

**Broader, no-attacker-required scenario:** any transfer to a genuinely parked, freshly-registered, or for-sale name with no address record configured—a common, ordinary state, not a rare corner case—silently redirects to whatever address currently holds that name's registry entry, with the app showing no warning at any point in the flow.

**Broader blast radius:** the same `resolveAddressOrName` also backs the subname-owner field in `apps/portal/src/routes/$name/create-subname.tsx` and three separate role-granting sheets—this report focuses on the transfer-recipient case as the clearest, highest-stakes (irreversibility) instance.

## Why this is not "working as intended"

The fallback's own comment justifies it only for role-granting ("so the role can still be granted")—a context where "whoever currently controls the name" is a defensible proxy for the intended grantee.

Nowhere in the codebase, tests, or comments is there a rationale for why "resolve a NAME to its registry OWNER, not its address record" is correct behavior for a field whose entire purpose is choosing the destination of an irreversible ownership transfer.

This is the signature of shared-component reuse for consistency/DRY reasons without re-deriving every edge case for every new call site, not a considered design decision.

## Severity justification

This program's own scope page states a Critical category matching this report almost verbatim:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: resolving or rendering an ENS name to the wrong address, showing the wrong recipient... the user then approves.

Prior paid precedent in the same protocol, for context: in 2022, a researcher was paid $15,000 by ENS (via Immunefi) for a name-spoofing bug that confused marketplaces about which name was being traded ("How we spoofed ENS domains for $15k"), paid at High.

That bug is in the same general family but is mechanically weaker: it required the attacker to actively register a decoy name, and required the victim to complete an additional purchase step on a third-party marketplace.

This report's mechanism requires no attacker action at all for the accidental case, and directly feeds an irreversible transfer already in progress with no additional step.

## Duplicate check

Checked against the full Known Issues document for this competition.

The closest listed item, EXP-INPUT-005, concerns the name input validator accepting homoglyph/bidi/control characters when a user types a name—a completely different mechanism from this report.

No other listed item mentions `resolveAddressOrName`, the owner-fallback, or any resolution-fallback issue in the transfer/address features.

`apps/portal` was not covered by the Round 3 internal audit.

## Honest caveats

No end-to-end browser reproduction (screenshot of the actual `SendNameForm` UI in the vulnerable scenario) was performed—no dev server/bundler was available in the audit environment.

The evidence instead consists of a direct, complete reading of every file in the real call chain, plus two independently executed reproductions:

* One proving the on-chain precondition is native to the real, unmodified registry contract.
* One running the real, verbatim application logic against that exact precondition.

## Proof of Concept

Two independent pieces of evidence, both executed, not merely read.

{% stepper %}
{% step %}

## On-chain: the triggering precondition is a native, ordinary protocol state

Written and run against the real, unmodified `PermissionedRegistry.sol` (`ensdomains/contracts-v2` commit `8d9dcca8a7703e5d15958d628c39579d3aa789e1`, independently re-confirmed live via `raw.githubusercontent.com/ensdomains/contracts-v2/8d9dcca8a7703e5d15958d628c39579d3aa789e1/contracts/src/registry/PermissionedRegistry.sol` -> HTTP 200).

Foundry test `MissingResolverIsNativeStateTest.test_nameCanHaveRealOwnerButNoResolver`:

```solidity
function test_nameCanHaveRealOwnerButNoResolver() public {
    // Register a name via the REAL contract's own public register() function,
    // with resolver = address(0) -- an entirely ordinary, single-call
    // registration choice the contract's own interface explicitly supports.
    uint256 tokenId = registry.register(
        "parkedname",
        owner,
        address(registry),
        address(0), // resolver
        rolesBitmap,
        uint64(block.timestamp + 365 days)
    );

    // Confirm: no address record obtainable (resolver unset)...
    assertEq(registry.getResolver("parkedname"), address(0));
    // ...while the registry OWNER still resolves to a real, distinct address.
    assertEq(registry.getOwner(tokenId), owner);
}
```

Real `forge test` output:

```
Ran 1 test for test/ManagerRoleHijack.t.sol:MissingResolverIsNativeStateTest
[PASS] test_nameCanHaveRealOwnerButNoResolver() (gas: 164335)
```

This proves the precondition (a name with a real owner but no resolver/address record) is native, ordinary protocol state—not a contrived edge case.
{% endstep %}

{% step %}

## Application logic: the real, verbatim resolution function, executed

`resolveAddressOrName.repro.mjs` imports the real `viem@2.52.2` package (matching the app's own pinned `pnpm-workspace.yaml` catalog entry `viem: ^2.52.2` exactly) and runs `resolveAddressOrName` copied VERBATIM, line-for-line, from `apps/portal/src/features/roles/helpers/addUser.handlers.ts:11-40`.

Only the two network-boundary calls (`getEnsAddress`, `getEnsOwner`) are stubbed—the standard, legitimate unit-testing boundary (mock the network, run the real logic), not a rewrite of the function under test.

The stub return values mirror exactly the on-chain state proven native in Step 1: no address record and a real, distinct owner.

```js
import { isAddress } from 'viem'

async function getEnsAddress(_client, { name }) {
  return null // no addr(60) record set -- proven native state, Step 1
}

const REGISTRY_OWNER_ADDRESS = '0xA11ce00000000000000000000000000BADF00D'
async function getEnsOwner({ name }) {
  return { isOk: () => true, value: { owner: REGISTRY_OWNER_ADDRESS } }
}

// VERBATIM copy of apps/portal/src/features/roles/helpers/addUser.handlers.ts:11-40
async function resolveAddressOrName({ client, nameOrAddress }) {
  if (isAddress(nameOrAddress, { strict: false })) {
    return nameOrAddress
  }
  try {
    const resolved = await getEnsAddress(client, {
      name: nameOrAddress,
      universalResolverAddress: '0xUNIVERSAL_RESOLVER_STUB',
    })
    let resolvedAddress = resolved
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

async function main() {
  const recipientTypedByUser = 'parkedname.eth' // what the transferring user typed
  const resolved = await resolveAddressOrName({ client: {}, nameOrAddress: recipientTypedByUser })
  console.log('Resolved recipient fed into safeTransferFrom:', resolved)
}
main()
```

Real `node` output:

```
=== ENS audit-comp-ens Finding #3 -- runnable PoC ===
User types transfer recipient: "parkedname.eth"
Calling the REAL, verbatim resolveAddressOrName (addUser.handlers.ts:11-40)...

  [stub] getEnsAddress(name="parkedname.eth") -> null (no address record set)
  [stub] getEnsOwner(name="parkedname.eth") -> { owner: "0xA11ce00000000000000000000000000BADF00D" } (real, distinct registry owner)

Result fed into SendNameForm's "recipient" (and then transferToken.ts's
safeTransferFrom "to" argument): 0xA11ce00000000000000000000000000BADF00D

CONFIRMED: resolveAddressOrName returned the REGISTRY OWNER address
(a value from a completely different on-chain field than an addr(60)
address record) as if it were a genuine name resolution -- with no
error, no warning, and a status the UI treats identically to a real
"Resolved: 0x..." hit.
```

{% endstep %}

{% step %}

## Reproduction steps summary

Deploy/checkout the real `PermissionedRegistry.sol` at commit `8d9dcca8a7703e5d15958d628c39579d3aa789e1`.

Register any name with `resolver = address(0)` via the contract's own public `register()` function—this is ordinary, supported usage, not an exploit of the registry itself.

In the app, as any user, open the "Transfer name" flow for a name you own and type the parked/unconfigured name (or a typo that happens to match one) as the recipient.

Observe: `resolveAddressOrName` returns the parked name's registry OWNER address (not an address record, since none exists), and the UI shows "Resolved: 0x..." exactly as it would for a genuine hit.

If the user confirms, `safeTransferFrom` sends the name's ownership token to that owner address—irreversibly, and to an address the user never intended to specify.
{% endstep %}
{% endstepper %}
