# 29332 - \[SC - Insight] Manta ReservesSetupHelper EOA as owner

Submitted on Mar 14th 2024 at 04:10:00 UTC by @mhmd\_alfa for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29332

Report type: Smart Contract

Report severity: Insight

Target: <https://pacific-explorer.manta.network/address/0xb8634e0a320d0f4861062514a63B659E52A87E21>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

An EOA 0x0f6e98a756a40dd050dc78959f45559f98d3289d is owner of the ReservesSetupHelper contract. Leaked private key could lead to a malicious actor tampering with the protocol parameters, such as supplyCap and borrowCap, which in turn coul ultimately lead to the whole protocol insolvency.

## Proof of Concept

1. go to <https://pacific-explorer.manta.network/address/0xb8634e0a320d0f4861062514a63b659e52a87e21?tab=read\\_contract>
2. inspect \_owner = 0x0f6e98a756a40dd050dc78959f45559f98d3289d --> EOA
