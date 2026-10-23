# #55046 \[SC-Insight] claimed rewards paid in legacy wnat after an upgrade are silently ignored by the balance delta fix

**Submitted on Sep 21st 2025 at 17:02:08 UTC by @Disqualified-User for** [**Mitigation Audit | Flare | FAssets**](https://immunefi.com/audit-competition/flare-fassets--mitigation-audit)

* **Report ID:** #55046
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/commit/92e1e2bdc6e8f75f61cfd9f10ddb05df4a7c8c6b>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The mitigation for #45893 changed `CollateralPool.claimDelegationRewards` and `CollateralPool.claimAirdropDistribution` to ignore the external call’s return value and instead compute a balance delta on the pool’s configured `wNat` token:

* read `balanceBefore = wNat.balanceOf(address(this))`
* call the external `claim(...)`
* read `balanceAfter = wNat.balanceOf(address(this))`
* compute `claimed = balanceAfter - balanceBefore`
* update `totalCollateral += claimed` and emit `ClaimedReward`

That closes the original vulnerability. However, it introduces a new, low-severity failure mode whenever the protocol upgrades `wNat` (via `upgradeWNatContract`) and any reward source (a distribution or reward manager) continues paying out in the **legacy** wNat contract for a period of time after the upgrade. In that situation the pool *does* receive tokens (the old wNat), but the pool’s configured `wNat` balance does not change, so the measured delta is zero. `totalCollateral` remains unchanged and the `ClaimedReward` event reports `0`. No value is lost, but the economic benefit of the claim is not delivered to users or reflected in protocol-level accounting until an out-of-band cleanup happens.

### Severity

Low – Contract fails to deliver promised returns, but doesn’t lose value. The pool holds the tokens (legacy wNat), yet `totalCollateral` is not credited and the system behaves as though nothing was received.

## Vulnerability Details

The new claim logic credits only by the configured, current `wNat` balance delta. This is correct when the reward source transfers the same token instance. The code path looks like this (post-fix for #45893):

```solidity
function claimDelegationRewards(
    IRewardManager _rewardManager,
    uint24 _lastRewardEpoch,
    IRewardManager.RewardClaimWithProof[] calldata _proofs
)
    external
    onlyAgent
    nonReentrant
    returns (uint256)
{
    uint256 balanceBefore = wNat.balanceOf(address(this));
    _rewardManager.claim(address(this), payable(address(this)), _lastRewardEpoch, true, _proofs);
    uint256 balanceAfter = wNat.balanceOf(address(this));
    uint256 claimed = balanceAfter - balanceBefore;
    totalCollateral += claimed;
    emit ClaimedReward(claimed, 1);
    return claimed;
}

function claimAirdropDistribution(
    IDistributionToDelegators _distribution,
    uint256 _month
)
    external
    onlyAgent
    nonReentrant
    returns(uint256)
{
    uint256 balanceBefore = wNat.balanceOf(address(this));
    _distribution.claim(address(this), payable(address(this)), _month, true);
    uint256 balanceAfter = wNat.balanceOf(address(this));
    uint256 claimed = balanceAfter - balanceBefore;
    totalCollateral += claimed;
    emit ClaimedReward(claimed, 0);
    return claimed;
}
```

The contract also supports upgrading the pool’s `wNat` instance:

```solidity
function upgradeWNatContract(IWNat _newWNat)
    external
    onlyAssetManager
    nonReentrant
{
    if (_newWNat == wNat) return;
    uint256 balance = wNat.balanceOf(address(this));
    internalWithdrawal = true;
    wNat.withdraw(balance);
    internalWithdrawal = false;
    _newWNat.deposit{value: balance}();
    wNat = _newWNat;
    assetManager.updateCollateral(agentVault, wNat);
}
```

After an upgrade, if a distribution or reward manager remains configured to send tokens from the old wNat, the pool’s balance of that old token increases while the configured `wNat.balanceOf(this)` does not change. The fix therefore computes `claimed == 0`, and the credit is skipped. The funds sit on the pool contract under the legacy token address, but they are invisible to pool math and to protocol-level accounting, and exits won’t reflect them. This is exactly “returns not delivered” with no value loss.

## Impact Details

Users rightly expect that a successful claim increases the pool’s collateral and immediately benefits them in exits and health metrics. After a `wNat` upgrade, if any reward source keeps paying in the legacy `wNat`, claims appear to work on-chain (the external `claim` succeeds and tokens move), but `totalCollateral` remains unchanged and `ClaimedReward` logs `0`. Holders temporarily get none of the expected economic benefit. The value is not stolen; it is stranded and untracked until an operator notices and takes corrective action.

## Recommendation

Make this failure mode explicit and impossible to miss. The cleanest approach is:

1. Track the **previous** wNat as `legacyWNat` when `upgradeWNatContract` executes.
2. In each claim function, record `legacyBefore = legacyWNat.balanceOf(this)` before the external call and `legacyAfter` after it.
3. If `legacyAfter > legacyBefore`, **revert** the claim with a clear error (for example, `claim-paid-in-legacy-wnat`). That makes misconfiguration loud, so operators fix the reward source to the new wNat.

Optionally, emit a specific event if you prefer not to revert, but don’t silently accept and ignore a nonzero legacy delta.

This preserves the security property of the fix while preventing “we claimed but nothing changed” confusion following a wNat upgrade.

## References

* Smart Contract - Fix of Report - 45893: <https://github.com/flare-foundation/fassets/commit/92e1e2bdc6e8f75f61cfd9f10ddb05df4a7c8c6b>
* Flare Mitigation Audit: <https://immunefi.com/audit-competition/flare-fassets--mitigation-audit/scope/#top>

## Proof of Concept

The PoC demonstrates that when a distribution pays out in legacy wNat (wNat v1) while the pool is configured to use wNat v2, the pool receives tokens but `totalCollateral` remains unchanged because the balance-delta is measured against the configured wNat only.

The PoC uses the repository’s test harness and includes two ERC-20 mocks to represent wNat v1 (legacy) and wNat v2 (current). The pool is configured to use wNat v2. A distribution intentionally transfers wNat v1 to the pool during `claimAirdropDistribution`. The test shows the pool’s balance of legacy wNat increases by 10e18, but `totalCollateral` and the current wNat balance remain unchanged and the claim credits zero.

Files (as used in the PoC):

contracts/test/AssetManagerProbe.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "../../contracts/assetManager/interfaces/IIAssetManager.sol";
import "../../contracts/assetManager/interfaces/IWNat.sol";

/**
 * Minimal AssetManager stub for testing CollateralPool claim paths.
 * It reports a configured wNat, recognizes one agent owner, and stubs everything else.
 */
contract AssetManagerProbe is IIAssetManager {
    IWNat public immutable WNAT;
    address public immutable AGENT_VAULT;
    mapping(address => bool) public isOwner;

    constructor(IWNat _wNat, address _agentVault, address _agentOwner) {
        WNAT = _wNat;
        AGENT_VAULT = _agentVault;
        isOwner[_agentOwner] = true;
    }

    // ---- Methods actually used by CollateralPool in the PoC path ----

    function getWNat() external view override returns (IWNat) {
        return WNAT;
    }

    function isAgentVaultOwner(address _agentVault, address _who)
        external
        view
        override
        returns (bool)
    {
        return _agentVault == AGENT_VAULT && isOwner[_who];
    }

    // ---- Remaining interface stubs (unused in this PoC) ----

    function updateCollateral(address, IWNat) external override {}
    function assetPriceNatWei() external pure override returns (uint256, uint256) { return (1, 1); }
    function getFAssetsBackedByPool(address) external pure override returns (uint256) { return 0; }
    function getAgentMinPoolCollateralRatioBIPS(address) external pure override returns (uint256) { return 15000; }
    function lotSize() external pure override returns (uint256) { return 1e18; }
    function maxRedemptionFromAgent(address) external pure override returns (uint256) { return type(uint256).max; }
    function redeemFromAgent(
        address payable, address payable, uint256, string memory, address payable
    ) external payable override {}
    function redeemFromAgentInCollateral(
        address payable, address payable, uint256
    ) external override {}
}
```

contracts/test/FakeDistributionLegacyWnat.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "../../contracts/assetManager/interfaces/IWNat.sol";

/**
 * Distribution that deliberately pays in a legacy wNat token (wNatV1),
 * while the pool is configured to use a different wNat (wNatV2).
 */
contract FakeDistributionLegacyWnat {
    IWNat public immutable wNatV1;
    uint256 public immutable drop;

    constructor(IWNat _wNatV1, uint256 _drop) {
        wNatV1 = _wNatV1;
        drop = _drop;
    }

    // Matches the signature CollateralPool expects:
    //   claim(address to, address payable rewardsOwner, uint256 month, bool wrap)
    function claim(
        address _to,
        address payable /*_rewardsOwner*/,
        uint256 /*_month*/,
        bool /*_wrap*/
    ) external returns (uint256) {
        // Transfer legacy wNat to the pool. Return value is ignored by CollateralPool after the fix.
        wNatV1.transfer(_to, drop);
        return drop;
    }

    function optOutOfAirdrop() external {}
}
```

test/unit/fasset/implementation/CollateralPool.LegacyWNatClaim.spec.ts

```ts
import { expect } from "chai";
const CollateralPool = artifacts.require("CollateralPool");
const ERC20Mock = artifacts.require("ERC20Mock"); // ERC20 used as stand-in for wNat in tests
const FakeDistributionLegacyWnat = artifacts.require("FakeDistributionLegacyWnat");
const AssetManagerProbe = artifacts.require("AssetManagerProbe");

contract("CollateralPool — claim paid in legacy wNat is ignored by balance-delta", (accounts) => {
  const [deployer, agent, agentVault] = accounts;

  it("claim transfers legacy wNat to pool yet credits 0 collateral", async () => {
    // Two token instances to simulate wNat v1 (legacy) and wNat v2 (current)
    const wNatV1 = await ERC20Mock.new("wNat V1", "WNAT1", 18, { from: deployer });
    const wNatV2 = await ERC20Mock.new("wNat V2", "WNAT2", 18, { from: deployer });

    // Minimal AssetManager that declares wNatV2 as the current wNat
    const assetManager = await AssetManagerProbe.new(wNatV2.address, agentVault, agent, { from: deployer });

    // Deploy the pool configured for the "current" wNat (wNatV2)
    // constructor initialize(agentVault, assetManager, fAsset, exitCR, topupCR, topupPriceFactor)
    const exitCR = 20000, topupCR = 100, topupPriceFactor = 100;
    const dummyFAsset = "0x0000000000000000000000000000000000000001"; // not used in this test path
    const pool = await CollateralPool.new(
      agentVault,
      assetManager.address,
      dummyFAsset,
      exitCR,
      topupCR,
      topupPriceFactor,
      { from: deployer }
    );

    // Distribution that will pay out in wNatV1 (legacy)
    const drop = web3.utils.toWei("10");
    const dist = await FakeDistributionLegacyWnat.new(wNatV1.address, drop, { from: deployer });

    // Fund the distribution with legacy wNat for the transfer
    await wNatV1.mint(dist.address, drop, { from: deployer });

    // Sanity: the pool starts with zero balances
    expect((await wNatV1.balanceOf(pool.address)).toString()).to.equal("0");
    expect((await wNatV2.balanceOf(pool.address)).toString()).to.equal("0");
    expect((await pool.totalCollateral()).toString()).to.equal("0");

    // Agent triggers airdrop claim; dist sends wNatV1 (legacy) to the pool
    await pool.claimAirdropDistribution(dist.address, 1, { from: agent });

    // The pool indeed received legacy wNat tokens...
    expect((await wNatV1.balanceOf(pool.address)).toString()).to.equal(drop);

    // ...but the configured wNatV2 balance didn't change, so the delta was 0
    expect((await wNatV2.balanceOf(pool.address)).toString()).to.equal("0");

    // Therefore totalCollateral remained unchanged and rewards were not credited
    expect((await pool.totalCollateral()).toString()).to.equal("0");
  });
});
```

Step-by-step reproduction:

{% stepper %}
{% step %}

### 1. Clone and install

Clone the repository and install dependencies.
{% endstep %}

{% step %}

### 2. Add PoC files

Save the two contracts under `contracts/test/` and the test file under `test/unit/fasset/implementation/`.
{% endstep %}

{% step %}

### 3. Compile

Run:

```
npx hardhat compile
```

{% endstep %}

{% step %}

### 4. Run the test

Run:

```
npx hardhat test test/unit/fasset/implementation/CollateralPool.LegacyWNatClaim.spec.ts
```

{% endstep %}

{% step %}

### 5. Observe

The pool’s balance of legacy wNat increases by `10e18`, while the configured wNat balance and `totalCollateral` remain unchanged (`0` → `0`). This demonstrates “returns not delivered” even though the contract holds the tokens.
{% endstep %}
{% endstepper %}

## Patch

The submitted patch adds a `legacyWNat` slot set during `upgradeWNatContract` and makes both claim functions revert if a legacy wNat balance increases during the claim. This prevents silent under-crediting and forces misconfigured reward sources to be updated after a wNat upgrade.

Key diff excerpt (conceptual):

```diff
+    IWNat public legacyWNat; // set on upgrade; used to detect legacy payouts in claim paths
...
+        uint256 legacyBefore = address(legacyWNat) != address(0) ? legacyWNat.balanceOf(address(this)) : 0;
         _rewardManager.claim(...);
+        uint256 legacyAfter = address(legacyWNat) != address(0) ? legacyWNat.balanceOf(address(this)) : 0;
+        if (legacyAfter > legacyBefore) {
+            revert("claim-paid-in-legacy-wnat");
+        }
...
+        legacyWNat = wNat;
```

The patch keeps the security benefits of the balance-delta fix, adds zero gas cost on the common path (no legacy configured), and turns a long-tail integration hazard into an explicit, debuggable error.
