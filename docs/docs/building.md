---
id: building
title: Building an upgradeable application
sidebar_label: Building an upgradeable app
---

After installing `zos` and setting up our ZeppelinOS project as described in the [setup](setup.md) guide, we are ready to create our upgradeable application.

Let's start by installing the [ZeppelinOS lib](https://github.com/zeppelinos/zos-lib):

```sh
npm install zos-lib --save
```

Next, we will create a file named `MyContract.sol` in the `contracts/` folder of our app with the following Solidity code:

```sol
import "zos-lib/contracts/migrations/Migratable.sol";

contract MyContract is Migratable {
  uint256 public x;

  function initialize(uint256 _x) isInitializer("MyContract", "0") public {
    x = _x;
  }
}
```

Notice that our sample contract has an `initialize` function: this works as a constructor for our upgradeable contract. This is a requirement of the [ZeppelinOS's upgradeability system](advanced.md#initializers-vs-constructors).

Now let's compile the contract:

```sh
npx truffle compile
```

## Initial deployment

To deploy our app, we need to register the first version of our contract:

```sh
zos add MyContract
```

> **Note**: If you are working in a local development network like [Ganache](http://truffleframework.com/ganache/), you will need to [configure](http://truffleframework.com/docs/getting_started/project#alternative-migrating-with-ganache) your `truffle.js` file before running the `push` command.

We can now push our application to the network by running:

```sh
zos push --network <network>
```

This  will create a `zos.<network>.json` file with all the information specific to the chosen network. You can read more about this file in the [advanced topics](advanced.md#format-of-zosjson-and-zos-network-json-files) section.

To create an upgradeable version of our contract, we need to run:

```sh
zos create MyContract --init initialize --args 42 --network <network>
```

The `create` command takes an optional `--init` flag to call the initialization function after creating the contract, while the `--args` flag is to pass arguments to it. This way, we are initializing our contract with `42` as the value of the `x` state variable.


After these simple steps, our upgradeable application is now on-chain!

## Upgrading our contract

If, at a later stage, we want to upgrade our smart contract code in order to fix a bug or add a new feature, we can do it seamlessly using ZeppelinOS. 

> **Note**: while ZeppelinOS supports arbitrary changes in functionality, you will need to preserve all variables that appear in prior versions of your contracts, declaring any new variables below the already existing ones. You can find details on this in the [advanced topics](advanced.md) page. 

Once we have made the desired changes to our contracts, we need to push them to the network:

```sh
zos push --network <network>
```

> **Note**: bear in mind that the `push` command of the ZeppelinOS CLI also compiles our contracts. To avoid this, use the `--skip-compile` flag.

Finally, let's upgrade the already deployed contract:

```sh
zos upgrade MyContract --network <network>
```

_Voilà_, we have deployed and upgraded an application using ZeppelinOS! The address of the upgraded contract is the same as before, but the code has been updated to the new version.

Next step is to [use the ZeppelinOS standard libraries](using.md) from our application.
