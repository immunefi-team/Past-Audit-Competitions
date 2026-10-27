# 52849 sc high claimers who claim after slash inactive updaterewardpertokenforvalidator which advances validatorlastupdatetimes to be more than slashtimestamp will lose rewards for a segment

* Submitted on: Aug 13th 2025 at 16:07:25 UTC by @IronsideSec for Attackathon | Plume Network
* Report ID: #52849
* Report Type: Smart Contract
* Severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>
* Impact summary: Theft of unclaimed yield — underpayment of users and validator commission; unaccrued tokens remain in treasury.

## Brief / Intro

When a validator becomes slashed or inactive, later admin operations can advance `validatorLastUpdateTimes` to a time greater than or equal to the effective cap (the slash/inactive timestamp). Because `updateRewardPerTokenForValidator`'s slashed/inactive branches write `validatorLastUpdateTimes[validatorId][token] = block.timestamp` and return without first settling rewards up to the cap, subsequent calls to `claim` skip the final pre-cap reward segment. Result: some users (and validator commission) are underpaid; the under-accrued tokens remain in the treasury.

Root cause: in `updateRewardPerTokenForValidator`, the slashed and inactive branches advance `validatorLastUpdateTimes` to `block.timestamp` without settling to the cap time.

## Vulnerability details

### PoC attack flow

{% stepper %}
{% step %}

### Step: Setup

1. Set pUSD rate to `1e18`.
2. user1 stakes 10 ETH to validator v0; user3 stakes 10 ETH to v0.
3. Advance time `vm.warp(+1000)`.
   {% endstep %}

{% step %}

### Step: Slash validator

4. `voteToSlashValidator(v0, block.timestamp + 1 days)` → validator is auto-slashed at `Tslash`.
5. user1 immediately `claim(pUSD, v0)` → receives pre-slash rewards (e.g. \~9500e18).
   {% endstep %}

{% step %}

### Step: Advance last update past cap

6. `vm.warp(+10)`. Anyone calls `forceSettleValidatorCommission(v0)` → this invokes `updateRewardPerTokenForValidator` which (in slashed branch) writes `validatorLastUpdateTimes[v0][pUSD] = now` (post-cap).
   {% endstep %}

{% step %}

### Step: Victim claim

7. user3 calls `claim(pUSD, v0)` → returns 0 because the pre-slash segment is skipped.
8. Outcome: user3 receives 0 rewards while user1 already received their payout; under-accrued tokens remain.
   {% endstep %}
   {% endstepper %}

### Flows that cause the bug

{% stepper %}
{% step %}

### Validator gets slashed

* `ValidatorFacet.voteToSlashValidator(vId, expiry)` sets `slashedAtTimestamp = Tslash`.
* Later, admin calls `ValidatorFacet.forceSettleValidatorCommission(vId)` (or `RewardsFacet.removeRewardToken(token)` / `RewardsFacet.setRewardRates(tokens, rates)`).
* Those call `PlumeRewardLogic.updateRewardPerTokenForValidator($, token, vId)` → slashed branch writes `validatorLastUpdateTimes[vId][token] = now`.
* Subsequent `RewardsFacet.claim(token, vId)` → `calculateRewardsWithCheckpoints` computes `effectiveEndTime = Tslash <= lastUpdateTime` → final pre-slash accrual skipped.
  {% endstep %}

{% step %}

### Validator set inactive

* Admin calls `ValidatorFacet.setValidatorStatus(vId, false)` → records `slashedAtTimestamp = Tinactive` and pushes 0-rate checkpoints.
* Any later admin op calls `updateRewardPerTokenForValidator($, token, vId)`, which (inactive branch) writes `validatorLastUpdateTimes[vId][token] = now`.
* Subsequent `RewardsFacet.claim(token, vId)` → final pre-inactive accrual skipped.
  {% endstep %}

{% step %}

### Token removal during/after slash/inactive

* Admin calls `RewardsFacet.removeRewardToken(token)`.
* Loop calls `PlumeRewardLogic.updateRewardPerTokenForValidator($, token, vId)` then `createRewardRateCheckpoint($, token, vId, 0)`.
* For slashed/inactive validators, updater writes `validatorLastUpdateTimes[vId][token] = now` (> cap).
* Later claims skip the pre-cap delta.
  {% endstep %}

{% step %}

### Set reward rates after slash/inactive

* Admin calls `RewardsFacet.setRewardRates(tokens, rates)` (or `setMaxRewardRate(token, newMax)`).
* `createRewardRateCheckpoint` first calls `updateRewardPerTokenForValidator`.
* Updater on slashed/inactive writes `validatorLastUpdateTimes[vId][token] = now`.
* Later claims skip the last pre-cap segment.
  {% endstep %}
  {% endstepper %}

## Impact details

* Users are underpaid for the final pre-cap time segment.
* Validator commission for that segment is under-accrued.
* Treasury retains more tokens than intended.
* Magnitude: missed\_time\_segment × applicable\_rate × stake / precision, aggregated over affected users/tokens.
* Likelihood and impact are high because common admin actions (settlement, rate updates, token removal) occur often and call the updater.

## Recommended remediation

* In `updateRewardPerTokenForValidator`, do not advance `validatorLastUpdateTimes` in slashed/inactive branches.
* Alternatively, when transitioning a validator to slashed/inactive, immediately settle rewards up to the cap and set `validatorLastUpdateTimes` to exactly that cap time; thereafter prevent any advancement past the cap while the validator remains slashed/inactive.
* Ensure checkpoint creation paths that invoke the updater either skip calling it when slashed/inactive, or use updater logic that respects the cap and never moves `lastUpdateTimes` beyond it.

## References / Relevant code excerpts

* The slashed and inactive branches currently set `validatorLastUpdateTimes[validatorId][token] = block.timestamp` and return without settling to the cap time — the root cause.
* In `calculateRewardsWithCheckpoints`, the slashed path sets `effectiveEndTime = validator.slashedAtTimestamp` and only accrues a final segment if `effectiveEndTime > validatorLastUpdateTime`. Advancing `validatorLastUpdateTimes` to now (≥ cap) causes that condition to be false and the pre-cap segment to be skipped.
* Common admin flows that call the updater after slash/inactive will systematically cause underpayment.

Code excerpts (lines referenced in original report):

```solidity
File: attackathon-plume-network/plume/src/lib/PlumeRewardLogic.sol
135:

136:     function updateRewardPerTokenForValidator(
137:         PlumeStakingStorage.Layout storage $, address token, uint16 validatorId
140:     ) internal {
141:         PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId]; // Get validator info
142:
143:         // --- REORDERED SLASHED/INACTIVE CHECKS ---
144:         // Check for slashed state FIRST since slashed validators are also inactive
145:         if (validator.slashed) {
146:             // For slashed validators, no further rewards or commission accrue.
147:             // We just update the timestamp to the current time to mark that the state is "settled" up to now.
148:   >>>       $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;
149:
150:             // Add a defensive check: A slashed validator should never have any stake. If it does, something is
151:             // wrong with the slashing logic itself.
152:             if ($.validatorTotalStaked[validatorId] > 0) {
153:                 revert InternalInconsistency("Slashed validator has non-zero totalStaked");
154:             }
155:             return;
156:         } else if (!validator.active) {
157:             // For inactive (but not slashed) validators, no further rewards or commission accrue.
158:             // We just update the timestamp to the current time to mark that the state is "settled" up to now.
159:   >>>       $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;
160:             return;
161:         }
164:         // --- END REORDERED CHECKS ---
...
203:         // Update last global update time for this validator/token AFTER all calculations for the segment
204:         $.validatorLastUpdateTimes[validatorId][token] = block.timestamp;
205:     }

400:   function calculateRewardsWithCheckpoints(
401:         PlumeStakingStorage.Layout storage $,
402:         address user,
403:         uint16 validatorId,
404:         address token,
405:         uint256 userStakedAmount
406:     ) internal returns (uint256 totalUserRewardDelta, uint256 totalCommissionAmountDelta, uint256 effectiveTimeDelta) {
407:         PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];
408:
411:         if (!validator.slashed) {
412:             // Normal case: update and use the updated cumulative.
413:             updateRewardPerTokenForValidator($, token, validatorId);
414:             uint256 finalCumulativeRewardPerToken = $.validatorRewardPerTokenCumulative[validatorId][token];
415:             return _calculateRewardsCore($, user, validatorId, token, userStakedAmount, finalCumulativeRewardPerToken);
416:         } else {
417:             // Slashed validator case: calculate what cumulative should be up to the slash timestamp.
418:             // We DO NOT call updateRewardPerTokenForValidator here because its logic is incorrect for slashed validators.
419:             uint256 currentCumulativeRewardPerToken = $.validatorRewardPerTokenCumulative[validatorId][token];
420:  >>>        uint256 effectiveEndTime = validator.slashedAtTimestamp;
421:
422:             uint256 tokenRemovalTime = $.tokenRemovalTimestamps[token];
423:             if (tokenRemovalTime > 0 && tokenRemovalTime < effectiveEndTime) {
424:                 effectiveEndTime = tokenRemovalTime;
425:             }
426:
427:   >>>       uint256 validatorLastUpdateTime = $.validatorLastUpdateTimes[validatorId][token];
428:
429:   >>>       if (effectiveEndTime > validatorLastUpdateTime) {
430:                 uint256 timeSinceLastUpdate = effectiveEndTime - validatorLastUpdateTime;
431:
432:                 if (userStakedAmount > 0) {
433:                     PlumeStakingStorage.RateCheckpoint memory effectiveRewardRateChk =
434:                         getEffectiveRewardRateAt($, token, validatorId, validatorLastUpdateTime); // Use rate at start of segment
435:                     uint256 effectiveRewardRate = effectiveRewardRateChk.rate;
436:
437:                     if (effectiveRewardRate > 0) {
438:                         uint256 rewardPerTokenIncrease = timeSinceLastUpdate * effectiveRewardRate;
439:                         currentCumulativeRewardPerToken += rewardPerTokenIncrease;
440:                     }
441:                 }
442:             }
443:
444:             return _calculateRewardsCore($, user, validatorId, token, userStakedAmount, currentCumulativeRewardPerToken);
445:         }
446:     }
```

## Proof of Concept

Follow the steps in the provided gist: <https://gist.github.com/IronsideSec/8b8eb5d9c902d7fd91ef9190f2bbecce>

(Original report PoC steps included above.)

***

If you want, I can prepare a suggested patch diff that implements the recommended fix (e.g., settling to the cap and avoiding advancing `validatorLastUpdateTimes` in slashed/inactive branches) based strictly on the code excerpts.
