Joyful Malachite Corgi

medium

# User can circumvent fee requirement by sending message with no data

## Summary

Messages are free when sent with no data allowing for fees to be bypassed.

## Vulnerability Detail

[AvailBridge.sol#L306-L308](https://github.com/sherlock-audit/2023-12-avail/blob/main/contracts/src/AvailBridge.sol#L306-L308)

    if (msg.value < getFee(length)) {
        revert FeeTooLow();
    }

    function getFee(uint256 length) public view returns (uint256) {
        return length * feePerByte;
    }

When sending a message, the contract enforces a fee per byte of data sent. If a user sends a message without any data then the fee will return as zero. This allows users to send message for free. This can be useful for quite a few purposes such as incrementing counters or sending ping messages back after completed message or ETH transfers.

Another aspect to consider is that the recipient address can also be used as free adhoc data transfer up to 32 bytes, since there is no callback or the like when messages are received on AVAIL.


## Impact

Users can send messages for free

## Code Snippet

[AvailBridge.sol#L450-L452](https://github.com/sherlock-audit/2023-12-avail/blob/main/contracts/src/AvailBridge.sol#L450-L452)

## Tool used

Manual Review

## Recommendation

Fee should be applied to data length + 32 since recipient is arbitrary user supplied data.