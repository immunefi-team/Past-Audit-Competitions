# #39871 \[BC-Critical] Lack of consensus voting in best cycle calculation allows a malicious validator to fake cycle data and crash all nodes

**Submitted on Feb 9th 2025 at 15:24:59 UTC by @throwing5tone7 for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39871
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardus-core/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

In the shardus core layer, nodes manage records of each cycle of the underlying chain that\
is used to manage the nodes / shards / etc. They then exchange certificates of\
these records with each other in order to reach an agreement over the current cycle state. Rather than using a voting\
(consensus) mechanism to agree, they use a complicated scoring system based on hashed values to decide which is the winning answer.\
This scoring system is gameable by an attacker, so a malicious validator can get other nodes to accept a bogus\
record for the latest cycle. The cycle record has security-critical data in it, like the status of all nodes, and also drives\
the timing of further cycle processing. By submitting a faked record with a bogus start time, I have demonstrated that\
a malicious validator can crash\
the network. However, other desirable attack outcomes may be possible by manipulating other data of the cycle record.

## Vulnerability Details

A cycle is split up into four quarters, with specific processing happening in each quarter.\
In the Q3 processing of a cycle, nodes update their own cycle record based on transactions they have received from\
other nodes (about the state of various nodes etc). Once they have done so, they engage in a complicated process\
whereby they exchange proposals of the current cycle record data and certificates of those proposed records with each other, trying to agree on the record that\
has the best "score". They will then accept whichever record has the best score as the starting point for the next cycle.

There are two parts to the scoring process:

1. Generate the score for an individual certificate, which is specific to a single node and single proposal for the record data
2. Sum these scores across nodes that have provided the same proposal

### Individual scoring

The score for a single certificate is generated as in <https://github.com/shardeum/core/blob/9dae0abe5232ed532a9285da82118b41a04b3711/src/p2p/CycleCreator.ts#L954C1-L965C2>

```
function scoreCert(cert: P2P.CycleCreatorTypes.CycleCert): number {
  try {
    const id = NodeList.byPubKey.get(cert.sign.owner).id // get node id from cert pub key
    const obj = { id }
    const hid = crypto.hash(obj) // Omar - use hash of id so the cert is not made by nodes that are near based on node id
    const out = utils.XOR(cert.marker, hid)
    return out
  } catch (err) {
    error('scoreCert ERR:', err)
    return 0
  }
}
```

Note that `cert.marker` is the hash of the cycle record being proposed. It can be seen that the score is just based\
on an XOR between a node's ID and the hash of the proposed data (`cert.marker`). This means that by arbitrarily changing\
a piece of the proposed data, you can attempt to generate a higher score for the certificate.

### Summing scores across nodes

There is additional protection provided by the fact that nodes sum all of the scores for the same proposal (after deduplicating\
by who sent the proposal). Essentially the nodes gossip their own certificates to each other when they think they\
have the best proposal, and if anybody else gossips a better proposal to them they will gossip it onwards also.\
This means that most of the time, there are around 3 individual scores being summed up to make the score for a proposal,\
according to my testing. This turns it into a pseudo-voting scheme, because the individual scores are always 4-byte numbers\
and in most cases when you choose s1, s2, s3 and s4 randomly over a 4-byte range, s1 + s2 + s3 > s4. But this is not\
guaranteed, which is why I call it a pseudo-voting scheme.

### Basic vulnerability

A malicious node can fake a high-scoring certificate, simply by generating some bogus record for the current cycle\
with data that they want (e.g. a far future start time in my PoC), and then keep changing the bogus record in ways\
that doesn't affect their attack (I add one second to the already bogus start time), generate the certificate score\
and see if it is above some threshold they choose. It is relatively quick to do this in such a way that you are basically\
guaranteed to find a record whose certificate score will be > 0xfff00000, which is bigger than the vast majority of\
random individual scores, and can sometimes (if the attacker gets lucky) be bigger than the a sum of 3 random scores.\
I have seen this happen in my testing, but it is rare.

### Exploit-enabling properties of the code

In general, summing together 3 random scores gives a 5-byte integer value, so the attacker can't always beat the best network\
agreed score and have their bogus record accepted.\
In order to guarantee getting their bogus record accepted, they can make\
use of two further properties of the cycle handling code:

1. Gossips of the best cycle are accepted when the other nodes aren't yet in Q3 - I suspect this is actually a bug
2. Once the node has a high scoring cycle, it won't gossip out it's own cycle record / certificate if that has a smaller score

Due to these properties the attacker can vastly improve their attack by:

1. Running their Q3 process slightly early during the cycle that they try to attack
2. Gossiping their bogus record and certificate to **all nodes** once it is generated - before any other node has generated it's own individual score

This ensures that all nodes have the high scoring bogus record, so unless the attacker gets very unlucky none of the other\
node's\
individual record scores will be higher, meaning no other cycle record proposals will be exchanged through the network,\
and the attacker's proposal will have the highest total score of any communicated, and be accepted.

## Impact Details

In my PoC I set the start of the cycle to be a bogus value that is greater than the maximum positive value that can\
be stored in a 4-byte signed integer. This causes all of the nodes in the network to crash almost immediately (within a\
minute or so normally), giving a total network shutdown.

Because the cycle record contains other security-relevant data, it is quite possible that other nefarious effects might\
be possible by faking the cycle (e.g. perhaps they can change the network to be only made of their own nodes), but I\
have not investigated this further.

## Link to Proof of Concept

<https://gist.github.com/throwin5tone7/d6101e59f2ee061d996f523e914d3011>

## Proof of Concept

## PoC Steps

The attack scenario is a malicious validator node that attacks the other nodes. It would in principle be possible\
to extract the attack logic into a separate script that just knows the attacker node keys, but there wasn't time\
to do this in the competition. Hence the PoC is provided as patches to the shardus core & shardeum codebases\
to turn a node malicious and automatically carry out the attack. The steps to demonstrate this are as follows.

### Configure a legit network

NOTE - I assume that the developers can do this another way - what is required is a network running non-modified code\
that the attacker's node can join (it therefore needs to at least allow connections from localhost IP addresses,\
hence the config patch).

The exact steps I used where:

* Get source code for real shardeum (<https://github.com/shardeum/shardeum>) into a folder of your choice, e.g. `LEGIT-Shardeum`
* Set an appropriate node version, e.g. `fnm use 18.19.1`
* Switch to the bugbounty tag, e.g. `git checkout bugbounty`
* Install dependencies using `npm ci`
* Apply the config to use a smaller net for testing, and to allow localhost network addresses for nodes & archivers`git apply legit-shardeum.patch`

### Patch a validator codebase to run the attack

* Download a fresh copy of shardus-core repo (<https://github.com/shardeum/shardus-core>) to a folder where you modify it, e.g. `HACKED-shardus-core`
  * Set an appropriate node version, e.g. `fnm use 18.19.1`
  * Switch to the bugbounty tag, e.g. `git checkout bugbounty`
  * Apply the patch to this `HACKED-shardus-core` folder - `git apply hacked-shardus-core.patch`
    * This has everything need to turn it into a malicious node that will automatically attack any cycle > 34
  * Run `npm install` to ensure it's all up-to-date with the patches and to fetch all dependencies
* Download a fresh copy of shardeum repo (<https://github.com/shardeum/shardeum>) to a folder where you modify it, e.g. `HACKED-shardeum`
  * Switch to the bugbounty tag, e.g. `git checkout bugbounty`
  * Apply the config to use a smaller net for testing, and to allow localhost network addresses for nodes & archivers - `git apply hacked-shardeum.patch`
    * Developers probably don't need to do this patching step, but just ensure that the config for this node matches the legit network
    * **NOTE**: along with the config changes, I found I needed a tiny code change to get the project to compile after shifting the dependency to the hacked shardus-core version (based off the bugbounty tag of shardus-core)
  * Set an appropriate node version, e.g. `fnm use 18.19.1`
  * Run `npm ci` to get the initial dependencies installed
  * Ensure that this hacked shardeum code correctly points to the HACKED-shardus-core folder:
    * e.g. `npm install ../HACKED-shardus-core`
    * **IMPORTANT**: this assumes that the hacked shardus-core folder can be found at path `../HACKED-shardus-core` - change the path if not
  * `npm run prepare` to get everything built & up to date
* Configure validator-cli to talk to local nodes and to run over hacked source (assuming the HACKED-shardeum folder has the relative path shown)
  * `ln -s "$(cd ../HACKED-shardeum && pwd)" ../validator` etc
  * Apply the patch to configure validator to talk to local networks `git apply validator-cli-config.patch`
    * Developers will probably apply slightly different config to get the connections to the legit network correct
  * Run `npm run compile` to get everything up to date and ready to launch

### Start up the legit nodes in a network

Devs might do this step differently, but to follow my exact steps, in the `LEGIT-Shardeum` folder launch a network that still has space for one node to join (the attacker's node)

* `shardus create-net 10 --no-start && shardus start --dir instances 9`

### Start up attacker node

In the validator-cli folder:

* Run `operator-cli start`

### Wait for attack

**IMPORTANT**: my PoC triggers when the cycle number is above 34. This is just used as a way to ensure that the network reaches\
a stable processing state and remains there for 15 cycles before the attack runs. If you don't want the attack to execute\
until a later cycle (maybe because your legitimate network is already past cycle 34 and you want to leave the attacker node\
to run normally at first), you should update the `MIN_CYCLE_FOR_ATTACK` constant in my shardus-core patch.

The PoC malicious validator does the following whenever cycle number > 34:

* Schedules it's own Q3 to run one second earlier than normal
* When this runs, take the current cycle record (legitimate at this point), and change `start` to be an unreachable number of seconds (`2147483648` which parses to a negative value when treated as a 4-byte value in some of the code) and then perform 4096 attempts to make a bogus cycle record:
  * Generate a score from the current bogus cycle record
  * If the score is bigger than 0xfff00000 return it as a successful fake record and high-scoring certificate
  * Else, increase the `start` value by one second and try again (up until 4096 attempts)
* If a high-scoring certificate on a bogus cycle record was successfully generated, gossip it to **all** nodes in the network
  * Else, just behave as normal and try again on the next cycle

### Effect of attack

When the attack runs, the nodes will stop processing cycles normally. In my testing, some of them crash almost instantly\
with an error like `unhandledRejection: Error: Received an invalid integer type: -68061988` (exact negative value will differ) and some of the crash within a minute or so later with `unhandledRejection: TypeError: Cannot read properties of undefined (reading 'shardGlobals')`
