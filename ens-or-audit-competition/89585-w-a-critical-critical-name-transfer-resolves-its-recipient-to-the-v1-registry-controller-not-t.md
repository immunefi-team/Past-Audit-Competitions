# 89585 w a critical critical name transfer resolves its recipient to the v1 registry controller not the erc 721 holder the explorer also labels that controller as owner

**Submitted on Aug 20th 2026 at 01:12:20 UTC by @jokerbuena for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89585
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief / Intro

When a name has no address record, `resolveAddressOrName` falls back to the **registry owner**. On a v1 name that address is the **controller**, not the ERC-721 registrant who actually holds the name. That value is what the transfer form uses as the **recipient**.

The same confusion is visible in the Explorer: the page titled *Ownership* prints the controller under `Owner` and under `Manager`, and never presents the registrant as the owner at all.

Everything below is at `1c9b47f`, branch `audit-comp-ready` - the branch the asset table pins.

## Vulnerability Details

### 1. The transfer form takes its recipient from the shared resolver

`apps/portal/src/features/transfer/components/SendNameForm.tsx`

```tsx
const resolution = useAddressResolution(recipientInput)        // :81
const { address: recipient, isResolving } = resolution         // :82
...
startTransfer({ recipient, options: effectiveOptions })        // :108
```

Line 117 of the same file renders the warning *"check the recipient address before proceeding"*. The user cannot honour it: the address they are asked to check is the one the app resolved wrongly.

### 2. The resolver's fallback returns the registry owner

`apps/portal/src/features/roles/helpers/addUser.handlers.ts`

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

`owner` here is `registry.owner(node)`. For a v1 name that is the controller. The registrant - `baseRegistrar.ownerOf(labelhash)` - is never consulted, and `ResolvedEnsOwner` carries no `registrant` field at all, so the distinction is erased at the type level rather than merely unread.

### 3. The chain is shared, so this is not confined to transfers

`useAddressResolution`'s own docblock states it is *"shared by every 'name or address' input in the app (transfer recipient, add-user sheets, ...)"*. The transfer path is the one with the Critical impact; the others inherit the same wrong address.

## Impact Details

The impact selected is this program's own Critical line, verbatim:

> *Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: ... **redirecting the recipient of a name transfer** ...*

A sender types the recipient's ENS name. If that name has no address record, the app resolves it to the controller and hands that address to `startTransfer`. The name leaves for an address the sender never chose and the recipient does not hold. On a v1 name the controller is frequently **not** the person who owns the NFT - that separation is the whole point of the registrar/registry split.

## Proof of Concept

### Live chain state, read before anything else

```
$ cast call <registry> 'owner(bytes32)(address)' <namehash of sandytest1759027793098.eth>
    0x742d35CC6634C0532925A3B8d0c9E3e0e8B9C5b1      <- controller
$ cast call <baseRegistrar> 'ownerOf(uint256)(address)' <labelhash>
    0x94D2F676913722d704E09931B9ca306977Dbc4CE      <- registrant, the ERC-721 holder
```

The two differ on Sepolia today. That is the precondition, and it is read from the chain, not assumed.

### The running application, same name

With the portal dev server pointed at Sepolia, `/<name>/ownership` prints:

```
Owner     0x742d...C5b1      <- the controller
Manager   0x742d...C5b1      <- the same address
HISTORY   0x94d2...c4ce      <- the registrant appears only here, never as owner
```

A screenshot of that page is attached. The page titled **Ownership** names the controller as Owner.

### The test suite, run and mutated

```
1) intact
   POC_WEB36_LIVE_sepolia.test.ts                OK    Tests  2 passed (2)
   POC_WEB36_owner_fallback_controller.test.ts   OK    Tests  7 passed (7)

2) mutation:  resolvedAddress = ownerResult.value?.owner ?? null   ->   resolvedAddress = null
   POC_WEB36_LIVE_sepolia.test.ts                ECHEC Tests  1 failed | 1 passed (2)
   POC_WEB36_owner_fallback_controller.test.ts   ECHEC Tests  3 failed | 4 passed (7)

3) restored -> 2/2 and 7/7, git status --porcelain: 0 entries
```

The five assertions that stay green under mutation are negative controls: inputs where the address record exists, so the fallback is never reached. The failure is targeted at the reported branch rather than global.

### What I did not do, stated rather than omitted

I did not capture the transfer form itself. It renders only for an ENSv2 name **and** only once the owning wallet is connected, and I did not connect a wallet. The recipient path is therefore evidenced by the code chain above plus the mutation, not by a screenshot of the input. The screenshot I do attach is the Ownership page, which is the display half of the same root cause.

## Notes on duplication, enumerated rather than sampled

**A. The sponsor's published known-issues document** - the baseline this competition defines. Probes on this finding's mechanism:

```
registrant   0    baseRegistrar  0    ownerOf              0    NFT holder      0
controller   0    resolveEnsOwner 0   useAddressResolution 0    owner fallback  0
transfer    11    <- control: the document is being read
```

The word `controller` does not appear once in it.

**B. The upstream repository `ensdomains/bigname`, enumerated in full** - **498 items** (274 issues + 224 pull requests, all states) pulled through the issues endpoint rather than keyword search, because a search misses a duplicate worded differently. Three thematic neighbours, each read in full:

* **#204** *(closed)* verified primary names check the normalized claim, not the on-chain string - a normalization divergence in the indexer. It verifies differently; this report is that the app resolves to the wrong party.
* **#106** *(closed)* bound-names rows use inverted owner/registrant precedence - an inconsistency between two indexer API surfaces, not a UI assertion or a transfer recipient.
* **#372** *(open)* a cleared controller shadows the registrar fallback in `address_names.rs` - the closest of the three in theme, and still a Rust data-model question in the indexer.

All three live in **bigname, the Rust indexer**, which is not one of the five assets in scope. This report is in `apps/portal`, which is. I name them because they cut both ways: they show the team has examined owner/registrant precedence **at the indexing layer**, and that the resolution path used by the Explorer's transfer flow was not covered by that work.

**C. What cannot be measured, and is not claimed.** Other competitors' submissions are private until the competition closes. On a contest, "nobody else found it" is not demonstrable by anyone, including me. What is demonstrable is the sponsor's published baseline and the public upstream history, and both are above.

## Proof of Concept

## Proof of Concept

Reproduce in three steps. Steps 1 and 2 need no wallet and no write.

### 1. Confirm the two addresses differ on chain

```
$ cast call 0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e \
    'owner(bytes32)(address)' $(cast namehash sandytest1759027793098.eth) \
    --rpc-url https://ethereum-sepolia-rpc.publicnode.com
  0x742d35CC6634C0532925A3B8d0c9E3e0e8B9C5b1

$ cast call 0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85 \
    'ownerOf(uint256)(address)' $(cast keccak sandytest1759027793098) \
    --rpc-url https://ethereum-sepolia-rpc.publicnode.com
  0x94D2F676913722d704E09931B9ca306977Dbc4CE
```

### 2. Open the running app on that name

`pnpm --filter portal dev`, then `/<name>/ownership`. The page prints the controller under `Owner` and under `Manager`. The registrant appears only in the history column. Screenshot attached.

### 3. Run the tests, then break the fallback

```
intact     POC_WEB36_LIVE_sepolia.test.ts                OK   2 passed (2)
           POC_WEB36_owner_fallback_controller.test.ts   OK   7 passed (7)

mutate     resolvedAddress = ownerResult.value?.owner ?? null  ->  resolvedAddress = null
           POC_WEB36_LIVE_sepolia.test.ts                ECHEC 1 failed | 1 passed (2)
           POC_WEB36_owner_fallback_controller.test.ts   ECHEC 3 failed | 4 passed (7)

restore    2/2 and 7/7, git status --porcelain: 0 entries
```

The mutation is what makes the green runs mean something: a test that cannot fail proves nothing. One of the two files reads Sepolia through a bare `http()` transport, so its preconditions are read from the chain rather than asserted.
