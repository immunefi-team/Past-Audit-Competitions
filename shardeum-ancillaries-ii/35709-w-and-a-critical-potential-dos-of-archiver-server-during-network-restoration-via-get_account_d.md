# #35709 \[W\&A-Critical] Potential DoS of archiver-server during network restoration via get\_account\_da

**Submitted on Oct 4th 2024 at 08:45:54 UTC by @periniondon630 for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35709
* **Report Type:** Websites and Applications
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/archive-server/tree/dev>
* **Impacts:**
  * Taking down the application/website

## Description

## Brief/Intro

There is no limit on the number of accounts that a validator can request from the archiver during network restoration, which can lead to a DoS attack on the archiver server.

## Vulnerability Details

The get\_account\_data\_archiver API call is used by validators during the network restoration phase to sync new validators from archivers. However, there is no limit set on the number of accounts (the maxRecords parameter) that can be requested. If the network contains billions of accounts, an attacker with an active validator could request all of them in a single small request. This would overload the archiver, which would need to fetch the data from the database, serialize the result, and send it back to the validator, leading to potential performance issues or a DoS attack. This could cause serious problems during network restoration, as other new validators would be unable to sync with the network and start processing transactions.

## Impact Details

The archiver server is vulnerable to a potential DoS attack during network restoration, which could cripple the network’s ability to process transactions, effectively paralyzing its operations.

## References

<https://github.com/shardeum/archive-server/blob/0337daa477b3a30f8fb65b87c23b021a261441bd/src/API.ts#L927>

## Link to Proof of Concept

<https://gist.github.com/periniondon630/e0bb113b033710c9e9ef677c8a0082b0>

## Proof of Concept

1. Apply the \`'bugbounty'\` API call patch from the documentation to the attacker's validator and wait for it to become active.
2. Put the network into the restoration phase.
3. Run \`poc.js\` from the gist using the following command: \`\`\`bash node poc.js ATTACKER\_ID:ATTACKER\_PORT \`\`\` To simplify the test, you can remove the currentNetworkMode !== 'restore' condition in the validateAccountDataRequest function. Additionally, you need to create a large number of accounts to simulate a real network—this can be done using a modified poc.js script from the documentation.
