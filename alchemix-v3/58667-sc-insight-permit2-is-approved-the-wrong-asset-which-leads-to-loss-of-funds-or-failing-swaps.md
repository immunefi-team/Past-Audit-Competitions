# 58667 sc insight permit2 is approved the wrong asset which leads to loss of funds or failing swaps

**Submitted on Nov 3rd 2025 at 23:13:23 UTC by @oxrex for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58667
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/PeapodsETH.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

Permit2 is approved the wrong asset which leads to loss of funds or failing swaps while later using the 0x Settler.

## Vulnerability Details

First off, the intention for the Permit2 approvals by the Alchemix strategy contracts is to approve the deposit/withdrawal asset (`USDC`, `WETH`) to the Permit2 contract.

For the EulerWETHStrategy for example, this asset is `WETH`, also for the PeapodsETH strategy this asset is `WETH`, not `_peapodsEth` as it currently does.

```solidity
contract PeapodsETHStrategy is MYTStrategy {
    IERC4626 public immutable vault;
    WETH public immutable weth;

    // @audit has 1 issue with wrong asset being approved to permit2
    constructor(address _myt, StrategyParams memory _params, address _peapodsEth, address _weth, address _permit2Address)
    // @audit wrong args
@>        MYTStrategy(_myt, _params, _permit2Address, _peapodsEth)
    {   
        vault = IERC4626(_peapodsEth);
        weth = WETH(_weth);
    }
...
}
```

```solidity
constructor(address _myt, StrategyParams memory _params, address _permit2Address, address _receiptToken) Ownable(_params.owner) {
        ...

        permit2Address = _permit2Address;

        // IERC20 vaultAsset = IERC20(address(MYT.asset()));
        // vaultAsset.approve(permit2Address, type(uint256).max);
@>        IERC20 receiptTokenContract = IERC20(receiptToken);
@>        receiptTokenContract.approve(permit2Address, type(uint256).max);

        // TODO add the strategy to the perpetual gauge in an authenticated manner
        // TODO perhap take initial snapshot now to set up start block
    }
```

As we can see above, we will approve `_peapodsEth` to the Permit2 contract instead of `WETH`.

These contracts below also wrongly approve the ERC4626 vault shares instead of the asset being allocated or will be later deallocated and swapped by the 0x Settler.

These are:

* TokeAutoUSDStrategy
* TokeAutoEthStrategy
* PeapodsUSDCStrategy

Only the asset being deposited into the third party vaults (Euler, Peapods, Aave) are to be approved to Permit2. Not the vault shares itself which opens way for theft of the vault shares from the Alchemix strategy contracts.

Some references from the Alchemix protocol for this:

* The allocator who is whitelisted to allocate/deallocate from the vault (and its corresponding strategies) fetches the 0x swap calldata from their rest api
* But we do not trust this calldata blindly because the strategy was not part of this quote process so it might contain invalid assets or amounts
* So the strategy calls the deployed library's verify call which is a simplified version of 0x's own internal verification again to ensure we only allow the strategies assets to be swapped, in one direction and with matching allocation amounts. Otherwise the 0x settler executes any quote in the name of the strategy contract that's valid.

## Impact Details

There are two distinct impacts:

1. The 0x Settler swaps in the `TokeAutoUSDStrategy`, `PeapodsUSDCStrategy`, `TokeAutoEthStrategy`, and `PeapodsETHStrategy` strategies will fail
2. The Permit2 address will be approved for the wrong asset class and it will be possible to use the 0x Settler to swap out vault shares (e.g `_peapodsEth` and `_peapodsUSDC`) from the strategy contract which is not what the Alchemix team wants for these strategy contracts

Follow the same implementation as done in `EulerWETHStrategy` for example where we pass in `WETH` to the `MYTStrategy` constructor:

```solidity
contract EulerWETHStrategy is MYTStrategy {
    IERC20 public immutable weth; // Mainnet WETH
    IERC4626 public immutable vault;

    constructor(address _myt, StrategyParams memory _params, address _weth, address _eulerVault, address _permit2Address)
        MYTStrategy(_myt, _params, _permit2Address, _weth)
    {
         weth = IERC20(_weth);
        vault = IERC4626(_eulerVault);
    }
}
```

or if the intent is to approve the vault shares to permit2 and not the vault asset, then maintain that for all the strategies in-scope otherwise it would be possible to steal/swap out the other asset via permit2 in 0x settler swaps or the swaps completely fail because of wrong token approval in the constructor.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/PeapodsETH.sol?utm\\_source=immunefi#L22-L27>

## Proof of Concept

## Proof of Concept

Paste the POC below into the `PeapodsETHStrategy.t.sol` test contract and run it:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {PeapodsETHStrategy} from "../../strategies/mainnet/PeapodsETH.sol";
import {IAllocator} from "../../interfaces/IAllocator.sol";

contract MockPeapodsETHStrategy is PeapodsETHStrategy {
    constructor(address _myt, StrategyParams memory _params, address _vault, address _weth, address _permit2Address)
        PeapodsETHStrategy(_myt, _params, _vault, _weth, _permit2Address)
    {}
}

interface IVault {
    function balanceOf(address user) external view returns (uint256);
}

interface IERC20 {
    function balanceOf(address user) external view returns (uint256);
}

interface P2 {

    struct TokenPermissions {
        // ERC20 token address
        address token;
        // the maximum amount that can be spent
        uint256 amount;
    }

    struct PermitTransferFrom {
        TokenPermissions permitted;
        // a unique value for every token owner's signature to prevent signature replays
        uint256 nonce;
        // deadline on the permit signature
        uint256 deadline;
    }

    struct SignatureTransferDetails {
        // recipient address
        address to;
        // spender requested amount
        uint256 requestedAmount;
    }
    
    function permitTransferFrom(
        PermitTransferFrom memory permit,
        SignatureTransferDetails calldata transferDetails,
        address owner,
        bytes calldata signature
    ) external;

    function DOMAIN_SEPARATOR() external returns (bytes32);
}

contract PeapodsETHStrategyTest is BaseStrategyTest {
    address public constant PEAPODS_ETH_VAULT = 0x9a42e1bEA03154c758BeC4866ec5AD214D4F2191;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    // address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant MAINNET_PERMIT2 = 0x000000000022D473030F116dDEE9F6B43aC78BA3;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "PeapodsETH",
            protocol: "PeapodsETH",
            riskClass: IMYTStrategy.RiskClass.HIGH,
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

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockPeapodsETHStrategy(vault, params, PEAPODS_ETH_VAULT, WETH, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 23_667_528;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function test_wrongAssetApproved() public {
        uint256 amountToTransferForSwap = 100e18;

        vm.startPrank(vault);
        deal(testConfig.vaultAsset, address(strategy), amountToTransferForSwap);
        vm.stopPrank();

        address swapper = makeAddr("0xsettler");

        uint256 nonce = 0;
        uint256 deadline = block.timestamp + 1 hours;

        P2.PermitTransferFrom memory permit = P2.PermitTransferFrom({
            permitted: P2.TokenPermissions({
                token: address(testConfig.vaultAsset),
                amount: amountToTransferForSwap
            }),
            nonce: nonce,
            deadline: deadline
        });

        // Domain separator (same as Permit2)
        bytes32 domainSeparator = P2(MAINNET_PERMIT2).DOMAIN_SEPARATOR();

        bytes32 PERMIT_TYPEHASH = keccak256("PermitTransferFrom(TokenPermissions permitted,address spender,uint256 nonce,uint256 deadline)TokenPermissions(address token,uint256 amount)");

        bytes32 TOKEN_PERMISSIONS_TYPEHASH = keccak256("TokenPermissions(address token,uint256 amount)");
        bytes32 tokenPermHash = keccak256(abi.encode(
            TOKEN_PERMISSIONS_TYPEHASH,
            address(testConfig.vaultAsset),
            amountToTransferForSwap
        ));

        bytes32 structHash = keccak256(
            abi.encode(
                PERMIT_TYPEHASH,
                tokenPermHash,
                nonce,
                deadline
            )
        );

        bytes32 digest = keccak256(abi.encodePacked("\x19\x01", domainSeparator, structHash));

        // BURNER API KEY FOR TESTING AND SIGNING MESSAGES OFFCHAIN
        uint256 key = uint256(0x2de680fdf234bd96a444c95a7dd79cac4fdebdc3cd9f6f7a98b96e65c0f0da85);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(key, digest);
        bytes memory signature = abi.encodePacked(r, s, v);

        P2.SignatureTransferDetails memory details = P2.SignatureTransferDetails({
                to: swapper,
                requestedAmount: amountToTransferForSwap
            });

        P2(MAINNET_PERMIT2).permitTransferFrom(permit, details, address(strategy), signature);
    }
}
```

Rightfully, the POC above reverts since we approve the vault shares to Permit2 rather than the vault asset.

```javascript
0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2::transferFrom(MockPeapodsETHStrategy: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1], 0xsettler: [0xeE107EAE3F9806261DE348c594eC0Ebfae54F0D4], 100000000000000000000 [1e20])
    │   │   └─ ← [Revert] EvmError: Revert
    │   └─ ← [Revert] TRANSFER_FROM_FAILED
    └─ ← [Revert] TRANSFER_FROM_FAILED
```
