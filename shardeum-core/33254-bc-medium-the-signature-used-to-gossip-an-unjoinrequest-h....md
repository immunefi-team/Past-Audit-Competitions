# 33254 - \[BC - Medium] The signature used to Gossip an UnjoinRequest h...

Submitted on Jul 16th 2024 at 13:58:37 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33254

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## For Triaggers

We enjoy sharing professional and detailed reports with runnable proof of concepts, but this bug can be undeniably confirmed by just reading a specific code snippet from the codebase.

Investing hours developing what may be interpreted as a "proof of concept" for a bug of this nature makes no sense. We expect the good judgment of the reviewers to agree with us after reading the report, but if not, let us know and we will code what you may consider a valid proof of concept for this report.

## Vulnerability Details

A node that is in StandBy, can sign and gossip an `UnjoinRequest` for multiple reasons - for example, if he needs to restart its server before joining the network again.

`UnjoinRequest` is declared as:

```
/**
 * A request to leave the network's standby node list.
 */
export type UnjoinRequest = SignedObject<{
  publicKey: hexstring
}
```

> Code snippet from: <https://github.com/shardeum/shardus-core/blob/dev/src/p2p/Join/v2/unjoin.ts#L12-L17>
>
> The `UnjoinRequest` is a `SignedObject`. Here are the fields of a \`SignedObject:

```
/** A `T` signed with a signature `sign`. */
export declare type SignedObject<T = LooseObject> = T & {
    sign: Signature;
};
export interface Signature {
    owner: string;
    sig: string;
}
```

As it can be seen from the definition of the `UnjoinRequest` object, the only thing the legit node is requested to sign is a `publicKey`.

Unfortunately, because he is not requested to sign a `nonce`, or a `timestamp`, or what the `currentCycle` is, or any extra field to prevent replay attacks, bad actors can re-gossip the same `UnjoinRequest` with the same signature in the future, to maliciously prevent this node from joining the network.

## Permanent fix

The object `UnjoinReuqest` must contain a `nonce`, a `timestamp`, or a field to store the `currentCycle`, to make sure if this request is replayed in the future it will be marked as invalid and not processed.

## Impact Details

A rogue node can collect these signatures that are gossiped, and when he owns enough of them, he can replay them to prevent nodes from joining.

The limits of this attack, are that only nodes in StandBy can become victims of this vector.

We selected the severity "**Shutdown of greater than or equal to 30% of network processing nodes without brute force actions but does not shut down the network**", as we believe it represents quite well the impact of reproducing this attack with a large list of signatures from nodes in StandBy.

If the Shardus team, with his deeper knowledge of their platform, knows there could be higher impacts by exploiting this attack vector, we invite them to increase the severity of the report.

## Proof of Concept

As explained in the first segment of this report, an undeniable proof of the validity of this report is by reading the structure of the object that has to be signed by a node when submitting an `UnjoinRequest`.

`UnjoinRequest` is declared as:

```
/**
 * A request to leave the network's standby node list.
 */
export type UnjoinRequest = SignedObject<{
  publicKey: hexstring
}
```

The object must contain a `nonce`, a `timestamp`, or a field to store the `currentCycle`, to make sure this request can't be replayed in the future.
