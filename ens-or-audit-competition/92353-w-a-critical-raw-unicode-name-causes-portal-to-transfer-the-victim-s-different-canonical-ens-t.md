# 92353 w a critical raw unicode name causes portal to transfer the victim s different canonical ens token

**Submitted on Sep 9th 2026 at 06:49:41 UTC by @calbebop for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92353
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief

Portal preserves a raw ENS name for ownership checks and every transfer confirmation, but normalizes it for the permission check and ERC-1155 token lookup. An attacker can register a distinct non-normalized token directly to a victim. When the victim uses Portal's normal Transfer flow to return that apparent gift, the approved transaction transfers the victim's separate canonical name to the attacker. The name shown by Portal remains owned by the victim and is never transferred.

## Vulnerability Details

ENSv2 permits the registrar to create a token from the exact UTF-8 label supplied to it. For example, the raw full-width label `ａｌｉｃｅ` and canonical ASCII label `alice` have distinct label hashes and distinct ERC-1155 token IDs.

Portal handles these identities inconsistently:

1. `resolveEnsOwner.ts` passes the raw route name to ENSJS owner and registry discovery. Portal therefore correctly determines that the victim owns the attacker-gifted raw token.
2. The Transfer page keeps that raw name for its ownership gate, page context, and confirmation text.
3. `getLabel.ts` calls `ens_normalize(name)`.
4. Both the transfer-permission check and `useTransferName.ts` call `getLabel(name)`. The latter supplies the normalized label to `getTokenId`, selecting the canonical token ID.
5. `transferToken.ts` encodes `safeTransferFrom(victim, recipient, tokenId, 1, 0x)` using that canonical token ID, while its transaction description still says `Transfer ${rawName}`.

The violated invariant is that the name used for ownership authorization and confirmation must identify the same token placed in calldata.

The attacker registers the distinct raw label directly to a victim who owns the corresponding canonical name and can transfer it. The attacker then sends that raw name's Portal Transfer link and asks the victim to return the gift. The victim deliberately selects the attacker as recipient and approves one normal transfer request; the undisclosed substitution is the asset being transferred. No race or stale state is involved.

## Impact

The victim loses ownership of the canonical ENSv2 name to the attacker. This is not a revocable approval: the mined ERC-1155 transfer changes the token owner, and recovery requires cooperation or authority from the new owner.

The demonstrated result maps to the selected Critical impacts:

* direct theft of a user NFT/name; and
* a malicious interaction with an already-connected wallet in which transaction arguments are not those represented by the application.

The recipient is deliberately selected. The substituted value is the ERC-1155 token ID/name.

Practical likelihood is low-to-moderate because the attack is targeted and requires one victim-approved Transfer flow. A production wallet with asset-aware simulation or metadata may independently identify the canonical token and warn the user; this has not been tested. When the victim owns the canonical token, holds its transfer role, and approves the request, the demonstrated transfer is deterministic.

## Suggested Remediation

Reject Transfer for any name where `ens_normalize(name) !== name`, or carry the exact raw label and token ID discovered during ownership resolution through permission checks, confirmation, and submission without re-normalizing it.

Add a regression test in which one wallet owns both raw and canonical tokens and assert that transferring either moves only the asset named in the UI.

## References

* In-scope Portal asset: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Competition scope and known issues: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Raw owner discovery: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/ens/resolveEnsOwner.ts#L44-L61>
* Normalizing label helper: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/token/getLabel.ts#L10-L19>
* Normalized token lookup: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useTransferName.ts#L106-L119>
* Raw modal name: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useTransferName.ts#L174-L180>
* Transfer encoder and raw description: <https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/helpers/transferToken.ts#L63-L89>

## Link to Proof of Concept

<https://gist.github.com/calbebop/b076f5cc88ba236cf5ce2cf4c64cfa95>

## Proof of Concept

## Proof of Concept

Attached files:

* `poc_portal_unicode_name_transfer.ts`
* `poc_portal_unicode_name_transfer.browser.spec.ts`
* `reproduce_portal_unicode_transfer.ps1`
* `unicode-transfer-form.png`
* `unicode-transfer-modal.png`

The browser harness uses the repository's headless test wallet and approves its single captured request. It returns an empty healthy response for indexer POSTs because names created on the disposable fork cannot appear in the public indexer.

Ownership, registry, role and token-ID reads, the wallet request, mined transaction, receipt, and final owners all use the local fork. No production-chain state is changed.

The setup uses Anvil time controls and test-token funding and clears delegated code found at a standard test account on the fork. These steps prepare disposable laboratory accounts; they do not alter Portal's name handling or asset-selection logic.

### Reproduction

From a directory containing the three attached PoC files, run:

```powershell
.\reproduce_portal_unicode_transfer.ps1
```

The runner:

1. Creates a new Docker volume and network.
2. Clones the competition repository inside Docker.
3. Checks out exact target commit `cda79acaad59711b943fc68207ebb3f1d0ff8596`.
4. Installs the frozen lockfile inside the Docker volume.
5. Starts the pinned Anvil Sepolia fork and frozen Portal.
6. Creates a canonical victim name and a distinct raw Unicode gift.
7. Drives the actual Portal Transfer flow in Chromium.
8. Captures and decodes the sole wallet transaction.
9. Verifies the mined transaction, receipt, and final owners.

No project dependencies or PoC code execute directly on the host.

The test verifies:

1. The raw and canonical token IDs differ and initially belong to the victim.
2. The inspected Transfer form and final Portal confirmation display the raw name, not the canonical name.
3. Exactly one `eth_sendTransaction` request is made.
4. Its calldata is `safeTransferFrom(victim, attacker, canonicalTokenId, 1, 0x)`.
5. The encoded ID equals the canonical ID and differs from the displayed raw token ID.
6. The mined transaction input matches the captured wallet request and succeeds.
7. The canonical owner becomes the attacker while the displayed raw token remains owned by the victim.

Representative exact-commit result:

```
POC_BROWSER={"displayedName":"ｕｎｉｃｏｄｅ－ｔｒａｎｓｆｅｒ－ｐｏｃ－１７８８９３２４７４.eth","canonicalName":"unicode-transfer-poc-1788932474.eth","displayedTokenId":"25644041079087190698526835734106737048935251306741923601100172801688914100224","canonicalTokenId":"24701099380016683194889040045663444104917629194153627479973838875436183977984","encodedTarget":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2","encodedFrom":"0x70997970C51812dc3A010C7d01b50e0d17dc79C8","encodedRecipient":"0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266","encodedAmount":"1","sendTransactionCount":1,"submittedHash":"0x5f4449b6231531dd59f61c9531597e3755d8ee2c39c2f931ee0f4cb10d98315f","receiptStatus":"success","receiptBlock":"11666054","displayedOwnerAfter":"0x70997970C51812dc3A010C7d01b50e0d17dc79C8","canonicalOwnerAfter":"0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266"}
1 passed (23.4s)
```

This result and both screenshots came from the exact target commit using a newly cloned checkout, empty Docker volume, and frozen-lockfile installation. A second exact-commit run with a different fresh name and transaction also passed.
