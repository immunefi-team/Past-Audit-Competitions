# 92905 w a low live resolver access refetch changes replace continue into a transaction to the previous owner s resolver

**Submitted on Sep 13th 2026 at 05:56:43 UTC by @Ailenia for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92905
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

I found that Manager treats permission for one ETH-address write as control of the entire resolver. A malicious previous owner can retain resolver-wide administration and temporarily grant that narrow permission. If it becomes writable while the confirmation is open, "Replace & Continue" switches from deploying a victim-controlled resolver to calling `setAddr` on the previous owner's resolver. The attacker can then revoke the permission and redirect the name.

## Vulnerability Details

The affected asset is `audit-comp-ens/apps/manager` at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

`audit-comp-ens/apps/manager/src/features/profile/service/resolverWriteAccess.ts:21-81` reduces resolver control to one Boolean. It builds the exact ETH-address setter and returns `true` whenever a call from the current owner would succeed:

```typescript
await publicClient.call({
  account: ownerAddress,
  to: resolverAddress,
  data: encodeFunctionData({ abi, functionName, args }),
})
return true
```

For a `PermissionedResolver`, successful name-scoped `ROLE_SET_ADDR` says only that the account may write one record. It does not establish that the account owns or administers the resolver, can revoke other users, or controls its root scope. A previous owner may retain root and admin roles while granting the recipient only this narrow, revocable permission.

`audit-comp-ens/apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx:418-432` consumes the live query result directly:

```typescript
const resolverBlocked = resolverWriteAccess.data === false
```

When it is `false`, clicking "Set as Primary" opens `ResolverSetupConfirmDialog`. The primary-name copy at `audit-comp-ens/apps/manager/src/features/profile/components/dialogs/ResolverSetupConfirmDialog.tsx:27-36` promises:

> This name still has profile details from a previous owner. To use it as your primary name, we'll clear those details and keep only your wallet address.

The dialog does not bind the checked resolver, access result, or intended operation to an immutable pending action. The write-access query remains mounted and stale while the nested confirmation is open. A normal focus event refetches it.

`runConfirm` at `ChoosePrimaryNameDialog.tsx:449-471` reads the current Boolean rather than the value that caused the confirmation:

```typescript
if (resolverBlocked) {
  await setupResolverMutation.mutateAsync()
} else if (needsEthAddressUpdate) {
  await updateEthAddressMutation.mutateAsync()
}
```

A `false -> true` transition silently changes the operation behind the unchanged confirmation. The represented safe branch calls `setupControlledResolver`, which selects an owner-controlled resolver, updates the registry pointer, and writes records atomically (`audit-comp-ens/apps/manager/src/features/profile/service/setupControlledResolver.ts:41-47,108-133`).

The unsafe branch instead calls `saveRecords` with the old resolver. `audit-comp-ens/apps/manager/src/features/profile/service/profileRecordTransactions.ts:372-456,520-566` submits that write through the transaction manager.

An attacker does not need access to the victim's browser or credentials. A realistic sequence is:

1. Attacker A owns `victim.eth` and administers official `PermissionedResolver` M with resolver-wide root/admin roles.
2. A transfers the name token to victim V while leaving M attached. A's resolver administration is independent of registry-token ownership and survives the transfer.
3. V uses the normal Manager "Choose Primary Name" flow. With no role for V on M, the preflight is `false`, so Manager presents the previous-owner cleanup confirmation.
4. A grants V only name-scoped `ROLE_SET_ADDR`. A controls this transition but cannot observe the modal; a normal focus refetch creates the demonstrated confirmation race.
5. The dialog remains unchanged. V clicks "Replace & Continue" and approves the wallet request.
6. Manager submits `setAddr` from V to M and submits no controlled-resolver setup or registry `setResolver` call.
7. A observes the public record write, revokes V's narrow role, and writes an attacker-controlled address. V still owns the registry token, but the active resolver remains administered by A until V separately discovers and replaces it.

My PoC executes steps 1-7 locally. The captured production request has `from = V`, `to = M`, and function `setAddr`. The controlled-resolver setup call count is zero, and A's existing `url` text record remains instead of being cleared. Official ENSv2 contracts then allow A to revoke V and redirect resolution. My final control proves that V can recover by changing the registry resolver.

## Impact Details

I select the project-authored Critical impact "Malicious interactions with an already-connected wallet," including modifying transaction arguments or parameters and substituting contract addresses.

The victim authorizes a confirmation that promises to remove the previous owner's profile state and retain only the victim's wallet address. After that confirmation is formed, the app substitutes both the target and operation. The wallet receives a transaction to the previous owner's resolver, while the controlled-resolver deployment and registry update are omitted. My PoC executes the substituted transaction through the production transaction manager and victim signer.

## Existing Safeguards and Counterarguments

* **"The attacker already controls the active resolver."** The confirmation exists to remove that prior-owner state. The defect replaces the confirmed safe branch with a transaction to that resolver and leaves attacker-authored text active. My PoC proves the changed target, operation, and post-state.
* **"The attacker cannot see the modal."** Correct. Blind timing lowers likelihood and likely concrete severity to Medium. It does not change that A controls the permission transition and can observe the victim's resulting on-chain write before revoking and redirecting.
* **"The victim can replace the resolver."** Recovery is unilateral but reactive. V must detect the redirect and submit a separate transaction. My PoC executes this control, and I do not claim permanent loss.

## Known Issue Boundary

QA-01 covers role combinations during transfer that cause later profile or primary-name errors, while preserving separately demonstrated unintended authority. My report does not concern an error: the wallet transaction succeeds, Manager suppresses its controlled-resolver replacement, and a retained administrator redirects the victim-owned name.

`WEB-1257` concerns a previous owner being unable to set a retained subname as primary. No published Known Issue describes this confirmation-to-operation substitution.

## Duplicate Check

The closed cross-resource removal report signed a correct one-name revoke and relied on a mock resolver for its consequence. My report changes the final contract target and operation after confirmation and executes the resolver behavior on official contracts. The unmappable-resource report instead converts a missing name to the root sentinel and widens authority. The producers, transactions, transitions, and fixes are distinct.

## Remediation

I recommend creating an immutable pending action before opening `ResolverSetupConfirmDialog`. Bind the selected name, resolver, owner, access result, and complete transaction branch. The confirmation and final request must consume that same object. Abort and require fresh confirmation if any value changes.

Do not treat one successful record-write simulation as proof that the owner controls a shared resolver. Verify trusted provenance and the relevant root or admin authority. Otherwise, use `setupControlledResolver` for transferred names. Display the resolver target and whether the registry pointer will change. Add a regression test that changes access `false -> true` while the confirmation remains open.

## References

* `audit-comp-ens/apps/manager/src/features/profile/service/resolverWriteAccess.ts:21-91`
* `audit-comp-ens/apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx:418-510`
* `audit-comp-ens/apps/manager/src/features/profile/components/dialogs/ResolverSetupConfirmDialog.tsx:23-86`
* `audit-comp-ens/apps/manager/src/features/profile/service/setupControlledResolver.ts:41-136`
* `audit-comp-ens/apps/manager/src/features/profile/service/profileRecordTransactions.ts:372-566`

## Link to Proof of Concept

<https://gist.github.com/niafreu/05a6afc47a864e543285caa65ad0b5d8>

## Proof of Concept

## Proof of Concept

My PoC is available at <https://gist.github.com/niafreu/05a6afc47a864e543285caa65ad0b5d8>.

It mounts the production Manager dialog, executes the production write-access query, changes resolver authorization, triggers the real focus refetch, submits through production `saveRecords` and `transactionManager`, and signs with the local victim wallet.

### Preconditions

* `immunefi-team/audit-comp-ens` at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, with dependencies installed.
* `ensdomains/contracts-v2` at `48b3e2d39513b9dd32ef1850877a29009bc807b9`, including submodules.
* Node.js 22 and Foundry.

### Run

```bash
export ENS_REPO_DIR=/absolute/path/to/audit-comp-ens
export CONTRACTS_V2_DIR=/absolute/path/to/contracts-v2/contracts

PATH=/absolute/path/to/node-22/bin:$PATH \
  "$ENS_REPO_DIR/apps/manager/node_modules/.bin/vitest" run \
  --disableConsoleIntercept \
  --reporter=verbose \
  --config /absolute/path/to/poc/vitest.config.ts
```

### Expected result

```
Test Files  1 passed (1)
Tests       1 passed (1)

"accessBeforeConfirmation": false
"accessAfterFocusRefetch": true
"signedRequestTo": "<attackerResolver>"
"signedRequestFunction": "setAddr"
"controlledResolverSetupCalls": 0
"retainedPreviousOwnerText": "https://old-profile.invalid"
"attackerRedirectAfterVictimTransaction": "<attacker address>"
"recoveredActiveResolver": "<different victim resolver>"
```

My PoC uses UI fixtures only for the owned-name inventory, reverse-name read, avatar, and final primary-name operation. Those are outside the security transition.

The resolver pointer and ETH record come from official local contracts. The production probe determines the branch, and production code builds, signs, submits, mines, and verifies the vulnerable request. I observe the safe setup service at its boundary to prove it is skipped. My recovery control executes the corresponding resolver replacement on-chain.
