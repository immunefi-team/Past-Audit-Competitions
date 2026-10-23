# 92503 w a medium stale v1 manager cache re grants a revoked manager during v2 migration

**Submitted on Sep 10th 2026 at 10:55:17 UTC by @alpha064 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92503
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

At the assessed ENS Audit Competition snapshot (commit 1c9b47f18fcddd2e864dfe385c4171061c9811ae), the Manager application can restore a revoked V1 manager during a normal V1-to-V2 migration. Alice first loads the migration page while Mallory is her legitimate V1 manager, revokes Mallory on-chain through another interface, and returns during the V1 query's five-minute freshness window. The migration still grants Mallory the persistent V2 ROLE\_SET\_RESOLVER permission. Mallory can then replace the resolver without any further action from Alice.

This is a post-revocation privilege re-escalation. It does not require Alice's private key, session theft, signature forgery, compromised ENS contracts, or compromised indexer infrastructure.

## Vulnerability Details

The affected path is the Manager application under apps/manager and the first-party migration package.

1. apps/manager/src/features/migration/service/v1SubgraphClient.ts queries Domain.registrant and Domain.owner from the V1 subgraph.
2. apps/manager/src/features/migration/hooks/useV1Names.ts keeps that result fresh for five minutes with staleTime: 5 \* 60 \* 1000.
3. packages/migration/src/service/classifyNames.ts treats an indexed Domain.owner different from the registrant as managerAddress. It does not read the authoritative V1 ENSRegistry owner(node).
4. packages/migration/src/service/preflightChecks.ts performs fresh BaseRegistrar ownerOf(labelhash) and V2 getStatus checks, but no V1 Registry owner(node) check. Alice therefore passes the fresh ownership preflight while the obsolete manager remains in the plan.
5. apps/manager/src/features/migration/service/buildRoleGrantCalls.ts encodes grantRoles(resource, 1 << 24, managerAddress).
6. apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts places the migration before the manager-role-grant in the same owner-authorized HCA batch.

The Manager source contains no V1 Registry setOwner or setSubnodeOwner transaction path that could perform an external revocation and invalidate this query. invalidateMigrationQueries runs only after migration succeeds. The UI says only “Restore the existing managers for your names” and does not identify the grantee address.

A paired passive observation on 2026-09-10 returned Sepolia block 11674130 and hash 0xa1159dee35d5529327a9fd1225466bbb18e3367cc473d94e2d443707eb1dd721 from both v1-graphql.ens.dev and a Sepolia RPC. The demonstrated attack therefore does not rely on production indexer lag; the realistic vector is client state that was correct when fetched and becomes obsolete after Alice's external revocation.

## Impact Details

The selected Critical impact is the program-specific category: “Hijacking or bricking an ENS name during a Manager flow,” including a V1-to-V2 migration that assigns the name, registry, or resolver to an attacker.

The disposable-chain PoC demonstrates the narrow listed impact directly. After a real delegation and real revocation, Mallory has no current V1 manager authority. Alice's ordinary migration nevertheless grants Mallory ROLE\_SET\_RESOLVER (bitmap 0x1000000). Mallory then successfully calls setResolver on the migrated V2 resource and replaces Alice's resolver with Mallory's address.

Alice must have loaded the affected query, revoke Mallory externally, and migrate while the obsolete value remains usable. Mallory cannot force that victim-side sequence. Once the role is restored, however, it persists and Mallory faces no further race or victim interaction.

The known-issues list does not disclose this root cause. QA-01 covers role combinations that break later operations and expressly says a demonstrated role combination granting unintended authority is a new finding.

## Remediation

At plan construction and again immediately before submission, read the current V1 ENSRegistry owner(node) for every unwrapped name. Derive manager restoration only from that authoritative value. If the owner or any authorization-relevant value changes, discard and rebuild the plan and require a fresh preview. Add regression tests for a revoked manager remaining in cached Domain.owner and for a manager change between plan construction and submission.

## References

* Contest scope and Critical impacts: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Assessed Manager source: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* First-party migration package: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/migration>

## Link to Proof of Concept

<https://gist.github.com/filtede98/32d180c1542ea0794988fd0648999839>

## Proof of Concept

## Proof of Concept

### Safety and topology

The exploit was reproduced only on the repository's disposable local Anvil Sepolia fork. No production contract, user name, wallet, or third-party system was modified. Alice is the BaseRegistrar token holder. Mallory is a previously legitimate V1 Registry manager who is revoked before migration.

Two controls are included:

* Positive path: the client loads Mallory while the delegation is valid, Alice revokes Mallory on-chain, and migration still grants Mallory V2 resolver control.
* Negative source-level control: when indexed Domain.owner already equals Alice, managerAddress is null and no manager grant can be constructed.

### Focused source-level reproduction

From the repository root, run:

```
pnpm --filter manager exec vitest run src/features/migration/service/staleManagerRegrant.poc.test.ts
```

The test uses the production classification, preflight, role-grant encoder, and batch builder. It records that the live preflight functions are only ownerOf and getStatus, decodes grantRoles(resource, 1 << 24, mallory), and verifies the phases migrate then manager-role-grant.

Observed result:

```
Test Files  1 passed (1)
Tests       2 passed (2)
```

### End-to-end reproduction

1. Start the disposable E2E stack:

   pnpm --filter @ens-apps/e2e infra:up
2. Start the localhost-only feature-flag stub:

   node e2e/projects/manager/local-posthog-poc.mjs
3. Start Manager with all test dependencies routed to localhost:

   VITE\_API\_URL=<http://127.0.0.1:9/> VITE\_SEPOLIA\_RPC\_URL=/rpc VITE\_PUBLIC\_POSTHOG\_KEY=local-poc-key VITE\_PUBLIC\_POSTHOG\_HOST=<http://127.0.0.1:3008> VITE\_FF\_USE\_EOA=false VITE\_RHINESTONE\_ENDPOINT\_URL=/orchestrator VITE\_RHINESTONE\_USE\_DEV\_CONTRACTS=true VITE\_RHINESTONE\_CUSTOM\_RPC\_URLS='{"11155111":"[http://127.0.0.1:8545"}](https://reports.immunefi.com/ens-or-audit-competition/http:/127.0.0.1:8545"})' VITE\_INDEXER\_GRAPHQL\_URL=/indexer pnpm --filter manager dev --host 127.0.0.1
4. Run the single Playwright test:

   MANAGER\_APP\_URL=<http://127.0.0.1:3000> ANVIL\_RPC\_URL=<http://127.0.0.1:8545> pnpm --filter @ens-apps/e2e exec playwright test --config=projects/manager/playwright.migration-poc.config.ts

### What the E2E test proves

1. It registers Alice's V1 BaseRegistrar token.
2. Alice delegates the real V1 Registry record to Mallory.
3. Manager loads the legitimate subgraph state identifying Mallory.
4. The test then executes the real V1 Registry transaction Mallory -> Alice, before Alice clicks Upgrade.
5. Every later mocked subgraph response identifies Alice, and the test confirms there is no fresh-owner response between revocation and the Upgrade click.
6. Alice completes the ordinary Manager migration through the current UI.
7. The V2 name changes from RESERVED to REGISTERED.
8. Mallory's V2 role bitmap contains exactly 0x1000000.
9. A transaction impersonating Mallory on the local chain successfully calls setResolver.
10. getResolver(label) returns Mallory's address.

Observed output:

```
[stale-manager-poc] manager cached before revocation 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 regained bitmap=0x1000000 and set the resolver for stale-manager-poc-1789033755.eth to 0x70997970C51812dc3A010C7d01b50e0d17dc79C8; V1-name requests=3, fresh-owner responses=1
1 passed (10.1s)
```

The one fresh-owner response happens after Alice starts migration and does not undo the already-consumed obsolete grant decision. This demonstrates that an up-to-date subgraph does not repair the client-side authorization decision.

### Cleanup

Stop the Manager and feature-flag helper processes, then run:

```
pnpm --filter @ens-apps/e2e infra:down
```

The accompanying PoC source files are intended to be supplied through the report's secret-Gist field because Immunefi's attachment control accepts only PNG/JPEG files.
