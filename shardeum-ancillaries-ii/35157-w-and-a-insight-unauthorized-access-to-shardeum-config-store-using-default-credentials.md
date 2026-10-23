# #35157 \[W\&A-Insight] Unauthorized Access to Shardeum Config Store using default credentials

**Submitted on Sep 7th 2024 at 16:30:22 UTC by @sujan\_shetty for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35157
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://immunefi.com>
* **Impacts:**
  * Unauthorized Access to Shardeum Config Store which leads to create,edit,delete configurations

## Description

## Vulnerability Details

I have found one endpoint <http://internal.network.shardeum.org/login> which use default credentials so that attacker can bypass auth and attacker can create,edit,delete Shardeum Config Store .

\##Steps to reproduce

1. Navigate to <http://internal.network.shardeum.org/login>
2. use below credentials username as admin password ad password
3. you will get access to internal Dashboard there you create,edit,delete the configurations

## Impact Details

attacker can bypass auth and attacker can create,edit,delete Shardeum Config Store .

## Proof of Concept

## Proof of Concept

screenshot is attached.
