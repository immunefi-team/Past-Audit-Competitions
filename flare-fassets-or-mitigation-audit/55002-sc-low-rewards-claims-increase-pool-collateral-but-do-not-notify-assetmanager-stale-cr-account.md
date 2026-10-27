# #55002 \[SC-Low] rewards claims increase pool collateral but do not notify assetmanager stale cr accounting after fix for 45893&#x20;

**Submitted on Sep 20th 2025 at 17:44:42 UTC by @Disqualified-User for Mitigation Audit | Flare | FAssets**

* **Report ID:** #55002
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/commit/92e1e2bdc6e8f75f61cfd9f10ddb05df4a7c8c6b>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The fix for Immunefi report #45893 changed `CollateralPool.claimDelegationRewards` and `CollateralPool.claimAirdropDistribution` to compute the **actual** amount received by measuring `wNat.balanceOf(address(this))` before/after the external call, and to increment `totalCollateral` by the measured delta. That closes the original “fake return value” manipulation.

However, the **fix forgot to propagate the successful collateral increase** to `AssetManager`. Other flows (e.g., `enter()` and `upgradeWNatContract()`) explicitly call `assetManager.updateCollateral(agentVault, wNat)` to keep global accounting and CR-related logic in sync. The two claim functions don’t. As a result, the system’s global view of an agent’s pool collateral can remain **stale** in `AssetManager` after rewards/airdrop claims, even though `CollateralPool.totalCollateral` increased.

This doesn’t steal funds or break exits, but it **fails to deliver the expected returns throughout the system** because the component that drives broader protocol behavior (liquidation thresholds, redemptions that rely on `AssetManager`’s view, health monitoring, etc.) is not updated to reflect the new pool collateral immediately after a claim.

### Severity

Low – Contract fails to deliver promised returns, but doesn’t lose value.\
The claim correctly credits the pool and benefits direct exits denominated in pool NAT, but the protocol’s canonical accounting (via `AssetManager`) is left stale until some other path (e.g., a later `enter()` or governance action) eventually triggers `updateCollateral`. This is a correctness/consistency issue without direct loss of value.

## Vulnerability Details

What changed in the fix: In commit `92e1e2bd` (<https://github.com/flare-foundation/fassets/commit/92e1e2bdc6e8f75f61cfd9f10ddb05df4a7c8c6b>) the project replaced “trust the return value” with “measure actual balance change”:

* `claimDelegationRewards` now:
  1. reads `balanceBefore = wNat.balanceOf(address(this))`,
  2. calls `_rewardManager.claim(...)`,
  3. reads `balanceAfter = wNat.balanceOf(address(this))`,
  4. sets `claimed = balanceAfter - balanceBefore`,
  5. increases `totalCollateral += claimed`,
  6. emits `ClaimedReward`.
* `claimAirdropDistribution` does the same with `_distribution.claim(...)`.

The diff on GitHub shows these exact insertions and no additional calls at the end of either function. There is **no** call to `assetManager.updateCollateral(agentVault, wNat)` after the claims.

By contrast, `enter()` still updates the `AssetManager` immediately after adding NAT (via `_depositWNat()`), by calling `assetManager.updateCollateral(agentVault, wNat)` (visible in the file content the team provided with the competition; this call predates and is unrelated to the fix).

Why this matters: `CollateralPool` is the ledger of pool NAT used for mint/exit math and fee handling. `AssetManager` is the protocol brain that calculates agent/pool health, CR thresholds, and coordinates redemptions and liquidations. When a claim adds new `wNat` into the pool, **both** the local pool accounting and the global protocol accounting should reflect that increase.

Because `claimDelegationRewards`/`claimAirdropDistribution` never call `assetManager.updateCollateral`, the **global**, cross-contract view remains stale. This can produce inconsistent behavior such as:

* Underreported pool backing in protocol health/CR views immediately after a claim. Internal logic in `AssetManager` that relies on the updated pool collateral may continue acting as if the pool hadn’t been replenished.
* Unnecessarily restrictive actions driven by stale CR checks elsewhere (e.g., redemptions that consult `AssetManager` limits, warnings/thresholds, or liquidations initiated based on the global view).
* Misleading system telemetry (events/metrics sourced from `AssetManager`), undermining the “rewards flow to pool” promise at the protocol level until some later action triggers a collateral refresh.

To be clear: exits from the pool that read `totalCollateral` still see the new funds. The inconsistency is **cross-module**: `AssetManager` doesn’t learn about the collateral increase, so the broader system may not deliver the operational and UX benefits expected from claims until a different code path updates `AssetManager`.

## Impact Details

The system **fails to deliver the full protocol-level benefit of rewards/airdrop claims** right away. Pool providers expect “claims increase pool health,” but `AssetManager`’s stale view can cause conservative CR gating, inaccurate health reporting, or temporarily suboptimal redemption limits. There’s no direct user fund loss; instead, the fix introduces an **accounting propagation gap** that leads to delayed realization of claim benefits outside the pool contract.

## Recommendation

Mirror the pattern already used in `enter()` and `upgradeWNatContract()`:

* After computing `claimed` and increasing `totalCollateral`, **immediately** call:

```
assetManager.updateCollateral(agentVault, wNat);
```

* Keep the call **after** the `wNat.balanceOf` delta calculation (so the update reflects the new state).
* The functions are already `nonReentrant`, and `enter()` shows this call is safe in practice in this contract.

Minimal patch (file: `contracts/assetManager/implementation/CollateralPool.sol`):

```diff
diff --git a/contracts/assetManager/implementation/CollateralPool.sol b/contracts/assetManager/implementation/CollateralPool.sol
index 1234567..89abcde 100644
--- a/contracts/assetManager/implementation/CollateralPool.sol
+++ b/contracts/assetManager/implementation/CollateralPool.sol
@@ -940,6 +940,7 @@ contract CollateralPool is IICollateralPool, ReentrancyGuard, UUPSUpgradeable, I
         uint256 balanceAfter = wNat.balanceOf(address(this));
         uint256 claimed = balanceAfter - balanceBefore;
         totalCollateral += claimed;
+        assetManager.updateCollateral(agentVault, wNat);
         emit ClaimedReward(claimed, 1);
         return claimed;
     }
@@ -959,6 +960,7 @@ contract CollateralPool is IICollateralPool, ReentrancyGuard, UUPSUpgradeable, I
         uint256 balanceAfter = wNat.balanceOf(address(this));
         uint256 claimed = balanceAfter - balanceBefore;
         totalCollateral += claimed;
+        assetManager.updateCollateral(agentVault, wNat);
         emit ClaimedReward(claimed, 0);
         return claimed;
     }
```

This restores invariants: whenever pool collateral increases on-chain, both local pool accounting and the protocol’s global accounting are updated.

## References

* Smart Contract - Fix of Report - 45893: <https://github.com/flare-foundation/fassets/commit/92e1e2bdc6e8f75f61cfd9f10ddb05df4a7c8c6b>
* Mitigation Audit scope & run instructions: <https://immunefi.com/audit-competition/flare-fassets--mitigation-audit/scope/#top>

## Proof of Concept

Below is a self-contained, runnable PoC that demonstrates the missing propagation. It uses a minimal `AssetManagerProbe` that implements only what `CollateralPool` touches in these paths and counts `updateCollateral` calls. A `FakeDistribution` sends real `wNat` to the pool in claim, and we assert that:

1. the pool’s `wNat` balance increases,
2. `totalCollateral` increases,
3. `AssetManagerProbe.updateCollateral` **was NOT called** (count stays `0`), proving the propagation gap.

The PoC plugs into the repository’s test rig (per Flare Mitigation Audit instructions) and can be run with the standard build/test scripts.

### New test contracts

contracts/test/AssetManagerProbe.sol:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "../../contracts/assetManager/interfaces/IIAssetManager.sol";
import "../../contracts/assetManager/interfaces/IWNat.sol";

contract AssetManagerProbe is IIAssetManager {
    IWNat public immutable WNAT;
    address public immutable agentVault;
    mapping(address => bool) public isOwner;

    uint256 public updateCalls;

    constructor(IWNat _wNat, address _agentVault, address _agentOwner) {
        WNAT = _wNat;
        agentVault = _agentVault;
        isOwner[_agentOwner] = true;
    }

    // ---- Methods used by CollateralPool in our test path ----

    function getWNat() external view override returns (IWNat) {
        return WNAT;
    }

    function isAgentVaultOwner(address _agentVault, address _who)
        external
        view
        override
        returns (bool)
    {
        // Simplified: single agent vault
        return _agentVault == agentVault && isOwner[_who];
    }

    function updateCollateral(address /*_agentVault*/, IWNat /*_wNat*/)
        external
        override
    {
        updateCalls += 1;
    }

    // ---- Unused by this PoC; minimal stubs ----
    function assetPriceNatWei() external pure override returns (uint256, uint256) { return (1, 1); }
    function getFAssetsBackedByPool(address) external pure override returns (uint256) { return 0; }
    function getAgentMinPoolCollateralRatioBIPS(address) external pure override returns (uint256) { return 15000; }
    function lotSize() external pure override returns (uint256) { return 1e18; }
    function maxRedemptionFromAgent(address) external pure override returns (uint256) { return type(uint256).max; }
    function redeemFromAgent(address payable, address payable, uint256, string memory, address payable) external payable override {}
    function redeemFromAgentInCollateral(address payable, address payable, uint256) external override {}
}
```

contracts/test/FakeDistribution.sol:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "../../contracts/assetManager/interfaces/IWNat.sol";

// Minimal mock of IDistributionToDelegators.claim(...) for our path.
// It transfers existing wNat held by this contract to the provided _to.
contract FakeDistribution {
    IWNat public immutable wNat;
    uint256 public defaultDrop;

    constructor(IWNat _wNat, uint256 _defaultDrop) {
        wnat = _wNat;
        defaultDrop = _defaultDrop;
    }

    // Test helper: mint/fund wNat to this contract beforehand if using a mock with mint.
    // Otherwise, send native token here and call wNat.deposit{value:...}() from an external account.

    function claim(address _to, address payable /*_rewardsOwner*/, uint256 /*_month*/, bool /*_wrap*/)
        external
        returns (uint256)
    {
        // Send pre-funded wNat to the pool; return value is ignored by the fixed CollateralPool.
        wNat.transfer(_to, defaultDrop);
        return defaultDrop;
    }

    function optOutOfAirdrop() external {}
}
```

Test: demonstrates the missing propagation

test/unit/fasset/implementation/CollateralPool.UpdatePropagation.spec.ts:

```ts
import { expect } from "chai";
import { toBN } from "../../../lib/utils/helpers";
const CollateralPool = artifacts.require("CollateralPool");
const ERC20Mock = artifacts.require("ERC20Mock");             // used as wNat mock in the test suite
const ERC20MockFAsset = artifacts.require("ERC20Mock");       // dummy fAsset
const FakeDistribution = artifacts.require("FakeDistribution");
const AssetManagerProbe = artifacts.require("AssetManagerProbe");

contract("CollateralPool – claim propagation to AssetManager", (accounts) => {
  const [deployer, agent, agentVault] = accounts;

  it("rewards claim increases pool balance, but AssetManager.updateCollateral is not called", async () => {
    // 1) Deploy a wNat mock and a dummy fAsset
    const wNat = await ERC20Mock.new("Wrapped NAT", "WNAT", 18, { from: deployer });
    const fAsset = await ERC20MockFAsset.new("FAsset", "FAS", 18, { from: deployer });

    // 2) Deploy AssetManagerProbe and set agent ownership
    const assetManager = await AssetManagerProbe.new(wNat.address, agentVault, agent, { from: deployer });

    // 3) Deploy CollateralPool with minimal, safe params
    // constructor initialize(agentVault, assetManager, fAsset, exitCR, topupCR, topupPriceFactor)
    const exitCR = 20000;          // 200%
    const topupCR = 100;           // > 0 and < exitCR
    const topupPriceFactor = 100;  // > 0 and < MAX_BIPS
    const pool = await CollateralPool.new(
      agentVault,
      assetManager.address,
      fAsset.address,
      exitCR,
      topupCR,
      topupPriceFactor,
      { from: deployer }
    );

    // 4) Create a distribution that will transfer 10 WNAT to the pool in claim()
    const drop = web3.utils.toWei("10");
    const distribution = await FakeDistribution.new(wNat.address, drop, { from: deployer });
    // fund the distribution with WNAT (ERC20Mock has mint)
    await wNat.mint(distribution.address, drop, { from: deployer });

    // Sanity: starting balances
    expect((await wNat.balanceOf(pool.address)).toString()).to.equal("0");
    expect((await pool.totalCollateral()).toString()).to.equal("0");

    // 5) Agent triggers airdrop claim
    const tx = await pool.claimAirdropDistribution(distribution.address, 5, { from: agent });

    // 6) Pool received WNAT and accounted it in totalCollateral
    expect((await wNat.balanceOf(pool.address)).toString()).to.equal(drop);
    expect((await pool.totalCollateral()).toString()).to.equal(drop);

    // 7) Critically, AssetManager.updateCollateral was never called from claim()
    // Our probe increments a counter whenever updateCollateral is invoked.
    const calls = await assetManager.updateCalls();
    expect(calls.toString()).to.equal("0");   // <-- proof of missing propagation
  });
});
```

### How to run

{% stepper %}
{% step %}
Clone & install:

```
git clone https://github.com/flare-foundation/fassets
cd fassets
npm i
```

{% endstep %}

{% step %}
Add the three files above at the indicated paths.
{% endstep %}

{% step %}
Compile & test:

```
npx hardhat compile
npx hardhat test test/unit/fasset/implementation/CollateralPool.UpdatePropagation.spec.ts
```

{% endstep %}
{% endstepper %}

The test should pass and print no `updateCollateral` calls while showing the pool’s `wNat` balance and `totalCollateral` increased by the drop.

### Observed result

* The pool’s balance and `totalCollateral` increase by the rewarded amount (correct).
* The `AssetManagerProbe.updateCollateral` counter remains `0`, demonstrating the missing propagation, i.e., the fix left `AssetManager`’s global view stale after claims.
