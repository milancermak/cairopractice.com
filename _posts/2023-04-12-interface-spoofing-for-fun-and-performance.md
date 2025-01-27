---
title: Interface spoofing for fun and performance
date: 2023-04-12
author: m
tags: [starknet, cairo]
---

*Deprecation notice: Since this post was published, Starknet and Cairo have evolved. As such, parts of the post might be obsolete. For the best, most up-to-date resources, please consult the [official Starknet documentation](https://docs.starknet.io/) and the [Cairo book](https://book.cairo-lang.org/).*

What happens when types in a [Starknet contract interface]({% post_url 2023-03-28-calling-contracts-using-dispatch %}) do not match the contract we're interfacing with? Let's find out.

We'll [use a simple contract](https://testnet.starkscan.co/contract/0x059a4016a63ea2efb46c94a18b87488cf5fd18925f9fa96af9d236d6c56a6b85#read-write-contract) just to get some data:

```rust
#[contract]
mod Source {
    use array::Array;
    use array::ArrayTrait;

    #[view]
    fn a_u16() -> u16 {
        42_u16
    }

    #[view]
    fn a_u32() -> u32 {
        3141592_u32
    }

    #[view]
    fn a_u256() -> u256 {
        // max value of u256
        u256 {
            low: 0xffffffffffffffffffffffffffffffff_u128,
            high: 0xffffffffffffffffffffffffffffffff_u128
        }
    }

    #[view]
    fn an_array() -> Array<felt252> {
        let mut a: Array<felt252> = ArrayTrait::new();
        a.append(10);
        a.append(20);
        a.append(30);
        a
    }
}
```
{: .nolineno }

And we'll [use this contract](https://testnet.starkscan.co/contract/0x051bf826010f76e1aefed6e3a2a585ad1028935fc12497c0e23ebc6419333cf8#read-write-contract) to query for the values:

```rust
use array::Array;

#[abi]
trait IProper {
    fn a_u16() -> u16;
    fn a_u32() -> u32;
    fn a_u256() -> u256;
    fn an_array() -> Array<felt252>;
}

#[abi]
trait ISpoofed {
    fn a_u16() -> u64;
    fn a_u32() -> u8;
    fn a_u256() -> felt252;
    fn an_array() -> felt252;
}

#[contract]
mod Examiner {
    use array::Array;
    use starknet::ContractAddress;
    use super::IProperDispatcher;
    use super::IProperDispatcherTrait;
    use super::ISpoofedDispatcher;
    use super::ISpoofedDispatcherTrait;

    struct Storage {
        src: ContractAddress
    }

    #[constructor]
    fn constructor(src_address: ContractAddress) {
        src::write(src_address);
    }

    #[view]
    fn proper_u16() -> u16 {
        IProperDispatcher { contract_address: src::read() }.a_u16()
    }

    #[view]
    fn spoofed_u16() -> u64 {
        ISpoofedDispatcher { contract_address: src::read() }.a_u16()
    }

    #[view]
    fn proper_u32() -> u32 {
        IProperDispatcher { contract_address: src::read() }.a_u32()
    }

    #[view]
    fn spoofed_u32() -> u8 {
        ISpoofedDispatcher { contract_address: src::read() }.a_u32()
    }

    #[view]
    fn proper_u256() -> u256 {
        IProperDispatcher { contract_address: src::read() }.a_u256()
    }

    #[view]
    fn spoofed_u256() -> felt252 {
        ISpoofedDispatcher { contract_address: src::read() }.a_u256()
    }

    #[view]
    fn proper_array() -> Array<felt252> {
        IProperDispatcher { contract_address: src::read() }.an_array()
    }

    #[view]
    fn spoofed_array() -> felt252 {
        ISpoofedDispatcher { contract_address: src::read() }.an_array()
    }
}
```
{: .nolineno }

Above, we declared two interfaces. `IProper` is the boring one - its types match the types in the `Source` contract. `ISpoofed` is the wicked one - its types are different. So what happens when we use the "spoofed" functions?

Calling `spoofed_u16` is the least surprising. It just works. The value of 42 fits in a `u64` so our main contract "transforms" a `u16` into a `u64` (this is not quite correct, we'll explore later in the post what's going on under the hood).

What about when a value returned by the source contract does not fit into the type declared by the calling contract? Such is the case of `spoofed_u32`. The number 3141592 overflows a `u8`, so naturally we would expect a runtime error. That's exaclty the outcome of calling this function. Executing it, we get a cryptic 0x52657475726e6564206461746120746f6f2073686f7274 error. Decoding that from a felt encoded string to a human readable one, we see the actual error message: "Returned data too short".

Surely then, when calling `spoofed_u256`, we would expect the same to happen. After all, the max value of `u256` does not fit into a `felt252`. Surprisingly, this time we do *not* get a runtime error! Instead, the function returns 340282366920938463463374607431768211455, the max value of `u128`. What the heck is going on?

To understand this trickery we have to understand how values are passed between contract calls. In other words serialization and deserialization or serde for short. In Cairo, serde is [implemented](https://github.com/starkware-libs/cairo/blob/main/corelib/src/serde.cairo) via the `Serde` trait:

```rust
trait Serde<T> {
    fn serialize(ref serialized: Array<felt252>, input: T);
    fn deserialize(ref serialized: Span<felt252>) -> Option<T>;
}
```
{: .nolineno }

When the source contract "sends" a `u256` value, the `Serde::<u256>::serialize` function gets called which in turn calls `Serde::<u128>::serialize` on the `low` and `high` members of the `u256` struct which in turn (again) calls `Serde::<felt252>::serialize` for each `u128` value. Here are the relevant functions from the serde module:

```rust
// serialization for u256
fn serialize(ref serialized: Array<felt252>, input: u256) {
    Serde::<u128>::serialize(ref serialized, input.low);
    Serde::<u128>::serialize(ref serialized, input.high);
}

// serialization for u128, called in the above
fn serialize(ref serialized: Array<felt252>, input: u128) {
    Serde::<felt252>::serialize(ref serialized, input.into());
}

// serialization for felt252, called in the above
fn serialize(ref serialized: Array<felt252>, input: felt252) {
    serialized.append(input);
}
```
{: .nolineno }

The result of this call chain is that the `low` and `high` values from the original `u256` get appended into the `serialized` array as `felt252` values.

Now let's investigate what happens on the other end during deserialization. Because we declared `felt252` as the return value of `fn a_u256()` in our `ISpoofed` interface, it's the `Serde::<felt252>::deserialize` that gets executed when reconstructing the serialized values:

```rust
fn deserialize(ref serialized: Span<felt252>) -> Option<felt252> {
    Option::Some(*serialized.pop_front()?)
}
```
{: .nolineno }

Mystery solved! When deserializing `felt252`, Cairo takes only the first value from the array (`serialized.pop_front()`) and returns it. In our case, it does not care that another value was appended to it during serialization. It got the felt we asked for so it's done. That's why `spoofed_u256` returns the max value of `u128` instead of overflowing during runtime.

You should now be able to understand why `spoofed_array` returns 3. [Let me know](https://twitter.com/cairopractice) if you do (or don't) figure it out.

---

That was a fun exercise, but what is this technique actually useful for? Well, I can think of a situation where it applies.

We know that [u256 was a mistake](https://twitter.com/VitalikButerin/status/1433255593683259398). Unfortunatelly it is so prevalent it is difficult to move away from it. Believe me, [I tried](https://community.starknet.io/t/a-felt-based-erc20-token/1956) 💀 However in the Cairo VM, using smaller types is [better for performance](https://www.youtube.com/watch?v=uW2tTR53J7o&t=3390s). So what if we could keep our code fast and cheap to run but still interoperable with the rest of the ecosystem? That's exactly what we can achieve with interface spoofing.

For example, we deploy a token (ERC20 or ERC721) where we *know* balances cannot exceed `u64`. We can then use this type to store user balances and also return it in the `balance_of` function. Yet anyone can still query our token using the standard interface `fn balance_of(addr: ContractAddress) -> u256` and receive the right value, because `u64` will get deserialized into `u256` on their end correctly ✨

---

In a more abstract sense, we can define our own way of interpreting return values of a protocol that fit our use case. I can see this being useful in for example onchain gaming. Can you think of other use cases for interface spoofing? [I'll be glad to hear from you.](https://twitter.com/cairopractice)
