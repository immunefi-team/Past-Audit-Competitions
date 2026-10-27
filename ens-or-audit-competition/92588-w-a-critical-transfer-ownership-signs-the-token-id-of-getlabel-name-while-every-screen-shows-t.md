# 92588 w a critical transfer ownership signs the token id of getlabel name while every screen shows the raw route param so the name that leaves the victim s wallet is not the name the flow displ&#x20;

**Submitted on Sep 10th 2026 at 19:28:59 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92588
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief

The Explorer's Transfer ownership flow takes the name from the route parameter and sends it down two different paths. Every surface the victim reads renders the parameter as-is. The token id that goes into the signed safeTransferFrom is derived from the ENSIP-15 normalization of that parameter. When the two differ, an ENS name leaves the victim's wallet and it is not the name any screen showed them.

The flow's own permission gate is split across the same two names, and the shape of that split is what green-lights the transfer in exactly the case where the wrong token can move.

The proof drives the real ENS v2 contracts on the Anvil Sepolia fork this repository ships and asserts on the end state: after the transfer, the displayed name is still the victim's and the undisplayed one belongs to the recipient.

## The split, in useTransferName.ts

Lines 112-115, the token id the flow will sign:

```
getEnsTokenId({ label: getLabel(name), registryAddress })
```

Line 131 and 132, the two forms side by side:

```
const common = { name, walletClient, publicClient: pc, signer, chainId }
const label = getLabel(name)
```

getLabel is apps/portal/src/utils/token/getLabel.ts and runs ens\_normalize then ens\_split, so label is the canonical form while common.name stays raw.

The two values are then handed to different steps:

Line 135, the ETH address step takes only common, so it writes against the RAW name:

```
setEthAddress({ ...common, recipient, id })
```

and setEthAddress.ts:37 resolves getResolver(client, { name }) on that raw name.

Lines 137 and 140, the detach steps take the NORMALIZED label:

```
detachNameResolver({ ...common, label, registryAddress, id })
detachNameRegistry({ ...common, label, registryAddress, id })
```

Line 143, the transfer step takes the token id derived from the NORMALIZED label:

```
transferToken({ ...common, registryAddress, tokenId, recipient, id })
```

and transferToken.ts:69-70 is the call that actually moves the asset:

```
functionName: 'safeTransferFrom',
args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x']
```

Meanwhile every string the victim reads is the raw one. useTransferName.ts:177 sets the per-step label:

```
transactionName: `${STEP_LABELS[step]} - ${name}`
```

and each helper's description does the same: transferToken.ts:88 is `Transfer ${name}`, detachNameResolver.ts:91 is `Detach resolver for ${name}`, detachNameRegistry.ts:92 is `Detach registry for ${name}`.

## How the victim gets there — the app renders the button itself

This is not a hand-typed URL. The whole path from the ownership page to the signed calldata carries the raw parameter, and no route on it normalizes.

apps/portal/src/routes/$name/ownership/index.tsx

```
:28   const { name } = Route.useParams()                          RAW
:31   useQuery(getEnsOwnerQueryOptions({ name }))                 RAW -> data.owner, data.registryAddress
:49   isConnectedOwner = isAddressEqual(address, data.owner)      owner of the RAW name
:58   useCanTransferName({ name, registryAddress, account: data.owner })
:102  canTransfer = isConnectedOwner && hasTransferRole && !isSubname
:116  <Link params={{ name }} to="/$name/ownership/transfer">     RAW forwarded
```

That file contains no isValidEnsName, isNormalized, normalize or getLabel call. Neither does the destination:

apps/portal/src/routes/$name/ownership/transfer.tsx

```
:22   const { name } = Route.useParams()                          RAW
:27   useQuery(getEnsOwnerQueryOptions({ name }))                 RAW
:104  isAddressEqual(address, data.owner)                         owner of the RAW name
:106  <AuthorizedTransfer name={name} registryAddress={data.registryAddress} owner={data.owner} />
```

and SendNameForm.tsx:85 finishes the handoff with the raw string:

```
useTransferName({ name, registryAddress, owner })
```

Compare apps/portal/src/routes/$name/index.tsx:442, which computes isValidEnsName(name) and returns InvalidNameMessage before the profile renders. That gate exists in the sibling route and is absent from both routes above.

## The permission gate is split the same way, and that is what enables it

AuthorizedTransfer in transfer.tsx calls useCanTransferName({ name, registryAddress, account: owner }). Inside that hook, useCanTransferName.ts:57-59 computes label = getLabel(name) — the CANONICAL label — and asks hasRoles(registryAddress, canonicalLabel, ROLE\_CAN\_TRANSFER\_ADMIN, account). The account it passes is the owner of the RAW name.

So the question the guard actually asks is: does the raw name's owner hold the transfer role on the canonical name's token. The form renders only when the answer is yes.

The hook's own doc comment says the check exists so the UI can "refuse to start such a transfer instead of walking the user into that trap", because the flow runs its irreversible detach steps before the token moves. The gate is doing its job against the canonical name. It is the only place in the flow that consults the canonical name for a permission decision, and it lets the flow proceed in precisely the situation where the canonical token is transferable — which is the situation where the wrong name moves.

## What lands on chain

A victim who holds both a stylised label and its plain form opens the ownership page for the stylised one, sees the Transfer button the app rendered, and walks the flow. The header, the step list and every transaction description say the stylised name. safeTransferFrom carries the plain name's token id. Afterwards the stylised name is still theirs and the plain one, which no screen ever named, belongs to the recipient.

The proof asserts exactly that, reading ownership through getState(tokenId).latestOwner on the leaf registry, which is how transferToken.ts itself documents ENS v2 ownership.

## Impact

Row 6288, malicious interactions with an already-connected wallet, modifying transaction arguments. The token id in the signed safeTransferFrom is not the token id of the name displayed on the screen that produced it.

Row 6286, hijacking an ENS name during a transfer flow. An ENS name is assigned to a third party by a flow that never displayed it, and the loss is irreversible without the recipient's cooperation. I have selected this row because the flow is a transfer and a name changes hands; I note that the row's wording says Manager flow while this particular flow lives in the Explorer, and I would rather raise that myself than have it read as a scope stretch. If only one row is accepted, 6288 fits without qualification.

## Limits, stated plainly

The recipient is chosen by the victim, so this is not a redirect of the transfer to an attacker address. The defect is which name moves, not who receives it.

The victim must hold both names for the flow to reach the form. If they hold only the stylised one, useCanTransferName returns false on the canonical label and transfer.tsx renders "Transfer not available" instead of the form, so the flow stops there. I am therefore not claiming the partial-state outcome — a raw-name ETH record written before a reverting transfer — as reachable through the UI; the guard does prevent that one.

Holding both a name and a variant of it is ordinary for anyone who registered defensively. It is also reachable without the victim registering anything: a third party can register the stylised label, which the deployed registrar accepts, and transfer that ERC-1155 to a victim who already owns the plain name.

## Recommended fix

Normalize the route parameter once, at the route boundary, and use that single value for the header, the step descriptions, the resolver lookup, the label, the ownership query and the token id. Adding the isValidEnsName gate that routes/$name/index.tsx:442 already applies to both ownership routes would also close it, by refusing to open the page for a name whose raw and canonical forms differ.

## Proof of Concept

## What this proves

The test below drives the deployed ENS v2 contracts on the Anvil Sepolia fork that this repository's own `pnpm e2e:infra:up` brings up. It registers two real names to the victim, runs the exact `safeTransferFrom` the Transfer ownership flow builds, and then reads ownership back off chain state. It asserts that the name shown on every screen did NOT move and that the name that was never shown DID.

It does not assert on string comparison. The assertions are on `getState(tokenId).latestOwner` after the transaction.

## Environment

Frozen revision under test: 1c9b47f. No live deployment is touched, everything runs against a local fork.

```
pnpm install
pnpm e2e:infra:up          # Anvil fork of Sepolia on 127.0.0.1:8545, chainId 11155111
```

Save the file below as:

```
apps/portal/src/features/transfer/poc-transfer-wrong-name-onchain.test.ts
```

Run it:

```
cd apps/portal
pnpm vitest run src/features/transfer/poc-transfer-wrong-name-onchain.test.ts
```

## Result, verbatim from the run

```
 RUN  v4.1.5 .../apps/portal

[poc] displayed on every screen : Treasurymtvwofbt.eth
[poc] label used for calldata    : treasurymtvwofbt
[poc] Treasurymtvwofbt.eth owner after  : 0xfd42fdd789cc8684add0ac50956f348b049753f5
[poc] treasurymtvwofbt.eth owner after  : 0x785d58a738efcc3b2fe0958be76c9de6084eb8aa
[poc] victim    = 0xfd42fdd789cc8684add0ac50956f348b049753f5
[poc] recipient = 0x785d58a738efcc3b2fe0958be76c9de6084eb8aa

 Test Files  1 passed (1)
      Tests  2 passed (2)
```

The victim opened the transfer page for Treasurymtvwofbt.eth. Every screen in the flow said Treasurymtvwofbt.eth. After the transaction, Treasurymtvwofbt.eth is still the victim's and treasurymtvwofbt.eth, which no screen ever named, belongs to the recipient.

Labels are suffixed with a timestamp so the test is idempotent, and both actors are freshly generated keys per run, so the pre-state assertions on line 155 and 156 are meaningful rather than inherited from an earlier run.

## Steps to reproduce in the UI

1. The victim holds a registration whose literal label is not its ENSIP-15 canonical form, and also holds the canonical one. Registering a non-canonical label is accepted by the deployed registrar. Either the victim registered both defensively, or a third party registered the stylised one and transferred the ERC-1155 to a victim who already held the plain name.
2. The victim opens the Explorer ownership page for the stylised name. routes/$name/ownership/index.tsx:28 reads the route parameter with no normalization gate.
3. The page renders a Transfer button, because :102 canTransfer is isConnectedOwner (owner of the RAW name) AND hasTransferRole (ROLE\_CAN\_TRANSFER\_ADMIN on the CANONICAL label, via useCanTransferName.ts:59 getLabel(name)) AND not a subname. Owning both names satisfies all three. :116 forwards the raw parameter into the transfer route.
4. transfer.tsx:22 reads the raw parameter again, :104 checks ownership of the raw name, and :106 passes the raw name to AuthorizedTransfer, which runs the same split role check and then renders SendNameForm. SendNameForm.tsx:85 calls useTransferName with the raw name.
5. The victim enters a recipient and walks the flow. The header, the step list and every transaction description show the stylised name.
6. On confirm, the signed safeTransferFrom carries getEnsTokenId({ label: getLabel(name) }) — the canonical label's token.
7. Read ownership back. The canonical name moved. The displayed one did not.

Note on step 3: if the victim held only the stylised name, the role check on the canonical label would return false and the page would render "Transfer not available" instead of the form. The guard stops that case. It does not stop this one.

## Full test file

/\*\*

* PoC — the Explorer's Transfer ownership flow moves a different registration
* than the one it displays.
*
* Drives the real ENS v2 contracts on the local Anvil Sepolia fork and asserts
* on the END STATE: which token actually changed hands.
*
* Requires: pnpm e2e:infra:up
*
* Production chain under test (frozen revision 1c9b47f):
* routes/$name/ownership/transfer.tsx:22 const { name } = Route.useParams() RAW
* ```
  (this route has no isValidEnsName / isNormalized gate, unlike
  ```
* ```
   routes/$name/index.tsx:442 which returns InvalidNameMessage)
  ```
* useTransferName.ts:113 getEnsTokenId({ label: getLabel(name) }) NORMALIZED
* useTransferName.ts:131 const common = { name, ... } RAW
* useTransferName.ts:132 const label = getLabel(name) NORMALIZED
* useTransferName.ts:177 transactionName: `${STEP_LABELS[step]} - ${name}` RAW
* transferToken.ts:69-70 safeTransferFrom(owner, recipient, tokenId, 1, 0x)
*
* getLabel is apps/portal/src/utils/token/getLabel.ts and applies ens\_normalize. \*/ import { createPublicClient, createWalletClient, http, parseAbi, encodeFunctionData, keccak256, stringToBytes, zeroAddress, zeroHash, type Address, } from 'viem' import { privateKeyToAccount } from 'viem/accounts' import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain' import { getTokenId } from '@ensdomains/ensjs/public/v2' import { universalResolverFindRegistriesSnippet } from '@ensdomains/ensjs-abi/universalResolver' import { permissionedRegistryGetStateSnippet } from '@ensdomains/ensjs-abi/v2/permissionedRegistry' import { packetToBytes } from 'viem/ens' import { toHex } from 'viem/utils' import { describe, expect, it } from 'vitest' import { getLabel } from '@/utils/token/getLabel'

const RPC = process.env.ANVIL\_RPC\_URL ?? '<http://127.0.0.1:8545>' const c = ensL1Contracts\[supportedL1Chains.sepolia] const REGISTRAR = c.ensEthRegistrar.address const REGISTRY = c.ensRegistry.address const USDC = c.usdc.address const DEDICATED\_RESOLVER = '0x640294a2b2d87e7f522db3e3e3e876764bce170d' as const

const REG = parseAbi(\[ 'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)', 'function commit(bytes32 commitment)', 'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)', 'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)', 'function MIN\_COMMITMENT\_AGE() view returns (uint64)', 'function isAvailable(string label) view returns (bool)', ]) const ERC20 = parseAbi(\[ 'function mint(address to, uint256 amount)', 'function approve(address spender, uint256 amount) returns (bool)', ]) const ERC1155 = parseAbi(\[ 'function safeTransferFrom(address from, address to, uint256 id, uint256 value, bytes data)', ])

const pc = createPublicClient({ transport: http(RPC) }) const chain = { id: 11155111, name: 'anvil', nativeCurrency: { name: 'E', symbol: 'E', decimals: 18 }, rpcUrls: { default: { http: \[RPC] } }, } as const const FUNDER = privateKeyToAccount('0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80') const randomKey = () => `0x${Array.from(crypto.getRandomValues(new Uint8Array(32))) .map((b) => b.toString(16).padStart(2, '0')).join('')}` as `0x${string}` const VICTIM = privateKeyToAccount(randomKey()) const RECIPIENT = privateKeyToAccount(randomKey())

const wc = (a: typeof FUNDER) => createWalletClient({ account: a, chain, transport: http(RPC) }) const wait = (h: `0x${string}`) => pc.waitForTransactionReceipt({ hash: h }) const rpc = (m: string, p: unknown\[] = \[]) => fetch(RPC, { method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ jsonrpc: '2.0', id: 1, method: m, params: p }), }).then((r) => r.json()) const send = async (acct: typeof FUNDER, to: Address, data: `0x${string}`) => wait(await wc(acct).sendTransaction({ to, data })) const DUR = 365n \* 24n \* 60n \* 60n

// happy-dom replaces globalThis.console, so write straight to stdout. const log = (m: string) => process.stdout.write(m + String.fromCharCode(10))

async function register(label: string, acct: typeof FUNDER) { await rpc('anvil\_setCode', \[acct.address, '0x']) const avail = await pc.readContract({ address: REGISTRAR, abi: REG, functionName: 'isAvailable', args: \[label] }) if (!avail) throw new Error(`${label} not available`) await send(FUNDER, USDC, encodeFunctionData({ abi: ERC20, functionName: 'mint', args: \[acct.address, 10\_000\_000\_000n] })) const secret = keccak256(stringToBytes(`s-${label}`)) const cm = await pc.readContract({ address: REGISTRAR, abi: REG, functionName: 'makeCommitment', args: \[label, acct.address, secret, zeroAddress, DEDICATED\_RESOLVER, DUR, zeroHash], }) await send(acct, REGISTRAR, encodeFunctionData({ abi: REG, functionName: 'commit', args: \[cm] })) const ma = await pc.readContract({ address: REGISTRAR, abi: REG, functionName: 'MIN\_COMMITMENT\_AGE' }) await rpc('evm\_increaseTime', \[Number(ma) + 10]) await rpc('evm\_mine') const \[b, p] = await pc.readContract({ address: REGISTRAR, abi: REG, functionName: 'getRegisterPrice', args: \[label, DUR, USDC], }) await send(acct, USDC, encodeFunctionData({ abi: ERC20, functionName: 'approve', args: \[REGISTRAR, (b + p) \* 2n] })) await send(acct, REGISTRAR, encodeFunctionData({ abi: REG, functionName: 'register', args: \[label, acct.address, secret, zeroAddress, DEDICATED\_RESOLVER, DUR, USDC, zeroHash], })) }

// The .eth leaf registry, discovered exactly as useTldData.ts does. const ethRegistry = async (): Promise

\=> { const regs = await pc.readContract({ address: c.ensUniversalResolver.address, abi: universalResolverFindRegistriesSnippet, functionName: 'findRegistries', args: \[toHex(packetToBytes('eth'))], }) return (regs as readonly Address\[])\[0] } // Production's own token-id read (useTokenId.ts -> ensjs getTokenId). const tokenIdOf = (label: string, registryAddress: Address) => getTokenId(pc as never, { label, registryAddress }) // ENS v2 ownership: getState(tokenId).latestOwner (transferToken.ts:4-5). const ownerOf = async (tokenId: bigint, registryAddress: Address) => { const st = await pc.readContract({ address: registryAddress, abi: permissionedRegistryGetStateSnippet, functionName: 'getState', args: \[tokenId], }) return (st as { latestOwner: Address }).latestOwner }

describe('Transfer ownership moves a different name than the one displayed', () => { it('the token that changes hands belongs to getLabel(name), not to the displayed name', async () => { const ts = Date.now().toString(36) const rawLabel = `Treasury${ts}` // literal on-chain label const canonLabel = `treasury${ts}` // its ENSIP-15 canonical form const rawName = `${rawLabel}.eth` // what the route param and every screen show const canonName = `${canonLabel}.eth`

```
for (const a of [VICTIM, RECIPIENT]) {
  await rpc('anvil_setBalance', [a.address, '0x21e19e0c9bab2400000'])
}

// The victim holds both: the stylised name and its plain form. Holding both
// is the ordinary case for anyone who registered a name and its variant.
await register(rawLabel, VICTIM)
await register(canonLabel, VICTIM)

const REG_ADDR = await ethRegistry()
const rawToken = await tokenIdOf(rawLabel, REG_ADDR)
const canonToken = await tokenIdOf(canonLabel, REG_ADDR)
expect(rawToken).not.toBe(canonToken)
expect((await ownerOf(rawToken, REG_ADDR)).toLowerCase()).toBe(VICTIM.address.toLowerCase())
expect((await ownerOf(canonToken, REG_ADDR)).toLowerCase()).toBe(VICTIM.address.toLowerCase())

// The victim opens /Treasury<ts>.eth/ownership/transfer. That route has no
// normalization gate, so `name` is the raw route param throughout the UI.
const displayedName = rawName

// useTransferName.ts:113 — the token id the flow will sign is derived from
// getLabel(name), i.e. the CANONICAL label, not the displayed one.
const labelUsedForCalldata = getLabel(displayedName)
expect(labelUsedForCalldata).toBe(canonLabel)
const signedTokenId = await tokenIdOf(labelUsedForCalldata, REG_ADDR)
expect(signedTokenId).toBe(canonToken)
expect(signedTokenId).not.toBe(rawToken)

// transferToken.ts:69-70 — the real safeTransferFrom the flow submits.
await send(VICTIM, REG_ADDR, encodeFunctionData({
  abi: ERC1155, functionName: 'safeTransferFrom',
  args: [VICTIM.address, RECIPIENT.address, signedTokenId, 1n, '0x'],
}))

// End state: the name on screen is still the victim's. The name that
// actually moved is the one that was never displayed.
const rawOwnerAfter = (await ownerOf(rawToken, REG_ADDR)).toLowerCase()
const canonOwnerAfter = (await ownerOf(canonToken, REG_ADDR)).toLowerCase()

log(`[poc] displayed on every screen : ${displayedName}`)
log(`[poc] label used for calldata    : ${labelUsedForCalldata}`)
log(`[poc] ${rawName} owner after  : ${rawOwnerAfter}`)
log(`[poc] ${canonName} owner after  : ${canonOwnerAfter}`)
log(`[poc] victim    = ${VICTIM.address.toLowerCase()}`)
log(`[poc] recipient = ${RECIPIENT.address.toLowerCase()}`)

// The displayed name did NOT move.
expect(rawOwnerAfter).toBe(VICTIM.address.toLowerCase())
// The undisplayed name DID.
expect(canonOwnerAfter).toBe(RECIPIENT.address.toLowerCase())
```

}, 600\_000)

it('control: for a canonical name the displayed name is the one that moves', async () => { const ts = Date.now().toString(36) const label = `plain${ts}` expect(getLabel(`${label}.eth`)).toBe(label) }) })
