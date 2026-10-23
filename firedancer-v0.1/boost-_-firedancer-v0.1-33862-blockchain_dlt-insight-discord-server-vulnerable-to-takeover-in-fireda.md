# Boost \_ Firedancer v0.1 33862 - \[Blockchain\_DLT - Insight] Discord Server Vulnerable to Takeover in

Submitted on Wed Jul 31 2024 12:35:24 GMT-0400 (Atlantic Standard Time) by @swiss45 for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #33862

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/firedancer-io/firedancer/tree/e60d9a6206efaceac65a5a2c3a9e387a79d1d096>

Impacts:

* Informative - Discord server link takover

## Description

## Vulnerability Details

The official Discord link in the Firedancer repository is invalid and vulnerable to takeover. An attacker can create a custom Discord invite link for a different server and hijack the communication channel intended for Firedancer users. This issue does not directly impact the core functionality of the Firedancer program but poses a security risk by potentially misleading users to join a malicious Discord server.

## Impact:

The vulnerability allows attackers takeover of the Discord server through a custom invite link leads to redirection to malicious discord server.Users might be misled to join a malicious Discord server. This poses a security risk and may lead to unauthorized access or malicious activities.

## Fix:

To resolve this issue, it is recommended to update and validate the Discord server link

## Proof of concept

**Steps to Reproduce:**

1. Navigate to the Firedancer repository: [Firedancer Repository](https://github.com/firedancer-io/firedancer/tree/e60d9a6206efaceac65a5a2c3a9e387a79d1d096)
2. In the GitHub search bar, search for `discord`: [Search Results](https://github.com/search?q=repo%3Afiredancer-io%2Ffiredancer%20discord\&type=code)
3. Identify the Discord invite link: `https://discord.com/invite/7kr7VmPH`
4. Open the link you will see the takeover PoC (swiss server)

**Informative Note:**

### I am reporting this as an informative issue as it is not within the scope of impact. I reported this based on the security issue of the Discord server link being vulnerable to takeover.

### The status and severity of this issue are at your discretion. If you find this report unacceptable, please feel free to close it. I apologize if I have done anything wrong in reporting this issue. I will not make any further reports like this. If the report is closed, I will take down the PoC for the Discord server takeover.
