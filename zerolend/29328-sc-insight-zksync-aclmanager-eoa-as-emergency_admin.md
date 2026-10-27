# 29328 - \[SC - Insight] zkSync ACLManager EOA as EMERGENCY\_ADMIN

Submitted on Mar 14th 2024 at 03:29:05 UTC by @mhmd\_alfa for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29328

Report type: Smart Contract

Report severity: Insight

Target: <https://explorer.zksync.io/address/0x9A60cce3da06d246b492931d2943A8F574e67389>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

An EOA 0x0f6e98a756a40dd050dc78959f45559f98d3289d has EMERGENCY\_ADMIN role. Leaked private key could lead to griefing and/or malicious halt of the protocol assets.

## Proof of Concept

1. go to <https://explorer.zksync.io/address/0x9A60cce3da06d246b492931d2943A8F574e67389#contract>
2. get EMERGENCY\_ADMIN\_ROLE = `0x5c91514091af31f62f596a314af7d5be40146b2f2355969392f055e12e0982fb`
3. inspect `hasRole(0x5c91514091af31f62f596a314af7d5be40146b2f2355969392f055e12e0982fb,0x0f6e98a756a40dd050dc78959f45559f98d3289d)` --> true
