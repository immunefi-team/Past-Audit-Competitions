# 31487 - \[SC - Low] Wrong condition check on RevenueHandlerconstruc...

Submitted on May 20th 2024 at 10:34:10 UTC by @cryptonoob2k for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31487

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The RevenueHandler constructor performs a wrong condition check allowing to break BPS maximum value (10\_000) implemented invariant and bypass maximum BPS value leading to set tresuary pct to arbitrary value so failing to fulfill expected restrictions set for example in RevenueHandler::setTreasuryPct

## Vulnerability Details

The vulnerability exists in RevenueHandler::constructor because wrongly checks the contract treasuryPct variable instead of constructor parameter value:

```js
constructor(address _veALCX, address _treasury, uint256 _treasuryPct) Ownable() {
    veALCX = _veALCX;
    require(_treasury != address(0), "treasury cannot be 0x0");
    treasury = _treasury; 
    require(treasuryPct <= BPS, "treasury pct too large"); //<@ Wrong check
    treasuryPct = _treasuryPct;
 }
```

So parameter `_treasuryPct` coult be an arbitrary value , instead of maximum 10\_000 value that is performed for eg in RevenueHandler::setTreasuryPct, and will be assigned as treasury pct.

## Impact Details

The impact of this vulnerability includes:

1. Bypass of maximum pct of treasury and breaking contract enviroment asumptions

### Risk Breakdown

The vulnerability is fairly easy to exploit leading to breaking BPS invariant, but can only occour during constructor.\
Also, the setted value can be modified using RevenueHandler::setTreasuryPct method.\
Overall Risk Severity: Low\
Likelihood: Medium\
Impact: Low

## References

<https://owasp.org/www-project-smart-contract-top-10/2023/en/src/SC07-logic-errors.html>

## Proof of Concept

To show the vulnerability\
Add this test on src/test/RevenueHandler.t.sol

```js
    function testTreasuryPctLimitBypass() external {
        RevenueHandler revenueHandler2;
        revenueHandler2 = new RevenueHandler(address(veALCX),admin,12345678987654321);
        console.log("revenueHandler2.treasuryPct ",revenueHandler2.treasuryPct());
    }
```

Run test with

```bash
reset ; forge compile  ; forge test --deny-warnings --fork-url https://rpc.ankr.com/eth --fork-block-number 17133822 --mt testTreasuryPctLimitBypass -vv
```

Observe revenueHandler2.treasuryPct is more than the intended max allowed (10\_000) and bypass restriction checks implemented in RevenueHandler::setTreasuryPct
