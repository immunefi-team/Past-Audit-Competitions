# #39980 \[W\&A-Critical] Malicious validator can inject its own cycle record into connected archiver

**Submitted on Feb 12th 2025 at 03:17:27 UTC by @periniondon630 for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39980
* **Report Type:** Websites and Applications
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/archive-server/tree/itn4>
* **Impacts:**
  * Taking and/modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as:
* Changing registration information
* Commenting
* Voting
* Making trades
* Withdrawals, etc.

## Description

## Brief/Intro

The archiver relies on the last or, at most, the fifth received cycle record (depending on network size) as the authoritative source for the network structure.

## Vulnerability Details

The archiver receives network structure information from two sources:

1. **Validators** it is directly connected to via socket connections.
2. **Adjacent archivers** through the `gossip-data` call.

Both sources invoke the `collectCycleData` function, which aggregates all received cycle records and verifies whether the minimum confirmation threshold has been met:

```typescript
const minCycleConfirmations = Math.min(
  Math.ceil(NodeList.getActiveNodeCount() / currentConsensusRadius),
  5
) || (cycle.counter <= 15 ? 1 : 3);
```

### Root Cause

The core issue is that the function processes the **last received cycle record** from the cycle data received via an adjacent archiver or a connected validator, instead of relying on a majority consensus of cycle records. This allows an attacker to manipulate which cycle record is ultimately used.

Relevant code snippet:

```typescript
export function collectCycleData(
  cycleData: P2PTypes.CycleCreatorTypes.CycleData[],
  senderInfo: string
): void {
  for (const cycle of cycleData) {

    // Non-relevant code omitted

    for (const value of Object.values(receivedCycleTracker[cycle.counter])) {
      if (value['saved']) {
        // If a cycle has already been saved, clear the cycleToSave list for this counter
        for (let i = 0; i < cycleToSave.length; i++) {
          // eslint-disable-next-line security/detect-object-injection
          receivedCycleTracker[cycle.counter][cycleToSave[i].marker]['saved'] = false;
        }
        cycleToSave = [];
        break;
      }
      if (value['receivedTimes'] >= minCycleConfirmations) {
        cycleToSave.push(cycle);
        value['saved'] = true;
      }
    }

    if (cycleToSave.length > 0) {
      processCycles(cycleToSave);
    }
  }
}
```

### Attack Exploitation

By carefully timing the submission of cycle records, an attacker can **inject their own cycle record** into a connected archiver and have it accepted as the authoritative version.

### Persistence Mechanism

To **maintain control** over an archiver, the attacker must:

1. **Spoof network topology** by injecting a fake list of added nodes while removing legitimate ones.
2. **Proxy traffic** through attacker-controlled nodes by using a combination of forged keys, IP addresses, and ports.
3. **Ensure reattachment** by influencing which validator the archiver selects for reconnection. If the archiver attempts to switch to another validator, it will still connect to an attacker-controlled node.

Once the archiver fully adopts the attacker's version of the network, further timing manipulations to inject cycle records are no longer necessary.

### Lateral Movement to Other Archivers

To **compromise additional archivers**, the attacker must:

* Manipulate delays to ensure their cycle record is the **fifth** record processed.
* Since there is no strict time limit, this process can be repeated indefinitely.
* Once a new archiver adopts the attacker's cycle record, it will only recognize attacker-controlled nodes.
* Over time, **all archivers** will fall under attacker control.

### Lateral Movement to Validators

New validators rely on archivers for:

* The **initial target node** for joining the network.
* The **initial node list** for discovering peers.
* The **network configuration**, including **developer keys**.

If an attacker-controlled archiver provides this data, new validators will unknowingly **join an attacker-controlled network segment** instead of the legitimate network.

This can lead to **network segmentation or even a full fork**:

* One segment controlled by the legitimate network.
* One segment controlled by the attacker.

Even **foundation nodes** will be compromised, as they also rely on archivers for initial data.

## Impact Details

This vulnerability allows an attacker to escalate from **manipulating a single archiver's perception of the network** to **gaining full control over the entire network**.

## References

<https://github.com/shardeum/archiver/blob/a131a752c5ab8ab5771142ad0f83ce06c610cd9b/src/Data/Data.ts#L354>

## Link to Proof of Concept

<https://gist.github.com/periniondon630/ca41b300c74ebd64e3d9c2cab8cdc911>

## Proof of Concept

## Proof of Concept

1. Apply the patch from the provided Gist to both the **archiver** and **core** repositories.
2. Build and start a standard local network with **10 nodes**.
3. In a network of this size, the **archiver** will use **2 validators** as `dataSenders`.
4. Check the **archiver's** `out.log` to determine which validator is sending the cycle first. Look for log messages such as `"new cycle from"` followed by an IP address.
5. Activate **attack mode** on the first sender (validator) by executing:

   ```sh
   curl -X POST <ip>:<port>/hackCycle
   ```

   This will modify the cycle record by adding the property `hacked: true`.
6. To ensure the archiver is the last one in the sequence, introduce a delay by running:

   ```sh
   curl "<ip>:<port>/cycleDelay?cycleDelay=<delay>"
   ```

   Replace `<delay>` with a number in milliseconds.
7. Experiment with different delay values until the **first sender (attacker’s node) becomes the last one** in the cycle. At this point, it will propagate its own cycle record containing the `hacked: true` property.

In real network there is only one way - by brute forcing different delay intervals and checking archiver /nodelist endpoint for malicious data.
