---
id: pattern
title: OpenZeppelin SDK Upgrades Pattern
---

This article describes the "unstructured storage" proxy pattern, the fundamental building block of the OpenZeppelin SDK's upgrades.

> Note: For a more in depth read, please see [our proxy-patterns blog post](https://blog.zeppelinos.org/proxy-patterns/), which discusses the need for proxies, goes into more technical detail on the subject, elaborates on other possible proxy patterns that were considered for the OpenZeppelin SDK, and more.

## Why upgrade a contract?

By design, smart contracts are immutable. On the other hand, software quality heavily depends on the ability to upgrade and patch source code in order to produce iterative releases. Even though blockchain based software profits significantly from the technology's immutability, still a certain degree of mutability is needed for bug fixing and potential product improvements. The OpenZeppelin SDK solves this apparent contradiction by providing an easy to use, simple, robust, and opt-in upgrade mechanism for smart contracts that can be controlled by any type of governance, be it a multi-sig wallet, a simple address or a complex DAO.

## Upgrading via the proxy pattern

The basic idea is using a proxy for upgrades. The first contract is a simple wrapper or "proxy" which users interact with directly and is in charge of forwarding transactions to and from the second contract, which contains the logic. The key concept to understand is that the logic contract can be replaced while the proxy, or the access point is never changed. Both contracts are still immutable in the sense that their code cannot be changed, but the logic contract can simply be swapped by another contract. The wrapper can thus point to a different logic implementation and in doing so, the software is "upgraded".

```
User ---- tx ---> Proxy ----------> Implementation_v0
                     |
                      ------------> Implementation_v1
                     |
                      ------------> Implementation_v2
```

## Proxy forwarding

The most immediate problem that proxies need to solve is how the proxy exposes the entire interface of the logic contract without requiring a one to one mapping of the entire logic contract's interface. That would be difficult to maintain, prone to errors, and would make the interface itself not upgradeable. Hence, a dynamic forwarding mechanism is required. The basics of such a mechanism are presented in the code below:

```solidity
assembly {
  let ptr := mload(0x40)

  // (1) copy incoming call data
  calldatacopy(ptr, 0, calldatasize)

  // (2) forward call to logic contract
  let result := delegatecall(gas, _impl, ptr, calldatasize, 0, 0)
  let size := returndatasize

  // (3) retrieve return data
  returndatacopy(ptr, 0, size)

  // (4) forward return data back to caller
  switch result
  case 0 { revert(ptr, size) }
  default { return(ptr, size) }
}
```

This code can be put in the [fallback function](https://solidity.readthedocs.io/en/v0.4.21/contracts.html#fallback-function) of a proxy, and will forward any call to any function with any set of parameters to the logic contract without it needing to know anything in particular of the logic contract's interface. In essence, (1) the `calldata` is copied to memory, (2) the call is forwarded to the logic contract, (3) the return data from the call to the logic contract is retrieved, and (4) the returned data is forwarded back to the caller. The technique needs to be implemented using Yul because [Solidity's `delegatecall`](https://solidity.readthedocs.io/en/v0.4.21/introduction-to-smart-contracts.html#delegatecall-callcode-and-libraries) returns a boolean instead of the callee's return data.

A very important thing to note is that the code makes use of the EVM's `delegatecall` opcode which executes the callee's code in the context of the caller's state. That is, the logic contract controls the proxy's state and the logic contract's state is meaningless. Thus, the proxy doesn't only forward transactions to and from the logic contract, but also represents the pair's state. The state is in the proxy and the logic is in the particular implementation that the proxy points to.

## Unstructured storage proxies

A problem that quickly comes up when using proxies has to do with the way in which variables are stored in the proxy contract. Suppose that the proxy stores the logic contract's address in it's only variable `address public _implementation;`. Now, suppose that the logic contract is a basic token whose first variable is `address public _owner`. Both variables are 32 byte in size, and as far as the EVM knows, occupy the first slot of the resulting execution flow of a proxied call. When the logic contract writes to `_owner`, it does so in the scope of the proxy's state, and in reality writes to `_implementation`. This problem can be referred to as a "storage collision".

```
|Proxy                     |Implementation           |
|--------------------------|-------------------------|
|address _implementation   |address _owner           | <=== Storage collision!
|...                       |mapping _balances        |
|                          |uint256 _supply          |
|                          |...                      |
```

There are many ways to overcome this problem, and the "unstructured storage" approach which the OpenZeppelin SDK implements works as follows. Instead of storing the `_implementation` address at the proxy's first storage slot, it chooses a pseudo random slot instead. This slot is sufficiently random, that the probability of a logic contract declaring a variable at the same slot is negligible. The same principle of randomizing slot positions in the proxy's storage is used in any other variables the proxy may have, such as an admin address (that is allowed to update the value of `_implementation`), etc.

```
|Proxy                     |Implementation           |
|--------------------------|-------------------------|
|...                       |address _owner           |
|...                       |mapping _balances        |
|...                       |uint256 _supply          |
|...                       |...                      |
|...                       |                         |
|...                       |                         |
|...                       |                         |
|...                       |                         |
|address _implementation   |                         | <=== Randomized slot.
|...                       |                         |
|...                       |                         |
```

An example of how the randomized storage is achieved, following [EIP 1967](http://eips.ethereum.org/EIPS/eip-1967):

```solidity
bytes32 private constant implementationPosition = bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1));
```

As a result, a logic contract doesn't need to care about overwriting any of the proxy's variables. Other proxy implementations that face this problem usually imply having the proxy know about the logic contract's storage structure and adapt to it, or instead having the logic contract know about the proxy's storage structure and adapt to it. This is why this approach is called "unstructured storage"; neither of the contracts needs to care about the structure of the other.

## Storage collisions between implementation versions

As discussed, the unstructured approach avoids storage collisions between the logic contract and the proxy. However, storage collisions between different versions of the logic contract can occur. In this case, imagine that the first implementation of the logic contract stores `address public _owner` at the first storage slot and an upgraded logic contract stores `address public _lastContributor` at the same first slot. When the updated logic contract attempts to write to the `_lastContributor` variable, it will be using the same storage position where the previous value for `_owner` was being stored, and overwrite it!

### Incorrect storage preservation:
```
|Implementation_v0   |Implementation_v1        |
|--------------------|-------------------------|
|address _owner      |address _lastContributor | <=== Storage collision!
|mapping _balances   |address _owner           |
|uint256 _supply     |mapping _balances        |
|...                 |uint256 _supply          |
|                    |...                      |
```

### Correct storage preservation:
```
|Implementation_v0   |Implementation_v1        |
|--------------------|-------------------------|
|address _owner      |address _owner           |
|mapping _balances   |mapping _balances        |
|uint256 _supply     |uint256 _supply          |
|...                 |address _lastContributor | <=== Storage extension.
|                    |...                      |
```

The unstructured storage proxy mechanism doesn't safeguard against this situation.
It is up to the user to have new versions of a logic contract extend previous versions, or otherwise guarantee that the storage hierarchy is always appended to but not modified.
However, the OpenZeppelin SDK does detect such collisions, and warns the developer appropriately.

## The constructor caveat

In Solidity, code that is inside a constructor or part of a global variable declaration is not part of a deployed contract's runtime bytecode. This code is executed only once, when the contract instance is deployed. As a consequence of this, the code within a logic contract's constructor will never be executed in the context of the proxy's state. To rephrase, proxies are completely oblivious to the existence of constructors. It's simply as if they weren't there for the proxy.

The problem is easily solved though. Logic contracts should move the code within the constructor to a regular 'initializer' function, and have this function be called whenever the proxy links to this logic contract. Special care needs to be taken with this initializer function so that it can only be called once, which is one of the properties of constructors in general programming.

This is why when the OpenZeppelin CLI creates a proxy, it allows you to indicate an initializer function:

```console
npx openzeppelin create MyLogicContract --init initialize --args arg1,arg2,arg3
```

With this command, the OpenZeppelin SDK creates a proxy that wraps around `MyLogicContract`, uses `MyLogicContract` as the logic contract, and calls the logic contract's `initialize` function.

To ensure that the `initialize` function can only be called once, a simple modifier is used. The OpenZeppelin SDK provides this functionality via a contract that can be extended:

```solidity

import "@openzeppelin/upgrades/contracts/Initializable.sol";

contract MyContract is Initializable {

  function initialize(address arg1, uint256 arg2, bytes arg3) initializer public payable {
    // "constructor" code...
  }

}

```

Notice how the contract extends `Initializable` and implements the `initializer` provided by it.

## Transparent proxies and function clashes

As described in the previous sections, upgradeable contract instances (or proxies) work by delegating all calls to a logic contract. However, the proxies need some functions of their own, such as `upgradeTo(address)` to upgrade to a new implementation. This begs the question of how to proceed if the logic contract also has a function named `upgradeTo(address)`: upon a call to that function, did the caller intend to call the proxy or the logic contract?

> Clashing can also happen among functions with different names. Every function that is part of a contract's public ABI is identified, at the bytecode level, by a 4-byte identifier. This identifier depends on the name and arity of the function, but since it's only 4 bytes, there is a possibility that two different functions with different names may end up having the same identifier. The Solidity compiler tracks when this happens within the same contract, but not when the collision happens across different ones, such as between a proxy and its logic contract. Read [this article](https://medium.com/nomic-labs-blog/malicious-backdoors-in-ethereum-proxies-62629adf3357) for more info on this.

The way the OpenZeppelin SDK deals with this problem is via the _transparent proxy_ pattern. A transparent proxy will decide which calls are delegated to the underlying logic contract based on the caller address (ie the `msg.sender`):
- If the caller is the admin of the proxy (the address with rights to upgrade the proxy), then the proxy will **not** delegate any calls, and only answer any messages it understands.
- If the caller is any other address, the proxy will **always** delegate a call, no matter if it matches one of the proxy's functions.

Assuming a proxy with an `owner()` and an `upgradeTo()` function, that delegates calls to an ERC20 contract with an `owner()` and a  `transfer()` function, the following table covers all scenarios:

| msg.sender | owner() | upgradeTo() | transfer() |
|---------------|---------|-------------|------------|
| Owner         | returns proxy.owner() | returns proxy.upgradeTo() | fails |
| Other         | returns erc20.owner() | fails | returns erc20.transfer() |

Fortunately, the OpenZeppelin CLI accounts for this situation, and creates an intermediary ProxyAdmin contract that is in charge of all the proxies you create via the cli. Even if you call the `create` command from your node's default account, or using the `from` address specified when you started your `openzeppelin session`, the ProxyAdmin contract will be the actual admin of all your proxies. This means that you will be able to interact with the proxies from any of your node's accounts, without having to worry about the nuances of the transparent proxy pattern. Only advanced users of the OpenZeppelin SDK that use it programmatically via `@openzeppelin/upgrades` need to be aware of the transparent proxies pattern.

## Summary

Any developer using the OpenZeppelin SDK should be familiar with proxies in the ways that are described in this article. In the end, the concept is very simple, and the OpenZeppelin SDK is designed to encapsulate all the proxy mechanics in a way that the amount of things you need to keep in mind when developing projects are reduced to an absolute minimum. It all comes down to the following list:

* Have a basic understanding of what a proxy is
* Always extend storage instead of modifying it
* Make sure your contracts use initializer functions instead of constructors

Furthermore, the OpenZeppelin SDK will let you know when something goes wrong with one of the items in this list.
