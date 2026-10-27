# 91645 w a critical wrong field ens ownership resolution in the explorer s recipient input sends name transfers to a v1 name s former controller instead of its owner

**Submitted on Sep 3rd 2026 at 09:38:44 UTC by @nomorecaffeine for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91645
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

When a user types an ENS name into any "ENS name or address" input in the Explorer and that name has no `addr` record, the app falls back to an ownership lookup. For an unwrapped `.eth` second-level name that lookup returns the **wrong address**: the v1 registry *controller*, not the name's actual owner. The app then displays it as a normal resolution result and uses it as the recipient of a name transfer.

ENS v1 stores two separate addresses for such a name. `BaseRegistrar.ownerOf()` holds the **registrant**, who owns the NFT. `Registry.owner()` holds the **controller**, who manages records. `BaseRegistrar.transferFrom` moves only the registrant and leaves the controller with the seller until the buyer calls `reclaim()`. That gap is routine after any sale or transfer.

ensjs hands the app both values plus a discriminator saying which is which:

```tsx
// @ensdomains/ensjs/dist/actions/public/v1/getOwner.js:108-112
// the registrant is the owner of the NFT
// the owner is the controller of the records
if (registrarOwner) {
    return { registrant: registrarOwner, owner: registryOwner, ownershipLevel: 'registrar' };
}
```

The app keeps `.owner` and throws the other two away:

```tsx
// apps/portal/src/utils/ens/resolveEnsOwner.ts:95-102
const v1Owner = await getOwnerV1(client, { name })
if (v1Owner?.owner && v1Owner.owner !== zeroAddress) {
  return { owner: v1Owner.owner, registryAddress: v1EthRegistry, protocolVersion: 'ENSv1' }
}
```

This repository already documents the correct convention elsewhere. `apps/portal/src/utils/names/mergeNamesData.ts:88-91` says *"For unwrapped names: registrant is Owner, registry owner is Manager"* and uses `registrant`. So `/addr/<addr>/names` lists the holder as Owner while `/<name>` lists the controller.

`resolveEnsOwner` backs `resolveAddressOrName` → `getResolvedAddress` → `useAddressResolution`, the shared recipient input. It has five consumers, and **four pass no `resolvedContent`**, so `AddressNameInput.tsx:62-68` renders only `Resolved: 0x1234…5678`, six leading and four trailing hex characters, with no name to compare against what the user typed. Rendering the shipped component with the address this bug produces gives, verbatim:

```
RENDERED TEXT -> "Resolved: 0x5A38…0615"
```

That is the same wording a genuine `addr` record resolution produces, so the two are indistinguishable to the user. The attached `poc-render-probe.test.tsx` reproduces it offline in one command.

| consumer                                                      | what the user sees                           |
| ------------------------------------------------------------- | -------------------------------------------- |
| `routes/$name/create-subname.tsx:271-283`                     | `Resolved: 0x1234…5678`                      |
| `features/roles/components/RolesAddUserSheet.tsx:73`          | `Resolved: 0x1234…5678`                      |
| `features/resolver/components/ResolverAddUserSheet.tsx:141`   | `Resolved: 0x1234…5678`                      |
| `features/registry/components/v2/RegistryAddUserSheet.tsx:66` | `Resolved: 0x1234…5678`                      |
| `features/transfer/components/SendNameForm.tsx:81`            | full address, plus any verified primary name |

**Impact.** The recipient is never re-derived after the form: `useTransferName.ts:111-115` snapshots it and `:150-163` passes it straight to `transferToken.ts:70`'s `safeTransferFrom`. A user who sends a name to `bob.eth` sends it to whoever last held the controller slot. On `create-subname` the same address receives `DEFAULT_ROLE_BITMAP`, every role on the new subname. The loss is permanent: recovery needs the unintended recipient to send it back.

No attacker is required. The wrong recipient is usually an uninvolved former owner, so this fires against ordinary users with nobody acting maliciously. An attacker who is the stale controller gains no capability they lacked, since they could already set an `addr` record; the defect is that the app invents a recipient no on-chain record supports and presents it as a resolution.

## Caller authorization

Any ordinary externally owned account can create the precondition, and the victim is an ordinary user doing an ordinary thing. No privileged role, admin key, leaked credential or compromised infrastructure appears anywhere on this path, and the attacker never touches the victim's wallet or session. The victim only has to use the recipient field the way it is meant to be used.

## Fix

Switch on `ownershipLevel` and return `registrant` for the `registrar` level. Treat `registrant === null` as expired. Separately, do not label an ownership-derived address `Resolved:`.

## Link to Proof of Concept

<https://gist.github.com/nguyen-xuan-quoc/9af801fff29455fbe60042f3058bb91e>

## Proof of Concept

Below is the step-by-step reproduction. Every step lists the value the application actually produced.

**Everything runs against a local `anvil` fork. No transaction is sent to any public testnet or mainnet, and the live application is never touched.** The only network traffic to Sepolia is the read-only RPC that anvil uses to populate fork state.

An executable version of these same steps is in a secret Gist: <https://gist.github.com/nguyen-xuan-quoc/9af801fff29455fbe60042f3058bb91e> (`poc-registrant-vs-controller.test.ts` for the chain steps below, and `poc-render-probe.test.tsx` for the rendering claim).

It is not a unit test over a mocked function. It forks the chain, calls the shipped `resolveEnsOwner` unmodified with nothing stubbed, and in step 6 it broadcasts a real state-changing transaction and reads the resulting owner back off chain. Vitest is only the runner, chosen because the target is a TypeScript application. It passes 6/6 on the first run and prints every value shown below.

**Requirements**

* Node 22 and pnpm 10, plus Foundry for `anvil`
* The repository at `audit-comp-ready`, with dependencies installed: `pnpm install --frozen-lockfile`
* No environment variables, API keys or config files are needed. The RPC URL is hardcoded in the test and the fixture addresses are constants at the top of the file.

**Setup**

```
anvil --fork-url <https://ethereum-sepolia-rpc.publicnode.com> --port 8545 --silent &
```

Do not pin `--fork-block-number`: step 6 needs account state that public RPCs prune at older heights.

Copy the file to `apps/portal/src/utils/ens/poc-registrant-vs-controller.test.ts`, then:

```
pnpm --filter portal exec vitest run src/utils/ens/poc-registrant-vs-controller.test.ts
```

**Steps**

1. **Reproduce the routine v1 state.** The test impersonates the registrant of `matoken.eth` and calls `BaseRegistrar.transferFrom` to a buyer. This moves the NFT and leaves the controller behind, exactly as a real sale does. It then reads both slots back: registrant is now the buyer `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`, controller is still the seller `0x5A384227B65FA093DEC03Ec34e111Db80A040615`.
2. **Confirm ensjs supplies the discriminator.** `getOwner` returns `{ registrant: 0x7099…79C8, owner: 0x5A38…0615, ownershipLevel: "registrar" }`. Both fields and the discriminator are present.
3. **Trigger the defect.** `resolveEnsOwner(client, 'matoken.eth')` returns `owner: 0x5A38…0615`, the seller. The buyer, who owns the name, is not returned.
4. **Confirm the fallback is reached.** The app's own forward lookup, viem's `getEnsAddress` with the app's `universalResolverAddress`, returns `null`, so `resolveAddressOrName` takes the ownership branch.
5. **Confirm the value reaches the transaction.** Replaying `resolveAddressOrName` and encoding as `transferToken.ts:70` does yields calldata `0xf242432a…` with `0x5a384227b65fa093dec03ec34e111db80a040615` in the `to` position.
6. **Execute it end to end.** The test reads the real v2 owner of `hello.eth` (`0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8`), impersonates them, **sends** the transfer to the address the app resolved, and reads the owner back:

```
transfer tx status -> success
new owner of hello.eth -> 0x5A384227B65FA093DEC03Ec34e111Db80A040615
```

The name is now held by the controller of `matoken.eth`, who was never the intended recipient and does not own that name.

**Expected:** the name arrives at the holder of `matoken.eth`. **Actual:** it arrives at the address that last held the controller slot.

**Value at risk.** Each occurrence loses one ENS name in full, since the unintended recipient holds it outright and only they can return it. The amount is the market value of the name transferred, which is set by the name rather than by this bug, so no single figure covers it. On the `create-subname` path the loss is instead a new subname issued to the wrong party with every role on it.
