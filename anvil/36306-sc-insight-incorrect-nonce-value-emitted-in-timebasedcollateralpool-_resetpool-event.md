# #36306 \[SC-Insight] Incorrect nonce value emitted in \`TimeBasedCollateralPool::\_resetPool\` event

**Submitted on Oct 28th 2024 at 18:44:16 UTC by @MrMorningstar for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36306
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://etherscan.io/address/0xd042C267758eDDf34B481E1F539d637e41db3e5a>
* **Impacts:**
  * Incorrect Emission of an Event

## Description

## Brief/Intro

Incorrect nonce value emitted in \`TimeBasedCollateralPool::\_resetPool\` event

## Vulnerability Details

If pool is in process to reset in \`\_resetPool\`(<https://github.com/AcronymFoundation/anvil-contracts/blame/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/TimeBasedCollateralPool.sol#L1022>) the nonce is updated in this line here:

\`\`\`js // NB: must be resetNonce++, NOT ++resetNonce uint256 resetNonce = contractStateStorage.resetNonce++; \`\`\` After the reset is done the the following event is emitted:

\`\`\`js emit PoolReset(IERC20(\_tokenAddress), resetNonce + 1, tokensToReset, unitsToReset); \`\`\` The issue here that the event uses already updated value of \`resetNonce\`.

## Impact Details

This does not impact functionality of the protocol, but it could impact place that rely on the value of the \`PoolReset\` event (maybe on front end if someone wants to know if reset happened or how many times reset is happened) because wrong data will be passed.

## Recommendation

Change the event emitted like this to pass the correct information

\`\`\`diff

* emit PoolReset(IERC20(\_tokenAddress), resetNonce + 1, tokensToReset, unitsToReset);
* emit PoolReset(IERC20(\_tokenAddress), resetNonce, tokensToReset, unitsToReset); \`\`\`

## Proof of Concept

## Proof of Concept

POC:

Lets assume there are no resets before and this is the first reset of the pool so the resetNonce will have following value: \`contractStateStorage.resetNonce = 0\`

If reset process is eligible and will proceed the \`resetNonce\` would be updated to the new value of 1 as we can see in this line

\`\`\`js // NB: must be resetNonce++, NOT ++resetNonce uint256 resetNonce = contractStateStorage.resetNonce++; \`\`\` So \`resetNonce = 1\` now.

After reset is processed the \`PoolReset\` event will be emitted with \`newResetNonce\` value of 2 instead of correct value of 1 because of this \`resetNonce + 1\`. This is incorrect because the actual value of \`resetNonce\` is 1 not 2.
