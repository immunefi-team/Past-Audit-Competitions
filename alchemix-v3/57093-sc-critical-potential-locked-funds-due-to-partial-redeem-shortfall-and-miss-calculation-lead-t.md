# 57093 sc critical potential locked funds due to partial redeem shortfall and miss calculation lead to user loss their myt token forever&#x20;

**Submitted on Oct 23rd 2025 at 12:28:40 UTC by @aua\_oo7 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57093
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary:

The redeemption incorrect logic can permanently lock a portion of yield tokens (myt) inside the Transmuter contract when `totalYield < scaledTransmuted` due to redeem shortfalls, lead to permanent loss for user.

## Description:

When transmuter yield token balance is less than user transmuted amount the tranmuter call redeem function in AlchemistV3 and redeem myt token. After redeeming from the AlchemistV3 the available amount myt token only send to user according to transmuted amount the shortfall amount is not transfer to user and user loss it forever due to wrong calculation, the code sets:

```javascript
function claimRedemption(uint256 id) external {
        StakingPosition storage position = _positions[id];

        if (position.maturationBlock == 0) {
            revert PositionNotFound();
        }

        if (position.startBlock == block.number) {
            revert PrematureClaim();
        }

        uint256 transmutationTime = position.maturationBlock - position.startBlock;
        uint256 blocksLeft = position.maturationBlock > block.number ? position.maturationBlock - block.number : 0;
        uint256 rounded = position.amount * blocksLeft / transmutationTime + (position.amount * blocksLeft % transmutationTime == 0 ? 0 : 1);
        uint256 amountNottransmuted = blocksLeft > 0 ? rounded : 0;
        uint256 amountTransmuted = position.amount - amountNottransmuted;

        if (_requireOwned(id) != msg.sender) {
            revert CallerNotOwner();
        }

        // Burn position NFT
        _burn(id);
        
        // Ratio of total synthetics issued by the alchemist / underlingying value of collateral stored in the alchemist
        // If the system experiences bad debt we use this ratio to scale back the value of yield tokens that are transmuted
        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

        uint256 scaledTransmuted = amountTransmuted;
        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }

// @audit after redeem when the totalYield that is transmuted amount of user less then after redeeming then only available amount myt
// will transmute to user the remaining portion is not transfer to user it remain in contract but for user the hole scaledTransmuted amount is burn. 
@>        uint256 debtValue = alchemist.convertYieldTokensToDebt(yieldTokenBalance);
@>        uint256 amountToRedeem = scaledTransmuted > debtValue ? scaledTransmuted - debtValue : 0;
 
@>        if (amountToRedeem > 0) alchemist.redeem(amountToRedeem);

@>        uint256 totalYield = alchemist.convertDebtTokensToYield(scaledTransmuted);

        // Cap to what we actually hold now (handles redeem() rounding shortfalls).
@>        uint256 balAfterRedeem = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
@>        uint256 distributable = totalYield <= balAfterRedeem ? totalYield : balAfterRedeem;
        // Split distributable amount. Round fee down; claimant gets the remainder.
        uint256 feeYield = distributable * transmutationFee / BPS;
        uint256 claimYield = distributable - feeYield;

        uint256 syntheticFee = amountNottransmuted * exitFee / BPS;
        uint256 syntheticReturned = amountNottransmuted - syntheticFee;
        // Remove untransmuted amount from the staking graph
        if (blocksLeft > 0) _updateStakingGraph(-position.amount.toInt256() * BLOCK_SCALING_FACTOR / transmutationTime.toInt256(), blocksLeft);

@>        TokenUtils.safeTransfer(alchemist.myt(), msg.sender, claimYield);
        TokenUtils.safeTransfer(alchemist.myt(), protocolFeeReceiver, feeYield);

        TokenUtils.safeTransfer(syntheticToken, msg.sender, syntheticReturned);
        TokenUtils.safeTransfer(syntheticToken, protocolFeeReceiver, syntheticFee);

        // Burn remaining synths that were not returned
@>        TokenUtils.safeBurn(syntheticToken, amountTransmuted);
        alchemist.reduceSyntheticsIssued(amountTransmuted);
        alchemist.setTransmuterTokenBalance(TokenUtils.safeBalanceOf(alchemist.myt(), address(this)));

        totalLocked -= position.amount;

        emit PositionClaimed(msg.sender, claimYield, syntheticReturned);

        delete _positions[id];
    }

```

This mean when the (myt) token balance of transmuter is less than the `scaledTransmuter` then the redeem happen to increase the myt token balance of tranmuter. after redeem the the transmuter balance is check against the `scaledTransmuter` amount if it's still less than `scaledTransmuter` amount then only avialable myt token will be send to user. The main problem is that the extra amount from available balance which is user synthetic token, is not transfer back to user with `amountNottransmuted` amount. The function burn hole `amountTransumted` which also include that shortfall amount. user will loss that amount of synthetic token forever due the wrong and miscalculation logic of function, below in code i provide comment for better understanding.

```javascript
// balance of transmuter (myt) yield token
@>        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

        uint256 scaledTransmuted = amountTransmuted;
        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }

// debtValut is Synthetic amount of myt balance of tranmuter
@>        uint256 debtValue = alchemist.convertYieldTokensToDebt(yieldTokenBalance);
// if scaledTransmuted balance of user get bigger than available balance of tranmuter redeem should happen.
@>        uint256 amountToRedeem = scaledTransmuted > debtValue ? scaledTransmuted - debtValue : 0;
 
@>        if (amountToRedeem > 0) alchemist.redeem(amountToRedeem);

@>        uint256 totalYield = alchemist.convertDebtTokensToYield(scaledTransmuted);

// after redeeming the balance of transmuter(myt) check against user transmuted amount if it's still less than only available myt in transmuter will transfer to user.
@>        uint256 balAfterRedeem = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
@>        uint256 distributable = totalYield <= balAfterRedeem ? totalYield : balAfterRedeem;

```

The function at end transfer only available myt compare to user scaledTransmuter amount transfer to user but burn complete amount for user.

```javascript
        uint256 feeYield = distributable * transmutationFee / BPS;
@>        uint256 claimYield = distributable - feeYield;

@>       uint256 syntheticFee = amountNottransmuted * exitFee / BPS;
        uint256 syntheticReturned = amountNottransmuted - syntheticFee;
 
        // @audit untransmuted amount is the hole position amount or amountNottransmuted???
        // Remove untransmuted amount from the staking graph
        if (blocksLeft > 0) _updateStakingGraph(-position.amount.toInt256() * BLOCK_SCALING_FACTOR / transmutationTime.toInt256(), blocksLeft);

@>        TokenUtils.safeTransfer(alchemist.myt(), msg.sender, claimYield);
        TokenUtils.safeTransfer(alchemist.myt(), protocolFeeReceiver, feeYield);

        TokenUtils.safeTransfer(syntheticToken, msg.sender, syntheticReturned);
        TokenUtils.safeTransfer(syntheticToken, protocolFeeReceiver, syntheticFee);

        // Burn remaining synths that were not returned
@>        TokenUtils.safeBurn(syntheticToken, amountTransmuted);
        alchemist.reduceSyntheticsIssued(amountTransmuted);
        alchemist.setTransmuterTokenBalance(TokenUtils.safeBalanceOf(alchemist.myt(), address(this)));

        totalLocked -= position.amount;

```

That remaining amount of `scaledTransmuted` amount(after transmuter redeeming if still the myt balance of transmuter is less than scaledTransmuter(totalYield in myt format)) that shortfall amount is not transfer as a syntheticToken to user back, this will lead to user loss that amount forever. this is completely wrong and lead to significant loss for user when using the transmuter contract for transmution.

## Impact:

Users suffer partial redeemption loss and lead to loss of thier synthetic token forever. this will disincentivize user to use this contract.

## Mitigation step

Add the following code into claimRedemption function to correct the logic and behavior of function.

```javascript

    function claimRedemption(uint256 id) external {
        StakingPosition storage position = _positions[id];

        if (position.maturationBlock == 0) {
            revert PositionNotFound();
        }

        if (position.startBlock == block.number) {
            revert PrematureClaim();
        }

        uint256 transmutationTime = position.maturationBlock - position.startBlock;
        uint256 blocksLeft = position.maturationBlock > block.number ? position.maturationBlock - block.number : 0;
        uint256 rounded = position.amount * blocksLeft / transmutationTime + (position.amount * blocksLeft % transmutationTime == 0 ? 0 : 1);
        uint256 amountNottransmuted = blocksLeft > 0 ? rounded : 0;
        uint256 amountTransmuted = position.amount - amountNottransmuted;

        if (_requireOwned(id) != msg.sender) {
            revert CallerNotOwner();
        }

        // Burn position NFT
        _burn(id);
        
        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;

        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

        uint256 scaledTransmuted = amountTransmuted;

        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }

        // If the contract has a balance of yield tokens from alchemist repayments then we only need to redeem partial or none from Alchemist earmarked
        uint256 debtValue = alchemist.convertYieldTokensToDebt(yieldTokenBalance);
        uint256 amountToRedeem = scaledTransmuted > debtValue ? scaledTransmuted - debtValue : 0;
 
        if (amountToRedeem > 0) alchemist.redeem(amountToRedeem);

        uint256 totalYield = alchemist.convertDebtTokensToYield(scaledTransmuted);

        // Cap to what we actually hold now (handles redeem() rounding shortfalls).
        uint256 balAfterRedeem = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        uint256 distributable = totalYield <= balAfterRedeem ? totalYield : balAfterRedeem;
++      if(distributable < totalYield){
++          uint256 returnSyntAdditionalAmount = totalYield + distributable;
++          synthiticReturned += alchemist.convertYieldTokensToDebt(returnSyntAdditionalAmount);
++      }


        uint256 feeYield = distributable * transmutationFee / BPS;
        uint256 claimYield = distributable - feeYield;

        uint256 syntheticFee = amountNottransmuted * exitFee / BPS;
        uint256 syntheticReturned = amountNottransmuted - syntheticFee;
 
        if (blocksLeft > 0) _updateStakingGraph(-position.amount.toInt256() * BLOCK_SCALING_FACTOR / transmutationTime.toInt256(), blocksLeft);

        TokenUtils.safeTransfer(alchemist.myt(), msg.sender, claimYield);
        TokenUtils.safeTransfer(alchemist.myt(), protocolFeeReceiver, feeYield);

        TokenUtils.safeTransfer(syntheticToken, msg.sender, syntheticReturned);
        TokenUtils.safeTransfer(syntheticToken, protocolFeeReceiver, syntheticFee);

        // Burn remaining synths that were not returned
        TokenUtils.safeBurn(syntheticToken, amountTransmuted);
        alchemist.reduceSyntheticsIssued(amountTransmuted);
        alchemist.setTransmuterTokenBalance(TokenUtils.safeBalanceOf(alchemist.myt(), address(this)));

        totalLocked -= position.amount;

        emit PositionClaimed(msg.sender, claimYield, syntheticReturned);
        delete _positions[id];
    }
```

## Proof of Concept

## Proof of Concept

```javascript
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {StdCheats} from "forge-std/StdCheats.sol";

import {AlchemistV3} from "../AlchemistV3.sol";
import {AlEth} from "../external/AlEth.sol";
import {Transmuter} from "../Transmuter.sol";
import {StakingGraph} from "../libraries/StakingGraph.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "../interfaces/ITransmuter.sol";

import "../base/TransmuterErrors.sol";

contract MockAlchemist {
    uint256 public constant FIXED_POINT_SCALAR = 1e18;
    uint256 public underlyingValue;
    uint256 public syntheticsIssued;
    address public myt;

    constructor(address _myt) {
        myt = _myt;
    }

    function setUnderlyingValue(uint256 amount) public {
        underlyingValue = amount;
    }

    function setSyntheticsIssued(uint256 amount) public {
        syntheticsIssued = amount;
    }

    function convertYieldTokensToUnderlying(uint256 amount) external pure returns (uint256) {
        return (amount * 2 * FIXED_POINT_SCALAR) / FIXED_POINT_SCALAR;
    }

    function convertUnderlyingTokensToYield(uint256 amount) public pure returns (uint256) {
        return amount * FIXED_POINT_SCALAR / (2 * FIXED_POINT_SCALAR);
    }

    function convertYieldTokensToDebt(uint256 amount) public pure returns (uint256) {
        return (amount * 2 * FIXED_POINT_SCALAR) / FIXED_POINT_SCALAR;
    }

    function convertDebtTokensToYield(uint256 amount) public pure returns (uint256) {
        return amount * FIXED_POINT_SCALAR / (2 * FIXED_POINT_SCALAR);
    }
// @audit redeem in tranmuter return only available myt yield token to transmuter.
// this mock version of the alchemist has problem which return max(uint256), in real case this wrong during redeem
// only available myt will send to tranmuter so we update it the below mock to run the real case test.
    function redeem(uint256 underlying) external {
        IERC20(myt).transfer(msg.sender, IERC20(myt).balanceOf(address(this)));
    }

    function totalDebt() external pure returns (uint256) {
        return type(uint256).max;
    }

    function totalSyntheticsIssued() external returns (uint256) {
        if (syntheticsIssued > 0) {
            return syntheticsIssued;
        } else {
            return type(uint256).max / 1e20;
        }
    }

    function reduceSyntheticsIssued(uint256 amount) external {}

    function setTransmuterTokenBalance(uint256 amount) external {}

    function yieldToken() external view returns (address) {
        return address(myt);
    }

    function underlyingToken() external view returns (address) {
        return address(myt);
    }

    function getTotalUnderlyingValue() external view returns (uint256) {
        if (underlyingValue > 0) {
            return underlyingValue;
        } else {
            return type(uint256).max / 1e20;
        }
    }
}

contract MockMorphoV2Vault is ERC20 {
    // Simplied vault for testing
    // Shares are still treated as 18 decimal erc20 tokens
    // regardless of the underlying token decimals
    constructor() ERC20("Mock Myt Vault", "MMV") {}
}

contract TransmuterTest is Test {
    using StakingGraph for StakingGraph.Graph;

    AlEth public alETH;
    ERC20 public collateralToken; // morpho vault 2 shares
    AlEth public underlyingToken;
    Transmuter public transmuter;

    MockAlchemist public alchemist;

    StakingGraph.Graph private graph;
    MockMorphoV2Vault public vault;
    address public admin;
    address public curator;

    event TransmuterLog(string message, uint256 value);

    function setUp() public {
        alETH = new AlEth();
        underlyingToken = new AlEth();
        vault = new MockMorphoV2Vault();
        collateralToken = ERC20(address(vault));
        alchemist = new MockAlchemist(address(collateralToken));
        transmuter = new Transmuter(ITransmuter.TransmuterInitializationParams(address(alETH), address(this), 5_256_000, 0, 0, 52_560_000 / 2));

        transmuter.setAlchemist(address(alchemist));

        transmuter.setDepositCap(uint256(type(int256).max));

        deal(alchemist.myt(), address(alchemist), 100 ether);
        deal(address(alETH), address(0xbeef), type(uint256).max);

        vm.prank(address(alchemist));
        IERC20(alchemist.myt()).approve(address(transmuter), type(uint256).max);

        vm.prank(address(0xbeef));
        alETH.approve(address(transmuter), type(uint256).max);
    }

    function testAA() public {
        deal(address(collateralToken), address(transmuter), 100 ether);
        deal(address(alETH), address(0xbeef), 1000 ether);

        console.log("Before claim request user synthetic token amount ", alETH.balanceOf(address(0xbeef)));
        console.log("Before claim and transmutation myt balance of user", collateralToken.balanceOf(address(0xbeef)));
        
        vm.prank(address(0xbeef));
        transmuter.createRedemption(1000e18);

        vm.roll(block.number + 5_256_000);

        vm.prank(address(0xbeef));
        transmuter.claimRedemption(1);

        console.log("ration between myt:synthetic token is 2:1, user must have 50e18 myt after redeemption");
        console.log("After claim redeemption actual user myt yield token balance ",collateralToken.balanceOf(address(0xbeef)));
        console.log("After redeemption claim user synthetic token amount", alETH.balanceOf(address(0xbeef)));

        console.log("We see only 400e18 of synthetic token is transmute to myt which is 200e18, user loss 600e18 of synthetic token forever!!!");
    }


```
