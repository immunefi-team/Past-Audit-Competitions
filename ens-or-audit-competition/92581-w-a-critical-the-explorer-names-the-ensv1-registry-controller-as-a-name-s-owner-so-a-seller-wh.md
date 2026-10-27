# 92581 w a critical the explorer names the ensv1 registry controller as a name s owner so a seller who has already handed over the nft is still shown as the owner on the official ens explorer

**Submitted on Sep 10th 2026 at 18:52:14 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92581
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

An ENSv1 `.eth` second-level name has two distinct on-chain rights and they can be held by two different addresses:

* the **registrant** — the holder of the BaseRegistrar ERC-721. This is the owner in every sense that matters: they can sell the name, transfer it, renew it, and reclaim the controller at will.
* the **controller** — the address in `ENSRegistry.owner(node)`. It may set records and subnames; it cannot transfer the name and it can be taken away by the registrant at any moment with one `reclaim()` call.

ensjs returns both. For an unwrapped `.eth` 2LD its v1 `getOwner` returns `{ registrant: <ERC-721 holder>, owner: <registry controller>, ownershipLevel: 'registrar' }`.

`resolveEnsOwner` keeps `owner` and throws `registrant` away:

```tsx
// apps/portal/src/utils/ens/resolveEnsOwner.ts:95-102
  const v1Owner = await getOwnerV1(client, { name })
  if (v1Owner?.owner && v1Owner.owner !== zeroAddress) {
    return {
      owner: v1Owner.owner,
      registryAddress: v1EthRegistry,
      protocolVersion: 'ENSv1',
    }
  }
```

That one value is the Explorer's single source of truth for who owns a v1 name. It feeds `/$name`, `/$name/ownership`, the profile sidebar, the ownership sidebar and the server-rendered OG social card. The party who actually holds the name is never displayed anywhere in the app.

The two rights diverge on the most ordinary event in the ENSv1 lifecycle: a sale. `BaseRegistrar.transferFrom` moves the ERC-721 and does not touch the registry controller. Until the buyer separately calls `reclaim()`, the seller remains the controller — and the Explorer keeps naming the **seller** as the owner of a name they no longer hold.

***

## Impact

Precondition: an unwrapped ENSv1 `.eth` name whose registrant and controller differ. That is the state every v1 name is left in immediately after a transfer, and it is a state the registrant can enter deliberately at any time with a single permissionless call.

### The seller keeps the badge

1. A owns `demo.eth`: registrant and controller are both A.
2. A sells the name and hands it over the only way the ERC-721 supports — `BaseRegistrar.transferFrom(A, B, tokenId)`. B is now the registrant. Nothing in the transfer touches the registry, so A is still the controller.
3. The official ENS Explorer's `/demo.eth/ownership` reports **Owner: A** and **Manager: A**. B, the person who paid and who holds the token, does not appear anywhere on the page.
4. A shows a second buyer, C, the Explorer page for `demo.eth` as proof of ownership. Every field agrees with A's claim. C pays A. A has nothing to deliver.

### Or the registrant fabricates it outright

A registrant does not need a sale. `ENSRegistry.setOwner(node, X)` — or `BaseRegistrar.reclaim(id, X)` — points the controller at any address the registrant chooses, for as long as they choose, reversible in one transaction. The Explorer will then assert that X owns the name. There is no forward check, no reconciliation with the registrar, and no badge or caveat anywhere on the page.

### What the buyer is denied

The same value gates what the app offers. Because the Explorer believes A owns `demo.eth`, B — the real holder, connected with the very wallet that holds the token — is shown someone else's address in the Owner row on their own name's page. The ownership surface of the app is inverted for exactly the person who just paid for the name.

The transaction-level defence does not apply here: no wallet prompt is involved. The wrong address is the app's answer to "who owns this name", which is the question a buyer asks the Explorer before they send money, and the answer is delivered on ENS's own domain with no qualifier.

***

## Root cause

### One field kept, one discarded

`@ensdomains/ensjs` v1 `getOwner` (`src/actions/public/v1/getOwner.ts:218-224`) for an unwrapped `.eth` 2LD:

```tsx
    // the registrant is the owner of the NFT
      return {
        registrant: registrarOwner,
        owner: registryOwner,
        ownershipLevel: 'registrar',
      }
```

`apps/portal/src/utils/ens/resolveEnsOwner.ts:95-102` takes `owner` and returns it as the name's owner. `registrant` is not read, not returned, and not present on the `resolveEnsOwner` result type at all, so no consumer can recover it.

### Every v1 ownership surface reads that one value

| File:line                                               | What it renders                                                                |
| ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `apps/portal/src/features/profile/hooks/useEnsOwner.ts` | `getEnsOwnerQueryOptions({ name })`, the shared query every surface below uses |
| `apps/portal/src/routes/$name/ownership/index.tsx`      | the Ownership tab's Owner and Manager rows                                     |
| `apps/portal/src/components/ProfileSidebar.tsx`         | the sidebar owner                                                              |
| `apps/portal/src/routes/$name/index.tsx`                | the overview                                                                   |
| `apps/portal/src/worker/ens.ts:84-90` (`resolveOwner`)  | the server-rendered OG social card, over the same `resolveEnsOwner`            |

`worker/ens.ts:84-90` is the same helper the React app uses, so the card shared into a chat or a marketplace listing carries the same wrong address.

### The app states the correct rule elsewhere, in its own comments

This is not an unsettled question inside the codebase. Three other places get it right and say so:

```tsx
// apps/portal/src/utils/names/mergeNamesData.ts:87-91
      v1Roles: {
        // For wrapped names: wrappedOwner controls both ownership and management
        // For unwrapped names: registrant is Owner, registry owner is Manager
        owner: relation.registrant || relation.wrappedOwner,
        manager: relation.owner || relation.wrappedOwner,
```

`apps/portal/src/features/migration/hooks/useMigrationStatus.ts:35` — "the registry `owner` is the controller, which may be a different address, so it is only a last-resort fallback". `apps/portal/src/features/registry/components/v2/MigrateRegistryPrompt.tsx:17` — "(wrappedOwner/registrant — NOT the registry controller)".

`resolveEnsOwner` implements the exact inverse of the rule `mergeNamesData` states in a comment.

### The same page prints the value twice, under two labels

`apps/portal/src/routes/$name/ownership/index.tsx:128-135` renders the Owner row from `data.owner` and, for an `ENSv1` name, renders `V1NameManagerRecord` beside it. That component independently reads the same registry slot:

```tsx
// apps/portal/src/features/ownership/components/V1NameManagerRecord.tsx:53-58
  } = useReadContract({
    abi,
    functionName: 'owner',
    args: [namehash(name)],
    address: ensRegistryAddress,
  })
```

and labels it **Manager**. So `ENSRegistry.owner(namehash(name))` appears on the page twice, once correctly labelled Manager and once incorrectly labelled Owner — which is why the screenshot shows the same address in both rows, and which forecloses any reading in which the Owner row was meant to carry the controller.

***

## Preempting the likely closure arguments

**"The buyer should call `reclaim()`."** They should, and nothing in the Explorer tells them to; the app does not show that the controller is stale, does not surface the registrant, and does not offer a reclaim action on this route. More to the point, the finding is not that the buyer failed to do something — it is that the app answers "who owns this name" with an address that does not own it, on the page whose entire purpose is that question.

**"This is long-standing ENSv1 semantics."** The registrar/registry split is. Presenting the controller as *the owner*, while discarding the registrant the same library call returned in the same object, is the app's choice. ENS's own v1 manager has always shown both, precisely because they differ.

**"No transaction is misdirected."** The impact row's first clause is *"display … an incorrect address"*, and the enumerated example is *"resolving or rendering an ENS name to an attacker-controlled address."* The Explorer renders a name to an address that does not hold it, and the funds a buyer sends on the strength of that page go to whoever holds the controller. The absence of a wallet prompt makes this worse, not better: there is no confirmation screen anywhere in the flow that could contradict the page.

**"The attacker needs a victim to buy something."** They need a buyer to check ownership on the official ENS Explorer before paying — which is the safe behaviour the Explorer exists to support.

**Recoverability.** For the double-sold buyer, none: the money is gone and the name was never theirs. For the first buyer, the display corrects itself only once they discover the stale controller and call `reclaim()` — which the app never prompts.

***

## Not a duplicate

* **#91314** (`useV1NamesForAddress` includes `resolvedAddress` in ensjs's default relations): that is a *list-membership* defect in a dashboard hook — which names appear under an address. This is the owner *field* of a single name, in `resolveEnsOwner`, and involves no relation filter.
* **#91900** (`addUser.handlers` falls back to the registry owner when a name has no `addr(60)`) is the closest, because it is the same function. It is not the same defect. #91900's claimed root cause is the **v2-before-v1 ordering** at `resolveEnsOwner.ts:83-92` plus the `addr(60)`→owner fallback at `addUser.handlers.ts:29-34`; its own PoC annotates the v1 branch with "This ordering is the defect", and it names "the one file:line that must change" as `addUser.handlers.ts:29-34`. It quotes `:95-102` only as correct context and never claims the `owner`vs-`registrant` choice is wrong. Its fix leaves `:98` untouched, and this report's fix (`return v1Owner.registrant`) leaves the ordering and the recipient fallback untouched.
* **#91838 / #91839** both open by explaining the ENSv1 registrant/controller split, but use it as an attacker *precondition* for planting a name. Their defective code is `packages/migration/src/service/classifyNames.ts:150`, `buildRoleGrantCalls.ts:23` and `migrationApprovals.ts:320-323` — the Manager's migration package, none of it in `apps/portal`, and neither concerns an ownership display.
* **#89270** (Manager profile: owner from the normalized name, records from the raw one): a different app, and a raw-vs-normalized split. No normalization is involved here.
* **#89441** (`allowMismatch` reverse record badged as verified): reverse resolution, not ownership.
* **WEB-1301** (known issue): *"The address view displays owner, registered, and expires values taken from the wallet's primary name, which is confusing in an address context."* That is the `/addr/…` view attributing a primary name's metadata to an address. This is the `/$name` view attributing the wrong address to a name, from a different code path.
* **QA-01 / WEB-1257 / WEB-1305** are role-combination and transfer-pending issues on v2 names.
* No filed report and no known issue touches `apps/portal/src/utils/ens/resolveEnsOwner.ts:95-102`.

***

## Recommendation

1. Return both. Add `registrant` to the `resolveEnsOwner` result for `protocolVersion: 'ENSv1'`, and make the Owner row render the registrant.
2. Label the controller for what it is — "Manager" / "Controller" — and never as "Owner".
3. Surface the divergence. When `registrant !== owner`, show it explicitly on `/$name` and `/$name/ownership`: the controller is stale and the name may have changed hands.
4. Offer the fix. Where the connected wallet is the registrant and the controller is someone else, offer `reclaim()`.
5. Apply the same rule to the OG card: `worker/ens.ts:84-90` (`resolveOwner`), called at `:147` and rendered at `og-render.ts:172`.

## Proof of Concept

## Proof of Concept

Executed against a local Anvil fork of Sepolia. No live network was touched.

### Environment

```bash
anvil --fork-url <sepolia> --chain-id 11155111 --port 8545
```

```
v1 BaseRegistrar   0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85
v1 ENSRegistry     0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e
```

### It is already true on live Sepolia, with no setup at all

`abc.eth` on Sepolia is an existing unwrapped v1 name whose registrant and controller already differ. Reading it through the Explorer's own `resolveEnsOwner`:

`apps/portal/.poc/v1-owner.test.ts` Run it with:

```bash
cd apps/portal && ./node_modules/.bin/vitest run .poc/v1-owner.test.ts --reporter=verbose
```

```tsx
import { createPublicClient, http, labelhash, namehash, parseAbi } from 'viem'
import { it } from 'vitest'
import { sepoliaWithEns } from '@/lib/wagmi'
import { resolveEnsOwner } from '@/utils/ens/resolveEnsOwner'

const RPC = '<http://127.0.0.1:8545>'
const brAbi = parseAbi(['function ownerOf(uint256) view returns (address)'])
const regAbi = parseAbi(['function owner(bytes32) view returns (address)'])

it('resolveEnsOwner returns the v1 controller, not the ERC-721 registrant', async () => {
  const client = createPublicClient({ chain: sepoliaWithEns, transport: http(RPC) })
  const BR = sepoliaWithEns.contracts.ensBaseRegistrarImplementation.address
  const REG = sepoliaWithEns.contracts.ensLegacyRegistry.address

  for (const name of ['abc.eth']) {
    const label = name.split('.')[0]
    const registrant = await client.readContract({
      address: BR, abi: brAbi, functionName: 'ownerOf', args: [BigInt(labelhash(label))],
    })
    const controller = await client.readContract({
      address: REG, abi: regAbi, functionName: 'owner', args: [namehash(name)],
    })
    const app = await resolveEnsOwner(client, name)

    console.log(`\n${name}`)
    console.log('  BaseRegistrar.ownerOf  (the ERC-721 holder) :', registrant)
    console.log('  ENSRegistry.owner      (the controller)     :', controller)
    console.log('  app resolveEnsOwner().owner                 :', app?.owner)
    console.log('  app protocolVersion                         :', app?.protocolVersion)
  }
})
```

Actual output:

```
abc.eth
  BaseRegistrar.ownerOf  (the ERC-721 holder) : 0xf4F1fd81201cA8156acfA8de8d7C8227cCD7666E
  ENSRegistry.owner      (the controller)     : 0xF34B448AC5f16d1165BD0A9d1a8d0C6F5605F38a
  app resolveEnsOwner().owner                 : 0xF34B448AC5f16d1165BD0A9d1a8d0C6F5605F38a <- CONTROLLER
  app protocolVersion                         : ENSv1
```

The Explorer answers `0xF34B…F38a`. The name is held by `0xf4F1…666E`.

### The sale, performed on the fork

`.poc/v1-sale.mjs` — the seller hands over the ERC-721 with one ordinary `transferFrom`, exactly as a sale does. Nothing else is touched. Run with `node v1-sale.mjs demo`.

```jsx
import { pub, rpc, send, fund, getAddress } from './lib.mjs'
const { labelhash, namehash, parseAbi, encodeFunctionData } = await import(VIEM)

const BR  = getAddress('0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85') // v1 BaseRegistrar
const REG = getAddress('0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e') // v1 ENSRegistry
const BUYER = getAddress('0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266') // the Explorer's connected wallet

const LABEL = 'demo', NAME = 'demo.eth'
const id = BigInt(labelhash(LABEL))
const brAbi  = parseAbi(['function ownerOf(uint256) view returns (address)','function transferFrom(address,address,uint256)'])
const regAbi = parseAbi(['function owner(bytes32) view returns (address)'])

const show = async (when) => {
  const registrant = await pub.readContract({ address: BR,  abi: brAbi,  functionName: 'ownerOf', args: [id] })
  const controller = await pub.readContract({ address: REG, abi: regAbi, functionName: 'owner',   args: [namehash(NAME)] })
  console.log(`${when.padEnd(7)} registrant(ERC-721 holder)=${registrant}  controller(ENSRegistry.owner)=${controller}`)
  return { registrant, controller }
}

const before = await show('BEFORE')
const SELLER = before.registrant
await fund(BUYER); await rpc('anvil_setBalance', [SELLER, '0x56BC75E2D63100000'])
await send(SELLER, BR, encodeFunctionData({ abi: brAbi, functionName: 'transferFrom', args: [SELLER, BUYER, id] }))
console.log(`\nSELLER ${SELLER} transferred the ERC-721 to BUYER ${BUYER}\n`)
await show('AFTER')
```

Actual output:

```
name: demo.eth   tokenId=26446653644574158990076181421067324203610722992339905729104254595349221766237
BEFORE  registrant(ERC-721 holder)=0xD613cb815902a1A6C469b899a18aB9B1901E8c30  controller(ENSRegistry.owner)=0xD613cb815902a1A6C469b899a18aB9B1901E8c30

SELLER 0xD613cb815902a1A6C469b899a18aB9B1901E8c30 transferred the ERC-721 to BUYER 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

AFTER   registrant(ERC-721 holder)=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266  controller(ENSRegistry.owner)=0xD613cb815902a1A6C469b899a18aB9B1901E8c30
```

### The rendered page

With `apps/portal/.env.local` pointing at the fork and the mock wallet connected as the **buyer** `0xf39F…2266`, run `vite dev --port 9623` and load `http://127.0.0.1:9623/demo.eth/ownership`. Captured with headless Chromium:

```
===== /demo.eth/ownership
  | Ownership
  | Expires    Mar 6, 2030 2:34am
  | Registered Sep 4, 2023 7:01am
  | Owner
  | Address
  | 0xD613…8c30
  | Manager
  | Address
  | 0xD613…8c30
  | Parent     eth
```

`0xD613…8c30` is the **seller**. The buyer is the connected wallet, holds the ERC-721, and is named nowhere on the page. Owner and Manager show the same address, so the page offers no way to tell the two rights apart or to notice that one of them has moved.

***
