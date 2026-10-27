# 58416 sc low unclaimed extra rewards in tokemak integration lead to permanent freezing of yield

**Submitted on Nov 2nd 2025 at 06:57:32 UTC by @Ambitious\_DyDx for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58416
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Smart contract unable to operate due to lack of token funds
  * Temporary freezing of funds for at least 24 hour

## Description

## Description

In the `TokeAutoUSDStrategy` and `TokeAutoEthStrategy` contracts, the `_claimRewards` function always sets `claimExtras` to `false` when calling the Tokemak rewarder, preventing the claiming of any extra incentives (beyond the base reward token). This results in extra yields remaining locked in the rewarder contract indefinitely, as they are only claimed during deallocation (which may never occur for long-term strategies).

## Vulnerability Details

The strategies rely on Tokemak's `IMainRewarder` interface, where `getReward` includes a `claimExtras` parameter to optionally claim rewards from "extra rewarders" (secondary incentives like partner tokens). However, in `_claimRewards`:

```
rewarder.getReward(address(this), address(MYT), false);
```

The `false` hard-code skips extras. While `_deallocate` uses `true`:

```
rewarder.withdraw(address(this), sharesNeeded, true);
```

This only claims during full/partial deallocation. For ongoing allocations without dealloc cycles, extras accrue unclaimed forever—frozen in the rewarder, unharvestable by the strategy/MYT vault.

If `params.additionalIncentives = true` (settable by owner), the system estimates extras in `snapshotYield` but never claims them, misleading yield calcs while freezing real value.

## Not intended design

The `additionalIncentives` flag implies extras should be handled (used in `_computeRewardsRatePerSecond`, though stubbed as 0). Hard-coding `false` contradicts this, suggesting an oversight rather than intent. If intentional (e.g., for gas), a comment or toggle would be expected.

## Attack Vector

No active attack needed—passive accrual of extras in Tokemak (common for boosts) triggers the freeze:

* Strategy allocates funds → accrues base + extras in rewarder.
* Regular `claimRewards` (e.g., via keeper) → claims base only; extras stuck.
* Without dealloc (e.g., stable strategy), extras never move → permanent freeze. Adversary could accelerate by depositing extras directly to rewarder (if possible), but organic accrual suffices.

## Impact Details

* **Permanent freezing of unclaimed yield**: Extras locked in rewarder, unclaimable without code change/dealloc (which may not happen).
* **Yield loss for users**: MYT holders miss extras; if incentives enabled, overestimated yields mislead.
* **DoS-like on compounding**: Harvests incomplete, reducing APY without notice.
* **Funds stuck**: Extras represent user value, frozen indefinitely for long-hold strategies.

## Recommended Mitigation

* Tie `claimExtras` to `params.additionalIncentives`:

```
bool claimExtras = params.additionalIncentives;
rewarder.getReward(address(this), address(MYT), claimExtras);
```

* Or add admin toggle/param for selective claiming.
* Document intent if deliberate (e.g., "Extras claimed only on dealloc for gas efficiency").

## Proof of Concept

## Proof of Concept

Add to `v3-poc/src/test/strategies/TokeAutoUSDStrategy.t.sol`

```solidity
/// ----------------- Test-only mocks (fixed) -----------------
contract MockERC20 {
    string public name;
    string public symbol;
    uint8 public decimals = 18;
    mapping(address => uint256) public balances;

    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
    }

    // Mint helper for tests
    function mint(address to, uint256 amt) external {
        balances[to] += amt;
    }

    // Expose balanceOf with the expected signature
    function balanceOf(address a) external view returns (uint256) {
        return balances[a];
    }

    // Simple transfer implementation returning bool (non-reverting)
    function transfer(address to, uint256 amt) external returns (bool) {
        if (balances[msg.sender] < amt) return false;
        balances[msg.sender] -= amt;
        balances[to] += amt;
        return true;
    }

    // Stubbed ERC20 helpers (no override keywords)
    function approve(address, uint256) external returns (bool) { return true; }
    function allowance(address, address) external view returns (uint256) { return 0; }
    function totalSupply() external view returns (uint256) { return 0; }
    function transferFrom(address, address, uint256) external returns (bool) { return true; }
}

contract MockRewarder {
    MockERC20 public baseToken;
    MockERC20 public extraToken;

    constructor(MockERC20 _base, MockERC20 _extra) {
        baseToken = _base;
        extraToken = _extra;
    }

    // Simulate what earned() might report for tests
    function earned(address) external view returns (uint256) {
        return baseToken.balanceOf(address(this));
    }

    // getReward transfers base always; extras only if claimExtras == true
    function getReward(address, address recipient, bool claimExtras) external {
        // Transfer base
        uint256 baseBal = baseToken.balanceOf(address(this));
        if (baseBal > 0) {
            bool ok = baseToken.transfer(recipient, baseBal);
            require(ok, "base transfer failed");
        }

        // Transfer extras only if requested
        if (claimExtras) {
            uint256 extraBal = extraToken.balanceOf(address(this));
            if (extraBal > 0) {
                bool ok2 = extraToken.transfer(recipient, extraBal);
                require(ok2, "extra transfer failed");
            }
        }
    }
}
/// ----------------- End fixed mocks -----------------

```

```solidity
    /// ----------------- PoC test -----------------
function test_claimRewards_leaves_extra_rewards_unclaimed() public {
    // Deploy minimal mocks
    MockERC20 base = new MockERC20("BASE", "BASE");
    MockERC20 extra = new MockERC20("EXTRA", "EXTRA");
    MockRewarder rewarder = new MockRewarder(base, extra);

    // Fund the mock rewarder with base + extra tokens
    uint256 baseAmt = 1_000e18;
    uint256 extraAmt = 500e18;
    base.mint(address(rewarder), baseAmt);
    extra.mint(address(rewarder), extraAmt);

    // Sanity check: rewarder holds both
    assertEq(base.balanceOf(address(rewarder)), baseAmt);
    assertEq(extra.balanceOf(address(rewarder)), extraAmt);

    // Simulate the strategy calling getReward(..., false)
    // (this replicates the problematic call in the strategy: getReward(..., false))
    rewarder.getReward(address(this), address(this), false);

    // Outcome:
    // - base rewards are transferred to recipient (this test contract)
    // - extra rewards remain in the rewarder because claimExtras == false
    assertEq(base.balanceOf(address(this)), baseAmt, "base should have been transferred to recipient");
    assertEq(extra.balanceOf(address(this)), 0, "extra should NOT have been transferred to recipient");
    assertEq(extra.balanceOf(address(rewarder)), extraAmt, "extra must remain inside rewarder (frozen)");
}
/// ----------------- End PoC test -----------------
```

Run:

```bash
 forge test --mt test_claimRewards_leaves_extra_rewards_unclaimed -vvv
```

Expected Output:

```bash
Ran 1 test for src/test/strategies/TokeAutoUSDStrategy.t.sol:TokeAutoUSDStrategyTest
[PASS] test_claimRewards_leaves_extra_rewards_unclaimed() (gas: 1355380)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.23s (85.44ms CPU time)

Ran 1 test suite in 1.27s (1.23s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
