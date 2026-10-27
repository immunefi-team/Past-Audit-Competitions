# 91335 w a critical portal applies the registry role schema to a permissionedresolver one ordinary registrar add user grant becomes global set address authority

**Submitted on Aug 31st 2026 at 17:53:58 UTC by @Jornason for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91335
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: resolving or rendering an ENS name to an attacker-controlled address; showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The ENS Portal's Registry roles page applies the Registry role vocabulary to any address its discovery and indexer pipeline labels as a registry, without ever verifying the underlying contract type. On a genuine ENSv2 Registry, role bit 0 is `ROLE_REGISTRAR` ("Registrar / Can register new labels"); on a Portal-deployed PermissionedResolver, the exact same bit position is `ROLE_SET_ADDR`, and both contracts expose the identical EnhancedAccessControl `grantRootRoles(uint256,address)` selector.

An attacker exploits this semantic substitution with only public operations: they register an ordinary decoy `.eth` name through the public ETHRegistrar, gift it to the victim, and set the victim's own PermissionedResolver as the decoy's subregistry. Portal's registry discovery returns that bare resolver address, the Portal generates its own `/registry/<address>/roles` link for it, and the page renders the resolver as a registry with Registry permission labels and an enabled Add User button.

The victim then performs one ordinary Add User flow — entering the attacker as the intended operator of the gifted decoy and checking only "Registrar / Can register new labels" — and with a single wallet confirmation Portal sends `grantRootRoles(1, attacker)` to the PermissionedResolver. The application displays "Registrar" while its own successful transaction grants Set Address: the attacker, who provably held no authority before, gains global address-writing power over every name covered by that resolver's root scope and immediately redirects the victim's genuine name `vladz.eth` to an attacker-controlled address.

This is an application-created wrong-schema authorization (capability false -> true commissioned by the app's own transaction), not a display-only mislabel and not generic untyped routing.

## Vulnerability Details

The required invariant is: the permission vocabulary an application displays and offers must be the vocabulary of the contract the resulting transaction actually targets. When two contract types share one grant selector but assign different meanings to the same bit position, binding the display schema to an unverified address lets the application itself commission authority the user never saw described. Portal violates that invariant between indexer-driven type assumption and calldata construction.

#### One shared selector, two different bit-0 meanings

|                     | ENSv2 Registry (what Portal displays)                     | PermissionedResolver (what Portal targets)                            |
| ------------------- | --------------------------------------------------------- | --------------------------------------------------------------------- |
| bit 0 user role     | `ROLE_REGISTRAR` — "Registrar / Can register new labels"  | `ROLE_SET_ADDR` — write any name's address record                     |
| bit 0 admin variant | `ROLE_REGISTRAR_ADMIN` (bit 128)                          | `ROLE_SET_ADDR_ADMIN` (bit 128)                                       |
| grant entrypoint    | `grantRootRoles(uint256,address)` (EnhancedAccessControl) | `grantRootRoles(uint256,address)` (same shared selector `0x072d5d77`) |

Because the grant selector is shared, a transaction built from the Registry schema executes successfully on the resolver; only the meaning of the granted bitmap changes. The victim's screen says "Registrar"; the mined transaction grants Set Address.

The contracts involved are consequence oracles only:

* ENSv2 `RegistryRolesLib.sol` defines Registry bit 0 as `ROLE_REGISTRAR`.
* `PermissionedResolverLib.sol` defines resolver bit 0 as `ROLE_SET_ADDR`.
* `PermissionedResolver.sol` checks `ROLE_SET_ADDR` before writing address records.
* `EnhancedAccessControl.sol` exposes the shared `grantRootRoles` sink.

The defect and the minimum fix are entirely in the in-scope Portal.

#### Demonstrated attack path

{% stepper %}
{% step %}

### Attacker preparation

The victim normally owns `vladz.eth` and administers its active Portal-deployed PermissionedResolver P; Portal's standard Change Resolver flow initialized the victim with the all-roles admin bitmap on P.

The attacker uses the public `ETHRegistrar.commit()` / `register()` flow to register an ordinary decoy name, assigns ownership to the victim, and sets P as the decoy's `subregistry`. The public registrar accepts a nonzero third-party owner and an arbitrary nonzero subregistry address; the ETH Registry stores and emits that pointer (`SubregistryUpdated`) without a contract-type check.
{% endstep %}

{% step %}

### Discovery and classification

Universal Resolver registry discovery (`findRegistries`) now returns P inside the decoy's registry ancestry array. Portal's name Roles page takes element zero of that bare address array and generates its own first-party `/registry/P/roles` View link; the ENSv2 indexer, following its documented heuristic that new `SubregistryUpdated` addresses indicate dynamically deployed registry contracts, lists P as a registry.

The attacker asks the victim to add them as an operator of the newly delivered decoy name — the same gift/operator premise class as this competition's already-escalated reports #90691 and #90749.
{% endstep %}

{% step %}

### Victim opens the Registry roles page

On the decoy's Roles page the victim clicks View, landing on Portal's own `/registry/P/roles` page, which renders P as "PermissionedRegistry" with Registry role labels and an enabled Add User button.

The route's admin check passes because the victim genuinely holds the bit-128 admin role on P.
{% endstep %}

{% step %}

### Victim grants the displayed Registrar role

The victim clicks Add User, enters the attacker's address, checks only "Registrar / Can register new labels", and confirms once.

Portal submits `P.grantRootRoles(1, attacker)`. On P, bit 0 is `ROLE_SET_ADDR`.
{% endstep %}

{% step %}

### Attacker redirects the victim's name

The attacker immediately calls `setAddr(namehash(vladz.eth), attackerAddress)` and the victim's genuine name resolves to the attacker.
{% endstep %}
{% endstepper %}

There is no victim-pasted contract address, no Retry sequence, no failed or reverted transaction, no race, no malicious resolver, no compromised key, and no pre-existing attacker role. The attacker's registration is preparation; the victim-side exploit is a single documented Add User flow with one successful wallet approval, and the terminal redirect needs no further victim interaction.

#### Root cause: the Registry roles route binds a Registry schema to an unverified address

The Portal downgrades a typed contract object into a bare address and later binds the Registry role schema to that address based only on indexer classification:

1. `apps/portal/src/features/registry/hooks/useNameRegistryDiscovery.ts:56` (`getNameRegistries`) calls ENSJS registry discovery and returns only addresses — no implementation, interface, or contract-kind field.
2. `apps/portal/src/features/registry/components/v2/NameRegistryRolesOverviewTable.tsx:39` treats array element zero as the name's registry (`const address = registries?.at(0)`), and `:47` creates the first-party `/registry/$address/roles` View link from that bare element.
3. `apps/portal/src/routes/registry/$address/roles.tsx:31` accepts the indexer's `getRegistryInfo(address)` result as the entire type gate: if the indexer returns a row, the address is treated as a Registry. The `RegistryInfo` fields requested (`apps/portal/src/features/registry/hooks/useRegistry.ts:39`: `address`, `name`, `namehash`, `parentRegistry`, `createdBlock`, `createdAt`) contain no implementation or contract-kind proof.
4. `apps/portal/src/routes/registry/$address/roles.tsx:40` checks Registry `ROLE_REGISTRAR_ADMIN` against that untyped address and `:77` enables Add User when it succeeds; `apps/portal/src/features/registry/hooks/useRegistryRoles.ts:47` reads the same address's role rows from the indexer and decodes them with Registry vocabulary.
5. `apps/portal/src/lib/roles/permissions.ts:93` labels bit zero `Registrar` with the description `Can register new labels`, and the Add User sheet renders exactly that Registry permission schema and snapshots `{ account, roles }` for the transaction (`apps/portal/src/features/registry/components/v2/RegistryAddUserSheet.tsx`).
6. `apps/portal/src/features/registry/helpers/grantRegistryRoles.ts:45` hard-codes `ROOT_RESOURCE = 0n`, and `:80` sets the transaction's `to` directly to the untyped route address.
7. The pinned ENSJS builder maps resource zero to the shared sink — `functionName: 'grantRootRoles'`, `args: [roleBitmap, account]` (`node_modules/@ensdomains/ensjs/src/actions/wallet/v2/grantRoles.ts:49,53`) — and maps `ROLE_REGISTRAR` to `1n << 0n` (`node_modules/@ensdomains/ensjs/src/utils/v2/roles/registryRoles.ts:2`).

The resulting dataflow is:

```
typed PermissionedResolver P
  -> attacker places P in a public subregistry field (real ETHRegistrar register)
  -> findRegistries returns bare P; Portal generates /registry/P/roles itself
  -> indexer lists P as a registry; route applies the Registry role schema
  -> UI offers bit 0 as "Registrar / Can register new labels"
  -> Portal sends grantRootRoles(1, attacker) to P
  -> P interprets bit 0 as ROLE_SET_ADDR
  -> attacker setAddr(vladz.eth) succeeds; genuine victim name redirected
```

#### The Portal already has the missing check — the Registry roles route does not use it

This is not a missing-capability problem; the fix surface already exists in the same app. `apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts` implements an implementation-aware PermissionedResolver check: it reads the EIP-1967 implementation slot (`0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc`) and compares the decoded implementation against the known `ensPermissionedResolverImpl` address.

That check is used by the resolver side of the app — resolver routes and cards (`apps/portal/src/routes/$name/resolver.tsx`, `ResolverCard.tsx`, `ResolverType.tsx`, `useCanEditRecords.ts`, `useResolverName.ts`) — and is mirrored worker-side in `apps/portal/src/worker/ens.ts:92-123` (`fetchIsPermissionedResolver`, same slot, same comparison).

The Registry roles route (`apps/portal/src/routes/registry/$address/roles.tsx`) contains zero references to it: the route that renders Registry permission labels and enables the grant transaction never asks whether the target is in fact a PermissionedResolver, even though the executed run shows the target's EIP-1967 implementation is exactly the known resolver implementation.

Applying that existing check, or an equivalent implementation/interface verification, on the Registry roles route before rendering Registry vocabulary or enabling the grant is the minimum fix location.

#### The indexer classification is a documented heuristic; the Portal defect stands independently of it

The contracts repository's official indexing documentation (`docs/indexing-ensv2-events.md`) says of `SubregistryUpdated`: "New subregistry addresses indicate dynamically deployed `UserRegistry` contracts for subnames." That is a watch-list heuristic — an indication to track an address — not a certification of contract type, and the same doc applies the symmetric "indicate" wording to `ResolverUpdated` and PermissionedResolvers.

The reported defect does not depend on the indexer being wrong: Portal generates its own `/registry/<address>/roles` link from a bare discovery array element, accepts a bare indexer row as the type gate, and applies the Registry role schema to the target without any on-chain verification of what the contract actually is. Even a perfectly accurate indexer could not supply a type guarantee Portal never requests, because Portal's own route and transaction builder never perform one.

#### Runtime proof (independent end-to-end run, local Sepolia fork)

The end-to-end PoC ran the real frozen Portal UI (competition `audit-comp-ready` head `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, clean tree) against real deployed ENSv2 contracts on a local Anvil fork of Sepolia (chain ID 11155111, fork pin block 11600064). All writes are local; no public Sepolia transaction was broadcast.

The bound target is the real PermissionedResolver proxy `0xa9a5c3bf02eac02ac6e0e5e4ecb5ea3e4d631308` (EIP-1967 implementation `0x9eae5c2730a7dd16bdd1dee6421a1b91e3b0365e`).

The run executed the real attacker-controlled ingress:

* USDC mock mint
* ETHRegistrar approval
* `commit()` (`0x83c00af318d9325ad8a99d21a5150fd44cf872b7e7bde8ca97255d4e877379b3`, block 11600067)
* `register()` (`0x40d81e38bc0c134e40619ec49b34e3e657f475a182d8037da32cd940ce32c8d7`, block 11600069)

The registration created the fresh decoy name owned by the victim with P as its subregistry and resolver, emitting `NameRegistered` and `SubregistryUpdated`.

The preserved original ENSv2 indexer binary materialized P as a registry from exactly that registration block. The preserved indexer's API schema predates the frozen Portal schema, so a read-only compatibility adapter served the two gate operations (`getRegistryInfo`, `getRegistryRoles`) directly from the materialized SQLite database in read-only mode.

A headless browser then drove the real frozen Portal:

* The decoy's Roles page rendered `DECOYW5J8X2.ETH REGISTRY ROLES` with a View link (`01_decoy_name_roles_page_w5j8x2.png`).
* The View route rendered P as `PermissionedRegistry` with an enabled Add User button (`02_registry_roles_page_w5j8x2.png`).
* The attacker address was entered and only the Registrar row's User checkbox was selected: `Registrar / Can register new labels` (`03_registrar_user_only_selected_w5j8x2.png`).
* The `Grant roles` transaction modal opened (`04_transaction_overview_w5j8x2.png`).
* One confirmation sent the transaction and Portal reported `Transaction Complete` (`05_portal_transaction_complete_w5j8x2.png`).

The captured wallet request byte-equals the mined transaction input:

```
transaction = 0x673e88be42b94ee9d40006fb26dca65302648932d64a69e8f221cfcdcb5879e5
block       = 11600070   status = 1
from        = 0xcf8d3063e7074b38f4548a7e2798ce2d498a8ee2 (victim)
to          = 0xa9a5c3bf02eac02ac6e0e5e4ecb5ea3e4d631308 (PermissionedResolver P)
calldata    = 0x072d5d77
              0000000000000000000000000000000000000000000000000000000000000001  (roleBitmap = 1)
              00000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8  (attacker)
decoded     = grantRootRoles(1, attacker)
```

The terminal state was:

| State                                                                                                                              | Before the Portal grant |              After the Portal grant |
| ---------------------------------------------------------------------------------------------------------------------------------- | ----------------------: | ----------------------------------: |
| Attacker root bit 0 (`ROLE_SET_ADDR`) on P                                                                                         |                   false |                                true |
| Attacker `setAddr(vladz.eth)` simulation                                                                                           |                 reverts |                            succeeds |
| `addr(vladz.eth)` via P                                                                                                            |                  victim |            victim (until exploited) |
| After attacker `setAddr` tx `0xb85428d7edbb2b34a5cc21d87f1c868220e47b0981dd70b37cbd5043a599cbb6` (block 11600071, attacker-signed) |                       — |                            attacker |
| Universal Resolver answer for `vladz.eth`                                                                                          |            victim via P | attacker via P (resolver unchanged) |

An independent verifier re-derived 37 checks across nine groups from the live fork, the RPC capture log, the final indexer database, the indexer log, and the frozen source tree: wallet/mined transaction identity, calldata decode, pre/post role delta, simulation revert then success, attacker-signed exploit, Universal Resolver flip, ingress provenance, indexer provenance, frozen Portal HEAD and tree cleanliness, and chain/implementation stability.

Verdict: PASS, 37/37.

#### Evidence boundaries and disclosures

* The evidence is a segmented composition: real attacker-signed ETHRegistrar ingress; preserved-original-indexer materialization with disclosed cursor-only checkpoint acceleration; a read-only schema bridge; the frozen Portal transaction; and fork terminal state. It is not an uninterrupted current-staging-indexer lifecycle.
* The current staging indexer did not ingest the local fork event; no public Sepolia writes were made, and the verifier's public-Sepolia negative control confirms the register hash is absent from public Sepolia.
* The victim intentionally adds the attacker as an operator of the gifted decoy name — the same premise class as this competition's escalated #90691 and #90749.
* A legitimate root admin can revoke the granted role and repair the address record after discovery; this reversibility is disclosed as a severity consideration, not a mitigation that prevents the grant or the immediate redirect.
* The victim's Portal-side signature in the PoC is mediated by the app's own target-provided mock wallet against an impersonated local-fork account; the wallet request bytes are the evidence, and they byte-equal the mined input.
* The indexer and the contracts are out-of-scope consequence/reachability oracles; the reported defect and the minimum fix are in the in-scope Portal.
* An earlier opposite-direction variant in the same target-type-validation family is superseded by this finding and will not be submitted separately.

#### Recommended remediation

1. On the Registry roles route, require a verified contract kind before rendering Registry vocabulary or enabling Registry role actions — at minimum apply the existing implementation-aware PermissionedResolver check (`useIsPermissionedResolver`, EIP-1967 slot comparison) and refuse to treat a known resolver implementation as a Registry; preferably verify a supported Registry implementation or Registry-specific interface.
2. Carry a typed contract-kind/schema identity from discovery (ENSJS `findRegistries`) through the indexer model (`RegistryInfo`) to transaction construction, instead of reducing the target to a bare address.
3. Reject the transaction when the target's verified schema does not match the displayed role vocabulary.
4. Regression: route a PermissionedResolver through registry discovery and assert that Registry actions are unavailable, no Registry permission labels render, and `grantRootRoles` cannot be prepared for that target from a Registry flow.

## Impact Details

Selected impact rows (Websites & Applications):

* **Primary (Critical): Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters / Substituting contract addresses / Submitting malicious transactions.** Portal's own transaction is built from a contract-type assumption the app never verified: the UI displays and the victim selects the narrow "Registrar / Can register new labels" permission, while the application encodes and submits `grantRootRoles(1, attacker)` to a PermissionedResolver where the same bitmap is global Set Address authority. The application affirmatively modifies the decisive transaction semantics — the contract type and therefore the authority represented by the role bitmap — relative to what it presented, and submits the result through the connected wallet in one ordinary confirmation. This is the same application-commissions-beyond-what-it-represented shape already triage-accepted in this competition (#90379, #90691, #90749); here the substitution is an entire role schema (Registrar -> Set Address), and the same Critical row was assessed in full for escalated #90749.
* **Secondary: Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient.** The app displays "Registrar" on the Registry roles page while submitting a Set Address grant to a resolver — the displayed transaction detail, including the permission being granted and the contract type of the grant target, is materially incorrect, and the victim submits the transaction relying on that display.

The application creates authority the attacker did not possess before the victim's action: the pre-state proves the attacker held no role and `setAddr` reverted; Portal's single successful transaction flips root `ROLE_SET_ADDR` false -> true for the attacker, and the attacker immediately redirects the victim's genuine name `vladz.eth` to an attacker-controlled address, with the Universal Resolver continuing to answer through the same now attacker-controlled resolver.

Every name relying on that resolver's root scope is affected, and the grant is global to the resolver, not scoped to the decoy the victim was looking at.

Severity: Critical. Reversibility is disclosed honestly: a legitimate root admin can revoke the role and repair the record after discovery, so this report does not claim permanent irreversibility — but the capability creation is immediate and exclusive to the attacker until discovery, the redirect requires no further victim interaction, and nothing in the product warns that the granted authority differs from the displayed permission.

## References

Scope, source, and public-state checks:

* In-scope asset: `apps/portal` (Explorer app, Websites & Apps): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Program scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Registry roles route (type gate, admin check, Add User enablement): <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/routes/registry/$address/roles.tsx>
* The existing implementation-aware PermissionedResolver check the route does not use: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/hooks/useIsPermissionedResolver.ts>
* Worker-side mirror of the same EIP-1967 check: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/worker/ens.ts>
* Registry permission vocabulary (bit 0 = "Registrar / Can register new labels"): <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/lib/roles/permissions.ts>
* Grant helper (hard-coded root resource, untyped `to`): <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/helpers/grantRegistryRoles.ts>
* First-party `/registry/<address>/roles` link generated from discovery element zero: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/components/v2/NameRegistryRolesOverviewTable.tsx>
* Indexer-driven Registry identity with no contract-kind field: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/hooks/useRegistry.ts>

Public-state refresh on 2026-09-01: `git ls-remote https://github.com/immunefi-team/audit-comp-ens` shows `audit-comp-ready` = `1c9b47f18fcddd2e864dfe385c4171061c9811ae` and a single pull ref (PR #1, a Hono dependency bump unrelated to the Portal Registry role/type path).

The competition's public Known Issues list (59 entries, dated 2026-08-14) was re-checked on 2026-09-01 for any row concerning the Registry roles page, indexer type classification, PermissionedResolver handling, or role bitmap schemas: no exact or near match exists. Private-report overlap remains unknowable and is not claimed either way.

## Link to Proof of Concept

<https://gist.github.com/Jornason/abb0b3c6776ad9420f0125574d991803>

## Proof of Concept

### What this PoC proves

The linked secret Gist bundle preserves an independent end-to-end reproduction (run tag w5j8x2) that drives the real frozen Portal UI through one ordinary Add User flow against real ENSv2 contracts on a local Sepolia fork.

It proves that a victim who opens Portal's own `/registry/<address>/roles` page for their PermissionedResolver, enters the attacker's address, and selects only "Registrar / Can register new labels" produces exactly one Portal wallet request; that the request sends `grantRootRoles(1, attacker)` to the resolver, where bit 0 is `ROLE_SET_ADDR`; that the transaction succeeds; and that the attacker immediately gains and uses global Set Address authority to redirect the victim's genuine name `vladz.eth`.

The setup is the real attacker-controlled public ingress: an attacker-key-signed ETHRegistrar commit/register pair that gifts the decoy name to the victim with the resolver as its subregistry.

All writes are local to the fork. No Sepolia transaction is broadcast, no production user or site is contacted, and no real user key is used. The fork uses Sepolia's chain ID only so the Portal's chain guard accepts it.

### Evidence composition and reproduction map

The evidence is a segmented composition, disclosed in full in the report:

1. Real attacker-signed ETHRegistrar ingress on the fork.
2. Preserved-original-indexer materialization of the resolver as a registry from the registration block, with disclosed cursor-only checkpoint acceleration.
3. A read-only schema bridge serving the two gate operations from the materialized database.
4. The frozen Portal transaction, with the captured wallet request byte-equaling the mined input.
5. Fork terminal state with the Universal Resolver flip.
6. An independent 37-check verifier verdict of PASS.

The bundle's scripts are the exact drivers:

* `preflight_w5j8x2.py` (15/15 gates)
* `attacker_ingress_w5j8x2.py` (7/7 gates)
* `cursor_accelerate_w5j8x2.py`
* `panoptes_watchdog_w5j8x2.sh`
* `panoptes_db_check_w5j8x2.py` (indexer segment, 10/10 database gates)
* `graphql_schema_bridge_w5j8x2.py`
* `rpc_capture_proxy_w5j8x2.py`
* `portal_drive_w5j8x2.py` (headless Chromium drive)
* `terminal_exploit_w5j8x2.py`
* `verifier_w5j8x2.py` (37/37)

Re-running the pipeline requires a Sepolia archive RPC, Docker for the preserved indexer image, the frozen competition checkout at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, Foundry's Anvil, Node 22 + pnpm 10, and Python 3 with Playwright. `run_config_w5j8x2.json` records the exact commands, environment, ports, and timeline.

### Captured exploit steps and retained witness

{% stepper %}
{% step %}

### Preflight

At pin block 11600064, preflight proves the target is a PermissionedResolver through EIP-1967 implementation verification, `vladz.eth` resolves through it to the victim, the victim holds the bit-128 admin role, and the attacker holds no bit-0 role with a reverting `setAddr`.

Evidence: `evidence/rpc/00_preflight_w5j8x2.json`
{% endstep %}

{% step %}

### Attacker ingress

The attacker's signed commit (`0x83c00af318d9325ad8a99d21a5150fd44cf872b7e7bde8ca97255d4e877379b3`) and register (`0x40d81e38bc0c134e40619ec49b34e3e657f475a182d8037da32cd940ce32c8d7`) create the decoy owned by the victim with `subregistry = resolver =` the target, emitting `NameRegistered` and `SubregistryUpdated`.

The register hash is confirmed absent from public Sepolia.

Evidence: `evidence/rpc/01_attacker_ingress_w5j8x2.json`
{% endstep %}

{% step %}

### Indexer materialization

The preserved indexer binary writes the registry row for the target from block 11600069 itself.

Evidence:

* `evidence/indexer/panoptes_indexer_full_log_w5j8x2.txt`
* `evidence/indexer/panoptes_db_verification_w5j8x2.json`
* `evidence/indexer/cursor_acceleration_w5j8x2.json`
  {% endstep %}

{% step %}

### Portal browser drive

The browser drive captures:

* The decoy Roles page with the Portal-generated View link: `01_decoy_name_roles_page_w5j8x2.png`
* The resolver rendered as `PermissionedRegistry` with Add User enabled: `02_registry_roles_page_w5j8x2.png`
* The attacker address with only the Registrar User checkbox selected: `03_registrar_user_only_selected_w5j8x2.png`
* The `Grant roles` modal: `04_transaction_overview_w5j8x2.png`
* The completed transaction: `05_portal_transaction_complete_w5j8x2.png`
  {% endstep %}

{% step %}

### Portal transaction

Portal's wallet request, capture sequence 25 in `evidence/rpc/portal_rpc_capture_w5j8x2.ndjson`, decodes to `grantRootRoles(1, attacker)` targeted at the resolver from the victim.

It is byte-equal to the mined input of `0x673e88be42b94ee9d40006fb26dca65302648932d64a69e8f221cfcdcb5879e5` at block 11600070, status 1.
{% endstep %}

{% step %}

### Terminal exploit

Post-grant snapshots prove the false -> true `ROLE_SET_ADDR` delta and a successful attacker `setAddr` simulation.

The attacker's signed `setAddr` transaction `0xb85428d7edbb2b34a5cc21d87f1c868220e47b0981dd70b37cbd5043a599cbb6`, at block 11600071, flips `vladz.eth` from the victim address to the attacker address through an unchanged resolver.

Evidence:

* `evidence/rpc/03_post_portal_snapshot_w5j8x2.json`
* `evidence/rpc/05_universal_resolution_flip_w5j8x2.json`
* `evidence/rpc/06_terminal_snapshot_w5j8x2.json`
  {% endstep %}
  {% endstepper %}

The fork transaction hashes are local Anvil transactions, not Sepolia transactions; they are reproducible within the documented fork setup. The independent verifier's full verdict is `evidence/rpc/07_independent_verdict_w5j8x2.json` (PASS, 37/37).

### Attached code and integrity

The complete bundle — all run scripts, the RPC/indexer/browser evidence tree, the bridge audit log, the run configuration, the contracts configuration, and `SHA256SUMS.w5j8x2.txt` covering every file including the hash-pinned indexer database snapshots and docker image layers retained in the original evidence workspace — is in the linked secret Gist:

<https://gist.github.com/Jornason/abb0b3c6776ad9420f0125574d991803>

The flattened manifest verifies in the Gist root as-is. Publication sanitization removed internal candidate identifiers from the run configuration's purpose field and mechanically path-sanitized the dev-server log; no evidence-bearing value, including addresses, calldata, role state, transaction hashes, receipts, or block numbers, was altered.

The five UI screenshots are attached to this report and are byte-identical to the bundle copies.
