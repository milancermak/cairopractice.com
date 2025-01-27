---
title: Storing user defined types, part 2
date: 2023-04-06
author: m
tags: [starknet, cairo]
---

*Deprecation notice: Since this post was published, Starknet and Cairo have evolved. As such, parts of the post might be obsolete. For the best, most up-to-date resources, please consult the [official Starknet documentation](https://docs.starknet.io/) and the [Cairo book](https://book.cairo-lang.org/).*

In the [first part]({% post_url 2023-04-03-storing-user-defined-types-pt-1 %}) of this series, we learnt how to implement `StorageAccess` for a custom data type so we can use it in contract storage. The struct itself was pretty simple, containing only primitive types. Now what if we want to store a more complex type, itself containing a custom struct?

```rust
#[derive(Drop, Serde)]
struct Pos2D {
    x: u16,
    y: u16
}

#[derive(Drop, Serde)]
struct Trip {
    origin: Pos2D,
    destination: Pos2D,
    tag: felt252
}
```
{: .nolineno }

A naive implementation of the StorageAccess trait for `Trip` could go through all of the embedded structs and call read/write one by one on every primitive type. Obviously though, we want to reuse what we've built already, i.e. we want to call `StorageAccess::read` and `StorageAccess::write` for `origin` and `destination` when dealing with an instance of `Trip`. Let's see how we can achieve that, dealing with `write` first:

```rust
fn write(address_domain: u32, base: StorageBaseAddress, value: Trip) -> SyscallResult::<()> {
    StorageAccess::write(address_domain, base, value.origin)?;

    let destination_base = storage_base_address_from_felt252(
      storage_address_from_base_and_offset(base, 2_u8).into()
    );
    StorageAccess::write(address_domain, destination_base, value.destination)?;

    let tag_base = storage_base_address_from_felt252(
      storage_address_from_base_and_offset(base, 4_u8).into()
    );
    StorageAccess::write(address_domain, tag_base, value.tag)
}
```
{: .nolineno }

To write `origin` and `destination`, we reuse the Pos2D `StorageAccess` implementation by calling `StorageAccess::write`. Similarly for `tag`, which is a native type (felt252), hence comes with `StorageAccess` out of the box.

It's worth noticing two things. One, the compiler is smart enough to infer the types, we don't have to do `StorageAccess::<Pos2D>::write`. Two, we have to use a *different base address* for each member when calling `write`. To understand why, let's check how an instance of `Trip` looks in Starknet's storage using the above implementation:

```text
┌───────────────────────────────────────────────────────────────────────────────────┐
│                                                                                   │
│                                      trip                                         │
│                                                                                   │
├─────────────────────────────────┬─────────────────────────────────┬───────────────┤
│                                 │                                 │               │
│             origin              │           destination           │      tag      │
│                                 │                                 │               │
├────────────────┬────────────────┼────────────────┬────────────────┼───────────────┤
│                │                │                │                │               │
│        x       │        y       │        x       │        y       │               │
│                │                │                │                │               │
├────────────────┼────────────────┼────────────────┼────────────────┼───────────────┤
│ ^              │                │ ^              │                │ ^             │
│ trip base      │                │ trip base + 2  │                │ trip base + 4 │
│ origin base    │                │ dest. base     │                │ tag base      │
└────────────────┴────────────────┴────────────────┴────────────────┴───────────────┘
```
{: .nolineno }

It's one continuous space, `origin` occupying two slots, `destination` next two and the last slot belonging to `tag`. The base address of `origin` and `trip` are the same, but we have to calculate it using `storage_address_from_base_and_offset` for the other members. Sadly, there's no "size_of" function in Cairo (yet?), so we have to specify the offsets manually. So base address for `destination` is trip base plus and offset of 2 (2 being the slot size necessary to store `origin`). Base address for `tag` is trip base plus 4 (2 for `origin` + 2 for `destination`). In other words, it functions as an array.

Given this knowledge, we can deal with `read` as well:

```rust
fn read(address_domain: u32, base: StorageBaseAddress) -> SyscallResult::<Trip> {
    let origin = StorageAccess::read(address_domain, base)?;

    let destination_base = storage_base_address_from_felt252(storage_address_from_base_and_offset(base, 2_u8).into());
    let destination = StorageAccess::read(address_domain, destination_base)?;

    let tag_base = storage_base_address_from_felt252(storage_address_from_base_and_offset(base, 4_u8).into());
    let tag = StorageAccess::read(address_domain, tag_base)?;

    Result::Ok(Trip { origin, destination, tag })
}
```
{: .nolineno }

In the same fashion as above, we use the appropriate base addresses for each member of the `Trip` struct.

That's it. We now have a full implementation of `StorageAccess` for our slightly more complex `Trip` data type. As usual, I'm attaching a full contract with a simple test (please use the lateset Cairo version (HEAD) as this won't compile with 1.0.0-alpha.6).

```rust
use starknet::StorageAccess;
use starknet::StorageBaseAddress;
use starknet::SyscallResult;
use starknet::storage_access;
use starknet::storage_read_syscall;
use starknet::storage_write_syscall;
use starknet::storage_base_address_from_felt252;
use starknet::storage_address_from_base_and_offset;
use traits::Into;
use traits::TryInto;
use option::OptionTrait;

#[derive(Drop, Serde)]
struct Pos2D {
    x: u16,
    y: u16
}

#[derive(Drop, Serde)]
struct Trip {
    origin: Pos2D,
    destination: Pos2D,
    tag: felt252
}

impl Pos2DStorageAccess of StorageAccess::<Pos2D> {
    fn read(address_domain: u32, base: StorageBaseAddress) -> SyscallResult::<Pos2D> {
        Result::Ok(
            Pos2D {
                x: storage_read_syscall(
                    address_domain,
                    storage_address_from_base_and_offset(base, 0_u8)
                )?.try_into().unwrap(),
                y: storage_read_syscall(
                    address_domain,
                    storage_address_from_base_and_offset(base, 1_u8)
                )?.try_into().unwrap(),
            }
        )
    }

    fn write(address_domain: u32, base: StorageBaseAddress, value: Pos2D) -> SyscallResult::<()> {
        storage_write_syscall(
            address_domain,
            storage_address_from_base_and_offset(base, 0_u8),
            value.x.into()
        )?;
        storage_write_syscall(
            address_domain,
            storage_address_from_base_and_offset(base, 1_u8),
            value.y.into()
        )
    }
}

impl TripStorageAccess of StorageAccess::<Trip> {
    fn read(address_domain: u32, base: StorageBaseAddress) -> SyscallResult::<Trip> {
        let origin = StorageAccess::read(address_domain, base)?;

        let destination_base = storage_base_address_from_felt252(storage_address_from_base_and_offset(base, 2_u8).into());
        let destination = StorageAccess::read(address_domain, destination_base)?;

        let tag_base = storage_base_address_from_felt252(storage_address_from_base_and_offset(base, 4_u8).into());
        let tag = StorageAccess::read(address_domain, tag_base)?;

        Result::Ok(Trip { origin, destination, tag })
    }

    fn write(address_domain: u32, base: StorageBaseAddress, value: Trip) -> SyscallResult::<()> {
        StorageAccess::write(address_domain, base, value.origin)?;

        let destination_base = storage_base_address_from_felt252(storage_address_from_base_and_offset(base, 2_u8).into());
        StorageAccess::write(address_domain, destination_base, value.destination)?;

        let tag_base = storage_base_address_from_felt252(storage_address_from_base_and_offset(base, 4_u8).into());
        StorageAccess::write(address_domain, tag_base, value.tag)
    }
}

#[contract]
mod Travel {
    use super::Pos2D;
    use super::Trip;

    struct Storage {
        trip: Trip
    }

    #[view]
    fn get_trip() -> Trip {
        trip::read()
    }

    #[external]
    fn set_trip(ox: u16, oy: u16, dx: u16, dy: u16, tag: felt252) {
        let origin = Pos2D { x: ox, y: oy };
        let destination = Pos2D { x: dx, y: dy };
        trip::write( Trip { origin, destination, tag });
    }
}

#[test]
#[available_gas(1000000)]
fn test_trip_storage() {
    let trip = Travel::get_trip();
    assert(trip.origin.x == 0_u16, 'init origin x');
    assert(trip.origin.y == 0_u16, 'init origin y');
    assert(trip.destination.x == 0_u16, 'init destination x');
    assert(trip.destination.y == 0_u16, 'init destination y');
    assert(trip.tag == 0, 'init tag');

    let ox = 12_u16;
    let oy = 5_u16;
    let dx = 24_u16;
    let dy = 10_u16;
    let tag = 'business';
    Travel::set_trip(ox, oy, dx, dy, tag);

    let trip = Travel::get_trip();
    assert(trip.origin.x == ox, 'set origin x');
    assert(trip.origin.y == oy, 'set origin y');
    assert(trip.destination.x == dx, 'set destination x');
    assert(trip.destination.y == dy, 'set destination y');
    assert(trip.tag == tag, 'set tag');
}
```
