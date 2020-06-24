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

In the LockProxy, `delegateAsset` can be implemented as:
```
struct DelegatedAsset {
    uint64 nativeChainId;
    bytes nativeLockProxy;
    bytes nativeAssetHash;
}

mapping(address => DelegatedAsset) delegatedAssets;
mapping(bytes32 => uint256) balances;

function delegateAsset(uint64 nativeChainId, bytes memory nativeLockProxy, bytes memory nativeAssetHash, uint256 totalSupply) public {
    address assetHash = _msgSender();

    require(nativeChainId != 0);
    require(delegatedAssets[assetHash].nativeChainId == 0);

    // `hash` is any supported hashing function of the respective blockchain
    bytes32 key = hash(assetHash, nativeChainId, nativeLockProxy, nativeAssetHash);

    require(balances[key] == 0);
    balances[key] = totalSupply;

    delegatedAssets[_msgSender()] = DelegatedAsset(nativeChainId, nativeLockProxy, nativeAssetHash);
}
```

This mapping will be used in the `unlock` function to ensure that delegated tokens will be unlocked only if the specified token is locked.

`setManagerProxy` can be restricted to be called once. After it is called the first time, the `managerProxyContract` address cannot be changed.

`bindProxyHash` and `bindAssetHash` functions can be removed, to replace them we can add a `registerAsset` function:

```
mapping(bytes32 => bool) registry;

function registerAsset(address fromAssetHash, uint64 toChainId, bytes memory targetProxyHash, bytes memory toAssetHash) {
    key = hash(fromAssetHash, toChainId, targetProxyHash, toAssetHash);
    registry[key] = true;
}
```

This function can be permissionless as its main purpose is to help prevent careless mistakes from the user when calling the `lock` function.

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
bytes32 key = hash(fromAssetHash, toChainId, targetProxyHash, toAssetHash);
require(registry[key] == true);

balances[key] += amount;

TxArgs memory txArgs = TxArgs({
    fromContractAddr: address(this),
    fromAssetHash: fromAssetHash,
    toAssetHash: toAssetHash,
    toAddress: toAddress,
    amount: amount
});
bytes memory txData = _serializeTxArgs(txArgs);

emit LockEvent(txData);
```

The `unlock` function should be modified to require the following parameters:
- bytes memory argsBs

Where `argsBs` contains `fromContractAddr`, `fromAssetHash`, `fromChainId`, `toAssetHash`, `toAddress` and `amount`.

The contract should check that the balances are sufficient, then reduce the balance and perform the unlock:
```
TxArgs memory args = _deserializTxArgs(argsBs);
DelegatedAsset dAsset = delegatedAssets[args.toAssetHash];

if (dAsset.nativeChainId != 0) {
    require(
        dAsset.nativeChainId == args.fromChainId &&
        dAsset.nativeLockProxy == args.fromContractAddr &&
        dAsset.nativeAssetHash == args.fromAssetHash
    );
}

bytes32 key = hash(args.toAssetHash, args.fromChainId, args.fromContractAddr, args.fromAssetHash);
require(balances[key] >= args.amount);
balances[key] -= args.amount;

emit UnlockEvent(args.fromChainId, args.fromContractAddr, args.fromAssetHash, args.toAssetHash, args.toAddress, args.amount);
```
