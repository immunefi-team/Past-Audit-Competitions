# #42214 \[SC-High] Leftover \`WBERA\` and \`YEET\` sent to \`StakeV2\` instead of to user who is claiming rewards

**Submitted on Mar 21st 2025 at 18:24:40 UTC by @Oxl33 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42214
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**
  * Theft of unclaimed royalties

## Description

**Description:**

When users claim rewards through one of the claim functions in `StakeV2`, after removing liquidity from `Kodiak` and swapping, commonly (if not always) there will be left unused `WBERA` and `YEET` tokens. The issue lies in the use of `_msgSender()` as the receiver of the leftover tokens, because when `StakeV2` calls `Zapper`, `msg.sender` is not the user who initiated the transaction, but `StakeV2` contract. Due to this, one of the leftover tokens (or in the case of `claimRewardsInToken` function - both `WBERA` and `YEET`) get sent to `StakeV2` and later added to rewards that other stakers can claim, when in reality, these leftover tokens belong to the user who initiated the transaction, because they originated from `Kodiak` LP tokens that the user had the right to claim as reward, based on their staked `YEET` amount and the duration of their staking.

`WBERA` and/or `YEET` tokens can be left unused during the claiming process in these cases:

1. If `Kodiak` returns bigger than expected output amounts of `WBERA` and `YEET` after burning `Kodiak` LP tokens
2. During swaps from `WBERA` to `YEET` and vice versa, if expected input amount is smaller than actual output from `Kodiak`
3. During swaps from `WBERA`/`YEET` to another whitelisted token (e.g. `KDK` or `oBERO`), if expected input amount is smaller than actual output from `Kodiak`

Considering all these possible cases and the fact that claiming rewards is one of the main functionalities of this protocol, I believe this issue will happen commonly and will affect the majority of the users.

The issue occurs in these functions:

`StakeV2::claimRewardsInNative` -> `Zapper::zapOutNative` -> `_swapToWBERA` - only `YEET` sent to `StakeV2`\
`StakeV2::claimRewardsInToken0` -> `Zapper::zapOutToToken0` - only `YEET` OR only `WBERA` sent to `StakeV2`\
`StakeV2::claimRewardsInToken1` -> `Zapper::zapOutToToken1` - only `YEET` OR only `WBERA` sent to `StakeV2`\
`StakeV2::claimRewardsInToken` -> `Zapper::zapOut` - both `YEET` AND `WBERA` sent to `StakeV2`

**Impact:**

I think the most fitting impact from all the in-scope impacts is `Theft of unclaimed royalties`, because the affected users lose a part of the royalties that belongs to them and other stakers benefit from them, so it is similar to theft.

This impact is meant for high severity issues and I believe it fits the overall severity of this issue, because this issue will occur frequently (high likelihood) and the funds of users are mismanaged, which leads to undeserved losses for some users and gains for others (high impact).

## Proof of Concept

**Proof of Concept:**

1. Alice stakes her YEET tokens using `StakeV2`
2. A percentage of BERA from yeets of users gets sent to `StakeV2`
3. Manager distributes the rewards with `executeRewardDistribution` and `executeRewardDistributionYeet` functions
4. Alice wants to claim her share of the rewards and she wants to receive them in the form of e.g. `oBERO` tokens, so she calls `StakeV2::claimRewardsInToken` and specifies output token as `oBERO` (assuming it is whitelisted, because it is used in `compound`)
5. `Zapper::zapOut` function gets called and there are leftover tokens of `WBERA` and `YEET`, due to the reasons I mentioned in `Description` section
6. `_clearUserDebt` gets called with `token0Debt` and `token1Debt` being more than 0, and `_msgSender()` being `StakeV2` contract
7. Leftover tokens that rightfully belong to Alice get sent to `StakeV2` contract

`Zapper::zapOut` function gets called here: <https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L391>

`_clearUserDebt` function gets called here (`_msgSender()` is `StakeV2`): <https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/contracts/Zapper.sol#L352>
