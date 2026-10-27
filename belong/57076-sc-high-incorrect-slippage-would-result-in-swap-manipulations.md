# 57076 sc high incorrect slippage would result in swap manipulations

**Submitted on Oct 23rd 2025 at 09:13:34 UTC by @kodyvim for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57076
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

Incorrect on-chain slippage calculation enables sandwich/MEV swap manipulation

### Vulnerability Details

`_swapExact()` is used to perform an exact USDC amount swaps to LONG via uniswap and vice versa.

```solidity
function _swapUSDCtoLONG(address recipient, uint256 amount) internal virtual returns (uint256 swapped) {
        PaymentsInfo memory p = belongCheckInStorage.paymentsInfo;
@>      return _swapExact(p.usdc, p.long, recipient, amount);
    }
function _swapLONGtoUSDC(address recipient, uint256 amount) internal virtual returns (uint256 swapped) {
        PaymentsInfo memory p = belongCheckInStorage.paymentsInfo;
        return _swapExact(p.long, p.usdc, recipient, amount);
    }
```

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
            IV3Quoter(_paymentsInfo.swapV3Quoter).quoteExactInput(path, amount).amountOutMin(_paymentsInfo.slippageBps);//@audit-info invalid slippage

        IV3Router.ExactInputParamsV1 memory swapParamsV1 = IV3Router.ExactInputParamsV1({
            path: path,
            recipient: recipient,
            deadline: block.timestamp,
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

The issue is that the minimum allowable slippage is determined within the transaction. This offers no slippage protection as the slippage is determined within the already manipulated pool.

### Impact Details

Incorrect slippage would lead to swap manipulation by MEV/attacker

### References

<https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol?utm\\_source=immunefi#L664C9-L666C1>

## Proof of Concept

{% stepper %}
{% step %}

### Step

Attacker front-run: pushes pool so now 1 USDC => 0.90 LONG (price worsens for buyer).
{% endstep %}

{% step %}

### Step

Victim quotes at 0.90 and gets amountOutMinimum ≈ 0.90\*(1−slippage).
{% endstep %}

{% step %}

### Step

Victim executes and receives \~0.90 LONG per USDC.
{% endstep %}

{% step %}

### Step

Attacker back-runs and profit = difference between what attacker got on the back-run and cost of front-run.
{% endstep %}
{% endstepper %}
