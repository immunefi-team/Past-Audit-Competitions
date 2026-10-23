# 89395 w a critical unsafe owner fallback in the shared address resolver leads to irreversible transfer of ens names to an unintended address

**Submitted on Aug 19th 2026 at 02:12:29 UTC by @Ispolin11 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89395
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as:
    * a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker
    * redirecting the recipient of a name transfer
    * leaving a name permanently inaccessible after the flow
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

A user who transfers an ENS name in the Explorer app, and types the recipient's ENS name instead of pasting a raw address, can irreversibly send that name to an address they never chose. In the cases this report covers, that address belongs to the previous owner of the recipient's name. The cause is one shared helper: five "ENS name or address" inputs across the app resolve through it, and when the typed name has no `addr` record it falls back to the name's registry `owner`. For an unwrapped ENSv1 `.eth` name the registry owner is the *controller*, a manager role, not the *registrant* who holds the registrar NFT and is the party the user means. The fallback was written for the roles feature ("so the role can still be granted"), but the hook that consumes it is documented as shared by *"every 'name or address' input in the app (transfer recipient, add-user sheets, …)"*. The same substitution decides who owns a newly created subname, and grants that address every role on it. On screen the substituted address is rendered in exactly the same words as an address taken from a genuine `addr` record, so nothing distinguishes the two.

## Vulnerability Details

### 1. The fallback returns the wrong role's address

[`apps/portal/src/features/roles/helpers/addUser.handlers.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/roles/helpers/addUser.handlers.ts#L11-L38)

```ts
const resolved = await getEnsAddress(client, { name: nameOrAddress, universalResolverAddress })
let resolvedAddress = resolved

// Fallback for names that do not set an address record:
// use current ENS owner address so the role can still be granted.
if (!resolvedAddress) {
  const ownerResult = await getEnsOwner({ name: nameOrAddress })
  if (ownerResult.isOk()) {
    resolvedAddress = ownerResult.value?.owner ?? null
  }
}
return resolvedAddress
```

`getEnsOwner` delegates to [`resolveEnsOwner`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/utils/ens/resolveEnsOwner.ts), which tries the V2 registry first and falls through to ensjs v1 `getOwner`. For an unwrapped `.eth` 2LD the pinned ensjs build returns both roles as separate fields, and labels them. In the ordinary case the two hold the same address. They diverge whenever the registry owner is not the NFT holder: most often after a sale the buyer has not reclaimed, but also when a holder has deliberately delegated management to another address, which the v1 registry allows:

```ts
// the registrant is the owner of the NFT
// the owner is the controller of the records
if (registrarOwner) {
  return {
    registrant: registrarOwner,   // the NFT holder
    owner: registryOwner,         // the registry controller, a manager role
    ownershipLevel: 'registrar',
  }
}
```

The application takes `.owner`.

This is not an obscure distinction in this codebase. Three other files document it, in the team's own words:

| File                                                                                                                                                                                                                   | Comment                                                                                                                                            |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`features/migration/hooks/useMigrationStatus.ts#L35`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/migration/hooks/useMigrationStatus.ts#L35)                       | *"the Base Registrar registrant (the registry `owner` is the controller, which may be a different address, so it is only a last-resort fallback)"* |
| [`features/registry/components/v2/MigrateRegistryPrompt.tsx#L17`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/components/v2/MigrateRegistryPrompt.tsx#L17) | *"the name's own V1 token holder (wrappedOwner/registrant — NOT the registry controller)"*                                                         |
| [`routes/addr/$addr/names.tsx#L185`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/routes/addr/%24addr/names.tsx#L185)                                                         | *"For unwrapped names: registrant is Owner, registry owner is Manager"*                                                                            |

The migration helper applies the precedence a fix would need: it prefers the registrant over the registry owner and falls back to the owner last, which is what its comment calls a "last-resort fallback". On the path described here, `registrant` is never consulted at all.

### 2. The shared type discards `registrant`

```ts
export type ResolvedEnsOwner = {
  owner: Address
  registryAddress: Address
  protocolVersion: ProtocolVersion
} | null
```

`resolveEnsOwner`'s own docstring says it is *"shared between the React app (useEnsOwner) and the SSR/OG-image worker"*. It was built for display, and a two-address concept was flattened into one field to serve a renderer. Once the type drops `registrant`, no consumer downstream can recover it, whatever it does with the value.

### 3. Where the substituted address ends up

`resolveAddressOrName` → [`getResolvedAddress`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/address/queries/getResolvedAddress.ts) → [`useAddressResolution`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/address/hooks/useAddressResolution.ts). Five inputs share that hook, and two of them assign an asset:

| Consumer                                                                                                                                                                                       | What the resolved address becomes                                                           |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| [`features/transfer/components/SendNameForm.tsx#L81`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/components/SendNameForm.tsx#L81) | recipient of an ENS name transfer                                                           |
| [`routes/$name/create-subname.tsx#L109`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/routes/%24name/create-subname.tsx#L109)                         | owner of a newly created subname, granted `DEFAULT_ROLE_BITMAP` = `0x1111…1111`, every role |
| `features/roles/components/RolesAddUserSheet.tsx#L73`                                                                                                                                          | grantee of registry roles                                                                   |
| `features/registry/components/v2/RegistryAddUserSheet.tsx#L66`                                                                                                                                 | grantee of registry roles                                                                   |
| `features/resolver/components/ResolverAddUserSheet.tsx#L141`                                                                                                                                   | grantee of resolver permissions                                                             |

The bottom three are the case the fallback was written for. Even there it grants the permission to the controller rather than the registrant, which is arguably also wrong, though handing a manager role to the manager is at least defensible and I am not filing it. The top two hand over an asset, which the fallback was never meant to decide.

In the transfer, `recipient` passes unchanged from the hook through [`useTransferName.ts#L111-L143`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/hooks/useTransferName.ts#L111-L143) into `transferToken.ts`:

```ts
encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
})
```

Nothing on that path checks ownership or where the address came from. The only recipient validation in `SendNameForm` is:

```ts
const hasValidRecipient = !!recipient && !isSelf && !isZeroAddress
```

### 4. The UI does not mark where the address came from

Rendering the shipped `AddressNameInput` with two resolutions that differ only in provenance gives the same line, the same truncation and the same CSS classes. Only the hex differs, and the user has no second source for it: they typed a name rather than an address because the address is what they do not have.

```
address from a genuine addr record : "Resolved: 0x7099…79C8"
address from the owner fallback    : "Resolved: 0xD613…8c30"
same CSS classes                   : true
```

The component does vary its wording by provenance when it chooses to: a raw address the user typed renders as `Using address: 0x…` rather than `Resolved: 0x…`. So the distinction is one it is built to make. There is simply no third case for "no addr record, using the name's registry controller".

`SendNameForm` overrides that line with `RecipientPreview`, which shows an avatar, the resolved address's primary name if it has one, and the full untruncated address. That cuts both ways and I do not want to overstate it. If the controller has a primary name, the screen shows a name Alice did not type, which is a visible discrepancy an attentive user could catch. The case that survives on this screen is a controller with no primary name: an avatar and a bare hex address, with nothing indicating its origin. The four consumers that use the default line, `create-subname` among them, offer no cross-check at all.

### 5. Preconditions, and why they persist

Both apply to the name typed into the field, not to the name being sent:

1. It is an ENSv1 name. `resolveEnsOwner` tries V2 first, so a V2 name resolves correctly.
2. It has no `addr` record. Otherwise `getEnsAddress` returns and the fallback never runs.

The two protocol versions here are independent of each other. The Explorer transfers ENSv2 names, which are ERC-1155 tokens on a v2 registry, while the recipient name only needs to be v1. Sending a v2 name to someone whose own ENS name is still v1 is an ordinary combination, not a contrived one.

The recipient cannot clear the second condition by setting a record. Resolver records authorise on the registry controller, so while the seller holds that slot the buyer's attempt to publish an `addr` record reverts on-chain — case C5 in the Proof of Concept section below. The exit is `reclaim()`, which the buyer has to know to call: it is not part of buying a name and nothing in the flow prompts it. Until they do, the state holds.

### 6. This is not a stale `addr` record

When the name does carry an `addr` record, even one still pointing at the seller, the app returns that record through a correct lookup. That is an ENS-wide data staleness problem and not this app's defect. The finding is the narrower case where the app substitutes an address that no resolver would have returned. Case C1 in the Proof of Concept section checks this explicitly, so the scope of the claim is bounded rather than assumed.

### 7. What a fix has to touch

Noting this because the shape of the fix bears on how the defect reads: it is not a null check. Removing the fallback from the two asset-assigning consumers would close the impact reported here, but the shared helper would still hand a manager address to anything that asks it for an owner. A fix that holds has to carry `registrant` through `ResolvedEnsOwner` and let each consumer say which role it means — the roles sheets genuinely want the controller, a recipient input wants the registrant.

That choice only arises on the v1 unwrapped path. For wrapped names ensjs returns no registrant at all — its own comment reads *`// there is no "registrant" for wrapped names`* — and ENSv2 has no such split: `getOwnerV2` returns a single owner.

## Impact Details

### What goes wrong, and to whom

The defect needs no adversary. Alice holds an ENSv2 name and wants to send it to Bob. She types `bob.eth`, the name Bob goes by. Bob bought that name recently: he holds its registrar NFT, but he has not called `reclaim()`, so the registry controller is still the seller. The name carries no `addr` record — it did not have one when he bought it, and he cannot add one while the seller holds the controller. The app resolves `bob.eth` to the seller, shows Alice that address in the same words it uses for one taken from a genuine record, and the transfer completes. Bob never receives the name, and Alice cannot recover it, which `SendNameForm` states in its own warning banner directly above the field.

There is an obvious question here: why would Bob go by a name that resolves to nothing? Because Alice is sending him a name, not funds. Identifying the recipient of a name transfer does not require the recipient's own name to resolve anywhere. Bob is someone Alice knows, `bob.eth` is what he goes by, and the field accepts exactly that. A recently bought name that has not been configured yet is the ordinary case rather than a contrived one: in the sampling further down, condition B — no `addr` record — holds for 96.4% of Sepolia names that changed hands, and 11.6% on mainnet. What is rare is that condition coinciding with an unreclaimed controller, which the same section quantifies.

Nobody in that sequence did anything wrong. Alice typed the correct name. Bob owns his name and holds its NFT. The application chose the wrong address.

### What is lost

An ENS name, permanently. ENSv2 transfer is a plain ERC-1155 `safeTransferFrom` to the resolved address. There is no recall, no escrow and no confirmation step that surfaces the substitution. For a valuable name the loss is its full market value.

On the subname surface the damage is different in kind: the substituted address becomes the owner of a subname inside the victim's own name and holds every role on it. That is continuing authority in the victim's namespace rather than a single loss.

### A seller who does not reclaim can hold the condition open

The two preconditions are not only stumbled into. A seller can construct them and keep them open, without ever interacting with the party who ends up losing the name:

{% stepper %}
{% step %}

### Complete a sale without reclaiming

The seller lists an ENSv1 name and completes the sale. The registrar NFT moves to the buyer. The seller keeps the registry controller slot, because nothing in the protocol forces its release: `reclaim()` is the buyer's action and nobody prompts them.
{% endstep %}

{% step %}

### Keep the name without an `addr` record

Records authorise on the controller, so the buyer cannot publish an `addr` record until they reclaim. The seller can also clear an existing one.
{% endstep %}

{% step %}

### Let the buyer use the name as their handle

The buyer uses the name as their handle. It is their name and they hold the NFT, and as above, a name does not need to resolve anywhere to be usable as the recipient of a name transfer.
{% endstep %}

{% step %}

### Receive transferred ENS names

Anyone who sends that buyer an ENS name through the Explorer, typing the buyer's name in the recipient field, sends it to the seller instead.
{% endstep %}
{% endstepper %}

In that sequence the substituted address is attacker-controlled, which is the wording the second in-scope impact uses.

**What this does not give the attacker, stated plainly:** targeting. The seller cannot choose who sends to their former buyer, when, or how valuable the name will be. They can only create the condition and wait. That is opportunistic rather than aimed, and I would not describe it as hijacking in the usual sense. It matters mainly because it shows the state is not a transient accident that clears itself — it can be maintained deliberately, and the party maintaining it is the one who benefits.

The stronger case for severity is the one above it, where no attacker is needed at all.

### How often the preconditions coincide

I measured this on both chains rather than reasoning about it, and re-ran both runs shortly before filing so the figures below are dated rather than approximate.

The app is configured for **Sepolia only** ([`apps/portal/src/lib/wagmi.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/lib/wagmi.ts), `chains: [sepoliaWithEns]`), which is the deployment as it stands; mainnet is the production semantics the app is being built for. The population sampled on each chain is the one this finding needs: ENSv1 names whose registrar NFT changed hands, mints excluded, wrapped names counted separately because they take the NameWrapper branch and not this fallback.

**Sampled 2026-08-18 23:52 UTC.** Sepolia at block 11,518,380 over 45,000 blocks; mainnet at block 25,785,389 over 8,000 blocks. No read failures in either run.

|                                    |    Sepolia |    mainnet |
| ---------------------------------- | ---------: | ---------: |
| unwrapped names that changed hands |         28 |        250 |
| A. controller ≠ registrant         |   1 (3.6%) | 29 (11.6%) |
| B. no `addr` record                | 27 (96.4%) | 29 (11.6%) |
| A and B together                   |          0 |          0 |

An earlier mainnet window of 265 names, sampled the previous day, contained exactly one name in this state. So across roughly 515 mainnet names sampled in two independent windows, the joint condition was observed once.

Three things follow.

**The joint condition is rare, and rarer than the two individual rates suggest.** One occurrence in \~515 mainnet names is on the order of 0.2%. Multiplying the two rates would predict roughly seven occurrences in that many names, so the conditions are not independent — they are negatively correlated, by a wide margin. The likely reason is mechanical: marketplace and wrapper flows that move the NFT tend to move the controller with it, so a name that changed hands and still carries no `addr` record is disproportionately one whose controller moved too. I flag this because it is the first thing anyone checking the arithmetic will notice, and because it cuts against the finding rather than for it. I will not defend a sharper figure than "low fractions of a percent" from this data, and I am not going to convert it into a names-per-day rate, since that would be an extrapolation from a single observation.

**On Sepolia the structure is different.** Condition B holds for 96.4% of names that change hands there, which is unsurprising on a testnet where few people configure records. The joint condition on this deployment is therefore governed almost entirely by A, the unreclaimed controller, at 3.6% — the binding constraint is narrower than on mainnet even though the observed joint count is zero in a 28-name sample.

**Rarity is not the same as inapplicability.** The defect is deterministic: whenever both conditions hold, the app resolves to the controller every time, and the resulting transfer is irreversible. The measurements bound how often the state occurs by chance. They say nothing about the seller-constructed case described above, where the party who benefits creates the state deliberately rather than waiting for it.

### Not WEB-1305

The nearest known issue is WEB-1305, *"Attempting to transfer V2 name roles to a V1 name stays pending indefinitely; this transfer should be blocked in the UI"*. That describes a transfer that never completes. Here the transfer completes successfully and the asset lands at an address the user did not choose, which the Proof of Concept shows by reading ownership back on-chain after a mined receipt. The symptom and the consequence are therefore opposites. I am working from the one-line summary and have not seen that issue's root cause, so I am not claiming the underlying mechanisms differ — only that a fix for a transfer that hangs would not address a transfer that succeeds to the wrong address.

### Mapping to the selected impacts

**Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer)**, sub-case *"redirecting the recipient of a name transfer"*. The recipient of a name transfer is redirected to an address the sender did not choose, and the PoC carries it through to a completed transfer with ownership read back on-chain rather than stopping at the constructed calldata. Worth noting for scoping: the transfer flow exists only in the Explorer app, since the Manager app has no transfer route, so the listed sub-case "or transfer" can only refer to this surface.

**Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient**, sub-case *"resolving or rendering an ENS name to an attacker-controlled address"*. Both halves hold independently of the row above: the app resolves the name to the wrong address, then displays that address in the same wording it uses for a genuine `addr` record. This row does not require the substituted address to be under anyone's deliberate control, though the sequence above shows it can be.

## References

### The defect and the path it travels

* [`apps/portal/src/features/roles/helpers/addUser.handlers.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/roles/helpers/addUser.handlers.ts) — the owner fallback
* [`apps/portal/src/utils/ens/resolveEnsOwner.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/utils/ens/resolveEnsOwner.ts) — V2-then-V1 resolution; `ResolvedEnsOwner` drops `registrant`
* [`apps/portal/src/features/address/queries/getResolvedAddress.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/address/queries/getResolvedAddress.ts)
* [`apps/portal/src/features/address/hooks/useAddressResolution.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/address/hooks/useAddressResolution.ts) — *"Shared by every 'name or address' input in the app (transfer recipient, add-user sheets, …)"*
* [`apps/portal/src/features/transfer/components/SendNameForm.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/components/SendNameForm.tsx)
* [`apps/portal/src/features/transfer/hooks/useTransferName.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/hooks/useTransferName.ts)
* [`apps/portal/src/features/transfer/helpers/transferToken.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/helpers/transferToken.ts)
* [`apps/portal/src/routes/$name/create-subname.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/routes/%24name/create-subname.tsx) and [`features/registry/utils/create-subname.helpers.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/utils/create-subname.helpers.ts) — `DEFAULT_ROLE_BITMAP`
* [`apps/portal/src/features/address/components/AddressNameInput.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/address/components/AddressNameInput.tsx) — the status line that marks no provenance

### Where the same codebase gets the distinction right

* [`apps/portal/src/features/migration/hooks/useMigrationStatus.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/migration/hooks/useMigrationStatus.ts)
* [`apps/portal/src/features/registry/components/v2/MigrateRegistryPrompt.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/registry/components/v2/MigrateRegistryPrompt.tsx)
* [`apps/portal/src/routes/addr/$addr/names.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/routes/addr/%24addr/names.tsx)

### Dependency

`@ensdomains/ensjs` is pinned in [`pnpm-workspace.yaml`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/pnpm-workspace.yaml) to a `pkg.pr.new` build at commit `e96662c`; its `package.json` reports `5.0.0-alpha.1`. The registrant/controller return shape is in `src/actions/public/v1/getOwner.ts`, and its own comments are the clearest statement of the two roles: *"the registrant is the owner of the NFT"* and *"the owner is the controller of the records"*.

## Link to Proof of Concept

<https://gist.github.com/Ispolin11/a514ecf0d3fa18e95cb488740111c463>

## Proof of Concept

Four runnable artefacts. Each one drives the application's own modules against a fork of Sepolia pinned to a fixed block, so every address and ownership outcome below reproduces identically rather than approximately.

{% hint style="info" %}
**Scope of the demonstration, stated up front.** These run the shipped resolution and transaction-building code, not the rendered application. The full end-to-end stack (`pnpm --filter @ens-apps/e2e infra:up`) needs Docker, which was not available to me, so I did not drive the browser UI. What that costs and what it does not: every module in the chain — `resolveEnsOwner`, `resolveAddressOrName`, `getResolvedAddress`, `prepareCreateSubnameTransaction`, ensjs, viem — is the shipped code, unmodified. The display half is covered by rendering the shipped `AddressNameInput` component itself (step 5). The only substitution anywhere is `safeGetClient`, whose entire job is to hand wagmi's client to `getEnsOwner`; it is repointed at the fork.
{% endhint %}

## Environment

Run once, before any step.

```bash
anvil --fork-url https://lb.drpc.live/sepolia/AnmpasF2C0JBqeAEzxVO8aRo7Ju0xlER8JS4QmlfqV1j \
      --fork-block-number 11517150 --port 8545
```

```bash
# REPO = a checkout of immunefi-team/audit-comp-ens @ audit-comp-ready,
# with `pnpm install` already run. No .env is needed — verified with
# apps/portal/.env absent.
REPO=/path/to/audit-comp-ens
```

**Both parts of that anvil command matter.**

**The endpoint.** Pinning to a block only works against an archive node. `ethereum-sepolia-rpc.publicnode.com` keeps only a short recent window — probing it directly, state 100 blocks back is still served and state 300 blocks back is already gone, so the usable window is well under an hour. Past it, `anvil --fork-block-number` aborts at startup with *"failed to create genesis … state at block #N is pruned"*, before any test code runs. The URL above is the audited app's own Sepolia RPC ([`apps/portal/src/lib/wagmi.ts#L25`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/lib/wagmi.ts#L25)), hardcoded in browser-bundled source and allowlisted in the app's own CSP, so it is public by construction, and it serves archive state.

**The block.** The tests read the live Sepolia names `demo.eth` (v1) and `hello.eth` (v2). If either changes hands during the competition, the setup assertions fail and a working PoC reads as a broken one. At block **11,517,150** the starting state is:

```
demo.eth   registrant 0xD613cb815902a1A6C469b899a18aB9B1901E8c30
           controller 0xD613cb815902a1A6C469b899a18aB9B1901E8c30
           resolver   0x8948458626811dd0c23EB25Cc74291247077cC51
hello.eth  ENSv2 owner 0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8
```

Registrant and controller are the *same* address here — that is the pristine state. Each test then simulates the sale that separates them.

**If that endpoint ever stops working** — pruning, key rotation, quota — drop `--fork-block-number`. Each test carries a drift guard that prints the mismatch and the exact command, and the setup assertions still hold for as long as `demo.eth` has not changed hands.

**On the console output quoted below.** Each block is trimmed to the decisive lines. Omitted from them are the transaction hashes and the address of the subregistry deployed in step 3, which differ on every run, and the ENSv2 token id, which is a 77-digit number. Nothing is altered; a real run prints these blocks plus those lines.

**Cast of addresses used throughout:**

|        | Address                                      | Role                                                  |
| ------ | -------------------------------------------- | ----------------------------------------------------- |
| Alice  | `0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8` | owns the ENSv2 name `hello.eth`, the asset being sent |
| Bob    | `0x70997970C51812dc3A010C7d01b50e0d17dc79C8` | registrant of `demo.eth` — the party Alice means      |
| Seller | `0xD613cb815902a1A6C469b899a18aB9B1901E8c30` | v1 registry controller of `demo.eth`                  |

{% stepper %}
{% step %}

## The transfer completes and the name lands at the wrong address

```bash
cp portal-recipient-owner-fallback.fork.test.ts "$REPO/apps/portal/"
cd "$REPO" && pnpm --filter portal exec vitest run portal-recipient-owner-fallback.fork.test.ts
```

The test sets up its own chain state first, reproducing an ordinary secondary-market sale of `demo.eth`: `BaseRegistrar.transferFrom` moves the registrar NFT to Bob, the controller slot is left with the seller, and the resolver is cleared so the name has no `addr` record. It then has Alice transfer her ENSv2 name `hello.eth`, typing `demo.eth` in the recipient field.

```
fork state for demo.eth (the recipient name Alice types):
  registrant / NFT holder (the real owner) : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
  v1 registry controller  (the seller)     : 0xD613cb815902a1A6C469b899a18aB9B1901E8c30
  resolver                                 : 0x000…000 (no addr record)

✓ 1. the v2 lookup misses, so the v1 branch is genuinely reached
  resolveEnsOwner("demo.eth") -> 0xD613cb815902a1A6C469b899a18aB9B1901E8c30 (ENSv1)

✓ 2. resolveAddressOrName returns the controller as the recipient
  Alice types            : demo.eth
  she means (registrant) : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
  app resolves to        : 0xD613cb815902a1A6C469b899a18aB9B1901E8c30  <- the seller

✓ 3. the transfer completes and the name ends up owned by the wrong address
  Alice transfers        : hello.eth  (ENSv2)          <- token id trimmed
  owner before           : 0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8
  owner AFTER            : 0xD613cb815902a1A6C469b899a18aB9B1901E8c30
  intended recipient     : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

Tests  3 passed (3)
```

`owner AFTER` is read back through ensjs `getOwner` after a mined receipt. The asset has moved to an address the user never chose.

The three assertions are ordered deliberately. Assertion 1 rules out the possibility that the V2 registry answers first and the v1 branch is unreachable. Assertion 2 rules out `getEnsAddress` throwing rather than returning `null`, which would make `resolveAddressOrName` fail closed and never reach the fallback. Only then does assertion 3 carry the transfer through.
{% endstep %}

{% step %}

## Reset the fork between tests

```bash
cast rpc anvil_reset \
  '{"forking":{"jsonRpcUrl":"https://lb.drpc.live/sepolia/AnmpasF2C0JBqeAEzxVO8aRo7Ju0xlER8JS4QmlfqV1j","blockNumber":11517150}}' \
  --rpc-url http://127.0.0.1:8545
```

It prints `null` on success — that is the expected output, not a failure.

Do this after each of steps 1 and 3. Every test mutates the same fork, and step 4 opens by asserting a pristine one. Running the three back to back against an un-reset node fires that guard and skips all seven of its cases, which reads as a broken PoC when it is the guard working as designed.
{% endstep %}

{% step %}

## The same defect assigns ownership of a subname

```bash
cp portal-subname-owner-fallback.fork.test.ts "$REPO/apps/portal/"
cd "$REPO" && pnpm --filter portal exec vitest run portal-subname-owner-fallback.fork.test.ts
```

This is the second ownership-assigning consumer, reached from a different route. The test reproduces the full portal flow: deploy a subregistry through `deploySubregistryWriteParameters`, attach it with `setSubregistryWriteParameters` — exactly what `ConfigureRegistryForm` does — then build the create call with the app's own `prepareCreateSubnameTransaction` and send its bytes.

```
Alice owns             : hello.eth
she creates            : pay.hello.eth
Owner field, she types : demo.eth
she means (registrant) : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
app resolved it to     : 0xD613cb815902a1A6C469b899a18aB9B1901E8c30

OWNER OF THE SUBNAME   : 0xD613cb815902a1A6C469b899a18aB9B1901E8c30
roles granted to it    : 0x1111111111111111111111111111111111111111111111111111111111111111 (every role)

Tests  1 passed (1)
```

A subname inside Alice's own name is now owned, with every role, by the seller of an unrelated v1 name.
{% endstep %}

{% step %}

## Adversarial matrix, including a negative control

```bash
cp portal-recipient-owner-adversarial.fork.test.ts "$REPO/apps/portal/"
cd "$REPO" && pnpm --filter portal exec vitest run portal-recipient-owner-adversarial.fork.test.ts --fileParallelism=false
```

Seven cases written to falsify the finding rather than to confirm it. Every setup write is mined and its receipt asserted `success` — `writeContract` alone returns a hash and hides an on-chain revert — and the on-chain state is read back and asserted **before** the application is asked anything, so an app answer can never be confused with a setup that silently failed. Resolution goes through `getResolvedAddress`, the query layer the UI calls, not the inner helper.

| #      | What it tries to prove wrong                                                             | Result                                                                                                                                               |
| ------ | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **C1** | That this is just a stale `addr` record, an ENS-wide issue rather than an app defect     | With the seller's stale record intact, the app returns **the addr record**, not the fallback. The finding is correctly scoped and does not overreach |
| **C2** | **Negative control** — that the harness returns the controller regardless of chain state | With the `addr` record pointed at Bob and the controller still the seller, the app returns **Bob**. The harness is not rigged                        |
| **C3** | That the finding depends on the artificial `resolver = 0` setup used in step 1           | It does not. Resolver *set*, `addr` empty — the ordinary "no ETH address configured yet" state — still falls back to the controller                  |
| **C4** | Resolver unset entirely                                                                  | Falls back to the controller                                                                                                                         |
| **C5** | That the recipient can simply fix this themselves                                        | **No — `mined-reverted`.** Bob cannot write his own `addr` record while the seller holds the controller. Records authorise on the registry owner     |
| **C6** | That the UI gate rejects the name before any resolution happens                          | `isNameOrAddress('demo.eth') === true` — the resolver is genuinely called                                                                            |
| **C7** | That some further recipient validation exists                                            | `hasValidRecipient = !!recipient && !isSelf && !isZeroAddress`, and the file contains no reference to `registrant`                                   |

```
Tests  7 passed (7)
```

C2 is the one that matters most for trusting the rest: if the harness were simply returning the controller for everything, C2 would fail.
{% endstep %}

{% step %}

## The UI marks no provenance

No chain required.

```bash
cp portal-recipient-owner-display.test.tsx "$REPO/apps/portal/"
cd "$REPO" && pnpm --filter portal exec vitest run portal-recipient-owner-display.test.tsx
```

Renders the shipped `AddressNameInput` twice with resolutions that differ only in where the address came from.

```
address from a genuine addr record : "Resolved: 0x7099…79C8"
address from the owner fallback    : "Resolved: 0xD613…8c30"
same CSS classes                   : true

a raw address the user typed       : "Using address: 0xD613…8c30"

Tests  2 passed (2)
```

The second assertion is the useful one: the component already varies its wording by provenance when it chooses to. There is simply no third case for a fallback-derived address.
{% endstep %}
{% endstepper %}

### What varies between runs, and what does not

Transaction hashes and the address of the subregistry deployed in step 3 differ on every run — they depend on nonces and timing. Everything the finding rests on is fixed by the pinned block: the registrant, the controller, the resolved address, the owner before and after the transfer, and the owner of the created subname.

The prevalence figures quoted in the Description come from a separate read-only script and are deliberately **not** pinned: it samples a live window on mainnet and Sepolia, so re-running gives numbers of the same order rather than the same numbers. The dated figures in the Description name the exact blocks they were taken at.

### Files

All four are in the secret Gist linked with this submission.

| File                                              | Step | What it establishes                                             |
| ------------------------------------------------- | ---: | --------------------------------------------------------------- |
| `portal-recipient-owner-fallback.fork.test.ts`    |    1 | resolution → signed transfer → ownership read back on-chain     |
| `portal-subname-owner-fallback.fork.test.ts`      |    3 | the same defect on a second ownership-assigning surface         |
| `portal-recipient-owner-adversarial.fork.test.ts` |    4 | seven falsification attempts, including a negative control      |
| `portal-recipient-owner-display.test.tsx`         |    5 | the substitution is indistinguishable in the rendered component |

Last run together on the pinned block, with `apps/portal/.env` absent: 13 tests, all passing (3 + 1 + 7 + 2).
