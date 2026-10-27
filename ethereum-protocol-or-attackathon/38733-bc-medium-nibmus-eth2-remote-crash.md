# #38733 \[BC-Medium] nibmus-eth2 remote crash

**Submitted on Jan 11th 2025 at 12:17:16 UTC by @gln for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38733
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/status-im/nimbus-eth2>
* **Impacts:**
  * Shutdown of less than 10% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

Nimbus-eth2 does not verify slot number of incoming attestations.

Because it uses checked integer calculations, malformed attestation may crash the node.

## Vulnerability Details

Nim signed integer arithmetic is checked, so in case of overflow Defect will be thrown.

Defects inherit from System.Defect, they are not catchable and terminate the running process.

Let's see how incoming attestations are processed <https://github.com/status-im/nimbus-eth2/blob/stable/beacon\\_chain/gossip\\_processing/eth2\\_processor.nim#L361>

```
proc processAttestation*(
    self: ref Eth2Processor, src: MsgSource,
    attestation: phase0.Attestation | electra.Attestation, subnet_id: SubnetId,
    checkSignature, checkValidator: bool
): Future[ValidationRes] {.async: (raises: [CancelledError]).} =
  var wallTime = self.getCurrentBeaconTime()
  let (afterGenesis, wallSlot) = wallTime.toSlot()

  logScope:
    attestation = shortLog(attestation)
    subnet_id
    wallSlot

  if not afterGenesis:
    notice "Attestation before genesis"
    return errIgnore("Attestation before genesis")

  let delay = wallTime - attestation.data.slot.attestation_deadline
  debug "Attestation received", delay

  # Now proceed to validation
  let v =
    await self.attestationPool.validateAttestation(
      self.batchCrypto, attestation, wallTime, subnet_id, checkSignature)
  return if v.isOk():
  ...
  ...
```

We are interested in attestation\_deadline function <https://github.com/status-im/nimbus-eth2/blob/stable/beacon\\_chain/spec/beacon\\_time.nim#L167>

```
type
  BeaconTime* = object
    ns_since_genesis*: int64

  TimeDiff* = object
    nanoseconds*: int64

const
  # Earlier spec versions had these at a different slot
  GENESIS_SLOT* = Slot(0)
  GENESIS_EPOCH* = Epoch(0) # compute_epoch_at_slot(GENESIS_SLOT)

  # https://github.com/ethereum/consensus-specs/blob/v1.5.0-alpha.9/specs/phase0/fork-choice.md#constant
  INTERVALS_PER_SLOT* = 3

  FAR_FUTURE_BEACON_TIME* = BeaconTime(ns_since_genesis: int64.high())

  NANOSECONDS_PER_SLOT* = SECONDS_PER_SLOT * 1_000_000_000'u64

const
  attestationSlotOffset* = TimeDiff(nanoseconds:
    NANOSECONDS_PER_SLOT.int64 div INTERVALS_PER_SLOT)

func start_beacon_time*(s: Slot): BeaconTime =
   const maxSlot = Slot(
    uint64(FAR_FUTURE_BEACON_TIME.ns_since_genesis) div NANOSECONDS_PER_SLOT)
1) if s > maxSlot: FAR_FUTURE_BEACON_TIME
   else: BeaconTime(ns_since_genesis: int64(uint64(s) * NANOSECONDS_PER_SLOT))

func attestation_deadline*(s: Slot): BeaconTime =
2)  s.start_beacon_time + attestationSlotOffset
```

1. Slot type is basically uint64, if it is very large, start\_beacon\_time() function will return FAR\_FUTURE\_BEACONTIME which is equal to int64.high
2. BeaconTime is int64, so if s.start\_beacon\_time is int64.high and attestationSlotOffset is not zero, this expression will overflow

As a result OverflowDefect will be thrown and beacon node will stop working

## Impact Details

Attacker could crash nimbus-eth2 node with a single malformed attestation.

## Link to Proof of Concept

<https://gist.github.com/gln7/034880a75da3dcc3b809798667291273>

## Proof of Concept

## Proof of Concept

To trigger the issue we need to modify attestation before broadcasting it to the network.

How to reproduce:

1. get nimbus source

```
$ git rev-parse stable
4e440277cf8a3fed72f32eb2f01fc5e910ad6768

```

2. apply poc.patch (see gist link)
3. start localnet

```
$ make VALIDATORS=50 NUM_NODES=6 USER_NODES=0 local-testnet-minimal
```

4. after localnet stops, you can find the following messages in local-testnet-minimal/logs/nimbus\_beacon\_node.1.jsonl

```
/nimbus-eth2/vendor/nim-libp2p/libp2p/protocols/pubsub/pubsub.nim(366) _ZN14eth
2_processor18processAttestationE3refIN14eth2_processor13Eth2ProcessorEEN17validator_monitor9MsgSourceEN6phase011Att
estationEN4base8SubnetIdE4bool4bool
/nimbus-eth2/vendor/nim-libp2p/libp2p/protocols/pubsub/pubsub.nim(371) _ZN12asy
ncfutures14futureContinueE3refIN7futures26FutureBasecolonObjectType_EE
/nimbus-eth2/vendor/nim-libp2p/libp2p/protocols/pubsub/pubsubpeer.nim(379) _ZN1
8processAttestation56processAttestation
/nimbus-eth2/beacon_chain/nimbus_beacon_node.nim(168) _ZN11beacon_time20attestation_deadlineEN9constants4SlotE
_ZN9gossipsub10rpcHandlerE3refIN5types25GossipSubcolonObjectType_EE3refIN10pubsubpeer26PubSubPeercolonObjectType_EE3seqI5uInt8E(15) raiseOverflow
/nimbus-eth2/vendor/nim-chronos/chronos/internal/asyncfutures.nim(53) _ZN18nimbus_beacon_node3runE3refIN11beacon_node26BeaconNodecolonObjectType_EE
/nimbus-eth2/vendor/nim-chronos/chronos/internal/asyncengine.nim(345) _ZN18nimbus_beacon_node3runE3refIN11beacon_node26BeaconNodecolonObjectType_EE
/nimbus-eth2/vendor/nimbus-build-system/vendor/Nim/lib/system/stacktraces.nim(62) /nimbus-eth2/beacon_chain/nimbus_beacon_node.nim
[[reraised from:
...
Error: unhandled exception: over- or underflow [OverflowDefect]

```
