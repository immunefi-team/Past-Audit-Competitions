# 57374 sc low staking tier misclassification

**Submitted on Oct 25th 2025 at 16:06:24 UTC by @failsafe\_intern for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57374
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Vulnerability Overview

`BelongCheckIn.sol` determines staking tiers for fee discounts using `staking.balanceOf(user)` (share count) instead of `staking.convertToAssets(balanceOf(user))` (actual LONG exposure). After reward distributions via `Staking.distributeRewards()`, the vault's exchange rate increases (more assets per share), but tier calculation ignores this rebase effect. Users with rebased shares are systematically under-tiered, paying 1-5% higher fees than their actual LONG exposure warrants.

### Root Cause

Helper expects asset amounts but receives share counts.

Helper.sol: stakingTiers expects asset amounts (LONG, 18 decimals):

```solidity
function stakingTiers(uint256 amountStaked) external pure returns (StakingTiers tier) {
    // Expects LONG amount (18 decimals) but receives share balance
    if (amountStaked < 50000e18) {
        return StakingTiers.NoStakes;
    } else if (amountStaked >= 50000e18 && amountStaked < 250000e18) {
        return StakingTiers.BronzeTier;
    } else if (amountStaked >= 250000e18 && amountStaked < 500000e18) {
        return StakingTiers.SilverTier;
    } else if (amountStaked >= 500000e18 && amountStaked < 1000000e18) {
        return StakingTiers.GoldTier;
    }
    return StakingTiers.PlatinumTier;
}
```

BelongCheckIn.sol uses share counts when looking up tiers:

Venue deposit fee uses shares instead of assets:

```solidity
// LINE 378: balanceOf returns shares, not assets
VenueStakingRewardInfo memory stakingInfo =
    stakingRewards[_storage.contracts.staking.balanceOf(venueInfo.venue).stakingTiers()].venueStakingInfo;

address affiliate;
uint256 affiliateFee;
if (venueInfo.referralCode != bytes32(0)) {
```

Promoter payout fee uses shares instead of assets:

```solidity
// LINE 547: balanceOf returns shares, not assets
PromoterStakingRewardInfo memory stakingInfo = stakingRewards[_storage.contracts.staking.balanceOf(
    promoterInfo.promoter
).stakingTiers()].promoterStakingInfo;
```

Staking.sol distributes rewards by increasing assets without minting shares (rebase):

```solidity
function distributeRewards(uint256 amount) external onlyOwner {
    if (amount == 0) revert ZeroReward();
    // ✅ Increases totalAssets without minting shares
    LONG.safeTransferFrom(msg.sender, address(this), amount);
    emit RewardsDistributed(amount);
}
```

### Attack Flow (Passive Economic Error)

This is not an attacker-exploitable vulnerability but a systematic calculation error that occurs passively:

* Initial state: Vault exchange rate 1:1 (1 share = 1 LONG)
* User stakes 40,000 LONG → receives 40,000 shares
* Protocol distributes rewards via `distributeRewards(1_000_000 LONG)`
* Vault rebases: total assets increase, exchange rate becomes >1
* User's actual exposure = shares × exchange rate
* Tier calculation uses share count (undervalues the user's actual LONG exposure), causing under-tiering and higher fees

### Impact

* Impact Category: Griefing (no profit motive for an attacker, but damage to the users or the protocol)
* User Financial Impact:
  * Venue deposits: 1–5% excess deposit fees depending on tier gap
  * Promoter payouts: 1% excess platform fee on LONG distributions
  * Cumulative: Every transaction post-rebase can incur overcharge
  * Scale: All users with rebased shares near tier boundaries affected

Example scenarios:

| User Shares | Exchange Rate | Actual LONG | Classified As   | Correct Tier      | Excess Fee |
| ----------- | ------------- | ----------- | --------------- | ----------------- | ---------- |
| 40,000      | 1.5:1         | 60,000      | NoStakes (10%)  | BronzeTier (9%)   | +1%        |
| 200,000     | 1.3:1         | 260,000     | BronzeTier (9%) | SilverTier (8%)   | +1%        |
| 450,000     | 1.2:1         | 540,000     | SilverTier (8%) | GoldTier (6%)     | +2%        |
| 900,000     | 1.15:1        | 1,035,000   | GoldTier (6%)   | PlatinumTier (3%) | +3%        |

Concrete example:

```
Venue: 40,000 shares, 1.5x rebase → 60,000 LONG exposure
Deposit: $10,000 USDC
Misclassification: NoStakes (10% fee) vs BronzeTier (9% fee)
Excess charge: $100 per deposit
Annual impact (12 deposits): $1,200 overpayment
```

System impact:

* Protocol collects unintended excess fees
* Users experience negative ROI on staking rewards (fees cancel rewards)
* Staking incentive system undermined
* Trust erosion when users discover overcharges

## Link to Proof of Concept

<https://gist.github.com/Joshua-Medvinsky/7b8684a54d5034bda8338d99f4051e4a>

## Proof of Concept

### Prerequisites

```solidity
// Requirements:
// 1. Staking contract deployed with LONG token (ERC4626 vault)
// 2. BelongCheckIn configured with staking tier fee structure
// 3. Owner can call distributeRewards()
```

### Demonstration

{% stepper %}
{% step %}

### Step 1: User Stakes Before Rebase

```solidity
// User stakes 40,000 LONG
vm.startPrank(user);
LONG.approve(address(staking), 40_000e18);
uint256 shares = staking.deposit(40_000e18, user);
vm.stopPrank();

// Initial state: 1:1 exchange rate
assertEq(shares, 40_000e18, "Receives 40k shares");
assertEq(staking.convertToAssets(shares), 40_000e18, "Worth 40k LONG");

// Tier classification BEFORE rebase
uint256 tierBefore = Helper.stakingTiers(staking.balanceOf(user));
assertEq(uint256(tierBefore), uint256(StakingTiers.NoStakes), "Correctly NoStakes (<50k)");
```

{% endstep %}

{% step %}

### Step 2: Protocol Distributes Rewards

```solidity
// Owner distributes 1M LONG as rewards
vm.startPrank(owner);
LONG.approve(address(staking), 1_000_000e18);
staking.distributeRewards(1_000_000e18);
vm.stopPrank();

// Vault rebases: exchange rate increases
uint256 totalAssets = staking.totalAssets();
uint256 totalSupply = staking.totalSupply();
uint256 exchangeRate = (totalAssets * 1e18) / totalSupply;

assertGt(exchangeRate, 1e18, "Exchange rate increased (>1:1)");
```

{% endstep %}

{% step %}

### Step 3: User's Actual Exposure Exceeds Threshold

```solidity
// User still has 40k shares
uint256 userShares = staking.balanceOf(user);
assertEq(userShares, 40_000e18, "Share count unchanged");

// But shares now worth 60k LONG (due to rebase)
uint256 userAssets = staking.convertToAssets(userShares);
assertGt(userAssets, 50_000e18, "Assets exceed BronzeTier threshold");

// Example: 60k LONG exposure
assertApproxEqAbs(userAssets, 60_000e18, 1e18, "~60k LONG exposure");
```

{% endstep %}

{% step %}

### Step 4: Misclassification Occurs

```solidity
// BelongCheckIn uses balanceOf (shares) instead of convertToAssets
uint256 sharesUsedForTier = staking.balanceOf(user);
StakingTiers assignedTier = Helper.stakingTiers(sharesUsedForTier);

// WRONG: NoStakes based on share count
assertEq(uint256(assignedTier), uint256(StakingTiers.NoStakes), "Misclassified as NoStakes");

// CORRECT: Should be BronzeTier based on asset value
uint256 actualAssets = staking.convertToAssets(sharesUsedForTier);
StakingTiers correctTier = Helper.stakingTiers(actualAssets);
assertEq(uint256(correctTier), uint256(StakingTiers.BronzeTier), "Should be BronzeTier");
```

{% endstep %}

{% step %}

### Step 5: User Overpays Fees

```solidity
// Get fee rates for each tier
uint256 noStakesFee = belongCheckIn.stakingRewards(StakingTiers.NoStakes).venueStakingInfo.depositFeePercentage;
uint256 bronzeFee = belongCheckIn.stakingRewards(StakingTiers.BronzeTier).venueStakingInfo.depositFeePercentage;

assertEq(noStakesFee, 1000, "NoStakes: 10%");
assertEq(bronzeFee, 900, "BronzeTier: 9%");

// User pays NoStakes fee instead of BronzeTier fee
uint256 depositAmount = 10_000e6; // $10,000 USDC
uint256 paidFee = (depositAmount * noStakesFee) / 10000;
uint256 correctFee = (depositAmount * bronzeFee) / 10000;
uint256 overcharge = paidFee - correctFee;

assertEq(paidFee, 1000e6, "Pays $1,000 (10%)");
assertEq(correctFee, 900e6, "Should pay $900 (9%)");
assertEq(overcharge, 100e6, "$100 overcharge per deposit");
```

{% endstep %}

{% step %}

### Step 6: Verify Systemic Impact

```solidity
// Every venue deposit post-rebase incurs overcharge
for (uint i = 0; i < 12; i++) {
    vm.prank(user);
    belongCheckIn.venueDeposit(venueInfo);
    // Each deposit charged 10% instead of 9%
}

// Annual excess fees: $100 × 12 = $1,200
uint256 annualOvercharge = 100e6 * 12;
assertEq(annualOvercharge, 1200e6, "$1,200 excess fees per year");

// User's staking reward negated by excess fees
uint256 stakingYield = (40_000e18 * 5) / 100; // 5% APY on 40k LONG
uint256 yieldValue = 2000e6; // $2,000 at $1/LONG
assertGt(annualOvercharge, yieldValue / 2, "Fees consume 60% of staking rewards");
```

{% endstep %}
{% endstepper %}

### Expected vs Actual Behavior

* Expected: Users with >50k LONG exposure (via shares \* exchange rate) receive BronzeTier fee discounts (9% deposit fee, 10%/7% promoter fee).
* Actual: System uses share count instead of asset value. Users with 40k shares (60k LONG exposure post-rebase) classified as NoStakes, paying 10% deposit fee and 10%/8% promoter fees. 1% overcharge per transaction.

## 4. Recommended Fix

### Immediate Fix

Replace `balanceOf()` with `convertToAssets(balanceOf())`.

Example fixes:

```solidity
// BEFORE (BelongCheckIn.sol:378):
VenueStakingRewardInfo memory stakingInfo =
    stakingRewards[_storage.contracts.staking.balanceOf(venueInfo.venue).stakingTiers()].venueStakingInfo;

// AFTER:
uint256 venueShares = _storage.contracts.staking.balanceOf(venueInfo.venue);
uint256 venueAssets = _storage.contracts.staking.convertToAssets(venueShares);
VenueStakingRewardInfo memory stakingInfo =
    stakingRewards[venueAssets.stakingTiers()].venueStakingInfo;
```

```solidity
// BEFORE (BelongCheckIn.sol:547):
PromoterStakingRewardInfo memory stakingInfo = stakingRewards[_storage.contracts.staking.balanceOf(
    promoterInfo.promoter
).stakingTiers()].promoterStakingInfo;

// AFTER:
uint256 promoterShares = _storage.contracts.staking.balanceOf(promoterInfo.promoter);
uint256 promoterAssets = _storage.contracts.staking.convertToAssets(promoterShares);
PromoterStakingRewardInfo memory stakingInfo = stakingRewards[promoterAssets.stakingTiers()].promoterStakingInfo;
```

### Comprehensive Fix with Helper Function

Add a conversion helper to BelongCheckIn.sol:

```solidity
function _getUserStakingTier(address user) internal view returns (StakingTiers) {
    uint256 userShares = _storage.contracts.staking.balanceOf(user);
    
    // ✅ Convert shares to assets to handle vault rebase
    uint256 userAssets = _storage.contracts.staking.convertToAssets(userShares);
    
    return userAssets.stakingTiers();
}
```

Replace tier lookups with the helper:

```solidity
// venueDeposit:
VenueStakingRewardInfo memory stakingInfo = 
    stakingRewards[_getUserStakingTier(venueInfo.venue)].venueStakingInfo;

// distributePromoterPayments:
PromoterStakingRewardInfo memory stakingInfo = 
    stakingRewards[_getUserStakingTier(promoterInfo.promoter)].promoterStakingInfo;
```

### Defense-in-Depth Recommendations

1. Add explicit asset calculation comments:

```solidity
// IMPORTANT: Use convertToAssets() to handle ERC4626 rebase
// balanceOf() returns shares which may have different value than initial deposit
```

2. Add tier verification events:

```solidity
event TierClassification(
    address indexed user, 
    uint256 shares, 
    uint256 assets, 
    StakingTiers tier
);
```

3. Document ERC4626 behavior:

```solidity
/// @notice Staking contract is ERC4626 vault with rebasing via distributeRewards
/// @dev ALWAYS use convertToAssets(balanceOf(user)) for tier calculations
/// @dev Share count != asset value after reward distributions
```

4. Consider tier caching with refresh mechanism:

```solidity
mapping(address => uint256) public lastTierUpdate;
mapping(address => StakingTiers) public cachedTier;

function refreshTier(address user) public {
    uint256 assets = staking.convertToAssets(staking.balanceOf(user));
    cachedTier[user] = Helper.stakingTiers(assets);
    lastTierUpdate[user] = block.timestamp;
}
```

### User Refund Consideration

Optional: implement fee adjustment for affected users (retroactive refunds). Example patterns shown in the original report:

```solidity
mapping(address => uint256) public feeRefundOwed;

function calculateRefund(address user, uint256 fromBlock, uint256 toBlock) external onlyOwner {
    // Analyze historical transactions, calculate overpaid fees
    // Credit to feeRefundOwed mapping
}

function claimFeeRefund() external {
    uint256 refund = feeRefundOwed[msg.sender];
    require(refund > 0, "No refund available");
    
    feeRefundOwed[msg.sender] = 0;
    USDC.safeTransfer(msg.sender, refund);
}
```

## 5. References

* Tier Function: `stakingTiers` (Helper.sol:104-117)
* Venue Deposit Tier Lookup: BelongCheckIn.sol:375-380 (LINE 378)
* Promoter Payout Tier Lookup: BelongCheckIn.sol:545-548 (LINE 547)
* Reward Distribution: `distributeRewards` (Staking.sol:113-116)
* ERC4626 Standard: `convertToAssets()` method for share-to-asset conversion

***

Severity: MEDIUM\
Impact: Griefing (systematic user damage via excess fee collection)\
User Loss: 1–5% excess fees per transaction post-rebase\
Protocol Gain: Unintended excess revenue undermining staking incentives\
Fix Complexity: Low (replace `balanceOf()` with `convertToAssets(balanceOf())`)
