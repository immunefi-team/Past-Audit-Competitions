# 28967 - \[SC - Insight] When fallback oracle is frozen fetchPrice can r...

Submitted on Mar 3rd 2024 at 16:40:29 UTC by @savi0ur for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28967

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/PriceFeed.sol>

Impacts:

* Protocol insolvency

## Description

## Bug Description

When fallback oracle is operational and chainlink oracle is frozen, status is `Status.usingFallbackChainlinkFrozen`. Now upon execution of `fetchPrice()` again, it will execute below block of if statement. <https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/PriceFeed.sol#L299-L363>

```solidity
// --- CASE 4: Using Fallback, and Chainlink is frozen ---
if (status == Status.usingFallbackChainlinkFrozen) {
    if (_chainlinkIsBroken(chainlinkResponse, prevChainlinkResponse)) {
        // ...
    }

    if (_chainlinkIsFrozen(chainlinkResponse)) {
        // ...
    }

    if (_chainlinkPriceChangeAboveMax(chainlinkResponse, prevChainlinkResponse)) {
        // ...
    }

    // if Chainlink is live and Fallback is broken, remember Fallback broke, and return Chainlink price
    if (_fallbackIsBroken(fallbackResponse)) {
        _changeStatus(Status.usingChainlinkFallbackUntrusted);
        return _storeChainlinkPrice(chainlinkResponse.answer);
    }

    // If Chainlink is live and Fallback is frozen, just use last good price (no status change) since we have no basis for comparison
    if (_fallbackIsFrozen(fallbackResponse)) {
        return INVALID_PRICE;
    }
```

If chainlink oracle is working this time and fallback oracle is frozen, status remain unchanged i.e, `Status.usingFallbackChainlinkFrozen` even if chainlink oracle is working as expected - Not Broken, Not Frozen and No Price change above 5% from last reported chainlink price. And then return `INVALID_PRICE` and caller will consume the `lastGoodPrice`.

Its blindly assuming that chainlink oracle is still not operational and its frozen and continue to be in the same state `Status.usingFallbackChainlinkFrozen` even if chainlink is working perfectly fine.

Suppose chainlink oracle is still operational and fallback oracle is still frozen, upon calling `fetchPrice()` again will execute same block of statements as shown above and report the same `INVALID_PRICE` and caller will consume the same `lastGoodPrice` as status is still \`Status.usingFallbackChainlinkFrozen.

Now being status in `Status.usingFallbackChainlinkFrozen`, this will always return `INVALID_PRICE` and caller consuming `lastGoodPrice` as long as fallback is frozen, even if chainlink is ready to serve correct price.

## Impact

Since its always returning `lastGoodPrice` when fallback is frozen and status = `Status.usingFallbackChainlinkFrozen` even if chainlink is operational, there is a possibility of actual price of eBTC feed deviates more/less from `lastGoodPrice`.

In such case, user can monitor fallback oracle to bring up. Once its up and report the price which is almost similar to chainlink price but deviates more/less from `lastGoodPrice`, then since status is still `Status.usingFallbackChainlinkFrozen`, it will then check `_bothOraclesSimilarPrice()` which return true as prices reported are almost similar and then change status to `Status.chainlinkWorking` and report chainlink price.

<https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/PriceFeed.sol#L365-L370>

```solidity
// If Chainlink is live and Fallback is working, compare prices. Switch to Chainlink
// if prices are within 5%, and return Chainlink price.
if (_bothOraclesSimilarPrice(chainlinkResponse, fallbackResponse)) {
	_changeStatus(Status.chainlinkWorking);
	return _storeChainlinkPrice(chainlinkResponse.answer);
}
```

Now in this situation, user can take advantage of this situation as follows:

* `Chainlink eBTC Price` < `lastGoodPrice`: A user can redeem a CDP with the `lastGoodPrice` and then open a new CDP with the current Chainlink price, obtaining a collateral surplus.
* `Chainlink eBTC Price` > `lastGoodPrice`: A user can open a CDP with the `lastGoodPrice` and then redeem it with the current Chainlink price, obtaining a collateral surplus.

In both cases, collateral surplus is obtained at the expense of the protocol with no risk for the user. Doing this multiple times, by multiple users, will results in collateral being depleted slowly and eventually leads to protocol insolvency where debt (eBTC) > collateral (stETH).

## Recommendation

Make sure to handle the case when `status == Status.usingFallbackChainlinkFrozen` and fallback is frozen, use chainlink price when chainlink is working as expected and fallback is broken/frozen. For example,

```diff
// --- CASE 4: Using Fallback, and Chainlink is frozen ---
if (status == Status.usingFallbackChainlinkFrozen) {
    if (_chainlinkIsBroken(chainlinkResponse, prevChainlinkResponse)) {
        // ...
    }

    if (_chainlinkIsFrozen(chainlinkResponse)) {
        // ...
    }

    if (_chainlinkPriceChangeAboveMax(chainlinkResponse, prevChainlinkResponse)) {
        // ...
    }

    // if Chainlink is live and Fallback is broken, remember Fallback broke, and return Chainlink price
    if (_fallbackIsBroken(fallbackResponse)) {
        _changeStatus(Status.usingChainlinkFallbackUntrusted);
        return _storeChainlinkPrice(chainlinkResponse.answer);
    }

    // If Chainlink is live and Fallback is frozen, just use last good price (no status change) since we have no basis for comparison
    if (_fallbackIsFrozen(fallbackResponse)) {
+		_changeStatus(Status.usingChainlinkFallbackUntrusted);
+       return _storeChainlinkPrice(chainlinkResponse.answer);
-       return INVALID_PRICE;
    }
```

## References

<https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/PriceFeed.sol>

## Proof Of Concept

**Steps to Run using Foundry:**

* Paste following foundry code in `/ebtc-boost/packages/contracts/foundry_test/PriceFeed.stateTransitions.t.sol`
* Run using `forge test --match-contract PriceFeedStateTransitionTest --match-test testStatusUnchangedWhenChainlinkWorking -vvvv`

```solidity
function testStatusUnchangedWhenChainlinkWorking() public {
    // froze CL
    _frozeChainlink(_mockChainLinkEthBTC);

    // update state machine
    priceFeedTester.fetchPrice();
    IPriceFeed.Status status = priceFeedTester.status();
    assertEq(uint256(status), 3); // usingFallbackChainlinkFrozen

    // restore CL and froze FB
    _restoreChainlinkPriceAndTimestamp(_mockChainLinkEthBTC, initEthBTCPrice);
    _frozeFallback();

    // update state machine again
    priceFeedTester.fetchPrice();
    status = priceFeedTester.status();
    assertEq(uint256(status), 3); // usingFallbackChainlinkFrozen
}
```
