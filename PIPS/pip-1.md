---
pip: 1
title: Permissionless Duo LockProxy
status: Draft
author: John Wong (@jwheur)
created: 2020-06-23
---

## Simple Summary
The current LockProxy implementation requires an operator to control `setManagerProxy`, `bindProxyHash` and `bindAssetHash`.

This PIP proposes an alternative implementation which does not require an operator.
The tradeoff of this implementation is that an asset can be transferred between at most two chains.

:white_check_mark: Supported Example:
1. Lock token A => Ethereum LockProxy --> Neo LockProxy => Unlock token A
2. Lock token A => Neo LockProxy --> Ethereum LockProxy => Unlock token A

:x: Not Supported Example:
1. Lock token A => Ethereum LockProxy --> Neo LockProxy => Unlock token A
2. Lock token A => Neo LockProxy --> Ontology LockProxy => Unlock token A
3. Lock token A => Ontology LockProxy --> Ethereum LockProxy => This will fail

## Specification
To support non-native tokens within the LockProxy, a token contract should be deployed to represent the non-native token.

The token's constructor should call `delegateAsset` on the LockProxy that it is delegated to, for example:

```
constructor (address lockProxyContractAddress, uint64 nativeChainId, bytes memory nativeLockProxy, bytes memory nativeAssetHash) public ERC20Detailed("ONT Token", "ONTX", 0) {
    uint256 totalSupply = 1000000000;
    _mint(lockProxyContractAddress, totalSupply);
    lockProxy.delegateAsset(nativeChainId, nativeLockProxy, nativeAssetHash, totalSupply);
}
```

There are two purposes of `delegateAsset`:
1. For the LockProxy contract to record assets that have been delegated to it
2. To send a `registerAsset` cross-chain transaction to the specified `nativeChainId` and `nativeLockProxy`

In the LockProxy, `delegateAsset` can be implemented as:
```
mapping(bytes32 => bool) registry;
mapping(bytes32 => uint256) balances;

struct RegisterAssetTxArgs {
  bytes assetHash;
  bytes nativeAssetHash;
}

function delegateAsset(uint64 nativeChainId, bytes memory nativeLockProxy, bytes memory nativeAssetHash, uint256 delegatedSupply) public {
    address assetHash = _msgSender();

    // `hash` is any supported hashing function of the respective blockchain
    key = hash(assetHash, nativeChainId, nativeLockProxy, nativeAssetHash);

    require(registry[key] != true);
    require(balances[key] == 0);
    require(getBalanceFor(assetHash) == delegatedSupply);

    registry[key] = true;
    balances[key] = delegatedSupply;

    RegisterAssetTxArgs memory txArgs = RegisterAssetTxArgs({
        assetHash: assetHash,
        nativeAssetHash: nativeAssetHash
    });

    bytes memory txData = _serializeRegisterAssetTxArgs(txArgs);
    require(eccm.crossChain(nativeChainId, nativeLockProxy, "registerAsset", txData), "EthCrossChainManager crossChain executed error!");

    emit DelegateAsset(assetHash, nativeChainId, nativeLockProxy, nativeAssetHash);
}
```

The `registerAsset` is called by cross-chain transactions from other LockProxies.
The purpose of this function is to update the `registry` mapping which will be checked when a user calls the `lock` function.
This helps to ensure that the user does not call `lock` with incorrect parameters.

In the LockProxy, `registerAsset` can be implemented as:
```
function registerAsset(bytes memory argsBs, bytes memory fromContractAddr, uint64 fromChainId) onlyManagerContract public {
  TxRegisterAssetArgs memory args = _deserializTxRegisterAssetArgs(argsBs);

  // `hash` is any supported hashing function
  key = hash(args.nativeAssetHash, fromChainId, fromContractAddr, args.assetHash);

  require(registry[key] != true);
  registry[key] = true;
}
```

`setManagerProxy` can be restricted to be called once. After it is called the first time, the `managerProxyContract` address cannot be changed.

`bindProxyHash` and `bindAssetHash` functions can be removed.

The LockProxy should have a `lock` function which locks tokens from the user and sends a cross-chain transaction to unlock tokens on the targeted `toChainId` and `targetProxyHash`:
```
struct TxArgs {
  bytes fromAssetHash;
  bytes toAssetHash;
  bytes toAddress;
  uint256 amount;
}

function lock(address fromAssetHash, uint64 toChainId, bytes memory targetProxyHash, bytes memory toAssetHash, bytes memory toAddress, uint256 amount) public {
  bytes32 key = hash(fromAssetHash, toChainId, targetProxyHash, toAssetHash);
  require(registry[key] == true);

  // Use SafeMath to ensure balances do not overflow
  balances[key] = balances[key].add(amount);

  // transfer tokens from user to LockProxy

  address fromContractAddr = address(this);
  TxArgs memory txArgs = TxArgs({
      fromAssetHash: fromAssetHash,
      toAssetHash: toAssetHash,
      toAddress: toAddress,
      amount: amount
  });
  bytes memory txData = _serializeTxArgs(txArgs);

  require(eccm.crossChain(toChainId, targetProxyHash, "unlock", txData), "EthCrossChainManager crossChain executed error!");

  emit LockEvent(address fromAssetHash, address fromAddress, uint64 toChainId, bytes toAssetHash, bytes toAddress, uint256 amount, ...optional);
}
```

The LockProxy should have an `unlock` function which can be called only by the cross-chain manager contract. The function validates the parameters then sends the tokens to the `toAddress`:
```
function unlock(bytes memory argsBs, bytes memory fromContractAddr, uint64 fromChainId) onlyManagerContract returns (bool) {
  TxArgs memory args = _deserializTxArgs(argsBs);
  bytes32 key = hash(args.toAssetHash, fromChainId, fromContractAddr, args.fromAssetHash);

  require(registry[key] == true);
  require(balances[key] >= args.amount);

  // Use SafeMath to ensure balances do not overflow
  balances[key] = balances[key].sub(args.amount);

  // send tokens to `toAddress`

  emit UnlockEvent(address toAssetHash, address toAddress, uint256 amount, ...optional);

  return true;
}
```
