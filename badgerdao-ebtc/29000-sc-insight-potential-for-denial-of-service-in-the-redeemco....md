# 29000 - \[SC - Insight] Potential for Denial-of-Service in the redeemCo...

Submitted on Mar 4th 2024 at 13:39:32 UTC by @cheatcode for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #29000

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/CdpManager.sol>

Impacts:

* Temporary freezing of funds for at least 15 minutes

## Description

## Brief/Intro

The CdpManager contract's redeemCollateral function is vulnerable to a potential DoS attack due to its loop mechanism for processing CDP redemptions. This vulnerability could be exploited to prevent legitimate users from accessing the redemption functionality.

## Vulnerability Details

The redeemCollateral function allows users to redeem collateral from CDPs by repaying part of the system's debt. It iterates over CDPs, starting from those with the lowest collateral ratio, until the requested debt amount is redeemed or the specified maximum number of iterations (\_maxIterations) is reached. The vulnerability arises due to the potential for this loop to consume an excessive amount of gas, particularly when the function is called with a large debt amount (\_debt) relative to the individual debts of CDPs. This situation can cause transactions to fail due to exceeding the block gas limit, effectively causing a DoS condition where legitimate users are unable to perform redemptions.

## Impact Details

An attacker, by specifying high values for both \_debt and \_maxIterations, could deliberately trigger the vulnerability, causing the function to consume an excessive amount of gas and potentially making the contract unusable for legitimate transactions. This would not only prevent users from redeeming collateral but could also undermine trust in the system's reliability.

## References

Add any relevant links to documentation or code

## Proof of Concept

The loop continues until either the `currentBorrower` is the zero address, the `totals.remainingDebtToRedeem` is zero, or the `_maxIterations` counter reaches zero. The `_maxIterations` counter is decremented inside the loop body.

The potential issue here is that if a user provides a large `_debt` value for redemption, and the CDPs involved have small debt values, the loop may need to iterate many times. If the loop iterations exceed the block gas limit, the transaction will revert, and the entire redemption process will fail.

```solidity
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("CdpManager DoS Test", function () {
  let cdpManager, deployer, user;

  beforeEach(async function () {
    [deployer, user] = await ethers.getSigners();

    // Assuming you have a deployment script or function to deploy your CdpManager and dependencies
    cdpManager = await deployCdpManagerAndDependencies();

    // Additional setup if necessary (e.g., minting and approving tokens, creating CDPs)
  });

  it("should demonstrate high gas consumption on redeemCollateral", async function () {
    // Simulate conditions leading to high gas consumption
    const debtValue = ethers.utils.parseUnits("1000", "ether"); // Large debt value for redemption
    const maxIterations = 1000; // High iteration limit to simulate potential DoS

    // Parameters for redeemCollateral (modify as per your contract's requirements)
    const firstRedemptionHint = "0x...";
    const upperPartialRedemptionHint = "0x...";
    const lowerPartialRedemptionHint = "0x...";
    const partialRedemptionHintNICR = ethers.utils.parseUnits("1", "ether"); // Example value
    const maxFeePercentage = ethers.utils.parseUnits("1", "percent"); // 1%

    // Monitor gas usage
    const tx = await cdpManager.connect(user).redeemCollateral(
      debtValue,
      firstRedemptionHint,
      upperPartialRedemptionHint,
      lowerPartialRedemptionHint,
      partialRedemptionHintNICR,
      maxIterations,
      maxFeePercentage
    );

    const receipt = await tx.wait();
    console.log(`Gas used for redemption: ${receipt.gasUsed.toString()}`);

    // Assert condition based on expected behavior (e.g., transaction succeeds but uses high gas)
    expect(receipt.gasUsed).to.be.gt(ethers.BigNumber.from("8000000")); // Example gas threshold
  });

  async function deployCdpManagerAndDependencies() {
    // Implement deployment logic for CdpManager and its dependencies
    // Return the deployed CdpManager contract instance
  }
});

```
