# #36005 \[W\&A-Insight] Reflected URL Manipulation and Phishing Risk

**Submitted on Oct 15th 2024 at 16:28:02 UTC by @Ouabala for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #36005
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/validator-cli/tree/dev>
* **Impacts:**
  * Redirecting users to malicious websites (open redirect)

## Description

\#Summary:

A vulnerability has been identified in the Shardeum Core interface where attackers can manipulate URLs in displayed links by altering the ip parameter in the following endpoint:

\`<http://localhost:3000/log?ip=example.com\\&port=3000\\`>

This vulnerability can still lead to phishing attacks by misleading users into clicking external links that they believe to be part of the Shardeum interface.

POC LInk --> <http://localhost:3000/log?ip=example.com\\&port=3000>

## Link to Proof of Concept

<https://gist.githubusercontent.com/ShellInjector/54c0b8d269d884091ff13abbfca18ae3/raw/f047868a35f144ecfd3af7d4c138ff00c19c32b1/gistfile1.txt>

## Proof of Concept

## Proof of Concept

<http://localhost:3000/log?ip=example.com\\&port=3000>
