# Mitigation Audit \_ Folks Finance 34942 - \[Smart Contract - Insight] In function function getTwapPric

Submitted on Sun Sep 01 2024 16:10:07 GMT-0400 (Atlantic Standard Time) by @Paludo0x for [Mitigation Audit | Folks Finance](https://immunefi.com/bounty/mitigation-audit-folksfinance/)

Report ID: #34942

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/Folks-Finance/folks-finance-xchain-contracts/pull/9>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Vulnerability Details

In `ChainlinkNode::getTwapPrice()` there's a `while (latestRoundId > 0)` iteration. Inside the try / catch block there's the following check:

```
                if (latestRoundId == 0) {
                    break;
                }
```

This check is useless and unnecessarily consumes gas because the termination condition of the while cycle is `latestRoundId > 0`, therefore the `break` from the while cycle will be done anyway when `latestRoundId = 0`.

This is the relevant code

```

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
                if (latestRoundId == 0) {
                    break;
                }
            }
        }

        return priceSum / priceCount;
    }

```

## Impact Details

The impact is unecessary consuming of gas

## Proof of concept

## Proof of Concept

This test shall be run in remix. First you need to deploy the contract `AlwaysRevert` and then deploy `Test` using address of `AlwaysRevert`.

```
pragma solidity >=0.7.0 <0.9.0;


contract AlwaysRevert {
        function always_revert(uint256) public {
        revert();
    } 

}

contract Test {

    AlwaysRevert public alwaysRevert;
    uint256 public latestRoundId;
    bool public enteredHere;

    constructor (address alwaysRevert_) {
        alwaysRevert = AlwaysRevert(alwaysRevert_);
    }

    function getTwapPrice() public returns ( uint256) {
        latestRoundId = 10;
        enteredHere = false;

        while (latestRoundId > 0) {
            try alwaysRevert.always_revert(--latestRoundId)
             { } catch {
                if (latestRoundId == 0) {
                    enteredHere = true;
                    break;
                }

            }
        }

        return latestRoundId;
    }

    function getTwapPrice_without_break() public returns ( uint256) {
        latestRoundId = 10;

        while (latestRoundId > 0) {
            try alwaysRevert.always_revert(--latestRoundId)
             { } catch {}
        }
        return latestRoundId;
    }
}
```

The function `getTwapPrice` is a simplified version of original version, while `getTwapPrice_without_break` is without if statement.

Both reach the status where `latestRoundId = 0` without issues.
