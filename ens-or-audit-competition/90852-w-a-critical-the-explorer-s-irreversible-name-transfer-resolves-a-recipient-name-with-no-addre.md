# 90852 w a critical the explorer s irreversible name transfer resolves a recipient name with no address record to its registry controller silently delivering the name to the previous owner while&#x20;

**Submitted on Aug 27th 2026 at 15:26:53 UTC by @Valisthea for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90852
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves
  * Direct theft of user NFTs

## Description

## 1. Summary

The Explorer's "Send name" form accepts an ENS name in the recipient field. When that name publishes no `addr(60)` address record, the app does not stop, and it does not warn. It falls back to the name's **ENS registry owner**, the administrative controller, and places that address into the `safeTransferFrom` calldata of an irreversible name transfer.

The two on-chain fields do not mean the same thing:

* `addr(60)` is the record whose entire purpose is to answer "where should things be sent for this name". It is set by the holder, deliberately, to receive.
* The registry owner answers "who is currently permitted to edit this name's records". It is an administrative pointer, and ENS v1 does not move it when the name changes hands.

The application substitutes the second for the first, in the one flow it itself labels irreversible.

The substitution is invisible by construction. Provenance is discarded at the type level in the resolver before any UI code can see it, so no component downstream is able to distinguish an address the name declared from an address the name never declared, even if it wanted to. The form then renders a warning telling the user to check the recipient address, and the address it renders is the substituted one, so a user who follows that instruction precisely learns nothing.

This is not a rare corner. On Sepolia, the chain this application reads, **9 of the 23 registered `.eth` names I sampled publish no address record**, so the fallback branch is what runs for them. Section 5 then takes one of those names, performs the single action an ordinary marketplace sale performs, and runs the application's own resolution code against the result: it returns the **seller** of the name where the user meant the party now holding it.

## 2. Impact

This maps to the following bullet of the program's Critical impact list, word for word:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: \[...] showing the wrong recipient or amount on a transaction the user then approves

The transaction here is a name transfer, and the incorrect detail is the recipient itself. The user approves a `safeTransferFrom` whose destination is an address that the name they typed never nominated as its address.

The program's Critical list additionally names, under hijacking a name during a Manager flow, redirecting a transfer recipient. I flag a caveat rather than lean on it: that bullet is scoped to Manager flows, and this transfer lives in the Explorer. The C1 bullet quoted above carries no such scope and is the one I am relying on.

Concrete loss to the victim:

1. The name leaves the sender's wallet permanently. An ERC-1155 or ERC-721 transfer to a party who did not expect it has no recall path, and the program's own transfer warning says as much.
2. The intended recipient never receives it, and has no claim on it. The asset is now held by whoever the registry happened to name.
3. The receiving party may be a previous owner who sold the name, an address that has not been used in years, or a contract that cannot transfer ERC-1155 tokens back at all.

Neither party has to make a mistake for this to happen. The sender types a correct name, and the recipient supplied a correct name.

**On the preconditions.** Three conditions must hold together, and I state them plainly rather than argue past them. The recipient name must publish no `addr(60)` record. It must be absent from the V2 registry, so that resolution falls through to the V1 branch. And its registry controller must differ from the party the sender means.

Section 5 demonstrates all three at once, on Sepolia, using a name that already satisfies the first two today and a single ERC-721 transfer to create the third. None of the three is visible to the sender, and none is chosen by them: all three are properties of the recipient's name.

## 3. Root cause

### 3.1 The fallback, and the flow it was written for

`apps/portal/src/features/roles/helpers/addUser.handlers.ts:19-36`

```ts
  try {
    const resolved = await getEnsAddress(client, {
      name: nameOrAddress,
      universalResolverAddress,
    })

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

The comment on lines 27 and 28 states the design intent in the authors' own words: **"so the role can still be granted"**.

That is a defensible trade for a role grant. The worst outcome there is that a permission lands on the wrong administrative address, and permissions are revocable. It is not a trade anyone argued for in a transfer, and nothing at the transfer call site re-opens the question.

The address returned is the ENS v1 registry owner:

`apps/portal/src/utils/ens/resolveEnsOwner.ts:95-101`

```ts
  const v1Owner = await getOwnerV1(client, { name })
  if (v1Owner?.owner && v1Owner.owner !== zeroAddress) {
    return {
      owner: v1Owner.owner,
      registryAddress: v1EthRegistry,
      protocolVersion: 'ENSv1',
    }
  }
```

That is the **V1** branch, and it is not the first branch tried. `resolveEnsOwner` attempts the V2 registry first for any `.eth` name of two or more labels, and returns early if it finds an owner there:

`apps/portal/src/utils/ens/resolveEnsOwner.ts:84-93`

```ts
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

So the defect requires a recipient name that the V2 registry does not own, which is the case for every name that exists only in V1. That is a real condition on the finding and I demonstrate it rather than assume it: section 5.4 shows `getOwnerV2` returning the zero address for the witness name, so `resolveV2EthOwner` returns `null` at `:53` and execution reaches `:95`.

### 3.2 The library hands the app both addresses, correctly labelled, and the app keeps the wrong one

This is the part that makes the defect a choice rather than a gap in available information.

`getOwnerV1` is `getOwner` from `@ensdomains/ensjs/public/v1`, at the commit this repository pins. For an unwrapped second-level `.eth` name, which is what all eight witnesses in section 5.2 are, it returns two addresses, and its own comments say which is which:

ensjs `e96662c`, `packages/ensjs/src/actions/public/v1/getOwner.ts:216-226`

```ts
    // if there is a registrar owner, then it's not a subdomain but we have also passed the namewrapper clause
    // this means that it's an unwrapped second-level name
    // the registrant is the owner of the NFT
    // the owner is the controller of the records
    if (registrarOwner) {
      return {
        registrant: registrarOwner,
        owner: registryOwner,
        ownershipLevel: 'registrar',
      } as GetOwnerReturnType<contract>
    }
```

The library states it in as many words: **"the registrant is the owner of the NFT"** and **"the owner is the controller of the records"**. Both values are returned, on distinct, accurately named fields.

`resolveEnsOwner.ts:95-98` reads `.owner`, the controller of the records, and discards `.registrant`, the owner of the NFT. That value is then carried to `transferToken.ts:70` and used as the destination of an NFT transfer.

The application is not missing the address of the party that holds the name. It receives it, on a field named `registrant`, from a library that documents exactly what it means, in the same object, on the same call, and drops it.

Two further branches of the same function are worth noting, because they widen the affected set beyond the marketplace case:

ensjs `e96662c`, `packages/ensjs/src/actions/public/v1/getOwner.ts:227-236`

```ts
    if (registryOwner !== null) {
      // if there is no registrar owner, but the label length is two, then the domain is an expired 2LD .eth
      // so we still want to return the ownership values
      if (labels.length === 2) {
        return {
          registrant: null,
          owner: registryOwner,
          ownershipLevel: 'registrar',
        } as GetOwnerReturnType<contract>
      }
```

An **expired** `.eth` name still returns a controller, with `registrant: null`. The Explorer's check at `resolveEnsOwner.ts:96` only tests `owner`, so an expired name resolves successfully and the transfer proceeds to whoever controlled the name before it lapsed.

For **wrapped** names the library returns the unwrapped owner rather than the NameWrapper contract (`getOwner.ts:210-215`), so wrapped names are not affected by this defect. I checked this rather than assume it, and all eight witnesses in section 5.2 are unwrapped names taking the `registrarOwner` branch above.

### 3.3 The transfer recipient field consumes it

The helper is re-exported through a shared query:

`apps/portal/src/features/address/queries/getResolvedAddress.ts:6`

```ts
import { resolveAddressOrName } from '@/features/roles/helpers/addUser.handlers'
```

The hook that wraps it states in its own docstring that the transfer recipient is one of its consumers:

`apps/portal/src/features/address/hooks/useAddressResolution.ts:43-45`

```ts
 * Shared by every "name or address" input in the app (transfer recipient,
 * add-user sheets, …). Pair with `AddressNameInput` for the standard field UI,
 * or read `status`/`address` directly for a custom presentation.
```

The transfer form binds directly to it:

`apps/portal/src/features/transfer/components/SendNameForm.tsx:81-82`

```tsx
  const resolution = useAddressResolution(recipientInput)
  const { address: recipient, isResolving } = resolution
```

That same value is submitted:

`apps/portal/src/features/transfer/components/SendNameForm.tsx:107-108`

```tsx
    if (!recipient || !canStart) return
    startTransfer({ recipient, options: effectiveOptions })
```

It reaches calldata unchanged:

`apps/portal/src/features/transfer/helpers/transferToken.ts:67-70`

```ts
  const data = encodeFunctionData({
    abi: erc1155Abi,
    functionName: 'safeTransferFrom',
    args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
  })
```

There is no validation between line 82 and the calldata that asks where the address came from. The only guards applied are identity checks against the sender and the zero address:

`apps/portal/src/features/transfer/components/SendNameForm.tsx:87-89`

```tsx
  const isSelf = !!recipient && isAddressEqual(recipient, owner)
  const isZeroAddress = !!recipient && isAddressEqual(recipient, zeroAddress)
  const hasValidRecipient = !!recipient && !isSelf && !isZeroAddress
```

A substituted controller address is neither the sender nor zero, so it passes.

### 3.4 Provenance is destroyed, so nothing downstream can warn

This is why the defect cannot be caught by the interface as written, rather than merely not being caught.

`apps/portal/src/features/address/queries/getResolvedAddress.ts:26-31`

```ts
  const address = await resolveAddressOrName({
    client,
    nameOrAddress: params.nameOrAddress,
  })

  return ok<Address | null>(address)
```

The return type is a bare `Address | null`. Whether the value came from `addr(60)` or from the owner fallback is not represented anywhere in it. The distinction is gone before any consumer exists.

The hook's status enum then collapses both cases into one label:

`apps/portal/src/features/address/hooks/useAddressResolution.ts:77`

```ts
    .with({ resolved: P.nonNullable }, () => 'resolved' as const)
```

An address recovered from the fallback reports exactly the same `'resolved'` status as a genuine address record, so any component branching on status treats them identically.

The recipient preview has no branch for it:

`apps/portal/src/features/transfer/components/SendNameForm.tsx:237-251`

```tsx
  match({ recipient, isSelf, isZeroAddress })
    .with({ isSelf: true }, () => (
      <p className="text-sm mt-1.5 text-destructive">
        The recipient already owns this name.
      </p>
    ))
    .with({ isZeroAddress: true }, () => (
      <p className="text-sm mt-1.5 text-destructive">
        Can’t transfer to the zero address.
      </p>
    ))
    .with({ recipient: P.nonNullable }, ({ recipient }) => (
      <RecipientPreview address={recipient} />
    ))
    .otherwise(() => null)
```

Four branches: sender, zero address, resolved, nothing. There is no branch that could say "this name declares no address record, so we are using its controller instead", because by this point nothing in scope knows that.

## 4. The control that exists to prevent this is the control the defect disables

The transfer form already carries a warning aimed at precisely this class of mistake:

`apps/portal/src/features/transfer/components/SendNameForm.tsx:113-119`

```tsx
      <Alert variant="warning">
        <AlertTriangle className="size-4" />
        <AlertDescription>
          Transferring ownership of an ENS name is irreversible. Make sure you
          check the recipient address before proceeding.
        </AlertDescription>
      </Alert>
```

The app instructs the user to check the recipient address. The address it renders for checking is the substituted one. A user who reads the warning, follows it exactly, and inspects the address character by character sees a valid checksummed address with no error state, and learns nothing, because the displayed address is precisely the wrong address.

The single control the product offers against sending a name to the wrong party is defeated by the defect it was written to prevent.

There is a further amplifier immediately below:

`apps/portal/src/features/transfer/components/SendNameForm.tsx:253-256`

```tsx
const RecipientPreview = ({ address }: { address: Address }) => {
  const { data: primaryName, isLoading } = useQuery(
    getPrimaryNameQueryOptions(address),
  )
```

The preview reverse-resolves the substituted address and renders that address's own primary name, with an avatar. A user who typed one name can therefore be shown a confident, well-formatted recipient card carrying a different name, and nothing on the card marks it as a substitution. The interface is not merely silent about the swap, it presents positive, reassuring evidence about the wrong party.

## Proof of Concept

## 5. Proof of concept

This section is executed, not argued. It runs on a fork of **Sepolia**, which is the only chain the Explorer reads, and every resolution step below is performed by the same functions the application calls, imported from the same packages, with the universal resolver address taken from `sepoliaWithEns` exactly as `apps/portal/src/lib/constants/universalResolver.ts:4-7` does.

I did not send any transaction to a public network. The one state-changing call is an ordinary ERC-721 transfer, executed against a local fork.

### 5.1 The chain the app reads

Before anything else, the target's chain binding, because it decides which registry the recipient field is even able to query.

`apps/portal/src/lib/wagmi.ts:67`

```ts
  chains: [sepoliaWithEns],
```

`apps/portal/src/lib/wagmi/helpers.ts:14-17`

```ts
export const safeGetClient = fromThrowable(
  () => wagmiConfig.getClient() as Client<Transport, typeof sepoliaWithEns>,
  (e) => new WagmiClientError({ cause: e as CreateClientErrorType }),
)
```

That client reaches the owner resolver at `apps/portal/src/features/profile/hooks/useEnsOwner.ts:37,50`:

```ts
  const client = yield* safeGetClient()
    resolveEnsOwner(client, name),
```

The app is Sepolia-only, and its own comment says so, at `apps/portal/src/lib/wagmiL2.ts:11-13`:

```ts
 * Why: the explorer/indexer is bound to L1 Sepolia (Tenderly fork), but
 * setting an L2 primary name requires sending the `setName` transaction to
 * the L2 reverse registrar on the appropriate L2 (Optimism / Arbitrum / Base /
```

Everything below is therefore performed against Sepolia state.

### 5.2 The starting state, read from Sepolia

`treasury.eth` on Sepolia is an unwrapped second-level `.eth` name with **no resolver at all**, so it cannot publish an `addr(60)` record even in principle. Its registry controller and its ERC-721 registrant are currently the same address, so the app resolves it harmlessly today. Read-only, on any public Sepolia endpoint:

```sh
cast call 0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e \
  "owner(bytes32)(address)" $(cast namehash treasury.eth) --rpc-url <sepolia>
# 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750

cast call 0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e \
  "resolver(bytes32)(address)" $(cast namehash treasury.eth) --rpc-url <sepolia>
# 0x0000000000000000000000000000000000000000

cast call 0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85 \
  "ownerOf(uint256)(address)" $(cast --to-uint256 $(cast keccak "treasury")) --rpc-url <sepolia>
# 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
```

Note that the fallback already fires for this name today, because there is no address record. Only the divergence is missing, and the next step creates it the way the real world creates it.

### 5.3 One ordinary sale creates the divergence

Fork Sepolia and perform the single action a marketplace sale performs: transfer the ERC-721. Nothing else. In particular, **`reclaim` is not called**, because a buyer must call it themselves and many do not.

```sh
anvil --fork-url <sepolia> --port 8546          # forked at block 11578645

SELLER=0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
BUYER=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266   # anvil account 0
ID=$(cast --to-uint256 $(cast keccak "treasury"))

cast rpc anvil_impersonateAccount $SELLER --rpc-url http://127.0.0.1:8546
cast send 0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85 \
  "transferFrom(address,address,uint256)" $SELLER $BUYER $ID \
  --from $SELLER --unlocked --rpc-url http://127.0.0.1:8546
# status  1 (success)
```

State afterwards:

```
registry controller : 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750   the SELLER
ERC-721 registrant  : 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266   the BUYER
```

The buyer holds the name. The registry still names the seller. This is not a misuse of the registrar; it is what the registrar does: section 5.5 shows from its source that no transfer path writes to the registry.

### 5.4 The application's own code resolves the recipient to the seller

Now run the Explorer's recipient resolution against that fork. The script below imports `getEnsAddress` from `viem/actions` (the import at `addUser.handlers.ts:2`), the two `getOwner` functions from `@ensdomains/ensjs` (the imports at `resolveEnsOwner.ts:11-15`), and builds the chain object the way `wagmi.ts:51-58` builds it, with only the RPC URL changed to the fork.

```js
const customSepolia = {
  ...sepolia,
  rpcUrls: { default: { http: [FORK] }, public: { http: [FORK] } },
}
const sepoliaWithEns = extendChainWithEns(customSepolia)
const client = createPublicClient({ chain: sepoliaWithEns, transport: http(FORK) })
const universalResolverAddress = getChainContractAddress({
  chain: sepoliaWithEns,
  contract: 'ensUniversalResolver',
})

const resolved = await getEnsAddress(client, { name: 'treasury.eth', universalResolverAddress })
const v2owner = await getOwnerV2(client, { name: 'treasury.eth' })
const v1      = await getOwnerV1(client, { name: 'treasury.eth' })
const appResult = resolved ?? v1?.owner ?? null
```

Output:

```
universalResolver (from sepoliaWithEns): 0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe
recipient name typed by the user       : treasury.eth

1) addUser.handlers.ts:20  getEnsAddress( treasury.eth )
   -> null   (no address record)

2) resolveEnsOwner.ts:85   getOwnerV2( treasury.eth )
   -> 0x0000000000000000000000000000000000000000   (falls through to the v1 branch)

3) resolveEnsOwner.ts:95   getOwnerV1( treasury.eth )
   -> ownershipLevel : registrar
      registrant     : 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266   <-- the BUYER, who holds the name
      owner          : 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750   <-- the SELLER, who sold it

4) resolveAddressOrName( treasury.eth )  ->  0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750

   the user meant  : 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 (holds the ERC-721)
   the app returns : 0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750
```

Four things are established by that output:

{% stepper %}
{% step %}

## The address record is absent

The fallback branch at `addUser.handlers.ts:29` runs. Line 1 of the output establishes this.
{% endstep %}

{% step %}

## The V2 lookup does not short-circuit

`getOwnerV2` returns the zero address, so `resolveV2EthOwner` returns `null` at `resolveEnsOwner.ts:53` and execution reaches the V1 branch at `:95`. Line 2 of the output establishes this.
{% endstep %}

{% step %}

## The library returns both addresses correctly labelled

Line 3 shows `registrant` holding the buyer and `owner` holding the seller, exactly as `getOwner.ts:216-226` documents.
{% endstep %}

{% step %}

## The application keeps the wrong address

Line 4 shows that `resolveAddressOrName` returns the seller.
{% endstep %}
{% endstepper %}

That value is what `SendNameForm.tsx:81` binds to, what `SendNameForm.tsx:108` submits, and what `transferToken.ts:70` places into `safeTransferFrom` as the destination. A user who typed `treasury.eth`, intending the party that holds `treasury.eth`, sends their name to the party that sold it.

### 5.5 Why the registrar leaves the two apart, from its own source

The divergence created in 5.3 is not an artefact of the fork. It is how the registrar is built.

Source below is the **Sepolia** verified source of `BaseRegistrarImplementation` at `0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85`, compiler `v0.8.17+commit.8df45f5f`. I use the Sepolia deployment rather than the mainnet one deliberately: the two share an address but **not** their runtime bytecode, which I checked (`keccak(code)` is `0xd7d99a68...` on Sepolia against `0x15557327...` on mainnet), and the chain that matters here is the one the application reads. The mainnet deployment is an older compiler with the same structure on every point below.

The contract declares itself as:

```solidity
contract BaseRegistrarImplementation is ERC721, IBaseRegistrar, Ownable {
```

It inherits `transferFrom` from OpenZeppelin's `ERC721` and does not override it. The token transfer path therefore cannot write to the ENS registry: there is no code in this contract on that path at all. Only two statements in the whole contract touch the registry, and both are reached by an explicit call.

The first is inside registration, behind a flag:

```solidity
    function _register(
        uint256 id,
        address owner,
        uint256 duration,
        bool updateRegistry
    ) internal live onlyController returns (uint256) {
        require(available(id));
        require(
            block.timestamp + duration + GRACE_PERIOD >
                block.timestamp + GRACE_PERIOD
        ); // Prevent future overflow

        expiries[id] = block.timestamp + duration;
        if (_exists(id)) {
            // Name was previously owned, and expired
            _burn(id);
        }
        _mint(owner, id);
        if (updateRegistry) {
            ens.setSubnodeOwner(baseNode, bytes32(id), owner);
        }
```

The registrar exposes that flag as a public entry point, with a docstring that states the effect in as many words:

```solidity
    /**
     * @dev Register a name, without modifying the registry.
     * @param id The token ID (keccak256 of the label).
     * @param owner The address that should own the registration.
     * @param duration Duration in seconds for the registration.
     */
    function registerOnly(
        uint256 id,
        address owner,
        uint256 duration
    ) external returns (uint256) {
        return _register(id, owner, duration, false);
    }
```

The second registry write is the whole body of `reclaim`:

```solidity
    /**
     * @dev Reclaim ownership of a name in ENS, if you own it in the registrar.
     */
    function reclaim(uint256 id, address owner) external override live {
        require(_isApprovedOrOwner(msg.sender, id));
        ens.setSubnodeOwner(baseNode, bytes32(id), owner);
    }
```

Three consequences follow, and none requires anyone to behave unusually:

1. **Selling or transferring a name does not move the controller.** That is what 5.3 executes. The existence of `reclaim` as a separate entry point, and its docstring, are themselves the evidence that transferring the token does not do this.
2. **Registration can skip the registry write entirely.** `registerOnly` is a public function whose documented purpose is to register "without modifying the registry", so a name can be minted to one party while the registry continues to name whoever it named before.
3. **Expiry does not clear the controller.** A previously owned name is handled with `_burn(id)`, which burns the ERC-721 only, with no corresponding registry clear.

Consequence 3 is observable on Sepolia today with no fork at all. `alice.eth`, `bob.eth` and `eth.eth` each carry a registry controller while `ownerOf` reverts on the registrar, because `ownerOf` requires `expiries[tokenId] > block.timestamp`. That is exactly the branch ensjs documents at `getOwner.ts:227-236` as an expired 2LD, returning `registrant: null` with the controller still populated. Each of those three names also publishes no address record, so the Explorer's recipient field resolves them to whoever controlled them before they lapsed.

This also gives the defect a deliberate operator. A seller retains the controller until the buyer calls `reclaim`, and can simply decline to help. During that window, every name that anyone sends to that label through the Explorer arrives at the seller.

### 5.6 What I did not demonstrate

I did not execute the final `safeTransferFrom` with the substituted recipient. Doing so proves only that an ERC-1155 transfer sends to the address in its `to` argument, which is not in dispute, and it would require standing up a v2 name and a funded signer to add nothing to the argument. The defect is the value that reaches that argument, and section 5.4 executes the code that produces it.

I also did not test against the project's own Tenderly fork, whose RPC key is committed in the repository. The program's rules forbid exercising credentials recovered from the repo, so I used public Sepolia state and my own fork of it instead. If the team's fork carries different state for `treasury.eth`, the mechanism is unchanged: it depends on the registrar's behaviour, not on which particular name is used.

## 6. Why this is not a duplicate

**QA-01** ("role combinations on transfer break later profile and primary-name actions") is a functional defect in what happens to roles after a transfer completes. This report is about the destination of the transfer itself, decided before any transaction is signed, and it involves no role combination.

**EXP-INPUT-005** requires confusable characters, and is framed as display confusion. This requires none. The name typed is exact, correctly normalized and genuinely registered, and the failure is not in reading the name but in which of that name's on-chain fields is consulted afterwards.

**EXP-4337-002**, **EXP-4337-003** and **SEC-TXM-002** are transaction-manager issues concerning chain id, the `from` account, and the smart-account address. This defect is in the `to` argument of the call, is produced entirely inside `apps/portal` before the transaction manager is reached, and is unaffected by all three.

**SEC-MGR-003** concerns a persisted setting overriding the backend API base URL in the Manager. No persisted setting, no backend URL and no Manager code is involved here.

**R2-04** and **R2-05** are worker-side issues, in the OG image worker and the SendGrid webhook. This is client-side, in a wallet-facing form.

**R3-02 through R3-08** are transaction lifecycle defects, covering actors that never settle, retries, archive keys and error states. This defect produces a transaction that succeeds exactly as the lifecycle intends. Nothing stalls, nothing retries, nothing misparses. The transaction is well-formed and correctly executed, and it sends the name to the wrong address.

**QA-06** and **WEB-382** concern reverse resolution and primary names. This flow performs no reverse resolution to decide the recipient. It does perform one for display, at `SendNameForm.tsx:253-256`, and I raise that only as an amplifier of the presentation, not as the defect.

I have found no published entry whose root cause is the substitution of the registry owner for the address record, in any flow.

## 7. Recommended fix

Stop discarding the provenance, then refuse it at the flow that cannot be undone.

`resolveAddressOrName` should report which source produced the address:

```ts
type ResolvedAddress = {
  address: Address
  source: 'address-record' | 'registry-owner'
}
```

`getResolvedAddress` should carry that through instead of narrowing to `Address | null`, and `useAddressResolution` should surface it as a distinct status, for example `resolved-via-owner` alongside the existing `'resolved'`.

The transfer form should then treat `resolved-via-owner` as **not a valid recipient**. Concretely, at `SendNameForm.tsx:89`, `hasValidRecipient` should additionally require that the address came from an address record, and `RecipientResolvedContent` should gain a branch telling the user that the name publishes no address record and that they should ask the recipient for an address directly.

This keeps the behaviour the fallback was written for, since the role and registry add-user sheets can go on accepting it deliberately, and removes it only from the flow where being wrong is permanent.

If a smaller change is preferred, gating the fallback at the call site rather than inside the shared helper closes it too: pass an option such as `allowOwnerFallback` and have the transfer form omit it. That is a two-line change, at the cost of leaving the shared default unsafe for the next consumer.

I would also suggest, independently of the fix, that `RecipientPreview` not render a reverse-resolved name for an address the user did not type. Displaying a different name than the one entered, with an avatar and no explanation, adds unearned confidence to whatever address the resolution produced.

## 8. Full citation index

| Claim                                                                                                       | Location                                                                    |
| ----------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| The fallback, and the comment stating its purpose                                                           | `apps/portal/src/features/roles/helpers/addUser.handlers.ts:19-36`          |
| The address record lookup that must fail first                                                              | `.../addUser.handlers.ts:20-23`                                             |
| The null check that enters the fallback                                                                     | `.../addUser.handlers.ts:29`                                                |
| The owner is fetched and returned as the resolution                                                         | `.../addUser.handlers.ts:30-33`                                             |
| The owner returned is the ENS v1 registry owner                                                             | `apps/portal/src/utils/ens/resolveEnsOwner.ts:95-101`                       |
| Library returns registrant and owner as distinct fields, and documents that the registrant is the NFT owner | ensjs `e96662c`, `packages/ensjs/src/actions/public/v1/getOwner.ts:216-226` |
| An expired 2LD still returns a controller, with a null registrant                                           | ensjs `e96662c`, `.../v1/getOwner.ts:227-236`                               |
| Wrapped names return the unwrapped owner, so they are not affected                                          | ensjs `e96662c`, `.../v1/getOwner.ts:210-215`                               |
| The shared query imports the roles helper                                                                   | `apps/portal/src/features/address/queries/getResolvedAddress.ts:6`          |
| Provenance discarded at the type level                                                                      | `.../getResolvedAddress.ts:26-31`                                           |
| The hook's docstring names the transfer recipient as a consumer                                             | `apps/portal/src/features/address/hooks/useAddressResolution.ts:43-45`      |
| Name validity accepted, shared query run                                                                    | `.../useAddressResolution.ts:53,61`                                         |
| Both sources collapse to one `'resolved'` status                                                            | `.../useAddressResolution.ts:77`                                            |
| The transfer form binds to the shared resolver                                                              | `apps/portal/src/features/transfer/components/SendNameForm.tsx:81-82`       |
| The only recipient guards are self and zero                                                                 | `.../SendNameForm.tsx:87-89`                                                |
| The resolved value is submitted                                                                             | `.../SendNameForm.tsx:107-108`                                              |
| The irreversibility warning shown beside it                                                                 | `.../SendNameForm.tsx:113-119`                                              |
| The recipient preview has no fallback branch                                                                | `.../SendNameForm.tsx:237-251`                                              |
| The preview reverse-resolves and renders a name                                                             | `.../SendNameForm.tsx:253-256`                                              |
| The recipient reaches `safeTransferFrom` calldata                                                           | `apps/portal/src/features/transfer/helpers/transferToken.ts:67-70`          |
| The app binds to Sepolia only                                                                               | `apps/portal/src/lib/wagmi.ts:67`                                           |
| The client handed to the resolver is typed to that chain                                                    | `apps/portal/src/lib/wagmi/helpers.ts:14-17`                                |
| That client reaches the owner resolver                                                                      | `apps/portal/src/features/profile/hooks/useEnsOwner.ts:37,50`               |
| The team states the explorer is bound to an L1 Sepolia fork                                                 | `apps/portal/src/lib/wagmiL2.ts:11-13`                                      |
| V2 is tried first and returns early when it owns the name                                                   | `apps/portal/src/utils/ens/resolveEnsOwner.ts:84-93`                        |
| Universal resolver address derived from `sepoliaWithEns`                                                    | `apps/portal/src/lib/constants/universalResolver.ts:4-7`                    |
| ENS v1 Registry on Sepolia                                                                                  | `0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e`                                |
| BaseRegistrar on Sepolia, verified source, compiler v0.8.17                                                 | `0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85`                                |
| Sepolia universal resolver used in the PoC                                                                  | `0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe`                                |
| Registry write is conditional on `updateRegistry`                                                           | `BaseRegistrarImplementation._register`                                     |
| The only other registry write is `reclaim`                                                                  | `BaseRegistrarImplementation.reclaim`                                       |
| Token transfer performs no registry write                                                                   | `BaseRegistrarImplementation.transferFrom`                                  |
