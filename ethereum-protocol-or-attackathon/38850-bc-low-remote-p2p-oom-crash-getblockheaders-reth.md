# #38850 \[BC-Low] Remote P2P OOM Crash (GetBlockHeaders) / Reth

**Submitted on Jan 15th 2025 at 17:12:20 UTC by \[redacted] for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38850
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/paradigmxyz/reth>
* **Impacts:**
  * Unintended chain split affecting less than 25% of the network (Network partition)

## Description

## Intro

A remote crash vulnerability has been identified in Reth v1.1.5 - 1 lightweight attacking machine can remotely crash Reth nodes regardless of server specs. If scaled it would result in the removal of all Reth nodes from the network and trigger partitioning.

## Vulnerability Details

By abusing the GetBlockHeaders P2P request - we can request large block ranges resulting in a fast OOM crash as the remote victim node tries to process the workload.

This is how we started Reth:

```
./reth node   --datadir=/data/storage/reth   --chain=sepolia   --http   --http.addr=0.0.0.0   --http.port=8545   --http.api=eth,net,web3,debug   --ws   --ws.addr=0.0.0.0   --ws.port=8546   --metrics=0.0.0.0:9001   --authrpc.addr=0.0.0.0   --authrpc.port=8551   --authrpc.jwtsecret=/data/storage/jwt.hex   --port=30303
```

and Lighthouse:

```
./lighthouse   --network sepolia   beacon   --datadir /data/storage/eth2/lighthouse   --disable-upnp   --listen-address=0.0.0.0   --port 9000   --discovery-port 9000   --http   --http-address 0.0.0.0   --http-port 5052   --execution-endpoint http://127.0.0.1:8551   --checkpoint-sync-url https://sepolia.beaconstate.info   --execution-jwt /data/storage/jwt.hex   --target-peers 100   --boot-nodes "enr:-KG4QMOEswP62yzDjSwWS4YEjtTZ5PO6r65CPqYBkgTTkrpaedQ8uEUo1uMALtJIvb2w_WWEVmg5yt1UAuK1ftxUU7QDhGV0aDKQu6TalgMAAAD__________4JpZIJ2NIJpcIQEnfA2iXNlY3AyNTZrMaEDfol8oLr6XJ7FsdAYE7lpJhKMls4G_v6qQOGKJUWGb_uDdGNwgiMog3VkcIIjKA,enr:-KG4QF4B5WrlFcRhUU6dZETwY5ZzAXnA0vGC__L1Kdw602nDZwXSTs5RFXFIFUnbQJmhNGVU6OIX7KVrCSTODsz1tK4DhGV0aDKQu6TalgMAAAD__________4JpZIJ2NIJpcIQExNYEiXNlY3AyNTZrMaECQmM9vp7KhaXhI-nqL_R0ovULLCFSFTa9CPPSdb1zPX6DdGNwgiMog3VkcIIjKA,enr:-Ku4QImhMc1z8yCiNJ1TyUxdcfNucje3BGwEHzodEZUan8PherEo4sF7pPHPSIB1NNuSg5fZy7qFsjmUKs2ea1Whi0EBh2F0dG5ldHOIAAAAAAAAAACEZXRoMpD1pf1CAAAAAP__________gmlkgnY0gmlwhBLf22SJc2VjcDI1NmsxoQOVphkDqal4QzPMksc5wnpuC3gvSC8AfbFOnZY_On34wIN1ZHCCIyg"   --debug-level info   --subscribe-all-subnets   --enable-private-discovery   --import-all-attestations
```

## Attack scenario

Bob operates a large botnet with the intent of fracturing the Ethereum network by disabling the vast majority of Reth nodes, effectively removing them from the network.

After installing the attack script from his botnet control center, it is instructed to remotely peer into each Reth node in the Ethereum network and strike from within using the multithreaded script.

Reth nodes are easily isolated and databased by hopping peers for each node's peer lists after handshaking into them to retrieve node User-Agents. This enables Bob the ability to tap start and remotely attack and subsequently crash every Reth node in the Ethereum network causing significant network partitioning.

## Attack code

`attack.go`: <https://www.dropbox.com/scl/fi/0kparo4vqdm54jdq3ha9m/attack.go?rlkey=8lcyxtz2w1abtvh295mfrtrwn\\&st=rjkivauj\\&dl=0>

Dropbox password: `alienbrain`

## Steps to reproduce (Ubuntu)

`snap install go --classic`

`ulimit -n 100000`

`nano attack.go`

Edit your `enode://hash@ip:port` into `attack.go` around line `264`

`go mod init 1`

`go mod tidy`

`go build attack.go`

`./attack`

Enter `5` threads, tap enter and monitor the resource consumption and ultimate crash of the process.

## The case for High severity (@Ethereum and Immunefi)

As the Ethereum Attackathon is following <https://immunefi.com/immunefi-vulnerability-severity-classification-system-v2-3/> lists "Unintended chain split (network partition)" as High - respectfully. I do understand the gravity of boosting the prize pool to 9, but it's time.

We've been co-researching and working hard under the assumption that the goalposts wouldn't budge given the established funding. We might need to have a multi-client multi-report discussion about network partitioning being listed as High - and networking issues, including DOS, being the 1st item in the L1 scope. We don't understand why thoroughly defined High severity has been violated with every engagement.

It's time to move our High reports into High severity because they are High in severity as stated, clearly, in the Attackathon rules. This isn't a technicality or social engineering attempt. Many of our reports have been erroneously marked as Insightful or Medium when they are, by any observant/logical account, High.

## Impact

The sudden disappearance of every Reth node in the Ethereum mainnet, by an attacker utilizing a manually built botnet or microbotnet, triggering mainnet partitioning.

## Proof of Concept

I have attached `before.png`, `during.png` and `after.png` - which are screenshots that illustrate both the attack in motion and the end result of it.

I am also, as also provided in the report, providing the attack code: <https://www.dropbox.com/scl/fi/0kparo4vqdm54jdq3ha9m/attack.go?rlkey=8lcyxtz2w1abtvh295mfrtrwn\\&st=rjkivauj\\&dl=0>

Dropbox password: `alienbrain`
