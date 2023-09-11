---
title: Store packing
date: 2023-09-07
author: m
tags: [cairo, starknet]
---

One of my favorite recent additions to Cairo is the [`StorePacking` trait](https://github.com/starkware-libs/cairo/blob/9e5427b9178bb6663c68b46203ea5493296b8e48/corelib/src/starknet/storage_access.cairo#L68):

```rust
trait StorePacking<T, PackedT> {
    fn pack(value: T) -> PackedT;
    fn unpack(value: PackedT) -> T;
}
```
{: .nolineno }

A super simple trait that declares an interface to transform a type `T` into it its compacted representation `PackedT` and vice versa. It's nothing special on its own. Its power shines when we consider a companion `impl` that's been added to corelib along side it:

```rust
impl StoreUsingPacking<
    T,
    PackedT,
    impl TPacking: StorePacking<T, PackedT>,
    impl PackedTStore: Store<PackedT>
> of Store<T>
```
{: .nolineno }

It's an implementation of the `Store` trait (the one responsible for reading and writing into contract storage) for packed types. Under the hood, it calls `pack(T)` when writing to storage and `unpack(TPacked)` when reading from it. Here's the magical part - it all happens automatically. If a type is packable, the compiler will know to use `StoreUsingPacking`, pack before writing and unpack after reading from storage.

Let's consider a practical example.

Say we have a `Pos3` type representing a position in 3D space using three `u64` values. To avoid paying for three storage updates when updating a position, we can pack all three 64 bit values into a single `felt252`. This way, we'll only ever pay for a single storage update.

It could look something like this:

```rust
#[derive(Copy, Drop, Serde)]
struct Pos3 {
    x: u64,
    y: u64,
    z: u64
}

const SHIFT: felt252 = 0x10000000000000000; // 2**64

impl PackPos3 of starknet::StorePacking<Pos3, felt252> {
    fn pack(value: Pos3) -> felt252 {
        (value.x.into() +
         value.y.into() * SHIFT +
         value.z.into() * SHIFT * SHIFT)
    }

    fn unpack(value: felt252) -> Pos3 {
        let value: u256 = value.into();
        let shift: NonZero<u256> = integer::u256_try_as_non_zero(SHIFT.into()).unwrap();
        let (rest, x) = integer::u256_safe_div_rem(value, shift);
        let (z, y) = integer::u256_safe_div_rem(rest, shift);

        Pos3 {
            x: x.try_into().unwrap(),
            y: y.try_into().unwrap(),
            z: z.try_into().unwrap()
        }
    }
}

#[starknet::contract]
mod game {
    use super::Pos3;

    #[storage]
    struct Storage {
        player_position: Pos3
    }

    #[external(v0)]
    fn update_player_position(ref self: ContractState, position: Pos3) {
        // this write will automatically pack the
        // position into a single felt252
        self.player_position.write(position);
    }
}

```

Notice the **only** thing we had to do is implement `StorePacking<Pos3, felt252>`. Since we're packing into a `felt252` which has an impl of `Store` in corelib, the rest will work seamlessly.

Yet another wonderful example of the power of Cairo. Go pack those bits!
