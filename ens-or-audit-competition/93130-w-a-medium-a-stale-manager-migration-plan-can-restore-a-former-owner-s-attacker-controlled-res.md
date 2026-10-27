# 93130 w a medium a stale manager migration plan can restore a former owner s attacker controlled resolver&#x20;

**Submitted on Sep 14th 2026 at 10:45:56 UTC by @unineko for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93130
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

**Resubmission evidence delta:** The earlier closure reported Foundry 0/4 with missing fixture environment variables and no executed frontend capture. This revision makes the independently reproduced browser/fork test the **primary** PoC: `1 passed`, V1 read B after refresh, the mined Manager wallet transaction input contains A but not B, and V2 stores A. The supplementary `run.sh` supplies the required fixtures and passes Manager 2/2 plus Foundry 2/2; directly running Forge is not the instructed reproduction. The indexer and headless-wallet test boundaries remain disclosed below.

**Summary**:

The Manager prepares a v1-to-v2 migration plan using complete V1 domain objects, but identifies the cached plan only by the wallet, HCA, selected domain IDs, and a selection-only revision. The identity does not include the resolver or an equivalent authority-state fingerprint.

As a result, for a wallet legitimately admitted to the feature-flagged migration rollout, the current V1-name query can refresh from a former owner's attacker-controlled resolver A to the victim's repaired resolver B while an A-based plan remains `ready`. The normal Upgrade action then submits the retained plan without comparing it to the current V1 resolver. On the demonstrated unwrapped `.eth` route, V2 stores A and the victim's name resolves to the attacker.

This directly matches the selected Critical impact: the in-scope Manager migration assigns the name's resolver to an attacker.

**Description**:

`useMigrationGasEstimate` derives the plan identity from account addresses and domain IDs, even though its query function closes over the full `domains` objects:

```typescript
const domains = selectDomainsFromNames(v1Names, selectedNames)
const domainIds = domains
  .map((domain) => domain.id)
  .sort()
  .join(',')

const query = useQuery({
  queryKey: [
    'migration-gas-estimate',
    ownerAddress?.toLowerCase() ?? '',
    hcaAddress?.toLowerCase() ?? '',
    domainIds,
    selectionRevisionRef.current.revision,
  ] as const,
  staleTime: 0,
  queryFn: async () => {
    const preflight = await ensurePreflight(domains, { staleTime: 0 })
    const plan = await buildMigrationPlan({
      domains,
      hcaAddress,
      migrationOwner: ownerAddress,
      publicClient: publicClient as unknown as PublicClient,
      preflight,
    })
    // ...
    return { estimate, plan }
  },
})
```

A resolver change from A to B does not change `domainIds` or the selection revision. A valid refresh completion ordering can therefore leave the earlier plan available:

```typescript
return {
  status: 'ready',
  // ...
  plan: query.data.plan,
}
```

This refresh is enabled by the production query configuration. `useV1Names` appears to request `staleTime: 5 * 60 * 1000`, but the local `resultQueryOptions` wrapper destructures `staleTime` without returning it. Manager's QueryClient default `staleTime: 0` therefore applies. On focus, TanStack Query 5.90.12 refetches both active queries: the gas refetch starts from its then-current A closure while the names query can return B. Publishing B retains the same gas-query identity, and replacing that same-key query function does not start a follow-up B fetch. Consequently, either completion order leaves the A plan authoritative; the supplied PoC demonstrates the ordinary focus-refresh path.

That plan contains a migration resolver derived from the captured domain state:

```typescript
const expectedResolver = resolverFor(classified, defaultResolver, resolver)
const migrationData = createMigrationData({
  label: classified.label,
  owner: wallet,
  subregistry: zeroAddress,
  resolver: expectedResolver,
})
```

The Upgrade handler checks only that the estimate is ready and forwards the plan unchanged:

```typescript
if (gasEstimate.status !== 'ready') return false

uiActor.send({
  type: 'migration.start',
  plan: gasEstimate.plan,
  signer,
  hcaClient,
  refreshAccount,
})
```

There is no final comparison against the live V1 resolver before the wallet request. The execution service does recheck approval state and deterministic route readiness, but it rebuilds each batch from the retained `plan.classified` and `plan.profiles`; neither check reloads the V1 resolver. Its post-state verifier is also derived from that retained plan and therefore expects A rather than detecting A as stale.

Preserving a current custom resolver is an intended and tested migration behavior, and the release notes explicitly say that custom resolvers are preserved. Silently reinstalling a historical resolver after the current owner replaced it is not documented or authorized by that design.

**Root cause**:

The prepared migration plan is not bound to a canonical snapshot of the authority-bearing V1 state, and the Manager does not perform a final freshness check before wallet submission. The downstream contracts correctly consume the supplied migration data; the vulnerability is that the in-scope Manager supplies obsolete security-critical data.

**Attack prerequisites**:

* Resolver A is a custom resolver, so the migration route preserves it rather than replacing it with the new owned resolver.
* The victim's wallet is legitimately included in the intended `migration === true` PostHog rollout; no feature-flag bypass is claimed.
* Manager first builds a plan while A is current. The victim then replaces A with B, and the same Manager session receives the updated B state for the same domain ID and selection.
* The victim performs the normal Manager Upgrade action.

The PoC and impact claim rely only on the demonstrated unwrapped `.eth` route. No broader wrapped/locked route claim is required. In particular, the locked `CANNOT_SET_RESOLVER` path is excluded because its receiver reads the V1 resolver itself.

**Attack sequence**:

1. While legitimately owning the name, the attacker installs immutable resolver A, which returns the attacker's address, and transfers the BaseRegistrar token to the victim.
2. The victim reclaims ENS Registry ownership. Resolver A remains, producing the complete and legitimate state `registrant=victim, owner=victim, resolver=A`.
3. The victim enters the feature-flagged Manager migration, selects the name, and Manager prepares plan A while A is current.
4. Before clicking Upgrade, the victim uses another tab or tool to replace A with safe resolver B. The attacker takes no action. After B is indexed, the victim returns to the Manager tab.
5. The ordinary focus refresh publishes B for the same domain ID, but the gas-plan refetch has captured A. The unchanged plan identity prevents a follow-up B build, so plan A remains ready under either completion order.
6. The victim clicks the normal Upgrade button. The production Manager transaction path sends wallet calldata containing A.
7. Migration succeeds with the victim as V2 owner but A as the V2 resolver. The name consequently resolves to the attacker until the victim notices and repairs it.

At migration time, the attacker has no name, resolver-admin, HCA, session, wallet, operator, controller, or governance role. Resolver A has no setter, owner, or upgrade path, and the attacker sends no transaction after the victim's A-to-B repair.

**Impact**:

The victim's completed resolver repair is silently undone during an in-scope Manager migration. V2 stores an attacker-controlled resolver, allowing the attacker to control resolution of the migrated name, including returning the attacker's address for address-record queries.

The victim remains the V2 owner and can repair the resolver after detection. This report does not claim ownership theft or permanent bricking; the demonstrated attacker resolver assignment is itself expressly covered by the program's Critical Manager-migration impact.

**Nearest Known Issue**: The [published Known Issues](https://immunefi.com/audit-competition/audit-competition-ens/information/#known-issues) list WEB-806 for migrated subname records not appearing in a Manager profile, WEB-390 for subnames missing from migration discovery, and WEB-1304 for wrapped-name expiry. None describes retaining and submitting a stale resolver for an unwrapped .eth name. This report does not rely on a privileged attacker, browser-storage modification, or a subgraph/contract defect; the issue is the in-scope Manager's stale transaction construction.

**Recommendation**:

Bind every migration plan to a canonical fingerprint of all security-relevant source state. At minimum, include each name's node/domain identity, registrant and Registry/wrapped owner, resolver, token type, parent, relevant fuses/expiry, and route-dependent approval state.

Before requesting a wallet signature:

1. Read the canonical V1 state for every selected name at a consistent block, preferably from the relevant Registry, Wrapper, and BaseRegistrar contracts.
2. Recompute the fingerprint and compare it with the plan's stored source fingerprint.
3. If any authority-bearing field changed, discard the plan and rebuild classification, preflight, routes, batches, verification expectations, and gas estimation; otherwise fail closed.
4. Bind post-state verification to the freshly approved expected resolver, not to the stale plan itself.
5. Display the exact resolver that will be installed before submission.

Including the fingerprint in the React Query key is useful but is not a complete fix: state can still change after plan construction and before wallet submission. Add a regression test for `same domain ID, resolver A -> B` that requires the wallet request to contain B or blocks submission.

## Link to Proof of Concept

<https://gist.github.com/unineko5555/5fa995d23b2055ff7629e113d3ee4304>

## Proof of Concept

[`PoC`](https://gist.github.com/unineko5555/5fa995d23b2055ff7629e113d3ee4304)

**PoC Result**: PASS — primary Manager browser/fork 1/1, deployed-contract Sepolia-fork Foundry 2/2, supplementary standalone Manager 2/2 and Foundry 2/2. The browser run observed V1 resolver B after refresh, then captured a mined Manager wallet transaction whose input contains A and excludes B. V2 stored A, which returned the attacker's address.

The complete 13-file PoC is the [unlisted Finding 1 Gist](https://gist.github.com/unineko5555/5fa995d23b2055ff7629e113d3ee4304), verified revision `a85f2dfa8f0883619e080d6eca5c7519b6fea803`. It must be placed inside the ENS apps checkout; it imports production source and E2E fixtures. GitHub Gist flattens directories. From a clean checkout where `poc/finding1` does not yet exist, restore the paths before running the **primary browser-to-deployed-contract proof**. The isolated `run.sh` is supplementary; running it or Forge alone does not establish the browser claim:

```bash
mkdir -p poc
git clone https://gist.github.com/unineko5555/5fa995d23b2055ff7629e113d3ee4304.git poc/finding1
git -C poc/finding1 checkout a85f2dfa8f0883619e080d6eca5c7519b6fea803
mkdir -p poc/finding1/manager poc/finding1/runtime/test poc/finding1/runtime/fixtures poc/finding1/e2e
mv poc/finding1/Finding1.poc.test.tsx poc/finding1/vitest.config.ts poc/finding1/manager/
mv poc/finding1/Finding1.t.sol poc/finding1/Finding1DeployedFork.t.sol poc/finding1/runtime/test/
mv poc/finding1/foundry.toml poc/finding1/runtime/
mv poc/finding1/migration-helper.runtime.hex poc/finding1/wallet-outer-calldata.hex poc/finding1/runtime/fixtures/
mv poc/finding1/finding1-ui.spec.ts poc/finding1/playwright.config.ts poc/finding1/package.json poc/finding1/e2e/
pnpm install --frozen-lockfile
```

### Primary reproduction: Manager browser to deployed V2 Registry

Use the complete ENS apps tree at local commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, Node v22 (v24 also tested), pnpm 10.27.0, Bash, Foundry forge/cast, Docker, and Playwright Chromium. Use an **exclusive disposable Sepolia fork**; the browser test makes fork-local setup writes. From the checkout root, start Anvil and mockestrator, then run the deployed-contract precheck, which also compiles the resolver artifact required by the browser test. Do not stop after this Forge precheck:

```bash
docker compose -f e2e/infra/docker-compose.yml up -d anvil mockestrator
bash ./poc/finding1/run-fork.sh
```

In a second terminal, leave the audited Manager development server running:

```bash
pnpm --filter manager dev --mode ci --host 127.0.0.1
```

After `http://127.0.0.1:3000` responds, run in a third terminal from the checkout root:

```bash
E2E_MOCK_INDEXER=true e2e/node_modules/.bin/playwright test \
  --config=poc/finding1/e2e/playwright.config.ts
```

Expected: `1 passed`; a V1 GraphQL read of B after the victim's repair; `WALLET_CALL ... inputContainsA=true inputContainsB=false`; and V2 resolver A returning the attacker's address. The new wallet assertion reads the mined EOA transaction input emitted by the real Manager browser flow. The E2E was reproduced on an independent fresh Docker VM as well as the earlier local fork. It does not use an external wallet or independent indexer, and sends no public-Sepolia transaction. The fork setup uses limited privileged impersonation to create a fresh name, not attacker privilege at migration time.

### Supplementary standalone suites

Only after the primary browser proof, run the self-contained local suites if additional controls are wanted:

```bash
bash ./poc/finding1/run.sh
```

`run.sh` verifies both bundled hex fixture hashes, exports `LIVE_RUNTIME` and `CAPTURED_OUTER_CALLDATA`, runs the production Manager Vitest, then runs Foundry. **Do not invoke `forge test` directly** without those fixture variables; that was the cause of the previous 0/4 triage run. Foundry may download Solidity 0.8.28 if uncached; afterward `FOUNDRY_OFFLINE=true bash ./poc/finding1/run.sh` runs fully offline. Expected: Manager 2/2, Foundry 2/2, and `[POC-PASS]`.

The Manager test imports the production migration page, queries, plan builder, encoder, UI state machine, migration service, transaction manager, EOA transport, and Viem. It starts from complete A state, makes a normal focus refresh publish complete B state, clicks Upgrade, and captures the EIP-1193 wallet request containing A. The fresh-B control sends B instead. The captured outer calldata hashes to 0x7dd068547afa4ac3035df4ad755a8c3a77861bd42512b245316caee77656333b. The standalone Foundry test executes those exact bytes through a local HCA owner path and a bundled MigrationHelper runtime whose hash matches the Manager pin. Its V1 stand-in reports B, while the V2 stand-in stores A and resolves to the attacker; the B control resolves to the victim. These are deterministic local boundaries, not live deployed downstream contracts.

The primary `run-fork.sh` precheck checks the chain ID, Anvil upstream-fork configuration, and deployed MigrationHelper runtime hash. Its two Foundry tests use real deployed V1 Controller, BaseRegistrar and Registry, and real deployed MigrationHelper and V2 Registry, with no vm.etch or vm.mockCall. Both tests set the real V1 resolver to B before migration; stale-A input installs A, while fresh-B input installs B. They run in isolated in-process forks and do not write to the shared Anvil.

The browser test and config live under `poc/finding1/e2e` and only import the repository's E2E fixtures; no file under the audited `e2e/` tree is added or changed. It confirms that the V1 GraphQL resolver read has changed to B on a visibility refresh, yet the real UI submits A. It reads the **mined Manager transaction input** and asserts the stale resolver A is encoded while current B is absent. It checks V2 status REGISTERED, resolver A, and the attacker's address through A. The fresh independent-VM run logged `WALLET_CALL ... inputContainsA=true inputContainsB=false` and `1 passed`; its transaction hash exists only on that disposable local fork.

The fork-only fresh-name setup impersonates the V1 BaseRegistrar owner to authorize a disabled Controller and the V2 Registrar to reserve the label. The browser fixture also clears local wallet-account code, funds test accounts, and uses mockestrator and a mocked V2 indexer. The public V1 subgraph cannot index fork-local writes, so the browser stubs its discovery/profile responses; the resolver field is read from the live fork-local V1 Registry on every request. These controls neither replace the deployed V1/V2 Registry or MigrationHelper calls nor give the attacker authority at migration time. This is a local real-UI/deployed-contract corroboration, **not** a no-mock production E2E. Stop the Manager server and the fork services after the check; the Gist README gives cleanup commands.
