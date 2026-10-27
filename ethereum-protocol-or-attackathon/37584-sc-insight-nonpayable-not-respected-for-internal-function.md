# #37584 \[SC-Insight] Nonpayable Not Respected For Internal Function

**Submitted on Dec 9th 2024 at 20:41:03 UTC by @anatomist for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37584
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Elimination of security checks
  * (Compiler) Incorrect bytecode generation leading to incorrect behavior
  * (Compiler) Unexpected behavior

## Description

## Brief/Intro

Vyper merges payable / nonpayable distinctions into mutability enum allowing those decorators to be used for internal functions, but does not fully enforces it. This may lead to ambiguity on what contract functions guard against. In the worst case, this may result in missed checks causing program behavior to diverge from user intentions, and eventually lead to funds loss.

## Vulnerability Details

Vyper `StateMutability` encompasses both `pure` / `view` / (`mutable`) and the slightly different `payable` / `nonpayable` distinctions.

```
class StateMutability(StringEnum):
    PURE = enum.auto()
    VIEW = enum.auto()
    NONPAYABLE = enum.auto()
    PAYABLE = enum.auto()
```

This means they're treated the same in many contexts, including whether the decorator can be specified for internal functions. While `pure` and `view` are required annotations for internal functions to keep semantic checks simple, `nonpayable` and `payable` are a lot more ambiguous when used in this context.

For example, compile time checks are enforced when attempting a call a `nonpayable` function from a `view` function.

```
# this fails to compile, since view functions can only call pure and view functions
@external
@view
def viewonly():
    self.internal_nonpayable()

@nonpayable
def internal_nonpayable():
    pass
```

Based on the compiler warning and the fact that practically all other decorators (nonreentrant, view, pure) are enforced for all functions, users may be inclined to believe `nonpayable` checks are respected for internal functions too, and `zerovalue()` in the following example reverts when `msg.value` is not 0.

```
@external
@payable
def test(zerovalue: bool) -> uint256:
    if zerovalue:
        self.zerovalue()
    else:
        pass

@nonpayable
def zerovalue():
    pass
```

However, `nonpayable` decorators are only enforced when specified for `external` and `deploy` functions, so in practice, the check is skipped.

```
def _ir_for_fallback_or_ctor(func_ast, *args, **kwargs):
    ...
    if not func_t.is_payable:
        callvalue_check = ["assert", ["iszero", "callvalue"]]
        ret.append(IRnode.from_list(callvalue_check, error_msg="nonpayable check"))
    ...

def _selector_section_linear(external_functions, module_t):
    ...
    for sig, entry_point in entry_points.items():
        ...
        if not func_t.is_payable:
            callvalue_check = ["assert", ["iszero", "callvalue"]]
            dispatch.append(IRnode.from_list(callvalue_check, error_msg="nonpayable check"))
        ...
    ...
```

Going into details, vyper currently defaults functions without `StateMutability` decorators to `nonpayable`. In our opinion, this is the root cause behind the lack of nonpayable checks bytecode emission, since the compiler can't differentiate between an explicitly specified `nonpayable` and an implicit placeholder `nonpayable`. There are two clean ways to work around this.

1. Disallow the usage of `payable` and `nonpayable` decorators for internal functions. This eliminates the ambiguity since the compiler mandates `payable` and `nonpayable` should only be applied to external functions. The approach is similar to solidity, which doesn't have `nonpayable`, but restricts `payable` attributes to external functions.
2. Default to `payable` for internal functions. Since `payable` is less restrictive, defaulting to `payable` for internal functions allows us to generate strict checks for user specified `nonpayable` everywhere. However, care must be taken for external functions, which should still default to `nonpayable` to not break developer expectations.

## Impact Details

In certain situations, this might lead to impactful scenarios. For example, the following vault contract has a funds draining bug due to the `nonpayable` check not being enforced for `zerovalue` when `no_mint` is set to `True`.

```
event Burn:
    shares: uint256
    balance: uint256

event Mint:
    shares: uint256
    balance: uint256

totalShares: uint256
shares: HashMap[address, uint256]

@deploy
@payable
def __init__():
    #setup values for testing, in practice, this can be reached through normal vault operations
    assert(msg.value == 1000)
    self.totalShares = 500
    self.shares[msg.sender] = 250
    self.shares[0x0000000000000000000000000000000000000000] = 250

@external
@payable
def adjust(no_mint: bool, shares: uint256):
    # after deployment, calling with (no_mint = True, shares = 500) and value = 1000 will result in draining the vault
    # compare against calling with (no_mint = False, shares = 500) and value = 1000, or splitting mint / burn into two separate txs

    # we omit implementation for handling 0 share and balance scenarios since it's not relevant to the PoC
    orig: uint256 = self.balance
    if no_mint:
        self.zerovalue()
    else:
        orig -= msg.value

    if msg.value > 0:
        mint: uint256 = msg.value * self.totalShares // orig
        self.shares[msg.sender] += mint
        self.totalShares += mint
        orig += msg.value
        log Mint(shares = mint, balance = msg.value)

    if shares > 0:
        amount: uint256 = shares * orig // self.totalShares
        self.totalShares -= shares
        self.shares[msg.sender] -= shares
        send(msg.sender, amount)
        log Burn(shares = shares, balance = amount)

@nonpayable
def zerovalue():
    # This function is used to check against msg.value, but fails to do so.
    # It may look strange to use a separate function for this, but in real world scenarios, if more actions
    # are performed in the function and it is reused several times, it starts to look more and more reasonable
    pass

# other functionalities such as yield accrual are omitted due since they're irrelevant to the PoC
```

## References

* <https://github.com/vyperlang/vyper/blob/c8691ac5dd95623991e51205bc90a720fc513766/vyper/semantics/analysis/base.py#L25>
* <https://github.com/vyperlang/vyper/blob/c8691ac5dd95623991e51205bc90a720fc513766/vyper/codegen/module.py#L403>

## Proof of Concept

## Proof of Concept

Already shown in Vulnerability Details and Impact Details section.
