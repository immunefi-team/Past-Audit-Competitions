# 58358 sc high mismatched collateralweight and rawlocked causes incorrect collateral removal in sync

**Submitted on Nov 1st 2025 at 15:31:29 UTC by @JoeMama for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58358
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When the minimum collateralization is updated, the new ratio does not automatically rescale `_totalLocked`.

If a `redeem` happens afterwards, `_collateralWeight` is calculated using the `_totalLocked,` that reflects the previous minimum collateralization ratio. While the `account.rawLocked` is updated to the new minimum collateralization scale during `_sync`.

Because `_collateralWeight` and `rawLocked` are now on different scales, collateral decay is miscalculated, causing users to lose more or less collateral than intended depending on a increase or decrease with the new `minimumCollateralization`.

## Vulnerability Details

During `_addDebt`:

```
    function _addDebt(uint256 tokenId, uint256 amount) internal {
        Account storage account = _accounts[tokenId];
        uint256 toLock = convertDebtTokensToYield(amount) * 
        -- Snip --
        _totalLocked += toLock;

    }
```

`_totalLocked` is using the current `minimumCollateralization` scale

During `redeem`:

```
uint256 old = _totalLocked;
_totalLocked = totalOut > old ? 0 : old - totalOut;

_collateralWeight += PositionDecay.WeightIncrement(
    totalOut > old ? old : totalOut,
    old
);
```

`_collateralWeight` is computed using the old `_totalLocked`, which does not account for any increase in `minimumCollateralization`

During `_sync`:

```
collateralToRemove = PositionDecay.ScaleByWeightDelta(
    account.rawLocked,
    _collateralWeight - account.lastCollateralWeight
);
```

`account.rawLocked` now reflects the new `minimumCollateralization`, but `_collateralWeight` is still based on the old `_totalLocked`.

This mismatch causes decay to be too aggressive, users lose more collateral than intended because the collateral weight did not scale with the updated `account.rawLocked`.

**What is needed?**

The token needs to be synced after a new `minimumCollateralization` but before a redeem, because rawLocked is then updated. This can be done by `poke`, `mint`, `deposit`, or other actions that trigger `_sync`.

## Impact Details

Users can lose too much or too little collateral during decay because `_collateralWeight` is calculated from the old `_totalLocked` while `account.rawLocked` uses the new `minimumCollateralization` scale.

## Link to Proof of Concept

<https://gist.github.com/hexens-joe/c8afa5451cd5aa12db21ed3c6b517106>

## Proof of Concept

## Proof of Concept

In the PoC there are 3 Scenario's:

Scenario 1 demonstrates the bug: too much collateral is removed.

Scenarios 2 and 3 behave correctly, removing the expected amount of collateral. They are included to show that the bug occurs when `minimumCollateralization` is updated after debts have already been created. Not by the `minimumCollateralization` itself.

In all three scenarios, the same debt (1e18) and collateral (1e18) should be removed. In Scenario 1, however, 1.2e18 collateral is removed instead

Please run the gist with `forge test --mt testDecayTooMuchCollateral -vvv`

```
contract TestPoc is AlchemistV3Test {
    function testDecayTooMuchCollateral() external {

        vm.prank(alOwner);
        alchemist.setMinimumCollateralization(1_052_631_578_950_000_000);

        console.log("Scenario 1 - After deposit set new setMinimumCollateralization to 1_252_631_578_950_000_000");

        uint256 snapshotId = vm.snapshot();
      
        address user1 = makeAddr("user1");
    
        deal(address(vault), address(user1), 1.5e18); // starts with 1.2e18 underlying

        vm.startPrank(address(user1));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(1.46e18, address(user1), 0);

        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(user1), address(alchemistNFT));
       
        alchemist.mint(tokenId, 1e18, user1);

        vm.roll(vm.getBlockNumber() + 1);

        IERC20(alToken).approve(address(transmuterLogic), 3000e18);
        IERC20(address(vault)).approve(address(alchemist), 100_000e18);
        transmuterLogic.createRedemption(1e18);
        vm.stopPrank();

        vm.prank(alOwner);
        //  from  alchemist.setMinimumCollateralization(1_052_631_578_950_000_000);
        alchemist.setMinimumCollateralization(1_252_631_578_950_000_000);
        vm.roll(vm.getBlockNumber() + 1);

        alchemist.poke(tokenId);

        vm.roll(vm.getBlockNumber() + transmuterLogic.timeToTransmute()); // full dulration of the redemption.

        vm.startPrank(user1);
        IERC20(alToken).approve(address(transmuterLogic), 3000e18);
        IERC20(address(vault)).approve(address(alchemist), 100_000e18);
        console.log("");
        (uint256 a, uint256 b, uint256 c) = alchemist.getCDP(tokenId);
        console.log("before claim collateral:", a);
        console.log("before claim debt:", b);
        console.log("before claim earmarked:", c);

        vm.roll(vm.getBlockNumber() + 1);

        vm.startPrank(user1);
        transmuterLogic.claimRedemption(1);

        vm.roll(vm.getBlockNumber() + 1);

        // just incase to have the lasest versions.
        alchemist.poke(tokenId);

        (uint256 a3, uint256 b3, uint256 c3) = alchemist.getCDP(tokenId);

        console.log("");
        console.log("scenario1 collateral:", a3 , " this is Wrong <========== ( it should be 460000000000000000 )=========");
        console.log("scenario1 debt:", b3);
        console.log("scenario1 earmarked:", c3 );
        console.log("");

        vm.stopPrank();
        vm.revertTo(snapshotId);

        // scenario2 : new setMinimumCollateralization before deposit.
        console.log(
            "Scenario 2 - No bug here -> Before a depost call setMinimumCollateralization to set it to 1_252_631_578_950_000_000."
        );

        vm.prank(alOwner);
        //  from  alchemist.setMinimumCollateralization(1_052_631_578_950_000_000);
        alchemist.setMinimumCollateralization(1_252_631_578_950_000_000);

        deal(address(vault), address(user1), 1.5e18); // starts with 1.2e18 underlying

        vm.startPrank(address(user1));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(1.46e18, address(user1), 0);
        alchemist.mint(tokenId, 1.0e18, user1);

        vm.roll(vm.getBlockNumber() + 1);

        IERC20(alToken).approve(address(transmuterLogic), 3000e18);
        IERC20(address(vault)).approve(address(alchemist), 100_000e18);
        transmuterLogic.createRedemption(1e18);
        vm.stopPrank();

        vm.roll(vm.getBlockNumber() + transmuterLogic.timeToTransmute()); // full dulration of the redemption.

        vm.startPrank(user1);
        IERC20(alToken).approve(address(transmuterLogic), 3000e18);
        IERC20(address(vault)).approve(address(alchemist), 100_000e18);

        (a, b, c) = alchemist.getCDP(tokenId);
        console.log("");
        console.log("before claim collateral:", a);
        console.log("before claim debt:", b);
        console.log("before claim earmarked:", c);

        vm.roll(vm.getBlockNumber() + 1);

        vm.startPrank(user1);
        transmuterLogic.claimRedemption(1);

        vm.roll(vm.getBlockNumber() + 1);

        // just incase to have the lasest versions.
        alchemist.poke(tokenId);

        (a3, b3, c3) = alchemist.getCDP(tokenId);

        console.log("");
        console.log("scenario2 collateral:", a3 , " this is correct");
        console.log("scenario2 debt:", b3 , " this is correct");
        console.log("scenario2 earmarked:", c3 , " this is correct");
        console.log("");

        vm.stopPrank();
        vm.revertTo(snapshotId);

        // scenario 3.

        console.log("Scenario 3 - No Bug here -> No calls to setMinimumCollateralization ( keep 1_052_631_578_950_000_000 )");

        deal(address(vault), address(user1), 1.5e18); 
       
        vm.startPrank(address(user1));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(1.46e18, address(user1), 0);
        alchemist.mint(tokenId, 1e18, user1);

        vm.roll(vm.getBlockNumber() + 1);

        IERC20(alToken).approve(address(transmuterLogic), 3000e18);
        IERC20(address(vault)).approve(address(alchemist), 100_000e18);
        transmuterLogic.createRedemption(1e18);
        vm.stopPrank();

        alchemist.poke(tokenId);

        vm.roll(vm.getBlockNumber() + transmuterLogic.timeToTransmute()); // full dulration of the redemption.

        vm.startPrank(user1);
        IERC20(alToken).approve(address(transmuterLogic), 3000e18);
        IERC20(address(vault)).approve(address(alchemist), 100_000e18);
        console.log("");
        (a, b, c) = alchemist.getCDP(tokenId);

        console.log("before claim collateral:", a);
        console.log("before claim:", b);
        console.log("before claim earmarked:", c);

        vm.startPrank(user1);
        transmuterLogic.claimRedemption(1);

        vm.roll(vm.getBlockNumber() + 1);

        // just incase to have the lasest versions.
        alchemist.poke(tokenId);

        (a3, b3, c3) = alchemist.getCDP(tokenId);

        console.log("");
        console.log("scenario3 collateral:", a3 , " this is correct");
        console.log("scenario3 debt:", b3 , " this is correct");
        console.log("scenario3 earmarked:", c3 , " this is correct");
        console.log("");
    }
}
```

output:

```
Logs:
  Scenario 1 - After deposit set new setMinimumCollateralization to 1_252_631_578_950_000_000
  
  before claim collateral: 1460000000000000000
  before claim debt: 999999999999999999
  before claim earmarked: 999999999999999999
  
  scenario1 collateral: 270000000000475001  this is Wrong <========== ( it should be 460000000000000000 )=========
  scenario1 debt: 0
  scenario1 earmarked: 0
  
  Scenario 2 - No bug here -> Before a depost call setMinimumCollateralization to set it to 1_252_631_578_950_000_000.
  
  before claim collateral: 1460000000000000000
  before claim debt: 1000000000000000000
  before claim earmarked: 1000000000000000000
  
  scenario2 collateral: 459999999999999999  this is correct
  scenario2 debt: 0  this is correct
  scenario2 earmarked: 0  this is correct
  
  Scenario 3 - No Bug here -> No calls to setMinimumCollateralization ( keep 1_052_631_578_950_000_000 )
  
  before claim collateral: 1460000000000000000
  before claim: 1000000000000000000
  before claim earmarked: 1000000000000000000
  
  scenario3 collateral: 460000000000000000  this is correct
  scenario3 debt: 0  this is correct
  scenario3 earmarked: 0  this is correct
```
