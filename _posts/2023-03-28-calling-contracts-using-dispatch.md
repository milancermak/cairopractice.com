---
title: Calling contracts using dispatch
date: 2023-03-28
author: m
tags: [starknet, cairo]
---

A defining feature of programmable blockchains is the ability to call other contracts. Let's explore how to do this in Starknet using Cairo.

We start by declaring an interface of the called contract by using the `#[abi]` attribute:

```rust
#[abi]
trait IMintable {
    fn mint_to(receiver: ContractAddress, amount: u128);
}
```
{: .nolineno }

When compiled, the `#[abi]` block is expanded into code that looks something like this:

```rust
trait IMintableDispatcherTrait<T> {
    fn mint_to(self: T, receiver: ContractAddress, amount: u128);
}

#[derive(Copy, Drop)]
struct IMintableDispatcher {
    contract_address: starknet::ContractAddress,
}

impl IMintableDispatcherImpl of IMintableDispatcherTrait::<IMintableDispatcher> {
    fn mint_to(self: IMintableDispatcher, receiver: ContractAddress, amount: u128) {
        // starknet::call_contract_syscall is called
        // in the body of the function
    }
}

#[derive(Copy, Drop)]
struct IMintableLibraryDispatcher {
    class_hash: starknet::ClassHash,
}

impl IMintableLibraryDispatcherImpl of IMintableDispatcherTrait::<IMintableLibraryDispatcher> {
    fn mint_to(self: IMintableLibraryDispatcher, receiver: ContractAddress, amount: u128) {
        // starknet::syscalls::library_call_syscall is called
        // in the body of the function
    }
}
```
{: .nolineno }

Behold the power of *Cairo plugins* ✨

When the compiler encounters the `#[abi]` attribute, it executes the [dispatcher plugin](https://github.com/starkware-libs/cairo/blob/main/crates/cairo-lang-starknet/src/plugin/dispatcher.rs) that generates all the boilerplate code for us. The generated code contains a new trait, two new structs and their implementation of this trait. We didn't have to write anything else besides the initial interface declaration.

To do a regular contract call, we only need to instantiate the `IMintableDispatcher` struct. It takes a `contract_address` value, the address of an external contract which we want to interact with. We can now call any of the interface functions on this struct. Note if we declared `IMintable` in a different scope, we'll need to import it via `use`. 

Here's a full example:

```rust
#[abi]
trait IMintable {
    fn mint_to(receiver: starknet::ContractAddress, amount: u128);
}   

#[contract]
mod Altruist {
    // importing IMintableDispacher and the trait into the module scope
    use super::IMintableDispatcher;
    use super::IMintableDispatcherTrait;
    use starknet::contract_address_const;
    use starknet::ContractAddress;

    #[external]
    fn mint_some(receiver: ContractAddress, amount: u128) {
        // address of the contract we want to call
        let token_addr: ContractAddress = contract_address_const::<0xc0ffee>();

        // create a dispatcher using the token address
        let token = IMintableDispatcher { contract_address: token_addr };

        // call a function from the IMintable interface
        token.mint_to(receiver, amount);
    }
}
```
{: .nolineno }

If we need to do a library call, we can do it in a similar manner using `IMintableLibraryDispatcher` struct.

```rust
let token = IMintableLibraryDispatcher { class_hash: token_class_hash };
token.mint_to(...);
```
{: .nolineno }
