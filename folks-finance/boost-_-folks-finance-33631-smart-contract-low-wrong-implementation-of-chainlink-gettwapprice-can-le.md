# Boost \_ Folks Finance 33631 - \[Smart Contract - Low] Wrong implementation of chainLink getTwapPrice

Submitted on Wed Jul 24 2024 23:49:26 GMT-0400 (Atlantic Standard Time) by @gizzy for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33631

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

The TWAP price calculation wrongly assumes that roundId is incremental which from chainlink docs is not ( <https://docs.chain.link/data-feeds/historical-data#getrounddata-return-values> ) .This assumption leads to the twap Price being wrong when a new aggregator is updated in chainlink which will in turn update the PhaseID

## Vulnerability Details

The getTwapPrice function the chainlink node is used to get the Time-Weighted Average Price (TWAP) given a particular twapTimeInterval .

```solidity
 function getTwapPrice(
        AggregatorV3Interface chainlink,
        uint80 latestRoundId,
        uint256 latestPrice,
        uint256 twapTimeInterval
    ) internal view returns (uint256 price) {
        uint256 priceSum = latestPrice;
        uint256 priceCount = 1;

        uint256 startTime = block.timestamp - twapTimeInterval;

        /// @dev Iterate over the previous rounds until reaching a round that was updated before the start time
        while (latestRoundId > 0) {
            try chainlink.getRoundData(--latestRoundId) returns (
                uint80,
                int256 answer,
                uint256,
                uint256 updatedAt,
                uint80
            ) {
                if (updatedAt < startTime) {
                    break;
                }
                priceSum += answer.toUint256();
                priceCount++;
            } catch {
                break;
            }
        }

        return priceSum / priceCount;
    }
```

The problem is that getRoundData(--latestRoundId) decreases the latestRoundId , when the phaseId is which currently 6 is incremented to 7 . Then the first roundID of phaseid 7 will be the latestRoundID = (7 <<64 | 1).(from chainlink doc : <https://docs.chain.link/data-feeds/historical-data#getrounddata-return-values> ) This latestRoundID will greatly differ from phase 6 latestRoundID . Calling the getTwapPrice with the phaseID 7 latestRoundID , in the loop when --latestRoundID occurs chain link will return instead of reverting will return 0 as price and 0 as updated at . updatedAt which is zero will be less than startTime . this will break the loop and return the latestPrice as the TWAP price .

The poc shows transition from phaseId of 5 to 6 of ETH/USD of mainnet. and its effects

## Impact Details

The flaw in the price if big enough can be exploited for functionalities that needs oracle to function

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/nodes/ChainlinkNode.sol#L47C1-L78C6>

## Proof of concept

## Proof of Concept

This POC shows an example with transition from phaseID 5 to 6 of ETH/USD Oracle of mainnet

```solidity
import "../libForge/forge-std/src/Test.sol";
import "../contracts/bridge/libraries/Messages.sol";
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
import "@openzeppelin/contracts/utils/math/SafeCast.sol";



contract testTwap is Test{

    using SafeCast for int256;

    function setUp() public virtual {
        vm.createSelectFork(vm.envString("RPC_URL"));
    }

    /// @notice Calculates the Time-Weighted Average Price (TWAP) for a Chainlink feed from all the prices between the latest round and the latest round before the time interval.
    /// @param chainlink The Chainlink aggregator contract.
    /// @param latestRoundId The latest round ID.
    /// @param latestPrice The latest price.
    /// @param twapTimeInterval The TWAP time interval.
    /// @return price The TWAP price.
    function getTwapPrice(
        AggregatorV3Interface chainlink,
        uint80 latestRoundId,
        uint256 latestPrice,
        uint256 twapTimeInterval,
        uint256 blockTimeStampOFtestTwap
    ) internal view returns (uint256 price) {
        uint256 priceSum = latestPrice;
        uint256 priceCount = 1;

        uint256 startTime = blockTimeStampOFtestTwap - twapTimeInterval;

        /// @dev Iterate over the previous rounds until reaching a round that was updated before the start time
        while (latestRoundId > 0) {
            try chainlink.getRoundData(--latestRoundId) returns (
                uint80,
                int256 answer,
                uint256,
                uint256 updatedAt,
                uint80
            ) {
                if (updatedAt < startTime) {
                    break;
                }
                priceSum += answer.toUint256();
                priceCount++;
            } catch {
                break;
            }
        }

        return priceSum / priceCount;
    }

    ///  the TWAP price calculation wrongly assumes that roundId is  incremental which will lead to big wrong price diversion when the phaseID increments meaning a new aggregator
    /// was added


    function testTwapPrice() public {
        //ETH/USD Chainlink Aggregator Mainnet
        AggregatorV3Interface chainlinkAggregator = AggregatorV3Interface(0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419);
        (uint80 roundId,int256 answer, , uint256 updatedAt, ) = chainlinkAggregator.latestRoundData();
        //chainlink has a PhaseID that is use to calculate the roundId  and from the docs 
        //phaseId = roundId >> 64;
       (uint80 roundId0,int256 answer0, , uint256 updatedAt0, )= chainlinkAggregator.getRoundData(uint80((5 <<64 | 45034)));
       console.log("last roundId of phaseid 5:",roundId0);
       // note roundId0 ++ will also return 0,0,0 as data

        uint256 phaseID = roundId >> 64;
         console.log("phaseID: ", phaseID);
        uint256 aggregatorRoundId = (roundId & 0xFFFFFFFFFFFFFFFF);
        console.log("aggregatorRoundId: ", aggregatorRoundId);
        uint256 firstRoundID_phase6 = (roundId - aggregatorRoundId) + 1;

       
        
        console.log("firstRoundID_phase6:", firstRoundID_phase6);

        //suppose the twaptimeInterval is 2hours = 7200;
        //and the chainlinkAggregator just switch from phaseID 5 to 6 .
        //with  latest roundId = firstRoundID_phase6 

        (uint80 roundId1,int256 answer1, , uint256 updatedAt1, ) = chainlinkAggregator.getRoundData(uint80(firstRoundID_phase6 ));
        console.log("answer1:",answer1.toUint256());

       uint256 priceTwap = getTwapPrice(chainlinkAggregator,roundId1,answer1.toUint256(),7200,updatedAt1 + 20 minutes);

       console.log("priceTwap:",priceTwap);

       

       
    }
    

  

}
```
