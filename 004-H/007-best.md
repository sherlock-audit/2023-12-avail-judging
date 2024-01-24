Ancient Eggshell Walrus

high

# Merkle.verify can return an unbalanced tree

## Summary
`Merkle.verify` function can return an unbalanced if the `index` is not zeroed out.

## Vulnerability Detail
### Merkle.verify
https://github.com/sherlock-audit/2023-12-avail/blob/main/contracts/src/lib/Merkle.sol#L12

The line `isValid := and(eq(leaf, root), iszero(index))` checks two conditions:
- If the final leaf value is equal to the root value.
- If the index has been zeroed out.

The index being zeroed out implies that all the elements of the proof have been processed, which would be the case for a balanced tree. If the index cannot be zeroed out, it means that not all elements of the proof have been processed, indicating that the tree is unbalanced.

So, if either of these conditions is not met, the function will return false, indicating that the verification failed. This could be due to an incorrect proof or an unbalanced tree. 

#### Cross referencing the `index` value
The `index` is part of the **proof input** that is passed as argument  `MerkleProofInput calldata input`. The index is in fact the `input.dataRootIndex`, so there's plenty of room of manipulation with this value to eventually make the line `isValid := and(eq(leaf, root), iszero(index))` return False.


## Impact
If the `index` cannot be zeroed out, then the Merkle tree is indeed unbalanced (resulting on failed verifications).

## Code Snippet
https://github.com/sherlock-audit/2023-12-avail/blob/main/contracts/src/lib/Merkle.sol#L12
https://github.com/sherlock-audit/2023-12-avail/blob/main/contracts/src/AvailBridge.sol#L482

## Tool used
Manual Review

## Recommendation
This code adds a require statement at the beginning of the verify function to check if the length of the proof is equal to the log base 2 of the index. If the lengths don’t match, the function will revert with the message “Unbalanced tree”. This ensures that the Merkle tree must be balanced for the verification to proceed.

```solidity
library Merkle {
    function verify(bytes32[] calldata proof, bytes32 root, uint256 index, bytes32 leaf)
        internal
        pure
        returns (bool isValid)
    {
        // Check if the proof length is equal to log2(index)
        require(proof.length == log2(index), "Unbalanced tree");

       // This code does the same thing as the original inline-assembly code
        bytes32 node = leaf;
        for (uint256 i = 0; i < proof.length; i++) {
            if (index % 2 == 0) {
                node = keccak256(abi.encodePacked(node, proof[i]));
            } else {
                node = keccak256(abi.encodePacked(proof[i], node));
            }
            index = index / 2;
        }
        isValid = node == root && index == 0;
    }

    // Helper function to calculate log base 2 of a number
    function log2(uint256 x) internal pure returns (uint256 y) {
        assembly {
            let arg := x
            x := sub(x,1)
            x := or(x, div(x, 0x02))
            x := or(x, div(x, 0x04))
            x := or(x, div(x, 0x10))
            x := or(x, div(x, 0x100))
            x := or(x, div(x, 0x10000))
            x := or(x, div(x, 0x100000000))
            x := or(x, div(x, 0x10000000000000000))
            x := or(x, div(x, 0x100000000000000000000000000000000))
            x := add(x, 1)
            let m := mload(0x40)
            mstore(m, 0x02)
            let p := 0x20
            for { } gt(x, 1) { } {
                for { } gt(x, 2) { x := div(x, 2) p := add(p, 1) }
                mstore(add(m, p), byte(0, div(sub(arg, x), x)))
                arg := x
                x := div(x, 2)
                p := add(p, 1)
            }
            mstore(add(m, p), byte(0, arg))
            y := keccak256(m, add(p, 1))
        }
    }
}
```