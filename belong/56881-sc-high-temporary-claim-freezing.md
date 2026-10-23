# 56881 sc high temporary claim freezing

**Submitted on Oct 21st 2025 at 13:41:16 UTC by @shadowHunter for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #56881
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/platform/Factory.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief / Intro

A temporary claim freezing vulnerability exists in `RoyaltiesReceiverV2` when factory parameters are updated via the Factory. Specifically:

* If fees are reduced for a beneficiary after partial payouts, subsequent claims by other parties may revert temporarily.
* This can freeze funds, delaying payouts to platform or referral beneficiaries until the contract state or allocations are reverted back.

Note: This is not due to incorrect factory params but due to past claims which have not yet been claimed. So the issue occurs for genuine inputs.

## Vulnerability Details

{% stepper %}
{% step %}

### Initial Setup

`RoyaltiesReceiverV2` initialized with:

* Creator: 50%
* Platform: 50%
* No referral

Funded with 100 ETH.
{% endstep %}

{% step %}

### Step — Partial Release

Creator claims 50 ETH. Platform has not yet claimed.
{% endstep %}

{% step %}

### Step — Fee Update via Factory

Factory updates royalties to:

* Creator: 40%
* Platform: 60%

Already-paid creator (50 ETH) exceeds new allocation (40 ETH).
{% endstep %}

{% step %}

### Step — Platform Claim Fails Temporarily

Platform calls `release()` which reverts as it demands 60 ETH even though only 50 ETH is left.

Effect: Payout is temporarily frozen to Platform.
{% endstep %}
{% endstepper %}

### Root Cause

* `RoyaltiesReceiverV2` calculates claimable shares dynamically based on current factory parameters.
* Factory `setFactoryParameters()` allows reducing a beneficiary’s share after payout.
* Contract does not account for historical payouts exceeding new allocations, causing temporary reverts.

## Impact

* Funds intended for platform/referral beneficiaries can be temporarily frozen.
* An attacker or even a benign update can cause griefing by reducing an already-partially-paid beneficiary's allocation, leading to reverts for other beneficiaries until state is reconciled.

## Recommendation

* Make new allocations apply only to future claims (do not retroactively reduce already-paid beneficiaries’ effective allocations).
* Before updating fees, compute and adjust for historical payouts (e.g., record and preserve already-accrued shares or introduce a pending-claims adjustment mechanism) so that past payouts cannot exceed new allocations.

## Proof of Concept

<details>

<summary>View PoC (Foundry test)</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

import "forge-std/Test.sol";
import "../contracts/v2/periphery/RoyaltiesReceiverV2.sol";
import "../contracts/v2/platform/Factory.sol";
import "openzeppelin-contracts/contracts/proxy/Clones.sol";

contract RoyaltiesReceiverStandaloneTest is Test {
    RoyaltiesReceiverV2 receiver;
    Factory factory;

    address creator = address(0x1);
    address platform = address(0x2);
    address public constant NATIVE_CURRENCY_ADDRESS = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;

    function setUp() public {
        // ===== Step 1: Deploy and initialize Factory clone =====
        Factory.FactoryParameters memory factoryParams = Factory.FactoryParameters({
            platformAddress: platform,
            signerAddress: address(this),
            defaultPaymentCurrency: address(0),
            platformCommission: 0,
            maxArraySize: 5,
            transferValidator: address(0)
        });

        Factory.RoyaltiesParameters memory royalties = Factory.RoyaltiesParameters({
            amountToCreator: 5000, // 50%
            amountToPlatform: 5000 // 50%
        });

        Factory.Implementations memory impls = Factory.Implementations({
            accessToken: address(0),
            creditToken: address(0),
            royaltiesReceiver: address(new RoyaltiesReceiverV2()), // master impl
            vestingWallet: address(0)
        });

        uint16[5] memory referralPercentages = [uint16(0), uint16(0), uint16(0), uint16(0), uint16(0)];

        Factory factoryImpl = new Factory();
        address payable factoryCloneAddr = payable(Clones.clone(address(factoryImpl)));
        factory = Factory(factoryCloneAddr);
        factory.initialize(factoryParams, royalties, impls, referralPercentages);

        // ===== Step 2: Deploy and initialize RoyaltiesReceiver clone =====
        RoyaltiesReceiverV2.RoyaltiesReceivers memory receivers = RoyaltiesReceiverV2.RoyaltiesReceivers({
            creator: creator,
            platform: platform,
            referral: address(0)
        });

        RoyaltiesReceiverV2 receiverImplementation = new RoyaltiesReceiverV2();
        address payable cloneAddr = payable(Clones.clone(address(receiverImplementation)));
        receiver = RoyaltiesReceiverV2(cloneAddr);
        receiver.initialize(receivers, factory, bytes32(0));

        // ===== Step 3: Fund the receiver =====
        vm.deal(address(this), 100 ether);
        payable(address(receiver)).transfer(100 ether);
    }

    function testPartialReleaseAndFeeUpdate() public {
        uint256 creatorBefore = creator.balance;
        uint256 platformBefore = platform.balance;

        // ===== Step 1: Only creator claims =====
        receiver.release(NATIVE_CURRENCY_ADDRESS, creator);

        uint256 creatorAfterStep1 = creator.balance;
        uint256 platformAfterStep1 = platform.balance;

        // Validate creator received 50 ETH (50%)
        assertEq(creatorAfterStep1 - creatorBefore, 50 ether);
        assertEq(platformAfterStep1 - platformBefore, 0);

        // ===== Step 2: Update royalties to 40% creator, 60% platform =====
        Factory.RoyaltiesParameters memory newRoyalties = Factory.RoyaltiesParameters({
            amountToCreator: 4000, // 40%
            amountToPlatform: 6000 // 60%
        });

        Factory.Implementations memory impls = factory.implementations();
        uint16[5] memory referralPercentages = [uint16(0), uint16(0), uint16(0), uint16(0), uint16(0)];

        factory.setFactoryParameters(factory.nftFactoryParameters(), newRoyalties, impls, referralPercentages);

        // ===== Step 3: Platform claims remaining =====
		// Bug: Fails due to insufficient balance
		vm.expectRevert();
        receiver.release(NATIVE_CURRENCY_ADDRESS, platform);
    }

    // Needed to receive ETH
    receive() external payable {}
}
```

Output: Release to Platform reverts due to insufficient ETH

</details>
