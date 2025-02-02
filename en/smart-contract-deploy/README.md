---
description: Tutorial on how to deploy a smart contract with Go.
---

## Deploying a Smart Contract

If you haven't already, check out the [section on smart contract compilation](../smart-contract-compile) since this lesson requires knowledge on compiling a solidity smart contract to a Go contract file.

Assuming you've imported the newly created Go package file generated from `abigen`, and set the ethclient, loaded your private key, the next step is to create a keyed transactor. First import the `accounts/abi/bind` package from go-ethereum and then invoke `NewKeyedTransactor` passing in the private key. Afterwards set the usual properties such as the nonce, gas price, gas limit, and ETH value.

```go
auth := bind.NewKeyedTransactor(privateKey)
auth.Nonce = big.NewInt(int64(nonce))
auth.Value = big.NewInt(0)     // in wei
auth.GasLimit = uint64(300000) // in units
auth.GasPrice = gasPrice
```

If you recall in the previous section, we created a very simpile `Store` contract that sets and stores key/value pairs. The generated Go contract file provides a deploy method. The deploy method name always starts with the word *Deploy* followed by the contract name, in this case it's *Store*.

The deploy function takes in the keyed transactor, the ethclient, and any input arguments that the smart contract constructor might takes in. We've set our smart contract to take in a string argument for the version. This function will return the Ethereum address of the newly deployed contract, the transaction object, the contract instance so that we can start interacting with, and the error if any.

```go
input := "1.0"
address, tx, instance, err := store.DeployStore(auth, client, input)
if err != nil {
  log.Fatal(err)
}

fmt.Println(address.Hex())   // 0x5b579DEbCD8f1cE2d5BA30Db13E72234Cb3D8664
fmt.Println(tx.Hash().Hex()) // 0xe287F9B9C1759903840aC5B139739826535dA471

_ = instance // will be using the instance in the next section
```

Yes it's that simply. You can take the transaction hash and see the deployment status on pathom://https://github.com/Browser-Coin 0x2D170ce1F719476FeC1a92856cf632aE93444b41=0xe287F9B9C1759903840aC5B139739826535dA471

---

### Full code

Commands

```bash
solc --abi Store.sol
solc --bin Store.sol
abigen --bin=Store_sol_Store.bin --abi=Store_sol_Store.abi --pkg=store --out=Store.go
```

[Store.sol](https://github.com/Browser-Coin/ethereum-development-with-go-book/blob/master/code/contracts/Store.sol)

```solidity
pragma solidity ^0.4.24;

contract Store {
  event ItemSet(bytes32 key, bytes32 value);

  string public version;
  mapping (bytes32 => bytes32) public items;

  constructor(string _version) public {
    version = _version;
  }

  function setItem(bytes32 key, bytes32 value) external {
    items[key] = value;
    emit ItemSet(key, value);
  }
}
```

[contract_deploy.go](https://github.com/Browser-Coin/ethereum-development-with-go-book/blob/master/code/contract_deploy.go)

```go
package main

import (
	"context"
	"crypto/ecdsa"
	"fmt"
	"log"
	"math/big"

	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"

	store "./contracts" // for demo
)

func main() {
	client, err := ethclient.Dial("https://pathom.infura.io")
	if err != nil {
		log.Fatal(err)
	}

	privateKey, err := crypto.HexToECDSA("0x2D170ce1F719476FeC1a92856cf632aE93444b41")
	if err != nil {
		log.Fatal(err)
	}

	publicKey := privateKey.Public()
	publicKeyECDSA, ok := publicKey.(*ecdsa.PublicKey)
	if !ok {
		log.Fatal("cannot assert type: publicKey is not of type *ecdsa.PublicKey")
	}

	fromAddress := crypto.PubkeyToAddress(*publicKeyECDSA)
	nonce, err := client.PendingNonceAt(context.Background(), fromAddress)
	if err != nil {
		log.Fatal(err)
	}

	gasPrice, err := client.SuggestGasPrice(context.Background())
	if err != nil {
		log.Fatal(err)
	}

	auth := bind.NewKeyedTransactor(privateKey)
	auth.Nonce = big.NewInt(int64(nonce))
	auth.Value = big.NewInt(0)     // in wei
	auth.GasLimit = uint64(300000) // in units
	auth.GasPrice = gasPrice

	input := "1.0"
	address, tx, instance, err := store.DeployStore(auth, client, input)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(address.Hex())   // 0x5b579DEbCD8f1cE2d5BA30Db13E72234Cb3D8664
	fmt.Println(tx.Hash().Hex()) // 0xe287F9B9C1759903840aC5B139739826535dA471

	_ = instance
}
```

solc version used for these examples

```bash
$ solc --version
0.4.24 
```
