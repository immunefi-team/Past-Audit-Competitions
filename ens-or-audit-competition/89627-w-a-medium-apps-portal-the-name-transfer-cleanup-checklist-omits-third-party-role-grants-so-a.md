# 89627 w a medium apps portal the name transfer cleanup checklist omits third party role grants so a stranger keeps setresolver on the sold name and repoints its eth address

**Submitted on Aug 20th 2026 at 07:53:27 UTC by @rev3rse for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89627
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

I transferred an ENS v2 name through the portal's transfer flow with **every** cleanup toggle switched on, and the delivered name was still hijackable: a third party the seller had added earlier through the portal's own "Add user" sheet kept `ROLE_SET_RESOLVER` across the transfer, then called `setResolver` while holding **zero tokens** and pointed the name's `addr(60)` record at itself. The buyer stayed `ownerOf` throughout, so the portal keeps showing them as owner while every payment sent to the name resolves to the attacker.

**The transfer flow is an explicit cleanup checklist, and I proved it incomplete.** `SendNameForm.tsx:36-61` offers three toggles (`setEthAddress`, `detachResolver`, `detachRegistry`), all pre-selected `true` at `:69-73`, each with its own `warning` string that appears when you switch it off. `buildTransferPlan.ts:16-43` compiles them into exactly four step kinds, `'set-eth-addr' | 'detach-resolver' | 'detach-registry' | 'transfer-token'`, and **not one of them revokes a role**. With every toggle ticked, a money-moving authority crosses the handoff. That is a correctness defect in `buildTransferPlan.ts`, not a feature request.

**The page that launches the transfer dropped a disclosure it still makes for v1 names.** `routes/$name/ownership/index.tsx:133-135` renders `V1NameManagerRecord` only when `protocolVersion === 'ENSv1'`. That component reads `owner(namehash(name))` off the legacy registry and labels it **"Manager"**, which is precisely the third party who can set that name's resolver. ENSv2 gets no equivalent row, the `Transfer` button sits on the same page at `:114-120`, and the v2 role data is already fetched elsewhere in the app by `useNameRoleAccounts`. This is a v1 to v2 disclosure-parity regression at an identical UI location.

## Vulnerability Details

### Root cause

Roles are keyed on the **label**, ownership is keyed on the **token id**, and the app never reconciles the two at the handoff.

`apps/portal/src/features/roles/helpers/grantRoles.ts:70-71`:

```ts
const { label } = makeLabelNodeAndParent(name)
const resource = labelToCanonicalId(label)
```

`labelToCanonicalId` is the labelhash with its low 32 bits (the ERC-1155 token version) cleared, so the EAC resource is a pure function of the label string. Nothing about the owner, the token id or the token version enters it. My PoC asserts that equality against ensjs's own function on live contract state, and the registry's own revert payload shows it directly:

```
labelhash 0xde6626b4c8cd9215cd250cc2516794a593b7acc1955493cf76369ed04d74e602
resource  0xde6626b4c8cd9215cd250cc2516794a593b7acc1955493cf76369ed000000000
                                                                    ^^^^^^^^ low 32 bits zeroed
```

I measured the resource unchanged across the transfer while `getTokenId` changed. Roles hang off the resource, ownership hangs off the token id, and they are decoupled by construction.

### R-1. The cleanup checklist is incomplete, and its own step list proves it

`apps/portal/src/features/transfer/utils/buildTransferPlan.ts:16-43` is the complete plan:

```ts
export type TransferStepKind =
  | 'set-eth-addr'
  | 'detach-resolver'
  | 'detach-registry'
  | 'transfer-token'
```

The app is not silent about post-transfer cleanup. It offers three toggles, defaults all three to on, and writes a bespoke warning for each one you turn off. It has decided what "handing this name over cleanly" means, and it ships that decision as a checklist. The checklist omits the single authority class that redirects money.

**The app states the outcome it is promising, and the retained role falsifies it.** `SendNameForm.tsx:49` is the `detachResolver` toggle's own description — the toggle whose entire subject is resolver hygiene:

> "Detaches this name's resolver so it stops resolving to your records entirely. **The recipient starts clean and sets up their own.**"

`ROLE_SET_RESOLVER` is precisely the authority that decides whose resolver the name uses. With that toggle on and that grant outstanding, the recipient does not start clean: a third party can re-point the resolver at any moment, and did so in the PoC. I am not asking you to add a feature — I am reporting that a sentence the interface prints, on the step it prints it for, is not true of what the step delivers.

The app already owns both halves of the missing step:

* `apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts:67-74` reads roles during the transfer flow, but for **one** account: `getNameRolesForAccountQueryOptions({ registryAddress, label, account: owner })`. It reads the seller's own roles, to decide which toggles to show. It never enumerates anybody else.
* `useNameRoleAccounts`, the only hook that lists every holder, has two importers repo-wide (`ProtocolVersionWithCounter.tsx:12` and `NameRolesOverviewTable.tsx:13`). Neither is under `features/transfer/**`.
* `apps/portal/src/features/roles/helpers/revokeRoles.ts:70-71` is a working revoke helper that computes the same `labelToCanonicalId(label)` resource. The transfer flow never calls it.

The query exists. The revoke exists. The flow connects neither.

### R-2. The ownership page discloses the v1 manager and discloses nothing for v2

`apps/portal/src/routes/$name/ownership/index.tsx`:

```tsx
114  {canTransfer && (
115    <Button asChild className="gap-2">
116      <Link params={{ name }} to="/$name/ownership/transfer">
117        Transfer
...
133  {data.protocolVersion === 'ENSv1' && (
134    <V1NameManagerRecord asRow name={name} />
135  )}
```

`V1NameManagerRecord` reads `owner(namehash(name))` from `ensLegacyRegistry` and renders it under the label **"Manager"**. For a v1 name, the screen that launches a transfer tells you who else controls the name's resolver. For a v2 name, on the same screen, with the same button, it tells you nothing. The security-relevant disclosure regressed between protocol versions while the transfer affordance stayed put.

## Impact Details

Primary, verbatim from the program's Critical impacts:

> "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow"

Secondary:

> "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves"

`apps/manager` has no transfer route at all, so reading "Manager flow" as "`apps/manager` only" makes the word *transfer* in that parenthetical dead text. The transfer flow lives in `apps/portal`, and that is where I am filing this.

**The harmed party is the buyer.** They never signed, saw or consented to the grant. Their loss is incremental to a normal transfer rather than identical to it: I measured that a clean transfer delivers a name whose only controller is the new owner, because the seller's own bitmap goes to `0x0`, whereas this one delivers a name with a stranger holding `ROLE_SET_RESOLVER`. This is not self-harm.

**The attacker is unprivileged.** M is an ordinary address with no protocol privilege, holding zero tokens, exercising authority over somebody else's name. Nothing here depends on a privileged-role action.

## Dedup note

**My claim is independence, and the test is disjoint fixes: fixing WEB-1257 does not fix this, and fixing this does not fix WEB-1257.** WEB-1257 is an existing step silently skipped. `detach-registry` **is** a `TransferStepKind`, with a toggle defaulting to `true`, hidden by `useTransferDetachTargets.ts:91-95` when the owner lacks `ROLE_SET_SUBREGISTRY`. This finding is an authority class the flow **does not model at all**: different file, different function, a new query and a new step.

The measured corroboration is decisive on the two axes that separate them:

| axis                             | WEB-1257                                           | this finding                                                 |
| -------------------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| whose authority remains          | the **previous owner's**, via the registry pointer | a **third party's**, in EAC role storage keyed on a resource |
| the seller's post-transfer state | retained authority is the mechanism                | **measured `0x0`**, the seller is fully cleared              |
| what the user experiences        | a call **errors**                                  | every call **succeeds**                                      |

After the transfer the previous owner's bitmap is `0x0`, so WEB-1257's mechanism, retained previous-owner authority, is demonstrably not operating here.

I am aware of the clause making reports that describe the same root cause duplicates "even if the report describes a different exploitation path or entry point", which is exactly why I separate at root-cause level rather than at path level. Retained previous-owner authority through an un-detached registry pointer and a retained third-party EAC role on a label-keyed resource are two different root causes in two different storage locations, and the `0x0` measurement is what proves it rather than asserts it. QA-01 states no root cause to be the same as, its status reads "combinations still under test", and its symptom is a call that **errors** where this is a call that **succeeds**.

**And you have already carved this in, in QA-01's own words.** The QA-01 entry on the program page reads, verbatim:

> "QA-01 (Functional, Manager + Explorer) — Certain role combinations when transferring a name, or a name with subnames, cause errors when later updating profile records or setting a primary name. **A demonstrated security consequence (e.g., a role combination that grants unintended authority) would be a new finding.**"

That is not an adjacent clause I am stretching to reach — it is the second sentence of the very entry a dedup would cite, and its worked example is this finding: *a role combination that grants unintended authority*. It is the only one of your 59 known-issue entries carrying that sentence.

The same carve-in appears again at the **Part 2 section header** of the disclosure document — *"a demonstrated security consequence of one of them … goes beyond the known breakage and is a new finding"* — which extends it to WEB-1257 as well. That header also declares Part 2 items *"functional or UX defects, not security findings"*, so WEB-1257's High is a functional priority and cannot absorb a security severity.

Part 1, the 26 security-audit entries, contains nothing on roles, name transfer, the roles page or the indexer. Every nearby item is filed in the functional part.

## References

1. `apps/portal/src/features/transfer/utils/buildTransferPlan.ts:16-43` - the complete transfer plan: four step kinds, none of which revokes a role. This is R-1's defect.
2. `apps/portal/src/features/transfer/components/SendNameForm.tsx:36-61, 69-73` - the three cleanup toggles with their per-toggle warnings, all defaulting to `true`. This is what makes the omission a correctness defect rather than a missing feature.
3. `apps/portal/src/routes/$name/ownership/index.tsx:114-120, 133-135` - the Transfer button and the ENSv1-only `V1NameManagerRecord` row on the same page. This is R-2, the disclosure-parity regression.
4. `apps/portal/src/features/roles/helpers/grantRoles.ts:70-71` - `resource = labelToCanonicalId(label)`, the label-keyed resource that the transfer cannot move. This is the root cause.
5. `apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts:67-74` - the transfer flow's only role read, scoped to `account: owner`, so third-party holders are never enumerated.
6. `apps/portal/src/features/roles/helpers/revokeRoles.ts:70-71` - the working revoke helper, computing the same resource, that the transfer flow never calls.
7. `apps/portal/src/features/roles/components/RolesAddUserSheet.tsx:210-214` - the Admin checkbox hardcoded `disabled`, which is why the buyer always inherits the `_ADMIN` variant and mitigation (b) always works.

## Link to Proof of Concept

<https://gist.github.com/almog-shachar/1142c4b6868324936025a2a287aba84e>

## Proof of Concept

## Proof of Concept

`poc.patch` adds **one** file, `apps/portal/src/__poc_f20v2__/app-driven-transfer.poc.test.tsx`, and modifies nothing. **12 tests, all passing.** It uses vitest, viem, `@testing-library/react` and `@ensdomains/ensjs-abi` — all production or dev dependencies of the app under audit — so no `cast` and no extra tooling is needed.

### The transfer is driven by your code, not by mine

This is the point the previous PoC missed. Your closure named three things — `buildTransferPlan`, `useTransferName` and `SendNameForm`. **All three now execute.**

**`SendNameForm` is rendered and driven as a user drives it:** the component mounts with the three props the transfer route computes, the recipient address is typed into its own `<AddressNameInput>`, the test waits for the component's own `canStart` gate to open, and then clicks its own **"Transfer name"** button. Its own `<TransactionModal>` (`SendNameForm.tsx:166`) opens, and its "Start" / "Open wallet" buttons are clicked; `useAutoAdvanceTransaction` sequences the rest. **The three toggles are asserted rendered and checked, and are never clicked** — this is the shipped default configuration. The executed plan is read off the modal's own rendered step headings, not recomputed: `["Detach resolver","Detach registry","Transfer name"]`, matching the three mined transactions below.

The file also renders `useTransferName` directly inside the app's own provider stack, copied from `src/routes/__root.tsx:48-66` (wagmi → react-query → `TransactionManagerProvider`), using the app's **real `wagmiConfig`** from `@/lib/wagmi` in the repo's own documented local-node wallet mode (`VITE_USE_MOCK_WALLET=true`, `src/lib/mockWallet.mock.ts:8-12`). `startTransfer` is called exactly as `SendNameForm.tsx:108` calls it, with `SendNameForm`'s own default options and **all three cleanup toggles on**. The app's real `TransactionModal` is mounted, so `useAutoAdvanceTransaction` performs the step sequencing; the test fires `onStart()` once.

The three transactions below were **read back off the fork**, not encoded by the test:

```
setResolver(0x9b3c…d5f96ef3, 0x0)                              from A → registry   success
setSubregistry(0x9b3c…d5f96ef3, 0x0)                           from A → registry   success
safeTransferFrom(0xf39f…2266, 0x3c44…93bc, …0200000001, 1, 0x) from A → registry   success
```

### Run it

```bash
git clone https://github.com/immunefi-team/audit-comp-ens && cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
git apply /path/to/poc.patch
pnpm install --frozen-lockfile

# terminal 2 — the fork. Any keyless public Sepolia endpoint works.
# Do NOT pin an old --fork-block-number: this endpoint retains roughly a day of
# state and older blocks return "-32000: historical state is not available".
# Forking at latest is correct here — the PoC registers its own name and deploys
# its own resolvers on the fork, so it needs no historical state.
export SEPOLIA_RPC_URL=https://ethereum-sepolia-rpc.publicnode.com
anvil --fork-url "$SEPOLIA_RPC_URL" --port 8547 --chain-id 11155111

# terminal 1
cd apps/portal
VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8547 VITE_USE_MOCK_WALLET=true \
  pnpm exec vitest run --reporter=verbose \
  src/__poc_f20v2__/app-driven-transfer.poc.test.tsx
```

**Fork and local only. There are no live-chain writes.** Every transaction goes to `http://127.0.0.1:8547`. The preflight calls `anvil_setBalance`, which a real node rejects, so the file cannot be pointed at a live chain by accident. If the fork is not running the preflight throws with the anvil command line rather than skipping, because a silent skip and a clean result look identical.

### What it does

1. Registers a fresh `.eth` 2LD through the real ETH registrar: real `makeCommitment`, real `commit`, real `MIN_COMMITMENT_AGE` wait, real USDC payment, real `register`.
2. Seller **A** grants third party **M** `roleBitmap = 16777216` (`1 << 24`, `ROLE_SET_RESOLVER`) through the app's own `features/roles/helpers/grantRoles.ts`. That is the exact bit the "Add user" sheet sends for the Manager column of Set Resolver on a 2LD: `lib/roles/permissions.ts:42,49` excludes only `ROLE_REGISTRAR`, `ROLE_RENEW` and `ROLE_UNREGISTER`.
3. **A** transfers the name to **B** through `useTransferName` with all three toggles on. The app emits the three transactions above; the two detach steps run and clear the resolver and subregistry.
4. Buyer **B** deploys its own resolver and sets the name up honestly, after the purchase, via the app's own `features/resolver/helpers/changeResolver.ts`.
5. **M** — holding **zero tokens**, with **B** as `ownerOf` — calls the same app helper and moves the name's `addr(60)` to itself.

### The app has this capability and does not use it here

`revokeRoles` is implemented in **six** files under `apps/portal/src`:

```
features/roles/helpers/revokeRoles.ts          features/registry/helpers/revokeRegistryRoles.ts
features/roles/hooks/useRevokeRoles.ts         features/resolver/helpers/revokeResolverRoles.ts
features/roles/utils/buildRoleTransactions.ts  features/roles/components/RolesSidebar.tsx
```

`git grep revokeRoles -- apps/portal/src/features/transfer` returns **zero** matches. (Control: `buildTransferPlan` returns four matches in that same path, so the search reaches it.) The transfer flow is not missing a feature that would have to be built — it does not call one the app already ships.

### Every toggle combination was checked

`buildTransferPlan` is pure, so the PoC enumerates all **8** reachable option combinations (`setEthAddress` × `detachResolver` × `detachRegistry`). **None emits a role step.** No configuration of the checklist — including switching every toggle on, which is the default — revokes the third party's grant.

Every role read is taken four ways, `hasRoles` keyed on the resource, on the raw labelhash and on the token id, plus the raw `roles(resource, account)` bitmap, so a decoder misalignment cannot masquerade as a result.

### Measured output

Full transcript in the PoC notes. The decisive parts, verbatim:

```
=== STAGE 2 - the portal transfers the name to B ===
  SendNameForm defaults      {"setEthAddress":true,"detachResolver":true,"detachRegistry":true}
  buildTransferPlan(options) ["detach-resolver","detach-registry","transfer-token"]
  step labels                Detach resolver | Detach registry | Transfer name

  CALLDATA THE APP PUT ON CHAIN (read back from the fork, decoded):
    from 0xf39f…2266  to 0xbdc8…f0e2  status success
      decoded setResolver(…296250444887, 0x0000000000000000000000000000000000000000)
    from 0xf39f…2266  to 0xbdc8…f0e2  status success
      decoded setSubregistry(…296250444887, 0x0000000000000000000000000000000000000000)
    from 0xf39f…2266  to 0xbdc8…f0e2  status success
      decoded safeTransferFrom(0xf39f…2266, 0x3c44…93bc, …293313814529, 1, 0x)

  STATE AFTER THE APP-DRIVEN TRANSFER
    getResolver(label)     0x0  (detached)
    getSubregistry(label)  0x0  (detached)
```

The two cleanup steps ran and did exactly what they promise. Then:

```
  ROLE BITMAPS - AFTER the app-driven transfer
  who              address                                      hasRoles  rawBitmap
  A   seller       0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266   false     0x0  (none)
  M   third party  0x70997970C51812dc3A010C7d01b50e0d17dc79C8   true      0x1000000  ROLE_SET_RESOLVER
  B   buyer        0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC   true      0x111…100000
  RND control      0x90F79bf6EB2c4f870365E785982E1f101E93b906   false     0x0  (none)

  A 974096550271817618822280174149280430581402828800 -> 0  |  M 16777216 -> 16777216 (SURVIVES)
  The plan the app ran was ["detach-resolver","detach-registry","transfer-token"] - no step revokes a role.
```

The seller's own authority is cleared to `0x0` by the transfer. The third party's is untouched. Then M — holding **zero tokens**, with B as `ownerOf` — repoints the name through the app's own helper:

```
=== STAGE 6 - M exercises the role the transfer left behind ===
  M token balance 0    B token balance 1
  app changeResolver(...) submitted by M -> 0xb63a9b07cc3b91ae…

  registry.getResolver(label)     0x09719c42Be8B9Bb33196ddc303143FEf5d460D29
                              ->  0x84371C4ad865Ea9E7C4ca771dAc1cFF27604242f
  app query getEthAddress(name)   0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
                              ->  0x70997970C51812dc3A010C7d01b50e0d17dc79C8   <- M, the attacker
  ERC-1155 owner of the name      0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC (B never lost the token)
```

**The address the app itself reports.** `getEthAddress` is the app's own ETH-address query (`features/transfer/queries/getEthAddress.ts`). After M's hijack it returns the attacker for the buyer's name, while the buyer remains `ownerOf`. That is Critical #11 read straight out of the application's query layer, on a name the application itself just delivered as clean.

And the negative control, which is what makes the positive result mean something:

```
=== STAGE 5 - negative control: RND makes the identical call ===
  RND -> setResolver reverts:
      Error: EACUnauthorizedAccountRoles(uint256 resource, uint256 roleBitmap, address account)
      sender: 0x90F79bf6EB2c4f870365E785982E1f101E93b906
```

Same call, same arguments, an address that was never granted anything — reverts. The only difference between M and RND is the grant the transfer failed to clear.

Adding the git patch url here again:L <https://gist.github.com/almog-shachar/1142c4b6868324936025a2a287aba84e>
