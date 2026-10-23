# #55049 \[SC-Insight] there is a issue related that the msg value not returned to payer in self close exit

**Submitted on Sep 21st 2025 at 17:44:47 UTC by @XDZIBECX for** [**Mitigation Audit | Flare | FAssets**](https://immunefi.com/audit-competition/flare-fassets--mitigation-audit)

* Report ID: #55049
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/flare-foundation/fassets/commit/03304ecf8110fd32f94620f111e2593f6969d573>
* Impacts:
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief / Intro

A change in CollateralPool’s self-close exit flow mis-forwards the caller’s msg.value to the redemption recipient instead of refunding the payer. Any user who includes ETH when no executor is used or when the flow takes the collateral path will lose that ETH to the recipient. In self-close exits, ETH is only needed if an executor is specified for underlying-chain redemption; therefore the function should ignore and refund any attached ETH when an executor is not used. The current code instead forwards this unintended ETH to the redemption recipient whenever no executor is used or when the flow takes the collateral path (for example, redeem-to-collateral or requiredFAssets < lotSize). A user who mistakenly includes ETH (due to a UI default or script error) effectively “donates” it to the recipient, causing user loss.

### Vulnerability Details

When `returnFunds` is true, the contract forwards the caller’s msg.value to the recipient `_recipient` instead of refunding the payer (`msg.sender`).

Vulnerable snippet:

```solidity
// redeem f-assets if necessary
bool returnFunds = true;
if (requiredFAssets > 0) {
    if (requiredFAssets < assetManager.lotSize() || _redeemToCollateral) {
        assetManager.redeemFromAgentInCollateral(agentVault, _recipient, requiredFAssets);
    } else {
        returnFunds = _executor == address(0);
        assetManager.redeemFromAgent{ value: returnFunds ? 0 : msg.value }(
            agentVault, _recipient, requiredFAssets, _redeemerUnderlyingAddress, _executor
        );
    }
}
// ...
if (returnFunds) {

    Transfers.transferNAT(_recipient, msg.value); // <-- issue: refunds to recipient instead of payer
}
```

The accidental msg.value should be refunded to the caller `msg.sender`, not gifted to `_recipient`.

### Impact Details

If users call the self-close exit without an executor or the flow uses the collateral path, any ETH they accidentally attach to the transaction is incorrectly forwarded to the redemption recipient instead of being refunded to them. A simple UI default or script mistake can cause the caller to lose ETH to a third party, producing guaranteed user loss with no benefit to the protocol.

Fix: change the refund target when no executor is used — in `_selfCloseExitTo`, replace sending msg.value to `_recipient` with refunding it to `msg.sender`, ensuring accidental ETH is returned to the payer and removing the griefing risk.

## References

* <https://github.com/flare-foundation/fassets/blob/d274320418134194cf74f69f95326ca40e2c1fed/contracts/collateralPool/implementation/CollateralPool.sol#L309C1-L312C10>

## Proof of Concept

<details>

<summary>Unit test PoC (run in test/unit/collateralPool/CollateralPool.ts)</summary>

Two test cases show the griefing: when redeeming to collateral and when requiredFAssets < lotSize. Both demonstrate the sender losing their attached msg.value while the recipient gains it.

Test 1 (redeemToCollateral = true):

```ts
it("griefing (redeemToCollateral=true): recipient gets sender's msg.value when no executor", async () => {
    // Setup pool and fees so self-close requires redeeming
    await collateralPool.enter({ value: ETH(10), from: accounts[0] });
    await givePoolFAssetFees(ETH(10));
    await collateralPool.enter({ value: ETH(3), from: accounts[0] });

    // Withdraw fees and approve fassets for self-close
    const exitTokens = ETH(10);
    await collateralPool.withdrawFees(await collateralPool.fAssetFeesOf(accounts[0]), { from: accounts[0] });
    await fAsset.approve(collateralPool.address, ETH(10), { from: accounts[0] });

    const unneededExecutorFee = ETH(1);
    const tokens = await collateralPoolToken.balanceOf(accounts[0]);
    const requiredF = await collateralPool.fAssetRequiredForSelfCloseExit(tokens);
    const lotSize = await assetManager.lotSize();

    console.log("[A] redeemToCollateral=true");
    console.log("[A] lotSize:", lotSize.toString());
    console.log("[A] requiredFAssets:", requiredF.toString());
    console.log("[A] executor:", ZERO_ADDRESS);

    const senderBefore = toBN(await web3.eth.getBalance(accounts[0]));
    const recipientBefore = toBN(await web3.eth.getBalance(accounts[2]));

    const receipt = await collateralPool.selfCloseExitTo(
        exitTokens,
        true,               // redeem to collateral → collateral path
        accounts[2],
        "underlying_A",
        ZERO_ADDRESS,       // no executor
        { from: accounts[0], value: unneededExecutorFee }
    );

    const holderDelta = await calculateReceivedNat(receipt, accounts[0]);
    const recipientDelta = await calculateReceivedNat(receipt, accounts[2]);
    const senderAfter = toBN(await web3.eth.getBalance(accounts[0]));
    const recipientAfter = toBN(await web3.eth.getBalance(accounts[2]));

    console.log("[A] holderReceivedNat (delta):", holderDelta.toString());
    console.log("[A] recipientReceivedNat (delta):", recipientDelta.toString());
    console.log("[A] sender NAT before/after:", senderBefore.toString(), senderAfter.toString());
    console.log("[A] recipient NAT before/after:", recipientBefore.toString(), recipientAfter.toString());

    // Griefing proof: sender loses msg.value; recipient gains msg.value on top of natShare
    assertEqualBN(holderDelta, unneededExecutorFee.neg(), "sender loses msg.value");
    assert.isTrue(recipientDelta.gte(exitTokens), "recipient should receive at least natShare");
    assertEqualBN(recipientDelta.sub(exitTokens), unneededExecutorFee, "recipient receives sender's msg.value (griefing)");
});
```

Test 2 (requiredFAssets < lotSize):

```ts
it("griefing (requiredFAssets < lotSize): recipient gets sender's msg.value when no executor", async () => {
  // Force requiredFAssets < lotSize
  await assetManager.setLotSize(ETH(1_000));

  await givePoolFAssetFees(ETH(50));
  const natToEnter = await poolFAssetFeeNatValue();
  await collateralPool.enter({ value: natToEnter });

  const tokens = await collateralPoolToken.balanceOf(accounts[0]);
  await collateralPool.withdrawFees(ETH(50), { from: accounts[0] });
  await fAsset.approve(collateralPool.address, ETH(50), { from: accounts[0] });

  const requiredF = await collateralPool.fAssetRequiredForSelfCloseExit(tokens);
  const lotSize = await assetManager.lotSize();
  console.log("[B] redeemToCollateral=false; requiredFAssets<lotSize path");
  console.log("[B] lotSize:", lotSize.toString());
  console.log("[B] requiredFAssets:", requiredF.toString());
  console.log("[B] executor:", ZERO_ADDRESS);
  assert.isTrue(requiredF.gt(toBN(0)));
  assert.isTrue(requiredF.lt(lotSize));

  // Snapshot expected natShare pre-state
  const expectedNatShare = await (async () => {
    const poolTokenSupply = await collateralPoolToken.totalSupply();
    const poolCollateral = await collateralPool.totalCollateral();
    return tokens.mul(poolCollateral).div(poolTokenSupply);
  })();
  console.log("[B] expectedNatShare (pre):", expectedNatShare.toString());

  const unneededExecutorFee = ETH(1);
  const senderBefore = toBN(await web3.eth.getBalance(accounts[0]));
  const recipientBefore = toBN(await web3.eth.getBalance(accounts[2]));

  const receipt = await collateralPool.selfCloseExitTo(
    tokens,
    false,                 // prefer underlying, but requiredF<lotSize forces collateral path
    accounts[2],
    "underlying_B",
    ZERO_ADDRESS,          // no executor -> returnFunds=true (vulnerable)
    { from: accounts[0], value: unneededExecutorFee }
  );

  // Collateral path event (not underlying)
  await expectEvent.notEmitted.inTransaction(receipt.tx, assetManager, "AgentRedemption");
  await expectEvent.inTransaction(receipt.tx, assetManager, "AgentRedemptionInCollateral");

  const holderDelta = await calculateReceivedNat(receipt, accounts[0]);
  const recipientDelta = await calculateReceivedNat(receipt, accounts[2]);
  const senderAfter = toBN(await web3.eth.getBalance(accounts[0]));
  const recipientAfter = toBN(await web3.eth.getBalance(accounts[2]));

  console.log("[B] holderReceivedNat (delta):", holderDelta.toString());
  console.log("[B] recipientReceivedNat (delta):", recipientDelta.toString());
  console.log("[B] sender NAT before/after:", senderBefore.toString(), senderAfter.toString());
  console.log("[B] recipient NAT before/after:", recipientBefore.toString(), recipientAfter.toString());

  // Proof of griefing: recipient gets natShare + sender's msg.value
  assertEqualBN(holderDelta, unneededExecutorFee.neg(), "sender loses msg.value");
  assert.isTrue(recipientDelta.gte(expectedNatShare), "recipient received at least natShare");
  assertEqualBN(recipientDelta.sub(expectedNatShare), unneededExecutorFee, "recipient also received sender's msg.value");
});
```

Observed results from the tests:

```
[A] redeemToCollateral=true
[A] lotSize: 1
[A] requiredFAssets: 10000000000000000000
[A] executor: 0x0000000000000000000000000000000000000000
[A] holderReceivedNat (delta): -1000000000000000000
[A] recipientReceivedNat (delta): 11000000000000000000
[A] sender NAT before/after: 99999999998986903959196660915342 99999999998985903206176949299060
[A] recipient NAT before/after: 100000000000000000000000000000000 100000000000011000000000000000000

[B] redeemToCollateral=false; requiredFAssets<lotSize path
[B] lotSize: 1000000000000000000000
[B] requiredFAssets: 50000000000000000000
[B] executor: 0x0000000000000000000000000000000000000000
[B] expectedNatShare (pre): 25000000000000000000
[B] holderReceivedNat (delta): -1000000000000000000
[B] recipientReceivedNat (delta): 26000000000000000000
[B] sender NAT before/after: 99999999998974904176847942150777 99999999998973903524511641055343
[B] recipient NAT before/after: 100000000000000000000000000000000 100000000000026000000000000000000
```

Both tests confirm that the sender loses the attached msg.value and the recipient gains it on top of their expected NAT share (griefing).

</details>

## Recommended Fix

In the code path where `returnFunds` is set true (no executor is used and the function did not consume msg.value in redeem call), refund `msg.value` to `msg.sender` instead of `_recipient`. This ensures accidental ETH is returned to the payer.

(Do not change any links or references above.)
