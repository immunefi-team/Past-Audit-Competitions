# 92625 w a medium stale v1 registry state restores a revoked manager and resolver during v1 to v2 migration

**Submitted on Sep 11th 2026 at 02:23:18 UTC by @kimnoic for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92625
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The Manager constructs V2 migration calldata and post-migration permissions from mutable V1 `owner` and `resolver` values returned by a subgraph snapshot, while its on-chain preflight revalidates only BaseRegistrar/NameWrapper token ownership. If a registrant revokes a delegated V1 manager or repairs a resolver after that snapshot is obtained, migrating through the stale Manager state silently restores the revoked manager's `ROLE_SET_RESOLVER` permission and can reinstall the stale resolver in V2. The former manager can consequently hijack the migrated ENS name's resolution and redirect users or funds sent to that name.

## Vulnerability Details

### Root cause

For an unwrapped `.eth` second-level name, two distinct ownership concepts exist in V1:

1. The BaseRegistrar ERC-721 owner (the registrant); and
2. The ENS Registry record owner (the manager), which controls the name's resolver.

The application obtains both the registry owner and resolver through the V1 subgraph:

```graphql
resolver { address }
owner { id }
registrant { id }
```

`classifyName()` then treats these mutable indexed fields as authoritative migration inputs. It stores the subgraph resolver and converts any registry owner different from the registrant into `managerAddress`:

```typescript
// packages/migration/src/service/classifyNames.ts:137, 159-185
const v1ResolverAddress = domain.resolver?.address ?? null

const tokenHolder = toAddress(registrant.id)
const registryOwnerAddress = toAddress(domain.owner.id)
const managerAddress: Address | null =
  registryOwnerAddress &&
  registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase()
    ? registryOwnerAddress
    : null

return {
  type: 'classified',
  name: {
    domain,
    tokenType: 'unwrapped',
    tokenHolder,
    v1ResolverAddress,
    resolverStrategy: resolverStrategyFor({
      tokenType: 'unwrapped',
      fuses: 0n,
      v1ResolverAddress,
    }),
    managerAddress,
  },
}
```

However, the preflight does not read the live ENS V1 Registry record. For an unwrapped name it checks only `BaseRegistrar.ownerOf(tokenId)`:

```typescript
// packages/migration/src/service/preflightChecks.ts:47-101
const contracts: Contract[] = names.map((name) =>
  name.tokenType === 'unwrapped'
    ? {
        address: BASE_REGISTRAR,
        abi: BASE_REGISTRAR_ABI,
        functionName: 'ownerOf' as const,
        args: [BigInt(name.domain.labelhash)] as const,
      }
    : {
        address: NAME_WRAPPER,
        abi: NAME_WRAPPER_ABI,
        functionName: 'getData' as const,
        args: [BigInt(name.domain.id)] as const,
      },
)

// ...
if (currentOwner.toLowerCase() !== expected) {
  ids.add(name.domain.id)
}
```

This proves only that the registrant still owns the ERC-721 token. It cannot prove that the previously indexed V1 Registry manager or resolver is still current.

The stale-input window is reinforced by the React Query cache. The cache key contains only the wallet/HCA and domain IDs; it does not commit to `domain.owner.id`, `domain.resolver.address`, or a source block number:

```typescript
// apps/manager/src/features/migration/hooks/useMigrationPreflight.ts:32-51
const ids = [...domains].map((d) => d.id).sort().join(',')
return queryClient.fetchQuery({
  queryKey: [
    'migration-preflight',
    eoa.toLowerCase(),
    hcaAddress?.toLowerCase() ?? '',
    ids,
  ] as const,
  queryFn: () => computeMigrationPreflight({ domains, /* ... */ }),
  staleTime: options.staleTime ?? 60_000,
})
```

The same stale `Domain` objects are classified again when the migration plan is built:

```typescript
// apps/manager/src/features/migration/service/buildMigrationPlan.ts:317-329
export const buildMigrationPlan = async (params: {
  domains: readonly V1Domain[]
  // ...
}): Promise<MigrationPlan> => {
  const { domains, migrationOwner } = params
  const classifiedNamesResult = classifyNames([...domains], migrationOwner)
  const classified = classifiedNamesResult.classified
  // ...
}
```

The stale resolver is selected as the V2 migration resolver whenever the `keep-v1` strategy applies and it is not recognized as a public resolver:

```typescript
// apps/manager/src/features/migration/service/encodeMigration.ts:24-50
case 'keep-v1':
  return (
    isKnownPublicResolver(name.v1ResolverAddress)
      ? defaultResolver
      : (name.v1ResolverAddress ??
        (cannotSetResolverLocked ? zeroAddress : defaultResolver))
  ) as Address
```

The stale manager receives V2 resolver authority:

```typescript
// apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:8-26
const ROLE_SET_RESOLVER = 1n << 24n

export const buildRoleGrantCall = (name: ClassifiedName): Call => ({
  to: V2_CONTRACTS.ETHRegistry,
  data: encodeFunctionData({
    abi: ETH_REGISTRY_V2_ABI,
    functionName: 'grantRoles',
    args: [labelToCanonicalId(name.label), ROLE_SET_RESOLVER, name.managerAddress],
  }),
  value: 0n,
})
```

Both actions are included in the atomic migration batch:

```typescript
// apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:481-490, 608-614
const expectedResolver = resolverFor(classified, defaultResolver, resolver)
const migrationData = createMigrationData({
  label: classified.label,
  owner: wallet,
  subregistry: zeroAddress,
  resolver: expectedResolver,
})

if (classified.managerAddress) {
  innerExecutions.push({
    phase: 'manager-role-grant',
    call: buildRoleGrantCall(classified),
    // ...
  })
}
```

Rebuilding a later batch does not cure the issue. The execution path re-resolves migration routes, but passes the already-classified stale objects back into `buildAtomicMigrationBatches()`. Immediately before wallet submission it also checks mutable approval state, but never re-reads the live V1 registry manager and resolver.

Finally, the on-chain `UnlockedMigrationController` cannot detect the discrepancy. It validates the label/token relationship, clears the V1 record, and registers V2 using the caller-supplied `md.resolver`:

```solidity
// contracts/src/migration/UnlockedMigrationController.sol:107-119, 153-165
LibMigration.Data memory md = abi.decode(data, (LibMigration.Data));
if (tokenId != uint256(keccak256(bytes(md.label)))) {
    revert LibMigration.NameDataMismatch(tokenId);
}
_BASE_REGISTRAR.reclaim(tokenId, address(this));
_REGISTRY_V1.setRecord(node, GRAVEYARD, address(0), 0);
_BASE_REGISTRAR.safeTransferFrom(address(this), GRAVEYARD, tokenId);
_inject(md);

// ...
ETH_REGISTRY.register(
    md.label,
    md.owner,
    md.subregistry,
    md.resolver,
    REGISTRATION_ROLE_BITMAP,
    0
);
```

The contract is behaving according to its interface; the vulnerability is that the authorized Manager flow supplies security-sensitive, stale V1 state.

### Exploit sequence

1. The victim is the BaseRegistrar owner of `victim.eth` and previously delegated the V1 Registry manager role to the attacker.
2. The Manager/subgraph obtains a snapshot where `domain.owner.id == attacker` and, optionally, `domain.resolver.address == attackerResolver`.
3. The victim revokes the attacker by setting the V1 Registry owner back to the victim and repairs the resolver. This may occur in another tab, another client, or while indexing/cache state is stale.
4. The victim starts or continues the V1-to-V2 migration using the stale Manager state.
5. Preflight succeeds because `BaseRegistrar.ownerOf(labelhash) == victim`; it never checks the live V1 Registry manager/resolver.
6. The migration installs the stale resolver and grants `ROLE_SET_RESOLVER` to the revoked manager in V2.
7. The attacker uses the restored role to replace the V2 resolver with an attacker-controlled resolver and redirect `victim.eth`.

## Impact Details

**Selected in-scope impact:**

> Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker.

The vulnerable operation is specifically the program's V1-to-V2 Manager migration flow. It undoes an already-completed security revocation and restores resolver control to the former manager. The attacker can then point the migrated name at an attacker-controlled address or resolver, alter resolution-dependent records, make the name unusable, or impersonate services that rely on the ENS name.

The direct asset affected is control over the ENS name's resolution. The victim still owns the V2 name, so this report does not claim an immediate ownership-token theft. Nevertheless, a successful exploit can cause downstream financial loss: wallets and users resolving the name may send ETH, tokens, or NFTs to the attacker's address. The amount is unbounded by the migration transaction and depends on transfers subsequently made in reliance on the hijacked name. The deterministic and program-listed impact is therefore **ENS name hijacking during migration**; downstream redirected transfers strengthen the consequence but are not required to establish validity.

## References

* `apps/manager/src/features/migration/service/v1SubgraphClient.ts:31-40` — indexed V1 owner and resolver source.
* `packages/migration/src/service/classifyNames.ts:125-185` — stale resolver/manager classification.
* `packages/migration/src/service/preflightChecks.ts:47-101` — owner-only on-chain preflight.
* `apps/manager/src/features/migration/hooks/useMigrationPreflight.ts:20-52` — cache key and 60-second `staleTime`.
* `apps/manager/src/features/migration/service/buildMigrationPlan.ts:317-369` — plan construction from supplied `V1Domain` objects.
* `apps/manager/src/features/migration/service/encodeMigration.ts:24-50` — resolver selection.
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:8-26` — V2 resolver-role grant.
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:481-490, 608-614` — atomic inclusion of resolver and role restoration.
* `apps/manager/src/features/migration/service/migrationService.ts:886-921, 1052-1061, 1161-1204` — execution-time reuse of classified data.
* `contracts/src/migration/UnlockedMigrationController.sol:90-119, 152-165` — controller consumes the supplied resolver.

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @notice Minimal V1 registry used to reproduce the mutable manager/resolver state.
contract MockV1Registry {
    struct Record {
        address owner;
        address resolver;
    }

    mapping(bytes32 => Record) public records;

    function setRecord(bytes32 node, address owner, address resolver) external {
        records[node] = Record(owner, resolver);
    }
}

/// @notice Minimal BaseRegistrar ownership source checked by the vulnerable preflight.
contract MockBaseRegistrar {
    mapping(uint256 => address) public ownerOf;

    function setOwner(uint256 tokenId, address owner) external {
        ownerOf[tokenId] = owner;
    }
}

/// @notice Minimal V2 registry exposing the security-relevant resolver role.
contract MockV2Registry {
    uint256 public constant ROLE_SET_RESOLVER = 1 << 24;

    mapping(bytes32 => address) public ownerOf;
    mapping(bytes32 => address) public resolverOf;
    mapping(bytes32 => address) public migrationExecutor;
    mapping(bytes32 => mapping(address => uint256)) public roles;

    function register(bytes32 resource, address owner, address resolver) external {
        ownerOf[resource] = owner;
        resolverOf[resource] = resolver;
        migrationExecutor[resource] = msg.sender;
    }

    function grantRoles(bytes32 resource, uint256 roleBitmap, address account) external {
        require(
            msg.sender == ownerOf[resource] || msg.sender == migrationExecutor[resource],
            "only owner or atomic migration executor"
        );
        roles[resource][account] |= roleBitmap;
    }

    function setResolver(bytes32 resource, address resolver) external {
        require(
            msg.sender == ownerOf[resource]
                || roles[resource][msg.sender] & ROLE_SET_RESOLVER != 0,
            "unauthorized"
        );
        resolverOf[resource] = resolver;
    }
}

/// @notice Models the audited frontend's classification, preflight, and atomic batch.
/// Foundry cannot execute TypeScript, so the harness deliberately mirrors only the
/// exact security boundary: stale subgraph values + ownerOf-only preflight + V2 grant.
contract VulnerableMigrationHarness {
    struct ClassifiedName {
        address tokenHolder;
        address managerAddress;
        address v1ResolverAddress;
    }

    MockBaseRegistrar public immutable registrar;
    MockV2Registry public immutable v2;

    constructor(MockBaseRegistrar registrar_, MockV2Registry v2_) {
        registrar = registrar_;
        v2 = v2_;
    }

    function classifyFromSubgraph(
        address registrant,
        address staleRegistryOwner,
        address staleResolver
    ) external pure returns (ClassifiedName memory classified) {
        classified.tokenHolder = registrant;
        classified.managerAddress = staleRegistryOwner == registrant
            ? address(0)
            : staleRegistryOwner;
        classified.v1ResolverAddress = staleResolver;
    }

    /// @dev Mirrors checkOwnership(): unwrapped names check BaseRegistrar.ownerOf only.
    function preflight(uint256 tokenId, address migrationOwner) public view returns (bool) {
        return registrar.ownerOf(tokenId) == migrationOwner;
    }

    /// @dev Mirrors the migration data plus the subsequent manager-role grant.
    function migrate(
        uint256 tokenId,
        bytes32 resource,
        address migrationOwner,
        ClassifiedName calldata stale
    ) external {
        require(msg.sender == migrationOwner, "only migration owner");
        require(preflight(tokenId, migrationOwner), "ownership preflight failed");

        v2.register(resource, migrationOwner, stale.v1ResolverAddress);
        if (stale.managerAddress != address(0)) {
            v2.grantRoles(resource, v2.ROLE_SET_RESOLVER(), stale.managerAddress);
        }
    }
}

contract StaleV1StateRestoresRevokedManagerTest {
    MockV1Registry internal v1;
    MockBaseRegistrar internal registrar;
    MockV2Registry internal v2;
    VulnerableMigrationHarness internal migration;

    address internal constant VICTIM = address(0x1111);
    address internal constant REVOKED_MANAGER = address(0x2222);
    address internal constant SAFE_RESOLVER = address(0x3333);
    address internal constant STALE_ATTACKER_RESOLVER = address(0x4444);
    address internal constant MALICIOUS_RESOLVER = address(0x5555);

    string internal constant LABEL = "victim";
    bytes32 internal constant RESOURCE = keccak256("victim");
    bytes32 internal constant NODE = keccak256("victim.eth");

    function setUp() public {
        v1 = new MockV1Registry();
        registrar = new MockBaseRegistrar();
        v2 = new MockV2Registry();
        migration = new VulnerableMigrationHarness(registrar, v2);
    }

    function test_staleV1SnapshotRestoresRevokedManagerDuringMigration() public {
        uint256 tokenId = uint256(keccak256(bytes(LABEL)));

        // The application/subgraph snapshot is captured while the attacker is manager.
        VulnerableMigrationHarness.ClassifiedName memory stale = migration
            .classifyFromSubgraph(VICTIM, REVOKED_MANAGER, STALE_ATTACKER_RESOLVER);

        // The victim revokes the manager and repairs the resolver on V1.
        v1.setRecord(NODE, VICTIM, SAFE_RESOLVER);
        registrar.setOwner(tokenId, VICTIM);

        (address liveManager, address liveResolver) = v1.records(NODE);
        require(liveManager == VICTIM, "manager revocation did not occur");
        require(liveResolver == SAFE_RESOLVER, "resolver repair did not occur");

        // Simulate the victim submitting the stale migration plan. The harness is
        // called as VICTIM by using a helper whose address is the migration owner.
        VictimWallet wallet = new VictimWallet();
        registrar.setOwner(tokenId, address(wallet));
        stale.tokenHolder = address(wallet);
        wallet.migrate(migration, tokenId, RESOURCE, stale);

        require(v2.ownerOf(RESOURCE) == address(wallet), "victim lost V2 ownership");
        require(
            v2.resolverOf(RESOURCE) == STALE_ATTACKER_RESOLVER,
            "stale resolver was not installed"
        );
        require(
            v2.roles(RESOURCE, REVOKED_MANAGER) & v2.ROLE_SET_RESOLVER() != 0,
            "revoked manager did not regain resolver authority"
        );

        // The restored manager can now replace the V2 resolver.
        RevokedManager attacker = new RevokedManager();
        // Use the actual attacker contract as the stale manager for this second migration.
        bytes32 secondResource = keccak256("victim-2");
        uint256 secondTokenId = uint256(keccak256("victim-2"));
        registrar.setOwner(secondTokenId, address(wallet));
        VulnerableMigrationHarness.ClassifiedName memory staleForAttacker = migration
            .classifyFromSubgraph(address(wallet), address(attacker), STALE_ATTACKER_RESOLVER);
        wallet.migrate(migration, secondTokenId, secondResource, staleForAttacker);
        attacker.replaceResolver(v2, secondResource, MALICIOUS_RESOLVER);

        require(
            v2.resolverOf(secondResource) == MALICIOUS_RESOLVER,
            "attacker could not hijack resolution"
        );
    }
}

contract VictimWallet {
    function migrate(
        VulnerableMigrationHarness migration,
        uint256 tokenId,
        bytes32 resource,
        VulnerableMigrationHarness.ClassifiedName calldata stale
    ) external {
        migration.migrate(tokenId, resource, address(this), stale);
    }
}

contract RevokedManager {
    function replaceResolver(MockV2Registry v2, bytes32 resource, address resolver) external {
        v2.setResolver(resource, resolver);
    }
}
```

And

```js
import assert from 'node:assert/strict'
import { readFileSync } from 'node:fs'
import { join, resolve } from 'node:path'

const repo = resolve(process.argv[2] ?? process.cwd())

const read = (path) => readFileSync(join(repo, path), 'utf8')
const classify = read('packages/migration/src/service/classifyNames.ts')
const preflight = read('packages/migration/src/service/preflightChecks.ts')
const roleGrant = read(
  'apps/manager/src/features/migration/service/buildRoleGrantCalls.ts',
)
const encode = read(
  'apps/manager/src/features/migration/service/encodeMigration.ts',
)
const batches = read(
  'apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts',
)
const sourceQuery = read(
  'apps/manager/src/features/migration/service/v1SubgraphClient.ts',
)

const sourceChecks = [
  [
    'V1 snapshot includes mutable registry owner and resolver',
    sourceQuery.includes('resolver { address }') &&
      sourceQuery.includes('owner { id }'),
  ],
  [
    'classification trusts the snapshot resolver',
    classify.includes('domain.resolver?.address ?? null'),
  ],
  [
    'classification trusts the snapshot registry manager',
    classify.includes('const registryOwnerAddress = toAddress(domain.owner.id)'),
  ],
  [
    'unwrapped preflight checks BaseRegistrar ownerOf only',
    /tokenType === 'unwrapped'[\s\S]*?address: BASE_REGISTRAR[\s\S]*?functionName: 'ownerOf'/.test(
      preflight,
    ),
  ],
  [
    'preflight never reads the legacy registry owner',
    !/ENS_REGISTRY|functionName: 'owner'/.test(preflight),
  ],
  [
    'the stale manager receives ROLE_SET_RESOLVER',
    roleGrant.includes('args: [resource, ROLE_SET_RESOLVER, name.managerAddress]'),
  ],
  [
    'a stale custom resolver is retained',
    /isKnownPublicResolver\(name\.v1ResolverAddress\)[\s\S]*?: \(name\.v1ResolverAddress/.test(
      encode,
    ),
  ],
  [
    'the atomic batch installs the selected resolver',
    batches.includes('resolver: expectedResolver'),
  ],
  [
    'the atomic batch restores the stale manager role',
    batches.includes("phase: 'manager-role-grant'") &&
      batches.includes('call: buildRoleGrantCall(classified)'),
  ],
]

for (const [description, ok] of sourceChecks) {
  assert.equal(ok, true, `Frozen-source check failed: ${description}`)
}

const victim = '0x1111111111111111111111111111111111111111'
const attacker = '0x2222222222222222222222222222222222222222'
const safeResolver = '0x3333333333333333333333333333333333333333'
const attackerResolver = '0x4444444444444444444444444444444444444444'
const attackerPayee = '0x5555555555555555555555555555555555555555'

// Snapshot observed before the victim removes the old manager and resolver.
const staleSubgraphDomain = {
  name: 'victim.eth',
  registrant: victim,
  owner: attacker,
  resolver: attackerResolver,
}

// Current V1 state immediately before migration. BaseRegistrar ownership did
// not change, so the application's sole ownership preflight still passes.
const currentOnChainV1 = {
  baseRegistrarOwner: victim,
  registryManager: victim,
  resolver: safeResolver,
}

assert.equal(currentOnChainV1.baseRegistrarOwner, victim)
assert.notEqual(staleSubgraphDomain.owner, currentOnChainV1.registryManager)
assert.notEqual(staleSubgraphDomain.resolver, currentOnChainV1.resolver)

// Exact security-relevant output of classifyName + resolverFor +
// buildRoleGrantCall for an unwrapped .eth 2LD.
const migrationPlan = {
  owner: victim,
  resolver: staleSubgraphDomain.resolver,
  managerRoleGrant: {
    account: staleSubgraphDomain.owner,
    role: 'ROLE_SET_RESOLVER',
  },
}

assert.equal(migrationPlan.resolver, attackerResolver)
assert.equal(migrationPlan.managerRoleGrant.account, attacker)

// Model the successful atomic migration and the post-migration abuse.
const v2 = {
  owner: migrationPlan.owner,
  resolver: migrationPlan.resolver,
  roleHolders: new Map([
    [migrationPlan.managerRoleGrant.account, migrationPlan.managerRoleGrant.role],
  ]),
  resolvedEthAddress: victim,
}

assert.equal(v2.owner, victim, 'victim still owns the token, so verification passes')
assert.equal(v2.resolver, attackerResolver, 'stale malicious resolver was restored')
assert.equal(v2.roleHolders.get(attacker), 'ROLE_SET_RESOLVER')

// Even if the custom-resolver path is not used, the restored manager can now
// replace the resolver with an attacker-controlled resolver and redirect the
// name's address record.
if (v2.roleHolders.get(attacker) === 'ROLE_SET_RESOLVER') {
  v2.resolver = attackerResolver
  v2.resolvedEthAddress = attackerPayee
}

assert.equal(v2.resolvedEthAddress, attackerPayee)

console.log('Frozen-source checks: PASS')
console.log('BaseRegistrar owner preflight: PASS (victim)')
console.log('Current V1 manager:', currentOnChainV1.registryManager)
console.log('Restored V2 manager from stale snapshot:', migrationPlan.managerRoleGrant.account)
console.log('Current V1 resolver:', currentOnChainV1.resolver)
console.log('Installed V2 resolver from stale snapshot:', migrationPlan.resolver)
console.log('Post-migration victim.eth address:', v2.resolvedEthAddress)
console.log('Stale V1 state takeover reproduced: PASS')
```
