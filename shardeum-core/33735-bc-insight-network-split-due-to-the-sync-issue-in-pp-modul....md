# 33735 - \[BC - Insight] Network split due to the sync issue in PP modul...

Submitted on Jul 27th 2024 at 23:33:17 UTC by @ret2happy for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33735

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Unintended chain split (network partition)

## Description

## Brief/Intro

Incorrect and insufficient type validation for the `sync-cycles` P2P http response, leading to the sync issue and the network partition during the sync.

## Vulnerability Details

When the node tries to get the cycles with counter from start to end inclusively, it iter each p2p node with the `sync-cycles` http POST request, and parse the request to get the cycles information. It validate the response from the HTTP POST response in \[1].

```
 const queryFn = async (node: SyncNode) => {
    const ip = node.ip ? node.ip : node.externalIp
    const port = node.port ? node.port : node.externalPort
    const resp = await http.post(`${ip}:${port}/sync-cycles`, data)
    return resp
  }

  /* prettier-ignore */ if (logFlags.p2pNonFatal) info(`getCycles: ${start} - ${end}...`)

  // use robust query so we can ask less nodes to get the cycles
  let redundancy = 1
  if (activeNodes.length > 5) redundancy = 2
  if (activeNodes.length > 10) redundancy = 3
  const { topResult: response, winningNodes: _responders } = await robustQuery(
    activeNodes,
    queryFn,
    util.isDeepStrictEqual,
    redundancy,
    true
  ) // [1] get response from the `sync-cycles` endpoint.

  // [TODO] Validate whatever came in
  const cycles = response as P2P.CycleCreatorTypes.CycleRecord[]

  const valid = validateCycles(cycles) // [1] validate response by the validateCycles function
  if (valid) return cycles
}
}
```

However, there's incorrect and insufficient type validation in the `validateCycles` functions.

In the `validateCycles` function, line 555 of the `Sync.ts`, it lacks checking whether `cycles` is iterable. Hence iter the `cycles` will result the js runtime exception. (Not that the previous response is `unknown` type and is force cast dynamically to the `P2P.CycleCreatorTypes.CycleRecord`, which doesn't necessarily to be the array type. )

Moreover, in line 579 - line 581, when the type is mismatched, it only log a warning without directly return false, leading to the execution on the wrongly typed `cycles` data. Hence in the following content of this function, each field is not type validated at all, and could cause many type iter exception and thus fail to validate the type as the correct one.

```
function validateCycles(cycles: P2P.CycleCreatorTypes.CycleRecord[]) {
  const archiverType = {
    publicKey: 's',
    ip: 's',
    port: 'n',
    curvePk: 's',
  }
  // [2] Since we lack of checking whether `cycles` is iterable, iter here will result the js runtime exception. (Not that the previous response is `unknown` type and is force cast dynamically to the `P2P.CycleCreatorTypes.CycleRecord`, which doesn't necessarily to be the array type. )
  for (const cycleRecord of cycles) {
    let err = validateTypes(cycleRecord, {
      safetyMode: 'b',
      safetyNum: 'n',
      networkStateHash: 's',
      refreshedArchivers: 'a',
      refreshedConsensors: 'a',
      joinedArchivers: 'a',
      leavingArchivers: 'a',
      syncing: 'n',
      joinedConsensors: 'a',
      active: 'n',
      activated: 'a',
      activatedPublicKeys: 'a',
      lost: 'a',
      refuted: 'a',
      joined: 'a',
      returned: 'a',
      apoptosized: 'a',
      networkDataHash: 'a',
      networkReceiptHash: 'a',
      networkSummaryHash: 'a',
      desired: 'n',
    })
    if (err) {
      warn('Type validation failed for cycleRecord: ' + err) // [2] if the type is not validated, this won't return false. Instead, it continue execution without further type validation. 
    }
    // [2] Since we fail to check the type of the refreshedArchivers field, the refreshedArchivers field can bt non-iterable, resulting the js runtime exception.
    for (const refreshedArchiver of cycleRecord.refreshedArchivers) {
      err = validateTypes(refreshedArchiver, archiverType)
      if (err) {
        warn('Validation failed for cycleRecord.refreshedArchivers: ' + err)
        return false
      }
    }
    ...
```

## Impact Details

The malicious P2P response can stuck and cause exception during the sync since there's no error catching for the cycle sync function. Moreover, there's missing validation on the wrong P2P response, which also create consensus issue among the nodes via p2p request.

## References

\[1] <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/p2p/Sync.ts#L495-L507>

\[2] <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/p2p/Sync.ts#L548-L582>

## Proof of Concept

I create a unit test case under the `shardus-core` repo, the file is named as `test/unit/src/ValidateCycles.test.ts`.

```
import {validateCycles} from "../../../src/p2p/Sync";
import {P2P} from '@shardus/types'

test('Error Response Parent Body Type Should Be Rejected', () => {
    const simulatedResponse = {}; // should be array rather than the map
    const cycles = simulatedResponse as P2P.CycleCreatorTypes.CycleRecord[];
    const valid = validateCycles(cycles); // should return false rather than raise exception
    console.log("valid: ", valid);
    // running result: exception
    // expected result: false
})

test('Error Response refreshedArchivers Field Type Should Be Rejected', () => {
    const simulatedResponse = [
        {refreshedArchivers: {}}
    ];
    const cycles = simulatedResponse as P2P.CycleCreatorTypes.CycleRecord[];
    // should return false (since refreshedArchivers should be array type) rather than raise exception
    const valid = validateCycles(cycles);
    console.log("valid: ", valid);
    // running result: exception
    // expected result: false
})

test('Error Response safetyMode/safetyNum/networkStateHash/desired Field Type Should Be Rejected', () => {
    const simulatedResponse: unknown = [
        {
            safetyMode: [], // should be bool type
            safetyNum: "", // should be the number type
            networkStateHash: [], // should be the string type
            desired: true, // should be the number type
            // other types are filled with correct ones
            refreshedArchivers: [],
            refreshedConsensors: [],
            joinedArchivers: [],
            leavingArchivers: [],
            joinedConsensors:[],
            activated: [],
            activatedPublicKeys:[],
            lost:[],
            refuted:[],
            joined:[],
            returned:[],
            apoptosized:[],

        }
    ];
    const cycles = simulatedResponse as P2P.CycleCreatorTypes.CycleRecord[];
    // should return false (since some fields have wrong assumed type) rather than return true
    const valid = validateCycles(cycles);
    console.log("valid: ", valid);
    // running result: false
    // expected: true
})

```

These PoC demonstrates three different case which `validateCycles` failed to validate.

The first case is that the `map` type (the `cycles` data type itself), (default by json response) is not handled by `validateCycles`, hence it cause exception rather than return false.

The second case is that the wrong type of `refreshedArchivers` field cause the exception rather than the false return value from the `validateCycles`. This is because the `validateCycles` doesn't return false even it checks the field types.

The third case is that the wrong types of unchecked/un-itered field such as `safetyMode/safetyNum/networkStateHash/desired` are not type checked. However, these field are later used in the cycle calculation, resulting the insufficient field checking. These arbitrary unchecked field can further break the p2p consensus.

Running these PoC by

```
jest test/unit/src/ValidateCycles.test.ts -v
```

You will get the console output:

```
// first one
cycles is not iterable
TypeError: cycles is not iterable
    at validateCycles (/shardus-core/src/p2p/Sync.ts:555:29)
    at Object.<anonymous> (/shardus-core/test/unit/src/ValidateCycles.test.ts:7:33)
    at Promise.then.completed (/shardus-core/node_modules/jest-circus/build/utils.js:391:28)
    at new Promise (<anonymous>)
```

```
// second one
cycleRecord.refreshedArchivers is not iterable
TypeError: cycleRecord.refreshedArchivers is not iterable
    at validateCycles (/shardus-core/src/p2p/Sync.ts:582:49)
    at Object.<anonymous> (/shardus-core/test/unit/src/ValidateCycles.test.ts:19:33)
    at Promise.then.completed (/shardus-core/node_modules/jest-circus/build/utils.js:391:28)
    at new Promise (<anonymous>)
```

```
// third one
valid:  true
```
