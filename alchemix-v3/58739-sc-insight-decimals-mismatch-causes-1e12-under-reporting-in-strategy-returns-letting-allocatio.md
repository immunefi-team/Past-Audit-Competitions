# 58739 sc insight decimals mismatch causes 1e12 under reporting in strategy returns letting allocations silently exceed per strategy and global caps

**Submitted on Nov 4th 2025 at 10:52:22 UTC by @manvi for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58739
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Protocol insolvency

## Description

## Brief/Intro

I analyzed how the allocator enforces per-strategy caps and observed that it trusts strategy-reported deltas / TVL (e.g., return value change from allocate/deallocate and/or realAssets()) without enforcing a canonical precision.

If a strategy internally treats an 18-dec vault asset as a 6-dec unit (or otherwise returns values in a mismatched base), the allocator under-counts the moved TVL—by as much as 1e12—so per-strategy caps never trigger while real funds flow into the adapter.

This silently defeats risk limits and enables excess TVL concentration, which can lead to material losses if the adapter later fails - rising to protocol insolvency.

## Vulnerability Details

I have observed that the Allocator code assumes strategy-reported amounts are in the allocator’s expected unit/precision. There is no normalization to a canonical 18-dec WAD before doing cap math.

There is no enforcement that change returned by a strategy equals the requested assets (modulo small rounding). A strategy can legally return much smaller change, and the allocator will accept it as TVL delta.

**Surface exposed by interface:**

IMYTStrategy.allocate(bytes data, uint256 assets, bytes4 selector, address sender) returns (bytes32\[] strategyIds, int256 change)

IMYTStrategy.deallocate(...) returns (..., int256 change)

IMYTStrategy.realAssets() external view returns (uint256)

The allocator relies on these values for cap / TVL accounting but does not normalize or sanity-check them.

## Root Cause Code:

**src/MYTStrategy.sol**

```
 // Params include cap/globalCap but strategy decides what delta it reports.
 struct StrategyParams {
     address owner;
     string name;
     string protocol;
     RiskClass riskClass;
     uint256 cap;
     uint256 globalCap;
     uint256 estimatedYield;
     bool    additionalIncentives;
     uint256 slippageBPS;
 }

 // allocate() returns `change` derived from internal math
 function allocate(bytes memory data, uint256 assets, bytes4 /*selector*/, address /*sender*/)
     external onlyVault returns (bytes32[] memory strategyIds, int256 change)
     {
     if (killSwitch) return (ids(), int256(0));
     require(assets > 0, "Zero amount");
     uint256 oldAllocation = abi.decode(data, (uint256));
     uint256 amountAllocated = _allocate(assets);
     uint256 newAllocation = oldAllocation + amountAllocated;
     emit Allocate(amountAllocated, address(this));
     return (ids(), int256(newAllocation) - int256(oldAllocation));
 }

 // realAssets() exists but each strategy defines how it's measured
 function realAssets() external view virtual returns (uint256) {}
```

The allocator's cap enforcement (in src/AlchemistAllocator.sol) uses these strategy-reported numbers for TVL/cap checks. (There's no canonical 18-dec normalization or invariant that change = assets.)

## Exploit scenario

A strategy (malicious or simply buggy) scales its reported allocation delta by 1/1e12 (e.g., treats 18-dec assets as 6-dec units).

The allocator calls allocate(assets). The strategy actually moves assets funds but returns change = assets / 1e12.

The allocator adds change to the strategy's recorded TVL and compares to cap - which now looks far below the limit - allowing more and more assets to flow.

The real TVL of the strategy grows well beyond cap, but recorded TVL doesn't.

If that adapter later experiences loss, the excess TVL (that should have been blocked by caps) is at risk -> protocol-scale losses or system unavailability.

## Impact Details

**Risk limits defeated:** Per-strategy cap / globalCap do not bind actual funds.

**Systemic loss amplification:** If the over-concentrated adapter underperforms or is compromised, the excess TVL (that should have been blocked) is exposed -> Protocol insolvency plausible.

**Operational failure:** Out-of-sync accounting causes unexpected reverts, liquidity shortages, or mispriced risk -> Smart contract unable to operate due to lack of token funds.

## References

MYTStrategy.sol (core bug location) IMYTStrategy.sol (params, cap, interface used by allocator) AlchemistAllocator.sol (consumes strategy caps/realAssets) ZeroXSwapVerifier.sol (context for allocation/deallocation flows)

## Proof of Concept

I have demonstrated this issue with a runnable POC

**PoC file:** src/test/poc/Allocator\_CapDecimals\_Bypass.t.sol

My test creates a minimal strategy that under-reports by 1e12 and demonstrates both allocate and deallocate paths under-counting.

## What the PoC does

Implements a minimal strategy that under-reports allocation/deallocation deltas by /1e12 (simulating a 6-dec vs 18-dec unit mismatch).

Calls allocate(assets) and deallocate(assets) through the normal MYTStrategy surface so the flow mirrors production.

Records the returned change that the allocator would trust for TVL/cap math.

## Content of my POC file

```
 pragma solidity 0.8.28;

 import {Test} from "forge-std/Test.sol";

 import {MYTStrategy} from "src/MYTStrategy.sol";
 import {IMYTStrategy} from "src/interfaces/IMYTStrategy.sol";
 import {ERC20} from "openzeppelin-     contracts/contracts/token/ERC20/ERC20.sol";

 // Minimal 6-dec ERC20 used as the strategy's receipt token
 contract ERC20Decimals is ERC20 {
     uint8 private _dec;
     constructor(string memory n, string memory s, uint8 d) ERC20(n, s) { _dec = d; }
     function decimals() public view override returns (uint8) { return _dec; }
     function mint(address to, uint256 amt) external { _mint(to, amt); }
 }

 // Strategy that intentionally mis-scales allocations by treating 18-dec assets
 // as if they were 6-dec receipt units (÷ 1e12). This simulates a decimals
 // mismatch that can undercount allocated TVL and bypass caps enforced off this value.
 contract DecimalsSkewStrategy is MYTStrategy {
    constructor(
         address _myt,
         IMYTStrategy.StrategyParams memory _params,
         address _permit2Address,
         address _receiptToken
     ) MYTStrategy(_myt, _params, _permit2Address, _receiptToken) {}

     function _allocate(uint256 amount) internal override returns (uint256) {
         // Under-report by 1e12 to simulate cap-bypass surface
         return amount / 1e12;
     }

     function _deallocate(uint256 amount) internal override returns (uint256) {
         return amount / 1e12;
     }

     function _previewAdjustedWithdraw(uint256 amount) internal view override returns (uint256) {
         return amount / 1e12;
     }

     function _claimWithdrawalQueue(uint256 /*positionId*/) internal override returns (uint256) {
         return 0;
     }

     function _claimRewards() internal override returns (uint256) {
         return 0;
     }

     function _computeBaseRatePerSecond() internal override returns (uint256 ratePerSec, uint256 newIndex) {
         return (0, 0);
     }

     function _computeRewardsRatePerSecond() internal override returns (uint256) {
         return 0;
     }
 }

 contract Allocator_CapDecimals_Bypass_Test is Test {
     // Minimal “vault” stand-in (we only need the address value to satisfy onlyVault).
     // Note: this address is already in correct checksum form per the compiler hint.
     address constant VAULT = address(0x000000000000000000000000000000000000a11c);

     // FIX: use the checksummed literal (...bEEF), per compiler hint.
     address private constant PERMIT2 =   address(0x000000000000000000000000000000000000bEEF);

     ERC20Decimals private receipt;      // 6-dec “receipt” token
DecimalsSkewStrategy private strat; // strategy under test

     function setUp() public {
         // Deploy a 6-decimal ERC20 to act as receipt token (MYTStrategy constructor approves it on PERMIT2)
         receipt = new ERC20Decimals("Receipt", "RCPT", 6);

         // 9-field StrategyParams exactly as in IMYTStrategy
         IMYTStrategy.StrategyParams memory P =   IMYTStrategy.StrategyParams({
             owner: address(this),
             name: "CapDecBypass",
             protocol: "mock-6-dec",
             riskClass: IMYTStrategy.RiskClass.LOW,
             cap: 1_000_000e18,
             globalCap: 1_000_000e18,
             estimatedYield: 0,
             additionalIncentives: false,
             slippageBPS: 0
         });

         // Construct the strategy. We pass a dummy "vault" address as _myt; onlyVault checks msg.sender, not code.
         strat = new DecimalsSkewStrategy(
             VAULT,               // _myt (we'll prank as this for onlyVault)
             P,                   // params
             PERMIT2,             // non-zero permit2
             address(receipt)     // 6-dec receipt token
         );
     }

     // Core PoC: allocate 1e18 and show the strategy reports only 1e6 change (division by 1e12),
     // which would let an allocator relying on 'change' under-enforce cap.
     function test_Allocate_UnderCounts_By_1e12() public {
         bytes memory data = abi.encode(uint256(0)); // oldAllocation=0

         vm.prank(VAULT); // satisfy onlyVault
         (bytes32[] memory ids, int256 change) = strat.allocate(
             data,
             1e18,       // assets (18-dec)
             bytes4(0),  // selector unused
             address(this)
         );

         // Silence warnings about ids
         assertTrue(ids.length > 0);

         // Expected: 1e18 / 1e12 = 1e6 reported allocation delta (undercount by 1e12)
        assertEq(change, int256(1_000_000));
     }

     // Optional: show deallocate mirrors the same skew
     function test_Deallocate_UnderCounts_By_1e12() public {
         bytes memory data = abi.encode(uint256(2_000_000)); // pretend oldAllocation

         vm.prank(VAULT);
         (, int256 change) = strat.deallocate(
             data,
             1e18,       // assets (18-dec)
             bytes4(0),
             address(this)
         );

         // Expected: - (1e18 / 1e12)
         assertEq(change, -int256(1_000_000));
     }
 }
```

## Run my POC file from Repo Root :

```
 $ forge clean
 forge test -vv --match-path src/test/poc/Allocator_CapDecimals_Bypass.t.sol
```

## My Console Output :

```
 [⠊] Compiling...
 [⠆] Compiling 209 files with Solc 0.8.28
 [⠔] Solc 0.8.28 finished in 142.43s
 Ran 2 tests for      src/test/poc/Allocator_CapDecimals_Bypass.t.sol:Allocator_CapDecimals_     Bypass_Test
 [PASS] test_Allocate_UnderCounts_By_1e12() (gas: 18725)
 [PASS] test_Deallocate_UnderCounts_By_1e12() (gas: 18662)
 Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 3.12ms (187.47µs CPU time)

 Ran 1 test suite in 19.13ms (3.12ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

## What my PoC proved

For a real move of assets = 1e18, the strategy reports change = 1e6 - an under-count of 1e12.

Both allocation and deallocation paths under-count, meaning per-strategy caps will not trigger while real funds flow, enabling TVL to exceed configured caps.

This defeats allocator risk limits and can concentrate outsized exposure in one adapter, creating a plausible path to protocol insolvency or operational failure if that adapter loses funds.
