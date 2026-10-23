# #36567 \[SC-Insight] Anyone can cancel anyone's LOC

**Submitted on Nov 6th 2024 at 05:37:39 UTC by @gladiator111 for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36567
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://immunefi.com/>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Anyone can cancel anyone's LOC through \`LetterOfCredit.sol::cancelLOC\` when it is expired preventing the users usage of \`LetterOfCredit.sol::createLOCFromExpired\`

## Vulnerability Details

\`LetterOfCredit.sol::cancelLOC\` contains a weak if statement which can be bypassed by anyone when the LOC has expired \`\`\`solidity @> if (msg.sender != loc.beneficiary && loc.expirationTimestamp > block.timestamp) { \_validateCancelAuth(\_locId, loc.beneficiary, \_beneficiaryAuthorization); } \`\`\`\
Through this if statement anyone can cancel anyone's LOC when it is expired. This completely prevents the users from using \`LetterOfCredit.sol::createLOCFromExpired\` as this function will get reverted because the LOC has already been deleted from the storage \`\`\`solidity function createLOCFromExpired( uint96 \_locId, address \_beneficiary, uint32 \_expirationTimestamp, bytes memory \_oraclePriceUpdate ) external payable refundExcess nonReentrant { LOC memory loc = locs\[\_locId]; uint256 creditedTokenAmount = loc.creditedTokenAmount;

```
    if (creditedTokenAmount &#x3D;&#x3D; 0) revert LOCNotFound(_locId);
```

@> if (msg.sender != loc.creator) revert AddressUnauthorizedForLOC(msg.sender, \_locId); //will get reverted here as loc.creator will become address(0) \`\`\`

## Impact Details

An attacker can keep canceling all the expired LOCs in the protocol and users will never be able to use \`LetterOfCredit.sol::createLOCFromExpired\` this function.

## Recommendation

modify the if statement as follows \`\`\`solidity

* if (msg.sender != loc.beneficiary && loc.expirationTimestamp > block.timestamp) {
* if (msg.sender != loc.beneficiary || loc.expirationTimestamp > block.timestamp) { \_validateCancelAuth(\_locId, loc.beneficiary, \_beneficiaryAuthorization); } \`\`\`

## References

<https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/LetterOfCredit.sol#L372> <https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/LetterOfCredit.sol#L275>

## Proof of Concept

## Proof of Concept

As the time for expiry for the contest is very near and the protocol doesn't contain any tests (publically), I am just submitting the impact and vulnerability details. I commit to submitting the POC within the next 48 hours.
