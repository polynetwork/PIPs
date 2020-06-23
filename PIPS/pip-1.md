---
pip: 1
title: Permissionless LockProxies
status: Draft
author: John Wong (@jwheur)
created: 2020-06-23
---

## Simple Summary
The current LockProxy implementation requires an operator to control `setManagerProxy`, `bindProxyHash` and `bindAssetHash`.

This PIP proposes an alternative implementation which does not require an operator.

## Specification
`setManagerProxy` can be restricted to be called once. After it is called the first time, the `managerProxyContract` address cannot be changed.

`bindProxyHash` and `bindAssetHash` functions can be removed.

The `lock` function can be modified to require the following parameters:
- address fromAssetHash
- uint64 toChainId
- bytes memory targetProxyHash
- bytes memory toAssetHash
- bytes memory toAddress
- uint256 amount

The LockProxy contract should maintain a `balances` mapping of (bytes32 => uint256).
This `balances` mapping should be updated within the `lock` function:
```
key = hash(fromAssetHash, toChainId, targetProxyHash, toAssetHash)
balances[key] += amount
```

Where `hash` is any supported hashing function of the respective blockchain.

The `unlock` function should be modified to require the following parameters:
- bytes memory argsBs

Where `argsBs` contains `fromAssetHash`, `fromChainId`, `fromProxyHash` and `toAssetHash`, `toAddress` and `amount`.

The contract should check that the balances are sufficient, then reduce the balance and perform the unlock:
```
key = hash(toAssetHash, fromChainId, fromProxyHash, fromAssetHash)
require(balances[key] > amount)
balances[key] -= amount
```
