# 92944 w a critical recipient resolution returns a different address than every conformant ens client sending an irreversible name transfer to whoever registered a look alike name

**Submitted on Sep 13th 2026 at 11:19:20 UTC by @Magnum for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92944
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer's recipient field resolves a pasted name to a different address than every conformant ENS client resolves the same string to. Measured on the repository's own local fork: one string, two destinations — `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` in any conformant client, and `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266` in this application. The second address belongs to whoever registered a look-alike of the name.

This is a transaction path, not a display one. The address the divergence produces is the destination of an irreversible ERC-1155 name transfer and of registry role grants, on a resolver whose own doc comment states it is shared by every name-or-address input in the application. The attacker's on-chain cost is one name registration, and no resolver or record configuration is needed.

## Vulnerability Details

### The divergence

`apps/portal/src/features/roles/helpers/addUser.handlers.ts`

```ts
export async function resolveAddressOrName({
  client,
  nameOrAddress,
}: ResolveAddressOrNameParams): Promise<Address | null> {
  if (isAddress(nameOrAddress, { strict: false })) {
    return nameOrAddress as Address
  }
  try {
    const resolved = await getEnsAddress(client, {
      name: nameOrAddress,
      universalResolverAddress,
    })
    let resolvedAddress = resolved
    // Fallback for names that do not set an address record:
    // use current ENS owner address so the role can still be granted.
    if (!resolvedAddress) {
      const ownerResult = await getEnsOwner({ name: nameOrAddress })
      resolvedAddress = ownerResult.value?.owner ?? null
    }
    return resolvedAddress
  } catch {
    return null
  }
}
```

Two properties produce the divergence.

`name: nameOrAddress` is the string the user typed, passed through unchanged. viem's `getEnsAddress` resolves whatever node that string hashes to; its own doc block states that ENS names have validation rules and that callers should apply the built-in `normalize` first. This application does not, so the node resolved is the look-alike's own node rather than the node the user intended.

The fallback then removes the attacker's remaining work. Written for role granting — "so the role can still be granted" — it returns the node's **owner** when the node carries no address record. The attacker therefore needs only to register the look-alike. No resolver deployment, no `setAddr`, no configuration.

### The path is shared by every name-or-address field

`apps/portal/src/features/address/hooks/useAddressResolution.ts:53-61`

```ts
const isValid = trimmed.length > 0 && isNameOrAddress(trimmed)
...
  ...getResolvedAddressQueryOptions({ nameOrAddress: debounced }),
```

`trimmed` is the raw input with surrounding whitespace removed. It is validated, then that same string is what gets resolved.

The hook's own doc comment: "Shared by every 'name or address' input in the app (transfer recipient, add-user sheets, …)."

Reached this way: `features/transfer/components/SendNameForm.tsx` — the recipient of an ERC-1155 name transfer — and the registry role sheets.

### Input validation does not stop it

`isNameOrAddress` uses `ens_normalize`. A look-alike normalizes cleanly; that is what makes it a look-alike. Validation therefore succeeds, and the un-normalized form proceeds to resolution. A stricter input filter would not help, because the string is valid — the defect is that the validated form is not the form that gets resolved.

### What the victim is shown

`features/transfer/components/SendNameForm.tsx:253-283` renders `RecipientPreview`: the resolved address in full (`truncate={false}`) with the address's primary name above it. The address is visible, and I say so plainly. Two things qualify that. A recipient field displays a name precisely so users need not compare hexadecimal. And the primary name shown beside the address is the reverse record of the attacker's own address, which the attacker also writes.

### Root cause, stated for completeness

The underlying reason the two forms differ is ENSIP-15: the specification exists because visually identical strings hash to different nodes, and it defines the canonical form every client must reduce input to before hashing. This application applies that reduction on its input-validation paths and not before resolution. That is one step in producing the divergence, not the finding itself — the finding is that the destination of a transaction differs from what every other client computes for the same input.

## Impact Details

Selected impact: causing the application to display or pre-fill an incorrect address, name, or transaction detail so that a user submits a transaction to an unintended recipient — specifically "**resolving** or rendering an ENS name to an attacker-controlled address".

This is the resolving half of that clause, established by measurement:

|                            |                                                  |
| -------------------------- | ------------------------------------------------ |
| string pasted              | `ｖitalik.eth` (U+FF56), renders as `vitalik.eth` |
| namehash of what was typed | `0x9fead700…`                                    |
| namehash after ENSIP-15    | `0xee6c4522…`                                    |
| conformant client sends to | `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`     |
| this application sends to  | `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`     |
| that address is            | the account that registered the look-alike       |
| attacker cost              | one registration, no records configured          |

Consequences by surface:

* **Name transfer** (`SendNameForm`): an ERC-1155 transfer is irreversible. The loss is the full value of the transferred name, delivered to an account of the attacker's choosing.
* **Registry roles** (add-user sheets): a role grant identifies its target by this same resolved address. Granting `ROLE_SET_RESOLVER` or `ROLE_SET_SUBREGISTRY` to the wrong account hands control of that name's resolution to the attacker.

No privileged position is required. The victim's action is ordinary — pasting a name into the recipient field — and names are pasted from chat, listings, comments and social posts, which is where a look-alike is planted.

I claim **High** rather than Critical for one honest reason: the resolved address is rendered before the victim confirms. The argument for Critical is that the impact text names resolving to an attacker-controlled address and sits in the Critical group; that the field shows a name so users need not verify hex; and that the name shown beside the address is itself attacker-authored. I leave the grading to the project.

## References

* `apps/portal/src/features/roles/helpers/addUser.handlers.ts` — `resolveAddressOrName`
* `apps/portal/src/features/address/queries/getResolvedAddress.ts`
* `apps/portal/src/features/address/hooks/useAddressResolution.ts` — lines 53, 61, and the shared-by-every-input doc comment
* `apps/portal/src/features/transfer/components/SendNameForm.tsx` — lines 253-283
* `apps/portal/src/utils/token/isNameOrAddress.ts`
* `node_modules/.pnpm/viem@*/node_modules/viem/_esm/actions/ens/getEnsAddress.js` — the doc block telling callers to normalize
* ENSIP-15 (name normalization)
* Tested at `1c9b47f`, HEAD of `audit-comp-ready`, against the repository's own `pnpm e2e:infra:up` fork (chain id 11155111)

## Link to Proof of Concept

<https://gist.github.com/Magnat79A/120bdd3accce60ee84a134e0b23a564b>

## Proof of Concept

## Overview

One idempotent script, run against the repository's own local fork. It performs the attacker's setup — registering a look-alike name — then measures the destination address this application resolves for a pasted string against the destination a conformant ENS client resolves for the same string.

No mocks. The resolution function is reproduced verbatim from `apps/portal/src/features/roles/helpers/addUser.handlers.ts` so the script runs standalone; every read and write goes to the fork.

All results below are from a clean run at revision `1c9b47f`. Files are in the linked gist.

## Step 1 — Setup

1. Clone the scope repository and check out `audit-comp-ready`
2. `pnpm install --frozen-lockfile`
3. `pnpm e2e:infra:up` — Anvil fork of Sepolia at chain id 11155111, so contract addresses match the deployed ones
4. Copy `poc-c06-full.mjs` to `apps/portal/`

## Step 2 — Run

cd apps/portal && node poc-c06-full.mjs

Anvil account #0 acts as the attacker. If the look-alike is already registered to it, setup is skipped and the script goes straight to the measurement.

## Step 3 — Output

STEP 1 - the two strings the victim pastes : ｖitalik.eth it reads on screen as : vitalik.eth ENSIP-15 maps it to : vitalik.eth namehash of what's typed : 0x9fead700dabc1d932c1bd9480dae3485f0d2a7059c1807db096829571d5fc29f namehash after ENSIP-15 : 0xee6c4522aab0003e8d14cd40a6af439055fd2577951148c14b6cea9a53475835

STEP 2 - attacker setup: register the look-alike owner of vitalik.eth : 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 (v1/nameWrapper) look-alike now owned by : 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 (v2) attacker address : 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

STEP 3 - what an ENSIP-15-conformant client resolves normalizes the input to : vitalik.eth address record : 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 owner : 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 (v1/nameWrapper) destination : 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045

STEP 4 - what this application resolves for the same input passed to the resolver : ｖitalik.eth (raw, un-normalized) destination : 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

Summary object printed by the script:

```json
{
  "victimPasted": "ｖitalik.eth",
  "readsOnScreenAs": "vitalik.eth",
  "namehashOfWhatWasTyped": "0x9fead700dabc1d932c1bd9480dae3485f0d2a7059c1807db096829571d5fc29f",
  "namehashAfterEnsip15": "0xee6c4522aab0003e8d14cd40a6af439055fd2577951148c14b6cea9a53475835",
  "conformantClientSendsTo": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
  "thisApplicationSendsTo": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "divergent": true,
  "lookAlikeOwnedBy": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "destinationIsTheLookAlikeOwner": true,
  "realNameOwner": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
  "attackerCost": "one registration",
  "cause": "resolveAddressOrName passes the raw input to viem getEnsAddress, which does not normalize (its own doc comment says callers should). The look-alike node is resolved; with no address record on it, the owner fallback returns whoever registered it."
}
```

Note that the conformant side resolves through the real name's **address record**, not through a fallback. Both sides being compared are full resolutions, and they disagree.

## Step 4 — Confirm the dependency leaves this to the caller

grep -rn "normalize" node\_modules/.pnpm/viem@\*/node\_modules/viem/\_esm/actions/ens/getEnsAddress.js | head -3

Returns viem's own doc block: ENS names have validation rules, and callers likely want to apply the built-in `normalize` before passing a name to `getEnsAddress`. The application does not.

## Step 5 — Confirm the path is shared

grep -rn "useAddressResolution|getResolvedAddressQueryOptions" --include=.ts --include=.tsx apps/portal/src | grep -v ".test." sed -n '40,65p' apps/portal/src/features/address/hooks/useAddressResolution.ts

The hook validates `trimmed` with `isNameOrAddress`, then passes that same unnormalized string to the resolution query. Its doc comment names the transfer recipient and the add-user sheets as consumers and states it is shared by every such input.

## Step 6 — Confirm validation does not stop it

`isNameOrAddress` uses `ens_normalize`. A look-alike normalizes cleanly, so validation returns true and the raw form proceeds to resolution. The string is valid; what differs is which node it hashes to.

## Counter-arguments, stated up front

* **"The user can see the resolved address."** True, and the report says so. `RecipientPreview` renders it in full. The argument is that a recipient field displays a name precisely so users do not compare hex, and the name displayed beside the address is the attacker's own reverse record.
* **"This is the rendering issue already on file."** Opposite direction. A rendering issue takes an address and produces a name to display. This takes a name and produces the address a transaction is sent to. Different function, different file, and the measured artefact is two destination addresses rather than one string in the DOM.
* **"An attacker must plant the string."** Ordinary: names are pasted from chat, listings, comments and social posts. The only on-chain cost is one registration, and the owner fallback means no records need configuring.
* **"The contracts should reject such names."** ENS contracts are byte-transparent by design. viem's documentation places normalization on the caller, and this repository already applies it on input validation — just not before resolution.
* **"Only a contrived character does this."** U+FF56 is an ordinary fullwidth Latin letter rendering as `v` in every proportional font.

## Artifacts (secret gist, linked above)

* `poc-c06-full.mjs` — setup plus measurement, one run
* `12-c06-full.txt` — its output, md5 `724edb23e852bde6d251123fd4c132c9`
* `poc-c06-resolve.mjs` — measurement only
* `11-c06-resolve.txt` — its output, md5 `15082072edc1c007f5fcbd1c50d55bfc`

## Note on revision

Tested at `1c9b47f`, HEAD of `audit-comp-ready`. The Resources page names `63772fd8…` (commit `cda79ac`, two commits earlier — a dependabot bump merged before the competition opened). The finding is present at both; none of the cited files are touched by that bump, and `git fetch` confirms the branch has not moved during the competition.
