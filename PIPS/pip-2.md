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

Example fee deduction implementation for lock:
```
struct TxArgs {
  bytes fromAssetHash;
  bytes toAssetHash;
  bytes toAddress;
  uint256 amount;
  uint256 feeAmount;
  bytes feeReceiverAddr;
}

function lock(
  address fromAssetHash,
  uint64 toChainId,
  bytes memory targetProxyHash,
  bytes memory toAssetHash,
  bytes memory toAddress,
  uint256 amount,
  bool deductFeeInLock,
  uint256 feeAmount,
  bytes memory feeReceiverAddr
)
  public
{
  bytes32 key = hash(fromAssetHash, toChainId, targetProxyHash, toAssetHash);
  require(registry[key] == true);

  // Use SafeMath to ensure balances do not overflow
  balances[key] = balances[key].add(amount);

  // transfer tokens from user to LockProxy
  require(_transferToContract(fromAssetHash, amount), "transfer asset from fromAddress to lock_proxy contract failed!");

  TxArgs memory txArgs = TxArgs({
      fromAssetHash: fromAssetHash,
      toAssetHash: toAssetHash,
      toAddress: toAddress,
      amount: amount,
      feeAmount: feeAmount,
      feeReceiverAddr: feeReceiverAddr
  });

  if (feeAmount > 0 && deductFeeInLock) {
    // ensure that there is no overflow
    uint256 afterFeeAmount = amount.sub(feeAmount);

    require(_transferFromContract(fromAssetHash, feeReceiverAddr, feeAmount), "transfer asset from lock_proxy contract to toAddress failed!");

    // change fee amount to zero as the fee has already been deducted
    txArgs.feeAmount = 0;
    txArgs.amount = afterFeeAmount
  }

  bytes memory txData = _serializeTxArgs(txArgs);

  require(eccm.crossChain(toChainId, targetProxyHash, "unlock", txData), "EthCrossChainManager crossChain executed error!");

  emit LockEvent(address(this), toChainId, targetProxyHash, txData);
}
```

Example fee deduction implementation for unlock:
```
function unlock(bytes memory argsBs, bytes memory fromContractAddr, uint64 fromChainId) onlyManagerContract returns (bool) {
  TxArgs memory args = _deserializTxArgs(argsBs);
  bytes32 key = hash(args.toAssetHash, fromChainId, fromContractAddr, args.fromAssetHash);

  require(registry[key] == true);
  require(balances[key] >= args.amount);

  // Use SafeMath to ensure balances do not overflow
  balances[key] = balances[key].sub(args.amount);

  uint256 afterFeeAmount = args.amount;
  if (args.feeAmount > 0) {
    // ensure that there is no overflow
    afterFeeAmount = amount.sub(args.feeAmount);

    // transfer feeAmount to feeReceiverAddr
    require(_transferFromContract(args.toAssetHash, feeReceiverAddr, feeAmount), "transfer asset from lock_proxy contract to toAddress failed!");
  }

  // send tokens to `toAddress`
  require(_transferFromContract(args.toAssetHash, toAddress, afterFeeAmount), "transfer asset from lock_proxy contract to toAddress failed!");

  emit UnlockEvent(fromContractAddr, fromChainId, args.toAddress, afterFeeAmount);

  return true;
}
```

If `deductFeeInLock` is set to `true`, it is possible for a relayer to receive the fee, and then not send the transaction to the PolyChain or the target chain.

This PIP does not propose a fixed solution to this issue, but possible solutions include:
1. Requiring deductFeeInLock to be false
2. Having a reputation system for relayers
3. Requiring relayers to have some value staked, which can be slashed if they are reported for slow broadcasts or non-broadcasts

This proposal does not allow for the fee asset to be different from the asset being transferred, the option for this could be covered in a different PIP.
