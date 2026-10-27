# Attackathon \_ Fuel Network 32730 - \[Smart Contract - Low] The Sway compiler currently disallows read

Submitted on Mon Jul 01 2024 01:21:53 GMT-0400 (Atlantic Standard Time) by @rbz for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32730

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* If contract A interacts with contract B and the interaction fails, causing contract B to revert, contract A may also revert as a consequence;

## Description

## Brief/Intro

The Sway compiler's current state Reverts any transaction that performs read access to storage if the call is made within the `fallback` function.

## Vulnerability Details

I adapted an existing collection of tests targetting `fallback` functionality (can found at <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59/test/src/sdk-harness/test\\_projects/run\\_external\\_proxy> and <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59/test/src/sdk-harness/test\\_projects/run\\_external\\_target>) to demonstrate an issue with the way compiler handles storage access in `fallback` function. While the compiler documentation suggests READ access to storage should be possible within fallback functions (since it's a common characteristic of any Sway function), the current state of the compiler prevents this. This difference between the documentation and the codebase's actual behavior raises doubts about whether fallback functions work as intended.

However, WRITE operations to storage within fallback functions are working as expected, indicating a potential inconsistency in how storage interactions are handled.

## Impact Details

* A malicious contract with a faulty fallback function could be deployed, deliberately hiding crucial storage access features within it. This could trick less technical users into thinking the contract is legitimate because it compiles successfully. This can potentially to griefing;
* If contract A interacts with contract B and the interaction fails, causing contract B to revert, contract A may also revert as a consequence;

## References

<https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/docs/book/src/blockchain-development/calling\\_contracts.md?plain=1#L174>

## Proof of concept

Please see following gists: <https://gist.github.com/0xZRA/c25f0ee9ef89d44adde573609c5553f3> <https://gist.github.com/0xZRA/19a9079f97f60a5143115e76a0eb84a9> <https://gist.github.com/0xZRA/18d8f25e6b3a97025de946868a230765> <https://gist.github.com/0xZRA/6e1684f538ad85dff5a39b25bb54523f> <https://gist.github.com/0xZRA/8875643f14e88092ac72e400079fd849>
