# 91913 w a medium incomplete known public resolvers skip of locked record safety wipes eth text records on manager v1 v2 upgrade

## #91913 \[W\&A-Medium] Incomplete \`KNOWN\_PUBLIC\_RESOLVERS\` skip of locked-record safety wipes ETH/text records on Manager v1→v2 Upgrade

**Submitted on Sep 5th 2026 at 13:11:41 UTC by @Omisanin0 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91913
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

### Description

### Brief/Intro

Manager Upgrade of a locked NameWrapper name (`CANNOT_SET_RESOLVER`) whose live V1 public resolver is `0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5` never runs the locked-record safety check, never replays profile records, and still sends the name through `MigrationHelper.migrate`. On chain, `LockedWrapperReceiver` treats `0xE996` as a `PublicResolverSet` member and writes the V2 resolver pointer to empty PublicResolverV2 (`0xe7B9…`). The confirmation copy says "restore their records". After the tx, V2 `getResolver` is empty PublicResolverV2 (UniversalResolverV2 follows that pointer), so `addr(60)` and `url` are empty. The owner has no `ROLE_SET_RESOLVER` to point back at `0xE996` where the records still live, and Manager `name-resolver` verify expects `0xE996` so the journal will not resubmit. I reproduced this on a local Anvil fork of Sepolia against live bytecode and Manager's own `classifyName` / `resolverFor` / `buildMigrationHelperCall`.

### Vulnerability Details

Locked CSR migration is a known special case in this codebase. `LockedWrapperReceiver` ignores `Migration.Data.resolver` when `CANNOT_SET_RESOLVER` is burned. It reads the live V1 registry resolver. If that address is in `PublicResolverSet`, it replaces it with the immutable V2 PublicResolver (`PUBLIC_RESOLVER`). That new contract has its own empty storage. Nothing copies V1 `addr`/`text`/`contenthash` across.

The app already knows this is dangerous. `assertLockedResolverReplacementRecordSafety` exists specifically to block a locked rotation when the V1 inventory is non-empty (the atomic plan cannot replay into shared PublicResolverV2). `assertLockedPublicResolverSetMembership` exists to prove set membership before the wallet signs.

`isKnownPublicResolver` lives in the workspace package `@ens-apps/migration` (`packages/migration`), which Manager imports as `workspace:*`. That package is not a separate Immunefi asset. The in-scope failure is Manager Upgrade: `apps/manager` `resolverFor` / `buildMigrationPlan` skip the safety candidate list when `replacement == snapshot`, Manager's own live `PublicResolverSet.includes` probe is gated on that same list so it never asks about `0xE996`, `buildMigrationHelperCall` encodes `migrate()`, verify expects `0xE996`, and the wallet copy says restore. Impact lands on the Manager v1→v2 flow.

Both gates are keyed off `isKnownPublicResolver()`, a static list in `packages/migration/src/contracts/knownResolvers.ts`:

```ts
const KNOWN_PUBLIC_RESOLVERS: readonly Address[] = [
  getChainContractAddress({ chain: sepoliaWithEns, contract: 'ensPublicResolver' }),
  '0x8FADE66B79cC9f707aB26799354482EB93a5B7dD',
  '0x640294a2b2d87e7f522db3e3e3e876764bce170d',
  // ... more historical PRs, never 0xE996
]
```

ensjs Sepolia `ensPublicResolver` is `0x5239A812ec9A62F46dbb5de8f346C8eFe7553A9f`. Live Sepolia PublicResolver used by wrapped names (and by `e2e/infra/panoptes/contracts.json`) is `0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5`. That address is not in the list.

I called `PublicResolverSet` `0xf2794eBD70C1fa74094A9eC653DA1c2dF9f5a5A9` on Sepolia (and again on the fork):

| address                                | in app list? | `includes()` |
| -------------------------------------- | ------------ | ------------ |
| `0xE996…` (live PR)                    | no           | **true**     |
| `0x5239A812…` (ensjs)                  | yes          | false        |
| `0x640294a2…` (listed "Sepolia V1 PR") | yes          | false        |
| every other hardcoded list member      | yes          | false        |
| PublicResolverV2 `0xe7B9…`             | n/a          | false        |

So the only replaceable public resolver that actually matters on this deployment is the one the app does not know about.

Classify always sets locked CSR names to `keep-v1` (`classifyNames.ts`, `cannotSetResolverLocked` returns before the known-PR check):

```ts
if (cannotSetResolverLocked) {
  return 'keep-v1'
}
```

`resolverFor` for `keep-v1` then does:

```ts
isKnownPublicResolver(name.v1ResolverAddress)
  ? defaultResolver          // PublicResolverV2
  : name.v1ResolverAddress   // pass through 0xE996
```

For `0xE996`, `resolverFor` returns `0xE996`. `lockedResolverReplacementsWithoutAtomicReplay` drops the name when `replacementResolver == snapshot`:

```ts
if (replacementResolver.toLowerCase() === name.v1ResolverAddress.toLowerCase()) {
  return []  // not a candidate, safety never runs
}
```

`requiresPinnedPublicResolverMembership` also requires `isKnownPublicResolver`, so the live `PublicResolverSet.includes` probe is skipped.

`profileForName` returns null unless strategy is `to-owned-permres`, so there is no `setAddr`/`setText` replay in the batch.

The wallet dialog still says:

> Migrate the selected names and restore their records.

The encoded helper call sets `md.resolver = 0xE996`. The receiver ignores that field, sees `includes(0xE996) == true`, and injects PublicResolverV2. `_tokenRoleBitmapFromFuses` omits `ROLE_SET_RESOLVER` when CSR is burned.

Manager then verifies `ETHRegistry.getResolver(label) == expectedResolver` (`0xE996`). Actual is `0xe7B9…`. `SubmittedAtomicMigrationVerificationError`: confirmed batch "will not be resubmitted". That throw is inside `executeRemainingAtomicBatches`, so `executeMigration` never reaches `revokeTemporaryOperatorApprovals`. Cleanup only revokes temporary `eth-registry:hca` operator grants (NameWrapper helper approval is intentionally never cleaned). Leftover HCA operator is a side effect when that grant was in the plan, not a second bug.

If the V1 resolver had been `0x640294` (in the list), `resolverFor` would return PublicResolverV2, the name would be a safety candidate, and a non-empty inventory would block Upgrade. That is the intended defense. It does not fire for the live PR.

This is not WEB-806. WEB-806 is Manager profile UI missing subname records after a successful replay. Here the name is a locked 2LD, records are empty on PublicResolverV2, and verify would fail closed.

Not an ENS-contract bug report. `LockedWrapperReceiver` is doing what it is specified to do. The in-scope bug is that Manager keyed both app-side gates for this rotation off a stale static list instead of `PublicResolverSet`.

The V2 NFT stays with the owner. The name is not gone. Live V2 resolution is empty. They cannot `setResolver` back to `0xE996` (`ROLE_SET_RESOLVER` is omitted when CSR is burned). PublicResolverV2 `canModifyName` may still allow the V2 owner to re-type records onto PRV2 by hand (`names(node)` then V2 `findOwner`). That is hand recovery, not the atomic restore the UI advertised. V1 NFT is already in the graveyard. The PoC logs `canModifyName` as CHECK \[6]; it is not required for `POC_SUCCESS`.

### Impact Details

**Severity: Critical**, mapped to:

1. **Changing NFT metadata (primary).** Official Upgrade writes the V2 resolver pointer to empty PublicResolverV2. `addr(60)`, `text(url)`, and other supported records are not replayed. Live V2 resolution of the name no longer returns the owner's ETH address or profile texts. Confirmation copy is restore. That is unauthorized mutation of name/NFT metadata relative to what the user confirmed.
2. **Hijacking or bricking an ENS name during a Manager v1→v2 migration (secondary).** The V2 token is still the user's. The name is not deleted. After Upgrade: V1 token is in the graveyard, V2 resolver is empty PRV2, `ROLE_SET_RESOLVER` is false so they cannot reattach `0xE996` where the records still live, and the journal treats the confirmed tx as a verification incident that will not be resubmitted — so the official replay never runs. Payments and dapp logins that follow V2 `getResolver` (UniversalResolverV2) miss the owner until someone re-enters every record by hand.

This is not "direct theft of user NFTs" (they still own the V2 token) and not "resolver assigned to an attacker" (PRV2 is the official empty public resolver). I am not selecting those.

**Who is affected.** Any user who clicks Manager Upgrade on a locked `CANNOT_SET_RESOLVER` name pointing at `0xE996`. No extra attacker. Capital required is 0 beyond the user's own migrate gas.

**Not WEB-806, not UX-only.** On-chain `getResolver` and PRV2 storage change. The name still exists; its live records do not.

### References

* `packages/migration/src/contracts/knownResolvers.ts` (`KNOWN_PUBLIC_RESOLVERS` / `isKnownPublicResolver`)
* `packages/migration/src/service/classifyNames.ts` (`cannotSetResolverLocked` → `keep-v1`)
* `apps/manager/src/features/migration/service/encodeMigration.ts` (`resolverFor`)
* `apps/manager/src/features/migration/service/buildMigrationPlan.ts` (`lockedResolverReplacementsWithoutAtomicReplay`, skip when replacement == snapshot)
* `apps/manager/src/features/migration/service/migrationInvariants.ts` (`requiresPinnedPublicResolverMembership`)
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts` (`profileForName` null on `keep-v1`; `name-resolver` expectation)
* `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts` (`getResolver` must equal `expectedResolver`)
* `apps/manager/src/features/migration/service/migrationService.ts` (journal "will not be resubmitted"; cleanup after the throw)
* `apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx` ("restore their records")
* `apps/manager/src/features/migration/service/buildMigrationHelperCall.ts` (same `migrate()` encoding as the PoC)
* Live Sepolia: `PublicResolverSet` `0xf2794eBD…`, live PR `0xE996…`, PublicResolverV2 `0xe7B9…`, MigrationHelper `0xddC597…` (Manager `V2_CONTRACTS.MigrationHelper`)
* `LockedWrapperReceiver._migrateWrapped` CSR branch (callee, OOS as a *root*; cited only as the specified rotation the app failed to gate)

***

### Proof of Concept

## Proof of Concept

Local Anvil **fork of Sepolia only**. No txs on public Sepolia or mainnet.

The script uses Manager source in-place (`classifyName`, `resolverFor`, `createMigrationData`, `buildMigrationHelperCall`) and live Sepolia bytecode on the fork. It does not mock the subgraph, the indexer, `LockedWrapperReceiver`, or `MigrationHelper`.

Manager's production submit wraps the same helper calldata in HCA `executeByOwner`. This PoC sends `migrate()` from the EOA using `buildMigrationHelperCall` (the inner encoding Upgrade signs). `LockedWrapperReceiver` does not care who wrapped it. `name-resolver` verify-fail is inferred from `getResolver != expectedResolver`, which is exactly what `verifyAtomicMigrationBatch` checks. UniversalResolverV2 is not eth\_called; after migrate, V2 `getResolver` is PRV2, which is the pointer UR follows.

Setup for minting a fresh locked 2LD on the fork (so we do not touch anyone else's live name): impersonate the live BaseRegistrar Ownable owner to `addController`, then `register`. That is Foundry impersonation of live bytecode, not a mocked registrar. V2 `RESERVED` uses the same `ETHRegistrar` impersonation as the repo's `reserveInV2` (locked controller only has `ROLE_REGISTER_RESERVED`).

### How to run

No attachments. Copy the two files below into the frozen Manager tree.

```bash
# deps: foundry anvil, node 18+, tsx (global, or pnpm exec tsx from apps/manager)
# FORK_URL is read-only. All txs go to 127.0.0.1.

mkdir -p apps/manager/poc/f560-locked-csr-allowlist-wipe
# save run.sh and poc.ts from the blocks below into that directory

chmod +x apps/manager/poc/f560-locked-csr-allowlist-wipe/run.sh
cd apps/manager/poc/f560-locked-csr-allowlist-wipe
FORK_URL=https://ethereum-sepolia-rpc.publicnode.com ./run.sh
```

`ANVIL_RPC_URL` must be `http://127.0.0.1:*` or `http://localhost:*`. The script exits if it is not. Relative imports in `poc.ts` resolve to Manager `encodeMigration.ts` and `buildMigrationHelperCall.ts`.

Success prints `POC_SUCCESS`. Failure prints `POC_FAILED` and exits 1.

### Step by step (what the script does)

1. Start anvil `--fork-url <Sepolia> --chain-id 11155111`. Refuse any non-local RPC for txs.
2. `PublicResolverSet.includes(0xE996)` true; includes of every `KNOWN_PUBLIC_RESOLVERS` address false; `isKnownPublicResolver(0xE996)` false (their function).
3. Mint a unique locked 2LD on the fork: wrap with `CANNOT_UNWRAP | CANNOT_SET_RESOLVER` (fuses land as `196617` with PCC + IS\_DOT\_ETH), resolver `0xE996`, `setAddr(60)` = owner, `text(url)` = `https://alice.example`. Reserve the V2 label.
4. Feed that on-chain state into their `classifyName`. Get `locked-2ld` / `keep-v1`. Their `resolverFor` returns `0xE996`, so the locked-record safety candidate list is empty.
5. Contrast: if V1 PR were `0x640294` (in the list), `resolverFor` returns PublicResolverV2 and safety would run.
6. `createMigrationData` + `buildMigrationHelperCall` (Manager encoding). `NameWrapper.setApprovalForAll(MigrationHelper)` as Manager preflight does. Send `migrate()` to live helper `0xddC597…`.
7. Read V2 `getResolver`, PRV2 `addr`/`text`, `hasRoles(ROLE_SET_RESOLVER)`, wrapper owner, `PRV2.canModifyName`.

### Result from a successful run (fork block 11537754)

```
PublicResolverSet.includes(0xE996 live PR)     = true
isKnownPublicResolver(0xE996)                  = false
classifyName tokenType                         locked-2ld
classifyName resolverStrategy                  keep-v1
fuses                                          196617
resolverFor →                                  0xE996   (safety skipped)

BEFORE
  resolver     0xE996…
  addr(60)     victim
  text(url)    https://alice.example

AFTER migrate()  tx=0x9f10c9eff4cde409203cbd712efbf8ff683390623e31f8fc5039e45634d8d6d2
  V2 status         2 REGISTERED
  V2 owner          victim
  V2 getResolver    0xe7B9A25607E02da8145E4eB1836CA539e53F11f7   (empty PRV2)
  expectedResolver  0xE996…   (Manager name-resolver verify would fail)
  PRV2 addr(60)     0x
  PRV2 text(url)    ""
  ROLE_SET_RESOLVER false
  wrapper owner     0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750 (graveyard)

CHECKS [1]..[5] true
POC_SUCCESS
```

(Transcript from fork block 11537754, before CHECK \[6] was added. Re-running the inlined `poc.ts` also logs `PRV2.canModifyName`.)

### run.sh

```bash
#!/usr/bin/env bash
# F-560 PoC runner. Forks Sepolia into *local* Anvil. No public-chain txs.
set -euo pipefail

ROOT="$(cd "$(dirname "$0")" && pwd)"
MANAGER="$(cd "$ROOT/../.." && pwd)"
ANVIL_RPC_URL="${ANVIL_RPC_URL:-http://127.0.0.1:8545}"
FORK_URL="${FORK_URL:-https://ethereum-sepolia-rpc.publicnode.com}"
ANVIL_PORT="${ANVIL_PORT:-8545}"

if ! command -v anvil >/dev/null 2>&1; then
  echo "Need foundry anvil on PATH (https://book.getfoundry.sh/getting-started/installation)"
  exit 1
fi

if [[ "$ANVIL_RPC_URL" != http://127.0.0.1:* && "$ANVIL_RPC_URL" != http://localhost:* ]]; then
  echo "ANVIL_RPC_URL must be local (got $ANVIL_RPC_URL). Refusing public RPC."
  exit 1
fi

echo "Starting anvil fork of Sepolia"
echo "  fork-url  $FORK_URL"
echo "  listen    $ANVIL_RPC_URL"
anvil --fork-url "$FORK_URL" --port "$ANVIL_PORT" --chain-id 11155111 --silent &
ANVIL_PID=$!
cleanup() {
  kill "$ANVIL_PID" 2>/dev/null || true
  wait "$ANVIL_PID" 2>/dev/null || true
}
trap cleanup EXIT

for i in $(seq 1 60); do
  if curl -s -X POST -H 'content-type: application/json' \
    --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' \
    "$ANVIL_RPC_URL" | grep -q 0xaa36a7; then
    break
  fi
  sleep 1
  if [[ "$i" -eq 60 ]]; then
    echo "anvil did not become ready"
    exit 1
  fi
done

export ANVIL_RPC_URL
cd "$MANAGER"
if command -v tsx >/dev/null 2>&1; then
  TSX=tsx
elif [[ -x "$MANAGER/../../e2e/node_modules/.bin/tsx" ]]; then
  TSX="$MANAGER/../../e2e/node_modules/.bin/tsx"
else
  TSX="pnpm exec tsx"
fi
echo "Running poc.ts with $TSX"
$TSX "$ROOT/poc.ts"
```

Relative imports require this path: `apps/manager/poc/f560-locked-csr-allowlist-wipe/poc.ts`.

### poc.ts

```ts
/**
 * F-560 end-to-end PoC — local Anvil *fork* of Sepolia only.
 *
 * Does NOT send any transaction to public Sepolia/mainnet.
 * Does NOT mock subgraph, indexer, LockedWrapperReceiver, or MigrationHelper.
 *
 * Uses:
 *   - @ens-apps/migration `isKnownPublicResolver` / `classifyName` / `FUSES`
 *   - Manager `resolverFor` + `createMigrationData` (encodeMigration.ts)
 *   - Manager `buildMigrationHelperCall` (same migrate() encoding the app signs)
 *   - Live Sepolia bytecode on the fork (NameWrapper, 0xE996 PR, PublicResolverSet,
 *     LockedMigrationController, MigrationHelper 0xddC597…)
 */
import {
  classifyName,
  FUSES,
  hasFuse,
  isKnownPublicResolver,
  type V1Domain,
} from '@ens-apps/migration'
import {
  type Address,
  createPublicClient,
  createTestClient,
  createWalletClient,
  encodeFunctionData,
  http,
  isAddressEqual,
  keccak256,
  namehash,
  parseAbi,
  toHex,
  zeroAddress,
} from 'viem'
import { generatePrivateKey, privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { buildMigrationHelperCall } from '../../src/features/migration/service/buildMigrationHelperCall.ts'
import {
  createMigrationData,
  resolverFor,
} from '../../src/features/migration/service/encodeMigration.ts'

const ANVIL_RPC = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'

const LIVE_V1_PUBLIC_RESOLVER =
  '0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5' as Address
const KNOWN_LIST_SEPOLIA_PR =
  '0x640294a2b2d87e7f522db3e3e3e876764bce170d' as Address
const ENSJS_SEPOLIA_PR =
  '0x5239A812ec9A62F46dbb5de8f346C8eFe7553A9f' as Address

const V1 = {
  registry: '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e' as Address,
  baseRegistrar: '0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85' as Address,
  nameWrapper: '0x0635513f179D50A207757E05759CbD106d7dFcE8' as Address,
  controller: '0xfb3cE5D01e0f33f41DbB39035dB9745962F1f968' as Address,
} as const

const V2 = {
  ethRegistry: '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2' as Address,
  ethRegistrar: '0xa88553F454b77203B0D036A05c894d555EAAa2Cc' as Address,
  publicResolverSet: '0xf2794eBD70C1fa74094A9eC653DA1c2dF9f5a5A9' as Address,
  publicResolverV2: '0xe7B9A25607E02da8145E4eB1836CA539e53F11f7' as Address,
  migrationHelper: '0xddC597d937618849348E18Db5D631Ce747bCDeEF' as Address,
  lockedMigrationController:
    '0x5c39E36a69A9897F08954c71aCB1F36E0Bd4f409' as Address,
} as const

const ROLE_SET_RESOLVER = 1n << 24n
const CANNOT_UNWRAP = 1
const CANNOT_SET_RESOLVER = 8
const PARENT_CANNOT_CONTROL = 65536
const IS_DOT_ETH = 131072

const chain = { ...sepolia, rpcUrls: { default: { http: [ANVIL_RPC] } } }
const transport = http(ANVIL_RPC, { timeout: 60_000 })
const publicClient = createPublicClient({ chain, transport })
const testClient = createTestClient({ chain, transport, mode: 'anvil' })
const walletClient = createWalletClient({ chain, transport })

const log = (msg: string) => console.log(msg)
const section = (title: string) => {
  log('')
  log('='.repeat(72))
  log(title)
  log('='.repeat(72))
}

async function must(hash: `0x${string}`, what: string) {
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  if (receipt.status !== 'success') {
    throw new Error(`${what} reverted (tx ${hash})`)
  }
  log(`  ok  ${what}  tx=${hash}`)
  return receipt
}

function labelHash(label: string): `0x${string}` {
  return keccak256(toHex(label))
}

async function readIncludes(addr: Address): Promise<boolean> {
  return publicClient.readContract({
    address: V2.publicResolverSet,
    abi: parseAbi(['function includes(address addr) view returns (bool)']),
    functionName: 'includes',
    args: [addr],
  })
}

async function registerV1Name(params: {
  account: ReturnType<typeof privateKeyToAccount>
  label: string
  duration: bigint
}): Promise<void> {
  const { account, label, duration } = params
  // Live NameWrapper.registrar() is 0x57f1887a (same as ensjs). Its Ownable
  // owner is 0x4ad56feb…; ensjs ETHRegistrarController is NOT in
  // controllers(). On the fork we add the victim as a controller via the
  // live owner (standard Foundry impersonation of live bytecode) then mint.
  const brOwner = await publicClient.readContract({
    address: V1.baseRegistrar,
    abi: parseAbi(['function owner() view returns (address)']),
    functionName: 'owner',
  })
  log(`  BaseRegistrar.owner (live) ${brOwner}`)
  await testClient.impersonateAccount({ address: brOwner })
  await testClient.setBalance({ address: brOwner, value: 10n ** 18n })
  try {
    const addHash = await walletClient.sendTransaction({
      account: brOwner,
      to: V1.baseRegistrar,
      data: encodeFunctionData({
        abi: parseAbi(['function addController(address controller)']),
        functionName: 'addController',
        args: [account.address],
      }),
    })
    await must(addHash, 'BaseRegistrar.addController(victim) via live Ownable owner')
  } finally {
    await testClient.stopImpersonatingAccount({ address: brOwner })
  }

  const hash = await walletClient.sendTransaction({
    account,
    to: V1.baseRegistrar,
    data: encodeFunctionData({
      abi: parseAbi([
        'function register(uint256 id, address owner, uint256 duration)',
      ]),
      functionName: 'register',
      args: [BigInt(labelHash(label)), account.address, duration],
    }),
  })
  await must(hash, `BaseRegistrar.register(${label}.eth)`)
}

async function reserveV2(label: string, expiry: bigint) {
  const status = await publicClient.readContract({
    address: V2.ethRegistry,
    abi: parseAbi(['function getStatus(uint256 anyId) view returns (uint8)']),
    functionName: 'getStatus',
    args: [BigInt(labelHash(label))],
  })
  log(`  V2 getStatus(${label}) before reserve = ${status} (0=AVAILABLE 1=RESERVED 2=REGISTERED)`)
  if (status === 1) {
    log('  already RESERVED — locked controller can claim it')
    return
  }
  if (status === 2) {
    throw new Error(`V2 ${label}.eth is already REGISTERED — pick another label`)
  }
  await testClient.impersonateAccount({ address: V2.ethRegistrar })
  await testClient.setBalance({
    address: V2.ethRegistrar,
    value: 10n ** 18n,
  })
  try {
    const hash = await walletClient.sendTransaction({
      account: V2.ethRegistrar,
      to: V2.ethRegistry,
      data: encodeFunctionData({
        abi: parseAbi([
          'function register(string label, address owner, address registry, address resolver, uint256 roleBitmap, uint64 expiry) returns (uint256)',
        ]),
        functionName: 'register',
        args: [
          label,
          zeroAddress,
          zeroAddress,
          LIVE_V1_PUBLIC_RESOLVER,
          0n,
          expiry,
        ],
      }),
    })
    await must(hash, 'V2 register(owner=0) → RESERVED (impersonate ETHRegistrar, same as e2e reserveInV2)')
  } finally {
    await testClient.stopImpersonatingAccount({ address: V2.ethRegistrar })
  }
}

async function main() {
  section('F-560 PoC — locked CSR allowlist miss wipes records (Anvil Sepolia fork)')
  const chainId = await publicClient.getChainId()
  const block = await publicClient.getBlockNumber()
  log(`RPC            ${ANVIL_RPC}`)
  log(`chainId        ${chainId} (must be 11155111 Sepolia fork, txs stay local)`)
  log(`fork block     ${block}`)
  if (chainId !== 11155111) {
    throw new Error('Refusing to run: chainId is not Sepolia. Start anvil --fork-url <sepolia>')
  }
  const rpcHost = new URL(ANVIL_RPC).hostname
  if (rpcHost !== '127.0.0.1' && rpcHost !== 'localhost') {
    throw new Error(`Refusing to send txs to ${ANVIL_RPC}. PoC must target local anvil only.`)
  }

  section('STEP 0 — Live PublicResolverSet vs app KNOWN_PUBLIC_RESOLVERS (no mock)')
  const includeLive = await readIncludes(LIVE_V1_PUBLIC_RESOLVER)
  const includeKnownList = await readIncludes(KNOWN_LIST_SEPOLIA_PR)
  const includeEnsjs = await readIncludes(ENSJS_SEPOLIA_PR)
  const includePrv2 = await readIncludes(V2.publicResolverV2)
  log(`PublicResolverSet.includes(0xE996 live PR)     = ${includeLive}`)
  log(`PublicResolverSet.includes(0x640294 in list)   = ${includeKnownList}`)
  log(`PublicResolverSet.includes(0x5239A812 ensjs)   = ${includeEnsjs}`)
  log(`PublicResolverSet.includes(0xe7B9 PRV2)        = ${includePrv2}`)
  log(`isKnownPublicResolver(0xE996)     = ${isKnownPublicResolver(LIVE_V1_PUBLIC_RESOLVER)}  (THEIR knownResolvers.ts)`)
  log(`isKnownPublicResolver(0x640294)   = ${isKnownPublicResolver(KNOWN_LIST_SEPOLIA_PR)}`)
  log(`isKnownPublicResolver(0x5239A812) = ${isKnownPublicResolver(ENSJS_SEPOLIA_PR)}`)
  if (!includeLive) {
    throw new Error('Fork PublicResolverSet does not include 0xE996 — finding precondition missing')
  }
  if (isKnownPublicResolver(LIVE_V1_PUBLIC_RESOLVER)) {
    throw new Error('App list unexpectedly contains 0xE996 — finding would not trigger')
  }

  section('STEP 1 — Create locked CSR 2LD on the fork (real V1 controller / NameWrapper / 0xE996)')
  const account = privateKeyToAccount(generatePrivateKey())
  await testClient.setBalance({ address: account.address, value: 100n * 10n ** 18n })
  log(`victim EOA     ${account.address}`)
  const label = `f560poc${Date.now().toString().slice(-8)}`
  const name = `${label}.eth`
  const node = namehash(name)
  const duration = BigInt(365 * 24 * 60 * 60)
  log(`name           ${name}`)
  log(`node           ${node}`)

  await registerV1Name({ account, label, duration })

  const registrarExpiry = await publicClient.readContract({
    address: V1.baseRegistrar,
    abi: parseAbi(['function nameExpires(uint256 id) view returns (uint256)']),
    functionName: 'nameExpires',
    args: [BigInt(labelHash(label))],
  })
  await reserveV2(label, registrarExpiry)

  const alreadyApproved = await publicClient.readContract({
    address: V1.baseRegistrar,
    abi: parseAbi([
      'function isApprovedForAll(address owner, address operator) view returns (bool)',
    ]),
    functionName: 'isApprovedForAll',
    args: [account.address, V1.nameWrapper],
  })
  if (!alreadyApproved) {
    const approveHash = await walletClient.sendTransaction({
      account,
      to: V1.baseRegistrar,
      data: encodeFunctionData({
        abi: parseAbi([
          'function setApprovalForAll(address operator, bool approved)',
        ]),
        functionName: 'setApprovalForAll',
        args: [V1.nameWrapper, true],
      }),
    })
    await must(approveHash, 'BaseRegistrar.setApprovalForAll(NameWrapper)')
  }

  const wrapFuses = CANNOT_UNWRAP | CANNOT_SET_RESOLVER
  const wrapHash = await walletClient.sendTransaction({
    account,
    to: V1.nameWrapper,
    data: encodeFunctionData({
      abi: parseAbi([
        'function wrapETH2LD(string label, address wrappedOwner, uint16 ownerControlledFuses, address resolver)',
      ]),
      functionName: 'wrapETH2LD',
      args: [label, account.address, wrapFuses, LIVE_V1_PUBLIC_RESOLVER],
    }),
  })
  await must(
    wrapHash,
    `NameWrapper.wrapETH2LD fuses=${wrapFuses} (CANNOT_UNWRAP|CANNOT_SET_RESOLVER) resolver=0xE996`,
  )

  const urlRecord = 'https://alice.example'
  const setTextHash = await walletClient.sendTransaction({
    account,
    to: LIVE_V1_PUBLIC_RESOLVER,
    data: encodeFunctionData({
      abi: parseAbi([
        'function setText(bytes32 node, string key, string value)',
      ]),
      functionName: 'setText',
      args: [node, 'url', urlRecord],
    }),
  })
  await must(setTextHash, '0xE996.setText(url)')

  const setAddrHash = await walletClient.sendTransaction({
    account,
    to: LIVE_V1_PUBLIC_RESOLVER,
    data: encodeFunctionData({
      abi: parseAbi([
        'function setAddr(bytes32 node, uint256 coinType, bytes a)',
      ]),
      functionName: 'setAddr',
      args: [node, 60n, account.address],
    }),
  })
  await must(setAddrHash, '0xE996.setAddr(coin=60, victim)')

  const wrapperData = await publicClient.readContract({
    address: V1.nameWrapper,
    abi: parseAbi([
      'function getData(uint256 id) view returns (address owner, uint32 fuses, uint64 expiry)',
    ]),
    functionName: 'getData',
    args: [BigInt(node)],
  })
  const v1Resolver = await publicClient.readContract({
    address: V1.registry,
    abi: parseAbi(['function resolver(bytes32 node) view returns (address)']),
    functionName: 'resolver',
    args: [node],
  })
  const v1Addr = await publicClient.readContract({
    address: LIVE_V1_PUBLIC_RESOLVER,
    abi: parseAbi([
      'function addr(bytes32 node, uint256 coinType) view returns (bytes)',
    ]),
    functionName: 'addr',
    args: [node, 60n],
  })
  const v1Url = await publicClient.readContract({
    address: LIVE_V1_PUBLIC_RESOLVER,
    abi: parseAbi([
      'function text(bytes32 node, string key) view returns (string)',
    ]),
    functionName: 'text',
    args: [node, 'url'],
  })
  log(`  WRAPPER owner  ${wrapperData[0]}`)
  log(`  WRAPPER fuses  ${wrapperData[1]} (expect CSR bit: ${(Number(wrapperData[1]) & CANNOT_SET_RESOLVER) !== 0})`)
  log(`  V1 resolver    ${v1Resolver}`)
  log(`  V1 addr(60)    ${v1Addr}`)
  log(`  V1 text(url)   ${v1Url}`)
  if (!isAddressEqual(v1Resolver, LIVE_V1_PUBLIC_RESOLVER)) {
    throw new Error(`V1 resolver is ${v1Resolver}, need 0xE996`)
  }
  if ((Number(wrapperData[1]) & CANNOT_SET_RESOLVER) === 0) {
    throw new Error('CANNOT_SET_RESOLVER not burned')
  }
  if ((Number(wrapperData[1]) & CANNOT_UNWRAP) === 0) {
    throw new Error('CANNOT_UNWRAP not burned — name is not locked')
  }

  section('STEP 2 — THEIR classifyName + resolverFor (Manager encodeMigration) — safety skip')
  const domain: V1Domain = {
    id: node,
    labelName: label,
    labelhash: labelHash(label),
    name,
    resolver: { address: v1Resolver },
    owner: { id: V1.nameWrapper },
    registrant: { id: account.address },
    wrappedOwner: { id: wrapperData[0] },
    parent: { name: 'eth', wrappedDomain: null },
    registration: { expiryDate: wrapperData[2].toString() },
    wrappedDomain: {
      expiryDate: wrapperData[2].toString(),
      fuses: Number(wrapperData[1]),
    },
  }
  const classifiedResult = classifyName(domain, account.address)
  if (classifiedResult?.type !== 'classified') {
    throw new Error(`classifyName rejected: ${JSON.stringify(classifiedResult)}`)
  }
  const classified = classifiedResult.name
  log(`  classifyName tokenType         ${classified.tokenType}`)
  log(`  classifyName resolverStrategy  ${classified.resolverStrategy}`)
  log(`  classifyName fuses             ${classified.fuses}`)
  log(`  CSR fuse set                   ${hasFuse(classified.fuses, FUSES.CANNOT_SET_RESOLVER)}`)
  if (classified.tokenType !== 'locked-2ld') {
    throw new Error(`expected locked-2ld, got ${classified.tokenType}`)
  }
  if (classified.resolverStrategy !== 'keep-v1') {
    throw new Error(`expected keep-v1, got ${classified.resolverStrategy}`)
  }

  const expectedResolver = resolverFor(classified, V2.publicResolverV2, null)
  log(`  resolverFor(..., DefaultResolver=PRV2, null)`)
  log(`    → ${expectedResolver}`)
  log(`  snapshot V1 resolver           ${v1Resolver}`)
  const safetyWouldRun =
    expectedResolver.toLowerCase() !== v1Resolver.toLowerCase()
  log(`  lockedResolverReplacementsWithoutAtomicReplay candidate?`)
  log(`    replacement == snapshot → SKIP safety  (${!safetyWouldRun})`)
  log(`    (buildMigrationPlan.ts:180-184 — this is why empty-inventory / no-replay never runs)`)
  if (safetyWouldRun) {
    throw new Error('Safety would have run — 0xE996 path should skip it')
  }

  const contrast = resolverFor(
    {
      ...classified,
      v1ResolverAddress: KNOWN_LIST_SEPOLIA_PR,
    },
    V2.publicResolverV2,
    null,
  )
  log(`  CONTRAST resolverFor if V1 PR were 0x640294 (IN the app list): ${contrast}`)
  log(`    that replacement != snapshot, so safety WOULD run (and block non-empty records).`)
  log(`    0x640294 is NOT in PublicResolverSet, so the live-PR hole is specifically 0xE996.`)

  const migrationData = createMigrationData({
    label,
    owner: account.address,
    resolver: expectedResolver,
  })
  log(`  createMigrationData.resolver (md.resolver the helper encodes) = ${migrationData.resolver}`)
  log(`  LockedWrapperReceiver CSR branch IGNORES md.resolver and reads live V1 registry.`)

  section('STEP 3 — Execute THEIR buildMigrationHelperCall against live MigrationHelper on the fork')
  const helperCall = buildMigrationHelperCall([
    {
      name,
      tokenType: 'locked-2ld',
      parentName: 'eth',
      data: migrationData,
    },
  ])
  log(`  helper to     ${helperCall.to}`)
  log(`  expected to   ${V2.migrationHelper}`)
  if (
    typeof helperCall.to !== 'string' ||
    !isAddressEqual(helperCall.to, V2.migrationHelper)
  ) {
    throw new Error(`buildMigrationHelperCall target ${helperCall.to} != Manager MigrationHelper`)
  }

  // Manager grants the helper operator on NameWrapper before migrate()
  // (OPERATOR_APPROVAL_ABI / ensureMigrationApprovals).
  const helperApproved = await publicClient.readContract({
    address: V1.nameWrapper,
    abi: parseAbi([
      'function isApprovedForAll(address owner, address operator) view returns (bool)',
    ]),
    functionName: 'isApprovedForAll',
    args: [account.address, V2.migrationHelper],
  })
  if (!helperApproved) {
    const opHash = await walletClient.sendTransaction({
      account,
      to: V1.nameWrapper,
      data: encodeFunctionData({
        abi: parseAbi([
          'function setApprovalForAll(address operator, bool approved)',
        ]),
        functionName: 'setApprovalForAll',
        args: [V2.migrationHelper, true],
      }),
    })
    await must(opHash, 'NameWrapper.setApprovalForAll(MigrationHelper) — same as Manager preflight')
  }

  const migrateHash = await walletClient.sendTransaction({
    account,
    to: helperCall.to,
    data: helperCall.data,
    value: helperCall.value ?? 0n,
  })
  await must(
    migrateHash,
    'MigrationHelper.migrate(locked 2LD) — same encoding as Manager Upgrade',
  )

  section('STEP 4 — Before/after on-chain state (the impact)')
  const v2Resolver = await publicClient.readContract({
    address: V2.ethRegistry,
    abi: parseAbi(['function getResolver(string label) view returns (address)']),
    functionName: 'getResolver',
    args: [label],
  })
  const v2Owner = await publicClient.readContract({
    address: V2.ethRegistry,
    abi: parseAbi(['function getOwner(uint256 anyId) view returns (address)']),
    functionName: 'getOwner',
    args: [BigInt(labelHash(label))],
  })
  const v2Status = await publicClient.readContract({
    address: V2.ethRegistry,
    abi: parseAbi(['function getStatus(uint256 anyId) view returns (uint8)']),
    functionName: 'getStatus',
    args: [BigInt(labelHash(label))],
  })
  const hasSetResolver = await publicClient.readContract({
    address: V2.ethRegistry,
    abi: parseAbi([
      'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
    ]),
    functionName: 'hasRoles',
    args: [BigInt(labelHash(label)), ROLE_SET_RESOLVER, account.address],
  })
  const prv2Addr = await publicClient.readContract({
    address: V2.publicResolverV2,
    abi: parseAbi([
      'function addr(bytes32 node, uint256 coinType) view returns (bytes)',
    ]),
    functionName: 'addr',
    args: [node, 60n],
  })
  const prv2Url = await publicClient.readContract({
    address: V2.publicResolverV2,
    abi: parseAbi([
      'function text(bytes32 node, string key) view returns (string)',
    ]),
    functionName: 'text',
    args: [node, 'url'],
  })
  const stillOnE996Addr = await publicClient.readContract({
    address: LIVE_V1_PUBLIC_RESOLVER,
    abi: parseAbi([
      'function addr(bytes32 node, uint256 coinType) view returns (bytes)',
    ]),
    functionName: 'addr',
    args: [node, 60n],
  })
  const wrapperAfter = await publicClient.readContract({
    address: V1.nameWrapper,
    abi: parseAbi([
      'function getData(uint256 id) view returns (address owner, uint32 fuses, uint64 expiry)',
    ]),
    functionName: 'getData',
    args: [BigInt(node)],
  })

  log('  BEFORE (V1, pre-migrate)')
  log(`    wrapper owner     victim ${account.address}`)
  log(`    resolver          ${LIVE_V1_PUBLIC_RESOLVER}`)
  log(`    addr(60)          ${v1Addr}  (victim)`)
  log(`    text(url)         ${v1Url}`)
  log('  AFTER (V2, post-migrate — live LockedWrapperReceiver)')
  log(`    V2 status         ${v2Status} (2=REGISTERED)`)
  log(`    V2 owner          ${v2Owner}`)
  log(`    V2 getResolver    ${v2Resolver}`)
  log(`    expectedResolver  ${expectedResolver}  (app verify name-resolver)`)
  log(`    PRV2 addr(60)     ${prv2Addr}`)
  log(`    PRV2 text(url)    "${prv2Url}"`)
  log(`    ROLE_SET_RESOLVER ${hasSetResolver}`)
  log(`    wrapper owner now ${wrapperAfter[0]}  (graveyard; V1 NFT left the victim)`)
  log(`    0xE996 addr(60)   ${stillOnE996Addr}  (V1 storage uncleared; UR-V2 does not read it)`)

  let canModifyPrv2 = false
  try {
    canModifyPrv2 = await publicClient.readContract({
      address: V2.publicResolverV2,
      abi: parseAbi([
        'function canModifyName(bytes32 node, address operator) view returns (bool)',
      ]),
      functionName: 'canModifyName',
      args: [node, account.address],
    })
  } catch {
    canModifyPrv2 = false
  }
  log(`    PRV2.canModifyName(node, victim) ${canModifyPrv2}  (hand re-type on PRV2 if true; not the advertised restore)`)

  const resolverIsPrv2 = isAddressEqual(v2Resolver, V2.publicResolverV2)
  const verifyWouldFail = !isAddressEqual(v2Resolver, expectedResolver)
  const addrWiped = prv2Addr === '0x' || prv2Addr.length <= 2
  const urlWiped = prv2Url === ''
  const cannotRetarget = hasSetResolver === false

  log('')
  log('  CHECKS')
  log(`    [1] V2 resolver is empty PublicResolverV2 0xe7B9…     ${resolverIsPrv2}`)
  log(`    [2] Manager name-resolver verify would FAIL          ${verifyWouldFail}`)
  log(`       (expects 0xE996 keep-v1, chain wrote PRV2)`)
  log(`    [3] PRV2 ETH record empty (payments miss victim)     ${addrWiped}`)
  log(`    [4] PRV2 url empty                                   ${urlWiped}`)
  log(`    [5] no ROLE_SET_RESOLVER (cannot reattach 0xE996)    ${cannotRetarget}`)
  log(`    [6] PRV2.canModifyName(node, victim)                 ${canModifyPrv2}`)

  if (!resolverIsPrv2 || !verifyWouldFail || !addrWiped || !urlWiped || !cannotRetarget) {
    throw new Error('PoC did not reproduce the after-state — see CHECKS above')
  }

  section('IMPACT')
  log('  Official Manager Upgrade of a locked CANNOT_SET_RESOLVER name whose')
  log('  live V1 resolver is 0xE996 (NOT in KNOWN_PUBLIC_RESOLVERS, IS in')
  log('  PublicResolverSet) installs empty PublicResolverV2.')
  log('  UI copy is "Migrate the selected names and restore their records."')
  log('  Records are not replayed (keep-v1). Safety gate never ran.')
  log('  Journal verify (name-resolver) would refuse resubmit.')
  log('  Victim still owns the V2 token but ETH resolution is empty.')
  log('  They cannot setResolver back to 0xE996 where the records still sit.')
  log('  They may re-type onto PRV2 if canModifyName(V2 owner) holds — that')
  log('  is recovery by hand, not the promised restore.')
  log('')
  log('  Funds at risk (this program is PRE-DEPLOYMENT / Sepolia):')
  log('  not a token-drain number. At risk is ETH/token *resolution* of every')
  log('  locked CSR 2LD whose V1 public resolver is the live Sepolia PR 0xE996.')
  log('  After Upgrade, senders using UniversalResolverV2 pay 0x / fail-open.')
  log('')
  log('POC_SUCCESS')
}

main().catch((err) => {
  console.error('')
  console.error('POC_FAILED')
  console.error(err)
  process.exit(1)
})
```
