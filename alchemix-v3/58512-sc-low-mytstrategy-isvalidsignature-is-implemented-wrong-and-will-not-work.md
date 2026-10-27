# 58512 sc low mytstrategy isvalidsignature is implemented wrong and will not work

**Submitted on Nov 2nd 2025 at 22:57:31 UTC by @oxrex for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58512
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

The MYTStrategy contract's `isValidSignature` function is implemented and will not work with Permit2 during 0x Settler contract calls to swap out assets from the strategy contracts.

## Vulnerability Details

```solidity
function isValidSignature(bytes32 _hash, bytes memory _signature) public view returns (bytes4) {
        return IPermit2(permit2Address).isValidSignature(_hash, _signature);
    }
```

This snippet above from the MYTStrategy contract is completely wrong.

The way Permit2 works is that, Permit2 will call the strategy contract during transfers from 0x Settler and the function permit2 will call is the `isValidSignature()` on the strategy contract. Permit2 makes this call when the `claimedSigner` aka address strategy which the address that approved Permit2 for tokens is not an EOA and instead a contract. Here's that specific call here <https://etherscan.io/address/0x000000000022d473030f116ddee9f6b43ac78ba3#code#F13#L46> from permit2:

```solidity
function verify(bytes calldata signature, bytes32 hash, address claimedSigner) internal view {
        bytes32 r;
        bytes32 s;
        uint8 v;

        if (claimedSigner.code.length == 0) {
            if (signature.length == 65) {
                (r, s) = abi.decode(signature, (bytes32, bytes32));
                v = uint8(signature[64]);
            } else if (signature.length == 64) {
                // EIP-2098
                bytes32 vs;
                (r, vs) = abi.decode(signature, (bytes32, bytes32));
                s = vs & UPPER_BIT_MASK;
                v = uint8(uint256(vs >> 255)) + 27;
            } else {
                revert InvalidSignatureLength();
            }
            address signer = ecrecover(hash, v, r, s);
            if (signer == address(0)) revert InvalidSignature();
            if (signer != claimedSigner) revert InvalidSigner();
        } else {
@>            bytes4 magicValue = IERC1271(claimedSigner).isValidSignature(hash, signature);
@>            if (magicValue != IERC1271.isValidSignature.selector) revert InvalidContractSignature();
        }
    }
```

As you can see, Permit2 wants us (strategy contract) to return the `4-byte` function selector of the `isValidSignature` function which is `0x1626ba7e`, it does not want us to call back into Permit2.

And this whole call starts from the `permitTransferFrom` call which 0x Settler will initiate on Permit2 during swaps:

```solidity
function permitTransferFrom(
        PermitBatchTransferFrom memory permit,
        SignatureTransferDetails[] calldata transferDetails,
        address owner,
        bytes calldata signature
    ) external {
        _permitTransferFrom(permit, transferDetails, owner, permit.hash(), signature);
    }
```

```solidity
function _permitTransferFrom(
        PermitBatchTransferFrom memory permit,
        SignatureTransferDetails[] calldata transferDetails,
        address owner,
        bytes32 dataHash,
        bytes calldata signature
    ) private {
        uint256 numPermitted = permit.permitted.length;

        if (block.timestamp > permit.deadline) revert SignatureExpired(permit.deadline);
        if (numPermitted != transferDetails.length) revert LengthMismatch();

        _useUnorderedNonce(owner, permit.nonce);
@>        signature.verify(_hashTypedData(dataHash), owner);

        ...
    }
```

The correct implementation is to return the `isValidSignature` function selector which we can do for example like:

```solidity
function isValidSignature(bytes32 hash, bytes memory signature)
        external
        view
        returns (bytes4)
    {   
        // @note we fetch the signer of the message
        address signer = ECDSA.recover(hash, signature);
        // @note we then verify if the signer is someone we trust e.g the _owner address or e.g a sig generator contract contract 
        if (signer == owner()) {
            // @note if the signer of the message is a person we trust, we then return the correct function selector
            return 0x1626ba7e;
        } else {
            // @note if the owner is not someone we trust, we return a false sig and then the swap is unauthorized
            return 0xffffffff;
        }
    }
```

## Impact Details

The `isValidSignature` will always force a revert from Permit2 contract as the Permit2 address does not implement a `isValidSignature` function. The MYTStrategy is implementing the `isValidSignature` function wrong as I have explained in the vulnerability details above.

For the severity of this bug, I have selected the High Risk severity classification because we can also see that since the function is implemented wrong, swaps with 0x Settler will technically not be successful even when the approvals to Permit2 are available and thus we can classify this as funds be locked for atleast 24 hours since the protocol would technically want to swap out while the batch withdrawal is implemented and this can cause the funds to be unable to be swapped for 24 hours and more.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol#L311-L314>

## Proof of Concept

## Proof of Concept

You can replace the content of the EulerUSDCStrategy.t.sol test file with the below code and run the test with `test_permitPOC` and verbosity of 3 (-vvv)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {EulerUSDCStrategy} from "../../strategies/mainnet/EulerUSDCStrategy.sol";
import {IAllocator} from "../../interfaces/IAllocator.sol";
import {IPermit2} from "../../interfaces/IPermit2.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
// import {Permit2} from "permit2/src/Permit2.sol";

contract MockEulerUSDCStrategy is EulerUSDCStrategy {
    constructor(address _myt, StrategyParams memory _params, address _vault, address _usdc, address _permit2Address)
        EulerUSDCStrategy(_myt, _params, _vault, _usdc, _permit2Address)
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

contract EulerUSDCStrategyTest is BaseStrategyTest {
    address public constant EULER_USDC_VAULT = 0xe0a80d35bB6618CBA260120b279d357978c42BCE;
    address public constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    // address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant MAINNET_PERMIT2 = 0x000000000022D473030F116dDEE9F6B43aC78BA3;

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

    struct SignatureTransferDetails {
        // recipient address
        address to;
        // spender requested amount
        uint256 requestedAmount;
    }

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "EulerUSDC",
            protocol: "EulerUSDC",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000e6,
            globalCap: 1e18,
            estimatedYield: 100e6,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: USDC, vaultInitialDeposit: 1000e6, absoluteCap: 10_000e6, relativeCap: 1e18, decimals: 6});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockEulerUSDCStrategy(vault, params, USDC, EULER_USDC_VAULT, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22_089_302;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function test_permitPOC() public {
        uint256 amountToTransferForSwap = 100e6;

        vm.startPrank(vault);
        deal(testConfig.vaultAsset, address(strategy), amountToTransferForSwap);
        vm.stopPrank();

        address swapper = makeAddr("swapper");
        console.log("Balance of Strategy pre Ox and Permit2 call: ", IERC20(USDC).balanceOf(address(strategy)));
        console.log("Balance of swapper pre Ox and Permit2 call: ", IERC20(USDC).balanceOf(swapper));

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

        console.log("Balance of Strategy post Ox and Permit2 call: ", IERC20(USDC).balanceOf(address(strategy)));
        console.log("Balance of swapper post Ox and Permit2 call: ", IERC20(USDC).balanceOf(swapper));
    }
}

```

You will notice the test reverts now, but when you fix the `isValidSignature` function inside the MYTStrategy to something like below for example (for a quick confirmation, please add the suggested recommendation after confirmation and don't blindly return the sig i.e verify signer is e.g owner):

```solidity
function isValidSignature(bytes32 _hash, bytes memory _signature) public view returns (bytes4) {
        return 0x1626ba7e;
    }
```

The test case will then pass after this fix and 0x Settler swaps will then be successful to move assets around that we approved to the Permit2 address.

Test logs for successful case with fix:

```javascript
[PASS] test_permitPOC() (gas: 325480)
Logs:
  Balance of Strategy pre Ox and Permit2 call:  100000000
  Balance of swapper pre Ox and Permit2 call:  0
  Balance of Strategy post Ox and Permit2 call:  0
  Balance of swapper post Ox and Permit2 call:  100000000
```
