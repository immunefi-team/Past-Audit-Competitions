# #39360 \[W\&A-Insight] getRandomActiveNodes may return inconsistent results

**Submitted on Jan 28th 2025 at 14:02:49 UTC by @Franfran for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39360
* **Report Type:** Websites and Applications
* **Report severity:** Insight
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

The `NodeList.getRandomActiveNodes(...)` function should return random nodes evenly distributed but it is not the case and may break some assumptions if it happens. Some other corectness issues have been identified.

## Vulnerability Details

The `getRandomActiveNodes` function and the underlying `getRandomItemFromArr` function both contains bugs.\
The `getRandomItemFromArr` is supposed to return an even distribution of random unique elements from the passed array, but this is not true because of the `nodeRejectPercentage` variable, and the result may contain duplicates if this is over 0, because it fakes the distribution and breaks the (partial) fisher-yates shuffle. The POC shows that this function may contain duplicates, meaning that the property of the returned elements being unique is wrong.\
Additionally, some other corectness issues have been found, which have minimal impact, but is still very important to keep in mind and to fix since these are broken assumptions, and the code should not be built upon those:

### Array is sorted by ID rather than by insertion order

When calling [getRandomActiveNodes](https://github.com/shardeum/archiver/blame/4e91b2db48c7eae54f4ce1e1729b3a7112466a4e/src/NodeList.ts#L369), the function used to get the list of nodes is [getActiveList](https://github.com/shardeum/archiver/blob/4e91b2db48c7eae54f4ce1e1729b3a7112466a4e/src/NodeList.ts#L370) which will return a [sorted list of nodes by ID](https://github.com/shardeum/archiver/blob/4e91b2db48c7eae54f4ce1e1729b3a7112466a4e/src/NodeList.ts#L256-L259) rather than by insertion order. Meaning that because elements in the lower part of the array (smaller ID) won't even be considered as they are within the `N_NODE_REJECT_PERCENT` (5% by default).\
The nodes list should be sorted by insertion order to select the most fresh nodes.

### Should send the full list if exceeding `node_count` instead of just one

In the [getRandomActiveNodes](https://github.com/shardeum/archiver/blame/4e91b2db48c7eae54f4ce1e1729b3a7112466a4e/src/NodeList.ts#L369), if the requested `node_count` is 1 or less, the function will just return one node. But what is odd is that if it exceeds the node list, it would also only return one. It might seems more correct to just return the full list.

## Impact Details

1. The most important and most severe: will worsen the consensus property of the [robustQuery](https://github.com/shardeum/archiver/blob/66e31b3ad656493a45dc301852969ab92d5423b2/src/Data/Data.ts#L560-L565) call in [syncFromNetworkConfig](https://github.com/shardeum/archiver/blob/66e31b3ad656493a45dc301852969ab92d5423b2/src/Data/Data.ts#L557-L558) with the redundancy being faked by the faulty distribution of random nodes. Note that according to the POC, it's only possible to have elements coming twice, never thrice or more.
2. Will send duplicated requests to nodes in the [exitArchiver](https://github.com/shardeum/archiver/blob/8145e6fd4233d670f4dd11074a0de9e86911cf1d/src/State.ts#L132-L136) and [sendRefute](https://github.com/shardeum/archiver/blob/30ef2fa8dde9cfeef8b88c190c7976c1d3379fa2/src/LostArchivers.ts#L85) functions or may only send to one node if the current node list has less than 5 elements.
3. [getCachedNodeList](https://github.com/shardeum/archiver/blob/4e91b2db48c7eae54f4ce1e1729b3a7112466a4e/src/NodeList.ts#L269) may return duplicate values which may make the archiver respond to `/nodelist` with duplicate nodes. Archivers currently handle that but they might start to consider the peer as being not trustworthy as it's wasting bandwidth if a reputation system is ever put in place.

## References

Links attached when applicable.

## Proof of Concept

## Proof of Concept

```typescript
import {getRandomItemFromArr} from "../Utils";  
  
const MAX_ELEMENTS = 50;  
const N_NODE_REJECT_PERCENT = 5;  
  
const randomArray = (maxSize: number): number[] => {  
  const size = Math.floor(1 + Math.random() * (maxSize - 1));  
  return fillArray(size);  
}  
  
const fillArray = (size: number): number[] => {  
  let arr = new Array(size);  
  for (let i = 0; i < size; i++) {  
    arr[i] = i;  
  }  
  return arr;  
}  
  
const randomItemDupCheck = () => {  
  for (let i = 0; i < 500; i++) {  
    const arr = randomArray(MAX_ELEMENTS);  
    const maxNum = Math.floor(1 + Math.random() * (arr.length - 1));  
    const rand = getRandomItemFromArr(arr, N_NODE_REJECT_PERCENT, maxNum);  
    let dups = new Map<number, boolean>();  
    for (let j = 0; j < rand.length; j++) {  
      const el = rand[j];  
      if (dups.get(el) && dups.get(el) == true) {  
        console.error(`found duplicate ${el}`);  
        process.exit(0);  
      } else {  
        dups.set(el, true);  
      }  
    }  
  }  
}  
  
randomItemDupCheck();
```

Simply run the script with

```sh
ts-node src/test/index.ts
```

And observe the output, if it has found a duplicate, it should show something like:

```sh
found duplicate 35
```
