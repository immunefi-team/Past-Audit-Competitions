# 92522 w a critical unconditional owner fallback in explorer name resolution leads to the app resolving a record less ens name to its owner an attacker controlled address pre filled into the nam&#x20;

**Submitted on Sep 10th 2026 at 12:56:46 UTC by @kylekkkk for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92522
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Explorer app (`apps/portal`) resolves ENS-name inputs in five flows (send/transfer name, grant name roles, grant resolver roles, grant registry roles, create subname) through one shared helper. When the entered name has no ETH address record, the helper — instead of returning the app's "unresolved" state — silently returns the name's on-chain **owner** address. The substitution is invisible: the transfer form shows the owner address as the resolved recipient, and the four dialog flows render it as `Resolved: 0x…`, identical to a record-based resolution. An attacker who owns a name and leaves its address record unset is therefore pre-filled as the recipient/owner whenever a victim enters that name, and the irreversible name-transfer flow inherits this fallback even though the code's stated rationale covers only the role-grant dialogs.

## Affected source path

* Asset: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Repository: <https://github.com/immunefi-team/audit-comp-ens> (fork snapshot of ensdomains/apps-monorepo)
* Commit: `63772fd872af472ced58b009499355f3430c2a86` (audited revision; mirrored at `cda79ac`)
* Package / component: `apps/portal` — `features/roles/helpers/addUser.handlers.ts`, shared resolution query, and five consumers
* Entry point: `resolveAddressOrName({ client, nameOrAddress })`
* Source path: `apps/portal/src/features/roles/helpers/addUser.handlers.ts`
* Affected lines: `27-34` (the owner fallback)
* Chain: Ethereum Sepolia (the network the app operates on)
* Chain ID: 11155111
* Live deployment: `https://explorer.ens.dev`

## Preconditions / actor model

* Actor: any ENS name owner on Sepolia (registration is permissionless; the payment token is faucet-mintable on the testnet)
* Initial privilege: none beyond owning a name (the attacker owns `alice.eth`-equivalent, sets no address record)
* Required capability: none — no transaction is needed at exploit time; the vulnerable resolution is a pure read performed by the victim's own browser
* Required protocol state: the attacker's name has **no** address record (the default state for freshly registered names, which do not set records automatically)
* Victim action: the victim opens a transfer / role-grant / subname flow and enters the attacker's name as the recipient
* Local setup: live deployment `explorer.ens.dev` (read-only UI interaction; no state-modifying transaction was submitted during verification)

## Root cause and trust boundary

* Violated invariant: a name with no address record must resolve to *nothing* (the app's own "Could not resolve an address for …" state), never to a substituted address
* Protected resource: the recipient/owner argument of name-transfer, role-grant, resolver-add-user, registry-add-user, and subname-creation transactions
* Resource owner: the victim (the connected wallet owner initiating the flow)
* Attacker-controlled input: the on-chain state of a name the attacker owns (owner field + absence of records)
* Trusted sink: the recipient/owner field of the transactions the victim signs

The universal resolver returns no address for a record-less name (`getEnsAddress` → `null`). The helper then substitutes the name's owner:

```ts
// apps/portal/src/features/roles/helpers/addUser.handlers.ts:27-34
if (!resolvedAddress) {
  const ownerResult = await getEnsOwner({ name: nameOrAddress })
  if (ownerResult.isOk()) {
    resolvedAddress = ownerResult.value?.owner ?? null
  }
}
```

The code comment states the rationale is the add-user (role-grant) dialogs only ("use current ENS owner address so the role can still be granted"), yet the shared query (`getResolvedAddress.ts`, docblock lines 13-16: "names go through the universal resolver, falling back to the ENS owner when the name has no address record") feeds all five flows, including the irreversible transfer and the subname-owner field, where the owner of the typed name is not the natural meaning of the input. The UI presents the fallback result identically to a record-based resolution: the dialog flows render `Resolved: 0x…` with no disclosure (`AddressNameInput.tsx:59-68`), and the transfer form renders the owner address in the recipient preview box with no indication that the name had no record.

Anticipated objection — "the attacker could simply set the name's address record to their own address, so the fallback adds nothing": the two states are not equivalent for the victim. If the attacker sets an address record, the name's public records show data; the resolution the victim reviews is the standard record path, and the victim's remaining mistake is trusting the name at all. If the attacker sets no record, the name is in the "unconfigured" state that the ENS standard resolution (the universal resolver — `getEnsAddress` returns null) and the app's own "Could not resolve an address" protection state both treat as *resolves to nothing*; the fallback is the only component that converts "nothing" into an address, and it does so with the same confident `Resolved: 0x…` presentation used for real records, while simultaneously making the app's protection state unreachable for that input. The fallback does not enable an attack that a malicious record could not — it removes the one state in which the app would otherwise refuse to answer, and applies it to an irreversible flow beyond its designed scope.

## Attack path

1. The attacker registers (or already owns) a name, e.g. `alice.eth`, and never sets an address record (default state of a fresh registration).
2. The victim opens any of the five flows and enters `alice.eth` as the recipient/owner.
3. The app resolves the name: `getEnsAddress` returns `null` (no record); the fallback substitutes the owner (the attacker's address).
4. The UI displays the attacker's address as the resolved recipient (`Resolved: 0x…` / recipient preview), indistinguishable from a real record resolution; the "Could not resolve an address" protection state is bypassed for any registered record-less name.
5. The victim approves; the transaction is submitted with the attacker's address as recipient (name transfer, irreversible), grantee (role grants), or subname owner.

## Impact

**Selected Impact:** Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

**Severity:** Critical

* Affected parties: Explorer users who transfer a name or grant roles/subname ownership by typing a name
* Funds / assets at risk: one ENS name per transfer (the victim's NFT/name), plus name-management authority in the grant flows
* Valuation timestamp: N/A (name valuation is subjective)
* Valuation source: N/A
* Maximum demonstrated result: the transfer form resolves a record-less, attacker-owned name to the attacker's address with no disclosure (live-verified, see Proof of Concept)
* Repeatability: repeatable for every record-less attacker-owned name

Demonstrated instance: entering `bitb.eth` (a live Sepolia name with no address record, owner `0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750`) in the transfer form of `explorer.ens.dev` resolves the recipient to `0xF83F…4750` — the owner, not a record — with a presentation identical to a record-based resolution (control: `vitalik.eth` → `0xd8dA…045`, its real record value). Both screenshots are attached. Context for calibration, stated plainly: the substituted address is displayed before signing (in full in the transfer form, with an irreversibility warning; truncated or absent from the final modal in the dialog flows), the victim must initiate the action, and the fallback is a code-commented design decision for the role-grant dialogs. The remaining harm — the undisclosed substitution applied beyond its designed scope to an irreversible flow, presented indistinguishably from a record resolution, and defeating the app's own "unresolved" state — is the app-level resolution defect described above.

## Feasibility assessment

* Product state: deployed instance (testnet deployment, `explorer.ens.dev`; the audited code is the frozen competition snapshot)
* Required capital: none (testnet registration; attacker-side setup is a permissionless registration)
* Required timing: none
* Transaction structure: single victim transaction per flow
* Competition / ordering: none
* Worst-case instance: victim transfers a high-value name to the attacker's address believing the typed name identified the intended recipient
* Feasibility limitation: the substituted address is visible in the transfer form before signing; the victim must approve

The attack requires no attacker-side transaction at exploit time, no timing constraint, and only the victim's normal in-app action. It is feasible whenever a user resolves a record-less name through these flows.

## Recommended fix

1. In `resolveAddressOrName`, return `null` when `getEnsAddress` returns `null`, so the existing "unresolved" state surfaces instead of a substituted address.
2. If the owner-convenience is retained for the add-user dialogs, restrict it to those dialogs and render it as a distinct state ("No address record — owner: 0x…"), never the standard `Resolved:` presentation, and never in the transfer or subname-owner flows.
3. Add regression coverage: a record-less owned name must resolve to the "unresolved" state in the transfer flow (exploit case), a record-bearing name must resolve normally (negative control), and the owner-fallback display (if retained) must be visually distinct.

## Observed local confirmation

* Baseline: record-bearing name `vitalik.eth` resolves to its record value `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` (screenshot `f1-transfer-vitalik-record-control.png`)
* Named mutation: the only variable changed between the two screenshots is the recipient input (`bitb.eth` vs `vitalik.eth`)
* Exploit: record-less name `bitb.eth` resolves to its owner `0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750` (screenshot `f1-transfer-bitb-fallback.png`), with no disclosure of the fallback
* Negative control: independent chain reads confirm `getEnsAddress('bitb.eth')` → `null` and `getEnsOwner('bitb.eth')` → `0xF83F…4750` (the exact two calls the helper makes)
* Independent repeat: repeated on three further record-less Sepolia names (`testprojecta.eth`, `secoia-dev.eth`, `kellyhendrix.eth`), all owned by `0xF83F…4750` with `resolver: null`
* Real user data or secrets: none (all addresses are public chain data and testnet test accounts)
* Remote writes: none (no state-modifying transaction was broadcast during verification)

## Deployment correspondence

* Chain: Ethereum Sepolia
* Chain ID: 11155111
* Deployment: `https://explorer.ens.dev` (live Explorer deployment serving the audited application)
* Source revision: `https://github.com/ensdomains/apps-monorepo@63772fd872af472ced58b009499355f3430c2a86` (mirror: immunefi-team/audit-comp-ens `cda79ac`)
* Runtime pin: browser client-side resolution via the universal resolver + v1 registry reads (`viem` `getEnsAddress` / `getEnsOwner` equivalents, same calls the app's ensjs-based helpers perform)
* Fork transport: N/A (web application; verification was live read-only UI + RPC reads)
* Remote writes: disabled

## References

* Asset: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Affected source: `apps/portal/src/features/roles/helpers/addUser.handlers.ts:27-34` (commit `63772fd872af472ced58b009499355f3430c2a86`)
* Shared query: `apps/portal/src/features/address/queries/getResolvedAddress.ts:13-32`
* Display: `apps/portal/src/features/address/components/AddressNameInput.tsx:59-73`
* Consumers: `apps/portal/src/features/transfer/components/SendNameForm.tsx:81`, `apps/portal/src/features/roles/components/RolesAddUserSheet.tsx:73`, `apps/portal/src/features/resolver/components/ResolverAddUserSheet.tsx:141`, `apps/portal/src/features/registry/components/v2/RegistryAddUserSheet.tsx:66`, `apps/portal/src/routes/$name/create-subname.tsx:109`
* Authoring comment referencing the fallback: `apps/portal/src/utils/token/isNameOrAddress.ts:18-21`
* Live name state (queried 2026-09-10): ENS Sepolia subgraph `api.studio.thegraph.com/query/49574/enssepolia` — `bitb.eth` owner `0xf83fe2658f702a072f3c7b0dc4a0ab8c7b044750`, resolver `null`

## Proof of Concept

This reproducer performs read-only interactions against the live deployed application and public Sepolia RPCs. No state-modifying transaction is submitted; the on-chain test names used (registered earlier for verification setup) are attacker-side prerequisites, not PoC actions.

## Fixed deployment context

* Chain: Ethereum Sepolia
* Chain ID: 11155111
* Deployment: `https://explorer.ens.dev`
* Repository: <https://github.com/ensdomains/apps-monorepo>
* Commit: `63772fd872af472ced58b009499355f3430c2a86`
* Source revision: `ensdomains/apps-monorepo@63772fd872af472ced58b009499355f3430c2a86`
* Runtime pin: live deployment (client-side resolution path verified against the audited source)
* RPC environment variable: public Sepolia RPC (any; `https://ethereum-sepolia-rpc.publicnode.com` used)

## Dependencies and configuration

No build required — the PoC is a UI walk plus a chain-read snippet. For the chain-read snippet: Node.js ≥ 20, `viem@2.x`, `@ensdomains/ensjs@<pinned>`, `@ensdomains/ensjs-abi` (any recent versions; the app's own versions are pinned in the monorepo lockfile).

## PoC source

Chain-read snippet reproducing the helper's exact two calls (equivalent to `addUser.handlers.ts`):

```ts
import { createPublicClient, http } from 'viem'
import { sepolia } from 'viem/chains'
import { namehash, normalize } from 'viem/ens'
import { getEnsAddress } from 'viem/actions'
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'

const c = ensL1Contracts[supportedL1Chains.sepolia]
const client = createPublicClient({ chain: sepolia, transport: http('https://ethereum-sepolia-rpc.publicnode.com') })

async function registryOwner(name: string) {
  const node = namehash(normalize(name))
  const res = await client.call({ to: c.ensLegacyRegistry.address, data: `0x02571be3${node.slice(2)}` })
  return `0x${res.data.slice(-40)}`
}

for (const name of ['bitb.eth', 'testprojecta.eth', 'secoia-dev.eth']) {
  const addr = await getEnsAddress(client, { name, universalResolverAddress: c.ensUniversalResolver.address })
  const owner = await registryOwner(name)
  console.log(`${name}: getEnsAddress=${addr ?? 'null'} owner=${owner} fallback=${addr ?? owner}`)
}
```

## Reproduction commands

### 1. UI walk (Explorer, read-only)

1. Connect a wallet that owns any Sepolia ENS name (or use any wallet — the resolution behavior is identical in the dialogs; the transfer form additionally requires the connected wallet to own the name shown in the URL).
2. Open `https://explorer.ens.dev/<owned-name>.eth/ownership/transfer`.
3. In the recipient field enter `bitb.eth`.
4. Observe the recipient preview: `0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750` (the owner of `bitb.eth`), with no indication that the name has no address record.
5. Control: replace the input with `vitalik.eth`; the presentation is identical (`0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`, the record value). No UI distinction exists between the two resolution sources.

### 2. Chain-read reproduction

```bash
node poc.mjs   # prints: getEnsAddress=null, owner=0xF83F…4750, fallback=0xF83F…4750
```

## Expected differential

| Control            | Expected result                                                                                                                          |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Baseline           | `vitalik.eth` (has record) → recipient shown as `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`                                             |
| Exploit            | `bitb.eth` (no record, attacker-owned) → recipient shown as `0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750` (owner), no fallback disclosure |
| Negative control   | `getEnsAddress('bitb.eth')` → `null` (the correct resolution the app discarded)                                                          |
| Independent repeat | three further record-less names owned by the same address resolve identically to that owner                                              |

## Observed output

```
bitb.eth
  getEnsAddress          = null
  getEnsOwner            = 0xf83fe2658f702a072f3c7b0dc4a0ab8c7b044750
  portal fallback result = 0xf83fe2658f702a072f3c7b0dc4a0ab8c7b044750
```

Transfer-form recipient preview (accessibility tree of the live page, 2026-09-10):

```
textbox "ENS name or address"  → "bitb.eth"
  container → "0xF83Fe2658F702A072f3c7b0DC4A0ab8c7b044750"
```

Screenshots: `f1-transfer-bitb-fallback.png` (fallback), `f1-transfer-vitalik-record-control.png` (control).

## Result

The Explorer resolves a record-less, attacker-owned ENS name to the attacker's address and pre-fills it as the recipient (transfer), grantee (role grants), or owner (subname creation) — with a presentation indistinguishable from a record-based resolution, bypassing the app's own "unresolved" state. No state-modifying transaction was broadcast during verification; the demonstrated result is the incorrect, undisclosed resolution that the victim then approves.

## Safety statement

* Execution environment: live deployed application (read-only UI interactions) + public Sepolia RPC reads
* Fork transport: N/A (web application)
* Mainnet transactions: none
* Public testnet transactions: none (verification was read-only; the prerequisite test names were registered separately for evidence setup)
* Remote writes: disabled
* Real user funds or data: none
