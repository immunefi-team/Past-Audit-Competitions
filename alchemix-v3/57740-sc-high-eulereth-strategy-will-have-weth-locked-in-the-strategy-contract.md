# 57740 sc high eulereth strategy will have weth locked in the strategy contract

**Submitted on Oct 28th 2025 at 15:28:01 UTC by @oxrex for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57740
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/EulerWETHStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

In the EulerWETHStrategy contract, we can allocate and deallocate assets (WETH). When we allocate, we take WETH from the VaultV2 and deposits it into the WETH Yield vault on Euler, when we deallocate, we withdraw the supplied WETH and send it back to the VaultV2. However, there is an issue whereby the WETH will be stuck in the Strategy contract and not go to the VaultV2.

## Vulnerability Details

To explain the issue, I will be grabbing relevant code snippets from the VaultV2, EulerWETHStrategy, MYTStrategy, and AlchemistAllocator contracts:

```solidity
FILE: VaultV2

function allocate(address adapter, bytes memory data, uint256 assets) external {
        require(isAllocator[msg.sender], ErrorsLib.Unauthorized());
        allocateInternal(adapter, data, assets);
    }

    function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

        accrueInterest();

        SafeERC20Lib.safeTransfer(asset, adapter, assets);

        (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);

        for (uint256 i; i < ids.length; i++) {
            Caps storage _caps = caps[ids[i]];
            _caps.allocation = (int256(_caps.allocation) + change).toUint256();

            require(_caps.absoluteCap > 0, ErrorsLib.ZeroAbsoluteCap());

            require(_caps.allocation <= _caps.absoluteCap, ErrorsLib.AbsoluteCapExceeded());

            require(
                _caps.relativeCap == WAD || _caps.allocation <= firstTotalAssets.mulDivDown(_caps.relativeCap, WAD),
                ErrorsLib.RelativeCapExceeded()
            );
        }
        emit EventsLib.Allocate(msg.sender, adapter, assets, ids, change);
    }

    function deallocate(address adapter, bytes memory data, uint256 assets) external {
        require(isAllocator[msg.sender] || isSentinel[msg.sender], ErrorsLib.Unauthorized());
        deallocateInternal(adapter, data, assets);
    }

    function deallocateInternal(address adapter, bytes memory data, uint256 assets)
        internal
        returns (bytes32[] memory)
    {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

        (bytes32[] memory ids, int256 change) = IAdapter(adapter).deallocate(data, assets, msg.sig, msg.sender);

        for (uint256 i; i < ids.length; i++) {
            Caps storage _caps = caps[ids[i]]; // e.g 500
            require(_caps.allocation > 0, ErrorsLib.ZeroAllocation());
            _caps.allocation = (int256(_caps.allocation) + change).toUint256(); // 0
        }

        SafeERC20Lib.safeTransferFrom(asset, adapter, address(this), assets);
        emit EventsLib.Deallocate(msg.sender, adapter, assets, ids, change);
        return ids;
    }
```

```solidity
FILE: EulerWETHStrategy
function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(weth), address(vault), amount);
        vault.deposit(amount, address(this));
        return amount;
    }

function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        vault.withdraw(amount, address(this), address(this));
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        return amount;
    }
```

```solidity
FILE: MYTStrategy

function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
        if (killSwitch) {
            return (ids(), int256(0));
        }
        require(assets > 0, "Zero amount");
        uint256 oldAllocation = abi.decode(data, (uint256));
        uint256 amountAllocated = _allocate(assets);
        uint256 newAllocation = oldAllocation + amountAllocated;
        emit Allocate(amountAllocated, address(this));
        return (ids(), int256(newAllocation) - int256(oldAllocation));
    }

    /// @notice See Morpho V2 vault spec
    function deallocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
        if (killSwitch) {
            return (ids(), int256(0));
        }
        require(assets > 0, "Zero amount");
        uint256 oldAllocation = abi.decode(data, (uint256));
        uint256 amountDeallocated = _deallocate(assets);
        uint256 newAllocation = oldAllocation - amountDeallocated;
        emit Deallocate(amountDeallocated, address(this));

        return (ids(), int256(newAllocation) - int256(oldAllocation));
    }
```

```solidity
FILE: AlchemistAllocator

function allocate(address adapter, uint256 amount) external {
        require(msg.sender == admin || operators[msg.sender], "PD");
        bytes32 id = IMYTStrategy(adapter).adapterId();
        uint256 absoluteCap = vault.absoluteCap(id);
        uint256 relativeCap = vault.relativeCap(id);
        // FIXME get this from the StrategyClassificationProxy for the respective risk class
        uint256 daoTarget = type(uint256).max;
        uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
            // caller is operator
            adjusted = adjusted > daoTarget ? adjusted : daoTarget;
        }
        // pass the old allocation to the adapter
        bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.allocate(adapter, oldAllocation, amount);
    }

    function deallocate(address adapter, uint256 amount) external {
        require(msg.sender == admin || operators[msg.sender], "PD");
        bytes32 id = IMYTStrategy(adapter).adapterId();
        uint256 absoluteCap = vault.absoluteCap(id);
        uint256 relativeCap = vault.relativeCap(id);
        // FIXME get this from the StrategyClassificationProxy for the respective risk class
        uint256 daoTarget = type(uint256).max;
        uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
            // caller is operator
            adjusted = adjusted < daoTarget ? adjusted : daoTarget;
        }
        // pass the old allocation to the adapter
        bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.deallocate(adapter, oldAllocation, amount);
    }
```

Operators as well as the AlchemistAllocator admin contract allocate user deposited assets from the VaultV2 contract into third party contracts such as Euler Yield vaults. In this case, the strategies (EulerWETHStrategy) is the middle man contract. This contract is never meant to hold onto any asset, it only passes it around. However, during the period when the kill switch is on, the contract will wrongly hold assets it shouldn't have held in the first place. Now, even when the kill switch is then turned off (set to `false`), these assets it held earlier cannot be recovered by the VaultV2 contract.

Consider the scenario below:

1. Bob deposits 200 WETH into the VaultV2 contract
2. Operator then allocates 100 WETH to the EulerWETHStrategy contract. This strategy contract deposits that 100 WETH into Euler WETH vault and receives e.g 100 shares
3. Then the kill switch gets set to true
4. While trying to allocate new funds to EulerWETHStrategy, the EulerWETHStrategy returns early since we can't supply assets into Euler at this time (because switch is on)

```solidity
function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
@>        if (killSwitch) {
@>            return (ids(), int256(0));
        }
    }
```

5. However, the VaultV2 contract has already sent 100 more WETH into the `EulerWETHStrategy` contract and the `change` returned is 0, which means we do not upscale the allocation in this case and the allocation of this strategy will remain 100 WETH whereas it has taken 200 WETH from the VaultV2:

```solidity
function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
       ...

@>        SafeERC20Lib.safeTransfer(asset, adapter, assets);

        (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);

        for (uint256 i; i < ids.length; i++) {
            Caps storage _caps = caps[ids[i]];
            _caps.allocation = (int256(_caps.allocation) + change).toUint256();

            require(_caps.absoluteCap > 0, ErrorsLib.ZeroAbsoluteCap());

            require(_caps.allocation <= _caps.absoluteCap, ErrorsLib.AbsoluteCapExceeded());

            require(
                _caps.relativeCap == WAD || _caps.allocation <= firstTotalAssets.mulDivDown(_caps.relativeCap, WAD),
                ErrorsLib.RelativeCapExceeded()
            );
        }
        emit EventsLib.Allocate(msg.sender, adapter, assets, ids, change);
    }
```

This is problematic because now, we have a state whereby:

* 100 WETH is in Euler WETH vault on Mainnet
* 100 WETH is inside the `EulerWETHStrategy` (which it shouldn't be there in the first place and should have been sent back to the VaultV2 contract when the change is 0 since kill switch is `true`)
* Now, regardless of the kill switch staying on (`true`) or us, setting it to off (`false`), these 100 WETH sitting inside the contract cannot be recovered.
* When we deallocate, only the first 100 WETH we supplied to Euler Yield vault will be withdrawn and sent to the VaultV2. The other 100 WETH will be locked.

## Impact Details

Assets being supplied to the 3rd party protocol, which in this case is the Euler WETH yield vault on Ethereum mainnet, will be stuck inside the `EulerWETHStrategy` strategy contract and cannot be recovered since we currently expose no recovery functions in the MYTStrategy or EulerWETHStrategy contracts.

Since the assets are locked in the strategy, I have decided to log this as critical but it can also be argued to be high severity.

When the kill switch is on and the change is 0 but the `assets` is non-zero, the `EulerWETHStrategy::allocate()` function should send back the assets it received.

For example, add that implementation in here before the return statement:

```solidity
function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
        if (killSwitch) {
            // @audit handle the return of `assets` here
            return (ids(), int256(0));
        }
        ...
    }
```

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol?utm\\_source=immunefi#L102-L116>

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/EulerWETHStrategy.sol?utm\\_source=immunefi>

## Proof of Concept

## Proof of Concept

Paste the POC inside the AlchemistAllocator.t.sol test file and run with verbosity of 3 (vvv)

```solidity
function testPOCAllocate() public {
        require(vault.adaptersLength() == 1, "adaptersLength is must be 1");

        // 1. USER DEPOSIT TO VAULT V2
        _magicDepositToVault(address(vault), user1, 200 ether);
        vm.startPrank(admin);

        // 2. WE ALLOCATE SOME OF THOSE DEPOSITS TO THIS STRAT
        allocator.allocate(address(mytStrategy), 100 ether);

        // 3. 3RD PARTY VAULT HAS ISSUE, WE SET KILL SWITCH
        mytStrategy.setKillSwitch(true);

        // 4. THIS NEW ALLOCATION IS MEANT TO NOT ALLOCATE TOKENS TO 3RD PARTY VAULT WHICH IT HANDLES INCORRECTLY BY TRANSFERRING ASSETS INTO THE STRATEGY THAT ARE NOT STUCK
        allocator.allocate(address(mytStrategy), 100 ether);

        // 5. WE CHECK BALANCES AND STATE UPDATES
        uint256 mytStrategyYieldTokenRealAssets = mytStrategy.realAssets();
        console.log("Strategy WETH balance in Euler: ", mytStrategyYieldTokenRealAssets);
        uint256 yieldTokenBalanceOfStrat = IMockYieldToken(mockStrategyYieldToken).balanceOf(address(mytStrategy));
        console.log("Yield token balance of Strategy: ", yieldTokenBalanceOfStrat);
        uint256 assetBalanceOfStrat = IERC20(mockVaultCollateral).balanceOf(address(mytStrategy));
        console.log("WETH balance of Strat: ", assetBalanceOfStrat);

        // 6. DEALLOCATIONS
        mytStrategy.setKillSwitch(false);
        // Try to de-allocate all 200 assets:
        // 1. 100 assets is deposited in 3rd party vault
        // 2. 100 assets is inside contract
        allocator.deallocate(address(mytStrategy), 100 ether);
        vm.expectRevert();
        allocator.deallocate(address(mytStrategy), 100 ether);
        vm.stopPrank();
    }
```
