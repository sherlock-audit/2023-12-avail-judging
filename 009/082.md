Joyful Malachite Corgi

medium

# AvailBridge#updateTokens will cause bridge to break for updated assets

## Summary

When updating the assetIDs for tokens if there are any outstanding receive transactions then they risk breaking the entire withdraw system. 

## Vulnerability Detail

[AvailBridge.sol#L132-L146](https://github.com/sherlock-audit/2023-12-avail/blob/main/contracts/src/AvailBridge.sol#L132-L146)

    function updateTokens(bytes32[] calldata assetIds, address[] calldata tokenAddresses)
        external
        onlyRole(DEFAULT_ADMIN_ROLE)
    {
        uint256 length = assetIds.length;
        if (length != tokenAddresses.length) {
            revert ArrayLengthMismatch();
        }
        for (uint256 i = 0; i < length;) {
            tokens[assetIds[i]] = tokenAddresses[i];
            unchecked {
                ++i;
            }
        }
    }

When updating tokens the new contract address replaces the old contract address for a given assetID. This is problematic due to the nature of the bridge. When users bridge from ETH to AVAIL the tokens are stored in the bridge contract then retrieved later when receiving. This setup also makes migration of assetIDs problematic. This is because users can start withdrawals from AVAIL using the old asset then complete them later after the asset has been migrated. Take the following example:

`Token A` is associated with `assetID ABC`. `User A` sends 100 from ETH to AVAIL. After some time they token needs to be migrated. `User A` can front run the migration call with a transaction sending 100 of `assetID ABC` from AVAIL to ETH. Now after the migration has occurred and now `token B` is associated with `assetID ABC`, they can now complete the transfer and receive 100 of `token B` from the bridge. This creates an imbalance of tokens that permanently breaks some AVAIL to ETH transfers.

## Impact

Updated tokens can be accidentally or maliciously broken during migration

## Code Snippet

[AvailBridge.sol#L132-L146](https://github.com/sherlock-audit/2023-12-avail/blob/main/contracts/src/AvailBridge.sol#L132-L146)

## Tool used

Manual Review

## Recommendation

Currently assetIds are associated with tokens only by the assetId (`tokens[assetId]`). To fix the above exploit the token should also be associated with the last trusted block before migration. As an example if the `assetId ABC` is migrated from `token A` to `token B` at block 100,000 then all messages initiated before block 100,000 will still be associated with `token A` and all messages after will be associated with `token B`.