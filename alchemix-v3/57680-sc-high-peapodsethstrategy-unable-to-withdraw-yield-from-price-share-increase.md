# 57680 sc high peapodsethstrategy unable to withdraw yield from price share increase

**Submitted on Oct 28th 2025 at 06:24:18 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57680
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/PeapodsETH.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Protocol insolvency

## Description

## Brief/Intro

`PeapodsETHStrategy` is utilizing peapods vault to generate yield. while user can deposit WETH into the VaultV2 to receive MYT, on the background the WETH deposited would be utilized in peapods strategy. but the contract can only allocate WETH amount and deallocate same amount. this amount is outside the yield of peapods vault generated, making the Strategy effectively cant receive the benefit of price per share increase.

## Vulnerability Details

the MYT VaultV2 would call the `PeapodsETHStrategy` at `allocate` and `deallocate`. it is using mint at `allocate` and withdraw at `deallocate`.

because of this, the VaultV2 would use the `amount` to account how much the strategy allocate into given peapods vault. note that the `amount` is in WETH amount.

the issue lies on how `deallocate` flow, it can only deallocate previous `amount` that is getting allocated:

```solidity
    function deallocateInternal(address adapter, bytes memory data, uint256 assets)
        internal
        returns (bytes32[] memory)
    {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

@>      (bytes32[] memory ids, int256 change) = IAdapter(adapter).deallocate(data, assets, msg.sig, msg.sender);

        for (uint256 i; i < ids.length; i++) {
            Caps storage _caps = caps[ids[i]];
            require(_caps.allocation > 0, ErrorsLib.ZeroAllocation());
@>          _caps.allocation = (int256(_caps.allocation) + change).toUint256();
        }
```

effectively this means that `allocate` and `deallocate` can only process raw WETH amount that is returned by the `PeapodsETHStrategy` function.

for example if the MYT VaultV2 is calling `PeapodsETHStrategy` to allocate 10 WETH, the `_caps.allocation` would be set into 10 WETH. lets assume we got 10 shares.

then after some time, yield on peapods is generated and 10 shares worth 11 WETH, but when `deallocate` is called, it can only withdraw 10 WETH because the `change` would return -10 WETH and this would get deducted with previous `_caps.allocation` . effectively leaving unburned shares from the `withdraw` inside `PeapodsETHStrategy` contract with no way to retrieve.9.

## Impact Details

it is crucial that Strategy can generate yield for the AlchemistV3 and Transmuter contract to operate normally. so this would break the self repaying loan that the protocol wants to do.

## References

<https://github.com/morpho-org/vault-v2/blob/406546763343b9ffa84c2f63742ae55a490b7c42/src/VaultV2.sol#L598-L615>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/PeapodsETH.sol#L29-L47>

## Proof of Concept

## Proof of Concept

add this diff:

```diff
diff --git a/src/test/strategies/PeapodsETHStrategy.t.sol b/src/test/strategies/PeapodsETHStrategy.t.sol
index b63da63..d77a795 100644
--- a/src/test/strategies/PeapodsETHStrategy.t.sol
+++ b/src/test/strategies/PeapodsETHStrategy.t.sol
@@ -10,6 +10,16 @@ contract MockPeapodsETHStrategy is PeapodsETHStrategy {
     {}
 }
 
+interface IERC20 {
+    function transfer(address to, uint256 amount) external returns(bool);
+    function balanceOf(address user) external returns(uint256);
+    function totalSupply() external view returns (uint256);
+}
+
+interface IERC4626 {
+    function convertToAssets(uint256 asset) external returns(uint256);
+}
+
 contract PeapodsETHStrategyTest is BaseStrategyTest {
     address public constant PEAPODS_ETH_VAULT = 0x9a42e1bEA03154c758BeC4866ec5AD214D4F2191;
     address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
@@ -45,6 +55,48 @@ contract PeapodsETHStrategyTest is BaseStrategyTest {
         return vm.envString("MAINNET_RPC_URL");
     }
 
+    function test_peapodETHYieldStuckInFormOfShares() public {
+        uint256 amountToAllocate = 10e18;
+        uint256 amountToDeallocate = amountToAllocate;
+        vm.startPrank(vault);
+        deal(testConfig.vaultAsset, strategy, amountToAllocate);
+        bytes memory prevAllocationAmount = abi.encode(0);
+        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
+        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
+        require(initialRealAssets > 0, "Initial real assets is 0");
+        uint256 initialShares = IERC20(PEAPODS_ETH_VAULT).balanceOf(address(strategy));
+        console.log("initial shares at allocate: %18e", initialShares);
+
+        // simulate yield by mocking totalSupply of peapods weth vault
+        bytes32 TOTAL_SUPPLY_SLOT = bytes32(uint256(2)); // we can get this by `cast storage vault_address` with configured etherscan api
+        uint256 totalSupply = IERC20(PEAPODS_ETH_VAULT).totalSupply();
+        bytes32 value = bytes32(totalSupply * 90 / 100); // we reduce total supply to mock increase price per share by 10% to show yield
+        vm.store(address(PEAPODS_ETH_VAULT), TOTAL_SUPPLY_SLOT, value);
+        assertEq(totalSupply * 90 / 100, IERC20(PEAPODS_ETH_VAULT).totalSupply());
+
+        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
+
+        // lets try to withdraw the realAssets which is total WETH converted from all shares
+        uint256 realAsset = IMYTStrategy(strategy).realAssets();
+        vm.expectRevert();
+        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, realAsset, "", address(vault));
+
+        // now we withdraw the same amount as previous allocate. this would not revert
+        (bytes32[] memory strategyIds, int256 change) = IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
+        assertApproxEqAbs(change, -int256(amountToDeallocate), 1 * 10 ** testConfig.decimals);
+        assertGt(strategyIds.length, 0, "strategyIds is empty");
+        assertEq(strategyIds[0], IMYTStrategy(strategy).adapterId(), "adapter id not in strategyIds");
+        uint256 finalRealAssets = IMYTStrategy(strategy).realAssets();
+        require(finalRealAssets < initialRealAssets, "Final real assets is not less than initial real assets");
+
+        // we get how many shares unburnt
+        uint256 leftOverShares = IERC20(PEAPODS_ETH_VAULT).balanceOf(address(strategy));
+        console.log("leftover shares: %18e", leftOverShares);
+        // which equal to
+        uint256 leftOverWETH = IERC4626(PEAPODS_ETH_VAULT).convertToAssets(leftOverShares);
+        console.log("leftover shares equal to: %18e WETH", leftOverWETH);
+    }
+
     // Add any strategy-specific tests here
     function test_strategy_full_deallocate_does_not_revert_due_to_rounding(uint256 amountToAllocate, uint256 amountToDeallocate) public {
         amountToAllocate = bound(amountToAllocate, 1 * 10 ** testConfig.decimals, testConfig.vaultInitialDeposit);

```

the test result:

```bash
[PASS] test_peapodETHYieldStuckInFormOfShares() (gas: 1430790)
Logs:
  initial shares at allocate: 9.73339568419290479
  leftover shares: 0.973339568419290479
  leftover shares equal to: 1.111111111111111111 WETH

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 36.27s (32.31s CPU time)

Ran 1 test suite in 36.28s (36.27s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

shows that initial 10 WETH equal to 9.73 shares. then after some time 10 WETH now equal to \~8.75 shares but the strategy cant deallocate more than the original 10 WETH. the logs shows that withdrawing 10 WETH would leave 0.97 shares in the Strategy which equal to \~1.1 WETH.

also shown on PoC if the strategy try to withdraw the realAsset (all shares owned converted to asset), it would revert of underflow.
