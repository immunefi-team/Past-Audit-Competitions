# 52890 sc low no recipient yield distribution locks yield tokens on arctoken efftotal 0&#x20;

**Submitted on Aug 14th 2025 at 02:11:17 UTC by @nitinaimshigh for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #52890
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Temporary freezing of funds for at least 24 hours

## Description

### Brief / Intro

ArcToken.distributeYield pulls the yield token from the distributor (treasury) before it computes the “eligible supply.” If every holder is ineligible (e.g., all blacklisted/sanctioned so effTotal == 0), no recipients are credited, yet the transferred amount remains stranded on the ArcToken contract address with no withdrawal mechanism. In production, anyone with YIELD\_DISTRIBUTOR\_ROLE (or a compromised treasury) could permanently lock arbitrary amounts of the yield token on ArcToken by calling distributeYield during an all-ineligible state, resulting in irreversible loss of unclaimed yield and potential depletion of the protocol’s yield funds.

Relevant function excerpt:

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
    for (uint256 i = 0; i < holderCount; i++) {
        address holder = $.holders.at(i);
        if (_isYieldAllowed(holder)) {
            effectiveTotalSupply += balanceOf(holder);
        }
    }

    if (effectiveTotalSupply == 0) {
        emit YieldDistributed(0, yieldTokenAddr);
        return;
    }

    uint256 lastProcessedIndex = holderCount > 0 ? holderCount - 1 : 0;
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

## Vulnerability Details

Root cause: In ArcToken.distributeYield, the contract pulls the yield token from the distributor (treasury) before validating that there is any eligible supply to receive it.

When all holders are ineligible (e.g., every account fails either YieldBlacklistRestrictions or the global isYieldAllowed check), the function proceeds to compute an effective total of 0, distributes to no one, and then returns — leaving the transferred amount stranded on the ArcToken contract address. There is no refund path so the funds become permanently locked.

What the execution shows:

* The caller approves and distributeYield(amount) is invoked.
* The first action is pulling funds:
  * ERC20Mock::transferFrom(treasury, ArcToken, 5e20) -> true
* All recipients are ineligible (isYieldAllowed(...) -> false for owner, alice, bob, charlie).
* The function emits:
  * YieldDistributed(amount: 0, token: ERC20Mock)
* Post-state: the yield token balance of ArcToken equals the whole amount, e.g.:
  * ERC20Mock::balanceOf(ArcToken) -> 500000000000000000000
* This results in a retained balance on the ArcToken contract with no mechanism to retrieve it.

## Impact Details

What can be lost? The entire yield-asset amount passed to distributeYield(amount) when the effective eligible supply is 0. Because the function pulls tokens from the distributor (e.g., treasury) before checking eligibility and credits no one when effTotal == 0, the full amount becomes permanently stranded at ArcToken.

If the treasury holds 5,000,000 units of the yield token and has unlimited approval to ArcToken, then a single call with amount = 5,000,000 while everyone is ineligible locks the full 5,000,000 on ArcToken.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L388-L460>

## Proof of Concept

The following Forge test demonstrates the issue. The second test makes every account ineligible, calls distributeYield from the treasury and asserts that no one received funds while the ArcToken contract retained the tokens (unexpected behavior).

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import { Test } from "forge-std/Test.sol";
import { console } from "forge-std/console.sol";

import { ArcToken } from "../src/ArcToken.sol";
import { ERC20Mock } from "openzeppelin-contracts/contracts/mocks/token/ERC20Mock.sol";

import { RestrictionsRouter } from "../src/restrictions/RestrictionsRouter.sol";
import { WhitelistRestrictions } from "../src/restrictions/WhitelistRestrictions.sol";
import { YieldBlacklistRestrictions } from "../src/restrictions/YieldBlacklistRestrictions.sol";

import { IYieldRestrictions } from "../src/restrictions/IYieldRestrictions.sol";
import { RestrictionTypes } from "../src/restrictions/RestrictionTypes.sol";

/// @dev Global module mock that supports BOTH transfer + yield checks.
contract MockGlobalSanctions is IYieldRestrictions {
    mapping(address => bool) public sanctioned;

    function setSanctioned(address who, bool isSanctioned) external {
        sanctioned[who] = isSanctioned;
    }

    // Used by GLOBAL_SANCTIONS_TYPE for transfers (router calls this during mint/transfer/burn)
    function isTransferAllowed(address from, address to, uint256 /*amount*/) external view returns (bool) {
        if (sanctioned[from]) return false;
        if (sanctioned[to]) return false;
        return true;
    }

    // Used by yield distribution path
    function isYieldAllowed(address account) external view override returns (bool) {
        return !sanctioned[account];
    }
}

contract InvariantYieldEligibilityTest is Test {
    // System under test
    ArcToken internal token;
    ERC20Mock internal yieldToken;
    RestrictionsRouter internal router;
    WhitelistRestrictions internal transferWhitelist;
    YieldBlacklistRestrictions internal yieldBlacklist;
    MockGlobalSanctions internal globalSanctions;

    // Actors
    address internal owner;
    address internal alice;
    address internal bob;
    address internal charlie;
    address internal treasury; // payer of yield; not a holder

    // Roles (replicated from ArcToken)
    bytes32 constant ADMIN_ROLE               = keccak256("ADMIN_ROLE");
    bytes32 constant YIELD_MANAGER_ROLE       = keccak256("YIELD_MANAGER_ROLE");
    bytes32 constant YIELD_DISTRIBUTOR_ROLE   = keccak256("YIELD_DISTRIBUTOR_ROLE");
    bytes32 constant MINTER_ROLE              = keccak256("MINTER_ROLE");

    // Module type IDs — constants, not functions.
    bytes32 constant TRANSFER_RESTRICTION_TYPE = RestrictionTypes.TRANSFER_RESTRICTION_TYPE;
    bytes32 constant YIELD_RESTRICTION_TYPE    = RestrictionTypes.YIELD_RESTRICTION_TYPE;
    bytes32 constant GLOBAL_SANCTIONS_TYPE     = RestrictionTypes.GLOBAL_SANCTIONS_TYPE;

    function setUp() public {
        owner = address(this);
        alice = makeAddr("alice");
        bob = makeAddr("bob");
        charlie = makeAddr("charlie");
        treasury = makeAddr("treasury");

        // 1) Deploy + init router
        router = new RestrictionsRouter();
        router.initialize(address(this));
        router.registerModuleType(TRANSFER_RESTRICTION_TYPE, false, address(0));
        router.registerModuleType(YIELD_RESTRICTION_TYPE,    false, address(0));

        globalSanctions = new MockGlobalSanctions();
        router.registerModuleType(GLOBAL_SANCTIONS_TYPE, true, address(globalSanctions));

        // 2) Deploy modules
        transferWhitelist = new WhitelistRestrictions();
        transferWhitelist.initialize(address(this));
        yieldBlacklist = new YieldBlacklistRestrictions();
        yieldBlacklist.initialize(address(this));

        // Whitelist our holders for transfers
        transferWhitelist.addToWhitelist(owner);
        transferWhitelist.addToWhitelist(alice);
        transferWhitelist.addToWhitelist(bob);
        transferWhitelist.addToWhitelist(charlie);

        // 3) Deploy ArcToken and wire router + modules
        token = new ArcToken();
        token.initialize(
            "Arc Token",
            "ARC",
            0,
            address(0),
            address(0),
            18,
            address(router)
        );

        token.setRestrictionModule(TRANSFER_RESTRICTION_TYPE, address(transferWhitelist));
        token.setRestrictionModule(YIELD_RESTRICTION_TYPE,    address(yieldBlacklist));

        // 4) Roles & mint balances
        token.grantRole(MINTER_ROLE, address(this));
        token.mint(owner,   500e18);
        token.mint(alice,   300e18);
        token.mint(bob,     200e18);
        token.mint(charlie, 100e18);

        // 5) Set yield token & fund treasury (payer)
        yieldToken = new ERC20Mock();
        yieldToken.mint(treasury, 10_000e18);
        token.setYieldToken(address(yieldToken));
        // Distributor role to treasury so it can call distributeYield
        if (!token.hasRole(YIELD_DISTRIBUTOR_ROLE, treasury)) {
            token.grantRole(YIELD_DISTRIBUTOR_ROLE, treasury);
        }
    }

    /// @dev Helper to mark eligibility across both modules (owner + 3 holders).
    function _setEligibility(
        bool ownerElig,
        bool aliceElig,
        bool bobElig,
        bool charlieElig
    ) internal {
        // Yield blacklist (per-token module): false => ineligible
        _setBlacklist(owner,   !ownerElig);
        _setBlacklist(alice,   !aliceElig);
        _setBlacklist(bob,     !bobElig);
        _setBlacklist(charlie, !charlieElig);

        // Global sanctions (router-provided global module): true => sanctioned => ineligible
        globalSanctions.setSanctioned(owner,   !ownerElig);
        globalSanctions.setSanctioned(alice,   !aliceElig);
        globalSanctions.setSanctioned(bob,     !bobElig);
        globalSanctions.setSanctioned(charlie, !charlieElig);
    }

    function _setBlacklist(address who, bool blacklisted) internal {
        bool currentlyAllowed = yieldBlacklist.isYieldAllowed(who);
        if (blacklisted && currentlyAllowed) {
            yieldBlacklist.addToBlacklist(who);
        } else if (!blacklisted && !currentlyAllowed) {
            yieldBlacklist.removeFromBlacklist(who);
        }
    }

    /// @notice Core check: only eligible holders receive yield, pro-rata by ARC.
    function test_DistributeYield_UsesEligibleSupplyOnly() public {
        // Eligible: owner + alice; ineligible: bob + charlie
        _setEligibility(true, true, false, false);

        uint256 amount = 1_000e18;

        // Snapshot pre-state
        address[4] memory H = [owner, alice, bob, charlie];
        uint256[4] memory preArc;
        uint256[4] memory preYield; // balances of the yield token
        bool[4] memory elig;
        uint256 effTotal;

        for (uint256 i = 0; i < 4; ++i) {
            preArc[i] = token.balanceOf(H[i]);
            preYield[i] = yieldToken.balanceOf(H[i]);
            bool ok1 = yieldBlacklist.isYieldAllowed(H[i]);
            bool ok2 = globalSanctions.isYieldAllowed(H[i]);
            elig[i] = (ok1 && ok2);
            if (elig[i]) effTotal += preArc[i];
        }
        assertGt(effTotal, 0, "effTotal must be > 0 for this test");

        // Approve from treasury and call distribute from treasury
        vm.startPrank(treasury);
        yieldToken.approve(address(token), amount);
        token.distributeYield(amount);
        vm.stopPrank();

        // Post-state assertions (on yieldToken balances)
        uint256 allocated;
        for (uint256 i = 0; i < 4; ++i) {
            uint256 postYield = yieldToken.balanceOf(H[i]);
            if (!elig[i]) {
                assertEq(postYield, preYield[i], "ineligible received yieldToken");
            } else {
                uint256 expectedGain = (amount * preArc[i]) / effTotal; // floor pro-rata
                uint256 actualGain = postYield - preYield[i];
                assertEq(actualGain, expectedGain, "eligible yieldToken gain mismatch");
                allocated += actualGain;
            }
        }

        assertLe(allocated, amount, "allocated > amount");
        assertLt(amount - allocated, 2, "excess rounding dust");
        // Sanity: the token contract shouldn't keep leftover yield (unless design says otherwise)
        assertEq(yieldToken.balanceOf(address(token)), 0, "contract retained yield unexpectedly");
    }

    /// @notice effTotal == 0 -> distribution should not credit anyone.
    function test_DistributeYield_NoEligibleSupplyCreditsNoOne() public {
        // Make EVERY account (including owner) ineligible
        _setEligibility(false, false, false, false);

        uint256 amount = 500e18;

        uint256 preOwner   = yieldToken.balanceOf(owner);
        uint256 preAlice   = yieldToken.balanceOf(alice);
        uint256 preBob     = yieldToken.balanceOf(bob);
        uint256 preCharlie = yieldToken.balanceOf(charlie);

        vm.startPrank(treasury);
        yieldToken.approve(address(token), amount);
        token.distributeYield(amount);
        vm.stopPrank();

        assertEq(yieldToken.balanceOf(owner),   preOwner,   "owner yield changed");
        assertEq(yieldToken.balanceOf(alice),   preAlice,   "alice yield changed");
        assertEq(yieldToken.balanceOf(bob),     preBob,     "bob yield changed");
        assertEq(yieldToken.balanceOf(charlie), preCharlie, "charlie yield changed");
        assertEq(yieldToken.balanceOf(address(token)), 0, "contract retained yield unexpectedly");
    }
}
```

Run the test: forge test --match-test test\_DistributeYield\_NoEligibleSupplyCreditsNoOne -vvvv
