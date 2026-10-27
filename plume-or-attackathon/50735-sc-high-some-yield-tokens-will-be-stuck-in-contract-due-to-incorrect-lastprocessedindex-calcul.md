# 50735 sc high some yield tokens will be stuck in contract due to incorrect lastprocessedindex calculation&#x20;

**Submitted on Jul 28th 2025 at 07:04:09 UTC by @maggie for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50735
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

Yield tokens are distributed based on proportion. Due to rounding operations in the calculation, there may be some remaining tokens at the end; those remaining tokens are sent to the last holder. In the contract, those remaining tokens are sent to the holder at `lastProcessedIndex`. If this holder is in the blacklist, the remaining tokens will not be sent and will get stuck in the contract.

### Vulnerability Details

In `ArcToken.sol#distributeYield()`, `lastProcessedIndex` is the index of the last holder, not the last non-blacklist holder. If the last holder is in the blacklist, remaining yield tokens will be stuck in the contract.

Example scenario (7 holders: Owner, Alice, Bob, Charlie, Lily, Lucy, Tom — Tom is blacklisted). Each holds 1e18 tokens. Distributing 1000 yield tokens results in 166 tokens per eligible holder × 6 = 996 distributed; 4 yield tokens remain. The contract attempts to send the leftover 4 tokens to the holder at `lastProcessedIndex` — but if that holder (Tom) is blacklisted, those 4 tokens remain in the contract.

Recommended fix (ensure `lastProcessedIndex` points to the last non-blacklisted holder and use effective supply computed over allowed holders):

{% code title="Suggested patch: distributeYield(...) in ArcToken.sol" %}

```solidity
function distributeYield(
    uint256 amount
) external onlyRole(YIELD_DISTRIBUTOR_ROLE) nonReentrant {
    ArcTokenStorage storage $ = _getArcTokenStorage();

    if (amount == 0) {
        revert ZeroAmount();
    }

    uint256 supply = totalSupply();
    if (supply == 0) {
        revert NoTokensInCirculation();
    }

    address yieldTokenAddr = $.yieldToken;
    if (yieldTokenAddr == address(0)) {
        revert YieldTokenNotSet();
    }

    ERC20Upgradeable yToken = ERC20Upgradeable(yieldTokenAddr);
    yToken.safeTransferFrom(msg.sender, address(this), amount);

    uint256 distributedSum = 0;
    uint256 holderCount = $.holders.length();
    if (holderCount == 0) {
        emit YieldDistributed(0, yieldTokenAddr);
        return;
    }

    uint256 effectiveTotalSupply = 0;
    uint256 lastProcessedIndex = 0;
    for (uint256 i = 0; i < holderCount; i++) {
        address holder = $.holders.at(i);
        if (_isYieldAllowed(holder)) {
            effectiveTotalSupply += balanceOf(holder);
            lastProcessedIndex = i;
        }
    }

    if (effectiveTotalSupply == 0) {
        emit YieldDistributed(0, yieldTokenAddr);
        return;
    }

    for (uint256 i = 0; i < lastProcessedIndex; i++) {
        address holder = $.holders.at(i);

        if (!_isYieldAllowed(holder)) {
            continue;
        }

        uint256 holderBalance = balanceOf(holder);
        if (holderBalance > 0) {
            uint256 share = (amount * holderBalance) / effectiveTotalSupply;
            if (share > 0) {
                yToken.safeTransfer(holder, share);
                distributedSum += share;
            }
        }
    }

    if (holderCount > 0) {
        address lastHolder = $.holders.at(lastProcessedIndex);
        if (_isYieldAllowed(lastHolder)) {
            uint256 lastShare = amount - distributedSum;
            if (lastShare > 0) {
                yToken.safeTransfer(lastHolder, lastShare);
                distributedSum += lastShare;
            }
        }
    }

    emit YieldDistributed(distributedSum, yieldTokenAddr);
}
```

{% endcode %}

## Impact Details

Remaining yield tokens will be left in the contract. Over time, as more distributions occur, accumulated stuck balances will grow and make the available yield pool less than expected.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol#L430>

## Proof of Concept

Add a new file `distribute.t.sol` to `/test` and run the test: forge test --via-ir --match-path test/distribute.t.sol --match-contract DistributeTest --match-test test\_YieldDistribution\_WithBlacklist\_last -vv

Initial yield balance is 7000000000000000000, after distribute, total yield balance is 6999999999999999996.

PoC test code:

{% code title="test/distribute.t.sol" %}

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import { ArcToken } from "../src/ArcToken.sol";
import { ERC20Mock } from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";
import { Test } from "forge-std/Test.sol";
import { console } from "forge-std/console.sol";

// Import necessary restriction contracts and interfaces

import { IRestrictionsRouter } from "../src/restrictions/IRestrictionsRouter.sol";
import { IERC20Errors } from "@openzeppelin/contracts/interfaces/draft-IERC6093.sol";

import { ITransferRestrictions } from "../src/restrictions/ITransferRestrictions.sol";
import { IYieldRestrictions } from "../src/restrictions/IYieldRestrictions.sol";
import { RestrictionsRouter } from "../src/restrictions/RestrictionsRouter.sol";
import { WhitelistRestrictions } from "../src/restrictions/WhitelistRestrictions.sol";
import { YieldBlacklistRestrictions } from "../src/restrictions/YieldBlacklistRestrictions.sol";
import {console} from "forge-std/console.sol";
contract DistributeTest is Test, IERC20Errors {

    ArcToken public token;
    ERC20Mock public yieldToken;
    RestrictionsRouter public router;
    WhitelistRestrictions public whitelistModule;
    YieldBlacklistRestrictions public yieldBlacklistModule;

    address public owner;
    address public alice;
    address public bob;
    address public charlie;
    address public lily;
    address public lucy;
    address public tom;

    uint256 public constant INITIAL_SUPPLY = 7e18;
    uint256 public constant ASSET_VALUATION = 1_000_000e18;
    uint256 public constant TOKEN_ISSUE_PRICE = 100e18;
    uint256 public constant ACCRUAL_RATE_PER_SECOND = 6_342_013_888_889; // ~0.054795% daily
    uint256 public constant TOTAL_TOKEN_OFFERING = 10_000e18;
    uint256 public constant YIELD_AMOUNT = 1000e18;

    event YieldDistributed(uint256 amount, address indexed token);

    // Define module type constants matching ArcToken
    bytes32 public constant TRANSFER_RESTRICTION_TYPE = keccak256("TRANSFER_RESTRICTION");
    bytes32 public constant YIELD_RESTRICTION_TYPE = keccak256("YIELD_RESTRICTION");

    function setUp() public {
        owner = address(this);
        alice = makeAddr("alice");
        bob = makeAddr("bob");
        charlie = makeAddr("charlie");
        lily = makeAddr("lily");
        lucy = makeAddr("lucy");
        tom = makeAddr("tom");
        // Deploy mock yield token
        yieldToken = new ERC20Mock();
        yieldToken.mint(owner, 7e18); // 

        // --- Deploy Infrastructure ---
        // 1. Deploy Router
        router = new RestrictionsRouter();
        router.initialize(owner); // Initialize router with owner as admin

        // 2. Deploy Per-Token Restriction Modules
        whitelistModule = new WhitelistRestrictions();
        whitelistModule.initialize(owner); // transfersAllowed is set to TRUE here by default

        yieldBlacklistModule = new YieldBlacklistRestrictions();
        yieldBlacklistModule.initialize(owner); // Owner manages yield blacklist

        // 3. Register Module Types in Router (optional for this test, but good practice)
        // router.registerModuleType(TRANSFER_RESTRICTION_TYPE, false, address(0));
        // router.registerModuleType(YIELD_RESTRICTION_TYPE, false, address(0));

        // --- Deploy ArcToken ---
        token = new ArcToken();
        token.initialize(
            "Arc Token",
            "ARC",
            INITIAL_SUPPLY,
            address(yieldToken),
            owner, // initial holder
            18, // decimals
            address(router) // router address
        );

        // --- Link Modules to Token ---
        token.setRestrictionModule(TRANSFER_RESTRICTION_TYPE, address(whitelistModule));
        token.setRestrictionModule(YIELD_RESTRICTION_TYPE, address(yieldBlacklistModule));

        // --- Grant MINTER_ROLE to owner (test contract) for minting in tests ---
        token.grantRole(token.MINTER_ROLE(), owner);

        // --- Setup Initial State ---
        // Whitelist addresses using the Whitelist Module
        whitelistModule.addToWhitelist(owner);
        whitelistModule.addToWhitelist(alice);
        whitelistModule.addToWhitelist(bob);
        whitelistModule.addToWhitelist(charlie);
        whitelistModule.addToWhitelist(lily);
        whitelistModule.addToWhitelist(lucy);

        // Now mint tokens after linking modules and whitelisting
        // Note: Initial supply is already minted to owner in initialize
        token.transfer(alice, 1e18); 
        token.transfer(bob, 1e18); 
        token.transfer(charlie, 1e18); 
        token.transfer(lily, 1e18); 
        token.transfer(lucy, 1e18); 
        token.transfer(tom, 1e18); 
    }

 

    function test_YieldDistribution_WithBlacklist_last() public {
        uint256 YIELD_AMOUNT = 1000;

        yieldBlacklistModule.addToBlacklist(tom);

        uint256 ownerInitialYieldBalance = yieldToken.balanceOf(owner); 
        uint256 aliceInitialYieldBalance = yieldToken.balanceOf(alice); 
        uint256 bobInitialYieldBalance = yieldToken.balanceOf(bob); 
        uint256 charlieInitialYieldBalance = yieldToken.balanceOf(charlie); 
        uint256 lilyInitialYieldBalance = yieldToken.balanceOf(lily); 
        uint256 lucyInitialYieldBalance = yieldToken.balanceOf(lucy); 
        console.log("ownerInitialYieldBalance = %s", ownerInitialYieldBalance);
        console.log("aliceInitialYieldBalance = %s", aliceInitialYieldBalance);
        console.log("bobInitialYieldBalance = %s", bobInitialYieldBalance);
        console.log("charlieInitialYieldBalance = %s", charlieInitialYieldBalance);
        console.log("lilyInitialYieldBalance = %s", lilyInitialYieldBalance);
        console.log("lucyInitialYieldBalance = %s", lucyInitialYieldBalance);

        yieldToken.approve(address(token), YIELD_AMOUNT);

        token.distributeYield(YIELD_AMOUNT);
        console.log("YIELD_AMOUNT = %s", YIELD_AMOUNT);

        console.log("yieldToken.balanceOf(owner) = %s", yieldToken.balanceOf(owner));
        console.log("yieldToken.balanceOf(alice) = %s", yieldToken.balanceOf(alice));
        console.log("yieldToken.balanceOf(bob) = %s", yieldToken.balanceOf(bob));
        console.log("yieldToken.balanceOf(charlie) = %s", yieldToken.balanceOf(charlie));
        console.log("yieldToken.balanceOf(lily) = %s", yieldToken.balanceOf(lily));
        console.log("yieldToken.balanceOf(lucy) = %s", yieldToken.balanceOf(lucy));

    }

}
```

{% endcode %}
