# 31284 - \[SC - Insight] cancel should allow to cancel the proposal of t...

Submitted on May 16th 2024 at 11:26:54 UTC by @OxG0P1 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31284

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

cancel() does not allow to cancel proposals which are Expired.

## Vulnerability Details

The state of being "Expired" depends on the GRACE\_PERIOD of the timelock, and the GRACE\_PERIOD may be altered due to upgrades. Once the GRACE\_PERIOD of the timelock is changed, the state of the proposal may also be altered, so "Expired" is not necessarily the final state.

## Impact Details

Funds in the Timelock will be lost.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/governance/L2Governor.sol#L445-L448> <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/governance/L2Governor.sol#L625-L627>

## Proof of Concept

Consider the following scenario:

Alice submits Proposal A to stake 20,000 ETH to a DeFi protocol, and it successfully passes. However, it cannot be executed because there are now only 15,000 ETH in the timelock (due to other proposals consuming the funds), and then Proposal A expires.

Subsequently, the DeFi protocol gets hacked or rug-pulled.

Meanwhile, Proposal B is about to be executed to upgrade the timelock and extend the GRACE\_PERIOD (for example, by 7 days). Alice wants to cancel Proposal A, but she cannot because it is in the "Expired" state.

Proposal B is then executed, causing Proposal A to change from "Expired" to "Queued." A malicious user sends 5,000 ETH to the timelock and immediately executes Proposal A, sending 20,000 ETH to the hacked protocol.
