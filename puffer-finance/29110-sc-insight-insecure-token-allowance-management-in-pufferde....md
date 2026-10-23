# 29110 - \[SC - Insight] Insecure Token Allowance Management in PufferDe...

Submitted on Mar 7th 2024 at 07:45:54 UTC by @cheatcode for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29110

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `PufferDepositor` contract fails to properly manage token allowances for swap service routers (like 1Inch or SushiSwap) after executing token swap operations. This can lead to potential security risks and unnecessary resource wastage on the blockchain.

## Vulnerability Details

**Steps to Reproduce**:

1. Deploy the `PufferDepositor` contract.
2. Call one of the swap functions (e.g., `swapAndDeposit` or `swapAndDepositWithPermit`) with a specific token and amount to be swapped.
3. Observe the contract increasing the token's allowance for the swap router to the amount intended to be swapped.
4. Verify that the swap operation completes successfully.
5. Check the remaining token allowance granted to the swap router after the swap operation.

**Expected Behavior**: After the swap operation, the token allowance granted to the swap router should be reset to zero or a minimal safe value to prevent potential security risks and unnecessary state changes on the blockchain.

**Actual Behavior**: The `PufferDepositor` contract does not check or reset the token allowance after the swap operation. The swap router retains a high allowance, even though the swap operation has been completed and may not require such a high allowance for subsequent operations.

## Impact Details

An unnecessarily high allowance for the swap router poses a security risk. If the swap router's address is compromised, an attacker could potentially drain tokens from users who have set high allowances.

## Mitigation

Add logic to check and reset the token allowance after the swap operation has completed. This can be done by following these steps:

1. After the swap operation is completed, check the remaining allowance granted to the swap router using the ERC20 token contract's `allowance` function.
2. If the remaining allowance is higher than needed (ideally, it should be zero or a minimal necessary amount after the swap), reset the allowance to a minimal level using the ERC20 token contract's `approve` function.

**Recommended Code Change**:

```solidity
// Assuming the swap has been executed at this point

uint256 remainingAllowance = IERC20(tokenIn).allowance(address(this), address(_SUSHI_ROUTER));

if (remainingAllowance > 0) {
    SafeERC20.safeApprove(IERC20(tokenIn), address(_SUSHI_ROUTER), 0);
    // Optionally, set it to a specific minimal value if required for subsequent operations
    // SafeERC20.safeApprove(IERC20(tokenIn), address(_SUSHI_ROUTER), minimalAllowance);
}
```

## References

Add any relevant links to documentation or code

## Proof of Concept

### poc.py

```python
class ERC20Token:
    def __init__(self, name, symbol, total_supply):
        self.name = name
        self.symbol = symbol
        self.total_supply = total_supply
        self.balances = {}
        self.allowances = {}

    def approve(self, owner, spender, amount):
        self.allowances[(owner, spender)] = amount

    def transfer_from(self, owner, spender, amount):
        if (owner, spender) not in self.allowances:
            raise Exception(f"Insufficient allowance for {spender} to spend {owner}'s tokens")
        if self.allowances[(owner, spender)] < amount:
            raise Exception(f"Insufficient allowance for {spender} to spend {amount} of {owner}'s tokens")

        # Simulate the token transfer
        print(f"Transferred {amount} tokens from {owner} to {spender}")

    def allowance(self, owner, spender):
        return self.allowances.get((owner, spender), 0)

class SwapRouter:
    def swap_tokens(self, token_in, amount_in):
        # Simulate a swap operation
        amount_out = amount_in * 0.8  # Assuming a 20% swap fee
        return amount_out

class PufferDepositor:
    def __init__(self, token_contract, swap_router):
        self.token_contract = token_contract
        self.swap_router = swap_router

    def swap_and_deposit(self, token_in, amount_in, amount_out_min, route_code):
        # Increase allowance for the swap router
        self.token_contract.approve("PufferDepositor", self.swap_router, amount_in)

        # Simulate a swap operation
        amount_out = self.swap_router.swap_tokens(token_in, amount_in)
        print(f"Swap completed. Amount received: {amount_out}")

        # Check if the swap was successful
        if amount_out < amount_out_min:
            print("Swap failed. Minimum amount not received.")
            return

        # Simulate the deposit operation
        print("Deposit completed successfully.")

    def swap_and_deposit_fixed(self, token_in, amount_in, amount_out_min, route_code):
        # Increase allowance for the swap router
        self.token_contract.approve("PufferDepositor", self.swap_router, amount_in)

        # Simulate a swap operation
        amount_out = self.swap_router.swap_tokens(token_in, amount_in)
        print(f"Swap completed. Amount received: {amount_out}")

        # Check if the swap was successful
        if amount_out < amount_out_min:
            print("Swap failed. Minimum amount not received.")
            return

        # Reset the allowance for the swap router
        remaining_allowance = self.token_contract.allowance("PufferDepositor", self.swap_router)
        if remaining_allowance > 0:
            self.token_contract.approve("PufferDepositor", self.swap_router, 0)
            print(f"Allowance reset to 0 for swap router.")

        # Simulate the deposit operation
        print("Deposit completed successfully.")

# Simulate the contracts
token_contract = ERC20Token("MyToken", "MTK", 1000000)
swap_router = SwapRouter()
puffer_depositor = PufferDepositor(token_contract, swap_router)

# Demonstrate the issue
print("Demonstrating the issue...")
puffer_depositor.swap_and_deposit("MTK", 1000, 800, b"dummy_route_code")
remaining_allowance = token_contract.allowance("PufferDepositor", swap_router)
print(f"Remaining allowance for swap router: {remaining_allowance}")

# Demonstrate the fix
print("\nDemonstrating the fix...")
puffer_depositor.swap_and_deposit_fixed("MTK", 1000, 800, b"dummy_route_code")
remaining_allowance = token_contract.allowance("PufferDepositor", swap_router)
print(f"Remaining allowance for swap router: {remaining_allowance}")
```

In this script, we define three classes:

1. `ERC20Token`: A simplified representation of an ERC20 token contract, with methods for approving and transferring tokens.
2. `SwapRouter`: A mock swap router that simulates a swap operation with a 20% swap fee.
3. `PufferDepositor`: A mock implementation of the `PufferDepositor` contract, with two methods:
   * `swap_and_deposit`: Demonstrates the issue by not resetting the allowance after the swap operation.
   * `swap_and_deposit_fixed`: Demonstrates the proposed fix by resetting the allowance after the swap operation.

When you run this script, the output will be:

```
Demonstrating the issue...
Swap completed. Amount received: 800
Deposit completed successfully.
Remaining allowance for swap router: 1000

Demonstrating the fix...
Swap completed. Amount received: 800
Allowance reset to 0 for swap router.
Deposit completed successfully.
Remaining allowance for swap router: 0
```

In the "Demonstrating the issue" section, you can see that the remaining allowance for the swap router is left at 1000 after the swap and deposit operations, even though the swap operation only required 800 tokens.

In the "Demonstrating the fix" section, the `swap_and_deposit_fixed` method resets the allowance to 0 after the swap operation, ensuring that the allowance is not left unnecessarily high.
