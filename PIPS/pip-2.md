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
Network fees can very expensive. For example, the total Ethereum network fees for some popular DApps can cost up to 500k USD over 30 days (source: https://ethgasstation.info/).

## Specification
The `lock` and `unlock` function of the LockProxy can be modified to include the following parameters:
- bytes memory feeReceiverAddr
- uint256 feeAmount
- uint64 feeChainId

The `feeAmount` indicates how much of the locked `amount` should be deducted and given to the `feeReceiverAddr`. The `feeChainId` allows control over which chain the fee deduction should be performed at.

Example fee deduction implementation:
```
uint256 remainingAmount = amount;

if (feeAmount > 0 && self.chainId == feeChainId) {
  remainingAmount -= feeAmount;
  // transfer feeAmount to feeReceiverAddr
}

// transfer remainingAmount to user address
```
