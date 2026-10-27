# 25927 - \[SC - Insight] MultiSig Owners can set malicious implementatio...

Submitted on Nov 21st 2023 at 05:25:48 UTC by @Shogoki for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25927

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0xf2991507952d9594e71a44a54fb19f3109d213a5#code>

Impacts:

* Introducing Malicious Contract

## Description

## Bug Description

The DeGate Team made it clear, that they want to prevent any possible malicious actions from the MultiSig Owners. Only exception should be a malicious action that can be detected early (45 days before) via the TimeLock feature. However, there is a possible "attack" that the Owners could run, by:

* deploying a (seemingly) harmless new version of the implementation contract
* queue a Transaction in Timelock to change the implementation on the proxy
* waiting for the Timelock to finsish
* replacing the implementation contract with a new malicious one at the same address
* call executeTransaction, to configure the malicious implementation contract

In this scenario the malicious contract cannot be seen by the users before it is actually deployed.

To deploy the malicious contract on the same address, the attackers could use a similiar technique like the Tornado Cash Governance Hack in May 2023, where the attackers deployed an intermediate contract with a predictable address using create2, which in turns creates the actual implementation contract. Leveraging `selfdestruct` and this technique it is possible to deploy another contract with different bytecode on the same address.

## Impact

* Possible Loss of Funds
* Malicious implementation contract get´s used by proxy

## Risk Breakdown

Difficulty to Exploit: Medium

## Recommendation

One way to avoid this would be to also pass the hash of the bytecode for the implementation contract that when changing it. So nobody could actually deploy a different contract there.

## References

<https://github.com/degatedev/protocols/blob/c8961f2cd354a6578bb332337f983ab4c39c1806/packages/loopring\\_v3/contracts/thirdparty/proxies/OwnedUpgradabilityProxy.sol#L74-L76>

## Proof of concept

I attached a PoC in Foundry as a Gist. Note: Because selfdestruct is only executed at the end of the call and a limitation in Foundry i added the first part of the test in Setup.
