Warm Brick Cuckoo

high

# verification of bridge leaf will always fail due to unhashed leaf input argument

## Summary
verification of bridge leaf will always fail due to unhashed leaf input argument

## Vulnerability Detail

`AvailBridge.verifyBridgeLeaf()`  ia used to verify the bridge leaf which takes a Merkle tree proof of inclusion for a bridge leaf and verifies it.  The issue here is, it takes a `input.leaf` as one of argument which is passed as without keccak256 hashed. The Natspec specifically says to pass the leaf as hash of keccak.

```solidity
440        // leaf must be keccak(message)
```

and `verifyBridgeLeaf()` is given as below,

```solidity
    function verifyBridgeLeaf(MerkleProofInput calldata input) public view returns (bool) {
        if (input.bridgeRoot == 0x0) {
            revert BridgeRootEmpty();
        }
        _checkDataRoot(input);
        // leaf must be keccak(message)                 @audit // comment is not incorporated 
        // we don't need to check that the leaf is non-zero because we check that the root is non-zero
        return input.leafProof.verify(input.bridgeRoot, input.leafIndex, input.leaf);          @audit // input leaf argument is not hashed
    }
```

While verifying the bridge leaf, the verification will always fail as input.leaf is passed as bytes32 argument instead of `keccak256(abi.encode(input.leaf)`. 

Let's consider a example in remix,
consider a input leaf in bytes32 argument, `input.leaf = 0x111122223333444455556666777788889999AAAABBBBCCCCDDDDEEEEFFFFCCCC;`

If this bytes32 argument is hashed via keccak(abi.encode(input.leaf) then the result would be `0x276b28b3bf3c9bd84af7f4ce06088d3c0df1ff732522ed263e466b1a63504ad6`

This is completely different and the same can be applied while verification of bridge leaf and it will always fail due to wrong input argument.

`verifyBridgeLeaf()` is used in private function `_checkBridgeLeaf()`

```solidity
    function _checkBridgeLeaf(Message calldata message, MerkleProofInput calldata input) private {
        bytes32 leaf = keccak256(abi.encode(message));
        if (isBridged[leaf]) {
            revert AlreadyBridged();
        }
        // validate that the leaf being proved is indeed the message hash!
        if (input.leaf != leaf) {
            revert InvalidLeaf();
        }
        // check proof of inclusion
        if (!verifyBridgeLeaf(input)) {             @audit // will always revert as bridge leaf verification will always fail.
            revert InvalidMerkleProof();
        }
        // mark as spent
        isBridged[leaf] = true;
    }
```

`_checkBridgeLeaf()` is further used in following funcions,

```solidity
    function receiveMessage(Message calldata message, MerkleProofInput calldata input)

   . . . some code

        _checkBridgeLeaf(message, input);         @audit // bridge verification will fail due to issue in internal used functions

   . . . some code
}


    function receiveAVAIL(Message calldata message, MerkleProofInput calldata input)

   . . . some code

        _checkBridgeLeaf(message, input);

   . . . some code
}


    function receiveETH(Message calldata message, MerkleProofInput calldata input)

   . . . some code

        _checkBridgeLeaf(message, input);
   . . . some code
}


    function receiveERC20(Message calldata message, MerkleProofInput calldata input)

   . . . some code

        _checkBridgeLeaf(message, input);

   . . . some code
}
```

All these functions would revert during the verification of bridge leaf. The major contract functionality would be bricked.

## Impact
Failure of bridge leaf verification would lead overall brick of contract functionality and the above functions would revert due to error while passing the input.leaf argument during bridge leaf verification. In addition, it does not incorporate the natspec design consideration while passing input.leaf argument.

## Code Snippet
https://github.com/sherlock-audit/2023-12-avail/blob/1afb56b8d4dfbf5d3f21bed0ddf80a04730204b5/contracts/src/AvailBridge.sol#L442

## Tool used
Manual Review

## Recommendation

Consider below change to mitigate the issue. This is also inline with Natspec design for leaf argument for bridge leaf verification.

```diff
    function verifyBridgeLeaf(MerkleProofInput calldata input) public view returns (bool) {
        if (input.bridgeRoot == 0x0) {
            revert BridgeRootEmpty();
        }
        _checkDataRoot(input);
        // leaf must be keccak(message)
        // we don't need to check that the leaf is non-zero because we check that the root is non-zero
-        return input.leafProof.verify(input.bridgeRoot, input.leafIndex, input.leaf);
+        return input.leafProof.verify(input.bridgeRoot, input.leafIndex, keccak256(abi.encode(input.leaf)));
    }
```