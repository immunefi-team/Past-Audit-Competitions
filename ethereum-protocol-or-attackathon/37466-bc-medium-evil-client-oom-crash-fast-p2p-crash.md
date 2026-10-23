# #37466 \[BC-Medium] Evil-client OOM crash (fast P2P crash)

**Submitted on Dec 5th 2024 at 12:13:38 UTC by \[redacted] for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37466
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/NethermindEth/nethermind>
* **Impacts:**
  * Unintended chain split affecting greater than or equal to 25% of the network (Network partition)

## Description

## Brief/Intro

A new critical remote P2P crash vulnerability has been identified in Nethermind 1.29.1 (latest). If used in the wild it would result in netsplits - as is also seen in report #37120, although this crash is triggered significantly faster from a single attacking machine. To be precise - both attacks result in the same netsplits - with this attack just being a bit faster.

## Vulnerability Details

Requesting block receipts isn't sufficiently rate-limited. By handshaking into `nethermind`, which oddly doesn't require the same 1 IP per 30 seconds rule as `geth` does, completing the handshakes and peering into victim nodes; it is possible to quickly crash any machine running the `nethermind` process, regardless of the machine's hardware.

## Attack Scenario

An attacker would traverse the network to gather peer `enode://hash@ip:port`, and then weaponize a modified version of this attack script on a very, very small botnet - one that could be easily, and inexpensively built for less than $1000.

The script would be modified to target a list of public listening nodes as opposed to aiming it at a single test node, as we will be doing later in this report. It is as simple as the zombies running down the lists of online nodes running `nethermind`, crashing them inside of 30 - 60 seconds and moving onto the next, etc. in a simultaneous, scaled way that would certainly partition the network.

## Attack code

Given the size of the attack script, over 1,000 lines, also given that it's something of a light Ethereum client, we have securely uploaded it to a private location in our internal Dropbox: <https://www.dropbox.com/scl/fi/ulm9ql1o5hmcfvnrnhwin/receipts.go?rlkey=x1s7cexwa5nt4d91g9831q1yx\\&st=cx1cd8p6\\&dl=0> - the password to download the attack script is: `alienbrain`

## Steps to reproduce (Ubuntu)

1. Open `receipts.go` in a text editor on your attack machine and change `enodeURL := "enode://fefddae3a6fe5c04910ea0ffc7295969ebfac94cba0beaf07b8f0021020214852d24c5658cb4616f0d90d534870e1b28a052da9da2de5a920eec9dbd5112fe59@95.216.219.8:30303"` to your test victim node's `enode://hash@ip:port`
2. Install go: `snap install go --classic`
3. `ulimit -n 100000`
4. `go build receipts.go` - and run the dependency commands it outputs, if any - then run `go build receipts.go` again.
5. `./receipts`
6. Enter `50` threads - but feel free to experiment with this, and pauses, etc.

## Outro and patch suggestion

This is an advanced attack given that it replicates what the remote peer expects of a valid peer, but is actually an evil peer that attacks from within by spamming the receipts protocol message. A patch is to simply rate limit the message because there is no rationale behind it not being limited.

## Impact Details

Network partitioning affecting >25% of the network. A single attacker with an inconsequential amount of resources would likely short the markets and fracture the network.

## Proof of Concept

## Proof of Concept

We're attaching screenshots of the attack:

1. Before the attack
2. During the attack
3. After the attack

and again, we have also securely uploaded functioning attack code to a private location in our internal Dropbox: <https://www.dropbox.com/scl/fi/ulm9ql1o5hmcfvnrnhwin/receipts.go?rlkey=x1s7cexwa5nt4d91g9831q1yx\\&st=cx1cd8p6\\&dl=0>

Dropbox download password: `alienbrain`

Screenshots attached.
