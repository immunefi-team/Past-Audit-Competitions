# #38275 \[BC-Low] Evil-client P2P headers-traversal leads to D/DoS and total peer removal

**Submitted on Dec 29th 2024 at 22:38:42 UTC by \[redacted] for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38275
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/hyperledger/besu>
* **Impacts:**
  * Shutdown of greater than or equal to 10% or equal to but less than 33% of network processing nodes without brute force actions, but does not shut down the network
  * Unintended chain split affecting less than 25% of the network (Network partition)
  * Shutdown of less than 10% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Intro

A remote P2P vulnerability has been identified in the Besu 24.12.2 (latest) with impact that is identical to that of a sustained D/DoS attack.

## Vulnerability Details

Using a multithreaded golang based evil-client that spoofs itself as a valid client and peers into a Besu node and starts spamming it with block header range protocol message requests, the remote machine running Besu's CPU runs to 97% - 100% - and all of its peers are remotely disconnected; (attached PoC screenshot) for as long as the attack is sustained. Fortunately the node does recover - but a botnet doesn't care about that. As a former prolific spammer, it's a pinky-lift for a lot of people, chiefly in Russia, to weaponize xxx,xxx zombies to launch an attack like this; and so the real-world viability is markedly there.

## Steps to reproduce (Ubuntu)

```
nano headers_traversal.go
change the existing enode//hash@ip:port to your own test machine
screen -S attack
ulimit -n 100000
snap install go --classic
go init 1
go mod tidy
go build headers-traversal.go
./headers-traversal
-- Experiment with thread counts, e.g. 15, 50, 100, 1000, 2500, etc.
Tap enter and wait..,
```

patiently - and watch it remotely disconnect the victim node from the (Sepolia) network by removing all of its peers. This is naturally a consequence of resource exhaustion and the node being rendered unable to process peer comms.

## Impact Details

Island networks. 16% of the Ethereum network being simultaneously disabled by a botnet would have significant consequences. We'd see netsplits, slower finality, transactions flying into the void and high reputational/operational damage.

## Proof of Concept

For Proof of Concepts we are linking to the relevant exploit code and providing 2 screenshots - of during and after the sustained attack.

Attack script (`headers_traversal.go`):\
<https://www.dropbox.com/scl/fi/mlis0t7cvl9m4weuwcqv8/headers\\_traversal.go?rlkey=5v5i2el1hzfc4tj26ibfwatb1\\&st=koknxe9h\\&dl=0>

Dropbox Password: `alienbrain`
