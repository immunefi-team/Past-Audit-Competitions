# 29329 - \[SC - Insight] Manta ACLManager EOA as EMERGENCY\_ADMIN

Submitted on Mar 14th 2024 at 03:33:09 UTC by @mhmd\_alfa for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29329

Report type: Smart Contract

Report severity: Insight

Target: <https://pacific-explorer.manta.network/address/0xb2178109A414C3a869E5104283Fcf1a18923D0B8>

Impacts:

* Temporary freezing of funds for at least 1 hour
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

An EOA 0x0f6e98a756a40dd050dc78959f45559f98d3289d has EMERGENCY\_ADMIN role. Leaked private key could lead to griefing and/or malicious halt of the protocol assets.

## Proof of Concept

1. go to <https://pacific-explorer.manta.network/address/0xb2178109a414c3a869e5104283fcf1a18923d0b8?tab=read\\_contract>
2. get EMERGENCY\_ADMIN\_ROLE = `0x5c91514091af31f62f596a314af7d5be40146b2f2355969392f055e12e0982fb`
3. inspect `hasRole(0x5c91514091af31f62f596a314af7d5be40146b2f2355969392f055e12e0982fb,0x0f6e98a756a40dd050dc78959f45559f98d3289d)` --> true
