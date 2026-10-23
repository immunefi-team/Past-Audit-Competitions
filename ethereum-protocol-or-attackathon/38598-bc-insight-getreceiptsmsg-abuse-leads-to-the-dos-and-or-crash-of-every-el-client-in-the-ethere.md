# #38598 \[BC-Insight] GetReceiptsMsg abuse leads to the DoS and/or crash of every EL client in the Ethereum network

**Submitted on Jan 7th 2025 at 13:56:46 UTC by \[redacted] for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38598
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ethereum/go-ethereum>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)
  * Unintended chain split affecting greater than or equal to 25% of the network (Network partition)
  * All EL clients vulnerable

## Description

## Preamble

This report has been marked as **High** as it can be adjusted, but these would be classified as **Critical** elsewhere in the industry. I'm not sure why that hasn't been there case here. I've found \~20 exploits in various Ethereum CL/ES clients that haven't been submitted. I'll sweep this contest by the rules laid out in the terms of engagement if you let me. There are just so many to report and I don't know if there's enough time.

For practicality's sake we could establish TG DM or group comms for TL;DR reports and quick patches to circumvent time constraints and the (I don't know why) necessity of massive organically written reports on Immunefi.

This is probably a total network shutdown, or such a significant partial-network shutdown that could theoretically lead to hard fork or an easy 51% attack. One technique to double spend is to send a tx to an exchange, cash out, roll back the state of the blockchain, send it again, etc. and this is made easy when the network is barely if at all talking to itself.

I already kicked the prize pool from 250 to 500 but it should've been 900 to 1.5 based on the Attackathon's rules that follow Immunefi's v2.3 classification guidelines. Looking to bump it higher with this report.

I'm prepared to surgically dismantle any technical argument about this attack and its efficiency if necessary, but considering reward payouts are guaranteed and funded - I don't anticipate needing to social engineer anyone into fairness.

Security exploits are like math. It's either 1 + 1 = 2 or bullsh\*t. I've provided functioning attack code for Geth, Nethermind and Besu in this report (albeit independently and not in a unified attack script) below. They work against synchronized clients.

Use them.

## Intro

Multiple remote OOM crash, > 100% machine CPU and MEM DoS vulnerabilities have been identified in the latest releases of Geth, Nethermind and Besu.

I can find others in the lesser known clients (5% of network) if necessary; but it is unnecessary given that we would encounter near-identical undesirable behavior in smaller EL clients. I haven't looked at the < %5 clients yet - but, again, they will fall to remote exploitation if anyone here deems it necessary.

This report contains attack code for Geth, Nethermind and Besu in their own individual evil clients that pull heavily asymmetric receipts from blocks with 1k+ txs, or through requesting large block header ranges. There are a lot of vulnerabilities in these clients and the Attackathon doesn't give me enough time to find them all (<https://i.imgur.com/JarudSi.png>) and write massive reports for them - I have found a lot.

## Vulnerability Details

**It would be easier to TL;DR the exploits.** - as to not insult the intelligence of Ethereum programmers and to write fewer massive, organically written reports.

**This report could be a single sentence. "Apply rate limiting to the receipts protocol message to thwart botnet attacks".** and they'd **instantly** understand exactly what I mean and how to patch.

The protocol message `GetReceiptsMsg` isn't rate-limited in any Ethereum client. Naughty low-level P2P attack clients can peer in and DoS/crash from the inside. This is the case with Nethermind, Geth and Besu - and although Geth is the only slow OOM - it jams the EL up and **halts P2P comms between the EL/CS layers before even OOM crashing**; timing out, e.g. Teku running Geth during a remote attack from 1 to 5 attacking machines depending on machine specs on both ends:

```
00:01:56.456 WARN  - Execution Client request timed out. Make sure the Execution Client is online and can respond to requests.
00:01:56.457 WARN  - Late Block Import *** Block: 465c74f22e13da2781a07f2469591ef10133dd697a12de32d0657774a2a446ce (6706209) Proposer: 421 Result: failed_execution_payload_execution Timings: arrival 370ms, gossip_validation +61ms, pre-state_retrieved +5ms, processed +20ms, data_availability_checked +5ms, execution_payload_result_received +7995ms, begin_importing +1ms, completed +0ms
00:02:00.755 INFO  - Execution Client is responding to requests again after a previous failure
00:02:04.002 INFO  - Slot Event  *** Slot: 6706210, Block: 0667bde308c1ba8c45c397378ccdf7a6570fce235a416c2ffdbf9058fb0d2502, Justified: 209568, Finalized: 209567, Peers: 100
00:02:16.002 INFO  - Slot Event  *** Slot: 6706211, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 100
00:02:20.423 WARN  - Execution Client request timed out. Make sure the Execution Client is online and can respond to requests.
00:02:20.440 WARN  - Late Block Import *** Block: e2b0a7b5db262732b62c466ee94360bca9ce0415e7c17b3930a03392e90618fd (6706211) Proposer: 284 Result: failed_execution_payload_execution Timings: arrival 335ms, gossip_validation +46ms, pre-state_retrieved +5ms, processed +21ms, data_availability_checked +0ms, execution_payload_result_received +8016ms, begin_importing +9ms, completed +7ms
00:02:27.502 INFO  - Execution Client is responding to requests again after a previous failure
00:02:28.002 INFO  - Slot Event  *** Slot: 6706212, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 100
00:02:40.004 INFO  - Slot Event  *** Slot: 6706213, Block: 1cec0b9f007c230cb637b954f31dd9da34c5339dc98782f81ddb306f27072a9d, Justified: 209568, Finalized: 209567, Peers: 100
00:02:52.003 INFO  - Slot Event  *** Slot: 6706214, Block: 9180388750c6941929d9e6f73332811d35499d043f259714a977071a5a5a9532, Justified: 209568, Finalized: 209567, Peers: 100
00:03:04.002 INFO  - Slot Event  *** Slot: 6706215, Block: c59df03bdf9a0c9dc0752047569620a6b7d62f76c9179bdfb000d37293b55c72, Justified: 209568, Finalized: 209567, Peers: 100
00:03:16.001 INFO  - Slot Event  *** Slot: 6706216, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 100
00:03:20.498 WARN  - Execution Client request timed out. Make sure the Execution Client is online and can respond to requests.
00:03:20.500 WARN  - Late Block Import *** Block: 8514d9e3b5ee816a17a8344f3ddb94c8beb30665ba4b316cce707b1189ef6e7c (6706216) Proposer: 1827 Result: failed_execution_payload_execution Timings: arrival 426ms, gossip_validation +42ms, pre-state_retrieved +9ms, processed +19ms, data_availability_checked +0ms, execution_payload_result_received +8003ms, begin_importing +0ms, completed +0ms
00:03:22.829 INFO  - Execution Client is responding to requests again after a previous failure
00:03:28.002 INFO  - Slot Event  *** Slot: 6706217, Block: 9e4345a9c06a1add16d2408a760eeb62ab33048ee399eea3fe3d6cab3d2f3c41, Justified: 209568, Finalized: 209567, Peers: 100
00:03:40.050 INFO  - Slot Event  *** Slot: 6706218, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 100
00:03:44.660 WARN  - Execution Client request timed out. Make sure the Execution Client is online and can respond to requests.
00:03:44.661 WARN  - Late Block Import *** Block: f990263536d163603acb693387d0bf6d165d67e183bcef49764c63c24af186fd (6706218) Proposer: 1464 Result: failed_execution_payload_execution Timings: arrival 577ms, gossip_validation +57ms, pre-state_retrieved +5ms, processed +17ms, data_availability_checked +37ms, execution_payload_result_received +7967ms, begin_importing +1ms, completed +0ms
00:03:51.500 INFO  - Execution Client is responding to requests again after a previous failure
00:03:52.004 INFO  - Slot Event  *** Slot: 6706219, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 100
00:04:04.001 INFO  - Slot Event  *** Slot: 6706220, Block: bbcd6fd4f662315a71c498ce5d9f7bb9b194c0533e322ea6fe1cdebbc51ac9c1, Justified: 209568, Finalized: 209567, Peers: 100
00:04:16.003 INFO  - Slot Event  *** Slot: 6706221, Block: 2c2f6201cbda10ff175ac983f408317e3cd48591a90964f2982afc9725b9262d, Justified: 209568, Finalized: 209567, Peers: 100
00:04:28.002 INFO  - Slot Event  *** Slot: 6706222, Block: ad020f7e3ebf8d17118558bcd1679486e62b755e677753de6139ce58495c64e4, Justified: 209568, Finalized: 209567, Peers: 100
00:04:40.007 INFO  - Slot Event  *** Slot: 6706223, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 100
00:04:44.587 WARN  - Execution Client request timed out. Make sure the Execution Client is online and can respond to requests.
00:04:44.589 WARN  - Late Block Import *** Block: 737225d470da18449f137c14c736e4a6dea2c02e94392a43fbc09cfbcd01d6d6 (6706223) Proposer: 1400 Result: failed_execution_payload_execution Timings: arrival 499ms, gossip_validation +65ms, pre-state_retrieved +4ms, processed +19ms, data_availability_checked +7ms, execution_payload_result_received +7994ms, begin_importing +0ms, completed +0ms
00:04:49.148 INFO  - Execution Client is responding to requests again after a previous failure
00:04:52.003 INFO  - Slot Event  *** Slot: 6706224, Block: 06f9d98b7ed0cd0a9c9fbb7383cc80db6596d0f7c6e3c6236e01f0abaa8d1ab8, Justified: 209568, Finalized: 209567, Peers: 100
00:05:04.007 INFO  - Slot Event  *** Slot: 6706225, Block: 1cf217b00b0fa23adca2f6c20a2887ad98c3f8544251b4c88f37ccb9a7491249, Justified: 209568, Finalized: 209567, Peers: 100
00:05:16.002 INFO  - Slot Event  *** Slot: 6706226, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 100
00:05:18.639 WARN  - Late Block Import *** Block: 1577bbc8fdf330dc8047eded7545330e7d8ac01d2a85d10b6ad4bdb294a9b484 (6706226) Proposer: 471 Result: success Timings: arrival 333ms, gossip_validation +42ms, pre-state_retrieved +8ms, processed +16ms, data_availability_checked +8ms, execution_payload_result_received +6231ms, begin_importing +0ms, transaction_prepared +0ms, transaction_committed +0ms, completed +1ms
00:05:28.002 INFO  - Slot Event  *** Slot: 6706227, Block: b85b0a72e5c2163496280a98105179bb0a25a97820341447b8b498392bb1650d, Justified: 209568, Finalized: 209567, Peers: 100
00:05:40.002 INFO  - Slot Event  *** Slot: 6706228, Block: 240cc1125d76a8c26105e721042e94373cf4c15061276b464ef8366c5b00ce25, Justified: 209568, Finalized: 209567, Peers: 100
00:05:52.001 INFO  - Slot Event  *** Slot: 6706229, Block: 6453d99f294045c2d6134924595f72a6522ecb695971ae7a18a72adf1d29925c, Justified: 209568, Finalized: 209567, Peers: 100
00:06:04.002 INFO  - Slot Event  *** Slot: 6706230, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 99
00:06:08.506 WARN  - Execution Client request timed out. Make sure the Execution Client is online and can respond to requests.
00:06:08.507 WARN  - Late Block Import *** Block: 4ffce3f0ad84a7513c480440273ffa1cd96069e6451e2569457b6d8314782fa4 (6706230) Proposer: 1447 Result: failed_execution_payload_execution Timings: arrival 419ms, gossip_validation +50ms, pre-state_retrieved +4ms, processed +25ms, data_availability_checked +13ms, execution_payload_result_received +7996ms, begin_importing +0ms, completed +0ms
00:06:16.001 INFO  - Slot Event  *** Slot: 6706231, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 100
00:06:22.798 INFO  - Execution Client is responding to requests again after a previous failure
00:06:28.003 INFO  - Slot Event  *** Slot: 6706232, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 100
00:06:32.762 WARN  - Execution Client request timed out. Make sure the Execution Client is online and can respond to requests.
00:06:32.763 WARN  - Late Block Import *** Block: c959d862d7c00a25a0f3569c15c7d95298b9ed6e642ea170da9c15a21030ea39 (6706232) Proposer: 1154 Result: failed_execution_payload_execution Timings: arrival 639ms, gossip_validation +71ms, pre-state_retrieved +5ms, processed +33ms, data_availability_checked +8ms, execution_payload_result_received +8007ms, begin_importing +0ms, completed +0ms
00:06:40.003 INFO  - Slot Event  *** Slot: 6706233, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 99
00:06:40.303 INFO  - Execution Client is responding to requests again after a previous failure
00:06:52.002 INFO  - Slot Event  *** Slot: 6706234, Block: 755ea9c6c160189b6d65f1d1efbb08de463e3edc41ae0ea9f731df5bb5dc096d, Justified: 209568, Finalized: 209567, Peers: 100
00:07:04.001 INFO  - Slot Event  *** Slot: 6706235, Block: 88ba6ca3396b93635f866ebc0238f33dd0857e8d7a39708abff4d244f69a0752, Justified: 209568, Finalized: 209567, Peers: 100
00:07:16.004 INFO  - Slot Event  *** Slot: 6706236, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 100
00:07:18.484 WARN  - Late Block Import *** Block: 2c52b90ad12c1a737f3e98a0114c6f218e8933d527f8fa3484adf4aee301e577 (6706236) Proposer: 979 Result: success Timings: arrival 1288ms, gossip_validation +25ms, pre-state_retrieved +12ms, processed +22ms, data_availability_checked +45ms, execution_payload_result_received +5091ms, begin_importing +0ms, transaction_prepared +0ms, transaction_committed +0ms, completed +1ms
00:07:28.001 INFO  - Slot Event  *** Slot: 6706237, Block: 5184745588cba766ddcd6255573a8a8e1e1aea772a6a5d57ba9c4fb33952c53e, Justified: 209568, Finalized: 209567, Peers: 100
00:07:40.001 INFO  - Slot Event  *** Slot: 6706238, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 100
00:07:44.469 WARN  - Execution Client request timed out. Make sure the Execution Client is online and can respond to requests.
00:07:44.474 WARN  - Late Block Import *** Block: cbf76210ab6b4c1ab46df32fb34972fd49ed6d6d42519617980091d9f2010dca (6706238) Proposer: 1569 Result: failed_execution_payload_execution Timings: arrival 397ms, gossip_validation +48ms, pre-state_retrieved +8ms, processed +15ms, data_availability_checked +0ms, execution_payload_result_received +8002ms, begin_importing +0ms, completed +4ms
00:07:52.002 INFO  - Slot Event  *** Slot: 6706239, Block:                                                        ... empty, Justified: 209568, Finalized: 209567, Peers: 100
00:07:54.114 INFO  - Execution Client is responding to requests again after a previous failure
```

## Attack scenario

Bob operates alongside a person or group that oversees sizable botnets, e.g. spam cartel actors, state actors, etc. with xxx,xxx zombies ready to nuke the network by slamming every public listening node with messages that result in network desynchronization, CPU/MEM DoS, crashes and unprecedented netsplits.

He weaponizes a modern P2P botnet that runs the attacks with a modified, unified attack script built specifically to remotely disable the Ethereum network that is able to communicate the state of the attack to itself.

Bob's attack nodes start peering into victim Geth nodes with peer slots available. From there he sends FINDNODE to handshake-ping for UA across every (exponentially) collected peer. Every Nethermind and Besu UA is returned, databased and passed into another routine for quick-crashing them in the future as they crash a lot faster than Geth does. The quick-crashing frees up \~55% of peer slots in Geth nodes, on average, and allow for more attack peers to seed in - slowly - waiting for the command to attack the network simultaneously - at scale, and from the inside while being interpreted as valid heathy peers handling PING/PONG/STATUS/HEADERS/etc. as keep-alive mechanisms.

> Much of the network is comprised of Docker instances that reboot clients after they've crashed

At this point - we have a peer list across the network known by distributed evil-client based on their user-agents. These are databased in real-time and are useful for our smartnet's resource distribution.

By running a full attack the only time condition to consider is whether the speed at which the Docker reboots and Geth lag can save the network from a simultaneous DDoS attack sourced by a formidable botnet.

Different machines have different specs, and so will not crash or disconnect from the network at the exact same time - when all threads attacking a node are rejected due to the Ethereum node having been killed, the workload is distributed to immediately focus on, e.g. 32 GB MEM machines versus 16 GB MEM machines that crash twice as fast. The speed at which nodes stress is communicated to the zombies and auto-scaled to redirect or use new resources on peers as they become totally unresponsive, creating network-wide splits, lost funds flying into the void, opens the door to 51% attacks, and so on.

Again, each `enode://[hash]@ip:port` would have its clients identified by the smart botnet, in addition to the speed at which they become disconnected from the network - something discoverable by evil peers working in concert. The end result is hit or miss finality, funds presumably flying into the void in addition to other undesirable behavior.

The `geth` crash takes awhile to OOM on a 16 GB MEM machine and the same with even heavier MEM machines. That said - again - before the `geth` process quits it's already been desynchronized from the network.

Header block range spamming also appears to work against all clients - but all of my scripts can be easily modified to use any other DoS or crash that I've found. I have too many vulnerabilities and not enough time to write reports. I found too many vulns after familiarizing myself with the Ethereum software ecosystem.

## Impact Details

51% attacks, total or partial network shutdown, funds lost during failed txs, island networks - netsplits **everywhere**, operational damage and even after applying a hotfix - reputational damage in permeance.

## Attack code (golang, Ubuntu)

Below in Proof of concept, but we'll place it here too:

```
Geth, Nethermind and Besu attack code: https://www.dropbox.com/scl/fo/vh52nqhqreqrrmvzkk4hd/AC3jGVyfNWOl5dPRdm_g8X0?rlkey=8s0n14a9moq427w1hoolh65jv&st=elis076n&dl=0 

Dropbox password: `alienbrain`
```

## Technical description of what the attack programs do

With Besu, block header ranges are spammed to trigger an OOM. With Nethermind, receipts from large blocks are requested in a quick way to produce the fastest OOM and/or CPU and/or desync. Geth also requests receipts from these big blocks (1200+ tx) as Nethermind does.

## Steps to reproduce the attack

1. open a .go attack file (below) in a text editor and change the `enode://hash@ip:port` to your test victim's `enode://hash@ip:port`
2. screen -S attack
3. ulimit -n 10000
4. snap install go --classic
5. nano attack.go
6. paste from text editor
7. save as 'attack.go'
8. `go mod init 1`
9. `go mod tidy`
10. `go build attack.go`
11. `./attack`

This will not work if all peer slots are in use on a node, which is why we hop other nodes and after UA checking each peer in the network - our botnet attacks accordingly - based on client and later, crashing Nethermind and Besu clients out of Geth's peer list to clear the way for beefier high CPU and port speed machines in the smart botnet.

## Outro

I could do this solo by renting a botnet. I know a lot of people think "Ethereum is time and battle tested!" - womp. The problem here is that I haven't, again, had time to find them all - and there are many asymmetric resource-burning networking issues in every client that I've touched.

Given the complexity of this attack - I almost definitely forgot to include at least one item of importance - as there are a lot of moving parts here. But again - a very easy patch - and I'm here to provide clarity on what I've written in addition to what I may have forgotten to write. Apologies for the wordy report - but I think they're required by Immunefi for reasons unknown. I don't think presentation should be so important.

**TL;DR** Add rate-limiting to block receipt requests. Blocks containing a lot of data hasten the attack.

Thank you all for this opportunity to earn crypto and to help secure the web3 ecosystem. If you run into difficulty running the scripts or reproducing anything I'm here to assist.

AdvancedElephant

🧙🐘

## Proof of Concept

**PoC**: Screenshots and 3 attack scripts for Geth, Besu and Nethermind:

<https://www.dropbox.com/scl/fo/vh52nqhqreqrrmvzkk4hd/AC3jGVyfNWOl5dPRdm\\_g8X0?rlkey=8s0n14a9moq427w1hoolh65jv\\&st=elis076n\\&dl=0>

**Dropbox password**: `alienbrain`

The steps to reproduce running the scripts are in the report. Run them against their associated, fully synchronized EL clients.
