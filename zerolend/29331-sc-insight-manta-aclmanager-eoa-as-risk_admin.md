# 29331 - \[SC - Insight] Manta ACLManager EOA as RISK\_ADMIN

Submitted on Mar 14th 2024 at 03:49:33 UTC by @mhmd\_alfa for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29331

Report type: Smart Contract

Report severity: Insight

Target: <https://explorer.zksync.io/address/0x9A60cce3da06d246b492931d2943A8F574e67389>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

An EOA 0x0f6e98a756a40dd050dc78959f45559f98d3289d has RISK\_ADMIN role. Leaked private key could lead to a malicious actor tampering with the protocol parameters, such as supplyCap and borrowCap, which in turn coul ultimately lead to the whole protocol insolvency.

## References

<https://docs.aave.com/developers/core-contracts/aclmanager> `RISK_ADMIN` row

## Proof of Concept

1. go to <https://pacific-explorer.manta.network/address/0xb2178109a414c3a869e5104283fcf1a18923d0b8?tab=read\\_contract>
2. get EMERGENCY\_ADMIN\_ROLE = 0x8aa855a911518ecfbe5bc3088c8f3dda7badf130faaf8ace33fdc33828e18167 inspect hasRole(0x8aa855a911518ecfbe5bc3088c8f3dda7badf130faaf8ace33fdc33828e18167,0x0f6e98a756a40dd050dc78959f45559f98d3289d) --> true
