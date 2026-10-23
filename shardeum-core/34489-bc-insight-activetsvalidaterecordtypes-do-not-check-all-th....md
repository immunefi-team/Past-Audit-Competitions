# 34489 - \[BC - Insight] ActivetsValidateRecordTypes do not check all th...

Submitted on Aug 14th 2024 at 00:37:23 UTC by @gladiator111 for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34489

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Shutdown of greater than 10% or equal to but less than 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

`Note - This is an insight report, since there is no option to submit an insight, I am submitting it under low impact, kindly downgrade to insight from low.`

`Active.ts::ValidateRecordTypes` miss out on checking some of the types

## Vulnerability Details

In the function `Active.ts::ValidateRecordTypes`

```typescript
export function validateRecordTypes(rec: P2P.ActiveTypes.Record): string {
  let err = validateTypes(rec, {
    active: 'n',
    activated: 'a',
    activatedPublicKeys: 'a',
  })
  if (err) return err
  for (const item of rec.activated) {
    if (typeof item !== 'string') return 'items of activated array must be strings'
  }
  for (const item of rec.activatedPublicKeys) {
    if (typeof item !== 'string') return 'items of activatedPublicKeys array must be strings'
  }
  return ''
}
```

The above function checks only for `active`, `activated` and `activatedPublicKeys` and their subelements (in case of arrays). The `rec: P2P.ActiveTypes.Record` type consists of 5 elements. These types are as follows

```typescript
export interface Record {
    active: number;
    standby: number;
    activated: string[];
    activatedPublicKeys: string[];
    maxSyncTime: number;
}
```

`standby` and `maxSyncTime` types are never checked. This can cause problems of type mismatch. The recommendation for correcting it is as follows.

## Recommendation

Modify the function as follows

```typescript
export function validateRecordTypes(rec: P2P.ActiveTypes.Record): string {
  let err = validateTypes(rec, {
    active: 'n',
@>  standby: 'n',
    activated: 'a',
    activatedPublicKeys: 'a',
@>  maxSyncTime:'n'
  })
  if (err) return err
  for (const item of rec.activated) {
    if (typeof item !== 'string') return 'items of activated array must be strings'
  }
  for (const item of rec.activatedPublicKeys) {
    if (typeof item !== 'string') return 'items of activatedPublicKeys array must be strings'
  }
  return ''
}
```

## Impact Details

It comes under `security best practices` of insight report

## References

<https://github.com/shardeum/shardus-core/blob/72fba67d3a551f21368c8b0fe94f951c8f5cc4f8/src/p2p/Active.ts#L137>

## Proof of Concept

The issue is fairly simple and is `insight`, so I am providing only the necessary info for proof of concept.

```typescript
export function validateRecordTypes(rec: P2P.ActiveTypes.Record): string {
@>let err = validateTypes(rec, {                     // no checking for standby and maxSyncTime
    active: 'n',
    activated: 'a',
    activatedPublicKeys: 'a',
  })
  if (err) return err
  for (const item of rec.activated) {
    if (typeof item !== 'string') return 'items of activated array must be strings'
  }
  for (const item of rec.activatedPublicKeys) {
    if (typeof item !== 'string') return 'items of activatedPublicKeys array must be strings'
  }
  return ''
}
```

The above function is used by various other functions such as `CycleCreator.ts::validateCertsRecordTypes`, so it is fairly crucial

```typescript
function validateCertsRecordTypes(inp, caller) {
  let err = utils.validateTypes(inp, { certs: 'a', record: 'o' })
  if (err) {
    warn(caller + ' bad input: ' + err + ' ' + Utils.safeStringify(inp))
    return false
  }
  for (const cert of inp.certs) {
    err = utils.validateTypes(cert, { marker: 's', score: 'n', sign: 'o' })
    if (err) {
      warn(caller + ' bad input.certs: ' + err)
      return false
    }
    err = utils.validateTypes(cert.sign, { owner: 's', sig: 's' })
    if (err) {
      warn(caller + ' bad input.sign: ' + err)
      return false
    }
  }
  err = utils.validateTypes(inp.record, {
    activated: 'a',
    activatedPublicKeys: 'a',
    active: 'n',
    apoptosized: 'a',
    counter: 'n',
    desired: 'n',
    duration: 'n',
    expired: 'n',
    joined: 'a',
    joinedArchivers: 'a',
    joinedConsensors: 'a',
    lost: 'a',
    previous: 's',
    refreshedArchivers: 'a',
    refreshedConsensors: 'a',
    refuted: 'a',
    removed: 'a',
    start: 'n',
    syncing: 'n',
  })
  if (err) {
    warn(caller + ' bad input.record: ' + err)
    return false
  }
  //  submodules need to validate their part of the record
  for (const submodule of submodules) {
@>  err = submodule.validateRecordTypes(inp.record)   // used here
    if (err) {
      warn(caller + ' bad input.record.* ' + err)
      return false
    }
  }
  return true
}
```
