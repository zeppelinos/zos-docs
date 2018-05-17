---
id: advanced
title: Advanced topics
---

We expand on several advanced topics for the more intrepid users of ZeppelinOS. 

## Preserving the storage structure
As mentioned in the [Building upgradeable applications](building-upgradeable.md) guide, when upgrading your contracts, you need to make sure that all variables declared in prior versions are kept in the code. New variables must be declared below the previously existing ones, as such:

    contract MyContract_v1 {
      uint256 public x;
    }

    contract MyContract_v2 {
      uint256 public x;
      uint256 public y;
    }

Note that this must be so _even if you no longer use the variables_. There is no restriction (apart from gas limits) on including new variables in the upgraded versions of your contracts, or on removing or adding functions. 

This necessity is due to how [Solidity uses the storage space](https://solidity.readthedocs.io/en/v0.4.21/miscellaneous.html#layout-of-state-variables-in-storage). In short, the variables are allocated storage space in the order they appear (for the whole variable or some pointer to the actual storage slot, in the case of dynamically sized variables). When we upgrade a contract, its storage contents are preserved. This entails that if we remove variables, the new ones will be assigned storage space that is already occupied by the old variables; all we would achieve is losing the pointers to that space, with no guarantees on its content.

## Initializers vs. constructors

## Format of `zos.json` and `zos.<network>.json` files

