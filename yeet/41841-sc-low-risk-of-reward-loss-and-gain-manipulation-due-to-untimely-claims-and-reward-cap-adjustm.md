# #41841 \[SC-Low] Risk of Reward Loss and Gain Manipulation Due to Untimely Claims and Reward Cap Adjustments

**Submitted on Mar 18th 2025 at 20:00:05 UTC by @Yaneca\_b for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41841
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The issue involves the ability for users to lose or unfairly gain rewards due to improper timing of claims, especially when the owner changes the reward cap adjustment factor. If exploited, users could miss out on rewards by not claiming in time, or manipulate the system by timing their claims to maximize rewards. This could lead to unfair distribution of rewards, negatively impacting users who fail to claim promptly, and creating potential manipulation opportunities for others, although it does not result in permanent loss of funds or protocol insolvency.

## Vulnerability Details

The vulnerability arises from the fact that the MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR can be modified by the contract owner during an active epoch, affecting the reward distribution for users who have large epoch volumes but haven't claimed their rewards yet. The key issue is that the maxClaimable amount for past epochs is calculated using the current value of the MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR, which may lead to unfair reward distributions for users who have delayed their claims.

Exploitation Scenarios:

* Back-running: If the MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR is reduced, users who haven’t claimed their rewards can back-run and claim after the reduction, which may lead to losing reward value since the cap is now lower.
* Front-running: If the MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR is increased, users can front-run their claims to benefit from the higher cap, thus gaining more rewards than intended.

This happens because maxClaimable for past epochs is always calculated based on the current value of MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR, meaning changes to the factor impact the reward calculations for users who have not yet claimed.

Code Context:\
The problem lies in the calculation of the maxClaimable amount for each user:

```
uint256 maxClaimable = (epochRewards[epoch] / rewardsSettings.MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR());
```

Since this calculation uses the current value of MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR for all epochs, users can manipulate the timing of their claims to either increase or decrease their rewards, depending on whether the factor is raised or lowered.

## Impact Details

This vulnerability creates unfair reward distribution where users can either lose rewards or gain excessive rewards depending on when they claim. The maxClaimable for past epochs is calculated using the current MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR, making it vulnerable to front-running or back-running by users.

* User Loss: If MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR is increased, users who delay claiming lose the opportunity to claim rewards at the original rate.
* User Gain: If the factor is reduced, users who back-run could claim rewards at the lower cap and gain more than intended.
* Protocol Risk: While this does not directly cause loss of funds, it could deplete the reward pool early, affecting the long-term sustainability of the protocol.
* User Frustration: Users who don’t claim on time might lose rewards or feel unfairly impacted, harming the protocol’s reputation.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Reward.sol#L187-L194\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/RewardSettings.sol#L41-L54>

## Proof of Concept

## Proof of Concept

Place inside Reward.Test.sol and run :)

```
contract RewardSettingsImpactTest is Test {
    Reward private reward;
    address user = address(bytes20("user"));
    MockERC20 token;
    RewardSettings settings;

    function setUp() public {
        token = new MockERC20("TEST", "TEST", 18);
        settings = new RewardSettings();
        //settings.setYeetRewardsSettings(1);

        reward = new Reward(token, settings);
        reward.setYeetContract(address(this));

        token.mint(address(reward), 40_000_000 * (10 ** 18));
    }

    function test_POC3() public {
        reward.addYeetVolume(user, 100e18);

        vm.warp(1 days + 1 minutes);

        reward.addYeetVolume(user, 100e18);

        uint256 amountIfClaimBeforeSettingsUpdate = reward.getClaimableAmount(user); // the user was able to claim this but did not for some reason 

        settings.setYeetRewardsSettings(50);

        uint256 amountIfClaimAfterSettingsUpdate = reward.getClaimableAmount(user); // because of his waiting he looses money in this scenario 

        console.log("Amount if claim before settings update ->", amountIfClaimBeforeSettingsUpdate);
        console.log("Amount if claim after settings update  ->", amountIfClaimAfterSettingsUpdate);
        // solution: tract the MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR for every epoch, 
        // this will prevent from the chance for the user to loose money 
         // or get more even though the epoch maths were done already without this calculation
    }
}
```
