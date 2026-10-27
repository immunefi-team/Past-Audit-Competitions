# 52669 sc low token minting is blocked for whitelisted addresses when transfersallowed is false

**Submitted on Aug 12th 2025 at 11:04:26 UTC by @magtentic for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52669
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

When `transfersAllowed` is disabled, the whitelist restriction module still allows whitelisted users to transfer tokens. However, minting new tokens to those same whitelisted users fails.

This occurs because `address(0)` (used as the source address for minting) is not and cannot be whitelisted, which causes the transfer validation to fail even if the recipient is whitelisted and the caller has the `MINTER_ROLE`.

### Vulnerability Details

Whitelisted users cannot have tokens minted to them because `address(0)` will never be whitelisted. Within the ArcToken contract, `_update` checks whether a `specificRestrictionModules` is configured and uses it to validate the transfer:

```solidity
function _update(address from, address to, uint256 amount) internal virtual override {
    ...
    address specificTransferModule = $.specificRestrictionModules[RestrictionTypes.TRANSFER_RESTRICTION_TYPE];
    if (specificTransferModule != address(0)) {
        transferAllowed =
            transferAllowed && ITransferRestrictions(specificTransferModule).isTransferAllowed(from, to, amount);
    }
    ...
    if (!transferAllowed) {
        revert TransferRestricted();
    }
    ...
}
```

If `transfersAllowed` is false, a transfer will only be allowed if the restriction module explicitly approves it.

In the `WhitelistRestrictions` module, approval is based on whether both `from` and `to` are whitelisted. Attempts to whitelist `address(0)` are blocked:

```solidity
function addToWhitelist(
    address account
) external onlyRole(MANAGER_ROLE) {
    if (account == address(0)) {
        revert InvalidAddress();
    }
    ...
}
```

And `batchAddToWhitelist` explicitly skips zero address:

```solidity
function batchAddToWhitelist(
    address[] calldata accounts
) external onlyRole(MANAGER_ROLE) {
    WhitelistStorage storage ws = _getWhitelistStorage();

    for (uint256 i = 0; i < accounts.length; i++) {
        address account = accounts[i];

        if (account == address(0)) {
            continue; // Skip zero address
        }
    ...
```

Because `address(0)` will never be whitelisted, `isTransferAllowed(address(0), to, amount)` returns false even when `to` is whitelisted and the caller has `MINTER_ROLE`. That causes mint operations to revert with `TransferRestricted()`.

### Impact Details

* Minting tokens to whitelisted addresses fails when `transfersAllowed` is disabled, even when the caller has the `MINTER_ROLE`.
* This unintentionally blocks minting in scenarios where the restriction system is otherwise configured to allow it.
* It creates inconsistency between transfer and mint semantics for whitelisted users.

### References

This is a resubmission of #49716 with an updated PoC and explanation. The project's own tests show expectations that minting/burning will work with temporary whitelist toggles (see the ArcToken test suite referenced below):

* Test suite reference: <https://github.com/plumenetwork/contracts/blob/main/arc/test/ArcToken.t.sol>

An example comment from tests:

```
// 1. Mint tokens to nonWhitelisted1 (requires whitelisting temporarily or transfersAllowed=true)
```

This implies tests expect temporary whitelisting or disabled transfer restrictions to allow minting — inconsistent with the observed behavior when `transfersAllowed` is false.

## Proof of Concept

{% stepper %}
{% step %}

### Steps to reproduce (summary)

* Attempt to add `address(0)` to the whitelist using `addToWhitelist` — this reverts with `InvalidAddress()`.
* Add a user (Alice) to the whitelist and validate `isWhitelisted(alice)`.
* Disable transfers via the whitelist module (`setTransfersAllowed(false)`), confirming transfers are restricted.
* Attempt to mint tokens to Alice — mint reverts with `TransferRestricted()` because `isTransferAllowed(address(0), alice, amount)` is false.
  {% endstep %}
  {% endstepper %}

<details>

<summary>Proof-of-Concept test (Solidity/Foundry)</summary>

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

contract ArcTokenTest is Test, IERC20Errors {

    ArcToken public token;
    ERC20Mock public yieldToken;
    RestrictionsRouter public router;
    WhitelistRestrictions public whitelistModule;
    YieldBlacklistRestrictions public yieldBlacklistModule;

    address public owner;
    address public alice;
    address public bob;
    address public charlie;
    address public zero;

    uint256 public constant INITIAL_SUPPLY = 1000e18;
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
        zero = address(0);

        // Deploy mock yield token
        yieldToken = new ERC20Mock();
        yieldToken.mint(owner, 1_000_000e18); // Initial balance for owner is 1e24

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
        token.grantRole(token.BURNER_ROLE(), owner);

        // --- Setup Initial State ---
        // Whitelist addresses using the Whitelist Module
        whitelistModule.addToWhitelist(owner);
        whitelistModule.addToWhitelist(alice);
        whitelistModule.addToWhitelist(bob);
        whitelistModule.addToWhitelist(charlie);

        // Now mint tokens after linking modules and whitelisting
        // Note: Initial supply is already minted to owner in initialize
        token.transfer(alice, 100e18); // Owner: 900e18, Alice: 100e18
    }

    // ============ Initialization Tests ============

    function test_Initialization() public {
        assertEq(token.name(), "Arc Token");
        assertEq(token.symbol(), "ARC");
        assertEq(token.decimals(), 18);
        assertEq(token.balanceOf(owner), INITIAL_SUPPLY - 100e18);
        assertEq(token.balanceOf(alice), 100e18);
        assertTrue(whitelistModule.isWhitelisted(alice)); // Check via module
        assertTrue(whitelistModule.transfersAllowed()); // Check default is true
    }

    // Test minting from whitelisted address
    function test_Minting_WhiteListed_Failure() public {
        //Add 0 address to whitelist
        vm.prank(owner);
        vm.expectRevert(WhitelistRestrictions.InvalidAddress.selector);
        whitelistModule.addToWhitelist(zero);

        //Validate that Alice is whitelisted
        assertTrue(whitelistModule.isWhitelisted(alice));

        //Explicitly restrict transfers
        whitelistModule.setTransfersAllowed(false);
        assertFalse(whitelistModule.transfersAllowed());

        //Test Minting for Alice to see a failure
        vm.prank(owner);
        vm.expectRevert(ArcToken.TransferRestricted.selector);
        token.mint(alice, 50e18);

        //Set transfers back to allowed for other tests
        whitelistModule.setTransfersAllowed(true);
    }
}
```

</details>
