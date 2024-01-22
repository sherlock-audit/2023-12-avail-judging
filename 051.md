Smooth Aqua Weasel

medium

# Proofs for verifyBridgeLeaf and verifyBlobLeaf methods can be swapped by specifyng crafted blobRoot and bridgeRoot in _checkDataRoot

## Summary

Proofs for `verifyBridgeLeaf` and `verifyBlobLeaf` methods can be swapped by specifyng crafted `input.blobRoot` and `input.bridgeRoot` in `_checkDataRoot` method. We can use any pair of (Merkle tree vertex, proof) from given proofs in `input` as (`input.blobRoot`, `input.bridgeRoot`) in `_checkDataRoot` method and still have the correct proof for the leaf. By selecting the specific Merkle tree vertex (with left proof or right proof) we are able to set `input.blobRoot` and `input.bridgeRoot` in a way that leads to choosing wrong root in `verifyBridgeLeaf` or `verifyBlobLeaf` methods. However, it doens't lead to contract attack, because leafs in this contract can be only `keccak256(keccak256(blob))` or `keccak256(message)` and cannot be swapped. 

## Vulnerability Detail
`_checkDataRoot` method verifies that [`keccak256(abi.encode(input.blobRoot, input.bridgeRoot))` exists in the Merkle tree](https://github.com/sherlock-audit/2023-12-avail/blob/main/contracts/src/AvailBridge.sol#L490-L491), however this structure of leaf is similar to any [intermidiate vertex structure `keccak256(leaf . proof(i))`](https://github.com/sherlock-audit/2023-12-avail/blob/main/contracts/src/lib/Merkle.sol#L29-L33). And by specifying `blobRoot` and `bridgeRoot` as `leaf` and `proof(i)` for any given proof in `input` (even from proof for the leaf, not for the dataRoot) we can still have the valid proof for dataRoot and leaf. By selecting the specific Merkle tree vertices with left proof or right proof we can interchange proofs for `verifyBridgeLeaf` and `verifyBlobLeaf` methods. To put it simply, we can call  `verifyBridgeLeaf` or `verifyBlobLeaf` with valid proof, but with the wrong type of leaf: `blob` for `verifyBridgeLeaf` and `message` for `verifyBlobLeaf`. But this scenario doesn't lead to direct contract attack, as `blob` is stored as `keccak256(keccak256(blob)` and `message` as `keccak256(message)`, so we can't craft any `blob` or `meesage` that fits the specified hash. 

```text
        dataRoot
        /      \
   blobRoot   bridgeRoot
              /        \
           vertex1   proof1
        ...
     /        \
   proof2   vertex2
            /     \
     message leaf  proofL

We can construct new proof with blobRoot as vertex1, bridgeRoot as proof1. 

This proof for message leaf will be valid for blobRoot as the new root, not bridgeRoot.
```
## Impact

Possible attack on leafs in the Merkle tree. Message can be interpretead as blob, and vice versa.

## Code Snippet

https://github.com/sherlock-audit/2023-12-avail/blob/main/contracts/src/AvailBridge.sol#L490-L491

https://github.com/sherlock-audit/2023-12-avail/blob/main/contracts/src/lib/Merkle.sol#L29-L33

## Tool used

Manual Review

## Recommendation

Change the leaf structure in `_checkDataRoot`, for example add more data to it. `keccak256(abi.encode(input.blobRoot, input.bridgeRoot))` -> `keccak256(abi.encode(<constant>, input.blobRoot, input.bridgeRoot))`