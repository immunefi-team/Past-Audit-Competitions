# 58093 sc medium morpho reward in morphoyearnogweth will be lost or stuck

**Submitted on Oct 30th 2025 at 15:33:32 UTC by @oxrex for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58093
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

For the `MorphoYearnOGWETH` vault, MORPHO portion of the earned yield will be lost since some percentage of the total APY is paid out in `MOR` tokens.

## Vulnerability Details

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {MYTStrategy} from "../../MYTStrategy.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {TokenUtils} from "../../libraries/TokenUtils.sol";

...

/**
 * @title MorphoYearnOGWETHStrategy
 * @notice This strategy is used to allocate and deallocate weth to the Morpho Yearn OG WETH vault on Mainnet
 */
contract MorphoYearnOGWETHStrategy is MYTStrategy {
    WETH public immutable weth;
    IERC4626 public immutable vault;

    event MorphoYearnOGWETHStrategyDebugLog(string message, uint256 value);

    constructor(address _myt, StrategyParams memory _params, address _vault, address _weth, address _permit2Address)
        MYTStrategy(_myt, _params, _permit2Address, _weth)
    {
        weth = WETH(_weth);
        vault = IERC4626(_vault);
        require(vault.asset() == _weth, "Vault asset != WETH");
    }

    // @note seen
    function _allocate(uint256 amount) internal override returns (uint256) {
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
        TokenUtils.safeApprove(address(weth), address(vault), amount);
        vault.deposit(amount, address(this));
        return amount;
    }

    function _deallocate(uint256 amount) internal override returns (uint256) {
        vault.withdraw(amount, address(this), address(this));
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(wethRedeemed + wethBalanceBefore >= amount, "Strategy balance is less than the amount needed");
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }

    ...
}

```

The `MorphoYearnOGWETH` contract we interact with on Ethereum mainnet has an APY of 3.97% per year (3.8% Native APY, 0.17% MORPHO APY). In that, there is a 0.19% performance fee charged to users. Now, the net APY wil be 3.79% i.e Morpho will take 0.19% from the Native Yield of 3.8% which means the Native APY will be 3.61% and then Morpho APY is 0.17% which results in the total APY of \~3.79%

Now, Morpho suppliers in this Yearn OG WETH `0xE89371eAaAC6D46d4C3ED23453241987916224FC` vault will get two rewards.

1. Reward 1 will be distributed in WETH tokens
2. Reward 2 will be distributed in MORPHO tokens

You can check the breakdown here at the Vault page in the Morpho App: <https://app.morpho.org/ethereum/vault/0xE89371eAaAC6D46d4C3ED23453241987916224FC/yearn-og-weth>

The problem is that in the `MorphoYearnOGWETHStrategy` Alchemix is deploying, there is nowhere they setup:

* The MORPHO URD CONTRACT which resides here onchain for MORPHO tokens: 0x330eefa8a787552DC5cAd3C3cA644844B1E61Ddb

Thus, MORPHO URD contract's `claim()` function will not be possible to be called from the contract. That is only one part of the issue.

The second part of the issue is that, these MORPHO tokens are sent to the `account` which in this case will be the `MorphoYearnOGWETHStrategy` address as that will be the address computed in the merkle reward root for the URD contract by Morpho protocol.

As a result of this, even if the Alchemix team were to obtain a valid proof and then call the URD's `claim()` function from another address (which is possible by the way to claim on behalf of someone), these tokens are in fact sent to the account which in this case is the `MorphoYearnOGWETHStrategy` address.

Thus, they will be locked inside the contract and cannot be claimed.

You can check the claim function of the URD here: <https://etherscan.io/address/0x330eefa8a787552dc5cad3c3ca644844b1e61ddb#code#F1#L136>

## Impact Details

A part of the yield earned (MORPHO) will be stuck inside the strategy contract and cannot be retrived. From my forked test case of allocating 1k WETH to the Morpho Yearn OG vault on Morpho, the relative amount stuck will be 0.17% of the total yield which amounts to 1.7 WETH and is 3417 MORPHO tokens.

At current MORPHO prices, that is about USD 6800.

In the `MorphoYearnOGWETHStrategy`, we should:

1. Add a new variable to keep track of the URD
2. Add a new function to claim and also to send the received MORPHO tokens out.

We can do something similar to:

```diff
contract MorphoYearnOGWETHStrategy is MYTStrategy {
    WETH public immutable weth;
    IERC4626 public immutable vault;
+   address public MORPHO_URD;
```

Then, set the address and implement a function send out the MORPHO tokens.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {MorphoYearnOGWETHStrategy} from "../../strategies/mainnet/MorphoYearnOGWETH.sol";
import {IAllocator} from "../../interfaces/IAllocator.sol";
import {MerkleProof} from "../../../lib/openzeppelin-contracts/contracts/utils/cryptography/MerkleProof.sol";

contract MockMorphoYearnOGWETHStrategy is MorphoYearnOGWETHStrategy {
    constructor(address _myt, StrategyParams memory _params, address _vault, address _weth, address _permit2Address)
        MorphoYearnOGWETHStrategy(_myt, _params, _vault, _weth, _permit2Address)
    {}
}

interface Token {
    function balanceOf(address user) external view returns (uint256);
}

interface IUniversalRewardsDistributorBase {
    function root() external view returns (bytes32);
    function owner() external view returns (address);
    function timelock() external view returns (uint256);
    function ipfsHash() external view returns (bytes32);
    function isUpdater(address) external view returns (bool);
    function claimed(address, address) external view returns (uint256);

    function acceptRoot() external;
    function setRoot(bytes32 newRoot, bytes32 newIpfsHash) external;
    function setTimelock(uint256 newTimelock) external;
    function setRootUpdater(address updater, bool active) external;
    function revokePendingRoot() external;
    function setOwner(address newOwner) external;

    function submitRoot(bytes32 newRoot, bytes32 ipfsHash) external;

    function claim(address account, address reward, uint256 claimable, bytes32[] memory proof)
        external
        returns (uint256 amount);
}

contract MorphoYearnOGWETHStrategyTest is BaseStrategyTest {
    address public constant MORPHO_YEARN_OG_VAULT = 0xE89371eAaAC6D46d4C3ED23453241987916224FC;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant MORPHO_URD = 0x330eefa8a787552DC5cAd3C3cA644844B1E61Ddb;
    address public constant MORPHO_TOKEN = 0x58D97B57BB95320F9a05dC918Aef65434969c2B2;
    address public constant URD_OWNER = 0xF057afeEc22E220f47AD4220871364e9E828b2e9;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "MorphoYearnOGETH",
            protocol: "MorphoYearnOGETH",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 1000e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 23_298_447;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockMorphoYearnOGWETHStrategy(vault, params, MORPHO_YEARN_OG_VAULT, WETH, MAINNET_PERMIT2));
    }

    function test_strategy_deallocate_reverts_due_to_slippage(uint256 amountToAllocate, uint256 amountToDeallocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        amountToDeallocate = amountToAllocate;
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        vm.expectRevert();
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }


    function _hashThePair(bytes32 a, bytes32 b) internal pure returns (bytes32) {
        return a < b
            ? keccak256(bytes.concat(a, b))
            : keccak256(bytes.concat(b, a));
    }

    event Root(bytes32);

    function test_stuckMORPHOTokensInContract() public {
        uint256 amountToAllocate = 1000e18;
        bytes32 ID = IMYTStrategy(strategy).adapterId();

        console.log("CURRENT BLOCK TIME AND NUMBER DURING ALLOC AND DEPOSIT INTO MORPHO YEARN OG");
        console.log("Block timestamp: ", block.timestamp);
        console.log("Block number: ", block.number);

        vm.startPrank(vault);
        deal(testConfig.vaultAsset, address(vault), amountToAllocate);
        vm.stopPrank();

        vm.startPrank(admin);
        IAllocator(allocator).allocate(address(strategy), amountToAllocate);

        uint256 allocationAmount0 = IVaultV2(vault).allocation(ID);
        console.log("Allocation amount to MorphoYearnOGWETH after allocate: ", allocationAmount0);

        vm.stopPrank();

        vm.warp(block.timestamp + 31536000);
        vm.roll(25926447);

        // Build MORPHO URD reward root
        uint256 claimable = 3417e18; // 3417 MORPHO tokens which is 0.17% of our 1k WETH supply to YearnOGWETH vault and that current rate, this is worth USD 6800
        address account = address(strategy);
        address reward = MORPHO_TOKEN;

        address account2 = makeAddr("someOtherUser");

        bytes32 leaf1 = keccak256(bytes.concat(keccak256(abi.encode(account, reward, claimable))));
        bytes32 leaf2 = keccak256(bytes.concat(keccak256(abi.encode(account2, reward, claimable))));

        bytes32 root = _hashThePair(leaf1, leaf2);

        bytes32[] memory proof = new bytes32[](1);
        proof[0] = leaf2;

        emit Root(root);

        vm.startPrank(URD_OWNER);
        IUniversalRewardsDistributorBase(MORPHO_URD).setRoot(root, root);
        vm.stopPrank();

        uint256 URD_MORPHO_BAL = Token(MORPHO_TOKEN).balanceOf(address(MORPHO_URD));
        console.log("Morpho Balance of URD before claim: ", URD_MORPHO_BAL);

        vm.startPrank(admin);
        IUniversalRewardsDistributorBase(MORPHO_URD).claim(account, reward, claimable, proof);
        vm.stopPrank();

        uint256 URD_MORPHO_BAL_1 = Token(MORPHO_TOKEN).balanceOf(address(MORPHO_URD));
        console.log("Morpho Balance of URD after claim: ", URD_MORPHO_BAL_1);

        uint256 MORPHO_TOKEN_STUCK_INSIDE_STRAT = Token(MORPHO_TOKEN).balanceOf(address(strategy));
        console.log("MORPHO token stuck inside Strategy: ", MORPHO_TOKEN_STUCK_INSIDE_STRAT);
    }
}

```
