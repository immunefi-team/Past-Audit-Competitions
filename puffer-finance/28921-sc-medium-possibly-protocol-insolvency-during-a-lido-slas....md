# 28921 - \[SC - Medium] Possibly protocol insolvency during a LIDO slas...

Submitted on Mar 1st 2024 at 21:43:56 UTC by @yixxas for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28921

Report type: Smart Contract

Report severity: Medium

Target: <https://etherscan.io/address/0xd9a442856c234a39a81a089c06451ebaa4306a72>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

The critical issue in this report is how `totalAssets()` is computed wrongly during a LIDO slashing event due to using `getPendingLidoETHAmount()`, where the received value can be different from the requested value from LIDO withdrawal queue.

## Vulnerability Details

The protocol is making one critically incorrect assumption in its implementation, that is, when we request from ETH withdrawal from LIDO, the amount that we receive is always going to be the same as the amount receive once the request can be claimed from the queue. This invariant is broken when there is a LIDO slashing event.

`totalAssets()` is currently computed as seen below

```javascript
function totalAssets() public view virtual override returns (uint256) {
	return _ST_ETH.balanceOf(address(this)) + getELBackingEthAmount() + getPendingLidoETHAmount()
		+ address(this).balance;
} 
```

Importantly, `getPendingLidoETHAmount()` is used to track the amount of ETH backed by the vault. `getPendingLidoETHAmount()` is simply the value of `lidoLockedETH` tracked in puffer vault storage.

This value is incremented by the amount requested when we initiate a request from LIDO using `initiateETHWithdrawalsFromLido()`.

When we call `claimWithdrawalsFromLido()`, we receive ETH back in our vault.

```javascript
receive() external payable virtual {
	// If we don't use this pattern, somebody can create a Lido withdrawal, claim it to this contract
	// Making `$.lidoLockedETH -= msg.value` revert
	VaultStorage storage $ = _getPufferVaultStorage();
	if ($.isLidoWithdrawal) {
		$.lidoLockedETH -= msg.value;
	}
}
```

`lidoLockedETH` is then subtracted by the amount that is received.

Now, we can see where lies the issue. If we make a request of 1000 ETH, but only receive 900 ETH due to the LIDO slashing event, `lidoLockedETH` will have an additional 100 ETH that is still being treated as part of the vault's assets, but has actually vanished and should not be accounted for anymore.

While the current vault implementation does not support withdrawal/redeem, `_getPufferVaultStorage()` is persistent across V2 of the vault. In `PufferVaultV2.sol`, `totalAssets()` continues to be computed using the incorrect `getPendingLidoETHAmount()`, with can lead to an inflated `totalAssets()` value, wrongly signalling that the vault has more assets then it has.

Now that we have established `totalAssets()` to be incorrect (an inflated value), we will show why this impact is critical to the protocol.

## Impact Details

Puffer vault works like the typical ERC4626 using the share based accounting system to track each user's shares of the vault.

We will describe a simple scenerio that highlights the insolvancy protocol can experience.

Alice, Bob and Carol each holds equal shares of the vault. The vault's total assets is currently 600 ETH (now assuming all of them are in stETH), so each of them owns 200 ETH.

Protocol intiates an ETH withdrawal request from LIDO now, burns 200 stETH, and records `lidoLockedETH += 200 ether`. Total assets of the vault is still 600.

A LIDO slashing event has occured. All stETH is rebased down by 10%, and amount from LIDO's withdrawal queue is reduced by 10%.

We now have 400 \* 90/100 = 360 stETH in the vault, **BUT** `lidoLockedETH` is still 200 ETH, hence `totalAssets()` is computed as 560 ETH.

Note that this is an inflated value from the actual assets vault should be holding now, which is 600 \* 90/100 = 540 ETH.

Alice, knowing this, quickly redeems all of her shares and receive 1/3 of 560 ETH, which is \~186.667 ETH, but the amount she should receive is 1/3 of 540 ETH = 180 ETH.

News of this spread and every users of puffer finance is made aware of this. This leads to a bankrun situation, where the amount of ETH that users can redeem is more than the total amount of assets the vault actually holds, leading to protocol insolvancy. The last user/users will not be able to redeem their shares for the right amount of ETH.

I am sending this report as overall high severity, considering the low-mid likelihood of a LIDO slashing event, but the **CRITICAL** impact on the protocol.

## References

<https://stake.lido.fi/withdrawals/request>

In the LIDO's FAQ,

```
Why is the claimable amount may differ from my requested amount?

The amount you can claim may differ from your initial request due to a slashing occurrence and penalties. For these reasons, the total claimable reward amount could be reduced.
```

If the protocol needs further proof that claimed amount can be different from requested amount, happy to assist.

## Proof of Concept

As `withdrawal()` and `redeem()` is still disabled in the current implementation, it is difficult to show the full impact via a runnable PoC.

But I believe this vulnerability is trivial to see once we have established that `totalAssets()` is wrongly computed and I have described fully the impact on the protocol how it can happen.
