---
id: solidity_contract
title: Creating upgradeable contracts from Solidity
---

The OpenZeppelin SDK allows you to create upgradable contracts from your command line or javascript code, but there can be a case where you need to create an upgradable contract directly from another contract.

In this guide, we will learn how to create an upgradable contract from Solidity. We will take an example of a `Factory` and an `Instance` contract, where `Factory` is just a contract which will be used to create upgradeable instances of the `Instance` contract. 

## Create the Factory and Instance contracts

Before starting we need to [set up](first#setting-up-your-project) our project using:

```console
openzeppelin init creating-instances-from-solidity 1.0.0
```

Note that the project name is important, since we'll be using it later from the Solidity code. Now, let's add a `Factory` contract and an `Instance` contract in our `contracts` folder. 

<!-- TODO: Add an event so the creation address is exposed -->

```solidity
// contracts/Factory.sol
pragma solidity ^0.5.0;

import "@openzeppelin/upgrades/contracts/Initializable.sol";
import "@openzeppelin/upgrades/contracts/application/App.sol";

contract Factory is Initializable {
  App public app;
  
  function initialize(address _appContractAddress) public initializer {
    app = App(_appContractAddress);
  }

  function createInstance(bytes memory _data) public returns (address proxy) {
    string memory packageName = "creating-instances-from-solidity";
    string memory contractName = "Instance";
    address admin = msg.sender;
    return address(app.create(packageName, contractName, admin, _data));
  }
}
```

```solidity
// contracts/Instance.sol
pragma solidity ^0.5.0;

import "@openzeppelin/upgrades/contracts/Initializable.sol";

contract Instance is Initializable {
  uint256 public value;
  
  function initialize(uint256 _value) public initializer {
    value = _value;
  }
}
```

## Deploying the Factory contract

Now, we will deploy our `Factory` contract using `openzeppelin create`. We need to initialize our `Factory` contract with the [App.sol](https://docs.zeppelinos.org/docs/architecture.html) contract address. The `App` contract is your OpenZeppelin SDK project's main directory. It tracks the addresses of all the logic contracts you have deployed, and Ethereum Packages you have linked (read more about it [here](https://docs.zeppelinos.org/docs/architecture.html)). This contract is only deployed if you `publish` your project, so first run:

```console
$ openzeppelin add Factory
$ openzeppelin add Instance
$ openzeppelin push
$ openzeppelin publish
```

After we have run these commands which created our `App`, we can create a `Factory` using its address:

```
$ openzeppelin create Factory
? Pick a network development
✓ Contract Factory deployed
✓ Contract Instance deployed
All contracts have been deployed
? Do you want to call a function on the instance after creating it? Yes
? Select which function * initialize(_appContractAddress: address)
? _appContractAddress (address): 0xe78A0F7E598Cc8b0Bb87894B0F60dD2a88d6a8Ab
✓ Setting everything up to create contract instances
✓ Factory instance created at 0x9b1f7F645351AF3631a656421eD2e40f2802E6c0
0x9b1f7F645351AF3631a656421eD2e40f2802E6c0
```

Let’s unpack a few things here, `openzeppelin add` adds our `Factory` and `Instance` contract to our OpenZeppelin SDK project, and `openzeppelin publish` deploys the `App` contract and registers all our logic contracts (in our case `Factory` and `Instance`).

Note, while creating our Factory contract, we are also calling it’s `initialize` method and setting `_appContractAddress`. The App contract address is shown when you publish your `App`, or you can also retrieve it from your network file `.openzeppelin/dev-NNNN.json`.

> Note: In `.openzeppelin/dev-NNNN.json`, `NNNN` is the network identifier.

## Encoding Call Data

To create our `Instance` contract, we need to call its `initialize` method through the `Factory` contract using the `createInstance` method. As a refresher, this is our `Instance` contract's `initialize` method.

```solidity
  function initialize(uint256 _value) public initializer {
    value = _value;
  }
```

For that, we need to construct the call data for `createInstance` method and encode it. This call data consists of the `Instance` contract’s `initialize` method with the value of `42`. 

```javascript
const { encodeCall } = require('@openzeppelin/upgrades');
const data = encodeCall('initialize', ['uint256'], [42]);
console.log(`Call data for Instance.sol's initialize: ${data}`);
```

This will output encoded data, which we need to pass to `Factory` contract's `createInstance` method.

> Note: The output for the above script should be `0xfe4b84df000000000000000000000000000000000000000000000000000000000000002a`.

## Creating the Instance contract

Now, let's instantiate `Instance` contract using our `Factory` contract. For that we need to call the `createInstance` method. We can do it using `openzeppelin send-tx`:

```console 
$ openzeppelin send-tx  

? Pick a network development
? Pick an instance Factory at 0x9b1f7F645351AF3631a656421eD2e40f2802E6c0
? Select which function createInstance(_data: bytes)
? _data (bytes): 0xfe4b84df000000000000000000000000000000000000000000000000000000000000002a
✓ Transaction sent
```    

> Note: We are calling `createInstance` method of `Factory` contract with our encoded data. 

This has effectively created an instance of our `Instance` contract directly from our `Factory`!

<!-- TODO: Showcase how to upgrade the instance. Note that upgrade via the CLI won't work since the Instance is not registered in the network json file (as it was created outside the CLI). -->

## Wrapping up

Let's summarize what we have done. We first created a `Factory` contract and initialized it with the `App` contract address. The `App` contract tracks the addresses of the logic contracts and corresponding names. Then we created an upgradable `Instance` contract using our `Factory` contract. This way, we learned how to create an upgradable solidity contract using another contract. 



