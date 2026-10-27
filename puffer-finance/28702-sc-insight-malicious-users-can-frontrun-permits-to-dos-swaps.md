# 28702 - \[SC - Insight] Malicious users can frontrun permits to DoS swaps

Submitted on Feb 24th 2024 at 12:22:34 UTC by @jaraxxus for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28702

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A novel attack recently discovered, if permit is used within a function, it can be frontrunned and griefed.

## Vulnerability Details

When swaps are available to the users (currently restricted), there is an option to do gasless transactions through the use of permit.

```
        try ERC20Permit(address(tokenIn)).permit({
            owner: msg.sender,
            spender: address(this),
            value: permitData.amount,
            deadline: permitData.deadline,
            v: permitData.v,
            s: permitData.s,
            r: permitData.r
        }) { } catch { }
```

If permit is used within a larger function, it can be frontrunned. Also, since the try/catch block does not catch any errors, nothing will prevent the frontrunning.

## Impact Details

Permit will be frontrunned and swaps can be griefed. The user will lose the functionality that follows the permit().

## References

<https://vscode.blockscan.com/ethereum/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Permit issue: <https://www.trust-security.xyz/post/permission-denied>

## Proof of Concept

The permit call is within the `swapAndDepositWithPermit()` function. Also, the permit call does not catch any errors.

```
  function swapAndDepositWithPermit(
        address tokenIn,
        uint256 amountOutMin,
        IPufferDepositor.Permit calldata permitData,
        bytes calldata routeCode
    ) public payable virtual restricted returns (uint256 pufETHAmount) {
>       try ERC20Permit(address(tokenIn)).permit({
            owner: msg.sender,
            spender: address(this),
            value: permitData.amount,
            deadline: permitData.deadline,
            v: permitData.v,
            s: permitData.s,
            r: permitData.r
        }) { } catch { }

>       return swapAndDeposit(tokenIn, permitData.amount, amountOutMin, routeCode);
```

A better way to code the try/catch block is as such:

```
try (permit) {
return;
} catch {
    if(IERC20(token).allowance(owner, spender) >= value) {
        return;
    }
} revert("permit failure");
}
```

1. Try the permit first
2. If the permit fails, check if there is still allowance for the spender
3. Otherwise, revert the whole function.
