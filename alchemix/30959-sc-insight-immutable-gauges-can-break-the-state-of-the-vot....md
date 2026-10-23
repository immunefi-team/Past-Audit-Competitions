# 30959 - \[SC - Insight] Immutable gauges can break the state of the vot...

Submitted on May 9th 2024 at 06:48:38 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30959

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Smart contract unable to operate due to lack of token funds

## Description

## Vulnerability Details

**Gauges** are smart contracts accounting for rewards and passing them through to the pools.

**Pools** are 3rd-party code subject to updates. If any function deprecates, smart contracts interacting with these pools must be able to update their code accordingly.

The **GaugeFactory** creates **Gauges**. There are 2 types of Gauges so far, and the source code of the implementations is hardcoded in the **GaugeFactory**. There is no way to update the implementation of a Gauge if required, nor to add new **Gauges** types with other implementations.

The **Voter** smart contract creates gauges by interacting with the **GaugeFactory**.

Unfortunately, the current implementation of the **Voter** smart contract receives the address of this immutable **GaugeFactory** on deployment, and there is no way to update it.

Even though the role "`emergencyCouncil`" in the **Voter** can deactivate a gauge by calling `Voter.killGauge(address _gauge)`, newly created gauges will still use the same implementation. Therefore the ability to kill gauges only solves half of the problem.

Alchemix must be able to deploy a new **GaugeFactory** and make the **Voter** smart contract use the latest factory, to fix problems/emergencies that can only be solved by updating a gauge's source code or adding a new type of gauge.

At the time of writing, **the only way to react to bugs in a gauge is to deploy a completely new Voter and GaugeFactory, breaking the entire state of the voting system.**

## Recommendation

Create a new function in the `Voter` smart contract allowing the *emergencyCouncil* to update the address of the *GaugeFactory*.

Here's our recommended implementation to future-proof the Voter smart contract:

```

/// @notice A not immutable gauge factory contract address ("immutable" keyword removed)
address public gaugefactory;

// Event emitted when the gauge factory address is updated
event GaugeFactoryUpdated(address indexed gauge);

function updateGaugeFactory(address _gaugefactory) external {
    require(msg.sender == emergencyCouncil, "not emergency council");
    gaugefactory = _gaugefactory;
    emit GaugeFactoryUpdated(_gaugefactory);
}
```

## About Severity

Due to the catastrophic impact this issue may have, we consider the report to be of `Critical` severity.

But because there is a prerequisite, we consider a fair severity to be `High`.

## Proof of Concept

This specific report related to a business logic flag does not require an "executable proof of concept" to verify its validity.

But following the terms of the Boost, here's a function that deploys the voter and explains with comments the consequences of using an immutable gauge when interacting with 3rd-party code (pools).

```
function testDeployVoter() public {
    // Deploying the Voter
    voter = new Voter(address(veALCX), address(gaugeFactory), address(bribeFactory), address(flux), address(alcx));

    // 1- The implementation of the gauges can't be updated

    // 2- The factory itself can't be updated

    // 3- The Voter can't point to a new gauge factory

    // The only solution is to deploy a new Voter contract pointing to a new factory,
    // which breaks the accounting in the voting system.

}

```
