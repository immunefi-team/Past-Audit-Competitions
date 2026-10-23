# 92779 w a high owner signed hca intents are signed from the orchestrator s returned intent with no execution equivalence check on the client or on chain

## #92779 \[W\&A-High] Owner-signed HCA intents are signed from the orchestrator's returned intent with no execution-equivalence check, on the client or on chain

**Submitted on Sep 12th 2026 at 07:59:57 UTC by @mussucal for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92779
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

### Description

### Brief/Intro

When the Manager submits a standalone-HCA action (registration, renewal, record edits) it builds an intent and hands it to the Rhinestone SDK, which asks the orchestrator to turn the app's requested calls into a signed-and-fillable intentOp. The wallet then signs that intentOp.

There are two ways the wallet can sign it:

Session-signed — the app attaches a scoped session with verifyExecutions: true. The SDK produces an emissary signature that the on-chain validator checks against a hardcoded ENS-only policy (verifyExecution → \_checkRegistrationPolicy). A substituted or out-of-policy execution reverts. Owner-signed — the app attaches no session. The SDK signs the orchestrator's returned intentOp as a plain owner ECDSA/ERC-1271 signature, with no execution verification, and the on-chain validator's owner branch validates that signature with no execution policy at all.

The owner-signed path is reachable: the app submits owner-signed whenever there is no active session, or when its own owner-divergence guard drops the session. And on that path there is no defense on either side — the client never compares the orchestrator's returned executions to what it asked for, and the on-chain validator never restricts them.

So a malicious or compromised orchestrator — the very party that returns the intentOp — can return an intentOp whose executions differ from the calls the app requested (a transfer of the HCA's assets, an added account owner, a token approval to an attacker). On the owner-signed path the wallet's owner signature authorizes that substituted intentOp, and the owner-validation branch does not impose the execution policy applied to session signatures. The user approved a registration; the chain executes the substituted executions.

The defended session path proves the developers know execution integrity must be enforced. The owner path — reachable, and the fallback the app deliberately routes to — has none of it.

### Vulnerability Details

The finding is one chain across three layers. Each link is quoted from its source.

1. The owner-signed path is reachable (in-scope app)

apps/manager/src/lib/smart-account/SmartAccountContext.tsx, buildRhinestoneSigner (≈:148–171):

```ts
const attachSession = !!sessionContext && !!sessionOwnerAddress
if (sessionContext && !sessionOwnerAddress) {
  logger.warn('Wallet owner mismatch; refusing to attach session to signer')  // WEB-287
}
return {
  account: params.baseClient as unknown as RhinestoneSigner['account'],
  config: …,
  …(attachSession ? { session: sessionContext } : {}),   // no session ⇒ owner-signed
}
```

The signer carries a session only when a session context exists and the owner is verified. It is built without a session — i.e. owner-signed — in ordinary states: before a session has been established, after one expires or is evicted, and whenever the WEB-287 guard fires because the connected wallet diverged from the HCA owner mid-flight. In every one of those states the next HCA intent is submitted owner-signed.

2. On the owner path the app sends no execution check (in-scope app)

packages/transaction-manager/src/actors/warp-transport.actor.ts:114–121, 163:

```ts
const sessionSigners = signer.session
  ? ({ type: 'experimental_session', session: signer.session.session,
       …(sessionEnableData ? { enableData: sessionEnableData } : {}),
       verifyExecutions: true } satisfies NonNullable<Transaction['signers']>)
  : undefined
…
const sdkParams = {
  … calls: [...calls], sponsored: UNSPONSORED, feeAsset: …,
  …(sessionSigners ? { signers: sessionSigners } : {}),   // owner path: NO signers
} satisfies Transaction
const transaction = await account.sendTransaction(sdkParams)
```

verifyExecutions: true is set only when a session exists. With no session, no signers object is sent, so verifyExecutions is never requested. The app calls the SDK's atomic sendTransaction (prepare + sign + submit in one call) and never receives the orchestrator's intentOp to inspect before it is signed.

3. The SDK signs the orchestrator's returned executions, with no comparison (dependency, @rhinestone/sdk\@1.8.0)

dist/src/execution/utils.js, prepareTransactionAsIntent — the app's calls are sent to the orchestrator and the orchestrator's route is returned unchecked:

```js
const metaIntent = { …, destinationExecutions: calls, … }
const orchestrator = getOrchestrator(…, config.endpointUrl, …)
const intentRoute = await orchestrator.getIntentRoute(metaIntent)
return { intentRoute, intentInput: serializedIntent }
```

// nothing compares intentRoute.intentOp's executions to `calls`

dist/src/execution/index.js, sendTransactionAsIntent — the returned op is what gets signed and submitted:

```js
const { intentRoute, intentInput } = prepared
const { originSignatures, destinationSignature } =
    await signIntent(config, intentRoute.intentOp, targetChain, signers)      // ← returned op
…
return await submitIntentInternal(config, …, intentRoute.intentOp, originSignatures, …)
```

signIntent builds the digest from the orchestrator's op (getIntentMessages(config, intentOp)), and signIntentTypedData on the owner path (no session signer) takes the plain branch — every verifyExecutions branch is guarded by isResolvedSessionSignerSet(signers), which is false without a session:

```js
const hash = hashTypedData(parameters)                 // parameters derived from the returned intentOp
if (isResolvedSessionSignerSet(signers) && signers.verifyExecutions) { … emissary … }   // session only
if (isResolvedSessionSignerSet(signers)) { … }                                          // session only
return await getEip1271Signature(config, signers, chain, { address: validator.address, isRoot }, hash)
```

So on the owner path the wallet produces a plain ERC-1271 signature over a digest computed from the orchestrator's returned executions, and nothing in the client compares those executions to the calls the app asked for. verifyExecutions — the one mechanism that binds executions to the on-chain policy check — is structurally unavailable without a session.

4. The on-chain validator's owner branch enforces no execution policy (verified deployed contract)

HCAOwnerAndSessionValidator, verified on Sepolia at 0x5f249FCa8bB4949105651146858c347E8BFb0F7E. The ERC-1271 entry the intent executor calls:

```solidity
function isValidSignatureWithSender(address sender, bytes32 hash, bytes calldata data)
    external view returns (bytes4 magicValue)
{
    if (sender != INTENT_EXECUTOR) revert CallerNotIntentExecutor();
    if (data.length == ECDSA_SIGNATURE_LENGTH) {           // 65-byte owner signature
        (address expectedOwner, ) = _ownerAndSessionNonce(msg.sender);
        if (_recover(hash, data) != expectedOwner) revert InvalidSigner();
        // returns the magic value — NO _checkRegistrationPolicy
    } else {
        _validateFixedSession(hash, data);                 // session path → policy
    }
    return ERC1271_MAGICVALUE;
}
```

The owner branch recovers the owner, checks it, and returns success — it never calls \_checkRegistrationPolicy. validateUserOp (the ERC-4337 entry) is the same: owner signature over the digest, no execution inspection. The session branches, by contrast, funnel through \_validateFixedSession → \_checkRegistrationPolicy → \_checkRegistrationExecutions, a hardcoded allow-list (registrar commit/register with the registrant pinned to the owner, resolver setters, approve to the registrar, value == 0 on every call) that reverts ActionNotAllowed / PolicyRuleFailed on anything else.

The chain

A malicious/compromised orchestrator returns an intentOp whose executions differ from the requested calls → the app (owner path) submits it via sendTransaction with no verifyExecutions and never inspects it → the SDK signs the returned op with the wallet as a plain ERC-1271 signature → the on-chain owner branch validates that signature without the execution policy applied to session signatures. Accordingly, once the orchestrator substitutes the executions, the owner-signed path provides no demonstrated client-side or on-chain execution-equivalence check before the substituted operation is authorized.

### Impact Details

On the owner-signed path the wallet's owner signature authorizes whatever executions the substituted intentOp contains — a transfer of the HCA's assets, an added or replaced account owner, a blanket token approval, a name registered or transferred to an attacker — and the owner-validation branch imposes none of the execution policy that constrains session signatures. The user believes they approved a registration; the chain executes the substituted executions. There is accordingly no demonstrated client-side or on-chain execution-equivalence check before the substituted operation is authorized.

This matches the Critical row "Malicious interactions with an already-connected wallet — Modifying transaction arguments or parameters; Substituting contract addresses; Submitting malicious transactions," and, where the substituted action adds an owner, "Unauthorized takeover of a user's smart account."

The asymmetry is the finding

The session path is defended twice — the client sets verifyExecutions: true, and the on-chain validator enforces the ENS-only policy. The owner path is defended neither — no client comparison, no on-chain policy — and it is the path the app deliberately falls back to (WEB-287) and the state every account is in before its first session. The developers built the on-chain policy and the verifyExecutions binding precisely because execution integrity matters; the gap is that both are bypassed on a reachable path.

Threat model, stated honestly

The substitution vector is the orchestrator — the party that returns the intentOp. The finding is that the app extends it complete, unchecked trust on the owner-signed path: nothing on the client and nothing on-chain constrains what a malicious or compromised orchestrator's returned intent may execute. Whether the orchestrator is treated as inside the threat model is the panel's to weigh; we note only that the competition's Critical examples name exactly this class of substitution ("substituting contract addresses," "modifying transaction arguments"), that the app has a working defense on the session path and none on the owner path, and that the owner path is reachable in ordinary operation. We present the asymmetry and leave the severity call to the panel; we believe Critical is correct because the owner path has no backstop of any kind.

### PoC descriptions

What is proven, and how

Stated plainly so the evidence is read for exactly what it is:

Executable (in-scope app): the attached test drives the real submitWarpTransaction and asserts that, given a session-less signer, the params handed to the SDK carry no signers / no verifyExecutions, while a session-bearing signer yields verifyExecutions: true. This proves the app-side link — the owner path requests no execution verification — by execution, not by reading. The app's own WARP debug log corroborates it, printing the SDK params both ways:

## owner path (no session):

\[WARP] SDK sendTransaction params: {"sourceChains":…,"calls":"Array(1)","sponsored":"object","feeAsset":"USDC","tokenRequests":"Array(0)"}

## session path:

\[WARP] SDK sendTransaction params: {…,"tokenRequests":"Array(0)","signers":"object"}

The signers key — which is the only carrier of verifyExecutions — is absent on the owner path and present on the session path, in the application's own instrumentation.

Source inspection (dependency): @rhinestone/sdk\@1.8.0 (execution/index.js, execution/utils.js) shows the owner-path signature is computed from the orchestrator's returned intentOp with no comparison to the requested calls, and that verifyExecutions is session-only. This is a dependency, not in-scope code, but it is the pinned version the app ships (pnpm-workspace.yaml catalog: @rhinestone/sdk: 1.8.0) and its behaviour is quoted above. Verified deployed contract: HCAOwnerAndSessionValidator at 0x5f249FCa8bB4949105651146858c347E8BFb0F7E is verified on Sepolia Etherscan; the owner branch applies no execution policy, quoted above. This is authoritative — it is the contract the live system uses. Not executed as a live exploit. Demonstrating the fill end-to-end requires an orchestrator that returns a substituted intentOp (a malicious/compromised orchestrator), which we cannot and should not stand up against the production service; the repo's local mock orchestrator cannot price or fill standalone-HCA intents at all. The three sources together establish the outcome without it: the app requests no verification (executed), the SDK signs the returned executions unchecked (source), and the validator enforces no policy on the owner signature (verified deployment). There is no link in the chain left to assumption. Attached PoC file File - ownerPathNoVerifyExecutions.audit.test.ts Repo path - packages/transaction-manager/src/actors/ What it proves - Drives the real submitWarpTransaction; asserts the owner path sends the SDK no signers/verifyExecutions, and the session path sends verifyExecutions: true

Run it:

```bash
pnpm --filter @ens-apps/transaction-manager exec vitest run \
  src/actors/ownerPathNoVerifyExecutions.audit.test.ts
```

Expected: Tests 2 passed. WARP-OWNER-1 (owner path → no signers) and WARP-OWNER-2 (session path → verifyExecutions: true) are the two halves; the passes are the observed behaviour.

### References

In-scope app

apps/manager/src/lib/smart-account/SmartAccountContext.tsx — buildRhinestoneSigner (≈:148–171): session attached only when owner is verified; owner-signed fallback (WEB-287) packages/transaction-manager/src/actors/warp-transport.actor.ts:114–121, 163 — verifyExecutions: true on the session path only; no signers on the owner path; atomic account.sendTransaction

Dependency (pinned @rhinestone/sdk\@1.8.0)

dist/src/execution/index.js — sendTransactionAsIntent: signs and submits intentRoute.intentOp (the orchestrator's returned op) dist/src/execution/utils.js — prepareTransactionAsIntent (no compare of returned op to calls), signIntent / getIntentMessages (digest from the returned op), signIntentTypedData (verifyExecutions gated on a session signer) pnpm-workspace.yaml — @rhinestone/sdk: 1.8.0

Verified deployed contract

HCAOwnerAndSessionValidator, Sepolia 0x5f249FCa8bB4949105651146858c347E8BFb0F7E — isValidSignatureWithSender (65-byte owner branch: no policy), validateUserOp (no policy), vs. \_validateFixedSession → \_checkRegistrationPolicy (session path: hardcoded ENS-only allow-list)

Standards

ERC-1271 — contract signature validation: <https://eips.ethereum.org/EIPS/eip-1271> ERC-7579 — modular smart-account validators: <https://eips.ethereum.org/EIPS/eip-7579> ERC-4337 — account abstraction: <https://eips.ethereum.org/EIPS/eip-4337>

### Proof of Concept

### Proof of Concept

#### packages/transaction-manager/src/actors/ownerPathNoVerifyExecutions.audit.test.ts

```ts
/**
 * Audit anchor — the OWNER-signed HCA intent path requests NO execution
 * verification from the SDK, while the session path requests it.
 *
 * This drives the REAL `submitWarpTransaction` and captures the params handed to
 * the SDK's `account.sendTransaction`. It is the executable half of the
 * owner-signed substitution finding:
 *
 *   - With no session, the app tells the SDK to owner-sign with NO
 *     `verifyExecutions` (no `signers` object at all). The SDK then signs the
 *     ORCHESTRATOR's returned intentOp (see @rhinestone/sdk@1.8.0
 *     execution/utils.js `signIntent`/`signIntentTypedData`), and the on-chain
 *     owner branch of HCAOwnerAndSessionValidator applies NO execution policy —
 *     so a substituted intent fills with full account authority.
 *   - With a session, the app sets `verifyExecutions: true`, which is the one
 *     mechanism that binds executions to the on-chain policy check.
 *
 * These assert OBSERVED behaviour, so the PASSES are the finding.
 *
 *   pnpm --filter @ens-apps/transaction-manager exec vitest run \
 *     src/actors/ownerPathNoVerifyExecutions.audit.test.ts
 */

import { sepolia } from 'viem/chains'
import { describe, expect, it } from 'vitest'
import { submitWarpTransaction } from './warp-transport.actor'

// A minimal RhinestoneAccount stand-in: captures the SDK params, and satisfies
// the actor's runtime guard (`typeof account.waitForExecution === 'function'`).
function makeAccount(captured: unknown[]) {
  return {
    getAddress: () => '0x1111111111111111111111111111111111111111',
    sendTransaction: async (params: unknown) => {
      captured.push(params)
      return { id: 1n }
    },
    waitForExecution: async () => ({ fill: { hash: '0xdeadbeef' } }),
  }
}

// A normal HCA intent: one call, no sessionEnableData (owner path is valid).
// biome-ignore lint/suspicious/noExplicitAny: audit harness constructs minimal shapes
const request = {
  type: 'rhinestone-intent',
  from: '0x2222222222222222222222222222222222222222',
  chainId: sepolia.id,
  rhinestoneParams: {
    calls: [
      { to: '0x3333333333333333333333333333333333333333', data: '0x', value: 0n },
    ],
  },
} as any

describe('owner-signed HCA intents request no execution verification', () => {
  it('WARP-OWNER-1: with NO session, the SDK params carry no signers / no verifyExecutions', async () => {
    const captured: unknown[] = []
    // Signer WITHOUT `.session` → the owner-signed path.
    // biome-ignore lint/suspicious/noExplicitAny: audit harness
    const signer = { account: makeAccount(captured), config: { chain: sepolia } } as any

    const res = await submitWarpTransaction({ request, signer })

    expect(res.isOk()).toBe(true)
    expect(captured).toHaveLength(1)
    // The app sends the SDK NO `signers` object at all — so the SDK owner-signs
    // and no `verifyExecutions` is ever requested.
    // biome-ignore lint/suspicious/noExplicitAny: audit harness
    expect('signers' in (captured[0] as any)).toBe(false)
  })

  it('WARP-OWNER-2 (control): WITH a session, the SDK params carry verifyExecutions:true', async () => {
    const captured: unknown[] = []
    // biome-ignore lint/suspicious/noExplicitAny: audit harness
    const signer = {
      account: makeAccount(captured),
      config: { chain: sepolia },
      session: { session: { id: 'x' } },
    } as any

    const res = await submitWarpTransaction({ request, signer })

    expect(res.isOk()).toBe(true)
    // biome-ignore lint/suspicious/noExplicitAny: audit harness
    const params = captured[0] as any
    expect(params.signers?.type).toBe('experimental_session')
    expect(params.signers?.verifyExecutions).toBe(true)
  })
})
```
