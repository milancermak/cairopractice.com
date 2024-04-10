---
title: Understanding upgradability
date: 2024-04-10
author: m
tags: [cairo, starknet]
---

Starknet gives us, developers, the power to make our contracts upgradable in a native way. Unlike in the EVM world where one has to fiddle with the inferior proxy pattern, in Starknet we can utilize a single syscall to upgrade the logic of our smart contracts.

Let's take a closer look at the whole mechanism.

## Contract class & contract instance

First thing to be aware of is the distinction between [a contract class and contract instance](https://docs.starknet.io/documentation/architecture_and_concepts/Smart_Contracts/contract-classes/).

A class is like a blueprint for a contract - it contains the logic (code), but has no state (storage). We create it via a `declare` transaction and as a result get its class hash. To create a contract (an instance of a class), we `deploy` the class.

One way to think about it is that a deployed contract is a pairing of logic with state. When upgrading a contract, we only want to change the logic and keep state untouched. That's why this separation is very convenient.

```text
┌─────────────────────────────────────────────────────────┐
│               deployed contract @ 0x1337                │
│                                                         │
├────────────────────────────┬────────────────────────────┤
│      logic of 0x1337       │      state of 0x1337       │
│              │             │                            │
└──────────────┼─────────────┴────────────────────────────┘
               │
┌──────────────┼─────────────┐
│              ▼             │
│  declared class @ 0xc0de   │
│                            │
└────────────────────────────┘
```

## replace_class_syscall

To upgrade a contract, we use the `replace_class_syscall`. It takes a single argument, a class hash (as obtained by `declare`) of the class that will serve as the contract's new logic. It's that simple - one syscall and we're done.

There's a few things to be aware of:

1. The address of the upgraded contract stays the same, regardless how many time it has been upgraded. Once it's deployed, it lives at the address forever. No need to update scripts, frontends and other integration points.

2. Obviously, the function in which `replace_class_syscall` is being called should be public, but protected by some kind of an auth check. We don't want North Korea upgrading our contracts at will.

3. We can always remove upgradability from a contract, but never add it back again.

4. When upgrading, the `constructor` is not called.

5. The class upgrade only takes effect after the function cointaining the `replace_class_syscall` ends. However if we were to use `call_contract_syscall` on self, we'd already be interacting with the new one.

## Conservation of storage

As mentioned above, when upgrading, contract storage stays untouched. Everything that was written to storage before an upgrade will be there after the upgrade. In other words, we can remove a variable declaration from `struct Storage` and that way we won't be able to `read` or `write` to it *direcly* anymore, but we can still use `storage_read_syscall` and `storage_write_syscall` to access its values. We can also redeclare the variable in the struct and upgrade the contract once more.

This has some cool implications. For example, let's say we have a contract that's [`Ownable`](https://docs.openzeppelin.com/contracts-cairo/0.11.0/access#ownership_and_ownable) and we want to migrate it to a two step Ownable (one where the proposed owner has to accept the ownership first). The *only* thing that needs to change in the contract code is the use of the impl.

From:

```rust
// before, 1 step ownable
#[abi(embed_v0)]
impl OwnableImpl = OwnableComponent::OwnableImpl<ContractState>;
```

To:

```rust
// new, 2 step ownable
#[abi(embed_v0)]
impl OwnableTwoStepImpl = OwnableComponent::OwnableTwoStepImpl<ContractState>;
```

Under the hood, the `OwnableComponent` uses a storage variable called `Ownable_owner` for both impls, so when upgrading the old owner is preserved.

Happy upgrading.
