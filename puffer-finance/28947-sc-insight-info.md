# 28947 - \[SC - Insight] Info

## Info

Submitted on Mar 2nd 2024 at 18:35:50 UTC by @OxJoyBoy03 for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28947

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Info

### Description

## NC Issues

### \[N-1] errors and events did not use!!!

#### Details

these errors and events were not used anywhere, you can remove or use them.

* Found in src/IPufferDepositor.sol [Line: 17](https://github.com/immunefi-team/Past-Audit-Competitions/blob/main/Puffer%20Finance/src/interface/IPufferDepositor.sol#L17)

  ```js
  //@audit-info This error is not used!!! 
  error TokenNotAllowed(address token);
  ```
* Found in src/IPufferDepositor.sol [Line: 28](https://github.com/immunefi-team/Past-Audit-Competitions/blob/main/Puffer%20Finance/src/interface/IPufferDepositor.sol#L28)

  ```js
  //@audit-info this event is not used!!!
  event TokenAllowed(IERC20 token);
  ```
* Found in src/IPufferDepositor.sol [Line: 33](https://github.com/immunefi-team/Past-Audit-Competitions/blob/main/Puffer%20Finance/src/interface/IPufferDepositor.sol#L33)

  ```js
  //@audit-info this event is not used!!!
  event TokenDisallowed(IERC20 token);

  ```

### Proof of Concept
