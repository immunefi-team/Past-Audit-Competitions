# #46520 \[SC-Low] ETH loss on \`selfCloseExitTo\` when redeeming to collateral

**Submitted on Jun 1st 2025 at 01:19:34 UTC by @Rhaydden for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46520
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

In `CollateralPool.selfCloseExitTo`, if a user passes `msg.value` (intended as an executor fee) but sets `_redeemToCollateral = true`, the contract never forwards or refunds that ETH. It remains locked in the pool contract resulting in permanent user fund loss.

## Vulnerability Details

Inside `_selfCloseExitTo`, the code handles f-asset redemption like this:

```
// … after calculating `requiredFAssets` …
if (requiredFAssets > 0) {
    if (requiredFAssets < assetManager.lotSize() || _redeemToCollateral) {
        // Non-payable call: msg.value is ignored
        assetManager.redeemFromAgentInCollateral(
            agentVault, _recipient, requiredFAssets
        );
    } else {
        // Cross-chain redemption: msg.value is forwarded as executor fee
        assetManager.redeemFromAgent{ value: msg.value }(
            agentVault, _recipient, requiredFAssets, _redeemerUnderlyingAddress, _executor
        );
    }
}

```

When the first branch is taken—either because the redemption amount is too small or `_redeemToCollateral==true`—the contract calls the non-payable `redeemFromAgentInCollateral` and never touches `msg.value`. That ETH is neither wrapped into `wNat` nor returned to the sender. Since there is no fallback path for raw ETH (the `receive()` guard only allows internal withdrawals), any ETH sent is irretrievably stuck.

## Impact Details

Any ETH sent as a fee under these conditions is locked in the pool contract. Users cannot retrieve it, nor can the protocol recover it via any existing withdrawal mechanism. Even accidental small-value calls lead to direct loss. This falls squarely under Permanent freezing of funds in the impacts in scope.

## Fix

In any branch that accepts msg.value, either

immediately call `wNat.deposit{value: msg.value}()` (via `_depositWNat()`) so it’s tracked as collateral, OR revert if `_redeemToCollateral` or `requiredFAssets < lotSize but msg.value > 0`

## References

<https://github.com/flare-labs-ltd/fassets//blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L363-L372>

## Proof of Concept

## Proof of Concept

Add this test to CollateralPool.ts:

```
    describe("self-close exit fee retention", async () => {
        it("should retain msg.value when redeemToCollateral=true", async () => {
            // enter pool to get tokens.
            // We put 10 ETH into the pool so the caller gets some pool tokens
            await collateralPool.enter(0, true, { value: ETH(10) });
            // Read the pool contract’s raw ETH balance before the exit
            const before = toBN(await web3.eth.getBalance(collateralPool.address));

            const tokenBalance = await collateralPoolToken.balanceOf(accounts[0]);
            // Define a 1 ETH “executor fee” to pass via msg.value
            const fee = ETH(1);
            // call self-close exit with redeemToCollateral=true and pass fee
            await collateralPool.selfCloseExitTo(tokenBalance, true, accounts[0], "", accounts[0], { value: fee });
            // Read the pool contract’s raw ETH balance after the exit
            const after = toBN(await web3.eth.getBalance(collateralPool.address));
            // Assert that the pool’s balance increased by exactly fee (1 ETH), showing that the ETH was stuck in the contract
            assertEqualBN(after.sub(before), fee, "msg.value should remain in contract");
        });
    });
```
