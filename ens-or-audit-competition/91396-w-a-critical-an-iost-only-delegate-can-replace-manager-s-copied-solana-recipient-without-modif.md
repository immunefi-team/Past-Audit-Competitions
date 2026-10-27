# 91396 w a critical an iost only delegate can replace manager s copied solana recipient without modifying the sol record

**Submitted on Sep 1st 2026 at 06:00:19 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91396
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The PermissionedResolver intentionally supports record-part authority. A name owner can authorize another account to update one exact address coin type without authorizing other coins. This is a useful least-privilege boundary: an IOST payment operator can maintain IOST coin `291` while being unable to change Solana coin `501`.

Manager breaks that boundary in its public profile presentation.

`addressRecords` orders nonpreferred coins numerically, so IOST `291` precedes Solana `501` when the name has no preferred or ETH record. The profile helper chooses ETH or the first populated address as the main address. It then lowercases every address string—without considering coin family—to decide which chain icons belong on the main card and which chain-specific cards to remove.

An IOST-only delegate can choose:

```
Owner's legitimate SOL/501 B:
4zvwRjXUKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS

Delegate's IOST/291 A:
4zvwRjXuKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS
```

These strings differ at index 7 (`X` versus `x`) and decode to distinct bytes:

```
B = 3b6a27bcceb6a42d62a3a8d02a6f0d73653215771de243a63ac048a18b59da29
A = 3b6a27bccf6f9d4d5d2211110d58775b75e479a5de46c9ab0d5a97318b59da29
```

Yet:

```
A.toLowerCase() === B.toLowerCase()
```

Manager renders both as the same truncated text:

```
4zvwRj...tajS
```

On the attack profile, the real production DOM contained:

* a card labelled `Main receiving address`;
* the IOST icon and the SOL icon on that one card;
* no `Chain-specific addresses` section;
* zero copy buttons whose value/title was authoritative SOL `B`; and
* one copy button whose value/title was delegate-selected IOST `A`.

Clicking that real button made Chromium's real `navigator.clipboard.readText()` return `A`, not `B`.

The transaction-level terminal used the exact value read back through Chromium's real `navigator.clipboard` API and serialized in the browser receipt, on an isolated real Agave validator. The delegate first spent `0.001 SOL` to create `A` as a System Program account. Two independent payer accounts then each executed a normal `solana transfer A 1` with no `--allow-unfunded-recipient` flag and no warning. Both confirmed transactions contained a native System Program transfer to `A`.

Final balances were:

```
                                  before       after preparation       final
Copied unintended A                   0            0.001 SOL          2.001 SOL
Authoritative intended B          1 SOL            1.000 SOL          1.000 SOL
Payer one                          3 SOL                    —          1.999995 SOL
Payer two                          3 SOL                    —          1.999995 SOL
```

The off-curve destination cannot be controlled by a normal Ed25519 signer. The intended beneficiary therefore cannot recover the confirmed payments from that destination.

## Finding Description

### Intended behavior

PermissionedResolver's `authorizeAddrRoles()` accepts an exact `coinType`:

```solidity
function authorizeAddrRoles(
    bytes calldata toName,
    uint256 coinType,
    address account,
    bool grant
) external returns (bool updated)
```

Address-record resources bind both name and coin type. Conceptually:

```
part     = keccak256(abi.encode(coinType))
resource = keccak256(abi.encode(namehash(name), part))
```

The frozen `ROLE_SET_ADDR` bit is `1`. Consequently, the resource for coin `291` is distinct from the resource for coin `501`, even for the same name and account.

My PoC's real resolver reads after authorization were:

```
delegate: 0xD5A53300Ca4E89C81071C3030eC64FC5E8E6936A

roles(resource291, delegate): 1
hasRoles(resource291, 1, delegate): true

roles(resource501, delegate): 0
hasRoles(resource501, 1, delegate): false
hasRootRoles(1, delegate): false
```

The delegate's attempted direct `setAddr(node, 501, A)` reverted. Its `setAddr(node, 291, A)` succeeded. Final raw resolver state remained:

```
IOST/291 = 0x3b6a27bccf6f9d4d5d2211110d58775b75e479a5de46c9ab0d5a97318b59da29
SOL/501  = 0x3b6a27bcceb6a42d62a3a8d02a6f0d73653215771de243a63ac048a18b59da29
```

The contract therefore behaves correctly. The defect is Manager allowing the lower-privileged IOST record to determine the Solana-facing copy value.

### Actual behavior

#### 1. Manager builds one cross-family address inventory

`apps/manager/src/features/profile/data/records/address.tsx:5-48` creates records from the complete address encoder map and sorts every nonpreferred entry by numeric coin type:

```tsx
const rawAddressRecords: AddressRecordDef[] = Object.entries(
  coinTypeToNameMap,
).map(([coinType, [notation, name]]) => ({
  coinType: Number(coinType),
  name,
  notation,
  icon: coinIcons[notation],
}))

export const addressRecords: AddressRecordDef[] = [...rawAddressRecords].sort(
  (a, b) => {
    const aPref = getAddressPreference(a)
    const bPref = getAddressPreference(b)
    if (aPref !== bPref) return aPref - bPref
    return a.coinType - b.coinType
  },
)
```

IOST is coin `291`; Solana is coin `501`. Neither appears in the preferred list. With no earlier populated preferred/ETH address, the real ENSjs result therefore places IOST before Solana.

`apps/manager/src/features/profile/service/profileRecords.ts:82-169` supplies that inventory to ENSjs and preserves the decoded `records.coins` array:

```ts
const coins = unique([
  ...alwaysProbeAddressRecords.map(Number),
  ...addressRecords.map((record) => record.coinType),
  ...indexerRecords.coins,
]).filter(isSupportedCoinType)

const records = yield* fromPromise(
  getRecords(client, {
    name,
    texts,
    coins,
    contentHash: true,
    abi: true,
    ignoreInvalidCoinTypes: true,
  }),
  // ...
)

const result: ProfileRecordsResult = {
  texts: records.texts,
  coins: records.coins,
  // ...
}
```

`transformProfileRecords()` then assigns those coins directly to profile addresses in `features/profile/utils/transformRecords.ts:166-179`:

```ts
const withAddresses: ProfileRecords = {
  ...baseRecords,
  addresses: profile.coins,
  contentHash: profile.contentHash,
  abi: profile.abi,
  resolverAddress: profile.resolverAddress,
}
```

#### 2. Manager selects IOST as the main address

`apps/manager/src/features/profile/components/view/ProfileView.helpers.ts:196-244` preserves the address order and selects ETH or the first record:

```ts
const getAddressItems = (
  records: ProfileRecords,
): IndexedProfileAddressItem[] =>
  records.addresses.flatMap((address, index) => {
    const item = toAddressItem(address, index)
    return item ? [item] : []
  })

const getMainAddressCandidate = (
  records: ProfileRecords,
): IndexedProfileAddressItem | undefined => {
  const addresses = getAddressItems(records)
  return (
    addresses.find((address) => address.coinType === ETH_COIN_TYPE) ??
    addresses[0]
  )
}
```

For the attack profile, this selects delegate-controlled IOST `A`.

#### 3. Manager lowercases unrelated address families

The root is in the same file at lines `246-277`:

```ts
export const getReceivingAddressChains = (
  records: ProfileRecords,
): ProfileAddressItem[] => {
  const mainAddress = getMainAddressCandidate(records)
  if (!mainAddress) return []

  const mainValue = mainAddress.value.toLowerCase()
  return getAddressItems(records)
    .filter((address) => address.value.toLowerCase() === mainValue)
    .map(omitSourceIndex)
}

export const getChainSpecificAddresses = (
  records: ProfileRecords,
): ProfileAddressItem[] => {
  const mainAddress = getMainAddressCandidate(records)
  const mainValue = mainAddress?.value.toLowerCase()

  return getAddressItems(records)
    .filter((address) => address.value.toLowerCase() !== mainValue)
    .map(omitSourceIndex)
}
```

The comment gives EVM chains sharing one `0x...` address as the motivating example. The implementation does not limit comparison to EVM-compatible families and does not compare decoded bytes. It case-folds all textual forms.

IOST and Solana both use Base58, where case is data. The two values above are canonical under the frozen codecs, lower-case equal, and byte-distinct.

Manager directly resolves `@ensdomains/address-encoder@1.1.4`. The pinned ENSjs path used for independent PoC round trips resolves `@ensdomains/address-encoder@1.1.1`. The relevant `iost.ts`, `sol.ts`, and `base58.ts` files are byte-identical in those two installed versions. My PoC target manifest binds both copies, so the result does not conflate their dependency-resolution paths.

As a result:

```
getMainAddressCandidate()       -> IOST A
getReceivingAddressChains()     -> [IOST A, SOL B]
getChainSpecificAddresses()     -> []
```

#### 4. One card displays both icons but copies only IOST

`apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:55-123` renders every grouped icon but passes only the main record's value to the copy button:

```tsx
const MainAddressCard = ({ address, chains, name, avatarUrl }) => {
  // ...
  return (
    <CopyableButton
      // ...
      value={address.value}
    >
      <div>
        {/* name and address */}
        <ReceivingChainIcons chains={chains} />
      </div>
    </CopyableButton>
  )
}
```

`ProfileAddressesSection()` labels this card `Main receiving address` and renders the chain-specific section only from the already-filtered list:

```tsx
<h3>
  <Trans>Main receiving address</Trans>
</h3>
<MainAddressCard
  address={mainAddress}
  chains={receivingChains}
  // ...
/>

{chainAddresses.length > 0 ? (
  <div>
    <h3><Trans>Chain-specific addresses</Trans></h3>
    {/* ... */}
  </div>
) : null}
```

`apps/manager/src/components/atoms/CopyableButton/CopyableButton.tsx:13-48` writes the exact `value`:

```tsx
const handleClick: React.MouseEventHandler<HTMLButtonElement> = async (e) => {
  onClick?.(e)
  if (e.defaultPrevented || isDisabled) return
  await copyToClipboard(value)
  setCopied(true)
}
```

Thus the SOL icon has no chain-bound copy value. On the attack card it is visually associated with IOST `A`, and clicking copies `A`.

#### 5. Truncation hides the only difference

`apps/manager/src/lib/utils.ts:8-12` keeps six leading and four trailing characters:

```ts
export function truncateAddress(address?: string | null) {
  if (!address) return ''
  return `${address.slice(0, 6)}...${address.slice(-4)}`
}
```

The case difference is at index 7, so both records visibly render as:

```
4zvwRj...tajS
```

The full IOST value exists as the button title, but the legitimate Solana value is absent from the attack page. A user cannot compare the title against `B`, because Manager suppressed `B`'s card and exposed no copy control for it.

#### Exact first-party call chain

```
Real PermissionedResolver
  owner setAddr(node, 501, B)
  owner authorizeAddrRoles(name, 291, delegate, true)
  delegate setAddr(node, 291, A)
  delegate setAddr(node, 501, A) -> reverts
        |
        v
Real Panoptes indexes coin types and raw resolver bytes
        |
        v
Manager getProfileRecords(name)
  addressRecords order: ... 291 before 501 ...
  ENSjs getRecords(... coins ...)
  decoded coins: [{291,A}, {501,B}]
        |
        v
transformProfileRecords()
  addresses = profile.coins
        |
        v
getMainAddressCandidate()
  no ETH -> first address -> IOST A
        |
        v
getReceivingAddressChains()
  A.toLowerCase() == B.toLowerCase()
  returns both IOST and SOL icons
        |
        v
getChainSpecificAddresses()
  excludes SOL B
        |
        v
MainAddressCard
  label: Main receiving address
  icons: IOST + SOL
  visible value: 4zvwRj...tajS
  CopyableButton.value: A
        |
        v
Actual Chromium `navigator.clipboard` readback: A
        |
        v
Two ordinary local native-SOL transfers to A
  A gains 2 SOL
  authoritative B gains 0
```

#### Reproduced attacker and victim boundary

The scenario requires:

1. an ENS name with a legitimate Solana coin-501 record;
2. no populated ETH or earlier preferred address that supersedes these records as the main card;
3. an account authorized for the IOST coin-291 address record, but not Solana;
4. that delegate choosing a lower-case-colliding, byte-distinct IOST value;
5. one small attacker-funded creation of the chosen Solana destination; and
6. a payer using the public Manager receiving card that displays the SOL icon and copying the presented value before an ordinary Solana payment.

The attack does not require:

* name ownership;
* registry authority;
* resolver root authority;
* permission to modify Solana coin 501;
* the victim's wallet, session, signing credential, approval, or signature;
* a browser or wallet defect;
* an invalid resolver record;
* any Ed25519 signing capability for copied destination `A`;
* an oracle or trusted administrator mistake; or
* a public-chain exploit transaction.

The owner has deliberately trusted the delegate to maintain IOST. It has not trusted that account to determine Solana payments. The on-chain role system enforces that distinction; Manager does not.

### End-to-end attack scenario

#### Participants

* `owner`: owns the local ENSv2 name and the legitimate Solana `B` key.
* `delegate`: has only IOST/291 resolver authority and selects `A`.
* `payer one` and `payer two`: independent Solana accounts that copy and pay the Manager-presented recipient.
* `same-name control`: the name before the delegate writes IOST; it has only legitimate SOL/501 `B`.
* `same-name attack`: the exact same name after the one authorized IOST/291 write adds `A`; SOL/501 stays `B`.

#### Real components

My PoC uses:

* the exact frozen competition repository and Manager source;
* unmodified production Manager build output;
* the frozen Manager dependency graph;
* real V2 registrar and registry deployments on pinned local Sepolia Anvil;
* one real VerifiableFactory-created PermissionedResolver proxy;
* real `authorizeAddrRoles`, `setAddr`, `addr`, `roles`, `hasRoles`, and `hasRootRoles` calls;
* real digest-pinned Panoptes indexer and API;
* real Manager RPC and GraphQL traffic;
* real Chromium DOM, click, and `navigator.clipboard` API;
* official Agave `solana-cli 2.1.0`;
* real isolated `solana-test-validator 2.1.0`; and
* parsed native System Program transaction receipts and account balances.

There is no mock resolver, mock record, mock indexer, mocked RPC response, patched Manager helper, synthetic UI, clipboard shim, mock Solana program, mock transaction, or public-chain write.

For transparency, the registration fixture pays the real forked registrar through its deployed Mock USDC test token and advances disposable Anvil time by the registrar's on-chain `MIN_COMMITMENT_AGE`. Its inherited `anvil_setCode(owner, 0x)` guard is also executed, but the selected EOA has `code=0x` both before and after, so the guard is a proved no-op. These setup operations create a normal real V2 name; they do not replace or modify the resolver, role enforcement, Panoptes, Manager, browser clipboard, Solana client, validator, or native-payment terminal being proved.

#### Control

The Solana-only control is captured from the exact same V2 name, official resolver, production Manager build, RPC, profile route, browser, and clipboard path immediately before the authorized IOST write.

It produced:

```
Main card title/value: B
Visible value:         4zvwRj...tajS
Icons:                 SOL only
Buttons copying B:     1
Buttons copying A:     0
Clipboard:             B
```

This proves the normal application path correctly exposes the owner's Solana record before the independently authorized IOST collision is introduced.

#### Attack

The attack capture differs from that control by exactly one authorized record write on the same resolver:

```
SOL/501 by owner:      B
IOST/291 by delegate:  A
delegate roles 291:    1
delegate roles 501:    0
delegate root role:    false
direct SOL write:      reverted
```

It produced:

```
Main card title/value: A
Visible value:         4zvwRj...tajS
Icons:                 IOST + SOL
Chain-specific section: absent
Buttons copying B:     0
Buttons copying A:     1
Clipboard:             A
```

This same-name before/after control isolates the causal state change. Without the IOST record, Manager copies `B`; after only the lower-case-colliding IOST write, Manager copies `A` and suppresses `B`, even though SOL/501 is unchanged.

#### Native-SOL terminal

Before preparation, an ordinary CLI transfer to empty `A` failed with:

```
The recipient address (...) is not funded. Add
`--allow-unfunded-recipient` to complete the transfer
```

The delegate then used the explicit override once:

```bash
solana --url http://127.0.0.1:18925 \
  --keypair solana-delegate-keypair.json \
  transfer 4zvwRjXuKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS 0.001 \
  --allow-unfunded-recipient \
  --output json-compact
```

After that preparation, `A` existed as a normal, non-executable account owned by the System Program. Both independent payer commands were ordinary:

```bash
solana --url http://127.0.0.1:18925 \
  --keypair solana-payerOne-keypair.json \
  transfer 4zvwRjXuKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS 1 \
  --output json-compact
```

```bash
solana --url http://127.0.0.1:18925 \
  --keypair solana-payerTwo-keypair.json \
  transfer 4zvwRjXuKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS 1 \
  --output json-compact
```

Neither command contains `--allow-unfunded-recipient`; both exited `0` with empty stderr and no warning.

The confirmed signatures were:

```
preparation:
4f7qGbTp1qj8pvxmC1Z6atuEJTMiBLay6DrpWAZbVKz82ZZGNmcj62VYKNzvZn9fowTNK9RtyUGTyLHXfdtxbJcN

payer one:
4QryXrccicbBgPGpp67cJQtztwNKV64o3wZrb7J2UPQT67R1fDsmsHfjgFGVCLxndQaXLvnrEGRL6tiaAv58xNfk

payer two:
4hjMMMZhm5i4Fjo5NtuVRHKGAs6mzTGid5vppWUyDb4Sg9wjSYxwR7EGQrjzWuxKY78vR7yB1ovm6yWbL2tnvNwY
```

Each payer transaction had:

```
program:      system
programId:    11111111111111111111111111111111
type:         transfer
destination:  4zvwRjXuKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS
lamports:     1,000,000,000
meta.err:     null
```

The copied destination gained exactly `2,000,000,000` lamports after preparation. The authoritative Solana record gained zero.

## Root Cause

Manager determines cross-chain address equivalence by applying textual `toLowerCase()` equality without checking the coin family. It then uses the result to attach chain icons to one main-address card and remove the supposedly independent chain-specific cards. Because Base58 is case-sensitive, byte-distinct IOST and Solana addresses can compare equal only inside Manager. The surviving card retains the IOST value as its copy target even while displaying a SOL icon, so an IOST-only delegate can determine the value copied for a Solana payment without coin-501 authority.

## Impact Explanation

Manager tells a payer that one main address applies to both the IOST and Solana icons, while copying only the IOST record. The authoritative SOL/501 record does not determine the copied destination. A user who selects the card displaying the SOL icon and sends funds therefore submits to a recipient different from the name's authoritative SOL/501 record.

I reproduced the complete UI-to-payment path with two native-SOL transactions and exact recipient and balance deltas.

The deterministic witness makes the destination off-curve and therefore not a normal Ed25519 signer address. The exact demonstrated impact is nevertheless two successful native-SOL payments to `A` instead of the authoritative coin-501 destination `B`.

## Likelihood Explanation

The attacker needs only delegated authority for an earlier nonpreferred coin record and a case-colliding Base58 value. The victim follows the ordinary Manager profile flow: select the main receiving address displaying the SOL icon, copy it, and submit a standard payment. The misleading card exposes no authoritative SOL copy control or chain-specific comparison value. My PoC reproduces this flow with the same name before and after one authorized IOST update.

## Attack Path / Reproduction Path

{% stepper %}
{% step %}

### Set the legitimate Solana record and grant limited authority

The owner sets legitimate Solana coin `501` to `B` and grants the delegate only IOST coin `291` address authority.
{% endstep %}

{% step %}

### Write a case-colliding IOST address

The delegate writes byte-distinct IOST address `A`, for which `A.toLowerCase() === B.toLowerCase()`; a direct coin-501 write reverts.
{% endstep %}

{% step %}

### Observe Manager grouping the records

Manager selects IOST first, groups the IOST and SOL icons on one card, and removes the authoritative SOL card.
{% endstep %}

{% step %}

### Copy the displayed recipient

The victim clicks the copy button on the card displaying the SOL icon; Chromium returns `A`.
{% endstep %}

{% step %}

### Send funds to the unintended recipient

Ordinary Solana transfers using that copied value credit `A`, while `B` receives nothing.
{% endstep %}
{% endstepper %}

## External Preconditions

* The ENS name has a legitimate Solana coin-501 record and no populated ETH or earlier preferred address that supersedes IOST/Solana as the main card.
* A payer uses the public Manager receiving card displaying the SOL icon and copies its value before making a normal Solana payment.
* The selected destination exists as a funded System Program account so an ordinary Solana CLI payment proceeds without the unfunded-recipient override.

## Internal Preconditions

* A delegate has exact address-record authority for IOST coin `291`, but no coin-501 or root authority.
* The delegate writes an IOST Base58 value that differs from the Solana value only by case and decodes to different bytes.
* IOST precedes Solana in Manager's populated nonpreferred address ordering.

## Affected Code

* [`address.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/data/records/address.tsx#L5-L48)
* [`profileRecords.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileRecords.ts#L82-L169)
* [`transformRecords.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/utils/transformRecords.ts#L166-L179)
* [`ProfileView.helpers.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileView.helpers.ts#L196-L277)
* [`ProfileAddressesSection.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx#L55-L193)
* [`CopyableButton.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/components/atoms/CopyableButton/CopyableButton.tsx#L13-L48)
* [`utils.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/lib/utils.ts#L8-L12)

Supporting intended authorization behavior:

* [PermissionedResolver fine-grained address roles](https://github.com/ensdomains/contracts-v2/blob/48b3e2d39513b9dd32ef1850877a29009bc807b9/contracts/src/resolver/PermissionedResolver.sol#L362-L388)
* [PermissionedResolver address setter authorization](https://github.com/ensdomains/contracts-v2/blob/48b3e2d39513b9dd32ef1850877a29009bc807b9/contracts/src/resolver/PermissionedResolver.sol#L660-L683)
* [Agave CLI transfer path](https://github.com/anza-xyz/agave/blob/c1080de464cfb578c301e975f498964b5d5313db/cli/src/wallet.rs#L864-L1000)
* [Agave System Program transfer processing](https://github.com/anza-xyz/agave/blob/c1080de464cfb578c301e975f498964b5d5313db/programs/system/src/system_processor.rs#L183-L243)

## Recommended Mitigation

Do not use case-folded text equality across unrelated address families. Group receiving-address icons only for explicitly compatible coin types. For case-sensitive Base58 families, preserve each distinct `(coinType, value)` record and bind every displayed chain icon and copy action to that exact record. Add a regression test using the IOST/291 and SOL/501 values above: Manager must keep separate cards and the SOL copy action must return the authoritative coin-501 value.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/cd19d00eae5202438c2f4ed556ba22ca/63b1fe3cbfbdc08b9bc7103159cdfc09a75abe50>

## Proof of Concept

### Description

This full E2E PoC runs the frozen production Manager, deployed ENS V2 contracts on a fixed Sepolia fork, real Panoptes, Chromium, and an isolated real Agave validator with official `solana-cli` and `solana-test-validator` 2.1.0.

The control profile contains only authoritative SOL/501 address `B`; Manager shows one SOL icon and Chromium copies `B`. The owner then grants a delegate only IOST/291 authority. The delegate's direct SOL/501 write reverts, but its authorized IOST/291 write adds distinct Base58 address `A`, whose lowercase text collides with `B`. Manager now shows IOST and SOL icons on the `A` card, removes the authoritative `B` card, and Chromium copies `A`.

The exact clipboard value is then used in two ordinary native-SOL transfers. Both confirmed System Program transactions credit `A`; authoritative `B` gains zero. The target Manager, resolver, indexer, browser clipboard, Solana validator, transaction parser, and balance reads are not mocked or replaced.

### Where to place files and their names

The Gist clones these files under the frozen repository root:

```
poc/manager-cross-family-address-copy/
├── README.md
├── SOURCE_SHA256SUMS
├── TARGET_SHA256SUMS
├── browser.mts
├── constants.mts
├── docker-compose.poc.yml
├── package.json
├── panoptes-contracts.json
├── run.sh
├── setup.mts
├── solana-impact.mts
└── verify-final.mts
```

### Command to run

Run from the frozen repository root with Node.js 22, Docker/Compose, and official `solana-cli` plus `solana-test-validator` 2.1.0 installed:

```bash
mkdir -p poc
git clone https://gist.github.com/aydarkhusaenov/cd19d00eae5202438c2f4ed556ba22ca.git \
  poc/manager-cross-family-address-copy
git -C poc/manager-cross-family-address-copy checkout --detach \
  63b1fe3cbfbdc08b9bc7103159cdfc09a75abe50

corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
pnpm --filter e2e exec playwright install chromium

POC_LOG_DIR=/tmp/ens-cross-family-address-copy-evidence \
bash poc/manager-cross-family-address-copy/run.sh
```

### Logs proving the bug

```
ENS25_ADDRESS_PAIR {"correctAddress":"4zvwRjXUKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS","collidingAddress":"4zvwRjXuKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS","correctBytesHex":"3b6a27bcceb6a42d62a3a8d02a6f0d73653215771de243a63ac048a18b59da29","collidingBytesHex":"3b6a27bccf6f9d4d5d2211110d58775b75e479a5de46c9ab0d5a97318b59da29","lowercaseCollision":true,"differingIndices":[7],"managerRendering":"4zvwRj...tajS","correctOnCurve":true,"collidingOnCurve":false,"correctSeedHex":"0000000000000000000000000000000000000000000000000000000000000000","signatureProofHex":"860e744ff849d83508f1ffc40a74d32200439bf2b354a5ca2bc6bb70151b956e6065fbebc3b982c7b12a8d1401d1dc2cf1645f9fb389300f4451de735cebd002"}
ENS25_REAL_RESOLVER_ROLE_AND_CONTROL_INDEXER_READY {"attackName":"000-ens25-cross-family-attack-1788211739.eth","attackResolver":"0xddf13adc56ddCDAfA7d53Ca6A0297af906D5D798","owner":"0xD47815115B7b84ff6af26702E9D947bC040171Ce","delegate":"0xD5A53300Ca4E89C81071C3030eC64FC5E8E6936A","roles291":"1","roles501":"0","delegateCanWriteSol":null}
ENS25_MANAGER_CONTROL {"name":"000-ens25-cross-family-attack-1788211739.eth","title":"4zvwRjXUKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS","visibleText":"000-ens25-cross-family-attack-1788211739.eth\n4zvwRj...tajS","icons":["http://127.0.0.1:13086/assets/sol-icon-D70fGrY8.svg"],"chainSpecificHeadingCount":0,"clipboard":"4zvwRjXUKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS","rpcStatus":200,"rpcRequestSha256":"1bb88d51461ad857824b2f921c96cae82e62fdcb7a77b56558c6c07878c12df2","rpcResponseSha256":"ae5d34603462dc71545caf1a83c2d92d92e4470cf18dc380d8b5643d1ee63c50"}
ENS25_IOST_ONLY_ATTACK_APPLIED {"name":"000-ens25-cross-family-attack-1788211739.eth","resolver":"0xddf13adc56ddCDAfA7d53Ca6A0297af906D5D798","attackHash":"0x7761ca8c6754e7317cc105d4a0242e3ec8a51e6110342d42d71702d14ce48f5b","delegateCanWriteSol":false,"raw291":"0x3b6a27bccf6f9d4d5d2211110d58775b75e479a5de46c9ab0d5a97318b59da29","raw501":"0x3b6a27bcceb6a42d62a3a8d02a6f0d73653215771de243a63ac048a18b59da29"}
ENS25_MANAGER_ATTACK {"name":"000-ens25-cross-family-attack-1788211739.eth","title":"4zvwRjXuKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS","visibleText":"000-ens25-cross-family-attack-1788211739.eth\n4zvwRj...tajS","icons":["http://127.0.0.1:13086/assets/iost-icon-DqPKJjE4.svg","http://127.0.0.1:13086/assets/sol-icon-D70fGrY8.svg"],"chainSpecificHeadingCount":0,"clipboard":"4zvwRjXuKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS","rpcStatus":200,"rpcRequestSha256":"1bb88d51461ad857824b2f921c96cae82e62fdcb7a77b56558c6c07878c12df2","rpcResponseSha256":"20fbe55afc959ba413efc548159ab8ecc5b1af46e5d4ce2d3adc79c9002a2914"}
ENS25_FULL_E2E_UNINTENDED_SOL_RECIPIENT_PASS {"attackName":"000-ens25-cross-family-attack-1788211739.eth","delegate":"0xD5A53300Ca4E89C81071C3030eC64FC5E8E6936A","roles291":"1","roles501":"0","managerClipboard":"4zvwRjXuKGfvwnParsHAS3HuSVzV5cA4McphgmoCtajS","ordinarySignatures":["4QryXrccicbBgPGpp67cJQtztwNKV64o3wZrb7J2UPQT67R1fDsmsHfjgFGVCLxndQaXLvnrEGRL6tiaAv58xNfk","4hjMMMZhm5i4Fjo5NtuVRHKGAs6mzTGid5vppWUyDb4Sg9wjSYxwR7EGQrjzWuxKY78vR7yB1ovm6yWbL2tnvNwY"],"unintendedGainLamports":2000000000,"intendedGainLamports":0}
ENS25_POC_PASS IOST-only delegate caused two ordinary native-SOL payments to Manager's unintended copied recipient
ENS25_CLEANUP_PASS local nodes, containers, and fork state removed
```

**Full source:** [**immutable twelve-file PoC Gist**](https://gist.github.com/aydarkhusaenov/cd19d00eae5202438c2f4ed556ba22ca/63b1fe3cbfbdc08b9bc7103159cdfc09a75abe50)
