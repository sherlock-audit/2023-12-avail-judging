Electric Rusty Haddock

medium

# The verification can lead to Merkle tree collision

## Summary
`AvailBridge` utilizes Merkle Proof for leaf verification; however, it overlooks the verification of the length of the leaf bytes.

## Vulnerability Detail
It's a must to ensure to avoid using leaf values that are 64 bytes long before hashing or utilize a hash function other than keccak256 for hashing leaves. This is because the concatenation of a sorted pair of internal nodes in the Merkle tree could be reinterpreted as a leaf value.

`abi.encode` encodes the params according to the ABI specs. Params are padded out to 32 bytes.
Since `abi.encode(bytes32,bytes32)` will also be 64 bytes it is possible to have a hash collision between a leaf and a parent node.

[Reference & Credits](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/3091)

## Impact
The leaf and parent data have the same 64-byte size, allowing hash collisions between a leaf and any node. This enables the repetition of proofs using subtrees as leaves, leading to the creation of fraudulent proofs.
## Code Snippet
https://github.com/sherlock-audit/2023-12-avail/blob/1afb56b8d4dfbf5d3f21bed0ddf80a04730204b5/contracts/src/interfaces/IAvailBridge.sol#L22C1-L39C6
```solidity
//contracts/src/interfaces/IAvailBridge.sol
    struct MerkleProofInput {
        // proof of inclusion for the data root
        bytes32[] dataRootProof;
        // proof of inclusion of leaf within blob/bridge root
        bytes32[] leafProof;
        // abi.encodePacked(startBlock, endBlock) of header range commitment on vectorx
        bytes32 rangeHash;
        // index of the data root in the commitment tree
        uint256 dataRootIndex;
        // blob root to check proof against, or reconstruct the data root
        bytes32 blobRoot; //@audit 
        // bridge root to check proof against, or reconstruct the data root
        bytes32 bridgeRoot;//@audit 
        // leaf being proven
        bytes32 leaf;
        // index of the leaf in the blob/bridge root tree
        uint256 leafIndex;
    }
```

https://github.com/sherlock-audit/2023-12-avail/blob/1afb56b8d4dfbf5d3f21bed0ddf80a04730204b5/contracts/src/AvailBridge.sol#L491
```solidity
 dataRootCommitment, input.dataRootIndex, keccak256(abi.encode(input.blobRoot, input.bridgeRoot))
```
## Tool used

Manual Review

## Recommendation
Use a combination of variables that does not sum to 64 bytes