# 26501 - \[SC - Insight] Timelock should handle queuing transactions and...

Submitted on Dec 4th 2023 at 04:58:15 UTC by @conqueror for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26501

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xf2991507952d9594e71a44a54fb19f3109d213a5#code>

Impacts:

* Timelock should handle queuing deploys and initializing implementation contracts

## Description

## Bug Description

Even if an unauthorized party was able to gain access to the Multisig, they should not be able to queue up transactions besides upgrade transactions (for example, ones changing the proxy ownership).

Furthermore, your implementation contracts are currently uninitialized. While this is not a concern at the moment, if you ever upgrade to an implementation contract that uses delegatecall in any way, this could become a big problem.

The timelock contract therefore should be in charge of:

1. Queueing up implementation contract deploys (you can set delay to 0 here), and calling initialize on the implementation contract
2. Only allow upgrade to be called on the proxy contracts, and not other functions. Do this by checking the signature.

## Impact

Mitigate damage if multisig is ever compromised and initialize implementations. Not initializing implementations can be extremely bad in the future if you ever upgrade them to use delegatecall

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: Low CVSS2 Score: L

## Recommendation

Refactor the timelock contract a bit, as described above

## References
