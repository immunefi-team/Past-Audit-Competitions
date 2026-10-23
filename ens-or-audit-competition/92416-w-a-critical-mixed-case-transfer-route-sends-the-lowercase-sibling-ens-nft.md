# 92416 w a critical mixed case transfer route sends the lowercase sibling ens nft

**Submitted on Sep 9th 2026 at 16:10:33 UTC by @PlasticGinnie98802 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92416
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs

## Description

## Summary

Portal treats the name in a transfer URL as two different identifiers. The page uses the raw value for its ownership check and for the text shown to the user. Later, the transfer code normalizes that value and looks up a token ID from the normalized label.

I reproduced the mismatch with two ENS v2 tokens owned by the same account: Victim.eth and victim.eth. They have different ERC-1155 token IDs. On the /Victim.eth/ownership/transfer route, Portal showed Victim.eth but prepared safeTransferFrom with the token ID for victim.eth. That split is the bug.

## Code path

The raw route value reaches the ownership and display logic in apps/portal/src/routes/$name/ownership/transfer.tsx:21-27 and 102-110. resolveEnsOwner.ts:44-61 and 81-91 receives the same raw name.

The transfer path does something else. useCanTransferName.ts:49-75 calls getLabel(name). useTransferName.ts:106-120 normalizes again before getEnsTokenId. The normalization in getLabel.ts:10-19 includes ASCII case folding. useTransferName.ts:124-145 and transferToken.ts:46-92 then place the resulting token ID into safeTransferFrom, even though the transaction description still names the raw route.

## Exploit

The victim already owns victim.eth. The attacker registers the separate raw label Victim to the victim, then sends the direct transfer link for Victim.eth and asks for that gift back.

Portal accepts the victim as the owner of the name on screen. The victim enters the attacker address and approves the transfer shown by the application. The successful transaction moves victim.eth to the attacker. Victim.eth, the token named by the page, never leaves the victim's account.

No signature forgery or wallet compromise is needed. I also did not rely on a malicious RPC, a smart-account takeover, a third-party failure, or a public-chain race.

## Impact

This transfers a user's canonical ENS NFT to the attacker. The selected impact is direct theft of a user NFT, which the competition lists as Critical.

## EXP-INPUT-005

The known issue covers transforming inputs as a display or homograph problem. Here the consequence goes past display confusion: the approved calldata moves one token while the page describes another. The on-screen token remains with the victim, so the mismatch is observable after the receipt succeeds.

## Fix

Resolve one resource identity before checking ownership and keep it unchanged through display, role checks, token lookup, and calldata construction. A compact representation would include the canonical name, registry, versioned token ID, and owner. Non-canonical transfer routes can also be rejected or redirected before authorization. The last check before opening the wallet should compare the displayed token ID with the token ID encoded in safeTransferFrom.

Target revision: 1c9b47f18fcddd2e864dfe385c4171061c9811ae

## Proof of Concept

## Reproduction

I ran this against the unmodified Portal at commit 1c9b47f18fcddd2e864dfe385c4171061c9811ae. The chain was a local Anvil fork of Sepolia block 11667468. I only read the upstream fork; every account, registration, approval, and transfer used for the test was created locally.

The Playwright test is e2e/projects/portal/tests/raw-normalized-transfer-v5.spec.ts. It reads PORTAL\_APP\_URL and ANVIL\_RPC\_URL, with defaults of <http://localhost:3001> and <http://127.0.0.1:8545>.

Step 1: Start the Portal and the Sepolia fork.

Step 2: Create local attacker and victim accounts. Register Victim.eth and victim.eth, assigning both to the victim. Read both versioned ERC-1155 token IDs and confirm that they differ.

Step 3: Connect the victim wallet and open /Victim.eth/ownership/transfer. The page identifies the connected account as owner and shows Victim.eth.

Step 4: Use the attacker address as the recipient and continue until Portal prepares the wallet request. Decode safeTransferFrom before approving it. The tokenId argument is the ID for victim.eth, not Victim.eth.

Step 5: Approve that request and wait for the receipt. It succeeds. A final ownerOf check shows victim.eth at the attacker address and Victim.eth still at the victim address.

The attached positive-raw-route.png is the mixed-case route used for this run.

## Controls

I reset to the same evm\_snapshot for each control, so the starting token IDs and owners stayed identical.

For the first control I opened /victim.eth/ownership/transfer. Portal displayed the lowercase name and transferred the lowercase token. There was no identity mismatch. This is shown in negative-canonical-route.png.

For the second control I kept the raw Victim.eth token but omitted its lowercase sibling. Portal returned Transfer not available and exposed no transfer action. Ownership did not change. This is shown in necessary-missing-canonical.png.

I repeated the positive case and both controls in a separate replay. The decoded calldata, receipt status, and final owners matched the first run. External browser requests were blocked, so the result did not depend on public RPC fallbacks.
