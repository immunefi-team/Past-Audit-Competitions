# #36024 \[BC-Insight] Use of Vulnerable function results in prediction of archivers

**Submitted on Oct 16th 2024 at 03:43:21 UTC by @gladiator111 for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #36024
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/shardus-core/tree/dev>
* **Impacts:**
  * Shutdown of greater than 10% or equal to but less than 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

\`This is an insight report, since there is no option to submit an insight, I am submitting it under low impact, kindly downgrade to insight from low. Why it is an insight and not a higher severity is explained below in the report\`\
Use of Vulnerable function Math.random() results in prediction of which archiver will be selected.

## Vulnerability Details

In the following function\
\`\`\`javascript // @audit - Math.random() is predictable export function getRandomArchiver(): P2P.ArchiversTypes.JoinedArchiver | null { if (archivers.size === 0) return null const list = Array.from(archivers.values()) return list\[Math.floor(Math.random() \* list.length)] } \`\`\`\
This function is used for selecting archivers in the startup of nodes for sync of data. The function is supposed to select a random archiver for this purpose but the function Math.random() is cryptographically insecure. This is even mentioned on the official mozilla website :-

\`Note: Math.random() does not provide cryptographically secure random numbers. Do not use them for anything related to security. Use the Web Crypto API instead, and more precisely the Crypto.getRandomValues() method.\`

This function (\`getRandomArchiver\`) is used in crucial functions such as contactArchiver which is used for getting list of active nodes.

\`\`\`javascript export async function contactArchiver(dbgContex:string): Promise\<P2P.P2PTypes.Node\[]> { const maxRetries = 10 let retry = maxRetries const failArchivers: string\[] = \[] let archiver: P2P.SyncTypes.ActiveNode let activeNodesSigned: P2P.P2PTypes.SignedObject\<SeedNodesList>

info(\`contactArchiver: enter archivers:${getNumArchivers()}\`)

while (retry > 0) { try { retry-- @> archiver = getRandomAvailableArchiver() // used here info(\`contactArchiver: communicate with:${archiver?.ip}\`)

```
  if (!failArchivers.includes(archiver.ip + &#x27;:&#x27; + archiver.port)){
    failArchivers.push(archiver.ip + &#x27;:&#x27; + archiver.port)
  }

  activeNodesSigned &#x3D; await getActiveNodesFromArchiver(archiver)
```

\`\`\`

## Why it is not higher severity

From the shardeum documentation it can be found out that :-

\`Archiver staking, reward and slashing - The archiver nodes will initially be run by the team. Future upgrades will require archivers to also stake and earn rewards as well as lose stake if slashed. Once this is implemented the community will also be able to run archiver nodes.\`

The archivers are not currently a source of problem but in the future community will be given the role which is why I placed it as an insight.

## Impact Details

Archivers can be predicted which completely destroys the randomness. As Archivers play a crucial role, their prediction is disasterous. One example is below :-

->Malicious archiver provides wrong list of malicious nodes for syncCycleChain function.\
->Syncing of data is completely manipulated for other nodes.

## Recommendation

As mentioned in the documentation use Crypto: getRandomValues() method instead.

## References

<https://github.com/shardeum/shardus-core/blob/23e06ded6744d8521cff9d749c1f1dd482c5fcb6/src/p2p/Archivers.ts#L88> <https://github.com/shardeum/shardus-core/blob/cf9dbaf32327fd2a985c941132738d76660d1ff2/src/p2p/Self.ts#L855>

## Proof of Concept

## Proof of Concept

Since the vulnerability is an insight and the bug is fairly simple and well known, I am pasting some links of how the vulnerability (Math.random()) can be exploited.\
<https://github.com/PwnFunction/v8-randomness-predictor\\>
This is an excellent repository exploiting Math.random with full guide.

Youtube video - <https://www.youtube.com/watch?v=-h\\_rj2-HP2E>
