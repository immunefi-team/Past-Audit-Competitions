# 93008 w a medium recipient controlled transfer failure leaves ens resolution attacker controlled and diverts payments

**Submitted on Sep 13th 2026 at 20:25:54 UTC by @anand0x for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93008
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

ENS Explorer can leave a failed V2 transfer in a state where the victim still owns the ENS name, but the name's authoritative ETH address record points to the rejected transfer recipient.

The issue occurs when the owner can transfer the name and edit resolver records but does not hold registry `ROLE_SET_RESOLVER`.

In this state, Explorer hides the resolver-detach option, while the ETH-address option remains checked and disabled. Explorer tells the user:

> Not needed while the resolver is being detached.

That statement is false. No resolver-detach transaction is generated.

Explorer instead executes two separate operations:

1. `setAddr(node, 60, recipient)`
2. `safeTransferFrom(victim, recipient, tokenId, 1, 0x)`

The address update confirms before the ownership transfer.

A malicious ERC-1155 recipient can change its receiver behavior before the final transfer is mined and cause the later `safeTransferFrom` to revert.

The resulting state is:

* the ownership transfer fails;
* the victim remains the owner;
* the resolver remains attached;
* `addr(name, 60)` points to the rejected recipient.

In my PoC, an independent payer resolved the victim-owned ENS name after the failed transfer, obtained the malicious recipient through ordinary ENS resolution, sent `0.001 ETH` to that resolved address, and the recipient withdrew the payment.

## Security invariant

If Explorer reports that a V2 ownership transfer failed and ownership did not change, application-generated pre-transfer steps must not leave the still-victim-owned name resolving to the rejected transfer recipient.

This is particularly important where Explorer tells the victim that the forced address operation is unnecessary because the resolver is supposedly being detached.

## Affected asset and revision

* Asset: `apps/portal` — ENS Explorer
* Official audited revision: `63772fd872af472ced58b009499355f3430c2a86`
* Tested competition import: `cda79acaad59711b943fc68207ebb3f1d0ff8596`
* Current competition cross-check: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

The relevant Explorer source was also cross-checked against the current competition tree.

## Root cause

The defect is caused by inconsistent use of the transfer form's raw option state and the effective state used to construct the actual transaction plan.

### 1. Transfer options default to enabled

`apps/portal/src/features/transfer/components/SendNameForm.tsx:69-73` initializes the stored transfer options to `true`.

### 2. Resolver detachment becomes unavailable

`apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts:83-95` hides resolver detachment when the owner lacks registry `ROLE_SET_RESOLVER`.

The ETH-address operation can remain visible when a coin-60 record exists.

### 3. The transaction plan correctly masks the hidden resolver option

`SendNameForm.tsx:91-96` computes `effectiveOptions`.

Because resolver detachment is unavailable, `effectiveOptions.detachResolver` becomes `false`.

Those effective options are used to construct the actual transaction plan.

### 4. The UI still reads the raw resolver-detach value

`SendNameForm.tsx:184-203` uses raw `options.detachResolver` when deciding whether the ETH-address option is disabled and which explanatory text is shown.

The victim therefore sees the ETH-address option forced on and disabled together with:

> Not needed while the resolver is being detached.

But the resolver is not being detached.

The accurate explanation of the ETH-address mutation is suppressed in this state.

### 5. The generated plan performs the address mutation before the ownership transfer

`buildTransferPlan.ts:27-40` produces:

1. `set-eth-addr`
2. `transfer-token`

There is no `detach-resolver` step.

`useTransferName.ts:134-180` sequences these operations.

`setEthAddress.ts:37-55` and `features/reverse-resolution/helpers/setForwardResolution.ts:84-92` submit the address-record update.

`transferToken.ts:73-96` later submits the separate ERC-1155 `safeTransferFrom`.

Because these are separate confirmed transactions, failure of the final recipient-controlled transfer cannot revert the already-confirmed address mutation.

## Preconditions

The demonstrated path requires:

* a victim-owned transferable V2 ENS name with an existing ETH address record;
* the victim retaining transfer authority and resolver record-writing authority;
* the victim lacking registry `ROLE_SET_RESOLVER`;
* the victim selecting an attacker-controlled ERC-1155 receiver as the intended transfer recipient;
* the victim using Explorer's normal transfer flow without changing the transfer options;
* the malicious recipient changing its ERC-1155 receive behavior before the final transfer is mined;
* a payer resolving and paying the ENS name before the victim repairs the address record.

The audited Roles UI can create the required role state by revoking `ROLE_SET_RESOLVER` while leaving transfer authority intact.

No victim private-key compromise or project-admin privilege is required.

## Threat model

The attacker is the intended transfer recipient, for example a malicious buyer or recipient of the ENS name.

The victim performs only Explorer's ordinary transfer workflow. The exploit does not require phishing, a compromised wallet, manually crafted victim transactions, or an abnormal UI interaction.

The attacker controls only the recipient contract and changes its ERC-1155 receive behavior before the final transfer is mined.

## Attack sequence

1. The victim opens Explorer's V2 transfer flow.
2. Resolver detachment is hidden because the victim lacks `ROLE_SET_RESOLVER`.
3. The ETH-address option remains checked and disabled.
4. Explorer displays:

   `Not needed while the resolver is being detached.`
5. No resolver-detach step is included in the generated plan.
6. Explorer generates `setAddr(node, 60, recipient)`.
7. The victim approves that transaction and it succeeds.
8. Explorer then submits the separate ERC-1155 `safeTransferFrom`.
9. Before the transfer is mined, the malicious recipient changes its receiver state to reject ERC-1155 receipt.
10. `safeTransferFrom` reverts.
11. Ownership remains with the victim.
12. The earlier address mutation remains committed.
13. `addr(name, 60)` resolves to the rejected recipient.
14. An independent payer resolves the victim-owned ENS name and receives that address.
15. The payer sends `0.001 ETH` to the resolution-derived address.
16. The recipient receives and withdraws the payment.

## Impact

The reproduced post-state was:

```
Explorer result                = transfer failed
owner(name)                    = victim
registry.balanceOf(victim)     = 1
registry.balanceOf(recipient)  = 0
resolver(name)                 = unchanged
addr(name, 60)                 = malicious recipient
```

This maps to the listed Critical impact covering an application resolving or rendering an ENS name to an attacker-controlled address so that a user sends funds to an unintended recipient.

The PoC demonstrates the complete consequence:

* authoritative ENS resolution returned the malicious recipient;
* the payer derived the recipient through ENS resolution rather than using a hardcoded attacker address;
* the payer sent `0.001 ETH`;
* the malicious recipient received the funds;
* the recipient withdrew them.

The victim can repair the address record through a later resolver transaction. That prevents future diversion but cannot recover funds already sent and withdrawn.

This report does not claim permanent name loss or permanent bricking.

## Known-issue distinction

This report does not claim that generic multi-transaction partial completion is itself a new vulnerability.

QA-01 / WEB-1257 concern successful transfers followed by broken follow-up behavior.

The security state demonstrated here is different:

* the ownership transfer itself reverts;
* ownership never changes;
* an earlier forced address mutation remains committed;
* the intended recipient controls the failure trigger;
* the victim-owned name resolves to that failed recipient;
* a third-party payment is diverted through authoritative ENS resolution.

The root cause is Explorer's inconsistent use of raw and effective resolver-detach state, which also causes it to present a false security-relevant explanation for the forced address mutation.

## Recommended remediation

Do not commit `addr(60)=recipient` before a recipient-controlled ownership transfer when the two operations cannot fail atomically.

Preferred fixes:

1. Where permissions allow, configure the recipient's address record only after ownership transfer has successfully completed.
2. If protocol permissions allow, execute the address mutation and ownership transfer atomically so a rejected transfer reverts both operations.
3. Otherwise remove the pre-transfer address repointing operation.

At minimum:

* derive disabled state and explanatory text from `effectiveOptions`;
* do not state that the resolver is being detached when no detach operation will execute;
* clearly disclose that earlier successful operations remain committed if a later transfer fails;
* provide an immediate restoration flow using the previous address record.

A receiver-interface check or preflight simulation alone does not remove the issue because the recipient's callback behavior can change before mining.

## Proof of Concept

## Environment

I reproduced this issue against a clean checkout of the frozen ENS Audit Competition source using a local, non-broadcast Anvil fork of Sepolia.

No public-chain state-changing transaction was performed.

```
Audited competition import:
cda79acaad59711b943fc68207ebb3f1d0ff8596

Official audited revision:
63772fd872af472ced58b009499355f3430c2a86

Sepolia fork block:
11625998

Node.js:
22

pnpm:
10

Browser automation:
Playwright with Google Chrome

Contract controls:
Foundry / Solidity 0.8.30
```

Validation results:

```
Clean-checkout browser PoC: 1 passed
Deterministic Foundry controls: 5 passed, 0 failed
```

## Actors and fixture

```
Victim:
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

Malicious recipient:
0xD98394c99151E9F7479447A52B6c4ECC521eb467

ENS name:
h3-rejected-transfer-1788550920.eth
```

Initial state:

* the victim owns the V2 ENS name;
* the victim can transfer the name;
* the victim can modify resolver records;
* the victim does not hold registry `ROLE_SET_RESOLVER`;
* coin type 60 initially resolves to the victim;
* the malicious recipient is an ERC-1155 receiver whose rejection behavior can be enabled before the final transfer is mined.

## Steps to reproduce

1. On the local Sepolia fork, register `h3-rejected-transfer-1788550920.eth` to the victim.
2. Set its coin-60 address record to:

   `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`
3. Preserve the victim's transfer and resolver record-writing authority, but revoke registry `ROLE_SET_RESOLVER`.
4. Confirm that the victim still owns the name and can initiate a transfer.
5. Open the name's V2 transfer page in the audited Explorer application as the victim.
6. Enter the malicious recipient:

   `0xD98394c99151E9F7479447A52B6c4ECC521eb467`
7. Do not change any transfer option.
8. Observe that the resolver-detach option is absent.
9. Observe that the ETH-address operation remains checked and disabled.
10. Observe the exact text displayed by Explorer:

```
Not needed while the resolver is being detached.
```

11. Inspect the generated transfer plan.
12. Confirm that there is no resolver-detach operation.
13. Confirm that the plan instead contains:

```
setAddr(node, 60, recipient)
```

followed by the separate ERC-1155 ownership transfer.

14. Approve Explorer's first generated transaction:

```
setAddr(node, 60, recipient)
```

15. Confirm that this transaction succeeds.
16. Allow wallet preflight of the final ownership-transfer transaction.
17. Pause automining after the ownership transfer enters the local mempool.
18. From the malicious recipient controller, submit a higher-priority transaction enabling ERC-1155 receipt rejection.
19. Mine the recipient state-change transaction first.
20. Mine Explorer's ownership-transfer transaction second.
21. Confirm that the recipient state change succeeds.
22. Confirm that Explorer's `safeTransferFrom` reverts with:

```
ReceiptRejected()
```

23. Confirm that ownership did not transfer:

```
owner(name) = victim
registry.balanceOf(victim) = 1
registry.balanceOf(recipient) = 0
```

24. Confirm that the resolver itself remains unchanged.
25. Resolve coin type 60 for the ENS name.
26. Confirm that resolution returns:

```
0xD98394c99151E9F7479447A52B6c4ECC521eb467
```

despite the ownership transfer having failed.

27. From an independent payer account, use that ENS resolution result as the payment recipient.
28. Send:

```
0.001 ETH
```

to the resolved address.

29. Confirm that the malicious recipient receives the payment.
30. Withdraw the payment from the malicious recipient contract.
31. As the victim, restore coin type 60 to the victim address.
32. Confirm that subsequent resolution is repaired.

## Captured transactions

### Explorer transaction 1 — address mutation

```
target:
0x2EE3e82Bfbe4b6ACc9B813809C00c83e9187808f

function:
setAddr(node, 60, 0xD98394c99151E9F7479447A52B6c4ECC521eb467)

hash:
0x116b70e1abf7452c73af804227a6267c808ca19564500f6cd16fd4939dcf1151

receipt:
success, block 11626011, transaction index 0
```

### Recipient state change

```
function:
setRejectReceipt(true)

hash:
0xace6fc603fce5871f80ea03ba48cf8e9a36a2f352118e8491cb2fa3986756777

receipt:
success, block 11626012, transaction index 0
```

### Explorer transaction 2 — failed ownership transfer

```
target:
0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2

function:
safeTransferFrom(victim, recipient, tokenId, 1, 0x)

hash:
0x690a774e559a9fbfafd6909afa04ef57b36b40981afe1e76e88da1a49498ded5

receipt:
reverted, block 11626012, transaction index 1

trace:
ReceiptRejected()
```

## Authoritative post-state

```
owner(name)
= 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

registry.balanceOf(victim)
= 1

registry.balanceOf(recipient)
= 0

resolver(name)
= unchanged

registries(name)
= unchanged

addr(name, 60)
= 0xD98394c99151E9F7479447A52B6c4ECC521eb467
```

The security-relevant state after the failed transfer is therefore:

```
ownership transfer = failed
owner              = victim
ETH resolution     = malicious rejected recipient
```

## Payment proof

The payer did not use a hardcoded attacker address.

The payer first resolved the ENS name and used the returned value as the payment recipient.

```
resolver result:
0xD98394c99151E9F7479447A52B6c4ECC521eb467

payment amount:
0.001 ETH

payment transaction:
0xd53218e1c27d8798cd554f9eeb64e57115f890e1f4cfc84aaf2d57a32af80327

recipient withdrawal:
0x129ec9ce841629ee36d0e1475fd429e3e9273f324a66349eee25fbcd60a385ef

victim repair transaction:
0x800df64aca44bc8dd1dbcfe90093a1a2bbd8054c794df8a917982a5401c07ea3
```

## Expected result

If the ownership transfer fails and the victim remains the owner, Explorer should not leave the victim-owned name resolving to the rejected recipient, particularly after telling the victim that the resolver was being detached.

## Actual result

The address mutation succeeds first.

The malicious recipient then causes the ownership transfer to revert.

Ownership remains with the victim, but authoritative coin-60 resolution continues to point to the malicious recipient.

An independent payer resolving the ENS name therefore sends funds to that recipient, who can withdraw them.

## Causal controls

Five deterministic Foundry tests passed:

```
test_defaultPlan_eoaRecipient_transferSucceeds
test_defaultPlan_ownerCanRestorePointers
test_defaultPlan_receiverReject_leavesDetachedState
test_ethRepointPlan_ownerCanRestoreEthAddr
test_ethRepointPlan_receiverReject_resolutionFollowsRecipient
```

These controls establish that:

* a normal recipient allows the transfer to complete;
* recipient callback rejection causes the final transfer failure;
* the earlier address mutation survives that failure;
* authoritative resolution follows the malicious recipient in the vulnerable path;
* the victim can restore the record afterward.

The final control is important: this report does not claim permanent name loss. Repair prevents future diversion but does not recover a payment that has already been sent and withdrawn.

## Browser evidence

I also reproduced the issue in a fresh browser run against competition commit:

`1c9b47f18fcddd2e864dfe385c4171061c9811ae`

That run passed `1/1`.

The three attached screenshots show:

1. the resolver-detach option absent while the ETH-address option is forced on and disabled, together with the false explanation `Not needed while the resolver is being detached.`;
2. Explorer presenting `Update ETH address` and `Transfer name` as two separate operations;
3. `Update ETH address` succeeding before `Transfer name` subsequently fails.

The screenshot rerun used the dynamically generated ENS name:

`h3-rejected-transfer-1789327536.eth`

The browser test also reproduced the same ownership, resolution, payment, withdrawal, and repair sequence.
