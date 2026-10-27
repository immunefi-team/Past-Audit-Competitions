# 25906 - \[SC - Insight] setDelay function doesnt revert even when the d...

Submitted on Nov 20th 2023 at 23:43:57 UTC by @ThreeHrSleep for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25906

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xf2991507952d9594e71a44a54fb19f3109d213a5#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

While using the `setDelay()` function,it's supposed to revert when the delay doesn't exceed the MINIMUM\_DELAY. But due to wrong validation logic, the delay can be set to exactly MINIMUM\_DELAY. In `setDelay()` function,it's making sure that `delay_ >= MINIMUM_DELAY`(delay is greater or equals minimum delay)

```
        require(delay_ >= MINIMUM_DELAY, "Timelock::setDelay: Delay must exceed minimum delay.");
```

where as it should have been `delay_ > MINIMUM_DELAY` to make sure that Delay is exceeding minimum delay.

## Impact

Intended behaviour of the smart contract and actual functionality doesn't match

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: N/A CVSS2 Score: N/A

## Recommendation

change the require statement at line 226 <https://etherscan.io/address/0xf2991507952d9594e71a44a54fb19f3109d213a5#code#F1#L226> from

```
        require(delay_ >= MINIMUM_DELAY, "Timelock::setDelay: Delay must exceed minimum delay.");
```

to

```
        require(delay_ > MINIMUM_DELAY, "Timelock::setDelay: Delay must exceed minimum delay.");
```

## References

<https://etherscan.io/address/0xf2991507952d9594e71a44a54fb19f3109d213a5#code#F1#L226>

```
    function setDelay(uint delay_) public {
        require(msg.sender == address(this), "Timelock::setDelay: Call must come from Timelock.");
        require(delay_ >= MINIMUM_DELAY, "Timelock::setDelay: Delay must exceed minimum delay.");
        require(delay_ <= MAXIMUM_DELAY, "Timelock::setDelay: Delay must not exceed maximum delay.");
        delay = delay_;

        emit NewDelay(delay);
    }
```

## Proof of concept

To demonstrate the issue,here the `setDelay()` function is called with the exact Minimum Delay(45 days,which is 3888000 seconds),and it is not reverting even though the delay isn't exceeding minimum delay

<https://dashboard.tenderly.co/shared/fork/ce6516a7-e525-4be2-abb9-130d0cb1446d/simulation/e72b88be-52ad-4e3b-a651-b9b31542bf41?trace=0>
