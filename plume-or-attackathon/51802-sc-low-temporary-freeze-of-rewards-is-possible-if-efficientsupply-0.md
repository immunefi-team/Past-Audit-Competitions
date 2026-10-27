# 51802 sc low temporary freeze of rewards is possible if efficientsupply 0

**Submitted on Aug 5th 2025 at 21:33:32 UTC by @Santi for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51802
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Permanent freezing rewards

## Description

### Brief/Intro

Rewards tokens may be stuck on the contract. This can happen if the `effectiveSupply` for distribution is equal to 0.

### Vulnerability Details

For yield token distribution, the user with role `YIELD_DISTRIBUTOR_ROLE` must call `ArcToken.distributeYield()` or `ArcToken.distributeYieldWithLimit()`.

The function `ArcToken.distributeYield()` transfers rewards token before the `effectiveTotalSupply == 0` check, which returns from the function if it is true.

Code snippet:

```solidity
function distributeYield(
        uint256 amount
    ) external onlyRole(YIELD_DISTRIBUTOR_ROLE) nonReentrant {
        ArcTokenStorage storage $ = _getArcTokenStorage();

.... 

        ERC20Upgradeable yToken = ERC20Upgradeable(yieldTokenAddr);
        yToken.safeTransferFrom(msg.sender, address(this), amount); // <- transfer yield tokens to contract

        uint256 distributedSum = 0;
        uint256 holderCount = $.holders.length();
        if (holderCount == 0) {
            emit YieldDistributed(0, yieldTokenAddr);
            return; 
        }

        uint256 effectiveTotalSupply = 0;
        for (uint256 i = 0; i < holderCount; i++) {
            address holder = $.holders.at(i);
            if (_isYieldAllowed(holder)) {
                effectiveTotalSupply += balanceOf(holder); // <- calculate effectiveTotalSupply
            }
        }

        if (effectiveTotalSupply == 0) {
            emit YieldDistributed(0, yieldTokenAddr);
            return; // <- return  from function if effectiveSupply == 0, but contract stores yield tokens.
        }

        
...
    }
```

It is incorrect to transfer yield tokens before the `effectiveTotalSupply` check. If `effectiveSupply` is equal to zero (for example, if all holders are blacklisted for yield distribution), the yield tokens will be stuck on the contract.

For comparison, `ArcToken.distributeYieldWithLimit()` checks that `effectiveSupply != 0` before tokens transfer. Link:\
<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L518-L528>

## Impact Details

It is possible to lose all yield tokens that the `YIELD_DISTRIBUTOR_ROLE` attempted to distribute. This is a rare case because:

* All token holders must be ineligible for yield tokens (e.g., blacklisted).
* `YIELD_DISTRIBUTOR_ROLE` must call this function when all holders are ineligible.

Recovery is possible via `ArcToken.distributeYieldWithLimit()`, but with constraints:

* A new token holder must appear, and the new token holder must be at index > 0 (for example, it can be admin).
* The new token holder must not be blacklisted.

Then `YIELD_DISTRIBUTOR_ROLE` can call `ArcToken.distributeYieldWithLimit()` and pass the correct `totalAmount`, `startIndex`, `maxHolders`.

Recovery is possible, but yield tokens will be temporarily frozen and distribution will be incorrect. That is why this finding is rated Low and `ArcToken.distributeYield()` should handle this case correctly.

## References

* distributeYield() function: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L388>

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import { ArcToken } from "../src/ArcToken.sol";
import { ERC20Mock } from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";
import { Test } from "forge-std/Test.sol";
import { console } from "forge-std/console.sol";

import { RestrictionsRouter } from "../src/restrictions/RestrictionsRouter.sol";
import { WhitelistRestrictions } from "../src/restrictions/WhitelistRestrictions.sol";
import { YieldBlacklistRestrictions } from "../src/restrictions/YieldBlacklistRestrictions.sol";

contract ArcTokenRewardsLostTest is Test {

    ArcToken public token;
    ERC20Mock public yieldToken;
    RestrictionsRouter public router;
    WhitelistRestrictions public whitelistModule;
    YieldBlacklistRestrictions public yieldBlacklistModule;

    address public owner;

    uint256 public constant INITIAL_SUPPLY = 1000e18;
    uint256 public constant YIELD_AMOUNT = 1000e18;

    bytes32 public constant TRANSFER_RESTRICTION_TYPE = keccak256("TRANSFER_RESTRICTION");
    bytes32 public constant YIELD_RESTRICTION_TYPE = keccak256("YIELD_RESTRICTION");

    function setUp() public {
        owner = address(this);

        // Deploy tokens
        yieldToken = new ERC20Mock();
        yieldToken.mint(owner, 1_000_000e18);

        // Deploy infrastructure
        router = new RestrictionsRouter();
        router.initialize(owner);

        whitelistModule = new WhitelistRestrictions();
        whitelistModule.initialize(owner);

        yieldBlacklistModule = new YieldBlacklistRestrictions();
        yieldBlacklistModule.initialize(owner);

        // Deploy ArcToken
        token = new ArcToken();
        token.initialize(
            "Arc Token",
            "ARC",
            INITIAL_SUPPLY,
            address(yieldToken),
            owner,
            18,
            address(router)
        );

        // Link modules
        token.setRestrictionModule(TRANSFER_RESTRICTION_TYPE, address(whitelistModule));
        token.setRestrictionModule(YIELD_RESTRICTION_TYPE, address(yieldBlacklistModule));

        // Whitelist owner
        whitelistModule.addToWhitelist(owner);
    }

    function test_AllHoldersBlacklisted_100PercentLoss() public {
        // Blacklist all holders
        yieldBlacklistModule.addToBlacklist(owner);

        // Try to distribute rewards
        yieldToken.approve(address(token), YIELD_AMOUNT);
        
        uint256 contractBalanceBefore = yieldToken.balanceOf(address(token));
        token.distributeYield(YIELD_AMOUNT);
        uint256 contractBalanceAfter = yieldToken.balanceOf(address(token));
        
        uint256 lostAmount = contractBalanceAfter - contractBalanceBefore;
        
        console.log("Distributed yield amount:", YIELD_AMOUNT);
        console.log("Lost rewards stuck in contract:", lostAmount);
        console.log("Loss percentage:", (lostAmount * 100) / YIELD_AMOUNT);
        
        // Assert that ALL rewards are stuck
        assertEq(lostAmount, YIELD_AMOUNT, "All rewards should be stuck in contract");
    }

}
```
