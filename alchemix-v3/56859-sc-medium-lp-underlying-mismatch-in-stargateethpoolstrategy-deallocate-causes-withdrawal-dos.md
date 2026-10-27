# 56859 sc medium lp underlying mismatch in stargateethpoolstrategy deallocate causes withdrawal dos

**Submitted on Oct 21st 2025 at 10:16:27 UTC by @yesofcourse for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56859
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

The Optimism **StargateEthPoolStrategy** treats a caller-supplied **underlying** amount (`WETH`) as if it were **LP shares** during deallocation and also over-reports what it allocates after rounding down deposits.

In practice this causes under-redemption and a hard `require` revert when exiting, freezing vault withdrawals/rebalances that depend on this strategy. When the LP rate is favorable, excess ETH/WETH becomes stranded on the strategy, creating accounting drift (reduced realized returns).

## Vulnerability Details

**A) Unit mismatch on deallocation (LP != underlying).** The strategy sizes the LP redemption as `lpNeeded = amount`, implicitly assuming 1:1 LP→underlying, but later enforces a **WETH** balance check against that same `amount`:

```solidity
// inside _deallocate(uint256 amount)
uint256 lpNeeded = amount;                // !! treats underlying as LP shares
pool.redeem(lpNeeded, address(this));     // returns ETH based on current exchange rate
// ...wrap some ETH to WETH...
require(
    TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount,
    "Strategy balance is less than the amount needed"
);
TokenUtils.safeApprove(address(weth), msg.sender, amount);
return amount;
```

If 1 LP < 1 ETH, redeeming `lpNeeded = amount` returns **less** than `amount` ETH; wrapping only the newly redeemed ETH leaves **WETH < amount** and the `require` reverts (withdrawal DoS). If 1 LP > 1 ETH, the function still approves/returns only `amount`, stranding surplus WETH on the strategy.

**B) Over-reporting on allocation + unwrapped dust.** On deposit, the adapter unwraps WETH to ETH and **rounds down** to a 1e12 multiple, but **returns the original `amount`** to the vault and leaves ETH dust on the contract:

```solidity
// inside _allocate(uint256 amount)
uint256 amountToDeposit = (amount / 1e12) * 1e12; // rounds down
pool.deposit{value: amountToDeposit}(address(this), amountToDeposit);
return amount;                                   // !! over-reports; dust stays as raw ETH
```

During exit, `_deallocate` wraps **only the new ETH redeemed**, ignoring any historical ETH dust that would otherwise bridge the shortfall. The subsequent WETH ≥ `amount` check then fails.

* The strategy API semantically treats `amount` as **underlying** (WETH): it checks WETH balance, approves WETH, and returns `amount`. Sizing the LP redemption as `lpNeeded = amount` is therefore inconsistent and unsafe for share-based LP tokens.
* Returning the unrounded amount in `_allocate` misleads the vault about what was actually put to work and increases the likelihood that `_deallocate(amount)` will fail later.

**Scenario PoC - coded test in next section**

1. Allocate an amount not divisible by 1e12 (e.g., `1e18 - 1`), which forces rounding down and creates ETH dust. `_allocate` returns the **unrounded** amount => vault thinks the full amount was deployed.
2. With LP rate = 1.0 (to isolate the effect), call `_deallocate` for the same `amount`. The adapter wraps only the newly redeemed ETH (ignoring the dust), then reverts on `require(WETH >= amount)`.
3. Changing the rate > 1.0 shows the complementary issue: the call succeeds but leaves **surplus WETH** stranded on the strategy (accounting drift).

## Impact Details

*Temporary freezing of funds for at least 1 hour*.\*\*

* Any vault withdrawal/rebalance that requires deallocation from this strategy can **revert**, temporarily freezing the portion of user funds routed through this adapter until conditions change or the contract is patched.

This is deterministic when the recorded allocation exceeds what can be redeemed or when rounding dust is left unwrapped.

**Secondary operational impacts**

* **Contract fails to deliver promised returns, but doesn’t lose value:** when the exchange rate > 1, over-redemption strands surplus WETH on the strategy, reducing realized yield for depositors and skewing accounting.
* **Cumulative dust growth:** repeated allocations build up raw ETH dust, increasing the chance of future deallocation reverts.

This is not theft; it is a correctness issue that leads to **service disruption** and **value marooned** on the adapter. An adversary (or just organic conditions) can push the system into repeated failures by timing exits under unfavorable LP rates or after many rounded allocations.

## References

* **Affected file:** `src/strategies/optimism/StargateEthPoolStrategy.sol`
  * `_allocate(uint256 amount)` — rounds down deposit but returns the unrounded `amount` (over-reporting). <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol#L52>
  * `_deallocate(uint256 amount)` — sizes redemption as `lpNeeded = amount` (unit mismatch), wraps only newly redeemed ETH, then requires WETH ≥ `amount`. <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol#L60>

## Proof of Concept

## Proof of Concept

In`src/test/strategies/StargateEthPoolStrategy.t.sol`:

1. Add these imports right below the existing two imports at the top:

```solidity
import "forge-std/Test.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC20Burnable} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
```

2. Append the following three blocks at the very end of the file:

* Harness that exposes the internal functions:

```solidity
// Expose _allocate() and _deallocate() for unit testing
contract StargateEthPoolStrategyHarness is StargateEthPoolStrategy {
    constructor(address _myt, StrategyParams memory _params, address _weth, address _pool, address _permit2Address)
        StargateEthPoolStrategy(_myt, _params, _weth, _pool, _permit2Address) {}

    function exposedAllocate(uint256 amount) external returns (uint256) {
        return _allocate(amount);
    }

    function exposedDeallocate(uint256 amount) external returns (uint256) {
        return _deallocate(amount);
    }
}
```

* Tiny local mocks:

```solidity
// ------------------------ Tiny local mocks ------------------------
contract PoCMockWETH is ERC20("Mock WETH", "WETH") {
    receive() external payable {}
    function deposit() external payable { _mint(msg.sender, msg.value); }
    function withdraw(uint256 amt) external {
        _burn(msg.sender, amt);
        (bool ok,) = msg.sender.call{value: amt}("");
        require(ok, "ETH send failed");
    }
}

contract PoCMockLP is ERC20("Stargate LP", "sgLP"), ERC20Burnable {
    function mint(address to, uint256 amt) external { _mint(to, amt); }
}

contract PoCMockStargatePool {
    PoCMockLP public immutable lp;
    uint256 public exchangeRateWad; // ETH per 1 LP, 1e18
    receive() external payable {}
    constructor(address _lp, uint256 _rateWad) { lp = PoCMockLP(_lp); exchangeRateWad = _rateWad; }
    function setRate(uint256 r) external { exchangeRateWad = r; }
    function lpToken() external view returns (address) { return address(lp); }
    function tvl() external view returns (uint256) { return address(this).balance; }
    function deposit(address receiver, uint256) external payable returns (uint256) {
        lp.mint(receiver, msg.value);
        return msg.value;
    }
    function redeem(uint256 lpAmount, address receiver) external returns (uint256) {
        lp.burnFrom(msg.sender, lpAmount);
        uint256 ethOut = (lpAmount * exchangeRateWad) / 1e18;
        (bool ok,) = payable(receiver).call{value: ethOut}("");
        require(ok, "redeem: ETH send failed");
        return ethOut;
    }
    function redeemable(address owner) external view returns (uint256) {
        uint256 bal = lp.balanceOf(owner);
        return (bal * exchangeRateWad) / 1e18;
    }
}
```

* The PoC test:

```solidity
// ------------------------ Dust / Over-reporting PoC (rate = 1.0) ------------------------
contract StargateEthPoolStrategy_PoC_Dust is Test {
    PoCMockWETH weth;
    PoCMockLP lp;
    PoCMockStargatePool pool;
    StargateEthPoolStrategyHarness strat;

    function test_PoC_OverReportAndDust_DoS_Rate1_0() public {
        vm.deal(address(this), 10_000 ether);
        weth = new PoCMockWETH();
        lp   = new PoCMockLP();
        pool = new PoCMockStargatePool(address(lp), 1e18); // EXACTLY 1.0 ETH per LP

        (bool okFund,) = address(pool).call{value: 5_000 ether}("");
        require(okFund, "fund pool");

        IMYTStrategy.StrategyParams memory p = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "StargateEthPool",
            protocol: "StargateEthPool",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: type(uint256).max,
            globalCap: type(uint256).max,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 0
        });

        strat = new StargateEthPoolStrategyHarness(
            address(0xdead), p, address(weth), address(pool), address(0xbeef)
        );

        uint256 amount = 1e18 - 1; // 1 WETH - 1 wei
        weth.deposit{value: amount}();
        weth.transfer(address(strat), amount);

        uint256 ret = strat.exposedAllocate(amount);
        assertEq(ret, amount, "over-reported: returned full amount, not rounded-down deposit");

        // sanity: redeemable LP + ETH dust equals the request, but only the delta gets wrapped
        assertEq(pool.redeemable(address(strat)) + address(strat).balance, amount);

        vm.expectRevert(bytes("Strategy balance is less than the amount needed"));
        strat.exposedDeallocate(amount);
    }
}
```

3. Run with:

```bash
forge test --match-test test_PoC_OverReportAndDust_DoS_Rate1_0
```
