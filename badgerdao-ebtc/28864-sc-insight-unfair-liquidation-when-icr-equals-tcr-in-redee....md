# 28864 - \[SC - Insight] Unfair Liquidation when ICR equals TCR in redee...

Submitted on Feb 29th 2024 at 06:08:41 UTC by @Saediek for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28864

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/LiquidationLibrary.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

According to the docs in recovery Mode liquidation can only occur when the ICR\<TCR but the implementation allows liquidation to occur on an ICR which is same as the TCR.

## Vulnerability Details

According to the docs i quote "Any CDP whose ICR is below the TCR can be liquidated after the conclusion of the Grace Period. In a way, during Recovery Mode, the TCR becomes the liquidation threshold. This incentivizes rapid repayment or adjustment of CDPs with low ICR." But according to the liquidation it's possible for an ICR which equals TCR to be liquidated.This issue is caused due to the use of the wrong operator in the EBTCBase contract{<https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/Dependencies/EbtcBase.sol#L126}> which is used in the LiquidationLibrary {<https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/LiquidationLibrary.sol#L83}> ,the \_checkICRAgainstLiqThreshold() in the EBTCBase makes an internal call \_checkICRAgainstTCR() where the operator <= is used instead of the < //Snippet function \_checkICRAgainstTCR(uint256 \_icr, uint \_tcr) internal view returns (bool) { return \_icr <= \_tcr; }

## Impact Details

This issue could give space for unfair liquidation, less trust in the eBTC protocol due to discrepancies between the docs and codebase. The codebase is supposed to adhere strictly to the documentation of a protocol.

## References

<https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/LiquidationLibrary.sol#L83> <https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/Dependencies/EbtcBase.sol#L126> <https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/Dependencies/EbtcBase.sol#L142>

## Proof of Concept

\--Check Code for how to run /\*\*

* SPDX-License-Identifier:UNLICENSED
* @author <Saediek@proton.me> \*/ import "@ebtc-contracts/BorrowerOperations.sol"; import "@ebtc-contracts/ActivePool.sol"; import "@ebtc-contracts/CollSurplusPool.sol"; import "@ebtc-contracts/SortedCdps.sol"; import "@ebtc-contracts/EBTCToken.sol"; import "@ebtc-contracts/PriceFeed.sol"; import "forge-std/Test.sol"; import "@ebtc-contracts/CdpManager.sol"; import "@ebtc-contracts/FeeRecipient.sol"; import "@ebtc-contracts/Dependencies/Auth.sol"; //sub-dependencies import "@ebtc-contracts/LiquidationLibrary.sol";

pragma solidity ^0.8; /\*\*

* To run the test create a new folder in foundry\_test and then name it anything of your choice
* in your remappings.txt set "@ebtc-contracts/=contracts/"
* run forge test --test-match test\_liquidation\_at\_tcr -vvv \*/

contract BorrowersOperationTest is Test { BorrowerOperations private borrowerOperations; //--done FeeRecipient private feeReceipient; //-done ActivePool private activepool; //--done EBTCToken private cdpToken; //-done PriceFeed private priceFeed; //--done SortedCdps private sortedcdps; //--done CdpManager private cdpManager; //--done MockAuthority private auth; // --done LiquidationLibrary private liquidationLib; //--done CollSurplusPool surplusPool; //--done address eth\_btc = 0xAc559F25B1619171CbC396a50854A3240b6A4e99; address steth\_eth = 0x86392dC19c0b719886221c78AB11eb8Cf5c52812;

```
address private steth = 0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84;
//USERS WHALES
address userA = 0x5F6AE08B8AeB7078cf2F96AFb089D7c9f51DA47d;
address userB = 0xE942cDd0AF66aB9AB06515701fa3707Ec7deB93e;
address userC = 0x47176B2Af9885dC6C4575d4eFd63895f7Aaa4790;
address userD = 0x18709E89BD403F470088aBDAcEbE86CC60dda12e;
address liquidator = makeAddr("LIQUIDATOR");

constructor() {
    //fork ethereum Mainnet

    //deploy all necessary contracts needed
    vm.createSelectFork(vm.envString("ETH_RPC_URL"));
    __deploy();
    //labels for debugging
    vm.label(address(borrowerOperations), "BORROWER-OPERATIONS");
    vm.label(address(cdpManager), "CDP-MANAGER");
    vm.label(address(liquidationLib), "LIQUIDATION-LIBRARY");
    vm.label(address(surplusPool), "SURPLUS-POOL");
    vm.label(address(priceFeed), "PRICE-FEED");
    vm.label(address(feeReceipient), "FEE-RECEIPIENT");
    vm.label(address(cdpToken), "EBTC-TOKEN");
    vm.label(address(auth), "AUTHORITY");
    vm.label(address(sortedcdps), "SORTED-CDPS");
    vm.label(address(activepool), "ACTIVE-POOL");
    vm.label(userA, "USER-A");
    vm.label(userB, "USER-B");
    vm.label(userC, "USER-C");
    vm.label(userD, "USER-D");
}

function __deploy() internal {
    /**
     * Some contracts are deployed with address(0) as params and then a setParams function is added with the immutable variables modified to mutable
     * This done because of intertwined dependencies i.e(Contract A requires Contract B as a contructor param and Contract B require Contract A as a constructor argument)
     * So it's just a work around
     */
    auth = new MockAuthority();

    priceFeed = new PriceFeed(address(0), address(auth), steth_eth, eth_btc, true);

    feeReceipient = new FeeRecipient(address(this), address(auth));
    __mock_authority();
    vm.mockCall(
        address(borrowerOperations),
        abi.encodeWithSignature("feeRecipientAddress()"),
        abi.encode(address(feeReceipient))
    );
    activepool = new ActivePool(
        address(borrowerOperations),
        address(cdpManager),
        steth,
        address(surplusPool)
    );

    borrowerOperations = new BorrowerOperations(
        address(cdpManager),
        address(activepool),
        address(surplusPool),
        address(priceFeed),
        address(sortedcdps),
        address(cdpToken),
        address(feeReceipient),
        steth
    );
    cdpToken = new EBTCToken(address(cdpManager), address(borrowerOperations), address(auth));
    surplusPool = new CollSurplusPool(
        address(borrowerOperations),
        address(cdpManager),
        address(activepool),
        steth
    );
    sortedcdps = new SortedCdps(0, address(cdpManager), address(borrowerOperations));
    liquidationLib = new LiquidationLibrary(
        address(borrowerOperations),
        address(surplusPool),
        address(cdpToken),
        address(sortedcdps),
        address(activepool),
        address(priceFeed),
        steth
    );
    cdpManager = new CdpManager(
        address(liquidationLib),
        address(auth),
        address(borrowerOperations),
        address(surplusPool),
        address(cdpToken),
        address(sortedcdps),
        address(activepool),
        address(priceFeed),
        steth
    );
    __clear_all_mocked_calls();
    /**
     * added functionalities due to intertwined dependencies
     *  i.e(Contract A require Contract B address for constructor arguments and Contract B requires Contract A for constructor arguments)
     *which to deploy first ??
     */
    __reinitializeParams();
}

function __reinitializeParams() internal {
    activepool.setParams(
        address(cdpManager),
        address(surplusPool),
        address(borrowerOperations),
        address(feeReceipient)
    );

    borrowerOperations.setParams(
        address(activepool),
        address(cdpManager),
        address(surplusPool),
        address(priceFeed),
        address(sortedcdps),
        address(cdpToken),
        address(feeReceipient),
        steth
    );
    surplusPool.setParams(address(cdpManager));
    cdpToken.setParams(address(cdpManager));
    sortedcdps.setParams(address(cdpManager));
}

function __mock_authority() internal {
    vm.mockCall(
        address(cdpManager),
        abi.encodeWithSignature("authority()"),
        abi.encode(address(auth))
    );
}

function __clear_all_mocked_calls() internal {
    vm.clearMockedCalls();
}

function __openCdpAndTakeLoan(address _user, uint256 amount) internal returns (bytes32 id) {
    vm.startPrank(_user);

    IERC20(steth).approve(address(borrowerOperations), amount + 0.2e18);
    id = borrowerOperations.openCdp(1e18, bytes32(uint256(0)), bytes32(0), amount + 0.2e18);

    vm.stopPrank();
}

/**
 * A test to show a scenario whereby when the System enters Recovery Mode
 * A Cdp with ICR==TCR could be liquidated
 * This obviously contradicts the documentation which says
 * "Any CDP whose ICR is below the TCR can be liquidated "
 * Which means that an ICR ==TCR is souldn't be liquidatable
 */
function test_liquidation_at_tcr() external {
    //Similar cdps are created to ensure ICR==TCR

    bytes32 _cpid = __openCdpAndTakeLoan(userA, _calculate_collAmount(1.45e18));

    bytes32 _cdpid = __openCdpAndTakeLoan(userB, _calculate_collAmount(1.45e18));
    //Anything happens which causes the steth/btc price to dip i.e(Increase in the price of BTC) just by 15%
    _simulate_dropPrice();

    //log users new debt after price drops
    uint256 ICR_AFTER_DIP = cdpManager.getCachedICR(_cpid, priceFeed.fetchPrice());
    console.log("users ICR after drop::{%s}", ICR_AFTER_DIP);
    uint256 TCR_AFTER_DIP = cdpManager.getCachedTCR(priceFeed.fetchPrice());
    console.log("TCR after dip::{%s}", TCR_AFTER_DIP);
    assertEq(ICR_AFTER_DIP, TCR_AFTER_DIP);
    //Liquidator Sees the ICR==TCR
    deal(address(cdpToken), liquidator, 1e18);
    //Attempt to liquidate the cdpId
    _setGracePeriod();
    _liquidate(_cpid);
}

function _liquidate(bytes32 _cpid) internal {
    vm.startPrank(liquidator);
    cdpManager.liquidate(_cpid);
    vm.stopPrank();
}

function _setGracePeriod() internal {
    //bypass the grace period
    vm.startPrank(address(borrowerOperations));
    //tcr doesn't matter since it's just emitted
    uint256 startTimestamp = block.timestamp;
    console.log("timestamp before warp", startTimestamp);
    cdpManager.notifyStartGracePeriod(0);
    skip(16 minutes);
    console.log("timestamp after warp", block.timestamp);
    vm.stopPrank();
}

function _simulate_dropPrice() internal {
    //Simulate a 15% dip in the price of steth_btc
    uint256 currentPrice = priceFeed.fetchPrice();
    uint256 newPrice = (currentPrice * 850) / 1000;
    //start Mock call with price discounted by 15%
    vm.mockCall(
        address(priceFeed),
        abi.encodeWithSignature("fetchPrice()"),
        abi.encode(newPrice)
    );
}

//returns steth deposit amount for 1e18 eBTC
function _calculate_collAmount(uint256 targetICR) internal returns (uint256) {
    return ((1e18 * targetICR) / priceFeed.fetchPrice());
}
```

}

contract MockAuthority { function canCall(address user, address target, bytes4 functionSig) external view returns (bool) { return true; } }
