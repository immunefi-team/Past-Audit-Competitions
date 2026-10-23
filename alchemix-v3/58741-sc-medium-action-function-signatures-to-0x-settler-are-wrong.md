# 58741 sc medium action function signatures to 0x settler are wrong

**Submitted on Nov 4th 2025 at 11:05:49 UTC by @kenzo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58741
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

Calldata passed to the 0x Settler calls will be incorrect as the action signatures (action to take on 0x Settler) are incorrect and not the same as the ones 0x Settler is familiar with.

## Vulnerability Details

In the library above, the different actions have function signatures that specify which action should be taken in 0x Settler but these function signatures are wrong and when encoded along with the call to 0x Settler will result in no action being taken thus failing swaps for example.

```solidity
library ZeroXSwapVerifier {

    // Constants for 0x Settler function selectors
    bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f; // execute(SlippageAndActions,bytes[])
    bytes4 private constant EXECUTE_META_TXN_SELECTOR = 0x0476baab; // executeMetaTxn(SlippageAndActions,bytes[],address,bytes)

    // Action selectors for different swap types
    bytes4 private constant BASIC_SELL_TO_POOL = 0x5228831d;
    bytes4 private constant UNISWAPV3_VIP = 0x9ebf8e8d;
    bytes4 private constant RFQ_VIP = 0x0dfeb419;
    bytes4 private constant METATXN_VIP = 0xc1fb425e;
    bytes4 private constant CURVE_TRICRYPTO_VIP = 0x103b48be;
    bytes4 private constant UNISWAPV4_VIP = 0x38c9c147;
    bytes4 private constant TRANSFER_FROM = 0x8d68a156;
    bytes4 private constant NATIVE_DEPOSIT = 0xc876d21d;
    bytes4 private constant SELL_TO_LIQUIDITY_PROVIDER = 0xf1e0a1c3;
    bytes4 private constant DODOV1_VIP = 0x40a07c6c;
    bytes4 private constant VELODROME_V2_VIP = 0xb8df6d4d;
    bytes4 private constant DODOV2_VIP = 0xd92aadfb;

    ...

}
```

These function signatures for the external call to the 0x Settler are wrong and will lead to failing swaps in the 99% case and wrong function calls in 0x Settler in the 1% case.

For example, the `UNISWAPV3_VIP` action type on 0x Settler has the function selector: `0x22ce6ede` not `0x9ebf8e8d` as the Alchemix team expects it to be. This will force uniswap v3 swap calldata transactions to fail.

This is also the case for all the function selectors in this library as they are wrong.

For reference please see the 0x Settler functions and the selectors they correspond to onchain from the 0x Settler address here: <https://etherscan.io/address/0x70bf6634ee8cb27d04478f184b9b8bb13e5f4710#code>

For a detailed code view of the 0x Settler contract, look here: <https://vscode.blockscan.com/ethereum/0x70bf6634ee8cb27d04478f184b9b8bb13e5f4710>

When you get the function selector for the `UNISWAPV3_VIP` function from the 0x Settler address it is computed as:

```solidity

struct TokenPermissions {
        // ERC20 token address
        address token;
        // the maximum amount that can be spent
        uint256 amount;
    }

    /// @notice The signed permit message for a single token transfer
    struct PermitTransferFrom {
        TokenPermissions permitted;
        // a unique value for every token owner's signature to prevent signature replays
        uint256 nonce;
        // deadline on the permit signature
        uint256 deadline;
    }

function UNISWAPV3_VIP(
        address recipient,
        bytes memory path,
        ISignatureTransfer.PermitTransferFrom memory permit,
        bytes memory sig,
        uint256 amountOutMin
    ) external;
```

Thus, the function signature of the action is `0x22ce6ede` and not `0x9ebf8e8d`

The same issue affects all other action signatures below:

```solidity
// Action selectors for different swap types
    bytes4 private constant BASIC_SELL_TO_POOL = 0x5228831d;
    bytes4 private constant UNISWAPV3_VIP = 0x9ebf8e8d;
    bytes4 private constant RFQ_VIP = 0x0dfeb419;
    bytes4 private constant METATXN_VIP = 0xc1fb425e;
    bytes4 private constant CURVE_TRICRYPTO_VIP = 0x103b48be;
    bytes4 private constant UNISWAPV4_VIP = 0x38c9c147;
    bytes4 private constant TRANSFER_FROM = 0x8d68a156;
    bytes4 private constant NATIVE_DEPOSIT = 0xc876d21d;
    bytes4 private constant SELL_TO_LIQUIDITY_PROVIDER = 0xf1e0a1c3;
    bytes4 private constant DODOV1_VIP = 0x40a07c6c;
    bytes4 private constant VELODROME_V2_VIP = 0xb8df6d4d;
    bytes4 private constant DODOV2_VIP = 0xd92aadfb;
```

## Impact Details

Swap calls to the 0x Settler contract will not work as the function signatures of the actions will not be found in the contract and the low level calls to the 0x Settler will be successful but the function will not execute.

Also, on another note the `EXECUTE_SELECTOR` and `EXECUTE_META_TXN_SELECTOR` are also incorrect but I have decided to highlight it in this single submission too so I don't mistakenly duplicate myself.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol?utm\\_source=immunefi>

## Proof of Concept

## Proof of Concept

## POC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {TestERC20} from "./mocks/TestERC20.sol";

interface ISettler {
    struct AllowedSlippage {
        address payable recipient;
        IERC20 buyToken;
        uint256 minAmountOut;
    }

    struct TokenPermissions {
        // ERC20 token address
        address token;
        // the maximum amount that can be spent
        uint256 amount;
    }

    /// @notice The signed permit message for a single token transfer
    struct PermitTransferFrom {
        TokenPermissions permitted;
        // a unique value for every token owner's signature to prevent signature replays
        uint256 nonce;
        // deadline on the permit signature
        uint256 deadline;
    }

    function execute(AllowedSlippage calldata slippage, bytes[] calldata actions, bytes32 /* zid & affiliate */ )
        external
        payable
        returns (bool);

    function UNISWAPV3_VIP(
        address recipient,
        bytes memory path,
        PermitTransferFrom memory permit,
        bytes memory sig,
        uint256 amountOutMin
    ) external;
}

contract ZeroXSwapVerifierTest is Test {
    TestERC20 internal token;
    address constant owner = address(1);
    address constant spender = address(2);

    bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f; // execute(SlippageAndActions,bytes[])
    bytes4 private constant EXECUTE_META_TXN_SELECTOR = 0x0476baab; // executeMetaTxn(SlippageAndActions,bytes[],address,bytes)

    // Action selectors for different swap types
    bytes4 private constant BASIC_SELL_TO_POOL = 0x5228831d;
    bytes4 private constant UNISWAPV3_VIP = 0x9ebf8e8d;
    bytes4 private constant RFQ_VIP = 0x0dfeb419;
    bytes4 private constant METATXN_VIP = 0xc1fb425e;
    bytes4 private constant CURVE_TRICRYPTO_VIP = 0x103b48be;
    bytes4 private constant UNISWAPV4_VIP = 0x38c9c147;
    bytes4 private constant TRANSFER_FROM = 0x8d68a156;
    bytes4 private constant NATIVE_DEPOSIT = 0xc876d21d;
    bytes4 private constant SELL_TO_LIQUIDITY_PROVIDER = 0xf1e0a1c3;
    bytes4 private constant DODOV1_VIP = 0x40a07c6c;
    bytes4 private constant VELODROME_V2_VIP = 0xb8df6d4d;
    bytes4 private constant DODOV2_VIP = 0xd92aadfb;
    uint256 private _forkId;

    event ReturnedBytes(bytes);
    event Selector(bytes4);

    address public constant SETTLER = 0x70bf6634eE8Cb27D04478f184b9b8BB13E5f4710;

     function getForkBlockNumber() internal pure returns (uint256) {
        return 21783555;
    }

    function getRpcUrl() internal view returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function setUp() public {
        token = new TestERC20(1000e18, 18);
        deal(address(token), owner, 100e18);
        deal(address(token), spender, 100e18);

        // Fork setup
        string memory rpc = getRpcUrl();
        if (getForkBlockNumber() > 0) {
            _forkId = vm.createFork(rpc, getForkBlockNumber());
        } else {
            _forkId = vm.createFork(rpc);
        }
        vm.selectFork(_forkId);
    }
    
    // Test Uniswap V3 VIP with valid slippage
    function testVerifyUniswapV3VIPWrongActionsFunctionSigEncodedPOC() public { 
        bytes memory _calldata = _buildUniswapV3VIPCalldata(token, spender, 300); // 300 bps = 3% slippage
        bool verified = ZeroXSwapVerifier.verifySwapCalldata(
            _calldata,
            owner, 
            address(token), 
            1000 // 1000 bps = 10% max slippage
        );

        assertTrue(verified);

        bytes4 univ3VipSelector = ISettler.UNISWAPV3_VIP.selector; // @note action selector in 0x Settler is // 0x22ce6ede and right
        emit Selector(univ3VipSelector);

        (bool success, bytes memory result) = SETTLER.call(_calldata);
        emit ReturnedBytes(result);
    }
    
    
    function _buildUniswapV3VIPCalldata(TestERC20 _token, address recipient, uint256 bps) internal pure returns (bytes memory) {
        bytes memory fills = abi.encode(address(_token), 100e18);
        bytes memory action = abi.encodeWithSelector(
            UNISWAPV3_VIP, // @note action selector is 0x9ebf8e8d and wrong
            recipient,
            bps, // bps
            3000, // feeOrTickSpacing
            false, // feeOnTransfer
            fills
        );
        
        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: recipient,
            buyToken: address(0),
            minAmountOut: 0,
            actions: new bytes[](1)
        });
        saa.actions[0] = action;
        
        return abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));
    }
    
}

```
