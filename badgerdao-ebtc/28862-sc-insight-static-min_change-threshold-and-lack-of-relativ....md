# 28862 - \[SC - Insight] Static MIN\_CHANGE threshold and lack of relativ...

Submitted on Feb 29th 2024 at 04:53:58 UTC by @cheatcode for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28862

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/BorrowerOperations.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Vulnerability Details

The \_requireZeroOrMinAdjustment function enforces debt/collateral changes to either be 0 or >= MIN\_CHANGE, currently set to 1000 wei.

```solidity
function _requireZeroOrMinAdjustment(uint256 _change) internal pure {
  require(
    _change == 0 || _change >= MIN_CHANGE,
    "BorrowerOperations: Collateral or debt change must be zero or above min"
  );
}
```

Over time as EBTC loses value relative to ETH, this 1000 wei minimum could become negligible compared to typical CDP sizes. This could allow borrowers to leave tiny "dust" amounts of debt/collateral in their CDPs.

## Impact

Tracking and closing out these tiny debt amounts has several disadvantages:

1. It wastes gas to perform operations on insignificant debt values
2. It creates unnecessary storage updates when dust debt is adjusted
3. It clutters the accounting of CDPs and the overall system
4. It can make liquidations more difficult when negligible amounts are left in CDPs

## Proof of concept

Reference: <https://forum.badger.finance/t/ebtc-builder-update-january/6145/1>

> Additionally, there is 1000 wei units minimum change of debt and a 1000 wei minimum change to collateral on user operations.

Allowing tiny debt amounts could open the door for griefing attacks:

* An attacker repeatedly borrows "dust" amounts from CDPs, forcing the system to track and close out useless micro-debts. This wastes gas.
* Small amounts are left in CDPs about to be liquidated, requiring a more complex liquidation process to handle the dust.
* Attackers could open CDPs with the minimum debt, wasting storage tracking barely used CDPs.

By leaving `MIN_CHANGE` static, any amount >= 1000 wei is permitted, even if negligible in dollar terms. There is also no check preventing tiny adjustments relative to total CDP amounts.
