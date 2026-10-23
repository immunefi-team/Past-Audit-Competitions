# 51847 sc critical dos via dust leftover in erc 20 approvals

**Submitted on Aug 6th 2025 at 07:52:55 UTC by @BeastBoy for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51847
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:**
  * Protocol insolvency
  * Permanent freezing of funds

## Description

The `_oneInchHelper` does:

```solidity
depositAsset.safeApprove(address(aggregator), depositAmount);
(supportedAssetAmount,) = aggregator.swap(executor, desc, data);
```

Because 1inch supports partial fills, it may pull only 995 of 1,000 USDT and return 5 USDT to the wrapper. The allowance is decremented by only the 995 consumed, leaving 5 units of non-zero allowance behind, so the next `safeApprove(..., newAmount)` becomes a forbidden non-zero→non-zero change and reverts.

In the vault deposit path the code calculates shares via:

```solidity
uint256 shares = depositAmount.mulDivDown(ONE_SHARE, rate);
vault.enter(msg.sender, asset, depositAmount, to, shares);
```

ERC-4626–style math rounds down, so depositing 1,000 units might mint only 999 shares, leaving 1 unit of asset stuck in the vault. That residual asset can combine with other dust vectors to block future approvals or withdrawals.

When interacting with a fee-on-transfer token the proxy calls:

```solidity
asset.safeApprove(spender, 100);
spender.transferFrom(proxy, target, 100);
// token charges 1% fee, net = 99
_allowances[proxy][spender] -= 99; // leaves 1 unit of allowance
```

That 1-unit dust makes the next `approve(…,50)` a non-zero→non-zero change and permanently DoSes that token’s flow.

This issue also exists in other functions like `TellerWithMultiAssetSupportPredicateProxy:deposit` & `TellerWithMultiAssetSupportPredicateProxy:depositAndBridge`.

## Impact

Any of these tiny “dust” scenarios—partial fills, rounding, or fees—leaves a residual allowance or balance that blocks all future `safeApprove` calls, causing an irreversible denial of service for that asset.

{% hint style="warning" %}
Residual non-zero allowances caused by partial token consumption, rounding, or fees can permanently prevent subsequent non-zero allowance updates for tokens that forbid non-zero → non-zero transitions (e.g., USDT). This results in a DoS that can freeze deposits/withdrawals or cause protocol insolvency.
{% endhint %}

## Recommendation

Always reset the existing allowance to zero before setting a new one:

```solidity
token.safeApprove(spender, 0);
token.safeApprove(spender, desiredAmount);
```

or perform a one-time infinite approval (`type(uint256).max`) to eliminate all non-zero→non-zero transitions.

{% hint style="info" %}
Either zero-then-set or use infinite approval are acceptable mitigations. Choose infinite approval only if trust and security considerations for that spender are acceptable.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Setup and initial approval

```solidity
// Setup USDT (assuming it's available in MainnetAddresses)
ERC20 USDT = ERC20(0xdAC17F958D2ee523a2206206994597C13D831ec7);

// Add USDT as supported asset
teller.addAsset(USDT);
accountant.setRateProviderData(USDT, true, address(0)); // Pegged to base

// Deal USDT to test contract (USDT has 6 decimals)
uint256 usdtAmount = 1000e6; // 1000 USDT
deal(address(USDT), address(this), usdtAmount);

// First approval - this should work fine
USDT.safeApprove(address(boringVault), usdtAmount);

console.log("First approval successful");
console.log("USDT allowance:", USDT.allowance(address(this), address(boringVault)));
```

{% endstep %}

{% step %}

### Simulate partial consumption

```solidity
// Simulate partial consumption by manually transferring some USDT
// This simulates what happens when DEX aggregators or vaults don't consume the full allowance
uint256 partialAmount = 999e6; // Transfer slightly less than approved amount
USDT.safeTransferFrom(address(this), address(boringVault), partialAmount);

console.log("After partial transfer:");
console.log("USDT allowance remaining:", USDT.allowance(address(this), address(boringVault)));
```

{% endstep %}

{% step %}

### Attempt to set a new non-zero allowance (reverts)

```solidity
// Try to approve again - this should REVERT with USDT
uint256 newApprovalAmount = 500e6;

// This will revert because USDT doesn't allow changing non-zero allowance to another non-zero value
vm.expectRevert();
USDT.safeApprove(address(boringVault), newApprovalAmount);

console.log("Second approval reverted as expected - VULNERABILITY CONFIRMED");
```

{% endstep %}

{% step %}

### Demonstrate the fix: reset to zero then approve

```solidity
// Reset to zero first, then approve
USDT.safeApprove(address(boringVault), 0);
console.log("Reset allowance to zero");

USDT.safeApprove(address(boringVault), newApprovalAmount);
console.log("New approval successful after reset:", USDT.allowance(address(this), address(boringVault)));
```

{% endstep %}
{% endstepper %}
