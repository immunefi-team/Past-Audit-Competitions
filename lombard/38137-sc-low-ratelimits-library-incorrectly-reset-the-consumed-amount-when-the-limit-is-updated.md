# #38137 \[SC-Low] \`RateLimits\` library incorrectly reset the consumed amount when the limit is updated

**Submitted on Dec 25th 2024 at 19:00:34 UTC by @OxAlix2 for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38137
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/libs/RateLimits.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`RateLimits` allows the protocol to limit the amount of withdrawn LBTC for a certain period, something like only 10 LBTC could be withdrawn in 10 minutes. This is done by keeping track of the amount in flight, i.e. the amount used in this period/window. However, it incorrectly handles an edge case just after the limit is increased.

## Vulnerability Details

To fetch the amount that could be used/withdrawn at a specific time `availableAmountToSend` is used, which calculates the decay and compares it to the amount in flight (already used in this window):

```solidity
uint256 decay = (_limit * timeSinceLastDeposit) / _window;
currentAmountInFlight = _amountInFlight <= decay
    ? 0
    : _amountInFlight - decay;
amountCanBeSent = _limit <= currentAmountInFlight
    ? 0
    : _limit - currentAmountInFlight;
```

All is good, until `_amountInFlight` is less than `decay`, this could happen if the limit is increased in some window, where `_amountInFlight` is > 0. In that case, the amount in flight will be reset, and the window usage will reset.

Let's take the following example, at the end of window X we have `_amountInFlight` as 9e8, and the window limit is 10e8. If the limit is increased to 20e8, the `_amountInFlight` will be reset to 0, and users will be able to use/deposit 20e8 this window, knowing that they've already used/deposited 9e8; leaving the window with 29e8 limit.

This breaks the rate limit functionality.

## Impact Details

Rate limit functionality will be bypassed/broken and users will be able to deposit a wrong exaggerated value of LBTC.

## References

<https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/libs/RateLimits.sol#L91>

## Mitigation

```diff
    function availableAmountToSend(
        uint256 _amountInFlight,
        uint256 _lastUpdated,
        uint256 _limit,
        uint256 _window
    )
        internal
        view
        returns (uint256 currentAmountInFlight, uint256 amountCanBeSent)
    {
        uint256 timeSinceLastDeposit = block.timestamp - _lastUpdated;
        if (timeSinceLastDeposit >= _window) {
            currentAmountInFlight = 0;
            amountCanBeSent = _limit;
        } else {
            uint256 decay = (_limit * timeSinceLastDeposit) / _window;
            currentAmountInFlight = _amountInFlight <= decay
-               ? 0
+               ? _amountInFlight
                : _amountInFlight - decay;
            amountCanBeSent = _limit <= currentAmountInFlight
                ? 0
                : _limit - currentAmountInFlight;
        }
    }
```

## Proof of Concept

## Proof of Concept

```solidity
contract RateLimitPoC is Test {
    RateLimits.Data rateLimit;

    function setUp() public {
        RateLimits.setRateLimit(
            rateLimit,
            RateLimits.Config({
                chainId: bytes32(uint256(1)),
                limit: 100,
                window: 100
            })
        );
    }

    function test_limitReset() public {
        vm.warp(block.timestamp + 10);

        // Use 100 units of the rate limit
        RateLimits.updateLimit(rateLimit, 100);

        (uint256 currentAmountInFlight, uint256 amountCanBeSent) = RateLimits
            .availableAmountToSend(rateLimit);

        // 100 units used, 0 units left for this window
        assertEq(currentAmountInFlight, 100);
        assertEq(amountCanBeSent, 0);

        // Update the rate limit to 2000 units
        RateLimits.setRateLimit(
            rateLimit,
            RateLimits.Config({
                chainId: bytes32(uint256(1)),
                limit: 2000,
                window: 100
            })
        );

        // 10 seconds have passed
        vm.warp(block.timestamp + 10);

        (currentAmountInFlight, amountCanBeSent) = RateLimits
            .availableAmountToSend(rateLimit);

        // 0 units used, 2000 units available for this window (which is wrong, it should be 2000 - 100 = 1900)
        assertEq(currentAmountInFlight, 0);
        assertEq(amountCanBeSent, 2000);
    }
}
```
