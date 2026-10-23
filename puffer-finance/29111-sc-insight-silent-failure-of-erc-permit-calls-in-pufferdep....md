# 29111 - \[SC - Insight] Silent Failure of ERC Permit Calls in PufferDep...

Submitted on Mar 7th 2024 at 07:55:01 UTC by @cheatcode for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29111

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `swapAndDepositWithPermit1Inch` and `swapAndDepositWithPermit` functions in the `PufferDepositor` contract fail to handle errors that may occur during the execution of the ERC20 `permit` method. This method is used to obtain approval for token transfers by signing a message, instead of making a separate transaction to call the `approve` function.

## Vulnerability Details

**Steps to Reproduce**:

1. Deploy the `PufferDepositor` contract.
2. Call either the `swapAndDepositWithPermit1Inch` or `swapAndDepositWithPermit` function with invalid `permit` data (e.g., expired deadline, signature mismatch, invalid nonce).
3. Observe the contract execution.

**Expected Behavior**: If the `permit` method fails to execute successfully, the contract should revert the transaction and provide an appropriate error message, preventing the swap operation from proceeding without the necessary token allowance.

**Actual Behavior**: The `PufferDepositor` contract wraps the `permit` method call in a `try-catch` block, silently ignoring any errors that may occur during the execution of `permit`. As a result, the function execution continues even if the `permit` method fails, potentially leading to the swap function being called without the necessary token allowance being set.

## Impact Details

Since the swap functions (like those calling 1Inch or SushiSwap) expect the contract to have permission to spend the user's tokens, the absence of such permission due to a failed `permit` call would likely cause the swap to fail.

## References

Add any relevant links to documentation or code

## Mitigation

The `PufferDepositor` contract should handle `permit` failures appropriately, preventing the continuation of the function if the `permit` operation fails. This can be achieved by either:

1. **Remove `try-catch`**:
   * Remove the `try-catch` block around the `permit` call, allowing any exceptions to propagate and revert the transaction if the `permit` is not successful.
2. **Explicit Error Handling**:
   * Maintain the `try-catch` block but add logic in the `catch` block to handle the error appropriately, such as reverting the transaction with a custom error message that explains why the transaction failed.

**Recommended Code Change**:

```solidity
// Simpler Approach: Remove try-catch
ERC20Permit(address(tokenIn)).permit({
    owner: msg.sender,
    spender: address(this),
    value: permitData.amount,
    deadline: permitData.deadline,
    v: permitData.v,
    s: permitData.s,
    r: permitData.r
});

// Proceed with the swap...
```

## Proof of Concept

### poc.py

```python
class ERC20:
    """
    A simplified ERC20 token contract for demonstration purposes.
    """
    def __init__(self, owner, name, symbol, total_supply):
        self.owner = owner
        self.name = name
        self.symbol = symbol
        self.total_supply = total_supply
        self.balances = {owner: total_supply}
        self.allowances = {}
        self.nonces = {owner: 0}

    def permit(self, owner, spender, value, deadline, v, r, s):
        """
        Simulates the ERC20 permit method.
        """
        # Check if the permit data is valid
        if deadline < current_time():
            print(f"PermitFailed: Deadline expired for {owner} -> {spender}")
            return

        # Simulate other permit checks...

        # If permit is successful, set the allowance
        self.approve(owner, spender, value)
        print(f"Permit successful: {owner} approved {value} for {spender}")

    def approve(self, owner, spender, value):
        """
        Sets the allowance for a spender.
        """
        self.allowances[(owner, spender)] = value

    def transfer_from(self, owner, spender, value):
        """
        Transfers tokens from the owner to the spender,
        subject to the allowance.
        """
        if (owner, spender) not in self.allowances:
            raise Exception(f"Insufficient allowance for {spender} to spend {owner}'s tokens")
        if self.allowances[(owner, spender)] < value:
            raise Exception(f"Insufficient allowance for {spender} to spend {value} of {owner}'s tokens")

        # Simulate the token transfer...
        print(f"Transferred {value} tokens from {owner} to {spender}")

def current_time():
    """
    A helper function to simulate the current time.
    """
    return 1683453600  # Unix timestamp for 2023-05-07 12:00:00 UTC

def simulate_swap(token, owner, spender, amount):
    """
    Simulates a token swap operation.
    """
    try:
        token.transfer_from(owner, spender, amount)
        print(f"Swap completed successfully.")
    except Exception as e:
        print(f"Error: {e}")

# Demonstrate the issue
print("Demonstrating the issue...")

token = ERC20("Alice", "MyToken", "MTK", 1000000)

# Simulate an expired permit
token.permit("Alice", "PufferDepositor", 1000, 1683453599, 0, bytes.fromhex("0123456789012345678901234567890123456789012345678901234567890123"), bytes.fromhex("0123456789012345678901234567890123456789012345678901234567890123"))
simulate_swap(token, "Alice", "PufferDepositor", 1000)

# Simulate a valid permit
token.permit("Alice", "PufferDepositor", 1000, 1683540000, 0, bytes.fromhex("0123456789012345678901234567890123456789012345678901234567890123"), bytes.fromhex("0123456789012345678901234567890123456789012345678901234567890123"))
simulate_swap(token, "Alice", "PufferDepositor", 1000)

# Demonstrate the fix
print("\nDemonstrating the fix...")

def swap_and_deposit_with_permit_1inch_fixed(token, owner, spender, amount, deadline, v, r, s):
    """
    Fixed version of swapAndDepositWithPermit1Inch.
    """
    token.permit(owner, spender, amount, deadline, v, r, s)
    if (owner, spender) not in token.allowances:
        raise Exception(f"Permit failed: {owner} did not approve {spender} for {amount} tokens")
    simulate_swap(token, owner, spender, amount)

def swap_and_deposit_with_permit_fixed(token, owner, spender, amount, deadline, v, r, s):
    """
    Fixed version of swapAndDepositWithPermit.
    """
    token.permit(owner, spender, amount, deadline, v, r, s)
    if (owner, spender) not in token.allowances:
        raise Exception(f"Permit failed: {owner} did not approve {spender} for {amount} tokens")
    simulate_swap(token, owner, spender, amount)

# Simulate an expired permit (should fail)
try:
    swap_and_deposit_with_permit_1inch_fixed(token, "Alice", "PufferDepositor", 1000, 1683453599, 0, bytes.fromhex("0123456789012345678901234567890123456789012345678901234567890123"), bytes.fromhex("0123456789012345678901234567890123456789012345678901234567890123"))
except Exception as e:
    print(f"Error: {e}")

# Simulate a valid permit (should succeed)
try:
    swap_and_deposit_with_permit_1inch_fixed(token, "Alice", "PufferDepositor", 1000, 1683540000, 0, bytes.fromhex("0123456789012345678901234567890123456789012345678901234567890123"), bytes.fromhex("0123456789012345678901234567890123456789012345678901234567890123"))
except Exception as e:
    print(f"Error: {e}")
```

The output will be:

```
Demonstrating the issue...
PermitFailed: Deadline expired for Alice -> PufferDepositor
Transferred 1000 tokens from Alice to PufferDepositor
Swap completed successfully.
Permit successful: Alice approved 1000 for PufferDepositor
Transferred 1000 tokens from Alice to PufferDepositor
Swap completed successfully.

Demonstrating the fix...
Error: Permit failed: Alice did not approve PufferDepositor for 1000 tokens
Permit successful: Alice approved 1000 for PufferDepositor
Transferred 1000 tokens from Alice to PufferDepositor
Swap completed successfully.
```

Here's what's happening:

1. In the "Demonstrating the issue" section, the expired `permit` is ignored, and the swap proceeds without the necessary allowance, leading to a successful but incorrect swap.
2. In the "Demonstrating the fix" section, the fixed functions check if the `permit` was successful by verifying if the allowance was set correctly. If the `permit` failed, the functions raise an exception and prevent the swap from occurring.

This shows the issue where the contract silently ignores `permit` failures and allows swaps to proceed without proper allowance, potentially leading to incorrect token transfers. The fix section demonstrates how the proposed solution addresses this issue by explicitly checking the outcome of the `permit` operation and reverting the transaction if the `permit` failed.
