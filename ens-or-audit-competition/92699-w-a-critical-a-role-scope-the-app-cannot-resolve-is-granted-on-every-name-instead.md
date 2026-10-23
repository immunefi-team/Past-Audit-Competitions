# 92699 w a critical a role scope the app cannot resolve is granted on every name instead

**Submitted on Sep 11th 2026 at 17:04:48 UTC by @nomorecaffeine for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92699
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

### Bug Description

`PermissionedResolver` keys roles by `resource(node, part)`. A name-scoped role uses `part = 0`, while `authorizeTextRoles` and `authorizeDataRoles` grant at `resource(node, partHash(key))`. The Explorer rebuilds that mapping client side, and can represent only the first kind:

[resolverRoles.ts#L147-L160](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/lib/roles/resolverRoles.ts#L147-L160)

```ts
export const buildResourceToNameMap = (
  nodes: readonly { readonly name: string }[],
): Map<string, string> => {
  const map = new Map<string, string>()
  map.set(ROOT_RESOURCE, '(root)')

  for (const node of nodes) {
    if (node.name) {
      map.set(computeNameResource(node.name).toLowerCase(), node.name)
    }
  }

  return map
}
```

`computeNameResource` is `keccak256(namehash(name) || bytes32(0))` (`:138-141`), so a part-scoped resource can never be a key and `resolvedNames` is left empty. The table renders nothing in the Name cell (`ResolverRolesTable.tsx:69-70`) and the sheet suppresses its scope line (`ResolverRolesSidebar.tsx:391`), so the row looks like any other.

The scope of the resulting write comes from one expression:

[ResolverRolesSidebar.tsx#L167-L169](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L167-L169)

```ts
  const resolvedNames = row?.original.resolvedNames ?? []

  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
```

`''` is also the value the app uses for "the admin deliberately chose root", and both write helpers branch on it:

[grantResolverRoles.ts#L45-L53](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts#L45-L53)

```ts
  const writeParams = grantResolverRolesWriteParameters(
    walletClient as Parameters<typeof grantResolverRolesWriteParameters>[0],
    name === ''
      ? {
          resolverAddress,
          targetAccount: account,
          scope: 'root',
          roles,
        }
```

`revokeResolverRoles.ts:45-53` is the same branch. An unresolvable scope and a deliberate root choice are indistinguishable by the time the transaction is built.

A third party places the row. In `ensdomains/contracts-v2` at `48b3e2d`, `authorizeTextRoles` and `authorizeDataRoles` check grant authority at the **name** resource and land the grant on the **part** resource for an **arbitrary** account:

* Bob runs a resolver deployed through the Explorer, which grants him every admin role at root.
* Bob delegates `ROLE_SET_TEXT_ADMIN` on `blog.bob.eth` to Alice, so she can manage that one name's text records.
* Alice calls `authorizeTextRoles(blog.bob.eth, "url", Eve, true)`. Her authority is checked at `blog.bob.eth`, which she holds, and the grant lands at `resource(namehash(blog.bob.eth), partHash("url"))` for Eve, an address she controls.
* Bob opens `/resolver/<address>/roles`. Eve's row shows an empty Name cell and no scope line. Nothing on the page or in the transaction modal says "root".
* Bob ticks "Set Address" for Eve, intending to let her set that one name's address record.
* The sheet emits `grantRootRoles(ROLE_SET_ADDR, Eve)`, not `authorizeNameRoles`. `_effectiveRoles` ORs root roles into every resource check, so Eve now holds `ROLE_SET_ADDR` on every name the resolver serves.
* Eve calls `setAddr(namehash("bob.eth"), 60, Eve)`. `resolve()` returns her address, and the Explorer displays it.
* Bob reopens the sheet and clicks "Remove user" under a modal reading "This action cannot be undone." It emits `revokeRootRoles`, mines with status 1, and Eve keeps every role.

Two adjacent defects share the cause. `ROLE_SET_DATA` (bit 36) is missing from the role table (`resolverRoles.ts:24-45`), so holders of it decode to no roles and `handleRemoveUser` early-returns (`ResolverRolesSidebar.tsx:307-313`). And `ROLE_SET_ALIAS` is root-only on chain, so granting it from a resolved row confers nothing while granting it from a blank row hands over alias control of the whole resolver.

### Impact

An administrator who ticks one permission on a row the interface presents as ordinary grants that permission across every name the resolver serves. The grantee is chosen by a delegate whose own authority covers one record key on one name, so this is an escalation from a single text key to resolver-wide control, and it needs no more than the one click the admin already believed they were making.

The grantee can then set the address record of any name on that resolver, which is what `resolve()` returns and what the Explorer displays, so funds sent to any of those names reach them. Cleanup does not work: the admin's own remove action reports success and revokes nothing, and for an alias grant the affected name's owner cannot reverse it at all.

This state exists today. 69 role rows render with a blank Name cell across 43 of 6,209 live `PermissionedResolver`s, 62 of them with a role the sheet can escalate.

### Attacker prerequisites

Alice needs a record-level admin role on one name the resolver serves, which is exactly the role an owner delegates to let somebody manage a single record. She cannot grant herself anything at root and she cannot force the click: Bob has to tick a permission that was not already set. What she controls is that his row shows no scope for him to check, and that her planted row is indistinguishable from a name-scoped one.

### Recommendation

Stop encoding "unresolved" as "root", so an unrepresentable row cannot be written at all:

[ResolverRolesSidebar.tsx#L169](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L169)

```diff
-  const roleName = resolvedNames.find((n) => n !== '(root)') ?? ''
+  const roleName = resolvedNames.includes('(root)')
+    ? ''
+    : resolvedNames.find((n) => n !== '(root)')
+  const scopeUnresolved = roleName === undefined
```

Then gate `handleSaveChanges` (`:295-305`) and `handleRemoveUser` (`:307-313`) on `!scopeUnresolved`, so the row is read-only rather than silently root-scoped. The scope can also be resolved properly: the indexer exposes `ResolverDetail.namedResources`, which names part-scoped resources, and `useResolverOverview.ts:67-109` never requests it.

## Link to Proof of Concept

<https://gist.github.com/nguyen-xuan-quoc/69d369328bc2fccb27e816a07d334cf0>

## Proof of Concept

### Proof of Concept

Three tests, 25 assertions, run against a pinned Sepolia fork. They import the shipped portal modules (`buildResourceToNameMap`, `computeNameResource`, `groupRolesByAccount`, `prepareGrantResolverRolesTransaction`, `prepareRevokeResolverRolesTransaction`) rather than reimplementing them, so what is measured is the app's own behaviour. No module the finding depends on is mocked, and no private keys are used: the tests call `anvil_impersonateAccount`, so real Sepolia accounts are impersonated locally and never signed for.

<https://gist.github.com/nguyen-xuan-quoc/69d369328bc2fccb27e816a07d334cf0>

#### What the tests do

The first test runs against resolver `0x0f98c60f734b363cab6e2b64c74fedc7d9075baf`, live on Sepolia, using its real root admin and a real part-scoped grantee read off chain:

1. **Control.** The grantee calls `setAddr` on `carol.dvod-test.eth`. It must revert. Without this failing first, step 4 proves nothing.
2. **Render.** The grantee's real on-chain role row is passed through the shipped grouping. `resolvedNames` comes back empty, so the Name cell and the scope line render nothing. The admin's own root row is passed through the same code as a positive control and still resolves to `(root)`, which shows the empty result is specific to this row and not a broken harness.
3. **Sentinel.** The empty name flows into the shipped grant helper. The emitted selector is `grantRootRoles`, not `authorizeNameRoles`. Both selectors are computed inside the test rather than asserted from memory.
4. **Capability.** The root admin submits the Explorer's own calldata. The call that reverted in step 1 now succeeds, on three names including the parent.
5. **Revoke.** "Remove user" emits `revokeRootRoles`, mines with status 1, and the grantee still holds every part-scoped role.

#### Steps to run

1. `pnpm install --frozen-lockfile`
2. Start a fork in its own terminal:

```sh
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com \
      --fork-block-number 11681304 --port 8545 --silent
```

3. `cp 01-resolverRolesRootScope.test.ts apps/portal/src/features/resolver/resolverRolesRootScope.test.ts`
4. Run it:

```sh
ANVIL_RPC=http://127.0.0.1:8545 pnpm --filter portal exec vitest run \
  src/features/resolver/resolverRolesRootScope.test.ts \
  --no-coverage --reporter=verbose --disableConsoleIntercept
```

`--disableConsoleIntercept` is required or the suite passes and prints nothing.

#### Output

```
[LEG1] hasRootRoles(ROLE_SET_ADDR, V) before: false
[LEG1] setAddr reverted: true
[LEG1] revert names EACUnauthorizedAccountRoles: true
[LEG2] V.resolvedNames: []
[LEG2] V.decodedRoles: ["ROLE_SET_TEXT","ROLE_SET_TEXT"]
[LEG2] admin.resolvedNames (positive control): ["(root)"]
[LEG3] roleName used as scope: ""
[LEG3] emitted selector (empty name): 0x072d5d77 => grantRootRoles? true
[LEG3] name-scoped selector: 0xbbd9abb5 => authorizeNameRoles? true
[LEG3] grantRootRoles selector: 0x072d5d77
[LEG3] authorizeNameRoles selector: 0xbbd9abb5
[LEG4] grant tx status: success
[LEG4] hasRootRoles(ROLE_SET_ADDR, V) after grant: true
 ✓ src/features/resolver/resolverRolesRootScope.test.ts > resolver roles root-scope confusion > LEG 1 CONTROL: grantee V cannot setAddr on carol.dvod-test.eth (must revert) 16ms
 ✓ src/features/resolver/resolverRolesRootScope.test.ts > resolver roles root-scope confusion > LEG 2 RENDER: shipped map+group over real rows => V.resolvedNames is empty 3ms
 ✓ src/features/resolver/resolverRolesRootScope.test.ts > resolver roles root-scope confusion > LEG 3 SENTINEL: empty name flows to scope root => grantRootRoles, not authorizeNameRoles 2ms
[LEG4] V setAddr carol.dvod-test.eth: status success, addr now 0x00000000000000000000000000000000000000ff
[LEG4] V setAddr dvod-test.eth (PARENT): status success, addr now 0x00000000000000000000000000000000000000ff
[LEG4] V setAddr bob.dvod-test.eth: status success, addr now 0x00000000000000000000000000000000000000ff
[LEG5] emitted revoke selector: 0xce156e82 => revokeRootRoles? true
[LEG5] revokeRootRoles tx status: success
[LEG5] V still holds part-scoped ROLE_SET_TEXT after "Remove user": true
[LEG5] V still holds part-scoped ROLE_SET_DATA after "Remove user": true
```

#### The other two tests

Repeat steps 3 and 4 with `03-resolutionTerminal.test.ts` and `05-aliasAuthorityChain.test.ts`, dropping the numeric prefix on copy the same way. **Restart the fork between files**, since each deploys its own resolver from the same funded accounts and a reused fork collides on addresses.

`03` carries the impact end to end, on a resolver deployed through the portal's own `getResolverInitCalldata`. It plants the row as a name-level text admin, ticks "Set Address" on the blank row, and lands at resolution (`Tests 8 passed`):

```
[RF6] hasRootRoles(ROLE_SET_ADDR, E2) before: false
[RF6] setAddr reverted: true
[RF6] revert names EACUnauthorizedAccountRoles: true
[RF7] grant tx status: success
[RF7] hasRootRoles(ROLE_SET_ADDR, E2) after: true
[RF8] E2 setAddr victim.eth: status success
[RF8] E2 setAddr treasury.eth: status success
[RF8] resolve(victim.eth).addr(60) = 0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000001400000000000000000000000000000000000000ee000000000000000000000000
```

`05` shows the alias permission cannot be delegated correctly and cannot be undone by the name's own admin (`Tests 12 passed`):

```
[LEG9] victim name admin deleteAlias reverted: true
[LEG9] revert names EACUnauthorizedAccountRoles: true
[LEG10] name-scoped ROLE_SET_ALIAS grant status: success
[LEG10] W holds name-scoped ROLE_SET_ALIAS on victim.eth: true
[LEG10] yet W setAlias still reverts (setAlias is onlyRootRoles): true
```
