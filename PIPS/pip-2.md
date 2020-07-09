---
pip: 2
title: Relayer Fees
status: Draft
author: John Wong (@jwheur)
created: 2020-06-30
---

## Simple Summary
The current LockProxy implementation does not have an in-built method for relayers to charge fees for their work.

This PIP proposes a simple method to charge fees.

## Motivation
Network fees can be very expensive. For example, the total Ethereum network fees for some popular DApps can cost up to 500k USD over 30 days (source: https://ethgasstation.info/).

## Specification
The `lock` and `unlock` function of the LockProxy can be modified to include the following parameters:
- bytes memory feeReceiverAddr
- address feeAssetHash
- uint256 feeAmount
- uint64 feeChainId

The `feeAmount` indicates how much of the locked `amount` should be deducted and given to the `feeReceiverAddr`. The `feeChainId` allows control over which chain the fee deduction should be performed at.

Example fee deduction implementation:
```
uint256 remainingAmount = amount;

if (feeAmount > 0 && self.chainId == feeChainId) {
  // for `lock` the comparison should be fromAssetHash == feeAssetHash
  // for `unlock` the comparison should be toAssetHash == feeAssetHash
  if (fromAssetHash == feeAssetHash) {
    remainingAmount -= feeAmount;
  } else {
    // deduct feeAmount from the user's balance
  }

  // transfer feeAmount to feeReceiverAddr
}

// transfer remainingAmount to user address
```

If the feeChainId is not the target chain, it is possible for a relayer to only send the transaction to the originating chain, claim their fee, and then not send the second transaction to the target chain.

This PIP does not propose a fixed solution to this issue, but possible solutions include:
1. Requiring the feeChainId to be the target chain
2. Having a reputation system for relayers
3. Requiring relayers to have some value staked, which can be slashed if they are reported for slow broadcasts or non-broadcasts
